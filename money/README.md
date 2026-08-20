# money — integer money for LOOK

Store and compute money as **integer minor units** (cents / kuruş). Every operation
stays on integers, so there are no float rounding errors — the reason you should never
keep money in a float. Only formatting produces the `major.minor` split and a currency
symbol.

## Install

```bash
lk module install github.com/codlook/look-modules/money
```

```lk
use money
```

## Use

```lk
use money

$price = money_parse("19.99")        # -> 1999  (cents)
$line  = money_mul($price, 3)         # -> 5997
$total = money_add($line, 500)        # -> 6497
money_format($total, "USD")           # -> "$64.97"
money_format(-99, "TRY")              # -> "-₺0.99"
```

## API

| Function | Returns | Description |
|----------|---------|-------------|
| `money_parse($str)` | `int` | `"12.34"` → `1234`, `"12"` → `1200`. Two decimals; extra digits are truncated, one digit is padded. |
| `money_format($cents, $cur)` | `string` | `1234, "USD"` → `"$12.34"`. Handles the sign and pads the minor part. |
| `money_add($a, $b)` | `int` | Sum of two amounts. |
| `money_sub($a, $b)` | `int` | Difference. |
| `money_mul($cents, $qty)` | `int` | Amount times an integer quantity. |

## Symbols

`USD` → `$`, `EUR` → `€`, `GBP` → `£`, `TRY` → `₺`. An unknown code is prefixed as
`"CODE "` (e.g. `SEK 1.00`).

## Notes

- Keep amounts as integers end to end — parse once at the boundary, format once for
  display; never round a float in between.
- Two-decimal currencies are assumed. Zero-decimal (JPY) or three-decimal (KWD)
  currencies would need their own formatting.
