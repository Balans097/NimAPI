# Справочник модуля `jsheaders`

> **Модуль:** `std/jsheaders` (стандартная библиотека Nim)  
> **Цель компиляции:** Только JavaScript — использование с нативным бэкендом является фатальной ошибкой компиляции.  
> **Назначение:** Nim-привязки к браузерному [`Headers`](https://developer.mozilla.org/en-US/docs/Web/API/Headers) API для построения и управления коллекциями HTTP-заголовков при использовании с `fetch()` и смежными API.

---

## Контекст: Headers API

В браузерном Fetch API HTTP-заголовки представлены не обычным словарём, а специальным объектом `Headers`. Такой дизайн отражает особенность спецификации HTTP: **имена заголовков регистронезависимы**, а **несколько значений под одним именем заголовка допустимы** и имеют специальную семантику слияния.

Например, если ответ содержит три заголовка `Set-Cookie`, все они различны и должны быть сохранены. При этом отправить несколько заголовков `Content-Type` — ошибка. `Headers` API управляет этим противоречием, предоставляя две отдельные операции записи с разной семантикой — `append` (допускает дубликаты) и `set` (обеспечивает уникальность), — которые модуль предоставляет как `add` и `[]=` соответственно.

**Ключевое поведение, которое нужно понять до начала работы с модулем:**

- `add` (→ `Headers.append`) **накапливает** значения — существующие записи для того же ключа сохраняются, новое значение добавляется рядом.
- `[]=` (→ `Headers.set`) **заменяет** — все существующие записи для ключа удаляются и заменяются одним новым значением.
- `[]` (→ `Headers.get`) возвращает все значения для ключа, **объединённые через `", "`** (запятая и пробел) в единую строку. Это определено спецификацией HTTP, а не данным модулем.
- Имена заголовков **автоматически нормализуются к нижнему регистру** браузерной реализацией `Headers`.

---

## Тип `Headers`

```nim
type Headers* = ref object of JsRoot
```

Ref-объект, оборачивающий JavaScript-объект `Headers`. Передаётся по ссылке, собирается сборщиком мусора. Все мутации выполняются на месте над базовым JavaScript-объектом.

---

## Экспортируемые функции

---

### `newHeaders`

```nim
func newHeaders*(): Headers
```

**Что делает**

Создаёт новый пустой объект `Headers`, вызывая `new Headers()` в JavaScript. Результирующий объект не содержит записей и готов к заполнению.

Объекты `Headers` используются как для исходящих запросов (передаются в `newFetchOptions` или `newRequest`), так и присутствуют на входящих объектах `Response` после завершения вызова `fetch()`.

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
assert h.len == 0
```

---

### `add`

```nim
func add*(self: Headers; key: cstring; value: cstring)
```

**Что делает**

Добавляет новую пару `key: value` в `Headers`, вызывая JavaScript `Headers.append(key, value)`. Если запись для `key` уже существует, новое значение добавляется **рядом** с существующим — оно не заменяется. Оба сосуществуют и будут объединены при получении через `[]`.

Это заголовочный эквивалент «добавить ещё одно значение к этому полю». Используйте его для заголовков, которые законно несут несколько значений (например, `Accept`, `Set-Cookie`). Для заголовков, которые должны быть уникальными (например, `Content-Type`, `Authorization`), вместо этого используйте `[]=`, чтобы избежать непреднамеренного накопления.

Имена заголовков регистронезависимы — `"Content-Type"` и `"content-type"` относятся к одному заголовку, браузер нормализует их к нижнему регистру.

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
h.add("Accept", "text/html".cstring)
h.add("Accept", "application/json".cstring)   # НЕ заменяет text/html

assert h["Accept"] == "text/html, application/json".cstring
# Несколько значений объединяются через ", " — поведение по спецификации HTTP
```

---

### `delete`

```nim
func delete*(self: Headers; key: cstring)
```

**Что делает**

Удаляет **все** записи, ключ которых совпадает с `key`, вызывая JavaScript `Headers.delete(key)`. Поскольку `add` может создать несколько записей для одного ключа, эта операция удаляет всю группу сразу. Удаления одной отдельной записи не существует.

Если `key` не существует — вызов не вызывает ошибок (no-op).

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
h.add("Accept", "text/html".cstring)
h.add("Accept", "application/json".cstring)   # 2 значения Accept

h.delete("Accept")   # удаляет ОБА
assert not h.hasKey("Accept")
assert h.len == 0
```

---

### `hasKey`

```nim
func hasKey*(self: Headers; key: cstring): bool
```

**Что делает**

Возвращает `true`, если существует хотя бы одна запись с заданным ключом, вызывая JavaScript `Headers.has(key)`. Проверка регистронезависима — `"content-type"` и `"Content-Type"` трактуются как один заголовок.

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
h["Authorization"] = "Bearer token123".cstring

assert h.hasKey("Authorization") == true
assert h.hasKey("authorization") == true    # регистронезависимо
assert h.hasKey("Content-Type") == false
```

---

### `keys`

```nim
func keys*(self: Headers): seq[cstring]
```

**Что делает**

Возвращает `seq[cstring]` всех имён заголовков в объекте `Headers` в порядке добавления, приведённых к нижнему регистру. Оборачивает `Array.from(headers.keys())`.

Браузерная реализация `Headers` автоматически нормализует имена заголовков к нижнему регистру — поэтому даже если вы добавили `"Content-Type"`, возвращаемый ключ будет `"content-type"`.

В отличие от `FormData.keys()`, дублирующиеся имена заголовков в `Headers` **объединяются** браузером на уровне хранения — поэтому даже если вы вызвали `add` с одним ключом дважды, он всё равно появляется в `keys()` один раз. Несколько значений доступны только через `[]` (которая их объединяет).

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
h["Content-Type"] = "application/json".cstring
h["Authorization"] = "Bearer abc".cstring

assert h.keys() == @["content-type".cstring, "authorization".cstring]
# Обратите внимание: нормализованы к нижнему регистру
```

---

### `values`

```nim
func values*(self: Headers): seq[cstring]
```

**Что делает**

Возвращает `seq[cstring]` всех значений заголовков в порядке добавления, по одному на ключ. Оборачивает `Array.from(headers.values())`. Если ключ имел несколько значений, добавленных через `add`, они появляются как единая строка с запятыми в этой последовательности (браузер объединяет их при хранении).

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
h["Content-Type"] = "application/json".cstring
h.add("Accept", "text/html".cstring)
h.add("Accept", "application/json".cstring)

let vs = h.values()
# Несколько значений "Accept" хранятся объединёнными:
assert "text/html" in $vs[1]
assert "application/json" in $vs[1]
```

---

### `entries`

```nim
func entries*(self: Headers): seq[tuple[key, value: cstring]]
```

**Что делает**

Возвращает `seq` кортежей `(key, value)` для всех записей `Headers` в порядке добавления, с ключами в нижнем регистре. Оборачивает `Array.from(headers.entries())`. Это наиболее полное представление заголовков — ключи и значения вместе. Ключи с несколькими значениями появляются как одна запись с объединённой через запятую строкой значений.

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
h["Content-Type"] = "application/json".cstring
h["Accept"] = "text/html".cstring

let e = h.entries()
assert e == @[
  ("content-type".cstring, "application/json".cstring),
  ("accept".cstring, "text/html".cstring)
]
```

---

### `[]` — получить (все значения, объединённые через запятую)

```nim
func `[]`*(self: Headers; key: cstring): cstring
```

**Что делает**

Возвращает значение, связанное с `key`, вызывая JavaScript `Headers.get(key)`. Поиск регистронезависим.

**Ключевая деталь:** если одно имя заголовка было добавлено несколько раз через `add`, все значения возвращаются в виде **единого `cstring`**, с отдельными значениями, объединёнными через `", "` (запятая и пробел). Это определено спецификацией HTTP — заголовки с одним именем семантически эквивалентны единому заголовку со списком значений, разделённых запятыми (за исключением `Set-Cookie`, который браузер обрабатывает особым образом).

Если `key` не существует, возвращает `nil`.

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
h.add("Accept", "text/html".cstring)
h.add("Accept", "application/json".cstring)
h.add("Accept", "image/webp".cstring)

assert h["Accept"] == "text/html, application/json, image/webp".cstring
# Все три значения объединены через ", "

assert h["X-Missing"] == nil   # несуществующий ключ
```

---

### `[]=` — установить (заменить)

```nim
func `[]=`*(self: Headers; key: cstring; value: cstring)
```

**Что делает**

Устанавливает заголовок `key` в `value`, вызывая JavaScript `Headers.set(key, value)`. Если записи для `key` уже существуют (включая дубликаты, созданные через `add`), они **все удаляются** и заменяются этой одной записью. Последующие чтения `h[key]` вернут именно это `value`.

Это правильная операция для заголовков, которые должны встречаться только один раз — `Content-Type`, `Authorization`, `Cache-Control` и т. д.

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
h.add("X-Custom", "first".cstring)
h.add("X-Custom", "second".cstring)   # теперь 2 значения

assert h["X-Custom"] == "first, second".cstring

h["X-Custom"] = "only".cstring        # заменяет ОБА
assert h["X-Custom"] == "only".cstring
```

---

### `clear`

```nim
func clear*(self: Headers)
```

**Что делает**

Удаляет все записи из объекта `Headers`, оставляя его пустым. Вспомогательная функция, реализованная в Nim — веб-спецификация `Headers` не имеет метода `clear()`. Внутри она перебирает все текущие ключи и вызывает `delete` для каждого, в результате чего объект становится пустым.

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
h["Content-Type"] = "application/json".cstring
h["Authorization"] = "Bearer abc".cstring
h["Accept"] = "text/html".cstring

assert h.len == 3
h.clear()
assert h.len == 0
assert h.entries() == @[]
```

---

### `toCstring`

```nim
func toCstring*(self: Headers): cstring
```

**Что делает**

Сериализует объект `Headers` в JSON-`cstring`. Внутри вызывает `JSON.stringify(Array.from(headers.entries()))`, создавая JSON-массив пар `[ключ, значение]`. Полезно для логирования, отладки и быстрого просмотра заголовков.

Формат вывода — JSON-массив двухэлементных массивов, например: `[["content-type","application/json"],["authorization","Bearer xyz"]]`.

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
h["key"] = "value".cstring
h["other"] = "another".cstring

assert h.toCstring() == """[["key","value"],["other","another"]]""".cstring
```

---

### `$`

```nim
func `$`*(self: Headers): string
```

**Что делает**

Оператор `$` (stringify) Nim для `Headers`. Возвращает Nim-строку с JSON-представлением всех записей. Тонкая обёртка над `toCstring`.

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
h["Content-Type"] = "application/json".cstring
echo $h   # [["content-type","application/json"]]
```

---

### `len`

```nim
func len*(self: Headers): int
```

**Что делает**

Возвращает количество **различных имён заголовков** в объекте `Headers`. Поскольку браузерная реализация `Headers` объединяет значения для одного ключа в единую запись (доступную через `[]` с объединёнными через запятую значениями), `len` считает уникальные ключи, а не количество вызовов `add`.

Это важное отличие от `FormData.len`, где дубликаты считаются отдельно. В `Headers` добавление одного ключа три раза через `add` даёт `len == 1`, потому что внутри они хранятся как один ключ с объединённым значением.

Реализовано через `Array.from(headers.entries()).length`.

**Пример**

```nim
import std/jsheaders

let h = newHeaders()
assert h.len == 0

h.add("Accept", "text/html".cstring)
h.add("Accept", "application/json".cstring)   # дублирующийся ключ
assert h.len == 1   # всё равно 1 — объединён в единую запись

h["Authorization"] = "Bearer xyz".cstring
assert h.len == 2   # 2 различных имени заголовка
```

---

## `add` против `[]=`: сравнение бок о бок

Та же ситуация, что и в `FormData`, но с одной принципиальной дополнительной деталью: объект `Headers` **объединяет** дублирующиеся значения в строку с запятыми, а не хранит их как отдельные записи. Это влияет на то, что возвращает `[]` и что считает `len`.

```nim
import std/jsheaders

let h = newHeaders()

# --- add ---
h.add("Accept", "text/html".cstring)
h.add("Accept", "image/webp".cstring)
assert h["Accept"] == "text/html, image/webp".cstring  # объединено, не список
assert h.len == 1                                       # ОДНА объединённая запись

# --- []= ---
h["Accept"] = "application/json".cstring   # заменяет объединённую запись
assert h["Accept"] == "application/json".cstring
assert h.len == 1                          # всё равно 1, теперь с одним значением
```

**Сравнение с `FormData`:**

| Поведение | `Headers` | `FormData` |
|-----------|-----------|------------|
| Хранение дублирующегося ключа | Объединяется в одно значение через запятую | Хранится как отдельные записи |
| `len` при 3 вызовах `add` | `1` | `3` |
| Получить все значения | `[]` возвращает `cstring` с запятыми | `getAll` возвращает `seq[cstring]` |
| `keys()` при дубликатах | Ключ появляется один раз | Ключ появляется несколько раз |

---

## Практические примеры

### Установка заголовков для запроса к JSON API

```nim
import std/[jsheaders, jsfetch, asyncjs]
from std/httpcore import HttpPost

proc callApi(token: cstring; payload: cstring) {.async.} =
  let h = newHeaders()
  h["Content-Type"] = "application/json"
  h["Authorization"] = "Bearer " & $token
  h["Accept"] = "application/json"

  let opts = newFetchOptions(metod = HttpPost, body = payload, headers = h)
  let resp = await fetch("https://api.example.com/data".cstring, opts)
  echo "Статус: ", $resp.status
```

### Просмотр заголовков ответа

```nim
import std/[jsfetch, asyncjs]

proc checkHeaders() {.async.} =
  let resp = await fetch("https://httpbin.org/get".cstring)
  let h = resp.headers

  if h.hasKey("Content-Type"):
    echo "Content-Type: ", $h["Content-Type"]

  echo "Все заголовки:"
  for (k, v) in h.entries():
    echo "  ", $k, ": ", $v
```

### Формирование заголовка Accept с несколькими MIME-типами

```nim
import std/jsheaders

let h = newHeaders()
h.add("Accept", "text/html".cstring)
h.add("Accept", "application/xhtml+xml".cstring)
h.add("Accept", "application/xml;q=0.9".cstring)
h.add("Accept", "*/*;q=0.8".cstring)

echo $h["Accept"]
# "text/html, application/xhtml+xml, application/xml;q=0.9, */*;q=0.8"
```

---

## Сводная таблица

| Символ | Вид | JS-эквивалент | Назначение |
|--------|-----|--------------|------------|
| `Headers` | тип | `Headers` | Коллекция HTTP-заголовков |
| `newHeaders()` | func | `new Headers()` | Создать пустой объект `Headers` |
| `add(key, value)` | func | `.append(key, value)` | **Добавить** — добавляет значение, объединяет с существующим |
| `delete(key)` | func | `.delete(key)` | Удалить запись для ключа (все объединённые значения) |
| `hasKey(key)` | func | `.has(key)` | Проверить существование ключа (регистронезависимо) |
| `keys()` | func | `Array.from(.keys())` | Все имена заголовков, в нижнем регистре, по порядку |
| `values()` | func | `Array.from(.values())` | Все значения заголовков (объединённые), по порядку |
| `entries()` | func | `Array.from(.entries())` | Все кортежи `(key, value)` по порядку |
| `h[key]` | func | `.get(key)` | Получить значение (несколько значений объединены через `", "`) |
| `h[key] = value` | func | `.set(key, value)` | **Заменить** — перезаписывает все значения для ключа |
| `clear()` | func | *(удобство Nim)* | Удалить все записи |
| `toCstring()` | func | `JSON.stringify(Array.from(.entries()))` | JSON-сериализация в `cstring` |
| `$` | func | *(Nim)* | JSON-сериализация в `string` |
| `len` | func | `Array.from(.entries()).length` | Количество различных имён заголовков |
