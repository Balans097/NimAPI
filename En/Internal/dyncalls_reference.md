# `dyncalls.nim` — Module Reference

> **Part of Nim's Runtime Library**  
> Internal runtime module — not imported directly by user code.  
> Inspired by Lua's dynamic library loading code.

---

## Overview

`dyncalls.nim` implements Nim's **dynamic library loading interface** — the runtime mechanism that allows Nim programs to load shared libraries (`.so`, `.dll`, `.dylib`) and call functions from them at runtime, without linking against them at compile time.

This is the foundation beneath Nim's `dynlib` pragma. When you write:

```nim
proc myFunc(x: int): int {.importc, dynlib: "mylib.so".}
```

…the compiler generates code that calls `nimLoadLibrary`, `nimGetProcAddr`, and eventually `nimUnloadLibrary` from this module.

### Platform dispatch

The module uses compile-time `when` branches to select the correct implementation for each target:

| Platform | Mechanism | C API used |
|---|---|---|
| POSIX (Linux, macOS, BSD, Solaris, AIX, HPUX…) | `dlfcn` interface | `dlopen`, `dlsym`, `dlclose`, `dlerror` |
| Windows / DOS | WinAPI | `LoadLibraryA`, `GetProcAddress`, `FreeLibrary` |
| Genode | Stub — raises `AssertionDefect` | — |
| Nintendo Switch / FreeRTOS / Zephyr / NuttX | Stub — writes to stderr and quits | — |
| Anything else | Compile error | — |

The three public-facing procedures (`nimLoadLibrary`, `nimUnloadLibrary`, `nimGetProcAddr`) have **exactly the same signature on every platform**. All platform differences are hidden inside them.

### Design constraints

The entire module is compiled with `{.push stack_trace: off.}`. This ensures that these low-level functions do not appear in Nim's stack traces — they operate below the level where tracing is meaningful, and tracing would require allocations that may not be safe here.

Two of the error-reporting procedures are also written to **avoid all heap allocation**. The reason is documented in the source comments: at the point where a library cannot be loaded or a symbol cannot be found, the program is in an undefined state, and calling the allocator could deadlock, corrupt state, or mask the real error. Output is done using `rawWrite` directly to `cstderr`.

---

## Constants

### `NilLibHandle`

```nim
const NilLibHandle: LibHandle = nil
```

A typed nil constant for `LibHandle`. Used as a sentinel value representing "no library loaded". Not exported, but referenced internally as the canonical "empty" handle value — a loaded library is always non-nil.

---

## Error Reporting Procedures

### `nimLoadLibraryError`

```nim
proc nimLoadLibraryError(path: string)
```

Called when `nimLoadLibrary` fails (returns nil). Writes a diagnostic message to `cstderr` and calls `rawQuit(1)` — the program terminates immediately and unconditionally.

The implementation is carefully crafted to **avoid all memory allocation**, because a library load failure may indicate a severely broken heap or environment. All string output uses `rawWrite` directly.

The message and behaviour differ by platform:

**POSIX without `-d:nimDebugDlOpen`:**
```
could not load: /path/to/lib.so
(compile with -d:nimDebugDlOpen for more information)
```

**POSIX with `-d:nimDebugDlOpen`:**  
The error string from `dlerror()` is written during the `nimLoadLibrary` call itself (before `nimLoadLibraryError` is called), so the user sees the full OS-level error reason.

**Windows — `ERROR_BAD_EXE_FORMAT`:**
```
could not load: mylib.dll
(bad format; library may be wrong architecture)
```
This error occurs when you try to load a 32-bit DLL from a 64-bit process (or vice versa) — a very common mistake.

**Windows GUI apps (`-d:guiapp`):**  
Because GUI applications have no visible console, the error is displayed as a Windows message box using `MessageBoxA`. The message is assembled into a fixed 1000-character stack buffer (again: no heap allocation) before being shown.

```nim
# Internally called like this:
let lib = nimLoadLibrary("mylib.so")
if lib == nil:
  nimLoadLibraryError("mylib.so")  # never returns
```

---

### `procAddrError`

```nim
proc procAddrError(name: cstring) {.compilerproc, nonReloadable, hcrInline.}
```

Called when `nimGetProcAddr` fails to find a named symbol in a loaded library. Writes a diagnostic and calls `rawQuit(1)`.

Output:
```
could not import: my_function_name
```

The pragmas carry specific meaning:
- **`{.compilerproc.}`** — the compiler knows this procedure by name and may call it directly from generated code, bypassing the normal dispatch.
- **`{.nonReloadable.}`** — this procedure must not be reloaded during hot-code-reload (HCR). Since it deals with the dynamic loading machinery itself, reloading it would be circular and dangerous.
- **`{.hcrInline.}`** — hint to the HCR system to inline this call, avoiding indirection through the reload table.

Like `nimLoadLibraryError`, this procedure is written without heap allocation.

```nim
# Internally called like this:
let fn = nimGetProcAddr(lib, "my_func")
# if fn is nil after all attempts, procAddrError("my_func") is called
```

---

## Core Dynamic Loading Procedures

These three procedures form the complete public interface of `dyncalls.nim`. They have uniform signatures across all platforms.

---

### `nimLoadLibrary`

```nim
proc nimLoadLibrary(path: string): LibHandle
```

Loads the shared library at `path` and returns an opaque handle to it. Returns `nil` on failure (the caller should then call `nimLoadLibraryError` with the path).

The handle is an opaque pointer (`LibHandle`) — its concrete type (`THINSTANCE` on Windows, a raw pointer on POSIX) is hidden from callers.

**POSIX implementation:**

Calls `dlopen(path, flags)` where `flags` is:
- `RTLD_NOW` by default — all symbols in the library are resolved immediately when the library is loaded, rather than lazily. This causes `dlopen` to fail immediately if any symbol is unresolvable, rather than failing later at the point of first call.
- `RTLD_NOW | RTLD_GLOBAL` when compiled with `-d:globalSymbols` — the library's symbols are also made available to subsequently loaded libraries. Useful for plugin systems where shared libraries need to call into each other.

With `-d:nimDebugDlOpen`, any `dlerror()` message is written to `cstderr` immediately, giving detailed OS-level diagnostics (e.g. missing dependencies, incompatible library versions).

On Linux and macOS, `RTLD_NOW` is a compile-time constant (value `2`); on other POSIX systems it is imported from `<dlfcn.h>` because the value may differ.

**Windows implementation:**

Calls `LoadLibraryA(path)`, the ANSI (8-bit) version of the Windows library loader. The `A` suffix means it accepts a `char*` path, not a wide-character `wchar_t*` path. The result is cast from `THINSTANCE` to `LibHandle`.

```nim
let lib = nimLoadLibrary("libssl.so.1.1")
if lib == nil:
  nimLoadLibraryError("libssl.so.1.1")
# lib is now a valid handle
```

---

### `nimUnloadLibrary`

```nim
proc nimUnloadLibrary(lib: LibHandle)
```

Unloads a previously loaded library, releasing OS resources. After this call, any function pointers obtained from `lib` via `nimGetProcAddr` become invalid and must not be called — doing so is undefined behaviour.

**POSIX implementation:** Calls `dlclose(lib)`. The OS decrements a reference count on the library; the library is only physically unloaded when the count reaches zero (other code may hold references to the same library).

**Windows implementation:** Calls `FreeLibrary(cast[THINSTANCE](lib))`. Same reference-counted semantics as POSIX.

**Genode:** Calls `raiseAssert` — dynamic unloading is not supported and will raise an `AssertionDefect`.

**Nintendo Switch / FreeRTOS / Zephyr / NuttX:** Writes an error to `cstderr` and calls `rawQuit(1)` — dynamic library operations are entirely unsupported on these platforms.

```nim
nimUnloadLibrary(lib)
# lib is now invalid — do not use any function pointers from it
```

---

### `nimGetProcAddr`

```nim
proc nimGetProcAddr(lib: LibHandle, name: cstring): ProcAddr
```

Looks up the exported symbol `name` in the loaded library `lib` and returns a function pointer to it. If the symbol cannot be found, calls `procAddrError(name)` which terminates the program — this procedure **never returns nil**.

`ProcAddr` is an opaque function pointer type. The caller is responsible for casting it to the correct function signature before calling it.

**POSIX implementation:**

Calls `dlsym(lib, name)`. On POSIX, if `dlsym` returns nil, the symbol is definitively not present and `procAddrError` is called immediately.

**Windows implementation — decorated name fallback:**

The Windows ABI for `__stdcall` functions (used by many Win32 APIs) decorates symbol names with a leading underscore and a trailing `@N` suffix indicating the total byte size of arguments (e.g. `_MyFunc@8` for a function taking 8 bytes of arguments). If the plain `GetProcAddress(lib, name)` call fails, the code systematically tries decorated names:

```
_name@0
_name@4
_name@8
_name@12
...
_name@200   (50 attempts, stepping by 4 bytes up to 200)
```

This brute-force search ensures that `nimGetProcAddr` works correctly for `__stdcall` functions even when the exact argument size is not known at the call site — a common situation when loading Windows system DLLs from Nim.

The decorated name is assembled into a fixed-size stack buffer of 250 characters (again: no heap allocation).

```nim
type MyFunc = proc(x: int): int {.cdecl.}

let fn = cast[MyFunc](nimGetProcAddr(lib, "my_function"))
let result = fn(42)
```

---

## Platform Stubs

### Genode

```nim
proc nimUnloadLibrary(lib: LibHandle) = raiseAssert("nimUnloadLibrary not implemented")
proc nimLoadLibrary(path: string): LibHandle = raiseAssert("nimLoadLibrary not implemented")
proc nimGetProcAddr(lib: LibHandle, name: cstring): ProcAddr = raiseAssert("nimGetProcAddr not implemented")
```

Genode is a capability-based microkernel OS with a strict resource model. Dynamic library loading is a privileged operation that does not fit the Genode programming model. Any attempt to use `dynlib` on Genode raises an `AssertionDefect` at runtime. Code that must run on Genode must link all libraries statically.

### Nintendo Switch / FreeRTOS / Zephyr / NuttX

These embedded/gaming platforms do not have filesystem-based shared library loaders. The stub implementations write an error message to stderr and exit:

```
nimLoadLibrary not implemented
```

Note that there is a typo in the original source: the error message for `nimGetProcAddr` reads `"nimGetProAddr not implemented"` (missing a 'c') — this is a known cosmetic bug in the source.

---

## Compile-Time Flags

| Flag | Effect |
|---|---|
| `-d:nimDebugDlOpen` | On POSIX: print `dlerror()` output to stderr after each `dlopen` call, even on success. Invaluable for diagnosing "could not load" failures caused by missing transitive dependencies. |
| `-d:globalSymbols` | On POSIX: open libraries with `RTLD_NOW \| RTLD_GLOBAL` instead of just `RTLD_NOW`. Makes the loaded library's symbols visible to all subsequently loaded libraries — needed for plugin systems that share code. |
| `-d:guiapp` | On Windows: display load errors as a `MessageBoxA` dialog instead of writing to stderr (which is invisible in GUI applications). |

---

## Internal POSIX C Imports

These are imported from `<dlfcn.h>` and used only within the POSIX branch. They are not exported.

| Function | Purpose |
|---|---|
| `dlopen(path, mode)` | Load a shared library; returns handle or nil |
| `dlsym(lib, name)` | Look up a symbol by name; returns function pointer or nil |
| `dlclose(lib)` | Decrement refcount and unload if zero |
| `dlerror()` | Return a human-readable string describing the last `dl*` error |

`RTLD_NOW` is either a compile-time constant (Linux/macOS: `2`) or imported from `<dlfcn.h>` on other POSIX platforms.

---

## Internal Windows C Imports

These are imported from `<windows.h>` and used only within the Windows branch. They are not exported.

| Function | Purpose |
|---|---|
| `LoadLibraryA(path)` | Load a DLL by path; returns `HINSTANCE` or nil |
| `GetProcAddress(lib, name)` | Look up an exported symbol; returns function pointer or nil |
| `FreeLibrary(lib)` | Decrement refcount and unload DLL if zero |
| `GetLastError()` | Retrieve the last Win32 error code |
| `MessageBoxA(hwnd, text, title, flags)` | Display a modal message box (GUI apps only) |

`THINSTANCE` wraps Windows' `HINSTANCE` type. In C++ mode it is a struct with a `pointer` field (matching the C++ definition); in C mode it is simply a `pointer`.

---

## Summary Table

| Symbol | Kind | Exported | Purpose |
|---|---|---|---|
| `NilLibHandle` | const | No | Typed nil sentinel for LibHandle |
| `nimLoadLibraryError` | proc | No | Report library load failure and exit |
| `procAddrError` | proc (compilerproc) | No | Report symbol lookup failure and exit |
| `nimLoadLibrary` | proc | No* | Load a shared library by path |
| `nimUnloadLibrary` | proc | No* | Unload a previously loaded library |
| `nimGetProcAddr` | proc | No* | Look up a symbol in a loaded library |

\* These procedures are not exported with `*` but are referenced by name from the compiler's code generator — they form part of the runtime ABI.
