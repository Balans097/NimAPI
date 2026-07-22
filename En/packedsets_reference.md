# packedsets — module reference

> **Import:** `import std/packedsets`
> **Scope:** an efficient set of values of an arbitrary `Ordinal` type, implemented as a sparse bit set.

The `packedsets` module solves the same problem as `std/sets` (storing unique values with fast membership testing), but is aimed at dense ranges of ordinal values — integers, `enum`, `char`, `distinct int`, and the like. Instead of a hash table of individual values, the module uses a hybrid representation: small sets (up to 34 elements) are stored in a linear array without a single heap allocation, and once that threshold is exceeded the structure transparently switches to a sparse bit set — a hash table of "trunks" (`Trunk`), each of which covers 512 consecutive values with a bit vector. This lets the module save memory in places where `std/sets` would spend it on hashing individual numbers.

Throughout this reference a single set of terms is used: **small-set mode** (up to 34 elements, the linear array `a`) and **bitset mode** (`Trunk` blocks with bit vectors). The switch between modes is one-way: once the threshold is crossed, the set never returns to small-set mode, even if elements are later removed from it.

---

## Table of Contents

I. The `PackedSet` type and creating a set
   1. [`PackedSet[A]`](#packedseta)
   2. [`initPackedSet`](#initpackedset)
   3. [`toPackedSet`](#topackedset)

II. Testing and adding single elements
   1. [`contains` / `in`](#contains--in)
   2. [`incl` (element)](#incl-element)
   3. [`containsOrIncl`](#containsorincl)

III. Removing single elements
   1. [`excl` (element)](#excl-element)
   2. [`missingOrExcl`](#missingorexcl)

IV. Operations on the set as a whole
   1. [`incl` (set)](#incl-set)
   2. [`excl` (set)](#excl-set)
   3. [`clear`](#clear)
   4. [`isNil`](#isnil)
   5. [`` `=copy` `` / `assign`](#copy--assign)

V. Set-theoretic operations
   1. [`union` / `+`](#union--)
   2. [`intersection` / `*`](#intersection--)
   3. [`difference` / `-`](#difference---)
   4. [`symmetricDifference`](#symmetricdifference)

VI. Comparing sets
   1. [`<=` (subset)](#-subset)
   2. [`<` (proper subset)](#-proper-subset)
   3. [`==` (equality)](#-equality)
   4. [`disjoint`](#disjoint)
   5. [`card`](#card)

VII. Iteration and output
   1. [`items`](#items)
   2. [`` `$` ``](#)

VIII. [Practical recipes](#practical-recipes)

IX. [Quick reference table](#quick-reference-table)

X. [Summary: which procedure to choose](#summary-which-procedure-to-choose)

---

## I. The `PackedSet` type and creating a set

### `PackedSet[A]`

```nim
type
  PackedSet*[A: Ordinal] = object
```

**What it does.** A generic set type parameterized over an ordinal type `A` (`int`, `enum`, `char`, `distinct int`, and so on). A value of this type is stored by value (not by reference), but internally it may contain reference-based nodes — which is why the module has to define its own `` `=copy` `` hook (see section IV.5), so that assigning `b = a` doesn't leave the two values dangerously sharing internal structures.

**Implementation notes.** The type combines two representations in one structure:

- the field `a: array[0..33, int]` together with the counter `elems` — a "built-in" linear buffer for 34 values. As long as `elems <= 34`, every operation works against this array by simple scanning, without any heap allocation;
- the fields `head: Trunk`, `data: seq[Trunk]`, `counter`, `max` — the sparse bit set. `data` is an open-addressed hash table (using the same probing scheme as Python dictionaries: `((5*h) + 1 + perturb) and maxHash`), keyed by `key shr TrunkShift`, i.e. the index of the group of 512 consecutive values. Each `Trunk` holds a bit vector `bits` of `IntsPerTrunk` machine words, where each bit corresponds to one value within the group. `head` is a separate singly linked list of the same blocks, used only for sequential traversal (see `items`), not for lookup.

The 34-element threshold was chosen, according to a comment in the source, based on profiling — a compromise between the size of the structure on the stack and the cost of switching to the hash table.

**Parameters.**
- `A: Ordinal` — the element type of the set; must support `ord()`.

**Examples.**

```nim
type Id = distinct int

var
  ints: PackedSet[int]
  ids: PackedSet[Id]
```

---

### `initPackedSet`

```nim
proc initPackedSet*[A]: PackedSet[A]
```

**What it does.** Returns an empty set of type `PackedSet[A]`, ready to use immediately after declaration via `var`, with no separate initialization step.

**Implementation notes.** Explicitly zeroes all fields of the bitset mode (`elems`, `counter`, `max`, `head`, `data`); the small-buffer field `a` is left unfilled — it is meaningless as long as `elems == 0`.

**Parameters.** None (the type `A` is inferred from the usage context or specified explicitly, `initPackedSet[int]()`).

**Examples.**

```nim
let a = initPackedSet[int]()
assert len(a) == 0

type Id = distinct int
var ids = initPackedSet[Id]()
incl(ids, 3.Id)
assert len(ids) == 1
```

---

### `toPackedSet`

```nim
proc toPackedSet*[A](x: openArray[A]): PackedSet[A] {.since: (1, 3).}
```

**What it does.** Builds a new set from an array, `seq`, or any other `openArray`. Duplicate values are automatically collapsed into a single occurrence.

**Implementation notes.** A thin wrapper: it creates an empty set via `initPackedSet` and includes each element of `x` in turn by calling `incl`. Cost — O(n) calls to `incl`, where n is the length of `x` (the insertion itself is amortized-fast, see the notes on `incl`).

**Parameters.**
- `x: openArray[A]` — the source sequence of values; left unchanged.

**Examples.**

```nim
let a = toPackedSet([5, 6, 7, 8, 8])
assert len(a) == 4          # the duplicate "8" is counted once
assert $a == "{5, 6, 7, 8}"

let empty = toPackedSet(newSeq[int]())  # boundary case: an empty array
assert len(empty) == 0
```

---

## II. Testing and adding single elements

### `contains` / `in`

```nim
proc contains*[A](s: PackedSet[A], key: A): bool
```

**What it does.** Checks whether `key` belongs to the set `s`. Used both as an ordinary procedure and through the `in` / `notin` operators.

**Implementation notes.** Branches on the current storage mode:

- in small-set mode (`elems <= len(s.a)`) — a linear scan of the array `a`, comparing `ord(key)` against each element, O(k) where k ≤ 34;
- in bitset mode — the block index `ord(key) shr TrunkShift` is computed, the block is looked up in the hash table `data` using the same probing as during insertion, and then a single bit `ord(key) and TrunkMask` is tested inside the block. O(1) on average.

**Parameters.**
- `s: PackedSet[A]` — the set being tested; left unchanged.
- `key: A` — the value being tested.

**Examples.**

```nim
type ABCD = enum A, B, C, D

let a = toPackedSet([1, 3, 5])
assert contains(a, 3)
assert 3 in a
assert not contains(a, 8)
assert 8 notin a

let letters = toPackedSet([A, C])
assert A in letters
assert C in letters
assert B notin letters

let empty = initPackedSet[int]()   # boundary case: an empty set
assert not contains(empty, 0)
```

---

### `incl` (element)

```nim
proc incl*[A](s: var PackedSet[A], key: A)
```

**What it does.** Adds `key` to the set. If `key` is already present, nothing happens (the operation is idempotent).

**Implementation notes.** This is the key procedure responsible for the transition between storage modes:

1. As long as `elems <= len(s.a)` (34), a linear search runs first: if the value is already in the buffer, the procedure returns without any change.
2. If the value is absent and the buffer still has room (`elems < len(s.a)`), the value is appended to the end of the buffer and `elems` is incremented.
3. If the buffer is already full, a one-time transition to bitset mode occurs: a table `data` of initial size `InitIntSetSize` (8) is allocated, and all 34 already-accumulated values from `a` are migrated into it through the internal procedure `bitincl`. Only after this "fall through" (as the source comment calls it) is the new `key` itself added to the set.

Inside bitset mode, insertion (`intSetPut`) doubles the table's size when needed (`intSetEnlarge`) — once the load factor exceeds 2/3 or fewer than four free slots remain — a classic open-addressing rehashing strategy.

**Parameters.**
- `s: var PackedSet[A]` — the mutable set.
- `key: A` — the value being added.

**Examples.**

```nim
var a = initPackedSet[int]()
incl(a, 3)
incl(a, 3)                 # inserting again — creates no duplicate
assert len(a) == 1

var big = initPackedSet[int]()
for i in 0..<100:           # boundary case: crossing the 34-element threshold
  incl(big, i)
assert len(big) == 100      # the switch to bitset mode happened transparently
```

---

### `containsOrIncl`

```nim
proc containsOrIncl*[A](s: var PackedSet[A], key: A): bool
```

**What it does.** Combines a check and an insertion in a single pass: includes `key` in the set and returns whether the value was already there **before** the call. Lets you avoid a double lookup (`contains` + `incl`) in code that needs both pieces of information.

**Implementation notes.** Mirrors the branching of `incl`, but along the way records and returns the result of the test: in small-set mode, from the outcome of the linear scan; in bitset mode, from the value of the bit before it is set. Unlike `incl`, on the first crossing of the 34-element threshold this procedure delegates the insertion to ordinary `incl` (i.e. it doesn't duplicate the rehashing logic).

**Parameters.**
- `s: var PackedSet[A]` — the mutable set.
- `key: A` — the value being tested and added.

**Examples.**

```nim
var a = initPackedSet[int]()
assert containsOrIncl(a, 3) == false   # the value wasn't there — it was added
assert containsOrIncl(a, 3) == true    # the value was already there
assert containsOrIncl(a, 4) == false
assert len(a) == 2
```

---

## III. Removing single elements

### `excl` (element)

```nim
proc excl*[A](s: var PackedSet[A], key: A)
```

**What it does.** Removes `key` from the set. If the value wasn't there, the operation does nothing and raises no exception.

**Implementation notes.** In small-set mode, removal is done via swap-remove: the found element is replaced with the last active element of the array and `elems` is decremented — O(k), without preserving insertion order. In bitset mode, the corresponding bit is simply cleared inside the found `Trunk` block — the block itself is not removed from either the linked list `head` or the table `data`, even if all its bits become zero. This has an important consequence: **memory once allocated for bitset mode is never shrunk** — neither by removing individual elements nor by emptying an entire block.

**Parameters.**
- `s: var PackedSet[A]` — the mutable set.
- `key: A` — the value being removed.

**Examples.**

```nim
var a = toPackedSet([3])
excl(a, 3)
excl(a, 3)                 # removing again is not an error
excl(a, 99)                # removing an absent value is not an error
assert len(a) == 0
```

---

### `missingOrExcl`

```nim
proc missingOrExcl*[A](s: var PackedSet[A], key: A): bool
```

**What it does.** Removes `key` from the set and reports whether it was already **missing** before the call. Note the direction, in contrast with `containsOrIncl`: here `true` means "the value was absent," not "the value was present."

**Implementation notes.** Measures `len(s)` before and after the removal and compares them: if the length didn't change, the element was absent and there was nothing to remove. An important performance caveat: in bitset mode, `len` is not kept as a separate counter but is computed with a full pass over `items` (see section VII.1) — meaning `missingOrExcl` costs O(n) in this mode, rather than the O(1) one might expect from a simple counter comparison.

**Parameters.**
- `s: var PackedSet[A]` — the mutable set.
- `key: A` — the value being tested and removed.

**Examples.**

```nim
var a = toPackedSet([5])
assert missingOrExcl(a, 5) == false   # the value was present and got removed
assert missingOrExcl(a, 5) == true    # the value was already absent
```

---

## IV. Operations on the set as a whole

### `incl` (set)

```nim
proc incl*[A](s: var PackedSet[A], other: PackedSet[A])
```

**What it does.** Adds every element of `other` into `s` — the in-place version of the union operation (see `union`).

**Implementation notes.** A simple traversal of `other` via `items`, calling the single-element `incl(s, item)` on each iteration — O(m) insertions, where m = `len(other)`.

**Parameters.**
- `s: var PackedSet[A]` — the set that elements are added into.
- `other: PackedSet[A]` — the source of elements; left unchanged.

**Examples.**

```nim
var a = toPackedSet([1])
incl(a, toPackedSet([5]))
assert len(a) == 2
assert 5 in a
```

---

### `excl` (set)

```nim
proc excl*[A](s: var PackedSet[A], other: PackedSet[A])
```

**What it does.** Removes from `s` every element present in `other` (the in-place version of the difference operation, see `difference`).

**Implementation notes.** Mirrors the previous procedure: a traversal of `other` via `items`, calling the single-element `excl(s, item)`.

**Parameters.**
- `s: var PackedSet[A]` — the set that elements are removed from.
- `other: PackedSet[A]` — the set of values to remove; left unchanged.

**Examples.**

```nim
var a = toPackedSet([1, 5])
excl(a, toPackedSet([5]))
assert len(a) == 1
assert 5 notin a
```

---

### `clear`

```nim
proc clear*[A](result: var PackedSet[A])
```

**What it does.** Returns the set to an empty state, releasing the internal bitset-mode structures (unlike `excl`, here the memory is actually released).

**Implementation notes.** Assigns every field its original empty value: `data` becomes an empty `seq`, `max` and `counter` are zeroed, `head` becomes `nil`, `elems` becomes `0`. The source contains a commented-out older implementation (based on `setLen` with manual zeroing of the `data` slots) — the current version is simpler and fully recreates the structures on the next insertion.

**Parameters.**
- `result: var PackedSet[A]` — the set being cleared (the parameter is named `result` in the module's signature; this is an ordinary parameter, not a special value).

**Examples.**

```nim
var a = toPackedSet([5, 7])
clear(a)
assert len(a) == 0
```

---

### `isNil`

```nim
proc isNil*[A](x: PackedSet[A]): bool
```

**What it does.** Returns `true` if the set is empty. The name is kept for historical reasons (from an older pointer-based implementation) — `PackedSet` is a value type, and this has nothing to do with a `nil` pointer in the usual sense.

**Implementation notes.** The condition is `x.head == nil` **and** `x.elems == 0`. Both conditions are needed together: in small-set mode `head` is always `nil` (no blocks have been created yet), so emptiness is checked via `elems`; in bitset mode `elems` is no longer meaningful, and emptiness is determined via `head`. This leads to an important boundary case: if a set has once switched to bitset mode and all of its elements were later removed via `excl`, the `Trunk` block stays in the linked list `head` (see the notes on `excl`) — so `isNil` will return `false` even when `len(x) == 0`. For large sets it is more reliable to test emptiness via `len(x) == 0` rather than `isNil`.

**Parameters.**
- `x: PackedSet[A]` — the set being tested.

**Examples.**

```nim
var a = initPackedSet[int]()
assert isNil(a)
incl(a, 2)
assert not isNil(a)
excl(a, 2)
assert isNil(a)             # true, as long as the set never left small-set mode
```

---

### `` `=copy` `` / `assign`

```nim
proc `=copy`*[A](dest: var PackedSet[A], src: PackedSet[A])
proc assign*[A](dest: var PackedSet[A], src: PackedSet[A]) {.deprecated.}
```

**What it does.** Copies the contents of `src` into `dest` as an independent copy. `assign` is a deprecated alias kept for backward compatibility, and internally it simply calls `` `=copy` ``.

**Implementation notes.** This is an overridden Nim assignment hook (`=copy`), which the compiler calls automatically on a plain `dest = src`, unless `dest` already is `src` itself. The need for it comes from the type's internal structure: in bitset mode, `PackedSet` contains reference-based `Trunk` nodes, and a naive bitwise copy would leave `dest` and `src` sharing the same mutable blocks — a change to one would affect the other. The implementation therefore branches:

- if `src` is in small-set mode, it's enough to copy the plain fields and the array `a` (the array is copied by value, no memory is shared);
- if `src` is in bitset mode, a new table `data` of the same size is allocated, and every `Trunk` node from the linked list `src.head` is cloned anew (`new(n)`, copying `key` and `bits`) and re-inserted into `dest.data` using the same probing algorithm as an ordinary insertion.

**Parameters.**
- `dest: var PackedSet[A]` — the destination; does not need to be pre-initialized via `initPackedSet`.
- `src: PackedSet[A]` — the source; left unchanged.

**Examples.**

```nim
var
  a = initPackedSet[int]()
  b = initPackedSet[int]()
incl(b, 5)
incl(b, 7)
assign(a, b)                # equivalent to "a = b" — =copy is invoked automatically
assert len(a) == 2

incl(a, 9)
assert 9 notin b             # the copy is independent — changing a didn't affect b
```

---

## V. Set-theoretic operations

### `union` / `+`

```nim
proc union*[A](s1, s2: PackedSet[A]): PackedSet[A]
proc `+`*[A](s1, s2: PackedSet[A]): PackedSet[A] {.inline.}
```

**What it does.** Returns a new set containing every element of both `s1` and `s2`. The operator `+` is an exact synonym for `union`.

**Implementation notes.** `result` is initialized as an independent copy of `s1` (via `assign` — copying, not shared ownership, matters here), after which the elements of `s2` are included into it via `incl` (set). Cost — O(|s1| + |s2|).

**Parameters.**
- `s1, s2: PackedSet[A]` — the source sets; left unchanged.

**Examples.**

```nim
let
  a = toPackedSet([1, 2, 3])
  b = toPackedSet([3, 4, 5])
  c = union(a, b)
assert len(c) == 5
assert c == toPackedSet([1, 2, 3, 4, 5])
assert (a + b) == c          # the operator synonym
```

---

### `intersection` / `*`

```nim
proc intersection*[A](s1, s2: PackedSet[A]): PackedSet[A]
proc `*`*[A](s1, s2: PackedSet[A]): PackedSet[A] {.inline.}
```

**What it does.** Returns a new set of elements present in both `s1` and `s2` at once.

**Implementation notes.** Starts from an empty set and iterates over the elements of `s1`, adding to the result only those also found in `s2` via `contains`. Cost — O(|s1|) membership tests.

**Parameters.**
- `s1, s2: PackedSet[A]` — the source sets; left unchanged.

**Examples.**

```nim
let
  a = toPackedSet([1, 2, 3])
  b = toPackedSet([3, 4, 5])
  c = intersection(a, b)
assert len(c) == 1
assert c == toPackedSet([3])
```

---

### `difference` / `-`

```nim
proc difference*[A](s1, s2: PackedSet[A]): PackedSet[A]
proc `-`*[A](s1, s2: PackedSet[A]): PackedSet[A] {.inline.}
```

**What it does.** Returns a new set of elements of `s1` that are absent from `s2`.

**Implementation notes.** Mirrors `intersection`: a traversal of `s1`, but the result includes elements for which `contains(s2, item)` is false.

**Parameters.**
- `s1, s2: PackedSet[A]` — the source sets; left unchanged.

**Examples.**

```nim
let
  a = toPackedSet([1, 2, 3])
  b = toPackedSet([3, 4, 5])
  c = difference(a, b)
assert len(c) == 2
assert c == toPackedSet([1, 2])
```

---

### `symmetricDifference`

```nim
proc symmetricDifference*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

**What it does.** Returns the elements that belong to exactly one of the two sets (the set-theoretic analogue of exclusive-or).

**Implementation notes.** A trick that saves a separate pass: `result` starts as a copy of `s1`, and then `containsOrIncl(result, item)` is called for each element of `s2`. If the element was already in `result` (meaning it was in both `s1` and `s2`), it is immediately removed via `excl`, since shared elements should not appear in the symmetric difference. If the element was added for the first time (meaning it was only in `s2`), it stays in the result. The net effect, in a single pass over `s2`, combines both the union step and the removal of the shared part.

**Parameters.**
- `s1, s2: PackedSet[A]` — the source sets; left unchanged.

**Examples.**

```nim
let
  a = toPackedSet([1, 2, 3])
  b = toPackedSet([3, 4, 5])
  c = symmetricDifference(a, b)
assert len(c) == 4
assert c == toPackedSet([1, 2, 4, 5])   # "3" is in both and gets excluded
```

---

## VI. Comparing sets

### `<=` (subset)

```nim
proc `<=`*[A](s1, s2: PackedSet[A]): bool
```

**What it does.** Returns `true` if every element of `s1` is also present in `s2`. Equal sets are considered subsets of each other.

**Implementation notes.** A traversal of `s1` checking `contains(s2, item)` at each step; on the first mismatch the procedure returns `false` right away (short-circuiting) — O(|s1|) in the worst case, but usually faster.

**Parameters.**
- `s1, s2: PackedSet[A]` — the sets being compared.

**Examples.**

```nim
let
  a = toPackedSet([1])
  b = toPackedSet([1, 2])
  c = toPackedSet([1, 3])
assert a <= b
assert b <= b                # a set is a subset of itself
assert not (c <= b)
```

---

### `<` (proper subset)

```nim
proc `<`*[A](s1, s2: PackedSet[A]): bool
```

**What it does.** Returns `true` if `s1` is a subset of `s2` but not equal to it (`s2` has at least one element beyond `s1`).

**Implementation notes.** Expressed in terms of the already-defined `<=`: `s1 <= s2 and not (s2 <= s1)` — up to two subset checks.

**Parameters.**
- `s1, s2: PackedSet[A]` — the sets being compared.

**Examples.**

```nim
let
  a = toPackedSet([1])
  b = toPackedSet([1, 2])
  c = toPackedSet([1, 3])
assert a < b
assert not (b < b)           # a set is not a proper subset of itself
assert not (c < b)
```

---

### `==` (equality)

```nim
proc `==`*[A](s1, s2: PackedSet[A]): bool
```

**What it does.** Returns `true` if both sets contain exactly the same elements (regardless of insertion order or the internal representation — a small-set-mode set and a bitset-mode set with the same contents are equal).

**Implementation notes.** A mutual subset check in both directions: `s1 <= s2 and s2 <= s1`.

**Parameters.**
- `s1, s2: PackedSet[A]` — the sets being compared.

**Examples.**

```nim
assert toPackedSet([1, 2]) == toPackedSet([2, 1])
assert toPackedSet([1, 2]) == toPackedSet([2, 1, 2])   # the duplicate during construction has no effect
```

---

### `disjoint`

```nim
proc disjoint*[A](s1, s2: PackedSet[A]): bool
```

**What it does.** Returns `true` if `s1` and `s2` have no elements in common.

**Implementation notes.** A traversal of `s1` checking `contains(s2, item)`; on the first match found, it returns `false` right away (short-circuiting).

**Parameters.**
- `s1, s2: PackedSet[A]` — the sets being compared.

**Examples.**

```nim
let
  a = toPackedSet([1, 2])
  b = toPackedSet([2, 3])
  c = toPackedSet([3, 4])
assert disjoint(a, b) == false
assert disjoint(a, c) == true
```

---

### `card`

```nim
proc card*[A](s: PackedSet[A]): int {.inline.}
```

**What it does.** A synonym for `len` — the number of elements in the set. The name refers to the mathematical term "cardinality."

**Implementation notes.** A direct delegation to `len(s)`, with no additional logic.

**Parameters.**
- `s: PackedSet[A]` — the set.

**Examples.**

```nim
let a = toPackedSet([1, 3, 5])
assert card(a) == 3
assert card(a) == len(a)
```

---

## VII. Iteration and output

### `items`

```nim
iterator items*[A](s: PackedSet[A]): A {.inline.}
```

**What it does.** Iterates over every element of the set `s` — the base iterator that `len`, `$`, the set-theoretic operations, and the comparison operations are all built on. Works with an ordinary `for value in s`.

**Implementation notes.** Branches on the storage mode:

- in small-set mode — a simple linear pass over the active part of the array `a` (`0 ..< elems`), converting each value back to type `A`;
- in bitset mode — a traversal of the linked list of blocks via `head` (not via the hash table `data` — the `head` list exists specifically for sequential traversal, whereas `data` serves fast lookup by key). Within each block, every machine word of the bit vector `bits` is scanned; for each nonzero word, its bits are extracted one at a time by shifting right (`w shr 1`) and testing the low bit, and the original value is reconstructed with the formula `(trunk.key shl TrunkShift) or (word_index shl IntShift + bit_index)`.

**A note on ordering.** Iteration does **not** guarantee ascending order: in small-set mode the order is insertion order (and can be further disturbed by the swap-remove used on deletion); in bitset mode the order is determined by the order of blocks in the linked list `head` (typically the reverse of their creation order), and only within a single block do values come out in ascending order.

**Parameters.**
- `s: PackedSet[A]` — the set being iterated; left unchanged. `s` must not be mutated during iteration — the source specifically notes that copying a bits word inside the loop is safe precisely because mutating operations are disallowed during traversal.

**Examples.**

```nim
let a = toPackedSet([10, 20, 30])
var total = 0
for value in a:               # order isn't guaranteed, but each value appears exactly once
  total = total + value
assert total == 60

let empty = initPackedSet[int]()   # boundary case: an empty set
for value in empty:
  assert false                      # the loop body never runs
```

---

### `` `$` ``

```nim
proc `$`*[A](s: PackedSet[A]): string
```

**What it does.** A string representation of the set as `{a, b, c}`.

**Implementation notes.** Implemented through the internal template `dollarImpl`, which relies on the same `items` iterator — so the order of elements in the string follows the same rules described above (not guaranteed to be ascending). That the output for small sets of integers usually looks sorted is a side effect of small-set mode preserving insertion order, not a guarantee of the module.

**Parameters.**
- `s: PackedSet[A]` — the set being rendered.

**Examples.**

```nim
let a = toPackedSet([1, 2, 3])
assert $a == "{1, 2, 3}"

let empty = initPackedSet[int]()   # boundary case: an empty set
assert $empty == "{}"
```

---

## VIII. Practical recipes

### Removing duplicates from a list

```nim
let raw = [7, 3, 7, 1, 3, 3, 9]
let unique = toPackedSet(raw)
echo unique                  # prints the deduplicated set, e.g. "{7, 3, 1, 9}"
assert len(unique) == 4
```

### Graph traversal that skips already-visited nodes

```nim
# a graph given as an adjacency list: node -> list of neighbours
let graph = @[
  @[1, 2],   # neighbours of node 0
  @[3],      # neighbours of node 1
  @[3],      # neighbours of node 2
  newSeq[int]()
]

var
  visited = initPackedSet[int]()
  queue = @[0]

while len(queue) > 0:
  let node = queue[0]
  queue.delete(0)
  # containsOrIncl tells us right away whether the node was already visited,
  # and marks it as visited in the same call
  if not containsOrIncl(visited, node):
    for neighbour in graph[node]:
      if neighbour notin visited:
        queue.add(neighbour)

assert len(visited) == 4
```

### Comparing two sets of permissions

```nim
let
  required = toPackedSet(["read", "write", "execute"].mapIt(ord(it[0])))
  granted = toPackedSet(["read", "write"].mapIt(ord(it[0])))

# check whether the granted permissions cover everything required
if not (required <= granted):
  let missing = difference(required, granted)
  echo missing                # which permissions are missing
```

### Counting the change between two snapshots of an identifier set

```nim
let
  before = toPackedSet([1, 2, 3, 4])
  after = toPackedSet([2, 3, 4, 5])
  changed = symmetricDifference(before, after)

echo card(changed)            # prints 2: identifiers "1" and "5"
assert 1 in changed            # was there, now gone
assert 5 in changed             # newly appeared
assert 2 notin changed           # unchanged — absent from the difference
```

---

## IX. Quick reference table

| Task                                                        | Mutates the argument | Returns a new value |
|--------------------------------------------------------------|:---:|:---:|
| Create an empty set                                          | —   | `initPackedSet[A]()` |
| Build a set from an array/`seq`, dropping duplicates          | —   | `toPackedSet(x)` |
| Test whether a value is a member                              | no  | `contains(s, key)`, `key in s` |
| Add a single value                                             | yes | `incl(s, key)` |
| Add a value and learn whether it was there before               | yes | `containsOrIncl(s, key)` |
| Remove a single value                                            | yes | `excl(s, key)` |
| Remove a value and learn whether it was absent before             | yes | `missingOrExcl(s, key)` |
| Add every element of another set                                   | yes | `incl(s, other)` |
| Remove every element of another set                                  | yes | `excl(s, other)` |
| Empty the set completely                                              | yes | `clear(s)` |
| Test whether the set is empty                                          | no  | `isNil(s)` (careful, see the caveat in section IV.4) or `len(s) == 0` |
| Copy a set independently                                                 | yes (target) | `` `=copy`(dest, src) ``, `assign(dest, src)` |
| Union of two sets                                                          | no  | `union(s1, s2)`, `s1 + s2` |
| Intersection of two sets                                                     | no  | `intersection(s1, s2)`, `s1 * s2` |
| Difference of two sets                                                        | no  | `difference(s1, s2)`, `s1 - s2` |
| Symmetric difference of two sets                                                | no  | `symmetricDifference(s1, s2)` |
| Test whether one set is a subset of another                                     | no  | `s1 <= s2` |
| Test proper subset                                                                | no  | `s1 < s2` |
| Test set equality                                                                    | no  | `s1 == s2` |
| Test that two sets share no elements                                                  | no  | `disjoint(s1, s2)` |
| Get the number of elements                                                              | no  | `len(s)`, `card(s)` |
| Iterate over every element                                                                | no  | `for x in s: ...` |
| Get a string representation                                                                 | no  | `` $s `` |

---

## X. Summary: which procedure to choose

- Need to quickly build a set from a ready-made list of values while dropping duplicates → use `toPackedSet`.
- Need to simply check whether a value is in the set → use `contains` or the `in` operator.
- Need to add a value and also find out whether it was already there (e.g. to avoid processing it twice) → use `containsOrIncl` instead of separate `contains` + `incl` calls.
- Need to remove a value and also find out whether it was absent (keeping in mind that in bitset mode this check costs O(n)) → use `missingOrExcl`, but avoid it in hot loops over large sets when possible.
- Need to union, intersect, or subtract an entire other set without creating a new object → use `incl(s, other)` / `excl(s, other)` (they mutate `s` in place).
- Need the result of a set-theoretic operation as a new set, without touching the originals → use `union` / `intersection` / `difference` / `symmetricDifference` (or the operators `+` / `*` / `-`).
- Need to find out which values changed between two "snapshots" of the same set → use `symmetricDifference`.
- Need to compare sets with each other (subset, proper subset, equality, disjointness) → use `<=`, `<`, `==`, `disjoint`, respectively.
- Need to independently clone a set before modifying the copy (since plain `=` assignment already invokes `` `=copy` `` automatically because of the internal reference nodes in bitset mode) → an explicit `` `=copy` ``/`assign` call is only needed in rare cases of manual memory management.
- Need to test emptiness of a set that may have once switched to bitset mode and then been fully emptied via `excl` → prefer `len(s) == 0` over `isNil(s)` (see the caveat in section IV.4).
- Need predictable (e.g. ascending) output order → `packedsets` guarantees no such thing, either in `items` or in `` `$` ``; if order matters, sort the result of `items` explicitly using `std/algorithm`.
