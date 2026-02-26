# sysstr Module Reference

> **Module:** `sysstr`  
> **Part of:** Nim's Runtime Library  
> **Purpose:** Implements the low-level string and sequence primitives that the Nim code generator relies on. Every string concatenation, slice, resize, and `add` in user-level Nim code ultimately compiles down to calls into this module. Understanding it is key to reasoning about string performance, memory layout, and the boundary between Nim and C strings.

---

## Core Concepts

### How Nim strings are laid out in memory

A `NimString` is a pointer to a heap block with this structure:

```
 ┌──────────────────────────────────────────────────┐
 │  TGenericSeq header                              │
 │  ├─ len      : int   (current character count)   │
 │  ├─ reserved : int   (capacity + flag bits)      │
 │  └─ [elemSize: int   (only with gogc)]           │
 ├──────────────────────────────────────────────────┤
 │  data[0..len-1]  : array of char                 │
 │  data[len]       : '\0'  (C-string terminator)  │
 └──────────────────────────────────────────────────┘
```

- `len` counts *bytes*, not Unicode code points.
- `reserved` encodes the current capacity; the top bits may carry special flags (`seqShallowFlag`, `strlitFlag`).
- The trailing `\0` means any `NimString` can be cast directly to a `cstring` with zero overhead.

### How sequences differ

A `PGenericSeq` has the same `TGenericSeq` header, but the element data follows with alignment padding. The element size and alignment come from the runtime type descriptor (`PNimType`), not from the header itself.

### Growth strategy (`resize`)

Throughout the module a single growth function governs how capacity expands:

```
capacity ≤ 0       → 4
0 < capacity < 64K → capacity × 2
capacity ≥ 64K     → capacity × 1.5
```

This balances fast doubling for small buffers against reduced waste for large ones.

---

## Table of Contents

1. [String Allocation](#string-allocation)
2. [String Conversion](#string-conversion)
3. [String Copying](#string-copying)
4. [String Mutation & Growth](#string-mutation--growth)
5. [Sequence Operations](#sequence-operations)
6. [Capacity Introspection](#capacity-introspection)

---

## String Allocation

### `rawNewString`

```nim
proc rawNewString(space: int): NimString {.compilerproc.}
```

**What it is:** Allocates a new, empty `NimString` with at least `space` bytes of reserved capacity. The string is fully initialized: `len` is set to `0` and the first byte is `'\0'`, making it immediately usable as a C string. Minimum actual capacity is always 7 (to avoid tiny, rapidly-reallocated buffers).

This is the workhorse behind `""` in generated C code and behind `newStringOfCap`.

```nim
# Generated C code for:  s = ""
# → rawNewString(0)
#
# Generated C code for:  tmp = rawNewString(6 + name.len + 17)
# (pre-allocates for a known-length concatenation)
```

> **Note on `NoInit` variant:** Internally, `rawNewStringNoInit` is used when the caller guarantees it will fill in `len` and the null terminator itself — skipping zeroing for a small speedup. The public-facing `rawNewString` always leaves the string in a safe, usable state.

---

### `mnewString`

```nim
proc mnewString(len: int): NimString {.compilerproc.}
```

**What it is:** Allocates a `NimString` of *exactly* `len` bytes, all set to `'\0'`. Both `len` and `reserved` are set to `len`, and the whole data region is zeroed. This is how `newString(n)` is implemented — it gives you a string pre-filled with null bytes, ready to be written to index by index.

The difference from `rawNewString`: `rawNewString` gives you an *empty* string with reserved space; `mnewString` gives you a string already *of* the requested length, filled with zero bytes.

```nim
# Nim:
let s = newString(10)   # len=10, all '\0'
# Internally → mnewString(10)
```

---

## String Conversion

### `nimToCStringConv`

```nim
proc nimToCStringConv(s: NimString): cstring {.compilerproc, nonReloadable, inline.}
```

**What it is:** The zero-cost bridge from `NimString` to `cstring`. Because Nim strings are already null-terminated, this is simply a pointer cast to `addr s.data` — no copy, no allocation. If `s` is `nil` or empty, returns the empty C string literal `""` to avoid passing a null pointer to C functions.

Called every time Nim passes a `string` to a C `cstring` parameter.

```nim
proc puts(s: cstring) {.importc, header: "<stdio.h>".}

let msg = "Hello"
puts(msg)   # compiler inserts nimToCStringConv(msg) here — free operation
```

---

### `toNimStr`

```nim
proc toNimStr(str: cstring, len: int): NimString {.compilerproc.}
```

**What it is:** Creates a new `NimString` by copying exactly `len` bytes from the C string `str`. The result is properly terminated with `'\0'`. Used by the compiler whenever a C string of known length needs to become a Nim string (e.g., string literals, FFI return values where the length is already computed).

```nim
# When a C function returns a char* with known length,
# the compiler wraps it: toNimStr(cResult, cResultLen)
```

---

### `cstrToNimstr`

```nim
proc cstrToNimstr(str: cstring): NimString {.compilerRtl.}
```

**What it is:** Converts a null-terminated C string of *unknown* length to a `NimString` by first calling `str.len` (which scans for `'\0'`) and then delegating to `toNimStr`. Returns `nil` if `str` is `nil`.

This is the runtime function behind `$someCstring` when the length must be discovered at runtime.

```nim
proc getenv(key: cstring): cstring {.importc, header: "<stdlib.h>".}

let path = $getenv("PATH")   # compiler inserts cstrToNimstr(getenv("PATH"))
```

---

### `copyStrLast`

```nim
proc copyStrLast(s: NimString, start, last: int): NimString {.compilerproc.}
```

**What it is:** Extracts the substring `s[start..last]` into a freshly allocated `NimString`. `start` is clamped to `0`; `last` is clamped to `s.len - 1`. Returns `nil` if `s` is `nil`. This procedure exists primarily for **bootstrapping** — newer compiler versions no longer emit calls to it, but it must remain for the compiler to compile itself.

```nim
# Equivalent to (older compiler):  s[2..5]
# → copyStrLast(s, 2, 5)
```

---

### `copyStr`

```nim
proc copyStr(s: NimString, start: int): NimString {.compilerproc.}
```

**What it is:** Like `copyStrLast`, but copies from `start` to the end of the string. Delegates to `copyStrLast(s, start, s.len - 1)`. Also present solely for **bootstrapping** compatibility.

```nim
# Equivalent to (older compiler):  s[3..^1]
# → copyStr(s, 3)
```

---

## String Copying

### `copyString`

```nim
proc copyString(src: NimString): NimString {.compilerRtl.}
```

**What it is:** The standard deep-copy of a `NimString`. Returns `nil` if `src` is `nil`. If `src` carries `seqShallowFlag` (marking it as a shallow copy), the same pointer is returned without copying. Otherwise allocates a new buffer and copies all bytes including the null terminator.

When `nimShallowStrings` is defined and `src` is a string literal (`strlitFlag`), the copy is made shallow to avoid unnecessary duplication of constant data.

Called by the compiler wherever Nim's value-semantics demand a copy — e.g., when assigning a string to a new variable in a non-`move` context.

```nim
var a = "hello"
var b = a          # compiler calls copyString(a) here
b[0] = 'H'        # a is unaffected
```

---

### `copyStringRC1`

```nim
proc copyStringRC1(src: NimString): NimString {.compilerRtl.}
```

**What it is:** A copy variant optimised for reference-counted GC backends (ARC/ORC). Under those backends, if a `newObjRC1` fast-path is available, the allocation is done with a reference count pre-set to 1, avoiding an extra atomic increment. Under region or non-RC GC, it falls back to `toOwnedCopy`, making it identical to `copyString`. The `seqShallowFlag` logic is the same.

---

### `copyDeepString`

```nim
proc copyDeepString(src: NimString): NimString {.inline, raises: [].}
```

**What it is:** An unconditional deep copy — always allocates a fresh buffer regardless of shallow flags. Used in contexts where Nim's `deepCopy` semantics are required (e.g., passing strings across threads or to channels).

```nim
# Sending across a channel always deep-copies:
chan.send(deepCopy(s))   # internally uses copyDeepString
```

---

## String Mutation & Growth

### `resizeString`

```nim
proc resizeString(dest: NimString, addlen: int): NimString {.compilerRtl.}
```

**What it is:** Ensures `dest` has enough capacity for `addlen` more bytes **without** updating `len`. If `dest` already has sufficient space, it is returned unchanged. If not, a new, larger buffer is allocated using the `resize` growth strategy (at least `dest.len + addlen`, possibly more), the existing content is copied, and the new buffer is returned.

This is the *pre-allocation step* the compiler inserts before a series of `appendString` calls. The two-phase pattern (resize once, then append multiple times) is the key to efficient `&=` and multi-part `&` concatenation.

```nim
# Nim:   s &= "Hello " & name & "!"
# Generated C (conceptually):
#   s = resizeString(s, 6 + name.len + 1)
#   appendString(s, "Hello ")
#   appendString(s, name)
#   appendString(s, "!")
```

> **Important:** After `resizeString`, the returned pointer may differ from the input (if reallocation occurred). Always use the return value.

---

### `appendString`

```nim
proc appendString(dest, src: NimString) {.compilerproc, inline.}
```

**What it is:** Copies all bytes of `src` (including its null terminator) to the end of `dest`, and increments `dest.len` by `src.len`. Does **not** check capacity — it is always preceded by a `resizeString` call in generated code. A no-op if `src` is `nil`.

This is the *cheapest possible append*: a single `copyMem` + one integer increment.

---

### `appendChar`

```nim
proc appendChar(dest: NimString, c: char) {.compilerproc, inline.}
```

**What it is:** Appends a single character `c` to `dest`, updates the null terminator, and increments `len`. Like `appendString`, does not check capacity. Called in tight loops where the compiler knows exactly one character is being appended.

```nim
# Nim:  s.add('\n')
# → appendChar(s, '\n')  (after ensuring capacity)
```

---

### `addChar`

```nim
proc addChar(s: NimString, c: char): NimString
```

**What it is:** A *safe* single-character append that handles capacity checking and reallocation internally. If `s` is `nil`, allocates a new string. If `s` is full, grows it using the `resize` strategy. Returns the (possibly new) string pointer. Used in `ccgexprs.nim` when a single-character add cannot be statically pre-sized.

---

### `setLengthStr`

```nim
proc setLengthStr(s: NimString, newLen: int): NimString {.compilerRtl.}
```

**What it is:** Resizes the string to exactly `newLen` characters. Handles all three cases cleanly:

- **Growth:** allocates a larger buffer, copies old content, zeroes the new tail.
- **Shrink:** keeps the same buffer, updates `len` and places `'\0'` at the new end.
- **No change:** still updates the null terminator at `s[newLen]` for safety.

Negative `newLen` is treated as 0. If `s` is `nil` and `newLen > 0`, returns a zeroed string of the requested length via `mnewString`.

This is the backing implementation of `setLen(s, n)` for strings.

```nim
var s = "Hello, World!"
s.setLen(5)    # → setLengthStr(s, 5) → s is now "Hello"
s.setLen(10)   # → setLengthStr(s, 10) → "Hello\0\0\0\0\0"
```

---

## Sequence Operations

### `incrSeq`

```nim
proc incrSeq(seq: PGenericSeq, elemSize, elemAlign: int): PGenericSeq {.compilerproc.}
```

**What it is:** Grows a sequence by exactly one element (increments `len` by 1) and returns the (possibly reallocated) sequence pointer. If the current length equals the capacity, `growObj` is called with the next `resize`-step capacity. The caller is then responsible for writing the actual element into `seq[seq.len - 1]`.

This is the direct implementation of `seq.add(elem)` in generated C code.

```nim
# Nim:
var s: seq[int]
s.add(42)
# Generated C (conceptually):
#   s = incrSeq(s, sizeof(int), alignof(int))
#   s[s.len - 1] = 42
```

---

### `incrSeqV3`

```nim
proc incrSeqV3(s: PGenericSeq, typ: PNimType): PGenericSeq {.compilerproc.}
```

**What it is:** A newer version of `incrSeq` that takes a full `PNimType` descriptor instead of raw element size/alignment, enabling proper use of `newSeq` for reallocation (important for GC correctness). Handles `nil` input by allocating a fresh length-1 sequence with `len` set to 0. When growing, steals the content from the old sequence (`s.len = 0`) to prevent double-accounting by the GC.

The "V3" suffix indicates this is the current default used by the compiler for most `add` operations on sequences.

---

### `setLengthSeqV2`

```nim
proc setLengthSeqV2(s: PGenericSeq, typ: PNimType, newLen: int, isTrivial: bool): PGenericSeq {.compilerRtl.}
```

**What it is:** Resizes a generic sequence to `newLen` elements, initializing any new elements to zero. Handles all three cases:

- **Growth:** extends capacity via `extendCapacityRaw` if needed, then zeroes the new element slots.
- **Shrink:** calls `truncateRaw`, which decrements reference counts for non-trivial types (under tracing GC backends) and zeroes freed slots.
- **No change:** in-place update of `len`.

This is the backing implementation of `setLen(mySeq, n)` for sequences.

The `isTrivial` flag signals that elements contain no managed references — when `true`, the destructor/zeroing logic for truncation is skipped, making shrinks faster.

```nim
var s = @[1, 2, 3, 4, 5]
s.setLen(2)    # → setLengthSeqV2(..., 2, true) → @[1, 2]
s.setLen(10)   # → setLengthSeqV2(..., 10, true) → @[1, 2, 0, 0, 0, 0, 0, 0, 0, 0]
```

---

### `setLengthSeqUninit`

```nim
proc setLengthSeqUninit(s: PGenericSeq; typ: PNimType; newLen: int; isTrivial: bool): PGenericSeq {.compilerRtl.}
```

**What it is:** Identical to `setLengthSeqV2` except that **new elements are not zeroed on growth**. Used when the compiler can prove that every newly created slot will be written before it is read — trading safety for speed. The `doInit = false` path in the underlying template is taken.

> **Do not rely on the content of new elements after `setLengthSeqUninit`.** They contain whatever was in the allocator's memory at that address.

---

## Capacity Introspection

### `capacity` (string)

```nim
func capacity*(self: string): int {.inline.}
```

**What it is:** Returns the number of bytes the string can hold *without reallocation* — i.e., the value of `reserved` in the underlying `NimString`. Returns `0` for `nil` strings. This is the string counterpart of `seq.capacity`.

Knowing the capacity matters when you want to pre-allocate exactly the right amount and avoid mid-loop reallocations.

```nim
var s = newStringOfCap(42)
s.add "Nim"
assert s.capacity == 42   # still 42; no reallocation happened
assert s.len == 3
```

---

### `capacity` (seq)

```nim
func capacity*[T](self: seq[T]): int {.inline.}
```

**What it is:** Returns the number of elements the sequence can hold without reallocation — the `space` field of the underlying `PGenericSeq`. Returns `0` for `nil` sequences.

```nim
var lst = newSeqOfCap[string](42)
lst.add "Nim"
assert lst.capacity == 42
assert lst.len == 1

# Pre-allocate to avoid reallocation in a known-length loop:
var results = newSeqOfCap[int](n)
for i in 0..<n:
  results.add computeSomething(i)
assert results.capacity >= n   # at most one allocation occurred
```

---

*This reference covers all exported and compiler-facing symbols from `sysstr.nim` in Nim's runtime library.*
