# 04 · Expressions

[← Index](./README.md) · Addresses review items X1, X2, X3

## 1. Shape

```ts
interface Expression { lang: string; expr: Json }
```

Expressions are **always explicit**. There is no bare JSON Logic shorthand, and `lang` is required:

```json
"conditions": { "visible": { "lang": "json-logic", "expr": { "==": [{ "var": "./type" }, "business"] } } }
```

Why: many slots accept `Json | Expression` (`defaultValue`, `newItem`, effect `value`, rule `value`, text `params`). There, a bare `{ "var": "./x" }` and a literal object look the same, and guessing by operator name would silently change meaning when a new function is registered. One rule everywhere is easier to read and to validate.

**Detection:** in a `Json | Expression` slot, an object with exactly the two keys `lang` and `expr`, where `lang` is a string, is an Expression. Detection applies **only at the top level of the slot**: objects nested inside a literal stay literals. Any other value is a literal. To use a literal object of exactly that shape at the top level (very rare), wrap it in an expression: `{ "lang": "json-logic", "expr": { "preserve": { "lang": …, "expr": … } } }`.

Unknown `lang` values are `E_UNKNOWN_EVALUATOR` at validate-core (evaluators live on the shared registry, so no engine is needed).

## 2. Evaluator registry

```ts
interface Evaluator {
  id: string;
  compile(expr: Json): CompiledExpr;          // parse once at schema compile
  collectDeps(expr: Json): PathRef[];         // for DAG inference (03 §5.1); must be exact, not approximate
}
interface CompiledExpr { (scope: EvalScope): Json }   // must be synchronous (§5.1)

registry.evaluators.register(jsonLogicEvaluator);   // built in
registry.evaluators.register(myEvaluator);          // third-party; on the shared registry (07 §1), so load steps can check `lang` without an engine
```

| `lang` | Status | Notes |
|--------|--------|-------|
| `json-logic` | Built in, default | Pure data, easy to statically analyze. Custom operators are added through the function registry |
| `fn` | Built in | `{ lang:'fn', expr:{ name:'vatRate', args:[{var:'./country'}] } }` calls a registered function. Each entry in `args` is evaluated as JSON Logic |
| `jsonata` | **Not shipped** | Decided in Q3 ([11](./11-open-questions-and-roadmap.md)): dependency inference is only approximate, and it supports async functions, which conflicts with [§5.1](#51-functions-are-synchronous). Use registered functions for logic JSON Logic can't express |
| `javascript_eval` | **Removed** | See §4 |
| `simple_match` | **Dropped** | It was never specified. `json-logic` `==` / `in` covers it |

The JSON Logic evaluator adds these operators to the standard set:
- `preserve`: returns its argument unevaluated, used for literal objects shaped like an Expression (§1).
- `sum`, `count`, `min`, `max` over an array, typically a `[*]` path: `{ "sum": { "var": "/items[*].qty" } }`. O(rows). `sum`, `min`, `max` skip non-numbers; `count` counts elements. An empty array gives `0` for `sum`/`count` and `null` for `min`/`max`. `sum` uses the number semantics of `+` ([06 §3.2](./06-effects-and-computed.md#32-aggregates)).

Everything else beyond standard JSON Logic comes from registered functions (§5).

**`var` takes a literal path.** Its argument must be a literal `PathRef` string, or `[PathRef, default]`. Computed paths (`{ "var": { "cat": … } }`), `""`, and numeric segments (`"./items.0"`) are `E_DYNAMIC_PATH` at compile, so `collectDeps` stays exact.

## 3. Evaluation scope

Every expression gets the same scope object. The JSON Logic `var` resolver is replaced with a path-grammar-aware one ([03 §2](./03-paths-and-dependency-graph.md#2-path-grammar)):

```ts
interface EvalScope {
  values: Json;            // resolved root values
  scopePath: DataPath;     // where the expression lives (for ./ and ../)
  row?: Json; index?: number;
  option?: SelectOption;   // $option, only in options filters (12 §2.1)
  selected?: SelectOption; // $selected, only in onSelect effects (12 §7)
  context: Record<string, Json>;   // host-provided JSON: user role, locale, feature flags, `now` (ISO string)
  meta: { dirty(p): boolean; touched(p): boolean; visible(p): boolean };
}
```

`context` is the supported way to let host-app state (permissions, tenant, locale) drive conditions without putting it into form values. Changes to `context` go through `engine.setContext(partial)`, which starts a tick. Expressions that read `context.x` get DAG edges from a synthetic `$context.x` node. `context` holds JSON only, never functions: there is no `now()`. A host that needs the time sets `now` itself (`setContext({ now: '2026-10-20T09:00:00Z' })`) as often as it wants, which keeps expressions pure and replayable.

## 4. Security model

**Contracts are untrusted input.** A compromised or misconfigured CMS must not be able to run code on clients.

- There is no `eval`, `new Function`, or `with` anywhere in the core or in the built-in evaluators.
- `fn` can only call functions the **host app registered** by name. An unknown name is `E_UNKNOWN_REGISTRY_ID` in validate-core: functions live on the `Registry` ([07 §1](./07-async-data-and-registries.md#1-registries)), so `loadContract` checks them without an engine.
- Registered functions get a frozen copy of their args and the scope, and can't write state. They return a value or throw. A throw is caught and logged (`W_FUNCTION_THREW`), and the facet falls back. Display facets fail open; anything that decides validity **fails closed**:

  | Facet | Fallback |
  |-------|----------|
  | `visible` | `true` |
  | `required` | blocking `error` Notice on the field (fail closed) |
  | `read-only` | `false` |
  | `disabled` | `false` |
  | `options` | keep the previous list (empty on first evaluation) |
  | `computed` / `value` | keep the previous value |
  | `default` | the field type's empty value |
  | `label` / `message` | the param is left unformatted (`{param}`) |
  | `validation` (rule `when` / `value` / validator) | blocking `error` Notice on the field (fail closed) |
  | effect `when` / `value` | the effect is skipped for this tick |

- **Step budget:** each evaluation is limited to `limits.evalSteps` steps (default 10,000; one step = one operator application or one array-iteration step). It is deterministic and applies in dev **and** production. Exceeding it aborts the evaluation, logs `W_EVAL_BUDGET`, and the facet falls back as in the table above (validation and `required` fail closed with a blocking error, so a budget-exhausting contract can't make invalid data submittable). Expression depth and size are also capped at compile ([01 §7.4](./01-schema-contract.md#74-limits)).
- **Regexes** from contracts (the `pattern` validator, and any evaluator op taking a pattern) go through `registry.regex` with source and input length caps ([05 §2.2](./05-validation.md#22-regex-safety)).
- Works under strict CSP (no `unsafe-eval`) and on Hermes and other JS engines that don't have `eval`.

**Trust boundary.** Trusted: the registry ([07 §1](./07-async-data-and-registries.md#1-registries)) and host code. Untrusted: contracts (including dynamic sub-forms), snapshots and drafts, `resume` state ([08 §9](./08-loading-and-submission.md#9-server-rendering)), locale bundles, external errors, and provider results. Untrusted data is validated and bounded ([01 §7.4](./01-schema-contract.md#74-limits)) and never executed; it can only reference trusted code by id.

**The client is not a security boundary.** Whoever can edit snapshots or `resume` state can also edit the DOM, call `setValue`, or send requests directly. These checks keep the engine robust (bounded work, allowed ids); they don't prove the data is what the server sent. Submit targets must validate on the server. Hosts that never want sub-schema nodes taken from `resume` export with `subSchemas: 'ref'`.

**Registry ids are capabilities.** By default any registered id (function, provider, submit target, validator, asset store, locale source, …) can be referenced by any contract. Hosts that load contracts from several sources restrict them per contract:

```ts
type AllowableRegistry =
  | 'fieldTypes' | 'macros' | 'evaluators' | 'functions' | 'validators' | 'validationSources'
  | 'providers' | 'queryTransformers' | 'resultTransformers' | 'selectedResolvers'
  | 'schemaSources' | 'entitySources' | 'inboundTransformers' | 'outboundTransformers'
  | 'submitTargets' | 'assetStores' | 'localeSources' | 'textFormatters' | 'effectHandlers';
type AllowList = Partial<Record<AllowableRegistry, string[]>>;   // omitted key = all allowed

loadContract(input, {
  allow: { functions: ['age'], providers: ['countries'], submitTargets: ['crm'] },
});
```

The keys are the registries a contract references by id (the "allow-listable" column in [07 §1](./07-async-data-and-registries.md#1-registries)); `origins`, `migrations`, `renderers` and `regex` aren't contract-referenced and can't be listed. Each kind is checked in the phase where it is first resolved:

| Kind | Phase |
|------|-------|
| `macros` | expand |
| `evaluators`, `fieldTypes` | validate-core |
| everything else | compile |

A reference to an id outside its list is `E_NOT_ALLOWED`, which takes precedence over the unknown-id codes (`E_UNKNOWN_MACRO`, `E_UNKNOWN_EVALUATOR`, `W_UNKNOWN_FIELD_TYPE`, `E_UNKNOWN_REGISTRY_ID`): a listed kind rejects every id not in the list, registered or not. A disallowed `x-` fieldType is therefore an error, never a fallback render. Dynamic sub-forms inherit the parent's `allow`.

## 5. Registered functions

```ts
registry.functions.register('vatRate', (country: string) => RATES[country] ?? 0, {
  pure: true,          // allows memoization by args
  deps: [],            // extra DAG deps (e.g. '$context.locale')
});
```

Functions are also exposed as JSON Logic operators (`{ "vatRate": [{ "var": "./country" }] }`), so authors can stay inside one language. So a function may not take the name of a built-in operator (`var`, `preserve`, `sum`, `if`, `==`, `and`, `in`, `cat`, …): `register` throws `E_RESERVED_FUNCTION_NAME`.

### 5.1 Functions are synchronous

Expressions are evaluated inside a synchronous scheduler tick ([03 §6](./03-paths-and-dependency-graph.md#6-scheduler)). A Promise-valued facet would break single-commit ticks, have no cancellation or ordering of stale results, and force every facet (including `visible`) to have a pending state.

- `register` rejects `async` functions (`E_ASYNC_FUNCTION`).
- A function that returns a thenable at runtime is treated as a throw (`W_FUNCTION_THREW`): the error is logged, and the facet falls back as in §4.
- There is **no async value type** in expressions, and no `$lookup` root.

### 5.2 Async data: where it goes instead

| Need | Use | Why |
|------|-----|-----|
| Small reference tables (VAT rates, currency precision, country rules) | Put the data in `context` or in the contract `meta`, and pass it to a sync function as an argument: `{ "vatRate": [{ "var": "/country" }, { "var": "$context.vatRates" }] }` | Deterministic, offline, no loading state. Passing the data as an argument keeps `pure` memoization correct, and dependencies are inferred |
| A value that has to come from outside the form (an API, storage, a device) and depends on form values (FX rate, eligibility flag) | The host loads it and calls `engine.setContext(...)` ([15 §15](./15-cookbook.md#15-externally-derived-inputs-via-setcontext)) | The host owns caching, cancellation, and retries. The loading state is explicit (`null`) and visible in conditions |
| Fill in a field from a lookup (postcode → city) | A `provider` options source + `autoSelectSingle` ([15 §16](./15-cookbook.md#16-postcode--city-autofill)) | Reuses the cancellation, validity, and edit-mode rules from [12](./12-option-sources-and-cascading.md) |
| External rules about a value | Async validation with a `severity` ([05 §4](./05-validation.md#4-async-validation), [§7](./05-validation.md#7-severity)) | Produces messages, which is what a rule check should produce |
| Authoritative totals or prices | Compute in the submit target (usually a server) at submit time. The client shows an estimate | The client can't be trusted with the final number anyway |
