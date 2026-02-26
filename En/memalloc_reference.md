# `memalloc.nim` — Module Reference

> **Module:** `memalloc` — Nim Runtime Library, Memory Allocation and Raw Memory Operations  
> **Availability:** Most of this module requires `hasAlloc` (not JS, not NimScript). Raw memory primitives (`zeroMem`, `copyMem`, etc.) are also available on most non-JS targets. See per-symbol notes for exact availability.  
> **Safety:** Every function in this module is explicitly **unsafe**. They operate below Nim's type system. Use them only when you need manual memory management for interop with C, performance-critical data structures, or embedded/OS-level programming.

---

## Overview

`memalloc.nim` provides two complementary layers of memory management:

**Layer 1 — Raw memory operations** (no allocation, just byte manipulation):  
`zeroMem`, `copyMem`, `moveMem`, `equalMem`, `cmpMem`

**Layer 2 — Allocator** (two separate heaps):
- **Thread-local heap** — memory that belongs to the allocating thread and must be freed by the same thread: `alloc`, `alloc0`, `realloc`, `realloc0`, `dealloc` and their typed wrappers `create`, `createU`, `resize`.
- **Shared heap** — memory that can be passed between threads and freed by any thread: `allocShared`, `allocShared0`, `reallocShared`, `reallocShared0`, `deallocShared` and their typed wrappers `createShared`, `createSharedU`, `resizeShared`, `freeShared`.

Additionally, the module exposes GC memory statistics (`getOccupiedMem`, `getFreeMem`, `getTotalMem`) and optional allocation counters (`AllocStats`, `getAllocStats`, `dumpAllocstats`).

### The key mental model

```
Thread-local heap       Shared heap
─────────────────       ─────────────────────────────────────
alloc / alloc0          allocShared / allocShared0
realloc / realloc0      reallocShared / reallocShared0
dealloc                 deallocShared
─────────────────       ─────────────────────────────────────
createU / create        createSharedU / createShared
resize                  resizeShared
(dealloc)               freeShared
```

The `U` suffix always means **U**ninitialized (faster, more dangerous). The absence of `U` means zero-initialized (slightly slower, safer).

---

## Group 1: Raw Memory Operations

These five procedures operate on raw byte ranges. They are available on all non-JS, non-NimScript targets. They map directly to `memset`/`memcpy`/`memmove`/`memcmp` from the C standard library.

---

### `zeroMem`

```nim
proc zeroMem*(p: pointer, size: Natural) {.inline, noSideEffect, tags: [], raises: [], enforceNoRaises.}
```

**What it does:**  
Fills exactly `size` bytes starting at address `p` with the byte value `0`. Every bit in the range becomes zero. Equivalent to C's `memset(p, 0, size)`.

**When to use it:**  
- Securely clearing sensitive data (passwords, keys) from memory before freeing.
- Initializing a raw buffer to a known state before use.
- Satisfying C APIs that expect zero-filled structs.

**Why it is different from `=` assignment:**  
`zeroMem` ignores Nim's type system entirely — it writes zero bytes unconditionally, with no destructor calls, no GC interaction, and no bounds checking beyond the `size` you provide. This makes it suitable for clearing memory that Nim's type system doesn't "own" (e.g. buffers allocated via `alloc`).

```nim
# Securely erase a password buffer before freeing it
var buf = alloc(64)
# ... use buf ...
zeroMem(buf, 64)   # wipe before dealloc — avoids leaving secrets in memory
dealloc(buf)
```

```nim
# Zero-initialize a C struct received via FFI
type CConfig {.importc, header: "config.h".} = object
  flags: cint
  size:  cint
var cfg: CConfig
zeroMem(addr cfg, sizeof(CConfig))
```

> ⚠️ **Unsafe.** Passing a wrong `size` silently corrupts adjacent memory.

---

### `copyMem`

```nim
proc copyMem*(dest, source: pointer, size: Natural) {.inline, gcsafe, tags: [], raises: [], enforceNoRaises.}
```

**What it does:**  
Copies exactly `size` bytes from `source` to `dest`. Equivalent to C's `memcpy`. **The source and destination regions must not overlap.** If they might overlap, use `moveMem` instead.

**When to use it:**  
- Copying raw buffers (audio samples, pixel data, network packets) between pre-allocated regions.
- Bulk-copying a value type to a raw buffer for serialisation.
- Interfacing with C code that works with raw byte arrays.

**Overlapping regions:**  
If `dest` and `source` overlap, the result is undefined behaviour — some bytes may be read after being overwritten. This is the same as C's `memcpy` restriction.

```nim
var src = [1'u8, 2, 3, 4, 5]
var dst: array[5, uint8]
copyMem(addr dst, addr src, 5)
# dst is now [1, 2, 3, 4, 5]
```

```nim
# Copy a struct to a raw byte buffer for network serialisation
type Header = object
  magic: uint32
  length: uint32

var h = Header(magic: 0xDEADBEEF'u32, length: 42)
var wire: array[8, byte]
copyMem(addr wire, addr h, sizeof(Header))
```

> ⚠️ **Unsafe.** No bounds checking. Overlapping regions → undefined behaviour.

---

### `moveMem`

```nim
proc moveMem*(dest, source: pointer, size: Natural) {.inline, gcsafe, tags: [], raises: [], enforceNoRaises.}
```

**What it does:**  
Copies exactly `size` bytes from `source` to `dest`, correctly handling the case where the two regions **overlap**. Equivalent to C's `memmove`. It is slightly slower than `copyMem` because it must handle the overlap case, but safe to use whenever you are unsure whether the regions overlap.

**Rule of thumb:**
- Regions *definitely* do not overlap → use `copyMem` (faster).
- Regions *might* overlap → use `moveMem` (safe).

```nim
# Shift elements left by one position within the same array (overlapping)
var data = [10'u8, 20, 30, 40, 50]
moveMem(addr data[0], addr data[1], 4)
# data is now [20, 30, 40, 50, 50]  (last byte is stale but that's fine)
```

> ⚠️ **Unsafe.** No bounds checking. Still undefined behaviour if `size` exceeds the allocated region.

---

### `equalMem`

```nim
proc equalMem*(a, b: pointer, size: Natural): bool {.inline, noSideEffect, tags: [], raises: [], enforceNoRaises.}
```

**What it does:**  
Compares exactly `size` bytes starting at `a` with `size` bytes starting at `b`. Returns `true` if all bytes are identical, `false` otherwise. Equivalent to `memcmp(a, b, size) == 0`.

**When to use it:**  
- Comparing raw buffers that have no meaningful Nim type (e.g. comparing two arbitrary byte arrays).
- Fast equality check for fixed-size structs when you trust their padding bytes are deterministic.

**Caveat with padding:** Structs may contain compiler-inserted padding bytes whose values are undefined. `equalMem` compares *all* bytes including padding, so two logically equal structs might not be `equalMem`-equal if their padding bytes differ. For structs, prefer field-by-field comparison unless you know there is no padding.

```nim
var a = [1'u8, 2, 3]
var b = [1'u8, 2, 3]
var c = [1'u8, 2, 4]
echo equalMem(addr a, addr b, 3)  # true
echo equalMem(addr a, addr c, 3)  # false
```

> ⚠️ **Unsafe.** No bounds checking. Padding bytes in structs may give unexpected results.

---

### `cmpMem`

```nim
proc cmpMem*(a, b: pointer, size: Natural): int {.inline, noSideEffect, tags: [], raises: [], enforceNoRaises.}
```

**What it does:**  
Lexicographically compares `size` bytes starting at `a` with `size` bytes at `b`, treating each byte as an unsigned value. Returns:
- A **negative** integer if the first differing byte in `a` is less than in `b`.
- A **positive** integer if the first differing byte in `a` is greater than in `b`.
- **Zero** if all `size` bytes are identical.

Equivalent to C's `memcmp(a, b, size)`.

**When to use it:**  
- Implementing custom sort or ordering for raw byte arrays.
- Comparing fixed-length binary keys (e.g. in a hash table or a binary search).

```nim
var x = [1'u8, 2, 3]
var y = [1'u8, 2, 5]
let r = cmpMem(addr x, addr y, 3)
if r < 0:
  echo "x < y"   # this branch is taken
```

> ⚠️ **Unsafe.** No bounds checking.

---

## Group 2: Thread-Local Heap Allocation

Memory on the thread-local heap **must be freed by the same thread that allocated it**. Never pass this memory to another thread and free it there. Use the shared heap for cross-thread data.

Available when `hasAlloc and not defined(js)`.

---

### `alloc`

```nim
template alloc*(size: Natural): pointer
```

**What it does:**  
Allocates a raw, **uninitialized** block of at least `size` bytes on the thread-local heap. Returns a raw `pointer`. The contents of the block are undefined — reading before writing is undefined behaviour.

**Lifecycle:** Must be freed with `dealloc(p)` or `realloc(p, 0)`. If you forget, the memory leaks for the lifetime of the process.

```nim
let buf = alloc(256)        # raw, uninitialized block
zeroMem(buf, 256)           # explicitly zero before reading
# ... use buf ...
dealloc(buf)
```

> Use `alloc0` if you want automatic zeroing. Use `createU` if you want a typed pointer.

---

### `alloc0`

```nim
template alloc0*(size: Natural): pointer
```

**What it does:**  
Like `alloc`, but **zero-initializes** the entire block before returning. Every byte is guaranteed to be `0`. Slightly slower than `alloc` due to the extra write pass.

```nim
let p = alloc0(sizeof(int) * 10)   # 10 ints, all zero
let arr = cast[ptr UncheckedArray[int]](p)
echo arr[0]   # safe to read: guaranteed 0
dealloc(p)
```

---

### `createU`

```nim
proc createU*(T: typedesc, size = 1.Positive): ptr T {.inline, gcsafe, raises: [].}
```

**What it does:**  
A typed wrapper around `alloc`. Allocates `T.sizeof * size` bytes and returns a `ptr T`. The block is **uninitialized**. `size` is the number of `T` elements, not a byte count — this is the main advantage over raw `alloc`.

**When to prefer over `alloc`:** Whenever you know the type of the data you're allocating — the type parameter catches size calculation errors at compile time.

```nim
type Vec3 = object
  x, y, z: float32

let v = createU(Vec3)   # one Vec3, uninitialized
v.x = 1.0
v.y = 2.0
v.z = 3.0
dealloc(v)

# Allocate an array of 100 Vec3 structs:
let arr = createU(Vec3, 100)
# arr[0] .. arr[99] are accessible, but uninitialized
dealloc(arr)
```

---

### `create`

```nim
proc create*(T: typedesc, size = 1.Positive): ptr T {.inline, gcsafe, raises: [].}
```

**What it does:**  
A typed wrapper around `alloc0`. Allocates `T.sizeof * size` zero-initialized bytes and returns a `ptr T`. Safer than `createU` because the memory is guaranteed clean.

```nim
type Node = object
  value: int
  next: ptr Node

let n = create(Node)     # zero-initialized: value = 0, next = nil
echo n.next.isNil        # true — guaranteed by zero initialization
dealloc(n)
```

---

### `realloc`

```nim
template realloc*(p: pointer, newSize: Natural): pointer
```

**What it does:**  
Resizes a previously allocated block. Behaviour by case:
- `p` is `nil` → acts like `alloc(newSize)`, allocates a fresh block.
- `newSize == 0` and `p != nil` → acts like `dealloc(p)`, frees the block.
- Otherwise → grows or shrinks the block to at least `newSize` bytes. The pointer `p` becomes **invalid** after this call; use only the returned pointer.

**New bytes when growing** are **uninitialized**. Use `realloc0` if you need them zeroed.

```nim
var p = alloc(64)
# Need more space:
p = realloc(p, 128)   # old p is invalid now; use the returned pointer
# ... use p with 128 bytes ...
dealloc(p)
```

---

### `realloc0`

```nim
template realloc0*(p: pointer, oldSize, newSize: Natural): pointer
```

**What it does:**  
Like `realloc`, but **zero-initializes any newly added bytes** when growing. Requires you to pass `oldSize` explicitly so the runtime knows which bytes are new. Slower than `realloc` when growing, but prevents reading uninitialized data from the expanded region.

```nim
var p = alloc0(32)
p = realloc0(p, 32, 64)  # bytes 32..63 are now guaranteed zero
dealloc(p)
```

---

### `resize`

```nim
proc resize*[T](p: ptr T, newSize: Natural): ptr T {.inline, gcsafe, raises: [].}
```

**What it does:**  
A typed wrapper around `realloc`. Resizes the block to hold `T.sizeof * newSize` bytes. Returns a `ptr T`. `newSize` is an element count, not a byte count.

```nim
var arr = createU(int, 10)
arr = resize(arr, 20)    # now holds 20 ints; new slots are uninitialized
dealloc(arr)
```

---

### `dealloc`

```nim
proc dealloc*(p: pointer) {.noconv, compilerproc, rtl, gcsafe, raises: [], tags: [].}
```

**What it does:**  
Frees a block previously obtained from `alloc`, `alloc0`, `realloc`, `create`, or `createU`. After calling `dealloc(p)`, the pointer `p` is **dangling** — any access to it is undefined behaviour.

**Rules:**
- Only the **allocating thread** may free thread-local memory.
- Double-free is undefined behaviour (likely crashes or corruption).
- Passing `nil` may or may not crash depending on the underlying allocator — do not rely on it.

```nim
let p = alloc(128)
# ... use p ...
dealloc(p)
# p is now dangling — do not use it anymore
```

> ⚠️ **Extremely dangerous.** Memory leaks, double-free, and use-after-free are all silent at the Nim level and undefined behaviour at the C level.

---

## Group 3: Shared Heap Allocation

The shared heap can be accessed from **any thread**. Memory allocated here may be passed to other threads and freed there. Internally backed by a mutex-protected allocator or the system `malloc` (with `-d:useMalloc`).

Available when `hasAlloc and not defined(js)`.

---

### `allocShared`

```nim
template allocShared*(size: Natural): pointer
```

**What it does:**  
Allocates at least `size` **uninitialized** bytes on the shared heap. The block may be used and freed by any thread.

```nim
# Producer thread allocates, consumer thread frees — valid with allocShared
let buf = allocShared(1024)
sendToOtherThread(buf)
```

---

### `allocShared0`

```nim
template allocShared0*(size: Natural): pointer
```

**What it does:**  
Like `allocShared` but zero-initializes the block.

```nim
let zeroBuf = allocShared0(512)
# All 512 bytes are 0
```

---

### `createSharedU`

```nim
proc createSharedU*(T: typedesc, size = 1.Positive): ptr T {.inline, tags: [], gcsafe, raises: [].}
```

**What it does:**  
Typed wrapper around `allocShared`. Allocates `T.sizeof * size` **uninitialized** bytes on the shared heap, returns `ptr T`.

```nim
type Msg = object
  kind: int
  data: array[64, byte]

let m = createSharedU(Msg)
m.kind = 42
sendMessage(m)   # m will be freed by the receiving thread
```

---

### `createShared`

```nim
proc createShared*(T: typedesc, size = 1.Positive): ptr T {.inline.}
```

**What it does:**  
Typed wrapper around `allocShared0`. Allocates zero-initialized `T.sizeof * size` bytes on the shared heap.

```nim
let m = createShared(Msg)   # m.kind == 0, m.data all zeros
```

---

### `reallocShared`

```nim
template reallocShared*(p: pointer, newSize: Natural): pointer
```

**What it does:**  
Resizes a shared-heap block. Same semantics as `realloc` (nil → alloc, 0 → free, otherwise resize). New bytes are uninitialized.

---

### `reallocShared0`

```nim
template reallocShared0*(p: pointer, oldSize, newSize: Natural): pointer
```

**What it does:**  
Like `reallocShared` but zero-initializes newly added bytes. Requires explicit `oldSize`.

---

### `resizeShared`

```nim
proc resizeShared*[T](p: ptr T, newSize: Natural): ptr T {.inline, raises: [].}
```

**What it does:**  
Typed wrapper around `reallocShared`. Resizes to `T.sizeof * newSize` bytes, returns `ptr T`.

---

### `deallocShared`

```nim
proc deallocShared*(p: pointer) {.noconv, compilerproc, rtl, gcsafe, raises: [], tags: [].}
```

**What it does:**  
Frees a block obtained from `allocShared`, `allocShared0`, or `reallocShared`. Like `dealloc`, but for the shared heap. Any thread may call this.

```nim
# Receiving thread frees memory allocated by the producer
proc receiver(buf: pointer) =
  processData(buf)
  deallocShared(buf)
```

> ⚠️ All the same dangers as `dealloc`: double-free, use-after-free, and dangling pointers.

---

### `freeShared`

```nim
proc freeShared*[T](p: ptr T) {.inline, gcsafe, raises: [].}
```

**What it does:**  
Typed convenience wrapper around `deallocShared`. Accepts a `ptr T` instead of a raw `pointer`. Functionally identical to `deallocShared(p)`.

```nim
let m = createShared(Msg)
# ... use m ...
freeShared(m)   # typed — cleaner at call site than deallocShared(m)
```

---

## Group 4: Allocation Statistics and Diagnostics

Available when `hasAlloc and not defined(js)`. Counter tracking requires `-d:nimAllocStats`.

---

### `AllocStats`

```nim
type AllocStats* = object
  allocCount: int
  deallocCount: int
```

**What it is:**  
A plain data object holding two counters: the total number of allocation calls and the total number of deallocation calls since the program started (or since the last snapshot, when used with subtraction).

The fields are not directly exported (lowercase), but the `getAllocStats` proc returns a full value and the `-` operator lets you compute differences between two snapshots.

---

### `getAllocStats`

```nim
proc getAllocStats*(): AllocStats
```

**What it does:**  
Returns the current global allocation counters. When compiled with `-d:nimAllocStats`, the counters are incremented atomically on every `alloc`/`dealloc` call. Without `-d:nimAllocStats`, always returns zero-valued stats (no overhead).

```nim
let before = getAllocStats()
runSomeCode()
let after = getAllocStats()
let delta = after - before
echo "Allocations:   ", delta.allocCount    # requires accessing fields via delta
echo "Deallocations: ", delta.deallocCount
```

---

### `dumpAllocstats`

```nim
template dumpAllocstats*(code: untyped)
```

**What it does:**  
A convenience template that wraps a block of code, takes a stats snapshot before and after, computes the difference, and echoes it. Useful for quick "how many allocations does this function make?" checks during development.

**Important:** Without `-d:nimAllocStats` all counters are zero and the output will be uninformative.

```nim
dumpAllocstats:
  var s = newSeq[int](1000)
  for i in 0..<1000:
    s[i] = i * i
# Prints the allocation delta caused by the block above
```

---

## Group 5: Memory Usage Queries

These functions query the GC/allocator about how the process uses memory. Available when `hasAlloc`.

---

### `getOccupiedMem`

```nim
proc getOccupiedMem*(): int {.rtl.}
```

**What it does:**  
Returns the number of bytes currently **in use** — owned by the process and containing live data. This is the sum of all currently allocated, not-yet-freed blocks tracked by Nim's allocator. On JS, returns `-1`.

```nim
echo "Live data: ", getOccupiedMem(), " bytes"
```

---

### `getFreeMem`

```nim
proc getFreeMem*(): int {.rtl.}
```

**What it does:**  
Returns the number of bytes the process **owns but is not currently using** — memory that has been allocated from the OS but is sitting in the allocator's free list, waiting to be reused. On JS, returns `-1`.

```nim
# A large free pool means the allocator has pre-claimed memory from the OS
echo "Free pool: ", getFreeMem(), " bytes"
```

---

### `getTotalMem`

```nim
proc getTotalMem*(): int {.rtl.}
```

**What it does:**  
Returns the total bytes owned by the process from the allocator's perspective: `getTotalMem() == getOccupiedMem() + getFreeMem()`. This is not the same as the process RSS (resident set size) reported by the OS, because it counts only memory tracked by Nim's allocator. On JS, returns `-1`.

```nim
echo "Total heap: ", getTotalMem(), " bytes"
```

---

### `getOccupiedSharedMem`

```nim
proc getOccupiedSharedMem*(): int {.rtl.}
```

**What it does:**  
Like `getOccupiedMem` but for the **shared heap**. Only available when threads are enabled (`--threads:on`) and not using system malloc (`-d:useMalloc`).

---

### `getFreeSharedMem`

```nim
proc getFreeSharedMem*(): int {.rtl.}
```

**What it does:**  
Like `getFreeMem` but for the shared heap. Same availability restrictions as `getOccupiedSharedMem`.

---

### `getTotalSharedMem`

```nim
proc getTotalSharedMem*(): int {.rtl.}
```

**What it does:**  
Like `getTotalMem` but for the shared heap. Same availability restrictions.

---

## Choosing the Right Allocator

| Need | Use |
|---|---|
| Typed, thread-local, uninitialized | `createU(T)` |
| Typed, thread-local, zero-initialized | `create(T)` |
| Typed, thread-local, resize | `resize(p, n)` |
| Raw bytes, thread-local | `alloc` / `alloc0` / `realloc` / `realloc0` |
| Free thread-local | `dealloc(p)` |
| Typed, shared (cross-thread), uninitialized | `createSharedU(T)` |
| Typed, shared, zero-initialized | `createShared(T)` |
| Typed, shared, resize | `resizeShared(p, n)` |
| Raw bytes, shared | `allocShared` / `allocShared0` / `reallocShared` |
| Free shared | `deallocShared(p)` or `freeShared(p)` |

## Safety Summary

| Risk | How to avoid |
|---|---|
| Memory leak | Every `alloc*` / `create*` must have a matching `dealloc` / `free` |
| Use-after-free | Set pointers to `nil` after freeing; never read a freed pointer |
| Double-free | Use ownership discipline; only one code path frees each block |
| Thread-local/shared mismatch | `dealloc` ≠ `deallocShared`; never mix them |
| Overlapping `copyMem` | Use `moveMem` when regions may overlap |
| Struct padding in `equalMem` | Only use `equalMem` on byte buffers without struct padding |
