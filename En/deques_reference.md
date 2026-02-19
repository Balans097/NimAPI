# Reference: `deques` Module (Nim)

> The `deques` module implements a **double-ended queue** (deque) backed by a ring-buffer `seq`.  
> Elements can be efficiently added and removed from **both the front and the back** of the queue.

> ⚠️ **Warning:** never call element-access or pop procedures on an empty deque. With `boundChecks` enabled an `IndexDefect` will be raised; with `-d:danger` or `--checks:off` the behaviour is undefined.

---

## Table of Contents

- [Data Type](#data-type)
- [Initialization](#initialization)
- [Adding Elements](#adding-elements)
- [Peeking (Non-Destructive Read)](#peeking-non-destructive-read)
- [Removing Elements](#removing-elements)
- [Index-Based Access](#index-based-access)
- [Size and Clearing](#size-and-clearing)
- [Search](#search)
- [Iteration](#iteration)
- [Comparison and Miscellaneous](#comparison-and-miscellaneous)
- [Quick Reference](#quick-reference)

---

## Data Type

### `Deque[T]`

A double-ended queue backed by an internal ring-buffer `seq[T]`. Supports O(1) amortized insertion and removal at both ends. Has **value semantics**: the `=` operator performs a full copy.

```nim
var d: Deque[int]
```

### `defaultInitialSize`

A constant — the default initial buffer capacity (`4`).

---

## Initialization

### `initDeque[T](initialSize: int = defaultInitialSize): Deque[T]`

Creates and returns a new empty deque. The `initialSize` parameter sets the initial buffer capacity as a performance hint (rounded up to the nearest power of two); the length of the new deque is always 0.

```nim
var a = initDeque[int]()
var b = initDeque[string](32)   # pre-allocate capacity

a.addLast(1)
a.addLast(2)
assert a.len == 2
```

---

### `toDeque[T](x: openArray[T]): Deque[T]`

Creates a new deque containing the elements of array or sequence `x` in the same order.

> Available since Nim v1.3.

```nim
let a = toDeque([10, 20, 30, 40])
assert a.len == 4
assert $a == "[10, 20, 30, 40]"

# From a sequence
let s = @[1, 2, 3]
let d = toDeque(s)
```

---

## Adding Elements

### `addFirst[T](deq: var Deque[T], item: sink T)`

Adds `item` to the **front** of the deque. The buffer grows automatically if needed. Amortized complexity: O(1).

```nim
var a = initDeque[int]()
a.addFirst(10)
a.addFirst(20)
a.addFirst(30)
assert $a == "[30, 20, 10]"   # last inserted is first in queue
```

---

### `addLast[T](deq: var Deque[T], item: sink T)`

Adds `item` to the **back** of the deque. The buffer grows automatically if needed. Amortized complexity: O(1).

```nim
var a = initDeque[int]()
a.addLast(10)
a.addLast(20)
a.addLast(30)
assert $a == "[10, 20, 30]"   # insertion order preserved
```

---

## Peeking (Non-Destructive Read)

### `peekFirst[T](deq: Deque[T]): lent T`

Returns the **first** element of the deque **without removing** it. Raises `IndexDefect` on an empty deque.

```nim
let a = [10, 20, 30].toDeque
assert a.peekFirst == 10
assert a.len == 3   # deque unchanged
```

---

### `peekFirst[T](deq: var Deque[T]): var T`

Returns a **mutable reference** to the first element, allowing in-place modification without removal.

> Available since Nim v1.3.

```nim
var a = [10, 20, 30].toDeque
a.peekFirst() = 99
assert $a == "[99, 20, 30]"

inc(a.peekFirst())
assert $a == "[100, 20, 30]"
```

---

### `peekLast[T](deq: Deque[T]): lent T`

Returns the **last** element of the deque **without removing** it. Raises `IndexDefect` on an empty deque.

```nim
let a = [10, 20, 30].toDeque
assert a.peekLast == 30
assert a.len == 3
```

---

### `peekLast[T](deq: var Deque[T]): var T`

Returns a **mutable reference** to the last element.

> Available since Nim v1.3.

```nim
var a = [10, 20, 30].toDeque
a.peekLast() = 99
assert $a == "[10, 20, 99]"
```

---

## Removing Elements

### `popFirst[T](deq: var Deque[T]): T`

Removes and returns the **first** element. Raises `IndexDefect` on an empty deque. Complexity: O(1). The return value is `discardable`.

```nim
var a = [10, 20, 30, 40].toDeque
assert a.popFirst == 10
assert $a == "[20, 30, 40]"

# Drain the queue:
while a.len > 0:
  echo a.popFirst
```

---

### `popLast[T](deq: var Deque[T]): T`

Removes and returns the **last** element. Raises `IndexDefect` on an empty deque. Complexity: O(1). The return value is `discardable`.

```nim
var a = [10, 20, 30, 40].toDeque
assert a.popLast == 40
assert $a == "[10, 20, 30]"
```

---

### `clear[T](deq: var Deque[T])`

Resets the deque to an empty state, destroying all elements. The internal buffer **retains** its allocated memory. Complexity: O(n).

```nim
var a = [10, 20, 30, 40, 50].toDeque
assert a.len == 5
a.clear()
assert a.len == 0
```

---

### `shrink[T](deq: var Deque[T], fromFirst = 0, fromLast = 0)`

Removes `fromFirst` elements from the **front** and `fromLast` elements from the **back** in a single call. If the combined count exceeds the deque length, the deque is fully cleared.

```nim
var a = [10, 20, 30, 40, 50].toDeque
a.shrink(fromFirst = 2, fromLast = 1)
assert $a == "[30, 40]"

# Removing more than available is safe:
var b = [1, 2, 3].toDeque
b.shrink(fromFirst = 10, fromLast = 10)
assert b.len == 0
```

---

## Index-Based Access

### `deq[i: Natural]: lent T`

Returns the element at forward index `i` (0-based). Raises `IndexDefect` on out-of-bounds access. Complexity: O(1).

```nim
let a = [10, 20, 30, 40, 50].toDeque
assert a[0] == 10
assert a[4] == 50
```

---

### `deq[i: Natural]: var T`  (on `var Deque`)

Returns a **mutable reference** to the element at index `i`. Allows in-place modification.

```nim
var a = [10, 20, 30].toDeque
inc(a[0])
a[2] = 99
assert $a == "[11, 20, 99]"
```

---

### `deq[i] = val` — operator `[]=` (forward index)

Sets the element at index `i` to `val`.

```nim
var a = [10, 20, 30, 40, 50].toDeque
a[0] = 99
a[3] = 66
assert $a == "[99, 20, 30, 66, 50]"
```

---

### `deq[^i: BackwardsIndex]: lent T`

Returns the element at **backwards** index `i`: `deq[^1]` is the last element, `deq[^2]` is the second-to-last, and so on.

```nim
let a = [10, 20, 30, 40, 50].toDeque
assert a[^1] == 50
assert a[^2] == 40
assert a[^5] == 10
```

---

### `deq[^i: BackwardsIndex]: var T`  (on `var Deque`)

Returns a **mutable reference** at backwards index `i`.

```nim
var a = [10, 20, 30, 40, 50].toDeque
inc(a[^1])
assert a[^1] == 51
```

---

### `deq[^i] = val` — operator `[]=` (backwards index)

Sets the element at backwards index `i` to `val`.

```nim
var a = [10, 20, 30, 40, 50].toDeque
a[^1] = 99
a[^3] = 77
assert $a == "[10, 20, 77, 40, 99]"
```

---

## Size and Clearing

### `len[T](deq: Deque[T]): int`

Returns the current number of elements in the deque. Complexity: O(1).

```nim
var a = [1, 2, 3].toDeque
assert a.len == 3
a.addLast(4)
assert a.len == 4
discard a.popFirst()
assert a.len == 3
```

---

## Search

### `contains[T](deq: Deque[T], item: T): bool`

Returns `true` if `item` is present in the deque. Linear search, complexity: O(n). Enables `in` / `notin` operator syntax.

```nim
let q = [7, 9, 13].toDeque
assert 9 in q
assert q.contains(7)
assert 5 notin q
```

---

## Iteration

### `items[T](deq: Deque[T]): lent T`  (iterator)

Yields every element from **front to back** as read-only references. Modifying the deque during iteration is not allowed.

```nim
let a = [10, 20, 30].toDeque
for x in a:
  echo x   # 10, 20, 30

# Convert to a sequence:
from std/sequtils import toSeq
assert toSeq(a.items) == @[10, 20, 30]
```

---

### `mitems[T](deq: var Deque[T]): var T`  (iterator)

Yields every element as a **mutable reference**, allowing in-place modification during iteration.

```nim
var a = [10, 20, 30, 40, 50].toDeque
for x in mitems(a):
  x = x * 2
assert $a == "[20, 40, 60, 80, 100]"
```

---

### `pairs[T](deq: Deque[T]): tuple[key: int, val: T]`  (iterator)

Yields `(index, value)` pairs from front to back.

```nim
let a = [10, 20, 30].toDeque
for idx, val in a.pairs:
  echo idx, ": ", val
# 0: 10
# 1: 20
# 2: 30

from std/sequtils import toSeq
assert toSeq(a.pairs) == @[(0, 10), (1, 20), (2, 30)]
```

---

## Comparison and Miscellaneous

### `==[T](deq1, deq2: Deque[T]): bool`

Returns `true` if both deques contain the same elements **in the same order**.

```nim
var a, b = initDeque[int]()
a.addFirst(2); a.addFirst(1)   # a = [1, 2]
b.addLast(1);  b.addLast(2)    # b = [1, 2]
assert a == b

let c = [1, 2, 3].toDeque
assert a != c
```

---

### `hash[T](deq: Deque[T]): Hash`

Computes a hash of the deque, taking element order into account. Useful when storing a deque as a key in a hash table or hash set.

```nim
import std/tables
var t: Table[Deque[int], string]
t[[1, 2, 3].toDeque] = "abc"
```

---

### `$[T](deq: Deque[T]): string`

Converts the deque to a string in the format `[e1, e2, e3]`. Intended for display and debugging. **Do not use for serialization**.

```nim
let a = [10, 20, 30].toDeque
echo $a   # [10, 20, 30]

let b = ["hello", "world"].toDeque
echo $b   # ["hello", "world"]
```

---

## Quick Reference

### Operations table

| Operation | Proc / operator | Complexity |
|---|---|---|
| Create empty | `initDeque[T]()` | O(1) |
| Create from array | `toDeque(arr)` | O(n) |
| Add to front | `addFirst(item)` | O(1) amortized |
| Add to back | `addLast(item)` | O(1) amortized |
| Read first | `peekFirst()` | O(1) |
| Read last | `peekLast()` | O(1) |
| Remove first | `popFirst()` | O(1) |
| Remove last | `popLast()` | O(1) |
| Access by index | `deq[i]`, `deq[^i]` | O(1) |
| Assign by index | `deq[i] = val` | O(1) |
| Count elements | `len()` | O(1) |
| Search | `contains()` / `in` | O(n) |
| Clear all | `clear()` | O(n) |
| Trim from ends | `shrink(fromFirst, fromLast)` | O(k) |
| Equality | `==` | O(n) |
| Hash | `hash()` | O(n) |
| String repr | `$` | O(n) |
| Iterate (read) | `items` / `for x in deq` | O(n) |
| Iterate (write) | `mitems` | O(n) |
| Iterate with index | `pairs` | O(n) |

### Common usage patterns

```nim
# FIFO queue (first-in, first-out):
var queue = initDeque[int]()
queue.addLast(1)
queue.addLast(2)
queue.addLast(3)
echo queue.popFirst   # 1
echo queue.popFirst   # 2

# LIFO stack (last-in, first-out):
var stack = initDeque[int]()
stack.addLast(1)
stack.addLast(2)
stack.addLast(3)
echo stack.popLast   # 3
echo stack.popLast   # 2

# Fixed-size sliding window:
const windowSize = 3
var window = initDeque[int]()
for value in [1, 2, 3, 4, 5, 6]:
  window.addLast(value)
  if window.len > windowSize:
    discard window.popFirst
  echo $window
```

### Notes

- The internal buffer **doubles in size** automatically when capacity is exceeded (always kept as a power of two).
- `clear` **does not free** the buffer memory — subsequent insertions do not require reallocation.
- Do not modify a deque while iterating through `items`, `mitems`, or `pairs` — this triggers an assertion error.
- `popFirst` and `popLast` are marked `discardable`: the return value may be ignored when only the side effect (removal) is needed.
