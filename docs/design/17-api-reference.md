# 17 · API Reference

[← Index](./README.md)

Every public signature of `@formgraph/engine` in one place. Semantics live in the linked sections; this page only lists. Diagnostics are in [16](./16-diagnostics.md).

## 1. Loading a contract

```ts
defineContract<const C, const L extends InferenceLimits = {}>(registry: Registry<E>, contract: C & ContractFor<E>, opts?: { inference?: L }): C;   // identity at runtime; type-checks ids/config (20 §2); L: type-only caps (20 §5.1)

loadContract<C = unknown>(                       // 01 §7.1; never throws for contract problems; C inferred from a defineContract result (20 §3)
  input: string | Json | C,
  opts: {
    registry: Registry; strict?: boolean; limits?: Limits; allow?: AllowList;
    contractSchema?: StandardSchemaV1<unknown, FormContractInput>;   // swap the contract validator (01 §7)
  },
): Promise<LoadResult>

type LoadResult =
  | { ok: true;  schema: CompiledSchema<C>; diagnostics: Diagnostic[] }   // warnings only; CompiledSchema<unknown> → engine widens to Json
  | { ok: false; diagnostics: Diagnostic[] };                          // at least one error (16 §1)
```

```ts
const res = await loadContract(json, { registry });
if (!res.ok) return showErrors(res.diagnostics);
const engine = createFormEngine({ contract: res.schema, registry });
```

Links: [01 §7.1](./01-schema-contract.md#71-contract-load-pipeline), [`Limits`](./01-schema-contract.md#74-limits), [`AllowList`](./04-expressions.md#4-security-model).

## 2. Creating an engine

```ts
createFormEngine<const P extends readonly Plugin[] = []>(    // sync; throws only for the stats/refs recheck
  options: FormEngineOptions & { plugins?: P },
): FormEngineWith<P, V>                                      // default entry: FormEngineWith<[...BuiltIns, ...P], V>; V from options.contract (20 §3)
createFormEngineFrom<const P extends readonly Plugin[] = []>(   // convenience: loadContract + createFormEngine
  input: string | Json,
  opts: Omit<FormEngineOptions, 'contract'> & { plugins?: P; contractSchema?: StandardSchemaV1<unknown, FormContractInput> },
): Promise<FormEngineWith<P>>   // loads with the same registry/limits/allow/strict; ok:false → throws ContractLoadError { diagnostics }; warnings → onDiagnostic

interface FormEngineOptions {
  contract: CompiledSchema<C>;                    // from loadContract (§1); stats/refs rechecked → E_LIMIT_EXCEEDED / E_NOT_ALLOWED, phase call (01 §7.4)
  registry: Registry;                             // wrapped in a per-engine child (07 §1)
  context?: Json;                                 // $context for expressions (04)
  platform?: Partial<Platform>;                   // now, timers, nextFrame, dev; defaults from globalThis; dev: guarded __DEV__/NODE_ENV, else false (10 §6)
  params?: Record<string, Json>;                  // e.g. route id for entity loading (08 §1)
  limits?: Limits;                                // 01 §7.4
  allow?: AllowList;                              // 04 §4
  strict?: boolean;                               // strict → error warnings (16 §1)
  plugins?: readonly Plugin[];                    // inferred as P above (02 §8.1)
  history?: { limit?: number; coalesceMs?: number };   // history plugin config, default entry (02 §6.2)
  i18n?: { locale?: LocaleTag; fallbackLocales?: LocaleTag[]; resolve?: TextResolver; format?: TextFormat };   // 09 §2
  resume?: EngineState;                           // server rendering (08 §9)
  server?: boolean;                               // server-safe mode; automatic without `window` (08 §9)
  idPrefix?: string;                              // FieldIds prefix (10 §8)
  testIds?: { prefix?: string; format?: (path: DataPath) => string } | false;   // e2e selectors, state.testId (10 §8.1)
  onSubmitFailed?(result: SubmitResult & { ok: false }): void;   // every failed submit; adapters install a default (10 §8)
  onDiagnostic?(d: Diagnostic): void;             // runtime diagnostics (16 §1)
}
```

Links: [01 §7](./01-schema-contract.md#7-versioning-and-load-time-validation), [02 §6.2](./02-state-model.md#62-undo--redo-opt-in), [02 §8](./02-state-model.md#8-plugins), [04 §4](./04-expressions.md#4-security-model), [07 §1](./07-async-data-and-registries.md#1-registries), [08 §1](./08-loading-and-submission.md#1-config), [08 §9](./08-loading-and-submission.md#9-server-rendering), [09 §2](./09-localization.md#2-resolution), [10 §8](./10-rendering-and-adapters.md#8-accessibility), [16 §1](./16-diagnostics.md#1-shape).

## 3. `FormEngine` (core)

```ts
interface FormEngine<V = Json, Paths extends string = string> {   // V = ValuesOf<C>, Paths = PathsOf<V> for typed contracts (20 §3)
  // lifecycle
  ready: Promise<void>;                                         // loads, sub-schemas, setup option fetches (08 §9)
  dispose(): void;                                              // aborts async work, drops subscriptions; later writes are no-ops
  signal: AbortSignal;                                          // aborted on dispose; tie host async work to the engine
  form: Readable<FormState> & {                                 // phase, valid, validating, dirty, notices, severity, counts, conflictCount… (02 §5)
    select<T>(pick: keyof FormState | ((s: FormState) => T), eq?: (a: T, b: T) => boolean): Readable<T>;   // notifies only when the pick changes
  };
  revision: Readable<number>;                                   // bumps per commit changing current/rows/stash (02 §7)

  // reading
  field<P extends Paths>(path: P): FieldController<ValueAt<V, P>>;                 // typed overload (20 §3)
  field<T = Json>(path: PathRef, scope?: ScopeController): FieldController<T>;   // resolves ./ against scope (10 §3)
  getValues(): V;
  getValues(path?: DataPath): Json;                             // resolved values, no policies or wire mapping (02 §7)
  firstInvalid(opts?: { include?: Array<'errors' | 'external' | 'conflicts'> }): DataPath | null;   // 10 §8
  summary(): Array<Notice & { path: DataPath }>;                // every current notice in form order, no validation run (05 §7.4)
  severityBase(s: Severity): CoreSeverity;                      // custom severity → its base; core → itself (05 §7.6)

  // writing (origin host)
  setValue<P extends Paths>(path: P, value: ValueAt<V, P>, opts?: { runEffects?: boolean; origin?: `x-${string}` }): void;   // typed overload (20 §3)
  setValue(path: PathRef, value: Json, opts?: { runEffects?: boolean; origin?: `x-${string}` }): void;   // 02 §7
  revert(path: PathRef): void;                                  // drop from current (02 §7)
  reset(opts?: { keepLoaded?: boolean }): void;                 // origin reset (02 §7)
  array(path: PathRef): ArrayHandle;                            // 02 §7
  touch(path: PathRef, opts?: { validate?: boolean }): void;    // mark touched (container: descendants); validate (default true) runs unvalidated blur rules (05 §3)
  batch(fn: () => void): void;                                  // host writes inside → one commit (02 §2.1)
  setContext(partial: Json): void;                              // starts a tick (04 §3)

  // loading, submission, external data
  reload(opts?: { entity?: Json; policy?: ReloadPolicy }): Promise<void>;   // rejects E_SUPERSEDED when superseded (08 §3)
  validate(opts?: { trigger?: 'submit'; paths?: PathRef[] }): Promise<{ ok: boolean; notices: Array<Notice & { path: DataPath }> }>;   // 08 §4
  serialize(opts?: { mode?: 'full' | 'patch'; validate?: boolean }): Json;   // throws E_INVALID with validate (08 §4)
  submit(opts?: { confirm?: string[] }): Promise<SubmitResult>;  // 08 §4; confirm: gates to confirm first (05 §7.3); while one is in flight, returns that same promise
  confirm(gate: string): void;                                  // 05 §7.3; valid while the gate's notices stay a subset of those confirmed
  unconfirm(gate: string): void;
  mapExternalErrors(errors: ExternalErrors): void;              // 08 §5
  applyExternal(changes: Array<{ path: string; value?: Json; op?: 'set' | 'remove' }>, opts?: { paths?: 'wire' | 'data' }): void;   // 08 §7
  exportState(): EngineState;                                   // 08 §9

  // localization
  setLocale(locale: LocaleTag): Promise<void>;                  // atomic, latest wins (09 §3.3)

  // extension
  use(hooks: CommitHooks): () => void;                          // returns detach (02 §2.5)
  plugins: {};                                                  // narrowed by PluginMembers<P> per engine (02 §8.1)
}

type SubmitResult =                                             // 08 §4
  | { ok: true }
  | { ok: false; reason: 'errors' | 'external' | 'transform' | 'validating' | 'assets-pending' }
  | { ok: false; reason: 'conflicts'; paths: DataPath[] }
  | { ok: false; reason: 'gated'; gates: GateStop[] };          // GateStop: 08 §4

type ReloadPolicy = 'preserve-edits' | 'replace-unchanged' | 'replace';   // 08 §3
```

Links: [02 §2.5](./02-state-model.md#25-commit-hooks), [02 §5](./02-state-model.md#5-field-meta-and-form-status), [02 §7](./02-state-model.md#7-engine-write-api-core), [04 §3](./04-expressions.md#3-evaluation-scope), [05 §3](./05-validation.md#3-triggers), [08 §3](./08-loading-and-submission.md#3-reload-policy), [08 §4](./08-loading-and-submission.md#4-submission-pipeline), [08 §5](./08-loading-and-submission.md#5-external-error-mapping), [08 §7](./08-loading-and-submission.md#7-external-updates-and-conflicts), [09 §3.3](./09-localization.md#33-loading-bundles-separately), [10 §8](./10-rendering-and-adapters.md#8-accessibility).

## 4. Handles and controllers

```ts
interface Readable<T> {                                         // 02 §2.2
  get(): T;                                                     // also reports to the active read observer (02 §2.6)
  subscribe(listener: (value: T) => void): () => void;          // not called immediately
}

// @formgraph/engine/interop: integration points for adapter authors (02 §2.6), not app code
function observeReads<T>(observer: (r: Readable<unknown>) => void, fn: () => T): T;   // dev: thenable result → W_ASYNC_TRACKED
function untracked<T>(fn: () => T): T;

// @formgraph/engine/reactive: optional, for code without a framework (02 §2.6)
type NotAsync<T> = T extends PromiseLike<unknown> ? never : T;   // async fns rejected at compile time
function derived<T>(fn: () => NotAsync<T>, opts?: { equals?: (a: T, b: T) => boolean }): Readable<T>;
function watch(fn: () => NotAsync<void | (() => void)>, opts?: { signal?: AbortSignal }): () => void;

interface ArrayHandle {                                         // engine.array(path) and ArrayController (02 §7)
  rows: Readable<RowKey[]>;
  insert(at?: number, row?: Json): RowKey;                      // position; default append; returns the new key
  remove(key: RowKey): void;
  move(key: RowKey, to: number): void;                          // which row by key, where by position
  set(key: RowKey, row: Json): void;
  replace(rows: Json[]): void;                                  // keys matched by identityKey if configured, else new
  keyAt(index: number): RowKey | undefined;
}

interface FieldController<V = Json> {                           // 10 §2
  path: Readable<DataPath>; key: RowKeyPath; compiled: CompiledNode;   // path changes on row moves; key doesn't
  state: Readable<FieldState<V>>;
  select<T>(pick: keyof FieldState<V> | ((s: FieldState<V>) => T), eq?: (a: T, b: T) => boolean): Readable<T>;
  ids: FieldIds;                                                // 10 §8
  value: Readable<V>; visible: Readable<boolean>; feedback: Readable<Notice[]>; dirty: Readable<boolean>;   // … one per FieldState member
  options?: Readable<OptionsState>;                             // 12 §9
  assets?: AssetController;                                     // assets plugin (07 §2.1)
  setValue(v: V): void;                                         // origin user
  revert(): void;
  handlers: { onFocus(): void; onBlur(): void };                // visited / touched + blur validation
  focus(): Promise<boolean>;                                    // DOM focus only; reveal + one frame wait (10 §8)
  registerElement(el: unknown): () => void;                     // returns unregister (10 §8)
  setInputError(message: Text | null): void;                    // 10 §2.1
  report(reporter: string, messages: ComponentNotice[] | null): void;   // 10 §2.2; replaces this reporter's messages
  onValidate(reporter: string, fn: ComponentValidate): () => void;      // 10 §2.2; joins submit/next/validate; returns unregister
  setBusy(token: string, busy: boolean): void;                  // 10 §2.2; holds the pending-work gate
  resolveConflict(choice: 'local' | 'external'): void;          // 08 §7.2
}

interface ArrayController extends FieldController<Json[]>, ArrayHandle {
  canAdd: Readable<boolean>; canRemove: Readable<boolean>;
  row(key: RowKey): ScopeController;
}

interface ScopeController {                                     // array row or dataScope group (10 §2)
  path: Readable<DataPath>; key: RowKeyPath;
  field(relPath: PathRef): FieldController;
  value: Readable<Json>; visible: Readable<boolean>;
  dirty: Readable<boolean>;                     // rows: derived on subscribe, O(row) (02 §4); groups: O(1) count
  testId: Readable<string>;                     // 10 §8.1
  feedback: Readable<Notice[]>; severity: Readable<Severity | null>; showFeedback: Readable<boolean>;
  registerElement(el: unknown): () => void;
  focus(): Promise<boolean>;
}
```

`FieldState`, `FieldIds` and `AssetController`: [10 §2](./10-rendering-and-adapters.md#2-headless-field-api), [10 §8](./10-rendering-and-adapters.md#8-accessibility). `OptionsState`: [12 §9](./12-option-sources-and-cascading.md#9-renderer-view).

## 5. Built-in plugin members

Installed by the default entry; with `@formgraph/engine/core`, only when the plugin is passed in `plugins` ([02 §8](./02-state-model.md#8-plugins)). Each built-in factory carries its members in its type; `PluginMembers<P>` adds them at top level and under `engine.plugins[name]`:

```ts
interface Plugin<Name extends string = string, Members = {}> { name: Name; setup(ctx: PluginContext): PluginInstance | void; /* phantom Members */ }
type FormEngineWith<P extends readonly Plugin[], V = Json> = FormEngine<V, PathsOf<V>> & PluginMembers<P>;

history(o?: { limit?: number; coalesceMs?: number }): Plugin<'history', { history: HistoryApi }>;   // 02 §6.2
drafts(): Plugin<'drafts', {                                                                         // 08 §8
  snapshot(): Snapshot;
  restore(snapshot: Snapshot, opts?: { onStale?: 'conflict' | 'keep-draft' | 'drop' }): Promise<RestoreResult>;
}>;
wizard(): Plugin<'wizard', { wizard(): WizardController }>;                                         // 10 §5
assets(): Plugin<'assets'>; icu(): Plugin<'icu'>;

interface HistoryApi {
  undo(): void; redo(): void; clear(): void;
  canUndo: Readable<boolean>; canRedo: Readable<boolean>;
}

interface WizardController {                                    // 10 §5
  steps: Readable<StepInfo[]>; current: Readable<number>; canNext: Readable<boolean>;
  next(opts?: { confirm?: string[] }): Promise<{
    ok: boolean;
    reason?: 'errors' | 'gated' | 'validating' | 'assets-pending' | 'last-step';
    gates?: GateStop[];
  }>;
  prev(): void; goTo(step: number): void;
}
```

- `assets` adds no engine members; it adds `FieldController.assets` ([07 §2.1](./07-async-data-and-registries.md#21-asset-stores)).
- `icu` adds no members; it sets the text formatter ([09 §2.1](./09-localization.md#21-formatter)).
- Calling a member of a missing built-in plugin is a type error on that engine and throws `E_PLUGIN_MISSING` at runtime (JS callers).

Links: [02 §6.2](./02-state-model.md#62-undo--redo-opt-in), [08 §8](./08-loading-and-submission.md#8-snapshots-drafts), [10 §5](./10-rendering-and-adapters.md#5-wizards).

## 6. Third-party plugins

Third-party plugin methods are only reachable as `engine.plugins[name]`, typed from the plugin's `Members` ([02 §8.1](./02-state-model.md#81-plugin-api)):

```ts
const audit = (): Plugin<'x-audit', AuditApi> => ({ name: 'x-audit', setup: … });
const engine = createFormEngine({ contract, registry, plugins: [audit()] });
engine.plugins['x-audit'].flush();
```

## 7. Renderers (adapter packages)

Shared by all adapters; each adapter supplies its component type `C` ([10 §4](./10-rendering-and-adapters.md#4-renderer-registry)).

```ts
defineRenderers<C>(map: {
  field?: Record<string, C>;                    // key: fieldType or `fieldType:variant`; props typed per field type
  group?: Record<string, C>;                    // key: ui.variant, or `step` / `step:variant`
  array?: Record<string, C>;                    // key: ui.variant
  'dynamic-group'?: Record<string, C>;          // falls back to group entries
  fallback?: C | {                              // one component = catch-all for every kind (10 §4)
    field?: C; group?: C; array?: C;
    'dynamic-group'?: C;                        // falls back to `group` first
    '*'?: C;                                    // catch-all for kinds not listed
  };                                            // uncovered kinds → the adapter's built-in fallback (10 §4.2)
// adapters also export DevFallback, ProdFallback, BuiltInFallback (picks by platform.dev) for reuse
}): Renderers<C>

interface Renderers<C> {
  register(component: C, tester: Tester): void;                         // 10 §4.1; later registration wins ties
  resolve(node: CompiledNode): C;                                        // cached per compiled node
  missing(schema: CompiledSchema): Array<{ fieldType: string; paths: DataPath[] }>;   // pure; field types that would fall back (10 §4.2)
}

type Tester = (node: CompiledNode, ctx: { platform: string; parent?: CompiledNode }) => number;   // -1 = no match
// helpers: rankWith, and, isFieldType, hasExt, isField
```

When rendering falls back for a field, the adapter reports `W_NO_RENDERER` once per field type ([16](./16-diagnostics.md)).

## 8. Resolved while compiling this page

- `onSubmitFailed` runs for every `{ ok: false }` result; `validating` and `assets-pending` are result reasons, not thrown diagnostics (their former catalog codes are removed).
- `engine.focus` / `engine.blur` removed; interaction is `controller.handlers`, `controller.focus()` is DOM-only.
- `engine.array` takes row keys (was indexes), shared with `ArrayController`; `replace` matches rows by `identityKey`.
- Built-in plugin members are top-level; third-party only under `engine.plugins`.
- Plugin typing is inferred from the `plugins` option (`FormEngineWith<P>`), not global module augmentation, so a missing plugin is a per-engine type error.
- Form status, form-level notices and `conflictCount` had no accessor; they are one `engine.form: Readable<FormState>` with `select(key | fn)`, like `controller.state`.
- `createFormEngine` takes only a `CompiledSchema`; contract problems reach the host through `LoadResult`, not a throw. `createFormEngineFrom` covers the raw-input case. `contractSchema` moved to `loadContract`; `FormState.phase` has no `compiling`.
- Locale state is part of `engine.form`: `locale` and `localeStatus` (select them like any form state); `setLocale` stays a method.
- `touch(path, { validate })` marks touched and, by default, runs blur-triggered rules that haven't run for the current value, so revealed feedback is correct.
