# Справочник модуля `std/sequtils` (Nim)

> Модуль реализует операции не только для типа `seq`, но и для всех контейнеров, подпадающих под тип `openArray`: последовательностей (`seq`), строк (`string`) и массивов (`array`).

---

## Содержание

1. [concat](#concat)
2. [addUnique](#addunique)
3. [count](#count)
4. [cycle](#cycle)
5. [repeat](#repeat)
6. [deduplicate](#deduplicate)
7. [min / max (с компаратором)](#min--max-с-компаратором)
8. [minIndex / maxIndex](#minindex--maxindex)
9. [minmax](#minmax)
10. [findIt](#findit)
11. [zip](#zip)
12. [unzip](#unzip)
13. [distribute](#distribute)
14. [map](#map)
15. [apply](#apply)
16. [filter (итератор и proc)](#filter-итератор-и-proc)
17. [keepIf](#keepif)
18. [delete](#delete)
19. [insert](#insert)
20. [filterIt](#filterit)
21. [keepItIf](#keepitif)
22. [countIt](#countit)
23. [all / allIt](#all--allit)
24. [any / anyIt](#any--anyit)
25. [toSeq](#toseq)
26. [foldl](#foldl)
27. [foldr](#foldr)
28. [mapIt](#mapit)
29. [applyIt](#applyit)
30. [newSeqWith](#newseqwith)
31. [mapLiterals](#mapliterals)
32. [items (для closure-итераторов)](#items-для-closure-итераторов)

---

## concat

```nim
func concat*[T](seqs: varargs[seq[T]]): seq[T]
```

**Описание:** Объединяет несколько последовательностей в одну новую. Все входные последовательности должны быть одного типа.

**Пример:**
```nim
let
  s1 = @[1, 2, 3]
  s2 = @[4, 5]
  s3 = @[6, 7]
  total = concat(s1, s2, s3)
assert total == @[1, 2, 3, 4, 5, 6, 7]
```

> **Обратная операция:** `distribute` — разбивает последовательность на несколько.

---

## addUnique

```nim
func addUnique*[T](s: var seq[T], x: sink T)
```

**Описание:** Добавляет элемент `x` в последовательность `s`, только если он там ещё не присутствует. Для сравнения используется оператор `==`.

**Пример:**
```nim
var a = @[1, 2, 3]
a.addUnique(4)
a.addUnique(4)  # Второй вызов ничего не добавляет
assert a == @[1, 2, 3, 4]
```

---

## count

```nim
func count*[T](s: openArray[T], x: T): int
```

**Описание:** Возвращает количество вхождений элемента `x` в контейнер `s`.

**Пример:**
```nim
let
  a = @[1, 2, 2, 3, 2, 4, 2]
  b = "abracadabra"
assert count(a, 2) == 4
assert count(a, 99) == 0
assert count(b, 'r') == 2
```

---

## cycle

```nim
func cycle*[T](s: openArray[T], n: Natural): seq[T]
```

**Описание:** Возвращает новую последовательность, в которой элементы контейнера `s` повторяются `n` раз. `n` должно быть неотрицательным числом.

**Пример:**
```nim
let
  s = @[1, 2, 3]
  total = s.cycle(3)
assert total == @[1, 2, 3, 1, 2, 3, 1, 2, 3]
```

---

## repeat

```nim
proc repeat*[T](x: T, n: Natural): seq[T]
```

**Описание:** Возвращает новую последовательность, в которой элемент `x` повторяется `n` раз. В отличие от `cycle`, работает с одиночным элементом, а не контейнером.

**Пример:**
```nim
let total = repeat(5, 3)
assert total == @[5, 5, 5]
```

---

## deduplicate

```nim
func deduplicate*[T](s: openArray[T], isSorted: bool = false): seq[T]
```

**Описание:** Возвращает новую последовательность без дубликатов. Параметр `isSorted` (по умолчанию `false`) включает более быстрый алгоритм для уже отсортированных данных.

**Пример:**
```nim
let
  dup1 = @[1, 1, 3, 4, 2, 2, 8, 1, 4]
  dup2 = @["a", "a", "c", "d", "d"]
  unique1 = deduplicate(dup1)
  unique2 = deduplicate(dup2, isSorted = true)
assert unique1 == @[1, 3, 4, 2, 8]
assert unique2 == @["a", "c", "d"]
```

> **Важно:** Без `isSorted = true` порядок первого вхождения каждого уникального элемента сохраняется. С `isSorted = true` работает как алгоритм `unique` — быстрее, но требует отсортированных входных данных.

---

## min / max (с компаратором)

```nim
proc min*[T](x: openArray[T], cmp: proc(a, b: T): int): T
proc max*[T](x: openArray[T], cmp: proc(a, b: T): int): T
```

**Описание:** Находит минимальный или максимальный элемент массива с использованием пользовательской функции сравнения. Функция `cmp` должна возвращать отрицательное число, ноль или положительное число (по аналогии с C `qsort`).

**Пример:**
```nim
import std/sugar

let words = @["apple", "fig", "banana"]
# Минимальное слово по длине
let shortest = words.min((a, b) => a.len - b.len)
assert shortest == "fig"

# Максимальное слово по длине
let longest = words.max((a, b) => a.len - b.len)
assert longest == "banana"
```

---

## minIndex / maxIndex

```nim
func minIndex*[T](s: openArray[T]): int
func minIndex*[T](s: openArray[T], cmp: proc(a, b: T): int): int

func maxIndex*[T](s: openArray[T]): int
func maxIndex*[T](s: openArray[T], cmp: proc(a, b: T): int): int
```

**Описание:** Возвращает индекс минимального или максимального элемента. Версии без компаратора требуют, чтобы тип `T` поддерживал оператор `<`. Версии с компаратором принимают пользовательскую функцию сравнения.

**Пример (без компаратора):**
```nim
let
  a = @[1, 2, 3, 4]
  b = @[6, 5, 4, 3]
  c = [2, -7, 8, -5]
assert minIndex(a) == 0   # минимум — 1, находится на индексе 0
assert minIndex(b) == 3   # минимум — 3, находится на индексе 3
assert minIndex(c) == 1   # минимум — -7, находится на индексе 1
assert maxIndex(c) == 2   # максимум — 8, находится на индексе 2
```

**Пример (с компаратором):**
```nim
import std/sugar

let s1 = @["foo", "bar", "hello"]
# Индекс самой короткой строки
assert minIndex(s1, (a, b) => a.len - b.len) == 0
# Индекс самой длинной строки
assert maxIndex(s1, (a, b) => a.len - b.len) == 2
```

---

## minmax

```nim
func minmax*[T](x: openArray[T]): (T, T)
func minmax*[T](x: openArray[T], cmp: proc(a, b: T): int): (T, T)
```

**Описание:** Возвращает кортеж `(минимум, максимум)` за один проход по контейнеру. Эффективнее последовательного вызова `min` и `max`.

**Пример:**
```nim
let
  a = @[3, 1, 4, 1, 5, 9, 2, 6]
  (lo, hi) = minmax(a)
assert lo == 1
assert hi == 9
```

---

## findIt

```nim
template findIt*(s, predicate: untyped): int
```

**Описание:** Перебирает контейнер и возвращает индекс первого элемента, удовлетворяющего предикату. Если ни один элемент не найден — возвращает `-1`. Предикат записывается как выражение с переменной `it`.

**Пример:**
```nim
let a = @[10, 20, 30, 40, 50]
assert findIt(a, it > 25) == 2    # первый элемент > 25 это 30, индекс 2
assert findIt(a, it > 100) == -1  # не найдено
```

---

## zip

```nim
proc zip*[S, T](s1: openArray[S], s2: openArray[T]): seq[(S, T)]
```

**Описание:** Объединяет два контейнера в последовательность кортежей. Если контейнеры разной длины, лишние элементы более длинного контейнера отбрасываются.

**Пример:**
```nim
let
  short = @[1, 2, 3]
  long  = @[6, 5, 4, 3, 2, 1]
  words = @["one", "two", "three"]
  zip1  = zip(short, long)
  zip2  = zip(short, words)
assert zip1 == @[(1, 6), (2, 5), (3, 4)]
assert zip2 == @[(1, "one"), (2, "two"), (3, "three")]
```

---

## unzip

```nim
proc unzip*[S, T](s: openArray[(S, T)]): (seq[S], seq[T])
```

**Описание:** Обратная операция к `zip`. Принимает последовательность двухэлементных кортежей и возвращает кортеж из двух последовательностей.

**Пример:**
```nim
let
  zipped   = @[(1, 'a'), (2, 'b'), (3, 'c')]
  (nums, chars) = zipped.unzip()
assert nums  == @[1, 2, 3]
assert chars == @['a', 'b', 'c']
```

---

## distribute

```nim
func distribute*[T](s: seq[T], num: Positive, spread = true): seq[seq[T]]
```

**Описание:** Разбивает последовательность `s` на `num` подпоследовательностей. Является частичной обратной операцией к `concat`.

- Если `spread = true` (по умолчанию): остаток от деления распределяется равномерно по подпоследовательностям — подходит для многопоточных задач.
- Если `spread = false`: первые подпоследовательности максимально заполняются, остаток помещается в последнюю.

**Пример:**
```nim
let numbers = @[1, 2, 3, 4, 5, 6, 7]
assert numbers.distribute(3)        == @[@[1, 2, 3], @[4, 5], @[6, 7]]
assert numbers.distribute(3, false) == @[@[1, 2, 3], @[4, 5, 6], @[7]]
```

---

## map

```nim
proc map*[T, S](s: openArray[T], op: proc(x: T): S): seq[S]
```

**Описание:** Применяет функцию `op` к каждому элементу контейнера и возвращает новую последовательность с результатами. Не изменяет исходный контейнер. Позволяет преобразовывать тип элементов.

**Пример:**
```nim
let
  a = @[1, 2, 3, 4]
  b = map(a, proc(x: int): string = $x)
assert b == @["1", "2", "3", "4"]
```

**С синтаксическим сахаром из модуля `sugar`:**
```nim
import std/sugar

let a = @[1, 2, 3, 4]
let b = a.map(x => x * x)
assert b == @[1, 4, 9, 16]
```

---

## apply

```nim
# Вариант 1: принимает var T — изменяет элемент напрямую
proc apply*[T](s: var openArray[T], op: proc(x: var T))

# Вариант 2: принимает T, возвращает T — заменяет элемент
proc apply*[T](s: var openArray[T], op: proc(x: T): T)

# Вариант 3: принимает T, ничего не возвращает — для побочных эффектов
proc apply*[T](s: openArray[T], op: proc(x: T))
```

**Описание:** Применяет функцию `op` к каждому элементу контейнера. В отличие от `map`, работает на месте (in-place) для первых двух вариантов, изменяя исходный контейнер.

**Пример (вариант 1 — изменение через var):**
```nim
var a = @["1", "2", "3", "4"]
apply(a, proc(x: var string) = x &= "42")
assert a == @["142", "242", "342", "442"]
```

**Пример (вариант 2 — замена через возврат):**
```nim
var a = @["1", "2", "3", "4"]
apply(a, proc(x: string): string = x & "42")
assert a == @["142", "242", "342", "442"]
```

**Пример (вариант 3 — только побочные эффекты):**
```nim
var message: string
apply([0, 1, 2, 3, 4], proc(item: int) = message.addInt item)
assert message == "01234"
```

---

## filter (итератор и proc)

### Итератор
```nim
iterator filter*[T](s: openArray[T], pred: proc(x: T): bool): T
```

**Описание:** Перебирает контейнер `s` и последовательно **выдаёт** (yield) только те элементы, которые удовлетворяют предикату `pred`. Полезен, когда не нужно создавать промежуточную последовательность.

**Пример:**
```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
var evens = newSeq[int]()
for n in filter(numbers, proc(x: int): bool = x mod 2 == 0):
  evens.add(n)
assert evens == @[4, 8, 4]
```

### Процедура
```nim
proc filter*[T](s: openArray[T], pred: proc(x: T): bool): seq[T]
```

**Описание:** Возвращает новую последовательность, содержащую только элементы, удовлетворяющие предикату.

**Пример:**
```nim
let
  colors = @["red", "yellow", "black"]
  f1 = filter(colors, proc(x: string): bool = x.len < 6)
  f2 = filter(colors, proc(x: string): bool = x.contains('y'))
assert f1 == @["red", "black"]
assert f2 == @["yellow"]
```

---

## keepIf

```nim
proc keepIf*[T](s: var seq[T], pred: proc(x: T): bool)
```

**Описание:** Модифицирует последовательность `s` на месте, оставляя только элементы, удовлетворяющие предикату. Аналог `filter`, но без создания новой последовательности.

**Пример:**
```nim
var floats = @[13.0, 12.5, 5.8, 2.0, 6.1, 9.9, 10.1]
keepIf(floats, proc(x: float): bool = x > 10)
assert floats == @[13.0, 12.5, 10.1]
```

---

## delete

```nim
func delete*[T](s: var seq[T]; slice: Slice[int])
```

**Описание:** Удаляет элементы последовательности `s` в диапазоне `slice`. Вызывает `IndexDefect`, если срез выходит за границы последовательности. Перемещает все элементы после удалённого диапазона — операция линейного времени O(n).

**Пример:**
```nim
var a = @[10, 11, 12, 13, 14]
a.delete(4..4)    # Удаляем последний элемент
assert a == @[10, 11, 12, 13]
a.delete(1..2)    # Удаляем элементы с индекса 1 по 2
assert a == @[10, 13]
a.delete(1..<1)   # Пустой срез — ничего не происходит
assert a == @[10, 13]
```

> **Устаревший вариант:** `delete(s, first, last)` (два отдельных индекса) помечен как deprecated — используйте срезы.

---

## insert

```nim
func insert*[T](dest: var seq[T], src: openArray[T], pos = 0)
```

**Описание:** Вставляет элементы из `src` в последовательность `dest`, начиная с позиции `pos`. Изменяет `dest` на месте. Типы элементов `src` и `dest` должны совпадать.

**Пример:**
```nim
var dest = @[1, 1, 1, 1, 1, 1, 1, 1]
let src  = @[2, 2, 2, 2, 2, 2]
dest.insert(src, 3)
assert dest == @[1, 1, 1, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1]
```

---

## filterIt

```nim
template filterIt*(s, pred: untyped): untyped
```

**Описание:** Шаблонная версия `filter`. Возвращает новую последовательность с элементами, удовлетворяющими предикату. Предикат записывается как выражение с переменной `it` — без необходимости определять анонимную функцию.

**Пример:**
```nim
let
  temperatures  = @[-272.15, -2.0, 24.5, 44.31, 99.9, -113.44]
  acceptable    = temperatures.filterIt(it < 50 and it > -10)
  notAcceptable = temperatures.filterIt(it > 50 or it < -10)
assert acceptable    == @[-2.0, 24.5, 44.31]
assert notAcceptable == @[-272.15, 99.9, -113.44]
```

---

## keepItIf

```nim
template keepItIf*(varSeq: seq, pred: untyped)
```

**Описание:** Шаблонная версия `keepIf`. Изменяет последовательность на месте, оставляя только элементы, удовлетворяющие предикату. Предикат использует переменную `it`.

**Пример:**
```nim
var candidates = @["foo", "bar", "baz", "foobar"]
candidates.keepItIf(it.len == 3 and it[0] == 'b')
assert candidates == @["bar", "baz"]
```

---

## countIt

```nim
template countIt*(s, pred: untyped): int
```

**Описание:** Возвращает количество элементов контейнера, удовлетворяющих предикату. Предикат записывается с переменной `it`. Работает также с closure-итераторами.

**Пример:**
```nim
let numbers = @[-3, -2, -1, 0, 1, 2, 3, 4, 5, 6]
assert numbers.countIt(it < 0) == 3
assert numbers.countIt(it mod 2 == 0) == 5
```

---

## all / allIt

### all
```nim
proc all*[T](s: openArray[T], pred: proc(x: T): bool): bool
```

**Описание:** Проверяет, удовлетворяют ли **все** элементы контейнера предикату. Возвращает `true` только если предикат истинен для каждого элемента. При первом несоответствии возвращает `false`.

**Пример:**
```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
assert all(numbers, proc(x: int): bool = x < 10) == true
assert all(numbers, proc(x: int): bool = x < 9)  == false
```

### allIt
```nim
template allIt*(s, pred: untyped): bool
```

**Описание:** Шаблонная версия `all` с переменной `it`.

**Пример:**
```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
assert numbers.allIt(it < 10) == true
assert numbers.allIt(it < 9)  == false
```

---

## any / anyIt

### any
```nim
proc any*[T](s: openArray[T], pred: proc(x: T): bool): bool
```

**Описание:** Проверяет, удовлетворяет ли **хотя бы один** элемент контейнера предикату. Возвращает `true` при первом совпадении.

**Пример:**
```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
assert any(numbers, proc(x: int): bool = x > 8) == true
assert any(numbers, proc(x: int): bool = x > 9) == false
```

### anyIt
```nim
template anyIt*(s, pred: untyped): bool
```

**Описание:** Шаблонная версия `any` с переменной `it`.

**Пример:**
```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
assert numbers.anyIt(it > 8) == true
assert numbers.anyIt(it > 9) == false
```

---

## toSeq

```nim
template toSeq*(iter: untyped): untyped
```

**Описание:** Преобразует любой итерируемый объект (диапазон, множество, итератор и т.д.) в последовательность `seq`.

**Пример:**
```nim
let
  myRange = 1..5
  mySet: set[int8] = {5'i8, 3, 1}
  mySeq1 = toSeq(myRange)
  mySeq2 = toSeq(mySet)
assert mySeq1 == @[1, 2, 3, 4, 5]
assert mySeq2 == @[1'i8, 3, 5]  # множества хранятся в отсортированном порядке
```

**С inline-итератором:**
```nim
iterator evens(n: int): int =
  for i in 0..<n:
    if i mod 2 == 0: yield i

assert toSeq(evens(10)) == @[0, 2, 4, 6, 8]
```

---

## foldl

### Без начального значения
```nim
template foldl*(sequence, operation: untyped): untyped
```

**Описание:** Сворачивает последовательность слева направо, используя выражение `operation` с переменными `a` (аккумулятор) и `b` (текущий элемент). Последовательность должна содержать хотя бы один элемент. Тип результата совпадает с типом элементов.

**Пример:**
```nim
let
  numbers      = @[5, 9, 11]
  addition     = foldl(numbers, a + b)       # ((5 + 9) + 11) = 25
  subtraction  = foldl(numbers, a - b)       # ((5 - 9) - 11) = -15
  words        = @["nim", "is", "cool"]
  concatenation = foldl(words, a & b)        # "nimiscool"
assert addition == 25
assert subtraction == -15
assert concatenation == "nimiscool"
```

### С начальным значением
```nim
template foldl*(sequence, operation, first): untyped
```

**Описание:** Версия `foldl` с начальным значением `first`. Тип результата определяется типом `first`, что позволяет накапливать результат другого типа, нежели тип элементов.

**Пример:**
```nim
let
  numbers = @[0, 8, 1, 5]
  # Преобразуем числа в строку цифр
  digits = foldl(numbers, a & (chr(b + ord('0'))), "")
assert digits == "0815"
```

---

## foldr

```nim
template foldr*(sequence, operation: untyped): untyped
```

**Описание:** Сворачивает последовательность справа налево. Переменные `a` и `b`: `a` — текущий элемент слева, `b` — аккумулятор справа. Скобки расставляются как `(1 op (2 op (3)))`.

**Пример:**
```nim
let
  numbers      = @[5, 9, 11]
  addition     = foldr(numbers, a + b)       # (5 + (9 + 11)) = 25
  subtraction  = foldr(numbers, a - b)       # (5 - (9 - 11)) = 7
  words        = @["nim", "is", "cool"]
  concatenation = foldr(words, a & b)        # "nimiscool"
assert addition == 25
assert subtraction == 7    # Отличается от foldl!
assert concatenation == "nimiscool"
```

> **Важно:** Для неассоциативных операций (вычитание, деление) `foldl` и `foldr` дают разные результаты.

---

## mapIt

```nim
template mapIt*(s: typed, op: untyped): untyped
```

**Описание:** Шаблонная версия `map`. Применяет выражение `op` к каждому элементу контейнера и возвращает новую последовательность. Использует переменную `it` для обращения к текущему элементу.

**Пример:**
```nim
let
  nums    = @[1, 2, 3, 4]
  strings = nums.mapIt($(4 * it))
assert strings == @["4", "8", "12", "16"]
```

**Цепочка вызовов:**
```nim
import std/sugar

let result = @[1, 2, 3, 4, 5]
  .mapIt(it * 2)
  .filterIt(it mod 4 != 0)
assert result == @[2, 6, 10]
```

---

## applyIt

```nim
template applyIt*(varSeq, op: untyped)
```

**Описание:** Шаблонная версия `apply`. Изменяет каждый элемент последовательности на месте. Использует переменную `it`. Выражение `op` должно возвращать значение того же типа, что и элемент.

**Пример:**
```nim
var nums = @[1, 2, 3, 4]
nums.applyIt(it * 3)
assert nums == @[3, 6, 9, 12]
assert nums[0] + nums[3] == 15
```

---

## newSeqWith

```nim
template newSeqWith*(len: int, init: untyped): untyped
```

**Описание:** Создаёт новую последовательность длиной `len`, вычисляя выражение `init` для каждого элемента. В отличие от `newSeq`, выражение `init` вычисляется заново для каждого индекса — это позволяет создавать вложенные последовательности без общих ссылок на один объект.

**Пример (вложенные последовательности):**
```nim
var seq2D = newSeqWith(5, newSeq[bool](3))
assert seq2D.len == 5
assert seq2D[0].len == 3
# Каждая внутренняя последовательность — отдельный объект
seq2D[0][0] = true
assert seq2D[1][0] == false  # другие строки не затронуты
```

**Пример (случайные числа):**
```nim
import std/random
var seqRand = newSeqWith(20, rand(1.0))
# Каждый элемент — отдельный вызов rand()
assert seqRand[0] != seqRand[1]  # с высокой вероятностью
```

---

## mapLiterals

```nim
macro mapLiterals*(constructor, op: untyped; nested = true): untyped
```

**Описание:** Применяет операцию `op` ко всем атомарным литералам (числа, строки) в конструкторе AST. Удобно для приведения типов в массивах и кортежах.

- `nested = true` (по умолчанию): операция применяется рекурсивно во вложенных структурах.
- `nested = false`: только к литералам первого уровня.

**Пример (приведение типов):**
```nim
let x = mapLiterals([0.1, 1.2, 2.3, 3.4], int)
assert x is array[4, int]
assert x == [int(0.1), int(1.2), int(2.3), int(3.4)]
# Результат: [0, 1, 2, 3]
```

**Пример (вложенные кортежи):**
```nim
let a = mapLiterals((1.2, (2.3, 3.4), 4.8), int)
let b = mapLiterals((1.2, (2.3, 3.4), 4.8), int, nested = false)
assert a == (1, (2, 3), 4)     # nested=true: преобразует везде
assert b == (1, (2.3, 3.4), 4) # nested=false: только верхний уровень
```

**Пример (преобразование в строку):**
```nim
let c = mapLiterals((1, (2, 3), 4, (5, 6)), `$`)
assert c == ("1", ("2", "3"), "4", ("5", "6"))
```

---

## items (для closure-итераторов)

```nim
iterator items*[T](xs: iterator: T): T
```

**Описание:** Позволяет перебирать элементы closure-итератора. Открывает возможность использовать closure-итераторы совместно с шаблонами `mapIt`, `filterIt`, `allIt`, `anyIt` и другими.

**Пример:**
```nim
iterator countdown3(): int {.closure.} =
  yield 3
  yield 2
  yield 1

let asSeq = toSeq(countdown3)
assert asSeq == @[3, 2, 1]

# Теперь можно использовать с шаблонами:
assert countdown3.allIt(it > 0)
assert countdown3.anyIt(it == 2)
```

---

## Быстрое сравнение: proc vs шаблон (`It`-версии)

| Задача | Proc/func версия | Шаблон (`It`-версия) |
|--------|-----------------|---------------------|
| Преобразовать элементы | `map(s, proc(x: T): S = ...)` | `s.mapIt(выражение с it)` |
| Изменить элементы на месте | `apply(s, proc(x: T): T = ...)` | `s.applyIt(выражение с it)` |
| Отфильтровать | `filter(s, proc(x: T): bool = ...)` | `s.filterIt(выражение с it)` |
| Фильтровать на месте | `keepIf(s, proc(x: T): bool = ...)` | `s.keepItIf(выражение с it)` |
| Найти индекс | *(нет прямого аналога)* | `findIt(s, выражение с it)` |
| Посчитать | *(нет прямого аналога)* | `countIt(s, выражение с it)` |
| Все ли | `all(s, proc(x: T): bool = ...)` | `s.allIt(выражение с it)` |
| Хоть один | `any(s, proc(x: T): bool = ...)` | `s.anyIt(выражение с it)` |

`It`-шаблоны короче в написании и не требуют явного объявления типов. Proc-версии более гибкие: их можно передавать как значения и использовать повторно.

---

## Пример комплексного использования

```nim
import std/sequtils, std/sugar

# Создать последовательность 1..10, удвоить, оставить не кратные 6
let
  foo = toSeq(1..10).map(x => x * 2).filter(x => x mod 6 != 0)
  bar = toSeq(1..10).mapIt(it * 2).filterIt(it mod 6 != 0)

assert foo == bar
assert foo == @[2, 4, 8, 10, 14, 16, 20]

assert foo.any(x => x > 17)        # есть ли элемент > 17?
assert not bar.allIt(it < 20)      # не все элементы < 20
assert foo.foldl(a + b) == 74      # сумма всех элементов
```
