# `packedsets` Module Reference

> **Module purpose:** `packedsets` implements a memory-efficient, high-performance set data structure for any `Ordinal` type — integers, enums, characters, and any type whose values map to a contiguous integer range. Under the hood it uses a **sparse bit set**: membership of each element is encoded as a single bit, so the set occupies a tiny fraction of the memory a hash set would require when elements cluster in ranges, and membership tests are extremely fast (a single bit-test instruction after two index calculations). For general-purpose sets over arbitrary types, see `std/sets`.

---

## Table of Contents

1. [Type: PackedSet](#type-packedset)
2. [Internal architecture at a glance](#internal-architecture-at-a-glance)
3. [Creation](#creation)
   - [initPackedSet](#initpackedset)
   - [toPackedSet](#topackedset)
4. [Membership](#membership)
   - [contains / in / notin](#contains--in--notin)
5. [Mutation — single elements](#mutation--single-elements)
   - [incl (element)](#incl-element)
   - [excl (element)](#excl-element)
   - [containsOrIncl](#containsorincl)
   - [missingOrExcl](#missingorexcl)
6. [Mutation — whole sets](#mutation--whole-sets)
   - [incl (set)](#incl-set)
   - [excl (set)](#excl-set)
   - [clear](#clear)
7. [Queries](#queries)
   - [len / card](#len--card)
   - [isNil](#isnil)
8. [Iteration](#iteration)
   - [items](#items)
9. [Set algebra — named procs](#set-algebra--named-procs)
   - [union](#union)
   - [intersection](#intersection)
   - [difference](#difference)
   - [symmetricDifference](#symmetricdifference)
   - [disjoint](#disjoint)
10. [Set algebra — operators](#set-algebra--operators)
    - [+ (union)](#-union)
    - [* (intersection)](#-intersection)
    - [- (difference)](#--difference)
11. [Comparison operators](#comparison-operators)
    - [== (equality)](#-equality)
    - [<= (subset)](#-subset)
    - [< (proper subset)](#-proper-subset)
12. [Copying and conversion](#copying-and-conversion)
    - [=copy](#copy)
    - [assign (deprecated)](#assign-deprecated)
    - [$ (string conversion)](#-string-conversion)
13. [PackedSet vs sets — when to choose which](#packedset-vs-sets--when-to-choose-which)
14. [Practical patterns](#practical-patterns)

---

## Type: `PackedSet`

```nim
type PackedSet*[A: Ordinal] = object
```

A generic, value-type set whose element type `A` must be `Ordinal`. The set is stored as an object (not a ref), so it has value semantics — assignment copies the set. The copy operator `=copy` is explicitly defined to perform a deep copy of the internal bit-trunk structure.

The type parameter constrains you to types that have a meaningful integer representation: `int`, `int8`–`int64`, `uint`, `uint8`–`uint64`, `char`, `bool`, and any `enum`. You cannot store `string`, `float`, or custom object types.

---

## Internal architecture at a glance

`PackedSet` uses two storage strategies depending on how many elements are present:

**Small mode (≤ 34 elements):** elements are stored in a flat `array[0..33, int]` directly inside the object. No heap allocation occurs. This makes tiny sets extremely cache-friendly.

**Large mode (> 34 elements):** the set transitions to a **sparse bit set** backed by a hash table of *trunks*. Each trunk covers 512 consecutive ordinal values and stores their membership as a 512-bit array (64 bytes of `uint`). Only trunks that contain at least one member are allocated, so a set containing `{1, 1_000_000}` uses just two trunks — not a million bits of contiguous memory.

The result is O(1) membership tests and very low memory usage for both dense ranges and sparse large sets.

---

## Creation

### `initPackedSet`

```nim
proc initPackedSet*[A]: PackedSet[A]
```

Returns a new, empty `PackedSet[A]`. This is the canonical way to create a set when you intend to populate it incrementally with `incl`. The set starts in small mode with no heap allocation.

`A` must be specified explicitly, either at the call site or by type inference from the surrounding context.

```nim
import std/packedsets

# Explicit type parameter
let empty = initPackedSet[int]()
assert empty.len == 0

# Works for enums too
type Color = enum Red, Green, Blue
var colors = initPackedSet[Color]()
colors.incl(Red)
colors.incl(Blue)
assert colors.len == 2

# Works for distinct types
type UserId = distinct int
var ids = initPackedSet[UserId]()
ids.incl(42.UserId)
```

---

### `toPackedSet`

```nim
proc toPackedSet*[A](x: openArray[A]): PackedSet[A]
```

Constructs a `PackedSet[A]` from any array or sequence of ordinal values. Duplicates in the input are automatically discarded — the resulting set contains each value at most once. This is the most concise way to create a set with known initial contents.

```nim
import std/packedsets

# Duplicates are silently removed
let a = [5, 6, 7, 8, 8].toPackedSet
assert a.len == 4
assert $a == "{5, 6, 7, 8}"

# Works with sequences
let primes = @[2, 3, 5, 7, 11, 13].toPackedSet
assert 7 in primes
assert 9 notin primes

# Works with enum values
type Dir = enum North, South, East, West
let horizontal = [East, West].toPackedSet
assert East in horizontal
assert North notin horizontal
```

---

## Membership

### `contains` / `in` / `notin`

```nim
proc contains*[A](s: PackedSet[A], key: A): bool
```

Returns `true` if `key` is a member of `s`. The `in` and `notin` operators are Nim's syntactic sugar for `contains` and `not contains` respectively, and work automatically because `contains` is defined.

Membership testing is O(1): in small mode it scans up to 34 elements; in large mode it performs two index calculations and one bit-test.

```nim
import std/packedsets

let a = [1, 3, 5].toPackedSet

# All three forms are equivalent
assert a.contains(3)
assert 3 in a
assert 5 notin a == false

# Useful in conditionals
if 3 in a:
  echo "3 is in the set"

# Enum example
type ABCD = enum A, B, C, D
let letters = [A, C].toPackedSet
assert A in letters
assert B notin letters
```

---

## Mutation — single elements

### `incl` (element)

```nim
proc incl*[A](s: var PackedSet[A], key: A)
```

Adds `key` to `s`. If `key` is already present, the call is a no-op — the set is unchanged and no error is raised. After the call, `key in s` is guaranteed to be `true`.

```nim
import std/packedsets

var a = initPackedSet[int]()
a.incl(3)
a.incl(3)   # second call has no effect
a.incl(7)
assert a.len == 2
assert 3 in a
assert 7 in a
```

---

### `excl` (element)

```nim
proc excl*[A](s: var PackedSet[A], key: A)
```

Removes `key` from `s`. If `key` is not present, the call is a no-op — no error is raised. After the call, `key in s` is guaranteed to be `false`.

```nim
import std/packedsets

var a = [3, 7].toPackedSet
a.excl(3)
a.excl(3)   # second call has no effect
a.excl(99)  # element not in set — no error
assert a.len == 1
assert 3 notin a
assert 7 in a
```

---

### `containsOrIncl`

```nim
proc containsOrIncl*[A](s: var PackedSet[A], key: A): bool
```

A combined check-and-insert operation. It adds `key` to `s` if it was absent, and returns a boolean indicating whether `key` was already present **before** this call:

- Returns `false` → `key` was **not** in `s`; it has been added now.
- Returns `true`  → `key` **was** already in `s`; the set is unchanged.

This is more efficient than a separate `contains` followed by `incl` because it avoids traversing the internal structure twice.

```nim
import std/packedsets

var a = initPackedSet[int]()

let wasPresent1 = a.containsOrIncl(3)
assert wasPresent1 == false   # 3 was absent, now added
assert 3 in a

let wasPresent2 = a.containsOrIncl(3)
assert wasPresent2 == true    # 3 was already present, nothing changed

let wasPresent3 = a.containsOrIncl(4)
assert wasPresent3 == false   # 4 was absent, now added
assert a.len == 2
```

---

### `missingOrExcl`

```nim
proc missingOrExcl*[A](s: var PackedSet[A], key: A): bool
```

A combined check-and-remove operation — the mirror image of `containsOrIncl`. It removes `key` from `s` if it was present, and returns a boolean indicating whether `key` was already **missing** before this call:

- Returns `false` → `key` **was** in `s`; it has been removed now.
- Returns `true`  → `key` was **not** in `s`; the set is unchanged.

```nim
import std/packedsets

var a = [5].toPackedSet

let wasMissing1 = a.missingOrExcl(5)
assert wasMissing1 == false   # 5 was present, now removed
assert 5 notin a

let wasMissing2 = a.missingOrExcl(5)
assert wasMissing2 == true    # 5 was already absent
```

---

## Mutation — whole sets

### `incl` (set)

```nim
proc incl*[A](s: var PackedSet[A], other: PackedSet[A])
```

Adds all elements from `other` into `s` in-place. Elements already in `s` are unaffected. After the call `s` is the union of the original `s` and `other`. This is the mutating counterpart of the `+` operator.

```nim
import std/packedsets

var a = [1, 2].toPackedSet
let b = [2, 3, 4].toPackedSet
a.incl(b)
assert a == [1, 2, 3, 4].toPackedSet
```

---

### `excl` (set)

```nim
proc excl*[A](s: var PackedSet[A], other: PackedSet[A])
```

Removes all elements of `other` from `s` in-place. Elements in `other` that are not in `s` are ignored. After the call `s` contains only elements that were in `s` but not in `other`. This is the mutating counterpart of the `-` operator.

```nim
import std/packedsets

var a = [1, 2, 3, 4].toPackedSet
let b = [2, 4].toPackedSet
a.excl(b)
assert a == [1, 3].toPackedSet
```

---

### `clear`

```nim
proc clear*[A](result: var PackedSet[A])
```

Resets `s` to the empty state, freeing all internal trunk allocations. After this call the set behaves exactly as if it had been freshly created with `initPackedSet`. More efficient than assigning a new empty set when you want to reuse the variable.

```nim
import std/packedsets

var a = [5, 7, 100, 200].toPackedSet
assert a.len == 4
a.clear()
assert a.len == 0
assert a.isNil
```

---

## Queries

### `len` / `card`

```nim
proc len*[A](s: PackedSet[A]): int
proc card*[A](s: PackedSet[A]): int   # alias for len
```

Returns the number of elements in `s`. `card` is a mathematical synonym (from *cardinality* of a set) that some people prefer when thinking in terms of set theory.

In small mode `len` is O(1). In large mode it iterates over all set bits, so it is O(n) where n is the number of elements. If you need to check emptiness, `isNil` is faster.

```nim
import std/packedsets

let a = [1, 3, 5].toPackedSet
assert len(a) == 3
assert card(a) == 3   # same thing

let empty = initPackedSet[int]()
assert len(empty) == 0
```

---

### `isNil`

```nim
proc isNil*[A](x: PackedSet[A]): bool
```

Returns `true` if the set is empty (contains no elements), `false` otherwise. Despite the name borrowed from pointer semantics, `PackedSet` is a value type — `isNil` simply means "empty". This check is O(1) and faster than `len(s) == 0` for large sets.

```nim
import std/packedsets

var a = initPackedSet[int]()
assert a.isNil          # freshly created — empty

a.incl(42)
assert not a.isNil      # has an element

a.excl(42)
assert a.isNil          # back to empty
```

---

## Iteration

### `items`

```nim
iterator items*[A](s: PackedSet[A]): A
```

Yields every element of `s` exactly once. The iteration order is **ascending by ordinal value** — elements are always yielded smallest first, regardless of the order they were inserted.

**Important:** do not modify the set (call `incl` or `excl`) while iterating over it. The internal structure can be invalidated by concurrent mutation, leading to missed elements or incorrect results.

```nim
import std/packedsets

let a = [5, 1, 3, 2, 4].toPackedSet

# Elements always come out sorted ascending
for x in a:
  echo x   # prints 1, 2, 3, 4, 5

# Collect into a seq
var sorted: seq[int]
for x in a:
  sorted.add(x)
assert sorted == @[1, 2, 3, 4, 5]

# Use in a for-in expression
import std/sequtils
assert toSeq(a.items) == @[1, 2, 3, 4, 5]
```

---

## Set algebra — named procs

### `union`

```nim
proc union*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

Returns a new set containing every element that is in `s1`, `s2`, or both. Neither `s1` nor `s2` is modified. Equivalent to the `+` operator.

```nim
import std/packedsets

let a = [1, 2, 3].toPackedSet
let b = [3, 4, 5].toPackedSet
let c = union(a, b)
assert c == [1, 2, 3, 4, 5].toPackedSet
```

---

### `intersection`

```nim
proc intersection*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

Returns a new set containing only elements that appear in **both** `s1` and `s2`. Equivalent to the `*` operator.

```nim
import std/packedsets

let a = [1, 2, 3].toPackedSet
let b = [3, 4, 5].toPackedSet
let c = intersection(a, b)
assert c == [3].toPackedSet
```

---

### `difference`

```nim
proc difference*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

Returns a new set containing elements that are in `s1` but **not** in `s2`. The order of operands matters: `difference(a, b) ≠ difference(b, a)`. Equivalent to the `-` operator.

```nim
import std/packedsets

let a = [1, 2, 3].toPackedSet
let b = [3, 4, 5].toPackedSet
assert difference(a, b) == [1, 2].toPackedSet
assert difference(b, a) == [4, 5].toPackedSet
```

---

### `symmetricDifference`

```nim
proc symmetricDifference*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

Returns a new set containing elements that are in **exactly one** of `s1` or `s2` — everything in the union that is not in the intersection. The operation is commutative: `symmetricDifference(a, b) == symmetricDifference(b, a)`.

```nim
import std/packedsets

let a = [1, 2, 3].toPackedSet
let b = [3, 4, 5].toPackedSet
let c = symmetricDifference(a, b)
assert c == [1, 2, 4, 5].toPackedSet   # 3 is in both, so excluded
```

---

### `disjoint`

```nim
proc disjoint*[A](s1, s2: PackedSet[A]): bool
```

Returns `true` if `s1` and `s2` share no elements — their intersection is empty. Returns `false` as soon as a common element is found, making it efficient for early-exit checks.

```nim
import std/packedsets

let a = [1, 2].toPackedSet
let b = [2, 3].toPackedSet
let c = [3, 4].toPackedSet

assert disjoint(a, c) == true    # no common elements
assert disjoint(a, b) == false   # 2 is shared
```

---

## Set algebra — operators

The operators are thin, `inline` aliases for the named procs. They exist purely for readability when writing set expressions.

### `+` (union)

```nim
proc `+`*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

```nim
import std/packedsets
let result = [1, 2].toPackedSet + [2, 3].toPackedSet
assert result == [1, 2, 3].toPackedSet
```

---

### `*` (intersection)

```nim
proc `*`*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

```nim
import std/packedsets
let result = [1, 2, 3].toPackedSet * [2, 3, 4].toPackedSet
assert result == [2, 3].toPackedSet
```

---

### `-` (difference)

```nim
proc `-`*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

```nim
import std/packedsets
let result = [1, 2, 3].toPackedSet - [2].toPackedSet
assert result == [1, 3].toPackedSet
```

---

## Comparison operators

### `==` (equality)

```nim
proc `==`*[A](s1, s2: PackedSet[A]): bool
```

Returns `true` if both sets contain exactly the same elements. Order of insertion is irrelevant. Implemented as mutual subset testing (`s1 <= s2 and s2 <= s1`), which avoids the need to sort or hash.

```nim
import std/packedsets

assert [1, 2].toPackedSet == [2, 1].toPackedSet       # insertion order irrelevant
assert [1, 2].toPackedSet == [1, 2, 1].toPackedSet    # duplicates in source irrelevant
assert [1, 2].toPackedSet != [1, 2, 3].toPackedSet
```

---

### `<=` (subset)

```nim
proc `<=`*[A](s1, s2: PackedSet[A]): bool
```

Returns `true` if `s1` is a **subset** of `s2`: every element of `s1` is also in `s2`. An equal set is considered a subset of itself (reflexive), so `s <= s` is always `true`.

```nim
import std/packedsets

let a = [1].toPackedSet
let b = [1, 2].toPackedSet

assert a <= b    # a ⊆ b
assert b <= b    # a set is a subset of itself
assert not (b <= a)
```

---

### `<` (proper subset)

```nim
proc `<`*[A](s1, s2: PackedSet[A]): bool
```

Returns `true` if `s1` is a **proper (strict) subset** of `s2`: every element of `s1` is in `s2`, and `s2` has at least one element that is not in `s1`. A set is never a proper subset of itself.

```nim
import std/packedsets

let a = [1].toPackedSet
let b = [1, 2].toPackedSet

assert a < b       # a is a proper subset of b
assert not (b < b) # a set is not a proper subset of itself
assert not (b < a)
```

---

## Copying and conversion

### `=copy`

```nim
proc `=copy`*[A](dest: var PackedSet[A], src: PackedSet[A])
```

The copy hook that Nim calls automatically on assignment (`dest = src`). It performs a **deep copy**: the destination gets its own independent copy of all internal trunks. Modifying `dest` after the copy does not affect `src`.

You do not normally call this directly — it is invoked implicitly whenever a `PackedSet` is assigned:

```nim
import std/packedsets

var a = [1, 2, 3].toPackedSet
var b = a          # =copy called here automatically

b.incl(99)
assert 99 notin a  # a is unaffected — truly independent copy
assert 99 in b
```

---

### `assign` (deprecated)

```nim
proc assign*[A](dest: var PackedSet[A], src: PackedSet[A]) {.deprecated.}
```

An older, explicit copy proc that predates Nim's copy hook system. It is now deprecated in favour of direct assignment (`dest = src`), which triggers `=copy` automatically. Included here for completeness; do not use in new code.

---

### `$` (string conversion)

```nim
proc `$`*[A](s: PackedSet[A]): string
```

Converts `s` to a human-readable string in the form `{e1, e2, e3}`, with elements listed in **ascending** ordinal order. Useful for debugging and logging.

```nim
import std/packedsets

let a = [3, 1, 2].toPackedSet
echo a          # {1, 2, 3}  — always sorted ascending
assert $a == "{1, 2, 3}"

let empty = initPackedSet[int]()
assert $empty == "{}"
```

---

## `PackedSet` vs `sets` — when to choose which

| Criterion | `PackedSet` | `HashSet` (std/sets) |
|---|---|---|
| Element type constraint | `Ordinal` only | Any hashable type |
| Element types supported | int, enum, char, bool, distinct ordinals | string, object, float, anything with `hash` |
| Memory (dense range, e.g. 0–999) | ~128 bytes (2 trunks) | ~8 KB (hash table) |
| Memory (sparse, e.g. 2 random ints) | 2 trunks + overhead | ~128 bytes |
| Membership test | O(1), branch-free bit op | O(1), hash + equality |
| Iteration order | Ascending ordinal | Arbitrary |
| Small set (≤ 34 elements) | Array, zero heap allocation | Hash table allocated |

**Choose `PackedSet` when:** elements are ordinal types clustered in a range, memory efficiency matters, or you need deterministic ascending iteration order.

**Choose `HashSet` when:** elements are strings, floats, or custom objects, or when the ordinal range is so sparse that `PackedSet`'s trunk overhead outweighs `HashSet`'s bucket cost.

---

## Practical patterns

### Pattern 1 — Visited node tracking in a graph traversal

```nim
import std/packedsets

type NodeId = int

proc bfs(start: NodeId, edges: seq[(NodeId, NodeId)]): seq[NodeId] =
  var visited = initPackedSet[NodeId]()
  var queue = @[start]
  while queue.len > 0:
    let node = queue[0]
    queue.delete(0)
    if containsOrIncl(visited, node):
      continue   # already visited
    result.add(node)
    for (a, b) in edges:
      if a == node and b notin visited: queue.add(b)
```

---

### Pattern 2 — Fast set operations on enum flags

```nim
import std/packedsets

type Permission = enum Read, Write, Execute, Admin

let userPerms  = [Read, Write].toPackedSet
let adminPerms = [Read, Write, Execute, Admin].toPackedSet
let filePerms  = [Read, Execute].toPackedSet

# What can this user do with this file?
let effective = userPerms * filePerms   # intersection
assert effective == [Read].toPackedSet

# Does the user have all admin permissions?
assert not (adminPerms <= userPerms)

# What permissions is the user missing?
let missing = adminPerms - userPerms
assert missing == [Execute, Admin].toPackedSet
```

---

### Pattern 3 — Deduplication of a large integer list

```nim
import std/packedsets

# toPackedSet removes duplicates far more efficiently than sorting + unique
# for ordinal types in a bounded range.
let rawIds = @[5, 3, 8, 3, 1, 5, 9, 1]
let uniqueIds = rawIds.toPackedSet

assert uniqueIds.len == 5
# Iteration yields them sorted:
import std/sequtils
assert toSeq(uniqueIds.items) == @[1, 3, 5, 8, 9]
```

---

### Pattern 4 — Symmetric difference to find changed elements

```nim
import std/packedsets

let before = [1, 2, 3, 4].toPackedSet
let after  = [2, 3, 4, 5].toPackedSet

let changed = symmetricDifference(before, after)
# changed contains elements that were added or removed
assert changed == [1, 5].toPackedSet

let added   = after - before
let removed = before - after
assert added   == [5].toPackedSet
assert removed == [1].toPackedSet
```

---

### Pattern 5 — Accumulating results with `containsOrIncl`

```nim
import std/packedsets

# Count how many unique values appear in a stream
proc countUnique(values: seq[int]): int =
  var seen = initPackedSet[int]()
  for v in values:
    if not containsOrIncl(seen, v):
      inc result

assert countUnique(@[1, 1, 2, 3, 2, 4]) == 4
```
