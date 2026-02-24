# Справочник модуля `packedsets`

> **Назначение модуля:** `packedsets` реализует эффективное с точки зрения памяти и производительности множество для любого типа `Ordinal` — целых чисел, перечислений, символов и любого типа, значения которого отображаются в непрерывный целочисленный диапазон. Под капотом используется **разреженное битовое множество** (sparse bit set): принадлежность каждого элемента кодируется одним битом, поэтому при кластеризованных значениях множество занимает ничтожную долю памяти по сравнению с хеш-множеством, а проверки принадлежности предельно быстры (одна операция проверки бита после двух вычислений индексов). Для множеств общего назначения над произвольными типами используйте `std/sets`.

---

## Содержание

1. [Тип: PackedSet](#тип-packedset)
2. [Внутренняя архитектура вкратце](#внутренняя-архитектура-вкратце)
3. [Создание](#создание)
   - [initPackedSet](#initpackedset)
   - [toPackedSet](#topackedset)
4. [Принадлежность](#принадлежность)
   - [contains / in / notin](#contains--in--notin)
5. [Мутация — отдельные элементы](#мутация--отдельные-элементы)
   - [incl (элемент)](#incl-элемент)
   - [excl (элемент)](#excl-элемент)
   - [containsOrIncl](#containsorincl)
   - [missingOrExcl](#missingorexcl)
6. [Мутация — целые множества](#мутация--целые-множества)
   - [incl (множество)](#incl-множество)
   - [excl (множество)](#excl-множество)
   - [clear](#clear)
7. [Запросы](#запросы)
   - [len / card](#len--card)
   - [isNil](#isnil)
8. [Итерация](#итерация)
   - [items](#items)
9. [Алгебра множеств — именованные процедуры](#алгебра-множеств--именованные-процедуры)
   - [union](#union)
   - [intersection](#intersection)
   - [difference](#difference)
   - [symmetricDifference](#symmetricdifference)
   - [disjoint](#disjoint)
10. [Алгебра множеств — операторы](#алгебра-множеств--операторы)
    - [+ (объединение)](#-объединение)
    - [* (пересечение)](#-пересечение)
    - [- (разность)](#--разность)
11. [Операторы сравнения](#операторы-сравнения)
    - [== (равенство)](#-равенство)
    - [<= (подмножество)](#-подмножество)
    - [< (строгое подмножество)](#-строгое-подмножество)
12. [Копирование и преобразование](#копирование-и-преобразование)
    - [=copy](#copy)
    - [assign (устаревший)](#assign-устаревший)
    - [$ (преобразование в строку)](#-преобразование-в-строку)
13. [PackedSet vs sets — когда что выбирать](#packedset-vs-sets--когда-что-выбирать)
14. [Практические паттерны](#практические-паттерны)

---

## Тип: `PackedSet`

```nim
type PackedSet*[A: Ordinal] = object
```

Обобщённое множество-значение (value type), чей тип элементов `A` обязан быть `Ordinal`. Множество хранится как объект (не как ref), поэтому имеет семантику значения — присваивание копирует множество. Оператор копирования `=copy` явно определён для выполнения глубокого копирования внутренней структуры битовых стволов.

Параметр типа ограничивает вас типами, имеющими осмысленное целочисленное представление: `int`, `int8`–`int64`, `uint`, `uint8`–`uint64`, `char`, `bool` и любой `enum`. Нельзя хранить `string`, `float` или пользовательские объекты.

---

## Внутренняя архитектура вкратце

`PackedSet` использует две стратегии хранения в зависимости от количества элементов:

**Малый режим (≤ 34 элементов):** элементы хранятся в плоском `array[0..33, int]` непосредственно внутри объекта. Выделения памяти в куче не происходит. Это делает крошечные множества очень дружественными к кешу.

**Большой режим (> 34 элементов):** множество переходит на **разреженное битовое множество**, поддерживаемое хеш-таблицей *стволов* (trunks). Каждый ствол охватывает 512 последовательных порядковых значений и хранит их принадлежность в виде 512-битного массива (64 байта `uint`). Выделяются только те стволы, которые содержат хотя бы один элемент, поэтому множество `{1, 1_000_000}` использует всего два ствола — а не миллион бит непрерывной памяти.

В результате — O(1) проверки принадлежности и очень низкое потребление памяти как для плотных диапазонов, так и для разреженных больших множеств.

---

## Создание

### `initPackedSet`

```nim
proc initPackedSet*[A]: PackedSet[A]
```

Возвращает новое пустое `PackedSet[A]`. Это канонический способ создать множество, когда вы планируете заполнять его инкрементально с помощью `incl`. Множество начинает в малом режиме без выделения памяти в куче.

`A` должен быть указан явно — либо на месте вызова, либо выведен из окружающего контекста.

```nim
import std/packedsets

# Явный параметр типа
let empty = initPackedSet[int]()
assert empty.len == 0

# Работает и для перечислений
type Color = enum Red, Green, Blue
var colors = initPackedSet[Color]()
colors.incl(Red)
colors.incl(Blue)
assert colors.len == 2

# Работает для distinct-типов
type UserId = distinct int
var ids = initPackedSet[UserId]()
ids.incl(42.UserId)
```

---

### `toPackedSet`

```nim
proc toPackedSet*[A](x: openArray[A]): PackedSet[A]
```

Создаёт `PackedSet[A]` из любого массива или последовательности порядковых значений. Дубликаты во входных данных автоматически отбрасываются — результирующее множество содержит каждое значение не более одного раза. Это наиболее краткий способ создать множество с известным начальным содержимым.

```nim
import std/packedsets

# Дубликаты тихо убираются
let a = [5, 6, 7, 8, 8].toPackedSet
assert a.len == 4
assert $a == "{5, 6, 7, 8}"

# Работает с последовательностями
let primes = @[2, 3, 5, 7, 11, 13].toPackedSet
assert 7 in primes
assert 9 notin primes

# Работает со значениями перечислений
type Dir = enum North, South, East, West
let horizontal = [East, West].toPackedSet
assert East in horizontal
assert North notin horizontal
```

---

## Принадлежность

### `contains` / `in` / `notin`

```nim
proc contains*[A](s: PackedSet[A], key: A): bool
```

Возвращает `true`, если `key` является членом `s`. Операторы `in` и `notin` — это синтаксический сахар Nim для `contains` и `not contains` соответственно, и работают автоматически, поскольку `contains` определён.

Проверка принадлежности — O(1): в малом режиме просматривается до 34 элементов; в большом режиме выполняются два вычисления индекса и одна проверка бита.

```nim
import std/packedsets

let a = [1, 3, 5].toPackedSet

# Все три формы эквивалентны
assert a.contains(3)
assert 3 in a
assert 5 notin a == false

# Удобно в условных выражениях
if 3 in a:
  echo "3 есть в множестве"

# Пример с перечислением
type ABCD = enum A, B, C, D
let letters = [A, C].toPackedSet
assert A in letters
assert B notin letters
```

---

## Мутация — отдельные элементы

### `incl` (элемент)

```nim
proc incl*[A](s: var PackedSet[A], key: A)
```

Добавляет `key` в `s`. Если `key` уже присутствует, вызов является холостым — множество не изменяется и ошибка не возникает. После вызова гарантируется, что `key in s` равно `true`.

```nim
import std/packedsets

var a = initPackedSet[int]()
a.incl(3)
a.incl(3)   # второй вызов не имеет эффекта
a.incl(7)
assert a.len == 2
assert 3 in a
assert 7 in a
```

---

### `excl` (элемент)

```nim
proc excl*[A](s: var PackedSet[A], key: A)
```

Удаляет `key` из `s`. Если `key` отсутствует, вызов является холостым — ошибка не возникает. После вызова гарантируется, что `key in s` равно `false`.

```nim
import std/packedsets

var a = [3, 7].toPackedSet
a.excl(3)
a.excl(3)   # второй вызов не имеет эффекта
a.excl(99)  # элемент не в множестве — ошибки нет
assert a.len == 1
assert 3 notin a
assert 7 in a
```

---

### `containsOrIncl`

```nim
proc containsOrIncl*[A](s: var PackedSet[A], key: A): bool
```

Совмещённая операция проверки и вставки. Добавляет `key` в `s`, если его там не было, и возвращает булево значение, показывающее, присутствовал ли `key` **до** этого вызова:

- Возвращает `false` → `key` **не было** в `s`; теперь он добавлен.
- Возвращает `true`  → `key` **уже был** в `s`; множество не изменено.

Это эффективнее, чем отдельный `contains` с последующим `incl`, потому что позволяет избежать двукратного обхода внутренней структуры.

```nim
import std/packedsets

var a = initPackedSet[int]()

let wasPresent1 = a.containsOrIncl(3)
assert wasPresent1 == false   # 3 отсутствовал, теперь добавлен
assert 3 in a

let wasPresent2 = a.containsOrIncl(3)
assert wasPresent2 == true    # 3 уже был, ничего не изменилось

let wasPresent3 = a.containsOrIncl(4)
assert wasPresent3 == false   # 4 отсутствовал, теперь добавлен
assert a.len == 2
```

---

### `missingOrExcl`

```nim
proc missingOrExcl*[A](s: var PackedSet[A], key: A): bool
```

Совмещённая операция проверки и удаления — зеркальное отражение `containsOrIncl`. Удаляет `key` из `s`, если он там был, и возвращает булево значение, показывающее, **отсутствовал** ли `key` до этого вызова:

- Возвращает `false` → `key` **был** в `s`; теперь он удалён.
- Возвращает `true`  → `key` **не было** в `s`; множество не изменено.

```nim
import std/packedsets

var a = [5].toPackedSet

let wasMissing1 = a.missingOrExcl(5)
assert wasMissing1 == false   # 5 был, теперь удалён
assert 5 notin a

let wasMissing2 = a.missingOrExcl(5)
assert wasMissing2 == true    # 5 уже отсутствовал
```

---

## Мутация — целые множества

### `incl` (множество)

```nim
proc incl*[A](s: var PackedSet[A], other: PackedSet[A])
```

Добавляет все элементы из `other` в `s` на месте. Элементы, уже присутствующие в `s`, не затрагиваются. После вызова `s` является объединением исходного `s` и `other`. Это мутирующий аналог оператора `+`.

```nim
import std/packedsets

var a = [1, 2].toPackedSet
let b = [2, 3, 4].toPackedSet
a.incl(b)
assert a == [1, 2, 3, 4].toPackedSet
```

---

### `excl` (множество)

```nim
proc excl*[A](s: var PackedSet[A], other: PackedSet[A])
```

Удаляет все элементы `other` из `s` на месте. Элементы `other`, которых нет в `s`, игнорируются. После вызова `s` содержит только те элементы, которые были в `s`, но не в `other`. Это мутирующий аналог оператора `-`.

```nim
import std/packedsets

var a = [1, 2, 3, 4].toPackedSet
let b = [2, 4].toPackedSet
a.excl(b)
assert a == [1, 3].toPackedSet
```

---

### `clear`

```nim
proc clear*[A](result: var PackedSet[A])
```

Сбрасывает `s` в пустое состояние, освобождая все внутренние выделения стволов. После этого вызова множество ведёт себя в точности как свежесозданное с помощью `initPackedSet`. Эффективнее, чем присвоение нового пустого множества, если нужно повторно использовать переменную.

```nim
import std/packedsets

var a = [5, 7, 100, 200].toPackedSet
assert a.len == 4
a.clear()
assert a.len == 0
assert a.isNil
```

---

## Запросы

### `len` / `card`

```nim
proc len*[A](s: PackedSet[A]): int
proc card*[A](s: PackedSet[A]): int   # псевдоним для len
```

Возвращает количество элементов в `s`. `card` — математический синоним (от *мощности* множества), который некоторые предпочитают при мышлении в терминах теории множеств.

В малом режиме `len` работает за O(1). В большом режиме он итерирует по всем установленным битам, поэтому имеет сложность O(n), где n — количество элементов. Для проверки пустоты `isNil` работает быстрее.

```nim
import std/packedsets

let a = [1, 3, 5].toPackedSet
assert len(a) == 3
assert card(a) == 3   # то же самое

let empty = initPackedSet[int]()
assert len(empty) == 0
```

---

### `isNil`

```nim
proc isNil*[A](x: PackedSet[A]): bool
```

Возвращает `true`, если множество пусто (не содержит элементов), `false` в противном случае. Несмотря на имя, заимствованное из семантики указателей, `PackedSet` — это тип-значение; `isNil` просто означает «пустой». Проверка выполняется за O(1) и быстрее, чем `len(s) == 0` для больших множеств.

```nim
import std/packedsets

var a = initPackedSet[int]()
assert a.isNil          # только что создан — пуст

a.incl(42)
assert not a.isNil      # есть элемент

a.excl(42)
assert a.isNil          # снова пуст
```

---

## Итерация

### `items`

```nim
iterator items*[A](s: PackedSet[A]): A
```

Порождает каждый элемент `s` ровно один раз. Порядок итерации — **возрастание по порядковому значению**: элементы всегда выдаются в порядке возрастания, независимо от порядка их вставки.

**Важно:** не изменяйте множество (не вызывайте `incl` или `excl`) во время итерации по нему. Внутренняя структура может стать некорректной при параллельных изменениях, что приведёт к пропуску элементов или неверным результатам.

```nim
import std/packedsets

let a = [5, 1, 3, 2, 4].toPackedSet

# Элементы всегда выходят отсортированными по возрастанию
for x in a:
  echo x   # печатает 1, 2, 3, 4, 5

# Собрать в последовательность
var sorted: seq[int]
for x in a:
  sorted.add(x)
assert sorted == @[1, 2, 3, 4, 5]

# Использование в выражении for-in
import std/sequtils
assert toSeq(a.items) == @[1, 2, 3, 4, 5]
```

---

## Алгебра множеств — именованные процедуры

### `union`

```nim
proc union*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

Возвращает новое множество, содержащее каждый элемент, который есть в `s1`, `s2` или в обоих. Ни `s1`, ни `s2` не изменяются. Эквивалентно оператору `+`.

```nim
import std/packedsets

let a = [1, 2, 3].toPackedSet
let b = [3, 4, 5].toPackedSet
let c = union(a, b)
assert c == [1, 2, 3, 4, 5].toPackedSet
```

---

### `intersection`

```nim
proc intersection*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

Возвращает новое множество, содержащее только элементы, присутствующие **в обоих** `s1` и `s2`. Эквивалентно оператору `*`.

```nim
import std/packedsets

let a = [1, 2, 3].toPackedSet
let b = [3, 4, 5].toPackedSet
let c = intersection(a, b)
assert c == [3].toPackedSet
```

---

### `difference`

```nim
proc difference*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

Возвращает новое множество, содержащее элементы, которые есть в `s1`, но **отсутствуют** в `s2`. Порядок операндов важен: `difference(a, b) ≠ difference(b, a)`. Эквивалентно оператору `-`.

```nim
import std/packedsets

let a = [1, 2, 3].toPackedSet
let b = [3, 4, 5].toPackedSet
assert difference(a, b) == [1, 2].toPackedSet
assert difference(b, a) == [4, 5].toPackedSet
```

---

### `symmetricDifference`

```nim
proc symmetricDifference*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

Возвращает новое множество, содержащее элементы, которые есть ровно в **одном** из `s1` или `s2` — всё из объединения, чего нет в пересечении. Операция коммутативна: `symmetricDifference(a, b) == symmetricDifference(b, a)`.

```nim
import std/packedsets

let a = [1, 2, 3].toPackedSet
let b = [3, 4, 5].toPackedSet
let c = symmetricDifference(a, b)
assert c == [1, 2, 4, 5].toPackedSet   # 3 есть в обоих, поэтому исключён
```

---

### `disjoint`

```nim
proc disjoint*[A](s1, s2: PackedSet[A]): bool
```

Возвращает `true`, если `s1` и `s2` не имеют общих элементов — их пересечение пусто. Возвращает `false` как только найден общий элемент, что делает функцию эффективной для проверок с ранним выходом.

```nim
import std/packedsets

let a = [1, 2].toPackedSet
let b = [2, 3].toPackedSet
let c = [3, 4].toPackedSet

assert disjoint(a, c) == true    # нет общих элементов
assert disjoint(a, b) == false   # 2 является общим
```

---

## Алгебра множеств — операторы

Операторы — это тонкие встраиваемые (`inline`) псевдонимы для именованных процедур. Они существуют исключительно для удобочитаемости при написании выражений над множествами.

### `+` (объединение)

```nim
proc `+`*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

```nim
import std/packedsets
let result = [1, 2].toPackedSet + [2, 3].toPackedSet
assert result == [1, 2, 3].toPackedSet
```

---

### `*` (пересечение)

```nim
proc `*`*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

```nim
import std/packedsets
let result = [1, 2, 3].toPackedSet * [2, 3, 4].toPackedSet
assert result == [2, 3].toPackedSet
```

---

### `-` (разность)

```nim
proc `-`*[A](s1, s2: PackedSet[A]): PackedSet[A]
```

```nim
import std/packedsets
let result = [1, 2, 3].toPackedSet - [2].toPackedSet
assert result == [1, 3].toPackedSet
```

---

## Операторы сравнения

### `==` (равенство)

```nim
proc `==`*[A](s1, s2: PackedSet[A]): bool
```

Возвращает `true`, если оба множества содержат ровно одинаковые элементы. Порядок вставки значения не имеет. Реализован через взаимную проверку подмножества (`s1 <= s2 and s2 <= s1`), что позволяет избежать необходимости сортировать или хешировать.

```nim
import std/packedsets

assert [1, 2].toPackedSet == [2, 1].toPackedSet         # порядок вставки неважен
assert [1, 2].toPackedSet == [1, 2, 1].toPackedSet      # дубликаты в источнике неважны
assert [1, 2].toPackedSet != [1, 2, 3].toPackedSet
```

---

### `<=` (подмножество)

```nim
proc `<=`*[A](s1, s2: PackedSet[A]): bool
```

Возвращает `true`, если `s1` является **подмножеством** `s2`: каждый элемент `s1` есть и в `s2`. Равное множество считается подмножеством самого себя (рефлексивность), поэтому `s <= s` всегда истинно.

```nim
import std/packedsets

let a = [1].toPackedSet
let b = [1, 2].toPackedSet

assert a <= b        # a ⊆ b
assert b <= b        # множество является подмножеством самого себя
assert not (b <= a)
```

---

### `<` (строгое подмножество)

```nim
proc `<`*[A](s1, s2: PackedSet[A]): bool
```

Возвращает `true`, если `s1` является **строгим (собственным) подмножеством** `s2`: каждый элемент `s1` есть в `s2`, и в `s2` есть хотя бы один элемент, которого нет в `s1`. Множество никогда не является строгим подмножеством самого себя.

```nim
import std/packedsets

let a = [1].toPackedSet
let b = [1, 2].toPackedSet

assert a < b         # a — строгое подмножество b
assert not (b < b)   # множество не является строгим подмножеством себя
assert not (b < a)
```

---

## Копирование и преобразование

### `=copy`

```nim
proc `=copy`*[A](dest: var PackedSet[A], src: PackedSet[A])
```

Хук копирования, который Nim вызывает автоматически при присваивании (`dest = src`). Выполняет **глубокое копирование**: получатель получает собственную независимую копию всех внутренних стволов. Изменение `dest` после копирования не влияет на `src`.

Обычно это не вызывается напрямую — вызов происходит неявно при каждом присваивании `PackedSet`:

```nim
import std/packedsets

var a = [1, 2, 3].toPackedSet
var b = a          # =copy вызывается здесь автоматически

b.incl(99)
assert 99 notin a  # a не затронут — по-настоящему независимая копия
assert 99 in b
```

---

### `assign` (устаревший)

```nim
proc assign*[A](dest: var PackedSet[A], src: PackedSet[A]) {.deprecated.}
```

Более старая явная процедура копирования, предшествующая системе хуков копирования Nim. Теперь она устаревшая в пользу прямого присваивания (`dest = src`), которое автоматически вызывает `=copy`. Приведена здесь для полноты; не используйте в новом коде.

---

### `$` (преобразование в строку)

```nim
proc `$`*[A](s: PackedSet[A]): string
```

Преобразует `s` в удобочитаемую строку вида `{e1, e2, e3}`, с элементами, перечисленными в **возрастающем** порядке порядковых значений. Полезно для отладки и логирования.

```nim
import std/packedsets

let a = [3, 1, 2].toPackedSet
echo a          # {1, 2, 3}  — всегда отсортировано по возрастанию
assert $a == "{1, 2, 3}"

let empty = initPackedSet[int]()
assert $empty == "{}"
```

---

## `PackedSet` vs `sets` — когда что выбирать

| Критерий | `PackedSet` | `HashSet` (std/sets) |
|---|---|---|
| Ограничение типа элемента | Только `Ordinal` | Любой хешируемый тип |
| Поддерживаемые типы | int, enum, char, bool, distinct ordinal | string, object, float, всё с `hash` |
| Память (плотный диапазон, напр. 0–999) | ~128 байт (2 ствола) | ~8 КБ (хеш-таблица) |
| Память (разреженный, напр. 2 случайных int) | 2 ствола + накладные расходы | ~128 байт |
| Проверка принадлежности | O(1), битовая операция без ветвления | O(1), хеш + сравнение |
| Порядок итерации | Возрастание порядкового значения | Произвольный |
| Малое множество (≤ 34 элементов) | Массив, нет выделения в куче | Выделяется хеш-таблица |

**Выбирайте `PackedSet`, когда:** элементы — порядковые типы, кластеризованные в диапазоне; важна эффективность памяти; или нужен детерминированный возрастающий порядок итерации.

**Выбирайте `HashSet`, когда:** элементы — строки, float или пользовательские объекты; или порядковый диапазон настолько разрежён, что накладные расходы стволов `PackedSet` перевешивают стоимость корзин `HashSet`.

---

## Практические паттерны

### Паттерн 1 — Отслеживание посещённых узлов при обходе графа

```nim
import std/packedsets

type NodeId = int

proc bfs(start: NodeId, edges: seq[(NodeId, NodeId)]): seq[NodeId] =
  var visited = initPackedSet[NodeId]()
  var queue = @[start]
  while queue.len > 0:
    let node = queue[0]
    queue.delete(0)
    if containsOrIncl(visited, node):
      continue   # уже посещён
    result.add(node)
    for (a, b) in edges:
      if a == node and b notin visited: queue.add(b)
```

---

### Паттерн 2 — Быстрые операции над множествами флагов перечисления

```nim
import std/packedsets

type Permission = enum Read, Write, Execute, Admin

let userPerms  = [Read, Write].toPackedSet
let adminPerms = [Read, Write, Execute, Admin].toPackedSet
let filePerms  = [Read, Execute].toPackedSet

# Что может делать этот пользователь с этим файлом?
let effective = userPerms * filePerms   # пересечение
assert effective == [Read].toPackedSet

# Есть ли у пользователя все права администратора?
assert not (adminPerms <= userPerms)

# Каких прав не хватает пользователю?
let missing = adminPerms - userPerms
assert missing == [Execute, Admin].toPackedSet
```

---

### Паттерн 3 — Дедупликация большого списка целых чисел

```nim
import std/packedsets

# toPackedSet удаляет дубликаты значительно эффективнее,
# чем сортировка + уникализация для порядковых типов в ограниченном диапазоне.
let rawIds = @[5, 3, 8, 3, 1, 5, 9, 1]
let uniqueIds = rawIds.toPackedSet

assert uniqueIds.len == 5
# Итерация даёт отсортированный результат:
import std/sequtils
assert toSeq(uniqueIds.items) == @[1, 3, 5, 8, 9]
```

---

### Паттерн 4 — Симметричная разность для нахождения изменившихся элементов

```nim
import std/packedsets

let before = [1, 2, 3, 4].toPackedSet
let after  = [2, 3, 4, 5].toPackedSet

let changed = symmetricDifference(before, after)
# changed содержит элементы, которые были добавлены или удалены
assert changed == [1, 5].toPackedSet

let added   = after - before
let removed = before - after
assert added   == [5].toPackedSet
assert removed == [1].toPackedSet
```

---

### Паттерн 5 — Накопление результатов с помощью `containsOrIncl`

```nim
import std/packedsets

# Подсчитать количество уникальных значений в потоке
proc countUnique(values: seq[int]): int =
  var seen = initPackedSet[int]()
  for v in values:
    if not containsOrIncl(seen, v):
      inc result

assert countUnique(@[1, 1, 2, 3, 2, 4]) == 4
```
