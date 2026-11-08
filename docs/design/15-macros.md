# 15 · Macros

[← Index](./README.md)

## 1. Purpose

Macros keep the **core contract small** while letting authors write common patterns briefly. A macro is a node that is **expanded at load time into core nodes** before compiling. The engine at runtime never sees a macro.

Use a macro when a pattern is already expressible with core primitives, but is verbose or easy to get wrong by hand ([16 · Cookbook](./16-cookbook.md) has the long forms). If something *isn't* expressible with core primitives, it needs a core change, not a macro.

## 2. Contract

A macro node has a `macro` key instead of `kind`. An unregistered `macro` id is `E_UNKNOWN_MACRO` (a macro excluded by `allow` is `E_NOT_ALLOWED`, checked in expand, [05 §4](./05-expressions.md#4-security-model)):

```json
{ "macro": "choice-with-other", "id": "source", "label": "How did you hear about us?",
  "options": [{ "value": "search", "label": "Search" }, { "value": "friend", "label": "Friend" }],
  "otherLabel": "Please specify" }
```

```ts
export interface MacroNode {
  macro: string;                  // registry id
  id: string;
  [arg: string]: Json;            // macro-specific arguments
}
// MacroNode is part of InputNode only; expanded contracts never contain it (01 §5.1)
export type InputNode = Field | Group<InputNode> | ArrayNode<InputNode> | DynamicGroup | Content | MacroNode;
```

- **Macro ids** are kebab-case with an optional `@MAJOR` suffix (`choice-with-other@2`, rule 6), the one exemption from D21 ([01 §2.1](./01-schema-contract.md#21-enumerated-values)). Only `@formgraph/macros` registers unprefixed ids; every other macro id must start with `x-`, like custom field types (`E_MACRO_PREFIX`, thrown by `register`).
- A macro node's own `id` follows the node id grammar ([01 §1](./01-schema-contract.md#1-principles)). Every other key is an argument, checked only by the macro's `args` schema: the contract's unknown-key rules ([01 §7](./01-schema-contract.md#7-versioning-and-load-time-validation)) and normalization (§6) don't look inside macro nodes.

## 3. Definition

```ts
interface Macro {
  id: string;
  args: StandardSchemaV1;         // validates the macro node's arguments (E_MACRO_ARGS)
  expand(node: MacroNode, ctx: MacroContext): Node | Node[];
}

interface MacroContext {
  pointer: string;                // JSON Pointer of the macro node in the (migrated) input
  uniqueId(hint: string): string; // deterministic: `${node.id}__${hint}` (§5 rule 3)
  expand(node: MacroNode): Node[]; // call another registered macro (§5.1); returns core nodes
}

registry.macros.register(choiceWithOther);   // the shared registry (08 §1); no engine needed
```

There is no access to the rest of the contract: a macro sees only its own node, so its output depends on nothing else.

## 4. Pipeline position

See the full load pipeline in [01 §7.1](./01-schema-contract.md#71-contract-load-pipeline):

```
detect → migrate → validate input → normalize → expand macros (+ normalize output) → validate core → compile
```

- Expansion runs **after** migration and normalization, so macros only ever see the current contract version. Normalization skips the inside of macro nodes, so a macro's **arguments** arrive as authored (its `args` schema decides which short forms it accepts).
- Normalization **runs again on each macro's output** before validate-core, so `expand` may return short forms (bare strings for `Text`, bare `options` arrays).
- The expanded contract is validated again against the core contract schema. A macro can't produce invalid core nodes without an error.

## 5. Rules

1. **Core output only.** `expand` returns core nodes. Output that contains `macro` nodes is an error (`E_MACRO_NESTED`). Expansion is one pass, with no fixpoint. Macros reuse other macros only by calling them explicitly (§5.1).
2. **Pure and synchronous.** No I/O, no randomness, no clock, no access to `context` or to the rest of the contract. The same input always gives the same output, so expansions are cached by `hash(macro node, macro id including @MAJOR, macro package version)`.
3. **Scoped ids.** Every node an expansion returns needs an id that is unique in its data scope. Ids can come from arguments (like `hierarchy`'s `levels[].id`) or be derived with `ctx.uniqueId(hint)`, which is deterministic: `${node.id}__${hint}` from the macro node's own `id` (for example `source__other`). Collisions **within one expansion** (including `ctx.expand` calls, and the same hint used twice) are `E_MACRO_ID_COLLISION`. Collisions with nodes outside the expansion are caught by the compiler as `E_DUPLICATE_DATA_KEY`, whose `location.source` names the macro. When no output node reuses the macro's own `id`, that id is only used in error messages, origins, and `uniqueId`.
4. **Errors point to the source.** Expansion records where each generated node came from in a **side map** (§8), never on the node itself, so expanded contracts stay valid, clean JSON. Diagnostics on those nodes use it to refer to the macro ("in macro `choice-with-other` at /structure/3"), not to generated ids ([17 §1](./17-diagnostics.md#1-shape)).
5. **Data shape is part of the macro's contract.** A macro's documentation must state the data paths it produces, because they appear in submitted data and in `toPayloadSchema` ([09 §6](./09-loading-and-submission.md#6-json-schema-export-of-the-submitted-data)). Changing them is a breaking change to the macro.
6. **Versioned ids.** Breaking changes ship as a new id with an `@MAJOR` suffix (`choice-with-other@2`). Old ids keep working.

### 5.1 Calling other macros

A macro can build on another one by calling `ctx.expand`:

```ts
const address: Macro = {
  id: 'address',
  args: AddressArgs,
  expand(node, ctx) {
    const location = ctx.expand({ macro: 'hierarchy', id: ctx.uniqueId('location'), levels: …, tree: node.regions });
    const street   = { kind: 'field', id: 'street',   fieldType: 'text', label: 'Street' };
    const postcode = { kind: 'field', id: 'postcode', fieldType: 'text', label: 'Postcode' };
    return [{ kind: 'group', id: node.id, dataScope: true, children: [street, ...location, postcode] }];
  },
};
```

- `ctx.expand` validates the called macro's `args`, runs it synchronously, and returns **core nodes**. The caller can place, wrap, or adjust them (for example `{ ...n, label }`). Its output is still checked by rule 1.
- Rule 3 applies to the **combined** output: ids from the called macro (here, the `levels[].id`s) and the caller's own ids must not collide in their data scope (`E_MACRO_ID_COLLISION`).
- **Depth limit:** calls nest at most `limits.macroCallDepth` deep, default 8 ([01 §7.4](./01-schema-contract.md#74-limits); `E_MACRO_CALL_DEPTH`, reporting the whole chain). Recursion, including a macro calling itself, is allowed within that limit, so expansion always terminates.
- **Output limit:** one top-level expansion (including its `ctx.expand` calls) may produce at most `limits.macroOutputNodes` nodes (default 2,000, [01 §7.4](./01-schema-contract.md#74-limits)); more is `E_LIMIT_EXCEEDED` at the macro's pointer. Nodes are counted as they are produced (each `ctx.expand` result and the re-normalization of the output), so expansion stops at the limit instead of after building everything.
- **Macros bound their own work.** Arguments come from the untrusted contract; the engine can't meter a macro's own code. A macro must keep its work proportional to its arguments and cap them in its argument schema (for example `maxItems`). The built-in macros (§7) do. Total input is already bounded by `contractBytes` and the detect size check ([01 §7.4](./01-schema-contract.md#74-limits)).
- The called macro gets the same `pointer`, and `uniqueId` derives from the called node's `id` (here `address__location__…`). The purity rules (rule 2) apply to the whole call tree. Caching applies to the top-level macro node.
- **Sources:** nodes returned by `ctx.expand` carry their source chain under a Symbol key while expansion runs. Symbol keys are invisible to `JSON.stringify`, `Object.keys`, and validation, and survive object spread. After the top-level macro returns, the expander computes each node's final JSON Pointer, writes the side map (§8), and strips the tags. A node the caller rebuilt from scratch (new object, `structuredClone`, JSON round-trip) loses the inner hop and gets the caller's source.

## 6. Built-in normalizations vs. macros

Some value-level shorthands exist already. They are **normalizations**, not macros: they rewrite a value in place rather than expanding a node. They run in step 4, skip the inside of macro nodes, and run again on macro output (§4).

| Short form | Normalized to |
|------------|---------------|
| Bare string | `{ default: string }` ([10](./10-localization.md)) |
| Bare `options` array of `SelectOption` | `{ source: 'static', items }` ([13 §2](./13-option-sources-and-cascading.md#2-contract)) |

`conditions.required` is **not** a normalization: it stays a live `Conditions` facet in the core contract, and the compiler turns it into a `required` rule with `when` ([06 §6](./06-validation.md#6-required-and-conditions)).

Normalizations are fixed and built in. Macros live in a registry and can be extended. Expressions have **no** short form ([05 §1](./05-expressions.md#1-shape)). Arrays of bare strings as options aren't a normalization either; a macro may accept them as its own argument format.

## 7. Standard library (`@formgraph/macros`, optional)

| Macro | Expands to | Long form |
|-------|-----------|-----------|
| `choice-with-other` | `choice` + text field visible/required when `other` is selected | [16 §3](./16-cookbook.md#3-other-option-with-free-text) |
| `multi-choice-with-none` | `multi-choice` + exclusive-`none` effect | [16 §4](./16-cookbook.md#4-exclusive-none-option) |
| `carry-forward` | `choice`/`multi-choice` with options filtered by another field's selection | [16 §5](./16-cookbook.md#5-carry-forward-options) |
| `variable` | Hidden computed field (`visible: false`, `whenHidden: 'exclude'`) | [16 §2](./16-cookbook.md#2-form-level-variables) |
| `yes-no` | `choice` with static `true` / `false` options and `ui.variant: 'radio'` | [16 §25](./16-cookbook.md#25-a-yesno-question-that-can-be-unanswered) |
| `list` | `array` whose `item` is one scalar field | [16 §11](./16-cookbook.md#11-multiple-values-for-one-field) |
| `reset-when` | Effects on the referenced fields that clear or reset this field | [16 §6](./16-cookbook.md#6-reset-a-field-when-another-changes) |
| `hierarchy` | One `choice` per level with `map` sources keyed by the ancestor path (§7.1) | [13 §2.2](./13-option-sources-and-cascading.md#22-map) |

### 7.1 `hierarchy`

Writes a static tree once and expands it into cascading `choice` fields (Q10).

```json
{ "macro": "hierarchy", "id": "location",
  "levels": [
    { "id": "country", "label": "Country" },
    { "id": "state",   "label": "State" },
    { "id": "city",    "label": "City", "ui": { "variant": "combobox" } } ],
  "tree": [
    { "value": "US", "label": "United States", "children": [
      { "value": "IL", "label": "Illinois", "children": [ { "value": "springfield", "label": "Springfield" } ] },
      { "value": "MO", "label": "Missouri", "children": [ { "value": "springfield", "label": "Springfield" } ] },
      { "value": "WA", "label": "Washington", "children": [ … ] } ] },
    { "value": "AU", "label": "Australia", "children": [
      { "value": "WA", "label": "Western Australia", "children": [ … ] } ] } ],
  "cascade": { "onChange": "clear" } }
```

Expands to:

```json
{ "kind": "field", "id": "country", "fieldType": "choice", "label": "Country",
  "options": { "source": "static", "items": [ { "value": "US", "label": "United States" }, { "value": "AU", "label": "Australia" } ] } },
{ "kind": "field", "id": "state", "fieldType": "choice", "label": "State",
  "options": { "source": "map", "key": "./country", "map": {
    "US": [ { "value": "IL", "label": "Illinois" }, { "value": "MO", "label": "Missouri" }, { "value": "WA", "label": "Washington" } ],
    "AU": [ { "value": "WA", "label": "Western Australia" } ] } },
  "cascade": { "onChange": "clear" } },
{ "kind": "field", "id": "city", "fieldType": "choice", "label": "City",
  "options": { "source": "map",
    "key": { "lang": "json-logic", "expr": { "cat": [ { "var": "./country" }, "/", { "var": "./state" } ] } },
    "map": { "US/IL": [ … ], "US/MO": [ … ], "US/WA": [ … ], "AU/WA": [ … ] } },
  "cascade": { "onChange": "clear" }, "ui": { "variant": "combobox" } }
```

- **Data shape:** one plain value per level, as siblings named by `levels[].id`: `{ "country": "US", "state": "MO", "city": "springfield" }`. Wrap the macro in a group for nesting. No transformer is needed for loading or submission.
- **Compound keys are lookup-only.** A level's `map` key joins all ancestor values, so values only need to be unique among siblings. Repeated names (`springfield`, `WA`) are fine. Duplicate sibling values are `E_MACRO_DUPLICATE_VALUE`.
- **Multi-select levels:** if a level's generated field is a `multi-choice` (via `levels[].fieldType`), every deeper level is keyed by the plain `PathRef` of its **immediate** parent instead of the compound expression, because a compound key can't be built from several selected values ([13 §2.2.1](./13-option-sources-and-cascading.md#221-multi-select-parents)). So values in each of those levels must be unique across the whole level, not just among siblings (`E_MACRO_DUPLICATE_VALUE`). The macro also emits `groupBy: 'parent'` for the level right below a multi-select unless `levels[].groupBy` is `false`.
- **`keySeparator`** defaults to `/`. A value containing the separator is `E_MACRO_SEPARATOR`.
- **`levels[]`** accepts any field keys (`label`, `ui`, `validations`, `conditions`, `autoSelectSingle`, …), copied onto the generated field. The top-level `cascade` applies to every level but the first; a level's own `cascade` overrides it.
- **`levels.length`** must be at least the tree depth (`E_MACRO_TREE_DEPTH` otherwise). Leaves shallower than that are allowed: deeper levels get no options (`whenEmpty` applies).
- **Size:** work and output are linear in the tree; the argument schema caps `tree` at 5,000 nodes and `levels` at 10 (§5.1). For large or frequently changing hierarchies, use a `provider` source with the same three fields instead.

Includes (reusable sub-contracts) are **not** a macro, because they need I/O. They're put together before loading, by the CMS or publishing pipeline ([16 §12](./16-cookbook.md#12-reusable-sub-contracts)).

## 8. Tooling

- `expandContract(input, { macros })` is exported from `@formgraph/contract` for CMS previews, tests, and snapshot diffs. It runs normalization and expansion (pipeline steps 4–5, including normalization of macro output) and needs no engine:

```ts
interface ExpandResult {
  contract: FormContract;                       // core nodes only
  pointers: Map<string, string>;                // every location in `contract` → its pointer in the (migrated) input
  sources: Map<string, MacroSource>;            // key: JSON Pointer of a generated node in `contract`
  diagnostics: Diagnostic[];
}
interface MacroSource {
  pointer: string;                              // JSON Pointer of the macro node the author wrote (always in the input)
  chain: string[];                              // macro ids, outermost first: ['address', 'hierarchy']
}
```

- **Pointer rule:** `Diagnostic.location.pointer` always points into the migrated input contract ([01 §7.1](./01-schema-contract.md#71-contract-load-pipeline)). Validate-core and compile diagnostics are translated through `pointers`. Hand-written nodes map to their own (possibly shifted) input location; generated nodes map to their macro node's pointer and also get `location.source`.
- Only nodes a macro generated have a `sources` entry; hand-written nodes don't. The compiler copies it onto `CompiledNode.source` (internal, not part of the contract).
- Devtools show the original and expanded contract side by side, linked through `sources`.
- `toPayloadSchema` and `fromJsonSchema` work on the **expanded** contract. `fromJsonSchema` never emits macros.
