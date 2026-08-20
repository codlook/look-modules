# csv — CSV encode/decode for LOOK

RFC 4180-style CSV in pure LOOK (`string::` only). A row is an array of string fields
and a table is an array of rows. Fields containing a comma, a double-quote or a newline
are quoted, embedded quotes are doubled, and the decoder reverses all of it — including
quoted fields that span newlines.

## Install

```bash
lk module install github.com/codlook/look-modules/csv
```

```lk
use csv
```

## Use

```lk
use csv

$text = csv_encode([
    ["name", "note"],
    ["Ali", "likes a, b and c"],
    ["Ays", "said \"hi\""]
])
# name,note
# Ali,"likes a, b and c"
# Ays,"said ""hi"""

$rows = csv_decode($text)
# [["name","note"], ["Ali","likes a, b and c"], ["Ays","said \"hi\""]]
```

## API

| Function | Returns | Description |
|----------|---------|-------------|
| `csv_encode($rows)` | `string` | Encode an array of row-arrays to CSV text (CRLF line endings). |
| `csv_decode($text)` | `[[field, ...], ...]` | Parse CSV text back into rows, honouring quoting. |

## Notes

- Fields are treated as strings; convert numbers with `int()` / `float()` after decoding
  and to strings before encoding if needed (`"" . $n`).
- For header-based data, keep the header as the first row and slice it off after
  decoding.
- The decoder tolerates both `\r\n` and `\n` line endings.
