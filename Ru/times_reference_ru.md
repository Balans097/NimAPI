# Справочник модуля `std/times` (Nim)

> Модуль `times` содержит процедуры и типы для работы со временем с использованием
> [пролептического григорианского календаря](https://en.wikipedia.org/wiki/Proleptic_Gregorian_calendar).
> Поддерживает разрешение до наносекунд, однако фактическая точность зависит от платформы
> (JavaScript ограничен миллисекундами).

---

## Оглавление

1. [Типы данных](#типы-данных)
2. [Константы](#константы)
3. [Вспомогательные функции](#вспомогательные-функции)
4. [Duration — фиксированная длительность](#duration--фиксированная-длительность)
5. [Time — момент времени](#time--момент-времени)
6. [DateTime — дата и время](#datetime--дата-и-время)
7. [Timezone — часовые пояса](#timezone--часовые-пояса)
8. [Форматирование и парсинг](#форматирование-и-парсинг)
9. [TimeInterval — календарный интервал](#timeinterval--календарный-интервал)
10. [Удобные конструкторы интервалов](#удобные-конструкторы-интервалов)
11. [Системное время и бенчмаркинг](#системное-время-и-бенчмаркинг)
12. [Паттерны форматирования](#паттерны-форматирования)
13. [Duration vs TimeInterval](#duration-vs-timeinterval)

---

## Типы данных

### `Month`
Перечисление месяцев года. Нумерация начинается с 1 (`ord(mJan) == 1`).

```nim
mJan, mFeb, mMar, mApr, mMay, mJun,
mJul, mAug, mSep, mOct, mNov, mDec
```

### `WeekDay`
Перечисление дней недели (от понедельника до воскресенья).

```nim
dMon, dTue, dWed, dThu, dFri, dSat, dSun
```

### `Time`
Представляет точку во времени в виде секунд (Unix-время) и наносекунд.

```nim
let t = getTime()
```

### `DateTime`
Представляет время в разбивке по составляющим (год, месяц, день, час, минута, секунда, наносекунда, часовой пояс и др.).

```nim
let dt = now()
echo dt.year, "-", dt.month.ord, "-", dt.monthday
```

### `Duration`
Фиксированная длительность, хранящаяся как секунды и наносекунды. Всегда нормализована.

```nim
let d = initDuration(hours = 2, minutes = 30)
```

### `TimeInterval`
Нефиксированная длительность в календарных единицах (поддерживает годы, месяцы и т.д.).

```nim
let ti = initTimeInterval(years = 1, months = 2)
```

### `Timezone`
Интерфейс часового пояса для работы с `DateTime`.

### `ZonedTime`
Вспомогательный объект для реализации часового пояса. Содержит `time`, `utcOffset`, `isDst`.

### `TimeFormat`
Скомпилированный формат для парсинга и форматирования времени.

### `DateTimeLocale`
Локализованные названия месяцев и дней недели для форматирования.

### Диапазонные типы

| Тип | Диапазон |
|---|---|
| `MonthdayRange` | 1..31 |
| `HourRange` | 0..23 |
| `MinuteRange` | 0..59 |
| `SecondRange` | 0..60 (60 — секунда координации) |
| `NanosecondRange` | 0..999_999_999 |
| `YeardayRange` | 0..365 |
| `IsoWeekRange` | 1..53 |
| `IsoYear` | distinct int |

---

## Константы

### `DurationZero`
Нулевая длительность — удобна для сравнений.

```nim
doAssert initDuration(seconds = 1) > DurationZero
doAssert initDuration(seconds = 0) == DurationZero
```

### `DefaultLocale`
Локаль по умолчанию (английские названия месяцев и дней).

---

## Вспомогательные функции

### `convert`
```nim
proc convert[T: SomeInteger](unitFrom, unitTo: FixedTimeUnit, quantity: T): T
```
Конвертирует количество единиц времени из одного типа в другой. Результат может быть усечён (целочисленное деление).

```nim
doAssert convert(Days, Hours, 2) == 48
doAssert convert(Days, Weeks, 13) == 1    # усечение
doAssert convert(Seconds, Milliseconds, -1) == -1000
```

---

### `isLeapYear`
```nim
proc isLeapYear(year: int): bool
```
Возвращает `true`, если год является високосным.

```nim
doAssert isLeapYear(2000)       # true — делится на 400
doAssert not isLeapYear(1900)   # false — делится на 100, но не на 400
doAssert isLeapYear(2024)       # true
```

---

### `getDaysInMonth`
```nim
proc getDaysInMonth(month: Month, year: int): int
```
Возвращает количество дней в указанном месяце с учётом високосного года.

```nim
doAssert getDaysInMonth(mFeb, 2000) == 29   # 2000 — високосный
doAssert getDaysInMonth(mFeb, 2001) == 28
doAssert getDaysInMonth(mJan, 2024) == 31
```

---

### `getDayOfYear`
```nim
proc getDayOfYear(monthday: MonthdayRange, month: Month, year: int): YeardayRange
```
Возвращает порядковый номер дня в году (начиная с 0).

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
Возвращает день недели для указанной даты.

```nim
doAssert getDayOfWeek(13, mJun, 1990) == dWed
doAssert $getDayOfWeek(13, mJun, 1990) == "Wednesday"
```

---

### `getDaysInYear`
```nim
proc getDaysInYear(year: int): int
```
Возвращает количество дней в году (365 или 366).

```nim
doAssert getDaysInYear(2000) == 366
doAssert getDaysInYear(2001) == 365
```

---

### `getWeeksInIsoYear`
```nim
proc getWeeksInIsoYear(y: IsoYear): IsoWeekRange
```
Возвращает количество ISO-недель в году (52 или 53).

```nim
assert getWeeksInIsoYear(IsoYear(2019)) == 52
assert getWeeksInIsoYear(IsoYear(2020)) == 53
```

---

### `getIsoWeekAndYear`
```nim
proc getIsoWeekAndYear(dt: DateTime): tuple[isoweek: IsoWeekRange, isoyear: IsoYear]
```
Возвращает ISO 8601 номер недели и год для `DateTime`.

> **Внимание:** ISO-год может отличаться от обычного года для дат с 29 декабря по 3 января.

```nim
let (week, year) = getIsoWeekAndYear(initDateTime(21, mApr, 2018, 0, 0, 0))
assert week == 16 and year == IsoYear(2018)

let (w2, y2) = getIsoWeekAndYear(initDateTime(30, mDec, 2019, 0, 0, 0))
assert w2 == 1 and y2 == IsoYear(2020)  # Относится к следующему году!
```

---

## Duration — фиксированная длительность

`Duration` хранит точное количество времени в виде секунд и наносекунд. Всегда нормализована.

### `initDuration`
```nim
proc initDuration(nanoseconds, microseconds, milliseconds,
                  seconds, minutes, hours, days, weeks: int64 = 0): Duration
```
Создаёт новый `Duration`. Все параметры необязательны и по умолчанию равны 0.

```nim
let dur = initDuration(seconds = 1, milliseconds = 1)
doAssert dur.inMilliseconds == 1001
doAssert dur.inSeconds == 1

let dur2 = initDuration(hours = 1, minutes = 30)
doAssert dur2.inMinutes == 90
```

---

### Конвертация Duration в единицы времени

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
Каждая функция возвращает длительность, преобразованную в указанные единицы (целая часть).

```nim
doAssert initDuration(days = 8).inWeeks == 1
doAssert initDuration(hours = -50).inDays == -2
doAssert initDuration(minutes = 60, days = 2).inHours == 49
doAssert initDuration(seconds = -2).inMilliseconds == -2000
doAssert initDuration(seconds = -2).inNanoseconds == -2000000000
```

---

### `toParts` (для Duration)
```nim
proc toParts(dur: Duration): DurationParts
```
Разбивает `Duration` на составляющие единицы времени. Результат — массив, индексируемый `FixedTimeUnit`.

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
Возвращает человекочитаемое строковое представление.

```nim
doAssert $initDuration(seconds = 2) == "2 seconds"
doAssert $initDuration(weeks = 1, days = 2) == "1 week and 2 days"
doAssert $initDuration(hours = 1, minutes = 2, seconds = 3) == "1 hour, 2 minutes, and 3 seconds"
doAssert $initDuration(milliseconds = -1500) == "-1 second and -500 milliseconds"
```

---

### Арифметика с Duration

```nim
proc `+`(a, b: Duration): Duration       # сложение
proc `-`(a, b: Duration): Duration       # вычитание
proc `-`(a: Duration): Duration          # унарный минус
proc `*`(a: int64, b: Duration): Duration  # умножение на скаляр
proc `*`(a: Duration, b: int64): Duration
proc `div`(a: Duration, b: int64): Duration  # целочисленное деление
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

# Сравнение абсолютных значений:
doAssert initDuration(seconds = -2).abs < initDuration(seconds = 1).abs == false
```

---

### `abs` (Duration)
```nim
proc abs(a: Duration): Duration
```
Возвращает абсолютное значение длительности.

```nim
doAssert initDuration(milliseconds = -1500).abs == initDuration(milliseconds = 1500)
```

---

### `high` / `low` (Duration)
```nim
proc high(typ: typedesc[Duration]): Duration
proc low(typ: typedesc[Duration]): Duration
```
Возвращают максимальную и минимальную представимые длительности.

---

## Time — момент времени

### `initTime`
```nim
proc initTime(unix: int64, nanosecond: NanosecondRange): Time
```
Создаёт `Time` из Unix-метки и наносекундной части.

```nim
let t = initTime(1_000_000, 500)
```

---

### `getTime`
```nim
proc getTime(): Time
```
Возвращает текущее время с точностью до наносекунд (зависит от платформы).

```nim
let now = getTime()
echo now
```

---

### `nanosecond` (Time)
```nim
proc nanosecond(time: Time): NanosecondRange
```
Дробная часть секунды в наносекундах.

---

### `fromUnix` / `toUnix`
```nim
proc fromUnix(unix: int64): Time
proc toUnix(t: Time): int64
```
Конвертация между Unix-меткой (секунды от `1970-01-01T00:00:00Z`) и `Time`.

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
Аналог `fromUnix`/`toUnix`, но с субсекундным разрешением через `float`.

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
Конвертация Windows File Time (100-наносекундные интервалы с `1601-01-01`).

---

### Арифметика с Time

```nim
proc `-`(a, b: Time): Duration       # разница между двумя моментами
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
Преобразует в строку в локальном часовом поясе, формат `yyyy-MM-dd'T'HH:mm:sszzz`.

---

## DateTime — дата и время

### Конструкторы

#### `dateTime`
```nim
proc dateTime(year: int, month: Month, monthday: MonthdayRange,
              hour: HourRange = 0, minute: MinuteRange = 0, second: SecondRange = 0,
              nanosecond: NanosecondRange = 0,
              zone: Timezone = local()): DateTime
```
Основной способ создания `DateTime`. Параметры `hour`, `minute`, `second`, `nanosecond` и `zone` необязательны.

```nim
assert $dateTime(2017, mMar, 30, zone = utc()) == "2017-03-30T00:00:00Z"
let dt = dateTime(2024, mFeb, 29, 15, 30, 0, 0, utc())
```

---

#### `initDateTime` (ISO-неделя)
```nim
proc initDateTime(weekday: WeekDay, isoweek: IsoWeekRange, isoyear: IsoYear,
                  hour, minute, second: ..., nanosecond: ...,
                  zone: Timezone = local()): DateTime
```
Создаёт `DateTime` по ISO 8601 (день недели + номер недели + ISO-год).

```nim
assert initDateTime(21, mApr, 2018, 0, 0, 0) ==
  initDateTime(dSat, 16, 2018.IsoYear, 0, 0, 0)
assert initDateTime(30, mDec, 2019, 0, 0, 0) ==
  initDateTime(dMon, 01, 2020.IsoYear, 0, 0, 0)  # ISO-год следующего года!
```

---

#### `now`
```nim
proc now(): DateTime
```
Текущее время в локальном часовом поясе. Сокращение для `getTime().local`.

> **Внимание:** не подходит для бенчмаркинга — используйте `cpuTime` или `monotimes.getMonoTime`.

```nim
let dt = now()
echo dt   # e.g. "2024-06-01T14:30:00+03:00"
```

---

### Аксессоры DateTime

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
proc utcOffset(dt: DateTime): int  # секунды к западу от UTC (противоположный знак от "+HH:MM")
```

```nim
let dt = now()
echo dt.year, "/", dt.month.ord, "/", dt.monthday
echo dt.hour, ":", dt.minute, ":", dt.second
echo "День недели: ", dt.weekday
echo "DST: ", dt.isDst
```

---

### `isLeapDay`
```nim
proc isLeapDay(dt: DateTime): bool
```
Возвращает `true`, если дата является 29 февраля в високосном году.

```nim
let dt = dateTime(2020, mFeb, 29, 0, 0, 0, 0, utc())
doAssert dt.isLeapDay
doAssert dt + 1.years - 1.years != dt  # Не возвращается в ту же дату!
```

---

### `isInitialized`
```nim
proc isInitialized(dt: DateTime): bool
```
Возвращает `true`, если `DateTime` был инициализирован (не является значением по умолчанию).

```nim
doAssert now().isInitialized
doAssert not default(DateTime).isInitialized
```

---

### `toTime`
```nim
proc toTime(dt: DateTime): Time
```
Конвертирует `DateTime` в `Time`.

```nim
let t = now().toTime()
```

---

### `$` (DateTime)
```nim
proc `$`(dt: DateTime): string
```
Форматирует как `yyyy-MM-dd'T'HH:mm:sszzz`.

```nim
let dt = dateTime(2000, mJan, 01, 12, 0, 0, 0, utc())
doAssert $dt == "2000-01-01T12:00:00Z"
doAssert $default(DateTime) == "Uninitialized DateTime"
```

---

### Арифметика с DateTime

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
Быстрое получение строки даты (`YYYY-MM-dd`) или времени (`HH:mm:ss`).

```nim
echo getDateStr()          # напр. "2024-06-01"
echo getClockStr()         # напр. "14:30:00"
echo getDateStr(now() - 1.months)   # вчерашняя дата минус месяц
```

---

## Timezone — часовые пояса

### `utc`
```nim
proc utc(): Timezone
```
Возвращает реализацию часового пояса UTC (`"Etc/UTC"`).

```nim
doAssert now().utc.timezone == utc()
doAssert utc().name == "Etc/UTC"
```

---

### `local`
```nim
proc local(): Timezone
```
Возвращает реализацию локального системного часового пояса (`"LOCAL"`).

```nim
doAssert now().timezone == local()
doAssert local().name == "LOCAL"
```

---

### `utc` / `local` (конвертация)
```nim
proc utc(dt: DateTime): DateTime   # dt.inZone(utc())
proc local(dt: DateTime): DateTime # dt.inZone(local())
proc utc(t: Time): DateTime
proc local(t: Time): DateTime
```
Краткая запись для конвертации в UTC или локальный часовой пояс.

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
Конвертирует `Time` или `DateTime` в другой часовой пояс.

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
Создаёт пользовательский часовой пояс. Полезно для реализации часовых поясов не из стандартной базы tz.

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
Возвращает имя часового пояса.

---

### `zonedTimeFromTime` / `zonedTimeFromAdjTime`
```nim
proc zonedTimeFromTime(zone: Timezone, time: Time): ZonedTime
proc zonedTimeFromAdjTime(zone: Timezone, adjTime: Time): ZonedTime
```
Низкоуровневые функции для реализации часовых поясов.

---

### `$` / `==` (Timezone)
```nim
proc `$`(zone: Timezone): string
proc `==`(zone1, zone2: Timezone): bool  # сравнение по имени
```

```nim
doAssert local() == local()
doAssert local() != utc()
```

---

## Форматирование и парсинг

### `initTimeFormat`
```nim
proc initTimeFormat(f: string): TimeFormat
```
Компилирует строку формата в `TimeFormat` для последующего использования.

```nim
let f = initTimeFormat("yyyy-MM-dd")
doAssert $f == "yyyy-MM-dd"
```

---

### `format` (DateTime)
```nim
proc format(dt: DateTime, f: TimeFormat, loc: DateTimeLocale = DefaultLocale): string
proc format(dt: DateTime, f: string, loc: DateTimeLocale = DefaultLocale): string
proc format(dt: DateTime, f: static[string]): string  # валидация на этапе компиляции
```
Форматирует `DateTime` в строку по шаблону.

```nim
let f = initTimeFormat("yyyy-MM-dd")
let dt = dateTime(2000, mJan, 01, 0, 0, 0, 0, utc())
doAssert "2000-01-01" == dt.format(f)

# Прямо строкой:
doAssert "2000-01-01" == format(dt, "yyyy-MM-dd")

# Со временем:
echo dt.format("yyyy-MM-dd HH:mm:ss")
```

---

### `format` (Time)
```nim
proc format(time: Time, f: string, zone: Timezone = local()): string
proc format(time: Time, f: static[string], zone: Timezone = local()): string
```
Форматирует `Time` в строку с использованием указанного часового пояса.

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
Адаптер для `std/strformat`. Позволяет использовать `DateTime` и `Time` в интерполяции строк.

```nim
import std/strformat
let dt = now()
echo fmt"{dt:yyyy-MM-dd}"
```

---

### `parse` (строка → DateTime)
```nim
proc parse(input: string, f: TimeFormat, zone: Timezone = local(),
    loc: DateTimeLocale = DefaultLocale): DateTime
proc parse(input, f: string, tz: Timezone = local(),
    loc: DateTimeLocale = DefaultLocale): DateTime
proc parse(input: string, f: static[string], zone: Timezone = local(),
    loc: DateTimeLocale = DefaultLocale): DateTime
```
Парсит строку в `DateTime`. Вызывает исключение `TimeParseError` при ошибке.

```nim
let f = initTimeFormat("yyyy-MM-dd")
let dt = dateTime(2000, mJan, 01, 0, 0, 0, 0, utc())
doAssert dt == "2000-01-01".parse(f, utc())

# Прямо строкой:
doAssert dt == parse("2000-01-01", "yyyy-MM-dd", utc())
```

---

### `parseTime`
```nim
proc parseTime(input, f: string, zone: Timezone): Time
proc parseTime(input: string, f: static[string], zone: Timezone): Time
```
Аналог `parse`, но возвращает `Time`.

```nim
let tStr = "1970-01-01T00:00:00+00:00"
doAssert parseTime(tStr, "yyyy-MM-dd'T'HH:mm:sszzz", utc()) == fromUnix(0)
```

---

## TimeInterval — календарный интервал

### `initTimeInterval`
```nim
proc initTimeInterval(nanoseconds = 0, microseconds = 0, milliseconds = 0,
                      seconds = 0, minutes = 0, hours = 0,
                      days = 0, weeks = 0, months = 0, years = 0): TimeInterval
```
Создаёт новый `TimeInterval`. **Не нормализует** единицы (`hours = 24 != days = 1`).

```nim
let day = initTimeInterval(hours = 24)
let dt = dateTime(2000, mJan, 01, 12, 0, 0, 0, utc())
doAssert $(dt + day) == "2000-01-02T12:00:00Z"
doAssert initTimeInterval(hours = 24) != initTimeInterval(days = 1)
```

---

### Арифметика с TimeInterval

```nim
proc `+`(ti1, ti2: TimeInterval): TimeInterval
proc `-`(ti: TimeInterval): TimeInterval         # унарный минус
proc `-`(ti1, ti2: TimeInterval): TimeInterval
proc `+=`(a: var TimeInterval, b: TimeInterval)
proc `-=`(a: var TimeInterval, b: TimeInterval)
```

```nim
let ti1 = initTimeInterval(hours = 24)
let ti2 = initTimeInterval(hours = 4)
doAssert (ti1 - ti2) == initTimeInterval(hours = 20)

let day = -initTimeInterval(hours = 24)
doAssert day.hours == -24
```

---

### Сложение/вычитание DateTime + TimeInterval

```nim
proc `+`(dt: DateTime, interval: TimeInterval): DateTime
proc `-`(dt: DateTime, interval: TimeInterval): DateTime
proc `+=`(a: var DateTime, b: TimeInterval)
proc `-=`(a: var DateTime, b: TimeInterval)
```

```nim
let dt = dateTime(2017, mMar, 30, 0, 0, 0, 0, utc())
doAssert $(dt + 1.months) == "2017-04-30T00:00:00Z"
doAssert $(dt - 1.months) == "2017-03-02T00:00:00Z"   # переполнение дня!
```

---

### Сложение/вычитание Time + TimeInterval

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
Вычисляет разницу между двумя `DateTime` в виде `TimeInterval`. Гарантирует, что `startDt + between(startDt, endDt) == endDt` (при одинаковом часовом поясе).

```nim
var a = dateTime(2015, mMar, 25, 12, 0, 0, 0, utc())
var b = dateTime(2017, mApr, 1, 15, 0, 15, 0, utc())
var ti = initTimeInterval(years = 2, weeks = 1, hours = 3, seconds = 15)
doAssert between(a, b) == ti
doAssert between(a, b) == -between(b, a)
```

---

### `toParts` (для TimeInterval)
```nim
proc toParts(ti: TimeInterval): TimeIntervalParts
```
Разбивает `TimeInterval` на массив, индексируемый `TimeUnit`.

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
Человекочитаемое строковое представление.

```nim
doAssert $initTimeInterval(years = 1, nanoseconds = 123) == "1 year and 123 nanoseconds"
doAssert $initTimeInterval() == "0 nanoseconds"
```

---

## Удобные конструкторы интервалов

Эти функции позволяют использовать синтаксис `5.seconds`, `2.hours` и т.д.

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

# Можно комбинировать:
let dt = now() + 1.years + 2.months + 3.days
```

---

## Системное время и бенчмаркинг

### `epochTime`
```nim
proc epochTime(): float
```
Возвращает количество секунд с начала Unix-эпохи (1970) в виде `float`. Имеет субсекундное разрешение, но **не подходит для бенчмаркинга**.

> Предпочтительнее использовать `getTime()`.

```nim
let t = epochTime()
# ... какой-то код ...
echo "Прошло секунд: ", epochTime() - t
```

---

### `cpuTime`
```nim
proc cpuTime(): float
```
Возвращает время CPU, затраченное текущим процессом (в секундах). Полезнее для бенчмаркинга чем `epochTime`. Может измерять реальное время в зависимости от ОС.

```nim
var t0 = cpuTime()
var fib = @[0, 1, 1]
for i in 1..10:
  fib.add(fib[^1] + fib[^2])
echo "CPU time [s]: ", cpuTime() - t0
```

---

## Паттерны форматирования

| Паттерн | Описание | Пример |
|---|---|---|
| `d` | День месяца (1–2 цифры) | `1` или `21` |
| `dd` | День месяца (всегда 2 цифры) | `01` или `21` |
| `ddd` | Сокращённое название дня | `Sat`, `Mon` |
| `dddd` | Полное название дня | `Saturday` |
| `GG` | Последние 2 цифры ISO-года | `13` |
| `GGGG` | ISO-год (4 цифры) | `2013` |
| `h` | Час 1–12 | `5` (pm), `2` (am) |
| `hh` | Час 1–12 (2 цифры) | `05`, `11` |
| `H` | Час 0–23 | `17`, `2` |
| `HH` | Час 0–23 (2 цифры) | `17`, `02` |
| `m` | Минуты (1–2 цифры) | `30`, `1` |
| `mm` | Минуты (2 цифры) | `30`, `01` |
| `M` | Месяц (1–2 цифры) | `9`, `12` |
| `MM` | Месяц (2 цифры) | `09`, `12` |
| `MMM` | Сокращённое название месяца | `Sep`, `Dec` |
| `MMMM` | Полное название месяца | `September` |
| `s` | Секунды (1–2 цифры) | `6` |
| `ss` | Секунды (2 цифры) | `06` |
| `t` | AM/PM (1 символ) | `A`, `P` |
| `tt` | AM/PM (2 символа) | `AM`, `PM` |
| `yy` | Последние 2 цифры года | `12` |
| `yyyy` | Год (мин. 4 цифры, всегда положительный) | `2012`, `+12345` |
| `YYYY` | Год (без паддинга, всегда положительный) | `2012`, `24` |
| `uuuu` | Год (мин. 4 цифры, может быть отрицательным для до н.э.) | `-0023` |
| `UUUU` | Год (без паддинга, может быть отрицательным) | `-23` |
| `V` | ISO номер недели (1–2 цифры) | `5`, `13` |
| `VV` | ISO номер недели (2 цифры) | `05`, `13` |
| `z` | Смещение UTC | `+7`, `-5` |
| `zz` | Смещение UTC (с нулём) | `+07`, `-05` |
| `zzz` | Смещение UTC с минутами | `+07:00` |
| `ZZZ` | Смещение UTC с минутами (без двоеточия) | `+0700` |
| `zzzz` | Смещение UTC с секундами | `+07:00:00` |
| `ZZZZ` | Смещение UTC с секундами (без двоеточий) | `+070000` |
| `g` | Эра: AD или BC | `AD`, `BC` |
| `fff` | Миллисекунды | `1` |
| `ffffff` | Микросекунды | `1000` |
| `fffffffff` | Наносекунды | `1000000` |

**Литералы** вставляются в одинарных кавычках: `hh'->'mm` → `01->56`.  
Без кавычек можно вставлять: `:`, `-`, `,`, `.`, `(`, `)`, `/`, `[`, `]`, пробел.

---

## Duration vs TimeInterval

| Характеристика | `Duration` | `TimeInterval` |
|---|---|---|
| Хранение | Секунды + наносекунды | Отдельные поля для каждой единицы |
| Нормализация | Всегда нормализован | Не нормализован |
| Производительность | Быстрая | Медленная (требует инфо о часовом поясе) |
| Поддержка лет/месяцев | ❌ | ✅ |
| Когда использовать | Большинство случаев | Только когда нужны годы/месяцы |

**Важно о днях:** `Duration` считает день всегда равным 86400 секундам. `TimeInterval` работает с календарными днями — например, при переходе на летнее время один день может быть равен 25 часам.

```nim
# Duration — фиксированные секунды
let dur = initDuration(hours = 25)  # ровно 90000 секунд

# TimeInterval — календарный день
let ti = initTimeInterval(days = 1)  # ровно 1 calendar день
```
