# Справочник модуля `sets` (Nim)

> Модуль `sets` реализует эффективное **хеш-множество** (`HashSet`) и **упорядоченное хеш-множество** (`OrderedSet`).  
> В отличие от встроенного типа `set`, эти структуры позволяют хранить любые значения, поддерживающие хеширование, и не содержат дубликатов.

---

## Содержание

- [Типы данных](#типы-данных)
- [HashSet — инициализация](#hashset--инициализация)
- [HashSet — добавление элементов](#hashset--добавление-элементов)
- [HashSet — удаление элементов](#hashset--удаление-элементов)
- [HashSet — проверка и поиск](#hashset--проверка-и-поиск)
- [HashSet — размер и сравнение](#hashset--размер-и-сравнение)
- [HashSet — математические операции](#hashset--математические-операции)
- [HashSet — итерация и преобразование](#hashset--итерация-и-преобразование)
- [OrderedSet — инициализация](#orderedset--инициализация)
- [OrderedSet — добавление и удаление](#orderedset--добавление-и-удаление)
- [OrderedSet — проверка и поиск](#orderedset--проверка-и-поиск)
- [OrderedSet — размер и сравнение](#orderedset--размер-и-сравнение)
- [OrderedSet — итерация и преобразование](#orderedset--итерация-и-преобразование)
- [Устаревшие функции](#устаревшие-функции)

---

## Типы данных

### `HashSet[A]`

Обобщённое хеш-множество. **Не** сохраняет порядок вставки. Реализует семантику значений: оператор `=` выполняет копирование.

```nim
var s: HashSet[int]
```

### `OrderedSet[A]`

Обобщённое хеш-множество, **сохраняющее порядок вставки**.

```nim
var s: OrderedSet[string]
```

### `SomeSet[A]`

Объединение типов `HashSet[A] | OrderedSet[A]`. Используется для обобщённых процедур, принимающих любой вид множества.

### `defaultInitialSize`

Константа — начальный размер внутреннего буфера по умолчанию (`64`).

---

## HashSet — инициализация

### `init(s: var HashSet[A], initialSize = defaultInitialSize)`

Инициализирует хеш-множество. Начиная с Nim v0.20, множества инициализируются автоматически, поэтому явный вызов необходим лишь для **сброса** уже заполненного множества.

**Параметры:**
- `s` — инициализируемое множество (передаётся по ссылке `var`).
- `initialSize` — начальная ёмкость внутреннего буфера (должна быть степенью двойки).

```nim
var a: HashSet[int]
a.init()          # явная инициализация
a.incl(1)
a.incl(2)
a.init()          # сброс — все элементы удалены
assert a.len == 0
```

---

### `initHashSet[A](initialSize = defaultInitialSize): HashSet[A]`

Создаёт и возвращает новое пустое хеш-множество. Удобно для однострочного объявления переменной.

```nim
var a = initHashSet[int]()
a.incl(42)
assert a.len == 1

# С указанием начального размера
var b = initHashSet[string](128)
```

---

### `toHashSet[A](keys: openArray[A]): HashSet[A]`

Создаёт хеш-множество из массива, последовательности или строки. Дубликаты автоматически удаляются.

```nim
let a = toHashSet([5, 3, 2, 3, 5])
assert a.len == 3   # дубликаты отброшены
echo a              # {5, 3, 2} — порядок не гарантирован

let b = toHashSet("hello")
assert b.len == 4   # {'h', 'e', 'l', 'o'}
```

---

## HashSet — добавление элементов

### `incl(s: var HashSet[A], key: A)`

Добавляет элемент `key` в множество `s`. Если элемент уже присутствует, ничего не происходит.

```nim
var s = initHashSet[int]()
s.incl(10)
s.incl(10)   # повторный вызов безопасен
assert s.len == 1
```

---

### `incl(s: var HashSet[A], other: HashSet[A])`

Добавляет все элементы из множества `other` в `s`. Это операция объединения «на месте» (аналог `s = s + other`, но без копирования).

```nim
var a = toHashSet([1, 2, 3])
let b = toHashSet([3, 4, 5])
a.incl(b)
assert a.len == 5
echo a   # {1, 2, 3, 4, 5}
```

---

### `incl(s: var HashSet[A], other: OrderedSet[A])`

Добавляет все элементы из `OrderedSet` в `HashSet`.

```nim
var h = toHashSet([1, 2])
let o = toOrderedSet([2, 3, 4])
h.incl(o)
assert h.len == 4
```

---

### `containsOrIncl(s: var HashSet[A], key: A): bool`

Проверяет, есть ли `key` в `s`. Если **нет** — добавляет его и возвращает `false`. Если **есть** — возвращает `true` (и ничего не меняет).

Полезно для паттерна «добавь, если ещё не было».

```nim
var s = initHashSet[int]()
assert s.containsOrIncl(7) == false   # 7 добавлен
assert s.containsOrIncl(7) == true    # 7 уже был
assert s.containsOrIncl(8) == false   # 8 добавлен
assert s.len == 2
```

---

## HashSet — удаление элементов

### `excl(s: var HashSet[A], key: A)`

Удаляет элемент `key` из множества `s`. Если элемент отсутствует — ничего не происходит.

```nim
var s = toHashSet([1, 2, 3])
s.excl(2)
s.excl(99)   # безопасно — 99 не было
assert s.len == 2
```

---

### `excl(s: var HashSet[A], other: HashSet[A])`

Удаляет из `s` все элементы, присутствующие в `other`. Это операция разности «на месте» (аналог `s = s - other`).

```nim
var numbers = toHashSet([1, 2, 3, 4, 5])
let even    = toHashSet([2, 4, 6])
numbers.excl(even)
echo numbers   # {1, 3, 5}
```

---

### `missingOrExcl(s: var HashSet[A], key: A): bool`

Удаляет `key` из `s`. Возвращает `true`, если элемента **не было** (отсутствовал до вызова). Возвращает `false`, если элемент был и **был удалён**.

```nim
var s = toHashSet([2, 3, 6, 7])
assert s.missingOrExcl(4) == true    # 4 не было
assert s.missingOrExcl(6) == false   # 6 был и удалён
assert s.missingOrExcl(6) == true    # 6 уже отсутствует
```

---

### `pop(s: var HashSet[A]): A`

Удаляет и возвращает **произвольный** элемент из множества. Порядок не определён.  
Вызывает `KeyError`, если множество пусто.

```nim
var s = toHashSet([1, 2, 3])
let x = s.pop()   # x — один из {1, 2, 3}, порядок не гарантирован
assert s.len == 2

var empty: HashSet[int]
try:
  discard empty.pop()
except KeyError:
  echo "Множество пусто!"
```

---

### `clear(s: var HashSet[A])`

Очищает множество, удаляя все элементы, **не освобождая** выделенную память (буфер сохраняется). Сложность: O(n) по размеру внутреннего буфера.

```nim
var s = toHashSet([3, 5, 7])
s.clear()
assert s.len == 0
```

---

## HashSet — проверка и поиск

### `contains(s: HashSet[A], key: A): bool`

Возвращает `true`, если `key` присутствует в множестве. Поддерживает синтаксис операторов `in` / `notin`.

```nim
var s = toHashSet([10, 20, 30])
assert s.contains(20)
assert 10 in s
assert 99 notin s
```

---

### `s[key]` — оператор `[]`

Возвращает **ссылку** на элемент, хранящийся в множестве, который равен `key`. Полезно, когда переопределены `hash` и `==`, но требуется доступ к оригинальному объекту по ссылке.  
Вызывает `KeyError`, если элемент не найден.

```nim
type Point = object
  x, y: int

var s: HashSet[Point]
s.incl(Point(x: 1, y: 2))
let p = s[Point(x: 1, y: 2)]
```

---

## HashSet — размер и сравнение

### `len(s: HashSet[A]): int`

Возвращает количество элементов в множестве. Безопасен для вызова на неинициализированном множестве (вернёт 0).

```nim
let s = toHashSet([1, 2, 3])
assert s.len == 3

var empty: HashSet[string]
assert empty.len == 0
```

---

### `card(s: HashSet[A]): int`

Псевдоним для `len`. Термин «кардинальность» используется в теории множеств.

```nim
let s = toHashSet(["a", "b", "c"])
assert s.card == 3
```

---

### `==(s, t: HashSet[A]): bool`

Возвращает `true`, если оба множества содержат одинаковые элементы (порядок не важен для `HashSet`).

```nim
let a = toHashSet([1, 2, 3])
let b = toHashSet([3, 1, 2])
assert a == b
```

---

### `<(s, t: HashSet[A]): bool`

Возвращает `true`, если `s` — **строгое (собственное) подмножество** `t`: все элементы `s` входят в `t`, но `t` содержит хотя бы один дополнительный элемент.

```nim
let a = toHashSet([1, 2])
let b = toHashSet([1, 2, 3])
assert a < b
assert not (b < a)
assert not (a < a)
```

---

### `<=(s, t: HashSet[A]): bool`

Возвращает `true`, если `s` — **подмножество** `t` (включая случай равенства).

```nim
let a = toHashSet([1, 2])
let b = toHashSet([1, 2, 3])
assert a <= b
assert a <= a   # множество — подмножество самого себя
```

---

### `disjoint(s1, s2: HashSet[A]): bool`

Возвращает `true`, если множества `s1` и `s2` **не имеют общих элементов**.

```nim
let a = toHashSet([1, 2])
let b = toHashSet([3, 4])
let c = toHashSet([2, 3])
assert disjoint(a, b) == true
assert disjoint(a, c) == false
```

---

## HashSet — математические операции

### `union(s1, s2: HashSet[A]): HashSet[A]`  /  оператор `+`

Возвращает **объединение** множеств (*A ∪ B*) — все элементы, принадлежащие хотя бы одному из множеств.

```nim
let a = toHashSet([1, 2, 3])
let b = toHashSet([3, 4, 5])
echo union(a, b)   # {1, 2, 3, 4, 5}
echo a + b         # то же самое
```

---

### `intersection(s1, s2: HashSet[A]): HashSet[A]`  /  оператор `*`

Возвращает **пересечение** множеств (*A ∩ B*) — элементы, присутствующие в обоих множествах.

```nim
let a = toHashSet([1, 2, 3])
let b = toHashSet([2, 3, 4])
echo intersection(a, b)   # {2, 3}
echo a * b                # то же самое
```

---

### `difference(s1, s2: HashSet[A]): HashSet[A]`  /  оператор `-`

Возвращает **разность** множеств (*A ∖ B*) — элементы `s1`, не входящие в `s2`.

```nim
let a = toHashSet([1, 2, 3, 4])
let b = toHashSet([3, 4, 5])
echo difference(a, b)   # {1, 2}
echo a - b              # то же самое
```

---

### `symmetricDifference(s1, s2: HashSet[A]): HashSet[A]`  /  оператор `-+-`

Возвращает **симметрическую разность** (*A △ B*) — элементы, принадлежащие ровно одному из множеств (но не обоим).

```nim
let a = toHashSet([1, 2, 3])
let b = toHashSet([2, 3, 4])
echo symmetricDifference(a, b)   # {1, 4}
echo a -+- b                     # то же самое
```

---

## HashSet — итерация и преобразование

### `items(s: HashSet[A]): A`  (итератор)

Итерирует по всем элементам множества. Порядок обхода **не определён**. Запрещено изменять множество во время итерации.

```nim
let s = toHashSet([10, 20, 30])
for x in s:
  echo x

# Преобразование в последовательность:
import sequtils
let sq = toSeq(s.items)
```

---

### `map[A, B](data: HashSet[A], op: proc(x: A): B): HashSet[B]`

Применяет функцию `op` к каждому элементу множества и возвращает новое множество из результатов.

```nim
let a = toHashSet([1, 2, 3])
let b = a.map(proc(x: int): string = $x)
assert b == toHashSet(["1", "2", "3"])

# С лямбдой
let doubled = a.map(proc(x: int): int = x * 2)
echo doubled   # {2, 4, 6}
```

---

### `hash(s: HashSet[A]): Hash`

Вычисляет хеш всего множества. Используется при хранении `HashSet` как ключа в другом хеш-контейнере.

---

### `$(s: HashSet[A]): string`

Преобразует множество в строку для вывода. **Не используйте для сериализации** — формат может измениться.

```nim
echo toHashSet([2, 4, 5])
# --> {2, 4, 5}
echo toHashSet(["hello", "world"])
# --> {hello, world}
```

---

## OrderedSet — инициализация

### `init(s: var OrderedSet[A], initialSize = defaultInitialSize)`

Инициализирует упорядоченное множество. При повторном вызове сбрасывает содержимое.

```nim
var a: OrderedSet[int]
a.init()
```

---

### `initOrderedSet[A](initialSize = defaultInitialSize): OrderedSet[A]`

Создаёт и возвращает новое пустое упорядоченное множество.

```nim
var a = initOrderedSet[string]()
a.incl("первый")
a.incl("второй")
```

---

### `toOrderedSet[A](keys: openArray[A]): OrderedSet[A]`

Создаёт упорядоченное множество из массива/последовательности/строки. **Порядок элементов сохраняется**, дубликаты удаляются.

```nim
let a = toOrderedSet([9, 5, 1])
echo a   # {9, 5, 1} — порядок вставки сохранён

let b = toOrderedSet("abracadabra")
echo b   # {a, b, r, c, d}
```

---

## OrderedSet — добавление и удаление

### `incl(s: var OrderedSet[A], key: A)`

Добавляет `key` в конец упорядоченного множества. Если элемент уже есть — ничего не происходит.

```nim
var s = initOrderedSet[int]()
s.incl(3)
s.incl(1)
s.incl(2)
s.incl(1)   # дубликат, игнорируется
echo s      # {3, 1, 2}
```

---

### `containsOrIncl(s: var OrderedSet[A], key: A): bool`

Аналог для `OrderedSet`: добавляет `key`, если его нет, возвращает `false`. Если уже есть — возвращает `true`.

```nim
var s = initOrderedSet[int]()
assert s.containsOrIncl(5) == false
assert s.containsOrIncl(5) == true
```

---

### `excl(s: var OrderedSet[A], key: A)`

Удаляет `key` из упорядоченного множества. Сложность: **O(n)** — из-за перестройки связного списка порядка. Если элемент отсутствует — ничего не происходит.

```nim
var s = toOrderedSet([2, 3, 6, 7])
s.excl(3)
echo s   # {2, 6, 7}
```

---

### `missingOrExcl(s: var OrderedSet[A], key: A): bool`

Удаляет `key`. Возвращает `true` если элемента не было, `false` если был и удалён. Сложность: O(n).

```nim
var s = toOrderedSet([2, 3, 6, 7])
assert s.missingOrExcl(4) == true    # 4 не было
assert s.missingOrExcl(6) == false   # 6 был и удалён
```

---

### `clear(s: var OrderedSet[A])`

Очищает упорядоченное множество без освобождения памяти. Сложность: O(n).

```nim
var s = toOrderedSet([1, 2, 3])
s.clear()
assert s.len == 0
```

---

## OrderedSet — проверка и поиск

### `contains(s: OrderedSet[A], key: A): bool`

Возвращает `true`, если `key` присутствует. Поддерживает операторы `in` / `notin`.

```nim
var s = toOrderedSet([10, 20, 30])
assert 20 in s
assert 99 notin s
```

---

## OrderedSet — размер и сравнение

### `len(s: OrderedSet[A]): int`

Возвращает количество элементов. Безопасен для вызова на неинициализированном множестве.

```nim
let s = toOrderedSet([1, 2, 3])
assert s.len == 3
```

---

### `card(s: OrderedSet[A]): int`

Псевдоним для `len`.

---

### `==(s, t: OrderedSet[A]): bool`

Сравнивает два упорядоченных множества. Учитывает **и состав, и порядок** элементов.

```nim
let a = toOrderedSet([1, 2, 3])
let b = toOrderedSet([1, 2, 3])
let c = toOrderedSet([3, 2, 1])
assert a == b
assert a != c   # разный порядок!
```

---

### `hash(s: OrderedSet[A]): Hash`

Вычисляет хеш упорядоченного множества с учётом порядка элементов.

---

### `$(s: OrderedSet[A]): string`

Преобразует упорядоченное множество в строку. Порядок элементов сохраняется.

```nim
echo toOrderedSet([9, 5, 1])
# --> {9, 5, 1}
```

---

## OrderedSet — итерация и преобразование

### `items(s: OrderedSet[A]): A`  (итератор)

Итерирует по элементам **в порядке вставки**. Запрещено изменять множество во время итерации.

```nim
var s = initOrderedSet[int]()
for v in [9, 2, 1, 5, 8]:
  s.incl(v)

for x in s:
  echo x   # 9, 2, 1, 5, 8 — в порядке добавления
```

---

### `pairs(s: OrderedSet[A]): tuple[a: int, b: A]`  (итератор)

Итерирует по парам `(позиция, значение)` в порядке вставки.

```nim
let s = toOrderedSet("abcde")
for idx, ch in s.pairs:
  echo idx, ": ", ch
# 0: a
# 1: b
# 2: c
# 3: d
# 4: e
```

---

## Устаревшие функции

Следующие функции помечены как **deprecated** и не должны использоваться в новом коде:

| Устаревшее имя | Замена |
|---|---|
| `initSet[A]()` | `initHashSet[A]()` |
| `toSet[A](keys)` | `toHashSet[A](keys)` |
| `isValid(s: HashSet[A])` | не нужно (инициализация автоматическая с v0.20) |

---

## Краткая шпаргалка по операторам

| Оператор | Описание | Пример |
|---|---|---|
| `+` | Объединение | `a + b` |
| `*` | Пересечение | `a * b` |
| `-` | Разность | `a - b` |
| `-+-` | Симметрическая разность | `a -+- b` |
| `==` | Равенство | `a == b` |
| `<` | Строгое подмножество | `a < b` |
| `<=` | Подмножество | `a <= b` |
| `in` | Принадлежность | `x in s` |
| `notin` | Непринадлежность | `x notin s` |

---

## Примечания

- Оба типа реализуют **семантику значений**: присваивание `=` создаёт полную копию.
- `HashSet` не гарантирует порядок обхода; `OrderedSet` обходится в порядке вставки.
- `excl` для `OrderedSet` имеет сложность **O(n)**, тогда как для `HashSet` — амортизированно **O(1)**.
- Не изменяйте множество во время итерации — это приведёт к ошибке assertion.
