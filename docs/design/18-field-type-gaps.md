# 18. Field type gaps (under discussion)

> Status: **open**. Nothing here is decided. These are possible gaps in the core field types ([01 §3.2](./01-schema-contract.md#32-field-type-registry)) recorded for later discussion. Until one is accepted, 01 is authoritative.

## Baseline

The eight core types (`text`, `number`, `boolean`, `date-time`, `choice`, `multi-choice`, `asset`, `multi-asset`) are believed to be sufficient as **value types**. Most commonly requested "types" already map to them:

| Request | Covered by |
|---------|------------|
| email, phone, url, password, color, OTP | `text` + `ui.inputMode` / `autocomplete` / `variant` + validation rules |
| rating, slider | `number` + `ui.variant` |
| search, cascading select | `choice` + option sources ([12](./12-option-sources-and-cascading.md)) |
| address, date range | `group` (+ cross-field rule for ranges) |
| signature, rich text, geolocation, key-value | `x-` custom types ([01 §3.4](./01-schema-contract.md#34-custom-field-types-and-typed-ui-extensions)) |

The items below are gaps *within* this baseline.

## G1. Display-only node

There is no node for a heading, a paragraph of help text, or a divider. Today you have to misuse a field to get one.

- Possible solution: a `kind: 'content'` node with `label`, `description`, `layout`, and `conditions.visible`. It holds no value, is never submitted, and is never validated.
- Open questions: should it have a `ui.variant` (`heading`/`text`/`divider`)? Can `required`/`disabled` conditions appear on it, or are they an error?

## G2. Hidden or constant values

Fixed values such as a tracking id or a `source` constant currently need a field with a visibility condition.

- Possible solution: a cookbook recipe using `text` with `conditions.visible: false` and `whenHidden: 'keep'`.
- Alternative: a value-only node or a `hidden: true` flag.
- Open questions: how does it interact with `defaultValue` expressions and computed values? Is a recipe enough?

## G3. `date-time` min/max

`number` has `min`/`max` as implied rules, but `date-time` doesn't, even though "not in the past" and "after the start date" are common.

- Possible solution: `DateTimeConfig.min` / `max`, each either an ISO string or an `Expression`, implying `min`/`max` rules. Export as `formatMinimum`/`formatMaximum`, or leave them out of the payload schema.
- Open questions: how should values be compared across granularities and timezones? Which implementation of "now" should be used (`Platform.now`, D24)?

## G4. `text.minLength`

Only `maxLength` is a config key, which is an asymmetry.

- Possible solution: `TextConfig.minLength`, implying a `min-length` rule and exporting as `minLength`.
- Open question: does an empty string skip the check (so it only applies together with `required`)?

## G5. `boolean` with no "unanswered" state

The value is always `true` or `false`, so a yes/no question can't be unanswered, and it's unclear what `required` means.

- Possible solution: make the value `boolean | null` with an empty value of `null`. `required` would mean "answered", and "must accept the terms" would use an explicit `equals: true` rule.
- Open questions: is this a breaking change for checkbox renderers? Should the empty value be configurable per field instead (`empty: false`)?

## G6. Money and decimals

`number` is a floating-point number, which is risky for currency.

- Option A: a `decimal: 'string'` config on `number` that stores the value as a decimal string.
- Option B: keep money out of the core, provide an `x-money` type as a recipe, and possibly ship it as a plugin later. This option keeps the core small (D13).
- Open question: do `min`/`max`/`step` and computed sums need decimal arithmetic in JSON Logic (connected to Q12)?

## G7. `timezone` storage

The docs don't say whether a stored `date-time` value includes an offset.

- Possible solution: `utc` and IANA zones store the value with an offset (`2026-10-21T09:00:00+02:00`). `local` stores a local time with no offset (`2026-10-21T09:00:00`). `granularity: 'date'`/`'time'` never carry an offset.
- Open questions: does an IANA-zone value need the zone id stored alongside it? How does this work without `Intl` in the core (D24)?
