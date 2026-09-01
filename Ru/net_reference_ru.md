# net — справочник модуля

> **Импорт:** `import std/net`
> **Область применения:** высокоуровневый кроссплатформенный интерфейс к сокетам (TCP, UDP, Unix-сокеты) для блокирующего ввода-вывода, включая работу с IP-адресами и опциональную поддержку SSL/TLS.

Модуль решает три связанные задачи: разбор и представление IP-адресов через тип `IpAddress`, создание и настройку сокетов через тип `Socket` (создание, `connect`/`bindAddr`/`listen`/`accept`, чтение и запись данных, буферизация), и — при компиляции с флагом `-d:ssl` — оборачивание уже созданных сокетов в защищённое SSL/TLS-соединение через `SslContext`.
Общая конвенция модуля: большинство операций поднимает `OSError` при системной ошибке сокета и `TimeoutError`, если явно передан параметр `timeout` и операция не уложилась в отведённое время; блокирующая природа модуля означает, что каждый вызов вроде `recv` или `accept` может "зависнуть" до наступления события, если таймаут не указан.
Для неблокирующих сокетов существует отдельный модуль `std/asyncnet`, работающий поверх `std/asyncdispatch` — этот справочник его не описывает.

---

## Оглавление

I. [Типы и константы](#типы-и-константы)
   1. [`IpAddress` и `IpAddressFamily`](#ipaddress-и-ipaddressfamily)
   2. [`Socket`](#socket)
   3. [`SocketFlag`, `SOBool`, `ReadLineResult`](#socketflag-sobool-readlineresult)
   4. [`TimeoutError`, `SslError`](#timeouterror-sslerror)
II. [Создание сокетов и установление соединения](#создание-сокетов-и-установление-соединения)
   1. [`newSocket`](#newsocket)
   2. [`dial`](#dial)
   3. [`connect` (по имени хоста)](#connect-по-имени-хоста)
   4. [`connect` (с таймаутом)](#connect-с-таймаутом)
III. [Работа с IP-адресами](#работа-с-ip-адресами)
   1. [`parseIpAddress`](#parseipaddress)
   2. [`isIpAddress`](#isipaddress)
   3. [`` `$` `` для `IpAddress`](#-для-ipaddress)
   4. [`` `==` `` для `IpAddress`](#-для-ipaddress-1)
   5. [`IPv4_any`, `IPv4_loopback`, `IPv4_broadcast`, `IPv6_any`, `IPv6_loopback`](#ipv4_any-ipv4_loopback-ipv4_broadcast-ipv6_any-ipv6_loopback)
   6. [`getPrimaryIPAddr`](#getprimaryipaddr)
   7. [`toSockAddr` и `fromSockAddr`](#tosockaddr-и-fromsockaddr)
IV. [Серверные сокеты](#серверные-сокеты)
   1. [`bindAddr`](#bindaddr)
   2. [`listen`](#listen)
   3. [`acceptAddr`](#acceptaddr)
   4. [`accept`](#accept)
   5. [`close`](#close)
V. [Отправка и приём данных](#отправка-и-приём-данных)
   1. [`send`](#send)
   2. [`` `&=` ``и `trySend`](#-и-trysend)
   3. [`sendTo`](#sendto)
   4. [`recv`](#recv)
   5. [`recvFrom`](#recvfrom)
   6. [`readLine` и `recvLine`](#readline-и-recvline)
   7. [`skip` и `hasDataBuffered`](#skip-и-hasdatabuffered)
VI. [Параметры и диагностика сокета](#параметры-и-диагностика-сокета)
   1. [`getSockOpt` и `setSockOpt`](#getsockopt-и-setsockopt)
   2. [`getLocalAddr` и `getPeerAddr`](#getlocaladdr-и-getpeeraddr)
   3. [`getFd` и `isSsl`](#getfd-и-issl)
   4. [`socketError`, `getSocketError`, `isDisconnectionError`](#socketerror-getsocketerror-isdisconnectionerror)
VII. [SSL/TLS](#ssltls)
   1. [`newContext`](#newcontext)
   2. [`wrapSocket`](#wrapsocket)
   3. [`wrapConnectedSocket`](#wrapconnectedsocket)
   4. [`getPeerCertificates`](#getpeercertificates)
   5. [`destroyContext`](#destroycontext)
VIII. [Практические рецепты](#практические-рецепты)
   1. [Простой TCP-клиент со строковым протоколом](#простой-tcp-клиент-со-строковым-протоколом)
   2. [Эхо-сервер с обработкой отключений](#эхо-сервер-с-обработкой-отключений)
   3. [UDP-обмен датаграммами](#udp-обмен-датаграммами)
   4. [Подключение с ограничением по времени](#подключение-с-ограничением-по-времени)
   5. [Определение собственного внешнего адреса](#определение-собственного-внешнего-адреса)
   6. [Клиент с TLS и проверкой сертификата](#клиент-с-tls-и-проверкой-сертификата)
IX. [Краткая таблица](#краткая-таблица)
X. [Сводка: какую процедуру выбрать](#сводка-какую-процедуру-выбрать)

---

## Типы и константы

### `IpAddress` и `IpAddressFamily`

```nim
type
  IpAddressFamily* {.pure.} = enum
    IPv6, IPv4

  IpAddress* = object
    case family*: IpAddressFamily
    of IpAddressFamily.IPv6:
      address_v6*: array[0..15, uint8]
    of IpAddressFamily.IPv4:
      address_v4*: array[0..3, uint8]
```

Что делает: `IpAddress` — это объект-объединение (`case`-объект), который хранит один и тот же логический адрес двумя разными способами в зависимости от значения дискриминанта `family`.
Если `family` равен `IPv4`, реально существует и доступно только поле `address_v4` — массив из 4 байт; при `IPv6` — только `address_v6`, массив из 16 байт.
Это экономит память (не нужно всегда держать 16 байт даже для IPv4) и одновременно защищает от ошибок: попытка прочитать `address_v6` у адреса с `family == IPv4` вызовет `FieldDefect` во время выполнения.

Список параметров:

- `family: IpAddressFamily` — дискриминант, определяет, какое из полей активно; значения `IPv4` или `IPv6`.
- `address_v4: array[0..3, uint8]` — доступно только при `family == IPv4`; байты адреса в сетевом порядке (старший байт первым).
- `address_v6: array[0..15, uint8]` — доступно только при `family == IPv6`; 16 байт адреса в сетевом порядке.

Пример:

```nim
import std/net

let addr4 = IpAddress(family: IpAddressFamily.IPv4, address_v4: [127'u8, 0, 0, 1])
echo addr4 # выводит 127.0.0.1

let addr6 = parseIpAddress("::1")
echo addr6.family # выводит IPv6
```

---

### `Socket`

```nim
type
  Socket* = ref SocketImpl
```

Что делает: `Socket` — это ссылочный (`ref`) объект, представляющий один сетевой сокет операционной системы вместе с сопутствующим состоянием: файловым дескриптором, флагом буферизации, внутренним буфером на 4000 байт (`BufferSize`), позицией чтения в этом буфере, доменом (`AF_INET`/`AF_INET6`), типом (`SOCK_STREAM`/`SOCK_DGRAM`) и протоколом, а также (если модуль скомпилирован с `-d:ssl`) состоянием TLS-сессии.
Поля объекта не экспортированы напрямую — доступ к ним даётся через процедуры модуля (`getFd`, `isSsl`, `hasDataBuffered` и т.д.), сам объект создаётся исключительно через `newSocket` или `dial`, а не через прямую конструкцию.
Буферизация (по умолчанию включена) означает, что `recv` читает данные из ОС крупными кусками до `BufferSize` байт и хранит их во внутреннем буфере — это уменьшает число системных вызовов при частом мелком чтении (например, построчном).

Список параметров: у самого типа параметров нет — конфигурация задаётся при создании через `newSocket`.

Пример:

```nim
let socket = newSocket() # TCP-сокет по умолчанию: AF_INET, SOCK_STREAM, IPPROTO_TCP
echo isSsl(socket) # выводит false
close(socket)
```

---

### `SocketFlag`, `SOBool`, `ReadLineResult`

```nim
type
  SocketFlag* {.pure.} = enum
    Peek, SafeDisconn

  SOBool* = enum
    OptAcceptConn, OptBroadcast, OptDebug, OptDontRoute, OptKeepAlive,
    OptOOBInline, OptReuseAddr, OptReusePort, OptNoDelay
```

Что делает: `SocketFlag` — это множество флагов, передаваемых в операции чтения/записи (`recv`, `send`, `accept` и т.п.); `Peek` означает "прочитать данные, но не убирать их из очереди сокета", а `SafeDisconn` — "не поднимать исключение при типичных ошибках разрыва соединения (ECONNRESET, EPIPE и т.п.), а просто считать это нормальным завершением".
`SOBool` перечисляет булевы опции сокета уровня ОС, которые можно читать и менять через `getSockOpt`/`setSockOpt` — например, `OptReuseAddr` разрешает повторно занять адрес сразу после закрытия предыдущего сокета на нём, а `OptNoDelay` отключает алгоритм Нейгла (буферизацию мелких TCP-пакетов).
`ReadLineResult` используется асинхронным аналогом чтения строк и в этом справочнике не рассматривается подробно, так как относится к неблокирующему API.

Список параметров: перечисляемые типы, параметров как таковых нет — используются как значения.

Пример:

```nim
var socket = newSocket()
setSockOpt(socket, OptReuseAddr, true)
setSockOpt(socket, OptNoDelay, true, level = IPPROTO_TCP.cint)
close(socket)
```

---

### `TimeoutError`, `SslError`

```nim
type
  TimeoutError* = object of CatchableError
```

Что делает: `TimeoutError` поднимается любой процедурой, принимающей параметр `timeout` (в миллисекундах), если операция не завершилась в отведённое время — например, `connect(socket, address, port, timeout)` или `recv(socket, data, size, timeout)`.
`SslError` (доступен только при `-d:ssl`) поднимается при ошибках уровня TLS: неудачном рукопожатии, несовпадении имени сертификата, ошибках библиотеки OpenSSL.
Оба типа — обычные `CatchableError`, их можно ловить блоком `try`/`except` как любое другое исключение Nim.

Пример:

```nim
let socket = newSocket()
try:
  connect(socket, "example.invalid", Port(81), timeout = 200)
except TimeoutError:
  echo "Не удалось подключиться за отведённое время" # выводит это сообщение при таймауте
close(socket)
```

---

## Создание сокетов и установление соединения

### `newSocket`

```nim
proc newSocket*(domain: Domain = AF_INET, sockType: SockType = SOCK_STREAM,
                protocol: Protocol = IPPROTO_TCP, buffered = true,
                inheritable = defined(nimInheritHandles)): owned(Socket)
proc newSocket*(fd: SocketHandle, domain: Domain = AF_INET,
                sockType: SockType = SOCK_STREAM,
                protocol: Protocol = IPPROTO_TCP, buffered = true): owned(Socket)
```

Что делает: создаёт новый объект `Socket`, обёртывающий сокет операционной системы указанного семейства адресов, типа и протокола.
Вариант без `fd` сам просит ОС выделить новый сокет-дескриптор (через `createNativeSocket`) и поднимает `OSError`, если система отказала (например, кончились дескрипторы); вариант с `fd` оборачивает уже существующий, ранее созданный вручную дескриптор — этим пользуются `dial` и `acceptAddr` внутри модуля.
По умолчанию создаётся TCP-сокет (`SOCK_STREAM`/`IPPROTO_TCP`) для IPv4 (`AF_INET`); для UDP нужно явно указать `SOCK_DGRAM` и `IPPROTO_UDP`.
Параметр `inheritable` контролирует, будет ли дескриптор унаследован дочерними процессами — по умолчанию нет, что предотвращает случайную утечку открытых сокетов в подпроцессы.

Список параметров:

- `domain: Domain` — семейство адресов, `AF_INET` (IPv4) или `AF_INET6` (IPv6).
- `sockType: SockType` — тип сокета: `SOCK_STREAM` (TCP, потоковый) или `SOCK_DGRAM` (UDP, датаграммный).
- `protocol: Protocol` — конкретный протокол, обычно `IPPROTO_TCP` или `IPPROTO_UDP`.
- `buffered: bool` — включает внутреннюю буферизацию чтения (см. раздел про `Socket`); по умолчанию `true`.
- `inheritable: bool` — наследуется ли дескриптор дочерними процессами.
- `fd: SocketHandle` (только в первом варианте) — уже существующий дескриптор сокета для оборачивания.

Примеры:

```nim
let tcpSocket = newSocket() # TCP-сокет по умолчанию
let udpSocket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP) # UDP-сокет
let ipv6Socket = newSocket(AF_INET6, SOCK_STREAM, IPPROTO_TCP) # TCP поверх IPv6
close(tcpSocket)
close(udpSocket)
close(ipv6Socket)
```

---

### `dial`

```nim
proc dial*(address: string, port: Port, protocol = IPPROTO_TCP,
           buffered = true): owned(Socket)
```

Что делает: одновременно создаёт сокет и подключает его к `address`:`port`, перебирая все адреса, в которые разрешается имя хоста (как IPv4, так и IPv6), пока подключение не удастся.
Это удобнее, чем `newSocket` + `connect`, потому что не нужно заранее знать, к какому семейству адресов относится хост — `dial` сам определяет подходящий домен по результату DNS-разрешения.

Разбор реализации: внутри процедура запрашивает список адресов через `getAddrInfo` с доменом `AF_UNSPEC` (то есть "любой"), затем идёт по списку и для каждого встреченного семейства адресов создаёт (при необходимости) отдельный сокет и пробует `connect`; при первом успехе — прекращает перебор и закрывает лишние, ранее созданные для других семейств сокеты, а при исчерпании списка без единого успеха поднимает `OSError` с последней увиденной ошибкой (или `IOError`, если имя вообще не разрешилось).
Такой перебор с закрытием "лишних" гнёзд гарантирует, что при выходе из процедуры остаётся ровно один открытый дескриптор.

Список параметров:

- `address: string` — доменное имя или IP-адрес.
- `port: Port` — порт назначения.
- `protocol: Protocol` — протокол транспорта, по умолчанию `IPPROTO_TCP`.
- `buffered: bool` — включить буферизацию созданного сокета.

Примеры:

```nim
let socket = dial("example.com", Port(80)) # TCP-подключение, IPv4 или IPv6 — как получится
send(socket, "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
echo recv(socket, 15) # выводит первые 15 байт ответа сервера, например "HTTP/1.1 200 OK"
close(socket)

doAssertRaises(IOError):
  discard dial("no-such-host.invalid", Port(80)) # выводит ошибку: имя не резолвится
```

---

### `connect` (по имени хоста)

```nim
proc connect*(socket: Socket, address: string, port = Port(0))
```

Что делает: подключает уже созданный (через `newSocket`) сокет к серверу по `address` (IP-адрес или доменное имя) и `port`; если `address` — имя хоста, процедура перебирает все адреса, в которые оно резолвится, пока одно из подключений не увенчается успехом.
Если сокет был предварительно обёрнут в SSL через `wrapSocket`, после установления TCP-соединения автоматически выполняется TLS-рукопожатие и (если проверка сертификатов не отключена) сверка имени хоста с сертификатом сервера.

Список параметров:

- `socket: Socket` — заранее созданный, ещё не подключённый сокет.
- `address: string` — IP-адрес или доменное имя сервера.
- `port: Port` — порт сервера.

Примеры:

```nim
let socket = newSocket()
connect(socket, "example.com", Port(80)) # обычное блокирующее подключение
close(socket)

let badSocket = newSocket()
doAssertRaises(OSError):
  connect(badSocket, "127.0.0.1", Port(1)) # порт 1 обычно ничего не слушает
close(badSocket)
```

---

### `connect` (с таймаутом)

```nim
proc connect*(socket: Socket, address: string, port = Port(0), timeout: int)
```

Что делает: тот же `connect`, но с ограничением по времени в миллисекундах — если подключение не установилось за `timeout` мс, поднимается `TimeoutError` вместо бесконечного ожидания.

Разбор реализации: процедура временно переводит сокет в неблокирующий режим (`setBlocking(false)`), инициирует подключение через внутреннюю `connectAsync` (она возвращается сразу же, не дожидаясь результата), а затем ждёт готовности сокета к записи через `select`/`poll`-подобный механизм (`timeoutWrite`) — именно готовность к записи на POSIX-системах означает, что TCP-рукопожатие завершилось (успешно или с ошибкой).
После этого сокет возвращается в блокирующий режим и, если это SSL-сокет, синхронно выполняется TLS-рукопожатие — но обратите внимание, что само рукопожатие таймаутом уже не ограничено.

Список параметров:

- `socket`, `address`, `port` — как у обычного `connect`.
- `timeout: int` — время ожидания установления TCP-соединения в миллисекундах.

Примеры:

```nim
let socket = newSocket()
connect(socket, "example.com", Port(80), timeout = 3000) # ждём не более 3 секунд
close(socket)

let unreachable = newSocket()
doAssertRaises(TimeoutError):
  connect(unreachable, "10.255.255.1", Port(80), timeout = 200) # адрес из "чёрной дыры"
close(unreachable)
```

---

## Работа с IP-адресами

### `parseIpAddress`

```nim
proc parseIpAddress*(addressStr: string): IpAddress
```

Что делает: разбирает строку в объект `IpAddress`, автоматически определяя, IPv4 перед нами или IPv6 (по наличию символа `:` в строке), и поднимает `ValueError`, если строка не является корректным адресом.
Для IPv4 принимается только строгая десятичная форма без ведущих нулей (`192.168.0.1`, но не `192.168.00.1`) — так модуль защищается от неоднозначности, вызванной историческим восьмеричным толкованием чисел с ведущим нулём.
Для IPv6 поддерживается сокращение `::` (не более одного на адрес) и смешанная запись с "хвостом" в виде IPv4-адреса (`::ffff:192.0.2.1`).

Разбор реализации: парсинг IPv6 — это конечный автомат, который идёт по строке символ за символом, накапливая 16-битные группы; при встрече `::` запоминается номер текущей группы (`dualColonGroup`), а в конце недостающие группы "раздвигаются" и заполняются нулями — это позволяет корректно восстановить полный 128-битный адрес независимо от того, где именно стоит сокращение.

Список параметров:

- `addressStr: string` — строка с адресом; не должна быть пустой.

Примеры:

```nim
let ipv4 = parseIpAddress("192.168.1.10")
echo ipv4 # выводит 192.168.1.10

let ipv6 = parseIpAddress("2001:db8::1")
echo ipv6 # выводит 2001:db8::1

let mixed = parseIpAddress("::ffff:192.0.2.128") # IPv6 с "вложенным" IPv4-хвостом
echo mixed.family # выводит IPv6

doAssertRaises(ValueError):
  discard parseIpAddress("192.168.01.1") # ведущий ноль запрещён

doAssertRaises(ValueError):
  discard parseIpAddress("") # пустая строка недопустима
```

---

### `isIpAddress`

```nim
proc isIpAddress*(addressStr: string): bool
```

Что делает: проверяет, является ли строка корректным IP-адресом, не поднимая исключений — по сути, это `parseIpAddress`, обёрнутый в `try`/`except ValueError`, возвращающий `true`/`false` вместо результата или ошибки.
Полезно там, где нужно отличить "это уже IP-адрес" от "это доменное имя, которое нужно резолвить" — именно так модуль сам решает, нужно ли передавать SNI-имя при TLS-рукопожатии.

Список параметров:

- `addressStr: string` — проверяемая строка.

Примеры:

```nim
echo isIpAddress("127.0.0.1") # выводит true
echo isIpAddress("example.com") # выводит false
echo isIpAddress("") # выводит false
```

---

### `` `$` `` для `IpAddress`

```nim
proc `$`*(address: IpAddress): string
```

Что делает: преобразует `IpAddress` в его каноническую строковую форму — точечно-десятичную для IPv4 и сокращённую с `::` для IPv6 (там, где это уместно, самая длинная последовательность нулевых групп схлопывается).

Список параметров:

- `address: IpAddress` — адрес для преобразования.

Примеры:

```nim
let a = parseIpAddress("2001:0db8:0000:0000:0000:0000:0000:0001")
echo a # выводит 2001:db8::1 — нулевые группы схлопнуты в "::"

let b = parseIpAddress("10.0.0.1")
echo $b # выводит 10.0.0.1
```

---

### `` `==` `` для `IpAddress`

```nim
proc `==`*(lhs, rhs: IpAddress): bool
```

Что делает: сравнивает два адреса на равенство; сначала сравнивается `family` (адреса из разных семейств никогда не равны, даже если один из них является IPv4-представлением другого в IPv6), а затем — побайтно соответствующий массив (`address_v4` или `address_v6`).

Список параметров:

- `lhs, rhs: IpAddress` — сравниваемые адреса.

Примеры:

```nim
echo parseIpAddress("127.0.0.1") == parseIpAddress("127.0.0.1") # выводит true
echo parseIpAddress("127.0.0.1") == parseIpAddress("::1") # выводит false — разные семейства
echo IPv4_loopback() == parseIpAddress("127.0.0.1") # выводит true
```

---

### `IPv4_any`, `IPv4_loopback`, `IPv4_broadcast`, `IPv6_any`, `IPv6_loopback`

```nim
proc IPv4_any*(): IpAddress
proc IPv4_loopback*(): IpAddress
proc IPv4_broadcast*(): IpAddress
proc IPv6_any*(): IpAddress
proc IPv6_loopback*(): IpAddress
```

Что делает: пять процедур-фабрик, возвращающих часто используемые "особые" адреса, чтобы не писать их строковые представления вручную и не рисковать опечаткой: `IPv4_any` — `0.0.0.0` (означает "любой локальный интерфейс" при `bindAddr`), `IPv4_loopback` — `127.0.0.1`, `IPv4_broadcast` — `255.255.255.255`, `IPv6_any` — `::`, `IPv6_loopback` — `::1`.

Список параметров: без параметров.

Примеры:

```nim
echo IPv4_any() # выводит 0.0.0.0
echo IPv4_loopback() # выводит 127.0.0.1
echo IPv4_broadcast() # выводит 255.255.255.255
echo IPv6_loopback() # выводит ::1
```

---

### `getPrimaryIPAddr`

```nim
proc getPrimaryIPAddr*(dest = parseIpAddress("8.8.8.8")): IpAddress
```

Что делает: определяет, какой из локальных сетевых адресов машины (обычно принадлежащий Ethernet- или Wi-Fi-интерфейсу) операционная система выбрала бы для отправки трафика в сторону `dest`.
Функция не отправляет ни единого пакета по сети: она создаёт UDP-сокет, "подключает" его (для UDP это лишь фиксирует адрес назначения на уровне ядра, не порождая сетевого обмена) и спрашивает у ОС локальный адрес этого сокета через `getLocalAddr` — именно так операционная система "выбирает маршрут", не пересылая данные.
Поднимает `OSError`, если внешняя сеть не настроена (например, нет маршрута по умолчанию).

Список параметров:

- `dest: IpAddress` — адрес, в сторону которого нужно определить исходящий интерфейс; по умолчанию публичный DNS Google (`8.8.8.8`).

Примеры:

```nim
echo getPrimaryIPAddr() # выводит, например, 192.168.1.42
echo getPrimaryIPAddr(parseIpAddress("2001:4860:4860::8888")) # то же самое, но для IPv6-маршрута
```

---

### `toSockAddr` и `fromSockAddr`

```nim
proc toSockAddr*(address: IpAddress, port: Port, sa: var Sockaddr_storage, sl: var SockLen)
proc fromSockAddr*(sa: Sockaddr_storage | SockAddr | Sockaddr_in | Sockaddr_in6,
                    sl: SockLen, address: var IpAddress, port: var Port)
```

Что делает: пара процедур низкого уровня для перехода между высокоуровневым представлением адреса (`IpAddress` + `Port`) и низкоуровневой структурой ОС `Sockaddr_storage`, которую напрямую используют системные вызовы POSIX/Winsock (`bind`, `connect`, `recvfrom` и т.п.).
Большинству пользователей модуля эти процедуры не нужны напрямую — они работают "под капотом" у `recvFrom`, `bindAddr`, `connect`; они пригодятся, только если нужно вручную взаимодействовать с системными вызовами сокетов через FFI.
`fromSockAddr` поднимает `ObjectConversionDefect` (документированное как "не IPv4 и не IPv6"), если переданная структура не соответствует ни одному из известных семейств адресов.

Список параметров:

- `address: IpAddress` (в `toSockAddr`) — исходный адрес для преобразования.
- `port: Port` — порт (в `toSockAddr` — исходный, в `fromSockAddr` — куда записать результат).
- `sa: var Sockaddr_storage` — буфер для записи (в `toSockAddr`) или источник для чтения (в `fromSockAddr`).
- `sl: var SockLen` — фактический размер заполненной структуры.

Примеры:

```nim
import std/nativesockets

var storage: Sockaddr_storage
var length: SockLen
toSockAddr(parseIpAddress("192.168.1.1"), Port(8080), storage, length)
echo length # выводит размер структуры sockaddr_in в байтах

var restoredAddress: IpAddress
var restoredPort: Port
fromSockAddr(storage, length, restoredAddress, restoredPort)
echo restoredAddress # выводит 192.168.1.1
echo restoredPort # выводит 8080
```

---

## Серверные сокеты

### `bindAddr`

```nim
proc bindAddr*(socket: Socket, port = Port(0), address = "")
```

Что делает: связывает сокет с локальным адресом и портом, на котором сокет затем будет либо слушать входящие соединения (`listen`/`accept`), либо принимать датаграммы (UDP).
Если `address` не задан (пустая строка), выбирается адрес "любой интерфейс" — `0.0.0.0` для IPv4-сокета или `::` для IPv6-сокета, в зависимости от домена, с которым был создан сокет.
Если `port` равен `Port(0)`, операционная система сама назначит свободный порт — его можно потом узнать через `getLocalAddr`.

Список параметров:

- `socket: Socket` — сокет для привязки.
- `port: Port` — локальный порт; `0` означает "выбрать автоматически".
- `address: string` — локальный адрес; пустая строка означает "любой интерфейс".

Примеры:

```nim
let server = newSocket()
setSockOpt(server, OptReuseAddr, true)
bindAddr(server, Port(8080)) # слушаем на всех интерфейсах, порт 8080
listen(server)
close(server)

let autoPortSocket = newSocket()
bindAddr(autoPortSocket, Port(0)) # порт выбирает ОС
echo getLocalAddr(autoPortSocket)[1] # выводит фактически выданный порт, например 54231
close(autoPortSocket)
```

---

### `listen`

```nim
proc listen*(socket: Socket, backlog = SOMAXCONN)
```

Что делает: переводит уже привязанный (`bindAddr`) TCP-сокет в режим ожидания входящих соединений, после чего к нему можно применять `accept`/`acceptAddr`.
Параметр `backlog` задаёт максимальную длину очереди уже установленных, но ещё не принятых прикладным кодом соединений; при переполнении очереди новые попытки подключения клиентов будут отклоняться операционной системой.

Список параметров:

- `socket: Socket` — привязанный сокет.
- `backlog: int` — максимальная длина очереди ожидающих `accept` соединений, по умолчанию `SOMAXCONN` (системный максимум).

Примеры:

```nim
let server = newSocket()
setSockOpt(server, OptReuseAddr, true)
bindAddr(server, Port(9000))
listen(server, backlog = 10) # не более 10 соединений в очереди одновременно
close(server)
```

---

### `acceptAddr`

```nim
proc acceptAddr*(server: Socket, client: var owned(Socket), address: var string,
                  flags = {SocketFlag.SafeDisconn},
                  inheritable = defined(nimInheritHandles))
```

Что делает: блокируется до тех пор, пока не подключится клиент, затем заполняет `client` новым сокетом этого соединения и `address` — строковым адресом клиента.
Новый клиентский сокет наследует настройку буферизации от `server`; если серверный сокет был обёрнут в SSL (`wrapSocket`), клиентский сокет автоматически оборачивается тем же `SslContext` и для него сразу выполняется серверная часть TLS-рукопожатия (`SSL_accept`).
Если во время `accept` клиент успел отключиться (гонка между установлением и разрывом TCP-соединения) и в `flags` присутствует `SafeDisconn`, процедура не поднимает исключение, а просто повторяет попытку — иначе поднимается `OSError`.

Список параметров:

- `server: Socket` — слушающий сокет.
- `client: var Socket` — принимающая переменная; если содержит `nil`, будет создан новый объект.
- `address: var string` — принимающая переменная для адреса клиента.
- `flags: set[SocketFlag]` — набор флагов; по умолчанию только `SafeDisconn`.
- `inheritable: bool` — наследуется ли дескриптор клиента дочерними процессами.

Примеры:

```nim
let server = newSocket()
setSockOpt(server, OptReuseAddr, true)
bindAddr(server, Port(9001))
listen(server)

var client: Socket
var clientAddress = ""
acceptAddr(server, client, clientAddress) # блокируется до подключения клиента
echo clientAddress # выводит адрес подключившегося клиента, например 127.0.0.1
close(client)
close(server)
```

---

### `accept`

```nim
proc accept*(server: Socket, client: var owned(Socket),
             flags = {SocketFlag.SafeDisconn},
             inheritable = defined(nimInheritHandles))
```

Что делает: то же самое, что и `acceptAddr`, но не сообщает адрес подключившегося клиента — используется, когда адрес клиента не нужен приложению.

Разбор реализации: процедура — это тонкая обёртка над `acceptAddr` с фиктивной локальной переменной под адрес, которая затем просто отбрасывается.

Список параметров: те же, что у `acceptAddr`, за вычетом `address`.

Примеры:

```nim
let server = newSocket()
setSockOpt(server, OptReuseAddr, true)
bindAddr(server, Port(9002))
listen(server)

var client: Socket
accept(server, client) # адрес клиента приложению не важен
send(client, "привет\n")
close(client)
close(server)
```

---

### `close`

```nim
proc close*(socket: Socket, flags = {SocketFlag.SafeDisconn})
```

Что делает: закрывает сокет и освобождает связанный с ним дескриптор операционной системы; для SSL-сокетов дополнительно отправляет пиру уведомление о закрытии TLS-сессии ("close_notify"), а если это не удаётся из-за уже разорванного соединения и указан флаг `SafeDisconn`, ошибка молча игнорируется, а не поднимается как исключение.
После вызова `close` объект `Socket` больше не должен использоваться для операций ввода-вывода.

Список параметров:

- `socket: Socket` — закрываемый сокет.
- `flags: set[SocketFlag]` — влияет на поведение при ошибках отправки TLS-уведомления о закрытии.

Примеры:

```nim
let socket = newSocket()
connect(socket, "example.com", Port(80))
close(socket) # штатное закрытие после использования
```

---

## Отправка и приём данных

### `send`

```nim
proc send*(socket: Socket, data: string, flags = {SocketFlag.SafeDisconn}, maxRetries = 100)
```

Что делает: отправляет строку `data` целиком, при необходимости выполняя несколько системных вызовов `send`, если операционная система за один раз согласилась принять не все байты, а также автоматически повторяя попытку при прерывании сигналом или временной невозможности записи (до `maxRetries` раз).
В отличие от низкоуровневого варианта `send(socket, data: pointer, size: int)` (используется редко, напрямую с указателем на память), эта версия гарантированно либо отправит все байты, либо поднимет исключение — частичная отправка "молча" не происходит.

Список параметров:

- `socket: Socket` — сокет для отправки; должен быть открыт.
- `data: string` — отправляемые данные.
- `flags: set[SocketFlag]` — влияет на трактовку ошибок разрыва соединения.
- `maxRetries: int` — сколько раз повторять попытку записи при временной блокировке ввода-вывода.

Примеры:

```nim
let socket = newSocket()
connect(socket, "example.com", Port(80))
send(socket, "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n") # весь запрос уходит одним вызовом
close(socket)
```

---

### `` `&=` ``и `trySend`

```nim
template `&=`*(socket: Socket; data: typed)
proc trySend*(socket: Socket, data: string): bool
```

Что делает: `&=` — это просто более короткий синоним для `send`, читаемый как "дописать в сокет"; `trySend` решает ту же задачу отправки строки, но вместо поднятия `OSError` при ошибке возвращает `false` — удобно там, где отдельная ошибка сети не должна прерывать выполнение всей программы.

Список параметров: те же `socket` и `data`, что у `send`.

Примеры:

```nim
let socket = newSocket()
connect(socket, "example.com", Port(80))
socket &= "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n" # то же самое, что send(socket, ...)

if not trySend(socket, "ещё данные"):
  echo "отправка не удалась" # выводится, если сокет уже разорван
close(socket)
```

---

### `sendTo`

```nim
proc sendTo*(socket: Socket, address: string, port: Port, data: string): int
proc sendTo*(socket: Socket, address: IpAddress, port: Port, data: pointer, size: int): int
```

Что делает: отправляет датаграмму по адресу `address`:`port`, не требуя предварительного `connect` — используется с UDP-сокетами (`SOCK_DGRAM`), где каждая датаграмма самостоятельно адресуется.
Строковый вариант принимает адрес как есть (может потребовать разрешения имени внутри), вариант с `IpAddress` работает с уже разобранным адресом и указателем на сырые данные — это быстрее при частой отправке, так как не нужно повторно резолвить адрес на каждый вызов.

Список параметров:

- `socket: Socket` — обычно UDP-сокет.
- `address: string | IpAddress` — адрес получателя.
- `port: Port` — порт получателя.
- `data: string | pointer` — отправляемые данные.
- `size: int` (для варианта с `pointer`) — размер данных в байтах.

Примеры:

```nim
let udpSocket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
discard sendTo(udpSocket, "192.168.0.1", Port(27960), "status\n") # без connect
close(udpSocket)

let ip = parseIpAddress("192.168.0.1")
let udpSocket2 = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
let message = "status\c\l"
let sent = sendTo(udpSocket2, ip, Port(27960), unsafeAddr message[0], message.len)
echo sent # выводит количество отправленных байт, например 8
close(udpSocket2)
```

---

### `recv`

```nim
proc recv*(socket: Socket, size: int, timeout = -1, flags = {SocketFlag.SafeDisconn}): string
proc recv*(socket: Socket, data: var string, size: int, timeout = -1,
           flags = {SocketFlag.SafeDisconn}): int
```

Что делает: читает **до** `size` байт из сокета. Есть два принципиально разных случая поведения, о которых важно помнить на границах: для буферизованных сокетов (по умолчанию) процедура старается дочитать именно `size` байт, обращаясь к ОС порциями по `BufferSize`; для небуферизованных — возвращает ровно столько, сколько отдала ОС за один системный вызов, даже если это меньше запрошенного.
Возврат `""` (или `0` для варианта с явным буфером) означает, что удалённая сторона закрыла соединение — это не ошибка, а нормальный сигнал конца потока; настоящая ошибка сети поднимает `OSError`.
Если указан `timeout` и данные не появились за отведённое время — поднимается `TimeoutError`.

Разбор реализации: процедура — диспетчер по трём веткам в зависимости от состояния сокета: (1) буферизованный сокет читает из внутреннего 4000-байтного буфера, дозаполняя его через ОС только когда буфер исчерпан; (2) SSL-сокет читает через `SSL_read`, с отдельной обработкой единственного "подсмотренного вперёд" байта (`sslPeekChar`), который мог остаться там после операции `peekChar`, использующейся, например, внутри `readLine` для распознавания `\r\n`; (3) обычный небуферизованный сокет вызывает системный `recv` напрямую.

Список параметров:

- `socket: Socket` — сокет для чтения.
- `size: int` — максимальное количество байт для чтения.
- `data: var string` (во втором варианте) — буфер, в который записывается результат; будет изменён в размере.
- `timeout: int` — таймаут в миллисекундах, `-1` означает "ждать бесконечно".
- `flags: set[SocketFlag]` — из всего набора реально учитывается только `SafeDisconn`.

Примеры:

```nim
let socket = newSocket()
connect(socket, "example.com", Port(80))
send(socket, "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
let response = recv(socket, 100) # читает до 100 байт ответа
echo response.len > 0 # выводит true, если сервер уже прислал данные
close(socket)

let closedSideSocket = newSocket()
connect(closedSideSocket, "example.com", Port(80))
close(closedSideSocket)
# после close чтение из этого же объекта работать не должно —
# на живом соединении опустошение потока выглядит так:
let liveSocket = newSocket()
connect(liveSocket, "example.com", Port(81)) # порт, где никто не отвечает данными
skip(liveSocket, 0) # граничный случай: запрос нулевого размера — без эффекта
close(liveSocket)
```

---

### `recvFrom`

```nim
proc recvFrom*[T: string | IpAddress](socket: Socket, data: var string, length: int,
               address: var T, port: var Port, flags = 0'i32): int
```

Что делает: принимает одну датаграмму на UDP-сокете и одновременно сообщает, откуда она пришла — адрес отправителя записывается в `address` (либо как строка, либо сразу как разобранный `IpAddress`, в зависимости от того, какой тип передан).
Явно рассчитана на сокеты без установленного соединения — вызов на TCP-сокете (`IPPROTO_TCP`) запрещён проверкой `assert`.

Список параметров:

- `socket: Socket` — UDP-сокет.
- `data: var string` — буфер для полученных данных.
- `length: int` — сколько байт максимум прочитать.
- `address: var (string | IpAddress)` — куда записать адрес отправителя.
- `port: var Port` — куда записать порт отправителя.
- `flags: int32` — низкоуровневые флаги системного вызова `recvfrom`.

Примеры:

```nim
let udpSocket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
bindAddr(udpSocket, Port(27960))

var buffer = ""
var senderAddress: IpAddress
var senderPort: Port
discard recvFrom(udpSocket, buffer, 1024, senderAddress, senderPort)
echo senderAddress # выводит адрес отправителя, например 192.168.0.5
echo buffer # выводит содержимое полученной датаграммы
close(udpSocket)
```

---

### `readLine` и `recvLine`

```nim
proc readLine*(socket: Socket, line: var string, timeout = -1,
               flags = {SocketFlag.SafeDisconn}, maxLength = MaxLineLength)
proc recvLine*(socket: Socket, timeout = -1, flags = {SocketFlag.SafeDisconn},
               maxLength = MaxLineLength): string
```

Что делает: читают из сокета одну строку, оканчивающуюся `\n` или `\r\n`, побайтно — сам символ(ы) конца строки в результат не попадают, за исключением особого случая, когда строка состоит **только** из `\r\n` (пустая строка), тогда в результат кладётся именно `\r\n`, чтобы отличить "прочитана пустая строка" от "соединение разорвано" (в последнем случае результат — пустая строка `""`).
`recvLine` — это `readLine`, возвращающая результат вместо записи в параметр `var`.
Параметр `maxLength` защищает от атаки переполнением памяти: если непрерывная последовательность байт без символа конца строки превышает `maxLength` (по умолчанию миллион байт), чтение обрывается с усечённым результатом, не дожидаясь настоящего конца строки.

Разбор реализации: процедура читает байт за байтом через `recv`; при встрече `\r` она "подглядывает" следующий байт через `peekChar`, не извлекая его из потока, и если это `\l`, то оба байта считаются одним разделителем и извлекаются вместе — это позволяет корректно обработать как чистый Unix-перевод строки (`\n`), так и Windows-стиль (`\r\n`), не считывая лишний байт следующей строки по ошибке.

Список параметров:

- `socket: Socket` — сокет для чтения.
- `line: var string` (в `readLine`) — куда записать прочитанную строку.
- `timeout: int` — таймаут ожидания данных в миллисекундах.
- `flags: set[SocketFlag]` — учитывается только `SafeDisconn`.
- `maxLength: int` — предел длины одной строки.

Примеры:

```nim
let socket = newSocket()
connect(socket, "example.com", Port(80))
send(socket, "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
let status = recvLine(socket)
echo status # выводит первую строку ответа, например "HTTP/1.1 200 OK"

var line = ""
readLine(socket, line)
echo line.len == 0 # выводит true, если это пустая разделительная строка между заголовками и телом
close(socket)

let disconnectedSocket = newSocket()
connect(disconnectedSocket, "example.com", Port(80))
close(disconnectedSocket)
```

---

### `skip` и `hasDataBuffered`

```nim
proc skip*(socket: Socket, size: int, timeout = -1)
proc hasDataBuffered*(s: Socket): bool
```

Что делает: `skip` читает и отбрасывает ровно `size` байт из сокета — удобно, когда нужно "пропустить" известную по протоколу часть данных, не выделяя под неё строку.
`hasDataBuffered` сообщает, есть ли прямо сейчас непрочитанные данные во внутреннем буфере сокета (без обращения к сети) — полезно перед вызовом `select`, чтобы не ждать событие от ОС, если данные уже лежат в локальном буфере и `recv` их вернёт немедленно.

Список параметров:

- `socket: Socket` / `s: Socket` — сокет.
- `size: int` (в `skip`) — сколько байт пропустить.
- `timeout: int` (в `skip`) — таймаут ожидания данных.

Примеры:

```nim
let socket = newSocket()
connect(socket, "example.com", Port(80))
send(socket, "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
discard recvLine(socket) # прочитали статусную строку
skip(socket, 4) # пропускаем, например, известный по протоколу префикс тела
echo hasDataBuffered(socket) # выводит true, если после skip в буфере ещё что-то осталось
close(socket)
```

---

## Параметры и диагностика сокета

### `getSockOpt` и `setSockOpt`

```nim
proc getSockOpt*(socket: Socket, opt: SOBool, level = SOL_SOCKET): bool
proc setSockOpt*(socket: Socket, opt: SOBool, value: bool, level = SOL_SOCKET)
```

Что делает: читает или устанавливает булеву опцию сокета уровня операционной системы (см. перечисление `SOBool` выше).
Параметр `level` определяет, к какому уровню сетевого стека относится опция — по умолчанию `SOL_SOCKET` (общие опции сокета), но, например, `OptNoDelay` относится к уровню `IPPROTO_TCP`, и это нужно указать явно.

Список параметров:

- `socket: Socket` — сокет.
- `opt: SOBool` — какая опция читается/устанавливается.
- `value: bool` (в `setSockOpt`) — новое значение.
- `level: cint` — уровень сетевого стека опции.

Примеры:

```nim
let socket = newSocket()
setSockOpt(socket, OptReuseAddr, true)
echo getSockOpt(socket, OptReuseAddr) # выводит true
setSockOpt(socket, OptNoDelay, true, level = IPPROTO_TCP.cint) # уровень важно указать явно
close(socket)
```

---

### `getLocalAddr` и `getPeerAddr`

```nim
proc getLocalAddr*(socket: Socket): (string, Port)
proc getPeerAddr*(socket: Socket): (string, Port)
```

Что делает: `getLocalAddr` возвращает адрес и порт, на которые в действительности привязан локальный конец сокета (актуально после `bindAddr` с портом `0`, чтобы узнать, какой порт реально выбрала ОС); `getPeerAddr` — то же самое для удалённой стороны установленного соединения.

Список параметров:

- `socket: Socket` — сокет, для которого запрашивается адрес.

Примеры:

```nim
let server = newSocket()
bindAddr(server, Port(0))
let (localAddress, localPort) = getLocalAddr(server)
echo localPort # выводит порт, выданный операционной системой
close(server)

let client = newSocket()
connect(client, "example.com", Port(80))
let (peerAddress, peerPort) = getPeerAddr(client)
echo peerPort # выводит 80
close(client)
```

---

### `getFd` и `isSsl`

```nim
proc getFd*(socket: Socket): SocketHandle
proc isSsl*(socket: Socket): bool
```

Что делает: `getFd` возвращает низкоуровневый дескриптор сокета — нужен, если требуется работать напрямую с системными вызовами (например, `select`/`poll` из других модулей) в обход высокоуровневого API `net`.
`isSsl` сообщает, обёрнут ли данный сокет в TLS-сессию через `wrapSocket`/`wrapConnectedSocket`.

Список параметров:

- `socket: Socket` — сокет.

Примеры:

```nim
let socket = newSocket()
echo isSsl(socket) # выводит false — обычный TCP-сокет
let fd = getFd(socket)
echo fd != osInvalidSocket # выводит true, пока сокет открыт
close(socket)
```

---

### `socketError`, `getSocketError`, `isDisconnectionError`

```nim
proc socketError*(socket: Socket, err: int = -1, async = false,
                   lastError = (-1).OSErrorCode, flags: set[SocketFlag] = {})
proc getSocketError*(socket: Socket): OSErrorCode
proc isDisconnectionError*(flags: set[SocketFlag], lastError: OSErrorCode): bool
```

Что делает: это внутренние по назначению, но экспортированные процедуры диагностики ошибок, на которых построены `send`, `recv`, `readLine` и другие процедуры модуля; напрямую их обычно вызывать не нужно, но они полезны при написании собственных обёрток над сокетом.
`getSocketError` читает системный код последней ошибки сокета (через `SO_ERROR`) и, если он ненулевой, сразу поднимает `OSError`.
`isDisconnectionError` определяет, относится ли код ошибки к "типичному разрыву соединения" (ECONNRESET, EPIPE, ENETRESET и их аналоги на Windows) — но возвращает `true` только если в переданных `flags` присутствует `SafeDisconn`; это и есть тот самый переключатель, которым процедуры вроде `send`/`recv` решают, поднимать исключение или трактовать разрыв как штатное завершение.
`socketError` — общая точка, которая, получив отрицательный результат системного вызова, определяет актуальный код ошибки и либо поднимает `OSError`, либо (если это "безопасный" разрыв соединения) игнорирует его.

Список параметров:

- `socket: Socket` — сокет, с которым связана ошибка.
- `err: int` — код возврата системного вызова, породившего проверку.
- `lastError: OSErrorCode` — уже определённый код последней ошибки, если он известен заранее.
- `flags: set[SocketFlag]` — влияет на трактовку ошибок разрыва соединения.

Примеры:

```nim
let socket = newSocket()
connect(socket, "example.com", Port(80))
echo getSocketError(socket) # выводит 0.OSErrorCode, если ошибок ещё не было
echo isDisconnectionError({SocketFlag.SafeDisconn}, EPIPE.OSErrorCode) # выводит true
echo isDisconnectionError({}, EPIPE.OSErrorCode) # выводит false — флаг не выставлен
close(socket)
```

---

## SSL/TLS

Все процедуры этого раздела доступны только при компиляции с флагом `-d:ssl` и требуют установленной библиотеки OpenSSL.

### `newContext`

```nim
proc newContext*(protVersion = protSSLv23, verifyMode = CVerifyPeer,
                  certFile = "", keyFile = "", cipherList = CiphersIntermediate,
                  caDir = "", caFile = "", ciphersuites = CiphersModern): SslContext
```

Что делает: создаёт объект `SslContext` — конфигурацию TLS-соединения (набор допустимых шифров, режим проверки сертификатов, доверенные корневые сертификаты), которую затем можно применить к одному или нескольким сокетам через `wrapSocket`.
`certFile`/`keyFile` нужны, если сокет впоследствии будет использоваться как серверный (для клиента обычно не требуются).
Если `verifyMode` не `CVerifyNone`, при отсутствии явных `caFile`/`caDir` контекст автоматически ищет системные корневые сертификаты в стандартных для данной ОС расположениях — и поднимает `IOError`, если ни одного не нашлось.

Список параметров:

- `protVersion: SslProtVersion` — желаемая версия протокола (на практике почти всегда игнорируется в пользу автоматического согласования TLS).
- `verifyMode: SslCVerifyMode` — `CVerifyNone` (не проверять сертификат), `CVerifyPeer` (проверять) или `CVerifyPeerUseEnvVars` (проверять, дополнительно используя переменные окружения `SSL_CERT_FILE`/`SSL_CERT_DIR`).
- `certFile, keyFile: string` — пути к сертификату и приватному ключу сервера.
- `cipherList, ciphersuites: string` — списки разрешённых наборов шифров для TLS ≤1.2 и TLS 1.3 соответственно.
- `caDir, caFile: string` — явные пути к доверенным корневым сертификатам.

Примеры:

```nim
let clientContext = newContext() # проверка сертификата включена по умолчанию
let insecureContext = newContext(verifyMode = CVerifyNone) # только для тестов/отладки!
let serverContext = newContext(certFile = "cert.pem", keyFile = "key.pem")
destroyContext(clientContext)
destroyContext(insecureContext)
destroyContext(serverContext)
```

---

### `wrapSocket`

```nim
proc wrapSocket*(ctx: SslContext, socket: Socket)
```

Что делает: превращает обычный, ещё не подключённый сокет в SSL-сокет, привязывая к нему контекст `ctx`; фактическое TLS-рукопожатие начнётся автоматически при последующем вызове `connect` (для клиента) или произойдёт внутри `acceptAddr` (для серверного сокета).
Вызов на уже подключённом сокете не поддерживается этой процедурой — для такого случая есть `wrapConnectedSocket`.

Список параметров:

- `ctx: SslContext` — заранее созданный контекст.
- `socket: Socket` — обычный, ещё не подключённый сокет.

Примеры:

```nim
let ctx = newContext()
let socket = newSocket()
wrapSocket(ctx, socket)
connect(socket, "example.com", Port(443)) # рукопожатие происходит здесь
echo isSsl(socket) # выводит true
close(socket)
destroyContext(ctx)
```

---

### `wrapConnectedSocket`

```nim
proc wrapConnectedSocket*(ctx: SslContext, socket: Socket,
                           handshake: SslHandshakeType, hostname: string = "")
```

Что делает: то же оборачивание в SSL, что и `wrapSocket`, но для уже подключённого сокета — рукопожатие выполняется немедленно, внутри самого вызова, а не откладывается до следующего `connect`.
Параметр `hostname` нужен клиенту для SNI (указания серверу, к какому виртуальному хосту подключаемся) и для последующей сверки этого имени с сертификатом сервера; при `handshake == handshakeAsServer` рукопожатие выполняется в серверной роли.

Список параметров:

- `ctx: SslContext` — контекст.
- `socket: Socket` — уже подключённый обычный сокет.
- `handshake: SslHandshakeType` — `handshakeAsClient` или `handshakeAsServer`.
- `hostname: string` — имя хоста для SNI и проверки сертификата (только для клиентской роли).

Примеры:

```nim
let ctx = newContext()
let socket = newSocket()
connect(socket, "example.com", Port(443)) # сначала обычное TCP-подключение
wrapConnectedSocket(ctx, socket, handshakeAsClient, "example.com") # затем TLS поверх него
echo isSsl(socket) # выводит true
close(socket)
destroyContext(ctx)
```

---

### `getPeerCertificates`

```nim
proc getPeerCertificates*(socket: Socket): seq[Certificate]
```

Что делает: возвращает цепочку сертификатов, предъявленных удалённой стороной, от листового сертификата к корневому — но только если TLS-рукопожатие завершилось и цепочка была успешно проверена; в противном случае (включая случай, когда сокет вообще не SSL) возвращается пустая последовательность, а не ошибка.

Список параметров:

- `socket: Socket` — SSL-сокет с уже завершённым рукопожатием.

Примеры:

```nim
let ctx = newContext()
let socket = newSocket()
wrapSocket(ctx, socket)
connect(socket, "example.com", Port(443))
let chain = getPeerCertificates(socket)
echo chain.len > 0 # выводит true при успешной проверке цепочки сертификатов
close(socket)
destroyContext(ctx)
```

---

### `destroyContext`

```nim
proc destroyContext*(ctx: SslContext)
```

Что делает: освобождает память и внутренние ресурсы OpenSSL, связанные с контекстом; после вызова контекст больше нельзя использовать для оборачивания новых сокетов.
Уже обёрнутые и работающие через этот контекст сокеты сами по себе не закрываются — их нужно закрыть отдельно через `close`.

Список параметров:

- `ctx: SslContext` — контекст для уничтожения.

Примеры:

```nim
let ctx = newContext()
let socket = newSocket()
wrapSocket(ctx, socket)
connect(socket, "example.com", Port(443))
close(socket)
destroyContext(ctx) # контекст больше не нужен
```

---

## Практические рецепты

### Простой TCP-клиент со строковым протоколом

```nim
import std/net

let socket = newSocket()
connect(socket, "example.com", Port(80))
send(socket, "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")

let statusLine = recvLine(socket)
echo statusLine # выводит, например, "HTTP/1.1 200 OK"

while true:
  let line = recvLine(socket)
  if line.len == 0: break # соединение закрыто сервером
  echo line

close(socket)
```

---

### Эхо-сервер с обработкой отключений

```nim
import std/net

let server = newSocket()
setSockOpt(server, OptReuseAddr, true)
bindAddr(server, Port(9090))
listen(server)
echo "Сервер слушает порт 9090"

while true:
  var client: Socket
  var clientAddress = ""
  acceptAddr(server, client, clientAddress) # SafeDisconn уже включён по умолчанию
  echo "Подключился клиент: ", clientAddress
  while true:
    let line = recvLine(client)
    if line.len == 0: break
    send(client, line & "\c\l") # эхо обратно клиенту
  close(client)
```

---

### UDP-обмен датаграммами

```nim
import std/net

let udpSocket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
bindAddr(udpSocket, Port(9091))

var incoming = ""
var senderAddress: IpAddress
var senderPort: Port
discard recvFrom(udpSocket, incoming, 1024, senderAddress, senderPort)
echo "Получено от ", senderAddress, ":", senderPort, " — ", incoming

discard sendTo(udpSocket, senderAddress, senderPort,
               unsafeAddr incoming[0], incoming.len) # отправляем то же самое обратно
close(udpSocket)
```

---

### Подключение с ограничением по времени

```nim
import std/net

proc connectWithFallback(hosts: seq[string], port: Port, timeoutMs: int): Socket =
  for host in hosts:
    let socket = newSocket()
    try:
      connect(socket, host, port, timeout = timeoutMs)
      return socket # первый успешно подключившийся хост
    except TimeoutError, OSError:
      close(socket) # пробуем следующий хост из списка
  raise newException(IOError, "Ни один из хостов не ответил")

let mirrors = @["mirror1.example.com", "mirror2.example.com", "mirror3.example.com"]
let socket = connectWithFallback(mirrors, Port(80), 1000)
close(socket)
```

---

### Определение собственного внешнего адреса

```nim
import std/net

let localAddress = getPrimaryIPAddr()
echo "Локальный адрес для внешнего трафика: ", localAddress

let localAddressV6 = getPrimaryIPAddr(parseIpAddress("2001:4860:4860::8888"))
echo "Локальный IPv6-адрес: ", localAddressV6
```

---

### Клиент с TLS и проверкой сертификата

```nim
import std/net

let ctx = newContext() # проверка сертификата включена по умолчанию
let socket = newSocket()
wrapSocket(ctx, socket)
connect(socket, "example.com", Port(443)) # рукопожатие и проверка имени — здесь

send(socket, "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
echo recvLine(socket) # выводит статусную строку HTTPS-ответа

close(socket)
destroyContext(ctx)
```

---

## Краткая таблица

| Задача | Изменяет аргумент | Возвращает новую сущность / просто действует |
| --- | --- | --- |
| Создать сокет | — | `newSocket`, `dial` |
| Разобрать строку в адрес | — | `parseIpAddress`, `isIpAddress` |
| Получить строку из адреса | — | `` `$` `` |
| Сравнить адреса | — | `` `==` `` |
| Подключиться к серверу | сокет (устанавливает соединение) | `connect` |
| Стать сервером на порту | сокет (привязка), затем очередь | `bindAddr`, `listen` |
| Принять входящее соединение | `client`/`address` (заполняются) | `acceptAddr`, `accept` |
| Отправить данные | — | `send`, `` `&=` ``, `trySend`, `sendTo` |
| Получить данные | `data`/`line` (заполняются) | `recv`, `recvLine`, `readLine`, `recvFrom` |
| Пропустить данные | — | `skip` |
| Проверить локальный буфер | — | `hasDataBuffered` |
| Прочитать/задать опцию сокета | сокет (для `setSockOpt`) | `getSockOpt`, `setSockOpt` |
| Узнать адрес сокета | — | `getLocalAddr`, `getPeerAddr` |
| Узнать код последней ошибки | — | `getSocketError` |
| Закрыть сокет | сокет (закрывается) | `close` |
| Настроить TLS | — | `newContext`, `wrapSocket`, `wrapConnectedSocket` |
| Проверить сертификат пира | — | `getPeerCertificates` |

---

## Сводка: какую процедуру выбрать

- Нужно просто скачать что-то по HTTP/HTTPS с известным хостом и портом → используйте `dial` (сам разберётся между IPv4 и IPv6).
- Уже есть сокет и его нужно подключить к серверу → используйте `connect`; если важно не "зависнуть" навсегда — вариант `connect` с параметром `timeout`.
- Нужно узнать, является ли строка IP-адресом, прежде чем решать, резолвить её как имя или нет → используйте `isIpAddress`.
- Нужно превратить строку в структуру для дальнейшей работы (сравнения, сохранения) → используйте `parseIpAddress`.
- Нужно принимать входящие TCP-соединения → последовательность `newSocket` → `bindAddr` → `listen` → `acceptAddr`/`accept`.
- Нужно узнать адрес подключившегося клиента → `acceptAddr`, а не `accept`.
- Нужно отправить данные и быть уверенным, что уйдут все байты → `send`; если ошибка сети не критична — `trySend`.
- Нужно работать с UDP (без установления соединения) → `sendTo`/`recvFrom`, а не `send`/`recv`.
- Нужно прочитать данные построчно (например, текстовый протокол вроде HTTP) → `recvLine`/`readLine`, а не `recv` с ручным поиском символа конца строки.
- Нужно прочитать ровно `size` байт, не заботясь о частичных пакетах → буферизованный `recv` (сокет создан с `buffered = true`, значение по умолчанию).
- Нужно пропустить известную по протоколу часть потока, не выделяя под неё память → `skip`.
- Нужно узнать порт, который операционная система выбрала сама после `bindAddr(socket, Port(0))` → `getLocalAddr`.
- Нужно определить собственный внешний IP-адрес, не отправляя пакетов → `getPrimaryIPAddr`.
- Нужно защитить соединение через TLS → `newContext` (один раз для конфигурации) + `wrapSocket` (до `connect`) или `wrapConnectedSocket` (после `connect`).
- Нужно убедиться, что сертификат сервера действителен и получить его цепочку → `getPeerCertificates` после установления соединения.
- Нужно освободить ресурсы TLS-конфигурации по завершении работы → `destroyContext`.
