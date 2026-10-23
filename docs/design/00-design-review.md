# Design Review: v4.0 Architecture Handoff

Status: **Review complete** · Input: *Form Architecture Context Handoff (v4.0)* · Output: the [design doc set](./README.md)

This review checks the v4.0 handoff for correctness, gaps, and internal contradictions before any code is written. Each finding has a severity, a short explanation, and a resolution that links to the design doc where it is handled.

Severity scale:
- **Critical**: causes data loss, security exposure, or wrong behavior under normal use.
- **Major**: blocks a real use case, or forces a breaking contract change later.
- **Minor**: unclear wording, ergonomics, or missing detail.

---

## Summary

The core ideas hold up: headless JSON contract, a framework-agnostic vanilla core, strategy registries instead of enums, layered state with a derived read model, and atomic arrays. The weak spots are where these ideas meet:

1. **The precedence model can't tell "unset" from "cleared"**, so a user can't clear a hydrated value. (Critical)
2. **Effects have no trigger semantics**, so `clear_value` effects would wipe hydrated data on load. (Critical)
3. **`javascript_eval` runs CMS-supplied code** on every client. (Critical)
4. **Index-based paths in arrays** break errors, touched state, and UI keys on reorder or delete. (Major)
5. **The handoff claims "unbounded flexibility" but uses closed enums** for field types, validation types, and effect actions. (Major)
6. **Some pipelines are missing entirely**: field meta (touched, dirty, validating), submission and serialization, server error mapping, and hidden-field value policy. (Major)

---

## Findings

### State model

| # | Sev. | Finding | Resolution |
|---|------|---------|------------|
| S1 | Critical | `current ?? hydrated ?? default` treats `null`/`undefined` as "fall through". If a user clears a hydrated field, the hydrated value shows up again. | Precedence is based on **key presence**, not nullishness. Writes into `currentState` are always explicit, and `undefined` is a real value. → [State Model §3](./02-state-model.md#3-precedence-by-presence-not-nullishness) |
| S2 | Major | `computedState` is listed as a stored field. If engine code writes it, it can drift from the write silos. | `computedState` is derived lazily inside `FormStore` and can't be written. Nothing outside the store can write to it. → [State Model §2](./02-state-model.md#2-store-topology) |
| S3 | Major | There is no field meta: touched, dirty, validating, async status, or submit status. `errors: Record<string, any>` is untyped. | Add a typed `meta` silo keyed by path, plus a `form` status slice. → [State Model §5](./02-state-model.md#5-field-meta-and-form-status) |
| S4 | Major | Once a user edits any row, the atomic array write shadows the hydrated array for good. Rehydration silently does nothing for that array. | Keep atomic arrays, but define rehydration policy explicitly (`preserve-edits` / `replace` / `merge-untouched`). → [Hydration §3](./08-loading-and-submission.md#3-reload-policy) |
| S5 | Minor | The 60fps claim assumes cheap merges. A naive deep merge on every keystroke is O(form size). | Structural sharing, lazy resolve, and path-indexed notification. Only the written branch changes identity, and only its subscribers are checked. → [State Model §6](./02-state-model.md#6-performance-contract) |

### Paths, identity, and the DAG

| # | Sev. | Finding | Resolution |
|---|------|---------|------------|
| P1 | Major | Errors and meta keyed by `addresses[0].city` go to the wrong row after a reorder or delete. React keys have the same problem. | Each array row gets a stable engine `rowKey`. Meta and errors are stored by **row key path** and translated to index paths only at the edges. → [Paths §4](./03-paths-and-dependency-graph.md#4-array-row-identity) |
| P2 | Major | Only `./` (sibling) is defined. There is no parent (`../`), root (`/`), or row-index reference. | Explicit path grammar (`/`, `./`, `../`, `.`, `$…`). Bare names are a compile error. → [Paths §2](./03-paths-and-dependency-graph.md#2-path-grammar) |
| P3 | Major | It isn't defined whether `group` creates a data namespace, or whether `id` or `serializeAs` is the data key. | `id` is the in-memory key. Groups are transparent by default, with opt-in `dataScope: true`. `serializeAs` only applies at serialization time. → [Paths §3](./03-paths-and-dependency-graph.md#3-data-keys-and-scoping) |
| P4 | Major | Manual `dependencies[]` lists can go stale, and nothing detects cycles across computed values and effects. | Infer dependencies from expressions where possible, treat manual lists as additions, and detect cycles at schema-compile time. Effects run with a bounded cascade depth. → [Paths §5](./03-paths-and-dependency-graph.md#5-dependency-graph) |

### Expressions and security

| # | Sev. | Finding | Resolution |
|---|------|---------|------------|
| X1 | Critical | `javascript_eval` runs remote, CMS-supplied code. It breaks under strict CSP, isn't available on Hermes/JSC configs, and is an XSS/RCE vector if the CMS is compromised. | Remove it from the contract. Replace it with **named client functions** (`fn` evaluator that calls registered functions by ID). → [Expressions §4](./04-expressions.md#4-security-model) |
| X2 | Major | `ExpressionLanguage` is a closed enum, and `simple_match` has no definition. | Use an evaluator registry keyed by string. Ship `json_logic` (default) and `fn`. `jsonata` is not shipped (Q3). `simple_match` is dropped unless someone specifies it. → [Expressions §2](./04-expressions.md#2-evaluator-registry) |
| X3 | Minor | It isn't defined what variables an expression can see (values, row, index, meta, context). | Define a fixed evaluation scope. → [Expressions §3](./04-expressions.md#3-evaluation-scope) |

### Contract shape

| # | Sev. | Finding | Resolution |
|---|------|---------|------------|
| C1 | Major | `fieldType` is meant to be "concrete" (a closed list), which conflicts with the registry philosophy. `BaseFormField<any, any>` throws away type safety. | Field types are declared in an extensible `FieldTypeRegistry`: each maps to a value type, flat engine config, and a typed `ui` object. Custom types use the `x-` prefix and register runtime schemas. → [Contract §3](./01-schema-contract.md#3-fields) |
| C2 | Major | `ResponsiveLayout` mixes geometry (`xs..xl`) with behavior (`visible`, `disabled`). Native platforms often have no grid. | Split it into `layout` (hints only) and `conditions` (`visible`, `disabled`, `readOnly`, `required`). → [Contract §4](./01-schema-contract.md#4-layout-vs-conditions) |
| C3 | Major | `ClientValidationRule.type` is a closed enum, but `custom` needs a way to point at a function. | `type` is a validator ID from a registry. Built-ins keep their names. → [Validation §2](./05-validation.md#2-rule-model) |
| C4 | Major | `ServerValidationRule` and `AsyncEvaluationRule` overlap, and `AsyncEvaluationRule` isn't attached to anything. | Merge them into one `AsyncValidationRule`. → [Validation §4](./05-validation.md#4-async-validation) |
| C5 | Major | `schemaVersion` has no compatibility rule, and there is no runtime validation of incoming JSON. | Semver-like versioning, a published JSON Schema, a load-time validator, and a migration chain. → [Contract §7](./01-schema-contract.md#7-versioning-and-load-time-validation) |
| C6 | Minor | `LocalizableString` has no interpolation params or plural support. Messages like "min {{value}}" can't be expressed. | Add `params` and ICU-style `default`. → [Localization](./09-localization.md) |
| C7 | Minor | `binary_asset` has no upload lifecycle, and `date_time` has no timezone or wire format. | Define the value shapes. → [Contract §3.2](./01-schema-contract.md#33-core-config-and-ui) |

### Behavior gaps

| # | Sev. | Finding | Resolution |
|---|------|---------|------------|
| B1 | Critical | Effects (`clear_value`, `set_value`) don't say *when* they fire. If they fire on hydration or default seeding, they destroy loaded data. | Effects fire only on **user-originated** writes (and optionally on `effect` cascade). They never fire during hydration or reset. Writes carry an `origin` tag. → [Effects §2](./06-effects-and-computed.md#2-write-origins-and-effect-triggers) |
| B2 | Major | Hidden fields: are their values kept, submitted, or validated? | Add `whenHidden` (`keep` default / `exclude` / `clear`) and `whenDisabled` (`exclude` default / `keep`) policies, set form-wide and overridable per container and field. Hidden and disabled fields aren't validated by default. → [Effects §4](./06-effects-and-computed.md#4-hidden-and-disabled-field-policy) |
| B3 | Major | There is no submission pipeline: serialization, `serializeAs`, dirty-only patches, or mapping server errors back to fields. | Define the submission pipeline. → [Submission §4](./08-loading-and-submission.md#4-submission-pipeline) |
| B4 | Major | Async lazy loads and validations have no cancellation or stale-response handling. `dependsOn` changes don't reset pagination. | AbortSignal per request, generation counters, and pagination reset on dependency change. → [Async §3](./07-async-data-and-registries.md#3-load-lifecycle) |
| B5 | Major | `DynamicSubFormGroup`: it isn't defined what happens to values and errors from the previous sub-schema when the switch field changes, or how races resolve. | Define a sub-schema lifecycle with an orphan policy and race handling. → [Dynamic Sub-forms](./07-async-data-and-registries.md#5-dynamic-sub-forms) |
| B6 | Minor | `computed` fields with `readOnly: false` aren't defined. Does user input override the formula? | `readOnly: false` means user input wins until the field is reset. → [Effects §3](./06-effects-and-computed.md#3-computed-values) |

### Integration

| # | Sev. | Finding | Resolution |
|---|------|---------|------------|
| I1 | Minor | Svelte "native compatibility" relies on store-contract shape. | Every framework gets a small first-party binding over `Readable<T>`. → [Adapters §3](./10-rendering-and-adapters.md#3-framework-bindings) |
| S6 | Major | TanStack Store (v0.11) is pre-1.0 and was rewritten between versions (`Derived` was removed). Its selector-based subscriptions cost O(all subscribers) per write, and it provides none of the form-specific behavior (origins, dependency ordering, rollback, write log). | Use our own `FormStore`, with no runtime dependency, and expose `Readable<T>`. → [State Model §2.4](./02-state-model.md#24-why-not-tanstack-store) |
| I2 | Major | The renderer registry, the field-to-component mapping, is implied but never specified. It is the main integration point for every platform. | Specify the renderer registry and the headless field API (`useField`). → [Adapters §2](./10-rendering-and-adapters.md#2-headless-field-api) |

---

## What's kept as-is

- Strategy registries for request and response transformers, including opaque pagination state.
- Atomic array writes (with the rehydration policy added).
- `LocalizableString` with key plus default (extended, not replaced).
- Recursive `group` / `array` / `dynamic_sub_group` composition.
