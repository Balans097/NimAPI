# `std/critbits` Module Reference (Nim)

The module implements a **crit bit tree** — an efficient data structure for storing a **sorted set of strings** or a **sorted mapping of strings to values**.

It is a form of Patricia trie (radix tree). All keys are stored in lexicographic order, enabling efficient prefix queries.

## Two Modes of Use

```nim
CritBitTree[void]   # Set of strings
CritBitTree[T]      # Dictionary: string → T
```

**Characteristics:**
- Lookup, insertion, deletion — O(k), where k is the key length.
- Iteration is always in **lexicographic order**.
- Keys must be strings.
- Value semantics (like ordinary Nim objects).

---

## Table of Contents

1. [The `CritBitTree` Type](#the-critbittree-type)
2. [Creation — `toCritBitTree`](#creation--tocritbittree)
3. [Count — `len`](#len--count)
4. [Insertion — `incl`](#incl--insertion)
5. [Insert/Update — `[]=`](#--insertupdate)
6. [Removal — `excl`](#excl--removal)
7. [Membership — `contains` / `hasKey`](#contains--haskey--membership)
8. [Combined Operations](#combined-operations)
9. [Reading Values — `[]`](#--reading-values)
10. [Increment — `inc`](#inc--increment)
11. [Common Prefix Length — `commonPrefixLen`](#commonprefixlen--common-prefix-length)
12. [String Representation — `$`](#-string-representation)
13. [Iterators — Full Traversal](#iterators--full-traversal)
14. [Iterators — Prefix Traversal](#iterators--prefix-traversal)
15. [Summary Table](#summary-table)
16. [Comprehensive Example](#comprehensive-example)

---

## The `CritBitTree` Type

```nim
type
  CritBitTree*[T] = object
    root: Node[T]   # internal tree root (not public)
    count: int      # number of elements (not public)
```

**When to prefer over `Table`:**
- You need iteration in **lexicographic order** without extra sorting.
- You need efficient **prefix queries** (`keysWithPrefix`, etc.).
- All keys are strings.

---

## Creation — `toCritBitTree`

### Creating a dictionary from pairs

```nim
proc toCritBitTree*[T](pairs: sink openArray[(string, T)]): CritBitTree[T]
```

**Description:** Creates a `CritBitTree[T]` from an array of `(key, value)` pairs.

**Example:**
```nim
let d = {"key1": 42, "key2": 43}.toCritBitTree
assert d["key1"] == 42
assert d["key2"] == 43
```

---

### Creating a set from strings

```nim
proc toCritBitTree*(items: sink openArray[string]): CritBitTree[void]
```

**Description:** Creates a `CritBitTree[void]` (set) from an array of strings. Duplicates are silently ignored.

**Example:**
```nim
from std/sequtils import toSeq

let s = ["kitten", "puppy", "kitten"].toCritBitTree  # duplicate ignored
assert s.len == 2
assert toSeq(s.items) == @["kitten", "puppy"]  # lexicographic order
```

---

### Creating an empty structure

```nim
var s: CritBitTree[void]              # empty set
var d: CritBitTree[int]               # empty dictionary
var d2 = default(CritBitTree[string]) # explicit default
```

---

## `len` — Count

```nim
func len*[T](c: CritBitTree[T]): int
```

**Description:** Returns the number of elements. Complexity O(1).

**Example:**
```nim
let c = ["key1", "key2"].toCritBitTree
assert c.len == 2

var empty: CritBitTree[void]
assert empty.len == 0
```

---

## `incl` — Insertion

### Into a set (`CritBitTree[void]`)

```nim
proc incl*(c: var CritBitTree[void], key: string)
```

**Description:** Adds a key to the set. If the key already exists, nothing happens.

**Example:**
```nim
var c: CritBitTree[void]
c.incl("apple")
c.incl("banana")
c.incl("apple")  # already present — no-op
assert c.len == 2
assert "apple" in c
```

---

### Into a dictionary (`CritBitTree[T]`)

```nim
proc incl*[T](c: var CritBitTree[T], key: string, val: sink T)
```

**Description:** Inserts `(key, value)`. If the key already exists, the value is **not overwritten**. Use `[]=` to update an existing key.

**Example:**
```nim
var c: CritBitTree[int]
c.incl("key", 42)
assert c["key"] == 42

c.incl("key", 99)  # key exists — value NOT changed
assert c["key"] == 42
```

---

## `[]=` — Insert/Update

```nim
proc `[]=`*[T](c: var CritBitTree[T], key: string, val: sink T)
```

**Description:** Inserts or **overwrites** the value for a key. The preferred way to add and update elements in a dictionary.

**Example:**
```nim
var c: CritBitTree[int]
c["key1"] = 42
c["key2"] = 0
assert c["key1"] == 42

c["key1"] = 99  # overwrites
assert c["key1"] == 99
```

---

## `excl` — Removal

```nim
proc excl*[T](c: var CritBitTree[T], key: string)
```

**Description:** Removes a key (and its associated value) from `c`. If the key is absent, nothing happens (no-op).

**Example:**
```nim
var c: CritBitTree[void]
c.incl("key")
assert "key" in c

c.excl("key")
assert "key" notin c

c.excl("key")  # key already gone — no error
```

---

## `contains` / `hasKey` — Membership

### `contains`

```nim
func contains*[T](c: CritBitTree[T], key: string): bool
```

**Description:** Returns `true` if the key is present. Enables the `in` and `notin` operators.

**Example:**
```nim
var c: CritBitTree[void]
c.incl("key")
assert c.contains("key")
assert "key" in c
assert "other" notin c
```

---

### `hasKey`

```nim
func hasKey*[T](c: CritBitTree[T], key: string): bool
```

**Description:** Alias for `contains`. Provided for consistency with the `tables` module.

**Example:**
```nim
var c: CritBitTree[int]
c["key"] = 5
assert c.hasKey("key")
assert not c.hasKey("missing")
```

---

## Combined Operations

### `containsOrIncl` for sets

```nim
proc containsOrIncl*(c: var CritBitTree[void], key: string): bool
```

**Description:** If the key is **absent** — inserts it and returns `false`. If it **exists** — returns `true` without changes.

**Example:**
```nim
var c: CritBitTree[void]
assert not c.containsOrIncl("key")  # absent → inserted → false
assert c.containsOrIncl("key")      # present → true
assert c.len == 1
```

---

### `containsOrIncl` for dictionaries

```nim
proc containsOrIncl*[T](c: var CritBitTree[T], key: string, val: sink T): bool
```

**Description:** If the key is absent — inserts `(key, val)` and returns `false`. If present — returns `true` without changes.

**Example:**
```nim
var c: CritBitTree[int]
assert not c.containsOrIncl("key", 42)  # inserted → false
assert c["key"] == 42

assert c.containsOrIncl("key", 99)  # present → true, unchanged
assert c["key"] == 42
```

---

### `missingOrExcl`

```nim
proc missingOrExcl*[T](c: var CritBitTree[T], key: string): bool
```

**Description:** If the key is **absent** — returns `true` without changes. If it **exists** — removes it and returns `false`.

**Example:**
```nim
var c: CritBitTree[void]
assert c.missingOrExcl("key")       # absent → true

c.incl("key")
assert not c.missingOrExcl("key")   # present → removed → false
assert "key" notin c
```

---

## `[]` — Reading Values

```nim
func `[]`*[T](c: CritBitTree[T], key: string): lent T      # read-only
func `[]`*[T](c: var CritBitTree[T], key: string): var T   # mutable reference
```

**Description:** Retrieves the value for a key. The second overload returns a mutable reference, allowing in-place modification. Raises `KeyError` if the key is absent.

**Example:**
```nim
var c: CritBitTree[int]
c["a"] = 10
c["b"] = 20

assert c["a"] == 10

# In-place modification via mutable overload:
c["a"] += 5
assert c["a"] == 15

# Missing key:
try:
  discard c["z"]
except KeyError:
  echo "KeyError raised"
```

---

## `inc` — Increment

```nim
proc inc*(c: var CritBitTree[int]; key: string, val: int = 1)
```

**Description:** Increments `c[key]` by `val` (default: 1). Specialised for `CritBitTree[int]` only. If the key is absent, it is created with a value of 0 and then incremented.

**Example:**
```nim
var c: CritBitTree[int]
c["key"] = 1
inc(c, "key")
assert c["key"] == 2

inc(c, "key", 10)
assert c["key"] == 12

inc(c, "new")     # key absent: 0 + 1 = 1
assert c["new"] == 1
```

---

## `commonPrefixLen` — Common Prefix Length

```nim
func commonPrefixLen*[T](c: CritBitTree[T]): int
```

**Description:** Returns the length of the longest common prefix shared by **all** keys in `c`. Returns 0 for an empty tree. Complexity O(1) — reads an internal field of the root node.

**Example:**
```nim
var c: CritBitTree[void]
assert c.commonPrefixLen == 0

c.incl("key1")
assert c.commonPrefixLen == 4  # single key — its length

c.incl("key2")
assert c.commonPrefixLen == 3  # common prefix "key"

c.incl("other")
assert c.commonPrefixLen == 0  # no common prefix
```

---

## `$` String Representation

```nim
func `$`*[T](c: CritBitTree[T]): string
```

**Description:** Converts the tree to a string. Called automatically by `echo`. An empty set is `"{}"`, an empty dictionary is `"{:}"`.

**Example:**
```nim
let d = {"key1": 1, "key2": 2}.toCritBitTree
assert $d == """{"key1": 1, "key2": 2}"""

let s = ["key1", "key2"].toCritBitTree
assert $s == """{"key1", "key2"}"""

assert $CritBitTree[int].default == "{:}"
assert $CritBitTree[void].default == "{}"
```

---

## Iterators — Full Traversal

All iterators traverse elements in **lexicographic key order**.

### `keys` / `items`

```nim
iterator keys*[T](c: CritBitTree[T]): string
iterator items*[T](c: CritBitTree[T]): string  # alias for keys
```

**Description:** Yields all keys in lexicographic order. `items` enables `for x in c` syntax.

**Example:**
```nim
from std/sequtils import toSeq
let c = {"z": 1, "a": 2, "m": 3}.toCritBitTree
assert toSeq(c.keys) == @["a", "m", "z"]

let s = ["banana", "apple", "cherry"].toCritBitTree
assert toSeq(s) == @["apple", "banana", "cherry"]
```

---

### `values`

```nim
iterator values*[T](c: CritBitTree[T]): lent T
```

**Description:** Yields all values in the lexicographic order of their keys. Read-only.

**Example:**
```nim
from std/sequtils import toSeq
let c = {"z": 1, "a": 2, "m": 3}.toCritBitTree
assert toSeq(c.values) == @[2, 3, 1]  # key order: a, m, z
```

---

### `mvalues`

```nim
iterator mvalues*[T](c: var CritBitTree[T]): var T
```

**Description:** Yields all values as mutable references. Requires `var`.

**Example:**
```nim
var c = {"a": 1, "b": 2, "c": 3}.toCritBitTree
for v in mvalues(c):
  v *= 10
from std/sequtils import toSeq
assert toSeq(c.values) == @[10, 20, 30]
```

---

### `pairs`

```nim
iterator pairs*[T](c: CritBitTree[T]): tuple[key: string, val: T]
```

**Description:** Yields `(key, value)` pairs in lexicographic order. Read-only.

**Example:**
```nim
from std/sequtils import toSeq
let c = {"key1": 1, "key2": 2}.toCritBitTree
assert toSeq(c.pairs) == @[(key: "key1", val: 1), (key: "key2", val: 2)]
```

---

### `mpairs`

```nim
iterator mpairs*[T](c: var CritBitTree[T]): tuple[key: string, val: var T]
```

**Description:** Yields pairs with mutable values. Requires `var`.

**Example:**
```nim
var c = {"a": 10, "b": 20}.toCritBitTree
for key, val in mpairs(c):
  if key == "a":
    val = 99
assert c["a"] == 99
assert c["b"] == 20
```

---

## Iterators — Prefix Traversal

All iterators with the `WithPrefix` suffix yield only elements whose keys **start with the given string**. Order is lexicographic.

### `keysWithPrefix` / `itemsWithPrefix`

```nim
iterator keysWithPrefix*[T](c: CritBitTree[T], prefix: string): string
iterator itemsWithPrefix*[T](c: CritBitTree[T], prefix: string): string  # alias
```

**Description:** Yields all keys beginning with `prefix`.

**Example:**
```nim
from std/sequtils import toSeq
let c = {"key1": 1, "key2": 2, "other": 3}.toCritBitTree
assert toSeq(c.keysWithPrefix("key")) == @["key1", "key2"]
assert toSeq(c.keysWithPrefix("xyz")) == @[]
```

---

### `valuesWithPrefix`

```nim
iterator valuesWithPrefix*[T](c: CritBitTree[T], prefix: string): lent T
```

**Description:** Yields values whose keys begin with `prefix`. Read-only.

**Example:**
```nim
from std/sequtils import toSeq
let c = {"key1": 42, "key2": 43, "other": 0}.toCritBitTree
assert toSeq(c.valuesWithPrefix("key")) == @[42, 43]
```

---

### `mvaluesWithPrefix`

```nim
iterator mvaluesWithPrefix*[T](c: var CritBitTree[T], prefix: string): var T
```

**Description:** Yields values for prefix-matched keys as mutable references.

**Example:**
```nim
var c = {"key1": 10, "key2": 20, "other": 30}.toCritBitTree
for v in mvaluesWithPrefix(c, "key"):
  v *= 2
assert c["key1"] == 20
assert c["key2"] == 40
assert c["other"] == 30  # untouched
```

---

### `pairsWithPrefix`

```nim
iterator pairsWithPrefix*[T](c: CritBitTree[T], prefix: string): tuple[key: string, val: T]
```

**Description:** Yields `(key, value)` pairs for prefix-matched keys. Read-only.

**Example:**
```nim
from std/sequtils import toSeq
let c = {"key1": 42, "key2": 43, "other": 0}.toCritBitTree
assert toSeq(c.pairsWithPrefix("key")) == @[
  (key: "key1", val: 42),
  (key: "key2", val: 43)
]
```

---

### `mpairsWithPrefix`

```nim
iterator mpairsWithPrefix*[T](c: var CritBitTree[T], prefix: string): tuple[key: string, val: var T]
```

**Description:** Yields pairs for prefix-matched keys with mutable values.

**Example:**
```nim
var c = {"api_v1": 1, "api_v2": 2, "debug": 0}.toCritBitTree
for key, val in mpairsWithPrefix(c, "api_"):
  val += 100
assert c["api_v1"] == 101
assert c["api_v2"] == 102
assert c["debug"] == 0
```

---

## Summary Table

| Function / Iterator | Mode | Description | Complexity |
|---------------------|------|-------------|------------|
| `toCritBitTree` (pairs) | map | Create from `(string, T)` pairs | O(n·k) |
| `toCritBitTree` (strings) | set | Create from strings | O(n·k) |
| `len` | both | Number of elements | O(1) |
| `contains` / `in` | both | Membership test | O(k) |
| `hasKey` | both | Alias for `contains` | O(k) |
| `incl` (set) | set | Insert a key | O(k) |
| `incl` (map) | map | Insert key+value (no overwrite) | O(k) |
| `[]=` | map | Insert or overwrite value | O(k) |
| `[]` (read) | map | Get value or raise `KeyError` | O(k) |
| `[]` (write) | map | Mutable reference to value | O(k) |
| `excl` | both | Remove a key | O(k) |
| `containsOrIncl` (set) | set | Check; insert if absent | O(k) |
| `containsOrIncl` (map) | map | Check; insert if absent | O(k) |
| `missingOrExcl` | both | Check; remove if present | O(k) |
| `inc` | `int` map | Increment `c[key]` | O(k) |
| `commonPrefixLen` | both | Length of all-keys common prefix | O(1) |
| `$` | both | String representation | O(n) |
| `keys` / `items` | both | Keys in lexicographic order | O(n) |
| `values` | map | Values (read-only) | O(n) |
| `mvalues` | map | Values (mutable) | O(n) |
| `pairs` | map | Key-value pairs (read-only) | O(n) |
| `mpairs` | map | Key-value pairs (mutable) | O(n) |
| `keysWithPrefix` / `itemsWithPrefix` | both | Keys with given prefix | O(k+m) |
| `valuesWithPrefix` | map | Values for prefix keys (read-only) | O(k+m) |
| `mvaluesWithPrefix` | map | Values for prefix keys (mutable) | O(k+m) |
| `pairsWithPrefix` | map | Pairs for prefix keys (read-only) | O(k+m) |
| `mpairsWithPrefix` | map | Pairs for prefix keys (mutable) | O(k+m) |

*k = key length, n = number of elements, m = number of prefix matches.*

---

## Comprehensive Example

```nim
import std/critbits
from std/sequtils import toSeq

# --- Set mode (CritBitTree[void]) ---
var words: CritBitTree[void]
for w in ["banana", "apple", "cherry", "apricot", "blueberry"]:
  words.incl(w)

assert words.len == 5
assert "apple" in words
assert "mango" notin words

# Always in lexicographic order:
assert toSeq(words) == @["apple", "apricot", "banana", "blueberry", "cherry"]

# Prefix search:
assert toSeq(words.keysWithPrefix("ap")) == @["apple", "apricot"]
assert toSeq(words.keysWithPrefix("b"))  == @["banana", "blueberry"]

# commonPrefixLen:
var ap: CritBitTree[void]
ap.incl("apple")
ap.incl("apricot")
assert ap.commonPrefixLen == 2  # "ap"

# --- Dictionary mode (CritBitTree[T]) ---
var scores = {"alice": 0, "bob": 0, "anna": 0}.toCritBitTree

# Increment:
inc(scores, "alice", 10)
inc(scores, "alice", 5)
inc(scores, "bob", 7)
assert scores["alice"] == 15
assert scores["bob"] == 7

# Iteration in sorted order:
assert toSeq(scores.keys)   == @["alice", "anna", "bob"]
assert toSeq(scores.values) == @[15, 0, 7]

# Prefix search:
assert toSeq(scores.keysWithPrefix("an")) == @["anna"]
assert toSeq(scores.pairsWithPrefix("al")) == @[(key: "alice", val: 15)]

# containsOrIncl and missingOrExcl:
assert not scores.containsOrIncl("charlie", 3)  # inserted
assert scores.containsOrIncl("charlie", 99)     # present — unchanged
assert scores["charlie"] == 3

assert not scores.missingOrExcl("charlie")  # present → removed
assert "charlie" notin scores

# Bulk modification via mpairs:
for key, val in mpairs(scores):
  val *= 2
assert scores["alice"] == 30
assert scores["bob"]   == 14
```
