# Модуль Nim `std/nativesockets` — полный справочник

> Низкоуровневый кросс-платформенный интерфейс сокетов. Модуль вплотную соответствует BSD/POSIX и WinSock2 API.  
> Для более высокоуровневого и удобного API используйте модуль `std/net`.

---

## Содержание

1. [Обзор](#обзор)
2. [Типы](#типы)
3. [Константы](#константы)
4. [Жизненный цикл сокета](#жизненный-цикл-сокета)
5. [Разрешение адресов](#разрешение-адресов)
6. [Преобразование порядка байт](#преобразование-порядка-байт)
7. [Параметры сокета](#параметры-сокета)
8. [Инспекция адресов](#инспекция-адресов)
9. [Готовность ввода-вывода — select](#готовность-ввода-вывода--select)
10. [Утилиты преобразования перечислений](#утилиты-преобразования-перечислений)
11. [Платформо-зависимые особенности](#платформо-зависимые-особенности)
12. [Таблица быстрого доступа](#таблица-быстрого-доступа)

---

## Обзор

`std/nativesockets` оборачивает нативный API сокетов ОС единообразно для Windows (WinSock2) и POSIX-систем (Linux, macOS, BSD и др.). Модуль реэкспортирует необходимые платформенные константы и низкоуровневые типы, чтобы код компилировался на всех поддерживаемых платформах без условных импортов.

```nim
import std/nativesockets

let sock = createNativeSocket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
if sock == osInvalidSocket:
  quit("Не удалось создать сокет")
```

---

## Типы

### `Port`

```nim
type Port* = distinct uint16
```

Строго типизированная обёртка над 16-битным номером порта. Использование отдельного типа предотвращает случайное смешение портов с другими целыми числами.

```nim
let httpPort  = Port(80)
let httpsPort = Port(443)
echo $httpPort   # → "80"
```

Операторы `==` и `$` предоставлены через `borrow`.

---

### `Domain`

```nim
type Domain* = enum
  AF_UNSPEC = 0   ## Не задано; может быть определено автоматически в getaddrinfo
  AF_UNIX   = 1   ## Unix domain сокеты (локальный IPC). Недоступно на Windows.
  AF_INET   = 2   ## IPv4
  AF_INET6  = …   ## IPv6 (значение зависит от ОС: 10 на Linux, 30 на macOS, 23 на Windows)
```

Задаёт семейство адресов / семейство протоколов для сокета.

---

### `SockType`

```nim
type SockType* = enum
  SOCK_STREAM    = 1  ## Надёжный потоковый сокет с установкой соединения (TCP)
  SOCK_DGRAM     = 2  ## Ненадёжные дейтаграммы без соединения (UDP)
  SOCK_RAW       = 3  ## Прямой доступ к протоколам сетевого уровня
  SOCK_SEQPACKET = 5  ## Надёжные упорядоченные пакеты с соединением
```

---

### `Protocol`

```nim
type Protocol* = enum
  IPPROTO_TCP    = 6   ## Протокол управления передачей
  IPPROTO_UDP    = 17  ## Протокол пользовательских дейтаграмм
  IPPROTO_IP          ## Интернет-протокол (общий)
  IPPROTO_IPV6        ## IPv6
  IPPROTO_RAW         ## Сырые пакеты (не поддерживается на Windows)
  IPPROTO_ICMP        ## Протокол управляющих сообщений Интернета
  IPPROTO_ICMPV6      ## ICMPv6
```

---

### `Servent`

```nim
type Servent* = object
  name*:    string        ## Официальное имя сервиса
  aliases*: seq[string]   ## Альтернативные имена
  port*:    Port          ## Номер порта
  proto*:   string        ## Имя протокола (например, "tcp")
```

Возвращается функциями `getServByName` и `getServByPort`.

---

### `Hostent`

```nim
type Hostent* = object
  name*:     string        ## Официальное имя хоста
  aliases*:  seq[string]   ## Альтернативные имена хоста
  addrtype*: Domain        ## Семейство адресов (AF_INET или AF_INET6)
  length*:   int           ## Длина каждого адреса в байтах
  addrList*: seq[string]   ## Список IP-адресов в виде строк
```

Возвращается функциями `getHostByName` и `getHostByAddr`.

---

## Константы

### `IPPROTO_NONE`

```nim
const IPPROTO_NONE* = IPPROTO_IP
```

Используйте, когда тип сокета требует нулевое значение протокола (например, Unix domain сокеты).

---

### `osInvalidSocket`

```nim
let osInvalidSocket*: SocketHandle
```

Значение-сторож, обозначающее недействительный или неудавшийся сокет. На POSIX это `-1`; на Windows — `INVALID_SOCKET`.

---

### Реэкспортируемые константы параметров сокета

| Константа | Значение |
|---|---|
| `SO_ERROR` | Получить и сбросить накопившуюся ошибку сокета |
| `SOL_SOCKET` | Уровень опций сокета |
| `SOMAXCONN` | Максимальная длина очереди ожидающих соединений для `listen` |
| `SO_ACCEPTCONN` | Сокет принимает соединения |
| `SO_BROADCAST` | Разрешить широковещательные сообщения |
| `SO_DEBUG` | Включить отладочную запись |
| `SO_DONTROUTE` | Обходить таблицу маршрутизации |
| `SO_KEEPALIVE` | Включить keep-alive зонды |
| `SO_OOBINLINE` | Получать OOB-данные в основном потоке |
| `SO_REUSEADDR` | Разрешить повторное использование локального адреса |
| `SO_REUSEPORT` | Разрешить несколько сокетов на одном порту |
| `MSG_PEEK` | Просмотр данных без их извлечения |
| `SO_NOSIGPIPE` | *(только macOS)* Подавить SIGPIPE |

---

## Жизненный цикл сокета

### `createNativeSocket` (перегрузка с перечислениями)

```nim
proc createNativeSocket*(
    domain:      Domain   = AF_INET,
    sockType:    SockType = SOCK_STREAM,
    protocol:    Protocol = IPPROTO_TCP,
    inheritable: bool     = defined(nimInheritHandles)
): SocketHandle
```

Создаёт новый сокет ОС с использованием высокоуровневых перечислений Nim.  
Возвращает `osInvalidSocket` при ошибке — **исключение не бросается**.

`inheritable` определяет, наследуют ли дочерние процессы, запущенные через `exec`, этот файловый дескриптор. По умолчанию значение берётся из флага компиляции `-d:nimInheritHandles`.

```nim
let tcp4 = createNativeSocket()                                    # TCP/IPv4
let udp4 = createNativeSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
let tcp6 = createNativeSocket(AF_INET6, SOCK_STREAM, IPPROTO_TCP)
```

---

### `createNativeSocket` (перегрузка с сырыми cint)

```nim
proc createNativeSocket*(
    domain:      cint,
    sockType:    cint,
    protocol:    cint,
    inheritable: bool = defined(nimInheritHandles)
): SocketHandle
```

Используйте эту перегрузку, когда нужны значения домена, типа или протокола, которых нет в перечислениях Nim. Аргументы передаются напрямую в системный вызов `socket()`.

---

### `close`

```nim
proc close*(socket: SocketHandle)
```

Закрывает сокет и освобождает файловый дескриптор ОС. На Windows вызывает `closesocket`, на POSIX — `close`. Возвращаемое значение отбрасывается — если нужно проверить ошибки, вызовите `getSockOptInt` с `SO_ERROR` заранее.

```nim
close(sock)
```

---

### `setInheritable`

```nim
proc setInheritable*(s: SocketHandle, inheritable: bool): bool
```

Устанавливает, наследует ли дочерний процесс данный дескриптор сокета после `exec`. Возвращает `true` при успехе.

> **Доступность:** Функция присутствует только на платформах, поддерживающих `setInheritable` для файловых дескрипторов. Проверяйте наличие через `declared(setInheritable)`.

---

### `bindAddr`

```nim
proc bindAddr*(socket: SocketHandle, name: ptr SockAddr, namelen: SockLen): cint
```

Привязывает `socket` к адресу, заданному структурой `name`. Возвращает 0 при успехе, -1 при ошибке (проверяйте `osLastError()`). Это тонкая обёртка над системным вызовом `bind()`.

---

### `listen`

```nim
proc listen*(socket: SocketHandle, backlog = SOMAXCONN): cint
```

Помечает `socket` как пассивный — принимающий входящие соединения.  
`backlog` — максимальное число ожидающих соединений в очереди.  
Возвращает 0 при успехе, -1 при ошибке.

```nim
if listen(serverSock, 128) != 0:
  raiseOSError(osLastError())
```

---

### `accept`

```nim
proc accept*(fd: SocketHandle,
             inheritable = defined(nimInheritHandles)): (SocketHandle, string)
```

Принимает новое входящее соединение на `fd`. Возвращает кортеж из нового `SocketHandle` клиента и строки с его IP-адресом.  
При ошибке возвращает `(osInvalidSocket, "")`.

На Linux и BSD использует `accept4` с `SOCK_CLOEXEC` для атомарного создания не-наследуемого сокета.

```nim
let (clientSock, clientAddr) = accept(serverSock)
if clientSock == osInvalidSocket:
  echo "Ошибка accept"
else:
  echo "Клиент подключился с адреса ", clientAddr
```

---

## Разрешение адресов

### `getAddrInfo` (с hints-структурой `AddrInfo`)

```nim
proc getAddrInfo*(address: string, port: Port, hints: AddrInfo): ptr AddrInfo
```

Разрешает `address` и `port` через системный вызов `getaddrinfo()`. Структура `hints` управляет фильтрацией по семейству адресов, типу сокета и протоколу.

> ⚠️ **Возвращаемый `ptr AddrInfo` необходимо освободить вызовом `freeAddrInfo`.**

---

### `getAddrInfo` (перегрузка с перечислениями)

```nim
proc getAddrInfo*(
    address:  string,
    port:     Port,
    domain:   Domain   = AF_INET,
    sockType: SockType = SOCK_STREAM,
    protocol: Protocol = IPPROTO_TCP
): ptr AddrInfo
```

Удобная перегрузка — строит структуру `AddrInfo` из перечислений Nim. На IPv6 и не-BSD платформах автоматически устанавливает флаг `AI_V4MAPPED`.

> ⚠️ **Возвращаемый `ptr AddrInfo` необходимо освободить вызовом `freeAddrInfo`.**

```nim
let ai = getAddrInfo("nim-lang.org", Port(80))
defer: freeAddrInfo(ai)
# используем ai …
```

---

### `toKnownDomain`

```nim
proc toKnownDomain*(family: cint): Option[Domain]
```

Преобразует сырое платформенное значение семейства адресов (`cint`) в перечисление `Domain`, обёрнутое в `Option`. Возвращает `none(Domain)` для неизвестных семейств.

```nim
let dom = toKnownDomain(2)  # → some(AF_INET) на всех платформах
```

---

### `getServByName` *(недоступно при `-d:nimNetLite`)*

```nim
proc getServByName*(name, proto: string): Servent
```

Находит сервис по имени и протоколу (например, `"http"`, `"tcp"`). На POSIX читает `/etc/services`. Бросает `OSError` если не найдено.

```nim
let svc = getServByName("http", "tcp")
echo svc.port   # → Port(80)
```

---

### `getServByPort` *(недоступно при `-d:nimNetLite`)*

```nim
proc getServByPort*(port: Port, proto: string): Servent
```

Находит сервис по номеру порта и протоколу. Бросает `OSError` если не найдено.

```nim
let svc = getServByPort(Port(443), "tcp")
echo svc.name   # → "https"
```

---

### `getHostByName` *(недоступно при `-d:nimNetLite`)*

```nim
proc getHostByName*(name: string): Hostent
```

Разрешает имя хоста в IP-адреса. Бросает `OSError` при ошибке.

```nim
let h = getHostByName("localhost")
echo h.addrList  # → @["127.0.0.1"]
```

---

### `getHostByAddr` *(недоступно при `-d:nimNetLite`)*

```nim
proc getHostByAddr*(ip: string): Hostent
```

Выполняет обратное разрешение IP-адреса в имя хоста. Бросает `OSError` или `IOError` при ошибке.

```nim
let h = getHostByAddr("8.8.8.8")
echo h.name  # например, "dns.google"
```

---

### `getHostname` *(недоступно при `-d:nimNetLite`)*

```nim
proc getHostname*(): string
```

Возвращает имя локальной машины (не FQDN). Бросает `OSError` при ошибке.

```nim
echo getHostname()  # например, "myworkstation"
```

---

### `getProtoByName`

```nim
proc getProtoByName*(name: string): int
```

Возвращает числовой код протокола по его имени (например, `"tcp"` → `6`). Бросает `OSError` если не найдено.

```nim
echo getProtoByName("udp")  # → 17
```

---

### `makeUnixAddr` *(только POSIX, недоступно при `-d:nimNetLite`)*

```nim
proc makeUnixAddr*(path: string): Sockaddr_un
```

Создаёт структуру `Sockaddr_un` для заданного пути к Unix domain сокету. Бросает `ValueError` если путь слишком длинный.

```nim
let addr = makeUnixAddr("/tmp/my.sock")
```

---

## Преобразование порядка байт

Сетевой порядок байт — big-endian. Эти функции выполняют конвертацию между порядком байт хоста и сетевым порядком. На big-endian машинах они являются no-op.

### `ntohl`

```nim
proc ntohl*(x: uint32): uint32
```

Преобразует 32-битное значение из **сетевого** порядка байт в **хостовый**.

---

### `ntohs`

```nim
proc ntohs*(x: uint16): uint16
```

Преобразует 16-битное значение из **сетевого** порядка байт в **хостовый**.

---

### `htonl`

```nim
template htonl*(x: uint32): untyped
```

Преобразует 32-битное значение из **хостового** порядка байт в **сетевой**. Реализован как псевдоним `ntohl` (операция является собственной обратной).

---

### `htons`

```nim
template htons*(x: uint16): untyped
```

Преобразует 16-битное значение из **хостового** порядка байт в **сетевой**. Псевдоним `ntohs`.

```nim
let port = htons(8080'u16)   # → big-endian 8080
let back = ntohs(port)       # → 8080
```

---

## Параметры сокета

### `getSockOptInt`

```nim
proc getSockOptInt*(socket: SocketHandle, level, optname: int): int
```

Читает целочисленный параметр сокета. `level` — обычно `SOL_SOCKET`. Бросает `OSError` при ошибке.

```nim
let err = getSockOptInt(sock, SOL_SOCKET, SO_ERROR)
if err != 0:
  echo "Накопленная ошибка сокета: ", err
```

---

### `setSockOptInt`

```nim
proc setSockOptInt*(socket: SocketHandle, level, optname, optval: int)
```

Устанавливает целочисленный параметр сокета. Бросает `OSError` при ошибке.

```nim
setSockOptInt(sock, SOL_SOCKET, SO_REUSEADDR, 1)
setSockOptInt(sock, SOL_SOCKET, SO_KEEPALIVE, 1)
```

---

### `setBlocking`

```nim
proc setBlocking*(s: SocketHandle, blocking: bool)
```

Переключает сокет между блокирующим и неблокирующим режимом. На Windows использует `ioctlsocket(FIONBIO)`, на POSIX — `fcntl(F_SETFL, O_NONBLOCK)`. Бросает `OSError` при ошибке.

```nim
setBlocking(sock, false)   # неблокирующий (подходит для async)
setBlocking(sock, true)    # блокирующий (по умолчанию)
```

---

## Инспекция адресов

### `getSockDomain`

```nim
proc getSockDomain*(socket: SocketHandle): Domain
```

Возвращает семейство адресов (`AF_INET` или `AF_INET6`) существующего сокета через вызов `getsockname`. Бросает `OSError` или `IOError` если семейство неизвестно.

```nim
let domain = getSockDomain(sock)
```

---

### `getSockName` *(недоступно при `-d:nimNetLite`)*

```nim
proc getSockName*(socket: SocketHandle): Port
```

Возвращает номер порта, к которому в данный момент привязан сокет. Бросает `OSError` при ошибке.

```nim
let boundPort = getSockName(sock)
echo "Привязан к порту ", boundPort
```

---

### `getLocalAddr`

```nim
proc getLocalAddr*(socket: SocketHandle, domain: Domain): (string, Port)
```

Возвращает локальный IP-адрес и порт сокета в виде кортежа. Аргумент `domain` должен быть `AF_INET` или `AF_INET6`. Бросает `OSError` при ошибке.

```nim
let (ip, port) = getLocalAddr(sock, AF_INET)
echo ip, ":", port
```

---

### `getPeerAddr` *(недоступно при `-d:nimNetLite`)*

```nim
proc getPeerAddr*(socket: SocketHandle, domain: Domain): (string, Port)
```

Возвращает IP-адрес и порт удалённого узла. Аналог POSIX `getpeername`. Бросает `OSError` при ошибке.

```nim
let (peerIp, peerPort) = getPeerAddr(clientSock, AF_INET)
echo "Подключён к ", peerIp, ":", peerPort
```

---

### `getAddrString` (перегрузка с `ptr SockAddr`)

```nim
proc getAddrString*(sockAddr: ptr SockAddr): string
```

Преобразует адрес в сыром `SockAddr`-указателе в читаемую строку. Поддерживает IPv4, IPv6 и Unix domain сокеты. Бросает `OSError` или `IOError` для неизвестных семейств.

---

### `getAddrString` (перегрузка с предвыделенной строкой) *(недоступно при `-d:nimNetLite`)*

```nim
proc getAddrString*(sockAddr: ptr SockAddr, strAddress: var string)
```

Записывает строку адреса в `strAddress`. `strAddress` **должна быть предварительно инициализирована длиной 46** до вызова.

```nim
var buf = newString(46)
getAddrString(addr mySockAddr, buf)
```

---

## Готовность ввода-вывода — select

### `selectRead`

```nim
proc selectRead*(readfds: var seq[SocketHandle], timeout = 500): int
```

Ожидает, пока один или несколько сокетов из `readfds` не будут готовы к чтению. Сокеты, **не** готовые к чтению, удаляются из `readfds` на месте. Возвращает количество готовых к чтению сокетов, 0 при таймауте, -1 при ошибке.

`timeout` задаётся в **миллисекундах**. Передайте `-1` для ожидания без ограничения по времени.

```nim
var fds = @[sock1, sock2]
let n = selectRead(fds, timeout = 1000)
if n > 0:
  for fd in fds:
    echo "Готов к чтению: ", fd
```

---

### `selectWrite`

```nim
proc selectWrite*(writefds: var seq[SocketHandle], timeout = 500): int
```

Ожидает, пока один или несколько сокетов из `writefds` не будут готовы к записи. Сокеты, **не** готовые к записи, удаляются на месте. Возвращает количество готовых к записи сокетов, 0 при таймауте, -1 при ошибке.

`timeout` задаётся в **миллисекундах**. Передайте `-1` для ожидания без ограничения по времени.

```nim
var fds = @[sock]
let n = selectWrite(fds, timeout = 2000)
if n > 0:
  echo "Сокет готов к записи"
```

---

## Утилиты преобразования перечислений

### `toInt(Domain)`

```nim
proc toInt*(domain: Domain): cint
```

Преобразует значение перечисления `Domain` в нативный для платформы `cint`, необходимый при системных вызовах.

---

### `toInt(SockType)`

```nim
proc toInt*(typ: SockType): cint
```

Преобразует значение перечисления `SockType` в нативный `cint`.

---

### `toInt(Protocol)`

```nim
proc toInt*(p: Protocol): cint
```

Преобразует значение перечисления `Protocol` в нативный `cint`.

---

### `toSockType`

```nim
proc toSockType*(protocol: Protocol): SockType
```

Отображает `Protocol` на соответствующий `SockType`:
- `IPPROTO_TCP` → `SOCK_STREAM`
- `IPPROTO_UDP` → `SOCK_DGRAM`
- Все остальные → `SOCK_RAW`

```nim
echo toSockType(IPPROTO_TCP)   # → SOCK_STREAM
echo toSockType(IPPROTO_UDP)   # → SOCK_DGRAM
echo toSockType(IPPROTO_ICMP)  # → SOCK_RAW
```

---

## Платформо-зависимые особенности

| Тема | Подробности |
|---|---|
| **Windows** | `wsaStartup` вызывается автоматически при загрузке модуля. Реэкспортируются: `WSAEWOULDBLOCK`, `WSAECONNRESET`, `WSAECONNABORTED`, `WSAENETRESET`, `WSANOTINITIALISED`, `WSAENOTSOCK`, `WSAEINPROGRESS`, `WSAEINTR`, `WSAEDISCON`, `ERROR_NETNAME_DELETED`. |
| **POSIX** | Реэкспортируются: `fcntl`, `F_GETFL`, `O_NONBLOCK`, `F_SETFL`, `EAGAIN`, `EWOULDBLOCK`, `MSG_NOSIGNAL`, `EINTR`, `EINPROGRESS`, `ECONNRESET`, `EPIPE`, `ENETRESET`, `EBADF`, `Sockaddr_storage`, `Sockaddr_un`, `Sockaddr_un_path_length`. |
| **macOS** | Реэкспортируется `SO_NOSIGPIPE` для подавления `SIGPIPE` вместо `MSG_NOSIGNAL`. |
| **Linux / BSD** | В `accept` и `createNativeSocket` используется `accept4` с `SOCK_CLOEXEC` для атомарной обработки неунаследуемых сокетов. |
| **`-d:nimNetLite`** | Отключает: `getServByName`, `getServByPort`, `getHostByName`, `getHostByAddr`, `getHostname`, `getSockName`, `getPeerAddr`, `makeUnixAddr` и двухаргументный `getAddrString`. Предназначен для ресурсно ограниченных сред (FreeRTOS, Zephyr, NuttX). |
| **`AF_UNIX`** | Не поддерживается на Windows. Числовое значение `AF_INET6` зависит от ОС: Linux — 10, macOS — 30, Windows — 23. |

---

## Таблица быстрого доступа

| Символ | Вид | Описание |
|---|---|---|
| `Port` | тип | Отдельный `uint16` для номера порта |
| `Domain` | перечисление | Семейство адресов (`AF_INET`, `AF_INET6`, `AF_UNIX`, `AF_UNSPEC`) |
| `SockType` | перечисление | Тип сокета (`SOCK_STREAM`, `SOCK_DGRAM`, `SOCK_RAW`, `SOCK_SEQPACKET`) |
| `Protocol` | перечисление | Протокол (`IPPROTO_TCP`, `IPPROTO_UDP`, …) |
| `Servent` | объект | Запись базы данных сервисов |
| `Hostent` | объект | Запись базы данных хостов |
| `IPPROTO_NONE` | константа | Псевдоним `IPPROTO_IP` (для сокетов с нулевым протоколом) |
| `osInvalidSocket` | let | Значение-сторож для недействительных сокетов |
| `createNativeSocket` | проц ×2 | Создать новый сокет (перегрузки с перечислениями и cint) |
| `close` | проц | Закрыть и освободить сокет |
| `setInheritable` | проц | Управление наследованием сокета дочерними процессами |
| `bindAddr` | проц | Привязать сокет к адресу |
| `listen` | проц | Пометить сокет как принимающий соединения |
| `accept` | проц | Принять входящее соединение, вернуть дескриптор и IP клиента |
| `getAddrInfo` | проц ×2 | Разрешить имя хоста + порт в `AddrInfo` (нужно освободить!) |
| `toKnownDomain` | проц | Преобразовать сырой `cint` в `Option[Domain]` |
| `getServByName` | проц | Поиск сервиса по имени и протоколу |
| `getServByPort` | проц | Поиск сервиса по порту и протоколу |
| `getHostByName` | проц | Разрешить имя хоста в `Hostent` |
| `getHostByAddr` | проц | Обратное разрешение IP в `Hostent` |
| `getHostname` | проц | Имя локальной машины |
| `getProtoByName` | проц | Числовой код протокола по имени |
| `makeUnixAddr` | проц | Построить `Sockaddr_un` для пути к Unix-сокету |
| `ntohl` | проц | Сетевой→хостовый порядок байт, 32 бита |
| `ntohs` | проц | Сетевой→хостовый порядок байт, 16 бит |
| `htonl` | шаблон | Хостовый→сетевой порядок байт, 32 бита |
| `htons` | шаблон | Хостовый→сетевой порядок байт, 16 бит |
| `getSockOptInt` | проц | Прочитать целочисленный параметр сокета |
| `setSockOptInt` | проц | Записать целочисленный параметр сокета |
| `setBlocking` | проц | Переключить блокирующий/неблокирующий режим |
| `getSockDomain` | проц | Получить семейство адресов сокета |
| `getSockName` | проц | Получить номер порта привязки |
| `getLocalAddr` | проц | Получить кортеж (локальный адрес, порт) |
| `getPeerAddr` | проц | Получить кортеж (адрес, порт) удалённого узла |
| `getAddrString` | проц ×2 | Преобразовать `SockAddr*` в строку |
| `selectRead` | проц | Ожидать сокеты, готовые к чтению (обёртка select) |
| `selectWrite` | проц | Ожидать сокеты, готовые к записи (обёртка select) |
| `toInt` | проц ×3 | Преобразовать `Domain`/`SockType`/`Protocol` в `cint` |
| `toSockType` | проц | Отобразить `Protocol` на `SockType` |
