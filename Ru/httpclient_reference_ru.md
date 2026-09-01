# httpclient — справочник модуля

> **Импорт:** `import std/httpclient`
> **Область применения:** синхронный и асинхронный HTTP/HTTPS-клиент для получения и отправки данных (GET, HEAD, POST, PUT, PATCH, DELETE), включая multipart-формы, прокси, редиректы и отслеживание прогресса загрузки.

Модуль реализует два вида клиента с одинаковым набором процедур: `HttpClient` для синхронной работы и `AsyncHttpClient` для асинхронной, основанной на `async`/`await`; выбор между ними не меняет сигнатуры вызовов — большинство процедур объявлены как `{.multisync.}` и просто оборачивают результат в `Future`, когда используется асинхронный клиент.

Действует общая конвенция парных процедур: `get` возвращает объект `Response`/`AsyncResponse` целиком, а `getContent` сразу читает и возвращает тело ответа строкой, поднимая `HttpRequestError`, если сервер вернул код ошибки 4xx или 5xx; то же соответствие верно для пар `post`/`postContent`, `put`/`putContent`, `patch`/`patchContent`, `delete`/`deleteContent`.

Модуль не проверяет URI на вредоносность — валидация недоверенных адресов остаётся на совести вызывающего кода.

---

## Оглавление

I. [Типы данных](#типы-данных)
&nbsp;&nbsp;&nbsp;1. [`Response` и `AsyncResponse`](#response-и-asyncresponse)
&nbsp;&nbsp;&nbsp;2. [`HttpClient` и `AsyncHttpClient`](#httpclient-и-asynchttpclient)
&nbsp;&nbsp;&nbsp;3. [`Proxy`](#proxy)
&nbsp;&nbsp;&nbsp;4. [`MultipartData`](#multipartdata)
&nbsp;&nbsp;&nbsp;5. [Исключения `ProtocolError` и `HttpRequestError`](#исключения-protocolerror-и-httprequesterror)

II. [Создание клиента и управление соединением](#создание-клиента-и-управление-соединением)
&nbsp;&nbsp;&nbsp;1. [`newHttpClient`](#newhttpclient)
&nbsp;&nbsp;&nbsp;2. [`newAsyncHttpClient`](#newasynchttpclient)
&nbsp;&nbsp;&nbsp;3. [`close`](#close)
&nbsp;&nbsp;&nbsp;4. [`getSocket`](#getsocket)

III. [Прокси](#прокси)
&nbsp;&nbsp;&nbsp;1. [`newProxy`](#newproxy)

IV. [Multipart-данные для форм и файлов](#multipart-данные-для-форм-и-файлов)
&nbsp;&nbsp;&nbsp;1. [`newMultipartData`](#newmultipartdata)
&nbsp;&nbsp;&nbsp;2. [`add` (запись имя/значение)](#add-запись-имязначение)
&nbsp;&nbsp;&nbsp;3. [`add` (список записей)](#add-список-записей)
&nbsp;&nbsp;&nbsp;4. [`addFiles`](#addfiles)
&nbsp;&nbsp;&nbsp;5. [`[]=` (запись значения или файла)](#-запись-значения-или-файла)
&nbsp;&nbsp;&nbsp;6. [`$` (строковое представление)](#-строковое-представление)

V. [Выполнение запросов](#выполнение-запросов)
&nbsp;&nbsp;&nbsp;1. [`request`](#request)
&nbsp;&nbsp;&nbsp;2. [`get` и `getContent`](#get-и-getcontent)
&nbsp;&nbsp;&nbsp;3. [`head`](#head)
&nbsp;&nbsp;&nbsp;4. [`post` и `postContent`](#post-и-postcontent)
&nbsp;&nbsp;&nbsp;5. [`put` и `putContent`](#put-и-putcontent)
&nbsp;&nbsp;&nbsp;6. [`patch` и `patchContent`](#patch-и-patchcontent)
&nbsp;&nbsp;&nbsp;7. [`delete` и `deleteContent`](#delete-и-deletecontent)
&nbsp;&nbsp;&nbsp;8. [`downloadFile`](#downloadfile)

VI. [Чтение данных ответа](#чтение-данных-ответа)
&nbsp;&nbsp;&nbsp;1. [`code`](#code)
&nbsp;&nbsp;&nbsp;2. [`contentType`](#contenttype)
&nbsp;&nbsp;&nbsp;3. [`contentLength`](#contentlength)
&nbsp;&nbsp;&nbsp;4. [`lastModified`](#lastmodified)
&nbsp;&nbsp;&nbsp;5. [`body`](#body)

VII. [Практические рецепты](#практические-рецепты)

VIII. [Краткая таблица](#краткая-таблица)

IX. [Сводка: какую процедуру выбрать](#сводка-какую-процедуру-выбрать)

---

## Типы данных

### `Response` и `AsyncResponse`

```nim
Response* = ref object
  version*: string
  status*: string
  headers*: HttpHeaders
  body: string
  bodyStream*: Stream

AsyncResponse* = ref object
  version*: string
  status*: string
  headers*: HttpHeaders
  body: string
  bodyStream*: FutureStream[string]
```

Что делает: это объекты, которые возвращают `request`, `get`, `post` и родственные процедуры; они хранят версию протокола, строку статуса, заголовки ответа и поток тела ответа.

Разбор реализации: поле `body` приватное и не читается напрямую — доступ к телу идёт только через процедуру `body`, которая при первом обращении вычитывает `bodyStream` целиком и кеширует результат; повторный вызов `body` вернёт уже сохранённую строку, не трогая поток заново.

Список параметров:

- `version` — строка версии протокола, например `"1.1"`.
- `status` — полная строка статуса, например `"200 OK"`.
- `headers` — заголовки ответа в виде `HttpHeaders`.
- `bodyStream` — поток, из которого читается тело; у синхронного клиента это `Stream`, у асинхронного — `FutureStream[string]`.

Пример:

```nim
import std/httpclient

let client = newHttpClient()
let resp = get(client, "http://example.com")
echo resp.status # выводит "200 OK"
close(client)
```

---

### `HttpClient` и `AsyncHttpClient`

```nim
HttpClientBase*[SocketType] = ref object
  headers*: HttpHeaders
  timeout*: int
  onProgressChanged*: ProgressChangedProc[...]

HttpClient* = HttpClientBase[Socket]
AsyncHttpClient* = HttpClientBase[AsyncSocket]
```

Что делает: `HttpClientBase` — обобщённый тип клиента, параметризованный видом сокета; `HttpClient` — его синхронная реализация поверх `Socket`, `AsyncHttpClient` — асинхронная поверх `AsyncSocket`.

Разбор реализации: тип объявлен как generic именно для того, чтобы синхронная и асинхронная версии переиспользовали одни и те же процедуры через ограничение `client: HttpClient | AsyncHttpClient` и прагму `{.multisync.}`, которая при компиляции генерирует обе версии тела процедуры из одного исходного кода; часть внутренних полей (например, `bodyStream`) объявлена через `when SocketType is AsyncSocket` — это ветвление типа поля в зависимости от того, какой сокет использует конкретный экземпляр.

Список параметров (публичные поля):

- `headers` — заголовки, которые будут отправляться со всеми запросами этого клиента; можно менять после создания.
- `timeout` — таймаут в миллисекундах; учитывается только синхронным `HttpClient`.
- `onProgressChanged` — колбэк, вызываемый примерно раз в секунду с информацией о ходе загрузки; чтобы отключить, присваивается `nil`.

Пример:

```nim
import std/httpclient

let client = newHttpClient()
client.headers = newHttpHeaders({"Accept": "application/json"})
client.timeout = 5000
echo client.timeout # выводит 5000
close(client)
```

---

### `Proxy`

```nim
Proxy* = ref object
  url*: Uri
```

Что делает: описывает адрес и (через `url.username`/`url.password`) учётные данные прокси-сервера, используемого клиентом для всех запросов.

Список параметров:

- `url` — адрес прокси в виде `Uri`; логин и пароль передаются как часть этого же URI (`http://user:pass@host`).

Пример: см. раздел [`newProxy`](#newproxy).

---

### `MultipartData`

```nim
MultipartEntries* = openArray[tuple[name, content: string]]
MultipartData* = ref object
```

Что делает: контейнер для полей и файлов multipart-формы, который передаётся в `request`, `post`, `postContent` и другие процедуры через параметр `multipart`.

Список параметров:

- `MultipartEntries` — вспомогательный тип-синоним для массива пар "имя, значение", используемый как компактная форма заполнения без файлов.

---

### Исключения `ProtocolError` и `HttpRequestError`

```nim
ProtocolError* = object of IOError
HttpRequestError* = object of IOError
```

Что делает: `ProtocolError` поднимается, когда сервер отвечает не по протоколу HTTP (например, некорректная строка статуса или отсутствует ожидаемый заголовок); `HttpRequestError` поднимается процедурами `getContent`, `postContent` и аналогичными, когда сервер вернул код ответа 4xx или 5xx.

Пример:

```nim
import std/httpclient

let client = newHttpClient()
doAssertRaises(HttpRequestError):
  discard getContent(client, "http://example.com/no-such-page")
close(client)
```

---

## Создание клиента и управление соединением

### `newHttpClient`

```nim
proc newHttpClient*(userAgent = defUserAgent, maxRedirects = 5,
                    sslContext = getDefaultSSL(), proxy: Proxy = nil,
                    timeout = -1, headers = newHttpHeaders()): HttpClient
```

Что делает: создаёт новый синхронный `HttpClient`; отдельный экземпляр держит одно TCP-соединение и не рассчитан на параллельные запросы из нескольких потоков.

Разбор реализации: `sslContext` по умолчанию берётся из `getDefaultSSL()` — это ленивый одиночный (per-thread) контекст, который создаётся один раз через `{.threadvar.}` и переиспользуется всеми клиентами потока, если явно не передан свой; `timeout = -1` означает отсутствие ограничения по времени.

Список параметров:

- `userAgent` — значение заголовка `User-Agent`.
- `maxRedirects` — сколько редиректов подряд разрешено пройти; `0` отключает следование редиректам.
- `sslContext` — контекст SSL/TLS для HTTPS-запросов.
- `proxy` — прокси-сервер (см. [`newProxy`](#newproxy)); `nil`, если прокси не используется.
- `timeout` — таймаут ожидания данных на сокете, в миллисекундах; `-1` — без таймаута.
- `headers` — заголовки, отправляемые со всеми запросами этого клиента.

Пример:

```nim
import std/httpclient

let client = newHttpClient(userAgent = "MyApp/1.0", maxRedirects = 0)
echo client.timeout # выводит -1
close(client)
```

---

### `newAsyncHttpClient`

```nim
proc newAsyncHttpClient*(userAgent = defUserAgent, maxRedirects = 5,
                         sslContext = getDefaultSSL(), proxy: Proxy = nil,
                         headers = newHttpHeaders()): AsyncHttpClient
```

Что делает: асинхронный аналог `newHttpClient`; параметра `timeout` у него нет — таймаут поддерживается только синхронным клиентом.

Список параметров: те же, что и у `newHttpClient`, за вычетом `timeout`.

Пример:

```nim
import std/[asyncdispatch, httpclient]

proc fetchTitle(): Future[string] {.async.} =
  let client = newAsyncHttpClient()
  result = await getContent(client, "http://example.com")
  close(client)

echo waitFor fetchTitle() # выводит HTML-содержимое страницы
```

---

### `close`

```nim
proc close*(client: HttpClient | AsyncHttpClient)
```

Что делает: закрывает сокет, если соединение было установлено; повторный вызов безопасен.

Разбор реализации: закрытие проверяется через внутренний флаг `connected`, поэтому вызов `close` для клиента, который ещё ни разу не подключался, не приводит к ошибке.

Список параметров:

- `client` — клиент, соединение которого нужно закрыть.

Пример:

```nim
import std/httpclient

let client = newHttpClient()
close(client)
close(client) # выводит ничего — повторный close безопасен
```

---

### `getSocket`

```nim
proc getSocket*(client: HttpClient): Socket
proc getSocket*(client: AsyncHttpClient): AsyncSocket
```

Что делает: возвращает низкоуровневый сокет клиента — полезно для диагностики соединения (локальный и удалённый адрес и т.п.).

Список параметров:

- `client` — клиент, чей сокет нужно получить.

Пример:

```nim
import std/httpclient

let client = newHttpClient()
discard get(client, "http://example.com")
if client.connected:
  echo getPeerAddr(getSocket(client)) # выводит адрес удалённого сервера
close(client)
```

---

## Прокси

### `newProxy`

```nim
proc newProxy*(url: Uri): Proxy
proc newProxy*(url: string): Proxy
```

Что делает: строит объект `Proxy` из адреса; учётные данные basic-авторизации передаются прямо в URL (`http://user:password@host`), а не отдельным параметром — старые перегрузки с отдельным параметром `auth` помечены как устаревшие.

Разбор реализации: поддерживаются схемы `http`/`https` (обычный прокси с методом `CONNECT` для HTTPS) и `socks5h` (SOCKS5 с DNS-резолвингом на стороне прокси) — конкретная схема читается из `url.scheme` в момент установления соединения, сам `newProxy` лишь сохраняет переданный URI.

Список параметров:

- `url` — адрес прокси как `Uri` или строка.

Примеры:

```nim
import std/httpclient

let plainProxy = newProxy("http://myproxy.network")
let client = newHttpClient(proxy = plainProxy)
close(client)
```

```nim
import std/httpclient

let authProxy = newProxy("http://user:password@myproxy.network")
let socksProxy = newProxy("socks5h://user:password@myproxy.network")
echo socksProxy.url.scheme # выводит "socks5h"
```

```nim
import std/[os, httpclient]

var proxyUrl = ""
try:
  if existsEnv("http_proxy"):
    proxyUrl = getEnv("http_proxy")
  elif existsEnv("https_proxy"):
    proxyUrl = getEnv("https_proxy")
except ValueError:
  echo "не удалось разобрать адрес прокси из переменных окружения"

let envProxy = newProxy(url = proxyUrl)
let client = newHttpClient(proxy = envProxy)
close(client)
```

---

## Multipart-данные для форм и файлов

### `newMultipartData`

```nim
proc newMultipartData*: MultipartData
proc newMultipartData*(xs: MultipartEntries): MultipartData
```

Что делает: первая форма создаёт пустой контейнер, вторая — сразу заполняет его переданными парами "имя, значение".

Список параметров:

- `xs` — массив пар `(name, content)` без файлов и без явного content-type.

Пример:

```nim
import std/httpclient

let empty = newMultipartData()
let filled = newMultipartData({"action": "login", "format": "json"})
echo filled # выводит блок с двумя полями "action" и "format"
```

---

### `add` (запись имя/значение)

```nim
proc add*(p: MultipartData, name, content: string, filename: string = "",
          contentType: string = "", useStream = true)
```

Что делает: добавляет в multipart-данные одну запись; если `filename` не пустая строка, запись считается файлом, иначе — обычным текстовым полем формы.

Разбор реализации: перед добавлением проверяются на символы новой строки `name`, `filename` и `contentType` — это защита от инъекции лишних заголовков внутрь multipart-тела через подделанные значения; при `filename != ""` дополнительно сохраняется `useStream`, который позднее решает, будет ли файл читаться с диска потоково или заранее целиком в память.

Список параметров:

- `p` — контейнер `MultipartData`, изменяется на месте.
- `name` — имя поля формы.
- `content` — значение поля или (для файла) путь к файлу на диске.
- `filename` — имя файла, если запись — файл; пустая строка означает обычное текстовое поле.
- `contentType` — MIME-тип файла.
- `useStream` — при `true` файл передаётся потоком прямо при отправке запроса, при `false` предварительно читается в память.

Примеры:

```nim
import std/httpclient

var data = newMultipartData()
add(data, "username", "NimUser")
echo data # выводит одно текстовое поле "username"
```

```nim
import std/httpclient

var data = newMultipartData()
doAssertRaises(ValueError):
  add(data, "bad\nname", "value")
```

---

### `add` (список записей)

```nim
proc add*(p: MultipartData, xs: MultipartEntries): MultipartData {.discardable.}
```

Что делает: добавляет сразу несколько текстовых полей из массива пар и возвращает тот же контейнер — удобно для цепочек вызовов.

Список параметров:

- `p` — контейнер, в который добавляются записи.
- `xs` — массив пар `(name, content)`.

Пример:

```nim
import std/httpclient

var data = newMultipartData()
add(data, {"action": "login", "format": "json"})
echo data # выводит два текстовых поля: "action" и "format"
```

---

### `addFiles`

```nim
proc addFiles*(p: MultipartData, xs: openArray[tuple[name, file: string]],
               mimeDb = newMimetypes(), useStream = true): MultipartData {.discardable.}
```

Что делает: добавляет в multipart-данные один или несколько файлов с диска; MIME-тип каждого файла определяется автоматически по расширению.

Разбор реализации: `mimeDb` по умолчанию создаётся заново при каждом вызове (`newMimetypes()`) — если процедура вызывается часто, это лишняя работа, поэтому модуль явно рекомендует передавать свою базу через параметр `mimeDb`, чтобы не пересоздавать её каждый раз.

Список параметров:

- `p` — контейнер, в который добавляются файлы.
- `xs` — массив пар `(name, file)`, где `file` — путь к файлу на диске.
- `mimeDb` — база MIME-типов для определения `Content-Type` по расширению.
- `useStream` — при `true` (по умолчанию) файлы передаются потоково, при `false` читаются в память целиком — экономнее по памяти не является, поэтому для больших файлов `false` не рекомендуется.

Пример:

```nim
import std/httpclient

var data = newMultipartData()
addFiles(data, {"uploaded_file": "test.html"})
echo data # выводит блок с полем "uploaded_file" и заголовком Content-Type
```

---

### `[]=` (запись значения или файла)

```nim
proc `[]=`*(p: MultipartData, name, content: string)
proc `[]=`*(p: MultipartData, name: string,
            file: tuple[name, contentType, content: string])
```

Что делает: краткая запись для добавления обычного текстового поля (первая форма) или файла с явно заданными именем, типом и содержимым (вторая форма) — обе формы под капотом вызывают `add`.

Список параметров:

- `p` — контейнер `MultipartData`.
- `name` — имя поля формы.
- `content` — значение текстового поля.
- `file` — кортеж `(name, contentType, content)` для файла: имя файла, MIME-тип и его содержимое строкой.

Примеры:

```nim
import std/httpclient

var data = newMultipartData()
data["username"] = "NimUser"
echo data # выводит текстовое поле "username"
```

```nim
import std/httpclient

var data = newMultipartData()
data["uploaded_file"] = ("test.html", "text/html",
  "<html><head></head><body><p>test</p></body></html>")
echo data # выводит блок с полем "uploaded_file" и телом HTML-файла
```

---

### `$` (строковое представление)

```nim
proc `$`*(data: MultipartData): string
```

Что делает: возвращает человекочитаемое представление всех записей `MultipartData` — удобно для отладки перед отправкой запроса.

Разбор реализации: каждая запись выводится в отдельном пронумерованном блоке с разделителями из тире; для файлов дополнительно печатается `filename` и `Content-Type`, для обычных полей — только имя и содержимое.

Список параметров:

- `data` — контейнер, который нужно вывести.

Пример:

```nim
import std/httpclient

var data = newMultipartData()
add(data, "action", "login")
echo data # выводит пронумерованный блок с полем "action" и значением "login"
```

---

## Выполнение запросов

### `request`

```nim
proc request*(client: HttpClient | AsyncHttpClient, url: Uri | string,
              httpMethod: HttpMethod | string = HttpGet, body = "",
              headers: HttpHeaders = nil,
              multipart: MultipartData = nil): Future[Response | AsyncResponse]
```

Что делает: базовая процедура, на которой построены все остальные методы запроса (`get`, `post` и т.д.); отправляет запрос произвольным HTTP-методом и следует редиректам до `client.maxRedirects` включительно.

Разбор реализации: следование редиректам устроено как диспетчер по коду статуса ответа — процедура не просто "повторяет запрос", а выбирает поведение по конкретному коду:

- Коды `301`, `302`, `303` — метод меняется на `GET`, если исходный метод не был `GET` или `HEAD`, тело запроса отбрасывается, а заголовки `Content-Length`, `Content-Type` и `Transfer-Encoding` удаляются, поскольку тела больше нет.
- Коды `307`, `308` — метод и тело запроса сохраняются без изменений, поскольку эти коды по спецификации не позволяют менять исходный запрос.
- При переходе на другой хост (не поддомен исходного) из заголовков удаляются `Host` и `Authorization`, чтобы не раскрывать чувствительные данные стороннему серверу.

Также `httpMethod` может быть передан старой строковой формой (`"GET"`, `"POST"` и т.д.) — это устаревший путь: строка проверяется по списку известных методов и конвертируется в `HttpMethod`, а при неизвестном имени поднимается `ValueError`.

Список параметров:

- `client` — HTTP-клиент, синхронный или асинхронный.
- `url` — адрес запроса; строка не должна содержать символов перевода строки — иначе будет поднят `AssertionDefect`.
- `httpMethod` — метод запроса, по умолчанию `HttpGet`.
- `body` — тело запроса.
- `headers` — заголовки, которые перекрывают `client.headers` только для этого запроса.
- `multipart` — multipart-данные формы; если заданы, `body` игнорируется.

Примеры:

```nim
import std/httpclient

let client = newHttpClient()
let resp = request(client, "http://example.com", HttpGet)
echo resp.status # выводит "200 OK"
close(client)
```

```nim
import std/[httpclient, json]

let client = newHttpClient()
client.headers = newHttpHeaders({"Content-Type": "application/json"})
let payload = %*{"data": "some text"}
let resp = request(client, "http://some.api", HttpPost, body = $payload)
echo resp.status
close(client)
```

```nim
import std/httpclient

let client = newHttpClient()
doAssertRaises(AssertionDefect):
  discard request(client, "http://example.com/\c\Lpath")
close(client)
```

---

### `get` и `getContent`

```nim
proc get*(client: HttpClient | AsyncHttpClient, url: Uri | string): Future[Response | AsyncResponse]
proc getContent*(client: HttpClient | AsyncHttpClient, url: Uri | string): Future[string]
```

Что делает: `get` выполняет GET-запрос и возвращает объект ответа целиком; `getContent` делает то же самое, но сразу читает и возвращает тело ответа строкой, поднимая `HttpRequestError` при коде 4xx или 5xx.

Список параметров:

- `client` — HTTP-клиент.
- `url` — адрес запроса.

Примеры:

```nim
import std/httpclient

let client = newHttpClient()
let page = getContent(client, "http://example.com")
echo "Pizza" in page # выводит false
close(client)
```

```nim
import std/httpclient

let client = newHttpClient()
doAssertRaises(HttpRequestError):
  discard getContent(client, "http://example.com/missing-page")
close(client)
```

---

### `head`

```nim
proc head*(client: HttpClient | AsyncHttpClient, url: Uri | string): Future[Response | AsyncResponse]
```

Что делает: выполняет HEAD-запрос — получает заголовки ответа без тела; полезно, чтобы узнать размер или тип ресурса, не скачивая его целиком.

Список параметров:

- `client` — HTTP-клиент.
- `url` — адрес запроса.

Пример:

```nim
import std/httpclient

let client = newHttpClient()
let resp = head(client, "http://example.com")
echo contentType(resp) # выводит MIME-тип ресурса
close(client)
```

---

### `post` и `postContent`

```nim
proc post*(client: HttpClient | AsyncHttpClient, url: Uri | string, body = "",
           multipart: MultipartData = nil): Future[Response | AsyncResponse]
proc postContent*(client: HttpClient | AsyncHttpClient, url: Uri | string, body = "",
                  multipart: MultipartData = nil): Future[string]
```

Что делает: `post` отправляет POST-запрос с телом или multipart-данными и возвращает ответ целиком; `postContent` дополнительно сразу читает тело ответа строкой.

Список параметров:

- `client` — HTTP-клиент.
- `url` — адрес запроса.
- `body` — тело запроса; игнорируется, если задан `multipart`.
- `multipart` — multipart-данные формы.

Примеры:

```nim
import std/httpclient

let client = newHttpClient()
var data = newMultipartData()
data["output"] = "soap12"
data["uploaded_file"] = ("test.html", "text/html",
  "<html><head></head><body><p>test</p></body></html>")
let result = postContent(client, "http://validator.w3.org/check", multipart = data)
echo result # выводит ответ сервера-валидатора
close(client)
```

```nim
import std/httpclient

let client = newHttpClient()
let resp = post(client, "http://example.com/api", body = "raw text body")
echo resp.status
close(client)
```

---

### `put` и `putContent`

```nim
proc put*(client: HttpClient | AsyncHttpClient, url: Uri | string, body = "",
          multipart: MultipartData = nil): Future[Response | AsyncResponse]
proc putContent*(client: HttpClient | AsyncHttpClient, url: Uri | string, body = "",
                 multipart: MultipartData = nil): Future[string]
```

Что делает: полные аналоги `post`/`postContent`, но с методом PUT — по конвенции REST используются для полной замены ресурса по указанному адресу.

Список параметров: те же, что у [`post`/`postContent`](#post-и-postcontent).

Пример:

```nim
import std/httpclient

let client = newHttpClient()
let resp = put(client, "http://example.com/items/1", body = "новое содержимое")
echo resp.status
close(client)
```

---

### `patch` и `patchContent`

```nim
proc patch*(client: HttpClient | AsyncHttpClient, url: Uri | string, body = "",
            multipart: MultipartData = nil): Future[Response | AsyncResponse]
proc patchContent*(client: HttpClient | AsyncHttpClient, url: Uri | string, body = "",
                   multipart: MultipartData = nil): Future[string]
```

Что делает: аналоги `post`/`postContent` с методом PATCH — по конвенции REST используются для частичного изменения ресурса.

Список параметров: те же, что у [`post`/`postContent`](#post-и-postcontent).

Пример:

```nim
import std/[httpclient, json]

let client = newHttpClient()
let payload = %*{"status": "done"}
let resp = patch(client, "http://example.com/items/1", body = $payload)
echo resp.status
close(client)
```

---

### `delete` и `deleteContent`

```nim
proc delete*(client: HttpClient | AsyncHttpClient, url: Uri | string): Future[Response | AsyncResponse]
proc deleteContent*(client: HttpClient | AsyncHttpClient, url: Uri | string): Future[string]
```

Что делает: `delete` выполняет DELETE-запрос и возвращает ответ целиком; `deleteContent` дополнительно сразу читает тело ответа строкой.

Список параметров:

- `client` — HTTP-клиент.
- `url` — адрес удаляемого ресурса.

Пример:

```nim
import std/httpclient

let client = newHttpClient()
let resp = delete(client, "http://example.com/items/1")
echo resp.status # выводит "204 No Content" или иной код успеха
close(client)
```

---

### `downloadFile`

```nim
proc downloadFile*(client: HttpClient, url: Uri | string, filename: string)
proc downloadFile*(client: AsyncHttpClient, url: Uri | string, filename: string): Future[void]
```

Что делает: скачивает содержимое `url` и сохраняет его напрямую в файл `filename`, не накапливая всё тело ответа в памяти.

Разбор реализации: перед запросом клиенту временно выставляется `getBody = false`, чтобы обычный путь чтения тела не сработал; вместо этого у синхронного клиента `bodyStream` подменяется на файловый поток (`newFileStream(filename, fmWrite)`), и внутренняя процедура разбора тела пишет данные сразу в файл — у асинхронной версии то же самое устроено через `FutureStream` и параллельную запись в файл через `writeFromStream`, пока данные ещё продолжают приходить по сети; после завершения (успешного или нет) флаг `getBody` возвращается в `true` через `defer`/колбэк, чтобы не повлиять на последующие запросы этого же клиента.

Список параметров:

- `client` — HTTP-клиент.
- `url` — адрес скачиваемого ресурса.
- `filename` — путь, по которому будет сохранён файл.

Примеры:

```nim
import std/httpclient

let client = newHttpClient()
downloadFile(client, "http://example.com/file.zip", "file.zip")
echo "файл сохранён" # выводит "файл сохранён"
close(client)
```

```nim
import std/[asyncdispatch, httpclient]

proc fetchAndSave(): Future[void] {.async.} =
  let client = newAsyncHttpClient()
  await downloadFile(client, "http://example.com/file.zip", "file.zip")
  close(client)

waitFor fetchAndSave()
```

---

## Чтение данных ответа

### `code`

```nim
proc code*(response: Response | AsyncResponse): HttpCode {.raises: [ValueError, OverflowDefect].}
```

Что делает: возвращает код ответа как `HttpCode`, разбирая первые три символа строки `status`.

Разбор реализации: поднимает `ValueError`, если строка статуса не начинается с трёхзначного числа — то есть сервер прислал ответ, не соответствующий формату HTTP.

Список параметров:

- `response` — объект ответа.

Пример:

```nim
import std/httpclient

let client = newHttpClient()
let resp = get(client, "http://example.com")
echo code(resp) # выводит Http200
close(client)
```

---

### `contentType`

```nim
proc contentType*(response: Response | AsyncResponse): string
```

Что делает: возвращает значение заголовка `Content-Type` ответа; если заголовок отсутствует, возвращает пустую строку.

Список параметров:

- `response` — объект ответа.

Пример:

```nim
import std/httpclient

let client = newHttpClient()
let resp = get(client, "http://example.com")
echo contentType(resp) # выводит "text/html; charset=UTF-8"
close(client)
```

---

### `contentLength`

```nim
proc contentLength*(response: Response | AsyncResponse): int
```

Что делает: возвращает значение заголовка `Content-Length` как число; если заголовок отсутствует, возвращает `-1`.

Разбор реализации: `-1` используется как явный маркер "заголовок не задан", а не `0`, поскольку `0` — валидное и отличное по смыслу значение (пустое тело).

Список параметров:

- `response` — объект ответа.

Примеры:

```nim
import std/httpclient

let client = newHttpClient()
let resp = get(client, "http://example.com")
echo contentLength(resp) >= 0 # выводит true, если заголовок присутствует
close(client)
```

```nim
import std/httpclient

let client = newHttpClient()
let resp = head(client, "http://example.com/no-length-header")
doAssertRaises(ValueError):
  discard contentLength(resp) # выводит ошибку, если значение заголовка не число
close(client)
```

---

### `lastModified`

```nim
proc lastModified*(response: Response | AsyncResponse): DateTime
```

Что делает: разбирает заголовок `Last-Modified` и возвращает его как `DateTime`.

Разбор реализации: используется фиксированный формат `"ddd, dd MMM yyyy HH:mm:ss 'GMT'"`, соответствующий HTTP-дате по стандарту; если заголовок отсутствует или не соответствует этому формату, поднимается `ValueError`.

Список параметров:

- `response` — объект ответа.

Пример:

```nim
import std/httpclient

let client = newHttpClient()
let resp = head(client, "http://example.com")
echo lastModified(resp) # выводит дату последнего изменения ресурса
close(client)
```

---

### `body`

```nim
proc body*(response: Response): string
proc body*(response: AsyncResponse): Future[string]
```

Что делает: читает и возвращает тело ответа целиком; повторные вызовы не читают поток заново, а возвращают ранее сохранённое значение.

Разбор реализации: обе версии проверяют `response.body.len == 0` перед чтением — это одновременно и признак "тело ещё не читалось", и упрощение: пустое тело будет читаться заново при каждом вызове, но для непустых тел повторное чтение из уже исчерпанного потока не требуется.

Список параметров:

- `response` — объект ответа, синхронный или асинхронный.

Пример:

```nim
import std/httpclient

let client = newHttpClient()
let resp = get(client, "http://example.com")
let content = body(resp)
echo content == body(resp) # выводит true — второй вызов вернул кешированное значение
close(client)
```

---

## Практические рецепты

### Скачивание страницы с обработкой ошибок

```nim
import std/httpclient

let client = newHttpClient()
try:
  let page = getContent(client, "http://example.com")
  echo "получено байт: " & $page.len
except HttpRequestError:
  echo "сервер вернул код ошибки"
finally:
  close(client)
```

Типичный шаблон: `getContent` избавляет от ручной проверки кода ответа, но требует блока `try`/`except`, поскольку ошибочный статус превращается в исключение.

---

### Отправка JSON и разбор ответа

```nim
import std/[httpclient, json]

let client = newHttpClient()
client.headers = newHttpHeaders({"Content-Type": "application/json"})
let payload = %*{"login": "user", "password": "secret"}
let resp = request(client, "http://example.com/login", HttpPost, body = $payload)
if code(resp) == Http200:
  let parsed = parseJson(body(resp))
  echo parsed["token"].getStr()
close(client)
```

Комбинация `request` с явным методом и заголовком `Content-Type` — стандартный способ работы с JSON-API, когда нужен доступ и к статусу, и к телу ответа.

---

### Загрузка файла на сервер с отслеживанием прогресса

```nim
import std/[asyncdispatch, httpclient]

proc onProgress(total, progress, speed: BiggestInt): Future[void] {.async.} =
  echo "загружено ", progress, " из ", total, ", скорость ", speed div 1000, " кб/с"

proc uploadFile(): Future[void] {.async.} =
  let client = newAsyncHttpClient()
  client.onProgressChanged = onProgress
  var data = newMultipartData()
  addFiles(data, {"uploaded_file": "big_archive.zip"})
  discard await postContent(client, "http://example.com/upload", multipart = data)
  client.onProgressChanged = nil
  close(client)

waitFor uploadFile()
```

Колбэк `onProgressChanged` полезен именно для крупных файлов — для мелких запросов он может ни разу не сработать, поскольку вызывается примерно раз в секунду.

---

### Скачивание файла напрямую на диск

```nim
import std/httpclient

let client = newHttpClient(timeout = 10000)
downloadFile(client, "http://example.com/report.pdf", "report.pdf")
close(client)
```

`downloadFile` предпочтительнее `getContent` для больших файлов, поскольку не держит всё содержимое в памяти клиента одновременно.

---

### Запрос через прокси из переменных окружения с ограничением редиректов

```nim
import std/[os, httpclient]

var proxyUrl = ""
if existsEnv("https_proxy"):
  proxyUrl = getEnv("https_proxy")

let client = newHttpClient(proxy = newProxy(proxyUrl), maxRedirects = 2)
let resp = get(client, "https://example.com")
echo code(resp)
close(client)
```

Ограничение `maxRedirects` до небольшого числа — практика для доверенных внутренних сервисов, где длинная цепочка редиректов обычно означает ошибку конфигурации, а не легитимный сценарий.

---

## Краткая таблица

| Задача | Что использовать |
| --- | --- |
| Получить содержимое страницы строкой | `getContent` |
| Получить статус, заголовки и тело отдельно | `get`, затем `code`/`contentType`/`body` |
| Узнать тип или размер ресурса без скачивания тела | `head`, затем `contentType`/`contentLength` |
| Отправить форму или JSON и получить только текст ответа | `postContent` |
| Отправить форму или JSON и разобрать статус | `post`, затем `code`/`body` |
| Загрузить файл(ы) в форме | `newMultipartData` + `addFiles`, передать в `postContent`/`post` |
| Частично или полностью изменить ресурс | `patch`/`patchContent` или `put`/`putContent` |
| Удалить ресурс | `delete`/`deleteContent` |
| Сохранить большой файл сразу на диск | `downloadFile` |
| Работать через прокси | `newProxy`, передать в `newHttpClient`/`newAsyncHttpClient` |
| Отслеживать ход длительной загрузки | поле `onProgressChanged` клиента |
| Ограничить время ожидания ответа | параметр `timeout` в `newHttpClient` |
| Произвольный HTTP-метод или полный контроль над запросом | `request` |

---

## Сводка: какую процедуру выбрать

- Нужна только строка тела ответа → используйте `getContent`.
- Нужны статус и заголовки вместе с телом → используйте `get`, а затем `code`, `contentType`, `body`.
- Нужно проверить ресурс, не скачивая его → используйте `head`.
- Нужно отправить данные формы или JSON → используйте `post`/`postContent`.
- Нужно отправить файл(ы) в форме → используйте `newMultipartData`/`addFiles` вместе с `post`/`postContent`.
- Нужно частично обновить ресурс → используйте `patch`/`patchContent`.
- Нужно полностью заменить ресурс → используйте `put`/`putContent`.
- Нужно удалить ресурс → используйте `delete`/`deleteContent`.
- Нужно сохранить крупный файл без лишнего расхода памяти → используйте `downloadFile`.
- Нужен нестандартный метод, точный контроль заголовков или ручная обработка редиректов → используйте `request` напрямую.
- Нужно работать через прокси-сервер → создайте `Proxy` через `newProxy` и передайте его при создании клиента.
- Нужно показывать пользователю прогресс длительной загрузки → назначьте `client.onProgressChanged`.
