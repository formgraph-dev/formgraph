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

After a successful submit, the engine reloads with the target's response (if it returns one) using `replace`, regardless of `reloadPolicy`, which resets dirty state.

## 4. Submission pipeline

```
engine.submit():                  // submit({ confirm?: string[] }): Promise<SubmitResult>
  if a submit is in flight → return its promise (no second run, no error)
  submitCount++ ; phase=submitting
  acquire edit lock (§4 below); clear lastSubmitError
  await schema loads for every eligible dynamic group; failed/mismatched → fail('schema')
  force-compile validators and option state for eligible fields (03 §6.3)
  start option loads that gate validity, and async rules with no settled result for the current value
  run all sync validations (trigger=submit) on fields eligible per validateHidden/validateDisabled
  if awaitAsync: await in-flight async validations and busy field-type work (assets, 01 §3.4)
  await pending option loads that gate validity (13 §4.2 pendingValidity)
  if any severity=error (including sources 'input' and 'component', 11 §2.1–2.2)
     or any field-type pending work failed (assets; 01 §3.4)   → fail('errors')
  if async validation still in flight (awaitAsync: false)       → fail('validating')
  if field-type work still busy (awaitAsync: false)             → fail('assets-pending')
  if conflicts and submission.conflicts=block                   → fail('conflicts', { paths })
  confirm(opts.confirm) ; gated = gates with notices whose policy is block,
                            or confirm and not confirmed (06 §7.3)
  if gated non-empty                                            → fail('gated', { gates })
  plugin 'before' hooks (§4.1)                                  → fail(hook's reason)
  payload = serialize()                   // outbound transformer throws → fail('transform') (08 §1.1)
  result  = await submitTargets[targetId]({ payload, mode, confirmed }, ctx)   // AsyncSource, 08 §1.1
            // throw/reject → lastSubmitError = error; fail('target')
  if result.errors → map external errors (§5)                   → fail('external')
  if result.entity: reload({ entity: result.entity, policy: 'replace' })
  phase=submitted
  release edit lock
  apply queued applyExternal changes (§7)
  return {ok:true}

fail(reason, extra?):
  phase=ready; release edit lock
  apply queued applyExternal changes (§7)
  r = {ok:false, reason, ...extra}
  run plugin failure observers, then onSubmitFailed(r) (11 §8); return r
```

```ts
type SubmitResult =
  | { ok: true }
  | { ok: false; reason: 'errors' | 'external' | 'transform' | 'validating' | 'assets-pending' | 'schema' | 'target' }
  | { ok: false; reason: 'conflicts'; paths: DataPath[] }
  | { ok: false; reason: 'gated'; gates: GateStop[] }
  | { ok: false; reason: `x-${string}`; notices?: NoticeInput[] };   // a plugin's `before` hook stopped the submit (§4.1)

type GateStop = { gate: string; policy: 'confirm' | 'block'; notices: Array<Notice & { path: DataPath }> };   // 06 §7.3
```

**Edit lock.** From entry into `submitting` through response application, user/host/plugin value or row-structure writes (`setValue`, `revert`, `reset`, array commands, history replay and host `reload`) are rejected with `E_SUBMIT_BUSY`; `restore` retains `E_RESTORE_BUSY`. Renderers use the form phase to prevent editing without changing `disabled` or submission eligibility. The lock is not just a UI convention: every public/plugin mutation entry checks it. `applyExternal` queues changes instead. Internal schema loads, option reconciliation, computed/default evaluation and validation needed to settle this submission remain allowed and finish before serialization; after payload capture, value-changing async completions are deferred until finalization. The submit's own response reload is internal and allowed. Failure, success and disposal always release the lock; deferred work is generation-checked before applying. Thus a response `replace` cannot silently discard a newer edit. Hosts wanting uninterrupted editing should use a separate editing instance rather than bypass this lock.

**Target failure.** A thrown/rejected target records the original error in `FormState.lastSubmitError` and reports a runtime `E_SUBMIT_TARGET`. It returns `{ ok: false, reason: 'target' }`, keeps local edits, restores `ready`, unlocks, drains external/deferred updates, then runs failure hooks and `onSubmitFailed`, in that order. The host may offer retry; the engine never automatically retries a potentially non-idempotent target. All other failure branches also drain the queue. Success still applies any response entity first, then unlocks and drains before success observers. Finalization is shared so cleanup cannot be skipped by a failure branch.

#### 4.1 Plugin submit hooks

Plugins observe the pipeline with `ctx.submission.on(event, fn)` ([03 §8.1](./03-state-model.md#81-plugin-api)); autosave, audit and analytics plugins use it.

```ts
type SubmitStop = { stop: `x-${string}`; notices?: NoticeInput[] };
type SubmitHook = (e: { mode: SubmitMode; confirmed: string[]; result?: SubmitResult; signal: AbortSignal }) => void | SubmitStop | Promise<void | SubmitStop>;
```

- `before` runs after validation and gates pass, before `serialize()`. It may be async (awaited, aborted with the submit's `signal`) and may return `{ stop: 'x-…' }`, which fails the submit with that reason (`notices` become form-level notices). It's for programmatic stops; asking the user to confirm is a gate ([06 §7.3](./06-validation.md#73-gates-submit-policy-for-warnings)).
- `success` and `failure` run after the result is known, observe only (a return value is ignored), and can't change it. `failure` hooks run **before** the host's `onSubmitFailed`.
- Hooks run in plugin order. A thrown hook is reported through `onDiagnostic` and doesn't change the result, except that a throwing `before` stops the submit with `reason: 'x-<plugin>'`.

`onSubmitFailed` runs for **every** `{ ok: false }` result; the adapters' default branches on `reason` ([11 §8](./11-rendering-and-adapters.md#8-accessibility)).

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
5. Run the outbound transformer if one is configured. It receives `({ payload, mode, loaded, values, dirtyPaths }, ctx)` (`OutboundTransformer`, [08 §1.1](./08-async-data-and-registries.md#11-shared-function-shapes)): the payload so far, the saved data as loaded (after the inbound mapping), the full resolved values, and the list of changed paths. That's enough to compute row-level diffs ([16 §17](./16-cookbook.md#17-row-level-diffs-for-patch-submissions)).

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
- During `submit()`, external changes are queued and applied after it finishes. If the submit reloaded from a response entity, each queued change is first compared with the new `loaded` value at its path; changes already reflected there are dropped.

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
  base: Record<DataPath, Json | null>;      // baseline of each stored leaf of `current` when taken
  stash?: Json;                             // dynamic-group orphans kept by orphanPolicy 'stash' (08 §5)
  touched?: DataPath[];                     // row key paths; restored so messages show as they did
  takenAt: string;                          // ISO timestamp
}

interface RestoreResult { applied: DataPath[]; conflicts: DataPath[]; dropped: DataPath[] }
```

**Granularity.** The paths of a snapshot (`base` keys, `RestoreResult` lists, stale checks) are the **stored leaves** of `current`: each array counts as one entry ([03 §3.1](./03-state-model.md#31-resolution-granularity)). `touched` is mapped through the restored row keys; entries under rows that no longer match are dropped.

**Field-type filter.** `snapshot()` stores each value through its field type's `toSnapshot` hook ([01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions)). For assets (`assets` plugin) only values with status `ready` are kept. Others (pending, uploading, error) are left out with `W_SNAPSHOT_ASSET_DROPPED`, since their uploads don't survive the session.

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
3. **Dynamic groups first.** Discriminators are ordinary paths in `current`. They are written first, the sub-schema loads they trigger are awaited, and then the rest is applied, so every restored path has its node.
4. **Writes with origin `restore`** ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)): first plan the surviving paths and stale decisions (rules 5–6), then replace `current` **atomically** (not merged), with no effects (origin policy `history: 'clear'`). Reconcile row keys against the arrays that actually survive: an applied draft array reuses its validated draft keys; an array dropped from `current` keeps the current baseline's keys, matching by `identityKey` where available and allocating fresh keys otherwise. Never install draft keys on a different baseline array. Reconcile nested arrays recursively and discard meta for removed/unmatched rows; restore `stash` and only `touched` entries whose row keys survive. Conflicts are cleared, then the conflicts planned by rule 6 are installed in the same commit.
5. **Paths that no longer exist** in the contract are dropped with `W_SNAPSHOT_PATH_DROPPED`. Values that exist but are now invalid (an option was removed, a rule changed) are kept and simply fail validation.
6. **Stale drafts.** For each path `p`, with `base` = `snapshot.base[p]`, `external` = the baseline now, and `local` = the draft value, the [§7.1](#71-conflict-detection) table decides: if `base` equals `external`, or there's no draft value, the draft applies; if `local` equals `external`, the path is dropped from `current` with no conflict. Otherwise (a real conflict) `onStale` decides:
   - `conflict` (default): the draft value is applied and flagged as a conflict (§7.2). Submit is blocked until resolved (`submission.conflicts`).
   - `keep-draft`: the draft value wins silently.
   - `drop`: the draft value is discarded for that path.
7. **Cascade invalidation** ([13 §4](./13-option-sources-and-cascading.md#4-invalidation)) then runs for paths whose baseline changed since the snapshot or that were dropped, since their dependents were saved against other values. Values that restored unchanged aren't invalidated.

### 8.3 Notes

- **Dirty vs. draft status are different things.** Field indicators show "changed from defaults / saved data" and survive any number of draft saves. "Changed since the last draft save" is host state: compare `engine.revision` with the `revision` of the last snapshot that was stored successfully ([16 §1](./16-cookbook.md#1-autosave-and-drafts)). The engine doesn't track saves, because only the host knows whether one succeeded. Any commit that changes `current`, `rows` or `stash` bumps it (including undo, reset, reload and restore), so a label may say "unsaved" when the values happen to match the draft again; saving again is harmless.
- **Submitting** still uses `serialize()`. After a successful submit, the host deletes the draft.
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
  meta: { touched: RowKeyPath[]; visited: RowKeyPath[]; conflict: Record<DataPath, Conflict>; stash?: Json };
  notices?: Record<RowKeyPath, Notice[]>;    // external and settled async messages only
  options: Record<string, Json>;                 // option cache entries by cache key (08 §3)
  subSchemas: Record<DataPath, { discriminator: Json; hash: string; nodes?: InputNode[] }>;   // loaded dynamic sub-schemas (08 §5)
  localeBundles?: Record<LocaleTag, LocaleBundle>;   // bundles loaded from the LocaleSource (10 §3.3)
  plugins?: Record<string, Json>;                // PluginInstance.exportState (03 §8.1)
}
```

**Resume.** `resume` is untrusted input: its shape is validated first, then `formId` and `contractHash` must match the contract; otherwise `createFormEngine` throws `E_RESUME_MISMATCH` (catch it and start without `resume`). On success:
- the entity load is skipped, and `layers`, `rows`, `meta` and `revision` are installed as one commit with origin `resume` ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)): no inbound mapping, no effects, no conflict detection, `history: 'clear'`; dirty bits and counts are rebuilt from the layers, and `revision` is set to the exported value;
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
