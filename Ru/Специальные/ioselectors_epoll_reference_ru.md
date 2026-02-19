# Справочник модуля `ioselectors` — реализация epoll (Nim)

> Этот файл — **Linux-специфичная реализация** универсального интерфейса `ioselectors` поверх системного вызова `epoll(7)`.  
> Предоставляет высокопроизводительное мультиплексирование I/O: мониторинг файловых дескрипторов, сокетов, таймеров, сигналов, пользовательских событий и завершения процессов.

> ℹ️ **Платформа:** только Linux. Не компилируется на macOS, Windows и других ОС.  
> Для работы с сигналами и процессами **не поддерживается** на Android.

---

## Содержание

- [Типы данных](#типы-данных)
- [Создание и закрытие селектора](#создание-и-закрытие-селектора)
- [Регистрация дескрипторов](#регистрация-дескрипторов)
- [Регистрация специальных источников событий](#регистрация-специальных-источников-событий)
- [Обновление и отмена регистрации](#обновление-и-отмена-регистрации)
- [Ожидание событий](#ожидание-событий)
- [Управление пользовательскими событиями](#управление-пользовательскими-событиями)
- [Доступ к данным и утилиты](#доступ-к-данным-и-утилиты)
- [Типы событий (Event)](#типы-событий-event)
- [Концепция работы](#концепция-работы)
- [Краткая шпаргалка](#краткая-шпаргалка)

---

## Типы данных

### `Selector[T]`

Основная структура — обёртка над файловым дескриптором `epoll`. Параметр `T` — произвольный пользовательский тип данных, ассоциируемый с каждым зарегистрированным дескриптором.

- При компиляции **с поддержкой потоков** (`hasThreadSupport`): `ptr SelectorImpl[T]` — размещается в разделяемой памяти.
- Без поддержки потоков: `ref SelectorImpl[T]` — обычный ссылочный объект.

Поле `count` (публичное) содержит количество активно отслеживаемых дескрипторов.

```nim
var sel = newSelector[MyData]()
```

---

### `SelectEvent`

Указатель на пользовательское событие, реализованное через Linux `eventfd`. Используется для пробуждения цикла ожидания из другого потока или произвольного кода.

```nim
var ev = newSelectEvent()
```

---

### `SignalFdInfo`

Структура, отображаемая на `struct signalfd_siginfo` из `<sys/signalfd.h>`. Заполняется ядром при поступлении сигнала через `signalfd`. Основные поля:

| Поле | Тип | Описание |
|---|---|---|
| `ssi_signo` | `uint32` | Номер сигнала |
| `ssi_pid` | `uint32` | PID отправителя |
| `ssi_uid` | `uint32` | UID отправителя |
| `ssi_code` | `int32` | Код источника сигнала |
| `ssi_status` | `int32` | Код выхода / номер сигнала для `SIGCHLD` |

> Не доступна на Android.

---

## Создание и закрытие селектора

### `newSelector[T](): Selector[T]`

Создаёт новый epoll-селектор. Вызывает `epoll_create1(O_CLOEXEC)`. Начальная ёмкость таблицы дескрипторов — 1024 записи (автоматически расширяется по мере необходимости). Максимальный лимит определяется `getrlimit()`.

Вызывает `OSError` при ошибке создания epoll-файлового дескриптора.

```nim
type MyData = object
  name: string

var sel = newSelector[MyData]()
defer: sel.close()
```

---

### `close[T](s: Selector[T])`

Закрывает epoll-дескриптор и освобождает все ресурсы. В многопоточном режиме освобождает разделяемую память.

> Вызовите обязательно по завершении работы, чтобы избежать утечки файловых дескрипторов.

```nim
sel.close()
```

---

## Регистрация дескрипторов

### `registerHandle[T](s: Selector[T], fd: int | SocketHandle, events: set[Event], data: T)`

Регистрирует существующий файловый дескриптор или сокет `fd` для наблюдения за событиями `events`. Пользовательские данные `data` ассоциируются с дескриптором и возвращаются при каждом событии.

Допустимые события: `Event.Read`, `Event.Write` (или их комбинация).  
Дополнительно всегда отслеживается `EPOLLRDHUP` (разрыв соединения).

```nim
import std/net

var sock = newSocket()
sock.bindAddr(Port(8080))
sock.listen()

type ConnData = object
  id: int

var sel = newSelector[ConnData]()
sel.registerHandle(sock.getFd(), {Event.Read}, ConnData(id: 1))
```

---

### `updateHandle[T](s: Selector[T], fd: int | SocketHandle, events: set[Event])`

Изменяет набор отслеживаемых событий для **уже зарегистрированного** дескриптора `fd`. Применим только к обычным дескрипторам и сокетам — не работает с таймерами, сигналами, процессами, пользовательскими событиями и Oneshot-записями.

- Если `events == {}` и дескриптор был активен — удаляет из epoll (`EPOLL_CTL_DEL`).
- Если `events != {}` и дескриптор был неактивен — добавляет в epoll (`EPOLL_CTL_ADD`).
- Иначе — модифицирует маску (`EPOLL_CTL_MOD`).

```nim
# Переключить с чтения на запись:
sel.updateHandle(fd, {Event.Write})

# Временно приостановить мониторинг:
sel.updateHandle(fd, {})

# Возобновить:
sel.updateHandle(fd, {Event.Read})
```

---

## Регистрация специальных источников событий

### `registerTimer[T](s: Selector[T], timeout: int, oneshot: bool, data: T): int`

Регистрирует таймер на основе `timerfd_create(CLOCK_MONOTONIC)`. Возвращает файловый дескриптор таймера (для последующего `unregister`).

- `timeout` — интервал в **миллисекундах**.
- `oneshot = true` — таймер срабатывает один раз и помечается `Event.Finished`; при последующих вызовах `select` больше не появляется, пока не будет отменён регистрацией.
- `oneshot = false` — таймер срабатывает периодически с указанным интервалом.

```nim
type TimerData = object
  label: string

# Периодический таймер каждые 500 мс:
let timerFd = sel.registerTimer(500, false, TimerData(label: "tick"))

# Одноразовый таймер через 2 секунды:
let onceFd = sel.registerTimer(2000, true, TimerData(label: "once"))
```

---

### `registerSignal[T](s: Selector[T], signal: int, data: T): int`

Регистрирует наблюдение за POSIX-сигналом через `signalfd`. Блокирует сигнал в маске текущего потока и перенаправляет его в epoll. Возвращает дескриптор `signalfd`.

> ℹ️ **Не доступно на Android.**

```nim
import std/posix

type SigData = object
  signum: int

# Перехватить SIGTERM:
let sigFd = sel.registerSignal(SIGTERM, SigData(signum: SIGTERM))
```

---

### `registerProcess[T](s: Selector, pid: int, data: T): int`

Регистрирует наблюдение за завершением дочернего процесса с заданным `pid` через `SIGCHLD` и `signalfd`. Помечает событие как `Event.Oneshot` — срабатывает один раз при завершении процесса. Возвращает дескриптор `signalfd`.

> ℹ️ **Не доступно на Android.**

```nim
type ProcData = object
  pid: int

let childPid = 12345
let procFd = sel.registerProcess(childPid, ProcData(pid: childPid))
```

---

### `registerEvent[T](s: Selector[T], ev: SelectEvent, data: T)`

Регистрирует пользовательское событие `ev` (созданное `newSelectEvent()`) в селекторе. Событие будет доставлено при вызове `trigger(ev)`.

```nim
var ev = newSelectEvent()
sel.registerEvent(ev, MyData(id: 42))
```

---

## Обновление и отмена регистрации

### `unregister[T](s: Selector[T], fd: int | SocketHandle)`

Отменяет регистрацию дескриптора `fd`. Корректно обрабатывает все типы: обычные дескрипторы/сокеты, таймеры (закрывает `timerfd`), сигналы и процессы (разблокирует сигнал, закрывает `signalfd`).

```nim
sel.unregister(sock.getFd())
sel.unregister(timerFd)
```

---

### `unregister[T](s: Selector[T], ev: SelectEvent)`

Отменяет регистрацию пользовательского события `ev`. Удаляет `eventfd`-дескриптор из epoll. Не закрывает сам `ev` — для этого используйте `ev.close()`.

```nim
sel.unregister(ev)
ev.close()
```

---

## Ожидание событий

### `select[T](s: Selector[T], timeout: int): seq[ReadyKey]`

Ожидает событий и возвращает последовательность готовых ключей `ReadyKey`. Вызывает `epoll_wait` с указанным таймаутом.

- `timeout > 0` — ждать не более `timeout` миллисекунд.
- `timeout == 0` — вернуть немедленно (неблокирующий опрос).
- `timeout == -1` — блокировать бесконечно.

Возвращает не более 64 событий за один вызов. Прерывание сигналом (`EINTR`) не считается ошибкой — возвращает пустой список.

```nim
while true:
  let events = sel.select(100)   # ждать 100 мс
  for key in events:
    echo "Событие на fd=", key.fd, " events=", key.events
```

---

### `selectInto[T](s: Selector[T], timeout: int, results: var openArray[ReadyKey]): int`

Аналог `select`, но записывает результаты в **предоставленный** буфер `results`. Возвращает количество заполненных элементов. Позволяет переиспользовать буфер без выделения памяти на каждый вызов.

```nim
var buf: array[64, ReadyKey]
while true:
  let n = sel.selectInto(100, buf)
  for i in 0 ..< n:
    processEvent(buf[i])
```

---

## Управление пользовательскими событиями

### `newSelectEvent(): SelectEvent`

Создаёт новый объект пользовательского события на основе Linux `eventfd(O_CLOEXEC | O_NONBLOCK)`. Используется для межпотоковой или межкомпонентной сигнализации.

```nim
var wakeup = newSelectEvent()
```

---

### `trigger(ev: SelectEvent)`

Активирует (сигнализирует) пользовательское событие. Записывает `1` в `eventfd`, что приводит к срабатыванию события `Event.User` в ожидающем `select` / `selectInto`.

Может вызываться из другого потока.

```nim
# В производителе / другом потоке:
wakeup.trigger()

# В главном цикле событий после select():
for key in events:
  if Event.User in key.events:
    echo "Получен сигнал пробуждения"
```

---

### `close(ev: SelectEvent)`

Закрывает `eventfd`-дескриптор и освобождает память. Вызывайте после `unregister(ev)`.

```nim
sel.unregister(wakeup)
wakeup.close()
```

---

## Доступ к данным и утилиты

### `getData[T](s: Selector[T], fd: SocketHandle | int): var T`

Возвращает изменяемую ссылку на пользовательские данные, ассоциированные с зарегистрированным дескриптором `fd`. Если `fd` не зарегистрирован — поведение не определено.

```nim
var d = sel.getData(sock.getFd())
d.name = "новое имя"
```

---

### `setData[T](s: Selector[T], fd: SocketHandle | int, data: T): bool`

Обновляет пользовательские данные для дескриптора `fd`. Возвращает `true`, если дескриптор зарегистрирован и данные обновлены; `false` иначе.

```nim
let ok = sel.setData(fd, MyData(id: 99))
assert ok
```

---

### `withData[T](s, fd, value, body)`  — шаблон (2 формы)

Безопасно обращается к данным дескриптора — выполняет `body`, только если `fd` зарегистрирован. Переменная `value` внутри `body` — указатель на данные (`ptr T`), позволяет изменять их на месте.

**Форма 1 — только при наличии:**
```nim
sel.withData(fd, val):
  val.counter += 1
```

**Форма 2 — с веткой «иначе»:**
```nim
sel.withData(fd, val):
  echo "fd зарегистрирован: ", val.name
do:
  echo "fd не найден"
```

---

### `contains[T](s: Selector[T], fd: SocketHandle | int): bool`

Возвращает `true`, если дескриптор `fd` зарегистрирован в селекторе. Поддерживает оператор `in`.

```nim
if fd in sel:
  echo "fd активен"

assert sel.contains(sock.getFd())
```

---

### `isEmpty[T](s: Selector[T]): bool`  — шаблон

Возвращает `true`, если в селекторе нет ни одного отслеживаемого дескриптора (`s.count == 0`).

```nim
if sel.isEmpty():
  echo "Нет активных дескрипторов"
```

---

### `getFd[T](s: Selector[T]): int`

Возвращает сырой файловый дескриптор epoll (`epollFD`). Используется для вложенного мультиплексирования — когда нужно зарегистрировать один epoll-fd в другом.

```nim
let epollFd = sel.getFd()
```

---

## Типы событий (Event)

Набор `set[Event]` используется при регистрации и возвращается в `ReadyKey.events`:

| Значение | Описание |
|---|---|
| `Event.Read` | Данные доступны для чтения (`EPOLLIN`) |
| `Event.Write` | Запись возможна без блокировки (`EPOLLOUT`) |
| `Event.Timer` | Таймер сработал |
| `Event.Signal` | Получен POSIX-сигнал (через `signalfd`) |
| `Event.Process` | Дочерний процесс завершился |
| `Event.User` | Сработало пользовательское событие (`eventfd`) |
| `Event.Error` | Ошибка на дескрипторе (`EPOLLERR` / `EPOLLHUP`) |
| `Event.Oneshot` | Событие срабатывает однократно (затем удаляется из epoll) |
| `Event.Finished` | Внутренний флаг: Oneshot-событие уже использовано |

### `ReadyKey`

Структура, возвращаемая `select` / `selectInto`:

```nim
type ReadyKey = object
  fd: int             # файловый дескриптор
  events: set[Event]  # набор произошедших событий
  errorCode: OSErrorCode  # код ошибки при Event.Error
```

---

## Концепция работы

### Жизненный цикл

```
newSelector()               # epoll_create1()
  │
  ├─ registerHandle(fd, ...)   # epoll_ctl(EPOLL_CTL_ADD)
  ├─ registerTimer(ms, ...)    # timerfd_create() + epoll_ctl(ADD)
  ├─ registerSignal(sig, ...)  # signalfd() + epoll_ctl(ADD)
  ├─ registerProcess(pid, ...) # signalfd(SIGCHLD) + epoll_ctl(ADD)
  ├─ registerEvent(ev, ...)    # eventfd уже создан, epoll_ctl(ADD)
  │
  └─ loop:
       select(timeout)         # epoll_wait()
         │
         ├─ Event.Read / Write → обработать данные
         ├─ Event.Timer        → читаем uint64 из timerfd
         ├─ Event.Signal       → читаем SignalFdInfo из signalfd
         ├─ Event.Process      → читаем SignalFdInfo, проверяем pid
         ├─ Event.User         → читаем uint64 из eventfd
         └─ Event.Error        → читаем SO_ERROR через getsockopt
  │
  ├─ unregister(fd)            # epoll_ctl(EPOLL_CTL_DEL), закрытие fd
  └─ close()                   # закрытие epollFD, освобождение памяти
```

### Автоматическое расширение таблицы

Внутренняя таблица дескрипторов начинается с 1024 записей и удваивается при необходимости (шаблон `checkFd`). Максимум ограничен системным лимитом `getrlimit(RLIMIT_NOFILE)`.

### Oneshot-события

При регистрации с `Event.Oneshot` (таймеры, процессы) epoll устанавливает флаг `EPOLLONESHOT`. После первого срабатывания дескриптор помечается `Event.Finished` и убирается из активного мониторинга, но **остаётся** в таблице — данные можно прочитать через `getData`. Для полного удаления нужен явный вызов `unregister`.

---

## Краткая шпаргалка

| Операция | Функция |
|---|---|
| Создать селектор | `newSelector[T]()` |
| Закрыть селектор | `close(s)` |
| Зарегистрировать fd/сокет | `registerHandle(s, fd, events, data)` |
| Обновить события fd | `updateHandle(s, fd, events)` |
| Отменить регистрацию fd | `unregister(s, fd)` |
| Зарегистрировать таймер | `registerTimer(s, ms, oneshot, data)` |
| Зарегистрировать сигнал | `registerSignal(s, signal, data)` |
| Зарегистрировать процесс | `registerProcess(s, pid, data)` |
| Создать польз. событие | `newSelectEvent()` |
| Зарегистрировать польз. событие | `registerEvent(s, ev, data)` |
| Сигнализировать событие | `trigger(ev)` |
| Закрыть польз. событие | `close(ev)` |
| Отменить польз. событие | `unregister(s, ev)` |
| Ждать событий → seq | `select(s, timeout)` |
| Ждать событий → буфер | `selectInto(s, timeout, buf)` |
| Получить данные fd | `getData(s, fd)` |
| Установить данные fd | `setData(s, fd, data)` |
| Безопасный доступ к данным | `withData(s, fd, val): body` |
| Проверить регистрацию | `contains(s, fd)` / `fd in s` |
| Нет дескрипторов? | `isEmpty(s)` |
| Получить epoll fd | `getFd(s)` |

### Типовой цикл событий

```nim
import std/[ioselectors, net, os]

type ConnData = object
  id: int

var sel = newSelector[ConnData]()
defer: sel.close()

var server = newSocket()
server.setSockOpt(OptReuseAddr, true)
server.bindAddr(Port(8080))
server.listen()
sel.registerHandle(server.getFd(), {Event.Read}, ConnData(id: 0))

while true:
  let events = sel.select(1000)
  for key in events:
    if Event.Error in key.events:
      echo "Ошибка на fd=", key.fd
      sel.unregister(key.fd)
    elif Event.Read in key.events:
      if key.fd == server.getFd().int:
        var client = server.accept()[0]
        sel.registerHandle(client.getFd(), {Event.Read}, ConnData(id: key.fd))
      else:
        # чтение данных от клиента
        discard
```

### Примечания

- Модуль — **внутренняя реализация**, используемая через `std/ioselectors`. Для переносимого кода импортируйте именно `std/ioselectors`.
- `select` возвращает не более **64 событий** за вызов (`MAX_EPOLL_EVENTS`).
- Прерывание `EINTR` в `epoll_wait` обрабатывается корректно — `select` возвращает 0 без ошибки.
- `registerSignal` и `registerProcess` **недоступны** на Android.
- В многопоточном режиме (`--threads:on`) `Selector` размещается в разделяемой памяти (`ptr`) и может использоваться из нескольких потоков.
