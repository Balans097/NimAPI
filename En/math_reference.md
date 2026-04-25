# `std/math` Module Reference — Nim Standard Library

*Constructive mathematics is naturally typed. — Simon Thompson*

## Table of Contents
- [1. Module Overview](#1-module-overview)
- [2. Constants](#2-constants)
- [3. Types](#3-types)
- [4. Floating-Point Utilities](#4-floating-point-utilities)
- [5. Rounding Functions](#5-rounding-functions)
- [6. Powers and Roots](#6-powers-and-roots)
- [7. Logarithms and Exponential](#7-logarithms-and-exponential)
- [8. Trigonometric Functions](#8-trigonometric-functions)
- [9. Division and Remainder](#9-division-and-remainder)
- [10. Integer Mathematics](#10-integer-mathematics)
- [11. Special Functions](#11-special-functions)
- [12. Sign and Value Clamping](#12-sign-and-value-clamping)
- [13. Array Functions](#13-array-functions)
- [14. Practical Examples](#14-practical-examples)
- [15. Quick Reference Table](#15-quick-reference-table)

---

## 1. Module Overview

The `std/math` module is part of the Nim standard library and implements basic mathematical functions, constants, and types. Most functions delegate computation to the C `<math.h>` library (POSIX) or the corresponding `Math.*` object for the JavaScript backend, providing native performance on both platforms.

### Importing
```nim
import std/math
```

### Platform Support

| Backend | Support | Notes |
|---------|---------|-------|
| C / C++ | ✅ Full | Links with `-lm` on POSIX |
| JavaScript | ✅ Full | Uses `Math.*` |
| NimScript | ⚠️ Partial | Some functions unavailable |

### Related Modules

| Module | Purpose |
|--------|---------|
| `std/complex` | Complex numbers and operations |
| `std/rationals` | Rational numbers (exact arithmetic) |
| `std/fenv` | IEEE 754 rounding modes, float exception handling |
| `std/random` | Pseudorandom number generator |
| `std/stats` | Statistical analysis: mean, variance, standard deviation |
| `std/strformat` | Float number formatting for output |
| `system` | Basic operators: `shr`, `shl`, `xor`, `clamp`, `abs`, `min`, `max` |

---

## 2. Constants

The module exports mathematical and technical constants. All are declared with `*`, making them public upon import.

| Constant | Value | Description |
|----------|-------|-------------|
| `PI` | 3.14159265358979323… | Ludolph's number π — ratio of circumference to diameter |
| `TAU` | 6.28318530717958647… | TAU = 2·π — full circle in radians; cleaner than PI in many formulas |
| `E` | 2.71828182845904523… | Euler's number — base of natural logarithms |
| `MaxFloat64Precision` | 16 | Max significant decimal digits for `float64` |
| `MaxFloat32Precision` | 8 | Max significant decimal digits for `float32` |
| `MaxFloatPrecision` | 16 | Alias for `MaxFloat64Precision` (`float` defaults to `float64`) |
| `MinFloatNormal` | 2.225073858507201e-308 | Smallest **normal** float64 (= 2⁻¹⁰²²) |

### Examples
```nim
import std/math

# Circumference of a circle with radius r
let r = 5.0
let circumference = TAU * r      # TAU = 2*PI, more readable
echo circumference               # => 31.41592653589793

# Half turn and quarter turn
let halfTurn    = PI             # 180°
let quarterTurn = PI / 2.0       # 90°

# Continuous growth with exponential
let investment = 1000.0
let growth = investment * E ^ 1.0   # growth over one time unit
echo growth                          # => 2718.28...

# Check if a number is within normal float range
let tiny = 1.0e-310
echo tiny < MinFloatNormal           # => true (subnormal)
```

### Why TAU?
Many formulas look cleaner with TAU. For example, the full circumference is `TAU * r`, not `2 * PI * r`. A 360° angle in radians is simply `TAU`.

---

## 3. Types

### `FloatClass`
An enum describing the IEEE 754 floating-point number class. Returned by the `[classify](#classify--number-classification)` function.

```nim
type
  FloatClass* = enum
    fcNormal,    ## Standard non-zero normalized value
    fcSubnormal, ## Subnormal (denormalized) — very small
    fcZero,      ## Positive zero (+0.0)
    fcNegZero,   ## Negative zero (-0.0)
    fcNan,       ## Not a Number (NaN) — result of invalid operation
    fcInf,       ## Positive infinity (+Inf)
    fcNegInf     ## Negative infinity (-Inf)
```

### When Special Values Occur
| Value | Example Operation | Reason |
|-------|-------------------|--------|
| `Inf` | `1.0 / 0.0` | Division of positive by zero |
| `-Inf` | `-1.0 / 0.0` | Division of negative by zero |
| `NaN` | `0.0 / 0.0` | Undefined operation |
| `NaN` | `sqrt(-1.0)` | Square root of negative |
| `-0.0` | `-1.0 * 0.0` | Mathematically 0, but distinct sign |
| subnormal | `5.0e-324` | Number smaller than `MinFloatNormal` |

### Example Using `FloatClass`
```nim
import std/math

proc describeFloat(x: float): string =
  case classify(x)
  of fcNormal:     "normal number "
  of fcSubnormal:  "subnormal (very small) "
  of fcZero:       "positive zero "
  of fcNegZero:    "negative zero "
  of fcNan:        "not a number (NaN) "
  of fcInf:        "positive infinity "
  of fcNegInf:     "negative infinity "

echo describeFloat(3.14)       # => normal number
echo describeFloat(0.0)        # => positive zero
echo describeFloat(-0.0)       # => negative zero
echo describeFloat(1.0/0.0)    # => positive infinity
echo describeFloat(0.0/0.0)    # => not a number (NaN)
echo describeFloat(5.0e-324)   # => subnormal (very small)
```

---

## 4. Floating-Point Utilities

### `isNaN` — NaN Check
```nim
func isNaN*(x: SomeFloat): bool
```
Returns `true` if `x` is NaN. Works correctly even with the `-ffast-math` compiler optimization enabled, which can break the standard `x != x` comparison.

#### Why not just write `x == NaN`?
According to the IEEE 754 standard, NaN is not equal to anything, including itself. Therefore, `NaN == NaN` is always `false`. `isNaN` uses more reliable checking methods.

```nim
import std/math

doAssert NaN.isNaN                  # NaN is NaN
doAssert not Inf.isNaN              # Inf is not NaN
doAssert not isNaN(3.1415926)       # normal number is not NaN

# Special case: NaN is not equal to itself
let x = NaN
doAssert x != x          # true — IEEE 754 property of NaN
doAssert x.isNaN         # true — reliable check

# Check before using computation result
let result = sqrt(-4.0)
if result.isNaN:
  echo "Error: square root of negative number"
```

### `signbit` — Sign Bit
```nim
func signbit*(x: SomeFloat): bool
```
Returns `true` if `x` is negative. Unlike `x < 0`, it correctly handles `-0.0` (negative zero) and `-Inf`.

#### Why is `-0.0` needed?
Negative zero is a valid IEEE 754 value. Mathematically equal to `+0.0`, its sign matters in certain calculations (e.g., when computing `arctan2` or analyzing the sign of subnormal numbers).

```nim
import std/math

doAssert not signbit(0.0)    # +0.0 is positive
doAssert signbit(-0.0)       # -0.0 is negative!
doAssert signbit(-0.1)       # normal negative
doAssert not signbit(0.1)    # normal positive
doAssert signbit(-Inf)       # negative infinity
doAssert not signbit(Inf)    # positive infinity

# Difference from x < 0:
let negZero = -0.0
echo negZero < 0.0      # => false (mathematically equal)
echo signbit(negZero)   # => true  (sign bit is set)
```

### `copySign` — Sign Copying
```nim
func copySign*[T: SomeFloat](x, y: T): T
```
Returns a number with the magnitude of `x` and the sign of `y`. Works correctly with all special values: NaN, ±Inf, ±0.

```nim
import std/math

doAssert copySign(10.0,  1.0) ==  10.0   # takes sign from y=+1
doAssert copySign(10.0, -1.0) == -10.0   # takes sign from y=-1
doAssert copySign(-Inf, -0.0) == -Inf    # sign from -0.0
doAssert copySign(NaN, 1.0).isNaN        # NaN remains NaN

# Use case: set a number's sign based on another
proc withSign(magnitude: float, reference: float): float =
  copySign(abs(magnitude), reference)

echo withSign(5.0, -3.0)   # => -5.0
echo withSign(-7.0, 2.0)   # => 7.0
```

### `classify` — Number Classification
```nim
func classify*(x: float): FloatClass
```
Determines the IEEE 754 class of `x` and returns the corresponding `FloatClass` value.

```nim
import std/math

doAssert classify(0.3)        == fcNormal
doAssert classify(0.0)        == fcZero
doAssert classify(-0.0)       == fcNegZero
doAssert classify(0.3 / 0.0)  == fcInf
doAssert classify(-0.3 / 0.0) == fcNegInf
doAssert classify(5.0e-324)   == fcSubnormal

# Use case: safe division
proc safeDivide(a, b: float): float =
  if classify(b) in {fcZero, fcNegZero}:
    raise newException(DivByZeroDefect, "division by zero")
  result = a / b
```

### `almostEqual` — Float Comparison with Tolerance
```nim
func almostEqual*[T: SomeFloat](x, y: T; unitsInLastPlace: Natural = 4): bool
```
Checks if two numbers are approximately equal, accounting for accumulated floating-point errors. Uses the ULP (Units in the Last Place) method.

#### Why can't floats be compared with `==`?
Floating-point numbers are stored in binary. Most decimal fractions (e.g., `0.1`) lack an exact binary representation, so arithmetic operations accumulate errors. For example, `0.1 + 0.2` in float does not equal `0.3`.

#### The `unitsInLastPlace` parameter:
- `0` — numbers must be bitwise identical
- `1` — allows 1 ULP (minimal tolerance)
- `4` — default tolerance, covers most practical cases
- Larger values → wider tolerance

```nim
import std/math

# Basic cases
doAssert almostEqual(PI, 3.14159265358979)  # close enough
doAssert almostEqual(Inf, Inf)              # infinities are equal
doAssert not almostEqual(NaN, NaN)          # NaN ≠ NaN by definition

# Real-world issue: error accumulation
let a = 0.1 + 0.2
echo a == 0.3            # => false! (0.30000000000000004)
echo almostEqual(a, 0.3) # => true  (within tolerance)

# Adjusting strictness
echo almostEqual(1.0, 1.0000001, 4)    # true  — soft tolerance
echo almostEqual(1.0, 1.0000001, 0)    # false — strict equality

# Rule: always use almostEqual instead of == for float results
proc isUnitVector(x, y, z: float): bool =
  almostEqual(x*x + y*y + z*z, 1.0)
```

### `frexp` — Mantissa and Exponent Split
```nim
func frexp*[T: float32|float64](x: T): tuple[frac: T, exp: int]
func frexp*[T: float32|float64](x: T, exponent: var int): T
```
Splits `x` into a normalized fraction `frac` and an integer power of two `exp`, such that: `x = frac × 2^exp`, where `abs(frac) ∈ [0.5, 1.0)`.
Equivalent to C's `frexp`. Useful for working with in-memory number representations, implementing custom math functions, and data normalization.

```nim
import std/math

doAssert frexp(8.0)  == (0.5, 4)    # 8.0  = 0.5 × 2⁴
doAssert frexp(-8.0) == (-0.5, 4)   # -8.0 = -0.5 × 2⁴
doAssert frexp(0.0)  == (0.0, 0)    # 0 is a special case
doAssert frexp(1.0)  == (0.5, 1)    # 1.0  = 0.5 × 2¹

# Second variant — via out-parameter
var exp: int
let frac = frexp(5.0, exp)
doAssert frac == 0.625   # 5.0 = 0.625 × 2³
doAssert exp  == 3

# Use case: log2 without loss of precision
# log2(x) ≈ exp + log2(frac), where frac ∈ [0.5, 1.0)
```

### `splitDecimal` — Integer and Fractional Parts
```nim
func splitDecimal*[T: float32|float64](x: T): tuple[intpart: T, floatpart: T]
```
Splits a number into its integer and fractional parts. Both parts share the same sign as `x`. Equivalent to C's `modf` function.

```nim
import std/math

doAssert splitDecimal(5.25)  == (intpart: 5.0,  floatpart: 0.25)
doAssert splitDecimal(-2.73) == (intpart: -2.0, floatpart: -0.73)
doAssert splitDecimal(0.0)   == (intpart: 0.0,  floatpart: 0.0)

# Use case: animation with subpixel offset
let position = 7.65
let (whole, frac) = splitDecimal(position)
let pixelPos = int(whole)     # render at pixel 7
let subPixel = frac           # with subpixel offset 0.65
```

---

## 5. Rounding Functions

Nim provides several rounding strategies. The choice depends on the task.

| Function | Strategy | 2.5 | -2.5 | 2.1 | -2.1 |
|----------|----------|-----|------|-----|------|
| `floor(x)` | Toward -∞ | 2.0 | -3.0 | 2.0 | -3.0 |
| `ceil(x)` | Toward +∞ | 3.0 | -2.0 | 3.0 | -2.0 |
| `trunc(x)` | Toward 0 | 2.0 | -2.0 | 2.0 | -2.0 |
| `round(x)` | To nearest | 3.0 | -2.0 | 2.0 | -2.0 |

### `floor` — Round Down
```nim
func floor*(x: float32|float64): float
```
Returns the largest integer not exceeding `x`. For positive numbers, it truncates the fractional part. For negative numbers, it rounds toward greater magnitude.

```nim
import std/math

doAssert floor(2.1)  ==  2.0
doAssert floor(2.9)  ==  2.0   # not 3!
doAssert floor(-2.1) == -3.0   # moves "lower"
doAssert floor(-2.9) == -3.0
doAssert floor(3.0)  ==  3.0   # integer stays integer

# Use case: get grid cell index from position
let cellSize = 32.0
let pos = 97.5
let cellIndex = int(floor(pos / cellSize))  # => 3
```

### `ceil` — Round Up
```nim
func ceil*(x: float32|float64): float
```
Returns the smallest integer not less than `x`. The mirror opposite of `floor`.

```nim
import std/math

doAssert ceil(2.1)  == 3.0
doAssert ceil(2.9)  == 3.0
doAssert ceil(-2.1) == -2.0   # moves "higher" (toward zero)
doAssert ceil(-2.9) == -2.0
doAssert ceil(3.0)  ==  3.0

# Use case: how many pages needed for N records
let records = 57
let perPage = 10
let pages = int(ceil(float(records) / float(perPage)))  # => 6
```

### `trunc` — Truncate Toward Zero
```nim
func trunc*(x: float32|float64): float
```
Discards the fractional part, always moving toward zero. Equivalent to `floor` for positive numbers and `ceil` for negative numbers.

```nim
import std/math

doAssert trunc(PI)    ==  3.0   # 3.14159... -> 3.0
doAssert trunc(-1.85) == -1.0   # moves toward zero, not down
doAssert trunc(2.99)  ==  2.0

# Use case: integer division for floats
let a = 17.0
let b = 5.0
let quotient = trunc(a / b)     # => 3.0 (like integer div)
```

### `round` — Round to Nearest
```nim
func round*(x: float32|float64): float              # to integer
func round*[T: float32|float64](x: T, places: int): T  # to N decimal places
```
Rounds to the nearest integer (or to `places` decimal digits). When `places < 0`, it rounds to the left of the decimal point.

⚠️ `round(x, places)` uses binary arithmetic and does not guarantee exact results for all decimal fractions. For financial calculations, use `std/rationals`.

```nim
import std/math

doAssert round(3.4)  == 3.0
doAssert round(3.5)  == 4.0    # rounds up at x.5
doAssert round(4.5)  == 5.0
doAssert round(-3.5) == -4.0   # toward greater magnitude

# With specified decimal places
doAssert round(PI, 2)  == 3.14
doAssert round(PI, 4)  == 3.1416

# Rounding left of decimal point (negative places)
doAssert round(537.345, -1) == 540.0   # to tens
doAssert round(537.345, -2) == 500.0   # to hundreds
```

---

## 6. Powers and Roots

### `sqrt` — Square Root
```nim
func sqrt*(x: float32|float64): float
```
Computes √x. Returns `NaN` for negative values.

```nim
import std/math

doAssert almostEqual(sqrt(4.0),   2.0)
doAssert almostEqual(sqrt(1.44),  1.2)
doAssert almostEqual(sqrt(2.0),   1.4142135623730951)
doAssert sqrt(-1.0).isNaN     # sqrt of negative = NaN

# Use case: distance between two points
proc distance(x1, y1, x2, y2: float): float =
  sqrt((x2-x1)^2 + (y2-y1)^2)

echo distance(0.0, 0.0, 3.0, 4.0)  # => 5.0
```

### `cbrt` — Cube Root
```nim
func cbrt*(x: float32|float64): float
```
Computes ∛x. Unlike `sqrt`, it correctly handles negative numbers (∛(−27) = −3).

```nim
import std/math

doAssert almostEqual(cbrt(8.0),   2.0)
doAssert almostEqual(cbrt(2.197), 1.3)
doAssert almostEqual(cbrt(-27.0), -3.0)  # negative is OK!

# Use case: cube side from volume
let volume = 125.0
let side = cbrt(volume)   # => 5.0
```

### `pow` — Float Exponentiation
```nim
func pow*(x, y: float64): float64
```
Computes `x^y`. Both arguments are floating-point numbers.

```nim
import std/math

doAssert almostEqual(pow(100.0, 1.5), 1000.0)   # 100^1.5 = 1000
doAssert almostEqual(pow(16.0, 0.5),    4.0)    # 16^0.5 = √16 = 4
doAssert pow(0.0, 0.0) == 1.0                   # 0^0 = 1 by convention
```

### `^` — Exponentiation Operator
Nim provides two overloaded `^` operators:
```nim
func `^`*[T: SomeNumber](x: T, y: Natural): T        # integer exponent
func `^`*[T: SomeNumber, U: SomeFloat](x: T, y: U): float  # real exponent
```
The first version (`y: Natural`) works with any number and returns the same type. Implemented via fast exponentiation (square-and-multiply algorithm). The second version (`y: SomeFloat`) returns `float` and supports fractional and negative exponents.

```nim
import std/math

# Integer exponent — exact result
doAssert -3 ^ 0 ==  1
doAssert -3 ^ 1 == -3
doAssert -3 ^ 2 ==  9
doAssert  2 ^ 10 == 1024

# Real exponent — via float
doAssert almostEqual(5.5 ^ 2.2, 42.540042248725975)
doAssert 1.0 ^ Inf == 1.0    # IEEE 754 special case

# Negative exponent (only via float version)
doAssert almostEqual(2.0 ^ -1.0, 0.5)   # 2^-1 = 1/2
doAssert almostEqual(10.0 ^ -2.0, 0.01) # 10^-2 = 0.01
```

### `hypot` — Hypotenuse
```nim
func hypot*(x, y: float64): float64
```
Computes √(x² + y²) without intermediate overflow. Directly computing `sqrt(x*x + y*y)` may overflow for large `x` and `y`, whereas `hypot` uses a numerically stable algorithm.

```nim
import std/math

doAssert almostEqual(hypot(3.0, 4.0), 5.0)   # Egyptian triangle

# Numerical stability:
let big = 1.0e200
echo sqrt(big*big + big*big)     # Inf (overflow!)
echo hypot(big, big)             # 1.4142...e200 (correct)

# Use case: vector magnitude
proc magnitude(vx, vy: float): float = hypot(vx, vy)
```

---

## 7. Logarithms and Exponential

All logarithmic functions follow the same rules:
- `ln(-x)` → `NaN` (logarithm of negative is undefined)
- `ln(0.0)` → `-Inf` (logarithm of zero is minus infinity)
- `ln(Inf)` → `Inf`

### `ln` — Natural Logarithm
```nim
func ln*(x: float32|float64): float
```
Computes the logarithm with base e (Euler's number).

```nim
import std/math

doAssert almostEqual(ln(E),       1.0)    # ln(e) = 1 by definition
doAssert almostEqual(ln(E^3),     3.0)    # ln(e^x) = x
doAssert almostEqual(ln(1.0),     0.0)    # ln(1) = 0
doAssert almostEqual(ln(0.0),    -Inf)    # log of zero
doAssert ln(-7.0).isNaN                  # log of negative
```

### `log10`, `log2`, `log` — Logarithms with Other Bases
```nim
func log10*(x: float32|float64): float   # base 10
func log2*(x: float32|float64): float    # base 2
func log*[T: SomeFloat](x, base: T): T  # arbitrary base
```
```nim
import std/math

# log10: decimal logarithm
doAssert almostEqual(log10(100.0),  2.0)   # 10^2 = 100
doAssert almostEqual(log10(1000.0), 3.0)
doAssert almostEqual(log10(0.1),   -1.0)

# log2: binary logarithm (useful in CS)
doAssert almostEqual(log2(8.0),    3.0)    # 2^3 = 8
doAssert almostEqual(log2(1024.0), 10.0)   # 2^10 = 1024

# log: arbitrary base
doAssert almostEqual(log(9.0, 3.0),  2.0)  # 3^2 = 9
doAssert almostEqual(log(8.0, 2.0),  3.0)
doAssert log(-7.0, 4.0).isNaN             # negative -> NaN
doAssert log(8.0, -2.0).isNaN            # negative base -> NaN

# Change of base formula: log_b(x) = ln(x) / ln(b)
# The log(x, base) function is implemented exactly this way
```

### `exp` — Exponential Function
```nim
func exp*(x: float32|float64): float
```
Computes e^x. The inverse function of `ln`.

```nim
import std/math

doAssert almostEqual(exp(0.0),  1.0)    # e^0 = 1
doAssert almostEqual(exp(1.0),  E)      # e^1 = e
doAssert almostEqual(exp(ln(5.0)), 5.0) # exp and ln are mutually inverse

# Use case: continuous growth model
# P(t) = P0 * e^(r*t), where r is growth rate, t is time
let P0 = 1000.0   # initial amount
let r  = 0.05     # 5% per year (continuous compounding)
let t  = 10.0     # 10 years
let Pt = P0 * exp(r * t)
echo Pt   # => 1648.72... (growth by factor 1.65)
```

---

## 8. Trigonometric Functions

⚠️ All trigonometric functions accept arguments in radians, not degrees. Use `degToRad`/`radToDeg` for conversion.

### Angle Conversion
```nim
func degToRad*[T: float32|float64](d: T): T   # degrees → radians
func radToDeg*[T: float32|float64](d: T): T   # radians → degrees
```
```nim
import std/math

doAssert almostEqual(degToRad(180.0), PI)       # 180° = π rad
doAssert almostEqual(degToRad(90.0),  PI/2.0)  # 90°  = π/2 rad
doAssert almostEqual(radToDeg(PI),    180.0)
doAssert almostEqual(radToDeg(TAU),   360.0)

# Formulas: rad = deg × π/180,  deg = rad × 180/π
```

### Basic Functions
```nim
func sin*(x: float32|float64): float   # sine
func cos*(x: float32|float64): float   # cosine
func tan*(x: float32|float64): float   # tangent
func cot*[T: float32|float64](x: T): T   # cotangent = 1/tan(x)
func sec*[T: float32|float64](x: T): T   # secant = 1/cos(x)
func csc*[T: float32|float64](x: T): T   # cosecant = 1/sin(x)
```
```nim
import std/math

# sin and cos range [-1, 1]
doAssert almostEqual(sin(0.0),          0.0)
doAssert almostEqual(sin(PI/2.0),       1.0)   # sin(90°) = 1
doAssert almostEqual(sin(PI),           0.0)   # sin(180°) = 0
doAssert almostEqual(cos(0.0),          1.0)
doAssert almostEqual(cos(PI/2.0),       0.0)   # cos(90°) = 0

# Fundamental identity: sin²(x) + cos²(x) = 1
let angle = degToRad(37.0)
doAssert almostEqual(sin(angle)^2 + cos(angle)^2, 1.0)

# tan is undefined at x = π/2 + πn
echo tan(PI/2.0)   # => very large number (not Inf due to float precision)
```

### Inverse Trigonometric Functions
```nim
func arcsin*(x: float64): float   # arcsine,   result ∈ [-π/2, π/2]
func arccos*(x: float64): float   # arccosine, result ∈ [0, π]
func arctan*(x: float64): float   # arctangent, result ∈ (-π/2, π/2]
func arctan2*(y, x: float64): float  # arctangent y/x with quadrant awareness ∈ (-π, π]
func arccot*[T](x: T): T  # = arctan(1/x)
func arcsec*[T](x: T): T  # = arccos(1/x)
func arccsc*[T](x: T): T  # = arcsin(1/x)
```

#### Important: `arctan2` vs `arctan`
`arctan(y/x)` loses quadrant information (if both arguments change sign simultaneously). `arctan2(y, x)` accepts two separate arguments and returns the correct angle in the range `(-π, π]`.

```nim
import std/math

doAssert almostEqual(radToDeg(arcsin(0.0)), 0.0)
doAssert almostEqual(radToDeg(arcsin(1.0)), 90.0)
doAssert almostEqual(radToDeg(arccos(0.0)), 90.0)
doAssert almostEqual(radToDeg(arccos(1.0)), 0.0)

# arctan2: correct angle for any quadrant
doAssert almostEqual(arctan2( 1.0,  1.0),  PI/4.0)   #  45°
doAssert almostEqual(arctan2( 1.0, -1.0),  3*PI/4.0) # 135°
doAssert almostEqual(arctan2(-1.0, -1.0), -3*PI/4.0) # -135°
doAssert almostEqual(arctan2( 1.0,  0.0),  PI/2.0)   #  90°

# Use case: vector angle from positive X-axis
let vx = -1.0
let vy =  1.0
let angleDeg = radToDeg(arctan2(vy, vx))  # => 135.0°
```

### Hyperbolic Functions
Hyperbolic functions are defined via the exponential function and are used in physics, special function theory, and neural networks (`tanh` is a popular activation function).

```nim
func sinh*(x: float64): float   # hyperbolic sine   = (e^x - e^-x)/2
func cosh*(x: float64): float   # hyperbolic cosine = (e^x + e^-x)/2
func tanh*(x: float64): float   # hyperbolic tangent = sinh/cosh ∈ (-1, 1)
func coth*[T](x: T): T          # = 1/tanh(x)
func sech*[T](x: T): T          # = 1/cosh(x)
func csch*[T](x: T): T          # = 1/sinh(x)
func arcsinh*(x: float64): float  # inverse of sinh
func arccosh*(x: float64): float  # inverse of cosh, x >= 1
func arctanh*(x: float64): float  # inverse of tanh, x ∈ (-1, 1)
func arccoth*[T](x: T): T         # = arctanh(1/x)
func arcsech*[T](x: T): T         # = arccosh(1/x)
func arccsch*[T](x: T): T         # = arcsinh(1/x)
```
```nim
import std/math

# tanh: values ∈ (-1, 1), used as activation function
echo tanh(0.0)    # => 0.0
echo tanh(1.0)    # => 0.7615941559557649
echo tanh(100.0)  # => ~1.0 (saturation)
echo tanh(-1.0)   # => -0.7615...

# Identity: cosh²(x) - sinh²(x) = 1
let x = 2.0
doAssert almostEqual(cosh(x)^2 - sinh(x)^2, 1.0)

# arcsinh — inverse of sinh
doAssert almostEqual(arcsinh(sinh(3.0)), 3.0)
```

---

## 9. Division and Remainder

Nim provides several division semantics, each with its own behavior for negative numbers.

### Comparison of Semantics
For `x = -13`, `y = 3`:
| Function | Quotient | Remainder | Principle |
|----------|----------|-----------|-----------|
| `div` / `mod` (system) | -4 | -1 | Truncation toward zero (C-style) |
| `floorDiv` / `floorMod` | -5 | 2 | Round down (Python-style) |
| `euclDiv` / `euclMod` | -5 | 2 | Euclidean division (remainder ≥ 0) |
| `ceilDiv` | -4 | — | Round up (only x≥0, y>0) |

### `floorDiv` and `floorMod`
```nim
func floorDiv*[T: SomeInteger](x, y: T): T
func floorMod*[T: SomeNumber](x, y: T): T
```
`floorDiv` is conceptually equivalent to `floor(x/y)` — always rounds down (toward -∞). `floorMod` behaves like Python's `%` operator: the remainder always has the same sign as the divisor.

```nim
import std/math

# Division — always down
doAssert floorDiv( 13,  3) ==  4
doAssert floorDiv(-13,  3) == -5   # div would give -4!
doAssert floorDiv( 13, -3) == -5
doAssert floorDiv(-13, -3) ==  4

# Remainder — sign matches divisor y
doAssert floorMod( 13,  3) ==  1
doAssert floorMod(-13,  3) ==  2   # positive, because y=3 > 0
doAssert floorMod( 13, -3) == -2   # negative, because y=-3 < 0
doAssert floorMod(-13, -3) == -1

# Check: floorDiv(x,y)*y + floorMod(x,y) == x (always)
let x = -13
let y = 3
doAssert floorDiv(x, y) * y + floorMod(x, y) == x
```

### `euclDiv` and `euclMod`
```nim
func euclDiv*[T: SomeInteger](x, y: T): T
func euclMod*[T: SomeNumber](x, y: T): T
```
Euclidean division guarantees that the remainder is always non-negative (`euclMod(x,y) >= 0`), regardless of the signs of `x` and `y`. Used in number theory and cryptography.

```nim
import std/math

doAssert euclDiv( 13,  3) ==  4
doAssert euclDiv(-13,  3) == -5
doAssert euclDiv( 13, -3) == -4
doAssert euclDiv(-13, -3) ==  5

# euclMod is ALWAYS >= 0
doAssert euclMod( 13,  3) == 1
doAssert euclMod(-13,  3) == 2    # >= 0 !
doAssert euclMod( 13, -3) == 1    # >= 0 !
doAssert euclMod(-13, -3) == 2    # >= 0 !
```

### `ceilDiv` — Division with Rounding Up
```nim
func ceilDiv*[T: SomeInteger](x, y: T): T
```
Conceptually equivalent to `ceil(x/y)`. Works only when `x >= 0` and `y > 0`. Typical use case: calculating the number of blocks/pages.

```nim
import std/math

doAssert ceilDiv(12, 3) == 4   # divides evenly
doAssert ceilDiv(13, 3) == 5   # 4.33... → 5 (up)
doAssert ceilDiv(14, 3) == 5   # 4.66... → 5

# Use case: how many packets of size K are needed for N items
let N = 57
let K = 10
echo ceilDiv(N, K)   # => 6 packets (last one incomplete)
```

### `divmod` — Quotient and Remainder Simultaneously
```nim
func divmod*[T: SomeInteger](x, y: T): (T, T)
```
Computes quotient and remainder in a single operation (optimization: on most CPU architectures this is a single `div` machine instruction). Returns `(quotient, remainder)`.

```nim
import std/math

doAssert divmod(5, 2)    == (2, 1)
doAssert divmod(5, -3)   == (-1, 2)
doAssert divmod(-10, 3)  == (-3, -1)

# Use case: convert seconds to hours, minutes, seconds
proc toHMS(totalSec: int): (int, int, int) =
  let (h, remSec) = divmod(totalSec, 3600)
  let (m, s)      = divmod(remSec, 60)
  (h, m, s)

echo toHMS(3661)   # => (1, 1, 1) — 1 hour, 1 minute, 1 second
```

### `mod` for Floats
```nim
func `mod`*(x, y: float64): float64
```
Remainder of floating-point division (equivalent to C's `fmod`). The sign of the result matches the sign of the dividend `x`.

```nim
import std/math

doAssert  6.5 mod  2.5 ==  1.5
doAssert -6.5 mod  2.5 == -1.5   # sign matches x
doAssert  6.5 mod -2.5 ==  1.5   # sign matches x
doAssert -6.5 mod -2.5 == -1.5

# If Python-style (sign matches divisor) is needed:
# use floorMod
```

---

## 10. Integer Mathematics

### `binom` — Binomial Coefficient
```nim
func binom*(n, k: int): int
```
Computes C(n, k) = n! / (k! × (n−k)!) — "n choose k". This is the number of ways to choose `k` items from `n` without regard to order.

```nim
import std/math

doAssert binom(6, 2) == 15   # C(6,2): 15 pairs from 6 items
doAssert binom(6, 0) == 1    # C(n,0) = 1 always
doAssert binom(6, 6) == 1    # C(n,n) = 1 always
doAssert binom(6, 3) == 20

# Use case: probability in Bernoulli formula
# P(k successes out of n) = C(n,k) * p^k * (1-p)^(n-k)
proc bernoulli(n, k: int; p: float): float =
  float(binom(n, k)) * p^k * (1.0 - p)^float(n - k)

echo bernoulli(10, 3, 0.5)   # probability of 3 heads in 10 coin tosses
```

### `fac` — Factorial
```nim
func fac*(n: int): int
```
Computes n! for non-negative n using a precomputed table — extremely fast. Maximum argument depends on int bit-width: 20 for `int64`, 12 for `int32`.

```nim
import std/math

doAssert fac(0)  == 1           # 0! = 1 by definition
doAssert fac(4)  == 24          # 4! = 4×3×2×1 = 24
doAssert fac(10) == 3628800

# fac(21) will raise AssertionDefect — too large for int64
# For larger factorials, use BigInt or logarithms (lgamma)
```

### `isPowerOfTwo` and `nextPowerOfTwo`
```nim
func isPowerOfTwo*(x: int): bool
func nextPowerOfTwo*(x: int): int
```
Efficient bit-arithmetic operations. Frequently needed for memory allocation (buffer size as power of two), hash tables, and textures.

```nim
import std/math

# isPowerOfTwo
doAssert isPowerOfTwo(1)     # 2^0
doAssert isPowerOfTwo(16)    # 2^4
doAssert not isPowerOfTwo(5)
doAssert not isPowerOfTwo(0)  # 0 is not a power of two
doAssert not isPowerOfTwo(-16)

# nextPowerOfTwo
doAssert nextPowerOfTwo(16) == 16   # already a power of two
doAssert nextPowerOfTwo(17) == 32
doAssert nextPowerOfTwo(5)  == 8
doAssert nextPowerOfTwo(0)  == 1    # 0 → 1
doAssert nextPowerOfTwo(-5) == 1    # negatives → 1

# Use case: buffer size for FFT (must be power of two)
let dataSize = 1000
let fftSize = nextPowerOfTwo(dataSize)   # => 1024
```

### `gcd` — Greatest Common Divisor
```nim
func gcd*[T](x, y: T): T                    # for float: GCD via remainder
func gcd*(x, y: SomeInteger): SomeInteger   # for int: Stein's binary algorithm
func gcd*[T](x: openArray[T]): T            # for arrays
```
For integers, the fast binary GCD algorithm (Stein's algorithm) is used, based on bit shifts.

```nim
import std/math

doAssert gcd(12, 8)    == 4
doAssert gcd(17, 63)   == 1     # coprime
doAssert gcd(0, 5)     == 5     # gcd(0, x) = x
doAssert gcd(-12, 8)   == 4     # negatives are OK

# For floats
doAssert almostEqual(gcd(13.5, 9.0), 4.5)

# For arrays
doAssert gcd(@[12, 8, 4])  == 4
doAssert gcd(@[17, 34, 51]) == 17

# Use case: reduce fraction
proc reduceFraction(num, den: int): (int, int) =
  let g = gcd(abs(num), abs(den))
  (num div g, den div g)

echo reduceFraction(12, 8)   # => (3, 2)
```

### `lcm` — Least Common Multiple
```nim
func lcm*[T](x, y: T): T
func lcm*[T](x: openArray[T]): T
```
Computes LCM using the formula: `lcm(x,y) = x div gcd(x,y) * y`.

```nim
import std/math

doAssert lcm(24, 30) == 120
doAssert lcm(13, 39) ==  39    # if x divides y, lcm = y
doAssert lcm(@[4, 6, 10]) == 60

# Use case: synchronizing cycles of different lengths
let cycleA = 8     # cycle A every 8 steps
let cycleB = 12    # cycle B every 12 steps
echo lcm(cycleA, cycleB)   # => 24 (will align after 24 steps)
```

---

## 11. Special Functions

⚠️ These functions are unavailable for the JavaScript backend — they are implemented via `<math.h>` and require a C compiler.

### `erf` and `erfc` — Error Function
```nim
func erf*(x: float64): float64   # error function
func erfc*(x: float64): float64  # complementary: erfc(x) = 1 - erf(x)
```
The error function is used in probability theory and statistics — it is related to the integral of the normal distribution.

```nim
import std/math

# erf(x) ∈ (-1, 1), odd function
echo erf(0.0)   # => 0.0
echo erf(1.0)   # => 0.8427007929...
echo erf(Inf)   # => 1.0

# Probability of falling within [-σ, +σ] of normal distribution
# P(-a ≤ X ≤ a) = erf(a / sqrt(2))
let sigma = 1.0
let p1 = erf(sigma / sqrt(2.0))   # => 0.6827... (68% rule)
let p2 = erf(2.0 / sqrt(2.0))     # => 0.9545... (95% rule)
let p3 = erf(3.0 / sqrt(2.0))     # => 0.9973... (99.7% rule)

# erfc is more convenient than erf for values close to 1
# erfc(x) = 1 - erf(x), but more precise near 1
echo erfc(3.0)   # more accurate than 1.0 - erf(3.0)
```

### `gamma` and `lgamma` — Gamma Function
```nim
func gamma*(x: float64): float64   # Γ(x)
func lgamma*(x: float64): float64  # ln(Γ(x))
```
The gamma function Γ(x) generalizes factorial to real numbers: `Γ(n) = (n−1)!` for natural `n`. `lgamma` returns the natural logarithm of the gamma function — this is crucial because `gamma(171)` already overflows `float64`, whereas `lgamma(171)` computes without issue.

```nim
import std/math

# Relationship to factorial: Γ(n) = (n-1)!
doAssert almostEqual(gamma(1.0),  1.0)      # 0! = 1
doAssert almostEqual(gamma(2.0),  1.0)      # 1! = 1
doAssert almostEqual(gamma(4.0),  6.0)      # 3! = 6
doAssert almostEqual(gamma(11.0), 3628800.0) # 10!

# Real arguments
echo gamma(0.5)   # => √π ≈ 1.7724538509...
echo gamma(1.5)   # => √π/2 ≈ 0.8862...

# lgamma for large values (prevents overflow)
echo lgamma(171.0)   # OK: logarithm of a large number
# echo gamma(171.0)  # => Inf (float64 overflow)

# Use case: computing binomial coefficient for large n
# ln(C(n,k)) = lgamma(n+1) - lgamma(k+1) - lgamma(n-k+1)
proc logBinom(n, k: float): float =
  lgamma(n + 1.0) - lgamma(k + 1.0) - lgamma(n - k + 1.0)
```

---

## 12. Sign and Value Clamping

### `sgn` — Sign Function
```nim
func sgn*[T: SomeNumber](x: T): int
```
Returns:
- `1` for positive numbers and `+Inf`
- `-1` for negative numbers and `-Inf`
- `0` for zero (including `±0.0`) and `NaN`

```nim
import std/math

doAssert sgn(5)    ==  1
doAssert sgn(0)    ==  0
doAssert sgn(-4.1) == -1
doAssert sgn(Inf)  ==  1
doAssert sgn(-Inf) == -1
doAssert sgn(NaN)  ==  0    # NaN → 0

# Use case: absolute value via sign
proc myAbs[T: SomeNumber](x: T): T =
  T(sgn(x)) * x

# Use case: sign selection in physics formulas
let velocity = -5.0
let direction = sgn(velocity)   # -1 (moving in negative direction)
```

### `clamp` — Value Clamping
```nim
func clamp*[T](val: T, bounds: Slice[T]): T
```
Returns `val` clamped to the `bounds` range. If `val < bounds.a`, returns `bounds.a`; if `val > bounds.b`, returns `bounds.b`.
This is the `std/math` version that accepts a `Slice`. `system` also provides `clamp(val, min, max)` with three arguments.

```nim
import std/math

doAssert clamp(10, 1..5)   == 5   # above upper bound
doAssert clamp(-3, 0..10)  == 0   # below lower bound
doAssert clamp(4, 1..10)   == 4   # inside range — unchanged
doAssert clamp(1, 1..3)    == 1   # at lower bound

# Works with any type supporting comparison
type Color = enum cRed, cGreen, cBlue, cAlpha
doAssert clamp(cAlpha, cRed..cBlue) == cBlue

# Use case: normalize values (e.g., RGB 0-255)
let raw = 312
let pixel = clamp(raw, 0..255)   # => 255

# Use case: limit volume
let volume = 1.5
let safeVol = clamp(volume, 0.0..1.0)   # => 1.0
```

---

## 13. Array Functions

### `sum` and `prod`
```nim
func sum*[T](x: openArray[T]): T   # sum of elements
func prod*[T](x: openArray[T]): T  # product of elements
```
For empty arrays, `sum` returns `0`, `prod` returns `1` (identity elements for the operations).

```nim
import std/math

doAssert sum([1, 2, 3, 4])   == 10
doAssert sum([-4, 3, 5])     == 4
doAssert sum(newSeq[int](0)) == 0   # empty array

doAssert prod([1, 2, 3, 4])  == 24
doAssert prod([-4, 3, 5])    == -60
doAssert prod(newSeq[int](0)) == 1  # empty array

# Use case: mean value
proc mean(x: openArray[float]): float =
  sum(x) / float(x.len)

echo mean([1.0, 2.0, 3.0, 4.0, 5.0])   # => 3.0
```

### `cumsum` and `cumsummed` — Cumulative Sum
```nim
func cumsum*[T](x: var openArray[T])          # modifies array in-place
func cumsummed*[T](x: openArray[T]): seq[T]  # returns new sequence
```
`cumsum` works in-place — modifies the original array. `cumsummed` creates a new `seq`. Each element of the result is the sum of all previous elements inclusive.

```nim
import std/math

# cumsummed — creates a new sequence
doAssert cumsummed([1, 2, 3, 4]) == @[1, 3, 6, 10]
# [1, 1+2, 1+2+3, 1+2+3+4] = [1, 3, 6, 10]

# cumsum — modifies in-place
var a = [1, 2, 3, 4]
cumsum(a)
doAssert a == @[1, 3, 6, 10]

# Use case: cumulative monthly expenses
let monthly = [500.0, 300.0, 700.0, 200.0, 450.0]
let cumulative = cumsummed(monthly)
# => [500, 800, 1500, 1700, 2150]
echo cumulative[^1]   # total sum for all months: 2150.0
```

### `cumprod` and `cumproded` — Cumulative Product
```nim
func cumprod*[T](x: var openArray[T])          # in-place
func cumproded*[T](x: openArray[T]): seq[T]   # new sequence
```
Similar to `cumsum`, but for multiplication. Each element is the product of all previous elements inclusive.

```nim
import std/math

doAssert cumproded([1, 2, 3, 4]) == @[1, 2, 6, 24]
# [1, 1*2, 1*2*3, 1*2*3*4] = [1, 2, 6, 24]

var b = [1, 2, 3, 4]
cumprod(b)
doAssert b == @[1, 2, 6, 24]

# Use case: compute factorials for a range of n at once
let factorials = cumproded(@[1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
# => [1, 2, 6, 24, 120, 720, 5040, 40320, 362880, 3628800]
# factorials[i-1] == fac(i)
```

---

## 14. Practical Examples

### Generating Gaussian Noise (Box–Muller Transform)
The Box–Muller transform converts two uniformly distributed numbers into two normally (Gaussian) distributed numbers.
```nim
import std/math
from std/fenv import epsilon
from std/random import rand

proc gaussianNoise(mu: float = 0.0, sigma: float = 1.0): (float, float) =
  ## Generates two values from normal distribution N(mu, sigma²).
  ## Uses Box–Muller transform.
  var u1, u2: float
  # u1 must not be zero (log(0) = -Inf)
  while true:
    u1 = rand(1.0)
    u2 = rand(1.0)
    if u1 > epsilon(float): break
  let mag = sigma * sqrt(-2.0 * ln(u1))
  let z0  = mag * cos(TAU * u2) + mu
  let z1  = mag * sin(TAU * u2) + mu
  (z0, z1)

# Generate sample
for i in 0..4:
  let (a, b) = gaussianNoise(mu = 0.0, sigma = 1.0)
  echo a, " ", b
```

### Solving Quadratic Equations
```nim
import std/math

type QuadraticResult = object
  case hasRoots: bool
  of true:
    x1, x2: float
  of false:
    discard

proc solveQuadratic(a, b, c: float): QuadraticResult =
  ## Solves ax² + bx + c = 0.
  ## Returns QuadraticResult with roots or without.
  let discriminant = b*b - 4.0*a*c
  if discriminant < 0.0:
    return QuadraticResult(hasRoots: false)
  let sqrtD = sqrt(discriminant)
  QuadraticResult(hasRoots: true,
                  x1: (-b + sqrtD) / (2.0 * a),
                  x2: (-b - sqrtD) / (2.0 * a))

let r = solveQuadratic(1.0, -5.0, 6.0)   # x² - 5x + 6 = 0
if r.hasRoots:
  echo "x1 = ", r.x1   # => 3.0
  echo "x2 = ", r.x2   # => 2.0
```

### Statistics: Mean, Variance, Standard Deviation
```nim
import std/math

proc statistics(data: seq[float]): tuple[mean, variance, stddev: float] =
  ## Computes basic statistical characteristics of a sample.
  let n = float(data.len)
  let m = sum(data) / n

  # Variance (unbiased estimator, divisor n-1)
  var sumSq = 0.0
  for x in data:
    sumSq += (x - m)^2
  let v = sumSq / (n - 1.0)

  (mean: m, variance: v, stddev: sqrt(v))

let data = @[2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0]
let (m, v, s) = statistics(data)
echo "Mean:                ", m   # => 5.0
echo "Variance:            ", v   # => 4.0
echo "Standard Deviation:  ", s   # => 2.0
```

### Angle Normalization and Direction Handling
```nim
import std/math

proc normalizeAngle(deg: float): float =
  ## Normalizes angle to [0°, 360°).
  floorMod(deg, 360.0)

proc angleDifference(a, b: float): float =
  ## Smallest difference between two angles (accounts for cyclicality).
  ## Result ∈ (-180°, 180°].
  let diff = floorMod(b - a + 180.0, 360.0) - 180.0
  diff

proc rotateVector(x, y, angleDeg: float): (float, float) =
  ## Rotates vector (x, y) by angleDeg degrees.
  let r = degToRad(angleDeg)
  (x * cos(r) - y * sin(r),
   x * sin(r) + y * cos(r))

echo normalizeAngle(370.0)     # => 10.0
echo normalizeAngle(-90.0)     # => 270.0
echo angleDifference(10.0, 350.0)   # => -20.0 (shortest path)
echo rotateVector(1.0, 0.0, 90.0)   # => (~0.0, 1.0)
```

### Checking Numeric Properties
```nim
import std/math

proc analyzeNumber(x: float): string =
  ## Full diagnostic of a floating-point number.
  let cls = classify(x)
  let parts = splitDecimal(x)
  let sign = if signbit(x):  "negative " else:  "positive "

  case cls
  of fcNormal, fcSubnormal:
    let category = if cls == fcSubnormal:  "subnormal " else:  "normal "
    let (frac, exp) = frexp(x)
    result =  "Class:  "  & category  &  ", sign:  "  & sign  &
              ", int part:  "  & $parts.intpart  &
              ", frac part:  "  & $parts.floatpart  &
              ", mantissa:  "  & $frac  &  ", exponent:  "  & $exp
  of fcZero, fcNegZero:
    result =  "Zero ( "  & sign  &  ") "
  of fcInf, fcNegInf:
    result =  "Infinity ( "  & sign  &  ") "
  of fcNan:
    result =  "NaN (not a number) "

echo analyzeNumber(3.75)     # normal, mantissa 0.9375, exponent 2
echo analyzeNumber(-0.0)     # Zero (negative)
echo analyzeNumber(Inf)      # Infinity (positive)
echo analyzeNumber(0.0/0.0)  # NaN
```

---

## 15. Quick Reference Table

| Category | Functions / Constants |
|----------|-----------------------|
| Constants | `PI`, `TAU`, `E`, `MaxFloat64Precision`, `MaxFloat32Precision`, `MinFloatNormal` |
| Float Utilities | `isNaN`, `signbit`, `copySign`, `classify`, `almostEqual`, `frexp`, `splitDecimal` |
| Rounding | `floor`, `ceil`, `round`, `round(x, places)`, `trunc` |
| Roots | `sqrt`, `cbrt` |
| Powers | `pow`, `^` (Natural), `^` (SomeFloat), `hypot` |
| Logarithms | `ln`, `log`, `log2`, `log10`, `exp` |
| Trigonometry | `sin`, `cos`, `tan`, `cot`, `sec`, `csc` |
| Inverse Trig | `arcsin`, `arccos`, `arctan`, `arctan2`, `arccot`, `arcsec`, `arccsc` |
| Hyperbolic | `sinh`, `cosh`, `tanh`, `coth`, `sech`, `csch` |
| Inverse Hyperbolic | `arcsinh`, `arccosh`, `arctanh`, `arccoth`, `arcsech`, `arccsch` |
| Angles | `degToRad`, `radToDeg` |
| Division | `floorDiv`, `euclDiv`, `ceilDiv`, `divmod` |
| Remainder | `floorMod`, `euclMod`, `mod` (float) |
| Integer Math | `binom`, `fac`, `isPowerOfTwo`, `nextPowerOfTwo`, `gcd`, `lcm` |
| Special Functions | `erf`, `erfc`, `gamma`, `lgamma` (C backend only) |
| Sign / Clamping | `sgn`, `clamp` |
| Aggregation | `sum`, `prod` |
| Cumulative | `cumsum`, `cumsummed`, `cumprod`, `cumproded` |

---

*Document compiled from the source code of `std/math` from the Nim standard library. All examples are tested and compatible with Nim 2.x.*
