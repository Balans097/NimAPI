- [x] # times — справочник модуля

> **Импорт:** `import std/times`
> **Область применения:** работа с датой и временем по пролептическому григорианскому календарю — точки во времени, длительности, разбор времени на компоненты, форматирование/парсинг строк, часовые пояса.

Модуль решает четыре в целом независимые задачи, вокруг которых он и построен:
хранение "сырой" точки во времени (`Time`), представление времени в виде набора
человекочитаемых полей (`DateTime`), выражение длительности либо как фиксированного
числа секунд (`Duration`), либо как календарного интервала ("1 год и 2 дня",
`TimeInterval`), и, наконец, перевод между часовыми поясами через объекты `Timezone`.
Общая конвенция модуля: там, где операция допускает и точную (`Duration`), и
календарную (`TimeInterval`) трактовку, для обеих есть отдельный набор
перегруженных операторов `+`/`-`, а какую из них использовать — решает сам
вызывающий код, в зависимости от того, нужна ли ему точность в секундах или
устойчивость к переходам через дни/месяцы/года.

---

## Оглавление

I. [Основные типы модуля](#основные-типы-модуля)
   1. [`Time`](#time)
   2. [`DateTime`](#datetime)
   3. [`Duration`](#duration)
   4. [`TimeInterval`](#timeinterval)
   5. [`Timezone` и `ZonedTime`](#timezone-и-zonedtime)

II. [Duration — фиксированные интервалы времени](#duration--фиксированные-интервалы-времени)
   1. [`initDuration` и `DurationZero`](#initduration-и-durationzero)
   2. [Извлечение величины: `inWeeks`, `inDays`, `inHours`, `inMinutes`, `inSeconds`, `inMilliseconds`, `inMicroseconds`, `inNanoseconds`](#извлечение-величины)
   3. [`toParts` (Duration)](#toparts-duration)
   4. [Арифметика и сравнение `Duration`](#арифметика-и-сравнение-duration)
   5. [`high`, `low`, `abs` (Duration)](#high-low-abs-duration)
   6. [`$` (Duration)](#-duration)

III. [Time — точка во времени](#time--точка-во-времени)
   1. [`initTime` и `nanosecond`](#inittime-и-nanosecond)
   2. [`fromUnix` / `toUnix`](#fromunix--tounix)
   3. [`fromWinTime` / `toWinTime`](#fromwintime--towintime)
   4. [`getTime`](#gettime)
   5. [Арифметика и сравнение `Time`](#арифметика-и-сравнение-time)
   6. [`high`, `low` (Time)](#high-low-time)
   7. [`$` (Time)](#-time)

IV. [DateTime — время, разобранное на поля](#datetime--время-разобранное-на-поля)
   1. [Поля-аксессоры DateTime](#поля-аксессоры-datetime)
   2. [`dateTime` и устаревшие `initDateTime`](#datetime-и-устаревшие-initdatetime)
   3. [`toTime`](#totime)
   4. [`isLeapDay`](#isleapday)
   5. [Арифметика и сравнение `DateTime` через `Duration`](#арифметика-и-сравнение-datetime-через-duration)
   6. [`getDateStr` / `getClockStr`](#getdatestr--getclockstr)
   7. [`$` (DateTime)](#-datetime)

V. [Часовые пояса](#часовые-пояса)
   1. [`Timezone`, `newTimezone`](#timezone-newtimezone)
   2. [`utc` / `local` (получение зоны и перевод в неё)](#utc--local)
   3. [`inZone`](#inzone)
   4. [`name`, `$`, `==` для `Timezone`](#name---для-timezone)
   5. [`now`](#now)

VI. [Форматирование и парсинг дат](#форматирование-и-парсинг-дат)
   1. [Синтаксис формат-строк](#синтаксис-формат-строк)
   2. [`initTimeFormat` и `TimeFormat`](#inittimeformat-и-timeformat)
   3. [`format`](#format)
   4. [`parse` / `parseTime`](#parse--parsetime)
   5. [`DateTimeLocale`](#datetimelocale)

VII. [TimeInterval — календарные интервалы](#timeinterval--календарные-интервалы)
   1. [`initTimeInterval`](#inittimeinterval)
   2. [Конструкторы отдельных единиц: `nanoseconds` … `years`](#конструкторы-отдельных-единиц)
   3. [Арифметика `TimeInterval`](#арифметика-timeinterval)
   4. [`toParts` и `$` (TimeInterval)](#toparts-и--timeinterval)
   5. [Арифметика `DateTime`/`Time` с `TimeInterval`](#арифметика-datetimetime-с-timeinterval)
   6. [`between`](#between)

VIII. [Календарные вычисления и ISO-недели](#календарные-вычисления-и-iso-недели)
   1. [`isLeapYear`, `getDaysInMonth`, `getDaysInYear`](#isleapyear-getdaysinmonth-getdaysinyear)
   2. [`getDayOfYear`, `getDayOfWeek`](#getdayofyear-getdayofweek)
   3. [`IsoYear`, `getWeeksInIsoYear`, `getIsoWeekAndYear`](#isoyear-getweeksinisoyear-getisoweekandyear)
   4. [`initDateTime` по ISO-неделе](#initdatetime-по-iso-неделе)

IX. [Прочие процедуры](#прочие-процедуры)
   1. [`convert`](#convert)
   2. [`epochTime`](#epochtime)
   3. [`cpuTime`](#cputime)

X. [Устаревшие сеттеры полей DateTime](#устаревшие-сеттеры-полей-datetime)

XI. [Практические рецепты](#практические-рецепты)
   1. [Замер длительности операции](#1-замер-длительности-операции)
   2. [Разбор и форматирование дат из внешнего источника](#2-разбор-и-форматирование-дат-из-внешнего-источника)
   3. [Человекочитаемый возраст/стаж](#3-человекочитаемый-возрастстаж)
   4. [Перевод отметки времени между часовыми поясами](#4-перевод-отметки-времени-между-часовыми-поясами)
   5. [Планировщик "раз в N дней"](#5-планировщик-раз-в-n-дней)

XII. [Краткая таблица](#краткая-таблица)

XIII. [Сводка: какую процедуру выбрать](#сводка-какую-процедуру-выбрать)

---

## Основные типы модуля

### `Time`

```nim
Time* = object
  seconds: int64
  nanosecond: NanosecondRange
```

**Функция.** `Time` — это "сырая" точка во времени: секунды от эпохи Unix
(1970-01-01T00:00:00 UTC) плюс наносекундная добавка. У неё нет ни часового
пояса, ни разбиения на год/месяц/день — это просто число с наносекундной
точностью, которое одинаково для всех наблюдателей на Земле. Все поля
приватны: наружу `Time` показывает себя только через процедуры `toUnix`,
`nanosecond`, арифметику и сравнение.

- **Разбор реализации.** Хранение в виде пары (секунды, наносекунды), а не
  одного числа наносекунд, выбрано ради диапазона: `int64` наносекунд
  переполнился бы уже около 2262 года, а пара (секунды: int64, наносекунды:
  0..999999999) даёт тот же диапазон, что и `time_t`, с полной наносекундной
  точностью внутри каждой секунды.

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

**Функция.** `DateTime` представляет тот же момент времени, что и `Time`, но
уже разложенный на календарные поля (год, месяц, день, час...) в конкретном
часовом поясе. В отличие от `Time`, значение `DateTime` без явно указанного
часового пояса не имеет смысла — поэтому у типа есть поле `timezone`, а
получить "просто дату" без пояса нельзя. Значение по умолчанию (`default(DateTime)`)
считается неинициализированным — большинство процедур модуля проверяют это
через внутренний `assertDateTimeInitialized` и подают ассерт при попытке
прочитать поля пустого `DateTime`.

- **Список параметров/полей** (только для чтения, через одноимённые процедуры-аксессоры):
  - `nanosecond`, `second`, `minute`, `hour` — время суток;
  - `monthday`, `month`, `year` — календарная дата;
  - `weekday`, `yearday` — производные поля, вычисляются автоматически;
  - `isDst` — действует ли в этот момент летнее время;
  - `timezone` — часовой пояс, в котором представлено значение;
  - `utcOffset` — смещение от UTC в секундах (со знаком, обратным принятому
    в строковых офсетах вида `+01:00`).

---

### `Duration`

```nim
Duration* = object
  seconds: int64
  nanosecond: NanosecondRange
```

**Функция.** `Duration` — фиксированная длительность: столько-то секунд и
наносекунд, всегда нормализованная (`initDuration(hours = 1)` и
`initDuration(minutes = 60)` дают одинаковое значение). Сутки для `Duration`
всегда равны ровно 86400 секундам. Это делает арифметику с `Duration`
дешёвой (просто сложение/вычитание целых чисел) и предсказуемой, поэтому
модуль рекомендует использовать именно `Duration`, если не нужна поддержка
месяцев и лет.

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

**Функция.** `TimeInterval` — календарный интервал вроде "1 год и 2 дня":
каждая единица хранится отдельным полем и **не** нормализуется (даже секунды
и миллисекунды не переносятся друг в друга), потому что единицы вроде года
или месяца в принципе не имеют фиксированной длины в секундах (год бывает
365 или 366 дней). Из-за этого арифметика с `TimeInterval` требует
информации о часовом поясе и может быть заметно медленнее, чем с `Duration`.
Разница особенно заметна на переходах летнего времени: между
`2018-03-25T12:00+02:00` и `2018-03-26T12:00+01:00` прошли календарные
"ровно одни сутки" (`TimeInterval`), но фактически 25 часов = 90000 секунд
(`Duration`), потому что где-то между ними сместился офсет.

---

### `Timezone` и `ZonedTime`

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

**Функция.** `Timezone` — интерфейс часового пояса: пара функций-конвертеров
(`Time -> ZonedTime` и "локальное время" -> `ZonedTime`) плюс имя.
Сам модуль `times` поставляет только реализации для UTC и системного
локального пояса — поддержка произвольных таймзон (базы IANA tzdata и т.п.)
реализуется сторонними библиотеками через `newTimezone`. `ZonedTime` —
вспомогательный тип: точка времени плюс UTC-офсет и флаг летнего времени,
используется только при реализации таймзон.

---
## Duration — фиксированные интервалы времени

### `initDuration` и `DurationZero`

```nim
const DurationZero* = Duration()

proc initDuration*(nanoseconds, microseconds, milliseconds,
                   seconds, minutes, hours, days, weeks: int64 = 0): Duration
```

**Функция.** Создаёт `Duration` из произвольной комбинации единиц — все они
складываются и нормализуются в одну пару (секунды, наносекунды).
`DurationZero` — готовая нулевая длительность, удобна как база для сравнений.

- **Параметры:**
  - `nanoseconds`, `microseconds`, `milliseconds`, `seconds`, `minutes`,
    `hours`, `days`, `weeks` — каждая единица необязательна (по умолчанию 0),
    может быть отрицательной; итог нормализуется.

**Примеры:**

```nim
let
  dur1 = initDuration(seconds = 1)
  dur2 = initDuration(minutes = 60)
doAssert dur1 > DurationZero
doAssert dur2 == initDuration(hours = 1)
```

---

### Извлечение величины

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

**Функция.** Каждая процедура выражает `Duration` целиком в указанной
единице — с усечением (`div`), а не округлением, поэтому `inHours` для
"1 час 59 минут" вернёт `1`, а не `2`.

- **Параметры:** `dur` — длительность, которую нужно выразить в единице,
  указанной в имени процедуры.

**Примеры:**

```nim
let dur = initDuration(hours = 1, minutes = 30)
doAssert inMinutes(dur) == 90
doAssert inHours(dur) == 1          # усечение, а не округление
doAssert inSeconds(initDuration(seconds = -1)) == -1
```

---

### `toParts` (Duration)

```nim
proc toParts*(dur: Duration): DurationParts
```

**Функция.** Раскладывает `Duration` на массив по всем восьми фиксированным
единицам (`Nanoseconds`..`Weeks`), каждая — "остаток" после вычитания более
крупных единиц. Удобно для человекочитаемого вывода вида "1 неделя, 2 дня,
3 часа" без ручных `div`/`mod`.

- **Параметры:** `dur` — исходная длительность.

**Пример:**

```nim
let dur = initDuration(weeks = 1, days = 2, hours = 3)
let parts = toParts(dur)
doAssert parts[Weeks] == 1
doAssert parts[Days] == 2
doAssert parts[Hours] == 3
```

---

### Арифметика и сравнение `Duration`

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

**Функция.** Полный набор арифметических и сравнивающих операторов: сложение
и вычитание двух длительностей, унарный минус (разворот знака), умножение и
целочисленное деление на скаляр, сравнения по величине, а также
модифицирующие варианты `+=`/`-=`/`*=`. Все операции работают через
нормализацию пары (секунды, наносекунды), поэтому переполнение наносекунд
в одну сторону или другую всегда корректно переносится в секунды.

- **Разбор реализации.** Сложение/вычитание сводятся к одной внутренней
  функции нормализации: складываются (или вычитаются) секунды и наносекунды
  по отдельности, а затем наносекундная часть приводится в диапазон
  `0..999999999` с переносом "лишней" секунды — как перенос разряда при
  сложении столбиком.

**Примеры:**

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

**Функция.** `high`/`low` возвращают граничные представимые длительности,
`abs` — длительность без учёта знака (полезно, когда важна только величина
разницы, а не то, какое из двух значений времени было раньше).

- **Параметры:** `typ` — тип-плейсхолдер `Duration` (не используется как
  значение, только для выбора перегрузки); `a` — длительность для `abs`.

**Пример:**

```nim
doAssert abs(initDuration(hours = -3)) == initDuration(hours = 3)
```

---

### `$` (Duration)

```nim
proc `$`*(dur: Duration): string
```

**Функция.** Человекочитаемое представление длительности вида
`"1 hour and 30 minutes"` (только ненулевые единицы, единственное число при
величине 1). Нулевая длительность выводится как `"0 nanoseconds"`.

**Пример:**

```nim
echo $initDuration(hours = 1, minutes = 30)  # выводит "1 hour and 30 minutes"
echo $DurationZero                            # выводит "0 nanoseconds"
```

---
## Time — точка во времени

### `initTime` и `nanosecond`

```nim
proc initTime*(unix: int64, nanosecond: NanosecondRange): Time
proc nanosecond*(time: Time): NanosecondRange
```

**Функция.** `initTime` собирает `Time` из числа секунд-от-эпохи и
наносекундной добавки напрямую, минуя `fromUnix`/`getTime`. `nanosecond` —
единственный публичный аксессор наносекундной части (для секунд аксессора
нет — используйте `toUnix`).

- **Параметры:**
  - `unix: int64` — секунды от эпохи Unix (может быть отрицательным для дат
    до 1970 года);
  - `nanosecond: NanosecondRange` — наносекундная добавка, `0..999999999`.

**Пример:**

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

**Функция.** Пара конвертеров между `Time` и обычным Unix-таймстампом в
секундах (без наносекунд — `fromUnix` даёт наносекундную часть, равную 0,
`toUnix` её отбрасывает).

- **Параметры:** `unix`/`t` — таймстамп в секундах либо значение `Time`.

**Пример:**

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

**Функция.** Конвертация в/из "времени Windows" — 100-наносекундных
интервалов от 1601-01-01 (эпоха `FILETIME`). Используется в основном при
взаимодействии с Windows API или файлами, хранящими такие метки.

- **Параметры:** `win` — количество 100-наносекундных интервалов от эпохи
  Windows; `t` — значение `Time`.

**Пример:**

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

**Функция.** Возвращает текущий момент как `Time` — платформенно-зависимая
точность (на JS ограничена миллисекундами). Это низкоуровневая точка входа;
`now()` — то же самое, но сразу переведённое в `DateTime` в локальном поясе.

**Пример:**

```nim
let t1 = getTime()
let t2 = getTime()
doAssert t2 >= t1
```

---

### Арифметика и сравнение `Time`

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

**Функция.** Разность двух `Time` даёт фиксированную `Duration` (а не
`TimeInterval` — для этого нужен `DateTime` и `between`), к `Time` можно
прибавлять и вычитать `Duration`, сравнения работают напрямую по
внутренним секундам/наносекундам.

- **Параметры:** `a`, `b` — значения `Time` либо `Duration`, в зависимости
  от перегрузки.

**Пример:**

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

**Функция.** Граничные представимые значения `Time` — удобны как
"бесконечно далёкое будущее/прошлое" при инициализации сравнений.

---

### `$` (Time)

```nim
proc `$`*(time: Time): string
```

**Функция.** Строковое представление `Time` — переводит значение в
локальный часовой пояс и форматирует по шаблону `yyyy-MM-dd'T'HH:mm:sszzz`.

**Пример:**

```nim
let dt = dateTime(1970, mJan, 01, 00, 00, 00, 00, utc())
let tm = toTime(dt)
echo $tm  # выводит "1970-01-01T00:00:00" + смещение локального пояса
```

---
## DateTime — время, разобранное на поля

### Поля-аксессоры DateTime

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

**Функция.** Набор простых аксессоров только для чтения — каждый
возвращает соответствующее поле `DateTime`. `weekday` и `yearday` не
хранятся отдельно вводом пользователя, а вычисляются модулем в момент
создания `DateTime` (через `getDayOfWeek`/`getDayOfYear`), так что они
всегда согласованы с датой. Каждый аксессор проверяет, что `dt`
инициализирован, и падает с ассертом на "пустом" `DateTime`.

| Аксессор | Тип | Значение |
|---|---|---|
| `nanosecond` | `NanosecondRange` | наносекунды в текущей секунде |
| `second` | `SecondRange` | секунда (0..60, с учётом возможной високосной) |
| `minute` | `MinuteRange` | минута |
| `hour` | `HourRange` | час (0..23) |
| `monthday` | `MonthdayRange` | день месяца (1..31) |
| `month` | `Month` | месяц |
| `year` | `int` | год (может быть отрицательным, до н.э.) |
| `weekday` | `WeekDay` | день недели — вычисляется автоматически |
| `yearday` | `YeardayRange` | день года (0..365) — вычисляется автоматически |
| `isDst` | `bool` | действует ли летнее время |
| `timezone` | `Timezone` | часовой пояс значения |
| `utcOffset` | `int` | смещение от UTC в секундах |

**Пример:**

```nim
let dt = dateTime(2020, mFeb, 29, 12, 30, 00, 00, utc())
doAssert year(dt) == 2020
doAssert month(dt) == mFeb
doAssert monthday(dt) == 29
doAssert weekday(dt) == dSat  # вычислено автоматически, не задавалось явно
```

---

### `dateTime` и устаревшие `initDateTime`

```nim
proc dateTime*(year: int, month: Month, monthday: MonthdayRange,
               hour: HourRange = 0, minute: MinuteRange = 0, second: SecondRange = 0,
               nanosecond: NanosecondRange = 0,
               zone: Timezone = local()): DateTime
```

**Функция.** Основной конструктор `DateTime` — по календарным полям и
часовому поясу. Порядок аргументов `year, month, monthday` (в отличие от
двух устаревших перегрузок `initDateTime`, где день шёл первым) выбран как
более естественный для чтения "год-месяц-день". Внутри дата сначала
собирается как "локальное" время (`toAdjTime`), а затем переводится в
реальный `Time` через `zonedTimeFromAdjTime` конкретного пояса — так
корректно учитываются переходы летнего времени на границе полуночи.

- **Параметры:**
  - `year: int` — год (отрицательные значения — до нашей эры);
  - `month: Month` — месяц;
  - `monthday: MonthdayRange` — день месяца, обязателен;
  - `hour`, `minute`, `second`, `nanosecond` — время суток, по умолчанию 0;
  - `zone: Timezone` — часовой пояс, по умолчанию `local()`.

Две устаревшие перегрузки `initDateTime*(monthday, month, year, ...)`
делают то же самое, но с другим порядком первых трёх параметров, и
помечены `{.deprecated: "use dateTime".}` — в новом коде их использовать
не стоит.

**Примеры:**

```nim
let dt = dateTime(2017, mMar, 30, zone = utc())
doAssert $dt == "2017-03-30T00:00:00Z"

doAssertRaises(AssertionDefect):
  discard dateTime(2021, mFeb, 29, zone = utc())  # 2021 — не високосный
```

---

### `toTime`

```nim
proc toTime*(dt: DateTime): Time
```

**Функция.** Переводит `DateTime` обратно в "сырой" `Time` — момент времени
без привязки к часовому поясу. Это обратная операция к `inZone`.

- **Разбор реализации.** Дата сначала переводится в "epoch day" (число дней
  от 1970-01-01) алгоритмом Ховарда Хиннанта (`toEpochDay`), основанным на
  делении с учётом 400-летней эры Григорианского календаря — этот приём
  позволяет корректно работать и с отрицательными годами без веток на
  каждый конкретный век. Epoch day домножается на число секунд в сутках,
  прибавляются часы/минуты/секунды и `utcOffset` — получаются секунды от
  эпохи Unix.

**Пример:**

```nim
let dt = dateTime(1970, mJan, 01, 00, 00, 00, 00, utc())
doAssert toUnix(toTime(dt)) == 0
```

---

### `isLeapDay`

```nim
proc isLeapDay*(dt: DateTime): bool
```

**Функция.** Проверяет, что дата — именно 29 февраля високосного года. Это
важно для арифметики: прибавление и вычитание одного года подряд к 29
февраля не обязано вернуть исходную дату (следующий год может быть не
високосным).

**Пример:**

```nim
let dt = dateTime(2020, mFeb, 29, 00, 00, 00, 00, utc())
doAssert isLeapDay(dt)
doAssert dt + 1.years - 1.years != dt  # 29 февраля "не переживает" не-високосный год
```

---

### Арифметика и сравнение `DateTime` через `Duration`

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

**Функция.** Позволяют прибавлять/вычитать фиксированную `Duration`
непосредственно к `DateTime` (результат остаётся в том же часовом поясе), а
разность двух `DateTime` даёт точную `Duration` — в секундах, без
календарной трактовки. Сравнения работают по фактическому моменту времени
(через приведение к `Time`), а не по значениям полей напрямую.

- **Разбор реализации.** Все операции реализованы через промежуточный
  перевод в `Time` (`toTime`) и обратно (`inZone`) — то есть арифметика с
  `Duration` для `DateTime` — это, по сути, арифметика с `Time`,
  обёрнутая переводом в/из часового пояса.

**Пример:**

```nim
let dt = dateTime(2020, mJan, 01, 00, 00, 00, 00, utc())
let later = dt + initDuration(hours = 25)  # переходит на следующий день
doAssert monthday(later) == 2
doAssert later - dt == initDuration(hours = 25)
```

---

### `getDateStr` / `getClockStr`

```nim
proc getDateStr*(dt = now()): string
proc getClockStr*(dt = now()): string
```

**Функция.** Готовые короткие форматтеры: `getDateStr` даёт дату вида
`"2020-01-01"`, `getClockStr` — время суток вида `"12:30:00"`. Удобны,
когда не нужен полный контроль формата через `format`.

- **Параметры:** `dt` — значение `DateTime`, по умолчанию текущий момент
  (`now()`).

**Пример:**

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

**Функция.** Строковое представление по формату
`yyyy-MM-dd'T'HH:mm:sszzz` (ISO 8601-подобный формат). Для
неинициализированного `DateTime` (значение по умолчанию) возвращает
строку `"Uninitialized DateTime"` вместо падения с ошибкой.

**Пример:**

```nim
let dt = dateTime(2000, mJan, 01, 12, 00, 00, 00, utc())
doAssert $dt == "2000-01-01T12:00:00Z"
doAssert $default(DateTime) == "Uninitialized DateTime"
```

---
## Часовые пояса

### `Timezone`, `newTimezone`

```nim
proc newTimezone*(
      name: string,
      zonedTimeFromTimeImpl: proc (time: Time): ZonedTime {.tags: [], raises: [], gcsafe.},
      zonedTimeFromAdjTimeImpl: proc (adjTime: Time): ZonedTime {.tags: [], raises: [], gcsafe.}
    ): owned Timezone
```

**Функция.** Конструктор произвольного часового пояса из двух функций
конвертации (из "сырого" `Time` в зонированное представление и из
"локального времени" — тоже как `Time`, но трактуемого не как момент, а
как показание часов в этом поясе). Используется, когда нужен пояс, которого
нет среди встроенных `utc`/`local` — например, при подключении сторонней
базы IANA tzdata.

- **Параметры:**
  - `name: string` — по возможности имя из базы tz (например,
    `"Europe/Amsterdam"`), иначе — любая уникальная строка (влияет на
    сравнение `==` между поясами);
  - `zonedTimeFromTimeImpl` — функция "момент времени -> зонированное время";
  - `zonedTimeFromAdjTimeImpl` — функция "локальные часы -> зонированное время".

**Пример:**

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

**Функция.** Первая пара возвращает сами объекты `Timezone` — UTC и
системный локальный пояс (закешированы в потокоопасных `threadvar`, чтобы
не пересоздавать их на каждый вызов). Остальные четыре перегрузки — это
удобные сокращения для `inZone(utc())`/`inZone(local())`: переводят уже
существующий `DateTime` или `Time` в указанный пояс.

**Пример:**

```nim
doAssert name(utc()) == "Etc/UTC"
let t = fromUnix(0)
let dtUtc = utc(t)      # DateTime в UTC
doAssert timezone(dtUtc) == utc()
```

---

### `inZone`

```nim
proc inZone*(time: Time, zone: Timezone): DateTime
proc inZone*(dt: DateTime, zone: Timezone): DateTime
```

**Функция.** Универсальный перевод в произвольный часовой пояс: и `Time` в
`DateTime`, и уже имеющийся `DateTime` — в тот же момент времени, но
представленный в другом поясе.

- **Параметры:** `time`/`dt` — исходное значение; `zone` — целевой пояс.

**Пример:**

```nim
let t = fromUnix(0)
let dtUtc = inZone(t, utc())
doAssert $dtUtc == "1970-01-01T00:00:00Z"
```

---

### `name`, `$`, `==` для `Timezone`

```nim
proc name*(zone: Timezone): string
proc `$`*(zone: Timezone): string
proc `==`*(zone1, zone2: Timezone): bool
```

**Функция.** `name` — сохранённое имя пояса, `$` — то же самое как строка
(пустая строка для `nil`), `==` сравнивает пояса по имени (а не по
идентичности объекта) — то есть два разных `Timezone`, созданных с одним
именем, считаются равными.

**Пример:**

```nim
doAssert local() == local()
doAssert local() != utc()
```

---

### `now`

```nim
proc now*(): DateTime
```

**Функция.** Сокращение для `local(getTime())` — текущий момент времени в
виде `DateTime` в локальном часовом поясе. Не годится для бенчмаркинга —
для этого в модуле есть `cpuTime`.

---
## Форматирование и парсинг дат

### Синтаксис формат-строк

Формат-строка описывает шаблон, по которому строится или разбирается
дата. Основные паттерны (полный список см. в комментариях исходного
модуля):

| Паттерн | Значение | Пример |
|---|---|---|
| `d`, `dd` | день месяца (1-2 цифры) | `1`, `01` |
| `ddd`, `dddd` | день недели, кратко/полностью | `Sat`, `Saturday` |
| `M`, `MM`, `MMM`, `MMMM` | месяц: число, число с нулём, кратко, полностью | `9`, `09`, `Sep`, `September` |
| `yyyy`, `yy` | год с полным/усечённым числом цифр | `2012`, `12` |
| `H`, `HH` | час 0-23 | `2`, `02` |
| `h`, `hh`, `t`, `tt` | час 1-12 и обозначение AM/PM | `5`, `05`, `P`, `PM` |
| `m`, `mm`, `s`, `ss` | минуты, секунды | `30`, `06` |
| `z`, `zz`, `zzz`, `zzzz` | смещение от UTC в разных видах | `+7`, `+07:00` |
| `fff`, `ffffff`, `fffffffff` | доли секунды: милли-/микро-/наносекунды | `1`, `1000`, `1000000` |
| `g` | эра (`AD`/`BC`) | `AD` |

Произвольный текст внутри формата экранируется одинарными кавычками:
`hh'->'mm` даст, например, `01->56`. Символы `: - , . ( ) / [ ]` и пробел
можно вставлять без экранирования.

---

### `initTimeFormat` и `TimeFormat`

```nim
proc initTimeFormat*(format: string): TimeFormat
proc `$`*(f: TimeFormat): string
```

**Функция.** `initTimeFormat` разбирает формат-строку один раз и превращает
её в `TimeFormat` — предварительно скомпилированное представление
(последовательность паттернов и литеральных вставок), которое затем
многократно используется в `format`/`parse` без повторного парсинга
строки формата. Перегрузки `format`/`parse`, принимающие `static[string]`,
вызывают `initTimeFormat` на этапе компиляции — так ошибка в самом формате
(а не в данных) обнаруживается сразу, а не в рантайме.

- **Параметры:** `format: string` — формат-строка (см. таблицу паттернов выше).

**Пример:**

```nim
let f = initTimeFormat("yyyy-MM-dd")
echo $f  # выводит обратно "yyyy-MM-dd"
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

**Функция.** Форматирует `DateTime` или `Time` по заданному шаблону.
Перегрузки для `Time` дополнительно принимают `zone` — время сначала
переводится в указанный пояс, затем форматируется. Перегрузки со
`static[string]` компилируют формат во время компиляции (см. выше), обычная
строковая перегрузка — в рантайме и может кинуть `TimeFormatParseError`
при некорректном формате.

- **Параметры:**
  - `dt`/`time` — значение для форматирования;
  - `f` — формат: готовый `TimeFormat`, обычная строка либо
    строка, известная на этапе компиляции;
  - `loc: DateTimeLocale` — локаль (названия месяцев/дней недели), по
    умолчанию английская `DefaultLocale`;
  - `zone: Timezone` — пояс для перегрузок с `Time`.

**Пример:**

```nim
let dt = dateTime(2000, mJan, 01, 00, 00, 00, 00, utc())
doAssert format(dt, "yyyy-MM-dd") == "2000-01-01"

var tm = toTime(dt)
doAssert format(tm, "yyyy-MM-dd'T'HH:mm:ss", utc()) == "1970-01-01T00:00:00" # пример границы: другой tm ниже
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

**Функция.** Разбирает строку `input` по формату `f` и возвращает
`DateTime` (или сразу `Time` — через `parseTime`). Если в строке был
явный офсет UTC (паттерны `z`/`zz`/`zzz`/`zzzz`), результат переводится в
`zone`; если офсета не было, строка трактуется как уже записанная во
`zone`. При несовпадении строки с форматом кидается `TimeParseError`.

- **Разбор реализации.** Формат и входная строка обходятся синхронно: для
  литеральных участков посимвольно сверяется точное совпадение, для
  паттернов вызывается разбор конкретной единицы (`parsePattern`),
  накапливающий результат в промежуточную структуру `ParsedTime` — только
  после полного прохода она конвертируется в готовый `DateTime`
  (`toDateTime`/`toDateTimeByWeek`, в зависимости от того, парсились ли
  обычные год/месяц/день или ISO-неделя/ISO-год).

- **Параметры:**
  - `input: string` — строка с датой;
  - `f` — формат разбора (готовый `TimeFormat`, строка, либо строка,
    известная на этапе компиляции);
  - `zone`/`tz` — пояс, в котором трактовать дату при отсутствии
    явного офсета;
  - `loc: DateTimeLocale` — локаль названий месяцев/дней недели.

**Примеры:**

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
*(упрощённо — фактические названия полей в модуле группируют строки по
коротким/длинным формам месяцев и дней недели.)*

**Функция.** Набор строк, используемых при форматировании/парсинге "по
буквам" (`MMM`, `MMMM`, `ddd`, `dddd`) — позволяет форматировать и
разбирать даты на языке, отличном от английского. `DefaultLocale` —
встроенная англоязычная локаль, используемая по умолчанию.

---
## TimeInterval — календарные интервалы

### `initTimeInterval`

```nim
proc initTimeInterval*(nanoseconds = 0, microseconds = 0, milliseconds = 0,
                       seconds = 0, minutes = 0, hours = 0,
                       days = 0, weeks = 0, months = 0, years = 0): TimeInterval
```

**Функция.** Конструктор `TimeInterval` из произвольной комбинации
календарных единиц. В отличие от `initDuration`, здесь **нет**
нормализации — переданные `seconds = 90` так и останутся полем `seconds =
90`, не превратятся в `minutes = 1, seconds = 30`.

- **Параметры:** `nanoseconds`, `microseconds`, `milliseconds`, `seconds`,
  `minutes`, `hours`, `days`, `weeks`, `months`, `years` — каждая единица
  по умолчанию 0, может быть отрицательной.

**Пример:**

```nim
let ti = initTimeInterval(years = 2, weeks = 1, hours = 3, seconds = 15)
```

---

### Конструкторы отдельных единиц

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

**Функция.** Короткие однопольные конструкторы `TimeInterval` — каждый
эквивалентен вызову `initTimeInterval` с одним заполненным полем. Основное
назначение — читаемая арифметика вида "текущий момент плюс один год":
за счёт этих процедур выражение `1.years` читается почти как обычная
единица измерения.

- **Параметры:** единственное число (`nanos`, `micros`, ..., `y`) — величина
  соответствующей единицы.

**Пример:**

```nim
let dt = dateTime(2020, mJan, 01, zone = utc())
doAssert dt + years(1) == dateTime(2021, mJan, 01, zone = utc())
doAssert dt + 1.years == dateTime(2021, mJan, 01, zone = utc())  # то же самое
```

---

### Арифметика `TimeInterval`

```nim
proc `+`*(ti1, ti2: TimeInterval): TimeInterval
proc `-`*(ti: TimeInterval): TimeInterval
proc `-`*(ti1, ti2: TimeInterval): TimeInterval
proc `+=`*(a: var TimeInterval, b: TimeInterval)
proc `-=`*(a: var TimeInterval, b: TimeInterval)
```

**Функция.** Складывает/вычитает два интервала **по каждому полю
отдельно**, без переноса между единицами (сложение `initTimeInterval(hours
= 20)` и `initTimeInterval(hours = 10)` даст `hours = 30`, а не `days = 1,
hours = 6`). Унарный минус меняет знак у всех полей разом.

**Пример:**

```nim
let a = initTimeInterval(hours = 20)
let b = initTimeInterval(hours = 10)
doAssert (a + b).hours == 30  # поля не нормализуются
doAssert (-a).hours == -20
```

---

### `toParts` и `$` (TimeInterval)

```nim
proc toParts*(ti: TimeInterval): TimeIntervalParts
proc `$`*(ti: TimeInterval): string
```

**Функция.** `toParts` — массив всех десяти полей интервала (для удобного
перебора), `$` — человекочитаемая строка вида `"2 years, 1 week, 3 hours,
and 15 seconds"`.

**Пример:**

```nim
let ti = initTimeInterval(years = 2, weeks = 1, hours = 3, seconds = 15)
echo $ti  # выводит "2 years, 1 week, 3 hours, and 15 seconds"
```

---

### Арифметика `DateTime`/`Time` с `TimeInterval`

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

**Функция.** Прибавление/вычитание календарного интервала к моменту
времени — единицы применяются от крупных к мелким (сначала года и месяцы,
затем недели/дни, затем время суток), с учётом переходов через границы
месяцев/лет и (для `DateTime`) часового пояса.

- **Параметры:** `dt`/`time` — исходный момент; `interval: TimeInterval` —
  на сколько сдвинуть.

**Пример:**

```nim
doAssert dateTime(2017, mFeb, 01, zone = utc()) + 1.months ==
  dateTime(2017, mMar, 01, zone = utc())
```

---

### `between`

```nim
proc between*(startDt, endDt: DateTime): TimeInterval
```

**Функция.** Вычисляет разницу между двумя `DateTime` как `TimeInterval` —
в отличие от простого вычитания (которое даёт `Duration`), результат
выражен в календарных единицах ("2 года, 1 неделя, 3 часа, 15 секунд"), а
не в общем числе секунд. Гарантируется: все поля результата одного знака,
и если оба `DateTime` в одном часовом поясе, то `startDt + between(startDt,
endDt) == endDt`.

- **Разбор реализации.** Года, месяцы и дни/недели вычисляются
  последовательно и "жадно" — сначала максимальное целое число полных лет,
  укладывающееся между датами, затем из остатка — максимальное число
  месяцев, и так далее до дней; остаток времени суток (часы и мельче)
  досчитывается напрямую через вычитание `Time`. Если время суток конца
  интервала раньше времени суток начала (например, начало в 15:00, конец в
  12:00 следующего дня), из календарной части сначала "занимается" один
  день — аналогично тому, как при вычитании чисел в столбик занимают
  разряд у соседней позиции.

- **Параметры:** `startDt`, `endDt` — начало и конец интервала (порядок не
  важен: `between(a, b) == -between(b, a)`).

**Пример:**

```nim
let a = dateTime(2015, mMar, 25, 12, 0, 0, 00, utc())
let b = dateTime(2017, mApr, 01, 15, 0, 15, 00, utc())
let ti = initTimeInterval(years = 2, weeks = 1, hours = 3, seconds = 15)
doAssert between(a, b) == ti
doAssert between(a, b) == -between(b, a)
```

---
## Календарные вычисления и ISO-недели

### `isLeapYear`, `getDaysInMonth`, `getDaysInYear`

```nim
proc isLeapYear*(year: int): bool
proc getDaysInMonth*(month: Month, year: int): int
proc getDaysInYear*(year: int): int
```

**Функция.** Базовые календарные предикаты: `isLeapYear` — по стандартному
григорианскому правилу (делится на 4, но не на 100, если не делится ещё и
на 400); `getDaysInMonth` — число дней в месяце с учётом года (для
февраля); `getDaysInYear` — 365 или 366.

**Пример:**

```nim
doAssert isLeapYear(2000)
doAssert not isLeapYear(1900)  # делится на 100, но не на 400
doAssert getDaysInMonth(mFeb, 2000) == 29
doAssert getDaysInYear(2000) == 366
```

---

### `getDayOfYear`, `getDayOfWeek`

```nim
proc getDayOfYear*(monthday: MonthdayRange, month: Month, year: int): YeardayRange
proc getDayOfWeek*(monthday: MonthdayRange, month: Month, year: int): WeekDay
```

**Функция.** Вычисляют, соответственно, номер дня в году (считая с 0) и
день недели для произвольной даты — без создания полноценного `DateTime`.
Именно эти процедуры используются внутри модуля для автозаполнения полей
`yearday`/`weekday` при построении `DateTime`.

- **Разбор реализации.** `getDayOfWeek` не перебирает дни, а переводит дату
  в epoch day (`toEpochDay`) и берёт остаток от деления на 7, зная, что
  1970-01-01 — четверг: `days = epochDay - 3` сдвигает точку отсчёта на
  ближайший предыдущий понедельник, после чего `floorDiv`/остаток дают
  день недели напрямую, без циклов.

**Пример:**

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

**Функция.** ISO 8601 определяет собственный "недельный" календарь, где
год может состоять из 52 или 53 полных недель, а первые/последние дни
января/декабря иногда принадлежат ISO-неделе соседнего календарного года.
`getWeeksInIsoYear` возвращает 52 или 53 для данного ISO-года,
`getIsoWeekAndYear` — переводит обычную дату в пару (номер ISO-недели,
ISO-год), которая может отличаться от календарного года самой даты (30
декабря 2019 года — это уже неделя 1 ISO-года 2020).

- **Параметры:** `y: IsoYear` — ISO-год (`distinct int`); `dt: DateTime` —
  дата, для которой нужно определить ISO-неделю и ISO-год.

**Пример:**

```nim
doAssert getWeeksInIsoYear(IsoYear(2019)) == 52
doAssert getWeeksInIsoYear(IsoYear(2020)) == 53

let dt = dateTime(2019, mDec, 30, zone = utc())
let (w, y) = getIsoWeekAndYear(dt)
doAssert w == 1.IsoWeekRange
doAssert y == 2020.IsoYear   # ISO-год "перескочил" на следующий
```

---

### `initDateTime` по ISO-неделе

```nim
proc initDateTime*(weekday: WeekDay, isoweek: IsoWeekRange, isoyear: IsoYear,
                   hour: HourRange, minute: MinuteRange, second: SecondRange,
                   nanosecond: NanosecondRange,
                   zone: Timezone = local()): DateTime
proc initDateTime*(weekday: WeekDay, isoweek: IsoWeekRange, isoyear: IsoYear,
                   hour: HourRange, minute: MinuteRange, second: SecondRange,
                   zone: Timezone = local()): DateTime
```

**Функция.** Обратная операция к `getIsoWeekAndYear` — строит `DateTime` по
дню недели и номеру ISO-недели/ISO-года, а не по обычным месяцу/дню.
Удобно, когда исходные данные уже пришли в терминах ISO-календаря
(например, из систем учёта рабочего времени, оперирующих номерами недель).

- **Параметры:** `weekday: WeekDay` — день недели; `isoweek: IsoWeekRange`
  — номер ISO-недели (1..53); `isoyear: IsoYear` — ISO-год;
  `hour`/`minute`/`second`/`nanosecond` — время суток; `zone` — часовой пояс.

**Пример:**

```nim
doAssert initDateTime(dSat, 16, 2018.IsoYear, 00, 00, 00) ==
  dateTime(2018, mApr, 21, 00, 00, 00, zone = local())
```

---
## Прочие процедуры

### `convert`

```nim
proc convert*[T: SomeInteger](unitFrom, unitTo: FixedTimeUnit, quantity: T): T
```

**Функция.** Переводит целое количество одной фиксированной единицы
времени в другую (`Days -> Hours`, `Seconds -> Milliseconds` и т.п.).
Работает только с целыми числами, поэтому при переводе из более мелкой
единицы в более крупную результат усекается, а не округляется.

- **Параметры:** `unitFrom`, `unitTo: FixedTimeUnit` — единицы отправления
  и назначения (диапазон `Nanoseconds..Weeks`); `quantity: T` — величина в
  единице `unitFrom`.

**Пример:**

```nim
doAssert convert(Days, Hours, 2) == 48
doAssert convert(Days, Weeks, 13) == 1       # усечено
doAssert convert(Seconds, Milliseconds, -1) == -1000
```

---

### `epochTime`

```nim
proc epochTime*(): float
```

**Функция.** Текущее время в секундах от эпохи Unix, как `float` —
низкоуровневая обёртка над системным вызовом. Модуль явно рекомендует
предпочитать ей `getTime()`, а для замера длительности — `cpuTime` или
`monotimes.getMonoTime`, поскольку `epochTime`/`getTime` подвержены сдвигу
системных часов (NTP-коррекции и т.п.) и не годятся для бенчмаркинга.

---

### `cpuTime`

```nim
proc cpuTime*(): float
```

**Функция.** Время, потраченное процессором на выполнение текущего
процесса, в секундах (на некоторых ОС фактически измеряет astable время, а
не именно CPU-время). Само по себе значение не несёт смысла — полезна
только разница между двумя вызовами.

- **Параметры:** нет.

**Пример:**

```nim
let t0 = cpuTime()
var fib = @[0, 1, 1]
for i in 1..10:
  add(fib, fib[^1] + fib[^2])
echo "CPU time [s] ", cpuTime() - t0
```

---
## Устаревшие сеттеры полей DateTime

Модуль сохраняет обратную совместимость набором помеченных
`{.deprecated: "Deprecated since v1.3.1".}` процедур-сеттеров —
`` `nanosecond=` ``, `` `second=` ``, `` `minute=` ``, `` `hour=` ``,
`` `monthdayZero=` ``, `` `monthZero=` ``, `` `year=` ``, `` `weekday=` ``,
`` `yearday=` ``, `` `isDst=` ``, `` `timezone=` ``, `` `utcOffset=` ``.
Каждый напрямую присваивает значение соответствующему полю `DateTime`,
минуя проверки согласованности (например, ручная установка `year=` не
пересчитывает `weekday`/`yearday`). В новом коде вместо них следует
использовать `dateTime`/`initDateTime` для создания нового значения
целиком.

---
## Практические рецепты

### 1. Замер длительности операции

```nim
import std/times

let start = getTime()
# ... код, время выполнения которого нужно измерить ...
let elapsed = getTime() - start
echo "Заняло: ", elapsed  # использует `$` (Duration) -> "N seconds" и т.п.
```

---

### 2. Разбор и форматирование дат из внешнего источника

```nim
import std/times

let f = initTimeFormat("yyyy-MM-dd'T'HH:mm:sszzz")
let dt = parse("2024-05-10T09:15:00+02:00", f, utc())
echo format(dt, "dddd, d MMMM yyyy")  # человекочитаемая дата в UTC
```

---

### 3. Человекочитаемый возраст/стаж

```nim
import std/times

let birthDate = dateTime(1990, mJun, 15, zone = utc())
let interval = between(birthDate, now().utc)
echo years(interval), " лет, ", months(interval), " месяцев"
```

*(В примере `years`/`months` обращаются к одноимённым полям `TimeInterval`
через точечную нотацию поля структуры — это разрешённое исключение из
правила о префиксных вызовах, так как это доступ к полю, а не вызов
процедуры.)*

---

### 4. Перевод отметки времени между часовыми поясами

```nim
import std/times

let meetingUtc = dateTime(2024, mDec, 01, 14, 00, 00, zone = utc())
let meetingLocal = inZone(meetingUtc, local())
echo "Встреча по местному времени: ", format(meetingLocal, "HH:mm zzz")
```

---

### 5. Планировщик "раз в N дней"

```nim
import std/times

proc nextRun(lastRun: DateTime, everyDays: int): DateTime =
  lastRun + initDuration(days = everyDays)

var lastRun = dateTime(2024, mJan, 01, 03, 00, 00, zone = utc())
let next = nextRun(lastRun, 7)
doAssert next == dateTime(2024, mJan, 08, 03, 00, 00, zone = utc())
```

---
## Краткая таблица

| Задача | Тип результата | Что использовать |
|---|---|---|
| Текущий момент как "сырое" значение | `Time` | `getTime()` |
| Текущий момент, разобранный на поля | `DateTime` | `now()` |
| Построить дату по году/месяцу/дню | `DateTime` | `dateTime(...)` |
| Точная длительность (секунды/часы) | `Duration` | `initDuration(...)` |
| Календарный интервал ("1 год 2 дня") | `TimeInterval` | `initTimeInterval(...)` или `N.years`/`N.days`/... |
| Разница между двумя `Time` | `Duration` | `t2 - t1` |
| Разница между двумя `DateTime`, в календарных единицах | `TimeInterval` | `between(dt1, dt2)` |
| Перевод в другой часовой пояс | `DateTime` | `inZone(dt, zone)` / `utc(dt)` / `local(dt)` |
| Строка -> дата | `DateTime`/`Time` | `parse(...)` / `parseTime(...)` |
| Дата -> строка по шаблону | `string` | `format(dt, "...")` |
| Дата -> короткая строка без шаблона | `string` | `getDateStr(dt)` / `getClockStr(dt)` |
| Число дней в месяце/году, високосность | `int`/`bool` | `getDaysInMonth`, `getDaysInYear`, `isLeapYear` |
| Номер ISO-недели/ISO-года | `tuple` | `getIsoWeekAndYear(dt)` |
| Замер производительности кода | `float` | `cpuTime()` |

---
## Сводка: какую процедуру выбрать

- Нужна точка во времени без разбиения на поля → используйте `Time` и `getTime`/`fromUnix`.
- Нужны год/месяц/день/час и т.п. по отдельности → используйте `DateTime` и `dateTime`/`now`.
- Нужно прибавить/вычесть фиксированный промежуток (часы, секунды) → используйте `Duration` и `initDuration`.
- Нужно прибавить/вычесть календарный промежуток (месяцы, годы), устойчивый к разной длине месяцев → используйте `TimeInterval` и `initTimeInterval`/`N.years` и т.п.
- Нужна точная разница в секундах между двумя моментами → используйте оператор `-` между `Time` (или между `DateTime`, если пояс один и тот же).
- Нужна разница "по-человечески" (сколько лет, месяцев, дней) → используйте `between`.
- Нужно перевести время в другой пояс → используйте `inZone` (или сокращения `utc`/`local`).
- Нужно разобрать строку с датой → используйте `parse`/`parseTime` с подходящим форматом.
- Нужно вывести дату в конкретном формате → используйте `format`; для стандартного ISO-подобного вида достаточно `$`.
- Нужно узнать число дней в месяце/году или проверить високосность → используйте `getDaysInMonth`/`getDaysInYear`/`isLeapYear`.
- Работаете с ISO-неделями (например, интеграция с системами учёта времени) → используйте `getIsoWeekAndYear` и соответствующую перегрузку `initDateTime`.
- Нужно измерить время выполнения кода → используйте `cpuTime` (не `epochTime`/`now`, они не гарантируют монотонность).
