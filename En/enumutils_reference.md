# enumutils — Module Reference

## Overview

The `enumutils` module extends Nim's built-in enum support with utilities for working with **holey enums** (enums whose integer values are not contiguous), **symbol introspection** (querying a field's position and source-code name independently of its ordinal value), and **string-to-enum parsing infrastructure** (a macro for generating efficient `case` statements driven by string matching).

Understanding two Nim type-trait concepts is essential before reading the individual entries:

- **`OrdinalEnum`** — an enum whose values form a contiguous integer range starting from its lowest member. The compiler can use the ordinal value directly as an array index. Example: `enum { a, b, c }` or `enum { a = 10, b, c }`.
- **`HoleyEnum`** — an enum that has gaps in its integer values. The values cannot be used directly as array indices without mapping. Example: `enum { a = 2, b = 4, c = 8 }`.

Several procedures in this module work exclusively with `HoleyEnum` because `OrdinalEnum` already has efficient built-in support through its ordinal value.

---

## `items` (iterator)

```nim
iterator items*[T: HoleyEnum](E: typedesc[T]): T
```

### Description

Iterates over every declared member of a holey enum type `E`, yielding them in declaration order. This fills the gap left by Nim's built-in `items` iterator, which only works on `OrdinalEnum` types. For a holey enum, there is no safe way to walk the values by simply incrementing an integer counter — doing so would pass through gaps and produce values that do not correspond to any declared field. This iterator solves the problem by using the compiler's complete knowledge of the enum's members, generated at compile time through the `enumFullRange` macro internally.

The iterator is called on the **type**, not on a value: `for x in MyEnum: …` — this mirrors the syntax used for ordinal enums.

### Parameters

| Name | Type | Description |
|---|---|---|
| `E` | `typedesc[T]` | The holey enum type to iterate over |

### Yields

Each declared member of `T`, in declaration order, as a value of type `T`.

### Examples

**Basic iteration over a holey enum:**

```nim
import std/enumutils

type Color = enum
  Red   = 1
  Green = 3
  Blue  = 7

for c in Color:
  echo c, " = ", c.ord
# Red = 1
# Green = 3
# Blue = 7
# (Note: 2, 4, 5, 6 are never yielded — they are gaps)
```

**Collecting all members into a sequence with `toSeq`:**

```nim
import std/enumutils
from std/sequtils import toSeq

type Priority = enum
  Low    = 10
  Medium = 20
  High   = 50

let all = Priority.toSeq
assert all == [Low, Medium, High]
```

**Generic enum type parameter:**

```nim
import std/enumutils
from std/sequtils import toSeq

type Container[T] = enum
  Empty  = 0
  Single = 5
  Many   = 9

assert Container[string].toSeq == [Container[string].Empty,
                                   Container[string].Single,
                                   Container[string].Many]
```

**Why this is needed — the danger of naive iteration:**

```nim
# WRONG: for a holey enum, this would iterate invalid ordinals
# for i in Low.ord .. High.ord:
#   echo Priority(i)  # crashes on i=11..19, 21..49, etc.

# CORRECT
for p in Priority:
  echo p  # always safe, never hits a gap
```

---

## `symbolRank`

```nim
template symbolRank*[T: enum](a: T): int
```

### Description

Returns the **zero-based position** of enum value `a` within the list of declared members of its type `T` — counting from the first declared member at rank `0`. This is fundamentally different from `ord(a)`, which returns the integer value assigned to the field. `symbolRank` counts *how many fields come before this one in the source code*, regardless of what integer values those fields have.

For `OrdinalEnum` types the result is computed trivially at compile time as `ord(a) - T.low.ord` — a single subtraction with zero runtime cost. For `HoleyEnum` types the implementation uses a compile-time-generated lookup table for small enums (span ≤ 255), making the lookup O(1) at runtime, and falls back to a linear scan through declared members for large enums.

If `a` holds an integer value that does not correspond to any declared field (which is possible through unsafe casting), `symbolRank` raises `IndexDefect`.

### Parameters

| Name | Type | Description |
|---|---|---|
| `a` | `T` (any enum) | The enum value whose rank is queried |

### Return value

A non-negative `int` in the range `0 ..< T.enumLen` representing the declaration-order position of `a`.

### `symbolRank` vs `ord` — the key distinction

| | `ord(a)` | `a.symbolRank` |
|---|---|---|
| What it returns | The integer value assigned to the field | The 0-based position in the declaration list |
| For `enum { a=10, b, c }` with `c` | `12` | `2` |
| For `enum { a=-3, b=10, c=11 }` with `c` | `11` | `2` |
| For `enum { a, b, c }` with `c` | `2` | `2` (same for contiguous) |

### Examples

**Contiguous (ordinal) enum — rank equals ordinal offset:**

```nim
import std/enumutils

type Direction = enum
  North  # ord = 0, rank = 0
  East   # ord = 1, rank = 1
  South  # ord = 2, rank = 2
  West   # ord = 3, rank = 3

assert South.symbolRank == 2
assert South.ord == 2   # happen to be the same for contiguous enums
```

**Holey enum starting at a non-zero value:**

```nim
import std/enumutils

type Status = enum
  Pending  = 100
  Active   = 200
  Closed   = 500

assert Pending.symbolRank == 0   # first declared → rank 0
assert Active.symbolRank  == 1   # second declared → rank 1
assert Closed.symbolRank  == 2   # third declared → rank 2

assert Pending.ord == 100        # ordinal is 100, not 0
assert Closed.ord  == 500        # ordinal is 500, not 2
```

**Enum with negative and non-contiguous values:**

```nim
import std/enumutils

type A = enum
  a0 = -3
  a1 = 10
  a2       # = 11 (auto-increments from previous)
  a3 = (20, "f3Alt")   # ordinal 20, custom string repr

assert a0.symbolRank == 0
assert a2.symbolRank == 2
assert a3.symbolRank == 3
assert a2.ord == 11
assert a3.ord == 20
```

**IndexDefect for an invalid ordinal:**

```nim
import std/enumutils

type Sparse = enum
  X = 1
  Y = 3

var bad = 2.Sparse   # unsafe: 2 is a gap, not a declared field
doAssertRaises(IndexDefect):
  discard bad.symbolRank
```

---

## `symbolName`

```nim
func symbolName*[T: enum](a: T): string
```

### Description

Returns the **source-code identifier name** of the enum field `a` — the name as it was written in the `type` declaration — as a string. This is distinct from `$a` (the `$` operator), which returns the field's *string representation*. In Nim, an enum field can be given a custom string representation that differs from its identifier name by writing `fieldName = (ordinalValue, "customString")` or `fieldName = "customString"`. In that case `$a` returns the custom string while `symbolName(a)` always returns the identifier.

This is useful any time you need the actual Nim identifier — for code generation, serialisation protocols that use identifiers rather than display names, debugging, or documentation.

Internally `symbolName` uses `symbolRank` to index into a compile-time-generated array of identifier strings, so its performance characteristics mirror those of `symbolRank`.

### Parameters

| Name | Type | Description |
|---|---|---|
| `a` | `T` (any enum) | The enum value whose identifier name is requested |

### Return value

A `string` containing the source-code identifier of `a`'s field.

### `symbolName` vs `$` — the key distinction

| Field declaration | `$a` | `a.symbolName` |
|---|---|---|
| `b0 = (10, "kb0")` | `"kb0"` | `"b0"` |
| `b1 = "kb1"` | `"kb1"` | `"b1"` |
| `b2` (no custom string) | `"b2"` | `"b2"` (same) |

### Examples

**Enum with custom string representations:**

```nim
import std/enumutils

type B = enum
  b0 = (10, "kb0")   # ordinal 10, displayed as "kb0"
  b1 = "kb1"         # ordinal 11 (auto), displayed as "kb1"
  b2                 # ordinal 12 (auto), displayed as "b2"

let x = B.low   # = b0

assert x.symbolName == "b0"    # identifier name
assert $x        == "kb0"      # custom display string

assert B.high.symbolName == "b2"
assert $B.high   == "b2"       # no custom string → same as identifier
```

**Holey enum — identifier name is always the declared name:**

```nim
import std/enumutils

type C = enum
  c0 = -3
  c1 = 4
  c2 = 20

assert c1.symbolName == "c1"
assert c1.ord == 4
```

**Using `symbolName` for serialisation that must round-trip via identifier:**

```nim
import std/enumutils

type Severity = enum
  Debug   = (0, "DBG")
  Info    = (1, "INF")
  Warning = (2, "WRN")
  Error   = (3, "ERR")

# $sev gives the short display form; symbolName gives the canonical identifier
for sev in Severity:
  echo "id=", sev.symbolName, "  display=", $sev
# id=Debug    display=DBG
# id=Info     display=INF
# id=Warning  display=WRN
# id=Error    display=ERR
```

**Compile-time use:**

```nim
import std/enumutils

type Flag = enum FlagA, FlagB, FlagC

static:
  assert FlagC.symbolName == "FlagC"
```

---

## `genEnumCaseStmt`

```nim
macro genEnumCaseStmt*(typ: typedesc, argSym: typed, default: typed,
    userMin, userMax: static[int],
    normalizer: static[proc(s: string): string]): untyped
```

### Description

Generates a `case` statement at compile time that converts a **normalised string** into a value of the enum type `typ`. This is the foundational building block for implementing `parseEnum`-style procedures with custom string normalisation — for example, case-insensitive parsing.

The macro walks the AST of `typ`'s enum declaration, applies `normalizer` to every field's string representation, and emits an `of` branch for each distinct normalised string mapping it to the corresponding enum value. If the input string matches no branch, the macro either raises `ValueError` (when `default` is `nil`) or falls through to a caller-supplied default value.

This macro is intended for **library and framework authors** who need to build generic string-to-enum parsers. Typical application code uses the higher-level `parseEnum` from `std/strutils` instead.

**Important constraints:**
- `argSym` must be a resolved symbol (`typed`) holding the string to match against.
- `default` is either `nil` (raise on no match) or a `typed` symbol naming the fallback enum value.
- `userMin` / `userMax` allow restricting which ordinal range of fields is included in the generated `case`.
- `normalizer` must be a `static` procedure because it is called at compile time to normalise the field strings that are baked into the `of` branches.
- Ambiguous enums — those where two fields normalise to the same string — are rejected with a compile-time error.

### Parameters

| Name | Type | Description |
|---|---|---|
| `typ` | `typedesc` | The enum type to generate the case statement for |
| `argSym` | `typed` | The symbol (variable) holding the input string |
| `default` | `typed` | `nil` to raise on no match, or a symbol for the fallback value |
| `userMin` | `static[int]` | Minimum ordinal value of fields to include |
| `userMax` | `static[int]` | Maximum ordinal value of fields to include (inclusive) |
| `normalizer` | `static[proc(s:string):string]` | String normalisation function applied at both compile time and runtime |

### Return value

A `case` statement `NimNode` that can be used directly as the body of a generated procedure.

### Example

The following shows a minimal custom `parseEnum` built on top of `genEnumCaseStmt`:

```nim
import std/enumutils
import std/macros

type Direction = enum
  North = "north"
  East  = "east"
  South = "south"
  West  = "west"

func myNormalize(s: string): string =
  # simple lowercase normalisation
  var r = s
  for c in mitems(r): c = c.toLowerAscii
  r

macro parseDirection(input: typed): Direction =
  genEnumCaseStmt(Direction, input, nil,
                  Direction.low.ord, Direction.high.ord,
                  myNormalize)

let d = parseDirection("NORTH")
assert d == North

# Calling with an invalid string raises ValueError at runtime:
# let bad = parseDirection("diagonal")  # → ValueError: "Invalid enum value: diagonal"
```

**With a fallback default instead of raising:**

```nim
import std/enumutils
import std/macros

type Color = enum
  Red, Green, Blue

func id(s: string): string = s   # identity normalizer

macro tryParseColor(input: typed): Color =
  let fallback = bindSym"Red"    # default to Red on no match
  genEnumCaseStmt(Color, input, fallback,
                  Color.low.ord, Color.high.ord, id)

assert tryParseColor("Green") == Green
assert tryParseColor("Purple") == Red   # no match → default
```

---

## Relationships between the exported symbols

```
symbolRank(a)        → 0-based position in the declaration list
    │
    └─ used by symbolName(a) → source-code identifier string

items(E)             → safe iteration over HoleyEnum
    │
    └─ internally uses enumFullRange (private macro) to get all members

genEnumCaseStmt(…)   → compile-time case-statement builder for string→enum parsing
    │
    └─ foundation for parseEnum-style utilities; uses symbolRank concepts internally
```

## Summary table

| Symbol | Kind | Works on | Purpose |
|---|---|---|---|
| `items(E)` | iterator | `HoleyEnum` | Safely iterate all declared members |
| `symbolRank(a)` | template | any enum | Declaration-order position of a value |
| `symbolName(a)` | func | any enum | Source-code identifier name of a value |
| `genEnumCaseStmt(…)` | macro | any enum | Generate a string→enum `case` statement |
