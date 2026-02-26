# `strmantle.nim` — Module Reference

> **Part of Nim's Runtime Library**  
> Internal compiler support module — not imported directly by user code.

---

## Overview

`strmantle.nim` is a **low-level runtime support module for string and character operations** in Nim. The name "mantle" reflects its role: it wraps and supports the core string infrastructure, sitting between the high-level string API and the machine.

Every procedure here is either `{.compilerproc.}` or `{.compilerRtl.}` — meaning the **Nim compiler emits calls to these functions automatically** when it compiles ordinary string operations like `==`, `<`, `<=`, or string-to-bool/char conversions. You never call them yourself; they are invoked implicitly on your behalf.

The module also contains the **canonical float-parsing routine** `nimParseBiggestFloat`, which all of Nim's numeric parsing ultimately depends on.

### Key design constraints

- Zero dependencies on Nim's higher-level standard library — only on `c_memcmp`, `c_strcmp`, `equalMem`, and a single C import (`strtod`).
- All comparison and hashing procedures use exactly the same algorithms the **compiler itself** uses when generating optimised `case` statements and comparisons for string literals. The comment *"the compiler needs exactly the same hash function!"* appears twice and is not an exaggeration.
- The float parser implements a **two-path architecture**: a fast path for numbers representable in 53 bits, and a slow path that delegates to C's `strtod` for everything else.

---

## Internal Constant: `IdentChars`

```nim
const IdentChars = {'a'..'z', 'A'..'Z', '0'..'9', '_'}
```

Not exported. A character set used by `nimParseBiggestFloat` to detect where a special literal like `NaN` or `Inf` ends and ensure it is not embedded in the middle of an identifier (e.g. `NaNometer` should not parse as `NaN`).

---

## Internal Constant: `powtens`

```nim
const powtens = [1e0, 1e1, 1e2, ..., 1e22]
```

Not exported. A lookup table of powers of ten from 10⁰ to 10²², used by the fast path of `nimParseBiggestFloat`. The upper bound of 10²² is precise: it is the largest power of ten that can be represented exactly in a 64-bit IEEE 754 double. Beyond it, exact multiplication is no longer possible and the slow `strtod` path must be used.

---

## String Comparison Procedures

These four procedures form the backbone of all Nim string comparisons. They are `{.compilerproc.}`: the compiler generates calls to them for `==`, `<`, and `<=` on `string` values.

---

### `cmpStrings`

```nim
proc cmpStrings(a, b: string): int {.inline, compilerproc.}
```

**The fundamental string comparison.** Returns a negative integer if `a < b`, zero if `a == b`, and a positive integer if `a > b`. All other string comparison functions (`leStrings`, `ltStrings`, `eqStrings`) are built on top of this one.

The algorithm follows the classic C `strcmp` contract with one important extension: it correctly handles strings that contain embedded null bytes (`\0`), because it compares by **length-prefixed byte sequences** rather than null-termination.

**How it works:**

1. Compute `minlen = min(a.len, b.len)`.
2. If `minlen > 0`, use `c_memcmp` to compare the first `minlen` bytes. This is a single, highly optimised machine instruction on most platforms.
3. If the byte comparison returns `0` (all shared bytes are equal), the result is `a.len - b.len`. A shorter string sorts before a longer one.
4. If either string is empty (`minlen == 0`), go directly to the length difference.

```nim
# These are the relationships the compiler uses:
# "abc" vs "abd"  → memcmp finds 'c' < 'd' → negative result
# "abc" vs "abcd" → memcmp finds 3 equal bytes → 3 - 4 = -1 (a is shorter)
# "abc" vs "abc"  → memcmp finds 3 equal bytes → 3 - 3 = 0 (equal)
```

---

### `leStrings`

```nim
proc leStrings(a, b: string): bool {.inline, compilerproc.}
```

Returns `true` if `a <= b` lexicographically. A thin wrapper: `cmpStrings(a, b) <= 0`.

The comment *"required by upcoming backends (NIR)"* indicates this was explicitly factored out as a named symbol so that Nim's new IR-based backend (NIR) can reference it directly by name rather than reconstructing the logic inline.

```nim
# The compiler translates:
#   if s1 <= s2: ...
# into a call to leStrings(s1, s2)
```

---

### `ltStrings`

```nim
proc ltStrings(a, b: string): bool {.inline, compilerproc.}
```

Returns `true` if `a < b` lexicographically. A thin wrapper: `cmpStrings(a, b) < 0`.

Same NIR motivation as `leStrings`. Together, `ltStrings` and `leStrings` give the backend all the comparison predicates it needs without requiring it to know about `cmpStrings` directly.

```nim
# The compiler translates:
#   if s1 < s2: ...
# into a call to ltStrings(s1, s2)
```

---

### `eqStrings`

```nim
proc eqStrings(a, b: string): bool {.inline, compilerproc.}
```

Returns `true` if `a` and `b` have identical content. Optimised specifically for the equality case — faster than `cmpStrings(a, b) == 0` because it takes a shortcut:

1. If `a.len != b.len` → immediately `false`. No byte comparison needed.
2. If `a.len == 0` → immediately `true`. Two empty strings are always equal.
3. Otherwise → `equalMem`, which compares raw bytes.

This three-step structure means **inequality is detected in O(1)** for any two strings of different lengths, regardless of content.

```nim
# The compiler translates:
#   if s1 == s2: ...
# into a call to eqStrings(s1, s2)
```

---

## String Hashing Procedures

### `hashString`

```nim
proc hashString(s: string): int {.compilerproc.}
```

Computes an integer hash of a Nim `string`. This is the hash function used by the compiler when generating **efficient `case` statements over string literals**. Instead of emitting a chain of `if s == "x" elif s == "y"...`, the compiler can emit a hash-based dispatch table. The compiled hash must match this function exactly, or the dispatch table would be wrong.

**The algorithm** is a variant of the Jenkins one-at-a-time hash:

```
for each character c:
    h += c
    h += h << 10
    h ^= h >> 6
# finalisation:
h += h << 3
h ^= h >> 11
h += h << 15
```

This produces a good avalanche effect (small input changes produce large output changes) with minimal code, making it suitable for hash tables and switch dispatch.

```nim
# Used implicitly when you write:
case myString
of "apple":  ...
of "banana": ...
of "cherry": ...
# The compiler hashes each literal at compile time and the runtime
# hashes the input value once, then dispatches by hash bucket.
```

---

### `hashCstring`

```nim
proc hashCstring(s: cstring): int {.compilerproc.}
```

The same Jenkins one-at-a-time hash as `hashString`, but for null-terminated `cstring` values. Key differences:

- Handles `nil` explicitly: a nil `cstring` hashes to `0`.
- Iterates by index until `'\0'` instead of using a length field.
- Used by the compiler when generating `case` statements over `cstring` literals.

```nim
# A nil cstring is handled safely:
# hashCstring(nil) → 0
```

The hash algorithm is byte-for-byte identical to `hashString` — two strings with the same content will always produce the same hash whether they are `string` or `cstring`.

---

## C-String Equality

### `eqCstrings`

```nim
proc eqCstrings(a, b: cstring): bool {.inline, compilerproc.}
```

Compares two `cstring` values for equality with careful nil-handling — something raw `c_strcmp` cannot do, because passing `nil` to `strcmp` is undefined behaviour in C.

The logic has three cases, checked in order:

1. `pointer(a) == pointer(b)` → both point to the same memory (or are both nil) → `true`. This is a fast pointer equality check.
2. `a.isNil or b.isNil` → exactly one is nil, the other is not → `false`.
3. Otherwise → `c_strcmp(a, b) == 0`, which compares byte by byte until a null terminator.

```nim
# Safe comparison that won't crash on nil:
var p: cstring = nil
var q: cstring = "hello"
# eqCstrings(p, q) → false  (no crash)
# eqCstrings(nil, nil) → true
# eqCstrings("hi", "hi") → true (same content, possibly different pointers)
```

---

## Float Parsing

### `nimParseBiggestFloat`

```nim
proc nimParseBiggestFloat(s: openArray[char], number: var BiggestFloat): int {.compilerproc.}
```

The **central floating-point parsing routine** for all of Nim's standard library. `parseFloat`, `parseBiggestFloat`, and the string-to-float conversion in `strconv` all ultimately call this. Returns the number of characters consumed from `s`, or `0` on parse error. The parsed value is written to the `number` out-parameter.

This is a sophisticated procedure with several distinct stages. Understanding it fully explains *why* Nim's float parsing is both fast and correct across platforms.

#### Stage 1 — Sign

Reads an optional leading `+` or `-`. Sets the sign multiplier.

#### Stage 2 — Special values (NaN and Inf)

Recognises `NaN`/`nan` and `Inf`/`inf` (case-insensitive). The boundary check against `IdentChars` is essential: it ensures `NaNometer` is rejected (the `N` that follows is an ident char) while `NaN` standing alone or followed by whitespace/punctuation is accepted.

```nim
var f: float
discard nimParseBiggestFloat("NaN", f)   # f = NaN,  returns 3
discard nimParseBiggestFloat("inf", f)   # f = +Inf, returns 3
discard nimParseBiggestFloat("-Inf", f)  # f = -Inf, returns 4
```

#### Stage 3 — Integer and fractional digit accumulation

Reads digits before and after the decimal point into a single `uint64` accumulator `integer`. Underscore separators (Nim's `1_000_000` style) are silently skipped. The number of integer digits (`kdigits`) and fractional digits (`fdigits`) are counted separately to compute the fractional exponent later.

#### Stage 4 — Exponent

Reads an optional `e` or `E` followed by an optional sign and decimal digits, again allowing underscores.

#### Stage 5 — Fast path (≤ 15–16 significant digits, |exponent| ≤ 22)

If the number fits in 53 bits (at most 15 significant digits, or 16 digits starting with `0`–`8`) **and** the final adjusted exponent is within `[-22, +22]`, the result can be computed exactly using integer arithmetic and the `powtens` lookup table:

```
number = sign × integer × 10^exponent   (using powtens[exponent])
```

This is the "fast path" described in the [Exploringbinary.com article](https://www.exploringbinary.com/fast-path-decimal-to-floating-point-conversion/) referenced in the source.

There is also a slight extension: if the exponent is just above 22, the procedure tries to absorb the excess into the integer part by multiplying it up, using the remaining "slop" digits of precision.

#### Stage 6 — Slow path (all other cases)

If the fast path cannot be used (too many digits, exponent too large/small), the procedure:

1. Re-encodes the entire number without decimal point, underscores, or sign into a temporary 500-character buffer `t` in the form `MANTISSA E±EXP`.
2. Calls C's `strtod` on that buffer.

This delegation to `strtod` handles the hard cases correctly and portably — the comment explains that this avoids decimal character portability problems (different C libraries may use different decimal separators in `strtod` depending on locale, but the procedure always constructs the buffer with `.` or `E` in ASCII).

#### Exponent overflow/underflow

Before the fast/slow path decision, if `absExponent > 999`:
- If `integer == 0` → result is `0.0` (zero × anything = zero).
- If exponent is negative → result is `0.0 * sign` (underflow to zero).
- If exponent is positive → result is `Inf * sign` (overflow to infinity).

```nim
var f: float
discard nimParseBiggestFloat("1.5e10", f)      # fast path: f = 15000000000.0
discard nimParseBiggestFloat("1.23456789012345678", f)  # slow path (17 digits)
discard nimParseBiggestFloat("1e9999", f)      # overflow: f = +Inf
discard nimParseBiggestFloat("1e-9999", f)     # underflow: f = 0.0
discard nimParseBiggestFloat("3.14_159", f)    # underscores OK: f ≈ 3.14159
discard nimParseBiggestFloat("abc", f)         # error: returns 0
```

The `{.push staticBoundChecks: off.}` / `{.pop.}` pragma pair around this procedure disables Nim's static array bounds checker for the `powtens` lookup. The index is always within `[0, 22]` by construction, so the check is provably unnecessary and would only slow down the inner loop.

---

## String/Char Conversion Procedures

### `nimBoolToStr`

```nim
proc nimBoolToStr(x: bool): string {.compilerRtl.}
```

Converts a `bool` to its canonical string representation. Returns `"true"` for `true` and `"false"` for `false`. This is the function the compiler calls when you write `$myBool` or interpolate a bool in a format string.

Marked `{.compilerRtl.}` (compiler runtime library) rather than `{.compilerproc.}` — a subtle distinction: `compilerRtl` symbols are part of the runtime ABI and may be called from generated C code by name, while `compilerproc` symbols may be inlined or substituted by the compiler more aggressively.

```nim
# The compiler translates:
#   let s = $true
# into a call to nimBoolToStr(true) → "true"
```

---

### `nimCharToStr`

```nim
proc nimCharToStr(x: char): string {.compilerRtl.}
```

Converts a single `char` to a one-character `string`. Allocates a new `string` of length 1 and assigns the character to `result[0]`. This is the function the compiler calls when you write `$myChar`.

```nim
# The compiler translates:
#   let s = $'A'
# into a call to nimCharToStr('A') → "A"
```

---

## Memory Statistics (ARC/ORC only)

### `GC_getStatistics*`

```nim
proc GC_getStatistics*(): string
```

*Only compiled when `-d:gcDestructors` is active (i.e. when using ARC or ORC memory management).*

Returns a human-readable multi-line string with basic memory statistics:

```
[GC] total memory: 1048576
[GC] occupied memory: 204800
```

Uses `getTotalMem()` and `getOccupiedMem()` from the memory manager. The commented-out line (`[GC] cycle collections:`) suggests that cycle collection statistics may be added in the future.

This is one of the few **exported** (`*`) procedures in this module — it forms part of Nim's public GC statistics API that is accessible to ordinary user code for diagnostics and memory profiling.

```nim
when defined(gcDestructors):
  echo GC_getStatistics()
  # [GC] total memory: ...
  # [GC] occupied memory: ...
```

---

## Summary Table

| Symbol | Kind | Exported | Trigger |
|---|---|---|---|
| `cmpStrings` | `compilerproc` | No | `cmp(s1, s2)`, internal use |
| `leStrings` | `compilerproc` | No | `s1 <= s2` on strings |
| `ltStrings` | `compilerproc` | No | `s1 < s2` on strings |
| `eqStrings` | `compilerproc` | No | `s1 == s2` on strings |
| `hashString` | `compilerproc` | No | `case` on string literals |
| `hashCstring` | `compilerproc` | No | `case` on cstring literals |
| `eqCstrings` | `compilerproc` | No | `cs1 == cs2` on cstrings |
| `nimParseBiggestFloat` | `compilerproc` | No | `parseFloat`, `parseBiggestFloat`, `$` on floats |
| `nimBoolToStr` | `compilerRtl` | No | `$myBool` |
| `nimCharToStr` | `compilerRtl` | No | `$myChar` |
| `GC_getStatistics` | proc | **Yes** | User-callable (ARC/ORC only) |
