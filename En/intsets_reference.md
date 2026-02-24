# std/intsets — Reference Manual

## Overview

`std/intsets` is a **thin specialisation wrapper** around the generic `std/packedsets` module. It exists purely for backward compatibility and naming convenience: it locks the type parameter to `int`, exports the entire `packedsets` API under the same namespace, and adds two `int`-specific constructor shorthands.

In modern Nim code you can use `std/packedsets` directly with any `Ordinal` type. The `intsets` module exists so that code written before `packedsets` was generalised continues to compile without changes.

### What `IntSet` actually is

```nim
type IntSet* = PackedSet[int]
```

`IntSet` is not a new type — it is a **type alias**. An `IntSet` value is bit-for-bit identical to a `PackedSet[int]`. Every procedure that accepts `PackedSet[int]` also accepts `IntSet`, and vice versa. The alias merely provides a shorter, self-documenting name for the common case.

### The data structure: sparse bit set

`PackedSet` (and therefore `IntSet`) is implemented as a **sparse bit set** — a Patricia trie (radix tree) of 64-bit bitmap leaves. This gives it fundamentally different performance characteristics than a hash set (`HashSet[int]`):

| Property | `IntSet` / `PackedSet[int]` | `HashSet[int]` |
|---|---|---|
| Memory for dense ranges | Very low (1 bit per element) | High (one slot per element) |
| Memory for sparse values | Moderate (trie nodes) | Proportional to count |
| `incl` / `excl` | O(log N) worst case | O(1) amortised |
| `contains` | O(log N) worst case | O(1) amortised |
| Set algebra (`union`, etc.) | O(N) bitwise on shared leaves | O(N) element-wise |
| Iteration order | Always **ascending** | Arbitrary |
| Key type constraint | **Ordinal only** | Any hashable type |

Choose `IntSet` / `PackedSet[int]` when:
- the values are drawn from a dense range (compiler variable IDs, instruction indices, small enums)
- you need ordered iteration without an explicit sort step
- set-algebra operations (`union`, `difference`, `intersection`) are frequent
- memory efficiency for large dense sets matters

Choose `HashSet[int]` when:
- values are arbitrary, sparse, or negative integers in a wide range
- O(1) average lookup is more important than memory or iteration order
- you need to store `float` or non-ordinal types (not relevant here, but worth knowing)

---

## `intsets`-specific API

These two procedures exist only in `intsets` and are not part of `packedsets`.

### `initIntSet`

```nim
proc initIntSet*(): IntSet {.inline.}
```

**Purpose:** creates and returns an empty `IntSet`. This is the constructor for the `int`-specific alias — it is exactly equivalent to `initPackedSet[int]()`, just with a shorter and more explicit name.

Since Nim v0.20, `PackedSet` variables declared as `var` are zero-initialised by default, so explicit construction is optional. However, calling `initIntSet()` is good practice when the intent should be clear to the reader, or when initialising a field in an object literal.

```nim
import std/intsets

var s = initIntSet()
s.incl(1)
s.incl(2)
s.incl(3)
assert s.len == 3
assert 2 in s
```

---

### `toIntSet`

```nim
proc toIntSet*(x: openArray[int]): IntSet {.since: (1, 3), inline.}
```

**Purpose:** creates a new `IntSet` pre-populated with every element from the array or sequence `x`. Duplicate values are silently ignored — the set contains each distinct value exactly once. Available since Nim 1.3.

This is the most convenient way to build an `IntSet` from a known collection of values in a single expression.

```nim
import std/intsets

let evens = toIntSet([2, 4, 6, 8, 8, 8])   # duplicates dropped
assert evens.len == 4
assert 6 in evens
assert 7 notin evens

# Also works with seq
let primes = toIntSet(@[2, 3, 5, 7, 11])
assert primes.len == 5
```

---

## Full exported API (via `packedsets`)

The `export packedsets` statement in `intsets` makes every public symbol from `packedsets` available when you `import std/intsets`. The full API follows, with `IntSet` substituted for `PackedSet[int]` throughout.

---

### Creating and initialising

#### `initPackedSet` / `initIntSet`

```nim
proc initPackedSet[A](): PackedSet[A]
proc initIntSet(): IntSet          # intsets-specific alias
```

Returns an empty set. Equivalent to each other for `int` element type.

```nim
var s = initIntSet()
assert s.len == 0
```

#### `toPackedSet` / `toIntSet`

```nim
proc toPackedSet[A](x: openArray[A]): PackedSet[A]
proc toIntSet(x: openArray[int]): IntSet    # intsets-specific alias
```

Bulk-constructs a set from an array or sequence. Duplicates are dropped silently.

```nim
let s = toIntSet([10, 20, 30, 20])   # {10, 20, 30}
assert s.len == 3
```

---

### Adding and removing elements

#### `incl` — include a single element

```nim
proc incl[A](s: var PackedSet[A]; key: A)
```

Adds `key` to `s`. If `key` is already present, the call is a no-op — `incl` is idempotent.

```nim
var s = initIntSet()
s.incl(5)
s.incl(5)   # no-op
assert s.len == 1
```

#### `incl` — include all elements of another set

```nim
proc incl[A](s: var PackedSet[A]; other: PackedSet[A])
```

Adds every element of `other` into `s` in place. This is the mutating equivalent of `union`. More efficient than calling `incl` in a loop for large sets because it operates on whole bitmap leaves at once.

```nim
var a = toIntSet([1, 2, 3])
let b = toIntSet([3, 4, 5])
a.incl(b)   # a is now {1, 2, 3, 4, 5}
assert a.len == 5
```

#### `excl` — remove a single element

```nim
proc excl[A](s: var PackedSet[A]; key: A)
```

Removes `key` from `s`. If `key` is not in `s`, the call is a no-op — no error is raised.

```nim
var s = toIntSet([1, 2, 3])
s.excl(2)
s.excl(99)   # no-op
assert s.len == 2
assert 2 notin s
```

#### `excl` — remove all elements of another set

```nim
proc excl[A](s: var PackedSet[A]; other: PackedSet[A])
```

Removes every element of `other` from `s` in place. The mutating equivalent of `difference`.

```nim
var a = toIntSet([1, 2, 3, 4])
let b = toIntSet([2, 4])
a.excl(b)   # a is now {1, 3}
assert a.len == 2
```

#### `containsOrIncl` — add and report prior presence

```nim
proc containsOrIncl[A](s: var PackedSet[A]; key: A): bool
```

Adds `key` to `s` and returns `true` if `key` was **already present** before the call, or `false` if `key` was newly added. Combines a membership test and an insertion into one operation — useful for deduplication loops where you want to process each value exactly once.

```nim
var seen = initIntSet()
for n in [1, 2, 1, 3, 2]:
  if not seen.containsOrIncl(n):
    echo "first time seeing: ", n
# prints: 1, 2, 3
```

#### `missingOrExcl` — remove and report prior absence

```nim
proc missingOrExcl[A](s: var PackedSet[A]; key: A): bool
```

Removes `key` from `s` and returns `true` if `key` was **already absent** before the call, or `false` if `key` was present and was removed. The logical mirror image of `containsOrIncl`.

```nim
var s = toIntSet([5, 10])
assert s.missingOrExcl(5) == false    # 5 was present, now removed
assert s.missingOrExcl(5) == true     # 5 is now absent
assert s.missingOrExcl(99) == true    # 99 was never there
```

---

### Querying membership and size

#### `contains` / `in` / `notin`

```nim
proc contains[A](s: PackedSet[A]; key: A): bool
```

Returns `true` if `key` is a member of `s`. The `in` and `notin` operators call `contains` under the hood.

```nim
let s = toIntSet([1, 2, 3])
assert s.contains(2)
assert 2 in s
assert 4 notin s
```

#### `len`

```nim
proc len[A](s: PackedSet[A]): int {.inline.}
```

Returns the number of distinct elements in the set. O(1) — the count is maintained incrementally.

```nim
let s = toIntSet([1, 1, 2, 3])
assert s.len == 3
```

#### `card` — cardinality alias

```nim
proc card[A](s: PackedSet[A]): int {.inline.}
```

Alias for `len`. The name reflects the mathematical term *cardinality* of a set (the count of its elements). Both names are equally valid; `len` is more idiomatic in Nim code.

```nim
let s = toIntSet([10, 20, 30])
assert s.card == 3
assert s.card == s.len
```

#### `isNil` — empty check

```nim
proc isNil[A](x: PackedSet[A]): bool {.inline.}
```

Returns `true` if the set has no elements. Despite the name `isNil` (inherited from the era when `IntSet` had an internal pointer that could be nil), this is a pure emptiness check — it is equivalent to `s.len == 0`.

```nim
var s = initIntSet()
assert s.isNil          # empty
s.incl(42)
assert not s.isNil      # not empty
s.excl(42)
assert s.isNil          # empty again
```

---

### Set algebra — creating new sets

All operators below return **new sets** and do not modify their arguments.

#### `union` / `+`

```nim
proc union[A](s1, s2: PackedSet[A]): PackedSet[A]
proc `+`[A](s1, s2: PackedSet[A]): PackedSet[A]   # alias
```

Returns a new set containing every element that is in `s1`, in `s2`, or in both.

```nim
let a = toIntSet([1, 2, 3])
let b = toIntSet([3, 4, 5])
assert union(a, b) == toIntSet([1, 2, 3, 4, 5])
assert (a + b).len == 5
```

#### `intersection` / `*`

```nim
proc intersection[A](s1, s2: PackedSet[A]): PackedSet[A]
proc `*`[A](s1, s2: PackedSet[A]): PackedSet[A]   # alias
```

Returns a new set containing only elements present in **both** `s1` and `s2`.

```nim
let a = toIntSet([1, 2, 3])
let b = toIntSet([2, 3, 4])
assert intersection(a, b) == toIntSet([2, 3])
assert (a * b) == toIntSet([2, 3])
```

#### `difference` / `-`

```nim
proc difference[A](s1, s2: PackedSet[A]): PackedSet[A]
proc `-`[A](s1, s2: PackedSet[A]): PackedSet[A]   # alias
```

Returns a new set containing elements that are in `s1` but **not** in `s2`. The order of operands matters.

```nim
let a = toIntSet([1, 2, 3])
let b = toIntSet([2, 3, 4])
assert difference(a, b) == toIntSet([1])
assert (a - b) == toIntSet([1])
assert (b - a) == toIntSet([4])   # different result
```

#### `symmetricDifference`

```nim
proc symmetricDifference[A](s1, s2: PackedSet[A]): PackedSet[A]
```

Returns a new set containing elements that are in either `s1` or `s2`, but **not in both** (the XOR of two sets). There is no operator shorthand.

```nim
let a = toIntSet([1, 2, 3])
let b = toIntSet([3, 4, 5])
assert symmetricDifference(a, b) == toIntSet([1, 2, 4, 5])
```

---

### Set relations

#### `disjoint`

```nim
proc disjoint[A](s1, s2: PackedSet[A]): bool
```

Returns `true` if `s1` and `s2` share no common elements. Equivalent to `intersection(s1, s2).len == 0` but without allocating a temporary set.

```nim
let a = toIntSet([1, 2])
let b = toIntSet([3, 4])
let c = toIntSet([2, 5])
assert disjoint(a, b) == true
assert disjoint(a, c) == false   # both contain 2
```

#### `<=` — subset

```nim
proc `<=`[A](s1, s2: PackedSet[A]): bool
```

Returns `true` if every element of `s1` is also in `s2`. `s1` and `s2` may be equal.

```nim
let a = toIntSet([1, 2])
let b = toIntSet([1, 2, 3])
assert a <= b    # a is a subset of b
assert b <= b    # a set is a subset of itself
assert not (b <= a)
```

#### `<` — proper subset

```nim
proc `<`[A](s1, s2: PackedSet[A]): bool
```

Returns `true` if `s1` is a subset of `s2` **and** `s1 != s2` (i.e. `s2` has strictly more elements).

```nim
let a = toIntSet([1, 2])
let b = toIntSet([1, 2, 3])
assert a < b     # proper subset
assert not (b < b)   # equal sets are not proper subsets of each other
```

#### `==` — equality

```nim
proc `==`[A](s1, s2: PackedSet[A]): bool
```

Returns `true` if both sets contain exactly the same elements. Order is irrelevant (sets have no order).

```nim
assert toIntSet([1, 2, 3]) == toIntSet([3, 1, 2])
assert toIntSet([1, 2, 2]) == toIntSet([1, 2])   # duplicates ignored
```

---

### Bulk operations

#### `clear`

```nim
proc clear[A](result: var PackedSet[A])
```

Removes all elements from the set, resetting it to the empty state. The internal trie nodes are freed and the set is ready for reuse.

```nim
var s = toIntSet([1, 2, 3])
s.clear()
assert s.len == 0
assert s.isNil
```

#### `assign` *(deprecated)*

```nim
proc assign[A](dest: var PackedSet[A]; src: PackedSet[A]) {.deprecated.}
```

Copies `src` into `dest`. **Deprecated** — use Nim's built-in `=` assignment operator instead, which performs the same deep copy automatically.

```nim
var a = initIntSet()
var b = toIntSet([1, 2, 3])
a = b   # preferred over a.assign(b)
assert a == b
```

#### `=copy`

```nim
proc `=copy`[A](dest: var PackedSet[A]; src: PackedSet[A])
```

The compiler-hooked copy hook. Called automatically by the `=` operator. You should never call this directly — it is documented here for completeness only. It performs a correct deep copy of the internal trie so that `dest` and `src` are fully independent after assignment.

---

### Iterating

#### `items`

```nim
iterator items[A](s: PackedSet[A]): A
```

Yields every element in `s` in **ascending order**. This is guaranteed — the underlying trie structure inherently produces sorted output. Modifying the set during iteration is not permitted and will trigger an assertion failure.

```nim
let s = toIntSet([30, 10, 20, 10])
for x in s:
  echo x    # prints 10, then 20, then 30 (ascending)
```

The `for x in s` syntax calls `items` implicitly.

---

### Display

#### `$`

```nim
proc `$`[A](s: PackedSet[A]): string
```

Returns a human-readable string representation of the set, e.g. `{1, 2, 3}`. Elements are always listed in ascending order. Used automatically by `echo`.

```nim
let s = toIntSet([3, 1, 2])
assert $s == "{1, 2, 3}"
```

---

## Complete usage example

```nim
import std/intsets

# Build a set of "visited" node IDs during a graph traversal
var visited = initIntSet()

proc dfs(graph: seq[seq[int]], node: int) =
  if visited.containsOrIncl(node):
    return   # already visited
  echo "visiting node ", node
  for neighbour in graph[node]:
    dfs(graph, neighbour)

let graph = @[
  @[1, 2],   # node 0 → 1, 2
  @[3],      # node 1 → 3
  @[3],      # node 2 → 3
  @[]        # node 3 (leaf)
]

dfs(graph, 0)
# visiting node 0
# visiting node 1
# visiting node 3
# visiting node 2

echo "visited: ", visited   # {0, 1, 2, 3}
echo "unvisited count: ", 4 - visited.len

# Set algebra: which nodes were NOT visited?
let allNodes = toIntSet([0, 1, 2, 3, 4, 5])
let unvisited = allNodes - visited
echo "unreachable: ", unvisited   # {4, 5}
```
