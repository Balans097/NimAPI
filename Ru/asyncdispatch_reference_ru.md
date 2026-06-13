# asyncdispatch.nim — справочник модуля

> **Импорт:** `import std/asyncdispatch`
> **Область применения:** реализация асинхронного ввода-вывода в Nim — диспетчер событий, тип `Future[T]`, и поддержка макроса `{.async.}`/`await`. Реэкспортирует `std/asyncfutures` (типы `Future`, `FutureStream` и операции над ними) и `std/asyncstreams`, а также `Port` и `SocketFlag` из `std/net`/`std/nativesockets`.

Модуль `asyncdispatch` — это сердце асинхронного программирования в Nim. Он не реализует конкретные протоколы (HTTP, TCP-сервер и т.п.) — этим занимаются модули более высокого уровня (`asyncnet`, `asynchttpserver` и др.). Вместо этого `asyncdispatch` предоставляет три кита, на которых всё остальное строится:

1. **Диспетчер (event loop)** — глобальный объект, который опрашивает операционную систему (через `epoll` на Linux, IO Completion Ports на Windows, `select`/`kqueue` на других платформах) на предмет завершённых операций ввода-вывода, сработавших таймеров и сигналов.
2. **`Future[T]`** — «обещание» значения, которое станет доступно позже. У future есть три состояния: *в ожидании* (pending), *завершён успешно* (со значением `T`), *завершён с ошибкой* (с исключением).
3. **Макрос `{.async.}` и `await`** — синтаксический сахар, превращающий код, написанный в привычном последовательном стиле, в цепочку обратных вызовов (callbacks) над future, без необходимости писать их вручную.

---

## Содержание

1. [Концепции: Future, async/await, асинхронные процедуры](#концепции-future-asyncawait-асинхронные-процедуры)
2. [Три способа работы с future: `asyncCheck`, `waitFor`, `await`](#три-способа-работы-с-future)
3. [Обработка исключений](#обработка-исключений)
4. [Основные типы](#основные-типы)
5. [Управление диспетчером](#управление-диспетчером)
6. [Цикл событий: `poll`, `drain`, `runForever`, `waitFor`](#цикл-событий)
7. [Таймеры и тайм-ауты: `sleepAsync`, `addTimer`, `withTimeout`](#таймеры-и-тайм-ауты)
8. [Асинхронные сокеты](#асинхронные-сокеты)
9. [Подключение, обнаружение адресов: `connect`, `dial`](#подключение-и-обнаружение-адресов)
10. [События, сигналы и процессы](#события-сигналы-и-процессы)
11. [Диагностика дескрипторов](#диагностика-дескрипторов)
12. [Полный пример: эхо-сервер](#полный-пример-эхо-сервер)
13. [Сводная таблица](#сводная-таблица)

---

## Концепции: Future, async/await, асинхронные процедуры

### `Future[T]`

`Future[T]` — это объект, который **в данный момент** может не содержать значения типа `T`, но получит его (или ошибку) в будущем. Любая асинхронная функция, выполняющая операцию ввода-вывода, немедленно возвращает `Future` и **не блокирует** поток выполнения — однако и не возвращает результат сразу же. Запрос на операцию выполняется, а её результат появится позже, когда диспетчер обработает соответствующее событие.

К future можно прикрепить обратный вызов через `addCallback`, который будет вызван автоматически по завершении:

```nim
var future = socket.recv(100)
future.addCallback(
  proc () =
    echo(future.read)
)
```

Состояние future проверяется процедурами:

- `finished(fut)` — завершилась ли операция (успешно или с ошибкой);
- `failed(fut)` — завершилась ли она именно с ошибкой;
- `read(fut)` — вернуть значение (если есть ошибка — поднять её как исключение).

### Асинхронные процедуры (`{.async.}`)

Процедура, помеченная прагмой `{.async.}`, должна возвращать `Future[T]` или вообще ничего (в последнем случае подразумевается `Future[void]`). Внутри такой процедуры доступно ключевое слово `await`:

```nim
proc fetchAndPrint(socket: AsyncFD) {.async.} =
  let data = await socket.recv(100)   # приостанавливает выполнение до получения данных
  echo "Получено: ", data
```

`await fut`:

- если `fut` ещё не завершён — приостанавливает текущую асинхронную процедуру (но **не** поток!) и передаёт управление диспетчеру, который в это время может выполнять другие асинхронные процедуры;
- когда `fut` завершается, выполнение текущей процедуры продолжается с того же места;
- если `fut` завершился с ошибкой — `await` **повторно поднимает это исключение** в точке вызова, поэтому его можно перехватывать обычным `try/except`.

`await` можно использовать:

- как часть выражения при объявлении переменной: `var data = await socket.recv(100)`;
- как самостоятельное выражение для future типа `void`: `await socket.send("hello")`;
- напрямую с future-объектом: `await someFuture`.

> ⚠️ **Ограничение.** Асинхронные процедуры **не поддерживают изменяемые (`var`) параметры**, например `var int`. Если нужно передавать значение по ссылке — используйте `ref`-типы (`ref int` и т.п.). Кроме того, система эффектов (`{.raises: [].}`) не работает корректно с асинхронными процедурами.

---

## Три способа работы с future

| Процедура       | Контекст вызова          | Блокирующая?                 |
|-----------------|---------------------------|-------------------------------|
| `asyncCheck`    | синхронный и асинхронный  | нет                            |
| `waitFor`       | только синхронный         | блокирует текущий поток        |
| `await`         | только асинхронный        | приостанавливает текущую процедуру (но не поток) |

- **`asyncCheck`** — «запустить и не ждать», но при этом, если future завершится с ошибкой, исключение будет поднято (а не молча проигнорировано). Используется, когда вам не интересен результат, но вы не хотите потерять ошибку. Не ждёт завершения — для этого нужен `waitFor` или `await`.
- **`waitFor`** — прогоняет цикл событий (`poll`) до тех пор, пока указанный `Future` не завершится, и **блокирует текущий поток**. Применяется для вызова асинхронного кода из синхронного контекста (например, из `main`). **Никогда** не используйте `waitFor` внутри `{.async.}`-процедуры — это приведёт к вложенному циклу событий и потенциальным дедлокам.
- **`await`** — используется **только** внутри `{.async.}`-процедур; приостанавливает текущую процедуру, позволяя диспетчеру выполнять другую работу, до завершения future.

> ⚠️ **Никогда не используйте `discard` для future напрямую** — future может содержать ошибку, и `discard` её замолчит. Вместо `discard someAsyncProc()` используйте `asyncCheck someAsyncProc()`. `await` сам по себе уже проверяет наличие ошибки, так что результат `await` можно безопасно отбрасывать.

```nim
# Неправильно — ошибка будет потеряна:
discard sock.send("data")

# Правильно — ошибка будет поднята, если она есть:
asyncCheck sock.send("data")

# В асинхронном контексте — ошибка автоматически проверяется await:
proc worker() {.async.} =
  await sock.send("data")
```

---

## Обработка исключений

Внутри `{.async.}`-процедур исключения обрабатываются так же, как и в обычном коде — через `try`/`except`:

```nim
proc fetchSafely(sock: AsyncFD) {.async.} =
  try:
    let data = await sock.recv(100)
    echo("Получено ", data)
  except:
    echo "Ошибка при получении данных: ", getCurrentExceptionMsg()
```

Альтернативный подход — использовать `yield` вместо `await`. В отличие от `await`, `yield` **не поднимает** исключение автоматически, и состояние ошибки нужно проверять вручную через `failed`:

```nim
proc fetchOrIgnore(sock: AsyncFD) {.async.} =
  var future = sock.recv(100)
  yield future
  if future.failed:
    echo "Операция завершилась с ошибкой, игнорируем"
  else:
    echo "Получено: ", future.read
```

`yield` полезен, когда вы хотите явно решить, как поступить с ошибкой, не прерывая выполнение исключением — например, при обработке множества параллельных future в цикле, где ошибка одной из них не должна прерывать обработку остальных.

---

## Основные типы

### `AsyncFD`

```nim
type AsyncFD* = distinct int
```

Асинхронный файловый/socket-дескриптор. Это «обёртка» над обычным дескриптором ОС (на Unix-системах — `SocketHandle`/`FileHandle`, на Windows — `Handle`), **зарегистрированная в диспетчере**. Большинство процедур модуля (`recv`, `send`, `accept`, `addRead` и т.д.) принимают именно `AsyncFD`, а не «голый» дескриптор — регистрация необходима, чтобы диспетчер мог отслеживать события на этом дескрипторе.

### `Callback`

```nim
type Callback* = proc (fd: AsyncFD): bool {.closure, gcsafe.}
```

Тип обратного вызова для низкоуровневых функций `addRead`, `addWrite`, `addTimer`, `addEvent`, `addSignal`, `addProcess`. Возвращаемое значение указывает диспетчеру, нужно ли **продолжать** наблюдение:

- `true` — снять наблюдение (например, событие было одноразовым);
- `false` — продолжать получать уведомления о событии.

### `AsyncEvent`

Непрозрачный (`ptr`) тип, представляющий объект-событие, которое можно «взвести» (`trigger`) из любого потока — это единственный безопасный для многопоточности способ «разбудить» цикл событий из другого потока.

### `PDispatcher` / `PDispatcherBase`

Тип диспетчера. Содержит очередь таймеров, очередь отложенных вызовов (`callbacks`, заполняемую через `callSoon`) и (на Unix) `selector` — обёртку над `epoll`/`kqueue`/`select`, на Windows — `ioPort` (IO Completion Port) и набор зарегистрированных дескрипторов `handles`.

Как правило, напрямую с `PDispatcher` работать не нужно — для этого есть `getGlobalDispatcher`/`setGlobalDispatcher`.

---

## Управление диспетчером

### `getGlobalDispatcher`

```nim
proc getGlobalDispatcher*(): PDispatcher
```

**Что делает.** Возвращает диспетчер событий **текущего потока** (диспетчер хранится в `{.threadvar.}`, то есть отдельный для каждого потока). Если диспетчер ещё не был создан — создаёт новый автоматически при первом вызове. В большинстве программ вы никогда не вызываете эту процедуру напрямую — она используется внутри `recv`, `send`, `poll` и т.п.

```nim
let disp = getGlobalDispatcher()
echo "Дескриптор диспетчера: ", disp.getIoHandler()
```

### `setGlobalDispatcher`

```nim
proc setGlobalDispatcher*(disp: sink PDispatcher)
```

**Что делает.** Заменяет диспетчер текущего потока на указанный. Полезно в редких случаях, когда требуется полностью «пересоздать» цикл событий (например, в тестах, чтобы изолировать состояние между запусками), либо при ручном создании диспетчера через `newDispatcher()`. Поднимает assertion-ошибку, если в текущем (старом) диспетчере остались необработанные `callbacks`.

```nim
import std/asyncdispatch

# Создать "чистый" диспетчер, например, перед запуском нового теста
setGlobalDispatcher(newDispatcher())
assert getGlobalDispatcher().callbacks.len == 0
```

### `register` / `unregister` / `contains`

```nim
proc register*(fd: AsyncFD)
proc unregister*(fd: AsyncFD)
proc contains*(disp: PDispatcher, fd: AsyncFD): bool
```

**Что делает.** `register` добавляет дескриптор `fd` в диспетчер текущего потока — после этого диспетчер начинает отслеживать события на нём (на Windows это привязка к IO Completion Port, на Unix — добавление в `selector`). `unregister` снимает дескриптор с наблюдения **без** его закрытия (закрытие — отдельная операция, для сокетов это `closeSocket`). `contains` (используется как оператор `in`) позволяет проверить, зарегистрирован ли дескриптор в данном диспетчере.

Большинство высокоуровневых процедур (`createAsyncNativeSocket`, `acceptAddr`) уже вызывают `register` за вас — вручную это требуется в основном при оборачивании «сырых» дескрипторов сторонних библиотек.

```nim
import std/[asyncdispatch, nativesockets]

let raw = createNativeSocket()
let fd = raw.AsyncFD
register(fd)
assert fd in getGlobalDispatcher()

unregister(fd)
assert fd notin getGlobalDispatcher()
```

### `callSoon`

```nim
proc callSoon*(cbproc: proc () {.gcsafe.})
```

**Что делает.** Планирует вызов `cbproc` как можно скорее — он будет выполнен, как только управление вернётся в цикл событий (то есть после текущего «кванта» выполнения, но до обработки таймеров/IO следующей итерации). Используется как примитив более низкого уровня, на котором, в частности, строится реализация `Future` из `asyncfutures` (вызов callback'ов future всегда проходит через `callSoon`, чтобы не приводить к глубокой рекурсии).

```nim
import std/asyncdispatch

var executed = false
callSoon(proc () = executed = true)
assert not executed
poll(0)               # выполнит запланированный через callSoon код
assert executed
```

---

## Цикл событий

### `poll`

```nim
proc poll*(timeout = 500)
```

**Что делает.** Выполняет **один** проход цикла событий: один вызов нижележащего системного примитива (`epoll_wait`, `kqueue`, `GetQueuedCompletionStatus` и т.п.) с тайм-аутом `timeout` миллисекунд, обрабатывает истёкшие таймеры и отложенные `callSoon`-вызовы. Если ни одного дескриптора, таймера или отложенного вызова не зарегистрировано — поднимает `ValueError` («No handles or timers registered in dispatcher»).

`poll` — это «движущая сила» всего асинхронного кода: если ничего не вызывает `poll` (прямо или через `waitFor`/`runForever`), ни одна асинхронная операция никогда не завершится, какой бы готовой она физически ни была.

```nim
import std/asyncdispatch

var fut = sleepAsync(10)
while not fut.finished:
  poll()   # без этого цикл fut никогда не завершится
echo "Готово"
```

### `drain`

```nim
proc drain*(timeout = 500)
```

**Что делает.** В отличие от `poll`, который делает ровно один проход, `drain` **продолжает обрабатывать события**, пока они доступны, до тех пор, пока не истечёт суммарный `timeout`, либо пока не закончатся ожидающие операции (`hasPendingOperations() == false`). Удобно для «вычистки» всех накопившихся событий перед завершением программы или перед переключением диспетчера.

```nim
import std/asyncdispatch

# Дождаться обработки всех накопившихся операций (но не дольше 1 секунды)
drain(1000)
```

### `runForever`

```nim
proc runForever*()
```

**Что делает.** Бесконечный цикл `while true: poll()`. Используется в серверных приложениях, которые должны работать «вечно», обслуживая входящие соединения. Завершить такой цикл «штатно» из самого процесса нельзя — обычно используют исключение/`quit` или сигнал ОС.

```nim
import std/[asyncdispatch, asyncnet]

proc serve() {.async.} =
  let server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(8080))
  server.listen()
  while true:
    let client = await server.accept()
    asyncCheck handleClient(client)   # handleClient — ваша {.async.}-процедура

asyncCheck serve()
runForever()
```

### `waitFor`

```nim
proc waitFor*[T](fut: Future[T]): T
```

**Что делает.** Запускает `poll()` в цикле до тех пор, пока `fut` не завершится, затем возвращает `fut.read` (то есть либо значение типа `T`, либо поднимает сохранённое исключение, если future завершился с ошибкой). Это **точка входа** для запуска асинхронного кода из синхронной программы — например, из `proc main()` без `{.async.}`.

```nim
import std/asyncdispatch

proc fetchData(): owned(Future[string]) {.async.} =
  await sleepAsync(100)
  return "результат"

let result = waitFor fetchData()
assert result == "результат"
```

> ⚠️ Никогда не вызывайте `waitFor` внутри другой `{.async.}`-процедуры: это запустит **второй**, вложенный цикл `poll`, что может привести к неожиданному порядку выполнения, повторной обработке событий или тупикам. Внутри асинхронного кода всегда используйте `await`.

### `hasPendingOperations`

```nim
proc hasPendingOperations*(): bool
```

**Что делает.** Возвращает `true`, если у глобального диспетчера есть хотя бы один зарегистрированный дескриптор, активный таймер или отложенный (`callSoon`) вызов. Используется внутри `drain`, а также может служить условием для собственных циклов: «работать, пока есть, что обрабатывать».

```nim
import std/asyncdispatch

while hasPendingOperations():
  poll()
echo "Все операции завершены"
```

---

## Таймеры и тайм-ауты

### `sleepAsync`

```nim
proc sleepAsync*(ms: int | float): owned(Future[void])
```

**Что делает.** Возвращает `Future[void]`, который завершится через `ms` миллисекунд (поддерживается как целочисленное, так и дробное значение — для дробного внутреннее разрешение увеличивается до наносекунд). Это асинхронный, **неблокирующий** аналог `os.sleep` — во время ожидания диспетчер может выполнять другую работу.

```nim
import std/asyncdispatch

proc demo() {.async.} =
  echo "Начало"
  await sleepAsync(100)   # пауза в 100 мс, не блокируя другие задачи
  echo "Прошло 100 мс"

waitFor demo()
```

### `addTimer`

```nim
proc addTimer*(timeout: int, oneshot: bool, cb: Callback)
```

**Что делает.** Низкоуровневый аналог `sleepAsync`, работающий через `Callback`, а не `Future`. Регистрирует таймер с периодом `timeout` мс:

- `oneshot = true` — сработает **один раз**;
- `oneshot = false` — будет срабатывать **периодически**, каждые `timeout` мс, до тех пор, пока `cb` не вернёт `true`.

Используется, когда нужен периодический опрос/heartbeat без накладных расходов на пересоздание `Future` на каждой итерации.

```nim
import std/asyncdispatch

var ticks = 0
addTimer(50, oneshot = false, cb = proc (fd: AsyncFD): bool =
  inc ticks
  echo "тик ", ticks
  result = ticks >= 3   # вернуть true после 3-го срабатывания, чтобы остановить таймер
)

while ticks < 3:
  poll()
```

### `withTimeout`

```nim
proc withTimeout*[T](fut: Future[T], timeout: int): owned(Future[bool])
```

**Что делает.** Оборачивает `fut` в новый `Future[bool]`, который завершится **раньше** из двух событий:

- если `fut` завершится первым — результат `true` (а сам `fut`, в зависимости от исхода, можно прочитать через `fut.read`/`fut.error` отдельно);
- если истечёт `timeout` миллисекунд раньше — результат `false`, при этом исходный `fut` **продолжает** работать в фоне (он не отменяется — у `Future` в Nim в принципе нет отмены), но его дальнейшие callback'и будут сброшены этой обёрткой.

Это базовый строительный блок для реализации тайм-аутов на операциях, которые сами по себе тайм-аут не поддерживают (например, `recv`).

```nim
import std/asyncdispatch

proc demo() {.async.} =
  let fut = sleepAsync(1000)            # "медленная" операция на 1 секунду
  if await withTimeout(fut, 100):       # ждём не больше 100 мс
    echo "успели вовремя"
  else:
    echo "тайм-аут!"

waitFor demo()  # выведет "тайм-аут!"
```

---

## Асинхронные сокеты

Эта часть модуля предоставляет низкоуровневые асинхронные операции над `AsyncFD`. На практике в прикладном коде чаще используется `std/asyncnet` (объектная обёртка `AsyncSocket`), но именно процедуры из `asyncdispatch` являются его фундаментом.

### `createAsyncNativeSocket`

```nim
proc createAsyncNativeSocket*(domain: Domain = Domain.AF_INET,
                              sockType: SockType = SOCK_STREAM,
                              protocol: Protocol = IPPROTO_TCP,
                              inheritable = defined(nimInheritHandles)): AsyncFD

proc createAsyncNativeSocket*(domain: cint, sockType: cint,
                              protocol: cint,
                              inheritable = defined(nimInheritHandles)): AsyncFD
```

**Что делает.** Создаёт новый неблокирующий («асинхронный») сокет, переводит его в неблокирующий режим (`setBlocking(false)`) и **автоматически регистрирует** его в диспетчере текущего потока (`register`). Возвращает `osInvalidSocket.AsyncFD`, если создать сокет на уровне ОС не удалось. Первая перегрузка принимает «удобные» enum-типы `Domain`/`SockType`/`Protocol` из `std/nativesockets`, вторая — «сырые» значения `cint` для совместимости с низкоуровневым кодом.

Параметр `inheritable` управляет тем, будет ли сокет унаследован дочерними процессами (по умолчанию — нет, что соответствует обычной практике для серверных сокетов).

```nim
import std/[asyncdispatch, nativesockets, net]

let sock = createAsyncNativeSocket(Domain.AF_INET, SockType.SOCK_STREAM, IPPROTO_TCP)
assert sock in getGlobalDispatcher()
```

### `recv`

```nim
proc recv*(socket: AsyncFD, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[string])
```

**Что делает.** Читает **до** `size` байт из `socket`. Future завершится, когда: будут получены все запрошенные данные, получена их часть, либо при отключении сокета (в последнем случае — со значением `""`). Флаг `SocketFlag.Peek` **не поддерживается на Windows**. Флаг `SafeDisconn` (включён по умолчанию) подавляет типичные «ошибки» разрыва соединения, превращая их в штатное завершение с пустой строкой, а не в исключение — это упрощает обработку клиентов, которые просто закрыли соединение.

```nim
import std/asyncdispatch

proc echoOnce(sock: AsyncFD) {.async.} =
  let data = await recv(sock, 1024)
  if data.len == 0:
    echo "Клиент отключился"
  else:
    echo "Получено: ", data
```

### `recvInto`

```nim
proc recvInto*(socket: AsyncFD, buf: pointer, size: int,
               flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

**Что делает.** То же самое, что `recv`, но пишет данные напрямую в предварительно выделенный буфер `buf` (как минимум `size` байт), а возвращает `Future[int]` — **количество** прочитанных байт (`0` при отключении). Позволяет избежать лишних аллокаций строки, если у вас уже есть буфер (например, в пуле буферов).

```nim
import std/asyncdispatch

proc readChunk(sock: AsyncFD): owned(Future[int]) =
  var buf = newString(4096)
  recvInto(sock, addr buf[0], buf.len)
```

### `send`

```nim
proc send*(socket: AsyncFD, data: string,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])

proc send*(socket: AsyncFD, buf: pointer, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

**Что делает.** Отправляет данные в `socket`; future завершается, когда **все** данные отправлены. Перегрузка со `string` — самая удобная для прикладного кода; перегрузка с `pointer`/`size` работает с «сырой» памятью.

> ⚠️ Для перегрузки с `pointer`: если `buf` указывает на управляемый GC объект, нужно самостоятельно удерживать его живым (`GC_ref`/`GC_unref`) на время асинхронной операции — иначе сборщик мусора может освободить память до завершения отправки. Перегрузка со `string` делает это за вас автоматически.

```nim
import std/asyncdispatch

proc reply(sock: AsyncFD) {.async.} =
  await send(sock, "HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nOK")
```

### `sendTo` / `recvFromInto`

```nim
proc sendTo*(socket: AsyncFD, data: pointer, size: int, saddr: ptr SockAddr,
             saddrLen: SockLen,
             flags = {SocketFlag.SafeDisconn}): owned(Future[void])

proc recvFromInto*(socket: AsyncFD, data: pointer, size: int,
                   saddr: ptr SockAddr, saddrLen: ptr SockLen,
                   flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

**Что делает.** Версии для **датаграммных** (UDP) сокетов, не требующих установленного соединения. `sendTo` отправляет данные конкретному адресу `saddr`; `recvFromInto` принимает одну датаграмму, записывая данные в `data` и заполняя `saddr`/`saddrLen` адресом отправителя. В прикладном коде обычно удобнее работать через `std/asyncnet` (`sendTo`/`recvFrom` с типом `IpAddress`), но при необходимости работы напрямую с `SockAddr` (например, для нестандартных доменов сокетов) эти процедуры — рабочий низкоуровневый инструмент.

### `acceptAddr` / `accept`

```nim
proc acceptAddr*(socket: AsyncFD, flags = {SocketFlag.SafeDisconn},
                 inheritable = defined(nimInheritHandles)):
    owned(Future[tuple[address: string, client: AsyncFD]])

proc accept*(socket: AsyncFD,
             flags = {SocketFlag.SafeDisconn},
             inheritable = defined(nimInheritHandles)): owned(Future[AsyncFD])
```

**Что делает.** Принимает новое входящее соединение на прослушивающем сокете `socket`. `acceptAddr` возвращает **и** клиентский сокет (`AsyncFD`, уже автоматически зарегистрированный в диспетчере), **и** адрес клиента в виде строки. `accept` — упрощённая обёртка, возвращающая только клиентский сокет (реализована через `acceptAddr`, отбрасывая адрес).

Если соединяющийся клиент отключился прямо во время `accept` и установлен флаг `SafeDisconn` (по умолчанию включён) — ошибка не поднимается, а `accept` автоматически вызывается повторно.

```nim
import std/asyncdispatch

proc serverLoop(server: AsyncFD) {.async.} =
  while true:
    let (address, client) = await acceptAddr(server)
    echo "Подключение от ", address
    asyncCheck handleClient(client)
```

### `closeSocket`

```nim
proc closeSocket*(socket: AsyncFD)
```

**Что делает.** Закрывает сокет на уровне ОС **и** снимает его регистрацию в диспетчере (комбинация `close` + `unregister`). После вызова `socket` нельзя использовать ни в каких операциях модуля.

```nim
import std/asyncdispatch

proc cleanup(sock: AsyncFD) =
  closeSocket(sock)
  assert sock notin getGlobalDispatcher()
```

### `setInheritable`

```nim
proc setInheritable*(fd: AsyncFD, inheritable: bool): bool
```

**Что делает.** Управляет тем, будет ли дескриптор `fd` унаследован дочерними процессами (через `fork`/`exec` на Unix, или `CreateProcess` на Windows). Возвращает `true` при успехе. Доступность на конкретной платформе можно проверить через `declared(setInheritable)`. Полезно, если вашему дочернему процессу нужно «передать» открытый сокет.

---

## Подключение и обнаружение адресов

### `connect`

```nim
proc connect*(socket: AsyncFD, address: string, port: Port,
              domain = Domain.AF_INET): owned(Future[void])
```

**Что делает.** Устанавливает соединение с `address:port` через уже созданный сокет `socket`, используя указанный `domain` (семейство адресов — IPv4/IPv6). Это «низкоуровневый» вариант: вы должны самостоятельно подобрать `domain`, совпадающий с тем, для которого был создан сокет (на Unix-системах при несовпадении сработает `assert`).

### `dial`

```nim
proc dial*(address: string, port: Port,
           protocol: Protocol = IPPROTO_TCP): owned(Future[AsyncFD])
```

**Что делает.** Высокоуровневая альтернатива `connect`: **сама** выполняет разрешение DNS-имени `address` (через `getAddrInfo`) и перебирает все полученные адреса (IPv4 **и** IPv6), пытаясь подключиться по каждому из них по очереди, до первого успеха. Возвращает уже подключённый и зарегистрированный в диспетчере `AsyncFD`, готовый к работе. В отличие от `connect`, не требует, чтобы сокет уже существовал и чтобы вы заранее знали, какая версия IP будет использоваться — `dial` сам создаёт сокет подходящего типа.

```nim
import std/[asyncdispatch, net]

proc fetchExample() {.async.} =
  let sock = await dial("example.com", Port(80))
  await send(sock, "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  let response = await recv(sock, 4096)
  echo response
  closeSocket(sock)

waitFor fetchExample()
```

> **`dial` против `connect`+`createAsyncNativeSocket`.** Если адрес сервера может быть как IPv4, так и IPv6 (типичная ситуация для доменных имён), и вам не важно, какой конкретно сокет в итоге будет создан — используйте `dial`. Если же вы работаете с уже существующим сокетом конкретного домена (например, доменным сокетом Unix или уже принятым через `acceptAddr` сокетом) — используйте `connect`.

---

## События, сигналы и процессы

Эта группа процедур позволяет встраивать в цикл событий произвольные источники уведомлений ОС — пользовательские события (для пробуждения из другого потока), сигналы Unix и завершение внешних процессов.

### `newAsyncEvent` / `trigger` / `close` / `addEvent`

```nim
proc newAsyncEvent*(): AsyncEvent
proc trigger*(ev: AsyncEvent)
proc close*(ev: AsyncEvent)
proc addEvent*(ev: AsyncEvent, cb: Callback)
```

**Что делает.** `newAsyncEvent` создаёт новый объект-событие (потокобезопасный примитив синхронизации). `addEvent` регистрирует callback `cb`, который будет вызван, когда событие переходит в «взведённое» состояние. `trigger` переводит событие в это состояние — **его можно безопасно вызывать из любого потока**, в том числе из того, который не запускал цикл событий, что делает `AsyncEvent` основным механизмом «разбудить» диспетчер снаружи. `close` освобождает ресурсы события.

```nim
import std/[asyncdispatch, os]

let ev = newAsyncEvent()
var notified = false
addEvent(ev, proc (fd: AsyncFD): bool =
  notified = true
  result = true   # снять наблюдение после первого срабатывания
)

# Из другого потока (или просто синхронно здесь для примера):
trigger(ev)

while not notified:
  poll()
close(ev)
```

### `addSignal` *(только Unix)*

```nim
proc addSignal*(signal: int, cb: Callback)
```

**Что делает.** Регистрирует `cb`, который будет вызван при получении процессом сигнала Unix `signal` (например, `SIGTERM`, `SIGUSR1`). Позволяет обрабатывать сигналы в рамках цикла событий, не прерывая выполнение асинхронного кода стандартным образом обработки сигналов ОС.

```nim
import std/[asyncdispatch, posix]

addSignal(SIGUSR1.int, proc (fd: AsyncFD): bool =
  echo "Получен SIGUSR1"
  result = false   # продолжать слушать сигнал
)
```

### `addProcess`

```nim
proc addProcess*(pid: int, cb: Callback)
```

**Что делает.** Регистрирует `cb`, который будет вызван, когда процесс с идентификатором `pid` завершится. Удобно для асинхронного ожидания дочерних процессов без блокирующего `os.waitForExit`.

### `addRead` / `addWrite` *(низкий уровень, в основном Unix; на Windows — с ограничениями)*

```nim
proc addRead*(fd: AsyncFD, cb: Callback)
proc addWrite*(fd: AsyncFD, cb: Callback)
```

**Что делает.** Регистрируют callback, вызываемый при готовности `fd` к чтению/записи соответственно. Это самый низкий уровень модуля — используется для адаптации сторонних, «синхронных по своей природе» файловых дескрипторов (трубы, устройства, сторонние библиотеки) к циклу событий. `cb` должен возвращать `true`, чтобы прекратить наблюдение, или `false`, чтобы продолжать получать уведомления.

> ⚠️ **На Windows** это не «родной» механизм IOCP, а эмуляция через `RegisterWaitForSingleObject` — используйте только если действительно необходимо (как правило, при портировании Unix-ориентированных библиотек). Если вы используете `addRead`/`addWrite` на Windows для сокета, **не смешивайте** это с `recv`/`send`/`accept` из этого же модуля — используйте низкоуровневые `nativesockets.recv`/`nativesockets.send`/`nativesockets.accept`.

---

## Диагностика дескрипторов

### `activeDescriptors`

```nim
proc activeDescriptors*(): int {.inline.}
```

**Что делает.** Возвращает текущее количество активных файловых дескрипторов, отслеживаемых диспетчером **текущего потока**. Дешёвая операция — не делает системных вызовов (на Windows читает размер набора `handles`, на Unix — счётчик `selector`).

```nim
import std/asyncdispatch

echo "Активных дескрипторов: ", activeDescriptors()
```

### `maxDescriptors`

```nim
proc maxDescriptors*(): int {.raises: OSError.}
```

**Что делает.** Возвращает **максимальное** количество файловых дескрипторов, которое может открыть текущий процесс (системный лимит, `RLIMIT_NOFILE` на Unix; на Windows возвращается приблизительная константа). В отличие от `activeDescriptors`, делает системный вызов. Полезно для расчёта допустимого размера пулов соединений: если `activeDescriptors() близко к maxDescriptors()`, пора отказывать новым соединениям, а не сваливаться с `EMFILE`.

```nim
import std/asyncdispatch

let limit = maxDescriptors()
if activeDescriptors() > limit - 100:
  echo "Приближаемся к лимиту дескрипторов!"
```

### `getFuturesInProgress` и отслеживание «зависших» future

При компиляции с флагом `-d:futureLogging` модуль `asyncfutures` (реэкспортируемый отсюда) ведёт учёт всех незавершённых `Future`. Процедура `getFuturesInProgress` (из `asyncfutures`) возвращает их список вместе со стек-трейсами момента создания — это незаменимый инструмент при диагностике утечек памяти, вызванных «забытыми» future, которые никогда не были завершены.

```sh
nim c -d:futureLogging --threads:on myapp.nim
```

---

## Полный пример: эхо-сервер

Пример объединяет основные процедуры модуля: `dial`/`acceptAddr` для сетевого ввода-вывода, `recv`/`send` для обмена данными, `asyncCheck`/`runForever` для управления циклом событий, и `sleepAsync` для демонстрации неблокирующей задержки.

```nim
import std/[asyncdispatch, nativesockets, net]

proc handleClient(client: AsyncFD) {.async.} =
  defer: closeSocket(client)
  while true:
    let line = await recv(client, 1024)
    if line.len == 0:
      echo "Клиент отключился"
      break
    await sleepAsync(10)              # имитация небольшой задержки обработки
    await send(client, "echo: " & line)

proc serve(port: Port) {.async.} =
  let server = createAsyncNativeSocket()
  server.SocketHandle.setSockOptInt(SOL_SOCKET, SO_REUSEADDR, 1)
  server.SocketHandle.bindAddr(port)
  server.SocketHandle.listen()

  while true:
    let (address, client) = await acceptAddr(server)
    echo "Новое подключение от ", address
    asyncCheck handleClient(client)

asyncCheck serve(Port(7777))
runForever()
```

---

## Сводная таблица

| Задача | Процедура(ы) |
|---|---|
| Запустить асинхронный код из синхронного `main` | `waitFor` |
| Запустить «фоновую» задачу, не теряя ошибок | `asyncCheck` |
| Подождать другую future внутри `{.async.}` | `await` |
| Один проход цикла событий | `poll` |
| Обработать все накопившиеся события | `drain` |
| Бесконечный цикл событий (сервер) | `runForever` |
| Неблокирующая задержка | `sleepAsync` |
| Периодический таймер на callback'ах | `addTimer` |
| Ограничить future тайм-аутом | `withTimeout` |
| Создать асинхронный сокет | `createAsyncNativeSocket` |
| Подключиться по имени хоста (IPv4/IPv6) | `dial` |
| Подключиться к уже известному адресу | `connect` |
| Принять соединение | `acceptAddr` / `accept` |
| Прочитать/записать данные сокета | `recv`, `recvInto`, `send` |
| UDP-обмен | `sendTo`, `recvFromInto` |
| Закрыть сокет и снять регистрацию | `closeSocket` |
| Зарегистрировать «сырой» дескриптор | `register` / `unregister` / `in` |
| Разбудить цикл событий из другого потока | `newAsyncEvent` + `trigger` + `addEvent` |
| Подождать сигнал ОС (Unix) | `addSignal` |
| Подождать завершения процесса | `addProcess` |
| Отложенный вызов без задержки | `callSoon` |
| Сколько дескрипторов открыто / можно открыть | `activeDescriptors`, `maxDescriptors` |
| Проверить, есть ли незавершённая работа | `hasPendingOperations` |
