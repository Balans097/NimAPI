# Модуль `system` Nim — полный справочник

> Модуль `system` — это **неявная стандартная библиотека** Nim. Он автоматически импортируется в каждый Nim-файл: `import system` никогда не требуется. Всё, что описано здесь, доступно «из коробки». Многие процедуры помечены `magic`, то есть компилятор обрабатывает их особым образом — специальной оптимизацией или генерацией кода, — а не обычным Nim-кодом.

---

## Содержание

1. [Утилиты системы типов](#1-утилиты-системы-типов)
2. [Память и выделение](#2-память-и-выделение)
3. [Порядковые типы и границы](#3-порядковые-типы-и-границы)
4. [Операции с последовательностями](#4-операции-с-последовательностями)
5. [Операции со строками](#5-операции-со-строками)
6. [Арифметика и преобразования](#6-арифметика-и-преобразования)
7. [Сравнение и проверка вхождения](#7-сравнение-и-проверка-вхождения)
8. [Управление потоком и жизненным циклом программы](#8-управление-потоком-и-жизненным-циклом-программы)
9. [Ввод/вывод и печать](#9-вводвывод-и-печать)
10. [Обработка исключений](#10-обработка-исключений)
11. [Семантика перемещения и хуки жизненного цикла](#11-семантика-перемещения-и-хуки-жизненного-цикла)
12. [Низкоуровневые и небезопасные утилиты](#12-низкоуровневые-и-небезопасные-утилиты)
13. [Метапрограммирование и отладка](#13-метапрограммирование-и-отладка)
14. [Важные типы и константы](#14-важные-типы-и-константы)

---

## 1. Утилиты системы типов

### `typeof(x)`
```
proc typeof(x: untyped; mode = typeOfIter): typedesc
```

Возвращает **тип выражения `x` во время компиляции** без его вычисления. Необходим при написании обобщённого кода, когда тип нужно вывести из выражения, а не задать явно. Необязательный параметр `mode` управляет тем, как интерпретируются перегруженные имена: как итераторы (`typeOfIter`) или как обычные процедуры (`typeOfProc`).

```nim
var n = 42
echo typeof(n)       # int

proc foo(): float = 1.0
echo typeof(foo())   # float (интерпретация typeOfProc)
```

---

### `is` / `isnot`
```
proc `is`[T, S](x: T, y: S): bool
template `isnot`(x, y: untyped): untyped
```

`is` проверяет, принадлежит ли значение `x` **тому же типу**, что и `y` (тип или значение). `isnot` — отрицание. Обе формы вычисляются **во время компиляции** внутри блоков `when` и являются основным инструментом ветвления по типам в обобщённом коде.

```nim
assert 42 is int
assert "hello" isnot int

proc process[T](x: T) =
  when T is string:
    echo "Получена строка: ", x
  else:
    echo "Получено нечто иное"
```

---

### `of`
```
proc `of`[T, S](x: T, y: typedesc[S]): bool
```

Проверяет, является ли объект или ссылка `x` **экземпляром** типа `S` (или потомком `S`). Это аналог `instanceof` в Java — **проверка во время выполнения** для иерархий объектов. В отличие от `is`, работает динамически.

```nim
type
  Animal = ref object of RootObj
  Dog    = ref object of Animal

var a: Animal = Dog()
assert a of Animal   # true — Dog IS-A Animal
assert a of Dog      # true — реальный тип Dog
```

---

### `default[T]`
```
proc default[T](_: typedesc[T]): T
```

Возвращает **значение по умолчанию** для типа `T`. Для целых — `0`, для строк — `""`, для последовательностей — `@[]`. В отличие от `zeroDefault`, учитывает поля объекта с явно заданными начальными значениями (`defaultVal`).

```nim
assert default(int)    == 0
assert default(string) == ""
assert default(bool)   == false

type Point = object
  x, y: float
assert default(Point) == Point(x: 0.0, y: 0.0)
```

---

### `zeroDefault[T]`
```
func zeroDefault[T](_: typedesc[T]): T
```

Аналог `default`, но всегда возвращает **двоичный ноль**, полностью игнорируя объявленные значения по умолчанию у полей объектов. Применяется в низкоуровневом коде, где требуется именно обнулённая память.

```nim
type Config = object
  retries: int = 3    # значение по умолчанию
assert zeroDefault(Config).retries == 0   # значение игнорируется
assert default(Config).retries     == 3   # значение учитывается
```

---

### `sizeof[T]`
```
proc sizeof[T](x: T): int
proc sizeof(x: typedesc): int
```

Возвращает **размер в байтах** значения или типа. Принимает как значения, так и дескрипторы типов. Главным образом используется при взаимодействии с C или при низкоуровневой работе с памятью.

```nim
echo sizeof(int)     # 8 на 64-битных системах
echo sizeof(char)    # 1
echo sizeof(float64) # 8
```

---

### `alignof[T]`
```
proc alignof[T](x: T): int
proc alignof(x: typedesc): int
```

Возвращает **требование к выравниванию** (в байтах) для типа. Выравнивание — граница в памяти, по которой должен располагаться тип. Необходимо при ручном создании размещений в памяти или обёртке C-структур.

```nim
echo alignof(int)    # обычно 8 на 64-битных
echo alignof(char)   # 1
```

---

### `offsetOf`
```
template offsetOf[T](t: typedesc[T]; member: untyped): int
```

Возвращает **байтовое смещение** именованного поля внутри структуры/объекта. Это аналог макроса `offsetof` из C, незаменимый при работе с точными позициями полей в памяти.

```nim
type Vec = object
  x: float32
  y: float32

echo offsetOf(Vec, x)  # 0
echo offsetOf(Vec, y)  # 4
```

---

## 2. Память и выделение

### `new[T]`
```
proc new[T](a: var ref T)
proc new(t: typedesc): auto
proc new[T](a: var ref T, finalizer: proc(x: ref T))
```

Выделяет новый объект типа `T` в куче и сохраняет **трассируемую (управляемую GC) ссылку** в `a`. Трассируемая ссылка сообщает сборщику мусора об объекте. Опциональный `finalizer` вызывается перед освобождением объекта, но не может его предотвратить.

```nim
type Node = object
  value: int
  next: ref Node

var n: ref Node
new(n)
n.value = 42
echo n.value   # 42
```

---

### `unsafeNew[T]`
```
proc unsafeNew[T](a: var ref T, size: Natural)
```

Как `new`, но выделяет ровно `size` байт вместо `sizeof(T)`. Позволяет выделить объект, чей реальный размер больше объявленного — полезно для C-interop или типов переменной длины. **Использовать с предельной осторожностью.**

---

### `wasMoved[T]`
```
proc wasMoved[T](obj: var T)
```

Обнуляет `obj` до двоичного нуля, сигнализируя, что его значение **логически перемещено** в другое место. После вызова деструктор объекта должен быть холостым (no-op). Используется внутренней машиной семантики перемещений.

---

### `deepCopy[T]`
```
proc deepCopy[T](x: out T, y: T)
proc deepCopy[T](y: T): T
```

Выполняет **рекурсивное глубокое копирование** `y` в `x`, обходя цепочки ссылок. В отличие от поверхностного копирования, затрагивающего лишь верхний уровень указателей, глубокое копирование воссоздаёт всю структуру данных. Требуется при использовании `spawn` в параллельном коде. При управлении памятью ARC/ORC поддержку нужно включить флагом `--deepcopy:on`.

```nim
var a = @[@[1, 2], @[3, 4]]
var b = deepCopy(a)
b[0][0] = 99
assert a[0][0] == 1   # a не изменился — настоящее глубокое копирование
```

---

## 3. Порядковые типы и границы

### `high` / `low`
```
proc high[T: Ordinal|enum|range](x: typedesc[T]): T
proc low[T: Ordinal|enum|range](x: typedesc[T]): T
proc high[T](x: openArray[T]): int
proc low[T](x: openArray[T]): int
proc high[I, T](x: array[I, T]): I
proc low[I, T](x: array[I, T]): I
proc high(x: string): int
proc low(x: string): int
```

`high` возвращает **максимальное допустимое значение или индекс**; `low` — **минимальное**. Работают с типами, массивами, последовательностями, строками и другими контейнерами. Это канонический способ написания переносимых диапазонных циклов.

```nim
echo high(int)      # 9223372036854775807
echo low(int8)      # -128

var s = @[10, 20, 30]
for i in low(s)..high(s):
  echo s[i]         # 10, 20, 30

echo high("hello")  # 4 (последний индекс)
```

---

### `ord[T]`
```
func ord[T: Ordinal|enum](x: T): int
```

Преобразует порядковое значение (целое, bool, символ или элемент перечисления) в его **базовое `int`-представление**. Необходимо, когда нужно числовое значение члена перечисления или символа.

```nim
echo ord('A')     # 65
echo ord(true)    # 1
echo ord(false)   # 0

type Color = enum red, green, blue
echo ord(green)   # 1
```

---

### `chr`
```
func chr(u: range[0..255]): char
```

Преобразует целое число в диапазоне `0..255` в соответствующий символ `char`. Обратная операция к `ord` для символов. При выходе за границы диапазона вызывает `RangeDefect`.

```nim
echo chr(65)    # A
echo chr(10)    # символ новой строки
```

---

## 4. Операции с последовательностями

### `newSeq[T]`
```
proc newSeq[T](s: var seq[T], len: Natural)
proc newSeq[T](len = 0.Natural): seq[T]
```

Создаёт новую последовательность длиной `len`, где все элементы **инициализированы нулями**. Эффективнее, чем наращивание последовательности поэлементно, если итоговый размер известен заранее.

```nim
var data = newSeq[int](5)
assert data.len == 5
data[0] = 100
# data == @[100, 0, 0, 0, 0]
```

---

### `newSeqOfCap[T]`
```
proc newSeqOfCap[T](cap: Natural): seq[T]
```

Создаёт пустую последовательность (длина 0) с **заранее выделенной ёмкостью** для `cap` элементов. Используйте, если примерное число добавляемых элементов известно, но инициализировать их пока не нужно. Предотвращает многократное перераспределение памяти при росте.

```nim
var s = newSeqOfCap[string](100)
assert s.len == 0          # длина по-прежнему 0
for i in 0..<50:
  s.add($i)               # перераспределение не требуется
```

---

### `newSeqUninit[T]`
```
func newSeqUninit[T](len: Natural): seq[T]
```

Как `newSeq`, но элементы **не инициализируются**. Доступна только для типов без управляемой памяти (простые типы значений: `int`, `float` и т.п.). Используйте, когда собираетесь немедленно присвоить значение каждому элементу и хотите избежать затрат на обнуление.

```nim
var buf = newSeqUninit[byte](1024)
# Сразу заполняем — не читаем неинициализированные слоты!
for i in 0..<buf.len:
  buf[i] = 0xFF.byte
```

---

### `setLen`
```
proc setLen[T](s: var seq[T], newlen: Natural)
proc setLen(s: var string, newlen: Natural)
```

Изменяет размер последовательности или строки. Если `newlen` больше текущей длины, **новые слоты обнуляются**. Если меньше — последовательность или строка **усекается**. Не освобождает базовое выделение памяти.

```nim
var x = @[1, 2, 3]
x.setLen(5)
assert x == @[1, 2, 3, 0, 0]

x.setLen(2)
assert x == @[1, 2]
```

---

### `add`
```
proc add[T](x: var seq[T], y: sink T)
proc add[T](x: var seq[T], y: openArray[T])
proc add(x: var string, y: char)
proc add(x: var string, y: string)
```

Добавляет элемент или коллекцию в последовательность или строку **на месте**. Это идиоматический способ наращивания контейнера по одному элементу. В Nim это имя используется единообразно для всех типов контейнеров.

```nim
var s: seq[int]
s.add(1)
s.add(2)
s.add([3, 4, 5])
assert s == @[1, 2, 3, 4, 5]

var t = "hello"
t.add(' ')
t.add("world")
assert t == "hello world"
```

---

### `del[T]`
```
proc del[T](x: var seq[T], i: Natural)
```

Удаляет элемент с индексом `i`, заменяя его **последним элементом** последовательности, а затем уменьшает длину на единицу. Операция **O(1)**, но **не сохраняет порядок**. Если порядок важен — используйте `delete`.

```nim
var a = @[10, 20, 30, 40, 50]
a.del(1)
# Элемент с индексом 1 заменён последним
assert a == @[10, 50, 30, 40]
```

---

### `delete[T]`
```
proc delete[T](x: var seq[T], i: Natural)
```

Удаляет элемент с индексом `i`, сдвигая все последующие элементы на одну позицию влево. Операция **O(n)**, которая **сохраняет порядок**, в отличие от `del`.

```nim
var a = @[10, 20, 30, 40, 50]
a.delete(1)
assert a == @[10, 30, 40, 50]   # порядок сохранён
```

---

### `insert[T]`
```
proc insert[T](x: var seq[T], item: sink T, i = 0.Natural)
```

Вставляет `item` в позицию `i`, сдвигая все последующие элементы вправо. Операция O(n). По умолчанию вставка происходит в начало (позиция 0).

```nim
var a = @[1, 3, 5]
a.insert(99, 1)
assert a == @[1, 99, 3, 5]
```

---

### `pop[T]`
```
proc pop[T](s: var seq[T]): T
```

Удаляет и возвращает **последний элемент** последовательности, используя её как стек. При пустой последовательности вызывает `IndexDefect`.

```nim
var stack = @[10, 20, 30]
let top = stack.pop()
assert top   == 30
assert stack == @[10, 20]
```

---

### `find[T]`
```
proc find[T, S](a: T, item: S): int
```

Ищет `item` в контейнере `a` и возвращает **первый индекс** вхождения или **-1**, если элемент не найден.

```nim
var a = @[5, 10, 15, 20]
assert a.find(15) == 2
assert a.find(99) == -1
```

---

### `contains[T]`
```
proc contains[T](a: openArray[T], item: T): bool
```

Возвращает `true`, если `item` присутствует в `a`. Обеспечивает работу оператора `in`: `x in a` и `a.contains(x)` равнозначны.

```nim
var a = @[1, 3, 5]
assert 3 in a
assert 99 notin a
```

---

### `@` (массив/openArray → последовательность)
```
proc `@`[IDX, T](a: sink array[IDX, T]): seq[T]
proc `@`[T](a: openArray[T]): seq[T]
```

Преобразует массив (или openArray) в последовательность, копируя все элементы. Так работают литералы последовательностей: `@[1, 2, 3]` — это синтаксический сахар для преобразования массива `[1, 2, 3]` в `seq[int]`.

```nim
let arr = [10, 20, 30]
let s   = @arr
assert s == @[10, 20, 30]
assert s is seq[int]
```

---

### `len`
```
func len[T](x: seq[T]): int
func len(x: string): int
func len[T](x: openArray[T]): int
func len(x: array): int
proc len[U, V](x: HSlice[U, V]): int
```

Возвращает **количество элементов** (или символов) в контейнере. Для `HSlice` — `max(0, b - a + 1)`.

```nim
echo @[1, 2, 3].len   # 3
echo "hello".len      # 5
echo (2..5).len       # 4
```

---

## 5. Операции со строками

### `&` (конкатенация строк)
```
proc `&`(x, y: string): string
proc `&`(x: string, y: char): string
proc `&`(x, y: char): string
```

Создаёт **новую строку**, являющуюся конкатенацией аргументов. Поддерживаются все комбинации строк и символов.

```nim
let s = "Hello" & ", " & "world" & '!'
assert s == "Hello, world!"
```

---

### `&=`
```
proc `&=`(x: var string, y: string)
```

Добавляет `y` к `x` **на месте**, без создания новой строки. Эффективнее, чем `x = x & y`, поскольку избегает лишнего выделения памяти.

```nim
var s = "Hello"
s &= ", world"
assert s == "Hello, world"
```

---

### `newString`
```
proc newString(len: Natural): string
```

Выделяет новую строку ровно в `len` символов с нулевой инициализацией. Затем заполняете посимвольно через оператор индексирования. Существует для оптимизаций, когда длина строки известна заранее.

```nim
var s = newString(5)
s[0] = 'h'; s[1] = 'e'; s[2] = 'l'; s[3] = 'l'; s[4] = 'o'
assert s == "hello"
```

---

### `newStringOfCap`
```
proc newStringOfCap(cap: Natural): string
```

Возвращает пустую строку (`len == 0`) с заранее выделенной внутренней ёмкостью для `cap` байт. Используется для предотвращения перераспределений при построении строки примерно известного размера.

---

### `newStringUninit`
```
proc newStringUninit(len: Natural): string
```

Как `newString`, но содержимое **не инициализировано** (мусор). Каждую позицию необходимо присвоить перед чтением. Полезна в производительном коде, заполняющем каждый байт немедленно.

---

### `insert` (строка)
```
proc insert(x: var string, item: string, i = 0.Natural)
```

Вставляет `item` в строку `x` на байтовую позицию `i`, сдвигая остаток строки вправо. Операция O(n).

```nim
var s = "abcdef"
s.insert("XY", 2)
assert s == "abXYcdef"
```

---

### `substr`
```
proc substr(s: string; first, last: int): string
proc substr(s: string; first = 0): string
proc substr(a: openArray[char]): string
```

Извлекает **подстроку** из `s` от индекса `first` до `last` включительно. Оба индекса безопасно зажимаются: отрицательный `first` становится 0, `last` за пределами строки — `high(s)`. При `first > last` возвращает `""`.

```nim
let s = "Hello, world!"
echo s.substr(7, 11)   # world
echo s.substr(7)       # world!
echo s.substr(-5, 3)   # Hell  (отрицательный first → 0)
```

---

### `addEscapedChar`
```
proc addEscapedChar(s: var string, c: char)
```

Добавляет символ `c` к `s` с применением **escape-последовательностей** для спецсимволов (`\n`, `\t`, `\\`, `\"`, `\'`, непечатаемые байты как `\xHH`). Используется внутри `repr` и при построении отладочного вывода.

```nim
var s = ""
s.addEscapedChar('\n')
s.addEscapedChar('A')
assert s == "\\nA"
```

---

### `addQuoted[T]`
```
proc addQuoted[T](s: var string, x: T)
```

Добавляет `x` к `s` с кавычками, соответствующими типу `x`: строки получают двойные кавычки, символы — одинарные, спецсимволы экранируются. Числа добавляются без изменений.

```nim
var s = ""
s.addQuoted("hi")
s.add(", ")
s.addQuoted('c')
assert s == "\"hi\", 'c'"
```

---

## 6. Арифметика и преобразования

### `abs`
```
proc abs[T: float64|float32](x: T): T
func abs(x: int): int
# также варианты для int8, int16, int32, int64
```

Возвращает **абсолютное значение** `x`. Для целых чисел вызывает исключение переполнения, если `x == low(T)` (поскольку `-low(int)` непредставимо). Для вещественных чисел корректно обрабатывает `NaN` и `-0.0`.

```nim
echo abs(-42)    # 42
echo abs(-3.14)  # 3.14
echo abs(0)      # 0
```

---

### `toFloat` / `toBiggestFloat`
```
proc toFloat(i: int): float
proc toBiggestFloat(i: BiggestInt): BiggestFloat
```

Преобразует целое число в число с плавающей точкой. Аналогично приведению типа `float(i)`, но на отдельных платформах может вызывать `ValueError` при потере точности (на практике редко).

```nim
let x: int = 7
echo x.toFloat / 2.0   # 3.5
```

---

### `toInt` / `toBiggestInt`
```
proc toInt(f: float): int
proc toBiggestInt(f: BiggestFloat): BiggestInt
```

Преобразует вещественное число в целое с **округлением «от нуля»** (0.5 → 1, -0.5 → -1). Это отличается от приведения типа, которое усекает к нулю.

```nim
echo toInt(2.5)    # 3
echo toInt(-2.5)   # -3
echo toInt(2.4)    # 2
```

---

### `/` (деление целых с получением вещественного результата)
```
proc `/`(x, y: int): float
```

Делит два целых числа и возвращает **вещественный** результат. Это отличается от `div`, выполняющего целочисленное (напольное) деление. Результат `7 / 2` равен `3.5`, а не `3`.

```nim
echo 7 / 2     # 3.5
echo 10 / 4    # 2.5
```

---

### `swap[T]`
```
proc swap[T](a, b: var T)
```

Меняет местами значения `a` и `b` **на месте**. Зачастую эффективнее трёхшагового обмена через временную переменную, особенно для ссылочных типов.

```nim
var x = 10
var y = 20
swap(x, y)
assert x == 20
assert y == 10
```

---

### `cmp`
```
proc cmp[T](x, y: T): int
proc cmp(x, y: string): int
```

Возвращает отрицательное значение, если `x < y`; ноль, если `x == y`; положительное, если `x > y`. Обобщённая версия использует `==` и `<`. Строковая версия работает через `strcmp` и более эффективна. Идеально подходит для функций сортировки.

```nim
import std/algorithm
var nums = @[3, 1, 4, 1, 5]
nums.sort(cmp[int])
assert nums == @[1, 1, 3, 4, 5]
```

---

## 7. Сравнение и проверка вхождения

### `contains` (срез)
```
proc contains[U, V, W](s: HSlice[U, V], value: W): bool
```

Проверяет, лежит ли `value` в включительном диапазоне `[s.a, s.b]`. Обеспечивает работу оператора `in` для диапазонов.

```nim
assert 3 in 1..5       # true
assert 6 notin 1..5    # true
```

---

### `..` (конструктор среза)
```
proc `..`[T, U](a: T, b: U): HSlice[T, U]
```

Создаёт включительный диапазон `[a, b]`. Результирующий `HSlice` используется при срезах массивов, конструкторах множеств и в операторе `case`.

```nim
let r = 2..7
echo r.a   # 2
echo r.b   # 7

let arr = [10, 20, 30, 40, 50]
echo arr[1..3]   # @[20, 30, 40]
```

---

### `isNil`
```
proc isNil[T](x: ref T): bool
proc isNil[T](x: ptr T): bool
proc isNil(x: pointer): bool
proc isNil(x: cstring): bool
```

Быстрая проверка на `nil` для ссылочных типов, указателей и замыканий. Нередко быстрее прямого сравнения с `nil`, поскольку компилятор может свести её к одной инструкции.

```nim
var p: ref int
assert p.isNil

new(p)
assert not p.isNil
```

---

## 8. Управление потоком и жизненным циклом программы

### `quit`
```
proc quit(errorcode: int = QuitSuccess)
proc quit(errormsg: string, errorcode = QuitFailure)
```

Немедленно завершает программу с указанным кодом выхода (вторая форма предварительно печатает сообщение в stderr). Выполняет зарегистрированные обработчики выхода. **Обходит** `defer`, обработчики исключений и деструкторы — используйте экономно, особенно в библиотечном коде.

```nim
if criticalFileNotFound:
  quit("Не найден обязательный конфигурационный файл", QuitFailure)
```

---

### `newException`
```
template newException(exceptn: typedesc, message: string;
                      parentException: ref Exception = nil): untyped
```

Создаёт объект исключения заданного типа с заполненным полем `msg`. Это идиоматический способ порождать исключения в Nim.

```nim
raise newException(ValueError, "значение должно быть положительным")
```

---

### `getCurrentException`
```
proc getCurrentException(): ref Exception
```

Внутри блока `except` возвращает **объект текущего активного исключения** или `nil`, если активного исключения нет. Используйте для проверки исключения, цепочки исключений или повторного вызова.

```nim
try:
  raise newException(IOError, "диск заполнен")
except IOError:
  let e = getCurrentException()
  echo e.msg   # "диск заполнен"
```

---

### `getCurrentExceptionMsg`
```
proc getCurrentExceptionMsg(): string
```

Сокращение для `getCurrentException().msg`. Возвращает `""`, если активного исключения нет.

---

### `setControlCHook`
```
proc setControlCHook(hook: proc() {.noconv.})
```

Переопределяет реакцию программы на нажатие Ctrl+C. Обработчик выполняется внутри C-обработчика сигналов, где большинство операций Nim (включая `echo`, работу со строками и исключениями) **небезопасны**. Используйте только атомарные операции или флаги.

```nim
import std/atomics
var shouldStop: Atomic[bool]

proc onCtrlC() {.noconv.} =
  shouldStop.store(true)

setControlCHook(onCtrlC)

while not shouldStop.load():
  doWork()
```

---

### `once`
```
template once(body: untyped)
```

Выполняет `body` **ровно один раз** за время работы программы, независимо от того, сколько раз управление достигает данного места в коде. Реализован через глобальный булев флаг.

```nim
proc render() =
  once:
    initGraphicsSubsystem()
  drawFrame()
```

---

### `closureScope`
```
template closureScope(body: untyped)
```

Оборачивает `body` в немедленно вызываемую анонимную процедуру, вынуждая все локальные переменные захватываться **по значению в момент создания замыкания**. Решает классическую проблему захвата переменной цикла замыканиями.

```nim
var callbacks: seq[proc()]
for i in 0..3:
  closureScope:
    let j = i       # j захватывается со значением ЭТОЙ итерации
    callbacks.add(proc() = echo j)

for cb in callbacks: cb()   # печатает 0, 1, 2, 3 (а не все 3)
```

---

### `likely` / `unlikely`
```
template likely(val: bool): bool
template unlikely(val: bool): bool
```

Подсказывает предсказателю переходов процессора, ожидается ли условие `val` истинным (`likely`) или ложным (`unlikely`) в большинстве случаев. На нативных целях компилируется в `__builtin_expect`. На JS и NimScript не оказывает эффекта.

```nim
for value in data:
  if likely(value >= 0):
    process(value)
  else:
    handleNegative(value)
```

---

### `procCall`
```
proc procCall(x: untyped)
```

Подавляет динамическую диспетчеризацию для вызовов методов, принудительно используя **статическое связывание** — аналог `super.method()` в других ОО-языках. Доступно только во время компиляции.

```nim
type
  Base = ref object of RootObj
  Child = ref object of Base

method greet(a: Base) {.base.} = echo "Привет из Base"
method greet(a: Child) =
  procCall greet(Base(a))   # вызов реализации из Base
  echo "...и из Child"
```

---

## 9. Ввод/вывод и печать

### `echo`
```
proc echo(x: varargs[typed, `$`])
```

Выводит все аргументы в stdout без разделителей, затем добавляет символ новой строки и сбрасывает буфер. Каждый аргумент преобразуется в строку через оператор `$`. Потокобезопасна. Работает на всех бэкендах, включая JavaScript.

```nim
echo "Счётчик: ", 42, ", флаг: ", true
# → Счётчик: 42, флаг: true
```

---

### `debugEcho`
```
proc debugEcho(x: varargs[typed, `$`])
```

Идентична `echo`, но объявлена **без побочных эффектов**. Это позволяет вызывать её внутри `func` (чистых функций) и процедур с `{.noSideEffect.}`, где `echo` была бы запрещена. Предназначена исключительно для отладки.

```nim
func pureComputation(x: int): int =
  debugEcho "вычисляем с x = ", x   # допустимо внутри func
  x * x
```

---

### `writeStackTrace`
```
proc writeStackTrace()
```

Записывает текущий стек вызовов в **stderr**. Полезна только в отладочных сборках (без флага `-d:release`).

---

### `getStackTrace`
```
proc getStackTrace(): string
proc getStackTrace(e: ref Exception): string
```

Возвращает текущий стек вызовов в виде строки (либо трассировку, захваченную при создании исключения `e`). Только для отладочных сборок.

---

## 10. Обработка исключений

### Иерархия исключений (типы)

Ключевые типы исключений, экспортируемые модулем `system`:

| Тип | Описание |
|---|---|
| `Exception` | Корень всех исключений |
| `Defect` | Невосстановимые ошибки времени выполнения (например, выход за границы) |
| `CatchableError` | Базовый класс восстановимых исключений |
| `IOError` | Ошибки ввода/вывода |
| `ValueError` | Неверные значения (например, неправильный формат) |
| `IndexDefect` | Индекс массива/последовательности вне диапазона |
| `RangeDefect` | Значение вне объявленного диапазона |
| `OverflowDefect` | Переполнение при целочисленной арифметике |
| `DivByZeroDefect` | Деление на ноль |
| `NilAccessDefect` | Разыменование `nil` |
| `ObjectConversionDefect` | Неверное приведение типа объекта |

```nim
try:
  var a = @[1, 2, 3]
  echo a[10]   # вызывает IndexDefect
except IndexDefect as e:
  echo "Поймано: ", e.msg
```

---

### `globalRaiseHook` / `localRaiseHook`
```
var globalRaiseHook: proc(e: ref Exception): bool
var localRaiseHook {.threadvar.}: proc(e: ref Exception): bool
```

Хук-процедуры, вызываемые при **каждом операторе `raise`**. Если хук возвращает `false`, исключение поглощается. `globalRaiseHook` глобален для всей программы; `localRaiseHook` — на уровне потока. Предназначены для продвинутых систем диагностики или логирования.

---

### `outOfMemHook`
```
var outOfMemHook: proc()
```

Вызывается средой выполнения при сбое выделения памяти. По умолчанию программа завершается. Задайте собственную процедуру, чтобы вместо этого вызывать исключение Nim (объект исключения необходимо предварительно выделить, пока OOM ещё не произошло).

---

## 11. Семантика перемещения и хуки жизненного цикла

### `move[T]`
```
proc move[T](x: var T): T
```

Передаёт значение `x` в возвращаемое значение и **обнуляет `x`**. После вызова `x` находится в допустимом, но «пустом» состоянии. Эффективнее копирования при передаче права владения.

```nim
var source = @[1, 2, 3, 4, 5]
var dest   = move(source)
assert source.len == 0        # source обнулён
assert dest  == @[1, 2, 3, 4, 5]
```

---

### `ensureMove[T]`
```
proc ensureMove[T](x: T): T
```

Как `move`, но **ошибка компиляции** возникает, если компилятор не может гарантировать реальное перемещение (то есть пришлось бы скопировать). Полезен, когда важно убедиться в отсутствии случайного копирования.

```nim
var s = "hello world"
let t = ensureMove(s)    # ошибка компиляции, если произошло бы копирование
```

---

### `=destroy` / `=copy` / `=sink` / `=dup` / `=trace`

Это **хуки жизненного цикла**, которые пользователь может переопределить для кастомных типов, подключившись к системе управления памятью ARC/ORC:

- `=destroy` — вызывается при выходе объекта из области видимости
- `=copy` — вызывается при присваивании (глубокое копирование)
- `=sink` — вызывается при присваивании с перемещением (передача владения)
- `=dup` — вызывается при дублировании объекта
- `=trace` — вызывается сборщиком циклов для обхода указателей

```nim
type Resource = object
  handle: int

proc `=destroy`(r: Resource) =
  if r.handle != 0:
    closeHandle(r.handle)

proc `=copy`(dest: var Resource, src: Resource) =
  dest.handle = dupHandle(src.handle)
```

---

### `reset[T]`
```
proc reset[T](obj: var T)
```

Сбрасывает `obj` к значению по умолчанию (вызывая деструктор при необходимости). После `reset` объект находится в том же состоянии, что и свежеобъявленная переменная.

```nim
var s = @[1, 2, 3]
reset(s)
assert s.len == 0   # s теперь @[]
```

---

## 12. Низкоуровневые и небезопасные утилиты

### `addr[T]`
```
proc addr[T](x: T): ptr T
```

Берёт **адрес памяти** `x` и возвращает нетрассируемый (сырой) указатель на него. В отличие от `ref`, указатель `ptr` не отслеживается сборщиком мусора. Никогда не сохраняйте результат за пределами времени жизни `x`.

```nim
var n = 42
let p = addr(n)
p[] = 100
assert n == 100
```

---

### `toOpenArray`
```
proc toOpenArray[T](x: seq[T]; first, last: int): openArray[T]
proc toOpenArray[T](x: openArray[T]; first, last: int): openArray[T]
proc toOpenArray[I,T](x: array[I,T]; first, last: I): openArray[T]
proc toOpenArray(x: string; first, last: int): openArray[char]
```

Создаёт **невладеющий вид (срез)** контейнера с индексами от `first` до `last` включительно. Данные не копируются — срез ссылается непосредственно на оригинальное хранилище. Полезен для передачи поддиапазонов в процедуры, принимающие `openArray[T]`.

```nim
proc sum(a: openArray[int]): int =
  for x in a: result += x

let s = @[0, 1, 2, 3, 4, 5]
echo sum(s.toOpenArray(1, 4))   # сумма 1+2+3+4 = 10
```

---

### `cstringArrayToSeq`
```
proc cstringArrayToSeq(a: cstringArray, len: Natural): seq[string]
proc cstringArrayToSeq(a: cstringArray): seq[string]
```

Преобразует C-массив строк с нулевым терминатором в `seq[string]`. Вторая форма определяет длину автоматически, сканируя массив до нулевого указателя.

```nim
# Типично при оборачивании C-API, возвращающих char**
let args = cstringArrayToSeq(rawArgv, rawArgc)
```

---

### `allocCStringArray` / `deallocCStringArray`
```
proc allocCStringArray(a: openArray[string]): cstringArray
proc deallocCStringArray(a: cstringArray)
```

Выделяет C-совместимый массив `char**` с нулевым терминатором из Nim-последовательности строк. После использования необходимо освободить с помощью `deallocCStringArray`.

```nim
let args = @["program", "--verbose", "file.txt"]
let cArgs = allocCStringArray(args)
runCProgram(cArgs)
deallocCStringArray(cArgs)
```

---

### `rawProc` / `rawEnv`
```
proc rawProc[T: proc{.closure.}](x: T): pointer
proc rawEnv[T: proc{.closure.}](x: T): pointer
```

Извлекает сырой указатель на функцию и указатель на захваченное окружение из замыкания. Используется при передаче Nim-замыканий в C-коллбэки, ожидающие параметр `void*` для окружения.

---

### `getTypeInfo[T]`
```
proc getTypeInfo[T](x: T): pointer
```

Возвращает указатель на метаданные типа (RTTI) значения `x`. Используется внутри модуля `typeinfo` и в сценариях рефлексии. В обычном коде предпочтительно использовать модуль `typeinfo`.

---

### `prepareMutation`
```
proc prepareMutation(s: var string)
```

Сигнализирует о предстоящей мутации строки через сырой указатель (через `addr`). В режиме ARC/ORC строковые литералы реализованы как copy-on-write, и эта процедура гарантирует, что у строки есть собственная записываемая копия. Пропуск вызова может привести к `SIGBUS` или `SIGSEGV`.

---

## 13. Метапрограммирование и отладка

### `repr[T]`
```
proc repr[T](x: T): string
```

Возвращает подробное строковое представление любого Nim-значения, включая адреса памяти для ссылок. Корректно обрабатывает циклические структуры данных. Незаменим при отладке сложных данных.

```nim
var data = @[@[1, 2], @[3, 4]]
echo repr(data)
# что-то вроде: 0x...[0x...[1, 2], 0x...[3, 4]]
```

---

### `instantiationInfo`
```
proc instantiationInfo(index = -1, fullPaths = false):
     tuple[filename: string, line: int, column: int]
```

Возвращает **имя файла, номер строки и столбца** места вызова шаблона во время компиляции. Доступна только внутри шаблонов. Используется для реализации макросов-утверждений, сообщающих позицию вызывающего кода.

```nim
template myAssert(cond: bool) =
  if not cond:
    let info = instantiationInfo()
    echo "Утверждение нарушено в ", info.filename, ":", info.line
```

---

### `locals`
```
proc locals(): RootObj
```

Возвращает **кортеж всех локальных переменных** текущей области видимости в точке вызова. Реальный возвращаемый тип — сгенерированный кортеж, а не `RootObj`. Полезен для отладочных дампов.

```nim
proc demo() =
  var x = 10
  var name = "Nim"
  for k, v in fieldPairs(locals()):
    echo k, " = ", v
# выводит:
# x = 10
# name = Nim
```

---

### `finished[T]`
```
proc finished[T: iterator{.closure.}](x: T): bool
```

Проверяет, **завершил ли** итератор-замыкание `x` итерацию (больше нет значений для yield).

```nim
iterator countUp(n: int): int {.closure.} =
  for i in 1..n: yield i

var it = countUp
discard it(3)
# После выдачи всех значений finished(it) == true
```

---

### `arrayWith[T]`
```
proc arrayWith[T](y: T, size: static int): array[size, T]
```

Создаёт массив фиксированного размера, в котором **каждый элемент** является копией `y`. Размер должен быть известен во время компиляции.

```nim
let grid = arrayWith(0, 5)
assert grid == [0, 0, 0, 0, 0]
```

---

### `arrayWithDefault[T]`
```
proc arrayWithDefault[T](size: static int): array[size, T]
```

Создаёт массив фиксированного размера, в котором каждый элемент равен **значению по умолчанию** для типа `T`.

```nim
let flags = arrayWithDefault[bool](4)
assert flags == [false, false, false, false]
```

---

## 14. Важные типы и константы

### Числовые константы
| Константа | Значение | Смысл |
|---|---|---|
| `Inf` | `+∞` (float64) | IEEE-положительная бесконечность |
| `NegInf` | `-∞` (float64) | IEEE-отрицательная бесконечность |
| `NaN` | `NaN` (float64) | IEEE «не число» |
| `QuitSuccess` | `0` | Код выхода при успехе |
| `QuitFailure` | `1` | Код выхода при ошибке |
| `NimVersion` | напр. `"2.0.0"` | Текущая версия Nim в виде строки |

### Компиляционные константы
| Константа | Тип | Смысл |
|---|---|---|
| `hostOS` | `string` | Имя ОС во время компиляции (напр. `"linux"`) |
| `hostCPU` | `string` | Имя процессора во время компиляции (напр. `"amd64"`) |
| `cpuEndian` | `Endianness` | `littleEndian` или `bigEndian` |
| `appType` | `string` | `"console"`, `"gui"` или `"lib"` |

### Ключевые типы
| Тип | Описание |
|---|---|
| `Natural` | `int` в диапазоне `0..high(int)` |
| `Positive` | `int` в диапазоне `1..high(int)` |
| `byte` | Псевдоним для `uint8` |
| `RootObj` | Основание иерархии объектов Nim |
| `RootRef` | `ref RootObj` |
| `NimNode` | Узел AST, используемый в макросах |
| `HSlice[T,U]` | Включительный диапазон `[a..b]` |
| `Slice[T]` | `HSlice[T,T]` — однородный диапазон |
| `Endianness` | Перечисление: `littleEndian` или `bigEndian` |

---

*Данный справочник охватывает публичный API модуля `system.nim` стандартной библиотеки Nim. Внутренние, компиляторные и устаревшие элементы намеренно опущены для наглядности.*
