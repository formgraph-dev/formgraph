# 20 · Static Type Inference

[← Index](./README.md)

Contracts written in TypeScript get compile-time checking and inferred value types. The types flow **registry → contract → compiled schema → engine**. Contracts that only exist at runtime (JSON from a server or a file) widen to `Json`; that path is the current untyped API, unchanged.

Declaration merging (`declare module '@formgraph/contract'`) is the **last resort**, kept only for registries whose contents can't be seen at the call site (§6).

## 1. Typed registries

`register` mutates, so it can't change a type. Typed registration returns a registry with a **wider type**. Use chaining or one object literal:

```ts
const registry = createRegistry({ parent: coreRegistry })     // Registry<CoreEntries>
  .fieldType('x-signature', {
    value:  v.nullable(v.string()),                            // any Standard Schema
    config: v.object({ minStrokes: v.optional(v.number()) }),
    ui:     v.object({ penColor: v.optional(v.string()) }),
    empty:  null,
  })
  .validator('iban', ibanValidator)
  .severity('x-legal', { extends: 'warning', gate: 'legal' });

// same result, one call
const registry2 = createRegistry({
  parent: coreRegistry,
  fieldTypes: { 'x-signature': { … } },
  validators: { iban: ibanValidator },
  severities: { 'x-legal': { extends: 'warning', gate: 'legal' } },
});
```

```ts
interface Registry<E extends RegistryEntries = RegistryEntries> { … }

interface RegistryEntries {
  fieldTypes: Record<string, { value: unknown; config: unknown; ui: unknown }>;
  validators: Record<string, { param: unknown }>;
  severities: string;            // union of names
  gates: string;                 // union of gate names (from severities, plus 'warnings')
  macros: Record<string, { params: unknown; output: unknown }>;
  // every other keyed registry (07 §1): Record<string, unknown>, keys only
}
```

- **Types come from the schemas.** `value`, `config`, `ui` and validator `param` types are `StandardSchemaV1.InferOutput<…>` of the registered schemas, so nothing is written twice. A field type's `empty` must be assignable to its `value` type.
- **Same runtime object.** `fieldType(…)` etc. register on the receiver and return it re-typed. The old `registry.fieldTypes.register(…)` stays, untyped (§6). Duplicate ids still throw `E_REGISTRY_DUPLICATE`.
- **Children.** `createRegistry({ parent })` has type `Parent & Own`. A child that overrides a parent id must use a schema whose output is assignable to the parent's (otherwise a type error), so contracts typed against the parent stay correct.
- **Plugins.** A plugin declares its registry entries as a type (`Plugin<Members, Entries>`). `FormEngineWith<P>` already carries plugin members; the engine's registry type also gains `Entries`. The default entry's registry type includes the built-in plugins (`asset`, `multi-asset`, `icu` formatters, …).
- **Key-only registries** (providers, sources, transformers, …) contribute their ids so the contract can be checked. Their payload types are in §3.

## 2. Typed contracts

```ts
const contract = defineContract(registry, {
  schemaVersion: '1.0',
  nodes: [
    { kind: 'field', id: 'name', fieldType: 'text', required: true },
    { kind: 'field', id: 'sig', fieldType: 'x-signature', minStrokes: 3 },
    { kind: 'array', id: 'addresses', item: [{ kind: 'field', id: 'city', fieldType: 'text' }] },
  ],
});
```

- `defineContract<const C>(registry, contract)` uses a `const` type parameter, so no `as const` is needed. At runtime it returns `contract` unchanged (zero bytes beyond the call).
- **Checked against the registry:** `fieldType`, validator `type`, `severity`, `submission.gates` keys, `macro`, `providerId`, `sourceId`, `schemaSourceId`, `entitySourceId`, `targetId`, transformer ids. A typo is a type error instead of a validate-core diagnostic.
- Each field's flat config keys and `ui` are checked against its type's `config` and `ui`. A `fallback` is checked against its core target type ([01 §3.5](./01-schema-contract.md#35-contract-fallbacks-for-custom-types)). Validator `param` is checked against the validator.
- `schemaVersion` must be the current major. Older contracts go through the runtime path and migrations ([01 §7.2](./01-schema-contract.md#72-migrations)).
- The runtime still runs the full load pipeline. Types never replace validation.

## 3. Inferred engine types

```ts
const res = await loadContract(contract, { registry });   // LoadResult<typeof contract>
const engine = createFormEngine({ contract: res.schema, registry });
// FormEngine<Values, Paths>

type Values = ValuesOf<typeof contract>;
// { name: string; sig: string | null; addresses: Array<{ city: string }> }

engine.field('addresses[0].city');      // FieldController<string>
engine.field('sig').value;              // string | null
engine.setValue('name', 42);            // type error
engine.getValues();                     // Values
```

`ValuesOf<C>`:

| Node | Type |
|------|------|
| field | its field type's `value` |
| group | object of its children (layout-only groups are flattened, as in 02) |
| array | `Array<item object>` |
| `hidden: 'exclude'` or optional by condition | key stays required in `getValues()` (resolved values always exist); the **payload** type marks it optional |
| dynamic group | union over its inline `schemas` with the discriminator literal; `Json` when schemas come from a `schemaSource` |
| macro node | its macro's declared `output` node type; `Json` when the macro declares none (declaring is optional) |

`PathsOf<Values>` gives `DataPath` literals: `'name'`, `'addresses'`, `` `addresses[${number}]` ``, `` `addresses[${number}].city` ``.

- **Typed overloads, untyped fallback.** `field`, `setValue`, `array`, `revert`, `touch` take `Paths` with the value type looked up from `Values`. A string that isn't a known path literal (relative `./` refs, `/` PathRefs, computed strings) falls back to `FieldController<Json>`.
- **Payload.** Without an outbound transformer, `serialize()` and the submit target receive `PayloadOf<C>`: values after `serializeAs` renames and hidden/disabled policies. With a transformer, the transformer's output type. `SubmitTarget` and `OutboundTransformer` are generic over it ([07 §1.1](./07-async-data-and-registries.md#11-shared-function-shapes)).
- **Renderers** get `FieldController<V>` and the field type's `ui`, as today.

## 4. Runtime contracts widen

```ts
const res = await loadContract(jsonFromServer, { registry });   // LoadResult<unknown>
createFormEngine({ contract: res.schema, registry });          // FormEngine<Json, string>
createFormEngine<MyValues>({ … });                             // host asserts a shape; no check
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
