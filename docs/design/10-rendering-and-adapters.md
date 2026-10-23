# 10 · Rendering & Adapters

[← Index](./README.md) · Addresses review items I1, I2

## 1. Layering

```
core (no framework, no platform, no deps; §6) → engine, FormStore, Readable handles, field controllers
adapter (@formgraph/react, @formgraph/vue, …)  → ~30-line Readable binding + hooks/composables + <FormRenderer/>
host renderers (app code)          → registry: fieldType / ui.variant → actual UI components
```

Adapters stay small (target: under 300 LOC each). All logic lives in core **field controllers**, so behavior can't drift between frameworks.

## 2. Headless field API

The core exposes a framework-neutral controller per node instance (per row key path):

```ts
interface FieldState<V = Json> {        // one immutable object; a new reference only when a member changes
  value: V; visible: boolean; disabled: boolean; readOnly: boolean; required: boolean;
  feedback: Notice[]; severity: Severity | null; showFeedback: boolean;
  errors: Notice[]; notices: Notice[];                            // 05 §7.4
  validated: boolean; valid: boolean;
  touched: boolean; dirty: boolean; validating: boolean;
  label: string; description?: string; placeholder?: string;
  conflict: Conflict | null;                                             // 08 §7
  testId: string;                            // e2e selector, index-based; changes when the row moves (§8.1); '' when testIds: false
}

interface FieldController<V = Json> {
  path: Readable<DataPath>;                  // index-based; changes when an ancestor row moves. key: stable across moves
  key: RowKeyPath; compiled: CompiledNode;   // compiled.node is the contract node (01 §7.3)
  state: Readable<FieldState<V>>;            // primary: adapters subscribe once per field (02 §2.2)
  select<T>(pick: keyof FieldState<V> | ((s: FieldState<V>) => T), eq?: (a: T, b: T) => boolean): Readable<T>;   // memoized: get() keeps its result while input unchanged or eq holds (default Object.is)
  ids: FieldIds;                             // §8
  // fine-grained handles, derived from `state` (select(s => s.value), …), for non-UI code
  value: Readable<V>; visible: Readable<boolean>; feedback: Readable<Notice[]>; dirty: Readable<boolean>;   // … one per FieldState member
  options?: Readable<OptionsState>;          // separate: changes on its own schedule (12 §9)
  assets?: AssetController;                  // asset / multi-asset fields only; added by the assets plugin
  // commands
  setValue(v: V): void;  revert(): void;
  handlers: { onFocus(): void; onBlur(): void };   // renderers report interaction: visited / touched + blur validation (05 §3)
  focus(): Promise<boolean>;                   // DOM focus only (§8); no meta change
  setInputError(message: Text | null): void;   // see below
  report(reporter: string, messages: ComponentNotice[] | null): void;   // §2.2
  onValidate(reporter: string, fn: ComponentValidate): () => void;      // §2.2; returns unregister
  setBusy(token: string, busy: boolean): void;                          // §2.2
  registerElement(el: unknown): () => void;    // focus target (§8); returns unregister
  resolveConflict(choice: 'local' | 'external'): void;   // 08 §7
  // option commands (loadMore, search, retry) live on OptionsState (12 §9)
}

interface ArrayController extends FieldController<Json[]>, ArrayHandle {   // ArrayHandle: 02 §7, same as engine.array(path)
  canAdd: Readable<boolean>; canRemove: Readable<boolean>;
  row(key: RowKey): ScopeController;
}

interface ScopeController {          // one array row, or a dataScope group
  path: Readable<DataPath>; key: RowKeyPath;   // path changes when the row (or an ancestor row) moves
  field(relPath: PathRef): FieldController;     // './city'
  value: Readable<Json>; visible: Readable<boolean>;
  dirty: Readable<boolean>;                     // rows: derived on subscribe, O(row) (02 §4); groups: O(1) count
  testId: Readable<string>;                     // row / group container selector (§8.1)
  feedback: Readable<Notice[]>; severity: Readable<Severity | null>; showFeedback: Readable<boolean>;   // container-level messages, 05 §7.4
  registerElement(el: unknown): () => void;    // container focus target (§8)
  focus(): Promise<boolean>;
}

interface AssetController {          // assets plugin, 07 §2.1
  items: Readable<Array<AssetRef & { id: string }>>;   // status / progress / error per file; `id` is engine-local (kept in meta, not in values)
  add(file: Blob | PlatformFile): void;   // queues as 'pending'; enforces accept / maxBytes / maxFiles
  retry(id: string): void;           // 'error' → 'pending'
  remove(id: string): void;          // aborts the store call if running
}
```

Full signatures: [17](./17-api-reference.md). Controllers are created lazily and cached by row key path, then disposed when the row or node is removed. `state` is assembled lazily: a member's facet is computed only while `state` has subscribers, and a write notifies a field's `state` once per tick however many of its members changed.

### 2.1 Raw input stays in the renderer

The core stores **parsed values only**. What the user is typing (`"12."`, `"1,5"`, a half-entered date) lives in the renderer's local state. The renderer calls `setValue` when the input parses (or on blur, if it prefers to commit on blur).

When the input can't be parsed, the renderer calls `setInputError(message)`. This:
- stores a message with `source: 'input'` and severity `error` for the field ([05 §5](./05-validation.md#5-notice-shape)), shown through `feedback` / `showFeedback` like any other error,
- **blocks submit** until cleared,
- leaves the last valid value in place.

`setInputError(null)` (or any successful `setValue`) clears it. The core has no parse or format pipeline. Locale-aware parsing is a renderer concern.

### 2.2 Component-reported messages and busy state

A renderer that wraps its own subsystem (a third-party grid, a map, a signature pad, an OCR widget) often has its own validation and its own async work. It feeds both into the engine through the controller, so they behave like the engine's own:

```ts
type ComponentNotice = Omit<NoticeInput, 'path'>;   // NoticeInput: 05 §4; severity default 'error'

controller.report('x-grid', [
  { message: 'Quantity must be positive', subPath: '/rows/3/qty' },
  { message: 'Totals were rounded', severity: 'info' },
]);
controller.report('x-grid', null);       // clear this reporter's messages
controller.setBusy('x-grid:recalc', true);
controller.setBusy('x-grid:recalc', false);
```

**`report(reporter, messages)`**
- Stores the messages as `Notice`s with `source: 'component'` and `rule: reporter` ([05 §5](./05-validation.md#5-notice-shape)). Each call **replaces** that reporter's earlier messages. Other reporters and engine sources are untouched, so one field can carry several reporters.
- Messages count like any other message: `feedback` and `severity` (strongest wins), `showFeedback` timing, `valid`, `firstInvalid`, and submit/`next()` blocking per severity ([05 §7](./05-validation.md#7-severity)).
- The engine doesn't clear them on `setValue`: the component knows when its findings are stale. They are cleared when the controller is disposed (the node or row is removed), and on `reset`/`restore`. Like other messages, they're suppressed while the field is hidden or disabled unless `validateHidden`/`validateDisabled` is set.
- They are transient: they aren't in drafts, `exportState`, or history. After a resume or restore, the component reports again on mount.
- `reporter` is a free string. `'input'` is reserved for `setInputError`, which is shorthand for a single `error` message with `source: 'input'`.

**`onValidate(reporter, fn)`**

```ts
type ComponentValidate = (ctx: { trigger: 'submit' | 'next' | 'validate'; signal: AbortSignal })
  => ComponentNotice[] | null | Promise<ComponentNotice[] | null>;
```

- Lets a component take part in **engine-initiated** validation. When `submit()`, `next()`, or `validate()` covers this field (and it's eligible: visible, or `validateHidden`), the engine calls every registered hook. It counts them in `validating` while they run, and stores each result as that reporter's messages, exactly like `report`.
- `report` is for findings the component makes on its own (as the user types). `onValidate` is for "check everything now", such as forcing a nested form to validate fields the user never touched.
- With `submission.awaitAsync`, submit waits for the hooks; otherwise it returns `reason: 'validating'` while they're pending. A later trigger aborts the previous call through `signal`.
- A hook that throws or rejects fails closed: the field gets a blocking `error` under that reporter, and `W_COMPONENT_VALIDATE_FAILED` is reported.
- Hooks are unregistered when the controller is disposed. One hook per reporter: registering a reporter name again replaces the previous hook.

**`setBusy(token, busy)`**
- Holds the pending-work gate ([01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions)) for the field while any token is busy. `submit()`/`next()` wait (with `submission.awaitAsync`) or return `reason: 'validating'`. `FieldState.validating` is true meanwhile.
- Tokens are per controller. Setting the same token twice is idempotent. All tokens are released when the controller is disposed, so an unmounted widget can't block submit forever.
- Use `pending()` on the field type when busy state is visible in the value (asset status). Use `setBusy` when it lives only inside the component.

What the component **can't** do: create paths or controllers inside its value, take part in conditions or effects for its internal parts, or bypass the type's `value` schema on `setValue`. If parts of the value need any of that, model them as a group or an array ([01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions)).

## 3. Framework bindings

Each adapter binds `Readable<T>` to its framework's reactivity. That binding is the only framework-specific reactive code. Adapters also install a read observer ([02 §2.6](./02-state-model.md#26-tracking-reads-for-framework-adapters)) so that `get()` calls inside the framework's own computed values and effects are tracked natively.

| Framework | Binding |
|-----------|---------|
| React / React Native | `useSyncExternalStore(r.subscribe, r.get)` |
| Vue 3 | `shallowRef(r.get())` + `r.subscribe(v => ref.value = v)`, unsubscribed `onScopeDispose` |
| Solid | `from(r.subscribe)` / `createSignal` + `onCleanup` |
| Svelte 5 | `createSubscriber` from `svelte/reactivity`. Svelte 4 / `$store` works as is, because `subscribe` follows the store contract (see note) |
| Angular | `toSignal`-style wrapper with `DestroyRef` |

### React

```tsx
const useReadable = <T,>(r: Readable<T>) => useSyncExternalStore(r.subscribe, r.get, r.get);  // 3rd arg = getServerSnapshot

function useField<V, T = FieldState<V>>(
  target: string | FieldController<V>, pick?: keyof FieldState<V> | ((s: FieldState<V>) => T), eq?: (a: T, b: T) => boolean,
) {
  const select = typeof pick === 'string' ? (s: FieldState<V>) => s[pick] as T : pick;   // key shorthand
  const engine = useFormEngine();
  const scope = useScope();
  const c = typeof target === 'string' ? engine.field<V>(target, scope) : target;   // path (./ resolved against row scope) or a controller from renderer props
  const sel = useRef({ select, eq }); sel.current = { select, eq };   // latest selector, no resubscribe
  const r = useMemo(() => select
    ? c.select(s => sel.current.select!(s), (a, b) => (sel.current.eq ?? Object.is)(a, b))
    : c.state, [c, !select]);                      // one subscription per field
  const state = useReadable(r);
  return { state, setValue: c.setValue, ...c.handlers, ids: c.ids, controller: c };   // onFocus / onBlur
}

const { state: { value, showFeedback, feedback } } = useField<string>('./city');
const { state: dirty } = useField('/notes', s => s.dirty);   // inline is fine; re-renders only when dirty flips
const { state: dirty2 } = useField('/notes', 'dirty');       // key shorthand, same effect

function useForm<T = FormState>(select?: keyof FormState | ((s: FormState) => T), eq?: (a: T, b: T) => boolean): T;
const phase = useForm('phase');                              // engine.form.select('phase'); re-renders only on phase change
const canSubmit = useForm(s => s.valid && !s.validating);

// any tracked computation, one subscription for the component (02 §2.6)
function useTracked<T>(fn: () => NotAsync<T>, eq?: (a: T, b: T) => boolean): T;   // runs fn in observeReads, subscribes to what it read, useSyncExternalStore on the result
const canShip = useTracked(() =>
  engine.field('country').value.get() !== '' && engine.form.select('counts').get().error === 0);
```

- `useTracked` keeps the latest `fn` in a ref, like inline selectors below, and re-renders only when its result changes. It's React-only: in Solid, Vue, Svelte 5 and Angular, plain `createMemo` / `computed` / `$derived` / `computed()` track `get()` calls directly through the adapter's read observer.

- Inline selectors are allowed: the adapter keeps the latest `select` / `eq` in a ref, so a new function each render does not resubscribe. Other adapters do the same with their equivalent.
- `select` memoizes: the derived `get()` returns the **same result object** while the input `FieldState` is unchanged, or while `eq(prev, next)` holds (default `Object.is`). `useSyncExternalStore` requires this. Selectors returning new objects need `eq` (for example `shallowEqual`).
- **Server rendering:** `get` doubles as `getServerSnapshot`, since the engine state is the same on server and client after `resume` ([08 §9](./08-loading-and-submission.md#9-server-rendering)). Every adapter must provide the equivalent (Vue `onServerPrefetch`, Solid/Svelte SSR reads `get()` once and doesn't subscribe).
- `<FormProvider engine>`, `<ScopeProvider path>` (rows and data-scoped groups), and `<FormRenderer renderers />`.
- `get()` returns a reference that stays the same until the value changes, which `useSyncExternalStore` requires.
- Svelte note: the store contract requires `subscribe` to call the listener immediately with the current value. `Readable.subscribe` doesn't, so the Svelte adapter wraps it. Hosts can also use `@tanstack/*-store` hooks if they want, but our adapters don't depend on them.

## 4. Renderer registry

```ts
const renderers = defineRenderers({
  field: { text: TextInput, number: NumberInput, choice: Select, 'x-signature': SignaturePad },
  group: { step: WizardStep, section: Section, card: Card, tabs: Tabs, none: Fragment },
  array: { stack: StackArray, table: TableArray },
  'dynamic-group': { none: Fragment },   // optional; otherwise the group entries are used
  fallback: { field: UnknownField, group: Fragment },
});
```

- Each map entry is shorthand for a **tester** (below). The default lookup key is:
  - **field:** `fieldType:ui.variant` (for example `choice:radio`), then `fieldType`.
  - **group:** with `step: true`, `step:ui.variant` then `step`; otherwise `ui.variant`.
  - **array:** `ui.variant`.
  - **dynamic-group:** `ui.variant`, then the group lookup.
  - then the node's contract `fallback` field type, if any ([01 §3.5](./01-schema-contract.md#35-contract-fallbacks-for-custom-types));
  - then `fallback`.
- **`fallback` is optional** and comes in two forms:
  - **one component**, the catch-all for every node kind: `fallback: Unsupported`. It gets `{ controller, node, children? }` with `node` as the `CompiledNode` union, so it switches on `node.kind` (and must render `children` for containers);
  - **per kind**, more specific: `fallback: { field: UnknownField, group: Fragment, array?, 'dynamic-group'?, '*'? }`. Each key gets props typed for that kind. `'*'` is the catch-all for kinds not listed; `dynamic-group` falls back to `group` first.
  
  Whatever isn't covered uses the adapter's built-in fallback (§4.2).
- A renderer gets `{ controller, node, children? }`, where `node` is `controller.compiled.node`, typed for the field type and uses the hooks itself. That way it re-renders only for its own slices.
- Renderer props are typed from the field type registry ([01 §3.2](./01-schema-contract.md#32-field-type-registry)). A renderer registered for `choice` receives `node: FieldOf<'choice'>`, so `node.options` and `node.ui` (`ChoiceUi`, including merged extensions) are fully typed:

```ts
type FieldRendererProps<FT extends FieldType> = {
  controller: FieldController<ValueOf<FT>>;
  node: FieldOf<FT>;
};

const Select = ({ controller, node }: FieldRendererProps<'choice'>) => {
  const variant = node.ui?.variant ?? 'select';       // 'select' | 'radio' | 'buttons' | 'combobox' | `x-${string}`
  …
};

defineRenderers({ field: { choice: Select } });   // type error if Select expects a different fieldType
```
- `FormRenderer` walks the CompiledSchema. Invisible nodes render nothing by default. A renderer can opt into `renderHidden` for animations.

### 4.1 Ranked testers

Borrowed from JSON Forms ([13](./13-prior-art.md)). Any renderer can be registered with a tester that scores how well it fits a node:

```ts
type Tester = (node: CompiledNode, ctx: { platform: string; parent?: CompiledNode }) => number; // -1 = no match

renderers.register(LargeOptionsCombobox, (n) =>
  isField(n, 'choice') && n.node.options?.source === 'provider' ? 20 : -1);   // isField narrows n.node to FieldOf<'choice'>
renderers.register(CountryFlagSelect, (n) => (n.node['x-widget'] === 'country' ? 50 : -1));
```

- The map shorthand uses fixed ranks: exact `fieldType:ui.variant` (or the group/array key) = 10, `fieldType` = 5, `fallback` = 0. A tester that ranks above 10 overrides the defaults without replacing them.
- On a tie, the renderer registered later wins, so apps can override library renderer sets.
- The result is computed once per compiled node (testers must be pure), so there's no cost per render.
- Helpers: `rankWith(n, and(isFieldType('choice'), hasExt('x-widget', 'country')))`.

### 4.2 Missing renderers

A field type can be registered in the engine with no renderer on this client (for example, an older app receiving a newer contract). The engine doesn't care: the field keeps its controller, value, validation and submit behaviour.

- **Diagnostic.** When lookup reaches `fallback` for a field, the adapter reports `W_NO_RENDERER` through `onDiagnostic`, once per field type, with `detail: { fieldType, paths }`.
- **Built-in fallback.** For any kind the host's `fallback` doesn't cover (§4), adapters use their own. Which one depends on `platform.dev` (§6):
  - **Dev (fields):** a clearly marked box (dashed border, warning colour) with the field type, the path, the value as truncated JSON (about 200 characters), and a fix hint ("add a renderer, or a contract `fallback`, 01 §3.5"). Raw JSON is fine here; only developers see it.
  - **Prod (fields):** looks like a normal read-only field. Label and description as usual; the value as text (strings, numbers and booleans formatted normally, choice values by their option label); anything else, or an empty value, shows the built-in text `{ key: 'formgraph.unsupported', default: 'Not available on this device' }`, which the host's resolver can translate or reword ([09 §2](./09-localization.md#2-resolution)). Never raw JSON or the type name.
  - **Groups, arrays, dynamic groups (both modes):** render the children, with no wrapper.

  Both field fallbacks wire the label, `ids`, `testId` (§8.1), notices and `registerElement` (a focusable element, `tabIndex=-1` on web), so `firstInvalid`, focus and the error summary still reach them. A required, invalid field the user can't edit still shows why submit is blocked.
- **Overriding the built-ins.** They are defaults only:
  - set `fallback` (one component or per kind, §4) to replace them for the kinds you cover;
  - adapters export them (`DevFallback`, `ProdFallback`, and `BuiltInFallback`, which picks by `platform.dev`), so a host can wrap or reuse them, for example `fallback: { field: (p) => <Tracked><ProdFallback {...p} /></Tracked> }`;
  - set `platform.dev` explicitly to choose the mode;
  - reword the production text through the resolver key `formgraph.unsupported`.
- **Check before mounting.** `renderers.missing(schema): Array<{ fieldType: string; paths: DataPath[] }>` is pure and runs the same lookup without rendering. Hosts call it after `loadContract` to refuse the form, load another contract version, or show "update the app". Treating a non-empty result as fatal is the host's choice; the engine has no `strict` for it, because it doesn't know about renderers.
- **Validation is unchanged.** A `required` or otherwise invalid field without a renderer still blocks submit; the engine doesn't quietly drop rules for fields it can't show. A host that wants that can check `missing()` and exclude those paths from validation itself.

## 5. Wizards

Provided by the `wizard` plugin ([02 §8](./02-state-model.md#8-plugins)), which the default entry installs: `engine.wizard()` returns the `WizardController`. Fields in inactive steps compile lazily ([02 §6.3](./02-state-model.md#63-lazy-compile)).

Steps are `group` nodes with `step: true`. Their presentation (tabs, cards, etc.) is still free via `ui.variant`. A core `WizardController` (built on top of the engine) provides `steps`, `current`, `canNext` (validates only the current step's subtree; errors always block, warnings follow their gate's policy, see [05 §7.3](./05-validation.md#73-gates-submit-policy-for-warnings)), `next({ confirm? })`, `prev`, and `goTo`. Steps can be conditionally visible, and hidden steps are skipped.

```ts
next(opts?: { confirm?: string[] }): Promise<{
  ok: boolean;
  reason?: 'errors' | 'gated' | 'validating' | 'assets-pending' | 'last-step';
  gates?: GateStop[];                       // 08 §4
}>;
```

`next()` validates the current step with every applicable rule ([05 §3](./05-validation.md#3-triggers)), **awaits** its async validation and pending assets (per `submission.awaitAsync`, [07 §2.1](./07-async-data-and-registries.md#21-asset-stores)), and only then moves. On `ok: false` it stays on the step and marks its fields touched so messages show. `canNext` is the synchronous, `Readable<boolean>` estimate for enabling the button.

```ts
if ((await wiz.next()).ok) await drafts.put(key, engine.snapshot());
```

## 6. Platforms

The core is platform-agnostic (D24); everything platform-specific lives in adapters or host code.

**What the core may use.** ES2020 plus `Promise`, `Map`/`WeakMap`, `AbortController`/`AbortSignal` (constructor, `abort()`, `aborted`, `addEventListener` only; no `AbortSignal.any`/`timeout`), and `RegExp` with the `u` flag. It never touches `window`, `document`, DOM types, `requestAnimationFrame`, `structuredClone`, `TextEncoder`, `crypto`, `Blob`/`File` at runtime, Node built-ins, or `Intl`:
- `contractBytes` counts UTF-8 bytes with a small loop; hashes (`contractHash`, cache keys, sub-schema hashes) are a bundled pure-JS hash.
- Asset inputs (`Blob | PlatformFile`, [07 §2.1](./07-async-data-and-registries.md#21-asset-stores)) are passed through to the host's store untouched.
- The `icu` plugin needs `Intl.PluralRules`/`NumberFormat` and checks for them at install and throws a plain error naming the missing API; Hermes ships them. Core `{name}` formatting needs nothing.

**Platform hooks.** Timing is injected, defaulting to `globalThis`:

```ts
interface Platform {
  now(): number;                                        // default Date.now
  setTimeout(fn: () => void, ms: number): unknown;      // debounce, ttl, async validation delays
  clearTimeout(handle: unknown): void;
  nextFrame(fn: () => void): void;                      // focus wait (§8); default requestAnimationFrame if present, else setTimeout(fn, 0)
  dev: boolean;                                         // dev-only behaviour: onDiagnostic default (16 §1), built-in fallbacks (§4.2)
}
createFormEngine({ …, platform: { nextFrame: (fn) => InteractionManager.runAfterInteractions(fn) } });
```

`platform.dev` defaults, when the host doesn't set it, to `true` if `globalThis.__DEV__ === true` (React Native) or `globalThis.process?.env?.NODE_ENV === 'development'`, each read guarded so it can't throw; otherwise `false`. **Unknown means production**, so a misconfigured app gets the safe behaviour. Pass it explicitly when your bundler has its own flag: `platform: { dev: import.meta.env.DEV }`. It is read at runtime, so dev-only code isn't stripped from production bundles; the dev pieces are small enough that this is accepted.

`@formgraph/testing`'s clock is a `Platform` (§9).

**Opaque UI handles.** The core stores but never inspects:
- **elements** from `registerElement(el: unknown)`. Moving focus is the adapter's `FocusDriver`: `{ focus(el: unknown): boolean | Promise<boolean> }`. Web adapters call `el.focus()`; React Native calls `ref.current?.focus()` on a `TextInput` ref, and a host can pass its own driver (custom inputs, scroll-into-view in a `ScrollView`).
- **ids.** `controller.ids` are plain strings (§8). Web renderers wire `id`/`htmlFor`/`aria-*`; React Native renderers use `nativeID`, `accessibilityLabelledBy` and `testID`.

**Adapters and escape hatches.** React Native uses `@formgraph/react` with RN renderers; desktop (Electron/Tauri) uses the web adapters. Every adapter:
- accepts `focusDriver`, `idPrefix`/`idFormat(ids)` and `platform` overrides on its provider, and installs the `onSubmitFailed` default only if the host didn't pass one;
- exposes the raw `engine` and `controller` (`useFormEngine()`, `useField(...).controller`), so anything the hooks don't cover is reachable directly;
- renders nothing itself outside `<FormRenderer/>`, which is optional: hosts can lay out fields by hand with `useField`.

Fully native hosts (Swift/Kotlin) are out of scope for 1.0; the contract and semantics docs are the spec for any future port.

**Enforcement (M0).** The core and contract packages build with `lib: ["ES2020"]` (no `dom`) and a lint rule banning the globals above; CI runs the core test suite under Node and under Hermes.

## 7. Devtools

`@formgraph/devtools` subscribes to the engine and shows the layers per path (which one won), the DAG with the last tick's evaluated nodes highlighted, recent commits with origins (its own capped buffer, filled through `onCommit`, [02 §2.5](./02-state-model.md#25-commit-hooks)), and async request timelines.

## 8. Accessibility

```ts
interface FieldIds { input: string; label: string; description: string; feedback: string }
createFormEngine({ …, idPrefix: 'checkout' });   // optional; see below
// /addresses[#k3]/city → { input: 'checkout-addresses-_rk3-city', label: '…-label', description: '…-desc', feedback: '…-feedback' }
```

- **Collision-free encoding.** Each row-key-path segment is escaped: `[A-Za-z0-9]` pass through, `_` becomes `__`, and every other character becomes `_` + its UTF-8 bytes as two-digit lowercase hex (`-` → `_2d`, `/` → `_2f`). Segments are joined with `-`, and row-key segments are written `_r<escaped key>` (`_r` is never produced by escaping, since `r` isn't hex), so a name segment can never look like a row key. `/a-b` → `a_2db`, `/a/b` → `a-b`: distinct.
- **Default prefix.** Without `idPrefix`, the adapter supplies the framework's unique id per `<FormProvider>` (React `useId`, Vue `useId`, Solid `createUniqueId`, Svelte `$props.id()`). These are SSR-stable, so server and client ids still match; two forms on one page never collide.

- `controller.ids` is derived from the row key path, so ids are stable across reorders and identical on server and client (row keys come with `resume`, [08 §9](./08-loading-and-submission.md#9-server-rendering)). Web renderers wire `id`, `htmlFor`, `aria-describedby` (description + feedback when `showFeedback`), `aria-invalid` (`severity === 'error'`) and `aria-required`; React Native renderers map them to `nativeID`, `accessibilityLabelledBy` and `accessibilityState` (§6).
- `engine.firstInvalid(opts?: { include?: Array<'errors' | 'external' | 'conflicts'> }): DataPath | null` (default: all three). Returns the first failing node (field, group or array) in form order: an `error`-severity message (including `source: 'input'`), an external error, or an unresolved conflict, filtered by `include`. Hidden nodes are skipped unless they are validated (`validateHidden`, [06 §4](./06-effects-and-computed.md#4-hidden-and-disabled-field-policy)).
- **Focus targets.** `registerElement(el)` (on field, scope and array controllers) returns an unregister function; adapters call it on unmount. Registrations form a stack: the latest wins, and unregistering falls back to the previous one. Removing a row drops every element registered under it. Without a registration, the adapter may fall back to `ids.input`.
- **`controller.focus(): Promise<boolean>`** moves focus through the adapter's `FocusDriver` (§6) only (no meta change; `visited` comes from `handlers.onFocus()`). With no registered element it runs the `reveal(path)` hook (core expands collapsed sections; plugins such as `wizard` add their own, [02 §8.1](./02-state-model.md#81-plugin-api)), waits one `platform.nextFrame` (§6) for a registration, and resolves `false` if none arrives.
- **Failed submit.** `submit()` (and scoped validation run by a plugin) marks fields touched where it validated, then calls `onSubmitFailed(result)` from `createFormEngine` options for **every** `{ ok: false }` result ([08 §4](./08-loading-and-submission.md#4-submission-pipeline)). Adapters install a default that branches on `result.reason`:

  | `reason` | Default |
  |----------|---------|
  | `errors`, `external`, `conflicts` | focus `firstInvalid()`: the field itself, or for a group/array its first focusable child, else the container's own registered element. Nothing focusable → no-op; the failure stays in the result |
  | `gated`, `validating`, `assets-pending` | nothing (the host shows its own confirm/progress UI) |
  | `transform` | nothing (dev-mode log) |

  Passing your own `onSubmitFailed` replaces the default.

### 8.1 Test ids

End-to-end tests need selectors they can write before the test runs. Labels are translated and repeat in every row, and `ids` use runtime row keys (`_rk3`) plus a framework prefix, so neither works well. The engine supplies a separate, predictable id; renderers decide whether to put it on an element.

```ts
createFormEngine({ …, testIds?: { prefix?: string; format?: (path: DataPath) => string } | false });
// default format: `${prefix}:` + the DataPath with indexes as segments → "checkout:addresses.2.city" (no prefix → "addresses.2.city")
```

- **Index-based.** Built from the `DataPath` (`controller.path`, a `Readable` for the same reason), not the row key path, so `addresses.0.city` is "the first row" no matter which rows were added or moved. When a row moves, its `testId` changes with it (fields: `state.testId`; rows and groups: `ScopeController.testId`). That's the opposite of `ids`, which follow row identity for accessibility.
- **Renderers apply it:** web as `data-testid`, React Native as `testID`. Which elements get it (input, row container, step button) is the renderer's choice. To leave it out of production, pass `testIds: platform.dev ? {…} : false`; with `false`, `testId` is `''` and renderers skip the attribute.
- **Cost.** A string derived from a path the engine already has, computed lazily per controller.
- **First-party pieces use it:** the built-in fallbacks (§4.2) and the conformance suite (§9) check that it's passed through.
- `@formgraph/testing/e2e` exports `fieldSelector(path, { prefix?, format? })`, a pure function that builds the same id or a `[data-testid="…"]` selector. See [15 §24](./15-cookbook.md#24-end-to-end-tests).

## 9. Testing

`@formgraph/testing` (dev dependency):

```ts
import { createTestEngine, fakeEntitySource, fakeProvider } from '@formgraph/testing';

const t = await createTestEngine({
  contract: fixture('order@1'),                 // contract fixtures: inline or from a fixtures folder; loaded with loadContract
  registry: { entitySources: { order: fakeEntitySource(({ id }) => orders[id]) },
              providers: { cities: fakeProvider((query) => cityList[query.deps.country]) } },   // identity transformer: query = { state, query, deps }
  params: { id: 42 },
});
await t.ready;
t.engine.array('/items').insert();               // fill requires existing rows
t.fill({ '/customer/email': 'a@b.c', '/items[0]/qty': 3 });   // origin user, handlers.onFocus/onBlur around each (visited, touched)
await t.clock.advance(300);                      // fake timers; flushes microtasks after each timer
await t.settle();                                // run until no tick, timer or fake call is pending
t.expectFeedback('/customer/email', []).expectFeedback('/items[0]/qty', [{ code: 'max', severity: 'error' }]);
expect(t.submitted).toEqual([{ payload: { … }, mode: 'full' }]);   // recording fake under the contract's targetId
```

- **One generic fake.** Every source has the `AsyncSource` shape ([07 §1.1](./07-async-data-and-registries.md#11-shared-function-shapes)), so one fake covers all of them:

  ```ts
  fakeSource<I, O>(impl?: (input: I, ctx: SourceCtx) => O | Promise<O>): FakeSource<I, O>;
  // .calls: { input, ctx, aborted }[]   .delay(ms)   .fail(err)   .pending() → resolve/reject a held call by hand
  ```

  `fakeProvider`, `fakeEntitySource`, `fakeSchemaSource`, `fakeValidationSource`, `fakeLocaleSource`, `fakeSelectedResolver` and `fakeSubmitTarget` are typed aliases. Fakes honour `ctx.signal`: an aborted call rejects with `AbortError` and is recorded as `aborted`, so race and generation tests can assert on it.
- **Submit target.** Unless the test registers one, `createTestEngine` registers a recording `fakeSubmitTarget` under the contract's `targetId`; `t.submitted` reads its calls.
- **`fill`** writes leaves only. A path into a missing row throws, with a hint to `insert()` first; rows are never created implicitly.
- **`t.clock`** replaces timers and the scheduler; no real timers run. `advance` is async and flushes microtasks after each timer it fires.
- **Assets** (with the `assets` plugin): `fakeAssetStore()` holds each upload until the test drives it: `store.progress(id, 0.5); store.resolve(id, { ref: 'r1' })` or `store.fail(id, err)`.
- The in-memory registry starts with the built-ins. Snapshot helpers (`t.snapshot()`, `t.exportState()`) make draft and SSR round-trips one-liners.

## 10. Adapter conformance

`@formgraph/testing/adapter` ships a conformance suite every adapter (first- and third-party) runs in CI with its own mount function:

```ts
runAdapterConformance({ name: 'vue', mount: (engine, renderers) => mountVue(engine, renderers) });
```

It checks: one subscription per mounted field; no re-render when an unrelated field changes; selector re-renders only on selected changes; unsubscribe on unmount and row removal; row reorder keeps component state (keyed by `RowKey`); server render equals first client render after `resume` (no mismatch, same ids); `onSubmitFailed` default focuses the first invalid node per reason; `registerElement` stack and unregister on unmount/row removal; renderer registry ranking and fallbacks; built-in fallback and `W_NO_RENDERER` (§4.2); `testId` passed through and updated on row moves (§8.1); native computed values/effects re-run when a read handle changes and stop tracking on cleanup (read observer, [02 §2.6](./02-state-model.md#26-tracking-reads-for-framework-adapters)). An adapter that passes may call itself conforming.
