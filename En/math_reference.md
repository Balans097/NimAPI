# `std/math` Module Reference (Nim)

> Basic math routines for Nim.  
> Trigonometric functions operate on **radians**. Use `degToRad` / `radToDeg` for conversion.  
> This module is available for the **JavaScript backend**.

---

## Table of Contents

1. [Constants](#constants)
2. [Types](#types)
3. [Float Classification and Properties](#float-classification-and-properties)
4. [Roots and Powers](#roots-and-powers)
5. [Logarithms and Exponentials](#logarithms-and-exponentials)
6. [Trigonometry](#trigonometry)
7. [Inverse Trigonometric Functions](#inverse-trigonometric-functions)
8. [Hyperbolic Functions](#hyperbolic-functions)
9. [Inverse Hyperbolic Functions](#inverse-hyperbolic-functions)
10. [Derived Trigonometric Functions](#derived-trigonometric-functions)
11. [Rounding and Division](#rounding-and-division)
12. [Special Functions (C backend only)](#special-functions-c-backend-only)
13. [Angle Conversion](#angle-conversion)
14. [Integer Combinatorics](#integer-combinatorics)
15. [Array Aggregate Functions](#array-aggregate-functions)

---

## Constants

| Constant | Value | Description |
|---|---|---|
| `PI` | `3.14159265358979…` | The circle constant π (Ludolph's number) |
| `TAU` | `6.28318530717958…` | TAU = 2 × π |
| `E` | `2.71828182845904…` | Euler's number |
| `MaxFloat64Precision` | `16` | Maximum meaningful decimal digits for `float64` |
| `MaxFloat32Precision` | `8` | Maximum meaningful decimal digits for `float32` |
| `MaxFloatPrecision` | `16` | Alias for `MaxFloat64Precision` |
| `MinFloatNormal` | `2.225e-308` | Smallest normal `float` value (= 2⁻¹⁰²²) |

```nim
import std/math

echo PI          # 3.141592653589793
echo TAU         # 6.283185307179586
echo E           # 2.718281828459045
echo MinFloatNormal  # 2.2250738585072014e-308
```

---

## Types

### `FloatClass`

An enum describing the class of a floating-point value. Returned by `classify`.

| Value | Description |
|---|---|
| `fcNormal` | An ordinary nonzero floating-point value |
| `fcSubnormal` | A subnormal (very small) floating-point value |
| `fcZero` | Positive zero |
| `fcNegZero` | Negative zero (`-0.0`) |
| `fcNan` | Not a Number (NaN) |
| `fcInf` | Positive infinity |
| `fcNegInf` | Negative infinity |

---

## Float Classification and Properties

### `classify(x: float): FloatClass`

Classifies a floating-point value and returns its `FloatClass`.

```nim
import std/math

echo classify(0.3)         # fcNormal
echo classify(0.0)         # fcZero
echo classify(-0.0)        # fcNegZero
echo classify(0.3 / 0.0)   # fcInf
echo classify(-0.3 / 0.0)  # fcNegInf
echo classify(5.0e-324)    # fcSubnormal
echo classify(NaN)         # fcNan
```

---

### `isNaN(x: SomeFloat): bool`

Returns `true` if `x` is NaN. More efficient than `classify(x) == fcNan` and works even with `-ffast-math`.

```nim
import std/math

echo NaN.isNaN          # true
echo Inf.isNaN          # false
echo (3.14).isNaN       # false
echo (0.0 / 0.0).isNaN  # true
```

---

### `signbit(x: SomeFloat): bool`

Returns `true` if `x` has a negative sign bit. Correctly distinguishes `-0.0` from `+0.0`.

```nim
import std/math

echo signbit(0.0)    # false
echo signbit(-0.0)   # true
echo signbit(-5.0)   # true
echo signbit(5.0)    # false
echo signbit(-Inf)   # true
```

---

### `copySign[T: SomeFloat](x, y: T): T`

Returns a value with the **magnitude** of `x` and the **sign** of `y`. Works correctly with NaN, infinity, and zero.

```nim
import std/math

echo copySign(10.0, 1.0)        # 10.0
echo copySign(10.0, -1.0)       # -10.0
echo copySign(-Inf, -0.0)       # -Inf
echo copySign(NaN, 1.0).isNaN   # true
echo copySign(1.0, copySign(NaN, -1.0))  # -1.0
```

---

### `almostEqual[T: SomeFloat](x, y: T; unitsInLastPlace: Natural = 4): bool`

Checks approximate equality of two floats using the [machine epsilon](https://en.wikipedia.org/wiki/Machine_epsilon).

`unitsInLastPlace` sets the tolerance in [ULPs](https://en.wikipedia.org/wiki/Unit_in_the_last_place). A value of `0` requires exact equality.

```nim
import std/math

echo almostEqual(PI, 3.14159265358979)  # true
echo almostEqual(Inf, Inf)              # true
echo almostEqual(NaN, NaN)             # false  (NaN != NaN)
echo almostEqual(1.0, 1.0 + 1e-15)    # true
echo almostEqual(1.0, 1.1)            # false
```

---

### `sgn[T: SomeNumber](x: T): int`

Sign function.

- Returns `1` for positive numbers and `+Inf`.
- Returns `-1` for negative numbers and `-Inf`.
- Returns `0` for `+0.0`, `-0.0`, and `NaN`.

```nim
import std/math

echo sgn(5)     # 1
echo sgn(0)     # 0
echo sgn(-4.1)  # -1
echo sgn(Inf)   # 1
echo sgn(NaN)   # 0
```

---

## Roots and Powers

### `sqrt(x: float32|float64): float32|float64`

Computes the square root of `x`. Returns NaN for negative inputs.

```nim
import std/math

echo sqrt(4.0)    # 2.0
echo sqrt(1.44)   # 1.2
echo sqrt(0.0)    # 0.0
echo sqrt(-1.0)   # NaN
```

---

### `cbrt(x: float32|float64): float32|float64`

Computes the cube root of `x`. Unlike `sqrt`, supports negative inputs.

```nim
import std/math

echo cbrt(8.0)    # 2.0
echo cbrt(2.197)  # ~1.3
echo cbrt(-27.0)  # -3.0
```

---

### `pow(x, y: float32|float64): float32|float64`

Computes `x` raised to the power of `y`. Consider using the `^` operator for integer or simple float exponents.

```nim
import std/math

echo pow(100.0, 1.5)  # 1000.0
echo pow(16.0, 0.5)   # 4.0
echo pow(2.0, 10.0)   # 1024.0
```

---

### `` `^`[T: SomeNumber](x: T, y: Natural): T ``

Raises `x` to a non-negative integer power `y`. Works with any numeric type. Uses fast binary exponentiation.

```nim
import std/math

echo 2 ^ 10    # 1024
echo -3 ^ 2    # 9
echo 2.0 ^ 8   # 256.0
echo -3 ^ 0    # 1
```

---

### `` `^`[T: SomeNumber, U: SomeFloat](x: T, y: U): float ``

Raises `x` to a floating-point power `y`, supporting negative and fractional exponents. Error handling follows the C++ specification.

```nim
import std/math

echo 5.5 ^ 2.2      # ~42.54
echo 1.0 ^ Inf      # 1.0
echo 4.0 ^ (-0.5)   # 0.5
```

---

### `hypot(x, y: float32|float64): float32|float64`

Computes the hypotenuse length `sqrt(x² + y²)` without intermediate overflow or underflow.

```nim
import std/math

echo hypot(3.0, 4.0)   # 5.0
echo hypot(5.0, 12.0)  # 13.0
```

---

### `isPowerOfTwo(x: int): bool`

Returns `true` if `x` is a power of two. Zero and negative numbers return `false`.

```nim
import std/math

echo isPowerOfTwo(16)   # true
echo isPowerOfTwo(5)    # false
echo isPowerOfTwo(0)    # false
echo isPowerOfTwo(-16)  # false
echo isPowerOfTwo(1)    # true
```

---

### `nextPowerOfTwo(x: int): int`

Returns `x` rounded **up** to the nearest power of two. Zero and negative values map to `1`.

```nim
import std/math

echo nextPowerOfTwo(16)   # 16
echo nextPowerOfTwo(5)    # 8
echo nextPowerOfTwo(0)    # 1
echo nextPowerOfTwo(-16)  # 1
echo nextPowerOfTwo(17)   # 32
```

---

## Logarithms and Exponentials

### `ln(x: float32|float64): float32|float64`

Computes the natural logarithm of `x` (base e).

```nim
import std/math

echo ln(exp(4.0))  # 4.0
echo ln(1.0)       # 0.0
echo ln(0.0)       # -Inf
echo ln(-7.0)      # NaN
```

---

### `log[T: SomeFloat](x, base: T): T`

Computes the logarithm of `x` in any base.

```nim
import std/math

echo log(9.0, 3.0)   # 2.0
echo log(0.0, 2.0)   # -Inf
echo log(-7.0, 4.0)  # NaN
echo log(8.0, -2.0)  # NaN
```

---

### `log10(x: float32|float64): float32|float64`

Computes the base-10 logarithm.

```nim
import std/math

echo log10(100.0)   # 2.0
echo log10(0.0)     # -Inf
echo log10(-100.0)  # NaN
echo log10(1000.0)  # 3.0
```

---

### `log2(x: float32|float64): float32|float64`

Computes the base-2 (binary) logarithm.

```nim
import std/math

echo log2(8.0)   # 3.0
echo log2(1.0)   # 0.0
echo log2(0.0)   # -Inf
echo log2(-2.0)  # NaN
```

---

### `exp(x: float32|float64): float32|float64`

Computes the exponential function `e^x`.

```nim
import std/math

echo exp(1.0)   # 2.718281828459045 (== E)
echo exp(0.0)   # 1.0
echo exp(-1.0)  # ~0.3679
```

---

### `frexp[T: float32|float64](x: T): tuple[frac: T, exp: int]`

Splits `x` into a normalized fraction `frac` and an integer exponent `exp` such that `abs(frac) ∈ [0.5, 1.0)` and `x == frac × 2^exp`.

```nim
import std/math

echo frexp(8.0)    # (frac: 0.5, exp: 4)
echo frexp(-8.0)   # (frac: -0.5, exp: 4)
echo frexp(0.0)    # (frac: 0.0, exp: 0)
```

---

### `frexp[T: float32|float64](x: T, exponent: var int): T`

Overload of `frexp` that stores the exponent in a variable and returns the fractional part.

```nim
import std/math

var exp: int
echo frexp(5.0, exp)  # 0.625
echo exp              # 3
```

---

### `splitDecimal[T: float32|float64](x: T): tuple[intpart: T, floatpart: T]`

Breaks `x` into its integer and fractional parts. Both parts carry the same sign as `x`. Analogous to C's `modf`.

```nim
import std/math

echo splitDecimal(5.25)   # (intpart: 5.0, floatpart: 0.25)
echo splitDecimal(-2.73)  # (intpart: -2.0, floatpart: -0.73)
echo splitDecimal(0.0)    # (intpart: 0.0, floatpart: 0.0)
```

---

## Trigonometry

> All functions take arguments in **radians**. Use `degToRad` to convert from degrees.

### `sin(x: float32|float64): float32|float64`

Computes the sine of angle `x` (in radians).

```nim
import std/math

echo sin(PI / 6)          # 0.5
echo sin(degToRad(90.0))  # 1.0
echo sin(0.0)             # 0.0
```

---

### `cos(x: float32|float64): float32|float64`

Computes the cosine of angle `x`.

```nim
import std/math

echo cos(2 * PI)          # 1.0
echo cos(degToRad(60.0))  # 0.5
echo cos(PI)              # -1.0
```

---

### `tan(x: float32|float64): float32|float64`

Computes the tangent of angle `x`.

```nim
import std/math

echo tan(degToRad(45.0))  # 1.0
echo tan(PI / 4)          # 1.0
echo tan(0.0)             # 0.0
```

---

### `cot[T: float32|float64](x: T): T`

Cotangent: `1 / tan(x)`.

```nim
import std/math

echo cot(PI / 4)  # 1.0
```

---

### `sec[T: float32|float64](x: T): T`

Secant: `1 / cos(x)`.

```nim
import std/math

echo sec(0.0)  # 1.0
```

---

### `csc[T: float32|float64](x: T): T`

Cosecant: `1 / sin(x)`.

```nim
import std/math

echo csc(PI / 2)  # 1.0
```

---

## Inverse Trigonometric Functions

### `arcsin(x: float32|float64): float32|float64`

Arc sine of `x`, result in `[-π/2, π/2]`.

```nim
import std/math

echo radToDeg(arcsin(0.0))  # 0.0
echo radToDeg(arcsin(1.0))  # 90.0
echo arcsin(-1.0)           # -1.5707... (-π/2)
```

---

### `arccos(x: float32|float64): float32|float64`

Arc cosine of `x`, result in `[0, π]`.

```nim
import std/math

echo radToDeg(arccos(0.0))  # 90.0
echo radToDeg(arccos(1.0))  # 0.0
```

---

### `arctan(x: float32|float64): float32|float64`

Arc tangent of `x`, result in `(-π/2, π/2)`.

```nim
import std/math

echo arctan(1.0)            # ~0.7854 (π/4)
echo radToDeg(arctan(1.0))  # 45.0
```

---

### `arctan2(y, x: float32|float64): float32|float64`

Two-argument arc tangent of `y/x` accounting for the signs of both arguments (full 4-quadrant arctangent). Result in `(-π, π]`. Handles `x ≈ 0` correctly.

```nim
import std/math

echo radToDeg(arctan2(1.0, 0.0))   # 90.0
echo radToDeg(arctan2(0.0, -1.0))  # 180.0
echo radToDeg(arctan2(-1.0, 0.0))  # -90.0
```

---

### `arccot[T: float32|float64](x: T): T`

Inverse cotangent: `arctan(1/x)`.

### `arcsec[T: float32|float64](x: T): T`

Inverse secant: `arccos(1/x)`.

### `arccsc[T: float32|float64](x: T): T`

Inverse cosecant: `arcsin(1/x)`.

---

## Hyperbolic Functions

### `sinh(x: float32|float64): float32|float64`

Hyperbolic sine.

```nim
import std/math

echo sinh(0.0)  # 0.0
echo sinh(1.0)  # ~1.1752
```

---

### `cosh(x: float32|float64): float32|float64`

Hyperbolic cosine.

```nim
import std/math

echo cosh(0.0)  # 1.0
echo cosh(1.0)  # ~1.5431
```

---

### `tanh(x: float32|float64): float32|float64`

Hyperbolic tangent (range `(-1, 1)`).

```nim
import std/math

echo tanh(0.0)  # 0.0
echo tanh(1.0)  # ~0.7616
```

---

### `coth[T: float32|float64](x: T): T`

Hyperbolic cotangent: `1 / tanh(x)`.

### `sech[T: float32|float64](x: T): T`

Hyperbolic secant: `1 / cosh(x)`.

### `csch[T: float32|float64](x: T): T`

Hyperbolic cosecant: `1 / sinh(x)`.

---

## Inverse Hyperbolic Functions

### `arcsinh(x: float32|float64): float32|float64`

Inverse hyperbolic sine.

### `arccosh(x: float32|float64): float32|float64`

Inverse hyperbolic cosine (defined for `x ≥ 1`).

### `arctanh(x: float32|float64): float32|float64`

Inverse hyperbolic tangent (defined for `|x| < 1`).

### `arccoth[T: float32|float64](x: T): T`

Inverse hyperbolic cotangent: `arctanh(1/x)`.

### `arcsech[T: float32|float64](x: T): T`

Inverse hyperbolic secant: `arccosh(1/x)`.

### `arccsch[T: float32|float64](x: T): T`

Inverse hyperbolic cosecant: `arcsinh(1/x)`.

```nim
import std/math

echo arcsinh(0.0)  # 0.0
echo arccosh(1.0)  # 0.0
echo arctanh(0.5)  # ~0.5493
```

---

## Rounding and Division

### `floor(x: float32|float64): float32|float64`

Rounds `x` down to the largest integer not greater than `x`.

```nim
import std/math

echo floor(2.1)   # 2.0
echo floor(2.9)   # 2.0
echo floor(-3.5)  # -4.0
echo floor(-2.0)  # -2.0
```

---

### `ceil(x: float32|float64): float32|float64`

Rounds `x` up to the smallest integer not smaller than `x`.

```nim
import std/math

echo ceil(2.1)   # 3.0
echo ceil(2.9)   # 3.0
echo ceil(-2.1)  # -2.0
```

---

### `trunc(x: float32|float64): float32|float64`

Truncates `x` toward zero (drops the fractional part).

```nim
import std/math

echo trunc(PI)     # 3.0
echo trunc(-1.85)  # -1.0
echo trunc(2.9)    # 2.0
```

---

### `round(x: float32|float64): float32|float64`

Rounds `x` to the nearest integer, with ties rounding away from zero.

```nim
import std/math

echo round(3.4)  # 3.0
echo round(3.5)  # 4.0
echo round(4.5)  # 5.0
```

---

### `round[T: float32|float64](x: T, places: int): T`

Rounds `x` to `places` decimal digits.

- `places > 0` — decimal places after the point.
- `places = 0` — round to integer (calls `round(x)`).
- `places < 0` — round to powers of ten left of the point.

> ⚠️ **Unreliable** for non-zero `places` due to binary floating-point representation.

```nim
import std/math

echo round(PI, 2)       # 3.14
echo round(PI, 4)       # 3.1416
echo round(537.345, -1) # 540.0
```

---

### `` `mod`(x, y: float32|float64): float32|float64 ``

Floating-point remainder of `x / y`. The sign of the result matches the sign of the **dividend** `x`.

```nim
import std/math

echo  6.5 mod  2.5  #  1.5
echo -6.5 mod  2.5  # -1.5
echo  6.5 mod -2.5  #  1.5
echo -6.5 mod -2.5  # -1.5
```

---

### `floorDiv[T: SomeInteger](x, y: T): T`

Integer division that rounds **down** toward negative infinity (`floor(x/y)`). Differs from `div`, which rounds toward zero.

```nim
import std/math

echo floorDiv( 13,  3)  #  4
echo floorDiv(-13,  3)  # -5  (div would give -4)
echo floorDiv( 13, -3)  # -5
echo floorDiv(-13, -3)  #  4
```

---

### `floorMod[T: SomeNumber](x, y: T): T`

Modulo consistent with `floorDiv`. Behaves like Python's `%` operator — the result always has the same sign as the **divisor** `y`.

```nim
import std/math

echo floorMod( 13,  3)  #  1
echo floorMod(-13,  3)  #  2   (not -1!)
echo floorMod( 13, -3)  # -2
echo floorMod(-13, -3)  # -1
```

---

### `euclDiv[T: SomeInteger](x, y: T): T`

Euclidean division. The remainder is always non-negative.

```nim
import std/math

echo euclDiv(13, 3)    #  4
echo euclDiv(-13, 3)   # -5
echo euclDiv(13, -3)   # -4
echo euclDiv(-13, -3)  #  5
```

---

### `euclMod[T: SomeNumber](x, y: T): T`

Euclidean modulo. **Always non-negative**, regardless of the signs of `x` and `y`.

```nim
import std/math

echo euclMod( 13,  3)  # 1
echo euclMod(-13,  3)  # 2
echo euclMod( 13, -3)  # 1
echo euclMod(-13, -3)  # 2
```

---

### `ceilDiv[T: SomeInteger](x, y: T): T`

Integer division that rounds **up** toward positive infinity (`ceil(x/y)`). Requires `x ≥ 0` and `y > 0`.

```nim
import std/math

echo ceilDiv(12, 3)  # 4
echo ceilDiv(13, 3)  # 5
echo ceilDiv(1, 3)   # 1
```

---

### `divmod[T: SomeInteger](x, y: T): (T, T)`

Computes both the quotient and remainder in a single operation. Returns `(quotient, remainder)`.

```nim
import std/math

echo divmod(5, 2)    # (2, 1)
echo divmod(5, -3)   # (-1, 2)
echo divmod(-10, 3)  # (-3, -1)
```

---

### `clamp[T](val: T, bounds: Slice[T]): T`

Clamps `val` to the range `bounds`, expressed as a Nim slice. An assertion is raised if `bounds.a > bounds.b`.

```nim
import std/math

echo clamp(10, 1..5)  # 5
echo clamp(3, 1..5)   # 3
echo clamp(0, 1..5)   # 1
```

---

## Special Functions (C backend only)

> ⚠️ The following functions are **not available on the JavaScript backend**.

### `erf(x: float32|float64): float32|float64`

Computes the [error function](https://en.wikipedia.org/wiki/Error_function).

```nim
import std/math

echo erf(0.0)  # 0.0
echo erf(1.0)  # ~0.8427
echo erf(Inf)  # 1.0
```

---

### `erfc(x: float32|float64): float32|float64`

Computes the complementary error function: `erfc(x) = 1 - erf(x)`.

```nim
import std/math

echo erfc(0.0)  # 1.0
echo erfc(1.0)  # ~0.1573
```

---

### `gamma(x: float32|float64): float32|float64`

Computes the [gamma function](https://en.wikipedia.org/wiki/Gamma_function). Generalizes the factorial: `gamma(n) == fac(n-1)` for positive integers.

```nim
import std/math

echo gamma(1.0)   # 1.0
echo gamma(4.0)   # 6.0        (= 3!)
echo gamma(11.0)  # 3628800.0  (= 10!)
echo gamma(0.5)   # ~1.7725    (= sqrt(PI))
```

---

### `lgamma(x: float32|float64): float32|float64`

Computes the natural logarithm of the gamma function `ln(gamma(x))`. Useful to avoid overflow for large inputs.

```nim
import std/math

echo lgamma(1.0)   # 0.0
echo lgamma(11.0)  # ~15.104  (= ln(3628800))
```

---

## Angle Conversion

### `degToRad[T: float32|float64](d: T): T`

Converts degrees to radians.

```nim
import std/math

echo degToRad(180.0)  # ~3.14159 (== PI)
echo degToRad(90.0)   # ~1.5708  (== PI/2)
echo degToRad(360.0)  # ~6.2832  (== TAU)
```

---

### `radToDeg[T: float32|float64](d: T): T`

Converts radians to degrees.

```nim
import std/math

echo radToDeg(PI)      # 180.0
echo radToDeg(2 * PI)  # 360.0
echo radToDeg(PI / 4)  # 45.0
```

---

## Integer Combinatorics

### `binom(n, k: int): int`

Computes the [binomial coefficient](https://en.wikipedia.org/wiki/Binomial_coefficient) C(n, k).

```nim
import std/math

echo binom(6, 2)   # 15
echo binom(6, 0)   # 1
echo binom(-6, 2)  # 1
echo binom(10, 3)  # 120
```

---

### `fac(n: int): int`

Computes the [factorial](https://en.wikipedia.org/wiki/Factorial) `n!`. The valid range is limited by the size of `int` (up to 20 on 64-bit systems).

```nim
import std/math

echo fac(0)   # 1
echo fac(4)   # 24
echo fac(10)  # 3628800
echo fac(20)  # 2432902008176640000
```

---

### `gcd[T](x, y: T): T`

Computes the greatest common divisor of `x` and `y`. For integers, uses the binary GCD (Stein's) algorithm. For floats, the result may not always have a simple decimal interpretation.

```nim
import std/math

echo gcd(12, 8)      # 4
echo gcd(17, 63)     # 1
echo gcd(13.5, 9.0)  # 4.5
```

---

### `gcd[T](x: openArray[T]): T`

Computes the GCD of all elements of `x`.

```nim
import std/math

echo gcd(@[12, 8, 4])    # 4
echo gcd(@[13.5, 9.0])   # 4.5
```

---

### `lcm[T](x, y: T): T`

Computes the least common multiple of `x` and `y`.

```nim
import std/math

echo lcm(24, 30)  # 120
echo lcm(13, 39)  # 39
echo lcm(4, 6)    # 12
```

---

### `lcm[T](x: openArray[T]): T`

Computes the LCM of all elements of `x`.

```nim
import std/math

echo lcm(@[24, 30])    # 120
echo lcm(@[4, 6, 10])  # 60
```

---

## Array Aggregate Functions

### `sum[T](x: openArray[T]): T`

Computes the sum of elements. Returns `0` for an empty array.

```nim
import std/math

echo sum([1, 2, 3, 4])    # 10
echo sum([-4, 3, 5])      # 4
echo sum([0.5, 1.5])      # 2.0
echo sum(newSeq[int]())   # 0
```

---

### `prod[T](x: openArray[T]): T`

Computes the product of elements. Returns `1` for an empty array.

```nim
import std/math

echo prod([1, 2, 3, 4])  # 24
echo prod([-4, 3, 5])    # -60
echo prod([2, 2, 2])     # 8
```

---

### `cumsummed[T](x: openArray[T]): seq[T]`

Returns a new sequence of cumulative (prefix) sums. Returns `@[]` for an empty input.

```nim
import std/math

echo cumsummed([1, 2, 3, 4])  # @[1, 3, 6, 10]
echo cumsummed([5, -1, 3])    # @[5, 4, 7]
```

---

### `cumsum[T](x: var openArray[T])`

Transforms `x` **in-place** into its cumulative sum. The array must be declared as `var`.

```nim
import std/math

var a = [1, 2, 3, 4]
cumsum(a)
echo a  # @[1, 3, 6, 10]
```

---

### `cumproded[T](x: openArray[T]): seq[T]`

Returns a new sequence of cumulative (prefix) products.

```nim
import std/math

echo cumproded([1, 2, 3, 4])  # @[1, 2, 6, 24]
```

---

### `cumprod[T](x: var openArray[T])`

Transforms `x` **in-place** into its cumulative product.

```nim
import std/math

var a = [1, 2, 3, 4]
cumprod(a)
echo a  # @[1, 2, 6, 24]
```

---

## Complete Examples

### Gaussian noise via Box–Muller transform

```nim
import std/math
import std/[random, fenv]

proc gaussianNoise(mu = 0.0, sigma = 1.0): (float, float) =
  var u1, u2: float
  while true:
    u1 = rand(1.0)
    u2 = rand(1.0)
    if u1 > epsilon(float): break
  let mag = sigma * sqrt(-2.0 * ln(u1))
  (mag * cos(2 * PI * u2) + mu,
   mag * sin(2 * PI * u2) + mu)

randomize()
echo gaussianNoise()
```

---

### Solving a right triangle

```nim
import std/math

let a = 3.0
let b = 4.0
let c = hypot(a, b)
echo "Hypotenuse: ", c             # 5.0

let angle_A = radToDeg(arctan2(a, b))
echo "Angle A: ", angle_A, "°"    # ~36.87°
```

---

### Comparing division modes

```nim
import std/math

let x = -13
let y = 3

echo "div (toward zero):    ", x div y         # -4
echo "floorDiv (down):      ", floorDiv(x, y)  # -5
echo "euclDiv (euclidean):  ", euclDiv(x, y)   # -5
echo "ceilDiv (up, x>=0):   ", ceilDiv(13, y)  #  5

echo "mod (toward zero):    ", x mod y         # -1
echo "floorMod (sign of y): ", floorMod(x, y)  #  2
echo "euclMod (always >=0): ", euclMod(x, y)   #  2
```

---

### Prefix sums for range queries

```nim
import std/math

let data = [10, 5, 8, 3, 7]
let prefix = cumsummed(data)
echo prefix  # @[10, 15, 23, 26, 33]

# Sum of elements in range [1..3] using prefix sums:
let rangeSum = prefix[3] - prefix[0]
echo rangeSum  # 16  (= 5 + 8 + 3)
```

---

### Float classification guard

```nim
import std/math

proc safeLog(x: float): float =
  case classify(x)
  of fcNormal, fcSubnormal:
    if x > 0: ln(x)
    else: NaN
  of fcZero, fcNegZero: -Inf
  of fcInf: Inf
  of fcNegInf, fcNan: NaN

echo safeLog(100.0)  # ~4.605
echo safeLog(-5.0)   # NaN
echo safeLog(0.0)    # -Inf
```
