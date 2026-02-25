# std/monotimes — Module Reference

> **Part of Nim's standard library** (`std/monotimes`).

## Overview

The `monotimes` module provides a single type — `MonoTime` — and a small set of operations that let you **measure elapsed time with high precision and reliability**.

The key guarantee the module makes is *monotonicity*: a timestamp obtained later is always greater than or equal to one obtained earlier. This sounds obvious, but Nim's standard `times.Time` type does **not** make this guarantee — wall-clock time can jump backward due to NTP corrections, daylight saving transitions, or manual clock adjustments. `MonoTime` is immune to all of these because it counts time from an arbitrary system-internal origin rather than from a calendar epoch.

**Trade-off:** Precisely because the origin is unknown, `MonoTime` cannot be converted to a human-readable date or time. It only makes sense when compared against another `MonoTime`. If you need a displayable timestamp, use `times.Time`; if you need a reliable duration measurement, use `MonoTime`.

Internally, ticks are stored as a 64-bit signed integer in **nanosecond resolution**. The actual hardware precision varies by platform (see platform notes in [`getMonoTime`](#getmonotime)).

---

## Type

### `MonoTime`

```nim
type MonoTime* = object
  ticks: int64   # nanoseconds since an arbitrary system origin
```

An opaque value type representing a point in monotonic time. The internal `ticks` field is private; use `ticks(t)` to read it and the arithmetic operators to work with it. Two `MonoTime` values are only meaningful relative to each other — you can subtract them to get a `Duration`, compare them with `<` / `<=` / `==`, or shift one by a `Duration` with `+` / `-`.

---

## Procedures

---

### `getMonoTime`

```nim
proc getMonoTime*(): MonoTime {.tags: [TimeEffect].}
```

**Returns the current monotonic timestamp.**

This is the primary entry point of the module — the procedure you call to sample "now" on the monotonic clock. Every call returns a value that is guaranteed to be greater than or equal to any previous call on the same system.

**Platform details:**

| Platform | Underlying mechanism |
|---|---|
| Linux / BSD (POSIX) | `clock_gettime(CLOCK_MONOTONIC)` |
| macOS | `mach_absolute_time()` + timebase conversion |
| Windows | `QueryPerformanceCounter` / `QueryPerformanceFrequency` |
| Zephyr RTOS | `k_uptime_ticks()` + `k_ticks_to_ns_floor64()` |
| JS (Node.js) | `process.hrtime()` |
| JS (Browser) | `window.performance.now()` |

Regardless of the platform, the returned value always uses nanosecond units internally. The *precision* (smallest measurable increment) is platform-dependent, but the *unit* is always nanoseconds.

**Example — basic elapsed time measurement:**

```nim
import std/monotimes, std/times

let start = getMonoTime()
# ... do some work ...
let finish = getMonoTime()

let elapsed: Duration = finish - start
echo "Elapsed: ", elapsed
```

**Example — the monotonicity guarantee:**

```nim
let a = getMonoTime()
let b = getMonoTime()
assert a <= b  # always true, no exceptions
```

---

### `ticks`

```nim
proc ticks*(t: MonoTime): int64
```

**Returns the raw nanosecond tick count stored inside a `MonoTime` value.**

This gives you a plain `int64` you can store, log, serialize, or do arithmetic on directly. The value has no meaningful absolute interpretation — it only becomes useful when compared with the `ticks` of another `MonoTime` sampled on the same system. The difference between two tick values is a duration in nanoseconds.

**Example:**

```nim
let t = getMonoTime()
let ns: int64 = t.ticks
echo "Ticks (ns): ", ns

# Manual duration in nanoseconds without using Duration:
let before = getMonoTime()
# ... work ...
let after = getMonoTime()
let diffNs = after.ticks - before.ticks
echo "Nanoseconds elapsed: ", diffNs
```

---

### `$`

```nim
proc `$`*(t: MonoTime): string
```

**Converts a `MonoTime` to its string representation.**

The result is simply the decimal representation of the internal tick count (nanoseconds). This is useful for quick logging or debugging. It does not produce a human-readable date/time string — for that, you need `times.Time`.

**Example:**

```nim
let t = getMonoTime()
echo t          # prints something like: 123456789012345
echo $t         # identical, explicit conversion
```

---

### `-` (MonoTime − MonoTime)

```nim
proc `-`*(a, b: MonoTime): Duration
```

**Computes the time elapsed between two `MonoTime` timestamps, returning a `Duration`.**

This is the central operation you'll use after capturing two timestamps. The result is a `times.Duration` value (from `std/times`), which you can then inspect in whatever unit you need — milliseconds, microseconds, nanoseconds, etc.

Note that if `b` is later than `a` the result will be a negative `Duration`. In most measurement scenarios you'll want to subtract the earlier timestamp from the later one.

**Example:**

```nim
import std/monotimes, std/times

let t1 = getMonoTime()
for i in 1..1_000_000:
  discard i * i  # dummy work
let t2 = getMonoTime()

let elapsed: Duration = t2 - t1
echo "Milliseconds: ", elapsed.inMilliseconds
echo "Microseconds: ", elapsed.inMicroseconds
echo "Nanoseconds:  ", elapsed.inNanoseconds
```

---

### `+` (MonoTime + Duration)

```nim
proc `+`*(a: MonoTime, b: Duration): MonoTime
```

**Returns a new `MonoTime` shifted forward by the given `Duration`.**

This lets you compute a future (or, with a negative `Duration`, past) timestamp relative to an existing one. Useful for scheduling, timeouts, and deadline arithmetic — you can determine whether "now" has passed a deadline by comparing `getMonoTime()` against a precomputed `deadline` value.

**Example:**

```nim
import std/monotimes, std/times

let now = getMonoTime()
let deadline = now + initDuration(milliseconds = 500)

# ... do work ...

if getMonoTime() > deadline:
  echo "Deadline exceeded!"
else:
  echo "Still within time."
```

---

### `-` (MonoTime − Duration)

```nim
proc `-`*(a: MonoTime, b: Duration): MonoTime
```

**Returns a new `MonoTime` shifted backward by the given `Duration`.**

This is the inverse of `+`. It produces a timestamp that is `b` earlier than `a`. Useful when you need to express a point in the past relative to a known reference, for example to check whether an event happened within the last N seconds.

**Example:**

```nim
import std/monotimes, std/times

let now = getMonoTime()
let oneSecondAgo = now - initDuration(seconds = 1)

# Check if some recorded timestamp is "recent" (within last second):
let recorded: MonoTime = getSomeTimestamp()
if recorded >= oneSecondAgo:
  echo "Event happened within the last second."
```

---

### `<`

```nim
proc `<`*(a, b: MonoTime): bool
```

**Returns `true` if timestamp `a` occurred strictly before timestamp `b`.**

Compares two `MonoTime` values by their tick counts. Because the clock is monotonic, "less than" directly maps to "happened earlier in time."

**Example:**

```nim
let t1 = getMonoTime()
let t2 = getMonoTime()
assert t1 < t2 or t1 == t2  # t1 cannot be greater than t2
```

---

### `<=`

```nim
proc `<=`*(a, b: MonoTime): bool
```

**Returns `true` if `a` occurred before `b`, or if they represent the same moment.**

This is the non-strict comparison. On some platforms successive calls to `getMonoTime()` can return identical values (when the clock resolution is coarser than the time between calls), so `<=` is more robust than `<` in assertions about ordering.

**Example:**

```nim
let a = getMonoTime()
let b = getMonoTime()
assert a <= b  # the module's own documented guarantee
```

---

### `==`

```nim
proc `==`*(a, b: MonoTime): bool
```

**Returns `true` if two `MonoTime` values represent the exact same tick count.**

In practice, two independently obtained timestamps being `==` is uncommon (it would mean the clock did not advance between calls), but it can happen on systems with low-resolution monotonic clocks. Equality is also meaningful when comparing a stored timestamp against itself, or when building data structures that key on `MonoTime`.

**Example:**

```nim
let t = getMonoTime()
let copy = t
assert t == copy   # same value, obviously equal

let later = getMonoTime()
echo t == later    # likely false, possibly true on low-res clocks
```

---

### `high`

```nim
proc high*(typ: typedesc[MonoTime]): MonoTime
```

**Returns the highest representable `MonoTime` value.**

The result corresponds to `high(int64)` nanoseconds — an astronomically large timestamp that will not be reached in practice. This is useful as a sentinel "infinity" value in algorithms that need an initial maximum, such as finding the earliest event among a set of candidates.

**Example:**

```nim
var earliest = MonoTime.high  # start with "positive infinity"
for ts in collectedTimestamps:
  if ts < earliest:
    earliest = ts
echo "Earliest timestamp ticks: ", earliest.ticks
```

---

### `low`

```nim
proc low*(typ: typedesc[MonoTime]): MonoTime
```

**Returns the lowest representable `MonoTime` value.**

Corresponds to `low(int64)` nanoseconds — a negative value far in the past that no real timestamp will ever reach. Useful as a sentinel "negative infinity" in algorithms that need an initial minimum.

**Example:**

```nim
var latest = MonoTime.low   # start with "negative infinity"
for ts in collectedTimestamps:
  if ts > latest:
    latest = ts
echo "Latest timestamp ticks: ", latest.ticks
```

---

## Summary Table

| Symbol | Signature | Purpose |
|--------|-----------|---------|
| `getMonoTime()` | `() → MonoTime` | Sample the current monotonic clock |
| `ticks(t)` | `MonoTime → int64` | Read the raw nanosecond tick count |
| `$t` | `MonoTime → string` | Stringify the tick count |
| `a - b` | `(MonoTime, MonoTime) → Duration` | Compute elapsed time between two timestamps |
| `a + d` | `(MonoTime, Duration) → MonoTime` | Shift a timestamp forward by a duration |
| `a - d` | `(MonoTime, Duration) → MonoTime` | Shift a timestamp backward by a duration |
| `a < b` | `(MonoTime, MonoTime) → bool` | Strictly earlier? |
| `a <= b` | `(MonoTime, MonoTime) → bool` | Earlier or simultaneous? |
| `a == b` | `(MonoTime, MonoTime) → bool` | Simultaneous? |
| `MonoTime.high` | `typedesc → MonoTime` | Maximum representable value (sentinel) |
| `MonoTime.low` | `typedesc → MonoTime` | Minimum representable value (sentinel) |

---

## When to use `MonoTime` vs `times.Time`

| Need | Use |
|------|-----|
| Measure how long code takes to run | `MonoTime` |
| Implement timeouts and deadlines | `MonoTime` |
| Order events reliably without clock-skew risk | `MonoTime` |
| Display a timestamp to the user | `times.Time` |
| Record when something happened (absolute) | `times.Time` |
| Serialize/deserialize a timestamp portably | `times.Time` |

---

> **See also:** [`std/times`](https://nim-lang.org/docs/times.html) for wall-clock timestamps, calendar arithmetic, and human-readable formatting.
