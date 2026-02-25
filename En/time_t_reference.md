# `time_t` Module Reference

> **Module purpose:** `time_t` provides a single exported type — `Time` — which is a portable, platform-correct wrapper around the C standard library's `time_t`. Its sole job is to let Nim code declare values of type `time_t` safely, without manually importing the C header or worrying about whether the underlying integer is 32-bit or 64-bit on the current platform.

---

## Background: What Is `time_t`?

`time_t` is a C type defined in `<time.h>` that stores a calendar time value — almost universally implemented as the number of seconds elapsed since the Unix epoch (1 January 1970, 00:00:00 UTC). It is used as the currency of time across most C-level system APIs: `time()`, `difftime()`, `mktime()`, `localtime()`, and so on.

The exact underlying integer size of `time_t` is **not specified by the C standard** — it is implementation-defined. In practice:

| Platform / Compiler | Underlying size |
|---|---|
| POSIX 64-bit systems | Typically `int64` (signed 64-bit) |
| Windows (modern, 64-bit) | `int64` (since MSVC 2005) |
| Windows (32-bit, GCC) | `clong` (32-bit) |

The 32-bit `time_t` representation overflows on **19 January 2038** (the Year 2038 problem). Modern 64-bit `time_t` values are safe until approximately the year 292 billion.

---

## Type Reference

### `Time`

```nim
type Time* = ...  # platform-dependent underlying type
```

A Nim `distinct` type wrapping `time_t` from `<time.h>`. Being `distinct` means a `Time` value cannot be accidentally used as a plain integer and vice versa — the compiler enforces the distinction. You must explicitly convert if you need the raw number.

`Time` is the type you use when calling or wrapping C functions that accept or return `time_t`. It is **not** the same as `times.Time` from `std/times`, which is Nim's higher-level time representation with nanosecond precision.

#### Platform-specific definitions

The module selects the correct underlying representation at compile time through a cascade of `when` conditions:

**POSIX (Linux, macOS, BSD, …)**

```nim
import std/posix
export posix.Time
```

On POSIX, `Time` is simply re-exported from `std/posix`, which itself imports the real C `time_t` type. This means `time_t.Time` and `posix.Time` are the **same type** — no conversion is needed when interoperating with POSIX APIs.

**Windows (64-bit, or any compiler other than 32-bit GCC)**

```nim
type Time* {.importc: "time_t", header: "<time.h>".} = distinct int64
```

Modern Windows — where `time_t` is 64 bits — maps `Time` to a `distinct int64`. Using `{.importc.}` with `{.header.}` tells the Nim compiler to use the real C type directly rather than a Nim-level alias, ensuring binary compatibility with Windows system libraries.

**Windows (32-bit, GCC specifically)**

```nim
type Time* {.importc: "time_t", header: "<time.h>".} = distinct clong
```

On 32-bit Windows with GCC, `time_t` is a `clong` (a 32-bit signed integer). This branch exists to match the GCC MinGW toolchain's ABI exactly.

**Documentation builds (`nimdoc`)**

```nim
type Time* = distinct int64
```

When Nim generates API documentation, it uses a synthetic `distinct int64` definition so that the type appears in docs without requiring a POSIX or Windows environment. This branch is never compiled into actual binaries.

---

## Usage

### Importing

```nim
import std/time_t
```

### Interoperating with C functions

The primary use case is wrapping C functions that deal with `time_t`:

```nim
import std/time_t

# Declare the C standard library's time() function
proc c_time(tloc: ptr Time): Time {.importc: "time", header: "<time.h>".}

# Get the current time as a raw time_t value
let now: Time = c_time(nil)
echo now.int64   # prints seconds since Unix epoch
```

Wrapping `difftime`, which computes the difference between two `time_t` values:

```nim
import std/time_t

proc c_difftime(time1, time0: Time): cdouble {.importc: "difftime", header: "<time.h>".}

let t0: Time = Time(1_000_000_000)  # Sep 9, 2001
let t1: Time = Time(1_700_000_000)  # Nov 14, 2023

let diff = c_difftime(t1, t0)
echo diff / 86400.0, " days between the two timestamps"
```

### Relationship with `std/times`

`time_t.Time` is a **low-level C-interop type**. `std/times` provides the higher-level `times.Time` with nanosecond precision, timezone handling, formatting, and arithmetic. To bridge between them you typically convert the `time_t.Time` value to an integer and then use `std/times` to interpret it:

```nim
import std/time_t, std/times

proc c_time(tloc: ptr time_t.Time): time_t.Time {.importc: "time", header: "<time.h>".}

let raw: time_t.Time = c_time(nil)

# Convert raw C time_t → std/times.Time
let nimTime: times.Time = fromUnix(raw.int64)
echo nimTime.format("yyyy-MM-dd HH:mm:ss")
```

### Explicit conversion

Because `Time` is a `distinct` type, arithmetic and integer operations require explicit conversion:

```nim
import std/time_t

let t = Time(1_700_000_000'i64)

# Extract the raw integer value
let seconds: int64 = t.int64

# Create a Time from a known epoch offset
let oneHourLater = Time(seconds + 3600)
```

---

## Summary

| Aspect | Detail |
|---|---|
| What it wraps | C `time_t` from `<time.h>` |
| Nim type kind | `distinct` (type-safe, not interchangeable with plain int) |
| POSIX behaviour | Re-exports `posix.Time` — same type, zero overhead |
| Windows 64-bit | `distinct int64`, imported from C header |
| Windows 32-bit GCC | `distinct clong`, imported from C header |
| Precision | Seconds (no sub-second resolution) |
| Epoch | Unix epoch: 1 Jan 1970, 00:00:00 UTC |
| Related module | `std/times` — higher-level Nim time with nanoseconds |
| Year 2038 risk | Only on 32-bit platforms using 32-bit `clong` |

---

## Notes

- This module contains **no procedures** — only the `Time` type definition. All time operations (arithmetic, formatting, conversion) are provided by `std/times` or by directly importing C functions via FFI.
- Never mix `time_t.Time` with `times.Time` without explicit conversion — they are different types with different precisions and semantics, and the Nim compiler will reject implicit use of one where the other is expected.
- On POSIX, importing `std/time_t` and importing `std/posix` both give you the same `Time` type — no double-definition or conflict occurs.
- The `{.importc.}` and `{.header.}` pragmas on the Windows definitions mean the actual C type is used in the compiled output, ensuring correct ABI alignment with system libraries regardless of Nim's internal integer representation.
