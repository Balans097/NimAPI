# Справочник модуля `asyncstreams`

> **Стандартная библиотека Nim — асинхронные потоки данных**
> Статус API: **нестабильный** — может изменяться.

---

## Общее описание

`asyncstreams` вводит единственную структуру данных — `FutureStream[T]` — которая соединяет **производителей**, генерирующих данные постепенно, с **потребителями**, читающими их асинхронно.

Думайте о `FutureStream` как об **типизированной асинхронной очереди**: производитель записывает значения по одному (или пакетами), а потребитель ожидает их с другого конца. Поток можно *завершить*, чтобы сообщить об окончании данных, или *переключить в ошибку*, чтобы передать исключение потребителю.

Это принципиально отличается от обычного `Future[T]`, который разрешается ровно один раз с одним значением. `FutureStream[T]` может разрешаться многократно — по одному разу на каждое записанное значение — что делает его естественным строительным блоком для:

- Потоковой передачи тела HTTP-ответа
- Скачивания файлов чанками по сети
- Лент данных в реальном времени
- Конвейеров «производитель-потребитель» в асинхронном коде

```nim
import std/asyncstreams
```

---

## Ментальная модель: жизненный цикл FutureStream

```
newFutureStream()          # создать открытый пустой поток
        │
        ├─ write(v1) ──►  потребитель читает v1 через read()
        ├─ write(v2) ──►  потребитель читает v2 через read()
        ├─ write(v3) ──►  потребитель читает v3 через read()
        │
        ├─ complete()      # сигнал «данных больше не будет»
        │   ИЛИ
        └─ fail(err)       # сигнал «что-то пошло не так»

После complete() + все данные прочитаны → finished() == true
После fail()                             → failed()   == true
```

---

## Тип

### `FutureStream[T]`

```nim
type FutureStream*[T] = ref object
  queue:    Deque[T]           # внутренний FIFO-буфер
  finished: bool               # true после complete() или fail()
  cb:       proc () {...}      # внутренний колбек пробуждения
  error*:   ref Exception      # не nil, если вызван fail()
```

Ссылочный тип (сборка мусора). Внутренняя очередь реализована через `Deque[T]`, поэтому значения поступают и уходят в **FIFO-порядке** — первым записано, первым прочитано. Поле `error` — единственное публичное поле; всё взаимодействие с потоком осуществляется через процедуры, описанные ниже.

---

## Процедуры

### `newFutureStream`

```nim
proc newFutureStream*[T](fromProc = "unspecified"): FutureStream[T]
```

#### Что делает

Создаёт новый **открытый** и **пустой** `FutureStream[T]`. Поток сразу готов принимать данные. Необязательный параметр `fromProc` — строка-метка этого потока в отладочном выводе. Передавайте имя процедуры-владельца потока, чтобы трассировки ошибок стало проще читать.

Внутренний колбек (`cb`) начинается как `nil` и устанавливается автоматически процедурами `read` и `callback=`, когда потребитель регистрирует интерес к потоку.

#### Пример

```nim
import std/[asyncdispatch, asyncstreams]

# Поток строковых чанков — например, тело HTTP-ответа, приходящее частями
let bodyStream = newFutureStream[string]("downloadBody")

# Поток показаний датчиков
let sensorStream = newFutureStream[float]("readSensors")
```

---

### `write`

```nim
proc write*[T](future: FutureStream[T], value: T): Future[void]
```

#### Что делает

Помещает `value` в **конец** внутренней очереди потока и немедленно будит ожидающего потребителя. Возвращает `Future[void]`, который уже завершён при успехе, поэтому `await` необязателен, но принят по соглашению.

Если поток уже был завершён через `complete` или `fail`, `write` переводит возвращаемый future в состояние ошибки `ValueError` — записывать данные в закрытый поток нельзя.

#### Важная деталь

На данный момент механизм **обратного давления (backpressure) отсутствует**. Если производитель пишет быстрее, чем потребитель читает, очередь растёт неограниченно. Будущая версия API планирует добавить управление потоком (см. TODO-комментарий в исходнике).

#### Пример

```nim
proc producer(stream: FutureStream[string]) {.async.} =
  await stream.write("первый чанк")
  await stream.write("второй чанк")
  await stream.write("третий чанк")
  stream.complete()   # сигнал: данных больше не будет

# Запись в завершённый поток вызывает ошибку:
let s = newFutureStream[int]()
s.complete()
let fut = s.write(42)
assert fut.failed   # нельзя писать после complete()
```

---

### `read`

```nim
proc read*[T](future: FutureStream[T]): owned(Future[(bool, T)])
```

#### Что делает

Возвращает `Future`, который разрешится кортежем `(bool, T)`, когда данные станут доступны:

- `(true, value)` — значение успешно извлечено из начала очереди. Значение удаляется из внутренней очереди потока.
- `(false, default(T))` — поток завершён через `complete()` и очередь пуста; данных больше не будет никогда.

Если поток **провалился** (`fail` был вызван), возвращаемый future сам переходит в состояние ошибки с ошибкой потока — её необходимо обрабатывать через `try/except` вокруг `await`.

`read` спроектирован для использования в цикле: читайте, пока не получите `false`, затем останавливайтесь. Это канонический паттерн потребления.

#### Пример

```nim
proc consumer(stream: FutureStream[string]) {.async.} =
  while true:
    let (hasData, chunk) = await stream.read()
    if not hasData:
      break               # поток завершён, чанков больше нет
    echo "Получено: ", chunk

# С обработкой ошибок:
proc safeConsumer(stream: FutureStream[string]) {.async.} =
  while true:
    try:
      let (hasData, chunk) = await stream.read()
      if not hasData: break
      echo chunk
    except Exception as e:
      echo "Ошибка потока: ", e.msg
      break
```

---

### `complete`

```nim
proc complete*[T](future: FutureStream[T])
```

#### Что делает

Сигнализирует **окончание данных**. После вызова `complete`:

- Дальнейшие вызовы `write` невозможны (завершатся с `ValueError`).
- Любые ожидающие вызовы `read` пробуждаются.
- Как только очередь полностью опустеет, `finished()` вернёт `true`.

Вызов `complete` для потока, который уже завершился ошибкой (был вызван `fail`), вызовет `AssertionDefect` — поток можно закрыть лишь один раз.

`complete` **не уничтожает** данные, уже находящиеся в очереди. Потребитель прочитает все ранее записанные значения, и только после этого `read` вернёт `(false, ...)`.

#### Пример

```nim
proc streamNumbers(s: FutureStream[int]) {.async.} =
  for i in 1..5:
    await s.write(i)
  s.complete()  # потребитель получит 1,2,3,4,5, затем (false, 0)
```

---

### `fail`

```nim
proc fail*[T](future: FutureStream[T], error: ref Exception)
```

#### Что делает

Завершает поток с **ошибкой**. Ошибка сохраняется в `future.error`, поток помечается как завершённый. Любой потребитель, ожидающий результата `read`, получит ошибку через свой future.

Используйте `fail` для передачи исключений через асинхронные границы — например, когда сетевое соединение обрывается в середине потока.

Как и `complete`, `fail` разрешён к вызову лишь единожды; повторный вызов на уже завершённом потоке вызовет `AssertionDefect`.

#### Пример

```nim
proc riskyProducer(s: FutureStream[string]) {.async.} =
  try:
    await s.write("частичные данные")
    raise newException(IOError, "соединение прервано")
  except IOError as e:
    s.fail(e)   # передаём ошибку потребителю

proc consumer(s: FutureStream[string]) {.async.} =
  try:
    while true:
      let (ok, chunk) = await s.read()
      if not ok: break
      echo chunk
  except IOError as e:
    echo "Производитель завершился с ошибкой: ", e.msg
```

---

### `finished`

```nim
proc finished*[T](future: FutureStream[T]): bool
```

#### Что делает

Возвращает `true` **только при одновременном выполнении обоих условий**:

1. Был вызван `complete()` или `fail()`.
2. Внутренняя очередь **пуста** (все записанные данные прочитаны).

Это более сильная гарантия, чем просто проверка факта вызова `complete` — она говорит о том, что потребитель «догнал» производителя и обрабатывать больше нечего.

```nim
let s = newFutureStream[int]()
assert not s.finished   # открытый пустой поток — ещё не завершён

s.complete()
assert not s.finished   # complete вызван, но в очереди ещё могут быть данные

# После прочтения всех элементов finished() станет true
```

> **Совет:** Внутри обработчика `callback=` всегда проверяйте `finished()`, чтобы понять — продолжать обработку или завершить её корректно.

---

### `failed`

```nim
proc failed*[T](future: FutureStream[T]): bool
```

#### Что делает

Возвращает `true`, если для потока был вызван `fail` — то есть если `future.error` не равен `nil`. Состояние очереди не проверяется.

```nim
let s = newFutureStream[string]()
assert not s.failed

s.fail(newException(IOError, "упс"))
assert s.failed
assert s.error.msg == "упс"
```

---

### `len`

```nim
proc len*[T](future: FutureStream[T]): int
```

#### Что делает

Возвращает количество значений, находящихся в данный момент во внутренней очереди — данные, которые были записаны через `write`, но ещё не прочитаны через `read`. Полезно для мониторинга глубины очереди или ручной реализации простого управления потоком.

```nim
let s = newFutureStream[int]()
assert s.len == 0

waitFor s.write(1)
waitFor s.write(2)
assert s.len == 2

discard waitFor s.read()   # читает и удаляет 1
assert s.len == 1
```

---

### `callback=`

```nim
proc `callback=`*[T](future: FutureStream[T],
    cb: proc (future: FutureStream[T]) {.closure, gcsafe.})
```

#### Что делает

Регистрирует **низкоуровневый колбек пробуждения**, который вызывается каждый раз, когда:

- В поток записаны новые данные (вызван `write`).
- Поток завершён или переведён в ошибку (вызван `complete` или `fail`).

Если на момент назначения в потоке уже есть данные или он уже завершён, `cb` немедленно планируется к вызову через `callSoon`.

Колбек получает сам `FutureStream` в качестве аргумента, чтобы иметь возможность проверить `finished()`, `failed()` и `len` и принять решение о дальнейших действиях.

> **Примечание:** В большинстве случаев следует использовать `read()` в цикле, а не устанавливать колбек напрямую. API на основе колбека является низкоуровневым и используется внутри `read`, а также в парсере тела запроса в `httpclient`.

#### Пример

```nim
import std/[asyncdispatch, asyncstreams]

let s = newFutureStream[int]()

s.callback = proc(fs: FutureStream[int]) =
  if fs.failed:
    echo "Ошибка: ", fs.error.msg
    return
  while fs.len > 0:
    # Здесь нельзя использовать await, поэтому проверяем глубину очереди
    echo "Данные есть, глубина очереди: ", fs.len

waitFor s.write(10)
waitFor s.write(20)
s.complete()

runForever()  # даём возможность сработать колбекам
```

---

## Полный пример использования

Следующий пример показывает производителя и потребителя, работающих параллельно в одном цикле событий, соединённых через `FutureStream`.

```nim
import std/[asyncdispatch, asyncstreams]

proc producer(stream: FutureStream[string]) {.async.} =
  let chunks = @["Привет, ", "асинхронный ", "мир!"]
  for chunk in chunks:
    await stream.write(chunk)
    await sleepAsync(100)   # имитация сетевой задержки между чанками
  stream.complete()
  echo "Производитель: завершён."

proc consumer(stream: FutureStream[string]) {.async.} =
  var assembled = ""
  while true:
    let (hasData, chunk) = await stream.read()
    if not hasData:
      break
    assembled.add(chunk)
    echo "Потребитель: получен '", chunk, "'"
  echo "Потребитель: полное сообщение = '", assembled, "'"

proc main() {.async.} =
  let stream = newFutureStream[string]("пример")
  await all(producer(stream), consumer(stream))

waitFor main()

# Вывод:
# Потребитель: получен 'Привет, '
# Потребитель: получен 'асинхронный '
# Потребитель: получен 'мир!'
# Производитель: завершён.
# Потребитель: полное сообщение = 'Привет, асинхронный мир!'
```

---

## Пример передачи ошибки

```nim
import std/[asyncdispatch, asyncstreams]

proc faultyProducer(stream: FutureStream[int]) {.async.} =
  await stream.write(1)
  await stream.write(2)
  stream.fail(newException(IOError, "соединение потеряно"))

proc consumer(stream: FutureStream[int]) {.async.} =
  try:
    while true:
      let (ok, v) = await stream.read()
      if not ok: break
      echo "Получено: ", v
  except IOError as e:
    echo "Поток завершился с ошибкой: ", e.msg

proc main() {.async.} =
  let stream = newFutureStream[int]("примерСОшибкой")
  await all(faultyProducer(stream), consumer(stream))

waitFor main()

# Вывод:
# Получено: 1
# Получено: 2
# Поток завершился с ошибкой: соединение потеряно
```

---

## Сводная таблица процедур

| Процедура | Сигнатура | Описание |
|-----------|-----------|---------|
| `newFutureStream` | `(fromProc = "unspecified"): FutureStream[T]` | Создать открытый пустой поток. |
| `write` | `(future, value: T): Future[void]` | Записать значение; завершится ошибкой, если поток закрыт. |
| `read` | `(future): Future[(bool, T)]` | Прочитать следующее значение; `false` — поток закончился. |
| `complete` | `(future)` | Сигнал окончания данных; дальнейшие записи запрещены. |
| `fail` | `(future, error: ref Exception)` | Завершить поток с ошибкой. |
| `finished` | `(future): bool` | `true` тогда и только тогда, когда закрыт **и** очередь пуста. |
| `failed` | `(future): bool` | `true`, если был вызван `fail`. |
| `len` | `(future): int` | Количество значений в буфере очереди. |
| `callback=` | `(future, cb: proc(FutureStream[T]))` | Зарегистрировать низкоуровневый колбек пробуждения. |

---

## Ключевые инварианты

- В поток можно записывать до тех пор, пока не вызваны `complete` или `fail`.
- `complete` и `fail` взаимно исключают друг друга, каждый может быть вызван не более одного раза.
- `finished()` равен `true` только после закрытия **и** после того, как очередь опустела.
- `read` извлекает значения в FIFO-порядке.
- Встроенного обратного давления нет; производители должны самостоятельно регулировать скорость записи, если потребитель медленный.
