# `dirs` Module Reference

> **Module:** `std/dirs`  
> **Language:** Nim  
> **Purpose:** Cross-platform directory operations — creating, removing, moving, copying, and traversing directories.

All paths in this module use the typed `Path` wrapper (from `std/paths`) instead of raw strings. This makes the API safer and more expressive because the type system prevents accidentally mixing file paths with arbitrary strings.

---

## Exported Type: `PathComponent`

Before diving into functions, it is important to understand `PathComponent` — an enum that classifies what kind of filesystem entry an iterator has encountered. It is re-exported from the standard library so you can use it without any additional import.

| Value | Meaning |
|---|---|
| `pcFile` | A regular file |
| `pcLinkToFile` | A symbolic link pointing to a file |
| `pcDir` | A regular directory |
| `pcLinkToDir` | A symbolic link pointing to a directory |

---

## Procedures

### `dirExists`

```nim
proc dirExists*(dir: Path): bool
```

**What it does:** Checks whether a directory exists at the given path. Returns `true` if a directory is found, `false` in every other case — including when the path points to a file rather than a directory. Symlinks are transparently followed, so if a symlink points to an existing directory, the result is `true`.

This is a read-only, non-destructive check. It is safe to call at any time.

**When to use it:** Before attempting to open, list, or create something inside a directory; as a guard condition in scripts that must not proceed if a required directory is missing.

```nim
import std/dirs
import std/paths

let config = Path("/etc/myapp")

if dirExists(config):
  echo "Config directory found, proceeding."
else:
  echo "Config directory missing!"
```

---

### `createDir`

```nim
proc createDir*(dir: Path)
```

**What it does:** Creates the directory specified by `dir`, including all intermediate parent directories that do not yet exist. Think of it as the equivalent of `mkdir -p` on Unix systems.

If the directory already exists, the procedure does nothing and does **not** raise an error. This "idempotent" behaviour is intentional — in most programs, a directory that already exists is not a problem.

If creation fails for any other reason (insufficient permissions, invalid path, etc.), `OSError` is raised.

**When to use it:** Whenever you need to ensure a directory exists before writing files into it. Because it handles nested paths automatically, you can pass a deeply nested target directory without worrying about creating each level manually.

```nim
import std/dirs
import std/paths

# Creates "output/reports/2024" even if none of the levels exist yet.
createDir(Path("output/reports/2024"))

# Calling it again is perfectly safe — no error is raised.
createDir(Path("output/reports/2024"))
```

---

### `existsOrCreateDir`

```nim
proc existsOrCreateDir*(dir: Path): bool
```

**What it does:** A combination of existence check and directory creation in a single call. It checks whether `dir` already exists:

- If it **does exist**, the procedure returns `true` and does nothing else.
- If it **does not exist**, the procedure creates it and returns `false`.

**Important limitation:** Unlike `createDir`, this procedure does **not** create missing parent directories. If the parent directory does not exist, `OSError` is raised. Use this when you know the parent already exists and only need to ensure the final directory level is present.

**When to use it:** When you need to know *whether you just created the directory* — for example, to decide whether to populate it with initial data.

```nim
import std/dirs
import std/paths

let cache = Path("/tmp/myapp/cache")

let alreadyExisted = existsOrCreateDir(cache)
if alreadyExisted:
  echo "Cache directory was already there, reusing existing data."
else:
  echo "Cache directory was just created, starting fresh."
```

---

### `removeDir`

```nim
proc removeDir*(dir: Path, checkDir = false)
```

**What it does:** Deletes the directory `dir` along with **all of its contents** — subdirectories, files, everything — recursively. This is the equivalent of `rm -rf` on Unix.

By default (`checkDir = false`), removing a directory that does not exist is silently ignored. This makes cleanup code straightforward: you can unconditionally remove a directory without first checking if it exists.

If you set `checkDir = true`, an `OSError` will be raised when the directory does not exist. Use this when the absence of the directory indicates a logic error and should not be silently swallowed.

⚠️ **There is no undo.** Deleted files and directories cannot be recovered.

**When to use it:** Cleaning up temporary build artifacts, test fixtures, cache directories, or any tree of files that should be wiped in one operation.

```nim
import std/dirs
import std/paths

let tmp = Path("build/tmp")

# Safe even if "build/tmp" does not exist.
removeDir(tmp)

# Raises OSError if the directory is not there.
removeDir(tmp, checkDir = true)
```

---

### `moveDir`

```nim
proc moveDir*(source, dest: Path)
```

**What it does:** Moves (renames) a directory from `source` to `dest`. The entire directory tree is relocated. If the underlying filesystem supports it, this is a fast atomic rename; otherwise, the contents may be copied and then the source deleted.

Symlinks inside the directory are moved as-is — the symlinks themselves are relocated, not the files or directories they point to.

If the operation fails for any reason, `OSError` is raised.

**When to use it:** Renaming a directory, archiving an old version of a folder, or atomically replacing one directory with another.

```nim
import std/dirs
import std/paths

# Rename the directory (or move it to a different location).
moveDir(Path("output/draft"), Path("output/final"))
```

---

### `walkDir` *(iterator)*

```nim
iterator walkDir*(dir: Path; relative = false, checkDir = false,
                  skipSpecial = false): tuple[kind: PathComponent, path: Path]
```

**What it does:** Iterates over the **immediate** contents of directory `dir` — one level deep only, not recursive. For each entry it yields a tuple containing:

- `kind` — a `PathComponent` value telling you what type of entry it is (`pcFile`, `pcDir`, `pcLinkToFile`, `pcLinkToDir`).
- `path` — the path to that entry.

**Parameters:**

- `relative` — when `true`, paths yielded are relative to `dir` (just the entry name). When `false` (default), the full path is returned.
- `checkDir` — when `true`, raises `OSError` if `dir` does not exist. When `false` (default), an empty iteration occurs silently.
- `skipSpecial` — on Unix, when `true`, special filesystem objects (FIFOs, device files, sockets) are excluded; only regular files and directories are yielded.

**When to use it:** Listing the direct children of a directory, building a file browser, or processing only the top level of a folder hierarchy.

```nim
import std/dirs
import std/paths

for (kind, path) in walkDir(Path("/home/user/documents")):
  case kind
  of pcFile:    echo "  FILE: ", path
  of pcDir:     echo "   DIR: ", path
  of pcLinkToFile, pcLinkToDir:
                echo "  LINK: ", path
```

Using `relative = true`:

```nim
for (kind, path) in walkDir(Path("/home/user/documents"), relative = true):
  echo path   # yields just "notes.txt", "projects", etc.
```

---

### `walkDirRec` *(iterator)*

```nim
iterator walkDirRec*(dir: Path,
                     yieldFilter = {pcFile},
                     followFilter = {pcDir},
                     relative = false, checkDir = false,
                     skipSpecial = false): Path
```

**What it does:** Recursively walks the entire directory tree rooted at `dir`, yielding a `Path` for each matching entry. Unlike `walkDir`, this descends into subdirectories automatically.

Two filters control the behaviour:

- **`yieldFilter`** — a set of `PathComponent` values specifying *which entries to yield*. The default `{pcFile}` yields only regular files. Add `pcDir` to also yield directories, or `pcLinkToFile` / `pcLinkToDir` to include symlinks.
- **`followFilter`** — a set of `PathComponent` values specifying *which directories to descend into*. The default `{pcDir}` follows only real directories. Add `pcLinkToDir` to also follow symlinks to directories (be careful of cycles!).

The `relative`, `checkDir`, and `skipSpecial` parameters behave the same as in `walkDir`.

⚠️ **Do not modify the directory structure while the iterator is running.** Adding or removing files/directories during iteration results in undefined behaviour.

**When to use it:** Recursively processing all files in a project tree, building an index of all files matching certain criteria, or implementing a recursive delete/copy manually.

```nim
import std/dirs
import std/paths

# Print every regular file under "src/".
for file in walkDirRec(Path("src")):
  echo file

# Also yield directories encountered along the way.
for entry in walkDirRec(Path("src"), yieldFilter = {pcFile, pcDir}):
  echo entry

# Follow symlinks to directories as well.
for entry in walkDirRec(Path("src"),
                         followFilter = {pcDir, pcLinkToDir}):
  echo entry
```

---

### `setCurrentDir`

```nim
proc setCurrentDir*(newDir: Path)
```

**What it does:** Changes the current working directory of the running process to `newDir`. All subsequent relative path operations (file opens, directory listings, etc.) will be resolved relative to this new directory.

If `newDir` does not exist or cannot be set for any reason, `OSError` is raised.

**When to use it:** When a program needs to switch its working context — for example, to run operations relative to a project root, or to change into a directory before invoking a subprocess that expects to find files there.

```nim
import std/dirs
import std/paths

setCurrentDir(Path("/var/myapp/data"))
# From this point on, relative paths resolve from /var/myapp/data.
```

> **Tip:** To read the current working directory, use `getCurrentDir` from `std/paths`.

---

### `copyDir`

```nim
proc copyDir*(source, dest: Path; skipSpecial = false)
```

**What it does:** Copies the entire directory tree from `source` to `dest`, including all files and subdirectories. The behaviour regarding symlinks and permissions differs by platform:

- **Non-Windows:** Symlinks are copied as symlinks (the link itself is recreated, not the target file). New files and directories receive the **default permissions** for newly created files on that system (i.e., subject to the user's `umask`). If you need to preserve the original permissions, use `copyDirWithPermissions` instead.
- **Windows:** Symlinks are skipped. File attributes (including permissions) are copied automatically.

If `skipSpecial` is `true`, special Unix filesystem objects (FIFOs, device files, sockets) are excluded from the copy.

If the operation fails, `OSError` is raised.

**When to use it:** Creating a backup copy of a directory, duplicating a template project, or staging files before deployment when preserving original permissions is not critical.

```nim
import std/dirs
import std/paths

# Copy the template directory into a new project directory.
copyDir(Path("templates/web"), Path("projects/my-new-site"))
```

---

### `copyDirWithPermissions`

```nim
proc copyDirWithPermissions*(source, dest: Path;
                              ignorePermissionErrors = true,
                              skipSpecial = false)
```

**What it does:** Like `copyDir`, but additionally preserves the Unix file permissions (read/write/execute bits) of all copied files and directories. On Windows, this is equivalent to `copyDir` because Windows already copies attributes.

On non-Windows systems, permissions are applied *after* the file or directory has been copied. This means the operation is not atomic: there is a brief window where the copied file exists but has incorrect permissions. In high-security or concurrent scenarios, be aware of this race condition.

**Parameters:**

- `ignorePermissionErrors` — when `true` (default), errors that occur while reading or setting permissions are silently ignored. Set to `false` if you want an `OSError` raised whenever a permission cannot be preserved — useful when correctness of permissions is critical.
- `skipSpecial` — same as in `copyDir`.

**When to use it:** Deploying configuration files or scripts that must retain their executable bit; archiving directories where the exact permission structure must be reproduced.

```nim
import std/dirs
import std/paths

# Copy and strictly preserve permissions; raise on any permission error.
copyDirWithPermissions(
  source = Path("/etc/myapp"),
  dest   = Path("/backup/myapp-config"),
  ignorePermissionErrors = false
)
```

---

## Quick Comparison: `createDir` vs `existsOrCreateDir`

| | `createDir` | `existsOrCreateDir` |
|---|---|---|
| Creates parent directories? | ✅ Yes | ❌ No |
| Returns whether it already existed? | ❌ No | ✅ Yes |
| Error if already exists? | ❌ No | ❌ No |
| Best for | Ensuring a full path exists | Checking + creating a single level |

## Quick Comparison: `copyDir` vs `copyDirWithPermissions`

| | `copyDir` | `copyDirWithPermissions` |
|---|---|---|
| Copies files and subdirectories | ✅ | ✅ |
| Preserves Unix permissions | ❌ (uses defaults) | ✅ |
| Handles Windows | ✅ (copies attrs) | ✅ (same as `copyDir` on Windows) |
| Extra parameter | `skipSpecial` | `skipSpecial`, `ignorePermissionErrors` |
