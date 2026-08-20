# storage — file storage for LOOK

A thin, honest storage layer over the local disk: `put` / `get` / `delete` / `url`,
every key confined to a root directory with `..` rejected. It exposes the surface an
S3-compatible package can implement later, so app code that stores an avatar today keeps
working when you move the bucket to S3/R2/MinIO tomorrow.

## Install

```bash
lk module install github.com/codlook/look-modules/storage
```

```lk
use storage
```

## Config (env — explicit, nothing hidden)

| Variable | Default | Meaning |
|----------|---------|---------|
| `STORAGE_ROOT` | `storage` | Base directory on disk. **Must live inside `LOOK_FILE_ROOT`** (the runtime's file sandbox). |
| `STORAGE_URL` | `/storage` | Public URL prefix returned by `storage_url`. |

## Use

```lk
use storage

storage_put("avatars/42.png", $bytes)        # write (overwrites)
$data = storage_get("avatars/42.png")         # read
if (storage_exists("avatars/42.png")) { }
$n = storage_size("avatars/42.png")           # bytes
$url = storage_url("avatars/42.png")          # -> "/storage/avatars/42.png"
storage_delete("avatars/42.png")
```

## API

| Function | Returns | Description |
|----------|---------|-------------|
| `storage_put($key, $content)` | `bool` | Write, overwriting. |
| `storage_get($key)` | `string` | Read the object. |
| `storage_exists($key)` | `bool` | Does it exist? |
| `storage_delete($key)` | `bool` | Remove it. |
| `storage_size($key)` | `int` | Size in bytes. |
| `storage_append($key, $data)` | `bool` | Append to the object. |
| `storage_url($key)` | `string` | Public URL, `STORAGE_URL` + key. |

## Notes

- `$key` is a relative path like `avatars/42.png`; a leading `/` is trimmed and any key
  containing `..` is rejected.
- **No mkdir:** LOOK has no directory-creation builtin, so `STORAGE_ROOT` and any
  sub-directory a key names must already exist — create your storage layout at deploy
  time.
- **Two layers of confinement:** this module's `..` guard, and beneath it the core
  `file::` sandbox (`LOOK_FILE_ROOT`) which rejects any path outside the sandbox before
  storage ever writes.
