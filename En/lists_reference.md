# `std/lists` Module Reference (Nim)

The module implements four kinds of linked lists:

| Type | Direction | Circular |
|------|-----------|----------|
| `SinglyLinkedList[T]` | Singly linked | No |
| `DoublyLinkedList[T]` | Doubly linked | No |
| `SinglyLinkedRing[T]` | Singly linked | Yes |
| `DoublyLinkedRing[T]` | Doubly linked | Yes |

**Key characteristics:**
- The `next`, `prev`, and `value` fields of nodes are public and can be manipulated directly.
- All four structures have **value semantics** (like ordinary Nim objects).
- Rings (circular lists) have the last node pointing back to the first.
- Nodes (`SinglyLinkedNode[T]`, `DoublyLinkedNode[T]`) are ref types and can be shared between lists.

---

## Table of Contents

1. [Data Types](#data-types)
2. [Creating Collections](#creating-collections)
3. [Creating Nodes](#creating-nodes)
4. [Appending — `add`](#add--appending-to-the-end)
5. [Prepending — `prepend`](#prepend--adding-to-the-beginning)
6. [Concatenation — `add` (list + list)](#add-list-concatenation)
7. [Move-Concatenation — `addMoved`](#addmoved--move-concatenation)
8. [Prepend list — `prepend` (list + list)](#prepend-list--list)
9. [Move-Prepend — `prependMoved`](#prependmoved--move-prepend)
10. [Copying — `copy`](#copy--shallow-copy)
11. [Removal — `remove`](#remove--node-removal)
12. [Search — `find` and `contains`](#find-and-contains--search)
13. [String Representation — `$`](#-string-representation)
14. [Iterators](#iterators)
15. [Converting from Arrays — `to*`](#converting-from-arrays)
16. [Aliases — `append` and `appendMoved`](#aliases-append-and-appendmoved)
17. [Summary Table](#summary-table)

---

## Data Types

### Nodes

```nim
type
  SinglyLinkedNodeObj*[T] = object
    next*: SinglyLinkedNode[T]
    value*: T
  SinglyLinkedNode*[T] = ref SinglyLinkedNodeObj[T]

  DoublyLinkedNodeObj*[T] = object
    next*: DoublyLinkedNode[T]
    prev* {.cursor.}: DoublyLinkedNode[T]
    value*: T
  DoublyLinkedNode*[T] = ref DoublyLinkedNodeObj[T]
```

Nodes are ref types. Fields `next`, `prev`, and `value` are publicly accessible. `prev` is marked `{.cursor.}` to avoid cyclic ownership issues with the GC.

---

### Collections

```nim
type
  SinglyLinkedList*[T] = object
    head*: SinglyLinkedNode[T]
    tail* {.cursor.}: SinglyLinkedNode[T]

  DoublyLinkedList*[T] = object
    head*: DoublyLinkedNode[T]
    tail* {.cursor.}: DoublyLinkedNode[T]

  SinglyLinkedRing*[T] = object
    head*: SinglyLinkedNode[T]
    tail* {.cursor.}: SinglyLinkedNode[T]

  DoublyLinkedRing*[T] = object
    head*: DoublyLinkedNode[T]
    # DoublyLinkedRing has no separate tail field —
    # the last element is accessible via head.prev
```

Fields `head` and `tail` are public. `DoublyLinkedRing` has no separate `tail` field: the predecessor of `head` is the last element (`head.prev`).

---

### Type Aliases

```nim
SomeLinkedList*[T]       = SinglyLinkedList[T] | DoublyLinkedList[T]
SomeLinkedRing*[T]       = SinglyLinkedRing[T] | DoublyLinkedRing[T]
SomeLinkedCollection*[T] = SomeLinkedList[T] | SomeLinkedRing[T]
SomeLinkedNode*[T]       = SinglyLinkedNode[T] | DoublyLinkedNode[T]
```

Used in generic procedure signatures to accept any collection or node type.

---

## Creating Collections

### `initSinglyLinkedList`

```nim
proc initSinglyLinkedList*[T](): SinglyLinkedList[T]
```

**Description:** Creates a new empty singly linked list. Since Nim v0.20, lists are zero-initialized by default and this call is not strictly necessary.

**Example:**
```nim
var a = initSinglyLinkedList[int]()
# Equivalent to:
var b: SinglyLinkedList[int]
```

---

### `initDoublyLinkedList`

```nim
proc initDoublyLinkedList*[T](): DoublyLinkedList[T]
```

**Description:** Creates a new empty doubly linked list.

**Example:**
```nim
var list = initDoublyLinkedList[int]()
list.add(1)
list.add(2)
assert $list == "[1, 2]"
```

---

### `initSinglyLinkedRing`

```nim
proc initSinglyLinkedRing*[T](): SinglyLinkedRing[T]
```

**Description:** Creates a new empty singly linked ring.

**Example:**
```nim
var ring = initSinglyLinkedRing[int]()
ring.add(1)
ring.add(2)
ring.add(3)
# Last node links back to the first:
# 1 → 2 → 3 → 1 → ...
```

---

### `initDoublyLinkedRing`

```nim
proc initDoublyLinkedRing*[T](): DoublyLinkedRing[T]
```

**Description:** Creates a new empty doubly linked ring.

**Example:**
```nim
var ring = initDoublyLinkedRing[int]()
ring.add(5)
ring.add(10)
# Each node knows its previous and next, forming a closed chain.
```

---

## Creating Nodes

### `newSinglyLinkedNode`

```nim
proc newSinglyLinkedNode*[T](value: T): SinglyLinkedNode[T]
```

**Description:** Creates a new node for a singly linked list or ring with the given value. Allows you to hold a reference to a node before inserting it.

**Example:**
```nim
let n = newSinglyLinkedNode[int](5)
assert n.value == 5
assert n.next == nil
```

---

### `newDoublyLinkedNode`

```nim
proc newDoublyLinkedNode*[T](value: T): DoublyLinkedNode[T]
```

**Description:** Creates a new node for a doubly linked list or ring with the given value.

**Example:**
```nim
let n = newDoublyLinkedNode[int](5)
assert n.value == 5
assert n.next == nil
assert n.prev == nil
```

---

## `add` — Appending to the End

Appends a node or value to the **end** of the collection. Complexity: O(1) for all types.

### For `SinglyLinkedList`

```nim
proc add*[T](L: var SinglyLinkedList[T], n: SinglyLinkedNode[T])
proc add*[T](L: var SinglyLinkedList[T], value: T)
```

**Example:**
```nim
var a = initSinglyLinkedList[int]()
a.add(10)
a.add(20)
let n = newSinglyLinkedNode[int](30)
a.add(n)
assert $a == "[10, 20, 30]"
```

---

### For `DoublyLinkedList`

```nim
proc add*[T](L: var DoublyLinkedList[T], n: DoublyLinkedNode[T])
proc add*[T](L: var DoublyLinkedList[T], value: T)
```

**Example:**
```nim
var list = initDoublyLinkedList[int]()
let
  a = newDoublyLinkedNode[int](3)
  b = newDoublyLinkedNode[int](7)
list.add(a)
list.add(b)
assert a.next == b
assert b.prev == a
assert b.next == nil
```

---

### For `SinglyLinkedRing`

```nim
proc add*[T](L: var SinglyLinkedRing[T], n: SinglyLinkedNode[T])
proc add*[T](L: var SinglyLinkedRing[T], value: T)
```

**Example:**
```nim
var ring = initSinglyLinkedRing[int]()
ring.add(1)
ring.add(2)
ring.add(3)
assert ring.tail.next == ring.head  # ring is closed
```

---

### For `DoublyLinkedRing`

```nim
proc add*[T](L: var DoublyLinkedRing[T], n: DoublyLinkedNode[T])
proc add*[T](L: var DoublyLinkedRing[T], value: T)
```

**Example:**
```nim
var ring = initDoublyLinkedRing[int]()
ring.add(1)
ring.add(2)
assert ring.head.prev.value == 2        # last element is 2
assert ring.head.prev.next == ring.head # ring is closed
```

---

## `prepend` — Adding to the Beginning

Adds a node or value to the **beginning** of the collection. Complexity: O(1).

### For `SinglyLinkedList`

```nim
proc prepend*[T](L: var SinglyLinkedList[T], n: SinglyLinkedNode[T])
proc prepend*[T](L: var SinglyLinkedList[T], value: T)
```

**Example:**
```nim
var a = initSinglyLinkedList[int]()
a.add(30)
a.prepend(20)
a.prepend(10)
assert $a == "[10, 20, 30]"
```

---

### For `DoublyLinkedList`

```nim
proc prepend*[T](L: var DoublyLinkedList[T], n: DoublyLinkedNode[T])
proc prepend*[T](L: var DoublyLinkedList[T], value: T)
```

**Example:**
```nim
var list = initDoublyLinkedList[int]()
let
  a = newDoublyLinkedNode[int](3)
  b = newDoublyLinkedNode[int](7)
  c = newDoublyLinkedNode[int](9)
list.add(a)
list.add(b)
list.prepend(c)
# Order: c → a → b
assert c.next == a
assert a.prev == c
assert c.prev == nil  # c is the head, no predecessor
```

---

### For `SinglyLinkedRing`

```nim
proc prepend*[T](L: var SinglyLinkedRing[T], n: SinglyLinkedNode[T])
proc prepend*[T](L: var SinglyLinkedRing[T], value: T)
```

**Example:**
```nim
var ring = initSinglyLinkedRing[int]()
ring.add(3)      # [3]
ring.add(7)      # [3, 7]
ring.prepend(9)  # [9, 3, 7]
assert ring.head.value == 9
assert ring.head.next.value == 3
assert ring.tail.next == ring.head  # ring is closed
```

---

### For `DoublyLinkedRing`

```nim
proc prepend*[T](L: var DoublyLinkedRing[T], n: DoublyLinkedNode[T])
proc prepend*[T](L: var DoublyLinkedRing[T], value: T)
```

**Example:**
```nim
var ring = initDoublyLinkedRing[int]()
ring.add(2)
ring.add(3)
ring.prepend(1)  # [1, 2, 3]
assert ring.head.value == 1
```

---

## `add` List Concatenation

Appends a **shallow copy** of list `b` to the end of list `a`. Works only for `SomeLinkedList` (not rings).

```nim
proc add*[T: SomeLinkedList](a: var T, b: T)
```

**Description:** `a` receives copies of all nodes from `b`. The original `b` is not modified. Self-appending (`a.add(a)`) is supported.

**Example:**
```nim
from std/sequtils import toSeq
var a = [1, 2, 3].toSinglyLinkedList
let b = [4, 5].toSinglyLinkedList
a.add(b)
assert a.toSeq == [1, 2, 3, 4, 5]
assert b.toSeq == [4, 5]  # b is unchanged

a.add(a)  # self-append is safe
assert a.toSeq == [1, 2, 3, 4, 5, 1, 2, 3, 4, 5]
```

---

## `addMoved` — Move-Concatenation

Moves list `b` to the end of `a`. Complexity: O(1). After the operation `b` becomes empty (if `a` and `b` are different objects).

### For `SinglyLinkedList`

```nim
proc addMoved*[T](a, b: var SinglyLinkedList[T])
```

### For `DoublyLinkedList`

```nim
proc addMoved*[T](a, b: var DoublyLinkedList[T])
```

**Example:**
```nim
from std/sequtils import toSeq
var
  a = [1, 2, 3].toSinglyLinkedList
  b = [4, 5].toSinglyLinkedList
a.addMoved(b)
assert a.toSeq == [1, 2, 3, 4, 5]
assert b.toSeq == []  # b is now empty
```

> **Self-appending** (`a.addMoved(a)`) creates a cycle — iterating over the result is infinite. Use with caution.

---

## `prepend` list + list

Prepends a **shallow copy** of `b` to the beginning of `a`. Complexity: O(|b|) due to copying. Works only for `SomeLinkedList`.

```nim
proc prepend*[T: SomeLinkedList](a: var T, b: T)
```

**Example:**
```nim
from std/sequtils import toSeq
var a = [4, 5].toSinglyLinkedList
let b = [1, 2, 3].toSinglyLinkedList
a.prepend(b)
assert a.toSeq == [1, 2, 3, 4, 5]
assert b.toSeq == [1, 2, 3]  # b is unchanged

a.prepend(a)  # self-prepend is supported
assert a.toSeq == [1, 2, 3, 4, 5, 1, 2, 3, 4, 5]
```

---

## `prependMoved` — Move-Prepend

Moves `b` to the beginning of `a`. Complexity: O(1). After the operation `b` becomes empty.

```nim
proc prependMoved*[T: SomeLinkedList](a, b: var T)
```

**Example:**
```nim
from std/sequtils import toSeq
var
  a = [4, 5].toSinglyLinkedList
  b = [1, 2, 3].toSinglyLinkedList
a.prependMoved(b)
assert a.toSeq == [1, 2, 3, 4, 5]
assert b.toSeq == []  # b is now empty
```

> **Self-prepend** (`a.prependMoved(a)`) creates a cycle.

---

## `copy` — Shallow Copy

Creates a shallow copy of the list. Works only for `SomeLinkedList`.

```nim
func copy*[T](a: SinglyLinkedList[T]): SinglyLinkedList[T]
func copy*[T](a: DoublyLinkedList[T]): DoublyLinkedList[T]
```

**Description:** New nodes are created, but their `value` fields point to the same objects (for ref types). For value types (`int`, `string`) this is equivalent to a deep copy.

**Example:**
```nim
from std/sequtils import toSeq

type Foo = ref object
  x: int

var f = Foo(x: 1)
var a = [f].toSinglyLinkedList
let b = a.copy
a.add([f].toSinglyLinkedList)

assert a.toSeq == [f, f]
assert b.toSeq == [f]  # b is structurally independent...
f.x = 42
assert b.head.value.x == 42  # ...but elements are shared
```

---

## `remove` — Node Removal

### For `SinglyLinkedList`

```nim
proc remove*[T](L: var SinglyLinkedList[T], n: SinglyLinkedNode[T]): bool {.discardable.}
```

**Description:** Removes node `n` from list `L`. Returns `true` if the node was found and removed. Complexity: O(n) — the list is traversed until `n` is found. Attempting to remove an absent node is a no-op. If the list is cyclic (after `addMoved(a, a)`), the cycle is preserved after removal.

**Example:**
```nim
from std/sequtils import toSeq
var a = [0, 1, 2].toSinglyLinkedList
let n = a.head.next  # node with value 1
assert n.value == 1
assert a.remove(n) == true
assert a.toSeq == [0, 2]
assert a.remove(n) == false  # already removed — no-op
```

---

### For `DoublyLinkedList`

```nim
proc remove*[T](L: var DoublyLinkedList[T], n: DoublyLinkedNode[T])
```

**Description:** Removes node `n` from the list. Complexity: O(1) thanks to `prev`/`next` pointers. Assumes `n` belongs to `L`; behaviour is undefined otherwise.

**Example:**
```nim
from std/sequtils import toSeq
var a = initDoublyLinkedList[int]()
for i in 1..5:
  a.add(10 * i)
# Remove the node with value 30, transform the rest via nodes
for x in nodes(a):
  if x.value == 30:
    a.remove(x)
  else:
    x.value = 5 * x.value - 1
assert $a == "[49, 99, 199, 249]"
```

---

### For `DoublyLinkedRing`

```nim
proc remove*[T](L: var DoublyLinkedRing[T], n: DoublyLinkedNode[T])
```

**Description:** Removes a node from the ring. Complexity: O(1). Assumes `n` belongs to `L`. If the sole remaining element is removed, `head` becomes `nil`.

**Example:**
```nim
var ring = initDoublyLinkedRing[int]()
let n = newDoublyLinkedNode[int](5)
ring.add(n)
assert 5 in ring
ring.remove(n)
assert 5 notin ring
assert ring.head == nil  # ring is now empty
```

---

## `find` and `contains` — Search

### `find`

```nim
proc find*[T](L: SomeLinkedCollection[T], value: T): SomeLinkedNode[T]
```

**Description:** Finds the first node with the given value. Returns `nil` if not found. Works for all four collection types. Complexity: O(n).

**Example:**
```nim
let a = [9, 8].toSinglyLinkedList
assert a.find(9).value == 9
assert a.find(1) == nil
```

---

### `contains`

```nim
proc contains*[T](L: SomeLinkedCollection[T], value: T): bool
```

**Description:** Returns `true` if the value is present in the collection. Enables the `in` and `notin` operators. Implemented via `find`. Complexity: O(n).

**Example:**
```nim
let a = [9, 8].toSinglyLinkedList
assert a.contains(9)
assert 8 in a
assert not a.contains(1)
assert 2 notin a
```

---

## `$` String Representation

```nim
proc `$`*[T](L: SomeLinkedCollection[T]): string
```

**Description:** Converts any collection to a string of the form `[e1, e2, e3]`. Called automatically by `echo`.

**Example:**
```nim
let a = [1, 2, 3, 4].toSinglyLinkedList
assert $a == "[1, 2, 3, 4]"

var ring = initSinglyLinkedRing[int]()
ring.add(10)
ring.add(20)
assert $ring == "[10, 20]"
```

---

## Iterators

### `items`

```nim
iterator items*[T](L: SomeLinkedList[T]): T
iterator items*[T](L: SomeLinkedRing[T]): T
```

**Description:** Yields the value of every node. For lists: from `head` to the end. For rings: from `head` back to `head` (one full cycle). Used in `for x in L`.

**Example:**
```nim
from std/sequtils import toSeq
let a = [10, 20, 30].toSinglyLinkedList
assert toSeq(a) == @[10, 20, 30]

var ring = initSinglyLinkedRing[int]()
for i in 1..3: ring.add(10 * i)
assert toSeq(ring) == @[10, 20, 30]
```

---

### `mitems`

```nim
iterator mitems*[T](L: var SomeLinkedList[T]): var T
iterator mitems*[T](L: var SomeLinkedRing[T]): var T
```

**Description:** Yields every value in a mutable form. The collection must be declared as `var`.

**Example:**
```nim
var a = initSinglyLinkedList[int]()
for i in 1..5:
  a.add(10 * i)
assert $a == "[10, 20, 30, 40, 50]"
for x in mitems(a):
  x = 5 * x - 1
assert $a == "[49, 99, 149, 199, 249]"
```

---

### `nodes`

```nim
iterator nodes*[T](L: SomeLinkedList[T]): SomeLinkedNode[T]
iterator nodes*[T](L: SomeLinkedRing[T]): SomeLinkedNode[T]
```

**Description:** Iterates over **nodes** rather than values. Removing the current node during traversal is explicitly supported — it is safe in contrast to modifying the list inside `items`. Returns a ref to each node, allowing both `value` modification and structural changes.

**Example (list):**
```nim
var a = initDoublyLinkedList[int]()
for i in 1..5:
  a.add(10 * i)
for x in nodes(a):
  if x.value == 30:
    a.remove(x)   # safe during traversal
  else:
    x.value = 5 * x.value - 1
assert $a == "[49, 99, 199, 249]"
```

**Example (ring):**
```nim
var ring = initDoublyLinkedRing[int]()
for i in 1..5:
  ring.add(10 * i)
for x in nodes(ring):
  if x.value == 30:
    ring.remove(x)
  else:
    x.value = 5 * x.value - 1
assert $ring == "[49, 99, 199, 249]"
```

---

## Converting from Arrays

Creates collections from an `openArray[T]` by calling `add` for each element in order.

### `toSinglyLinkedList`

```nim
func toSinglyLinkedList*[T](elems: openArray[T]): SinglyLinkedList[T]
```

**Example:**
```nim
from std/sequtils import toSeq
let a = [1, 2, 3, 4, 5].toSinglyLinkedList
assert a.toSeq == [1, 2, 3, 4, 5]
```

---

### `toDoublyLinkedList`

```nim
func toDoublyLinkedList*[T](elems: openArray[T]): DoublyLinkedList[T]
```

**Example:**
```nim
from std/sequtils import toSeq
let a = [1, 2, 3, 4, 5].toDoublyLinkedList
assert a.toSeq == [1, 2, 3, 4, 5]
assert a.head.value == 1
assert a.tail.value == 5
assert a.tail.prev.value == 4
```

---

### `toSinglyLinkedRing`

```nim
func toSinglyLinkedRing*[T](elems: openArray[T]): SinglyLinkedRing[T]
```

**Example:**
```nim
from std/sequtils import toSeq
let a = [1, 2, 3, 4, 5].toSinglyLinkedRing
assert a.toSeq == [1, 2, 3, 4, 5]
assert a.tail.next == a.head  # ring is closed
```

---

### `toDoublyLinkedRing`

```nim
func toDoublyLinkedRing*[T](elems: openArray[T]): DoublyLinkedRing[T]
```

**Example:**
```nim
from std/sequtils import toSeq
let a = [1, 2, 3, 4, 5].toDoublyLinkedRing
assert a.toSeq == [1, 2, 3, 4, 5]
assert a.head.prev.value == 5  # predecessor of head is the last element
```

---

## Aliases `append` and `appendMoved`

### `append`

```nim
proc append*[T](a: var (SinglyLinkedList[T] | SinglyLinkedRing[T]),
                b: SinglyLinkedList[T] | SinglyLinkedNode[T] | T)

proc append*[T](a: var (DoublyLinkedList[T] | DoublyLinkedRing[T]),
                b: DoublyLinkedList[T] | DoublyLinkedNode[T] | T)
```

**Description:** Alias for `add`. Behaviour is identical.

---

### `appendMoved`

```nim
proc appendMoved*[T: SomeLinkedList](a, b: var T)
```

**Description:** Alias for `addMoved`. Behaviour is identical.

---

## Summary Table

| Function | SinglyLinked List | DoublyLinked List | SinglyLinked Ring | DoublyLinked Ring | Complexity |
|----------|:-----------------:|:-----------------:|:-----------------:|:-----------------:|------------|
| `init*` | ✓ | ✓ | ✓ | ✓ | O(1) |
| `newSinglyLinkedNode` | ✓ | — | ✓ | — | O(1) |
| `newDoublyLinkedNode` | — | ✓ | — | ✓ | O(1) |
| `add` (node/value) | ✓ | ✓ | ✓ | ✓ | O(1) |
| `prepend` (node/value) | ✓ | ✓ | ✓ | ✓ | O(1) |
| `add` (list+list) | ✓ | ✓ | — | — | O(\|b\|) |
| `addMoved` | ✓ | ✓ | — | — | O(1) |
| `prepend` (list+list) | ✓ | ✓ | — | — | O(\|b\|) |
| `prependMoved` | ✓ | ✓ | — | — | O(1) |
| `copy` | ✓ | ✓ | — | — | O(n) |
| `remove` | ✓ O(n) | ✓ O(1) | — | ✓ O(1) | — |
| `find` | ✓ | ✓ | ✓ | ✓ | O(n) |
| `contains` / `in` | ✓ | ✓ | ✓ | ✓ | O(n) |
| `$` | ✓ | ✓ | ✓ | ✓ | O(n) |
| `items` | ✓ | ✓ | ✓ | ✓ | — |
| `mitems` | ✓ | ✓ | ✓ | ✓ | — |
| `nodes` | ✓ | ✓ | ✓ | ✓ | — |
| `to*` (from array) | ✓ | ✓ | ✓ | ✓ | O(n) |
| `append` / `appendMoved` | ✓ | ✓ | ✓ | ✓ | — |

---

## Comprehensive Example

```nim
import std/lists
from std/sequtils import toSeq

# --- SinglyLinkedList: basic operations ---
var slist = initSinglyLinkedList[int]()
for i in 1..5:
  slist.add(i * 10)
slist.prepend(0)
assert $slist == "[0, 10, 20, 30, 40, 50]"

# Remove node with value 30 via safe nodes iterator
for node in nodes(slist):
  if node.value == 30:
    slist.remove(node)
assert $slist == "[0, 10, 20, 40, 50]"

# --- DoublyLinkedList: bidirectional access ---
var dlist = [1, 2, 3, 4, 5].toDoublyLinkedList
assert dlist.head.value == 1
assert dlist.tail.value == 5
assert dlist.tail.prev.value == 4

# Modify values in place via mitems
for x in mitems(dlist):
  x *= 2
assert $dlist == "[2, 4, 6, 8, 10]"

# --- Concatenation ---
var a = [1, 2, 3].toSinglyLinkedList
var b = [4, 5].toSinglyLinkedList
a.addMoved(b)          # O(1), b becomes empty
assert a.toSeq == @[1, 2, 3, 4, 5]
assert b.toSeq == @[]

# --- SinglyLinkedRing: circular structure ---
var ring = [1, 2, 3].toSinglyLinkedRing
assert ring.tail.next == ring.head  # closed
assert ring.contains(2)

# --- DoublyLinkedRing: O(1) removal ---
var dring = initDoublyLinkedRing[int]()
dring.add(10)
dring.add(20)
dring.add(30)
for node in nodes(dring):
  if node.value == 20:
    dring.remove(node)  # O(1)
assert $dring == "[10, 30]"
assert dring.head.prev.value == 30  # ring is preserved
```
