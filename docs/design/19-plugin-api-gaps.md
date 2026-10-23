# 19. Plugin API gaps (under discussion)

> Status: **open**. Nothing here is decided. These are possible gaps in the plugin API ([02 §8.1](./02-state-model.md#81-plugin-api)) recorded for later discussion. Until one is accepted, 02 is authoritative. G1–G3 likely need answers before M1, because built-in plugins rely on these abilities while third-party plugins can't use them. G4–G7 are design questions.

## Baseline

`PluginContext` currently exposes:
- `engine` (the public API),
- `store: PrivilegedStore` (transactions under any origin, all slices),
- `origins.register`,
- `meta.slice`,
- `use(CommitHooks)` (observe or veto, never rewrite),
- `fieldTypes.register` and `validators.register`,
- `text.setFormat` / `text.messages`,
- `onReveal`,
- `onDispose`.

`PluginInstance` returns `methods` (exposed as `engine.plugins[name]`, or at top level for built-ins) and `exportState`/`resumeState`.

## G1. Controllers can't be extended

`assets` adds `controller.assets` ([10 §2](./10-rendering-and-adapters.md#2-headless-field-api)), but `PluginContext` has no way to do that, so built-ins are quietly privileged. A third-party plugin can't add, for example, `controller['x-geo']`.

- Possible solution: `controllers.extend(name, (controller) => members, { fieldTypes? })`. Members would be created lazily with the controller and disposed with it. Third-party plugins would be limited to `x-` names, reachable as `controller.plugins[name]`, mirroring `engine.plugins`. Types would be inferred from `plugins` as for engine members.
- Open questions: should this cover field controllers only, or also scope and array controllers? How do adapters' `useField` hooks expose the extra members?

## G2. Two registration paths

Registration was settled as happening only on the append-only `Registry` ([07 §1](./07-async-data-and-registries.md#1-registries)), but `PluginContext` also has `fieldTypes.register` and `validators.register`.

- Possible solution: plugins register into the engine's per-engine child registry, with the same append-only rule. A plugin id that collides with a host id raises `E_REGISTRY_DUPLICATE`, naming the plugin.
- Alternative: plugins declare their registrations statically (`plugin.registers = { fieldTypes, validators }`), so `loadContract` can see them before an engine exists. This matters because compile-time checks, such as unregistered `x-` types and `E_PLUGIN_MISSING`, run before setup.
- Open question: how does `loadContract` (with no engine) learn about plugin-provided types and validators?

## G3. `PrivilegedStore` is too broad for third-party plugins

It can commit under built-in origins (`load`, `user`, `restore`, …) and write any slice. That bypasses the `x-` origin rule and lets one plugin corrupt another's state.

- Possible solution: built-ins keep `PrivilegedStore`. Third-party plugins get a `PluginStore` that can commit only under the plugin's own registered `x-` origins, write values and its own meta slices, and read everything.
- Open questions: does any realistic third-party plugin need to write `errors` (for example to add form-level messages)? If so, through which narrow API?

## G4. No submit lifecycle hooks

Autosave, audit, and analytics plugins need "before submit", "submit succeeded", and "submit failed". Today the closest options are `onCommit`, registry transformers, and the host's `onSubmitFailed`.

- Possible solution: `submission.on('before' | 'success' | 'failure', fn)`, observe-only. `before` could add a veto that returns `reason: 'x-…'`.
- Open questions: should a veto be allowed, or only observing? What order do these hooks run in relative to transformers and the host's `onSubmitFailed`?

## G5. No form-level validation participation

Plugins can't add form-wide checks or join the pending-work gate except through a field type's `pending` hook.

- Possible solution: `validation.onValidate(reporter, fn)`, the engine-level counterpart of the controller hook ([10 §2.2](./10-rendering-and-adapters.md#22-component-reported-messages-and-busy-state)). Results would go to `FormState.notices`, or onto paths when messages carry one. Plus `busy(token, flag)` at engine level.
- Open question: does this overlap too much with group rules on the root node?

## G6. Only one text formatter

`text.setFormat` holds a single formatter, so two plugins that both set one silently overwrite each other.

- Option A: a second `setFormat` throws a new error naming both plugins.
- Option B: formatters form a chain, each handling what it recognizes and passing the rest on.
- Open question: do `text.messages` defaults from several plugins merge, and with what precedence?

## G7. No contract or compile hooks

Plugins can't declare their own contract keys or check them at load time. Only `x-` keys pass through, unchecked.

- Option A: state explicitly that plugins don't extend the contract format beyond field types, validators, and `x-` keys (D13, small core).
- Option B: allow plugins to provide a Standard Schema for their `x-<plugin>` node key, checked in validate-core like field config.
- Open question: if B, does `loadContract` need the plugin list (same problem as G2)?
