# 04 · Paths & Dependency Graph

[← Index](./README.md) · Addresses review items P1, P2, P3, P4

## 1. Three kinds of path

| Kind | Example | Used for |
|------|---------|----------|
| **Schema path** | `addresses[*].city` | Compiled node lookup and DAG edges (wildcard per array level) |
| **Data path** | `addresses[2].city` | Reading and writing values, serialization, external errors |
| **Row key path** | `addresses[#k7].city` | Meta, errors, UI keys (stable across reorder) |

The engine converts between them using `meta.rows`.

```ts
type SchemaPath = string & { __brand: 'SchemaPath' };   // addresses[*].city
type DataPath   = string & { __brand: 'DataPath' };     // addresses[2].city
type RowKeyPath = string & { __brand: 'RowKeyPath' };   // addresses[#k7].city
type RowKey     = `#k${number}`;
```

All three use the same notation: dot-separated keys, no leading `/`, and one bracket segment per array level (`[*]`, `[2]`, or `[#k7]`). The root is the empty string. Contract references (`PathRef`, §2) are a different, author-facing syntax that the compiler resolves into these.

## 2. Path grammar

References in the contract (`discriminator`, `dependsOn`, expression `var`s, effect `target`, `include`, `equals`, …) **must be explicit**. Each one starts with `/`, `./`, `../`, or a `$` root:

| Syntax | Meaning |
|--------|---------|
| `/email` | Absolute, from the root data scope |
| `./city` | Field in the current data scope (the current row, for arrays) |
| `../country` | Field in the parent data scope. Can be repeated: `../../x` |
| `.` | The current data scope itself (the row value, for an array of scalars) |
| `./items[*].qty` · `/items[*].qty` | All rows of an array (aggregations, read-only) |
| `$row` | The current array row object (alias of `.` inside a row) |
| `$index` | The current row index (read-only, expression scope only) |
| `$context.role` | Host context ([05 §3](./05-expressions.md#3-evaluation-scope)) |
| `$option` · `$selected` | The option being tested by an options `filter` or `disableWhen`, and the option just picked in `onSelect` ([13](./13-option-sources-and-cascading.md)). Only available in those expressions |

Rules:
- **A bare name (`email`, `addresses.city`) is a compile error** (`E_IMPLICIT_PATH`). There's no lexical lookup and no `strict: false` escape hatch, so adding a field later can't silently change what an existing expression refers to.
- "Scope" always means **data scope**, not layout. A field in a transparent `group` ([§3](#3-data-keys-and-scoping)) uses `./` to refer to fields in the nearest enclosing scope that has `dataScope` (or a row, or the root), whichever section they are laid out in.
- `../` past the root is `E_PATH_OUT_OF_SCOPE`. A path that doesn't resolve to a node is `E_UNKNOWN_PATH`. Both are reported when the schema is compiled, with the referencing node's location.
- A row can only be addressed directly from inside that row, through `./`. From outside, use `[*]` (aggregation). Fixed indexes like `/items[0].qty` aren't allowed in contracts (`E_FIXED_INDEX`), because indexes move when rows are reordered.
- `$option` and `$selected` outside the expressions listed above are `E_ROOT_UNAVAILABLE`.
- **Read-only references:** `$index`, `$context…`, `$option`, `$selected`, and any path containing `[*]` can't be write targets (effect `target`, `onSelect` fill, `reset-when`). Doing so is `E_READ_ONLY_PATH`.

All three are compile errors reported with the referencing node's location.

Internally, the engine uses absolute data paths and row key paths ([§1](#1-three-kinds-of-path)) without the leading `/`. The explicit grammar applies only to references written in contracts.

## 3. Data keys and scoping

- A field's **data key** is its `id`. `serializeAs` only affects the outbound/inbound mapping ([09](./09-loading-and-submission.md)).
- `group` is **transparent** by default: its children write into the parent scope. This suits sections and wizard steps, which are layout, not data.
- `group` with `dataScope: true` nests its children: `{ [group.id]: { ...children } }`.
- `array` always nests, and each row is a scope.
- `dynamic-group` follows `dataScope` the same way a group does.
- `content` nodes ([01 §3.6](./01-schema-contract.md#36-content-nodes)) have no data key or path: they write nothing and can't be referenced.
- The compiler reports `E_DUPLICATE_DATA_KEY` when two fields resolve to the same data path.

## 4. Array row identity

- Each row gets a `RowKey` (a short monotonic id, `#k1`, `#k2`, …) when it enters state (from defaults, loading, insert, or replace).
- `meta.rows[arrayDataPath]: RowKey[]` runs parallel to the array value.
- `meta.baselineRows[arrayRowKeyPath]: BaselineRowKeys` independently records baseline order, unresolved-old-row provenance and nested row associations ([03 §3.2](./03-state-model.md#32-atomic-array-writes)). It does not follow current reorders/inserts and is persisted in drafts/resume state.
- `move`/`remove` update both the value array (in `current`) and the key list together in one batch.
- Errors and meta are keyed by row key path, so reorders don't move errors to the wrong row.
- Adapters use the `RowKey` as the framework list key.
- When a whole array value is replaced (for example by reloading), keys are reassigned unless the transformer provides an identity field (`array.identityKey`, such as `"id"`). With an identity field, keys are matched and kept.

## 5. Dependency graph

### 5.1 Nodes and edges

Graph nodes are **schema paths** combined with a *facet* (`FACETS`, [01 §7.3](./01-schema-contract.md#73-compile)): `value`, `visible`, `disabled`, `read-only`, `required`, `validation`, `options`, `computed`, `default` (expression defaults, [07 §3.1](./07-effects-and-computed.md#31-expression-defaults-replaces-mode-initial)), `label` / `message` (when text params use expressions). The `read-only` facet comes from the `conditions.readOnly` key. An options source's `filter`, `disableWhen`, map `key`, and `dependsOn` all feed the `options` facet.

An edge `A.value → B.facet` exists when B's facet expression reads A. Edges come from:
1. **Inference:** the compiler walks parsed expressions and collects `var` references (each evaluator plugin must implement `collectDeps(expr)`).
2. **Declared:** `dependsOn` arrays, added on top of inferred edges for things inference can't see, such as `fn` calls.
3. **Structural:** a parent's `visible`/`disabled` → its descendants' effective facets.

### 5.2 Cycle detection

At compile time, Tarjan's SCC runs over the read edges (§5.1) plus every **value-writing** edge. A writer W that writes field T adds the edge W's inputs → `T.value`:

| Writer | Edge |
|--------|------|
| `computed` on T | each read → `T.value` |
| Effect `set` / `clear` / `reset` on target T | the effect's trigger field and `when`/`value` reads → `T.value` |
| `whenHidden: 'clear'` on T | reads of T's effective `visible` → `T.value` |
| Cascade `onChange: 'clear'` / `'reset'` on T | T's options dependencies → `T.value` |
| `autoSelectSingle` on T | `T.options` → `T.value` |
| `onSelect` reverse fill into T | user-selection event → `T.value`; event-only writer, checked with its protected batch below |
| `reset-when` macro output | already plain effects; listed for completeness |

Any cycle is `E_CYCLE` and lists the path with each edge's kind. Effects are checked conservatively: an effect from A targeting B counts as an edge A → B.value even if its `when` is rarely true. **Exception:** an effect that targets its own field (A → A) isn't a static cycle error, because it is meant to converge (for example normalizing a selection). It is limited by `limits.effectDepth` at runtime ([§6](#6-scheduler)), and repeated `E_EFFECT_DEPTH` hits are reported in dev.

**Reverse-fill exception, not a general cycle escape.** `onSelect` runs only for an explicit user selection of its field, never for writes from cascade invalidation, auto-select, effects, loading or replay. Before accepting an apparent child→ancestor→child cycle, the compiler checks that every reverse edge is an `onSelect` write to a transitive options dependency (`E_ON_SELECT_TARGET` otherwise), and that every return value-writing edge is a built-in cascade invalidation targeting the selecting child or another target protected by that same batch. Removing those event-only reverse edges must leave an acyclic propagation graph; any return path through an ordinary effect, computed/default writer, auto-select or another field's selection batch is still `E_CYCLE`. This certificate permits only the batch-backed reverse-fill loop, not unrelated cycles in the same component. The scheduler topologically orders the acyclic propagation graph; the user-selection batch runs before that propagation, with the certified invalidations suppressed only for its protected targets ([13 §7](./13-option-sources-and-cascading.md#7-reverse-fill)). Ordinary parent edits still invalidate normally.

### 5.3 Wildcards

An edge from `items[*].qty` to `total.computed` means a change in any row (or adding or removing a row) marks `total` dirty. An edge inside one row (`./qty → ./subtotal`) is **row-local**: only the same row's dependent is re-evaluated.

### 5.4 Dynamic sub-schemas

When a `dynamic-group` attaches a sub-schema ([08 §5](./08-async-data-and-registries.md#5-dynamic-sub-forms)), the engine compiles it, merges its nodes and edges into the graph, re-runs the cycle check (§5.2) over the merged graph, and rebuilds the topological order. Detaching removes them the same way. Diagnostics from the sub-schema use the `dynamic-group`'s pointer as `location.pointer`, with `location.subSchema: { groupPath, discriminator, pointer }` locating the node inside the sub-schema ([17 §1](./17-diagnostics.md#1-shape)). A sub-schema that fails (including `E_CYCLE`) isn't attached.

## 6. Scheduler

Every external write starts a **tick**:

```
tick(writes):
  batch(() => {
    pending = writes (each with chainDepth 0)
    while pending not empty:
      apply pending                               // origin-tagged
      dirty = dependents(pending), in topo order
      for facet in dirty:
        re-evaluate; if changed → add its dependents to dirty (still in topo order)
      effects = effects triggered by pending whose origin policy allows it (07 §2.1)
      pending = writes produced by effects, each with chainDepth = cause's chainDepth + 1
      drop writes with chainDepth > limits.effectDepth (E_EFFECT_DEPTH, logged with the chain)
    schedule sync validations for affected paths (by trigger mode)
  })
  schedule async work (debounced, outside batch)
```

- The loop repeats until no effect writes anything (stable). Depth is counted **per effect chain** (a write's cause), not per tick, so many independent effects in one tick don't add up. `limits.effectDepth` is set in `createFormEngine({ limits })`, default 8 (all limits: [01 §7.4](./01-schema-contract.md#74-limits)).
- Topological order is precomputed. A tick touches only reachable nodes.
- **Tick budget:** steps across all evaluations of one tick count against `limits.tickSteps` (default 1,000,000, [01 §7.4](./01-schema-contract.md#74-limits)). On excess the tick stops: writes already made commit, facets not yet evaluated keep their previous value and are marked stale, and they are re-evaluated in the next tick. `W_TICK_BUDGET` names the write that started the tick. This bounds fan-out (many dependents × many rows) that per-evaluation `evalSteps` and per-chain `effectDepth` don't.
- Subscribers fire once per tick.
- Async results (options, async validation, dynamic schemas) start **new ticks** when they resolve.
