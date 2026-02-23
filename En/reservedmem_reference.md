# Nim `reservedmem` Module — Complete Reference

## Overview

The `reservedmem` module provides two data structures for managing **virtual address space reservations**: `ReservedMem` (a raw byte buffer) and `ReservedMemSeq[T]` (a typed sequence built on top of it). Both share the same fundamental promise:

> Reserve a large region of virtual address space upfront, then commit (make physically backed) only the pages you actually use — and the buffer's base address **never moves**.

This is the key distinction from Nim's ordinary `seq[T]`. A `seq` grows by allocating a new, larger block and copying everything over; the old pointer becomes invalid. A `ReservedMemSeq` never copies, because it has already claimed its maximum address range from the OS. As pages are needed they are committed in-place; as pages become unused they can be decommitted to return physical RAM to the OS, while the virtual address range stays stable.

### The virtual memory model in brief

Modern operating systems give each process a large virtual address space (typically 128 TiB on 64-bit Linux, or 8 TiB on 64-bit Windows). Mapping a region of that space costs essentially nothing — it does not consume RAM or swap. Only when you **commit** (or **mprotect** on POSIX) a page does the OS arrange to back it with physical memory.

`reservedmem` exploits this by:
1. **Reserving** the maximum size with a single OS call at init time (cheap).
2. **Committing** pages incrementally as `setLen` / `add` requests more space.
3. **Decommitting** pages when `setLen` shrinks the used region significantly.

### When to use this module

- You need a buffer or array that **must not move** (e.g. you hand out raw pointers or slices to other code).
- You know an upper bound on the size but do not know the actual size in advance.
- You want to avoid the overhead of copying on growth (useful for large buffers or hot paths).
- You are building custom allocators, ring buffers, or memory-mapped-like data structures.

### API stability

Marked **Unstable API** — signatures or behaviour may change without deprecation.

---

## Exported API — Quick Reference

### Pointer utilities

| Symbol | Kind | Description |
|--------|------|-------------|
| `distance` | template | Signed byte distance from one pointer to another |
| `shift` | template | Advance a pointer by a byte offset |

### Memory access flags

| Constant | Windows | POSIX | Meaning |
|----------|---------|-------|---------|
| `memExec` | `PAGE_EXECUTE` | `PROT_EXEC` | Execute only |
| `memExecRead` | `PAGE_EXECUTE_READ` | `PROT_EXEC\|PROT_READ` | Execute + read |
| `memExecReadWrite` | `PAGE_EXECUTE_READWRITE` | `PROT_EXEC\|PROT_READ\|PROT_WRITE` | Execute + read + write |
| `memRead` | `PAGE_READONLY` | `PROT_READ` | Read only |
| `memReadWrite` | `PAGE_READWRITE` | `PROT_READ\|PROT_WRITE` | Read + write (default) |

### Types

| Type | Description |
|------|-------------|
| `MemAccessFlags` | Integer alias for OS page-protection flags |
| `ReservedMem` | Raw byte buffer over a reserved address range |
| `ReservedMemSeq[T]` | Typed sequence over a reserved address range |

### `ReservedMem` API

| Symbol | Kind | Description |
|--------|------|-------------|
| `ReservedMem.init` | proc | Reserve and optionally commit an initial region |
| `len` | func | Number of bytes currently in the used region |
| `commitedLen` | func | Number of bytes currently committed (backed by RAM) |
| `maxLen` | func | Total bytes in the reserved address range |
| `setLen` | proc | Resize the used region, committing or decommitting pages |

### `ReservedMemSeq[T]` API

| Symbol | Kind | Description |
|--------|------|-------------|
| `ReservedMemSeq.init` | proc | Reserve and optionally commit typed element storage |
| `len` | func | Number of elements in the sequence |
| `commitedLen` | func | Number of elements whose memory is committed |
| `maxLen` | func | Maximum number of elements the reservation can hold |
| `setLen` | proc | Resize the sequence |
| `add` | proc | Append one element |
| `pop` | proc | Remove and return the last element |
| `[]` | func | Index access (by position or `BackwardsIndex`) |

---

## Pointer Utility Templates

### `distance`

```nim
template distance*(lhs, rhs: pointer): int
```

Returns the **signed byte distance** from pointer `lhs` to pointer `rhs`, computed as `cast[int](rhs) - cast[int](lhs)`. A positive result means `rhs` is at a higher address; a negative result means `rhs` is at a lower address.

This template exists because Nim does not allow arithmetic directly on `pointer`; you must cast to `int` first. `distance` hides that casting boilerplate.

```nim
import std/reservedmem

var buf = ReservedMem.init(maxLen = 4096)
let p = buf.mem.memStart          # base pointer (internal field)
let q = buf.mem.usedMemEnd        # end-of-used pointer

echo distance(p, q)   # 0, since initLen defaults to 0
```

---

### `shift`

```nim
template shift*(p: pointer, distance: int): pointer
```

Returns a new pointer that is `distance` bytes ahead of `p` in memory. Negative values move backwards. Equivalent to C's `(char*)p + distance`.

Like `distance`, it exists to avoid repetitive `cast[pointer](cast[int](p) + d)` noise.

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 1024, initLen = 64)
# The used region ends at:
let endPtr = shift(m.mem.memStart, m.len)
```

---

## Memory Access Flag Constants

These constants are the Nim equivalents of the OS page-protection flags passed to `VirtualAlloc` (Windows) or `mmap`/`mprotect` (POSIX). They control what operations are allowed on committed pages.

| Constant | Read | Write | Execute | Typical use |
|----------|------|-------|---------|------------|
| `memRead` | ✓ | — | — | Read-only data |
| `memReadWrite` | ✓ | ✓ | — | General data (**default**) |
| `memExec` | — | — | ✓ | Rarely used standalone |
| `memExecRead` | ✓ | — | ✓ | JIT: read + execute, no write |
| `memExecReadWrite` | ✓ | ✓ | ✓ | JIT: write code then execute |

The `memNoAccess` constant (inaccessible pages) is defined internally but not exported — it is used for the initial reservation on Windows (`PAGE_NOACCESS`) and as the POSIX `PROT_NONE` equivalent for the `mmap` reservation.

```nim
import std/reservedmem

# Read-only reserved buffer:
var roMem = ReservedMem.init(maxLen = 65536, accessFlags = memRead)

# JIT code buffer (write first, then switch to exec+read):
var codeBuf = ReservedMem.init(maxLen = 1024 * 1024,
                               accessFlags = memExecReadWrite)
```

---

## `ReservedMem`

```nim
type ReservedMem* = object
```

An object that manages a reserved region of virtual address space. Internally it tracks four pointers into the region:

| Internal field | Meaning |
|---------------|---------|
| `memStart` | Base address of the reserved region (constant after init) |
| `usedMemEnd` | One byte past the last byte in the "used" portion |
| `committedMemEnd` | One byte past the last byte backed by physical memory |
| `memEnd` | One byte past the entire reserved region |

The invariant is always: `memStart ≤ usedMemEnd ≤ committedMemEnd ≤ memEnd`.

Physical memory is committed in **page-aligned chunks** (the OS allocation granularity, typically 4 KiB on POSIX and 64 KiB on Windows). The committed region may therefore be larger than the used region.

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

Creates and returns a new `ReservedMem`. This is the only constructor; there is no equivalent of `new`.

#### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `maxLen` | *(required)* | Total bytes to reserve in the virtual address space. This is the hard upper limit — the buffer can never grow beyond this. It does **not** consume physical RAM. |
| `initLen` | `0` | The initial "used" size in bytes. Bytes in `[0, initLen)` are considered valid data immediately after init. |
| `initCommitLen` | `initLen` | How many bytes to physically back with RAM at init time. Must be ≥ `initLen`. Rounded up to the OS page size internally. |
| `memStart` | `nil` | Hint to the OS for where to place the reservation. `nil` lets the OS choose (strongly recommended). |
| `accessFlags` | `memReadWrite` | Page-protection flags for all committed pages. |
| `maxCommittedAndUnusedPages` | `3` | How many extra committed-but-unused pages are tolerated before decommitting on `setLen` shrink. Higher values reduce decommit/recommit thrashing at the cost of keeping more RAM pinned. |

`initLen ≤ initCommitLen` is asserted; violating this crashes in debug mode.

#### What happens on the OS level

**Windows:** `VirtualAlloc(MEM_RESERVE)` reserves the address range, then `VirtualAlloc(MEM_COMMIT)` backs the initial pages.

**POSIX:** `mmap(PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS)` reserves without backing; `mprotect(accessFlags)` backs the initial pages.

#### Examples

```nim
import std/reservedmem

# Reserve 1 MiB, commit nothing yet:
var m1 = ReservedMem.init(maxLen = 1024 * 1024)
assert m1.len == 0
assert m1.commitedLen == 0
assert m1.maxLen == 1024 * 1024

# Reserve 1 MiB, commit and mark 64 KiB as used immediately:
var m2 = ReservedMem.init(maxLen = 1024 * 1024,
                          initLen = 65536,
                          initCommitLen = 65536)
assert m2.len == 65536
assert m2.maxLen == 1024 * 1024
```

---

### `len` (ReservedMem)

```nim
func len*(m: ReservedMem): int
```

Returns the number of bytes in the **used** portion of the buffer — i.e. `distance(memStart, usedMemEnd)`. This is the logical size that your code cares about; committed but unused bytes beyond this point are invisible.

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 4096, initLen = 100)
assert m.len == 100
```

---

### `commitedLen` (ReservedMem)

```nim
func commitedLen*(m: ReservedMem): int
```

Returns the number of bytes currently **backed by physical memory** — `distance(memStart, committedMemEnd)`. This value is always ≥ `len` and is always a multiple of the OS page size. It is useful for diagnostics and for deciding whether a `setLen` call will trigger an OS page commit.

> **Note:** The name `commitedLen` (single `t`) matches the source code spelling — this is not a typo in the reference.

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 1024 * 1024, initLen = 1)
# commitedLen is rounded up to page size (e.g. 4096 on Linux):
assert m.commitedLen >= 1
assert m.commitedLen mod 4096 == 0   # aligned to page boundary
assert m.commitedLen >= m.len
```

---

### `maxLen` (ReservedMem)

```nim
func maxLen*(m: ReservedMem): int
```

Returns the total number of bytes in the reserved address range — `distance(memStart, memEnd)`. This is the ceiling passed to `init` as `maxLen`. `setLen` will assert/raise if you try to grow beyond this.

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 1024 * 1024)
assert m.maxLen == 1024 * 1024
```

---

### `setLen` (ReservedMem)

```nim
proc setLen*(m: var ReservedMem, newLen: int)
```

Changes the used size of the buffer to `newLen` bytes.

**Growing** (`newLen > len`): if the new end falls outside the currently committed region, additional pages are committed (Windows: `VirtualAlloc(MEM_COMMIT)`; POSIX: `mprotect` with the stored `accessFlags`). The commit extension is always rounded up to the OS page granularity.

**Shrinking** (`newLen < len`): the used pointer moves back. If the gap between `usedMemEnd` and `committedMemEnd` now exceeds `maxCommittedAndUnusedPages * pageSize`, the excess pages are decommitted (Windows: `VirtualFree(MEM_DECOMMIT)`; POSIX: `posix_madvise(POSIX_MADV_DONTNEED)`), freeing physical RAM while keeping the virtual address range intact.

**Stability guarantee:** the base address `memStart` never changes, regardless of how many grow/shrink cycles occur.

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 4096 * 100)

m.setLen(4096)       # commit first page
assert m.len == 4096

m.setLen(4096 * 10)  # commit more pages
assert m.len == 4096 * 10

m.setLen(0)          # shrink; excess pages may be decommitted
assert m.len == 0
```

#### Writing data after setLen

`setLen` does not initialise the newly committed memory — that is your responsibility:

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 1024)
m.setLen(64)

# Write to the committed region via a typed pointer:
let buf = cast[ptr array[64, byte]](m.mem.memStart)
for i in 0 ..< 64:
  buf[i] = byte(i)
```

---

## `ReservedMemSeq[T]`

```nim
type ReservedMemSeq*[T] = object
  mem: ReservedMem
```

A typed sequence over a `ReservedMem`. Internally it is nothing more than a `ReservedMem` with `sizeof(T)` used as the conversion factor between element counts and byte offsets. It provides a high-level, `seq`-like API (index, `add`, `pop`, `setLen`) on top of the stable-address guarantee.

**Important limitations vs `seq[T]`:**

- No automatic memory management — you must call `setLen(0)` or let the object go out of scope (but there is no destructor, so OS memory is returned only when the process exits, or if you call the OS APIs manually). Destructors are noted as a TODO in the source.
- No copying on growth.
- Index bounds are checked via `rangeCheck` (disabled with `-d:danger`).
- `pop` does **not** run the destructor of the removed element (TODO in source).

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

Creates a `ReservedMemSeq[T]`. All size parameters are in **element counts**, not bytes — the proc internally multiplies by `sizeof(T)` before calling `ReservedMem.init`.

#### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `maxLen` | *(required)* | Maximum number of elements the sequence can ever hold |
| `initLen` | `0` | Number of elements considered used immediately |
| `initCommitLen` | `0` | Number of elements whose storage is physically committed at init |
| `memStart` | `nil` | OS hint for base address (leave as `nil`) |
| `accessFlags` | `memReadWrite` | Page-protection flags |
| `maxCommittedAndUnusedPages` | `3` | Decommit threshold (in OS pages, not elements) |

```nim
import std/reservedmem

# A stable-address sequence of up to 1 million ints, nothing committed yet:
var s = ReservedMemSeq[int].init(maxLen = 1_000_000)
assert s.len == 0
assert s.maxLen == 1_000_000

# Pre-commit storage for 1000 elements:
var t = ReservedMemSeq[float64].init(maxLen = 100_000, initCommitLen = 1000)
assert t.commitedLen >= 1000
```

---

### `len` (ReservedMemSeq)

```nim
func len*[T](s: ReservedMemSeq[T]): int
```

Returns the number of elements currently in the sequence (`mem.len div sizeof(T)`).

```nim
import std/reservedmem

var s = ReservedMemSeq[int32].init(maxLen = 1000, initLen = 5)
assert s.len == 5
```

---

### `commitedLen` (ReservedMemSeq)

```nim
func commitedLen*[T](s: ReservedMemSeq[T]): int
```

Returns the number of elements whose underlying storage is physically committed (`mem.commitedLen div sizeof(T)`). May be larger than `len` due to page-granularity rounding.

---

### `maxLen` (ReservedMemSeq)

```nim
func maxLen*[T](s: ReservedMemSeq[T]): int
```

Returns the maximum number of elements the sequence can ever contain without failure (`mem.maxLen div sizeof(T)`).

---

### `setLen` (ReservedMemSeq)

```nim
proc setLen*[T](s: var ReservedMemSeq[T], newLen: int)
```

Resizes the sequence to `newLen` elements, committing or decommitting OS pages as needed. Behaviour mirrors `ReservedMem.setLen`, scaled by `sizeof(T)`.

> **Note:** Destructors for removed elements are **not** called (marked as a TODO in the source). For types with non-trivial cleanup, call destructors manually before shrinking.

```nim
import std/reservedmem

var s = ReservedMemSeq[int].init(maxLen = 10_000)
s.setLen(500)
assert s.len == 500
s.setLen(100)
assert s.len == 100
```

---

### `add`

```nim
proc add*[T](s: var ReservedMemSeq[T], val: T)
```

Appends a single element to the end of the sequence. Internally calls `setLen(len + 1)` then assigns `val` to the new last slot. This may trigger a page commit if the new element falls outside the committed region.

```nim
import std/reservedmem

var s = ReservedMemSeq[int].init(maxLen = 10_000)
s.add(42)
s.add(99)
assert s.len == 2
assert s[0] == 42
assert s[1] == 99
```

---

### `pop`

```nim
proc pop*[T](s: var ReservedMemSeq[T]): T
```

Removes and returns the last element of the sequence. Calls `setLen(len - 1)` after reading the value, which may trigger page decommitting.

Asserts that the sequence is non-empty (`usedMemEnd != memStart`); calling `pop` on an empty sequence crashes in debug mode.

> **Note:** The destructor of the popped element is **not** run.

```nim
import std/reservedmem

var s = ReservedMemSeq[int].init(maxLen = 100)
s.add(10)
s.add(20)
s.add(30)

assert s.pop() == 30
assert s.pop() == 20
assert s.len == 1
```

---

### `[]` — element access

```nim
func `[]`*[T](s: ReservedMemSeq[T], pos: Natural): lent T
func `[]`*[T](s: var ReservedMemSeq[T], pos: Natural): var T
func `[]`*[T](s: ReservedMemSeq[T], rpos: BackwardsIndex): lent T
func `[]`*[T](s: var ReservedMemSeq[T], rpos: BackwardsIndex): var T
```

Provides index access to elements. Both forward (`Natural`) and backwards (`BackwardsIndex`, written as `s[^1]`) indexing are supported. Mutable (`var`) access is available when `s` is a `var`.

Bounds are checked with `rangeCheck` — out-of-bounds access raises an exception in normal builds and is undefined in `-d:danger` builds.

```nim
import std/reservedmem

var s = ReservedMemSeq[float64].init(maxLen = 1000)
s.add(1.5)
s.add(2.5)
s.add(3.5)

assert s[0] == 1.5
assert s[2] == 3.5
assert s[^1] == 3.5   # BackwardsIndex
assert s[^2] == 2.5

s[0] = 99.9           # mutable access
assert s[0] == 99.9
```

---

## Complete Practical Examples

### Pattern 1 — Stable pointer to a growing log buffer

```nim
import std/reservedmem

# Reserve space for 1 million log entries, never moves in memory
var log = ReservedMemSeq[uint64].init(maxLen = 1_000_000)

proc recordEvent(id: uint64) =
  log.add(id)

recordEvent(1)
recordEvent(2)
recordEvent(3)

# A raw pointer taken before any add() calls is still valid afterwards:
let firstEntry = cast[ptr uint64](log.mem.memStart)
assert firstEntry[] == 1   # valid — base address never changed
```

---

### Pattern 2 — Double-ended usage (stack discipline)

```nim
import std/reservedmem

var stack = ReservedMemSeq[int].init(maxLen = 10_000)

# Push
for i in 1..5:
  stack.add(i)

# Pop
while stack.len > 0:
  echo stack.pop()   # prints 5 4 3 2 1
```

---

### Pattern 3 — Pre-committing for latency-sensitive code

In some scenarios (real-time audio, game loops) you do not want the first few `add` calls to trigger OS page faults. Pre-commit the storage you will definitely need:

```nim
import std/reservedmem

const MaxFrameObjects = 8192

var objects = ReservedMemSeq[int64].init(
  maxLen = 1_000_000,
  initCommitLen = MaxFrameObjects   # pre-commit first 8192 slots
)

# First 8192 adds are page-fault-free:
for i in 0 ..< MaxFrameObjects:
  objects.add(int64(i))
```

---

### Pattern 4 — Raw byte buffer for a custom serialiser

```nim
import std/reservedmem

var buf = ReservedMem.init(maxLen = 16 * 1024 * 1024)   # 16 MiB max

proc writeInt32(m: var ReservedMem, value: int32) =
  let offset = m.len
  m.setLen(offset + 4)
  cast[ptr int32](m.mem.memStart.shift(offset))[] = value

proc writeBytes(m: var ReservedMem, data: openArray[byte]) =
  let offset = m.len
  m.setLen(offset + data.len)
  let dst = m.mem.memStart.shift(offset)
  copyMem(dst, unsafeAddr data[0], data.len)

buf.writeInt32(42)
buf.writeInt32(-1)
echo buf.len   # 8
```

---

### Pattern 5 — Read-only committed region

```nim
import std/reservedmem

# Allocate and fill the data, then restrict access to read-only:
var data = ReservedMemSeq[int32].init(
  maxLen = 1000,
  accessFlags = memRead       # pages committed as read-only
)
# (Note: you cannot write via add/setLen with memRead;
#  this pattern requires writing first with memReadWrite,
#  then using OS calls to change protection — or managing
#  two separate views.)
```

---

## Error Conditions

All OS-level operations are checked. On failure, `raiseOSError(osLastError())` is called, which raises an `OSError` with the platform error message.

| Situation | Behaviour |
|-----------|----------|
| `init` — OS refuses to reserve | `OSError` raised |
| `setLen` — commit fails | `OSError` raised |
| `setLen` — decommit/madvise fails | `OSError` raised |
| `initLen > initCommitLen` | `AssertionDefect` (debug) |
| `[]` out of bounds | `RangeDefect` (unless `-d:danger`) |
| `pop` on empty sequence | `AssertionDefect` (debug) |

---

## See Also

- [`std/oserrors`](https://nim-lang.org/docs/oserrors.html) — `raiseOSError`, `osLastError`
- [`std/posix`](https://nim-lang.org/docs/posix.html) — `mmap`, `mprotect`, `posix_madvise`
- [Windows VirtualAlloc documentation](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualalloc)
- [mmap(2) Linux man page](https://man7.org/linux/man-pages/man2/mmap.2.html)
