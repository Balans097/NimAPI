# Справочник модуля `std/asyncfutures` для Nim

> Фундаментальный строительный блок async-системы Nim. Модуль определяет, чем является `Future`, и все операции над ним: создание, завершение, провал, проверку состояния, композицию и реакцию на события. Он автоматически доступен при импорте `std/asyncdispatch`, поэтому напрямую его импортируют редко.

---

## Содержание

1. [Ключевая концепция: что такое Future?](#ключевая-концепция-что-такое-future)
2. [Типы данных](#типы-данных)
   - [FutureBase](#futurebase)
   - [Future\[T\]](#futuret)
   - [FutureVar\[T\]](#futurevart)
   - [FutureError](#futureerror)
3. [Создание futures](#создание-futures)
   - [newFuture](#newfuture)
   - [newFutureVar](#newfuturevar)
4. [Завершение и провал futures](#завершение-и-провал-futures)
   - [complete (Future\[T\])](#complete-futuret)
   - [complete (Future\[void\])](#complete-futurevoid)
   - [complete (FutureVar\[T\])](#complete-futurevart)
   - [fail](#fail)
5. [Чтение результатов](#чтение-результатов)
   - [read](#read)
   - [readError](#readerror)
   - [mget](#mget)
6. [Проверка состояния](#проверка-состояния)
   - [finished](#finished)
   - [failed](#failed)
7. [Колбэки](#колбэки)
   - [addCallback (без типа)](#addcallback-без-типа)
   - [addCallback (типизированный)](#addcallback-типизированный)
   - [callback= (сеттер)](#callback-сеттер)
   - [clearCallbacks](#clearcallbacks)
8. [Композиция futures](#композиция-futures)
   - [Оператор and](#оператор-and)
   - [Оператор or](#оператор-or)
   - [all](#all)
9. [Запуск без ожидания](#запуск-без-ожидания)
   - [asyncCheck](#asynccheck)
10. [Утилиты FutureVar](#утилиты-futurevar)
    - [clean](#clean)
11. [Инфраструктура callSoon](#инфраструктура-callsoon)
    - [callSoon](#callsoon)
    - [getCallSoonProc](#getcallsoonproc)
    - [setCallSoonProc](#setcallsoonproc)
12. [Логирование futures](#логирование-futures)
    - [isFutureLoggingEnabled](#isfutureloggingenabled)
    - [getFuturesInProgress](#getfuturesinprogress)
13. [Полный практический пример](#полный-практический-пример)
14. [Шпаргалка](#шпаргалка)

---

## Ключевая концепция: что такое Future?

`Future[T]` — это контейнер-обещание для значения типа `T`, которого ещё не существует. Он представляет асинхронную операцию, выполняющуюся прямо сейчас, которая в какой-то момент либо даст результат, либо завершится ошибкой.

Удобная аналогия — талон в ресторане: вы делаете заказ (запускаете async-операцию), получаете пронумерованный талон (`Future`), и можете вернуться за едой позже (за результатом). Пока вы свободны делать что угодно другое.

```
Машина состояний Future:
────────────────────────
           ┌──────────┐
           │ ожидание │  ← только создан, ещё не завершён
           └────┬─────┘
                │
       ┌────────┴────────┐
       ▼                 ▼
  ┌─────────┐       ┌──────────┐
  │завершён │       │  ошибка  │  ← оба варианта — «finished»
  └─────────┘       └──────────┘
```

`Future` считается **finished** (завершённым), когда он либо успешно завершился (содержит значение), либо закончился ошибкой. Проверяется это через `finished` и `failed`. Переводится в завершённое состояние через `complete` или `fail`. Результат извлекается через `read` (которое переброшет исключение, если Future завершился с ошибкой).

---

## Типы данных

### `FutureBase`

```nim
type FutureBase* = ref object of RootObj
  error*:           ref Exception  ## Сохранённая ошибка, если есть
  errorStackTrace*: string         ## Стек вызовов в момент ошибки
  # (колбэки, флаг finished и отладочные поля — внутренние)
```

**Что это:** Нетипизированный базовый класс, общий для всех futures вне зависимости от типа результата. Содержит ошибку, флаг завершённости и список колбэков. В прикладном коде с `FutureBase` работают редко — используйте типизированный `Future[T]`. `FutureBase` нужен в API, которому нужно принимать или возвращать futures произвольных типов (например, обобщённая утилита таймаута).

```nim
proc onDone(f: FutureBase) =
  if f.failed:
    echo "Ошибка: ", f.error.msg
  else:
    echo "Успешно (тип значения на этом уровне неизвестен)"
```

---

### `Future[T]`

```nim
type Future*[T] = ref object of FutureBase
  # value: T  (внутреннее поле)
```

**Что это:** Типизированная «рабочая лошадка» async-системы. `T` — тип значения, которое future будет содержать после завершения. `Future[void]` используется для операций, которые сигнализируют о завершении без возврата значения (аналог `proc` без возвращаемого типа).

Это тип, с которым сталкиваются чаще всего — он возвращается каждой `async`-процедурой и большинством I/O-операций в `asyncdispatch`, `asyncfile`, `asyncnet` и т.д.

```nim
import std/asyncdispatch

proc fetchNumber(): Future[int] {.async.} =
  await sleepAsync(100)
  return 42

let f: Future[int] = fetchNumber()
```

---

### `FutureVar[T]`

```nim
type FutureVar*[T] = distinct Future[T]
```

**Что это:** Обёртка-`distinct` над `Future[T]`, предназначенная для **повторного использования**. Обычный `Future[T]` можно завершить только один раз — попытка завершить его повторно является ошибкой программы. `FutureVar[T]` позволяет вызвать `clean` и переиспользовать тот же объект для следующей операции, избегая повторного выделения памяти в горячих циклах. Это низкоуровневая оптимизация, редко нужная в повседневном коде.

Ключевое отличие: у `FutureVar` есть процедура `mget`, позволяющая получить доступ к хранимому значению даже до завершения future.

---

### `FutureError`

```nim
type FutureError* = object of Defect
  cause*: FutureBase  ## Future, вызвавший эту ошибку
```

**Что это:** `Defect`, возбуждаемый при попытке завершить future, который уже был завершён. Это указывает на логическую ошибку в программе — future был разрешён дважды. Поле `cause` указывает на виновный future, чтобы его можно было изучить в отладчике или обработчике ошибок.

Проверка срабатывает только в не-release-сборках; в `--define:release` она пропускается ради производительности.

---

## Создание futures

### `newFuture`

```nim
proc newFuture*[T](fromProc: string = "unspecified"): owned(Future[T])
```

**Что делает:** Создаёт и возвращает новый незавершённый `Future[T]`. Строка `fromProc` — это человекочитаемая метка (по соглашению — имя процедуры-владельца future), которая появляется в сообщениях об ошибках и в отладочном выводе, существенно упрощая поиск того, откуда взялся зависший или упавший future.

**Когда использовать:** Всякий раз, когда реализуете низкоуровневую async-операцию вручную — то есть когда нельзя просто выполнить `await` чего-либо и нужно самому создать future, зарегистрировать колбэк в ОС и разрешить future когда он сработает.

```nim
import std/asyncdispatch

proc delay(ms: int): Future[void] =
  var fut = newFuture[void]("delay")
  addTimer(ms, oneshot = true, cb = proc(fd: AsyncFD): bool =
    fut.complete()
    return true
  )
  return fut

proc main() {.async.} =
  echo "ожидаем..."
  await delay(500)
  echo "готово через 500мс"

waitFor main()
```

---

### `newFutureVar`

```nim
proc newFutureVar*[T](fromProc = "unspecified"): owned(FutureVar[T])
```

**Что делает:** Создаёт переиспользуемый `FutureVar[T]`. Внутри один раз выделяет `Future[T]` и оборачивает его. После завершения `FutureVar` можно вызвать `clean`, сбросить состояние и использовать тот же объект для следующей операции.

**Когда использовать:** В производительно-критичных async-циклах, где однотипная операция выполняется многократно и нужно избежать выделения памяти на каждой итерации. Для большинства прикладного кода обычный `newFuture` понятнее и проще.

```nim
import std/asyncdispatch

var fv = newFutureVar[string]("reusable-reader")

for i in 0..<1000:
  fv.clean()              # сбрасываем состояние — готов к следующему использованию
  fv.complete("chunk-" & $i)
  echo fv.read()          # "chunk-0", "chunk-1", ...
```

---

## Завершение и провал futures

Эти операции переводят future из состояния **ожидание** в **завершён**. Они также запускают все зарегистрированные колбэки.

### `complete` (Future[T])

```nim
proc complete*[T](future: Future[T], val: sink T)
```

**Что делает:** Помечает `future` как успешно завершённый и сохраняет `val` в качестве результата. Все колбэки, зарегистрированные через `addCallback`, планируются к вызову через `callSoon`. Попытка завершить уже завершённый future вызывает `FutureError` (в не-release-сборках).

**Когда использовать:** В той точке кода, где async-операция завершается успешно и готова вернуть результат.

```nim
import std/asyncdispatch

proc resolveWithAnswer(): Future[int] =
  result = newFuture[int]("resolveWithAnswer")
  result.complete(42)  # немедленно разрешаем
```

---

### `complete` (Future[void])

```nim
proc complete*(future: Future[void])
```

**Что делает:** Помечает `Future[void]` как успешно завершённый. Значение не сохраняется — просто сигнализирует, что операция завершилась без ошибки. Правила планирования колбэков те же.

**Когда использовать:** Когда async-операция не возвращает значимого результата — например, операция записи, которой нужно лишь сообщить о своём завершении.

```nim
import std/asyncdispatch

proc signalDone(): Future[void] =
  result = newFuture[void]("signalDone")
  result.complete()
```

---

### `complete` (FutureVar[T])

```nim
proc complete*[T](future: FutureVar[T])
proc complete*[T](future: FutureVar[T], val: sink T)
```

**Что делает:** Две перегрузки для завершения `FutureVar`. Первая (без значения) помечает его завершённым, полагаясь на то, что значение уже было записано через `mget`. Вторая сохраняет новое значение (перезаписывая предыдущее) и помечает завершённым. Обе запускают колбэки.

---

### `fail`

```nim
proc fail*[T](future: Future[T], error: ref Exception)
```

**Что делает:** Помечает `future` как завершённый с ошибкой. Исключение `error` сохраняется внутри future и будет переброшено, когда кто-нибудь вызовет `read` на этом future (или когда колбэки проверят `future.failed`). Стек вызовов в момент ошибки также захватывается и сохраняется.

**Когда использовать:** Когда async-операция столкнулась с невосстановимой ошибкой и не может предоставить результат.

```nim
import std/asyncdispatch

proc mightFail(value: int): Future[int] =
  result = newFuture[int]("mightFail")
  if value < 0:
    result.fail(newException(ValueError, "Значение должно быть неотрицательным"))
  else:
    result.complete(value * 2)

proc main() {.async.} =
  try:
    let x = await mightFail(-1)
  except ValueError as e:
    echo "Поймали: ", e.msg

waitFor main()
```

---

## Чтение результатов

### `read`

```nim
proc read*[T](future: Future[T] | FutureVar[T]): lent T
proc read*(future: Future[void] | FutureVar[void])
```

**Что делает:** Извлекает значение из завершённого future. Если future завершился с ошибкой, `read` пробрасывает сохранённое исключение (в не-release-сборках с добавленным async-трейсбэком). Если future ещё не завершён, `read` возбуждает `ValueError` — он **не** блокирует и не ожидает.

**Когда использовать:** После того, как вы убедились (через `finished`), что future завершён, или внутри колбэка, где известно, что future готов. В `async`-процедурах обычно используется `await`, который вызывает `read` автоматически.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[string]("example")
  f.complete("привет")

  # Безопасно вызывать read(), потому что мы знаем, что он завершён
  echo f.read()   # "привет"

waitFor main()
```

---

### `readError`

```nim
proc readError*[T](future: Future[T]): ref Exception
```

**Что делает:** Извлекает сохранённое исключение из завершившегося с ошибкой future. Если у future нет ошибки (он либо успешно завершился, либо ещё в ожидании), возбуждает `ValueError`. В отличие от `read`, это не перебрасывает ошибку — оно возвращает её как значение для последующего изучения.

**Когда использовать:** Когда вы явно проверили `future.failed` и хотите изучить или залогировать исключение, не перебрасывая его.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[void]("example")
  f.fail(newException(IOError, "диск заполнен"))

  if f.failed:
    let err = f.readError()
    echo "Тип ошибки:    ", err.name
    echo "Сообщение: ", err.msg

waitFor main()
```

---

### `mget`

```nim
proc mget*[T](future: FutureVar[T]): var T
```

**Что делает:** Возвращает **изменяемую** ссылку на значение, хранящееся внутри `FutureVar`. В отличие от `read`, не проверяет завершённость future — даёт прямой доступ на запись ко внутреннему значению даже пока future ещё в ожидании. Это механизм для инкрементального заполнения `FutureVar` до его завершения.

**Когда использовать:** Только с `FutureVar`, в низкоуровневом коде, где результат накапливается по месту (например, паттерн async-чтения в буфер).

```nim
import std/asyncdispatch

var fv = newFutureVar[seq[int]]("accumulate")
mget(fv).add(1)
mget(fv).add(2)
mget(fv).add(3)
fv.complete()

echo fv.read()   # @[1, 2, 3]
```

---

## Проверка состояния

### `finished`

```nim
proc finished*(future: FutureBase | FutureVar): bool
```

**Что делает:** Возвращает `true`, если future завершён (либо успешно, либо с ошибкой). Возвращает `false`, если операция ещё выполняется. Важно: `true` не означает успех — нужно дополнительно проверить `failed`, чтобы различить два исхода.

**Когда использовать:** Перед вызовом `read` за пределами `async`-процедуры, или для неблокирующей проверки состояния future.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[int]("example")
  echo f.finished   # false — ещё в ожидании

  f.complete(7)
  echo f.finished   # true — готов
  echo f.failed     # false — завершился успешно

waitFor main()
```

---

### `failed`

```nim
proc failed*(future: FutureBase): bool
```

**Что делает:** Возвращает `true`, если future завершился с ошибкой (т.е. на нём был вызван `fail`). Возвращает `false`, если future ещё в ожидании или завершился успешно.

**Важно:** `failed == false` не означает, что future завершён — он может быть ещё в ожидании. Всегда проверяйте `finished` сначала, если нужно знать, завершён ли он вообще.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[void]("example")
  f.fail(newException(ValueError, "упс"))

  echo f.finished   # true
  echo f.failed     # true — завершился с ошибкой

  let g = newFuture[void]("example2")
  g.complete()

  echo g.finished   # true
  echo g.failed     # false — завершился успешно

waitFor main()
```

---

## Колбэки

Колбэки — это процедуры, зарегистрированные на future и планируемые к вызову при его завершении. Это низкоуровневый механизм, лежащий в основе `await`. Большинство прикладного кода должно использовать `await` напрямую; колбэки нужны для построения комбинаторов или интеграции с не-async-кодом.

### `addCallback` (без типа)

```nim
proc addCallback*(future: FutureBase, cb: proc() {.closure, gcsafe.})
```

**Что делает:** Регистрирует `cb` для вызова при завершении `future`. Можно добавлять несколько колбэков — они хранятся во внутреннем связном списке и все вызываются (через `callSoon`) при завершении future. Если future **уже завершён** на момент вызова `addCallback`, `cb` немедленно планируется через `callSoon`. Колбэк не получает аргументов.

**Когда использовать:** Когда нужно среагировать на завершение future, не имея доступа к его типовому параметру, или при построении обобщённых async-комбинаторов.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[int]("example")

  f.addCallback proc() =
    echo "Future завершён! С ошибкой: ", f.failed

  f.complete(10)  # колбэк планируется здесь

waitFor main()
```

---

### `addCallback` (типизированный)

```nim
proc addCallback*[T](future: Future[T], cb: proc(future: Future[T]) {.closure, gcsafe.})
```

**Что делает:** То же, что нетипизированная перегрузка, но колбэк получает завершённый `Future[T]` в качестве аргумента, что делает вызов `read` или проверку результата типобезопасной.

**Когда использовать:** Предпочтительнее нетипизированной перегрузки, когда тип future известен и колбэку нужен доступ к результату.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[string]("greet")

  f.addCallback proc(fut: Future[string]) =
    if not fut.failed:
      echo "Результат: ", fut.read()   # безопасно — fut завершён

  f.complete("Привет!")

waitFor main()
```

---

### `callback=` (сеттер)

```nim
proc `callback=`*(future: FutureBase, cb: proc() {.closure, gcsafe.})
proc `callback=`*[T](future: Future[T], cb: proc(future: Future[T]) {.closure, gcsafe.})
```

**Что делает:** **Заменяет** все существующие колбэки одним новым. Внутри вызывает `clearCallbacks`, затем `addCallback`. Если future уже завершён, новый колбэк немедленно планируется через `callSoon`.

**Когда использовать:** Когда нужен ровно один колбэк, заменяющий все ранее зарегистрированные. Документация модуля рекомендует предпочитать `addCallback` или `then` этому сеттеру, так как молчаливое отбрасывание существующих колбэков может порождать труднозаметные баги.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[void]("example")
  f.addCallback proc() = echo "первый"

  # Заменяет колбэк "первый" — он никогда не выполнится
  f.callback = proc() = echo "второй"

  f.complete()   # печатается только "второй"

waitFor main()
```

---

### `clearCallbacks`

```nim
proc clearCallbacks*(future: FutureBase)
```

**Что делает:** Удаляет все колбэки, зарегистрированные на `future`. После этого завершение или провал future не вызовет ни одного колбэка. Используется внутренне сеттером `callback=` для реализации семантики замены.

**Когда использовать:** Редко в прикладном коде. Полезно, когда нужно «отсоединить» future от его нижележащих потребителей — например, при реализации механизма отмены.

---

## Композиция futures

Эти комбинаторы позволяют декларативно выражать зависимости между несколькими futures без ручной разводки колбэков.

### Оператор `and`

```nim
proc `and`*[T, Y](fut1: Future[T], fut2: Future[Y]): Future[void]
```

**Что делает:** Возвращает новый `Future[void]`, который завершается только тогда, когда завершились **оба** — `fut1` и `fut2`. Если любой из них завершится с ошибкой, возвращаемый future немедленно проваливается с той же ошибкой. Значения `fut1` и `fut2` отбрасываются — используйте `all`, если они нужны.

**Когда использовать:** Когда две независимые операции должны обе завершиться перед продолжением, и их возвращаемые значения не нужны.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let a = sleepAsync(200)
  let b = sleepAsync(300)

  await (a and b)   # ждёт ~300мс (дольшую из двух)
  echo "Обе завершены"

waitFor main()
```

---

### Оператор `or`

```nim
proc `or`*[T, Y](fut1: Future[T], fut2: Future[Y]): Future[void]
```

**Что делает:** Возвращает новый `Future[void]`, который завершается как только завершится **любой** из двух future — whichever происходит первым. Если первый завершившийся содержит ошибку, возвращаемый future проваливается с этой ошибкой. Значение «победителя» отбрасывается — используйте этот оператор главным образом для паттернов гонки или таймаута.

**Когда использовать:** Для реализации таймаутов (`операция or futureТаймаута`) или для получения первого доступного результата из двух конкурирующих источников.

```nim
import std/asyncdispatch

proc withTimeout[T](fut: Future[T], ms: int): Future[void] =
  return fut or sleepAsync(ms)

proc main() {.async.} =
  let op = sleepAsync(1000)
  await withTimeout(op, 200)
  if op.finished:
    echo "завершилось вовремя"
  else:
    echo "истекло время ожидания"

waitFor main()
```

---

### `all`

```nim
proc all*[T](futs: varargs[Future[T]]): auto
```

**Что делает:** Возвращает future, который завершается когда завершились **все** futures из `futs`. Тип возвращаемого значения зависит от `T`:

- Если `T` — `void` → возвращает `Future[void]`
- Если `T` — любой другой тип → возвращает `Future[seq[T]]` с результатами в том же порядке, что входные futures

Если любой из futures проваливается, возвращаемый future немедленно проваливается с той же ошибкой (уже завершённые результаты других futures отбрасываются). Если `futs` пуст, возвращаемый future завершается немедленно.

**Когда использовать:** Когда есть коллекция параллельных однотипных операций и нужны все их результаты.

```nim
import std/asyncdispatch

proc fetchNumber(n: int): Future[int] {.async.} =
  await sleepAsync(n * 10)
  return n * n

proc main() {.async.} =
  let results = await all(fetchNumber(1), fetchNumber(2), fetchNumber(3))
  echo results   # @[1, 4, 9]

waitFor main()
```

---

## Запуск без ожидания

### `asyncCheck`

```nim
proc asyncCheck*[T](future: Future[T])
```

**Что делает:** Присоединяет к `future` колбэк, который **пробросит** ошибку future как необработанное исключение, если тот завершится с ошибкой. Это делает паттерн «запустил и забыл» безопасным: вместо молчаливого проглатывания ошибок (что делает `discard`) `asyncCheck` гарантирует, что ошибки всплывут и завалят программу, делая баги видимыми.

**Никогда не используйте `discard` на `Future`.** Ошибки отброшенного future исчезают бесследно. Для futures, которые не `await`-ируются, используйте `asyncCheck`.

**Когда использовать:** Когда запускаете async-операцию в фоне, не нуждаетесь в её возвращаемом значении и не хотите её `await`-ировать.

```nim
import std/asyncdispatch

proc backgroundTask() {.async.} =
  await sleepAsync(100)
  echo "фоновая задача завершена"
  # если здесь возникнет исключение, asyncCheck его пробросит

proc main() {.async.} =
  asyncCheck backgroundTask()    # запущена в фоне — без await
  echo "main продолжается немедленно"
  await sleepAsync(200)          # даём фоновой задаче время завершиться

waitFor main()
```

---

## Утилиты FutureVar

### `clean`

```nim
proc clean*[T](future: FutureVar[T])
```

**Что делает:** Сбрасывает флаг `finished` и очищает сохранённую ошибку в `future`, возвращая его в состояние «ожидание». Хранимое значение (если есть) **не** очищается — оно остаётся как есть до перезаписи. Это позволяет переиспользовать тот же объект `FutureVar` для новой операции без нового выделения памяти.

**Когда использовать:** Всегда вызывайте `clean` перед повторным использованием `FutureVar`. Иначе `checkFinished` бросит `FutureError`, потому что future всё ещё находится в состоянии «завершён» от предыдущей операции.

```nim
import std/asyncdispatch

var fv = newFutureVar[int]("counter")

for i in 1..5:
  fv.clean()          # сбрасываем для следующего использования
  fv.complete(i * 10)
  echo fv.read()      # 10, 20, 30, 40, 50
```

---

## Инфраструктура callSoon

`callSoon` — это мост между `asyncfutures` и циклом событий. Когда future завершается, его колбэки не вызываются синхронно — они *планируются* через `callSoon` для вызова на следующем тике цикла событий. Это предотвращает глубокие стеки вызовов и обеспечивает предсказуемый порядок выполнения.

### `callSoon`

```nim
proc callSoon*(cbproc: proc() {.gcsafe.})
```

**Что делает:** Планирует `cbproc` к вызову «скоро» — либо на следующем тике диспетчера, если цикл событий запущен, либо немедленно, если диспетчер ещё не инициализирован. Эта гарантия отложенного выполнения предотвращает переполнение стека при цепочках async-колбэков.

**Когда использовать:** Редко в прикладном коде. Полезно, когда нужно отложить выполнение некоторой логики до следующего прохода цикла событий, не привязывая её к конкретному future.

---

### `getCallSoonProc`

```nim
proc getCallSoonProc*(): (proc(cbproc: proc()) {.gcsafe.})
```

**Что делает:** Возвращает текущую реализацию `callSoon` — процедуру, которую внедрил диспетчер событий. Полезна для отладки или для кастомных async-бэкендов, которым нужно инспектировать планировщик.

---

### `setCallSoonProc`

```nim
proc setCallSoonProc*(p: (proc(cbproc: proc()) {.gcsafe.}))
```

**Что делает:** Заменяет реализацию `callSoon`. Вызывается внутри `asyncdispatch` при инициализации его цикла событий, соединяя механизм `callSoon` с очередью диспетчера. Прикладной код не должен вызывать это, если только не реализует кастомный async-бэкенд.

---

## Логирование futures

Логирование futures — это отладочный инструмент, отслеживающий, какие futures находятся в процессе выполнения. Включается компиляцией с флагом `-d:futureLogging`.

### `isFutureLoggingEnabled`

```nim
const isFutureLoggingEnabled* = defined(futureLogging)
```

**Что это:** Константа времени компиляции, равная `true`, когда логирование futures включено. Используйте её для условной компиляции кода, зависящего от логирования.

---

### `getFuturesInProgress`

```nim
proc getFuturesInProgress*(): var Table[FutureInfo, int]
```

**Что делает:** Возвращает изменяемую ссылку на глобальную таблицу, отслеживающую, сколько экземпляров каждого future (с ключом по стеку вызовов и имени процедуры-источника) находятся в ожидании прямо сейчас. Доступно только при компиляции с `-d:futureLogging`.

**Когда использовать:** Для диагностики зависаний или утечки памяти путём инспекции застрявших futures. Таблица отображает `FutureInfo` (стек вызовов + имя процедуры) на количество незавершённых экземпляров.

```nim
when defined(futureLogging):
  import std/asyncfutures
  let inProgress = getFuturesInProgress()
  for info, count in inProgress:
    if count > 0:
      echo info.fromProc, ": ", count, " в ожидании"
```

---

## Полный практический пример

Самодостаточный пример, демонстрирующий создание, композицию и обработку ошибок futures на нескольких уровнях:

```nim
import std/asyncdispatch

# Низкий уровень: вручную создаём и разрешаем future
proc resolveAfterDelay(value: int, ms: int): Future[int] =
  var fut = newFuture[int]("resolveAfterDelay")
  addTimer(ms, oneshot = true, cb = proc(fd: AsyncFD): bool =
    fut.complete(value)
    return true
  )
  return fut

# Средний уровень: комбинируем futures
proc sumTwoAsync(a, b: int): Future[int] {.async.} =
  # Запускаем оба параллельно через `all`
  let results = await all(
    resolveAfterDelay(a, 100),
    resolveAfterDelay(b, 150)
  )
  return results[0] + results[1]

# Путь ошибки
proc mayFail(shouldFail: bool): Future[string] =
  result = newFuture[string]("mayFail")
  if shouldFail:
    result.fail(newException(ValueError, "намеренный сбой"))
  else:
    result.complete("успех!")

# Верхний уровень
proc main() {.async.} =
  # Параллельное вычисление
  let sum = await sumTwoAsync(10, 32)
  echo "10 + 32 = ", sum   # 42

  # Обработка ошибок
  let f = mayFail(true)
  echo "завершился с ошибкой? ", f.failed   # true
  let err = f.readError()
  echo "ошибка: ", err.msg     # "намеренный сбой"

  # Гонка через or (таймаут)
  let winner = resolveAfterDelay(1, 50) or resolveAfterDelay(2, 200)
  await winner
  echo "гонка завершена"

  # Фоновая задача без ожидания
  asyncCheck (proc() {.async.} =
    await sleepAsync(10)
    echo "фон завершён"
  )()
  await sleepAsync(50)   # даём фоновой задаче поработать

waitFor main()
```

---

## Шпаргалка

| Задача | API |
|---|---|
| Создать новый future | `newFuture[T]("имяПроцедуры")` |
| Создать переиспользуемый future | `newFutureVar[T]("имяПроцедуры")` |
| Завершить успешно (со значением) | `future.complete(value)` |
| Завершить успешно (void) | `future.complete()` |
| Завершить с ошибкой | `future.fail(exception)` |
| Прочитать результат | `future.read()` |
| Прочитать ошибку | `future.readError()` |
| Изменяемый доступ (только FutureVar) | `mget(futureVar)` |
| Проверить, завершён ли (успех или ошибка) | `future.finished` |
| Проверить, завершился ли с ошибкой | `future.failed` |
| Реагировать на завершение (типизированно) | `future.addCallback proc(f: Future[T]) = ...` |
| Реагировать на завершение (без типа) | `future.addCallback proc() = ...` |
| Заменить все колбэки | `future.callback = proc() = ...` |
| Удалить все колбэки | `future.clearCallbacks()` |
| Ждать оба | `fut1 and fut2` |
| Ждать любой из двух | `fut1 or fut2` |
| Ждать все (с результатами) | `await all(fut1, fut2, fut3)` |
| Запустить без ожидания (безопасно) | `asyncCheck future` |
| Сбросить FutureVar для повторного использования | `futureVar.clean()` |
| Отложить выполнение до следующего тика | `callSoon(proc() = ...)` |
