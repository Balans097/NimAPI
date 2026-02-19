# `std/tables` Module Reference (Nim)

The module implements three variants of hash tables (dictionaries):

| Type | Ref variant | Feature |
|------|-------------|---------|
| `Table[A, B]` | `TableRef[A, B]` | Standard hash table |
| `OrderedTable[A, B]` | `OrderedTableRef[A, B]` | Remembers insertion order |
| `CountTable[A]` | `CountTableRef[A]` | Counts occurrences of keys |

**Value semantics vs reference semantics.** Types without the `Ref` suffix have *value semantics*: `=` creates an independent copy. `Ref` types are reference types: assignment shares the same underlying data between variables.

**Custom keys.** To use objects as keys you must define a `hash(x: T): Hash` procedure from `std/hashes`.

---

## Table of Contents

- [Table — Standard Hash Table](#table--standard-hash-table)
- [TableRef — Reference Table](#tableref--reference-table)
- [OrderedTable — Insertion-Ordered Table](#orderedtable--insertion-ordered-table)
- [OrderedTableRef — Reference Ordered Table](#orderedtableref--reference-ordered-table)
- [CountTable — Counting Table](#counttable--counting-table)
- [CountTableRef — Reference Counting Table](#counttableref--reference-counting-table)
- [Common Iterators](#common-iterators)
- [Utility Functions](#utility-functions)
- [Function Summary Table](#function-summary-table)

---

## Table — Standard Hash Table

### `initTable`

```nim
proc initTable*[A, B](initialSize = defaultInitialSize): Table[A, B]
```

**Description:** Creates a new empty hash table. Starting from Nim v0.20, tables are initialized by default and this call is not necessary. `initialSize` must be a power of two.

**Example:**
```nim
let
  a = initTable[int, string]()
  b = initTable[char, seq[int]]()
```

---

### `toTable`

```nim
proc toTable*[A, B](pairs: openArray[(A, B)]): Table[A, B]
```

**Description:** Creates a new hash table from a collection of `(key, value)` pairs.

**Example:**
```nim
let a = [('a', 5), ('b', 9)]
let b = toTable(a)
assert b == {'a': 5, 'b': 9}.toTable
```

---

### `[]=` (insertion)

```nim
proc `[]=`*[A, B](t: var Table[A, B], key: A, val: sink B)
```

**Description:** Inserts a `(key, value)` pair into the table. If the key already exists, its value is overwritten.

**Example:**
```nim
var a = initTable[char, int]()
a['x'] = 7
a['y'] = 33
assert a == {'x': 7, 'y': 33}.toTable
```

---

### `[]` (lookup)

```nim
proc `[]`*[A, B](t: Table[A, B], key: A): lent B
proc `[]`*[A, B](t: var Table[A, B], key: A): var B
```

**Description:** Retrieves the value for the given key. The second overload returns a mutable reference. Raises `KeyError` if the key is not present.

**Example:**
```nim
let a = {'a': 5, 'b': 9}.toTable
assert a['a'] == 5
# a['z']  # would raise KeyError
```

---

### `hasKey`

```nim
proc hasKey*[A, B](t: Table[A, B], key: A): bool
```

**Description:** Returns `true` if the key is present in the table.

**Example:**
```nim
let a = {'a': 5, 'b': 9}.toTable
assert a.hasKey('a') == true
assert a.hasKey('z') == false
```

---

### `contains`

```nim
proc contains*[A, B](t: Table[A, B], key: A): bool
```

**Description:** Alias of `hasKey` for use with the `in` operator.

**Example:**
```nim
let a = {'a': 5, 'b': 9}.toTable
assert 'b' in a
assert 'z' notin a
```

---

### `hasKeyOrPut`

```nim
proc hasKeyOrPut*[A, B](t: var Table[A, B], key: A, val: B): bool
```

**Description:** Returns `true` if the key exists. If it does not exist, inserts `(key, val)` and returns `false`. Useful for the "insert if absent" pattern.

**Example:**
```nim
var a = {'a': 5, 'b': 9}.toTable
if a.hasKeyOrPut('a', 50):
  a['a'] = 99       # 'a' existed, update manually
if a.hasKeyOrPut('z', 50):
  a['z'] = 99       # 'z' was absent, this block won't run
assert a == {'a': 99, 'b': 9, 'z': 50}.toTable
```

---

### `getOrDefault`

```nim
proc getOrDefault*[A, B](t: Table[A, B], key: A): B
proc getOrDefault*[A, B](t: Table[A, B], key: A, def: B): B
```

**Description:** Returns the value for the key. The first overload returns the zero value of type `B` (0 for `int`, `""` for `string`, etc.) when the key is absent. The second returns the explicitly supplied `def` value.

**Example:**
```nim
let a = {'a': 5, 'b': 9}.toTable
assert a.getOrDefault('a') == 5
assert a.getOrDefault('z') == 0       # zero value for int
assert a.getOrDefault('a', 99) == 5
assert a.getOrDefault('z', 99) == 99  # custom default
```

---

### `mgetOrPut`

```nim
proc mgetOrPut*[A, B](t: var Table[A, B], key: A, val: B): var B
proc mgetOrPut*[A, B](t: var Table[A, B], key: A): var B
```

**Description:** Returns a mutable reference to the value. If the key is absent, inserts `val` (or the zero value) and returns a reference to it. Allows in-place modification through the return value.

> **Warning:** Assigning the result to a separate variable creates a copy, not a reference. For `seq` and strings, always modify the return value directly in-line.

**Example:**
```nim
var a = {'a': 5, 'b': 9}.toTable
assert a.mgetOrPut('a', 99) == 5     # 'a' exists
assert a.mgetOrPut('z', 99) == 99    # 'z' is inserted
assert a == {'a': 5, 'b': 9, 'z': 99}.toTable

# Correct way to modify a seq value in-place:
var t = initTable[int, seq[int]]()
t.mgetOrPut(25, @[25]).add(35)
assert t[25] == @[25, 35]
```

---

### `len`

```nim
proc len*[A, B](t: Table[A, B]): int
```

**Description:** Returns the number of keys in the table.

**Example:**
```nim
let a = {'a': 5, 'b': 9}.toTable
assert len(a) == 2
```

---

### `del`

```nim
proc del*[A, B](t: var Table[A, B], key: A)
```

**Description:** Removes the key from the table. Does nothing if the key is not present.

**Example:**
```nim
var a = {'a': 5, 'b': 9, 'c': 13}.toTable
a.del('a')
assert a == {'b': 9, 'c': 13}.toTable
a.del('z')  # no error
```

---

### `pop`

```nim
proc pop*[A, B](t: var Table[A, B], key: A, val: var B): bool
```

**Description:** Removes the key and writes its value into `val`. Returns `true` if the key was found; otherwise returns `false` and `val` is unchanged.

**Example:**
```nim
var a = {'a': 5, 'b': 9, 'c': 13}.toTable
var i: int
assert a.pop('b', i) == true
assert i == 9
assert a.pop('z', i) == false  # i unchanged
```

---

### `take`

```nim
proc take*[A, B](t: var Table[A, B], key: A, val: var B): bool
```

**Description:** Alias for `pop`.

---

### `clear`

```nim
proc clear*[A, B](t: var Table[A, B])
```

**Description:** Empties the table completely.

**Example:**
```nim
var a = {'a': 5, 'b': 9, 'c': 13}.toTable
clear(a)
assert len(a) == 0
```

---

### `$` (string representation)

```nim
proc `$`*[A, B](t: Table[A, B]): string
```

**Description:** Converts the table to a string. Called automatically by `echo`.

**Example:**
```nim
let a = {'a': 5, 'b': 9}.toTable
echo a  # {'a': 5, 'b': 9}
```

---

### `==`

```nim
proc `==`*[A, B](s, t: Table[A, B]): bool
```

**Description:** Returns `true` if both tables contain the same key-value pairs. Insertion order does not matter.

**Example:**
```nim
let
  a = {'a': 5, 'b': 9, 'c': 13}.toTable
  b = {'b': 9, 'c': 13, 'a': 5}.toTable
assert a == b
```

---

### `indexBy`

```nim
proc indexBy*[A, B, C](collection: A, index: proc(x: B): C): Table[C, B]
```

**Description:** Indexes a collection by the key returned from the `index` function. A convenient way to turn a list of objects into a lookup dictionary.

**Example:**
```nim
type User = object
  name: string
  age: int

let users = @[User(name: "Alice", age: 30), User(name: "Bob", age: 25)]
let byName = users.indexBy(proc(u: User): string = u.name)
assert byName["Alice"].age == 30
```

---

### `withValue`

```nim
template withValue*[A, B](t: var Table[A, B], key: A, value, body: untyped)
template withValue*[A, B](t: var Table[A, B], key: A, value, body1, body2: untyped)
template withValue*[A, B](t: Table[A, B], key: A, value, body1, body2: untyped)
```

**Description:** Safe access to a value by key through a code block. `value` is an injected variable pointing to the value.

- Single-block variant: executes `body` only if the key exists.
- Two-block variant: `body1` if found, `body2` (the `do:` block) if not.
- `var`-table variant: the value can be modified.
- `let`-table variant: the value is read-only.

Particularly useful for types with copy disabled (`{.error.}` on `=copy`).

**Example:**
```nim
type User = object
  name: string
  uid: int

var t = initTable[int, User]()
t[1] = User(name: "Hello", uid: 99)

# Mutable access (var-variant):
t.withValue(1, value):
  value.name = "Nim"
  value.uid = 1314

assert t[1].name == "Nim"

# With else branch:
t.withValue(999, value):
  echo "found"
do:
  echo "not found"  # this block executes
```

---

### `Table` Iterators

```nim
iterator pairs*[A, B](t: Table[A, B]): (A, B)
iterator mpairs*[A, B](t: var Table[A, B]): (A, var B)
iterator keys*[A, B](t: Table[A, B]): lent A
iterator values*[A, B](t: Table[A, B]): lent B
iterator mvalues*[A, B](t: var Table[A, B]): var B
```

**Description:**

- `pairs` — iterates over `(key, value)` pairs, read-only.
- `mpairs` — iterates over pairs; values are mutable (`var`).
- `keys` — iterates over keys only.
- `values` — iterates over values, read-only.
- `mvalues` — iterates over values; values are mutable.

> **Important:** Do not change the table size (add/remove keys) during iteration. Modifying values via `mpairs`/`mvalues` is safe.

**Example:**
```nim
var a = {'o': @[1, 5, 7, 9], 'e': @[2, 4, 6, 8]}.toTable

for k, v in a.pairs:
  echo k, ": ", v

for v in a.mvalues:
  v.add(99)

assert a == {'e': @[2, 4, 6, 8, 99], 'o': @[1, 5, 7, 9, 99]}.toTable
```

---

## TableRef — Reference Table

`TableRef[A, B]` is `ref Table[A, B]`. All methods mirror `Table`, but instances are created via `new*` functions and assignment shares data between variables.

### `newTable`

```nim
proc newTable*[A, B](initialSize = defaultInitialSize): TableRef[A, B]
proc newTable*[A, B](pairs: openArray[(A, B)]): TableRef[A, B]
```

**Description:** Creates a new empty reference table, or one populated from a collection of pairs.

**Example:**
```nim
var a = {1: "one", 2: "two"}.newTable
var b = a         # b and a share the same data
b[3] = "three"
assert 3 in a     # change through b is visible in a
```

---

### `newTableFrom`

```nim
proc newTableFrom*[A, B, C](collection: A, index: proc(x: B): C): TableRef[C, B]
```

**Description:** Reference-type analogue of `indexBy`. Indexes a collection by the key returned from a function.

**Example:**
```nim
let users = @[(name: "Alice", age: 30), (name: "Bob", age: 25)]
let byName = users.newTableFrom(proc(u: auto): string = u.name)
assert byName["Bob"].age == 25
```

---

All other `TableRef` methods (`[]`, `[]=`, `hasKey`, `contains`, `hasKeyOrPut`, `getOrDefault`, `mgetOrPut`, `len`, `del`, `pop`, `take`, `clear`, `$`, `==`) and iterators (`pairs`, `mpairs`, `keys`, `values`, `mvalues`) are identical in behaviour to their `Table` counterparts, accepting `TableRef` instead of `Table`.

---

## OrderedTable — Insertion-Ordered Table

`OrderedTable[A, B]` remembers the insertion order of keys. When iterating, keys are visited in the order they were added.

### `initOrderedTable`

```nim
proc initOrderedTable*[A, B](initialSize = defaultInitialSize): OrderedTable[A, B]
```

**Description:** Creates an empty ordered table.

**Example:**
```nim
var t = initOrderedTable[string, int]()
t["b"] = 2
t["a"] = 1
t["c"] = 3
for k, v in t.pairs:
  echo k  # prints: b, a, c — in insertion order
```

---

### `toOrderedTable`

```nim
proc toOrderedTable*[A, B](pairs: openArray[(A, B)]): OrderedTable[A, B]
```

**Description:** Creates an ordered table from an array of pairs. Order matches the array order.

**Example:**
```nim
let a = [('z', 1), ('y', 2), ('x', 3)]
let ot = a.toOrderedTable
assert $ot == "{'z': 1, 'y': 2, 'x': 3}"
```

---

### `sort` (OrderedTable)

```nim
proc sort*[A, B](t: var OrderedTable[A, B], cmp: proc(x, y: (A, B)): int,
    order = SortOrder.Ascending)
```

**Description:** Sorts the table using a custom comparison function. After sorting, insertion order is lost, but the table remains usable for lookups and further insertions.

**Example:**
```nim
import std/algorithm
var a = initOrderedTable[char, int]()
for i, c in "cab":
  a[c] = 10 * i
assert a == {'c': 0, 'a': 10, 'b': 20}.toOrderedTable
a.sort(system.cmp)
assert a == {'a': 10, 'b': 20, 'c': 0}.toOrderedTable
a.sort(system.cmp, order = SortOrder.Descending)
assert a == {'c': 0, 'b': 20, 'a': 10}.toOrderedTable
```

---

### `==` (OrderedTable)

```nim
proc `==`*[A, B](s, t: OrderedTable[A, B]): bool
```

**Description:** Returns `true` only if both the content **and the order** are equal. Unlike `Table`, insertion order matters here.

**Example:**
```nim
let
  a = {'a': 5, 'b': 9}.toOrderedTable
  b = {'b': 9, 'a': 5}.toOrderedTable
assert a != b  # different order!
```

---

All other `OrderedTable` methods (`[]`, `[]=`, `hasKey`, `contains`, `hasKeyOrPut`, `getOrDefault`, `mgetOrPut`, `len`, `del`, `pop`, `clear`, `$`) and iterators (`pairs`, `mpairs`, `keys`, `values`, `mvalues`) are analogous to `Table`. Iterators traverse elements **in insertion order**.

`del` for `OrderedTable` has O(n) complexity (vs O(1) for `Table`) because it must rebuild the internal ordering linked list.

---

## OrderedTableRef — Reference Ordered Table

Reference variant of `OrderedTable`. Created via:

```nim
proc newOrderedTable*[A, B](initialSize = defaultInitialSize): OrderedTableRef[A, B]
proc newOrderedTable*[A, B](pairs: openArray[(A, B)]): OrderedTableRef[A, B]
```

All methods (`[]`, `[]=`, `hasKey`, `contains`, `hasKeyOrPut`, `getOrDefault`, `mgetOrPut`, `len`, `del`, `pop`, `clear`, `sort`, `$`, `==`) and iterators are identical to `OrderedTable`.

**Example:**
```nim
let ot = {'z': 1, 'y': 2, 'x': 3}.newOrderedTable
assert $ot == "{'z': 1, 'y': 2, 'x': 3}"
```

---

## CountTable — Counting Table

`CountTable[A]` is a specialised table where the value is always an `int` — a counter of how many times a key has been seen. A count of `0` means the key is absent.

> **Important:** After calling `sort`, the table must not be modified — `[]`, `[]=`, `inc`, and `hasKey` will raise an assertion error.

### `initCountTable`

```nim
proc initCountTable*[A](initialSize = defaultInitialSize): CountTable[A]
```

**Description:** Creates an empty count table.

**Example:**
```nim
var ct = initCountTable[char]()
for c in "hello":
  ct.inc(c)
echo ct  # {'h': 1, 'e': 1, 'l': 2, 'o': 1}
```

---

### `toCountTable`

```nim
proc toCountTable*[A](keys: openArray[A]): CountTable[A]
```

**Description:** Creates a count table from a collection, counting the occurrences of each element.

**Example:**
```nim
let freq = toCountTable("abracadabra")
assert $freq == "{'a': 5, 'd': 1, 'b': 2, 'r': 2, 'c': 1}"
```

---

### `[]` (CountTable)

```nim
proc `[]`*[A](t: CountTable[A], key: A): int
```

**Description:** Returns the count for the key. Returns `0` (not an exception) if the key is absent.

**Example:**
```nim
let ct = toCountTable("hello")
assert ct['l'] == 2
assert ct['z'] == 0  # no KeyError
```

---

### `[]=` (CountTable)

```nim
proc `[]=`*[A](t: var CountTable[A], key: A, val: int)
```

**Description:** Manually sets the counter value. `val` must be ≥ 0. Setting a value of 0 removes the key.

---

### `inc` (CountTable)

```nim
proc inc*[A](t: var CountTable[A], key: A, val = 1)
```

**Description:** Increments the counter for a key by `val` (default: 1). Creates the key if absent. If the counter reaches 0, the key is removed.

**Example:**
```nim
var a = toCountTable("aab")
a.inc('a')      # 'a' → 3
a.inc('b', 10)  # 'b' → 11
assert a == toCountTable("aaabbbbbbbbbbb")
```

---

### `smallest` / `largest`

```nim
proc smallest*[A](t: CountTable[A]): tuple[key: A, val: int]
proc largest*[A](t: CountTable[A]): tuple[key: A, val: int]
```

**Description:** Returns the `(key, value)` pair with the smallest or largest counter. O(n) complexity.

**Example:**
```nim
let ct = toCountTable("abracadabra")
assert ct.largest == ('a', 5)
assert ct.smallest.val == 1
```

---

### `sort` (CountTable)

```nim
proc sort*[A](t: var CountTable[A], order = SortOrder.Descending)
```

**Description:** Sorts the table by counter value. Default order is descending (most frequent elements first).

> **Warning:** This operation is **destructive**. After calling `sort`, the table must not be modified — only iterated.

**Example:**
```nim
import std/[algorithm, sequtils]
var a = toCountTable("abracadabra")
a.sort()
assert toSeq(a.values) == @[5, 2, 2, 1, 1]
a.sort(SortOrder.Ascending)
assert toSeq(a.values) == @[1, 1, 2, 2, 5]
```

---

### `merge` (CountTable)

```nim
proc merge*[A](s: var CountTable[A], t: CountTable[A])
```

**Description:** Merges table `t` into table `s`, summing the counters of matching keys.

**Example:**
```nim
var a = toCountTable("aaabbc")
let b = toCountTable("bcc")
a.merge(b)
assert a == toCountTable("aaabbbccc")
```

---

### `hasKey`, `contains`, `getOrDefault`, `del`, `pop`, `clear`, `$`, `==`

Analogous to `Table`, but oriented towards `int` values:

- `getOrDefault(t, key, def = 0)` — returns the counter or `def`.
- `del` — removes the key (no error if absent).
- `pop` — removes and returns the value; `true` if the key existed.
- `==` — compares content without regard to insertion order.

---

## CountTableRef — Reference Counting Table

Reference variant of `CountTable`. Created via:

```nim
proc newCountTable*[A](initialSize = defaultInitialSize): CountTableRef[A]
proc newCountTable*[A](keys: openArray[A]): CountTableRef[A]
```

**Example:**
```nim
let a = newCountTable("aaabbc")
let b = newCountTable("bcc")
a.merge(b)
assert a == newCountTable("aaabbbccc")
```

All methods (`[]`, `[]=`, `inc`, `smallest`, `largest`, `hasKey`, `contains`, `getOrDefault`, `len`, `del`, `pop`, `clear`, `sort`, `merge`, `$`, `==`) and iterators are identical to `CountTable`.

---

## Common Iterators

All six table types provide the same set of iterators:

| Iterator | Table type | Description |
|----------|-----------|-------------|
| `pairs` | any (non-`var`) | Iterates `(key, value)` pairs, read-only |
| `mpairs` | `var` | Iterates `(key, value)` pairs; values are mutable |
| `keys` | any (non-`var`) | Iterates keys only |
| `values` | any (non-`var`) | Iterates values, read-only |
| `mvalues` | `var` | Iterates values; values are mutable |

For `OrderedTable` and `OrderedTableRef`, traversal follows **insertion order** (or sorted order after `sort`).

**Example (CountTable):**
```nim
var a = toCountTable("abracadabra")
for k, v in mpairs(a):
  v = 2  # set all counters to 2
assert a == toCountTable("aabbccddrr")
```

**Example (OrderedTable):**
```nim
var a = {'o': @[1, 5], 'e': @[2, 4]}.toOrderedTable
for k, v in a.mpairs:
  v.add(v[0] + 10)
assert a == {'o': @[1, 5, 11], 'e': @[2, 4, 12]}.toOrderedTable
```

---

## Utility Functions

### `hash` (for tables)

```nim
proc hash*[K, V](s: Table[K, V]): Hash
proc hash*[K, V](s: OrderedTable[K, V]): Hash
proc hash*[V](s: CountTable[V]): Hash
```

**Description:** Computes a hash for the entire table. Allows tables to be used as keys in other tables.

- `Table` and `CountTable`: hash is order-independent.
- `OrderedTable`: hash depends on element order.

---

## Function Summary Table

| Function | Table | TableRef | OrderedTable | OrderedTableRef | CountTable | CountTableRef |
|----------|:-----:|:--------:|:------------:|:---------------:|:----------:|:-------------:|
| `init*/new*` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `to*Table` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `[]` / `[]=` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `hasKey` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `contains` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `hasKeyOrPut` | ✓ | ✓ | ✓ | ✓ | — | — |
| `getOrDefault` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mgetOrPut` | ✓ | ✓ | ✓ | ✓ | — | — |
| `len` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `del` | ✓ | ✓ | ✓ O(n) | ✓ O(n) | ✓ | ✓ |
| `pop` / `take` | ✓ | ✓ | ✓ O(n) | ✓ O(n) | ✓ | ✓ |
| `clear` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `inc` | — | — | — | — | ✓ | ✓ |
| `smallest` / `largest` | — | — | — | — | ✓ | ✓ |
| `sort` | — | — | ✓ | ✓ | ✓ destructive | ✓ destructive |
| `merge` | — | — | — | — | ✓ | ✓ |
| `indexBy` / `newTableFrom` | ✓ | ✓ | — | — | — | — |
| `withValue` | ✓ | — | — | — | — | — |
| `$` / `==` / `hash` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `pairs` / `mpairs` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `keys` / `values` / `mvalues` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

---

## Comprehensive Example

```nim
import std/tables

# --- Table: word frequency with manual counting ---
var wordCount = initTable[string, int]()
for word in ["the", "quick", "brown", "fox", "the", "fox"]:
  wordCount[word] = wordCount.getOrDefault(word, 0) + 1
assert wordCount["the"] == 2

# --- CountTable: the same, but simpler ---
let ct = toCountTable(["the", "quick", "brown", "fox", "the", "fox"])
assert ct["the"] == 2
assert ct["quick"] == 1

# --- OrderedTable: preserving insertion order ---
var ot = initOrderedTable[string, int]()
for w in ["first", "second", "third"]:
  ot[w] = ot.len
var keys: seq[string]
for k in ot.keys:
  keys.add(k)
assert keys == @["first", "second", "third"]  # order preserved

# --- Value semantics vs reference semantics ---
var a = {"x": 1}.toTable
var b = a        # independent copy
b["y"] = 2
assert "y" notin a

var ra = {"x": 1}.newTable
var rb = ra      # shared data
rb["y"] = 2
assert "y" in ra  # change is visible through ra
```
