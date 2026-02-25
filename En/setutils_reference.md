# std/setutils — Module Reference

> **Part of Nim's standard library** (`std/setutils`).

## Overview

Nim has a built-in `set` type that represents a mathematical set of small ordinal values (characters, bytes, booleans, small integers, enumerations). It is extraordinarily fast — implemented as a compact bitset — but the built-in operators cover only the basics: union (`+`), intersection (`*`), difference (`-`), membership (`in`/`notin`), inclusion (`incl`), exclusion (`excl`), and cardinality (`card`).

The `setutils` module adds the operations that are conspicuously absent from that baseline:

- Converting any iterable to a set (`toSet`)
- Building the complete set of all values of a type (`fullSet`)
- Computing the set complement (`complement`)
- Conditional include/exclude via subscript syntax (`[]=`)
- Symmetric difference — elements in either set but not both — both as a function (`symmetricDifference`) and as an operator (`-+-`)
- In-place symmetric difference (`toggle`)

All operations work on the same built-in `set[T]` type; no wrapper types or imports of alternative set libraries are needed.

---

## Allowed element types

A built-in Nim `set` can only hold elements of types with a small, finite ordinal range: `char`, `byte`, `bool`, `int8`, `uint8`, `int16`, `uint16`, and any `enum`. This constraint is a property of the built-in `set` type itself, not of this module.

---

## API Reference

---

### `toSet`

```nim
template toSet*(iter: untyped): untyped
```

**Converts any iterable into a built-in set containing exactly the elements produced by the iterable.**

This is the idiomatic way to build a set from a sequence, array, string, range, or any other value you can iterate over. The element type of the resulting set is inferred automatically from the iterable's element type using `elementType`. Duplicate values in the source are silently collapsed — sets contain each value at most once by definition.

Because `toSet` is a template, it is inlined at the call site and carries no overhead beyond the loop itself.

**Examples:**

```nim
import std/setutils

# From a string — each character becomes a set element:
assert "helloWorld".toSet == {'W', 'd', 'e', 'h', 'l', 'o', 'r'}
# Note: duplicate 'l' and 'o' appear only once in the result.

# From an array of uint16:
assert toSet([10u16, 20, 30]) == {10u16, 20, 30}

# From a seq of int16:
assert toSet(@[1321i16, 321, 90]) == {90i16, 321, 1321}

# From a range:
assert toSet(0u8..10) == {0u8..10}

# From a single-element array:
assert toSet([false]) == {false}
```

**Typical use — collecting unique characters from input:**

```nim
import std/setutils

proc uniqueLetters(s: string): set[char] =
  s.toSet

let letters = uniqueLetters("abracadabra")
echo letters   # {'a', 'b', 'c', 'd', 'r'}
```

---

### `fullSet`

```nim
func fullSet*[T](U: typedesc[T]): set[T] {.inline.}
```

**Returns a set that contains every possible value of the type `T`.**

This is useful whenever you need to start with a "universe" of all values and then subtract from it — for example when computing complements, or checking whether a set of flags is exhaustive.

For ordinary ordinal types (`bool`, integer subtypes, `range`s), the full set is constructed as `{T.low..T.high}`. For enumerations with holes (where the enum values are not consecutive integers), `fullSet` uses a macro to enumerate the actual declared values.

**Examples:**

```nim
import std/setutils

# All boolean values:
assert bool.fullSet == {true, false}

# A range type — only the values in range are included:
type A = range[1..3]
assert A.fullSet == {1.A, 2, 3}

# int8 covers -128..127, so exactly 256 values:
assert int8.fullSet.len == 256

# An enum with holes — only declared members:
type Colors = enum
  red, green = 3, blue
assert Colors.fullSet == {red, green, blue}
```

**Typical use — starting with everything and removing what you don't want:**

```nim
import std/setutils

type Permission = enum
  read, write, execute

# Grant all permissions, then revoke one:
var granted = Permission.fullSet
granted.excl(execute)
assert granted == {read, write}
```

---

### `complement`

```nim
func complement*[T](s: set[T]): set[T] {.inline.}
```

**Returns the set complement of `s` — all values of type `T` that are *not* in `s`.**

Mathematically, if `U` is the universal set of all values of `T`, then `complement(s) == U - s`. Internally it is simply `fullSet(T) - s`.

This operation answers the question: "which values of this type are *absent* from this set?" It is particularly readable when working with flag enumerations or character classes — for example, "all characters that are *not* digits."

**Examples:**

```nim
import std/setutils

type Colors = enum
  red, green = 3, blue

# Everything except red and blue:
assert complement({red, blue}) == {green}

# The complement of a full set is empty:
assert complement({red, green, blue}).card == 0

# All chars that are not ASCII digits:
assert complement({'0'..'9'}) == {0.char..255.char} - {'0'..'9'}

# Range types — complement stays within the range:
assert complement({range[0..10](0), 1, 2, 3}) == {range[0..10](4), 5, 6, 7, 8, 9, 10}
```

**Typical use — inverted character filter:**

```nim
import std/setutils

const nonDigits = complement({'0'..'9'})

proc stripDigits(s: string): string =
  for ch in s:
    if ch in nonDigits:
      result.add(ch)

echo stripDigits("a1b2c3")  # "abc"
```

---

### `[]=`

```nim
func `[]=`*[T](t: var set[T], key: T, val: bool) {.inline.}
```

**Conditionally includes or excludes a single element from a set, depending on a boolean value.**

This operator is shorthand for the common pattern `if val: t.incl(key) else: t.excl(key)`. It lets you set membership of an element using the familiar subscript-assignment syntax, treating the set like a mapping from elements to booleans. This is especially clean when element membership is driven by runtime conditions or configuration flags.

**Examples:**

```nim
import std/setutils

type A = enum
  a0, a1, a2, a3

var s = {a0, a3}

# false → exclude:
s[a0] = false
s[a1] = false   # a1 was not in s; excluding a non-member is a no-op
assert s == {a3}

# true → include:
s[a2] = true
s[a3] = true    # a3 is already in s; including an existing member is a no-op
assert s == {a2, a3}
```

**Typical use — building a permission set from individual flags:**

```nim
import std/setutils

type Flag = enum
  flagA, flagB, flagC

var flags: set[Flag]
let config = (a: true, b: false, c: true)

flags[flagA] = config.a
flags[flagB] = config.b
flags[flagC] = config.c

assert flags == {flagA, flagC}
```

---

### `symmetricDifference`

```nim
func symmetricDifference*[T](x, y: set[T]): set[T]
```

**Returns the symmetric difference of two sets — the set of elements that belong to exactly one of `x` or `y`, but not both.**

Mathematically: `symmetricDifference(x, y) == (x - y) + (y - x)` — elements unique to `x` plus elements unique to `y`. Elements present in both sets cancel out.

On compilers that define `nimHasXorSet` (Nim 2.x with built-in XOR support) this is a single `magic` instruction operating directly on the underlying bitsets, making it as fast as any other set operation. On older compilers it falls back to `x + y - (x * y)`, which is equivalent but involves three operations.

**Examples:**

```nim
import std/setutils

# Elements in {1,2,3} or {2,3,4} but not in both:
assert symmetricDifference({1, 2, 3}, {2, 3, 4}) == {1, 4}
# 2 and 3 are in both → excluded; 1 is only in x, 4 only in y → included.

# Symmetric difference with an empty set is the set itself:
assert symmetricDifference({1, 2}, {}) == {1, 2}

# Symmetric difference of identical sets is empty:
assert symmetricDifference({1, 2}, {1, 2}) == {}
```

**Typical use — finding what changed between two states:**

```nim
import std/setutils

type Option = enum
  optA, optB, optC, optD

let before = {optA, optB, optC}
let after  = {optB, optC, optD}

let changed = symmetricDifference(before, after)
# changed == {optA, optD}
# optA was removed; optD was added.
```

---

### `-+-`

```nim
proc `-+-`*[T](x, y: set[T]): set[T] {.inline.}
```

**Operator alias for `symmetricDifference(x, y)`.**

Both forms produce identical results; choose whichever reads more naturally in context. The operator form works well in chained expressions, while the function form is often clearer in documentation or when the name itself provides the explanation.

**Examples:**

```nim
import std/setutils

assert {1, 2, 3} -+- {2, 3, 4} == {1, 4}

# Chained: elements unique to exactly one of three sets (applied left-to-right):
let r = {1, 2} -+- {2, 3} -+- {3, 4}
assert r == {1, 4}
```

---

### `toggle`

```nim
proc toggle*[T](x: var set[T], y: set[T]) {.inline.}
```

**Toggles the membership of each element of `y` in `x`: elements already in `x` are removed; elements absent from `x` are added.**

This is the in-place mutation equivalent of `symmetricDifference`. It modifies `x` directly rather than returning a new set, which avoids an assignment when you just want to update a variable. Calling `x.toggle(y)` is exactly equivalent to `x = symmetricDifference(x, y)`.

The name "toggle" is deliberate: each element in `y` flips its presence in `x`, just as a toggle switch flips between on and off.

**Examples:**

```nim
import std/setutils

var x = {1, 2, 3}

# Toggle {2, 3, 4}: 2 and 3 leave (were in x), 4 enters (was not in x):
x.toggle({2, 3, 4})
assert x == {1, 4}

# Toggle the same set again — everything flips back:
x.toggle({2, 3, 4})
assert x == {1, 2, 3}
```

**Typical use — cycling through selection states:**

```nim
import std/setutils

type Item = enum
  itemA, itemB, itemC, itemD

var selected: set[Item] = {itemA, itemB}

proc toggleSelection(toToggle: set[Item]) =
  selected.toggle(toToggle)

toggleSelection({itemB, itemC})
assert selected == {itemA, itemC}   # itemB deselected, itemC selected

toggleSelection({itemC, itemD})
assert selected == {itemA, itemD}   # itemC deselected, itemD selected
```

---

## Summary Table

| Symbol | Signature | Purpose |
|--------|-----------|---------|
| `toSet(iter)` | `untyped → set[T]` | Build a set from any iterable |
| `fullSet(T)` | `typedesc[T] → set[T]` | Set of all values of a type |
| `complement(s)` | `set[T] → set[T]` | All values of `T` not in `s` |
| `t[key] = val` | `(var set[T], T, bool) → void` | Conditionally include or exclude a single element |
| `symmetricDifference(x, y)` | `(set[T], set[T]) → set[T]` | Elements in exactly one of the two sets |
| `x -+- y` | `(set[T], set[T]) → set[T]` | Operator alias for `symmetricDifference` |
| `toggle(x, y)` | `(var set[T], set[T]) → void` | In-place symmetric difference |

---

## Relationship between the symmetric-difference operations

The three symmetric-difference symbols cover the same concept at different levels of convenience:

```
symmetricDifference(x, y)   ← named function; most explicit
x -+- y                     ← infix operator; good in expressions
x.toggle(y)                 ← in-place mutation; good when updating a variable
```

All three produce the same logical result; the choice is purely stylistic.

---

> **See also:** [`std/sets`](https://nim-lang.org/docs/sets.html) for hash-based sets with no element-type restrictions; [`std/packedsets`](https://nim-lang.org/docs/packedsets.html) for memory-efficient sets of integers with a larger range than the built-in type supports.
