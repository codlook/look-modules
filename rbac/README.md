# rbac — role-based access control for LOOK

Roles, permissions and their links, straight over `db::`. Every access decision is a
plain SQL `JOIN` you could run by hand — no ORM, no hidden policy engine. A role may
hold the wildcard permission `*` to grant everything. Four namespaced tables
(`rbac_role`, `rbac_permission`, `rbac_role_permission`, `rbac_user_role`) are created
on setup, using only `VARCHAR` columns so the same code runs on MySQL, PostgreSQL and
SQLite.

## Install

```bash
lk module install github.com/codlook/look-modules/rbac
```

```lk
use rbac
```

## Use

```lk
use rbac
use db

$conn = db::connect(env("DB_DSN"))
rbac_setup($conn)                      # create tables (idempotent)

rbac_grant($conn, "editor", "post.edit")
rbac_grant($conn, "editor", "post.view")
rbac_grant($conn, "admin",  "*")       # wildcard — every permission

rbac_assign($conn, $user_id, "editor")

if (rbac_can($conn, $user_id, "post.edit")) {
    # allowed
}
```

Guard a route in one line:

```lk
route("POST", "/admin/posts", function() {
    if (!rbac_can($conn, session::get("user_id"), "post.edit")) {
        return response::error(403, "Forbidden")
    }
    # ...
})
```

## API

| Function | Returns | Description |
|----------|---------|-------------|
| `rbac_setup($conn)` | — | Create the four tables (idempotent). |
| `rbac_add_role($conn, $role)` | — | Register a role. |
| `rbac_add_permission($conn, $perm)` | — | Register a permission. |
| `rbac_grant($conn, $role, $perm)` | — | Give a permission to a role (`*` = all). Idempotent. |
| `rbac_revoke($conn, $role, $perm)` | — | Take a permission back. |
| `rbac_assign($conn, $user_id, $role)` | — | Give a role to a user. Idempotent. |
| `rbac_unassign($conn, $user_id, $role)` | — | Take the role away. |
| `rbac_can($conn, $user_id, $perm)` | `bool` | Does the user hold `$perm` via any role (or `*`)? |
| `rbac_roles($conn, $user_id)` | `[role, ...]` | Roles assigned to the user. |
| `rbac_permissions($conn, $role)` | `[perm, ...]` | Permissions granted to the role. |

## Notes

- `user_id` is stored as text, so it works whether your users table keys on an integer
  or a uuid — pass either.
- `grant`/`assign` are check-then-insert, so they are safe to call repeatedly and need
  no vendor-specific upsert.
- Permissions are opaque strings; a dotted convention (`post.edit`, `user.delete`)
  reads well but nothing enforces it.
