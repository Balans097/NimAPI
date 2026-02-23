# Модуль Nim `typetraits` — полный справочник

> **Назначение модуля:** Рефлексия типов на этапе компиляции. Каждый экспортируемый символ этого модуля работает исключительно во время компиляции: принимает дескрипторы типов или типизированные выражения и возвращает факты о типах — их имена, арности, базовые типы, обобщённые параметры и возможности. Ничто в этом модуле не порождает накладных расходов в рантайме.
>
> **Импорт:** `import std/typetraits`
>
> **Стабильность API:** Помечен как *нестабильный* — публичный интерфейс может меняться между релизами Nim.
>
> **Ключевая концепция:** Большинство процедур принимают `typedesc` — способ Nim передать тип как значение. Вы пишете `name(int)` или `int.name`, чтобы запросить информацию о самом типе `int`, а не о его экземпляре.

---

## Содержание

1. [Псевдонимы типов для классификации перечислений](#псевдонимы-типов-для-классификации-перечислений)
   - [`HoleyEnum`](#holeyenum)
   - [`OrdinalEnum`](#ordinalenum)
2. [Базовая интроспекция типов](#базовая-интроспекция-типов)
   - [`name`](#name)
   - [`arity`](#arity)
3. [Интроспекция обобщённых типов](#интроспекция-обобщённых-типов)
   - [`genericHead`](#generichead)
   - [`stripGenericParams`](#stripgenericparams)
   - [`genericParams`](#genericparams)
   - [`StaticParam`](#staticparam)
4. [Запросы о возможностях типа](#запросы-о-возможностях-типа)
   - [`supportsCopyMem`](#supportscopymem)
   - [`hasDefaultValue`](#hasdefaultvalue)
5. [Структурные запросы о типах](#структурные-запросы-о-типах)
   - [`isNamedTuple`](#isnamedtuple)
   - [`tupleLen` (тип)](#tuplelen-тип)
   - [`tupleLen` (значение)](#tuplelen-значение)
   - [`get`](#get)
   - [`enumLen`](#enumlen)
   - [`elementType`](#elementtype)
6. [Типы указателей и ссылок](#типы-указателей-и-ссылок)
   - [`pointerBase`](#pointerbase)
7. [Типы-диапазоны](#типы-диапазоны)
   - [`rangeBase` (тип)](#rangebase-тип)
   - [`rangeBase` (значение)](#rangebase-значение)
8. [Типы distinct](#типы-distinct)
   - [`distinctBase` (тип)](#distinctbase-тип)
   - [`distinctBase` (значение)](#distinctbase-значение)
9. [Преобразование знака целых чисел](#преобразование-знака-целых-чисел)
   - [`toUnsigned`](#tounsigned)
   - [`toSigned`](#tosigned)
10. [Инспекция замыканий](#инспекция-замыканий)
    - [`hasClosure`](#hasclosure)

---

## Псевдонимы типов для классификации перечислений

### `HoleyEnum`

```nim
type HoleyEnum* = (not Ordinal) and enum
```

**Класс типов**, соответствующий перечислениям с *дырами* — enum-типам, у которых значения не идут подряд (то есть между числовыми значениями есть пропуски). Перечисление становится «дырявым», когда вы явно задаёте числовые значения с пропусками.

«Дырявые» enum-типы **не являются** `Ordinal`, поскольку на них нельзя безопасно применять `succ`/`pred` без перепрыгивания через неопределённые значения.

```nim
type
  Status = enum
    Pending = 0
    Active  = 10   # пропуск 1..9 — дырявое!
    Closed  = 20

  Priority = enum
    Low, Medium, High   # 0, 1, 2 — сплошное, НЕ дырявое

assert Status   is HoleyEnum
assert Priority is OrdinalEnum
assert Priority isnot HoleyEnum
```

Используйте `HoleyEnum` в ветках `when` или в ограничениях generic, чтобы корректно обрабатывать оба вида перечислений — особенно при сериализации или итерации.

---

### `OrdinalEnum`

```nim
type OrdinalEnum* = Ordinal and enum
```

**Класс типов**, соответствующий перечислениям с *непрерывными* значениями (без дыр). Это обычные `Ordinal`-типы: `ord`, `succ`, `pred`, `low`/`high` работают ожидаемым образом, безопасная итерация возможна.

```nim
type Color = enum Red, Green, Blue   # 0, 1, 2 — непрерывные

assert Color is OrdinalEnum
assert Color isnot HoleyEnum
assert int isnot OrdinalEnum   # int — Ordinal, но не enum
```

---

## Базовая интроспекция типов

### `name`

```nim
proc name*(t: typedesc): string {.magic: "TypeTrait".}
```

**Возвращает имя типа `t` в виде строки.** Для обобщённых типов возвращается полная инстанциированная форма. Это псевдоним `system.$` для `typedesc` (доступен с Nim 0.20), поэтому `name(int)` и `$int` эквивалентны.

Строка — это **каноническое Nim-имя** типа, не обязательно то, что вы объявили: например, псевдонимы типов могут раскрываться.

```nim
import std/typetraits

doAssert name(int)             == "int"
doAssert name(string)          == "string"
doAssert name(seq[string])     == "seq[string]"
doAssert name(array[3, float]) == "array[0..2, float]"

type MyInt = distinct int
doAssert name(MyInt) == "MyInt"
```

**Практическое применение:** Генерация сообщений об ошибках, отладочного вывода или сериализации на основе рефлексии, где имя типа нужно в рантайме.

---

### `arity`

```nim
proc arity*(t: typedesc): int {.magic: "TypeTrait".}
```

**Возвращает количество обобщённых (типовых) параметров** данного типа — его *арность* в математическом смысле. Обычный тип без обобщённых параметров имеет арность 0. Арность кортежа равна числу его полей. Арность обобщённого типа равна числу переданных ему типовых аргументов.

```nim
import std/typetraits

doAssert arity(int)                       == 0  # нет обобщённых параметров
doAssert arity(seq[string])               == 1  # один: тип элемента
doAssert arity(array[3, int])             == 2  # два: тип индекса + тип элемента
doAssert arity((int, int, float, string)) == 4  # четыре поля кортежа
doAssert arity(Table[string, int])        == 2  # ключ + значение
```

**Практическое применение:** Полезно в обобщённых процедурах, которым нужно ветвиться в зависимости от того, сколько типовых параметров несёт данный тип.

---

## Интроспекция обобщённых типов

### `genericHead`

```nim
proc genericHead*(t: typedesc): typedesc {.magic: "TypeTrait".}
```

**Принимает инстанциированный обобщённый тип и возвращает его неинстанциированную (голую) форму.** Для `Foo[int]` возвращает `Foo` — обобщённый тип без каких-либо аргументов. Это *обобщённая голова* (generic head) в смысле теории типов.

**Ошибка компиляции** возникает, если переданный тип не является обобщённым — используйте `stripGenericParams`, если нужна безопасная версия, работающая и с необобщёнными типами.

```nim
import std/typetraits

type Foo[T] = object

doAssert genericHead(Foo[int])         is Foo
doAssert genericHead(Foo[seq[string]]) is Foo

# Использование для написания концепта, требующего обобщённый тип:
type Generic = concept f
  type _ = genericHead(typeof(f))

proc identity(a: Generic): typeof(a) = a
```

**Практическое применение:** Написание концептов или макросов, которым нужно рассуждать о чистом обобщённом контейнере независимо от его типовых аргументов. Например: «является ли это значение каким-либо `seq`, независимо от типа элемента?»

---

### `stripGenericParams`

```nim
proc stripGenericParams*(t: typedesc): typedesc {.magic: "TypeTrait".}
```

**Аналог `genericHead`, но безопасный для необобщённых типов.** Если `t` — инстанциация обобщённого типа, возвращается голый обобщённый тип. Если `t` вообще не является обобщённым — возвращается без изменений. Это «снисходительная» версия `genericHead`.

```nim
import std/typetraits

type Foo[T] = object

doAssert stripGenericParams(Foo[string]) is Foo   # обобщённый → снят
doAssert stripGenericParams(int)         is int   # необобщённый → без изменений
doAssert stripGenericParams(string)      is string
```

**Используйте вместо `genericHead`**, когда входной тип может быть как обобщённым, так и нет, и вы просто хотите голую форму во всех случаях без риска ошибок компиляции.

---

### `genericParams`

```nim
template genericParams*(T: typedesc): untyped
```

*(Доступен с Nim 1.1)*

**Возвращает типовые аргументы обобщённой инстанциации в виде кортежа типов.** Для `Foo[float, string]` возвращает тип кортежа `(float, string)`, который можно инспектировать с помощью `is`, `get` и других операций над типами.

Статические (компайл-тайм) параметры оборачиваются в `StaticParam[value]`, чтобы их можно было получить как значение на уровне типов.

```nim
import std/typetraits

type Foo[T1, T2] = object

doAssert genericParams(Foo[float, string]) is (float, string)
doAssert genericParams(seq[int]).get(0) is int

type Bar[N: static float, T] = object
doAssert genericParams(Bar[1.0, string]) is (StaticParam[1.0], string)
doAssert genericParams(Bar[1.0, string]).get(0).value == 1.0

# Вложенный:
doAssert genericParams(seq[Bar[2.0, string]]).get(0) is Bar[2.0, string]
```

> **Оговорка про array:** Для встроенного типа `array` параметр размера всегда нормализуется в `range` после проверки типов. Поэтому `genericParams(array[10, int])` даёт `(StaticParam[10], int)` в типовом выражении, но `genericParams(typeof(a))` для переменной `a: array[10, int]` даёт `(range[0..9], int)`.

---

### `StaticParam`

```nim
type StaticParam*[value: static type] = object
```

Вспомогательный тип, используемый `genericParams` для оборачивания **статических (компайл-тайм) обобщённых параметров** — чтобы их можно было представить в кортеже типов. `StaticParam` вы не создаёте вручную — вы получаете его как часть кортежа, возвращённого `genericParams`, и читаете его поле `.value`.

```nim
import std/typetraits

type Arr = array[5, int]
# Первый параметр — StaticParam[5]:
doAssert genericParams(Arr).get(0) is StaticParam[5]
```

---

## Запросы о возможностях типа

### `supportsCopyMem`

```nim
proc supportsCopyMem*(t: typedesc): bool {.magic: "TypeTrait".}
```

**Возвращает `true`, если значения типа `t` можно безопасно дублировать копированием сырых байт** (то есть через `copyMem`). Такие типы называются *plain old data* (POD) или *blob* в других языках. Они не содержат управляемых полей (не имеют `string`, `seq`, `ref`, указателей под GC и т.п.) и не имеют пользовательских деструкторов.

```nim
import std/typetraits

doAssert supportsCopyMem(int)      == true
doAssert supportsCopyMem(float)    == true
doAssert supportsCopyMem(char)     == true
doAssert supportsCopyMem(string)   == false  # внутри ref-подсчёт
doAssert supportsCopyMem(seq[int]) == false  # выделение на куче

type Point = object
  x, y: float
doAssert supportsCopyMem(Point) == true  # чистая структура, нет управляемых полей
```

**Практическое применение:** Низкоуровневая сериализация, запись в memory-mapped файлы, zero-copy сетевые буферы, передача данных в C-библиотеки — любой контекст, где копирование сырых байт должно быть корректным.

---

### `hasDefaultValue`

```nim
proc hasDefaultValue*(t: typedesc): bool {.magic: "TypeTrait".}
```

**Возвращает `true`, если тип `t` можно инициализировать по умолчанию** (то есть `default(t)` или `var x: t` допустимы без явного начального значения). Возвращает `false` для:
- `var T` (изменяемая ссылка — требует явной привязки)
- ref-типов с ограничением `not nil` (nil не является допустимым значением по умолчанию)
- Типов с полями, помеченными `{.requiresInit.}`

```nim
import std/typetraits

{.experimental: "strictNotNil".}
type
  NilableObject = ref object
    a: int
  Object = NilableObject not nil   # не может быть nil → нет умолчания

assert hasDefaultValue(int)           == true
assert hasDefaultValue(string)        == true
assert hasDefaultValue(NilableObject) == true   # nil — допустимое умолчание
assert hasDefaultValue(Object)        == false  # not nil → нет nil-умолчания
assert hasDefaultValue(var string)    == false

type RequiresInit[T] = object
  a {.requiresInit.}: T
assert hasDefaultValue(RequiresInit[int]) == false
```

**Практическое применение:** Обобщённый код, условно инициализирующий значение нулём, или обобщённые контейнеры, которым нужно знать, можно ли безопасно создать «пустой» элемент.

---

## Структурные запросы о типах

### `isNamedTuple`

```nim
proc isNamedTuple*(T: typedesc): bool {.magic: "TypeTrait".}
```

**Возвращает `true` только для именованных кортежей** — кортежей, где у каждого поля есть явное имя (объявленных через `tuple[имя: Тип, …]`). Возвращает `false` для анонимных кортежей `(Тип, Тип, …)` и для всех типов, не являющихся кортежами.

```nim
import std/typetraits

doAssert not isNamedTuple(int)
doAssert not isNamedTuple((string, int))               # анонимный кортеж
doAssert     isNamedTuple(tuple[name: string, age: int])  # именованный

type Person = tuple[name: string, age: int]
doAssert isNamedTuple(Person)
```

**Практическое применение:** Обобщённые сериализаторы, которые могут выводить имена полей для именованных кортежей, но вынуждены использовать позиционный доступ для анонимных.

---

### `tupleLen` (тип)

```nim
proc tupleLen*(T: typedesc[tuple]): int {.magic: "TypeTrait".}
```

*(Доступен с Nim 1.1)*

**Возвращает количество полей** в типе кортежа `T`. Работает как для именованных, так и для анонимных кортежей. Принимает только типы кортежей — передача другого типа вызывает ошибку компиляции.

```nim
import std/typetraits

doAssert tupleLen((int, int, float, string))       == 4
doAssert tupleLen(tuple[name: string, age: int])   == 2
doAssert tupleLen(tuple[])                         == 0
```

---

### `tupleLen` (значение)

```nim
template tupleLen*(t: tuple): int
```

*(Доступен с Nim 1.1)*

**Перегрузка `tupleLen`, принимающая значение кортежа** вместо типа. Просто делегирует к `tupleLen(typeof(t))`. Удобна, когда значение уже есть под рукой и не хочется писать `tupleLen(typeof(myTuple))`.

```nim
import std/typetraits

let p = (1, "hello")
doAssert tupleLen(p) == 2

let person = (name: "Алиса", age: 30)
doAssert tupleLen(person) == 2
```

---

### `get`

```nim
template get*(T: typedesc[tuple], i: static int): untyped
```

*(Доступен с Nim 1.1)*

**Возвращает тип `i`-го поля** (нумерация с нуля) типа кортежа `T`. Это операция над типами — возвращает *тип*, а не значение. Используется совместно с `genericParams` для инспекции отдельных обобщённых аргументов по позиции.

```nim
import std/typetraits

doAssert get((int, float, string), 0) is int
doAssert get((int, float, string), 1) is float
doAssert get((int, float, string), 2) is string

# Наиболее полезно с genericParams:
type Pair[A, B] = object
doAssert genericParams(Pair[int, string]).get(0) is int
doAssert genericParams(Pair[int, string]).get(1) is string
```

---

### `enumLen`

```nim
macro enumLen*(T: typedesc[enum]): int
```

**Возвращает количество вариантов** в перечислении `T` в виде целого числа времени компиляции. Работает как для порядковых, так и для дырявых enum. Результат — `static int`, поэтому его можно использовать в размерах массивов, статических утверждениях и других контекстах времени компиляции.

```nim
import std/typetraits

type Direction = enum North, South, East, West

doAssert Direction.enumLen == 4

type Status = enum
  Pending = 0, Active = 10, Closed = 20   # дырявый enum

doAssert Status.enumLen == 3   # 3 варианта, независимо от пропусков

# Используем счётчик для размера таблицы поиска:
var names: array[Direction.enumLen, string]
```

---

### `elementType`

```nim
template elementType*(a: untyped): typedesc
```

*(Доступен с Nim 1.3.5)*

**Возвращает тип элемента любого итерируемого `a`** — всего, по чему можно пройти циклом `for`: последовательности, массивы, строки, пользовательские итераторы, диапазоны, множества и многое другое. Это чисто компайл-тайм операция; `a` никогда реально не итерируется.

```nim
import std/typetraits

doAssert elementType(@[1, 2, 3])    is int
doAssert elementType("hello")       is char
doAssert elementType({1, 2, 3})     is int   # set

iterator countdown3: int =
  yield 3; yield 2; yield 1

doAssert elementType(countdown3()) is int

# Полезно в обобщённых процедурах:
proc firstElement[C](c: C): elementType(c) =
  for x in c: return x
```

**Практическое применение:** Написание обобщённых алгоритмов, которым нужно знать тип элемента контейнера или итератора без требования явного типового параметра.

---

## Типы указателей и ссылок

### `pointerBase`

```nim
template pointerBase*[T](_: typedesc[ptr T | ref T]): typedesc
```

**Возвращает тип, на который указывает `T`** для типов `ptr T` и `ref T` — один уровень косвенности снимается. **Не рекурсивен**: `(ref ptr int).pointerBase` даёт `ptr int`, а не `int`.

```nim
import std/typetraits

assert (ref int).pointerBase   is int
assert (ptr float).pointerBase is float

type A = ptr seq[float]
assert A.pointerBase is seq[float]

assert (ref A).pointerBase is A              # только один уровень: A = ptr seq[float]
assert (ref A).pointerBase isnot seq[float]

# Работает и с адресами переменных:
var s = "hello"
assert s[0].addr.typeof.pointerBase is char
```

**Практическое применение:** Обобщённый код над типами указателей, которому нужно работать с разыменованным типом или объявлять его — например, аллокаторы, обёртки или FFI-биндинги.

---

## Типы-диапазоны

### `rangeBase` (тип)

```nim
proc rangeBase*(T: typedesc[range]): typedesc {.magic: "TypeTrait".}
```

**Возвращает базовый тип для типа-диапазона.** Тип `range[lo..hi]` ограничивает значения подмножеством другого типа; `rangeBase` снимает это ограничение и возвращает сырой тип под ним.

Принимает только типы-диапазоны — передача другого типа вызывает ошибку компиляции.

```nim
import std/typetraits

type MyRange    = range[0..5]
type MyEnum     = enum a, b, c
type EnumRange  = range[b..c]

doAssert rangeBase(MyRange)            is int     # range от int → int
doAssert rangeBase(EnumRange)          is MyEnum  # range от MyEnum → MyEnum
doAssert rangeBase(range['a'..'z'])    is char
```

---

### `rangeBase` (значение)

```nim
template rangeBase*[T: range](a: T): untyped
```

**Перегрузка `rangeBase` для значений.** Преобразует значение типа-диапазона в значение базового типа. Эквивалентно `rangeBase(typeof(a))(a)` — приведение типов на уровне типов, снимающее ограничение диапазона.

```nim
import std/typetraits

type MyRange = range[0..5]
let x = MyRange(3)
doAssert rangeBase(x) is int
doAssert rangeBase(x) == 3          # то же числовое значение, базовый тип

type MyEnum = enum a, b, c
type EnumRange = range[b..c]
let e: EnumRange = c
doAssert rangeBase(e) is MyEnum
doAssert rangeBase(e) == c
```

**Практическое применение:** Преобразование значения типа-диапазона в базовое для арифметики, сравнения с переменными обычного типа или передачи в процедуры, не принимающие тип-диапазон.

---

## Типы distinct

### `distinctBase` (тип)

```nim
proc distinctBase*(T: typedesc, recursive: static bool = true): typedesc {.magic: "TypeTrait".}
```

**Возвращает тип под одним или несколькими слоями `distinct`.** По умолчанию (`recursive = true`) рекурсивно снимает все слои до достижения типа, не являющегося `distinct`. С `recursive = false` снимается ровно один слой.

Для типов, не являющихся `distinct`, тип возвращается без изменений.

```nim
import std/typetraits

type MyInt      = distinct int
type MyOtherInt = distinct MyInt   # два слоя вглубь

doAssert distinctBase(MyInt)              is int    # один слой → int
doAssert distinctBase(MyOtherInt)         is int    # два слоя → int (рекурсивно)
doAssert distinctBase(MyOtherInt, false)  is MyInt  # только один слой
doAssert distinctBase(int)                is int    # не distinct → без изменений
```

**Практическое применение:** Обобщённые утилиты (парсеры, сериализаторы, форматтеры), получающие тип `distinct`, но работающие с лежащим в основе целым числом, строкой и т.д.

---

### `distinctBase` (значение)

```nim
template distinctBase*[T](a: T, recursive: static bool = true): untyped
```

*(Доступен с Nim 1.1)*

**Перегрузка `distinctBase` для значений.** Снимает обёртку `distinct` с значения и возвращает его с базовым типом. Если `T` не является `distinct`, значение возвращается без изменений (без предупреждения о само-преобразовании).

```nim
import std/typetraits

type MyInt      = distinct int
type MyOtherInt = distinct MyInt

doAssert 12.MyInt.distinctBase      == 12           # значение 12, тип int
doAssert 12.MyOtherInt.distinctBase == 12           # снимает оба слоя
doAssert 12.MyOtherInt.distinctBase(false) is MyInt # один слой
doAssert 12.distinctBase            == 12           # не distinct → без изменений
```

---

## Преобразование знака целых чисел

### `toUnsigned`

```nim
template toUnsigned*(T: typedesc[SomeInteger and not range]): untyped
```

**Возвращает беззнаковый целочисленный тип с той же разрядностью, что и `T`.** Если `T` уже беззнаковый — возвращается без изменений. Типы-диапазоны не поддерживаются и вызывают ошибку компиляции.

| Входной тип | Результат |
|---|---|
| `int8` | `uint8` |
| `int16` | `uint16` |
| `int32` | `uint32` |
| `int64` | `uint64` |
| `int` | `uint` |
| `uint`, `uint8`, … | без изменений |

```nim
import std/typetraits

assert int8.toUnsigned  is uint8
assert int16.toUnsigned is uint16
assert int.toUnsigned   is uint
assert uint.toUnsigned  is uint    # уже беззнаковый — без изменений

# Для типов-диапазонов не работает:
assert not compiles(toUnsigned(range[0..7]))
```

**Практическое применение:** Обобщённый код, которому нужно интерпретировать целое число как беззнаковое без изменения разрядности — например, утилиты для работы с битами или низкоуровневое сетевое кодирование.

---

### `toSigned`

```nim
template toSigned*(T: typedesc[SomeInteger and not range]): untyped
```

**Возвращает знаковый целочисленный тип с той же разрядностью, что и `T`.** Если `T` уже знаковый — возвращается без изменений. Типы-диапазоны не поддерживаются.

| Входной тип | Результат |
|---|---|
| `uint8` | `int8` |
| `uint16` | `int16` |
| `uint32` | `int32` |
| `uint64` | `int64` |
| `uint` | `int` |
| `int`, `int8`, … | без изменений |

```nim
import std/typetraits

assert uint8.toSigned  is int8
assert uint16.toSigned is int16
assert uint.toSigned   is int
assert int8.toSigned   is int8    # уже знаковый — без изменений

assert not compiles(toSigned(range[0..7]))
```

---

## Инспекция замыканий

### `hasClosure`

```nim
proc hasClosure*(fn: NimNode): bool {.since: (1, 5, 1).}
```

*(Доступен с Nim 1.5.1)*

**Возвращает `true`, если процедура, представленная `NimNode` `fn`, имеет среду замыкания** — то есть захватывает переменные из охватывающей области видимости. Эта процедура предназначена для вызова из **макросов**, принимающих `typed`-аргументы, поскольку `fn` должен быть полностью разрешённым узлом `nnkSym` (а не нетипизированным фрагментом AST).

```nim
import std/typetraits, std/macros

macro checkClosure(fn: typed): bool =
  newLit(hasClosure(fn))

proc pureProc(x: int): int = x + 1

var captured = 10
let closureProc = proc(x: int): int = x + captured

assert not checkClosure(pureProc)      # нет захватов → нет замыкания
assert     checkClosure(closureProc)   # захватывает `captured` → есть замыкание
```

**Практическое применение:** Кодогенераторы и макросы, которым нужно решить, требует ли вызываемый объект скрытый указатель на среду — актуально при генерации C FFI-колбэков, взаимодействии с C-библиотеками, ожидающими сырые указатели на функции, или оптимизации диспетчеризации.

---

## Полный рабочий пример

```nim
import std/typetraits

# --- Базовая интроспекция ---
doAssert name(seq[int])             == "seq[int]"
doAssert arity(Table[string, int])  == 2

# --- Классификация перечислений ---
type
  Sparse = enum s0 = 0, s1 = 100, s2 = 200  # дырявое
  Dense  = enum d0, d1, d2                   # порядковое

assert Sparse is HoleyEnum
assert Dense  is OrdinalEnum
assert Dense.enumLen == 3

# --- Интроспекция обобщённых типов ---
type Box[T] = object
  value: T

doAssert genericHead(Box[string]) is Box
doAssert genericParams(Box[float]).get(0) is float

# --- Запросы о возможностях ---
type RawPoint = object
  x, y: float32

doAssert supportsCopyMem(RawPoint)       # безопасно memcpy
doAssert not supportsCopyMem(seq[int])   # выделение на куче

# --- Утилиты типов distinct ---
type Meters  = distinct float
type Seconds = distinct float

proc speed(d: Meters, t: Seconds): float =
  distinctBase(d) / distinctBase(t)

doAssert speed(Meters(100.0), Seconds(10.0)) == 10.0

# --- Преобразование знака ---
type Storage  = int32
type UStorage = Storage.toUnsigned   # uint32

# --- Базы диапазонов и указателей ---
type SmallInt = range[0..15]
doAssert rangeBase(SmallInt) is int

doAssert (ref string).pointerBase is string

# --- Структурные запросы о кортежах ---
type Record = tuple[name: string, score: int]

doAssert isNamedTuple(Record)
doAssert tupleLen(Record) == 2
doAssert get(Record, 0)   is string
doAssert get(Record, 1)   is int

# --- Тип элемента ---
doAssert elementType(@[1.0, 2.0]) is float
```

---

*Справочник охватывает модуль `std/typetraits` стандартной библиотеки Nim согласно исходному коду модуля.*
