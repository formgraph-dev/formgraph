# 22 · Debugging & Devtools

[← Index](./README.md)

**Inspectability is an architectural requirement, not a UI feature added after the engine is built.** The engine must explain its state and decisions through a supported observation boundary. This chapter specifies that boundary; no implementation or devtools client is built yet.

## 1. Three layers

```text
Core subsystems                    Optional built-in plugin             Clients
store · DAG · scheduler       →    debug / inspection bridge       →    embedded panel
validation · async · submit        bounded, redacted JSON views          browser extension
schema/compiler source maps        versioned snapshots + events          native / CLI tooling
```

| Layer | Owns | Does not own |
|-------|------|--------------|
| **Core observation points** | Stable identities, read-only projections and lifecycle/cause information from the subsystem that knows it | UI, transports, retained trace history |
| **`debug` plugin** | Sessions, bounded collection, redaction, safe projections and the inspection protocol | Form behavior, validation decisions, arbitrary writes |
| **`@formgraph/devtools`** | Inspector UI and client-side presentation/transport integration | Private store access or a second implementation of engine semantics |

The store's commit hooks are useful but insufficient: they cannot reconstruct which expression ran, why a cascade was suppressed, or why an async result was ignored. Those facts must be observable at the scheduler/evaluator/async boundaries themselves. Public debugger clients never import private engine modules.

## 2. What the engine can explain

| View | Information | Source of truth |
|------|-------------|-----------------|
| State | Layer presence and precedence, computed source, dirty/touched state, current/baseline row identities, conflicts | [03 · State Model](./03-state-model.md) |
| Behavior | Dependency edges, effective visibility/disabled/required policies, evaluated or skipped facets and reasons | [04 · Graph & Scheduler](./04-paths-and-dependency-graph.md), [07 · Effects](./07-effects-and-computed.md) |
| Validation | Rule/source/severity, trigger, eligibility, pending validation, gates and submit stops | [06 · Validation](./06-validation.md) |
| Async | Operation kind/registry id, generation, start/end, cache reuse, deduplication, cancellation, supersession, ignored results and failures | [08 · Async Data](./08-async-data-and-registries.md) |
| Submission | Preflight/lock/finalization phases, serialized and acknowledged units, uncovered edits, reconciliation availability and failures | [09 §4](./09-loading-and-submission.md#4-submission-pipeline) |
| Schema | Input locations, normalized/expanded nodes, macro source links and dynamic graft/detach revisions | [01 §7](./01-schema-contract.md#7-versioning-and-load-time-validation), [15 §8](./15-macros.md#8-tooling) |
| Extensions | Installed plugin names and available capabilities; optionally their explicitly allowed data slices | [03 §8](./03-state-model.md#8-plugins) |

For example, selecting a new country should yield an explainable chain: user write → affected option facet → generation superseded → child selection cleared by its cascade policy → required notice → submit blocked. The UI presents recorded decisions; it must not infer them by comparing two snapshots.

This is the **contract dependency graph**, not the framework's signal/component graph ([D27](./README.md#key-decisions-adr-summary)). Custom fields remain opaque (D25): their internal controls or nested engines are separate inspectable engines, not invented core paths.

## 3. Core observation boundary

Each subsystem exposes a small internal `InspectionHost` capability to the built-in debug factory. It provides core-owned projections and an optional event sink. The plugin does not reach through `PrivilegedPluginStore` to traverse mutable scheduler queues, closures or controllers. This internal capability is not a new application store API or a third-party privilege grant.

- **Single source of truth.** Query the same layer resolver metadata, compiled graph, eligibility decisions and operation generations used by normal execution; do not create a parallel debug rule engine.
- **Non-materializing reads.** Inspection must not call `Readable.get()`, `getValues()`, `serialize()`, validators, formatters, type snapshot hooks or providers to discover state. It reads immutable committed layers, compiled metadata and already-produced facet caches. A stale/unmaterialized facet is reported as such, not evaluated for the debugger. Inspecting does not create controllers, subscriptions on every field or option requests. Adapter read observers are not notified by inspection.
- **Stable boundaries.** Snapshots describe a committed store/graph revision. Event delivery happens after a stable tick/operation boundary, outside store notification and evaluation. Candidate writes rolled back by a veto/exception are labeled rolled back; a partial/stale tick is not presented as a completed evaluation. Debug consumers cannot veto a commit.
- **Causal identities.** Events carry engine-scoped sequence numbers and, where applicable, tick, transaction, operation, parent-cause, compiled-node and row-key identities. A compiled node keeps its id until removed; replacing a node at the same path creates a new id. Unaffected nodes keep ids across dynamic grafts. Index-based data paths are capture-time labels, not row identity.
- **Late attachment.** A session can inspect current state and in-flight work even if it missed their start. Unknown start times/causes stay unknown; no fabricated history. Every snapshot/page identifies its store/graph revision, and pagination restarts on a stale cursor.
- **No effect on business outcomes.** The built-in observation path never writes form state, starts work, alters scheduling policies or changes submission results. External callbacks are trusted host code, not a sandbox; the inspector API itself grants no write/commit capability.

The compiler retains enough source-location and macro mapping metadata for these views. A contract loaded before debug is installed does not require recompilation to be inspected. Original raw text/values are still subject to capture policy, not automatically exported with source links.

## 4. Opt-in plugin and client API

`debug` is a built-in factory at `@formgraph/engine/plugins/debug`, installed through the same registry path as other plugins (D28). **Neither the slim nor the batteries-included entry installs it automatically.** It registers no contract types, adds no contract keys and changes no form defaults. Its absence is never a contract-load error; an explicit call to a missing debug member follows `E_PLUGIN_MISSING`.

```ts
import { createRegistry, loadContract, createFormEngine } from '@formgraph/engine';
import { debug } from '@formgraph/engine/plugins/debug';

const registry = createRegistry().plugin(debug);
const result = await loadContract(json, { registry });
if (!result.ok) throw new ContractLoadError(result.diagnostics);
const engine = createFormEngine({
  contract: result.schema, registry,
  plugins: { debug: { capture: { values: false } } },   // metadata-only is the default
});
const session = engine.debug.connect();             // starts bounded collection explicitly
const off = session.subscribe(batch => client.receive(batch));
// An embedded inspector can call session.snapshot/inspect/graph/history directly.
// A host transport may forward their JSON results; core opens no connection.
// teardown: off(); session.dispose();
```

```ts
interface DebugOptions {
  capture?: { values?: boolean; context?: boolean; payloads?: boolean; contracts?: boolean; pluginSlices?: boolean };
  redact?: Array<{ source: 'values' | 'context' | 'payloads' | 'contracts' | 'plugin-slices'; pointer: string }>;
  limits?: { maxEvents?: number; maxBytes?: number; maxPageNodes?: number; maxDepth?: number; maxValueBytes?: number; maxSessions?: number };
}
interface DebugApi { connect(): DebugSession }   // throws E_DEBUG_SESSION_LIMIT at maxSessions
interface DebugSession {
  snapshot(cursor?: string): DebugReply<DebugSnapshot>;
  inspect(nodeId: string, rowKeyPath?: RowKeyPath): DebugReply<DebugInspection>;
  graph(cursor?: string): DebugReply<DebugGraphPage>;
  history(afterSequence?: number): DebugReply<DebugBatch>;
  subscribe(listener: (batch: DebugBatch) => void): () => void;   // future batches, no immediate call; after dispose() returns a no-op unsubscribe
  clear(): void;                         // clears debug retention only, never form state/history
  dispose(): void;
}
type DebugReply<T> =
  | { ok: true; protocolVersion: 1; engineId: string; epoch: number; data: T }
  | { ok: false; reason: 'disposed' | 'not-found' | 'stale-cursor' | 'limit' };
type DebugValue =
  | { state: 'captured'; value: Json }
  | { state: 'absent' | 'redacted' | 'unavailable' | 'truncated' };
```

The protocol is versioned independently of the JSON contract format. Its exported DTO schemas, not a dump of `FormStore`, define the client boundary:

| DTO | Required content |
|-----|------------------|
| `DebugSnapshot` | Store/graph revisions; form phase/dirty/valid/pending/conflict counts; paged node and active-operation summaries; next cursor if truncated |
| `DebugInspection` | Node identity/kind/source location; optional row instance; layer presence/`DebugValue`s and resolved source; facet cache states (`cached`, `stale`, `unmaterialized`); notice/gate summaries and bounded causal references |
| `DebugGraphPage` | Graph revision; node ids and facet-to-facet edges with read/write/structural/event-only kinds; pagination cursor |
| `DebugBatch` | Protocol version/engine id/retention epoch; sequence range; bounded events; dropped-event and consumer-error counts |
| `DebugEvent` | Sequence/type/phase, platform timestamp, applicable correlation ids, outcome/reason and a category-specific allowlisted projection |

Event categories cover tick/commit, evaluation, async, schema, validation, submission/reconciliation and diagnostics. Values, resolved message text and expression parameters use `DebugValue`; message text and parameters follow `capture.values` (they can embed field values), and no separate message flag exists. A message, parameter or expression result derived from a source whose capture is off is never captured: the projection records provenance per value, and a derived text/parameter that read excluded context, payload, contract or plugin data (for example `$context.accountId`) is `redacted` before retention, listeners or transport. Capture needs **every** contributing source enabled. Category projections have versioned schemas; arbitrary internal objects are not valid event payloads. Runtime schema graph revisions are distinct from contract `schemaVersion`. No engine/controller references, callbacks, abort controllers, raw file handles or reconciliation capability ids cross this boundary.

The host supplies policy at engine creation. Client requests cannot enable values, enlarge limits or bypass redaction. Unsupported protocol versions are rejected by the host bridge before dispatch; the client must negotiate a compatible version rather than reinterpret another version's payload.

## 5. Privacy and access

- All capture categories default to **false**: the default view contains structural/status metadata, not field values, message text, context, query/response bodies, raw contracts or plugin slice contents. Names/paths/timing can themselves be sensitive; metadata-only does not mean anonymous.
- Opting into values does not opt into context, payloads, contracts or plugin slices. Payload capture includes source/target arguments and responses only when explicitly permitted. Credentials, opaque platform handles and executable objects are never a default export.
- Redaction rules use JSON Pointer prefixes within each capture source (`''` matches the entire source). Rules only mask; they cannot grant capture. There is no contract-controlled redaction override. Policy is applied **before** retention, listeners or transport, so exports cannot recover an unredacted buffer.
- Capture uses bounded core-owned JSON projections, never application serialization/getter/snapshot hooks to inspect opaque objects. Unsupported/custom opaque data is `unavailable`; it must not be coerced by invoking `toJSON` or other user code. Returned DTOs are detached copies, not mutable references into engine state.
- Debugging is an explicit host capability. No global engine discovery, automatic network listeners, telemetry upload or remote eval. Authentication, local/remote consent and secure transport belong to the host/client. This is protection against accidental disclosure, not a sandbox against host/plugin code already holding an engine.

## 6. Lifecycle, limits and performance

- Registering the factory does not retain a trace. With no session, probes have a disabled fast path: no event payload allocation, clock reads, value copying, per-field subscription or historical buffer. Core hook code still counts toward the core budget; it is not assumed free or magically tree-shaken.
- Sessions share **one per-engine budget/buffer**, not an unbounded buffer per panel. Draft defaults (not yet benchmarked; set in M0): `maxEvents` 500 retained events, `maxBytes` 1 MiB of retained plus queued DTO bytes counted once per engine rather than per session, `maxPageNodes` 100 nodes per snapshot/graph page, `maxDepth` 6 levels of nesting inside one captured JSON value, `maxValueBytes` 16 KiB per captured value and `maxSessions` 4. Limits are positive finite integers; invalid options fail creation with `E_DEBUG_OPTIONS`, and `connect()` beyond `maxSessions` throws `E_DEBUG_SESSION_LIMIT`. An oversized record is first truncated (`DebugValue` state `truncated`); it is dropped, with a counted sequence gap, only if its minimal record still exceeds `maxBytes`. Limits apply during a tick, not just when flushing.
- Delivery is batched after execution boundaries, never awaited by the engine. Slow/unattached consumers cannot hold a submit open or backpressure the scheduler. Listener/transport failures are isolated as debug consumer errors, not new form diagnostics (which would recursively trace themselves). The ring drops oldest entries, records sequence gaps and exposes drops/errors in subsequent queries/batches.
- `clear()` clears the shared debug buffer and advances its epoch; other consumers see the reset. It does not clear undo, drafts, notices or pending work. Disposing a session removes its listeners; the last session detaches collection and releases retention. Engine disposal closes all sessions, drops retained references and rejects later queries with `disposed`. Async work completing after that cannot resurrect a session.
- Inspection pagination and capture have bounded work. The plugin uses `platform.now()` and a pure-JS size estimator, with no `TextEncoder`, `structuredClone` or DOM ([11 §6](./11-rendering-and-adapters.md#6-platforms)). Enabling debug can alter timings and must be visible in performance reports. CI benchmarks both disabled probes and enabled bounded capture; the core/default entry budgets include the disabled probes and exclude the plugin. The optional plugin and UI have separate reported sizes and are not imported by default.

## 7. UI and transport are replaceable

`@formgraph/devtools` consumes this protocol for layer inspection, dependency/cause navigation, timelines and contract/macro comparison. A browser panel is one client, not the engine integration model. React Native, workers, server processes and tests use the same ES2020 inspection plugin with host-supplied bridges; a native UI or remote agent is not required for core correctness.

The inspection protocol is **read-only**. A host may deliberately add buttons that call existing APIs (`setValue`, `validate`, history, reload), with its own authorization/confirmation, but these are separate from inspection requests and generate ordinary observable operations. No store patching, replaying debug DTOs as drafts, or automatic time travel: redacted/truncated history is not a recoverable execution log, and replaying I/O or acknowledgement state is unsafe.

## 8. Delivery and acceptance

Define protocol identities, cache/materialization semantics, budgets and the internal observation boundary during **M0**. Instrument transactions/DAG/scheduling/validation as they arrive in **M1**, async/loading/submission in **M2**, and macros/localization in **M3**. The polished client stays **M4**; earlier milestones must already be inspectable and tested without it ([12](./12-open-questions-and-roadmap.md)).

Acceptance tests must show that attaching/querying/disconnecting does not change values, notices, outcomes, source-call counts, validation counts, graph materialization or framework read tracking. Test rollback and partial ticks; row reorder and node replacement; late attachment; async supersession/ignored results; submission coverage and reconciliation; privacy before retention/export; stale cursors; budget overflow; throwing consumers; disposal; and two engines/two panels without cross-talk. Tests use deterministic ids/`platform` time, not browser globals. No M0 implementation is authorized by this document alone.
