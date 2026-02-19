# Справочник модуля `std/tables` (Nim)

Модуль реализует три варианта хэш-таблиц (словарей):

| Тип | Ref-версия | Особенность |
|-----|-----------|-------------|
| `Table[A, B]` | `TableRef[A, B]` | Обычная хэш-таблица |
| `OrderedTable[A, B]` | `OrderedTableRef[A, B]` | Запоминает порядок вставки |
| `CountTable[A]` | `CountTableRef[A]` | Считает количество вхождений ключей |

**Семантика значений vs ссылок.** Типы без суффикса `Ref` имеют *семантику значений*: `=` создаёт независимую копию. `Ref`-типы — ссылочные: присваивание разделяет одни и те же данные между переменными.

**Пользовательские ключи.** Для использования объектов как ключей необходимо определить процедуру `hash(x: T): Hash` из модуля `std/hashes`.

---

## Содержание

- [Table — обычная хэш-таблица](#table--обычная-хэш-таблица)
- [TableRef — ссылочная таблица](#tableref--ссылочная-таблица)
- [OrderedTable — таблица с порядком](#orderedtable--таблица-с-порядком)
- [OrderedTableRef — ссылочная таблица с порядком](#orderedtableref--ссылочная-таблица-с-порядком)
- [CountTable — таблица счётчиков](#counttable--таблица-счётчиков)
- [CountTableRef — ссылочная таблица счётчиков](#counttableref--ссылочная-таблица-счётчиков)
- [Общие итераторы](#общие-итераторы)
- [Вспомогательные функции](#вспомогательные-функции)
- [Сводная таблица функций](#сводная-таблица-функций)

---

## Table — обычная хэш-таблица

### `initTable`

```nim
proc initTable*[A, B](initialSize = defaultInitialSize): Table[A, B]
```

**Описание:** Создаёт новую пустую хэш-таблицу. Начиная с Nim v0.20 таблицы инициализируются по умолчанию, явный вызов необязателен. `initialSize` должен быть степенью двойки.

**Пример:**
```nim
let
  a = initTable[int, string]()
  b = initTable[char, seq[int]]()
```

---

### `toTable`

```nim
proc toTable*[A, B](pairs: openArray[(A, B)]): Table[A, B]
```

**Описание:** Создаёт таблицу из набора пар `(ключ, значение)`.

**Пример:**
```nim
let a = [('a', 5), ('b', 9)]
let b = toTable(a)
assert b == {'a': 5, 'b': 9}.toTable
```

---

### `[]=` (вставка)

```nim
proc `[]=`*[A, B](t: var Table[A, B], key: A, val: sink B)
```

**Описание:** Вставляет пару `(ключ, значение)` в таблицу. Если ключ уже существует, значение перезаписывается.

**Пример:**
```nim
var a = initTable[char, int]()
a['x'] = 7
a['y'] = 33
assert a == {'x': 7, 'y': 33}.toTable
```

---

### `[]` (чтение)

```nim
proc `[]`*[A, B](t: Table[A, B], key: A): lent B
proc `[]`*[A, B](t: var Table[A, B], key: A): var B
```

**Описание:** Возвращает значение по ключу. Вторая версия возвращает изменяемую ссылку. Если ключ отсутствует — выбрасывается `KeyError`.

**Пример:**
```nim
let a = {'a': 5, 'b': 9}.toTable
assert a['a'] == 5
# a['z']  # выбросит KeyError
```

---

### `hasKey`

```nim
proc hasKey*[A, B](t: Table[A, B], key: A): bool
```

**Описание:** Возвращает `true`, если ключ присутствует в таблице.

**Пример:**
```nim
let a = {'a': 5, 'b': 9}.toTable
assert a.hasKey('a') == true
assert a.hasKey('z') == false
```

---

### `contains`

```nim
proc contains*[A, B](t: Table[A, B], key: A): bool
```

**Описание:** Псевдоним `hasKey` для использования с оператором `in`.

**Пример:**
```nim
let a = {'a': 5, 'b': 9}.toTable
assert 'b' in a
assert 'z' notin a
```

---

### `hasKeyOrPut`

```nim
proc hasKeyOrPut*[A, B](t: var Table[A, B], key: A, val: B): bool
```

**Описание:** Если ключ существует — возвращает `true`. Если не существует — вставляет `(key, val)` и возвращает `false`. Удобен для паттерна «вставить, если отсутствует».

**Пример:**
```nim
var a = {'a': 5, 'b': 9}.toTable
if a.hasKeyOrPut('a', 50):
  a['a'] = 99       # 'a' уже был, обновляем вручную
if a.hasKeyOrPut('z', 50):
  a['z'] = 99       # 'z' не был, этот блок не выполнится
assert a == {'a': 99, 'b': 9, 'z': 50}.toTable
```

---

### `getOrDefault`

```nim
proc getOrDefault*[A, B](t: Table[A, B], key: A): B
proc getOrDefault*[A, B](t: Table[A, B], key: A, def: B): B
```

**Описание:** Возвращает значение по ключу. Первая версия при отсутствии ключа возвращает нулевое значение типа `B` (0 для `int`, `""` для `string` и т.д.). Вторая версия возвращает явно переданное значение `def`.

**Пример:**
```nim
let a = {'a': 5, 'b': 9}.toTable
assert a.getOrDefault('a') == 5
assert a.getOrDefault('z') == 0       # нулевое значение int
assert a.getOrDefault('a', 99) == 5
assert a.getOrDefault('z', 99) == 99  # пользовательское значение
```

---

### `mgetOrPut`

```nim
proc mgetOrPut*[A, B](t: var Table[A, B], key: A, val: B): var B
proc mgetOrPut*[A, B](t: var Table[A, B], key: A): var B
```

**Описание:** Возвращает изменяемую ссылку на значение. Если ключ отсутствует — вставляет `val` (или нулевое значение) и возвращает ссылку на него. Позволяет модифицировать значение непосредственно через результат вызова.

> **Предупреждение:** Присвоение результата в отдельную переменную создаёт копию, а не ссылку. Для `seq` и строк изменяйте результат прямо «на месте».

**Пример:**
```nim
var a = {'a': 5, 'b': 9}.toTable
assert a.mgetOrPut('a', 99) == 5     # 'a' существует
assert a.mgetOrPut('z', 99) == 99    # 'z' вставляется
assert a == {'a': 5, 'b': 9, 'z': 99}.toTable

# Правильный способ изменить seq-значение:
var t = initTable[int, seq[int]]()
t.mgetOrPut(25, @[25]).add(35)
assert t[25] == @[25, 35]
```

---

### `len`

```nim
proc len*[A, B](t: Table[A, B]): int
```

**Описание:** Возвращает количество ключей в таблице.

**Пример:**
```nim
let a = {'a': 5, 'b': 9}.toTable
assert len(a) == 2
```

---

### `del`

```nim
proc del*[A, B](t: var Table[A, B], key: A)
```

**Описание:** Удаляет ключ из таблицы. Если ключ отсутствует — ничего не происходит.

**Пример:**
```nim
var a = {'a': 5, 'b': 9, 'c': 13}.toTable
a.del('a')
assert a == {'b': 9, 'c': 13}.toTable
a.del('z')  # нет ошибки
```

---

### `pop`

```nim
proc pop*[A, B](t: var Table[A, B], key: A, val: var B): bool
```

**Описание:** Удаляет ключ и записывает его значение в `val`. Возвращает `true` если ключ был найден, иначе `false`, а `val` остаётся неизменным.

**Пример:**
```nim
var a = {'a': 5, 'b': 9, 'c': 13}.toTable
var i: int
assert a.pop('b', i) == true
assert i == 9
assert a.pop('z', i) == false  # i не изменился
```

---

### `take`

```nim
proc take*[A, B](t: var Table[A, B], key: A, val: var B): bool
```

**Описание:** Псевдоним для `pop`.

---

### `clear`

```nim
proc clear*[A, B](t: var Table[A, B])
```

**Описание:** Полностью очищает таблицу.

**Пример:**
```nim
var a = {'a': 5, 'b': 9, 'c': 13}.toTable
clear(a)
assert len(a) == 0
```

---

### `$` (строковое представление)

```nim
proc `$`*[A, B](t: Table[A, B]): string
```

**Описание:** Преобразует таблицу в строку. Вызывается автоматически при `echo`.

**Пример:**
```nim
let a = {'a': 5, 'b': 9}.toTable
echo a  # {'a': 5, 'b': 9}
```

---

### `==`

```nim
proc `==`*[A, B](s, t: Table[A, B]): bool
```

**Описание:** Возвращает `true`, если таблицы содержат одинаковые пары ключ-значение. Порядок вставки не имеет значения.

**Пример:**
```nim
let
  a = {'a': 5, 'b': 9, 'c': 13}.toTable
  b = {'b': 9, 'c': 13, 'a': 5}.toTable
assert a == b
```

---

### `indexBy`

```nim
proc indexBy*[A, B, C](collection: A, index: proc(x: B): C): Table[C, B]
```

**Описание:** Индексирует коллекцию по ключу, возвращаемому функцией `index`. Позволяет быстро превратить список объектов в словарь.

**Пример:**
```nim
type User = object
  name: string
  age: int

let users = @[User(name: "Alice", age: 30), User(name: "Bob", age: 25)]
let byName = users.indexBy(proc(u: User): string = u.name)
assert byName["Alice"].age == 30
```

---

### `withValue`

```nim
template withValue*[A, B](t: var Table[A, B], key: A, value, body: untyped)
template withValue*[A, B](t: var Table[A, B], key: A, value, body1, body2: untyped)
template withValue*[A, B](t: Table[A, B], key: A, value, body1, body2: untyped)
```

**Описание:** Безопасный доступ к значению по ключу через блок кода. `value` — инжектируемая переменная с указателем/ссылкой на значение.

- Версия с одним блоком: выполняет `body` только если ключ существует.
- Версия с двумя блоками: `body1` — если ключ найден, `body2` — если нет.
- Версия для `var`-таблицы: значение можно изменять.
- Версия для `let`-таблицы: значение только для чтения.

Особенно полезен для типов с отключённым копированием (`{.error.}` на `=copy`).

**Пример:**
```nim
type User = object
  name: string
  uid: int

var t = initTable[int, User]()
t[1] = User(name: "Hello", uid: 99)

# Изменение (var-версия):
t.withValue(1, value):
  value.name = "Nim"
  value.uid = 1314

assert t[1].name == "Nim"

# С else-веткой:
t.withValue(999, value):
  echo "found"
do:
  echo "not found"  # выполнится этот блок
```

---

### Итераторы `Table`

```nim
iterator pairs*[A, B](t: Table[A, B]): (A, B)
iterator mpairs*[A, B](t: var Table[A, B]): (A, var B)
iterator keys*[A, B](t: Table[A, B]): lent A
iterator values*[A, B](t: Table[A, B]): lent B
iterator mvalues*[A, B](t: var Table[A, B]): var B
```

**Описание:**

- `pairs` — перебирает пары `(ключ, значение)`, только чтение.
- `mpairs` — перебирает пары, значения можно изменять (`var`).
- `keys` — перебирает только ключи.
- `values` — перебирает только значения, только чтение.
- `mvalues` — перебирает значения, можно изменять.

> **Важно:** Нельзя изменять размер таблицы во время итерации (добавлять/удалять ключи). Изменение значений через `mpairs`/`mvalues` безопасно.

**Пример:**
```nim
var a = {'o': @[1, 5, 7, 9], 'e': @[2, 4, 6, 8]}.toTable

for k, v in a.pairs:
  echo k, ": ", v

for v in a.mvalues:
  v.add(99)

assert a == {'e': @[2, 4, 6, 8, 99], 'o': @[1, 5, 7, 9, 99]}.toTable
```

---

## TableRef — ссылочная таблица

`TableRef[A, B]` — это `ref Table[A, B]`. Все методы аналогичны `Table`, но создаются через `new*`-функции и присваивание делит данные между переменными.

### `newTable`

```nim
proc newTable*[A, B](initialSize = defaultInitialSize): TableRef[A, B]
proc newTable*[A, B](pairs: openArray[(A, B)]): TableRef[A, B]
```

**Описание:** Создаёт новую пустую ссылочную таблицу или из набора пар.

**Пример:**
```nim
var a = {1: "one", 2: "two"}.newTable
var b = a         # b и a указывают на одни данные
b[3] = "three"
assert 3 in a     # изменение через b видно в a
```

---

### `newTableFrom`

```nim
proc newTableFrom*[A, B, C](collection: A, index: proc(x: B): C): TableRef[C, B]
```

**Описание:** Ссылочный аналог `indexBy`. Индексирует коллекцию по ключу-результату функции.

**Пример:**
```nim
let users = @[(name: "Alice", age: 30), (name: "Bob", age: 25)]
let byName = users.newTableFrom(proc(u: auto): string = u.name)
assert byName["Bob"].age == 25
```

---

Все остальные методы `TableRef` (`[]`, `[]=`, `hasKey`, `contains`, `hasKeyOrPut`, `getOrDefault`, `mgetOrPut`, `len`, `del`, `pop`, `take`, `clear`, `$`, `==`) и итераторы (`pairs`, `mpairs`, `keys`, `values`, `mvalues`) полностью аналогичны версиям для `Table`, но принимают `TableRef` вместо `Table`. Семантика идентична.

---

## OrderedTable — таблица с порядком

`OrderedTable[A, B]` запоминает порядок вставки ключей. При итерации ключи перебираются в том порядке, в котором они были добавлены.

### `initOrderedTable`

```nim
proc initOrderedTable*[A, B](initialSize = defaultInitialSize): OrderedTable[A, B]
```

**Описание:** Создаёт пустую упорядоченную таблицу.

**Пример:**
```nim
var t = initOrderedTable[string, int]()
t["b"] = 2
t["a"] = 1
t["c"] = 3
for k, v in t.pairs:
  echo k  # выводит: b, a, c — в порядке вставки
```

---

### `toOrderedTable`

```nim
proc toOrderedTable*[A, B](pairs: openArray[(A, B)]): OrderedTable[A, B]
```

**Описание:** Создаёт упорядоченную таблицу из массива пар. Порядок соответствует порядку в массиве.

**Пример:**
```nim
let a = [('z', 1), ('y', 2), ('x', 3)]
let ot = a.toOrderedTable
assert $ot == "{'z': 1, 'y': 2, 'x': 3}"
```

---

### `sort` (для OrderedTable)

```nim
proc sort*[A, B](t: var OrderedTable[A, B], cmp: proc(x, y: (A, B)): int,
    order = SortOrder.Ascending)
```

**Описание:** Сортирует таблицу по пользовательской функции сравнения. После сортировки порядок вставки теряется, но таблица остаётся пригодной для поиска и вставки.

**Пример:**
```nim
import std/algorithm
var a = initOrderedTable[char, int]()
for i, c in "cab":
  a[c] = 10 * i
assert a == {'c': 0, 'a': 10, 'b': 20}.toOrderedTable
a.sort(system.cmp)
assert a == {'a': 10, 'b': 20, 'c': 0}.toOrderedTable
a.sort(system.cmp, order = SortOrder.Descending)
assert a == {'c': 0, 'b': 20, 'a': 10}.toOrderedTable
```

---

### `==` (для OrderedTable)

```nim
proc `==`*[A, B](s, t: OrderedTable[A, B]): bool
```

**Описание:** Возвращает `true`, если содержимое **и порядок** обеих таблиц совпадают. В отличие от `Table`, порядок ключей важен.

**Пример:**
```nim
let
  a = {'a': 5, 'b': 9}.toOrderedTable
  b = {'b': 9, 'a': 5}.toOrderedTable
assert a != b  # разный порядок!
```

---

Все остальные методы `OrderedTable` (`[]`, `[]=`, `hasKey`, `contains`, `hasKeyOrPut`, `getOrDefault`, `mgetOrPut`, `len`, `del`, `pop`, `clear`, `$`) и итераторы (`pairs`, `mpairs`, `keys`, `values`, `mvalues`) аналогичны `Table`. Итераторы обходят элементы **в порядке вставки**.

Метод `del` для `OrderedTable` имеет сложность O(n) (в отличие от O(1) для `Table`), поскольку требует перестройки связного списка порядка.

---

## OrderedTableRef — ссылочная таблица с порядком

Ссылочный вариант `OrderedTable`. Создаётся через:

```nim
proc newOrderedTable*[A, B](initialSize = defaultInitialSize): OrderedTableRef[A, B]
proc newOrderedTable*[A, B](pairs: openArray[(A, B)]): OrderedTableRef[A, B]
```

Все методы (`[]`, `[]=`, `hasKey`, `contains`, `hasKeyOrPut`, `getOrDefault`, `mgetOrPut`, `len`, `del`, `pop`, `clear`, `sort`, `$`, `==`) и итераторы аналогичны `OrderedTable`.

**Пример:**
```nim
let ot = {'z': 1, 'y': 2, 'x': 3}.newOrderedTable
assert $ot == "{'z': 1, 'y': 2, 'x': 3}"
```

---

## CountTable — таблица счётчиков

`CountTable[A]` — специализированная таблица, где значением всегда является `int` — счётчик вхождений ключа. Значение `0` означает отсутствие ключа.

> **Важно:** После вызова `sort` таблица не должна модифицироваться — методы `[]`, `[]=`, `inc`, `hasKey` вызовут ошибку assert.

### `initCountTable`

```nim
proc initCountTable*[A](initialSize = defaultInitialSize): CountTable[A]
```

**Описание:** Создаёт пустую таблицу счётчиков.

**Пример:**
```nim
var ct = initCountTable[char]()
for c in "hello":
  ct.inc(c)
echo ct  # {'h': 1, 'e': 1, 'l': 2, 'o': 1}
```

---

### `toCountTable`

```nim
proc toCountTable*[A](keys: openArray[A]): CountTable[A]
```

**Описание:** Создаёт таблицу счётчиков из коллекции, подсчитывая количество вхождений каждого элемента.

**Пример:**
```nim
let freq = toCountTable("abracadabra")
assert $freq == "{'a': 5, 'd': 1, 'b': 2, 'r': 2, 'c': 1}"
```

---

### `[]` (CountTable)

```nim
proc `[]`*[A](t: CountTable[A], key: A): int
```

**Описание:** Возвращает счётчик для ключа. Если ключ отсутствует — возвращает `0` (не выбрасывает `KeyError`).

**Пример:**
```nim
let ct = toCountTable("hello")
assert ct['l'] == 2
assert ct['z'] == 0  # не выбросит ошибку
```

---

### `[]=` (CountTable)

```nim
proc `[]=`*[A](t: var CountTable[A], key: A, val: int)
```

**Описание:** Устанавливает значение счётчика вручную. `val` должен быть > 0. Установка значения 0 удалит ключ.

---

### `inc` (CountTable)

```nim
proc inc*[A](t: var CountTable[A], key: A, val = 1)
```

**Описание:** Увеличивает счётчик ключа на `val` (по умолчанию на 1). Если ключ отсутствует — создаёт его. Если счётчик достигает 0 — ключ удаляется.

**Пример:**
```nim
var a = toCountTable("aab")
a.inc('a')      # теперь 'a' → 3
a.inc('b', 10)  # теперь 'b' → 11
assert a == toCountTable("aaabbbbbbbbbbb")
```

---

### `smallest` / `largest`

```nim
proc smallest*[A](t: CountTable[A]): tuple[key: A, val: int]
proc largest*[A](t: CountTable[A]): tuple[key: A, val: int]
```

**Описание:** Возвращают пару `(ключ, значение)` с наименьшим или наибольшим счётчиком. Сложность O(n).

**Пример:**
```nim
let ct = toCountTable("abracadabra")
assert ct.largest == ('a', 5)
assert ct.smallest.val == 1  # 'b', 'c', 'd' или 'r'
```

---

### `sort` (CountTable)

```nim
proc sort*[A](t: var CountTable[A], order = SortOrder.Descending)
```

**Описание:** Сортирует таблицу по значению счётчика. По умолчанию — по убыванию (наиболее частые элементы первыми).

> **Предупреждение:** Операция **деструктивная**. После вызова `sort` таблицу нельзя модифицировать — только итерировать.

**Пример:**
```nim
import std/[algorithm, sequtils]
var a = toCountTable("abracadabra")
a.sort()
assert toSeq(a.values) == @[5, 2, 2, 1, 1]
a.sort(SortOrder.Ascending)
assert toSeq(a.values) == @[1, 1, 2, 2, 5]
```

---

### `merge` (CountTable)

```nim
proc merge*[A](s: var CountTable[A], t: CountTable[A])
```

**Описание:** Объединяет таблицу `t` в таблицу `s`, суммируя счётчики одинаковых ключей.

**Пример:**
```nim
var a = toCountTable("aaabbc")
let b = toCountTable("bcc")
a.merge(b)
assert a == toCountTable("aaabbbccc")
```

---

### `hasKey`, `contains`, `getOrDefault`, `del`, `pop`, `clear`, `$`, `==`

Аналогичны `Table`, но ориентированы на `int`-значения:

- `getOrDefault(t, key, def = 0)` — возвращает счётчик или `def`.
- `del` — удаляет ключ (не выбрасывает ошибку при отсутствии).
- `pop` — удаляет и возвращает значение; `true` если ключ существовал.
- `==` — сравнивает содержимое без учёта порядка вставки.

---

## CountTableRef — ссылочная таблица счётчиков

Ссылочный вариант `CountTable`. Создаётся через:

```nim
proc newCountTable*[A](initialSize = defaultInitialSize): CountTableRef[A]
proc newCountTable*[A](keys: openArray[A]): CountTableRef[A]
```

**Пример:**
```nim
let a = newCountTable("aaabbc")
let b = newCountTable("bcc")
a.merge(b)
assert a == newCountTable("aaabbbccc")
```

Все методы (`[]`, `[]=`, `inc`, `smallest`, `largest`, `hasKey`, `contains`, `getOrDefault`, `len`, `del`, `pop`, `clear`, `sort`, `merge`, `$`, `==`) и итераторы аналогичны `CountTable`.

---

## Общие итераторы

Все шесть типов таблиц предоставляют одинаковый набор итераторов:

| Итератор | Тип таблицы | Описание |
|----------|------------|----------|
| `pairs` | любой (не `var`) | Перебирает `(ключ, значение)`, только чтение |
| `mpairs` | `var` | Перебирает `(ключ, значение)`, значение изменяемо |
| `keys` | любой (не `var`) | Перебирает только ключи |
| `values` | любой (не `var`) | Перебирает только значения, только чтение |
| `mvalues` | `var` | Перебирает только значения, изменяемые |

Для `OrderedTable` и `OrderedTableRef` — обход в **порядке вставки** (или отсортированном порядке после `sort`).

**Пример (CountTable):**
```nim
var a = toCountTable("abracadabra")
for k, v in mpairs(a):
  v = 2  # устанавливаем все счётчики в 2
assert a == toCountTable("aabbccddrr")
```

**Пример (OrderedTable):**
```nim
var a = {'o': @[1, 5], 'e': @[2, 4]}.toOrderedTable
for k, v in a.mpairs:
  v.add(v[0] + 10)
assert a == {'o': @[1, 5, 11], 'e': @[2, 4, 12]}.toOrderedTable
```

---

## Вспомогательные функции

### `hash` (для таблиц)

```nim
proc hash*[K, V](s: Table[K, V]): Hash
proc hash*[K, V](s: OrderedTable[K, V]): Hash
proc hash*[V](s: CountTable[V]): Hash
```

**Описание:** Вычисляет хэш всей таблицы. Позволяет использовать таблицы как ключи в других таблицах.

- `Table` и `CountTable`: хэш не зависит от порядка вставки.
- `OrderedTable`: хэш учитывает порядок элементов.

---

## Сводная таблица функций

| Функция | Table | TableRef | OrderedTable | OrderedTableRef | CountTable | CountTableRef |
|---------|:-----:|:--------:|:------------:|:---------------:|:----------:|:-------------:|
| `init*/new*` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `to*Table` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `[]` / `[]=` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `hasKey` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `contains` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `hasKeyOrPut` | ✓ | ✓ | ✓ | ✓ | — | — |
| `getOrDefault` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `mgetOrPut` | ✓ | ✓ | ✓ | ✓ | — | — |
| `len` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `del` | ✓ | ✓ | ✓ O(n) | ✓ O(n) | ✓ | ✓ |
| `pop` / `take` | ✓ | ✓ | ✓ O(n) | ✓ O(n) | ✓ | ✓ |
| `clear` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `inc` | — | — | — | — | ✓ | ✓ |
| `smallest` / `largest` | — | — | — | — | ✓ | ✓ |
| `sort` | — | — | ✓ | ✓ | ✓ деструктивный | ✓ деструктивный |
| `merge` | — | — | — | — | ✓ | ✓ |
| `indexBy` / `newTableFrom` | ✓ | ✓ | — | — | — | — |
| `withValue` | ✓ | — | — | — | — | — |
| `$` / `==` / `hash` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `pairs` / `mpairs` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `keys` / `values` / `mvalues` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

---

## Комплексный пример

```nim
import std/tables

# --- Table: подсчёт слов ---
var wordCount = initTable[string, int]()
for word in ["the", "quick", "brown", "fox", "the", "fox"]:
  wordCount[word] = wordCount.getOrDefault(word, 0) + 1
assert wordCount["the"] == 2

# --- CountTable: то же самое проще ---
let ct = toCountTable(["the", "quick", "brown", "fox", "the", "fox"])
assert ct["the"] == 2
assert ct["quick"] == 1

# --- OrderedTable: сохранение порядка ---
var ot = initOrderedTable[string, int]()
for w in ["first", "second", "third"]:
  ot[w] = ot.len
var keys: seq[string]
for k in ot.keys:
  keys.add(k)
assert keys == @["first", "second", "third"]  # порядок сохранён

# --- Семантика значений vs ссылок ---
var a = {"x": 1}.toTable
var b = a        # независимая копия
b["y"] = 2
assert "y" notin a

var ra = {"x": 1}.newTable
var rb = ra      # общие данные
rb["y"] = 2
assert "y" in ra  # изменение видно через ra
```
