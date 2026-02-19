# Справочник модуля `std/critbits` (Nim)

Модуль реализует **crit bit tree** (дерево критических битов) — эффективную структуру данных для хранения **сортированного множества строк** или **сортированного словаря строк → значений**.

Это форма Patricia-дерева (radix trie). Все ключи хранятся в лексикографически упорядоченном виде, что позволяет эффективно делать запросы по префиксу.

## Два режима использования

```nim
CritBitTree[void]   # Множество строк (set)
CritBitTree[T]      # Словарь строк → T (map)
```

**Характеристики:**
- Поиск, вставка, удаление — O(k), где k — длина ключа.
- Итерация всегда в **лексикографическом порядке**.
- Хранит только строковые ключи.
- Семантика значений (value semantics).

---

## Содержание

1. [Тип `CritBitTree`](#тип-critbittree)
2. [Создание — `toCritBitTree`](#создание--tocritbittree)
3. [Количество элементов — `len`](#len--количество-элементов)
4. [Вставка — `incl`](#incl--вставка)
5. [Вставка/обновление — `[]=`](#--вставкаобновление)
6. [Удаление — `excl`](#excl--удаление)
7. [Проверка наличия — `contains` / `hasKey`](#contains--haskey--проверка-наличия)
8. [Комбинированные операции](#комбинированные-операции)
9. [Чтение значений — `[]`](#--чтение-значений)
10. [Инкремент — `inc`](#inc--инкремент)
11. [Длина общего префикса — `commonPrefixLen`](#commonprefixlen--длина-общего-префикса)
12. [Строковое представление — `$`](#-строковое-представление)
13. [Итераторы — полный обход](#итераторы--полный-обход)
14. [Итераторы — обход по префиксу](#итераторы--обход-по-префиксу)
15. [Сводная таблица](#сводная-таблица)
16. [Комплексный пример](#комплексный-пример)

---

## Тип `CritBitTree`

```nim
type
  CritBitTree*[T] = object
    root: Node[T]   # внутренний корень дерева (не публичен)
    count: int      # количество элементов (не публичен)
```

**Когда использовать вместо `Table`:**
- Нужна итерация в **лексикографическом порядке** без дополнительной сортировки.
- Нужны быстрые запросы по **префиксу** (`keysWithPrefix` и т.п.).
- Только строковые ключи.

---

## Создание — `toCritBitTree`

### Создание словаря из пар

```nim
proc toCritBitTree*[T](pairs: sink openArray[(string, T)]): CritBitTree[T]
```

**Описание:** Создаёт `CritBitTree[T]` из массива пар `(ключ, значение)`.

**Пример:**
```nim
let d = {"key1": 42, "key2": 43}.toCritBitTree
assert d["key1"] == 42
assert d["key2"] == 43
```

---

### Создание множества из строк

```nim
proc toCritBitTree*(items: sink openArray[string]): CritBitTree[void]
```

**Описание:** Создаёт `CritBitTree[void]` (множество) из массива строк. Дубликаты игнорируются.

**Пример:**
```nim
from std/sequtils import toSeq

let s = ["kitten", "puppy", "kitten"].toCritBitTree  # дубликат игнорируется
assert s.len == 2
assert toSeq(s.items) == @["kitten", "puppy"]  # лексикографически
```

---

### Создание пустой структуры

```nim
var s: CritBitTree[void]              # пустое множество
var d: CritBitTree[int]               # пустой словарь
var d2 = default(CritBitTree[string]) # явный default
```

---

## `len` — количество элементов

```nim
func len*[T](c: CritBitTree[T]): int
```

**Описание:** Возвращает количество элементов. Сложность O(1).

**Пример:**
```nim
let c = ["key1", "key2"].toCritBitTree
assert c.len == 2

var empty: CritBitTree[void]
assert empty.len == 0
```

---

## `incl` — вставка

### В множество (`CritBitTree[void]`)

```nim
proc incl*(c: var CritBitTree[void], key: string)
```

**Описание:** Добавляет ключ в множество. Если ключ уже есть — ничего не происходит.

**Пример:**
```nim
var c: CritBitTree[void]
c.incl("apple")
c.incl("banana")
c.incl("apple")  # уже есть — игнорируется
assert c.len == 2
assert "apple" in c
```

---

### В словарь (`CritBitTree[T]`)

```nim
proc incl*[T](c: var CritBitTree[T], key: string, val: sink T)
```

**Описание:** Вставляет пару `(ключ, значение)`. Если ключ уже существует, значение **не перезаписывается**. Для обновления существующего ключа используйте `[]=`.

**Пример:**
```nim
var c: CritBitTree[int]
c.incl("key", 42)
assert c["key"] == 42

c.incl("key", 99)  # ключ уже есть — значение НЕ изменится
assert c["key"] == 42
```

---

## `[]=` — вставка/обновление

```nim
proc `[]=`*[T](c: var CritBitTree[T], key: string, val: sink T)
```

**Описание:** Вставляет или **перезаписывает** значение по ключу. Предпочтительный способ добавления и обновления элементов в словаре.

**Пример:**
```nim
var c: CritBitTree[int]
c["key1"] = 42
c["key2"] = 0
assert c["key1"] == 42

c["key1"] = 99  # перезаписывает
assert c["key1"] == 99
```

---

## `excl` — удаление

```nim
proc excl*[T](c: var CritBitTree[T], key: string)
```

**Описание:** Удаляет ключ (и связанное значение) из `c`. Если ключ отсутствует — ничего не происходит (no-op).

**Пример:**
```nim
var c: CritBitTree[void]
c.incl("key")
assert "key" in c

c.excl("key")
assert "key" notin c

c.excl("key")  # ключа нет — ошибки нет
```

---

## `contains` / `hasKey` — проверка наличия

### `contains`

```nim
func contains*[T](c: CritBitTree[T], key: string): bool
```

**Описание:** Возвращает `true`, если ключ присутствует. Включает поддержку операторов `in` и `notin`.

**Пример:**
```nim
var c: CritBitTree[void]
c.incl("key")
assert c.contains("key")
assert "key" in c
assert "other" notin c
```

---

### `hasKey`

```nim
func hasKey*[T](c: CritBitTree[T], key: string): bool
```

**Описание:** Псевдоним для `contains`. Удобен для единообразия с модулем `tables`.

**Пример:**
```nim
var c: CritBitTree[int]
c["key"] = 5
assert c.hasKey("key")
assert not c.hasKey("missing")
```

---

## Комбинированные операции

### `containsOrIncl` для множества

```nim
proc containsOrIncl*(c: var CritBitTree[void], key: string): bool
```

**Описание:** Если ключ **отсутствует** — вставляет и возвращает `false`. Если **присутствует** — возвращает `true` без изменений.

**Пример:**
```nim
var c: CritBitTree[void]
assert not c.containsOrIncl("key")  # не было → вставлено → false
assert c.containsOrIncl("key")      # уже есть → true
assert c.len == 1
```

---

### `containsOrIncl` для словаря

```nim
proc containsOrIncl*[T](c: var CritBitTree[T], key: string, val: sink T): bool
```

**Описание:** Если ключ отсутствует — вставляет `(key, val)` и возвращает `false`. Если присутствует — возвращает `true` без изменений.

**Пример:**
```nim
var c: CritBitTree[int]
assert not c.containsOrIncl("key", 42)  # вставлено → false
assert c["key"] == 42

assert c.containsOrIncl("key", 99)  # уже есть → true, не изменено
assert c["key"] == 42
```

---

### `missingOrExcl`

```nim
proc missingOrExcl*[T](c: var CritBitTree[T], key: string): bool
```

**Описание:** Если ключ **отсутствует** — возвращает `true` без изменений. Если **присутствует** — удаляет и возвращает `false`.

**Пример:**
```nim
var c: CritBitTree[void]
assert c.missingOrExcl("key")       # нет ключа → true

c.incl("key")
assert not c.missingOrExcl("key")   # был → удалён → false
assert "key" notin c
```

---

## `[]` — чтение значений

```nim
func `[]`*[T](c: CritBitTree[T], key: string): lent T      # только чтение
func `[]`*[T](c: var CritBitTree[T], key: string): var T   # изменяемая ссылка
```

**Описание:** Возвращает значение по ключу. Вторая версия позволяет изменять значение на месте. Если ключ отсутствует — выбрасывается `KeyError`.

**Пример:**
```nim
var c: CritBitTree[int]
c["a"] = 10
c["b"] = 20

assert c["a"] == 10

# Изменение через var-версию:
c["a"] += 5
assert c["a"] == 15

# Несуществующий ключ:
try:
  discard c["z"]
except KeyError:
  echo "KeyError raised"
```

---

## `inc` — инкремент

```nim
proc inc*(c: var CritBitTree[int]; key: string, val: int = 1)
```

**Описание:** Увеличивает значение `c[key]` на `val` (по умолчанию 1). Специализирован только для `CritBitTree[int]`. Если ключ отсутствует — создаётся со значением 0 и к нему прибавляется `val`.

**Пример:**
```nim
var c: CritBitTree[int]
c["key"] = 1
inc(c, "key")
assert c["key"] == 2

inc(c, "key", 10)
assert c["key"] == 12

inc(c, "new")    # ключа не было: 0 + 1 = 1
assert c["new"] == 1
```

---

## `commonPrefixLen` — длина общего префикса

```nim
func commonPrefixLen*[T](c: CritBitTree[T]): int
```

**Описание:** Возвращает длину наибольшего общего префикса **всех** ключей в `c`. Для пустого дерева возвращает 0. Сложность O(1).

**Пример:**
```nim
var c: CritBitTree[void]
assert c.commonPrefixLen == 0

c.incl("key1")
assert c.commonPrefixLen == 4  # один ключ — его длина

c.incl("key2")
assert c.commonPrefixLen == 3  # общий префикс "key"

c.incl("other")
assert c.commonPrefixLen == 0  # нет общего начала
```

---

## `$` Строковое представление

```nim
func `$`*[T](c: CritBitTree[T]): string
```

**Описание:** Преобразует дерево в строку. Вызывается автоматически при `echo`. Пустое множество — `"{}"`, пустой словарь — `"{:}"`.

**Пример:**
```nim
let d = {"key1": 1, "key2": 2}.toCritBitTree
assert $d == """{"key1": 1, "key2": 2}"""

let s = ["key1", "key2"].toCritBitTree
assert $s == """{"key1", "key2"}"""

assert $CritBitTree[int].default == "{:}"
assert $CritBitTree[void].default == "{}"
```

---

## Итераторы — полный обход

Все итераторы перебирают элементы в **лексикографическом порядке** ключей.

### `keys` / `items`

```nim
iterator keys*[T](c: CritBitTree[T]): string
iterator items*[T](c: CritBitTree[T]): string  # псевдоним keys
```

**Описание:** Перебирает все ключи в лексикографическом порядке. `items` позволяет использовать синтаксис `for x in c`.

**Пример:**
```nim
from std/sequtils import toSeq
let c = {"z": 1, "a": 2, "m": 3}.toCritBitTree
assert toSeq(c.keys) == @["a", "m", "z"]

let s = ["banana", "apple", "cherry"].toCritBitTree
assert toSeq(s) == @["apple", "banana", "cherry"]
```

---

### `values`

```nim
iterator values*[T](c: CritBitTree[T]): lent T
```

**Описание:** Перебирает все значения в порядке лексикографической сортировки ключей. Только чтение.

**Пример:**
```nim
from std/sequtils import toSeq
let c = {"z": 1, "a": 2, "m": 3}.toCritBitTree
assert toSeq(c.values) == @[2, 3, 1]  # порядок ключей: a, m, z
```

---

### `mvalues`

```nim
iterator mvalues*[T](c: var CritBitTree[T]): var T
```

**Описание:** Перебирает все значения с возможностью изменения. Требует `var`.

**Пример:**
```nim
var c = {"a": 1, "b": 2, "c": 3}.toCritBitTree
for v in mvalues(c):
  v *= 10
from std/sequtils import toSeq
assert toSeq(c.values) == @[10, 20, 30]
```

---

### `pairs`

```nim
iterator pairs*[T](c: CritBitTree[T]): tuple[key: string, val: T]
```

**Описание:** Перебирает пары `(ключ, значение)` в лексикографическом порядке. Только чтение.

**Пример:**
```nim
from std/sequtils import toSeq
let c = {"key1": 1, "key2": 2}.toCritBitTree
assert toSeq(c.pairs) == @[(key: "key1", val: 1), (key: "key2", val: 2)]
```

---

### `mpairs`

```nim
iterator mpairs*[T](c: var CritBitTree[T]): tuple[key: string, val: var T]
```

**Описание:** Перебирает пары с изменяемыми значениями. Требует `var`.

**Пример:**
```nim
var c = {"a": 10, "b": 20}.toCritBitTree
for key, val in mpairs(c):
  if key == "a":
    val = 99
assert c["a"] == 99
assert c["b"] == 20
```

---

## Итераторы — обход по префиксу

Все итераторы с суффиксом `WithPrefix` перебирают только элементы, чьи ключи **начинаются с заданной строки**. Порядок — лексикографический.

### `keysWithPrefix` / `itemsWithPrefix`

```nim
iterator keysWithPrefix*[T](c: CritBitTree[T], prefix: string): string
iterator itemsWithPrefix*[T](c: CritBitTree[T], prefix: string): string  # псевдоним
```

**Описание:** Перебирает все ключи, начинающиеся с `prefix`.

**Пример:**
```nim
from std/sequtils import toSeq
let c = {"key1": 1, "key2": 2, "other": 3}.toCritBitTree
assert toSeq(c.keysWithPrefix("key")) == @["key1", "key2"]
assert toSeq(c.keysWithPrefix("xyz")) == @[]
```

---

### `valuesWithPrefix`

```nim
iterator valuesWithPrefix*[T](c: CritBitTree[T], prefix: string): lent T
```

**Описание:** Перебирает значения, чьи ключи начинаются с `prefix`. Только чтение.

**Пример:**
```nim
from std/sequtils import toSeq
let c = {"key1": 42, "key2": 43, "other": 0}.toCritBitTree
assert toSeq(c.valuesWithPrefix("key")) == @[42, 43]
```

---

### `mvaluesWithPrefix`

```nim
iterator mvaluesWithPrefix*[T](c: var CritBitTree[T], prefix: string): var T
```

**Описание:** Перебирает значения по префиксу с возможностью изменения.

**Пример:**
```nim
var c = {"key1": 10, "key2": 20, "other": 30}.toCritBitTree
for v in mvaluesWithPrefix(c, "key"):
  v *= 2
assert c["key1"] == 20
assert c["key2"] == 40
assert c["other"] == 30  # не затронут
```

---

### `pairsWithPrefix`

```nim
iterator pairsWithPrefix*[T](c: CritBitTree[T], prefix: string): tuple[key: string, val: T]
```

**Описание:** Перебирает пары `(ключ, значение)` для ключей с заданным префиксом. Только чтение.

**Пример:**
```nim
from std/sequtils import toSeq
let c = {"key1": 42, "key2": 43, "other": 0}.toCritBitTree
assert toSeq(c.pairsWithPrefix("key")) == @[
  (key: "key1", val: 42),
  (key: "key2", val: 43)
]
```

---

### `mpairsWithPrefix`

```nim
iterator mpairsWithPrefix*[T](c: var CritBitTree[T], prefix: string): tuple[key: string, val: var T]
```

**Описание:** Перебирает пары для ключей с заданным префиксом с изменяемыми значениями.

**Пример:**
```nim
var c = {"api_v1": 1, "api_v2": 2, "debug": 0}.toCritBitTree
for key, val in mpairsWithPrefix(c, "api_"):
  val += 100
assert c["api_v1"] == 101
assert c["api_v2"] == 102
assert c["debug"] == 0
```

---

## Сводная таблица

| Функция / итератор | Режим | Описание | Сложность |
|--------------------|-------|----------|-----------|
| `toCritBitTree` (пары) | map | Создать из пар `(string, T)` | O(n·k) |
| `toCritBitTree` (строки) | set | Создать из строк | O(n·k) |
| `len` | оба | Количество элементов | O(1) |
| `contains` / `in` | оба | Проверка наличия ключа | O(k) |
| `hasKey` | оба | Псевдоним `contains` | O(k) |
| `incl` (set) | set | Вставить ключ | O(k) |
| `incl` (map) | map | Вставить ключ+значение (без перезаписи) | O(k) |
| `[]=` | map | Вставить или перезаписать значение | O(k) |
| `[]` (read) | map | Получить значение или `KeyError` | O(k) |
| `[]` (write) | map | Изменяемая ссылка на значение | O(k) |
| `excl` | оба | Удалить ключ | O(k) |
| `containsOrIncl` (set) | set | Проверить; вставить если нет | O(k) |
| `containsOrIncl` (map) | map | Проверить; вставить если нет | O(k) |
| `missingOrExcl` | оба | Проверить; удалить если есть | O(k) |
| `inc` | `int` map | Инкрементировать `c[key]` | O(k) |
| `commonPrefixLen` | оба | Длина общего префикса всех ключей | O(1) |
| `$` | оба | Строковое представление | O(n) |
| `keys` / `items` | оба | Ключи в лексикогр. порядке | O(n) |
| `values` | map | Значения (только чтение) | O(n) |
| `mvalues` | map | Значения (изменяемые) | O(n) |
| `pairs` | map | Пары (только чтение) | O(n) |
| `mpairs` | map | Пары (изменяемые) | O(n) |
| `keysWithPrefix` / `itemsWithPrefix` | оба | Ключи с заданным префиксом | O(k+m) |
| `valuesWithPrefix` | map | Значения по префиксу (чтение) | O(k+m) |
| `mvaluesWithPrefix` | map | Значения по префиксу (изменяемые) | O(k+m) |
| `pairsWithPrefix` | map | Пары по префиксу (чтение) | O(k+m) |
| `mpairsWithPrefix` | map | Пары по префиксу (изменяемые) | O(k+m) |

*k — длина ключа, n — количество элементов, m — количество совпадений по префиксу.*

---

## Комплексный пример

```nim
import std/critbits
from std/sequtils import toSeq

# --- Режим множества (CritBitTree[void]) ---
var words: CritBitTree[void]
for w in ["banana", "apple", "cherry", "apricot", "blueberry"]:
  words.incl(w)

assert words.len == 5
assert "apple" in words
assert "mango" notin words

# Итерация всегда в лексикографическом порядке:
assert toSeq(words) == @["apple", "apricot", "banana", "blueberry", "cherry"]

# Поиск по префиксу:
assert toSeq(words.keysWithPrefix("ap")) == @["apple", "apricot"]
assert toSeq(words.keysWithPrefix("b"))  == @["banana", "blueberry"]

# commonPrefixLen:
var ap: CritBitTree[void]
ap.incl("apple")
ap.incl("apricot")
assert ap.commonPrefixLen == 2  # "ap"

# --- Режим словаря (CritBitTree[T]) ---
var scores = {"alice": 0, "bob": 0, "anna": 0}.toCritBitTree

# Инкремент:
inc(scores, "alice", 10)
inc(scores, "alice", 5)
inc(scores, "bob", 7)
assert scores["alice"] == 15
assert scores["bob"] == 7

# Итерация в отсортированном порядке:
assert toSeq(scores.keys)   == @["alice", "anna", "bob"]
assert toSeq(scores.values) == @[15, 0, 7]

# Поиск по префиксу:
assert toSeq(scores.keysWithPrefix("an")) == @["anna"]
assert toSeq(scores.pairsWithPrefix("al")) == @[(key: "alice", val: 15)]

# containsOrIncl и missingOrExcl:
assert not scores.containsOrIncl("charlie", 3)  # вставлено
assert scores.containsOrIncl("charlie", 99)     # уже есть — не изменено
assert scores["charlie"] == 3

assert not scores.missingOrExcl("charlie")  # был — удалён
assert "charlie" notin scores

# Массовое изменение через mpairs:
for key, val in mpairs(scores):
  val *= 2
assert scores["alice"] == 30
assert scores["bob"]   == 14
```
