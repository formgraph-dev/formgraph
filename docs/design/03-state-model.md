# 03 · State Model

[← Index](./README.md) · Addresses review items S1, S2, S3, S5, S6

## 1. Layers and slices

The `FormStore` snapshot is divided into **slices**. Three of them are value **layers**: they stack, and a path's value comes from the topmost layer that has it (§3). The other slices hold engine state and don't take part in precedence.

| Slice | Layer? | Written by (origin, [07 §2.1](./07-effects-and-computed.md#21-origin-policy)) | Contents |
|-------|--------|------------|----------|
| `default` | bottom layer | `default` | Defaults from the contract (`defaultValue`, array `defaultValue`, expression defaults) |
| `loaded` | middle layer | origins with `layer: 'loaded'` (`load`, `external`) | Values given to the form from outside, after the inbound transform: a record from an `entitySource`, a prefill, values passed to `reload({ entity })`, or `applyExternal` changes. Not necessarily persisted |
| `current` | top layer | origins with `layer: 'current'` (`user`, `host`, `effect`, `reset`, `history`, `restore`, custom `x-` origins) | Explicit overrides only (sparse) |
| `meta` | no | engine | Engine bookkeeping, see below |
| `notices` | no | validation pipeline and reporters | Per-path notice lists by source (every severity, 06 §5) |
| `form` | no | engine | Form-level status |

`meta` holds these sub-structures. All per-path entries are keyed by row key paths ([§5](#5-field-meta-and-form-status)):

| Sub-structure | Contents | Undoable ([§6.2](#62-undo--redo-opt-in)) |
|---------------|----------|----------|
| `meta.rows[arrayPath]` | Row keys of each array ([§3.2](#32-atomic-array-writes)) | yes |
| `meta.baselineRows[arrayRowKeyPath]` | Baseline-position → row-key map and unresolved-old-row keys, including nested arrays; independent of current order/inserts | no (follows the baseline) |
| `meta.stash` | Dynamic-group orphans kept by `orphanPolicy: 'stash'` ([08 §5](./08-async-data-and-registries.md#5-dynamic-sub-forms)) | yes |
| `meta.conflict[p]` | Open conflicts ([09 §7.2](./09-loading-and-submission.md#72-resolving)) | no |
| `meta.touched[p]`, `meta.visited[p]` | Interaction flags ([§5](#5-field-meta-and-form-status)) | no |
| `meta.field[p]` | `validating`, `asyncStatus` ([§5](#5-field-meta-and-form-status)) | no |

Non-overridable computed values aren't stored in any slice; they're evaluated during resolve ([07 §3](./07-effects-and-computed.md#3-computed-values)).

## 2. Store topology

The engine owns its store (`FormStore`). The core has **no runtime dependency** on a state library. See [§2.4](#24-why-not-tanstack-store) for why.

```
                         FormStore
 ┌──────────────────────────────────────────────────────────────┐
 │ slices (immutable snapshot)  layers: default · loaded ·       │
 │                              current; meta · notices · form   │
 │ commit entries (per tx)      [{ slice, path, prev, next,     │
 │                                 origin }], dropped after hooks │
 │ resolved cache               values / facets, by path         │
 │ subscriber index             path → Set<listener>             │
 └──────────────────────────────────────────────────────────────┘
        ▲ transact(origin, fn)                 │ notify(changed paths)
        │                                      ▼
   engine commands                     Readable<T> handles → adapters
```

### 2.1 Transactions

All writes go through one entry point:

```ts
store.transact(origin, (tx) => {
  tx.set('current', path, value);
  tx.delete('current', path);
  tx.meta(rowKeyPath, patch);
  tx.notices(rowKeyPath, source, list);
});
```

- A transaction builds a new immutable snapshot with structural sharing ([§6](#6-performance-contract)) and builds its commit entries `{ slice, path, prev, next, origin }`. They are passed to commit hooks ([§2.5](#25-commit-hooks)) and plugins, then dropped; the store keeps no log.
- Transactions nest. Only the outermost commits, so a whole scheduler tick ([04 §6](./04-paths-and-dependency-graph.md#6-scheduler)), including cascades, is **one commit and one notification pass**.
- If `fn` throws, the snapshot is discarded and nothing is notified (atomic rollback).
- Writes made by listeners during notification are queued as a new transaction after the current pass. They never interleave with it, so subscribers can't see half-applied state.

### 2.2 Reading: `Readable<T>`

```ts
interface Readable<T> {
  get(): T;
  subscribe(listener: (value: T) => void): () => void;
}

store.select(path, facet?)   // Readable for one path: 'value' | 'visible' | 'notices' | 'meta' | …
store.selectForm()           // Readable<FormState>, exposed as engine.form
store.snapshot()             // whole immutable snapshot (devtools, tests)
```

- `Readable` handles are cached by `(path, facet)`, so the same handle is returned each time.
- `get()` also reports the read to the active read observer, if any (§2.6). That's how adapters connect handles to their framework's own tracking. Outside one it's a plain read.
- `get()` is lazy and memoized against the snapshot version. Values and facets are resolved on first read after a change, not on write.
- The shape fits React's `useSyncExternalStore` and Svelte's store contract. It also covers the `get`/`subscribe` that TanStack's `Readable` type has, though whether the `@tanstack/*-store` hooks accept a foreign object isn't verified ([11 §3](./11-rendering-and-adapters.md#3-framework-bindings)).

### 2.3 Notifying by path

After a commit, the store has the set of changed data paths (from the commit entries) plus the set of facets the scheduler re-evaluated. It notifies only:
1. subscribers of exactly those paths,
2. subscribers of **ancestor** paths (a write to `a.b.c` changes `a.b` and `a`),
3. subscribers of **descendant** paths when a whole subtree was replaced (for example array `replace`, reloading). This is only done when the descendant's value identity actually changed.

A listener runs only if its value changed by `Object.is` (or by the equality the handle was created with). The cost per commit is O(changed paths × path depth), not O(all subscribers).

### 2.4 Why not TanStack Store

The v4 handoff used `@tanstack/store`. We reviewed v0.11.2 (current at the time of writing):

- It is pre-1.0 and was rewritten onto `alien-signals` between 0.7 and 0.11. `Derived`, which the earlier draft of this doc relied on, no longer exists.
- Its subscriptions work by running selectors. Every subscriber's selector runs on every write (O(all fields) per keystroke). Our commit entries already say which paths changed, so notifying by path is cheaper.
- Write origins, dependency-graph ordering, cycle detection when the schema is compiled, row keys and wildcard dependencies, cancellable async work, and per-commit entries aren't provided. We would build all of them ourselves anyway.

What it would have given us is batching and framework bindings. Batching is the transactions in §2.1, and our own bindings are about 30 lines each ([11 §3](./11-rendering-and-adapters.md#3-framework-bindings)). Tracking is left to each framework through a read hook (§2.6), so the core needs no signal graph.

### 2.5 Commit hooks

Borrowed from JSON Forms middleware, but **deliberately narrower** ([14 §3](./14-prior-art.md#json-forms)). JSON Forms middleware replaces the reducer and can rewrite any state. Ours is one hook type around commits that can observe or veto, but can't rewrite. Plugins use it ([§8.1](#81-plugin-api)), and hosts attach it at runtime for audit logging, autosave, analytics, and dev assertions:

```ts
interface CommitHooks {
  onBeforeCommit?(tx: PendingTx): void | 'abort';   // inspect, or veto with 'abort' (rollback)
  onCommit?(tx: CommittedTx): void;                 // after notify; read-only
}
const off = engine.use(hooks);                      // returns detach
```

- `PendingTx` / `CommittedTx` carry `{ txId, origin, entries }`, entries as in [§2.1](#21-transactions). Hooks **can't change** entries. To change values, use effects, which are visible to the dependency graph.
- Entries live only for the commit. A hook that needs history (devtools, history plugin) copies what it needs in `onCommit`.
- Hooks run in registration order, plugins first. An exception in `onBeforeCommit` aborts the commit and is reported. An exception in `onCommit` is logged and doesn't affect other hooks.
- `onCommit` can start new transactions (for example autosave status). They are queued like listener writes ([§2.1](#21-transactions)).

### 2.6 Tracking reads for framework adapters

The core doesn't ship its own signal system. Fine-grained frameworks (Solid, Vue, Svelte 5, Angular) already have tracking, computed values and effects; a second system next to theirs would only add concepts. Instead, `get()` reports each read to an observer, and **adapters** plug in their framework's own tracking:

```ts
// @formgraph/engine/interop — for adapter authors, not app code
type ReadObserver = (r: Readable<unknown>) => void;
function observeReads<T>(observer: ReadObserver, fn: () => T): T;   // runs fn with observer active; a stack, so nesting and two frameworks on one page are safe
function untracked<T>(fn: () => T): T;                               // runs fn with no observer
```

- **Inside the core:** `get()` calls the active observer (if any), then returns as before. About 20 lines; nothing else changes. `subscribe` and path notification (§2.3) are untouched.
- **Fine-grained adapters** install an observer, once per provider/owner, that maps each handle to a cached native signal (`WeakMap<Readable, NativeSignal>`, fed by `subscribe`) and reads it. The framework then tracks that read itself, so app code uses only native primitives:

  ```ts
  const total = createMemo(() => engine.field('qty').value.get() * engine.field('price').value.get());   // Solid
  const total = computed(() => engine.field('qty').value.get() * engine.field('price').value.get());     // Vue
  ```

  Batching, glitch-freedom, cleanup and lifetime are the framework's, and behave the way its users expect. Notifications only fire after a commit, so a native computed never sees a half-applied transaction.
- **React** has no native tracking. Its adapter's `useTracked(fn)` runs `fn` inside `observeReads`, subscribes to the handles it read (re-collected on every run, so dependencies can change), and feeds the result to `useSyncExternalStore` ([11 §3](./11-rendering-and-adapters.md#3-framework-bindings)).
- **Async reads.** Reads after an `await` (or in a later callback) aren't observed, as with every framework's own tracking: by then no observer is active, and the read looks like any deliberate untracked read.
  - `observeReads` checks `fn`'s result in development: a thenable means reads after the first `await` were lost, reported as `W_ASYNC_TRACKED` (one `typeof` check, stripped from production builds).
  - `useTracked`, `derived` and `watch` reject async functions in their types (`fn: () => T` with `T` not a `PromiseLike`).
  - Framework-owned computed values (`createMemo`, Vue `computed`, …) are called by the framework, so their return values aren't seen; their own rules and warnings apply.
  - The real fix is carrying the observer across `await`, which needs TC39 `AsyncContext` (stage 2, not shipped). Node's `AsyncLocalStorage` isn't portable (D24). If `AsyncContext` ships, the observer stack moves onto it without an API change.
- **Member handles.** Read member handles (`controller.value`, `controller.dirty`, or `select`) rather than `controller.state`, so a computation only re-runs for what it uses.
- **Batching writes** isn't reactive; it's a transaction: `engine.batch(fn)` turns several host writes into one commit (§2.1).
- **No framework.** Plain scripts, tests and plugins use `subscribe`. An optional subpath, `@formgraph/engine/reactive`, offers `derived(fn)` (a lazily recomputed `Readable`) and `watch(fn)` built on the same hook. It's tree-shaken and not counted in the core budget. Writing inside a `derived` throws `E_WRITE_IN_DERIVED` in development.
- **Not part of the contract graph.** Conditions, effects and computed fields stay in the compiled, cycle-checked graph ([04](./04-paths-and-dependency-graph.md)). Tracked code lives in the host; to feed host data into expressions, use `setContext` or registered functions.
- **Isolation rule.** `Readable` (`get`, `subscribe`, the read observer) is implemented in **one core module**. Nothing else in the core, plugins or adapters depends on how notification works internally; they only use `Readable` and `observeReads`. This keeps a change of reactive backend local.

#### Moving to native Signals

The [TC39 Signals proposal](https://github.com/tc39/proposal-signals) is at Stage 1 (since April 2024) and its API is still changing, so nothing depends on it yet. **This is a note only: nothing here is planned or built, including no `@formgraph/signals` package or polyfill support.** It records how a move would work if the proposal ships, so the isolation rule above keeps it cheap. Neither step changes app code:

1. **Adapter bridge (no core change).** A generic observer (for example a future `@formgraph/signals`) keeps one `Signal.State` per handle, fed by `subscribe`, and reads it, so any native `Signal.Computed` tracks our handles. One bridge serves every framework built on native signals:

   ```ts
   const states = new WeakMap<Readable<unknown>, Signal.State<unknown>>();
   observeReads((r) => {
     let s = states.get(r);
     if (!s) { s = new Signal.State(r.get()); r.subscribe(v => s!.set(v)); states.set(r, s); }
     s.get();
   }, fn);
   ```

2. **Native backing in the core (feature-detected).** When `globalThis.Signal` exists, the `Readable` module backs each handle with native signals; otherwise it keeps the current implementation (D24: Hermes and older engines will lag).

   ```ts
   const version = new Signal.State(0);                                          // bumped on commit for changed paths (§2.3)
   const value = new Signal.Computed(() => (version.get(), resolve(path)), { equals });   // stays lazy (§2.2)
   ```

   - Values stay lazy: only version counters change on commit, the same set path notification already computes.
   - Native watchers fire synchronously and may not read or write in `notify`; we already notify only after a commit.
   - `equals` maps to the `Signal.Computed` option. `subscribe` stays, built on `Signal.subtle.Watcher`, for React and plain code. `observeReads` stays for frameworks not on native signals.

- **Risks, if it's ever done.** API churn while the proposal is at Stage 1 (only the bridge and the `Readable` module touch it); two backends in the core until all target engines ship it (both tested under Node and Hermes, [12](./12-open-questions-and-roadmap.md)); async reads remain a separate problem (`AsyncContext`, Stage 2).

## 3. Precedence by presence, not nullishness

For any path `p`, the value is taken from the **topmost source that contains the key**. Computed values ([07 §3](./07-effects-and-computed.md#3-computed-values)) take part as virtual layers:

```
values(p) = isComputed(p) && !overridable(p) ? computed(p)     // replaces all layers
          : has(current, p)                  ? current[p]
          : isComputed(p)                    ? computed(p)     // overridable: between current and loaded
          : has(loaded, p)                   ? loaded[p]
          : default[p]
```

`has` uses `Object.prototype.hasOwnProperty.call(object, key)` at each segment along the path (ES2020-compatible). Because of this:
- The user clears a loaded value: `current.email = ''` (or `null`). The cleared value wins.
- An explicit `null` in `loaded` hides the default. An **absent** key falls through to `default`, unless the load config says `missing: 'empty'` ([09 §2.1](./09-loading-and-submission.md#21-missing-vs-null)).
- **`undefined` is never stored** in `current` (it can't survive a JSON snapshot, [09 §8](./09-loading-and-submission.md#8-snapshots-drafts)). Clearing stores the field type's empty value (`null`, `''`, `false`, `[]`), the same as the `clear` action ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)). `setValue(p, undefined)` is `E_UNDEFINED_VALUE`. To *revert* to the lower layer, call `engine.revert(path)`: delete the key outside arrays; reconstruct only the target inside arrays (§3.2).

### 3.1 Resolution granularity

Resolution works per key, recursively, for **plain objects** (groups with `dataScope`), and **atomically for arrays**:

- Object at `p`: merge key by key, using presence precedence for each child.
- Array at `p`: take the whole array from the topmost layer that has `p`. No index-wise merging.
- **Field value at `p`: always atomic**, whatever its shape. A custom field whose value is an object or array (01 §3.4) is taken whole from the topmost layer, like an array. It is never stitched together from several layers. Only nodes (groups with `dataScope`, array rows) merge per key.

### 3.2 Atomic array writes

Writing `addresses[1].city` when `current` has no `addresses`:
1. Clone `values.addresses` (the resolved array) shallowly, and clone row 1 along the path.
2. Apply the leaf write.
3. Store the whole array in `current.addresses`.

From then on, `addresses` comes entirely from `current`. Reloading interacts with this through the policy in [09 §3](./09-loading-and-submission.md#3-reload-policy).

Atomic storage does not widen the target of a command. `revert` or an effect's `reset` on an array descendant restores only that target in a cloned resolved array, preserving other rows and cells. Baseline rows are matched through `meta.baselineRows`, never by current index; a genuinely inserted row uses field defaults. A descendant operation **keeps** the enclosing `current` array override even if its value now equals the baseline: equality controls dirty state, not presence. Only an explicit whole-array `revert`, `reset`/reload policies that clear overrides, or conflict resolution can remove it. Thus `preserve-edits` keeps an explicit same-valued array just as it keeps an explicit same-valued scalar ([07 §2](./07-effects-and-computed.md#2-write-origins-and-effect-triggers)).

`meta.baselineRows` maps each baseline array's positions to stable keys; nested array instances are addressed by enclosing row keys, not indexes:

```ts
interface BaselineRowKeys {
  keys: RowKey[];                  // positions in the current baseline array
  unresolved?: RowKey[];           // formerly baseline rows that cannot be matched after a baseline change
}
```

It is created with defaults/loading, updated when the baseline changes, and is not reordered or extended when current rows move or are inserted. Drafts and SSR persist it alongside current row keys ([09 §8](./09-loading-and-submission.md#8-snapshots-drafts), [09 §9](./09-loading-and-submission.md#9-server-rendering)). On an unchanged baseline, restore reinstates saved associations, so reordered old rows and new rows remain distinguishable without `identityKey`. A changed baseline without reliable identity never uses guessed indexes: old baseline keys move to `unresolved`, and descendant revert rejects with `E_ROW_BASELINE_UNKNOWN`. A key in neither list is a genuinely inserted row and uses defaults. Persisting unresolved provenance prevents a second draft save/restore from misclassifying an old row as new. Accept the external whole array or explicitly revert the whole array to recover known associations. Choosing `local` or `keep-draft` keeps unresolved markers: dismissing a conflict cannot invent row identity.

Row keys (see [04 §4](./04-paths-and-dependency-graph.md#4-array-row-identity)) are stored in a parallel structure, `meta.rows[arrayPath]: RowKey[]`, never inside the user data.

## 4. Dirty semantics

`dirty(p) = !deepEqual(values(p), baseline(p))`, where `baseline(p)` is the §3 resolve order **without `current`**:

```
baseline(p) = isComputed(p) ? computed(p)        // overridable: the computed value is the baseline
            : has(loaded, p) ? loaded[p]
            : default[p]
```

- For a plain object, the baseline is built with the same per-key presence merge as values ([§3.1](#31-resolution-granularity)), so an object is dirty exactly when one of its children is. Arrays compare atomically.
- A `whenHidden: 'clear'` write ([07 §4](./07-effects-and-computed.md#4-hidden-and-disabled-field-policy)) is a real change: the field becomes dirty and is sent in `patch` mode, because the server value will be cleared. Use `keep` or `exclude` to avoid it.
- Non-overridable computed fields are **never dirty** and are left out of `patch` payloads ([09 §4](./09-loading-and-submission.md#serialize)).

**Cost.** No deep compare runs per keystroke:
- Each stored leaf of `current` ([§3.1](#31-resolution-granularity)) gets a dirty bit on write: equal by reference to its baseline → clean without further work; otherwise `deepEqual` runs on **that leaf only** (arrays: once per atomic array write). A field whose type defines `equals` ([01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions)) uses it instead of `deepEqual`, here and wherever the engine compares that field's values (history coalescing, conflict detection, async result cache).
- Each ancestor keeps a **dirty count** (number of dirty stored leaves below it), adjusted by ±1 along the path when a leaf's bit flips. `dirty(p)` for a group or the form is `count(p) > 0`: O(1). "Form is dirty" is the root count.
- **Rows.** Arrays are atomic leaves, so they carry one bit, not per-row counts. A row's dirty state is **derived**: its value compared with the baseline row matched by row key (or `identityKey`, [04 §4](./04-paths-and-dependency-graph.md#4-array-row-identity)). It is computed only while subscribed (`ScopeController.dirty`), O(row size) per change to that row, and cached by reference, so unchanged rows (shared by structural sharing) cost nothing.
- A baseline change (`load`, `external`, default re-evaluation, overridable computed) re-checks only the stored leaves of `current` under the changed paths.

## 5. Field meta and form status

```ts
interface FieldMeta {
  touched: boolean;         // blurred at least once (handlers.onBlur()/touch() only; no write of any origin sets it)
  visited: boolean;         // focused at least once (handlers.onFocus())
  validating: number;       // count of in-flight async validations
  asyncStatus?: 'idle' | 'loading' | 'error';   // for option sources
}

interface FormState {                // engine.form (18); select a key or a function to narrow re-renders
  phase: 'loading' | 'ready' | 'submitting' | 'submitted' | 'error';
  submitCount: number;
  lastSubmitError?: unknown;
  valid: boolean;                     // no blocking error on any eligible node
  validating: boolean;                // async validation or field-type work in flight
  dirty: boolean;                     // root dirty count > 0 (§4)
  notices: Notice[];                  // form-level only: notices that map to no path (09 §5)
  severity: Severity | null;          // strongest across the whole form (form-level and per-path)
  counts: { error: number; warning: number; info: number; custom: Record<`x-${string}`, number> };   // error/warning/info by base severity (06 §7.6); whole form, eligible nodes; kept like dirty counts (§4), O(1) to read
  gates: Record<string, { policy: 'allow' | 'confirm' | 'block'; count: number; confirmed: boolean }>;   // per warning gate (06 §7.3); gates with notices or a policy
  conflictCount: number;              // 09 §7
  locale: LocaleTag;                  // active locale; switches atomically (10 §3.3)
  localeStatus: 'loading' | 'ready' | 'error';   // bundle loading for setLocale (10 §3.3)
  pendingAssets: number;
}

type Notices = Record<RowKeyPath, Notice[]>;   // see 06 §5
```

Meta and notices are keyed by **row key paths** (for example `addresses[#k3].city`), so they follow rows when rows move.

## 6. Performance contract

- **Structural sharing:** a write to `a.b.c` creates new objects only along `a → a.b → a.b.c`. Every other reference stays the same.
- **Incremental resolve:** the resolved cache is invalidated only for changed paths and their ancestors. Unchanged subtrees keep their previous identity.
- **Path-indexed notification:** only subscribers of affected paths are checked ([§2.3](#23-notifying-by-path)). A field re-renders only when its own value or facet changes.
- **Lazy evaluation:** a facet with no subscribers and no dependents in the current tick isn't computed.
- **Combined field state:** each field has one `Readable<FieldState>` ([11 §2](./11-rendering-and-adapters.md#2-headless-field-api)); adapters subscribe once per field and select from it. The per-facet handles are derived from it.
- **Dirty in O(depth):** per-leaf bits plus per-subtree counts ([§4](#4-dirty-semantics)).
- **Expressions over `[*]`** (including `sum`/`count`/`min`/`max`) cost O(rows) per evaluation ([07 §3.2](./07-effects-and-computed.md#32-aggregates)).
- **Lazy compile:** see [§6.3](#63-lazy-compile).
- **Budget:** one keystroke costs O(path depth) plus O(direct dependents) evaluations plus O(affected subscribers). There is a benchmark target in [12](./12-open-questions-and-roadmap.md).

## 6.1 Commit entries

Entries are built per transaction and dropped after hooks run ([§2.5](#25-commit-hooks)); nothing is retained in production. Devtools keeps its own capped buffer through `onCommit` ([11 §7](./11-rendering-and-adapters.md#7-devtools)). Undo/redo keeps its own stack (§6.2).

## 6.2 Undo / redo (opt-in)

Decided in Q5 ([12](./12-open-questions-and-roadmap.md)). Provided by the `history` plugin ([§8](#8-plugins)), which the default entry installs. Off until configured:

```ts
createFormEngine({ …, plugins: { history: { limit: 100, coalesceMs: 1000 } } });

engine.history.undo(); engine.history.redo();
engine.history.canUndo / canRedo        // Readable<boolean>
engine.history.clear();
```

**What one step is:**
- One tick whose origin has `history: 'record'` (`user`, `host`, [07 §2.1](./07-effects-and-computed.md#21-origin-policy)), including everything it cascaded in the same tick (`history: 'merge'`): effects, cascade clears ([13 §4](./13-option-sources-and-cascading.md#4-invalidation)), `onSelect` reverse fills, `clear` from `whenHidden`, and array row-key changes. Undoing a country change also restores the state it cleared.
- **Typing is coalesced:** consecutive user ticks that write only the same single path, within `coalesceMs` of each other, merge into one step. Blur, or a write to another path, ends the step.
- Async results that arrive later (options, async validation, `setContext`) are **not** part of any step. They're recomputed from the restored values.

**What's recorded:** only `current`, `meta.rows`, and `meta.stash` (dynamic group orphans). Touched/visited state, validation messages, and external errors aren't undone. Validation reruns on the restored values.

**What isn't undoable:** the `history` field of the origin policy ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)) decides. `clear` origins **clear** both stacks, because the baseline they'd undo against has changed. `clear-overlap` origins clear them only if a written path appears in a recorded step ([09 §7.3](./09-loading-and-submission.md#73-interaction-with-undo)). `skip` origins are simply not recorded. A dynamic-group schema switch also clears both stacks ([08 §5](./08-async-data-and-registries.md#5-dynamic-sub-forms)).

**How it's applied:** undo writes the recorded `prev` values in one transaction with origin `history`. Effects and cascade invalidation **don't run** for `history` writes, because the restored state already contains the cascaded results. Computed values, conditions, and validation are re-evaluated as usual. In-flight async work for the undone values is aborted through its normal generation check. A new user step after an undo clears the redo stack.

## 6.3 Lazy compile

Compiling splits into a cheap eager part and a lazy part:

| Eager (at load) | Lazy (on first access) |
|-----------------|------------------------|
| Node tree, paths, data keys | Field controllers and their `Readable`s |
| Dependency graph edges and cycle detection ([04 §5](./04-paths-and-dependency-graph.md#5-dependency-graph)) | Compiled validators (rule closures, message templates) |
| Default values, including expression defaults | Option state (no provider call until the field mounts or a dependent needs it) |
| Computed values, conditions and effects that something depends on; every `hidden`/`disabled` condition (including inherited ones) | Renderer tester results ([11 §4.1](./11-rendering-and-adapters.md#41-ranked-testers)) |

"First access" is a controller lookup, a subscription, or a read by a dependent. Nodes in unmounted steps, collapsed sections and dynamic sub-schemas pay only the eager part until then. The graph has to be complete up front, because a write anywhere must find all dependents, including unmounted ones; so do defaults, since they decide values and dirty. Hidden/disabled state is always eager because it decides eligibility, serialization and `firstInvalid()`. `submit()`, `validate()` and `serialize()` first force-compile whatever is missing for eligible fields (validators, option state), start option loads that gate validity ([13 §4.2](./13-option-sources-and-cascading.md#42-built-in-in-options-validator)) and async validators with no settled result for the current value ([06 §4](./06-validation.md#4-async-validation)); with `awaitAsync`, submit awaits them. `firstInvalid()` reads compiled validators, so it relies on this force-compile.

Engine-initiated validation also creates each eligible field's controller and installs its registered controller extensions **before** invoking component validation hooks. A lifecycle plugin can therefore install a fail-closed guard even when no renderer has mounted; it must not rely on a renderer effect to register the first guard ([16 §22](./16-cookbook.md#22-another-engine-as-a-custom-field)).

## 7. Engine write API (core)

```ts
engine.setValue(path, value, { runEffects = true, origin? })   // origin: registered x- origins only (07 §2.1)
engine.revert(path)                 // restore target's baseline; array descendants preserve other cells (§3.2)
engine.reset({ keepLoaded = true })   // see below
engine.array(path)                  // ArrayHandle, see below
engine.touch(path, { validate? })   // mark touched (shows messages); container paths touch every descendant; validate (default true)
                                    // runs blur-triggered rules that have no result for the current value. Interaction is reported by renderers (11 §2)
engine.revision                                // Readable<number>; see below
engine.exportState()                           // server rendering (09 §9)
engine.firstInvalid(opts?)                     // first failing node in form order (11 §8)
engine.getValues(path?)             // resolved values (all layers, computed included), no policies or wire mapping; serialize() is for payloads
```

Full signatures: [18](./18-api-reference.md). `snapshot`/`restore` come from the `drafts` plugin ([09 §8](./09-loading-and-submission.md#8-snapshots-drafts)).

**Arrays.** `engine.array(path)` returns the same `ArrayHandle` that array controllers expose ([11 §2](./11-rendering-and-adapters.md#2-headless-field-api)). Rows are named by `RowKey`, positions by index:

```ts
interface ArrayHandle {
  rows: Readable<RowKey[]>;
  insert(at?: number, row?: Json): RowKey;      // position; default append; returns the new key
  remove(key: RowKey): void;
  move(key: RowKey, to: number): void;          // which row by key, where by position
  set(key: RowKey, row: Json): void;
  replace(rows: Json[]): void;                  // keys matched by identityKey if configured (04 §4), else new
  keyAt(index: number): RowKey | undefined;
}
```

Each call updates values and row keys together in one tick.

Writes through this API get origin `host`. Renderers write through `FieldController.setValue`, which gets `user`. Callers never pass built-in origins; what each origin does is defined once in [07 §2.1](./07-effects-and-computed.md#21-origin-policy). Host code can skip effects for one write with `runEffects: false` (for example for bulk writes).

**`reset`** is one tick with origin `reset` ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)). It:
- clears `current`, `notices`, `meta.touched`, `meta.visited`, `meta.stash`, and `meta.conflict`;
- restores `meta.rows` from the known baseline row-key maps when clearing `current`; with `keepLoaded: false`, rebuilds the default baseline's keys and associations together. It never leaves current keys pointing at a different baseline row;
- with `keepLoaded: false`, also empties `loaded`, so values fall back to defaults (as for a new record);
- has origin policy `history: 'clear'`.

**`revision`** is a core counter. It bumps once per commit whose diff to `current`, `meta.rows` or `meta.stash` is non-empty, whatever the origin (including `reset`, `load` on reload, `restore`); commits that change nothing there don't bump it. `resume` installs the exported value instead ([09 §9](./09-loading-and-submission.md#9-server-rendering)).

## 8. Plugins

Decision D23 ([README](./README.md#key-decisions-adr-summary)). A feature may be a plugin only if the host uses it purely through explicit calls **and** its absence changes no default behaviour of fields, validation or submission. Plugins: `history` (§6.2), `drafts` (snapshot/restore, [09 §8](./09-loading-and-submission.md#8-snapshots-drafts)), `wizard` ([11 §5](./11-rendering-and-adapters.md#5-wizards)), `icu` (ICU message formatting, [10 §2.1](./10-localization.md#21-formatter)), `assets` (`asset` / `multi-asset` fields, [08 §2.1](./08-async-data-and-registries.md#21-asset-stores)). Conflicts, async validation, options, loading and submission are core.

```ts
import { createFormEngine } from '@formgraph/engine';           // default entry: history, drafts, wizard, icu, assets are already in the registry
createFormEngine({ contract, registry, plugins: { history: { limit: 100 } } });   // per-engine options, by plugin name

import { createFormEngine, createRegistry } from '@formgraph/engine/core';   // slim (README size budgets): record only what you use
import { history, drafts } from '@formgraph/engine/plugins';
const registry = createRegistry().plugin(history).plugin(drafts);
createFormEngine({ contract, registry, plugins: { history: { limit: 100 } } });
```

A plugin is a **factory recorded in the `Registry`** ([08 §1](./08-async-data-and-registries.md#1-registries)), not something passed to the engine: `registry.plugin(factory)` registers the field types and validators it declares (so `loadContract` sees them with no engine) and records the factory. Each engine then creates its own **instance** from it, so engines sharing one registry have separate plugin state. The default entry's `createRegistry` starts with the built-ins, and its `loadContract` / `createFormEngine` add any built-in a registry lacks; the slim entry starts empty. The API is the same in both cases. Built-in plugins (`history`, `drafts`, `wizard`, `assets`, `icu`) attach their members at top level (`engine.history.undo()`, `engine.snapshot()`, `engine.wizard()`; full list in [18 §5](./18-api-reference.md#5-built-in-plugin-members)). With the slim entry, calling a method of a plugin whose factory isn't in the registry throws `E_PLUGIN_MISSING`, naming the plugin and its import. Engine types are inferred from the registry's type (§8.1), so a missing plugin is also a per-engine type error; the runtime throw remains for JS callers. Third-party plugins are reachable only as `engine.plugins[name]` (§8.1).

`icu` and `assets` have no methods; slim core detects their absence from the contract instead: an ICU construct in a message or an `asset`/`multi-asset` field fails with `E_PLUGIN_MISSING` at compile (or, for `LocaleSource` bundles, when they arrive), naming the import. The form never runs with raw message syntax or unhandled fields.

### 8.1 Plugin API

```ts
function definePlugin<const Name extends string, Opts = void, Members = {}, const R extends PluginRegisters = {}>(
  name: Name,
  def: {
    requires?: string[];                          // other plugin names; missing → E_PLUGIN_MISSING
    registers?: R;                                // static: { fieldTypes?, validators? }, visible to loadContract; types flow from R (21 §1)
    setup(ctx: PluginContext, opts: Opts): PluginInstance | void;   // once per engine, with that engine's options
  },
): PluginFactory<Name, Opts, Members, R>;

interface PluginContext {
  engine: FormEngine;
  store: PluginStore;                           // read everything; commit under own x- origins; values and own meta slices only
  origins: { register(name: string, policy: OriginPolicy): void };   // third-party: x- names only
  meta: { slice<T extends Json>(name: string, init: T): Slice<T, string> };  // own meta slice, named <plugin>-…, e.g. x-audit-queue
  use(hooks: CommitHooks): void;                // §2.5, ordered before host hooks
  submission: { on(event: 'before' | 'success' | 'failure', fn: SubmitHook): void };   // 09 §4
  notices: { set(key: string, notices: NoticeInput[]): void };       // form-level notices owned by this plugin; set(key, []) clears; cleared on dispose
  controllers: { extend<M>(create: (c: FieldController) => { members: M; dispose?(): void }, opts?: { fieldTypes?: string[] }): void };
  text: { setFormat(f: TextFormat): void; messages(m: Record<string, Text>): void };   // icu (10 §2.1)
  onReveal(fn: (path: DataPath) => void | Promise<void>): void;      // reveal hook (below)
  onDispose(fn: () => void): void;
}

interface PluginInstance { methods?: Record<string, unknown>; exportState?(): Json; resumeState?(s: Json): void }
```

```ts
// one base, customized by type parameters
interface BasePluginStore<O extends string, S extends SliceAccess> {
  read: StoreReader;                                          // everything; same for both
  commit(origin: O, fn: (tx: PluginTx<S>) => void): void;     // same rules as store.transact (§2.1): nested, atomic, one commit
}
type PluginStore<Name extends string = string> = BasePluginStore<`x-${string}`, OwnSlices<Name>>;   // third-party
type PrivilegedPluginStore = BasePluginStore<CoreOrigin | `x-${string}`, AllSlices>;                // built-ins

type CoreOrigin = keyof typeof ORIGIN_POLICY;                 // 07 §2.1: user, host, load, external, effect, reset, history, restore, …
type CoreMetaKey = 'rows' | 'baselineRows' | 'stash' | 'conflict' | 'touched' | 'visited' | 'field';   // the meta sub-structures in §1

type StoreReader = Pick<FormStore, 'snapshot' | 'select' | 'selectForm'>;   // §2.2: immutable snapshot (all slices) and Readable handles

interface SliceAccess {                                       // what a commit may write
  values: boolean;                                            // tx.set / tx.delete; the layer is the one the origin's policy names (§1)
  core: CoreMetaKey;                                          // core meta sub-structures writable through tx.meta; `never` = none
  notices: boolean;                                           // per-path notices of any source (§1 `notices`)
  slices: string;                                             // names of plugin meta slices writable through tx.slice; `never` = none
}
type OwnSlices<Name extends string = string> = { values: true; core: never; notices: false; slices: `${Name}-${string}` };
type AllSlices = { values: true; core: CoreMetaKey; notices: true; slices: string };

type PluginTx<S extends SliceAccess> =                        // members exist only where S allows them, so a forbidden write is a type error
  & (S['values'] extends true ? { set(path: DataPath, value: Json): void; delete(path: DataPath): void } : {})
  & ([S['core']] extends [never] ? {} : { meta(key: S['core'], path: DataPath, patch: Json): void })
  & (S['notices'] extends true ? { notices(path: DataPath, source: Notice['source'], list: NoticeInput[]): void } : {})
  & ([S['slices']] extends [never] ? {} : { slice<N extends S['slices'], T extends Json>(s: Slice<T, N>, next: T): void });
```

```ts
// typing: registry + typed compiled contract + per-engine options (18 §1–2, §5)
// FormEngineWith<R, V = Json, O = undefined> filters members by enabled plugins.

const audit = definePlugin('x-audit', {
  registers: { validators: { 'x-audit-trail': auditTrail } },
  setup(ctx, opts: { sink: (e: AuditEvent) => void }) {
    ctx.submission.on('success', (r) => opts.sink({ type: 'submit', mode: r.mode }));
    return { methods: { flush() { … } } };
  },
});
const registry = createRegistry({ … }).plugin(audit);          // 'x-audit-trail' is now known to loadContract and typed in contracts
const engine = createFormEngine({ contract, registry, plugins: { 'x-audit': { sink } } });   // options required when Opts isn't void
engine.history.undo();               // HistoryApi (default entry)
engine.plugins['x-audit'].flush();   // typed AuditApi
```

- **Registration.** `registry.plugin(factory)` (or `createRegistry({ plugins: [...] })`) registers `registers` on the same append-only registry (a clash throws `E_REGISTRY_DUPLICATE`, naming the plugin) and records the factory; the same name twice in one registry throws `E_PLUGIN_DUPLICATE`. `plugins: { name: false }` skips one plugin for an engine (a contract that needs its types then fails with `E_PLUGIN_MISSING`). Plugin code has no way to register types or validators at `setup`, so there is one registration path.
- `methods` are exposed as `engine.plugins[name]`. Only built-in plugins also attach top-level members (`engine.history`, `engine.snapshot`/`restore`, `engine.wizard`, `assets`/`icu` have none); third-party plugins can't add top-level names, so they never collide with core or future built-ins. `PluginMembers<R, O>` and `PluginMap<R, O>` derive the top-level and namespaced members from both registry `R` and per-engine options `O`. Literal `false` removes a plugin's members; options that may disable it make those members optional. With no overrides (`O = undefined`), all registered plugins are enabled. The default entry adds built-ins before this filtering, never after it ([18 §5](./18-api-reference.md#5-built-in-plugin-members)).
- The core `ORIGIN_POLICY` table holds core origins only ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)). Plugin-owned built-in origins are registered at install: `history` registers `history`, `drafts` registers `restore`. Third-party plugins register `x-` origins only and name meta slices `<plugin>-…` (the plugin name already starts with `x-`).
- Core never refers to plugin behaviour by name. It exposes policy flags and hooks that plugins act on or implement:

  | Hook | Core use | Implemented by |
  |------|----------|----------------|
  | origin `history` flag (`record`, `merge`, `skip`, `clear`, `clear-overlap`) | set on every commit | `history` |
  | `reveal(path)` | make a node reachable before focusing it (`controller.focus()`, `focusFirstInvalid`, [11 §8](./11-rendering-and-adapters.md#8-accessibility)); core expands collapsed sections | `wizard` (switches to the step) |
  | validation scope (`validate({ paths })`, [09 §4](./09-loading-and-submission.md#4-submission-pipeline)) | scoped full-rule validation, same warning policy as submit | `wizard` (`canNext`/`next` on the step subtree) |
  | `exportState`/`resumeState` | server rendering | any |
  | field type `pending` (pending-work gate, [01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions)) | `submit()`/`next()` wait or fail like async validation; `failed` blocks | `assets` |
  | field type `toSnapshot` | value filter in `snapshot()` | `assets` (keeps `ready` refs) |
  | text `format` | message formatting beyond `{name}` | `icu` |
- `exportState`/`resumeState` let a plugin take part in server rendering ([09 §9](./09-loading-and-submission.md#9-server-rendering)). Built-ins: `wizard` exports the current and visited steps; `history`, `drafts`, `assets` and `icu` export nothing (the undo stack starts empty, as after a load). `resumeState` input is untrusted: the plugin validates it, and on failure starts from its default state with `W_RESUME_PLUGIN_UNKNOWN`.
- **Privileges.** Built-in plugins are defined with an internal variant of `definePlugin` (not exported) whose `ctx.store` is the `PrivilegedPluginStore`: commits under any origin, every slice. `PluginStore` and `PrivilegedPluginStore` both extend `BasePluginStore<Origin, Slices>`, which fixes the read side and the shape of `commit`; the type parameters set which origins and slices a plugin may use. The type parameters are only a compile-time guard. At runtime the engine also checks that a commit's origin is one this plugin registered, and that a slice name carries the plugin's prefix, and throws otherwise. Third-party plugins get `PluginStore`, which cannot write other plugins' slices, `errors` or core meta, and cannot use built-in origins. This is enforced by types and the plugin API, not a security boundary: plugins are trusted code the host chose to install and run in the same process, and the restriction prevents accidents and collisions with core or other plugins. Contracts are the untrusted input ([05 §4](./05-expressions.md#4-security-model)). Form-level messages from a plugin go through `ctx.notices.set` (`source: 'plugin'`, `rule` = the plugin name); they count like any notice, so an `error` from a plugin blocks submit with reason `errors`.
- **Controller extensions.** `ctx.controllers.extend(create, { fieldTypes? })` adds `controller.plugins[name]` to field controllers (all, or only those of the listed types). `create` runs lazily when the controller is first created and may return `dispose`, called when the field or its row is removed. Members live as long as the row key, so they survive reorders, unlike a path string. State belongs in the plugin's own meta slice (keyed by row key), written through `ctx.store`, so it shows up in snapshots and `exportState` like other plugin state. `controller.assets` is built the same way. Third-party plugins use `x-` names only. Scope and array controllers can't be extended yet. Types: `controller.plugins` is typed like `engine.plugins`, from the engine's registry (`FieldController<V, R>`); adapters pass the controller through unchanged.
- **Submit hooks.** `ctx.submission.on('before' | 'success' | 'failure', fn)`; see [09 §4](./09-loading-and-submission.md#4-submission-pipeline). Observe-only, except that `before` may stop the submit.
- **Text.** One text formatter: a second `setFormat` throws `E_PLUGIN_CONFLICT`, naming both plugins. `text.messages` defaults from several plugins merge; the host's messages win, and between plugins the first installed wins ([10 §2](./10-localization.md#2-resolution)).
- **No contract extension.** Plugins add field types, validators and `x-` keys, nothing else. The engine doesn't check `x-` keys: it owns no schema for them, and a typo is indistinguishable from another extension. Their owner (a renderer or plugin) checks its own `x-` data where it uses it ([01 §1](./01-schema-contract.md#1-principles)). Form-wide checks use group rules on the root node and the component `report`/`onValidate` hooks ([11 §2.2](./11-rendering-and-adapters.md#22-component-reported-messages-and-busy-state)); there are no plugin validation hooks.
- Plugins run `setup` in registry order (parent registry first, then `.plugin()` call order), before `ready`. Third-party plugin codes use `E_X_<PLUGIN>_…` ([17](./17-diagnostics.md)).
