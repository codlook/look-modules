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

A migration is an assoc with a sortable `id` and an `up` body (one or more statements
separated by `;`):

```lk
$migrations = [
    ["id" => "0001_users", "up" => "CREATE TABLE users (id INTEGER PRIMARY KEY, name VARCHAR(100))"],
    ["id" => "0002_posts", "up" => "CREATE TABLE posts (id INTEGER PRIMARY KEY, title VARCHAR(200)); CREATE INDEX idx_posts_title ON posts(title)"]
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

Give a migration an optional `down` body and you can undo the most recent one:

```lk
$migrations = [
    ["id" => "0001_users",
     "up"   => "CREATE TABLE users (id INTEGER PRIMARY KEY, name VARCHAR(100))",
     "down" => "DROP TABLE users"]
]

migrate_rollback($conn, $migrations)   # -> "0001_users"  (runs the down, un-tracks it)
```

## Terminal usage (no artisan needed)

LOOK scripts don't take CLI arguments, so the command is selected with an env var.
Put your migrations in a small runner and call `migrate_cli`:

```lk
# migrate.lk
use migrate
use db
migrate_cli(db::connect(env("DB_DSN")), [
    ["id" => "0001_users", "up" => "CREATE TABLE users (id INTEGER PRIMARY KEY)", "down" => "DROP TABLE users"]
])
```

```bash
MIGRATE_CMD=up     lk migrate.lk    # apply pending   (default)
MIGRATE_CMD=status lk migrate.lk    # show applied/pending
MIGRATE_CMD=down   lk migrate.lk    # roll back the last one
```

This is the LOOK counterpart to `php artisan migrate` / `migrate:rollback` — a plain
`.lk` you run in the terminal. Migrations are explicit data in that file rather than
auto-discovered from a folder (LOOK has no directory-listing, and explicit beats magic).

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
- Each migration runs in a transaction. On MySQL, DDL statements auto-commit, so a
  failure part-way through a multi-statement DDL migration may leave earlier statements
  applied — split risky DDL into separate migrations.
- `id` is the identity of a migration; never re-use or rename one that has shipped.
