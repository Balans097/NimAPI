# `pathnorm` Module Reference

> **Module:** `std/pathnorm`  
> **Stability:** Unstable API  
> **Purpose:** OS-path normalisation — collapsing redundant separators, resolving `.` and `..` components, and iterating over path segments without allocating intermediate strings.

---

## Overview

File-system paths are frequently messier than they need to be. User input, string concatenation, and cross-platform tooling all tend to produce paths like `./foo//bar/../baz`. Before comparing paths, passing them to system calls, or displaying them in error messages, it is important to reduce them to a canonical, clean form.

`pathnorm` provides two layers for this:

- **High level:** `normalizePath` — takes a raw path string, returns a clean one. This is the function you reach for in everyday code.
- **Low level:** `PathIter` + `next` + `hasNext` — a zero-allocation iterator that yields the byte-slice bounds of each individual path component (segment) inside the original string. Used internally by `normalizePath` and by `os.joinPath`, but also available for callers that need fine-grained path processing without materialising intermediate strings.

### What "normalisation" means

| Input pattern | What happens |
|---|---|
| Multiple consecutive separators (`foo//bar`) | Collapsed into one (`foo/bar`) |
| Current-directory dot (`./foo`) | The `.` component is discarded (`foo`) |
| Parent-directory dotdot (`foo/bar/../baz`) | The preceding component is removed (`foo/baz`) |
| Dotdot at root (`/..`) | Left as-is; cannot go above root |
| Dotdot with nothing to cancel (`../foo`) | Kept as-is |
| Entire path reduces to empty | Replaced with `"."` |
| Windows drive prefix (`C:\…`) | Kept verbatim at the front |

Normalisation operates purely on the string — it does **not** consult the real file system. Symlinks are not resolved.

### Platform awareness

The module uses `DirSep` and `AltSep` from `std/private/osseps`, which are the platform-native separator constants. On POSIX, `DirSep = '/'`. On Windows (and DOS-like systems), `DirSep = '\\'` and `AltSep = '/'`. All public procedures accept a `dirSep` parameter so you can override the separator when processing paths for a different platform or a path format with specific conventions.

---

## Exported Types

---

### `PathIter`

```nim
type PathIter* = object
  i, prev: int
  notFirst: bool
```

#### What it is

A lightweight cursor object used to iterate over the components of a path string one segment at a time. `PathIter` holds only three integers and operates entirely on the original string — it does not copy or allocate any substring. Each call to `next` advances the cursor and returns the byte-index bounds of the next component within the original string.

#### Fields

All fields are public (exported), but you should treat them as opaque state managed by `next` and `hasNext`. Do not set them manually.

| Field | Type | Internal meaning |
|---|---|---|
| `i` | `int` | Current scan position in the path string. |
| `prev` | `int` | Position before the most recent `next` call; used internally to compute segment bounds. |
| `notFirst` | `bool` | `false` before the first `next` call; set to `true` afterwards, enabling the iterator to distinguish the leading separator of an absolute path from subsequent separators. |

#### How to use it

Always initialise with `default(PathIter)` (or `var it: PathIter` which zero-initialises), then loop with `hasNext` / `next`:

```nim
import std/pathnorm

let path = "/usr/local/bin"
var it = default(PathIter)
while hasNext(it, path):
  let (a, b) = next(it, path)
  echo path[a..b]
# Output:
# /
# usr
# local
# bin
```

---

## Exported Procedures

---

### `hasNext`

```nim
proc hasNext*(it: PathIter; x: string): bool
```

#### What it does

Returns `true` if there are still unvisited characters in the path string `x` — that is, if calling `next` would yield another component. This is the loop-condition check used alongside `next`.

The check is simply `it.i < x.len`. It does not peek at the content; a tail of separators with no further content will still report `true` until `next` advances past them.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `it` | `PathIter` | The iterator cursor (read-only for this call). |
| `x` | `string` | The path string being iterated. |

#### Example

```nim
import std/pathnorm

var it = default(PathIter)
let p = "a/b"
echo hasNext(it, p)  # → true (not started yet)
discard next(it, p)  # consume "a"
echo hasNext(it, p)  # → true  ("b" remains)
discard next(it, p)  # consume "b"
echo hasNext(it, p)  # → false (exhausted)
```

---

### `next`

```nim
proc next*(it: var PathIter; x: string): (int, int)
```

#### What it does

Advances the `PathIter` cursor to the next path component in `x` and returns its bounds as a tuple `(startIndex, endIndex)` — both inclusive — into the original string. To extract the component text, use `x[startIndex..endIndex]`.

The procedure handles the special case of the leading separator in absolute paths: on the very first call, if the path starts with a separator, the function yields `(0, 0)` pointing at the separator itself (representing the root `/`). After that, it yields each non-separator segment. Runs of consecutive separators between segments are automatically skipped after yielding each component.

#### Return value

`(a, b)` where `x[a..b]` is the text of the component. For the root component of an absolute path, `x[a..b]` is `"/"` (or `"\\"` on Windows). For a regular segment, it is the segment text without separators.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `it` | `var PathIter` | The iterator cursor; advanced in place. |
| `x` | `string` | The path string being iterated. |

#### Examples

```nim
import std/pathnorm

# Relative path
var it = default(PathIter)
let p = "foo/bar/baz"
while hasNext(it, p):
  let (a, b) = next(it, p)
  echo p[a..b]
# foo
# bar
# baz
```

```nim
import std/pathnorm

# Absolute path — first component is the root separator
var it = default(PathIter)
let p = "/etc/hosts"
while hasNext(it, p):
  let (a, b) = next(it, p)
  echo repr(p[a..b])
# "/"
# "etc"
# "hosts"
```

```nim
import std/pathnorm

# Multiple consecutive separators are collapsed during iteration
var it = default(PathIter)
let p = "a//b///c"
while hasNext(it, p):
  let (a, b) = next(it, p)
  echo p[a..b]
# a
# b
# c
```

```nim
import std/pathnorm

# Custom processing: count depth of a path without allocations
proc pathDepth(p: string): int =
  var it = default(PathIter)
  while hasNext(it, p):
    let (a, b) = next(it, p)
    if p[a..b] notin ["/", ".", ".."]:
      inc result

echo pathDepth("usr/local/lib")   # → 3
echo pathDepth("/usr/local/lib")  # → 3 (root "/" not counted)
echo pathDepth("./foo/../bar")    # → 2 (dot and dotdot counted as-is here)
```

---

### `normalizePath`

```nim
proc normalizePath*(path: string; dirSep = DirSep): string
```

#### What it does

The main high-level function of the module. Takes a raw path string and returns a new, normalised string. It applies all the cleaning rules described in the overview:

- Collapses runs of consecutive directory separators into one.
- Removes `.` (current-directory) components.
- Resolves `..` (parent-directory) components by removing the preceding path component, as long as there is one to remove.
- If the entire path normalises to an empty string (e.g., `"."` alone), returns `"."` instead of an empty string.
- On Windows/DOS systems, preserves the drive letter prefix (e.g., `C:`) unchanged.

Normalisation is purely lexical — no file system access, no symlink resolution.

#### Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `path` | `string` | *(required)* | The raw path to normalise. |
| `dirSep` | `char` | `DirSep` | The separator character to use in the output. Defaults to the platform-native separator. Override to produce POSIX paths on Windows or vice versa. |

#### Return value

A new string with the normalised path. The input string is never modified.

#### Examples

```nim
import std/pathnorm

# Collapsing double slashes
echo normalizePath("foo//bar")        # → foo/bar
```

```nim
import std/pathnorm

# Removing current-directory dots
echo normalizePath("./foo/./bar")     # → foo/bar
echo normalizePath(".")               # → .
```

```nim
import std/pathnorm

# Resolving parent-directory references
echo normalizePath("foo/bar/../baz")  # → foo/baz
echo normalizePath("/foo/../bar")     # → /bar
```

```nim
import std/pathnorm

# Dotdot that cannot be resolved stays in place
echo normalizePath("../foo")          # → ../foo
echo normalizePath("../../a/b")       # → ../../a/b
```

```nim
import std/pathnorm

# A path that entirely cancels out becomes "."
echo normalizePath("foo/..")          # → .
```

```nim
import std/pathnorm

# Combining everything
echo normalizePath("./foo//bar/../baz")  # → foo/baz
```

```nim
import std/pathnorm

# Cross-platform: produce a POSIX path even on Windows
echo normalizePath("C:\\Users\\..\\Public", dirSep = '/')
# → C:/Public  (drive letter preserved, slashes converted)
```

```nim
import std/pathnorm, std/os

# Practical use: normalise user-provided paths before passing to OS
proc openSafe(userPath: string) =
  let clean = normalizePath(userPath)
  echo "Opening: ", clean
  # now pass `clean` to open(), readFile(), etc.

openSafe("./data/../config/app.cfg")
# Opening: config/app.cfg
```

---

### `addNormalizePath`

```nim
proc addNormalizePath*(x: string; result: var string; state: var int;
    dirSep = DirSep)
```

> **Note:** The module documentation marks this procedure as *"Low level proc. Undocumented."* It is nonetheless exported and used by `os.joinPath`. The description below reflects its observable behaviour.

#### What it does

An incremental, stateful variant of `normalizePath`. Instead of returning a new string, it **appends** the normalised form of `x` into an existing `result` string, updating a `state` integer that carries normalisation context between calls. This allows joining and normalising multiple path segments in a single pass without repeated string allocation.

`normalizePath` itself is implemented as a one-liner that calls `addNormalizePath` with a fresh empty string and a zero state.

#### When to use it

Use `addNormalizePath` when you need to concatenate several path fragments and want the result normalised as a whole, not fragment-by-fragment. `os.joinPath` uses this internally. In application code, prefer `os.joinPath` or `normalizePath` unless you specifically need incremental construction.

#### State encoding

The `state` integer is a packed bitfield:

| Bits | Meaning |
|---|---|
| Bit 0 (`state and 1`) | Set to 1 if the accumulated path so far is absolute (starts with a separator). |
| Bits 1+ (`state shr 1`) | The count of non-root, non-dot-dot path components accumulated so far. Each normal component adds 2; each `..` that successfully cancels a component subtracts 2. |

You must initialise `state` to `0` at the start of a new path. You must use the same `result` and `state` across multiple `addNormalizePath` calls for the same logical path.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `x` | `string` | The path fragment to append and normalise. |
| `result` | `var string` | The accumulation buffer. Pass an empty string to start a new path. |
| `state` | `var int` | Normalisation state. Must be `0` at the start; carried between calls. |
| `dirSep` | `char` | Output separator character; defaults to `DirSep`. |

#### Examples

```nim
import std/pathnorm

# Replicating what normalizePath does internally:
var res = ""
var st = 0
addNormalizePath("./foo//bar/../baz", res, st)
echo res  # → foo/baz
```

```nim
import std/pathnorm

# Incremental construction: append multiple fragments
var res = ""
var st = 0
addNormalizePath("/usr",   res, st)
addNormalizePath("local",  res, st)
addNormalizePath("../lib", res, st)  # ".." cancels "local"
echo res  # → /usr/lib
```

```nim
import std/pathnorm

# State carries over: absolute-path flag persists across calls
var res = ""
var st = 0
addNormalizePath("/",    res, st)
addNormalizePath("etc",  res, st)
addNormalizePath("hosts",res, st)
echo res  # → /etc/hosts
echo (st and 1)  # → 1  (path is absolute)
```

---

## Design Notes and Pitfalls

**1. The API is marked Unstable**  
The module header warns *"Unstable API."* Types and procedure signatures may change between Nim releases. For code that must remain stable across Nim versions, consider copying or wrapping only the `normalizePath` function.

**2. Normalisation is lexical, not semantic**  
`normalizePath("foo/../bar")` yields `"bar"` even if `foo` is a symlink to `/completely/different/directory` — in that case the real resolved path would be `/completely/different/directory/../bar` = `/completely/different/bar`. If you need symlink-aware resolution, use `os.expandFilename` instead.

**3. `PathIter` state is tied to a specific string**  
An iterator value is only meaningful in combination with the exact string it was created for. Do not reuse the same `PathIter` with a different string, and do not modify the string while iterating.

**4. `next` must only be called when `hasNext` is true**  
Calling `next` on an exhausted iterator (when `it.i >= x.len`) will access the string out of bounds. Always guard with `hasNext`.

**5. `addNormalizePath` state must be reset between unrelated paths**  
Reusing a `state` variable from a previous path with a new `result` and path string will produce wrong results because the component count and absolute-path flag carry over incorrectly.

**6. Empty-path edge case**  
`normalizePath("")` returns `"."`. If an empty path has a distinct meaning in your domain (e.g., "use the current directory"), handle it before calling `normalizePath`.

**7. The `dirSep` parameter affects only the output, not parsing**  
Both `/` and `\` (and their platform-specific `AltSep`) are always recognised as separators during parsing, regardless of the `dirSep` argument. The argument only controls which character appears in the returned string.
