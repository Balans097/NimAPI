# Справочник модуля `std/httpclient` (Nim)

> Модуль реализует HTTP-клиент для получения веб-страниц и других данных.  
> **Предупреждение:** Не передавайте непроверенные данные в URI-парсеры — они не защищены от вредоносных URI.

---

## Содержание

1. [Типы и структуры данных](#типы-и-структуры-данных)
2. [Работа с прокси](#работа-с-прокси)
3. [Работа с MultipartData](#работа-с-multipartdata)
4. [Создание клиентов](#создание-клиентов)
5. [Управление клиентом](#управление-клиентом)
6. [Работа с ответом сервера (Response)](#работа-с-ответом-сервера)
7. [HTTP-запросы](#http-запросы)
8. [Константы](#константы)
9. [Исключения](#исключения)

---

## Типы и структуры данных

### `Response`

Синхронный ответ HTTP-сервера.

```nim
type Response* = ref object
  version*: string       # Версия HTTP ("1.0" или "1.1")
  status*: string        # Строка статуса, например "200 OK"
  headers*: HttpHeaders  # Заголовки ответа
  bodyStream*: Stream    # Поток тела ответа
```

### `AsyncResponse`

Асинхронный ответ HTTP-сервера.

```nim
type AsyncResponse* = ref object
  version*: string
  status*: string
  headers*: HttpHeaders
  bodyStream*: FutureStream[string]
```

### `MultipartData`

Объект для формирования тела запроса типа `multipart/form-data`.

### `MultipartEntries`

Псевдоним для `openArray[tuple[name, content: string]]` — список пар «имя поля / значение».

### `Proxy`

Объект, описывающий HTTP/SOCKS5-прокси.

```nim
type Proxy* = ref object
  url*: Uri
```

### `ProgressChangedProc`

Тип колбэка для отслеживания прогресса загрузки.

```nim
type ProgressChangedProc*[ReturnType] =
  proc (total, progress, speed: BiggestInt): ReturnType {.closure, gcsafe.}
```

- `total` — полный ожидаемый размер (может быть 0, если сервер не передаёт `Content-Length`).
- `progress` — уже скачанный объём в байтах.
- `speed` — скорость за последнюю секунду в байтах.

---

## Работа с прокси

### `newProxy(url: Uri): Proxy`
### `newProxy(url: string): Proxy`

Создаёт объект прокси. Авторизационные данные можно указать прямо в URL (`http://user:password@host`).

```nim
import std/httpclient

# Без авторизации
let proxy = newProxy("http://myproxy.example.com:8080")

# С авторизацией
let authProxy = newProxy("http://user:secret@myproxy.example.com:8080")

# SOCKS5 с резолвингом DNS на стороне прокси
let socks5Proxy = newProxy("socks5h://user:secret@myproxy.example.com:1080")

let client = newHttpClient(proxy = proxy)
```

> **Устаревшие перегрузки:** `newProxy(url, auth)` и `proc auth*(p: Proxy)` — помечены как deprecated. Передавайте авторизацию прямо в URL.

---

## Работа с MultipartData

### `newMultipartData(): MultipartData`

Создаёт пустой объект `MultipartData`.

```nim
var data = newMultipartData()
```

---

### `newMultipartData(xs: MultipartEntries): MultipartData`

Создаёт `MultipartData` и сразу заполняет его переданными полями.

```nim
var data = newMultipartData({"action": "login", "format": "json"})
```

---

### `add(p: MultipartData, name, content: string, filename = "", contentType = "", useStream = true)`

Добавляет поле в `MultipartData`.

| Параметр | Описание |
|---|---|
| `name` | Имя поля формы |
| `content` | Значение поля (или путь к файлу, если `filename` задан) |
| `filename` | Имя файла (если это файловое поле) |
| `contentType` | MIME-тип файла |
| `useStream` | `true` — файл читается потоково с диска; `false` — файл читается полностью в память |

Генерирует `ValueError`, если `name`, `filename` или `contentType` содержат символ новой строки.

```nim
var data = newMultipartData()
data.add("comment", "Hello!")
data.add("avatar", "/path/to/image.png", filename = "avatar.png",
         contentType = "image/png")
```

---

### `add(p: MultipartData, xs: MultipartEntries): MultipartData`

Добавляет список текстовых полей. Возвращает `p` (можно использовать в цепочке).

```nim
data.add({"username": "alice", "role": "admin"})
```

---

### `addFiles(p: MultipartData, xs: openArray[tuple[name, file: string]], mimeDb = newMimetypes(), useStream = true): MultipartData`

Добавляет файлы из файловой системы. MIME-тип определяется автоматически по расширению.

- Файл при `useStream = true` читается с диска потоково во время запроса (экономит память).
- При `useStream = false` — весь файл загружается в память заранее.

Генерирует `IOError`, если файл не может быть открыт.

```nim
import std/[httpclient, mimetypes]

let mimes = newMimetypes()
var data = newMultipartData()
data.addFiles({"report": "report.pdf", "logo": "logo.png"}, mimeDb = mimes)

let client = newHttpClient()
defer: client.close()
echo client.postContent("https://example.com/upload", multipart = data)
```

---

### `p[name] = content: string`

Краткая запись для добавления текстового поля.

```nim
data["username"] = "alice"
```

---

### `p[name] = (filename, contentType, content): tuple`

Краткая запись для добавления файлового поля с явным указанием имени, типа и содержимого.

```nim
data["page"] = ("index.html", "text/html", "<html>...</html>")
```

---

### `$(data: MultipartData): string`

Преобразует `MultipartData` в читабельную строку для отладки.

```nim
echo $data
```

---

## Создание клиентов

### `newHttpClient(...): HttpClient`

Создаёт синхронный HTTP-клиент.

**Сигнатура:**
```nim
proc newHttpClient*(
  userAgent = defUserAgent,
  maxRedirects = 5,
  sslContext = getDefaultSSL(),
  proxy: Proxy = nil,
  timeout = -1,
  headers = newHttpHeaders()
): HttpClient
```

| Параметр | Тип | По умолчанию | Описание |
|---|---|---|---|
| `userAgent` | `string` | `"Nim-httpclient/<version>"` | Строка User-Agent |
| `maxRedirects` | `int` | `5` | Макс. число редиректов; `0` — отключить |
| `sslContext` | `SslContext` | `CVerifyPeer` | Контекст SSL/TLS |
| `proxy` | `Proxy` | `nil` | HTTP- или SOCKS5-прокси |
| `timeout` | `int` | `-1` (нет лимита) | Таймаут в миллисекундах |
| `headers` | `HttpHeaders` | пустые | Глобальные заголовки клиента |

```nim
import std/httpclient

# Простой клиент
let client = newHttpClient()
defer: client.close()
echo client.getContent("http://example.com")

# Клиент с таймаутом 5 секунд и кастомным заголовком
import std/httpclient
let client2 = newHttpClient(
  timeout = 5000,
  headers = newHttpHeaders({"Accept": "application/json"})
)
defer: client2.close()
```

---

### `newAsyncHttpClient(...): AsyncHttpClient`

Создаёт асинхронный HTTP-клиент. Параметры аналогичны `newHttpClient`, за исключением отсутствия `timeout` (пока не реализован).

```nim
import std/[asyncdispatch, httpclient]

proc fetchPage(): Future[string] {.async.} =
  let client = newAsyncHttpClient()
  defer: client.close()
  return await client.getContent("http://example.com")

echo waitFor fetchPage()
```

> **Важно:** Один экземпляр `AsyncHttpClient` обрабатывает только один запрос одновременно. Для параллельных запросов создавайте несколько клиентов.

---

## Управление клиентом

### `close(client: HttpClient | AsyncHttpClient)`

Закрывает сетевое соединение, удерживаемое клиентом.

```nim
client.close()
```

Рекомендуется всегда закрывать клиент в блоке `try/finally` или через `defer`.

---

### `getSocket(client: HttpClient): Socket`
### `getSocket(client: AsyncHttpClient): AsyncSocket`

Возвращает сетевой сокет клиента. Полезно для получения деталей соединения.

```nim
if client.connected:
  echo client.getSocket().getLocalAddr()
  echo client.getSocket().getPeerAddr()
```

---

### Поле `onProgressChanged`

Колбэк для отслеживания прогресса загрузки. Вызывается примерно раз в секунду.

```nim
import std/[asyncdispatch, httpclient]

proc onProgress(total, progress, speed: BiggestInt) {.async.} =
  echo "Скачано ", progress, " из ", total, " байт"
  echo "Скорость: ", speed div 1024, " КБ/с"

proc main() {.async.} =
  let client = newAsyncHttpClient()
  client.onProgressChanged = onProgress
  defer: client.close()
  discard await client.getContent("https://example.com/bigfile.zip")

waitFor main()

# Отключить колбэк:
# client.onProgressChanged = nil
```

---

### Поле `headers`

`HttpHeaders` — заголовки, которые будут отправлены во всех запросах этого клиента. Можно переопределить для конкретного запроса через параметр `headers` метода `request`.

```nim
client.headers = newHttpHeaders({
  "Authorization": "Bearer mytoken",
  "Accept": "application/json"
})
```

---

### Поле `timeout`

Таймаут в миллисекундах для синхронного клиента. `-1` означает отсутствие таймаута. При превышении генерируется `TimeoutError`.

> **Примечание:** Таймаут ограничивает не всю операцию, а каждый отдельный вызов сокета. Если сервер активно передаёт данные, исключение не возникнет даже при медленном соединении.

```nim
let client = newHttpClient(timeout = 3000) # 3 секунды
```

---

## Работа с ответом сервера

### `code(response: Response | AsyncResponse): HttpCode`

Возвращает числовой HTTP-код ответа.

Генерирует `ValueError`, если статус нечитаем.

```nim
let resp = client.get("https://example.com")
echo resp.code          # 200
echo resp.code == Http200 # true
```

---

### `contentType(response: Response | AsyncResponse): string`

Возвращает значение заголовка `Content-Type`.

```nim
echo resp.contentType   # "text/html; charset=utf-8"
```

---

### `contentLength(response: Response | AsyncResponse): int`

Возвращает значение заголовка `Content-Length`. Если заголовок отсутствует — возвращает `-1`.

Генерирует `ValueError`, если значение нечисловое.

```nim
echo resp.contentLength # 42315
```

---

### `lastModified(response: Response | AsyncResponse): DateTime`

Возвращает дату последней модификации ресурса из заголовка `Last-Modified`.

Генерирует `ValueError` при ошибке парсинга.

```nim
echo resp.lastModified  # 2024-01-15T10:30:00+00:00
```

---

### `body(response: Response): string`
### `body(response: AsyncResponse): Future[string]`

Возвращает тело ответа как строку. Результат кешируется — повторное чтение не обращается к сети.

```nim
# Синхронно
let resp = client.get("https://example.com")
echo resp.body

# Асинхронно
let resp = await client.get("https://example.com")
echo await resp.body
```

---

## HTTP-запросы

> Все методы работают как синхронно (`HttpClient`), так и асинхронно (`AsyncHttpClient`).  
> Асинхронные версии возвращают `Future[...]`.

---

### `request(client, url, httpMethod, body, headers, multipart): Response | AsyncResponse`

Универсальный метод для выполнения HTTP-запроса любого типа.

```nim
proc request*(
  client: HttpClient | AsyncHttpClient,
  url: Uri | string,
  httpMethod: HttpMethod | string = HttpGet,
  body = "",
  headers: HttpHeaders = nil,
  multipart: MultipartData = nil
): Future[Response | AsyncResponse]
```

| Параметр | Описание |
|---|---|
| `url` | Адрес запроса; не должен содержать `\r` или `\n` |
| `httpMethod` | Метод: `HttpGet`, `HttpPost`, `HttpPut`, `HttpDelete`, `HttpPatch`, `HttpHead`, `HttpOptions`, `HttpTrace`, `HttpConnect` |
| `body` | Тело запроса |
| `headers` | Заголовки, действующие только для этого запроса (переопределяют `client.headers`) |
| `multipart` | Данные формы |

```nim
import std/[httpclient, json]

let client = newHttpClient()
defer: client.close()

let body = $(%*{"name": "Alice", "age": 30})
let resp = client.request(
  "https://api.example.com/users",
  httpMethod = HttpPost,
  body = body,
  headers = newHttpHeaders({"Content-Type": "application/json"})
)
echo resp.status   # "201 Created"
echo resp.body
```

---

### `head(client, url): Response | AsyncResponse`

Выполняет запрос `HEAD`. Тело ответа не возвращается.

```nim
let resp = client.head("https://example.com/file.zip")
echo resp.contentLength  # Размер файла без скачивания
```

---

### `get(client, url): Response | AsyncResponse`

Выполняет запрос `GET`. Возвращает полный ответ.

```nim
let resp = client.get("https://httpbin.org/get")
echo resp.code
echo resp.body
```

---

### `getContent(client, url): string | Future[string]`

Выполняет запрос `GET` и возвращает тело ответа в виде строки.  
Генерирует `HttpRequestError` при кодах 4xx и 5xx.

```nim
let html = client.getContent("https://example.com")
echo html
```

---

### `delete(client, url): Response | AsyncResponse`

Выполняет запрос `DELETE`.

```nim
let resp = client.delete("https://api.example.com/items/42")
echo resp.status
```

---

### `deleteContent(client, url): string | Future[string]`

Выполняет запрос `DELETE` и возвращает тело ответа.  
Генерирует `HttpRequestError` при кодах 4xx/5xx.

```nim
echo client.deleteContent("https://api.example.com/items/42")
```

---

### `post(client, url, body, multipart): Response | AsyncResponse`

Выполняет запрос `POST`.

```nim
# JSON-запрос
client.headers = newHttpHeaders({"Content-Type": "application/json"})
let resp = client.post("https://api.example.com/data", body = """{"key":"value"}""")

# Multipart-запрос
var data = newMultipartData()
data["file"] = ("report.csv", "text/csv", "a,b,c\n1,2,3")
let resp2 = client.post("https://example.com/upload", multipart = data)
```

---

### `postContent(client, url, body, multipart): string | Future[string]`

Выполняет `POST` и возвращает тело ответа.  
Генерирует `HttpRequestError` при кодах 4xx/5xx.

```nim
let result = client.postContent(
  "https://api.example.com/echo",
  body = "Hello!",
  headers = newHttpHeaders({"Content-Type": "text/plain"}) # через client.headers
)
echo result
```

---

### `put(client, url, body, multipart): Response | AsyncResponse`

Выполняет запрос `PUT`.

```nim
let resp = client.put(
  "https://api.example.com/items/1",
  body = """{"name":"Updated"}"""
)
```

---

### `putContent(client, url, body, multipart): string | Future[string]`

Выполняет `PUT` и возвращает тело ответа. Генерирует `HttpRequestError` при ошибках.

---

### `patch(client, url, body, multipart): Response | AsyncResponse`

Выполняет запрос `PATCH`.

```nim
let resp = client.patch(
  "https://api.example.com/items/1",
  body = """{"status":"active"}"""
)
```

---

### `patchContent(client, url, body, multipart): string | Future[string]`

Выполняет `PATCH` и возвращает тело ответа. Генерирует `HttpRequestError` при ошибках.

---

### `downloadFile(client: HttpClient, url, filename: string)`
### `downloadFile(client: AsyncHttpClient, url, filename): Future[void]`

Скачивает файл по URL и сохраняет на диск. Данные записываются потоково — без загрузки в память целиком.

Генерирует `HttpRequestError` при кодах 4xx/5xx.  
Генерирует `IOError`, если файл не может быть открыт для записи.

```nim
# Синхронно
let client = newHttpClient()
defer: client.close()
client.downloadFile("https://example.com/data.zip", "data.zip")
echo "Файл скачан!"

# Асинхронно
import std/asyncdispatch

proc downloadAsync() {.async.} =
  let client = newAsyncHttpClient()
  defer: client.close()
  await client.downloadFile("https://example.com/data.zip", "data.zip")

waitFor downloadAsync()
```

---

## Константы

### `defUserAgent*: string`

Значение User-Agent по умолчанию. Формат: `"Nim-httpclient/<NimVersion>"`.

```nim
echo defUserAgent  # "Nim-httpclient/2.x.x"
```

---

## Исключения

| Тип | Наследует | Когда возникает |
|---|---|---|
| `ProtocolError` | `IOError` | Сервер нарушил HTTP-протокол |
| `HttpRequestError` | `IOError` | Сервер вернул ошибку 4xx или 5xx (в `getContent`, `postContent` и т.д.) |

---

## Полные примеры

### Скачать страницу (синхронно)

```nim
import std/httpclient

let client = newHttpClient()
try:
  echo client.getContent("https://example.com")
finally:
  client.close()
```

### Отправить JSON (асинхронно)

```nim
import std/[asyncdispatch, httpclient, json]

proc sendJson() {.async.} =
  let client = newAsyncHttpClient()
  client.headers = newHttpHeaders({"Content-Type": "application/json"})
  defer: client.close()

  let body = $(%*{"message": "hello"})
  let resp = await client.post("https://httpbin.org/post", body = body)
  echo await resp.body

waitFor sendJson()
```

### Загрузка файла с прогрессом

```nim
import std/[asyncdispatch, httpclient]

proc onProgress(total, progress, speed: BiggestInt) {.async.} =
  if total > 0:
    echo progress * 100 div total, "% | ", speed div 1024, " KB/s"

proc main() {.async.} =
  let client = newAsyncHttpClient()
  client.onProgressChanged = onProgress
  defer: client.close()
  await client.downloadFile("https://example.com/bigfile.bin", "output.bin")

waitFor main()
```

### Использование прокси с SSL

```nim
import std/[net, httpclient]

let proxy = newProxy("http://proxyuser:proxypass@proxy.example.com:3128")
let ctx = newContext(verifyMode = CVerifyPeer)
let client = newHttpClient(proxy = proxy, sslContext = ctx, timeout = 10_000)
defer: client.close()
echo client.getContent("https://example.com")
```

### Получить прокси из переменных окружения

```nim
import std/[os, httpclient]

var proxyUrl = ""
try:
  if existsEnv("http_proxy"):
    proxyUrl = getEnv("http_proxy")
  elif existsEnv("https_proxy"):
    proxyUrl = getEnv("https_proxy")
except ValueError:
  echo "Ошибка при чтении переменной окружения прокси"

let client = if proxyUrl.len > 0:
  newHttpClient(proxy = newProxy(proxyUrl))
else:
  newHttpClient()
defer: client.close()
```

---

## SSL/TLS

Поддержка SSL активируется автоматически при использовании `https://`-ссылок.  
Требуется компиляция с флагом: `nim c -d:ssl yourfile.nim`.

Доступные режимы верификации сертификата:

| Режим | Описание |
|---|---|
| `CVerifyNone` | Сертификаты не проверяются |
| `CVerifyPeer` | Сертификаты проверяются (по умолчанию) |
| `CVerifyPeerUseEnvVars` | Проверяются + учитываются `SSL_CERT_FILE` и `SSL_CERT_DIR` |

```nim
import std/[net, httpclient]
let client = newHttpClient(sslContext = newContext(verifyMode = CVerifyNone))
```

---

## Перенаправления (Redirects)

Клиент автоматически следует редиректам (301, 302, 303, 307, 308).

- 301/302/303: метод меняется на `GET`, тело удаляется (кроме `GET`/`HEAD`).
- 307/308: метод и тело сохраняются.
- При смене домена удаляются заголовки `Host` и `Authorization`.

```nim
let client = newHttpClient(maxRedirects = 0) # отключить редиректы
let client2 = newHttpClient(maxRedirects = 10) # увеличить лимит
```
