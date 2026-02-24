# Справочник по модулю `jsonutils`

> **Модуль:** `std/jsonutils`  
> **Язык:** Nim  
> **Назначение:** Расширяемая, типобезопасная сериализация и десериализация произвольных Nim-типов в JSON и обратно — без написания ручных конвертеров для большинства распространённых типов.

---

## Ключевая идея

`jsonutils` строится вокруг двух центральных функций: `toJson` (Nim → JSON) и `jsonTo` / `fromJson` (JSON → Nim). Они автоматически обрабатывают объекты, кортежи, последовательности, множества, таблицы, опциональные значения, перечисления и примитивные типы — через анализ типов на этапе компиляции.

Главное отличие от базового модуля `json` — **расширяемость через хуки**: если вы определяете `toJsonHook` или `fromJsonHook` для своего типа, они подхватываются автоматически — без регистрации и без наследования. Это позволяет настраивать сериализацию сторонних типов, не меняя их исходный код.

---

## Экспортируемые типы

### `Joptions`

```nim
type Joptions* = object
  allowExtraKeys*: bool
  allowMissingKeys*: bool
```

Объект настроек, управляющий тем, насколько строго `fromJson` / `jsonTo` проверяют соответствие формы входящего JSON при десериализации в Nim-объекты. Оба поля по умолчанию равны `false` — то есть по умолчанию применяется строгое соответствие.

**`allowExtraKeys`** — при значении `true` JSON-объект может содержать ключи, для которых нет соответствующего поля в Nim-типе. Лишние ключи молча игнорируются. Полезно при получении JSON от внешних API, которые могут добавлять новые поля.

**`allowMissingKeys`** — при значении `true` Nim-тип может иметь поля, которым нет соответствующего ключа в JSON. Отсутствующие поля сохраняют своё нулевое значение по умолчанию. Полезно для частичных обновлений или необязательных данных.

```nim
import std/[jsonutils, json]

type Config = object
  host: string
  port: int
  debug: bool

let j = parseJson("""{"host": "localhost", "port": 8080, "extra": "ignored"}""")

# Строгий режим (по умолчанию) — упадёт, потому что "extra" неизвестен:
# let c = jsonTo(j, Config)

# Мягкий режим — лишние JSON-ключи разрешены:
let c = jsonTo(j, Config, Joptions(allowExtraKeys: true))
assert c.host == "localhost"
```

---

### `EnumMode`

```nim
type EnumMode* = enum
  joptEnumOrd     # сериализовать как целое (ordinal) — по умолчанию
  joptEnumSymbol  # сериализовать как внутреннее имя символа
  joptEnumString  # сериализовать как строку через $
```

Управляет тем, как значения перечислений записываются в JSON функцией `toJson`. Влияет только на направление сериализации (Nim → JSON). Десериализация (`fromJson`) принимает как целые числа, так и строки и выбирает нужный вариант автоматически.

| Режим | Пример для `Color.Red` |
|---|---|
| `joptEnumOrd` | `0` |
| `joptEnumSymbol` | `"Red"` |
| `joptEnumString` | `"Red"` |

Разница между `joptEnumSymbol` и `joptEnumString` важна, когда для перечисления перегружен оператор `$`: `joptEnumSymbol` всегда использует объявленный идентификатор, а `joptEnumString` — то, что возвращает `$`.

```nim
import std/[jsonutils, json]

type Direction = enum dNorth, dSouth, dEast, dWest

let d = dEast
echo toJson(d)                                               # 2
echo toJson(d, ToJsonOptions(enumMode: joptEnumString))      # "dEast"
echo toJson(d, ToJsonOptions(enumMode: joptEnumSymbol))      # "dEast"
```

---

### `JsonNodeMode`

```nim
type JsonNodeMode* = enum
  joptJsonNodeAsRef    # вернуть ссылку на JsonNode как есть (по умолчанию)
  joptJsonNodeAsCopy   # вернуть глубокую копию JsonNode
  joptJsonNodeAsObject # обработать JsonNode как обычный ref-объект
```

Управляет поведением `toJson`, когда он встречает значение, которое само является `JsonNode`. Поскольку `JsonNode` — это уже JSON, повторная обёртка в большинстве случаев неправильна.

- **`joptJsonNodeAsRef`** (по умолчанию): возвращается оригинальная ссылка на `JsonNode`. Быстро, но изменения результата затронут исходный объект.
- **`joptJsonNodeAsCopy`**: создаётся глубокая копия. Безопасно для независимых изменений.
- **`joptJsonNodeAsObject`**: `JsonNode` трактуется как обычный Nim ref-объект и сериализуется по полям (нужно редко — в основном для интроспекции).

---

### `ToJsonOptions`

```nim
type ToJsonOptions* = object
  enumMode*: EnumMode
  jsonNodeMode*: JsonNodeMode
```

Объект, объединяющий все настройки, управляющие поведением `toJson`. Создавайте его через `initToJsonOptions()` и при необходимости меняйте отдельные поля.

---

## Процедуры и итераторы

### `initToJsonOptions`

```nim
proc initToJsonOptions*(): ToJsonOptions
```

**Что делает:** Возвращает `ToJsonOptions` с разумными значениями по умолчанию: `enumMode = joptEnumOrd` и `jsonNodeMode = joptJsonNodeAsRef`. Именно эти настройки `toJson` использует, когда вы не передаёте ничего явно.

**Когда использовать:** Как отправную точку, когда нужно изменить только один аспект сериализации, не задавая все поля с нуля.

```nim
import std/jsonutils

var opts = initToJsonOptions()
opts.enumMode = joptEnumString   # переопределяем только режим перечислений
```

---

### `toJson`

```nim
proc toJson*[T](a: T, opt = initToJsonOptions()): JsonNode
```

**Что делает:** Преобразует любое Nim-значение `a` в `JsonNode`. Это главная точка входа для сериализации. Поддерживает:

- **Примитивы** (`bool`, целые числа, числа с плавающей точкой, `string`, `cstring`, `char`) — отображаются в их естественные JSON-аналоги. Важный нюанс: специальные float-значения `NaN`, `Inf` и `-Inf` сериализуются как JSON-строки (`"nan"`, `"inf"`, `"-inf"`), поскольку JSON не поддерживает их как числа.
- **Перечисления** — сериализуются согласно `opt.enumMode` (по умолчанию — как целое ordinal).
- **Объекты и именованные кортежи** — становятся JSON-объектами `{}` с именами полей в качестве ключей.
- **Анонимные кортежи** — становятся JSON-массивами `[]`.
- **Последовательности, массивы, множества** — становятся JSON-массивами `[]`.
- **`ref` и `ptr`** — нулевой указатель становится `null`; ненулевой разыменовывается, и его содержимое сериализуется.
- **`distinct`-типы** — сериализуется базовый тип.
- **`JsonNode`** — обрабатывается согласно `opt.jsonNodeMode`.
- **Пользовательские хуки**: если в области видимости есть `toJsonHook(a)` или `toJsonHook(a, opt)` для типа `T`, вызывается он. Именно так подключается собственная логика.

**Когда использовать:** Всякий раз, когда нужно получить JSON из Nim-структуры данных — для сохранения на диск, отправки по сети, логирования или тестирования.

```nim
import std/[jsonutils, json]

type Person = object
  name: string
  age: int

let p = Person(name: "Alice", age: 30)
let j = toJson(p)
assert $j == """{"name":"Alice","age":30}"""

# Последовательности и вложенность работают автоматически:
let people = @[Person(name: "Bob", age: 25), Person(name: "Carol", age: 40)]
echo $toJson(people)
# [{"name":"Bob","age":25},{"name":"Carol","age":40}]

# Специальная обработка float:
assert $toJson(Inf)  == "\"inf\""
assert $toJson(0.5)  == "0.5"
```

---

### `fromJson`

```nim
proc fromJson*[T](a: var T, b: JsonNode, opt = Joptions())
```

**Что делает:** Десериализует `JsonNode` `b` в существующую Nim-переменную `a` **на месте** (in-place). Это изменяемый аналог `jsonTo`. Обе функции выполняют одну работу; `fromJson` удобен, когда переменная уже существует и её нужно заполнить (например, в горячем цикле, чтобы избежать лишних выделений памяти).

Процедура применяет в обратном порядке те же правила типов, что и `toJson`. Перед основной логикой проверяет наличие `fromJsonHook` в области видимости и при необходимости передаёт управление ему.

Ошибки валидации вызывают `ValueError` с сообщением, точно описывающим, какое поле или условие не выполнено.

**Когда использовать:** Когда есть готовая переменная для заполнения, или при реализации `fromJsonHook` для собственного типа — чтобы вызвать десериализацию по умолчанию для отдельных полей.

```nim
import std/[jsonutils, json]

type Config = object
  host: string
  port: int

var c: Config
fromJson(c, parseJson("""{"host":"0.0.0.0","port":443}"""))
assert c.host == "0.0.0.0"
assert c.port == 443
```

---

### `jsonTo`

```nim
proc jsonTo*(b: JsonNode, T: typedesc, opt = Joptions()): T
```

**Что делает:** Десериализует `JsonNode` в **новое** значение типа `T` и возвращает его. Это функциональный, возвращающий значение аналог `fromJson`. Внутри просто вызывает `fromJson` на свежесозданном значении по умолчанию и возвращает результат.

**Когда использовать:** Наиболее удобный способ десериализации JSON, когда нужно получить чистое новое значение, а не изменять существующее. Естественно сочетается с `toJson` для образования пары «туда и обратно».

```nim
import std/[jsonutils, json]

type Point = object
  x, y: float

let j = parseJson("""{"x": 1.5, "y": -3.0}""")
let p = jsonTo(j, Point)
assert p.x == 1.5
assert p.y == -3.0

# Полный круговой обход:
assert p.toJson.jsonTo(Point) == p
```

---

### `fromJsonHook` для `Table` и `OrderedTable`

```nim
proc fromJsonHook*[K: string|cstring, V](
    t: var (Table[K, V] | OrderedTable[K, V]),
    jsonNode: JsonNode, opt = Joptions())
```

**Что делает:** Обучает `fromJson` десериализовывать JSON-объект в Nim `Table` или `OrderedTable` с ключами типа `string` или `cstring`. Каждый JSON-ключ становится ключом таблицы; каждое JSON-значение десериализуется в тип `V`.

Этот хук вызывается автоматически из `fromJson` — вы никогда не вызываете его напрямую. Именно его существование делает возможным `fromJson(myTable, jsonNode)`.

Входящий `jsonNode` **обязан** быть типа `JObject`; в противном случае срабатывает утверждение (assert).

```nim
import std/[jsonutils, json, tables]

var scores: Table[string, int]
fromJson(scores, parseJson("""{"alice": 95, "bob": 87}"""))
assert scores["alice"] == 95
assert scores["bob"] == 87
```

---

### `toJsonHook` для `Table` и `OrderedTable`

```nim
proc toJsonHook*[K: string|cstring, V](
    t: (Table[K, V] | OrderedTable[K, V]),
    opt = initToJsonOptions()): JsonNode
```

**Что делает:** Обучает `toJson` сериализовывать `Table` или `OrderedTable` с ключами `string`/`cstring` в JSON-объект. Каждая пара ключ-значение таблицы становится парой в JSON-объекте `{}`.

Для таблиц с нестроковыми ключами (например, `Table[int, string]`) этот хук не применяется. В таком случае нужно сначала преобразовать таблицу в последовательность кортежей и сериализовать её.

```nim
import std/[jsonutils, json, tables]

let prices = {"apple": 1.2, "banana": 0.5}.toTable
echo $toJson(prices)
# {"apple": 1.2, "banana": 0.5}   (порядок ключей для Table не гарантирован)

# Нестроковые ключи — сначала конвертируем в seq кортежей:
import std/sugar
let ids = {42: "foo", 99: "bar"}.toTable
let pairs = collect: (for k, v in ids: (k, v))
echo $toJson(pairs)
# [[42,"foo"],[99,"bar"]]
```

---

### `fromJsonHook` для `HashSet` и `OrderedSet`

```nim
proc fromJsonHook*[A](s: var SomeSet[A], jsonNode: JsonNode, opt = Joptions())
```

**Что делает:** Обучает `fromJson` десериализовывать JSON-массив в Nim `HashSet` или `OrderedSet`. Каждый элемент JSON-массива десериализуется в тип `A` и добавляется в множество.

Входящий `jsonNode` **обязан** быть типа `JArray`; иначе срабатывает утверждение. Дубликаты в JSON-массиве молча отбрасываются — как и ожидается для множества.

```nim
import std/[jsonutils, json, sets]

var tags: HashSet[string]
fromJson(tags, parseJson("""["nim", "json", "nim"]"""))
assert tags == ["nim", "json"].toHashSet  # дубликат удалён
```

---

### `toJsonHook` для `HashSet` и `OrderedSet`

```nim
proc toJsonHook*[A](s: SomeSet[A], opt = initToJsonOptions()): JsonNode
```

**Что делает:** Обучает `toJson` сериализовывать `HashSet` или `OrderedSet` в JSON-массив. Каждый элемент множества становится элементом массива.

Обратите внимание: `HashSet` не имеет гарантированного порядка итерации, поэтому порядок элементов в результирующем JSON-массиве недетерминирован. `OrderedSet` сохраняет порядок вставки.

```nim
import std/[jsonutils, json, sets]

let os = ["first", "second", "third"].toOrderedSet
echo $toJson(os)
# ["first","second","third"]  — порядок сохранён
```

---

### `fromJsonHook` для `Option[T]`

```nim
proc fromJsonHook*[T](self: var Option[T], jsonNode: JsonNode, opt = Joptions())
```

**Что делает:** Обучает `fromJson` десериализовывать JSON-значение в Nim `Option[T]`. Правило простое и интуитивное:

- JSON `null` → `none[T]()` (отсутствующее значение)
- Любое другое JSON-значение → `some(десериализованноеЗначение)` (присутствующее значение типа `T`)

Это делает `Option` естественным инструментом для работы с nullable-полями в JSON.

```nim
import std/[jsonutils, json, options]

var maybePort: Option[int]

fromJson(maybePort, parseJson("8080"))
assert maybePort == some(8080)

fromJson(maybePort, parseJson("null"))
assert maybePort == none[int]()
```

---

### `toJsonHook` для `Option[T]`

```nim
proc toJsonHook*[T](self: Option[T], opt = initToJsonOptions()): JsonNode
```

**Что делает:** Обучает `toJson` сериализовывать Nim `Option[T]` в JSON. Правило зеркально `fromJsonHook`:

- `some(value)` → JSON-представление `value`
- `none[T]()` → JSON `null`

```nim
import std/[jsonutils, json, options]

echo $toJson(some("hello"))   # "hello"
echo $toJson(none[string]())  # null

type User = object
  name: string
  nickname: Option[string]

let u = User(name: "Alice", nickname: none[string]())
echo $toJson(u)
# {"name":"Alice","nickname":null}
```

---

### `fromJsonHook` для `StringTableRef`

```nim
proc fromJsonHook*(a: var StringTableRef, b: JsonNode)
```

**Что делает:** Обучает `fromJson` десериализовывать JSON-объект в `StringTableRef`. Ожидаемый формат JSON хранит таблицу в двух частях:

```json
{
  "mode": 0,
  "table": {"key1": "value1", "key2": "value2"}
}
```

Поле `"mode"` — целое число, соответствующее значению перечисления `StringTableMode` (`modeCaseSensitive`, `modeCaseInsensitive`, `modeStyleInsensitive`). Поле `"table"` содержит непосредственно пары ключ-значение.

```nim
import std/[jsonutils, json, strtabs]

var t: StringTableRef
fromJsonHook(t, parseJson("""{"mode": 0, "table": {"HOST": "localhost"}}"""))
assert t["HOST"] == "localhost"
```

---

### `toJsonHook` для `StringTableRef`

```nim
proc toJsonHook*(a: StringTableRef): JsonNode
```

**Что делает:** Обучает `toJson` сериализовывать `StringTableRef` в JSON. Вывод следует двухпольному формату, который ожидает `fromJsonHook`: поле `"mode"` (как строковое имя режима) и поле `"table"` со всеми парами ключ-значение.

```nim
import std/[jsonutils, json, strtabs]

let t = newStringTable("lang", "nim", "version", "2", modeCaseSensitive)
let j = toJsonHook(t)
echo $j
# {"mode":"modeCaseSensitive","table":{"lang":"nim","version":"2"}}
```

---

## Как написать собственный хук

Система хуков — это то, что делает `jsonutils` расширяемым. Если вы определяете любую из следующих процедур для своего типа в любой области видимости, доступной в месте вызова, `toJson` и `fromJson` будут предпочитать её встроенной логике:

```nim
proc toJsonHook*(a: MyType): JsonNode = ...
proc toJsonHook*(a: MyType, opt: ToJsonOptions): JsonNode = ...  # с опциями

proc fromJsonHook*(a: var MyType, b: JsonNode) = ...
proc fromJsonHook*(a: var MyType, b: JsonNode, opt: Joptions) = ...  # с опциями
```

Версия с параметром `opt` имеет приоритет над версией без него, поэтому предпочтительнее использовать именно её — чтобы иметь возможность передавать опции во вложенные вызовы.

**Пример:** Сериализация пользовательского типа `Date` как строки:

```nim
import std/[jsonutils, json, strformat, strutils]

type Date = object
  year, month, day: int

proc toJsonHook*(d: Date): JsonNode =
  newJString(fmt"{d.year:04d}-{d.month:02d}-{d.day:02d}")

proc fromJsonHook*(d: var Date, b: JsonNode) =
  let parts = b.getStr().split('-')
  d = Date(year: parts[0].parseInt, month: parts[1].parseInt, day: parts[2].parseInt)

let date = Date(year: 2024, month: 3, day: 15)
let j = toJson(date)
assert $j == "\"2024-03-15\""
assert j.jsonTo(Date) == date
```

---

## Гарантия кругового обхода (round-trip)

Для полностью поддерживаемых типов без пользовательских хуков `toJson` и `jsonTo` образуют пару «туда и обратно» без потерь данных:

```nim
let original = (name: "Nim", version: (major: 2, minor: 0))
assert original.toJson.jsonTo(typeof(original)) == original
```

---

## Обработка float: специальные значения

Стандарт JSON не определяет представления для `NaN`, положительной и отрицательной бесконечности. `jsonutils` сериализует их как строки, чтобы не потерять данные, и парсит обратно симметрично:

```nim
import std/[jsonutils, json]

let values = [NaN, Inf, -Inf, 0.0, 1.5]
let j = toJson(values)
echo $j
# ["nan","inf","-inf",0.0,1.5]

let back = j.jsonTo(typeof(values))
assert back[1] == Inf
```
