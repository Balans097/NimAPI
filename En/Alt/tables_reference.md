# std/tables — Reference Manual

## Overview

The `tables` module provides three families of generic hash-table containers. Each family comes in two flavours — a **value type** and a **reference type** — giving six concrete types in total.

| Family | Value type | Reference type | Key feature |
|---|---|---|---|
| General hash table | `Table[A, B]` | `TableRef[A, B]` | Unordered, fast O(1) average |
| Order-preserving table | `OrderedTable[A, B]` | `OrderedTableRef[A, B]` | Remembers insertion order |
| Counting table | `CountTable[A]` | `CountTableRef[A]` | Maps items to occurrence counts |

### Value semantics vs. reference semantics

This is the single most important concept in the module. **Value types** (`Table`, `OrderedTable`, `CountTable`) behave like `int` or `string`: assigning one table to another variable creates a fully independent copy. Mutating the copy does not affect the original.

**Reference types** (`TableRef`, `OrderedTableRef`, `CountTableRef`) behave like `ref object`: assignment copies only the pointer, so both variables point to the same underlying data. Mutating through one variable is immediately visible through the other.

```nim
# Value semantics — independent copy
var a = {"x": 1, "y": 2}.toTable
var b = a          # deep copy
b["z"] = 3
assert "z" notin a # a is unaffected

# Reference semantics — shared data
var c = {"x": 1, "y": 2}.newTable
var d = c          # same object
d["z"] = 3
assert "z" in c    # c is also changed
```

Choose value types by default. Use reference types when you need to share a single table across multiple variables or pass it cheaply without copying.

### Custom key types

Any type used as a key must have:
- a `hash` proc returning `Hash` (from `std/hashes`)
- an `==` proc consistent with `hash` (the standard `==` for objects already does a field-by-field comparison)

```nim
import std/hashes
type Point = object
  x, y: int

proc hash(p: Point): Hash =
  result = p.x.hash !& p.y.hash
  result = !$result

var t = initTable[Point, string]()
t[Point(x: 1, y: 2)] = "origin-ish"
```

### Do not modify a table while iterating over it

All iterators check that the table length has not changed. Inserting or deleting during iteration will trigger an assertion failure. Collect keys first if you need to modify while traversing.

---

## Table / TableRef — General hash table

### Creating

#### `initTable`

```nim
proc initTable*[A, B](initialSize = defaultInitialSize): Table[A, B]
```

Creates an empty `Table`. Since Nim v0.20, a `Table` declared as `var` is already zero-initialised, so calling `initTable` is optional. It is still useful when you want to pre-size the internal array and avoid early rehashing.

```nim
var t = initTable[string, int]()       # default capacity
var t2 = initTable[string, int](128)   # hint: ~128 items before rehash
```

#### `toTable`

```nim
proc toTable*[A, B](pairs: openArray[(A, B)]): Table[A, B]
```

Converts a sequence or array of `(key, value)` tuples into a new `Table` in one step. The most convenient way to create a pre-populated table from a literal.

```nim
let colours = {"red": 0xFF0000, "green": 0x00FF00, "blue": 0x0000FF}.toTable
assert colours["red"] == 0xFF0000
```

#### `newTable` (TableRef)

```nim
proc newTable*[A, B](initialSize = defaultInitialSize): TableRef[A, B]
proc newTable*[A, B](pairs: openArray[(A, B)]): TableRef[A, B]
```

The reference-type counterparts of `initTable` and `toTable`. Return a heap-allocated table behind a `ref`. Two variables assigned from the same `newTable` share the same data.

```nim
let t = newTable([("a", 1), ("b", 2)])
let u = t            # same object, not a copy
u["c"] = 3
assert t["c"] == 3  # visible through t as well
```

#### `newTableFrom`

```nim
proc newTableFrom*[A, B, C](collection: A, index: proc(x: B): C): TableRef[C, B]
```

Indexes a collection into a `TableRef` using a key-extraction function. Each element of `collection` becomes a value; the function `index` computes the key. This is useful when you have a list of objects and want to look them up by some field.

```nim
type User = object
  id: int
  name: string

let users = @[User(id: 1, name: "Alice"), User(id: 2, name: "Bob")]
let byId = newTableFrom(users, proc(u: User): int = u.id)
assert byId[1].name == "Alice"
```

---

### Reading values

#### `[]` — subscript operator

```nim
proc `[]`*[A, B](t: Table[A, B], key: A): lent B          # immutable
proc `[]`*[A, B](t: var Table[A, B], key: A): var B        # mutable
```

Returns the value stored at `key`. Raises `KeyError` if the key is absent. Use this when you are certain the key exists, or when you want the error to propagate.

```nim
let t = {"a": 10}.toTable
echo t["a"]          # 10
# echo t["z"]        # raises KeyError: key not found: z
```

The `var` overload returns a mutable reference, letting you modify the stored value directly: `t["a"] += 5`.

#### `getOrDefault` — safe read with fallback

```nim
proc getOrDefault*[A, B](t: Table[A, B], key: A): B
proc getOrDefault*[A, B](t: Table[A, B], key: A, def: B): B
```

Returns the value at `key`, or a fallback if the key is missing. The one-argument form returns the zero value for type `B` (0 for integers, `""` for strings, etc.). The two-argument form lets you specify any fallback.

```nim
let t = {"a": 5}.toTable
assert t.getOrDefault("a") == 5
assert t.getOrDefault("z") == 0        # zero value for int
assert t.getOrDefault("z", 99) == 99   # custom fallback
```

`getOrDefault` never inserts anything into the table — it is a pure read.

---

### Checking for key existence

#### `hasKey` / `contains` / `in`

```nim
proc hasKey*[A, B](t: Table[A, B], key: A): bool
proc contains*[A, B](t: Table[A, B], key: A): bool
```

`hasKey` checks whether `key` exists. `contains` is an alias, enabling the idiomatic `key in t` syntax (because Nim's `in` operator calls `contains`).

```nim
let t = {"a": 1, "b": 2}.toTable
assert t.hasKey("a")
assert "b" in t
assert "z" notin t
```

---

### Inserting and updating

#### `[]=` — insert or replace

```nim
proc `[]=`*[A, B](t: var Table[A, B], key: A, val: sink B)
```

The fundamental write operation. If `key` does not exist, a new entry is created. If it does, the existing value is replaced. There is never more than one entry per key after using `[]=`.

```nim
var t = initTable[string, int]()
t["x"] = 1   # insert
t["x"] = 2   # replace — t["x"] is now 2
```

#### `hasKeyOrPut` — insert if absent, report presence

```nim
proc hasKeyOrPut*[A, B](t: var Table[A, B], key: A, val: B): bool
```

Returns `true` if the key was already present (table unchanged). Returns `false` and inserts `(key, val)` if it was absent. This combines an existence check and a conditional insert into one atomic call — no risk of accidentally overwriting a value that was just set.

```nim
var t = {"a": 1}.toTable
assert t.hasKeyOrPut("a", 99) == true   # already there, 99 ignored
assert t.hasKeyOrPut("b", 99) == false  # inserted with value 99
assert t["b"] == 99
```

#### `mgetOrPut` — get-or-insert with mutable result

```nim
proc mgetOrPut*[A, B](t: var Table[A, B], key: A, val: B): var B
proc mgetOrPut*[A, B](t: var Table[A, B], key: A): var B
```

Returns a mutable reference to the value at `key`. If the key is absent, `val` (or the zero value in the no-argument form) is inserted first. The main use case is initialising-on-first-access: you can call `t.mgetOrPut(key, @[]).add(item)` to accumulate items into a per-key list.

**Important pitfall:** assigning the result to a `var` creates a copy for value types like `seq` or `string`. Always chain the call directly if you want to modify the stored value.

```nim
var t = initTable[string, seq[int]]()

# Correct: chain directly — modifies the stored seq
t.mgetOrPut("nums", @[]).add(42)
assert t["nums"] == @[42]

# Incorrect: creates a copy — original is unchanged
var copy = t.mgetOrPut("nums2", @[])
copy.add(99)
assert t["nums2"] == @[]  # still empty!
```

---

### Deleting entries

#### `del` — silent delete

```nim
proc del*[A, B](t: var Table[A, B], key: A)
```

Removes the entry with the given key. If the key does not exist, the call is a no-op — no error is raised. Use this when absence is an acceptable condition.

```nim
var t = {"a": 1, "b": 2}.toTable
t.del("a")
t.del("z")   # no-op
assert "a" notin t
```

#### `pop` — delete and retrieve

```nim
proc pop*[A, B](t: var Table[A, B], key: A, val: var B): bool
```

Removes the entry and, if found, stores the value in `val` and returns `true`. Returns `false` and leaves `val` unchanged if the key was absent. Use `pop` when you need to consume a value atomically (remove it from the table and use it).

```nim
var t = {"a": 10}.toTable
var v: int
assert t.pop("a", v) == true and v == 10
assert t.pop("z", v) == false         # v still 10
```

#### `take` — alias for `pop`

```nim
proc take*[A, B](t: var Table[A, B], key: A, val: var B): bool
```

Identical to `pop`. Provided as an alternative name that reads more naturally in some contexts.

#### `clear` — empty the table

```nim
proc clear*[A, B](t: var Table[A, B])
```

Removes all entries, resetting the counter to zero. The internal storage array is kept but all slots are marked empty, so the capacity is retained and the table can be reused without reallocation.

```nim
var t = {"a": 1, "b": 2}.toTable
t.clear()
assert t.len == 0
```

---

### Metadata

#### `len`

```nim
proc len*[A, B](t: Table[A, B]): int
```

Returns the number of key-value pairs currently in the table. O(1).

---

### In-place access with `withValue`

#### `withValue` — two-arm form (found only)

```nim
template withValue*[A, B](t: var Table[A, B], key: A, value, body: untyped)
template withValue*[A, B](t: Table[A, B], key: A, value, body: untyped)
```

Looks up `key`. If found, injects a reference to the stored value as `value` and executes `body`. If the key is absent, `body` is skipped silently. This is the best way to read and optionally update a value without copying it out of the table, especially for types that cannot be copied.

```nim
var t = {"a": @[1, 2, 3]}.toTable
t.withValue("a", v):
  v[].add(4)       # modifies the seq stored in the table directly
assert t["a"] == @[1, 2, 3, 4]
```

#### `withValue` — three-arm form (found / not found)

```nim
template withValue*[A, B](t: var Table[A, B], key: A, value, body1, body2: untyped)
template withValue*[A, B](t: Table[A, B], key: A, value, body1, body2: untyped)
```

Same as the two-arm form but accepts a `do:` block as `body2` that runs when the key is absent.

```nim
var t = initTable[string, int]()
t.withValue("x", v):
  echo "found: ", v[]
do:
  t["x"] = 0          # not found: initialise
```

---

### Iterators

All iterators for `Table` and `TableRef` visit entries in **unspecified (hash) order**. Do not rely on a particular order.

#### `pairs` / `mpairs`

```nim
iterator pairs*[A, B](t: Table[A, B]): (A, B)
iterator mpairs*[A, B](t: var Table[A, B]): (A, var B)
```

`pairs` yields each `(key, value)` tuple. `mpairs` yields mutable value references, letting you modify values in-place during iteration.

```nim
var t = {"a": 1, "b": 2}.toTable
for k, v in t.mpairs:
  v *= 10
# t is now {"a": 10, "b": 20}
```

#### `keys`

```nim
iterator keys*[A, B](t: Table[A, B]): lent A
```

Yields only the keys.

#### `values` / `mvalues`

```nim
iterator values*[A, B](t: Table[A, B]): lent B
iterator mvalues*[A, B](t: var Table[A, B]): var B
```

`values` yields immutable references to stored values. `mvalues` yields mutable ones.

---

### Operators

#### `==`

```nim
proc `==`*[A, B](s, t: Table[A, B]): bool
```

Two tables are equal if they contain exactly the same key-value pairs, regardless of insertion order. For `TableRef`, both `nil` values are considered equal.

#### `$`

```nim
proc `$`*[A, B](t: Table[A, B]): string
```

Converts the table to a human-readable string, e.g. `{'a': 1, 'b': 2}`. Used automatically by `echo`.

#### `hash`

```nim
proc hash*[K, V](s: Table[K, V]): Hash
```

Computes a hash of the whole table (XOR of hashes of all pairs). Order-independent, so two equal tables always produce the same hash. Enables using a `Table` as a key in another table.

---

---

## OrderedTable / OrderedTableRef — Insertion-ordered table

`OrderedTable` behaves identically to `Table` for all lookup and mutation operations. The only difference is that iteration always happens in the order keys were first inserted. Internally, it stores a linked list of insertion indices alongside the hash table.

### Key differences from `Table`

**`==` compares order.** Two `OrderedTable`s are equal only if they contain the same pairs _in the same insertion order_.

```nim
let a = {'x': 1, 'y': 2}.toOrderedTable
let b = {'y': 2, 'x': 1}.toOrderedTable
assert a != b   # same pairs, different insertion order
```

**`sort` reorders the insertion order.** After calling `sort`, all iterators visit entries in sorted order. Importantly, `sort` marks the table as reordered — it does not destroy it. Repeated sorts are allowed.

**`del` is O(n).** Deleting from an `OrderedTable` requires relinking the insertion chain, which takes linear time in the worst case. For frequent deletions, `Table` is faster.

### Creating

#### `initOrderedTable` / `toOrderedTable`

```nim
proc initOrderedTable*[A, B](initialSize = defaultInitialSize): OrderedTable[A, B]
proc toOrderedTable*[A, B](pairs: openArray[(A, B)]): OrderedTable[A, B]
```

Same semantics as their `Table` counterparts. The pairs are inserted in array order, which becomes the iteration order.

```nim
let ot = [('z', 1), ('y', 2), ('x', 3)].toOrderedTable
assert $ot == "{'z': 1, 'y': 2, 'x': 3}"  # z first, as inserted
```

#### `newOrderedTable` (OrderedTableRef)

```nim
proc newOrderedTable*[A, B](initialSize = defaultInitialSize): OrderedTableRef[A, B]
proc newOrderedTable*[A, B](pairs: openArray[(A, B)]): OrderedTableRef[A, B]
```

Reference-type version. Shares data between assignments.

---

### Sorting

#### `sort` — reorder the insertion chain

```nim
proc sort*[A, B](t: var OrderedTable[A, B],
                 cmp: proc(x, y: (A, B)): int,
                 order = SortOrder.Ascending)
```

Rearranges entries so that subsequent iteration visits them in the order defined by `cmp`. The comparison function receives full `(key, value)` tuples, giving you full control: sort by value, by key, by a combination, etc.

**Warning:** after sorting, the table must not be modified. The sort rewires the internal linked list but does not prevent further `[]=` or `del` calls — such modifications will silently corrupt the iteration order.

```nim
var t = {"banana": 3, "apple": 1, "cherry": 2}.toOrderedTable
t.sort(proc(a, b: (string, int)): int = cmp(a[1], b[1]))
# Now iterates in ascending value order: apple(1), cherry(2), banana(3)
for k, v in t:
  echo k, ": ", v
```

---

### All other operations

`[]`, `[]=`, `hasKey`, `contains`, `getOrDefault`, `mgetOrPut`, `hasKeyOrPut`, `del`, `pop`, `clear`, `len`, `withValue`, `pairs`, `mpairs`, `keys`, `values`, `mvalues`, `$`, `==` — all work identically to their `Table` equivalents, with the addition that `pairs`, `mpairs`, `keys`, `values`, and `mvalues` visit entries **in insertion order**.

---

---

## CountTable / CountTableRef — Occurrence-counting table

`CountTable[A]` is a specialised hash table that maps items of type `A` to `int` counts. The value type is always `int` — you do not specify it. It is optimised for the common pattern of "count how many times each item appears". Unlike `Table`, it uses zero to mean "absent", so `t["missing"]` returns 0 without raising an error.

### Creating

#### `initCountTable` / `toCountTable`

```nim
proc initCountTable*[A](initialSize = defaultInitialSize): CountTable[A]
proc toCountTable*[A](keys: openArray[A]): CountTable[A]
```

`initCountTable` creates an empty counter. `toCountTable` counts occurrences of every element in the given array or sequence.

```nim
let freq = toCountTable("abracadabra")
# freq: {'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1}
assert freq['a'] == 5
assert freq['z'] == 0   # absent key returns 0, no exception
```

#### `newCountTable` (CountTableRef)

```nim
proc newCountTable*[A](initialSize = defaultInitialSize): CountTableRef[A]
proc newCountTable*[A](keys: openArray[A]): CountTableRef[A]
```

Reference-type version. `newCountTable(keys)` counts occurrences in `keys`, just like `toCountTable`.

---

### Reading

#### `[]`

```nim
proc `[]`*[A](t: CountTable[A], key: A): int
```

Returns the count for `key`, or **0** if the key has never been inserted. Unlike `Table`, this does not raise `KeyError` — zero is the natural "not seen yet" value for a counter.

#### `getOrDefault`

```nim
proc getOrDefault*[A](t: CountTable[A], key: A; def: int = 0): int
```

Like `[]`, but lets you specify a custom default (instead of 0).

---

### Inserting and counting

#### `[]=` — set an exact count

```nim
proc `[]=`*[A](t: var CountTable[A], key: A, val: int)
```

Sets the count for `key` to exactly `val`. The value must be a positive integer (`assert val >= 0`). Setting to 0 removes the key (since 0 means absent).

#### `inc` — increment a count

```nim
proc inc*[A](t: var CountTable[A], key: A, val = 1)
```

Increments the count of `key` by `val` (default 1). If the key does not exist it is created with count `val`. If incrementing brings the count to 0 (via negative `val`), the entry is automatically removed. This is the primary way to build up a count table element by element.

```nim
var freq = initCountTable[char]()
for c in "hello world":
  freq.inc(c)
assert freq['l'] == 3
```

---

### Querying extremes

#### `smallest` / `largest`

```nim
proc smallest*[A](t: CountTable[A]): tuple[key: A, val: int]  # O(n)
proc largest*[A](t: CountTable[A]): tuple[key: A, val: int]   # O(n)
```

Return the entry with the minimum or maximum count respectively. Both scan the entire table in O(n). If you need this repeatedly, call `sort` once and iterate from the beginning or end.

```nim
let freq = toCountTable("abracadabra")
assert freq.largest == ('a', 5)
assert freq.smallest.val == 1   # one of: 'c', 'd'
```

---

### Sorting

#### `sort` — sort by count

```nim
proc sort*[A](t: var CountTable[A], order = SortOrder.Descending)
```

Rearranges entries by their count values. The default order is descending (most frequent first), which is the natural "top-N" usage. After sorting, calling `[]`, `inc`, `del`, or any mutation raises an assertion error — the internal flag `isSorted` is set. Sorting is irreversible short of clearing the table.

```nim
var freq = toCountTable("abracadabra")
freq.sort()
for k, v in freq:
  echo k, ": ", v
# a: 5
# b: 2
# r: 2
# c: 1
# d: 1
```

---

### Merging

#### `merge` — add counts from another table

```nim
proc merge*[A](s: var CountTable[A], t: CountTable[A])
```

Adds all counts from `t` into `s`. For each key in `t`, its count is added to the existing count in `s` (or inserted if absent). This is equivalent to inserting every item of `t`'s original source collection into `s` again.

```nim
var a = toCountTable("aab")   # a:2, b:1
let b = toCountTable("bc")    # b:1, c:1
a.merge(b)
assert a == toCountTable("aabbc")  # a:2, b:2, c:1
```

---

### All other operations

`del`, `pop`, `clear`, `hasKey`, `contains`, `len`, `$`, `==`, `pairs`, `mpairs`, `keys`, `values`, `mvalues` — work the same as in `Table`, with counts as values. The iterators skip entries with count 0 (which represent absent keys).

---

---

## Quick-reference: choosing the right type

| Need | Use |
|---|---|
| Simple key→value mapping, no order needed | `Table` |
| Need to share/alias the table | `TableRef` |
| Insertion order matters (config, menus, logs) | `OrderedTable` |
| Need iteration in a user-defined sort order | `OrderedTable` + `sort` |
| Counting occurrences of items | `CountTable` |
| Finding top-N most frequent items | `CountTable` + `sort` + iterate |

## Quick-reference: getting a value safely

| Situation | Recommended call |
|---|---|
| Key definitely exists | `t[key]` |
| Key may be absent, zero default is fine | `t.getOrDefault(key)` |
| Key may be absent, custom default | `t.getOrDefault(key, def)` |
| Need to mutate in-place (no copy) | `t.withValue(key, v): v[]...` |
| Insert default if absent, then modify | `t.mgetOrPut(key, default).modify(...)` |
