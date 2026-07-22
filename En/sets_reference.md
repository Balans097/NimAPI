# sets — module reference

> **Import:** `import std/sets`
> **Scope:** efficient hash sets (`HashSet`) and ordered hash sets (`OrderedSet`) — storing unique values, membership testing, set-theoretic operations (union, intersection, difference).

The `sets` module implements two container types that store only unique values of any type for which a `hash` function is defined. `HashSet` does not guarantee element order and is optimized for speed; `OrderedSet` additionally remembers insertion order at the cost of slightly higher memory use and O(n) removal. The module's general convention is pairs of procedures — "in-place / returns a new value" (for example, `incl`/`union`, `excl`/`difference`) — plus a set of operator aliases (`+`, `*`, `-`, `-+-`) for the set-theoretic functions. Both types have *value semantics* — assignment `=` copies the whole set.

---

## Table of contents

I. [Types and general information](#i-types-and-general-information)
  1. [`HashSet[A]`](#hashseta)
  2. [`OrderedSet[A]`](#orderedseta)
  3. [`SomeSet[A]`](#someseta)

II. [Creation and initialization](#ii-creation-and-initialization)
  1. [`init` (variant A — HashSet)](#init-variant-a--hashset)
  2. [`init` (variant B — OrderedSet)](#init-variant-b--orderedset)
  3. [`initHashSet`](#inithashset)
  4. [`initOrderedSet`](#initorderedset)
  5. [`toHashSet`](#tohashset)
  6. [`toOrderedSet`](#toorderedset)

III. [Membership testing and size](#iii-membership-testing-and-size)
  1. [`contains` (variant A — HashSet)](#contains-variant-a--hashset)
  2. [`contains` (variant B — OrderedSet)](#contains-variant-b--orderedset)
  3. [`` `[]` ``](#)
  4. [`len`](#len)
  5. [`card`](#card)

IV. [Adding elements](#iv-adding-elements)
  1. [`incl` (element, HashSet)](#incl-element-hashset)
  2. [`incl` (element, OrderedSet)](#incl-element-orderedset)
  3. [`incl` (set into HashSet)](#incl-set-into-hashset)
  4. [`incl` (OrderedSet into HashSet)](#incl-orderedset-into-hashset)
  5. [`containsOrIncl`](#containsorincl)

V. [Removing elements](#v-removing-elements)
  1. [`excl` (element, HashSet)](#excl-element-hashset)
  2. [`excl` (element, OrderedSet)](#excl-element-orderedset)
  3. [`excl` (set from HashSet)](#excl-set-from-hashset)
  4. [`missingOrExcl`](#missingorexcl)
  5. [`pop`](#pop)
  6. [`clear`](#clear)

VI. [Set-theoretic operations](#vi-set-theoretic-operations)
  1. [`union` / `` `+` ``](#union--)
  2. [`intersection` / `` `*` ``](#intersection--)
  3. [`difference` / `` `-` ``](#difference--)
  4. [`symmetricDifference` / `` `-+-` ``](#symmetricdifference---)
  5. [`disjoint`](#disjoint)

VII. [Relations between sets](#vii-relations-between-sets)
  1. [`` `<` ``](#-1)
  2. [`` `<=` ``](#-2)
  3. [`` `==` `` (HashSet)](#-hashset)
  4. [`` `==` `` (OrderedSet)](#-orderedset)

VIII. [Iteration](#viii-iteration)
  1. [`items` (HashSet)](#items-hashset)
  2. [`items` (OrderedSet)](#items-orderedset)
  3. [`pairs` (OrderedSet)](#pairs-orderedset)

IX. [Conversion and utility procedures](#ix-conversion-and-utility-procedures)
  1. [`map`](#map)
  2. [`hash` (HashSet)](#hash-hashset)
  3. [`hash` (OrderedSet)](#hash-orderedset)
  4. [`` `$` `` (HashSet)](#-hashset-1)
  5. [`` `$` `` (OrderedSet)](#-orderedset-1)

X. [Deprecated procedures](#x-deprecated-procedures)
  1. [`initSet`](#initset)
  2. [`toSet`](#toset)
  3. [`isValid`](#isvalid)

XI. [Practical recipes](#xi-practical-recipes)
  1. [Deduplicating a sequence](#1-deduplicating-a-sequence)
  2. [Finding elements common to several collections](#2-finding-elements-common-to-several-collections)
  3. [Tracking already-processed elements (a visited cache)](#3-tracking-already-processed-elements-a-visited-cache)
  4. [Comparing two versions of a configuration](#4-comparing-two-versions-of-a-configuration)
  5. [A scheduler that preserves queueing order](#5-a-scheduler-that-preserves-queueing-order)

XII. [Quick reference table](#xii-quick-reference-table)

XIII. [Summary: which procedure to pick](#xiii-summary-which-procedure-to-pick)

---

## I. Types and general information

### `HashSet[A]`

```nim
type
  HashSet*[A] = object
```

**Purpose.** The module's core type — an unordered set of unique values of type `A`. Internally it holds a single flat `seq` of hash-code/key pairs; the order in which elements come up during iteration is determined purely by their position inside that internal array and has nothing to do with insertion order.

**Implementation notes.** `HashSet` is built on open addressing with linear probing: each slot of the internal `seq` is either empty or holds a `(hcode, key)` pair. Because of this, insertion and lookup are O(1) on average, but degrade toward O(n) if collisions pile up. The `counter` field tracks the current element count separately, so `len` doesn't have to recount occupied slots on every call.

- **Type parameters:**
  - `A` — the element type; must support `hash(x: A): Hash` and `==`.

**Example:**

```nim
var s: HashSet[int]  # since Nim 0.20 explicit initialization isn't required
incl(s, 5)
echo len(s)  # prints 1
```

---

### `OrderedSet[A]`

```nim
type
  OrderedSet*[A] = object
```

**Purpose.** A variant of the hash set that additionally remembers insertion order — iterating (`items`, `pairs`) yields elements in exactly the order they were included.

**Implementation notes.** Each slot of the internal array stores not just `(hcode, key)` but also a `next` index — a pointer to the next occupied slot in insertion order. The `first`/`last` fields hold the head and tail indices of that linked list. As a result, removing an element (`excl`) requires walking the list to restore its links — hence the O(n) cost, unlike the average O(1) of a plain `HashSet`.

- **Type parameters:**
  - `A` — the element type; same requirements as `HashSet`.

**Example:**

```nim
var o: OrderedSet[string]
incl(o, "b")
incl(o, "a")
for x in items(o):
  echo x  # prints "b" first, then "a" — insertion order is preserved
```

---

### `SomeSet[A]`

```nim
type
  SomeSet*[A] = HashSet[A] | OrderedSet[A]
```

**Purpose.** A type union that lets generic code work uniformly with either `HashSet` or `OrderedSet` (used internally by the module in the implementation of shared templates).

- **Type parameters:**
  - `A` — the element type shared by both set variants.

---

## II. Creation and initialization

### `init` (variant A — HashSet)

```nim
proc init*[A](s: var HashSet[A], initialSize = defaultInitialSize)
```

**Purpose.** Initializes the hash set `s`, bringing it to an empty state. Since Nim 0.20 this call isn't required — a variable of type `HashSet[A]` is already usable right after declaration. Calling it on an already-populated set discards all its values — which can be more convenient than iterating over the existing values and calling `excl` on each of them.

**Implementation notes.** `initialSize` sets the initial size of the internal `seq`; it is rounded up to the nearest power of two (this lets a slot's index be computed with a bitwise `and` instead of a modulo — so slot lookup is cheaper).

- **Parameters:**
  - `s: var HashSet[A]` — the set being initialized (mutable).
  - `initialSize: int` — the expected starting size, defaults to `defaultInitialSize` (64).

**Example:**

```nim
var a: HashSet[int]
init(a)  # not required, but valid — brings a to an empty state
echo len(a)  # prints 0
```

---

### `init` (variant B — OrderedSet)

```nim
proc init*[A](s: var OrderedSet[A], initialSize = defaultInitialSize)
```

**Purpose.** The `OrderedSet` counterpart of the previous procedure — brings the set to an empty state and resets the internal `first`/`last` pointers.

- **Parameters:**
  - `s: var OrderedSet[A]` — the set being initialized.
  - `initialSize: int` — the expected starting size, defaults to 64.

**Example:**

```nim
var a: OrderedSet[int]
init(a)
echo len(a)  # prints 0
```

---

### `initHashSet`

```nim
proc initHashSet*[A](initialSize = defaultInitialSize): HashSet[A]
```

**Purpose.** A wrapper around `init` that returns an empty hash set as a value — convenient for a one-line declaration inside `var`/`let`. Like `init`, it hasn't been required since version 0.20, but it remains the idiomatic way to create a set "on the fly".

- **Parameters:**
  - `initialSize: int` — the expected starting size, defaults to 64.

**Example:**

```nim
var a = initHashSet[int]()
incl(a, 3)
assert len(a) == 1
```

---

### `initOrderedSet`

```nim
proc initOrderedSet*[A](initialSize = defaultInitialSize): OrderedSet[A]
```

**Purpose.** The `OrderedSet` counterpart of `initHashSet`.

- **Parameters:**
  - `initialSize: int` — the expected starting size, defaults to 64.

**Example:**

```nim
var a = initOrderedSet[int]()
incl(a, 3)
assert len(a) == 1
```

---

### `toHashSet`

```nim
proc toHashSet*[A](keys: openArray[A]): HashSet[A]
```

**Purpose.** Builds a new hash set from the contents of the collection `keys` (an array, `seq`, or string), automatically dropping duplicates. The order of elements in the resulting set is unspecified.

- **Parameters:**
  - `keys: openArray[A]` — the source collection, immutable.

**Example:**

```nim
let
  a = toHashSet([5, 3, 2])
  b = toHashSet("abracadabra")
assert len(a) == 3   # a == {2, 3, 5}
assert len(b) == 5   # b == {'a', 'b', 'c', 'd', 'r'}
```

Edge case — an empty collection:

```nim
let empty = toHashSet(newSeq[int]())
assert len(empty) == 0
```

---

### `toOrderedSet`

```nim
proc toOrderedSet*[A](keys: openArray[A]): OrderedSet[A]
```

**Purpose.** Builds an ordered set from the collection `keys`, dropping duplicates while preserving the order of each value's first occurrence. Unlike `toHashSet`, the resulting traversal order is predictable and matches the order of first occurrences.

- **Parameters:**
  - `keys: openArray[A]` — the source collection.

**Example:**

```nim
let
  a = toOrderedSet([5, 3, 2])
  b = toOrderedSet("abracadabra")
assert len(a) == 3   # a == {5, 3, 2} — order of first occurrences
assert len(b) == 5   # b == {'a', 'b', 'r', 'c', 'd'}
```

---

## III. Membership testing and size

### `contains` (variant A — HashSet)

```nim
proc contains*[A](s: HashSet[A], key: A): bool
```

**Purpose.** Returns `true` if `key` is present in `s`. This procedure is what enables the `in`/`notin` operators.

- **Parameters:**
  - `s: HashSet[A]` — the set, immutable.
  - `key: A` — the value whose presence is being checked.

**Example:**

```nim
var values = initHashSet[int]()
assert(not contains(values, 2))
assert 2 notin values

incl(values, 2)
assert contains(values, 2)
assert 2 in values
```

---

### `contains` (variant B — OrderedSet)

```nim
proc contains*[A](s: OrderedSet[A], key: A): bool
```

**Purpose.** The `OrderedSet` counterpart of the previous procedure — membership testing itself doesn't depend on insertion order and is just as fast as for `HashSet`.

- **Parameters:**
  - `s: OrderedSet[A]` — the set.
  - `key: A` — the value being looked up.

**Example:**

```nim
var values = initOrderedSet[int]()
assert(not contains(values, 2))
incl(values, 2)
assert 2 in values
```

---

### `` `[]` ``

```nim
proc `[]`*[A](s: var HashSet[A], key: A): var A
```

**Purpose.** Returns not the `key` argument itself, but the actual instance of the value stored inside the set `s` that equals `key`. Useful when `hash`/`==` are overloaded so that "equal" values may still differ in other fields, and you need access to the specific instance stored in the set (reference semantics layered on top of value types). If `key` isn't found, a `KeyError` is raised.

**Implementation notes.** The procedure uses `rawGet` — the internal lookup by hash code and key comparison; if a slot is found (`index >= 0`), a reference to the key stored there is returned, which is what allows modifying its fields in place.

- **Parameters:**
  - `s: var HashSet[A]` — the set, mutable (because a mutable reference into internal storage is returned).
  - `key: A` — the value whose equivalent is being looked up in the set.

**Example:**

```nim
type Person = object
  name: string
  visits: int

var people = initHashSet[Person]()
incl(people, Person(name: "Ann", visits: 0))
# get a reference to the stored instance and change its "non-key" field
`[]`(people, Person(name: "Ann"))[].visits.inc
```

Error case — looking up a missing key:

```nim
var s = initHashSet[int]()
doAssertRaises(KeyError):
  discard `[]`(s, 42)
```

---

### `len`

```nim
proc len*[A](s: HashSet[A]): int
```

**Purpose.** Returns the number of elements in `s`. It's safe to call this procedure even on a not-yet-initialized variable — in that case it returns 0. `OrderedSet` has an identically-named procedure with the same behavior.

- **Parameters:**
  - `s: HashSet[A]` (or `OrderedSet[A]`) — the set, immutable.

**Example:**

```nim
var a: HashSet[string]
assert len(a) == 0          # edge case — an uninitialized set
let s = toHashSet([3, 5, 7])
assert len(s) == 3
```

---

### `card`

```nim
proc card*[A](s: HashSet[A]): int
```

**Purpose.** A synonym for `len` — the name refers to the mathematical term "cardinality" of a set. It exists for both `HashSet` and `OrderedSet`; in meaning and result it's identical to `len`.

- **Parameters:**
  - `s: HashSet[A]` (or `OrderedSet[A]`) — the set.

**Example:**

```nim
let s = toHashSet([1, 2, 3])
assert card(s) == len(s)
```

---

## IV. Adding elements

### `incl` (element, HashSet)

```nim
proc incl*[A](s: var HashSet[A], key: A)
```

**Purpose.** Adds `key` to the set `s`. If `key` is already present, nothing happens — including it again does not create a duplicate and is not an error.

- **Parameters:**
  - `s: var HashSet[A]` — the set, mutable.
  - `key: A` — the value being added.

**Example:**

```nim
var values = initHashSet[int]()
incl(values, 2)
incl(values, 2)  # edge case — re-including doesn't grow the set
assert len(values) == 1
```

---

### `incl` (element, OrderedSet)

```nim
proc incl*[A](s: var OrderedSet[A], key: A)
```

**Purpose.** The `OrderedSet` counterpart of the previous procedure: a new element is appended to the end of the insertion order, and re-including an existing element changes neither the set nor its order.

- **Parameters:**
  - `s: var OrderedSet[A]` — the set.
  - `key: A` — the value being added.

**Example:**

```nim
var values = initOrderedSet[int]()
incl(values, 2)
incl(values, 2)
assert len(values) == 1
```

---

### `incl` (set into HashSet)

```nim
proc incl*[A](s: var HashSet[A], other: HashSet[A])
```

**Purpose.** Adds every element of `other` into `s`. This is the "in-place" version of union — equivalent to `s = s + other`, but without allocating an intermediate copy of the whole result.

**Implementation notes.** The implementation is a plain loop, `for item in other: incl(s, item)`, so its cost is O(m), where m is the size of `other`, given that insertion is O(1) on average.

- **Parameters:**
  - `s: var HashSet[A]` — the set being extended.
  - `other: HashSet[A]` — the source of the added elements, immutable.

**Example:**

```nim
var
  values = toHashSet([1, 2, 3])
  others = toHashSet([3, 4, 5])
incl(values, others)
assert len(values) == 5  # 3 wasn't duplicated
```

---

### `incl` (OrderedSet into HashSet)

```nim
proc incl*[A](s: var HashSet[A], other: OrderedSet[A])
```

**Purpose.** Adds every element from the ordered set `other` into the plain hash set `s`. This lets you mix both set variants in a single expression — for example, building a plain set out of several ordered sources.

- **Parameters:**
  - `s: var HashSet[A]` — the destination set.
  - `other: OrderedSet[A]` — the source of the elements.

**Example:**

```nim
var
  values = toHashSet([1, 2, 3])
  others = toOrderedSet([3, 4, 5])
incl(values, others)
assert len(values) == 5
```

---

### `containsOrIncl`

```nim
proc containsOrIncl*[A](s: var HashSet[A], key: A): bool
proc containsOrIncl*[A](s: var OrderedSet[A], key: A): bool
```

**Purpose.** Checks and adds in one call: includes `key` in the set and returns whether the set already contained that element *before* the call. Unlike `incl`, this lets you learn whether the element was new with a single call, without a separate `contains` check before inserting (which avoids looking up the slot twice).

- **Parameters:**
  - `s: var HashSet[A]` (or `var OrderedSet[A]`) — the set, mutable.
  - `key: A` — the value being checked and added.
- **Returns:** `true` if `key` was already in the set; `false` if it was added for the first time.

**Example:**

```nim
var values = initHashSet[int]()
assert containsOrIncl(values, 2) == false  # added for the first time
assert containsOrIncl(values, 2) == true   # was already in the set
assert containsOrIncl(values, 3) == false
```

---

## V. Removing elements

### `excl` (element, HashSet)

```nim
proc excl*[A](s: var HashSet[A], key: A)
```

**Purpose.** Removes `key` from the set `s`. If `key` isn't found in the set, nothing happens — that's not an error.

- **Parameters:**
  - `s: var HashSet[A]` — the set, mutable.
  - `key: A` — the value being removed.

**Example:**

```nim
var s = toHashSet([2, 3, 6, 7])
excl(s, 2)
excl(s, 2)  # edge case — removing an already-absent element is safe
assert len(s) == 3
```

---

### `excl` (element, OrderedSet)

```nim
proc excl*[A](s: var OrderedSet[A], key: A)
```

**Purpose.** The `OrderedSet` counterpart of the previous procedure. Its efficiency is O(n), because keeping the insertion-order linked list consistent requires walking it and re-linking the removed node's neighbors.

- **Parameters:**
  - `s: var OrderedSet[A]` — the set.
  - `key: A` — the value being removed.

**Example:**

```nim
var s = toOrderedSet([2, 3, 6, 7])
excl(s, 2)
excl(s, 2)
assert len(s) == 3
```

---

### `excl` (set from HashSet)

```nim
proc excl*[A](s: var HashSet[A], other: HashSet[A])
```

**Purpose.** Removes from `s` every element that's present in `other`. This is the "in-place" version of difference — equivalent to `s = s - other`.

- **Parameters:**
  - `s: var HashSet[A]` — the set elements are removed from.
  - `other: HashSet[A]` — the set of values to remove.

**Example:**

```nim
var
  numbers = toHashSet([1, 2, 3, 4, 5])
  even = toHashSet([2, 4, 6, 8])
excl(numbers, even)
assert len(numbers) == 3  # numbers == {1, 3, 5}
```

---

### `missingOrExcl`

```nim
proc missingOrExcl*[A](s: var HashSet[A], key: A): bool
proc missingOrExcl*[A](s: var OrderedSet[A], key: A): bool
```

**Purpose.** The mirror image of `containsOrIncl`: removes `key` from the set and reports whether it was absent *before* the call. Useful when it matters whether the removal was "real" (the element actually existed) or a no-op.

- **Parameters:**
  - `s: var HashSet[A]` (or `var OrderedSet[A]`) — the set.
  - `key: A` — the value being removed.
- **Returns:** `true` if `key` was absent from `s` (there was nothing to remove); `false` if the element was actually removed.

**Example:**

```nim
var s = toHashSet([2, 3, 6, 7])
assert missingOrExcl(s, 4) == true   # 4 wasn't in the set either
assert missingOrExcl(s, 6) == false  # 6 was actually removed
assert missingOrExcl(s, 6) == true   # removing again — now absent
```

---

### `pop`

```nim
proc pop*[A](s: var HashSet[A]): A
```

**Purpose.** Removes and returns a single, arbitrary element from the set. Which exact element is returned is unspecified (it depends on the internal slot layout). If the set is empty, a `KeyError` is raised.

- **Parameters:**
  - `s: var HashSet[A]` — the set, mutable.

**Example:**

```nim
var s = toHashSet([2, 1])
assert [pop(s), pop(s)] in [[1, 2], [2, 1]]  # order isn't guaranteed
```

Error case:

```nim
var empty = initHashSet[int]()
doAssertRaises(KeyError):
  discard pop(empty)
```

---

### `clear`

```nim
proc clear*[A](s: var HashSet[A])
proc clear*[A](s: var OrderedSet[A])
```

**Purpose.** Brings the set to an empty state without shrinking the memory already allocated for it — subsequent insertions into the same set will be cheaper than inserting into a freshly created one. The operation is O(n), where n is the size of the internal storage (not just the occupied slots).

- **Parameters:**
  - `s: var HashSet[A]` (or `var OrderedSet[A]`) — the set, mutable.

**Example:**

```nim
var s = toHashSet([3, 5, 7])
clear(s)
assert len(s) == 0
```

---

## VI. Set-theoretic operations

Every procedure in this section works only with `HashSet` and does not modify the source sets — each one returns a new result.

### `union` / `` `+` ``

```nim
proc union*[A](s1, s2: HashSet[A]): HashSet[A]
proc `+`*[A](s1, s2: HashSet[A]): HashSet[A]
```

**Purpose.** Returns the union of `s1` and `s2` — the set of every object belonging to at least one of the two sets (mathematically *A ∪ B*). The `+` operator is an exact alias for `union`.

- **Parameters:**
  - `s1, s2: HashSet[A]` — the source sets, immutable.

**Example:**

```nim
let
  a = toHashSet(["a", "b"])
  b = toHashSet(["b", "c"])
  c = union(a, b)
assert c == toHashSet(["a", "b", "c"])
```

---

### `intersection` / `` `*` ``

```nim
proc intersection*[A](s1, s2: HashSet[A]): HashSet[A]
proc `*`*[A](s1, s2: HashSet[A]): HashSet[A]
```

**Purpose.** Returns the intersection of `s1` and `s2` — the set of objects belonging to both sets at once (*A ∩ B*).

**Implementation notes.** The implementation always iterates over the *smaller* of the two sets and tests each element's membership in the larger one — this minimizes total cost, since membership testing is O(1) on average and the smaller set is shorter to walk.

- **Parameters:**
  - `s1, s2: HashSet[A]` — the source sets.

**Example:**

```nim
let
  a = toHashSet(["a", "b"])
  b = toHashSet(["b", "c"])
  c = intersection(a, b)
assert c == toHashSet(["b"])
```

---

### `difference` / `` `-` ``

```nim
proc difference*[A](s1, s2: HashSet[A]): HashSet[A]
proc `-`*[A](s1, s2: HashSet[A]): HashSet[A]
```

**Purpose.** Returns the difference of `s1` and `s2` — the elements that belong to `s1` but not to `s2` (*A ∖ B*). Note that the operation is asymmetric: `difference(a, b)` and `difference(b, a)` generally give different results.

- **Parameters:**
  - `s1, s2: HashSet[A]` — the source sets.

**Example:**

```nim
let
  a = toHashSet(["a", "b"])
  b = toHashSet(["b", "c"])
  c = difference(a, b)
assert c == toHashSet(["a"])
```

---

### `symmetricDifference` / `` `-+-` ``

```nim
proc symmetricDifference*[A](s1, s2: HashSet[A]): HashSet[A]
proc `-+-`*[A](s1, s2: HashSet[A]): HashSet[A]
```

**Purpose.** Returns the symmetric difference — the elements that belong to exactly one of the two sets but not to both at once (*A △ B*). Unlike `difference`, this operation is symmetric: `symmetricDifference(a, b) == symmetricDifference(b, a)`.

**Implementation notes.** The implementation starts from a copy of `s1`, then for each element of `s2` uses `containsOrIncl`: if the element was already there (meaning it belonged to both sets), it's immediately removed with `excl` — so the result is reached in a single pass over `s2`, without building an intermediate intersection.

- **Parameters:**
  - `s1, s2: HashSet[A]` — the source sets.

**Example:**

```nim
let
  a = toHashSet(["a", "b"])
  b = toHashSet(["b", "c"])
  c = symmetricDifference(a, b)
assert c == toHashSet(["a", "c"])
```

---

### `disjoint`

```nim
proc disjoint*[A](s1, s2: HashSet[A]): bool
```

**Purpose.** Returns `true` if `s1` and `s2` have no elements in common.

- **Parameters:**
  - `s1, s2: HashSet[A]` — the sets being compared.

**Example:**

```nim
let
  a = toHashSet(["a", "b"])
  b = toHashSet(["b", "c"])
assert disjoint(a, b) == false
assert disjoint(a, difference(b, a)) == true
```

---

## VII. Relations between sets

### `` `<` ``

```nim
proc `<`*[A](s, t: HashSet[A]): bool
```

**Purpose.** Returns `true` if `s` is a strict (proper) subset of `t` — every element of `s` belongs to `t`, but `t` also has other elements that `s` doesn't have.

- **Parameters:**
  - `s, t: HashSet[A]` — the sets being compared.

**Example:**

```nim
let
  a = toHashSet(["a", "b"])
  b = toHashSet(["b", "c"])
  c = intersection(a, b)
assert c < a and c < b
assert(not (a < a))  # edge case — a set is not a strict subset of itself
```

---

### `` `<=` ``

```nim
proc `<=`*[A](s, t: HashSet[A]): bool
```

**Purpose.** Returns `true` if `s` is a subset of `t` — every element of `s` belongs to `t`; `s` and `t` may also be equal.

- **Parameters:**
  - `s, t: HashSet[A]` — the sets being compared.

**Example:**

```nim
let
  a = toHashSet(["a", "b"])
  b = toHashSet(["b", "c"])
  c = intersection(a, b)
assert c <= a and c <= b
assert a <= a  # edge case — a set is a subset of itself
```

---

### `` `==` `` (HashSet)

```nim
proc `==`*[A](s, t: HashSet[A]): bool
```

**Purpose.** Returns `true` if `s` and `t` contain the same elements (that is, they're mutual subsets of each other and have the same size). The internal layout of elements has no effect on equality.

- **Parameters:**
  - `s, t: HashSet[A]` — the sets being compared.

**Example:**

```nim
var
  a = toHashSet([1, 2])
  b = toHashSet([2, 1])
assert a == b  # insertion order doesn't matter for HashSet
```

---

### `` `==` `` (OrderedSet)

```nim
proc `==`*[A](s, t: OrderedSet[A]): bool
```

**Purpose.** Returns `true` if `s` and `t` contain the same elements **in the same order**. This is the key difference from `HashSet`: two ordered sets with the same contents but different insertion order are considered not equal.

- **Parameters:**
  - `s, t: OrderedSet[A]` — the sets being compared.

**Example:**

```nim
let
  a = toOrderedSet([1, 2])
  b = toOrderedSet([2, 1])
assert(not (a == b))  # same contents, different order — not equal
```

---

## VIII. Iteration

### `items` (HashSet)

```nim
iterator items*[A](s: HashSet[A]): A
```

**Purpose.** Iterates over the elements of the set `s`. Traversal order is determined by the internal slot layout and is not guaranteed. If the set's size changes during iteration, an assertion fires — you can't modify a set while looping over it with `for`.

- **Parameters:**
  - `s: HashSet[A]` — the set, immutable during traversal.

**Example:**

```nim
var a = initHashSet[(int, int)]()
incl(a, (2, 3))
incl(a, (3, 2))
incl(a, (2, 3))  # edge case — the duplicate doesn't create a new pair
assert len(a) == 2
```

---

### `items` (OrderedSet)

```nim
iterator items*[A](s: OrderedSet[A]): A
```

**Purpose.** Iterates over the elements of the ordered set strictly in the order they were inserted.

- **Parameters:**
  - `s: OrderedSet[A]` — the set.

**Example:**

```nim
var a = initOrderedSet[int]()
for value in [9, 2, 1, 5, 1, 8, 4, 2]:
  incl(a, value)
for value in items(a):
  echo "Got: ", value
# --> Got: 9
# --> Got: 2
# --> Got: 1
# --> Got: 5
# --> Got: 8
# --> Got: 4
```

---

### `pairs` (OrderedSet)

```nim
iterator pairs*[A](s: OrderedSet[A]): tuple[a: int, b: A]
```

**Purpose.** Iterates over (position, value) pairs for the ordered set `s` — handy when you need not just the element but also its insertion index.

- **Parameters:**
  - `s: OrderedSet[A]` — the set.

**Example:**

```nim
let a = toOrderedSet("abracadabra")
var p = newSeq[(int, char)]()
for x in pairs(a):
  add(p, x)
assert p == @[(0, 'a'), (1, 'b'), (2, 'r'), (3, 'c'), (4, 'd')]
```

---

## IX. Conversion and utility procedures

### `map`

```nim
proc map*[A, B](data: HashSet[A], op: proc (x: A): B): HashSet[B]
```

**Purpose.** Builds a new set by applying the function `op` to each element of `data` — handy for changing the element type (for example, numbers into strings). Note: if `op` is not injective (different inputs produce the same output), the resulting set may end up smaller than the source — duplicate results collapse.

- **Parameters:**
  - `data: HashSet[A]` — the source set, immutable.
  - `op: proc (x: A): B` — the element-transforming function.

**Example:**

```nim
let
  a = toHashSet([1, 2, 3])
  b = map(a, proc (x: int): string = $x)
assert b == toHashSet(["1", "2", "3"])
```

Edge case — a non-injective transformation:

```nim
let
  nums = toHashSet([1, 2, 3, 4])
  parity = map(nums, proc (x: int): bool = x mod 2 == 0)
assert len(parity) == 2  # {true, false} — smaller than the source set
```

---

### `hash` (HashSet)

```nim
proc hash*[A](s: HashSet[A]): Hash
```

**Purpose.** Computes a hash code for the whole set — this lets a `HashSet` be used as a key in another hash-based container (for example a `Table`, or another `HashSet`).

**Implementation notes.** The hash codes of all occupied slots are combined with `xor`, so the result doesn't depend on traversal order — an important property, since `HashSet` order is unspecified anyway, and equal sets must produce equal hashes.

- **Parameters:**
  - `s: HashSet[A]` — the set.

**Example:**

```nim
let
  a = toHashSet([1, 2, 3])
  b = toHashSet([3, 2, 1])
assert hash(a) == hash(b)  # equal sets hash equally regardless of order
```

---

### `hash` (OrderedSet)

```nim
proc hash*[A](s: OrderedSet[A]): Hash
```

**Purpose.** The `OrderedSet` counterpart of the previous procedure, with the difference that it uses the `!&` bitwise combiner, which is sensitive to traversal order — consistent with the fact that `==` for `OrderedSet` is also order-sensitive.

- **Parameters:**
  - `s: OrderedSet[A]` — the set.

**Example:**

```nim
let
  a = toOrderedSet([1, 2])
  b = toOrderedSet([2, 1])
assert hash(a) != hash(b)  # different order — almost certainly a different hash
```

---

### `` `$` `` (HashSet)

```nim
proc `$`*[A](s: HashSet[A]): string
```

**Purpose.** Converts the set to a string of the form `{element1, element2, ...}` — mainly for logging and debug printing. Not meant for serialization: the representation may change in future versions, and string values inside are not escaped.

- **Parameters:**
  - `s: HashSet[A]` — the set.

**Example:**

```nim
echo toHashSet([2, 4, 5])
# --> {2, 4, 5}
echo toHashSet(["no", "esc'aping", "is \" provided"])
# --> {no, esc'aping, is " provided}
```

---

### `` `$` `` (OrderedSet)

```nim
proc `$`*[A](s: OrderedSet[A]): string
```

**Purpose.** The `OrderedSet` counterpart of the previous procedure; the order of elements in the string representation matches insertion order.

- **Parameters:**
  - `s: OrderedSet[A]` — the set.

**Example:**

```nim
echo toOrderedSet([2, 4, 5])
# --> {2, 4, 5}
```

---

## X. Deprecated procedures

These three procedures are kept only for backward compatibility with code written before Nim 0.20 and are marked `{.deprecated.}`. They should not be used in new code.

### `initSet`

```nim
proc initSet*[A](initialSize = defaultInitialSize): HashSet[A] {.deprecated.}
```

**Purpose.** A deprecated alias for `initHashSet`. The compiler emits a warning when it's used.

---

### `toSet`

```nim
proc toSet*[A](keys: openArray[A]): HashSet[A] {.deprecated.}
```

**Purpose.** A deprecated alias for `toHashSet`.

---

### `isValid`

```nim
proc isValid*[A](s: HashSet[A]): bool {.deprecated.}
```

**Purpose.** Used to check whether `init`/`initHashSet` had been explicitly called on a set before use. Since Nim 0.20, sets are initialized by default, so the check has lost its meaning and is kept only for old code.

---

## XI. Practical recipes

### 1. Deduplicating a sequence

```nim
proc uniqueValues[A](xs: openArray[A]): seq[A] =
  ## Removes duplicate values, preserving first-occurrence order.
  let seen = toOrderedSet(xs)  # OrderedSet keeps first occurrences in order
  result = newSeq[A]()
  for x in items(seen):
    add(result, x)

echo uniqueValues([3, 1, 3, 2, 1, 4])  # prints @[3, 1, 2, 4]
```

---

### 2. Finding elements common to several collections

```nim
proc commonToAll[A](groups: seq[seq[A]]): HashSet[A] =
  ## Finds the elements present in every group at once.
  if len(groups) == 0:
    return initHashSet[A]()
  result = toHashSet(groups[0])
  for i in 1 ..< len(groups):
    result = intersection(result, toHashSet(groups[i]))

let
  team1 = @["Ann", "Bob", "Carl"]
  team2 = @["Bob", "Carl", "Dan"]
  team3 = @["Carl", "Bob"]
echo commonToAll(@[team1, team2, team3])  # prints {Bob, Carl} (order not guaranteed)
```

---

### 3. Tracking already-processed elements (a visited cache)

```nim
proc processOnce[A](items: openArray[A], visited: var HashSet[A]) =
  ## Processes only the elements not already present in `visited`.
  for item in items:
    if not containsOrIncl(visited, item):
      echo "Processing: ", item
    # if containsOrIncl returned true, the element was already seen — skip it

var seen = initHashSet[int]()
processOnce([1, 2, 2, 3, 1], seen)
# --> Processing: 1
# --> Processing: 2
# --> Processing: 3
```

---

### 4. Comparing two versions of a configuration

```nim
proc diffConfigs(old, new: openArray[string]): tuple[added, removed: HashSet[string]] =
  ## Shows which keys appeared and which disappeared between two versions.
  let
    oldSet = toHashSet(old)
    newSet = toHashSet(new)
  result.added = difference(newSet, oldSet)
  result.removed = difference(oldSet, newSet)

let (added, removed) = diffConfigs(
  ["timeout", "retries", "host"],
  ["timeout", "host", "max_connections"])
echo "Added: ", added      # --> {max_connections}
echo "Removed: ", removed  # --> {retries}
```

---

### 5. A scheduler that preserves queueing order

```nim
proc scheduleTasks(order: var OrderedSet[string], task: string) =
  ## Adds a task to the queue if it isn't there yet — queueing order is preserved.
  incl(order, task)

proc runNext(order: var OrderedSet[string]): string =
  ## Takes the first task by queueing order and removes it from the queue.
  for t in items(order):
    result = t
    excl(order, t)
    return result
  raise newException(ValueError, "queue is empty")

var queue = initOrderedSet[string]()
scheduleTasks(queue, "build")
scheduleTasks(queue, "test")
scheduleTasks(queue, "build")  # duplicate — the queue doesn't change
echo runNext(queue)  # prints "build"
echo runNext(queue)  # prints "test"
```

---

## XII. Quick reference table

| Task | Mutates the argument | Returns a new value |
|---|---|---|
| Build a set from a collection | — | `toHashSet`, `toOrderedSet` |
| Test membership | — | `contains`, `in` |
| Find the size | — | `len`, `card` |
| Add one element | `incl(s, key)` | — |
| Add and learn whether it was already there | `containsOrIncl(s, key)` | — |
| Remove one element | `excl(s, key)` | — |
| Remove and learn whether it was already absent | `missingOrExcl(s, key)` | — |
| Extract an arbitrary element | `pop(s)` | — |
| Clear the set | `clear(s)` | — |
| Union two sets | `incl(s, other)` | `union`, `` `+` `` |
| Intersect two sets | — | `intersection`, `` `*` `` |
| Find the difference | `excl(s, other)` | `difference`, `` `-` `` |
| Find the symmetric difference | — | `symmetricDifference`, `` `-+-` `` |
| Test for no overlap | — | `disjoint` |
| Test for subset | — | `` `<=` ``, `` `<` `` |
| Compare for equality | — | `` `==` `` |
| Transform elements | — | `map` |
| Get a string representation | — | `` `$` `` |
| Iterate elements (unordered) | — | `items` on `HashSet` |
| Iterate elements (insertion order) | — | `items`/`pairs` on `OrderedSet` |

---

## XIII. Summary: which procedure to pick

- Need to quickly drop duplicates and order doesn't matter → `toHashSet`.
- Need to drop duplicates but the order of first appearance matters → `toOrderedSet`.
- Just need to check "is this element present" → `contains` or the `in` operator.
- Need to add an element and learn in the same step whether it was new → `containsOrIncl`, rather than `incl` plus a separate `contains`.
- Need to remove an element and learn whether there was anything to remove → `missingOrExcl`, rather than `excl` plus a separate `contains`.
- Need to pull out "any" element and remove it from the set at the same time → `pop`.
- Need to union/intersect/subtract sets without touching the sources → `union`/`intersection`/`difference`/`symmetricDifference` (or the `+`/`*`/`-`/`-+-` operators).
- Need the same thing but modifying one of the sets in place → `incl(s, other)` instead of `+`, `excl(s, other)` instead of `-`.
- Need to check that two sets share no elements → `disjoint`.
- Need to compare sets by contents, insertion order doesn't matter → `HashSet` and its `==`.
- Need to compare sets by both contents and insertion order → `OrderedSet` and its `==`.
- Need to transform the element type of a set → `map`.
- Need to use a set as a key in another container → `hash` is already defined for both types, no need to call it separately.
- Ran into `initSet`, `toSet`, or `isValid` in old code → replace them with `initHashSet`, `toHashSet`, and drop the `isValid` call entirely — since 0.20 it's not needed.
