# gc_regions Module Reference

> **Module:** `gc_regions`  
> **Part of:** Nim's Runtime Library  
> **Purpose:** Implements a "Stack GC" — a region-based, bump-pointer memory allocator designed for embedded systems, real-time applications, or any scenario where deterministic, ultra-low-latency memory management is required. Instead of tracing objects and collecting garbage, memory is arranged in chunks and freed all at once when a *region* is discarded.

---

## Core Concept

This module replaces Nim's usual garbage collector with a **region allocator** (also called an *arena* or *obstack*). The mental model is a stack of memory:

```
 ┌──────────────────────────────────────────┐
 │  Chunk 1  │  Chunk 2  │  ... │  Chunk N  │  ← linked list
 └──────────────────────────────────────────┘
                                     ▲
                               bump pointer
                          (next free byte lives here)
```

- Allocation is **O(1)**: just advance the bump pointer.
- Deallocation of individual objects is **optional** and has limited coalescing.
- Freeing the entire region is **O(chunks)**: walk the list and return pages to the OS.
- Regions are **thread-local** by default, but can be moved between threads.

---

## Table of Contents

1. [Types](#types)
2. [Thread-Local Region API](#thread-local-region-api)
3. [Per-Region API](#per-region-api)
4. [Memory Statistics](#memory-statistics)
5. [GC Compatibility Stubs](#gc-compatibility-stubs)
6. [Internal / Compiler-RT Procedures](#internal--compiler-rt-procedures)

---

## Types

### `MemRegion`

```nim
type MemRegion* = object
```

**What it is:** The central data structure of this module. A `MemRegion` holds an entire arena of memory: a linked list of OS-allocated chunks, the current bump pointer, and size accounting. You create one with a simple `var r: MemRegion` (zero-init is a valid empty region).

**Key fields (internal):**
- `bump` / `remaining` — where the next allocation goes and how many bytes are left in the current chunk.
- `head` / `tail` — first and last chunk in the linked list.
- `nextChunkSize` — the size to request for the next chunk (doubles each time, bounded below by `PageSize * nimMinHeapPages`).
- `totalSize` — sum of all chunk sizes currently held.
- `lock` — present only when `hasThreadSupport` is defined; guards cross-thread access.

```nim
var arena: MemRegion     # start with an empty region
mkDir(".")               # just an illustration; allocations go into tlRegion
```

---

### `StackPtr`

```nim
type StackPtr = object
  bump: pointer
  remaining: int
  current: Chunk
```

**What it is:** A lightweight snapshot of a region's allocation state — essentially a *saved bump-pointer position*. Obtained via `obstackPtr()` and restored via `setObstackPtr()`. This is how you implement scoped, stack-like memory lifetimes inside a region.

---

## Thread-Local Region API

The thread-local variable `tlRegion` is the *implicit* region used by the Nim runtime for all normal `new` and `newSeq` calls when the region GC is active. These free procedures operate on `tlRegion` without requiring you to pass a region explicitly.

---

### `obstackPtr` (thread-local)

```nim
proc obstackPtr*(): StackPtr
```

**What it is:** Takes a snapshot of the current allocation position in the thread-local region. Call this before a block of temporary work; then call `setObstackPtr` afterwards to free everything allocated in between — in O(1) per chunk freed.

This is the region-allocator equivalent of "save the stack pointer".

```nim
let saved = obstackPtr()
# ... allocate lots of temporary objects ...
setObstackPtr(saved)   # all those objects are gone; memory is reclaimed
```

---

### `setObstackPtr` (thread-local)

```nim
proc setObstackPtr*(sp: StackPtr)
```

**What it is:** Restores the thread-local region to the position captured in `sp`, freeing all memory allocated after that point. Any objects with finalizers that were allocated after `sp` will have their finalizers called before the memory is reclaimed.

```nim
let mark = obstackPtr()
let tmp = newString(1024)   # allocated in tlRegion
# ... use tmp ...
setObstackPtr(mark)         # tmp is gone; its memory is back in the pool
```

> **Important:** After calling `setObstackPtr`, any pointers to objects that were allocated after `sp` become dangling. Do not use them.

---

### `deallocAll` (thread-local)

```nim
proc deallocAll*()
```

**What it is:** Frees **every** chunk in the thread-local region and resets it to empty. Finalizers are called on all objects that have them. The region itself is zeroed out and ready to be reused.

```nim
# After this, tlRegion is empty; all memory returned to OS
deallocAll()
```

---

### `withScratchRegion`

```nim
template withScratchRegion*(body: untyped)
```

**What it is:** A template that swaps in a **fresh, empty region** as the thread-local region for the duration of `body`, then discards all memory allocated during `body` and restores the previous region. Think of it as a scoped scratch pad — perfect for temporary work that must not pollute the main region.

```nim
withScratchRegion:
  let temp = newSeq[int](10_000)
  doSomeWork(temp)
  # temp and everything else allocated here is automatically freed
# original tlRegion is fully restored; no leaks, no lingering temporaries
```

> **This is the highest-level and safest API in the module.** Prefer it over manual `obstackPtr`/`setObstackPtr` pairs whenever the scope of temporary allocations matches a block of code.

---

## Per-Region API

These procedures take an explicit `MemRegion` parameter, letting you manage multiple independent arenas simultaneously.

---

### `withRegion`

```nim
template withRegion*(r: var MemRegion; body: untyped)
```

**What it is:** Temporarily makes `r` the active thread-local region for the duration of `body`. All Nim runtime allocations (`new`, `newSeq`, string operations) that happen inside `body` go into `r`. On exit — whether normal or via exception — the previous thread-local region is restored and `r` is updated to reflect any allocations that occurred.

```nim
var myArena: MemRegion
withRegion(myArena):
  let obj = new(MyType)   # allocated in myArena, not in tlRegion
  doWork(obj)
# myArena now owns obj; tlRegion is unchanged
```

---

### `obstackPtr` (per-region)

```nim
proc obstackPtr*(r: MemRegion): StackPtr
```

**What it is:** Same as the thread-local version, but for an explicit region `r`. Returns a `StackPtr` snapshot of `r`'s current allocation state.

```nim
var arena: MemRegion
let mark = arena.obstackPtr()
```

---

### `setObstackPtr` (per-region)

```nim
proc setObstackPtr*(r: var MemRegion; sp: StackPtr)
```

**What it is:** Rolls back region `r` to the allocation state captured in `sp`, freeing and finalizing everything allocated after that point.

```nim
var arena: MemRegion
let mark = arena.obstackPtr()
withRegion(arena):
  let tmp = new(HeavyObject)
arena.setObstackPtr(mark)  # HeavyObject freed, finalizer called if any
```

---

### `deallocAll` (per-region)

```nim
proc deallocAll*(r: var MemRegion)
```

**What it is:** Frees all memory in region `r`, calls all pending finalizers, and resets `r` to the empty state. After this call, `r` is as if it were freshly declared.

```nim
var scratch: MemRegion
withRegion(scratch):
  # ... lots of work ...
scratch.deallocAll()   # wipe the slate clean
```

---

### `isOnHeap`

```nim
proc isOnHeap*(r: MemRegion; p: pointer): bool
```

**What it is:** Returns `true` if the pointer `p` points to memory that lives inside region `r`. Useful for safety checks and debugging — e.g., to verify that a pointer you received actually belongs to the arena you think it does.

The implementation checks the tail chunk first (fastest path, since it holds the active bump pointer), then walks the rest of the chunk list.

```nim
var arena: MemRegion
withRegion(arena):
  let obj = new(MyType)
  assert arena.isOnHeap(cast[pointer](obj))
```

---

### `getOccupiedMem` (per-region)

```nim
proc getOccupiedMem*(r: MemRegion): int
```

**What it is:** Returns the number of bytes currently **in use** in region `r`. Computed as `totalSize - remaining` — i.e., all allocated chunk space minus the unused tail of the last chunk.

```nim
echo "Used: ", myArena.getOccupiedMem(), " bytes"
```

---

### `getFreeMem` (per-region)

```nim
proc getFreeMem*(r: MemRegion): int
```

**What it is:** Returns the number of bytes still **available** in the current (tail) chunk before a new chunk will be requested from the OS. This is just `r.remaining` — it does **not** include the space that would be available in a newly allocated chunk.

```nim
echo "Free in current chunk: ", myArena.getFreeMem(), " bytes"
```

---

### `getTotalMem` (per-region)

```nim
proc getTotalMem*(r: MemRegion): int
```

**What it is:** Returns the total number of bytes across **all** chunks currently held by region `r`, whether those bytes are occupied or not.

```nim
echo "Total arena size: ", myArena.getTotalMem(), " bytes"
```

---

## Memory Statistics

These are the thread-local counterparts of the per-region statistics functions above. They query `tlRegion`.

### `getOccupiedMem`

```nim
proc getOccupiedMem(): int
```

**What it is:** Bytes in use in the thread-local region.

```nim
echo "TL region occupied: ", getOccupiedMem()
```

---

### `getFreeMem`

```nim
proc getFreeMem(): int
```

**What it is:** Bytes remaining in the current chunk of the thread-local region.

---

### `getTotalMem`

```nim
proc getTotalMem(): int
```

**What it is:** Total bytes held by the thread-local region (occupied + free in tail chunk).

```nim
let ratio = getOccupiedMem().float / getTotalMem().float
echo "Utilization: ", ratio * 100.0, "%"
```

---

## GC Compatibility Stubs

Because this module replaces the standard GC, it must provide no-op implementations of all GC control procedures that other Nim code might call. These are all safe to call but do nothing.

| Procedure | Signature | Description |
|---|---|---|
| `GC_disable` | `proc GC_disable()` | No-op. There is no GC to disable. |
| `GC_enable` | `proc GC_enable()` | No-op. |
| `GC_fullCollect` | `proc GC_fullCollect()` | No-op. No collection cycle exists. |
| `GC_setStrategy` | `proc GC_setStrategy(strategy: GC_Strategy)` | No-op. |
| `GC_enableMarkAndSweep` | `proc GC_enableMarkAndSweep()` | No-op. |
| `GC_disableMarkAndSweep` | `proc GC_disableMarkAndSweep()` | No-op. |
| `GC_getStatistics` | `proc GC_getStatistics(): string` | Always returns `""`. |
| `nimGC_setStackBottom` | `proc nimGC_setStackBottom(p: pointer)` | No-op. |
| `nimGCref` | `proc nimGCref(x: pointer)` | No-op. Reference counting stub. |
| `nimGCunref` | `proc nimGCunref(x: pointer)` | No-op. Reference counting stub. |

```nim
# Safe to call; does nothing in region mode
GC_fullCollect()
GC_disable()
```

---

## Internal / Compiler-RT Procedures

These procedures are called by Nim-generated code or the compiler internals. You will almost never call them directly from user code, but understanding them helps when reading runtime diagnostics or writing very low-level code.

---

### `newObj`

```nim
proc newObj(typ: PNimType, size: int): pointer {.compilerRtl.}
```

**What it is:** The compiler calls this to allocate a new non-sequence, non-string object. Allocates `size` bytes in `tlRegion`, zeroes the memory, and registers the finalizer if the type has one. Cannot be used for sequences or strings.

---

### `newObjNoInit`

```nim
proc newObjNoInit(typ: PNimType, size: int): pointer {.compilerRtl.}
```

**What it is:** Like `newObj` but **skips the `zeroMem` call**. Marginally faster when the caller is going to initialize every field anyway. Used by the compiler when it can prove that all fields will be written before being read.

---

### `newSeq`

```nim
proc newSeq(typ: PNimType, len: int): pointer {.compilerRtl.}
```

**What it is:** Allocates a new sequence of `len` elements in `tlRegion`. The allocated block is zeroed. Both `len` and `reserved` are set to `len`. Each sequence carries a `SeqHeader` that holds a pointer back to its owning region, enabling it to grow in-place via `growObj`.

---

### `newStr`

```nim
proc newStr(typ: PNimType, len: int; init: bool): pointer {.compilerRtl.}
```

**What it is:** Allocates a new string buffer of `len` bytes in `tlRegion`. If `init` is `true`, the buffer is zeroed. The string length is set to `0` and reserved capacity to `len`. Like sequences, strings hold a back-pointer to their region.

---

### `growObj`

```nim
proc growObj(old: pointer, newsize: int): pointer {.rtl.}
```

**What it is:** Called by the compiler when a sequence or string needs to grow beyond its current capacity. Reads the `SeqHeader` of the old object to find its owning region, allocates a larger block in that same region, copies the old data, zeroes the new tail, and deallocates the old block (using bump-pointer rollback if possible).

The key insight: because the `SeqHeader` records the original region, a sequence can grow even if `tlRegion` has changed since the sequence was created.

---

### `unsureAsgnRef` / `asgnRef` / `asgnRefNoCycle`

```nim
proc unsureAsgnRef(dest: PPointer, src: pointer) {.compilerproc, inline.}
proc asgnRef(dest: PPointer, src: pointer) {.compilerproc, inline.}
proc asgnRefNoCycle(dest: PPointer, src: pointer) {.compilerproc, inline, deprecated.}
```

**What they are:** Assignment helpers called by the compiler whenever a reference is stored into a pointer slot. In a tracing GC these would update write barriers or reference counts. In region mode they are trivial `dest[] = src` assignments — there is nothing to track.

`asgnRefNoCycle` is deprecated and exists only for backward compatibility with older compiled code.

---

### `allocImpl` / `alloc0Impl` / `reallocImpl` / `realloc0Impl` / `deallocImpl`

```nim
proc allocImpl(size: Natural): pointer
proc alloc0Impl(size: Natural): pointer
proc reallocImpl(p: pointer, newsize: Natural): pointer
proc realloc0Impl(p: pointer, oldsize, newsize: Natural): pointer
proc deallocImpl(p: pointer)
```

**What they are:** The low-level implementations behind `alloc`, `alloc0`, `realloc`, `realloc0`, and `dealloc` in the system module. These bypass the region entirely and call `c_malloc` / `c_realloc` / `c_free` directly. They raise `OutOfMemError` on allocation failure.

> **Note:** These are for *system-level* allocations (e.g., channels, mutexes) that must live outside any region. Prefer `withRegion` + normal `new` / `newSeq` for application objects.

---

### `allocSharedImpl` / `allocShared0Impl` / `reallocSharedImpl` / `reallocShared0Impl` / `deallocSharedImpl`

```nim
proc allocSharedImpl(size: Natural): pointer
proc allocShared0Impl(size: Natural): pointer
proc reallocSharedImpl(p: pointer, newsize: Natural): pointer
proc reallocShared0Impl(p: pointer, oldsize, newsize: Natural): pointer
proc deallocSharedImpl(p: pointer)
```

**What they are:** Implementations for the `allocShared` / `deallocShared` family. In this allocator they are identical to their non-shared counterparts — there is no shared heap; all "shared" memory also goes through `c_malloc`. Thread safety must be ensured by the caller.

---

*This reference covers all exported and compiler-facing symbols from `gc_regions.nim` in Nim's runtime library.*
