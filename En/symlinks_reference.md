# `symlinks` Module Reference

> **Module purpose:** `symlinks` provides a thin, type-safe wrapper around Nim's lower-level symlink operations, exposing them through the strongly-typed `Path` abstraction from `std/paths`. It covers the three fundamental operations you need to work with symbolic links: checking existence, creating them, and resolving what they point to.

---

## Background: What Is a Symbolic Link?

A symbolic link (symlink) is a special filesystem entry that acts as a pointer to another file or directory. It stores a path string rather than actual data. The operating system transparently follows the link when you access it through normal file operations — but symlink-specific operations let you inspect or manipulate the link itself.

```
/home/user/config  →  /etc/myapp/config.json   (symlink → target)
```

Symlinks can point to files, directories, other symlinks, or even paths that don't exist yet (dangling links). This module handles all these cases.

---

## Type Used

### `Path`

All procedures in this module use `Path` from `std/paths` rather than raw `string`. `Path` is a distinct type wrapping a string, which prevents accidentally mixing up path values with arbitrary strings at compile time. You construct one with `Path("some/path")` and get the underlying string back with `.string`.

---

## Procedure Reference

---

### `symlinkExists`

```nim
proc symlinkExists*(link: Path): bool
  {.inline, tags: [ReadDirEffect], sideEffect.}
```

#### What it does

Checks whether a symbolic link exists at the given path `link` and returns `true` if it does. Crucially, this procedure checks for the **link itself** — not its target. This means:

- If the link exists but its target has been deleted (a *dangling* or *broken* link), `symlinkExists` still returns `true`.
- If the path exists as a regular file or directory (not a symlink), it returns `false`.
- If the path does not exist at all, it returns `false`.

This is the key distinction from `fileExists` or `dirExists`, which follow the link and check the target. `symlinkExists` stops at the link.

#### Tags

| Tag | Meaning |
|---|---|
| `ReadDirEffect` | The procedure reads directory metadata from the filesystem |
| `sideEffect` | The result may differ between calls (filesystem state can change) |

#### Examples

```nim
import std/symlinks, std/paths

# Create a symlink for demonstration purposes
createSymlink(Path("/etc/hosts"), Path("/tmp/my_hosts_link"))

# The link itself exists
assert symlinkExists(Path("/tmp/my_hosts_link")) == true

# A plain file is not a symlink
assert symlinkExists(Path("/etc/hosts")) == false

# Non-existent path
assert symlinkExists(Path("/tmp/no_such_link")) == false
```

Detecting a broken (dangling) link — something `fileExists` cannot do:

```nim
import std/symlinks, std/paths, std/os

createSymlink(Path("/nonexistent/target"), Path("/tmp/broken_link"))

# symlinkExists: sees the link → true
assert symlinkExists(Path("/tmp/broken_link")) == true

# fileExists: follows the link, target missing → false
assert not fileExists("/tmp/broken_link")
```

#### When to use it

Use `symlinkExists` when you specifically need to know whether a path **is a symlink**, regardless of whether the target is reachable. For example: cleaning up stale links, auditing a directory for symlinks, or deciding whether to call `expandSymlink`.

---

### `createSymlink`

```nim
proc createSymlink*(src, dest: Path) {.inline.}
```

#### What it does

Creates a new symbolic link at `dest` that points to `src`. After the call, accessing `dest` through normal file I/O will transparently redirect to `src`.

`src` does not have to exist at the time the link is created — the OS stores the path string and resolves it lazily when the link is accessed. This means you can create a link to a future or optional resource.

#### Failure conditions

The procedure will raise an `OSError` if:

- A file, directory, or symlink already exists at `dest`.
- The calling process lacks the necessary permissions (see the Windows note below).
- The parent directory of `dest` does not exist.

#### ⚠️ Windows restriction

On most Unix-like systems any user can create symlinks freely. On **Windows**, creating symlinks requires either:

- Running as Administrator, or
- Having **Developer Mode** enabled in system settings (available since Windows 10 version 1703).

If neither condition is met, the call will fail with an access-denied error. This is an OS-level policy, not a Nim limitation.

#### Parameter summary

| Parameter | Type | Description |
|---|---|---|
| `src` | `Path` | The target the symlink will point to (need not exist yet) |
| `dest` | `Path` | The path where the new symlink will be created |

#### Examples

```nim
import std/symlinks, std/paths

# Create a symlink: /tmp/link_to_hosts → /etc/hosts
createSymlink(Path("/etc/hosts"), Path("/tmp/link_to_hosts"))

assert symlinkExists(Path("/tmp/link_to_hosts"))

# Now reading /tmp/link_to_hosts is the same as reading /etc/hosts
```

Symlinking a directory:

```nim
import std/symlinks, std/paths

# Make a convenient alias for a deep path
createSymlink(Path("/var/log/myapp/current"), Path("/tmp/myapp_log"))

# Access files through the symlink as if it were the real directory
```

Creating a link to a not-yet-existing target (future resource):

```nim
import std/symlinks, std/paths

# This succeeds even if /mnt/data/store doesn't exist yet
createSymlink(Path("/mnt/data/store"), Path("/tmp/store"))
# Once /mnt/data/store is created/mounted, the link will work
```

Guarding against overwrite:

```nim
import std/symlinks, std/paths

let dest = Path("/tmp/my_link")
if not symlinkExists(dest):
  createSymlink(Path("/some/target"), dest)
else:
  echo "Link already exists, skipping"
```

---

### `expandSymlink`

```nim
proc expandSymlink*(symlinkPath: Path): Path {.inline.}
```

#### What it does

Reads the symbolic link at `symlinkPath` and returns the path it points to — that is, the value stored *inside* the link, not the resolved final file. This is sometimes called "reading" a symlink.

The returned path may be:

- **Absolute** (`/etc/hosts`) — if the link was created with an absolute target.
- **Relative** (`../config/settings.json`) — if the link was created with a relative target. In this case, the relative path is interpreted relative to the *directory containing the link*, not the current working directory.

The procedure does **not** check whether the target exists or follow chains of symlinks. It simply returns what the link stores.

#### Windows behaviour

On Windows, `expandSymlink` is effectively a **no-op**: the input `symlinkPath` is returned unchanged. This is a known limitation. Windows symlinks require special API calls that may not be uniformly supported; to avoid silent errors, the function simply returns the input path as-is. Cross-platform code that depends on expanding symlinks should handle this case explicitly.

#### Parameter summary

| Parameter | Type | Description |
|---|---|---|
| `symlinkPath` | `Path` | The path of the symlink to read |

**Returns:** `Path` — the target stored inside the symlink (or the input unchanged on Windows).

#### Examples

```nim
import std/symlinks, std/paths

# Create a known symlink
createSymlink(Path("/etc/hosts"), Path("/tmp/my_link"))

# Expand it
let target = expandSymlink(Path("/tmp/my_link"))
assert target == Path("/etc/hosts")
```

Relative symlinks:

```nim
import std/symlinks, std/paths

# If this link was created as: ln -s ../config/app.cfg /tmp/links/app
let target = expandSymlink(Path("/tmp/links/app"))
# target might be Path("../config/app.cfg")
# To get the real path, resolve relative to the link's directory:
# /tmp/links/ + ../config/app.cfg = /tmp/config/app.cfg
```

Checking and then expanding safely:

```nim
import std/symlinks, std/paths

let link = Path("/tmp/maybe_link")
if symlinkExists(link):
  let target = expandSymlink(link)
  echo "Link points to: ", target.string
else:
  echo "No symlink at that path"
```

Detecting the Windows no-op:

```nim
import std/symlinks, std/paths

let link = Path("/some/link")
let expanded = expandSymlink(link)
when defined(windows):
  assert expanded == link  # no-op on Windows
```

---

## How the Three Procedures Work Together

A typical symlink workflow uses all three procedures in concert:

```nim
import std/symlinks, std/paths, std/os

proc ensureSymlink(src, dest: Path) =
  if symlinkExists(dest):
    let current = expandSymlink(dest)
    if current == src:
      return  # already correct, nothing to do
    else:
      removeFile(dest.string)  # stale link, remove it

  createSymlink(src, dest)
  echo "Created symlink: ", dest.string, " → ", src.string
```

| Step | Procedure | Purpose |
|---|---|---|
| 1 | `symlinkExists` | Check whether a link is already in place |
| 2 | `expandSymlink` | Verify that the existing link points to the right target |
| 3 | `createSymlink` | Create the link if it's missing or wrong |

---

## Summary Table

| Procedure | Reads/Writes | Follows link? | Windows note |
|---|---|---|---|
| `symlinkExists` | Reads | ❌ No — checks link itself | Works normally |
| `createSymlink` | Writes | — | Requires admin or Developer Mode |
| `expandSymlink` | Reads | ❌ No — returns stored path | Returns input unchanged (no-op) |

---

## Common Pitfalls

**Confusing `symlinkExists` with `fileExists`/`dirExists`.** The latter two follow symlinks and check the target. Use `symlinkExists` when you care about the link itself, not what it points to.

**Forgetting that `expandSymlink` may return a relative path.** If the link was created with a relative target, you must join the result with the link's parent directory to get the actual absolute path.

**Windows Developer Mode.** If your code creates symlinks and needs to run on Windows without administrator privileges, make sure to document the Developer Mode requirement or provide a fallback (e.g., copying the file instead of linking it).

**Overwriting an existing link.** `createSymlink` does not overwrite — it raises `OSError` if `dest` already exists. Always check with `symlinkExists` first if you may be re-running the operation.
