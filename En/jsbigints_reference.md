# `jsbigints` Module Reference

> **Module:** `std/jsbigints` (Nim standard library)  
> **Target:** JavaScript backend **only** — using this module with a native Nim backend is a fatal compile-time error.  
> **Purpose:** Arbitrary-precision integer arithmetic for Nim programs compiled to JavaScript, wrapping the native JavaScript `BigInt` type.

---

## Background: Why This Module Exists

JavaScript's standard `number` type is a 64-bit IEEE 754 floating-point value. This means integers larger than 2⁵³ − 1 (`Number.MAX_SAFE_INTEGER`) cannot be represented exactly — arithmetic on them silently loses precision. This limitation is a real problem for cryptography, hashing, financial calculations, and anything involving 64-bit IDs or timestamps.

JavaScript solved this in 2020 by introducing `BigInt`: a native arbitrary-precision integer type with its own literal syntax (`42n`) and a full set of operators. Nim's `jsbigints` module exposes this capability to Nim code compiled with `--backend:js`, mapping Nim-idiomatic syntax directly to JavaScript `BigInt` operations.

**Key characteristics of `JsBigInt`:**
- No fixed upper or lower bound — limited only by available memory.
- All operations are exact (no floating-point rounding).
- Cannot be mixed with JavaScript `number` in arithmetic — explicit conversion is required.
- Not available at Nim compile time (`nimvm`) or in `static` contexts.
- The `default` value is `0`.

---

## The `JsBigInt` Type

```nim
type JsBigInt* = distinct JsBigIntImpl
```

`JsBigInt` is a `distinct` type wrapping the JavaScript `bigint` primitive. Being `distinct` means it is not interchangeable with Nim's `int` at the type level — you cannot accidentally pass a `JsBigInt` where an `int` is expected, or vice versa. This prevents the silent precision loss that would occur if the two were mixed.

The default value of any uninitialized `JsBigInt` variable is `0`:

```nim
var x: JsBigInt
doAssert x == big"0"
doAssert x == JsBigInt.default
```

---

## Exported Symbols

---

### Constructors

There are three ways to create a `JsBigInt` value — all compile to `BigInt(...)` in JavaScript:

---

#### `big` (from integer)

```nim
func big*(integer: SomeInteger): JsBigInt
```

**What it does**

Converts any Nim integer type (`int`, `int32`, `int64`, `uint`, etc.) to a `JsBigInt`. The argument is passed directly to JavaScript's `BigInt()` constructor. This is the natural path when you already have a Nim integer at runtime.

**Example**

```nim
import std/jsbigints

let a = big(1234567890)
let b = 999.big              # method-call syntax works too

doAssert a == big"1234567890"
doAssert 0b1111100111.big == 0o1747.big and 0o1747.big == b
```

---

#### `` `'big` `` (literal suffix)

```nim
func `'big`*(num: cstring): JsBigInt
```

**What it does**

A custom numeric literal suffix that lets you write `JsBigInt` literals directly in source code with the `'big` suffix, much like `'f32` for floats. The suffix supports all of Nim's numeric literal bases: decimal, binary (`0b`), octal (`0o`), and hexadecimal (`0x`).

This is the most ergonomic way to express large constants that exceed the range of native Nim integers, since the number is passed as a string to `BigInt()` and never goes through Nim's integer types at all.

**Example**

```nim
import std/jsbigints

doAssert -1'big == 1'big - 2'big
doAssert 12'big == 12.big
doAssert 0b101'big == 0b101.big
doAssert 0o701'big == 0o701.big
doAssert 0xdeadbeaf'big == 0xdeadbeaf.big

# Numbers exceeding 64-bit int range — expressed exactly:
doAssert 0xffffffffffffffff'big == (1'big shl 64'big) - 1'big
```

Note: `static(12'big)` does not compile — `JsBigInt` cannot be used in compile-time contexts.

---

#### `big` (from string)

```nim
func big*(integer: cstring): JsBigInt
```

**What it does**

An alias for the `'big` literal suffix, but callable with an explicit `cstring` argument. Useful when you want to construct a `JsBigInt` from a string variable or a string expression computed at runtime. The string is passed directly to JavaScript's `BigInt()` — it must be a valid integer representation (decimal, hex with `0x` prefix, etc.).

**Example**

```nim
import std/jsbigints

let s: cstring = "99999999999999999999"
let x = big(s)
doAssert x == big"99999999999999999999"

doAssert big"-12" == -12'big
```

---

### Conversion Functions

---

#### `toCstring` (with radix)

```nim
func toCstring*(this: JsBigInt; radix: 2..36): cstring
```

**What it does**

Converts a `JsBigInt` to its string representation in a given numeric base. `radix` must be between 2 and 36 inclusive — the same constraint as JavaScript's `BigInt.prototype.toString(radix)`, which this wraps directly. Base 2 gives binary, base 16 gives hexadecimal, base 10 gives decimal, and any base up to 36 uses digits `0–9` followed by letters `a–z`.

The sign is preserved: negative values produce a leading `-`.

**Example**

```nim
import std/jsbigints

doAssert big"2147483647".toCstring(2) == "1111111111111111111111111111111".cstring
doAssert big"255".toCstring(16) == "ff".cstring
doAssert big"8".toCstring(2) == "1000".cstring
doAssert big"-10".toCstring(2) == "-1010".cstring
```

---

#### `toCstring` (decimal)

```nim
func toCstring*(this: JsBigInt): cstring
```

**What it does**

Shorthand for `toCstring(this, 10)` — converts a `JsBigInt` to its decimal string representation. Wraps JavaScript's `BigInt.prototype.toString()` with no argument.

**Example**

```nim
import std/jsbigints

let x = big"123456789012345678901234567890"
let s = x.toCstring()
doAssert s == "123456789012345678901234567890".cstring
```

---

#### `$`

```nim
func `$`*(this: JsBigInt): string
```

**What it does**

Returns a Nim `string` representation of the `JsBigInt`. The output includes a trailing `n` suffix — matching JavaScript's own `BigInt` literal notation (e.g. `42n`). This makes it immediately clear when printing that a value is a `BigInt` rather than an ordinary number.

Internally calls `toCstring` and appends `'n'`.

**Example**

```nim
import std/jsbigints

doAssert $big"1024" == "1024n"
doAssert $big"-7" == "-7n"
doAssert $(2'big ** 64'big) == "18446744073709551616n"
```

---

#### `toNumber`

```nim
func toNumber*(this: JsBigInt): int
```

**What it does**

Converts a `JsBigInt` to a JavaScript `Number` (Nim `int`). Wraps JavaScript's `Number()` constructor. **There is no bounds check** — if the value exceeds JavaScript's safe integer range (±2⁵³ − 1), the result will be an inexact approximation. This is an inherently lossy conversion and should be used only when you are certain the value fits.

**Example**

```nim
import std/jsbigints

doAssert toNumber(big"2147483647") == 2147483647.int

# Dangerous — value exceeds safe integer range:
# toNumber(big"99999999999999999999")  # result will be imprecise
```

---

#### `wrapToInt`

```nim
func wrapToInt*(this: JsBigInt; bits: Natural): JsBigInt
```

**What it does**

Wraps the value into the signed range of a `bits`-wide two's complement integer: `−2^(bits−1)` to `2^(bits−1) − 1`. Any bits above the width are discarded, and the result is interpreted as signed. Wraps JavaScript's `BigInt.asIntN(bits, value)`.

This is useful for simulating fixed-width signed integer arithmetic on arbitrary-precision values — for example, mimicking `int32` overflow behavior or performing modular arithmetic in a signed domain.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `this` | `JsBigInt` | The value to wrap. |
| `bits` | `Natural` | The width in bits of the target signed integer type. |

**Example**

```nim
import std/jsbigints

# 3 + 2^66 is a huge number, but wrapping to 13 bits gives 3
# (because 2^66 mod 2^13 == 0):
doAssert (big"3" + big"2" ** big"66").wrapToInt(13) == big"3"

# Simulate int8 overflow:
doAssert big"200".wrapToInt(8) == big"-56"  # 200 in int8 wraps to -56
```

---

#### `wrapToUint`

```nim
func wrapToUint*(this: JsBigInt; bits: Natural): JsBigInt
```

**What it does**

Wraps the value into the unsigned range of a `bits`-wide integer: `0` to `2^bits − 1`. Any bits above the width are discarded. Wraps JavaScript's `BigInt.asUintN(bits, value)`. The result is always non-negative.

Useful for emulating fixed-width unsigned integer arithmetic or computing values modulo a power of two.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `this` | `JsBigInt` | The value to wrap. |
| `bits` | `Natural` | The width in bits of the target unsigned integer type. |

**Example**

```nim
import std/jsbigints

# 3 + 2^66 wrapped to 66 bits is 3 (2^66 mod 2^66 == 0):
doAssert (big"3" + big"2" ** big"66").wrapToUint(66) == big"3")

# Compute value modulo 2^8 (i.e. as uint8):
doAssert big"300".wrapToUint(8) == big"44"  # 300 mod 256 == 44
```

---

### Arithmetic Operators

All binary arithmetic operators require **both operands to be `JsBigInt`** — you cannot mix `JsBigInt` with `int` directly. Convert first with `big()` or `toNumber()`.

---

#### `+` — addition

```nim
func `+`*(x, y: JsBigInt): JsBigInt
```

```nim
doAssert big"9" + big"1" == big"10"
doAssert big"99999999999999999999" + big"1" == big"100000000000000000000"
```

---

#### `-` — subtraction

```nim
func `-`*(x, y: JsBigInt): JsBigInt
```

```nim
doAssert big"9" - big"1" == big"8"
doAssert big"0" - big"1" == big"-1"
```

---

#### `*` — multiplication

```nim
func `*`*(x, y: JsBigInt): JsBigInt
```

```nim
doAssert big"42" * big"9" == big"378"
doAssert big"1000000000" * big"1000000000" == big"1000000000000000000"
```

---

#### `div` — integer division (truncating toward zero)

```nim
func `div`*(x, y: JsBigInt): JsBigInt
```

**What it does**

Integer division, truncating the result toward zero. This matches Nim's native `div` semantics (and JavaScript's `BigInt /` operator). The sign of the result follows the rule: if exactly one of the operands is negative, the result is negative.

```nim
doAssert big"13"  div big"3"  == big"4"
doAssert big"-13" div big"3"  == big"-4"   # toward zero, not floor
doAssert big"13"  div big"-3" == big"-4"
doAssert big"-13" div big"-3" == big"4"
```

---

#### `mod` — remainder

```nim
func `mod`*(x, y: JsBigInt): JsBigInt
```

**What it does**

Returns the remainder after division, with the sign matching the **dividend** (left operand). This is JavaScript's `%` operator behavior for `BigInt`. Note that this differs from mathematical modulo when the dividend is negative.

```nim
doAssert big"13"  mod big"3"  == big"1"
doAssert big"-13" mod big"3"  == big"-1"   # sign follows dividend
doAssert big"13"  mod big"-3" == big"1"
doAssert big"-13" mod big"-3" == big"-1"
```

---

#### `**` — exponentiation

```nim
func `**`*(x, y: JsBigInt): JsBigInt
```

**What it does**

Raises `x` to the power `y`. The exponent must be non-negative — passing a negative exponent raises a JavaScript `RangeError` at runtime (not a compile-time error). The result can be astronomically large: `BigInt` imposes no upper bound.

```nim
doAssert big"2"  ** big"64" == big"18446744073709551616"
doAssert big"-2" ** big"3"  == big"-8"
doAssert -big"2" ** big"2"  == big"4"   # parsed as (-2n) ** 2n
doAssert big"0"  ** big"0"  == big"1"   # edge case: 0^0 = 1

var ok = false
try: discard big"2" ** big"-1"  # RangeError: negative exponent
except: ok = true
doAssert ok
```

---

#### `-` — unary negation

```nim
func `-`*(this: JsBigInt): JsBigInt
```

Negates the value. Positive becomes negative, negative becomes positive, zero stays zero.

```nim
doAssert -(big"10101010101") == big"-10101010101"
doAssert -(big"-7") == big"7"
doAssert -(big"0") == big"0"
```

**Note on unary `+`:** Unary `+` is intentionally disabled (marked `{.error.}`) because JavaScript's `BigInt` specification explicitly forbids it for compatibility reasons. Attempting to use `+big"5"` is a compile-time error.

---

### Comparison Operators

All comparison operators return `bool`.

---

#### `<`, `<=`, `==`

```nim
func `<`*(x, y: JsBigInt): bool
func `<=`*(x, y: JsBigInt): bool
func `==`*(x, y: JsBigInt): bool
```

Standard ordering and equality comparisons. The operators `>`, `>=`, and `!=` are derived automatically from these by Nim.

```nim
doAssert big"2"  < big"9"
doAssert big"1"  <= big"5"
doAssert big"42" == big"42"

let a = big"2147483647"
let b = big"666"
doAssert a != b
doAssert a > b
doAssert a >= b
doAssert b < a
doAssert b <= a
```

---

### Bitwise Operators

Bitwise operators work on the two's complement representation of the value. For negative numbers, this is an infinite-precision two's complement (all high bits are `1` conceptually). Results are consistent with what JavaScript's `BigInt` bitwise operators produce.

---

#### `and` — bitwise AND

```nim
func `and`*(x, y: JsBigInt): JsBigInt
```

```nim
doAssert (big"555" and big"2") == big"2"
# 555 = 0b1000101011, 2 = 0b10 → AND = 0b10 = 2
```

---

#### `or` — bitwise OR

```nim
func `or`*(x, y: JsBigInt): JsBigInt
```

```nim
doAssert (big"555" or big"2") == big"555"
# 555 already has bit 1 set, so OR with 2 is still 555
```

---

#### `xor` — bitwise XOR

```nim
func `xor`*(x, y: JsBigInt): JsBigInt
```

```nim
doAssert (big"555" xor big"2") == big"553"
# flips bit 1 of 555
```

---

#### `shl` — left shift

```nim
func `shl`*(a, b: JsBigInt): JsBigInt
```

Shifts `a` left by `b` bit positions, which is equivalent to multiplying by `2^b`. Unlike fixed-width integers, there is no overflow — the result simply grows.

```nim
doAssert big"999" shl big"2" == big"3996"   # 999 * 4
doAssert big"1"   shl big"64" == big"18446744073709551616"
```

---

#### `shr` — right shift (arithmetic)

```nim
func `shr`*(a, b: JsBigInt): JsBigInt
```

Shifts `a` right by `b` bit positions, equivalent to integer division by `2^b` (floor division for positive numbers). For negative `BigInt` values, this is an arithmetic shift (sign-extending).

```nim
doAssert big"999" shr big"2" == big"249"    # 999 / 4, truncated
doAssert big"8"   shr big"1" == big"4"
```

---

### In-Place Mutation

All in-place operators modify the variable directly, like their `int` counterparts.

---

#### `inc` (by 1)

```nim
func inc*(this: var JsBigInt)
```

Increments the value by 1.

```nim
var x = big"1"
inc x
doAssert x == big"2"
```

---

#### `inc` (by amount)

```nim
func inc*(this: var JsBigInt; amount: JsBigInt)
```

Increments the value by `amount`.

```nim
var x = big"1"
inc x, big"2"
doAssert x == big"3"
```

---

#### `dec` (by 1)

```nim
func dec*(this: var JsBigInt)
```

Decrements the value by 1.

```nim
var x = big"2"
dec x
doAssert x == big"1"
```

---

#### `dec` (by amount)

```nim
func dec*(this: var JsBigInt; amount: JsBigInt)
```

Decrements the value by `amount`.

```nim
var x = big"1"
dec x, big"2"
doAssert x == big"-1"
```

---

#### `+=`, `-=`, `*=`, `/=`

```nim
func `+=`*(x: var JsBigInt; y: JsBigInt)
func `-=`*(x: var JsBigInt; y: JsBigInt)
func `*=`*(x: var JsBigInt; y: JsBigInt)
func `/=`*(x: var JsBigInt; y: JsBigInt)
```

Compound assignment operators. `/=` performs truncating integer division (same as `div`).

```nim
var a = big"1";  a += big"2";  doAssert a == big"3"
var b = big"1";  b -= big"2";  doAssert b == big"-1"
var c = big"2";  c *= big"4";  doAssert c == big"8"
var d = big"11"; d /= big"2";  doAssert d == big"5"   # truncates
```

---

### Intentionally Disabled Operations

Three operations are permanently disabled with `{.error.}` pragmas — using them is a **compile-time error**, by design:

| Operation | Reason |
|-----------|--------|
| Unary `+big"x"` | JavaScript's `BigInt` spec forbids unary `+` for asm.js compatibility reasons. |
| `low(JsBigInt)` | Arbitrary-precision integers have no minimum value. |
| `high(JsBigInt)` | Arbitrary-precision integers have no maximum value. |

---

## Common Patterns and Pitfalls

**Mixing `JsBigInt` with `int` is a compile error:**

```nim
# let x = big"5" + 3   # Error: type mismatch
let x = big"5" + big(3)  # correct — convert first
```

**`JsBigInt` cannot be used at compile time:**

```nim
# static: discard big"5"  # Error: JsBigInt cannot be used in static context
```

**Checking for divisibility:**

```nim
let n = big"1000000000000000000"
if n mod big"2" == big"0":
  echo "even"
```

**Computing large factorials:**

```nim
func factorial(n: JsBigInt): JsBigInt =
  result = big"1"
  var i = big"2"
  while i <= n:
    result *= i
    inc i

echo $factorial(big"50")  # exact result, no overflow
```

---

## Summary Table

| Category | Symbols |
|----------|---------|
| **Type** | `JsBigInt` |
| **Constructors** | `big(int)`, `` n'big ``, `big(cstring)` |
| **To string** | `toCstring(radix)`, `toCstring()`, `$` |
| **To number** | `toNumber` |
| **Bit-width wrapping** | `wrapToInt`, `wrapToUint` |
| **Arithmetic** | `+`, `-`, `*`, `div`, `mod`, `**`, unary `-` |
| **Comparison** | `<`, `<=`, `==` (and derived `>`, `>=`, `!=`) |
| **Bitwise** | `and`, `or`, `xor`, `shl`, `shr` |
| **Mutation** | `inc`, `dec`, `+=`, `-=`, `*=`, `/=` |
| **Disabled** | unary `+`, `low`, `high` |
