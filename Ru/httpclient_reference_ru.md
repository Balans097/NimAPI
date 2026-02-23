# Справочник модуля `httpclient`

> **Стандартная библиотека Nim — HTTP-клиент**
> Статус API: частично **нестабильный**; основной API стабилен.

---

## Общее описание

`httpclient` предоставляет полноценный HTTP/1.1-клиент как для **синхронного**, так и для **асинхронного** использования. Одного импорта достаточно, чтобы выполнять GET-, POST-, PUT-, PATCH-, DELETE- и HEAD-запросы, обрабатывать редиректы, работать с multipart-формами, потоково передавать большие файлы, отслеживать прогресс загрузки и соединяться через прокси.

Модуль предоставляет два типа клиентов с идентичным API:

- `HttpClient` — блокирующий, подходит для скриптов и простых утилит.
- `AsyncHttpClient` — неблокирующий, построен на `asyncdispatch`, подходит для серверов и конкурентного кода.

```nim
import std/httpclient
```

Для HTTPS компилируйте с флагом `-d:ssl`.

> ⚠️ **Замечание о безопасности:** парсеры URI в этом модуле не обнаруживают вредоносные URI. Всегда проверяйте ненадёжные данные перед формированием URL запроса.

---

## Типы

### `Response` / `AsyncResponse`

Объект, возвращаемый каждой процедурой запроса. Оба типа предоставляют идентичные поля и процедуры-аксессоры.

```nim
type Response* = ref object
  version*: string      # HTTP-версия: "1.0" или "1.1"
  status*: string       # Строка статуса, например "200 OK"
  headers*: HttpHeaders # Все заголовки ответа
  bodyStream*: Stream   # Сырой поток тела (синхронный)

type AsyncResponse* = ref object
  version*: string
  status*: string
  headers*: HttpHeaders
  bodyStream*: FutureStream[string]  # Сырой поток тела (асинхронный)
```

### `Proxy`

Хранит параметры прокси-соединения. Создаётся через `newProxy`.

### `MultipartData`

Хранит поля формы и файлы для запросов `multipart/form-data`. Создаётся через `newMultipartData`.

### `ProgressChangedProc`

Тип колбека для отчёта о прогрессе загрузки:

```nim
type ProgressChangedProc[ReturnType] =
  proc (total, progress, speed: BiggestInt): ReturnType {.closure, gcsafe.}
```

Для `HttpClient` возвращаемый тип — `void`; для `AsyncHttpClient` — `Future[void]`.

### `ProtocolError`

Генерируется, когда ответ сервера не соответствует HTTP/1.x.

### `HttpRequestError`

Генерируется процедурами `getContent`, `postContent` и всеми остальными `*Content`-процедурами при получении от сервера статуса 4xx или 5xx.

---

## Константы

### `defUserAgent`

```nim
const defUserAgent* = "Nim-httpclient/" & NimVersion
```

Значение заголовка `User-Agent` по умолчанию, отправляемое с каждым запросом. Можно переопределить для конкретного клиента через `newHttpClient(userAgent = "MyApp/1.0")`.

---

## Создание клиента

### `newHttpClient`

```nim
proc newHttpClient*(
  userAgent    = defUserAgent,
  maxRedirects = 5,
  sslContext   = getDefaultSSL(),
  proxy        : Proxy = nil,
  timeout      = -1,
  headers      = newHttpHeaders()
): HttpClient
```

#### Что делает

Создаёт синхронный HTTP-клиент. Клиент поддерживает постоянное TCP-соединение — повторные запросы к тому же хосту переиспользуют его без переподключения. По завершении работы необходимо вызвать `close()`.

#### Параметры

| Параметр | По умолчанию | Описание |
|----------|-------------|---------|
| `userAgent` | `"Nim-httpclient/X.Y.Z"` | Значение заголовка запроса `User-Agent`. |
| `maxRedirects` | `5` | Максимальное число автоматических редиректов. Установите `0` для отключения. |
| `sslContext` | системный | Контекст OpenSSL для HTTPS. Управляет режимом проверки сертификатов. |
| `proxy` | `nil` | Необязательный HTTP- или SOCKS5-прокси. |
| `timeout` | `-1` (без ограничений) | Таймаут сокета в **миллисекундах**. Применяется к отдельным операциям сокета, а не ко всему запросу. |
| `headers` | пустые | Заголовки по умолчанию, отправляемые с каждым запросом этого клиента. |

#### Пример

```nim
import std/httpclient

# Минимально: GET в одну строку
let html = newHttpClient().getContent("http://example.com")

# Настроенный клиент
let client = newHttpClient(
  userAgent    = "MyBot/2.0",
  timeout      = 5_000,   # таймаут сокета 5 секунд
  maxRedirects = 3
)
defer: client.close()
echo client.getContent("https://api.example.com/data")
```

---

### `newAsyncHttpClient`

```nim
proc newAsyncHttpClient*(
  userAgent    = defUserAgent,
  maxRedirects = 5,
  sslContext   = getDefaultSSL(),
  proxy        : Proxy = nil,
  headers      = newHttpHeaders()
): AsyncHttpClient
```

#### Что делает

Создаёт **асинхронный** HTTP-клиент для использования внутри `async`-процедур. Принимает те же параметры, что и `newHttpClient`, кроме `timeout` — для async-клиентов он пока не поддерживается.

Один экземпляр `AsyncHttpClient` обрабатывает **один запрос за раз**. Для параллельных запросов создайте несколько экземпляров.

#### Пример

```nim
import std/[asyncdispatch, httpclient]

proc fetchAll(): Future[void] {.async.} =
  # Два клиента → два параллельных запроса
  let c1 = newAsyncHttpClient()
  let c2 = newAsyncHttpClient()
  defer:
    c1.close()
    c2.close()

  let (r1, r2) = await all(
    c1.getContent("https://api.example.com/users"),
    c2.getContent("https://api.example.com/posts")
  )
  echo r1
  echo r2

waitFor fetchAll()
```

---

## Управление клиентом

### `close`

```nim
proc close*(client: HttpClient | AsyncHttpClient)
```

#### Что делает

Закрывает TCP-сокет и помечает клиент как отключённый. Всегда должна вызываться по завершении работы с клиентом — используйте `defer`, чтобы гарантировать вызов даже при исключении.

```nim
let client = newHttpClient()
defer: client.close()   # выполнится даже если тело бросит исключение
let body = client.getContent("https://example.com")
```

---

### `getSocket`

```nim
proc getSocket*(client: HttpClient): Socket
proc getSocket*(client: AsyncHttpClient): AsyncSocket
```

#### Что делает

Возвращает низкоуровневый сетевой сокет. Полезно для получения деталей соединения — локального и удалённого адресов.

```nim
let client = newHttpClient()
defer: client.close()
discard client.get("http://example.com")

if client.connected:
  echo "Локальный:  ", client.getSocket.getLocalAddr
  echo "Удалённый:  ", client.getSocket.getPeerAddr
```

---

## Аксессоры ответа

Все процедуры работают как с `Response`, так и с `AsyncResponse`.

### `code`

```nim
proc code*(response: Response | AsyncResponse): HttpCode
```

Разбирает строку `status` ответа и возвращает числовой HTTP-код в виде `HttpCode`. Генерирует `ValueError`, если строка статуса сформирована некорректно.

```nim
let resp = client.get("https://httpbin.org/status/404")
echo resp.code           # Http404
echo resp.code == Http200  # false
```

---

### `contentType`

```nim
proc contentType*(response: Response | AsyncResponse): string
```

Возвращает значение заголовка `Content-Type` ответа, или пустую строку, если заголовок отсутствует.

```nim
let resp = client.get("https://httpbin.org/json")
echo resp.contentType   # "application/json"
```

---

### `contentLength`

```nim
proc contentLength*(response: Response | AsyncResponse): int
```

Возвращает значение заголовка `Content-Length` как целое число. Возвращает `-1`, если заголовок отсутствует. Генерирует `ValueError`, если значение не является целым числом.

```nim
let resp = client.get("https://example.com")
echo resp.contentLength   # например, 1256, или -1 для chunked-ответов
```

---

### `lastModified`

```nim
proc lastModified*(response: Response | AsyncResponse): DateTime
```

Разбирает заголовок `Last-Modified` и возвращает `DateTime` в UTC. Генерирует `ValueError`, если заголовок отсутствует или не может быть разобран как HTTP-дата.

```nim
let resp = client.get("https://example.com")
echo resp.lastModified   # например, 2024-01-15T10:30:00Z
```

---

### `body` (синхронный)

```nim
proc body*(response: Response): string
```

Читает и возвращает всё тело ответа в виде строки. Результат **кэшируется**: повторный вызов `body` вернёт уже прочитанные данные без повторного чтения потока.

```nim
let resp = client.get("https://httpbin.org/get")
echo resp.body   # полная JSON-строка
```

---

### `body` (асинхронный)

```nim
proc body*(response: AsyncResponse): Future[string] {.async.}
```

Асинхронный аналог: читает и кэширует всё тело, возвращая `Future[string]`.

```nim
let resp = await client.get("https://httpbin.org/get")
echo await resp.body
```

---

## Процедуры запросов

Каждая процедура существует в двух вариантах:

- `verb(client, url)` → возвращает `Response`/`AsyncResponse`. Позволяет отдельно проверить статус, заголовки и тело.
- `verbContent(client, url)` → возвращает тело напрямую в виде строки, **и генерирует `HttpRequestError`** при ответах 4xx/5xx.

### `request`

```nim
proc request*(
  client     : HttpClient | AsyncHttpClient,
  url        : Uri | string,
  httpMethod : HttpMethod | string = HttpGet,
  body       = "",
  headers    : HttpHeaders = nil,
  multipart  : MultipartData = nil
): Future[Response | AsyncResponse] {.multisync.}
```

#### Что делает

Универсальная процедура запроса — все остальные (`get`, `post` и т.д.) делегируют работу ей. Отправляет HTTP-запрос любым методом, с необязательным телом, заголовками для конкретного запроса и multipart-данными. Автоматически следует редиректам до `client.maxRedirects`.

Параметр `headers` **переопределяет** `client.headers` только для этого запроса — изменения не сохраняются.

URL не должен содержать символы новой строки; нарушение вызывает `AssertionDefect`.

Передача строки в `httpMethod` устарела с Nim 1.5 — используйте значения перечисления `HttpMethod` (`HttpGet`, `HttpPost` и т.д.).

#### Поведение при редиректах

| Код статуса | Метод после редиректа | Тело после редиректа |
|------------|----------------------|---------------------|
| 301, 302, 303 | Меняется на `GET` (если исходный не GET/HEAD) | Очищается |
| 307, 308 | Сохраняется | Сохраняется |

При редиректе на другой хост заголовки `Authorization` и `Host` автоматически удаляются, чтобы предотвратить утечку учётных данных.

#### Пример

```nim
import std/[httpclient, json]

let client = newHttpClient()
defer: client.close()

# POST с JSON-телом и заголовком Content-Type для этого запроса
let body = $(%*{"name": "Alice", "score": 42})
let resp = client.request(
  "https://api.example.com/users",
  httpMethod = HttpPost,
  body       = body,
  headers    = newHttpHeaders({"Content-Type": "application/json"})
)
echo resp.status    # "201 Created"
echo resp.body
```

---

### `head`

```nim
proc head*(client: HttpClient | AsyncHttpClient,
           url: Uri | string): Future[Response | AsyncResponse] {.multisync.}
```

Отправляет HTTP-запрос `HEAD`. Сервер возвращает заголовки, **но не тело**. Полезно для проверки существования ресурса или получения его метаданных без скачивания.

```nim
let resp = client.head("https://example.com")
echo resp.code          # Http200
echo resp.contentLength # размер в байтах без скачивания
```

---

### `get`

```nim
proc get*(client: HttpClient | AsyncHttpClient,
          url: Uri | string): Future[Response | AsyncResponse] {.multisync.}
```

Отправляет HTTP-запрос `GET` и возвращает полный объект ответа с доступом к статусу, заголовкам и телу.

```nim
let resp = client.get("https://httpbin.org/get")
if resp.code == Http200:
  echo resp.body
```

---

### `getContent`

```nim
proc getContent*(client: HttpClient | AsyncHttpClient,
                 url: Uri | string): Future[string] {.multisync.}
```

Отправляет `GET`-запрос и возвращает **только тело** в виде строки. Генерирует `HttpRequestError` при любом ответе 4xx или 5xx, что упрощает обработку ошибок.

```nim
try:
  let html = client.getContent("https://example.com")
  echo html
except HttpRequestError as e:
  echo "Ошибка сервера: ", e.msg
```

---

### `delete`

```nim
proc delete*(client: HttpClient | AsyncHttpClient,
             url: Uri | string): Future[Response | AsyncResponse] {.multisync.}
```

Отправляет HTTP-запрос `DELETE`. Возвращает полный ответ.

```nim
let resp = client.delete("https://api.example.com/items/42")
echo resp.status   # "204 No Content"
```

---

### `deleteContent`

```nim
proc deleteContent*(client: HttpClient | AsyncHttpClient,
                    url: Uri | string): Future[string] {.multisync.}
```

Отправляет `DELETE`-запрос и возвращает тело ответа. Генерирует `HttpRequestError` при 4xx/5xx.

---

### `post`

```nim
proc post*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[Response | AsyncResponse] {.multisync.}
```

Отправляет HTTP-запрос `POST` с необязательным телом в виде текста или multipart-данными. Возвращает полный ответ.

```nim
let resp = client.post("https://httpbin.org/post", body = "hello=world")
echo resp.body
```

---

### `postContent`

```nim
proc postContent*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[string] {.multisync.}
```

Отправляет `POST`-запрос и возвращает тело ответа в виде строки. Генерирует `HttpRequestError` при ошибке.

```nim
let result = client.postContent(
  "https://api.example.com/echo",
  body = "ping"
)
echo result  # "pong"
```

---

### `put`

```nim
proc put*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[Response | AsyncResponse] {.multisync.}
```

Отправляет HTTP-запрос `PUT`. Семантически используется для **полной замены** существующего ресурса.

```nim
let resp = client.put(
  "https://api.example.com/items/1",
  body = """{"name":"Обновлено"}"""
)
echo resp.status
```

---

### `putContent`

```nim
proc putContent*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[string] {.multisync.}
```

Отправляет `PUT`-запрос и возвращает тело ответа. Генерирует `HttpRequestError` при ошибке.

---

### `patch`

```nim
proc patch*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[Response | AsyncResponse] {.multisync.}
```

Отправляет HTTP-запрос `PATCH`. Используется для **частичного обновления** ресурса.

```nim
let resp = client.patch(
  "https://api.example.com/items/1",
  body = """{"status":"archived"}"""
)
echo resp.code
```

---

### `patchContent`

```nim
proc patchContent*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[string] {.multisync.}
```

Отправляет `PATCH`-запрос и возвращает тело ответа. Генерирует `HttpRequestError` при ошибке.

---

### `downloadFile`

```nim
proc downloadFile*(client: HttpClient, url: Uri | string, filename: string)
proc downloadFile*(client: AsyncHttpClient, url: Uri | string, filename: string): Future[void]
```

#### Что делает

Скачивает ресурс по `url` и сохраняет его напрямую в файл `filename` на диске. Тело ответа **передаётся потоком** — никогда полностью не буферизуется в памяти, что делает процедуру безопасной для файлов произвольного размера.

Генерирует `HttpRequestError` при ответах 4xx/5xx. Генерирует `IOError`, если файл не удаётся открыть на запись.

Колбек прогресса (`onProgressChanged`) работает в штатном режиме в процессе скачивания.

#### Пример

```nim
# Синхронно
let client = newHttpClient()
defer: client.close()
client.downloadFile("https://example.com/large.zip", "large.zip")

# Асинхронно
proc dlAsync() {.async.} =
  let client = newAsyncHttpClient()
  defer: client.close()
  await client.downloadFile("https://example.com/large.zip", "large.zip")

waitFor dlAsync()
```

---

## Поддержка прокси

### `newProxy`

```nim
proc newProxy*(url: Uri): Proxy
proc newProxy*(url: string): Proxy
```

#### Что делает

Создаёт объект `Proxy` из URL. URL может содержать учётные данные для базовой аутентификации (`http://user:pass@host:port`). Поддерживаемые схемы:

- `http://` — обычный HTTP-прокси.
- `https://` — HTTPS-прокси (требует `-d:ssl`).
- `socks5h://` — SOCKS5-прокси с **разрешением DNS на стороне прокси**.

Двухаргументные перегрузки `newProxy(url, auth)` **устарели** — включайте учётные данные прямо в URL.

#### Пример

```nim
import std/httpclient

# Простой прокси
let client = newHttpClient(proxy = newProxy("http://proxy.company.com:8080"))

# Прокси с учётными данными
let client2 = newHttpClient(proxy = newProxy("http://alice:s3cret@proxy.company.com:8080"))

# SOCKS5-прокси (DNS разрешается на стороне прокси)
let client3 = newHttpClient(proxy = newProxy("socks5h://proxy.company.com:1080"))

# Получить прокси из переменных окружения
import std/os
let proxyUrl = getEnv("http_proxy", getEnv("https_proxy"))
if proxyUrl.len > 0:
  let client4 = newHttpClient(proxy = newProxy(proxyUrl))
```

---

## Multipart-данные форм

Multipart-данные используются для HTML-форм с загрузкой файлов (`enctype="multipart/form-data"`).

### `newMultipartData` (пустой)

```nim
proc newMultipartData*(): MultipartData
```

Создаёт пустой контейнер `MultipartData`. Поля и файлы добавляются затем через `add`, `[]=` и `addFiles`.

---

### `newMultipartData` (с полями)

```nim
proc newMultipartData*(xs: MultipartEntries): MultipartData
```

Создаёт `MultipartData`, сразу заполненный текстовыми полями.

```nim
var data = newMultipartData({"action": "login", "format": "json"})
```

---

### `add` (одно поле)

```nim
proc add*(p: MultipartData, name, content: string,
          filename    = "",
          contentType = "",
          useStream   = true)
```

#### Что делает

Добавляет одну запись в `p`. Если `filename` непустой, запись считается файловым вложением — заголовок `Content-Disposition` будет включать `filename=`, а `contentType` задаёт MIME-тип части. Генерирует `ValueError`, если любой из строковых аргументов содержит символы новой строки (это повредило бы MIME-разграничение).

Когда `useStream` равен `true` (по умолчанию) и запись является файлом, он читается с диска потоком во время отправки запроса. При `false` — считывается в память немедленно.

```nim
var data = newMultipartData()
data.add("username", "Alice")
data.add("avatar", "/home/alice/photo.png", "photo.png", "image/png")
```

---

### `add` (несколько полей)

```nim
proc add*(p: MultipartData, xs: MultipartEntries): MultipartData {.discardable.}
```

Пакетно добавляет список текстовых полей. Возвращает `p` для цепочки вызовов.

```nim
data.add({"field1": "value1", "field2": "value2"})
```

---

### `[]=` (текстовое поле)

```nim
proc `[]=`*(p: MultipartData, name, content: string)
```

Сокращённая запись для добавления простого текстового поля — без имени файла и без типа содержимого.

```nim
data["username"] = "Alice"
data["language"] = "Nim"
```

---

### `[]=` (файловый кортеж)

```nim
proc `[]=`*(p: MultipartData, name: string,
            file: tuple[name, contentType, content: string])
```

Добавляет файловую запись с явно заданными именем файла, MIME-типом и содержимым (в виде строки). Удобно, когда содержимое файла уже находится в памяти.

```nim
data["document"] = ("report.html", "text/html",
                    "<html><body>Привет</body></html>")
```

---

### `addFiles`

```nim
proc addFiles*(
  p         : MultipartData,
  xs        : openArray[tuple[name, file: string]],
  mimeDb    = newMimetypes(),
  useStream = true
): MultipartData {.discardable.}
```

#### Что делает

Читает файлы с диска и добавляет их как части multipart. MIME-тип **определяется автоматически** по расширению файла с помощью `mimeDb`. При `useStream = true` файлы читаются чанками во время запроса, а не загружаются в память — подходит для больших файлов.

Генерирует `IOError`, если файл не удаётся открыть. Передайте предварительно созданный `mimeDb`, чтобы избежать создания новой базы MIME-типов при каждом вызове.

```nim
import std/[httpclient, mimetypes]

let mimes = newMimetypes()
var data = newMultipartData()
data.addFiles(
  {"photo": "images/avatar.jpg", "resume": "docs/cv.pdf"},
  mimeDb = mimes
)
let resp = client.post("https://upload.example.com/profile", multipart = data)
```

---

### `$` (строковое представление)

```nim
proc `$`*(data: MultipartData): string
```

Преобразует `MultipartData` в читаемую строку, показывая имя, имя файла, тип содержимого и содержимое каждой части. Удобно для отладки.

```nim
echo $data
# ------ 0 ------
# name="username"
#
# Alice
# ------ 1 ------
# name="avatar"; filename="photo.png"
# Content-Type: image/png
# ...
```

---

## Отчёт о прогрессе

Назначьте колбек `client.onProgressChanged`, чтобы получать обновления прогресса загрузки раз в секунду.

```nim
import std/[asyncdispatch, httpclient]

proc onProgress(total, progress, speed: BiggestInt) {.async.} =
  if total > 0:
    echo progress, " / ", total, " байт  (", speed div 1024, " КБ/с)"
  else:
    echo progress, " байт  (общий размер неизвестен)"

proc downloadWithProgress() {.async.} =
  let client = newAsyncHttpClient()
  client.onProgressChanged = onProgress
  defer: client.close()
  await client.downloadFile("https://example.com/large.iso", "large.iso")

waitFor downloadWithProgress()
```

> ⚠️ Параметр `total` может быть равен `0`, если сервер не отправляет заголовок `Content-Length` (например, при chunked transfer encoding).

Чтобы убрать колбек: `client.onProgressChanged = nil`.

---

## SSL/TLS

HTTPS активируется автоматически, когда схема URL — `https://`. Необходимо скомпилировать с `-d:ssl`:

```sh
nim c -d:ssl myapp.nim
```

Проверка сертификатов **включена по умолчанию** (`CVerifyPeer`). Для настройки:

```nim
import std/[net, httpclient]

# Проверять сертификаты (по умолчанию)
let client = newHttpClient(sslContext = newContext(verifyMode = CVerifyPeer))

# Отключить проверку (не рекомендуется для продакшна)
let clientNoVerify = newHttpClient(sslContext = newContext(verifyMode = CVerifyNone))

# Использовать переменные окружения SSL_CERT_FILE / SSL_CERT_DIR
let clientEnv = newHttpClient(sslContext = newContext(verifyMode = CVerifyPeerUseEnvVars))
```

---

## Таймауты

Только `HttpClient` (синхронный) поддерживает таймаут. Он задаётся в **миллисекундах** и применяется к отдельным **операциям сокета**, а не ко всему запросу. Если сервер продолжает передавать данные — таймаут не сработает; он срабатывает только когда отдельная операция чтения/записи зависает.

```nim
# Генерировать TimeoutError, если любая операция сокета занимает более 3 секунд
let client = newHttpClient(timeout = 3_000)
```

---

## Краткая шпаргалка

```
Задача                                         Процедура
──────────────────────────────────────────────────────────────────────
Получить тело страницы (ошибка при 4xx/5xx)    getContent(client, url)
Получить полный ответ                          get(client, url)
Проверить существование / метаданные           head(client, url)
Отправить форму / JSON                         post(client, url, body)
Отправить форму / JSON, получить тело          postContent(client, url, body)
Заменить ресурс целиком                        put(client, url, body)
Частично обновить ресурс                       patch(client, url, body)
Удалить ресурс                                 delete(client, url)
Загрузить файлы на сервер                      post(client, url, multipart=data)
Скачать файл на диск                           downloadFile(client, url, path)
Произвольный метод / полный контроль           request(client, url, httpMethod)
```
