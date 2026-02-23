# options — Справочник по модулю

> **Стандартная библиотека Nim** | `import std/options`

## Что такое Option?

**`Option[T]`** — это контейнер, который либо содержит ровно одно значение типа `T`, либо не содержит ничего. Он делает понятие «значение, которого может не быть» явным на уровне системы типов, избавляя от необходимости прибегать к соглашениям вроде возврата `-1`, `""` или `nil`.

```
Option[int]
├── some(42)   — содержит целое число 42
└── none(int)  — не содержит ничего
```

**Зачем это нужно?** В Nim `nil` работает только для указательных и ссылочных типов. Для типов-значений — `int`, `float`, `object` — естественного «отсутствующего» значения нет. `Option[T]` решает эту проблему единообразно для любого типа. Кроме того, он заставляет вызывающий код явно обрабатывать ситуацию «значения нет» — компилятор не позволит молча использовать значение, которого может не существовать.

**Золотое правило:** Никогда не вызывайте `get` без предварительной проверки `isSome`. Вызов `get` на `none` поднимает `UnpackDefect` — а поскольку он наследует `Defect`, его не следует перехватывать.

---

## Содержание

1. [Типы](#типы)
   - [Option\[T\]](#optiont)
   - [UnpackDefect](#unpackdefect)
2. [Конструкторы](#конструкторы)
   - [some](#some)
   - [none (typedesc)](#none-typedesc)
   - [none (обобщённый)](#none-обобщённый)
   - [option](#option)
3. [Проверка наличия значения](#проверка-наличия-значения)
   - [isSome](#issome)
   - [isNone](#isnone)
4. [Извлечение значения](#извлечение-значения)
   - [get (безопасный, только чтение)](#get-безопасный-только-чтение)
   - [get (с умолчанием)](#get-с-умолчанием)
   - [get (изменяемый)](#get-изменяемый)
   - [unsafeGet](#unsafeget)
5. [Преобразование Option](#преобразование-option)
   - [map (побочный эффект)](#map-побочный-эффект)
   - [map (трансформация)](#map-трансформация)
   - [flatMap](#flatmap)
   - [flatten](#flatten)
   - [filter](#filter)
6. [Операторы и утилиты](#операторы-и-утилиты)
   - [==](#оператор-)
   - [$](#оператор--1)
7. [Сопоставление с образцом](#сопоставление-с-образцом)
8. [Полный пример](#полный-пример)
9. [Краткая сводка](#краткая-сводка)

---

## Типы

### `Option[T]`

```nim
type Option*[T] = object
```

Обобщённый контейнер-тип-значение. Для указательных типов (`ref`, `ptr`, `pointer`, `proc`, `iterator {.closure.}`) отсутствующее значение представлено внутри как `nil` — дополнительный булев флаг не нужен. Для всех остальных типов скрытое поле `has: bool` отслеживает наличие значения. Это различие полностью внутреннее; публичный API ведёт себя одинаково в обоих случаях.

Поскольку `Option[T]` — тип-значение (`object`, а не `ref object`), присваивание копирует его целиком. Алиасинга нет.

---

### `UnpackDefect`

```nim
type UnpackDefect* = object of Defect
```

Выбрасывается методом `get`, если вызван на `none`. Поскольку наследует `Defect` (а не `Exception`), он представляет **ошибку программиста** — ту же категорию, что выход за границы массива или разыменование `nil`. Перехватывать его не следует; вместо этого проверяйте `isSome` до вызова `get`.

---

## Конструкторы

### `some`

```nim
proc some*[T](val: sink T): Option[T]
```

Создаёт `Option`, **содержащий** значение `val`. Значение перемещается в option (`sink`), поэтому для перемещаемых типов копирования не происходит.

Для указательных типов `some` проверяет в рантайме, что `val` не равен `nil`. Если у вас есть указатель, который может быть `nil`, используйте вместо этого `option`.

```nim
let a = some(42)          # Option[int], содержит значение
let b = some("привет")    # Option[string], содержит значение
let c = some(@[1, 2, 3])  # Option[seq[int]], содержит значение

assert a.isSome
assert a.get == 42
```

---

### `none` (typedesc)

```nim
proc none*(T: typedesc): Option[T]
```

Создаёт **пустой** `Option` для указанного типа. Тип нужно передать явно, так как Nim не может вывести его из ничего.

```nim
let a = none(int)          # пустой Option[int]
let b = none(string)       # пустой Option[string]
let c = none(seq[float])   # пустой Option[seq[float]]

assert a.isNone
```

---

### `none` (обобщённый)

```nim
proc none*[T]: Option[T]
```

Обобщённый псевдоним для `none(T)`. Удобен, когда параметр типа может быть выведен из контекста (например, как возвращаемое значение в типизированной функции), — позволяет не писать имя типа дважды.

```nim
proc safeDivide(a, b: int): Option[int] =
  if b == 0: return none[int]()   # или: return none(int)
  some(a div b)
```

---

### `option`

```nim
proc option*[T](val: sink T): Option[T]
```

«Умный» конструктор для указательных типов: преобразует `nil` в `none(T)`, а любое ненулевое значение — в `some(val)`. Для не-указательных типов ведёт себя идентично `some(val)`.

Используйте его, когда получаете указатель, который может законно быть `nil`, и хотите поднять его в мир `Option` без явной проверки `if`.

```nim
type Node = ref object
  value: int

proc findNode(id: int): Node = nil  # возвращает nil, если не найдено

# option() сам обрабатывает преобразование nil → none
let n = option(findNode(99))
assert n.isNone

# Для не-указательных типов option и some эквивалентны:
assert option(42) == some(42)
```

---

## Проверка наличия значения

Всегда проверяйте перед извлечением. Эти два предиката — основа безопасного использования `Option`.

### `isSome`

```nim
proc isSome*[T](self: Option[T]): bool
```

Возвращает `true`, если option содержит значение, и `false`, если он пуст. Это стандартная защита перед вызовом `get`.

```nim
let x = some(10)
let y = none(int)

assert x.isSome      # true
assert not y.isSome  # false

if x.isSome:
  echo x.get  # безопасно — мы проверили перед вызовом
```

---

### `isNone`

```nim
proc isNone*[T](self: Option[T]): bool
```

Логическая инверсия `isSome`. Возвращает `true`, если option пуст. Используйте тот вариант, который читается естественнее в вашем контексте.

```nim
let result = none(string)

if result.isNone:
  echo "Ничего не найдено."
```

---

## Извлечение значения

### `get` (безопасный, только чтение)

```nim
proc get*[T](self: Option[T]): lent T
```

Возвращает содержащееся значение как `lent`-ссылку (borrowed). Выбрасывает `UnpackDefect`, если option пуст. Возвращённая ссылка действительна, пока жив сам option.

Это стандартный способ прочитать значение после подтверждения его наличия через `isSome`.

```nim
let x = some("привет")
echo x.get          # "привет"
echo x.get.len      # 6 — методы можно вызывать прямо на результате

let y = none(string)
# y.get  ← здесь будет UnpackDefect в рантайме
```

---

### `get` (с умолчанием)

```nim
proc get*[T](self: Option[T], otherwise: T): T
```

Возвращает содержащееся значение, если оно есть, или `otherwise`, если option пуст. Устраняет необходимость в отдельной проверке `if isSome` во многих распространённых ситуациях и является идиоматическим способом задать запасное значение.

```nim
let found    = some(42)
let notFound = none(int)

echo found.get(0)     # 42  — значение есть, умолчание игнорируется
echo notFound.get(0)  # 0   — option пуст, возвращается умолчание

# Удобно в одну строку, когда умолчание имеет смысл:
let name = getUserName().get("Аноним")
```

---

### `get` (изменяемый)

```nim
proc get*[T](self: var Option[T]): var T
```

Возвращает содержащееся значение как изменяемую ссылку, позволяя менять его на месте. Выбрасывает `UnpackDefect`, если option пуст. Option должен быть `var` для выбора этой перегрузки.

```nim
var counter = some(0)
counter.get += 1       # инкремент на месте
counter.get += 1
assert counter.get == 2

var empty = none(int)
# empty.get += 1  ← UnpackDefect
```

---

### `unsafeGet`

```nim
proc unsafeGet*[T](self: Option[T]): lent T
```

Возвращает содержащееся значение **без проверки** того, является ли option `some`. Если вызвать на `none` — поведение не определено: в отладочных сборках срабатывает assertion, в релизных возможна порча памяти.

Используйте только в горячих внутренних циклах, где вы уже проверили `isSome` и каждая наносекунда на счету. Во всех остальных случаях предпочитайте `get`.

```nim
let x = some(99)
# Мы проверили isSome снаружи — здесь unsafeGet безопасен
if x.isSome:
  echo x.unsafeGet  # 99, без повторной проверки внутри
```

---

## Преобразование Option

Эти процедуры позволяют работать с `Option`-значениями в функциональном стиле — применять преобразования, выстраивать цепочки операций и фильтровать — без ручного распаковывания option.

### `map` (побочный эффект)

```nim
proc map*[T](self: Option[T], callback: proc (input: T))
```

Вызывает `callback` с содержащимся значением, если option является `some`. Ничего не делает, если он `none`. Callback ничего не возвращает (`void`) — эта перегрузка только для побочных эффектов.

Это паттерн «сделать что-то, если значение есть», выраженный без оператора `if`.

```nim
var log: seq[string]

some("событие").map(proc(s: string) = log.add(s))
assert log == @["событие"]

none(string).map(proc(s: string) = log.add(s))
assert log == @["событие"]  # не изменился — none, ничего не произошло
```

---

### `map` (трансформация)

```nim
proc map*[T, R](self: Option[T], callback: proc (input: T): R): Option[R]
```

Применяет `callback` к содержащемуся значению и оборачивает результат в новый `Option`. Если option является `none`, возвращает `none(R)` без вызова callback. Параметр типа `R` выводится из возвращаемого типа callback.

Это фундаментальный строительный блок для преобразования необязательных значений без их распаковки.

```nim
let x = some(6)
let doubled = x.map(proc(n: int): int = n * 2)
assert doubled == some(12)

let y = none(int)
assert y.map(proc(n: int): int = n * 2) == none(int)

# Цепочка трансформаций:
let result = some("  привет  ")
  .map(proc(s: string): string = s.strip)
  .map(proc(s: string): int = s.len)
assert result == some(6)
```

---

### `flatMap`

```nim
proc flatMap*[T, R](self: Option[T],
                    callback: proc (input: T): Option[R]): Option[R]
```

Как `map`, но callback сам возвращает `Option[R]` вместо обычного `R`. Результат автоматически «выравнивается» — вы получаете `Option[R]`, а не `Option[Option[R]]`.

Это ключевой комбинатор для цепочек операций, каждая из которых может завершиться неудачей. Вместо вложенных проверок `if isSome` можно написать чистый пайплайн.

```nim
proc parseIntSafe(s: string): Option[int] =
  try: some(s.parseInt)
  except: none(int)

proc safeSqrt(x: int): Option[float] =
  if x >= 0: some(sqrt(float(x)))
  else: none(float)

# Цепочка: разобрать строку, затем вычислить квадратный корень
let r1 = some("16").flatMap(parseIntSafe).flatMap(safeSqrt)
assert r1 == some(4.0)

let r2 = some("abc").flatMap(parseIntSafe).flatMap(safeSqrt)
assert r2 == none(float)  # разбор не удался, цепочка прервана

let r3 = some("-4").flatMap(parseIntSafe).flatMap(safeSqrt)
assert r3 == none(float)  # корень из отрицательного не определён
```

---

### `flatten`

```nim
proc flatten*[T](self: Option[Option[T]]): Option[T]
```

Убирает один уровень вложенности из `Option[Option[T]]`. Если внешний option является `some`, возвращает внутренний option без изменений. Если внешний option является `none`, возвращает `none(T)`.

Напрямую вызывать это нужно редко — `flatMap` вызывает его внутри. Полезно, когда у вас есть код, независимо порождающий вложенную структуру option.

```nim
let nested    = some(some(42))
let unwrapped = flatten(nested)
assert unwrapped == some(42)

let outerNone = none(Option[int])
assert flatten(outerNone) == none(int)

let innerNone = some(none(int))
assert flatten(innerNone) == none(int)
```

---

### `filter`

```nim
proc filter*[T](self: Option[T], callback: proc (input: T): bool): Option[T]
```

Применяет предикат к содержащемуся значению. Если option является `some` и предикат вернул `true`, option возвращается без изменений. Если предикат вернул `false`, возвращается `none(T)`. Если option уже `none`, он проходит сквозь без изменений.

Думайте об этом как о «сохранить значение, только если оно удовлетворяет условию».

```nim
proc isPositive(x: int): bool = x > 0

assert some(42).filter(isPositive)  == some(42)
assert some(-5).filter(isPositive)  == none(int)
assert none(int).filter(isPositive) == none(int)

# Комбинирование с map:
let result = some("привет мир")
  .filter(proc(s: string): bool = s.len <= 20)
  .map(proc(s: string): string = s.toUpperAscii)
assert result == some("ПРИВЕТ МИР")
```

---

## Операторы и утилиты

### Оператор `==`

```nim
proc `==`*[T](a, b: Option[T]): bool
```

Два option равны, если оба являются `none`, или если оба являются `some` и их содержащиеся значения равны по `==`. `some` и `none` никогда не равны, даже если значение по умолчанию для типа `T` совпадало бы с содержимым `some`.

```nim
assert some(42)   == some(42)   # оба some, равные значения
assert none(int)  == none(int)  # оба none
assert some(42)   != none(int)  # один some, один none
assert some(42)   != some(43)   # оба some, но разные значения
```

---

### Оператор `$`

```nim
proc `$`*[T](self: Option[T]): string
```

Возвращает удобочитаемое строковое представление. `some(x)` даёт `"some(значение)"`, `none(T)` — `"none(ИмяТипа)"`. Удобно для отладки и `echo`.

```nim
echo some(42)          # some(42)
echo some("привет")    # some("привет")
echo none(int)         # none(int)
echo some(@[1, 2, 3])  # some(@[1, 2, 3])
```

---

## Сопоставление с образцом

Если установить пакет [`fusion`](https://github.com/nim-lang/fusion), можно использовать сопоставление с образцом для `Option` через модуль `fusion/matching`. Требует `{.experimental: "caseStmtMacros".}`.

```nim
{.experimental: "caseStmtMacros".}
import fusion/matching

case some(42)
of Some(@value):
  echo "Получено: ", value   # Получено: 42
of None():
  echo "Ничего"

# Вложенные option тоже поддерживаются:
assertMatch(some(some(none(int))), Some(Some(None())))
```

Это выразительнее связки `isSome`/`get` для сложного деструктурирования, но требует отдельного пакета.

---

## Полный пример

Следующий пример показывает реалистичный пайплайн: разбор пользовательского ввода из веб-формы, пошаговая валидация с помощью `Option` и формирование ответа — без единой вложенной проверки `if` и явных проверок `nil`.

```nim
import std/options
import std/strutils

proc parseAge(s: string): Option[int] =
  ## Разобрать строку как целое число; вернуть none, если не получилось.
  try: some(parseInt(s.strip))
  except ValueError: none(int)

proc validateAge(age: int): Option[int] =
  ## Принять только возраст от 0 до 150.
  some(age).filter(proc(a: int): bool = a in 0..150)

proc ageCategory(age: int): string =
  if age < 18: "несовершеннолетний"
  elif age < 65: "взрослый"
  else: "пожилой"

proc processInput(rawAge: string): string =
  let category = parseAge(rawAge)
    .flatMap(validateAge)
    .map(ageCategory)
  category.get("некорректный ввод")

echo processInput("34")    # взрослый
echo processInput("  17 ") # несовершеннолетний
echo processInput("200")   # некорректный ввод  (не прошла валидация)
echo processInput("abc")   # некорректный ввод  (не прошёл разбор)
echo processInput("-5")    # некорректный ввод  (не прошла валидация)
```

---

## Краткая сводка

| Символ | Сигнатура | Описание |
|---|---|---|
| `some` | `some(T)` → `Option[T]` | Обернуть значение — создать непустой option |
| `none` | `none(typedesc)` или `none[T]()` → `Option[T]` | Создать пустой option |
| `option` | `option(T)` → `Option[T]` | Умный конструктор: nil → none, иначе some |
| `isSome` | `Option[T]` → `bool` | True, если option содержит значение |
| `isNone` | `Option[T]` → `bool` | True, если option пуст |
| `get` | `Option[T]` → `lent T` | Извлечь значение; UnpackDefect если none |
| `get` | `Option[T], T` → `T` | Извлечь значение или вернуть умолчание |
| `get` | `var Option[T]` → `var T` | Извлечь изменяемую ссылку |
| `unsafeGet` | `Option[T]` → `lent T` | Извлечь без проверки (UB если none) |
| `map` | `Option[T], (T→void)` → `void` | Выполнить побочный эффект, если some |
| `map` | `Option[T], (T→R)` → `Option[R]` | Трансформировать содержащееся значение |
| `flatMap` | `Option[T], (T→Option[R])` → `Option[R]` | Цепочка операций, каждая из которых может завершиться неудачей |
| `flatten` | `Option[Option[T]]` → `Option[T]` | Убрать один уровень вложенности |
| `filter` | `Option[T], (T→bool)` → `Option[T]` | Сохранить значение, только если предикат истинен |
| `==` | `Option[T], Option[T]` → `bool` | Равенство: оба none, или оба some с равными значениями |
| `$` | `Option[T]` → `string` | Удобочитаемое представление |
