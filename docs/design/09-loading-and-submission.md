# 09 · Loading & Submission

[← Index](./README.md) · Addresses review items S4, B3

## 1. Config

```ts
interface LoadConfig {
  entitySourceId: string;
  paramKey: string;                    // key in engine `params` (e.g. route id)
  transformerId?: string;              // inbound: raw entity → form values
  reloadPolicy?: 'preserve-edits' | 'replace' | 'replace-unchanged';   // default 'preserve-edits' (§3)
  missing?: 'default' | 'empty';       // fields absent from the loaded values; default 'default' (§2.1)
}

interface SubmissionConfig {
  targetId: string;                    // submitTargets registry
  transformerId?: string;              // outbound: form values → payload
  mode?: 'full' | 'patch';             // patch = dirty paths only; default 'full'
  awaitAsync?: boolean;                // wait for in-flight async validation; default true
  gates?: Record<string, 'allow' | 'confirm' | 'block'>;   // per-gate warning policy; default { warnings: 'allow' }, see 06 §7.3
  conflicts?: 'block' | 'allow';              // default 'block', see §7
  timeoutMs?: number;                        // positive finite deadline for the whole submit, default 30_000 (§4)
}
```

## 2. Data load pipeline

```
phase=loading
entity = await entitySources[id]({ id: params[paramKey] }, ctx)   // AsyncSource, 08 §1.1
values = applyInboundMapping(entity)        // serializeAs reverse map, then inbound transformer
assign row keys (identityKey-aware)
await dynamic-group schema loads for loaded discriminators (parallel)
fill missing fields per `missing` (§2.1), including under those dynamic groups
write loaded layer            (one write, origin 'load' → no effects)
evaluate expression defaults into the default layer (origin 'default')
phase=ready                   (`ready` resolves)
```

### 2.1 Missing vs. null

Precedence goes by key presence ([03 §3](./03-state-model.md#3-precedence-by-presence-not-nullishness)), so an explicit `null` in the loaded values is a real value and hides the default. A key that's **absent** is different: by default the field falls through to its `default`.

| Loaded values | `missing: 'default'` (default) | `missing: 'empty'` |
|---------------|--------------------------------|--------------------|
| `{ "country": "FR" }` | `"FR"` | `"FR"` |
| `{ "country": null }` | `null` | `null` |
| `{}` (key absent) | the default, e.g. `"US"` | the field type's empty value (`null`, `''`, `false`, `[]`, as for the [`clear` action](./07-effects-and-computed.md#21-origin-policy)), written into `loaded` |

- `'default'` suits records created before a field existed: the new field shows its default. Note that full-mode submit then sends that default.
- `'empty'` suits backends where "absent" means "never set": defaults apply only to new records (when nothing is loaded at all).
- **What is filled:** every field of the compiled schema reachable through object scopes (groups with `dataScope`), whether the scope is present in the loaded values or absent. For arrays, only the array key itself is filled (a missing array becomes `[]`); inside each loaded row, the `item` fields are filled. Fields under a `dynamic-group` are filled from the sub-schema of the loaded discriminator, which is awaited first, in the same load write, before `ready`.
- Filled values are ordinary loaded values: not dirty, and `revert` returns to them.
- It applies to `load` and `reload({ entity })`. `applyExternal` changes only the paths it's given, so it never fills.
- For per-field control, the inbound transformer can still write explicit values (it runs before filling).

**Inbound mapping:** if a field has `serializeAs: "customer.email_address"`, loading reads that wire path and stores it under the field's data path. The custom transformer runs **after** the `serializeAs` mapping and receives both the raw entity and the mapped values.

## 3. Reload policy

`engine.reload({ entity?, policy? })` (for example after a save, or when an external change arrives). `policy` defaults to `load.reloadPolicy`. Without `entity`, it loads again from the `entitySource`. A passed entity is raw, like a loaded one, so the inbound mapping (`serializeAs` reverse map, then inbound transformer) and `missing` filling always run. The write has origin `load` (origin policy `history: 'clear'`).

| Policy | `current` | Also clears |
|--------|-----------|-------------|
| `preserve-edits` (default) | Kept. Each path in `current` whose baseline changed goes through conflict detection ([§7.1](#71-conflict-detection)) | Conflicts on paths that are no longer in `current` |
| `replace-unchanged` | Like `preserve-edits`, then every path in `current` whose value equals the **old** baseline (edited but not really changed) is reverted. Arrays are compared atomically | Same as `preserve-edits` |
| `replace` | Cleared | `meta.touched`, `meta.visited`, `meta.conflict`, `meta.stash`, external errors |

With every policy, row keys of replaced arrays are matched by `identityKey` and kept; without one they're reassigned ([04 §4](./04-paths-and-dependency-graph.md#4-array-row-identity)), and meta on the old keys is dropped.

A `reload` or `restore` call that starts while an earlier one is still pending supersedes it: the earlier call rejects with `E_SUPERSEDED` and writes nothing (latest wins).

After a successful submit, a returned entity advances the baseline through acknowledgement-aware replacement (§4.2), not the host's `reloadPolicy`: acknowledged overrides clear, while excluded and later edits survive. Ordinary explicit `reload({ policy: 'replace' })` still follows the table above and is an explicit discard operation.

## 4. Submission pipeline

```
engine.submit():                  // submit({ confirm?: string[]; signal?: AbortSignal; timeoutMs?: number }): Promise<SubmitResult>
  if a submit is in flight → return its promise (no second run, no error)
  if a failed-response reconciliation is pending (§4.2) → return { ok:false, reason:'reconcile-pending', reconciliationId }   // no count, lock, hooks or save
  submitCount++; start deadline/cancellation race (§4 below); clear lastSubmitError
  await ready without changing initialization's loading/error phase; readiness failure → fail('load')
  phase=submitting
  repeat preflight until value/eligibility version is stable:
    await schema loads for eligible dynamic groups; failed/mismatched → install group notice; fail('schema')
    force-compile controllers, validators and option state for eligible fields (03 §6.3)
    start option loads and current-value async/component validation, including child settlement
    await value-changing options/schema/computed work; if awaitAsync: await validation and busy work
    run full submit validation against the resulting values, not the pre-load values
    if that starts work or values/eligibility change across an await, repeat; apply deadline/tick budgets
  // With awaitAsync:false, pending validation/busy work returns its existing failure reason, never a successful stale pass.
  if any severity=error (including sources 'input' and 'component', 11 §2.1–2.2)
     or any field-type pending work failed (assets; 01 §3.4)   → fail('errors')
  if async validation still in flight (awaitAsync: false)       → fail('validating')
  if field-type work still busy (awaitAsync: false)             → fail('assets-pending')
  if conflicts and submission.conflicts=block                   → fail('conflicts', { paths })
  confirm(opts.confirm) ; gated = gates with notices whose policy is block,
                            or confirm and not confirmed (06 §7.3)
  if gated non-empty                                            → fail('gated', { gates })
  acquire edit lock and capture the settled state atomically, rechecking its version; if changed → repeat preflight
  plugin 'before' hooks (§4.1)                                  → fail(hook's reason)
  payload = serialize()                   // outbound transformer throws → fail('transform') (08 §1.1)
  retain frozen checkpoint + entire current overrides + serializedPaths (§4.2)
  result  = await submitTargets[targetId]({ payload, mode, confirmed, serializedPaths }, ctx)   // AsyncSource, 08 §1.1
            // throw/reject → lastSubmitError = error; fail('target')
  if (result.errors?.length ?? 0) > 0 → map external errors (§5) → fail('external')
  targetAccepted = true
  response = 'none'
  validate final acknowledgement coverage (§4.2), even without an entity
    // missing required/invalid coverage → response='failed', retain checkpoint/state, report E_SUBMIT_RESPONSE + reconciliationId
  if response !== 'failed' && Object.prototype.hasOwnProperty.call(result, 'entity'):
    stage acknowledgement replacement; reapply excluded edits by stable identity; atomically commit (§4.2)
      // failure/cancellation/deadline → response='failed'; retain checkpoint/state; responseDiagnostics={diagnostics,reconciliationId}; record E_SUBMIT_RESPONSE
    if applied: response = 'applied'
  savedRevision = engine.revision.get()   // after response application, BEFORE deferred/listener changes
  savedVersion = value/eligibility version
  finalize('submitted')        // unlock + drain queued updates + phase change; notify only after all are complete
  pendingChanges = retained unacknowledged edits exist OR value/eligibility version changed since savedVersion OR engine.revision.get() !== savedRevision
  return {ok:true, response, savedRevision, pendingChanges, ...responseDiagnostics}

fail(reason, extra?):
  finalize(initialization.pending ? 'loading' : initialization.failed ? 'error' : 'ready')
  r = {ok:false, reason, ...extra}
  run plugin failure observers, then onSubmitFailed(r) (11 §8); return r
```

```ts
type SubmitSuccess =
  | { ok: true; response: 'applied' | 'none'; savedRevision: number; pendingChanges: boolean }
  | { ok: true; response: 'failed'; savedRevision: number; pendingChanges: boolean; diagnostics: Diagnostic[]; reconciliationId: string };
type SubmitResult = SubmitSuccess
  | { ok: false; reason: 'load' | 'errors' | 'external' | 'transform' | 'validating' | 'assets-pending' | 'schema' | 'target' }
  | { ok: false; reason: 'reconcile-pending'; reconciliationId: string }
  | { ok: false; reason: 'conflicts'; paths: DataPath[] }
  | { ok: false; reason: 'gated'; gates: GateStop[] }
  | { ok: false; reason: 'cancelled' | 'timeout'; outcome: 'not-sent' | 'unknown' }
  | { ok: false; reason: `x-${string}`; notices?: NoticeInput[] };   // a plugin's `before` hook stopped the submit (§4.1)

type GateStop = { gate: string; policy: 'confirm' | 'block'; notices: Array<Notice & { path: DataPath }> };   // 06 §7.3
```

**Settlement before locking.** `phase: 'submitting'` includes a cancellable preflight during which value writes remain allowed. Options, schema loads and component/child validation can change values, so a full submit-validation pass must run **after** they settle. Any new work or value/eligibility change restarts settlement and validation; unchanged settled async results are reused, not blindly restarted. Gates are computed from the final pass. Capture and lock are synchronous with a version recheck, so no write can slip between the checked snapshot and locking. A constantly changing form is bounded by the overall deadline, not an infinite stabilization loop. `validate()` and wizard `next()` use the same stable-validation rule for their scope. This includes waiting for component busy tokens when awaitAsync is enabled.

**Edit lock.** Only after successful preflight, before `before` hooks and serialization, does the exclusive lock begin. From then through response application, user/host/plugin value or row-structure writes (`setValue`, `revert`, `reset`, array commands, history replay and host `reload`) reject with `E_SUBMIT_BUSY`; `restore` retains `E_RESTORE_BUSY` throughout submission. Renderers use the form phase to prevent editing without changing `disabled` or eligibility. Every mutation entry checks the lock. `applyExternal` queues throughout submission. Once locked, core value-changing completions and context-driven recomputation are deferred to finalization; custom components buffer rejected value updates as subsequent edits, rather than throwing from subscription callbacks ([16 §22](./16-cookbook.md#22-another-engine-as-a-custom-field)). The submit's own response reload is internal and allowed. Failure, success, cancellation and disposal release the lock; deferred work is generation-checked before applying. Thus a response `replace` cannot silently discard a newer accepted edit.

**Cancellation and deadline.** The first call's `signal` and `timeoutMs` govern a shared in-flight submit promise; later calls return that promise and do not replace its controls. The default overall deadline is `submission.timeoutMs ?? 30_000`; per-call `timeoutMs` may override it with a positive finite value. `engine.cancelSubmit()` cancels the current attempt (no-op when idle). Abort/deadline is raced against **every** awaited stage, including readiness, sources, component/busy settlement, hooks and response processing. It does not depend on the callee honoring its signal: revoke the attempt generation, stop its continuation, abort submit-owned work, release the lock, drain queued updates and resolve once. Before target acceptance, this returns `reason: 'cancelled'` or `'timeout'`; after acceptance, response-application interruption is the saved-but-unapplied success below. Late resolutions/rejections cannot call the target, apply its response or finalize the old attempt. While waiting for initialization, cancellation preserves `loading` (or the actual initialization `error`), never fabricates `ready`; the independent load continues and owns its eventual phase transition. Once initialized, ordinary failure/cancellation returns to `ready`. Shared field loads may continue for other consumers, but never resume the cancelled submit. Final observers cannot hold completion hostage; timers use `platform`, not DOM globals.

Before the target is invoked, cancellation `outcome` is `'not-sent'`; while its result is unknown it is `'unknown'`, even if its signal is aborted. A timed-out/cancelled target may already have saved remotely. Never auto-retry it; reconcile with the backend or use host idempotency keys first. Once a no-errors response is accepted, the save is known to have succeeded and the result stays `ok: true` even if response application fails. The same uncertainty can apply to target transport rejection. Editing state survives cancellation; disposal is not required.

**Target failure.** A thrown/rejected target records the original error in `FormState.lastSubmitError` and reports a runtime `E_SUBMIT_TARGET`. It returns `{ ok: false, reason: 'target' }`, keeps local edits, restores `ready`, unlocks, drains external/deferred updates, then runs failure hooks and `onSubmitFailed`, in that order. The host may offer retry; the engine never automatically retries a potentially non-idempotent target. All other failure branches also drain the queue. Success still applies any response entity first, then unlocks and drains before success observers. Finalization is shared so cleanup cannot be skipped by a failure branch.

**Saved, but response not applied.** `entity` is tested by own-property presence, so `null`, `false`, `0` and `''` all run the inbound pipeline (a present `undefined` is not JSON and is an invalid response, not an omitted entity). After a no-errors target response, stage mapping, schema loads, acknowledgement replacement and retained-edit replay off-store (§4.2). Any failure/interruption discards the plan, preserves live state, reports `E_SUBMIT_RESPONSE` and retains the checkpoint/raw response. Finalize/unlock/drain normally, set `phase: 'submitted'`, dispatch success observers and return `ok: true, response: 'failed'` with diagnostics and an engine-scoped `reconciliationId`; do not call `onSubmitFailed` or the save target again. The host shows “Saved, refresh required” and retries `reconcileSubmit(reconciliationId)` after fixing the local failure, optionally passing a refetched raw entity. Plain `reload({ policy: 'preserve-edits' })` is **not** acknowledgement reconciliation: it would mistake the already-submitted current layer for newer unsaved edits. `response: 'none'` means no entity was returned; `applied` means the acknowledgement committed with unacknowledged edits preserved.

**Unacknowledged work.** Every success returns `savedRevision`: the local engine revision after its internal acknowledgement commit (or unchanged state when none/failed), before deferred work and listeners can write. `pendingChanges` is computed after synchronous finalization/listener/observer work: true if overrides outside the **final** acknowledgement coverage remain, or either revision or submission value/eligibility version changed after the acknowledged state. This includes serializer/transformer omissions and deferred defaults/computed/context changes that do not bump `revision`. It may conservatively retain externally changed work; it must not omit an unsubmitted change. Finalization drains synchronous listener ticks before resolving submit. Retain the editor/draft if `pendingChanges` is true **or** a fresh revision comparison differs from the token. This is not a backend revision or a promise about future writes. Failed response application always requires reconciliation even when neither counter changed.

`finalize` releases the lock and drains queued updates before publishing the final phase, in one notification boundary. A phase subscriber that sees `ready`/`submitted` may write immediately; it must not observe a still-held lock or a baseline that has not yet drained. Such listener writes form a subsequent tick, not part of the frozen submitted payload.

If cancellation occurs before initialization establishes a baseline, its external-update queue remains engine-owned and is drained after initialization commits, not applied to a fabricated ready state. Cancellation does not supersede the independent initial load or alter its eventual success/error phase.

#### 4.1 Plugin submit hooks

Plugins observe the pipeline with `ctx.submission.on(event, fn)` ([03 §8.1](./03-state-model.md#81-plugin-api)); autosave, audit and analytics plugins use it.

```ts
type SubmitStop = { stop: `x-${string}`; notices?: NoticeInput[] };
type SubmitHook = (e: { mode: SubmitMode; confirmed: string[]; result?: SubmitResult; signal: AbortSignal }) => void | SubmitStop | Promise<void | SubmitStop>;
```

- `before` runs after validation and gates pass, before `serialize()`. It may be async (awaited, aborted with the submit's `signal`) and may return `{ stop: 'x-…' }`, which fails the submit with that reason (`notices` become form-level notices). It's for programmatic stops; asking the user to confirm is a gate ([06 §7.3](./06-validation.md#73-gates-submit-policy-for-warnings)).
- `success` and `failure` are dispatched after finalization, observe only, and cannot change the result. Invoke them in registration order without awaiting their returned promises; async rejections are reported separately and cannot stall completion. `failure` hooks are invoked **before** the host's `onSubmitFailed` (their asynchronous continuations may finish later).
- Hooks run in plugin order. A thrown hook is reported through `onDiagnostic` and doesn't change the result, except that a throwing `before` stops the submit with `reason: 'x-<plugin>'`.

`onSubmitFailed` runs for **every** `{ ok: false }` result; the adapters' default branches on `reason` ([11 §8](./11-rendering-and-adapters.md#8-accessibility)).

#### 4.2 Acknowledgement and response reconciliation

The frozen checkpoint contains resolved values, all current overrides, row identities and `serializedPaths`: exact data-bearing units included by core serialization before outbound transformation. A field path means its whole persistent value (custom fields remain atomic); an array-node path means **row topology only**, never an implicit acknowledgement of all descendant cells. Group paths/wildcards are not coverage units. Snapshot indexes resolve to the captured row keys, not whatever rows occupy those indexes later.

**Final coverage is explicit across a transform.** `SubmitTargetResult.acknowledgedPaths` names the exact frozen units actually persisted. Without an outbound transformer it may be omitted (defaults to `serializedPaths`) or restrict that set. With a transformer it is required, including when no entity is returned: the host target/transformer pair must attest which form inputs contributed to the accepted payload. It may name a known field intentionally added from full form values, but must not claim an input the transformer removed. Validate all units against the frozen schema; custom subPaths, unknown paths and group/wildcard claims are invalid. Missing required/invalid coverage retains all edits and returns `response: 'failed'` with `E_SUBMIT_RESPONSE` and a reconciliation id, even if the save succeeded. Correct metadata through `reconcileSubmit(id, { acknowledgedPaths })`, never by repeating the save. This is a trusted host integration contract, not inferred backend persistence or a contract-controlled security claim.

**No entity: acknowledged value.** Without an entity the `loaded` layer is stale for covered paths. Core therefore records a bounded, runtime-only *acknowledged value* for each covered path (the frozen value; arrays and custom fields atomic) and uses it, instead of `loaded`, for dirty and `patch` comparisons of that path. A field saved as `B` over loaded `A` and then edited back to `A` is still dirty, and a later patch resends `A`; `revert`/`reset` still restore `loaded` and remain dirty relative to the acknowledgement. The record is not part of drafts or SSR state and is cleared by any entity-bearing acknowledgement, load/reload that replaces the layer, reset, restore, or external update to that path. It is a claim about what was sent, not about server normalization; hosts that need the server's value should return an entity or refetch.

Coverage acknowledges the **captured values/intents**, not future values at those paths. A later edit to an acknowledged field is still unsent. If coverage is corrected without any entity (retained or supplied), do not replace the baseline or delete covered current overrides: only classify the accepted snapshot, preserving the existing no-entity behavior. Return the original saved revision token and mark uncovered or post-capture work pending. Supplying an entity enables actual baseline reconciliation and captured-override removal; neither form can erase later edits just because their paths were acknowledged earlier.

For example, if a transformer removes `privateNote`, its target returns `acknowledgedPaths: serializedPaths.filter(p => p !== 'privateNote')`. A returned entity with the old note cannot then erase the edited note or make draft cleanup appear safe. Exact units matter for arrays: acknowledging an array's topology does not acknowledge a cell omitted by the transformer.

**Retain uncovered edits.** All overrides outside final coverage (whether omitted by policy, transformer or partial target acknowledgement) stay in `current` and force `pendingChanges: true`, with or without a response entity. With an entity, stage its baseline, clear only acknowledged overrides and overlay uncovered edits before committing. Stashes and their unsent overrides also survive. Match array cells by stable identity/`identityKey`, not index; array storage remains atomic. If correspondence is unavailable, retain the local array as a conflict against the acknowledged baseline, with unresolved row provenance. Uncovered fields removed from the new schema fail the plan closed (`response: 'failed'`), not silently disappear. Retained edits remain present, dirty when unequal to baseline, and included in drafts. A save of other fields is not implicit permission to discard them.

**Retry against the submitted checkpoint, not the old loaded layer.** For a failed response application, core retains one checkpoint, its raw result/coverage, coalesced post-capture local intents (including reverts and row identities), and a coalesced external-update overlay. This is bounded pending-operation state, not a commit log. Reconciliation stages the acknowledged baseline, applies the external overlay, removes captured acknowledged overrides, then replays uncovered/later intents. Reverts follow the resulting baseline. Ordinary editing may continue while staging; the final synchronous plan includes latest intents/updates. A submitted `" Alice "` acknowledged as `"Alice"` with no later edit displays `"Alice"` without a conflict; later `"Bob"` is preserved. Genuine conflicts between external changes and later edits use §7's rules; false conflicts caused only by already-acknowledged overrides are removed, and explicit conflict resolutions are not resurrected without a newer external change. Ambiguous replay fails without partially applying or dropping the draft.

**External updates do not expire recovery.** `applyExternal` updates, including those drained by submit finalization, remain authoritative in live state and are also folded into the pending checkpoint's overlay (set/remove and ancestor replacements retain arrival-order semantics). They do **not** invalidate its id. Default retry uses the retained acknowledgement plus all such updates, so it cannot roll back a newer external value. If the caller supplies a refetched `entity`, it asserts that this is authoritative for external updates already known at that call; only updates arriving after that boundary overlay it. Capture/recheck the external generation at commit, restaging if needed. External-driven schema/identity changes that cannot be replayed safely reject staging while preserving live state and the same recovery id; the host can supply an authoritative refetch or resolve the retained draft explicitly. Thus finalization never returns an id made unusable merely by draining its own external-update queue.

```ts
interface SubmitReconciliation { savedRevision: number; pendingChanges: boolean }
engine.reconcileSubmit(reconciliationId: string, opts?: {
  entity?: Json; acknowledgedPaths?: DataPath[]; signal?: AbortSignal; timeoutMs?: number;
}): Promise<SubmitReconciliation>;   // no target call; defaults to retained entity/coverage, which may be corrected explicitly
```

The opaque id belongs to this engine's current pending acknowledgement. While it is pending, `submit()` refuses to run (`reason: 'reconcile-pending'`), so a later partial save cannot orphan the first acknowledgement or let uncovered local overrides mask a normalized server value. Only a successful `reconcileSubmit`, a successful explicit host reload/reset/restore, unrelated contract replacement or disposal invalidates it (`E_RECONCILE_UNAVAILABLE`); ordinary external updates instead rebase it as above. A failed/staged host load does not itself expire recovery. Calls during submit reject with `E_SUBMIT_BUSY`; overlapping reconciliation calls use latest-wins (`E_SUPERSEDED`). Failure/interruption rejects the **call promise** with `E_SUBMIT_RESPONSE` and diagnostics, keeping live state and the same id/checkpoint; it does not return another `SubmitSuccess` or create a new id. A successful commit consumes the id and returns revision/pending-work metadata, including uncovered edits. Staging shares cancellation/deadline/generation guards. IDs/checkpoints are runtime-only, not draft/SSR data; after restart, use ordinary loading and explicit stale-draft handling instead of an expired id.

### Public building blocks

`submit()` is a composition of public methods that hosts can call directly (for example for drafts, or submitting while invalid, see [16 §1](./16-cookbook.md#1-autosave-and-drafts) and [§14](./16-cookbook.md#14-submit-while-invalid)):

```ts
engine.validate({ trigger?: 'submit', paths?: PathRef[] }): Promise<{ ok: boolean; notices: Array<Notice & { path: DataPath }> }>   // path: the node each message belongs to
engine.serialize({ mode?: 'full' | 'patch', validate?: boolean /* default false */ }): Json
engine.mapExternalErrors(errors: ExternalErrors): void
engine.reload({ entity?, policy? })
```

`serialize()` is synchronous and never mutates state; like `validate()`, it force-compiles what it needs first ([03 §6.3](./03-state-model.md#63-lazy-compile)). With `validate: true`, it runs a sync validation pass (no async sources are called or awaited), then throws `E_INVALID` if any `error`-severity messages exist, including existing async and external ones. Warnings and info don't affect it.

### serialize()

1. Walk the CompiledSchema and include each field according to its effective `whenHidden` / `whenDisabled` policies ([07 §4](./07-effects-and-computed.md#4-hidden-and-disabled-field-policy)).
2. `mode:'patch'`: only dirty paths are included (arrays as whole values).
3. Assets serialize as `{ ref, name, size, mime, meta? }` ([01 §3.3](./01-schema-contract.md#33-core-config-and-ui)); `status`, `progress`, and `error` are stripped.
4. Remap data paths to `serializeAs` wire paths (dot paths create nested objects).
5. Run the outbound transformer if one is configured. It receives `({ payload, mode, loaded, values, dirtyPaths, confirmed }, ctx)` (`OutboundTransformer`, [08 §1.1](./08-async-data-and-registries.md#11-shared-function-shapes)): the payload so far, the saved data as loaded (after the inbound mapping), the full resolved values, the list of changed paths and the confirmed gates. That's enough to compute row-level diffs ([16 §17](./16-cookbook.md#17-row-level-diffs-for-patch-submissions)).

In `patch` mode, **arrays are always atomic** (Q4): if anything in an array changed, the whole array is sent. Row-level diffs are a transformer concern, not a contract feature.

## 5. External error mapping

The submit target returns:

```ts
type ExternalErrors = Array<{ path: string; message: Text; code?: string; severity?: Severity }>;  // severity default 'error'; registered x- severities allowed, unknown → 'error' (06 §7.6)
```

`path` is a **wire path** (`serializeAs` space) or a data path. The engine maps it back to data paths, then to row key paths. If the path goes **deeper than a field** (`/grid/rows/3/qty` when `grid` is a field), the message goes on that field and the remainder becomes its `subPath` ([06 §5](./06-validation.md#5-notice-shape)). Errors that can't be mapped go to `FormState.notices` (a form-level banner). External errors on a path clear on the next user edit to that path. `code` is kept on the resulting `Notice` ([06 §5](./06-validation.md#5-notice-shape)).

## 6. JSON Schema export of the submitted data

Borrowed from JSON Forms ([14](./14-prior-art.md)). This lets backends validate submissions without knowing the contract format.

```ts
import { toPayloadSchema } from '@formgraph/contract';
type JsonSchema = Record<string, Json>;   // a JSON Schema document (draft 2020-12 by default)
const schema = toPayloadSchema(contract, { draft: '2020-12', mode: 'full' }); // JSON Schema of serialize() output
```

This is **not** the JSON Schema of the contract itself ([01 §7](./01-schema-contract.md#7-versioning-and-load-time-validation)), which describes forms. This one describes the *data a form submits*.

Mapping:
- Field → property at its `serializeAs` wire path. The type comes from the field's value shape ([01 §3.3](./01-schema-contract.md#33-core-config-and-ui)):
  - `text` → `string`; `number` → `["number", "null"]`; `boolean` → `boolean`; `date-time` → `string`. Date-only values get `format: 'date'`; offset-bearing UTC/IANA date-times get `format: 'date-time'`. Offsetless local date-times and all time-only values omit RFC 3339 `date-time`/`time` formats (those require an offset); their granularity/timezone is recorded in `x-formgraph` annotations instead.
  - `choice` → `enum` of the option values plus `null` (no `enum` when `validateInOptions: false`); `multi-choice` → `array` with `items.enum` and `uniqueItems: true`. Static sources give the list; `map` sources give the union of all lists; `provider` sources give the scalar type only.
  - `asset` → object `{ ref, name, size, mime, meta? }` or `null`; `multi-asset` → array of those.
  - Custom types use their registered `payloadSchema` ([01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions)).
- Rules → keywords, including rules implied by config ([01 §3.3.1](./01-schema-contract.md#331-implied-rules)): `required`, `min/max` → `minimum/maximum`, `step` → `multipleOf` (only when `min` is absent or a multiple of `step`, since steps count from `min`; otherwise listed in `x-formgraph-unexported`), `minLength/maxLength`, `formatMinimum/formatMaximum` (`date-time` min/max with literal bounds), `pattern`, `minItems/maxItems`, `email` → `format: email`.
  - Optional text with a positive `min-length` uses `anyOf: [{ const: '' }, { type: 'string', minLength: N }]`, because the rule skips empty input. Other rules that skip empty input belong inside the non-empty branch; they must not invalidate the empty branch from outside it. A statically required text field uses the non-empty branch (and at least `minLength: 1`). Conditional required is conservatively optional in the export. Other empty-skipping rules must likewise preserve the empty branch rather than narrowing the runtime's accepted values.
  - Temporal bounds use `formatMinimum`/`formatMaximum` only with an applicable exported format and a validator extension that matches the engine's comparison semantics. Otherwise (including local date-times, time-only values and expression bounds) they are listed in `x-formgraph-unexported`; they must not invent an RFC 3339 format for offsetless storage.
- `content` nodes have no value and never appear ([01 §3.6](./01-schema-contract.md#36-content-nodes)).
- Fields that are **statically** hidden with `whenHidden: 'exclude'` or `'clear'` (for example `visible: false`, as the `variable` macro produces), or statically disabled with `whenDisabled: 'exclude'` (the default), are left out entirely. Computed fields are included like any other field.
- **Outbound transformer:** if `submission.transformerId` is set, the payload can change shape. The schema then describes the payload **before** the transformer, and says so in `x-formgraph-unexported`. A transformer registration may provide `payloadSchema(input) → JsonSchema` to describe its output instead.
- Arrays → `items` with the row schema. `dynamic-group` → `oneOf` over cached sub-schemas, keyed by the discriminator with `if/then`, or left open (`additionalProperties: true`) when the sub-schemas aren't known ahead of time.
- **Can't be expressed, so it's left out with a note:** conditional `required`/`visible` with arbitrary expressions, registered functions, async validators, provider options. Each is listed under `x-formgraph-unexported` so backends know what isn't covered.
- `mode: 'patch'` makes every property optional.
- Fields that are hidden or disabled under an `exclude` policy can still appear in a payload (when their condition is false), so they are never `required` unless the condition is static.

The export is a best-effort **outer bound**. Every valid submission validates, but not everything that validates is a valid submission.

## 7. External updates and conflicts

Decided in Q6 ([12](./12-open-questions-and-roadmap.md)). For changes from outside this form instance that arrive while the user is editing: another user or tab saved, a workflow changed a status, a sync engine merged data, or a push channel sent a patch. This isn't real-time collaboration: there's no text merging and no presence.

```ts
engine.applyExternal(
  changes: Array<{ path: string; value?: Json; op?: 'set' | 'remove' }>,
  { paths?: 'wire' | 'data' }      // default 'wire': paths are mapped back through serializeAs
);
```

- Updates **only the given paths** in the `loaded` layer, with origin `external`: no effects, and cascades trust the value ([13 §4](./13-option-sources-and-cascading.md#4-invalidation)). The inbound transformer is **not** run, because it works on whole entities. Send the mapped value shape.
- Arrays are atomic, as everywhere else. A change replaces the whole array, and row keys are matched by `identityKey` when there is one ([04 §4](./04-paths-and-dependency-graph.md#4-array-row-identity)).
- During `submit()`, external changes are queued and applied after it finishes. If the submit reloaded from a response entity, each queued change is first compared with the new `loaded` value at its path; changes already reflected there are dropped. If the response application failed and a reconciliation is pending, queued changes are also folded into its overlay and do not invalidate its id ([§4.2](#42-acknowledgement-and-response-reconciliation)).

### 7.1 Conflict detection

The same rules apply to every changed path in `applyExternal`, in `reload` with `preserve-edits` or `replace-unchanged` (§3), and to stale paths in `restore` (§8.2). For each changed path `p`, with `base` = the old loaded value, `external` = the new one, and `local` = `current[p]` if present:

| Situation | Result |
|-----------|--------|
| No local edit at `p` | The user sees `external`. No conflict |
| `local` equals `external` | Already in sync: the local edit is dropped (revert). No conflict |
| `local` differs from `external`, and `base` equals `external` | Nothing actually changed externally. No conflict |
| `local` differs from both | **Conflict**: `meta.conflict[p] = { base, local, external }`. The field keeps showing `local` |

### 7.2 Resolving

```ts
interface Conflict { path: DataPath; base: Json; local: Json; external: Json }
```

Per-field handles on the `FieldController` ([11 §2](./11-rendering-and-adapters.md#2-headless-field-api)): `conflict: Readable<Conflict | null>` and `resolveConflict('local' | 'external')`.

- `local`: keep the local value. The flag is cleared. The field stays dirty, now relative to the new `loaded`.
- `external`: revert `current[p]`. The flag is cleared, and the field shows `external`.
- A user edit to a conflicted field **doesn't** clear the flag, so a conflict can't be dismissed by accident.

Conflicts live in `meta.conflict`, keyed by row key path like other meta ([03 §5](./03-state-model.md#5-field-meta-and-form-status)), so they follow moved rows. A conflict is cleared by:
- `resolveConflict` (either side);
- deleting `current[p]` by any means (`revert`, undo, a later external change that matches `local`);
- `reload` with `replace`, `reset`, and `restore` (which recomputes conflicts for its own paths).

Form level: `conflictCount`, and `submission.conflicts: 'block' | 'allow'` (default `block`). With `block`, `submit()` returns `{ ok: false, reason: 'conflicts', paths }`.

### 7.3 Interaction with undo

The `external` origin has `history: 'clear-overlap'` ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)): `applyExternal` clears undo/redo history only if a changed path appears in a recorded step ([03 §6.2](./03-state-model.md#62-undo--redo-opt-in)). Otherwise history is kept, so frequent unrelated pushes don't wipe the user's undo stack.

## 8. Snapshots (drafts)

A draft must reopen with the user's edits **and** their change indicators. Saving the payload and loading it back doesn't do that: the draft would become the baseline, so nothing would be dirty. `serialize()` also drops excluded fields and renames paths, so it loses data.

Provided by the `drafts` plugin ([03 §8](./03-state-model.md#8-plugins)), which the default entry installs.

Instead, a snapshot saves the **`current` layer** (exactly what the user changed), and `restore` puts it back on top of the **same baseline**: the contract defaults for a new record, or the same saved entity for an existing one. Dirty ([03 §4](./03-state-model.md#4-dirty-semantics)) then works unchanged, and the indicators come back without any stored indicator state.

```ts
engine.snapshot(): Snapshot                 // synchronous, pure JSON
engine.restore(snapshot, { onStale?: 'conflict' | 'keep-draft' | 'drop' /* default 'conflict' */ }): Promise<RestoreResult>
engine.revision: Readable<number>           // core (03 §7): bumps on each commit with a non-empty diff to `current`, `rows` or `stash`

interface Snapshot {
  version: 1;
  formId: string;
  schemaVersion: string;
  contractHash: string;                     // hash of the normalized contract
  revision: number;                         // engine.revision when taken; only meaningful in the session that took it
  current: Json;                            // the sparse top layer
  rows: Record<DataPath, RowKey[]>;         // array row identity, so per-row indicators and errors line up
  baselineRows: Record<RowKeyPath, BaselineRowKeys>; // keys + unresolved provenance (03 §3.2), paired with base values
  base: Record<DataPath, Json | null>;      // baseline of each stored leaf of `current` when taken
  optionDeps: Record<RowKeyPath, Json>;      // canonical dependency-input tuples for saved choice selections (§8.2)
  stash?: Json;                             // dynamic-group orphans kept by orphanPolicy 'stash' (08 §5)
  touched?: RowKeyPath[];                   // row key paths; restored so messages show as they did
  takenAt: string;                          // ISO timestamp
}

interface RestoreResult { applied: DataPath[]; conflicts: DataPath[]; dropped: DataPath[] }
```

**Granularity.** The paths of a snapshot (`base` keys, `RestoreResult` lists, stale checks) are the **stored leaves** of `current`: each array counts as one entry ([03 §3.1](./03-state-model.md#31-resolution-granularity)). `touched` is mapped through the restored row keys; entries under rows that no longer match are dropped.

`baselineRows` preserves which current keys originally belonged to each saved baseline array, independently of current order and insertions. Drafts include maps for arrays under their saved atomic `base` entries (and stash baselines); arrays with no restored current override keep live baseline maps, not draft current keys. For an unchanged baseline, restore installs saved associations against `base` values recursively before descendant revert is available. Validate key uniqueness, `keys.length` against the saved baseline array, disjoint `keys`/`unresolved` lists, nesting and map/value consistency (`E_SNAPSHOT_INVALID` otherwise). With a changed baseline, match by `identityKey` where available; otherwise assign the new baseline fresh keys and retain formerly-baseline draft keys in `unresolved`, including any already-unresolved keys. Descendant revert rejects those keys with `E_ROW_BASELINE_UNKNOWN`, never guessed indexes. Keys in neither list are genuinely inserted and use defaults. These markers survive another snapshot/restore and stash round-trip. Accepting the external whole array or whole-array revert restores known associations; choosing `local` or `keep-draft` retains unresolved markers even without a conflict flag.

**Field-type filter.** `snapshot()` stores each value through its field type's `toSnapshot` hook ([01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions)). For assets (`assets` plugin) only values with status `ready` are kept. Others (pending, uploading, error) are left out with `W_SNAPSHOT_ASSET_DROPPED`, since their uploads don't survive the session.

**Saved option dependencies.** `optionDeps` stores each saved choice/multi-choice selection's cascade dependency-input tuple from the **resolved draft**, not its baseline. Include explicit/inferred form/context dependency reads (`dependsOn`, map keys and external inputs to filters/disable expressions), with tagged presence so missing and `null` differ. Exclude `$option`/`$selected` local roots, result lists, search/pagination UI state and provider locale variation that merely reloads labels. Provider variation still reloads/revalidates options; it does not by itself justify clearing a restored selection. Capture unedited dependency fields too, even though they are absent from `current`/`base`; nested instances use stable row-key paths. Stashes retain corresponding tuples. These are canonical JSON values compared structurally, not lossy hashes. They are untrusted hints about prior selection context, never a way to skip current `in-options` or other validation. Their schema, paths and size are validated with the snapshot.

### 8.1 Flow

```ts
// Session 1
const engine = createFormEngine({ contract, registry });
await engine.ready;                         // defaults (and loading, if editing a record)
// … user edits → fields show dirty indicators …
const snap = engine.snapshot();
await drafts.put(key, snap);                // host storage: API, IndexedDB, file (D17)
savedRevision = snap.revision;              // host state: only once storage succeeded

// Session 2
const engine = createFormEngine({ contract, registry });
const draft = await drafts.get(key);
if (draft) await engine.restore(draft);     // waits for `ready`, then applies
// → same values, same dirty indicators
savedRevision = engine.revision.get();     // the restored state is what's stored
```

### 8.2 Restore rules

1. **Waits for `ready`**: loading and dynamic-group schema loads finish first. Rejects with `E_RESTORE_BUSY` while `phase` is `submitting`. A later `reload` or `restore` supersedes a pending one (§3).
2. **Validates the snapshot.** Snapshots are untrusted input (they come back from storage): the shape is checked against the `Snapshot` schema first (`E_SNAPSHOT_INVALID`). Older snapshot `version`s are upgraded by built-in migrations, like contract migrations ([01 §7](./01-schema-contract.md#7-versioning-and-load-time-validation)). A different `formId`, or a `version` with no migration, rejects with `E_SNAPSHOT_MISMATCH`. A different `contractHash` isn't an error: each path is checked instead (rule 5).
3. **Plan discriminators before loading schemas.** Against an immutable baseline/contract generation, decide stale handling (rule 6) for each discriminator or its containing atomic array **without writing live state**. A dropped draft discriminator selects the baseline schema; kept/conflicted draft values select the draft schema. Stage required schema loads, normalization, compilation and graft/orphan plans off-store, recursively for nested groups. Do not detach the live schema, run orphan disposal or alter any live layer while awaiting. A failed/superseded stage discards the plan and leaves live state unchanged. A baseline or relevant live-value change during staging supersedes the restore with `E_SUPERSEDED`; the caller may retry against the new state.
4. **Commit with origin `restore`** ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)): after rules 5–6 resolve all remaining paths against the planned schemas, recheck the captured generation and commit planned grafts, layers, row maps, conflicts, stash and touched state **atomically**, with no effects (origin policy `history: 'clear'`). An applied draft array reuses validated current draft keys and reconciled baseline associations; a dropped array keeps baseline current keys and baseline associations. Never install draft keys on a different baseline array. Reconcile nested arrays recursively and discard meta for removed/unmatched rows; restore only touched entries whose keys survive. Observers never see draft discriminators with baseline children or a partially restored schema.
5. **Paths that no longer exist** in the contract are dropped with `W_SNAPSHOT_PATH_DROPPED`. Values that exist but are now invalid (an option was removed, a rule changed) are kept and simply fail validation.
6. **Stale drafts.** For each path `p`, with `base` = `snapshot.base[p]`, `external` = the baseline now, and `local` = the draft value, the [§7.1](#71-conflict-detection) table decides, in its order: with no draft value there is nothing to apply; if `local` equals `external`, the path is dropped from `current` with no conflict (even when `base` also equals it); else if `base` equals `external`, the draft applies. Otherwise (a real conflict) `onStale` decides:
   - `conflict` (default): the draft value is applied and flagged as a conflict (§7.2). Submit is blocked until resolved (`submission.conflicts`).
   - `keep-draft`: the draft value wins silently.
   - `drop`: the draft value is discarded for that path.
7. **Cascade invalidation** ([13 §4](./13-option-sources-and-cascading.md#4-invalidation)) compares each retained selection's saved `optionDeps` tuple with its effective dependency tuple in the **planned resolved result**, after stale decisions and computed/default resolution. Invalidate that selection only when those inputs differ; a baseline change or dropped override alone is not evidence of changed inputs. Thus retaining draft country `US` also retains a city saved for `US`, even if the external country changed; an unedited country that now resolves differently invalidates the city because its saved value was captured too. A removed/unknown dependency tuple is conservatively changed for a retained selection. Process in dependency order, recomputing downstream tuples after each cascade write. Include synchronous changes in the final commit's tick and generation-check subsequent async option work normally. Current option membership/validation still runs even when a tuple is unchanged. Notifications occur after the complete structural/value tick.

### 8.3 Notes

- **Dirty vs. draft status are different things.** Field indicators show "changed from defaults / saved data" and survive any number of draft saves. "Changed since the last draft save" is host state: compare `engine.revision` with the `revision` of the last snapshot that was stored successfully ([16 §1](./16-cookbook.md#1-autosave-and-drafts)). The engine doesn't track saves, because only the host knows whether one succeeded. Any commit that changes `current`, `rows` or `stash` bumps it (including undo, reset, reload and restore), so a label may say "unsaved" when the values happen to match the draft again; saving again is harmless.
- **Submitting** still uses `serialize()`. Delete a submitted draft only after a reconciled success with `pendingChanges: false` and no fresh revision difference from `savedRevision` (§4); retain it and the editor otherwise.
- A snapshot contains user data. Where and how it's stored (and encrypted) is the host's choice.
- Snapshots also cover crash recovery, moving an in-progress form between tabs or devices, and test fixtures.

## 9. Server rendering

The engine runs on the server, renders once, and the client continues from the exported state instead of loading again.

```ts
// server; contract = (await loadContract(json, { registry })).schema, checked ok (18 §1)
const engine = createFormEngine({ contract, registry, params, server: true });
await engine.ready;                              // loads, dynamic sub-schemas, mounted option fetches
const html = render(engine);
const state = engine.exportState();              // JSON; embed in the page (escape for <script>)

// client
const engine = createFormEngine({ contract, registry, params, resume: state });
```

```ts
interface EngineState {
  version: 1;
  formId: string;
  contractHash: string;
  locale: LocaleTag;
  revision: number;
  layers: { default: Json; loaded: Json; current: Json };
  rows: Record<DataPath, RowKey[]>;
  baselineRows: Record<RowKeyPath, BaselineRowKeys>; // baseline associations and unresolved provenance
  meta: { touched: RowKeyPath[]; visited: RowKeyPath[]; conflict: Record<RowKeyPath, Conflict>; stash?: Json };
  notices?: Record<RowKeyPath, Notice[]>;    // external and settled async messages only
  options: Record<string, Json>;                 // option cache entries by cache key (08 §3)
  subSchemas: Record<DataPath, { discriminator: Json; hash: string; nodes?: InputNode[] }>;   // loaded dynamic sub-schemas (08 §5)
  localeBundles?: Record<LocaleTag, LocaleBundle>;   // bundles loaded from the LocaleSource (10 §3.3)
  plugins?: Record<string, Json>;                // PluginInstance.exportState (03 §8.1)
}
```

**Resume.** `resume` is untrusted input: its shape is validated first, then `formId` and `contractHash` must match the contract; otherwise `createFormEngine` throws `E_RESUME_MISMATCH` (catch it and start without `resume`). On success:
- the entity load is skipped, and `layers`, `rows`, `baselineRows`, `meta` and `revision` are installed as one commit with origin `resume` ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)): no inbound mapping, no effects, no conflict detection, `history: 'clear'`; baseline row maps are validated against the exported layers, dirty bits and counts are rebuilt, and `revision` is set to the exported value;
- dynamic sub-schemas in `subSchemas` are compiled from their `nodes` instead of calling the `SchemaSource`. They take the same path as fetched ones (the detect size check and steps 3–7, the engine's `limits`, `allow` and `strict`, node counting; [08 §5](./08-async-data-and-registries.md#5-dynamic-sub-forms)), and the engine recomputes their hash. A sub-schema that fails a check or whose hash doesn't match is dropped and fetched again (`W_RESUME_SCHEMA_REJECTED`); the rest of the form still resumes; entries exported with `exportState({ subSchemas: 'ref' })` carry no `nodes` and are fetched again, and fetched nodes whose `hash` differs are used but logged with `W_RESUME_SCHEMA_CHANGED`;
- option fetches whose cache entry is included are skipped; the rest run as usual;
- the engine starts in the exported `locale`, even if `i18n.locale` differs, so the first render matches the server; if it differs, `setLocale(i18n.locale)` runs automatically after `ready` (the normal atomic switch, [10 §3.3](./10-localization.md#33-loading-bundles-separately)). Pick the locale on the server (cookie, header) to avoid the brief switch;
- `localeBundles` are installed before the first render (runtime checks from [10 §3.4](./10-localization.md#34-rules) run as on arrival), so the `LocaleSource` isn't called for those tags and the first client render shows the same text as the server;
- sync validation runs again (cheap, and it doesn't trust the payload); async validation is not rerun and the exported messages stand until the next trigger, but fields with async rules are marked *unvalidated* (not valid), so `submit()` runs them;
- plugin state: after the core commit, each plugin's `resumeState(state.plugins[name])` runs in plugin order, before `ready`. A key with no installed plugin is ignored with `W_RESUME_PLUGIN_UNKNOWN` (dev); an installed plugin with no key, or whose `resumeState` rejects its input, starts from its default state ([03 §8.1](./03-state-model.md#81-plugin-api));
- `ready` resolves once the remaining sub-schemas are in.

Server output and the client's first render must match: element ids are derived from row keys ([11 §8](./11-rendering-and-adapters.md#8-accessibility)), and `rows` carries the keys over.

**Server-safe mode** is explicit: `createFormEngine({ server: true })`; it defaults to `false`. Server adapters must opt in. Absence of `window` is never used to infer it: React Native and workers can run fully interactive forms without a DOM.
- no timers: debounce is disabled and async validation isn't started;
- asset stores and `applyExternal` aren't used; `submit()` rejects with `E_SERVER_UNSUPPORTED`;
- `engine.ready` resolves after loads, sub-schemas and option fetches for fields read during setup; `dispose()` after rendering.

Adapters render from the same snapshot on both sides (`getServerSnapshot`, [11 §3](./11-rendering-and-adapters.md#3-framework-bindings)).
