# Nim `fenv` Module — Complete Reference

## Overview

The `fenv` module is Nim's interface to the C standard library header `<fenv.h>`, which gives you direct control over the **floating-point environment** of your process. The floating-point environment consists of two orthogonal things:

1. **Exception flags** — sticky status bits that the hardware sets whenever a notable event occurs during a floating-point operation (division by zero, overflow, an inexact rounding, etc.). They accumulate silently until you inspect or clear them.
2. **Rounding mode** — a control setting that tells the FPU (floating-point unit) how to round results that cannot be represented exactly. The default is "round to nearest, ties to even."

Beyond runtime control, the module also exposes **compile-time constants** and **type-querying templates** that describe the numeric limits of `float32` and `float64`.

### When would you use this module?

- **Numerical analysis and scientific computing** — detecting that an algorithm produced an intermediate overflow or underflow without the final result obviously being wrong.
- **Implementing correctly-rounded arithmetic** — temporarily switching to a different rounding mode (e.g. `FE_UPWARD`) to compute interval arithmetic bounds.
- **Defensive programming** — clearing exception flags before a computation block and checking them afterwards to confirm no anomalies occurred.
- **Writing generic numeric code** — querying `epsilon`, `maximumPositiveValue`, etc. instead of hard-coding magic numbers.

### Platform notes

- This module is a thin binding over C's `<fenv.h>`. It is available on all POSIX platforms and Windows (with an MSVC or MinGW C runtime). On POSIX (excluding macOS and Genode), `-lm` is linked automatically.
- The exact values of `FE_DIVBYZERO`, `FE_OVERFLOW`, etc. are platform-defined at C level; Nim imports them from the C header at compile time, so they are always correct for the target platform.
- Manipulating the floating-point environment is a **global, thread-local** operation. Changes affect all floating-point operations in the current thread.

---

## Exported API — Quick Reference

### Exception flag constants

| Constant | Meaning |
|----------|---------|
| `FE_DIVBYZERO` | A division by zero occurred |
| `FE_INEXACT` | The result was rounded (not exact) |
| `FE_INVALID` | An invalid operation occurred (e.g. `0.0 / 0.0`, `sqrt(-1)`) |
| `FE_OVERFLOW` | The result was too large to represent (became infinity) |
| `FE_UNDERFLOW` | The result was too small to represent (became zero or subnormal) |
| `FE_ALL_EXCEPT` | Bitwise OR of all the above — a convenient mask for "everything" |

### Rounding mode constants

| Constant | Rounding behaviour |
|----------|-------------------|
| `FE_TONEAREST` | Round to the nearest representable value (default, ties go to even) |
| `FE_DOWNWARD` | Round toward −∞ (always round down) |
| `FE_UPWARD` | Round toward +∞ (always round up) |
| `FE_TOWARDZERO` | Round toward zero (truncate, like integer division) |

### Environment constant

| Constant | Meaning |
|----------|---------|
| `FE_DFL_ENV` | A handle representing the default floating-point environment |

### Types

| Type | Meaning |
|------|---------|
| `Tfenv` | Opaque object that holds an entire saved FP environment |
| `Tfexcept` | Opaque object that holds a saved snapshot of exception flags |

### Exception-flag procedures

| Procedure | Purpose |
|-----------|---------|
| `feclearexcept` | Clear (reset) selected exception flags |
| `fegetexceptflag` | Save current exception flags into a `Tfexcept` |
| `fesetexceptflag` | Restore exception flags from a saved `Tfexcept` |
| `feraiseexcept` | Artificially raise (set) exception flags |
| `fetestexcept` | Test which exception flags are currently set |

### Rounding-mode procedures

| Procedure | Purpose |
|-----------|---------|
| `fegetround` | Read the current rounding mode |
| `fesetround` | Change the rounding mode |

### Full-environment procedures

| Procedure | Purpose |
|-----------|---------|
| `fegetenv` | Save the entire FP environment |
| `fesetenv` | Restore a previously saved FP environment |
| `feholdexcept` | Save environment, clear flags, enable non-stop mode |
| `feupdateenv` | Merge saved environment with currently pending exceptions |

### Numeric-property templates (float32)

`mantissaDigits`, `digits`, `minExponent`, `maxExponent`, `min10Exponent`, `max10Exponent`, `minimumPositiveValue`, `maximumPositiveValue`, `epsilon`

### Numeric-property templates (float64)

Same template names, overloaded for `float64`.

### Other templates

`fpRadix`

---

## Exception Flag Constants

Exception flags are individual bits. They are combined with bitwise OR to address multiple exceptions at once.

### `FE_DIVBYZERO`

Set when a floating-point operation divides a finite number by zero and the result is ±infinity. Ordinary integer division by zero is a different, non-FPU mechanism.

```nim
import std/fenv

discard feclearexcept(FE_ALL_EXCEPT)
let x = 1.0 / 0.0          # IEEE 754: result is +Inf, flag is set
doAssert fetestexcept(FE_DIVBYZERO) != 0
```

### `FE_INEXACT`

Set whenever the true mathematical result of an operation cannot be represented exactly and must be rounded. This flag fires extremely often — almost every floating-point multiplication or division sets it. It is mostly useful in high-precision numerical work where you want to confirm that a critical step was exact.

```nim
import std/fenv

discard feclearexcept(FE_INEXACT)
let _ = 1.0 / 3.0           # 1/3 is not representable exactly
doAssert fetestexcept(FE_INEXACT) != 0
```

### `FE_INVALID`

Set when an operation has no well-defined result in real-number arithmetic: `0.0 / 0.0`, `sqrt` of a negative number, `Inf - Inf`, etc. The result is NaN.

```nim
import std/fenv
import std/math

discard feclearexcept(FE_INVALID)
let _ = sqrt(-1.0)
doAssert fetestexcept(FE_INVALID) != 0
```

### `FE_OVERFLOW`

Set when the mathematical result is finite but too large in magnitude to be represented in the floating-point type; the result becomes ±Inf.

```nim
import std/fenv

discard feclearexcept(FE_OVERFLOW)
let _ = maximumPositiveValue(float32) * 2.0'f32
doAssert fetestexcept(FE_OVERFLOW) != 0
```

### `FE_UNDERFLOW`

Set when the mathematical result is nonzero but too small in magnitude to be represented as a normal floating-point number. The result is zero or a subnormal (denormalized) number with reduced precision.

```nim
import std/fenv

discard feclearexcept(FE_UNDERFLOW)
let _ = minimumPositiveValue(float32) / 1000.0'f32
doAssert fetestexcept(FE_UNDERFLOW) != 0
```

### `FE_ALL_EXCEPT`

The bitwise OR of all exception flags supported by the platform. Use it as a convenient shorthand when you want to operate on every exception at once.

```nim
import std/fenv

# Clear everything at once
discard feclearexcept(FE_ALL_EXCEPT)

# Test whether anything at all was raised
let anyRaised = fetestexcept(FE_ALL_EXCEPT)
```

---

## Rounding Mode Constants

IEEE 754 requires that every floating-point operation produce the correctly-rounded result for the active rounding mode. Changing the mode affects all subsequent FP operations in the thread.

### `FE_TONEAREST`

The default mode on virtually all systems. When the true result lies exactly halfway between two representable values, the result is rounded to the one with an even last bit ("round half to even" or "banker's rounding"). This minimises statistical bias over many operations.

```nim
import std/fenv

doAssert fegetround() == FE_TONEAREST   # true on a fresh process
```

### `FE_DOWNWARD`

Every result is rounded toward −∞. `2.9` becomes `2.0`, `−2.1` becomes `−3.0`. Used in interval arithmetic to compute a guaranteed lower bound.

### `FE_UPWARD`

Every result is rounded toward +∞. `2.1` becomes `3.0`, `−2.9` becomes `−2.0`. Used in interval arithmetic to compute a guaranteed upper bound.

### `FE_TOWARDZERO`

Every result is rounded toward zero — equivalent to truncation. `2.9` becomes `2.0`, `−2.9` becomes `−2.0`.

---

## Types

### `Tfenv`

```nim
type Tfenv {.importc: "fenv_t", ...} = object
```

An opaque object that captures the **complete** floating-point environment: both the rounding mode and all exception flags. Used as the target of `fegetenv` / `fesetenv` / `feholdexcept` / `feupdateenv`. You cannot inspect individual fields — treat it as a black-box snapshot.

### `Tfexcept`

```nim
type Tfexcept {.importc: "fexcept_t", ...} = object
```

An opaque object that captures only the **exception flag** part of the FP environment. Used with `fegetexceptflag` and `fesetexceptflag`. Like `Tfenv`, treat it as opaque — platform internals may store extra state alongside the flags.

---

## Exception-Flag Procedures

### `feclearexcept`

```nim
proc feclearexcept*(excepts: cint): cint
```

Clears (resets to zero) the exception flags specified by `excepts`. Returns zero on success. Call this before a computation block when you want to check whether that specific block raised any exceptions.

```nim
import std/fenv

discard feclearexcept(FE_ALL_EXCEPT)   # start with a clean slate
# ... perform computation ...
if fetestexcept(FE_OVERFLOW or FE_INVALID) != 0:
  echo "Something went wrong!"
```

---

### `fegetexceptflag`

```nim
proc fegetexceptflag*(flagp: ptr Tfexcept, excepts: cint): cint
```

Saves a snapshot of the exception flags indicated by `excepts` into the `Tfexcept` object pointed to by `flagp`. Returns zero on success. This is a lower-level save than `fegetenv` — it captures only the requested flags, not the rounding mode.

```nim
import std/fenv

var saved: Tfexcept
discard fegetexceptflag(addr saved, FE_ALL_EXCEPT)
# ... do something that may change flags ...
discard fesetexceptflag(addr saved, FE_ALL_EXCEPT)   # restore
```

---

### `fesetexceptflag`

```nim
proc fesetexceptflag*(flagp: ptr Tfexcept, excepts: cint): cint
```

Restores the exception flags specified by `excepts` from the snapshot in `flagp`. Importantly, this function **does not raise the floating-point exceptions** as signals — it simply sets the flag bits. If you need to actually trigger a hardware exception/signal, use `feraiseexcept` instead. Returns zero on success.

```nim
import std/fenv

var saved: Tfexcept
discard fegetexceptflag(addr saved, FE_OVERFLOW)
# clear the flag, do work, then restore original state:
discard feclearexcept(FE_OVERFLOW)
# ... work ...
discard fesetexceptflag(addr saved, FE_OVERFLOW)
```

---

### `feraiseexcept`

```nim
proc feraiseexcept*(excepts: cint): cint
```

Artificially raises (triggers) the floating-point exceptions specified by `excepts`, as if the corresponding exceptional arithmetic had actually occurred. This may invoke a signal handler if the exception is unmasked. Returns zero on success.

Primarily useful in testing and in library code that needs to propagate exception state through a section that temporarily suppressed exceptions (e.g. after `feholdexcept`).

```nim
import std/fenv

discard feclearexcept(FE_ALL_EXCEPT)
discard feraiseexcept(FE_DIVBYZERO or FE_OVERFLOW)
doAssert fetestexcept(FE_DIVBYZERO) != 0
doAssert fetestexcept(FE_OVERFLOW)  != 0
```

---

### `fetestexcept`

```nim
proc fetestexcept*(excepts: cint): cint
```

Tests which of the flags specified by `excepts` are currently set. Returns a bitmask of the flags that are set (a subset of `excepts`). Returns zero if none of the requested flags are set.

```nim
import std/fenv

discard feclearexcept(FE_ALL_EXCEPT)
let _ = 1.0 / 0.0

let flags = fetestexcept(FE_ALL_EXCEPT)
if (flags and FE_DIVBYZERO) != 0: echo "Division by zero occurred"
if (flags and FE_OVERFLOW)  != 0: echo "Overflow occurred"
```

---

## Rounding-Mode Procedures

### `fegetround`

```nim
proc fegetround*(): cint
```

Returns the identifier of the currently active rounding mode. The return value is one of the `FE_*` rounding constants. Useful for saving the rounding mode before changing it so you can restore it afterwards.

```nim
import std/fenv

let original = fegetround()
echo original == FE_TONEAREST   # typically true
```

---

### `fesetround`

```nim
proc fesetround*(roundingDirection: cint): cint
```

Sets the active rounding mode to `roundingDirection`. Returns zero on success. Always restore the original mode after you are done — leaving a non-default rounding mode active is a common source of subtle bugs.

```nim
import std/fenv

let saved = fegetround()

discard fesetround(FE_UPWARD)
let upper = 1.0 / 3.0   # rounded toward +Inf

discard fesetround(FE_DOWNWARD)
let lower = 1.0 / 3.0   # rounded toward -Inf

discard fesetround(saved)   # always restore!
echo "Interval: [", lower, ", ", upper, "]"
```

---

## Full-Environment Procedures

### `fegetenv`

```nim
proc fegetenv*(envp: ptr Tfenv): cint
```

Saves the **complete** floating-point environment (rounding mode + all exception flags) into the `Tfenv` object at `envp`. Returns zero on success. This is the heaviest-weight save operation — use it when you need to preserve and restore everything.

```nim
import std/fenv

var env: Tfenv
discard fegetenv(addr env)
# ... modify environment ...
discard fesetenv(addr env)   # restore everything
```

---

### `fesetenv`

```nim
proc fesetenv*(a1: ptr Tfenv): cint
```

Restores the complete floating-point environment from the `Tfenv` object. Unlike `feupdateenv`, this **discards** any exception flags that were raised between the save and the restore. Returns zero on success.

The special constant `FE_DFL_ENV` can be passed (cast to `ptr Tfenv`) to reset to the process-startup default environment.

```nim
import std/fenv

var env: Tfenv
discard fegetenv(addr env)
discard fesetround(FE_DOWNWARD)
# ... work ...
discard fesetenv(addr env)   # flags raised during work are discarded
```

---

### `feholdexcept`

```nim
proc feholdexcept*(envp: ptr Tfenv): cint
```

A combined operation that:
1. Saves the current environment into `envp` (like `fegetenv`).
2. Clears all exception flags.
3. Installs a "non-stop" mode where exceptions are masked (no signals/traps).

Returns zero on success. It is the standard preamble for a section of code that wants to perform floating-point operations without interruption and then inspect the flags at the end. Pair it with `feupdateenv` to propagate any exceptions that occurred.

```nim
import std/fenv

var saved: Tfenv
discard feholdexcept(addr saved)   # save + clear + mask

let _ = 1.0 / 0.0   # FE_DIVBYZERO flag set, but no signal

discard feupdateenv(addr saved)    # restore env and re-raise flags
```

---

### `feupdateenv`

```nim
proc feupdateenv*(envp: ptr Tfenv): cint
```

A combined operation that:
1. Saves the currently pending exception flags in a temporary location.
2. Restores the environment from `envp` (like `fesetenv`).
3. Re-raises the temporarily saved flags (like `feraiseexcept`).

This is the complement of `feholdexcept`. The effect is: restore the old environment but do not lose any exceptions that were raised during the non-stop section. Returns zero on success.

```nim
import std/fenv

var saved: Tfenv
discard feholdexcept(addr saved)

# Perform computation — exceptions accumulate in flags silently
let a = 1.0 / 0.0   # sets FE_DIVBYZERO internally

# Restore original env; FE_DIVBYZERO is raised in the now-restored env
discard feupdateenv(addr saved)
doAssert fetestexcept(FE_DIVBYZERO) != 0
```

---

## Numeric-Property Templates

These templates are evaluated at compile time and return the numeric limits of the floating-point types. They take a `typedesc` (`float32` or `float64`) as their argument, making them generic and self-documenting.

### `fpRadix`

```nim
template fpRadix*: int
```

The base (radix) of the floating-point representation used by this architecture. On all modern hardware this is `2` (binary). It is a property of the hardware, not of a specific type, so it takes no argument.

```nim
import std/fenv
doAssert fpRadix == 2
```

---

### `mantissaDigits`

```nim
template mantissaDigits*(T: typedesc[float32 | float64]): int
```

The number of base-`fpRadix` digits in the significand (mantissa). For `float32` this is 24 (one implicit leading 1 plus 23 stored bits); for `float64` it is 53. This tells you the precision of the type in its native base.

```nim
import std/fenv
doAssert mantissaDigits(float32) == 24
doAssert mantissaDigits(float64) == 53
```

---

### `digits`

```nim
template digits*(T: typedesc[float32 | float64]): int
```

The number of **decimal** digits of precision — how many decimal digits you can round-trip through the type without losing information. `float32` gives 6, `float64` gives 15. Useful when formatting floating-point output.

```nim
import std/fenv
doAssert digits(float32) == 6
doAssert digits(float64) == 15
```

---

### `minExponent` and `maxExponent`

```nim
template minExponent*(T: typedesc[float32 | float64]): int
template maxExponent*(T: typedesc[float32 | float64]): int
```

The minimum and maximum exponent of a normalised floating-point number, expressed as a power of `fpRadix` (i.e. base 2). For `float32`: min = −125, max = 128. For `float64`: min = −1021, max = 1024.

```nim
import std/fenv
doAssert minExponent(float32) == -125
doAssert maxExponent(float64) == 1024
```

---

### `min10Exponent` and `max10Exponent`

```nim
template min10Exponent*(T: typedesc[float32 | float64]): int
template max10Exponent*(T: typedesc[float32 | float64]): int
```

The same exponent limits expressed in base 10. For `float32`: −37 to 38. For `float64`: −307 to 308. Corresponds directly to the range printed in scientific notation.

```nim
import std/fenv
doAssert min10Exponent(float64) == -307
doAssert max10Exponent(float64) == 308
```

---

### `minimumPositiveValue`

```nim
template minimumPositiveValue*(T: typedesc[float32]): float32
template minimumPositiveValue*(T: typedesc[float64]): float64
```

The smallest **positive normalised** floating-point number representable in the type. Values smaller than this are subnormal (denormalised) and have reduced precision; values smaller than the smallest subnormal become zero (underflow). For `float32` this is approximately `1.18e-38`; for `float64` it is approximately `2.23e-308`.

```nim
import std/fenv
echo minimumPositiveValue(float32)   # 1.1754944e-38
echo minimumPositiveValue(float64)   # 2.2250738585072014e-308
```

---

### `maximumPositiveValue`

```nim
template maximumPositiveValue*(T: typedesc[float32]): float32
template maximumPositiveValue*(T: typedesc[float64]): float64
```

The largest finite positive number representable in the type. Values exceeding this overflow to `+Inf`. For `float32` this is approximately `3.40e+38`; for `float64` approximately `1.80e+308`.

```nim
import std/fenv
echo maximumPositiveValue(float32)   # 3.4028235e+38
echo maximumPositiveValue(float64)   # 1.7976931348623157e+308
```

---

### `epsilon`

```nim
template epsilon*(T: typedesc[float32]): float32
template epsilon*(T: typedesc[float64]): float64
```

The **machine epsilon**: the difference between `1.0` and the next larger representable value. It is the smallest `e` such that `1.0 + e != 1.0`. For `float32`: approximately `1.19e-7`; for `float64`: approximately `2.22e-16`.

Machine epsilon is fundamental in numerical analysis — it defines the relative precision of the type and serves as a tolerance in floating-point equality comparisons.

```nim
import std/fenv

# Correct way to compare floats for near-equality:
proc nearlyEqual(a, b: float64): bool =
  abs(a - b) < epsilon(float64) * max(abs(a), abs(b))

doAssert nearlyEqual(0.1 + 0.2, 0.3)
```

---

## Complete Practical Examples

### Pattern 1 — Check a computation block for exceptions

```nim
import std/fenv
import std/math

proc safeCompute(x: float64): float64 =
  discard feclearexcept(FE_ALL_EXCEPT)

  result = sqrt(x) * log(x + 1.0)

  let flags = fetestexcept(FE_ALL_EXCEPT)
  if (flags and FE_INVALID)    != 0: echo "Warning: invalid operation"
  if (flags and FE_DIVBYZERO)  != 0: echo "Warning: division by zero"
  if (flags and FE_OVERFLOW)   != 0: echo "Warning: overflow"

echo safeCompute(4.0)    # clean
echo safeCompute(-1.0)   # triggers FE_INVALID (log of negative)
```

---

### Pattern 2 — Interval arithmetic with directed rounding

```nim
import std/fenv

proc intervalAdd(aLo, aHi, bLo, bHi: float64): (float64, float64) =
  let saved = fegetround()

  discard fesetround(FE_DOWNWARD)
  let lo = aLo + bLo

  discard fesetround(FE_UPWARD)
  let hi = aHi + bHi

  discard fesetround(saved)
  (lo, hi)

let (lo, hi) = intervalAdd(0.1, 0.1, 0.2, 0.2)
echo "0.1 + 0.2 is guaranteed in [", lo, ", ", hi, "]"
```

---

### Pattern 3 — Non-stop section with `feholdexcept` / `feupdateenv`

```nim
import std/fenv

var savedEnv: Tfenv
discard feholdexcept(addr savedEnv)   # mute all exceptions

# This block may produce exceptional results, but won't trap/signal
var acc = 0.0
for i in 1..1000:
  acc += 1.0 / float(i)   # FE_INEXACT accumulates silently

discard feupdateenv(addr savedEnv)    # restore and re-raise accumulated flags

if fetestexcept(FE_INEXACT) != 0:
  echo "Results were rounded during summation"
```

---

### Pattern 4 — Querying type properties generically

```nim
import std/fenv

proc describeType(T: typedesc) =
  echo "Type: ", $T
  echo "  Decimal digits:  ", digits(T)
  echo "  Machine epsilon: ", epsilon(T)
  echo "  Max value:       ", maximumPositiveValue(T)

describeType(float32)
describeType(float64)
```

---

## Summary of Return Values

All `fe*` procedures return `cint`. The convention (inherited from C) is:

| Return value | Meaning |
|-------------|---------|
| `0` | Success |
| Non-zero | Failure (the operation is not supported or failed) |

Always check the return value in production code. On most platforms all operations succeed, but portability requires the check.

---

## See Also

- [C `<fenv.h>` reference (cppreference)](https://en.cppreference.com/w/c/numeric/fenv)
- [`std/math`](https://nim-lang.org/docs/math.html) — mathematical functions (`sqrt`, `log`, `isNaN`, `isInf`, etc.)
- [IEEE 754 standard](https://en.wikipedia.org/wiki/IEEE_754) — the underlying floating-point standard
