# html_parser — HTML scraping helper for LOOK

A small, dependency-free module for pulling data out of HTML. It wraps LOOK's core
`html::` and `string::regex_*` builtins to give you a practical, BeautifulSoup-style
API — find elements by tag and class, read attributes, and extract text.

LOOK has no built-in DOM tree, so this is regex-based, best-effort parsing. It fits
well-formed, reasonably stable markup. Always escape any value you render back into a
page with `html::escape` (text) or `html::attr` (attribute context).

## Install

```bash
lk module install github.com/codlook/look-modules/html_parser
```

Then import it (installed module):

```lk
use html_parser
```

…or, as a local file during development:

```lk
use "html_parser.lk"
```

## Functions

| Function | Returns | Description |
|---|---|---|
| `html_parser_find_all($html, $tag, $class)` | `[element_html, …]` | Every `<tag … class="…class…">…</tag>` element, as full HTML strings. |
| `html_parser_find($html, $tag, $class)` | `element_html` \| `""` | The first matching element, or `""`. |
| `html_parser_attr($tag_html, $name)` | `value` \| `""` | The value of an attribute (`href`, `src`, `data-id`, …) in a tag/fragment. |
| `html_parser_text($html)` | `string` | Visible text with tags removed (via `html::strip`) and trimmed. |
| `html_parser_links($html)` | `[href, …]` | Every `href` value in the fragment. |

## Example — a listings scraper with de-duplication

This is the full flow: fetch a search page, extract each listing's title, price and
URL, and store new ones in the embedded SQLite database (duplicates are skipped by the
`UNIQUE` url). The selectors below match a listing-table layout like
`<tr class="searchResultsItem"> … <a class="classifiedTitle" href> … <span class="classifiedPrice"> … </tr>`.

```lk
use html_parser
use string

$db = db::connect("sqlite://listings.db")
db::exec($db, "CREATE TABLE IF NOT EXISTS listings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT, price TEXT, url TEXT UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)", [])

function scrape($db, $html) {
    $items = html_parser_find_all($html, "tr", "searchResultsItem")
    print("[+] listings found: " . count($items))
    $new = 0
    foreach ($items as $item) {
        $a     = html_parser_find($item, "a", "classifiedTitle")
        $title = html_parser_text($a)
        $url   = "https://www.example.com" . html_parser_attr($a, "href")
        $price = html_parser_text(html_parser_find($item, "span", "classifiedPrice"))

        $exist = db::query($db, "SELECT id FROM listings WHERE url = ?", [$url])
        if (count($exist) == 0) {
            db::exec($db, "INSERT INTO listings (title, price, url) VALUES (?, ?, ?)", [$title, $price, $url])
            print("    [new] " . $title . " | " . $price)
            $new = $new + 1
        }
    }
    print("[+] inserted: " . $new)
    return $new
}

# fetch the page — send a realistic User-Agent
$headers = ["User-Agent" => "Mozilla/5.0", "Accept-Language" => "tr-TR,tr;q=0.9"]
$res = http::get("https://www.example.com/search", $headers)
if ($res["status"] != 200) {
    print("[-] request blocked (status " . $res["status"] . ") — likely anti-bot protection")
    return
}
scrape($db, $res["body"])
```

Running `scrape()` twice over the same markup inserts on the first pass and skips
everything on the second (verified against a fixture of this layout):

```
[+] listings found: 3
    [new] Bahcelievler 3+1 Furnished Flat | 2.750.000 TL
    [new] Roadside Zoned Land            | 1.100.000 TL
    [new] 2019 Diesel Automatic          | 985.000 TL
[+] inserted: 3
# second pass → [+] inserted: 0   (UNIQUE url de-dup)
```

Run it on a schedule with the built-in scheduler (no cron needed):

```lk
timer::every(900000, function() {   # every 15 minutes (ms)
    print("scanning…")
    # call your scraper function here
})
```

## Notes & limits

- **Regex, not a DOM.** For deeply nested or malformed markup a real DOM parser is
  more robust; this module targets predictable listing/table markup.
- **Anti-bot.** Large sites (Cloudflare/Akamai) may block a plain `http::get`. Route
  requests through a proxy pool, or feed HTML from a headless-browser service into
  these functions.
- **Escaping.** Values you extract are raw HTML text; escape with `html::escape` before
  putting them back into a page.
