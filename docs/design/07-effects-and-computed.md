# 07 · Effects & Computed Values

[← Index](./README.md) · Addresses review items B1, B2, B6

## 1. Effects vs. computed: when to use which

| | Computed | Effect |
|---|----------|--------|
| Model | Declarative: "B is always f(A)" | Imperative: "when A changes, do X to B" |
| Runs on | Any change to its dependencies, from any origin | Changes to the source field written by an origin with `effects: true` (§2.1); host code can opt out with `runEffects: false` |
| Survives loading | Yes, recomputed | Doesn't fire, so loaded data is safe |
| Example | `subtotal = qty * price` | Changing `country` clears `state` |

## 2. Write origins and effect triggers

Every write is tagged with an **origin**: a label saying where it came from. Callers never pass a built-in origin. The engine sets it from the method that was called, so code can't claim to be "loading" to skip the rules.

| Origin | Set by |
|--------|--------|
| `user` | `FieldController.setValue` / array controller commands, i.e. a person through a renderer |
| `host` | `engine.setValue`, `engine.array(...)`, and other direct calls from host code |
| `effect` | Effects, cascade invalidation, `autoSelectSingle`, reverse fill, `whenHidden: 'clear'` |
| `computed` | Computed values |
| `load` | Loading and reloading ([09 §2](./09-loading-and-submission.md#2-data-load-pipeline), [§3](./09-loading-and-submission.md#3-reload-policy)) |
| `external` | `engine.applyExternal` ([09 §7](./09-loading-and-submission.md#7-external-updates-and-conflicts)) |
| `resume` | Installing exported server-rendering state ([09 §9](./09-loading-and-submission.md#9-server-rendering)) |
| `reset` | `engine.reset()` |
| `default` | Defaults, including expression defaults |
| `history` | Undo/redo |
| `restore` | `engine.restore(snapshot)` ([09 §8](./09-loading-and-submission.md#8-snapshots-drafts)) |
| `x-…` | A host-registered custom origin (§2.1) |

### 2.1 Origin policy

What an origin **does** is defined in one table. Effects, cascades, plugins, and the store read this table and never test origin names directly. The core table below holds core origins; plugins register their own rows at install (`history` → `history`, `drafts` → `restore`, [03 §8.1](./03-state-model.md#81-plugin-api)):

```ts
interface OriginPolicy {
  layer: 'current' | 'loaded' | 'default' | null;    // which value layer is written (null: not stored)
  effects: boolean;                                   // run effects on the written paths
  invalidate: boolean;                                // apply cascade onChange to dependents (13 §4)
  history: 'record' | 'merge' | 'skip' | 'clear' | 'clear-overlap';   // flag for the history plugin; core ignores it
}

export const ORIGIN_POLICY = {
  user:     { layer: 'current',  effects: true,  invalidate: true,  history: 'record' },
  host:     { layer: 'current',  effects: true,  invalidate: true,  history: 'record' },
  effect:   { layer: 'current',  effects: true,  invalidate: true,  history: 'merge'  },
  computed: { layer: null,       effects: false, invalidate: true,  history: 'skip'   },
  load:     { layer: 'loaded',   effects: false, invalidate: false, history: 'clear'  },
  resume:   { layer: null,       effects: false, invalidate: false, history: 'clear'  },   // installs all layers (09 §9)
  external: { layer: 'loaded',   effects: false, invalidate: false, history: 'clear-overlap' },
  reset:    { layer: 'current',  effects: false, invalidate: false, history: 'clear'  },
  default:  { layer: 'default',  effects: false, invalidate: false, history: 'skip'   },
} as const satisfies Record<string, OriginPolicy>;

// registered by plugins in setup (ctx.origins.register)
origins.register('history', { layer: 'current', effects: false, invalidate: false, history: 'skip'  });   // history plugin
origins.register('restore', { layer: 'current', effects: false, invalidate: false, history: 'clear' });   // drafts plugin
```

`history` values ([03 §6.2](./03-state-model.md#62-undo--redo-opt-in)):

| Value | Meaning |
|-------|---------|
| `record` | The tick starts a new undo step |
| `merge` | Joins the step of the tick's root write. If the root wasn't recorded (`skip`), neither is this write; if the root cleared history, nothing is recorded |
| `skip` | Not recorded; history is left as is |
| `clear` | Clears both stacks (the baseline changed) |
| `clear-overlap` | Clears both stacks only if a written path appears in a recorded step ([09 §7.3](./09-loading-and-submission.md#73-interaction-with-undo)) |

Notes:
- `computed` invalidates because a computed parent that changes really did change (it's derived from edited values). `load` and `external` don't: loaded children are trusted.
- `resume` writes no single layer: it installs `default`, `loaded`, `current`, rows and meta as exported, with no inbound mapping, effects or conflict detection; dirty bits and counts are rebuilt from the layers ([09 §9](./09-loading-and-submission.md#9-server-rendering)).
- `reset({ keepLoaded: false })` also empties `loaded` in the same tick. This is the one write to `loaded` outside `layer`; it's part of the reset's definition ([03 §7](./03-state-model.md#7-engine-write-api-core)).
- Origins never set `touched`. Only `handlers.onBlur()` ([11 §2](./11-rendering-and-adapters.md#2-headless-field-api)) and `engine.touch()` do, so values written by host code don't reveal validation messages before the person has interacted with the field.
- **The only per-call override** is `engine.setValue(p, v, { runEffects: false })` (and the same option on array commands), for bulk host writes. It turns off `effects` for that write only, and its commit entries record it.

**Custom origins.** A host can register its own origin, for example for a sync engine or an import, so observers can tell those writes apart. It must use the `x-` prefix and extend a built-in origin's policy (`extends`, any built-in except `user` and `resume`, so host code can't pass its writes off as user input), optionally overriding fields:

```ts
registry.origins.register('x-import', { extends: 'host', history: 'skip' });   // a host write, but not undoable
engine.setValue('/email', v, { origin: 'x-import' });
```

- `engine.setValue` accepts `origin` **only** for registered `x-` origins. Passing a built-in name (`'load'`) or an unregistered one is `E_UNKNOWN_ORIGIN`. Registering a name without `x-` is `E_RESERVED_ORIGIN`.
- A custom origin can't use `layer: 'loaded'` or `'default'`. Those layers are written only through loading, `applyExternal`, and defaults, which keep their own bookkeeping (row keys, conflicts, dirty baselines). Any of these violations (and `extends: 'user'`) is `E_RESERVED_ORIGIN`.
- Effects cascaded from a custom-origin write use origin `effect` with `history: 'merge'`, so they follow the root: with `history: 'skip'` on the custom origin, its cascades aren't recorded either.

Observers (commit hooks, `subscribe`, devtools) see the origin as plain data on every write: `{ path, prev, next, origin }`. For custom origins, `origin` is the registered name (`'x-import'`), not the origin it extends.

```ts
interface Effect extends Extensible {
  on?: 'change' | 'blur';             // default 'change'
  when?: Condition;                    // optional gate
  target: PathRef;
  action: 'clear' | 'set' | 'reset' | 'revert' | 'validate' | `x-${string}`;
  value?: Json | Expression;           // for 'set'
}
```

Rules:
1. Effects run only for writes whose origin policy has `effects: true` (§2.1), including cascades. Cascade depth is limited by `limits.effectDepth` in the engine options (default 8). Going deeper is `E_EFFECT_DEPTH`: the remaining effects are dropped, and the diagnostic lists the chain of `(source → target)` effects that led there.
2. Host code can turn them off per call with `{ runEffects: false }`.
3. So effects **never** run for origins with `effects: false`, such as loading or undo/redo ([03 §6.2](./03-state-model.md#62-undo--redo-opt-in)). This prevents a classic bug (present in the v4 draft) where a clear effect wipes loaded data during load.
4. `on: 'change'` effects only fire when the value actually changes (`!deepEqual(prev, next)`); `on: 'blur'` effects fire on the blur event without a change test.

Action semantics:
- `clear`: write the empty value for the field type (`''`, `null`, `[]`, `false`) into `current`.
- `set`: write the evaluated `value` into `current`.
- `reset`: write the **default** value into `current`, ignoring loaded.
- `revert`: delete the key from `current`, falling back through the normal resolve order ([03 §3](./03-state-model.md#3-precedence-by-presence-not-nullishness): computed for an overridable computed field, then loaded, then default).
- Arrays are atomic **in storage**, not in command targeting ([03 §3.2](./03-state-model.md#32-atomic-array-writes)). For `items[2].qty`, all four actions clone the resolved array and change only that cell. `reset` uses that field's default in its row scope; `revert` uses its baseline value through the persisted baseline row-key map, not current index. A genuinely new row uses the field's default/empty value; an unresolved old row after a stale-baseline change rejects with `E_ROW_BASELINE_UNKNOWN`. Other cells, rows and row keys stay unchanged. **Keep the enclosing array override**, including after a descendant revert and even when it equals the baseline. Equality never erases explicit `set`/`clear`/`reset` intent. Targeting `items` itself still resets the whole array (an explicit current value) or reverts it (deletes its override).
- `validate`: schedule validation of the target (respecting "already validated once").
- `x-*`: a host-registered effect handler. It receives the engine write API with origin `effect`:

```ts
type EffectHandler = (input: { target: DataPath; value?: Json; source: DataPath; effect: Effect },
                      api: { setValue(p: DataPath, v: Json): void; clear(p: DataPath): void; validate(p: DataPath): void; get(p: DataPath): Json }) => void;   // synchronous
```

## 3. Computed values

```ts
interface ComputedConfig extends Extensible {
  expr: Expression;
  overridable?: boolean;          // default false
}
```

- Not overridable: the value is **not stored** in any layer. It is evaluated into `values` during resolve, and the field renders read-only. This is the simplest model and needs no sync.
- Overridable: the computed result acts as a **virtual layer between `loaded` and `current`** ([03 §3](./03-state-model.md#3-precedence-by-presence-not-nullishness)), and is the field's dirty baseline ([03 §4](./03-state-model.md#4-dirty-semantics)). A user write goes into `current` and wins. `revert` brings back the computed value. The view exposes `isOverridden(path)`.

Computed values take part in the DAG and in cycle detection.

### 3.1 Expression defaults (replaces `mode: 'initial'`)

The earlier `initial` mode wrote a pre-filled value into `current`, which made a freshly opened form **dirty**. It's replaced by letting `defaultValue` be an Expression:

```json
{ "kind": "field", "id": "startDate", "fieldType": "date-time",
  "defaultValue": { "lang": "json-logic", "expr": { "var": "$context.today" } } }
{ "kind": "field", "id": "shippingName", "fieldType": "text",
  "defaultValue": { "lang": "json-logic", "expr": { "var": "../billingName" } } }
```

- The result is written to the **`default` layer** (origin `default`, so no effects run). The field isn't dirty, because dirty compares against the layers below `current` ([03 §4](./03-state-model.md#4-dirty-semantics)).
- It is re-evaluated whenever its dependencies change. Because of presence precedence ([03 §3](./03-state-model.md#3-precedence-by-presence-not-nullishness)), that only shows when neither `current` nor `loaded` has the path. So it's a **live default until the user edits or saved data exists**, with no extra bookkeeping.
- The `reset` effect writes the current evaluated default into `current`. `engine.reset()` instead clears `current`, so values fall back to `loaded`, then this default (`keepLoaded: false` empties `loaded`, so only the default remains).

| Want | Use |
|------|-----|
| Always equal to f(x), not editable | `computed` |
| f(x) until the user overrides it, revertable | `computed` + `overridable` |
| Pre-fill from f(x) for new records, never dirty | `defaultValue: Expression` |

### 3.2 Aggregates

Totals are ordinary expressions with the built-in `sum`, `count`, `min`, `max` operators ([05 §2](./05-expressions.md#2-evaluator-registry)):

```json
{ "kind": "field", "id": "totalQty", "fieldType": "number",
  "computed": { "expr": { "lang": "json-logic", "expr": { "sum": { "var": "/items[*].qty" } } } } }
```

- Cost is O(rows) per evaluation, the same as a full recompute; any `[*]` expression re-reads every row when one changes.
- There's no incremental (delta) aggregate in core. It may return as a plugin if the M1 benchmark demands it ([12](./12-open-questions-and-roadmap.md)).

## 4. Hidden and disabled field policy

```ts
type HiddenPolicy   = 'keep' | 'exclude' | 'clear';
type DisabledPolicy = 'keep' | 'exclude';

// FormContract.policies: form-wide defaults
interface FieldPolicies {
  whenHidden?: HiddenPolicy;       // built-in default 'keep'
  whenDisabled?: DisabledPolicy;   // built-in default 'exclude'
  validateHidden?: boolean;        // built-in default false
  validateDisabled?: boolean;      // built-in default false
}
// Any field / group / array / dynamic-group can set the same keys to override.
```

**Hidden** (`whenHidden`):

| Policy | Value in state | Submitted |
|--------|----------------|-----------|
| `keep` (default) | kept | yes |
| `exclude` | kept (comes back when shown again) | no |
| `clear` | the field's **empty value** is written into `current` when it becomes hidden (origin `effect`, so cascades run), so loaded or default values don't reappear when shown again. See below for when this happens | no |

`clear` writes only when the field **becomes** hidden in a tick whose root origin has `effects: true` (§2.1). For other origins (loading, `restore`, `reset`, defaults, undo), the hidden state is computed without writing, so a field that is hidden right after loading keeps its loaded value in state (it's still not submitted).

**Disabled** (`whenDisabled`):

| Policy | Value in state | Submitted |
|--------|----------------|-----------|
| `exclude` (default) | kept | no |
| `keep` | kept | yes |

These defaults match React Hook Form / Formik for hidden fields and native HTML / Angular for disabled ones.

### 4.1 Resolving the effective policy

For each policy key, the **nearest explicit setting wins**:

```
field setting ?? nearest ancestor container setting ?? FormContract.policies ?? built-in default
```

Example: the form sets `whenHidden: 'exclude'`, a group sets `whenHidden: 'clear'`, and one field inside it sets `whenHidden: 'keep'`. That field keeps its value; its siblings are cleared; fields outside the group are excluded.

Visibility and disabled state are inherited: a field inside a hidden (or disabled) group counts as hidden (or disabled) and uses its **own** effective policy.

### 4.2 Combining hidden and disabled

A field is submitted only if **neither** applicable policy excludes it: if hidden, `whenHidden` must be `keep`; if disabled, `whenDisabled` must be `keep`. `clear` takes effect as soon as the field becomes hidden, whatever its disabled state.

### 4.3 Validation

Hidden fields are validated only if `validateHidden` is true, and disabled fields only if `validateDisabled` is true. Both default to false and resolve with the same precedence as above. With `keep`, a hidden value may be submitted unvalidated; set `validateHidden: true` where that matters.
