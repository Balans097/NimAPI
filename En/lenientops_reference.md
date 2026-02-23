# lenientops — Module Reference

## Overview

The `lenientops` module provides overloaded versions of the common binary operators `+`, `-`, `*`, `/`, `<`, and `<=` that accept **mixed integer and float operands**. Without this module, Nim's strict type system requires you to explicitly convert one operand before combining an integer and a float — for example, `float(x) + y`. Importing `lenientops` makes such expressions work directly.

### The core rule

Whenever an integer and a float appear together, **the integer is converted to the float's type**. The result is always a float — specifically, the same float type that was already in the expression. There is no implicit promotion from `float32` to `float64` or vice versa: the float operand's type is preserved exactly.

```
int  op  float32  →  float32
int  op  float64  →  float64
float32  op  int  →  float32
float64  op  int  →  float64
```

### Why is this a separate module?

Nim does not perform implicit numeric conversions by default. Converting an integer to a float can **silently lose information**: a 64-bit integer can represent values that a 64-bit float cannot represent exactly (integers beyond 2⁵³ cannot all be represented in `float64`). Because this trade-off is not always acceptable, the operators are not part of the standard library's default scope — you opt in deliberately by importing `lenientops`.

### What about `==`?

The equality operator `==` is intentionally **not provided**. Comparing a float and an integer for equality is ambiguous: should the integer be cast to float (risking precision loss), or should the float be rounded to integer? The right answer depends on context, so you are expected to make an explicit choice in your own code rather than relying on an implicit conversion.

### What about `>=` and `>`?

These are also **not provided** in this module. `system.nim` already defines templates `>=` and `>` with an `(x, y: untyped)` signature, and adding overloads here would cause ambiguous call errors. In practice, `a > b` is equivalent to `b < a`, and `a >= b` is equivalent to `b <= a`, so you can rewrite comparisons using the provided `<` and `<=`.

---

## Exported Operators

All operators are generic over `I: SomeInteger` (any integer type: `int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8`, …) and `F: SomeFloat` (`float32` or `float64`). Every operator is a `func` (pure, no side effects) and is `inline` (zero call overhead).

Each operator is provided in **two overloads**: one for `(integer, float)` and one for `(float, integer)`, covering both argument orders.

---

### `+` — Addition

```nim
func `+`*[I: SomeInteger, F: SomeFloat](i: I, f: F): F
func `+`*[I: SomeInteger, F: SomeFloat](f: F, i: I): F
```

Adds an integer and a float. The integer is first converted to type `F`, then ordinary float addition is performed. The result type is `F`.

**Example:**

```nim
import lenientops

let a = 3 + 1.5       # int + float64  →  4.5 : float64
let b = 2.0'f32 + 10  # float32 + int  →  12.0 : float32
let c = 7 + 0.5'f32   # int + float32  →  7.5 : float32

echo a  # 4.5
echo b  # 12.0
echo c  # 7.5
```

---

### `-` — Subtraction

```nim
func `-`*[I: SomeInteger, F: SomeFloat](i: I, f: F): F
func `-`*[I: SomeInteger, F: SomeFloat](f: F, i: I): F
```

Subtracts a float from an integer or an integer from a float. The integer is converted to `F` before the subtraction, and operand order is respected as usual.

**Example:**

```nim
import lenientops

let a = 10 - 3.5      # int - float64  →  6.5 : float64
let b = 10.0 - 3      # float64 - int  →  7.0 : float64
let c = 1 - 1.5'f32   # int - float32  →  -0.5 : float32

echo a  # 6.5
echo b  # 7.0
echo c  # -0.5
```

---

### `*` — Multiplication

```nim
func `*`*[I: SomeInteger, F: SomeFloat](i: I, f: F): F
func `*`*[I: SomeInteger, F: SomeFloat](f: F, i: I): F
```

Multiplies an integer and a float. Because multiplication is commutative, both overloads produce the same numeric result regardless of argument order — but the return type is always `F`.

**Example:**

```nim
import lenientops

let a = 4 * 2.5        # int * float64  →  10.0 : float64
let b = 0.1'f32 * 3    # float32 * int  →  0.3 : float32
let c = 100 * 0.01     # int * float64  →  1.0 : float64

echo a  # 10.0
echo b  # 0.3
echo c  # 1.0
```

---

### `/` — Division

```nim
func `/`*[I: SomeInteger, F: SomeFloat](i: I, f: F): F
func `/`*[I: SomeInteger, F: SomeFloat](f: F, i: I): F
```

Divides an integer by a float, or a float by an integer. The integer is converted to `F` first, and then standard float division is performed. Note that unlike integer division (`div`), this always produces a floating-point quotient, so there is no truncation.

**Example:**

```nim
import lenientops

let a = 7 / 2.0        # int / float64  →  3.5 : float64
let b = 1.0 / 4        # float64 / int  →  0.25 : float64
let c = 1'i32 / 3.0'f32  # int32 / float32  →  0.3333... : float32

echo a  # 3.5
echo b  # 0.25
echo c  # 0.33333334
```

---

### `<` — Less-than comparison

```nim
func `<`*[I: SomeInteger, F: SomeFloat](i: I, f: F): bool
func `<`*[I: SomeInteger, F: SomeFloat](f: F, i: I): bool
```

Returns `true` if the left operand is strictly less than the right operand, after converting the integer to type `F`. Because converting large integers to float may lose precision, results near the boundary between representable float values should be interpreted with care.

**Example:**

```nim
import lenientops

echo 3 < 3.5      # true  — int converted to float64: 3.0 < 3.5
echo 4 < 3.5      # false — 4.0 < 3.5 is false
echo 2.9 < 3      # true  — 2.9 < 3.0
echo 3.0 < 3      # false — 3.0 < 3.0 is false (strict)
```

---

### `<=` — Less-than-or-equal comparison

```nim
func `<=`*[I: SomeInteger, F: SomeFloat](i: I, f: F): bool
func `<=`*[I: SomeInteger, F: SomeFloat](f: F, i: I): bool
```

Returns `true` if the left operand is less than or equal to the right operand, after converting the integer to type `F`. This is the non-strict variant of `<`.

**Example:**

```nim
import lenientops

echo 3 <= 3.0     # true  — 3.0 <= 3.0
echo 3 <= 2.9     # false — 3.0 <= 2.9 is false
echo 2.9 <= 3     # true  — 2.9 <= 3.0
echo 3.0 <= 3     # true  — 3.0 <= 3.0
```

**Replacing `>=` and `>`:** since those operators are not exported, use the following equivalents:

```nim
import lenientops

let x = 5
let y = 4.5

# Instead of x > y, write:
echo y < x     # true

# Instead of x >= y, write:
echo y <= x    # true
```

---

## Precision Warning

Converting an integer to a float is **not always lossless**. A `float64` has a 52-bit mantissa, meaning it can represent all integers up to 2⁵³ (9 007 199 254 740 992) exactly, but larger `int64` values may be rounded. A `float32` has a 23-bit mantissa and can only represent integers up to 2²⁴ (16 777 216) exactly.

```nim
import lenientops

let big = 9_007_199_254_740_993'i64  # 2^53 + 1
echo big + 0.0  # may print 9007199254740992.0 — the last bit is lost
```

This is why the module carries the name "lenient" and its doc comment says *"use with care"*.

---

## Summary Table

| Operator | Signature | Returns | Notes |
|---|---|---|---|
| `+` | `(int, float)` or `(float, int)` | `F` | Integer converted to `F` first |
| `-` | `(int, float)` or `(float, int)` | `F` | Operand order respected |
| `*` | `(int, float)` or `(float, int)` | `F` | Commutative |
| `/` | `(int, float)` or `(float, int)` | `F` | Always float division, no truncation |
| `<` | `(int, float)` or `(float, int)` | `bool` | Strict inequality |
| `<=` | `(int, float)` or `(float, int)` | `bool` | Non-strict inequality |
| `==` | — | — | **Not provided** — ambiguous semantics |
| `>` / `>=` | — | — | **Not provided** — conflicts with `system.nim` templates |
