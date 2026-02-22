# `complex` Module Reference

> **Module:** `complex`  
> **Source:** Nim's Runtime Library.
> **Purpose:** Complex number type, arithmetic operators, transcendental functions, polar/rectangular conversions, and string formatting.

---

## Overview

A **complex number** is a pair `(re, im)` where `re` is the real part and `im` is the imaginary part. Together they describe a point — or equivalently a vector — in the 2D complex plane. This module gives Nim first-class support for that mathematical object: a generic `Complex[T]` type that works with both `float32` and `float64`, a full set of arithmetic operators that mix freely with plain real numbers, the usual transcendental functions extended to the complex domain (trigonometric, hyperbolic, logarithmic, exponential, power), and conversions between rectangular `(re, im)` and polar `(r, φ)` representations.

The implementation is entirely in `func` (no side effects). Internal checks are disabled for performance.

---

## Types

### `Complex[T]`

```nim
type Complex*[T: SomeFloat] = object
  re*, im*: T
```

The core type. `T` must be `float32` or `float64` (constrained by `SomeFloat`). Both fields are public, so you can read and write `z.re` and `z.im` directly.

### `Complex64`

```nim
type Complex64* = Complex[float64]
```

Convenience alias for the most common case — 64-bit precision.

### `Complex32`

```nim
type Complex32* = Complex[float32]
```

Alias for 32-bit precision, useful in graphics or signal-processing code where memory footprint matters.

---

## Exported Symbols at a Glance

| Symbol | Kind | One-line summary |
|--------|------|-----------------|
| `complex` | func | Generic constructor `(re, im)` |
| `complex32` | func | Constructor for `Complex32` |
| `complex64` | func | Constructor for `Complex64` |
| `im` | template ×4 | Imaginary unit or imaginary number from a float |
| `==` | func | Exact equality |
| `+`, `-`, `*`, `/` | func ×3 each | Arithmetic: complex/complex, real/complex, complex/real |
| `+=`, `-=`, `*=`, `/=` | func | In-place arithmetic |
| `abs` | func | Modulus (distance from origin) |
| `abs2` | func | Squared modulus (faster than `abs^2`) |
| `sgn` | func | Unit-length phase vector (or 0) |
| `conjugate` | func | Complex conjugate |
| `inv` | func | Multiplicative inverse (1/z) |
| `sqrt` | func | Principal square root |
| `exp` | func | Exponential `e^z` |
| `ln` | func | Natural logarithm (principal value) |
| `log10` | func | Base-10 logarithm |
| `log2` | func | Base-2 logarithm |
| `pow` | func ×2 | Power: complex^complex, complex^real |
| `sin`, `cos`, `tan` | func | Trigonometric functions |
| `arcsin`, `arccos`, `arctan` | func | Inverse trigonometric |
| `cot`, `sec`, `csc` | func | Cotangent, secant, cosecant |
| `arccot`, `arcsec`, `arccsc` | func | Inverse cot/sec/csc |
| `sinh`, `cosh`, `tanh` | func | Hyperbolic functions |
| `arcsinh`, `arccosh`, `arctanh` | func | Inverse hyperbolic |
| `coth`, `sech`, `csch` | func | Hyperbolic cot/sec/csc |
| `arccoth`, `arcsech`, `arccsch` | func | Inverse hyperbolic cot/sec/csc |
| `phase` | func | Angle in polar form (arctan2) |
| `polar` | func | `(r, phi)` polar coordinates |
| `rect` | func | Complex from polar `(r, phi)` |
| `almostEqual` | func | Near-equality with ULP tolerance |
| `$` | func | String `"(re, im)"` |
| `formatValue` | proc | `strformat` integration (`&` macro support) |

---

## Constructors

### `complex`

```nim
func complex*[T: SomeFloat](re: T; im: T = 0.0): Complex[T]
```

The generic constructor. `T` is inferred from `re`. The imaginary part defaults to `0.0`, so `complex(3.0)` creates the purely real number `3 + 0i`.

```nim
let z1 = complex(1.0, 2.0)          # 1 + 2i  (float64)
let z2 = complex(3.0)               # 3 + 0i
let z3 = complex(0.5'f32, 1.0'f32)  # float32 version
```

---

### `complex32` and `complex64`

```nim
func complex32*(re: float32; im: float32 = 0.0): Complex32
func complex64*(re: float64; im: float64 = 0.0): Complex64
```

Typed versions for when precision must be explicit — particularly useful when the generic `T` cannot be inferred automatically.

```nim
let a = complex32(1.5, -0.5)   # explicit float32
let b = complex64(1.5, -0.5)   # explicit float64
```

---

### `im` — Imaginary unit and imaginary numbers

```nim
template im*(arg: typedesc[float32]): Complex32   # imaginary unit i (32-bit)
template im*(arg: typedesc[float64]): Complex64   # imaginary unit i (64-bit)
template im*(arg: float32): Complex32             # arg * i  (32-bit)
template im*(arg: float64): Complex64             # arg * i  (64-bit)
```

Four overloads let you write complex literals naturally. Passing a **type** gives the pure imaginary unit `0 + 1i`. Passing a **value** gives `0 + arg*i`.

```nim
let i64 = im(float64)      # 0 + 1i  (Complex64)
let z   = 3.0 + 4.0.im    # 3 + 4i  — natural algebraic notation
let z2  = complex(2.0) + im(float64)   # 2 + 1i
```

The value-taking overloads make `a + b.im` read almost like mathematical notation `a + bi`.

---

## Arithmetic Operators

All operators are generic over `[T: SomeFloat]`. Each of `+`, `-`, `*`, `/` comes in three overloads: `complex op complex`, `real op complex`, and `complex op real`, so you never need to wrap a plain float before mixing it with a complex number.

### `==`

```nim
func `==`*[T](x, y: Complex[T]): bool
```

Exact bitwise equality of both components. For results of floating-point calculations, prefer `almostEqual`.

```nim
assert complex(1.0, 2.0) == complex(1.0, 2.0)
assert not (complex(1.0, 2.0) == complex(1.0, 2.000001))
```

---

### `+`  — Addition

```nim
func `+`*[T](x: T; y: Complex[T]): Complex[T]      # real + complex
func `+`*[T](x: Complex[T]; y: T): Complex[T]      # complex + real
func `+`*[T](x, y: Complex[T]): Complex[T]         # complex + complex
```

Component-wise: `(a+bi) + (c+di) = (a+c) + (b+d)i`. Adding a real number only shifts the real component.

```nim
let z1 = complex(1.0, 2.0)
let z2 = complex(3.0, -4.0)
assert z1 + z2 == complex(4.0, -2.0)
assert z1 + 10.0 == complex(11.0, 2.0)
assert 10.0 + z1 == complex(11.0, 2.0)
```

---

### `-`  — Subtraction and Negation

```nim
func `-`*[T](z: Complex[T]): Complex[T]            # unary: -(a+bi) = -a-bi
func `-`*[T](x: T; y: Complex[T]): Complex[T]
func `-`*[T](x: Complex[T]; y: T): Complex[T]
func `-`*[T](x, y: Complex[T]): Complex[T]
```

```nim
let z = complex(3.0, -4.0)
assert -z == complex(-3.0, 4.0)                     # unary minus
assert complex(1.0, 2.0) - z == complex(-2.0, 6.0)
assert 10.0 - z == complex(7.0, 4.0)               # real - complex: re=10-3, im=0-(-4)
```

---

### `*`  — Multiplication

```nim
func `*`*[T](x: T; y: Complex[T]): Complex[T]
func `*`*[T](x: Complex[T]; y: T): Complex[T]
func `*`*[T](x, y: Complex[T]): Complex[T]
```

Complex multiplication uses `(ac−bd) + (bc+ad)i`:

```nim
let z1 = complex(1.0, 2.0)
let z2 = complex(3.0, -4.0)
# (1·3 − 2·(-4)) + (2·3 + 1·(-4))i = 11 + 2i
assert z1 * z2 == complex(11.0, 2.0)
assert 2.0 * z1 == complex(2.0, 4.0)   # scales both components
```

---

### `/`  — Division

```nim
func `/`*[T](x: Complex[T]; y: T): Complex[T]
func `/`*[T](x: T; y: Complex[T]): Complex[T]
func `/`*[T](x, y: Complex[T]): Complex[T]
```

Implemented as `x * conjugate(y) / abs2(y)` — numerically stable.

```nim
from std/math import almostEqual
let z1 = complex(1.0, 2.0)
let z2 = complex(3.0, -4.0)
assert almostEqual(z1 / z2, complex(-0.2, 0.4))
assert z1 / 2.0 == complex(0.5, 1.0)
```

---

### In-place operators: `+=`, `-=`, `*=`, `/=`

```nim
func `+=`*[T](x: var Complex[T]; y: Complex[T])
func `-=`*[T](x: var Complex[T]; y: Complex[T])
func `*=`*[T](x: var Complex[T]; y: Complex[T])
func `/=`*[T](x: var Complex[T]; y: Complex[T])
```

`*=` uses a temporary to avoid clobbering the real part before computing the imaginary part.

```nim
var z = complex(1.0, 2.0)
z += complex(0.0, 1.0)   # z = 1 + 3i
z *= complex(2.0, 0.0)   # z = 2 + 6i
z -= complex(1.0, 1.0)   # z = 1 + 5i
z /= complex(0.0, 1.0)   # z = 5 - 1i
```

---

## Magnitude and Direction

### `abs`

```nim
func abs*[T](z: Complex[T]): T
```

Returns the **modulus** — the Euclidean distance from the origin `√(re² + im²)`. Computed via `hypot` for numerical safety.

```nim
from std/math import almostEqual
let z = complex(3.0, 4.0)
assert almostEqual(abs(z), 5.0)   # 3-4-5 right triangle
assert abs(im(float64)) == 1.0    # imaginary unit lies on the unit circle
```

---

### `abs2`

```nim
func abs2*[T](z: Complex[T]): T
```

Returns the **squared modulus** `re² + im²`, without taking a square root. More efficient than `abs(z) * abs(z)` when you only need the squared value — for normalisation checks or magnitude comparisons.

```nim
let z = complex(3.0, 4.0)
assert abs2(z) == 25.0   # 9 + 16

# Faster than computing abs twice:
let big = complex(1000.0, 1000.0)
if abs2(big) > 100.0 * 100.0:
  echo "outside unit disk of radius 100"
```

---

### `sgn`

```nim
func sgn*[T](z: Complex[T]): Complex[T]
```

Returns the **unit complex number** pointing in the same direction as `z` — i.e. `z / |z|`. If `z == 0`, returns zero safely.

Geometrically, `sgn(z)` is the projection of `z` onto the unit circle, retaining only its direction.

```nim
from std/math import almostEqual
let z = complex(3.0, 4.0)
let u = sgn(z)
assert almostEqual(abs(u), 1.0)
assert almostEqual(u, complex(0.6, 0.8))   # (3/5, 4/5)

assert sgn(complex(0.0, 0.0)) == complex(0.0, 0.0)
```

---

### `conjugate`

```nim
func conjugate*[T](z: Complex[T]): Complex[T]
```

Flips the sign of the imaginary part: `conjugate(a+bi) = a-bi`. Geometrically this reflects the point across the real axis.

Key property: `z * conjugate(z) = abs2(z)` — always a non-negative real number. This is the basis of complex division.

```nim
let z = complex(1.0, 2.0)
assert conjugate(z) == complex(1.0, -2.0)

let prod = z * conjugate(z)
assert prod.im == 0.0         # no imaginary part
assert prod.re == abs2(z)     # = 1 + 4 = 5
```

---

### `inv`

```nim
func inv*[T](z: Complex[T]): Complex[T]
```

Returns the **multiplicative inverse** `1/z = conjugate(z) / abs2(z)`.

```nim
from std/math import almostEqual
let z  = complex(1.0, 2.0)
assert almostEqual(inv(z) * z, complex(1.0, 0.0))   # z*(1/z) = 1

let i  = im(float64)
assert inv(i) == complex(0.0, -1.0)   # 1/i = -i
```

---

## Transcendental Functions

### `sqrt`

```nim
func sqrt*[T](z: Complex[T]): Complex[T]
```

The **principal square root**: the result has non-negative real part (and non-negative imaginary part when the real part is zero). The algorithm is numerically stable.

```nim
from std/math import almostEqual
let i = im(float64)

assert almostEqual(sqrt(complex(-1.0, 0.0)), i)              # √(-1) = i
assert almostEqual(sqrt(complex(0.0, 2.0)), complex(1.0, 1.0)) # √(2i) = 1+i

let z = complex(3.0, 4.0)
assert almostEqual(sqrt(z) * sqrt(z), z)   # round-trip
```

---

### `exp`

```nim
func exp*[T](z: Complex[T]): Complex[T]
```

`e^z` via Euler's formula: `e^(a+bi) = e^a · (cos b + i·sin b)`. The real part controls the magnitude; the imaginary part controls the angle.

```nim
from std/math import almostEqual, E, PI

# Euler's identity: e^(iπ) = -1
let euler = exp(complex(0.0, PI))
assert almostEqual(euler.re, -1.0)
assert almostEqual(euler.im,  0.0)

# Pure real exponent works like regular exp:
assert almostEqual(exp(complex(1.0, 0.0)).re, E)
```

---

### `ln`

```nim
func ln*[T](z: Complex[T]): Complex[T]
```

**Principal value** of the natural logarithm: `ln(z) = ln(|z|) + i·arctan2(im, re)`. The imaginary part (phase) lies in `(-π, π]`.

```nim
from std/math import almostEqual, PI

assert almostEqual(ln(exp(complex(1.0, 0.0))), complex(1.0, 0.0))

# ln(-1) = iπ  (principal value)
let lnNeg1 = ln(complex(-1.0, 0.0))
assert almostEqual(lnNeg1.re, 0.0)
assert almostEqual(lnNeg1.im, PI)
```

---

### `log10` and `log2`

```nim
func log10*[T](z: Complex[T]): Complex[T]
func log2*[T](z: Complex[T]): Complex[T]
```

Base-10 and base-2 logarithms computed as `ln(z) / ln(base)`. Same branch-cut behaviour as `ln`.

```nim
from std/math import almostEqual
assert almostEqual(log10(complex(100.0, 0.0)), complex(2.0, 0.0))
assert almostEqual(log2(complex(8.0, 0.0)),    complex(3.0, 0.0))
```

---

### `pow`

```nim
func pow*[T](x, y: Complex[T]): Complex[T]    # x^y, both complex
func pow*[T](x: Complex[T]; y: T): Complex[T] # x^y, y is real
```

General complex power with several fast paths: `0^0=1`, `x^1=x`, `x^(-1)=inv(x)`, `x^2=x*x`, `x^0.5=sqrt(x)`, real base with real exponent falls back to scalar `pow`, `e^y` uses `exp(y)`.

```nim
from std/math import almostEqual
assert almostEqual(pow(complex(2.0, 0.0), complex(3.0, 0.0)), complex(8.0, 0.0))

# i^i = e^(-π/2) ≈ 0.20788 — famously a real number:
assert almostEqual(pow(im(float64), im(float64)), complex(0.20788, 0.0), 1000)

assert almostEqual(pow(complex(4.0, 0.0), 0.5), complex(2.0, 0.0))  # real exponent overload
```

---

## Trigonometric Functions

All circular functions extend to the complex plane via standard analytic continuation. They reduce to their real counterparts when `z.im == 0`.

### `sin` and `arcsin`

```nim
func sin*[T](z: Complex[T]): Complex[T]
func arcsin*[T](z: Complex[T]): Complex[T]
```

`sin(a+bi) = sin(a)·cosh(b) + i·cos(a)·sinh(b)`

```nim
from std/math import almostEqual, PI
assert almostEqual(sin(complex(PI/2, 0.0)), complex(1.0, 0.0))   # sin(π/2) = 1

let z = complex(1.0, 1.0)
assert almostEqual(arcsin(sin(z)), z)   # round-trip
```

---

### `cos` and `arccos`

```nim
func cos*[T](z: Complex[T]): Complex[T]
func arccos*[T](z: Complex[T]): Complex[T]
```

`cos(a+bi) = cos(a)·cosh(b) − i·sin(a)·sinh(b)`

```nim
from std/math import almostEqual
assert almostEqual(cos(complex(0.0, 0.0)), complex(1.0, 0.0))
let z = complex(1.0, 1.0)
assert almostEqual(arccos(cos(z)), z)
```

---

### `tan`, `arctan`, `cot`, `arccot`, `sec`, `arcsec`, `csc`, `arccsc`

```nim
func tan*[T](z: Complex[T]): Complex[T]      # sin(z)/cos(z)
func arctan*[T](z: Complex[T]): Complex[T]
func cot*[T](z: Complex[T]): Complex[T]      # cos(z)/sin(z)
func arccot*[T](z: Complex[T]): Complex[T]
func sec*[T](z: Complex[T]): Complex[T]      # 1/cos(z)
func arcsec*[T](z: Complex[T]): Complex[T]
func csc*[T](z: Complex[T]): Complex[T]      # 1/sin(z)
func arccsc*[T](z: Complex[T]): Complex[T]
```

The complete set of circular functions and their inverses, all derived from `sin`/`cos` via standard identities. Inverse functions use complex logarithm formulae.

```nim
from std/math import almostEqual
let z = complex(0.5, 0.5)
assert almostEqual(arctan(tan(z)), z)
assert almostEqual(arccot(cot(z)), z)
assert almostEqual(arcsec(sec(z)), z)
assert almostEqual(arccsc(csc(z)), z)
```

---

## Hyperbolic Functions

### `sinh`, `cosh`, `tanh`, `coth`, `sech`, `csch`

```nim
func sinh*[T](z: Complex[T]): Complex[T]    # (exp(z) - exp(-z)) / 2
func cosh*[T](z: Complex[T]): Complex[T]    # (exp(z) + exp(-z)) / 2
func tanh*[T](z: Complex[T]): Complex[T]    # sinh(z)/cosh(z)
func coth*[T](z: Complex[T]): Complex[T]    # cosh(z)/sinh(z)
func sech*[T](z: Complex[T]): Complex[T]    # 2/(exp(z)+exp(-z))
func csch*[T](z: Complex[T]): Complex[T]    # 2/(exp(z)-exp(-z))
```

Connected to circular functions via `sinh(iz) = i·sin(z)` and `cosh(iz) = cos(z)`.

```nim
from std/math import almostEqual
let z = complex(1.0, 1.0)
# Pythagorean identity: cosh² - sinh² = 1
assert almostEqual(cosh(z)*cosh(z) - sinh(z)*sinh(z), complex(1.0, 0.0))
```

---

### `arcsinh`, `arccosh`, `arctanh`, `arccoth`, `arcsech`, `arccsch`

```nim
func arcsinh*[T](z: Complex[T]): Complex[T]
func arccosh*[T](z: Complex[T]): Complex[T]
func arctanh*[T](z: Complex[T]): Complex[T]
func arccoth*[T](z: Complex[T]): Complex[T]
func arcsech*[T](z: Complex[T]): Complex[T]
func arccsch*[T](z: Complex[T]): Complex[T]
```

All implemented via the complex logarithm, extending the real inverse hyperbolic functions to the full complex plane.

```nim
from std/math import almostEqual
let z = complex(0.5, 0.5)
assert almostEqual(arcsinh(sinh(z)), z)
assert almostEqual(arctanh(tanh(z)), z)
```

---

## Polar Coordinates

Complex numbers have two equivalent representations. **Rectangular** `(re, im)` — a point in the plane. **Polar** `(r, φ)` — a distance from the origin and an angle from the positive real axis.

### `phase`

```nim
func phase*[T](z: Complex[T]): T
```

Returns the **argument** (angle) of `z` in radians, via `arctan2(z.im, z.re)`. Result is in `(-π, π]`.

```nim
from std/math import almostEqual, PI
assert phase(complex(1.0, 0.0))  == 0.0         # positive real axis
assert almostEqual(phase(complex(0.0, 1.0)), PI/2)   # 90°
assert almostEqual(phase(complex(-1.0, 0.0)), PI)    # 180°
assert almostEqual(phase(complex(1.0, 1.0)), PI/4)   # 45°
```

---

### `polar`

```nim
func polar*[T](z: Complex[T]): tuple[r, phi: T]
```

Converts to polar form, returning `(r: abs(z), phi: phase(z))`. Inverse is `rect`.

```nim
from std/math import almostEqual, sqrt, PI
let z = complex(1.0, 1.0)
let (r, phi) = z.polar
assert almostEqual(r, sqrt(2.0))
assert almostEqual(phi, PI/4)
```

---

### `rect`

```nim
func rect*[T](r, phi: T): Complex[T]
```

Constructs a complex number from polar coordinates: `re = r·cos(φ)`, `im = r·sin(φ)`. Inverse of `polar`.

```nim
from std/math import almostEqual, PI

# Round-trip:
let z = complex(3.0, 4.0)
let (r, phi) = z.polar
assert almostEqual(rect(r, phi), z)

# Unit vector at 90°:
assert almostEqual(rect(1.0, PI/2), complex(0.0, 1.0))
```

---

## Comparison

### `almostEqual`

```nim
func almostEqual*[T: SomeFloat](x, y: Complex[T]; unitsInLastPlace: Natural = 4): bool
```

Checks whether two complex numbers are **approximately equal**, comparing `re` and `im` parts separately using ULP (Units in the Last Place) tolerance. Default `unitsInLastPlace = 4` suits most floating-point arithmetic. Use this instead of `==` after any chain of computations.

`unitsInLastPlace = 0` means strict equality; larger values allow more accumulated error.

```nim
from std/math import almostEqual

let z1 = complex(1.0, 2.0)
let z2 = complex(3.0, -4.0)

assert almostEqual(z1 + z2, complex(4.0, -2.0))
assert almostEqual(z1 * z2, complex(11.0, 2.0))
assert almostEqual(z1 / z2, complex(-0.2, 0.4))

# Tighter tolerance:
assert almostEqual(z1, z1, unitsInLastPlace = 0)   # exact match
```

---

## String Conversion

### `$`

```nim
func `$`*(z: Complex): string
```

Returns the string `"(re, im)"` using Nim's default float formatting.

```nim
echo complex(1.0, 2.0)    # → (1.0, 2.0)
echo complex(0.0, -1.0)   # → (0.0, -1.0)
```

---

### `formatValue`

```nim
proc formatValue*(result: var string; value: Complex; specifier: string)
```

Integrates `Complex` with Nim's `strformat` `&` macro. Two modes:

- **No `j` in specifier:** formats as `"(re, im)"` tuple, applying the numeric format to each component.
- **`j` in specifier:** formats as `"(A+Bj)"` (Python/engineering notation). The `j` is replaced internally by `g` (or your chosen format type) before being passed to the float formatter.

```nim
import std/strformat

let z = complex(1.5, -2.75)

echo &"{z}"        # → (1.5, -2.75)
echo &"{z:.2f}"    # → (1.50, -2.75)
echo &"{z:j}"      # → (1.5-2.75j)
echo &"{z:.3fj}"   # → (1.500-2.750j)
echo &"{z:+.2fj}"  # → (+1.50-2.75j)
```

---

## Common Patterns

### Rotating a 2D point

Multiplying by a unit complex number at angle `φ` rotates any complex number by `φ`:

```nim
from std/math import PI, almostEqual
let point    = complex(1.0, 0.0)    # on real axis
let rotation = rect(1.0, PI/2)      # 90° rotation
let rotated  = point * rotation
assert almostEqual(rotated, complex(0.0, 1.0))
```

### Verifying i² = −1

```nim
from std/math import almostEqual
let i = im(float64)
assert almostEqual(i * i, complex(-1.0, 0.0))
```

### DFT twiddle factor

```nim
from std/math import PI
proc twiddleFactor(k, n: int): Complex64 =
  rect(1.0, -2.0 * PI * k.float / n.float)
```

### Accumulating in place

```nim
var sum = complex(0.0, 0.0)
for z in myComplexSeq:
  sum += z
```

---

## Error Reference

| Situation | Behaviour | Notes |
|-----------|-----------|-------|
| Division by zero (`z / complex(0,0)`) | `(Inf, NaN)` or similar | IEEE 754 float semantics, no exception |
| `sqrt(complex(0,0))` | Returns `complex(0,0)` | Explicit zero guard in implementation |
| `sgn(complex(0,0))` | Returns `complex(0,0)` | Safe: zero input → zero output |
| `T` outside `SomeFloat` | Compile-time error | Only `float32`/`float64` accepted |
| `almostEqual` with `unitsInLastPlace=0` | Exact equality | Equivalent to `==` |
