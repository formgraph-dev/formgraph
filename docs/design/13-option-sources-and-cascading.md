# 13 · Option Sources & Cascading Selects

[← Index](./README.md) · Builds on [04](./04-paths-and-dependency-graph.md), [07](./07-effects-and-computed.md), [08](./08-async-data-and-registries.md)

## 1. Problem

A cascading select is a `choice` / `multi-choice` field whose **available options depend on other values**: country → state → city, make → model → trim, category → product, and so on. Implementations need to handle:

1. **Narrowing:** the child's options change when the parent changes.
2. **Invalidation:** a child value that's no longer valid must be dealt with.
3. **Chains:** A → B → C, where invalidating B must reach C.
4. **Empty parent:** what the child shows before the parent has a value.
5. **Loading data:** the child's value is loaded together with its parent, and must *not* be invalidated just because the parent was loaded.
6. **Async races:** the parent changes again while the child's options are still loading.
7. **Arrays:** each row has its own cascade (`./country → ./state`).
8. **Multi-select parents:** the child's options combine the options for every selected parent value.
9. **Reverse fill (optional):** picking a city fills in its state and country.

The approach: **don't add a "cascading select" field type.** Cascading is what you get when any options source depends on other values. One mechanism handles static, mapped, and provider options, and the existing pieces do the rest: the dependency graph, write origins, effects, and request handling.

## 2. Contract

```ts
// Flat config keys on choice / multi-choice fields (01 §3.3)
interface ChoiceConfig {
  options?: OptionsSource;
  cascade?: CascadeConfig;
  autoSelectSingle?: boolean;          // default false
  onSelect?: Effect[];                 // reverse fill, §7
  resolveSelectedSourceId?: string;    // §4.1
  validateInOptions?: boolean;         // default true, §4.2
}

type OptionsSource = StaticSource | MapSource | ProviderSource;

interface StaticSource extends Extensible {
  source: 'static';
  items: SelectOption[];
  filter?: Expression;                 // §2.1
  disableWhen?: Expression;            // §2.1.1
}
interface MapSource extends Extensible {
  source: 'map';
  key: Expression | PathRef;           // which value picks the list
  map: Record<string, SelectOption[]>; // key value → options
  fallback?: SelectOption[];           // key matches nothing
  disableWhen?: Expression;
  groupBy?: 'parent';                  // §2.2.1, display only
}
interface ProviderSource extends ProviderOptions {   // ProviderOptions extends Extensible   // 08 §2
  source: 'provider';
  disableWhen?: Expression;
}

interface CascadeConfig {
  dependsOn?: PathRef[];                                    // static/map only: extra deps beyond inferred ones. Providers use options.dependsOn
  whenEmpty?: 'disable' | 'hide' | 'show-empty' | 'load';   // parent empty; default 'disable'
  onChange?: 'keep-if-valid' | 'clear' | 'reset' | 'keep';  // default 'keep-if-valid'
}
```

`SelectOption` arrays in the contract are still plain JSON. A bare array for `options` still loads and is normalized to `{ source: 'static', items }`.

### 2.1 Static + filter

Every option is tested with `filter`. `$option` (the option being tested) is added to the expression scope ([05 §3](./05-expressions.md#3-evaluation-scope)):

```json
{ "id": "state", "kind": "field", "fieldType": "choice",
  "options": { "source": "static",
    "items": [
      { "value": "CA", "label": "California", "meta": { "country": "US" } },
      { "value": "ON", "label": "Ontario",    "meta": { "country": "CA" } } ],
    "filter": { "lang": "json-logic", "expr": { "==": [{ "var": "$option.meta.country" }, { "var": "./country" }] } } } }
```

Dependencies are **inferred** from `filter` (`./country`), so `cascade.dependsOn` isn't needed here.

### 2.1.1 Disabling options

`filter` *removes* options. `disableWhen` keeps them visible but not selectable. It's evaluated per option with `$option` in scope, on any source type, and combined with the option's static `disabled` using OR:

```json
"disableWhen": { "lang": "json-logic", "expr": { ">": [{ "var": "$option.meta.minAge" }, { "var": "/age" }] } }
```

A currently selected value whose option becomes disabled is **kept** (disabling doesn't invalidate), but the `in-options` check ([§4.2](#42-built-in-in-options-validator)) flags it. Use `filter` instead if the value should be cleared.

### 2.2 Map

This is shorthand for the most common case: look options up by the parent's value.

```json
"options": { "source": "map", "key": "./make",
  "map": { "toyota": [{ "value": "corolla", "label": "Corolla" }],
           "honda":  [{ "value": "civic",   "label": "Civic" }] } }
```

`key` can be a `PathRef` or an Expression (for compound keys such as `make + "|" + year`). 
#### 2.2.1 Multi-select parents

For a `multi-choice` parent, the options are the **union** of `map[v]` for each selected `v`, in the parent's selection order.

**Keys from arrays.** Whenever the map `key` evaluates to an array (a `PathRef` to a `multi-choice`, or an expression returning an array), each element is looked up and the lists are merged. An expression that mixes a multi-select value into a string (for example `cat` over an array) produces one meaningless key; the compiler warns (`W_ARRAY_IN_KEY`) when a `cat`/string operator reads a `multi-choice` path. Keep expression keys over single-value parents, and key directly by the multi-select parent instead (the `hierarchy` macro does this, [15 §7.1](./15-macros.md#71-hierarchy)).

**Child values must be unique across the merged lists.** If two selected parents contribute the same child `value` (for example `springfield` under both `IL` and `MO`), the stored value couldn't say which one was meant. So instead of silently merging them:

- **Load time:** when the `key` is a plain `PathRef` to a `multi-choice` field, every child `value` must be unique across the **whole** map. A duplicate is `E_AMBIGUOUS_OPTION`, naming both map keys. (For expression keys, the engine can't know which keys combine, so the check happens at runtime.)
- **Runtime** (expression keys and provider sources): a collision in the merged list sets `status: 'error'` with `E_AMBIGUOUS_OPTION`, and the colliding options are left out of `items`. This is a contract bug, so it also logs a dev error.

Use globally unique values (`springfield-il`) for children of multi-select parents. The `hierarchy` macro enforces this at expansion ([15 §7.1](./15-macros.md#71-hierarchy)).

**Grouping (opt-in, display only).** With `groupBy: 'parent'`, each resolved option carries its parent:

```json
"options": { "source": "map", "key": "./states", "groupBy": "parent",
  "map": { "IL": [{ "value": "springfield-il", "label": "Springfield" }],
           "MO": [{ "value": "springfield-mo", "label": "Springfield" }] } }
```

```ts
interface ResolvedOption {
  value: Scalar; label: string; disabled: boolean; meta?: Json;
  group?: { value: Scalar; label: string };   // the parent option, label localized
}
```

- The parent is the option of the field named by `key` when it's a `PathRef`. With an expression key (as the `hierarchy` macro generates), it's the **immediate** parent level: the last `var` path in the key expression.
- `group` is set whenever `groupBy: 'parent'` is on, even with one parent selected, so the list doesn't change shape when a second parent is picked. Renderers that want sections only for several parents can check `new Set(items.map(o => o.group?.value)).size > 1`.
- Grouping never changes values, validation, or submission. Renderers without grouped lists (`optgroup`, sectioned pickers) ignore `group`.
- Provider sources can't infer the parent, so their result transformer sets `group` itself on each `SelectOption` if wanted (it receives `deps`, [08 §2](./08-async-data-and-registries.md#2-provider-options)).

### 2.3 Provider

```json
"options": { "source": "provider", "providerId": "geo.cities",
  "queryTransformerId": "geo.byRegion", "dependsOn": ["./country", "./state"] }
```

The query transformer receives `deps: { './country': 'US', './state': 'CA' }` ([08 §2](./08-async-data-and-registries.md#2-provider-options)). The engine can't see inside a transformer, so `options.dependsOn` is **required** (`cascade.dependsOn` isn't used for providers). Leaving it out is a compile error (`E_PROVIDER_DEPS_REQUIRED`). Set it to `[]` if the options really don't depend on anything.

One data source can serve a whole hierarchy (`geo.options` with a `level` param), so the registry doesn't grow by one entry per level.

## 3. How it runs

The available options are a **facet** of the field (`options`) in the dependency graph ([04 §5](./04-paths-and-dependency-graph.md#5-dependency-graph)):

```
./country.value ──► ./state.options ──► ./state.value (validity check) ──► ./city.options ──► …
```

When a dependency's value changes in a tick:
1. `options` becomes stale. For static and map sources it is recomputed synchronously in the same tick. For provider sources, the in-flight load is aborted, a new request generation starts, and `status` becomes `loading` ([08 §3](./08-async-data-and-registries.md#3-load-lifecycle)).
2. The **validity check** for the child's current value runs (§4). If the value changes, that is a normal write with origin `effect`, so the grandchild reacts in the same way. Chains need no special handling.
3. `whenEmpty` is checked (§5).

## 4. Invalidation

`onChange` only applies when the dependency change came from a write whose origin has `ORIGIN_POLICY[origin].invalidate === true` ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)); custom `x-` origins inherit the flag from the origin they extend. For every origin with `invalidate: false`, the child value is **trusted**: the options still reload, but the value isn't touched. This document never lists origins itself; the policy table is the single source. It prevents "loading the form cleared every dependent select".

A query transformer that returns `null` (skip, [08 §2](./08-async-data-and-registries.md#2-provider-options)) is not an options change for this purpose: `status` is `'idle'`, the value is left alone, and the validity check waits until a real query returns options.

| `onChange` | Behavior |
|------------|----------|
| `keep-if-valid` (default) | Keep the value if it's still among the available options, otherwise clear it. For `multi-choice`, drop only the values that are no longer valid |
| `clear` | Always clear (to `null` / `[]`) |
| `reset` | Go back to the default value. If the default isn't valid, clear |
| `keep` | Never touch the value. A `required`/`in-options` validation error is left for the user to resolve |

### 4.1 Deciding "valid"

- **static / map:** synchronous. Is the value among the newly computed options?
- **provider:** possibly asynchronous. While options are loading, the value is kept and the field is marked `meta.optionsPending`. Once they arrive:
  - If the response is complete (`nextState === null` on the first page), membership decides.
  - If the results are paginated or search-based, a missing value doesn't mean it's invalid. The engine calls the field's `resolveSelectedSourceId` ([08 §2](./08-async-data-and-registries.md#2-provider-options)) with the new deps. The value is valid if that source resolves it. If neither check is available, the engine falls back to `clear` and warns at compile time (`W_CASCADE_UNVERIFIABLE`).
- If the parent changes again before this resolves, the load generation check throws away the stale answer. Only the latest parent value can invalidate the child.

### 4.2 Built-in `in-options` validator

Every field with an options source gets an implicit `in-options` rule. It reuses the validity check and also fails for values whose option is disabled ([§2.1.1](#211-disabling-options)), so a value set by host code, or kept by `keep`, gets a visible error instead of being submitted silently. It can be turned off with `validateInOptions: false`, for example for a combobox renderer that accepts free entry. (For an "Other, please specify" pattern, prefer a separate text field, [16 §3](./16-cookbook.md#3-other-option-with-free-text).)

Submit waits for any `optionsPending` validity checks, the same way it waits for async validation ([06 §4](./06-validation.md#4-async-validation)).

## 5. Empty parent

A dependency counts as "empty" if it is `null`, `undefined`, `''`, or `[]`.

| `whenEmpty` | Behavior |
|-------------|----------|
| `disable` (default) | The `disabled` facet is true, options are `[]`, and no load happens. Under the default `whenDisabled: 'exclude'` policy ([07 §4](./07-effects-and-computed.md#4-hidden-and-disabled-field-policy)), the value isn't submitted |
| `hide` | The `visible` facet is false. The field's `whenHidden` policy applies (example below) |
| `show-empty` | Enabled, with an empty list and a "Select {parent} first" placeholder |
| `load` | Treat empty as a real value and load/compute anyway (for example "all cities" when no state is picked) |

These combine with the author's own `conditions` using AND: a field is visible only if both say so, and disabled if either does.

**Value of the child when the parent becomes empty.** The options are `[]` (except under `load`), and `onChange` is applied against that empty list, but only if the write that emptied the parent has an invalidating origin (§4). So with the default `keep-if-valid`, clearing the country by hand clears the state; a `load` or `restore` that leaves the country empty leaves a loaded state alone (it then fails `in-options` if visible and enabled). Under `disable` and `hide`, the field's `whenDisabled` / `whenHidden` policy decides what is submitted.

```json
{ "id": "vatNumber", "kind": "field", "fieldType": "choice",
  "options": { "source": "map", "key": "./country", "map": { "DE": [ … ], "FR": [ … ] } },
  "cascade": { "whenEmpty": "hide", "onChange": "clear" },
  "whenHidden": "exclude" }
```

No country: the VAT field isn't shown or submitted. Picking a country shows it; changing the country clears it.

## 6. Auto-select

With `autoSelectSingle: true`, if the options settle on exactly one item and the field is empty, that item is written with origin `effect`. It is skipped when:
- the options settled because of a write whose origin has `invalidate: false` (§4), for example during loading or `restore`: those values are trusted as they are;
- the field has a value in the `loaded` layer (in edit mode, an empty saved value stays empty until the user changes a parent);
- the user deliberately cleared the field in this session, which is tracked in `meta.userCleared`.

## 7. Reverse fill

Selecting a child can fill in its ancestors through `onSelect` effects. The selected option is available as `$selected`:

```json
"onSelect": [
  { "target": "./state",   "action": "set", "value": { "lang": "json-logic", "expr": { "var": "$selected.meta.state" } } },
  { "target": "./country", "action": "set", "value": { "lang": "json-logic", "expr": { "var": "$selected.meta.country" } } } ]
```

These writes have origin `effect`, so they would normally trigger the parents' cascades and clear the very child that was just selected. To prevent this, the user's selection and all writes of one `onSelect` run form a **batch** tagged with a unique batch id and the selecting child's stable row-key path. `onSelect` is event-only: cascade, effect, auto-select, load and replay writes do not invoke it. Invalidation runs once **after all batch writes**, and skips only the selecting child and targets explicitly written by that batch (so setting `./country` cannot clear the `./state` just set). Other dependents invalidate normally. The compiler accepts the reverse-fill cycle only under the narrow certificate in [04 §5.2](./04-paths-and-dependency-graph.md#52-cycle-detection); other cycles remain errors, and a target outside the transitive options dependencies is `E_ON_SELECT_TARGET`.

Options still reload and `in-options` still validates the selection against the resulting parents; metadata isn't proof of membership. For async loads, the suppression token travels only with loads/selected-resolver checks caused by this batch's exact dependency snapshot and request generation. It expires when those checks settle or a later write supersedes them. No unrelated batch or later parent change inherits it, and a protected but invalid selection receives a validation error rather than being silently trusted or submitted.

## 8. Arrays and caching

- Inside a row, `./country` resolves to that row ([04 §2](./04-paths-and-dependency-graph.md#2-path-grammar)). Option state is kept per **row key path**, so reordering rows doesn't mix up their lists.
- Static and map results are memoized per dependency values, so 50 rows with `country = US` share one computed list.
- Provider results are cached by provider id plus the query transformer's output, with `cache.key` deciding whether deps and search text also vary the key ([08 §3](./08-async-data-and-registries.md#3-load-lifecycle)). Identical in-flight loads from different rows share one call: it is reference-counted, aborted only when no row still waits for it, and each row applies the result only if its own generation still matches.

## 9. Renderer view

The `options` handle on a `FieldController` ([11 §2](./11-rendering-and-adapters.md#2-headless-field-api)) exposes:

```ts
interface OptionsState {
  items: ResolvedOption[];            // labels already localized
  status: 'idle' | 'loading' | 'ready' | 'error';
  blockedBy?: PathRef[];              // empty dependencies (for "Select country first")
  pendingValidity: boolean;
  hasMore: boolean;
  loadMore(): void; search(q: string): void; retry(): void;
  open(): void; close(): void;        // trigger: 'open'
}
```

With `trigger: 'open'`, the renderer calls `open()` when the list is shown and `close()` when it's hidden. Calls are **reference-counted** (a combobox and a "show all" dialog may both be open): the first `open()` starts a load if needed, and an in-flight load is aborted only when the count returns to zero (or the field unmounts). Finished results stay cached.

Renderers don't need any cascade-specific code. A plain select renderer works for every level.

## 10. Worked example: country → state → city, in an address array

```json
{ "id": "addresses", "kind": "array", "item": { "id": "address", "kind": "group", "children": [
  { "id": "country", "kind": "field", "fieldType": "choice",
    "options": { "source": "provider", "providerId": "geo.options",
                 "queryTransformerId": "geo.level", "dependsOn": [] } },
  { "id": "state", "kind": "field", "fieldType": "choice",
    "options": { "source": "provider", "providerId": "geo.options",
                 "queryTransformerId": "geo.level", "dependsOn": ["./country"] },
    "autoSelectSingle": true },
  { "id": "city", "kind": "field", "fieldType": "choice",
    "options": { "source": "provider", "providerId": "geo.options",
                 "queryTransformerId": "geo.level", "trigger": "query",
                 "dependsOn": ["./country", "./state"] },
    "cascade": { "whenEmpty": "show-empty" },
    "resolveSelectedSourceId": "geo.resolve",
    "ui": { "variant": "combobox" } }
] } }
```

- The user changes row 2's country: row 2's state options reload. The state isn't in the new list, so it's cleared (origin `effect`). That clears row 2's city in the same tick. Row 1 is unaffected.
- The form loads with existing addresses: the options load for each row and each level, but no values are cleared.
- The user switches the country again while states are loading: the first load is aborted, and only the second can invalidate.

## 11. Open questions

- **Q10 (resolved: C).** Whole hierarchies are written with the `hierarchy` macro ([15 §7.1](./15-macros.md#71-hierarchy)), which expands into flat per-level `map` sources keyed by the joined ancestor values. Submitted values stay plain. Options considered:
  - **A, flat per level (current):** each level has its own validation, `whenEmpty`, and cascade handling. Child keys must be unique across parents, or compound (`"US|IL"`).
  - **B, a core `tree` source:** the hierarchy lives in one place, and duplicate names under different parents are natural. But it's more contract surface, needs a depth check, and bloats contracts.
  - **C, a `hierarchy` macro ([15](./15-macros.md)):** expands a tree into A with compound keys.

- **Q11 (resolved: detect + opt-in grouping).** Colliding child values under a multi-select parent are an error (`E_AMBIGUOUS_OPTION`), and `groupBy: 'parent'` adds display-only `group` info ([§2.2.1](#221-multi-select-parents)). Qualified values (`{ parent, value }`) were deferred until a real form needs the same child under several selected parents.
