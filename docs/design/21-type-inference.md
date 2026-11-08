# 21 · Static Type Inference

[← Index](./README.md)

Contracts written in TypeScript get compile-time checking and inferred value types. The types flow **registry → contract → compiled schema → engine**. Contracts that only exist at runtime (JSON from a server or a file) widen to `Json`; that path is the current untyped API, unchanged.

Declaration merging (`declare module '@formgraph/contract'`) is the **last resort**, kept only for registries whose contents can't be seen at the call site (§6).

## 1. Typed registries

`register` mutates, so it can't change a type. Typed registration returns a registry with a **wider type**. Use chaining or one object literal:

```ts
const registry = coreRegistry.extend()                      // Registry<CoreEntries>, new child
  .fieldType('x-signature', {
    value:  v.nullable(v.string()),                            // any Standard Schema
    config: v.object({ minStrokes: v.optional(v.number()) }),
    ui:     v.object({ penColor: v.optional(v.string()) }),
    empty:  null,
  })
  .validator('iban', ibanValidator)
  .severity('x-legal', { extends: 'warning', gate: 'legal' });

// same result, one call
const registry2 = coreRegistry.extend({
  fieldTypes: { 'x-signature': { … } },
  validators: { iban: ibanValidator },
  severities: { 'x-legal': { extends: 'warning', gate: 'legal' } },
});

// Object registration also infers plugin entries, options and members.
const debugRegistry = coreRegistry.extend({ plugins: { debug } });
```

```ts
interface Registry<E extends RegistryEntries = RegistryEntries> { … }

interface RegistryEntries {
  fieldTypes: Record<string, { value: unknown; config: unknown; ui: unknown }>;
  validators: Record<string, { param: unknown }>;
  severities: string;            // union of names
  gates: string;                 // union of gate names (from severities, plus 'warnings')
  macros: Record<string, { params: unknown; output: unknown }>;
  // every other keyed registry (08 §1): Record<string, unknown>, keys only
}
```

- **Types come from the schemas.** `value`, `config`, `ui` and validator `param` types are `StandardSchemaV1.InferOutput<…>` of the registered schemas, so nothing is written twice. A field type's `empty` must be assignable to its `value` type.
- **Same runtime object.** `fieldType(…)` etc. register on the receiver and return it re-typed. The old `registry.fieldTypes.register(…)` stays, untyped (§6). Duplicate ids still throw `E_REGISTRY_DUPLICATE`.
- **Roots and children.** `createRegistry()` / `createRegistry({ … })` create roots. `registry.extend()` / `registry.extend({ … })` create a new child without modifying the receiver; the object accepts the same registration categories as `createRegistry`. Child types retain inherited entries and infer each own registration, including plugin options/members. A child that overrides a parent id must use a schema whose output is assignable to the parent's (otherwise a type error), so contracts typed against the parent stay correct. Chaining registrations on the returned child still mutates that child, not its parent. There is no `parent` creation option or static `createRegistry.extend` method.
- **Plugins.** A plugin declares its entries in `registers`, and `registry.plugin(factory)` returns the registry re-typed with them plus the plugin's options and members. Contracts see plugin-provided types and validators with no extra step. Engine members use `FormEngineWith<R, V, O>`, filtering by per-engine options: literal `false` removes the plugin; a possibly disabled plugin has optional members ([18 §5](./18-api-reference.md#5-built-in-plugin-members)). The default entry adds built-ins before filtering (`asset`, `multi-asset`, `icu` formatters, …).
- **Key-only registries** (providers, sources, transformers, …) contribute their ids so the contract can be checked. Their payload types are in §3.

## 2. Typed contracts

```ts
const contract = defineContract(registry, {
  schemaVersion: '1.0',
  formId: 'profile',
  structure: [
    { kind: 'field', id: 'name', fieldType: 'text', conditions: { required: true } },
    { kind: 'field', id: 'sig', fieldType: 'x-signature', minStrokes: 3 },
    { kind: 'array', id: 'addresses', item: { kind: 'group', id: 'address', children: [
      { kind: 'field', id: 'city', fieldType: 'text' },
    ] } },
  ],
});
```

- `defineContract(registry, contract)` uses `const` type parameters, so no `as const` is needed. It returns `TypedContract<E, C, L>`: the literal input plus phantom registry entries and inference limits. At runtime it returns `contract` unchanged; no metadata property is written. The brand makes the registry's value schemas part of the contract's type, so two registries with the same field-type id but different schemas infer different values. Full signatures are in [18 §1](./18-api-reference.md#1-loading-a-contract).
- **Checked against the registry:** `fieldType`, validator `type`, `severity`, `submission.gates` keys, `macro`, `providerId`, `sourceId`, `schemaSourceId`, `entitySourceId`, `targetId`, transformer ids. A typo is a type error instead of a validate-core diagnostic.
- Each field's flat config keys and `ui` are checked against its type's `config` and `ui`. A `fallback` is checked against its core target type ([01 §3.5](./01-schema-contract.md#35-contract-fallbacks-for-custom-types)). Validator `param` is checked against the validator.
- `schemaVersion` must be the current major. Older contracts go through the runtime path and migrations ([01 §7.2](./01-schema-contract.md#72-migrations)).
- The runtime still runs the full load pipeline. Types never replace validation.

## 3. Inferred engine types

```ts
const res = await loadContract(contract, { registry });   // LoadResult<typeof contract>
if (!res.ok) throw new ContractLoadError(res.diagnostics);
const engine = createFormEngine({ contract: res.schema, registry });
// FormEngineWith<typeof registry, Values, undefined, PayloadOf<typeof contract>>

type Values = ValuesOf<typeof contract>;
// { name: string; sig: string | null; addresses: Array<{ city: string }> }

engine.field('addresses[0].city');      // FieldController<string>
engine.field('sig').value.get();        // string | null (value itself is a Readable)
engine.setValue('name', 42);            // type error
engine.getValues();                     // Values
```

`ValuesOf<C>`:

| Node | Type |
|------|------|
| field | its field type's `value` |
| group | object of its children (layout-only groups are flattened, as in 03) |
| array | `Array<item object>` |
| array with a scalar field item | `Array<field value>` |
| content | no value or key |
| `whenHidden: 'exclude'` or optional by condition | key stays required in `getValues()` (resolved values always exist); the **payload** type marks it optional |
| dynamic group | runtime `schemaSourceId` output widens to `Json`; there is no inline `schemas` contract key. A scoped group widens its own subtree; a transparent group widens the containing object whose keys it can add |
| macro node | its macro's declared `output` node type; `Json` when the macro declares none (declaring is optional) |

`PathsOf<Values>` gives `DataPath` literals: `'name'`, `'addresses'`, `` `addresses[${number}]` ``, `` `addresses[${number}].city` ``.

`ValuesOf<C>` and `PayloadOf<C>` first extract the registry entries, literal input and caps from `TypedContract<E, Input, L>`, then walk the input using those entries. An unbranded/unknown input yields `Json`. `CompiledSchema<C>` retains that same phantom contract type through loading; it never reconstructs types from runtime registry ids. Inference limits remain attached to the brand through both stages.

- **Typed overloads, untyped fallback.** `field`, `setValue`, `array`, `revert`, `touch` take `Paths` with the value type looked up from `Values`. A string that isn't a known path literal (relative `./` refs, `/` PathRefs, computed strings) falls back to `FieldController<Json>`.
- **Payload.** Without an outbound transformer, `serialize()` and the submit target receive `PayloadOf<C>`: values after `serializeAs` renames and hidden/disabled policies. With a transformer, the transformer's output type. `SubmitTarget` and `OutboundTransformer` are generic over it ([08 §1.1](./08-async-data-and-registries.md#11-shared-function-shapes)).
- **Renderers** get `FieldController<V>` and the field type's `ui`, as today.

## 4. Runtime contracts widen

```ts
const res = await loadContract(jsonFromServer, { registry });   // LoadResult<unknown>
if (!res.ok) throw new ContractLoadError(res.diagnostics);
createFormEngine({ contract: res.schema, registry });          // FormEngine<Json, string>
// Hosts may assert a value shape explicitly at their boundary; it is not inferred or validated by a type argument.
```

`FormEngine` with no type arguments is `FormEngine<Json, string>`, the same API as before this doc.

## 5. Limits

- **JSON files.** TypeScript widens imported JSON, so a `.json` contract loses its literals. `formgraph typegen contract.json` emits a `.ts` module with `defineContract` (M3 tooling).
- **Expressions aren't typed.** Expression strings and the `PathRef`s inside them are checked at load, not by TypeScript. `ref`/`dependsOn` paths could be checked with template literal types, but it's costly; not planned.
- **Compiler cost.** Inference is capped: past 500 nodes or 8 levels of nesting, the affected subtree widens to `Json` instead of hitting instantiation-depth errors. A type-performance benchmark (`tsc --extendedDiagnostics` on a 500-node fixture) runs in CI with a budget. The caps are configurable (§5.1).

### 5.1 Configuring the caps

```ts
// global defaults (configuration, not registry contents, so a module interface is fine here)
declare module '@formgraph/contract' {
  interface TypeConfig { maxDepth: 12; maxNodes: 1000 }
}

// per contract; type-only, ignored at runtime; wins over TypeConfig
defineContract(registry, contract, { inference: { maxDepth: 12, maxNodes: 1000 } });
```

- Order: per-contract option, then `TypeConfig`, then the built-in defaults (8 / 500).
- `maxDepth` is hard-capped at 16: a larger literal is a type error (TypeScript's own recursion limit is about 50, and `PathsOf` slows sharply before that).
- `maxNodes` has no hard cap; raising it trades compile time, which the CI benchmark tracks.
- **Mutable or global registries** whose entries aren't visible where the contract is defined can't be inferred. Use §6.

## 6. Fallback: declaration merging

For registries assembled at runtime (plugins chosen by config, a shared global registry), or for adding `ui` keys to core types from a renderer package, the module interfaces remain:

```ts
declare module '@formgraph/contract' {
  interface FieldTypeRegistry { 'x-signature': { value: string | null; config: { minStrokes?: number }; ui: { penColor?: string } } }
  interface ChoiceUi { chipColor?: 'primary' | 'neutral' }
}
```

Merged entries are included in every `Registry<E>` type. Prefer typed registration; use this only when the registry's type can't flow to the contract.

For typed access to those merged entries, use the strong `KnownField`/`FieldOf` view, not the open wire `Field` union ([01 §3.2](./01-schema-contract.md#32-field-type-registry)):

```ts
function describeMerged(field: KnownField) {   // registry/config-validated known field, not arbitrary JSON
  if (field.fieldType === 'x-signature') {
    const minStrokes: number | undefined = field.minStrokes;
    const color: string | undefined = field.ui?.penColor;
  }
}
// Locally registered entries use FieldFor<E>, with E carried by Registry<E>/TypedContract.
```

The open `CustomField` branch remains necessary at input/schema-source boundaries. It does not weaken these separate strong views, and a name check on raw wire data is not a substitute for validation.
