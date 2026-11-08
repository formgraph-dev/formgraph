# 19. Field type gaps (resolved)

> Status: **resolved**. Every gap below was decided and written into [01](./01-schema-contract.md) and the docs it points to. This file keeps the reasoning.

The eight core types (`text`, `number`, `boolean`, `date-time`, `choice`, `multi-choice`, `asset`, `multi-asset`) stay the **value types**. Most requested "types" are covered by them:

| Request | Covered by |
|---------|------------|
| email, phone, url, password, color, OTP | `text` + `ui.inputMode` / `autocomplete` / `variant` + validation rules |
| rating, slider | `number` + `ui.variant` |
| search, cascading select | `choice` + option sources ([13](./13-option-sources-and-cascading.md)) |
| address, date range | `group` (+ cross-field rule for ranges) |
| signature, rich text, geolocation, key-value | `x-` custom types ([01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions)) |

| Gap | Decision | Where |
|-----|----------|-------|
| G1. Display-only node | New node kind `content` (heading / text / divider / `x-` variants): no value or data path, `conditions.visible` only, own renderer kind and `ContentController`, plain-text built-in fallback | 01 §3.6, 11 §2, §4.2 |
| G2. Hidden or constant values | No core change: the `variable` macro (hidden computed field, `whenHidden: 'exclude'`) or a `text` field with `visible: false` and `whenHidden: 'keep'` | 15 §7, 16 §2 |
| G3. `date-time` min/max | `DateTimeConfig.min` / `max` (ISO string or Expression, `$context.now` for "now"), implied `min`/`max` rules, exported as `formatMinimum`/`formatMaximum` for literal bounds; offsets compare as instants | 01 §3.3.1 |
| G4. `text.minLength` | Added; implies `min-length`; an empty string passes | 01 §3.3.1 |
| G5. Unanswered `boolean` | `boolean` unchanged (empty `false`, `required` = must be true). A yes/no question that can be unanswered is a `choice` with `true`/`false` options: recipe and `yes-no` macro | 16 §25, 15 §7 |
| G6. Money and decimals | Out of the core: an `x-money` custom type as a recipe, a macro or later a plugin. No decimal arithmetic in the core | 01 §3.4 |
| G7. `timezone` storage | `utc` and IANA zones store an offset, `local` none, `date`/`time` never; the zone id isn't stored; IANA without `Intl` is `E_TIMEZONE_UNSUPPORTED` | 01 §3.3.1 |
