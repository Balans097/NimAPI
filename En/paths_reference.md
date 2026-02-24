# `paths` Module Reference

> **Module:** `std/paths`  
> **Language:** Nim  
> **Purpose:** Type-safe, cross-platform path manipulation — joining, splitting, normalizing, and interrogating filesystem paths without performing any actual I/O.

---

## Core Idea: Why a `Path` Type?

In most languages, filesystem paths are plain strings. This causes a silent class of bugs: you can accidentally pass a URL where a path is expected, concatenate paths with `&` instead of the proper separator, or compare paths with `==` that are actually the same file spelled differently. None of these mistakes are caught at compile time.

`std/paths` solves this by making `Path` a **distinct type** — it is backed by a `string` internally, but the compiler treats it as a completely different type. You cannot accidentally mix `Path` with `string` or with other distinct string types. All path operations in this module work exclusively with `Path` values, and the type system enforces correctness at compile time.

This module is **purely computational**: it manipulates path strings. It performs no I/O, makes no system calls, and does not check whether any file or directory actually exists. For operations that touch the filesystem, see `std/files` and `std/dirs`.

---

## Exported Platform Constants (from `osseps`)

These constants describe the separator conventions of the current operating system. They are re-exported so you can write cross-platform code without importing `osseps` directly.

| Constant | Unix | Windows | Description |
|---|---|---|---|
| `DirSep` | `'/'` | `'\\'` | Primary directory separator character |
| `AltSep` | `'/'` | `'/'` | Alternative separator (on Windows, `/` is also accepted) |
| `PathSep` | `':'` | `';'` | Separator between multiple paths in `PATH`-style variables |
| `FileSystemCaseSensitive` | `true` | `false` | Whether the OS filesystem distinguishes `Foo` from `foo` |
| `ExeExt` | `""` | `"exe"` | Extension of executable files (without leading dot) |
| `ScriptExt` | `""` | `"bat"` | Extension of script files |
| `DynlibFormat` | `"lib$1.so"` | `"$1.dll"` | Format string for shared library filenames |

---

## The `Path` Type

```nim
type Path* = distinct string
```

A `Path` is a `string` under the hood, but treated as a completely separate type by the compiler. You convert between them explicitly:

```nim
let p = Path("/usr/local/bin")   # wrap a string literal into a Path
let s = string(p)                 # or: p.string — unwrap back to string
echo $p                           # $ converts Path to string for display
```

---

## Operators

### `/` — Path Joining

```nim
func `/`*(head, tail: Path): Path
```

**What it does:** Joins two path components into one, placing the platform-appropriate separator between them. This is the safe, idiomatic way to build paths in Nim — never use string concatenation `&` for this purpose.

The operator normalizes the result: redundant separators are collapsed, `.` components (current directory) are simplified, and the trailing separator is preserved if and only if `tail` ends with one (or if `tail` is empty and `head` ends with one). It does **not** resolve `..` or symlinks — for that, use `normalizePath` or `absolutePath`.

```nim
import std/paths

let base = Path("/home/user")
let config = Path(".config/myapp")

let full = base / config
assert $full == "/home/user/.config/myapp"

# Works correctly regardless of trailing slashes:
assert $(Path("/usr/") / Path("bin")) == "/usr/bin"
assert $(Path("/usr")  / Path("bin")) == "/usr/bin"

# Building paths piece by piece:
let log = Path("/var") / Path("log") / Path("myapp") / Path("error.log")
assert $log == "/var/log/myapp/error.log"
```

---

### `/../` — Go Up Then Down

```nim
func `/../`*(head, tail: Path): Path
```

**What it does:** Equivalent to `parentDir(head) / tail`, but safe when `head` has no parent. If `head` already is a root directory (has no parent), it falls back to `head / tail` instead of going above the root.

Think of it as "go up one level from `head`, then go down into `tail`".

```nim
import std/paths

# Normal case: go up from "project/src", then into "docs":
let p = Path("project/src") /.. Path("docs")
# parentDir("project/src") == "project", then / "docs"
assert $p == "project/docs"

# Edge case: head is root — stays at root:
let q = Path("/") /.. Path("etc")
assert $q == "/etc"
```

---

### `==` — Path Comparison

```nim
func `==`*(x, y: Path): bool
```

**What it does:** Compares two paths for equality in a platform-aware manner. On case-sensitive filesystems (Linux, most Unix), the comparison is case-sensitive. On case-insensitive filesystems (Windows, macOS by default), the comparison is case-insensitive.

This means `Path("Foo/Bar") == Path("foo/bar")` is `true` on Windows but `false` on Linux — matching the actual behavior of the underlying filesystem.

```nim
import std/paths

when defined(windows):
  assert Path("C:/Users/Alice") == Path("c:/users/alice")
else:
  assert Path("/home/alice") == Path("/home/alice")
  assert Path("/home/Alice") != Path("/home/alice")
```

---

### `$` — Convert to String

```nim
template `$`*(x: Path): string
```

**What it does:** Converts a `Path` to a plain `string`, suitable for passing to `echo`, string formatting, or any API that does not accept `Path` directly.

```nim
import std/paths

let p = Path("/etc/hosts")
echo $p            # /etc/hosts
echo "File: " & $p # File: /etc/hosts
```

---

### `add` — In-Place Append

```nim
func add*(x: var Path, y: Path)
```

**What it does:** Appends `y` to `x` in-place, with the same normalization semantics as `/`. Useful when building a path incrementally inside a loop without creating intermediate `Path` values.

```nim
import std/paths

var p = Path("/var/log")
p.add Path("myapp")
p.add Path("access.log")
assert $p == "/var/log/myapp/access.log"
```

---

## Splitting and Decomposition

### `splitPath`

```nim
func splitPath*(path: Path): tuple[head, tail: Path]
```

**What it does:** Splits `path` into a `(head, tail)` pair where `head` is everything up to the last separator and `tail` is the final component. The invariant `head / tail == path` holds for most paths (with edge cases like `"/"` which has an empty tail).

This is the low-level split: it always divides at the last separator regardless of whether the last component is a file or a directory.

```nim
import std/paths

let (head, tail) = splitPath(Path("/home/user/documents/report.txt"))
assert $head == "/home/user/documents"
assert $tail == "report.txt"

let (h2, t2) = splitPath(Path("/usr"))
assert $h2 == "/"
assert $t2 == "usr"

# Round-trip:
assert head / tail == Path("/home/user/documents/report.txt")
```

---

### `splitFile`

```nim
func splitFile*(path: Path): tuple[dir, name: Path, ext: string]
```

**What it does:** Splits `path` into three parts: the directory component, the base filename (without extension), and the extension (including the leading dot). This is the most complete decomposition — useful when you need to reconstruct a path with a different name or extension.

Rules:
- `dir` does not end with a separator (unless it is the root `/`).
- `ext` includes the leading `.` (e.g. `".txt"`), or is `""` if there is no extension.
- If the path has no directory component, `dir` is an empty `Path`.
- Dotfiles like `.bashrc` have an empty `name` and `.bashrc` as the `ext` — wait, actually: `.bashrc` is treated as a name with no extension by most path libraries. Verify with your Nim version.

```nim
import std/paths

let (dir, name, ext) = splitFile(Path("/home/user/report.tar.gz"))
assert $dir  == "/home/user"
assert $name == "report.tar"
assert ext   == ".gz"

# Reconstructing with a different extension:
let newPath = dir / Path($name & ".bz2")
assert $newPath == "/home/user/report.tar.bz2"

# No extension:
let (d2, n2, e2) = splitFile(Path("/usr/bin/ls"))
assert $n2 == "ls"
assert e2  == ""

# No directory:
let (d3, n3, e3) = splitFile(Path("readme.md"))
assert $d3 == ""
assert $n3 == "readme"
assert e3  == ".md"
```

---

## Filename Extraction

### `extractFilename`

```nim
func extractFilename*(path: Path): Path
```

**What it does:** Returns only the filename portion of `path` — the last component including its extension. Equivalent to `splitFile(path).name & splitFile(path).ext`, or `splitPath(path).tail`.

A trailing separator means the path refers to a directory: in that case `extractFilename` returns an empty `Path`.

```nim
import std/paths

assert $extractFilename(Path("/home/user/photo.jpg")) == "photo.jpg"
assert $extractFilename(Path("/usr/bin/nim"))         == "nim"
assert $extractFilename(Path("/var/log/"))            == ""
```

---

### `lastPathPart`

```nim
func lastPathPart*(path: Path): Path
```

**What it does:** Like `extractFilename`, but **ignores trailing separators**. If `path` ends with a `/`, it strips it first and then returns the last component. This corresponds to `basename` in shell scripts and most other languages.

Use `lastPathPart` when you want the meaningful final name of a path, whether it refers to a file or a directory.

```nim
import std/paths

assert $lastPathPart(Path("/home/user/photo.jpg")) == "photo.jpg"
assert $lastPathPart(Path("/var/log/"))             == "log"   # trailing / ignored
assert $lastPathPart(Path("/usr/bin"))              == "bin"
```

---

## Extension Manipulation

### `changeFileExt`

```nim
func changeFileExt*(filename: Path, ext: string): Path
```

**What it does:** Returns a new path identical to `filename` but with the file extension replaced by `ext`. If `filename` has no extension, `ext` is added. If `ext` is `""`, any existing extension is removed.

Pass `ext` **without** the leading dot — `"txt"`, not `".txt"`. The function adds the dot for you.

```nim
import std/paths

assert $changeFileExt(Path("report.docx"), "pdf")  == "report.pdf"
assert $changeFileExt(Path("/src/main.nim"), "c")  == "/src/main.c"
assert $changeFileExt(Path("archive.tar.gz"), "")  == "archive.tar"
assert $changeFileExt(Path("Makefile"), "bak")     == "Makefile.bak"
```

---

### `addFileExt`

```nim
func addFileExt*(filename: Path, ext: string): Path
```

**What it does:** Adds extension `ext` to `filename`, but **only if** the filename does not already have an extension. If an extension is already present, returns `filename` unchanged. Like `changeFileExt`, pass `ext` without the leading dot.

This is the "ensure an extension exists" variant — it will not double-add extensions.

```nim
import std/paths

assert $addFileExt(Path("report"), "pdf")       == "report.pdf"
assert $addFileExt(Path("report.pdf"), "pdf")   == "report.pdf"  # unchanged
assert $addFileExt(Path("archive.tar"), "gz")   == "archive.tar" # already has ext
```

---

## Directory Navigation

### `parentDir`

```nim
func parentDir*(path: Path): Path
```

**What it does:** Returns the parent directory of `path`. Handles path normalization: a trailing separator is stripped before computing the parent, so `parentDir(Path("/foo/bar/"))` correctly returns `/foo`, not `/foo/bar`.

If `path` is already a root directory, `parentDir` returns the root itself.

```nim
import std/paths

assert $parentDir(Path("/home/user/file.txt"))  == "/home/user"
assert $parentDir(Path("/home/user/"))          == "/home"
assert $parentDir(Path("/"))                    == "/"

# Works with relative paths too:
assert $parentDir(Path("project/src/main.nim")) == "project/src"
```

---

### `tailDir`

```nim
func tailDir*(path: Path): Path
```

**What it does:** Returns everything in `path` after the first directory component — the opposite of `parentDir`. If `path` has only one component (or is a root), returns an empty `Path`.

This is useful for stripping a known prefix from a path.

```nim
import std/paths

assert $tailDir(Path("/home/user/file.txt"))  == "user/file.txt"
assert $tailDir(Path("/usr"))                 == ""
assert $tailDir(Path("a/b/c"))               == "b/c"
```

---

### `parentDirs` *(iterator)*

```nim
iterator parentDirs*(path: Path, fromRoot=false, inclusive=true): Path
```

**What it does:** Iterates over all ancestor directories of `path`, yielding each one as a `Path`.

- **`inclusive = true`** (default): the iteration includes `path` itself as the first (or last) yielded value.
- **`inclusive = false`**: only the strict ancestors are yielded; `path` itself is excluded.
- **`fromRoot = false`** (default): traversal proceeds from `path` upward toward the root. The first value yielded is `path` (or its parent if `inclusive = false`), and each subsequent value is one level closer to the root.
- **`fromRoot = true`**: traversal proceeds from the root downward toward `path`. The first value is the root.

Relative paths are traversed as-is without being expanded to absolute paths first.

```nim
import std/paths

# From leaf to root (default):
for p in parentDirs(Path("/a/b/c/file.txt")):
  echo $p
# /a/b/c/file.txt
# /a/b/c
# /a/b
# /a
# /

# From root to leaf, excluding the path itself:
for p in parentDirs(Path("/a/b/c"), fromRoot=true, inclusive=false):
  echo $p
# /
# /a
# /a/b
```

---

## Absolute vs. Relative

### `isAbsolute`

```nim
func isAbsolute*(path: Path): bool
```

**What it does:** Returns `true` if `path` is an absolute path. On Unix, this means it starts with `/`. On Windows, this includes both drive-letter paths (`C:\...`) and UNC network paths (`\\server\share`).

```nim
import std/paths

assert isAbsolute(Path("/usr/bin"))        == true
assert isAbsolute(Path("relative/path"))   == false
assert isAbsolute(Path("./local"))         == false
```

---

### `isRootDir`

```nim
func isRootDir*(path: Path): bool
```

**What it does:** Returns `true` if `path` is a filesystem root — `/` on Unix, a drive root like `C:\` on Windows.

```nim
import std/paths

assert isRootDir(Path("/"))    == true
assert isRootDir(Path("/usr")) == false
```

---

### `isRelativeTo`

```nim
proc isRelativeTo*(path: Path, base: Path): bool
```

**What it does:** Returns `true` if `path` is a descendant of `base` — that is, `base` is a prefix of `path` in terms of path components (not just string prefix). Both paths should ideally be absolute for a meaningful result.

```nim
import std/paths

assert isRelativeTo(Path("/home/user/docs"), Path("/home/user"))  == true
assert isRelativeTo(Path("/home/user"),      Path("/home/user"))  == true
assert isRelativeTo(Path("/home/bob"),       Path("/home/user"))  == false
```

---

### `relativePath`

```nim
proc relativePath*(path, base: Path, sep = DirSep): Path
```

**What it does:** Computes a relative path from `base` to `path` — the path you would need to write if your current directory were `base` and you wanted to reach `path`.

The optional `sep` parameter controls the separator used in the output. Setting `sep = '/'` ensures the result uses forward slashes only, which is useful for URL construction or cross-platform relative path output.

On Windows, if `path` and `base` are on different drives, it is impossible to form a relative path — the function returns `path` unchanged (which is absolute).

```nim
import std/paths

assert $relativePath(Path("/home/user/docs"), Path("/home/user")) == "docs"
assert $relativePath(Path("/a/b/c"), Path("/a/x/y"))             == "../../b/c"
assert $relativePath(Path("/a/b"),   Path("/a/b"))               == "."

# Force forward slashes in the output:
assert $relativePath(Path("/a/b/c"), Path("/a"), '/') == "b/c"
```

---

## Normalization

### `normalizePath`

```nim
proc normalizePath*(path: var Path)
```

**What it does:** Normalizes `path` in-place by resolving `.` (current directory) and `..` (parent directory) components, collapsing redundant separators, and adjusting separators to the platform convention. Does **not** make the path absolute and does **not** resolve symlinks.

```nim
import std/paths

var p = Path("/home/user/./docs/../photos")
normalizePath(p)
assert $p == "/home/user/photos"

var q = Path("a//b///c")
normalizePath(q)
assert $q == "a/b/c"
```

---

### `normalizePathEnd`

```nim
proc normalizePathEnd*(path: var Path, trailingSep = false)
```

**What it does:** Adjusts only the trailing separator of `path`. If `trailingSep = true`, ensures the path ends with a separator. If `trailingSep = false` (default), ensures it does not. Does not touch any other part of the path.

Useful when you need to ensure consistency of the trailing separator before comparing or joining paths.

```nim
import std/paths

var p = Path("/home/user/")
normalizePathEnd(p, trailingSep = false)
assert $p == "/home/user"

var q = Path("/home/user")
normalizePathEnd(q, trailingSep = true)
assert $q == "/home/user/"
```

---

### `normalizeExe`

```nim
proc normalizeExe*(file: var Path)
```

**What it does:** Normalizes an executable path. On Unix, if `file` contains no separator (i.e. it is a bare name like `"python"`), it is left unchanged — it will be found via `PATH`. If it does contain a separator, `"./"` is prepended if not already present, ensuring it is treated as an explicit path rather than a `PATH` lookup. On Windows, ensures the `.exe` extension is present.

```nim
import std/paths

var p = Path("myprogram")
normalizeExe(p)
# On Unix: p == "myprogram" (searched via PATH)

var q = Path("./myprogram")
normalizeExe(q)
# On Unix: q == "./myprogram" (explicit relative path)
```

---

### `absolutePath`

```nim
proc absolutePath*(path: Path, root = getCurrentDir()): Path
```

**What it does:** If `path` is already absolute, returns it unchanged. If `path` is relative, resolves it against `root` (which defaults to the current working directory) and returns the resulting absolute path. Then normalizes the result.

Note: this is a **lexical** operation — it does not call `realpath` or resolve symlinks. Use it to produce an absolute path string; do not rely on it to guarantee the path refers to an existing file.

```nim
import std/paths

# Absolute paths pass through unchanged:
assert $absolutePath(Path("/etc/hosts")) == "/etc/hosts"

# Relative paths are resolved against CWD by default:
let abs = absolutePath(Path("src/main.nim"))
assert isAbsolute(abs)

# Or against an explicit root:
let abs2 = absolutePath(Path("config.toml"), root = Path("/opt/myapp"))
assert $abs2 == "/opt/myapp/config.toml"
```

---

## Working Directory and User Paths

### `getCurrentDir`

```nim
proc getCurrentDir*(): Path
```

**What it does:** Returns the current working directory of the running process as an absolute `Path`. This is determined at runtime, not at compile time — it reflects wherever the process was started or wherever `setCurrentDir` (from `std/dirs`) last changed it to.

```nim
import std/paths

let cwd = getCurrentDir()
echo "Running from: ", $cwd

# Useful as a base for resolving relative paths:
let configFile = cwd / Path("config.toml")
```

---

### `expandTilde`

```nim
proc expandTilde*(path: Path): Path
```

**What it does:** Expands a leading `~` in `path` to the user's home directory (as returned by `getHomeDir()` from `std/appdirs`). The expansion rules are:

- `Path("~")` alone → the full home directory path.
- `Path("~/foo")` → home directory joined with `foo`.
- `Path("~\foo")` → same on Windows (both separators are handled).
- Any path that does not start with `~` → returned unchanged.

Note: the `~username` form (home directory of a specific user) is **not** supported and is returned as-is.

```nim
import std/paths, std/appdirs

let home = getHomeDir()

assert expandTilde(Path("~"))            == home
assert expandTilde(Path("~/Documents"))  == home / Path("Documents")
assert expandTilde(Path("/etc/hosts"))   == Path("/etc/hosts")  # unchanged
```

---

### `unixToNativePath`

```nim
func unixToNativePath*(path: Path, drive = Path("")): Path
```

**What it does:** Converts a Unix-style path (using `/`, `.`, `..`) to the native path format of the current operating system. On Unix systems, this is a no-op. On Windows, it converts `/` to `\`, and handles `.` and `..` appropriately.

The `drive` parameter is used on systems that have drive letters. If `path` is absolute (starts with `/`) and `drive` is provided, the drive letter prefix is applied. If `drive` is empty (default), the drive of the current working directory is used.

```nim
import std/paths

when defined(windows):
  assert $unixToNativePath(Path("/home/user")) == "\\home\\user"
else:
  assert $unixToNativePath(Path("/home/user")) == "/home/user"
```

---

## Hashing

### `hash`

```nim
func hash*(x: Path): Hash
```

**What it does:** Computes a hash for `Path` suitable for use in `Table`, `HashSet`, and other hash-based containers. The hash is computed after normalizing the path (so `"/foo/./bar"` and `"/foo/bar"` produce the same hash) and is case-insensitive on case-insensitive filesystems.

This function is not usually called directly — it is invoked automatically when a `Path` is used as a hash table key.

```nim
import std/[paths, tables]

var registry: Table[Path, string]
registry[Path("/etc/hosts")] = "system hosts file"
registry[Path("/etc/./hosts")] = "overwrite"  # same key after normalization

assert registry.len == 1
```

---

## Quick Reference: Decomposition Functions

| Function | Input | Returns |
|---|---|---|
| `splitPath` | `"/a/b/c.txt"` | `head="/a/b"`, `tail="c.txt"` |
| `splitFile` | `"/a/b/c.tar.gz"` | `dir="/a/b"`, `name="c.tar"`, `ext=".gz"` |
| `extractFilename` | `"/a/b/c.txt"` | `"c.txt"` |
| `lastPathPart` | `"/a/b/c/"` | `"c"` (trailing `/` ignored) |
| `parentDir` | `"/a/b/c.txt"` | `"/a/b"` |
| `tailDir` | `"/a/b/c"` | `"b/c"` |

## Quick Reference: Extension Functions

| Function | Has extension already? | Behaviour |
|---|---|---|
| `changeFileExt(p, "pdf")` | Yes | Replaces existing extension |
| `changeFileExt(p, "pdf")` | No | Adds the extension |
| `changeFileExt(p, "")` | Yes | Removes the extension |
| `addFileExt(p, "pdf")` | Yes | **Returns unchanged** |
| `addFileExt(p, "pdf")` | No | Adds the extension |
