# 16 · Cookbook

[← Index](./README.md)

Patterns built only from core primitives. Several have a macro shortcut in `@formgraph/macros` ([15 §7](./15-macros.md#7-standard-library-formgraphmacros-optional)). The long form here is what the macro expands to.

## 1. Autosave and drafts

Drafts save a **snapshot** (the user's edits plus what they were based on), not the submit payload, so reopening shows the same values **and** the same change indicators ([09 §8](./09-loading-and-submission.md#8-snapshots-drafts)):

```ts
// open
const engine = await createFormEngineFrom(contractJson, { registry });   // load + create (18 §2)
const draft = await drafts.get(formKey);
if (draft) {
  const { conflicts } = await engine.restore(draft);
  if (conflicts.length) showBanner('This record changed since your draft. Review the highlighted fields.');
}

// autosave: one save at a time; edits during a save queue exactly one follow-up
let saving: Promise<void> | null = null;
let savedRevision = engine.revision.get();     // after restore: the restored state is what's stored
let again = false;
let stopped = false;
async function save() {
  if (stopped) return;
  if (saving) { again = true; return; }
  do {
    again = false;
    const snap = engine.snapshot();
    setStatus('saving');
    saving = drafts.put(formKey, snap);
    try { await saving; savedRevision = snap.revision; setStatus('saved'); }   // only once stored
    catch { setStatus('error'); }
    finally { saving = null; }
  } while (again && !stopped);                  // latest state wins; no follow-up after stopping
}
const debouncedSave = debounce(save, 1000);
const offRevision = engine.revision.subscribe((rev) => {
  setUnsaved(rev !== savedRevision);         // "Unsaved changes" / "Draft saved"
  if (rev !== savedRevision) debouncedSave();
});
// submit
const result = await engine.submit();
if (result.ok) {
  if (result.response === 'failed' || result.pendingChanges || engine.revision.get() !== result.savedRevision) {
    showBanner(result.response === 'failed' ? 'Saved, refresh required.' : 'Saved; unsent changes remain.');
    debouncedSave();                           // retain editor, subscription and draft for remaining/reconciliation work
  } else {
    stopped = true; again = false;
    offRevision(); debouncedSave.cancel();      // no unsubmitted revision remains; exit this editing flow
    engine.dispose();
    await saving?.catch(() => {});             // an older put must settle before deletion
    await drafts.delete(formKey);              // no late put can recreate the submitted draft
  }
}
```

`revision` (core, [03 §7](./03-state-model.md#7-engine-write-api-core)) bumps on any commit that changes `current`, `rows` or `stash`, whatever its origin (user edits, host writes, effects, undo, reset, reload), because all of them change what the draft must contain. `savedRevision` only moves when a `put` succeeds, so a failed save leaves the form marked unsaved. The save status lives in host state. A step-based draft saves after `if ((await wiz.next()).ok)` instead ([11 §5](./11-rendering-and-adapters.md#5-wizards)).

Only a reconciled success with no unsent work closes/unmounts the editor. `pendingChanges` includes serializer-excluded edits retained from before submit as well as deferred resolved/default/computed changes; a fresh comparison against `savedRevision` catches subsequent revision writes. Remaining work keeps the live editor and autosave subscription. For `response: 'failed'` (a later `submit()` returns `reason: 'reconcile-pending'` until this succeeds), retain the result and retry `engine.reconcileSubmit(result.reconciliationId)` (optionally with a refetched entity), not submit or plain preserve-edits reload: reconciliation clears the already-acknowledged snapshot and preserves only excluded/later edits. Recheck its returned pending-work/revision fields before closing. To keep editing after a clean save, start a fresh engine/autosave session after draft deletion or use a pause/reattach flow. Do not reuse a disposed instance.

## 2. Form-level variables

A hidden computed field. With `whenHidden: 'exclude'` set on the node (the form default is `keep`), it isn't submitted, validated, or rendered. Other expressions read it like any other path.

```json
{ "id": "riskScore", "kind": "field", "fieldType": "number",
  "conditions": { "visible": false }, "whenHidden": "exclude",
  "computed": { "expr": { "lang": "json-logic", "expr": { "+": [{ "var": "/age" }, { "*": [{ "var": "/claims" }, 10] }] } } } }
```

Macro: `variable`.

## 3. "Other" option with free text

```json
{ "id": "source", "kind": "field", "fieldType": "choice",
  "options": [{ "value": "search", "label": "Search" }, { "value": "other", "label": "Other" }] },
{ "id": "sourceOther", "kind": "field", "fieldType": "text", "label": "Please specify",
  "whenHidden": "clear",
  "conditions": {
    "visible":  { "lang": "json-logic", "expr": { "==": [{ "var": "./source" }, "other"] } },
    "required": { "lang": "json-logic", "expr": { "==": [{ "var": "./source" }, "other"] } } } }
```

Macro: `choice-with-other`.

## 4. Exclusive "None" option

```json
{ "id": "allergies", "kind": "field", "fieldType": "multi-choice",
  "effects": [{ "target": "./allergies", "action": "set", "value": ["none"],
    "when": { "lang": "json-logic", "expr": { "and": [
      { "in": ["none", { "var": "./allergies" }] },
      { ">": [{ "count": { "var": "./allergies" } }, 1] }] } } }] }
```

`count` is one of the built-in aggregate functions ([05 §2](./05-expressions.md#2-evaluator-registry)).

Picking "None" alongside other values collapses the selection to `["none"]`. To do the reverse (picking another value drops "None"), use a second effect that removes it. An effect may target its own field. Such effects are exempt from the compile-time cycle check, but limited by `limits.effectDepth` ([04 §6](./04-paths-and-dependency-graph.md#6-scheduler), [04 §5.2](./04-paths-and-dependency-graph.md#52-cycle-detection)). Macro: `multi-choice-with-none`.

## 5. Carry-forward options

```json
{ "id": "favourite", "kind": "field", "fieldType": "choice",
  "options": { "source": "static", "items": [ … ],
    "filter": { "lang": "json-logic", "expr": { "in": [{ "var": "$option.value" }, { "var": "./interests" }] } } } }
```

The `items` are the same as `./interests`. Repeating the list is the authoring cost. Macro: `carry-forward`, which copies the items from the source field during expansion.

## 6. Reset a field when another changes

This is an effect on the **source** field ([07 §2](./07-effects-and-computed.md#2-write-origins-and-effect-triggers)):

```json
{ "id": "plan", "kind": "field", "fieldType": "choice",
  "effects": [{ "target": "./addons", "action": "reset" }] }
```

Macro: `reset-when` lets you declare it on the target instead.

## 7. Draft rows (edit, then save or cancel)

Edit the row in a **separate engine instance** seeded with the row's value, then write it back:

```ts
// host code: buildRowContract wraps the array's `item` node in a FormContract with the same
// formId suffix and schemaVersion, and no serializeAs / inbound transformer, so the row's
// form values load unchanged.
const rowContract = buildRowContract(contract, '/addresses');
const rows = engine.array('/addresses');
const key = rows.keyAt(index);                  // capture identity and value together, before any await
if (!key) throw new Error('Row no longer exists');
const rowValue = engine.getValues(`addresses[${index}]` as DataPath);
const loadedRow = await loadContract(rowContract, { registry });
if (!loadedRow.ok) throw new ContractLoadError(loadedRow.diagnostics);
const rowEngine = createFormEngine({
  contract: loadedRow.schema, registry,
  context: { parent: engine.getValues() },   // whatever the row's expressions read as $context.parent
});
try {
  await rowEngine.ready;
  await rowEngine.reload({ entity: rowValue });  // row value becomes `loaded`, so dirty works
  // … show editor and await the user's save decision …
  if ((await rowEngine.validate()).ok) {
    const currentIndex = rows.rows.get().indexOf(key);
    if (currentIndex < 0) throw new Error('Row was removed while editing');
    const currentRow = engine.getValues(`addresses[${currentIndex}]` as DataPath);
    if (!deepEqual(currentRow, rowValue)) throw new Error('Row changed while editing: reopen or merge before saving');
    rows.set(key, rowEngine.getValues());        // same row after reorders; values, not serialized payload
  }
} finally { rowEngine.dispose(); }
// on cancel: rowEngine.dispose()
```

**Limitation:** `../` and `/` references inside the row can't reach the parent form. Pass the parent values the row needs through `context` (and refer to them as `$context.parent.x`), or use inline editing.

`deepEqual` is a host equality helper (for example `dequal`, as in §17). A changed row is an explicit conflict: this example refuses to overwrite it; the host must reopen the editor on the new row or merge and validate the result. Identity lookup, comparison and `rows.set` have no intervening await.

## 8. View mode / read-only review page

Put a `readOnly` condition on the root group driven by host context. `readOnly` is inherited ([01 §4](./01-schema-contract.md#4-layout-vs-conditions)):

```json
{ "id": "root", "kind": "group", "ui": { "variant": "none" },
  "conditions": { "readOnly": { "lang": "json-logic", "expr": { "==": [{ "var": "$context.mode" }, "view"] } } }, "children": [ … ] }
```

How read-only fields look (plain text, formatted values) is up to the renderer.

## 9. One page, steps, or one question at a time

This is purely a rendering choice. Register different group renderers by platform or mode using ranked testers ([11 §4.1](./11-rendering-and-adapters.md#41-ranked-testers)), for example rendering `step: true` groups as plain sections when `ctx.platform === 'desktop-single-page'`.

## 10. Delayed error display, debounced effects, tabs, row labels

- **Delayed errors:** a timer in the adapter around the `showFeedback` handle.
- **Debounced effects:** keep the engine value current and debounce only downstream work (analytics, autosave, host requests). If a renderer must delay committing text, it **flushes pending writes before `submit()`, wizard navigation and unmount**, because Enter can submit before blur or a timer, and a timer firing during the submit lock gets `E_SUBMIT_BUSY`. Otherwise commit on blur in the renderer, or debounce in host code and write with `engine.setValue(path, value, { origin: 'x-…' })` using a registered custom origin ([07 §2.1](./07-effects-and-computed.md#21-origin-policy)). Effect handlers are synchronous and write only inside the tick, so they cannot debounce.
- **Tabs:** a parent group with `ui: { variant: 'tabs' }` whose child groups are the tabs. Use an `x-` variant plus a registered renderer for custom tab styles.
- **Row labels:** `label: { "default": "{name}", "params": { "name": { "lang": "json-logic", "expr": { "var": "./name" } } } }` on the row group.

## 11. Multiple values for one field

```json
{ "id": "emails", "kind": "array", "item": { "id": "email", "kind": "field", "fieldType": "text",
  "validations": [{ "type": "email" }] } }
```

Macro: `list`.

## 12. Reusable sub-contracts

Put contracts together **before** loading (in the CMS, a build step, or the host's contract fetcher), and pin versions there. The engine receives a single, complete contract. Within one document, share definitions with a `$defs`-style section and resolve it in the same pre-load step. The engine itself doesn't accept `$defs` (it would be `E_UNKNOWN_PROPERTY`), so remove it after resolving, or keep it under an `x-defs` key.

## 13. Fields that aren't submitted

- Hidden helper fields: `whenHidden: 'exclude'` (see §2).
- Visible but not submitted (for example "confirm email"): remove it in the outbound transformer (`submission.transformerId`, [09 §1](./09-loading-and-submission.md#1-config)).

## 14. Submit while invalid

Skip `submit()`: `await api.save(engine.serialize({ validate: false }))` (the host's API call, not the registered `SubmitTarget` protocol; the host then owns acknowledgement/draft cleanup). Or mark rules `severity: 'warning'` with the `warnings` gate set to `allow` (the default, `submission.gates`) so they don't block ([06 §7](./06-validation.md#7-severity)).

## 15. Externally derived inputs via `setContext`

For values that have to come from outside the form (an API, storage, a device) and depend on form values. The engine stays synchronous: the host loads it, then pushes the result into `context` ([05 §5.2](./05-expressions.md#52-async-data-where-it-goes-instead)).

```ts
let call: AbortController | null = null;
engine.signal.addEventListener('abort', () => call?.abort());   // once: aborted on engine.dispose(); works on React Native
const country = engine.field('/country').value;          // Readable (11 §2)
const onCountry = async (country: Json) => {
  call?.abort();                                          // cancel the previous request
  call = new AbortController();
  const signal = call.signal;
  engine.setContext({ fx: null });                       // explicit "unknown", never a stale rate
  if (country == null) return;
  try {
    const fx = await loadFx(country, signal);             // host code: fetch, TanStack Query, …; owns cache and retry
    if (!signal.aborted) engine.setContext({ fx });       // drop superseded responses
  } catch {
    if (!signal.aborted) engine.setContext({ fx: { error: true } });
  }
};
country.subscribe(onCountry);                             // dispose() drops it
onCountry(country.get());                                 // subscribe doesn't fire for the initial value
```

Guard everything that depends on it, so "unknown" is visible rather than wrong:

```json
{ "id": "totalLocal", "kind": "field", "fieldType": "number",
  "computed": { "expr": { "lang": "json-logic", "expr": { "if": [{ "var": "$context.fx.rate" },
      { "*": [{ "var": "./total" }, { "var": "$context.fx.rate" }] }, null] } } },
  "conditions": { "visible": { "lang": "json-logic", "expr": { "!!": [{ "var": "$context.fx.rate" }] } } } }
```

Notes:
- `Readable.subscribe` fires on any value change, including loading and reset, so this works in edit mode. It doesn't fire for the initial value, hence the explicit first call.
- `context` changes start a normal tick. They never dirty the form and are never submitted. If a submitted value has to wait for the result, guard `submit()` in the host, or add an `expr` validation on the dependent field that fails while `$context.fx` is `null`.
- `engine.signal` is aborted by `dispose()`, and writes to a disposed engine are no-ops, so a late response is harmless ([18 §3](./18-api-reference.md#3-formengine-core)).

## 16. Postcode → city autofill

Model the city as a select whose options come from the postcode. A single match fills itself in:

```json
{ "id": "city", "kind": "field", "fieldType": "choice",
  "options": { "source": "provider", "providerId": "geo.citiesByPostcode", "dependsOn": ["./postcode"] },
  "cascade": { "whenEmpty": "disable", "onChange": "keep-if-valid" },
  "autoSelectSingle": true }
```

This gets cancellation, out-of-order handling, validity checks, and "loading saved data doesn't clear it" from [13](./13-option-sources-and-cascading.md). If the postcode maps to several cities, the user picks one. If it maps to none, the list is empty, and `in-options` / `required` report it. Have the query transformer return `null` while the postcode is too short. That skips the request ([08 §2](./08-async-data-and-registries.md#2-provider-options)) and avoids lookups on partial input.

## 17. Row-level diffs for patch submissions

`patch` mode sends changed arrays whole ([09 §4](./09-loading-and-submission.md#4-submission-pipeline)). If the submit target wants row operations, compute them in the outbound transformer. You need a row identity field (the same one used for `identityKey`, [04 §4](./04-paths-and-dependency-graph.md#4-array-row-identity)):

```ts
import { dequal as deepEqual } from 'dequal';

registry.outboundTransformers.register('orders.patch', ({ payload, mode, loaded, values }) => {
  if (mode !== 'patch' || !('lines' in payload)) return payload;   // only patch, only when lines changed
  const before = new Map((loaded.lines ?? []).map((r) => [r.id, r]));
  const after = values.lines ?? [];
  return {
    ...payload,
    lines: {
      added:   after.filter((r) => r.id == null || !before.has(r.id)),
      changed: after.filter((r) => r.id != null && before.has(r.id) && !deepEqual(r, before.get(r.id))),
      removed: [...before.keys()].filter((id) => !after.some((r) => r.id === id)),
      order:   after.map((r) => r.id ?? null),
    },
  };
});
```

**Two shapes.** `payload` is the wire shape (after `serializeAs` and hidden/excluded fields are dropped, [09 §4](./09-loading-and-submission.md#4-submission-pipeline)); `values` and `loaded` are the form shape. This version is safe when `lines` has no `serializeAs` and its rows have no excluded fields. Otherwise read the rows from `payload.lines` and map the `loaded` rows the same way before comparing, or a row can look changed because of a field that is never sent.

The target's contract for this shape is yours to define. The engine stays out of it.

## 18. Order totals

Use the built-in `count`/`sum` operators ([05 §2](./05-expressions.md#2-evaluator-registry)) and combine them with ordinary expressions:

```json
{ "kind": "field", "id": "lineCount", "fieldType": "number", "computed": { "expr": { "lang": "json-logic", "expr": { "count": { "var": "/items" } } } } },
{ "kind": "field", "id": "subtotal",  "fieldType": "number", "computed": { "expr": { "lang": "json-logic", "expr": { "sum": { "var": "/items[*].amount" } } } } },
{ "kind": "field", "id": "total",     "fieldType": "number",
  "computed": { "expr": { "lang": "json-logic", "expr": { "*": [{ "var": "/subtotal" }, { "+": [1, { "var": "$context.taxRate" }] }] } } } }
```

`amount` itself is a per-row computed (`qty * price`), so a keystroke in one row costs one row computation plus O(rows) for `subtotal` ([07 §3.2](./07-effects-and-computed.md#32-aggregates)).

## 19. Server-rendered form

```ts
// server
const engine = await createFormEngineFrom(contractJson, { registry, params, server: true });
await engine.ready;
const html = renderToString(<FormProvider engine={engine}><FormRenderer renderers={renderers} /></FormProvider>);
send(html, { state: engine.exportState() });

// client; `contract` is the CompiledSchema loaded once (18 §1)
const app = (engine) => <FormProvider engine={engine}><FormRenderer renderers={renderers} /></FormProvider>;
try {
  const engine = createFormEngine({ contract, registry, params, resume: window.__FORM_STATE__ });
  hydrateRoot(container, app(engine));
} catch (e) {
  if (e.code !== 'E_RESUME_MISMATCH') throw e;
  // server HTML no longer matches: discard it and render client-only
  const engine = createFormEngine({ contract, registry, params });
  await engine.ready;
  createRoot(container).render(app(engine));
}
```

Same element ids on both sides come from row keys ([11 §8](./11-rendering-and-adapters.md#8-accessibility)); option lists, dynamic sub-schemas and locale bundles (`localeBundles`) aren't fetched again ([09 §9](./09-loading-and-submission.md#9-server-rendering)). Choose the locale on the server (cookie or `Accept-Language`): a client asking for another locale first renders the server's, then switches after `ready`.

## 20. Slim bundle

Size-sensitive apps that don't need undo, drafts or wizards import the core entry and opt in per feature ([03 §8](./03-state-model.md#8-plugins)):

```ts
import { createFormEngine, createRegistry } from '@formgraph/engine/core';
import { drafts } from '@formgraph/engine/plugins';
const registry = createRegistry().plugin(drafts);
const engine = createFormEngine({ contract, registry });
engine.history.undo();   // type error; at runtime (JS) throws E_PLUGIN_MISSING ('history')

type DraftEngine = FormEngineWith<typeof registry>;
function autosave(e: DraftEngine) { save(e.snapshot()); }
```

## 21. Custom control with a structured value

Wrapping a third-party control (here, an editable grid) whose value is an object, which has its own validation and its own async work. Uses [01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions) and [11 §2.2](./11-rendering-and-adapters.md#22-component-reported-messages-and-busy-state).

**1. Declare the type.** This tells the engine what the value looks like, what "empty" and "equal" mean, and how it's exported:

```ts
type GridValue = { columns: string[]; rows: Array<Record<string, number | string | null>> };

const registry = coreRegistry.extend().fieldType('x-grid', {   // typed: contracts see x-grid (21 §1)
  value:  v.object({ columns: v.array(v.string()), rows: v.array(v.record(v.string(), v.nullable(v.union([v.number(), v.string()])))) }),
  config: v.object({ maxRows: v.optional(v.number()) }),
  ui:     v.object({ dense: v.optional(v.boolean()) }),
  empty:  { columns: [], rows: [] },
  isEmpty: (g) => g.rows.length === 0,                    // `required` = at least one row, whatever the columns
  payloadSchema: { type: 'object', properties: { columns: { type: 'array' }, rows: { type: 'array' } } },
})
  // Engine-side rule on the whole value, pointing at cells with subPath (typed registration, so defineContract sees it)
  .validator('grid-qty-positive', {
    validate: (g: GridValue) => g.rows.flatMap((r, i) =>
      typeof r.qty === 'number' && r.qty <= 0 ? [{ subPath: `/rows/${i}/qty`, message: 'Must be positive' }] : []),
  });
```

```json
{ "kind": "field", "id": "lines", "fieldType": "x-grid", "label": "Lines", "maxRows": 200,
  "conditions": { "required": true },
  "validations": [{ "type": "grid-qty-positive" }] }
```

**2. Write the renderer.** Commit whole values, forward the library's findings, and hold the gate while it's busy:

```tsx
const Grid = ({ controller, node }: FieldRendererProps<'x-grid'>) => {
  useEffect(() => () => controller.setBusy('recalc', false), [controller]);
  // ThirdPartyGrid must cancel callbacks/work on unmount, so none can re-acquire this token.
  const { value, feedback, showFeedback } = useReadable(controller.state);
  const cellMessages = (sub: string) => showFeedback ? feedback.filter(m => m.subPath === sub) : [];

  return (
    <ThirdPartyGrid
      data={value}
      dense={node.ui?.dense}
      onChange={(next) => controller.setValue(next)}             // whole value; one atomic write, one history entry
      onValidated={(issues) => controller.report('x-grid', issues.length ? issues.map(i => ({
        message: i.text, severity: i.level === 'warn' ? 'warning' : i.level === 'hint' ? 'info' : 'error',
        subPath: `/rows/${i.row}/${i.col}`, code: i.code,
      })) : null)}
      onRecalcStart={() => controller.setBusy('recalc', true)}
      onRecalcEnd={() => controller.setBusy('recalc', false)}
      cellError={(row, col) => cellMessages(`/rows/${row}/${col}`)}
      onFocus={controller.handlers.onFocus} onBlur={controller.handlers.onBlur}
    />
  );
};

defineRenderers({ field: { 'x-grid': Grid } });
```

What you get:
- **Messages from three places, one list.** Engine rules (`grid-qty-positive`), the grid's own checks (`report`), and server errors such as `{ path: '/lines/rows/3/qty' }` (mapped to `subPath` `/rows/3/qty`, [09 §5](./09-loading-and-submission.md#5-external-error-mapping)) all land in `feedback`. Severity, submit blocking, `firstInvalid`, and `showFeedback` timing work the same for all of them. A generic renderer that ignores `subPath` still shows them at the field.
- **Busy gate.** Submit waits while `recalc` is busy, or returns `reason: 'validating'` without `awaitAsync`.
- **Dirty, undo, drafts, submit.** These work on the whole value. Dirty uses `deepEqual`, or `equals` if you define it.
- **Lifecycle.** `report` messages persist until the grid replaces them (the engine doesn't guess when they're stale), and they and busy tokens are dropped when the field is removed. Renderer cleanup also releases its busy tokens and cancels callbacks even if the controller remains. They aren't saved in drafts, so the grid should re-validate on mount.

**When not to use this.** If rows need their own conditions, effects, option sources, or per-cell controllers, use an `array` of groups with an `x-` array renderer (`ui.variant`) instead. The engine then knows each cell as a real path.

## 22. Another engine as a custom field

Embedding a whole, separately owned form (another formgraph contract, or a foreign form engine) as one field of the parent. It builds on §21: the child's output is the field's structured value, and its messages, busy state, and validation are bridged through the controller ([11 §2.2](./11-rendering-and-adapters.md#22-component-reported-messages-and-busy-state)).

**First, check whether you need a separate engine.** If the child is a formgraph contract and should behave as part of the parent (cross-field references, one undo history, one dependency graph), use a dynamic sub-form instead ([08 §5](./08-async-data-and-registries.md#5-dynamic-sub-forms)). Use a nested engine when you want **isolation**: a different registry, allow list, or contract version, a form owned by another team, or a different engine entirely. The parent then sees the child only as an opaque value.

**1. The field type.** The value is whatever the child submits:

```ts
const toPointer = (p: string) => '/' + p.replace(/\[(\d+)\]/g, '.$1').split('.').join('/');
const errorCode = (e: unknown) => e !== null && typeof e === 'object' && 'code' in e ? e.code : undefined;
type SubformBridge = { child: FormEngine | null; error?: string; ready?: Promise<void>; publish?: () => void };
const subformBridges = new WeakMap<FieldController, SubformBridge>();

// Install the fail-closed guard when the controller is created, before a renderer can mount.
// Engine-initiated validation creates eligible controllers even for unmounted fields.
const subformLifecycle = definePlugin('x-subform-lifecycle', {
  setup(ctx) {
    ctx.controllers.extend(c => {
      const bridge: SubformBridge = { child: null };
      subformBridges.set(c, bridge);
      const off = c.onValidate('subform', async ({ signal }) => {
        while (bridge.ready) {
          const ready = bridge.ready;
          await ready;                               // includes bridge installation, not only the child's load phase
          if (signal.aborted) return null;
          if (bridge.ready === ready) break;          // a newer reseed supersedes the awaited one
        }
        if (!bridge.child) return [{ message: bridge.error ?? 'Subform is not ready.' }];
        const child = bridge.child;
        const { notices } = await child.validate({ trigger: 'submit' });
        if (signal.aborted || bridge.child !== child) return null;   // obsolete validation cannot publish
        bridge.publish?.();                          // settle + synchronize while parent preflight is still unlocked
        return notices.length ? notices.map(n => ({
          message: n.message, severity: child.severityBase(n.severity), code: n.code, subPath: toPointer(n.path),
        })) : null;
      });
      return { members: {}, dispose() { off(); subformBridges.delete(c); } };
    }, { fieldTypes: ['x-subform'] });
  },
});

const registry = coreRegistry.extend().fieldType('x-subform', {   // opaque value: the child's payload type isn't known statically
  value: v.any(), config: v.object({ contractUrl: v.string() }), ui: v.object({}),
  empty: null,
  // payloadSchema: the child's toPayloadSchema(), if it's known when the parent is built
}).plugin(subformLifecycle);
```

**2. The renderer.** It owns the child engine and keeps six bridges in sync:

```tsx
const SubForm = ({ controller, node }: FieldRendererProps<'x-subform'>) => {
  const parent = useFormEngine();
  const [child, setChild] = useState<FormEngine | null>(null);
  const [initError, setInitError] = useState<string | null>(null);
  const [reloading, setReloading] = useState(false);
  const [reseedError, setReseedError] = useState<string | null>(null);
  const retryReseed = useRef<(() => void) | null>(null);
  const submitting = useReadable(parent.form.select('phase')) === 'submitting';

  // Create once per contract; seed from the parent's value; dispose on unmount
  useEffect(() => {
    const bridge = subformBridges.get(controller)!;
    bridge.child = null; bridge.error = undefined;
    setChild(null); setInitError(null); setReseedError(null);
    controller.setBusy('subform:init', true);
    controller.report('subform:init', null);
    let cancelled = false;
    const request = new AbortController();
    let engine: FormEngine | undefined;
    const offs: Array<() => void> = [];
    const abort = () => { request.abort(); engine?.dispose(); };
    parent.signal.addEventListener('abort', abort, { once: true });
    const stale = () => cancelled || parent.signal.aborted;
    const initialization = (async () => {
      try {
        const json = await fetchJson(node.contractUrl, { signal: request.signal });
        if (stale()) return;
        const e = await createFormEngineFrom(json, {
          registry: childRegistry,                     // isolated validators, sources and allow list
          plugins: { history: false },                 // parent's history covers undo
          context: hostContext,
          i18n: { locale: parent.form.get().locale },
        });
        engine = e;
        if (stale()) { e.dispose(); return; }
        await e.ready;
        if (stale()) { e.dispose(); return; }
        let seed: Json;
        do {                                         // parent may change while the initial reload is pending
          seed = controller.value.get();
          await e.reload({ entity: seed ?? {}, policy: 'replace' });
          if (stale()) { e.dispose(); return; }
        } while (!deepEqual(controller.value.get(), seed));
        let pushed: Json = seed;                      // the actual loaded value, never a newer unread parent value
        let deferredChild: Json | undefined;
        let reseedGeneration = 0;
        let reseeding = false;
        let reseedTarget: Json = seed;
        let publishing = false;
        const publishValue = (next: Json) => {
          if (stale()) return;
          if (deepEqual(next, controller.value.get())) { deferredChild = undefined; return; }
          const previous = pushed; pushed = next;     // suppress the echo only for this exact accepted write
          publishing = true;
          try { controller.setValue(next); deferredChild = undefined; }
          catch (error) {
            pushed = previous;
            if (errorCode(error) === 'E_SUBMIT_BUSY') deferredChild = next;   // after snapshot lock: subsequent edit
            else throw error;
          }
          finally { publishing = false; }
        };
        const reseed = (v: Json) => {
          const run = async () => {
            const generation = ++reseedGeneration;
            reseedTarget = v;
            reseeding = true; setReloading(true); setReseedError(null); bridge.child = null;
            try {
              await e.reload({ entity: v ?? {}, policy: 'replace' });
              if (stale() || generation !== reseedGeneration) return;
              pushed = v; bridge.child = e; bridge.error = undefined; setReseedError(null);
              controller.report('subform:reload', null);
            } catch (error) {
              if (stale() || generation !== reseedGeneration) return; // includes expected superseded older requests
              bridge.child = null; bridge.error = 'Subform could not be reloaded.';
              setReseedError(bridge.error);            // keep stale content disabled until a successful retry
              controller.report('subform:reload', [{ message: bridge.error }]);
            } finally {
              if (!stale() && generation === reseedGeneration) { reseeding = false; setReloading(false); }
            }
          };
          const ready = run();
          bridge.ready = ready;
          return ready;
        };
        retryReseed.current = () => { void reseed(controller.value.get()); };
        const syncBusy = () => controller.setBusy('subform:work', e.form.get().validating || e.form.get().phase === 'loading');
        offs.push(
          // 1. Value: child → parent. One parent write per child commit
          e.revision.subscribe(() => {
            if (stale() || reseeding || bridge.child !== e) return; // failed/replaying child cannot publish stale values
            publishValue(e.serialize({ mode: 'full' }));
          }),
          // 2. Value: parent → child, for reset, undo, reload and restore
          controller.value.subscribe((v) => {
            if (publishing && deepEqual(v, pushed)) return; // our accepted write, even before bridge.child is ready
            const expected = reseeding || bridge.child !== e ? reseedTarget : pushed;
            if (!deepEqual(v, expected)) void reseed(v); // A→B→A supersedes B even though A equals old pushed
          }),
          // 3. Busy: the child's async validation and loads hold the parent's gate
          e.form.subscribe(syncBusy),
          // 4. Validation is already guarded by subformLifecycle; publish after setup.
          // 5. Locale follows the parent
          parent.form.select('locale').subscribe((l) => { void e.setLocale(l); }),
          parent.form.select('phase').subscribe(phase => {
            if (phase !== 'submitting' && deferredChild !== undefined) {
              const next = deferredChild;
              publishValue(next);                     // lock released; preserve late child edits after response/reload
              void reseed(next);                     // response reseeding may have superseded the displayed child
            }
          }),
        );
        syncBusy();
        bridge.publish = () => { if (!reseeding) publishValue(e.serialize({ mode: 'full' })); };
        bridge.publish();                            // publish initialized defaults even when the parent started null
        bridge.child = e;                             // all bridges are installed before opening the parent gate
        controller.report('subform', null);           // clear any earlier not-ready finding from the guard
        controller.setBusy('subform:init', false);
        setChild(e);
      } catch (error) {
        offs.splice(0).forEach(off => off());
        if (!stale()) {
          bridge.child = null; bridge.error = 'Subform could not be initialized.';
          controller.report('subform:init', [{ message: bridge.error }]);
          setInitError(bridge.error);                 // fail closed, never an endless spinner without a notice
          controller.setBusy('subform:init', false);
        }
        engine?.dispose();
      }
    })();
    bridge.ready = initialization;
    return () => {
      cancelled = true;
      parent.signal.removeEventListener('abort', abort);
      abort(); offs.forEach(off => off());
      bridge.child = null; bridge.ready = undefined; bridge.publish = undefined; // validation guard closes again
      retryReseed.current = null;
      controller.setBusy('subform:init', false);
      controller.setBusy('subform:work', false);
    };
  }, [controller, parent, node.contractUrl, childRegistry, hostContext]);

  // 6. Server errors on the parent field (mapped to subPath, 09 §5) go into the child, next to the right input
  const { feedback } = useReadable(controller.state);
  useEffect(() => {
    const external = feedback.filter(m => m.source === 'external' && m.subPath);
    if (child && external.length) child.mapExternalErrors(external.map(m => ({
      path: m.subPath!, message: m.message, code: m.code, severity: parent.severityBase(m.severity) })));
  }, [child, feedback, parent]);

  if (initError) return <p role="alert">{initError}</p>; // host may remount to retry
  if (!child) return <Spinner />;
  return (
    <FormProvider engine={child}>
      {reseedError && <p role="alert">{reseedError} <button type="button" onClick={() => retryReseed.current?.()}>Retry</button></p>}
      <fieldset disabled={submitting || reloading || !!reseedError}><FormRenderer renderers={childRenderers} /></fieldset>
      {/* the parent's own messages for this field, but not the per-path ones the child already shows */}
      <FieldMessages messages={feedback.filter(m => !m.subPath)} />
    </FormProvider>
  );
};
```

What each bridge gives you:

| Concern | How | Result |
|---------|-----|--------|
| Value | child `revision` → `setValue(serialize())`; parent value → `child.reload` | Parent dirty, drafts, `exportState`, and submit see the child's output. Parent reset, undo, and draft restore re-seed the child |
| Validation | `onValidate` → `child.validate({ trigger: 'submit' })` | Parent `submit()`/`next()` blocks on child errors, including untouched child fields. Warnings and info keep their severity |
| Live messages | Shown by the child's own renderers | No duplicates: the parent renderer hides messages that have a `subPath` |
| Busy | `child.form.select('validating')` → `setBusy` | The parent waits for the child's async checks |
| Server errors | parent `subPath` → `child.mapExternalErrors` | `{ path: '/profile/address/zip' }` from the parent's submit target appears next to the child's zip input |
| Locale | `setLocale` follows the parent | One language switch for both |

Things to know:
- **Initialization.** The lifecycle plugin installs the validation guard before rendering (including when submit lazily creates the controller). Until every bridge is installed and the initialized child value is published, validation fails closed and the initialization token holds pending work. The seed/recheck loop catches parent changes during async setup. Failed creation reports an error; unmount closes the guard, aborts requests, releases tokens and disposes any engine that arrives late. Keep `childRegistry` and `hostContext` references stable. `deepEqual` is the host comparator from §17. Native renderers use a non-editable container while the parent submits/reseeds, rather than HTML `fieldset`.
- **Submit synchronization.** Parent preflight stays unlocked while `onValidate` settles the child and publishes its latest output. Any resulting parent value change restarts full parent validation before snapshot capture ([09 §4](./09-loading-and-submission.md#4-submission-pipeline)). A child completion after that lock is a subsequent edit: the listener catches `E_SUBMIT_BUSY`, buffers the value and publishes only after finalization unlocks, then reseeds if a response reload changed the child. It never throws a lock error from a subscription or mutates the captured payload. Applications may instead pause all child-originated work for the frozen-snapshot interval.
- **Severity boundary.** This isolated-registry example maps child notices through `child.severityBase`, preserving info/warning/error instead of upgrading unknown `x-` severities to errors. To preserve custom ranks or legal-gate semantics, share compatible registrations or provide an explicit child-to-parent severity mapping; never forward unknown severity names blindly.
- **Undo.** Each child commit is a parent write, so the parent's history records it (coalesced per `coalesceMs`). Undo restores the value and re-seeds the child through bridge 2. Leave history off the child to avoid two undo stacks.
- **Re-seeding resets the child's meta.** A `replace` reload after undo or reset clears the child's touched state. That's usually what you want. If you need to keep it, compute the diff and use `child.applyExternal` instead.
- **Reseed targets and failures.** Compare parent updates with the active target, not only the last published value, so A→B→A supersedes B. Ignore only stale/older generations; a failure of the current generation keeps the child disabled and its validation guard closed until Retry successfully reloads the latest parent value. The revision listener also refuses to publish from a failed/stale child, even if an internal callback runs while its UI is disabled.
- **No cross-references.** The parent sees the child's output only as one opaque value. Parent rules and functions can inspect it, but parent paths can't point inside it, and the child can't see the parent except through `context`. If you need more than that, use a dynamic sub-form.
- **Round-trip.** Bridge 2 seeds the child with what it serialized. That only works if `serializeAs` and outbound transformers can be loaded back. If excluded/hidden child input must also survive parent reload/undo/drafts, store `child.getValues()` (or its snapshot) instead and serialize at the parent's submit with an outbound transformer. The publishing guard suppresses the initial bridge's own echo; initializing from a payload is not permission to reload it and lose hidden child input.
- **Server rendering.** The child is created on the client after mount. For SSR, also render the child on the server and pass its `exportState()` next to the parent's ([16 §19](#19-server-rendered-form)).
- **Foreign engines.** The same six bridges work for any engine that exposes a value-change event, a validate call that returns located messages, and a busy flag. Map its paths to JSON Pointers for `subPath`.

## 23. Dirty indicator on every field

Mark changed fields ("●" next to the label) with a revert button, for every field type, without touching each renderer. Uses `dirty` and `revert` on the field controller ([11 §2](./11-rendering-and-adapters.md#2-headless-field-api)) and dirty semantics from [03 §4](./03-state-model.md#4-dirty-semantics).

**1. A wrapper that adds the indicator.** It subscribes only to `dirty`, so it re-renders when the flag flips, not on every keystroke:

```tsx
function withDirty<P extends { controller: FieldController<any> }>(Inner: ComponentType<P>) {
  return function Dirty(props: P) {
    const c = props.controller;
    const dirty = useReadable(useMemo(() => c.select(s => s.dirty), [c]));   // one subscription; O(1) per field (03 §4)
    return (
      <div className="field" data-dirty={dirty || undefined}>
        <Inner {...props} />
        {dirty && (
          <button type="button" onClick={() => {
            try { c.revert(); }
            catch (error) {
              if (errorCode(error) === 'E_ROW_BASELINE_UNKNOWN') announce('Review or revert the whole array; this row cannot be matched to its new baseline.');
              else throw error;
            }
          }} aria-label="Undo change">
            <span aria-hidden>●</span>
          </button>
        )}
      </div>
    );
  };
}
```

**2. Wrap the field map once.** Every field type, including custom ones and the fallback, gets it:

```ts
const mapValues = <T extends object>(o: T, f: (v: any) => any) =>
  Object.fromEntries(Object.entries(o).map(([k, v]) => [k, f(v)])) as T;

const fields = { text: TextInput, number: NumberInput, choice: Select, 'x-grid': GridField };
const renderers = defineRenderers({
  field: mapValues(fields, withDirty),
  fallback: { field: withDirty(UnknownField), group: Fragment },
  // groups and arrays: same idea with `ScopeController.dirty` for "section changed" badges
});
```

The renderer prop types are kept, since `withDirty` returns the same props as the component it wraps.

**3. Opt out per field (optional).** To let the contract switch it off, add a typed `ui` key to the field types that need it ([01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions)) and check `props.node.ui?.dirtyIndicator !== false` in the wrapper.

Notes:
- **What "dirty" means.** It compares with the baseline (loaded value, else default), not with the last keystroke. Typing a value back to the original clears the dot. Custom types compare with their `equals`.
- **`revert()`** restores only the targeted field's baseline. Outside arrays it drops the key from `current`; inside arrays it reconstructs the atomic array with that one field restored by row identity, preserving edits in other rows/cells ([03 §3.2](./03-state-model.md#32-atomic-array-writes)). It's a host write, so it's undoable when history is on ([03 §6.2](./03-state-model.md#62-undo--redo-opt-in)) and effects run as usual.
- **Stale array identity.** The button catches `E_ROW_BASELINE_UNKNOWN` instead of guessing a row. `errorCode` is the helper from §22; `announce` is the host's accessible status-message helper. Accepting an external whole array or reverting that array restores known identity; accepting local values alone does not.
- **Computed fields.** Non-overridable computed fields are never dirty, so they never show the dot. Hide the button for disabled or read-only fields if reverting shouldn't be offered there.
- **After submit.** A returned entity advances the acknowledged baseline; dots clear for acknowledged fields, but retained serializer-excluded/later edits stay present and may remain dirty ([09 §4.2](./09-loading-and-submission.md#42-acknowledgement-and-response-reconciliation)). Without `entity`, fields stay dirty until reconciliation or an explicit reload.
- **Accessibility.** Don't rely on the dot alone: give the button a text label, and announce the state with visually hidden text (for example "changed") if it matters to users.
- **Whole form.** `engine.form.select('dirty')` drives a "You have unsaved changes" banner or a leave-page guard with the same O(1) cost.

## 24. End-to-end tests

Stable selectors for Playwright, Cypress or Detox, using `testId` ([11 §8.1](./11-rendering-and-adapters.md#81-test-ids)).

**1. Turn it on** (here, everywhere except production):

```ts
const engine = createFormEngine({
  contract, registry,
  testIds: import.meta.env.PROD ? false : { prefix: 'checkout' },
});
```

**2. Put it on elements in your renderers.** One line per renderer:

```tsx
function TextInput({ controller }: FieldRendererProps<'text'>) {
  const { state, setValue, onFocus, onBlur } = useField<string>(controller);   // the controller itself, not a path
  return (
    <label htmlFor={controller.ids.input}>
      {state.label}
      <input id={controller.ids.input} data-testid={state.testId || undefined}
             value={state.value} onChange={(e) => setValue(e.target.value)} onFocus={onFocus} onBlur={onBlur} />
    </label>
  );
}

// React Native
<TextInput nativeID={controller.ids.input} testID={state.testId || undefined} value={state.value} onChangeText={setValue} onFocus={onFocus} onBlur={onBlur} />

// array rows: the row container
<View testID={useReadable(row.testId) || undefined}>…</View>
```

**3. Write tests by data path:**

```ts
// Playwright
await page.getByTestId('checkout:name').fill('Ada');
await page.getByRole('button', { name: 'Add address' }).click();
await page.getByTestId('checkout:addresses.0.city').fill('Berlin');

// same id from the path, with the engine's format
import { fieldSelector } from '@formgraph/testing/e2e';
await page.locator(fieldSelector('addresses[0].city', { prefix: 'checkout' })).fill('Berlin');

// Detox
await element(by.id('checkout:addresses.0.city')).typeText('Berlin');
```

Notes:
- **Rows follow position.** After moving the second address up, it's `addresses.0`. Tests describe rows by position, so this is what they expect. Don't use `controller.ids` in tests: they follow row identity (`_rk3`), which a test can't know.
- **Custom `format`.** If you change `format` in the engine, pass the same one to `fieldSelector`.
- **Third-party renderer sets** that pass the conformance suite already apply `testId`, so the same tests work when you swap renderers.
- **Prefer roles for buttons and headings** (`getByRole`); use test ids for contract-driven fields, where labels are translated and repeat per row.

## 25. A yes/no question that can be unanswered

A `boolean` field is always `true` or `false`: its empty value is `false`, and `required` means "must be true" (the "accept the terms" case). For a question that starts unanswered and must be answered either way, use a `choice` with two options:

```json
{ "id": "hasPets", "kind": "field", "fieldType": "choice", "label": "Do you have pets?",
  "options": { "source": "static", "items": [ { "value": true, "label": "Yes" }, { "value": false, "label": "No" } ] },
  "conditions": { "required": true },
  "ui": { "variant": "radio" } }
```

- The value is `true`, `false` or `null` (unanswered), so `required` means "answered".
- To demand a specific answer, add an `expr` rule (`{ "var": "./hasPets" }`). Use a plain `boolean` for a checkbox that must be ticked.
- `choice` isn't affected by checkbox renderers, so nothing changes for existing `boolean` fields.

Macro: `yes-no`.
