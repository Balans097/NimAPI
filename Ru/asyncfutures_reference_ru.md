# `asyncfutures.nim` — Справочник модуля

> **Импорт:** `import std/asyncfutures`
>
> **Область применения:** Асинхронное программирование — низкоуровневые примитивы Future для координации конкурентных операций в однопоточном цикле событий без блокировки потока ОС.

Модуль `asyncfutures` является фундаментальным слоем асинхронной экосистемы Nim. Он определяет тип `Future[T]` — заместитель для значения, которого ещё не существует — вместе с механизмами завершения, сигнализации об ошибке и композиции Future. Модуль **не** запускает цикл событий самостоятельно — это задача `asyncdispatch`. Вместо этого он предоставляет примитивы, на которых строятся и диспетчер, и пользовательский код: создание, завершение, регистрация коллбэков, операторы композиции (`and`, `or`, `all`) и подключаемый хук планировщика (`callSoon`).

Всё, что есть в `asyncdispatch`, макросах трансформации `async`/`await` и любых сторонних асинхронных библиотеках для Nim, в конечном счёте опирается на типы и процедуры, определённые здесь.

---

## Оглавление

1. [Обзор API](#обзор-api)
   - [Типы](#обзор-типов)
   - [Функции и операторы](#обзор-функций-и-операторов)
   - [Константы](#обзор-констант)
2. [Константы, типы и вспомогательные средства](#константы-типы-и-вспомогательные-средства)
   - [`isFutureLoggingEnabled`](#isfutureloggingenabled)
   - [`NimAsyncContinueSuffix`](#nimasynccontinuesuffix)
   - [`FutureBase`](#futurebase)
   - [`Future[T]`](#futuret)
   - [`FutureVar[T]`](#futurevart)
   - [`FutureError`](#futureerror)
   - [`FutureInfo`](#futureinfo-только-при-futurelogging)
   - [`CallbackList` (внутренний)](#callbacklist-внутренний)
   - [`CallbackFunc` (внутренний)](#callbackfunc-внутренний)
3. [Справочник функций](#справочник-функций)
   - [`newFuture`](#newfuturet-1)
   - [`newFutureVar`](#newfuturevart-1)
   - [`clean`](#cleant-1)
   - [`complete` — Future[T]](#completet--futuret-1)
   - [`complete` — Future[void]](#complete--futurevoid-1)
   - [`complete` — FutureVar (без значения)](#complete--futurevar-без-значения)
   - [`complete` — FutureVar (со значением)](#complete--futurevar-со-значением)
   - [`fail`](#failt-1)
   - [`read`](#readt-1)
   - [`readError`](#readerrort-1)
   - [`mget`](#mgett-1)
   - [`finished`](#finished-1)
   - [`failed`](#failed-1)
   - [`asyncCheck`](#asynccheckt-1)
   - [`addCallback` (без параметров)](#addcallback-без-параметров)
   - [`addCallback` (типизированный)](#addcallback-типизированный)
   - [`callback=` (без параметров)](#callback-без-параметров)
   - [`callback=` (типизированный)](#callback-типизированный)
   - [`clearCallbacks`](#clearcallbacks-1)
   - [`callSoon`](#callsoon-1)
   - [`getCallSoonProc`](#getcallsoonproc-1)
   - [`setCallSoonProc`](#setcallsoonproc-1)
   - [`and`](#and-1)
   - [`or`](#or-1)
   - [`all`](#allt-1)
   - [`getFuturesInProgress`](#getfuturesinprogress-только-при-futurelogging)
4. [Диагностика и отладка](#диагностика-и-отладка)
5. [Потокобезопасность](#потокобезопасность)
6. [Полный пример](#полный-пример)

---

## Обзор API

### Обзор типов

| Тип | Вид | Описание |
|---|---|---|
| `FutureBase` | `ref object` | Нетипизированная база для всех Future. Хранит коллбэки, статус, ошибку. |
| `Future[T]` | `ref object of FutureBase` | Типизированный Future, хранящий значение типа `T`. |
| `FutureVar[T]` | `distinct Future[T]` | Переиспользуемая обёртка над Future для экономии аллокаций. |
| `FutureError` | `object of Defect` | Выбрасывается при повторном завершении одного Future. |
| `FutureInfo` | `object` | Отладочная запись (только с флагом `-d:futureLogging`). |

### Обзор функций и операторов

| Процедура | Сигнатура | Описание |
|---|---|---|
| `newFuture` | `[T](fromProc = "unspecified"): owned Future[T]` | Создать новый незавершённый Future. |
| `newFutureVar` | `[T](fromProc = "unspecified"): owned FutureVar[T]` | Создать новый переиспользуемый FutureVar. |
| `clean` | `[T](future: FutureVar[T])` | Сбросить FutureVar для повторного использования. |
| `complete` | `[T](future: Future[T], val: sink T)` | Завершить типизированный Future значением. |
| `complete` | `(future: Future[void])` | Завершить void-Future. |
| `complete` | `[T](future: FutureVar[T])` | Завершить FutureVar (сохраняя текущее значение). |
| `complete` | `[T](future: FutureVar[T], val: sink T)` | Завершить FutureVar с новым значением. |
| `fail` | `[T](future: Future[T], error: ref Exception)` | Завершить Future с ошибкой. |
| `read` | `[T](future: Future[T] \| FutureVar[T]): lent T` | Прочитать значение завершённого Future. |
| `readError` | `[T](future: Future[T]): ref Exception` | Вернуть хранящееся исключение без его выброса. |
| `mget` | `[T](future: FutureVar[T]): var T` | Изменяемый доступ к хранимому значению (без проверки завершённости). |
| `finished` | `(future: FutureBase \| FutureVar): bool` | True, если Future завершён (успешно или с ошибкой). |
| `failed` | `(future: FutureBase): bool` | True, если Future завершился с ошибкой. |
| `asyncCheck` | `[T](future: Future[T])` | Установить коллбэк, выбрасывающий ошибку, если Future завершился неудачно. |
| `addCallback` | `(future: FutureBase, cb: proc() {.closure, gcsafe.})` | Добавить коллбэк без параметров. |
| `addCallback` | `[T](future: Future[T], cb: proc(future: Future[T]))` | Добавить типизированный коллбэк, принимающий Future. |
| `callback=` | `(future: FutureBase, cb: proc())` | Заменить все коллбэки одним (без параметров). |
| `callback=` | `[T](future: Future[T], cb: proc(future: Future[T]))` | Заменить все коллбэки одним (типизированным). |
| `clearCallbacks` | `(future: FutureBase)` | Удалить все зарегистрированные коллбэки. |
| `callSoon` | `(cbproc: proc() {.gcsafe.})` | Запланировать процедуру на следующий тик диспетчера. |
| `getCallSoonProc` | `(): proc(cbproc: proc()) {.gcsafe.}` | Получить текущую реализацию `callSoon`. |
| `setCallSoonProc` | `(p: proc(cbproc: proc()) {.gcsafe.})` | Заменить реализацию `callSoon`. |
| `and` | `[T,Y](fut1: Future[T], fut2: Future[Y]): Future[void]` | Завершиться, когда завершатся **оба** Future. |
| `or` | `[T,Y](fut1: Future[T], fut2: Future[Y]): Future[void]` | Завершиться, когда завершится **любой** из Future. |
| `all` | `[T](futs: varargs[Future[T]]): auto` | Завершиться, когда завершатся **все** Future из списка. |
| `getFuturesInProgress` | `(): var Table[FutureInfo, int]` | *(futureLogging)* Живая таблица незавершённых Future. |

### Обзор констант

| Константа | Тип | Значение | Описание |
|---|---|---|---|
| `isFutureLoggingEnabled` | `bool` | `defined(futureLogging)` | Включён ли диагностический журнал Future. |
| `NimAsyncContinueSuffix` | `string` | `"NimAsyncContinue"` | Внутренний суффикс, используемый макросами async. |

---

## Константы, типы и вспомогательные средства

### `isFutureLoggingEnabled`

```nim
const isFutureLoggingEnabled* = defined(futureLogging)
```

Булева константа времени компиляции, которая равна `true`, если программа собрана с флагом `-d:futureLogging`. При включении каждый вызов `newFuture` регистрирует новый Future в потоко-локальной таблице (`futuresInProgress`), а каждый `complete`/`fail` удаляет его оттуда. Это позволяет в любой момент исполнения проверить, какие Future всё ещё ожидают завершения — полезно для поиска утечек Future в долгоживущих серверах.

Константа также управляет определением `FutureInfo`, `getFuturesInProgress` и внутренних вспомогательных функций `logFutureStart`/`logFutureFinish`. Без флага эти символы не существуют вовсе.

---

### `NimAsyncContinueSuffix`

```nim
const NimAsyncContinueSuffix* = "NimAsyncContinue"
```

Строковая метка, используемая внутренне при трансформации `async`/`await` макросом компилятора. Когда компилятор переписывает `async proc` в конечный автомат-продолжение, сгенерированные вспомогательные процедуры получают это имя в качестве суффикса. Пользовательский код никогда не должен конструировать или сравнивать эту строку напрямую. Она экспортируется только для инструментального слоя (отладчики, форматтеры трассировок), чтобы они могли распознавать и фильтровать сгенерированные имена.

---

### `FutureBase`

```nim
type
  FutureBase* = ref object of RootObj
    callbacks: CallbackList        # связный список ожидающих коллбэков
    finished: bool                 # true после complete() или fail()
    error*: ref Exception          # не nil при завершении с ошибкой
    errorStackTrace*: string       # трассировка стека, записанная в fail()
    # только в debug-режиме (отсутствуют при -d:release):
    stackTrace: seq[StackTraceEntry]
    id: int
    fromProc: string
```

`FutureBase` — нетипизированный корень иерархии типов Future. Он существует для того, чтобы код, которому не нужно знать тип значения, — механизм коллбэков, операторы `and`/`or`, `asyncCheck`, интроспекция ошибок — мог работать с любым Future через единый унифицированный интерфейс.

**Публичные поля:**

- `error` — хранит `ref Exception`, переданный в `fail()`. Чтение безопасно в любое время; значение равно `nil` для незавершённого или успешно завершённого Future.
- `errorStackTrace` — строка трассировки стека, захваченная при вызове `fail()`. Оператор `$` над `seq[StackTraceEntry]` форматирует её, отфильтровывая внутренние фреймы Nim async.

**Поля только для debug** (удаляются при `-d:release`):

- `stackTrace` — стек вызовов в момент вызова `newFuture`. Выводится в сообщениях `FutureError` и в асинхронной трассировке, добавляемой `injectStacktrace`.
- `id` — монотонно возрастающее целое число (`currentID`), присваиваемое при создании. Уникально в рамках потока и одного запуска.
- `fromProc` — строка, переданная в `newFuture("имяПроцедуры")`. Идентифицирует владеющую процедуру в сообщениях об ошибках.

---

### `Future[T]`

```nim
type
  Future*[T] = ref object of FutureBase
    value: T
```

Основной типизированный Future. Содержит хранимое значение типа `T` вместе с унаследованными от `FutureBase` статусом, коллбэками и ошибкой. Поскольку это `ref object`, присваивание копирует ссылку (не объект), и передача Future занимает O(1).

Поле `value` закрыто; его можно прочитать только через `read()` или `mget()` (последний — только для `FutureVar[T]`). Запись в `value` из пользовательского кода невозможна — оно устанавливается исключительно через `complete()`.

Когда `T` равно `void`, `Future[void]` используется для операций, которые сигнализируют о завершении без производства значения. `complete()` для `Future[void]` не принимает аргументов; `read()` только выполняет проверку ошибки и возвращает `void`.

---

### `FutureVar[T]`

```nim
type
  FutureVar*[T] = distinct Future[T]
```

`FutureVar[T]` — обёртка (`distinct`) над `Future[T]`. Ключевое отличие: он явно спроектирован для повторного использования. После вызова `complete()` нужно вызвать `clean()` для сброса флага `finished`, и тогда тот же объект в heap можно завершать снова. Это устраняет нагрузку на аллокатор и GC от создания нового `Future[T]` на каждую итерацию операции.

Тип `distinct` предотвращает случайное смешение с `Future[T]`. Процедуры, принимающие `FutureVar[T]` — `clean`, `mget` и специфичные перегрузки `complete` — отдельны от процедур для `Future[T]`.

**Типичный жизненный цикл:**

```
newFutureVar → [цикл: mget → заполнить → complete → read → clean → повторить]
```

`FutureVar` наиболее полезен в плотных I/O-циклах (например, чтение из сокета кусками фиксированного размера), где накладные расходы на аллокацию имеют значение.

---

### `FutureError`

```nim
type
  FutureError* = object of Defect
    cause*: FutureBase
```

Выбрасывается внутренней защитой `checkFinished`, когда `complete()` или `fail()` вызывается на уже завершённом Future. Поле `cause` содержит провинившийся Future, чтобы можно было проверить его `id`, `fromProc` и захваченный `stackTrace`.

Полное сообщение об ошибке включает:
- Числовой ID Future и строку `fromProc`.
- Трассировку стека в момент вызова `newFuture` (место создания).
- Строковое значение, если `T == string` (для облегчения отладки).
- Трассировку стека из второй (ошибочной) точки завершения.

`FutureError` — это `Defect`, а не `Exception`. Дефекты в Nim представляют программные ошибки, которые в production-коде не ожидается перехватывать через `try/except`.

**Эта защита полностью отсутствует в сборках `-d:release`.** Двойное завершение в release-режиме приводит к тихому повреждению состояния Future или, в лучшем случае, к лишним вызовам коллбэков.

---

### `FutureInfo` *(только при futureLogging)*

```nim
when isFutureLoggingEnabled:
  type
    FutureInfo* = object
      stackTrace*: seq[StackTraceEntry]
      fromProc*: string
```

Составной ключ, используемый в таблице `futuresInProgress`. Два Future считаются «одного вида», если у них совпадают трассировка стека при создании и строка `fromProc`. Таблица считает, сколько Future каждого вида в данный момент выполняется.

`FutureInfo` реализует пользовательский `hash`, который комбинирует хеши элементов `stackTrace` (каждый хешируется из `procname`, `line` и `filename`) и `fromProc`.

---

### `CallbackList` *(внутренний)*

```nim
type
  CallbackList = object
    function: CallbackFunc
    next: owned(ref CallbackList)
```

Односвязный список коллбэков, встроенный напрямую в `FutureBase`. Первый коллбэк хранится **inline** в объекте `FutureBase` (нет аллокации heap для типичного случая с одним коллбэком). Дополнительные коллбэки аллоцируются в heap через `owned ref`.

Когда вызывается `callbacks.call()` (внутри `complete`/`fail`), он обходит список, вызывая `callSoon(fn)` для каждой ненулевой функции, а затем обнуляет оба поля — `function` и `next`. Это позволяет GC немедленно собрать всю цепочку коллбэков после её срабатывания, а не ждать окончания жизни самого Future.

Обход в порядке добавления гарантирует вызов коллбэков в том же порядке, в котором они были зарегистрированы (FIFO).

---

### `CallbackFunc` *(внутренний)*

```nim
type
  CallbackFunc = proc() {.closure, gcsafe.}
```

Конкретный тип для каждого коллбэка, хранящегося в `CallbackList`. Прагма `{.closure.}` означает, что процедура может захватывать переменные из своего замыкающего контекста. Прагма `{.gcsafe.}` утверждает, что захваченные ссылки безопасны для передачи GC между потоками (важно для интерфейса `callSoon` при использовании многопоточного диспетчера). Пользовательский код никогда не именует этот тип напрямую — он выводится из лямбда-литералов, передаваемых в `addCallback` или `callback=`.

---

## Справочник функций

---

### `newFuture[T]`

```nim
proc newFuture*[T](fromProc: string = "unspecified"): owned(Future[T])
```

Создаёт и возвращает новый незавершённый `Future[T]`.

**Параметры:**

| Параметр | Тип | По умолчанию | Описание |
|---|---|---|---|
| `fromProc` | `string` | `"unspecified"` | Имя процедуры-создателя. Отображается в отладочных сообщениях. |

**Возвращает:** `owned Future[T]` — вызывающий код принимает единоличное владение.

**Поведение:**

1. Аллоцирует новый `Future[T]` в heap (`new(result)` через шаблон `setupFutureBase`).
2. Устанавливает `finished = false`.
3. В не-release сборках: захватывает текущий стек вызовов через `getStackTraceEntries()`, записывает его в `stackTrace`, присваивает следующее значение `currentID` и записывает `fromProc`.
4. Если `isFutureLoggingEnabled`: вызывает `logFutureStart`, увеличивая счётчик для ключа `FutureInfo` этого Future в `futuresInProgress`.

**Особенности реализации:** Шаблон `setupFutureBase` (не экспортируется) является фактической реализацией. Это именно `template`, а не `proc`, чтобы `getStackTraceEntries()` захватывал стек **вызывающего**, а не стек внутри `newFuture`. Это критично: если бы это была `proc`, трассировка стека всегда указывала бы внутрь `newFuture`, что было бы бесполезно.

Тип возврата `owned` участвует в экспериментальной системе владения Nim. На практике для большинства кода это означает «вызывающий несёт ответственность за время жизни этого объекта» — Future следует сохранять в переменной или возвращать, а не отбрасывать.

```nim
import std/asyncfutures

let f = newFuture[int]("вычислитьСумму")
assert not f.finished
assert f.error == nil

let g = newFuture[string]()   # fromProc по умолчанию "unspecified"
```

---

### `newFutureVar[T]`

```nim
proc newFutureVar*[T](fromProc = "unspecified"): owned(FutureVar[T])
```

Создаёт новый `FutureVar[T]` — переиспользуемую обёртку над Future.

Внутренне вызывает `newFuture[T](fromProc)` и оборачивает результат в `FutureVar[T]` через приведение `typeof(result)(fo)`. Как `Future[T]`, так и обёртка `FutureVar[T]` разделяют один и тот же объект в heap; обёртка `distinct` ничего не стоит во время выполнения.

При включённом `isFutureLoggingEnabled` вызывается `logFutureStart` для базового `Future[T]` (приведённого из `FutureVar[T]`).

```nim
import std/asyncfutures

var fv = newFutureVar[seq[byte]]("socket.readChunk")
# fv можно переиспользовать многократно через clean()
```

---

### `clean[T]`

```nim
proc clean*[T](future: FutureVar[T])
```

Сбрасывает `FutureVar[T]` так, чтобы его можно было завершить снова.

Устанавливает `Future[T](future).finished = false` и `Future[T](future).error = nil`.

**Что НЕ сбрасывается:**

- `value` — ранее сохранённое значение остаётся на месте. Это намеренно: можно проверить старое значение после `clean()`, а следующий `complete(val)` просто перезапишет его.
- `callbacks` — список коллбэков не сбрасывается. Любые коллбэки, не срабатывавшие (зарегистрированные после завершения Future) — остаются. На практике `clean()` вызывается уже после того, как все коллбэки сработали и были очищены `callbacks.call()`, так что список уже пуст.
- Debug-поля (`stackTrace`, `id`, `fromProc`) — не меняются. Future сохраняет свою исходную идентичность для диагностики.

**Почему только `FutureVar`?** Сброс обычного `Future[T]` был бы логической ошибкой: любой код, удерживающий на него ссылку и уже наблюдавший `finished == true`, молча увидел бы, что он снова стал незавершённым. `FutureVar` — это намеренный, явный сигнал, что повторное использование запланировано.

```nim
var fv = newFutureVar[int]("пример")
fv.complete(1); echo fv.read()   # 1
fv.clean()
assert not fv.finished
fv.complete(2); echo fv.read()   # 2
```

---

### `complete[T]` — `Future[T]`

```nim
proc complete*[T](future: Future[T], val: sink T)
```

Успешно завершает `future`, сохраняя `val` как его результат.

**Шаги (через `completeImpl`):**

1. Вызывает `checkFinished(future)` — в debug-сборках выбрасывает `FutureError`, если Future уже завершён.
2. Утверждает `future.error == nil` (проверка корректности; должна всегда выполняться в этой точке).
3. Записывает `val` в `future.value` (с семантикой `sink` — когда компилятор может это организовать, право владения передаётся, а не копируется).
4. Устанавливает `future.finished = true`.
5. Вызывает `future.callbacks.call()` — обходит список коллбэков и планирует каждый через `callSoon`.
6. При включённом логировании: вызывает `logFutureFinish`, уменьшая счётчик в `futuresInProgress`.

После `complete`: `future.finished == true`, `future.error == nil`, `future.value` содержит `val`. Любой вызов `read()` вернёт (заимствованную ссылку на) `val`.

```nim
let f = newFuture[string]("приветствие")
f.complete("привет, мир")
assert f.finished and not f.failed
assert f.read() == "привет, мир"
```

---

### `complete` — `Future[void]`

```nim
proc complete*(future: Future[void], val = Future[void].default)
```

Завершает `Future[void]`. Параметр `val` — фиктивный с дефолтным значением, никогда не используется; он существует только для удовлетворения обобщённого `completeImpl`. На практике всегда вызывайте как `future.complete()` без аргумента.

```nim
let f = newFuture[void]("сигнал")
f.complete()
assert f.finished
```

---

### `complete` — `FutureVar` (без значения)

```nim
proc complete*[T](future: FutureVar[T])
```

Завершает `FutureVar[T]`, используя значение, уже хранящееся в `mget()`. Вызывает `checkFinished`, устанавливает `finished = true`, запускает коллбэки. Параметра значения нет: эта перегрузка предназначена для паттерна `mget → заполнить → complete`, при котором буфер мутируется на месте, а затем сигнализируется готовность.

```nim
var fv = newFutureVar[array[4, byte]]("readFixed")
fv.mget() = [0xDE'u8, 0xAD, 0xBE, 0xEF]
fv.complete()    # сигнал о готовности буфера
```

---

### `complete` — `FutureVar` (со значением)

```nim
proc complete*[T](future: FutureVar[T], val: sink T)
```

Завершает `FutureVar[T]`, перезаписывая хранимое значение на `val`. Эквивалентно `fv.mget() = val; fv.complete()`, но в одном вызове.

```nim
var fv = newFutureVar[string]("строка")
fv.complete("первая строка")
echo fv.read()    # "первая строка"
fv.clean()
fv.complete("вторая строка")
echo fv.read()    # "вторая строка"
```

---

### `fail[T]`

```nim
proc fail*[T](future: Future[T], error: ref Exception)
```

Завершает `future` в состоянии ошибки.

**Шаги:**

1. `checkFinished(future)` — в debug-сборках защита от двойного завершения.
2. Устанавливает `future.finished = true`.
3. Устанавливает `future.error = error`.
4. Захватывает трассировку стека:
   - Если `getStackTrace(error)` непустой (исключение уже было выброшено и поймано) — используется эта трассировка.
   - Иначе — используется `getStackTrace()` из текущей точки вызова.
   Это сохраняет оригинальное место выброса, когда исключение поймано и передано в `fail`, а не заменяет его местом вызова `fail`.
5. Запускает коллбэки через `future.callbacks.call()`.
6. При логировании: вызывает `logFutureFinish`.

После `fail`: `future.finished == true`, `future.failed == true`, любой вызов `read()` перевыбросит `future.error` (с расширенной асинхронной трассировкой в debug-сборках).

```nim
let f = newFuture[int]("читатьПорт")
try:
  raise newException(OSError, "соединение отклонено")
except OSError as e:
  f.fail(e)       # оригинальная трассировка OSError сохранена

assert f.failed
assert f.error of OSError
```

---

### `read[T]`

```nim
proc read*[T](future: Future[T] | FutureVar[T]): lent T
proc read*(future: Future[void] | FutureVar[void])
```

Возвращает значение завершённого Future.

**Тип возврата:** `lent T` — заимствованная ссылка на внутреннее поле `value`, без копирования. Для `void` ничего не возвращается.

**Поведение:**

- Если `future.finished` и `future.error == nil`: возвращает (заимствование) `future.value`.
- Если `future.finished` и `future.error != nil`: в debug-сборках вызывает `injectStacktrace(future)` для добавления асинхронной трассировки к сообщению исключения, затем перевыбрасывает `future.error`.
- Если `not future.finished`: выбрасывает `ValueError("Future still in progress.")`.

**Детали `injectStacktrace`:** Функция проверяет, присутствует ли уже заголовок `"\nAsync traceback:\n"` в сообщении исключения (идемпотентность — повторный вызов `read()` на одном и том же провалившемся Future не дублирует секцию). Если отсутствует, добавляет:
- Форматированную асинхронную трассировку из `getStackTraceEntries(future.error)`.
- Строку `"Exception message: <исходное сообщение>\n"` в качестве подвала.

Внутри используется шаблон `readImpl` с локальной переменной `{.cursor.}`, чтобы избежать лишней операции со счётчиком ссылок на Future.

```nim
let f = newFuture[float]("вычисление")
f.complete(3.14)
let x: float = f.read()     # lent float — без копирования
assert x == 3.14

# Чтение незавершённого Future:
let g = newFuture[int]("ожидание")
try:
  discard g.read()
except ValueError as e:
  assert e.msg == "Future still in progress."
```

---

### `readError[T]`

```nim
proc readError*[T](future: Future[T]): ref Exception
```

Возвращает исключение, хранящееся в `future`, без его перевыброса.

Выбрасывает `ValueError("No error in future.")`, если `future.error == nil`.

Это аналог `read()` для режима «только инспекция без выброса». Полезен, когда нужен сам объект исключения — для логирования, оборачивания в другое исключение или проверки типа — без запуска механизма обработки исключений.

```nim
let f = newFuture[int]("сеть")
f.fail(newException(TimeoutError, "время ожидания чтения истекло"))

let err = f.readError()
assert err of TimeoutError
echo err.msg     # "время ожидания чтения истекло"
# err НЕ был выброшен — мы только его проинспектировали
```

---

### `mget[T]`

```nim
proc mget*[T](future: FutureVar[T]): var T
```

Возвращает изменяемую ссылку (`var T`) на значение, хранящееся внутри `FutureVar[T]`. В отличие от `read()`, **не** проверяет `finished`, не выбрасывает исключение на незавершённом Future и не перевыбрасывает ошибки. Это «сырой» доступ к слоту хранения.

Типичное использование: подготовить выходной буфер **перед** сигнализацией о завершении, избегая лишней аллокации. Поскольку возвращаемая ссылка является `var`, можно индексировать в неё, передавать в `copyMem` или присваивать срезы — всё это без затрагивания статуса Future.

**Доступен только для `FutureVar[T]`**, не для `Future[T]`. Ограничение обеспечивается системой типов: `FutureVar` является `distinct`-типом, поэтому `mget` недостижим через обычный `Future[T]`.

```nim
var fv = newFutureVar[seq[byte]]("приём")
let buf = addr fv.mget()          # взять указатель до завершения
# ... ОС заполняет buf принятыми байтами ...
fv.mget().setLen(bytesRead)       # обрезать до фактического размера
fv.complete()
```

---

### `finished`

```nim
proc finished*(future: FutureBase | FutureVar): bool
```

Возвращает `true`, если `future` завершён — успешно или с ошибкой.

Для `FutureVar` процедура приводит к `FutureBase` перед проверкой, поскольку `FutureVar` является `distinct`-типом и не наследует методы `FutureBase` напрямую.

**Не делает различия между успехом и ошибкой.** Используйте `failed()` или проверяйте `future.error != nil` для различения.

```nim
let f = newFuture[int]("x")
assert not f.finished
f.complete(0)
assert f.finished   # true независимо от успеха/ошибки
```

---

### `failed`

```nim
proc failed*(future: FutureBase): bool
```

Возвращает `true` тогда и только тогда, когда `future.error != nil`.

**Предостережение:** Возвращает `false` для **незавершённого** Future (потому что `error` равен nil до вызова `fail()`). Всегда сочетайте с проверкой `finished`, когда различие важно:

```nim
if fut.finished and fut.failed:
  обработатьОшибку(fut.error)
elif fut.finished:
  обработатьЗначение(fut.read())
else:
  # ещё ожидается
```

---

### `asyncCheck[T]`

```nim
proc asyncCheck*[T](future: Future[T])
```

Устанавливает коллбэк, который перевыбрасывает ошибку Future, если тот завершился неудачно. Используется как безопасная альтернатива `discard` для Future, чьи значения намеренно не используются.

**Почему не `discard`?**

```nim
discard someAsyncProc()    # ← ошибка молча проглочена навсегда
asyncCheck someAsyncProc() # ← ошибка всплывёт на следующем тике диспетчера
```

**Реализация:** Использует `future.callback = asyncCheckCallback` (форма присваивания, не `addCallback`), которая очищает все предыдущие коллбэки. Замыкание `asyncCheckCallback` захватывает `future` и при вызове выполняет `injectStacktrace(future)` перед перевыбросом. Это означает, что ошибка — вместе с полной асинхронной трассировкой — распространяется до обработчика необработанных исключений диспетчера.

**Не используйте `asyncCheck`, если планируете впоследствии `await`ить этот Future** — `callback=` очищает предыдущие коллбэки, включая установленные механизмом `await`.

```nim
proc фоноваяЗадача(): Future[void] =
  result = newFuture[void]("фоноваяЗадача")
  result.fail(newException(IOError, "диск заполнен"))

asyncCheck фоноваяЗадача()   # IOError не будет молча потеряна
```

---

### `addCallback` (без параметров)

```nim
proc addCallback*(future: FutureBase, cb: proc() {.closure, gcsafe.})
```

Добавляет `cb` в список коллбэков Future. Если Future уже завершён на момент вызова, `cb` немедленно планируется через `callSoon(cb)`, а не добавляется в список.

`assert cb != nil` защищает от nil-коллбэков; передача nil — программная ошибка, которая выбросит исключение во всех режимах сборки.

**Механика списка (`CallbackList.add`):**

- Если `callbacks.function` равен nil (пустой список): присвоить `cb` напрямую — без аллокации.
- Иначе: аллоцировать новый узел `ref CallbackList`, установить его `function`, и обойти до хвоста цепочки для добавления. Это O(n) обход — намеренный: списки коллбэков ожидаются очень короткими (как правило, 1 или 2 записи).

После срабатывания `future.callbacks.call()` (внутри `complete`/`fail`) список очищается (`nil`). Коллбэки, зарегистрированные **после** срабатывания списка, но до сборки объекта Future, будут запланированы немедленно (Future в этот момент уже завершён).

```nim
let f = newFuture[int]("x")
f.addCallback proc() =
  echo "готово (без параметров)"
f.addCallback proc() =
  echo "также готово"
f.complete(1)
# Оба срабатывают в порядке FIFO: сначала "готово (без параметров)", затем "также готово"
```

---

### `addCallback` (типизированный)

```nim
proc addCallback*[T](future: Future[T],
                     cb: proc(future: Future[T]) {.closure, gcsafe.})
```

Удобная перегрузка, оборачивающая `cb` в замыкание без параметров:

```nim
future.addCallback(
  proc() = cb(future)
)
```

Внутреннее замыкание захватывает `future` по ссылке. Поскольку `future` является `ref object`, копирования не происходит. Типизированная перегрузка удобнее формы без параметров, когда коллбэку нужно проверить значение или ошибку Future:

```nim
f.addCallback proc(fut: Future[string]) =
  if fut.failed:
    echo "ошибка: ", fut.error.msg
  else:
    echo "значение: ", fut.read()
```

---

### `callback=` (без параметров)

```nim
proc `callback=`*(future: FutureBase, cb: proc() {.closure, gcsafe.})
```

Заменяет **все** существующие коллбэки одним `cb`. Сначала вызывает `clearCallbacks`, затем `addCallback`. Если Future уже завершён, `cb` немедленно планируется через `callSoon`.

**Когда использовать вместо `addCallback`:** Только когда нужна гарантия, что сработает ровно один коллбэк и вы явно хотите отбросить все ранее зарегистрированные. Используется внутри `asyncCheck` и операторов `and`/`or`.

**Предупреждение:** Использование `callback=` на Future, у которого уже есть коллбэки, зарегистрированные `await`, сломает приостановку `await` — продолжение никогда не будет вызвано. Не смешивайте `callback=` с `await` на одном Future.

---

### `callback=` (типизированный)

```nim
proc `callback=`*[T](future: Future[T],
    cb: proc(future: Future[T]) {.closure, gcsafe.})
```

Типизированная форма `callback=`. Реализована как:

```nim
future.callback = proc() = cb(future)
```

---

### `clearCallbacks`

```nim
proc clearCallbacks*(future: FutureBase)
```

Устанавливает `callbacks.function = nil` и `callbacks.next = nil`, отбрасывая всю цепочку коллбэков. После этого GC может собрать все аллоцированные узлы коллбэков.

Это низкоуровневый «люк аварийного выхода». Очистка коллбэков на Future, которого собираются `await`ить, приведёт к вечному зависанию ожидания. Используйте с осторожностью.

---

### `callSoon`

```nim
proc callSoon*(cbproc: proc() {.gcsafe.})
```

Планирует `cbproc` для выполнения «в ближайшее время» — на следующем тике активного диспетчера событий.

**Поведение:**

- Если `callSoonProc` равен nil (диспетчер не запущен): вызывает `cbproc()` **синхронно и немедленно**. Это позволяет механизму Future (включая `complete`, `fail` и коллбэки) корректно работать до запуска `asyncdispatch.runForever` / `waitFor`.
- Если `callSoonProc` установлен: делегирует ему. В `asyncdispatch` это помещает `cbproc` в очередь готовности диспетчера для выполнения в начале следующей итерации цикла событий.

**Потоко-локальность:** `callSoonProc` является `{.threadvar.}`. Каждый поток ОС имеет собственную независимую копию, что позволяет иметь отдельные диспетчеры на поток.

Перенаправление через `callSoonProc` — это то, что отделяет `asyncfutures` от `asyncdispatch`: модуль futures никогда не импортирует диспетчер; вместо этого диспетчер внедряет себя, вызывая `setCallSoonProc` при запуске.

```nim
callSoon proc() =
  echo "Это выполнится на следующем тике диспетчера"
```

---

### `getCallSoonProc`

```nim
proc getCallSoonProc*(): (proc(cbproc: proc()) {.gcsafe.})
```

Возвращает текущее значение `callSoonProc` для вызывающего потока. Полезно для сохранения и восстановления планировщика (например, в тестовых фреймворках, которые временно подменяют его на синхронный).

---

### `setCallSoonProc`

```nim
proc setCallSoonProc*(p: (proc(cbproc: proc()) {.gcsafe.}))
```

Заменяет реализацию `callSoon` для вызывающего потока. Вызывается `asyncdispatch` при инициализации его цикла событий. Может вызываться тестовым кодом или альтернативными диспетчерами для внедрения собственного планировщика.

**Пример — синхронный тестовый планировщик:**

```nim
import std/asyncfutures
import std/deques

var queue: Deque[proc()]

setCallSoonProc proc(cb: proc()) =
  queue.addLast(cb)

proc runSync() =
  while queue.len > 0:
    queue.popFirst()()

let f = newFuture[int]("тест")
f.addCallback proc() = echo "сработал: ", f.read()
f.complete(42)
runSync()   # выводит "сработал: 42"
```

---

### `and`

```nim
proc `and`*[T, Y](fut1: Future[T], fut2: Future[Y]): Future[void]
```

Возвращает `Future[void]`, который завершается, когда завершатся **оба** — `fut1` и `fut2`.

**Семантика ошибок:** Если любой Future завершается неудачно, возвращаемый Future немедленно завершается с той же ошибкой (побеждает та, которая провалилась первой). Результат другого Future игнорируется.

**Особенности реализации:**

Каждый из `fut1` и `fut2` получает коллбэк через `future.callback = ...` (**не** `addCallback`). Это означает, что все ранее установленные коллбэки на `fut1` или `fut2` **молча отбрасываются**. Для Future, которые собираетесь комбинировать через `and`, не регистрируйте коллбэки до вызова `and`.

Внутри каждого коллбэка проверка `if not retFuture.finished` защищает от двойного завершения. Это обрабатывает случай, когда оба Future завершаются «одновременно» в одном тике диспетчера — только первый сработавший коллбэк завершит `retFuture`.

Возвращаемый Future **не хранит значений**; для доступа к индивидуальным значениям `fut1` и `fut2` читайте их напрямую после завершения комбинированного Future.

```nim
let a = newFuture[int]("a")
let b = newFuture[string]("b")

let оба = a and b
оба.addCallback proc() =
  echo a.read(), " ", b.read()   # "1 привет"

a.complete(1)
b.complete("привет")
```

---

### `or`

```nim
proc `or`*[T, Y](fut1: Future[T], fut2: Future[Y]): Future[void]
```

Возвращает `Future[void]`, который завершается, как только завершится **любой** из `fut1`, `fut2`.

**Семантика ошибок:** Если первый завершившийся Future провалился, возвращаемый Future завершается с этой ошибкой. Результат второго Future полностью игнорируется.

**Реализация:**

Внутреннее обобщённое замыкание `cb[X]` привязывается к обоим Future. Защита `if not retFuture.finished` гарантирует, что победит только первое завершение. Как и `and`, использует `future.callback = ...`, что перезаписывает существующие коллбэки.

**Классический паттерн таймаута:**

```nim
import std/asyncfutures, std/asyncdispatch

proc сТаймаутом[T](fut: Future[T], мс: int): Future[void] =
  result = fut or sleepAsync(мс)
```

```nim
let запрос = httpGetAsync("https://example.com")
let готово = запрос or sleepAsync(5000)
await готово
if запрос.finished:
  echo запрос.read()
else:
  echo "время ожидания истекло"
```

---

### `all[T]`

```nim
proc all*[T](futs: varargs[Future[T]]): auto
```

Возвращает Future, который завершается, когда завершится каждый Future из `futs`.

**Тип возврата зависит от `T`:**

| `T` | Тип возврата | Значение при успехе |
|---|---|---|
| `void` | `Future[void]` | (нет) |
| что-либо ещё | `Future[seq[T]]` | Значения в том же порядке, что и `futs` |

**Пустой ввод:** Возвращает немедленно завершённый Future (`complete()` вызывается до возврата из процедуры).

**Семантика ошибок:** Как только любой Future из `futs` провалится, возвращаемый Future немедленно завершается с этой ошибкой. Уже вычисленные значения в `retValues` отбрасываются.

**Особенности реализации (ветка не-void):**

Проблема захвата индекса: наивное `for i, fut in futs: fut.addCallback proc() = retValues[i] = ...` приведёт к тому, что все коллбэки замкнутся на **одну и ту же** переменную цикла `i` (её конечное значение). Модуль решает это вложенной процедурой `setCallback(i: int)`:

```nim
for i, fut in futs:
  proc setCallback(i: int) =
    fut.addCallback proc(f: Future[T]) =
      retValues[i] = f.read()   # i теперь свежая копия для каждой итерации
      ...
  setCallback(i)
```

Каждый вызов `setCallback` создаёт новый стековый фрейм со своим `i`, так что каждый коллбэк замыкается на разное значение.

Счётчик `completedFutures` увеличивается внутри каждого коллбэка. Когда он достигает `len(retValues)`, вызывается `retFuture.complete(retValues)`.

```nim
let f1 = newFuture[int]("f1")
let f2 = newFuture[int]("f2")
let f3 = newFuture[int]("f3")

let агрегат = all(f1, f2, f3)   # Future[seq[int]]
агрегат.addCallback proc() =
  echo агрегат.read()   # @[10, 20, 30]

f3.complete(30)
f1.complete(10)
f2.complete(20)          # последний запускает завершение
```

---

### `getFuturesInProgress` *(только при futureLogging)*

```nim
when isFutureLoggingEnabled:
  proc getFuturesInProgress*(): var Table[FutureInfo, int]
```

Возвращает изменяемую ссылку на потоко-локальную таблицу `futuresInProgress`. Каждая запись отображает `FutureInfo` (трассировка при создании + `fromProc`) на количество в данный момент активных Future с такой сигнатурой.

Увеличивается `logFutureStart` (вызываемым из `newFuture`/`newFutureVar`) и уменьшается `logFutureFinish` (вызываемым из `complete`/`fail`). Счётчик, который неограниченно растёт, указывает на Future, который никогда не завершается — утечку Future.

```nim
# Скомпилировать с: nim c -d:futureLogging myapp.nim
when isFutureLoggingEnabled:
  import std/tables
  for info, count in getFuturesInProgress():
    if count > 0:
      echo "[УТЕЧКА] ", info.fromProc, " имеет ", count, " незавершённых Future"
      for entry in info.stackTrace:
        echo "  ", entry.filename, "(", entry.line, ") ", entry.procname
```

---

## Диагностика и отладка

### Флаги компиляции

| Флаг | Эффект |
|---|---|
| *(нет / debug)* | `checkFinished` активен; поля `stackTrace`, `id`, `fromProc` присутствуют; полные асинхронные трассировки добавляются. |
| `-d:release` | `checkFinished` удалён; debug-поля отсутствуют; `injectStacktrace` — пустышка; максимальная производительность. |
| `-d:futureLogging` | Активирует отслеживание через таблицу `FutureInfo`. Можно комбинировать с `-d:release`. |
| `-d:nimStackTraceOverride` | Включает разрешение внешних отладочных символов в трассировках, форматируемых `$`. |
| `-d:nimPreviewSlimSystem` | Требует явного импорта `std/objectdollar` (для `StackTraceEntry`) и `std/assertions`. |

### Внедрение асинхронной трассировки

Когда ошибка провалившегося Future перевыбрасывается через `read()`, модуль обогащает сообщение исключения секцией **асинхронной трассировки** в не-release сборках:

```
IOError: соединение отклонено
Async traceback:
  mymodule.nim(42) получитьДанные
  mymodule.nim(78) обработатьЗапрос
Exception message: соединение отклонено
```

Процедура `injectStacktrace`:

1. Проверяет, присутствует ли уже заголовок `"\nAsync traceback:\n"` в сообщении (идемпотентность — повторный вызов `read()` на одном провалившемся Future не дублирует секцию).
2. Обрезает сообщение до оригинального (предшествующего внедрению) текста.
3. Вызывает `$` на `getStackTraceEntries(future.error)` для форматирования трассировки, что:
   - Применяет `addDebuggingInfo`, если установлен `-d:nimStackTraceOverride`.
   - Фильтрует дублирующиеся записи через `seenEntries` (`HashSet`).
   - Фильтрует внутренние фреймы Nim async через `isInternal` (записи из `asyncdispatch.nim`, `asyncfutures.nim`, `threadimpl.nim`).
   - Пропускает записи с отрицательными номерами строк (маркеры перевыброса).
4. Добавляет форматированную трассировку и подвал.

### Форматирование трассировки стека

Оператор `$` над `seq[StackTraceEntry]` экспортируется и форматирует трассировку как многострочную строку. Каждая строка имеет вид:

```
  filename.nim(номерСтроки) имяПроцедуры
```

Внутренние фреймы async автоматически исключаются, так что трассировка показывает только точки вызова на уровне приложения.

---

## Потокобезопасность

`asyncfutures` спроектирован для модели **однопоточного цикла событий**. Ключевые аспекты потокобезопасности:

- **`callSoonProc` является `{.threadvar.}`** — каждый поток ОС имеет независимый планировщик. Два потока могут каждый иметь свой диспетчер, не мешая друг другу.
- **`currentID` — обычная переменная уровня модуля** (не threadvar, присутствует только в не-release сборках). В многопоточной программе с конкурентным созданием Future в не-release сборках этот счётчик не является потокобезопасным. Это приемлемо, поскольку он предназначен только для отладки.
- **`futuresInProgress` является `{.threadvar.}`** — таблица обнаружения утечек является потоко-локальной.
- **Доступ к одному и тому же `Future[T]` из нескольких потоков одновременно не является безопасным.** В `FutureBase` нет блокировок или атомарных операций. Прагмы `{.gcsafe.}` на коллбэках утверждают безопасность GC, а не свободу от гонок данных. Храните Future в том потоке, в котором они были созданы.

---

## Полный пример

```nim
import std/asyncfutures

# ── 1. Базовый жизненный цикл Future ────────────────────────────────────────
block базовыйЦикл:
  let f = newFuture[int]("базовыйЦикл")
  assert not f.finished

  f.addCallback proc(fut: Future[int]) =
    echo "значение = ", fut.read()   # значение = 99

  f.complete(99)
  assert f.finished and not f.failed

# ── 2. Провал и инспекция ошибки ────────────────────────────────────────────
block путьОшибки:
  let f = newFuture[string]("путьОшибки")
  f.fail(newException(IOError, "диск заполнен"))

  assert f.failed
  assert f.error of IOError
  assert f.readError().msg == "диск заполнен"

  try:
    discard f.read()
  except IOError as e:
    echo "поймано: ", e.msg    # поймано: диск заполнен

# ── 3. FutureVar — переиспользуемый Future ──────────────────────────────────
block паттернПовторногоИспользования:
  var fv = newFutureVar[seq[byte]]("паттернПовторногоИспользования")
  for chunk in 1..3:
    fv.clean()
    fv.mget() = @[byte(chunk * 10)]
    fv.complete()
    echo fv.read()   # @[10], @[20], @[30]

# ── 4. Параллельная композиция: and ─────────────────────────────────────────
block иКомпозиция:
  let a = newFuture[int]("a")
  let b = newFuture[float]("b")
  let оба = a and b
  оба.addCallback proc() =
    echo a.read(), " ", b.read()   # 7 3.14
  a.complete(7)
  b.complete(3.14)

# ── 5. Гонка: or ────────────────────────────────────────────────────────────
block гонка:
  let быстрый = newFuture[void]("быстрый")
  let медленный = newFuture[void]("медленный")
  let гонка = быстрый or медленный
  гонка.addCallback proc() = echo "победитель!"
  быстрый.complete()   # "победитель!" срабатывает; медленный игнорируется

# ── 6. Агрегация: all ───────────────────────────────────────────────────────
block агрегация:
  let задачи = [newFuture[int]("з0"),
                newFuture[int]("з1"),
                newFuture[int]("з2")]
  let агрегат = all(задачи)
  агрегат.addCallback proc() =
    echo агрегат.read()   # @[100, 200, 300]
  задачи[2].complete(300)
  задачи[0].complete(100)
  задачи[1].complete(200)

# ── 7. asyncCheck — выброс ошибок из отброшенных Future ─────────────────────
block проверкаОшибок:
  proc рискованнаяОперация(): Future[void] =
    result = newFuture[void]("рискованнаяОперация")
    result.fail(newException(ValueError, "неверный ввод"))

  asyncCheck рискованнаяОперация()
  # Без запущенного диспетчера коллбэк срабатывает немедленно.
  # В реальной async-программе ValueError распространился бы до диспетчера.

# ── 8. Пользовательский callSoon для тестирования ───────────────────────────
block пользовательскийПланировщик:
  import std/deques
  var очередь: Deque[proc()]
  setCallSoonProc proc(cb: proc()) = очередь.addLast(cb)

  let f = newFuture[int]("пользовательский")
  f.addCallback proc() = echo "запланировано: ", f.read()
  f.complete(42)          # коллбэк поставлен в очередь, но ещё не вызван

  while очередь.len > 0:
    очередь.popFirst()()  # выводит "запланировано: 42"

  # Восстановить дефолтный (nil → немедленное выполнение)
  setCallSoonProc nil
```

---

*Основано на исходном коде `asyncfutures.nim` из стандартной библиотеки Nim
