# system/alloc.nim — Module Reference (Public API)

> **Import:** no separate `import` is needed. `alloc.nim` is pulled into
> `system.nim` via an `include` directive, so its procedures become part of
> the `system` module and are available in any Nim program without an
> explicit import.
> **Scope:** the low-level heap allocator that backs both manual memory
> management (`alloc`/`dealloc`) and the garbage collector.

This file implements Nim's runtime-level memory allocator: a TLSF-like
scheme for "big" blocks and an accumulator/free-cell scheme for "small"
blocks inside page-sized chunks. What follows documents only the
**publicly used layer** — the procedures ordinary Nim code calls
(`alloc`, `dealloc`, `realloc`, `allocShared`, `getFreeMem`, and so on).
The internal machinery (the TLSF `flBitmap`/`slBitmap` matrix, the AVL
tree used for interval lookups, the `IntSet` of chunk starts, and the
`MemRegion` structure itself) is not documented procedure-by-procedure —
it gets only a summary overview in section VI.

General convention of the module: every operation has a "per-thread /
shared" pair (`alloc` works on the current thread's heap, `allocShared`
works on a heap reachable from any thread), and a `0` suffix means
"zero the memory" (`alloc0`, `realloc0`, `allocShared0`).

---

## Table of Contents

I. [Memory model and the `MemRegion` type](#i-memory-model-and-the-memregion-type)
&nbsp;&nbsp;1. [`MemRegion`](#1-memregion)
&nbsp;&nbsp;2. [How chunks are organized (brief)](#2-how-chunks-are-organized-brief)

II. [Allocating and freeing memory on the current thread](#ii-allocating-and-freeing-memory-on-the-current-thread)
&nbsp;&nbsp;1. [`alloc`](#1-alloc)
&nbsp;&nbsp;2. [`alloc0`](#2-alloc0)
&nbsp;&nbsp;3. [`dealloc`](#3-dealloc)
&nbsp;&nbsp;4. [`realloc`](#4-realloc)
&nbsp;&nbsp;5. [`realloc0`](#5-realloc0)

III. [Shared memory across threads](#iii-shared-memory-across-threads)
&nbsp;&nbsp;1. [`allocShared`](#1-allocshared)
&nbsp;&nbsp;2. [`allocShared0`](#2-allocshared0)
&nbsp;&nbsp;3. [`deallocShared`](#3-deallocshared)
&nbsp;&nbsp;4. [`reallocShared` / `reallocShared0`](#4-reallocshared--reallocshared0)

IV. [Memory statistics and diagnostics](#iv-memory-statistics-and-diagnostics)
&nbsp;&nbsp;1. [`getFreeMem`](#1-getfreemem)
&nbsp;&nbsp;2. [`getTotalMem`](#2-gettotalmem)
&nbsp;&nbsp;3. [`getOccupiedMem`](#3-getoccupiedmem)
&nbsp;&nbsp;4. [`getMaxMem`](#4-getmaxmem)
&nbsp;&nbsp;5. [`getFreeSharedMem` / `getTotalSharedMem` / `getOccupiedSharedMem`](#5-getfreesharedmem--gettotalsharedmem--getoccupiedsharedmem)
&nbsp;&nbsp;6. [`getMemCounters`](#6-getmemcounters)
&nbsp;&nbsp;7. [`ptrSize`](#7-ptrsize)

V. [Returning pages to the operating system](#v-returning-pages-to-the-operating-system)
&nbsp;&nbsp;1. [`deallocOsPages`](#1-deallocospages)

VI. [How it works under the hood (summary overview)](#vi-how-it-works-under-the-hood-summary-overview)

VII. [Practical recipes](#vii-practical-recipes)
&nbsp;&nbsp;1. [A fixed-size object pool](#1-a-fixed-size-object-pool)
&nbsp;&nbsp;2. [A growable buffer (like `seq`) via `realloc`](#2-a-growable-buffer-like-seq-via-realloc)
&nbsp;&nbsp;3. [Passing a buffer between threads via `allocShared`](#3-passing-a-buffer-between-threads-via-allocshared)
&nbsp;&nbsp;4. [Monitoring memory usage](#4-monitoring-memory-usage)

VIII. [Quick reference table](#viii-quick-reference-table)

IX. [Summary: which procedure to use](#ix-summary-which-procedure-to-use)

---

## I. Memory model and the `MemRegion` type

### 1. `MemRegion`

```nim
type MemRegion = object
```

**What it does.** `MemRegion` is a single "heap": a self-contained data
structure with its own chunks, TLSF bitmaps, an AVL tree of big blocks,
and lists of free small cells. In an ordinary program one `MemRegion` is
created implicitly per thread (the per-thread heap), and another one
serves as the shared heap for `allocShared`. The type is not exported and
is never constructed directly by user code: all work with it goes through
the procedures in sections II–V, which are already bound to the right
instance.

- **Parameters**: none (the type is only ever used as an implicit
  receiver inside internal procedures; it never surfaces to the caller).

---

### 2. How chunks are organized (brief)

A region's memory is divided into **chunks**, each starting at an
address that is a multiple of the page size:

- **Small chunk** (`SmallChunk`, one page in size) is cut into cells of
  a single size class; cells are reused through a linked free-cell list
  plus a shared "accumulator" (a pointer that advances into the still
  untouched part of the chunk).
- **Big chunk** (`BigChunk`) backs requests that don't fit in a small
  chunk; such chunks live in a TLSF matrix indexed by size class and can
  merge (coalesce) with their neighbors on free.
- **Huge chunk** (larger than `HugeChunkSize`) is allocated and freed
  directly through the operating system, bypassing the matrix entirely.

The public procedures below decide on their own which of the three paths
to take, based on the request size — calling code never has to worry
about this.

---

## II. Allocating and freeing memory on the current thread

### 1. `alloc`

```nim
proc alloc(size: Natural): pointer {.gcsafe.}
```

**What it does.** Allocates `size` bytes of untyped memory on the current
thread's heap and returns a pointer to it. The contents are **not**
guaranteed to be zeroed — use `alloc0` if that matters. If `size` is 0,
the behavior follows from the internal size rounding (a minimal-sized
block is allocated rather than `nil`).

- **Implementation notes.** For requests that fit in a small chunk, the
  memory is almost free to hand out: either the chunk's "accumulator" is
  advanced, or the head of the free-cell list is popped — both are O(1).
  For larger requests, a suitable block is looked up in the TLSF matrix
  (also amortized O(1), thanks to the bitmaps), requesting new pages from
  the OS if needed.

- **Parameters**:
  - `size: Natural` — the requested size in bytes (an immutable input).

- **Examples**:

```nim
# Typical case: allocate memory for an object and work through casting.
type
  Vec3 = object
    x, y, z: float32

var p = cast[ptr Vec3](alloc(sizeof(Vec3)))
p.x = 1.0
p.y = 2.0
p.z = 3.0
echo p.x, " ", p.y, " ", p.z  # prints 1.0 2.0 3.0
dealloc(p)                    # every alloc must be matched by a dealloc

# Boundary case: zero size — no exception is raised,
# but the returned pointer must not be used to store data.
var empty = alloc(0)
dealloc(empty)
```

---

### 2. `alloc0`

```nim
proc alloc0(size: Natural): pointer
```

**What it does.** Same as `alloc`, but additionally zeroes all `size`
bytes before returning the pointer. Useful wherever a predictable initial
memory state matters (for example, buffers that will later be
interpreted as an array of numbers).

- **Implementation notes.** The implementation is a thin wrapper: `alloc`
  is called first, then `zeroMem` sweeps the entire range. The extra cost
  is linear in size and only noticeable for large buffers.

- **Parameters**:
  - `size: Natural` — the requested size in bytes.

- **Examples**:

```nim
# Typical case: an int array guaranteed to be zeroed.
var arr = cast[ptr UncheckedArray[int]](alloc0(5 * sizeof(int)))
echo arr[0], " ", arr[4]  # prints 0 0
dealloc(arr)

# Practical scenario: a counter buffer that will only ever be incremented.
var counters = cast[ptr UncheckedArray[int32]](alloc0(256 * sizeof(int32)))
inc(counters[10])
inc(counters[10])
echo counters[10]  # prints 2 — the starting value was guaranteed to be 0
dealloc(counters)
```

---

### 3. `dealloc`

```nim
proc dealloc(p: pointer)
```

**What it does.** Frees memory previously obtained via `alloc` or
`alloc0` **on the same thread's heap**. After the call, the pointer `p`
must not be used — accessing already-freed memory is undefined behavior,
not a raised Nim exception. Calling `dealloc` twice on the same pointer
(a double free) is likewise undefined behavior and cannot be safely
demonstrated via `doAssertRaises`.

- **Implementation notes.** Depending on whether the pointer belongs to a
  small or a big chunk, the cell is either returned to the chunk's
  free-cell list (and the chunk is not freed on the spot once exhausted —
  there is a dedicated guard against a race between threads sharing free
  cells), or the chunk is handed back to the TLSF matrix and may merge
  with neighboring free chunks.

- **Parameters**:
  - `p: pointer` — a pointer previously returned by `alloc`/`alloc0`
    (logically mutated — it becomes invalid).

- **Examples**:

```nim
# Typical case.
var p = alloc(64)
dealloc(p)

# Practical scenario: freeing at the end of scope via defer.
proc useTemporaryBuffer() =
  var buf = alloc(1024)
  defer: dealloc(buf)
  # ... work with buf ...
  discard
useTemporaryBuffer()
```

---

### 4. `realloc`

```nim
proc realloc(p: pointer, newSize: Natural): pointer
```

**What it does.** Resizes a previously allocated block to `newSize`
bytes, preserving contents up to `min(old_size, newSize)` bytes, and
returns a pointer to the (possibly new) block. If `p` is `nil`, it
behaves like `alloc(newSize)`. If `newSize` is 0, the block is freed and
`nil` is returned.

- **Implementation notes.** Unlike C's classic `realloc`, there is no
  attempt to grow the block in place: the implementation always
  allocates a fresh block of the required size, copies the data via
  `copyMem`, and frees the old one. This is simpler and safer with
  respect to fragmentation, at the cost of extra copying on frequent
  growth — hence the practical recommendation to grow buffers with
  headroom (see section VII, recipe 2).

- **Parameters**:
  - `p: pointer` — a pointer to an existing block, or `nil`.
  - `newSize: Natural` — the new requested size in bytes.

- **Examples**:

```nim
# Typical case: growing a buffer.
var buf = cast[ptr UncheckedArray[int32]](alloc(4 * sizeof(int32)))
buf[0] = 1
buf[1] = 2
buf[2] = 3
buf[3] = 4
buf = cast[ptr UncheckedArray[int32]](realloc(buf, 8 * sizeof(int32)))
echo buf[0], " ", buf[3]  # prints 1 4 — old data was preserved
dealloc(buf)

# Boundary case: newSize == 0 is equivalent to dealloc.
var p = alloc(16)
p = realloc(p, 0)
echo p == nil  # prints true
```

---

### 5. `realloc0`

```nim
proc realloc0(p: pointer, oldSize, newSize: Natural): pointer
```

**What it does.** Same as `realloc`, but additionally zeroes the "tail" —
the bytes between `oldSize` and `newSize`, if the new size is larger than
the old one. Unlike `realloc`, it requires the old size `oldSize` to be
passed explicitly, since it's exactly what the zeroing boundary is
measured from.

- **Implementation notes.** A wrapper around `realloc`: after copying the
  old data, `zeroMem` clears only the "new" part of the buffer
  (`newSize - oldSize` bytes), leaving already-copied data untouched.

- **Parameters**:
  - `p: pointer` — a pointer to an existing block, or `nil`.
  - `oldSize: Natural` — the block's previous size in bytes (needed only
    to compute the zeroing boundary).
  - `newSize: Natural` — the new requested size in bytes.

- **Examples**:

```nim
# Typical case: extending a buffer with a guaranteed-zero new tail.
var buf = cast[ptr UncheckedArray[byte]](alloc0(4))
buf[0] = 1
buf = cast[ptr UncheckedArray[byte]](realloc0(buf, 4, 8))
echo buf[0], " ", buf[4], " ", buf[7]  # prints 1 0 0
dealloc(buf)
```

---

## III. Shared memory across threads

### 1. `allocShared`

```nim
proc allocShared(size: Natural): pointer {.gcsafe.}
```

**What it does.** Allocates `size` bytes on a heap shared by every thread
in the program. Memory obtained here can be handed off to another thread
and safely freed there via `deallocShared` — unlike memory from `alloc`,
which is tied to its owning thread.

- **Implementation notes.** When thread support is present, access to the
  shared heap is guarded by a lock (`heapLock`); without thread support
  it's simply an alias for regular `alloc`. In the destructor-based GC
  configuration, lock-free deferred-free lists are used instead of a
  lock — freeing "someone else's" memory doesn't block the thread, it
  defers the actual release until the owner's next allocator call
  (details in section VI).

- **Parameters**:
  - `size: Natural` — the requested size in bytes.

- **Examples**:

```nim
# Typical case: a buffer that will be freed from a different thread.
var shared = allocShared(128)
deallocShared(shared)
```

---

### 2. `allocShared0`

```nim
proc allocShared0(size: Natural): pointer
```

**What it does.** The zeroing variant of `allocShared` — analogous to the
`alloc`/`alloc0` pair.

- **Parameters**:
  - `size: Natural` — the requested size in bytes.

- **Examples**:

```nim
var shared = cast[ptr UncheckedArray[int]](allocShared0(4 * sizeof(int)))
echo shared[0]  # prints 0
deallocShared(shared)
```

---

### 3. `deallocShared`

```nim
proc deallocShared(p: pointer)
```

**What it does.** Frees a block allocated via `allocShared`/
`allocShared0`. Unlike regular `dealloc`, it's safe to call from a thread
other than the one that allocated the memory — that's exactly the
practical point of the shared heap.

- **Implementation notes.** If the freeing thread is the same as the
  chunk's owner, the release happens immediately; otherwise the block is
  placed on the owner's deferred-free list and actually released later,
  which avoids holding a lock for the duration of the free operation
  itself.

- **Parameters**:
  - `p: pointer` — a pointer previously returned by `allocShared`/
    `allocShared0`.

- **Examples**:

```nim
var shared = allocShared(64)
deallocShared(shared)
```

---

### 4. `reallocShared` / `reallocShared0`

```nim
proc reallocShared(p: pointer, newSize: Natural): pointer
proc reallocShared0(p: pointer, oldSize, newSize: Natural): pointer
```

**What they do.** Full counterparts of `realloc`/`realloc0` for the
shared heap: they resize a block while preserving its data, and
`reallocShared0` additionally zeroes the new tail.

- **Parameters**: same as `realloc`/`realloc0` above, except the pointer
  must have come from `allocShared`/`allocShared0`.

- **Examples**:

```nim
var shared = cast[ptr UncheckedArray[int32]](allocShared(4 * sizeof(int32)))
shared[0] = 42
shared = cast[ptr UncheckedArray[int32]](reallocShared(shared, 8 * sizeof(int32)))
echo shared[0]  # prints 42
deallocShared(shared)
```

---

## IV. Memory statistics and diagnostics

### 1. `getFreeMem`

```nim
proc getFreeMem(): int
```

**What it does.** Returns the number of bytes that the current thread's
heap has already requested from the operating system but isn't currently
using for live objects (i.e. available for future `alloc` calls without
touching the OS).

- **Parameters**: none.

- **Examples**:

```nim
let before = getFreeMem()
var p = alloc(1024)
let after = getFreeMem()
echo before >= after  # prints true — free memory didn't increase
dealloc(p)
```

---

### 2. `getTotalMem`

```nim
proc getTotalMem(): int
```

**What it does.** Returns the total amount of memory (in bytes) that the
current thread's heap has ever requested from the operating system and
still holds onto (the occupied part plus the free part within
already-obtained pages).

- **Parameters**: none.

- **Examples**:

```nim
echo getTotalMem() >= getFreeMem()  # prints true — total is never less than free
```

---

### 3. `getOccupiedMem`

```nim
proc getOccupiedMem(): int
```

**What it does.** Returns the amount of memory (in bytes) actually
occupied by live objects right now, i.e. essentially
`getTotalMem() - getFreeMem()` (though internally it's tracked with a
dedicated counter rather than recomputed by subtraction each time).

- **Parameters**: none.

- **Examples**:

```nim
let before = getOccupiedMem()
var p = alloc(256)
echo getOccupiedMem() > before  # prints true
dealloc(p)
```

---

### 4. `getMaxMem`

```nim
proc getMaxMem*(): int
```

**What it does.** Returns the historical maximum amount of memory held by
the heap over the program's entire run — i.e. the peak value of
`getTotalMem()`, not the current one.

- **Implementation notes.** The peak value isn't updated on every
  allocation, only at the moment pages are released back to the
  operating system (`decCurrMem`), so the returned value is effectively
  `max(current_value, last_recorded_peak)` — which also covers the case
  where the current moment is itself a new peak.

- **Parameters**: none.

- **Examples**:

```nim
var p = alloc(1024 * 1024)
let peak = getMaxMem()
dealloc(p)
echo getMaxMem() >= peak  # prints true — the peak never decreases after a free
```

---

### 5. `getFreeSharedMem` / `getTotalSharedMem` / `getOccupiedSharedMem`

```nim
proc getFreeSharedMem(): int
proc getTotalSharedMem(): int
proc getOccupiedSharedMem(): int
```

**What they do.** Full counterparts of the three statistics procedures
above (`getFreeMem`, `getTotalMem`, `getOccupiedMem`), but for the shared
heap rather than the current thread's heap.

- **Implementation notes.** Without destructor-based GC, access to the
  shared heap's counters is guarded by the same lock as the
  `allocShared`/`deallocShared` operations themselves.

- **Parameters**: none.

- **Examples**:

```nim
var shared = allocShared(4096)
echo getOccupiedSharedMem() > 0  # prints true
deallocShared(shared)
```

---

### 6. `getMemCounters`

```nim
proc getMemCounters*(): (int, int)
```

**What it does.** Returns a pair of counters `(number of alloc calls,
number of dealloc calls)` for the current thread's heap. Only available
when compiled with `-d:nimTypeNames` — in other builds the procedure
doesn't exist at all.

- **Parameters**: none.

- **Examples**:

```nim
# Compile with -d:nimTypeNames, otherwise this procedure is unavailable.
when defined(nimTypeNames):
  var p = alloc(16)
  let (allocs, deallocs) = getMemCounters()
  echo allocs > deallocs  # prints true — dealloc hasn't been called yet
  dealloc(p)
```

---

### 7. `ptrSize`

```nim
proc ptrSize(p: pointer): int
```

**What it does.** Returns the actual usable size of the block that `p`
points to — it can be **larger** than the size originally requested, due
to rounding up to a cell size class (for small chunks) or to a page size
(for big chunks). This is exactly why `realloc` can sometimes grant
access to extra bytes "for free", if the original `alloc`'s rounding was
more generous than strictly needed.

- **Parameters**:
  - `p: pointer` — a pointer previously obtained from `alloc`/`alloc0`.

- **Examples**:

```nim
var p = alloc(1)
echo ptrSize(p) >= 1  # prints true — the real size is never less than requested
dealloc(p)
```

---

## V. Returning pages to the operating system

### 1. `deallocOsPages`

```nim
proc deallocOsPages()
```

**What it does.** Returns **all** pages held by the current thread's heap
back to the operating system, regardless of whether they're occupied by
live objects or not. It's meant to be called at thread/program shutdown,
not during ordinary operation: after the call, any access to memory
previously allocated on this heap is undefined behavior.

- **Parameters**: none.

- **Examples**:

```nim
proc workerThreadShutdown() =
  # ... shutting down; no object on this heap is needed anymore ...
  deallocOsPages()
```

---

## VI. How it works under the hood (summary overview)

The public procedures in sections II–III ultimately boil down to two
internal operations — `rawAlloc`/`rawDealloc` — which decide which of
three paths to take, depending on the request size:

- **Small block** (fits in a `SmallChunk`) is served with almost no
  arithmetic: if the active chunk of the right size class has free
  cells, the head of the list is taken; if there are no cells but there
  is room in the "accumulator", the accumulator pointer is advanced.
  Both paths are O(1). Only when there's no suitable chunk at all does
  the code request a new chunk from the big-block machinery.
- **Big block** is looked up in the TLSF matrix `a.matrix[fl][sl]` — a
  two-dimensional table where the `fl`/`sl` ("first level"/"second
  level" indices) are found almost instantly through bitwise operations
  on `flBitmap`/`slBitmap` (finding the position of the first set bit),
  with no list traversal at all. If there's no suitable chunk, new pages
  are requested from the OS, and any leftover remainder is cut off
  (`splitChunk`) and placed back into the matrix.
- **Huge block** (larger than `HugeChunkSize`) is allocated and freed
  directly through the OS, bypassing the matrix entirely — such blocks
  are rare and large enough that caching them isn't worthwhile.

When big blocks are freed, adjacent free chunks are **coalesced** into
one — this guards against heap fragmentation at the cost of an extra
left/right neighbor check on every `dealloc`.

Tracking where each big chunk begins is handled by two structures: an
`IntSet` (a bitset of chunk-start addresses — a fast check for "is this
address the start of a chunk") and an AVL tree of intervals (a fast
lookup for "which chunk does this pointer fall inside", needed by the
garbage collector to validate interior pointers).

For the shared heap across threads, one more layer is added: instead of
immediately locking when freeing a "foreign" block (one allocated by a
different thread), the operation often simply places the block on the
owner's lock-free deferred-free list — actual release happens later,
when the owner itself next calls into the allocator.

---

## VII. Practical recipes

### 1. A fixed-size object pool

```nim
type
  Particle = object
    x, y, vx, vy: float32

proc newParticlePool(capacity: int): ptr UncheckedArray[Particle] =
  # alloc0 guarantees zero velocities/coordinates for every particle at once.
  result = cast[ptr UncheckedArray[Particle]](
    alloc0(capacity * sizeof(Particle)))

proc freeParticlePool(pool: ptr UncheckedArray[Particle]) =
  dealloc(pool)

var pool = newParticlePool(1000)
pool[0].x = 5.0
echo pool[0].x, " ", pool[1].x  # prints 5.0 0.0
freeParticlePool(pool)
```

---

### 2. A growable buffer (like `seq`) via `realloc`

```nim
type GrowBuffer = object
  data: ptr UncheckedArray[byte]
  len, cap: int

proc push(buf: var GrowBuffer, b: byte) =
  if buf.len >= buf.cap:
    # Grow with headroom (doubling) so we don't pay for a realloc on every push:
    let newCap = max(8, buf.cap * 2)
    buf.data = cast[ptr UncheckedArray[byte]](realloc(buf.data, newCap))
    buf.cap = newCap
  buf.data[buf.len] = b
  inc buf.len

proc destroy(buf: var GrowBuffer) =
  dealloc(buf.data)
  buf.len = 0
  buf.cap = 0

var buf = GrowBuffer(data: alloc(8), len: 0, cap: 8)
for i in 0 ..< 20:
  push(buf, byte(i))
echo buf.len, " ", buf.cap  # prints 20 32 — capacity grew by doubling
destroy(buf)
```

---

### 3. Passing a buffer between threads via `allocShared`

```nim
# The buffer is allocated on the main thread but freed by whichever
# thread finishes working with it — this is only valid for memory from
# allocShared/deallocShared, not for regular alloc/dealloc.
proc makeSharedPayload(size: int): pointer =
  result = allocShared0(size)

proc consumeAndFree(p: pointer) {.thread.} =
  # ... process the data in the child thread ...
  deallocShared(p)  # safe, even though it was allocated on another thread

var payload = makeSharedPayload(4096)
var worker: Thread[pointer]
createThread(worker, consumeAndFree, payload)
joinThread(worker)
```

---

### 4. Monitoring memory usage

```nim
proc logMemoryUsage(tag: string) =
  echo tag, ": occupied=", getOccupiedMem(),
       " total=", getTotalMem(),
       " peak=", getMaxMem()

logMemoryUsage("before allocation")
var buffers: seq[pointer]
for i in 0 ..< 100:
  buffers.add(alloc(4096))
logMemoryUsage("after allocation")
for p in buffers:
  dealloc(p)
logMemoryUsage("after freeing")
```

---

## VIII. Quick reference table

| Task | Crosses owner/thread | Call |
|---|---|---|
| Allocate memory on the thread's heap | — | `alloc(size)` |
| Allocate zeroed memory on the thread's heap | — | `alloc0(size)` |
| Free memory allocated by your own thread | — | `dealloc(p)` |
| Resize a block from your own heap | — | `realloc(p, newSize)` |
| Resize with zeroing of the new tail | — | `realloc0(p, oldSize, newSize)` |
| Allocate memory reachable from any thread | cross-thread | `allocShared(size)` |
| Same, zeroed | cross-thread | `allocShared0(size)` |
| Free shared memory (from any thread) | cross-thread | `deallocShared(p)` |
| Resize a shared block | cross-thread | `reallocShared(p, newSize)` |
| Find the thread heap's free memory | — | `getFreeMem()` |
| Find the thread heap's total memory | — | `getTotalMem()` |
| Find the thread heap's occupied memory | — | `getOccupiedMem()` |
| Find the historical peak usage | — | `getMaxMem()` |
| Same three metrics for the shared heap | cross-thread | `getFreeSharedMem` / `getTotalSharedMem` / `getOccupiedSharedMem` |
| Find alloc/dealloc call counts (`-d:nimTypeNames` only) | — | `getMemCounters()` |
| Find the actual (rounded) size of a block | — | `ptrSize(p)` |
| Return all of the thread's pages to the OS | — | `deallocOsPages()` |

---

## IX. Summary: which procedure to use

- Need to allocate memory that only the current thread will use →
  `alloc` (or `alloc0`, if a zeroed initial state matters).
- Need to free such memory → `dealloc`.
- Need to resize an already-allocated block while keeping its data →
  `realloc`; if the growth also needs zeroing → `realloc0`.
- Need to hand a buffer to another thread, or free it from a thread
  other than the one that allocated it → use the `allocShared`/
  `deallocShared` pair from the start, instead of `alloc`/`dealloc`.
- Need to log or bound memory usage → `getOccupiedMem` for the current
  value, `getMaxMem` for the historical peak, `getFreeMem`/`getTotalMem`
  for heap details.
- Need to know how many bytes are actually available through a pointer
  (not just what was requested) → `ptrSize`.
- Need to fully release a thread's memory before it shuts down →
  `deallocOsPages` (after this call, old pointers into this heap must not
  be used).
- Need to understand the chunk-selection/coalescing/TLSF-matrix
  mechanism itself → section VI, not the individual procedures in this
  reference — that's internal machinery, not public API.
