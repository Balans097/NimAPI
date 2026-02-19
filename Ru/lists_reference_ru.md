# Справочник модуля `std/lists` (Nim)

Модуль реализует четыре вида связных списков:

| Тип | Направление | Кольцевой |
|-----|------------|-----------|
| `SinglyLinkedList[T]` | Односвязный | Нет |
| `DoublyLinkedList[T]` | Двусвязный | Нет |
| `SinglyLinkedRing[T]` | Односвязный | Да |
| `DoublyLinkedRing[T]` | Двусвязный | Да |

**Ключевые особенности:**
- Поля `next`, `prev` и `value` узлов публичны и доступны напрямую.
- Все четыре структуры имеют **семантику значений** (как обычные объекты Nim).
- Кольца (rings) — циклические: последний узел ссылается обратно на первый.
- Узлы (`SinglyLinkedNode[T]`, `DoublyLinkedNode[T]`) — ref-типы, разделяются между списками.

---

## Содержание

1. [Типы данных](#типы-данных)
2. [Создание коллекций](#создание-коллекций)
3. [Создание узлов](#создание-узлов)
4. [Добавление элементов — `add`](#add--добавление-в-конец)
5. [Добавление в начало — `prepend`](#prepend--добавление-в-начало)
6. [Конкатенация — `add` (список + список)](#add-конкатенация-списков)
7. [Конкатенация с перемещением — `addMoved`](#addmoved--конкатенация-с-перемещением)
8. [Добавление в начало списка — `prepend` (список + список)](#prepend-список--список)
9. [Добавление в начало с перемещением — `prependMoved`](#prependmoved--prepend-с-перемещением)
10. [Копирование — `copy`](#copy--поверхностное-копирование)
11. [Удаление — `remove`](#remove--удаление-узла)
12. [Поиск — `find` и `contains`](#find-и-contains--поиск)
13. [Строковое представление — `$`](#-строковое-представление)
14. [Итераторы](#итераторы)
15. [Преобразование из массива — `to*`](#преобразование-из-массива)
16. [Псевдонимы — `append` и `appendMoved`](#псевдонимы-append-и-appendmoved)
17. [Сводная таблица](#сводная-таблица)

---

## Типы данных

### Узлы

```nim
type
  SinglyLinkedNodeObj*[T] = object
    next*: SinglyLinkedNode[T]
    value*: T
  SinglyLinkedNode*[T] = ref SinglyLinkedNodeObj[T]

  DoublyLinkedNodeObj*[T] = object
    next*: DoublyLinkedNode[T]
    prev* {.cursor.}: DoublyLinkedNode[T]
    value*: T
  DoublyLinkedNode*[T] = ref DoublyLinkedNodeObj[T]
```

Узлы — ref-типы. Поля `next`, `prev`, `value` доступны напрямую. Поле `prev` помечено `{.cursor.}` для избежания циклических владений в GC.

---

### Коллекции

```nim
type
  SinglyLinkedList*[T] = object
    head*: SinglyLinkedNode[T]
    tail* {.cursor.}: SinglyLinkedNode[T]

  DoublyLinkedList*[T] = object
    head*: DoublyLinkedNode[T]
    tail* {.cursor.}: DoublyLinkedNode[T]

  SinglyLinkedRing*[T] = object
    head*: SinglyLinkedNode[T]
    tail* {.cursor.}: SinglyLinkedNode[T]

  DoublyLinkedRing*[T] = object
    head*: DoublyLinkedNode[T]
    # у DoublyLinkedRing нет отдельного tail — он доступен через head.prev
```

Поля `head` и `tail` публичны и доступны напрямую. У `DoublyLinkedRing` нет отдельного поля `tail`: предыдущий от `head` — это последний элемент (`head.prev`).

---

### Псевдонимы типов

```nim
SomeLinkedList*[T]       = SinglyLinkedList[T] | DoublyLinkedList[T]
SomeLinkedRing*[T]       = SinglyLinkedRing[T] | DoublyLinkedRing[T]
SomeLinkedCollection*[T] = SomeLinkedList[T] | SomeLinkedRing[T]
SomeLinkedNode*[T]       = SinglyLinkedNode[T] | DoublyLinkedNode[T]
```

Используются в сигнатурах обобщённых процедур для принятия любого из типов коллекций.

---

## Создание коллекций

### `initSinglyLinkedList`

```nim
proc initSinglyLinkedList*[T](): SinglyLinkedList[T]
```

**Описание:** Создаёт пустой односвязный список. Начиная с Nim v0.20 списки инициализируются по умолчанию, явный вызов необязателен.

**Пример:**
```nim
var a = initSinglyLinkedList[int]()
# Эквивалентно:
var b: SinglyLinkedList[int]
```

---

### `initDoublyLinkedList`

```nim
proc initDoublyLinkedList*[T](): DoublyLinkedList[T]
```

**Описание:** Создаёт пустой двусвязный список.

**Пример:**
```nim
var list = initDoublyLinkedList[int]()
list.add(1)
list.add(2)
assert $list == "[1, 2]"
```

---

### `initSinglyLinkedRing`

```nim
proc initSinglyLinkedRing*[T](): SinglyLinkedRing[T]
```

**Описание:** Создаёт пустое односвязное кольцо.

**Пример:**
```nim
var ring = initSinglyLinkedRing[int]()
ring.add(1)
ring.add(2)
ring.add(3)
# Последний узел замыкается обратно на первый:
# 1 → 2 → 3 → 1 → ...
```

---

### `initDoublyLinkedRing`

```nim
proc initDoublyLinkedRing*[T](): DoublyLinkedRing[T]
```

**Описание:** Создаёт пустое двусвязное кольцо.

**Пример:**
```nim
var ring = initDoublyLinkedRing[int]()
ring.add(5)
ring.add(10)
# Каждый узел знает своего предыдущего и следующего,
# образуя замкнутую цепочку.
```

---

## Создание узлов

### `newSinglyLinkedNode`

```nim
proc newSinglyLinkedNode*[T](value: T): SinglyLinkedNode[T]
```

**Описание:** Создаёт новый узел односвязного списка с заданным значением. Позволяет работать с узлом напрямую до вставки в список.

**Пример:**
```nim
let n = newSinglyLinkedNode[int](5)
assert n.value == 5
assert n.next == nil
```

---

### `newDoublyLinkedNode`

```nim
proc newDoublyLinkedNode*[T](value: T): DoublyLinkedNode[T]
```

**Описание:** Создаёт новый узел двусвязного списка или кольца с заданным значением.

**Пример:**
```nim
let n = newDoublyLinkedNode[int](5)
assert n.value == 5
assert n.next == nil
assert n.prev == nil
```

---

## `add` — добавление в конец

Добавляет узел или значение в **конец** коллекции. Сложность: O(1) для всех типов.

### Для `SinglyLinkedList`

```nim
proc add*[T](L: var SinglyLinkedList[T], n: SinglyLinkedNode[T])
proc add*[T](L: var SinglyLinkedList[T], value: T)
```

**Пример:**
```nim
var a = initSinglyLinkedList[int]()
a.add(10)
a.add(20)
let n = newSinglyLinkedNode[int](30)
a.add(n)
assert $a == "[10, 20, 30]"
```

---

### Для `DoublyLinkedList`

```nim
proc add*[T](L: var DoublyLinkedList[T], n: DoublyLinkedNode[T])
proc add*[T](L: var DoublyLinkedList[T], value: T)
```

**Пример:**
```nim
var list = initDoublyLinkedList[int]()
let
  a = newDoublyLinkedNode[int](3)
  b = newDoublyLinkedNode[int](7)
list.add(a)
list.add(b)
assert a.next == b
assert b.prev == a
assert b.next == nil
```

---

### Для `SinglyLinkedRing`

```nim
proc add*[T](L: var SinglyLinkedRing[T], n: SinglyLinkedNode[T])
proc add*[T](L: var SinglyLinkedRing[T], value: T)
```

**Пример:**
```nim
var ring = initSinglyLinkedRing[int]()
ring.add(1)
ring.add(2)
ring.add(3)
# ring.tail.next == ring.head (кольцо замкнуто)
assert ring.tail.next == ring.head
```

---

### Для `DoublyLinkedRing`

```nim
proc add*[T](L: var DoublyLinkedRing[T], n: DoublyLinkedNode[T])
proc add*[T](L: var DoublyLinkedRing[T], value: T)
```

**Пример:**
```nim
var ring = initDoublyLinkedRing[int]()
ring.add(1)
ring.add(2)
assert ring.head.prev.value == 2   # последний = 2
assert ring.head.prev.next == ring.head  # кольцо замкнуто
```

---

## `prepend` — добавление в начало

Добавляет узел или значение в **начало** коллекции. Сложность: O(1).

### Для `SinglyLinkedList`

```nim
proc prepend*[T](L: var SinglyLinkedList[T], n: SinglyLinkedNode[T])
proc prepend*[T](L: var SinglyLinkedList[T], value: T)
```

**Пример:**
```nim
var a = initSinglyLinkedList[int]()
a.add(30)
a.prepend(20)
a.prepend(10)
assert $a == "[10, 20, 30]"
```

---

### Для `DoublyLinkedList`

```nim
proc prepend*[T](L: var DoublyLinkedList[T], n: DoublyLinkedNode[T])
proc prepend*[T](L: var DoublyLinkedList[T], value: T)
```

**Пример:**
```nim
var list = initDoublyLinkedList[int]()
let
  a = newDoublyLinkedNode[int](3)
  b = newDoublyLinkedNode[int](7)
  c = newDoublyLinkedNode[int](9)
list.add(a)
list.add(b)
list.prepend(c)
# Порядок: c → a → b
assert c.next == a
assert a.prev == c
assert c.prev == nil  # c — начало, нет предыдущего
```

---

### Для `SinglyLinkedRing`

```nim
proc prepend*[T](L: var SinglyLinkedRing[T], n: SinglyLinkedNode[T])
proc prepend*[T](L: var SinglyLinkedRing[T], value: T)
```

**Пример:**
```nim
var ring = initSinglyLinkedRing[int]()
ring.add(3)      # [3]
ring.add(7)      # [3, 7]
ring.prepend(9)  # [9, 3, 7]
assert ring.head.value == 9
assert ring.head.next.value == 3
assert ring.tail.next == ring.head  # кольцо
```

---

### Для `DoublyLinkedRing`

```nim
proc prepend*[T](L: var DoublyLinkedRing[T], n: DoublyLinkedNode[T])
proc prepend*[T](L: var DoublyLinkedRing[T], value: T)
```

**Пример:**
```nim
var ring = initDoublyLinkedRing[int]()
ring.add(2)
ring.add(3)
ring.prepend(1)  # [1, 2, 3]
assert ring.head.value == 1
```

---

## `add` конкатенация списков

Добавляет **поверхностную копию** списка `b` в конец списка `a`. Работает только для `SomeLinkedList` (не для колец).

```nim
proc add*[T: SomeLinkedList](a: var T, b: T)
```

**Описание:** `a` получает копию всех узлов `b`. Исходный список `b` не изменяется. Самоприсоединение (`a.add(a)`) поддерживается.

**Пример:**
```nim
from std/sequtils import toSeq
var a = [1, 2, 3].toSinglyLinkedList
let b = [4, 5].toSinglyLinkedList
a.add(b)
assert a.toSeq == [1, 2, 3, 4, 5]
assert b.toSeq == [4, 5]  # b не изменился

a.add(a)  # самоприсоединение — безопасно
assert a.toSeq == [1, 2, 3, 4, 5, 1, 2, 3, 4, 5]
```

---

## `addMoved` — конкатенация с перемещением

Перемещает список `b` в конец `a`. Сложность: O(1). После операции `b` становится пустым (если `a` и `b` — разные объекты).

### Для `SinglyLinkedList`

```nim
proc addMoved*[T](a, b: var SinglyLinkedList[T])
```

### Для `DoublyLinkedList`

```nim
proc addMoved*[T](a, b: var DoublyLinkedList[T])
```

**Пример:**
```nim
from std/sequtils import toSeq
var
  a = [1, 2, 3].toSinglyLinkedList
  b = [4, 5].toSinglyLinkedList
a.addMoved(b)
assert a.toSeq == [1, 2, 3, 4, 5]
assert b.toSeq == []  # b опустел
```

> **Самоприсоединение** (`a.addMoved(a)`) создаёт цикл — итерация по нему бесконечна. Используйте с осторожностью.

---

## `prepend` список + список

Добавляет **поверхностную копию** `b` в начало `a`. Сложность: O(|b|) из-за копирования. Работает только для `SomeLinkedList`.

```nim
proc prepend*[T: SomeLinkedList](a: var T, b: T)
```

**Пример:**
```nim
from std/sequtils import toSeq
var a = [4, 5].toSinglyLinkedList
let b = [1, 2, 3].toSinglyLinkedList
a.prepend(b)
assert a.toSeq == [1, 2, 3, 4, 5]
assert b.toSeq == [1, 2, 3]  # b не изменился

a.prepend(a)  # самоприсоединение поддерживается
assert a.toSeq == [1, 2, 3, 4, 5, 1, 2, 3, 4, 5]
```

---

## `prependMoved` — prepend с перемещением

Перемещает `b` в начало `a`. Сложность: O(1). После операции `b` становится пустым.

```nim
proc prependMoved*[T: SomeLinkedList](a, b: var T)
```

**Пример:**
```nim
from std/sequtils import toSeq
var
  a = [4, 5].toSinglyLinkedList
  b = [1, 2, 3].toSinglyLinkedList
a.prependMoved(b)
assert a.toSeq == [1, 2, 3, 4, 5]
assert b.toSeq == []  # b опустел
```

> **Самоприсоединение** (`a.prependMoved(a)`) создаёт цикл.

---

## `copy` — поверхностное копирование

Создаёт поверхностную копию списка. Работает только для `SomeLinkedList`.

```nim
func copy*[T](a: SinglyLinkedList[T]): SinglyLinkedList[T]
func copy*[T](a: DoublyLinkedList[T]): DoublyLinkedList[T]
```

**Описание:** Создаются новые узлы, но их `value`-поля указывают на те же объекты (для ref-типов). Для значимых типов (`int`, `string`) это эквивалентно полному копированию.

**Пример:**
```nim
from std/sequtils import toSeq

type Foo = ref object
  x: int

var f = Foo(x: 1)
var a = [f].toSinglyLinkedList
let b = a.copy
a.add([f].toSinglyLinkedList)

assert a.toSeq == [f, f]
assert b.toSeq == [f]  # b независим по структуре...
f.x = 42
assert b.head.value.x == 42  # ...но элементы разделяются
```

---

## `remove` — удаление узла

### Для `SinglyLinkedList`

```nim
proc remove*[T](L: var SinglyLinkedList[T], n: SinglyLinkedNode[T]): bool {.discardable.}
```

**Описание:** Удаляет узел `n` из списка `L`. Возвращает `true`, если узел был найден и удалён. Сложность: O(n) — список обходится до нахождения `n`. Попытка удалить отсутствующий узел — no-op. Если список является циклическим (после `addMoved(a, a)`), цикл сохраняется после удаления.

**Пример:**
```nim
from std/sequtils import toSeq
var a = [0, 1, 2].toSinglyLinkedList
let n = a.head.next  # узел со значением 1
assert n.value == 1
assert a.remove(n) == true
assert a.toSeq == [0, 2]
assert a.remove(n) == false  # уже удалён — no-op
```

---

### Для `DoublyLinkedList`

```nim
proc remove*[T](L: var DoublyLinkedList[T], n: DoublyLinkedNode[T])
```

**Описание:** Удаляет узел `n` из списка. Сложность: O(1) благодаря указателям `prev`/`next`. Предполагает, что `n` принадлежит `L`; иначе поведение не определено.

**Пример:**
```nim
from std/sequtils import toSeq
var a = initDoublyLinkedList[int]()
for i in 1..5:
  a.add(10 * i)
# Удаляем узел со значением 30, остальные умножаем через nodes
for x in nodes(a):
  if x.value == 30:
    a.remove(x)
  else:
    x.value = 5 * x.value - 1
assert $a == "[49, 99, 199, 249]"
```

---

### Для `DoublyLinkedRing`

```nim
proc remove*[T](L: var DoublyLinkedRing[T], n: DoublyLinkedNode[T])
```

**Описание:** Удаляет узел из кольца. Сложность: O(1). Предполагает принадлежность `n` к `L`. Если удаляется единственный элемент, `head` становится `nil`.

**Пример:**
```nim
var ring = initDoublyLinkedRing[int]()
let n = newDoublyLinkedNode[int](5)
ring.add(n)
assert 5 in ring
ring.remove(n)
assert 5 notin ring
assert ring.head == nil  # кольцо пусто
```

---

## `find` и `contains` — поиск

### `find`

```nim
proc find*[T](L: SomeLinkedCollection[T], value: T): SomeLinkedNode[T]
```

**Описание:** Ищет первый узел с заданным значением. Возвращает `nil`, если значение не найдено. Работает для всех четырёх типов коллекций. Сложность: O(n).

**Пример:**
```nim
let a = [9, 8].toSinglyLinkedList
assert a.find(9).value == 9
assert a.find(1) == nil
```

---

### `contains`

```nim
proc contains*[T](L: SomeLinkedCollection[T], value: T): bool
```

**Описание:** Возвращает `true`, если значение присутствует в коллекции. Используется с операторами `in` и `notin`. Реализован через `find`. Сложность: O(n).

**Пример:**
```nim
let a = [9, 8].toSinglyLinkedList
assert a.contains(9)
assert 8 in a
assert not a.contains(1)
assert 2 notin a
```

---

## `$` Строковое представление

```nim
proc `$`*[T](L: SomeLinkedCollection[T]): string
```

**Описание:** Преобразует любую коллекцию в строку вида `[e1, e2, e3]`. Вызывается автоматически при `echo`.

**Пример:**
```nim
let a = [1, 2, 3, 4].toSinglyLinkedList
assert $a == "[1, 2, 3, 4]"

var ring = initSinglyLinkedRing[int]()
ring.add(10)
ring.add(20)
assert $ring == "[10, 20]"
```

---

## Итераторы

### `items`

```nim
iterator items*[T](L: SomeLinkedList[T]): T
iterator items*[T](L: SomeLinkedRing[T]): T
```

**Описание:** Перебирает значения всех узлов. Для списков — от `head` до конца. Для колец — от `head` до возврата к `head` (полный цикл). Используется в `for x in L`.

**Пример:**
```nim
from std/sequtils import toSeq
let a = [10, 20, 30].toSinglyLinkedList
assert toSeq(a) == @[10, 20, 30]

var ring = initSinglyLinkedRing[int]()
for i in 1..3: ring.add(10 * i)
assert toSeq(ring) == @[10, 20, 30]
```

---

### `mitems`

```nim
iterator mitems*[T](L: var SomeLinkedList[T]): var T
iterator mitems*[T](L: var SomeLinkedRing[T]): var T
```

**Описание:** Перебирает значения с возможностью изменения. Список должен быть объявлен как `var`.

**Пример:**
```nim
var a = initSinglyLinkedList[int]()
for i in 1..5:
  a.add(10 * i)
assert $a == "[10, 20, 30, 40, 50]"
for x in mitems(a):
  x = 5 * x - 1
assert $a == "[49, 99, 149, 199, 249]"
```

---

### `nodes`

```nim
iterator nodes*[T](L: SomeLinkedList[T]): SomeLinkedNode[T]
iterator nodes*[T](L: SomeLinkedRing[T]): SomeLinkedNode[T]
```

**Описание:** Перебирает **узлы** (а не значения). Особенность: поддерживает удаление текущего узла во время обхода — безопасно по сравнению с `items`. Возвращает ref на узел, что позволяет изменять и `value`, и структуру списка.

**Пример (список):**
```nim
var a = initDoublyLinkedList[int]()
for i in 1..5:
  a.add(10 * i)
for x in nodes(a):
  if x.value == 30:
    a.remove(x)   # безопасно во время обхода
  else:
    x.value = 5 * x.value - 1
assert $a == "[49, 99, 199, 249]"
```

**Пример (кольцо):**
```nim
var ring = initDoublyLinkedRing[int]()
for i in 1..5:
  ring.add(10 * i)
for x in nodes(ring):
  if x.value == 30:
    ring.remove(x)
  else:
    x.value = 5 * x.value - 1
assert $ring == "[49, 99, 199, 249]"
```

---

## Преобразование из массива

Создают коллекции из `openArray[T]`, добавляя элементы по порядку через `add`.

### `toSinglyLinkedList`

```nim
func toSinglyLinkedList*[T](elems: openArray[T]): SinglyLinkedList[T]
```

**Пример:**
```nim
from std/sequtils import toSeq
let a = [1, 2, 3, 4, 5].toSinglyLinkedList
assert a.toSeq == [1, 2, 3, 4, 5]
```

---

### `toDoublyLinkedList`

```nim
func toDoublyLinkedList*[T](elems: openArray[T]): DoublyLinkedList[T]
```

**Пример:**
```nim
from std/sequtils import toSeq
let a = [1, 2, 3, 4, 5].toDoublyLinkedList
assert a.toSeq == [1, 2, 3, 4, 5]
assert a.head.value == 1
assert a.tail.value == 5
assert a.tail.prev.value == 4
```

---

### `toSinglyLinkedRing`

```nim
func toSinglyLinkedRing*[T](elems: openArray[T]): SinglyLinkedRing[T]
```

**Пример:**
```nim
from std/sequtils import toSeq
let a = [1, 2, 3, 4, 5].toSinglyLinkedRing
assert a.toSeq == [1, 2, 3, 4, 5]
assert a.tail.next == a.head  # кольцо замкнуто
```

---

### `toDoublyLinkedRing`

```nim
func toDoublyLinkedRing*[T](elems: openArray[T]): DoublyLinkedRing[T]
```

**Пример:**
```nim
from std/sequtils import toSeq
let a = [1, 2, 3, 4, 5].toDoublyLinkedRing
assert a.toSeq == [1, 2, 3, 4, 5]
assert a.head.prev.value == 5  # предыдущий от head — последний
```

---

## Псевдонимы `append` и `appendMoved`

### `append`

```nim
proc append*[T](a: var (SinglyLinkedList[T] | SinglyLinkedRing[T]),
                b: SinglyLinkedList[T] | SinglyLinkedNode[T] | T)

proc append*[T](a: var (DoublyLinkedList[T] | DoublyLinkedRing[T]),
                b: DoublyLinkedList[T] | DoublyLinkedNode[T] | T)
```

**Описание:** Псевдоним для `add`. Полностью идентичен по поведению.

---

### `appendMoved`

```nim
proc appendMoved*[T: SomeLinkedList](a, b: var T)
```

**Описание:** Псевдоним для `addMoved`. Полностью идентичен по поведению.

---

## Сводная таблица

| Функция | SinglyLinked List | DoublyLinked List | SinglyLinked Ring | DoublyLinked Ring | Сложность |
|---------|:-----------------:|:-----------------:|:-----------------:|:-----------------:|-----------|
| `init*` | ✓ | ✓ | ✓ | ✓ | O(1) |
| `newSinglyLinkedNode` | ✓ | — | ✓ | — | O(1) |
| `newDoublyLinkedNode` | — | ✓ | — | ✓ | O(1) |
| `add` (узел/значение) | ✓ | ✓ | ✓ | ✓ | O(1) |
| `prepend` (узел/значение) | ✓ | ✓ | ✓ | ✓ | O(1) |
| `add` (список+список) | ✓ | ✓ | — | — | O(\|b\|) |
| `addMoved` | ✓ | ✓ | — | — | O(1) |
| `prepend` (список+список) | ✓ | ✓ | — | — | O(\|b\|) |
| `prependMoved` | ✓ | ✓ | — | — | O(1) |
| `copy` | ✓ | ✓ | — | — | O(n) |
| `remove` | ✓ O(n) | ✓ O(1) | — | ✓ O(1) | — |
| `find` | ✓ | ✓ | ✓ | ✓ | O(n) |
| `contains` / `in` | ✓ | ✓ | ✓ | ✓ | O(n) |
| `$` | ✓ | ✓ | ✓ | ✓ | O(n) |
| `items` | ✓ | ✓ | ✓ | ✓ | — |
| `mitems` | ✓ | ✓ | ✓ | ✓ | — |
| `nodes` | ✓ | ✓ | ✓ | ✓ | — |
| `to*` (из массива) | ✓ | ✓ | ✓ | ✓ | O(n) |
| `append` / `appendMoved` | ✓ | ✓ | ✓ | ✓ | — |

---

## Комплексный пример

```nim
import std/lists
from std/sequtils import toSeq

# --- SinglyLinkedList: базовые операции ---
var slist = initSinglyLinkedList[int]()
for i in 1..5:
  slist.add(i * 10)
slist.prepend(0)
assert $slist == "[0, 10, 20, 30, 40, 50]"

# Поиск и удаление через nodes
for node in nodes(slist):
  if node.value == 30:
    slist.remove(node)
assert $slist == "[0, 10, 20, 40, 50]"

# --- DoublyLinkedList: двунаправленный доступ ---
var dlist = [1, 2, 3, 4, 5].toDoublyLinkedList
assert dlist.head.value == 1
assert dlist.tail.value == 5
assert dlist.tail.prev.value == 4

# Изменение значений через mitems
for x in mitems(dlist):
  x *= 2
assert $dlist == "[2, 4, 6, 8, 10]"

# --- Конкатенация ---
var a = [1, 2, 3].toSinglyLinkedList
var b = [4, 5].toSinglyLinkedList
a.addMoved(b)          # O(1), b становится пустым
assert a.toSeq == @[1, 2, 3, 4, 5]
assert b.toSeq == @[]

# --- SinglyLinkedRing: кольцевой обход ---
var ring = [1, 2, 3].toSinglyLinkedRing
assert ring.tail.next == ring.head  # замкнуто
assert ring.contains(2)

# --- DoublyLinkedRing: удаление из кольца ---
var dring = initDoublyLinkedRing[int]()
dring.add(10)
dring.add(20)
dring.add(30)
for node in nodes(dring):
  if node.value == 20:
    dring.remove(node)  # O(1)
assert $dring == "[10, 30]"
assert dring.head.prev.value == 30  # кольцо сохраняется
```
