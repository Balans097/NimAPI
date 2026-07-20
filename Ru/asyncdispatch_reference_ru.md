# std/asyncdispatch — справочник модуля

> **Импорт:** `import std/asyncdispatch`
> **Область применения:** асинхронный ввод-вывод (сокеты, таймеры, сигналы, дочерние процессы), диспетчер событий и механизм `Future`/`{.async.}`/`await`.

Модуль реализует событийный цикл (диспетчер), поверх которого построены асинхронные операции. Диспетчер опрашивается процедурой `poll` (или тем, что вызывает `poll` за вас — `waitFor`, `drain`, `runForever`). Каждая асинхронная операция представлена объектом `Future[T]` — значением, которого ещё нет, но которое может появиться позже; готовность проверяется через `finished(fut)`, успех или ошибка — через `failed(fut)`.

Важные сквозные конвенции модуля:

- Реализация диспетчера platform-specific: на Windows это IOCP (`CreateIoCompletionPort`), на Linux/BSD/macOS — `select`/`epoll`/`kqueue` через `std/selectors`. Публичный API процедур (`recv`, `send`, `accept`, `connect`, `addRead`, `addWrite` и т.п.) одинаков на обеих ветках, но их внутренняя механика различна — это оговорено в разделе "Разбор реализации" для каждой процедуры.
- Три основных способа обработки `Future`: `asyncCheck` (не ждёт и не возвращает значение, только поднимает исключение при ошибке), `waitFor` (блокирует текущий поток до завершения, вызывается только из синхронного кода), `await` (приостанавливает текущую `{.async.}`-процедуру, не блокируя поток целиком — используется внутри асинхронных процедур).
- Процедуры `addRead`/`addWrite`/`addTimer`/`addProcess`/`addSignal` работают в терминах callback-функций (`Callback = proc (fd: AsyncFD): bool`), а не `Future` — это низкоуровневый слой, на котором построены `recv`, `send`, `accept` и другие high-level процедуры.

---

## Оглавление

1. [Типы и базовые понятия](#типы-и-базовые-понятия)
   1. [`AsyncFD`](#asyncfd)
   2. [`Callback`](#callback)
   3. [`AsyncEvent`](#asyncevent)
2. [Диспетчер: получение и регистрация дескрипторов](#диспетчер-получение-и-регистрация-дескрипторов)
   1. [`getGlobalDispatcher`](#getglobaldispatcher)
   2. [`setGlobalDispatcher`](#setglobaldispatcher)
   3. [`hasPendingOperations`](#haspendingoperations)
   4. [`register`](#register)
   5. [`unregister`](#unregister)
   6. [`contains`](#contains)
3. [Цикл событий: опрос и ожидание](#цикл-событий-опрос-и-ожидание)
   1. [`poll`](#poll)
   2. [`drain`](#drain)
   3. [`runForever`](#runforever)
   4. [`waitFor`](#waitfor)
   5. [`callSoon`](#callsoon)
4. [Подписка на события файловых дескрипторов](#подписка-на-события-файловых-дескрипторов)
   1. [`addRead`](#addread)
   2. [`addWrite`](#addwrite)
   3. [`addTimer`](#addtimer)
   4. [`addProcess`](#addprocess)
   5. [`addSignal`](#addsignal)
5. [`AsyncEvent` — пользовательские события](#asyncevent--пользовательские-события)
   1. [`newAsyncEvent`](#newasyncevent)
   2. [`trigger`](#trigger)
   3. [`close` (событие)](#close-событие)
   4. [`addEvent`](#addevent)
6. [Сокеты: создание, приём соединений, обмен данными](#сокеты-создание-приём-соединений-обмен-данными)
   1. [`createAsyncNativeSocket`](#createasyncnativesocket)
   2. [`closeSocket`](#closesocket)
   3. [`setInheritable`](#setinheritable)
   4. [`recv`](#recv)
   5. [`recvInto`](#recvinto)
   6. [`send`](#send)
   7. [`sendTo`](#sendto)
   8. [`recvFromInto`](#recvfrominto)
   9. [`acceptAddr`](#acceptaddr)
   10. [`accept`](#accept)
7. [Установление исходящих соединений](#установление-исходящих-соединений)
   1. [`dial`](#dial)
   2. [`connect`](#connect)
8. [Тайминги и ожидания](#тайминги-и-ожидания)
   1. [`sleepAsync`](#sleepasync)
   2. [`withTimeout`](#withtimeout)
9. [Потоковые данные](#потоковые-данные)
   1. [`readAll`](#readall)
10. [Диагностика ресурсов](#диагностика-ресурсов)
    1. [`activeDescriptors`](#activedescriptors)
    2. [`maxDescriptors`](#maxdescriptors)
11. [Практические рецепты](#практические-рецепты)
12. [Краткая таблица](#краткая-таблица)
13. [Сводка: какую процедуру выбрать](#сводка-какую-процедуру-выбрать)

---

## Типы и базовые понятия

### `AsyncFD`

```nim
type AsyncFD* = distinct int
```

**Что делает.** Это отдельный (`distinct`) тип-обёртка над целочисленным дескриптором сокета/файла. `distinct int` защищает от случайного смешивания асинхронного дескриптора с обычным `int` или `SocketHandle` — конвертация возможна только явным приведением типа.

**Разбор реализации.** На Windows `AsyncFD` регистрируется в IOCP через `createIoCompletionPort`, на Unix-подобных системах — через `selector` (`epoll`/`kqueue`/`select`, обёрнутые `std/selectors`). Внешне интерфейс один и тот же, поэтому код, написанный поверх `AsyncFD`, переносим между платформами.

- **Параметры:** отсутствуют — это тип, а не процедура.

```nim
var fd: AsyncFD = createAsyncNativeSocket()
assert int(fd) >= 0  # дескриптор — валидный неотрицательный int
```

---

### `Callback`

```nim
type Callback* = proc (fd: AsyncFD): bool {.closure, gcsafe.}
```

**Что делает.** Тип функции обратного вызова для низкоуровневых процедур `addRead`/`addWrite`/`addTimer`/`addProcess`/`addSignal`/`addEvent`. Возвращаемое значение управляет подпиской: `true` означает "снять callback с наблюдения", `false` — "оставить callback, вызвать его снова при следующем срабатывании события".

**Разбор реализации.** Это соглашение (`bool` как "снять/оставить") единообразно используется во всех диспетчерах: на Unix-ветке callback хранится в списке `readList`/`writeList` внутри данных селектора и удаляется из списка, если вернул `true`; на Windows-ветке аналогичную роль играет повторная регистрация `RegisterWaitForSingleObject`.

- **Параметры:**
  - `fd: AsyncFD` — дескриптор, на котором сработало событие.

```nim
var callsLeft = 3
proc countdownCb(fd: AsyncFD): bool =
  dec(callsLeft)
  result = callsLeft <= 0  # true — прекратить наблюдение после третьего срабатывания
```

---

### `AsyncEvent`

```nim
when defined(windows):
  type AsyncEvent* = ptr AsyncEventImpl
else:
  type AsyncEvent* = distinct SelectEvent
```

**Что делает.** Пользовательское, программно взводимое событие ("самодельный" сигнал), не привязанное к конкретному сокету или таймеру. Создаётся один раз (`newAsyncEvent`), затем на него подписываются (`addEvent`), а взводится оно вручную из любого места кода (`trigger`) — типичный сценарий: сигнал остановки диспетчера из другого потока.

**Разбор реализации.** На Windows это именованный объект ядра (`CreateEvent`), на который навешивается `RegisterWaitForSingleObject`; на Unix — обёртка над `SelectEvent` из `std/selectors`, зарегистрированным в селекторе как ещё один наблюдаемый дескриптор. Оба варианта дают одинаковую наблюдаемую семантику: `trigger` будит диспетчер, даже если тот "спит" в `poll`.

- **Параметры:** отсутствуют — это тип.

```nim
let stopEvent = newAsyncEvent()
proc onStop(fd: AsyncFD): bool =
  echo "получен сигнал остановки"
  result = true
addEvent(stopEvent, onStop)
trigger(stopEvent)
```

---

## Диспетчер: получение и регистрация дескрипторов

### `getGlobalDispatcher`

```nim
proc getGlobalDispatcher*(): PDispatcher
```

**Что делает.** Возвращает диспетчер текущего потока, создавая его при первом обращении. Диспетчер хранится в thread-local переменной, поэтому у каждого потока — собственный независимый диспетчер и собственный набор зарегистрированных дескрипторов, таймеров и отложенных callback'ов.

**Разбор реализации.** Ленивая инициализация через проверку `isNil` — идиома "создать при первом использовании". Побочный эффект вызова `setGlobalDispatcher` внутри — регистрация функции `callSoon` как обработчика отложенных вызовов для модуля `std/asyncfutures`, чтобы `Future`-callback'и планировались тем же диспетчером.

- **Параметры:** нет.

```nim
let disp1 = getGlobalDispatcher()
let disp2 = getGlobalDispatcher()
assert disp1 == disp2  # повторный вызов возвращает тот же диспетчер потока
```

---

### `setGlobalDispatcher`

```nim
proc setGlobalDispatcher*(disp: PDispatcher)
```

**Что делает.** Принудительно заменяет диспетчер текущего потока на переданный. Нужен редко — например, при написании собственного пула потоков, где каждый воркер должен явно создать и подставить свой диспетчер до начала работы с сокетами.

**Разбор реализации.** На Windows-ветке перед заменой проверяется `assert callbacks.len == 0` — предполагается, что старый диспетчер к моменту замены пуст, иначе отложенные вызовы будут потеряны. Явное предупреждение: замена диспетчера "на лету", когда в нём уже есть незавершённые операции, — источник трудноуловимых утечек и потерянных callback'ов.

- **Параметры:**
  - `disp: PDispatcher` — новый диспетчер, обычно созданный через `newDispatcher()`.

```nim
import std/asyncdispatch

let customDisp = newDispatcher()
setGlobalDispatcher(customDisp)
assert getGlobalDispatcher() == customDisp
```

---

### `hasPendingOperations`

```nim
proc hasPendingOperations*(): bool
```

**Что делает.** Сообщает, есть ли у глобального диспетчера незавершённая работа: зарегистрированные дескрипторы, ожидающие таймеры или отложенные callback'и. Используется как условие остановки циклов `poll`/`drain` — если операций больше нет, дальнейший опрос диспетчера не имеет смысла (и `poll` на пустом диспетчере поднимет исключение).

**Разбор реализации.** Простая проверка трёх счётчиков (`handles`/`timers`/`callbacks` на Windows, `selector`/`timers`/`callbacks` на Unix) без побочных эффектов — O(1) на обеих платформах.

- **Параметры:** нет.

```nim
assert hasPendingOperations() == false  # свежий диспетчер пуст
discard sleepAsync(10)
assert hasPendingOperations() == true   # таймер добавил незавершённую операцию
```

---

### `register`

```nim
proc register*(fd: AsyncFD)
```

**Что делает.** Регистрирует уже созданный (обычно нативный, "сырой") дескриптор в диспетчере текущего потока, делая его доступным для `addRead`/`addWrite`/`recv`/`send` и остальных операций модуля. Без регистрации диспетчер ничего не знает о дескрипторе и не сможет доставить по нему события.

**Разбор реализации.** На Windows регистрация — это привязка дескриптора к порту завершения ввода-вывода (`createIoCompletionPort`) и добавление в множество `handles`; на Unix — добавление записи в `selector` с пустыми списками callback'ов на чтение/запись. `createAsyncNativeSocket` вызывает `register` автоматически, поэтому явный вызов нужен в основном для дескрипторов, полученных извне (например, принятых через `accept4` вручную).

- **Параметры:**
  - `fd: AsyncFD` — дескриптор для регистрации.

```nim
import std/nativesockets

let rawSocket = createNativeSocket(Domain.AF_INET, SockType.SOCK_STREAM, Protocol.IPPROTO_TCP)
setBlocking(rawSocket, false)
let fd = AsyncFD(rawSocket)
register(fd)
assert contains(getGlobalDispatcher(), fd)
```

---

### `unregister`

```nim
proc unregister*(fd: AsyncFD)
proc unregister*(ev: AsyncEvent)
```

**Что делает.** Снимает дескриптор (или пользовательское событие) с наблюдения диспетчера. После вызова диспетчер больше не будет доставлять по этому `fd`/`ev` никаких событий; сам сокет при этом не закрывается — для закрытия с одновременным снятием подписки предназначена `closeSocket`.

**Разбор реализации.** На Unix-ветке это прямой вызов `unregister` селектора; на Windows аналогичной по названию публичной процедуры для `fd` нет в общей части (снятие происходит внутри `closeSocket`/при завершении callback'ов), что отражает разницу моделей: IOCP привязывает дескриптор к порту почти необратимо, тогда как `selectors` позволяет дёшево добавлять и убирать записи.

- **Параметры:**
  - `fd: AsyncFD` **или** `ev: AsyncEvent` — то, что снимается с наблюдения.

```nim
let fd = createAsyncNativeSocket()
assert contains(getGlobalDispatcher(), fd)
unregister(fd)
assert not contains(getGlobalDispatcher(), fd)
```

---

### `contains`

```nim
proc contains*(disp: PDispatcher, fd: AsyncFD): bool
```

**Что делает.** Проверяет, зарегистрирован ли дескриптор `fd` в диспетчере `disp`. Используется как оператор `in`: `fd in disp`.

**Разбор реализации.** На Unix — проверка `fd.SocketHandle in disp.selector`, то есть делегирование операции `contains` селектора; на Windows — `fd in disp.handles` (проверка в `HashSet`). В обоих случаях сложность амортизированная O(1).

- **Параметры:**
  - `disp: PDispatcher` — диспетчер, в котором ищем.
  - `fd: AsyncFD` — искомый дескриптор.

```nim
let disp = getGlobalDispatcher()
let fd = createAsyncNativeSocket()
assert contains(disp, fd)
assert fd in disp  # эквивалентная инфиксная форма того же вызова
```

---

## Цикл событий: опрос и ожидание

### `poll`

```nim
proc poll*(timeout = 500)
```

**Что делает.** Однократно опрашивает диспетчер: ждёт наступления хотя бы одного события (истечения таймера, готовности сокета, срабатывания сигнала) не дольше `timeout` миллисекунд и обрабатывает все события, накопившиеся к этому моменту. Если у диспетчера нет ни одной зарегистрированной операции — поднимает `ValueError`.

**Разбор реализации.** Единственная строка тела — `discard runOnce(timeout)`. `runOnce` вызывает системный примитив ожидания ровно один раз (`epoll_wait`/`kqueue`/`GetQueuedCompletionStatus`), поэтому `poll` — это "один тик" событийного цикла, а не цикл сам по себе; постоянный цикл строится вызывающим кодом (см. `runForever`, `waitFor`, `drain`).

- **Параметры:**
  - `timeout: int` — максимальное время ожидания в миллисекундах (по умолчанию 500).

```nim
var fut = sleepAsync(10)
while not finished(fut):
  poll()          # каждый вызов — один тик цикла, пока таймер не сработает
assert finished(fut)
```

---

### `drain`

```nim
proc drain*(timeout = 500)
```

**Что делает.** В отличие от `poll`, который делает ровно один системный опрос, `drain` вызывает `runOnce` в цикле, пока есть незавершённые операции (`hasPendingOperations`), и суммарно не тратит больше `timeout` миллисекунд. Полезна, когда нужно "вычерпать" всю доступную на данный момент работу диспетчера одним вызовом, а не вручную городить цикл `while`.

**Разбор реализации.** Ведёт учёт прошедшего времени через `getMonoTime()` (монотонные, не подверженные переводу системных часов, метки времени) и на каждой итерации уменьшает остаток `timeout - elapsed`, передаваемый в `runOnce`. Цикл останавливается по любому из двух условий: операций не осталось, либо истёк лимит времени — то есть `drain` не гарантирует полного завершения всех операций, а лишь "выбирает" отведённое время.

- **Параметры:**
  - `timeout: int` — суммарный бюджет времени в миллисекундах на обработку всех доступных событий.

```nim
discard sleepAsync(5)
discard sleepAsync(5)
drain(100)                       # обработает оба таймера одним вызовом
assert not hasPendingOperations()
```

---

### `runForever`

```nim
proc runForever*()
```

**Что делает.** Запускает бесконечный цикл `poll()`. Используется как "точка входа" сервера, который никогда не должен завершаться сам — выход возможен только через исключение или завершение процесса.

**Разбор реализации.** Тело — буквально `while true: poll()`. Никакой дополнительной логики нет: вся защита от busy-loop обеспечивается тем, что `poll` внутри блокируется в системном вызове ожидания до наступления события или истечения таймаута.

- **Параметры:** нет.

```nim
# Пример иллюстративный: runForever() не возвращает управление,
# поэтому в реальном коде вызывается последней строкой программы,
# после регистрации всех обработчиков.
proc startServer() =
  discard accept(serverSocket)  # регистрируем ожидание подключений
  runForever()
```

---

### `waitFor`

```nim
proc waitFor*[T](fut: Future[T]): T
```

**Что делает.** Блокирует текущий поток, крутя `poll()` в цикле, пока переданный `Future` не завершится, а затем возвращает его значение (или поднимает исключение, если `Future` завершился с ошибкой). Это точка "входа" в асинхронный код из синхронного — типичное место вызова: `main`-процедура, тесты, CLI-утилиты.

**Разбор реализации.** Тело — `while not finished(fut): poll()`, затем `read(fut)`. Важное ограничение: `waitFor` нельзя вызывать внутри `{.async.}`-процедуры — там она заблокирует единственный поток, в котором и должны выполняться остальные асинхронные процедуры, что приведёт к взаимной блокировке (deadlock), если ожидаемый `Future` зависит от прогресса других асинхронных задач.

- **Параметры:**
  - `fut: Future[T]` — ожидаемый `Future` любого типа `T` (включая `void`).

```nim
proc delayedAnswer(): owned(Future[int]) {.async.} =
  await sleepAsync(5)
  result = 42

let answer = waitFor(delayedAnswer())
assert answer == 42
```

---

### `callSoon`

```nim
proc callSoon*(cbproc: proc () {.gcsafe.})
```

**Что делает.** Планирует вызов `cbproc` "как можно скорее" — как только управление в очередной раз вернётся к диспетчеру (то есть на следующем тике `poll`), но не немедленно и не синхронно в точке вызова. Полезна, чтобы отложить выполнение кода до момента, когда текущий стек вызовов развернётся, избегая глубокой рекурсии или переиспользуя диспетчер как планировщик "микрозадач".

**Разбор реализации.** Добавляет `cbproc` в очередь `callbacks` (`Deque`) диспетчера; очередь опустошается в `processPendingCallbacks` на каждой итерации `runOnce`, причём **до** учёта таймаутов на системный опрос (`adjustTimeout` немедленно возвращает 0, если очередь не пуста) — это гарантирует, что отложенные вызовы не будут "затянуты" ожиданием сетевых событий.

- **Параметры:**
  - `cbproc: proc ()` — процедура без параметров и без возвращаемого значения, вызываемая асинхронно позже.

```nim
var order: seq[int] = @[]
add(order, 1)
callSoon(proc () = add(order, 3))
add(order, 2)
poll()
assert order == @[1, 2, 3]  # отложенный вызов выполнился после текущего кода
```

---

## Подписка на события файловых дескрипторов

### `addRead`

```nim
proc addRead*(fd: AsyncFD, cb: Callback)
```

**Что делает.** Подписывает callback `cb` на готовность дескриптора `fd` к чтению. Это низкоуровневый строительный блок: high-level процедуры вроде `recv` вызывают `addRead` внутри себя, оборачивая callback в завершение `Future`.

**Разбор реализации.** На Unix callback добавляется в список `readList` записи селектора, соответствующей `fd`, после чего маска отслеживаемых событий дескриптора обновляется (`updateHandle`). На Windows единого понятия "чтение" в терминах IOCP нет (там события уже привязаны к конкретной операции ввода-вывода), поэтому реализация `addRead` на Windows — это "хак" через `WSAEventSelect`, предназначенный преимущественно для портирования Unix-библиотек, о чём явно предупреждает документация процедуры в исходнике: она конфликтует с `recv`/`accept` этого же модуля на Windows, если использовать их одновременно с `addRead` на одном сокете.

- **Параметры:**
  - `fd: AsyncFD` — наблюдаемый дескриптор.
  - `cb: Callback` — вызывается при готовности к чтению; возвращает `true`, чтобы прекратить наблюдение.

```nim
let fd = createAsyncNativeSocket()
var triggered = false
proc onReadable(sock: AsyncFD): bool =
  triggered = true
  result = true
addRead(fd, onReadable)
```

---

### `addWrite`

```nim
proc addWrite*(fd: AsyncFD, cb: Callback)
```

**Что делает.** Симметрична `addRead`, но подписывает на готовность к записи. Используется, например, внутри `send`/`connect` (Unix-ветка), чтобы дождаться момента, когда сокет готов принять очередную порцию данных или когда неблокирующее `connect` завершилось.

**Разбор реализации.** На Unix — та же схема со списком `writeList` и обновлением маски событий селектора; на Windows — аналогичный `addRead` "хак" через `WSAEventSelect` с маской `FD_WRITE or FD_CONNECT or FD_CLOSE`.

- **Параметры:**
  - `fd: AsyncFD` — наблюдаемый дескриптор.
  - `cb: Callback` — вызывается при готовности к записи; возвращает `true`, чтобы прекратить наблюдение.

```nim
let fd = createAsyncNativeSocket()
proc onWritable(sock: AsyncFD): bool =
  result = true  # разовая проверка готовности, дальше не наблюдаем
addWrite(fd, onWritable)
```

---

### `addTimer`

```nim
proc addTimer*(timeout: int, oneshot: bool, cb: Callback)
```

**Что делает.** Регистрирует таймер, который вызовет `cb` через `timeout` миллисекунд. Если `oneshot == true`, событие сработает один раз; если `false` — будет повторяться периодически (каждые `timeout` миллисекунд), пока `cb` не вернёт `true`.

**Разбор реализации.** На Unix, где доступны платформенные таймеры селектора (`ioselSupportedPlatform`, то есть Linux/BSD/macOS/Solaris), таймер регистрируется напрямую в `selector.registerTimer`. На Windows — это `CreateEvent` плюс `RegisterWaitForSingleObject` с флагом `WT_EXECUTEONLYONCE`, выставляемым при `oneshot == true`. Общий для обеих платформ таймерный механизм диспетчера (`processTimers`, основанный на `HeapQueue`, используемый, например, в `sleepAsync`) — это отдельный, более простой путь, работающий через `Future`; `addTimer` — низкоуровневая callback-версия для случаев, где `Future` избыточен.

- **Параметры:**
  - `timeout: int` — интервал в миллисекундах, должен быть больше 0.
  - `oneshot: bool` — `true` — одно срабатывание, `false` — периодические срабатывания.
  - `cb: Callback` — вызывается по истечении таймера.

```nim
var ticks = 0
proc onTick(fd: AsyncFD): bool =
  inc(ticks)
  result = ticks >= 3    # снимаем таймер после третьего тика
addTimer(5, oneshot = false, cb = onTick)
drain(200)
assert ticks == 3
```

---

### `addProcess`

```nim
proc addProcess*(pid: int, cb: Callback)
```

**Что делает.** Регистрирует callback, который будет вызван, когда процесс с идентификатором `pid` завершится. Позволяет асинхронно дожидаться завершения дочернего процесса, не блокируя поток.

**Разбор реализации.** На Windows — открытие хендла процесса (`openProcess` с флагом `SYNCHRONIZE`) и `RegisterWaitForSingleObject` на этот хендл с флагом `WT_EXECUTEONLYONCE` (процесс либо жив, либо мёртв — повторных срабатываний не бывает). На Unix, где доступен `ioselSupportedPlatform`, — регистрация через `selector.registerProcess`, полагающийся на платформенный механизм отслеживания (например, `pidfd` на современных ядрах Linux или `kqueue`-события `EVFILT_PROC` на BSD/macOS).

- **Параметры:**
  - `pid: int` — идентификатор отслеживаемого процесса.
  - `cb: Callback` — вызывается при завершении процесса.

```nim
import std/osproc

let p = startProcess("sleep", args = @["0"])
proc onExit(fd: AsyncFD): bool =
  echo "процесс завершился"
  result = true
addProcess(processID(p), onExit)
```

---

### `addSignal`

```nim
proc addSignal*(signal: int, cb: Callback)
```

**Что делает.** Подписывает callback на получение POSIX-сигнала (например, `SIGTERM`, `SIGUSR1`) — асинхронная альтернатива установке классического обработчика сигнала. Доступна только на платформах с поддержкой сигналов в `std/selectors` (`ioselSupportedPlatform`); на Windows такой процедуры нет вовсе, поскольку у Windows нет POSIX-сигналов в этом смысле.

**Разбор реализации.** Делегирует регистрацию `selector.registerSignal`, который на Linux использует `signalfd`, а на BSD/macOS — `kqueue`-фильтр `EVFILT_SIGNAL`: сигнал превращается в обычное событие готовности дескриптора, обрабатываемое тем же циклом `runOnce`, что и сокеты и таймеры.

- **Параметры:**
  - `signal: int` — номер сигнала (см. `std/posix`, например `SIGTERM`).
  - `cb: Callback` — вызывается при получении сигнала.

```nim
when defined(posix):
  import std/posix
  proc onTerm(fd: AsyncFD): bool =
    echo "получен SIGTERM, завершаемся"
    result = true
  addSignal(SIGTERM, onTerm)
```

---

## `AsyncEvent` — пользовательские события

### `newAsyncEvent`

```nim
proc newAsyncEvent*(): AsyncEvent
```

**Что делает.** Создаёт новый, ещё ни на что не зарегистрированный объект пользовательского события. Само по себе событие не наблюдается диспетчером, пока на него явно не подписаться через `addEvent`.

**Разбор реализации.** На Unix — тонкая обёртка над `newSelectEvent()` из `std/selectors` (как правило, это `eventfd` на Linux или аналог на BSD/macOS); на Windows — именованный объект ядра (`CreateEvent`) с ручным сбросом, обёрнутый в структуру `AsyncEventImpl`.

- **Параметры:** нет.

```nim
let ev = newAsyncEvent()
```

---

### `trigger`

```nim
proc trigger*(ev: AsyncEvent)
```

**Что делает.** Переводит событие `ev` в сигнальное состояние, будя диспетчер (если тот в этот момент ожидает в `poll`) и вызывая все подписанные на `ev` callback'и на следующем тике цикла.

**Разбор реализации.** Потокобезопасна — событие можно взводить из другого потока, что делает `AsyncEvent` основным механизмом межпоточной коммуникации с однопоточным диспетчером (например, сигнал "пора остановиться" из потока, обрабатывающего Ctrl+C).

- **Параметры:**
  - `ev: AsyncEvent` — взводимое событие.

```nim
let ev = newAsyncEvent()
var fired = false
proc onFire(fd: AsyncFD): bool =
  fired = true
  result = true
addEvent(ev, onFire)
trigger(ev)
drain(100)
assert fired
```

---

### `close` (событие)

```nim
proc close*(ev: AsyncEvent)
```

**Что делает.** Освобождает системные ресурсы, связанные с `ev` (закрывает файловый дескриптор события на Unix, хендл ядра на Windows). После вызова событие использовать нельзя.

**Разбор реализации.** Симметрична `close` для сокетов/файлов: событие — это тоже дескриптор ядра, который необходимо явно закрыть, иначе он будет удерживаться до завершения процесса.

- **Параметры:**
  - `ev: AsyncEvent` — закрываемое событие.

```nim
let ev = newAsyncEvent()
close(ev)
```

---

### `addEvent`

```nim
proc addEvent*(ev: AsyncEvent, cb: Callback)
```

**Что делает.** Подписывает `cb` на срабатывание пользовательского события `ev` — с этого момента диспетчер будет вызывать `cb` каждый раз, когда `ev` взводится через `trigger`.

**Разбор реализации.** Регистрирует `ev` как ещё одну запись в том же селекторе/порте завершения, что и обычные сокеты, — с точки зрения цикла `runOnce` пользовательское событие неотличимо от готовности сокета к чтению, что и позволяет единому циклу опроса обслуживать сокеты, таймеры, сигналы и ручные события одновременно.

- **Параметры:**
  - `ev: AsyncEvent` — наблюдаемое событие.
  - `cb: Callback` — вызывается при взведении события.

```nim
let shutdown = newAsyncEvent()
proc onShutdown(fd: AsyncFD): bool =
  echo "остановка диспетчера"
  result = true
addEvent(shutdown, onShutdown)
```

---

## Сокеты: создание, приём соединений, обмен данными

### `createAsyncNativeSocket`

```nim
proc createAsyncNativeSocket*(domain: cint, sockType: cint, protocol: cint,
                              inheritable = defined(nimInheritHandles)): AsyncFD
proc createAsyncNativeSocket*(domain: Domain = Domain.AF_INET,
                              sockType: SockType = SOCK_STREAM,
                              protocol: Protocol = IPPROTO_TCP,
                              inheritable = defined(nimInheritHandles)): AsyncFD
```

**Что делает.** Создаёт новый нативный сокет, переводит его в неблокирующий режим и регистрирует в диспетчере текущего потока — то есть заменяет собой связку "создать сокет вручную + `setBlocking(false)` + `register`". Есть два перегруженных варианта: низкоуровневый (сырые `cint`-константы) и высокоуровневый (типизированные перечисления `Domain`/`SockType`/`Protocol` из `std/nativesockets`).

**Разбор реализации.** На macOS дополнительно выставляется опция сокета `SO_NOSIGPIPE`, чтобы попытка записи в закрытое соединение возвращала ошибку вместо доставки сигнала `SIGPIPE` процессу — детали, специфичные для BSD-семейства сокетов на Дарвине. При ошибке создания (`createNativeSocket` вернул `osInvalidSocket`) процедура тихо возвращает `osInvalidSocket.AsyncFD` без исключения — вызывающему коду стоит проверять результат.

- **Параметры:**
  - `domain: Domain` — семейство адресов (`AF_INET`, `AF_INET6` и т.п.).
  - `sockType: SockType` — тип сокета (`SOCK_STREAM` для TCP, `SOCK_DGRAM` для UDP).
  - `protocol: Protocol` — протокол (`IPPROTO_TCP`, `IPPROTO_UDP`).
  - `inheritable: bool` — может ли дескриптор наследоваться дочерними процессами (по умолчанию — нет).

```nim
let tcpSocket = createAsyncNativeSocket(Domain.AF_INET, SockType.SOCK_STREAM, Protocol.IPPROTO_TCP)
assert contains(getGlobalDispatcher(), tcpSocket)  # сокет уже зарегистрирован диспетчером
```

---

### `closeSocket`

```nim
proc closeSocket*(socket: AsyncFD)
```

**Что делает.** Закрывает сокет и одновременно снимает его с наблюдения диспетчера, разблокируя все ожидающие на нём callback'и чтения/записи (им сообщается о закрытии, чтобы они могли корректно завершить связанные `Future` ошибкой, а не "зависнуть" навсегда).

**Разбор реализации.** На Unix порядок важен: сначала запоминаются списки `readList`/`writeList`, затем сокет снимается с селектора и закрывается на уровне ОС, и только после этого вызываются сохранённые callback'и — если хотя бы один из них не вернёт `true` (то есть не согласится завершиться), это расценивается как ошибка использования и поднимает исключение, поскольку операции над уже закрытым дескриптором продолжаться не могут.

- **Параметры:**
  - `socket: AsyncFD` — закрываемый сокет.

```nim
let s = createAsyncNativeSocket()
closeSocket(s)
assert not contains(getGlobalDispatcher(), s)
```

---

### `setInheritable`

```nim
proc setInheritable*(fd: AsyncFD, inheritable: bool): bool
```

**Что делает.** Управляет тем, будет ли файловый дескриптор унаследован дочерними процессами (аналог флага `FD_CLOEXEC`/`O_NOINHERIT`). Возвращает `true` при успехе. Доступна не на всех платформах — перед использованием стоит проверять `declared(setInheritable)`.

**Разбор реализации.** Это тонкая обёртка над платформенной процедурой `setInheritable` для `FileHandle`, вызываемая через "грязный" (`{.dirty.}`) шаблон `implementSetInheritable`, который подключает реализацию только там, где соответствующая системная процедура вообще объявлена (`when declared(setInheritable)`), — приём, позволяющий не дублировать процедуру вручную на каждой ветке платформы.

- **Параметры:**
  - `fd: AsyncFD` — изменяемый дескриптор.
  - `inheritable: bool` — `true`, чтобы разрешить наследование.

```nim
when declared(setInheritable):
  let s = createAsyncNativeSocket()
  assert setInheritable(s, false)
```

---

### `recv`

```nim
proc recv*(socket: AsyncFD, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[string])
```

**Что делает.** Читает **до** `size` байт из `socket`. Возвращённый `Future` завершается, когда прочитана хотя бы часть данных, либо пустой строкой `""`, если соединение было разорвано другой стороной — в последнем случае это **не** ошибка (`Future` не завершается неудачей), а штатный признак конца потока, который необходимо явно проверять сравнением с `""`.

**Разбор реализации.** На Unix операция реализована через регистрацию callback'а на готовность к чтению (`addRead`): при срабатывании выполняется неблокирующий системный `recv`, а коды ошибок `EINTR`/`EWOULDBLOCK`/`EAGAIN` трактуются как "данных пока нет, ждём ещё" (`result = false` в callback'е — наблюдение продолжается) в отличие от прочих ошибок, которые завершают `Future` неудачей. На Windows та же операция построена вокруг `WSARecv` и структуры `OVERLAPPED`, результат приходит уже готовым через порт завершения — модель "просим и ждём готовый результат" вместо "ждём готовности и читаем сами".

- **Параметры:**
  - `socket: AsyncFD` — сокет для чтения (уже зарегистрирован).
  - `size: int` — верхняя граница числа читаемых байт.
  - `flags: set[SocketFlag]` — флаги поведения, по умолчанию `SafeDisconn` (не считать типичные ошибки разрыва соединения фатальными).

```nim
proc echoOnce(socket: AsyncFD) {.async.} =
  let data = await recv(socket, 1024)
  if data == "":
    echo "соединение закрыто удалённой стороной"
  else:
    echo "получено: ", data
```

---

### `recvInto`

```nim
proc recvInto*(socket: AsyncFD, buf: pointer, size: int,
               flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

**Что делает.** То же, что `recv`, но пишет данные напрямую в уже выделенный буфер `buf` (не выделяя новую строку) и возвращает не строку, а число фактически прочитанных байт. Разрыв соединения сигнализируется значением `0`. Используется там, где важно избежать лишних аллокаций — например, в цикле повторного использования одного и того же буфера.

**Разбор реализации.** Отличается от `recv` только тем, куда пишется результат системного вызова: вместо промежуточной строки `readBuffer` данные копируются прямо в `buf`, переданный вызывающим кодом, — экономия одной копии памяти по сравнению с `recv`.

- **Параметры:**
  - `socket: AsyncFD` — сокет для чтения.
  - `buf: pointer` — буфер назначения, должен вмещать не менее `size` байт и оставаться валидным (не собранным GC) до завершения `Future`.
  - `size: int` — сколько байт максимум читать.
  - `flags: set[SocketFlag]` — как у `recv`.

```nim
proc readFixed(socket: AsyncFD): Future[int] {.async.} =
  var buf = newString(64)
  result = await recvInto(socket, addr buf[0], 64)
```

---

### `send`

```nim
proc send*(socket: AsyncFD, buf: pointer, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])
proc send*(socket: AsyncFD, data: string,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

**Что делает.** Отправляет данные в `socket`; возвращённый `Future` завершается, когда **все** переданные данные гарантированно отправлены. Есть низкоуровневый вариант с сырым указателем и размером (платформо-зависимая реализация) и высокоуровневый вариант со `string` — общий для обеих платформ, построенный поверх варианта с указателем.

**Разбор реализации.** На Unix цикл записи явно отслеживает прогресс (`written`) и продолжает вызывать неблокирующий системный `send`, пока не отправит все `size` байт — частичная запись не считается завершением, callback возвращает `false` и продолжает наблюдение через `addWrite`. Строковый вариант дополнительно решает проблему времени жизни данных: строка `data`, переданная вызывающим кодом, может быть собрана сборщиком мусора раньше, чем завершится асинхронная запись, поэтому используется вспомогательная функция `keepAlive`, которая "утверждает" параметр как escaping-переменную, заставляя компилятор поместить его в замыкание callback'а и удерживать живым до завершения отправки.

- **Параметры:**
  - `socket: AsyncFD` — сокет для записи.
  - `buf: pointer` / `data: string` — отправляемые данные.
  - `size: int` — размер данных в байтах (только для варианта с указателем).
  - `flags: set[SocketFlag]` — флаги поведения.

```nim
proc greet(socket: AsyncFD) {.async.} =
  await send(socket, "привет\n")
```

---

### `sendTo`

```nim
proc sendTo*(socket: AsyncFD, data: pointer, size: int, saddr: ptr SockAddr,
             saddrLen: SockLen, flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

**Что делает.** Отправляет датаграмму по указанному адресу `saddr` без предварительного `connect` — процедура для UDP-сокетов (`SOCK_DGRAM`), где каждый пакет адресуется индивидуально.

**Разбор реализации.** Адрес назначения копируется в собственный стековый буфер фиксированного размера (128 байт — размер `SOCKADDR_STORAGE`, вмещающий как IPv4-, так и IPv6-адреса), чтобы не зависеть от времени жизни указателя `saddr`, переданного вызывающим кодом: на момент фактической отправки (которая на Unix происходит асинхронно, при готовности сокета к записи) исходный `saddr` может быть уже недействителен.

- **Параметры:**
  - `socket: AsyncFD` — UDP-сокет.
  - `data: pointer`, `size: int` — отправляемые данные и их размер.
  - `saddr: ptr SockAddr`, `saddrLen: SockLen` — адрес и размер структуры адреса получателя.
  - `flags: set[SocketFlag]` — флаги поведения.

```nim
proc sendDatagram(socket: AsyncFD, data: string, address: ptr SockAddr, addrLen: SockLen) {.async.} =
  await sendTo(socket, unsafeAddr data[0], len(data), address, addrLen)
```

---

### `recvFromInto`

```nim
proc recvFromInto*(socket: AsyncFD, data: pointer, size: int,
                   saddr: ptr SockAddr, saddrLen: ptr SockLen,
                   flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

**Что делает.** Принимает одну датаграмму в буфер `data`, одновременно записывая адрес отправителя в `saddr`/`saddrLen`. Возвращённый `Future` завершается размером принятого пакета в байтах.

**Разбор реализации.** Парная процедура к `sendTo` для той же модели UDP-обмена; в отличие от `recv`/`recvInto` возвращает не только данные, но и адрес источника — необходимая информация для сервера, обслуживающего множество клиентов через один UDP-сокет, где каждый входящий пакет может быть от другого адресата.

- **Параметры:**
  - `socket: AsyncFD` — UDP-сокет.
  - `data: pointer`, `size: int` — буфер приёма и его размер.
  - `saddr: ptr SockAddr`, `saddrLen: ptr SockLen` — куда записать адрес отправителя.
  - `flags: set[SocketFlag]` — флаги поведения.

```nim
proc receiveDatagram(socket: AsyncFD): Future[int] {.async.} =
  var buf = newString(512)
  var address: Sockaddr_storage
  var addrLen = SockLen(sizeof(address))
  result = await recvFromInto(socket, addr buf[0], 512, cast[ptr SockAddr](addr address), addr addrLen)
```

---

### `acceptAddr`

```nim
proc acceptAddr*(socket: AsyncFD, flags = {SocketFlag.SafeDisconn},
                 inheritable = defined(nimInheritHandles)):
    owned(Future[tuple[address: string, client: AsyncFD]])
```

**Что делает.** Принимает одно входящее соединение на слушающем `socket` и возвращает как принятый клиентский сокет, так и текстовый адрес подключившейся стороны — в отличие от `accept`, который отдаёт только сам сокет.

**Разбор реализации.** На Unix используется `accept4` там, где она объявлена (позволяет атомарно выставить `SOCK_CLOEXEC`, избегая гонки между `accept` и последующей установкой флага "не наследовать"), с падением на обычный `accept` плюс отдельный вызов `setInheritable` там, где `accept4` недоступна. Принятый клиентский дескриптор автоматически регистрируется в диспетчере (`register(client.AsyncFD)`) — вызывающему коду не нужно делать это самостоятельно.

- **Параметры:**
  - `socket: AsyncFD` — слушающий сокет.
  - `flags: set[SocketFlag]` — флаги поведения.
  - `inheritable: bool` — наследуется ли клиентский дескриптор дочерними процессами.

```nim
proc acceptOne(server: AsyncFD) {.async.} =
  let (address, client) = await acceptAddr(server)
  echo "подключение от ", address
```

---

### `accept`

```nim
proc accept*(socket: AsyncFD, flags = {SocketFlag.SafeDisconn},
             inheritable = defined(nimInheritHandles)): owned(Future[AsyncFD])
```

**Что делает.** То же, что `acceptAddr`, но возвращает только клиентский сокет, отбрасывая адрес — удобный сокращённый вариант для случаев, когда адрес подключившейся стороны не нужен.

**Разбор реализации.** Реализована поверх `acceptAddr`: создаёт собственный `Future[AsyncFD]`, навешивает на `Future` результата `acceptAddr` callback, который при успехе извлекает поле `client` из кортежа и завершает им внешний `Future`, а при неудаче — пробрасывает ошибку дальше. Типичный пример "тонкой обёртки", сужающей интерфейс более общей процедуры.

- **Параметры:**
  - `socket: AsyncFD` — слушающий сокет.
  - `flags: set[SocketFlag]` — флаги поведения.
  - `inheritable: bool` — наследуется ли клиентский дескриптор.

```nim
proc acceptLoop(server: AsyncFD) {.async.} =
  while true:
    let client = await accept(server)
    echo "новое соединение: ", int(client)
```

---

## Установление исходящих соединений

### `dial`

```nim
proc dial*(address: string, port: Port,
           protocol: Protocol = IPPROTO_TCP): owned(Future[AsyncFD])
```

**Что делает.** Устанавливает соединение с `address:port`, самостоятельно перебирая все адреса, которые вернул DNS/`getaddrinfo` для этого имени (учитывает и IPv4, и IPv6), и возвращает уже готовый к работе, зарегистрированный сокет. Предпочтительный высокоуровневый способ подключиться к удалённому хосту, когда неважно, по какому конкретно IP-адресу установится соединение.

**Разбор реализации.** Перебор адресов реализован в общем для `dial` и `connect` шаблоне `asyncAddrInfoLoop`: он идёт по связному списку `AddrInfo`, для каждого адреса создаёт сокет подходящего домена и пробует `doConnect`; при неудаче с одним адресом сокет закрывается и предпринимается попытка со следующим, при удаче — попытки прекращаются и итоговый `Future` завершается найденным сокетом. Такая стратегия ("happy eyeballs" в упрощённом, последовательном варианте) избавляет вызывающий код от необходимости вручную обрабатывать множественность записей DNS.

- **Параметры:**
  - `address: string` — имя хоста или IP-адрес.
  - `port: Port` — TCP/UDP-порт.
  - `protocol: Protocol` — протокол транспортного уровня, по умолчанию `IPPROTO_TCP`.

```nim
proc fetchOnce() {.async.} =
  let socket = await dial("example.org", Port(80))
  await send(socket, "GET / HTTP/1.0\r\n\r\n")
```

---

### `connect`

```nim
proc connect*(socket: AsyncFD, address: string, port: Port,
              domain = Domain.AF_INET): owned(Future[void])
```

**Что делает.** Устанавливает соединение на уже созданном сокете `socket` с конкретным `domain` (в отличие от `dial`, который сам подбирает домен и создаёт сокет). Подходит, когда сокет уже создан заранее (например, с особыми опциями) и его нужно только подключить.

**Разбор реализации.** На Windows перед подключением сокет обязан быть предварительно привязан (`bindToDomain`) — таково требование `ConnectEx`, которого нет у обычного `connect(2)` на Unix; после этого используется тот же перебор адресов через `asyncAddrInfoLoop`, что и в `dial`, но без создания нового сокета (`shouldCreateFd = false`), поскольку сокет уже передан вызывающим кодом.

- **Параметры:**
  - `socket: AsyncFD` — заранее созданный сокет.
  - `address: string`, `port: Port` — адрес и порт назначения.
  - `domain: Domain` — семейство адресов, должно соответствовать домену, с которым создан `socket`.

```nim
proc connectManual() {.async.} =
  let socket = createAsyncNativeSocket()
  await connect(socket, "example.org", Port(80))
```

---

## Тайминги и ожидания

### `sleepAsync`

```nim
proc sleepAsync*(ms: int | float): owned(Future[void])
```

**Что делает.** Возвращает `Future`, который завершается через `ms` миллисекунд — асинхронный аналог `sleep`, не блокирующий поток целиком: пока таймер тикает, диспетчер продолжает обслуживать остальные операции.

**Разбор реализации.** Регистрирует пару `(момент срабатывания, Future)` в очереди с приоритетом (`HeapQueue`) диспетчера, упорядоченной по времени срабатывания, — это тот же механизм таймеров, что обрабатывается `processTimers` на каждой итерации `runOnce`, только выраженный через `Future`, а не через низкоуровневый `Callback` (в отличие от `addTimer`).

- **Параметры:**
  - `ms: int | float` — задержка в миллисекундах; поддерживает дробные значения через `float` для суб-миллисекундной точности (переводится в наносекунды).

```nim
let start = epochTime()
waitFor sleepAsync(20)
assert epochTime() - start >= 0.02 - 0.005  # прошло не меньше ~20 мс (с запасом на погрешность)
```

---

### `withTimeout`

```nim
proc withTimeout*[T](fut: Future[T], timeout: int): owned(Future[bool])
```

**Что делает.** Оборачивает произвольный `fut` таймаутом: возвращённый `Future[bool]` завершается значением `true`, если `fut` успел завершиться раньше `timeout` миллисекунд, и `false`, если время вышло первым. Ни в одном из случаев исходный `fut` не отменяется принудительно — он продолжает существовать (и может завершиться позже сам по себе), просто `withTimeout` перестаёт на него ссылаться после таймаута.

**Разбор реализации.** Запускает параллельно два `Future` — исходный `fut` и `sleepAsync(timeout)` — и навешивает на оба свои callback'и; какой бы из двух ни завершился первым, он "побеждает": завершает общий `retFuture` соответствующим значением и явно очищает callback'и (`clearCallbacks`) у проигравшей стороны, чтобы не удерживать в памяти лишние замыкания после того, как результат уже не нужен.

- **Параметры:**
  - `fut: Future[T]` — ожидаемая операция.
  - `timeout: int` — предельное время ожидания в миллисекундах.

```nim
proc slowOp(): owned(Future[int]) {.async.} =
  await sleepAsync(50)
  result = 1

let completed = waitFor withTimeout(slowOp(), 10)
assert completed == false  # операция не успела уложиться в 10 мс
```

---

## Потоковые данные

### `readAll`

```nim
proc readAll*(future: FutureStream[string]): owned(Future[string]) {.async.}
```

**Что делает.** Последовательно читает из потокового `FutureStream[string]` все значения по мере их поступления и склеивает их в одну строку; завершается, когда поток сигнализирует об окончании данных.

**Разбор реализации.** Написана как обычная `{.async.}`-процедура, а не через низкоуровневые callback'и: в цикле вызывается `await read(future)`, возвращающий пару `(hasValue, value)` — паттерн, характерный для `FutureStream` из `std/asyncstreams` (в отличие от `Future[T]`, который завершается один раз, `FutureStream` может отдавать значения многократно, поэтому чтение оформлено как явный цикл с условием остановки по `hasValue == false`).

- **Параметры:**
  - `future: FutureStream[string]` — источник потоковых строковых данных.

```nim
proc demo() {.async.} =
  var stream = newFutureStream[string]("demo")
  await write(stream, "привет, ")
  await write(stream, "мир")
  complete(stream)
  let whole = await readAll(stream)
  assert whole == "привет, мир"
```

---

## Диагностика ресурсов

### `activeDescriptors`

```nim
proc activeDescriptors*(): int {.inline.}
```

**Что делает.** Возвращает текущее число активных (зарегистрированных в диспетчере) файловых дескрипторов. Дешёвая операция, не требующая системного вызова — полезна для мониторинга утечек дескрипторов в долгоживущих серверах.

**Разбор реализации.** На Windows — размер множества `handles` диспетчера; на Unix — счётчик `selector.count`, поддерживаемый самим селектором. Оба варианта — просто чтение уже существующего внутреннего счётчика, отсюда и пометка `{.inline.}`.

- **Параметры:** нет.

```nim
let before = activeDescriptors()
let s = createAsyncNativeSocket()
assert activeDescriptors() == before + 1
```

---

### `maxDescriptors`

```nim
proc maxDescriptors*(): int {.raises: OSError.}
```

**Что делает.** Возвращает максимально допустимое число одновременно открытых файловых дескрипторов для текущего процесса — системный лимит, а не текущее использование (в отличие от `activeDescriptors`). Полезна для расчёта размера пулов соединений заранее, до того как лимит будет исчерпан на практике.

**Разбор реализации.** В отличие от `activeDescriptors`, эта процедура **выполняет системный вызов** на каждое обращение: на большинстве Unix-платформ — `getrlimit(RLIMIT_NOFILE, ...)` с вычитанием единицы (одного дескриптора-резерва); на Windows возвращается захардкоженная константа `16_700_000` (документированный практический предел IOCP на количество хендлов), поскольку понятия "лимита дескрипторов на процесс" в POSIX-смысле там нет. Доступна только на платформах, явно перечисленных в условии компиляции (Linux, Windows, macOS, BSD, Solaris и ряд встраиваемых ОС).

- **Параметры:** нет.

```nim
let limit = maxDescriptors()
assert limit > 0
```

---

## Практические рецепты

### Эхо-сервер на `acceptAddr` + `recv` + `send`

Каждое новое соединение обслуживается независимым асинхронным циклом; сервер продолжает принимать новые подключения, не дожидаясь завершения предыдущих.

```nim
proc handleClient(client: AsyncFD) {.async.} =
  while true:
    let data = await recv(client, 1024)
    if data == "":
      closeSocket(client)
      break
    await send(client, data)  # отправляем обратно то же самое

proc serve(server: AsyncFD) {.async.} =
  while true:
    let (address, client) = await acceptAddr(server)
    asyncCheck handleClient(client)  # запускаем обработку, не дожидаясь её завершения
```

---

### Таймаут на сетевую операцию

Комбинация `withTimeout` и явной проверки результата — типичный шаблон для операций, которые не должны "висеть" бесконечно (например, чтение от медленного или зависшего клиента).

```nim
proc recvWithLimit(socket: AsyncFD, timeoutMs: int): owned(Future[string]) {.async.} =
  let dataFut = recv(socket, 1024)
  let completed = await withTimeout(dataFut, timeoutMs)
  if completed:
    result = read(dataFut)
  else:
    raise newException(TimeoutError, "операция не уложилась в отведённое время")
```

---

### Периодическая фоновая задача через `addTimer`

Низкоуровневый таймер удобен, когда фоновая задача не нуждается в `Future` — например, периодическая проверка состояния без возврата значения наружу.

```nim
proc startHeartbeat(intervalMs: int) =
  proc tick(fd: AsyncFD): bool =
    echo "heartbeat"
    result = false  # false — продолжаем периодические срабатывания
  addTimer(intervalMs, oneshot = false, cb = tick)
```

---

### Плавная остановка диспетчера через `AsyncEvent`

`AsyncEvent` — единственный безопасный способ разбудить диспетчер из другого потока (например, из обработчика системного сигнала завершения), не прибегая к опасным приёмам вроде принудительной остановки потока.

```nim
var running = true
let stopEvent = newAsyncEvent()

proc onStopSignal(fd: AsyncFD): bool =
  running = false
  result = true

addEvent(stopEvent, onStopSignal)

proc mainLoop() =
  while running:
    poll(100)

# из другого потока или обработчика сигнала:
# trigger(stopEvent)
```

---

### Подключение к первому доступному адресу через `dial`

`dial` избавляет от ручного перебора записей DNS — полезно для клиентов, которые должны одинаково хорошо работать с хостами, отдающими и IPv4-, и IPv6-адреса.

```nim
proc connectResilient(host: string, port: int): owned(Future[AsyncFD]) {.async.} =
  result = await dial(host, Port(port))
  echo "подключено, дескриптор = ", int(result)
```

---

## Краткая таблица

| Задача | Что использовать |
|---|---|
| Получить/создать диспетчер потока | `getGlobalDispatcher` |
| Зарегистрировать внешний дескриптор | `register` |
| Проверить, зарегистрирован ли дескриптор | `contains` |
| Снять дескриптор/событие с наблюдения | `unregister` |
| Один тик цикла событий | `poll` |
| Вычерпать все доступные события за бюджет времени | `drain` |
| Бесконечный цикл обслуживания | `runForever` |
| Дождаться `Future` из синхронного кода | `waitFor` |
| Отложить вызов до следующего тика | `callSoon` |
| Низкоуровневая подписка на чтение/запись | `addRead` / `addWrite` |
| Низкоуровневый таймер без `Future` | `addTimer` |
| Дождаться завершения процесса | `addProcess` |
| Дождаться POSIX-сигнала | `addSignal` |
| Пользовательское, ручное событие | `newAsyncEvent` + `addEvent` + `trigger` |
| Создать сокет и сразу зарегистрировать | `createAsyncNativeSocket` |
| Закрыть сокет и снять с наблюдения | `closeSocket` |
| Прочитать данные из TCP-сокета | `recv` / `recvInto` |
| Отправить данные в TCP-сокет | `send` |
| Обменяться UDP-датаграммами | `sendTo` / `recvFromInto` |
| Принять входящее соединение | `accept` / `acceptAddr` |
| Подключиться к хосту по имени, с перебором адресов | `dial` |
| Подключить уже созданный сокет | `connect` |
| Асинхронная задержка | `sleepAsync` |
| Ограничить операцию по времени | `withTimeout` |
| Собрать все данные из `FutureStream[string]` | `readAll` |
| Число используемых дескрипторов сейчас / лимит ОС | `activeDescriptors` / `maxDescriptors` |

---

## Сводка: какую процедуру выбрать

- Нужно просто подключиться к хосту по имени и не думать про IPv4/IPv6 → используйте `dial`.
- Уже есть готовый сокет с особыми опциями, нужно только подключить его → используйте `connect`.
- Нужно принять соединение и знать адрес клиента → используйте `acceptAddr`; если адрес не важен → `accept`.
- Нужно прочитать данные без лишних аллокаций в горячем цикле → используйте `recvInto` вместо `recv`.
- Нужно ограничить операцию по времени, не отменяя её принудительно → оберните `Future` в `withTimeout`.
- Нужна асинхронная пауза → `sleepAsync`, а не блокирующий `sleep`.
- Нужно вызвать асинхронный код из синхронной точки входа (`main`, тесты) → `waitFor`, никогда не `waitFor` внутри `{.async.}`-процедуры.
- Нужно запустить операцию "и забыть", не дожидаясь и не проверяя результат вручную → `asyncCheck` (из `std/asyncfutures`), а не голый `discard`.
- Нужно разбудить диспетчер из другого потока → `AsyncEvent` (`newAsyncEvent`/`addEvent`/`trigger`), а не попытки писать в переменные напрямую из другого потока.
- Нужен периодический фоновый вызов без создания `Future` → `addTimer` с `oneshot = false`.
- Нужно узнать, не утекают ли дескрипторы → сравнивайте `activeDescriptors()` до и после операций; для расчёта лимитов заранее — `maxDescriptors()`.
- Нужно поддержать сигнал завершения процесса (`SIGTERM`) асинхронно, без блокирующих обработчиков → `addSignal` (только на POSIX-платформах).
