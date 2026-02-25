# `staticos` Module Reference

> **Module purpose:** `staticos` provides path-handling utilities that work **exclusively at compile time** (`{.compileTime.}`). It mirrors a subset of the standard `os` module but operates inside the Nim virtual machine (VM) during compilation — making it safe and functional even when **cross-compiling** to platforms where `os` is unavailable or unreliable.

---

## Why `staticos`?

When you cross-compile a Nim program for an embedded system, WebAssembly, or an exotic OS, the standard `os` module may not be available at runtime. But sometimes you need to inspect files or directories **while the compiler itself is running** — for example, to embed all files in a folder into your binary, or to assert that a required resource exists before compilation succeeds.

`staticos` solves this: all its procedures run inside the Nim compiler's VM at compile time. Your production binary never calls any of these functions — the work is done before the binary even exists.

---

## Type Reference

### `PathComponent`

```nim
type PathComponent* = enum
  pcFile,        # path refers to a regular file
  pcLinkToFile,  # path refers to a symbolic link to a file
  pcDir,         # path refers to a directory
  pcLinkToDir    # path refers to a symbolic link to a directory
```

An enumeration that classifies what kind of filesystem entry a path points to. It is used as the `kind` field in results returned by `staticWalkDir`.

| Value | Meaning |
|---|---|
| `pcFile` | A regular file |
| `pcLinkToFile` | A symbolic link whose target is a file |
| `pcDir` | A directory |
| `pcLinkToDir` | A symbolic link whose target is a directory |

---

## Procedure Reference

---

### `staticFileExists`

```nim
proc staticFileExists*(filename: string): bool {.compileTime.}
```

#### What it does

Returns `true` if `filename` exists on the **host** filesystem (the machine running the compiler) and is a **regular file** or a **symbolic link to a file**.

Returns `false` for:
- Directories
- Device files (`/dev/sda`, etc.)
- Named pipes (FIFOs)
- Sockets
- Anything that doesn't exist

#### Important distinction

This procedure checks the host machine at **compile time**, not the target machine at runtime. If you cross-compile on Linux for Windows, `staticFileExists` sees the Linux filesystem.

#### Example

```nim
# At compile time, verify a required config file is present.
# If it's missing, the build fails with a clear error message.

const configPath = "assets/config.json"

static:
  if not staticFileExists(configPath):
    error "Required file '" & configPath & "' is missing. Cannot compile."

# Safe to embed — we know the file exists.
const configData = staticRead(configPath)
```

Another common use: conditional compilation based on file presence.

```nim
static:
  when staticFileExists("src/platform_override.nim"):
    echo "Build note: using platform override"
```

---

### `staticDirExists`

```nim
proc staticDirExists*(dir: string): bool {.compileTime.}
```

#### What it does

Returns `true` if the path `dir` exists and is a **directory**. Follows symbolic links — if `dir` is a symlink pointing to a directory, the result is `true`.

Returns `false` if:
- The path does not exist
- The path exists but is a file, not a directory

#### Relationship with `staticFileExists`

These two procedures are complementary and mutually exclusive for regular filesystem entries: for a given path, at most one of them returns `true` (unless the path doesn't exist, in which case both return `false`).

#### Example

```nim
# Guard against missing asset directories before walking them.

const assetsDir = "assets/images"

static:
  if not staticDirExists(assetsDir):
    error "Assets directory '" & assetsDir & "' not found. Run 'make assets' first."

# Now safe to walk the directory and embed images.
```

You can also use it to choose between fallback paths:

```nim
const dataDir =
  when staticDirExists("data/override"):
    "data/override"
  else:
    "data/default"
```

---

### `staticWalkDir`

```nim
proc staticWalkDir*(dir: string; relative = false): seq[
    tuple[kind: PathComponent, path: string]] {.compileTime.}
```

#### What it does

Lists the contents of directory `dir` **one level deep** (non-recursive) and returns them as a `seq` of `(kind, path)` tuples. Each tuple contains:

- **`kind`** — a `PathComponent` value describing the type of the entry
- **`path`** — the path to the entry (absolute or relative, depending on the `relative` parameter)

#### The `relative` parameter

| `relative` value | Returned path |
|---|---|
| `false` (default) | Full path: `"assets/images/logo.png"` |
| `true` | Path relative to `dir`: `"logo.png"` |

#### Non-recursive behaviour

`staticWalkDir` **does not descend** into subdirectories. It only lists what is directly inside `dir`. To walk a tree recursively, call `staticWalkDir` again on each `pcDir` entry you encounter (see example below).

#### Example — embed all shaders from a directory

```nim
import staticos

const shadersDir = "src/shaders"

# Collect all .glsl files at compile time and embed their source code.
const shaderSources = block:
  var result: seq[tuple[name, src: string]]
  for (kind, path) in staticWalkDir(shadersDir, relative = false):
    if kind == pcFile and path.endsWith(".glsl"):
      result.add((name: path, src: staticRead(path)))
  result

# shaderSources is now a compile-time constant seq — zero I/O at runtime.
```

#### Example — recursive directory walk

```nim
proc collectFiles(dir: string): seq[string] {.compileTime.} =
  for (kind, path) in staticWalkDir(dir):
    case kind
    of pcFile, pcLinkToFile:
      result.add(path)
    of pcDir, pcLinkToDir:
      result.add(collectFiles(path))  # recurse

const allFiles = collectFiles("assets")
```

#### Example — inspecting entry types

```nim
static:
  for (kind, path) in staticWalkDir("resources", relative = true):
    case kind
    of pcFile:       echo "  file: ", path
    of pcLinkToFile: echo "  link→file: ", path
    of pcDir:        echo "  dir: ", path
    of pcLinkToDir:  echo "  link→dir: ", path
```

---

## Compile-Time vs Runtime: A Summary

| Feature | `staticos` | `os` module |
|---|---|---|
| Runs when | During compilation | At runtime |
| Sees | Host filesystem | Target filesystem |
| Cross-compile safe | ✅ Yes | ⚠️ May fail |
| Result embedded in binary | As `const` | No |
| Recursive walking | Manual (call again) | `walkDirRec` available |

---

## Common Patterns

### Pattern 1 — Fail-fast build validation

```nim
static:
  doAssert staticFileExists("LICENSE"), "LICENSE file must be present in project root"
  doAssert staticDirExists("assets"), "assets/ directory is required"
```

### Pattern 2 — Auto-embed a whole directory

```nim
const files = block:
  var t: seq[tuple[name, content: string]]
  for (kind, path) in staticWalkDir("web/static"):
    if kind == pcFile:
      t.add((path, staticRead(path)))
  t
```

### Pattern 3 — Conditional compilation by file presence

```nim
when staticFileExists("src/debug_helpers.nim"):
  include "src/debug_helpers.nim"
```

---

## Notes

- All procedures are **`{.compileTime.}`** — calling them outside a `static:` block, a `const` definition, or another compile-time context is a compile error.
- Paths are interpreted relative to the **current working directory of the compiler**, which is usually the project root (where you run `nim c`).
- The actual implementations are provided by the Nim compiler's VM ops layer (`vmops`), not by Nim code itself.
