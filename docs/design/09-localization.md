# 09 · Localization

[← Index](./README.md) · Addresses review item C6

## 1. Contract

```ts
interface LocalizableString {
  key?: string;                               // external catalog key
  default: string;                            // message (§3.4); used if key missing/unresolved
  params?: Record<string, Json | Expression>; // interpolation values
}
type Text = string | LocalizableString;
```

Example:

```json
{ "key": "validation.minItems",
  "default": "{count, plural, one {Add at least # item} other {Add at least # items}}",
  "params": { "count": 2 } }
```

## 2. Resolution

`ResolvedText` is a plain `string`: the final, formatted text for the current locale. Handles that expose text (`label`, `Notice.message`, option labels) are already resolved, and re-resolve on `setLocale`.

The core never formats strings itself. The host provides a resolver:

```ts
type TextResolver = (t: { key?: string; default: string; params: Record<string, Json> }, locale: LocaleTag) => string | undefined;  // undefined = unknown key, fall through (§3.1)

createFormEngine({ ..., i18n: { locale: 'de-DE', fallbackLocales: ['en'], resolve: i18nextResolver } });
```

The default resolver (when none is given) knows no keys: it always returns `undefined`, so lookup goes straight to the contract bundles and then `default` (§3.1). Every step formats with the same formatter (§2.1).

Adapters' own built-in texts use keys under `formgraph.` (for example `formgraph.unsupported` in the production fallback, [10 §4.2](./10-rendering-and-adapters.md#42-missing-renderers)), so a resolver can translate or reword them like any other key.

### 2.1 Formatter

The core keeps the resolution pipeline (resolver, bundles, fallback chain, `params`) but formats `{name}` substitution only. The ICU subset formatter (§3.4) is the `icu` plugin, `@formgraph/engine/icu`, which the default entry installs ([02 §8](./02-state-model.md#8-plugins)).

```ts
import { createFormEngine } from '@formgraph/engine/core';
import { icu } from '@formgraph/engine/icu';
createFormEngine({ contract, registry, plugins: [icu()] });

// or reuse the host's parser, so only one ships:
createFormEngine({ …, i18n: { resolve: i18nextResolver, format: i18nextFormat } });
type TextFormat = (message: string, params: Record<string, Json>, locale: LocaleTag) => string;
```

- Precedence: `i18n.format` if given, else the `icu` plugin, else core `{name}`.
- **Never raw syntax.** With core `{name}` only, a message using `plural`, `select`, `selectordinal`, `number`, `date` or `time` fails with `E_PLUGIN_MISSING` naming `@formgraph/engine/icu`: at compile for contract text, or when a `LocaleSource` bundle arrives (phase `runtime`; that key falls through as if missing).
- Built-in validator messages in slim core use non-plural wording (`"Minimum items: {count}"`). The `icu` plugin replaces them with pluralized versions (`"{count, plural, one {Add at least # item} other {Add at least # items}}"`).
- The shipped i18next and FormatJS adapters export a `format` alongside the resolver.

**Text is plain text.** Every resolved string (labels, descriptions, messages, option labels) is plain text. Renderers must output it as text, never as HTML or Markdown, so contract and server text can't inject markup. Rich text is opt-in: the host registers a formatter (`registry.textFormatters.register('markdown', fn)`) and a node opts in with `ui.textFormat: 'markdown'`; the renderer then passes the resolved string through that formatter. Sanitizing its output is the formatter's job.

Adapter resolvers are thin: for example i18next's `t(key, { defaultValue: default, ...params })`, or FormatJS's `intl.formatMessage({ id: key, defaultMessage: default }, params)`.

## 3. Contract locales

Optional, and expected to be rare. Most hosts translate through their own resolver (§2). A contract may carry its own text, for example when CMS authors write labels the host app has never seen.

```ts
type LocaleTag = string;                      // BCP 47, see §3.2
interface LocaleBundle {
  translations?: { [key: string]: string };   // messages, as `default`
}

interface FormContract {
  locales?: Record<LocaleTag, LocaleBundle>;
  localeSourceId?: string;
  …
}
```

```json
"locales": {
  "en":    { "translations": { "addr.city": "City", "validation.minItems": "{count, plural, one {Add # item} other {Add # items}}" } },
  "de":    { "translations": { "addr.city": "Stadt" } },
  "de-AT": { "translations": { "addr.city": "Ort" } }
}
```

`LocaleBundle` is an object so per-locale settings (for example text direction) can be added later without a breaking change. Today it only holds `translations`.

### 3.1 Lookup order

For a `Text` with a `key`, in locale `L`:

Let the chain be `L` followed by its fallbacks (§3.2): `de-AT` → `de` → `i18n.fallbackLocales` (host-configured, e.g. `['en']`). For each locale `C` in the chain, in order:

1. **Host resolver** for exactly `C`. If it knows the key in `C`, its text wins. The host can always override CMS text.
2. **Contract bundle** for exactly `C`.

If nothing in the chain matches, **`default`** on the `LocalizableString` is used (always present, so lookup always ends there at the latest).

Interleaving per locale means a contract's `de-AT` text beats the host's `de` text, and neither side's fallback can hide a more specific translation on the other side.

**Resolver contract.** The engine walks the fallback chain itself, so the resolver must:
- answer for **exactly** the locale passed, with its own fallback disabled (i18next: `t(key, { lng: C, fallbackLng: false })`; FormatJS: look up `messages[C][key]` only);
- return `undefined` when it has no text for that key in that locale (i18next: `exists` check; FormatJS: missing id). It must not return the key itself or the `default`;
- be synchronous and pure for a given `(key, params, locale)`. Async catalogs are loaded by the host before `setLocale`, or through a `LocaleSource` (§3.3).

The shipped i18next and FormatJS adapters follow this contract. Contract text is formatted with the same formatter (§2.1) and `params` as `default`.

Plain `string` labels (no key) are never translated. Use `LocalizableString` when text must be localized.

### 3.2 Locale tags

Keys in `locales`, `setLocale`, and `fallbackLocales` are **BCP 47 tags**: `de`, `de-DE`, `pt-BR`, `zh-Hant-TW`, `es-419`. This is what `Intl`, browsers, iOS, and Android use.

- **Case is normalized** at load time, so `de-de`, `DE-de` and `de-DE` are the same tag (language lowercase, script title case, region uppercase). The engine does this itself and doesn't need `Intl.getCanonicalLocale`, which some JS engines lack.
- **Underscores are rejected** (`E_INVALID_LOCALE`, with a hint): `de_DE` is POSIX/Java style, not BCP 47. Hosts that get `de_DE` from the OS convert it before calling `setLocale`.
- **Duplicates after normalization** (`de-DE` and `de-de` in one contract) are `E_DUPLICATE_LOCALE`.
- **Fallback is by truncation**, dropping one subtag at a time: `zh-Hant-TW` → `zh-Hant` → `zh`, then `fallbackLocales` in order. Bundles are looked up per key along this chain, so `de-AT` only needs the keys that differ from `de`.
- There's no cross-region guessing: `de-AT` never falls back to `de-DE`. Put shared text in `de`.

### 3.3 Loading bundles separately

Shipping every locale in every contract can be heavy. Instead, set `localeSourceId` and register a source:

```ts
type LocaleSource = AsyncSource<{ locale: LocaleTag }, LocaleBundle | null>;   // 07 §1.1; null = no bundle for this tag
```

- The engine loads the active locale (and its fallback chain) on start and on `setLocale`. Inline `locales` and loaded bundles merge per key, loaded keys winning.
- **`setLocale` is atomic.** `engine.form`'s `locale` and every lookup keep using the **old** locale until all bundles in the new chain have loaded (or failed). Then the locale switches and all text re-resolves in one tick (§4). If a bundle fails, the switch still happens, `status` is `'error'`, and the missing keys fall through the chain to `default`.
- **Latest call wins.** Each `setLocale` bumps a generation and aborts the previous call's loads ([07 §3](./07-async-data-and-registries.md#3-load-lifecycle)); a slower earlier call can never switch the locale after a later one. `setLocale` returns a Promise that resolves when its switch is applied, or when it is superseded.
- `engine.form`'s `localeStatus` is `'loading' | 'ready' | 'error'` (`useForm('localeStatus')`). While it is `'loading'`, text shows the previous locale, so there's no flash of raw keys.
- Like every source, it may read a CDN, a bundled file, or storage ([D17](./README.md)).
- **Server rendering:** loaded bundles are exported in `EngineState.localeBundles` and installed on `resume` without calling the source, so the first client render matches the server ([08 §9](./08-loading-and-submission.md#9-server-rendering)).

### 3.4 Rules

- **Locale switches touch text only.** Values, meta, validity, and history are untouched. Validation messages are re-resolved, not re-run.
- **Macros** that generate their **own** text (not text passed in as arguments) must emit `LocalizableString`s with keys namespaced by the macro (for example `macro.choice-with-other.otherLabel`) and an English `default`. The contract can then translate them like any other key ([14](./14-macros.md)).
- **Load-time checks:** a `key` that no bundle provides for some declared locale is a warning (`W_MISSING_TRANSLATION`, `strict`: error), since `default` will show instead. A bundle key that no node references is `W_UNUSED_TRANSLATION`. Bundle values are parsed by the active formatter (§2.1) at load time, so a malformed message fails early (`E_INVALID_MESSAGE`).
- **Bundles from a `LocaleSource`** can't be checked at load time. The same checks run when each bundle arrives (phase `runtime`): `W_MISSING_TRANSLATION` and `W_UNUSED_TRANSLATION` are reported as dev warnings (never errors, even under `strict`, since the form is already running), and a malformed message is `E_INVALID_MESSAGE` for that key only. The key then falls through as if it were missing.
- **Supported ICU subset.** The `icu` plugin (§2.1) supports exactly:
  - simple arguments `{name}`;
  - `{n, plural, …}` with `=N` exact matches, the CLDR categories `zero`/`one`/`two`/`few`/`many`/`other` (via `Intl.PluralRules`), `offset:N`, and `#`;
  - `{n, selectordinal, …}`;
  - `{x, select, …}` with a required `other`;
  - `{n, number}` and `{n, number, integer | percent}`;
  - `{d, date, short | medium | long}` and `{d, time, short | medium}`;
  - apostrophe quoting (`'{'`, `''`).

  Anything else (number/date skeletons `::`, `spellout`, `duration`, rich-text tags `<b>…</b>`) is `E_INVALID_MESSAGE` at parse time. Hosts that need more supply their own `format` (§2.1).
- `locales` is not exported by `toPayloadSchema` and is ignored by `fromJsonSchema`.

## 4. Reactivity

- `params` that contain Expressions become DAG nodes (facet `label` / `message`), so labels like "Item {index}" or "Max {../limit}" update reactively.
- A locale change (`engine.setLocale`) invalidates every resolved text in a single tick.
- Validators provide their **param values** automatically: `min` passes `{ min }`, `max-length` passes `{ max, actual }`, and so on.

## 5. Option labels from data sources

Result transformers return `SelectOption.label` as `Text`. A provider that already localizes can return plain strings, and the resolver passes them through untouched.
