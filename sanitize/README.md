# sanitize — safe HTML for LOOK

Make untrusted HTML safe to render. LOOK has no DOM, so instead of the fragile
"strip the bad tags" regex that sanitizers get bypassed on, this **escapes everything
first and then re-enables only a fixed set of attribute-less formatting tags**. An
injected `<script>`, an `onerror=` handler, a `javascript:` href — all stay escaped,
because they never match an exact attribute-less tag token. That makes it provably
safe, not best-effort.

## Install

```bash
lk module install github.com/codlook/look-modules/sanitize
```

```lk
use sanitize
```

## Use

```lk
use sanitize

sanitize_text("<b>hi</b> <script>x</script>")     # -> "hi x"  (safe plain text)
sanitize_basic("<b>ok</b> <script>alert(1)</script>")
# -> "<b>ok</b> &lt;script&gt;alert(1)&lt;/script&gt;"
```

## API

| Function | Returns | Description |
|----------|---------|-------------|
| `sanitize_text($html)` | `string` | Strip **all** tags → trimmed, escaped plain text. The reliable default for user input like comments. |
| `sanitize_basic($html)` | `string` | Allow only safe, attribute-less formatting tags; everything else stays escaped. |

## Allowed by `sanitize_basic`

`b strong i em u s code pre p br ul ol li blockquote h1 h2 h3 h4 h5 h6`

Nothing with attributes is allowed — so there is no `href` / `src` / `on*` surface at
all. Need a link? Build it yourself with `html::attr` on a value you trust. Pairs
naturally with the [`markdown`](../markdown) module: render Markdown, then
`sanitize_basic` the result.
