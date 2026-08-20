# i18n — translation for LOOK

Dictionary-based translation, **stateless and explicit**: there is no hidden global
"current locale". You hold the dictionary and the active locale (for example in the
session) and pass them in. `{placeholder}` tokens are filled from a params assoc, and
a missing key falls back to a default locale, then to the key itself.

## Install

```bash
lk module install github.com/codlook/look-modules/i18n
```

```lk
use i18n
```

## Use

```lk
use i18n

$dict = [
    "tr" => ["hello" => "Merhaba {name}", "bye" => "Güle güle"],
    "en" => ["hello" => "Hello {name}",   "bye" => "Bye"]
]

i18n_t($dict, "tr", "hello", ["name" => "Ali"])   # -> "Merhaba Ali"
i18n_t($dict, "fr", "bye", [])                     # -> "Bye"  (fallback to en)
i18n_t($dict, "en", "missing", [])                 # -> "missing"  (key echoed)
```

## API

| Function | Returns | Description |
|----------|---------|-------------|
| `i18n_t($dict, $locale, $key, $params)` | `string` | Translate `$key` in `$locale`; fall back to `I18N_FALLBACK` (default `en`), then the key. Pass `[]` for `$params` when there is nothing to interpolate. |
| `i18n_has($dict, $locale, $key)` | `bool` | Is `$key` defined for `$locale`? |
| `i18n_locales($dict)` | `[locale, ...]` | The locales present in the dictionary. |

## Notes

- The fallback locale is read from the `I18N_FALLBACK` env var (default `en`).
- Load your dictionary however you like — inline, or from JSON files with
  `file::read` + `json::decode`. i18n only reads it.
