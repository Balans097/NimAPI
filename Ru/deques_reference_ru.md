# Справочник модуля `deques` (Nim)

> Модуль `deques` реализует **двустороннюю очередь** (deque, double-ended queue) на основе кольцевого буфера (`seq`).  
> Элементы можно эффективно добавлять и удалять **как с начала, так и с конца** очереди.

> ⚠️ **Внимание:** никогда не вызывайте процедуры чтения/удаления элементов на пустом deque. При включённых проверках границ (`boundChecks`) будет поднято исключение `IndexDefect`; при отключённых (`-d:danger`) — неопределённое поведение.

---

## Содержание

- [Тип данных](#тип-данных)
- [Инициализация](#инициализация)
- [Добавление элементов](#добавление-элементов)
- [Просмотр элементов (без удаления)](#просмотр-элементов-без-удаления)
- [Удаление элементов](#удаление-элементов)
- [Доступ по индексу](#доступ-по-индексу)
- [Размер и очистка](#размер-и-очистка)
- [Поиск](#поиск)
- [Итерация](#итерация)
- [Сравнение и прочее](#сравнение-и-прочее)
- [Краткая шпаргалка](#краткая-шпаргалка)

---

## Тип данных

### `Deque[T]`

Двусторонняя очередь с внутренним кольцевым буфером на основе `seq[T]`. Поддерживает добавление и удаление элементов с обоих концов за амортизированное O(1). Имеет **семантику значений**: оператор `=` создаёт полную копию.

```nim
var d: Deque[int]
```

### `defaultInitialSize`

Константа — начальная ёмкость буфера по умолчанию (`4`).

---

## Инициализация

### `initDeque[T](initialSize: int = defaultInitialSize): Deque[T]`

Создаёт и возвращает новый пустой deque. Параметр `initialSize` задаёт начальную ёмкость буфера (оптимизация производительности); длина нового deque всегда равна 0. Ёмкость округляется до ближайшей степени двойки.

```nim
var a = initDeque[int]()
var b = initDeque[string](32)   # с предварительным резервированием

a.addLast(1)
a.addLast(2)
assert a.len == 2
```

---

### `toDeque[T](x: openArray[T]): Deque[T]`

Создаёт новый deque из массива или последовательности `x`. Порядок элементов сохраняется.

> Доступно начиная с Nim v1.3.

```nim
let a = toDeque([10, 20, 30, 40])
assert a.len == 4
assert $a == "[10, 20, 30, 40]"

# Из последовательности
let b = @[1, 2, 3]
let d = toDeque(b)
```

---

## Добавление элементов

### `addFirst[T](deq: var Deque[T], item: sink T)`

Добавляет элемент `item` в **начало** deque. При необходимости буфер автоматически расширяется. Амортизированная сложность: O(1).

```nim
var a = initDeque[int]()
a.addFirst(10)
a.addFirst(20)
a.addFirst(30)
assert $a == "[30, 20, 10]"   # последний добавленный — первый в очереди
```

---

### `addLast[T](deq: var Deque[T], item: sink T)`

Добавляет элемент `item` в **конец** deque. При необходимости буфер автоматически расширяется. Амортизированная сложность: O(1).

```nim
var a = initDeque[int]()
a.addLast(10)
a.addLast(20)
a.addLast(30)
assert $a == "[10, 20, 30]"   # порядок вставки сохранён
```

---

## Просмотр элементов (без удаления)

### `peekFirst[T](deq: Deque[T]): lent T`

Возвращает **первый** элемент deque, **не удаляя** его. Вызывает `IndexDefect` на пустом deque.

```nim
let a = [10, 20, 30].toDeque
assert a.peekFirst == 10
assert a.len == 3   # deque не изменился
```

---

### `peekFirst[T](deq: var Deque[T]): var T`

Возвращает **изменяемую ссылку** на первый элемент. Позволяет модифицировать элемент на месте без его удаления.

> Доступно начиная с Nim v1.3.

```nim
var a = [10, 20, 30].toDeque
a.peekFirst() = 99
assert $a == "[99, 20, 30]"

inc(a.peekFirst())
assert $a == "[100, 20, 30]"
```

---

### `peekLast[T](deq: Deque[T]): lent T`

Возвращает **последний** элемент deque, **не удаляя** его. Вызывает `IndexDefect` на пустом deque.

```nim
let a = [10, 20, 30].toDeque
assert a.peekLast == 30
assert a.len == 3
```

---

### `peekLast[T](deq: var Deque[T]): var T`

Возвращает **изменяемую ссылку** на последний элемент.

> Доступно начиная с Nim v1.3.

```nim
var a = [10, 20, 30].toDeque
a.peekLast() = 99
assert $a == "[10, 20, 99]"
```

---

## Удаление элементов

### `popFirst[T](deq: var Deque[T]): T`

Удаляет **первый** элемент из deque и возвращает его. Вызывает `IndexDefect` на пустом deque. Сложность: O(1).

```nim
var a = [10, 20, 30, 40].toDeque
assert a.popFirst == 10
assert $a == "[20, 30, 40]"

# Использование в цикле для опустошения очереди:
while a.len > 0:
  echo a.popFirst
```

---

### `popLast[T](deq: var Deque[T]): T`

Удаляет **последний** элемент из deque и возвращает его. Вызывает `IndexDefect` на пустом deque. Сложность: O(1).

```nim
var a = [10, 20, 30, 40].toDeque
assert a.popLast == 40
assert $a == "[10, 20, 30]"
```

---

### `clear[T](deq: var Deque[T])`

Сбрасывает deque в пустое состояние, уничтожая все элементы. Внутренний буфер **сохраняет** выделенную память. Сложность: O(n).

```nim
var a = [10, 20, 30, 40, 50].toDeque
assert a.len == 5
a.clear()
assert a.len == 0
```

---

### `shrink[T](deq: var Deque[T], fromFirst = 0, fromLast = 0)`

Удаляет `fromFirst` элементов с **начала** и `fromLast` элементов с **конца** deque за один вызов. Если суммарное количество удаляемых элементов превышает длину deque, очередь полностью опустошается.

```nim
var a = [10, 20, 30, 40, 50].toDeque
a.shrink(fromFirst = 2, fromLast = 1)
assert $a == "[30, 40]"

# Обрезать больше, чем есть — безопасно:
var b = [1, 2, 3].toDeque
b.shrink(fromFirst = 10, fromLast = 10)
assert b.len == 0
```

---

## Доступ по индексу

### `deq[i: Natural]: lent T`

Возвращает элемент по прямому индексу `i` (от 0). Вызывает `IndexDefect` при выходе за границы.

```nim
let a = [10, 20, 30, 40, 50].toDeque
assert a[0] == 10
assert a[4] == 50
```

---

### `deq[i: Natural]: var T`  (для `var Deque`)

Возвращает **изменяемую ссылку** на элемент по индексу `i`. Позволяет менять элемент на месте.

```nim
var a = [10, 20, 30].toDeque
inc(a[0])
a[2] = 99
assert $a == "[11, 20, 99]"
```

---

### `deq[i] = val` — оператор `[]=` (прямой индекс)

Устанавливает элемент по индексу `i` равным `val`.

```nim
var a = [10, 20, 30, 40, 50].toDeque
a[0] = 99
a[3] = 66
assert $a == "[99, 20, 30, 66, 50]"
```

---

### `deq[^i: BackwardsIndex]: lent T`

Возвращает элемент по **обратному** индексу: `deq[^1]` — последний элемент, `deq[^2]` — предпоследний и т.д.

```nim
let a = [10, 20, 30, 40, 50].toDeque
assert a[^1] == 50
assert a[^2] == 40
assert a[^5] == 10
```

---

### `deq[^i: BackwardsIndex]: var T`  (для `var Deque`)

Возвращает **изменяемую ссылку** по обратному индексу.

```nim
var a = [10, 20, 30, 40, 50].toDeque
inc(a[^1])
assert a[^1] == 51
```

---

### `deq[^i] = val` — оператор `[]=` (обратный индекс)

Устанавливает элемент по обратному индексу равным `val`.

```nim
var a = [10, 20, 30, 40, 50].toDeque
a[^1] = 99
a[^3] = 77
assert $a == "[10, 20, 77, 40, 99]"
```

---

## Размер и очистка

### `len[T](deq: Deque[T]): int`

Возвращает текущее количество элементов в deque. Сложность: O(1).

```nim
var a = [1, 2, 3].toDeque
assert a.len == 3
a.addLast(4)
assert a.len == 4
a.popFirst()
assert a.len == 3
```

---

## Поиск

### `contains[T](deq: Deque[T], item: T): bool`

Возвращает `true`, если `item` присутствует в deque. Линейный поиск, сложность: O(n). Поддерживает синтаксис операторов `in` / `notin`.

```nim
let q = [7, 9, 13].toDeque
assert 9 in q
assert q.contains(7)
assert 5 notin q
```

---

## Итерация

### `items[T](deq: Deque[T]): lent T`  (итератор)

Перебирает все элементы deque **слева направо** (от начала к концу). Возвращает неизменяемые ссылки. Запрещено изменять deque во время итерации.

```nim
let a = [10, 20, 30].toDeque
for x in a:
  echo x   # 10, 20, 30

# Преобразование в последовательность:
from std/sequtils import toSeq
assert toSeq(a.items) == @[10, 20, 30]
```

---

### `mitems[T](deq: var Deque[T]): var T`  (итератор)

Перебирает все элементы, возвращая **изменяемые** ссылки. Позволяет модифицировать элементы непосредственно в цикле.

```nim
var a = [10, 20, 30, 40, 50].toDeque
for x in mitems(a):
  x = x * 2
assert $a == "[20, 40, 60, 80, 100]"
```

---

### `pairs[T](deq: Deque[T]): tuple[key: int, val: T]`  (итератор)

Перебирает пары `(индекс, значение)` в порядке от начала к концу.

```nim
let a = [10, 20, 30].toDeque
for idx, val in a.pairs:
  echo idx, ": ", val
# 0: 10
# 1: 20
# 2: 30

from std/sequtils import toSeq
assert toSeq(a.pairs) == @[(0, 10), (1, 20), (2, 30)]
```

---

## Сравнение и прочее

### `==[T](deq1, deq2: Deque[T]): bool`

Возвращает `true`, если оба deque содержат одинаковые элементы **в одинаковом порядке**.

```nim
var a, b = initDeque[int]()
a.addFirst(2); a.addFirst(1)   # a = [1, 2]
b.addLast(1);  b.addLast(2)    # b = [1, 2]
assert a == b

let c = [1, 2, 3].toDeque
assert a != c
```

---

### `hash[T](deq: Deque[T]): Hash`

Вычисляет хеш deque с учётом порядка элементов. Используется при хранении deque как ключа в хеш-таблице или хеш-множестве.

```nim
import std/tables
var t: Table[Deque[int], string]
t[[1, 2, 3].toDeque] = "abc"
```

---

### `$[T](deq: Deque[T]): string`

Преобразует deque в строковое представление вида `[e1, e2, e3]`. Используется для вывода и отладки. **Не применяйте для сериализации**.

```nim
let a = [10, 20, 30].toDeque
echo $a   # [10, 20, 30]

let b = ["hello", "world"].toDeque
echo $b   # ["hello", "world"]
```

---

## Краткая шпаргалка

| Операция | Процедура / оператор | Сложность |
|---|---|---|
| Создать пустой | `initDeque[T]()` | O(1) |
| Создать из массива | `toDeque(arr)` | O(n) |
| Добавить в начало | `addFirst(item)` | O(1) амор. |
| Добавить в конец | `addLast(item)` | O(1) амор. |
| Прочитать первый | `peekFirst()` | O(1) |
| Прочитать последний | `peekLast()` | O(1) |
| Удалить первый | `popFirst()` | O(1) |
| Удалить последний | `popLast()` | O(1) |
| Доступ по индексу | `deq[i]`, `deq[^i]` | O(1) |
| Изменить по индексу | `deq[i] = val` | O(1) |
| Количество элементов | `len()` | O(1) |
| Поиск | `contains()` / `in` | O(n) |
| Очистить всё | `clear()` | O(n) |
| Обрезать с концов | `shrink(fromFirst, fromLast)` | O(k) |
| Равенство | `==` | O(n) |
| Хеш | `hash()` | O(n) |
| Строка | `$` | O(n) |
| Итерация (чтение) | `items` / `for x in deq` | O(n) |
| Итерация (запись) | `mitems` | O(n) |
| Итерация с индексом | `pairs` | O(n) |

### Типичные паттерны использования

```nim
# FIFO-очередь (первым вошёл — первым вышел):
var queue = initDeque[int]()
queue.addLast(1)
queue.addLast(2)
queue.addLast(3)
echo queue.popFirst   # 1
echo queue.popFirst   # 2

# LIFO-стек (первым вошёл — последним вышел):
var stack = initDeque[int]()
stack.addLast(1)
stack.addLast(2)
stack.addLast(3)
echo stack.popLast   # 3
echo stack.popLast   # 2

# Скользящее окно фиксированного размера:
const windowSize = 3
var window = initDeque[int]()
for value in [1, 2, 3, 4, 5, 6]:
  window.addLast(value)
  if window.len > windowSize:
    discard window.popFirst
  echo $window
```

### Примечания

- Внутренний буфер автоматически **удваивается** при переполнении (удваивание гарантирует степень двойки).
- `clear` **не освобождает** память буфера — повторное добавление элементов не требует перераспределения.
- Не изменяйте deque во время итерации через `items`, `mitems` или `pairs` — это приведёт к ошибке assertion.
- `popFirst` / `popLast` помечены как `discardable`: результат можно игнорировать, если нужно просто удалить элемент.
