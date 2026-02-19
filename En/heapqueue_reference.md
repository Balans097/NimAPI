# Reference: `heapqueue` Module (Nim)

> The `heapqueue` module implements a **binary heap** — a data structure usable as a **priority queue**.  
> The internal array always satisfies the heap invariant: `a[k] <= a[2*k+1]` and `a[k] <= a[2*k+2]`. This guarantees that **the smallest element is always at `heap[0]`**.

---

## Table of Contents

- [Data Type](#data-type)
- [Initialization](#initialization)
- [Adding Elements](#adding-elements)
- [Removing Elements](#removing-elements)
- [Combined Operations](#combined-operations)
- [Deletion by Index and Clearing](#deletion-by-index-and-clearing)
- [Access and Search](#access-and-search)
- [Size and Iteration](#size-and-iteration)
- [Miscellaneous](#miscellaneous)
- [Custom Types](#custom-types)
- [Quick Reference](#quick-reference)

---

## Data Type

### `HeapQueue[T]`

A binary min-heap backed by a `seq[T]`. The smallest element is always accessible as `heap[0]`. Has **value semantics**: the `=` operator performs a full copy.

To use `HeapQueue` with a custom type, you must implement the `<` operator for that type.

```nim
var heap: HeapQueue[int]
```

---

## Initialization

### `initHeapQueue[T](): HeapQueue[T]`

Creates and returns a new empty heap. Since Nim v0.20, heaps are initialized by default, so an explicit call is usually unnecessary.

```nim
var heap = initHeapQueue[int]()
heap.push(5)
heap.push(1)
heap.push(3)
assert heap[0] == 1   # smallest element is always first
```

---

### `toHeapQueue[T](x: openArray[T]): HeapQueue[T]`

Creates a new heap from an array or sequence `x`. Uses Floyd's heap-construction algorithm for **O(n)** time — more efficient than n individual `push` calls.

> Available since Nim v1.3.

```nim
var heap = [9, 5, 8, 1, 3].toHeapQueue
assert heap[0] == 1    # smallest element at root

# Note: the order of other elements reflects heap structure,
# not the original array order
assert heap.pop() == 1
assert heap.pop() == 3
assert heap.pop() == 5
```

---

## Adding Elements

### `push[T](heap: var HeapQueue[T], item: sink T)`

Pushes `item` onto the heap while maintaining the heap invariant. Complexity: **O(log n)**.

```nim
var heap = initHeapQueue[int]()
heap.push(10)
heap.push(2)
heap.push(7)
heap.push(1)
assert heap[0] == 1   # smallest always at root

# Adding in a loop:
var h = initHeapQueue[string]()
for word in ["banana", "apple", "cherry"]:
  h.push(word)
assert h[0] == "apple"
```

---

## Removing Elements

### `pop[T](heap: var HeapQueue[T]): T`

Removes and returns the **smallest** element from the heap, maintaining the invariant. Complexity: **O(log n)**.

> ⚠️ Calling on an empty heap causes a panic or undefined behaviour.

```nim
var heap = [9, 5, 8, 1, 3].toHeapQueue

# Elements are returned in ascending order:
assert heap.pop() == 1
assert heap.pop() == 3
assert heap.pop() == 5
assert heap.pop() == 8
assert heap.pop() == 9

# Heap-sort pattern:
var h = [3, 1, 4, 1, 5, 9, 2, 6].toHeapQueue
var sorted: seq[int]
while h.len > 0:
  sorted.add(h.pop())
assert sorted == @[1, 1, 2, 3, 4, 5, 6, 9]
```

---

## Combined Operations

### `replace[T](heap: var HeapQueue[T], item: sink T): T`

Atomically pops and returns the current smallest element, then places `item` at the root and sifts down. More efficient than `pop()` + `push()` because only one sift pass is needed.

> ⚠️ The returned value **may be larger than `item`**. Make sure this is acceptable in your logic.

```nim
var heap = [5, 12, 20].toHeapQueue
let old = heap.replace(6)
assert old == 5       # the previous minimum was returned
assert heap.len == 3  # size unchanged
assert heap[0] == 6   # 6 is the new minimum

# Case where the result > item:
let old2 = heap.replace(4)
assert old2 == 6      # current minimum (6) was returned
assert heap[0] == 4   # 4 is now the minimum
```

---

### `pushpop[T](heap: var HeapQueue[T], item: sink T): T`

Fast combination of `push()` + `pop()`. Pushes `item` and immediately returns the smallest of all elements (including the new one). The heap size remains unchanged.

If `item` is smaller than or equal to the current minimum, it is returned immediately without modifying the heap.

```nim
var heap = [5, 12].toHeapQueue

# 6 > 5, so the current minimum (5) is returned and 6 is added:
assert heap.pushpop(6) == 5
assert heap[0] == 6

# 4 < 6 (current minimum), so 4 is returned immediately:
assert heap.pushpop(4) == 4
assert heap[0] == 6   # heap unchanged
```

**Difference between `replace` and `pushpop`:**

| | `replace(item)` | `pushpop(item)` |
|---|---|---|
| Returns | current minimum (always) | `min(current_min, item)` |
| When item < min | item enters, old min exits | item exits immediately, heap unchanged |
| Typical use | swap root with new value | stream filtering |

---

## Deletion by Index and Clearing

### `del[T](heap: var HeapQueue[T], index: Natural)`

Removes the element at `index` from the heap while maintaining the invariant. Complexity: **O(log n)**.

Use `find` to get the index beforehand. Note that after `del`, the indices of other elements may change.

```nim
var heap = [9, 5, 8].toHeapQueue
# heap[0] == 5, heap[1] == 9, heap[2] == 8

heap.del(1)   # remove element at index 1 (which is 9)
assert heap.len == 2
assert heap[0] == 5

# Typical pattern: find and remove a specific value:
var h = [10, 30, 20, 40].toHeapQueue
let idx = h.find(30)
if idx >= 0:
  h.del(idx)
```

---

### `clear[T](heap: var HeapQueue[T])`

Removes all elements from the heap. Complexity: O(1) (resets the length of the internal `seq`).

```nim
var heap = [9, 5, 8].toHeapQueue
assert heap.len == 3
heap.clear()
assert heap.len == 0
```

---

## Access and Search

### `heap[i: Natural]: lent T`  — operator `[]`

Returns the element at index `i` in the internal array. `heap[0]` is always the smallest element. Other indices **do not correspond** to sorted order.

```nim
let heap = [9, 5, 8, 1].toHeapQueue
assert heap[0] == 1   # smallest
# heap[1], heap[2], heap[3] are in heap order, not sorted order
```

---

### `find[T](heap: HeapQueue[T], x: T): int`

Linear scan for element `x`. Returns its **index** in the internal array, or `-1` if not found. Complexity: **O(n)**.

> Available since Nim v1.3.

```nim
let heap = [9, 5, 8].toHeapQueue
assert heap.find(5) == 0    # 5 is the min, at index 0
assert heap.find(9) == 1
assert heap.find(777) == -1  # not found
```

---

### `contains[T](heap: HeapQueue[T], x: T): bool`

Returns `true` if `x` is present in the heap. Equivalent to `find(heap, x) >= 0`. Supports `in` / `notin` syntax. Complexity: **O(n)**.

> Available since Nim v2.1.1.

```nim
let heap = [9, 5, 8].toHeapQueue
assert 5 in heap
assert heap.contains(8)
assert 42 notin heap
```

---

## Size and Iteration

### `len[T](heap: HeapQueue[T]): int`

Returns the number of elements in the heap. Complexity: O(1).

```nim
let heap = [9, 5, 8].toHeapQueue
assert heap.len == 3
```

---

### `items[T](heap: HeapQueue[T]): lent T`  (iterator)

Iterates over all elements in **internal array order** (not in ascending order!). Modifying the heap during iteration is not allowed.

> Available since Nim v2.1.1.

```nim
let heap = [5, 9, 8, 1, 3].toHeapQueue
for x in heap:
  echo x   # order reflects heap structure, not sorted order

# To iterate in sorted (ascending) order, use pop in a loop:
var h = [5, 9, 8, 1, 3].toHeapQueue
while h.len > 0:
  echo h.pop()   # 1, 3, 5, 8, 9
```

---

## Miscellaneous

### `$[T](heap: HeapQueue[T]): string`

Converts the heap to a string representation of the form `[e1, e2, e3]`. Element order reflects the internal array. Intended for display and debugging.

```nim
let heap = [1, 2].toHeapQueue
echo $heap   # [1, 2]

let h = [9, 5, 8].toHeapQueue
echo $h      # [5, 9, 8] — heap order, not sorted
```

---

## Custom Types

To use `HeapQueue` with custom objects, implement the `<` operator. It defines the ordering: the element that is "less than" all others will sit at the root (`heap[0]`).

```nim
type Task = object
  priority: int
  name: string

# Lower priority value = higher urgency:
proc `<`(a, b: Task): bool = a.priority < b.priority

var tasks = initHeapQueue[Task]()
tasks.push(Task(priority: 3, name: "Low"))
tasks.push(Task(priority: 1, name: "Critical"))
tasks.push(Task(priority: 2, name: "Normal"))

assert tasks[0].name == "Critical"    # highest urgency at root
assert tasks.pop().name == "Critical"
assert tasks.pop().name == "Normal"
assert tasks.pop().name == "Low"
```

```nim
# Max-heap via inverted comparison:
type MaxInt = object
  val: int

proc `<`(a, b: MaxInt): bool = a.val > b.val   # inverted

var maxHeap = initHeapQueue[MaxInt]()
maxHeap.push(MaxInt(val: 3))
maxHeap.push(MaxInt(val: 7))
maxHeap.push(MaxInt(val: 1))
assert maxHeap[0].val == 7   # largest is now at root
```

---

## Quick Reference

### Operations table

| Operation | Proc | Complexity |
|---|---|---|
| Create empty heap | `initHeapQueue[T]()` | O(1) |
| Create from array | `toHeapQueue(arr)` | O(n) |
| Push element | `push(item)` | O(log n) |
| Pop minimum | `pop()` | O(log n) |
| Peek minimum | `heap[0]` | O(1) |
| Replace minimum | `replace(item)` | O(log n) |
| Push then pop | `pushpop(item)` | O(log n) |
| Delete by index | `del(index)` | O(log n) |
| Find index | `find(x)` | O(n) |
| Membership test | `contains(x)` / `in` | O(n) |
| Count elements | `len()` | O(1) |
| Clear all | `clear()` | O(1) |
| Iterate | `items` / `for x in heap` | O(n) |
| String repr | `$` | O(n) |

### Heap invariant visualised

```
           1  (index 0 — always the minimum)
          / \
         3   5  (indices 1, 2)
        / \ / \
       9  4 8  7  (indices 3, 4, 5, 6)

a[k] <= a[2*k+1]  and  a[k] <= a[2*k+2]
```

### Notes

- `HeapQueue` is a **min-heap**: `pop()` always returns the **smallest** element.
- For a **max-heap**, invert the `<` operator in your type.
- Iterating with `items` yields elements in **internal array order**, not sorted order. For sorted traversal, use `pop()` in a loop.
- Never call `pop()` or `replace()` on an empty heap.
- `replace` is more efficient than `pop` + `push`; `pushpop` is more efficient than `push` + `pop`.
