# files — Module Reference

## Overview

The `std/files` module provides **type-safe file operations** built on top of the `Path` type from `std/paths`. It is the recommended way to work with files in modern Nim code because using `Path` instead of raw strings prevents accidental mixing of path strings with ordinary strings at compile time.

Every procedure in this module is a thin, inlined wrapper around the corresponding procedure in `std/private/osfiles`. The actual logic lives there; `std/files` simply re-exposes it with `Path`-typed parameters.

**Two types are re-exported from the underlying layer** so callers do not need to import anything else:

- `FilePermission` — an enum describing individual POSIX-style permission bits.
- `CopyFlag` — an enum controlling symlink behaviour during copy operations.

### `FilePermission` values

| Value | Meaning |
|---|---|
| `fpUserRead` | Owner can read |
| `fpUserWrite` | Owner can write |
| `fpUserExec` | Owner can execute |
| `fpGroupRead` | Group can read |
| `fpGroupWrite` | Group can write |
| `fpGroupExec` | Group can execute |
| `fpOthersRead` | Others can read |
| `fpOthersWrite` | Others can write |
| `fpOthersExec` | Others can execute |

On Windows only the `readonly` concept exists. The only `FilePermission` that has effect there is `fpUserWrite`: its presence means the file is writable; its absence means the file is read-only. All other bits are accepted without error but have no practical effect.

### `CopyFlag` values

| Value | Meaning |
|---|---|
| `cfSymlinkFollow` | If source is a symlink, copy the file it points to (default) |
| `cfSymlinkAsIs` | Copy the symlink itself, not its target |

`CopyFlag` is ignored on Windows — symlinks are always skipped there.

---

## `fileExists`

```nim
proc fileExists*(filename: Path): bool
  {.inline, tags: [ReadDirEffect], sideEffect.}
```

### Description

Returns `true` if `filename` exists on the filesystem **and** is a regular file or a symlink pointing to one. Returns `false` for anything else: directories, device files, named pipes, sockets, or paths that do not exist at all.

The `sideEffect` pragma signals that this procedure performs an actual filesystem query on every call — it is not pure and its result can change between calls even with the same argument.

### Parameters

| Name | Type | Description |
|---|---|---|
| `filename` | `Path` | Path of the file to check |

### Return value

`true` if the path names an existing regular file or a symlink to one; `false` otherwise.

### Examples

**Basic existence check before reading:**

```nim
import std/files, std/paths

let cfg = Path("config.toml")
if fileExists(cfg):
  echo "Config found, loading…"
else:
  echo "No config file — using defaults."
```

**Distinguishing a file from a directory:**

```nim
import std/files, std/paths

let p = Path("/etc")
echo fileExists(p)   # false — /etc is a directory, not a file
```

**Checking before overwriting (to warn the user):**

```nim
import std/files, std/paths

let output = Path("report.pdf")
if fileExists(output):
  echo "Warning: ", output, " will be overwritten."
```

---

## `tryRemoveFile`

```nim
proc tryRemoveFile*(file: Path): bool {.inline, tags: [WriteDirEffect].}
```

### Description

Attempts to delete `file` from the filesystem. Returns `true` on success, or `false` if deletion failed for any reason — including lack of permissions, the file being locked, or the path pointing to a directory. Crucially, it also returns `true` if the file never existed in the first place; a missing file is not considered a failure.

On Windows the read-only attribute is ignored, so a read-only file can be deleted with this procedure.

Use `tryRemoveFile` when deletion is a best-effort cleanup step and failure is acceptable. Use `removeFile` when failure should be treated as an error.

### Parameters

| Name | Type | Description |
|---|---|---|
| `file` | `Path` | Path of the file to delete |

### Return value

`true` if the file was deleted or did not exist; `false` if deletion failed.

### Examples

**Silent cleanup — ignore failure:**

```nim
import std/files, std/paths

discard tryRemoveFile(Path("session.lock"))
# If the lock file existed, it's gone. If it didn't, no problem.
```

**Log a warning when cleanup fails:**

```nim
import std/files, std/paths

let tmp = Path("/tmp/myapp_scratch.bin")
if not tryRemoveFile(tmp):
  echo "Warning: could not remove temporary file ", tmp
```

**Idempotent cleanup in a test suite:**

```nim
import std/files, std/paths

proc setupTest() =
  discard tryRemoveFile(Path("test_output.txt"))  # start clean, don't care if absent
  # … create fresh test data …
```

---

## `removeFile`

```nim
proc removeFile*(file: Path) {.inline, tags: [WriteDirEffect].}
```

### Description

Deletes `file` from the filesystem. If deletion fails, `OSError` is raised with a system-level error message. Like `tryRemoveFile`, a non-existent file is **not** an error — the procedure returns normally in that case.

On Windows the read-only attribute is ignored.

This is the strict counterpart of `tryRemoveFile`. Prefer it when a deletion failure indicates a genuine problem that the caller cannot or should not silently ignore.

### Parameters

| Name | Type | Description |
|---|---|---|
| `file` | `Path` | Path of the file to delete |

### Raises

`OSError` if the file exists but cannot be deleted.

### Examples

**Mandatory cleanup after processing:**

```nim
import std/files, std/paths

let tmpPath = Path("intermediate.bin")
processData(tmpPath)
removeFile(tmpPath)   # must succeed; failure means something is seriously wrong
```

**Removing a file as part of an atomic replace pattern:**

```nim
import std/files, std/paths

# Write to a temp file first, then replace the target atomically
let target = Path("data.db")
let tmp    = Path("data.db.tmp")
writeDatabase(tmp)
removeFile(target)   # remove old; raise if we can't
moveFile(tmp, target)
```

**Handling `OSError` explicitly:**

```nim
import std/files, std/paths

try:
  removeFile(Path("locked_file.txt"))
except OSError as e:
  echo "Could not delete file: ", e.msg
```

---

## `moveFile`

```nim
proc moveFile*(source, dest: Path)
  {.inline, tags: [ReadDirEffect, ReadIOEffect, WriteIOEffect].}
```

### Description

Moves (or renames) the file at `source` to `dest`. If `dest` already exists, it is **overwritten**. If the move fails, `OSError` is raised.

Symlinks are handled at the link level — if `source` is a symlink, the symlink itself is moved, not the file it points to. This makes `moveFile` safe to use for relocating symlinks without accidentally following them.

On the same filesystem, the operation is typically implemented as an atomic rename, making it an efficient and reliable way to replace a file with an updated version. Across filesystem boundaries the implementation falls back to a copy-then-delete sequence.

### Parameters

| Name | Type | Description |
|---|---|---|
| `source` | `Path` | Existing file or symlink to move |
| `dest` | `Path` | Destination path (parent directory must exist) |

### Raises

`OSError` if the move fails.

### Examples

**Renaming a file:**

```nim
import std/files, std/paths

moveFile(Path("draft.md"), Path("final.md"))
```

**Atomic replace — write to temp, then move into place:**

```nim
import std/files, std/paths

let live = Path("config.json")
let tmp  = Path("config.json.new")
writeConfig(tmp)
moveFile(tmp, live)   # overwrites live atomically
```

**Moving a symlink without following it:**

```nim
import std/files, std/paths

# If "link" is a symlink → "target.txt", after this call
# "newlink" → "target.txt" and "link" no longer exists.
moveFile(Path("link"), Path("newlink"))
```

---

## `copyFile`

```nim
proc copyFile*(source, dest: Path;
               options = cfSymlinkFollow;
               bufferSize = 16_384)
  {.inline, tags: [ReadDirEffect, ReadIOEffect, WriteIOEffect].}
```

### Description

Copies the content of `source` to `dest`. The parent directory of `dest` must already exist; the procedure does not create intermediate directories.

**Permission handling is platform-dependent:**

- **Windows:** file attributes are automatically copied alongside the content. No extra steps are needed.
- **Non-Windows:** only the file content is copied. The destination file inherits the default permissions for a newly created file (typically controlled by the process `umask`). To preserve permissions, use `copyFileWithPermissions` instead, or call `getFilePermissions` / `setFilePermissions` manually.

If `dest` already exists, its attributes (owner, permissions) are preserved and only its content is overwritten.

The `options` parameter controls symlink handling on non-Windows systems. The default `cfSymlinkFollow` means: if `source` is a symlink, copy the file it points to. Use `cfSymlinkAsIs` to copy the symlink itself.

`bufferSize` (default 16 KiB) controls the read/write buffer used during the copy. Larger values can improve throughput on fast storage at the cost of more memory; smaller values reduce memory use at the cost of more system calls.

### Parameters

| Name | Type | Description |
|---|---|---|
| `source` | `Path` | File to copy |
| `dest` | `Path` | Destination path (parent dir must exist) |
| `options` | `CopyFlag` | Symlink behaviour; default `cfSymlinkFollow` |
| `bufferSize` | `int` | I/O buffer size in bytes; default `16_384` |

### Raises

`OSError` if the copy fails.

### Examples

**Simple file copy:**

```nim
import std/files, std/paths

copyFile(Path("template.html"), Path("output/index.html"))
```

**Copy with a larger buffer for big files:**

```nim
import std/files, std/paths

copyFile(Path("video.mp4"), Path("backup/video.mp4"),
         bufferSize = 1_048_576)   # 1 MiB buffer
```

**Copying a symlink as a symlink (not following it):**

```nim
import std/files, std/paths

copyFile(Path("link_to_data"), Path("backup/link_to_data"),
         options = cfSymlinkAsIs)
```

**Preserving permissions on non-Windows (manual approach):**

```nim
import std/files, std/paths

let src  = Path("script.sh")
let dest = Path("bin/script.sh")
copyFile(src, dest)
setFilePermissions(dest, getFilePermissions(src))
```

---

## `copyFileWithPermissions`

```nim
proc copyFileWithPermissions*(source, dest: Path;
                              ignorePermissionErrors = true;
                              options = cfSymlinkFollow) {.inline.}
```

### Description

Copies `source` to `dest` and then also copies the file's **permission bits**, combining `copyFile`, `getFilePermissions`, and `setFilePermissions` into a single convenient call.

On Windows this is equivalent to `copyFile` because Windows already copies attributes automatically. On non-Windows systems the permission copy happens after the content copy and is **not atomic**: there is a brief window between the two steps during which the destination file exists with the wrong permissions. This is usually acceptable but may matter in security-sensitive contexts.

If reading or setting permissions fails, the behaviour depends on `ignorePermissionErrors`:

- `true` (default) — the error is silently swallowed; the file content is still copied.
- `false` — `OSError` is raised.

### Parameters

| Name | Type | Description |
|---|---|---|
| `source` | `Path` | File to copy |
| `dest` | `Path` | Destination path (parent dir must exist) |
| `ignorePermissionErrors` | `bool` | If `true`, silently ignore permission copy errors; default `true` |
| `options` | `CopyFlag` | Symlink behaviour; default `cfSymlinkFollow` |

### Raises

`OSError` if the file copy fails, or if `ignorePermissionErrors` is `false` and a permission operation fails.

### Examples

**Copy preserving permissions (tolerant of permission errors):**

```nim
import std/files, std/paths

copyFileWithPermissions(Path("setup.sh"), Path("dist/setup.sh"))
# dest will have the same permission bits as source
```

**Strict mode — fail if permissions cannot be copied:**

```nim
import std/files, std/paths

copyFileWithPermissions(Path("private_key.pem"),
                        Path("backup/private_key.pem"),
                        ignorePermissionErrors = false)
# Raises OSError if chmod fails — important for security-sensitive files
```

---

## `copyFileToDir`

```nim
proc copyFileToDir*(source, dir: Path;
                    options = cfSymlinkFollow;
                    bufferSize = 16_384) {.inline.}
```

### Description

Copies the file at `source` into the directory `dir`, keeping the original filename. The destination directory must already exist. This is a convenience wrapper around `copyFile` that computes `dest = dir / source.lastPathPart` for you, saving the boilerplate of extracting the filename and constructing the destination path.

The `options` and `bufferSize` parameters have the same meaning as in `copyFile`.

### Parameters

| Name | Type | Description |
|---|---|---|
| `source` | `Path` | File to copy |
| `dir` | `Path` | Destination directory (must exist) |
| `options` | `CopyFlag` | Symlink behaviour; default `cfSymlinkFollow` |
| `bufferSize` | `int` | I/O buffer size in bytes; default `16_384` |

### Raises

`OSError` if the copy fails or if `dir` does not exist.

### Examples

**Copy a single file into a directory:**

```nim
import std/files, std/paths

copyFileToDir(Path("README.md"), Path("dist/"))
# Result: dist/README.md
```

**Batch copy — copy several files into the same directory:**

```nim
import std/files, std/paths

let assets = [Path("icon.png"), Path("style.css"), Path("app.js")]
for f in assets:
  copyFileToDir(f, Path("public/static/"))
```

---

## `getFilePermissions`

```nim
proc getFilePermissions*(filename: Path): set[FilePermission]
  {.inline, tags: [ReadDirEffect].}
```

### Description

Returns the permission bits of `filename` as a Nim set of `FilePermission` values. Each element present in the set means the corresponding permission is granted.

On Windows only the read-only status is meaningful. If the file is writable, `fpUserWrite` is included in the result; if it is read-only, `fpUserWrite` is absent. All other permission values are always present on Windows (since the concept does not exist there).

Raises `OSError` on failure (e.g. file does not exist, insufficient privileges to query).

### Parameters

| Name | Type | Description |
|---|---|---|
| `filename` | `Path` | File whose permissions are queried |

### Return value

`set[FilePermission]` — the set of currently granted permissions.

### Examples

**Reading and printing permissions:**

```nim
import std/files, std/paths

let perms = getFilePermissions(Path("data.csv"))
if fpUserWrite in perms:
  echo "File is writable by owner."
if fpOthersRead in perms:
  echo "File is world-readable."
```

**Saving permissions before modifying them:**

```nim
import std/files, std/paths

let f = Path("script.sh")
let original = getFilePermissions(f)

# Temporarily make it non-executable
setFilePermissions(f, original - {fpUserExec, fpGroupExec, fpOthersExec})
doWork(f)
setFilePermissions(f, original)   # restore
```

---

## `setFilePermissions`

```nim
proc setFilePermissions*(filename: Path;
                         permissions: set[FilePermission];
                         followSymlinks = true)
  {.inline, tags: [ReadDirEffect, WriteDirEffect].}
```

### Description

Applies the permission set `permissions` to `filename`, replacing any existing permissions entirely.

When `followSymlinks` is `true` (the default), and `filename` is a symlink, the permissions are applied to the **target file**, not the symlink itself. When `false`, the permissions are applied to the symlink — but this is a no-op on Windows and on many POSIX systems (including Linux) where `lchmod` is unavailable or always fails, because symlink permissions are not observed on those systems.

On Windows only `fpUserWrite` matters: its absence makes the file read-only; its presence makes it writable.

Raises `OSError` on failure.

### Parameters

| Name | Type | Description |
|---|---|---|
| `filename` | `Path` | File whose permissions are to be changed |
| `permissions` | `set[FilePermission]` | The complete set of permissions to apply |
| `followSymlinks` | `bool` | Follow symlinks (default `true`); `false` is a no-op on most POSIX systems |

### Raises

`OSError` if the operation fails.

### Examples

**Making a script executable:**

```nim
import std/files, std/paths

setFilePermissions(Path("deploy.sh"),
  {fpUserRead, fpUserWrite, fpUserExec,
   fpGroupRead, fpGroupExec,
   fpOthersRead, fpOthersExec})
# chmod 755 equivalent
```

**Making a file read-only for everyone:**

```nim
import std/files, std/paths

setFilePermissions(Path("archive.tar.gz"),
  {fpUserRead, fpGroupRead, fpOthersRead})
# chmod 444 equivalent
```

**Restricting a private key to owner-read-only:**

```nim
import std/files, std/paths

setFilePermissions(Path("id_rsa"), {fpUserRead})
# chmod 400 equivalent — typical requirement for SSH private keys
```

---

## Summary table

| Procedure | Effect tags | Raises | Description |
|---|---|---|---|
| `fileExists(filename)` | `ReadDirEffect` | — | Check if a path is an existing regular file |
| `tryRemoveFile(file)` | `WriteDirEffect` | — | Delete a file; return `false` on failure |
| `removeFile(file)` | `WriteDirEffect` | `OSError` | Delete a file; raise on failure |
| `moveFile(source, dest)` | `ReadDirEffect`, `ReadIOEffect`, `WriteIOEffect` | `OSError` | Move or rename a file |
| `copyFile(source, dest, …)` | `ReadDirEffect`, `ReadIOEffect`, `WriteIOEffect` | `OSError` | Copy file content |
| `copyFileWithPermissions(source, dest, …)` | same as `copyFile` | `OSError` | Copy file content and permission bits |
| `copyFileToDir(source, dir, …)` | same as `copyFile` | `OSError` | Copy file into a directory, keeping filename |
| `getFilePermissions(filename)` | `ReadDirEffect` | `OSError` | Read permission bits of a file |
| `setFilePermissions(filename, perms, …)` | `ReadDirEffect`, `WriteDirEffect` | `OSError` | Write permission bits to a file |
