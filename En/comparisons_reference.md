# comparisons Module Reference

> **Module:** `comparisons`  
> **Part of:** Nim's Runtime Library  
> **Purpose:** Defines all comparison operators and related utilities for Nim's built-in types. This is the single source of truth for `==`, `<`, `<=`, `!=`, `>`, `>=` across integers, floats, strings, characters, booleans, enums, sets, pointers, references, procedures, arrays, sequences, and open arrays — as well as `min`, `max`, and `clamp`.

---

## Core Concepts

### Operator families

The module exposes six comparison operators. Four are *primitive* (backed by compiler magic or direct implementation), and two are *derived* templates:

| Operator | Derived from | Notes |
|---|---|---|
| `==` | primitive | one overload per type |
| `<` | primitive | one overload per type |
| `<=` | primitive | one overload per type |
| `!=` | `not (x == y)` | universal template |
| `>` | `y < x` | universal template |
| `>=` | `y <= x` | universal template |

Because `!=`, `>`, and `>=` are templates that delegate to `==`, `<`, and `<=`, **defining `==` and `<` for a custom type automatically gives you all six operators for free**.

### Unsigned reinterpretation operators (`<%`, `<=%`, `>%`, `>=%`)

A separate family of operators treats *signed* integer values as if they were *unsigned*. They carry a `%` suffix to make the reinterpretation explicit at the call site. These are crucial for low-level bit manipulation, pointer arithmetic emulation, and protocol parsing where the sign bit is data.

---

## Table of Contents

1. [Equality: `==`](#equality-)
2. [Less-than: `<`](#less-than-)
3. [Less-than-or-equal: `<=`](#less-than-or-equal-)
4. [Derived operators: `!=`, `>`, `>=`](#derived-operators---)
5. [Unsigned reinterpretation: `<%`, `<=%`, `>%`, `>=%`](#unsigned-reinterpretation-----)
6. [Minimum and Maximum: `min`, `max`](#minimum-and-maximum-min-max)
7. [Clamping: `clamp`](#clamping-clamp)
8. [Compile-time safety guards](#compile-time-safety-guards)

---

## Equality: `==`

### Integers and unsigned integers

```nim
proc `==`*(x, y: int): bool
proc `==`*(x, y: int8): bool
proc `==`*(x, y: int16): bool
proc `==`*(x, y: int32): bool
proc `==`*(x, y: int64): bool

proc `==`*(x, y: uint): bool
proc `==`*(x, y: uint8): bool
proc `==`*(x, y: uint16): bool
proc `==`*(x, y: uint32): bool
proc `==`*(x, y: uint64): bool
```

**What it is:** Bitwise value equality for all integer widths — signed and unsigned. Each overload is handled by the compiler's `"EqI"` magic, which maps directly to the target CPU's equality instruction with no overhead.

```nim
assert 42 == 42
assert 0'u8 == 0'u8
assert not (1'i32 == 2'i32)
```

---

### Floats

```nim
proc `==`*(x, y: float32): bool
proc `==`*(x, y: float): bool   # float64
```

**What it is:** IEEE 754 floating-point equality. Note the standard IEEE caveat: `NaN != NaN` is always `true` — a `NaN` value is **not equal to itself**. If you need to test for `NaN`, use `math.isNaN`.

```nim
assert 1.0 == 1.0
assert not (0.1 + 0.2 == 0.3)   # classic floating-point imprecision
let nan = 0.0 / 0.0
assert nan != nan                # NaN is never equal to itself
```

---

### Strings

```nim
proc `==`*(x, y: string): bool
```

**What it is:** Lexicographic byte-for-byte equality of two strings. Two strings are equal if and only if they have the same length and every corresponding byte is identical. This is case-sensitive.

```nim
assert "hello" == "hello"
assert not ("Hello" == "hello")   # different case → not equal
assert "" == ""
```

---

### Characters

```nim
proc `==`*(x, y: char): bool
```

**What it is:** Compares the numeric (ASCII / Unicode ordinal) values of two characters. Equivalent to comparing `ord(x) == ord(y)`.

```nim
assert 'a' == 'a'
assert not ('A' == 'a')   # 65 ≠ 97
```

---

### Booleans

```nim
proc `==`*(x, y: bool): bool
```

**What it is:** Standard boolean equality. `true == true` and `false == false`; mixed values are not equal.

```nim
assert true == true
assert false == false
assert not (true == false)
```

---

### Enums

```nim
proc `==`*[Enum: enum](x, y: Enum): bool
```

**What it is:** Compares two values of the *same* enum type by their underlying integer value. Crucially, this operator is **type-safe**: comparing values from two different enum types is a compile-time error, even if their ordinal values happen to be identical.

```nim
type
  Color = enum red = 0, green = 1
  Status = enum ok = 0, fail = 1

assert red == red
assert not (red == green)
# assert red == ok  # ← compile error: cannot compare different enum types
```

The following example from the source shows that two enum values *from different types* with the same ordinal can be made equal only by explicit conversion:

```nim
type
  Enum1 = enum field1 = 3, field2
  Enum2 = enum place1, place2 = 3

var e1 = field1
var e2 = place2.ord.Enum1   # explicit re-interpretation
assert e1 == e2             # same underlying value (3)
assert not compiles(e1 == place2)  # direct cross-type compare is forbidden
```

---

### Sets

```nim
proc `==`*[T](x, y: set[T]): bool
```

**What it is:** Two sets are equal if they contain exactly the same members. Duplicate elements in a set literal are irrelevant — a set holds each value at most once by definition.

```nim
assert {1, 2, 3} == {3, 1, 2}     # order doesn't matter
assert {1, 2, 2, 3} == {1, 2, 3}  # duplicates are ignored
assert not ({1, 2} == {1, 2, 3})
```

---

### References (`ref T`)

```nim
proc `==`*[T](x, y: ref T): bool
```

**What it is:** Tests whether two `ref` variables point to the **same heap object** — i.e., identity comparison, not structural equality. Two different objects with identical contents are **not** equal under this operator.

```nim
type Node = ref object
  val: int

let a = Node(val: 5)
let b = Node(val: 5)
let c = a

assert a == c   # same object
assert not (a == b)  # different objects, same content
```

> If you need structural (deep) equality for `ref` types, you must define your own `==` overload.

---

### Raw pointers (`ptr T`, `pointer`)

```nim
proc `==`*[T](x, y: ptr T): bool
proc `==`*(x, y: pointer): bool
```

**What it is:** Numeric address equality — two pointers are equal if and only if they hold the same memory address. `nil` and a pointer cast from `0` are considered equal.

```nim
var x = 42
let p1 = addr x
let p2 = addr x
assert p1 == p2   # same address

let null1 = cast[pointer](0)
let null2 = cast[pointer](nil)
assert null1 == null2
```

---

### Procedures and iterators

```nim
proc `==`*[T: proc | iterator](x, y: T): bool
```

**What it is:** Compares two procedure *variables* (closures or function pointers) for identity — they are equal if they refer to the same underlying procedure. Does not compare closed-over state; two closures created from the same `proc` are generally not equal.

---

### Arrays

```nim
proc `==`*[I, T](x, y: array[I, T]): bool
```

**What it is:** Element-wise equality for fixed-size arrays. Returns `true` only if every corresponding pair of elements compares equal using the element type's own `==`. Both arrays must have the same index type `I` and element type `T`.

```nim
assert [1, 2, 3] == [1, 2, 3]
assert not ([1, 2, 3] == [1, 2, 4])
```

---

### Open arrays

```nim
proc `==`*[T](x, y: openArray[T]): bool
```

**What it is:** Element-wise equality for any contiguous sequence view — works on arrays, sequences, and slices uniformly. First checks lengths, then compares element by element.

```nim
proc check(a, b: openArray[int]): bool = a == b

assert check([1, 2, 3], @[1, 2, 3])    # array vs seq: equal
assert not check([1, 2], [1, 2, 3])    # different lengths
```

---

### Sequences

```nim
proc `==`*[T](x, y: seq[T]): bool
```

**What it is:** Element-wise equality for dynamically-sized sequences. Includes an important fast path: if both sequences share the same underlying memory pointer (i.e., one was assigned from the other without modification), `true` is returned immediately without iterating.

```nim
assert @[1, 2, 3] == @[1, 2, 3]
assert not (@[1, 2] == @[1, 2, 3])

var a = @[1, 2, 3]
var b = a           # b shares pointer with a (shallow copy)
assert a == b       # hits the fast-path pointer check
```

---

## Less-than: `<`

### Integers (signed and unsigned)

```nim
proc `<`*(x, y: int): bool      # also int8, int16, int32, int64
proc `<`*(x, y: uint): bool     # also uint8, uint16, uint32, uint64
```

**What it is:** Numeric less-than. Signed variants use signed comparison; unsigned variants use unsigned comparison. Mixing signed and unsigned requires an explicit cast.

```nim
assert 1 < 2
assert not (2 < 2)
assert 0'u8 < 255'u8
```

---

### Floats

```nim
proc `<`*(x, y: float32): bool
proc `<`*(x, y: float): bool
```

**What it is:** IEEE 754 less-than. Any comparison involving `NaN` returns `false`.

```nim
assert 1.0 < 2.0
let nan = 0.0 / 0.0
assert not (nan < 1.0)   # NaN comparisons are always false
assert not (1.0 < nan)
```

---

### Strings

```nim
proc `<`*(x, y: string): bool
```

**What it is:** Lexicographic byte-by-byte less-than. Comparison proceeds character by character; the first differing byte determines the result. If all characters match up to the shorter string's length, the shorter string is considered smaller. Uppercase ASCII letters (65–90) have lower ordinal values than lowercase (97–122), so `"Z" < "a"` is `true`.

```nim
assert "abc" < "abd"
assert "abc" < "abcd"   # "abc" is a prefix of "abcd" and shorter
assert "Z" < "a"        # uppercase < lowercase in ASCII
assert not ("abc" < "abc")
```

---

### Characters

```nim
proc `<`*(x, y: char): bool
```

**What it is:** Ordinal (ASCII/Unicode code point) less-than for characters. The same ASCII ordering applies: digits < uppercase < lowercase.

```nim
assert 'a' < 'b'
assert 'A' < 'a'
assert not ('z' < 'a')
```

---

### Enums

```nim
proc `<`*[Enum: enum](x, y: Enum): bool
```

**What it is:** Compares two enum values by their underlying integer ordinal. For a standard enum without explicit values, the declaration order determines the ordinal: the first variant is 0, the next is 1, and so on.

```nim
type Priority = enum low, medium, high

assert low < high
assert medium < high
assert not (high < low)
```

---

### Sets

```nim
proc `<`*[T](x, y: set[T]): bool
```

**What it is:** **Strict (proper) subset** test. Returns `true` if every member of `x` is also in `y`, and `y` has at least one member that `x` does not. A set is **not** a strict subset of itself.

```nim
assert {1, 2} < {1, 2, 3}    # {1,2} ⊂ {1,2,3}
assert not ({1, 2} < {1, 2}) # a set is not a proper subset of itself
assert not ({4} < {1, 2, 3}) # 4 is not in the right set
```

---

### Pointers and refs

```nim
proc `<`*[T](x, y: ref T): bool
proc `<`*[T](x, y: ptr T): bool
proc `<`*(x, y: pointer): bool
```

**What it is:** Numeric address comparison — `x` has a lower memory address than `y`. Primarily useful for pointer arithmetic, custom allocators, and building data structures over raw memory blocks.

---

## Less-than-or-equal: `<=`

### Integers (signed and unsigned)

```nim
proc `<=`*(x, y: int): bool     # also int8, int16, int32, int64
proc `<=`*(x, y: uint): bool    # also uint8, uint16, uint32, uint64
```

**What it is:** Numeric less-than-or-equal for all integer widths.

```nim
assert 2 <= 2
assert 1 <= 2
assert not (3 <= 2)
```

---

### Floats

```nim
proc `<=`*(x, y: float32): bool
proc `<=`*(x, y: float): bool
```

**What it is:** IEEE 754 less-than-or-equal. Comparisons involving `NaN` return `false`.

---

### Strings

```nim
proc `<=`*(x, y: string): bool
```

**What it is:** Lexicographic less-than-or-equal. Returns `true` if `x < y` or `x == y`.

```nim
assert "abc" <= "abc"
assert "abc" <= "abd"
assert not ("abd" <= "abc")
```

---

### Characters

```nim
proc `<=`*(x, y: char): bool
```

**What it is:** Ordinal less-than-or-equal for characters.

```nim
assert 'a' <= 'a'
assert 'a' <= 'b'
assert not ('b' <= 'a')
```

---

### Enums

```nim
proc `<=`*[Enum: enum](x, y: Enum): bool
```

**What it is:** Ordinal less-than-or-equal for enum values. An enum value is `<=` itself.

---

### Sets

```nim
proc `<=`*[T](x, y: set[T]): bool
```

**What it is:** **Subset-or-equal** test. Returns `true` if every member of `x` is in `y`. Unlike `<`, this returns `true` when `x == y`.

```nim
let a = {3, 5}
let b = {1, 3, 5, 7}

assert a <= b    # {3,5} ⊆ {1,3,5,7}
assert a <= a    # a set is a subset of itself
assert not ({2} <= a)
```

---

### Booleans, references, and pointers

```nim
proc `<=`*(x, y: bool): bool
proc `<=`*[T](x, y: ref T): bool
proc `<=`*(x, y: pointer): bool
```

For `bool`: `false <= true`, `false <= false`, `true <= true`. For refs and pointers: numeric address order.

---

## Derived operators: `!=`, `>`, `>=`

These three operators are universal templates that work for any type that has `==` and `<` / `<=` defined. They carry the `{.callsite.}` pragma, which preserves source-location information for error messages.

### `!=`

```nim
template `!=`*(x, y: untyped): untyped
```

**What it is:** Shorthand for `not (x == y)`. Because it is a template, the compiler expands it at the call site — there is no function-call overhead.

```nim
assert 1 != 2
assert "hello" != "world"
assert not (42 != 42)
```

---

### `>`

```nim
template `>`*(x, y: untyped): untyped
```

**What it is:** Expands to `y < x`. Defining `<` for a custom type automatically gives you `>`.

```nim
assert 3 > 2
assert "b" > "a"
assert not (2 > 3)
```

---

### `>=`

```nim
template `>=`*(x, y: untyped): untyped
```

**What it is:** Expands to `y <= x`. Defining `<=` for a custom type automatically gives you `>=`.

```nim
assert 3 >= 3
assert 3 >= 2
assert not (2 >= 3)
```

---

## Unsigned reinterpretation: `<%`, `<=%`, `>%`, `>=%`

This operator family is designed for situations where you hold *signed* integers but need to compare them as if they were *unsigned*. This is common in low-level code: hash table probing, bitfield manipulation, memory range checks, and protocol parsers.

The `%` suffix is a mnemonic: "modular" or "raw bit pattern" semantics, the same convention used elsewhere in Nim (`+%`, `*%`, etc.).

### `<%`

```nim
proc `<%`*(x, y: int): bool    # also int8, int16, int32, int64
```

**What it is:** Reinterprets both `x` and `y` as their unsigned counterparts and performs unsigned less-than. The bit pattern is unchanged; only the interpretation differs.

```nim
assert 1 <% 2      # same as unsigned: 1 < 2
assert -1 >% 0     # -1 as uint is MaxUInt, which is > 0
assert not (0 <% 0)
```

The classic use case: checking whether a signed integer falls within a `[0, n)` range using a single unsigned comparison instead of two signed ones:

```nim
proc inRange(i, n: int): bool = i <% n
# Equivalent to: i >= 0 and i < n
# -1 <% 10 → false (because MaxUInt is not < 10)
#  5 <% 10 → true
# 15 <% 10 → false
```

---

### `<=%`

```nim
proc `<=%`*(x, y: int): bool   # also int8, int16, int32, int64
```

**What it is:** Unsigned less-than-or-equal via reinterpretation.

```nim
assert 0 <=% 0
assert 0 <=% 5
assert not (-1 <=% 0)   # -1 as uint > 0
```

---

### `>%` and `>=%`

```nim
template `>%`*(x, y: untyped): untyped    # expands to y <% x
template `>=%`*(x, y: untyped): untyped   # expands to y <=% x
```

**What they are:** The "greater than" and "greater-than-or-equal" counterparts for unsigned reinterpretation. Both are templates deriving from `<%` and `<=%`.

```nim
assert -1 >% 0    # MaxUInt > 0
assert 10 >=% 5
```

---

## Minimum and Maximum: `min`, `max`

### `min` and `max` for two values

```nim
# Integers (specialized for each width):
proc min*(x, y: int): int
proc min*(x, y: int8): int8
proc min*(x, y: int16): int16
proc min*(x, y: int32): int32
proc min*(x, y: int64): int64

proc max*(x, y: int): int
# ... same for int8/16/32/64

# Floats (with NaN propagation):
proc min*(x, y: float32): float32
proc min*(x, y: float64): float64
proc max*(x, y: float32): float32
proc max*(x, y: float64): float64

# Generic fallback for any non-float type with <=:
proc min*[T: not SomeFloat](x, y: T): T
proc max*[T: not SomeFloat](x, y: T): T
```

**What it is:** Returns the smaller (or larger) of two values. Integer variants are backed by compiler magic for maximum efficiency. The float variants handle the special case of `NaN`: if `y` is `NaN`, `x` is returned — this is a deliberate propagation choice consistent with many numeric libraries (if either operand is `NaN`, the result is the non-`NaN` value; if both are `NaN`, the first operand is returned).

```nim
assert min(3, 7) == 3
assert max(3, 7) == 7
assert min(1.5, 2.5) == 1.5

let nan = 0.0 / 0.0
assert min(1.0, nan) == 1.0   # NaN is silently discarded
assert min(nan, 1.0) == nan   # order matters when x is NaN
```

> **Float NaN caveat:** `min(nan, 1.0)` returns `nan` because the check is `if x <= y or y != y: x else: y` — when `y` is NaN, we return `x` (the NaN); when `x` is NaN, `x <= y` is false and `y != y` is false, so `y` (the non-NaN) is returned. The behaviour is not symmetric — be aware of this in numerical code.

---

### `min` and `max` over a collection

```nim
proc min*[T](x: openArray[T]): T
proc max*[T](x: openArray[T]): T
```

**What it is:** Scans the entire collection and returns the smallest (or largest) element. Requires the element type to have a `<` operator. Works on arrays, sequences, and any other type that satisfies `openArray[T]`.

```nim
assert min([5, 2, 8, 1, 9]) == 1
assert max([5, 2, 8, 1, 9]) == 9
assert min(@["banana", "apple", "cherry"]) == "apple"
```

> **Warning:** Calling `min` or `max` on an empty collection causes an index-out-of-bounds error at runtime, since the implementation unconditionally reads `x[0]`.

---

## Clamping: `clamp`

```nim
proc clamp*[T](x, a, b: T): T
```

**What it is:** Restricts the value `x` to the closed interval `[a, b]`. If `x < a`, returns `a`. If `x > b`, returns `b`. Otherwise returns `x` unchanged. Works for any type that has `<` and `>` defined.

> **Assumption:** `a <= b` is not checked by the implementation. Passing `a > b` produces undefined behaviour.

This is semantically equivalent to `max(a, min(b, x))` but implemented with two direct comparisons rather than two function calls, making it slightly faster.

```nim
assert clamp(0.5, 0.0, 1.0) == 0.5   # within range, unchanged
assert clamp(1.4, 0.0, 1.0) == 1.0   # above max, clamped to 1.0
assert clamp(-0.3, 0.0, 1.0) == 0.0  # below min, clamped to 0.0
assert clamp(4, 1, 3) == 3
assert clamp(0, 1, 3) == 1

# Practical use: normalizing user input
let volume = clamp(userInput, 0, 100)
```

---

## Compile-time safety guards

### `isNil` for strings (error proc)

```nim
proc isNil*(x: string): bool {.error: "'isNil' is invalid for 'string'".}
```

**What it is:** A deliberate compile-time error that fires if you accidentally call `isNil` on a `string`. In Nim's current memory model, strings are value types and can never be `nil` — calling `isNil(s)` is always a programming mistake. This overload exists solely to produce a helpful error message instead of a confusing type mismatch.

```nim
var s = "hello"
# if isNil(s): ...  ← compile error: 'isNil' is invalid for 'string'
```

---

### `==` nil comparisons for strings (error procs)

```nim
proc `==`*(x: string; y: typeof(nil) | typeof(nil)): bool {.error: "'nil' is invalid for 'string'".}
proc `==`*(x: typeof(nil) | typeof(nil); y: string): bool {.error: "'nil' is invalid for 'string'".}
```

**What they are:** Guard overloads that prevent comparing a `string` directly with `nil`. In Nim, strings are never `nil`; this comparison is always a logic error. These procs exist to catch the mistake at compile time (rather than silently returning `false` due to an unexpected implicit conversion).

```nim
var s = "hello"
# if s == nil: ...   ← compile error: 'nil' is invalid for 'string'
# if nil == s: ...   ← compile error: 'nil' is invalid for 'string'
```

---

*This reference covers all exported symbols from `comparisons.nim` in Nim's standard library.*
