# enumerate — Module Reference

## Overview

The `enumerate` module provides a single macro of the same name that brings Python-style enumerated iteration to Nim. Instead of manually maintaining a counter variable alongside your loop variable, `enumerate` pairs each element with its index automatically, yielding `(index, value)` tuples on every iteration.

Under the hood `enumerate` is a **macro that rewrites a `for` loop** at compile time. It does not create a new runtime iterator or allocate any extra memory; it simply transforms the special `for … in enumerate(…)` syntax into an ordinary `for` loop with a counter variable prepended to it. The transformation is invisible to the programmer but produces exactly the same machine code as the manual counter approach.

**Availability:** Nim 1.3 and later.

---

## `enumerate`

```nim
macro enumerate*(x: ForLoopStmt): untyped
```

### Description

`enumerate` wraps any iterable — arrays, sequences, strings, custom iterators — and makes both the current **index** (an `int` counter starting at `0` by default) and the current **value** available in the loop body simultaneously.

The macro accepts an optional first argument to set a **custom starting value** for the counter. This is useful when you need 1-based indexing, when you want to continue counting from a previous loop's ending point, or when the numeric values of the indices carry domain meaning (for example, ASCII code points, line numbers, etc.).

Because `enumerate` is a `ForLoopStmt` macro — a special macro kind in Nim that intercepts `for` loop syntax — it is invoked purely through the `for` statement and never as a regular function call. You do not call it and store its result; you write a `for` loop whose iterable is `enumerate(…)`.

### Syntax

```nim
# Default counter starting at 0
for i, value in enumerate(collection):
  …

# Custom starting counter
for i, value in enumerate(startValue, collection):
  …

# Tuple unpacking syntax (also valid)
for (i, value) in enumerate(collection):
  …

for (i, value) in enumerate(startValue, collection):
  …
```

### Parameters (inside the `for` statement)

| Position | Role | Description |
|---|---|---|
| First `for` variable | Index | Receives the current counter value. Always an `int`. |
| Remaining `for` variables | Value(s) | Receive the elements yielded by the underlying iterator. |
| First `enumerate` argument (optional) | Start value | Integer expression giving the initial counter value. Defaults to `0` if omitted. |
| Last `enumerate` argument | Collection | The iterable to loop over: array, seq, string, iterator, etc. |

### How the macro transforms the loop

To understand the behaviour precisely it helps to see what `enumerate` expands into. Given:

```nim
for i, x in enumerate(collection):
  body
```

the macro generates the equivalent of:

```nim
block:
  var i = -1          # initialised to startValue − 1
  for x in collection:
    inc i             # incremented to startValue before the body runs
    body
```

The counter is pre-decremented by one and then incremented at the very start of each loop iteration. This approach allows the counter variable to hold its final value after the loop finishes, which is important for composability with other macros that inspect the loop state.

### Unpacking requirement

The macro **requires** that both the index and the value are bound to names. You cannot write `for i in enumerate(collection)` with a single variable — there is nowhere to put the value. The macro will emit a compile-time error (`"Missing second for loop variable"`) if you attempt this. Use either the comma form (`for i, x in`) or the tuple-unpacking form (`for (i, x) in`).

---

### Examples

**Basic usage — iterating a sequence with indices:**

```nim
import std/enumerate

let fruits = @["apple", "banana", "cherry"]
for i, fruit in enumerate(fruits):
  echo i, ": ", fruit
# 0: apple
# 1: banana
# 2: cherry
```

**Iterating a string character by character:**

```nim
import std/enumerate

for i, ch in enumerate("hello"):
  echo i, " → ", ch
# 0 → h
# 1 → e
# 2 → l
# 3 → l
# 4 → o
```

**Custom starting index — 1-based numbering:**

```nim
import std/enumerate

let tasks = ["Write tests", "Fix bug", "Review PR"]
for i, task in enumerate(1, tasks):
  echo "Task ", i, ": ", task
# Task 1: Write tests
# Task 2: Fix bug
# Task 3: Review PR
```

**Custom starting index — using ASCII code points as indices:**

```nim
import std/enumerate

for code, ch in enumerate(97, "abcd"):
  echo ch, " = ", code
# a = 97
# b = 98
# c = 99
# d = 100
```

**Tuple-unpacking form:**

```nim
import std/enumerate

let values = [100, 200, 300]
for (i, v) in enumerate(values):
  echo "(", i, ", ", v, ")"
# (0, 100)
# (1, 200)
# (2, 300)
```

**Collecting index–value pairs into a sequence:**

```nim
import std/enumerate

let letters = ['x', 'y', 'z']
var indexed: seq[(int, char)]
for i, c in enumerate(letters):
  indexed.add((i, c))
assert indexed == @[(0, 'x'), (1, 'y'), (2, 'z')]
```

**Using enumerate with a custom iterator:**

```nim
import std/enumerate

iterator evens(n: int): int =
  var x = 0
  while x < n:
    yield x
    inc x, 2

for i, v in enumerate(evens(10)):
  echo i, ": ", v
# 0: 0
# 1: 2
# 2: 4
# 3: 6
# 4: 8
```

**Counter value is available after the loop:**

Because the counter variable is declared in the surrounding block, its final value is accessible after the loop ends — useful for knowing how many elements were actually visited:

```nim
import std/enumerate

let data = @[10, 20, 30, 40]
var count: int
for count, v in enumerate(data):
  discard
# After the loop, count == 3 (the index of the last element)
```

---

### Common mistakes and how to avoid them

**Mistake 1 — single loop variable (no value binding):**

```nim
# WRONG — compile-time error: "Missing second for loop variable"
for i in enumerate(mySeq):
  discard

# CORRECT
for i, x in enumerate(mySeq):
  discard
```

**Mistake 2 — confusing the argument order:**

The optional start value always comes **before** the collection, not after it.

```nim
# WRONG — tries to iterate the integer 1 with "hello" as start
for i, ch in enumerate("hello", 1):  # runtime or type error
  discard

# CORRECT
for i, ch in enumerate(1, "hello"):
  discard
```

**Mistake 3 — using `enumerate` outside a `for` loop:**

`enumerate` is a `ForLoopStmt` macro. It has no meaning outside of a `for` statement and cannot be stored in a variable or passed as an argument.

```nim
# WRONG — does not compile
let e = enumerate(mySeq)

# CORRECT — always use directly in a for statement
for i, x in enumerate(mySeq):
  discard
```

---

### Design notes

- **Zero overhead.** The transformation is purely syntactic. The generated code is identical to writing the counter loop by hand.
- **Works with any iterable.** Because the macro simply moves the collection expression into a plain `for` loop, every type that Nim's `for` statement supports — including custom iterators, `items`, `pairs`, ranges — works without any additional setup.
- **Block scoping.** The macro wraps the generated code in a `block` statement, so the counter variable does not leak into the surrounding scope. This is consistent with how `for` loop variables are scoped in Nim.
- **`ForLoopStmt` macro kind.** Unlike ordinary macros, a `ForLoopStmt` macro receives the entire `for` statement as its single argument. This is what allows `enumerate` to intercept the loop syntax transparently rather than requiring any special call syntax.
