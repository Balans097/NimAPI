# Справочник модуля `asyncdispatch` (Nim)

> Модуль стандартной библиотеки Nim для асинхронного ввода-вывода.  
> Реализует диспетчер событий, тип `Future[T]` и макрос `async`/`await`.

---

## Содержание

1. [Концепции](#концепции)
2. [Типы](#типы)
3. [Диспетчер событий](#диспетчер-событий)
4. [Сокеты: приём данных](#сокеты-приём-данных)
5. [Сокеты: отправка данных](#сокеты-отправка-данных)
6. [Сокеты: подключение и принятие соединений](#сокеты-подключение-и-принятие-соединений)
7. [Таймеры и задержки](#таймеры-и-задержки)
8. [События, процессы, сигналы](#события-процессы-сигналы)
9. [Цикл событий](#цикл-событий)
10. [Вспомогательные функции](#вспомогательные-функции)
11. [Быстрая шпаргалка](#быстрая-шпаргалка)

---

## Концепции

### Future и проакторная модель

`Future[T]` — это значение, которое может быть недоступно прямо сейчас, но станет доступным позже. Когда асинхронная операция завершается, Future «выполняется» (`complete`) значением или «завершается ошибкой» (`fail`).

```nim
import asyncdispatch

proc main() {.async.} =
  let fut = sleepAsync(100)   # возвращает Future[void]
  await fut                   # ждём без блокировки потока
  echo "Прошло 100 мс"

waitFor main()
```

### async / await

Процедура, помеченная `{.async.}`, должна возвращать `Future[T]` или ничего (тогда подразумевается `Future[void]`).  
Внутри такой процедуры `await` приостанавливает её выполнение до завершения ожидаемого Future, не блокируя поток.

```nim
proc fetchData(): Future[string] {.async.} =
  await sleepAsync(50)
  return "данные"

proc main() {.async.} =
  let data = await fetchData()
  echo data   # "данные"

waitFor main()
```

### Обработка исключений

```nim
proc main() {.async.} =
  try:
    let data = await someAsyncOp()
    echo data
  except IOError as e:
    echo "Ошибка: ", e.msg

waitFor main()
```

---

## Типы

| Тип | Описание |
|-----|----------|
| `AsyncFD` | Асинхронный файловый дескриптор (обёртка над `int`/`cint`) |
| `PDispatcher` | Диспетчер событий (глобальный на поток) |
| `Callback` | `proc (fd: AsyncFD): bool {.closure, gcsafe.}` — возвращает `true`, чтобы удалить обработчик |
| `AsyncEvent` | Потокобезопасное событие (аналог Event/semaphore) |

---

## Диспетчер событий

### `newDispatcher`

```nim
proc newDispatcher*(): owned PDispatcher
```

Создаёт новый экземпляр диспетчера. Обычно не нужен напрямую — используйте `getGlobalDispatcher`.

```nim
let disp = newDispatcher()
setGlobalDispatcher(disp)
```

---

### `setGlobalDispatcher`

```nim
proc setGlobalDispatcher*(disp: sink PDispatcher)
```

Устанавливает переданный диспетчер как глобальный для текущего потока.

```nim
setGlobalDispatcher(newDispatcher())
```

---

### `getGlobalDispatcher`

```nim
proc getGlobalDispatcher*(): PDispatcher
```

Возвращает глобальный диспетчер текущего потока. Если он не создан — создаёт автоматически.

```nim
let p = getGlobalDispatcher()
echo p.timers.len
```

---

### `getIoHandler`

```nim
# Windows:
proc getIoHandler*(disp: PDispatcher): Handle
# Unix:
proc getIoHandler*(disp: PDispatcher): Selector[AsyncData]
```

Возвращает низкоуровневый IO-обработчик (IO Completion Port на Windows или selector на Unix). Полезно для интеграции с внешними библиотеками.

---

### `register`

```nim
proc register*(fd: AsyncFD)
```

Регистрирует файловый дескриптор в глобальном диспетчере. Необходимо вызвать перед использованием `fd` с асинхронными операциями.

```nim
let sock = createAsyncNativeSocket()
register(sock)
```

---

### `unregister` (fd)

```nim
proc unregister*(fd: AsyncFD)
```

Отменяет регистрацию файлового дескриптора в диспетчере.

```nim
unregister(mySocket)
```

---

### `contains`

```nim
proc contains*(disp: PDispatcher, fd: AsyncFD): bool
```

Проверяет, зарегистрирован ли `fd` в диспетчере `disp`.

```nim
if getGlobalDispatcher().contains(myFd):
  echo "Дескриптор активен"
```

---

### `hasPendingOperations`

```nim
proc hasPendingOperations*(): bool
```

Возвращает `true`, если в глобальном диспетчере есть незавершённые операции (дескрипторы, таймеры или отложенные колбэки).

```nim
while hasPendingOperations():
  poll()
```

---

### `closeSocket`

```nim
proc closeSocket*(socket: AsyncFD)
```

Закрывает сокет и отменяет его регистрацию в диспетчере. Предпочтительнее, чем `close` + `unregister` по отдельности.

```nim
closeSocket(clientSock)
```

---

### `callSoon`

```nim
proc callSoon*(cbproc: proc () {.gcsafe.}) {.gcsafe.}
```

Планирует вызов `cbproc` при следующем проходе цикла событий. Колбэк не блокирует и не ждёт.

```nim
callSoon(proc () = echo "выполнится вскоре")
poll()
```

---

### `activeDescriptors`

```nim
proc activeDescriptors*(): int {.inline.}
```

Возвращает текущее количество активных файловых дескрипторов в цикле событий. Не выполняет системных вызовов.

```nim
echo "Активных соединений: ", activeDescriptors()
```

---

### `maxDescriptors`

```nim
proc maxDescriptors*(): int {.raises: OSError.}
```

Возвращает максимальное число файловых дескрипторов для текущего процесса (выполняет системный вызов). Поддерживается на Windows, Linux, macOS, BSD, Solaris.

```nim
echo "Макс. дескрипторов: ", maxDescriptors()
```

---

### `setInheritable`

```nim
proc setInheritable*(fd: AsyncFD, inheritable: bool): bool
```

Управляет наследованием файлового дескриптора дочерними процессами. Доступность зависит от платформы — проверяйте через `declared(setInheritable)`.

```nim
when declared(setInheritable):
  discard myFd.setInheritable(false)
```

---

## Сокеты: приём данных

### `recv`

```nim
proc recv*(socket: AsyncFD, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[string])
```

Читает **до** `size` байт из `socket`. Future завершается, когда данные получены, часть данных прочитана, или сокет отключился (тогда результат — пустая строка `""`).

> ⚠️ Флаг `Peek` не поддерживается на Windows.

```nim
proc handler() {.async.} =
  let data = await socket.recv(1024)
  if data.len == 0:
    echo "Соединение закрыто"
  else:
    echo "Получено: ", data

waitFor handler()
```

---

### `recvInto`

```nim
proc recvInto*(socket: AsyncFD, buf: pointer, size: int,
               flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

Читает **до** `size` байт из `socket` в предоставленный буфер `buf`. Future возвращает количество прочитанных байт (0 при разрыве соединения).

```nim
proc handler() {.async.} =
  var buf = newString(4096)
  let bytesRead = await socket.recvInto(addr buf[0], buf.len)
  buf.setLen(bytesRead)
  echo "Прочитано байт: ", bytesRead

waitFor handler()
```

---

### `recvFromInto`

```nim
proc recvFromInto*(socket: AsyncFD, data: pointer, size: int,
                   saddr: ptr SockAddr, saddrLen: ptr SockLen,
                   flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

Получает UDP-датаграмму из `socket` в буфер `data`. Адрес отправителя записывается в `saddr`/`saddrLen`. Future возвращает размер полученного пакета.

```nim
proc udpHandler() {.async.} =
  var buf = newString(65535)
  var sender: Sockaddr_storage
  var senderLen = sizeof(sender).SockLen
  let size = await socket.recvFromInto(
    addr buf[0], buf.len,
    cast[ptr SockAddr](addr sender), addr senderLen
  )
  echo "UDP пакет размером ", size, " байт"

waitFor udpHandler()
```

---

## Сокеты: отправка данных

### `send` (из буфера)

```nim
proc send*(socket: AsyncFD, buf: pointer, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

Отправляет `size` байт из буфера `buf` в `socket`. Future завершается после отправки всех данных.

> ⚠️ При работе с GC-объектами используйте `GC_ref`/`GC_unref` для защиты буфера от досрочной очистки.

```nim
proc sendRaw() {.async.} =
  var data = "Hello"
  GC_ref(data)
  await socket.send(unsafeAddr data[0], data.len)
  GC_unref(data)

waitFor sendRaw()
```

---

### `send` (строка)

```nim
proc send*(socket: AsyncFD, data: string,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

Высокоуровневая версия: отправляет строку `data` в `socket`. Безопаснее, чем вариант с указателем.

```nim
proc sendMsg() {.async.} =
  await socket.send("Привет, мир!\n")

waitFor sendMsg()
```

---

### `sendTo`

```nim
proc sendTo*(socket: AsyncFD, data: pointer, size: int,
             saddr: ptr SockAddr, saddrLen: SockLen,
             flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

Отправляет UDP-датаграмму `data` на адрес `saddr`. Future завершается после отправки.

```nim
proc udpSend() {.async.} =
  var msg = "ping"
  var dest: Sockaddr_in
  # ... заполните dest ...
  await socket.sendTo(
    addr msg[0], msg.len,
    cast[ptr SockAddr](addr dest), sizeof(dest).SockLen
  )

waitFor udpSend()
```

---

## Сокеты: подключение и принятие соединений

### `createAsyncNativeSocket`

```nim
proc createAsyncNativeSocket*(
  domain: Domain = Domain.AF_INET,
  sockType: SockType = SOCK_STREAM,
  protocol: Protocol = IPPROTO_TCP,
  inheritable = defined(nimInheritHandles)
): AsyncFD
```

Создаёт нативный асинхронный сокет и автоматически регистрирует его в диспетчере. Есть перегрузка с `cint`-параметрами для нестандартных протоколов.

```nim
let sock = createAsyncNativeSocket()
# sock готов к использованию с async операциями
```

---

### `connect`

```nim
proc connect*(socket: AsyncFD, address: string, port: Port,
              domain = Domain.AF_INET): owned(Future[void])
```

Асинхронно подключает существующий сокет `socket` к `address:port`.

```nim
proc main() {.async.} =
  let sock = createAsyncNativeSocket()
  await sock.connect("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\n\r\n")
  let resp = await sock.recv(4096)
  echo resp

waitFor main()
```

---

### `dial`

```nim
proc dial*(address: string, port: Port,
           protocol: Protocol = IPPROTO_TCP): owned(Future[AsyncFD])
```

Создаёт сокет, перебирает все IP-адреса (IPv4 и IPv6) для `address` и подключается к первому успешному. Возвращает готовый к работе `AsyncFD`.

```nim
proc main() {.async.} =
  let sock = await dial("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\n\r\n")
  echo await sock.recv(4096)

waitFor main()
```

---

### `acceptAddr`

```nim
proc acceptAddr*(socket: AsyncFD,
                 flags = {SocketFlag.SafeDisconn},
                 inheritable = defined(nimInheritHandles)):
    owned(Future[tuple[address: string, client: AsyncFD]])
```

Принимает входящее соединение на `socket`. Future возвращает кортеж с адресом клиента и его сокетом. Клиентский сокет автоматически регистрируется в диспетчере.

```nim
proc server(listener: AsyncFD) {.async.} =
  while true:
    let (address, client) = await listener.acceptAddr()
    echo "Подключился: ", address
    asyncCheck handleClient(client)

waitFor server(listenerSocket)
```

---

### `accept`

```nim
proc accept*(socket: AsyncFD,
             flags = {SocketFlag.SafeDisconn},
             inheritable = defined(nimInheritHandles)): owned(Future[AsyncFD])
```

Упрощённый вариант `acceptAddr` — возвращает только сокет клиента без адреса.

```nim
proc server(listener: AsyncFD) {.async.} =
  let client = await listener.accept()
  await client.send("Добро пожаловать!\n")

waitFor server(listenerSocket)
```

---

## Таймеры и задержки

### `sleepAsync`

```nim
proc sleepAsync*(ms: int | float): owned(Future[void])
```

Приостанавливает выполнение текущей async-процедуры на `ms` миллисекунд. Принимает `int` или `float`.

```nim
proc main() {.async.} =
  echo "Начало"
  await sleepAsync(1000)    # 1 секунда
  await sleepAsync(500.0)   # 0.5 секунды
  echo "Конец"

waitFor main()
```

---

### `withTimeout`

```nim
proc withTimeout*[T](fut: Future[T], timeout: int): owned(Future[bool])
```

Возвращает Future, который завершается `true`, если `fut` завершился раньше тайм-аута, или `false`, если истёк таймаут `timeout` мс.

```nim
proc main() {.async.} =
  let longOp = sleepAsync(5000)
  let completed = await longOp.withTimeout(1000)
  if completed:
    echo "Завершено вовремя"
  else:
    echo "Тайм-аут!"

waitFor main()
```

---

### `addTimer`

```nim
proc addTimer*(timeout: int, oneshot: bool, cb: Callback)
```

Регистрирует низкоуровневый колбэк `cb`, вызываемый через `timeout` мс.  
- `oneshot = true` — только один раз  
- `oneshot = false` — повторяется каждые `timeout` мс  

Колбэк должен возвращать `true`, чтобы удалить себя, или `false` для повторного вызова.

> Доступно только на платформах с поддержкой `ioselSupportedPlatform` (Linux, macOS, BSD) и на Windows.

```nim
var count = 0
addTimer(500, false, proc(fd: AsyncFD): bool =
  inc count
  echo "Тик #", count
  return count >= 5  # остановить после 5 вызовов
)
runForever()
```

---

## События, процессы, сигналы

### `newAsyncEvent`

```nim
proc newAsyncEvent*(): AsyncEvent
```

Создаёт потокобезопасный объект события. Не регистрируется в диспетчере автоматически — используйте `addEvent`.

```nim
let ev = newAsyncEvent()
```

---

### `trigger`

```nim
proc trigger*(ev: AsyncEvent)
```

Переводит событие `ev` в сигнальное состояние. Безопасно вызывать из другого потока.

```nim
ev.trigger()
```

---

### `addEvent`

```nim
proc addEvent*(ev: AsyncEvent, cb: Callback)
```

Регистрирует колбэк `cb`, который будет вызван при переходе `ev` в сигнальное состояние.

```nim
let ev = newAsyncEvent()
addEvent(ev, proc(fd: AsyncFD): bool =
  echo "Событие сработало!"
  return true  # удалить обработчик
)
ev.trigger()
poll()
```

---

### `unregister` (ev)

```nim
proc unregister*(ev: AsyncEvent)
```

Отменяет регистрацию события в диспетчере.

```nim
unregister(ev)
```

---

### `close` (ev)

```nim
proc close*(ev: AsyncEvent)
```

Закрывает и освобождает ресурсы события.

```nim
ev.close()
```

---

### `addProcess`

```nim
proc addProcess*(pid: int, cb: Callback)
```

Регистрирует колбэк `cb`, вызываемый при завершении процесса с PID `pid`.

> Доступно на Windows и на Unix-платформах с поддержкой `ioselSupportedPlatform`.

```nim
addProcess(childPid, proc(fd: AsyncFD): bool =
  echo "Процесс завершился"
  return true
)
```

---

### `addSignal` (только Unix)

```nim
proc addSignal*(signal: int, cb: Callback)
```

Регистрирует колбэк `cb`, вызываемый при получении сигнала `signal`.

> Доступно только на платформах с `ioselSupportedPlatform` (Linux, macOS, BSD).

```nim
import posix
addSignal(SIGTERM.int, proc(fd: AsyncFD): bool =
  echo "Получен SIGTERM, завершаем..."
  return true
)
```

---

### `addRead`

```nim
proc addRead*(fd: AsyncFD, cb: Callback)
```

Следит за `fd` на готовность к чтению и вызывает `cb`. На Windows использует не-IOCP механизм — применяйте только при необходимости (например, для совместимости с Unix-библиотеками).

```nim
addRead(myFd, proc(fd: AsyncFD): bool =
  echo "Данные доступны для чтения"
  return true  # true = прекратить наблюдение
)
```

---

### `addWrite`

```nim
proc addWrite*(fd: AsyncFD, cb: Callback)
```

Следит за `fd` на готовность к записи и вызывает `cb`. На Windows не использует IOCP.

```nim
addWrite(myFd, proc(fd: AsyncFD): bool =
  echo "Сокет готов к записи"
  return true
)
```

---

## Цикл событий

### `poll`

```nim
proc poll*(timeout = 500)
```

Выполняет **один** проход цикла событий: ожидает готовые события и обрабатывает их. Вызывает исключение `ValueError`, если нет ни одной зарегистрированной операции.

```nim
# Ручной цикл событий:
while hasPendingOperations():
  poll(100)
```

---

### `drain`

```nim
proc drain*(timeout = 500)
```

Обрабатывает **все** доступные события в течение `timeout` мс. В отличие от `poll`, не останавливается после первого прохода.

```nim
drain(2000)  # обрабатывать события до 2 секунд
```

---

### `runForever`

```nim
proc runForever*()
```

Запускает бесконечный цикл событий. Используется, когда программа должна работать постоянно (серверы).

```nim
runForever()  # не возвращается
```

---

### `waitFor`

```nim
proc waitFor*[T](fut: Future[T]): T
```

**Блокирует** текущий поток до завершения `fut`, крутя цикл событий. Используется для вызова async-кода из синхронного контекста. **Никогда не используйте внутри async-процедуры!**

```nim
proc fetchPage(): Future[string] {.async.} =
  await sleepAsync(100)
  return "страница"

let result = waitFor fetchPage()
echo result
```

---

### `readAll`

```nim
proc readAll*(future: FutureStream[string]): owned(Future[string]) {.async.}
```

Читает все строковые данные из `FutureStream` до его закрытия и возвращает их как одну строку.

```nim
proc main() {.async.} =
  let stream = newFutureStream[string]()
  # ... запись в stream ...
  stream.complete()
  let allData = await readAll(stream)
  echo allData

waitFor main()
```

---

## Вспомогательные функции

### `newCustom` (только Windows)

```nim
proc newCustom*(): CustomRef
```

Создаёт новую структуру `OVERLAPPED` для использования с IOCP на Windows. Применяется при реализации нестандартных асинхронных операций.

---

### `scheduleCallbacks` (только Genode)

```nim
proc scheduleCallbacks*(): bool {.discardable.}
```

Планирует обработку очереди `callSoon` на платформе Genode. Эквивалентно неблокирующему `poll()`.

---

## Быстрая шпаргалка

| Задача | Функция |
|--------|---------|
| Запустить async из sync | `waitFor myFut` |
| Ждать в async-процедуре | `await myFut` |
| Проверить без ожидания | `asyncCheck myFut` |
| Задержка | `await sleepAsync(ms)` |
| Тайм-аут | `await myFut.withTimeout(ms)` |
| Создать сокет | `createAsyncNativeSocket()` |
| Подключиться | `await dial(host, port)` |
| Принять соединение | `await listener.acceptAddr()` |
| Читать | `await socket.recv(size)` |
| Отправить | `await socket.send(data)` |
| Бесконечный сервер | `runForever()` |
| Один проход | `poll()` |
| Все события | `drain()` |
| Активные соединения | `activeDescriptors()` |

---

### Ограничения модуля

- Система эффектов (`raises: []`) не работает с async-процедурами.
- Изменяемые параметры (`var T`) не поддерживаются в async. Используйте `ref T`.
- `waitFor` нельзя вызывать внутри async-процедуры.
- Никогда не используйте `discard` с Future — используйте `asyncCheck`.
