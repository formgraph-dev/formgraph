# 20. Plugin API gaps (resolved)

> Status: **resolved**. Every gap below was decided and written into [03 §8](./03-state-model.md#8-plugins) (decision D28 in the [README](./README.md)). This file keeps the reasoning.

| Gap | Decision | Where |
|-----|----------|-------|
| G1. Controllers can't be extended | `ctx.controllers.extend(create, { fieldTypes? })` adds `controller.plugins[name]` on field controllers. State lives in the plugin's meta slice keyed by row key, so members follow row moves and are disposed with the row. `x-` names only for third parties. Scope and array controllers aren't covered yet; they can be added without breaking anything | 03 §8.1 |
| G2. Two registration paths | Plugins are factories recorded in the `Registry` (`registry.plugin(factory)`). Types and validators are declared statically (`registers`), so `loadContract` sees them with no engine; `setup` runs once per engine with that engine's options. `PluginContext.fieldTypes/validators.register` removed | 03 §8, 08 §1 |
| G3. `PrivilegedPluginStore` too broad | Built-ins keep it. Third parties get `PluginStore` (read all; commit under own `x-` origins; values and own slices) and `ctx.notices.set` for form-level messages (`source: 'plugin'`) | 03 §8.1, 06 §5 |
| G4. No submit hooks | `ctx.submission.on('before' \| 'success' \| 'failure')`. `before` may stop with an `x-` reason; the others observe. Failure hooks run before the host's `onSubmitFailed` | 09 §4.1 |
| G5. No form-level validation | Not added. Group rules on the root node, the component `report`/`onValidate` hooks and `notices.set` cover it; the pending gate stays a field type hook | 03 §8.1 |
| G6. One text formatter | A second `setFormat` throws `E_PLUGIN_CONFLICT`. `text.messages` defaults merge: host first, then the first-installed plugin | 03 §8.1, 17 |
| G7. No contract hooks | Plugins don't extend the contract format beyond field types, validators and `x-` keys. The engine doesn't check `x-` keys; their user checks them where it reads them | 01 §1, 03 §8.1 |
