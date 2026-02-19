# Reference: `sets` Module (Nim)

> The `sets` module implements an efficient **hash set** (`HashSet`) and an **ordered hash set** (`OrderedSet`).  
> Unlike Nim's built-in `set` type, these structures can store any hashable value and automatically eliminate duplicates.

---

## Table of Contents

- [Data Types](#data-types)
- [HashSet — Initialization](#hashset--initialization)
- [HashSet — Adding Elements](#hashset--adding-elements)
- [HashSet — Removing Elements](#hashset--removing-elements)
- [HashSet — Lookup and Testing](#hashset--lookup-and-testing)
- [HashSet — Size and Comparison](#hashset--size-and-comparison)
- [HashSet — Set-Theoretic Operations](#hashset--set-theoretic-operations)
- [HashSet — Iteration and Transformation](#hashset--iteration-and-transformation)
- [OrderedSet — Initialization](#orderedset--initialization)
- [OrderedSet — Adding and Removing](#orderedset--adding-and-removing)
- [OrderedSet — Lookup and Testing](#orderedset--lookup-and-testing)
- [OrderedSet — Size and Comparison](#orderedset--size-and-comparison)
- [OrderedSet — Iteration and Transformation](#orderedset--iteration-and-transformation)
- [Deprecated Procs](#deprecated-procs)

---

## Data Types

### `HashSet[A]`

A generic hash set. **Does not** preserve insertion order. Has value semantics: the `=` operator performs a full copy.

```nim
var s: HashSet[int]
```

### `OrderedSet[A]`

A generic hash set that **preserves insertion order**.

```nim
var s: OrderedSet[string]
```

### `SomeSet[A]`

A type union `HashSet[A] | OrderedSet[A]`. Used for generic procedures that accept either kind of set.

### `defaultInitialSize`

A constant — the default initial internal buffer size (`64`).

---

## HashSet — Initialization

### `init(s: var HashSet[A], initialSize = defaultInitialSize)`

Initializes a hash set. Since Nim v0.20, sets are initialized by default, so this is mainly useful to **reset** an already-populated set.

**Parameters:**
- `s` — the set to initialize (passed by `var` reference).
- `initialSize` — initial bucket capacity (should be a power of two).

```nim
var a: HashSet[int]
a.init()        # explicit initialization
a.incl(1)
a.incl(2)
a.init()        # reset — all elements discarded
assert a.len == 0
```

---

### `initHashSet[A](initialSize = defaultInitialSize): HashSet[A]`

Creates and returns a new empty hash set. Convenient for one-line `var` declarations.

```nim
var a = initHashSet[int]()
a.incl(42)
assert a.len == 1

# With explicit initial capacity
var b = initHashSet[string](128)
```

---

### `toHashSet[A](keys: openArray[A]): HashSet[A]`

Creates a hash set from an array, sequence, or string. Duplicates are automatically removed.

```nim
let a = toHashSet([5, 3, 2, 3, 5])
assert a.len == 3   # duplicates removed
echo a              # {5, 3, 2} — order not guaranteed

let b = toHashSet("hello")
assert b.len == 4   # {'h', 'e', 'l', 'o'}
```

---

## HashSet — Adding Elements

### `incl(s: var HashSet[A], key: A)`

Adds element `key` to set `s`. If `key` is already present, does nothing.

```nim
var s = initHashSet[int]()
s.incl(10)
s.incl(10)   # safe to call again
assert s.len == 1
```

---

### `incl(s: var HashSet[A], other: HashSet[A])`

Adds all elements from `other` into `s`. This is the in-place union (equivalent to `s = s + other` but without an extra copy).

```nim
var a = toHashSet([1, 2, 3])
let b = toHashSet([3, 4, 5])
a.incl(b)
assert a.len == 5
echo a   # {1, 2, 3, 4, 5}
```

---

### `incl(s: var HashSet[A], other: OrderedSet[A])`

Adds all elements from an `OrderedSet` into a `HashSet`.

```nim
var h = toHashSet([1, 2])
let o = toOrderedSet([2, 3, 4])
h.incl(o)
assert h.len == 4
```

---

### `containsOrIncl(s: var HashSet[A], key: A): bool`

Checks whether `key` is in `s`. If **not** — adds it and returns `false`. If **yes** — returns `true` without modifying the set.

Useful for the "add if not already present" pattern.

```nim
var s = initHashSet[int]()
assert s.containsOrIncl(7) == false   # 7 was added
assert s.containsOrIncl(7) == true    # 7 already existed
assert s.containsOrIncl(8) == false   # 8 was added
assert s.len == 2
```

---

## HashSet — Removing Elements

### `excl(s: var HashSet[A], key: A)`

Removes element `key` from `s`. Does nothing if `key` is not present.

```nim
var s = toHashSet([1, 2, 3])
s.excl(2)
s.excl(99)   # safe — 99 was never in s
assert s.len == 2
```

---

### `excl(s: var HashSet[A], other: HashSet[A])`

Removes all elements of `other` from `s`. This is the in-place set difference (equivalent to `s = s - other`).

```nim
var numbers = toHashSet([1, 2, 3, 4, 5])
let even    = toHashSet([2, 4, 6])
numbers.excl(even)
echo numbers   # {1, 3, 5}
```

---

### `missingOrExcl(s: var HashSet[A], key: A): bool`

Removes `key` from `s`. Returns `true` if the element was **already absent** before the call. Returns `false` if the element **existed and was removed**.

```nim
var s = toHashSet([2, 3, 6, 7])
assert s.missingOrExcl(4) == true    # 4 was not there
assert s.missingOrExcl(6) == false   # 6 existed and was removed
assert s.missingOrExcl(6) == true    # 6 is now absent
```

---

### `pop(s: var HashSet[A]): A`

Removes and returns an **arbitrary** element from the set. Order is undefined.  
Raises `KeyError` if the set is empty.

```nim
var s = toHashSet([1, 2, 3])
let x = s.pop()   # x is one of {1, 2, 3} — order not guaranteed
assert s.len == 2

var empty: HashSet[int]
try:
  discard empty.pop()
except KeyError:
  echo "Set is empty!"
```

---

### `clear(s: var HashSet[A])`

Clears the set, removing all elements **without** releasing the allocated memory (the buffer is kept). Complexity: O(n) over the internal bucket size.

```nim
var s = toHashSet([3, 5, 7])
s.clear()
assert s.len == 0
```

---

## HashSet — Lookup and Testing

### `contains(s: HashSet[A], key: A): bool`

Returns `true` if `key` is present in the set. Enables `in` / `notin` operator syntax.

```nim
var s = toHashSet([10, 20, 30])
assert s.contains(20)
assert 10 in s
assert 99 notin s
```

---

### `s[key]` — subscript operator `[]`

Returns a **var reference** to the element stored in the set that equals `key`. Useful when `hash` and `==` are overloaded and you need the original stored object by reference.  
Raises `KeyError` if the element is not found.

```nim
type Point = object
  x, y: int

var s: HashSet[Point]
s.incl(Point(x: 1, y: 2))
let p = s[Point(x: 1, y: 2)]
```

---

## HashSet — Size and Comparison

### `len(s: HashSet[A]): int`

Returns the number of elements in the set. Safe to call on an uninitialized set (returns 0).

```nim
let s = toHashSet([1, 2, 3])
assert s.len == 3

var empty: HashSet[string]
assert empty.len == 0
```

---

### `card(s: HashSet[A]): int`

An alias for `len`. The term "cardinality" comes from set theory.

```nim
let s = toHashSet(["a", "b", "c"])
assert s.card == 3
```

---

### `==(s, t: HashSet[A]): bool`

Returns `true` if both sets contain the same elements (order does not matter for `HashSet`).

```nim
let a = toHashSet([1, 2, 3])
let b = toHashSet([3, 1, 2])
assert a == b
```

---

### `<(s, t: HashSet[A]): bool`

Returns `true` if `s` is a **strict (proper) subset** of `t`: every element of `s` is in `t`, and `t` has at least one additional element.

```nim
let a = toHashSet([1, 2])
let b = toHashSet([1, 2, 3])
assert a < b
assert not (b < a)
assert not (a < a)
```

---

### `<=(s, t: HashSet[A]): bool`

Returns `true` if `s` is a **subset** of `t` (equality counts).

```nim
let a = toHashSet([1, 2])
let b = toHashSet([1, 2, 3])
assert a <= b
assert a <= a   # a set is always a subset of itself
```

---

### `disjoint(s1, s2: HashSet[A]): bool`

Returns `true` if sets `s1` and `s2` share **no common elements**.

```nim
let a = toHashSet([1, 2])
let b = toHashSet([3, 4])
let c = toHashSet([2, 3])
assert disjoint(a, b) == true
assert disjoint(a, c) == false
```

---

## HashSet — Set-Theoretic Operations

### `union(s1, s2: HashSet[A]): HashSet[A]`  /  operator `+`

Returns the **union** (*A ∪ B*) — all elements that belong to at least one of the sets.

```nim
let a = toHashSet([1, 2, 3])
let b = toHashSet([3, 4, 5])
echo union(a, b)   # {1, 2, 3, 4, 5}
echo a + b         # same result
```

---

### `intersection(s1, s2: HashSet[A]): HashSet[A]`  /  operator `*`

Returns the **intersection** (*A ∩ B*) — elements present in both sets simultaneously.

```nim
let a = toHashSet([1, 2, 3])
let b = toHashSet([2, 3, 4])
echo intersection(a, b)   # {2, 3}
echo a * b                # same result
```

---

### `difference(s1, s2: HashSet[A]): HashSet[A]`  /  operator `-`

Returns the **set difference** (*A ∖ B*) — elements of `s1` that are not in `s2`.

```nim
let a = toHashSet([1, 2, 3, 4])
let b = toHashSet([3, 4, 5])
echo difference(a, b)   # {1, 2}
echo a - b              # same result
```

---

### `symmetricDifference(s1, s2: HashSet[A]): HashSet[A]`  /  operator `-+-`

Returns the **symmetric difference** (*A △ B*) — elements belonging to exactly one of the two sets (but not both).

```nim
let a = toHashSet([1, 2, 3])
let b = toHashSet([2, 3, 4])
echo symmetricDifference(a, b)   # {1, 4}
echo a -+- b                     # same result
```

---

## HashSet — Iteration and Transformation

### `items(s: HashSet[A]): A`  (iterator)

Iterates over all elements of the set. Traversal order is **undefined**. Modifying the set during iteration is not allowed.

```nim
let s = toHashSet([10, 20, 30])
for x in s:
  echo x

# Convert to a sequence:
import sequtils
let sq = toSeq(s.items)
```

---

### `map[A, B](data: HashSet[A], op: proc(x: A): B): HashSet[B]`

Applies `op` to each element of the set and returns a new set of the results.

```nim
let a = toHashSet([1, 2, 3])
let b = a.map(proc(x: int): string = $x)
assert b == toHashSet(["1", "2", "3"])

# With a lambda
let doubled = a.map(proc(x: int): int = x * 2)
echo doubled   # {2, 4, 6}
```

---

### `hash(s: HashSet[A]): Hash`

Computes a hash value for the entire set. Used when storing a `HashSet` as a key in another hash container.

---

### `$(s: HashSet[A]): string`

Converts the set to a string for display purposes. **Do not use for serialization** — the representation may change.

```nim
echo toHashSet([2, 4, 5])
# --> {2, 4, 5}
echo toHashSet(["hello", "world"])
# --> {hello, world}
```

---

## OrderedSet — Initialization

### `init(s: var OrderedSet[A], initialSize = defaultInitialSize)`

Initializes an ordered set. If called on a previously filled set, discards all existing elements.

```nim
var a: OrderedSet[int]
a.init()
```

---

### `initOrderedSet[A](initialSize = defaultInitialSize): OrderedSet[A]`

Creates and returns a new empty ordered set.

```nim
var a = initOrderedSet[string]()
a.incl("first")
a.incl("second")
```

---

### `toOrderedSet[A](keys: openArray[A]): OrderedSet[A]`

Creates an ordered set from an array, sequence, or string. **Insertion order is preserved**; duplicates are removed.

```nim
let a = toOrderedSet([9, 5, 1])
echo a   # {9, 5, 1} — insertion order preserved

let b = toOrderedSet("abracadabra")
echo b   # {a, b, r, c, d}
```

---

## OrderedSet — Adding and Removing

### `incl(s: var OrderedSet[A], key: A)`

Appends `key` to the end of the ordered set. If already present, does nothing.

```nim
var s = initOrderedSet[int]()
s.incl(3)
s.incl(1)
s.incl(2)
s.incl(1)   # duplicate, ignored
echo s      # {3, 1, 2}
```

---

### `containsOrIncl(s: var OrderedSet[A], key: A): bool`

Same semantics as for `HashSet`: adds `key` if absent (returns `false`), or returns `true` if already present.

```nim
var s = initOrderedSet[int]()
assert s.containsOrIncl(5) == false
assert s.containsOrIncl(5) == true
```

---

### `excl(s: var OrderedSet[A], key: A)`

Removes `key` from the ordered set. Complexity: **O(n)** due to the need to relink the ordering chain. Does nothing if the element is absent.

```nim
var s = toOrderedSet([2, 3, 6, 7])
s.excl(3)
echo s   # {2, 6, 7}
```

---

### `missingOrExcl(s: var OrderedSet[A], key: A): bool`

Removes `key`. Returns `true` if it was absent, `false` if it existed and was removed. Complexity: O(n).

```nim
var s = toOrderedSet([2, 3, 6, 7])
assert s.missingOrExcl(4) == true    # 4 was not there
assert s.missingOrExcl(6) == false   # 6 existed and was removed
```

---

### `clear(s: var OrderedSet[A])`

Clears the ordered set without releasing allocated memory. Complexity: O(n).

```nim
var s = toOrderedSet([1, 2, 3])
s.clear()
assert s.len == 0
```

---

## OrderedSet — Lookup and Testing

### `contains(s: OrderedSet[A], key: A): bool`

Returns `true` if `key` is present. Enables `in` / `notin` operator syntax.

```nim
var s = toOrderedSet([10, 20, 30])
assert 20 in s
assert 99 notin s
```

---

## OrderedSet — Size and Comparison

### `len(s: OrderedSet[A]): int`

Returns the number of elements. Safe to call on an uninitialized set.

```nim
let s = toOrderedSet([1, 2, 3])
assert s.len == 3
```

---

### `card(s: OrderedSet[A]): int`

An alias for `len`.

---

### `==(s, t: OrderedSet[A]): bool`

Compares two ordered sets considering **both membership and insertion order**.

```nim
let a = toOrderedSet([1, 2, 3])
let b = toOrderedSet([1, 2, 3])
let c = toOrderedSet([3, 2, 1])
assert a == b
assert a != c   # different order!
```

---

### `hash(s: OrderedSet[A]): Hash`

Computes a hash of the ordered set, taking element order into account.

---

### `$(s: OrderedSet[A]): string`

Converts the ordered set to a string. Element order is preserved in output.

```nim
echo toOrderedSet([9, 5, 1])
# --> {9, 5, 1}
```

---

## OrderedSet — Iteration and Transformation

### `items(s: OrderedSet[A]): A`  (iterator)

Iterates over elements **in insertion order**. Modifying the set during iteration is not allowed.

```nim
var s = initOrderedSet[int]()
for v in [9, 2, 1, 5, 8]:
  s.incl(v)

for x in s:
  echo x   # 9, 2, 1, 5, 8 — in insertion order
```

---

### `pairs(s: OrderedSet[A]): tuple[a: int, b: A]`  (iterator)

Iterates over `(index, value)` pairs in insertion order.

```nim
let s = toOrderedSet("abcde")
for idx, ch in s.pairs:
  echo idx, ": ", ch
# 0: a
# 1: b
# 2: c
# 3: d
# 4: e
```

---

## Deprecated Procs

The following procs are marked **deprecated** and should not be used in new code:

| Deprecated name | Replacement |
|---|---|
| `initSet[A]()` | `initHashSet[A]()` |
| `toSet[A](keys)` | `toHashSet[A](keys)` |
| `isValid(s: HashSet[A])` | Not needed — sets initialize automatically since v0.20 |

---

## Operator Quick Reference

| Operator | Description | Example |
|---|---|---|
| `+` | Union | `a + b` |
| `*` | Intersection | `a * b` |
| `-` | Difference | `a - b` |
| `-+-` | Symmetric difference | `a -+- b` |
| `==` | Equality | `a == b` |
| `<` | Strict subset | `a < b` |
| `<=` | Subset | `a <= b` |
| `in` | Membership test | `x in s` |
| `notin` | Non-membership test | `x notin s` |

---

## Notes

- Both types have **value semantics**: assignment `=` creates a full copy.
- `HashSet` does not guarantee traversal order; `OrderedSet` is always traversed in insertion order.
- `excl` on `OrderedSet` is **O(n)**; on `HashSet` it is amortized **O(1)**.
- Do not modify a set during iteration — doing so triggers an assertion error.
