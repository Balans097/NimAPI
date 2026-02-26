# `osalloc.nim` — Module Reference

> **Part of Nim's Runtime Library**  
> Author: Andreas Rumpf (© 2016)  
> Compiled with `{.push raises: [], gcsafe.}`

---

## Overview

`osalloc.nim` is a **low-level, platform-specific memory allocation module** that sits at the very foundation of Nim's runtime memory manager. Its sole purpose is to ask the **operating system** for large, page-aligned blocks of raw memory, and to hand them back when they are no longer needed.

It does not manage individual objects. It does not track allocation metadata. It simply bridges the gap between Nim's higher-level allocator and the OS-specific mechanisms (`mmap`, `VirtualAlloc`, `malloc`, bump-pointer, etc.).

The module selects its implementation at **compile time** via `when` branches, targeting:

| Platform / Flag | Mechanism |
|---|---|
| `-d:nimAllocPagesViaMalloc` (ARC/ORC only) | `c_malloc` + manual page alignment |
| Emscripten (WASM) | `mmap` with embedded metadata block |
| POSIX (Linux, macOS, BSD, Solaris, Haiku…) | `mmap` with `MAP_ANONYMOUS | MAP_PRIVATE` |
| Windows | `VirtualAlloc` / `VirtualFree` |
| Standalone / bare-metal | Static array bump-pointer allocator |

All three public procedures share the same interface regardless of platform — that is the key design goal of this module.

---

## Internal Helper: `roundup`

```nim
proc roundup(x, v: int): int {.inline.}
```

**Not exported**, but foundational. Rounds integer `x` **up** to the nearest multiple of `v`, where `v` must be a power of two. Used internally to align allocation sizes to page boundaries.

```nim
roundup(14, 4096)  # → 4096  (one full page)
roundup(15, 8)     # → 16
roundup(65, 8)     # → 72
```

The implementation uses a classic bitmask trick: `(x + (v-1)) and not (v-1)`. This is both branchless and extremely fast.

---

## Compile-Time Constant: `doNotUnmap`

```nim
const doNotUnmap = not (defined(amd64) or defined(i386)) or
                   defined(windows) or defined(nimAllocNoUnmap)
```

This constant controls whether `osDeallocPages` actually releases memory back to the OS. It is `true` (do **not** unmap) when:

- The architecture is **not** x86/amd64 (some platforms like Linux/PowerPC exhibit broken `munmap` behaviour where unmapping one page frees the entire block).
- The OS is **Windows** (because `MEM_RELEASE` requires size `0` and has special semantics).
- The flag `-d:nimAllocNoUnmap` is explicitly set by the developer.

When `doNotUnmap` is `true`, `osDeallocPages` becomes a no-op for safety.

---

## Exported Procedures

The module exports exactly **three** procedures. They form a minimal, clean OS memory interface.

---

### `osAllocPages`

```nim
proc osAllocPages(size: int): pointer {.inline.}
```

#### What it does

Allocates at least `size` bytes of **raw, page-aligned memory** from the operating system. The memory is readable and writable. This procedure **never returns `nil`** — if the allocation fails for any reason (out of memory, resource limits, etc.), it calls `raiseOutOfMem()`, which raises an `OutOfMemDefect` and terminates the program.

#### When to use it

Use this when you **must have the memory or die**. It is the primary allocation path in Nim's memory manager. If your code cannot reasonably continue without the allocation succeeding, prefer this over `osTryAllocPages`.

#### Platform details

| Platform | Underlying call |
|---|---|
| malloc-backed | `c_malloc(size + PageSize - 1 + 4)` + manual alignment |
| Emscripten | `mmap` with extra space for metadata header |
| POSIX | `mmap(nil, size, PROT_READ\|PROT_WRITE, MAP_ANON\|MAP_PRIVATE, -1, 0)` |
| Windows | `VirtualAlloc(nil, size, MEM_RESERVE\|MEM_COMMIT, PAGE_READWRITE)` |
| Standalone | Bump pointer into a static `float64` array |

#### Example

```nim
# Allocate exactly one OS page of raw memory
let pageSize = 4096
let mem = osAllocPages(pageSize)
# mem is now a page-aligned pointer to 4096 writable bytes
# If the system is out of memory, the program will have terminated
# before reaching this line.
defer: osDeallocPages(mem, pageSize)
```

---

### `osTryAllocPages`

```nim
proc osTryAllocPages(size: int): pointer {.inline.}
```

#### What it does

Like `osAllocPages`, but with **soft failure semantics**: if the OS cannot provide the requested memory, this procedure returns `nil` instead of aborting. The caller is responsible for checking the return value.

#### When to use it

Use this when your code has a **fallback plan** if memory is unavailable — for example, flushing a cache, compacting existing data, waiting and retrying, or reporting a handled error to the user. This is the "polite" allocation request.

#### Platform details

| Platform | Failure behaviour |
|---|---|
| malloc-backed | Same as `osAllocPages` — `c_malloc` failure raises immediately (no soft fail on this path) |
| Emscripten | Delegates to `osAllocPages` directly |
| POSIX | `mmap` returns `MAP_FAILED` (`-1`), which is converted to `nil` |
| Windows | `VirtualAlloc` returns `nil` on failure, which is passed through |
| Standalone | Returns `nil` if the static heap is exhausted |

> **Note:** On the malloc-backed path (`-d:nimAllocPagesViaMalloc`), `osTryAllocPages` behaves identically to `osAllocPages` — a failed `c_malloc` raises immediately. This is a known limitation of that path.

#### Example

```nim
proc tryGrowHeap(size: int): bool =
  let mem = osTryAllocPages(size)
  if mem == nil:
    # The OS refused. We can try to compact or report the situation.
    echo "Warning: could not grow heap by ", size, " bytes."
    return false
  # Got the memory — register it with our allocator
  registerChunk(mem, size)
  return true
```

---

### `osDeallocPages`

```nim
proc osDeallocPages(p: pointer, size: int) {.inline.}
```

#### What it does

Returns a block of memory previously obtained from `osAllocPages` or `osTryAllocPages` back to the operating system. The pointer `p` must be the **exact pointer** returned by the allocation call, and `size` must match what was originally requested.

Passing the wrong pointer or wrong size leads to **undefined behaviour** — this is a raw OS-level call with no bookkeeping.

#### The `doNotUnmap` guard

On platforms where `doNotUnmap` is `true`, this procedure is a **complete no-op** — it does nothing and the memory is never returned. This is intentional and safe: it avoids platform bugs (Linux/PowerPC) and Windows `MEM_RELEASE` limitations. The memory leak is acceptable in these cases because Nim's allocator reuses freed pages internally before asking the OS for more.

#### Platform details

| Platform | Mechanism |
|---|---|
| malloc-backed | Reads 4-byte alignment offset stored just before `p`, calls `c_free(p - offset)` |
| Emscripten | Reads metadata block stored just before `p`, calls `munmap(realPointer, realSize)` |
| POSIX | `munmap(p, size)` if `reallyOsDealloc` is true |
| Windows | `VirtualFree(p, 0, MEM_RELEASE)` if `reallyOsDealloc` is true |
| Standalone | Decrements bump pointer if `p` was the last allocation (simple stack-like dealloc) |

#### Example

```nim
let pageSize = 4096
let mem = osAllocPages(pageSize)

# ... use the memory ...

# Return it to the OS (or no-op if doNotUnmap is true for this platform)
osDeallocPages(mem, pageSize)
# After this call, `mem` is a dangling pointer — do not use it.
```

---

## Standalone / Bare-Metal Allocator

When `hostOS == "standalone"` or `-d:StandaloneHeapSize` is defined, all three procedures operate on a **statically allocated array** of `float64` values (chosen for natural 8-byte alignment). The heap size defaults to `1024 * PageSize` but can be overridden:

```nim
# Compile with:
# nim c -d:StandaloneHeapSize=65536 myprogram.nim
```

The allocator is a **bump-pointer allocator**: it simply advances a pointer forward on each allocation. Deallocation only works if the freed block is the **most recently allocated** one (LIFO order). This makes it extremely fast and suitable for embedded systems where heap fragmentation is not a concern and the OS is absent.

---

## Design Principles

1. **Zero overhead on the fast path.** All procedures are `{.inline.}`, so call overhead disappears entirely.
2. **No metadata in the allocated region.** The memory returned is clean. The malloc-backed path stores a 4-byte offset *before* the user pointer; Emscripten stores a small descriptor before the pointer — but these are invisible to the caller.
3. **Crash-early by default.** `osAllocPages` treats out-of-memory as a fatal condition. Memory exhaustion at the OS level almost always means something has gone seriously wrong.
4. **Platform safety over memory efficiency.** The `doNotUnmap` mechanism sacrifices memory return to the OS in exchange for correctness on platforms with broken unmap semantics.

---

## Summary Table

| Procedure | Returns `nil` on failure? | Aborts on failure? | Notes |
|---|---|---|---|
| `osAllocPages` | No | Yes (`raiseOutOfMem`) | Primary allocation path |
| `osTryAllocPages` | Yes (mostly) | No (mostly) | Soft allocation, check result |
| `osDeallocPages` | — | Yes (Windows, if `virtualFree` fails) | May be no-op depending on platform |
