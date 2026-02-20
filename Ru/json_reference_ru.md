# Справочник модуля `std/json` (Nim)

> Модуль реализует высокопроизводительный парсер и генератор JSON для языка Nim.  
> Ключевой тип — `JsonNode`, вариантный объект, представляющий любое JSON-значение.

---

## Содержание

1. [Типы данных](#типы-данных)
2. [Создание узлов](#создание-узлов)
3. [Конструктор `%` и макрос `%*`](#конструктор--и-макрос-)
4. [Получение значений](#получение-значений)
5. [Доступ к элементам и обход](#доступ-к-элементам-и-обход)
6. [Изменение узлов](#изменение-узлов)
7. [Проверки и поиск](#проверки-и-поиск)
8. [Сравнение и хеширование](#сравнение-и-хеширование)
9. [Итераторы](#итераторы)
10. [Сериализация](#сериализация)
11. [Экранирование строк](#экранирование-строк)
12. [Парсинг](#парсинг)
13. [Десериализация в типы Nim](#десериализация-в-типы-nim)

---

## Типы данных

### `JsonNodeKind`

Перечисление, описывающее все возможные виды JSON-значений:

| Значение  | JSON-тип         |
|-----------|------------------|
| `JNull`   | `null`           |
| `JBool`   | `true` / `false` |
| `JInt`    | целое число      |
| `JFloat`  | вещественное     |
| `JString` | строка           |
| `JObject` | объект `{}`      |
| `JArray`  | массив `[]`      |

### `JsonNode`

Ссылочный тип (`ref`), хранящий JSON-значение любого вида. Поле `kind` доступно напрямую:

```nim
import std/json

let node = parseJson("42")
echo node.kind  # JInt
```

---

## Создание узлов

### `newJString(s: string): JsonNode`

Создаёт узел типа `JString`.

```nim
let s = newJString("Привет")
echo s        # "Привет"
echo s.kind   # JString
```

---

### `newJInt(n: BiggestInt): JsonNode`

Создаёт узел типа `JInt`.

```nim
let i = newJInt(42)
echo i        # 42
echo i.kind   # JInt
```

---

### `newJFloat(n: float): JsonNode`

Создаёт узел типа `JFloat`.

```nim
let f = newJFloat(3.14)
echo f        # 3.14
echo f.kind   # JFloat
```

---

### `newJBool(b: bool): JsonNode`

Создаёт узел типа `JBool`.

```nim
let b = newJBool(true)
echo b        # true
echo b.kind   # JBool
```

---

### `newJNull(): JsonNode`

Создаёт узел типа `JNull`.

```nim
let n = newJNull()
echo n        # null
echo n.kind   # JNull
```

---

### `newJObject(): JsonNode`

Создаёт пустой узел типа `JObject`.

```nim
let obj = newJObject()
obj["key"] = newJInt(1)
echo obj      # {"key":1}
```

---

### `newJArray(): JsonNode`

Создаёт пустой узел типа `JArray`.

```nim
let arr = newJArray()
arr.add(newJInt(1))
arr.add(newJInt(2))
echo arr      # [1,2]
```

---

## Конструктор `%` и макрос `%*`

### `%(s: string): JsonNode`

Оборачивает строку в `JString`.

```nim
let node = %"hello"
echo node   # "hello"
```

---

### `%(n: int): JsonNode`

Оборачивает целое число в `JInt`.

```nim
let node = %100
echo node   # 100
```

---

### `%(n: float): JsonNode`

Оборачивает `float` в `JFloat`. Особые значения (`NaN`, `Inf`, `-Inf`) сохраняются как `JString`.

```nim
echo %3.14       # 3.14
echo %NaN        # "nan"
echo %Inf        # "inf"
echo %(% [NaN, Inf, -Inf, 0.0])  # ["nan","inf","-inf",0.0]
```

---

### `%(b: bool): JsonNode`

Оборачивает `bool` в `JBool`.

```nim
let node = %true
echo node   # true
```

---

### `%[T](elements: openArray[T]): JsonNode`

Создаёт `JArray` из массива или последовательности.

```nim
let arr = %[1, 2, 3]
echo arr    # [1,2,3]
```

---

### `%[T: object](o: T): JsonNode`

Сериализует объект Nim в `JObject`. Поля сохраняются в порядке объявления.

```nim
type Point = object
  x, y: int

let p = Point(x: 10, y: 20)
echo %p     # {"x":10,"y":20}
```

---

### `%(o: ref object): JsonNode`

Сериализует ссылочный объект. Если ссылка `nil` — возвращает `JNull`.

```nim
type Node = ref object
  val: int

let n: Node = nil
echo %n     # null
```

---

### `%(o: enum): JsonNode`

Сериализует значение перечисления как строку `JString`.

```nim
type Color = enum Red, Green, Blue
echo %Green  # "Green"
```

---

### `%[T](table: Table[string, T]|OrderedTable[string, T]): JsonNode`

Создаёт `JObject` из таблицы.

```nim
import std/tables
let t = {"a": 1, "b": 2}.toTable
echo %t   # {"a":1,"b":2} (порядок не гарантирован для Table)
```

---

### `%[T](opt: Option[T]): JsonNode`

Если `Option` пуст — возвращает `JNull`, иначе оборачивает значение.

```nim
import std/options
echo %some(42)   # 42
echo %none(int)  # null
```

---

### `macro %*(x: untyped): untyped`

Удобный макрос для создания JSON-структур прямо из синтаксиса Nim. Применяет `%` к каждому элементу автоматически.

```nim
import std/json

var name = "Alice"
let age = 30

let data = %* {
  "user": name,
  "age": age,
  "scores": [10, 20, 30]
}
echo data
# {"user":"Alice","age":30,"scores":[10,20,30]}

# Вложенные структуры
var j = %* {"a": {"b": {"c": 42}}}
echo j{"a", "b", "c"}  # 42
```

---

## Получение значений

Все `get`-процедуры безопасны: при несоответствии типа или при `nil`-узле возвращают значение по умолчанию (не бросают исключение).

### `getStr(n: JsonNode, default = ""): string`

Возвращает строковое значение `JString`-узла.

```nim
let j = parseJson("""{"name": "Bob"}""")
echo j["name"].getStr()         # Bob
echo j{"missing"}.getStr("?")  # ?
```

---

### `getInt(n: JsonNode, default = 0): int`

Возвращает целочисленное значение `JInt`-узла.

```nim
let j = parseJson("""{"n": 7}""")
echo j["n"].getInt()       # 7
echo j{"x"}.getInt(-1)    # -1
```

---

### `getBiggestInt(n: JsonNode, default = 0): BiggestInt`

Аналогично `getInt`, но возвращает `BiggestInt` — для очень больших целых чисел.

```nim
let j = parseJson("9999999999999999")
echo j.getBiggestInt()   # 9999999999999999
```

---

### `getFloat(n: JsonNode, default = 0.0): float`

Возвращает вещественное значение. Принимает как `JFloat`, так и `JInt`.

```nim
let j = parseJson("""{"pi": 3.14, "n": 2}""")
echo j["pi"].getFloat()   # 3.14
echo j["n"].getFloat()    # 2.0  (автоматическое преобразование)
echo j{"x"}.getFloat(1.0) # 1.0
```

---

### `getBool(n: JsonNode, default = false): bool`

Возвращает булево значение `JBool`-узла.

```nim
let j = parseJson("""{"flag": true}""")
echo j["flag"].getBool()       # true
echo j{"miss"}.getBool(true)  # true
```

---

### `getFields(n: JsonNode, default = ...): OrderedTable[string, JsonNode]`

Возвращает таблицу полей `JObject`-узла.

```nim
let j = parseJson("""{"a": 1, "b": 2}""")
let fields = j.getFields()
for k, v in fields:
  echo k, " = ", v
# a = 1
# b = 2
```

---

### `getElems(n: JsonNode, default = @[]): seq[JsonNode]`

Возвращает последовательность элементов `JArray`-узла.

```nim
let j = parseJson("[10, 20, 30]")
let elems = j.getElems()
echo elems.len   # 3
echo elems[1]    # 20
```

---

## Доступ к элементам и обход

### `[](node: JsonNode, name: string): JsonNode`

Получает поле объекта по имени. Бросает `KeyError`, если ключ не найден.

```nim
let j = parseJson("""{"x": 5}""")
echo j["x"]   # 5
# j["y"]  -> KeyError!
```

---

### `[](node: JsonNode, index: int): JsonNode`

Получает элемент массива по индексу.

```nim
let j = parseJson("[10, 20, 30]")
echo j[0]   # 10
echo j[2]   # 30
```

---

### `[](node: JsonNode, index: BackwardsIndex): JsonNode`

Доступ к массиву с конца через оператор `^`.

```nim
let j = parseJson("[1, 2, 3, 4, 5]")
echo j[^1]   # 5
echo j[^2]   # 4
```

---

### `[][U, V](a: JsonNode, x: HSlice[U, V]): JsonNode`

Срез массива (включительный диапазон). Возвращает новый `JArray`.

```nim
let arr = %[0, 1, 2, 3, 4, 5]
echo arr[2..4]    # [2,3,4]
echo arr[2..^2]   # [2,3,4]
echo arr[^4..^2]  # [2,3,4]
```

---

### `{}(node: JsonNode, keys: varargs[string]): JsonNode`

Безопасный доступ по цепочке ключей. Возвращает `nil`, если любой ключ не найден.

```nim
let j = parseJson("""{"a": {"b": {"c": 99}}}""")
echo j{"a", "b", "c"}   # 99
echo j{"a", "x"}        # (nil — нет исключения)
echo j{"a", "x"}.getInt(-1)  # -1
```

---

### `{}(node: JsonNode, index: varargs[int]): JsonNode`

Безопасный доступ по цепочке индексов массива. Возвращает `nil`, если индекс выходит за границы.

```nim
let j = parseJson("[[1,2],[3,4]]")
echo j{0, 1}   # 2
echo j{5, 0}   # (nil)
```

---

### `getOrDefault(node: JsonNode, key: string): JsonNode`

Возвращает поле объекта или `nil`, если узел не является объектом / ключ отсутствует.

```nim
let j = parseJson("""{"a": 1}""")
echo j.getOrDefault("a")    # 1
echo j.getOrDefault("b")    # (nil)
```

---

### `{}=(node: JsonNode, keys: varargs[string], value: JsonNode)`

Устанавливает значение по цепочке ключей, создавая промежуточные объекты при необходимости (автовивификация).

```nim
var j = newJObject()
j{"x", "y", "z"} = %42
echo j   # {"x":{"y":{"z":42}}}
```

---

### `len(n: JsonNode): int`

Возвращает количество элементов в `JArray` или пар в `JObject`. Для остальных типов возвращает `0`.

```nim
let arr = parseJson("[1,2,3]")
echo arr.len   # 3

let obj = parseJson("""{"a":1,"b":2}""")
echo obj.len   # 2

let s = parseJson("\"hello\"")
echo s.len     # 0
```

---

## Изменение узлов

### `add(father, child: JsonNode)`

Добавляет элемент в `JArray`.

```nim
var arr = newJArray()
arr.add(%1)
arr.add(%"two")
echo arr   # [1,"two"]
```

---

### `add(obj: JsonNode, key: string, val: JsonNode)`

Устанавливает поле в `JObject` (синоним `obj[key] = val`).

```nim
var obj = newJObject()
obj.add("name", %"Alice")
echo obj   # {"name":"Alice"}
```

---

### `[]=(obj: JsonNode, key: string, val: JsonNode)`

Оператор присваивания поля в `JObject`.

```nim
var obj = newJObject()
obj["age"] = %25
echo obj   # {"age":25}
```

---

### `delete(obj: JsonNode, key: string)`

Удаляет поле из `JObject`. Бросает `KeyError`, если ключ не найден.

```nim
var j = parseJson("""{"a": 1, "b": 2}""")
j.delete("a")
echo j   # {"b":2}
```

---

### `copy(p: JsonNode): JsonNode`

Выполняет глубокое копирование узла.

```nim
let original = parseJson("""{"x": [1,2,3]}""")
var clone = original.copy()
clone["x"][0] = %99
echo original   # {"x":[1,2,3]}  — оригинал не изменился
echo clone      # {"x":[99,2,3]}
```

---

## Проверки и поиск

### `hasKey(node: JsonNode, key: string): bool`

Проверяет существование ключа в `JObject`.

```nim
let j = parseJson("""{"a": 1}""")
echo j.hasKey("a")   # true
echo j.hasKey("b")   # false
```

---

### `contains(node: JsonNode, key: string): bool`

Проверяет наличие ключа в `JObject`. Позволяет использовать оператор `in`.

```nim
let j = parseJson("""{"x": 1}""")
echo "x" in j   # true
echo "y" in j   # false
```

---

### `contains(node: JsonNode, val: JsonNode): bool`

Проверяет наличие значения в `JArray`.

```nim
let arr = parseJson("[1, 2, 3]")
echo arr.contains(%2)   # true
echo arr.contains(%9)   # false
```

---

## Сравнение и хеширование

### `==(a, b: JsonNode): bool`

Сравнивает два узла на равенство. Для объектов порядок ключей не важен.

```nim
let a = parseJson("""{"x": 1, "y": 2}""")
let b = parseJson("""{"y": 2, "x": 1}""")
echo a == b   # true

echo parseJson("null") == newJNull()   # true
```

---

### `hash(n: JsonNode): Hash`

Вычисляет хеш узла. Позволяет использовать `JsonNode` в таблицах и множествах.

```nim
import std/tables
var t = initTable[JsonNode, string]()
t[%42] = "answer"
echo t[%42]   # answer
```

---

## Итераторы

### `items(node: JsonNode): JsonNode`

Итерация по элементам `JArray`.

```nim
let arr = parseJson("[10, 20, 30]")
for item in arr:
  echo item
# 10
# 20
# 30
```

---

### `mitems(node: var JsonNode): var JsonNode`

Итерация по элементам `JArray` с возможностью изменения.

```nim
var arr = parseJson("[1, 2, 3]")
for item in arr.mitems:
  item = %(item.getInt * 10)
echo arr   # [10,20,30]
```

---

### `pairs(node: JsonNode): tuple[key: string, val: JsonNode]`

Итерация по парам «ключ–значение» `JObject`. Порядок сохраняется (OrderedTable).

```nim
let j = parseJson("""{"b": 2, "a": 1}""")
for k, v in j:
  echo k, ": ", v
# b: 2
# a: 1
```

---

### `keys(node: JsonNode): string`

Итерация только по ключам `JObject`.

```nim
let j = parseJson("""{"x": 1, "y": 2}""")
for k in j.keys:
  echo k
# x
# y
```

---

### `mpairs(node: var JsonNode): tuple[key: string, val: var JsonNode]`

Итерация по парам `JObject` с возможностью изменения значений.

```nim
var j = parseJson("""{"a": 1, "b": 2}""")
for k, v in j.mpairs:
  v = %(v.getInt * 100)
echo j   # {"a":100,"b":200}
```

---

### `parseJsonFragments(s: Stream, filename = "", rawIntegers = false, rawFloats = false): JsonNode`

Итератор, читающий несколько JSON-значений из потока, разделённых пробельными символами. Закрывает поток после завершения.

```nim
import std/[json, streams]

let s = newStringStream("""{"a":1} {"b":2} [1,2,3]""")
for node in parseJsonFragments(s):
  echo node
# {"a":1}
# {"b":2}
# [1,2,3]
```

---

## Сериализация

### `toUgly(result: var string, node: JsonNode)`

Сериализует узел в компактный JSON без форматирования. Быстрее, чем `pretty`.

```nim
var output = ""
let j = parseJson("""{"a": 1, "b": [1,2]}""")
toUgly(output, j)
echo output   # {"a":1,"b":[1,2]}
```

---

### `pretty(node: JsonNode, indent = 2): string`

Возвращает отформатированный JSON с отступами. Аналог `json.dumps(..., indent=2)` в Python.

```nim
let j = %* {"name": "Isaac", "books": ["Robot Dreams"],
             "details": {"age": 35, "pi": 3.1415}}
echo pretty(j)
# {
#   "name": "Isaac",
#   "books": [
#     "Robot Dreams"
#   ],
#   "details": {
#     "age": 35,
#     "pi": 3.1415
#   }
# }

# Можно менять размер отступа:
echo pretty(j, indent = 4)
```

---

### `$(node: JsonNode): string`

Оператор преобразования в строку — компактный JSON (однострочный). Используется при `echo`.

```nim
let j = %* {"x": 1, "y": [2, 3]}
echo $j   # {"x":1,"y":[2,3]}
echo j    # то же самое (echo вызывает $)
```

---

## Экранирование строк

### `escapeJsonUnquoted(s: string): string`

### `escapeJsonUnquoted(s: string; result: var string)`

Экранирует специальные символы строки для JSON, **без** обрамляющих кавычек.

```nim
echo escapeJsonUnquoted("line1\nline2")   # line1\nline2
echo escapeJsonUnquoted("tab\there")      # tab\there
```

---

### `escapeJson(s: string): string`

### `escapeJson(s: string; result: var string)`

Экранирует строку для JSON **с** обрамляющими кавычками.

```nim
echo escapeJson("say \"hello\"")   # "say \"hello\""
echo escapeJson("a\nb")            # "a\nb"
```

---

## Парсинг

### `parseJson(buffer: string; rawIntegers = false, rawFloats = false): JsonNode`

Парсит JSON из строки. Бросает `JsonParsingError` при наличии лишних данных после JSON-значения.

- `rawIntegers = true` — целые числа не конвертируются в `JInt`, хранятся как `JString`.
- `rawFloats = true` — аналогично для вещественных чисел.

```nim
let j = parseJson("""{"key": 42, "list": [1, 2, 3]}""")
echo j["key"].getInt()    # 42
echo j["list"][1].getInt() # 2

# rawIntegers:
let j2 = parseJson("12345678901234567890", rawIntegers = true)
echo j2.kind    # JString (число слишком большое для BiggestInt)
echo j2.getStr  # 12345678901234567890
```

---

### `parseJson(s: Stream, filename = "", rawIntegers = false, rawFloats = false): JsonNode`

Парсит JSON из потока `Stream`. Закрывает поток после завершения.

```nim
import std/[json, streams]

let s = newStringStream("""[1, 2, 3]""")
let j = parseJson(s)
echo j[0].getInt()   # 1
```

---

### `parseFile(filename: string): JsonNode`

Парсит JSON из файла. Бросает `IOError`, если файл не может быть открыт.

```nim
# Допустим, файл data.json содержит {"answer": 42}
let j = parseFile("data.json")
echo j["answer"].getInt()   # 42
```

---

## Десериализация в типы Nim

### `to[T](node: JsonNode, t: typedesc[T]): T`

Десериализует `JsonNode` в произвольный тип Nim. Поддерживает объекты, кортежи, последовательности, массивы, таблицы, `Option`, перечисления и вложенные структуры.

Ограничения:
- Гетерогенные массивы не поддерживаются.
- Варианты объектов с множествами не поддерживаются.
- Аннотации `not nil` не поддерживаются.

```nim
import std/[json, options]

type
  Address = object
    city: string
    zip: int

  User = object
    name: string
    age: int
    address: Address
    nickname: Option[string]

let json = parseJson("""
{
  "name": "Alice",
  "age": 30,
  "address": {"city": "Moscow", "zip": 101000}
}
""")

let user = json.to(User)
echo user.name            # Alice
echo user.age             # 30
echo user.address.city    # Moscow
echo user.nickname.isSome # false
```

```nim
# Десериализация в seq
let arr = parseJson("[10, 20, 30]")
let nums = arr.to(seq[int])
echo nums   # @[10, 20, 30]

# Десериализация перечисления
type Color = enum Red, Green, Blue
let c = parseJson(""""Green"""").to(Color)
echo c   # Green

# Десериализация таблицы
import std/tables
let tbl = parseJson("""{"a": 1, "b": 2}""").to(Table[string, int])
echo tbl["a"]   # 1
```

---

## Краткая шпаргалка

```nim
import std/json

# Парсинг
let j = parseJson("""{"name":"Nim","ver":2,"tags":["fast","safe"]}""")

# Чтение
echo j["name"].getStr()        # Nim
echo j["ver"].getInt()         # 2
echo j["tags"][0].getStr()     # fast
echo j{"missing"}.getStr("?") # ?  (безопасно, без исключения)

# Создание
var data = %* {
  "scores": [95, 87, 100],
  "active": true,
  "ratio": 0.99
}
data["extra"] = %* {"note": "ok"}

# Вывод
echo data          # компактный JSON
echo pretty(data)  # форматированный JSON

# Десериализация
type Config = object
  scores: seq[int]
  active: bool

let cfg = data.to(Config)
echo cfg.active   # true
```
