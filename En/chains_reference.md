# chains — Reference Manual

## Overview

The `chains` module provides three **generic templates** for manipulating singly and doubly linked lists. Unlike most Nim container modules, `chains` does **not** define any types of its own. Instead, it works with whatever struct types the caller provides, as long as those types have the right field names. This is an intrusive linked-list approach: the linkage fields (`next`, `prev`) live directly inside your own data nodes, and a separate header object owns the `head`/`tail` pointers.

### The intrusive linked-list pattern

In a classic linked list library the library owns the node type and you store your data inside it. In an intrusive list the situation is reversed: **you own the node type and the library manipulates your objects directly**. The benefit is zero extra allocations — each node is the data object itself, and the same object can participate in multiple lists by having multiple sets of link fields.

`chains` uses Nim's compile-time `when compiles(...)` checks to adapt automatically to whatever combination of fields your types expose:

| Fields on header | Fields on node | List kind |
|---|---|---|
| `head` only | `next` only | Singly linked, forward only |
| `head` + `tail` | `next` only | Singly linked, tracked tail |
| `head` + `tail` | `next` + `prev` | Doubly linked (full) |

You never pass a type parameter — the templates inspect your actual objects at compile time and generate the appropriate code.

### Minimal setup

```nim
# A node type — any name is fine, fields must be named next/prev
type
  Node = ref object
    value: int
    next: Node      # required for all list operations
    prev: Node      # optional — enables doubly linked operations

# A header type — any name is fine, fields must be named head/tail
type
  List = object
    head: Node      # required
    tail: Node      # optional — enables O(1) append and tail tracking
```

The field types must be pointer-like (refs or pointers) so that `nil` represents the absence of a neighbour. There are no other constraints.

---

## Exported API

### `prepend`

```nim
template prepend*(header, node)
```

**Purpose:** Inserts `node` at the **front** of the list (before the current head). After the call, `node` becomes the new head. This is an O(1) operation regardless of list length.

**What the template does, step by step:**

1. If the header has a `head` field:
   - If `node.prev` exists and the list is non-empty, sets the old head's `prev` to point back to `node`.
   - Sets `node.next` to the old head.
   - Sets `header.head` to `node`.
2. If the header has a `tail` field and the list was empty, sets `header.tail` to `node` (the new node is both head and tail).

**Example — building a stack (LIFO) with prepend**

```nim
type
  Node = ref object
    val: int
    next: Node
    prev: Node

type
  List = object
    head: Node
    tail: Node

var list: List
var a = Node(val: 1)
var b = Node(val: 2)
var c = Node(val: 3)

list.prepend(a)   # list: [1]
list.prepend(b)   # list: [2, 1]
list.prepend(c)   # list: [3, 2, 1]

# Iterating from head gives insertion reverse order: 3, 2, 1
var cur = list.head
while cur != nil:
  echo cur.val    # 3, then 2, then 1
  cur = cur.next
```

**When to use `prepend`:** whenever you build a list by accumulating items and the final order doesn't matter, or when you specifically want the most-recently-added item first (stack semantics). It is always O(1).

---

### `append`

```nim
template append*(header, node)
```

**Purpose:** Inserts `node` at the **end** of the list (after the current tail). After the call, `node` becomes the new tail. This is an O(1) operation when the header has a `tail` field. Without a `tail` field, the template only updates `head` if the list was empty — it cannot reach the current tail without traversal, so there is no `next`-chaining of the new node to the previous tail in that case. For correct O(1) append you should always include `tail` in your header.

**What the template does, step by step:**

1. If the header has a `head` field and the list is empty, sets `header.head` to `node`.
2. If the header has a `tail` field:
   - If `node.prev` exists, sets `node.prev` to the old tail.
   - If the list was non-empty, sets the old tail's `next` to `node`.
   - Sets `header.tail` to `node`.

**Example — building a queue (FIFO) with append**

```nim
type
  Node = ref object
    val: int
    next: Node
    prev: Node

type
  List = object
    head: Node
    tail: Node

var list: List
var a = Node(val: 1)
var b = Node(val: 2)
var c = Node(val: 3)

list.append(a)   # list: [1]
list.append(b)   # list: [1, 2]
list.append(c)   # list: [1, 2, 3]

# Iterating from head gives insertion order: 1, 2, 3
var cur = list.head
while cur != nil:
  echo cur.val    # 1, then 2, then 3
  cur = cur.next
```

**When to use `append`:** whenever you want to preserve insertion order (queue semantics, event lists, task queues). Always include `tail` in your header for correct and efficient operation.

---

### `unlink`

```nim
template unlink*(header, node)
```

**Purpose:** Removes `node` from the list by relinking its neighbours around it. After the call, `node` is detached — its own `next` and `prev` fields are not cleared (they still point to their former neighbours), but neither neighbour points back to `node` anymore. The header's `head` and `tail` are updated if `node` happened to be the first or last element. This is an O(1) operation.

**What the template does, step by step:**

1. If `node.next` is not nil, sets `node.next.prev` to `node.prev` (skip over `node` backwards).
2. If `node.prev` is not nil, sets `node.prev.next` to `node.next` (skip over `node` forwards).
3. If `header.head == node`, sets `header.head` to `node.prev`.
4. If `header.tail == node`, sets `header.tail` to `node.next`.

**Note on the head/tail update logic:** the assignments `header.head = node.prev` and `header.tail = node.next` look counterintuitive (you might expect `node.next` for a new head). This reflects an assumption that the list may be traversed in *either* direction — `prev` of the head is the "previous in traversal order" which in a typical doubly linked ring would be the predecessor. For a standard singly-linked or doubly-linked list where `prev` of the first node is `nil`, removing the head sets `header.head = nil` correctly only if `node.prev == nil`. Be aware of this if your pointer structure differs from the canonical layout.

**Example — removing a node from the middle**

```nim
type
  Node = ref object
    val: int
    next: Node
    prev: Node

type
  List = object
    head: Node
    tail: Node

var list: List
var a = Node(val: 1)
var b = Node(val: 2)
var c = Node(val: 3)

list.append(a)
list.append(b)
list.append(c)
# list: [1] <-> [2] <-> [3]

list.unlink(b)
# list: [1] <-> [3]
# b.next still points to c, b.prev still points to a (stale, ignore them)

assert list.head.val == 1
assert list.tail.val == 3
assert list.head.next.val == 3
```

**Example — removing the head**

```nim
list.unlink(a)
# list: [3]
assert list.head.val == 3
assert list.tail.val == 3
```

**When to use `unlink`:** whenever you need to remove an arbitrary known node from the middle, front, or back of the list in O(1) time. Because the node must be doubly linked for `prev`-based relinking, `unlink` effectively requires `prev` on the node type. Calling `unlink` on a singly linked node (no `prev` field) will compile but will produce wrong results or a compile error because `node.next.prev` won't exist.

---

## Combining the templates: a complete example

The following example implements a simple doubly linked priority queue where tasks can be inserted at the front or back and removed individually by reference.

```nim
type
  Task = ref object
    name: string
    next: Task
    prev: Task

type
  TaskQueue = object
    head: Task
    tail: Task

proc newTask(name: string): Task =
  Task(name: name)

var q: TaskQueue

let t1 = newTask("low-priority job")
let t2 = newTask("high-priority job")
let t3 = newTask("normal job")

q.append(t1)    # [low]
q.prepend(t2)   # [high, low]
q.append(t3)    # [high, low, normal]

# Process high-priority first (from head)
echo q.head.name    # "high-priority job"

# Cancel the normal job while it's still in the queue
q.unlink(t3)        # [high, low]

# Drain the queue
var cur = q.head
while cur != nil:
  echo "processing: ", cur.name
  let next = cur.next
  q.unlink(cur)
  cur = next
```

---

## Design notes and caveats

**Node must not be in two lists simultaneously (with the same fields).** Because `prev` and `next` are fields on the node itself, a node can only track one set of neighbours at a time. If you need a node to participate in two lists, give it two separate pairs of link fields (e.g. `nextByTime`, `prevByTime`, `nextByPriority`, `prevByPriority`) and create two separate header types.

**`unlink` does not nil out the unlinked node's fields.** After `unlink(header, node)`, `node.next` and `node.prev` still hold their old values. Do not traverse through an unlinked node's link fields — they are stale. If your code might accidentally do so, nil them out manually after unlinking.

**No bounds or nil-checks beyond what the template provides.** The templates do not guard against unlinking a node that is not in the list, prepending/appending `nil`, or double-unlinking. Such misuse will silently corrupt the list structure.

**Singly linked lists:** if your node type has only `next` (no `prev`), `prepend` and `append` still compile and work for maintaining `head`/`tail`. However, `unlink` requires `prev` for correct relinking of neighbours — without `prev`, the `node.next.prev = ...` assignment will fail to compile or produce wrong results.
