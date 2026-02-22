# Справочник модуля `std/asynchttpserver` для Nim

> Высокопроизводительный асинхронный HTTP/1.1 сервер, построенный на `std/asyncdispatch` и `std/asyncnet`.
>
> ⚠️ **Предупреждение о production-среде:** Этот сервер предназначен **исключительно для локальной разработки и тестирования**. Для продакшн-развёртывания необходимо использовать обратный прокси (например, nginx или Caddy). Сервер не поддерживает TLS, не имеет расширенного управления соединениями и production-защиты.

---

## Содержание

1. [Обзор архитектуры](#обзор-архитектуры)
2. [Типы данных](#типы-данных)
   - [Request](#request)
   - [AsyncHttpServer](#asynchttpserver)
3. [Создание сервера](#создание-сервера)
   - [newAsyncHttpServer](#newasyncHttpserver)
4. [Запуск и остановка](#запуск-и-остановка)
   - [listen](#listen)
   - [serve](#serve)
   - [close](#close)
5. [Цикл обработки запросов](#цикл-обработки-запросов)
   - [shouldAcceptRequest](#shouldacceptrequest)
   - [acceptRequest](#acceptrequest)
   - [getPort](#getport)
6. [Ответ на запросы](#ответ-на-запросы)
   - [respond](#respond)
   - [sendHeaders](#sendheaders)
7. [Экспортируемые константы](#экспортируемые-константы)
   - [nimMaxDescriptorsFallback](#nimmaxdescriptorsfallback)
8. [Жизненный цикл запроса](#жизненный-цикл-запроса)
9. [Полные примеры](#полные-примеры)
   - [Минимальный сервер](#минимальный-сервер)
   - [Собственный цикл с обработкой ошибок](#собственный-цикл-с-обработкой-ошибок)
   - [Маршрутизация по методу и пути](#маршрутизация-по-методу-и-пути)
10. [Шпаргалка](#шпаргалка)

---

## Обзор архитектуры

Модель конкурентности сервера полностью опирается на асинхронный цикл событий Nim. Каждое входящее TCP-соединение принимается и немедленно передаётся `processClient`, который запускается как «свободная» async-задача (через `asyncCheck`). Это позволяет сотням соединений выполняться параллельно в одном потоке — каждое приостановлено на `await` в ожидании сетевого ввода/вывода.

```
                ┌────────────────────────┐
                │   Ваш серверный цикл   │
                │  (цикл while true)     │
                └──────────┬─────────────┘
                           │ await acceptRequest(cb)
                           ▼
                ┌────────────────────────┐
                │  processClient (async) │  ← по одному на соединение, параллельно
                │  ┌──────────────────┐  │
                │  │  processRequest  │  │  ← разбор HTTP, чтение тела
                │  │  → ваш cb(req)   │  │  ← здесь вызывается ваш обработчик
                │  └──────────────────┘  │
                │  (цикл keep-alive)     │
                └────────────────────────┘
```

Соединение поддерживается живым для HTTP/1.1 по умолчанию и закрывается после первого запроса для HTTP/1.0 — в соответствии со стандартом RFC 7230.

---

## Типы данных

### `Request`

```nim
type Request* = object
  client*:    AsyncSocket                             # Сырой сокет подключённого клиента
  reqMethod*: HttpMethod                              # GET, POST, PUT, DELETE, …
  headers*:   HttpHeaders                             # Все заголовки запроса
  protocol*:  tuple[orig: string, major, minor: int]  # например ("HTTP/1.1", 1, 1)
  url*:       Uri                                     # Разобранный URL запроса
  hostname*:  string                                  # IP-адрес клиента
  body*:      string                                  # Тело запроса (целиком в памяти)
```

**Что это:** Объект-значение, передаваемый в ваш колбэк-обработчик запросов. Содержит всё необходимое для понимания HTTP-запроса и ответа на него. Все поля публичны и доступны для чтения.

Ключевые замечания:
- `body` полностью считывается и буферизируется до вызова вашего колбэка. Сервер принудительно ограничивает размер тела значением `maxBody` байт, автоматически отвечая `413`, если лимит превышен.
- `client` даёт прямой доступ к сокету для продвинутых сценариев (апгрейд до WebSocket, потоковые ответы). В обычной работе используйте `respond`.
- `protocol.major` и `protocol.minor` позволяют различать HTTP/1.0 и HTTP/1.1 клиентов.
- `url` — это разобранный `Uri`: можно напрямую обращаться к `url.path`, `url.query`, `url.hostname` и т.д.
- `hostname` — это **IP-адрес клиента** в виде строки (не DNS-имя).

```nim
proc handler(req: Request) {.async.} =
  echo req.reqMethod   # GET, POST и т.д.
  echo req.url.path    # "/api/users"
  echo req.url.query   # "page=2&limit=10"
  echo req.hostname    # "192.168.1.42"
  echo req.body        # тело запроса (для POST/PUT)

  for key, val in req.headers:
    echo key, ": ", val
```

---

### `AsyncHttpServer`

```nim
type AsyncHttpServer* = ref object
  # socket:    AsyncSocket  (внутреннее)
  # reuseAddr: bool         (внутреннее)
  # reusePort: bool         (внутреннее)
  # maxBody:   int          (внутреннее)
  # maxFDs:    int          (внутреннее)
```

**Что это:** Ссылочный тип, представляющий запущенный HTTP-сервер. Создаётся через `newAsyncHttpServer`, привязывается к порту через `listen`, после чего запросы обрабатываются через `acceptRequest` или `serve`. Все поля внутренние — конфигурируйте сервер при создании через параметры `newAsyncHttpServer`.

---

## Создание сервера

### `newAsyncHttpServer`

```nim
proc newAsyncHttpServer*(reuseAddr = true,
                         reusePort = false,
                         maxBody   = 8_388_608): AsyncHttpServer
```

**Что делает:** Создаёт новый экземпляр сервера и настраивает параметры сокета и лимит тела запроса. **Не** привязывает порт и не начинает слушать — для этого вызовите `listen`.

**Параметры:**

- `reuseAddr` — при `true` (по умолчанию) устанавливает `SO_REUSEADDR` на сокет. Это позволяет серверу быстро перезапускаться без ожидания, пока ОС освободит сокет из состояния TIME_WAIT. Почти всегда желательно.
- `reusePort` — при `true` устанавливает `SO_REUSEPORT`, позволяя нескольким процессам сервера привязываться к одному порту (полезно для многопроцессной балансировки нагрузки). По умолчанию отключено. Не поддерживается на NuttX.
- `maxBody` — максимальное количество байт, которое сервер прочитает из тела запроса. По умолчанию 8 МиБ (8 388 608 байт). Запросы с `Content-Length`, превышающим этот лимит, автоматически получают ответ `413 Request Entity Too Large`.

```nim
import std/asynchttpserver

# По умолчанию: лимит тела 8 МиБ, reuseAddr включён
var server = newAsyncHttpServer()

# API-сервер с более строгим лимитом тела (1 МиБ)
var apiServer = newAsyncHttpServer(maxBody = 1_048_576)

# Многопроцессная настройка (сокет разделяется между воркерами)
var workerServer = newAsyncHttpServer(reusePort = true)
```

---

## Запуск и остановка

### `listen`

```nim
proc listen*(server:  AsyncHttpServer;
             port:    Port;
             address  = "";
             domain   = AF_INET)
```

**Что делает:** Привязывает сокет сервера к указанному `port` и `address`, затем вызывает на нём `listen()` — заставляя ОС начать ставить входящие соединения в очередь. Эта процедура **синхронная** (не требует `await`). После возврата `listen` сервер готов принимать соединения; вызовите `acceptRequest` или `serve`, чтобы начать их обработку.

**Параметры:**

- `port` — TCP-порт для привязки. Используйте `Port(0)`, чтобы ОС сама выбрала свободный порт (удобно в тестах); получить назначенный порт можно через `getPort`.
- `address` — локальный IP-адрес для привязки. Пустая строка (по умолчанию) привязывает ко всем интерфейсам (`0.0.0.0` для IPv4). Используйте `"127.0.0.1"` для приёма только локальных соединений.
- `domain` — семейство адресов сокета. `AF_INET` (по умолчанию) для IPv4, `AF_INET6` для IPv6.

```nim
import std/asynchttpserver
from std/nativesockets import AF_INET6

var server = newAsyncHttpServer()

# Все интерфейсы, порт 8080
server.listen(Port(8080))

# Только localhost
server.listen(Port(8080), address = "127.0.0.1")

# IPv6
server.listen(Port(8080), domain = AF_INET6)

# Порт выбирает ОС (для тестов)
server.listen(Port(0))
echo "Слушаем на порту ", server.getPort.uint16
```

---

### `serve`

```nim
proc serve*(server:   AsyncHttpServer,
            port:     Port,
            callback: proc(request: Request): Future[void] {.closure, gcsafe.},
            address   = "";
            assumedDescriptorsPerRequest = -1;
            domain    = AF_INET) {.async.}
```

**Что делает:** Удобная корутина «всё в одном», которая вызывает `listen` и затем зацикливается навсегда, вызывая `acceptRequest` для каждого входящего соединения. Предназначена для быстрых прототипов, где собственный цикл не нужен.

**Важно:** Документация самого модуля рекомендует предпочитать `acceptRequest` с собственным циклом для production-качественного кода, поскольку `serve` не даёт возможности обрабатывать ошибки `acceptRequest` или логировать события соединений.

`assumedDescriptorsPerRequest` управляет защитой по лимиту FD:
- `-1` (по умолчанию): **отключена** — защита не проверяется, соединения принимаются всегда.
- `≥ 0`: предполагаемое количество файловых дескрипторов, потребляемых одним запросом. Сервер откажется принять новое соединение, если `activeDescriptors() + assumedDescriptorsPerRequest ≥ maxFDs`, вместо этого вызывая `poll()`. Используйте `5` (значение из примеров) как отправную точку.

```nim
import std/[asynchttpserver, asyncdispatch]

proc handler(req: Request) {.async.} =
  await req.respond(Http200, "Привет!")

var server = newAsyncHttpServer()
waitFor server.serve(Port(8080), handler)
```

---

### `close`

```nim
proc close*(server: AsyncHttpServer)
```

**Что делает:** Закрывает слушающий сокет сервера. Это **синхронный** вызов. После `close` новые соединения приниматься не будут. Соединения, уже находящиеся в обработке (`processClient`), **не** прерываются — они завершатся штатно.

**Когда использовать:** Для корректного завершения работы в ответ на сигнал, при завершении теста или в любой ситуации, когда нужно прекратить приём новых клиентов.

```nim
import std/[asynchttpserver, asyncdispatch, os]

var server = newAsyncHttpServer()
server.listen(Port(8080))

setControlCHook(proc() {.noconv.} =
  server.close()
  quit(0)
)
```

---

## Цикл обработки запросов

Рекомендуемый паттерн запуска сервера — написать явный цикл `while true`, который вызывает `shouldAcceptRequest` перед каждым `acceptRequest`. Это даёт вам контроль над обработкой ошибок, логированием и противодавлением.

### `shouldAcceptRequest`

```nim
proc shouldAcceptRequest*(server: AsyncHttpServer;
                          assumedDescriptorsPerRequest = 5): bool {.inline.}
```

**Что делает:** Возвращает `true`, если прямо сейчас безопасно принять ещё одно соединение. Внутри проверяет: `activeDescriptors() + assumedDescriptorsPerRequest < server.maxFDs`. Когда процессу не хватает файловых дескрипторов, принятие нового соединения может привести к тому, что ОС откажется открывать дополнительные сокеты или файлы — эта защита предотвращает такую ситуацию.

- `assumedDescriptorsPerRequest` — сколько файловых дескрипторов предположительно потребляет один запрос (сокет + возможные лог-файлы, временные файлы и т.д.). Значение по умолчанию `5` — консервативное и подходит для большинства приложений.
- Возвращает `true` безусловно при `assumedDescriptorsPerRequest < 0`.

**Когда использовать:** Всегда вызывайте перед `acceptRequest` в собственном цикле. Если возвращает `false`, подождите немного (например, `await sleepAsync(500)`), чтобы открытые соединения успели закрыться.

```nim
while true:
  if server.shouldAcceptRequest():
    await server.acceptRequest(handler)
  else:
    await sleepAsync(500)  # отступаем назад, FD на исходе
```

---

### `acceptRequest`

```nim
proc acceptRequest*(server:   AsyncHttpServer,
                    callback: proc(request: Request): Future[void] {.closure, gcsafe.})
                   {.async.}
```

**Что делает:** Принимает одно входящее TCP-соединение и немедленно запускает `processClient` как фоновую async-задачу (через `asyncCheck`). `processClient` занимается разбором HTTP, чтением тела, вызовом вашего `callback` и управлением keep-alive. Сам `acceptRequest` возвращается сразу после передачи соединения — он **не** ждёт обработки запроса.

**Сигнатура колбэка:** Ваш обработчик должен быть:
```nim
proc myHandler(req: Request): Future[void] {.async.}
```
Он не должен выбрасывать исключения напрямую — оборачивайте любые ошибки в `try/except` и вызывайте `req.respond` с соответствующим кодом ошибки.

**Когда использовать:** В собственном цикле `while true`, в паре с `shouldAcceptRequest`.

```nim
import std/[asynchttpserver, asyncdispatch]

proc handler(req: Request) {.async.} =
  try:
    await req.respond(Http200, "OK")
  except Exception as e:
    echo "Ошибка обработчика: ", e.msg

proc main() {.async.} =
  var server = newAsyncHttpServer()
  server.listen(Port(8080))
  echo "Слушаем на порту 8080"

  while true:
    if server.shouldAcceptRequest():
      await server.acceptRequest(handler)
    else:
      await sleepAsync(500)

waitFor main()
```

---

### `getPort`

```nim
proc getPort*(self: AsyncHttpServer): Port
```

**Что делает:** Возвращает порт, к которому в данный момент привязан сервер. Особенно полезно, если привязка производилась к `Port(0)` и нужно узнать, какой порт в итоге назначила ОС.

Вызывать нужно **после** `listen` — до этого сокет ещё не привязан.

```nim
import std/asynchttpserver

var server = newAsyncHttpServer()
server.listen(Port(0))

let port = server.getPort
echo "Сервер запущен на порту ", port.uint16   # например, "Сервер запущен на порту 54321"
```

---

## Ответ на запросы

### `respond`

```nim
proc respond*(req:     Request,
              code:    HttpCode,
              content: string,
              headers: HttpHeaders = nil): Future[void]
```

**Что делает:** Отправляет полный HTTP-ответ клиенту. Собирает строку статуса, ваши пользовательские заголовки, автоматический заголовок `Content-Length` (если вы его не указали) и тело, затем отправляет всё одной операцией записи. Возвращает `Future[void]`, который завершается когда данные переданы ОС.

**Важно:** `respond` **не** закрывает клиентский сокет. Сервер сам управляет временем жизни соединения (keep-alive vs. close) на основе версии HTTP и заголовков `Connection`. Никогда не вызывайте `req.client.close()` из своего колбэка, если только не делаете апгрейд WebSocket.

**Параметры:**

- `code` — `HttpCode` из `std/httpcore`. Распространённые значения: `Http200`, `Http201`, `Http400`, `Http404`, `Http500`. Можно также использовать числовые литералы: `HttpCode(418)`.
- `content` — тело ответа в виде строки. Может быть пустым для ответов типа `204 No Content`.
- `headers` — необязательные `HttpHeaders` с заголовками ответа. Если `nil`, отправляется только `Content-Length`. Если передать заголовки без `Content-Length`, сервер добавит его автоматически.

```nim
proc handler(req: Request) {.async.} =
  case req.url.path
  of "/":
    await req.respond(Http200, "Добро пожаловать!")

  of "/json":
    let headers = newHttpHeaders([("Content-Type", "application/json")])
    await req.respond(Http200, """{"ok": true}""", headers)

  of "/created":
    let headers = newHttpHeaders([
      ("Content-Type", "text/plain"),
      ("Location", "/resource/42")
    ])
    await req.respond(Http201, "Создано", headers)

  else:
    await req.respond(Http404, "Не найдено")
```

---

### `sendHeaders`

```nim
proc sendHeaders*(req: Request, headers: HttpHeaders): Future[void]
```

**Что делает:** Отправляет клиенту **только** указанные `HttpHeaders`, без строки статуса и тела. Это низкоуровневый строительный блок для сценариев, требующих точного контроля над отправляемыми данными — например, потоковые ответы или апгрейд протокола (WebSocket handshake).

**Когда использовать:** Редко — большинство обработчиков должны использовать `respond`. Применяйте `sendHeaders`, когда хотите вручную начать ответ и затем напрямую писать тело в `req.client`.

```nim
proc streamingHandler(req: Request) {.async.} =
  # Строку статуса отправляем вручную
  await req.client.send("HTTP/1.1 200 OK\r\n")

  # Отправляем заголовки
  let h = newHttpHeaders([
    ("Content-Type", "text/event-stream"),
    ("Cache-Control", "no-cache"),
    ("Transfer-Encoding", "chunked"),
  ])
  await req.sendHeaders(h)
  await req.client.send("\r\n")  # пустая строка завершает заголовки

  # Потоковая передача данных напрямую
  for i in 0..9:
    await req.client.send("data: событие " & $i & "\n\n")
    await sleepAsync(500)
```

---

## Экспортируемые константы

### `nimMaxDescriptorsFallback`

```nim
const nimMaxDescriptorsFallback* {.intdefine.} = 16_000
```

**Что это:** Запасное значение для максимального числа открытых файловых дескрипторов, когда системный вызов `maxDescriptors()` недоступен. По умолчанию 16 000.

Можно переопределить при компиляции:
```sh
nim c -d:nimMaxDescriptorsFallback=8000 myserver.nim
```

Эта константа питает защитный механизм `shouldAcceptRequest`. Если лимит ОС ниже этого запасного значения (например, стандартный `ulimit -n` равный 1024 на многих Linux-системах), возможно, стоит снизить эту константу или увеличить `ulimit -n` в продакшне.

---

## Жизненный цикл запроса

Понимание полного жизненного цикла помогает писать корректные обработчики:

```
Клиент подключается
    │
    ▼
listen() ставит соединение в очередь
    │
    ▼
acceptRequest() принимает TCP-сокет
    │
    ▼
processClient() запускает цикл
    │
    ▼
processRequest() выполняется:
  1. Читает строку запроса (GET /path HTTP/1.1)   → 413, если > 8 КБ
  2. Читает заголовки по одному                   → 413, если > 8 КБ/заголовок
                                                  → 400, если заголовков слишком много
  3. Обрабатывает Expect: 100-continue (только для POST)
  4. Читает тело:
       Content-Length есть → recv(N) байт          → 413, если N > maxBody
       Transfer-Encoding: chunked                  → собирает чанки
       POST без того и другого                     → 411 Length Required
  5. Вызывает ваш callback(request)
  6. Проверяет Connection: close/upgrade
  7. HTTP/1.1 → удерживает соединение живым
     HTTP/1.0 → закрывает соединение
    │
    ▼
Возврат в цикл или закрытие сокета
```

**Что нужно делать в вашем колбэке:**
- Всегда вызывайте `await req.respond(...)` (или пишите в `req.client` напрямую). Если вы вернётесь не ответив, клиент получит частичный или пустой ответ.
- Обрабатывайте все исключения внутри колбэка. Необработанное исключение из вашего колбэка будет молча проглочено `asyncCheck` — используйте `try/except` для логирования и ответа с кодом 500.

---

## Полные примеры

### Минимальный сервер

```nim
import std/[asynchttpserver, asyncdispatch]

proc handler(req: Request) {.async.} =
  let headers = newHttpHeaders([("Content-Type", "text/plain; charset=utf-8")])
  await req.respond(Http200, "Привет, Мир!", headers)

proc main() {.async.} =
  var server = newAsyncHttpServer()
  server.listen(Port(8080))
  echo "Слушаем на http://localhost:8080"

  while true:
    if server.shouldAcceptRequest():
      await server.acceptRequest(handler)
    else:
      await sleepAsync(500)

waitFor main()
```

---

### Собственный цикл с обработкой ошибок

```nim
import std/[asynchttpserver, asyncdispatch, logging]

addHandler(newConsoleLogger())

proc handler(req: Request) {.async.} =
  try:
    info req.reqMethod, " ", req.url.path, " от ", req.hostname
    await req.respond(Http200, "OK")
  except Exception as e:
    error "Исключение в обработчике: ", e.msg
    try:
      await req.respond(Http500, "Internal Server Error")
    except: discard

proc main() {.async.} =
  var server = newAsyncHttpServer()
  server.listen(Port(8080))
  info "Сервер запущен на порту 8080"

  while true:
    if server.shouldAcceptRequest():
      try:
        await server.acceptRequest(handler)
      except Exception as e:
        error "acceptRequest завершился с ошибкой: ", e.msg
        await sleepAsync(100)  # краткая пауза перед повторной попыткой
    else:
      warn "Лимит FD близок, пауза..."
      await sleepAsync(500)

waitFor main()
```

---

### Маршрутизация по методу и пути

```nim
import std/[asynchttpserver, asyncdispatch, json, strutils]

type User = object
  id:   int
  name: string

var users = @[User(id: 1, name: "Алиса"), User(id: 2, name: "Боб")]

proc handler(req: Request) {.async.} =
  let path = req.url.path

  if path == "/users" and req.reqMethod == HttpGet:
    let body = $ %* users.mapIt(%*{"id": it.id, "name": it.name})
    let headers = newHttpHeaders([("Content-Type", "application/json")])
    await req.respond(Http200, body, headers)

  elif path.startsWith("/users/") and req.reqMethod == HttpGet:
    try:
      let id = path[7..^1].parseInt
      for u in users:
        if u.id == id:
          let headers = newHttpHeaders([("Content-Type", "application/json")])
          await req.respond(Http200, $ %*{"id": u.id, "name": u.name}, headers)
          return
      await req.respond(Http404, "Пользователь не найден")
    except ValueError:
      await req.respond(Http400, "Неверный ID пользователя")

  elif path == "/users" and req.reqMethod == HttpPost:
    try:
      let data = parseJson(req.body)
      let newUser = User(id: users.len + 1, name: data["name"].getStr)
      users.add(newUser)
      let headers = newHttpHeaders([
        ("Content-Type", "application/json"),
        ("Location", "/users/" & $newUser.id),
      ])
      await req.respond(Http201, $ %*{"id": newUser.id}, headers)
    except JsonParsingError:
      await req.respond(Http400, "Неверный JSON")

  else:
    await req.respond(Http404, "Не найдено")

proc main() {.async.} =
  var server = newAsyncHttpServer()
  server.listen(Port(8080))
  while true:
    if server.shouldAcceptRequest():
      await server.acceptRequest(handler)
    else:
      await sleepAsync(500)

waitFor main()
```

---

## Шпаргалка

| Задача | API |
|---|---|
| Создать сервер | `newAsyncHttpServer(reuseAddr, reusePort, maxBody)` |
| Привязать к порту | `server.listen(Port(8080))` |
| Порт выбирает ОС | `server.listen(Port(0))` |
| Узнать привязанный порт | `server.getPort.uint16` |
| Проверить запас FD | `server.shouldAcceptRequest()` |
| Принять одно соединение | `await server.acceptRequest(handler)` |
| Запуск «всё в одном» | `await server.serve(Port(8080), handler)` |
| Остановить приём | `server.close()` |
| Ответить с телом | `await req.respond(Http200, "тело", headers)` |
| Ответить без тела | `await req.respond(Http204, "")` |
| Отправить только заголовки | `await req.sendHeaders(headers)` |
| Путь запроса | `req.url.path` |
| Строка запроса (query) | `req.url.query` |
| Тело запроса | `req.body` |
| HTTP-метод | `req.reqMethod` (HttpGet, HttpPost, …) |
| IP-адрес клиента | `req.hostname` |
| Прочитать заголовок | `req.headers["Content-Type"]` |
| Безопасное чтение заголовка | `req.headers.getOrDefault("X-Key", "")` |
| Версия HTTP | `req.protocol.major`, `req.protocol.minor` |
| Изменить лимит тела | `newAsyncHttpServer(maxBody = 1_048_576)` |
| Переопределить лимит FD | компиляция с `-d:nimMaxDescriptorsFallback=N` |
