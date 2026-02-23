# Модуль Nim `selectors` — полный справочник

> **Назначение модуля:** Высокоуровневое и эффективное мультиплексирование ввода-вывода. Позволяет одному потоку (или пулу потоков с защитой) одновременно следить за множеством файловых дескрипторов, сокетов, таймеров, сигналов, процессов и пользовательских событий — без активного ожидания (busy-wait).
>
> **Используемые примитивы ОС:** `epoll` (Linux), `kqueue` (macOS/BSD), `poll` (Solaris/запасной), `select` (Windows). Нужный бэкенд выбирается автоматически; можно переопределить флагом `-d:nimIoselector=epoll|kqueue|poll|select`.
>
> **Потокобезопасность:** Компилируйте с `-d:threadsafe --threads:on`, чтобы получить потокобезопасную версию.

---

## Содержание

1. [Типы и константы](#типы-и-константы)
   - [`ioselSupportedPlatform`](#ioselSupportedPlatform)
   - [`Event`](#event)
   - [`ReadyKey`](#readykey)
   - [`SelectEvent`](#selectevent)
   - [`Selector[T]`](#selectort)
   - [`IOSelectorsException`](#ioselectorsexception)
2. [Жизненный цикл селектора](#жизненный-цикл-селектора)
   - [`newSelector`](#newselector)
   - [`close` (Selector)](#close-selector)
   - [`getFd`](#getfd)
   - [`isEmpty`](#isempty)
   - [`maxDescriptors`](#maxdescriptors)
3. [Регистрация дескрипторов](#регистрация-дескрипторов)
   - [`registerHandle`](#registerhandle)
   - [`updateHandle`](#updatehandle)
   - [`registerTimer`](#registertimer)
   - [`registerSignal`](#registersignal)
   - [`registerProcess`](#registerprocess)
   - [`registerVnode`](#registervnode)
4. [Пользовательские события](#пользовательские-события)
   - [`newSelectEvent`](#newselectevent)
   - [`trigger`](#trigger)
   - [`close` (SelectEvent)](#close-selectevent)
   - [`registerEvent`](#registerevent)
   - [`unregister` (SelectEvent)](#unregister-selectevent)
5. [Отмена регистрации](#отмена-регистрации)
   - [`unregister` (fd)](#unregister-fd)
6. [Ожидание событий](#ожидание-событий)
   - [`select`](#select)
   - [`selectInto`](#selectinto)
7. [Доступ к пользовательским данным](#доступ-к-пользовательским-данным)
   - [`getData`](#getdata)
   - [`setData`](#setdata)
   - [`withData` (1 ветка)](#withdata-1-ветка)
   - [`withData` (2 ветки)](#withdata-2-ветки)
   - [`contains`](#contains)

---

## Типы и константы

### `ioselSupportedPlatform`

```nim
const ioselSupportedPlatform*: bool
```

Булева константа времени компиляции. Равна `true`, если текущая платформа **полностью поддерживается** (macOS, FreeBSD, NetBSD, OpenBSD, DragonFly, NuttX, Linux без Android/Emscripten). На частично поддерживаемых платформах (Windows, Solaris, Android) ряд возможностей — мониторинг сигналов, процессов, vnode — недоступен.

Используйте эту константу для условной компиляции платформозависимого кода:

```nim
when ioselSupportedPlatform:
  discard sel.registerSignal(SIGTERM, myData)
else:
  echo "Отслеживание сигналов на этой ОС не поддерживается"
```

---

### `Event`

```nim
type Event* {.pure.} = enum
  Read, Write, Timer, Signal, Process, Vnode, User, Error,
  VnodeWrite, VnodeDelete, VnodeExtend, VnodeAttrib,
  VnodeLink, VnodeRename, VnodeRevoke
```

Перечисление, описывающее **что именно произошло** с дескриптором. Значения используются и при регистрации (чтобы объявить интерес), и при чтении результатов (чтобы понять, что сработало).

| Значение | Смысл |
|---|---|
| `Read` | Доступны данные для чтения (или новое соединение на слушающем сокете) |
| `Write` | В буфере есть место; запись не заблокируется |
| `Timer` | Зарегистрированный таймер истёк |
| `Signal` | Получен Unix-сигнал |
| `Process` | Отслеживаемый процесс завершился |
| `Vnode` | Событие файловой системы BSD/macOS (используйте конкретные `Vnode*`-варианты) |
| `User` | Сработало пользовательское событие `SelectEvent` |
| `Error` | Произошла ошибка; подробности в `ReadyKey.errorCode` |
| `VnodeWrite` … `VnodeRevoke` | Детализированные уведомления об изменении файлов на BSD/macOS |

```nim
# Регистрируем интерес к чтению и записи:
sel.registerHandle(sock, {Event.Read, Event.Write}, myData)

# Проверяем, что сработало:
for key in sel.select(1000):
  if Event.Read in key.events:
    let data = sock.recv(buf, 1024)
```

---

### `ReadyKey`

```nim
type ReadyKey* = object
  fd*: int               # дескриптор, который стал готов
  events*: set[Event]    # какие события сработали
  errorCode*: OSErrorCode # ненулевой при наличии Event.Error
```

Значение, возвращаемое функциями `select` / `selectInto`. Сообщает, **какой дескриптор** стал готов и **какой вид готовности** был обнаружен.

```nim
for key in sel.select(500):
  if Event.Error in key.events:
    echo "Ошибка на fd ", key.fd, ": ", osErrorMsg(key.errorCode)
  elif Event.Read in key.events:
    echo "fd ", key.fd, " готов к чтению"
```

---

### `SelectEvent`

```nim
type SelectEvent* = object
```

Непрозрачный (opaque) дескриптор, представляющий **пользовательское событие** — лёгкий примитив сигнализации между потоками или корутинами. Создаётся через `newSelectEvent`, активируется через `trigger`, уничтожается через `close`.

---

### `Selector[T]`

```nim
type Selector*[T] = ref object
```

Центральный объект модуля. Параметр `T` — тип **прикладных данных**, которые вы прикрепляете к каждому зарегистрированному дескриптору. Конкретный тип (например, `Selector[MyConn]`) позволяет избежать поиска в хэш-таблице — данные приходят вместе с событием.

```nim
type ConnData = object
  id: int
  buf: string

var sel = newSelector[ConnData]()
```

---

### `IOSelectorsException`

```nim
type IOSelectorsException* = object of CatchableError
```

Выбрасывается при сбое внутреннего системного вызова (например, `epoll_create` вернул ошибку). Перехватывайте её, если нужно обрабатывать сбои селектора в штатном режиме.

```nim
try:
  var sel = newSelector[int]()
except IOSelectorsException as e:
  echo "Не удалось создать селектор: ", e.msg
```

---

## Жизненный цикл селектора

### `newSelector`

```nim
proc newSelector*[T](): Selector[T]
```

**Создаёт и инициализирует новый селектор.** Внутри открывается экземпляр `epoll`/`kqueue`/`poll`/`select`. Дескрипторов пока не зарегистрировано.

Параметр типа `T` — тип данных, который будет прикреплён к каждому fd. `void` допустим, если дополнительные данные не нужны.

```nim
var sel = newSelector[string]()  # каждый fd будет нести строковую метку
```

---

### `close` (Selector)

```nim
proc close*[T](s: Selector[T])
```

**Освобождает все ресурсы ОС**, удерживаемые селектором (epoll/kqueue-дескриптор, внутренние буферы, мьютексы в потокобезопасном режиме). После вызова `close` использовать селектор нельзя.

```nim
var sel = newSelector[int]()
# ... работаем с sel ...
sel.close()
```

---

### `getFd`

```nim
proc getFd*[T](s: Selector[T]): int
```

Возвращает **файловый дескриптор самого селектора** на уровне ОС (epoll- или kqueue-fd). Полезно, когда нужно вложить один селектор в другой или передать его fd внешнему мультиплексору. Для бэкендов `poll` и `select` возвращает `-1`, так как у них нет единого fd.

```nim
let fd = sel.getFd()
if fd >= 0:
  echo "Селектор живёт на fd ", fd
```

---

### `isEmpty`

```nim
template isEmpty*[T](s: Selector[T]): bool
```

Возвращает `true`, если в селекторе **нет ни одного зарегистрированного дескриптора**. Удобен как условие выхода из цикла при постепенном удалении дескрипторов.

```nim
while not sel.isEmpty():
  let events = sel.select(100)
  for key in events:
    sel.unregister(key.fd)
```

---

### `maxDescriptors`

```nim
template maxDescriptors*(): int
```

*(Доступен на Windows, Linux, macOS, BSD, Solaris)*

Возвращает **максимальное число файловых дескрипторов**, которые текущий процесс может держать открытыми — через `getrlimit(RLIMIT_NOFILE)` или фиксированную константу на Windows. Удобно для предварительного выделения структур данных.

```nim
echo "Лимит fd процесса: ", maxDescriptors()
```

---

## Регистрация дескрипторов

### `registerHandle`

```nim
proc registerHandle*[T](s: Selector[T], fd: int | SocketHandle,
                        events: set[Event], data: T)
```

**Добавляет файловый дескриптор или сокет** в селектор и объявляет, за какими событиями следить (`Read`, `Write` или оба). Значение `data` хранится внутри и возвращается вместе с каждым событием для этого fd — используйте его для хранения состояния соединения, буферов, любого контекста на fd.

Каждый fd можно зарегистрировать **не более одного раза**. Повторная регистрация одного и того же fd вызовет `IOSelectorsException`.

```nim
var sel = newSelector[string]()
let sock: SocketHandle = createSocket()
sel.registerHandle(sock, {Event.Read}, "client-1")

for key in sel.select(2000):
  if Event.Read in key.events:
    echo sel.getData(key.fd)  # выведет "client-1"
```

---

### `updateHandle`

```nim
proc updateHandle*[T](s: Selector[T], fd: int | SocketHandle,
                      events: set[Event])
```

**Изменяет маску событий** у уже зарегистрированного fd — без отмены регистрации и повторной. Полезно, когда нужно переключить соединение с чтения на запись и обратно по мере изменения состояния приложения (например, переключиться на `Write`, когда появились данные для отправки, и вернуться к `Read` после опустошения буфера).

```nim
# Изначально интересует только чтение:
sel.registerHandle(sock, {Event.Read}, myData)

# Есть данные для отправки — переключаемся на запись:
sel.updateHandle(sock, {Event.Write})

# Буфер отправлен — обратно на чтение:
sel.updateHandle(sock, {Event.Read})
```

---

### `registerTimer`

```nim
proc registerTimer*[T](s: Selector[T], timeout: int, oneshot: bool,
                       data: T): int {.discardable.}
```

**Регистрирует таймер**, который срабатывает через `timeout` миллисекунд. Если `oneshot = true` — таймер срабатывает ровно один раз и останавливается. Если `false` — срабатывает периодически каждые `timeout` мс. Возвращает внутренний fd таймера (нужен для досрочной отмены регистрации).

Значение `data` передаётся вместе с событием `Timer` — точно так же, как в `registerHandle`.

```nim
# Одноразовый таймаут 5 секунд:
let timerFd = sel.registerTimer(5000, true, "дедлайн")

for key in sel.select(-1):   # блокируем до любого события
  if Event.Timer in key.events:
    echo "Таймер сработал: ", sel.getData(key.fd)
    break
```

```nim
# Периодический пульс каждую секунду:
discard sel.registerTimer(1000, false, "heartbeat")
```

---

### `registerSignal`

```nim
proc registerSignal*[T](s: Selector[T], signal: int,
                        data: T): int {.discardable.}
```

*(Не поддерживается на Windows)*

**Перехватывает Unix-сигнал** и доставляет его как событие `Signal` через селектор — вместо асинхронного прерывания процесса. Это позволяет безопасно обрабатывать сигналы внутри цикла событий.

Возвращает внутренний fd сигнала (для последующего `unregister`, если нужно).

```nim
import std/posix

discard sel.registerSignal(SIGTERM.int, "получен-sigterm")

for key in sel.select(-1):
  if Event.Signal in key.events:
    echo "Получен сигнал: ", sel.getData(key.fd)
    break
```

---

### `registerProcess`

```nim
proc registerProcess*[T](s: Selector[T], pid: int,
                         data: T): int {.discardable.}
```

**Следит за дочерним процессом** с идентификатором `pid`. Когда процесс завершается, доставляется событие `Process`. Это альтернатива блокирующему `waitpid` — завершение процесса обрабатывается внутри цикла событий наравне с I/O-событиями.

```nim
import std/osproc

let child = startProcess("sleep", args=["3"])
discard sel.registerProcess(child.processID, "дочерний-завершён")

for key in sel.select(-1):
  if Event.Process in key.events:
    echo "Дочерний процесс завершился"
    child.close()
    break
```

---

### `registerVnode`

```nim
proc registerVnode*[T](s: Selector[T], fd: cint, events: set[Event],
                       data: T)
```

*(Только BSD и macOS)*

**Отслеживает изменения файловой системы** на уровне открытого файлового дескриптора через kqueue `EVFILT_VNODE`. Выбирайте нужные события из набора `Vnode*`:

| Событие | Когда срабатывает |
|---|---|
| `VnodeWrite` | В файл были записаны данные |
| `VnodeDelete` | Файл был удалён (`unlink`) |
| `VnodeExtend` | Файл увеличился в размере |
| `VnodeAttrib` | Изменились атрибуты файла (права доступа, временны́е метки) |
| `VnodeLink` | Изменилось число жёстких ссылок |
| `VnodeRename` | Файл был переименован |
| `VnodeRevoke` | Файл был отозван (`revoke`) |

```nim
when defined(bsd) or defined(macosx):
  import std/posix
  let fd = open("config.toml", O_RDONLY)
  sel.registerVnode(fd, {VnodeWrite, VnodeDelete}, "config-file")

  for key in sel.select(-1):
    if VnodeDelete in key.events:
      echo "Файл конфигурации был удалён!"
```

---

## Пользовательские события

Пользовательские события позволяют разбудить селектор **из другого потока или корутины** — без участия сети или ОС-сигналов.

### `newSelectEvent`

```nim
proc newSelectEvent*(): SelectEvent
```

**Создаёт новый объект пользовательского события.** Внутри это, как правило, пара «самозаписывающий канал» (self-pipe) или `eventfd`. Прежде чем использовать, необходимо зарегистрировать в селекторе через `registerEvent`.

```nim
var ev = newSelectEvent()
```

---

### `trigger`

```nim
proc trigger*(ev: SelectEvent)
```

**Сигнализирует событие**: селектор, в котором зарегистрирован `ev`, пробуждается и сообщает о событии `User`. Безопасно вызывать из любого потока.

```nim
# В другом потоке или колбэке:
ev.trigger()
```

---

### `close` (SelectEvent)

```nim
proc close*(ev: SelectEvent)
```

**Освобождает ресурсы ОС**, связанные с событием (внутренний канал или eventfd). Вызывайте после того, как событие отменено регистрации в селекторе.

```nim
sel.unregister(ev)
ev.close()
```

---

### `registerEvent`

```nim
proc registerEvent*[T](s: Selector[T], ev: SelectEvent, data: T)
```

**Добавляет пользовательское событие в список наблюдения селектора.** После этого каждый вызов `trigger(ev)` породит событие `User` при следующем вызове `select`/`selectInto`.

```nim
var sel = newSelector[string]()
var ev  = newSelectEvent()
sel.registerEvent(ev, "пробуждение")

# Из другого потока:
ev.trigger()

# В цикле событий:
for key in sel.select(5000):
  if Event.User in key.events:
    echo sel.getData(key.fd)  # "пробуждение"
```

---

### `unregister` (SelectEvent)

```nim
proc unregister*[T](s: Selector[T], ev: SelectEvent)
```

**Удаляет пользовательское событие** из селектора. После этого вызов `trigger` не будет вызывать пробуждение селектора.

```nim
sel.unregister(ev)
ev.close()
```

---

## Отмена регистрации

### `unregister` (fd)

```nim
proc unregister*[T](s: Selector[T], fd: int | SocketHandle | cint)
```

**Удаляет файловый дескриптор, сокет, fd таймера, сигнала или процесса** из селектора. Связанные данные `data` удаляются. После отмены регистрации события для этого fd больше не поступают.

```nim
# Клиент отключился — убираем из селектора:
sel.unregister(clientSock)
clientSock.close()
```

---

## Ожидание событий

### `select`

```nim
proc select*[T](s: Selector[T], timeout: int): seq[ReadyKey]
```

**Блокирует выполнение, пока не станет готов хотя бы один зарегистрированный дескриптор**, затем возвращает `seq` из объектов `ReadyKey`, описывающих все готовые дескрипторы.

- `timeout > 0` — ждать не дольше указанного числа **миллисекунд**.
- `timeout = 0` — вернуться немедленно (неблокирующий опрос).
- `timeout = -1` — блокировать **бесконечно** до появления хотя бы одного события.

`select` выделяет новый `seq` при каждом вызове. Если нужен цикл без аллокаций, используйте `selectInto`.

```nim
while true:
  let ready = sel.select(1000)   # ждём до 1 секунды
  for key in ready:
    if Event.Read in key.events:
      handleRead(key.fd)
    if Event.Error in key.events:
      handleError(key.fd, key.errorCode)
```

---

### `selectInto`

```nim
proc selectInto*[T](s: Selector[T], timeout: int,
                    results: var openArray[ReadyKey]): int
```

**Та же семантика, что у `select`**, но результаты записываются в **заранее выделенный массив**, который вы передаёте сами — без аллокации на куче. Возвращает количество событий, записанных в `results`.

Используйте в критически важных по производительности циклах, когда нужен полный контроль над аллокациями.

```nim
var buf: array[64, ReadyKey]

while true:
  let n = sel.selectInto(500, buf)
  for i in 0 ..< n:
    let key = buf[i]
    if Event.Write in key.events:
      handleWrite(key.fd)
```

---

## Доступ к пользовательским данным

### `getData`

```nim
proc getData*[T](s: Selector[T], fd: SocketHandle | int): var T
```

**Возвращает изменяемую ссылку** на прикладные данные, хранящиеся для `fd`. Если `fd` не зарегистрирован, возвращается нулевое/пустое значение типа `T` (без исключения). Так как возвращается `var T`, данные можно изменить на месте.

```nim
type Conn = object
  bytesRead: int

var sel = newSelector[Conn]()
sel.registerHandle(sock, {Event.Read}, Conn(bytesRead: 0))

for key in sel.select(-1):
  sel.getData(key.fd).bytesRead += 100   # изменяем на месте
```

---

### `setData`

```nim
proc setData*[T](s: Selector[T], fd: SocketHandle | int, data: var T): bool
```

**Заменяет прикладные данные** для `fd` новым значением. Возвращает `true` в случае успеха, `false` — если `fd` не зарегистрирован в селекторе.

```nim
var updated = Conn(bytesRead: 999)
if not sel.setData(sock, updated):
  echo "fd не найден в селекторе"
```

---

### `withData` (1 ветка)

```nim
template withData*[T](s: Selector[T], fd: SocketHandle | int, value,
                      body: untyped)
```

**Безопасно получает данные для `fd`** в переменную `value` и выполняет `body` — но **только если `fd` зарегистрирован**. Если fd отсутствует, тело молча пропускается. `value` изменяема, и изменения сохраняются.

Это идиоматический способ обратиться к данным fd без ручной проверки через `contains`.

```nim
sel.withData(key.fd, conn) do:
  conn.bytesRead += bytesReceived
  if conn.bytesRead > 1_000_000:
    sel.unregister(key.fd)
```

---

### `withData` (2 ветки)

```nim
template withData*[T](s: Selector[T], fd: SocketHandle | int, value,
                      body1, body2: untyped)
```

**Двухветочная версия `withData`**: `body1` выполняется, если `fd` зарегистрирован (с привязанным `value`), `body2` — если fd не найден. Используйте вторую ветку для обработки ошибок или журналирования.

```nim
sel.withData(key.fd, conn) do:
  echo "Обрабатываем fd ", key.fd, ", байт прочитано: ", conn.bytesRead
do:
  echo "Неизвестный fd: ", key.fd
  # fd исчез между вызовом select() и этим местом
```

---

### `contains`

```nim
proc contains*[T](s: Selector[T], fd: SocketHandle | int): bool {.inline.}
```

**Возвращает `true`**, если `fd` в данный момент зарегистрирован в селекторе. Простая проверка наличия — эквивалентна условию-страже в `withData`, но явная.

```nim
if sel.contains(sock):
  sel.unregister(sock)
  sock.close()
```

---

## Полный рабочий пример

```nim
import std/selectors, std/nativesockets, std/net

type ConnData = object
  label: string

var sel = newSelector[ConnData]()

# Регистрируем слушающий сокет
let server = newSocket()
server.bindAddr(Port(8080))
server.listen()
setBlocking(server.getFd(), false)
sel.registerHandle(server.getFd().int, {Event.Read}, ConnData(label: "server"))

# Регистрируем сторожевой таймер на 10 секунд
discard sel.registerTimer(10_000, true, ConnData(label: "watchdog"))

# Регистрируем пользовательское событие для штатного завершения
var shutdownEv = newSelectEvent()
sel.registerEvent(shutdownEv, ConnData(label: "shutdown"))

while not sel.isEmpty():
  for key in sel.select(500):
    sel.withData(key.fd, conn) do:
      case conn.label
      of "server":
        # Принимаем новое соединение
        let (client, _) = server.accept()
        setBlocking(client.getFd(), false)
        sel.registerHandle(client.getFd().int, {Event.Read},
                           ConnData(label: "client"))
      of "watchdog":
        echo "Сторожевой таймер сработал — завершаем работу"
        shutdownEv.trigger()
      of "shutdown":
        sel.unregister(shutdownEv)
        shutdownEv.close()
        sel.unregister(server.getFd().int)
        server.close()
      of "client":
        if Event.Read in key.events:
          # читаем данные ...
          sel.unregister(key.fd)
      else: discard

sel.close()
```

---

*Справочник охватывает модуль `std/selectors` стандартной библиотеки Nim согласно исходному коду модуля. Платформозависимое поведение (epoll/kqueue/poll/select) абстрагировано самим модулем.*
