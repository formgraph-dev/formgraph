# 08 · Async Data & Registries

[← Index](./README.md) · Addresses review items B4, B5

## 1. Registries

All registries share one shape: a typed `Map<string, T>` with `register`, `get`, `has`. The registry also has typed registration methods (`fieldType`, `validator`, `severity`, …, or one object literal to `createRegistry` / `registry.extend`) that preserve inferred entries, so contracts can be type-checked against it ([21 §1](./21-type-inference.md#1-typed-registries)); `register` itself is untyped. Everything is registered on a `Registry`, never on an engine; `loadContract` and `createFormEngine({ registry })` receive it.

- **Append-only.** Registering a new id is always allowed; registering an id that already exists in the same registry throws `E_REGISTRY_DUPLICATE`, so two plugins can't silently override each other. A compiled schema keeps what it resolved; later registrations don't change running engines.
- **Overrides use a child.** `registry.extend()` creates a child that looks up its own entries first, then the parent's, so it can compatibly replace a parent id (tests, tenants, overriding a built-in validator). `registry.extend({ fieldTypes: { … }, validators: { … }, plugins: { … } })` registers entries on the new child in one call, with the same categories, inference and validation as `createRegistry({ … })` and fluent registration. Neither form modifies the parent. Duplicate own ids still fail; inherited ids use the existing compatible-override rules ([21 §1](./21-type-inference.md#1-typed-registries)). `createRegistry` creates roots; inheritance uses the instance's `extend`, not a `parent` option.
- **Per engine.** `createFormEngine` wraps the host's registry in its own child; plugins ([03 §8.1](./03-state-model.md#81-plugin-api)) are recorded on the registry as factories (`registry.plugin(factory)`, which also registers the types and validators they declare), and each engine makes its own instances, so several engines can share one host registry without sharing plugin state.

| Registry | Key in contract | Implementation type | Allow-listable |
|----------|-----------------|---------------------|----|
| `fieldTypes` | `fieldType` (`x-*`) | `{ config, ui, value, empty, payloadSchema? }` (01 §3.4) | yes |
| `macros` | `MacroNode.macro` | `Macro` (15 §3) | yes |
| `origins` | `setValue({ origin })` (`x-*`) | `{ extends: Origin } & Partial<OriginPolicy>` (07 §2.1) | no |
| `severities` | `severity` on rules, notices, external errors (`x-*`) | `{ extends: CoreSeverity; rank?: number; gate?: string }` (06 §7.6) | no |
| `migrations` | `schemaVersion` major | `Migration` (01 §7.2) | no |
| `evaluators` | `Expression.lang` | `Evaluator` (05) | yes |
| `functions` | `fn` name / JSON Logic op | `(…args) => Json` | yes |
| `validators` | `ValidationRule.type` | `Validator` (06) | yes |
| `validationSources` | `AsyncValidationRule.sourceId` | `ValidationSource` (06) | yes |
| `providers` | `ProviderOptions.providerId` | `OptionsProvider` (below) | yes |
| `queryTransformers` | `queryTransformerId` | `QueryTransformer` (§2) | yes |
| `resultTransformers` | `resultTransformerId` | `ResultTransformer` (§2) | yes |
| `selectedResolvers` | `resolveSelectedSourceId` | `SelectedResolver` (§2) | yes |
| `schemaSources` | `DynamicGroup.schemaSourceId` | `SchemaSource` (§5) | yes |
| `entitySources` | `LoadConfig.entitySourceId` | `EntitySource` (below, 09 §2) | yes |
| `inboundTransformers` | `LoadConfig.transformerId` | `InboundTransformer` (below, 09 §2) | yes |
| `outboundTransformers` | `SubmissionConfig.transformerId` | `OutboundTransformer` (below, 09 §4) | yes |
| `submitTargets` | `SubmissionConfig.targetId` | `SubmitTarget` (below, 09 §4) | yes |
| `assetStores` | `AssetConfig.storeId` | `AssetStore` (§2.1) | yes |
| `localeSources` | `FormContract.localeSourceId` | `LocaleSource` (10 §3.3) | yes |
| `textFormatters` | `ui.textFormat` | `Transformer<{ text: string }, unknown>` (§1.1; renderer-specific output, 10 §2) | yes |
| `effectHandlers` | `Effect.action` (`x-*`) | `EffectHandler` | yes |
| `renderers` | `fieldType` / `ui.variant` / `step` | adapter-specific (11) | no |
| `regex` | — (all contract patterns) | `RegexEngine` (06 §2.2); a single slot, not a map, inherited from the parent | no |

A contract id with no matching registration is `E_UNKNOWN_REGISTRY_ID` at compile time, naming the registry, except for macros (`E_UNKNOWN_MACRO`, expand), evaluators (`E_UNKNOWN_EVALUATOR`, validate-core), field types (`W_UNKNOWN_FIELD_TYPE`), severities and migrations, which have their own codes ([17](./17-diagnostics.md)). Allow-listable registries can be restricted per contract with `allow` (`E_NOT_ALLOWED` takes precedence, [05 §4](./05-expressions.md#4-security-model)).

Every `*Source`, `provider`, `store`, and `target` is just a registered function. It may read a network API, a bundled JSON file, the file system, IndexedDB, a native module, a worker, or an in-memory array. The engine only sees its inputs and outputs (§4).

### 1.1 Shared function shapes

Every async source (providers, selected resolvers, schema, entity, validation and locale sources, submit targets) has one shape:

```ts
interface SourceCtx { signal: AbortSignal; context: Json; formId: string; locale: LocaleTag }
type AsyncSource<I, O> = (input: I, ctx: SourceCtx) => O | Promise<O>;

type EntitySource = AsyncSource<{ id: Json }, Json>;                                    // 09 §2
interface SubmitTargetResult {
  errors?: ExternalErrors; entity?: Json;       // errors: [] means accepted; any present JSON entity is applied
  acknowledgedPaths?: DataPath[];               // exact frozen-form coverage; required with an outbound transformer (09 §4.2)
}
type SubmitTarget<P = Json> = AsyncSource<{ payload: P; mode: 'full' | 'patch'; confirmed: string[]; serializedPaths: DataPath[] }, SubmitTargetResult>; // 09 §4; serializedPaths: core coverage before transformation, not a claim about the transformed payload
```

The engine wraps every source call the same way: it aborts `signal` when the call is superseded (§3), applies the result only if its generation still matches, treats a synchronous return like a resolved Promise (applied in a later tick), and normalizes a thrown error or rejection into the call's error state and reports it.

Transformers are synchronous and share one generic:

```ts
interface TransformCtx { context: Json; formId: string; locale: LocaleTag }
type Transformer<I, O> = (input: I, ctx: TransformCtx) => O;   // synchronous

type InboundTransformer  = Transformer<{ raw: Json; mapped: Json }, Json>;              // returns values, 09 §2
type OutboundTransformer<P = Json, O = Json> = Transformer<{ payload: P; mode: 'full' | 'patch'; loaded: Json; values: Json; dirtyPaths: DataPath[]; confirmed: string[] }, O>;   // 09 §4; P/O inferred for typed contracts (21 §3)
```

An outbound transform may omit, rename or synthesize data. Its paired target must return `acknowledgedPaths` identifying the frozen form inputs actually persisted, not guess coverage from output key names or from a successful request. Without a transformer, omitted coverage defaults to `serializedPaths`; explicit coverage can restrict it. `[]` explicitly acknowledges no units; it is not omitted metadata. `serializedPaths` is local integration metadata, not automatically part of the backend request. Missing required or invalid coverage is a saved-but-unreconciled `E_SUBMIT_RESPONSE`, never permission to clear all input edits. The caller may correct coverage via `reconcileSubmit` without saving again ([09 §4.2](./09-loading-and-submission.md#42-acknowledgement-and-response-reconciliation)).

A transformer that throws fails the operation that owns it, writes nothing and never changes values; it is reported once as `E_TRANSFORM_FAILED` with `detail: { kind, id }`:

| Kind | Outcome |
|------|---------|
| query | option list `status: 'error'`, provider not called, value left alone |
| result | option list `status: 'error'`, previous options kept, response discarded, value left alone |
| inbound | the load fails: `ready` or `reload()` rejects, `loaded` unchanged |
| outbound | `submit()` returns `{ ok: false, reason: 'transform' }`, the target isn't called |

An inbound failure while applying an **accepted submit response** is different from a failed save: the response plan is discarded, `E_SUBMIT_RESPONSE` is reported, and submit returns `ok: true, response: 'failed'` with diagnostics and a reconciliation id ([09 §4](./09-loading-and-submission.md#4-submission-pipeline)). The host retries `reconcileSubmit` (optionally with a refetched entity), not a generic preserve-edits reload or the already-accepted target call.

`QueryTransformer` and `ResultTransformer` are in §2. The four transformer registries stay separate so each kind's signature can change independently and their ids don't share a namespace.

```ts
const registry = globalRegistry.extend();
const engine = createFormEngine({ contract, registry, context });
```

## 2. Provider options

Options supplied by a registered **provider** (`source: 'provider'`, [13 §2.3](./13-option-sources-and-cascading.md#23-provider)):

```ts
interface ProviderOptions extends Extensible {
  providerId: string;
  queryTransformerId?: string;       // default 'identity'
  resultTransformerId?: string;      // default 'identity'
  initialState?: Json;               // opaque pagination/cursor state
  dependsOn?: PathRef[];             // change → reset & reload
  trigger?: 'mount' | 'open' | 'query';   // default 'open'
  queryDebounceMs?: number;          // default 250
  cache?: { ttlMs?: number; key?: 'deps' | 'deps-query' };
}
```

What happens to the field's value when a dependency changes (`cascade.onChange`), and the static and map sources, are covered in [13 · Option Sources & Cascading](./13-option-sources-and-cascading.md).

```ts
type QueryTransformer  = Transformer<{ state: Json; query?: string; deps: Record<string, Json> }, Json | null>;  // null = skip
type OptionsProvider   = AsyncSource<Json, unknown>;   // input = the transformed query
type ResultTransformer = Transformer<{ raw: unknown; state: Json; query?: string; deps: Record<string, Json>; request: Json },  // request = the query transformer's output
  { options: SelectOption[]; nextState: Json | null; total?: number; selected?: SelectOption[] }>;
```

- The **query is opaque `Json`.** The engine never interprets it. For an HTTP provider it might be `{ path, params }`, for a file provider a file name, and for IndexedDB a key range. The default `identity` query transformer passes `{ state, query, deps }` through.
- A provider may return a value directly (for example from an in-memory table) or a Promise. The engine treats both the same way: the result is applied in a new tick, so option loading never blocks typing.
- `nextState === null` means there are no more pages. The engine never interprets the state, so offset, cursor, time window, or geo radius pagination all fit.

A query transformer can return `null` to **skip the call** (for example a partial postcode). Options become `[]` with `status: 'idle'` and any in-flight load is aborted. The field's value is **left alone**: no cascade invalidation runs and the validity check is deferred until a real query produces options ([13 §4](./13-option-sources-and-cascading.md#4-invalidation)).

Option state lives in `meta[path].options` (internal: `{ items, state, status, error, query, generation }`). It isn't stored in values. Renderers see it through the `OptionsState` handle defined in [13 §9](./13-option-sources-and-cascading.md#9-renderer-view).

**Selected-label problem:** a loaded value like `"US"` may not be on the current page of options. Result transformers can return `selected?: SelectOption[]`, or the field can set `resolveSelectedSourceId` to load labels for the current values. Until a label is resolved, renderers show the raw value.

```ts
type SelectedResolver = AsyncSource<{ values: Scalar[]; deps: Record<string, Json> }, SelectOption[]>;
```

### 2.1 Asset stores

Provided by the `assets` plugin ([03 §8](./03-state-model.md#8-plugins)), which the default entry installs. It registers the reserved `asset` / `multi-asset` types, the `accept` / `max-bytes` validators and the field's `AssetController`. On slim core without it, a contract with an asset field fails at compile with `E_PLUGIN_MISSING`. Asset fields hand files to a registered store ([01 §3.3](./01-schema-contract.md#33-core-config-and-ui)):

```ts
type PlatformFile = { uri: string; name: string; size: number; mime: string };   // RN/Tauri/native file handle
type AssetStore = (file: Blob | PlatformFile, ctx: SourceCtx & { onProgress(p: number): void })
  => Promise<{ ref: string; meta?: Json }>;   // ctx as in §1.1, plus progress
```

`ref` is an opaque reference chosen by the store: an upload id, a URL, a local file path, an IndexedDB key, or a content hash. A store may also transform the file (resize, scan, encrypt) before returning.

Lifecycle of an `AssetRef`:

| Status | Meaning | Set by |
|--------|---------|--------|
| `pending` | Selected, not yet handed to the store (queued behind a concurrency limit, or waiting until online) | The engine, when the renderer adds a file |
| `processing` | The store is running. `progress` updates through `onProgress` | The engine, when it calls the store |
| `ready` | The store resolved. `ref` and `meta` are set | The engine, from the store's result |
| `error` | The store threw or was aborted for a reason other than removal. `error` holds the message. The renderer offers retry (back to `pending`) or remove | The engine |

Loaded assets arrive as `ready`. Removing a file aborts its store call. Renderers drive this through the field's `AssetController` ([11 §2](./11-rendering-and-adapters.md#2-headless-field-api)).

```ts
interface AssetConfig {   // 01 §3.3; adds:
  concurrency?: number;   // max store calls in flight for this field; default 3. Extra files wait as 'pending'
}
```

- **Submit and `next()`**: the plugin implements the core pending-work gate ([01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions)), reporting `busy` for `pending` / `processing` and `failed` for `error`. Core treats `busy` like in-flight async validation ([06 §4](./06-validation.md#4-async-validation)): with `submission.awaitAsync` (default `true`) it waits; with `awaitAsync: false` it returns `{ ok: false, reason: 'assets-pending' }`. `failed` blocks submit like a validation error (`E_ASSET_FAILED` on the field) until retried or removed.
- **Snapshots** (through the core `toSnapshot` hook) keep only `ready` refs. `pending`, `processing`, and `error` entries hold local file handles that can't be serialized, so they are left out, and the user re-adds those files after `restore` ([09 §8](./09-loading-and-submission.md#8-snapshots-drafts)).

## 3. Load lifecycle

Every async call the engine makes (option loads, async validation, schema loads, asset stores) has:
- an `AbortController`, aborted when a newer call supersedes it, on dependency change, on field unmount (if `trigger:'open'`), and on engine `dispose()`,
- a **generation** number. A result is only applied if `generation === meta.options.generation` (or the equivalent counter for that call),
- for option loads, a **cache key** (below). In-flight loads with the same key are deduplicated.

**Option cache key.** The key is built from what is actually sent: `hash(providerId, transformedQuery)`, where `transformedQuery` is the query transformer's output (which already contains whatever `state`, `deps`, and search text the transformer chose to use). `cache.key` picks how much of the raw input may vary the key when a transformer drops it:

| `cache.key` | Key |
|-------------|-----|
| `deps` (default) | `hash(providerId, transformedQuery, deps)`; the search text is only in the key if the transformer put it in the query |
| `deps-query` | `hash(providerId, transformedQuery, deps, query)`; search text always separates entries |

**Locale and context.** A provider can return different results for a different `ctx.locale` or `ctx.context` (localized labels, role- or tenant-filtered lists). Each provider declares what it varies on when registered:

```ts
registry.providers.register('countries', fn, { varies: ['locale'] });   // default ['locale', 'context']
```

- Every declared input is added to the cache key: `hash(providerId, transformedQuery, …, locale?, context?)`.
- `setLocale` (after the switch, [10 §3.3](./10-localization.md#33-loading-bundles-separately)) and `setContext` reload the open option lists whose provider varies on that input; closed lists reload on next open. Values are handled like any option reload ([13 §4](./13-option-sources-and-cascading.md#4-invalidation)).
- `varies: []` opts out for providers that ignore both, so their entries survive locale and context changes.
- `selectedResolvers` take the same option and follow the same rule.

`cache.ttlMs` expires entries (default: for the life of the engine). A `null` query (skip) is never cached or sent.

**Shared (deduplicated) loads.** Several fields or rows asking for the same key share one provider call:
- The shared call keeps a **subscriber count**. A subscriber leaves when its field supersedes the request (new deps or query), unmounts under `trigger: 'open'`, or is disposed. The call is aborted only when **no subscribers are left**.
- Each subscriber keeps its own generation. When the shared result arrives, it is applied only to subscribers whose generation still matches the one they joined with; the rest ignore it.

On a `dependsOn` change: abort, reset `state` to `initialState`, clear `items`, then reload if the trigger condition is met. The field's value is handled by the cascade rules in [13 §4](./13-option-sources-and-cascading.md#4-invalidation).

Errors set `status:'error'`, and the view exposes `retry()`. Retry policy (backoff) belongs to the provider, not the engine.

**Server rendering.** `exportState()` includes settled option cache entries and loaded sub-schemas, and `resume` seeds the cache with them, so the client makes no fetch the server already made ([09 §9](./09-loading-and-submission.md#9-server-rendering)). Entries keep their `ttlMs`, counted from the client's resume. On the server, debounced and async-validation calls aren't started.

## 4. The engine does no I/O

The engine never touches the network, the file system, or storage. All I/O goes through registered functions (§1). This keeps the core platform-neutral (web, React Native, desktop, workers, tests), and it lets hosts reuse their own clients, auth, caches (for example TanStack Query), offline storage, and fixtures.

## 5. Dynamic sub-forms

```ts
type SchemaSource = AsyncSource<{ discriminator: Json }, InputNode[]>;   // may contain macros (01 §5.1)
```

Lifecycle when the discriminator value changes:
1. Abort the pending load and bump the generation.
2. Mark the group `status:'loading'`. The previous children stay rendered but are disabled, which avoids layout flash. Policy: `loadingBehavior: 'keep-stale' | 'empty'` on the node.
3. On resolve (if the generation matches): run the sub-schema through the detect size check (bounded walk) and load steps 3–7 ([01 §7.1](./01-schema-contract.md#71-contract-load-pipeline): validate, normalize, expand, validate, compile; no migration, it shares the parent's `schemaVersion`), **graft** it into the CompiledSchema under the group's path, and extend the DAG. Cycle detection runs on the merged subgraph.
4. Old values under the group scope follow the `orphanPolicy`:
   - `discard` (default): remove the old sub-schema's keys from **every layer** (`default`, `loaded`, and `current`) plus `meta` and `errors`, so a loaded value can't resurface if a later sub-schema declares the same key. Removing them from `loaded` also means they are no longer in the baseline, so they don't count as dirty and aren't sent as removed in `patch` mode.
   - `stash`: move them (from every layer, as above) to `meta.stash[groupPath][oldDiscriminator]`, and restore them if the user switches back.
5. Defaults from the new sub-schema are merged into `default`.
6. Results are cached per discriminator value, at most `schemaCache` (default 10) per group, least recently used evicted. The new nodes' controllers, validators and option state compile lazily; their graph edges and defaults are added eagerly in the graft ([03 §6.3](./03-state-model.md#63-lazy-compile)).

If attaching would exceed `limits.nodes` (counting attached sub-schemas only, [01 §7.4](./01-schema-contract.md#74-limits)), or the result fails any load step, the group gets `status: 'error'` with that diagnostic (e.g. `E_LIMIT_EXCEEDED`), and the previous sub-schema stays attached.

Keeping the old schema is a display-only recovery measure: it does not match the new discriminator and cannot satisfy submission readiness. `submit()`, scoped `validate()` and wizard `next()` await schema loads in their eligible scope, then install a blocking group notice and fail with `reason: 'schema'` (validation returns `ok: false` with that notice) if any selected schema is failed or mismatched. The notice is installed **before every failure returns**, including submit's pre-validation check, so `firstInvalid()` can focus the group; it clears on successful matching-schema attachment or discriminator recovery. They never serialize the new discriminator with the old children. `DynamicGroupController.retry(): Promise<void>` retries the current discriminator through the same generation-checked pipeline; renderers expose it alongside the group's status/diagnostic. Switching back to a successfully cached discriminator also recovers.

**The graft is a trusted structural tick.** Steps 3–5 commit as one transaction that behaves like origin `reset` ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)): no effects and no cascade invalidation run for the removed or defaulted values (the new children start from their own defaults, so there is nothing to invalidate). Computed values, conditions, and option sources of the new children are evaluated normally.

**History.** The graft commit carries the history flag `clear` ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)). The earlier discriminator write was a normal recorded step, but replaying it can't bring back a sub-schema whose values were already discarded from every layer.

The same mechanism handles loading: when a loaded entity already has a discriminator, the schema load starts **during** loading, and the form only becomes `ready` once both have resolved. `restore` instead plans discriminator conflicts first and stages schema loads/grafts off-store; only its final atomic commit changes the live discriminator and schema together ([09 §8.2](./09-loading-and-submission.md#82-restore-rules)).
