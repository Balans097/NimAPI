# arithmetics.nim — Complete Reference

> **Module**: `system/arithmetics` — Nim's Runtime Library  
> **Purpose**: Defines all built-in arithmetic operators and a handful of ordinal helpers for every numeric type in Nim. Everything here is implemented as a compiler *magic* — the Nim compiler maps these directly to CPU instructions or C operators rather than calling any actual procedure body.  
> **Exported via**: automatically available in every Nim program through the implicit `system` import.

---

## How to Read This File

Most entries carry the pragma `{.magic: "XYZ", noSideEffect.}`. This means:

- The procedure body (if present) is for documentation and runnable examples only — the compiler replaces every call with the corresponding low-level instruction.
- `noSideEffect` guarantees the compiler that the operation has no observable effect other than its return value, allowing it to be used in `func`, constant expressions, and strict mode.

The operators are defined once for each concrete numeric type (`int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8`, …, `float32`, `float64`). This reference groups the overloads and explains the semantics once, noting type-specific differences where they exist.

---

## Table of Contents

1. [Ordinal Helpers](#1-ordinal-helpers)
2. [Signed Integer Arithmetic](#2-signed-integer-arithmetic)
3. [Integer Division and Modulo](#3-integer-division-and-modulo)
4. [Bitwise Operations (Signed)](#4-bitwise-operations-signed)
5. [Bit Shifts (Signed)](#5-bit-shifts-signed)
6. [Unsigned Integer Arithmetic](#6-unsigned-integer-arithmetic)
7. [Unsigned Bit Operations and Shifts](#7-unsigned-bit-operations-and-shifts)
8. [Compound Assignment Operators](#8-compound-assignment-operators)
9. [Floating-Point Arithmetic](#9-floating-point-arithmetic)
10. [Wrapping Arithmetic Operators (`%`)](#10-wrapping-arithmetic-operators-)
11. [Operator Precedence Quick Reference](#11-operator-precedence-quick-reference)

---

## 1. Ordinal Helpers

These four procedures work on any **ordinal type** — integers, characters, booleans, and enumerations. "Ordinal" means the type has a well-defined, finite, ordered sequence of values.

---

### `succ[T: Ordinal, V: SomeInteger](x: T, y: V = 1): T`

Returns the `y`-th **successor** of `x` — the value that comes `y` steps *after* `x` in the ordinal sequence. The default step is 1. Raises `OverflowDefect` at runtime (or a compile-time error for constants) if the result would fall outside the type's range.

**When to use**: anywhere you need to advance an ordinal value by an arbitrary step and want overflow to be caught explicitly. For simple `+1` on a plain integer, `inc` is more idiomatic.

```nim
assert succ(5) == 6          # one step forward
assert succ(5, 3) == 8       # three steps forward
assert succ('a') == 'b'      # works on char too
assert succ(true) == ???     # OverflowDefect: bool has no successor after true
```

---

### `pred[T: Ordinal, V: SomeInteger](x: T, y: V = 1): T`

Returns the `y`-th **predecessor** of `x` — the value `y` steps *before* `x`. Raises `OverflowDefect` if the result would underflow.

```nim
assert pred(5) == 4          # one step back
assert pred(5, 3) == 2       # three steps back
assert pred('z') == 'y'
```

---

### `inc[T: Ordinal, V: SomeInteger](x: var T, y: V = 1)`

**Mutates** `x` in place by advancing it `y` steps. Equivalent to `x = succ(x, y)`. Raises `OverflowDefect` on overflow.

```nim
var i = 2
inc(i)        # i == 3
inc(i, 3)     # i == 6
```

`inc` and `dec` are the idiomatic Nim way to advance loop counters and ordinal state variables. They communicate intent more clearly than `i += 1` for non-numeric ordinals.

---

### `dec[T: Ordinal, V: SomeInteger](x: var T, y: V = 1)`

**Mutates** `x` in place by retreating it `y` steps. Equivalent to `x = pred(x, y)`. Raises `OverflowDefect` on underflow.

```nim
var i = 2
dec(i)        # i == 1
dec(i, 3)     # i == -2
```

---

## 2. Signed Integer Arithmetic

All operators in this section work on `int`, `int8`, `int16`, `int32`, and `int64`. When overflow checking is enabled (the default in debug builds), exceeding the type's range raises `OverflowDefect`.

---

### Unary `+x`

```nim
proc `+`*(x: int): int
```

Returns `x` unchanged. The unary plus is a no-op provided for symmetry with unary minus and to satisfy syntactic contexts that require an expression.

```nim
let x = +42    # x == 42
```

---

### Unary `-x`

```nim
proc `-`*(x: int): int
```

Returns the **arithmetic negation** of `x`. For signed integers: `-x == 0 - x`. Note that negating `low(int)` (the most negative value) overflows, since its positive counterpart does not fit.

```nim
assert -7 == -7
assert -(-7) == 7
# -low(int8) would raise OverflowDefect in checked mode
```

---

### `x + y` (binary)

```nim
proc `+`*(x, y: int): int
```

Standard **addition**. The result has the same type as the operands. Overflow is checked in debug builds.

```nim
assert 3 + 4 == 7
assert 127'i8 + 1'i8  # OverflowDefect in checked mode
```

---

### `x - y` (binary)

```nim
proc `-`*(x, y: int): int
```

Standard **subtraction**. Overflow is checked.

```nim
assert 10 - 3 == 7
assert -128'i8 - 1'i8  # OverflowDefect in checked mode
```

---

### `x * y`

```nim
proc `*`*(x, y: int): int
```

Standard **multiplication**. Overflow is checked.

```nim
assert 6 * 7 == 42
```

---

## 3. Integer Division and Modulo

These operators implement **truncation-towards-zero** semantics — the same as C's `/` and `%`. The sign of the result follows from the signs of the operands in a specific way; see the tables below.

---

### `x div y`

```nim
proc `div`*(x, y: int): int
```

Computes the **integer quotient**, truncating towards zero. This is approximately `trunc(x / y)` as a real-number division.

Division by zero raises `DivByZeroDefect`.

| x | y | x div y | Explanation |
|---|---|---------|-------------|
| 7 | 3 | 2 | 7 / 3 = 2.33… → truncate to 2 |
| -7 | 3 | -2 | -7 / 3 = -2.33… → truncate to -2 |
| 7 | -3 | -2 | 7 / -3 = -2.33… → truncate to -2 |
| -7 | -3 | 2 | -7 / -3 = 2.33… → truncate to 2 |

```nim
assert (7 div 3) == 2
assert (-7 div 3) == -2
assert (7 div -3) == -2
assert (-7 div -3) == 2
```

---

### `x mod y`

```nim
proc `mod`*(x, y: int): int
```

Computes the **remainder** after `div`. Defined as `x - (x div y) * y`, so the sign of the result always matches the sign of `x` (the dividend), regardless of the sign of `y`.

Division by zero raises `DivByZeroDefect`.

| x | y | x mod y | Check: x - (x div y)*y |
|---|---|---------|------------------------|
| 7 | 5 | 2 | 7 - 2*5 = 2 |
| -7 | 5 | -2 | -7 - (-2)*5 = -7+10 = … wait: -7 div 5 = -1, so -7 - (-1)*5 = -7+5 = -2 |
| 7 | -5 | 2 | 7 div -5 = -1, 7 - (-1)*(-5) = 7-5 = 2 |
| -7 | -5 | -2 | -7 div -5 = 1, -7 - 1*(-5) = -7+5 = -2 |

```nim
assert (7 mod 5) == 2
assert (-7 mod 5) == -2
assert (7 mod -5) == 2
assert (-7 mod -5) == -2
```

> **Gotcha**: this is *not* the mathematical modulo (which is always non-negative). For a non-negative result use `floorMod` from `std/math`.

---

## 4. Bitwise Operations (Signed)

These operate on the binary representation of the value. For signed integers the two's complement representation is used.

---

### `not x`

```nim
proc `not`*(x: int): int
```

**Bitwise complement** — flips every bit. Equivalent to `-x - 1` in two's complement.

```nim
assert not 0'u8 == 255         # all bits set
assert not 0'i8 == -1          # two's complement: all bits set = -1
assert not 1000'u16 == 64535
assert not 1000'i16 == -1001   # -(1000+1)
```

---

### `x and y`

```nim
proc `and`*(x, y: int): int
```

**Bitwise AND** — a result bit is 1 only if both corresponding input bits are 1. Commonly used to **mask off** bits.

```nim
assert (0b0011 and 0b0101) == 0b0001   # only bit 0 is set in both
assert (0xFF and 0x0F) == 0x0F         # keep lower nibble only
```

---

### `x or y`

```nim
proc `or`*(x, y: int): int
```

**Bitwise OR** — a result bit is 1 if *at least one* corresponding input bit is 1. Commonly used to **set** specific bits.

```nim
assert (0b0011 or 0b0101) == 0b0111
assert (flags or 0b0100) == flags with bit 2 set
```

---

### `x xor y`

```nim
proc `xor`*(x, y: int): int
```

**Bitwise XOR** (exclusive or) — a result bit is 1 if the corresponding input bits *differ*. Commonly used to **toggle** bits or check for differences.

```nim
assert (0b0011 xor 0b0101) == 0b0110
assert (x xor x) == 0    # XOR with itself always produces zero
```

---

## 5. Bit Shifts (Signed)

### `x shr y`

```nim
proc `shr`*(x: int, y: SomeInteger): int
```

**Arithmetic shift right** (sign-extending). Shifts the bits of `x` rightward by `y` positions. Vacant bit positions on the *left* are filled with **the sign bit** — so negative numbers remain negative and positive numbers remain positive.

`y` is taken modulo `sizeof(x) * 8` (e.g. for `int32`, `shr 35` acts as `shr 3`).

> **Note**: operator precedence differs from C. In Nim `a + b shr c` parses as `a + (b shr c)`, because `shr` has lower precedence than `+`.

```nim
assert 16 shr 2 == 4          # 0b10000 → 0b00100
assert -16 shr 2 == -4        # sign bit preserved
assert 0b1000_0000'i8 shr 4 == 0b1111_1000'i8  # sign-extension fills with 1s
assert -1 shr 5 == -1         # shifting -1 right always yields -1
```

When `nimOldShiftRight` is defined, `shr` on signed integers performs a **logical** (zero-filling) shift instead and is deprecated.

For unsigned integers, `shr` always performs a **logical** (zero-filling) shift — vacant bits are filled with 0.

---

### `ashr(x, y)` — Explicit Arithmetic Shift Right

```nim
proc ashr*(x: int, y: SomeInteger): int
```

Identical in behaviour to `shr` for signed integers (arithmetic, sign-extending). Provided as a **named function** (not an operator) for cases where operator syntax would be ambiguous or when you want to make the intent explicit in code.

```nim
assert ashr(0b0001_0000'i8, 2) == 0b0000_0100'i8
assert ashr(0b1000_0000'i8, 1) == 0b1100_0000'i8  # sign bit copied
assert ashr(0b1000_0000'i8, 8) == 0b1000_0000'i8  # shift ≥ width: all sign bits
```

---

### `x shl y`

```nim
proc `shl`*(x: int, y: SomeInteger): int
```

**Shift left**. Shifts bits leftward by `y` positions; vacated positions on the right are filled with 0. Equivalent to multiplication by 2^y (without overflow checking). `y` is taken modulo `sizeof(x) * 8`.

```nim
assert 1'i32 shl 4 == 16          # 0b1 → 0b10000
assert 1'i64 shl 4 == 16
assert 0b0000_0001'i8 shl 7 == 0b1000_0000'i8   # sets the sign bit
```

---

## 6. Unsigned Integer Arithmetic

All operators in this section work on `uint`, `uint8`, `uint16`, `uint32`, `uint64`. Unsigned arithmetic **never raises overflow errors** — it wraps modulo 2^N where N is the bit width.

---

### `x + y`, `x - y`, `x * y` (unsigned)

```nim
proc `+`*(x, y: uint): uint
proc `-`*(x, y: uint): uint
proc `*`*(x, y: uint): uint
```

Standard unsigned addition, subtraction, and multiplication. All wrap silently on overflow/underflow.

```nim
assert 255'u8 + 1'u8 == 0'u8    # wraps around
assert 0'u8 - 1'u8 == 255'u8    # underflow wraps to max
assert 200'u8 * 2'u8 == 144'u8  # 400 mod 256 = 144
```

---

### `x div y` (unsigned)

```nim
proc `div`*(x, y: uint): uint
```

Unsigned integer division — always truncates towards zero (which for non-negative numbers is the same as towards negative infinity). No sign complications. Raises `DivByZeroDefect` on division by zero.

```nim
assert 7'u div 3'u == 2'u
```

---

### `x mod y` (unsigned)

```nim
proc `mod`*(x, y: uint): uint
```

Unsigned remainder. Result is always in `[0, y)`. Raises `DivByZeroDefect` on zero divisor.

```nim
assert 7'u mod 5'u == 2'u
```

---

## 7. Unsigned Bit Operations and Shifts

### `not x` (unsigned)

```nim
proc `not`*(x: uint): uint
```

Bitwise complement. For unsigned types the result is simply `(2^N - 1) xor x`, which is always non-negative and fits in the type.

```nim
assert not 0'u8 == 255'u8
assert not 255'u8 == 0'u8
```

---

### `x and y`, `x or y`, `x xor y` (unsigned)

Same bitwise semantics as for signed types. Operate on the raw binary representation.

```nim
assert (0xFF'u8 and 0x0F'u8) == 0x0F'u8
assert (0xF0'u8 or 0x0F'u8) == 0xFF'u8
assert (0xFF'u8 xor 0xFF'u8) == 0x00'u8
```

---

### `x shr y` (unsigned)

```nim
proc `shr`*(x: uint, y: SomeInteger): uint
```

**Logical shift right** (zero-filling). Always fills vacated positions with 0, regardless of the value of the most significant bit. This is the natural behaviour for unsigned types.

```nim
assert 0xFF'u8 shr 4 == 0x0F'u8   # upper nibble gone, padded with 0
```

---

### `x shl y` (unsigned)

```nim
proc `shl`*(x: uint, y: SomeInteger): uint
```

**Shift left**, zero-filling on the right. Equivalent to multiplication by 2^y, wrapping on overflow.

```nim
assert 1'u8 shl 7 == 128'u8
assert 3'u8 shl 6 == 192'u8
```

---

## 8. Compound Assignment Operators

These mutate the left operand in place. They are thin wrappers — `x += y` desugars to the same code as `x = x + y`.

---

### `x += y` — Add and Assign

```nim
proc `+=`*[T: SomeInteger](x: var T, y: T)
```

Increments `x` by `y`. Works on all integer types (signed and unsigned). For floating-point types a separate overload exists (see §9).

```nim
var n = 10
n += 5    # n == 15
```

---

### `x -= y` — Subtract and Assign

```nim
proc `-=`*[T: SomeInteger](x: var T, y: T)
```

Decrements `x` by `y`.

```nim
var n = 10
n -= 3    # n == 7
```

---

### `x *= y` — Multiply and Assign

```nim
proc `*=`*[T: SomeInteger](x: var T, y: T)
```

Multiplies `x` by `y` in place. Implemented as `x = x * y`.

```nim
var n = 6
n *= 7    # n == 42
```

---

## 9. Floating-Point Arithmetic

All operators below work on both `float` (= `float64`, 64-bit IEEE 754 double precision) and `float32` (32-bit single precision). IEEE 754 rules apply: operations on `NaN` produce `NaN`; overflow produces `Inf` or `-Inf`; division by zero also produces `Inf`/`-Inf` rather than raising an exception.

---

### Unary `+x`, `-x` (float)

```nim
proc `+`*(x: float): float
proc `-`*(x: float): float
```

Unary plus (no-op) and negation. Negation flips the sign bit — it correctly handles `+0.0`, `-0.0`, `Inf`, `-Inf`, and `NaN` per IEEE 754.

```nim
assert -3.14 == -3.14
assert -(-3.14) == 3.14
assert -(Inf) == -Inf
```

---

### `x + y`, `x - y`, `x * y` (float)

Standard IEEE 754 addition, subtraction, multiplication. The result is a rounded real-number result.

```nim
assert 0.1 + 0.2 != 0.3      # classic floating-point rounding
assert 1.0 + Inf == Inf
assert Inf - Inf == NaN
```

---

### `x / y` (float)

```nim
proc `/`*(x, y: float): float
```

IEEE 754 floating-point division. Division by zero produces `Inf`, `-Inf`, or `NaN` (0.0/0.0) rather than raising an exception.

```nim
assert 1.0 / 2.0 == 0.5
assert 1.0 / 0.0 == Inf
assert -1.0 / 0.0 == -Inf
assert 0.0 / 0.0 != 0.0 / 0.0   # NaN != NaN
```

> For integer (truncating) division use `div`; `/` is only defined for floating-point types.

---

### Floating-point compound assignment: `+=`, `-=`, `*=`

```nim
proc `+=`*[T: float|float32|float64](x: var T, y: T)
proc `-=`*[T: float|float32|float64](x: var T, y: T)
proc `*=`*[T: float|float32|float64](x: var T, y: T)
```

Mutate `x` in place. Inline wrappers around the base operators.

```nim
var f = 1.0
f += 0.5    # f == 1.5
f *= 2.0    # f == 3.0
f -= 0.1    # f == 2.9 (approximately)
```

---

### `/=` — Divide and Assign (float)

```nim
proc `/=`*(x: var float64, y: float64)
proc `/=`*[T: float|float32](x: var T, y: T)
```

Divides `x` by `y` in place.

```nim
var f = 10.0
f /= 4.0    # f == 2.5
```

---

## 10. Wrapping Arithmetic Operators (`%`)

These operators treat their operands as **unsigned** regardless of the declared type and always wrap on overflow/underflow without raising any exception. They are available for all signed integer types (`int`, `int8`, `int16`, `int32`, `int64`).

The `%` suffix is a visual reminder that the operation is *modular* (no overflow defect, ever).

---

### `x +% y` — Wrapping Addition

```nim
proc `+%`*(x, y: int): int
```

Adds `x` and `y` as if they were unsigned, then reinterprets the bit pattern as the signed type. Equivalent to `cast[int](cast[uint](x) + cast[uint](y))`.

```nim
assert high(int8) +% 1'i8 == low(int8)   # 127 + 1 = -128 (wraps)
assert -1'i8 +% 1'i8 == 0'i8
```

**Use case**: hash functions, checksum accumulators, and any arithmetic where you deliberately want two's complement wrap-around without an exception.

---

### `x -% y` — Wrapping Subtraction

```nim
proc `-%`*(x, y: int): int
```

Subtracts without overflow checking, treating operands as unsigned.

```nim
assert low(int8) -% 1'i8 == high(int8)   # -128 - 1 = 127 (wraps)
assert 0'i8 -% 1'i8 == -1'i8             # same as regular subtraction here
```

---

### `x *% y` — Wrapping Multiplication

```nim
proc `*%`*(x, y: int): int
```

Multiplies without overflow checking, treating operands as unsigned before multiplying.

```nim
assert 100'i8 *% 100'i8 == 16'i8   # 10000 mod 256 reinterpreted as int8
```

---

### `x /% y` — Wrapping Division

```nim
proc `/%`*(x, y: int): int
```

Divides treating both operands as unsigned. For non-negative values this is the same as `div`. The key difference appears when one or both values are negative (large unsigned numbers):

```nim
assert (-1'i8 /% 2'i8) == 127'i8
# Because cast[uint8](-1) = 255, and 255 div 2 = 127
```

---

### `x %% y` — Wrapping Modulo

```nim
proc `%%`*(x, y: int): int
```

Computes the remainder treating both operands as unsigned. The result is always non-negative when interpreted as the signed type (within the unsigned range of the type).

```nim
assert (-7'i32 %% 5'i32) == 3'i32
# cast[uint32](-7) = 4294967289, 4294967289 mod 5 = 3 (reinterpreted as int32: 3)
```

> **Gotcha**: unlike `mod`, the result sign for `%%` comes from the *unsigned interpretation*, not the sign of `x`.

---

## 11. Operator Precedence Quick Reference

Nim's operator precedence for the operators in this module, from highest to lowest:

| Level | Operators | Notes |
|-------|-----------|-------|
| 9 (highest arithmetic) | unary `+`, unary `-`, `not` | Right-to-left |
| 8 | `*`, `/`, `div`, `mod`, `shl`, `shr`, `%%`, `*%`, `/%` | Left-to-right |
| 7 | `+`, `-`, `+%`, `-%`, `or`, `xor` | Left-to-right |
| 6 | `and` | Left-to-right |

> **Important**: `shl` and `shr` have lower precedence than `+` in Nim, unlike in C. `a + b shl 2` means `a + (b shl 2)`, not `(a + b) shl 2`.

---

## Summary Table

| Operator / Function | Types | Overflow behaviour | Notes |
|---|---|---|---|
| `succ`, `pred` | Any ordinal | `OverflowDefect` | Step can be > 1 |
| `inc`, `dec` | Any ordinal (mut) | `OverflowDefect` | In-place |
| Unary `+` | Signed int, float | — | No-op |
| Unary `-` | Signed int, float | `OverflowDefect` on min int | Negation |
| `+`, `-`, `*` | Signed int | `OverflowDefect` | Checked in debug |
| `+`, `-`, `*` | Unsigned int | Wraps silently | Modular |
| `+`, `-`, `*`, `/` | float32, float64 | IEEE 754 (Inf/NaN) | No exception |
| `div`, `mod` | Signed int | `DivByZeroDefect` | Truncates to zero |
| `div`, `mod` | Unsigned int | `DivByZeroDefect` | Always positive |
| `not` | Int, uint | — | Bitwise complement |
| `and`, `or`, `xor` | Int, uint | — | Bitwise |
| `shr` | Signed int | — | Arithmetic (sign-extending) |
| `shr` | Unsigned int | — | Logical (zero-filling) |
| `ashr` | Signed int | — | Named form of arithmetic shr |
| `shl` | Int, uint | — | Zero-fills right |
| `+=`, `-=`, `*=` | Int, float | Same as base op | In-place |
| `/=` | float only | IEEE 754 | In-place |
| `+%`, `-%`, `*%` | Signed int | Never raises | Wrapping |
| `/%`, `%%` | Signed int | Never raises | Unsigned-treating wrapping |
