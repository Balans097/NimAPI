# `formatfloat` Module Reference

> **Module:** `std/formatfloat` (Nim standard library)  
> **Purpose:** Converting floating-point numbers (`float`, `float32`, `BiggestFloat`) to their string representations.

This module provides a layered set of tools — from low-level buffer-writing routines to convenient string-appending helpers — all centered on one problem: how do you correctly and efficiently turn a floating-point number into human-readable text?

The module offers two underlying strategies:

- **Roundtrip** (default) — uses the Dragonbox / Schubfach algorithms to produce the *shortest* decimal string that, when parsed back, yields exactly the same IEEE 754 bit pattern. This is the modern, preferred approach.
- **Sprintf** (legacy, `nimLegacySprintf` flag) — delegates to the C standard library's `snprintf` with `%.16g` format, then patches locale-dependent quirks (commas instead of dots) and platform-specific infinity/NaN spellings.

---

## Exported Functions

---

### `writeFloatToBufferRoundtrip` *(two overloads)*

```nim
proc writeFloatToBufferRoundtrip*(buf: var array[65, char]; value: BiggestFloat): int
proc writeFloatToBufferRoundtrip*(buf: var array[65, char]; value: float32): int
```

**What it does**

Writes a floating-point number into a fixed-size `array[65, char]` buffer using the roundtrip-safe Dragonbox (for `float64`/`BiggestFloat`) or Schubfach (for `float32`) algorithm. A null terminator `'\0'` is always appended. Returns the number of characters written, *not* counting the terminator.

The "roundtrip" guarantee means: if you parse the resulting string back with any standard parser, you get the *exact same* float. No precision is lost, and the output is as short as possible.

**When to use it**

Use this when you need raw, allocation-free float serialization — for example, inside a custom serializer, a logger, or anywhere you want to avoid heap allocation.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `buf` | `var array[65, char]` | Output buffer. 65 characters is always enough for any float. Passed by reference and modified in place. |
| `value` | `BiggestFloat` or `float32` | The number to convert. |

**Returns:** `int` — number of characters written (excluding the null terminator).

**Example**

```nim
import std/formatfloat

var buf: array[65, char]

# float64 overload
let n1 = writeFloatToBufferRoundtrip(buf, 3.14159265358979)
echo cast[cstring](buf[0].addr)  # "3.14159265358979"
echo n1                          # 16

# float32 overload
let n2 = writeFloatToBufferRoundtrip(buf, 1.5'f32)
echo cast[cstring](buf[0].addr)  # "1.5"
echo n2                          # 3
```

The roundtrip property in action:

```nim
import std/strutils

var buf: array[65, char]
let original = 0.1 + 0.2            # 0.30000000000000004 in IEEE 754
discard writeFloatToBufferRoundtrip(buf, original)
let s = $cast[cstring](buf[0].addr) # "0.30000000000000004"
assert parseFloat(s) == original    # true — exact roundtrip
```

---

### `writeFloatToBufferSprintf`

```nim
proc writeFloatToBufferSprintf*(buf: var array[65, char]; value: BiggestFloat): int
```

**What it does**

Writes a `BiggestFloat` into the buffer using the C standard library's `snprintf` with the `"%.16g"` format specifier. This gives up to 16 significant decimal digits. After formatting, the function performs several normalization steps:

1. Replaces any locale-introduced comma separators (`','`) with a dot (`'.'`), ensuring the output is always locale-independent.
2. Detects whether the result contains a decimal point or exponent marker; if not, appends `.0` so that the string always *looks like* a float (not an integer).
3. Normalizes platform-specific infinity and NaN representations: Windows may produce `1.#INF`, `-1.#IND`, `nan(ind)` etc. — all are normalized to `inf`, `-inf`, or `nan`.

**When to use it**

You generally don't call this directly. It is the backend used when the `nimLegacySprintf` compile-time flag is set. It exists for compatibility with older code and environments where the Dragonbox/Schubfach algorithms are unavailable.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `buf` | `var array[65, char]` | Output buffer, modified in place. |
| `value` | `BiggestFloat` | The float64 value to format. |

**Returns:** `int` — number of characters written (excluding null terminator).

**Example**

```nim
# This function is normally used indirectly via writeFloatToBuffer
# when compiled with -d:nimLegacySprintf.
# Direct usage:
var buf: array[65, char]
let n = writeFloatToBufferSprintf(buf, 42.0)
echo cast[cstring](buf[0].addr)  # "42.0"
echo n                           # 4

let n2 = writeFloatToBufferSprintf(buf, 1.23e10)
echo cast[cstring](buf[0].addr)  # "1.23e+10"  (platform-dependent notation)

# Special value handling:
let n3 = writeFloatToBufferSprintf(buf, 1.0 / 0.0)
echo cast[cstring](buf[0].addr)  # "inf"

let n4 = writeFloatToBufferSprintf(buf, 0.0 / 0.0)
echo cast[cstring](buf[0].addr)  # "nan"
```

---

### `writeFloatToBuffer`

```nim
proc writeFloatToBuffer*(buf: var array[65, char]; value: BiggestFloat | float32): int {.inline.}
```

**What it does**

A unified, compile-time-dispatched entry point for writing a float to a buffer. Depending on whether the program was compiled with `-d:nimLegacySprintf`, it calls either `writeFloatToBufferSprintf` (legacy) or `writeFloatToBufferRoundtrip` (default). Accepts both `BiggestFloat` and `float32` via a type union.

Think of this as *the* canonical low-level float-to-buffer function — it hides the backend selection from you. Use this instead of calling the roundtrip or sprintf variants directly, unless you specifically need one of them.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `buf` | `var array[65, char]` | Output buffer, modified in place. |
| `value` | `BiggestFloat` or `float32` | The number to format. |

**Returns:** `int` — number of characters written (excluding null terminator).

**Example**

```nim
import std/formatfloat

var buf: array[65, char]

let n = writeFloatToBuffer(buf, 2.718281828)
echo cast[cstring](buf[0].addr)  # "2.718281828"

# Works with float32 too:
let n2 = writeFloatToBuffer(buf, 0.5'f32)
echo cast[cstring](buf[0].addr)  # "0.5"
```

---

### `addFloatRoundtrip`

```nim
proc addFloatRoundtrip*(result: var string; x: float | float32)
```

**What it does**

Appends the roundtrip representation of `x` to an existing `string`. This is the string-level companion to `writeFloatToBufferRoundtrip`. Internally it allocates a stack buffer, converts the number, and then copies the characters into the string — so the string grows by exactly the right amount, with no wasted space.

Not available in `nimvm` (compile-time VM) contexts; calling it there raises an assertion error.

**When to use it**

Use this when you want to build a string incrementally and need to embed a float with maximum precision and shortest representation. It's slightly more explicit than `addFloat` — you know you're getting the roundtrip backend regardless of compile flags.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `result` | `var string` | The string to append to. |
| `x` | `float` or `float32` | The number to convert and append. |

**Example**

```nim
import std/formatfloat

var s = "value="
s.addFloatRoundtrip(0.1)
echo s  # "value=0.1"

var s2 = "pi≈"
s2.addFloatRoundtrip(3.141592653589793)
echo s2  # "pi≈3.141592653589793"

# Building a list of numbers:
var csv = ""
for v in [1.0, 2.5, 0.1 + 0.2]:
  if csv.len > 0: csv.add(',')
  csv.addFloatRoundtrip(v)
echo csv  # "1.0,2.5,0.30000000000000004"
```

---

### `addFloatSprintf`

```nim
proc addFloatSprintf*(result: var string; x: float)
```

**What it does**

Appends the `snprintf`-based representation of `x` to an existing string. This is the string-level companion to `writeFloatToBufferSprintf`. Like its buffer counterpart, it normalizes locale quirks and platform-specific special-value spellings.

Not available in `nimvm` contexts.

**When to use it**

Primarily a legacy/compatibility function. You would call this if you specifically want the `%.16g` C formatting behavior regardless of compile flags — for example, to match output from an older version of the code. In new code, prefer `addFloat` or `addFloatRoundtrip`.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `result` | `var string` | The string to append to. |
| `x` | `float` | The number to convert and append. |

**Example**

```nim
import std/formatfloat

var s = "result: "
s.addFloatSprintf(12345.6789)
echo s  # "result: 12345.6789"

var s2 = ""
s2.addFloatSprintf(1.0 / 0.0)
echo s2  # "inf"

var s3 = ""
s3.addFloatSprintf(0.0 / 0.0)
echo s3  # "nan"
```

---

### `addFloat`

```nim
proc addFloat*(result: var string; x: float | float32) {.inline.}
```

**What it does**

The primary, high-level function for appending a float to a string. This is the function you should reach for in everyday code. It dispatches to the correct backend automatically:

- On JS targets: uses a JavaScript-native formatting function (`nimFloatToString`) that ensures integers aren't printed without a decimal point, handles `-0.0`, `NaN`, and `Infinity` properly.
- On native targets with `-d:nimLegacySprintf`: uses `addFloatSprintf`.
- On native targets (default): uses `addFloatRoundtrip`.

The function always guarantees that the output *looks like a float* — it will never produce `"2"` for `2.0`; it will always produce `"2.0"`.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `result` | `var string` | The string to append to. |
| `x` | `float` or `float32` | The number to convert and append. |

**Example**

```nim
import std/formatfloat

# Basic usage:
var s = "foo:"
s.addFloat(45.67)
assert s == "foo:45.67"

# Integer-valued floats always get ".0":
var s2 = ""
s2.addFloat(100.0)
assert s2 == "100.0"

# Special values:
var s3 = ""
s3.addFloat(1.0 / 0.0)
assert s3 == "inf"

var s4 = ""
s4.addFloat(-1.0 / 0.0)
assert s4 == "-inf"

var s5 = ""
s5.addFloat(0.0 / 0.0)
assert s5 == "nan"

# float32:
var s6 = ""
s6.addFloat(3.14'f32)
echo s6  # "3.14"
```

---

### `$` *(for float and float32)*

```nim
func `$`*(x: float | float32): string
```

> **Availability:** Only when compiled with `-d:nimPreviewSlimSystem`.

**What it does**

The `$` operator (Nim's universal "stringify" operator) for `float` and `float32`. Returns a new string containing the float's representation. It is implemented as a thin wrapper around `addFloat` — it creates an empty string and appends the float to it.

When `nimPreviewSlimSystem` is not defined, this operator is provided by the `system` module instead. The version here is the modern, slimmed-down replacement.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `x` | `float` or `float32` | The number to convert. |

**Returns:** A new `string` containing the float's text representation.

**Example**

```nim
# compile with: nim c -d:nimPreviewSlimSystem yourfile.nim
import std/formatfloat

let s = $(3.14159)
echo s          # "3.14159"

let s2 = $(2.0)
echo s2         # "2.0"

# Often used in string concatenation:
let msg = "Calculated value: " & $(1.0 / 3.0)
echo msg        # "Calculated value: 0.3333333333333333"
```

---

## Design Notes

**Why 65 characters?** The IEEE 754 double-precision format can produce at most 17 significant decimal digits. Combined with a sign, a decimal point, the letter `e`, an exponent sign, and up to 3 exponent digits, the maximum representation length is well under 64 characters. The buffer is sized to 65 to always leave room for the null terminator.

**Roundtrip vs. sprintf — which is more "accurate"?** Neither loses information — both represent the exact float. The difference is presentation: `sprintf` with `%.16g` may produce more digits than necessary (e.g. `0.10000000000000001` instead of `0.1`), while the roundtrip algorithm always finds the *shortest* string that maps back to the same bit pattern.

**Special values summary:**

| Value | Output |
|-------|--------|
| `1.0 / 0.0` | `"inf"` |
| `-1.0 / 0.0` | `"-inf"` |
| `0.0 / 0.0` | `"nan"` |
| `-0.0` | `"-0.0"` |
| `2.0` | `"2.0"` (never `"2"`) |
