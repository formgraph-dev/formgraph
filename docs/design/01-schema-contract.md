# 01 · Schema Contract

[← Index](./README.md)

## 1. Principles

- The contract is **pure JSON**: no functions, no code strings, no `Date` objects.
- Anything that varies between deployments is referenced by **string ID** and resolved through client registries.
- Every node has a stable `id`. Ids must be unique within their **data scope** (the nearest enclosing `dataScope` group, array row, or the root), because the id is the data key ([04 §3](./04-paths-and-dependency-graph.md#3-data-keys-and-scoping)). The compiler reports duplicates as `E_DUPLICATE_DATA_KEY`.
- **Id grammar:** `^[A-Za-z_][A-Za-z0-9_-]*$` (`E_INVALID_ID`). `__proto__`, `constructor`, and `prototype` are reserved and rejected as ids, data keys, and `serializeAs` names (`E_RESERVED_KEY`). Both are checked at validate-input. State layers ([03](./03-state-model.md)) use null-prototype objects (`Object.create(null)`) regardless.
- Unknown properties prefixed with `x-` are preserved and passed to renderers untouched. This is the extension point. The engine doesn't check `x-` keys (it owns no schema for them, and a typo can't be told from another extension); whoever uses one, such as a renderer or plugin, checks it where it reads it. `x-` keys are allowed on **every contract object**: `FormContract`, nodes, option sources, `SelectOption`, rules, effects, and expressions' host objects (not inside `expr`). Each of these types carries the `Extensible` index signature below.

## 2. Primitives

```ts
export type Json = string | number | boolean | null | Json[] | { [k: string]: Json };
export type Scalar = string | number | boolean;

/** Mixed into every contract object type (§1). */
export interface Extensible { [ext: `x-${string}`]: Json | undefined }

/** Evaluated by a registered evaluator. See 05-expressions.md */
export interface Expression {
  lang: string;                 // 'json-logic' | 'fn' | plugin id. Always explicit (05 §1)
  expr: Json;
}
export type Condition = boolean | Expression;

/** Explicit reference to another node. Bare names are rejected (04 §2) */
export type PathRef = '.' | `/${string}` | `./${string}` | `../${string}` | `$${string}`;

/** See 10-localization.md */
export interface LocalizableString {
  key?: string;
  default: string;              // `{name}`; ICU subset with the icu plugin (10 §2.1)
  params?: Record<string, Json | Expression>;
}
export type Text = string | LocalizableString;   // bare string = default with no key

export interface SelectOption extends Extensible {
  label: Text;
  value: Scalar;
  disabled?: boolean;
  meta?: Record<string, Json>;
  group?: { value: Scalar; label: Text };   // display grouping (13 §2.2.1): set by the engine for map + groupBy, by result transformers for providers
}
```

### 2.1 Enumerated values

Many keys take one of a fixed set of strings (`kind`, `fieldType`, `source`, `whenHidden`, `origin`, …). They follow three rules:

1. **Kebab-case values, camelCase keys.** Every enumerated value is lowercase kebab-case: `dynamic-group`, `multi-choice`, `keep-if-valid`, `json-logic`, `date-time`. Object keys stay camelCase (`whenHidden`, `maxLength`). Validator ids are values too, so the `maxLength` config key implies the `max-length` rule. Diagnostic codes (`E_…`) are a separate, uppercase namespace ([17](./17-diagnostics.md)).
2. **One source of truth.** Each set is exported once as a `const` tuple, and the TS type, the runtime validator, and the published JSON Schema `enum` are all derived from it:

   ```ts
   export const HIDDEN_POLICIES = ['keep', 'exclude', 'clear'] as const;
   export type HiddenPolicy = typeof HIDDEN_POLICIES[number];
   ```

   Host code can iterate them (for example to build a CMS dropdown). Open sets (`fieldType`, `ui.variant`, effect `action`, `lang`, macro ids) additionally accept `x-…` values. Tuples defined in this doc: `GRANULARITIES`, `ASSET_STATUSES`, `ORPHAN_POLICIES`, `LOADING_BEHAVIORS`, `FACETS` (§7.3).

   **One exemption:** macro ids may carry an `@MAJOR` version suffix (`choice-with-other@2`, [15 §5](./15-macros.md#5-rules)). The part before `@` is kebab-case; the suffix is a positive integer.
3. **Same word, same meaning.** Where a value appears in several sets, it means the same thing:

   | Value | Always means |
   |-------|--------------|
   | `clear` | Write the field type's **empty** value (`''`, `null`, `[]`, `false`) into `current` (effect action, `whenHidden`, cascade `onChange`) |
   | `reset` | Write the field's **default** value into `current`, ignoring loaded (effect action, cascade `onChange`); the `reset` origin is `engine.reset()` doing this form-wide |
   | `keep` | Leave the value alone (`whenHidden`, `whenDisabled`, cascade `onChange`) |
   | `exclude` | Keep the value in state, leave it out of the payload (`whenHidden`, `whenDisabled`) |
   | `change` / `blur` / `submit` | The same three events everywhere (rule `trigger`, effect `on`, validation `mode`) |

## 3. Fields

### 3.1 Base

```ts
export interface NodeBase extends FieldPolicies, Extensible {   // whenHidden/whenDisabled/validate*, see 07 §4
  id: string;
  label?: Text;
  description?: Text;
  layout?: LayoutHints;
  conditions?: Conditions;
}

export interface FieldBase<FT extends string, V, UI> extends NodeBase {
  kind: 'field';
  fieldType: FT;
  placeholder?: Text;
  defaultValue?: V | Expression;              // expression = live default, see 07 §3.1
  validations?: ValidationRule[];             // 06-validation.md
  asyncValidations?: AsyncValidationRule[];
  effects?: Effect[];                         // 07-effects-and-computed.md
  computed?: ComputedConfig;
  serializeAs?: string;                       // 09-loading-and-submission.md
  ui?: UI;                                    // renderer-only hints, typed per fieldType (§3.4)
  fallback?: FieldFallback;                   // x- field types only: what to use when this client lacks the type or its renderer (§3.5)
}
```

> `type` was renamed to `kind` so it doesn't collide with `fieldType` and the many libraries that use `type`.

### 3.2 Field type registry

Every field type is declared once, in a registry interface that maps a `fieldType` to its **value**, its **config** (engine-read keys, placed flat on the node), and its **ui** (renderer-only hints, in the `ui` object):

```ts
export interface FieldTypeRegistry {
  text:           { value: string;          config: TextConfig;        ui: TextUi };
  number:         { value: number | null;   config: NumberConfig;      ui: NumberUi };
  boolean:        { value: boolean;         config: {};                ui: BooleanUi };
  'date-time':    { value: string;          config: DateTimeConfig;    ui: DateTimeUi };
  choice:         { value: Scalar | null;   config: ChoiceConfig;      ui: ChoiceUi };
  'multi-choice': { value: Scalar[];        config: MultiChoiceConfig; ui: ChoiceUi };
  asset:          { value: AssetRef | null; config: AssetConfig;       ui: AssetUi };
  'multi-asset':  { value: AssetRef[];      config: MultiAssetConfig;  ui: AssetUi };
}

export type FieldType = keyof FieldTypeRegistry;
type Def<FT extends FieldType> = FieldTypeRegistry[FT];

/** A field of a given type: shared keys + its config keys (flat) + typed `ui`. */
export type FieldOf<FT extends FieldType> =
  FieldBase<FT, Def<FT>['value'], Def<FT>['ui']> & Def<FT>['config'];

/** Known core/merged entries plus a runtime envelope for locally registered custom types. */
export type KnownField = { [FT in FieldType]: FieldOf<FT> }[FieldType];
export interface CustomField extends FieldBase<`x-${string}`, Json, Record<string, Json>> {
  [configKey: string]: unknown;   // wire envelope only; runtime input must still be JSON
}
export type Field = KnownField | CustomField;

export type ValueOf<FT extends FieldType> = Def<FT>['value'];

/** Strong view for a typed registry's entries, without the open wire envelope. */
export type FieldFor<E extends Pick<RegistryEntries, 'fieldTypes'>> = {
  [FT in keyof E['fieldTypes'] & string]:
    FieldBase<FT, E['fieldTypes'][FT]['value'], E['fieldTypes'][FT]['ui']> & E['fieldTypes'][FT]['config']
}[keyof E['fieldTypes'] & string];
```

`Field` is the permissive wire/runtime union, so an `x-` discriminant alone cannot prove an open field's config types. Use `FieldFor<E>` for registry-checked typed contracts and `KnownField` (or `FieldOf<'x-…'>`) for the declaration-merging fallback. Those strong views exclude `CustomField`; narrowing a merged `x-signature` then retains its `minStrokes?: number`. Do not cast an arbitrary wire field to a strong view based only on its name: the actual registry/config validation must have succeeded, not merely a missing-type fallback.

Narrowing works as usual:

```ts
function describe(f: Field) {
  if (f.fieldType === 'choice') {
    f.options;            // OptionsSource | undefined
    f.ui?.variant;        // 'select' | 'radio' | 'buttons' | 'combobox' | `x-${string}` | undefined
  }
}
```

### 3.3 Core config and ui

**Config** is read by the engine and lives **flat on the node**:

```ts
interface TextConfig        { minLength?: number; maxLength?: number }   // implied rules, §3.3.1
interface NumberConfig      { min?: number; max?: number; step?: number; precision?: number }   // min/max: implied rules
export const GRANULARITIES = ['date', 'time', 'date-time'] as const;
interface DateTimeConfig    {
  granularity?: typeof GRANULARITIES[number];
  timezone?: string;                          // open string, not an enum: 'utc', 'local', or an IANA zone ('Europe/Berlin'); default 'local'
  min?: string | Expression; max?: string | Expression;   // ISO bounds or live expressions (e.g. `$context.now`); implied rules, §3.3.1
}
interface ChoiceConfig {                                                 // see 13
  options?: OptionsSource;            // input may also be a bare SelectOption[] (normalized, 15 §6)
  cascade?: CascadeConfig;
  autoSelectSingle?: boolean; onSelect?: Effect[];
  resolveSelectedSourceId?: string; validateInOptions?: boolean;
}
interface MultiChoiceConfig extends ChoiceConfig { minSelected?: number; maxSelected?: number }
interface AssetConfig       { accept?: string[]; maxBytes?: number; storeId: string; concurrency?: number }   // storeId required, no default; concurrency: 08 §2.1
interface MultiAssetConfig  extends AssetConfig { maxFiles?: number }
```

**UI** is never read by the engine. It can't change values, validity, or submission:

```ts
interface TextUi     { variant?: 'input' | 'textarea' | `x-${string}`; rows?: number; inputMode?: 'text' | 'email' | 'tel' | 'url' | 'numeric'; mask?: string; autocomplete?: string }
interface NumberUi   { variant?: 'input' | 'slider' | `x-${string}`; format?: 'plain' | 'currency' | 'percent'; currency?: string; stepper?: boolean }
interface BooleanUi  { variant?: 'checkbox' | 'switch' | `x-${string}` }
interface DateTimeUi { variant?: 'calendar' | 'input' | `x-${string}`; displayFormat?: string }
interface ChoiceUi   { variant?: 'select' | 'radio' | 'buttons' | 'combobox' | `x-${string}`; columns?: number }
interface AssetUi    { variant?: 'dropzone' | 'button' | `x-${string}`; showPreview?: boolean }
```

Every `ui` type (fields, groups, arrays) also accepts `textFormat?: string`: the id of a registered text formatter for opt-in rich text in labels and descriptions ([08 §1](./08-async-data-and-registries.md#1-registries), [10](./10-localization.md)). Without it, text is plain.

| `fieldType` | Value | Config (flat) | `ui` |
|-------------|-------|---------------|------|
| `text` | `string` | `minLength`, `maxLength` | `variant`, `rows`, `inputMode`, `mask`, `autocomplete` |
| `number` | `number \| null` | `min`, `max`, `step`, `precision` | `variant`, `format`, `currency`, `stepper` |
| `boolean` | `boolean` | | `variant` |
| `date-time` | `string` (ISO 8601) | `granularity`, `timezone`, `min`, `max` | `variant`, `displayFormat` |
| `choice` | `Scalar \| null` | `options`, `cascade`, `autoSelectSingle`, `onSelect`, … ([13](./13-option-sources-and-cascading.md)) | `variant`, `columns` |
| `multi-choice` | `Scalar[]` | same as `choice`, plus `minSelected`, `maxSelected` | same as `choice` |
| `asset` | `AssetRef \| null` | `accept`, `maxBytes`, `storeId`, `concurrency` | `variant`, `showPreview` |
| `multi-asset` | `AssetRef[]` | same as `asset`, plus `maxFiles` | same as `asset` |

```json
{ "kind": "field", "id": "state", "fieldType": "choice", "label": "State",
  "options": { "source": "map", "key": "./country", "map": { "US": [ … ] } },
  "autoSelectSingle": true,
  "ui": { "variant": "radio", "columns": 2 } }
```

Rules:
1. **Unknown top-level keys are validation errors** (`E_UNKNOWN_PROPERTY`, load steps 3 and 6), except `x-` keys. Typos in engine config can't pass silently. A contract with a newer minor version downgrades this to `W_UNKNOWN_PROPERTY_NEWER_MINOR` and drops the key (§7). Nodes with an **unregistered** `x-` fieldType skip unknown-key checks and keep all their keys (there's no schema to check against).
2. **Reserved names:** config keys may not reuse `NodeBase` / `FieldBase` keys. This is checked at type level and when a field type is registered (`E_RESERVED_CONFIG_KEY`, thrown by `register`).
3. **Unknown `ui` keys are warnings** (`W_UNKNOWN_UI_PROPERTY`), not errors, because ui is non-semantic. With `strict: true` they're errors.
4. **`ui.variant`** is the standard key for picking a presentation on every node type (fields, groups, arrays). The default renderer lookup uses it ([11 §4](./11-rendering-and-adapters.md#4-renderer-registry)).
5. **Where does a key go?** If the engine needs it to compute values, validity, options, or submission, it's config. Otherwise it's ui. A `mask` that only formats display is ui. Parsing lives in renderers anyway (D14).

#### 3.3.1 Implied rules

Some config keys imply validation rules. The compiler adds them **before** the field's own `validations`, with the default message and severity `error`. A field's explicit rule of the same `type` replaces the implied one (to change the message or severity).

| Config key | Implied rule (06 §2) | Payload schema ([09 §6](./09-loading-and-submission.md#6-json-schema-export-of-the-submitted-data)) |
|------------|----------------------|------------------|
| `text.minLength` / `maxLength` | `min-length` / `max-length` (an empty string passes `min-length`; use `required` to demand input) | `minLength` / `maxLength`, with an empty-string branch for optional text (09 §6) |
| `number.min` / `max` | `min` / `max` | `minimum` / `maximum` |
| `number.step` | `step` | `multipleOf` |
| `date-time.min` / `max` | `min` / `max` (compared as described under value shapes) | `formatMinimum` / `formatMaximum` only for literal bounds with compatible exported formats/comparators; otherwise `x-formgraph-unexported` (09 §6) |
| `multi-choice.minSelected` / `maxSelected` | `min-items` / `max-items` | `minItems` / `maxItems` |
| any options source | `in-options` (unless `validateInOptions: false`, 13 §4.2) | `enum` for static sources, union of all lists for `map`, omitted for `provider` |
| `asset.accept` / `maxBytes` | `accept` / `max-bytes` | not exported |
| `array.minItems` / `maxItems` | `min-items` / `max-items` | `minItems` / `maxItems` |
| `conditions.required` on an array | `min-items` of 1 with `when` = the condition (unless `minItems` is larger) | `minItems: 1` when the condition is literal `true` |
| `multi-asset.maxFiles` | `max-items` | `maxItems` |

Value shapes for complex fields:
- Date values are **always ISO strings** in state, so they serialize and compare without surprises. Storage by `granularity` and `timezone`:

  | | `utc` | IANA zone (`Europe/Berlin`) | `local` |
  |---|---|---|---|
  | `date-time` | `2026-10-21T07:00:00Z` | `2026-10-21T09:00:00+02:00` | `2026-10-21T09:00:00` (no offset) |
  | `date` | `2026-10-21` | `2026-10-21` | `2026-10-21` |
  | `time` | `07:00:00` | `09:00:00` | `09:00:00` |

  `date` and `time` never carry an offset or zone. The zone id isn't stored with the value: the field's `timezone` config names it. Renderers produce the value (D14), so a renderer for a `utc` or IANA field must include the offset. IANA zones need `Intl` for offsets ([D24](./README.md#key-decisions-adr-summary)); without it, a contract that names one fails at compile with `E_TIMEZONE_UNSUPPORTED`, while `utc` and `local` keep working.
- **`min` / `max` comparison.** Values with an offset compare as instants. Values without one (`local`, `date`, `time`) compare as strings in field order, which is chronological for ISO forms. A bound is parsed like a value of the field; a date-only bound on a `date-time` field means the start of that day in the field's timezone. An expression bound re-evaluates live and is skipped while it yields `null`. "Now" comes from the host: `{ "min": { "lang": "json-logic", "expr": { "var": "$context.now" } } }` ([05 §3](./05-expressions.md#3-evaluation-scope)); the engine has no clock of its own for contracts.
- Assets are stored as references, never as blobs:

```ts
export const ASSET_STATUSES = ['pending', 'processing', 'ready', 'error'] as const;
export interface AssetRef {
  status: typeof ASSET_STATUSES[number];   // lifecycle: 08 §2.1
  ref?: string;          // opaque reference returned by the asset store
  name: string; size: number; mime: string;
  meta?: Json;           // returned by the store (dimensions, hash, …)
  error?: Text;          // set with status 'error'
  progress?: number;     // 0..1
}
```

`asset` / `multi-asset` are reserved built-in type names implemented by the `assets` plugin ([08 §2.1](./08-async-data-and-registries.md#21-asset-stores)), which the default entry installs. On slim core without it, a contract with an asset field fails at compile with `E_PLUGIN_MISSING`. The form is not submittable while any asset is `pending` or `processing`, or in `error` (pending-work gate, §3.4). On submit, a `ready` asset serializes as `{ ref, name, size, mime, meta? }` (`status`, `progress`, and `error` are stripped). An outbound transformer can reduce that to `ref` alone.

There's no separate "search" type: search is a `provider` options source with a `query` trigger on `choice`/`multi-choice`. Cascading selects aren't a separate type either ([13](./13-option-sources-and-cascading.md)).

### 3.4 Custom field types and typed ui extensions

Custom types are registered with Standard Schemas. Typed registration returns a registry whose type includes the new entry, so contracts written with `defineContract` are checked against it and their values are inferred ([21](./21-type-inference.md)):

```ts
const registry = coreRegistry.extend().fieldType('x-signature', {
  value:  v.nullable(v.string()),                                   // data URL or asset ref
  config: v.object({ minStrokes: v.optional(v.number()) }),        // flat on the node, engine-visible; any Standard Schema
  ui:     v.object({ penColor: v.optional(v.string()), height: v.optional(v.number()) }),
  empty:  null,
});
```

Declaration merging (`interface FieldTypeRegistry`) remains as a fallback for registries assembled where their type can't flow to the contract ([21 §6](./21-type-inference.md#6-fallback-declaration-merging)).

- Custom field types also declare their **empty value** (`empty: Json`, used by `clear` and `required`) and, optionally, a **payload schema** (`payloadSchema: JsonSchema`) for `toPayloadSchema`. Without one, the field exports as `{}` and is listed in `x-formgraph-unexported`.
- **Values may be any JSON**: scalars, objects, or arrays. The engine treats a field value as one atomic leaf ([03 §3.1](./03-state-model.md#31-resolution-granularity)). It doesn't index into it and creates no paths or controllers inside it. Messages about a part of the value carry a `subPath` ([06 §5](./06-validation.md#5-notice-shape)). If parts of the value need their own conditions, effects, or dependencies, model it as a group or array instead. The value must be JSON, so drafts, `exportState`, history, and submit work unchanged.
- `equals` and `isEmpty` are optional. Define them when structural equality is wrong for the type: `{ rows: [] }` can count as empty, and key order or float noise can be ignored.
- Two optional hooks let a field type hold work outside the value ([03 §8.1](./03-state-model.md#81-plugin-api)); the `assets` plugin implements both:

  ```ts
  interface FieldTypeImpl<V> {
    config: StandardSchema; ui: StandardSchema; value: StandardSchema; empty: Json; payloadSchema?: JsonSchema;
    equals?(a: V, b: V): boolean;                                       // default deepEqual; dirty, history, conflicts (03 §4)
    isEmpty?(value: V): boolean;                                        // default deepEqual(value, empty); used by `required`
    pending?(value: V, path: DataPath): 'idle' | 'busy' | 'failed';   // pending-work gate
    toSnapshot?(value: V): V | undefined;                              // drafts filter; undefined = leave out
  }
  ```

  - **Pending-work gate.** `submit()` and `next()` treat `busy` like in-flight async validation ([06 §4](./06-validation.md#4-async-validation)): with `submission.awaitAsync` they wait, otherwise they fail. `failed` blocks like a validation error. The type re-evaluates through the store, so changes notify as usual. Renderers can also hold the gate for busy state that isn't in the value, with `controller.setBusy` ([11 §2.2](./11-rendering-and-adapters.md#22-component-reported-messages-and-busy-state)).
  - **Snapshot filter.** `snapshot()` stores `toSnapshot(value)` instead of the value ([09 §8](./09-loading-and-submission.md#8-snapshots-drafts)).
- Custom `fieldType`s must start with `x-` (otherwise `E_FIELD_TYPE_PREFIX`). Their config keys are flat like core ones. The engine validates them with the registered `config` and `ui` schemas (`E_FIELD_CONFIG` on failure), but gives them no meaning beyond being available to validators, effects, and renderers.
- An unregistered `x-` fieldType is `W_UNKNOWN_FIELD_TYPE` (an error with `strict`); the node renders through the fallback renderer. With a contract `fallback` (§3.5), it compiles as that core type instead. It needs both a type registration and a renderer ([11 §4](./11-rendering-and-adapters.md#4-renderer-registry)).

**Renderer sets can extend the `ui` of core types** without forking them:

```ts
declare module '@formgraph/contract' {
  interface ChoiceUi { chipColor?: 'primary' | 'neutral' }     // interface merging: renderer packages can't see the host's registry type
}
registry.fieldTypes.extendUi('choice', v.object({ chipColor: v.optional(v.picklist(['primary', 'neutral'])) }));
```

Renderers receive the typed `ui`. A React renderer for `choice` gets `ui: ChoiceUi` (including extensions), not `Record<string, unknown>`.

### 3.5 Contract fallbacks for custom types

A contract can say what to use when a client doesn't support one of its custom types, like `<input type="date">` degrading to text:

```json
{ "kind": "field", "id": "summary", "fieldType": "x-agentic-description",
  "fallback": { "fieldType": "text", "readOnly": true, "ui": { "variant": "textarea", "rows": 6 } } }
```

```ts
type CoreFieldType = 'text' | 'number' | 'boolean' | 'date-time' | 'choice' | 'multi-choice';   // not asset types: they come from a plugin
type FieldFallback = {
  [FT in CoreFieldType]: { fieldType: FT; readOnly?: boolean; ui?: Def<FT>['ui'] } & Def<FT>['config']
}[CoreFieldType];
```

- **Only on `x-` field types, only to a core type** (not `asset`/`multi-asset`, which come from a plugin). Core types are always registered, so a fallback always resolves and can't chain. Its config and `ui` are checked against the target type. Otherwise `E_FALLBACK_TARGET`.
- **Type not registered in this engine.** The node compiles as the fallback type: value schema, empty value, `equals`, renderer. Its own `validations` still apply where their validators are registered. Instead of `W_UNKNOWN_FIELD_TYPE` (*strict → error*), load reports `W_FIELD_TYPE_FALLBACK`, which `strict` doesn't escalate: the author planned for it.
- **Type registered, no renderer on this client.** The engine keeps the custom type (its validators, `equals`, hooks). Only renderer lookup changes: exact match, then `fieldType`, then the contract's `fallback` type, then the host's `fallback` ([11 §4](./11-rendering-and-adapters.md#4-renderer-registry)). The renderer gets the same controller, with the fallback's `ui`.
- **The value never changes shape.** Path, value and payload are the same with or without the fallback, so the server sees no difference. If the current value doesn't match the fallback type's value (a structured value shown as `text`), the field is shown **read-only** and the value is kept untouched, with `W_FALLBACK_VALUE` once per path. It can't be edited into a different shape.
- `readOnly: true` forces read-only while the fallback is in use, for types that can be shown but shouldn't be edited without their real control.
- Typed contracts ([21 §2](./21-type-inference.md#2-typed-contracts)) check `fallback` against the core type.

### 3.6 Content nodes

A `content` node is display-only: a heading, a paragraph of help text or a divider inside a form. It holds no value.

```ts
export interface Content extends Extensible {          // not a NodeBase: no field policies (whenHidden, validate*, …)
  kind: 'content';
  id: string;                                          // layout and renderer key only; no data path
  label?: Text;                                        // heading text, or an optional caption
  description?: Text;                                  // paragraph text
  layout?: LayoutHints;
  conditions?: { visible?: Condition };                // the only condition: others are E_UNKNOWN_PROPERTY
  ui?: ContentUi;
}
export interface ContentUi {
  variant?: 'heading' | 'text' | 'divider' | `x-${string}`;   // default 'text'
  level?: 1 | 2 | 3 | 4 | 5 | 6;                       // heading level, an accessibility hint
  textFormat?: string;                                 // opt-in rich text, as on every ui (§3.3)
}
```

```json
{ "kind": "content", "id": "billingIntro", "label": "Billing", "description": "We use this address on your invoice.",
  "ui": { "variant": "heading", "level": 2 },
  "conditions": { "visible": { "lang": "json-logic", "expr": { "var": "./needsInvoice" } } } }
```

- **No data.** A content node has no data path ([04 §3](./04-paths-and-dependency-graph.md#3-data-keys-and-scoping)): it is absent from values, the payload and `toPayloadSchema`, from notices, dirty and touched tracking, and from `counts`. Validation, gates, `firstInvalid` and `submit()` ignore it. Nothing can reference it in a `PathRef` (`E_UNKNOWN_PATH`), though its `id` must be unique like any node id.
- **Visibility.** `conditions.visible` is evaluated in the enclosing scope, so `./field` works inside array rows. It's a subscriber in the dependency graph, never a dependency.
- **Where.** A child of the root, a group, a wizard step, a dynamic group's sub-schema or an array item's group. It can't be an array's `item` (an array of content has no values); that is `E_SCHEMA`.
- **Text** is localized like any node text ([10](./10-localization.md)). `heading` shows `label`, `text` shows `description`, `divider` shows no text (a `label` is an optional caption). Markdown or HTML is only available through `ui.textFormat` or an `x-` variant, so the core stays small (D13).
- **Rendering.** Renderer kind `content`, looked up by `ui.variant` like other kinds ([11 §4](./11-rendering-and-adapters.md#4-renderer-registry)); its controller is a `ContentController` ([11 §2](./11-rendering-and-adapters.md#2-headless-field-api)). With no renderer, the built-in fallback prints the text ([11 §4.2](./11-rendering-and-adapters.md#42-missing-renderers)).
- **Version.** A new node kind is a minor change of the contract format ([§7](#7-versioning-and-load-time-validation)); an engine without it rejects the unknown `kind` like any unknown kind. Macros may output content nodes; `fromJsonSchema` doesn't produce them.

## 4. Layout vs. conditions

```ts
/** Hints only. Renderers may ignore them (e.g. native stacks). Grid = 12 columns. */
export interface LayoutHints {
  span?: Partial<Record<'xs' | 'sm' | 'md' | 'lg' | 'xl', number>>;
  order?: number;
}

/** Behavior. The engine evaluates these. Renderers only read the results. */
export interface Conditions {
  visible?: Condition;         // default true
  disabled?: Condition;        // default false; value kept, not editable; submitted per `whenDisabled` (default exclude, 07 §4)
  readOnly?: Condition;        // default false; shown as text
  required?: Condition;        // live facet; the compiler turns it into a 'required' rule with `when` (06 §6)
}
// Groups and arrays also take `readOnly`, `disabled`, `required` (required on a group = at least one
// descendant has a value; on an array = minItems ≥ 1).
```

`disabled`, `readOnly`, and `visible` are inherited: a hidden group hides all its descendants, and a read-only group makes all its descendants read-only. A child can't override an inherited `true` back to `false`. What happens to their values is set by the hidden/disabled policies ([07 §4](./07-effects-and-computed.md#4-hidden-and-disabled-field-policy)).

## 5. Containers

```ts
export const ORPHAN_POLICIES   = ['discard', 'stash'] as const;
export const LOADING_BEHAVIORS = ['keep-stale', 'empty'] as const;

export interface Group<N = Node> extends NodeBase {
  kind: 'group';
  step?: boolean;               // wizard step; the engine reads this (11 §5)
  dataScope?: boolean;          // default false (transparent). See 04 §3
  children: N[];
  validations?: ValidationRule[];   // cross-field; errors attach to the group path
  serializeAs?: string;         // only meaningful with dataScope: true
  ui?: GroupUi;
}

export interface ArrayNode<N = Node> extends NodeBase {
  kind: 'array';
  item: N;                      // usually a Group with dataScope semantics implied; never a Content node (E_SCHEMA)
  defaultValue?: Json[];
  newItem?: Json | Expression;  // template for "add row"
  minItems?: number;
  maxItems?: number;
  reorderable?: boolean;
  identityKey?: string;         // row field used to match row keys across reloading, 04 §4
  validations?: ValidationRule[];
  serializeAs?: string;
  ui?: ArrayUi;
}

export interface DynamicGroup extends NodeBase {
  kind: 'dynamic-group';
  discriminator: PathRef;
  schemaSourceId: string;        // registry id
  dataScope?: boolean;
  orphanPolicy?: typeof ORPHAN_POLICIES[number];        // see 08 §5
  loadingBehavior?: typeof LOADING_BEHAVIORS[number];   // see 08 §5
  schemaCache?: number;                                 // cached sub-schemas per group, LRU; default 10 (08 §5)
  validations?: ValidationRule[];
  serializeAs?: string;                          // only meaningful with dataScope: true
  ui?: GroupUi;
}

/** Renderer-only hints for containers. Mergeable like field ui (§3.4). */
export interface GroupUi {
  variant?: 'section' | 'card' | 'fieldset' | 'tabs' | 'none' | `x-${string}`;
  collapsible?: boolean; defaultCollapsed?: boolean; columns?: number;
}
export interface ArrayUi {
  variant?: 'stack' | 'table' | 'accordion' | 'carousel' | `x-${string}`;
  addLabel?: Text; removeConfirm?: boolean; emptyText?: Text;
}

export type Node = Field | Group<Node> | ArrayNode<Node> | DynamicGroup | Content;   // Content: §3.6   // Field: §3.2
```

The `item` of an array always creates a data scope (a row object), whether or not it is a group. If `item` is a single field, rows are scalars (`string[]`) and the item's path is `.` (the row itself). Its `id` is still required (for error messages, testers, and devtools) but isn't a data key, so it never collides.

### 5.1 Input vs. normalized types

The contract package exports **two** sets of types:

| | Input (`FormContractInput`, `InputNode`) | Normalized (`FormContract`, `Node`) |
|-|------------------------------------------|-------------------------------------|
| Accepted by | `loadContract`, the published JSON Schema, CMS editors | the compiler, engine, `toPayloadSchema` |
| Macro nodes | allowed anywhere a node is (`MacroNode`, 15 §2) | none |
| Short forms | bare `options` array, bare `string` text (15 §6) | expanded |

Input fields, unlike normalized fields, accept option-array shorthands recursively:

```ts
type InputFieldOf<F> = F extends { fieldType: 'choice' | 'multi-choice' }
  ? Omit<F, 'options'> & { options?: OptionsSource | SelectOption[] }
  : F;
export type InputField = InputFieldOf<Field>;   // includes CustomField without global declaration merging
export type InputNode = InputField | Group<InputNode> | ArrayNode<InputNode> | DynamicGroup | Content | MacroNode;
```

`DynamicGroup` has no inline children (its sub-schema comes from a `schemaSource`), so it isn't generic. `FormContractInput` is `Omit<FormContract, 'structure'> & { structure: InputNode[] }`; the published input JSON Schema is generated from these input-specific types. Normalized choice options remain `OptionsSource`-only. A macro in an array's single `item` slot must expand to exactly one non-content node, not a list of siblings ([15 §5](./15-macros.md#5-rules)). `expandContract(input, { macros }) → { contract, pointers, sources, diagnostics }` is the only bridge ([15 §8](./15-macros.md#8-tooling)).

**Custom fields at the wire boundary.** Both published input/core schemas have a distinct `fieldType: x-…` branch matching `CustomField`: shared field keys are checked, additional flat config keys and custom `ui` keys are allowed, and all values must be JSON. This envelope lets a `SchemaSource` return a locally registered `x-signature` without global augmentation; it does not claim that arbitrary config is valid. Validate-core resolves the actual registry entry and checks its value/config/ui schemas (or applies the documented missing-type fallback). Unknown names without `x-` still fail; only the custom branch is open. Statically authored `ContractFor<E>`/`defineContract` derives custom fields from `E`, not this permissive envelope, so ids and registered config remain checked ([21](./21-type-inference.md)). Normalization retains the same custom envelope; typed compiled schemas retain the stronger registry/contract brand.

## 6. Document

```ts
export interface FormContract extends Extensible {
  $schema?: string;               // URL of the published JSON Schema
  schemaVersion: `${number}.${number}`;
  formId: string;
  title?: Text;
  meta?: Record<string, Json>;
  policies?: FieldPolicies;       // form-wide hidden/disabled defaults (07 §4)
  validation?: ValidationConfig;  // form-wide triggers (06 §3)
  load?: LoadConfig;         // 09
  submission?: SubmissionConfig;  // 09
  locales?: Record<LocaleTag, LocaleBundle>;   // optional, rarely needed (10 §3)
  localeSourceId?: string;        // load bundles via a registry instead (10 §3.3)
  structure: Node[];
}
```

## 7. Versioning and load-time validation

- `schemaVersion` is `MAJOR.MINOR`, versioning the **contract format**, independently of package versions. The first stable release of the format is `1.0`. Before that, packages are `0.x` and contracts declare `0.MINOR`; every `0.x` minor may break, and no migrations are provided between them. Major `0` means only "pre-release format"; it is never used for legacy or foreign formats (§7.2).
- From `1.0` on, engines accept any minor version of a supported major version, and breaking changes need a new major plus a migration (§7.2).
- **Unknown keys and newer minors:** an unknown key (not `x-`) is `E_UNKNOWN_PROPERTY` when the contract's minor version is **≤** the engine's. When the contract is **newer**, validate-input reports `W_UNKNOWN_PROPERTY_NEWER_MINOR` (with the key's pointer) and **drops** the key, so validate-core never sees it. So typos are always caught, and older engines can still load contracts that use optional newer features. Macro arguments aren't covered by this rule: each macro's own `args` schema decides ([15 §2](./15-macros.md#2-contract)).
- `packages/contract` ships:
  - a generated **JSON Schema** of `FormContractInput` (from the TS types) for CMS-side validation,
  - a **runtime validator** exposed as a [Standard Schema](https://standardschema.dev) (`StandardSchemaV1<unknown, FormContractInput>`). The engine depends only on the `~standard` interface, so hosts can swap in their own Zod/Valibot/ArkType contract schema through `loadContract(json, { contractSchema })`,
  - the **migration chain** (§7.2), **normalization and macro expansion** (§7.1, [15](./15-macros.md)), and `toPayloadSchema` ([09 §6](./09-loading-and-submission.md#6-json-schema-export-of-the-submitted-data)).

### 7.1 Contract load pipeline

```
loadContract(json, { registry, strict, limits, allow, textFormat, contractSchema }) → Promise<LoadResult>   // { ok, schema?, diagnostics } (18 §1); textFormat checker: 10 §2.1; limits §7.4, allow: AllowList (05 §4)
  1. detect          size check first (string: bytes > limits.contractBytes; object: bounded walk over
                     nodes and nesting, limits.nodes / limits.jsonDepth, stopping at the first excess)
                     → E_LIMIT_EXCEEDED; then read schemaVersion; unsupported major → E_UNSUPPORTED_VERSION
  2. migrate         older major → current, one step at a time (§7.2). Runs on raw JSON. No-op today
  3. validate-input  against the input schema (FormContractInput: macros and short forms allowed). Shape errors: E_SCHEMA.
                     Ids and reserved keys (§1), unknown keys (§7), inline `locales` translations (10 §3.4)
  4. normalize       short forms → full forms (15 §6). Skips the inside of macro nodes
  5. expand          macros → core nodes (15), each macro's args validated by its own schema;
                     then normalize runs again on the macro output
  6. validate-core   against the core schema (FormContract), plus registered fieldType config/ui schemas
  7. compile         paths, scopes, registry ids, DAG, cycles → CompiledSchema
```

Steps 1–6 live in `@formgraph/contract` and need only a `Registry` (for evaluators, field types, and macros, [08 §1](./08-async-data-and-registries.md#1-registries)), not an engine. CMS previews and tests call them directly. Step 7 lives in core. Dynamic sub-schemas ([08 §5](./08-async-data-and-registries.md#5-dynamic-sub-forms)) go through steps 3–7 and are interpreted under the parent contract's `schemaVersion` (including the unknown-key rule above). Attaching one merges its edges into the graph ([04 §5.4](./04-paths-and-dependency-graph.md#54-dynamic-sub-schemas)).

Message syntax checks (`E_INVALID_MESSAGE`) cover contract text/defaults and inline `locales`, using the formatter checker available to loading ([10 §2.1](./10-localization.md#21-formatter)). Translation coverage checks (`W_MISSING_TRANSLATION`, `W_UNUSED_TRANSLATION`) cover inline `locales` only. Bundles from a `localeSourceId` are loaded later and checked on arrival.

**Diagnostic pointers.** `location.pointer` ([17 §1](./17-diagnostics.md#1-shape)) always points into the **input contract after migration** (the output of step 2, as the author wrote it otherwise). Normalization and expansion maintain a pointer map from every output location back to its input location ([15 §8](./15-macros.md#8-tooling)); diagnostics from validate-core and compile are translated through it. Nodes a macro generated point at the macro node and carry `location.source`. Migration steps report each rewrite's original (pre-migration) pointer as `related`.

### 7.2 Migrations

No contract format has been released yet, so there are no migrations today. The mechanism exists so that future breaking changes, and imports from other formats, have one well-defined place to live.

```ts
interface Migration {
  from: number;                        // contract major it accepts
  to: number;                          // always from + 1
  migrate(json: unknown, ctx: { options: Record<string, Json>; report(d: Diagnostic): void }): unknown;
}
registry.migrations.register(m);
```

- `migrate` is a **pure JSON → JSON** function on raw input, before any validation (§7.1 step 2). Steps chain one major at a time (1 → 2 → 3).
- A step reports **all** problems it finds through `report` (code `E_MIGRATION`, with a step-specific `detail`) rather than stopping at the first. Any error stops loading after the step.
- No path from the contract's major to the engine's is `E_MIGRATION_MISSING`.
- Steps **never guess**. Input that can't be translated mechanically (for example a reference that could mean two things) is reported with its pointer. Steps that need information the old contract doesn't contain take it from `options` and fail without it.
- Each step ships with fixture tests: input, expected output, expected diagnostics.
- Hosts can register their own steps for later majors of this format. **Importers** for in-house legacy or foreign formats aren't migrations: they run in host code **before** `loadContract` (like `fromJsonSchema`, §8) and must output a contract with a real `schemaVersion`.

### 7.3 Compile

Compiling checks that ids are unique per data scope (`E_DUPLICATE_DATA_KEY`), every path resolves (`E_UNKNOWN_PATH`, `E_PATH_OUT_OF_SCOPE`), every referenced registry id exists (`E_UNKNOWN_REGISTRY_ID`), the DAG has no cycles (`E_CYCLE`), and ranges are ordered: `minItems ≤ maxItems`, `min ≤ max`, `minSelected ≤ maxSelected` (`E_MIN_MAX`).

Output is a **CompiledSchema**: a flat map `schemaPath → CompiledNode`, with expressions pre-parsed and dependencies resolved. Everything at runtime works on this map, not on the raw tree.

```ts
export const FACETS = ['value', 'visible', 'disabled', 'read-only', 'required', 'validation',
                       'options', 'computed', 'default', 'label', 'message'] as const;   // 04 §5.1
export type Facet = typeof FACETS[number];   // facet values; `readOnly` stays the Conditions key (§4)

interface CompiledRule {
  type: string;                      // validator id
  implied: boolean;                  // from config (§3.3.1)
  param?: CompiledExpr | Json;       // `value`, compiled if it's an Expression
  when?: CompiledExpr;
  rule: ValidationRule;              // the source rule (synthesized for implied rules)
}

interface CompiledNode<N extends Node = Node> {
  schemaPath: SchemaPath;            // e.g. addresses[*].city (04 §1)
  node: Readonly<N>;                 // the normalized node, including config, ui, and x- keys
  kind: N['kind'];
  fieldType?: string;                // fields only
  parent?: SchemaPath;
  children: SchemaPath[];
  dataScope: SchemaPath;             // nearest scope (04 §3)
  facets: Partial<Record<Facet, CompiledExpr>>;   // visible, disabled, read-only, required, default, computed, label, …
  rules: CompiledRule[];             // implied rules (§3.3.1) first, then validations
  policies: Required<FieldPolicies>; // effective whenHidden/whenDisabled/validate* after inheritance (07 §4)
  source?: MacroSource;              // set if a macro generated this node (15 §8)
}
```

Renderers and testers read config and `ui` through `compiled.node` (typed as `FieldOf<FT>` once `fieldType` is narrowed, [11 §4](./11-rendering-and-adapters.md#4-renderer-registry)).

`allow` lists (`AllowList`, `E_NOT_ALLOWED`) are enforced in the phase where each kind is resolved: macros in expand, evaluators and field types in validate-core, the rest in compile ([05 §4](./05-expressions.md#4-security-model)). Compile also enforces load limits (§7.4).

### 7.4 Limits

Contracts are untrusted ([05 §4](./05-expressions.md#4-security-model)), so loading and evaluation are bounded. One `Limits` interface is accepted by both `loadContract(input, { limits })` and `createFormEngine({ limits })`; the engine rechecks a compiled contract against its own limits (below) and applies them to sub-schemas it loads. `createFormEngineFrom` ([18 §2](./18-api-reference.md#2-creating-an-engine)) passes the same limits to `loadContract`. Every field is optional; omitted fields use the default.

```ts
interface Limits {
  contractBytes?: number;      // serialized input size; default 5_000_000
  nodes?: number;              // nodes after expansion; default 10_000
  exprDepth?: number;          // nesting depth of one expression; default 32
  exprSize?: number;           // operator/operand count of one expression; default 1_000
  macroOutputNodes?: number;   // nodes one top-level macro expansion may produce; default 2_000 (15 §5.1)
  regexLength?: number;        // pattern source length; default 1_000 (06 §2.2)
  regexInputLength?: number;   // input length a pattern is tested against; default 10_000 chars (06 §2.2)
  evalSteps?: number;          // steps per evaluation, dev and prod; default 10_000 (05 §4)
  effectDepth?: number;        // effect chain depth; default 8 (04 §6)
  macroCallDepth?: number;     // nested ctx.expand calls; default 8 (15 §5.1)
  tickSteps?: number;          // evaluation steps across one scheduler tick; default 1_000_000 (04 §6)
  jsonDepth?: number;          // nesting depth of object input (loadContract objects, SchemaSource results); default 64
}
```

| Limit | Checked in | On exceed |
|-------|-----------|-----------|
| `contractBytes` | detect (size check) | `E_LIMIT_EXCEEDED`, load stops |
| `jsonDepth`, `nodes` (object input) | detect (size check), before any validator runs | `E_LIMIT_EXCEEDED`, load stops |
| `nodes` | validate-core | `E_LIMIT_EXCEEDED` |
| `exprDepth`, `exprSize` | compile | `E_LIMIT_EXCEEDED` with the expression's pointer |
| `macroOutputNodes` | expand | `E_LIMIT_EXCEEDED` with the macro's pointer |
| `regexLength` | compile (and when a dynamic sub-form attaches) | `E_LIMIT_EXCEEDED` with the pattern's pointer |
| `regexInputLength` | runtime | the rule fails (blocking error), no regex is run |
| `evalSteps` | runtime | `W_EVAL_BUDGET`; display facets fall back, validation and `required` fail closed ([05 §4](./05-expressions.md#4-security-model)) |
| `macroCallDepth` | expand | `E_MACRO_CALL_DEPTH` with the call chain ([15 §5.1](./15-macros.md#51-calling-other-macros)) |
| `tickSteps` | runtime | `W_TICK_BUDGET`; the tick stops, commits stay, unevaluated facets keep their value and are re-evaluated next tick ([04 §6](./04-paths-and-dependency-graph.md#6-scheduler)) |
| `effectDepth` | runtime | `E_EFFECT_DEPTH` ([04 §6](./04-paths-and-dependency-graph.md#6-scheduler)) |

`E_LIMIT_EXCEEDED` has `detail: { limit, max, actual }` naming the limit.

**Contracts compiled elsewhere.** A contract may be compiled with `loadContract` under one policy (at build time, in a shared cache) and run by an engine with another. The `CompiledSchema` records what it contains, not the policy it was compiled under:

```ts
interface CompiledStats { nodes: number; exprDepth: number; exprSize: number; regexLength: number }   // maxima
// CompiledSchema also carries refs: Partial<Record<AllowableRegistry, string[]>>   (05 §4)
```

`createFormEngine` checks `stats` against its own load-time limits and `refs` against its own `allow`, so the stricter side always wins without recompiling (`E_LIMIT_EXCEEDED` / `E_NOT_ALLOWED`, phase `call`, thrown synchronously). `createFormEngine` accepts only a `CompiledSchema`; it never loads raw input. `contractBytes` and `macroOutputNodes` only bound work during loading and aren't rechecked; their result is covered by `nodes`. Runtime limits (`evalSteps`, `effectDepth`, `regexInputLength`) are always the engine's.

Dynamic sub-forms ([08 §5](./08-async-data-and-registries.md#5-dynamic-sub-forms)) always use the engine's `limits`, `allow` and `strict`, and go through the detect size check. Only **attached** sub-schemas count toward `nodes`: switching the discriminator subtracts the old one and adds the new one; cached, detached sub-schemas and stashed values don't count.

## 8. JSON Schema import

Borrowed from JSON Forms ([14](./14-prior-art.md)). This is a quick start for teams that already have JSON Schema or OpenAPI models:

```ts
import { fromJsonSchema } from '@formgraph/contract/json-schema';
const contract = fromJsonSchema(personSchema, { formId: 'person', ui: uiHints });
```

- `string` → `text` (`format: date|date-time|time` → `date-time`, `format: email` → `text` + `email` rule), `number/integer` → `number`, `boolean` → `boolean`, `enum` / `oneOf[const]` → `choice` with static options, `array` of `enum` → `multi-choice`, other `array` → `array`, `object` → `group` with `dataScope: true`.
- `required`, `minimum`, `maxLength`, `pattern`, … → validation rules. `title` / `description` → `label` / `description`. `default` → `defaultValue`.
- `$ref` is resolved, and refs within the same document become shared node definitions. Recursive refs are rejected.
- `if/then/else`, `dependentRequired`, and `dependentSchemas` are translated to `conditions` / `required` where the pattern is recognizable (`const` on a sibling). Anything else is reported in an `unsupported[]` list instead of being silently dropped.
- `ui` hints (optional) can override field types, order, grouping, and variants, playing the role of JSON Forms' UI schema. They're merged by JSON Pointer.

This is a **one-way generator**. The output is a normal contract that authors then edit. We don't keep it in sync with the schema at runtime, because that would make JSON Schema our contract format (see [14 §3](./14-prior-art.md#3-notes-per-library)).
