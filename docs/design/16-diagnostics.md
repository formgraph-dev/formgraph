# 16 · Diagnostics

[← Index](./README.md)

Every problem the contract package, compiler, or engine reports has one shape and one code from this catalog. Docs that mention a code link here.

## 1. Shape

```ts
interface Diagnostic {
  code: `E_${string}` | `W_${string}`;   // E_ = error, W_ = warning
  phase: DiagnosticPhase;               // call = thrown by a method host code called
  message: string;                       // English, for developers (not localized)
  location?: {
    pointer: string;                     // JSON Pointer into the input contract after migration, e.g. /structure/2/options/key
    schemaPath?: SchemaPath;             // after compile, e.g. addresses[*].city (03 §1)
    source?: MacroSource;                // { pointer, chain } when a macro generated the node (14 §8)
    subSchema?: {                        // set when the node comes from a dynamic sub-schema (03 §5.4); pointer is then the dynamic-group
      groupPath: SchemaPath;             // e.g. addresses[*].details
      discriminator: Json;               // which sub-schema
      pointer: string;                   // JSON Pointer into that sub-schema's nodes
    };
  };
  related?: Array<{ pointer: string; note: string }>;   // e.g. the other half of a duplicate
  hint?: string;                         // suggested fix
  detail?: Json;                         // code-specific data (e.g. a migration step's own reason code)
}
```

```ts
export const DIAGNOSTIC_PHASES = ['detect', 'migrate', 'validate-input', 'normalize', 'expand', 'validate-core', 'compile', 'runtime', 'call'] as const;
type DiagnosticPhase = typeof DIAGNOSTIC_PHASES[number];
```

- `location.pointer` always points into the input contract after migration (as authored, apart from migration rewrites). Normalization and expansion keep a pointer map from every output location back to the input (`ExpandResult.pointers`, [14 §8](./14-macros.md#8-tooling)); diagnostics from validate-core and compile are translated through it. Nodes a macro generated point at the macro node and carry `location.source`. Migration steps report the original pre-migration pointer as `related`.
- Load steps 1–7 ([01 §7.1](./01-schema-contract.md#71-contract-load-pipeline)) **collect** diagnostics and fail at the end of a step if any are errors, so authors see every problem at once, not one per run.
- `strict: true` turns every warning marked *strict → error* below into an error.
- Runtime diagnostics go to the engine's `onDiagnostic` hook (default: `console.warn` when `platform.dev` is true, silent otherwise; [10 §6](./10-rendering-and-adapters.md#6-platforms)). They never throw into a user interaction, except where the table says *throws*.
- Diagnostics are for developers. Messages shown to end users are `Notice`s ([05 §5](./05-validation.md#5-notice-shape)), which are a separate thing.

## 2. Catalog

### Detect and migrate

| Code | Phase | Meaning | Doc |
|------|-------|---------|-----|
| `E_UNSUPPORTED_VERSION` | detect | `schemaVersion` major not supported by this engine | [01 §7.1](./01-schema-contract.md#71-contract-load-pipeline) |
| `E_LIMIT_EXCEEDED` | detect / expand / validate-core / compile / call | A load limit was exceeded; the phase is where that limit is checked (`contractBytes` and the `jsonDepth`/`nodes` walk of object input in detect, `macroOutputNodes` in expand, `nodes` in validate-core, `exprDepth`/`exprSize`/`regexLength` in compile, a precompiled contract's `stats` at `createFormEngine` (call)). `detail: { limit, max, actual }` | [01 §7.4](./01-schema-contract.md#74-limits) |
| `E_MIGRATION_MISSING` | migrate | No registered migration path from the contract's major to the engine's | [01 §7.2](./01-schema-contract.md#72-migrations) |
| `E_MIGRATION` | migrate | A migration step found input it can't translate; `detail` carries the step-specific reason | [01 §7.2](./01-schema-contract.md#72-migrations) |

### Validate (input and core schema)

| Code | Phase | Meaning | Doc |
|------|-------|---------|-----|
| `E_SCHEMA` | validate-input / validate-core | Value doesn't match the contract schema (wrong type, missing required key). Carries the Standard Schema issue | [01 §7](./01-schema-contract.md#7-versioning-and-load-time-validation) |
| `E_UNKNOWN_PROPERTY` | validate-input / validate-core | Unknown non-`x-` key, contract minor ≤ engine minor | [01 §7](./01-schema-contract.md#7-versioning-and-load-time-validation) |
| `W_UNKNOWN_PROPERTY_NEWER_MINOR` | validate-input | Unknown key from a newer minor contract; dropped before validate-core | [01 §7](./01-schema-contract.md#7-versioning-and-load-time-validation) |
| `W_UNKNOWN_UI_PROPERTY` | validate-input | Unknown key inside `ui` (*strict → error*) | [01 §3.3](./01-schema-contract.md#33-core-config-and-ui) |
| `W_UNKNOWN_FIELD_TYPE` | validate-core | `x-` fieldType not registered (and not excluded by `allow`, which is `E_NOT_ALLOWED`); unknown-key checks skipped, keys kept; renders via fallback (*strict → error*) | [01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions) |
| `E_UNKNOWN_SEVERITY` | validate-core | A rule uses a severity that is neither core nor registered | [05 §7.6](./05-validation.md#76-custom-severities) |
| `W_FIELD_TYPE_FALLBACK` | validate-core | `x-` fieldType not registered; the node compiles as its contract `fallback` type. Not escalated by `strict` | [01 §3.5](./01-schema-contract.md#35-contract-fallbacks-for-custom-types) |
| `E_FALLBACK_TARGET` | validate-core | `fallback` on a core field type, or targeting a non-core type, or its config/`ui` invalid for the target | [01 §3.5](./01-schema-contract.md#35-contract-fallbacks-for-custom-types) |
| `E_FIELD_TYPE_PREFIX` | validate-core | Non-core fieldType without the `x-` prefix | [01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions) |
| `E_FIELD_CONFIG` | validate-core | Custom field type's config or `ui` fails its registered schema | [01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions) |
| `E_UNKNOWN_EVALUATOR` | validate-core | `Expression.lang` not registered | [04 §1](./04-expressions.md#1-shape) |
| `E_INVALID_LOCALE` | validate-input | Locale key isn't a BCP 47 tag (e.g. `de_DE`) | [09 §3.2](./09-localization.md#32-locale-tags) |
| `E_DUPLICATE_LOCALE` | validate-input | Two locale keys normalize to the same tag | [09 §3.2](./09-localization.md#32-locale-tags) |
| `E_INVALID_ID` | validate-input | Node id doesn't match `^[A-Za-z_][A-Za-z0-9_-]*$` | [01 §1](./01-schema-contract.md#1-principles) |
| `E_RESERVED_KEY` | validate-input | `__proto__`, `constructor` or `prototype` used as an id, data key or `serializeAs` | [01 §1](./01-schema-contract.md#1-principles) |
| `E_INVALID_MESSAGE` | validate-input / runtime | Malformed message, or outside the supported ICU subset ([09 §2.1](./09-localization.md#21-formatter)); at runtime only that key falls through | [09 §3.4](./09-localization.md#34-rules) |
| `W_MISSING_TRANSLATION` | validate-input / runtime | A key has no translation for a declared locale, so `default` shows (*strict → error at load*; for bundles from a `LocaleSource`, checked when the bundle arrives) | [09 §3.4](./09-localization.md#34-rules) |
| `W_UNUSED_TRANSLATION` | validate-input / runtime | A bundle key no node references (runtime: checked when the bundle arrives) | [09 §3.4](./09-localization.md#34-rules) |

### Expand (macros)

| Code | Phase | Meaning | Doc |
|------|-------|---------|-----|
| `E_UNKNOWN_MACRO` | expand | `macro` id not registered | [14 §2](./14-macros.md#2-contract) |
| `E_MACRO_ARGS` | expand | Macro arguments fail the macro's `args` schema | [14 §3](./14-macros.md#3-definition) |
| `E_MACRO_NESTED` | expand | Expansion output contains a macro node | [14 §5](./14-macros.md#5-rules) |
| `E_MACRO_ID_COLLISION` | expand | Ids collide within one expansion (including `ctx.expand` calls and a repeated `uniqueId` hint); cross-node collisions are `E_DUPLICATE_DATA_KEY` with `location.source` | [14 §5](./14-macros.md#5-rules) |
| `E_MACRO_DUPLICATE_VALUE` | expand | `hierarchy`: duplicate sibling value, or duplicate value across a level below a multi-select | [14 §7.1](./14-macros.md#71-hierarchy) |
| `E_MACRO_SEPARATOR` | expand | `hierarchy`: a value contains `keySeparator` | [14 §7.1](./14-macros.md#71-hierarchy) |
| `E_MACRO_TREE_DEPTH` | expand | `hierarchy`: tree deeper than `levels` | [14 §7.1](./14-macros.md#71-hierarchy) |
| `E_MACRO_CALL_DEPTH` | expand | `ctx.expand` calls nested deeper than `limits.macroCallDepth` (default 8) | [14 §5.1](./14-macros.md#51-calling-other-macros) |
| `E_MACRO_PREFIX` | call | Third-party macro id without the `x-` prefix (*throws*) | [14 §2](./14-macros.md#2-contract) |

### Compile

| Code | Phase | Meaning | Doc |
|------|-------|---------|-----|
| `E_IMPLICIT_PATH` | compile | Bare name used as a path | [03 §2](./03-paths-and-dependency-graph.md#2-path-grammar) |
| `E_PATH_OUT_OF_SCOPE` | compile | `../` past the root | [03 §2](./03-paths-and-dependency-graph.md#2-path-grammar) |
| `E_UNKNOWN_PATH` | compile | Path doesn't resolve to a node | [03 §2](./03-paths-and-dependency-graph.md#2-path-grammar) |
| `E_FIXED_INDEX` | compile | Contract path uses a fixed row index (`/items[0].qty`) | [03 §2](./03-paths-and-dependency-graph.md#2-path-grammar) |
| `E_ROOT_UNAVAILABLE` | compile | `$option`/`$selected` used outside an options `filter`/`disableWhen` or `onSelect` | [03 §2](./03-paths-and-dependency-graph.md#2-path-grammar) |
| `E_READ_ONLY_PATH` | compile | Write target is `$index`, `$context…`, `$option`, `$selected`, or contains `[*]` | [03 §2](./03-paths-and-dependency-graph.md#2-path-grammar) |
| `E_DYNAMIC_PATH` | compile | JSON Logic `var` with a computed path, `""`, or numeric segments | [04 §2](./04-expressions.md#2-evaluator-registry) |
| `E_DUPLICATE_DATA_KEY` | compile | Two nodes resolve to the same data path; `location.source` set when a macro generated one | [03 §3](./03-paths-and-dependency-graph.md#3-data-keys-and-scoping) |
| `E_UNKNOWN_REGISTRY_ID` | validate-core / compile | A `*Id` (providerId, storeId, sourceId, fn name, …) isn't registered. Names the registry | [07 §1](./07-async-data-and-registries.md#1-registries) |
| `E_REGISTRY_DUPLICATE` | call | An id registered twice in the same registry; override through a child registry (*throws*) | [07 §1](./07-async-data-and-registries.md#1-registries) |
| `E_CYCLE` | compile | Cycle among value-writing edges (computed, effects, whenHidden clear, cascade clear/reset, autoSelectSingle, onSelect fill) | [03 §5.2](./03-paths-and-dependency-graph.md#52-cycle-detection) |
| `E_NOT_ALLOWED` | expand / validate-core / compile / call | An id is referenced outside the contract's `allow` list for that registry; checked where the kind is resolved (macros in expand, evaluators/fieldTypes in validate-core, rest in compile, a precompiled contract's `refs` at `createFormEngine` (call)); takes precedence over unknown-id codes | [04 §4](./04-expressions.md#4-security-model) |
| `E_INVALID_PATTERN` | compile | A contract regex failed to compile with `registry.regex` (always with the `u` flag) | [05 §2.2](./05-validation.md#22-regex-safety) |
| `W_UNSAFE_PATTERN` | compile | Best-effort check found nested quantifiers (skipped for linear engines) (*strict → error*) | [05 §2.2](./05-validation.md#22-regex-safety) |
| `E_DYNAMIC_PATTERN` | compile | A `pattern` rule's `value` is an Expression instead of a literal | [05 §2.2](./05-validation.md#22-regex-safety) |
| `E_PROVIDER_DEPS_REQUIRED` | compile | Provider source without `options.dependsOn` | [12 §2.3](./12-option-sources-and-cascading.md#23-provider) |
| `E_TRANSFORM_FAILED` | runtime / call | A transformer threw; the owning operation fails and nothing is written. `detail: { kind, id }` | [07 §1.1](./07-async-data-and-registries.md#11-shared-function-shapes) |
| `E_AMBIGUOUS_OPTION` | compile / runtime | Child option values collide under a multi-select parent | [12 §2.2.1](./12-option-sources-and-cascading.md#221-multi-select-parents) |
| `W_ARRAY_IN_KEY` | compile | A map `key` expression turns a multi-select value into a string | [12 §2.2.1](./12-option-sources-and-cascading.md#221-multi-select-parents) |
| `W_CASCADE_UNVERIFIABLE` | compile | A cascade's validity can't be checked statically | [12](./12-option-sources-and-cascading.md) |
| `E_ON_SELECT_TARGET` | compile | `onSelect` target isn't among the field's (transitive) dependencies | [12 §7](./12-option-sources-and-cascading.md#7-reverse-fill) |
| `E_MIN_MAX` | compile | `minItems > maxItems` (or `min > max`, `minSelected > maxSelected`) | [01 §7.3](./01-schema-contract.md#73-compile) |

### Runtime and calls

| Code | Phase | Meaning | Doc |
|------|-------|---------|-----|
| `E_ASYNC_FUNCTION` | call | `functions.register` given an async function (*throws*) | [04 §5.1](./04-expressions.md#51-functions-are-synchronous) |
| `E_RESERVED_FUNCTION_NAME` | call | Function registered under a built-in JSON Logic operator name (*throws*) | [04 §5](./04-expressions.md#5-registered-functions) |
| `E_RESERVED_CONFIG_KEY` | call | Field type registered with a config key that reuses a `NodeBase`/`FieldBase` key (*throws*) | [01 §3.3](./01-schema-contract.md#33-core-config-and-ui) |
| `W_ASYNC_TRACKED` | runtime | The function passed to `observeReads` (via `useTracked`, `derived`, `watch` or an adapter bridge) returned a thenable; reads after its first `await` weren't tracked. Dev only | [02 §2.6](./02-state-model.md#26-tracking-reads-for-framework-adapters) |
| `E_WRITE_IN_DERIVED` | call | A write was made while a `derived` from `@formgraph/engine/reactive` was computing (*throws*, dev only) | [02 §2.6](./02-state-model.md#26-tracking-reads-for-framework-adapters) |
| `E_SEVERITY_NAME` | call | `severities.register` given a name without the `x-` prefix, or a core severity name (*throws*) | [05 §7.6](./05-validation.md#76-custom-severities) |
| `E_SEVERITY_GATE` | call | `severities.register` given a `gate` on a severity that doesn't extend `warning` (*throws*) | [05 §7.6](./05-validation.md#76-custom-severities) |
| `W_FALLBACK_VALUE` | runtime | A field shown through its contract `fallback` has a value the fallback type can't hold; shown read-only, value kept. `detail: { path, fieldType, fallback }` | [01 §3.5](./01-schema-contract.md#35-contract-fallbacks-for-custom-types) |
| `W_UNKNOWN_SEVERITY` | runtime | An external error, async source, or component notice used an unregistered severity; treated as `error` (fail closed). `detail: { path, severity }` | [05 §7.6](./05-validation.md#76-custom-severities) |
| `W_NO_RENDERER` | runtime | Renderer lookup fell back for a registered field type; reported by the adapter once per type. `detail: { fieldType, paths }` | [10 §4.2](./10-rendering-and-adapters.md#42-missing-renderers) |
| `W_FUNCTION_THREW` | runtime | A registered function threw or returned a thenable; facet used its fallback, validation fails closed (04 §4 table) | [04 §4](./04-expressions.md#4-security-model) |
| `W_COMPONENT_VALIDATE_FAILED` | runtime | An `onValidate` hook threw or rejected; the field gets a blocking `error` message under that reporter (fails closed). `detail: { path, reporter }` | [10 §2.2](./10-rendering-and-adapters.md#22-component-reported-messages-and-busy-state) |
| `W_EVAL_BUDGET` | runtime | An evaluation exceeded `limits.evalSteps` (deterministic, dev and prod); evaluation aborted; display facets fall back, validation and `required` yield a blocking error | [04 §4](./04-expressions.md#4-security-model) |
| `W_TICK_BUDGET` | runtime | One tick exceeded `limits.tickSteps`; the tick stopped, unevaluated facets are stale until the next tick. Names the write that started it | [03 §6](./03-paths-and-dependency-graph.md#6-scheduler) |
| `E_EFFECT_DEPTH` | runtime | Effect chain deeper than `limits.effectDepth` (default 8); writes past the limit dropped; `detail` carries the chain | [03 §6](./03-paths-and-dependency-graph.md#6-scheduler) |
| `E_SNAPSHOT_MISMATCH` | call | `restore` given a snapshot for another `formId`, or a snapshot `version` with no migration | [08 §8.2](./08-loading-and-submission.md#82-restore-rules) |
| `W_SNAPSHOT_PATH_DROPPED` | runtime | A snapshot path no longer exists in the contract | [08 §8.2](./08-loading-and-submission.md#82-restore-rules) |
| `W_SNAPSHOT_ASSET_DROPPED` | runtime | `snapshot()` left out an asset value whose status isn't `ready` | [08 §8](./08-loading-and-submission.md#8-snapshots-drafts) |
| `E_SNAPSHOT_INVALID` | call | Snapshot fails shape validation | [08 §8.2](./08-loading-and-submission.md#82-restore-rules) |
| `E_RESTORE_BUSY` | call | `restore` called while `phase` is `submitting` | [08 §8.2](./08-loading-and-submission.md#82-restore-rules) |
| `E_SUPERSEDED` | call | A pending `reload`/`restore` was superseded by a later call; it writes nothing | [08 §3](./08-loading-and-submission.md#3-reload-policy) |
| `E_UNDEFINED_VALUE` | call | `setValue` given `undefined`; use the empty value or `revert` | [02 §3](./02-state-model.md#3-precedence-by-presence-not-nullishness) |
| `E_PLUGIN_MISSING` | call / compile / runtime | Names the plugin and its import. *call*: method of a plugin that isn't installed (slim entry), or a plugin's `requires` is unmet (*throws*). *compile*: the contract needs `icu` (a message uses plural/select/selectordinal/number/date/time) or `assets` (an `asset`/`multi-asset` field). *runtime*: a `LocaleSource` bundle needs `icu`; that key falls through | [02 §8](./02-state-model.md#8-plugins) |
| `E_PLUGIN_DUPLICATE` | call | Two plugins with the same `name` in `plugins` (*throws*) | [02 §8.1](./02-state-model.md#81-plugin-api) |
| `E_RESUME_MISMATCH` | call | `resume` state fails shape validation, or its `formId`/`contractHash` doesn't match the contract (*throws*; render client-only without `resume`, [15 §19](./15-cookbook.md#19-server-rendered-form)) | [08 §9](./08-loading-and-submission.md#9-server-rendering) |
| `W_RESUME_SCHEMA_CHANGED` | runtime | A dynamic sub-schema fetched again during resume has a different hash than the exported one | [08 §9](./08-loading-and-submission.md#9-server-rendering) |
| `W_RESUME_SCHEMA_REJECTED` | runtime | A sub-schema from `resume` failed a load check or its recomputed hash; it was dropped and fetched again | [08 §9](./08-loading-and-submission.md#9-server-rendering) |
| `W_RESUME_PLUGIN_UNKNOWN` | runtime | `resume` has state for a plugin that isn't installed, or a plugin rejected its state; ignored, the plugin starts from defaults | [08 §9](./08-loading-and-submission.md#9-server-rendering) |
| `E_SERVER_UNSUPPORTED` | call | `submit()` or another client-only call in server-safe mode | [08 §9](./08-loading-and-submission.md#9-server-rendering) |
| `E_UNKNOWN_ORIGIN` | call | `setValue` given a built-in or unregistered origin | [06 §2.1](./06-effects-and-computed.md#21-origin-policy) |
| `E_RESERVED_ORIGIN` | call | Registering an origin without the `x-` prefix, with layer `loaded`/`default`, or extending `user` | [06 §2.1](./06-effects-and-computed.md#21-origin-policy) |
| `E_INVALID` | call | `serialize({ validate: true })` with error-severity messages (*throws*) | [08 §4](./08-loading-and-submission.md#4-submission-pipeline) |
| `E_ASSET_FAILED` | runtime | An asset ended in `error`; blocks submit until retried or removed | [07 §2.1](./07-async-data-and-registries.md#21-asset-stores) |

## 3. Rules for new codes

- A code is never reused for a different meaning. Removed codes stay reserved.
- New codes are added to this catalog in the same change that introduces them.
- Macros and plugins use their own prefix after the letter: `E_X_<PLUGIN>_…` / `W_X_<PLUGIN>_…`.
