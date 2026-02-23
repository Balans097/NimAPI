# `dynlib` Module Reference

**Module:** `dynlib` (part of Nim's standard library)  
**Purpose:** Load shared libraries (`.so`, `.dll`, `.dylib`) at runtime, resolve symbols by name, and call or access them from Nim code.

---

## Overview

Most of the time Nim links libraries at *compile time* — you list them in your `.nimble` file or pass `-l` flags, and the OS loader wires everything up before `main` runs. But sometimes you genuinely need to decide *at runtime* which library to open: plugin systems, optional hardware drivers, language-selectable backends, hot-reloading, and so on. `dynlib` provides exactly that capability.

The module is a thin, cross-platform wrapper:

| Platform | Underlying mechanism |
|---|---|
| Linux / macOS / most POSIX | `dlopen` / `dlsym` / `dlclose` (from `<dlfcn.h>`) |
| Windows | `LoadLibraryA` / `GetProcAddress` / `FreeLibrary` |
| Nintendo Switch | Not supported — raises `OSError` |
| Genode (without POSIX) | Not supported — raises `OSError` |

Everything revolves around a single opaque type, `LibHandle`, which is just a `pointer` under the hood. You open a library → get a handle → resolve symbols through it → close it when done.

### Typical workflow

```
loadLib / loadLibPattern
        │
        ▼
   LibHandle ──► symAddr / checkedSymAddr ──► cast ──► call
        │
        ▼
    unloadLib
```

---

## Exported API

| Symbol | Kind | Short description |
|---|---|---|
| `LibHandle` | type alias | Opaque handle to a loaded library |
| `loadLib(path, globalSymbols)` | proc | Load a library by file path |
| `loadLib()` | proc | Get a handle to the running executable itself |
| `unloadLib(lib)` | proc | Release a loaded library |
| `symAddr(lib, name)` | proc | Look up a symbol; returns `nil` on failure |
| `checkedSymAddr(lib, name)` | proc | Look up a symbol; raises on failure |
| `raiseInvalidLibrary(name)` | proc | Raise `LibraryError` for a missing symbol |
| `libCandidates(s, dest)` | proc | Expand a name pattern into candidate filenames |
| `loadLibPattern(pattern, globalSymbols)` | proc | Load the first library that matches a pattern |

---

## `LibHandle`

```nim
type LibHandle* = pointer
```

### What it is

`LibHandle` is an opaque token that represents a successfully loaded shared library. Its concrete value is a platform pointer (`HMODULE` on Windows, `void*` from `dlopen` on POSIX). You never inspect it directly — you only pass it to `symAddr`, `checkedSymAddr`, and `unloadLib`.

A `nil` `LibHandle` means "no library loaded" and is the value returned by the load functions on failure.

### Example

```nim
import dynlib

let lib: LibHandle = loadLib("mylib.so")
if lib == nil:
  echo "Library not found"
else:
  echo "Loaded successfully"
  unloadLib(lib)
```

---

## `loadLib(path, globalSymbols)`

```nim
proc loadLib*(path: string, globalSymbols = false): LibHandle {.gcsafe.}
```

### What it does

Opens the shared library at the given file system `path` and returns an opaque handle to it. Returns `nil` if the file does not exist, cannot be opened, or has unresolved dependencies.

**`path`** follows the same rules as the underlying OS loader:
- An absolute path (`/usr/lib/libfoo.so`) is used as-is.
- A relative path (`./myplugin.so`) is resolved relative to the current working directory.
- A bare name (`libfoo.so`) triggers the OS library search path (`LD_LIBRARY_PATH`, `PATH`, etc.).

**`globalSymbols`** (POSIX only, ignored on Windows):  
When `false` (the default), the library is loaded with `RTLD_NOW` — all symbols are resolved immediately but remain local to this handle (`RTLD_LOCAL`). When `true`, the library is loaded with `RTLD_NOW | RTLD_GLOBAL`, making its symbols visible to subsequently loaded libraries. This matters when a plugin expects certain symbols to be available globally (e.g., it was linked against a shared runtime that you want to reuse).

### When to use

Use this proc whenever you know the exact name or path of the library you want at the point of calling. For pattern-based or version-tolerant loading, see `loadLibPattern`.

### Examples

```nim
import dynlib

# Example 1 — absolute path, defensive check
let lib = loadLib("/usr/lib/x86_64-linux-gnu/libz.so.1")
assert lib != nil, "zlib not found at expected path"
unloadLib(lib)

# Example 2 — bare name, let the OS search
let lib2 = loadLib("libpng16.so.16")
if lib2 == nil:
  echo "libpng not found on this system"

# Example 3 — globalSymbols for a plugin that needs a shared runtime
let runtime = loadLib("myruntime.so", globalSymbols = true)
let plugin   = loadLib("plugin_a.so")   # can now see runtime's symbols
```

---

## `loadLib()` — executable self-handle

```nim
proc loadLib*(): LibHandle {.gcsafe.}
```

### What it does

Returns a handle to the *currently running executable* rather than to an external file. On POSIX this calls `dlopen(nil, RTLD_NOW)`; on Windows it calls `LoadLibraryA(nil)`.

This lets you look up symbols that were compiled into the executable itself — useful for implementing plugin dispatch tables, introspection, or calling functions by name that are already linked.

### When to use

- Introspection of your own binary's exported symbols.
- Implementing a scripting bridge where scripts call functions by name strings.
- Testing `symAddr` / `checkedSymAddr` without an external library.

### Example

```nim
import dynlib

# Look up a function compiled into the executable itself
let self = loadLib()
assert self != nil

# Suppose the executable exports `myRegisteredCallback`
let cb = self.symAddr("myRegisteredCallback")
if cb != nil:
  echo "Found own callback at address: ", cast[int](cb)

unloadLib(self)
```

> **Note:** For a symbol to be visible through the self-handle on Linux, the executable must be compiled with `-rdynamic` (exports all symbols to the dynamic symbol table) or the symbols must be explicitly marked for export.

---

## `unloadLib`

```nim
proc unloadLib*(lib: LibHandle) {.gcsafe.}
```

### What it does

Decrements the OS reference count for the loaded library. When the count reaches zero the library is unmapped from the process's address space. On POSIX this calls `dlclose`; on Windows, `FreeLibrary`.

### Important semantics

- **Reference counting.** `loadLib` on the same path increments a counter; `unloadLib` decrements it. The library is only physically unloaded when the count hits zero. Do not assume a single `unloadLib` immediately frees memory if the library was loaded multiple times.
- **Dangling pointers.** Any function pointers or data pointers obtained through `symAddr` become invalid after the library is unloaded. Using them is undefined behaviour.
- **Order matters.** If library B depends on library A, unload B before A.

### Example

```nim
import dynlib

type AddFn = proc(a, b: int): int {.cdecl.}

let lib = loadLib("libmath_ext.so")
assert lib != nil

let add = cast[AddFn](lib.symAddr("add_integers"))
echo add(3, 4)   # 7

unloadLib(lib)   # free resources
# echo add(1, 2) # ← DANGEROUS: lib is gone, `add` is a dangling pointer
```

---

## `symAddr`

```nim
proc symAddr*(lib: LibHandle, name: cstring): pointer {.gcsafe.}
```

### What it does

Searches the symbol table of the loaded library for a symbol named `name` and returns its address as a raw `pointer`. Returns `nil` if the symbol does not exist in the library.

The returned pointer can represent:
- A **function** — cast to a matching `proc` type and call it.
- A **global variable** — cast to a `ptr T` and dereference.

### When to use

Use `symAddr` when symbol absence is an acceptable, handled condition (optional feature detection, capability probing). For mandatory symbols, prefer `checkedSymAddr`.

### Example

```nim
import dynlib

type
  VersionFn = proc(): cstring {.cdecl.}

let lib = loadLib("libsomething.so")
assert lib != nil

# Optional feature: only use if present
let getVer = cast[VersionFn](lib.symAddr("get_version"))
if getVer != nil:
  echo "Library version: ", getVer()
else:
  echo "Version function not available in this build"

unloadLib(lib)
```

---

## `checkedSymAddr`

```nim
proc checkedSymAddr*(lib: LibHandle, name: cstring): pointer
```

### What it does

Exactly like `symAddr`, but instead of returning `nil` on failure it calls `raiseInvalidLibrary(name)`, which raises a `LibraryError` exception with a descriptive message.

This proc is the right choice when the symbol is *required* — if it's missing, the program cannot continue in a meaningful way. It removes the need for repetitive `if result == nil: raise ...` guards around every symbol lookup.

### When to use

Use `checkedSymAddr` for all mandatory symbols. Use `symAddr` only when you explicitly want to handle a missing symbol gracefully (optional features, version detection, compatibility shims).

### Example

```nim
import dynlib

type
  InitFn  = proc(): cint {.cdecl.}
  CleanFn = proc() {.cdecl.}

let lib = loadLib("libplugin.so")
assert lib != nil

# These symbols must exist — raise if they don't
let pluginInit  = cast[InitFn] (lib.checkedSymAddr("plugin_init"))
let pluginClean = cast[CleanFn](lib.checkedSymAddr("plugin_cleanup"))

discard pluginInit()
# ... use the plugin ...
pluginClean()

unloadLib(lib)
```

---

## `raiseInvalidLibrary`

```nim
proc raiseInvalidLibrary*(name: cstring) {.noinline, noreturn.}
```

### What it does

Raises a `LibraryError` exception with the message `"could not find symbol: <name>"`. The proc is marked `noreturn` — execution never continues past this call. It is marked `noinline` so the raising path does not bloat the hot call path of `checkedSymAddr`.

### When to use

You will rarely call this directly. Its primary purpose is to be the shared raising point for `checkedSymAddr`. However, you can call it in your own helper wrappers that load several symbols at once and want consistent error messages.

### Example

```nim
import dynlib

proc requireSym(lib: LibHandle, name: string): pointer =
  let p = lib.symAddr(name)
  if p == nil:
    raiseInvalidLibrary(name)   # ← consistent error, never returns
  result = p

let lib = loadLib("libfoo.so")
assert lib != nil
let fn = requireSym(lib, "foo_compute")  # raises LibraryError if absent
```

---

## `libCandidates`

```nim
proc libCandidates*(s: string, dest: var seq[string])
```

### What it does

Expands a library name *pattern* (the same mini-language used by the `dynlib` pragma) into a list of concrete candidate filenames and appends them to `dest`.

The pattern language has one construct: `(a|b|c)` is an *alternation group*. Everything outside the parentheses is a literal prefix or suffix. The proc recursively expands all alternatives, so nested groups work too.

| Pattern | Candidates produced |
|---|---|
| `libfoo.so` | `libfoo.so` |
| `libfoo(1\|2).so` | `libfoo1.so`, `libfoo2.so` |
| `lib(foo\|bar)(1\|2).so` | `libfoo1.so`, `libfoo2.so`, `libbar1.so`, `libbar2.so` |

### When to use

Call this when you need to enumerate candidate names before trying to load them, for example to display them in an error message, to pre-check their existence on disk, or to implement custom loading logic beyond what `loadLibPattern` provides.

### Example

```nim
import dynlib

var candidates: seq[string]
libCandidates("libssl(1.0|1.1|3).so", candidates)
echo candidates
# @["libssl1.0.so", "libssl1.1.so", "libssl3.so"]

for name in candidates:
  echo "Would try: ", name
```

---

## `loadLibPattern`

```nim
proc loadLibPattern*(pattern: string, globalSymbols = false): LibHandle
```

### What it does

Combines `libCandidates` and `loadLib` into one convenient call. It expands `pattern` into candidate names (using `libCandidates`) and tries to load each one in order, returning the first handle that succeeds. Returns `nil` if none of the candidates could be loaded.

This mirrors what the `dynlib` pragma does internally when you write something like `{.dynlib: "libssl(1.0|1.1|3).so".}`.

**`globalSymbols`** has the same meaning as in `loadLib`.

> **Warning (from the source):** This proc uses the GC (to build the candidates sequence) and therefore cannot be used to load the GC itself or called before the GC is initialised.

### When to use

Use `loadLibPattern` whenever the library may exist under several versioned names and you want to try them all without writing the loop yourself. It is particularly useful for system libraries whose SONAME varies across Linux distributions.

### Example

```nim
import dynlib

# Try libssl 3, then 1.1, then 1.0
let ssl = loadLibPattern("libssl(3|1.1|1.0).so")
if ssl == nil:
  quit("OpenSSL not found on this system")

# Try the Windows name first, then the Linux name
when defined(windows):
  let zlib = loadLibPattern("(zlib1|zlib).dll")
else:
  let zlib = loadLibPattern("libz.so(|.1|.1.2)")

assert zlib != nil, "zlib not found"
```

---

## Full worked example — runtime plugin system

```nim
# compile: nim c -r plugin_host.nim

import dynlib

# --- Type declarations matching the plugin ABI ---
type
  PluginInfo = object
    name: cstring
    version: cint
  GetInfoFn  = proc(): ptr PluginInfo {.cdecl.}
  ProcessFn  = proc(data: ptr UncheckedArray[byte], len: int): int {.cdecl.}

proc loadPlugin(path: string) =
  # 1. Open the library
  let lib = loadLib(path)
  if lib == nil:
    echo "Cannot load plugin: ", path
    return

  # 2. Resolve mandatory symbols
  let getInfo: GetInfoFn =
    cast[GetInfoFn](lib.checkedSymAddr("plugin_get_info"))
  let process: ProcessFn =
    cast[ProcessFn](lib.checkedSymAddr("plugin_process"))

  # 3. Resolve optional symbol
  type FlushFn = proc() {.cdecl.}
  let flush = cast[FlushFn](lib.symAddr("plugin_flush"))

  # 4. Use the plugin
  let info = getInfo()
  echo "Loaded plugin: ", info.name, " v", info.version

  var buf = [byte 0x01, 0x02, 0x03]
  let result = process(cast[ptr UncheckedArray[byte]](addr buf[0]), buf.len)
  echo "Plugin processed ", result, " bytes"

  if flush != nil:
    flush()
    echo "Plugin flushed"

  # 5. Clean up
  unloadLib(lib)

loadPlugin("./plugins/compressor.so")
```

---

## Error handling summary

| Situation | Recommended approach |
|---|---|
| Library file not found | Check `loadLib` / `loadLibPattern` returns `nil`; show a clear message and bail out |
| Required symbol missing | Use `checkedSymAddr`; handle `LibraryError` at the top level |
| Optional symbol missing | Use `symAddr`; skip the feature if `nil` |
| Using pointers after `unloadLib` | **Never do this** — undefined behaviour |

---

## Platform notes

- **`globalSymbols`** is silently ignored on Windows. All Windows DLLs share a single flat namespace by default.
- On **macOS**, library extensions are typically `.dylib`; on Linux, `.so` (often with a version suffix like `.so.6`).
- **Nintendo Switch** and **Genode** (without POSIX): all procs raise `OSError`. Dynamic loading is not supported on these targets.
- Symbol names in C libraries are usually unmangled (plain `add`), but C++ libraries may have mangled names (`_ZN3Foo3addEii`). Use `extern "C"` on the C++ side to avoid this.
