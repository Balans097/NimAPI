# Nim `alloc.nim` — Internal Allocator Reference

> **Audience notice.** `alloc.nim` is Nim's *internal* low-level memory allocator. It is **not** a public API — it is part of the runtime library and is compiled directly into every Nim program. The functions described here are called transparently by `system.alloc`, `system.dealloc`, and the garbage collector. You would interact with this module directly only when working on the Nim runtime itself, a custom GC, or an embedded port without an OS allocator.

---

## Architecture Overview

The allocator manages memory in page-aligned blocks called **chunks**. Two categories of chunk exist:

- **Small chunks** — exactly one OS page (`PageSize`, typically 4 KB). Divided internally into fixed-size *cells* of the same size class. Reusing freed cells avoids page-level OS calls for the common case.
- **Big chunks** — one or more pages. Managed with a **TLSF** (Two-Level Segregated Fit) free-list matrix for near-O(1) best-fit allocation.
- **Huge chunks** — bigger than `MaxBigChunkSize`. Allocated and freed directly from the OS on every call.

All allocations live inside a `MemRegion` — a self-contained heap. Thread-local regions share nothing with each other except through an explicit shared-heap path protected by a lock.

### Three-path small allocation

```
alloc
 └─► rawAlloc
       ├─ Path 1: No free chunk for this size class → request a new page from OS
       ├─ Path 2: Free chunk exists, accumulator has room → bump acc pointer
       └─ Path 3: Free chunk has a free-list cell → pop cell from free list  ← preferred
```

### Two-path small deallocation

```
dealloc
 └─► rawDealloc
       ├─ We own this chunk → add cell to free list, maybe activate chunk
       └─ Different thread owns this chunk → enqueue in sharedFreeLists (ARC/ORC)
                                          → or mark in AVL tree (legacy GC)
```

---

## Key Constants

| Constant | Meaning |
|---|---|
| `nimMinHeapPages` | Minimum number of pages to request from OS at once (default 128 = 0.5 MB). Overridable at compile time with `-d:nimMinHeapPages=N`. |
| `SmallChunkSize` | Equal to `PageSize` (typically 4096 bytes). The maximum size of a small chunk. |
| `MaxFli` | Maximum first-level index in the TLSF matrix (30 on 64-bit, 14 on 16-bit). |
| `MaxLog2Sli` | Maximum log₂ of the second-level index count (5, giving 32 sub-buckets). |
| `MaxBigChunkSize` | Largest chunk size that fits in the TLSF matrix. Chunks larger than this become *huge* chunks. |
| `HugeChunkSize` | `MaxBigChunkSize + 1` — the threshold above which chunks are huge. |
| `nimMaxHeap` | Optional compile-time ceiling on total heap size in MB (`-d:nimMaxHeap=N`). Zero means unlimited. |

---

## Key Types

### `MemRegion`

The root data structure representing one entire heap. On a single-threaded program there is one `MemRegion`. With threads, each thread has its own region; a shared heap region is used for `allocShared`.

Important fields:

| Field | Purpose |
|---|---|
| `freeSmallChunks[s]` | Array of free small-chunk lists, one entry per size class (index = `size / MemAlign`). The head is the *active* chunk for that class. |
| `sharedFreeLists[s]` | *(ARC/ORC only)* Cross-thread free cells waiting to be reclaimed by the owning thread. |
| `flBitmap` / `slBitmap` | TLSF bitmaps: indicate which size-class buckets in the matrix are non-empty. Enable O(1) best-fit lookup via bit-scan. |
| `matrix` | TLSF free-list matrix `[RealFli][MaxSli]` of big chunk doubly-linked lists. |
| `currMem` / `maxMem` / `freeMem` / `occ` | Accounting: total OS memory, peak OS memory, free OS memory, currently occupied bytes. |
| `chunkStarts` | `IntSet` (a compact bitset) of all page indices that start a chunk. Used to look up which chunk a pointer belongs to. |
| `heapLinks` | Linked list of `(chunk, size)` pairs — the complete OS-level allocation log. Used during `deallocOsPages` to return everything to the OS. |
| `llmem` | Low-level bump allocator for the allocator's own internal data structures (AVL nodes, trunk nodes, heap links). Lifetime = lifetime of the `MemRegion`. |

---

### `SmallChunk`

One OS page subdivided into cells of a single size class.

| Field | Purpose |
|---|---|
| `size` | Cell size for this chunk (all cells are identical in size). |
| `acc` | Accumulator — byte offset of the next uninitialized cell from `data`. Advances monotonically until the page is full. |
| `free` | Total bytes this chunk can still provide (via both free list and accumulator). When `free < size`, the chunk is removed from `freeSmallChunks`. |
| `freeList` | Singly-linked list of recycled cells. Cells may belong to this chunk or be *foreign* (donated from another chunk or thread). |
| `foreignCells` | Count of free cells in `freeList` that originate from a different chunk. A chunk with `foreignCells > 0` must not be freed: doing so would orphan those cells. |
| `owner` | Pointer to the `MemRegion` that created this chunk. Used to detect cross-thread deallocations. |
| `data` | Aligned start of the usable memory region within the page. |

---

### `BigChunk`

A page-aligned block of one or more pages, used for allocations too large to fit in a small chunk. Also serves as the base for huge chunks.

| Field | Purpose |
|---|---|
| `size` | Total byte size of this chunk including the `BigChunk` header. |
| `prevSize` | Size of the *preceding* adjacent chunk in the virtual address space; the least-significant bit encodes whether *this* chunk is currently *in use*. This dual-purpose encoding drives the TLSF coalescing logic. |
| `prev` / `next` | Free-list doubly-linked list pointers. **While the chunk is allocated**, `prev` is overwritten with the aligned data pointer (the value returned to the user) — this allows the GC and deallocation path to find the user pointer from the raw chunk pointer. |
| `owner` | Owning `MemRegion`. |

---

### `FreeCell`

A recycled small-chunk cell that is available for reuse. The `FreeCell` header is **overlaid** on top of whatever data used to occupy the cell, so it occupies zero extra space.

| Field | Purpose |
|---|---|
| `next` | Next cell in the free list. |
| `zeroField` *(non-destructors)* | `0` = free cell, `1` = manually managed pointer, otherwise points to `PNimType`. Used by the legacy GC to determine whether a cell is live. |
| `alignment` *(ARC/ORC)* | Padding to keep cell layout compatible. |

---

### `LLChunk` (Low-Level Chunk)

A bump-pointer arena used *only* for the allocator's own housekeeping structures (AVL nodes, trunk nodes, heap link records). Each `LLChunk` is exactly one page. It is never returned to the OS during normal operation; `llDeallocAll` returns them all at shutdown.

---

### `IntSet` / `Trunk`

A compact bitset mapping page indices to boolean membership. Implemented as a hash table of `Trunk` nodes, where each `Trunk` covers a 256-bit window of addresses. Used for `chunkStarts` — the set of page addresses that start a chunk — enabling O(1) "does pointer `p` point into a valid chunk?" queries.

---

### `AvlNode` *(non-ARC/ORC only)*

Nodes in an AVL tree (`root`) that maps big-chunk data addresses to their size ranges. The GC uses this tree for interior-pointer scanning — given an arbitrary pointer into the middle of an allocation, find the base of that allocation.

---

### `HeapLinks`

A linked list of arrays, each entry recording `(PBigChunk, size)` — every region of memory obtained from the OS. Used by `deallocOsPages` to return all OS memory at program exit or region teardown.

---

## Core Internal Functions

### `rawAlloc`
```
proc rawAlloc(a: var MemRegion, requestedSize: int, alignment: int = 0): pointer
```

The central allocation workhorse. All public `alloc`/`alloc0` calls pass through here.

**What it does:**

1. Rounds `requestedSize` up to a multiple of `MemAlign`.
2. **Small path** (`size ≤ SmallChunkSize - overhead`, no custom alignment): looks up the active chunk in `a.freeSmallChunks[s]`. If a chunk exists and has a free-list cell, pops from the free list. If no free-list cell, bumps the accumulator. If no active chunk at all, allocates a fresh OS page via `getSmallChunk` and sets it as the active chunk.
3. **Big/huge path**: processes any deferred cross-thread frees (`sharedFreeListBigChunks`), then calls `getBigChunk` or `getHugeChunk`. Computes an alignment pad if custom alignment was requested, stores the aligned data pointer in `c.prev` for later recovery by the GC and dealloc.
4. In non-ARC/ORC builds, inserts the allocation range into the AVL tree (`a.root`) for interior-pointer scanning.
5. Updates `a.occ`.

**Returns** a raw pointer to usable memory. The caller is responsible for any header bookkeeping.

---

### `rawAlloc0`
```
proc rawAlloc0(a: var MemRegion, requestedSize: int): pointer
```

Thin wrapper around `rawAlloc` that zero-initialises the returned memory with `zeroMem`. Used as the implementation of `alloc0`.

---

### `rawDealloc`
```
proc rawDealloc(a: var MemRegion, p: pointer)
```

The central deallocation workhorse. Determines whether `p` is in a small or big chunk, then takes the appropriate path.

**Small chunk path:**

- If `c.owner == addr(a)` (we own it): decrements `a.occ`, optionally overwrites with `0xFF` (debug mode), and adds the cell to the *active* chunk's free list (the chunk currently at the head of `freeSmallChunks[s]`). If the cell belongs to a different chunk than the active one, it is *lent* as a foreign cell. If the current chunk was previously exhausted, it is re-activated.
- If the chunk belongs to another thread (ARC/ORC): adds the cell to `a.sharedFreeLists[s]` for deferred processing.

**Big chunk path:**

- If `c.owner == addr(a)`: calls `deallocBigChunk`, which removes the chunk from the AVL tree, then either calls `freeBigChunk` (TLSF coalescing + re-insertion into the matrix) or `freeHugeChunk` (direct OS dealloc).
- If the chunk belongs to another thread (ARC/ORC): pushes it onto `a.sharedFreeListBigChunks`.

---

### `alloc`
```
proc alloc(allocator: var MemRegion, size: Natural): pointer
```

Public-facing allocation for the legacy GC build. Calls `rawAlloc(size + sizeof(FreeCell))`, stamps `zeroField = 1` on the returned `FreeCell` header to mark the cell as live, then returns the pointer advanced past the header. The header is invisible to the caller.

In ARC/ORC builds, this is a direct pass-through to `rawAlloc` with no header.

---

### `alloc0`
```
proc alloc0(allocator: var MemRegion, size: Natural): pointer
```

Like `alloc` but zero-fills the returned memory.

---

### `dealloc`
```
proc dealloc(allocator: var MemRegion, p: pointer)
```

Public-facing deallocation. In legacy GC mode, verifies the `FreeCell` header, rewinds `p` by `sizeof(FreeCell)`, and calls `rawDealloc`. In ARC/ORC mode, calls `rawDealloc` directly.

---

### `realloc`
```
proc realloc(allocator: var MemRegion, p: pointer, newsize: Natural): pointer
```

Reallocates the block at `p` to `newsize` bytes. Always allocates a new block, copies `min(old_size, newsize)` bytes, then frees the old block. There is no in-place resize optimisation. If `newsize == 0` and `p != nil`, this is equivalent to `dealloc`. If `p == nil`, this is equivalent to `alloc`.

---

### `realloc0`
```
proc realloc0(allocator: var MemRegion, p: pointer, oldsize, newsize: Natural): pointer
```

Like `realloc`, but if `newsize > oldsize`, the extra bytes (`[oldsize..newsize)`) are explicitly zero-filled.

---

### `getBigChunk`
```
proc getBigChunk(a: var MemRegion, size: int): PBigChunk
```

Finds or creates a big chunk of at least `size` bytes using the TLSF algorithm:

1. Calls `mappingSearch(size, fl, sl)` to round `size` up to the next TLSF bucket boundary (page-aligned).
2. Calls `findSuitableBlock(a, fl, sl)` to scan the TLSF bitmaps for a free block at least as large.
3. If no block found: calls `requestOsChunks` to obtain fresh memory from the OS. Adapts the request size dynamically — starts at 4 pages, doubles up to `MaxBigChunkSize` based on current heap occupancy.
4. If a block was found and is larger than needed: calls `splitChunk` to split it, returning the remainder to the matrix.
5. Marks the chunk as "in use" (sets the used bit in `prevSize`).

---

### `freeBigChunk`
```
proc freeBigChunk(a: var MemRegion, c: PBigChunk)
```

Returns a big chunk to the TLSF free pool. Implements **boundary-tag coalescing**:

1. Clears the used bit in `c.prevSize`.
2. If the *left* neighbour is free and under `MaxBigChunkSize`: removes it from the matrix and merges it with `c`.
3. If the *right* neighbour is free and under `MaxBigChunkSize`: removes it from the matrix and merges it with `c`.
4. Calls `addChunkToMatrix` to re-insert the (possibly enlarged) chunk.

---

### `getHugeChunk` / `freeHugeChunk`
```
proc getHugeChunk(a: var MemRegion; size: int): PBigChunk
proc freeHugeChunk(a: var MemRegion; c: PBigChunk)
```

For allocations above `MaxBigChunkSize`. Each call goes directly to the OS (`allocPages` / `osDeallocPages`). There is no pooling or coalescing.

---

### `getSmallChunk`
```
proc getSmallChunk(a: var MemRegion): PSmallChunk
```

Obtains a fresh `SmallChunk` by allocating exactly one page via `getBigChunk(a, PageSize)`. The page is then reinterpreted as a `SmallChunk` — no data is moved.

---

### `requestOsChunks`
```
proc requestOsChunks(a: var MemRegion, size: int): PBigChunk
```

Requests memory from the OS via `osAllocPages` / `osTryAllocPages`. Implements an adaptive growth strategy: starts conservatively at 4 pages (16 KB), then doubles the request on each call up to `MaxBigChunkSize`, scaled by current heap occupancy. If the OS cannot satisfy the larger request, falls back to the exact `size`. Records the allocation in `a.heapLinks` for later teardown.

---

### `splitChunk` / `splitChunk2`
```
proc splitChunk(a: var MemRegion, c: PBigChunk, size: int)
proc splitChunk2(a: var MemRegion, c: PBigChunk, size: int): PBigChunk
```

Carves a prefix of `size` bytes from chunk `c`, leaving the remainder as a new independent chunk. `splitChunk2` returns the remainder without inserting it into the matrix; `splitChunk` also calls `addChunkToMatrix` on the remainder.

The boundary-tag `prevSize` field of the next-adjacent chunk is updated to reflect the new layout.

---

### `deallocBigChunk`
```
proc deallocBigChunk(a: var MemRegion, c: PBigChunk)
```

Cleans up bookkeeping before freeing a big chunk:

- Decrements `a.occ`.
- In non-ARC/ORC builds, removes the chunk's address range from the AVL tree.
- Resets `c.prev` (which was overwritten with the aligned data pointer while the chunk was live).
- Dispatches to `freeHugeChunk` or `freeBigChunk` based on chunk size.

---

### `deallocOsPages`
```
proc deallocOsPages(a: var MemRegion)
```

Returns **all** OS-level pages held by the region to the OS. Called at program exit or when tearing down a thread's heap. Iterates the `heapLinks` list and calls `osDeallocPages` on each recorded allocation, then calls `llDeallocAll` to free the internal bump-allocator pages.

---

### `llAlloc`
```
proc llAlloc(a: var MemRegion, size: int): pointer
```

A private bump-pointer allocator for the allocator's own metadata (AVL nodes, trunk nodes, heap link records). Allocates from `a.llmem`; if `a.llmem` is exhausted, obtains a fresh OS page. Memory allocated here is **never individually freed** — it is all released at once by `llDeallocAll`.

---

### `llDeallocAll`
```
proc llDeallocAll(a: var MemRegion)
```

Walks the `llmem` chain and calls `osDeallocPages` on every page used for internal metadata. Called at region teardown.

---

### `compensateCounters` *(ARC/ORC only)*
```
proc compensateCounters(a: var MemRegion; c: PSmallChunk; size: int)
```

When cells from `sharedFreeLists` are absorbed into a chunk (`fetchSharedCells`), this function adjusts the chunk's `free` counter and `a.occ` to account for the reclaimed capacity. Also increments `c.foreignCells` for each cell that originated from a different chunk.

---

### `freeDeferredObjects` *(ARC/ORC only)*
```
proc freeDeferredObjects(a: var MemRegion; root: PBigChunk)
```

Processes the `sharedFreeListBigChunks` queue, calling `deallocBigChunk` for each entry. Bounded to `MaxSteps = 20` iterations per call to prevent unbounded latency spikes; any remainder is re-enqueued.

---

### `addToSharedFreeList` / `addToSharedFreeListBigChunks` *(ARC/ORC only)*
```
proc addToSharedFreeList(c: PSmallChunk; f: ptr FreeCell; size: int)
proc addToSharedFreeListBigChunks(a: var MemRegion; c: PBigChunk)
```

Lock-free atomic prepend operations that deposit a freed cell or big chunk onto the owning thread's deferred-free queue. Uses compare-and-swap when thread support is enabled.

---

## TLSF Bitmap Operations

The following small functions implement the O(1) free-list lookup at the heart of TLSF.

### `msbit` / `lsbit`
```
proc msbit(x: uint32): int
proc lsbit(x: uint32): int
```

Find the position of the most-significant and least-significant set bit in a 32-bit word, respectively. `msbit` uses a 256-entry lookup table (`fsLookupTable`) to handle one byte at a time. `lsbit` is derived from `msbit` via `x & (-x)`.

Used to map a requested size to a TLSF matrix index (`fl`, `sl`) and to find the lowest non-empty bucket.

---

### `mappingSearch`
```
proc mappingSearch(r, fl, sl: var int)
```

Given a requested size `r`, rounds it **up** to the nearest TLSF bucket boundary (page-aligned), then computes the first-level index `fl` and second-level index `sl` into the matrix. This "search" rounding ensures that any block found in the resulting bucket is guaranteed to be at least as large as the original request.

---

### `mappingInsert`
```
proc mappingInsert(r: int): tuple[fl, sl: int]
```

Given an exact chunk size `r`, computes the `fl`/`sl` coordinates for *inserting* into the matrix without any rounding. The chunk goes into the bucket that exactly matches its size.

---

### `findSuitableBlock`
```
proc findSuitableBlock(a: MemRegion; fl, sl: var int): PBigChunk
```

Scans the TLSF bitmap to find the **smallest free block** that is at least as large as the requested size. First checks whether the exact `(fl, sl)` bucket has anything; if not, scans to the next higher non-empty bucket via bit-scan operations on `slBitmap` and `flBitmap`. Updates `fl` and `sl` to point to the found bucket.

---

### `addChunkToMatrix` / `removeChunkFromMatrix`
```
proc addChunkToMatrix(a: var MemRegion; b: PBigChunk)
proc removeChunkFromMatrix(a: var MemRegion; b: PBigChunk)
proc removeChunkFromMatrix2(a: var MemRegion; b: PBigChunk; fl, sl: int)
```

Manage doubly-linked lists of free big chunks within the TLSF matrix. `addChunkToMatrix` prepends a chunk to the head of the list for its `(fl, sl)` bucket and sets the corresponding bits in `slBitmap`/`flBitmap`. `removeChunkFromMatrix` splices a chunk out from anywhere in the list, clearing the bitmap bits if the list becomes empty.

---

## IntSet Operations (Chunk Tracking)

### `intSetGet` / `intSetPut`
```
proc intSetGet(t: IntSet, key: int): PTrunk
proc intSetPut(a: var MemRegion, t: var IntSet, key: int): PTrunk
```

Hash-table lookup and insert for the `IntSet` bitset. Keys are page indices divided by `TrunkShift` — each `Trunk` covers `IntsPerTrunk * IntSize` pages. `intSetPut` allocates new `Trunk` nodes from `llAlloc` when a hash bucket is first used.

---

### `contains` / `incl` / `excl`
```
proc contains(s: IntSet, key: int): bool
proc incl(a: var MemRegion, s: var IntSet, key: int)
proc excl(s: var IntSet, key: int)
```

Bit-level get/set/clear on the `IntSet`. Each operation first locates the relevant `Trunk` node (via hash), then computes a bit index within the trunk's `bits` array.

---

## Memory Statistics Functions

### `getFreeMem` / `getTotalMem` / `getOccupiedMem`
```
proc getFreeMem(a: MemRegion): int
proc getTotalMem(a: MemRegion): int
proc getOccupiedMem(a: MemRegion): int
```

Direct accessors for `MemRegion` accounting fields:

- **`getFreeMem`** — `a.freeMem`: bytes of OS-owned memory not currently held in any allocated chunk (free big chunks sitting in the TLSF matrix).
- **`getTotalMem`** — `a.currMem`: total bytes obtained from the OS.
- **`getOccupiedMem`** — `a.occ`: bytes that have been handed out to callers and not yet freed.

---

### `getMaxMem`
```
proc getMaxMem(a: var MemRegion): int
```

Returns the **peak** OS-level memory usage since the region was created: `max(a.currMem, a.maxMem)`. `maxMem` is updated whenever `currMem` decreases (pages are returned to the OS), so this gives the true watermark.

---

## Per-Thread Instantiation Template

### `instantiateForRegion`
```
template instantiateForRegion(allocator: untyped)
```

A `dirty` template that binds the entire public allocator interface to a specific `MemRegion` variable. Generates these free-standing procedures for the calling module:

| Generated proc | Delegates to |
|---|---|
| `allocImpl(size)` | `alloc(allocator, size)` |
| `alloc0Impl(size)` | `alloc0(allocator, size)` |
| `deallocImpl(p)` | `dealloc(allocator, p)` |
| `reallocImpl(p, newSize)` | `realloc(allocator, p, newSize)` |
| `realloc0Impl(p, oldSize, newSize)` | `realloc0(allocator, p, newSize)` + zero-fill |
| `allocSharedImpl(size)` | Shared heap alloc (lock-guarded on threads) |
| `allocShared0Impl(size)` | Zero-filling shared heap alloc |
| `deallocSharedImpl(p)` | Shared heap dealloc |
| `reallocSharedImpl(p, newSize)` | Shared heap realloc |
| `reallocShared0Impl(p, oldSize, newSize)` | Zero-filling shared heap realloc |
| `getFreeMem()` | `allocator.freeMem` |
| `getTotalMem()` | `allocator.currMem` |
| `getOccupiedMem()` | `allocator.occ` |
| `getMaxMem*()` | `getMaxMem(allocator)` |

In a threaded build without ARC/ORC, `allocShared*` and `deallocShared*` acquire/release `heapLock` around a separate `sharedHeap: MemRegion`.

---

## Pointer Inspection Utilities *(non-ARC/ORC only)*

### `isAllocatedPtr`
```
proc isAllocatedPtr(a: MemRegion, p: pointer): bool
```

Returns `true` if `p` points to an *allocated and live* cell. Checks that `p` is accessible (its page index is in `chunkStarts`), the chunk is not unused, and the `FreeCell.zeroField > 1` (indicating an active GC-managed object). Used by the GC for sanity assertions.

---

### `interiorAllocatedPtr`
```
proc interiorAllocatedPtr(a: MemRegion, p: pointer): pointer
```

Given a pointer `p` that may point anywhere *inside* an allocation (not just to its base), returns the **base pointer** of that allocation, or `nil` if `p` is not inside any live allocation.

For small chunks: computes the cell base from the chunk's cell size and the offset of `p` within the page.

For big chunks: uses `c.prev` (which stores the aligned data pointer while the chunk is in use).

For pointers outside any known chunk: consults the AVL tree (`a.root`) via `inRange` — the tree maps address ranges to their base pointers, enabling O(log n) interior-pointer lookup.

---

### `prepareForInteriorPointerChecking`
```
proc prepareForInteriorPointerChecking(a: var MemRegion)
```

Caches `lowGauge(a.root)` and `highGauge(a.root)` into `a.minLargeObj` and `a.maxLargeObj` before a GC cycle. This range check filters out 99.96%+ of interior-pointer queries without needing to descend the AVL tree.

---

### `isAccessible`
```
proc isAccessible(a: MemRegion, p: pointer): bool
```

Returns `true` if the page containing pointer `p` is tracked in `a.chunkStarts`, meaning `p` falls within memory managed by this region. This is the first filter applied before any deeper pointer checks.

---

## Chunk List Helpers

### `listAdd` / `listRemove`
```
proc listAdd[T](head: var T, c: T)
proc listRemove[T](head: var T, c: T)
```

Generic intrusive doubly-linked list operations. `listAdd` prepends `c` to `head`; `listRemove` splices `c` out. Both include debug assertions (via `sysAssert`) that verify list consistency.

---

### `updatePrevSize`
```
proc updatePrevSize(a: var MemRegion, c: PBigChunk, prevSize: int)
```

After a chunk is resized or split, updates the `prevSize` field of the *immediately following* chunk in the virtual address space. This is the boundary-tag mechanism — it allows the free-chunk coalescing code in `freeBigChunk` to find and merge adjacent free chunks without maintaining a separate data structure.

---

## GC Object Iteration

### `allObjects` *(iterator)*
```
iterator allObjects(m: var MemRegion): pointer
```

Yields a raw pointer to every currently-allocated object in the region. Iterates `m.chunkStarts`, then for each active chunk:

- **Small chunk**: walks from `data` to `data + acc` in steps of `c.size`, yielding each cell address.
- **Big chunk**: yields `c.prev` (the aligned data pointer stored there during allocation).

Sets `m.locked = true` for the duration to prevent modifications. Used by GC mark phases.

---

### `iterToProc`
```
proc iterToProc*(iter: typed, envType: typedesc; procName: untyped)
```

Compiler magic (`{.magic: "Plugin", compileTime.}`) that converts a closure iterator into a callback-style procedure. Used internally to adapt `allObjects` into the form expected by the GC's C interface.

---

## Auxiliary Chunk Queries

### `ptrSize`
```
proc ptrSize(p: pointer): int
```

Returns the usable byte size of the allocation at `p`. For small chunks this is `c.size - sizeof(FreeCell)` (legacy GC) or `c.size` (ARC/ORC). For big chunks it subtracts `bigChunkOverhead()`.

This is what `realloc` uses to determine how many bytes to copy from the old allocation.

---

### `isSmallChunk`
```
proc isSmallChunk(c: PChunk): bool
```

Returns `true` when `c.size ≤ SmallChunkSize - smallChunkOverhead()`. The threshold is exactly one page minus the `SmallChunk` struct header.

---

### `chunkUnused`
```
proc chunkUnused(c: PChunk): bool
```

Returns `true` when bit 0 of `c.prevSize` is 0, meaning the chunk is free (not currently allocated). The dual use of `prevSize` for both the previous-chunk size and the in-use flag is a classic boundary-tag trick.

---

### `pageIndex` / `pageAddr`
```
proc pageIndex(c: PChunk): int
proc pageIndex(p: pointer): int
proc pageAddr(p: pointer): PChunk
```

Convert between pointer values and page indices (shift right by `PageShift`). `pageAddr` rounds a pointer *down* to its page boundary, giving the `PChunk` header for any pointer within that page. These are used constantly throughout the allocator to move between "a user pointer" and "the chunk it lives in".

---

## Thread Safety Summary

| Operation | Thread safety |
|---|---|
| Allocate from thread-local `MemRegion` | Safe — no locks needed |
| Free to a thread-local `MemRegion` | Safe — lock-free fast path when `c.owner == addr(a)` |
| Free to a *foreign* `MemRegion` (ARC/ORC) | Lock-free atomic push to `sharedFreeLists` / `sharedFreeListBigChunks` |
| Free to a *foreign* `MemRegion` (legacy GC) | Not supported by the small-chunk path; big-chunk path is supported |
| `allocShared` / `deallocShared` (no ARC/ORC) | Lock-guarded via `heapLock` |
| `allocShared` / `deallocShared` (ARC/ORC) | Delegates to thread-local path (no shared heap distinction) |
| `allObjects` iteration | Locks the region (`m.locked = true`); must not allocate/free during traversal |

---

*This document covers `alloc.nim` as found in the Nim runtime library. Internal-only helpers, debug-only code paths, and platform-specific `#ifdef`-equivalent `when` branches have been included only where they are architecturally significant.*
