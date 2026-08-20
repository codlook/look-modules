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
lk migrate.lk up       # apply pending   (default when no argument)
lk migrate.lk status   # show applied/pending
lk migrate.lk down     # roll back the last one
```

This is the LOOK counterpart to `php artisan migrate` / `migrate:rollback` — a plain
`.lk` you run in the terminal. Migrations are explicit data in that file rather than
auto-discovered from a folder (LOOK has no directory-listing, and explicit beats magic).

> Requires a LOOK runtime with `args()`. On an older build, pass the command via the
> `MIGRATE_CMD` env var instead (`MIGRATE_CMD=up lk migrate.lk`).

## API

| Function | Returns | Description |
|----------|---------|-------------|
| `migrate_run($conn, $migrations)` | `[id, ...]` | Apply every not-yet-applied migration, in array order, each in its own transaction. Returns the ids newly applied. |
| `migrate_rollback($conn, $migrations)` | `id` \| `""` | Undo the most recently applied migration (run its `down`, drop its tracking row). `""` if nothing was applied. |
| `migrate_status($conn, $migrations)` | `[["id"=>.., "applied"=>bool], ...]` | Applied/pending state for each migration. |
| `migrate_applied($conn)` | `[id, ...]` | Ids already recorded as applied. |
| `migrate_cli($conn, $migrations)` | *(prints)* | Terminal dispatch on `MIGRATE_CMD` (`up`/`down`/`status`). |

## Notes

- Migrations are applied **in the order of the array**, so keep ids sortable
  (`0001_`, `0002_`, …) and append new ones at the end.
- Statements are explicit — a single string, or an array of strings. There is **no `;`
  splitting**, so a `;` inside a value or a trigger body is safe.
- Each migration runs inside a transaction, which is real on SQLite/PostgreSQL. **On
  MySQL, DDL auto-commits**, so a migration with several DDL statements isn't atomic
  there — keep one logical change per migration (one `CREATE TABLE`, or a table plus its
  index) and you avoid the half-applied case.
- Data migrations (`UPDATE`/backfill) are ordinary migrations too — just raw SQL in `up`
  with an appropriate `down` (or `"down" => ""` if irreversible).
- `id` is the identity of a migration; never re-use or rename one that has shipped.
