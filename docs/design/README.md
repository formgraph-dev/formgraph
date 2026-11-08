# formgraph: Design

Status: **Draft**, pre-1.0 (nothing released yet) · Based on: the v4.0 handoff draft (never published)

A headless, metadata-driven form engine. A backend or CMS sends a JSON **contract**, a framework-agnostic **core** (dependency-free TypeScript with its own transactional store) owns all state and behavior, and thin **adapters** bind it to React, Vue, Solid, Svelte, or native renderers.

Start with [02 · Big Picture](./02-big-picture.md) for the main pieces, their responsibilities and how they fit together.

## Goals

- **Headless.** The core has no DOM or UI imports. Rendering is entirely the host's job.
- **Contract-driven.** Forms are data. A form that only uses built-ins can be shipped without a client release.
- **Extensible by registry, not by enum.** Field types, validators, expression evaluators, option providers, stores, and transformers are all string IDs resolved against client-registered implementations.
- **Safe by default.** Contracts can't run arbitrary code on clients.
- **Predictable state.** Stacked value layers, one derived read model, and explicit write origins.
- **Fast.** A keystroke costs O(depth of path) plus the work of its dependents, not O(form size).

## Non-goals

- Visual form builder or CMS UI.
- Executing external (server-side) validation. The engine only calls registered validation sources.
- Shipping styled components. Adapters may ship *unstyled* reference renderers.
- Offline sync or conflict resolution (hooks only).

## Architecture at a glance

```
            ┌──────────────── Contract (JSON) ───────────────┐
            │ structure · conditions · validations · effects │
            └───────────────────────┬────────────────────────┘
                                    │ detect → migrate → validate input → normalize → expand → validate core → compile
                                    ▼
 ┌───────────────────────────── Core (vanilla TS) ──────────────────────────────┐
 │  Compiled Schema  ──►  Dependency Graph (DAG)  ──►  Scheduler (batched ticks) │
 │        │                                                │                     │
 │        ▼                                                ▼                     │
 │  FormStore  layers: default · loaded · current (values)                       │
 │             slices: meta · notices · form                                     │
 │  transactions · commit hooks · lazy resolved cache · path-indexed subscribers │
 │                                                                               │
 │  Registries (08 §1): fieldTypes · macros · evaluators · functions ·           │
 │    validators · validationSources · providers · query/result/inbound/         │
 │    outboundTransformers · selectedResolvers · schemaSources · entitySources · │
 │    submitTargets · assetStores · localeSources · textFormatters ·             │
 │    effectHandlers · origins · migrations · renderers · regex                  │
 └───────────────────────────────────┬───────────────────────────────────────────┘
                                     │ Readable<T> per path/facet (get + subscribe)
          ┌───────────────┬──────────┴──────┬───────────────┬──────────────┐
          ▼               ▼                 ▼               ▼              ▼
       React            Vue             Solid           Svelte      Native (RN, etc.)
   renderer registry maps fieldType → platform component
```

## Document map

| # | Doc | Covers |
|---|-----|--------|
| 01 | [Schema Contract](./01-schema-contract.md) | Full TypeScript contract, fields, containers, versioning |
| 02 | [Big Picture](./02-big-picture.md) | Top-down architecture overview: main pieces, responsibilities, lifecycle and links to details |
| 03 | [State Model](./03-state-model.md) | Own transactional store, presence-based precedence, path-indexed subscriptions, meta, performance, plugins |
| 04 | [Paths & Dependency Graph](./04-paths-and-dependency-graph.md) | Path grammar, scoping, row identity, DAG, scheduling |
| 05 | [Expressions](./05-expressions.md) | Evaluator registry, scope, security model |
| 06 | [Validation](./06-validation.md) | Rule model, triggers, async validation, error shape |
| 07 | [Effects & Computed Values](./07-effects-and-computed.md) | Write origins, effects, computed fields, hidden policy |
| 08 | [Async Data & Registries](./08-async-data-and-registries.md) | Registries, provider options, asset stores, pagination, load lifecycle, dynamic sub-forms |
| 09 | [Loading & Submission](./09-loading-and-submission.md) | Loading entities, reloading, serialization, external errors, conflicts, snapshots (drafts), server rendering |
| 10 | [Localization](./10-localization.md) | `LocalizableString`, params, plurals, resolver, contract `locales`, BCP 47 tags |
| 11 | [Rendering & Adapters](./11-rendering-and-adapters.md) | Headless field API, renderer registry, framework bindings, accessibility, testing, adapter conformance |
| 12 | [Open Questions & Roadmap](./12-open-questions-and-roadmap.md) | Unresolved decisions and milestones |
| 13 | [Option Sources & Cascading](./13-option-sources-and-cascading.md) | Static, map, and provider options; cascading selects, invalidation, reverse fill |
| 14 | [Prior Art](./14-prior-art.md) | Comparison with JSON Forms, Form.io, SurveyJS, Formily, RHF/TanStack Form, and what we borrowed |
| 15 | [Macros](./15-macros.md) | Load-time expansion of short forms into core nodes; standard macro library |
| 16 | [Cookbook](./16-cookbook.md) | Patterns built from core primitives: autosave, variables, "Other" options, draft rows, view mode, totals, SSR, … |
| 17 | [Diagnostics](./17-diagnostics.md) | Diagnostic shape and the catalog of every error and warning code |
| 18 | [API Reference](./18-api-reference.md) | Every public signature: `loadContract`, `createFormEngine` options, `FormEngine`, controllers, `ArrayHandle`, plugin members |
| 19 | [Field Type Gaps](./19-field-type-gaps.md) | Resolved: content node, date min/max, `minLength`, yes/no recipe, money as a recipe, timezone storage |
| 20 | [Plugin API Gaps](./20-plugin-api-gaps.md) | Resolved (D28): controller extension, registry-recorded plugin factories, store scoping, submit hooks, text formatter conflicts, no contract hooks |
| 21 | [Static Type Inference](./21-type-inference.md) | Typed registries by return value, `defineContract`, inferred values/paths/payload, widening for runtime contracts, declaration merging as fallback |

## Key decisions (ADR summary)

| ID | Decision | Doc |
|----|----------|-----|
| D1 | Precedence is decided by **key presence** in a layer (`current` > `loaded` > `default`), not by nullish coalescing | [03](./03-state-model.md) |
| D2 | The engine owns its store (`FormStore`): transactions, per-commit entries passed to `CommitHooks` and then dropped (no retained log), path-indexed `Readable`s. There's no runtime dependency on a state library, and resolved values are never written directly | [03](./03-state-model.md) |
| D3 | Every write carries an `origin` label set by the engine from the method called (core: `user`, `host`, `effect`, `computed`, `load`, `resume`, `reset`, `default`, `external`; plugin-registered: `history`, `restore`; or `x-…`). What each origin does (layer, effects, cascade invalidation, history flag) is defined in one `ORIGIN_POLICY` table: core rows built in, plugin rows registered at install | [07](./07-effects-and-computed.md) |
| D4 | Array rows have stable engine-assigned `rowKey`s; meta and errors are keyed by row key | [04](./04-paths-and-dependency-graph.md) |
| D5 | No `eval`. Custom logic goes through registered, named functions | [05](./05-expressions.md) |
| D6 | `json-logic` is the only built-in expression language (plus `fn`, which calls registered functions); others are plugins | [05](./05-expressions.md) |
| D7 | Layout hints and behavioral conditions are separate properties | [01](./01-schema-contract.md) |
| D8 | By default, hidden values are kept and submitted, disabled values aren't submitted, and neither is validated. Configurable per form, then per container and field | [07](./07-effects-and-computed.md) |
| D9 | Every async operation is cancellable and generation-checked | [08](./08-async-data-and-registries.md) |
| D10 | Contracts are validated against a published JSON Schema on load. The format is versioned from `1.0` (first release); later majors ship migrations | [01](./01-schema-contract.md) |
| D11 | Every path reference in a contract is explicit (`/`, `./`, `../`, `$…`). Bare names don't compile | [04](./04-paths-and-dependency-graph.md) |
| D12 | Cascading selects aren't a field type. They're any options source with dependencies. Invalidation follows write origins, so loading data never clears dependent selects | [13](./13-option-sources-and-cascading.md) |
| D13 | The core contract only grows for things that can't be expressed with existing primitives. Convenience goes into load-time **macros** or the **cookbook** | [15](./15-macros.md), [16](./16-cookbook.md) |
| D14 | The core stores parsed values only. Raw input lives in renderers, which report unparseable input with `setInputError` | [11](./11-rendering-and-adapters.md#21-raw-input-stays-in-the-renderer) |
| D15 | Expressions are synchronous and pure. There are no async functions or async values. Async data reaches the engine through `context` (host-loaded), provider options, or async validation | [05 §5.1](./05-expressions.md#51-functions-are-synchronous) |
| D16 | Field type settings are flat on the node when the engine reads them, and in a typed `ui` object when only renderers do. Types come from an extensible `FieldTypeRegistry`. Options sources are discriminated by `source`, not `kind`. `ui.variant` picks the presentation on every node type; only `step: true` (wizard) is engine-visible | [01 §3](./01-schema-contract.md#3-fields) |
| D17 | The engine does no I/O and assumes no transport. Providers, stores, sources, and targets are registered functions that take opaque queries, so they work with network APIs, local files, storage, or memory alike | [08 §4](./08-async-data-and-registries.md#4-the-engine-does-no-io) |
| D18 | Contracts may optionally ship per-locale bundles in `locales` (inline or via `localeSources`), keyed by BCP 47 tags. Lookup order: host resolver → contract bundle → fallback locales → `default`. Locale switches never touch form state | [10 §3](./10-localization.md#3-contract-locales) |
| D19 | Expressions are always explicit `{ lang, expr }`. No bare JSON Logic shorthand, so `Json \| Expression` slots are never ambiguous | [05 §1](./05-expressions.md#1-shape) |
| D20 | One `Diagnostic` shape and a single code catalog. Load steps collect all problems before failing | [17](./17-diagnostics.md) |
| D21 | Enumerated string values are kebab-case, keys are camelCase. Each set is one exported `const` tuple that the types, validator, and JSON Schema derive from, and shared words (`clear`, `reset`, `keep`, `exclude`) mean the same thing everywhere. Every name a host or third party adds to a built-in set (origins, field types, UI variants, plugin names and their meta slices, macro ids) is `x-` plus kebab-case, so extensions never collide with future built-ins. Only exemption: macro ids may carry an `@MAJOR` suffix | [01 §2.1](./01-schema-contract.md#21-enumerated-values) |
| D22 | Drafts are snapshots of the `current` layer restored onto the same baseline, so edits and their change indicators survive reopening. Stale drafts surface as conflicts | [09 §8](./09-loading-and-submission.md#8-snapshots-drafts) |
| D23 | **Runtime plugin boundary.** A feature may be a plugin only if (a) the host uses it purely through explicit calls, and (b) its absence changes no default behaviour of fields, validation or submission. Everything else is core. Plugins: `history` (undo/redo), `drafts` (snapshot/restore; `revision` is core), `wizard`, `icu` (ICU message formatting; core does `{name}` only), `assets` (`asset`/`multi-asset`, which stay reserved type names). A contract that needs a missing `icu` or `assets` fails at load with `E_PLUGIN_MISSING`, never runs degraded. External updates and conflicts stay core (they block submit by default), as do async validation, options, loading and submission. The default `@formgraph/engine` entry installs all plugins, so the split is invisible; `@formgraph/engine/core` is the slim entry | [03 §8](./03-state-model.md#8-plugins) |
| D24 | **Platform-agnostic core.** `@formgraph/contract` and `@formgraph/engine` (every entry and built-in plugin) run on any ES2020 runtime with `Promise`, `Map`/`WeakMap` and `AbortController`: browsers, Node, Deno, Bun, workers and React Native (Hermes). No DOM, Node or Web-only globals; timing comes from the `platform` option; focus, element ids and file handles stay opaque and are handled by adapters, which each expose escape hatches | [11 §6](./11-rendering-and-adapters.md#6-platforms) |
| D25 | **Custom fields are opaque units.** A custom field, including one that wraps a nested engine or a foreign form, is a single atomic value to the engine. The engine creates no paths, controllers, dependencies, conditions or effects inside it. It connects only through the field controller: `setValue` (whole value), `report`/`onValidate` (messages, with `subPath` for location only), `setBusy` (pending-work gate), and the type hooks (`equals`, `isEmpty`, `pending`, `toSnapshot`). Deep integration (shared graph, cross-references, one history) means using groups, arrays or dynamic sub-forms instead | [01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions), [11 §2.2](./11-rendering-and-adapters.md#22-component-reported-messages-and-busy-state), [16 §22](./16-cookbook.md#22-another-engine-as-a-custom-field) |
| D26 | **Types flow from values, not global augmentation.** Typed registration returns a wider `Registry<E>` (chained or one object literal); `defineContract(registry, c)` checks ids and config against it; `ValuesOf`/`PathsOf`/`PayloadOf` type the engine. Runtime contracts widen to `Json`. `declare module` is the last-resort fallback for registries whose contents aren't visible at the call site | [21](./21-type-inference.md) |
| D27 | **Adapters own reactivity.** The core has no signal system of its own. `Readable.get()` reports reads to an observer (`observeReads`, `@formgraph/engine/interop`), and adapters connect it to their framework's native tracking (Solid/Vue/Svelte/Angular computed values and effects; React `useTracked`). Rule: the core may expose integration points, but they're meant for adapter authors, not app code; app code uses its framework's primitives. `engine.batch` groups writes. Framework-free code uses `subscribe` or the optional `@formgraph/engine/reactive` subpath. `Readable` lives in one core module that nothing else depends on internally, so a possible future move to native TC39 Signals (a note only, not planned) stays local | [03 §2.6](./03-state-model.md#26-tracking-reads-for-framework-adapters) |
| D28 | **Plugins are registry-recorded factories.** `registry.plugin(factory)` registers the field types/validators the plugin declares (`registers`) and records it; each engine creates its own instance with per-engine options (`plugins: { name: opts \| false }`). One registration path, so `loadContract` sees plugin types with no engine. Third-party plugins get a narrow `PluginStore` (own `x-` origins and slices, values), `notices.set`, submit hooks (`before` may stop with an `x-` reason), and field-controller extensions; built-ins keep `PrivilegedPluginStore`. One text formatter (`E_PLUGIN_CONFLICT`). Plugins don't extend the contract format and add no form-level validation hooks | [03 §8.1](./03-state-model.md#81-plugin-api) |
| D29 | **Display-only `content` node.** A node kind with no value or data path (heading/text/divider/`x-` variants), `conditions.visible` only, its own renderer kind and `ContentController`. Core field types stay the same eight; `date-time` gains `min`/`max`, `text` gains `minLength`, and yes/no-unanswered, money and constants are recipes or macros (D13) | [01 §3.6](./01-schema-contract.md#36-content-nodes) |

## Package layout (proposed)

All packages publish under the `@formgraph` npm scope (for example `@formgraph/contract`, `@formgraph/react`).

```
packages/
  contract/        # types + JSON Schema + migrations (zero deps); macro expansion (expandContract) added in M3
  macros/          # optional standard macro library
  core/            # @formgraph/engine (deps: contract only). Entries:
                   #   @formgraph/engine        batteries-included: core + history, drafts, wizard, icu, assets (03 §8)
                   #   @formgraph/engine/core   slim: core only, add plugins explicitly
                   #   @formgraph/engine/plugins   barrel; also /plugins/{history,drafts,wizard,assets}
                   #   @formgraph/engine/icu    ICU subset formatter (10 §2.1)
  expr-jsonlogic/  # default evaluator
  react/ vue/ solid/ svelte/   # thin adapters binding Readable<T> to each framework
  devtools/        # graph + state inspector
  testing/         # in-memory registry, fakes, clock, fill/expect helpers, adapter conformance suite (11 §9)
```

**Size budgets** (targets, min+gz, checked in CI with a size-limit step; exceeding one fails the build):

| Entry | Budget |
|-------|--------|
| `@formgraph/engine/core` (store, DAG, scheduler, validation, options, async, load/submit, conflicts, `{name}` text) | ≤ 15 kB |
| `@formgraph/engine/icu` | ~2 kB |
| `assets` plugin | ~1.5 kB |
| `@formgraph/engine` (core + history + drafts + wizard + icu + assets) | ≤ 23 kB |
| `@formgraph/expr-jsonlogic` | ≤ 4 kB |
| Each adapter (`@formgraph/react`, …) | ≤ 3 kB |
