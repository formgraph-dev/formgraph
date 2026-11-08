# 15 · Cookbook

[← Index](./README.md)

Patterns built only from core primitives. Several have a macro shortcut in `@formgraph/macros` ([14 §7](./14-macros.md#7-standard-library-formgraphmacros-optional)). The long form here is what the macro expands to.

## 1. Autosave and drafts

Drafts save a **snapshot** (the user's edits plus what they were based on), not the submit payload, so reopening shows the same values **and** the same change indicators ([08 §8](./08-loading-and-submission.md#8-snapshots-drafts)):

```ts
// open
const engine = await createFormEngineFrom(contractJson, { registry });   // load + create (17 §2)
const draft = await drafts.get(formKey);
if (draft) {
  const { conflicts } = await engine.restore(draft);
  if (conflicts.length) showBanner('This record changed since your draft. Review the highlighted fields.');
}

// autosave: one save at a time; edits during a save queue exactly one follow-up
let saving: Promise<void> | null = null;
let savedRevision = engine.revision.get();     // after restore: the restored state is what's stored
let again = false;
async function save() {
  if (saving) { again = true; return; }
  do {
    again = false;
    const snap = engine.snapshot();
    setStatus('saving');
    saving = drafts.put(formKey, snap);
    try { await saving; savedRevision = snap.revision; setStatus('saved'); }   // only once stored
    catch { setStatus('error'); }
    finally { saving = null; }
  } while (again);                              // latest state wins; older snapshots are never written after newer ones
}
const debouncedSave = debounce(save, 1000);
engine.revision.subscribe((rev) => {
  setUnsaved(rev !== savedRevision);         // "Unsaved changes" / "Draft saved"
  if (rev !== savedRevision) debouncedSave();
});
// submit
if ((await engine.submit()).ok) await drafts.delete(formKey);
```

`revision` (core, [02 §7](./02-state-model.md#7-engine-write-api-core)) bumps on any commit that changes `current`, `rows` or `stash`, whatever its origin (user edits, host writes, effects, undo, reset, reload), because all of them change what the draft must contain. `savedRevision` only moves when a `put` succeeds, so a failed save leaves the form marked unsaved. The save status lives in host state. A step-based draft saves after `if ((await wiz.next()).ok)` instead ([10 §5](./10-rendering-and-adapters.md#5-wizards)).

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

`count` is one of the built-in aggregate functions ([04 §2](./04-expressions.md#2-evaluator-registry)).

Picking "None" alongside other values collapses the selection to `["none"]`. To do the reverse (picking another value drops "None"), use a second effect that removes it. An effect may target its own field. Such effects are exempt from the compile-time cycle check, but limited by `limits.effectDepth` ([03 §6](./03-paths-and-dependency-graph.md#6-scheduler), [03 §5.2](./03-paths-and-dependency-graph.md#52-cycle-detection)). Macro: `multi-choice-with-none`.

## 5. Carry-forward options

```json
{ "id": "favourite", "kind": "field", "fieldType": "choice",
  "options": { "source": "static", "items": [ … ],
    "filter": { "lang": "json-logic", "expr": { "in": [{ "var": "$option.value" }, { "var": "./interests" }] } } } }
```

The `items` are the same as `./interests`. Repeating the list is the authoring cost. Macro: `carry-forward`, which copies the items from the source field during expansion.

## 6. Reset a field when another changes

This is an effect on the **source** field ([06 §2](./06-effects-and-computed.md#2-write-origins-and-effect-triggers)):

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
const rowEngine = createFormEngine({
  contract: rowContract, registry,
  context: { parent: engine.getValues() },   // whatever the row's expressions read as $context.parent
});
await rowEngine.reload({ entity: engine.getValues(`/addresses/${index}`) });   // row value becomes `loaded`, so dirty works
// on save:
const rows = engine.array('/addresses');
const key = rows.keyAt(index)!;                 // capture the RowKey when editing starts; survives reorders meanwhile
// …
if ((await rowEngine.validate()).ok) rows.set(key, rowEngine.getValues());   // values, not the serialized payload
rowEngine.dispose();
// on cancel: rowEngine.dispose()
```

**Limitation:** `../` and `/` references inside the row can't reach the parent form. Pass the parent values the row needs through `context` (and refer to them as `$context.parent.x`), or use inline editing.

## 8. View mode / read-only review page

Put a `readOnly` condition on the root group driven by host context. `readOnly` is inherited ([01 §4](./01-schema-contract.md#4-layout-vs-conditions)):

```json
{ "id": "root", "kind": "group", "ui": { "variant": "none" },
  "conditions": { "readOnly": { "lang": "json-logic", "expr": { "==": [{ "var": "$context.mode" }, "view"] } } }, "children": [ … ] }
```

How read-only fields look (plain text, formatted values) is up to the renderer.

## 9. One page, steps, or one question at a time

This is purely a rendering choice. Register different group renderers by platform or mode using ranked testers ([10 §4.1](./10-rendering-and-adapters.md#41-ranked-testers)), for example rendering `step: true` groups as plain sections when `ctx.platform === 'desktop-single-page'`.

## 10. Delayed error display, debounced effects, tabs, row labels

- **Delayed errors:** a timer in the adapter around the `showFeedback` handle.
- **Debounced effects:** commit text on blur in the renderer, or write a custom `x-` effect handler that debounces.
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
- Visible but not submitted (for example "confirm email"): remove it in the outbound transformer (`submission.transformerId`, [08 §1](./08-loading-and-submission.md#1-config)).

## 14. Submit while invalid

Skip `submit()`: `await target(engine.serialize({ validate: false }))`. Or mark rules `severity: 'warning'` with the `warnings` gate set to `allow` (the default, `submission.gates`) so they don't block ([05 §7](./05-validation.md#7-severity)).

## 15. Externally derived inputs via `setContext`

For values that have to come from outside the form (an API, storage, a device) and depend on form values. The engine stays synchronous: the host loads it, then pushes the result into `context` ([04 §5.2](./04-expressions.md#52-async-data-where-it-goes-instead)).

```ts
let call: AbortController | null = null;
const country = engine.field('/country').value;          // Readable (10 §2)
const onCountry = async (country: Json) => {
  call?.abort();                                          // cancel the previous request
  call = new AbortController();
  const ctl = call;
  engine.signal.addEventListener('abort', () => ctl.abort());     // also aborted on engine.dispose(); works on React Native
  const signal = ctl.signal;
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
- `engine.signal` is aborted by `dispose()`, and writes to a disposed engine are no-ops, so a late response is harmless ([17 §3](./17-api-reference.md#3-formengine-core)).

## 16. Postcode → city autofill

Model the city as a select whose options come from the postcode. A single match fills itself in:

```json
{ "id": "city", "kind": "field", "fieldType": "choice",
  "options": { "source": "provider", "providerId": "geo.citiesByPostcode", "dependsOn": ["./postcode"] },
  "cascade": { "whenEmpty": "disable", "onChange": "keep-if-valid" },
  "autoSelectSingle": true }
```

This gets cancellation, out-of-order handling, validity checks, and "loading saved data doesn't clear it" from [12](./12-option-sources-and-cascading.md). If the postcode maps to several cities, the user picks one. If it maps to none, the list is empty, and `in-options` / `required` report it. Have the query transformer return `null` while the postcode is too short. That skips the request ([07 §2](./07-async-data-and-registries.md#2-provider-options)) and avoids lookups on partial input.

## 17. Row-level diffs for patch submissions

`patch` mode sends changed arrays whole ([08 §4](./08-loading-and-submission.md#4-submission-pipeline)). If the submit target wants row operations, compute them in the outbound transformer. You need a row identity field (the same one used for `identityKey`, [03 §4](./03-paths-and-dependency-graph.md#4-array-row-identity)):

```ts
import { dequal as deepEqual } from 'dequal';

registry.outboundTransformers.register('orders.patch', ({ payload, mode, loaded, values }) => {
  if (mode !== 'patch' || !('lines' in payload)) return payload;   // only patch, only when lines changed
  const before = new Map((loaded.lines ?? []).map((r) => [r.id, r]));
  const after = values.lines ?? [];
  return {
    ...payload,
    lines: {
      added:   after.filter((r) => r.id == null),
      changed: after.filter((r) => r.id != null && !deepEqual(r, before.get(r.id))),
      removed: [...before.keys()].filter((id) => !after.some((r) => r.id === id)),
      order:   after.map((r) => r.id ?? null),
    },
  };
});
```

**Two shapes.** `payload` is the wire shape (after `serializeAs` and hidden/excluded fields are dropped, [08 §4](./08-loading-and-submission.md#4-submission-pipeline)); `values` and `loaded` are the form shape. This version is safe when `lines` has no `serializeAs` and its rows have no excluded fields. Otherwise read the rows from `payload.lines` and map the `loaded` rows the same way before comparing, or a row can look changed because of a field that is never sent.

The target's contract for this shape is yours to define. The engine stays out of it.

## 18. Order totals

Use the built-in `count`/`sum` operators ([04 §2](./04-expressions.md#2-evaluator-registry)) and combine them with ordinary expressions:

```json
{ "kind": "field", "id": "lineCount", "fieldType": "number", "computed": { "expr": { "lang": "json-logic", "expr": { "count": { "var": "/items" } } } } },
{ "kind": "field", "id": "subtotal",  "fieldType": "number", "computed": { "expr": { "lang": "json-logic", "expr": { "sum": { "var": "/items[*].amount" } } } } },
{ "kind": "field", "id": "total",     "fieldType": "number",
  "computed": { "expr": { "lang": "json-logic", "expr": { "*": [{ "var": "/subtotal" }, { "+": [1, { "var": "$context.taxRate" }] }] } } } }
```

`amount` itself is a per-row computed (`qty * price`), so a keystroke in one row costs one row computation plus O(rows) for `subtotal` ([06 §3.2](./06-effects-and-computed.md#32-aggregates)).

## 19. Server-rendered form

```ts
// server
const engine = await createFormEngineFrom(contractJson, { registry, params });
await engine.ready;
const html = renderToString(<FormProvider engine={engine}><FormRenderer renderers={renderers} /></FormProvider>);
send(html, { state: engine.exportState() });

// client; `contract` is the CompiledSchema loaded once (17 §1)
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

Same element ids on both sides come from row keys ([10 §8](./10-rendering-and-adapters.md#8-accessibility)); option lists, dynamic sub-schemas and locale bundles (`localeBundles`) aren't fetched again ([08 §9](./08-loading-and-submission.md#9-server-rendering)). Choose the locale on the server (cookie or `Accept-Language`): a client asking for another locale first renders the server's, then switches after `ready`.

## 20. Slim bundle

Size-sensitive apps that don't need undo, drafts or wizards import the core entry and opt in per feature ([02 §8](./02-state-model.md#8-plugins)):

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

Wrapping a third-party control (here, an editable grid) whose value is an object, which has its own validation and its own async work. Uses [01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions) and [10 §2.2](./10-rendering-and-adapters.md#22-component-reported-messages-and-busy-state).

**1. Declare the type.** This tells the engine what the value looks like, what "empty" and "equal" mean, and how it's exported:

```ts
type GridValue = { columns: string[]; rows: Array<Record<string, number | string | null>> };

const registry = createRegistry({ parent: coreRegistry }).fieldType('x-grid', {   // typed: contracts see x-grid (20 §1)
  value:  v.object({ columns: v.array(v.string()), rows: v.array(v.record(v.string(), v.nullable(v.union([v.number(), v.string()])))) }),
  config: v.object({ maxRows: v.optional(v.number()) }),
  ui:     v.object({ dense: v.optional(v.boolean()) }),
  empty:  { columns: [], rows: [] },
  isEmpty: (g) => g.rows.length === 0,                    // `required` = at least one row, whatever the columns
  payloadSchema: { type: 'object', properties: { columns: { type: 'array' }, rows: { type: 'array' } } },
});

// Engine-side rule on the whole value, pointing at cells with subPath
registry.validators.register('grid-qty-positive', {
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
- **Messages from three places, one list.** Engine rules (`grid-qty-positive`), the grid's own checks (`report`), and server errors such as `{ path: '/lines/rows/3/qty' }` (mapped to `subPath` `/rows/3/qty`, [08 §5](./08-loading-and-submission.md#5-external-error-mapping)) all land in `feedback`. Severity, submit blocking, `firstInvalid`, and `showFeedback` timing work the same for all of them. A generic renderer that ignores `subPath` still shows them at the field.
- **Busy gate.** Submit waits while `recalc` is busy, or returns `reason: 'validating'` without `awaitAsync`.
- **Dirty, undo, drafts, submit.** These work on the whole value. Dirty uses `deepEqual`, or `equals` if you define it.
- **Lifecycle.** `report` messages persist until the grid replaces them (the engine doesn't guess when they're stale), and they and busy tokens are dropped when the field is removed. They aren't saved in drafts, so the grid should re-validate on mount.

**When not to use this.** If rows need their own conditions, effects, option sources, or per-cell controllers, use an `array` of groups with an `x-` array renderer (`ui.variant`) instead. The engine then knows each cell as a real path.

## 22. Another engine as a custom field

Embedding a whole, separately owned form (another formgraph contract, or a foreign form engine) as one field of the parent. It builds on §21: the child's output is the field's structured value, and its messages, busy state, and validation are bridged through the controller ([10 §2.2](./10-rendering-and-adapters.md#22-component-reported-messages-and-busy-state)).

**First, check whether you need a separate engine.** If the child is a formgraph contract and should behave as part of the parent (cross-field references, one undo history, one dependency graph), use a dynamic sub-form instead ([07 §5](./07-async-data-and-registries.md#5-dynamic-sub-forms)). Use a nested engine when you want **isolation**: a different registry, allow list, or contract version, a form owned by another team, or a different engine entirely. The parent then sees the child only as an opaque value.

**1. The field type.** The value is whatever the child submits:

```ts
const registry = createRegistry({ parent: coreRegistry }).fieldType('x-subform', {   // opaque value: the child's payload type isn't known statically
  value: v.any(), config: v.object({ contractUrl: v.string() }), ui: v.object({}),
  empty: null,
  // payloadSchema: the child's toPayloadSchema(), if it's known when the parent is built
});
```

**2. The renderer.** It owns the child engine and keeps five bridges in sync:

```tsx
const toPointer = (p: string) => '/' + p.replace(/\[(\d+)\]/g, '.$1').split('.').join('/');   // addresses[2].city → /addresses/2/city

const SubForm = ({ controller, node }: FieldRendererProps<'x-subform'>) => {
  const parent = useFormEngine();
  const [child, setChild] = useState<FormEngine | null>(null);

  // Create once per contract; seed from the parent's value; dispose on unmount
  useEffect(() => {
    let engine: FormEngine | undefined;
    createFormEngineFrom(fetchJson(node.contractUrl), {
      registry: childRegistry,                       // isolated: its own validators, sources, allow list
      plugins: { history: false },                   // the parent's history covers undo (see below)
      context: hostContext,                          // whatever the host passes to the parent, or a narrowed copy
      i18n: { locale: parent.form.get().locale },
    }).then(async (e) => {
      if (parent.signal.aborted) return e.dispose();
      engine = e;
      await e.reload({ entity: controller.value.get() ?? {}, policy: 'replace' });
      setChild(e);
    });
    return () => engine?.dispose();
  }, [node.contractUrl]);

  useEffect(() => {
    if (!child) return;
    let pushed: Json = controller.value.get();
    const offs = [
      // 1. Value: child → parent. One parent write per child commit
      child.revision.subscribe(() => {
        pushed = child.serialize({ mode: 'full' });
        controller.setValue(pushed);
      }),
      // 2. Value: parent → child, for writes that didn't come from the child (reset, undo, reload, restore)
      controller.value.subscribe((v) => {
        if (v !== pushed) child.reload({ entity: v ?? {}, policy: 'replace' });
      }),
      // 3. Busy: the child's async validation and loads hold the parent's gate
      child.form.select('validating').subscribe((busy) => controller.setBusy('subform', busy)),
      // 4. Validation: the parent's submit/next/validate validates the whole child, touched or not
      controller.onValidate('subform', async ({ trigger }) => {
        const { notices } = await child.validate({ trigger: 'submit' });   // also shows them inside the child
        return notices.length ? notices.map(e => ({
          message: e.message, severity: e.severity, code: e.code, subPath: toPointer(e.path),
        })) : null;
      }),
      // 5. Locale follows the parent
      parent.form.select('locale').subscribe((l) => child.setLocale(l)),
    ];
    return () => offs.forEach(off => off());
  }, [child]);

  // 6. Server errors on the parent field (mapped to subPath, 08 §5) go into the child, next to the right input
  const { feedback } = useReadable(controller.state);
  useEffect(() => {
    const external = feedback.filter(m => m.source === 'external' && m.subPath);
    if (child && external.length) child.mapExternalErrors(external.map(m => ({
      path: m.subPath!, message: m.message, code: m.code, severity: m.severity })));
  }, [child, feedback]);

  if (!child) return <Spinner />;
  return (
    <FormProvider engine={child}>
      <FormRenderer renderers={childRenderers} />
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
- **Undo.** Each child commit is a parent write, so the parent's history records it (coalesced per `coalesceMs`). Undo restores the value and re-seeds the child through bridge 2. Leave history off the child to avoid two undo stacks.
- **Re-seeding resets the child's meta.** A `replace` reload after undo or reset clears the child's touched state. That's usually what you want. If you need to keep it, compute the diff and use `child.applyExternal` instead.
- **No cross-references.** The parent sees the child's output only as one opaque value. Parent rules and functions can inspect it, but parent paths can't point inside it, and the child can't see the parent except through `context`. If you need more than that, use a dynamic sub-form.
- **Round-trip.** Bridge 2 seeds the child with what it serialized. That only works if `serializeAs` and outbound transformers in the child contract can be loaded back. Otherwise, store `child.getValues()` instead, and serialize at the parent's submit with an outbound transformer.
- **Server rendering.** The child is created on the client after mount. For SSR, also render the child on the server and pass its `exportState()` next to the parent's ([15 §19](#19-server-rendered-form)).
- **Foreign engines.** The same six bridges work for any engine that exposes a value-change event, a validate call that returns located messages, and a busy flag. Map its paths to JSON Pointers for `subPath`.

## 23. Dirty indicator on every field

Mark changed fields ("●" next to the label) with a revert button, for every field type, without touching each renderer. Uses `dirty` and `revert` on the field controller ([10 §2](./10-rendering-and-adapters.md#2-headless-field-api)) and dirty semantics from [02 §4](./02-state-model.md#4-dirty-semantics).

**1. A wrapper that adds the indicator.** It subscribes only to `dirty`, so it re-renders when the flag flips, not on every keystroke:

```tsx
function withDirty<P extends { controller: FieldController<any> }>(Inner: ComponentType<P>) {
  return function Dirty(props: P) {
    const c = props.controller;
    const dirty = useReadable(useMemo(() => c.select(s => s.dirty), [c]));   // one subscription; O(1) per field (02 §4)
    return (
      <div className="field" data-dirty={dirty || undefined}>
        <Inner {...props} />
        {dirty && (
          <button type="button" onClick={() => c.revert()} aria-label="Undo change">
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
- **`revert()`** drops the field from `current`, so it shows the baseline again. It's a host write, so it's undoable when history is on ([02 §6.2](./02-state-model.md#62-undo--redo-opt-in)) and effects run as usual.
- **Computed fields.** Non-overridable computed fields are never dirty, so they never show the dot. Hide the button for disabled or read-only fields if reverting shouldn't be offered there.
- **After submit.** If the submit target returns `entity`, the engine reloads with `replace`, which moves the baseline, so the dots clear on their own ([08 §4](./08-loading-and-submission.md#4-submission-pipeline)). Without `entity`, fields stay dirty until you reload.
- **Accessibility.** Don't rely on the dot alone: give the button a text label, and announce the state with visually hidden text (for example "changed") if it matters to users.
- **Whole form.** `engine.form.select('dirty')` drives a "You have unsaved changes" banner or a leave-page guard with the same O(1) cost.

## 24. End-to-end tests

Stable selectors for Playwright, Cypress or Detox, using `testId` ([10 §8.1](./10-rendering-and-adapters.md#81-test-ids)).

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
  const { state, setValue } = useField<string>(controller);   // the controller itself, not a path
  return (
    <label htmlFor={controller.ids.input}>
      {state.label}
      <input id={controller.ids.input} data-testid={state.testId || undefined}
             value={state.value} onChange={(e) => setValue(e.target.value)} />
    </label>
  );
}

// React Native
<TextInput nativeID={controller.ids.input} testID={state.testId || undefined} value={state.value} onChangeText={setValue} />

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
