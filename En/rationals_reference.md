# Nim `rationals` Module — Complete Reference

## Overview

The `rationals` module implements **exact rational arithmetic** for Nim. A rational number is a fraction of two integers — a numerator and a denominator — and can represent values like ½, −⅗, or 8/15 without any floating-point rounding error.

The module is built around the generic type `Rational[T]`, where `T` can be any integer type (`int`, `int64`, `int32`, etc.). All operations automatically reduce fractions to their lowest terms, so you never have to worry about `2/4` vs `1/2` — they are always the same.

### Why use rationals instead of floats?

Floating-point arithmetic is fast but inherently imprecise. For example:

```nim
echo 0.1 + 0.2   # prints 0.30000000000000004, not 0.3
```

With rationals:

```nim
import std/rationals
echo 1 // 10 + 2 // 10   # prints 3/10 — exact
```

### Quick taste

```nim
import std/rationals

let r1 = 1 // 2
let r2 = -3 // 4

doAssert r1 + r2 == -1 // 4
doAssert r1 - r2 ==  5 // 4
doAssert r1 * r2 == -3 // 8
doAssert r1 / r2 == -2 // 3
```

---

## The `Rational[T]` Type

```nim
type Rational*[T] = object
  num*, den*: T
```

The core data type. It holds two public fields:

| Field | Description |
|-------|-------------|
| `num` | Numerator — the top part of the fraction |
| `den` | Denominator — the bottom part of the fraction; must never be 0 |

You can access these fields directly for reading, but it is almost always better to use the provided functions so that reduction is applied correctly.

---

## Exported API — Quick Reference

| Symbol | Category | Description |
|--------|----------|-------------|
| `Rational[T]` | Type | Rational number type |
| `reduce` | Construction | Reduce a rational to lowest terms in-place |
| `initRational` | Construction | Create a rational from numerator/denominator |
| `//` | Construction | Shorthand operator for `initRational` |
| `$` | Conversion | Convert to string `"num/den"` |
| `toRational(T)` | Conversion | Integer → rational |
| `toRational(float)` | Conversion | Float → best rational approximation |
| `toFloat` | Conversion | Rational → float |
| `toInt` | Conversion | Rational → int (truncates) |
| `+`, `+=` | Arithmetic | Addition (rational/rational, rational/int) |
| `-` (unary) | Arithmetic | Negation |
| `-`, `-=` | Arithmetic | Subtraction |
| `*`, `*=` | Arithmetic | Multiplication |
| `reciprocal` | Arithmetic | Flip numerator and denominator |
| `/`, `/=` | Arithmetic | Division |
| `div` | Arithmetic | Truncated integer division of two rationals |
| `mod` | Arithmetic | Remainder after truncated division |
| `floorDiv` | Arithmetic | Floor integer division of two rationals |
| `floorMod` | Arithmetic | Remainder after floor division (Python-style `%`) |
| `^` | Arithmetic | Integer exponentiation |
| `abs` | Arithmetic | Absolute value |
| `cmp` | Comparison | Three-way comparison |
| `<`, `<=`, `==` | Comparison | Relational operators |
| `hash` | Utility | Hash for use in tables/sets |

---

## Construction and Initialisation

### `reduce`

```nim
func reduce*[T: SomeInteger](x: var Rational[T])
```

Reduces the fraction `x` in-place so that the numerator and denominator share no common divisors other than 1. The sign is always moved to the numerator — the denominator is always positive after reduction.

This function is called **automatically** by all arithmetic operations and by `initRational` / `//`, so you generally do not need to call it yourself. It is useful if you construct a `Rational` manually by setting `.num` and `.den` directly.

Raises `DivByZeroDefect` if the denominator is 0.

```nim
import std/rationals

var r = Rational[int](num: 6, den: -9)   # −6/−9, built manually
reduce(r)
doAssert r.num == 2
doAssert r.den == 3   # sign moved; den is always positive
```

---

### `initRational`

```nim
func initRational*[T: SomeInteger](num, den: T): Rational[T]
```

Creates a new `Rational[T]` from explicit numerator and denominator values, immediately reduces the fraction, and returns it. This is the canonical constructor.

`den` must not be 0 (checked with `assert` when assertions are enabled).

```nim
import std/rationals

let r = initRational(3, 6)
doAssert r.num == 1   # automatically reduced to 1/2
doAssert r.den == 2
```

---

### `//` (double-slash operator)

```nim
func `//`*[T](num, den: T): Rational[T]
```

A concise, operator-style alternative to `initRational`. It reads naturally as a fraction: `1 // 3` means "one third". This is what you will use most of the time.

```nim
import std/rationals

let r = 1 // 3 + 1 // 5
doAssert r == 8 // 15
```

Negative fractions: the sign goes in the numerator by convention.

```nim
let r = -3 // 4
doAssert r.num == -3
doAssert r.den ==  4
```

---

## Conversion Functions

### `$` — to string

```nim
func `$`*[T](x: Rational[T]): string
```

Returns a string of the form `"numerator/denominator"`. Useful for display and debugging.

```nim
import std/rationals

doAssert $(1 // 2) == "1/2"
doAssert $(-3 // 4) == "-3/4"
```

---

### `toRational` (from integer)

```nim
func toRational*[T: SomeInteger](x: T): Rational[T]
```

Wraps an integer as a rational number with denominator 1. Every integer is a valid rational (e.g. 5 = 5/1).

```nim
import std/rationals

doAssert toRational(42) == 42 // 1
doAssert toRational(0)  ==  0 // 1
```

---

### `toRational` (from float)

```nim
func toRational*(x: float, n: int = ...): Rational[int]
```

Finds the **best rational approximation** of the floating-point number `x` whose denominator does not exceed `n`. The default for `n` is very large, giving maximum precision.

The algorithm is based on **continued fractions** (David Eppstein, UC Irvine, 1993) and is guaranteed to find the simplest fraction that rounds back to the original `float`.

```nim
import std/rationals

let r = (1.2).toRational
doAssert r.toFloat == 1.2   # exact round-trip

doAssert (0.5).toRational == 1 // 2
doAssert (0.1).toRational.toFloat == 0.1
```

Limiting the denominator gives simpler but less precise results:

```nim
import std/rationals

let approx = toRational(3.14159, 100)
echo approx   # 22/7 or similar small-denominator approximation
```

---

### `toFloat`

```nim
func toFloat*[T](x: Rational[T]): float
```

Converts the rational to a `float` by performing the division `num / den`. This may lose precision for fractions that cannot be represented exactly in floating point.

```nim
import std/rationals

doAssert (1 // 4).toFloat == 0.25
doAssert (1 // 3).toFloat == 0.3333333333333333   # not exact in float
```

---

### `toInt`

```nim
func toInt*[T](x: Rational[T]): int
```

Converts the rational to an integer by performing integer division `num div den`, which **truncates toward zero** (discards the fractional part).

```nim
import std/rationals

doAssert (7 // 2).toInt ==  3   # 3.5 → 3
doAssert (-7 // 2).toInt == -3  # -3.5 → -3 (toward zero)
doAssert (1 // 2).toInt ==  0
```

---

## Arithmetic Operations

All binary operators come in three flavours:
- `rational OP rational`
- `rational OP integer`
- `integer OP rational`

All compound assignment operators (`+=`, `-=`, `*=`, `/=`) modify the left-hand variable in-place.

Every operation that produces a rational automatically calls `reduce`, so results are always in lowest terms.

---

### Addition: `+`, `+=`

```nim
func `+`*[T](x, y: Rational[T]): Rational[T]
func `+`*[T](x: Rational[T], y: T): Rational[T]
func `+`*[T](x: T, y: Rational[T]): Rational[T]
func `+=`*[T](x: var Rational[T], y: Rational[T])
func `+=`*[T](x: var Rational[T], y: T)
```

Adds two values. When adding rationals with different denominators, the implementation finds the least common multiple (LCM) to avoid unnecessary growth of numerator and denominator.

```nim
import std/rationals

doAssert 1 // 4 + 1 // 4 == 1 // 2   # same denominator
doAssert 1 // 3 + 1 // 4 == 7 // 12  # different denominators → LCM(3,4) = 12
doAssert 1 // 2 + 1      == 3 // 2   # rational + int

var r = 1 // 6
r += 1 // 3
doAssert r == 1 // 2
```

---

### Negation (unary minus): `-`

```nim
func `-`*[T](x: Rational[T]): Rational[T]
```

Returns the additive inverse of `x` — flips the sign of the numerator.

```nim
import std/rationals

doAssert -(3 // 4)  == -3 // 4
doAssert -(-1 // 5) ==  1 // 5
```

---

### Subtraction: `-`, `-=`

```nim
func `-`*[T](x, y: Rational[T]): Rational[T]
func `-`*[T](x: Rational[T], y: T): Rational[T]
func `-`*[T](x: T, y: Rational[T]): Rational[T]
func `-=`*[T](x: var Rational[T], y: Rational[T])
func `-=`*[T](x: var Rational[T], y: T)
```

Subtracts the second value from the first. Like `+`, uses LCM of denominators.

```nim
import std/rationals

doAssert 3 // 4 - 1 // 4 == 1 // 2
doAssert 3 // 4 - 1      == -1 // 4   # rational - int
doAssert 1 - 3 // 4      ==  1 // 4   # int - rational

var r = 5 // 6
r -= 1 // 3
doAssert r == 1 // 2
```

---

### Multiplication: `*`, `*=`

```nim
func `*`*[T](x, y: Rational[T]): Rational[T]
func `*`*[T](x: Rational[T], y: T): Rational[T]
func `*`*[T](x: T, y: Rational[T]): Rational[T]
func `*=`*[T](x: var Rational[T], y: Rational[T])
func `*=`*[T](x: var Rational[T], y: T)
```

Multiplies numerators together and denominators together, then reduces.

```nim
import std/rationals

doAssert 2 // 3 * 3 // 4  == 1 // 2   # (2·3)/(3·4) = 6/12 → 1/2
doAssert 3 // 5 * 2       == 6 // 5   # rational * int
doAssert 4 * (1 // 8)     == 1 // 2   # int * rational

var r = 1 // 3
r *= 3
doAssert r == 1 // 1
```

---

### `reciprocal`

```nim
func reciprocal*[T](x: Rational[T]): Rational[T]
```

Returns `1/x` — swaps numerator and denominator. If `x` is zero, raises `DivByZeroDefect`. The sign is handled correctly: `reciprocal(-2/3)` returns `-3/2`.

```nim
import std/rationals

doAssert reciprocal(2 // 3) == 3 // 2
doAssert reciprocal(-2 // 3) == -3 // 2
```

---

### Division: `/`, `/=`

```nim
func `/`*[T](x, y: Rational[T]): Rational[T]
func `/`*[T](x: Rational[T], y: T): Rational[T]
func `/`*[T](x: T, y: Rational[T]): Rational[T]
func `/=`*[T](x: var Rational[T], y: Rational[T])
func `/=`*[T](x: var Rational[T], y: T)
```

Divides `x` by `y`. Dividing by a rational is equivalent to multiplying by its reciprocal.

```nim
import std/rationals

doAssert (1 // 2) / (3 // 4) == 2 // 3
doAssert (3 // 4) / 3        == 1 // 4   # rational / int
doAssert 3 / (3 // 4)        == 4 // 1   # int / rational

var r = 3 // 4
r /= 3
doAssert r == 1 // 4
```

---

### `div` — truncated integer division

```nim
func `div`*[T: SomeInteger](x, y: Rational[T]): T
```

Computes the integer part of `x / y`, truncating toward zero. The result is an **integer** (type `T`), not a rational. This mirrors the built-in `div` for integers.

```nim
import std/rationals

doAssert (7 // 2) div (1 // 1) == 3   #  3.5 → 3
doAssert (-7 // 2) div (1 // 1) == -3 # -3.5 → -3 (toward zero)
```

---

### `mod` — remainder after truncated division

```nim
func `mod`*[T: SomeInteger](x, y: Rational[T]): Rational[T]
```

Returns the remainder after truncated division, equivalent to `x - (x div y) * y`. The result is a rational. The sign of the result matches the sign of the dividend `x`.

```nim
import std/rationals

doAssert (7 // 4) mod (1 // 2) == 1 // 4
# 7/4 ÷ 1/2 = 3.5; trunc = 3; remainder = 7/4 - 3·(1/2) = 7/4 - 6/4 = 1/4
```

---

### `floorDiv` — floor integer division

```nim
func floorDiv*[T: SomeInteger](x, y: Rational[T]): T
```

Computes `floor(x / y)` — rounds **down** toward negative infinity, unlike `div` which rounds toward zero. For non-negative results, `floorDiv` and `div` are identical. They differ when the result is negative.

```nim
import std/rationals

doAssert floorDiv(7 // 2,  1 // 1) ==  3  # same as div
doAssert floorDiv(-7 // 2, 1 // 1) == -4  # div gives -3; floorDiv gives -4
```

---

### `floorMod` — remainder after floor division

```nim
func floorMod*[T: SomeInteger](x, y: Rational[T]): Rational[T]
```

Returns the remainder after floor division, equivalent to `x - floorDiv(x, y) * y`. The sign of the result always matches the sign of the **divisor** `y` — this is the same behaviour as Python's `%` operator.

```nim
import std/rationals

# Positive divisor → non-negative remainder
doAssert floorMod(-7 // 4, 1 // 2) == 1 // 4

# Compare: mod gives negative result for negative dividend
doAssert (-7 // 4) mod (1 // 2) == -1 // 4
```

---

### `^` — exponentiation

```nim
func `^`*[T: SomeInteger](x: Rational[T], y: T): Rational[T]
```

Raises rational `x` to integer power `y`. Works for both positive and negative exponents. A negative exponent flips the fraction (equivalent to raising the reciprocal to the positive power). Exponent 0 always returns `1/1`.

```nim
import std/rationals

doAssert (-3 // 5) ^ 0  == 1 // 1    # anything^0 = 1
doAssert (-3 // 5) ^ 1  == -3 // 5
doAssert (-3 // 5) ^ 2  == 9 // 25   # (-3)²/5² = 9/25
doAssert (-3 // 5) ^ -2 == 25 // 9   # reciprocal squared
```

Because all powers of an already-reduced fraction are also reduced, `reduce` is **not** called inside `^` — it is safe and efficient.

---

### `abs` — absolute value

```nim
func abs*[T](x: Rational[T]): Rational[T]
```

Returns the absolute value of the rational: both numerator and denominator become non-negative.

```nim
import std/rationals

doAssert abs( 1 // 2) == 1 // 2
doAssert abs(-1 // 2) == 1 // 2
doAssert abs(-3 // 4) == 3 // 4
```

---

## Comparison Functions

### `cmp`

```nim
func cmp*(x, y: Rational): int
```

Three-way comparison. Returns a negative number if `x < y`, zero if `x == y`, and a positive number if `x > y`. Useful when you need to sort rationals or branch on ordering.

```nim
import std/rationals

doAssert cmp(1 // 3, 1 // 2) < 0   # 1/3 < 1/2
doAssert cmp(1 // 2, 1 // 2) == 0
doAssert cmp(2 // 3, 1 // 2) > 0   # 2/3 > 1/2
```

---

### `<`, `<=`, `==`

```nim
func `<`*(x, y: Rational): bool
func `<=`*(x, y: Rational): bool
func `==`*(x, y: Rational): bool
```

Standard relational operators. All three are implemented by subtracting the two rationals and checking the sign of the numerator of the result, so comparison is always exact — no floating-point rounding.

Equality works correctly across different but equivalent representations, because both sides are reduced before comparison.

```nim
import std/rationals

doAssert 1 // 3 <  1 // 2
doAssert 1 // 2 <= 1 // 2
doAssert 2 // 4 == 1 // 2   # different form, same value
doAssert not (1 // 3 == 1 // 4)
```

---

## Utility

### `hash`

```nim
func hash*[T](x: Rational[T]): Hash
```

Computes a hash value for the rational, suitable for use as a key in a `Table` or element of a `HashSet`. Critically, the function first reduces `x` before hashing, which guarantees that mathematically equal rationals (like `2/4` and `1/2`) always produce the **same hash**.

```nim
import std/[rationals, tables]

var t = initTable[Rational[int], string]()
t[1 // 2] = "one half"

doAssert t[2 // 4] == "one half"   # 2/4 == 1/2, same hash, same bucket
```

---

## Practical Examples

### Exact fraction arithmetic

```nim
import std/rationals

# Sum of a harmonic series: 1 + 1/2 + 1/3 + 1/4
var sum = 0 // 1
for i in 1..4:
  sum += 1 // i
doAssert sum == 25 // 12
```

### Float → rational → float round-trip

```nim
import std/rationals

for x in [0.1, 0.5, 1.25, 3.14159]:
  let r = x.toRational
  doAssert r.toFloat == x
  echo x, " ≈ ", r
```

### Using rationals as table keys

```nim
import std/[rationals, tables]

var fractions = initTable[Rational[int], string]()
fractions[1 // 2] = "half"
fractions[1 // 3] = "third"
fractions[1 // 4] = "quarter"

doAssert fractions[2 // 4] == "quarter"   # 2/4 reduces to 1/4 → same key
```

### Exponentiation and reciprocal

```nim
import std/rationals

let base = 2 // 3
doAssert base ^ 3  == 8 // 27
doAssert base ^ -3 == 27 // 8
doAssert reciprocal(base ^ 3) == base ^ -3
```

---

## Error Conditions

| Situation | Error raised |
|-----------|-------------|
| `reduce` called with `den == 0` | `DivByZeroDefect` |
| `initRational` / `//` with `den == 0` | `AssertionDefect` (assertions on) |
| `reciprocal` called on zero | `DivByZeroDefect` |

---

## See Also

- [`std/math`](https://nim-lang.org/docs/math.html) — `gcd`, `lcm`, `floorDiv`, `floorMod` used internally
- [`std/hashes`](https://nim-lang.org/docs/hashes.html) — hashing infrastructure
- [`std/complex`](https://nim-lang.org/docs/complex.html) — complex numbers in the standard library
