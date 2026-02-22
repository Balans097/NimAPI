# `hashes` Module Reference

> **Nim Standard Library — `std/hashes`**
> Efficient computation of hash values for built-in and user-defined Nim types.
> The foundation for all hash-table-based data structures in Nim.

---

## Table of Contents

1. [Overview & Mental Model](#overview--mental-model)
2. [Types](#types)
   - [Hash](#hash)
3. [Building-Block Operators](#building-block-operators)
   - [!& (mix)](#-mix)
   - [!$ (finish)](#-finish)
4. [Specialized Hash Functions](#specialized-hash-functions)
   - [hashWangYi1](#hashwangyi1)
   - [hashData](#hashdata)
   - [hashIdentity](#hashidentity)
5. [General `hash` Overloads](#general-hash-overloads)
   - [hash — integers and enums](#hash--integers-and-enums)
   - [hash — float](#hash--float)
   - [hash — pointer](#hash--pointer)
   - [hash — ptr\[T\]](#hash--ptrt)
   - [hash — ref\[T\]](#hash--reft)
   - [hash — string (full)](#hash--string-full)
   - [hash — string (slice)](#hash--string-slice)
   - [hash — cstring](#hash--cstring)
   - [hash — tuple, object, proc, closure](#hash--tuple-object-proc-closure)
   - [hash — openArray\[A\] (full)](#hash--openarrayt-full)
   - [hash — openArray\[A\] (slice)](#hash--openarrayt-slice)
   - [hash — set\[A\]](#hash--seta)
6. [Case- and Style-Insensitive Hashing](#case--and-style-insensitive-hashing)
   - [hashIgnoreCase (full)](#hashignorecase--full)
   - [hashIgnoreCase (slice)](#hashignorecase--slice)
   - [hashIgnoreStyle (full)](#hashignorestyle--full)
   - [hashIgnoreStyle (slice)](#hashignorestyle--slice)
7. [Implementing `hash` for Custom Types](#implementing-hash-for-custom-types)
8. [Important Rules & Pitfalls](#important-rules--pitfalls)
9. [Complete Worked Examples](#complete-worked-examples)

---

## Overview & Mental Model

A **hash function** takes a value of any type and produces a single integer (`Hash`) that acts as a compact fingerprint of that value. Hash tables (like Nim's `Table`, `HashSet`) use these fingerprints to find stored values in O(1) time on average, instead of scanning the whole collection.

The `hashes` module provides:

1. **Ready-made `hash` procs** for all built-in Nim types — integers, floats, strings, arrays, sequences, tuples, objects, etc. You don't need to do anything for these.
2. **Two building blocks** (`!&` and `!$`) for composing hashes when you write a `hash` proc for your own type.
3. **Variant hashing** functions for strings that ignore case or naming style.

```
Your value
    │
    ▼
 hash(x)         ← dispatch to the right overload
    │
    ▼
  Hash           ← a plain int; fits in a CPU register; used as a table index
```

The two building blocks implement a specific algorithm: you mix each component of your data into a running hash with `!&`, then "avalanche" the accumulated bits with `!$` to produce the final well-distributed value.

```
h = 0
h = h !& hash(field1)    ← mix
h = h !& hash(field2)    ← mix
result = !$h             ← finish (avalanche)
```

---

## Types

### `Hash`

```nim
type Hash* = int
```

The type of all hash values. It is simply an alias for `int` — a signed integer whose width matches the platform word size (32 bits on 32-bit systems, 64 bits on 64-bit systems).

Two fundamental constraints flow from this type:

1. **Hash tables must have power-of-two sizes.** This allows the runtime to use a fast bitwise `and` operation to map a `Hash` to a slot index, rather than the slower `mod` operation.
2. **The hash-equality contract must be respected:** if `a == b`, then `hash(a) == hash(b)`. The converse is not required — hash *collisions* (two unequal values sharing a hash) are allowed and handled by the table.

You never need to worry about the underlying int arithmetic: the building blocks handle all of it.

---

## Building-Block Operators

These two operators are the only things you need when writing `hash` procs for custom types. Everything else in this module uses them internally.

### `!&` (mix)

```nim
proc `!&`*(h: Hash, val: int): Hash {.inline.}
```

**What it does:** Mixes an integer `val` into a running hash accumulator `h`, and returns the new accumulator. The operation is a carefully chosen combination of addition, shifts, and XOR designed to spread bits and reduce collisions.

This is an operator, not a named procedure, so it is used with infix syntax: `h = h !& someValue`.

**When to use it:** Each time you want to incorporate one piece of data (one field, one element) into the hash you are building. You always pair it with `!$` at the end.

```nim
var h: Hash = 0
h = h !& 42          # mix in an integer directly
h = h !& hash("hi")  # mix in the hash of another value
result = !$h
```

```nim
# Typical pattern when iterating over parts of a value:
var h: Hash = 0
for item in myCollection:
  h = h !& hash(item)
result = !$h
```

---

### `!$` (finish)

```nim
proc `!$`*(h: Hash): Hash {.inline.}
```

**What it does:** Finalises a hash accumulator and returns the completed hash value. It applies a final "avalanche" step — more shifts and XOR operations — to ensure that small differences in the accumulated value produce very different output bits. Without this step, the accumulated hash would have poor distribution for short or similar inputs.

This is a unary prefix operator: `result = !$h`.

**When to use it:** Exactly once, at the very end of your `hash` proc, after all fields have been mixed in with `!&`. Never call it in the middle of accumulation.

```nim
# Correct: one call to !$ at the end
var h: Hash = 0
h = h !& hash(x.name)
h = h !& hash(x.age)
result = !$h          # ← finalise here

# Wrong: do NOT call !$ mid-way, then continue mixing
var h: Hash = 0
h = h !& hash(x.name)
h = !$h               # ← wrong! this finalises prematurely
h = h !& hash(x.age)  # subsequent mixing is now invalid
result = !$h
```

---

## Specialized Hash Functions

### `hashWangYi1`

```nim
proc hashWangYi1*(x: int64 | uint64 | Hash): Hash {.inline.}
```

**What it does:** Implements Wang Yi's `hash_v1` algorithm for 64-bit integers. This is a high-quality, single-step hash function that passed all scrambling tests in the SMHasher test suite. It is the algorithm used internally by `hash(T: Ordinal)` when the `nimIntHash1` compile flag is *not* set (the default).

You would call this directly if you want to hash a 64-bit integer with this specific algorithm — for example when building a hash for a custom type that contains `int64` fields and you want consistent, well-distributed output.

The note in the source is a helpful hint: you can safely narrow other integer widths to 64 bits before passing them in: `proc hash(x: int16): Hash = hashWangYi1(Hash(x))`.

**Platform note:** On JavaScript targets without `BigInt` support, this falls back to a 32-bit result.

```nim
import std/hashes

echo hashWangYi1(0'i64)       # some well-distributed hash of 0
echo hashWangYi1(1'i64)       # a very different value despite differing by just 1
echo hashWangYi1(high(int64)) # handles extreme values correctly
```

---

### `hashData`

```nim
proc hashData*(data: pointer, size: int): Hash
```

**What it does:** Computes a hash of `size` raw bytes starting at the memory address `data`. This is a low-level building block for situations where you have untyped binary data — for example, a pointer to a C struct, a raw buffer, or any memory region that doesn't map cleanly to a Nim type.

The function reads each byte and mixes it into a running hash using `!&`, then finalises with `!$`. This means it is intentionally simple and portable, not as fast as FarmHash or MurmurHash for large data, but correct and dependency-free.

**When to use it:** When working with raw memory (`pointer`, `ptr UncheckedArray[byte]`, FFI data). For ordinary Nim arrays or sequences of bytes, prefer the typed `hash(openArray[byte])` overload instead.

```nim
import std/hashes

var buffer = [0x48'u8, 0x65, 0x6c, 0x6c, 0x6f]  # "Hello" as bytes
let h = hashData(addr buffer[0], buffer.len)
echo h  # some hash value for these 5 bytes
```

---

### `hashIdentity`

```nim
proc hashIdentity*[T: Ordinal | enum](x: T): Hash {.inline, since: (1, 3).}
```

**What it does:** Returns `ord(x)` cast directly to `Hash` — the value is its own hash. There is no scrambling or avalanche step. This is called the "identity hash" because `hashIdentity(x) == ord(x)` for all `x`.

**When to use it:** In performance-critical scenarios where the keys of a hash table are already uniformly distributed integers (e.g. unique IDs, densely packed enum values, or externally generated random integers). Using the identity hash avoids the overhead of scrambling and can make lookups faster. However, it performs *badly* if the keys are not well-distributed — for example, consecutive small integers like `0, 1, 2, 3` will cluster in the same hash-table buckets.

```nim
import std/hashes

type Status = enum
  Pending, Running, Done, Failed

echo hashIdentity(Pending)  # 0
echo hashIdentity(Running)  # 1
echo hashIdentity(Done)     # 2

# For integers:
echo hashIdentity(42)   # 42
echo hashIdentity(100)  # 100
```

---

## General `hash` Overloads

### `hash` — integers and enums

```nim
proc hash*[T: Ordinal | enum](x: T): Hash {.inline.}
```

**What it does:** Hashes any integer type (`int`, `int8`, `int16`, `int32`, `int64`, `uint`, etc.) or `enum` value. Internally calls `hashWangYi1` by default (or falls back to the identity hash if compiled with `-d:nimIntHash1`).

The hash is computed on the `ord` value of `x`, so enums are hashed by their ordinal position, not their name.

```nim
import std/hashes

echo hash(0)          # well-distributed, not 0
echo hash(42)
echo hash(-1)
echo hash(high(int))

type Color = enum Red, Green, Blue
echo hash(Red)    # hash of 0
echo hash(Green)  # hash of 1
echo hash(Blue)   # hash of 2
```

---

### `hash` — float

```nim
proc hash*(x: float): Hash {.inline.}
```

**What it does:** Hashes a floating-point number by reinterpreting its bit pattern as an integer and then applying `hashWangYi1`. Before hashing, `x + 0.0` is computed to normalise denormalised values (ensuring that `+0.0` and `-0.0` produce the same bit pattern and thus the same hash).

```nim
import std/hashes

echo hash(3.14)
echo hash(0.0)
echo hash(-0.0)  # same as hash(0.0) due to normalisation
echo hash(float(1))  # same as hash(1.0)
```

---

### `hash` — pointer

```nim
proc hash*(x: pointer): Hash {.inline.}
```

**What it does:** Hashes a raw `pointer` value — i.e. the memory address itself, not the thing it points to. Two pointers to the same memory location produce the same hash; two pointers to different locations (even with the same pointee value) produce different hashes.

On JavaScript targets, where pointers have no numeric address, an incremental object ID tag is assigned on first hash and reused on subsequent calls.

```nim
import std/hashes

var a, b: int = 0
let pa = addr a
let pb = addr b
echo hash(pa) == hash(pa)  # true — same address
echo hash(pa) == hash(pb)  # false — different addresses
```

---

### `hash` — ptr\[T\]

```nim
proc hash*[T](x: ptr[T]): Hash {.inline.}
```

**What it does:** Hashes a typed `ptr[T]` by its address. This is a thin wrapper that casts the typed pointer to an untyped `pointer` and delegates to `hash(pointer)`. The hash captures *where* the pointer points, not *what* it points to.

```nim
import std/hashes

var arr: array[10, uint8]
# Adjacent elements have different addresses → different hashes
assert arr[0].addr.hash != arr[1].addr.hash
# A ptr and the equivalent untyped pointer hash the same way
assert cast[pointer](arr[0].addr).hash == arr[0].addr.hash
```

---

### `hash` — ref\[T\]

```nim
proc hash*[T](x: ref[T]): Hash {.inline.}
```

**What it does:** Hashes a `ref T` by the address of the heap object it refers to (i.e., the reference identity, not the value). Two `ref` variables pointing to the same object produce the same hash; two `ref` variables pointing to different objects with identical fields produce *different* hashes.

> **Important:** This proc is only available when you compile with `-d:nimPreviewHashRef`. It is expected to become the default in a future Nim release. Without this flag, attempting to hash a `ref` is a compile-time error.

```nim
# Compile with: nim c -d:nimPreviewHashRef myfile.nim
import std/hashes

type Node = ref object
  value: int

let a = Node(value: 3)
let b = Node(value: 3)  # same content, different object
let c = a               # same object as a

assert hash(a) == hash(c)  # true: same ref (same address)
assert hash(a) != hash(b)  # true: different ref (different address)

# To hash by content instead, define your own hash:
proc hash(n: Node): Hash = hash(n.value)
assert hash(a) == hash(b)  # now true: same content
```

---

### `hash` — string (full)

```nim
proc hash*(x: string): Hash
```

**What it does:** Hashes a complete string. The algorithm used is **FarmHash** on native targets (the default — fast, high-quality, passes SMHasher) or **MurmurHash3** on JavaScript targets and when compiled with `-d:nimStringHash2`. Both algorithms are case-sensitive: `"Hello"` and `"hello"` produce different hashes.

```nim
import std/hashes

echo hash("abracadabra")
assert hash("Hello") != hash("hello")   # case-sensitive
assert hash("") != hash(" ")            # even whitespace differs
assert hash("abc") == hash("abc")       # deterministic within a process
```

---

### `hash` — string (slice)

```nim
proc hash*(sBuf: string, sPos, ePos: int): Hash
```

**What it does:** Hashes a *substring* of `sBuf` from index `sPos` to `ePos`, **inclusive** on both ends. Avoids allocating a new string for the substring. `hash(s, 0, s.high)` is exactly equivalent to `hash(s)`.

```nim
import std/hashes

let s = "abracadabra"
# "abra" appears at positions 0..3 and 7..10
assert hash(s, 0, 3) == hash(s, 7, 10)  # same substring → same hash
assert hash(s, 0, 3) != hash(s, 0, 4)   # different length → different hash
```

---

### `hash` — cstring

```nim
proc hash*(x: cstring): Hash
```

**What it does:** Hashes a null-terminated C string. The result is identical to hashing the equivalent Nim `string` — the two are interchangeable for hashing purposes. Useful when interfacing with C libraries or working with `cstring` values directly.

```nim
import std/hashes

assert hash(cstring"abracadabra") == hash("abracadabra")
assert hash(cstring"Hello") != hash(cstring"hello")
```

---

### `hash` — tuple, object, proc, closure

```nim
proc hash*[T: tuple | object | proc | iterator {.closure.}](x: T): Hash
```

**What it does:** A generic hash for compound types:

- **Tuples and objects**: iterates over all fields using `fields()` and mixes their individual hashes with `!&` / `!$`. For this to work, every field type must itself have a `hash` proc defined. You can override this for any specific object type by defining your own `hash` proc for that type.
- **Plain procedures** (non-closures): hashes the raw function pointer address. Two proc values pointing to the same function have the same hash.
- **Closures**: hashes the pair of `(rawProc, rawEnv)` — the function pointer and the environment pointer — so two closures over different environments hash differently even if they share the same function body.

```nim
import std/hashes

# Tuple hashing — automatic, field-by-field
let t1 = (1, "hello")
let t2 = (1, "world")
assert hash(t1) != hash(t2)

# Object hashing — automatic
type Point = object
  x, y: int
assert hash(Point(x: 1, y: 2)) != hash(Point(x: 1, y: 3))

# Override for an object: hash only by x, ignore y
type WeirdPoint = object
  x, y: int
proc hash(p: WeirdPoint): Hash = hash(p.x)
assert hash(WeirdPoint(x: 5, y: 10)) == hash(WeirdPoint(x: 5, y: 99))

# Proc hashing
proc myFunc() = discard
let f1 = myFunc
let f2 = myFunc
assert hash(f1) == hash(f2)  # same function
```

---

### `hash` — openArray\[A\] (full)

```nim
proc hash*[A](x: openArray[A]): Hash
```

**What it does:** Hashes an entire array or sequence. The element type `A` must have a `hash` proc defined. For `byte` and `char` elements, FarmHash (or MurmurHash on JS) is used directly on the underlying bytes — this is very fast. For other element types, each element is hashed individually and the results are mixed.

Works seamlessly with `array`, `seq`, and `openArray` parameters.

```nim
import std/hashes

let a = [1, 2, 3]
let b = [1, 2, 3]
let c = [1, 2, 4]

assert hash(a) == hash(b)   # same elements → same hash
assert hash(a) != hash(c)   # different elements

# Works with sequences too
let s1 = @["x", "y"]
let s2 = @["x", "z"]
assert hash(s1) != hash(s2)

# Byte arrays use the fast path
let bytes = [0x48'u8, 0x65, 0x6c, 0x6c, 0x6f]
echo hash(bytes)
```

---

### `hash` — openArray\[A\] (slice)

```nim
proc hash*[A](aBuf: openArray[A], sPos, ePos: int): Hash
```

**What it does:** Hashes a contiguous sub-range of an array or sequence, from index `sPos` to `ePos` (both inclusive). Avoids creating a temporary slice. `hash(a, 0, a.high)` is equivalent to `hash(a)`.

```nim
import std/hashes

let a = [1, 2, 5, 1, 2, 6]
# Elements [1,2] appear at positions 0..1 and 3..4
assert hash(a, 0, 1) == hash(a, 3, 4)
assert hash(a, 0, 2) != hash(a, 3, 5)  # [1,2,5] != [1,2,6]
```

---

### `hash` — set\[A\]

```nim
proc hash*[A](x: set[A]): Hash
```

**What it does:** Hashes a Nim `set` (a bit-set of small `Ordinal` values). Iterates over all present elements, hashes each one individually, and mixes the results. The element type `A` must be a type compatible with Nim's `set` (typically `enum`, `char`, or small integers).

```nim
import std/hashes

let s1 = {'a', 'b', 'c'}
let s2 = {'a', 'b', 'c'}
let s3 = {'a', 'b', 'd'}

assert hash(s1) == hash(s2)
assert hash(s1) != hash(s3)
```

---

## Case- and Style-Insensitive Hashing

These specialised procedures hash strings while normalising certain textual differences. They use the `!&`/`!$` building blocks (not FarmHash or MurmurHash), so their output **is different** from the standard `hash(string)` even for identical inputs — they are not drop-in replacements.

> **Note:** All four of these procedures use the same mixing algorithm as the `!&`/`!$` building blocks, not the faster FarmHash/MurmurHash used by `hash(string)`. This is intentional: the normalisation pass touches every character individually, which is already the bottleneck.

### `hashIgnoreCase` — full

```nim
proc hashIgnoreCase*(x: string): Hash
```

**What it does:** Hashes a string after converting all ASCII uppercase letters to lowercase. All other characters (including underscores, digits, and non-ASCII bytes) are hashed as-is.

```nim
import std/hashes

assert hashIgnoreCase("ABRAcaDABRA") == hashIgnoreCase("abracadabra")
assert hashIgnoreCase("Hello!")      == hashIgnoreCase("HELLO!")
assert hashIgnoreCase("abc") != hash("abc")  # different algorithm!
```

**Use case:** Building a case-insensitive hash table for identifiers, HTTP headers, or any domain where `"Content-Type"` and `"content-type"` should be treated as the same key.

---

### `hashIgnoreCase` — slice

```nim
proc hashIgnoreCase*(sBuf: string, sPos, ePos: int): Hash
```

**What it does:** Like `hashIgnoreCase(string)` but operates on the substring of `sBuf` from `sPos` to `ePos` (inclusive). Avoids an allocation.

```nim
import std/hashes

let s = "ABracadabRA"
# "AB" (0..1) should equal "RA" reversed... let's check a real overlap:
assert hashIgnoreCase(s, 0, 3) == hashIgnoreCase(s, 7, 10)  # "ABra" == "abRA"
```

---

### `hashIgnoreStyle` — full

```nim
proc hashIgnoreStyle*(x: string): Hash
```

**What it does:** Hashes a string while ignoring both case *and* underscores. This reflects Nim's own identifier comparison rules, where `myVariable`, `my_variable`, and `MyVariable` are all considered the same identifier. Specifically: all `A-Z` characters are mapped to their lowercase equivalents, and all `_` characters are skipped entirely.

```nim
import std/hashes

assert hashIgnoreStyle("aBr_aCa_dAB_ra") == hashIgnoreStyle("abracadabra")
assert hashIgnoreStyle("my_func")        == hashIgnoreStyle("myFunc")
assert hashIgnoreStyle("MY_CONSTANT")    == hashIgnoreStyle("myConstant")

# It still differs from hash(string)
assert hashIgnoreStyle("abcdefghi") != hash("abcdefghi")
```

**Use case:** Building symbol tables, import resolution, or any system that should follow Nim's style-insensitive identifier matching.

---

### `hashIgnoreStyle` — slice

```nim
proc hashIgnoreStyle*(sBuf: string, sPos, ePos: int): Hash
```

**What it does:** Like `hashIgnoreStyle(string)` but operates on a substring. `hashIgnoreStyle(myBuf, 0, myBuf.high)` is equivalent to `hashIgnoreStyle(myBuf)`.

```nim
import std/hashes

let s = "ABracada_b_r_a"
# "ABra" (0..3) ignoring style == "b_r_a" ... let's verify:
assert hashIgnoreStyle(s, 0, 3) == hashIgnoreStyle(s, 7, s.high)
```

---

## Implementing `hash` for Custom Types

When you define a new type that you want to use as a key in a `Table` or `HashSet`, you need to implement a `hash` proc for it. There are two patterns.

### Pattern 1: Mix field hashes directly

The most common and idiomatic approach. Hash each field, mix with `!&`, finish with `!$`.

```nim
import std/hashes

type Person = object
  name: string
  age:  int

proc hash(p: Person): Hash =
  var h: Hash = 0
  h = h !& hash(p.name)
  h = h !& hash(p.age)
  result = !$h
```

### Pattern 2: Delegate to a tuple

A concise alternative that leverages the built-in tuple hash:

```nim
import std/hashes

type Color = object
  r, g, b: uint8

proc hash(c: Color): Hash =
  hash((c.r, c.g, c.b))  # hash the equivalent tuple
```

### Pattern 3: Iterate with a custom iterator

For types with variable-length or computed components:

```nim
import std/hashes

type TagSet = object
  tags: seq[string]
  priority: int

iterator items(t: TagSet): Hash =
  yield hash(t.priority)
  for tag in t.tags:
    yield hash(tag)

proc hash(t: TagSet): Hash =
  var h: Hash = 0
  for atom in t:
    h = h !& atom
  result = !$h
```

### The mandatory contract

Whenever you define `hash` for a type, you **must** ensure:

> If `a == b`, then `hash(a) == hash(b)`.

This means: if you also define a custom `==` for your type, your `hash` must be consistent with it — equal objects must have equal hashes. Violating this contract causes silent, hard-to-debug failures in hash tables.

---

## Important Rules & Pitfalls

**Rule 1 — Always finish with `!$`.**
Forgetting `!$` produces a poorly distributed hash. Every `hash` proc you write must end with `result = !$h`.

**Rule 2 — Never call `!$` mid-accumulation.**
Once you call `!$`, the value is finalised. Mixing more data in afterwards destroys the quality guarantees. `!$` must be the very last step.

**Rule 3 — `hashIgnoreCase` / `hashIgnoreStyle` are not compatible with `hash(string)`.**
These functions use a different algorithm. Do not mix them in the same hash table. If your table uses `hashIgnoreCase` keys, all lookups must also use `hashIgnoreCase`.

**Rule 4 — `hash(ref T)` requires `-d:nimPreviewHashRef`.**
Without this flag, hashing a `ref` value will not compile. The flag changes the semantic: `ref` is hashed by address (identity), not by content.

**Rule 5 — Hashes are not stable across runs or versions.**
Nim hash values are not guaranteed to be the same across different program invocations, different Nim versions, or different platforms. Do not serialize `Hash` values to disk or send them over a network.

**Rule 6 — `hash(pointer)` / `hash(ref)` hash the address, not the content.**
If you want to use a `ref` as a hash-table key based on the *identity* of the object (not its fields), use the built-in (with the preview flag). If you want content-based hashing, define a custom `hash` proc.

---

## Complete Worked Examples

### Example 1: Custom hash for a compound type

```nim
import std/[hashes, tables]

type
  Point = object
    x, y: float

proc hash(p: Point): Hash =
  var h: Hash = 0
  h = h !& hash(p.x)
  h = h !& hash(p.y)
  result = !$h

proc `==`(a, b: Point): bool =
  a.x == b.x and a.y == b.y

# Now Point can be used as a Table key
var distances: Table[Point, float]
distances[Point(x: 0.0, y: 0.0)] = 0.0
distances[Point(x: 3.0, y: 4.0)] = 5.0

echo distances[Point(x: 3.0, y: 4.0)]  # 5.0
```

---

### Example 2: Case-insensitive string table

```nim
import std/[hashes, tables]

# To build a case-insensitive map, we wrap the key so we can
# define custom hash and == on it.
type CIString = distinct string

proc hash(s: CIString): Hash =
  hashIgnoreCase(string(s))

proc `==`(a, b: CIString): bool =
  cmpIgnoreCase(string(a), string(b)) == 0

import std/strutils  # for cmpIgnoreCase

var headers: Table[CIString, string]
headers[CIString("Content-Type")]   = "application/json"
headers[CIString("Authorization")]  = "Bearer xyz"

# Lookup is case-insensitive:
echo headers[CIString("content-type")]   # "application/json"
echo headers[CIString("AUTHORIZATION")]  # "Bearer xyz"
```

---

### Example 3: Hashing raw binary data

```nim
import std/hashes

proc hashFile(path: string): Hash =
  let data = readFile(path)
  # Option A: hash the string directly (treats content as bytes)
  result = hash(data)

proc hashBuffer(buf: ptr UncheckedArray[byte], len: int): Hash =
  # Option B: use hashData for raw pointer + size
  result = hashData(buf, len)

echo hashFile("mydata.bin")
```

---

### Example 4: Verifying the hash contract

Demonstrate that equal objects must have equal hashes (and show the contract in action):

```nim
import std/hashes

type
  Fraction = object
    num, den: int  # numerator, denominator — may not be in lowest terms

# Two fractions are equal if cross-multiplication matches
proc `==`(a, b: Fraction): bool =
  a.num * b.den == b.num * a.den

# CORRECT hash: normalise to lowest terms first, then hash
proc gcd(a, b: int): int =
  var (a, b) = (a.abs, b.abs)
  while b != 0:
    a = a mod b
    swap a, b
  a

proc hash(f: Fraction): Hash =
  let g = gcd(f.num, f.den)
  var h: Hash = 0
  h = h !& hash(f.num div g)
  h = h !& hash(f.den div g)
  result = !$h

let half1 = Fraction(num: 1, den: 2)
let half2 = Fraction(num: 2, den: 4)  # same value, different representation

assert half1 == half2              # equal by definition
assert hash(half1) == hash(half2)  # contract satisfied
```

---

### Example 5: Using `hashIdentity` for a dense-key table

```nim
import std/[hashes, tables]

# Suppose we have user IDs that are dense (e.g. 0 to N-1).
# hashIdentity avoids scrambling overhead.
type UserId = distinct int

proc hash(id: UserId): Hash =
  hashIdentity(int(id))

proc `==`(a, b: UserId): bool {.borrow.}

var usernames: Table[UserId, string]
usernames[UserId(0)] = "alice"
usernames[UserId(1)] = "bob"
usernames[UserId(2)] = "carol"

echo usernames[UserId(1)]  # "bob"
```
