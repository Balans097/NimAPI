# `std/times` Module Reference (Nim)

> The `times` module provides routines and types for handling time using the
> [proleptic Gregorian calendar](https://en.wikipedia.org/wiki/Proleptic_Gregorian_calendar).
> It supports nanosecond resolution, though the actual precision of `getTime()` depends on
> the platform (JavaScript is limited to millisecond precision).

---

## Table of Contents

1. [Types](#types)
2. [Constants](#constants)
3. [Helper Functions](#helper-functions)
4. [Duration — Fixed-length Durations](#duration--fixed-length-durations)
5. [Time — Point in Time](#time--point-in-time)
6. [DateTime — Date and Time](#datetime--date-and-time)
7. [Timezone — Timezone Handling](#timezone--timezone-handling)
8. [Formatting and Parsing](#formatting-and-parsing)
9. [TimeInterval — Calendar Intervals](#timeinterval--calendar-intervals)
10. [Convenience Interval Constructors](#convenience-interval-constructors)
11. [System Time and Benchmarking](#system-time-and-benchmarking)
12. [Format Patterns](#format-patterns)
13. [Duration vs TimeInterval](#duration-vs-timeinterval)

---

## Types

### `Month`
Enum for months of the year. Ordinal values start at 1 (`ord(mJan) == 1`).

```nim
mJan, mFeb, mMar, mApr, mMay, mJun,
mJul, mAug, mSep, mOct, mNov, mDec
```

### `WeekDay`
Enum for days of the week (Monday through Sunday).

```nim
dMon, dTue, dWed, dThu, dFri, dSat, dSun
```

### `Time`
Represents a specific point in time as Unix seconds and nanoseconds.

```nim
let t = getTime()
```

### `DateTime`
Represents time broken down into components (year, month, day, hour, minute, second, nanosecond, timezone, etc.).

```nim
let dt = now()
echo dt.year, "-", dt.month.ord, "-", dt.monthday
```

### `Duration`
A fixed-length duration stored as seconds and nanoseconds. Always normalized.

```nim
let d = initDuration(hours = 2, minutes = 30)
```

### `TimeInterval`
A non-fixed duration expressed in calendar units (supports years and months).

```nim
let ti = initTimeInterval(years = 1, months = 2)
```

### `Timezone`
A timezone interface for use with `DateTime`.

### `ZonedTime`
A helper type for implementing custom timezones. Contains `time`, `utcOffset`, and `isDst`.

### `TimeFormat`
A compiled format string for parsing and formatting time values.

### `DateTimeLocale`
Localized names of months and weekdays used for formatting.

### Range Types

| Type | Range |
|---|---|
| `MonthdayRange` | 1..31 |
| `HourRange` | 0..23 |
| `MinuteRange` | 0..59 |
| `SecondRange` | 0..60 (60 allows leap seconds) |
| `NanosecondRange` | 0..999_999_999 |
| `YeardayRange` | 0..365 |
| `IsoWeekRange` | 1..53 |
| `IsoYear` | distinct int |

---

## Constants

### `DurationZero`
A zero-valued `Duration`. Useful for comparisons.

```nim
doAssert initDuration(seconds = 1) > DurationZero
doAssert initDuration(seconds = 0) == DurationZero
```

### `DefaultLocale`
The default locale with English month and day names.

---

## Helper Functions

### `convert`
```nim
proc convert[T: SomeInteger](unitFrom, unitTo: FixedTimeUnit, quantity: T): T
```
Converts a quantity from one time unit to another. Results are truncated (integer division).

```nim
doAssert convert(Days, Hours, 2) == 48
doAssert convert(Days, Weeks, 13) == 1     # truncated
doAssert convert(Seconds, Milliseconds, -1) == -1000
```

---

### `isLeapYear`
```nim
proc isLeapYear(year: int): bool
```
Returns `true` if the given year is a leap year.

```nim
doAssert isLeapYear(2000)       # true — divisible by 400
doAssert not isLeapYear(1900)   # false — divisible by 100 but not 400
doAssert isLeapYear(2024)       # true
```

---

### `getDaysInMonth`
```nim
proc getDaysInMonth(month: Month, year: int): int
```
Returns the number of days in the given month, accounting for leap years.

```nim
doAssert getDaysInMonth(mFeb, 2000) == 29   # 2000 is a leap year
doAssert getDaysInMonth(mFeb, 2001) == 28
doAssert getDaysInMonth(mJan, 2024) == 31
```

---

### `getDayOfYear`
```nim
proc getDayOfYear(monthday: MonthdayRange, month: Month, year: int): YeardayRange
```
Returns the zero-based day-of-year index (January 1 = 0).

```nim
doAssert getDayOfYear(1, mJan, 2000) == 0
doAssert getDayOfYear(10, mJan, 2000) == 9
doAssert getDayOfYear(10, mFeb, 2000) == 40
```

---

### `getDayOfWeek`
```nim
proc getDayOfWeek(monthday: MonthdayRange, month: Month, year: int): WeekDay
```
Returns the weekday for a given date.

```nim
doAssert getDayOfWeek(13, mJun, 1990) == dWed
doAssert $getDayOfWeek(13, mJun, 1990) == "Wednesday"
```

---

### `getDaysInYear`
```nim
proc getDaysInYear(year: int): int
```
Returns the number of days in the year (365 or 366).

```nim
doAssert getDaysInYear(2000) == 366
doAssert getDaysInYear(2001) == 365
```

---

### `getWeeksInIsoYear`
```nim
proc getWeeksInIsoYear(y: IsoYear): IsoWeekRange
```
Returns the number of ISO weeks in the given ISO year (52 or 53).

```nim
assert getWeeksInIsoYear(IsoYear(2019)) == 52
assert getWeeksInIsoYear(IsoYear(2020)) == 53
```

---

### `getIsoWeekAndYear`
```nim
proc getIsoWeekAndYear(dt: DateTime): tuple[isoweek: IsoWeekRange, isoyear: IsoYear]
```
Returns the ISO 8601 week number and year for the given `DateTime`.

> **Warning:** The ISO week-based year may differ from the calendar year for dates between December 29 and January 3.

```nim
let (week, year) = getIsoWeekAndYear(initDateTime(21, mApr, 2018, 0, 0, 0))
assert week == 16 and year == IsoYear(2018)

let (w2, y2) = getIsoWeekAndYear(initDateTime(30, mDec, 2019, 0, 0, 0))
assert w2 == 1 and y2 == IsoYear(2020)  # belongs to next ISO year!
```

---

## Duration — Fixed-length Durations

`Duration` stores an exact amount of time as seconds and nanoseconds. It is always normalized.

### `initDuration`
```nim
proc initDuration(nanoseconds, microseconds, milliseconds,
                  seconds, minutes, hours, days, weeks: int64 = 0): Duration
```
Creates a new `Duration`. All parameters are optional and default to 0.

```nim
let dur = initDuration(seconds = 1, milliseconds = 1)
doAssert dur.inMilliseconds == 1001
doAssert dur.inSeconds == 1

let dur2 = initDuration(hours = 1, minutes = 30)
doAssert dur2.inMinutes == 90
```

---

### Duration Conversion Procedures

```nim
proc inWeeks(dur: Duration): int64
proc inDays(dur: Duration): int64
proc inHours(dur: Duration): int64
proc inMinutes(dur: Duration): int64
proc inSeconds(dur: Duration): int64
proc inMilliseconds(dur: Duration): int64
proc inMicroseconds(dur: Duration): int64
proc inNanoseconds(dur: Duration): int64
```
Each procedure returns the duration expressed in the specified unit (integer part only).

```nim
doAssert initDuration(days = 8).inWeeks == 1
doAssert initDuration(hours = -50).inDays == -2
doAssert initDuration(minutes = 60, days = 2).inHours == 49
doAssert initDuration(seconds = -2).inMilliseconds == -2000
doAssert initDuration(seconds = -2).inNanoseconds == -2000000000
```

---

### `toParts` (Duration)
```nim
proc toParts(dur: Duration): DurationParts
```
Splits a `Duration` into its constituent time unit components. The result is an array indexed by `FixedTimeUnit`.

```nim
var dp = toParts(initDuration(weeks = 2, days = 1))
doAssert dp[Days] == 1
doAssert dp[Weeks] == 2
doAssert dp[Minutes] == 0

dp = toParts(initDuration(days = -1))
doAssert dp[Days] == -1
```

---

### `$` (Duration)
```nim
proc `$`(dur: Duration): string
```
Returns a human-readable string representation.

```nim
doAssert $initDuration(seconds = 2) == "2 seconds"
doAssert $initDuration(weeks = 1, days = 2) == "1 week and 2 days"
doAssert $initDuration(hours = 1, minutes = 2, seconds = 3) == "1 hour, 2 minutes, and 3 seconds"
doAssert $initDuration(milliseconds = -1500) == "-1 second and -500 milliseconds"
```

---

### Duration Arithmetic

```nim
proc `+`(a, b: Duration): Duration         # addition
proc `-`(a, b: Duration): Duration         # subtraction
proc `-`(a: Duration): Duration            # negation
proc `*`(a: int64, b: Duration): Duration  # scalar multiplication
proc `*`(a: Duration, b: int64): Duration
proc `div`(a: Duration, b: int64): Duration  # integer division
proc `<`(a, b: Duration): bool
proc `<=`(a, b: Duration): bool
proc `==`(a, b: Duration): bool
proc `+=`(d1: var Duration, d2: Duration)
proc `-=`(dt: var Duration, ti: Duration)
proc `*=`(a: var Duration, b: int)
```

```nim
let a = initDuration(seconds = 5)
let b = initDuration(seconds = 3)

doAssert a + b == initDuration(seconds = 8)
doAssert a - b == initDuration(seconds = 2)
doAssert 2 * a == initDuration(seconds = 10)
doAssert a div 2 == initDuration(milliseconds = 2500)
doAssert a > b
doAssert -a == initDuration(seconds = -5)

# Comparing absolute values:
doAssert initDuration(seconds = -2).abs < initDuration(seconds = 1).abs == false
```

---

### `abs` (Duration)
```nim
proc abs(a: Duration): Duration
```
Returns the absolute value of the duration.

```nim
doAssert initDuration(milliseconds = -1500).abs == initDuration(milliseconds = 1500)
```

---

### `high` / `low` (Duration)
```nim
proc high(typ: typedesc[Duration]): Duration
proc low(typ: typedesc[Duration]): Duration
```
Returns the maximum and minimum representable durations.

---

## Time — Point in Time

### `initTime`
```nim
proc initTime(unix: int64, nanosecond: NanosecondRange): Time
```
Creates a `Time` from a Unix timestamp and a nanosecond part.

```nim
let t = initTime(1_000_000, 500)
```

---

### `getTime`
```nim
proc getTime(): Time
```
Returns the current time with up to nanosecond resolution (platform-dependent).

```nim
let now = getTime()
echo now
```

---

### `nanosecond` (Time)
```nim
proc nanosecond(time: Time): NanosecondRange
```
Returns the fractional second part as nanoseconds.

---

### `fromUnix` / `toUnix`
```nim
proc fromUnix(unix: int64): Time
proc toUnix(t: Time): int64
```
Converts between a Unix timestamp (seconds since `1970-01-01T00:00:00Z`) and `Time`.

```nim
doAssert $fromUnix(0).utc == "1970-01-01T00:00:00Z"
doAssert fromUnix(0).toUnix() == 0
```

---

### `fromUnixFloat` / `toUnixFloat`
```nim
proc fromUnixFloat(seconds: float): Time
proc toUnixFloat(t: Time): float
```
Same as `fromUnix`/`toUnix` but with sub-second resolution via `float`.

```nim
doAssert fromUnixFloat(123456.0) == fromUnixFloat(123456)
let t = getTime()
doAssert abs(t.toUnixFloat().fromUnixFloat - t) < initDuration(nanoseconds = 1000)
```

---

### `fromWinTime` / `toWinTime`
```nim
proc fromWinTime(win: int64): Time
proc toWinTime(t: Time): int64
```
Converts to/from Windows File Time (100-nanosecond intervals since `1601-01-01`).

---

### Time Arithmetic

```nim
proc `-`(a, b: Time): Duration       # difference between two times
proc `+`(a: Time, b: Duration): Time
proc `-`(a: Time, b: Duration): Time
proc `<`(a, b: Time): bool
proc `<=`(a, b: Time): bool
proc `==`(a, b: Time): bool
proc `+=`(t: var Time, b: Duration)
proc `-=`(t: var Time, b: Duration)
```

```nim
doAssert initTime(1000, 100) - initTime(500, 20) ==
  initDuration(minutes = 8, seconds = 20, nanoseconds = 80)
doAssert (fromUnix(0) + initDuration(seconds = 1)) == fromUnix(1)
doAssert (fromUnix(0) - initDuration(seconds = 1)) == fromUnix(-1)
doAssert initTime(50, 0) < initTime(99, 0)
```

---

### `high` / `low` (Time)
```nim
proc high(typ: typedesc[Time]): Time
proc low(typ: typedesc[Time]): Time
```

---

### `$` (Time)
```nim
proc `$`(time: Time): string
```
Converts to a string in the local timezone, format `yyyy-MM-dd'T'HH:mm:sszzz`.

---

## DateTime — Date and Time

### Constructors

#### `dateTime`
```nim
proc dateTime(year: int, month: Month, monthday: MonthdayRange,
              hour: HourRange = 0, minute: MinuteRange = 0, second: SecondRange = 0,
              nanosecond: NanosecondRange = 0,
              zone: Timezone = local()): DateTime
```
The primary way to create a `DateTime`. The `hour`, `minute`, `second`, `nanosecond`, and `zone` parameters are optional.

```nim
assert $dateTime(2017, mMar, 30, zone = utc()) == "2017-03-30T00:00:00Z"
let dt = dateTime(2024, mFeb, 29, 15, 30, 0, 0, utc())
```

---

#### `initDateTime` (ISO week)
```nim
proc initDateTime(weekday: WeekDay, isoweek: IsoWeekRange, isoyear: IsoYear,
                  hour, minute, second, nanosecond: ...,
                  zone: Timezone = local()): DateTime
```
Creates a `DateTime` from an ISO 8601 week date (weekday + ISO week number + ISO year).

```nim
assert initDateTime(21, mApr, 2018, 0, 0, 0) ==
  initDateTime(dSat, 16, 2018.IsoYear, 0, 0, 0)
assert initDateTime(30, mDec, 2019, 0, 0, 0) ==
  initDateTime(dMon, 01, 2020.IsoYear, 0, 0, 0)  # ISO year belongs to next year!
```

---

#### `now`
```nim
proc now(): DateTime
```
Returns the current local time. Shorthand for `getTime().local`.

> **Warning:** Not suitable for benchmarking — use `cpuTime` or `monotimes.getMonoTime` instead.

```nim
let dt = now()
echo dt   # e.g. "2024-06-01T14:30:00+03:00"
```

---

### DateTime Accessors

```nim
proc nanosecond(dt: DateTime): NanosecondRange   # 0..999_999_999
proc second(dt: DateTime): SecondRange            # 0..59
proc minute(dt: DateTime): MinuteRange            # 0..59
proc hour(dt: DateTime): HourRange                # 0..23
proc monthday(dt: DateTime): MonthdayRange        # 1..31
proc month(dt: DateTime): Month
proc year(dt: DateTime): int
proc weekday(dt: DateTime): WeekDay
proc yearday(dt: DateTime): YeardayRange          # 0..365
proc isDst(dt: DateTime): bool
proc timezone(dt: DateTime): Timezone
proc utcOffset(dt: DateTime): int  # seconds west of UTC (opposite sign from "+HH:MM")
```

```nim
let dt = now()
echo dt.year, "/", dt.month.ord, "/", dt.monthday
echo dt.hour, ":", dt.minute, ":", dt.second
echo "Weekday: ", dt.weekday
echo "DST: ", dt.isDst
```

---

### `isLeapDay`
```nim
proc isLeapDay(dt: DateTime): bool
```
Returns `true` if the date is February 29 in a leap year.

```nim
let dt = dateTime(2020, mFeb, 29, 0, 0, 0, 0, utc())
doAssert dt.isLeapDay
doAssert dt + 1.years - 1.years != dt  # Does not round-trip through leap day arithmetic!
```

---

### `isInitialized`
```nim
proc isInitialized(dt: DateTime): bool
```
Returns `true` if the `DateTime` has been initialized (is not the default zero value).

```nim
doAssert now().isInitialized
doAssert not default(DateTime).isInitialized
```

---

### `toTime`
```nim
proc toTime(dt: DateTime): Time
```
Converts a `DateTime` to a `Time` representing the same point in time.

```nim
let t = now().toTime()
```

---

### `$` (DateTime)
```nim
proc `$`(dt: DateTime): string
```
Formats as `yyyy-MM-dd'T'HH:mm:sszzz`.

```nim
let dt = dateTime(2000, mJan, 01, 12, 0, 0, 0, utc())
doAssert $dt == "2000-01-01T12:00:00Z"
doAssert $default(DateTime) == "Uninitialized DateTime"
```

---

### DateTime Arithmetic

```nim
proc `+`(dt: DateTime, dur: Duration): DateTime
proc `-`(dt: DateTime, dur: Duration): DateTime
proc `-`(dt1, dt2: DateTime): Duration
proc `<`(a, b: DateTime): bool
proc `<=`(a, b: DateTime): bool
proc `==`(a, b: DateTime): bool
proc `+=`(a: var DateTime, b: Duration)
proc `-=`(a: var DateTime, b: Duration)
```

```nim
let dt = dateTime(2017, mMar, 30, 0, 0, 0, 0, utc())
let dur = initDuration(hours = 5)
doAssert $(dt + dur) == "2017-03-30T05:00:00Z"

let dt2 = dateTime(2017, mMar, 25, 0, 0, 0, 0, utc())
doAssert dt - dt2 == initDuration(days = 5)
```

---

### `getDateStr` / `getClockStr`
```nim
proc getDateStr(dt = now()): string
proc getClockStr(dt = now()): string
```
Quickly returns the date (`YYYY-MM-dd`) or time (`HH:mm:ss`) as a string.

```nim
echo getDateStr()              # e.g. "2024-06-01"
echo getClockStr()             # e.g. "14:30:00"
echo getDateStr(now() - 1.months)   # same day last month
```

---

## Timezone — Timezone Handling

### `utc`
```nim
proc utc(): Timezone
```
Returns the UTC timezone implementation (`"Etc/UTC"`).

```nim
doAssert now().utc.timezone == utc()
doAssert utc().name == "Etc/UTC"
```

---

### `local`
```nim
proc local(): Timezone
```
Returns the local system timezone implementation (`"LOCAL"`).

```nim
doAssert now().timezone == local()
doAssert local().name == "LOCAL"
```

---

### `utc` / `local` (conversions)
```nim
proc utc(dt: DateTime): DateTime   # shorthand for dt.inZone(utc())
proc local(dt: DateTime): DateTime # shorthand for dt.inZone(local())
proc utc(t: Time): DateTime
proc local(t: Time): DateTime
```
Shorthand for converting to UTC or local time.

```nim
let dtUtc = now().utc
let dtLocal = getTime().local
```

---

### `inZone`
```nim
proc inZone(time: Time, zone: Timezone): DateTime
proc inZone(dt: DateTime, zone: Timezone): DateTime
```
Converts a `Time` or `DateTime` to another timezone.

```nim
let utcDt = getTime().inZone(utc())
let localDt = getTime().inZone(local())
```

---

### `newTimezone`
```nim
proc newTimezone(
    name: string,
    zonedTimeFromTimeImpl: proc(time: Time): ZonedTime,
    zonedTimeFromAdjTimeImpl: proc(adjTime: Time): ZonedTime
  ): owned Timezone
```
Creates a custom timezone. Useful for implementing timezones not in the standard tz database.

```nim
proc utcTzInfo(time: Time): ZonedTime =
  ZonedTime(utcOffset: 0, isDst: false, time: time)

let myUtc = newTimezone("Etc/UTC", utcTzInfo, utcTzInfo)
```

---

### `name` (Timezone)
```nim
proc name(zone: Timezone): string
```
Returns the name of the timezone.

---

### `zonedTimeFromTime` / `zonedTimeFromAdjTime`
```nim
proc zonedTimeFromTime(zone: Timezone, time: Time): ZonedTime
proc zonedTimeFromAdjTime(zone: Timezone, adjTime: Time): ZonedTime
```
Low-level functions used when implementing custom timezones.

---

### `$` / `==` (Timezone)
```nim
proc `$`(zone: Timezone): string
proc `==`(zone1, zone2: Timezone): bool  # compared by name
```

```nim
doAssert local() == local()
doAssert local() != utc()
```

---

## Formatting and Parsing

### `initTimeFormat`
```nim
proc initTimeFormat(f: string): TimeFormat
```
Compiles a format string into a `TimeFormat` for repeated use.

```nim
let f = initTimeFormat("yyyy-MM-dd")
doAssert $f == "yyyy-MM-dd"
```

---

### `format` (DateTime)
```nim
proc format(dt: DateTime, f: TimeFormat, loc: DateTimeLocale = DefaultLocale): string
proc format(dt: DateTime, f: string, loc: DateTimeLocale = DefaultLocale): string
proc format(dt: DateTime, f: static[string]): string  # compile-time validation
```
Formats a `DateTime` into a string using the given pattern.

```nim
let f = initTimeFormat("yyyy-MM-dd")
let dt = dateTime(2000, mJan, 01, 0, 0, 0, 0, utc())
doAssert "2000-01-01" == dt.format(f)

# Inline format string:
doAssert "2000-01-01" == format(dt, "yyyy-MM-dd")

# With time:
echo dt.format("yyyy-MM-dd HH:mm:ss")
```

---

### `format` (Time)
```nim
proc format(time: Time, f: string, zone: Timezone = local()): string
proc format(time: Time, f: static[string], zone: Timezone = local()): string
```
Formats a `Time` into a string using the given timezone and format.

```nim
var dt = dateTime(1970, mJan, 01, 0, 0, 0, 0, utc())
var tm = dt.toTime()
doAssert format(tm, "yyyy-MM-dd'T'HH:mm:ss", utc()) == "1970-01-01T00:00:00"
```

---

### `formatValue`
```nim
proc formatValue(result: var string; value: DateTime | Time, specifier: string)
```
Adapter for `std/strformat`. Enables using `DateTime` and `Time` in format strings.

```nim
import std/strformat
let dt = now()
echo fmt"{dt:yyyy-MM-dd}"
```

---

### `parse` (string → DateTime)
```nim
proc parse(input: string, f: TimeFormat, zone: Timezone = local(),
    loc: DateTimeLocale = DefaultLocale): DateTime
proc parse(input, f: string, tz: Timezone = local(),
    loc: DateTimeLocale = DefaultLocale): DateTime
proc parse(input: string, f: static[string], zone: Timezone = local(),
    loc: DateTimeLocale = DefaultLocale): DateTime
```
Parses a string into a `DateTime`. Raises `TimeParseError` on failure.

```nim
let f = initTimeFormat("yyyy-MM-dd")
let dt = dateTime(2000, mJan, 01, 0, 0, 0, 0, utc())
doAssert dt == "2000-01-01".parse(f, utc())

# Inline format string:
doAssert dt == parse("2000-01-01", "yyyy-MM-dd", utc())
```

---

### `parseTime`
```nim
proc parseTime(input, f: string, zone: Timezone): Time
proc parseTime(input: string, f: static[string], zone: Timezone): Time
```
Like `parse`, but returns a `Time` instead of `DateTime`.

```nim
let tStr = "1970-01-01T00:00:00+00:00"
doAssert parseTime(tStr, "yyyy-MM-dd'T'HH:mm:sszzz", utc()) == fromUnix(0)
```

---

## TimeInterval — Calendar Intervals

### `initTimeInterval`
```nim
proc initTimeInterval(nanoseconds = 0, microseconds = 0, milliseconds = 0,
                      seconds = 0, minutes = 0, hours = 0,
                      days = 0, weeks = 0, months = 0, years = 0): TimeInterval
```
Creates a new `TimeInterval`. **Does not normalize** units (`hours = 24 != days = 1`).

```nim
let day = initTimeInterval(hours = 24)
let dt = dateTime(2000, mJan, 01, 12, 0, 0, 0, utc())
doAssert $(dt + day) == "2000-01-02T12:00:00Z"
doAssert initTimeInterval(hours = 24) != initTimeInterval(days = 1)
```

---

### TimeInterval Arithmetic

```nim
proc `+`(ti1, ti2: TimeInterval): TimeInterval
proc `-`(ti: TimeInterval): TimeInterval          # negation
proc `-`(ti1, ti2: TimeInterval): TimeInterval
proc `+=`(a: var TimeInterval, b: TimeInterval)
proc `-=`(a: var TimeInterval, b: TimeInterval)
```

```nim
let ti1 = initTimeInterval(hours = 24)
let ti2 = initTimeInterval(hours = 4)
doAssert (ti1 - ti2) == initTimeInterval(hours = 20)

let neg = -initTimeInterval(hours = 24)
doAssert neg.hours == -24
```

---

### DateTime + TimeInterval Arithmetic

```nim
proc `+`(dt: DateTime, interval: TimeInterval): DateTime
proc `-`(dt: DateTime, interval: TimeInterval): DateTime
proc `+=`(a: var DateTime, b: TimeInterval)
proc `-=`(a: var DateTime, b: TimeInterval)
```

```nim
let dt = dateTime(2017, mMar, 30, 0, 0, 0, 0, utc())
doAssert $(dt + 1.months) == "2017-04-30T00:00:00Z"
doAssert $(dt - 1.months) == "2017-03-02T00:00:00Z"  # monthday overflow!
```

---

### Time + TimeInterval Arithmetic

```nim
proc `+`(time: Time, interval: TimeInterval): Time
proc `-`(time: Time, interval: TimeInterval): Time
proc `+=`(t: var Time, b: TimeInterval)
proc `-=`(t: var Time, b: TimeInterval)
```

```nim
let tm = fromUnix(0)
doAssert tm + 5.seconds == fromUnix(5)
doAssert tm - 5.seconds == fromUnix(-5)
```

---

### `between`
```nim
proc between(startDt, endDt: DateTime): TimeInterval
```
Computes the difference between two `DateTime` values as a `TimeInterval`. Guarantees `startDt + between(startDt, endDt) == endDt` when both share the same timezone.

```nim
var a = dateTime(2015, mMar, 25, 12, 0, 0, 0, utc())
var b = dateTime(2017, mApr, 1, 15, 0, 15, 0, utc())
var ti = initTimeInterval(years = 2, weeks = 1, hours = 3, seconds = 15)
doAssert between(a, b) == ti
doAssert between(a, b) == -between(b, a)
```

---

### `toParts` (TimeInterval)
```nim
proc toParts(ti: TimeInterval): TimeIntervalParts
```
Splits a `TimeInterval` into an array indexed by `TimeUnit`.

```nim
var tp = toParts(initTimeInterval(years = 1, nanoseconds = 123))
doAssert tp[Years] == 1
doAssert tp[Nanoseconds] == 123
```

---

### `$` (TimeInterval)
```nim
proc `$`(ti: TimeInterval): string
```
Returns a human-readable string representation.

```nim
doAssert $initTimeInterval(years = 1, nanoseconds = 123) == "1 year and 123 nanoseconds"
doAssert $initTimeInterval() == "0 nanoseconds"
```

---

## Convenience Interval Constructors

These procedures enable the idiomatic `5.seconds`, `2.hours` syntax.

```nim
proc nanoseconds(nanos: int): TimeInterval
proc microseconds(micros: int): TimeInterval
proc milliseconds(ms: int): TimeInterval
proc seconds(s: int): TimeInterval
proc minutes(m: int): TimeInterval
proc hours(h: int): TimeInterval
proc days(d: int): TimeInterval
proc weeks(w: int): TimeInterval
proc months(m: int): TimeInterval
proc years(y: int): TimeInterval
```

```nim
echo now() + 1.hours
echo now() + 2.days
echo now() - 1.months
echo now() + 1.years

# Chaining:
let dt = now() + 1.years + 2.months + 3.days
```

---

## System Time and Benchmarking

### `epochTime`
```nim
proc epochTime(): float
```
Returns seconds since the Unix epoch (1970) as a `float`. Offers sub-second resolution, but **not suitable for benchmarking**.

> Prefer `getTime()` for general-purpose time retrieval.

```nim
let t = epochTime()
# ... some work ...
echo "Elapsed seconds: ", epochTime() - t
```

---

### `cpuTime`
```nim
proc cpuTime(): float
```
Returns the CPU time used by the current process in seconds. More useful for benchmarking than `epochTime`. May measure real time depending on the OS.

```nim
var t0 = cpuTime()
var fib = @[0, 1, 1]
for i in 1..10:
  fib.add(fib[^1] + fib[^2])
echo "CPU time [s]: ", cpuTime() - t0
```

---

## Format Patterns

| Pattern | Description | Example |
|---|---|---|
| `d` | Day of month (1–2 digits) | `1` or `21` |
| `dd` | Day of month (always 2 digits) | `01` or `21` |
| `ddd` | Abbreviated weekday name | `Sat`, `Mon` |
| `dddd` | Full weekday name | `Saturday` |
| `GG` | Last 2 digits of ISO year | `13` |
| `GGGG` | ISO year (4 digits) | `2013` |
| `h` | Hour 1–12 | `5` (pm), `2` (am) |
| `hh` | Hour 1–12 (2 digits) | `05`, `11` |
| `H` | Hour 0–23 | `17`, `2` |
| `HH` | Hour 0–23 (2 digits) | `17`, `02` |
| `m` | Minutes (1–2 digits) | `30`, `1` |
| `mm` | Minutes (2 digits) | `30`, `01` |
| `M` | Month number (1–2 digits) | `9`, `12` |
| `MM` | Month number (2 digits) | `09`, `12` |
| `MMM` | Abbreviated month name | `Sep`, `Dec` |
| `MMMM` | Full month name | `September` |
| `s` | Seconds (1–2 digits) | `6` |
| `ss` | Seconds (2 digits) | `06` |
| `t` | AM/PM (1 character) | `A`, `P` |
| `tt` | AM/PM (2 characters) | `AM`, `PM` |
| `yy` | Last 2 digits of year | `12` |
| `yyyy` | Year (min. 4 digits, always positive) | `2012`, `+12345` |
| `YYYY` | Year (no padding, always positive) | `2012`, `24` |
| `uuuu` | Year (min. 4 digits, negative for BC) | `-0023` |
| `UUUU` | Year (no padding, negative for BC) | `-23` |
| `V` | ISO week number (1–2 digits) | `5`, `13` |
| `VV` | ISO week number (2 digits) | `05`, `13` |
| `z` | UTC offset | `+7`, `-5` |
| `zz` | UTC offset (with leading zero) | `+07`, `-05` |
| `zzz` | UTC offset with minutes | `+07:00` |
| `ZZZ` | UTC offset with minutes (no colon) | `+0700` |
| `zzzz` | UTC offset with seconds | `+07:00:00` |
| `ZZZZ` | UTC offset with seconds (no colons) | `+070000` |
| `g` | Era: AD or BC | `AD`, `BC` |
| `fff` | Milliseconds | `1` |
| `ffffff` | Microseconds | `1000` |
| `fffffffff` | Nanoseconds | `1000000` |

**Literals** are inserted using single quotes: `hh'->'mm` → `01->56`.  
Without quotes, these characters are allowed inline: `:`, `-`, `,`, `.`, `(`, `)`, `/`, `[`, `]`, space.

---

## Duration vs TimeInterval

| Characteristic | `Duration` | `TimeInterval` |
|---|---|---|
| Storage | Seconds + nanoseconds | Separate fields per unit |
| Normalization | Always normalized | Not normalized |
| Performance | Fast | Slow (requires timezone info) |
| Supports years/months | ❌ | ✅ |
| When to use | Most cases | Only when years/months needed |

**Important note on days:** `Duration` treats a day as exactly 86400 seconds. `TimeInterval` works with calendar days — for example, due to DST transitions, one calendar day can equal 25 hours.

```nim
# Duration — fixed seconds
let dur = initDuration(hours = 25)  # exactly 90000 seconds

# TimeInterval — calendar day
let ti = initTimeInterval(days = 1)  # exactly 1 calendar day
```

**Example illustrating the difference:**

Consider the timestamps `2018-03-25T12:00+02:00` and `2018-03-26T12:00+01:00` in the same timezone. They look one calendar day apart, but the UTC offset changed, so the actual elapsed time is 25 hours.

- `TimeInterval` says: 1 day has passed
- `Duration` says: 90000 seconds (25 hours) have passed
