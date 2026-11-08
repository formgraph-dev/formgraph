# 11 · Open Questions & Roadmap

[← Index](./README.md)

## Open questions

| # | Question | Options / leaning |
|---|----------|-------------------|
| ~~Q1~~ | ~~Default `whenHidden`?~~ | **Resolved:** hidden = `keep`, disabled = `exclude`. Configurable form-wide via `FormContract.policies`, overridable per container and per field ([06 §4](./06-effects-and-computed.md#4-hidden-and-disabled-field-policy)) |
| ~~Q2~~ | ~~Bare-name path resolution?~~ | **Resolved:** every reference must be explicit (`/`, `./`, `../`, `$…`). A bare name is a compile error ([03 §2](./03-paths-and-dependency-graph.md#2-path-grammar)) |
| ~~Q3~~ | ~~Should JSONata ship as an official plugin?~~ | **Resolved: no.** JSON Logic is the only shipped language. The evaluator registry stays open for third parties, but evaluators must provide exact `collectDeps` and be synchronous ([04 §2](./04-expressions.md#2-evaluator-registry)) |
| ~~Q4~~ | ~~Should `patch` submission treat arrays atomically, or diff them with JSON Patch?~~ | **Resolved: atomic.** Hosts that need row diffs compute them in the outbound transformer ([08 §4](./08-loading-and-submission.md#4-submission-pipeline), [15 §17](./15-cookbook.md#17-row-level-diffs-for-patch-submissions)) |
| ~~Q5~~ | ~~Undo/redo?~~ | **Resolved: built in, opt-in** (`plugins: { history: { limit } }`). One step = one user tick plus its cascades. Typing is coalesced. Load/reset carry origin policy `history: 'clear'`, which the history plugin acts on ([02 §6.2](./02-state-model.md#62-undo--redo-opt-in)) |
| ~~Q6~~ | ~~Collaborative or live external changes into `current`?~~ | **Resolved: per-path external updates with conflict flags** (`applyExternal`, `resolveConflict`). No text merging or presence ([08 §7](./08-loading-and-submission.md#7-external-updates-and-conflicts)) |
| ~~Q7~~ | ~~Should computed values be submitted?~~ | **Resolved:** yes, they are values. Hidden helpers use `whenHidden: 'exclude'`. Visible but not submitted fields are removed by the outbound transformer ([15 §13](./15-cookbook.md#13-fields-that-arent-submitted)). No `submit: false` flag |
| ~~Q8~~ | ~~Schema-level i18n bundles (contract ships its own catalogs)?~~ | **Resolved: (a), optional `locales` bundles in the contract** (BCP 47 keys, `translations` inside), plus a `localeSources` registry for loading bundles separately. Host resolver wins; locale switch re-resolves text only ([09 §3](./09-localization.md#3-contract-locales)). Rejected: per-locale contracts from the CMS (structure can drift between locales), and deferring |
| ~~Q9~~ | ~~Valibot or Zod for the runtime contract validator?~~ | **Resolved:** the engine depends only on the [Standard Schema](https://standardschema.dev) interface. The default contract schema's implementation library is an internal detail. Field validators also accept Standard Schemas ([05 §2.1](./05-validation.md#21-standard-schema-validators)) |
| ~~Q10~~ | ~~Nested `map` option sources for a whole hierarchy in one field?~~ | **Resolved: a `hierarchy` macro**, no core change. It expands to per-level `map` sources keyed by joined ancestor values, so submitted values stay plain and repeated names are fine ([14 §7.1](./14-macros.md#71-hierarchy)) |
| ~~Q11~~ | ~~Group child options by parent when the parent is a multi-select?~~ | **Resolved:** child values colliding across selected parents are `E_AMBIGUOUS_OPTION` (load time where possible, otherwise runtime). Opt-in `groupBy: 'parent'` adds display-only `ResolvedOption.group` ([12 §2.2.1](./12-option-sources-and-cascading.md#221-multi-select-parents)) |
| Q12 | Incremental aggregates? | `sum`/`count`/`min`/`max` are O(rows) JSON Logic operators ([06 §3.2](./06-effects-and-computed.md#32-aggregates)). Revisit as a plugin with delta maintenance only if the M1 benchmark (100 rows) misses its budget |

## Roadmap

### M0: Foundations
- `packages/contract`: input and normalized types, JSON Schema generation, Standard Schema contract validator, the migration mechanism (no steps yet, [01 §7.2](./01-schema-contract.md#72-migrations)), load pipeline steps 1–4 and 6 ([01 §7.1](./01-schema-contract.md#71-contract-load-pipeline)), and diagnostics ([16](./16-diagnostics.md)). No macros, no `toPayloadSchema`.
- Path grammar ([03 §2](./03-paths-and-dependency-graph.md#2-path-grammar)), compile to CompiledSchema, `Limits` and `allow` ([01 §7.4](./01-schema-contract.md#74-limits), [04 §4](./04-expressions.md#4-security-model)).
- Monorepo tooling: pnpm, a library bundler (tsup is unmaintained; evaluate tsdown), Vitest, Changesets, Biome, and a CI bundle-size check against the budgets in the [README](./README.md). Core and contract build with `lib: ["ES2020"]` (no `dom`), a lint rule bans platform globals, and CI runs core tests under Node and Hermes (D24, [10 §6](./10-rendering-and-adapters.md#6-platforms)).

### M1: Vertical slice
- Value layers with presence precedence, `FormStore` transactions, commit entries and `CommitHooks` ([02 §2.5](./02-state-model.md#25-commit-hooks)), path-indexed `Readable`s, structural sharing. Row keys, array operations, `getValues`, `setValue`, array `set`/`replace`.
- Read observer (`observeReads`/`untracked`, [02 §2.6](./02-state-model.md#26-tracking-reads-for-framework-adapters)) in the core, plus `engine.batch`. Adapter bridges and the optional `@formgraph/engine/reactive` subpath come with their adapters.
- `ORIGIN_POLICY` table and origin plumbing on every write, plus the effect loop. Declarative effect actions stay M3 ([06 §2.1](./06-effects-and-computed.md#21-origin-policy)).
- Plugin host API ([02 §8.1](./02-state-model.md#81-plugin-api)) and the `@formgraph/engine/core` vs default entry split. The default entry equals core until plugins land (`assets` M2, `icu` M3, the rest M4).
- Full `FieldState` type with stubs: text via a passthrough resolver, options empty until M2, computed until M3.
- DAG, cycle detection, scheduler.
- JSON Logic evaluator with dependency inference, step budget, `fn` registry.
- Conditions and hidden policy.
- Sync validation, implied rules ([01 §3.3.1](./01-schema-contract.md#331-implied-rules)), severity, and the error model, including `setInputError`. Regex engine hook ([05 §2.2](./05-validation.md#22-regex-safety)).
- Minimal React adapter: `useField`.
- Realistic benchmark: 1,000 fields / 100 rows, including dependents and validation. Keystroke tick under 1ms p95 on M1-class hardware.
- Property tests (precedence, array operations) and contract fuzzing (random valid contracts → no crashes, no cycles missed) start here and grow with each milestone.

### M2: Async and data
- Registries, provider options with pagination and cancellation, async cascade validity checks, reverse fill.
- Async validation.
- `assets` plugin: asset stores, pending-work gate and `toSnapshot` field-type hooks ([07 §2.1](./07-async-data-and-registries.md#21-asset-stores)).
- Static and map option sources, cascade invalidation, `in-options`, `whenEmpty`, auto-select, multi-select parents with `E_AMBIGUOUS_OPTION` and `groupBy` ([12](./12-option-sources-and-cascading.md)).
- Loading, reload policies, submission, external error mapping.
- External updates and conflicts (`applyExternal`, [08 §7](./08-loading-and-submission.md#7-external-updates-and-conflicts)).
- `EngineState` shape and server-safe mode, designed alongside loading and options.
- Dynamic sub-forms ([07 §5](./07-async-data-and-registries.md#5-dynamic-sub-forms)).

### M3: Behavior and authoring
- Computed values and effects with origins, in full.
- Macro registry and `expandContract`: normalization, expansion, `ctx.expand` with the depth and output limits, and the `sources` side map ([14](./14-macros.md)). The `hierarchy` macro ships as the reference macro ([14 §7.1](./14-macros.md#71-hierarchy)). `@formgraph/macros` standard library, each macro tested against its cookbook long form ([15](./15-cookbook.md)).
- `toPayloadSchema` ([08 §6](./08-loading-and-submission.md#6-json-schema-export-of-the-submitted-data)).
- `fromJsonSchema` importer ([01 §8](./01-schema-contract.md#8-json-schema-import)).
- Localization: resolver chain, core `{name}` formatting, `icu` plugin (`@formgraph/engine/icu`, [09 §2.1](./09-localization.md#21-formatter)), contract `locales`, `localeSources`, BCP 47 normalization, `setLocale` ([09](./09-localization.md)).

### M4: Plugins and ecosystem
- Undo/redo history ([02 §6.2](./02-state-model.md#62-undo--redo-opt-in)).
- Snapshots and drafts (`snapshot`/`restore`, [08 §8](./08-loading-and-submission.md#8-snapshots-drafts)).
- Wizard controller.
- Adapter SSR glue only (`getServerSnapshot` etc.), on top of the M2 server-safe mode.
- Full React adapter and reference unstyled renderers with ranked tester resolution; then Vue, Solid, Svelte. Adapter conformance suite.
- `@formgraph/testing`.
- Devtools.

### M5: Hardening and 1.0
- Hardening: fuzzing and property-test coverage, performance regressions.
- Security review (limits, regex, `allow`, trust boundary, [04 §4](./04-expressions.md#4-security-model)).
- Docs site with a live contract playground.
- 1.0 release: `schemaVersion` `1.0` ([01 §7](./01-schema-contract.md#7-versioning-and-load-time-validation)).
