# Справочник по модулю `strtabs`

> **Модуль:** `strtabs`  
> **Назначение:** Эффективная хеш-таблица «строка → строка» с поддержкой чувствительного, нечувствительного к регистру и стиленечувствительного режимов сравнения ключей. Также предоставляет оператор `%` для подстановки значений в шаблонные строки.

---

## Обзор

`strtabs` — специализированная хеш-таблица «строка → строка», более простая и лёгкая по сравнению с универсальным модулем `tables`, но с уникальной особенностью: **режим сравнения ключей** задаётся при создании и встроен в саму таблицу. Вы выбираете один раз — сравнивать ли ключи точно, без учёта регистра букв, или в «стиленечувствительном» режиме Nim, где игнорируются и регистр, и символы подчёркивания. Это делает модуль идеальным для конфигурационных файлов, поиска по переменным окружения, подстановки в шаблонах и любых сценариев с гибким написанием ключей.

Вместе с таблицей поставляется оператор `%` для подстановки `${ключ}` в строки — лёгкий встроенный шаблонизатор.

---

## Типы

### `StringTableMode`

```nim
type StringTableMode* = enum
  modeCaseSensitive,    ## точное совпадение: "Name" ≠ "name"
  modeCaseInsensitive,  ## регистр ASCII-букв игнорируется: "Name" == "name"
  modeStyleInsensitive  ## регистр И подчёркивания игнорируются: "firstName" == "first_name"
```

Режим выбирается один раз при создании таблицы и определяет поведение каждого поиска и вставки. Изменить его после создания можно только вызовом `clear`.

`modeStyleInsensitive` следует правилам сравнения идентификаторов в самом Nim: все ASCII-буквы приводятся к нижнему регистру, а символы подчёркивания удаляются перед сравнением. Это делает `"myKey"`, `"my_key"`, `"MyKey"` и `"MY_KEY"` эквивалентными.

---

### `StringTableRef`

```nim
type StringTableRef* = ref StringTableObj
```

Ссылка на объект таблицы. Все публичные функции принимают или возвращают `StringTableRef`. Поскольку это `ref`, таблица передаётся по ссылке — изменения видны всем, кто держит её копию.

---

### `FormatFlag`

```nim
type FormatFlag* = enum
  useEnvironment,  ## при отсутствии ключа использовать переменную окружения
  useEmpty,        ## вместо ошибки подставить "" для отсутствующего ключа
  useKey           ## оставить "$key" в тексте, если ключ не найден
```

Набор флагов `FormatFlag` передаётся оператору `%` для управления поведением при отсутствии ключа в таблице. Флаги комбинируются: можно использовать `{useEnvironment, useEmpty}` одновременно.

`useEnvironment` игнорируется на целевых платформах JS, NimScript и Standalone.

---

## Экспортируемые символы

| Символ | Вид | Краткое описание |
|--------|-----|-----------------|
| `newStringTable` (только режим) | proc | Создать пустую таблицу с заданным режимом |
| `newStringTable` (плоские пары) | proc | Создать из чередующихся строк ключ/значение |
| `newStringTable` (кортежи) | proc | Создать из `(key, val)`-кортежей или литерала `{k:v}` |
| `mode` | proc | Прочитать режим сравнения таблицы |
| `len` | proc | Количество хранимых пар |
| `[]` | proc | Получить значение по ключу; `KeyError` если отсутствует |
| `[]=` | proc | Вставить или перезаписать пару ключ-значение |
| `getOrDefault` | proc | Получить значение или вернуть умолчание при отсутствии |
| `hasKey` | proc | Проверить существование ключа |
| `contains` | proc | Псевдоним для `hasKey`; включает операторы `in`/`notin` |
| `del` | proc | Удалить ключ из таблицы |
| `clear` (с режимом) | proc | Очистить таблицу, при необходимости изменив режим |
| `clear` (без аргументов) | proc | Очистить таблицу, сохранив режим |
| `pairs` | итератор | Перебрать все пары `(key, value)` |
| `keys` | итератор | Перебрать все ключи |
| `values` | итератор | Перебрать все значения |
| `$` | proc | Строковое представление `{key: value, ...}` |
| `%` | proc | Подстановка в шаблон с синтаксисом `${key}`/`$key` |

---

## Создание таблиц

### `newStringTable(mode)`

```nim
proc newStringTable*(mode: StringTableMode): owned(StringTableRef)
```

Создаёт пустую таблицу с заданным режимом сравнения. Минимальный конструктор — используйте его, когда таблица заполняется вручную позже.

```nim
var t = newStringTable(modeCaseSensitive)
t["host"] = "localhost"
t["port"] = "8080"
```

---

### `newStringTable(keyValuePairs, mode)` — плоский список

```nim
proc newStringTable*(keyValuePairs: varargs[string],
                     mode: StringTableMode): owned(StringTableRef)
```

Создаёт предзаполненную таблицу из плоского списка чередующихся строк-ключей и строк-значений. Параметр `mode` **обязателен** и должен идти последним. Аргументы чередуются: аргумент 0 — ключ, аргумент 1 — значение, аргумент 2 — следующий ключ и т. д. Нечётное суммарное количество аргументов: крайний ключ без пары молча игнорируется.

```nim
var t = newStringTable("host", "localhost", "port", "8080", modeCaseSensitive)
assert t["host"] == "localhost"
assert t["port"] == "8080"
```

Эта перегрузка удобна, когда список пар строится программно или поступает от парсера конфига как плоская последовательность строк.

---

### `newStringTable(keyValuePairs, mode)` — перегрузка с кортежами

```nim
proc newStringTable*(keyValuePairs: varargs[tuple[key, val: string]],
    mode: StringTableMode = modeCaseSensitive): owned(StringTableRef)
```

Наиболее удобный конструктор для статических данных. Принимает либо массив кортежей `(key, val)`, либо литерал таблицы `{"key": "value", ...}`. Параметр `mode` опционален, по умолчанию `modeCaseSensitive`.

```nim
# Из литерала таблицы (наиболее частый случай):
var cfg = {"host": "localhost", "port": "8080"}.newStringTable

# С явным режимом:
var ci = {"name": "Alice"}.newStringTable(modeCaseInsensitive)
assert ci["NAME"] == "Alice"

# Из явного массива кортежей:
var t = newStringTable([("a", "1"), ("b", "2")])
```

---

## Чтение режима

### `mode`

```nim
proc mode*(t: StringTableRef): StringTableMode
```

Возвращает режим сравнения, с которым создана таблица. Полезен, когда функция получает `StringTableRef` извне и должна знать, как будут вести себя поиск по ключу.

```nim
var t = newStringTable(modeStyleInsensitive)
assert t.mode == modeStyleInsensitive
```

---

## Размер таблицы

### `len`

```nim
proc len*(t: StringTableRef): int
```

Возвращает количество пар ключ-значение в таблице. Пустая таблица возвращает `0`. Работает за O(1).

```nim
var t = {"a": "1", "b": "2"}.newStringTable
assert t.len == 2
t.del("a")
assert t.len == 1
```

---

## Доступ к значениям

### `[]` — Чтение по индексу

```nim
proc `[]`*(t: StringTableRef, key: string): var string
```

Ищет `key` и возвращает **изменяемую ссылку** на его значение. Если ключ отсутствует, выбрасывается `KeyError`. Поскольку возвращается `var string`, значение можно изменять на месте.

```nim
var t = {"score": "10"}.newStringTable
echo t["score"]        # → "10"
t["score"] = "20"      # то же, что []=
t["score"].add("!")    # изменение на месте → "20!"
```

Перед `[]` используйте `getOrDefault` или `hasKey`, если ключ может отсутствовать.

---

### `getOrDefault`

```nim
proc getOrDefault*(t: StringTableRef; key: string, default: string = ""): string
```

Ищет `key` и возвращает его значение. Если ключ не найден, возвращает `default` (пустую строку по умолчанию) вместо исключения. Это **безопасная** альтернатива `[]` для опциональных ключей.

```nim
var t = {"name": "John", "city": "Monaco"}.newStringTable

assert t.getOrDefault("name")              == "John"
assert t.getOrDefault("occupation")        == ""          # нет → ""
assert t.getOrDefault("occupation", "N/A") == "N/A"       # своё умолчание
assert t.getOrDefault("name", "Fallback")  == "John"      # есть → фактическое значение
```

---

### `hasKey`

```nim
proc hasKey*(t: StringTableRef, key: string): bool
```

Возвращает `true`, если `key` есть в таблице. Сравнение учитывает режим таблицы.

```nim
var t = {"name": "John"}.newStringTable(modeCaseInsensitive)
assert t.hasKey("name")
assert t.hasKey("NAME")     # нечувствительный к регистру: оба совпадают
assert not t.hasKey("age")
```

---

### `contains`

```nim
proc contains*(t: StringTableRef, key: string): bool
```

Псевдоним для `hasKey`, включающий идиоматические операторы `in` / `notin`. Предпочтительнее для условий.

```nim
var t = {"name": "John", "city": "Monaco"}.newStringTable
assert "name" in t
assert "occupation" notin t

if "city" in t:
  echo t["city"]   # → Monaco
```

---

## Изменение таблицы

### `[]=` — Вставка или обновление

```nim
proc `[]=`*(t: StringTableRef, key, val: string)
```

Вставляет новую пару `(key, value)` или перезаписывает значение, если `key` уже существует. Таблица автоматически расширяется (в 2 раза) при высокой загруженности.

```nim
var t = newStringTable(modeCaseSensitive)
t["lang"]  = "Nim"
t["lang"]  = "Nim 2"   # перезапись
t["year"]  = "2024"

assert t["lang"] == "Nim 2"
assert t.len == 2
```

---

### `del`

```nim
proc del*(t: StringTableRef, key: string)
```

Удаляет `key` и связанное с ним значение из таблицы. Если ключ отсутствует, вызов игнорируется без исключения. Алгоритм удаления (Алгоритм 6.4R Кнута) корректно перестраивает цепочки открытой адресации — все оставшиеся пары остаются доступными.

```nim
var t = {"a": "1", "b": "2", "c": "3"}.newStringTable
t.del("b")
assert t.len == 2
assert "b" notin t
assert "a" in t    # другие пары не затронуты
t.del("z")         # отсутствующий ключ — молча игнорируется
```

---

### `clear` (с режимом)

```nim
proc clear*(s: StringTableRef, mode: StringTableMode)
```

Полностью очищает таблицу и сбрасывает её до начального (малого) размера. Одновременно можно изменить режим сравнения — удобно при повторном использовании таблицы для другой предметной области.

```nim
var t = {"name": "John", "city": "Monaco"}.newStringTable
assert t.len == 2

t.clear(modeCaseInsensitive)
assert t.len == 0
assert t.mode == modeCaseInsensitive

# Теперь можно заполнить заново с новым режимом:
t["LANG"] = "Nim"
assert t["lang"] == "Nim"   # теперь нечувствительно к регистру
```

---

### `clear` (без аргументов)

```nim
proc clear*(s: StringTableRef)
```

Очищает таблицу, сохраняя текущий режим. Эквивалентно `clear(s, s.mode)`.

```nim
var t = {"a": "1"}.newStringTable(modeStyleInsensitive)
t.clear()
assert t.len == 0
assert t.mode == modeStyleInsensitive   # не изменился
```

---

## Перебор содержимого

Все три итератора прозрачно пропускают пустые внутренние слоты. Порядок перебора **не гарантирован** и не соответствует порядку вставки.

### `pairs`

```nim
iterator pairs*(t: StringTableRef): tuple[key, value: string]
```

Возвращает каждую пару `(key, value)` в таблице.

```nim
var t = {"a": "1", "b": "2", "c": "3"}.newStringTable
for key, val in t.pairs:
  echo key, " → ", val
# порядок вывода не определён; все три пары будут выведены
```

---

### `keys`

```nim
iterator keys*(t: StringTableRef): string
```

Возвращает каждый ключ в таблице. Используйте, когда нужны только имена ключей.

```nim
var t = {"host": "localhost", "port": "8080"}.newStringTable
var ks: seq[string]
for k in t.keys:
  ks.add(k)
assert "host" in ks
assert "port" in ks
```

---

### `values`

```nim
iterator values*(t: StringTableRef): string
```

Возвращает каждое значение в таблице. Используйте, когда нужны только значения.

```nim
var t = {"x": "10", "y": "20"}.newStringTable
var total = 0
for v in t.values:
  total += v.parseInt
assert total == 30
```

---

## Строковое представление

### `$`

```nim
proc `$`*(t: StringTableRef): string
```

Возвращает читаемое представление таблицы в формате `{key: value, key: value}`. Пустая таблица выводится как специальный токен `{:}` (чтобы отличить её от пустой последовательности `{}`). Порядок пар в строке не определён.

```nim
var t = {"name": "John", "city": "Monaco"}.newStringTable
echo t   # например → {name: John, city: Monaco}

var empty = newStringTable(modeCaseSensitive)
echo empty   # → {:}
```

---

## Подстановка в шаблон

### `%`

```nim
proc `%`*(f: string, t: StringTableRef, flags: set[FormatFlag] = {}): string
```

Выполняет подстановку `${key}` и `$key` в строке `f`, ища каждый ключ в таблице `t`. Превращает `StringTableRef` в лёгкий шаблонизатор.

**Синтаксис подстановки:**

| Шаблон | Результат |
|--------|-----------|
| `${key}` | Подставить значение ключа `key` |
| `$key` | Подставить значение `key` (ключ заканчивается при первом не-идентификаторном символе) |
| `$$` | Литеральный символ `$` |
| Прочее `$x` | Передаётся без изменений |

Ключ — это ASCII-буквы, цифры, `_` и байты `\x80`–`\xFF`.

**Поведение при отсутствии ключа** (управляется `flags`):

| Флаг | Поведение при отсутствии ключа |
|------|-------------------------------|
| *(нет флагов)* | Выбрасывает `ValueError` |
| `useEmpty` | Подставляет `""` |
| `useKey` | Оставляет `$key` или `${key}` в тексте буквально |
| `useEnvironment` | Ищет переменную окружения ОС; при неудаче — по другим флагам |

```nim
var t = {"name": "John", "city": "Monaco"}.newStringTable

# Базовая подстановка:
assert "${name} lives in ${city}" % t == "John lives in Monaco"
assert "$name lives in $city"    % t == "John lives in Monaco"

# Литеральный доллар:
assert "$$name is $name" % t == "$name is John"

# Отсутствующий ключ — по умолчанию ValueError:
try:
  discard "$missing" % t
except ValueError as e:
  echo e.msg   # → "format string: key not found: missing"

# Отсутствующий ключ — пустая строка:
assert "$missing" % (t, {useEmpty}) == ""

# Отсутствующий ключ — оставить токен подстановки:
assert "$missing here" % (t, {useKey}) == "$missing here"

# Откат к переменным окружения (только не-JS платформы):
import std/os
putEnv("USER", "alice")
var empty = newStringTable(modeCaseSensitive)
assert "$USER" % (empty, {useEnvironment}) == "alice"
```

---

## Режимы сравнения подробно

Режим влияет **как на запись, так и на чтение**. Если вы записываете ключ в одном написании в стиленечувствительную таблицу, поиск по любому эквивалентному написанию найдёт его.

```nim
var t = newStringTable(modeStyleInsensitive)
t["first_name"] = "John"

assert t["firstName"]  == "John"   # подчёркивание убрано, регистр сложен
assert t["FIRST_NAME"] == "John"
assert t["firstname"]  == "John"
assert t["FirstName"]  == "John"

# В таблице хранится только одна запись:
assert t.len == 1
```

В режиме `modeCaseInsensitive` регистр ASCII игнорируется, но подчёркивания значимы:

```nim
var ci = newStringTable(modeCaseInsensitive)
ci["LastName"] = "Doe"
assert ci["lastname"] == "Doe"
assert ci["LASTNAME"] == "Doe"
# "last_name" НЕ совпадёт — подчёркивание важно в case-insensitive режиме
```

---

## Типичные паттерны

### Разбор конфигурационного файла

```nim
var cfg = newStringTable(modeCaseInsensitive)
for line in configFile.lines:
  let parts = line.split('=', 1)
  if parts.len == 2:
    cfg[parts[0].strip] = parts[1].strip

let host = cfg.getOrDefault("host", "127.0.0.1")
let port = cfg.getOrDefault("port", "80")
```

### Рендеринг шаблона

```nim
let tmpl = "Уважаемый ${name},\n\nВаш заказ №${id} готов в ${city}."
var vars = {"name": "Алиса", "id": "42", "city": "Монако"}.newStringTable
echo tmpl % vars
```

### Безопасное удаление при итерации

**Не удаляйте** ключи во время перебора через `pairs`, `keys` или `values`. Сначала соберите ключи, затем удалите:

```nim
var t = {"a": "оставить", "b": "удалить", "c": "оставить"}.newStringTable
var toRemove: seq[string]
for key, val in t.pairs:
  if val == "удалить":
    toRemove.add(key)
for key in toRemove:
  t.del(key)
assert t.len == 2
```

---

## Таблица ошибок

| Ситуация | Исключение | Сообщение |
|----------|-----------|-----------|
| `t[key]` — ключ отсутствует | `KeyError` | `"key not found: <key>"` |
| `"$key" % t` — ключ отсутствует, нет флагов | `ValueError` | `"format string: key not found: <key>"` |
| Нечётное кол-во аргументов в плоском конструкторе | *(молча)* | последний ключ без пары игнорируется |
