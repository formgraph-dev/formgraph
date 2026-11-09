# 18 · API Reference

[← Index](./README.md)

Every public signature of `@formgraph/engine` in one place. Semantics live in the linked sections; this page only lists. Diagnostics are in [17](./17-diagnostics.md).

## 1. Loading a contract

```ts
declare const contractTypes: unique symbol;       // type-only, no runtime property
type TypedContract<E extends RegistryEntries, C, L extends InferenceLimits = {}> = C & {
  readonly [contractTypes]: { entries: E; input: C; inference: L };
};
function defineContract<const E extends RegistryEntries, const C, const L extends InferenceLimits = {}>(
  registry: Registry<E>, contract: C & ContractFor<NoInfer<E>>, opts?: { inference?: L },
): TypedContract<E, C, L>;                         // identity at runtime (21 §2); retains schemas + type-only caps

interface LoadOptions<R extends Registry = Registry> {
  registry: R; strict?: boolean; limits?: Limits; allow?: AllowList;
  contractSchema?: StandardSchemaV1<unknown, FormContractInput>;
  textFormat?: TextFormat;                        // parse-only validation available before engine creation (10 §2.1)
}
function loadContract<const E extends RegistryEntries, const C, const L extends InferenceLimits>(
  input: TypedContract<E, C, L>, opts: LoadOptions<Registry<NoInfer<E>>>,
): Promise<LoadResult<TypedContract<E, C, L>>>;
function loadContract(input: unknown, opts: LoadOptions): Promise<LoadResult<unknown>>;   // runtime JSON/string; never infers literals from unchecked input

interface CompiledSchema<C = unknown> {           // runtime shape: 01 §7.3
  readonly [contractTypes]: C;                    // phantom: original typed contract, not just its input tree
  // compiled nodes, graph, stats, refs and formatter identity; no runtime brand property
}
type RegistryFor<C> = C extends { readonly [contractTypes]: { entries: infer E extends RegistryEntries } } ? Registry<E> : Registry;
type LoadResult<C = unknown> =
  | { ok: true;  schema: CompiledSchema<C>; diagnostics: Diagnostic[] }   // warnings only; CompiledSchema<unknown> → engine widens to Json
  | { ok: false; diagnostics: Diagnostic[] };                          // at least one error (17 §1)
```

```ts
const res = await loadContract(json, { registry });
if (!res.ok) return showErrors(res.diagnostics);
const engine = createFormEngine({ contract: res.schema, registry });
```

Links: [01 §7.1](./01-schema-contract.md#71-contract-load-pipeline), [`Limits`](./01-schema-contract.md#74-limits), [`AllowList`](./05-expressions.md#4-security-model).

## 2. Creating an engine

```ts
function createFormEngine<const R extends Registry, C = unknown, const O extends PluginOptionsOf<R> | undefined = undefined>(
  options: FormEngineOptions<R, C, O>,
): FormEngineWith<R, ValuesOf<C>, O, PayloadOf<C>>;           // sync; retain value + payload inference; default entry adds built-ins before O
function createFormEngineFrom<const R extends Registry, const O extends PluginOptionsOf<R> | undefined = undefined>(
  input: string | Json,
  opts: Omit<FormEngineOptions<R, unknown, O>, 'contract'> & { contractSchema?: StandardSchemaV1<unknown, FormContractInput>; textFormat?: TextFormat },
): Promise<FormEngineWith<R, Json, O>>;   // runtime input widens; same registry/limits/allow/strict; ok:false → throws ContractLoadError { diagnostics }; warnings → onDiagnostic

interface FormEngineOptions<R extends Registry = Registry, C = unknown, O extends PluginOptionsOf<R> | undefined = PluginOptionsOf<R> | undefined> {
  contract: CompiledSchema<C>;                    // from loadContract (§1); stats/refs rechecked → E_LIMIT_EXCEEDED / E_NOT_ALLOWED, phase call (01 §7.4)
  registry: R & RegistryFor<NoInfer<C>>;             // compatible with the typed contract's entries; per-engine child (08 §1)
  context?: Record<string, Json>;                 // $context for expressions (05)
  platform?: Partial<Platform>;                   // now, timers, nextFrame, dev; defaults from globalThis; dev: guarded __DEV__/NODE_ENV, else false (11 §6)
  params?: Record<string, Json>;                  // e.g. route id for entity loading (09 §1)
  limits?: Limits;                                // 01 §7.4
  allow?: AllowList;                              // 05 §4
  strict?: boolean;                               // strict → error warnings (17 §1)
  plugins?: O;                                    // inferred per-engine options; false removes that plugin's members (03 §8)
  i18n?: { locale?: LocaleTag; fallbackLocales?: LocaleTag[]; resolve?: TextResolver; format?: TextFormat };   // 10 §2
  resume?: EngineState;                           // server rendering (09 §9)
  server?: boolean;                               // explicit server-safe mode, default false; no window detection (09 §9)
  idPrefix?: string;                              // FieldIds prefix (11 §8)
  testIds?: { prefix?: string; format?: (path: DataPath) => string } | false;   // e2e selectors, state.testId (11 §8.1)
  onSubmitFailed?(result: SubmitResult & { ok: false }): void;   // every failed submit; adapters install a default (11 §8)
  onDiagnostic?(d: Diagnostic): void;             // runtime diagnostics (17 §1)
}
```

Links: [01 §7](./01-schema-contract.md#7-versioning-and-load-time-validation), [03 §6.2](./03-state-model.md#62-undo--redo-opt-in), [03 §8](./03-state-model.md#8-plugins), [05 §4](./05-expressions.md#4-security-model), [08 §1](./08-async-data-and-registries.md#1-registries), [09 §1](./09-loading-and-submission.md#1-config), [09 §9](./09-loading-and-submission.md#9-server-rendering), [10 §2](./10-localization.md#2-resolution), [11 §8](./11-rendering-and-adapters.md#8-accessibility), [17 §1](./17-diagnostics.md#1-shape).

## 3. `FormEngine` (core)

```ts
interface FormEngine<V = Json, Paths extends string = string, Payload = Json> {   // ValuesOf<C>, PathsOf<V>, PayloadOf<C> for typed contracts (21 §3)
  // lifecycle
  ready: Promise<void>;                                         // loads, sub-schemas, setup option fetches (09 §9)
  dispose(): void;                                              // aborts async work, drops subscriptions; later writes are no-ops
  signal: AbortSignal;                                          // aborted on dispose; tie host async work to the engine
  form: Readable<FormState> & {                                 // phase, valid, validating, dirty, notices, severity, counts, conflictCount… (03 §5)
    select<K extends keyof FormState>(key: K, eq?: (a: FormState[K], b: FormState[K]) => boolean): Readable<FormState[K]>;
    select<T>(pick: (s: FormState) => T, eq?: (a: T, b: T) => boolean): Readable<T>;
  };
  revision: Readable<number>;                                   // bumps per commit changing current/rows/stash (03 §7)

  // reading
  field<P extends Paths>(path: P): FieldController<ValueAt<V, P>>;                 // typed overload (21 §3)
  field<T = Json>(path: PathRef, scope?: ScopeController): FieldController<T>;   // resolves ./ against scope (11 §3)
  getValues(): V;
  getValues(path?: DataPath): Json;                             // resolved values, no policies or wire mapping (03 §7)
  firstInvalid(opts?: { include?: Array<'errors' | 'external' | 'conflicts'> }): DataPath | null;   // 11 §8
  summary(): Array<Notice & { path: DataPath }>;                // every current notice in form order, no validation run (06 §7.4)
  severityBase(s: Severity): CoreSeverity;                      // custom severity → its base; core → itself (06 §7.6)

  // writing (origin host)
  setValue<P extends Paths>(path: P, value: ValueAt<V, P>, opts?: { runEffects?: boolean; origin?: `x-${string}` }): void;   // typed overload (21 §3)
  setValue(path: PathRef, value: Json, opts?: { runEffects?: boolean; origin?: `x-${string}` }): void;   // 03 §7
  revert<P extends Paths>(path: P): void;                       // typed overload (21 §3)
  revert(path: PathRef): void;                                  // restore target baseline; preserve other array cells (03 §3.2)
  reset(opts?: { keepLoaded?: boolean }): void;                 // origin reset (03 §7)
  array<P extends Paths>(path: P): ArrayHandle;                 // typed overload (21 §3)
  array(path: PathRef): ArrayHandle;                            // 03 §7
  touch<P extends Paths>(path: P, opts?: { validate?: boolean }): void;   // typed overload (21 §3)
  touch(path: PathRef, opts?: { validate?: boolean }): void;    // mark touched (container: descendants); validate (default true) runs unvalidated blur rules (06 §3)
  batch(fn: () => void): void;                                  // host writes inside → one commit (03 §2.1)
  setContext(partial: Record<string, Json>): void;                              // starts a tick (05 §3)

  // loading, submission, external data
  reload(opts?: { entity?: Json; policy?: ReloadPolicy }): Promise<void>;   // rejects E_SUPERSEDED when superseded (09 §3)
  validate(opts?: { trigger?: 'submit'; paths?: PathRef[] }): Promise<{ ok: boolean; notices: Array<Notice & { path: DataPath }> }>;   // 09 §4
  serialize(opts?: { mode?: 'full'; validate?: boolean }): Payload;   // throws E_INVALID with validate (09 §4)
  serialize(opts: { mode: 'patch'; validate?: boolean }): PatchOf<Payload>;   // optional object paths, whole arrays
  serialize(opts: { mode: 'full' | 'patch'; validate?: boolean }): Payload | PatchOf<Payload>;
  submit(opts?: { confirm?: string[]; signal?: AbortSignal; timeoutMs?: number }): Promise<SubmitResult>;  // 09 §4; first call's controls govern the shared promise
  cancelSubmit(): void;                                        // cancels even non-cooperative work; no-op when idle (09 §4)
  reconcileSubmit(reconciliationId: string, opts?: { entity?: Json; acknowledgedPaths?: DataPath[]; signal?: AbortSignal; timeoutMs?: number }): Promise<SubmitReconciliation>;   // saved response/coverage retry; rejects on failure, retains id; never calls target (09 §4.2)
  confirm(gate: string): void;                                  // 06 §7.3; valid while the gate's notices stay a subset of those confirmed
  unconfirm(gate: string): void;
  mapExternalErrors(errors: ExternalErrors): void;              // 09 §5
  applyExternal(changes: Array<{ path: string; value?: Json; op?: 'set' | 'remove' }>, opts?: { paths?: 'wire' | 'data' }): void;   // 09 §7
  exportState(): EngineState;                                   // 09 §9

  // localization
  setLocale(locale: LocaleTag): Promise<void>;                  // atomic, latest wins (10 §3.3)

  // extension
  use(hooks: CommitHooks): () => void;                          // returns detach (03 §2.5)
  plugins: {};                                                  // typed PluginMap<R> per engine (03 §8.1)
}

type SubmitSuccess =                                           // 09 §4; savedRevision is a local pre-finalization token
  | { ok: true; response: 'applied' | 'none'; savedRevision: number; pendingChanges: boolean }
  | { ok: true; response: 'failed'; savedRevision: number; pendingChanges: boolean; diagnostics: Diagnostic[]; reconciliationId: string };
type SubmitResult = SubmitSuccess
  | { ok: false; reason: 'load' | 'errors' | 'external' | 'transform' | 'validating' | 'assets-pending' | 'schema' | 'target' }
  | { ok: false; reason: 'reconcile-pending'; reconciliationId: string }
  | { ok: false; reason: 'conflicts'; paths: DataPath[] }
  | { ok: false; reason: 'gated'; gates: GateStop[] }           // GateStop: 09 §4
  | { ok: false; reason: 'cancelled' | 'timeout'; outcome: 'not-sent' | 'unknown' }
  | { ok: false; reason: `x-${string}`; notices?: NoticeInput[] };   // plugin before-hook stop

type ReloadPolicy = 'preserve-edits' | 'replace-unchanged' | 'replace';   // 09 §3
interface SubmitReconciliation { savedRevision: number; pendingChanges: boolean }   // 09 §4.2
type PatchOf<T> = T extends readonly unknown[] ? T : T extends object ? { [K in keyof T]?: PatchOf<T[K]> } : T;
```

Links: [03 §2.5](./03-state-model.md#25-commit-hooks), [03 §5](./03-state-model.md#5-field-meta-and-form-status), [03 §7](./03-state-model.md#7-engine-write-api-core), [05 §3](./05-expressions.md#3-evaluation-scope), [06 §3](./06-validation.md#3-triggers), [09 §3](./09-loading-and-submission.md#3-reload-policy), [09 §4](./09-loading-and-submission.md#4-submission-pipeline), [09 §5](./09-loading-and-submission.md#5-external-error-mapping), [09 §7](./09-loading-and-submission.md#7-external-updates-and-conflicts), [10 §3.3](./10-localization.md#33-loading-bundles-separately), [11 §8](./11-rendering-and-adapters.md#8-accessibility).

## 4. Handles and controllers

```ts
interface Readable<T> {                                         // 03 §2.2
  get(): T;                                                     // also reports to the active read observer (03 §2.6)
  subscribe(listener: (value: T) => void): () => void;          // not called immediately
}

// @formgraph/engine/interop: integration points for adapter authors (03 §2.6), not app code
function observeReads<T>(observer: (r: Readable<unknown>) => void, fn: () => T): T;   // dev: thenable result → W_ASYNC_TRACKED
function untracked<T>(fn: () => T): T;

// @formgraph/engine/reactive: optional, for code without a framework (03 §2.6)
type NotAsync<T> = T extends PromiseLike<unknown> ? never : T;   // async fns rejected at compile time
function derived<T>(fn: () => NotAsync<T>, opts?: { equals?: (a: T, b: T) => boolean }): Readable<T>;
function watch(fn: () => NotAsync<void | (() => void)>, opts?: { signal?: AbortSignal }): () => void;

interface ArrayHandle {                                         // engine.array(path) and ArrayController (03 §7)
  rows: Readable<RowKey[]>;
  insert(at?: number, row?: Json): RowKey;                      // position; default append; returns the new key
  remove(key: RowKey): void;
  move(key: RowKey, to: number): void;                          // which row by key, where by position
  set(key: RowKey, row: Json): void;
  replace(rows: Json[]): void;                                  // keys matched by identityKey if configured, else new
  keyAt(index: number): RowKey | undefined;
}

interface FieldController<V = Json> {                           // 11 §2
  path: Readable<DataPath>; key: RowKeyPath; compiled: CompiledNode;   // path changes on row moves; key doesn't
  state: Readable<FieldState<V>>;
  select<K extends keyof FieldState<V>>(key: K, eq?: (a: FieldState<V>[K], b: FieldState<V>[K]) => boolean): Readable<FieldState<V>[K]>;
  select<T>(pick: (s: FieldState<V>) => T, eq?: (a: T, b: T) => boolean): Readable<T>;
  ids: FieldIds;                                                // 11 §8
  value: Readable<V>; visible: Readable<boolean>; feedback: Readable<Notice[]>; dirty: Readable<boolean>;   // … one per FieldState member
  options?: Readable<OptionsState>;                             // 13 §9
  assets?: AssetController;                                     // assets plugin (08 §2.1)
  setValue(v: V): void;                                         // origin user
  revert(): void;
  handlers: { onFocus(): void; onBlur(): void };                // visited / touched + blur validation
  focus(): Promise<boolean>;                                    // DOM focus only; reveal + one frame wait (11 §8)
  registerElement(el: unknown): () => void;                     // returns unregister (11 §8)
  setInputError(message: Text | null): void;                    // 11 §2.1
  report(reporter: string, messages: ComponentNotice[] | null): void;   // 11 §2.2; replaces this reporter's messages
  onValidate(reporter: string, fn: ComponentValidate): () => void;      // 11 §2.2; joins submit/next/validate; returns unregister
  setBusy(token: string, busy: boolean): void;                  // 11 §2.2; holds the pending-work gate
  resolveConflict(choice: 'local' | 'external'): void;          // 09 §7.2
}

interface ArrayController extends FieldController<Json[]>, ArrayHandle {
  canAdd: Readable<boolean>; canRemove: Readable<boolean>;
  row(key: RowKey): ScopeController;
}

interface ContentController { key: RowKeyPath; compiled: CompiledNode; visible: Readable<boolean>; label: Readable<string | undefined>; description: Readable<string | undefined>; testId: Readable<string> }   // content nodes (01 §3.6, 11 §2)

interface ScopeController {                                     // array row or dataScope group (11 §2)
  path: Readable<DataPath>; key: RowKeyPath;
  field(relPath: PathRef): FieldController;
  value: Readable<Json>; visible: Readable<boolean>;
  dirty: Readable<boolean>;                     // rows: derived on subscribe, O(row) (03 §4); groups: O(1) count
  testId: Readable<string>;                     // 11 §8.1
  feedback: Readable<Notice[]>; severity: Readable<Severity | null>; showFeedback: Readable<boolean>;
  registerElement(el: unknown): () => void;
  focus(): Promise<boolean>;
}

interface DynamicGroupController extends ScopeController {      // 08 §5; old schema may remain displayed
  status: Readable<'idle' | 'loading' | 'ready' | 'error'>;
  diagnostic: Readable<Diagnostic | null>;
  retry(): Promise<void>;                                       // current discriminator, generation-checked
}
```

`FieldState`, `FieldIds` and `AssetController`: [11 §2](./11-rendering-and-adapters.md#2-headless-field-api), [11 §8](./11-rendering-and-adapters.md#8-accessibility). `OptionsState`: [13 §9](./13-option-sources-and-cascading.md#9-renderer-view).

## 5. Built-in plugin members

Recorded in the default entry's registry; with `@formgraph/engine/core`, only when `registry.plugin(factory)` is called ([03 §8](./03-state-model.md#8-plugins)). Each built-in factory carries its members in its type; `PluginMembers<R, O>` and `PluginMap<R, O>` expose the enabled ones at top level and under `engine.plugins[name]`:

The per-engine options are a second type input, `O` (default `undefined`, meaning no overrides). Literal `false` removes a plugin from both its top-level members and `engine.plugins`. A widened `boolean` or optional disabling option makes its members optional, requiring a presence check; it must not claim that a runtime-selected plugin is always installed. Required plugin options are checked at creation (and statically when visible); disabling a plugin explicitly remains allowed unless a contract/dependency needs it.

```ts
type PluginOptionAt<O, N extends PropertyKey> = O extends undefined ? undefined : N extends keyof O ? O[N] : undefined;
type DisabledPluginNames<R extends Registry, O> = {
  [N in PluginName<R>]: [PluginOptionAt<O, N>] extends [false] ? N : never
}[PluginName<R>];
type MaybeDisabledPluginNames<R extends Registry, O> = Exclude<{
  [N in PluginName<R>]: false extends PluginOptionAt<O, N> ? N : never
}[PluginName<R>], DisabledPluginNames<R, O>>;
type EnabledPluginNames<R extends Registry, O> = Exclude<PluginName<R>, DisabledPluginNames<R, O> | MaybeDisabledPluginNames<R, O>>;
type PluginMap<R extends Registry, O = undefined> =
  { [N in EnabledPluginNames<R, O>]: MembersOf<R, N> } &
  { [N in MaybeDisabledPluginNames<R, O>]?: MembersOf<R, N> };
type FormEngineWith<R extends Registry, V = Json, O = undefined, Payload = Json> =
  Omit<FormEngine<V, PathsOf<V>, Payload>, 'plugins'> & PluginMembers<R, O> & { plugins: PluginMap<R, O> };
```

`PluginMembers<R, O>` intersects the built-in top-level member objects for `EnabledPluginNames<R, O>` and the `Partial` member objects for `MaybeDisabledPluginNames<R, O>`; disabled and third-party plugins add no top-level members. `PluginName<R>` and `MembersOf<R, N>` read the factories recorded in `R`. This filtering applies after the default entry adds its built-in factories; it never adds a disabled built-in back. Controller extensions use the same enabled/optional filtering.

```ts
const e = createFormEngine({ contract, registry, plugins: { history: false } });
e.history.undo();                 // type error
e.plugins.history;               // type error
type WithoutHistory = FormEngineWith<typeof registry, Json, { history: false }>;
```

```ts
function definePlugin<…>(name, def): PluginFactory<Name, Opts, Members, Registers>;   // 03 §8.1
registry.plugin(factory): Registry<E & …>;                    // registers its types/validators, records the factory

history: PluginFactory<'history', { limit?: number; coalesceMs?: number } | void, { history: HistoryApi }>;   // 03 §6.2
drafts: PluginFactory<'drafts', void, {                                                                       // 09 §8
  snapshot(): Snapshot;
  restore(snapshot: Snapshot, opts?: { onStale?: 'conflict' | 'keep-draft' | 'drop' }): Promise<RestoreResult>;
}>;
wizard: PluginFactory<'wizard', void, { wizard(): WizardController }>;                                       // 11 §5
assets: PluginFactory<'assets', void>; icu: PluginFactory<'icu', void>;
debug: PluginFactory<'debug', DebugOptions | void, { debug: DebugApi }>;   // opt-in, `@formgraph/engine/plugins/debug`; never installed by either entry (22 §4)

interface HistoryApi {
  undo(): void; redo(): void; clear(): void;
  canUndo: Readable<boolean>; canRedo: Readable<boolean>;
}

interface WizardController {                                    // 11 §5
  steps: Readable<StepInfo[]>; current: Readable<number>; canNext: Readable<boolean>;
  next(opts?: { confirm?: string[] }): Promise<{
    ok: boolean;
    reason?: 'errors' | 'gated' | 'validating' | 'assets-pending' | 'schema' | 'last-step';
    gates?: GateStop[];
  }>;
  prev(): void; goTo(step: number): void;
}
```

- `assets` adds no engine members; it adds `FieldController.assets` ([08 §2.1](./08-async-data-and-registries.md#21-asset-stores)).
- `icu` adds no members; it sets the text formatter ([10 §2.1](./10-localization.md#21-formatter)).
- Calling a member of a missing built-in plugin is a type error on that engine and throws `E_PLUGIN_MISSING` at runtime (JS callers).

- `debug` is opt-in, adds only `engine.debug` and is excluded from the default built-in set; `DebugOptions`, `DebugApi` and `DebugSession` are defined in [22 §4](./22-debugging-and-devtools.md#4-opt-in-plugin-and-client-api). Passing `plugins: { debug }` options without registering it throws `E_PLUGIN_MISSING`.

Links: [03 §6.2](./03-state-model.md#62-undo--redo-opt-in), [09 §8](./09-loading-and-submission.md#8-snapshots-drafts), [11 §5](./11-rendering-and-adapters.md#5-wizards), [22](./22-debugging-and-devtools.md).

### Registry composition

`createRegistry()` and `createRegistry({ … })` create roots. `registry.extend()` creates an empty child; `registry.extend({ … })` creates a child with inferred own registrations. Both object forms accept the same categories listed in [08 §1](./08-async-data-and-registries.md#1-registries), plus plugin factories. Extension leaves the parent unchanged and uses inherited lookup and compatible overrides, not a shallow object merge. Fluent registration still mutates its receiver and returns its widened type ([21 §1](./21-type-inference.md#1-typed-registries)).

```ts
const registry = coreRegistry.extend({
  fieldTypes: { 'x-signature': signature },
  validators: { uniqueEmail },
  plugins: { 'x-audit': audit },             // keyed by factory name; arrays also supported
});
// Equivalent: coreRegistry.extend()
//   .fieldType('x-signature', signature).validator('uniqueEmail', uniqueEmail).plugin(audit)
```

There is no `createRegistry({ parent })` option or static `createRegistry.extend()` method.

## 6. Third-party plugins

Third-party plugin methods are only reachable as `engine.plugins[name]`, typed from the plugin's `Members` ([03 §8.1](./03-state-model.md#81-plugin-api)):

```ts
const audit = definePlugin('x-audit', { setup(ctx, opts: { sink: Sink }) { … } });
const registry = createRegistry({ … }).plugin(audit);
const engine = createFormEngine({ contract, registry, plugins: { 'x-audit': { sink } } });
engine.plugins['x-audit'].flush();

type AuditPlugins = PluginMap<typeof registry>;   // named type for helpers: { 'x-audit': AuditApi, … }
```

## 7. Renderers (adapter packages)

Shared by all adapters; each adapter supplies its component type `C` ([11 §4](./11-rendering-and-adapters.md#4-renderer-registry)).

```ts
defineRenderers<C>(map: {
  field?: Record<string, C>;                    // key: fieldType or `fieldType:variant`; props typed per field type
  group?: Record<string, C>;                    // key: ui.variant, or `step` / `step:variant`
  array?: Record<string, C>;                    // key: ui.variant
  'dynamic-group'?: Record<string, C>;          // falls back to group entries
  content?: Record<string, C>;                  // key: ui.variant (01 §3.6)
  fallback?: C | {                              // one component = catch-all for every kind (11 §4)
    field?: C; group?: C; array?: C; content?: C;
    'dynamic-group'?: C;                        // falls back to `group` first
    '*'?: C;                                    // catch-all for kinds not listed
  };                                            // uncovered kinds → the adapter's built-in fallback (11 §4.2)
// adapters also export DevFallback, ProdFallback, BuiltInFallback (picks by platform.dev) for reuse
}): Renderers<C>

interface Renderers<C> {
  register(component: C, tester: Tester): void;                         // 11 §4.1; later registration wins ties
  resolve(node: CompiledNode): C;                                        // cached per compiled node
  missing(schema: CompiledSchema): Array<{ fieldType: string; paths: DataPath[] }>;   // pure; field types that would fall back (11 §4.2)
}

type Tester = (node: CompiledNode, ctx: { platform: string; parent?: CompiledNode }) => number;   // -1 = no match
// helpers: rankWith, and, isFieldType, hasExt, isField
```

When rendering falls back for a field, the adapter reports `W_NO_RENDERER` once per field type ([17](./17-diagnostics.md)).

## 8. Resolved while compiling this page

- `onSubmitFailed` runs for every `{ ok: false }` result; `validating` and `assets-pending` are result reasons, not thrown diagnostics (their former catalog codes are removed).
- `engine.focus` / `engine.blur` removed; interaction is `controller.handlers`, `controller.focus()` is DOM-only.
- `engine.array` takes row keys (was indexes), shared with `ArrayController`; `replace` matches rows by `identityKey`.
- Built-in plugin members are top-level; third-party only under `engine.plugins`.
- Plugin typing is inferred from the registry and per-engine options (`FormEngineWith<R, V, O>`), not global module augmentation, so a missing/disabled plugin is a per-engine type error. Plugins are registry-recorded factories; `PluginContext.fieldTypes/validators.register` is gone (declare `registers`).
- Form status, form-level notices and `conflictCount` had no accessor; they are one `engine.form: Readable<FormState>` with `select(key | fn)`, like `controller.state`.
- `createFormEngine` takes only a `CompiledSchema`; contract problems reach the host through `LoadResult`, not a throw. `createFormEngineFrom` covers the raw-input case. `contractSchema` moved to `loadContract`; `FormState.phase` has no `compiling`.
- Locale state is part of `engine.form`: `locale` and `localeStatus` (select them like any form state); `setLocale` stays a method.
- `touch(path, { validate })` marks touched and, by default, runs blur-triggered rules that haven't run for the current value, so revealed feedback is correct.
