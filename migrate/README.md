# migrate — schema migrations for LOOK

An honest, dependency-free migration runner. You pass an **ordered list of migrations
as plain data**; `migrate_run` applies the ones that haven't run yet, each inside a
transaction, and records them in a tracking table. There is no ORM and no model layer —
your SQL stays SQL, and nothing is hidden. The tracking table uses only `VARCHAR`
columns, so the same code runs on MySQL, PostgreSQL and SQLite.

## Install

```bash
lk module install github.com/codlook/look-modules/migrate
```

```lk
use migrate
```

## A migration

A migration is an assoc with a sortable `id` and an `up` body. The body is **one SQL
string**, or an **array of strings** for several statements — explicit, with no `;`
parsing (a `;` inside a string value or a trigger body must never split a statement):

```lk
$migrations = [
    ["id" => "0001_users", "up" => "CREATE TABLE users (id INTEGER PRIMARY KEY, name VARCHAR(100))"],
    ["id" => "0002_posts", "up" => [
        "CREATE TABLE posts (id INTEGER PRIMARY KEY, title VARCHAR(200))",
        "CREATE INDEX idx_posts_title ON posts(title)"
    ]]
]
```

## Schema migrations — portable DDL

Raw SQL is fine, but a `CREATE TABLE` written by hand isn't portable: `AUTOINCREMENT`
(SQLite) vs `AUTO_INCREMENT` (MySQL) vs `BIGSERIAL` (PostgreSQL). Describe the table
**declaratively** and migrate compiles the right DDL for whatever database you're
connected to (via `db::driver`). There are exactly three schema operations —
`create_table`, `add_column`, `add_index` — and **everything else is raw `up` SQL, an
equal first-class path** (see below). Schema for schema, SQL for data; both are normal.

Columns read top-to-bottom like a struct — `"name" => "type spec"`:

```lk
$migrations = [
    ["id" => "0001_users", "create_table" => "users", "columns" => [
        "id"         => "pk",                    # auto-increment PK per dialect
        "name"       => "string(100)",           # VARCHAR(100) NOT NULL
        "email"      => "string(150) unique",    # + UNIQUE
        "age"        => "int null",              # nullable
        "created_at" => "datetime"
    ]],
    ["id" => "0002_avatar", "add_column" => "users",
        "column" => ["avatar", "string", 255, ["null" => true]]],
    ["id" => "0003_email_idx", "add_index" => "users", "columns" => ["email"]]
]
```

**Column spec:** `"name" => "<type>[(<size>)] [modifier ...]"`
- **types:** `pk string int bigint text datetime date bool float json` — or a raw SQL type
  as an escape hatch, e.g. `"price" => "decimal(10,2)"` (passed through verbatim).
- **(size):** VARCHAR length for `string` (`"string(255)"`).
- **modifiers:** `unique` · `null` (nullable) · `default=<val>`. Columns are **NOT NULL by
  default** (strict-default: most columns are, so you don't repeat it); add `null` to opt
  out. Compare GORM's `` `gorm:"type:varchar(255);unique;not null"` `` — here it's just
  `"string(255) unique"`.
- **fail-loud:** an unknown modifier (a typo like `"uniqe"`) **throws** — it is never
  silently dropped. Indexes are a separate `add_index` migration, not a column flag.

The explicit array form is also accepted (`["email", "string", 150, ["unique"=>true]]`) —
use whichever reads better; both compile to the same DDL.

Schema migrations **roll back automatically** — `create_table` → `DROP TABLE`,
`add_column` → `DROP COLUMN`, `add_index` → `DROP INDEX` (dialect-correct). You never
write a `down` for them (though an explicit `down` still wins if you provide one).

Inspect the generated DDL without a database with `migrate_ddl($driver, $migration)`
(`$driver` is `"sqlite"`, `"mysql"` or `"postgres"`).

> Why declarative, beyond portability: this same table description is the metadata
> **LookAdmin** reads to build its CRUD forms and tables — a concrete second consumer,
> not just a "might need it" bet.

### Data migrations stay raw — and that's normal

A data change (backfill, rename, a column type tweak DDL doesn't cover) is written as raw
`up` SQL, exactly like a schema op — not a fallback, an equal path:

```lk
["id" => "0004_backfill_slug", "up" => "UPDATE posts SET slug = LOWER(title) WHERE slug IS NULL",
 "down" => ""]   # data migrations are usually irreversible → explicit empty down
```

## Run

```lk
use migrate
use db

$conn = db::connect(env("DB_DSN"))

$applied = migrate_run($conn, $migrations)   # -> ["0001_users", "0002_posts"] on first run
# run again → [] (already applied; idempotent)

foreach (migrate_status($conn, $migrations) as $s) {
    print($s["id"] . " => " . ($s["applied"] ? "applied" : "pending"))
}
```

## Rollback

Give a migration a `down` body to undo the most recent one:

```lk
["id" => "0001_users",
 "up"   => "CREATE TABLE users (id INTEGER PRIMARY KEY, name VARCHAR(100))",
 "down" => "DROP TABLE users"]

migrate_rollback($conn, $migrations)   # -> "0001_users"  (runs the down, un-tracks it)
```

Rollback is **explicit about irreversibility**: a migration with **no `down` key**
throws when you try to roll it back (so a schema-changing migration is never silently
un-tracked). If a migration genuinely can't be undone (a data backfill, say), say so on
purpose with `"down" => ""` — that un-tracks it and runs nothing.

## Terminal usage (no artisan needed)

Put your migrations in a small runner that calls `migrate_cli`, then run it with the
command as the first argument (via LOOK's `args()`):

```lk
# migrate.lk
use migrate
use db
migrate_cli(db::connect(env("DB_DSN")), [
    ["id" => "0001_users", "up" => "CREATE TABLE users (id INTEGER PRIMARY KEY)", "down" => "DROP TABLE users"]
])
```

```bash
lk migrate.lk make user   # scaffold a new migration (files + paste-ready entry)
lk migrate.lk up          # apply pending   (default when no argument)
lk migrate.lk status      # show applied/pending
lk migrate.lk down        # roll back the last one
```

`make` comes with the module — it's a sub-command of `migrate_cli`, so a new module
version brings it without touching your `migrate.lk`. It scaffolds a **timestamp-named**
migration (`migrate/20260815_143022_user.sql` + `.down.sql`) and prints the entry to paste
into your migrations list — both the portable `create_table` schema (recommended) and the
raw-SQL `file::read` form. Timestamp ids (like Rails/Laravel) need no counter, so the
missing directory-listing isn't a limitation, and two developers never clash on the same
number. The name is validated (`^[a-z0-9_]+$`, so `make ../../etc/passwd` is rejected) and
existing files are never overwritten.

Adding the printed entry to `migrate.lk` is the one manual step — there is no
auto-discovery (LOOK has no directory-listing, and explicit beats magic). This is the LOOK
counterpart to `php artisan make:migration` / `migrate` / `migrate:rollback` — a plain
`.lk` you run in the terminal, where you can always see which file runs.

> **Known limitation:** because nothing scans the folder, a migration file you create but
> forget to add to the list simply **won't run** — it's skipped silently, not flagged.
> `make` prints the entry precisely so you paste it and don't forget. The migrations that
> run are exactly the ones in your list, no more.

> Requires a LOOK runtime with `args()`. On an older build, pass the command via the
> `MIGRATE_CMD` env var instead (`MIGRATE_CMD=up lk migrate.lk`).

### Your own CLI — mix migrate with your commands

`migrate_cli` is a convenience. For full control, skip it and write your own dispatch on
`args()`, calling the module functions (`migrate_run`, `migrate_status`,
`migrate_rollback`) directly — then you own every command and a module re-install never
touches your file:

```lk
# app.lk — your project's CLI, framework-free
use migrate
use db

$conn = db::connect(env("DB_DSN"))
$migrations = [
    ["id" => "0001_users", "create_table" => "users", "columns" => [
        "id" => "pk", "email" => "string(255) unique", "created_at" => "datetime"
    ]]
]

$a = args()
$cmd = count($a) > 0 ? $a[0] : "help"

if      ($cmd == "migrate") { migrate_run($conn, $migrations); print("migrated") }
else if ($cmd == "seed")    { db::exec($conn, "INSERT INTO users (email,created_at) VALUES (?,?)", ["a@x", "2026-01-01 00:00:00"]); print("seeded") }
else if ($cmd == "fresh")   { db::exec($conn, "DROP TABLE users", []); migrate_run($conn, $migrations); print("fresh") }  # ⚠ dev only — deletes data
else    { print("usage: lk app.lk [migrate | seed | fresh]"); exit(1) }
```

```bash
lk app.lk migrate
lk app.lk seed
```

## API

| Function | Returns | Description |
|----------|---------|-------------|
| `migrate_run($conn, $migrations)` | `[id, ...]` | Apply every not-yet-applied migration, in array order, each in its own transaction. Returns the ids newly applied. |
| `migrate_rollback($conn, $migrations)` | `id` \| `""` | Undo the most recently applied migration (run its `down`, drop its tracking row). `""` if nothing was applied. |
| `migrate_status($conn, $migrations)` | `[["id"=>.., "applied"=>bool], ...]` | Applied/pending state for each migration. |
| `migrate_applied($conn)` | `[id, ...]` | Ids already recorded as applied. |
| `migrate_cli($conn, $migrations)` | *(prints)* | Terminal dispatch on the first CLI arg (`make <name>`/`up`/`down`/`status`). |
| `migrate_ddl($driver, $migration)` | `string` | The DDL a schema migration compiles to for `$driver` (`"sqlite"`/`"mysql"`/`"postgres"`) — for inspection/testing without a DB. |

## Notes

- Migrations are applied **in the order of the array**, so keep ids sortable
  (`0001_`, `0002_`, …) and append new ones at the end.
- Statements are explicit — a single string, or an array of strings. There is **no `;`
  splitting**, so a `;` inside a value or a trigger body is safe.
- Migrations are **not** wrapped in an explicit transaction: DDL auto-commits on MySQL
  anyway (so a wrapper is false assurance), and wrapping interfered with PostgreSQL. Keep
  **one logical change per migration** (one `create_table`, or a table plus its index)
  and application stays predictable — the statement runs, then the tracking row records
  it. Schema migrations require a runtime with `db::driver()` (to pick the dialect).
- Data migrations (`UPDATE`/backfill) are ordinary migrations too — just raw SQL in `up`
  with an appropriate `down` (or `"down" => ""` if irreversible).
- `id` is the identity of a migration; never re-use or rename one that has shipped.
