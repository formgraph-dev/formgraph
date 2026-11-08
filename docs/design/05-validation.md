# 05 · Validation

[← Index](./README.md) · Addresses review items C3, C4

## 1. Pipeline overview

```
trigger (change | blur | submit | dependency)
  └─► skip if hidden and not `validateHidden`, or disabled and not `validateDisabled` (06 §4)
      └─► sync rules in order (`bail` stops at the first *error*; warning/info rules always run)
          └─► if no sync errors: async rules (debounced, cancellable)
              └─► feedback[rowKeyPath] = [...sync, ...async, ...external, ...input]   (each tagged with severity, §7)
```

## 2. Rule model

```ts
interface ValidationRule extends Extensible {
  type: string;                 // validator id: built-in or registered
  value?: Json | Expression;    // rule parameter (expression allowed: e.g. min = ../startDate)
  when?: Condition;             // gate; rule runs only if true (was `condition`)
  dependsOn?: PathRef[];        // extra deps beyond inferred ones
  message?: Text;               // default comes from validator's message
  severity?: Severity;          // default 'error', see §7
  trigger?: 'change' | 'blur' | 'submit';  // override form default
}
```

Built-in validator IDs: `required`, `pattern` (literal value only, §2.2), `min`, `max`, `step`, `min-length`, `max-length`, `min-items`, `max-items`, `email`, `url`, `equals` (`PathRef`), `expr` (runs `value` as a boolean Expression), and the engine-added `in-options`. The `assets` plugin adds `accept` and `max-bytes` ([07 §2.1](./07-async-data-and-registries.md#21-asset-stores)). Default messages in slim core avoid plural syntax (`"Minimum items: {count}"`); the `icu` plugin swaps in pluralized ones ([09 §2.1](./09-localization.md#21-formatter)). Several are implied by field config ([01 §3.3.1](./01-schema-contract.md#331-implied-rules)).

Custom validators:

```ts
registry.validators.register('iban', {
  validate: (value, param, ctx) => isValidIban(value),
  // returns: true | false | Text (message override) | { message?: Text; severity?: Severity; subPath?: string } | array of those objects
  message: { key: 'validation.iban', default: 'Invalid IBAN' },
});
```

### 2.1 Standard Schema validators

Any [Standard Schema](https://standardschema.dev) (Zod, Valibot, ArkType, …) can be registered as a validator. The engine calls `schema['~standard'].validate(value)` and maps each `issue` to a `Notice`. If an issue has a `path`, it is resolved relative to the rule's node: on a group/array rule it targets a child node, and on a field rule it becomes the message's `subPath` inside the value (§5).

```ts
registry.validators.registerSchema('iban', z.string().refine(isValidIban), {
  message: { key: 'validation.iban', default: 'Invalid IBAN' },   // optional; else issue.message
});
```

Standard Schema `validate` may return a Promise. A schema that does is treated as async: it is debounced and cancellable like §4. Standard Schema issues have no severity, so every issue takes the rule's `severity`.

### 2.2 Regex safety

Patterns come from untrusted contracts ([04 §4](./04-expressions.md#4-security-model)), and backtracking engines can take exponential time (ReDoS). Every contract-supplied pattern (`pattern` rules and any other regex from a contract) is compiled through `registry.regex`:

```ts
interface RegexEngine {
  linear?: boolean;            // true for linear-time engines; skips the nested-quantifier check
  compile(source: string, flags: 'u'): { test(input: string): boolean };   // throws on invalid pattern
}
registry.regex = re2Engine;   // e.g. a re2-wasm wrapper: linear time
```

- **Length cap:** a source longer than `limits.regexLength` (default 1,000, [01 §7.4](./01-schema-contract.md#74-limits)) is `E_LIMIT_EXCEEDED` at compile.
- **Literal string only.** A `pattern` rule's `value` is a plain `string`, never an `Expression` or object, so every pattern is checked at compile. An expression is `E_DYNAMIC_PATTERN`; a pattern the engine can't compile is `E_INVALID_PATTERN`.
- **JSON Schema semantics:** the pattern matches anywhere in the value (unanchored); write `^…$` for a whole-value match. It is always compiled with the `u` flag and no others, so `fromJsonSchema` / `toPayloadSchema` carry it unchanged and client and server run the same check. Case-insensitive matching is written into the pattern (`[Aa]`) or done by a normalize step.
- **Input cap:** a value longer than `limits.regexInputLength` (default 10,000 chars) fails the rule (blocking error) without running the regex.
- **Default engine** is native `RegExp` with the length caps only. It does **not** prevent catastrophic backtracking; hosts loading contracts from sources they don't control should register a linear-time engine.
- **Nested-quantifier check** (best effort): patterns such as `(a+)+` are `W_UNSAFE_PATTERN` (*strict → error*). Known misses: overlapping alternation (`(a|a)*`), adjacent quantifiers (`a*a*`), backreferences. Skipped when the engine has `linear: true`.
- **Fail closed:** a validator or rule `when`/`value` that throws or exceeds `limits.evalSteps` yields a blocking `error` Notice (plus `W_FUNCTION_THREW` / `W_EVAL_BUDGET`); same for `required` ([04 §4](./04-expressions.md#4-security-model)).
- `registry.regex` is a singleton slot, inherited from the parent registry ([07 §1](./07-async-data-and-registries.md#1-registries)).

## 3. Triggers

Form-level defaults are set in `FormContract.validation`:

```ts
interface ValidationConfig {
  mode?: 'change' | 'blur' | 'submit' | 'touched';   // when to first validate; default 'touched'
  revalidate?: 'change' | 'blur' | 'submit';         // after first submit; default 'change'
  bail?: boolean;                                             // default true
}
```

`touched` means validate on blur first, then on every change after that. The `blur` event is the renderer calling `controller.handlers.onBlur()` ([10 §2](./10-rendering-and-adapters.md#2-headless-field-api)); `engine.touch(path)` marks touched without running a blur pass. **Dependency triggers** (for example `confirmPassword` depending on `password`) re-run only if the dependent field has already been validated once, so untouched fields don't show errors early.

**How rule `trigger` combines with `mode` / `revalidate`.** The form setting decides *when a field is first validated* and *how often it re-validates*; a rule's `trigger` only narrows which of those passes the rule takes part in. Triggers are ordered `change` < `blur` < `submit` (later = less frequent):

- A field-level pass runs on the event selected by `mode` (before the first submit) or `revalidate` (after it). In that pass, a rule runs only if the pass event is at least as late as needed by the rule: a `trigger: 'blur'` rule skips `change` passes, and a `trigger: 'submit'` rule skips both `change` and `blur` passes.
- A rule's `trigger` never makes a field validate *more* often than `mode`/`revalidate` say. Under `mode: 'submit'`, a `trigger: 'change'` rule still first runs at submit.
- **`submit()` and `validate()` (including scoped calls from plugin hooks, [02 §8.1](./02-state-model.md#81-plugin-api)) run every applicable rule** (its `when` is true and the field isn't skipped by the hidden/disabled policy), regardless of `trigger`. A rule's trigger can delay it, never skip it at submit.
- Dependency re-runs use the same filter as a `change` pass.

## 4. Async validation

This replaces both `ServerValidationRule` and `AsyncEvaluationRule`:

```ts
interface AsyncValidationRule extends Extensible {
  sourceId: string;              // validationSources registry id
  include?: PathRef[];           // extra paths to send along
  debounceMs?: number;           // default 300
  trigger?: 'change' | 'blur' | 'submit';   // default 'blur'
  when?: Condition;
  pendingMessage?: Text;
  severity?: Severity;           // default for results that don't set their own
}

type ValidationSource = AsyncSource<{ value: Json; included: Record<string, Json> },
  true | NoticeInput | NoticeInput[]>;   // 07 §1.1

interface NoticeInput { message: Text; severity?: Severity; path?: PathRef; subPath?: string; code?: string }
```

- A new trigger cancels the in-flight call (`AbortController`) and bumps a generation counter. Stale results are dropped.
- **Sync errors win.** When a field's sync rules produce an `error`, every pending async rule for that field is aborted (generation bumped) and its `async` messages, including `pendingMessage`, are cleared. Async rules run again once the sync errors are gone.
- `meta.validating` counts in-flight checks. `submit()` and `validate()` first force-compile the validators they need ([02 §6.3](./02-state-model.md#63-lazy-compile)) and start every async rule of an eligible field that has no settled result for its current value (regardless of `trigger`). With `submission.awaitAsync` (default) submit awaits them; otherwise it returns `{ ok: false, reason: 'validating' }` ([08 §4](./08-loading-and-submission.md#4-submission-pipeline)).
- After `resume` ([08 §9](./08-loading-and-submission.md#9-server-rendering)), fields with async rules are *unvalidated* (not valid), so the next submit runs them.
- **Result cache.** Successful results (`true` or messages) are cached per `(sourceId, value, included, context)`, so re-blurring an unchanged value doesn't call the source again. `context` is part of the key because sources receive it.
  - **Thrown errors and aborts are never cached**; the next trigger calls the source again.
  - The cache is **cleared** on `reload`, `restore`, and `setContext` (any of them can change what the server would answer).
  - Bounded per source: an LRU with `cacheSize` entries (default 100). An optional `cacheTtlMs` expires entries early (default: no expiry). Both are set when registering the source:

```ts
registry.validationSources.register('user.emailFree', checkEmail, { cacheSize: 500, cacheTtlMs: 60_000 });
```

## 5. Notice shape

```ts
type CoreSeverity = 'error' | 'warning' | 'info';
type Severity = CoreSeverity | `x-${string}`;   // custom severities, §7.6

interface Notice {
  source: 'sync' | 'async' | 'external' | 'group' | 'input' | 'component' | 'plugin';
  rule: string;                   // validator id, sourceId, 'external' / 'input', the reporter name for 'component', or the plugin name for 'plugin'
  code?: string;                  // machine-readable code, e.g. from ExternalErrors[].code
  message: ResolvedText;          // localized, see 09
  severity: Severity;
  subPath?: string;               // JSON Pointer into the field's value, for structured custom values (01 §3.4)
}
// sync      = a field's own sync rule
// async     = an asyncValidations source
// external  = from a submit target or mapExternalErrors (08 §5)
// group     = a group/array rule, whether it lands on the container or (via `path`) on a child
// input     = unparseable input, via setInputError (10 §2.1)
// component = reported by the renderer for its own subsystem, via report (10 §2.2)
```

- **`subPath`** points inside a field's value (`/rows/3/qty`). The message still belongs to the field: it counts toward the field's `severity`, `valid`, `firstInvalid`, and submit blocking, and it appears in `feedback`. Renderers that understand the value's structure filter by `subPath` to show it next to the matching cell. Renderers that don't just show it at the field. Custom validators, async sources, and external errors can set it. The engine doesn't check that `subPath` exists in the value.

A `Notice` is one message shown to the user, of any severity, from any source. `errors` in API names (controller `errors`, submit `reason: 'errors'`) means severity `error` specifically. Lists of every severity are called `notices`.

- Messages from each source are kept separately. `sync` and `async` messages are recomputed. `external` messages are stale once the value they describe changes, so they are cleared by:
  - any write whose origin policy targets the `current` layer with `effects: true` (built-ins: `user`, `host`, `effect`, and `x-` origins extending them; [06 §2.1](./06-effects-and-computed.md#21-origin-policy)), and also by `reset` and `restore`, which replace `current` wholesale;
  - to the path itself **or any descendant** (editing a row's field clears the row's own messages);
  - and the container-level `external` messages of every **ancestor** of the written path (a server message on `/addresses` saying "duplicate address" is stale once any address field changes). Sibling paths keep theirs.
  - Other origins (`load`, `default`, `computed`, `history`) leave them in place. Undo restores old values without restoring old server messages, so it doesn't clear them either; a reload that replaces the record normally brings fresh server state anyway ([08 §3](./08-loading-and-submission.md#3-reload-policy)).
- **Group/array rules** attach to the container path. Rules can return `{ path: './endDate', message }` to place the message on a child field instead.
- Field and form handles are listed in §7.4.

## 6. Required and conditions

`conditions.required` is sugar for a `required` rule whose `when` is that condition. It also feeds the `required` facet so renderers can show an asterisk.

## 7. Severity

Three levels, following SurveyJS `notificationType` ([13](./13-prior-art.md)). They are reserved; hosts can add `x-` severities on top of them (§7.6):

| Severity | Meaning | Blocks submit / `next()` |
|----------|---------|--------------------------|
| `error` | The value is invalid | Always |
| `warning` | The value is suspicious but allowed | Per the policy of its gate (§7.3) |
| `info` | Guidance about the value | Never |

There's no `success` severity. "Valid" is derived state (`validated && no errors`), exposed as the `validated` / `valid` handles (§7.4). A success severity would require a message on every rule.

### 7.1 Where severity comes from

| Source | Severity |
|--------|----------|
| Sync rule | `rule.severity` (default `error`) |
| Custom validator | The result's `severity` if returned, otherwise the rule's |
| Standard Schema validator | The rule's (issues carry no severity) |
| Async validation source | Per `NoticeInput.severity`, otherwise `AsyncValidationRule.severity`, otherwise `error` |
| External errors | `ExternalErrors[].severity` ([08 §5](./08-loading-and-submission.md#5-external-error-mapping)), default `error` |
| `setInputError` | Always `error` |
| `report` (component) | Per message, default `error` |

Severity is static per rule. For thresholds (warn above 10k, fail above 50k), write two rules.

### 7.2 Evaluation and display

- **Evaluation:** `bail` stops at the first `error`. `warning` and `info` rules always run, so they're ready as soon as errors are fixed. Async rules run only when there are no sync *errors*. Sync warnings don't hold them back.
- **Display, strongest wins:** the `feedback` handle returns only messages of the **highest severity present** on the field. Warnings appear once errors are resolved, and info once there are no errors or warnings.
- **Timing:** `showFeedback` uses the same rule for all severities (touched, or after the first submit, per `ValidationConfig.mode`). For guidance that should always be visible, use the field's `description`, not an info rule.
- **Hidden / disabled fields** produce no messages of any severity unless `validateHidden` / `validateDisabled` is set ([06 §4](./06-effects-and-computed.md#4-hidden-and-disabled-field-policy)).

### 7.3 Gates: submit policy for warnings

Every warning-based notice belongs to a **gate**. Plain `warning` belongs to the built-in gate `warnings`. A custom severity names its own gate (§7.6). The contract sets a policy per gate:

```ts
interface SubmissionConfig { /* … */ gates?: Record<string, 'allow' | 'confirm' | 'block'> }
// default: { warnings: 'allow' }; a gate used by a registered severity but not listed here: 'confirm'
```

| Policy | `submit()` with notices in the gate |
|--------|----------------------------------|
| `allow` | Proceeds |
| `confirm` | Stops unless the gate is confirmed. The host shows "Submit anyway?", then calls `submit({ confirm: ['warnings'] })` (or `engine.confirm(gate)` first) |
| `block` | Stops, like errors. The notices still display with their own severity. Confirmation doesn't apply |

```ts
engine.submit({ confirm?: string[] });
engine.confirm(gate: string): void;          // e.g. from an inline "I confirm" checkbox
engine.unconfirm(gate: string): void;

| { ok: false; reason: 'gated'; gates: Array<{ gate: string; policy: 'confirm' | 'block'; notices: Array<Notice & { path: DataPath }> }> }
```

- **One result for every gate.** `gates` lists each gate that stopped the submit, so the host can show one dialog with a section per gate. Errors are checked first: while there are errors, the result is `reason: 'errors'`.
- **Confirmation lifetime.** A confirmation records the notices it covered, identified by `(path, rule, code)`. It stays valid while the gate's current notices are a **subset** of those. Editing an unrelated field, or fixing a warning, keeps it; a new or changed notice in that gate drops it. `reset` and `restore` clear all confirmations. They aren't in drafts or `exportState`.
- **Audit.** Confirmed gates reach the submit target and outbound transformers as `confirmed: string[]` ([07 §1.1](./07-async-data-and-registries.md#11-shared-function-shapes)), so the server can record what the user confirmed.
- **State.** `FormState.gates: Record<string, { policy; count; confirmed }>` ([02 §5](./02-state-model.md#5-field-meta-and-form-status)) lets the UI show "2 legal notices need confirmation" without calling submit.
- **Only warnings are gated.** `error` always blocks and `info` never does. That keeps "invalid" meaningful; see [15 §14](./15-cookbook.md#14-submit-while-invalid) for deliberately submitting invalid data.

Scoped validation (`validate({ paths })`, the validation-scope hook of [02 §8.1](./02-state-model.md#81-plugin-api)) and wizard `next()` apply the same gates to their scope ([10 §5](./10-rendering-and-adapters.md#5-wizards)). A confirmation made on a step still counts at the final submit while its notices are unchanged.

### 7.4 Handles

Per field (on `FieldController`, [10 §2](./10-rendering-and-adapters.md#2-headless-field-api)):

| Handle | Value |
|--------|-------|
| `feedback` | `Notice[]` of the strongest severity present |
| `severity` | `Severity \| null`, the strongest present |
| `showFeedback` | Whether to display `feedback` now (timing rules above) |
| `errors` | `Notice[]` with severity `error` only (for logic and `aria-invalid`) |
| `notices` | Every notice, every severity (devtools, summaries) |
| `validated` | Validated at least once since the last value change |
| `valid` | `validated` and no `error` notices |

Per form (`FormState`, [02 §5](./02-state-model.md#5-field-meta-and-form-status)): `valid`, `severity` (strongest anywhere in the form), `counts: { error, warning, info, custom }` (whole form, eligible nodes; by base severity, §7.6), and `notices` (form-level only: those that map to no path). `engine.summary()` lists every notice with its path, in form order, for an error summary at the top of the form.

### 7.5 Accessibility (renderer guidance)

- Only `error` sets `aria-invalid="true"`.
- All shown messages are linked through `aria-describedby`. Errors go in a live region with `role="alert"`; warnings and info use `aria-live="polite"`.
- Don't communicate severity by color alone. Use an icon and/or a text prefix.
- Custom severities (§7.6) follow their base for all of the above.

### 7.6 Custom severities

The three core severities are reserved. Hosts can add `x-` severities, each extending one core severity:

```ts
registry.severities.register('x-critical', { extends: 'error', rank: 1 });    // above plain error
registry.severities.register('x-success',  { extends: 'info',  rank: -1 });   // below plain info
registry.severities.register('x-hint',     { extends: 'info',  rank: -2 });
registry.severities.register('x-legal',    { extends: 'warning', gate: 'legal', rank: 1 });   // own confirmation, §7.3
```

- **Names.** A name must start with `x-` (`E_SEVERITY_NAME`, thrown by `register`). A contract using an unregistered severity on a rule fails at validate-core with `E_UNKNOWN_SEVERITY`.
- **Behaviour comes from the base.** `valid`, `firstInvalid`, `aria-invalid`, live regions, and the controller `errors` handle (base `error`) all use `extends`. Blocking does too: `error`-based severities always block, `info`-based never do, and `warning`-based ones block per their **gate** (§7.3). A `warning`-based severity may name its own `gate` (default `warnings`). `gate` on an `error`- or `info`-based severity is `E_SEVERITY_GATE`. Gate names are free strings; `warnings` is the built-in gate.
- **Order.** Severities sort by base (`error > warning > info`), then by `rank` within the base (core severities have rank 0; equal rank sorts as equal). "Strongest wins" in `feedback`, plus the field and form `severity`, use this order. So `x-critical` hides plain errors in `feedback`, and `x-success` shows only when no plain `info` is present.
- **Counts.** `FormState.counts` stays keyed by base (`error`, `warning`, `info`), so `counts.error > 0` keeps meaning "something blocks". Custom severities are also counted individually in `counts.custom`.
- **Sources.** Any place that accepts a severity accepts a registered custom one: rules, validator results, async sources, `report`/`onValidate`, and external errors. An unknown severity from outside the contract (external errors, async sources, component notices) is treated as `error` (fail closed) and reported with `W_UNKNOWN_SEVERITY`.
- **Renderers** receive the actual name (`notice.severity === 'x-success'`) and style it. A renderer that doesn't know a name styles it as its base, via `severityBase(name)` on the engine.
- `x-success` is the expected common case. It doesn't contradict §7: "valid" stays derived state, and an `x-success` notice appears only where a rule explicitly produces one.
