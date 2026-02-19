# Справочник модуля `std/net` (Nim)

> Высокоуровневый кросс-платформенный интерфейс для работы с сокетами.
> Предназначен преимущественно для **блокирующих** сокетов.
> Для неблокирующих асинхронных сокетов используйте `asyncnet` + `asyncdispatch`.
>
> SSL/TLS: для включения поддержки скомпилируйте с флагом `-d:ssl`.

---

## Оглавление

1. [Константы](#константы)
2. [Типы](#типы)
   - [Socket, SocketImpl](#socket-socketimpl)
   - [IpAddress, IpAddressFamily](#ipaddress-ipaddressfamily)
   - [SOBool — опции сокета](#sobool--опции-сокета)
   - [SocketFlag](#socketflag)
   - [Типы, специфичные для SSL](#типы-специфичные-для-ssl)
3. [Создание сокета](#создание-сокета)
4. [Работа с IP-адресами](#работа-с-ip-адресами)
5. [Адреса специального назначения](#адреса-специального-назначения)
6. [Подключение (клиент)](#подключение-клиент)
7. [Сервер: bind, listen, accept](#сервер-bind-listen-accept)
8. [Unix-сокеты (POSIX)](#unix-сокеты-posix)
9. [Отправка данных](#отправка-данных)
10. [Получение данных](#получение-данных)
11. [Закрытие сокета](#закрытие-сокета)
12. [Опции сокета](#опции-сокета)
13. [Буферизация и состояние](#буферизация-и-состояние)
14. [Обработка ошибок](#обработка-ошибок)
15. [SSL/TLS API](#ssltls-api)
16. [SockAddr утилиты](#sockaddr-утилиты)
17. [Быстрые рецепты](#быстрые-рецепты)

---

## Константы

```nim
BufferSize*: int = 4000
  ## Размер внутреннего буфера буферизованного сокета.

MaxLineLength* = 1_000_000
  ## Максимальная длина строки при чтении readLine/recvLine.
```

---

## Типы

### Socket, SocketImpl

```nim
type
  SocketImpl* = object
    fd: SocketHandle
    isBuffered: bool
    buffer: array[0..BufferSize, char]
    currPos: int
    bufLen: int
    # (SSL-поля только при -d:ssl)
    lastError: OSErrorCode
    domain: Domain
    sockType: SockType
    protocol: Protocol

  Socket* = ref SocketImpl
```

`Socket` — основной тип модуля, ссылка на `SocketImpl`. Все операции выполняются над этим типом.

---

### IpAddress, IpAddressFamily

```nim
type
  IpAddressFamily* {.pure.} = enum
    IPv6  ## IPv6-адрес
    IPv4  ## IPv4-адрес

  IpAddress* = object
    case family*: IpAddressFamily
    of IpAddressFamily.IPv6:
      address_v6*: array[0..15, uint8]
    of IpAddressFamily.IPv4:
      address_v4*: array[0..3, uint8]
```

Хранит IPv4 или IPv6 адрес в виде байтов. Поле `family` определяет, какое поле данных активно.

---

### SOBool — опции сокета

```nim
type SOBool* = enum
  OptAcceptConn   ## Сокет принимает входящие соединения
  OptBroadcast    ## Разрешить широковещательные пакеты
  OptDebug        ## Включить отладку
  OptDontRoute    ## Не использовать маршрутизацию
  OptKeepAlive    ## Отправлять keepalive-пакеты
  OptOOBInline    ## Получать OOB-данные в обычном потоке
  OptReuseAddr    ## Разрешить повторное использование адреса
  OptReusePort    ## Разрешить несколько сокетов на одном порту
  OptNoDelay      ## Отключить алгоритм Нэйгла (Nagle)
```

---

### SocketFlag

```nim
type SocketFlag* {.pure.} = enum
  Peek         ## Читать без удаления из буфера
  SafeDisconn  ## Не выбрасывать исключение при разрыве соединения
```

`SafeDisconn` позволяет корректно обрабатывать случаи, когда клиент закрывает соединение в момент `accept` или `send`.

---

### ReadLineResult

```nim
type ReadLineResult* = enum
  ReadFullLine      ## Прочитана полная строка
  ReadPartialLine   ## Прочитана только часть строки
  ReadDisconnected  ## Соединение разорвано
  ReadNone          ## Данных нет
```

---

### Типы, специфичные для SSL

Доступны только при компиляции с `-d:ssl`.

```nim
Certificate* = string  ## DER-encoded сертификат

SslError*           # исключение для SSL-ошибок (CatchableError)
TimeoutError*       # исключение при таймауте (CatchableError)

SslCVerifyMode* = enum
  CVerifyNone               ## Сертификаты не проверяются
  CVerifyPeer               ## Сертификаты проверяются
  CVerifyPeerUseEnvVars     ## Проверяются + учитываются SSL_CERT_FILE / SSL_CERT_DIR

SslProtVersion* = enum
  protSSLv2, protSSLv3, protTLSv1, protSSLv23

SslHandshakeType* = enum
  handshakeAsClient
  handshakeAsServer

SslAcceptResult* = enum
  AcceptNoClient = 0
  AcceptNoHandshake
  AcceptSuccess

SslContext* = ref object  ## Контекст SSL
  context*: SslCtx

SslClientGetPskFunc* = proc(hint: string): tuple[identity: string, psk: string]
SslServerGetPskFunc* = proc(identity: string): string
```

---

## Создание сокета

### `newSocket` (из SocketHandle)
```nim
proc newSocket*(fd: SocketHandle, domain: Domain = AF_INET,
    sockType: SockType = SOCK_STREAM,
    protocol: Protocol = IPPROTO_TCP, buffered = true): owned(Socket)
```
Создаёт объект `Socket` из уже существующего файлового дескриптора. Применяется при работе с нативными API.

---

### `newSocket` (создание нового)
```nim
proc newSocket*(domain: Domain = AF_INET, sockType: SockType = SOCK_STREAM,
                protocol: Protocol = IPPROTO_TCP, buffered = true,
                inheritable = defined(nimInheritHandles)): owned(Socket)

proc newSocket*(domain, sockType, protocol: cint, buffered = true,
                inheritable = defined(nimInheritHandles)): owned(Socket)
```
Создаёт новый сокет. Параметры:
- `domain` — семейство адресов (`AF_INET`, `AF_INET6`, `AF_UNIX` и др.)
- `sockType` — тип (`SOCK_STREAM` для TCP, `SOCK_DGRAM` для UDP)
- `protocol` — протокол (`IPPROTO_TCP`, `IPPROTO_UDP`, `IPPROTO_NONE`)
- `buffered` — использовать внутренний буфер (рекомендуется `true`)
- `inheritable` — наследовать дескриптор дочерними процессами

При ошибке выбрасывает `OSError`.

```nim
# TCP сокет (по умолчанию)
let tcpSock = newSocket()

# UDP сокет
let udpSock = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)

# IPv6 TCP
let sock6 = newSocket(AF_INET6, SOCK_STREAM, IPPROTO_TCP)
```

---

### `dial`
```nim
proc dial*(address: string, port: Port,
           protocol = IPPROTO_TCP, buffered = true): owned(Socket)
```
Устанавливает соединение с сервером и возвращает готовый к использованию сокет. В отличие от `newSocket` + `connect`, `dial` автоматически перебирает все IP-адреса хоста (работает для IPv4 и IPv6). При ошибке выбрасывает `OSError` или `IOError`.

```nim
let sock = dial("nim-lang.org", Port(80))
sock.send("GET / HTTP/1.0\r\nHost: nim-lang.org\r\n\r\n")
echo sock.recvLine()
sock.close()
```

---

## Работа с IP-адресами

### `parseIpAddress`
```nim
proc parseIpAddress*(addressStr: string): IpAddress
```
Парсит строку в `IpAddress`. Поддерживает IPv4 (строгий RFC 6943) и IPv6. При ошибке выбрасывает `ValueError`. Для IPv4 не допускаются ведущие нули (восьмеричная форма запрещена).

```nim
let ip4 = parseIpAddress("192.168.1.1")
let ip6 = parseIpAddress("::1")
let ip6full = parseIpAddress("2001:db8::1")
```

---

### `isIpAddress`
```nim
proc isIpAddress*(addressStr: string): bool
```
Возвращает `true`, если строка является корректным IP-адресом (IPv4 или IPv6). Не выбрасывает исключений.

```nim
doAssert isIpAddress("192.168.1.1") == true
doAssert isIpAddress("::1") == true
doAssert isIpAddress("not-an-ip") == false
```

---

### `$` (IpAddress → string)
```nim
proc `$`*(address: IpAddress): string
```
Преобразует `IpAddress` в строку. IPv6-адреса форматируются по RFC (с сокращением нулевых групп с помощью `::`).

```nim
let ip = parseIpAddress("192.168.1.1")
echo $ip  # "192.168.1.1"

let ip6 = parseIpAddress("::1")
echo $ip6  # "::1"
```

---

### `==` (IpAddress)
```nim
proc `==`*(lhs, rhs: IpAddress): bool
```
Сравнивает два IP-адреса. Адреса разных семейств всегда неравны.

```nim
doAssert parseIpAddress("127.0.0.1") == parseIpAddress("127.0.0.1")
doAssert parseIpAddress("::1") != parseIpAddress("127.0.0.1")
```

---

### `toSockAddr` / `fromSockAddr`
```nim
proc toSockAddr*(address: IpAddress, port: Port, sa: var Sockaddr_storage, sl: var SockLen)
proc fromSockAddr*(sa: ..., sl: SockLen, address: var IpAddress, port: var Port)
```
Конвертация между `IpAddress`/`Port` и низкоуровневыми структурами `Sockaddr_storage`. Нужны при работе с нативными системными вызовами.

---

### `getPrimaryIPAddr`
```nim
proc getPrimaryIPAddr*(dest = parseIpAddress("8.8.8.8")): IpAddress
```
Возвращает локальный IP-адрес, используемый для достижения внешней сети. Трафик не отправляется. Поддерживает IPv4 и IPv6. Полезно для публикации локальных сервисов.

```nim
echo getPrimaryIPAddr()  # например "192.168.1.2"
```

---

## Адреса специального назначения

```nim
proc IPv4_any*(): IpAddress        ## 0.0.0.0 — все интерфейсы
proc IPv4_loopback*(): IpAddress   ## 127.0.0.1
proc IPv4_broadcast*(): IpAddress  ## 255.255.255.255

proc IPv6_any*(): IpAddress        ## :: — все интерфейсы
proc IPv6_loopback*(): IpAddress   ## ::1
```

```nim
# Слушать на всех интерфейсах
socket.bindAddr(Port(8080), $IPv4_any())

# Подключиться к локальному серверу
socket.connect($IPv4_loopback(), Port(3000))
```

---

## Подключение (клиент)

### `connect` (без таймаута)
```nim
proc connect*(socket: Socket, address: string, port = Port(0))
```
Подключает сокет к `address:port`. `address` может быть IP-адресом или именем хоста. Перебирает все IP хоста при необходимости. При ошибке выбрасывает `OSError`. Для SSL-сокетов автоматически выполняет handshake.

```nim
let socket = newSocket()
socket.connect("google.com", Port(80))
```

---

### `connect` (с таймаутом)
```nim
proc connect*(socket: Socket, address: string, port = Port(0), timeout: int)
```
Аналог с таймаутом в **миллисекундах**. При истечении времени выбрасывает `TimeoutError`. Внутри временно переводит сокет в неблокирующий режим.

```nim
let socket = newSocket()
socket.connect("example.com", Port(80), timeout = 5000)  # 5 секунд
```

---

## Сервер: bind, listen, accept

### `bindAddr`
```nim
proc bindAddr*(socket: Socket, port = Port(0), address = "")
```
Привязывает сокет к адресу `address:port`. Если `address` — пустая строка, привязывается к `ADDR_ANY` (все интерфейсы: `0.0.0.0` для IPv4, `::` для IPv6). `Port(0)` означает автоматический выбор порта системой.

```nim
let server = newSocket()
server.bindAddr(Port(8080))
```

---

### `listen`
```nim
proc listen*(socket: Socket, backlog = SOMAXCONN)
```
Переводит сокет в режим ожидания входящих соединений. `backlog` — максимальная длина очереди ожидающих соединений. При ошибке выбрасывает `OSError`.

```nim
server.listen()
```

---

### `acceptAddr`
```nim
proc acceptAddr*(server: Socket, client: var owned(Socket), address: var string,
                 flags = {SocketFlag.SafeDisconn},
                 inheritable = defined(nimInheritHandles))
```
Блокирует до прихода входящего соединения. Записывает клиентский сокет в `client` и адрес клиента в `address`. Клиентский сокет наследует настройки буферизации серверного.

По умолчанию флаг `SafeDisconn` обеспечивает автоматический повторный вызов `accept` при разрыве соединения во время ожидания. Для SSL-серверов автоматически выполняет SSL-handshake.

```nim
var client: Socket
var address = ""
server.acceptAddr(client, address)
echo "Клиент: ", address
```

---

### `accept`
```nim
proc accept*(server: Socket, client: var owned(Socket),
             flags = {SocketFlag.SafeDisconn},
             inheritable = defined(nimInheritHandles))
```
Эквивалент `acceptAddr`, но не возвращает адрес клиента. Удобен когда адрес не нужен.

```nim
var client: Socket
server.accept(client)
client.send("Привет!\n")
```

---

### Пример TCP-сервера

```nim
let server = newSocket()
server.setSockOpt(OptReuseAddr, true)
server.bindAddr(Port(8080))
server.listen()

while true:
  var client: Socket
  var address = ""
  server.acceptAddr(client, address)
  echo "Соединение от: ", address
  client.send("HTTP/1.0 200 OK\r\n\r\nHello!\r\n")
  client.close()
```

---

## Unix-сокеты (POSIX)

Доступны только на POSIX-системах (Linux, macOS, BSD).

### `connectUnix`
```nim
proc connectUnix*(socket: Socket, path: string)
```
Подключается к Unix-сокету по пути `path`.

### `bindUnix`
```nim
proc bindUnix*(socket: Socket, path: string)
```
Привязывает сокет к Unix-пути `path`.

```nim
let server = newSocket(AF_UNIX, SOCK_STREAM, IPPROTO_NONE)
server.bindUnix("/tmp/myapp.sock")
server.listen()

let client = newSocket(AF_UNIX, SOCK_STREAM, IPPROTO_NONE)
client.connectUnix("/tmp/myapp.sock")
```

---

## Отправка данных

### `send` (низкоуровневый)
```nim
proc send*(socket: Socket, data: pointer, size: int): int
```
Низкоуровневая отправка данных. Возвращает количество отправленных байт. Не гарантирует отправку всех данных.

---

### `send` (строка)
```nim
proc send*(socket: Socket, data: string,
           flags = {SocketFlag.SafeDisconn}, maxRetries = 100)
```
Высокоуровневая отправка строки. Обрабатывает прерывания и неполные записи, повторяя попытку до `maxRetries` раз. При ошибке выбрасывает `OSError`.

```nim
socket.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
```

---

### `&=` (оператор отправки)
```nim
template `&=`*(socket: Socket; data: typed)
```
Псевдоним для `send`. Удобен для последовательной отправки.

```nim
socket &= "Hello, "
socket &= "World!\n"
```

---

### `trySend`
```nim
proc trySend*(socket: Socket, data: string): bool
```
Безопасная версия `send`. Не выбрасывает исключений — возвращает `false` при ошибке.

```nim
if not socket.trySend("data"):
  echo "Ошибка отправки"
```

---

### `sendTo` (UDP)
```nim
# По имени хоста (низкоуровневый)
proc sendTo*(socket: Socket, address: string, port: Port, data: pointer,
             size: int, af: Domain = AF_INET, flags = 0'i32)

# По имени хоста (строка)
proc sendTo*(socket: Socket, address: string, port: Port, data: string)

# По IpAddress (возвращает число отправленных байт)
proc sendTo*(socket: Socket, address: IpAddress, port: Port,
             data: string, flags = 0'i32): int {.discardable.}
```
Отправляет данные на указанный адрес без предварительного подключения. Предназначен для UDP-сокетов. Для TCP использование запрещено (assertion).

```nim
let udpSock = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
udpSock.sendTo("192.168.0.1", Port(27960), "status\n")

let ip = parseIpAddress("192.168.0.1")
doAssert udpSock.sendTo(ip, Port(27960), "status\c\l") == 8
```

---

## Получение данных

### `recv` (низкоуровневый, pointer)
```nim
proc recv*(socket: Socket, data: pointer, size: int): int
proc recv*(socket: Socket, data: pointer, size: int, timeout: int): int
```
Низкоуровневое чтение в буфер памяти. Для буферизованных сокетов пытается прочитать все `size` байт. Для небуферизованных — возвращает столько, сколько дала ОС. Возвращает 0 при разрыве соединения, отрицательное значение при ошибке.

---

### `recv` (строка, запись в переменную)
```nim
proc recv*(socket: Socket, data: var string, size: int, timeout = -1,
           flags = {SocketFlag.SafeDisconn}): int
```
Читает до `size` байт в строку `data`. Возвращает реальное количество прочитанных байт. При разрыве соединения возвращает 0 и `data = ""`. При ошибке выбрасывает `OSError`. Таймаут в миллисекундах (-1 = без ограничения).

```nim
var data: string
let n = socket.recv(data, 1024, timeout = 5000)
echo "Получено: ", n, " байт"
```

---

### `recv` (строка, возврат значения)
```nim
proc recv*(socket: Socket, size: int, timeout = -1,
           flags = {SocketFlag.SafeDisconn}): string
```
Удобная версия, возвращающая строку напрямую. При разрыве соединения возвращает `""`.

```nim
let response = socket.recv(4096)
if response == "":
  echo "Соединение закрыто"
```

---

### `readLine`
```nim
proc readLine*(socket: Socket, line: var string, timeout = -1,
               flags = {SocketFlag.SafeDisconn}, maxLength = MaxLineLength)
```
Читает строку из сокета. Символы `\r\L` в конце строки удаляются, кроме случая когда строка состоит только из `\r\L`. При разрыве соединения `line = ""`. Таймаут в мс. `maxLength` защищает от DoS (строка обрезается после превышения).

```nim
var line: string
socket.readLine(line, timeout = 10_000)
echo "Строка: ", line
```

---

### `recvLine`
```nim
proc recvLine*(socket: Socket, timeout = -1,
               flags = {SocketFlag.SafeDisconn},
               maxLength = MaxLineLength): string
```
Удобная версия `readLine`, возвращающая строку напрямую.

```nim
let line = socket.recvLine(timeout = 5000)
```

---

### `recvFrom` (UDP)
```nim
proc recvFrom[T: string | IpAddress](socket: Socket, data: var string, length: int,
               address: var T, port: var Port, flags = 0'i32): int
```
Получает UDP-пакет. Сохраняет данные в `data`, адрес отправителя в `address` (строка или `IpAddress`), порт в `port`. Возвращает число прочитанных байт. Не предназначен для TCP (assertion).

**Внимание:** буферизованная реализация отсутствует — данные из буфера сокета не возвращаются.

```nim
var data = ""
var address = ""
var port: Port
let udpSock = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
udpSock.bindAddr(Port(9999))
let n = udpSock.recvFrom(data, 1024, address, port)
echo "От ", address, ":", port, ": ", data
```

---

### `skip`
```nim
proc skip*(socket: Socket, size: int, timeout = -1)
```
Пропускает (игнорирует) `size` байт из сокета. Опциональный таймаут в мс.

```nim
socket.skip(4)  # пропустить заголовок
```

---

## Закрытие сокета

### `close`
```nim
proc close*(socket: Socket, flags = {SocketFlag.SafeDisconn})
```
Закрывает сокет. Для SSL-сокетов предварительно отправляет уведомление о закрытии (`close_notify`). Если `SafeDisconn` установлен, ошибки при отправке уведомления игнорируются. Сбрасывает `fd` в `osInvalidSocket`.

```nim
socket.close()
```

---

## Опции сокета

### `getSockOpt`
```nim
proc getSockOpt*(socket: Socket, opt: SOBool, level = SOL_SOCKET): bool
```
Возвращает значение булевой опции сокета.

```nim
let reuse = socket.getSockOpt(OptReuseAddr)
echo "ReuseAddr: ", reuse
```

---

### `setSockOpt`
```nim
proc setSockOpt*(socket: Socket, opt: SOBool, value: bool, level = SOL_SOCKET)
```
Устанавливает булевую опцию сокета.

```nim
socket.setSockOpt(OptReuseAddr, true)
socket.setSockOpt(OptReusePort, true)
socket.setSockOpt(OptNoDelay, true, level = IPPROTO_TCP.cint)
socket.setSockOpt(OptKeepAlive, true)
```

---

### `toCInt`
```nim
proc toCInt*(opt: SOBool): cint
```
Конвертирует `SOBool` в соответствующую константу OS для использования в нативных системных вызовах.

---

### `getLocalAddr`
```nim
proc getLocalAddr*(socket: Socket): (string, Port)
```
Возвращает локальный адрес и порт сокета. Обёртка над `getsockname`.

```nim
let (addr, port) = socket.getLocalAddr()
echo "Слушаем на: ", addr, ":", port
```

---

### `getPeerAddr`
```nim
proc getPeerAddr*(socket: Socket): (string, Port)
```
Возвращает адрес и порт удалённого конца соединения. Обёртка над `getpeername`.

```nim
let (peerAddr, peerPort) = socket.getPeerAddr()
echo "Соединён с: ", peerAddr, ":", peerPort
```

---

### `getFd`
```nim
proc getFd*(socket: Socket): SocketHandle
```
Возвращает нативный файловый дескриптор сокета для использования в системных вызовах.

---

### `isSsl`
```nim
proc isSsl*(socket: Socket): bool
```
Возвращает `true`, если сокет является SSL-сокетом.

---

## Буферизация и состояние

### `hasDataBuffered`
```nim
proc hasDataBuffered*(s: Socket): bool
```
Возвращает `true`, если в внутреннем буфере сокета есть непрочитанные данные. Полезно для select-подобных паттернов без блокировки.

```nim
if socket.hasDataBuffered:
  let line = socket.recvLine()
```

---

## Обработка ошибок

### `socketError`
```nim
proc socketError*(socket: Socket, err: int = -1, async = false,
                  lastError = (-1).OSErrorCode,
                  flags: set[SocketFlag] = {})
```
Анализирует код ошибки и выбрасывает соответствующее исключение. Для SSL-сокетов использует `SSL_get_error`, для остальных — `osLastError`. При `async = true` не выбрасывает исключение, если данных просто ещё нет.

---

### `getSocketError`
```nim
proc getSocketError*(socket: Socket): OSErrorCode
```
Возвращает последний код ошибки. Если `osLastError` сброшен, берёт сохранённую ошибку из объекта сокета.

---

### `isDisconnectionError`
```nim
proc isDisconnectionError*(flags: set[SocketFlag], lastError: OSErrorCode): bool
```
Возвращает `true`, если ошибка является ошибкой разрыва соединения и `SafeDisconn` установлен. Обрабатывает платформо-специфичные коды ошибок (ECONNRESET, EPIPE, WSAECONNRESET и т.д.).

---

### `toOSFlags`
```nim
proc toOSFlags*(socketFlags: set[SocketFlag]): cint
```
Преобразует набор `SocketFlag` в битовую маску для системных вызовов. `SafeDisconn` в маску не включается.

---

## SSL/TLS API

Все процедуры ниже доступны только при `-d:ssl`.

### `newContext`
```nim
proc newContext*(protVersion = protSSLv23, verifyMode = CVerifyPeer,
                 certFile = "", keyFile = "", cipherList = CiphersIntermediate,
                 caDir = "", caFile = "", ciphersuites = CiphersModern): SslContext
```
Создаёт SSL-контекст. Параметры:
- `verifyMode` — режим проверки сертификатов
- `certFile` / `keyFile` — файлы сертификата и ключа (нужны для сервера)
- `cipherList` / `ciphersuites` — список разрешённых шифров (TLS 1.2 и 1.3)
- `caDir` / `caFile` — пути к корневым сертификатам CA

Если `caDir` и `caFile` не заданы, выполняется автоматический поиск по известным путям. При неудаче выбрасывает `IOError`.

Генерация самоподписанного сертификата:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout mykey.pem -out mycert.pem
```

```nim
# Клиентский контекст
let ctx = newContext()

# Серверный контекст
let serverCtx = newContext(certFile = "mycert.pem", keyFile = "mykey.pem")
```

---

### `wrapSocket`
```nim
proc wrapSocket*(ctx: SslContext, socket: Socket)
```
Оборачивает **неподключённый** сокет в SSL-контекст. SSL-сессия начинается при подключении. Вызывайте перед `connect`.

```nim
let socket = newSocket()
let ctx = newContext()
wrapSocket(ctx, socket)
socket.connect("google.com", Port(443))
```

---

### `wrapConnectedSocket`
```nim
proc wrapConnectedSocket*(ctx: SslContext, socket: Socket,
                           handshake: SslHandshakeType, hostname: string = "")
```
Оборачивает **уже подключённый** сокет в SSL и немедленно выполняет handshake. `hostname` нужен для SNI и проверки имени в сертификате.

```nim
# Клиентская сторона
ctx.wrapConnectedSocket(socket, handshakeAsClient, "example.com")

# Серверная сторона
ctx.wrapConnectedSocket(clientSock, handshakeAsServer)
```

---

### `destroyContext`
```nim
proc destroyContext*(ctx: SslContext)
```
Освобождает память, занятую SSL-контекстом.

---

### `sessionIdContext=`
```nim
proc `sessionIdContext=`*(ctx: SslContext, sidCtx: string)
```
Устанавливает идентификатор контекста сессии для возможности повторного использования сессий клиентами. Максимум 32 символа. Должен быть уникальным для приложения. Устанавливается на сервере.

```nim
serverCtx.sessionIdContext = "myapp-v1"
```

---

### `getPeerCertificates`
```nim
proc getPeerCertificates*(sslHandle: SslPtr): seq[Certificate]
proc getPeerCertificates*(socket: Socket): seq[Certificate]
```
Возвращает цепочку сертификатов удалённой стороны (DER-encoded). Handshake должен быть завершён и проверка сертификата должна пройти успешно. Цепочка упорядочена от листового к корневому. Возвращает пустую последовательность при ошибке.

```nim
let certs = socket.getPeerCertificates()
echo "Сертификатов: ", certs.len
```

---

### `gotHandshake`
```nim
proc gotHandshake*(socket: Socket): bool
```
Возвращает `true`, если SSL-handshake был успешно завершён. Выбрасывает `SslError` если сокет не является SSL-сокетом.

---

### `raiseSSLError`
```nim
proc raiseSSLError*(s = "") {.raises: [SslError].}
```
Выбрасывает `SslError`. Если `s` пустой, берёт описание из стека ошибок OpenSSL.

---

### `sslHandle`
```nim
proc sslHandle*(self: Socket): SslPtr
```
Возвращает указатель на SSL-структуру OpenSSL (`SSL*`). Для продвинутого взаимодействия с `openssl`.

---

### `getExtraData` / `setExtraData`
```nim
proc getExtraData*(ctx: SslContext, index: int): RootRef
proc setExtraData*(ctx: SslContext, index: int, data: RootRef)
```
Позволяют хранить и извлекать произвольные данные внутри `SslContext`. Управляют счётчиком ссылок через GC.

---

### PSK (Pre-Shared Key) функции

```nim
proc `pskIdentityHint=`*(ctx: SslContext, hint: string)
  ## Устанавливает подсказку идентичности для клиента (PSK-шифрование)

proc `clientGetPskFunc=`*(ctx: SslContext, fun: SslClientGetPskFunc)
  ## Устанавливает функцию получения PSK на клиентской стороне

proc `serverGetPskFunc=`*(ctx: SslContext, fun: SslServerGetPskFunc)
  ## Устанавливает функцию получения PSK на серверной стороне

proc clientGetPskFunc*(ctx: SslContext): SslClientGetPskFunc
proc serverGetPskFunc*(ctx: SslContext): SslServerGetPskFunc

proc getPskIdentity*(socket: Socket): string
  ## Возвращает PSK-идентичность, предоставленную клиентом
```

Использование PSK-шифрования (без сертификатов):

```nim
# Серверная сторона
let ctx = newContext(verifyMode = CVerifyNone)
ctx.serverGetPskFunc = proc(identity: string): string =
  if identity == "user": return "secretkey"
  return ""
```

---

## SockAddr утилиты

### `toSockAddr`
```nim
proc toSockAddr*(address: IpAddress, port: Port, sa: var Sockaddr_storage, sl: var SockLen)
```
Конвертирует `IpAddress` + `Port` в `Sockaddr_storage` для нативных вызовов.

### `fromSockAddr`
```nim
proc fromSockAddr*(sa: ..., sl: SockLen, address: var IpAddress, port: var Port)
```
Обратная операция. Выбрасывает `ValueError` при неизвестном семействе адресов.

---

## Быстрые рецепты

### Простой TCP-клиент

```nim
import std/net

let socket = newSocket()
socket.connect("example.com", Port(80))
socket.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")

var response = ""
while true:
  let chunk = socket.recv(4096)
  if chunk == "": break
  response.add(chunk)

socket.close()
echo response
```

---

### HTTPS-клиент (SSL)

```nim
import std/net

let socket = newSocket()
let ctx = newContext()
wrapSocket(ctx, socket)
socket.connect("example.com", Port(443))
socket.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
echo socket.recvLine()
socket.close()
```

---

### UDP-обмен

```nim
import std/net

# Отправитель
let sender = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
sender.sendTo("127.0.0.1", Port(9999), "Hello UDP!")

# Получатель
let receiver = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
receiver.bindAddr(Port(9999))
var data = ""
var fromAddr = ""
var fromPort: Port
receiver.recvFrom(data, 1024, fromAddr, fromPort)
echo "Получено от ", fromAddr, ": ", data
```

---

### Многопоточный TCP-сервер

```nim
import std/net, std/threads

proc handleClient(client: Socket) {.thread.} =
  let line = client.recvLine()
  client.send("Эхо: " & line & "\r\n")
  client.close()

let server = newSocket()
server.setSockOpt(OptReuseAddr, true)
server.bindAddr(Port(8080))
server.listen()

while true:
  var client: Socket
  var address = ""
  server.acceptAddr(client, address)
  var t: Thread[Socket]
  createThread(t, handleClient, client)
  t.joinThread()
```
