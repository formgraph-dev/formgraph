# 02 · Big Picture

[← Index](./README.md)

**A contract describes the form, the engine runs its behavior, and the host decides how it looks and where its data goes.** This is a top-down view of the current design, not an implementation status report; the project is still in the design stage.

## 1. The main pieces

```text
                      JSON contract
                            │
                  validate, expand, compile
                            │
                            ▼
                   Headless form engine
              ┌──────────────────────────┐
              │ State store              │
              │ Dependency graph         │
              │ Batched scheduler        │
              │ Validation + async work  │
              └──────────────────────────┘
                 ▲                    │
            user/host writes     observable state
                 │                    ▼
                 └──── Adapter + renderers ──── UI

  Host-registered implementations supply custom logic, data sources,
  submission targets and plugins; the engine coordinates their use.
```

| Piece | Responsibility | Details |
|-------|----------------|---------|
| **Contract** | JSON describing fields, structure, conditions, validation and behavior. It names registered capabilities rather than embedding executable code. | [01 · Schema Contract](./01-schema-contract.md) |
| **Registry** | Connects those names to trusted implementations: field types, validators, expression functions, data providers, transformers and more. | [08 §1 · Registries](./08-async-data-and-registries.md#1-registries) |
| **Engine** | Owns form state and coordinates dependencies, validation, loading and submission without knowing the UI framework. | [03 · State Model](./03-state-model.md), [04 · Dependency Graph](./04-paths-and-dependency-graph.md#5-dependency-graph) |
| **Adapters and renderers** | Adapters bind observable state to framework reactivity; renderers turn node controllers into platform components and report user input. | [11 · Rendering & Adapters](./11-rendering-and-adapters.md) |
| **Host application** | Supplies context, integrations and presentation. Network, storage, authentication and transport choices stay outside the engine. | [08 §4 · No I/O in core](./08-async-data-and-registries.md#4-the-engine-does-no-io) |

## 2. From a description to a running form

The host registers the capabilities it supports, then loads a contract. Loading validates the input, normalizes shortcuts, expands macros and compiles paths and dependencies. Invalid references or dependency cycles are reported before the form runs. The host creates an engine with the resulting schema and binds it to its UI.

The contract tree contains **fields** for values, **groups** for structure, **arrays** for repeating rows, **dynamic groups** for schema selected at runtime, and **content** for display-only text. Layout is separate from behavior; a visual section need not create a nested data object.

Details: [01 §7.1 · Load pipeline](./01-schema-contract.md#71-contract-load-pipeline), [01 §5 · Containers](./01-schema-contract.md#5-containers), [04 §3 · Data scoping](./04-paths-and-dependency-graph.md#3-data-keys-and-scoping).

## 3. State and behavior work together

Ordinary field values resolve from three layers: **current edits → loaded data → defaults**, using key presence rather than truthiness. Computed fields add derived values. Interaction metadata, notices and form status are kept separately. Repeating rows have stable identities, so their state follows them when reordered.

A write starts a batched update. The dependency graph identifies affected conditions, computed values, effects and option lists; the scheduler updates them in dependency order, then notifies affected subscribers. Write origins distinguish user edits from loading or resets, so loading a record does not accidentally trigger user-edit effects or clear cascading selections.

Details: [03 · State Model](./03-state-model.md), [04 §6 · Scheduler](./04-paths-and-dependency-graph.md#6-scheduler), [07 · Effects & Computed Values](./07-effects-and-computed.md).

## 4. Rules, data and saving

- **Expressions** drive conditions and calculations. They are synchronous and use supported evaluators or registered functions—never arbitrary code from the contract. Host context supplies external inputs.
- **Async integrations** provide options, validation, entities, dynamic schemas and asset storage. The engine coordinates cancellation and ignores superseded results; registered implementations perform the actual I/O.
- **Validation** produces notices with severities, not just errors. Submission checks validation, pending work, conflicts and confirmation gates before building a payload and calling a registered target. Inbound and outbound mapping keep form values separate from wire formats.

Details: [05 · Expressions](./05-expressions.md), [06 · Validation](./06-validation.md), [08 · Async Data](./08-async-data-and-registries.md), [09 · Loading & Submission](./09-loading-and-submission.md), [13 · Cascading Options](./13-option-sources-and-cascading.md).

## 5. Presentation stays outside the core

Controllers expose values, conditions, notices and interaction methods without depending on HTML. Adapters connect them to framework-native primitives; renderers own raw input, parsing, focus and visual presentation. The same core targets browsers, servers, workers and React Native. Localized text is resolved independently of form values.

Details: [11 · Rendering & Adapters](./11-rendering-and-adapters.md), [10 · Localization](./10-localization.md).

## 6. Extension without growing the core

**Registrations** add capabilities, **macros** expand convenient authoring forms into existing nodes, and **plugins** add runtime features with per-engine instances. Built-in plugins cover history, drafts, wizard navigation, ICU formatting and assets. The default engine entry includes them; a slim entry allows explicit selection. Custom fields remain atomic values, not hidden nested dependency graphs.

Types flow from registrations and statically authored contracts; contracts received at runtime use wider JSON types. Diagnostics, testing helpers and devtools make the same architecture inspectable without becoming UI requirements.

Details: [03 §8 · Plugins](./03-state-model.md#8-plugins), [15 · Macros](./15-macros.md), [01 §3.4 · Custom Fields](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions), [21 · Type Inference](./21-type-inference.md), [17 · Diagnostics](./17-diagnostics.md), [11 §9 · Testing](./11-rendering-and-adapters.md#9-testing).

For concrete usage patterns, continue with the [Cookbook](./16-cookbook.md); for signatures, see the [API Reference](./18-api-reference.md). Implementation milestones and unresolved questions live in the [Roadmap](./12-open-questions-and-roadmap.md).
