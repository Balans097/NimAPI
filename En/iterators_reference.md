# iterators Module Reference

> **Module:** `iterators`  
> **Part of:** Nim's Runtime Library  
> **Purpose:** Provides the standard iteration protocol for all of Nim's built-in collection types. Every `for` loop over a string, sequence, array, set, tuple, or object relies on an iterator defined here. The module also establishes the naming conventions — `items`, `mitems`, `pairs`, `mpairs`, `fields`, `fieldPairs` — that the language uses as extension points for user-defined types.

---

## Core Concepts

### Naming conventions

The module follows four consistent naming patterns:

| Name | What it yields | Mutation? |
|---|---|---|
| `items` | each element value | no |
| `mitems` | each element by `var` reference | **yes** |
| `pairs` | `(index, element)` tuples | no |
| `mpairs` | `(index, element var)` tuples | **yes** |

`fields` and `fieldPairs` are the structural equivalents for tuples and objects, operating at compile time.

### `lent` return type

Most `items` iterators yield `lent T` (a read-only borrow) rather than a copy of `T`. This means the loop variable is an alias into the collection — no copy is made, which is especially valuable for large element types. The alias cannot be reassigned; use `mitems` when you need to mutate elements in place.

### Mutation safety on `seq` and `string`

Iterators over `seq[T]` and `string` capture the length at the start of the loop and assert on every iteration that the length has not changed. This catches accidental `add` or `delete` calls inside the loop body, which would invalidate the iterator.

### `unCheckedInc`

The index variable is incremented without overflow checking (`{.overflowChecks: off.}`). This is intentional: for a collection of length `high(int)`, the final increment would overflow, but the loop condition is checked *before* the yield, so the increment of the last element is never observed. This avoids a runtime penalty in tight loops.

---

## Table of Contents

1. [items — value iteration](#items--value-iteration)
2. [mitems — mutable iteration](#mitems--mutable-iteration)
3. [pairs — indexed iteration](#pairs--indexed-iteration)
4. [mpairs — mutable indexed iteration](#mpairs--mutable-indexed-iteration)
5. [fields — tuple/object field iteration](#fields--tupleobject-field-iteration)
6. [fieldPairs — named field iteration](#fieldpairs--named-field-iteration)

---

## items — value iteration

All `items` iterators support the implicit `for x in collection` syntax. The `in` keyword desugars to a call to `items`.

---

### `items` for `openArray[T]` (non-char)

```nim
iterator items*[T: not char](a: openArray[T]): lent T {.inline.}
```

**What it is:** The most general items iterator — works on any contiguous sequence: arrays, sequences, and slices passed as `openArray`. Yields each element as a `lent T` (read-only alias), avoiding unnecessary copies. The `not char` constraint exists because `char` elements have a separate overload (see below).

```nim
let nums = [10, 20, 30]
for n in nums:     # desugars to: for n in items(nums)
  echo n           # prints 10, 20, 30

proc sumAll(a: openArray[int]): int =
  for x in a: result += x

echo sumAll([1, 2, 3])    # 6
echo sumAll(@[4, 5, 6])   # 15 — same iterator, different container
```

---

### `items` for `openArray[char]`

```nim
iterator items*[T: char](a: openArray[T]): T {.inline.}
```

**What it is:** Identical to the non-char version but yields `char` by value rather than by `lent` reference. The reason is a VM-level bug: taking the address of an `openArray[char]` element when the array was converted from a string causes a failure in the compile-time evaluator. Since there is no performance benefit to returning a `char` by reference anyway (a `char` is one byte — smaller than a pointer), yielding by value is both correct and efficient.

```nim
let s = "hello"
for c in openArray[char](s):
  echo c   # prints h, e, l, l, o
```

---

### `items` for `array[IX, T]`

```nim
iterator items*[IX, T](a: array[IX, T]): T {.inline.}
```

**What it is:** Iterates over a fixed-size array. The index type `IX` can be any ordinal — an integer range, an enum, or a `char`. The loop silently produces no iterations if `a.len == 0` (checked at compile time via `when`), avoiding the awkward `low(IX) > high(IX)` edge case for empty arrays.

The loop structure (`while true` with a break on `i >= high(IX)`) is used instead of a simple `while i <= high(IX)` to avoid overflow when `IX` is a type whose `high` is `high(int)`.

```nim
let arr = [1, 2, 3]
for x in arr: echo x   # 1, 2, 3

# Enum-indexed array:
type Dir = enum North, South, East, West
let labels: array[Dir, string] = ["N", "S", "E", "W"]
for s in labels: echo s
```

---

### `items` for `set[T]`

```nim
iterator items*[T](a: set[T]): T {.inline.}
```

**What it is:** Iterates over the elements that are *actually present* in the set — not over all possible values that the set *could* hold. Internally scans the full range of `T` and yields only values where membership is confirmed. The iteration order is always ascending by ordinal value.

```nim
let s = {3, 1, 4, 1, 5}   # duplicates collapsed
for x in s: echo x         # prints 1, 3, 4, 5 (in order)

type Color = enum red, green, blue
let colors = {red, blue}
for c in colors: echo c    # red, blue
```

---

### `items` for `cstring`

```nim
iterator items*(a: cstring): char {.inline.}
```

**What it is:** Iterates over characters of a C string until the null terminator `'\0'`. On native backends the iteration stops at the first `'\0'`, so embedded nulls in the middle of a `cstring` are never reached. On JS and NimVM backends, the length is computed first (reflecting those platforms' string semantics).

> **Warning:** Passing `nil` as the argument causes a segfault (SIGSEGV). The implementation deliberately does not add a nil-check to avoid paying a cost on every iteration.

```nim
from std/sequtils import toSeq
assert toSeq("abc\0def".cstring) == @['a', 'b', 'c']
# The '\0' terminates iteration; "def" is never reached
```

---

### `items` for `seq[T]`

```nim
iterator items*[T](a: seq[T]): lent T {.inline.}
```

**What it is:** Yields each element of a sequence by `lent` reference. Captures the length before the loop and asserts that it has not changed on each iteration — this assertion catches bugs where the loop body modifies the sequence (appends or removes elements), which would make the loop unsound.

```nim
let fruits = @["apple", "banana", "cherry"]
for f in fruits:
  echo f   # apple, banana, cherry

# Unsafe pattern caught by the assertion:
var xs = @[1, 2, 3]
# for x in xs: xs.add(x)  # → assertion failure: length changed
```

---

### `items` for `string`

```nim
iterator items*(a: string): char {.inline.}
```

**What it is:** Yields each byte (character) of the string. Like the `seq` variant, it captures the length and asserts stability. Note that for multi-byte UTF-8 sequences, each `char` is one *byte*, not one Unicode code point. If you need Unicode-aware iteration, use `std/unicode.runes`.

```nim
for c in "hello":
  echo c   # h, e, l, l, o

var count = 0
for c in "Nim 🚀":
  if c != ' ': inc count
echo count   # counts bytes, not glyphs
```

---

### `items` for enum type descriptor

```nim
iterator items*[T: enum and Ordinal](E: typedesc[T]): T
```

**What it is:** Iterates over all values of an enum type. Unlike the set `items`, this works on the *type* itself (passed as `typedesc`), not on an instance. The iteration covers the full contiguous range from `low(E)` to `high(E)`. 

> **Important:** This only works correctly for enums *without holes* (where all ordinal values between `low` and `high` are declared). For enums with gaps, use `enumutils.items`.

```nim
type Goo = enum g0 = 2, g1, g2

from std/sequtils import toSeq
assert Goo.toSeq == [g0, g1, g2]

for v in Goo:
  echo v   # g0, g1, g2
```

---

### `items` for `Slice[T]`

```nim
iterator items*[T: Ordinal](s: Slice[T]): T
```

**What it is:** Iterates over all ordinal values within the slice `s`, from `s.a` to `s.b` inclusively. This makes slices directly usable in `for` loops without spelling out the range syntax manually.

```nim
for i in (3 .. 7):   # items is called on the Slice[int]
  echo i   # 3, 4, 5, 6, 7

for c in ('a' .. 'e'):
  echo c   # a, b, c, d, e
```

---

## mitems — mutable iteration

`mitems` yields each element as a `var T` — a mutable reference directly into the collection. Modifying the loop variable modifies the original element.

---

### `mitems` for `openArray[T]`

```nim
iterator mitems*[T](a: var openArray[T]): var T {.inline.}
```

**What it is:** Mutable element iteration over any contiguous collection. The container must be declared `var`. This is the idiomatic way to transform every element of an array or sequence in place without index arithmetic.

```nim
var nums = [1, 2, 3, 4, 5]
for n in mitems(nums):
  n *= 2
echo nums   # [2, 4, 6, 8, 10]

var words = @["hello", "world"]
for w in mitems(words):
  w = w.toUpperAscii()
echo words   # @["HELLO", "WORLD"]
```

---

### `mitems` for `array[IX, T]`

```nim
iterator mitems*[IX, T](a: var array[IX, T]): var T {.inline.}
```

**What it is:** Mutable iteration over a fixed-size array. The same overflow-safe loop structure as the read-only `items[IX,T]` version.

```nim
var data: array[3, float] = [1.0, 2.0, 3.0]
for x in mitems(data):
  x = x * x
echo data   # [1.0, 4.0, 9.0]
```

---

### `mitems` for `cstring`

```nim
iterator mitems*(a: var cstring): var char {.inline.}
```

**What it is:** Mutable character iteration over a C string. Allows in-place modification of individual characters. On native backends, stops at the first `'\0'`. The recommended pattern is to use `prepareMutation` before mutating a `cstring` that was originally a Nim `string`, to ensure the underlying buffer is not shared.

```nim
from std/sugar import collect
var a = "abc\0def"
prepareMutation(a)
var b = a.cstring
let s = collect:
  for bi in mitems(b):
    if bi == 'b': bi = 'B'
    bi
assert s == @['a', 'B', 'c']
assert b == "aBc"
assert a == "aBc\0def"  # the mutation propagated to the original string
```

---

### `mitems` for `seq[T]`

```nim
iterator mitems*[T](a: var seq[T]): var T {.inline.}
```

**What it is:** Mutable element iteration over a sequence, with the same length-stability assertion as the read-only `items` version. Do not add or remove elements during iteration — the assertion will fire.

```nim
var scores = @[85, 92, 78, 95]
for s in mitems(scores):
  if s < 80: s = 80   # floor all scores at 80
echo scores   # @[85, 92, 80, 95]
```

---

### `mitems` for `string`

```nim
iterator mitems*(a: var string): var char {.inline.}
```

**What it is:** Mutable byte iteration over a string. Lets you modify individual characters in place. Length-stability is asserted.

```nim
var s = "hello world"
for c in mitems(s):
  if c == ' ': c = '_'
echo s   # "hello_world"
```

---

## pairs — indexed iteration

`pairs` yields `(index, value)` tuples, giving you both the position and the element at the same time.

---

### `pairs` for `openArray[T]`

```nim
iterator pairs*[T](a: openArray[T]): tuple[key: int, val: T] {.inline.}
```

**What it is:** Yields `(index, element)` for each element in the collection. The index is always a plain `int`, regardless of the underlying collection's index type.

```nim
let fruits = ["apple", "banana", "cherry"]
for i, f in fruits:
  echo i, ": ", f
# 0: apple
# 1: banana
# 2: cherry
```

---

### `pairs` for `array[IX, T]`

```nim
iterator pairs*[IX, T](a: array[IX, T]): tuple[key: IX, val: T] {.inline.}
```

**What it is:** Like `pairs` for `openArray`, but the key type is the array's actual index type `IX` — which may be an enum, a character, or a custom integer range. This is important when the array is indexed by something other than a plain integer.

```nim
type Dir = enum North, South, East, West
let costs: array[Dir, int] = [10, 20, 15, 25]
for dir, cost in costs:
  echo dir, " costs ", cost
# North costs 10
# South costs 20
# ...
```

---

### `pairs` for `seq[T]`

```nim
iterator pairs*[T](a: seq[T]): tuple[key: int, val: T] {.inline.}
```

**What it is:** Indexed iteration over a sequence. Length is captured before the loop and asserted to be stable on each step.

```nim
let items = @["x", "y", "z"]
for idx, val in items:
  echo idx, " → ", val
```

---

### `pairs` for `string`

```nim
iterator pairs*(a: string): tuple[key: int, val: char] {.inline.}
```

**What it is:** Yields `(byte_index, char)` pairs for each byte in the string. Useful when you need both the position and the character — for example, building a replacement string while knowing where changes occur.

```nim
for i, c in "nim":
  echo i, ": ", c
# 0: n
# 1: i
# 2: m
```

---

### `pairs` for `cstring`

```nim
iterator pairs*(a: cstring): tuple[key: int, val: char] {.inline.}
```

**What it is:** Indexed iteration over a C string. On native backends, stops at `'\0'`. On JS, uses the computed string length. Behaviour with embedded nulls is the same as `items` for `cstring`.

---

## mpairs — mutable indexed iteration

`mpairs` combines index access with mutability: yields `(index, var element)` tuples. The `val` field of the yielded tuple is a mutable reference into the collection.

---

### `mpairs` for `openArray[T]`

```nim
iterator mpairs*[T](a: var openArray[T]): tuple[key: int, val: var T] {.inline.}
```

**What it is:** Indexed mutable iteration over any contiguous collection. The `key` gives you the position; the `val` gives you direct write access to the element. Useful when the mutation logic depends on both the index and the value.

```nim
var data = @[10, 20, 30, 40]
for i, v in mpairs(data):
  if i mod 2 == 0: v = 0   # zero out even-indexed elements
echo data   # @[0, 20, 0, 40]
```

---

### `mpairs` for `array[IX, T]`

```nim
iterator mpairs*[IX, T](a: var array[IX, T]): tuple[key: IX, val: var T] {.inline.}
```

**What it is:** Indexed mutable iteration over a fixed-size array. The key preserves the original index type `IX`.

```nim
var arr: array[3, int] = [5, 10, 15]
for i, v in mpairs(arr):
  v += i
echo arr   # [5, 11, 17]
```

---

### `mpairs` for `seq[T]`

```nim
iterator mpairs*[T](a: var seq[T]): tuple[key: int, val: var T] {.inline.}
```

**What it is:** Indexed mutable iteration with length-stability assertion. Do not resize the sequence inside the loop.

```nim
var words = @["apple", "banana", "cherry"]
for i, w in mpairs(words):
  w = $i & "_" & w
echo words   # @["0_apple", "1_banana", "2_cherry"]
```

---

### `mpairs` for `string`

```nim
iterator mpairs*(a: var string): tuple[key: int, val: var char] {.inline.}
```

**What it is:** Indexed mutable byte iteration over a string with length-stability assertion.

```nim
var s = "abcde"
for i, c in mpairs(s):
  if i mod 2 == 0: c = '-'
echo s   # "-b-d-"
```

---

### `mpairs` for `cstring`

```nim
iterator mpairs*(a: var cstring): tuple[key: int, val: var char] {.inline.}
```

**What it is:** Indexed mutable character iteration over a C string. Stops at `'\0'` on native backends.

---

## fields — tuple/object field iteration

### `fields` for a single tuple or object

```nim
iterator fields*[T: tuple|object](x: T): RootObj {.magic: "Fields", noSideEffect.}
```

**What it is:** Iterates over every field of a tuple or object, yielding each field's value. The loop variable has the type of each field in turn — this is a *compile-time unrolling*, not a runtime loop. The compiler transforms the `for` statement into separate code for each field, so the loop body can use the actual type of each field.

> **Warning:** Because the loop is unrolled at compile time, you cannot use a runtime `if` to branch on type — you must use `when` with the `is` operator. Also, there is a known bug affecting symbol binding in the loop body; prefer `fieldPairs` for complex cases.

```nim
var t = (1, "foo")
for v in fields(t):
  v = default(typeof(v))
assert t == (0, "")

# Clearing all fields of an object:
type Config = object
  debug: bool
  maxRetries: int
  host: string

var cfg = Config(debug: true, maxRetries: 3, host: "localhost")
for v in fields(cfg):
  v = default(typeof(v))
assert cfg == Config()
```

---

### `fields` for two tuples or objects

```nim
iterator fields*[S: tuple|object, T: tuple|object](x: S, y: T): tuple[key: string, val: RootObj] {.magic: "Fields", noSideEffect.}
```

**What it is:** Simultaneously iterates over corresponding fields of two tuples or objects. At each step, the loop yields a pair of values — one from `x` and one from `y` — allowing pairwise operations across structurally compatible types.

```nim
var t1 = (1, "foo")
var t2 = default(typeof(t1))
for v1, v2 in fields(t1, t2):
  v2 = v1          # copy each field from t1 into t2
assert t1 == t2
```

---

## fieldPairs — named field iteration

### `fieldPairs` for a single tuple or object

```nim
iterator fieldPairs*[T: tuple|object](x: T): tuple[key: string, val: RootObj] {.magic: "FieldPairs", noSideEffect.}
```

**What it is:** Like `fields`, but also yields the field's *name* as a string alongside its value. The `key` is the field name as it appears in the source code; `val` is the field value. Like `fields`, this is a compile-time unroll — use `when` instead of `if` for type-based branching.

This is the standard tool for implementing generic serialization, pretty-printing, and introspection of tuples and objects.

```nim
type Custom = object
  foo: string
  bar: bool

proc `$`(x: Custom): string =
  result = "Custom:"
  for name, value in x.fieldPairs:
    when value is bool:
      result.add("\n\t" & name & " is " & $value)
    else:
      result.add("\n\t" & name & " '" & value & "'")

echo Custom(foo: "hello", bar: true)
# Custom:
#   foo 'hello'
#   bar is true
```

---

### `fieldPairs` for two tuples or objects

```nim
iterator fieldPairs*[S: tuple|object, T: tuple|object](x: S, y: T): tuple[key: string, a, b: RootObj] {.magic: "FieldPairs", noSideEffect.}
```

**What it is:** Pairwise named field iteration over two structurally compatible types. Yields `(name, valueFromX, valueFromY)` at each step. Useful for diffing, merging, or selectively copying fields between two instances.

```nim
type Foo = object
  x1: int
  x2: string

var a1 = Foo(x1: 12, x2: "abc")
var a2: Foo
for name, v1, v2 in fieldPairs(a1, a2):
  when name == "x2":
    v2 = v1   # copy only the x2 field
assert a2 == Foo(x1: 0, x2: "abc")
```

---

## Quick Reference

| Iterator | Collection types | Yields | Mutable? |
|---|---|---|---|
| `items` | openArray, array, set, seq, string, cstring, typedesc, Slice | element value | no |
| `mitems` | openArray, array, seq, string, cstring | `var` element | **yes** |
| `pairs` | openArray, array, seq, string, cstring | `(index, value)` | no |
| `mpairs` | openArray, array, seq, string, cstring | `(index, var value)` | **yes** |
| `fields` | tuple, object (×1 or ×2) | field value(s) | depends |
| `fieldPairs` | tuple, object (×1 or ×2) | `(name, value)` or `(name, v1, v2)` | depends |

---

*This reference covers all exported iterators from `iterators.nim` in Nim's standard library.*
