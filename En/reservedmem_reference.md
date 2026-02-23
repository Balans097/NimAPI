# `reservedmem` Module Reference

**Module:** `reservedmem` (part of Nim's standard library)  
**Status:** ⚠️ Unstable API — interfaces may change between Nim versions.  
**Purpose:** Reserve a large block of virtual address space without immediately consuming physical RAM, then commit physical pages into it on demand as the buffer grows.

---

## The core idea

Modern operating systems separate two concepts that look like one:

- **Virtual address space** — the range of memory addresses a process may name. Reserving it is almost free: no RAM is consumed, no swap is used.
- **Physical (committed) memory** — actual RAM (or swap) backing those addresses. Reading or writing a committed page is what costs real resources.

`reservedmem` exploits this split. You call `init` once, telling the OS: *"I might eventually need up to N bytes starting here — please keep this address range for me."* The OS reserves the range but does not allocate RAM. Later, as the buffer grows, the module commits only the pages that are actually needed, in aligned chunks. When the buffer shrinks significantly, excess committed pages are returned to the OS.

The payoff: **the buffer's base address never changes**, even as it grows, because the virtual range was locked in at creation time. This is something an ordinary `realloc`-based buffer cannot guarantee.

```
Virtual address space (reserved, no RAM cost):
┌─────────────────────────────────────────────────────────────────┐
│  committed + used  │  committed, free  │  reserved (no RAM)     │
└────────────────────┴───────────────────┴────────────────────────┘
^memStart            ^usedMemEnd         ^committedMemEnd          ^memEnd
```

### Platform mapping

| Platform | Reserve | Commit | Decommit |
|---|---|---|---|
| Windows | `VirtualAlloc(MEM_RESERVE)` | `VirtualAlloc(MEM_COMMIT)` | `VirtualFree(MEM_DECOMMIT)` |
| POSIX | `mmap(PROT_NONE)` | `mprotect(accessFlags)` | `posix_madvise(DONTNEED)` |

---

## Exported symbols

### Types

| Symbol | Description |
|---|---|
| `MemAccessFlags` | Integer alias for OS memory protection constants |
| `ReservedMem` | A raw byte buffer backed by virtual memory reservation |
| `ReservedMemSeq[T]` | A typed, sequence-like view over a `ReservedMem` |

### Access-flag constants

| Constant | Windows | POSIX | Meaning |
|---|---|---|---|
| `memExec` | `PAGE_EXECUTE` | `PROT_EXEC` | Execute only |
| `memExecRead` | `PAGE_EXECUTE_READ` | `PROT_EXEC\|READ` | Execute + read |
| `memExecReadWrite` | `PAGE_EXECUTE_READWRITE` | `PROT_EXEC\|READ\|WRITE` | Execute + read + write |
| `memRead` | `PAGE_READONLY` | `PROT_READ` | Read only |
| `memReadWrite` | `PAGE_READWRITE` | `PROT_READ\|WRITE` | Read + write *(default)* |

### Utility templates

| Symbol | Description |
|---|---|
| `distance(lhs, rhs)` | Signed byte distance between two pointers: `rhs - lhs` |
| `shift(p, distance)` | Advance a pointer by `distance` bytes |

### Procedures and functions

| Symbol | Belongs to | Kind |
|---|---|---|
| `ReservedMem.init(...)` | `ReservedMem` | proc |
| `ReservedMem.len` | `ReservedMem` | func |
| `ReservedMem.commitedLen` | `ReservedMem` | func |
| `ReservedMem.maxLen` | `ReservedMem` | func |
| `ReservedMem.setLen` | `ReservedMem` | proc |
| `ReservedMemSeq.init(...)` | `ReservedMemSeq[T]` | proc |
| `ReservedMemSeq[][pos]` | `ReservedMemSeq[T]` | func (4 overloads) |
| `ReservedMemSeq.len` | `ReservedMemSeq[T]` | func |
| `ReservedMemSeq.setLen` | `ReservedMemSeq[T]` | proc |
| `ReservedMemSeq.add` | `ReservedMemSeq[T]` | proc |
| `ReservedMemSeq.pop` | `ReservedMemSeq[T]` | proc |
| `ReservedMemSeq.commitedLen` | `ReservedMemSeq[T]` | func |
| `ReservedMemSeq.maxLen` | `ReservedMemSeq[T]` | func |

---

## Utility templates

### `distance`

```nim
template distance*(lhs, rhs: pointer): int
```

Returns the signed byte offset from `lhs` to `rhs` as an `int`. Equivalent to `cast[int](rhs) - cast[int](lhs)`. A negative result means `rhs` is before `lhs` in address space.

Used internally to compute how many bytes lie between the bookmarks inside `ReservedMem`, but also useful in your own low-level code.

```nim
import reservedmem

var a: array[10, byte]
let p = addr a[0]
let q = addr a[7]
echo distance(p, q)   # 7  (q is 7 bytes after p)
echo distance(q, p)   # -7
```

### `shift`

```nim
template shift*(p: pointer, distance: int): pointer
```

Returns a new pointer that is `distance` bytes ahead of `p`. A negative `distance` moves backwards. This is pointer arithmetic without the type constraints of typed pointers.

```nim
import reservedmem

var buf: array[16, byte]
let base = addr buf[0]
let mid  = base.shift(8)   # points at buf[8]
echo distance(base, mid)   # 8
```

---

## `MemAccessFlags`

```nim
type MemAccessFlags* = int
```

An integer alias whose values are OS memory-protection flags. You always use the named constants (`memRead`, `memReadWrite`, etc.) rather than constructing raw integers. The same symbolic name maps to the correct OS constant on each platform automatically.

The default for all `init` calls is `memReadWrite` — readable and writable pages. Use `memRead` for shared read-only data, or `memExecReadWrite` for JIT-compiled code buffers (where you write machine instructions and then execute them).

---

## `ReservedMem`

```nim
type ReservedMem* = object
```

A raw, untyped, byte-oriented buffer whose virtual address range is fixed at creation time. Think of it as a low-level foundation; for typed data prefer `ReservedMemSeq[T]`.

Internal bookmarks (all private):

| Field | Meaning |
|---|---|
| `memStart` | First byte of the reserved range |
| `usedMemEnd` | One byte past the last *used* byte |
| `committedMemEnd` | One byte past the last *committed* page |
| `memEnd` | One byte past the entire reserved range |

---

### `ReservedMem.init`

```nim
proc init*(T: type ReservedMem,
           maxLen: Natural,
           initLen: Natural = 0,
           initCommitLen = initLen,
           memStart = pointer(nil),
           accessFlags = memReadWrite,
           maxCommittedAndUnusedPages = 3): ReservedMem
```

**The constructor.** Creates a new `ReservedMem` by reserving `maxLen` bytes of virtual address space and optionally committing an initial portion.

#### Parameters explained in depth

**`maxLen`** — the hard ceiling on how large the buffer can ever grow. The OS reserves this many bytes of virtual address space immediately. Choosing a large value (e.g. 1 GB) is fine on 64-bit systems where address space is plentiful; no RAM is consumed until pages are committed.

**`initLen`** — how many bytes are considered *used* from the start (i.e. the initial logical length of the buffer). Must be `≤ initCommitLen`. Defaults to 0.

**`initCommitLen`** — how many bytes of physical memory to commit up front. The actual committed size is rounded up to the allocation granularity (page size on POSIX, allocation granularity on Windows). Defaults to `initLen`. Setting this higher than `initLen` pre-warms memory for the expected early growth, avoiding page faults on the first writes.

**`memStart`** — a hint to the OS about the preferred base address. Pass `nil` to let the OS choose (recommended). Passing a specific address may fail silently if the OS cannot honour it.

**`accessFlags`** — the protection bits for committed pages. Default: `memReadWrite`.

**`maxCommittedAndUnusedPages`** — how many pages of slack are kept committed after `setLen` shrinks the buffer. A larger value reduces the frequency of decommit/recommit cycles at the cost of slightly more RAM usage. Default: 3.

#### Example

```nim
import reservedmem

# Reserve 64 MB of address space; commit nothing yet.
var buf = ReservedMem.init(maxLen = 64 * 1024 * 1024)

# Reserve 64 MB; commit 4 KB initially; logical length is 1 KB.
var buf2 = ReservedMem.init(
  maxLen       = 64 * 1024 * 1024,
  initLen      = 1024,
  initCommitLen= 4096
)

echo buf2.len         # 1024
echo buf2.commitedLen # 4096 (or aligned up)
echo buf2.maxLen      # 67108864
```

---

### `ReservedMem.len`

```nim
func len*(m: ReservedMem): int
```

Returns the number of bytes currently considered *used* — the logical length of the buffer. This is the distance from `memStart` to `usedMemEnd`.

```nim
var buf = ReservedMem.init(maxLen = 1024 * 1024, initLen = 256)
echo buf.len   # 256
```

---

### `ReservedMem.commitedLen`

```nim
func commitedLen*(m: ReservedMem): int
```

Returns how many bytes of physical memory are currently committed (i.e. backed by real RAM or swap). This is always `≥ len` and is aligned to the allocation granularity. The extra committed-but-unused bytes are the "slack" kept to avoid frequent OS calls on repeated small grows.

> **Note:** The name `commitedLen` (single 't') is a typo in the original source preserved for API stability.

```nim
var buf = ReservedMem.init(maxLen = 1024 * 1024, initLen = 100)
echo buf.len         # 100
echo buf.commitedLen # ≥ 100, aligned to page size (e.g. 4096)
```

---

### `ReservedMem.maxLen`

```nim
func maxLen*(m: ReservedMem): int
```

Returns the total size of the reserved virtual address range — the value passed as `maxLen` to `init`. This is the absolute maximum the buffer can ever grow to without creating a new reservation.

```nim
var buf = ReservedMem.init(maxLen = 10 * 1024 * 1024)
echo buf.maxLen   # 10485760
```

---

### `ReservedMem.setLen`

```nim
proc setLen*(m: var ReservedMem, newLen: int)
```

Resizes the logical length of the buffer to `newLen` bytes. This is the central operation — it drives all committing and decommitting.

**Growing (`newLen > len`):**  
If `newLen` exceeds the currently committed range, the module commits additional pages to cover the new length. The extension size is rounded up to the allocation granularity, so a single byte of growth may commit a full 4 KB page. Raw bytes in the newly committed area are zeroed on Windows (guaranteed by `VirtualAlloc`) but may contain garbage on POSIX.

**Shrinking (`newLen < len`):**  
The logical length moves back, but committed pages are not immediately returned. Pages are decommitted only when the unused committed area exceeds `maxCommittedAndUnusedPages` pages worth of slack. This prevents thrashing when the buffer oscillates around a size boundary.

`newLen` must be between 0 and `maxLen`. Violating this causes undefined behaviour (the OS call will fail or corrupt state).

```nim
import reservedmem

var buf = ReservedMem.init(maxLen = 4 * 1024 * 1024)

buf.setLen(1000)
echo buf.len         # 1000
echo buf.commitedLen # 4096 (one page committed)

buf.setLen(5000)
echo buf.len         # 5000
echo buf.commitedLen # 8192 (two pages now)

buf.setLen(100)
echo buf.len         # 100
# committed pages may still be 8192 until slack exceeds the threshold
```

---

## `ReservedMemSeq[T]`

```nim
type ReservedMemSeq*[T] = object
```

A typed, sequence-like wrapper around `ReservedMem`. It stores elements of type `T` contiguously in memory, at a fixed address, growing on demand. The interface closely mirrors Nim's built-in `seq[T]`, but with bounded maximum capacity and stable base address.

All length/index parameters of `ReservedMemSeq` are in units of **elements**, not bytes.

---

### `ReservedMemSeq.init`

```nim
proc init*(SeqType: type ReservedMemSeq,
           maxLen: Natural,
           initLen: Natural = 0,
           initCommitLen: Natural = 0,
           memStart = pointer(nil),
           accessFlags = memReadWrite,
           maxCommittedAndUnusedPages = 3): SeqType
```

Creates a typed sequence backed by a virtual memory reservation. Parameters have the same meaning as `ReservedMem.init`, but `maxLen`, `initLen`, and `initCommitLen` are in **elements** of type `T`, not bytes. The proc internally multiplies by `sizeof(T)` before calling `ReservedMem.init`.

#### Example

```nim
import reservedmem

# A sequence of at most 1 million int64 values.
# Reserve address space for all; commit nothing initially.
var s = ReservedMemSeq[int64].init(maxLen = 1_000_000)

echo s.len      # 0
echo s.maxLen   # 1000000
```

---

### `ReservedMemSeq.[](pos)` — element access

```nim
func `[]`*[T](s: ReservedMemSeq[T], pos: Natural): lent T
func `[]`*[T](s: var ReservedMemSeq[T], pos: Natural): var T
func `[]`*[T](s: ReservedMemSeq[T], rpos: BackwardsIndex): lent T
func `[]`*[T](s: var ReservedMemSeq[T], rpos: BackwardsIndex): var T
```

Four overloads of the indexing operator, covering the four combinations of:
- `const` vs `var` receiver (read-only `lent T` vs mutable `var T`)
- Forward index (`Natural`) vs backwards index (`BackwardsIndex`, written `^1`, `^2`, …)

A `rangeCheck` assertion guards against out-of-bounds access. In release builds without range checks this degrades to a raw pointer dereference.

#### Example

```nim
import reservedmem

var s = ReservedMemSeq[float32].init(maxLen = 1000, initLen = 3)
# initLen=3 means 3 elements are considered "used"; their memory is committed
# but content is undefined (POSIX) / zeroed (Windows).

# Write via mutable indexing
s[0] = 1.5
s[1] = 2.5
s[2] = 3.5

# Read via const indexing
echo s[0]    # 1.5
echo s[^1]   # 3.5  (last element, backwards index)
echo s[^2]   # 2.5
```

---

### `ReservedMemSeq.len`

```nim
func len*[T](s: ReservedMemSeq[T]): int
```

The number of elements currently in the sequence (logical length). Equivalent to `s.mem.len div sizeof(T)`.

```nim
var s = ReservedMemSeq[int32].init(maxLen = 100)
echo s.len   # 0
s.setLen(5)
echo s.len   # 5
```

---

### `ReservedMemSeq.setLen`

```nim
proc setLen*[T](s: var ReservedMemSeq[T], newLen: int)
```

Changes the number of elements in the sequence. Internally converts to bytes and delegates to `ReservedMem.setLen`.

> ⚠️ **Destructors not called.** The source code contains a `TODO` note: when shrinking, destructors for the removed elements are not invoked. For types with managed resources (strings, seqs, refs) this leaks memory. Use this proc safely only with plain-old-data types (`int`, `float`, `object` without ref fields, etc.).

```nim
import reservedmem

var s = ReservedMemSeq[int32].init(maxLen = 1000)
s.setLen(10)
echo s.len   # 10
s.setLen(3)
echo s.len   # 3
```

---

### `ReservedMemSeq.add`

```nim
proc add*[T](s: var ReservedMemSeq[T], val: T)
```

Appends one element to the end of the sequence. Calls `setLen(len + 1)` then writes `val` at the new last position. If the new length requires committing additional pages, this happens transparently.

This will panic (or corrupt state) if `len` is already at `maxLen`.

```nim
import reservedmem

var s = ReservedMemSeq[int32].init(maxLen = 1_000_000)

for i in 0 ..< 5:
  s.add(i * 10)

echo s.len    # 5
echo s[0]     # 0
echo s[4]     # 40
```

---

### `ReservedMemSeq.pop`

```nim
proc pop*[T](s: var ReservedMemSeq[T]): T
```

Removes and returns the last element. Calls `setLen(len - 1)` after reading the value.

> ⚠️ **Assertion:** An `assert` guards against popping from an empty sequence. In release builds this assertion may be compiled away, leading to a read from invalid memory. Always check `len > 0` before calling `pop` in release code.

> ⚠️ **Destructors not called** (same caveat as `setLen`).

```nim
import reservedmem

var s = ReservedMemSeq[int32].init(maxLen = 100)
s.add(10)
s.add(20)
s.add(30)

echo s.pop()   # 30
echo s.pop()   # 20
echo s.len     # 1
```

---

### `ReservedMemSeq.commitedLen`

```nim
func commitedLen*[T](s: ReservedMemSeq[T]): int
```

The number of elements for which physical memory is currently committed. Always `≥ len`. Useful for diagnosing how much RAM the sequence is actually using compared to its logical size.

```nim
var s = ReservedMemSeq[int64].init(maxLen = 100_000)
s.add(42)
echo s.len          # 1
echo s.commitedLen  # depends on page size; likely 512 (4096 bytes / 8 bytes per int64)
```

---

### `ReservedMemSeq.maxLen`

```nim
func maxLen*[T](s: ReservedMemSeq[T]): int
```

The maximum number of elements the sequence can ever hold (the value passed as `maxLen` to `init`). Adding beyond this limit is not handled gracefully — always track your upper bound.

```nim
var s = ReservedMemSeq[uint8].init(maxLen = 1024)
echo s.maxLen   # 1024
```

---

## Full worked example — stable-address growing log buffer

```nim
# Demonstrates the key property: the buffer's address never changes
# while it grows up to its reserved maximum.

import reservedmem

const MaxEvents = 1_000_000

type Event = object
  timestamp: int64
  code: int32
  value: float32

var log = ReservedMemSeq[Event].init(
  maxLen            = MaxEvents,
  maxCommittedAndUnusedPages = 5  # keep 5 pages of slack when shrinking
)

# Record a stable base address
let baseAddr = addr log.mem.memStart   # address will not change

echo "Log capacity: ", log.maxLen, " events"
echo "Initial RAM:  ", log.commitedLen * sizeof(Event), " bytes"

# Simulate an event loop
for i in 0 ..< 10_000:
  log.add(Event(timestamp: i, code: int32(i mod 16), value: float32(i) * 0.1))

echo "After 10k events:"
echo "  len          = ", log.len
echo "  commitedLen  = ", log.commitedLen
echo "  maxLen       = ", log.maxLen

# Access elements — base address is guaranteed stable
echo "First event timestamp: ", log[0].timestamp
echo "Last  event timestamp: ", log[^1].timestamp

# Shrink — excess pages decommitted lazily
log.setLen(100)
echo "After shrink to 100:"
echo "  len          = ", log.len
echo "  commitedLen  = ", log.commitedLen  # still ≥ 100, may retain some pages
```

---

## Design considerations and caveats

**Address stability is the defining feature.** Unlike `seq[T]`, which may reallocate and move its buffer, `ReservedMemSeq` guarantees that element addresses remain valid for the lifetime of the object. This makes it safe to store raw pointers or references into the sequence.

**`maxLen` must be chosen up front.** There is no way to extend the reserved region after `init`. Over-reserving on 64-bit systems is harmless (no RAM cost), but on 32-bit systems address space is a scarce resource.

**Destructors are not called** by `setLen`, `pop`, or anything else in this module. Only use it with plain-old-data types unless you manage cleanup manually.

**POSIX commit does not zero memory.** `mprotect` merely changes protection on already-`mmap`d memory that was zeroed at `mmap` time (kernel zeros new pages before giving them to a process). In practice newly committed pages on Linux are zeroed, but this is a kernel guarantee tied to `mmap`, not to `mprotect`. Windows `VirtualAlloc(MEM_COMMIT)` explicitly zeroes.

**This is an unstable API.** The module header warns that interfaces may change. Do not depend on it in library code without pinning your Nim version.
