# algorithm.nim — module reference

> **Import:** `import std/algorithm`
> **Scope:** Generic algorithms for `openArray[T]` — work with arrays, sequences, and any contiguous container, including slices obtained via `toOpenArray`.

The module implements a standard set of array algorithms: sorting, binary search, filling, reversing, rotation, permutations, merging, and Cartesian product. Almost every procedure exists in two flavors: a **mutating** one that modifies the container in place, and a **non-mutating** one that returns a new sequence. Non-mutating variants are easy to spot by the `-ed` suffix in their name: `reversed`, `rotatedLeft`, `sorted`.

Most procedures accept an optional `order: SortOrder` parameter and/or an optional comparison function `cmp`. If `cmp` is not supplied, the built-in `system.cmp[T]` is used, which works out of the box for numbers, strings, enums, characters, and tuples (including tuples composed of such types).

---

## Types and helpers

### `SortOrder`

```nim
type SortOrder* = enum
  Descending, Ascending
```

An enum that specifies the sort direction used throughout the module. `Ascending` (increasing order) is the default almost everywhere; `Descending` reverses the order.

Note the declaration order of the values: `Descending = 0`, `Ascending = 1`. This is not accidental — it's exactly what lets the `*` operator (see below) work as a simple bitwise trick without any branching.

`SortOrder` is passed to `sort`, `sorted`, `isSorted`, and is used internally by `mergeAlt` — a private helper procedure that the merge-sort implementation is built on. `mergeAlt` is not exported from the module, but knowing it exists helps explain why `order` affects the comparison of *neighboring elements* rather than a final reversal of the result: the module does **not** sort ascending and then reverse the whole thing — it compares elements "in the right direction" from the start, which is what preserves stability (the sort's stable property) in both directions.

---

### `*` (order multiplier)

```nim
proc `*`*(x: int, order: SortOrder): int
```

**What it does.** A small arithmetic helper: flips the sign of a comparator's result if `order == Descending`, and leaves it unchanged for `Ascending`. This lets sorting algorithms reuse the same comparison function for both directions without extra branches.

The implementation looks like this:

```nim
var y = order.ord - 1   # 0 → -1 for Descending, 0 → 0 for Ascending
result = (x xor y) - y
```

For `order == Ascending` (`y == 0`) the expression reduces to `(x xor 0) - 0 == x` — the number is unchanged. For `order == Descending` (`y == -1`, i.e. all bits set), `xor` flips every bit of `x` (equivalent to `not x == -x - 1`), and subtracting `-1` (i.e. adding `1`) yields `-x`. The end result is the classic "negate" operation, but performed without a conditional and without the overflow risk of `low(int)` for the typical comparator return values (`-1`, `0`, `1`).

You'll rarely need to call this directly, but it's handy when writing your own algorithms that rely on `SortOrder` — for example, a custom `isSorted` for a specialized data structure, or a custom binary search that supports both directions.

```nim
# Comparator returned +5 (left > right)
assert 5  * Ascending  ==  5   # order unchanged
assert 5  * Descending == -5   # sign flipped → left is now "less than"
assert -3 * Descending ==  3
assert 0  * Descending ==  0   # zero stays zero in both directions

# Practical use: a generic descending comparator
proc cmpDesc[T](x, y: T): int =
  cmp(x, y) * Descending

assert cmpDesc(1, 2) > 0   # from a "descending" point of view, 1 is "greater" than 2
```

---

## Filling

### `fill` (slice)

```nim
proc fill*[T](a: var openArray[T], first, last: Natural, value: T)
```

**What it does.** Writes `value` into every element of `a` within the index range `first..last`, inclusive. Elements outside the range are left untouched. If the range is out of bounds (e.g. `last >= a.len`), an `IndexDefect` is raised.

You can picture this as "painting" a fixed-width section of the array — the brush runs strictly from `first` to `last`.

Parameters:

- `a` — the container to mutate in place;
- `first`, `last` — the bounds of the range (both inclusive, `Natural`, i.e. non-negative);
- `value` — the value written into the range.

```nim
var a = [0, 0, 0, 0, 0, 0]
a.fill(1, 4, 7)
assert a == [0, 7, 7, 7, 7, 0]

# Filling an overlapping range — the last call wins
a.fill(3, 5, 9)
assert a == [0, 7, 7, 9, 9, 9]

# A single-element range
a.fill(0, 0, -1)
assert a == [-1, 7, 7, 9, 9, 9]

# Out-of-bounds range -> runtime error
doAssertRaises(IndexDefect):
  a.fill(2, 10, 0)
```

---

### `fill` (whole container)

```nim
proc fill*[T](a: var openArray[T], value: T)
```

**What it does.** A simplified variant: fills the **entire** container with `value`. Equivalent to calling `fill(a, 0, a.high, value)`. For an empty container (`a.len == 0`) the call is safe and does nothing, because `a.high == -1`, and the range `first..last` simply contains no positions to iterate over.

```nim
var a = [1, 2, 3, 4, 5]
a.fill(0)
assert a == [0, 0, 0, 0, 0]

# Commonly used to "reset" a buffer before reuse
var buffer = newSeq[int](10)
for i in 0..<10: buffer[i] = i * i
buffer.fill(0)
assert buffer == newSeq[int](10)

# Works with any type, including strings and objects
var names = ["Anna", "Boris", "Victor"]
names.fill("???")
assert names == ["???", "???", "???"]
```

---

## Reversing

### `reverse` (slice)

```nim
proc reverse*[T](a: var openArray[T], first, last: Natural)
```

**What it does.** Reverses the elements of `a` **in place** within the sub-range `first..last`. Elements outside the range are left untouched. Calling it twice with the same range restores the original order (the operation is its own inverse — an *involution*). Raises `IndexDefect` for an invalid range (e.g. `last >= a.len`).

The algorithm is the classic two-pointer approach — `x = first` and `y = last` walk toward each other, swapping (`swap`) as they go, while `x < y`. If the range has an odd number of elements, the middle element stays put — it's never swapped with anything.

```nim
var a = [1, 2, 3, 4, 5, 6]
a.reverse(1, 4)
assert a == [1, 5, 4, 3, 2, 6]  # only indices 1..4 are reversed

# Calling it again with the same range restores the order
a.reverse(1, 4)
assert a == [1, 2, 3, 4, 5, 6]

# A single-element range leaves the array unchanged
a.reverse(2, 2)
assert a == [1, 2, 3, 4, 5, 6]

# Reversing the whole array via explicit bounds
a.reverse(0, a.high)
assert a == [6, 5, 4, 3, 2, 1]
```

---

### `reverse` (whole container)

```nim
proc reverse*[T](a: var openArray[T])
```

**What it does.** Reverses the entire container in place. An empty container is handled safely: internally it calls `reverse(a, 0, max(0, a.high))`, and `max` protects against accessing index `-1` when `a.high == -1` for an empty array.

```nim
var a = [1, 2, 3, 4, 5]
a.reverse()
assert a == [5, 4, 3, 2, 1]

# Empty and single-element containers are safe
var empty: seq[int] = @[]
empty.reverse()
assert empty == @[]

var single = @[42]
single.reverse()
assert single == @[42]

# Common pattern: reversing a string represented as seq[char]
var chars = @['n', 'i', 'm']
chars.reverse()
assert chars == @['m', 'i', 'n']
```

---

### `reversed`

```nim
proc reversed*[T](a: openArray[T]): seq[T]
```

**What it does.** The non-mutating counterpart of `reverse`. Returns a new `seq[T]` with the elements in reverse order; the source container is not touched. Safe for empty input — for `a.len == 0` it returns `@[]`.

The implementation pre-allocates the result (`setLen(n)`) and fills it in a single linear pass via `for i in 0..<n: result[i] = a[n - (i + 1)]`, so it runs in one pass with no intermediate copies and without calling `reverse` under the hood.

```nim
let original = [10, 20, 30]
let rev = original.reversed()
assert rev      == @[30, 20, 10]
assert original == [10, 20, 30]   # unchanged

# Convenient in computation chains
import std/sequtils
let result = @[1, 2, 3, 4, 5].reversed().filterIt(it mod 2 == 0)
assert result == @[4, 2]

# Empty sequence
assert seq[string].default.reversed == @[]
```

> ⚠️ **Deprecated overload.** There is also
> `reversed[T](a: openArray[T], first: Natural, last: int): seq[T]`,
> which is marked `deprecated` and emits a compile-time warning.
> Use `reversed(toOpenArray(a, first, last))` instead — this makes it
> explicit that a slice is taken first and then reversed:
>
> ```nim
> let a = [1, 2, 3, 4, 5]
> # Instead of the deprecated a.reversed(1, 3)
> let part = reversed(toOpenArray(a, 1, 3))
> assert part == @[4, 3, 2]
> ```

---

## Searching

### `binarySearch` (with a custom comparator)

```nim
proc binarySearch*[T, K](a: openArray[T], key: K,
                         cmp: proc (x: T, y: K): int): int
```

**What it does.** Performs a binary search for `key` in the sorted array `a`. Returns the **index** of the found element, or `-1` if it's not found. The array must already be sorted in the order implied by `cmp` — otherwise the result is unpredictable (the algorithm does not check sortedness for you; see `isSorted`).

The types `T` and `K` are independent: you can search a `seq[Person]` using a plain name string, without constructing a temporary `Person`. The comparator follows the standard convention: a negative result means "left < right", zero means "equal", and a positive result means "left > right".

Binary search runs in O(log n) — dramatically faster than a linear scan on large sorted data. The module's implementation includes an extra optimization: if the array length is a power of two, a variant using bit-shift (`shr`) is used instead of division, which is slightly faster on some platforms.

```nim
let words = ["apple", "banana", "cherry", "date"]
assert binarySearch(words, "cherry", system.cmp[string]) == 2
assert binarySearch(words, "grape",  system.cmp[string]) == -1

# Searching by a struct field (K differs from T)
type Employee = tuple[name: string, id: int]
let staff = [("Alice", 101), ("Bob", 205), ("Carol", 310)]

proc byId(e: Employee, id: int): int = cmp(e.id, id)

assert staff.binarySearch(205, byId) == 1
assert staff.binarySearch(999, byId) == -1

# Searching with reverse order — the comparator must match
# the array's actual order!
let desc = [9, 7, 5, 3, 1]
proc cmpDescInt(x, y: int): int = cmp(y, x)  # flipped comparison
assert desc.binarySearch(5, cmpDescInt) == 2
```

---

### `binarySearch` (default comparator)

```nim
proc binarySearch*[T](a: openArray[T], key: T): int
```

**What it does.** A simplified variant that uses `system.cmp[T]` — no need to supply a comparator for standard comparable types (numbers, strings, characters, enums, tuples of these).

```nim
assert binarySearch([0, 1, 2, 3, 4], 3) == 3
assert binarySearch([0, 1, 2, 3, 4], 9) == -1

# Also works with strings
let langs = ["c", "go", "nim", "rust", "zig"]
assert langs.binarySearch("nim") == 2

# Searching a single-element and an empty array — edge cases
# handled by a dedicated branch in the algorithm
assert binarySearch([42], 42) == 0
assert binarySearch([42], 0)  == -1
assert binarySearch(seq[int].default, 1) == -1
```

---

### `lowerBound` (with a comparator)

```nim
proc lowerBound*[T, K](a: openArray[T], key: K,
                       cmp: proc(x: T, k: K): int): int
```

**What it does.** In a sorted sequence, returns the index of the **first element that is ≥ key** (the "left insertion bound"). If every element is less than `key`, returns `a.len` — the position past the last element. Inserting at this index (`insert(thing, elm, lowerBound(thing, elm))`) keeps the sequence sorted.

The key difference from `binarySearch`: `lowerBound` **always** returns a valid insertion position, even when the searched-for element is absent, whereas `binarySearch` would return `-1` in that case. In other words, `lowerBound` answers "where should this go", while `binarySearch` answers "does this element exist, and if so, where exactly".

Algorithmically this is a classic "half-open" binary search: on each step the range `[result, result+count)` is halved, where `count` always points to the first element that is **definitely not less** than `key`.

```nim
let a = @[1, 2, 3, 5, 6, 7]
# 4 is absent — lowerBound shows where to insert it
assert a.lowerBound(4, system.cmp[int]) == 3   # between 3 and 5
assert a.lowerBound(3, system.cmp[int]) == 2   # points at the existing 3

# If key is less than every element — position 0
assert a.lowerBound(0, system.cmp[int]) == 0

# If key is greater than every element — position a.len
assert a.lowerBound(100, system.cmp[int]) == a.len

# Using a custom comparator (search by field)
type Event = tuple[time: int, name: string]
let timeline = @[(10, "start"), (20, "processing"), (30, "finish")]
proc byTime(e: Event, t: int): int = cmp(e.time, t)
assert timeline.lowerBound(20, byTime) == 1
```

---

### `lowerBound` (default comparator)

```nim
proc lowerBound*[T](a: openArray[T], key: T): int
```

**What it does.** Same as above, using `system.cmp[T]`.

```nim
var s = @[10, 20, 30, 40]
s.insert(25, s.lowerBound(25))   # insert at the right position
assert s == @[10, 20, 25, 30, 40]

# Keeping a sequence sorted while inserting elements one by one
var sortedSeq: seq[int] = @[]
for x in [5, 1, 4, 2, 3]:
  sortedSeq.insert(x, sortedSeq.lowerBound(x))
assert sortedSeq == @[1, 2, 3, 4, 5]
```

---

### `upperBound` (with a comparator)

```nim
proc upperBound*[T, K](a: openArray[T], key: K,
                       cmp: proc(x: T, k: K): int): int
```

**What it does.** Returns the index of the **first element strictly greater than `key`** (the "right insertion bound"). Inserting at this index (`insert(thing, elm, upperBound(thing, elm))`) keeps the sequence sorted and places the new element **after** all existing elements equal to `key`.

The difference between `lowerBound` and `upperBound` only matters when duplicates are present: `lowerBound` points **before** a run of equal elements, `upperBound` points **after** it. If there are no duplicates, both values coincide.

A useful consequence: `upperBound(key) - lowerBound(key)` equals the **number of elements equal to `key`** in the sorted array — a convenient O(log n) way to count occurrences without a full scan.

```nim
let a = @[1, 2, 3, 3, 3, 6]
assert a.lowerBound(3, system.cmp[int]) == 2  # before the first 3
assert a.upperBound(3, system.cmp[int]) == 5  # after the last 3

# Counting occurrences in O(log n)
let count3 = a.upperBound(3, system.cmp[int]) - a.lowerBound(3, system.cmp[int])
assert count3 == 3

# For a value with no duplicates, lowerBound == upperBound - 1
assert a.lowerBound(2, system.cmp[int]) == a.upperBound(2, system.cmp[int]) - 1
```

---

### `upperBound` (default comparator)

```nim
proc upperBound*[T](a: openArray[T], key: T): int
```

**What it does.** Same as above, using `system.cmp[T]`.

```nim
let scores = @[60, 70, 70, 70, 85, 90]
# How many elements are less than or equal to 70?
assert scores.upperBound(70) == 4

# Inserting a duplicate always lands at the end of the equal-value group
var s = @[1, 3, 3, 5]
s.insert(3, s.upperBound(3))
assert s == @[1, 3, 3, 3, 5]
```

---

## Sorting

### `sort` (with a comparator)

```nim
func sort*[T](a: var openArray[T],
              cmp: proc (x, y: T): int,
              order = SortOrder.Ascending)
```

**What it does.** Sorts `a` **in place** using an iterative merge sort. Guarantees:

- **Stability** — elements that are "equal" according to `cmp` keep the relative order they had before sorting. This matters for multi-level sorting: after sorting once by one key, sorting again by a different key preserves the previous ordering among elements that tie on the new key (though for clarity it's usually better to compare a tuple of keys explicitly).
- **O(n log n) worst case** — no degradation on "adversarial" inputs (e.g. already reverse-sorted or specially crafted data), unlike, for example, naive quicksort.
- Uses a **temporary buffer of size `n div 2`** — i.e. requires extra memory roughly half the size of the input, unlike fully in-place algorithms such as heapsort, but gains stability in exchange.
- Includes an optimization: if the last element of the left half is `<=` the first element of the right half (with respect to `order`), the merge step for that pair of blocks is skipped entirely — on already (partially) sorted data this gives a substantial speedup (roughly up to ~40% on random data according to the source comment, and significantly more on "nearly sorted" data).

The sort criterion is given by `cmp`. The `do` syntax lets you write the comparator inline at the call site without a separate declaration. The `order` parameter is applied to the result of `cmp` via the `*` operator described above — so there's **no need** to manually invert `cmp` for descending order.

```nim
var people = [("Alice", 30), ("Bob", 25), ("Carol", 30)]

# Sort by age first, then by name on ties
people.sort do (x, y: (string, int)) -> int:
    result = cmp(x[1], y[1])
    if result == 0: result = cmp(x[0], y[0])

assert people == [("Bob", 25), ("Alice", 30), ("Carol", 30)]

# Descending sort using the same comparator — thanks to order,
# the comparator itself doesn't need to be rewritten
var nums = [3, 1, 2]
nums.sort(system.cmp[int], Descending)
assert nums == [3, 2, 1]

# Demonstrating stability: elements with the same key
# keep their original relative order
type Item = tuple[group: int, original: int]
var items = @[(1, 0), (2, 1), (1, 2), (2, 3), (1, 4)]
items.sort(proc (x, y: Item): int = cmp(x.group, y.group))
assert items == @[(1, 0), (1, 2), (1, 4), (2, 1), (2, 3)]
# within group 1 the order 0,2,4 is preserved; within group 2 the order 1,3 is preserved
```

---

### `sort` (default comparator)

```nim
proc sort*[T](a: var openArray[T], order = SortOrder.Ascending)
```

**What it does.** A shortcut using `system.cmp[T]` — sufficient for numbers, strings, characters, enums, and tuples of these with standard comparison. Equivalent to `sort(a, system.cmp[T], order)`.

```nim
var nums = [5, 3, 1, 4, 2]
nums.sort()
assert nums == [1, 2, 3, 4, 5]

nums.sort(Descending)
assert nums == [5, 4, 3, 2, 1]

# Sorting strings — lexicographic order (by UTF-8 byte values)
var words = @["banana", "Avocado", "cherry"]
words.sort()
echo words   # order depends on character codes; uppercase letters sort before lowercase in ASCII/UTF-8

# Sorting tuples compares element-by-element, "left to right"
var pairs = @[(2, "b"), (1, "z"), (1, "a")]
pairs.sort()
assert pairs == @[(1, "a"), (1, "z"), (2, "b")]
```

---

### `sorted` (with a comparator)

```nim
proc sorted*[T](a: openArray[T], cmp: proc(x, y: T): int,
                order = SortOrder.Ascending): seq[T]
```

**What it does.** The non-mutating twin of `sort`. Returns a new sorted `seq[T]`; the source container is left untouched. The implementation simply copies the elements of `a` into a new `seq` and calls `sort` on the copy — so the asymptotic complexity is the same (O(n log n)), plus an O(n) copy. Handy in functional-style chains, or when you need to keep both the original and the sorted version.

```nim
let original = [3, 1, 4, 1, 5]
let s = original.sorted(system.cmp[int])
assert s        == @[1, 1, 3, 4, 5]
assert original == [3, 1, 4, 1, 5]   # unchanged

# Descending sort without modifying the source array
let topDown = original.sorted(system.cmp[int], Descending)
assert topDown == @[5, 4, 3, 1, 1]

# Sorting with a custom comparator without modifying the source data
type Task = tuple[priority: int, title: string]
let tasks = @[(2, "B"), (1, "A"), (3, "C")]
let byPriority = tasks.sorted(proc(x, y: Task): int = cmp(x.priority, y.priority))
assert byPriority == @[(1, "A"), (2, "B"), (3, "C")]
assert tasks == @[(2, "B"), (1, "A"), (3, "C")]  # original order preserved
```

---

### `sorted` (default comparator)

```nim
proc sorted*[T](a: openArray[T], order = SortOrder.Ascending): seq[T]
```

**What it does.** A shortcut using `system.cmp[T]`.

```nim
assert [4, 2, 7, 1].sorted == @[1, 2, 4, 7]
assert [4, 2, 7, 1].sorted(Descending) == @[7, 4, 2, 1]

# Often combined with other procedures in the module
let data = @[5, 3, 1, 4, 2]
assert data.sorted.isSorted == true
assert data.sorted == data.reversed.sorted  # sorting "forgets" the original order
```

---

### `sortedByIt`

```nim
template sortedByIt*(seq1, op: untyped): untyped
```

**What it does.** A convenience template built around `sorted`. It injects a variable `it`, which takes on each element of `seq1` in turn, and sorts by the value of the expression `op` evaluated for `it`. This avoids writing a separate comparator function when sorting by a struct field or by a tuple of keys.

Under the hood the template builds a comparator of the form `cmp(op(x), op(y))`, so the type of `op` must support `cmp` — true for numbers, strings, and tuples of such types. If `op` is a tuple `(a, b)`, the sort becomes "multi-level": first by `a`, then by `b` for ties, and so on, exactly the way tuples are compared in Nim.

Since the template calls `sorted`, it does **not** modify the source sequence and returns a new one.

```nim
type Person = tuple[name: string, age: int]
let people = @[("Carol", 30), ("Alice", 25), ("Bob", 30)]

# By age
let byAge = people.sortedByIt(it.age)
assert byAge == @[("Alice", 25), ("Carol", 30), ("Bob", 30)]

# By age first, then by name (multi-level sort)
let byAgeThenName = people.sortedByIt((it.age, it.name))
assert byAgeThenName == @[("Alice", 25), ("Bob", 30), ("Carol", 30)]

# Sorting by a computed expression (string length)
let words = @["fork", "knife", "tablespoon", "spoon"]
let byLen = words.sortedByIt(it.len)
assert byLen == @["fork", "spoon", "knife", "tablespoon"]

# Descending: wrap the cmp result with * Descending
import std/algorithm as algo
let byAgeDesc = sorted(people, proc(x, y: Person): int =
  cmp(x.age, y.age) * Descending)
assert byAgeDesc == @[("Carol", 30), ("Bob", 30), ("Alice", 25)]
```

---

### `isSorted` (with a comparator)

```nim
func isSorted*[T](a: openArray[T], cmp: proc(x, y: T): int,
                  order = SortOrder.Ascending): bool
```

**What it does.** Checks in O(n) time whether `a` is sorted according to `cmp` and `order`. Returns `true` if no pair of **adjacent** elements violates the order, and `false` as soon as the first violation is found (early exit — for a large unsorted array the check can finish almost instantly). An empty array and a single-element array are always considered sorted (`true`), since there are no pairs to compare.

Useful as a precondition before `binarySearch`/`lowerBound`/`upperBound` (which require sortedness but don't verify it), or as a postcondition after sorting in tests.

```nim
assert isSorted([1, 2, 3, 4])             == true
assert isSorted([1, 3, 2, 4])             == false
assert isSorted([4, 3, 2, 1], Descending) == true

# Equal adjacent elements don't violate sortedness (non-strict order)
assert isSorted([1, 1, 2, 2, 3])          == true

# Checking with a custom comparator by field
type Record = tuple[id: int, name: string]
let recs = @[(1, "A"), (2, "B"), (2, "C"), (5, "D")]
assert recs.isSorted(proc(x, y: Record): int = cmp(x.id, y.id)) == true
```

---

### `isSorted` (default comparator)

```nim
proc isSorted*[T](a: openArray[T], order = SortOrder.Ascending): bool
```

**What it does.** A shortcut using `system.cmp[T]`.

```nim
assert @[1, 1, 2, 3].isSorted == true    # equal elements are allowed
assert @["a", "c", "b"].isSorted == false

# Empty and single-element arrays are always "sorted"
assert seq[int].default.isSorted == true
assert @[42].isSorted == true

# A typical assertion after sort
var data = @[3, 1, 2]
data.sort()
assert data.isSorted
```

---

## Merging

### `merge` (with a comparator)

```nim
proc merge*[T](result: var seq[T], x, y: openArray[T],
               cmp: proc(x, y: T): int)
```

**What it does.** Merges two already-sorted arrays `x` and `y` into `result`, preserving the overall sorted order (the classic merge-sort step, available as a standalone procedure). The merge is **stable**: when elements compare equal under `cmp`, the element from `x` comes first in the result — this preserves the relative order of elements that were considered equal.

**Important:** new elements are **appended to the end** of `result` — existing content is preserved (`result.setLen` grows the sequence rather than recreating it). To get only the merged result, clear `result` first with `result.setLen(0)`.

The algorithm runs in O(len(x) + len(y)) — linear in the combined size, with no binary search involved: two pointers, one per input array, advance as the smaller of the two current elements is "taken".

Available since Nim **1.5.1** (marked with the `{.since: (1, 5, 1).}` pragma).

```nim
let evens = @[2, 4, 6]
let odds = @[1, 3, 5]
var merged: seq[int]
merged.merge(odds, evens, system.cmp[int])
assert merged == @[1, 2, 3, 4, 5, 6]

# Appending to existing data
var withPrefix = @[0]
withPrefix.merge(@[1, 3], @[2, 4], system.cmp[int])
assert withPrefix == @[0, 1, 2, 3, 4]   # 0 stays first, followed by the merged data

# Merging by a custom key (e.g. by the second tuple field)
type Pair = (int, int)
var res: seq[Pair]
res.merge([(1, 1), (3, 1)], [(2, 2), (4, 2)],
          proc(a, b: Pair): int = cmp(a[0], b[0]))
assert res == @[(1, 1), (2, 2), (3, 1), (4, 2)]

# Using dup for a functional style
import std/sugar
assert seq[int].default.dup(merge([1, 3], [2, 4], system.cmp[int])) == @[1, 2, 3, 4]
```

---

### `merge` (default comparator)

```nim
proc merge*[T](result: var seq[T], x, y: openArray[T])
```

**What it does.** A shortcut using `system.cmp[T]`.

```nim
var r: seq[int]
r.merge([1, 3, 5], [2, 4, 6])
assert r == @[1, 2, 3, 4, 5, 6]
assert r.isSorted

# Merging the results of two independent sorts
var a = @[5, 1, 3]
var b = @[6, 2, 4]
a.sort()
b.sort()
var combined: seq[int]
combined.merge(a, b)
assert combined == @[1, 2, 3, 4, 5, 6]
```

---

## Combinatorics

### `product`

```nim
proc product*[T](x: openArray[seq[T]]): seq[seq[T]]
```

**What it does.** Computes the **Cartesian product** of a collection of sequences. Each element of the result is one combination (a `seq[T]` of length `x.len`), where slot `i` is filled with an element from `x[i]`. The number of results equals the product of the lengths of all input sequences — it can grow very fast (exponentially in the number of factors), so use it carefully on large inputs. If any input sequence is empty, the result is empty (`@[]`) — no combinations exist.

Note: the implementation iterates indices "from the end", so the **order of combinations in the result may differ** from the "natural" left-to-right lexicographic order over indices — visible in the example below, where the first combination uses the last elements of the input sequences. If order matters for your use case, sort the result separately.

```nim
# All combinations of suit and rank for a card
let suits = @["♥", "♠"]
let ranks = @["A", "K"]
let combos = product(@[suits, ranks])
# @[@["♠", "K"], @["♥", "K"], @["♠", "A"], @["♥", "A"]]  (order may vary)
assert combos.len == 4

# Three factors — 2 * 2 * 3 = 12 combinations
let result = product(@[@["x", "y"], @["1", "2"], @["a", "b", "c"]])
assert result.len == 12
for combo in result:
  assert combo.len == 3   # each combination has exactly one element from each factor

# An empty factor -> empty result
assert product(@[@[1, 2], @[Natural](newSeq[int]())]).len == 0

# A single factor -> a copy of its elements, each wrapped as its own "combination"
assert product(@[@[1, 2, 3]]) == @[@[1], @[2], @[3]]
```

---

### `nextPermutation`

```nim
proc nextPermutation*[T](x: var openArray[T]): bool {.discardable.}
```

**What it does.** Transforms `x` in place into the **next lexicographic permutation**, relative to the current arrangement of its elements (using `cmp[T]`). Returns `true` if such a permutation existed; `false` if `x` was already the last one (sorted in descending order) — in this case the array is **left unchanged**.

The algorithm (Narayana's "next permutation") runs in O(n):

1. Scan from the end and find the position `i` where the longest decreasing "tail" begins, i.e. `x[i-1] < x[i]`.
2. If no such position exists (the whole array is decreasing) — this is the last permutation, return `false`.
3. In the tail, find the rightmost element `x[j]` that is greater than `x[i-1]`, and swap `x[i-1]` and `x[j]`.
4. Reverse the tail `x[i..^1]` so it becomes minimal again (sorted ascending).

To iterate over **all** permutations of a set of `n` distinct elements, start from a sequence sorted in ascending order and call `nextPermutation` in a loop — this yields all `n!` permutations. If the starting array is not sorted, the iteration begins "in the middle" of the lexicographic order and stops at the last permutation without going through all `n!` variants and without wrapping back to the start.

```nim
var v = @[1, 2, 3]
var permutations: seq[seq[int]]
permutations.add(v)
while v.nextPermutation():
    permutations.add(v)
assert permutations.len == 6   # 3! = 6
# @[1,2,3], @[1,3,2], @[2,1,3], @[2,3,1], @[3,1,2], @[3,2,1]

# Permutations with repeated elements — duplicates are automatically "collapsed"
var dup = @[1, 1, 2]
var count = 1
while dup.nextPermutation():
  inc count
assert count == 3   # the multiset {1,1,2} has exactly 3 distinct permutations

# If the array isn't sorted, the iteration won't cover all permutations
var mid = @[2, 1, 3]
var steps = 0
while mid.nextPermutation():
  inc steps
assert steps < 5   # less than 3! - 1 = 5
```

---

### `prevPermutation`

```nim
proc prevPermutation*[T](x: var openArray[T]): bool {.discardable.}
```

**What it does.** The mirror of `nextPermutation` — transforms `x` into the **previous lexicographic permutation**. Returns `true` on success; `false` if `x` was already the first one (sorted ascending) — the array is **left unchanged**. The algorithm is symmetric to `nextPermutation`'s: it finds the longest non-decreasing "tail", then performs a reversal and a single swap of adjacent elements.

`nextPermutation` and `prevPermutation` are mutual inverses: after `x.nextPermutation()`, calling `x.prevPermutation()` (if it returns `true`) restores `x` to its original state, and vice versa.

```nim
var v = @[3, 2, 1]   # the last permutation of {1,2,3}
assert v.prevPermutation() == true
assert v == @[3, 1, 2]
assert v.prevPermutation() == true
assert v == @[2, 3, 1]

# Mutual invertibility
var w = @[1, 2, 3, 4]
let original = w
discard w.nextPermutation()
discard w.prevPermutation()
assert w == original

# The first permutation has no predecessor
var first = @[1, 2, 3]
assert first.prevPermutation() == false
assert first == @[1, 2, 3]   # unchanged
```

---

## Rotation

Rotation cyclically shifts elements within a range. A left rotation by `dist` positions moves the element at index `i` to index `i - dist` (wrapping around the range boundary). A right rotation is just a left rotation with a negative `dist`. The value of `dist` is taken modulo the length of the range/container (`((dist mod n) + n) mod n`), so any integer is allowed — very large or negative — the modulo operation automatically maps it into the range `0 ..< n`.

All four procedures are wrappers around a shared internal implementation, `rotateInternal`/`rotatedInternal` — a port of the `std::rotate` algorithm from the C++ standard library, performing a series of swaps in O(n) without allocating a temporary array (for `rotateLeft`), or with a single copying pass (for `rotatedLeft`).

### `rotateLeft` (slice, in place)

```nim
proc rotateLeft*[T](arg: var openArray[T]; slice: HSlice[int, int];
                    dist: int): int {.discardable.}
```

**What it does.** Rotates the elements within `slice` to the left by `dist` positions, mutating the container in place. Elements outside the slice are untouched. Returns an integer — the index (in the original coordinate system) that the element originally at position `slice.a` was moved to (this is an implementation detail inherited from `std::rotate`, used internally by other algorithms built on top of rotate; in typical user code this return value is usually not needed, hence `{.discardable.}`). `dist` can be any integer: a negative value rotates right, and a value larger than the slice length is taken modulo. An invalid `slice` range raises `IndexDefect`.

```nim
var a = [0, 1, 2, 3, 4, 5]
a.rotateLeft(1 .. 4, 2)
assert a == [0, 3, 4, 1, 2, 5]
# indices 0 and 5 are unchanged; within 1..4: shifted left by 2

# Negative dist -> rotates right
var b = [0, 1, 2, 3, 4, 5]
b.rotateLeft(1 .. 4, -1)
assert b == [0, 4, 1, 2, 3, 5]

# dist that's a multiple of the range length leaves it unchanged
var c = [0, 1, 2, 3, 4, 5]
c.rotateLeft(1 .. 4, 4)   # range length = 4
assert c == [0, 1, 2, 3, 4, 5]

# Invalid range -> IndexDefect
var d = [0, 1, 2]
doAssertRaises(IndexDefect):
  discard d.rotateLeft(0 .. 5, 1)
```

---

### `rotateLeft` (whole container, in place)

```nim
proc rotateLeft*[T](arg: var openArray[T]; dist: int): int {.discardable.}
```

**What it does.** Same as above, but for the entire container — equivalent to `rotateLeft(arg, 0 .. arg.high, dist)`.

```nim
var a = [1, 2, 3, 4, 5]
a.rotateLeft(2)
assert a == [3, 4, 5, 1, 2]   # "3 4 5" moved to the front

a.rotateLeft(-2)
assert a == [1, 2, 3, 4, 5]   # rotating right by 2 restores the original

# dist larger than the array length is taken modulo
var b = [1, 2, 3, 4, 5]
b.rotateLeft(7)   # 7 mod 5 == 2
assert b == [3, 4, 5, 1, 2]

# Typical use: implementing a ring buffer / cyclic queue shift
var queue = @["task1", "task2", "task3"]
queue.rotateLeft(1)   # move the completed "task1" to the back
assert queue == @["task2", "task3", "task1"]
```

---

### `rotatedLeft` (slice, non-mutating)

```nim
proc rotatedLeft*[T](arg: openArray[T]; slice: HSlice[int, int],
                     dist: int): seq[T]
```

**What it does.** The non-mutating version of slice rotation. Returns a new `seq[T]` with the rotation applied; `arg` is left untouched. Unlike `rotateLeft`, the internal implementation (`rotatedInternal`) builds the result with a single copying pass into a new buffer, rather than a series of swaps — simpler to follow, though the asymptotic complexity is the same O(n).

```nim
let a = @[1, 2, 3, 4, 5]
let b = a.rotatedLeft(1 .. 3, 2)
assert b == @[1, 4, 2, 3, 5]
assert a == @[1, 2, 3, 4, 5]   # source unchanged

# Chaining non-mutating rotations
let c = a.rotatedLeft(1 .. 4, 1).rotatedLeft(1 .. 4, 1)
assert c == a.rotatedLeft(1 .. 4, 2)

# Rotating by an amount equal to the range length -> no change
assert a.rotatedLeft(1 .. 3, 3) == a
```

---

### `rotatedLeft` (whole container, non-mutating)

```nim
proc rotatedLeft*[T](arg: openArray[T]; dist: int): seq[T]
```

**What it does.** Non-mutating rotation of the entire container — equivalent to `rotatedLeft(arg, 0 .. arg.high, dist)`.

```nim
let a = @[1, 2, 3, 4, 5]
assert a.rotatedLeft(2)  == @[3, 4, 5, 1, 2]
assert a.rotatedLeft(-1) == @[5, 1, 2, 3, 4]
assert a == @[1, 2, 3, 4, 5]   # unchanged

# rotatedLeft(n) and rotatedLeft(-(len - n)) give the same result
assert a.rotatedLeft(2) == a.rotatedLeft(-(a.len - 2))

# A full circle returns the original array
assert a.rotatedLeft(a.len) == a
```

---

## Practical recipes

This section shows common combinations of procedures from the module that tend to come up in practice.

### Maintaining a sorted top-N leaderboard

```nim
type Score = tuple[player: string, points: int]
var leaderboard: seq[Score] = @[]

proc addScore(board: var seq[Score], s: Score, maxSize: int) =
  # kept in descending order of points
  let pos = board.lowerBound(s, proc(x, y: Score): int =
    cmp(y.points, x.points))   # inverted cmp -> descending order
  board.insert(s, pos)
  if board.len > maxSize:
    board.setLen(maxSize)

leaderboard.addScore(("Alice", 50), 3)
leaderboard.addScore(("Bob", 80), 3)
leaderboard.addScore(("Carol", 65), 3)
leaderboard.addScore(("Dave", 40), 3)   # won't make the top 3

assert leaderboard == @[("Bob", 80), ("Carol", 65), ("Alice", 50)]
```

### Removing duplicates from a sorted array using `upperBound`

```nim
proc uniqueSorted[T](a: openArray[T]): seq[T] =
  result = @[]
  var i = 0
  while i < a.len:
    result.add(a[i])
    i = a.upperBound(a[i])   # skip straight past the group of duplicates

assert uniqueSorted([1, 1, 2, 3, 3, 3, 4]) == @[1, 2, 3, 4]
```

### A round-robin scheduler based on `rotateLeft`

```nim
var workers = @["worker-A", "worker-B", "worker-C"]

proc nextTask(workers: var seq[string]): string =
  result = workers[0]
  workers.rotateLeft(1)   # shift the queue by one position

assert nextTask(workers) == "worker-A"
assert nextTask(workers) == "worker-B"
assert nextTask(workers) == "worker-C"
assert nextTask(workers) == "worker-A"   # the cycle repeats
```

### A full permutation scan with filtering

```nim
# Find all permutations of [1,2,3,4] where the sum of the first two elements is even
var v = @[1, 2, 3, 4]
var valid: seq[seq[int]]
proc check(s: seq[int]): bool = (s[0] + s[1]) mod 2 == 0

if check(v): valid.add(v)
while v.nextPermutation():
  if check(v): valid.add(v)

assert valid.len > 0
for p in valid:
  assert (p[0] + p[1]) mod 2 == 0
```

---

## Quick reference table

| Task | Mutates the argument | Returns a new seq |
|---|---|---|
| Sorting | `sort` | `sorted`, `sortedByIt` |
| Reversing | `reverse` | `reversed` |
| Left rotation | `rotateLeft` | `rotatedLeft` |
| Find an element (exact) | `binarySearch` | — |
| Find a position (≥ key) | `lowerBound` | — |
| Find a position (> key) | `upperBound` | — |
| Check sortedness | `isSorted` | — |
| Fill a range | `fill` | — |
| Merge two sorted arrays | `merge` | — |
| Cartesian product | — | `product` |
| Next permutation | `nextPermutation` | — |
| Previous permutation | `prevPermutation` | — |
| Count occurrences in a sorted array | — | `upperBound - lowerBound` |
| Flip a comparator's sign by `order` | — | `*` (operator) |

---

## Cheat sheet: which procedure to use

- **Need to sort an array and don't care about the original order** → `sort` (in place, saves memory) or `sorted` (if you also need the original array).
- **Sorting by a field/expression without writing a comparator** → `sortedByIt`.
- **Need to check whether an element exists in a sorted array, and find its index** → `binarySearch`.
- **Need to find where to insert a new element while keeping the sequence sorted** → `lowerBound` (before duplicates) or `upperBound` (after duplicates).
- **Need to combine two already-sorted lists into one sorted list** → `merge`.
- **Need to iterate over every possible ordering of elements** → `nextPermutation` in a loop, starting from a sorted array.
- **Need every combination of values drawn from several lists** → `product`.
- **Need to cyclically shift elements (e.g. implementing a ring buffer)** → `rotateLeft`/`rotatedLeft`.
- **Need a quick precondition check before binary search** → `isSorted`.
