# markdown — Markdown → HTML for LOOK

A practical, dependency-free Markdown renderer built on the core `string::` and
`html::` builtins. It covers the common subset — headings, **bold**/*italic*, inline
and fenced code, links, images, ordered and unordered lists, blockquotes, horizontal
rules and paragraphs. LOOK has no full CommonMark parser, so this is best-effort, like
`html_parser`.

## Install

```bash
lk module install github.com/codlook/look-modules/markdown
```

```lk
use markdown
```

## Use

```lk
use markdown

$html = markdown_to_html("# Hello\n\nSome **bold** and a [link](https://look.codlook.com).")
# <h1>Hello</h1>
# <p>Some <strong>bold</strong> and a <a href="https://look.codlook.com">link</a>.</p>
```

## API

| Function | Returns | Description |
|----------|---------|-------------|
| `markdown_to_html($src)` | `string` | Render a Markdown document to HTML. |
| `markdown_inline($text)` | `string` | Render only inline spans (bold/italic/code/link/image) of a single fragment. |

## Security

The source is **HTML-escaped before rendering**, so an injected `<script>` or any raw
tag becomes inert text rather than live markup. Common dangerous URL schemes
(`javascript:`, `data:`, `vbscript:`) are neutralised in links. For fully untrusted
input, still pass the output through the [`sanitize`](../sanitize) module — regex
rendering is best-effort, and defence in depth is cheap.

## Supported syntax

- `#` … `######` headings
- `**bold**` / `__bold__`, `*italic*`, `` `code` ``
- `[text](url)`, `![alt](url)`
- `- ` / `* ` unordered lists, `1. ` ordered lists
- `> ` blockquotes
- ` ``` ` fenced code blocks
- `---` / `***` horizontal rules
