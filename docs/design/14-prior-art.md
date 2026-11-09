# 14 · Prior Art

[← Index](./README.md)

How the design compares to existing form libraries, and what we borrowed. Claims marked **✔︎** were checked against current docs or source on 2026-10-20 (see [§6](#6-verification-log)). Everything else is from general knowledge and should be checked before it's quoted externally.

## 1. Landscape

| Library | Model | Closest overlap with us |
|---------|-------|-------------------------|
| **JSON Forms** (EclipseSource) | JSON Schema (data) + UI Schema (layout) → renderer sets | Headless core, renderer registry, multiple frameworks |
| **Form.io** | JSON component tree, hosted builder and platform | Contract-driven forms from a CMS, conditionals, calculated values, data-source selects |
| **SurveyJS** | JSON survey model, its own expression language | Conditions, cascading choices, hidden-value policy |
| **Formily** (Alibaba) | JSON Schema extended with `x-*` + reactive core | Reactive field graph, `x-reactions` linkage, headless core |
| **React Hook Form / TanStack Form** | Forms defined in code, no schema | State, validation, and subscription performance |

## 2. Capability matrix

✅ built in · ◐ partial / needs config · ❌ custom code needed · — not documented / not checked

| Capability | JSON Forms | Form.io | SurveyJS | Formily | RHF / TanStack Form | **Ours** |
|---|---|---|---|---|---|---|
| Schema / contract driven | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Show/hide/enable conditions | ✅ ✔︎ (exactly 4 effects: `HIDE`, `SHOW`, `ENABLE`, `DISABLE`) | ✅ | ✅ | ✅ ✔︎ (`x-reactions`) | ❌ | ✅ |
| Computed values | ❌ | ✅ ✔︎ (`calculateValue`) | ✅ (expressions) | ✅ | ❌ | ✅ |
| Effects on other fields | ◐ ✔︎ (only through middleware, see §3) | ◐ | ◐ (triggers) | ✅ ✔︎ (`x-reactions` with `target`) | ❌ | ✅ (origin-aware) |
| Cascading selects | ❌ | ✅ ✔︎ (`refreshOn`, `clearOnRefresh`) | ✅ ✔︎ (`choicesByUrl` with `{placeholder}` URLs) | ✅ (reactions) | ❌ | ✅ (doc 13) |
| Writes tagged by origin (loading data can't trigger clears) | — | — | — | — | — | ✅ |
| Async / server validation | ◐ ✔︎ (`additionalErrors` injection only) | ✅ | ◐ | ✅ | ✅ | ✅ |
| Hidden-value policy | ❌ | ✅ ✔︎ `clearOnHide` (default `true`) | ✅ ✔︎ `clearInvisibleValues` (4 modes, default `onComplete`) | ◐ | ◐ `shouldUnregister` ✔︎ (default off) | ✅ hierarchical |
| Layered state (default / loaded / user) | ❌ ✔︎ (single `data` object) | ❌ | ❌ | ◐ (`initialValues` vs `values`) | ◐ (defaults vs values) | ✅ |
| Stable array row identity | ❌ | ❌ | ◐ | ◐ | ✅ ✔︎ (RHF `useFieldArray` `keyName='id'`) | ✅ |
| No eval of remote code | ✅ | ◐ ✔︎ (`new Function` by default; a `noeval` flag turns it off) | ✅ | ❌ ✔︎ (`new Function` + `with`) | n/a | ✅ |
| Validation cost per change | Whole form ✔︎ (deep clone + full AJV run on each `UPDATE_DATA`) | — | — | — | Per field | Per affected field |
| Ready-made styled renderers | ✅ ✔︎ (React Material, Angular Material, React Vanilla, Vue Vanilla, Vue Vuetify) | ✅ | ✅ | ✅ | n/a | ❌ (unstyled only) |
| Standard Schema | ❌ (AJV) | ❌ | ❌ | ❌ | ✅ ✔︎ (TanStack Form) | ✅ |

## 3. Notes per library

### JSON Forms
- **Rules ✔︎:** `rule: { effect: HIDE|SHOW|ENABLE|DISABLE, condition: { scope, schema } }`. The condition is a JSON Schema validated against the data at `scope`. An `undefined` scope **passes** unless `failWhenUndefined: true` is set. That's a surprising default worth knowing about.
- **Validation ✔︎:** done by AJV. The modes are `ValidateAndShow` (default), `ValidateAndHide`, and `NoValidation`. Backend errors are injected with `additionalErrors`. In the core reducer, every `UPDATE_DATA` deep-clones the whole data object and re-validates the whole form, which is the root of its large-form performance issues.
- **Testers ✔︎:** every renderer is registered with a tester `(uischema, schema) => number`, and the highest number wins (`rankWith`).
- **Middleware ✔︎:** it **replaces the reducer** ("can apply arbitrary changes and therefore has full power over the JSON Forms state"). It receives the state, the action, and the default reducer. We borrowed the hook point but **deliberately narrowed** it ([03 §2.5](./03-state-model.md#25-commit-hooks)): ours can observe and veto, but can't rewrite changes, so the dependency graph stays the only path for value changes.
- **i18n ✔︎:** `translate(key, defaultMessage, context)`. Keys default to the property path **without array indices** (`comments.message.label`) and can be overridden with an `i18n` option.
- **Why we don't build on it:** its state is one `data` object behind a reducer, with no layers, origins, or row keys. We'd bypass its state model and keep only its renderers.

### Form.io
- **Defaults ✔︎:** components default to `clearOnHide: true`. Selects have `refreshOn` (a field to watch) and `clearOnRefresh` (default `false`). This is the same shape as our `dependsOn` + `cascade.onChange`.
- **Code evaluation ✔︎:** `@formio/core`'s `DefaultEvaluator` compiles custom logic with `new Function(...)`. Passing `noeval: true` disables evaluation (it logs a warning and returns a no-op), which also disables those custom-JS features. Our design has no eval to switch off ([05 §4](./05-expressions.md#4-security-model)).
- **Not checked:** how `clearOnRefresh` behaves when a form is first loaded with existing data.

### SurveyJS
- **Hidden values ✔︎:** `clearInvisibleValues` is one of `"none"`, `"onComplete"` (default), `"onHidden"`, or `"onHiddenContainer"` (`true`/`false` are aliases for `onComplete`/`none`). How they map to ours:

  | SurveyJS | Ours (`whenHidden`) |
  |----------|---------------------|
  | `none` | `keep` |
  | `onComplete` | `exclude` (kept while editing, dropped from the result) |
  | `onHidden` | `clear` |
  | `onHiddenContainer` | `clear` (our policies are already inherited from containers) |

  Their default is effectively our `exclude`. Our default is `keep`, chosen to match general-purpose form libraries ([07 §4](./07-effects-and-computed.md#4-hidden-and-disabled-field-policy)).
- **Cascading ✔︎:** `choicesByUrl.url` and `path` go through the text processor (`{question}` placeholders). If any placeholder has no value, the URL is set to empty and **no request is made**, which is the same as our `whenEmpty: 'disable'`. It's simple, but tied to HTTP URLs. Our request transformers generalize it.

### Formily
- **Reactions ✔︎:** `x-reactions` entries have `dependencies`, `when`, `target`, `effects`, `fulfill`/`otherwise` (`state`, `schema`, `run`), or a function. This is the closest equivalent to our effects and conditions.
- **Expressions ✔︎:** `{{ … }}` strings are compiled with `new Function('$root', 'with($root) { return (…) }')`. This breaks under strict CSP, and it is the kind of remote code execution we rule out.
- **Not checked:** the details of its state layering.

### React Hook Form / TanStack Form
- RHF `useFieldArray` gives each row a generated `id` (`keyName` defaults to `'id'`) ✔︎, which is the precedent for our row keys. `shouldUnregister` is off by default, so hidden or unmounted values are kept ✔︎. That informed our `whenHidden: 'keep'` default.
- TanStack Form accepts Standard Schema validators (Zod, Valibot, ArkType, Effect Schema) directly ✔︎.

## 4. What we borrowed

| Idea | From | Where |
|------|------|-------|
| Ranked tester renderer resolution | JSON Forms | [11 §4.1](./11-rendering-and-adapters.md#41-ranked-testers) |
| JSON Schema export of the submitted data | JSON Forms (schema reuse) | [09 §6](./09-loading-and-submission.md#6-json-schema-export-of-the-submitted-data) |
| JSON Schema import (`fromJsonSchema`) | JSON Forms | [01 §8](./01-schema-contract.md#8-json-schema-import) |
| Commit middleware (narrowed to observe and veto) | JSON Forms middleware | [03 §2.5](./03-state-model.md#25-commit-hooks) |
| Hidden-value policy semantics | SurveyJS, Form.io | [07 §4](./07-effects-and-computed.md#4-hidden-and-disabled-field-policy) |
| Watch-field + clear-on-refresh cascade shape; "no request until deps are filled" | Form.io, SurveyJS | [13](./13-option-sources-and-cascading.md) |
| Stable array row ids | React Hook Form | [04 §4](./04-paths-and-dependency-graph.md#4-array-row-identity) |
| Standard Schema validators | TanStack Form | [06 §2.1](./06-validation.md#21-standard-schema-validators) |
| Expression defaults (`defaultValueExpression`) | SurveyJS | [07 §3.1](./07-effects-and-computed.md#31-expression-defaults-replaces-mode-initial) |
| Per-option disable (`choicesEnableIf`) | SurveyJS | [13 §2.1.1](./13-option-sources-and-cascading.md#211-disabling-options) |
| Raw input vs. value (kept in the renderer, with `setInputError`) | Formily `inputValue`, SurveyJS `textUpdateMode` | [11 §2.1](./11-rendering-and-adapters.md#21-raw-input-stays-in-the-renderer) |
| Three severities, strongest-wins display | SurveyJS `notificationType` (`error`/`warning`/`info`) ✔︎. Formily's `success` was not adopted | [06 §7](./06-validation.md#7-severity) |
| Variables, "Other"/"None", carry-forward, `resetValueIf`, `multiple`, as macros | SurveyJS, Form.io | [15 §7](./15-macros.md#7-standard-library-formgraphmacros-optional) |
| Autosave, draft rows, view mode, page modes, `persistent: false`, `canSubmitWhenInvalid`, `delayError`, row labels, tabs, as recipes | SurveyJS, Form.io, Formily, RHF, TanStack Form, JSON Forms | [16](./16-cookbook.md) |

## 5. Where we deliberately differ

- **No remote code.** Form.io (unless `noeval`) and Formily compile schema strings with `new Function`. We only allow registered functions.
- **Write origins.** None of the libraries surveyed documents tagging writes by origin. Origins are what make edit mode safe for effects, cascades, and auto-select.
- **Layered state.** Keeping loaded data and user edits separate gives us revert, dirty tracking, and reload policies without the host having to diff.
- **Narrow middleware.** Unlike JSON Forms, middleware can't rewrite state, so behavior stays visible to the dependency graph and devtools.
- **Per-field work.** We avoid JSON Forms' whole-form clone and validation on every change.
- **No styled renderers or builder.** We compete on the engine, not the UI kit.

## 6. Verification log

Checked 2026-10-20:

| Claim | Source |
|-------|--------|
| JSON Forms rule effects, `SchemaBasedCondition`, `failWhenUndefined` | jsonforms.io/docs/uischema/rules |
| JSON Forms validation modes, AJV, `additionalErrors` | jsonforms.io/docs/validation |
| JSON Forms whole-form clone + validate on `UPDATE_DATA` | `eclipsesource/jsonforms` `packages/core/src/reducers/core.ts` |
| JSON Forms middleware semantics | jsonforms.io/docs/middleware |
| JSON Forms i18n key rules | jsonforms.io/docs/i18n |
| JSON Forms testers / `rankWith` | jsonforms.io/docs/tutorial/custom-renderers |
| JSON Forms renderer sets | jsonforms.io/docs/renderer-sets |
| Form.io `clearOnHide: true`, `refreshOn`, `calculateValue` defaults | `formio/formio.js` `src/components/_classes/component/Component.js` |
| Form.io `clearOnRefresh: false` | `formio/formio.js` `src/components/select/Select.js` |
| Form.io `new Function` + `noeval` | `formio/core` `src/utils/Evaluator.ts` |
| SurveyJS `clearInvisibleValues` modes and default | surveyjs.io API reference; `survey-library` `packages/survey-core/src/survey.ts` |
| SurveyJS `choicesByUrl` placeholder processing | `survey-library` `packages/survey-core/src/choicesRestful.ts` |
| Formily `new Function` + `with` compiler | `alibaba/formily` `packages/json-schema/src/compiler.ts` |
| Formily `x-reactions` shape | `alibaba/formily` `packages/json-schema/src/types.ts` |
| RHF `keyName='id'`, `shouldUnregister` | `react-hook-form` `src/useFieldArray.ts`, `src/logic/createFormControl.ts` |
| TanStack Form Standard Schema | tanstack.com/form validation guide |
| SurveyJS validator `notificationType` (`error`/`warning`/`info`, strongest shown) | `survey-library` `packages/survey-core/src/validator.ts`, `survey-error.ts` |
| Formily feedback types `'error' \| 'success' \| 'warning'` | `alibaba/formily` `packages/core/src/types.ts` |

**Not verified:** JSON Forms async options (none found in the docs reviewed, but absence isn't proven), Form.io async validation details, SurveyJS async validation, Formily state layering, and visual-builder availability.
