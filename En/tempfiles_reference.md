# `tempfiles` Module Reference

> **Module purpose:** `tempfiles` provides safe, cross-platform creation of temporary files and directories. It handles the classic race-condition-free creation pattern internally — generating a random unique name, atomically opening or creating it, and retrying automatically if a collision occurs. The caller receives a ready-to-use handle and path, without worrying about name uniqueness.
>
> ⚠️ **Experimental API.** The interface may change in future Nim versions.

---

## Background: Why Temporary Files Are Tricky

Naively creating a temporary file by first generating a name and then opening it is vulnerable to a **time-of-check/time-of-use (TOCTOU) race condition**: between the moment you decide the name is free and the moment you actually create the file, another process could grab the same name. `tempfiles` avoids this by using atomic OS primitives (`O_EXCL` on POSIX, `CREATE_NEW` on Windows) that fail immediately if the path already exists, then automatically retrying with a new random name.

---

## Internal Constants

These constants are not exported but shape the module's behaviour:

| Constant | Value | Purpose |
|---|---|---|
| `maxRetry` | `10000` | Maximum number of name-generation attempts before raising `OSError` |
| `letters` | `a-z A-Z 0-9` (62 chars) | Character pool for random name segments |
| `nimTempPathLength` | `8` (default) | Length of the random segment; overridable with `-d:nimTempPathLength=N` |

The random segment draws from 62 characters over 8 positions, giving 62⁸ ≈ 218 trillion possible names — collisions in practice are essentially impossible.

### Customising the random segment length

At compile time you can change the random segment length by passing a define:

```sh
nim c -d:nimTempPathLength=16 myprogram.nim
```

This makes every generated name contain a 16-character random segment instead of the default 8.

---

## Procedure Reference

---

### `genTempPath`

```nim
proc genTempPath*(prefix, suffix: string, dir = ""): string
```

#### What it does

Generates and returns a **candidate** path string for a temporary file or directory. The path is constructed as:

```
dir / (prefix & <random_segment> & suffix)
```

where `<random_segment>` is a string of `nimTempPathLength` randomly chosen alphanumeric characters.

This procedure **only produces a string** — it does not touch the filesystem, does not check whether the path exists, and does not create anything. It is a building block used internally by `createTempFile` and `createTempDir`, but is also exported so you can integrate it into custom creation logic.

#### The `dir` parameter

If `dir` is an empty string (the default), the system's temporary directory is used — the same path returned by `getTempDir()` from `std/appdirs` (e.g. `/tmp` on Linux/macOS, `%TEMP%` on Windows). The directory **must already exist**; `genTempPath` does not create it.

#### Parameter summary

| Parameter | Type | Description |
|---|---|---|
| `prefix` | `string` | Text placed before the random segment. Can be empty. |
| `suffix` | `string` | Text placed after the random segment. Can be empty. |
| `dir` | `string` | Directory for the path. Empty = system temp dir. |

**Returns:** `string` — a candidate path, not guaranteed to be unused.

#### Examples

```nim
import tempfiles

# Basic usage — generates something like /tmp/tmp_aBcDeFgH_.dat
let candidate = genTempPath("tmp_", "_.dat")
echo candidate

# Custom directory
let inLocal = genTempPath("session_", ".lock", dir = "/run/myapp")
# → /run/myapp/session_XyZ12345.lock

# No prefix or suffix — just a random name in the system temp dir
let bare = genTempPath("", "")
echo bare  # e.g. /tmp/kQmVwPnR

# Use it in your own creation loop if you need special open flags
import std/os
var path: string
var f: File
for attempt in 0 ..< 100:
  path = genTempPath("work_", ".bin")
  # try to open exclusively ...
  if open(f, path, fmReadWrite):
    break
```

#### When to use `genTempPath` directly

Prefer `createTempFile` or `createTempDir` for most use cases — they handle retries and atomic creation for you. Use `genTempPath` when you need to construct a temporary *path* without immediately creating the file, for example to pass as a future output path to an external tool.

---

### `createTempFile`

```nim
proc createTempFile*(prefix, suffix: string, dir = ""):
    tuple[cfile: File, path: string]
```

#### What it does

Creates a new, uniquely-named temporary file and returns both an open file handle (`cfile`) and the file's path (`path`) as a tuple. The file is opened in read-write mode (`w+`) and is **exclusively created** — it is guaranteed not to have existed before this call returned. If name collisions occur, the procedure retries automatically up to `maxRetry` (10 000) times.

The file is ready for immediate reading and writing through the returned `File` handle.

#### Caller responsibilities

Two things are explicitly left to the caller:

1. **Close the file handle** — call `close(result.cfile)` when done writing/reading.
2. **Delete the file** — call `removeFile(result.path)` when the file is no longer needed.

The module does not register any finaliser or atexit hook. If your program crashes or you forget to clean up, the file will remain on disk.

#### Failure conditions

`OSError` is raised if:
- The directory `dir` does not exist.
- After `maxRetry` attempts, no free name could be found (extraordinarily unlikely in practice).
- The process lacks write permission in `dir`.

#### Parameter summary

| Parameter | Type | Description |
|---|---|---|
| `prefix` | `string` | Text before the random segment in the filename. Can be empty. |
| `suffix` | `string` | Text after the random segment (e.g. a file extension). Can be empty. |
| `dir` | `string` | Directory to create the file in. Empty = system temp dir. |

**Returns:** `tuple[cfile: File, path: string]`
- `cfile` — open `File` handle in read-write mode, positioned at the start
- `path` — absolute path to the created file

#### Examples

```nim
import tempfiles, std/os

# Basic usage
let (cfile, path) = createTempFile("tmpprefix_", "_end.tmp")
# path is something like: /tmp/tmpprefix_FDCIRZA0_end.tmp

cfile.write("hello, world")
cfile.setFilePos(0)
echo readAll(cfile)   # → hello, world

close(cfile)
echo readFile(path)   # still accessible by path after close

removeFile(path)      # clean up
```

Using a specific directory:

```nim
import tempfiles, std/os

# Store the temp file alongside the project's build output
let (f, p) = createTempFile("build_", ".o", dir = "build/obj")
defer:
  close(f)
  removeFile(p)

# write intermediate compilation output to f ...
```

Robust cleanup with `defer`:

```nim
import tempfiles, std/os

proc processData(input: string) =
  let (tmp, tmpPath) = createTempFile("proc_", ".tmp")
  defer:
    close(tmp)
    removeFile(tmpPath)

  tmp.write(input)
  tmp.setFilePos(0)
  # pass tmpPath to an external tool or further processing ...
```

Non-existent directory raises immediately:

```nim
import tempfiles
doAssertRaises(OSError):
  discard createTempFile("", "", "nonexistent_dir")
```

#### Generated path anatomy

```
/tmp / tmpprefix_ FDCIRZA0 _end.tmp
 ↑dir    ↑prefix  ↑random   ↑suffix
```

The random segment is 8 alphanumeric characters by default (overridable at compile time).

---

### `createTempDir`

```nim
proc createTempDir*(prefix, suffix: string, dir = ""): string
```

#### What it does

Creates a new, uniquely-named temporary **directory** and returns its path. Like `createTempFile`, it generates a random name using `genTempPath` and uses an atomic OS call (`existsOrCreateDir`) to ensure the directory did not exist before. It retries automatically on collision.

Unlike `createTempFile`, there is no file handle to return — you get back a plain `string` path to the freshly created, empty directory.

#### Caller responsibilities

The caller is responsible for **removing the directory** (and its contents, if any were added) when it is no longer needed. Use `removeDir` from `std/os`.

#### Failure conditions

`OSError` is raised if:
- The parent directory `dir` does not exist.
- After `maxRetry` attempts, no free name could be found.
- The process lacks write permission in `dir`.

#### Parameter summary

| Parameter | Type | Description |
|---|---|---|
| `prefix` | `string` | Text before the random segment in the directory name. Can be empty. |
| `suffix` | `string` | Text after the random segment. Can be empty. |
| `dir` | `string` | Parent directory. Empty = system temp dir. |

**Returns:** `string` — the absolute path to the newly created temporary directory.

#### Examples

```nim
import tempfiles, std/os

let tmpDir = createTempDir("tmpprefix_", "_end")
# tmpDir is something like: /tmp/tmpprefix_YEl9VuVj_end

assert dirExists(tmpDir)

# Use the directory ...
writeFile(tmpDir / "output.txt", "results")

removeDir(tmpDir)   # removes the dir and all its contents
```

Using `defer` for guaranteed cleanup:

```nim
import tempfiles, std/os

proc runSandboxed(cmd: string) =
  let sandbox = createTempDir("sandbox_", "")
  defer: removeDir(sandbox)

  # run cmd with sandbox as working directory
  # all output files are isolated; cleaned up automatically
```

Using a specific parent directory:

```nim
import tempfiles, std/os

# Temp dir next to the project's cache
let workDir = createTempDir("job_", "_work", dir = "cache")
defer: removeDir(workDir)
```

Non-existent parent raises immediately:

```nim
import tempfiles
doAssertRaises(OSError):
  discard createTempDir("", "", "nonexistent")
```

---

## Internal Mechanics

### Random name generation

A per-thread `Rand` state (`nimTempPathState`) is lazily initialised the first time a name is generated on a given thread. This means:

- **Thread-safe:** each thread has its own independent random state, so concurrent calls from multiple threads do not interfere with each other.
- **No global lock:** name generation is lock-free.
- **Lazy seeding:** the RNG is seeded from the OS entropy source on first use, not at program startup.

### Atomic creation (`safeOpen`)

The private `safeOpen` procedure wraps the platform-specific atomic-open call:

| Platform | Mechanism | Guarantee |
|---|---|---|
| POSIX (Linux, macOS, …) | `open(O_RDWR | O_CREAT | O_EXCL)` | Fails atomically if file exists |
| Windows | `CreateFileW(CREATE_NEW)` | Fails atomically if file exists |

If the file already exists, `safeOpen` returns `nil` (no exception), and the caller retries with a fresh random name. Any other OS error is propagated as `OSError`.

---

## Summary Table

| Procedure | Creates | Returns | Cleanup needed |
|---|---|---|---|
| `genTempPath` | Nothing | Candidate path `string` | None |
| `createTempFile` | A file | `(cfile: File, path: string)` | `close(cfile)` + `removeFile(path)` |
| `createTempDir` | A directory | Path `string` | `removeDir(path)` |

---

## Common Patterns

### Pattern 1 — Temporary file as a safe intermediate output

```nim
import tempfiles, std/os

proc atomicWrite(dest, content: string) =
  # Write to a temp file first, then rename → atomic from the reader's perspective
  let (tmp, tmpPath) = createTempFile("atomic_", ".tmp",
                                       dir = parentDir(dest))
  defer: close(tmp)
  tmp.write(content)
  moveFile(tmpPath, dest)   # atomic on most filesystems if same partition
```

### Pattern 2 — Isolated working directory for a subprocess

```nim
import tempfiles, std/os, std/osproc

proc runIsolated(exe: string, args: seq[string]): int =
  let workDir = createTempDir("run_", "")
  defer: removeDir(workDir)
  let p = startProcess(exe, workingDir = workDir, args = args)
  result = waitForExit(p)
```

### Pattern 3 — Custom temp path for passing to external tools

```nim
import tempfiles, std/os, std/osproc

# Some tools want to decide whether to create the file themselves.
# Give them a guaranteed-unique path without creating the file.
let path = genTempPath("ffmpeg_out_", ".mp4")
discard execCmd("ffmpeg -i input.mkv " & path)
defer: removeFile(path)
```

---

## Notes

- All three procedures accept an empty `dir` argument, which silently falls back to `getTempDir()`. Make this explicit in production code for clarity.
- `createTempFile` opens the file in `w+` (read-write) mode. The file position starts at 0. After writing, call `setFilePos(0)` before reading back.
- The random segment uses only alphanumeric characters (`a-z`, `A-Z`, `0-9`), so generated names are safe on all major filesystems and shells.
- The module is marked **experimental** — pin your Nim version in production use.
