# Модуль Nim `httpcore` — полный справочник

> **Назначение модуля:** Общие HTTP-примитивы, используемые как в `httpclient`, так и в `asynchttpserver`. Определяет типы, константы и вспомогательные функции, которые формируют словарь HTTP в Nim: коды статусов, методы, версии протокола и заголовки.
>
> **Стабильность API:** Помечен как *нестабильный* — публичный интерфейс может меняться между релизами Nim.
>
> **Ключевые архитектурные решения:**
> - `HttpCode` — это `distinct`-целое, что исключает случайное смешение с обычным `int`.
> - `HttpHeaders` по умолчанию хранит все ключи в нижнем регистре (или в Title-Case при явном запросе), отражая спецификацию HTTP/1.1, согласно которой имена заголовков регистронезависимы.
> - У одного заголовочного ключа может быть **несколько значений** — тип хранения `seq[string]` на ключ — что требуется спецификацией HTTP (например, несколько строк `Set-Cookie`).

---

## Содержание

1. [Типы](#типы)
   - [`HttpHeaders`](#httpheaders)
   - [`HttpHeaderValues`](#httpheadervalues)
   - [`HttpCode`](#httpcode)
   - [`HttpVersion`](#httpversion)
   - [`HttpMethod`](#httpmethod)
2. [Константы](#константы)
   - [`httpNewLine`](#httpnewline)
   - [`headerLimit`](#headerlimit)
   - [Константы HTTP-кодов статуса](#константы-http-кодов-статуса)
3. [HttpHeaders — создание](#httpheaders--создание)
   - [`newHttpHeaders` (пустой)](#newhttpheaders-пустой)
   - [`newHttpHeaders` (из пар)](#newhttpheaders-из-пар)
4. [HttpHeaders — чтение](#httpheaders--чтение)
   - [`[]` (только ключ)](#-только-ключ)
   - [`[]` (ключ + индекс)](#-ключ--индекс)
   - [`getOrDefault`](#getordefault)
   - [`hasKey`](#haskey)
   - [`contains` (HttpHeaderValues)](#contains-httpheadervalues)
   - [`len`](#len)
   - [`pairs`](#pairs)
   - [`$` (HttpHeaders)](#-httpheaders)
5. [HttpHeaders — запись](#httpheaders--запись)
   - [`[]=` (одно значение)](#--одно-значение)
   - [`[]=` (список значений)](#--список-значений)
   - [`add`](#add)
   - [`del`](#del)
   - [`clear`](#clear)
6. [HttpHeaders — внутреннее](#httpheaders--внутреннее)
   - [`toCaseInsensitive`](#tocaseinsensitive)
7. [HttpCode — утилиты](#httpcode--утилиты)
   - [`$` (HttpCode)](#-httpcode)
   - [`==` (HttpCode)](#-httpcode-1)
   - [`is1xx`](#is1xx)
   - [`is2xx`](#is2xx)
   - [`is3xx`](#is3xx)
   - [`is4xx`](#is4xx)
   - [`is5xx`](#is5xx)
8. [HttpVersion — утилиты](#httpversion--утилиты)
   - [`==` (кортеж протокола vs HttpVersion)](#-кортеж-протокола-vs-httpversion)
9. [HttpMethod — утилиты](#httpmethod--утилиты)
   - [`contains` (множество HttpMethod)](#contains-множество-httpmethod)
10. [Парсинг](#парсинг)
    - [`parseHeader`](#parseheader)
11. [Конвертер](#конвертер)
    - [`toString`](#tostring)

---

## Типы

### `HttpHeaders`

```nim
type HttpHeaders* = ref object
  table*: TableRef[string, seq[string]]
  isTitleCase: bool
```

Центральный контейнер для HTTP-заголовков. Оборачивает хэш-таблицу, отображающую имена заголовков (строки) в **списки** значений (`seq[string]`), поскольку спецификация HTTP допускает повторение одного и того же имени заголовка несколько раз (например, несколько строк `Set-Cookie` или `Accept-Encoding`).

Доступны два режима нормализации:
- **Режим нижнего регистра** (по умолчанию) — все ключи хранятся в нижнем регистре. Поиск тоже всегда нормализуется, обеспечивая надёжную регистронезависимость.
- **Режим Title-Case** (`titleCase=true`) — ключи хранятся в HTTP Title-Case (например, `Content-Type`, `X-My-Header`). Полезно, когда сервер или прокси downstream нестандартно чувствителен к регистру.

Поле `table` экспортировано — при необходимости вы можете работать с хранилищем напрямую: итерировать или выполнять массовые операции.

```nim
let h = newHttpHeaders()
h["Content-Type"] = "application/json"
h["content-type"] = "text/html"   # тот же ключ — заменяет!
echo h["CONTENT-TYPE"]             # "text/html" (регистронезависимо)
```

---

### `HttpHeaderValues`

```nim
type HttpHeaderValues* = distinct seq[string]
```

Тонкая обёртка вокруг `seq[string]`, возвращаемая оператором `[]` с одним ключом. `distinct` предотвращает случайное использование как обычного `seq`. Встроенный **конвертер** `toString` автоматически извлекает первое значение, когда результат передаётся туда, где ожидается обычная `string`, — это делает типичный доступ к одному значению удобным:

```nim
let h = newHttpHeaders({"Content-Length": "42"})
let s: string = h["Content-Length"]   # конвертер срабатывает: s == "42"
```

Чтобы получить все значения, приведите обратно к `seq[string]`:

```nim
let all = seq[string](h["Accept"])
```

---

### `HttpCode`

```nim
type HttpCode* = distinct range[0 .. 599]
```

Типобезопасный код статуса HTTP. Лежащее в основе целое число ограничено диапазоном `0..599` на уровне типов. `distinct` означает, что нельзя случайно передать обычное целое туда, где ожидается `HttpCode`, и наоборот.

Для всех стандартных кодов предоставлены именованные константы (например, `Http200`, `Http404`). Можно также сконструировать напрямую: `HttpCode(418)`.

```nim
let code = Http404
echo code           # "404 Not Found"
echo code.int       # 404
echo code.is4xx     # true
```

---

### `HttpVersion`

```nim
type HttpVersion* = enum
  HttpVer11,   # HTTP/1.1
  HttpVer10    # HTTP/1.0
```

Перечисляет две версии HTTP, которые распознаёт этот модуль. Используется внутри `httpclient` и `asynchttpserver` для отслеживания версии протокола соединения.

```nim
let ver = HttpVer11
```

---

### `HttpMethod`

```nim
type HttpMethod* = enum
  HttpHead, HttpGet, HttpPost, HttpPut, HttpDelete,
  HttpTrace, HttpOptions, HttpConnect, HttpPatch
```

Все девять стандартных методов HTTP-запросов. Строковое представление каждого варианта (например, `"GET"`, `"POST"`) точно соответствует формату на проводе: `$HttpGet == "GET"`.

| Вариант | Строка | Типичное применение |
|---|---|---|
| `HttpGet` | `GET` | Получить ресурс |
| `HttpPost` | `POST` | Отправить данные, создать ресурс |
| `HttpPut` | `PUT` | Заменить ресурс |
| `HttpPatch` | `PATCH` | Частично изменить ресурс |
| `HttpDelete` | `DELETE` | Удалить ресурс |
| `HttpHead` | `HEAD` | Как GET, но без тела ответа |
| `HttpOptions` | `OPTIONS` | Узнать поддерживаемые методы |
| `HttpConnect` | `CONNECT` | Открыть TCP-туннель (прокси) |
| `HttpTrace` | `TRACE` | Эхо запроса (отладка) |

```nim
echo $HttpPost   # "POST"
let m = parseEnum[HttpMethod]("DELETE")  # HttpDelete
```

---

## Константы

### `httpNewLine`

```nim
const httpNewLine* = "\c\L"
```

Ограничитель строки HTTP: возврат каретки плюс перевод строки (`\r\n`). HTTP/1.x требует именно эту двухбайтовую последовательность в конце каждой строки заголовка и в пустой строке, разделяющей заголовки и тело. Всегда используйте эту константу вместо `"\n"` при построении сырых HTTP-сообщений.

```nim
let raw = "HTTP/1.1 200 OK" & httpNewLine &
          "Content-Length: 0" & httpNewLine &
          httpNewLine
```

---

### `headerLimit`

```nim
const headerLimit* = 10_000
```

Максимальное количество **байт**, которые обработает парсер заголовков. Этот предел защищает от атак исчерпания памяти (HTTP request smuggling, медленные заголовочные флуды). Если блок заголовков превышает этот лимит, парсер в `asynchttpserver` прекращает чтение и отклоняет запрос.

---

### Константы HTTP-кодов статуса

```nim
const Http200* = HttpCode(200)
const Http404* = HttpCode(404)
# ... и так далее для каждого стандартного кода
```

Именованные константы предоставлены для всех зарегистрированных кодов статуса HTTP от 100 до 511. Они существуют исключительно ради читаемости — `Http404` яснее, чем `HttpCode(404)` в логике маршрутизации или формирования ответа.

```nim
if response.code == Http301:
  let location = response.headers["Location"]
  redirect(location)
```

Полный список охватывает семейства 1xx (Информационные), 2xx (Успех), 3xx (Перенаправление), 4xx (Ошибки клиента) и 5xx (Ошибки сервера), включая расширения WebDAV (102, 207, 208, 423, 424, 507, 508), Early Hints (103), Delta Encoding (226) и другие.

---

## HttpHeaders — создание

### `newHttpHeaders` (пустой)

```nim
func newHttpHeaders*(titleCase = false): HttpHeaders
```

**Создаёт пустой объект `HttpHeaders`.** Внутренняя таблица не имеет записей. Передайте `titleCase = true`, если ключи должны храниться и сериализоваться в HTTP Title-Case, а не в нижнем регистре.

```nim
let h = newHttpHeaders()                   # ключи в нижнем регистре
let h2 = newHttpHeaders(titleCase = true)  # ключи в Title-Case
h2["content-type"] = "text/plain"
echo h2["content-type"]  # внутри хранится как "Content-Type"
```

---

### `newHttpHeaders` (из пар)

```nim
func newHttpHeaders*(keyValuePairs: openArray[tuple[key: string, val: string]],
                     titleCase = false): HttpHeaders
```

**Создаёт предзаполненный объект `HttpHeaders`** из массива пар `(ключ, значение)`. Дубликаты ключей обрабатываются корректно: если один и тот же ключ встречается несколько раз во входном массиве, все значения накапливаются по порядку — ни одно не теряется молча.

```nim
let h = newHttpHeaders({
  "Accept": "text/html",
  "Accept": "application/json",   # второй Accept — оба сохраняются
  "Content-Type": "text/plain"
})
echo seq[string](h["Accept"])   # @["text/html", "application/json"]
```

---

## HttpHeaders — чтение

### `[]` (только ключ)

```nim
func `[]`*(headers: HttpHeaders, key: string): HttpHeaderValues
```

**Возвращает все значения для данного ключа заголовка** в виде `HttpHeaderValues` (обёрнутый `seq[string]`). Поиск всегда регистронезависим, независимо от того, как ключ был сохранён. Если ключ не существует, выбрасывается `KeyError`.

Когда результат используется в контексте `string`, неявный **конвертер** возвращает только первое значение. Для явного доступа ко всем значениям приведите к `seq[string]` или используйте двухаргументную форму ниже.

```nim
let h = newHttpHeaders({"Accept": "text/html", "Accept": "application/json"})
let first: string = h["Accept"]             # "text/html" через конвертер
let all = seq[string](h["Accept"])          # @["text/html", "application/json"]
```

---

### `[]` (ключ + индекс)

```nim
func `[]`*(headers: HttpHeaders, key: string, i: int): string
```

**Возвращает `i`-е значение** (нумерация с нуля) для данного ключа заголовка. Выбрасывает `KeyError`, если ключ не существует, и `IndexDefect`, если `i` выходит за пределы.

```nim
let h = newHttpHeaders({"Accept": "text/html", "Accept": "application/json"})
echo h["Accept", 0]   # "text/html"
echo h["Accept", 1]   # "application/json"
```

---

### `getOrDefault`

```nim
func getOrDefault*(headers: HttpHeaders, key: string,
                   default = @[""].HttpHeaderValues): HttpHeaderValues
```

**Безопасная версия `[]`** — возвращает значения для `key`, если он существует, или `default` — если нет. В отличие от `[]`, никогда не выбрасывает исключение для отсутствующего ключа. По умолчанию возвращается `@[""]` (одна пустая строка в `HttpHeaderValues`), которая преобразуется в пустую строку через конвертер `toString`.

```nim
let h = newHttpHeaders()
let ct: string = h.getOrDefault("Content-Type")   # "" (без исключения)
let custom = h.getOrDefault("X-My-Header", @["fallback"].HttpHeaderValues)
```

---

### `hasKey`

```nim
func hasKey*(headers: HttpHeaders, key: string): bool
```

**Возвращает `true`**, если заголовок с данным ключом существует (регистронезависимо). Используйте это перед доступом через `[]`, когда исключение недопустимо.

```nim
if headers.hasKey("Authorization"):
  let token = headers["Authorization"]
  authenticate(token)
```

---

### `contains` (HttpHeaderValues)

```nim
func contains*(values: HttpHeaderValues, value: string): bool
```

**Проверяет, присутствует ли конкретная строка** среди значений коллекции `HttpHeaderValues`. Сравнение **регистронезависимо**, что соответствует семантике HTTP (например, `"gzip"` и `"GZIP"` — одна и та же кодировка).

```nim
let h = newHttpHeaders({"Accept-Encoding": "gzip, deflate"})
# Примечание: после парсинга заголовок хранится как два отдельных значения
if "gzip" in h["Accept-Encoding"]:
  echo "Клиент принимает gzip"
```

---

### `len`

```nim
func len*(headers: HttpHeaders): int
```

**Возвращает количество различных ключей заголовков**, хранящихся в данный момент. Обратите внимание: считаются ключи, а не суммарное число значений — заголовок с тремя значениями `Accept` всё равно даёт 1 в `len`.

```nim
let h = newHttpHeaders({"A": "1", "B": "2"})
echo h.len   # 2
```

---

### `pairs`

```nim
iterator pairs*(headers: HttpHeaders): tuple[key, value: string]
```

**Итерирует по каждой отдельной паре ключ-значение**, выдавая по одному кортежу `(key, value)` за раз. Если у ключа несколько значений — он выдаётся по одному разу на каждое значение. Это правильный итератор для сериализации заголовков в проводной формат.

```nim
let h = newHttpHeaders({"Accept": "text/html", "Accept": "application/json",
                         "Content-Type": "text/plain"})
for key, val in h.pairs:
  echo key, ": ", val
# Accept: text/html
# Accept: application/json
# Content-Type: text/plain
```

---

### `$` (HttpHeaders)

```nim
func `$`*(headers: HttpHeaders): string
```

**Преобразует объект заголовков в его строковое представление** — идентично вызову `$` на лежащем в основе `TableRef`. Полезно для отладки; вывод — синтаксис Nim-таблицы, а не корректный блок HTTP-заголовков.

```nim
let h = newHttpHeaders({"X-Foo": "bar"})
echo h   # {"x-foo": @["bar"]}
```

---

## HttpHeaders — запись

### `[]=` (одно значение)

```nim
proc `[]=`*(headers: HttpHeaders, key, value: string)
```

**Устанавливает заголовок ровно в одно значение**, полностью заменяя все существующие значения для этого ключа. Если нужно сохранить существующие значения и добавить новое, используйте `add`.

```nim
let h = newHttpHeaders()
h["Content-Type"] = "text/html"
h["Content-Type"] = "application/json"  # заменяет предыдущее
echo h["Content-Type"]  # "application/json"
```

---

### `[]=` (список значений)

```nim
proc `[]=`*(headers: HttpHeaders, key: string, value: seq[string])
```

**Устанавливает заголовок сразу в список значений**, заменяя все существующие записи. Если переданная последовательность `value` **пуста** — ключ заголовка полностью удаляется из таблицы. Это удобный способ удалить заголовок.

```nim
let h = newHttpHeaders()
h["Accept"] = @["text/html", "application/json"]
echo h["Accept", 0]   # "text/html"

h["Accept"] = @[]     # удаляет заголовок "Accept"
echo h.hasKey("Accept")  # false
```

---

### `add`

```nim
proc add*(headers: HttpHeaders, key, value: string)
```

**Добавляет значение** к списку значений для `key`. Если ключ ещё не существует — создаётся с этим единственным значением. В отличие от `[]=`, существующие значения сохраняются. Именно так представляются многозначные заголовки вроде `Set-Cookie`.

```nim
let h = newHttpHeaders()
h.add("Set-Cookie", "session=abc; HttpOnly; Secure")
h.add("Set-Cookie", "lang=en")
echo seq[string](h["Set-Cookie"])
# @["session=abc; HttpOnly; Secure", "lang=en"]
```

---

### `del`

```nim
proc del*(headers: HttpHeaders, key: string)
```

**Удаляет все значения**, связанные с данным ключом. Если ключ не существует — ничего не происходит (без исключения). Поиск регистронезависим.

```nim
let h = newHttpHeaders({"Authorization": "Bearer token123"})
h.del("Authorization")
echo h.hasKey("Authorization")  # false
```

---

### `clear`

```nim
proc clear*(headers: HttpHeaders)
```

**Удаляет все записи** из объекта заголовков, приводя его в то же состояние, что и свежесозданный пустой `HttpHeaders`. Сам объект остаётся валидным и может быть использован повторно.

```nim
let h = newHttpHeaders({"X-A": "1", "X-B": "2"})
h.clear()
echo h.len   # 0
```

---

## HttpHeaders — внутреннее

### `toCaseInsensitive`

```nim
func toCaseInsensitive*(headers: HttpHeaders, s: string): string
```

**Нормализует строку ключа** в соответствии с режимом объекта заголовков: возвращает версию в нижнем регистре в режиме по умолчанию или в Title-Case при `isTitleCase = true`. Документирован как «только для внутреннего использования» — напрямую вы его, как правило, не вызываете, но понимание его работы проясняет, почему доступ к заголовкам регистронезависим.

---

## HttpCode — утилиты

### `$` (HttpCode)

```nim
func `$`*(code: HttpCode): string
```

**Преобразует `HttpCode` в полную строку HTTP-статуса**, включая числовой код и стандартную фразу-причину. Это формат, ожидаемый на проводе и в журналах.

```nim
echo $Http200   # "200 OK"
echo $Http404   # "404 Not Found"
echo $Http418   # "418 I'm a teapot"
echo $HttpCode(999)  # "999"  (неизвестные коды — только число)
```

Неизвестные коды (не входящие в стандартную таблицу) отображаются лишь как числовая строка без фразы-причины.

---

### `==` (HttpCode)

```nim
func `==`*(a, b: HttpCode): bool
```

**Сравнение на равенство** двух значений `HttpCode` (заимствовано от лежащего в основе целого). Позволяет писать естественные сравнения вида `response.code == Http200`.

```nim
doAssert Http200 == HttpCode(200)
doAssert Http404 != Http500
```

---

### `is1xx`

```nim
func is1xx*(code: HttpCode): bool
```

**Возвращает `true`**, если код находится в диапазоне 100–199 (Информационные ответы). Они означают, что сервер получил запрос и клиент должен продолжать.

```nim
doAssert is1xx(Http100)   # Continue
doAssert is1xx(Http103)   # Early Hints
doAssert not is1xx(Http200)
```

---

### `is2xx`

```nim
func is2xx*(code: HttpCode): bool
```

**Возвращает `true`**, если код в диапазоне 200–299 (Успешные ответы). Самое важное семейство: запрос был получен, понят и принят.

```nim
if response.code.is2xx:
  processBody(response.body)
```

---

### `is3xx`

```nim
func is3xx*(code: HttpCode): bool
```

**Возвращает `true`**, если код в диапазоне 300–399 (Перенаправление). Клиент должен предпринять дополнительное действие — как правило, перейти по URL из заголовка `Location`.

```nim
if response.code.is3xx:
  let newUrl = response.headers["Location"]
  followRedirect(newUrl)
```

---

### `is4xx`

```nim
func is4xx*(code: HttpCode): bool
```

**Возвращает `true`**, если код в диапазоне 400–499 (Ошибки клиента). Запрос содержит синтаксическую ошибку или не может быть выполнен. Проблема на стороне клиента.

```nim
if response.code.is4xx:
  echo "Ошибка запроса: ", $response.code
```

---

### `is5xx`

```nim
func is5xx*(code: HttpCode): bool
```

**Возвращает `true`**, если код в диапазоне 500–599 (Ошибки сервера). Сервер не смог выполнить корректный запрос. Проблема на стороне сервера.

```nim
if response.code.is5xx:
  retryWithBackoff()
```

---

## HttpVersion — утилиты

### `==` (кортеж протокола vs HttpVersion)

```nim
func `==`*(protocol: tuple[orig: string, major, minor: int],
           ver: HttpVersion): bool
```

**Сравнивает распарсенный дескриптор протокола в виде кортежа** (возвращаемый HTTP-парсерами Nim в форме `(orig: "HTTP/1.1", major: 1, minor: 1)`) со значением перечисления `HttpVersion`. Позволяет писать читаемые проверки версии без ручного сравнения чисел major/minor.

```nim
let proto = (orig: "HTTP/1.1", major: 1, minor: 1)
doAssert proto == HttpVer11
doAssert not (proto == HttpVer10)
```

---

## HttpMethod — утилиты

### `contains` (множество HttpMethod)

```nim
func contains*(methods: set[HttpMethod], x: string): bool
```

**Проверяет, принадлежит ли строка с именем метода** (например, `"GET"`, `"POST"`) заданному множеству значений `HttpMethod`. Это включает оператор `in` со строковым именем метода слева, что удобно при маршрутизации запросов.

```nim
let readOnly = {HttpGet, HttpHead, HttpOptions}
if "POST" in readOnly:
  echo "не будет выведено"
if "GET" in readOnly:
  echo "GET — безопасный метод"   # выведет это
```

---

## Парсинг

### `parseHeader`

```nim
func parseHeader*(line: string): tuple[key: string, value: seq[string]]
```

**Парсит одну сырую строку HTTP-заголовка** в кортеж `(key, value)`. Ключ — всё до первого `:`, значения — разделённые запятыми токены, следующие за ним (после удаления ведущих пробелов). Заголовок `Cookie` обрабатывается особым образом: он сохраняется как единое нераздроблённое значение, поскольку куки используют `;` как разделитель, а не `,`.

Предназначен для внутреннего использования в `asynchttpserver` и `httpclient`. Вы будете вызывать это напрямую только если реализуете низкоуровневый HTTP-парсер.

```nim
let h = parseHeader("Content-Type: text/html, application/xhtml+xml")
echo h.key           # "Content-Type"
echo h.value         # @["text/html", "application/xhtml+xml"]

let c = parseHeader("Cookie: session=abc; user=john")
echo c.key           # "Cookie"
echo c.value         # @["session=abc; user=john"]  (не разбивается)
```

---

## Конвертер

### `toString`

```nim
converter toString*(values: HttpHeaderValues): string
```

**Неявное преобразование** из `HttpHeaderValues` в `string`, извлекающее первое значение. Именно этот конвертер делает доступ к одному значению заголовка удобным — можно присваивать `headers["Content-Type"]` напрямую в переменную типа `string` без явного приведения. Однако имейте в виду: если у заголовка несколько значений, все, кроме первого, молча игнорируются через этот путь.

```nim
let h = newHttpHeaders({"Content-Length": "128"})
let len: string = h["Content-Length"]   # конвертер срабатывает автоматически
# Эквивалентная явная форма:
let len2 = seq[string](h["Content-Length"])[0]
```

---

## Полный рабочий пример

```nim
import std/httpcore

# Формируем заголовки ответа
let headers = newHttpHeaders({
  "Content-Type": "application/json",
  "Cache-Control": "no-store"
}, titleCase = true)

headers.add("Set-Cookie", "session=abc; HttpOnly; Secure")
headers.add("Set-Cookie", "lang=ru")

# Читаем заголовки
echo headers["Content-Type"]          # "application/json"
echo headers["Set-Cookie", 1]         # "lang=ru"
echo headers.len                       # 3 различных ключа

# Итерация для сериализации на провод
for key, val in headers.pairs:
  echo key & ": " & val

# Обработка кодов статуса
proc handleResponse(code: HttpCode) =
  if code.is2xx:
    echo "Успех: ", $code
  elif code.is3xx:
    echo "Перенаправление: ", $code
  elif code.is4xx:
    echo "Ошибка клиента: ", $code
  elif code.is5xx:
    echo "Ошибка сервера: ", $code

handleResponse(Http200)   # Успех: 200 OK
handleResponse(Http301)   # Перенаправление: 301 Moved Permanently
handleResponse(Http503)   # Ошибка сервера: 503 Service Unavailable

# Маршрутизация по методу
let safeMethods = {HttpGet, HttpHead, HttpOptions}
if "DELETE" in safeMethods:
  echo "не будет выведено"
if "GET" in safeMethods:
  echo "GET — безопасный метод"
```

---

*Справочник охватывает модуль `std/httpcore` стандартной библиотеки Nim согласно исходному коду модуля.*
