# times — module reference

> **Import:** `import std/times`
> **Scope:** working with dates and time under the proleptic Gregorian calendar — points in time, durations, breaking time down into fields, formatting/parsing strings, time zones.

The module solves four largely independent problems, and its structure
reflects that: storing a "raw" point in time (`Time`), representing time as
a set of human-readable fields (`DateTime`), expressing a span of time
either as a fixed number of seconds (`Duration`) or as a calendar interval
("1 year and 2 days", `TimeInterval`), and finally converting between time
zones through `Timezone` objects. A general convention across the module:
wherever an operation admits both an exact (`Duration`) and a calendar
(`TimeInterval`) interpretation, there is a separate set of overloaded
`+`/`-` operators for each, and it's up to the calling code to decide which
one it needs, depending on whether it cares about second-level precision or
about being robust across day/month/year boundaries.

---

## Table of contents

I. [Core types of the module](#core-types-of-the-module)
   1. [`Time`](#time)
   2. [`DateTime`](#datetime)
   3. [`Duration`](#duration)
   4. [`TimeInterval`](#timeinterval)
   5. [`Timezone` and `ZonedTime`](#timezone-and-zonedtime)

II. [Duration — fixed spans of time](#duration--fixed-spans-of-time)
   1. [`initDuration` and `DurationZero`](#initduration-and-durationzero)
   2. [Extracting a value: `inWeeks`, `inDays`, `inHours`, `inMinutes`, `inSeconds`, `inMilliseconds`, `inMicroseconds`, `inNanoseconds`](#extracting-a-value)
   3. [`toParts` (Duration)](#toparts-duration)
   4. [Arithmetic and comparison for `Duration`](#arithmetic-and-comparison-for-duration)
   5. [`high`, `low`, `abs` (Duration)](#high-low-abs-duration)
   6. [`$` (Duration)](#-duration)

III. [Time — a point in time](#time--a-point-in-time)
   1. [`initTime` and `nanosecond`](#inittime-and-nanosecond)
   2. [`fromUnix` / `toUnix`](#fromunix--tounix)
   3. [`fromWinTime` / `toWinTime`](#fromwintime--towintime)
   4. [`getTime`](#gettime)
   5. [Arithmetic and comparison for `Time`](#arithmetic-and-comparison-for-time)
   6. [`high`, `low` (Time)](#high-low-time)
   7. [`$` (Time)](#-time)

IV. [DateTime — time broken into fields](#datetime--time-broken-into-fields)
   1. [DateTime field accessors](#datetime-field-accessors)
   2. [`dateTime` and the deprecated `initDateTime`](#datetime-and-the-deprecated-initdatetime)
   3. [`toTime`](#totime)
   4. [`isLeapDay`](#isleapday)
   5. [Arithmetic and comparison of `DateTime` via `Duration`](#arithmetic-and-comparison-of-datetime-via-duration)
   6. [`getDateStr` / `getClockStr`](#getdatestr--getclockstr)
   7. [`$` (DateTime)](#-datetime)

V. [Time zones](#time-zones)
   1. [`Timezone`, `newTimezone`](#timezone-newtimezone)
   2. [`utc` / `local` (getting a zone and converting into it)](#utc--local)
   3. [`inZone`](#inzone)
   4. [`name`, `$`, `==` for `Timezone`](#name---for-timezone)
   5. [`now`](#now)

VI. [Formatting and parsing dates](#formatting-and-parsing-dates)
   1. [Format-string syntax](#format-string-syntax)
   2. [`initTimeFormat` and `TimeFormat`](#inittimeformat-and-timeformat)
   3. [`format`](#format)
   4. [`parse` / `parseTime`](#parse--parsetime)
   5. [`DateTimeLocale`](#datetimelocale)

VII. [TimeInterval — calendar intervals](#timeinterval--calendar-intervals)
   1. [`initTimeInterval`](#inittimeinterval)
   2. [Single-unit constructors: `nanoseconds` … `years`](#single-unit-constructors)
   3. [Arithmetic for `TimeInterval`](#arithmetic-for-timeinterval)
   4. [`toParts` and `$` (TimeInterval)](#toparts-and--timeinterval)
   5. [Arithmetic for `DateTime`/`Time` with `TimeInterval`](#arithmetic-for-datetimetime-with-timeinterval)
   6. [`between`](#between)

VIII. [Calendar calculations and ISO weeks](#calendar-calculations-and-iso-weeks)
   1. [`isLeapYear`, `getDaysInMonth`, `getDaysInYear`](#isleapyear-getdaysinmonth-getdaysinyear)
   2. [`getDayOfYear`, `getDayOfWeek`](#getdayofyear-getdayofweek)
   3. [`IsoYear`, `getWeeksInIsoYear`, `getIsoWeekAndYear`](#isoyear-getweeksinisoyear-getisoweekandyear)
   4. [`initDateTime` by ISO week](#initdatetime-by-iso-week)

IX. [Other procedures](#other-procedures)
   1. [`convert`](#convert)
   2. [`epochTime`](#epochtime)
   3. [`cpuTime`](#cputime)

X. [Deprecated DateTime field setters](#deprecated-datetime-field-setters)

XI. [Practical recipes](#practical-recipes)
   1. [Timing an operation](#1-timing-an-operation)
   2. [Parsing and formatting dates from an external source](#2-parsing-and-formatting-dates-from-an-external-source)
   3. [Human-readable age/tenure](#3-human-readable-agetenure)
   4. [Converting a timestamp between time zones](#4-converting-a-timestamp-between-time-zones)
   5. [A "every N days" scheduler](#5-a-every-n-days-scheduler)

XII. [Quick reference table](#quick-reference-table)

XIII. [Summary: which procedure to pick](#summary-which-procedure-to-pick)

---

## Core types of the module

### `Time`

```nim
Time* = object
  seconds: int64
  nanosecond: NanosecondRange
```

**Function.** `Time` is a "raw" point in time: seconds since the Unix epoch
(1970-01-01T00:00:00 UTC) plus a nanosecond fraction. It has no time zone
and no year/month/day breakdown — it's just a number with nanosecond
precision that is the same for every observer on Earth. All fields are
private: `Time` only exposes itself through `toUnix`, `nanosecond`,
arithmetic, and comparison.

- **Implementation notes.** Storing it as a (seconds, nanoseconds) pair
  instead of a single nanosecond count is a range choice: a nanosecond
  count in `int64` would overflow around the year 2262, whereas a
  (seconds: int64, nanoseconds: 0..999999999) pair gives the same range as
  `time_t` while retaining full nanosecond precision inside each second.

---

### `DateTime`

```nim
DateTime* = object of RootObj
  nanosecond: NanosecondRange
  second: SecondRange
  minute: MinuteRange
  hour: HourRange
  monthdayZero: int
  monthZero: int
  year: int
  weekday: WeekDay
  yearday: YeardayRange
  isDst: bool
  timezone: Timezone
  utcOffset: int
```

**Function.** `DateTime` represents the same moment as `Time`, but already
broken down into calendar fields (year, month, day, hour...) in a specific
time zone. Unlike `Time`, a `DateTime` value without an explicit time zone
doesn't make sense — hence the `timezone` field, and there's no way to get
a "bare date" without a zone. The default value (`default(DateTime)`) is
considered uninitialized — most procedures in the module check this via an
internal `assertDateTimeInitialized` and assert if you try to read fields
of an empty `DateTime`.

- **Fields/parameters** (read-only, exposed through same-named accessor
  procedures):
  - `nanosecond`, `second`, `minute`, `hour` — time of day;
  - `monthday`, `month`, `year` — calendar date;
  - `weekday`, `yearday` — derived fields, computed automatically;
  - `isDst` — whether daylight saving is in effect at this moment;
  - `timezone` — the time zone the value is expressed in;
  - `utcOffset` — offset from UTC in seconds (with a sign opposite to the
    one used in string offsets like `+01:00`).

---

### `Duration`

```nim
Duration* = object
  seconds: int64
  nanosecond: NanosecondRange
```

**Function.** `Duration` is a fixed span of time: a number of seconds and
nanoseconds, always fully normalized (`initDuration(hours = 1)` and
`initDuration(minutes = 60)` are equal). A day is always exactly 86400
seconds for `Duration`. This makes arithmetic with `Duration` cheap (plain
integer addition/subtraction) and predictable, which is why the module
recommends `Duration` unless you specifically need support for months and
years.

---

### `TimeInterval`

```nim
TimeInterval* = object
  nanoseconds*: int
  microseconds*: int
  milliseconds*: int
  seconds*: int
  minutes*: int
  hours*: int
  days*: int
  weeks*: int
  months*: int
  years*: int
```

**Function.** `TimeInterval` is a calendar interval such as "1 year and 2
days": every unit is kept in its own field and is **not** normalized (even
seconds and milliseconds are left as-is, with no carrying between them),
because units like a year or a month don't have a fixed length in seconds
in the first place (a year is 365 or 366 days). Because of this, arithmetic
with `TimeInterval` needs time-zone information and can be noticeably
slower than with `Duration`. The difference is especially visible across
daylight-saving transitions: between `2018-03-25T12:00+02:00` and
`2018-03-26T12:00+01:00`, calendar-wise "exactly one day" has passed
(`TimeInterval`), but in reality 25 hours = 90000 seconds have passed
(`Duration`), because the UTC offset changed somewhere in between.

---

### `Timezone` and `ZonedTime`

```nim
Timezone* = ref object
  zonedTimeFromTimeImpl: proc (x: Time): ZonedTime
  zonedTimeFromAdjTimeImpl: proc (x: Time): ZonedTime
  name: string

ZonedTime* = object
  time*: Time
  utcOffset*: int
  isDst*: bool
```

**Function.** `Timezone` is a time-zone interface: a pair of converter
functions (`Time -> ZonedTime` and "local time" -> `ZonedTime`) plus a
name. The `times` module itself only ships implementations for UTC and the
system's local zone — support for arbitrary time zones (IANA tzdata and so
on) is implemented by third-party libraries via `newTimezone`. `ZonedTime`
is a helper type: a point in time plus a UTC offset and a DST flag, used
only when implementing time zones.

---
## Duration — fixed spans of time

### `initDuration` and `DurationZero`

```nim
const DurationZero* = Duration()

proc initDuration*(nanoseconds, microseconds, milliseconds,
                   seconds, minutes, hours, days, weeks: int64 = 0): Duration
```

**Function.** Builds a `Duration` from any combination of units — they are
all added together and normalized into a single (seconds, nanoseconds)
pair. `DurationZero` is a ready-made zero-length duration, handy as a
baseline for comparisons.

- **Parameters:**
  - `nanoseconds`, `microseconds`, `milliseconds`, `seconds`, `minutes`,
    `hours`, `days`, `weeks` — every unit is optional (defaults to 0) and
    may be negative; the result is normalized.

**Examples:**

```nim
let
  dur1 = initDuration(seconds = 1)
  dur2 = initDuration(minutes = 60)
doAssert dur1 > DurationZero
doAssert dur2 == initDuration(hours = 1)
```

---

### Extracting a value

```nim
proc inWeeks*(dur: Duration): int64
proc inDays*(dur: Duration): int64
proc inHours*(dur: Duration): int64
proc inMinutes*(dur: Duration): int64
proc inSeconds*(dur: Duration): int64
proc inMilliseconds*(dur: Duration): int64
proc inMicroseconds*(dur: Duration): int64
proc inNanoseconds*(dur: Duration): int64
```

**Function.** Each procedure expresses the whole `Duration` in the given
unit — truncated (`div`), not rounded, so `inHours` on "1 hour 59 minutes"
returns `1`, not `2`.

- **Parameters:** `dur` — the duration to express in the unit named by the
  procedure.

**Examples:**

```nim
let dur = initDuration(hours = 1, minutes = 30)
doAssert inMinutes(dur) == 90
doAssert inHours(dur) == 1          # truncation, not rounding
doAssert inSeconds(initDuration(seconds = -1)) == -1
```

---

### `toParts` (Duration)

```nim
proc toParts*(dur: Duration): DurationParts
```

**Function.** Breaks a `Duration` down into an array over all eight fixed
units (`Nanoseconds`..`Weeks`), each being the "remainder" after subtracting
the larger units. Useful for human-readable output like "1 week, 2 days, 3
hours" without manual `div`/`mod` arithmetic.

- **Parameters:** `dur` — the source duration.

**Example:**

```nim
let dur = initDuration(weeks = 1, days = 2, hours = 3)
let parts = toParts(dur)
doAssert parts[Weeks] == 1
doAssert parts[Days] == 2
doAssert parts[Hours] == 3
```

---

### Arithmetic and comparison for `Duration`

```nim
proc `+`*(a, b: Duration): Duration
proc `-`*(a, b: Duration): Duration
proc `-`*(a: Duration): Duration
proc `<`*(a, b: Duration): bool
proc `<=`*(a, b: Duration): bool
proc `==`*(a, b: Duration): bool
proc `*`*(a: int64, b: Duration): Duration
proc `*`*(a: Duration, b: int64): Duration
proc `div`*(a: Duration, b: int64): Duration
proc `+=`*(d1: var Duration, d2: Duration)
proc `-=`*(dt: var Duration, ti: Duration)
proc `*=`*(a: var Duration, b: int)
```

**Function.** A full set of arithmetic and comparison operators: adding and
subtracting two durations, unary minus (flipping the sign), multiplying and
integer-dividing by a scalar, magnitude comparisons, and the mutating
variants `+=`/`-=`/`*=`. Every operation goes through normalization of the
(seconds, nanoseconds) pair, so nanosecond overflow in either direction is
always correctly carried into the seconds field.

- **Implementation notes.** Addition and subtraction both go through a
  single internal normalization routine: seconds and nanoseconds are added
  (or subtracted) separately, and the nanosecond part is then brought back
  into the `0..999999999` range, carrying over an extra second if needed —
  much like carrying a digit when adding numbers on paper.

**Examples:**

```nim
let
  half = initDuration(minutes = 30)
  hour = initDuration(hours = 1)
doAssert half + half == hour
doAssert hour - half == half
doAssert -half < DurationZero
doAssert half * 2 == hour
doAssert hour div 2 == half
var acc = DurationZero
acc += hour
doAssert acc == hour
```

---

### `high`, `low`, `abs` (Duration)

```nim
proc high*(typ: typedesc[Duration]): Duration
proc low*(typ: typedesc[Duration]): Duration
proc abs*(a: Duration): Duration
```

**Function.** `high`/`low` return the boundary representable durations,
`abs` returns a duration with the sign stripped (useful when only the
magnitude of a difference matters, not which of two time values came
first).

- **Parameters:** `typ` — a placeholder type `Duration` (not used as a
  value, only to select the overload); `a` — the duration for `abs`.

**Example:**

```nim
doAssert abs(initDuration(hours = -3)) == initDuration(hours = 3)
```

---

### `$` (Duration)

```nim
proc `$`*(dur: Duration): string
```

**Function.** A human-readable representation of a duration, like `"1 hour
and 30 minutes"` (only non-zero units are shown, singular form is used for
a value of 1). A zero-length duration prints as `"0 nanoseconds"`.

**Example:**

```nim
echo $initDuration(hours = 1, minutes = 30)  # prints "1 hour and 30 minutes"
echo $DurationZero                            # prints "0 nanoseconds"
```

---
## Time — a point in time

### `initTime` and `nanosecond`

```nim
proc initTime*(unix: int64, nanosecond: NanosecondRange): Time
proc nanosecond*(time: Time): NanosecondRange
```

**Function.** `initTime` assembles a `Time` directly from a
seconds-since-epoch count and a nanosecond fraction, bypassing
`fromUnix`/`getTime`. `nanosecond` is the only public accessor for the
nanosecond part (there is no accessor for the seconds — use `toUnix`
instead).

- **Parameters:**
  - `unix: int64` — seconds since the Unix epoch (may be negative for dates
    before 1970);
  - `nanosecond: NanosecondRange` — the nanosecond fraction, `0..999999999`.

**Example:**

```nim
let t = initTime(0, 500_000_000)
doAssert nanosecond(t) == 500_000_000
```

---

### `fromUnix` / `toUnix`

```nim
proc fromUnix*(unix: int64): Time
proc toUnix*(t: Time): int64
```

**Function.** A pair of converters between `Time` and a plain Unix
timestamp in seconds (no nanoseconds — `fromUnix` produces a zero
nanosecond part, `toUnix` drops it).

- **Parameters:** `unix`/`t` — a timestamp in seconds, or a `Time` value.

**Example:**

```nim
let t = fromUnix(0)
doAssert toUnix(t) == 0
doAssert $t.utc == "1970-01-01T00:00:00Z"
```

---

### `fromWinTime` / `toWinTime`

```nim
proc fromWinTime*(win: int64): Time
proc toWinTime*(t: Time): int64
```

**Function.** Converts to/from "Windows time" — 100-nanosecond intervals
since 1601-01-01 (the `FILETIME` epoch). Mostly used when interacting with
the Windows API or with files that store timestamps this way.

- **Parameters:** `win` — the number of 100-nanosecond intervals since the
  Windows epoch; `t` — a `Time` value.

**Example:**

```nim
let t = fromWinTime(0)
doAssert $t.utc == "1601-01-01T00:00:00Z"
doAssert toWinTime(t) == 0
```

---

### `getTime`

```nim
proc getTime*(): Time
```

**Function.** Returns the current moment as a `Time` — the precision is
platform-dependent (limited to milliseconds on JS). This is the low-level
entry point; `now()` is the same thing, converted right away into a
`DateTime` in the local time zone.

**Example:**

```nim
let t1 = getTime()
let t2 = getTime()
doAssert t2 >= t1
```

---

### Arithmetic and comparison for `Time`

```nim
proc `-`*(a, b: Time): Duration
proc `+`*(a: Time, b: Duration): Time
proc `-`*(a: Time, b: Duration): Time
proc `<`*(a, b: Time): bool
proc `<=`*(a, b: Time): bool
proc `==`*(a, b: Time): bool
proc `+=`*(t: var Time, b: Duration)
proc `-=`*(t: var Time, b: Duration)
```

**Function.** Subtracting two `Time` values gives a fixed `Duration` (not a
`TimeInterval` — that requires `DateTime` and `between`); you can add and
subtract a `Duration` to/from a `Time`, and comparisons work directly on
the internal seconds/nanoseconds.

- **Parameters:** `a`, `b` — `Time` or `Duration` values, depending on the
  overload.

**Example:**

```nim
let
  t1 = fromUnix(0)
  t2 = fromUnix(3600)
doAssert t2 - t1 == initDuration(hours = 1)
doAssert t1 + initDuration(hours = 1) == t2
doAssert t1 < t2
```

---

### `high`, `low` (Time)

```nim
proc high*(typ: typedesc[Time]): Time
proc low*(typ: typedesc[Time]): Time
```

**Function.** The boundary representable `Time` values — handy as an
"infinitely far future/past" when initializing a comparison.

---

### `$` (Time)

```nim
proc `$`*(time: Time): string
```

**Function.** A string representation of a `Time` — converts the value
into the local time zone and formats it using the pattern
`yyyy-MM-dd'T'HH:mm:sszzz`.

**Example:**

```nim
let dt = dateTime(1970, mJan, 01, 00, 00, 00, 00, utc())
let tm = toTime(dt)
echo $tm  # prints "1970-01-01T00:00:00" plus the local zone's offset
```

---
## DateTime — time broken into fields

### DateTime field accessors

```nim
proc nanosecond*(dt: DateTime): NanosecondRange
proc second*(dt: DateTime): SecondRange
proc minute*(dt: DateTime): MinuteRange
proc hour*(dt: DateTime): HourRange
proc monthday*(dt: DateTime): MonthdayRange
proc month*(dt: DateTime): Month
proc year*(dt: DateTime): int
proc weekday*(dt: DateTime): WeekDay
proc yearday*(dt: DateTime): YeardayRange
proc isDst*(dt: DateTime): bool
proc timezone*(dt: DateTime): Timezone
proc utcOffset*(dt: DateTime): int
```

**Function.** A set of simple read-only accessors — each returns the
corresponding field of `DateTime`. `weekday` and `yearday` are not supplied
by the caller but computed by the module at the moment `DateTime` is built
(via `getDayOfWeek`/`getDayOfYear`), so they are always consistent with the
date. Every accessor checks that `dt` is initialized and asserts on an
"empty" `DateTime`.

| Accessor | Type | Value |
|---|---|---|
| `nanosecond` | `NanosecondRange` | nanoseconds within the current second |
| `second` | `SecondRange` | the second (0..60, to allow for a leap second) |
| `minute` | `MinuteRange` | the minute |
| `hour` | `HourRange` | the hour (0..23) |
| `monthday` | `MonthdayRange` | day of the month (1..31) |
| `month` | `Month` | the month |
| `year` | `int` | the year (may be negative, i.e. BC) |
| `weekday` | `WeekDay` | day of the week — computed automatically |
| `yearday` | `YeardayRange` | day of the year (0..365) — computed automatically |
| `isDst` | `bool` | whether daylight saving is in effect |
| `timezone` | `Timezone` | the time zone the value is expressed in |
| `utcOffset` | `int` | offset from UTC in seconds |

**Example:**

```nim
let dt = dateTime(2020, mFeb, 29, 12, 30, 00, 00, utc())
doAssert year(dt) == 2020
doAssert month(dt) == mFeb
doAssert monthday(dt) == 29
doAssert weekday(dt) == dSat  # computed automatically, was never given explicitly
```

---

### `dateTime` and the deprecated `initDateTime`

```nim
proc dateTime*(year: int, month: Month, monthday: MonthdayRange,
               hour: HourRange = 0, minute: MinuteRange = 0, second: SecondRange = 0,
               nanosecond: NanosecondRange = 0,
               zone: Timezone = local()): DateTime
```

**Function.** The main `DateTime` constructor — built from calendar fields
and a time zone. The argument order `year, month, monthday` (unlike the two
deprecated `initDateTime` overloads, where the day came first) reads more
naturally as "year-month-day". Internally the date is first assembled as
"local" time (`toAdjTime`) and then converted into a real `Time` via the
specific zone's `zonedTimeFromAdjTime` — this correctly handles daylight
saving transitions that fall right around midnight.

- **Parameters:**
  - `year: int` — the year (negative values mean BC);
  - `month: Month` — the month;
  - `monthday: MonthdayRange` — day of the month, required;
  - `hour`, `minute`, `second`, `nanosecond` — time of day, defaulting to 0;
  - `zone: Timezone` — the time zone, defaulting to `local()`.

The two deprecated `initDateTime*(monthday, month, year, ...)` overloads do
the same thing, only with the first three parameters in a different order,
and are marked `{.deprecated: "use dateTime".}` — avoid them in new code.

**Examples:**

```nim
let dt = dateTime(2017, mMar, 30, zone = utc())
doAssert $dt == "2017-03-30T00:00:00Z"

doAssertRaises(AssertionDefect):
  discard dateTime(2021, mFeb, 29, zone = utc())  # 2021 is not a leap year
```

---

### `toTime`

```nim
proc toTime*(dt: DateTime): Time
```

**Function.** Converts a `DateTime` back into a "raw" `Time` — a moment in
time without any attached time zone. This is the reverse of `inZone`.

- **Implementation notes.** The date is first converted into an "epoch
  day" (the number of days since 1970-01-01) using Howard Hinnant's
  algorithm (`toEpochDay`), based on division with a 400-year Gregorian
  era — this trick lets the calculation work correctly even for negative
  years, without special-casing individual centuries. The epoch day is
  multiplied by the number of seconds in a day, hours/minutes/seconds and
  `utcOffset` are added — yielding seconds since the Unix epoch.

**Example:**

```nim
let dt = dateTime(1970, mJan, 01, 00, 00, 00, 00, utc())
doAssert toUnix(toTime(dt)) == 0
```

---

### `isLeapDay`

```nim
proc isLeapDay*(dt: DateTime): bool
```

**Function.** Checks whether a date is specifically February 29 of a leap
year. This matters for arithmetic: adding and then subtracting one year to
February 29 is not guaranteed to return the original date (the following
year might not be a leap year).

**Example:**

```nim
let dt = dateTime(2020, mFeb, 29, 00, 00, 00, 00, utc())
doAssert isLeapDay(dt)
doAssert dt + 1.years - 1.years != dt  # Feb 29 doesn't "survive" a non-leap year
```

---

### Arithmetic and comparison of `DateTime` via `Duration`

```nim
proc `+`*(dt: DateTime, dur: Duration): DateTime
proc `-`*(dt: DateTime, dur: Duration): DateTime
proc `-`*(dt1, dt2: DateTime): Duration
proc `<`*(a, b: DateTime): bool
proc `<=`*(a, b: DateTime): bool
proc `==`*(a, b: DateTime): bool
proc `+=`*(a: var DateTime, b: Duration)
proc `-=`*(a: var DateTime, b: Duration)
```

**Function.** Let you add/subtract a fixed `Duration` directly to/from a
`DateTime` (the result stays in the same time zone), and subtracting two
`DateTime` values yields an exact `Duration` — in seconds, with no calendar
interpretation. Comparisons operate on the actual moment in time (via
conversion to `Time`), not on the raw field values.

- **Implementation notes.** All of these operations are implemented via an
  intermediate conversion to `Time` (`toTime`) and back (`inZone`) — in
  other words, `Duration` arithmetic on `DateTime` is really `Duration`
  arithmetic on `Time`, wrapped in a conversion into and out of the time
  zone.

**Example:**

```nim
let dt = dateTime(2020, mJan, 01, 00, 00, 00, 00, utc())
let later = dt + initDuration(hours = 25)  # rolls over into the next day
doAssert monthday(later) == 2
doAssert later - dt == initDuration(hours = 25)
```

---

### `getDateStr` / `getClockStr`

```nim
proc getDateStr*(dt = now()): string
proc getClockStr*(dt = now()): string
```

**Function.** Ready-made short formatters: `getDateStr` gives a date like
`"2020-01-01"`, `getClockStr` gives a time of day like `"12:30:00"`. Handy
when you don't need full control over the format via `format`.

- **Parameters:** `dt` — the `DateTime` value, defaulting to the current
  moment (`now()`).

**Example:**

```nim
let dt = dateTime(2020, mJan, 01, 12, 30, 00, 00, utc())
doAssert getDateStr(dt) == "2020-01-01"
doAssert getClockStr(dt) == "12:30:00"
```

---

### `$` (DateTime)

```nim
proc `$`*(dt: DateTime): string
```

**Function.** A string representation using the format
`yyyy-MM-dd'T'HH:mm:sszzz` (an ISO-8601-like format). For an
uninitialized `DateTime` (the default value) it returns the string
`"Uninitialized DateTime"` instead of failing.

**Example:**

```nim
let dt = dateTime(2000, mJan, 01, 12, 00, 00, 00, utc())
doAssert $dt == "2000-01-01T12:00:00Z"
doAssert $default(DateTime) == "Uninitialized DateTime"
```

---
## Time zones

### `Timezone`, `newTimezone`

```nim
proc newTimezone*(
      name: string,
      zonedTimeFromTimeImpl: proc (time: Time): ZonedTime {.tags: [], raises: [], gcsafe.},
      zonedTimeFromAdjTimeImpl: proc (adjTime: Time): ZonedTime {.tags: [], raises: [], gcsafe.}
    ): owned Timezone
```

**Function.** Constructs an arbitrary time zone out of two conversion
functions (from a "raw" `Time` into a zoned representation, and from
"local time" — also represented as a `Time`, but interpreted not as a
moment but as a clock reading in this zone). Used when you need a zone
that isn't among the built-in `utc`/`local` — for example, when plugging
in a third-party IANA tzdata source.

- **Parameters:**
  - `name: string` — ideally a name from the tz database (e.g.
    `"Europe/Amsterdam"`), otherwise any unique string (affects `==`
    comparisons between zones);
  - `zonedTimeFromTimeImpl` — a "moment in time -> zoned time" function;
  - `zonedTimeFromAdjTimeImpl` — a "clock reading -> zoned time" function.

**Example:**

```nim
proc utcTzInfo(time: Time): ZonedTime =
  ZonedTime(utcOffset: 0, isDst: false, time: time)

let myUtc = newTimezone("Etc/UTC", utcTzInfo, utcTzInfo)
```

---

### `utc` / `local`

```nim
proc utc*(): Timezone
proc local*(): Timezone
proc utc*(dt: DateTime): DateTime
proc local*(dt: DateTime): DateTime
proc utc*(t: Time): DateTime
proc local*(t: Time): DateTime
```

**Function.** The first pair returns the `Timezone` objects themselves —
UTC and the system's local zone (cached in thread-safe `threadvar`s so they
aren't recreated on every call). The remaining four overloads are
convenient shorthands for `inZone(utc())`/`inZone(local())`: they convert
an existing `DateTime` or `Time` into the given zone.

**Example:**

```nim
doAssert name(utc()) == "Etc/UTC"
let t = fromUnix(0)
let dtUtc = utc(t)      # DateTime in UTC
doAssert timezone(dtUtc) == utc()
```

---

### `inZone`

```nim
proc inZone*(time: Time, zone: Timezone): DateTime
proc inZone*(dt: DateTime, zone: Timezone): DateTime
```

**Function.** A universal conversion into an arbitrary time zone: both
`Time` into `DateTime`, and an existing `DateTime` into the same moment,
represented in a different zone.

- **Parameters:** `time`/`dt` — the source value; `zone` — the target
  zone.

**Example:**

```nim
let t = fromUnix(0)
let dtUtc = inZone(t, utc())
doAssert $dtUtc == "1970-01-01T00:00:00Z"
```

---

### `name`, `$`, `==` for `Timezone`

```nim
proc name*(zone: Timezone): string
proc `$`*(zone: Timezone): string
proc `==`*(zone1, zone2: Timezone): bool
```

**Function.** `name` is the stored zone name, `$` is the same thing as a
string (an empty string for `nil`), and `==` compares zones by name (not
by object identity) — so two distinct `Timezone` objects created with the
same name are considered equal.

**Example:**

```nim
doAssert local() == local()
doAssert local() != utc()
```

---

### `now`

```nim
proc now*(): DateTime
```

**Function.** A shorthand for `local(getTime())` — the current moment as a
`DateTime` in the local time zone. Not suitable for benchmarking — use
`cpuTime` for that.

---
## Formatting and parsing dates

### Format-string syntax

A format string describes the pattern used to build or parse a date. The
main patterns (see the source module's comments for the full list):

| Pattern | Meaning | Example |
|---|---|---|
| `d`, `dd` | day of the month (1-2 digits) | `1`, `01` |
| `ddd`, `dddd` | day of the week, short/full | `Sat`, `Saturday` |
| `M`, `MM`, `MMM`, `MMMM` | month: numeric, zero-padded, short, full | `9`, `09`, `Sep`, `September` |
| `yyyy`, `yy` | year with full/truncated digit count | `2012`, `12` |
| `H`, `HH` | hour 0-23 | `2`, `02` |
| `h`, `hh`, `t`, `tt` | hour 1-12 and AM/PM marker | `5`, `05`, `P`, `PM` |
| `m`, `mm`, `s`, `ss` | minutes, seconds | `30`, `06` |
| `z`, `zz`, `zzz`, `zzzz` | UTC offset in various forms | `+7`, `+07:00` |
| `fff`, `ffffff`, `fffffffff` | sub-second precision: milli-/micro-/nanoseconds | `1`, `1000`, `1000000` |
| `g` | era (`AD`/`BC`) | `AD` |

Arbitrary literal text inside a format is escaped with single quotes:
`hh'->'mm` yields, for example, `01->56`. The characters
`: - , . ( ) / [ ]` and space can be inserted without escaping.

---

### `initTimeFormat` and `TimeFormat`

```nim
proc initTimeFormat*(format: string): TimeFormat
proc `$`*(f: TimeFormat): string
```

**Function.** `initTimeFormat` parses a format string once and turns it
into a `TimeFormat` — a pre-compiled representation (a sequence of patterns
and literal chunks) that is then reused by `format`/`parse` many times
without re-parsing the format string. The `format`/`parse` overloads that
accept a `static[string]` call `initTimeFormat` at compile time — so an
error in the format itself (as opposed to the data) is caught immediately,
not at runtime.

- **Parameters:** `format: string` — the format string (see the pattern
  table above).

**Example:**

```nim
let f = initTimeFormat("yyyy-MM-dd")
echo $f  # prints back "yyyy-MM-dd"
```

---

### `format`

```nim
proc format*(dt: DateTime, f: TimeFormat, loc: DateTimeLocale = DefaultLocale): string
proc format*(dt: DateTime, f: string, loc: DateTimeLocale = DefaultLocale): string
proc format*(dt: DateTime, f: static[string]): string
proc format*(time: Time, f: string, zone: Timezone = local()): string
proc format*(time: Time, f: static[string], zone: Timezone = local()): string
```

**Function.** Formats a `DateTime` or a `Time` according to a given
template. The `Time` overloads additionally take a `zone` — the value is
first converted into the given zone, then formatted. The `static[string]`
overloads compile the format at compile time (see above); the plain string
overload compiles it at runtime and may raise `TimeFormatParseError` on an
invalid format.

- **Parameters:**
  - `dt`/`time` — the value to format;
  - `f` — the format: a ready-made `TimeFormat`, a plain string, or a
    compile-time-known string;
  - `loc: DateTimeLocale` — the locale (month/weekday names), defaulting to
    the English `DefaultLocale`;
  - `zone: Timezone` — the zone for the `Time` overloads.

**Example:**

```nim
let dt = dateTime(2000, mJan, 01, 00, 00, 00, 00, utc())
doAssert format(dt, "yyyy-MM-dd") == "2000-01-01"

var tm = toTime(dt)
doAssert format(tm, "yyyy-MM-dd'T'HH:mm:ss", utc()) == "2000-01-01T00:00:00"
```

---

### `parse` / `parseTime`

```nim
proc parse*(input: string, f: TimeFormat, zone: Timezone = local(), loc: DateTimeLocale = DefaultLocale): DateTime
proc parse*(input, f: string, tz: Timezone = local(), loc: DateTimeLocale = DefaultLocale): DateTime
proc parse*(input: string, f: static[string], zone: Timezone = local(), loc: DateTimeLocale = DefaultLocale): DateTime
proc parseTime*(input, f: string, zone: Timezone): Time
proc parseTime*(input: string, f: static[string], zone: Timezone): Time
```

**Function.** Parses the string `input` according to format `f` and
returns a `DateTime` (or a `Time` right away, via `parseTime`). If the
string contained an explicit UTC offset (patterns `z`/`zz`/`zzz`/`zzzz`),
the result is converted into `zone`; if there was no offset, the string is
interpreted as already being expressed in `zone`. `TimeParseError` is
raised when the string doesn't match the format.

- **Implementation notes.** The format and the input string are walked
  through in lockstep: literal chunks are compared character-by-character
  for an exact match, and patterns are handed off to a per-unit parser
  (`parsePattern`) that accumulates results into an intermediate
  `ParsedTime` structure — only after the whole pass completes is it
  converted into the final `DateTime` (`toDateTime`/`toDateTimeByWeek`,
  depending on whether ordinary year/month/day fields or an ISO
  week/ISO year were parsed).

- **Parameters:**
  - `input: string` — the date string;
  - `f` — the parsing format (a ready-made `TimeFormat`, a string, or a
    compile-time-known string);
  - `zone`/`tz` — the zone the date should be interpreted in when there's
    no explicit offset;
  - `loc: DateTimeLocale` — the locale for month/weekday names.

**Examples:**

```nim
let f = initTimeFormat("yyyy-MM-dd")
let dt = dateTime(2000, mJan, 01, 00, 00, 00, 00, utc())
doAssert dt == parse("2000-01-01", f, utc())

let tStr = "1970-01-01T00:00:00+00:00"
doAssert parseTime(tStr, "yyyy-MM-dd'T'HH:mm:sszzz", utc()) == fromUnix(0)
```

---

### `DateTimeLocale`

```nim
DateTimeLocale* = object
  MMM*, MMMM*: array[Month, string]
  ddd*, dddd*: array[WeekDay, string]
```
*(simplified — the module's actual field names group the strings for
short/long forms of months and weekdays.)*

**Function.** A set of strings used when formatting/parsing "by name"
(`MMM`, `MMMM`, `ddd`, `dddd`) — lets you format and parse dates in a
language other than English. `DefaultLocale` is the built-in English
locale used by default.

---
## TimeInterval — calendar intervals

### `initTimeInterval`

```nim
proc initTimeInterval*(nanoseconds = 0, microseconds = 0, milliseconds = 0,
                       seconds = 0, minutes = 0, hours = 0,
                       days = 0, weeks = 0, months = 0, years = 0): TimeInterval
```

**Function.** Builds a `TimeInterval` from any combination of calendar
units. Unlike `initDuration`, there is **no** normalization here — passing
`seconds = 90` leaves the field as `seconds = 90`, it does not turn into
`minutes = 1, seconds = 30`.

- **Parameters:** `nanoseconds`, `microseconds`, `milliseconds`, `seconds`,
  `minutes`, `hours`, `days`, `weeks`, `months`, `years` — every unit
  defaults to 0 and may be negative.

**Example:**

```nim
let ti = initTimeInterval(years = 2, weeks = 1, hours = 3, seconds = 15)
```

---

### Single-unit constructors

```nim
proc nanoseconds*(nanos: int): TimeInterval
proc microseconds*(micros: int): TimeInterval
proc milliseconds*(ms: int): TimeInterval
proc seconds*(s: int): TimeInterval
proc minutes*(m: int): TimeInterval
proc hours*(h: int): TimeInterval
proc days*(d: int): TimeInterval
proc weeks*(w: int): TimeInterval
proc months*(m: int): TimeInterval
proc years*(y: int): TimeInterval
```

**Function.** Short single-field constructors for `TimeInterval` — each is
equivalent to calling `initTimeInterval` with just one field filled in.
Their main purpose is readable arithmetic like "the current moment plus one
year": thanks to these procedures, the expression `1.years` reads almost
like an ordinary unit of measurement.

- **Parameters:** a single number (`nanos`, `micros`, ..., `y`) — the
  magnitude of the corresponding unit.

**Example:**

```nim
let dt = dateTime(2020, mJan, 01, zone = utc())
doAssert dt + years(1) == dateTime(2021, mJan, 01, zone = utc())
doAssert dt + 1.years == dateTime(2021, mJan, 01, zone = utc())  # same thing
```

---

### Arithmetic for `TimeInterval`

```nim
proc `+`*(ti1, ti2: TimeInterval): TimeInterval
proc `-`*(ti: TimeInterval): TimeInterval
proc `-`*(ti1, ti2: TimeInterval): TimeInterval
proc `+=`*(a: var TimeInterval, b: TimeInterval)
proc `-=`*(a: var TimeInterval, b: TimeInterval)
```

**Function.** Adds/subtracts two intervals **field by field**, with no
carrying between units (adding `initTimeInterval(hours = 20)` and
`initTimeInterval(hours = 10)` gives `hours = 30`, not `days = 1, hours =
6`). Unary minus flips the sign of every field at once.

**Example:**

```nim
let a = initTimeInterval(hours = 20)
let b = initTimeInterval(hours = 10)
doAssert (a + b).hours == 30  # fields are not normalized
doAssert (-a).hours == -20
```

---

### `toParts` and `$` (TimeInterval)

```nim
proc toParts*(ti: TimeInterval): TimeIntervalParts
proc `$`*(ti: TimeInterval): string
```

**Function.** `toParts` gives an array of all ten interval fields (for easy
iteration), `$` gives a human-readable string like `"2 years, 1 week, 3
hours, and 15 seconds"`.

**Example:**

```nim
let ti = initTimeInterval(years = 2, weeks = 1, hours = 3, seconds = 15)
echo $ti  # prints "2 years, 1 week, 3 hours, and 15 seconds"
```

---

### Arithmetic for `DateTime`/`Time` with `TimeInterval`

```nim
proc `+`*(dt: DateTime, interval: TimeInterval): DateTime
proc `-`*(dt: DateTime, interval: TimeInterval): DateTime
proc `+`*(time: Time, interval: TimeInterval): Time
proc `-`*(time: Time, interval: TimeInterval): Time
proc `+=`*(a: var DateTime, b: TimeInterval)
proc `-=`*(a: var DateTime, b: TimeInterval)
proc `+=`*(t: var Time, b: TimeInterval)
proc `-=`*(t: var Time, b: TimeInterval)
```

**Function.** Adds/subtracts a calendar interval to/from a point in time —
units are applied from largest to smallest (years and months first, then
weeks/days, then time of day), taking into account month/year boundaries
and (for `DateTime`) the time zone.

- **Parameters:** `dt`/`time` — the source moment; `interval: TimeInterval`
  — how far to shift it.

**Example:**

```nim
doAssert dateTime(2017, mFeb, 01, zone = utc()) + 1.months ==
  dateTime(2017, mMar, 01, zone = utc())
```

---

### `between`

```nim
proc between*(startDt, endDt: DateTime): TimeInterval
```

**Function.** Computes the difference between two `DateTime` values as a
`TimeInterval` — unlike a plain subtraction (which gives a `Duration`), the
result is expressed in calendar units ("2 years, 1 week, 3 hours, 15
seconds") rather than a total number of seconds. It's guaranteed that all
fields of the result share the same sign, and that if both `DateTime`
values are in the same time zone, `startDt + between(startDt, endDt) ==
endDt`.

- **Implementation notes.** Years, months, and days/weeks are computed one
  after another, "greedily" — first the largest whole number of years that
  fits between the dates, then, from the remainder, the largest number of
  months, and so on down to days; the remaining time-of-day part (hours and
  smaller) is computed directly by subtracting `Time` values. If the end
  date's time of day is earlier than the start date's (say, start at 15:00,
  end at 12:00 the next day), one day is first "borrowed" from the calendar
  part — similar to borrowing a digit when subtracting numbers on paper.

- **Parameters:** `startDt`, `endDt` — the start and end of the interval
  (order doesn't matter: `between(a, b) == -between(b, a)`).

**Example:**

```nim
let a = dateTime(2015, mMar, 25, 12, 0, 0, 00, utc())
let b = dateTime(2017, mApr, 01, 15, 0, 15, 00, utc())
let ti = initTimeInterval(years = 2, weeks = 1, hours = 3, seconds = 15)
doAssert between(a, b) == ti
doAssert between(a, b) == -between(b, a)
```

---
## Calendar calculations and ISO weeks

### `isLeapYear`, `getDaysInMonth`, `getDaysInYear`

```nim
proc isLeapYear*(year: int): bool
proc getDaysInMonth*(month: Month, year: int): int
proc getDaysInYear*(year: int): int
```

**Function.** Basic calendar predicates: `isLeapYear` follows the standard
Gregorian rule (divisible by 4, but not by 100 unless also by 400);
`getDaysInMonth` gives the number of days in a month, accounting for the
year (for February); `getDaysInYear` gives 365 or 366.

**Example:**

```nim
doAssert isLeapYear(2000)
doAssert not isLeapYear(1900)  # divisible by 100, but not by 400
doAssert getDaysInMonth(mFeb, 2000) == 29
doAssert getDaysInYear(2000) == 366
```

---

### `getDayOfYear`, `getDayOfWeek`

```nim
proc getDayOfYear*(monthday: MonthdayRange, month: Month, year: int): YeardayRange
proc getDayOfWeek*(monthday: MonthdayRange, month: Month, year: int): WeekDay
```

**Function.** Compute, respectively, the day-of-year number (0-based) and
the weekday for an arbitrary date — without building a full `DateTime`.
These are the very procedures used internally by the module to auto-fill
the `yearday`/`weekday` fields when constructing a `DateTime`.

- **Implementation notes.** `getDayOfWeek` doesn't iterate over days; it
  converts the date to an epoch day (`toEpochDay`) and takes the remainder
  after dividing by 7, knowing that 1970-01-01 was a Thursday: `days =
  epochDay - 3` shifts the reference point to the nearest preceding Monday,
  after which `floorDiv`/the remainder give the weekday directly, with no
  loop involved.

**Example:**

```nim
doAssert getDayOfYear(10, mFeb, 2000) == 40
doAssert getDayOfWeek(13, mJun, 1990) == dWed
```

---

### `IsoYear`, `getWeeksInIsoYear`, `getIsoWeekAndYear`

```nim
proc getWeeksInIsoYear*(y: IsoYear): IsoWeekRange
proc getIsoWeekAndYear*(dt: DateTime): tuple[isoweek: IsoWeekRange, isoyear: IsoYear]
```

**Function.** ISO 8601 defines its own "week-based" calendar, in which a
year can consist of 52 or 53 full weeks, and the first/last days of
January/December sometimes belong to the ISO week of a neighboring
calendar year. `getWeeksInIsoYear` returns 52 or 53 for a given ISO year;
`getIsoWeekAndYear` converts an ordinary date into a pair (ISO week number,
ISO year), which can differ from the date's own calendar year (December 30,
2019 already falls into week 1 of ISO year 2020).

- **Parameters:** `y: IsoYear` — the ISO year (`distinct int`); `dt:
  DateTime` — the date whose ISO week and ISO year need to be determined.

**Example:**

```nim
doAssert getWeeksInIsoYear(IsoYear(2019)) == 52
doAssert getWeeksInIsoYear(IsoYear(2020)) == 53

let dt = dateTime(2019, mDec, 30, zone = utc())
let (w, y) = getIsoWeekAndYear(dt)
doAssert w == 1.IsoWeekRange
doAssert y == 2020.IsoYear   # the ISO year "jumped" to the next one
```

---

### `initDateTime` by ISO week

```nim
proc initDateTime*(weekday: WeekDay, isoweek: IsoWeekRange, isoyear: IsoYear,
                   hour: HourRange, minute: MinuteRange, second: SecondRange,
                   nanosecond: NanosecondRange,
                   zone: Timezone = local()): DateTime
proc initDateTime*(weekday: WeekDay, isoweek: IsoWeekRange, isoyear: IsoYear,
                   hour: HourRange, minute: MinuteRange, second: SecondRange,
                   zone: Timezone = local()): DateTime
```

**Function.** The reverse of `getIsoWeekAndYear` — builds a `DateTime` from
a weekday plus an ISO week number/ISO year, rather than from an ordinary
month/day. Useful when the source data already comes expressed in
ISO-calendar terms (for example, from time-tracking systems that operate on
week numbers).

- **Parameters:** `weekday: WeekDay` — the day of the week; `isoweek:
  IsoWeekRange` — the ISO week number (1..53); `isoyear: IsoYear` — the ISO
  year; `hour`/`minute`/`second`/`nanosecond` — time of day; `zone` — the
  time zone.

**Example:**

```nim
doAssert initDateTime(dSat, 16, 2018.IsoYear, 00, 00, 00) ==
  dateTime(2018, mApr, 21, 00, 00, 00, zone = local())
```

---
## Other procedures

### `convert`

```nim
proc convert*[T: SomeInteger](unitFrom, unitTo: FixedTimeUnit, quantity: T): T
```

**Function.** Converts a whole quantity of one fixed time unit into another
(`Days -> Hours`, `Seconds -> Milliseconds`, and so on). Works only with
integers, so converting from a smaller unit to a larger one truncates the
result rather than rounding it.

- **Parameters:** `unitFrom`, `unitTo: FixedTimeUnit` — the source and
  target units (range `Nanoseconds..Weeks`); `quantity: T` — the amount in
  the `unitFrom` unit.

**Example:**

```nim
doAssert convert(Days, Hours, 2) == 48
doAssert convert(Days, Weeks, 13) == 1       # truncated
doAssert convert(Seconds, Milliseconds, -1) == -1000
```

---

### `epochTime`

```nim
proc epochTime*(): float
```

**Function.** The current time in seconds since the Unix epoch, as a
`float` — a low-level wrapper around a system call. The module explicitly
recommends preferring `getTime()` over it, and using `cpuTime` or
`monotimes.getMonoTime` for timing purposes, since `epochTime`/`getTime`
are affected by system clock adjustments (NTP corrections and the like) and
are not suitable for benchmarking.

---

### `cpuTime`

```nim
proc cpuTime*(): float
```

**Function.** The amount of time the CPU has spent running the current
process, in seconds (on some OSes it actually measures wall-clock time
instead). The value on its own carries no meaning — only the difference
between two calls is useful.

- **Parameters:** none.

**Example:**

```nim
let t0 = cpuTime()
var fib = @[0, 1, 1]
for i in 1..10:
  add(fib, fib[^1] + fib[^2])
echo "CPU time [s] ", cpuTime() - t0
```

---
## Deprecated DateTime field setters

The module keeps a set of backward-compatibility setter procedures marked
`{.deprecated: "Deprecated since v1.3.1".}` —
`` `nanosecond=` ``, `` `second=` ``, `` `minute=` ``, `` `hour=` ``,
`` `monthdayZero=` ``, `` `monthZero=` ``, `` `year=` ``, `` `weekday=` ``,
`` `yearday=` ``, `` `isDst=` ``, `` `timezone=` ``, `` `utcOffset=` ``.
Each one assigns directly to the corresponding `DateTime` field, bypassing
any consistency checks (for example, manually setting `year=` doesn't
recompute `weekday`/`yearday`). In new code, use `dateTime`/`initDateTime`
to build a whole new value instead.

---
## Practical recipes

### 1. Timing an operation

```nim
import std/times

let start = getTime()
# ... code whose running time you want to measure ...
let elapsed = getTime() - start
echo "Took: ", elapsed  # uses `$` (Duration) -> "N seconds" and so on
```

---

### 2. Parsing and formatting dates from an external source

```nim
import std/times

let f = initTimeFormat("yyyy-MM-dd'T'HH:mm:sszzz")
let dt = parse("2024-05-10T09:15:00+02:00", f, utc())
echo format(dt, "dddd, d MMMM yyyy")  # a human-readable date, in UTC
```

---

### 3. Human-readable age/tenure

```nim
import std/times

let birthDate = dateTime(1990, mJun, 15, zone = utc())
let interval = between(birthDate, now().utc)
echo years(interval), " years, ", months(interval), " months"
```

*(In this example, `years`/`months` access the same-named fields of
`TimeInterval` through dotted field access — this is the allowed exception
to the prefix-call rule, since it's a field access, not a procedure call.)*

---

### 4. Converting a timestamp between time zones

```nim
import std/times

let meetingUtc = dateTime(2024, mDec, 01, 14, 00, 00, zone = utc())
let meetingLocal = inZone(meetingUtc, local())
echo "Meeting in local time: ", format(meetingLocal, "HH:mm zzz")
```

---

### 5. A "every N days" scheduler

```nim
import std/times

proc nextRun(lastRun: DateTime, everyDays: int): DateTime =
  lastRun + initDuration(days = everyDays)

var lastRun = dateTime(2024, mJan, 01, 03, 00, 00, zone = utc())
let next = nextRun(lastRun, 7)
doAssert next == dateTime(2024, mJan, 08, 03, 00, 00, zone = utc())
```

---
## Quick reference table

| Task | Result type | What to use |
|---|---|---|
| Current moment as a "raw" value | `Time` | `getTime()` |
| Current moment, broken into fields | `DateTime` | `now()` |
| Build a date from year/month/day | `DateTime` | `dateTime(...)` |
| Exact span of time (seconds/hours) | `Duration` | `initDuration(...)` |
| Calendar interval ("1 year 2 days") | `TimeInterval` | `initTimeInterval(...)` or `N.years`/`N.days`/... |
| Difference between two `Time` values | `Duration` | `t2 - t1` |
| Difference between two `DateTime` values, in calendar units | `TimeInterval` | `between(dt1, dt2)` |
| Convert to another time zone | `DateTime` | `inZone(dt, zone)` / `utc(dt)` / `local(dt)` |
| String -> date | `DateTime`/`Time` | `parse(...)` / `parseTime(...)` |
| Date -> string, by a template | `string` | `format(dt, "...")` |
| Date -> short string, no template | `string` | `getDateStr(dt)` / `getClockStr(dt)` |
| Number of days in a month/year, leap-year check | `int`/`bool` | `getDaysInMonth`, `getDaysInYear`, `isLeapYear` |
| ISO week/ISO year number | `tuple` | `getIsoWeekAndYear(dt)` |
| Timing code performance | `float` | `cpuTime()` |

---
## Summary: which procedure to pick

- Need a point in time with no field breakdown → use `Time` and `getTime`/`fromUnix`.
- Need year/month/day/hour and so on individually → use `DateTime` and `dateTime`/`now`.
- Need to add/subtract a fixed span (hours, seconds) → use `Duration` and `initDuration`.
- Need to add/subtract a calendar span (months, years) that's robust to varying month lengths → use `TimeInterval` and `initTimeInterval`/`N.years` and so on.
- Need an exact difference in seconds between two moments → use the `-` operator between `Time` values (or between `DateTime` values, if they share the same zone).
- Need a "human" difference (how many years, months, days) → use `between`.
- Need to convert time into another zone → use `inZone` (or the `utc`/`local` shorthands).
- Need to parse a string containing a date → use `parse`/`parseTime` with the right format.
- Need to print a date in a specific format → use `format`; for the default ISO-like form, `$` is enough.
- Need to know the number of days in a month/year or check for a leap year → use `getDaysInMonth`/`getDaysInYear`/`isLeapYear`.
- Working with ISO weeks (e.g. integrating with time-tracking systems) → use `getIsoWeekAndYear` and the corresponding `initDateTime` overload.
- Need to time how long code takes to run → use `cpuTime` (not `epochTime`/`now`, which aren't guaranteed to be monotonic).
