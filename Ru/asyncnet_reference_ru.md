# Модуль `asyncnet` Nim — Полный справочник

> **Модуль:** `std/asyncnet`  
> **Назначение:** Высокоуровневый асинхронный API для работы с сокетами, построенный поверх `asyncdispatch`. Обеспечивает буферизованные, неблокирующие операции с TCP/UDP/Unix-сокетами и опциональную поддержку SSL.

---

## Архитектура асинхронного ввода-вывода

Асинхронный ввод-вывод в Nim построен послойно (сверху вниз):

```
asyncnet        ← этот модуль (высокоуровневые буферизованные сокеты)
async/await     ← трансформация корутин
asyncdispatch   ← цикл событий, Future, IOCP/epoll/kqueue
selectors       ← абстракция над select/epoll/kqueue ОС
```

Все процедуры работают в **одном потоке** и являются **полностью неблокирующими**. Функции с `async` возвращают `Future[T]` и могут использоваться с `await`.

### SSL

Для включения SSL компилируйте с флагом `-d:ssl`. Создайте `SslContext` через `net.newContext`, затем вызовите `wrapSocket` для вашего сокета.

---

## Содержание

1. [Типы](#типы)
2. [Создание сокета](#создание-сокета)
3. [Установка соединения](#установка-соединения)
4. [Отправка данных](#отправка-данных)
5. [Получение данных](#получение-данных)
6. [Серверные операции](#серверные-операции)
7. [Параметры и информация о сокете](#параметры-и-информация-о-сокете)
8. [SSL-операции](#ssl-операции)
9. [Unix Domain Sockets](#unix-domain-sockets)
10. [Закрытие сокета](#закрытие-сокета)
11. [Полные примеры](#полные-примеры)

---

## Типы

### `AsyncSocket`

```nim
type AsyncSocket* = ref AsyncSocketDesc
```

Основной тип, представляющий асинхронный сокет. Является ссылочным типом. Внутренние поля:

| Поле | Описание |
|------|----------|
| `fd` | Дескриптор сокета ОС (`SocketHandle`) |
| `closed` | Флаг закрытия сокета |
| `isBuffered` | Включена ли буферизация чтения |
| `buffer` | Внутренний буфер чтения (размер `BufferSize`) |
| `currPos` | Текущая позиция чтения в буфере |
| `bufLen` | Количество действительных байт в буфере |
| `isSsl` | Активен ли SSL |
| `domain` | Семейство адресов (`AF_INET`, `AF_INET6`) |
| `sockType` | Тип сокета (`SOCK_STREAM`, `SOCK_DGRAM`) |
| `protocol` | Протокол (`IPPROTO_TCP`, `IPPROTO_UDP`) |

> **Поля только для SSL** (доступны при компиляции с `-d:ssl`):  
> `sslHandle`, `sslContext`, `bioIn`, `bioOut`, `sslNoShutdown`

---

## Создание сокета

### `newAsyncSocket` (из AsyncFD)

```nim
proc newAsyncSocket*(fd: AsyncFD,
                     domain: Domain = AF_INET,
                     sockType: SockType = SOCK_STREAM,
                     protocol: Protocol = IPPROTO_TCP,
                     buffered = true,
                     inheritable = defined(nimInheritHandles)): owned(AsyncSocket)
```

Создаёт `AsyncSocket` на основе **существующего** файлового дескриптора. Неблокирующий режим включается автоматически.

> **Важно:** Эта функция **не регистрирует** `fd` в глобальном асинхронном диспетчере. Если дескриптор создан через `newAsyncNativeSocket`, регистрация уже выполнена.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `fd` | Существующий `AsyncFD` (не должен быть `osInvalidSocket`) |
| `domain` | Семейство адресов. По умолчанию: `AF_INET` |
| `sockType` | Тип сокета. По умолчанию: `SOCK_STREAM` |
| `protocol` | Протокол. По умолчанию: `IPPROTO_TCP` |
| `buffered` | Включить буферизацию чтения. По умолчанию: `true` |
| `inheritable` | Разрешить наследование дочерними процессами. По умолчанию: зависит от платформы |

**Пример:**
```nim
import std/[asyncnet, asyncdispatch, nativesockets]

let rawFd = createAsyncNativeSocket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
let sock = newAsyncSocket(rawFd, AF_INET, SOCK_STREAM, IPPROTO_TCP)
```

---

### `newAsyncSocket` (создаёт новый дескриптор)

```nim
proc newAsyncSocket*(domain: Domain = AF_INET,
                     sockType: SockType = SOCK_STREAM,
                     protocol: Protocol = IPPROTO_TCP,
                     buffered = true,
                     inheritable = defined(nimInheritHandles)): owned(AsyncSocket)
```

Создаёт новый асинхронный сокет **с новым файловым дескриптором**. Наиболее распространённый способ создания сокета.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `domain` | Семейство адресов. По умолчанию: `AF_INET` (IPv4) |
| `sockType` | Тип сокета. По умолчанию: `SOCK_STREAM` (TCP) |
| `protocol` | Протокол. По умолчанию: `IPPROTO_TCP` |
| `buffered` | Включить буферизацию. По умолчанию: `true` |
| `inheritable` | Наследуемость дескриптора. По умолчанию: зависит от платформы |

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

# TCP/IPv4 сокет по умолчанию:
let sock = newAsyncSocket()

# UDP/IPv6 сокет:
let udpSock = newAsyncSocket(AF_INET6, SOCK_DGRAM, IPPROTO_UDP)

# Небуферизованный сокет:
let rawSock = newAsyncSocket(buffered = false)
```

---

### `newAsyncSocket` (перегрузка с cint)

```nim
proc newAsyncSocket*(domain, sockType, protocol: cint,
                     buffered = true,
                     inheritable = defined(nimInheritHandles)): owned(AsyncSocket)
```

То же самое, но принимает «сырые» значения `cint`. Полезно при взаимодействии с внешними C-библиотеками, использующими целочисленные константы.

**Пример:**
```nim
import std/asyncnet

# Использование целочисленных констант C:
let sock = newAsyncSocket(2.cint, 1.cint, 6.cint)
# 2 = AF_INET, 1 = SOCK_STREAM, 6 = IPPROTO_TCP
```

---

## Установка соединения

### `dial`

```nim
proc dial*(address: string, port: Port,
           protocol = IPPROTO_TCP,
           buffered = true): owned(Future[AsyncSocket]) {.async.}
```

Устанавливает соединение с `address:port` и возвращает готовый к использованию `AsyncSocket`. Автоматически перебирает все IP-адреса, возвращённые DNS-резолвером, обеспечивая прозрачную работу с IPv4 и IPv6.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `address` | Имя хоста или IP-адрес |
| `port` | Порт назначения |
| `protocol` | Транспортный протокол. По умолчанию: `IPPROTO_TCP` |
| `buffered` | Включить буферизацию. По умолчанию: `true` |

**Возвращает:** `Future[AsyncSocket]` — выполняется при успешном соединении.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  # Работает и с IPv4, и с IPv6 автоматически:
  let sock = await dial("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  let response = await sock.recv(4096)
  echo response
  sock.close()

waitFor main()
```

---

### `connect`

```nim
proc connect*(socket: AsyncSocket, address: string, port: Port) {.async.}
```

Подключает **уже созданный** сокет к серверу по адресу `address:port`. При активном SSL немедленно выполняет SSL-рукопожатие после TCP-соединения.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Неподключённый `AsyncSocket` |
| `address` | Имя хоста или IP-адрес сервера |
| `port` | Порт сервера |

**Возвращает:** `Future[void]` — выполняется при успешном соединении.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("httpbin.org", Port(80))
  echo "Подключено!"
  
  await sock.send("GET /get HTTP/1.0\r\nHost: httpbin.org\r\n\r\n")
  let data = await sock.recv(2048)
  echo data
  sock.close()

waitFor main()
```

---

### `connectUnix`

```nim
proc connectUnix*(socket: AsyncSocket, path: string): owned(Future[void])
```

Подключается к Unix domain socket по пути `path`. **Только POSIX** (Linux, macOS, BSD).

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Неподключённый `AsyncSocket` |
| `path` | Путь в файловой системе до Unix-сокета |

**Возвращает:** `Future[void]` — выполняется при установке соединения.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket(AF_UNIX, SOCK_STREAM, 0)
  await sock.connectUnix("/var/run/myapp.sock")
  await sock.send("hello\n")
  let reply = await sock.recvLine()
  echo "Ответ: ", reply
  sock.close()

waitFor main()
```

---

## Отправка данных

### `send` (строка)

```nim
proc send*(socket: AsyncSocket, data: string,
           flags = {SocketFlag.SafeDisconn}) {.async.}
```

Отправляет строку через сокет. Future завершается только после отправки **всех** данных. Прозрачно поддерживает SSL.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Подключённый `AsyncSocket` |
| `data` | Строка данных для отправки |
| `flags` | Флаги сокета. По умолчанию: `{SafeDisconn}` |

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  
  # Отправка HTTP-запроса:
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  echo "Запрос отправлен"
  sock.close()

waitFor main()
```

---

### `send` (указатель на буфер)

```nim
proc send*(socket: AsyncSocket, buf: pointer, size: int,
           flags = {SocketFlag.SafeDisconn}) {.async.}
```

Отправляет `size` байт из «сырого» буфера памяти. Полезно для сценариев с нулевым копированием или при взаимодействии с C-структурами.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Подключённый `AsyncSocket` |
| `buf` | Указатель на буфер данных |
| `size` | Количество байт для отправки |
| `flags` | Флаги сокета. По умолчанию: `{SafeDisconn}` |

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("127.0.0.1", Port(9000))
  
  var message = "бинарные данные"
  await sock.send(addr message[0], message.len)
  sock.close()

waitFor main()
```

---

### `sendTo`

```nim
proc sendTo*(socket: AsyncSocket, address: string, port: Port, data: string,
             flags = {SocketFlag.SafeDisconn}): owned(Future[void]) {.async.}
```

Отправляет датаграмму на конкретный `address:port`. Предназначен для **UDP-сокетов** — нельзя использовать с TCP. Перебирает все IP-адреса, разрешённые для `address`.

> Доступно с Nim 1.3.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | UDP-сокет (`AsyncSocket`, не TCP) |
| `address` | Имя хоста или IP назначения |
| `port` | Порт назначения |
| `data` | Данные для отправки |
| `flags` | Флаги сокета. По умолчанию: `{SafeDisconn}` |

**Исключения:** `IOError` если адрес не разрешён; `OSError` при ошибке системы.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var udpSock = newAsyncSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
  
  await udpSock.sendTo("127.0.0.1", Port(9999), "ping")
  echo "Датаграмма отправлена"
  udpSock.close()

waitFor main()
```

---

## Получение данных

### `recv`

```nim
proc recv*(socket: AsyncSocket, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[string]) {.async.}
```

Читает **до** `size` байт из сокета и возвращает их в виде строки.

- **Буферизованный режим:** пытается набрать `size` байт блоками `BufferSize`.
- **Небуферизованный режим:** возвращает то, что предоставляет ОС за один вызов.
- Возвращает `""` при разрыве соединения и отсутствии данных.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Подключённый `AsyncSocket` |
| `size` | Максимальное количество байт для чтения |
| `flags` | Флаги сокета. По умолчанию: `{SafeDisconn}` |

**Возвращает:** `Future[string]` — полученные данные, возможно короче `size`.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  
  # Читаем до 4096 байт:
  let data = await sock.recv(4096)
  if data.len == 0:
    echo "Соединение закрыто"
  else:
    echo "Получено ", data.len, " байт"
    echo data[0..min(200, data.len-1)]
  
  sock.close()

waitFor main()
```

---

### `recvInto`

```nim
proc recvInto*(socket: AsyncSocket, buf: pointer, size: int,
               flags = {SocketFlag.SafeDisconn}): owned(Future[int]) {.async.}
```

Читает **до** `size` байт из сокета напрямую в заранее выделенный буфер `buf`. Возвращает количество реально прочитанных байт. Подходит для протоколов с фиксированными буферами или нулевым копированием.

- **Буферизованный режим:** читает блоками `BufferSize` до заполнения `size` байт.
- **Небуферизованный режим:** одно системное чтение.
- Возвращает `0` при разрыве соединения.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Подключённый `AsyncSocket` |
| `buf` | Указатель на буфер получения |
| `size` | Размер буфера / максимум байт для чтения |
| `flags` | Флаги сокета. По умолчанию: `{SafeDisconn}` |

**Возвращает:** `Future[int]` — количество прочитанных байт.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  
  var buf: array[1024, char]
  let bytesRead = await sock.recvInto(addr buf[0], buf.len)
  echo "Прочитано ", bytesRead, " байт"
  echo cast[string](buf[0..<bytesRead])
  
  sock.close()

waitFor main()
```

---

### `recvLine`

```nim
proc recvLine*(socket: AsyncSocket,
               flags = {SocketFlag.SafeDisconn},
               maxLength = MaxLineLength): owned(Future[string]) {.async.}
```

Читает одну полную строку (заканчивающуюся на `\r\L` или `\L`) из сокета.

- Терминатор `\r\L` **не включается** в результат.
- Если получено только `\r\L` (пустая строка), результат **равен** `"\r\L"`.
- Возвращает `""` при разрыве соединения (данные незаконченной строки теряются).
- `maxLength` защищает от DoS-атак со слишком длинными строками.

> **Предупреждение:** Флаг `Peek` ещё не реализован.  
> **Предупреждение:** На небуферизованных сокетах предполагаются окончания строк `\r\L`.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Подключённый буферизованный `AsyncSocket` |
| `flags` | Флаги сокета. По умолчанию: `{SafeDisconn}` |
| `maxLength` | Максимальная длина строки (`MaxLineLength` по умолчанию) |

**Возвращает:** `Future[string]` — одна строка без терминатора.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("127.0.0.1", Port(8080))
  
  # Читаем строки в цикле:
  while true:
    let line = await sock.recvLine()
    if line == "":
      echo "Соединение закрыто"
      break
    echo "> ", line
  
  sock.close()

waitFor main()
```

---

### `recvLineInto`

```nim
proc recvLineInto*(socket: AsyncSocket, resString: FutureVar[string],
                   flags = {SocketFlag.SafeDisconn},
                   maxLength = MaxLineLength) {.async.}
```

Версия `recvLine` с минимальными аллокациями. Записывает строку в заранее созданный `FutureVar[string]`, избегая выделения памяти на каждый вызов. Идеально для высоконагруженных серверов.

> **Предупреждение:** Флаг `Peek` ещё не реализован.  
> **Предупреждение:** На небуферизованных сокетах предполагаются окончания строк `\r\L`.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Подключённый `AsyncSocket` |
| `resString` | Предварительно созданный `FutureVar[string]` для записи |
| `flags` | Флаги сокета. По умолчанию: `{SafeDisconn}` |
| `maxLength` | Максимальная длина строки |

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc processClient(client: AsyncSocket) {.async.} =
  # Переиспользуем один FutureVar для всех чтений:
  var lineVar = newFutureVar[string]("processClient")
  while true:
    lineVar.mget() = ""
    await client.recvLineInto(lineVar)
    let line = lineVar.mget()
    if line == "": break
    echo "Получено: ", line

waitFor processClient(newAsyncSocket())
```

---

### `recvFrom` (с FutureVar)

```nim
proc recvFrom*(socket: AsyncSocket, data: FutureVar[string], size: int,
               address: FutureVar[string], port: FutureVar[Port],
               flags = {SocketFlag.SafeDisconn}): owned(Future[int]) {.async.}
```

Получает одну UDP-датаграмму в заранее подготовленные `FutureVar`-буферы. Возвращает количество прочитанных байт. Адрес и порт отправителя записываются в `address` и `port`.

> Доступно с Nim 1.3.

**Требования:**
- `data` должен быть предварительно инициализирован до длины `size`.
- `address` должен быть предварительно инициализирован до длины 46 (максимальная длина строки IPv6).

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | UDP `AsyncSocket` |
| `data` | Предварительно инициализированный `FutureVar[string]` для полезной нагрузки |
| `size` | Ожидаемый размер датаграммы |
| `address` | `FutureVar[string]` длиной 46 для IP-адреса отправителя |
| `port` | `FutureVar[Port]` для порта отправителя |
| `flags` | Флаги сокета |

**Возвращает:** `Future[int]` — реально принятые байты.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var udpSock = newAsyncSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
  udpSock.bindAddr(Port(9999))
  
  var data = newFutureVar[string]()
  var address = newFutureVar[string]()
  var port = newFutureVar[Port]()
  
  data.mget().setLen(1024)
  address.mget().setLen(46)
  
  let bytesRead = await udpSock.recvFrom(data, 1024, address, port)
  echo "Получено ", bytesRead, " байт от ", address.mget(), ":", port.mget()
  echo "Данные: ", data.mget()
  
  udpSock.close()

waitFor main()
```

---

### `recvFrom` (упрощённая перегрузка с кортежем)

```nim
proc recvFrom*(socket: AsyncSocket, size: int,
               flags = {SocketFlag.SafeDisconn}):
              owned(Future[tuple[data: string, address: string, port: Port]]) {.async.}
```

Более простая перегрузка `recvFrom`, выполняющая внутренние аллокации и возвращающая именованный кортеж. Удобнее, но немного менее эффективна по сравнению с версией `FutureVar`.

> Доступно с Nim 1.3.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | UDP `AsyncSocket` |
| `size` | Максимальный размер принимаемой датаграммы |
| `flags` | Флаги сокета |

**Возвращает:** `Future[tuple[data, address, port]]`

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var udpSock = newAsyncSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
  udpSock.bindAddr(Port(9999))
  
  # Простой цикл приёма:
  while true:
    let (data, address, port) = await udpSock.recvFrom(1024)
    echo "От ", address, ":", port, " -> ", data
  
  udpSock.close()

waitFor main()
```

---

## Серверные операции

### `bindAddr`

```nim
proc bindAddr*(socket: AsyncSocket, port = Port(0), address = "") {.tags: [ReadIOEffect].}
```

Привязывает сокет к локальному `address:port`. Если `address = ""`, привязывается ко всем интерфейсам (`INADDR_ANY` / `::` для IPv6).

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Непривязанный `AsyncSocket` |
| `port` | Локальный порт. `Port(0)` = ОС выбирает свободный порт |
| `address` | Локальный адрес. `""` = все интерфейсы |

**Исключения:** `OSError` при ошибке; `ValueError` при неизвестном семействе адресов без указания адреса.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

var server = newAsyncSocket()
server.setSockOpt(OptReuseAddr, true)

# Привязка ко всем интерфейсам на порту 8080:
server.bindAddr(Port(8080))

# Привязка к конкретному интерфейсу:
server.bindAddr(Port(8080), "127.0.0.1")

# Автоматический выбор порта:
server.bindAddr(Port(0))
let (_, assignedPort) = server.getLocalAddr()
echo "Слушаем порт: ", assignedPort
```

---

### `listen`

```nim
proc listen*(socket: AsyncSocket, backlog = SOMAXCONN) {.tags: [ReadIOEffect].}
```

Переводит сокет в режим ожидания входящих соединений (пассивный режим). Вызывается после `bindAddr` и до `accept`.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Привязанный `AsyncSocket` |
| `backlog` | Максимальное число ожидающих соединений в очереди. По умолчанию: `SOMAXCONN` |

**Исключения:** `OSError` при ошибке.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

var server = newAsyncSocket()
server.setSockOpt(OptReuseAddr, true)
server.bindAddr(Port(8080))
server.listen()
echo "Сервер слушает..."
```

---

### `accept`

```nim
proc accept*(socket: AsyncSocket,
             flags = {SocketFlag.SafeDisconn}): owned(Future[AsyncSocket])
```

Ожидает и принимает одно входящее соединение. Возвращает только клиентский сокет (без адреса). Если нужен IP-адрес клиента, используйте `acceptAddr`.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Слушающий `AsyncSocket` |
| `flags` | Флаги сокета. По умолчанию: `{SafeDisconn}` |

**Возвращает:** `Future[AsyncSocket]` — подключённый клиентский сокет.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc serve() {.async.} =
  var server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(8080))
  server.listen()
  
  echo "Ожидание подключений..."
  while true:
    let client = await server.accept()
    echo "Клиент подключился"
    asyncCheck (proc() {.async.} =
      let line = await client.recvLine()
      await client.send("Эхо: " & line & "\r\n")
      client.close()
    )()

waitFor serve()
```

---

### `acceptAddr`

```nim
proc acceptAddr*(socket: AsyncSocket,
                 flags = {SocketFlag.SafeDisconn},
                 inheritable = defined(nimInheritHandles)):
      owned(Future[tuple[address: string, client: AsyncSocket]])
```

Ожидает и принимает одно входящее соединение. Возвращает и клиентский сокет, **и** строку с удалённым адресом.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Слушающий `AsyncSocket` |
| `flags` | Флаги сокета. По умолчанию: `{SafeDisconn}` |
| `inheritable` | Наследуем ли дескриптор клиента дочерними процессами |

**Возвращает:** `Future[tuple[address: string, client: AsyncSocket]]`

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc serve() {.async.} =
  var server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(8080))
  server.listen()
  
  while true:
    let (address, client) = await server.acceptAddr()
    echo "Соединение от: ", address
    asyncCheck (proc() {.async.} =
      await client.send("Привет от сервера\r\n")
      client.close()
    )()

waitFor serve()
```

---

### `bindUnix`

```nim
proc bindUnix*(socket: AsyncSocket, path: string) {.tags: [ReadIOEffect].}
```

Привязывает Unix domain socket к пути `path` в файловой системе. **Только POSIX** (Linux, macOS, BSD). После этого вызовите `listen` для начала приёма соединений.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Непривязанный `AsyncSocket` с семейством `AF_UNIX` |
| `path` | Путь к файлу сокета |

**Исключения:** `OSError` при ошибке.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch, os]

proc serve() {.async.} =
  let sockPath = "/tmp/myapp.sock"
  if fileExists(sockPath): removeFile(sockPath)
  
  var server = newAsyncSocket(AF_UNIX, SOCK_STREAM, 0)
  server.bindUnix(sockPath)
  server.listen()
  echo "Unix-сокет сервера: ", sockPath
  
  let client = await server.accept()
  let msg = await client.recvLine()
  echo "Получено: ", msg
  client.close()
  server.close()

waitFor serve()
```

---

## Параметры и информация о сокете

### `getLocalAddr`

```nim
proc getLocalAddr*(socket: AsyncSocket): (string, Port)
```

Возвращает локальный адрес и порт, к которым привязан сокет. Обёртка над `getsockname`.

**Возвращает:** Кортеж `(address: string, port: Port)`.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

var sock = newAsyncSocket()
sock.bindAddr(Port(0))  # ОС выбирает порт
let (addr, port) = sock.getLocalAddr()
echo "Привязан к: ", addr, ":", port
```

---

### `getPeerAddr`

```nim
proc getPeerAddr*(socket: AsyncSocket): (string, Port)
```

Возвращает удалённый адрес и порт подключённого пира. Обёртка над `getpeername`. **Недоступно** при сборке с `nimNetLite`.

**Возвращает:** Кортеж `(address: string, port: Port)`.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  let (peerAddr, peerPort) = sock.getPeerAddr()
  echo "Подключено к: ", peerAddr, ":", peerPort
  sock.close()

waitFor main()
```

---

### `getSockOpt`

```nim
proc getSockOpt*(socket: AsyncSocket, opt: SOBool, level = SOL_SOCKET): bool {.tags: [ReadIOEffect].}
```

Читает булев параметр сокета. Часто используемые параметры: `OptReuseAddr`, `OptReusePort`, `OptNoDelay`, `OptKeepAlive`.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Сокет для опроса |
| `opt` | Параметр для чтения (значение `SOBool`) |
| `level` | Уровень параметра. По умолчанию: `SOL_SOCKET` |

**Возвращает:** `true`, если параметр включён.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

var sock = newAsyncSocket()
echo "ReuseAddr: ", sock.getSockOpt(OptReuseAddr)
echo "KeepAlive: ", sock.getSockOpt(OptKeepAlive)
sock.close()
```

---

### `setSockOpt`

```nim
proc setSockOpt*(socket: AsyncSocket, opt: SOBool, value: bool,
                 level = SOL_SOCKET) {.tags: [WriteIOEffect].}
```

Устанавливает булев параметр сокета.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `socket` | Конфигурируемый сокет |
| `opt` | Параметр для установки (значение `SOBool`) |
| `value` | `true` — включить, `false` — выключить |
| `level` | Уровень параметра. По умолчанию: `SOL_SOCKET` |

**Часто используемые параметры:**

| Параметр | Описание |
|----------|----------|
| `OptReuseAddr` | Разрешить повторное использование адреса (избегает ошибки «адрес уже занят») |
| `OptReusePort` | Разрешить нескольким сокетам привязываться к одному порту |
| `OptNoDelay` | Отключить алгоритм Нэйгла (только TCP) |
| `OptKeepAlive` | Включить TCP keepalive для обнаружения разорванных соединений |
| `OptBroadcast` | Разрешить отправку широковещательных датаграмм (UDP) |

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

var server = newAsyncSocket()
server.setSockOpt(OptReuseAddr, true)   # избегать ошибки «адрес занят»
server.setSockOpt(OptKeepAlive, true)   # обнаруживать мёртвые соединения
server.bindAddr(Port(8080))
server.listen()
```

---

### `getFd`

```nim
proc getFd*(socket: AsyncSocket): SocketHandle
```

Возвращает «сырой» дескриптор сокета ОС (`SocketHandle`). Полезен для низкоуровневого взаимодействия с API ОС или нативными библиотеками сокетов.

**Возвращает:** `SocketHandle`

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

var sock = newAsyncSocket()
let fd = sock.getFd()
echo "Дескриптор сокета: ", fd.int
```

---

### `isClosed`

```nim
proc isClosed*(socket: AsyncSocket): bool
```

Возвращает `true`, если сокет был закрыт через `close()`.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

var sock = newAsyncSocket()
echo "До закрытия: ", sock.isClosed()  # false
sock.close()
echo "После закрытия: ", sock.isClosed()   # true
```

---

### `isSsl`

```nim
proc isSsl*(socket: AsyncSocket): bool
```

Возвращает `true`, если сокет был обёрнут в SSL-контекст через `wrapSocket` или `wrapConnectedSocket`.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

var sock = newAsyncSocket()
echo "SSL активен: ", sock.isSsl()  # false

# После wrapSocket(ctx, sock):
# echo "SSL активен: ", sock.isSsl()  # true
sock.close()
```

---

### `hasDataBuffered`

```nim
proc hasDataBuffered*(s: AsyncSocket): bool
```

Возвращает `true`, если в внутреннем буфере чтения сокета есть непрочитанные данные. Позволяет проверить, можно ли прочитать данные без ожидания.

> Доступно с Nim 1.5.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  
  # Читаем немного данных в буфер:
  discard await sock.recv(100)
  
  if sock.hasDataBuffered():
    echo "В буфере ещё есть данные"
  sock.close()

waitFor main()
```

---

## SSL-операции

> Все SSL-процедуры требуют компиляции с `-d:ssl`.

### `wrapSocket`

```nim
proc wrapSocket*(ctx: SslContext, socket: AsyncSocket)
```

Преобразует `AsyncSocket` в SSL-сокет с помощью указанного `SslContext`. Должна вызываться **до** подключения или принятия соединения. Настраивает внутренние BIO-буферы памяти OpenSSL.

> **Отказ от ответственности:** Поддержка SSL может содержать уязвимости. Используйте с осторожностью.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `ctx` | `SslContext`, созданный через `net.newContext()` |
| `socket` | Сокет для обёртывания |

**Пример:**
```nim
import std/[asyncnet, asyncdispatch, net]

proc main() {.async.} =
  let ctx = newContext()  # из std/net
  var sock = newAsyncSocket()
  wrapSocket(ctx, sock)
  await sock.connect("example.com", Port(443))
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  let data = await sock.recv(4096)
  echo data
  sock.close()

waitFor main()
```

---

### `wrapConnectedSocket`

```nim
proc wrapConnectedSocket*(ctx: SslContext, socket: AsyncSocket,
                          handshake: SslHandshakeType,
                          hostname: string = "")
```

Обёртывает **уже подключённый** сокет в SSL и немедленно выполняет SSL-рукопожатие. Используйте `handshakeAsClient` для исходящих соединений, `handshakeAsServer` — для принятых.

Если указан `hostname` (не IP-адрес), устанавливает SNI-расширение для TLS 1.0+.

> **Отказ от ответственности:** Поддержка SSL может содержать уязвимости. Используйте с осторожностью.

**Параметры:**

| Параметр | Описание |
|----------|----------|
| `ctx` | `SslContext` |
| `socket` | Уже TCP-подключённый сокет |
| `handshake` | `handshakeAsClient` или `handshakeAsServer` |
| `hostname` | SNI-имя хоста (только для клиента, необязательно) |

**Пример:**
```nim
import std/[asyncnet, asyncdispatch, net]

proc main() {.async.} =
  let ctx = newContext()
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(443))
  
  # Апгрейд до SSL на уже подключённом сокете:
  wrapConnectedSocket(ctx, sock, handshakeAsClient, "example.com")
  
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  let resp = await sock.recv(4096)
  echo resp[0..200]
  sock.close()

waitFor main()
```

---

### `getPeerCertificates`

```nim
proc getPeerCertificates*(socket: AsyncSocket): seq[Certificate]
```

Возвращает цепочку сертификатов, предоставленную удалённой стороной после завершённого SSL-рукопожатия. Цепочка упорядочена от конечного сертификата к корневому. Возвращает пустую последовательность, если сокет не является SSL или рукопожатие не было успешно завершено.

> Доступно с Nim 1.1.

**Возвращает:** `seq[Certificate]` — цепочка сертификатов, листовой первым.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch, net]

proc main() {.async.} =
  let ctx = newContext()
  var sock = newAsyncSocket()
  wrapSocket(ctx, sock)
  await sock.connect("example.com", Port(443))
  
  let certs = sock.getPeerCertificates()
  echo "Длина цепочки сертификатов: ", certs.len
  for cert in certs:
    echo "  Субъект: ", cert.subject
  
  sock.close()

waitFor main()
```

---

### `sslHandle`

```nim
proc sslHandle*(self: AsyncSocket): SslPtr
```

Возвращает «сырой» дескриптор `SSL*` из OpenSSL. Полезен для продвинутого взаимодействия с OpenSSL (например, запрос шифра, информации о сессии или прямой вызов функций OpenSSL).

**Возвращает:** `SslPtr` (указатель `SSL*` из OpenSSL).

**Пример:**
```nim
import std/[asyncnet, asyncdispatch, net, openssl]

proc main() {.async.} =
  let ctx = newContext()
  var sock = newAsyncSocket()
  wrapSocket(ctx, sock)
  await sock.connect("example.com", Port(443))
  
  let ssl = sock.sslHandle()
  # Теперь можно вызывать функции OpenSSL напрямую с `ssl`
  sock.close()

waitFor main()
```

---

## Unix Domain Sockets

### Сводка

| Функция | Направление | Платформа |
|---------|-------------|-----------|
| `bindUnix` | Сервер | Только POSIX |
| `connectUnix` | Клиент | Только POSIX |

Подробную документацию и примеры смотрите в соответствующих разделах выше.

---

## Закрытие сокета

### `close`

```nim
proc close*(socket: AsyncSocket)
```

Закрывает сокет и освобождает все связанные ресурсы. Если сокет уже закрыт, вызов является безопасным нейтральным действием.

Для SSL-сокетов выполняет корректное `SSL_shutdown` перед закрытием, если SSL-рукопожатие было завершено.

**Пример:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  
  # ... работа с сокетом ...
  
  sock.close()
  echo "Сокет закрыт: ", sock.isClosed()  # true
  
  # Безопасно вызвать повторно:
  sock.close()  # нейтральная операция

waitFor main()
```

---

## Полные примеры

### Эхо-сервер

```nim
import std/[asyncnet, asyncdispatch]

proc processClient(client: AsyncSocket) {.async.} =
  while true:
    let line = await client.recvLine()
    if line.len == 0:
      break
    await client.send(line & "\r\n")
  client.close()

proc serve() {.async.} =
  var server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(7777))
  server.listen()
  echo "Эхо-сервер слушает порт 7777"
  
  while true:
    let client = await server.accept()
    asyncCheck processClient(client)

waitFor serve()
```

---

### Чат-сервер

```nim
import std/[asyncnet, asyncdispatch]

var clients: seq[AsyncSocket] = @[]

proc processClient(client: AsyncSocket) {.async.} =
  while true:
    let line = await client.recvLine()
    if line.len == 0:
      clients.keepIf(proc(c: AsyncSocket): bool = c != client)
      break
    for c in clients:
      if c != client:
        await c.send(line & "\r\n")
  client.close()

proc serve() {.async.} =
  var server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(12345))
  server.listen()
  echo "Чат-сервер на порту 12345"
  
  while true:
    let client = await server.accept()
    clients.add(client)
    asyncCheck processClient(client)

waitFor serve()
```

---

### HTTP-клиент (минимальный)

```nim
import std/[asyncnet, asyncdispatch, strutils]

proc httpGet(host: string, port: Port, path: string): Future[string] {.async.} =
  let sock = await dial(host, port)
  await sock.send("GET " & path & " HTTP/1.0\r\nHost: " & host & "\r\n\r\n")
  
  result = ""
  while true:
    let chunk = await sock.recv(4096)
    if chunk.len == 0: break
    result.add chunk
  sock.close()

proc main() {.async.} =
  let response = await httpGet("example.com", Port(80), "/")
  let lines = response.splitLines()
  echo lines[0]  # строка HTTP-статуса

waitFor main()
```

---

### UDP-сервер с эхо

```nim
import std/[asyncnet, asyncdispatch]

proc serve() {.async.} =
  var sock = newAsyncSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
  sock.bindAddr(Port(9999))
  echo "UDP-сервер слушает порт 9999"
  
  while true:
    let (data, address, port) = await sock.recvFrom(1024)
    echo "От ", address, ":", port, " -> ", data
    # Отправляем обратно:
    await sock.sendTo(address, port, data)

waitFor serve()
```

---

## Сводная таблица доступности по платформам

| Функция | Linux | macOS | Windows | Примечания |
|---------|:-----:|:-----:|:-------:|-----------|
| `newAsyncSocket` | ✓ | ✓ | ✓ | Все перегрузки |
| `dial` | ✓ | ✓ | ✓ | |
| `connect` | ✓ | ✓ | ✓ | |
| `connectUnix` | ✓ | ✓ | ✗ | Только POSIX |
| `bindAddr` | ✓ | ✓ | ✓ | |
| `bindUnix` | ✓ | ✓ | ✗ | Только POSIX |
| `listen` | ✓ | ✓ | ✓ | |
| `accept` | ✓ | ✓ | ✓ | |
| `acceptAddr` | ✓ | ✓ | ✓ | |
| `send` (строка) | ✓ | ✓ | ✓ | |
| `send` (указатель) | ✓ | ✓ | ✓ | |
| `sendTo` | ✓ | ✓ | ✓ | UDP; с 1.3 |
| `recv` | ✓ | ✓ | ✓ | |
| `recvInto` | ✓ | ✓ | ✓ | |
| `recvLine` | ✓ | ✓ | ✓ | |
| `recvLineInto` | ✓ | ✓ | ✓ | |
| `recvFrom` | ✓ | ✓ | ✓ | UDP; с 1.3 |
| `getLocalAddr` | ✓ | ✓ | ✓ | |
| `getPeerAddr` | ✓ | ✓ | ✓ | Недоступно с nimNetLite |
| `getSockOpt` | ✓ | ✓ | ✓ | |
| `setSockOpt` | ✓ | ✓ | ✓ | |
| `getFd` | ✓ | ✓ | ✓ | |
| `isClosed` | ✓ | ✓ | ✓ | |
| `isSsl` | ✓ | ✓ | ✓ | |
| `hasDataBuffered` | ✓ | ✓ | ✓ | с 1.5 |
| `close` | ✓ | ✓ | ✓ | |
| `wrapSocket` | ✓ | ✓ | ✓ | требует `-d:ssl` |
| `wrapConnectedSocket` | ✓ | ✓ | ✓ | требует `-d:ssl` |
| `getPeerCertificates` | ✓ | ✓ | ✓ | требует `-d:ssl`; с 1.1 |
| `sslHandle` | ✓ | ✓ | ✓ | требует `-d:ssl` |

---

*Сгенерировано на основе модуля `std/asyncnet` стандартной библиотеки Nim.*
