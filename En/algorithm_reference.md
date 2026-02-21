# algorithm.nim — Module Reference

> **Import:** `import std/algorithm`  
> **Scope:** Generic algorithms operating on `openArray[T]` — works with arrays, sequences, and any contiguous container.

This module is the standard Nim toolkit for the most common array-oriented operations: sorting, searching, filling, reversing, rotating, and generating permutations. Almost every procedure comes in two flavours: an **in-place** mutating version and a **non-mutating** version that returns a new `seq`. Knowing which is which at a glance is easy — non-mutating names end in `-ed` (e.g. `reversed`, `rotatedLeft`, `sorted`).

---

## Types and Utilities

### `SortOrder`

```nim
type SortOrder* = enum
  Descending, Ascending
```

An enum used across the entire module to control the direction of ordered operations. Pass `Ascending` (the default almost everywhere) or `Descending` to flip the result.

---

### `*` (order multiplier)

```nim
proc `*`*(x: int, order: SortOrder): int
```

**What it does.** A small arithmetic helper that flips the sign of a comparator result when `order == Descending`, leaving it unchanged when `order == Ascending`. This lets the sorting internals reuse a single comparison function for both directions without branching.

You will rarely call this directly, but it is useful when writing your own ordered algorithms that piggyback on `SortOrder`.

```nim
# Comparator returned +5 (meaning "left > right")
assert 5  * Ascending  ==  5   # ascending: keep sign → still "left > right"
assert 5  * Descending == -5   # descending: flip sign → now "left < right"
assert -3 * Descending ==  3
```

---

## Filling

### `fill` (slice)

```nim
proc fill*[T](a: var openArray[T], first, last: Natural, value: T)
```

**What it does.** Writes `value` into every element of `a` within the index range `first..last` (both inclusive). Elements outside that range are untouched. Raises `IndexDefect` if the range is out of bounds.

Think of it as a targeted "paint bucket" for a sub-range of an array.

```nim
var a = [0, 0, 0, 0, 0, 0]
a.fill(1, 4, 7)
assert a == [0, 7, 7, 7, 7, 0]
```

---

### `fill` (whole container)

```nim
proc fill*[T](a: var openArray[T], value: T)
```

**What it does.** The simpler sibling — fills the **entire** container with `value`. Equivalent to calling the slice version with `first = 0` and `last = a.high`.

```nim
var a = [1, 2, 3, 4, 5]
a.fill(0)
assert a == [0, 0, 0, 0, 0]
```

---

## Reversing

### `reverse` (slice)

```nim
proc reverse*[T](a: var openArray[T], first, last: Natural)
```

**What it does.** Reverses the elements of `a` **in-place** within the sub-range `first..last`. Elements outside the range are not moved. Raises `IndexDefect` for an invalid range. Calling it twice on the same range restores the original order.

```nim
var a = [1, 2, 3, 4, 5, 6]
a.reverse(1, 4)
assert a == [1, 5, 4, 3, 2, 6]  # only indices 1..4 were reversed
```

---

### `reverse` (whole container)

```nim
proc reverse*[T](a: var openArray[T])
```

**What it does.** Reverses the entire container in-place. An empty container is handled safely.

```nim
var a = [1, 2, 3, 4, 5]
a.reverse()
assert a == [5, 4, 3, 2, 1]
```

---

### `reversed`

```nim
proc reversed*[T](a: openArray[T]): seq[T]
```

**What it does.** The non-mutating counterpart to `reverse`. Returns a new `seq[T]` containing the elements of `a` in reverse order; the original is unchanged. Safe on empty inputs.

```nim
let original = [10, 20, 30]
let rev = original.reversed()
assert rev      == @[30, 20, 10]
assert original == [10, 20, 30]   # untouched
```

---

## Searching

### `binarySearch` (with custom comparator)

```nim
proc binarySearch*[T, K](a: openArray[T], key: K,
                         cmp: proc (x: T, y: K): int): int
```

**What it does.** Performs a binary search for `key` in `a`, returning the **index** of the matching element, or `-1` if not found. The array must already be sorted according to `cmp`. Because the element type `T` and the key type `K` are independent type parameters, you can search a `seq[Person]` by a plain `string` name, for instance.

The comparator follows the standard convention: negative means "left < right", zero means equal, positive means "left > right".

Binary search is O(log n) — vastly faster than linear scan for large sorted data.

```nim
let words = ["apple", "banana", "cherry", "date"]
assert binarySearch(words, "cherry", system.cmp[string]) == 2
assert binarySearch(words, "grape",  system.cmp[string]) == -1
```

---

### `binarySearch` (default comparator)

```nim
proc binarySearch*[T](a: openArray[T], key: T): int
```

**What it does.** Shortcut that uses `system.cmp[T]` — no need to supply a comparator for standard comparable types.

```nim
assert binarySearch([0, 1, 2, 3, 4], 3) == 3
assert binarySearch([0, 1, 2, 3, 4], 9) == -1
```

---

### `lowerBound` (with comparator)

```nim
proc lowerBound*[T, K](a: openArray[T], key: K,
                       cmp: proc(x: T, k: K): int): int
```

**What it does.** In a sorted sequence, returns the index of the **first element that is ≥ key**. If all elements are smaller than `key`, returns `a.len` (one past the last element). This is the classic "left insertion point" — inserting at this index keeps the sequence sorted.

The key distinction from `binarySearch`: `lowerBound` always returns a valid insertion point, even when the exact key is absent.

```nim
let a = @[1, 2, 3, 5, 6, 7]
# 4 is not present; lowerBound tells us where it would go
assert a.lowerBound(4, system.cmp[int]) == 3   # between 3 and 5
assert a.lowerBound(3, system.cmp[int]) == 2   # points at the existing 3
```

---

### `lowerBound` (default comparator)

```nim
proc lowerBound*[T](a: openArray[T], key: T): int
```

**What it does.** Same as above using `system.cmp[T]`.

```nim
var s = @[10, 20, 30, 40]
s.insert(25, s.lowerBound(25))   # insert in sorted position
assert s == @[10, 20, 25, 30, 40]
```

---

### `upperBound` (with comparator)

```nim
proc upperBound*[T, K](a: openArray[T], key: K,
                       cmp: proc(x: T, k: K): int): int
```

**What it does.** Returns the index of the **first element that is strictly greater than `key`**. This is the "right insertion point" — inserting here keeps the sequence sorted and places the new element after any existing equal elements. Raises `IndexDefect` for an invalid range.

The difference between `lowerBound` and `upperBound` becomes visible when there are duplicates: `lowerBound` points before the run of equal elements, `upperBound` points after it.

```nim
let a = @[1, 2, 3, 3, 3, 6]
assert a.lowerBound(3, system.cmp[int]) == 2  # before the first 3
assert a.upperBound(3, system.cmp[int]) == 5  # after the last 3
```

---

### `upperBound` (default comparator)

```nim
proc upperBound*[T](a: openArray[T], key: T): int
```

**What it does.** Same as above using `system.cmp[T]`.

---

## Sorting

### `sort` (with comparator)

```nim
func sort*[T](a: var openArray[T],
              cmp: proc (x, y: T): int,
              order = SortOrder.Ascending)
```

**What it does.** Sorts `a` **in-place** using an iterative merge sort. Guarantees:
- **Stability** — equal elements keep their original relative order.
- **O(n log n)** worst case — no degenerate behaviour on adversarial inputs.
- Uses a temporary buffer of size `n / 2`.

You control the sort key by supplying `cmp`. Use Nim's `do` notation for concise inline comparators. Pass `Descending` to reverse the direction.

```nim
var people = [("Alice", 30), ("Bob", 25), ("Carol", 30)]

# Sort by age, then by name for ties
people.sort do (x, y: (string, int)) -> int:
    result = cmp(x[1], y[1])
    if result == 0: result = cmp(x[0], y[0])

assert people == [("Bob", 25), ("Alice", 30), ("Carol", 30)]
```

---

### `sort` (default comparator)

```nim
proc sort*[T](a: var openArray[T], order = SortOrder.Ascending)
```

**What it does.** Shortcut using `system.cmp[T]`. Perfect for sorting numbers, strings, or any type for which the default comparison is appropriate.

```nim
var nums = [5, 3, 1, 4, 2]
nums.sort()
assert nums == [1, 2, 3, 4, 5]

nums.sort(Descending)
assert nums == [5, 4, 3, 2, 1]
```

---

### `sorted` (with comparator)

```nim
proc sorted*[T](a: openArray[T], cmp: proc(x, y: T): int,
                order = SortOrder.Ascending): seq[T]
```

**What it does.** The non-mutating twin of `sort`. Returns a new sorted `seq[T]`; the original `a` is left intact. Useful in functional-style pipelines or when you need to preserve the original ordering alongside the sorted copy.

```nim
let original = [3, 1, 4, 1, 5]
let s = original.sorted(system.cmp[int])
assert s        == @[1, 1, 3, 4, 5]
assert original == [3, 1, 4, 1, 5]   # unchanged
```

---

### `sorted` (default comparator)

```nim
proc sorted*[T](a: openArray[T], order = SortOrder.Ascending): seq[T]
```

**What it does.** Shortcut using `system.cmp[T]`.

```nim
assert [4, 2, 7, 1].sorted == @[1, 2, 4, 7]
assert [4, 2, 7, 1].sorted(Descending) == @[7, 4, 2, 1]
```

---

### `sortedByIt`

```nim
template sortedByIt*(seq1, op: untyped): untyped
```

**What it does.** A convenience template that sorts a sequence by an expression, injecting each element as `it`. Significantly reduces boilerplate compared to writing a full comparator. The expression can be a field access, a computed value, or even a tuple for nested sort keys.

```nim
type Person = tuple[name: string, age: int]
let people = @[("Carol", 30), ("Alice", 25), ("Bob", 30)]

# Sort by age
let byAge = people.sortedByIt(it.age)
assert byAge == @[("Alice", 25), ("Carol", 30), ("Bob", 30)]

# Nested sort: age first, then name
let byAgeThenName = people.sortedByIt((it.age, it.name))
assert byAgeThenName == @[("Alice", 25), ("Bob", 30), ("Carol", 30)]
```

---

### `isSorted` (with comparator)

```nim
func isSorted*[T](a: openArray[T], cmp: proc(x, y: T): int,
                  order = SortOrder.Ascending): bool
```

**What it does.** Checks in O(n) time whether `a` is already sorted according to `cmp` and `order`. Returns `true` if no adjacent pair is out of order, `false` the moment any violation is found (short-circuits). Useful as a precondition check before binary search or as a post-condition assertion after sorting.

```nim
assert isSorted([1, 2, 3, 4])             == true
assert isSorted([1, 3, 2, 4])             == false
assert isSorted([4, 3, 2, 1], Descending) == true
```

---

### `isSorted` (default comparator)

```nim
proc isSorted*[T](a: openArray[T], order = SortOrder.Ascending): bool
```

**What it does.** Shortcut using `system.cmp[T]`.

```nim
assert @[1, 1, 2, 3].isSorted == true    # equal elements are fine
assert @["a", "c", "b"].isSorted == false
```

---

## Merging

### `merge` (with comparator)

```nim
proc merge*[T](result: var seq[T], x, y: openArray[T],
               cmp: proc(x, y: T): int)
```

**What it does.** Merges two already-sorted sequences `x` and `y` into `result`, maintaining sorted order. The merge is stable: when `cmp` returns 0 (equal), the element from `x` comes first. 

**Important:** new elements are **appended** to `result` — any existing content is preserved. Clear `result` beforehand with `result.setLen(0)` if you want a fresh merge.

```nim
let evens = @[2, 4, 6]
let odds  = @[1, 3, 5]
var merged: seq[int]
merged.merge(odds, evens, system.cmp[int])
assert merged == @[1, 2, 3, 4, 5, 6]
```

---

### `merge` (default comparator)

```nim
proc merge*[T](result: var seq[T], x, y: openArray[T])
```

**What it does.** Shortcut using `system.cmp[T]`.

```nim
var r: seq[int]
r.merge([1, 3, 5], [2, 4, 6])
assert r == @[1, 2, 3, 4, 5, 6]
```

---

## Combinatorics

### `product`

```nim
proc product*[T](x: openArray[seq[T]]): seq[seq[T]]
```

**What it does.** Computes the **Cartesian product** of a list of sequences. Each element in the result is one combination, with the i-th slot drawn from `x[i]`. The total number of results is the product of all individual sequence lengths — this can grow very quickly, so use with care on large inputs.

```nim
# All suit/value combinations for two card attributes
let suits  = @["♥", "♠"]
let values = @["A", "K"]
let combos = product(@[suits, values])
# @[@["♥", "A"], @["♠", "A"], @["♥", "K"], @["♠", "K"]]
assert combos.len == 4
```

---

### `nextPermutation`

```nim
proc nextPermutation*[T](x: var openArray[T]): bool {.discardable.}
```

**What it does.** Transforms `x` in-place into the **next lexicographic permutation**. Returns `true` if a next permutation existed, `false` if `x` was already the last (i.e. sorted in descending order). On `false`, the array is left unchanged.

To iterate over all permutations, start with the sequence in ascending order and call `nextPermutation` in a loop until it returns `false`.

```nim
var v = @[1, 2, 3]
var permutations: seq[seq[int]]
permutations.add(v)
while v.nextPermutation():
    permutations.add(v)
assert permutations.len == 6   # 3! = 6
# @[1,2,3], @[1,3,2], @[2,1,3], @[2,3,1], @[3,1,2], @[3,2,1]
```

---

### `prevPermutation`

```nim
proc prevPermutation*[T](x: var openArray[T]): bool {.discardable.}
```

**What it does.** The reverse of `nextPermutation` — transforms `x` into the **previous lexicographic permutation**. Returns `true` if it succeeded, `false` if `x` was already the first permutation (sorted ascending). The array is unchanged on `false`.

```nim
var v = @[3, 2, 1]   # last permutation of [1,2,3]
assert v.prevPermutation() == true
assert v == @[3, 1, 2]
assert v.prevPermutation() == true
assert v == @[2, 3, 1]
```

---

## Rotating

Rotation shifts elements cyclically within a range. A left rotation by `dist` moves element at index `i` to index `i - dist` (wrapping around). Right rotation is a left rotation with a negative `dist`.

### `rotateLeft` (slice, in-place)

```nim
proc rotateLeft*[T](arg: var openArray[T]; slice: HSlice[int, int];
                    dist: int): int {.discardable.}
```

**What it does.** Rotates the elements at `slice` leftward by `dist` positions, in-place. Elements outside the slice are untouched. Returns the index of what was the first element before rotation (useful for some algorithms). `dist` may be any integer — negative values rotate right, values beyond the slice length wrap around.

```nim
var a = [0, 1, 2, 3, 4, 5]
a.rotateLeft(1 .. 4, 2)
assert a == [0, 3, 4, 1, 2, 5]
# indices 0 and 5 unchanged; within 1..4: shifted left by 2
```

---

### `rotateLeft` (whole container, in-place)

```nim
proc rotateLeft*[T](arg: var openArray[T]; dist: int): int {.discardable.}
```

**What it does.** Same as above but operates on the entire container.

```nim
var a = [1, 2, 3, 4, 5]
a.rotateLeft(2)
assert a == [3, 4, 5, 1, 2]   # "3 4 5" shifted to the front

a.rotateLeft(-2)
assert a == [1, 2, 3, 4, 5]   # rotate right by 2 restores original
```

---

### `rotatedLeft` (slice, non-mutating)

```nim
proc rotatedLeft*[T](arg: openArray[T]; slice: HSlice[int, int],
                     dist: int): seq[T]
```

**What it does.** Non-mutating version of the slice rotation. Returns a new `seq[T]` with the rotation applied; `arg` is unchanged.

```nim
let a = @[1, 2, 3, 4, 5]
let b = a.rotatedLeft(1 .. 3, 2)
assert b == @[1, 4, 2, 3, 5]
assert a == @[1, 2, 3, 4, 5]   # original intact
```

---

### `rotatedLeft` (whole container, non-mutating)

```nim
proc rotatedLeft*[T](arg: openArray[T]; dist: int): seq[T]
```

**What it does.** Non-mutating rotation of the entire container.

```nim
let a = @[1, 2, 3, 4, 5]
assert a.rotatedLeft(2)  == @[3, 4, 5, 1, 2]
assert a.rotatedLeft(-1) == @[5, 1, 2, 3, 4]
assert a == @[1, 2, 3, 4, 5]   # unchanged
```

---

## Quick Reference

| Goal | In-place | Returns new seq |
|---|---|---|
| Sort | `sort` | `sorted`, `sortedByIt` |
| Reverse | `reverse` | `reversed` |
| Rotate left | `rotateLeft` | `rotatedLeft` |
| Find (exact) | `binarySearch` | — |
| Find (insertion point ≥ key) | `lowerBound` | — |
| Find (insertion point > key) | `upperBound` | — |
| Check sorted | `isSorted` | — |
| Fill range | `fill` | — |
| Merge two sorted | `merge` | — |
| Cartesian product | — | `product` |
| Next permutation | `nextPermutation` | — |
| Previous permutation | `prevPermutation` | — |
