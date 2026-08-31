# httpcore — справочник модуля

> **Импорт:** `import std/httpcore`
> **Область применения:** общая функциональность для работы с HTTP — заголовки, коды состояния, HTTP-методы и версии протокола, — вынесенная в отдельный модуль, чтобы `httpclient` и `asynchttpserver` не дублировали одну и ту же логику.

Модуль httpcore не отправляет запросы и не открывает соединений сам по себе — он задаёт общий словарь типов и операций, которым пользуются и клиент, и сервер, поэтому знание httpcore полезно даже тем, кто напрямую импортирует только httpclient или asynchttpserver.

В модуле фактически три слабо связанные области: работа с заголовками через тип `HttpHeaders` (нечувствительный к регистру доступ по ключу, поддержка нескольких значений на один ключ), представление кода состояния через тип `HttpCode` вместе с готовым набором именованных констант и предикатами вида `is2xx`, и перечисления `HttpMethod`/`HttpVersion` для типизации метода запроса и версии протокола вместо голых строк.

Общая конвенция модуля: там, где операция должна быть нечувствительна к регистру названия заголовка (а имена HTTP-заголовков регистронезависимы по стандарту), внутри вызывается приватная процедура `toCaseInsensitive`, приводящая ключ либо к нижнему регистру, либо к Title-Case — в зависимости от флага, заданного при создании `HttpHeaders`.

---

## Оглавление

I. [Типы и константы модуля](#i-типы-и-константы-модуля)
   1. [`HttpHeaders`](#1-httpheaders)
   2. [`HttpHeaderValues`](#2-httpheadervalues)
   3. [`HttpCode`](#3-httpcode)
   4. [`HttpVersion`](#4-httpversion)
   5. [`HttpMethod`](#5-httpmethod)
   6. [`httpNewLine` и `headerLimit`](#6-httpnewline-и-headerlimit)
II. [Создание и настройка заголовков](#ii-создание-и-настройка-заголовков)
   1. [`newHttpHeaders` (без начальных значений)](#1-newhttpheaders-без-начальных-значений)
   2. [`newHttpHeaders` (из массива пар ключ-значение)](#2-newhttpheaders-из-массива-пар-ключ-значение)
   3. [`toCaseInsensitive`](#3-tocaseinsensitive)
   4. [`$` (представление HttpHeaders строкой)](#4--представление-httpheaders-строкой)
   5. [`clear`](#5-clear)
III. [Доступ к значениям и модификация заголовков](#iii-доступ-к-значениям-и-модификация-заголовков)
   1. [`[]` (все значения ключа)](#1--все-значения-ключа)
   2. [`toString` (неявное преобразование в string)](#2-tostring-неявное-преобразование-в-string)
   3. [`[]` (i-е значение ключа)](#3--i-е-значение-ключа)
   4. [`[]=` (одно значение)](#4--одно-значение)
   5. [`[]=` (список значений)](#5--список-значений)
   6. [`add`](#6-add)
   7. [`del`](#7-del)
   8. [`pairs`](#8-pairs)
   9. [`contains` (для HttpHeaderValues)](#9-contains-для-httpheadervalues)
   10. [`hasKey`](#10-haskey)
   11. [`getOrDefault`](#11-getordefault)
   12. [`len`](#12-len)
IV. [Разбор сырых данных и сравнение версий](#iv-разбор-сырых-данных-и-сравнение-версий)
   1. [`parseHeader`](#1-parseheader)
   2. [`==` (кортеж протокола и HttpVersion)](#2--кортеж-протокола-и-httpversion)
   3. [`contains` (для набора HttpMethod)](#3-contains-для-набора-httpmethod)
V. [Коды состояния HTTP](#v-коды-состояния-http)
   1. [Константы `Http100` .. `Http511`](#1-константы-http100--http511)
   2. [`$` (текстовое представление HttpCode)](#2--текстовое-представление-httpcode)
   3. [`==` (сравнение двух HttpCode)](#3--сравнение-двух-httpcode)
   4. [`is1xx`](#4-is1xx)
   5. [`is2xx`](#5-is2xx)
   6. [`is3xx`](#6-is3xx)
   7. [`is4xx`](#7-is4xx)
   8. [`is5xx`](#8-is5xx)
VI. [Практические рецепты](#vi-практические-рецепты)
   1. [Заголовки запроса с несколькими Cookie](#1-заголовки-запроса-с-несколькими-cookie)
   2. [Проверка успешности ответа по коду состояния](#2-проверка-успешности-ответа-по-коду-состояния)
   3. [Разбор сырого заголовка из сокета](#3-разбор-сырого-заголовка-из-сокета)
   4. [Слияние заголовков по умолчанию с пользовательскими](#4-слияние-заголовков-по-умолчанию-с-пользовательскими)
   5. [Проверка метода против списка разрешённых](#5-проверка-метода-против-списка-разрешённых)
   6. [Нормализация регистра заголовков перед логированием](#6-нормализация-регистра-заголовков-перед-логированием)
VII. [Краткая таблица](#vii-краткая-таблица)
VIII. [Сводка: какую процедуру выбрать](#viii-сводка-какую-процедуру-выбрать)

---

## I. Типы и константы модуля

### 1. `HttpHeaders`

```nim
type
  HttpHeaders* = ref object
    table*: TableRef[string, seq[string]]
    isTitleCase: bool
```

Что делает: хранит набор HTTP-заголовков как ссылочный объект, где под капотом лежит таблица `имя заголовка -> список значений`, потому что один и тот же заголовок (например, `Set-Cookie`) может встречаться в запросе или ответе несколько раз.

Разбор реализации: поле `table` объявлено экспортируемым (`*`) и доступно напрямую, если нужен полный обход без учёта регистронезависимости, тогда как поле `isTitleCase` скрыто — оно лишь переключает, в каком виде процедура `toCaseInsensitive` нормализует ключи при чтении и записи, и напрямую снаружи модуля не используется.

Список полей:

- `table: TableRef[string, seq[string]]` — изменяемая ссылочная таблица; ключи — уже нормализованные имена заголовков, значения — список из одной или нескольких строк.
- `isTitleCase: bool` — если true, при выводе и обращении по ключу имена заголовков приводятся к Title-Case (`Content-Length`), иначе — к нижнему регистру.

Пример:

```nim
var h = newHttpHeaders()
add(h, "X-Trace-Id", "abc123")
echo contains(h.table, "x-trace-id") # выводит true
```

---

### 2. `HttpHeaderValues`

```nim
type
  HttpHeaderValues* = distinct seq[string]
```

Что делает: оборачивает список значений одного заголовка в отдельный тип, чтобы в местах, где ожидается одиночная строка, автоматически подставлялось первое значение, а в местах, где нужен полный список, — можно было явно привести тип обратно к `seq[string]`.

Разбор реализации: `distinct seq[string]` — это тот же список строк по представлению в памяти, но как отдельный номинальный тип, поэтому напрямую операции над `seq[string]` (например `[]`) к нему не применяются без явного приведения; именно эта изоляция и позволяет безопасно повесить на тип отдельный конвертер `toString`, не рискуя случайно применить его не там.

Пример:

```nim
var h = newHttpHeaders()
add(h, "Accept", "text/html")
add(h, "Accept", "application/json")
let values = h["Accept"]
echo seq[string](values) # выводит @["text/html", "application/json"]
```

---

### 3. `HttpCode`

```nim
type
  HttpCode* = distinct range[0 .. 599]
```

Что делает: представляет числовой код состояния HTTP (200, 404, 500 и так далее), ограничивая допустимый диапазон значениями 0..599 на уровне системы типов.

Разбор реализации: диапазон специально начинается с 0, а не с наименьшего реального кода состояния (100), чтобы значение по умолчанию для непроинициализированной переменной этого типа было корректным (0), а не приводило к ошибке проверки диапазона при объявлении `var code: HttpCode` без явного значения.

Пример:

```nim
let code = HttpCode(404)
echo int(code) # выводит 404
```

---

### 4. `HttpVersion`

```nim
type
  HttpVersion* = enum
    HttpVer11,
    HttpVer10
```

Что делает: перечисляет поддерживаемые версии протокола HTTP — 1.1 и 1.0 — вместо использования отдельных чисел `major`/`minor` по всему коду.

Список значений:

- `HttpVer11` — HTTP/1.1, версия по умолчанию для современных клиентов и серверов.
- `HttpVer10` — HTTP/1.0, более старая версия без постоянных соединений по умолчанию.

Пример:

```nim
let ver = HttpVer11
echo ver == HttpVer11 # выводит true
```

---

### 5. `HttpMethod`

```nim
type
  HttpMethod* = enum
    HttpHead = "HEAD",
    HttpGet = "GET",
    HttpPost = "POST",
    HttpPut = "PUT",
    HttpDelete = "DELETE",
    HttpTrace = "TRACE",
    HttpOptions = "OPTIONS",
    HttpConnect = "CONNECT",
    HttpPatch = "PATCH"
```

Что делает: перечисляет HTTP-методы, каждый вариант связан со своим текстовым представлением через `= "..."`, поэтому значение перечисления и стандартная строка метода взаимно однозначны.

Разбор реализации: благодаря строковым значениям в объявлении, встроенный `$` для enum уже возвращает правильную строку метода (`"GET"`, `"POST"` и т.д.) без ручного написания процедуры `$`, а обратное преобразование строки в значение перечисления штатно делает `parseEnum[HttpMethod]` — на этом построена процедура `contains` для `set[HttpMethod]` в разделе IV.

Список значений: `HttpHead` (HEAD — тот же ответ, что и на GET, но без тела), `HttpGet` (GET — получение ресурса), `HttpPost` (POST — отправка данных на обработку), `HttpPut` (PUT — загрузка представления ресурса), `HttpDelete` (DELETE — удаление ресурса), `HttpTrace` (TRACE — эхо запроса, полезно для отладки промежуточных серверов), `HttpOptions` (OPTIONS — список поддерживаемых методов), `HttpConnect` (CONNECT — превращает соединение в туннель, обычно для прокси), `HttpPatch` (PATCH — частичное изменение ресурса).

Пример:

```nim
echo $HttpPost # выводит POST
```

---

### 6. `httpNewLine` и `headerLimit`

```nim
const httpNewLine* = "\c\L"
const headerLimit* = 10_000
```

Что делает: `httpNewLine` — это разделитель строк HTTP-протокола (CR LF), а `headerLimit` — верхняя граница на количество заголовков, которую могут использовать `httpclient` и `asynchttpserver`, чтобы отбросить заведомо испорченный или вредоносный запрос ещё до полного разбора.

Пример:

```nim
echo len(httpNewLine) # выводит 2
```

---

## II. Создание и настройка заголовков

### 1. `newHttpHeaders` (без начальных значений)

```nim
func newHttpHeaders*(titleCase = false): HttpHeaders
```

Что делает: создаёт пустой объект `HttpHeaders` с новой внутренней таблицей.

Список параметров:

- `titleCase: bool` — если true, заголовки при доступе по ключу и при выводе будут в Title-Case (`Content-Type`), по умолчанию false (нижний регистр).

Пример:

```nim
let h = newHttpHeaders()
echo len(h) # выводит 0
```

---

### 2. `newHttpHeaders` (из массива пар ключ-значение)

```nim
func newHttpHeaders*(keyValuePairs: openArray[tuple[key: string, val: string]],
                      titleCase = false): HttpHeaders
```

Что делает: создаёт объект `HttpHeaders` и сразу заполняет его переданными парами ключ-значение, при этом повторяющиеся ключи не перезаписывают друг друга, а накапливаются в общий список значений.

Разбор реализации: для каждой пары ключ приводится к нормализованной форме через `toCaseInsensitive`, и если такой ключ в таблице уже встречался, новое значение добавляется в существующий список через `add`, иначе создаётся новая запись с однoэлементным списком; вставка выполнена внутри блока `{.cast(noSideEffect).}`, потому что компилятор не всегда может статически доказать отсутствие побочных эффектов у операций над `TableRef` внутри `func`, хотя фактически процедура работает только с только что созданным локальным объектом и никаких внешних данных не трогает.

Список параметров:

- `keyValuePairs: openArray[tuple[key: string, val: string]]` — исходные пары для заполнения.
- `titleCase: bool` — регистр отображения ключей, по умолчанию false.

Пример:

```nim
let h = newHttpHeaders([("Accept", "text/html"), ("Accept", "application/json")])
echo seq[string](h["Accept"]) # выводит @["text/html", "application/json"]
```

---

### 3. `toCaseInsensitive`

```nim
func toCaseInsensitive*(headers: HttpHeaders, s: string): string {.inline.}
```

Что делает: приводит имя заголовка `s` к той нормализованной форме, в которой оно хранится внутри `headers` — либо к Title-Case, либо к нижнему регистру, в зависимости от `headers.isTitleCase`.

Разбор реализации: сама процедура не хранит состояние и не принимает решения о смысле заголовков, она лишь маршрутизирует вызов либо в приватную `toTitleCase`, либо во встроенную `toLowerAscii`; `toTitleCase`, в свою очередь, идёт по строке слева направо и переводит символ в верхний регистр, если предыдущим символом было начало строки или дефис — этим достигается вид `Content-Length` из `content-length`.

Согласно комментарию в исходном коде, эта процедура предназначена только для внутреннего использования модулями httpclient и asynchttpserver и не рекомендована к прямому вызову из пользовательского кода.

Список параметров:

- `headers: HttpHeaders` — источник настройки регистра (используется только флаг `isTitleCase`, не содержимое).
- `s: string` — исходное имя заголовка.

Пример:

```nim
let h = newHttpHeaders(titleCase = true)
echo toCaseInsensitive(h, "content-length") # выводит Content-Length
```

---

### 4. `$` (представление HttpHeaders строкой)

```nim
func `$`*(headers: HttpHeaders): string {.inline.}
```

Что делает: возвращает строковое представление всех заголовков целиком, делегируя работу стандартному `$` для `TableRef[string, seq[string]]`.

Пример:

```nim
let h = newHttpHeaders([("Accept", "text/html")])
echo $h # выводит {accept: @["text/html"]}
```

---

### 5. `clear`

```nim
proc clear*(headers: HttpHeaders) {.inline.}
```

Что делает: удаляет из объекта все заголовки, оставляя пустую таблицу, но не создавая новый объект `HttpHeaders` — все существующие ссылки на этот объект увидят опустевшую таблицу.

Пример:

```nim
var h = newHttpHeaders([("Accept", "text/html")])
clear(h)
echo len(h) # выводит 0
```

---

## III. Доступ к значениям и модификация заголовков

### 1. `[]` (все значения ключа)

```nim
func `[]`*(headers: HttpHeaders, key: string): HttpHeaderValues
```

Что делает: возвращает все значения, связанные с ключом `key`, в виде `HttpHeaderValues`; если такого ключа нет вообще, поднимается исключение, а не возвращается пустой результат.

Список параметров:

- `headers: HttpHeaders` — источник данных.
- `key: string` — имя заголовка, регистр не важен.

Пример:

```nim
let h = newHttpHeaders([("Accept", "text/html")])
echo toString(h["Accept"]) # выводит text/html
doAssertRaises(KeyError):
  discard h["X-Not-Set"]
```

---

### 2. `toString` (неявное преобразование в string)

```nim
converter toString*(values: HttpHeaderValues): string
```

Что делает: неявно превращает `HttpHeaderValues` в обычную строку, беря из списка первый элемент — благодаря этому `HttpHeaderValues`, возвращённый оператором `[]`, можно передавать в любое место, ожидающее `string`, без ручного вызова процедуры.

Разбор реализации: как конвертер, эта процедура вызывается компилятором автоматически на границе типов, поэтому в примерах модуля значение `headers["Accept"]` часто выглядит как обычная строка, хотя формально имеет тип `HttpHeaderValues`; если список значений пуст, обращение к нулевому элементу поднимет исключение выхода за границы — на практике этого не происходит, потому что оператор `[]=` не допускает сохранения пустого списка (см. пункт 5 ниже).

Пример:

```nim
let h = newHttpHeaders([("Accept", "text/html")])
let s: string = h["Accept"]
echo s # выводит text/html
```

---

### 3. `[]` (i-е значение ключа)

```nim
func `[]`*(headers: HttpHeaders, key: string, i: int): string
```

Что делает: возвращает конкретное значение с индексом `i` из списка, связанного с ключом; если ключа нет или индекс выходит за пределы списка, поднимается исключение.

Список параметров:

- `headers: HttpHeaders` — источник данных.
- `key: string` — имя заголовка.
- `i: int` — индекс значения, начиная с 0.

Пример:

```nim
let h = newHttpHeaders([("Accept", "text/html"), ("Accept", "application/json")])
echo h["Accept", 1] # выводит application/json
```

---

### 4. `[]=` (одно значение)

```nim
proc `[]=`*(headers: HttpHeaders, key, value: string)
```

Что делает: заменяет все существующие значения ключа одним новым значением `value`, стирая предыдущий список полностью, даже если в нём было несколько элементов.

Список параметров:

- `headers: HttpHeaders` — изменяемый объект.
- `key: string` — имя заголовка.
- `value: string` — новое единственное значение.

Пример:

```nim
var h = newHttpHeaders([("Accept", "text/html"), ("Accept", "application/json")])
h["Accept"] = "text/plain"
echo seq[string](h["Accept"]) # выводит @["text/plain"]
```

---

### 5. `[]=` (список значений)

```nim
proc `[]=`*(headers: HttpHeaders, key: string, value: seq[string])
```

Что делает: заменяет значения ключа целым списком за один вызов; граничный случай — если передать пустой список, ключ вообще удаляется из заголовков, а не остаётся с пустым списком значений.

Список параметров:

- `headers: HttpHeaders` — изменяемый объект.
- `key: string` — имя заголовка.
- `value: seq[string]` — новый список значений; пустой список означает удаление ключа.

Пример:

```nim
var h = newHttpHeaders([("Accept", "text/html")])
h["Accept"] = @["text/plain", "text/csv"]
echo seq[string](h["Accept"]) # выводит @["text/plain", "text/csv"]
h["Accept"] = newSeq[string]()
echo hasKey(h, "Accept") # выводит false
```

---

### 6. `add`

```nim
proc add*(headers: HttpHeaders, key, value: string)
```

Что делает: добавляет значение к существующему списку значений ключа, не удаляя то, что там уже было; если ключа ещё нет, он создаётся с одним значением.

Список параметров:

- `headers: HttpHeaders` — изменяемый объект.
- `key: string` — имя заголовка.
- `value: string` — добавляемое значение.

Пример:

```nim
var h = newHttpHeaders()
add(h, "Set-Cookie", "a=1")
add(h, "Set-Cookie", "b=2")
echo seq[string](h["Set-Cookie"]) # выводит @["a=1", "b=2"]
```

---

### 7. `del`

```nim
proc del*(headers: HttpHeaders, key: string)
```

Что делает: полностью удаляет ключ и все его значения из заголовков; если ключа не было, вызов не вызывает ошибки.

Список параметров:

- `headers: HttpHeaders` — изменяемый объект.
- `key: string` — удаляемое имя заголовка.

Пример:

```nim
var h = newHttpHeaders([("Accept", "text/html")])
del(h, "Accept")
echo hasKey(h, "Accept") # выводит false
```

---

### 8. `pairs`

```nim
iterator pairs*(headers: HttpHeaders): tuple[key, value: string]
```

Что делает: последовательно выдаёт пары `(key, value)` для всех заголовков; граничный случай — если у ключа несколько значений, для этого ключа будет выдано несколько отдельных пар подряд, а не одна пара со списком.

Пример:

```nim
let h = newHttpHeaders([("Accept", "text/html"), ("Accept", "application/json")])
for key, value in pairs(h):
  echo key, " -> ", value
# выводит:
# accept -> text/html
# accept -> application/json
```

---

### 9. `contains` (для HttpHeaderValues)

```nim
func contains*(values: HttpHeaderValues, value: string): bool
```

Что делает: проверяет, есть ли строка `value` среди значений `values`, сравнивая без учёта регистра.

Разбор реализации: процедура явно приводит и сохранённые значения, и искомую строку к нижнему регистру перед сравнением, поэтому `"text/HTML"` и `"text/html"` считаются совпадающими; сложность — O(n) от количества значений, так как выполняется линейный перебор без предварительной сортировки.

Список параметров:

- `values: HttpHeaderValues` — список значений одного заголовка.
- `value: string` — искомая строка.

Пример:

```nim
let h = newHttpHeaders([("Accept", "text/HTML")])
echo contains(h["Accept"], "text/html") # выводит true
```

---

### 10. `hasKey`

```nim
func hasKey*(headers: HttpHeaders, key: string): bool
```

Что делает: проверяет наличие ключа среди заголовков без риска поднять исключение, в отличие от оператора `[]`.

Список параметров:

- `headers: HttpHeaders` — источник данных.
- `key: string` — имя заголовка.

Пример:

```nim
let h = newHttpHeaders([("Accept", "text/html")])
echo hasKey(h, "accept") # выводит true
```

---

### 11. `getOrDefault`

```nim
func getOrDefault*(headers: HttpHeaders, key: string,
                    default = HttpHeaderValues(@[""])): HttpHeaderValues
```

Что делает: безопасный вариант чтения — возвращает значения ключа, если он есть, иначе возвращает `default`, не поднимая исключений.

Разбор реализации: значение по умолчанию для параметра `default` в исходном коде записано через постфиксный синтаксис `@[""].HttpHeaderValues`; при переносе в справочник эта запись переписана в требуемой префиксной форме `HttpHeaderValues(@[""])` — это то же самое приведение списка из одной пустой строки к типу `HttpHeaderValues`.

Список параметров:

- `headers: HttpHeaders` — источник данных.
- `key: string` — имя заголовка.
- `default: HttpHeaderValues` — значение на случай отсутствия ключа, по умолчанию список из одной пустой строки.

Пример:

```nim
let h = newHttpHeaders()
echo toString(getOrDefault(h, "X-Trace-Id")) # выводит (пустая строка)
```

---

### 12. `len`

```nim
func len*(headers: HttpHeaders): int {.inline.}
```

Что делает: возвращает количество различных ключей заголовков; граничный случай, о котором легко забыть — это число ключей, а не суммарное число значений по всем ключам.

Пример:

```nim
let h = newHttpHeaders([("Accept", "text/html"), ("Accept", "application/json")])
echo len(h) # выводит 1
```

---

## IV. Разбор сырых данных и сравнение версий

### 1. `parseHeader`

```nim
func parseHeader*(line: string): tuple[key: string, value: seq[string]]
```

Что делает: разбирает одну сырую строку HTTP-заголовка вида `"Key: value1, value2"` на имя заголовка и список значений.

Разбор реализации: сначала процедура вызывает `parseUntil` до символа `:`, чтобы выделить ключ; если после двоеточия ничего нет, но ключ непустой, значением становится список из одной пустой строки, а если и ключа нет — пустой список значений (пустая строка целиком). Если ключ, приведённый к нижнему регистру, равен `"cookie"`, то оставшаяся часть строки берётся целиком как единственное значение — это отдельная ветка, потому что значения cookie сами могут содержать запятые, и обычное разбиение по запятой их бы испортило. Во всех остальных случаях остаток строки передаётся приватной вспомогательной процедуре `parseList`, которая идёт по строке, пропуская пробелы (`skipWhitespace`) и вычленяя фрагменты до ближайшей запятой или конца строки (`parseUntil`), добавляя каждый найденный фрагмент в список и пропуская сам разделитель-запятую — по сути, `parseList` реализует упрощённый CSV-разбор в пределах одной строки заголовка.

Список параметров:

- `line: string` — сырая строка заголовка без завершающего перевода строки.

Примечание из исходного кода: процедура используется `asynchttpserver` и `httpclient` внутри и не предназначена для прямого вызова пользователем.

Пример:

```nim
let parsed = parseHeader("Accept: text/html, application/json")
echo parsed.key # выводит Accept
echo parsed.value # выводит @["text/html", "application/json"]

let cookieLine = parseHeader("Cookie: a=1, b=2")
echo cookieLine.value # выводит @["a=1, b=2"]

let emptyValue = parseHeader("X-Empty:")
echo emptyValue.value # выводит @[""]
```

---

### 2. `==` (кортеж протокола и HttpVersion)

```nim
func `==`*(protocol: tuple[orig: string, major, minor: int], ver: HttpVersion): bool
```

Что делает: сравнивает разобранную тройку `(orig, major, minor)` (например, результат парсинга строки `"HTTP/1.1"`) со значением перечисления `HttpVersion`, чтобы не писать сравнение чисел вручную в клиентском коде.

Разбор реализации: перечисление сначала переводится в пару чисел (`major`, `minor`) через `case`, а затем оба числа сравниваются с полями кортежа; поле `orig` кортежа в сравнении не участвует вовсе, то есть исходная текстовая запись версии игнорируется, важны только числовые `major`/`minor`.

Список параметров:

- `protocol: tuple[orig: string, major, minor: int]` — разобранные компоненты версии протокола.
- `ver: HttpVersion` — версия, с которой производится сравнение.

Пример:

```nim
let protocol = (orig: "HTTP/1.1", major: 1, minor: 1)
echo protocol == HttpVer11 # выводит true
echo protocol == HttpVer10 # выводит false
```

---

### 3. `contains` (для набора HttpMethod)

```nim
func contains*(methods: set[HttpMethod], x: string): bool
```

Что делает: проверяет, входит ли метод, заданный строкой `x` (например, `"GET"`), в набор разрешённых методов `methods`.

Разбор реализации: строка сначала преобразуется в значение перечисления через `parseEnum[HttpMethod]`, а затем стандартным образом проверяется принадлежность множеству; если строка не совпадает ни с одним из известных методов, `parseEnum` поднимает исключение `ValueError`, а не возвращает false — это важный граничный случай, отличающий поведение от простого текстового поиска.

Список параметров:

- `methods: set[HttpMethod]` — набор разрешённых методов.
- `x: string` — проверяемый метод в виде строки (регистр важен, должен совпадать со значением перечисления, например `"GET"`, а не `"get"`).

Пример:

```nim
let allowed = {HttpGet, HttpPost}
echo contains(allowed, "GET") # выводит true
echo contains(allowed, "DELETE") # выводит false
doAssertRaises(ValueError):
  discard contains(allowed, "not-a-method")
```

---

## V. Коды состояния HTTP

### 1. Константы `Http100` .. `Http511`

```nim
const
  Http100* = HttpCode(100)
  Http200* = HttpCode(200)
  Http404* = HttpCode(404)
  Http500* = HttpCode(500)
```

Что делает: модуль объявляет именованную константу `HttpCode` практически для каждого зарегистрированного кода состояния HTTP, чтобы в коде можно было писать `Http404` вместо магического числа `404`.

Список групп по стандартным классам ответа:

| Класс | Диапазон | Примеры констант |
| --- | --- | --- |
| Информационные | 100–103 | `Http100`, `Http101`, `Http102`, `Http103` |
| Успешные | 200–226 | `Http200`, `Http201`, `Http204`, `Http226` |
| Перенаправления | 300–308 | `Http301`, `Http302`, `Http304`, `Http307` |
| Ошибки клиента | 400–451 | `Http400`, `Http404`, `Http418`, `Http429` |
| Ошибки сервера | 500–511 | `Http500`, `Http502`, `Http503`, `Http511` |

Отдельные менее распространённые коды в исходном модуле снабжены комментариями со ссылками на конкретный RFC — например, `Http102` и ряд кодов 4xx/5xx относятся к расширению WebDAV (RFC 4918/2518), `Http103` — к Early Hints (RFC 8297), а `Http425` — к Early Data (RFC 8470); это стоит иметь в виду, встречая в логах редкие коды за пределами привычного набора 200/301/404/500.

Пример:

```nim
echo int(Http404) # выводит 404
echo Http404 == HttpCode(404) # выводит true
```

---

### 2. `$` (текстовое представление HttpCode)

```nim
func `$`*(code: HttpCode): string
```

Что делает: превращает числовой код в привычную строку вида `"404 Not Found"`, сразу с пояснением статуса, а не только с числом.

Разбор реализации: внутри — большой `case` по `code.int`, перечисляющий все стандартные коды из константного блока выше; если код не входит ни в один из перечисленных вариантов (например, редкий или ещё не зарегистрированный код), ветка `else` возвращает просто число без пояснения — то есть для нестандартных кодов функция не падает с ошибкой, а мягко деградирует к числовому представлению.

Список параметров:

- `code: HttpCode` — код состояния для форматирования.

Пример:

```nim
echo $Http404 # выводит 404 Not Found
echo $HttpCode(499) # выводит 499
```

---

### 3. `==` (сравнение двух HttpCode)

```nim
func `==`*(a, b: HttpCode): bool {.borrow.}
```

Что делает: сравнивает два кода состояния на равенство.

Разбор реализации: реализация не пишется вручную, а «занимается» (`borrow`) у операции `==`, уже существующей для базового типа диапазона `range[0..599]`, то есть фактически сравниваются обычные целые числа без какой-либо дополнительной обёртки — это самый дешёвый по времени выполнения способ добавить оператор для `distinct`-типа.

Список параметров:

- `a, b: HttpCode` — сравниваемые коды.

Пример:

```nim
echo Http200 == HttpCode(200) # выводит true
echo Http200 == Http404 # выводит false
```

---

### 4. `is1xx`

```nim
func is1xx*(code: HttpCode): bool {.inline, since: (1, 5).}
```

Что делает: определяет, относится ли код к классу информационных ответов (100–199).

Список параметров:

- `code: HttpCode` — проверяемый код.

Пример:

```nim
echo is1xx(HttpCode(103)) # выводит true
echo is1xx(Http200) # выводит false
```

---

### 5. `is2xx`

```nim
func is2xx*(code: HttpCode): bool {.inline.}
```

Что делает: определяет, относится ли код к классу успешных ответов (200–299).

Пример:

```nim
echo is2xx(Http200) # выводит true
```

---

### 6. `is3xx`

```nim
func is3xx*(code: HttpCode): bool {.inline.}
```

Что делает: определяет, относится ли код к классу перенаправлений (300–399).

Пример:

```nim
echo is3xx(Http301) # выводит true
```

---

### 7. `is4xx`

```nim
func is4xx*(code: HttpCode): bool {.inline.}
```

Что делает: определяет, относится ли код к классу ошибок клиента (400–499).

Пример:

```nim
echo is4xx(Http404) # выводит true
```

---

### 8. `is5xx`

```nim
func is5xx*(code: HttpCode): bool {.inline.}
```

Что делает: определяет, относится ли код к классу ошибок сервера (500–599).

Пример:

```nim
echo is5xx(Http503) # выводит true
```

---

## VI. Практические рецепты

### 1. Заголовки запроса с несколькими Cookie

```nim
var headers = newHttpHeaders()
add(headers, "Cookie", "session=abc")
add(headers, "Cookie", "theme=dark")
for key, value in pairs(headers):
  echo key, ": ", value
# выводит:
# cookie: session=abc
# cookie: theme=dark
```

Здесь `add` использован дважды подряд для одного и того же ключа именно потому, что `[]=` перезаписал бы предыдущее значение целиком, а `add` аккуратно накапливает список.

---

### 2. Проверка успешности ответа по коду состояния

```nim
proc describeResponse(code: HttpCode): string =
  if is2xx(code):
    result = "успех: " & $code
  elif is4xx(code):
    result = "ошибка клиента: " & $code
  elif is5xx(code):
    result = "ошибка сервера: " & $code
  else:
    result = "прочее: " & $code

echo describeResponse(Http200) # выводит успех: 200 OK
echo describeResponse(Http404) # выводит ошибка клиента: 404 Not Found
echo describeResponse(Http503) # выводит ошибка сервера: 503 Service Unavailable
```

Комбинация предикатов `is2xx`/`is4xx`/`is5xx` с `$` даёт готовую классификацию ответа без ручного сравнения диапазонов чисел.

---

### 3. Разбор сырого заголовка из сокета

```nim
let rawLines = @["Content-Type: application/json", "Accept: text/html, text/plain"]
var headers = newHttpHeaders()
for line in rawLines:
  let parsed = parseHeader(line)
  headers[parsed.key] = parsed.value

echo toString(headers["Content-Type"]) # выводит application/json
echo seq[string](headers["Accept"]) # выводит @["text/html", "text/plain"]
```

`parseHeader` возвращает готовую пару ключ/список значений, которую можно напрямую передать в `[]=`, не разбирая строку заголовка вручную второй раз.

---

### 4. Слияние заголовков по умолчанию с пользовательскими

```nim
proc mergeHeaders(defaults, overrides: HttpHeaders): HttpHeaders =
  result = newHttpHeaders()
  for key, value in pairs(defaults):
    add(result, key, value)
  for key, value in pairs(overrides):
    if hasKey(result, key):
      result[key] = value
    else:
      add(result, key, value)

let defaults = newHttpHeaders([("Accept", "*/*"), ("User-Agent", "nim-client")])
let overrides = newHttpHeaders([("Accept", "application/json")])
let merged = mergeHeaders(defaults, overrides)
echo toString(merged["Accept"]) # выводит application/json
echo toString(merged["User-Agent"]) # выводит nim-client
```

Проверка `hasKey` перед перезаписью нужна, чтобы пользовательский заголовок полностью заменял значение по умолчанию (через `[]=`), а не просто добавлялся к нему (через `add`).

---

### 5. Проверка метода против списка разрешённых

```nim
proc isMethodAllowed(allowed: set[HttpMethod], methodName: string): bool =
  try:
    result = contains(allowed, methodName)
  except ValueError:
    result = false

let allowed = {HttpGet, HttpHead, HttpOptions}
echo isMethodAllowed(allowed, "GET") # выводит true
echo isMethodAllowed(allowed, "DELETE") # выводит false
echo isMethodAllowed(allowed, "totally-not-a-method") # выводит false
```

Обёртка в `try`/`except` нужна из-за граничного поведения `contains`: для произвольной строки, не совпадающей ни с одним значением `HttpMethod`, вызов поднимает `ValueError`, а не просто возвращает false.

---

### 6. Нормализация регистра заголовков перед логированием

```nim
proc toTitleCaseHeaders(source: HttpHeaders): HttpHeaders =
  result = newHttpHeaders(titleCase = true)
  for key, value in pairs(source):
    add(result, key, value)

let raw = newHttpHeaders([("content-type", "text/html"), ("x-request-id", "42")])
let normalized = toTitleCaseHeaders(raw)
echo $normalized # выводит {Content-Type: @["text/html"], X-Request-Id: @["42"]}
```

Поскольку регистр задаётся один раз при создании объекта через `titleCase`, а не для каждого ключа отдельно, для смены регистра существующих заголовков проще собрать новый объект `HttpHeaders`, чем пытаться отредактировать имеющийся.

---

## VII. Краткая таблица

| Задача | Что использовать |
| --- | --- |
| Создать пустой набор заголовков | `newHttpHeaders()` |
| Создать заголовки сразу из пар | `newHttpHeaders(keyValuePairs)` |
| Прочитать первое значение как строку | `headers[key]` (с неявным `toString`) |
| Прочитать все значения ключа | `seq[string](headers[key])` |
| Прочитать конкретное значение по индексу | `headers[key, i]` |
| Заменить значения ключа целиком | `headers[key] = value` |
| Добавить значение, не теряя старые | `add(headers, key, value)` |
| Удалить ключ | `del(headers, key)` или `headers[key] = @[]` |
| Проверить наличие ключа без исключений | `hasKey(headers, key)` |
| Прочитать значение с запасным вариантом | `getOrDefault(headers, key, default)` |
| Перебрать все пары ключ-значение | `for k, v in pairs(headers)` |
| Узнать число различных ключей | `len(headers)` |
| Разобрать сырую строку заголовка | `parseHeader(line)` |
| Сравнить разобранную версию протокола | `protocol == HttpVer11` |
| Проверить метод против набора разрешённых | `contains(methods, methodString)` |
| Классифицировать код состояния | `is1xx`/`is2xx`/`is3xx`/`is4xx`/`is5xx` |
| Получить текстовое пояснение кода | `$code` |

---

## VIII. Сводка: какую процедуру выбрать

- Нужно создать заголовки → используйте `newHttpHeaders()` или `newHttpHeaders(keyValuePairs)`.
- Нужно один раз задать значение заголовка, отбросив старое → используйте `[]=`.
- Нужно накопить несколько значений на один ключ (например, несколько Cookie) → используйте `add`.
- Нужно прочитать значение, не рискуя поймать исключение при отсутствии ключа → используйте `getOrDefault` или предварительно проверьте `hasKey`.
- Нужно получить все значения ключа списком, а не только первое → приведите результат `[]` через `seq[string](...)`.
- Нужно пройтись по всем заголовкам, включая повторяющиеся ключи, → используйте итератор `pairs`.
- Нужно превратить сырую строку из сокета в пару ключ/значения → используйте `parseHeader`.
- Нужно сравнить разобранную версию протокола с известной константой → используйте оператор `==` для кортежа протокола и `HttpVersion`.
- Нужно проверить, разрешён ли метод запроса → используйте `contains` для `set[HttpMethod]`, обернув вызов в `try`/`except ValueError`, если строка метода не гарантированно валидна.
- Нужно понять, к какому классу относится код ответа → используйте `is1xx`/`is2xx`/`is3xx`/`is4xx`/`is5xx`.
- Нужно получить человекочитаемое пояснение кода для логов → используйте `$code`.
- Нужно сравнить два кода состояния → используйте `==` для `HttpCode`.
