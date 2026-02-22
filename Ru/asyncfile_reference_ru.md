# Справочник модуля `std/asyncfile` для Nim

> Асинхронный файловый ввод/вывод в экосистеме `async`/`await` Nim. Модуль позволяет читать файлы и писать в них, не блокируя цикл событий: пока ждёт операция с файлом, другие асинхронные задачи (сетевые запросы, таймеры и т.д.) продолжают выполняться.
>
> Внутри модуль использует **IOCP** (I/O Completion Ports) на Windows и **неблокирующие POSIX-дескрипторы** на Unix-подобных системах. Механизм различается по платформам, однако публичный API одинаков на каждой из них.
>
> Требует импорта `std/asyncdispatch` и работающего цикла событий (через `waitFor` или `runForever`).

---

## Содержание

1. [Ключевая концепция: асинхронный файловый ввод/вывод](#ключевая-концепция-асинхронный-файловый-вводвывод)
2. [Типы данных](#типы-данных)
   - [AsyncFile](#asyncfile)
3. [Открытие и создание файлов](#открытие-и-создание-файлов)
   - [openAsync](#openasync)
   - [newAsyncFile](#newasyncfile)
4. [Чтение](#чтение)
   - [read](#read)
   - [readBuffer](#readbuffer)
   - [readLine](#readline)
   - [readAll](#readall)
   - [readToStream](#readtostream)
5. [Запись](#запись)
   - [write](#write)
   - [writeBuffer](#writebuffer)
   - [writeFromStream](#writefromstream)
6. [Указатель файла и метаданные](#указатель-файла-и-метаданные)
   - [getFilePos](#getfilepos)
   - [setFilePos](#setfilepos)
   - [getFileSize](#getfilesize)
   - [setFileSize](#setfilesize)
7. [Закрытие](#закрытие)
   - [close](#close)
8. [Справка по FileMode](#справка-по-filemode)
9. [Полный практический пример](#полный-практический-пример)
10. [Шпаргалка](#шпаргалка)

---

## Ключевая концепция: асинхронный файловый ввод/вывод

В синхронном Nim-коде вызов `file.read(1024)` блокирует текущий поток до тех пор, пока ОС не доставит данные. В async-программе это заморозило бы весь цикл событий: никакие другие корутины не продвигались бы вперёд, пока идёт ожидание.

`std/asyncfile` решает эту проблему, возвращая `Future[T]` из каждого вызова ввода/вывода. `Future` разрешается позднее — циклом событий, когда ОС сигнализирует о завершении операции. Ваша корутина приостанавливается на каждом `await` и автоматически возобновляется, как только данные готовы.

```
Ваша async-процедура       Цикл событий            ОС
──────────────────────     ────────────            ──
await f.read(4096)  ──────► регистрирует колбэк
(приостановлена)            выполняет другие задачи ◄──── сигнал: данные готовы
(возобновлена)   ◄────────── вызывает ваш колбэк
```

Это позволяет одному потоку обрабатывать множество параллельных файловых операций — идеально для высоконагруженных сценариев: обработчиков логов, HTTP-серверов, отдающих крупные файлы.

---

## Типы данных

### `AsyncFile`

```nim
type AsyncFile* = ref object
  fd: AsyncFD    # Платформенный файловый дескриптор, зарегистрированный в цикле событий
  offset: int64  # Текущая позиция чтения/записи (внутреннее поле)
```

**Что это:** Ссылочный тип (`ref`), обёртывающий открытый файл. Так как это `ref`, его можно свободно передавать и хранить без опасений о копировании. Поля `fd` и `offset` — внутренние, вы никогда не обращаетесь к ним напрямую; вместо этого используйте `getFilePos`, `setFilePos` и процедуры чтения/записи.

`AsyncFile` **всегда** нужно закрывать через `close` по завершении работы с ним. Сборщик мусора Nim не закроет его за вас.

---

## Открытие и создание файлов

### `openAsync`

```nim
proc openAsync*(filename: string, mode = fmRead): AsyncFile
```

**Что делает:** Открывает (или создаёт) файл по пути `filename` с указанным режимом `mode` и регистрирует его файловый дескриптор в асинхронном цикле событий. Это стандартная точка входа — почти каждый async-файловый сценарий начинается с неё.

При ошибке (файл не найден, отказ в доступе и т.д.) немедленно возбуждает `OSError`. Сам `openAsync` **не** является асинхронным — только последующие операции чтения/записи будут асинхронными.

Параметр `mode` — это стандартное перечисление Nim `FileMode`. Все значения и их поведение описаны в таблице [Справка по FileMode](#справка-по-filemode).

**Когда использовать:** Всегда, когда нужно открыть файл для асинхронного чтения или записи. Предпочтительнее `newAsyncFile`, если только вы не интегрируетесь с уже существующим низкоуровневым дескриптором.

```nim
import std/[asyncfile, asyncdispatch, os]

proc example() {.async.} =
  # Открыть для чтения (по умолчанию)
  let r = openAsync("data.txt")
  defer: r.close()

  # Открыть для записи — создаёт или очищает файл
  let w = openAsync("output.txt", fmWrite)
  defer: w.close()

  # Открыть для дозаписи — никогда не очищает
  let a = openAsync("log.txt", fmAppend)
  defer: a.close()

  # Открыть существующий файл для чтения и записи
  let rw = openAsync("existing.db", fmReadWriteExisting)
  defer: rw.close()

waitFor example()
```

---

### `newAsyncFile`

```nim
proc newAsyncFile*(fd: AsyncFD): AsyncFile
```

**Что делает:** Оборачивает **уже открытый** `AsyncFD` (асинхронный файловый дескриптор) в объект `AsyncFile` и регистрирует `fd` в цикле событий через `register`. Это низкоуровневый «выход на волю» для случаев, когда дескриптор получен вне данного модуля — например, из C FFI-кода, из `socketpair` или из `os.open`.

**Когда использовать:** Редко. Большинство кода должно использовать `openAsync`. `newAsyncFile` нужен лишь тогда, когда у вас уже есть `AsyncFD` из другого источника.

```nim
import std/[asyncfile, asyncdispatch]

# Гипотетически: C-библиотека возвращает сырой дескриптор
let rawFd: cint = someExternalLibraryOpen("file.bin")
let f = newAsyncFile(rawFd.AsyncFD)
defer: f.close()
```

> **Важно:** Вы сами несёте ответственность за то, чтобы `fd` был открыт с флагами неблокирующего ввода/вывода на POSIX или с `FILE_FLAG_OVERLAPPED` на Windows. `openAsync` делает это автоматически.

---

## Чтение

### `read`

```nim
proc read*(f: AsyncFile, size: int): Future[string]
```

**Что делает:** Асинхронно читает **до** `size` байт от текущей позиции файлового указателя и возвращает их как `string`. Указатель продвигается на количество фактически прочитанных байт.

Два важных граничных случая:
- Если файловый указатель уже стоит на конце файла (или за ним), `Future` разрешается в **пустую строку** `""`.
- Возвращённая строка может быть **короче** `size` байт, если в процессе чтения встретился конец файла. Всегда проверяйте `result.len`, а не только факт пустоты.

`size` должен быть больше нуля (проверяется через `assert`).

**Когда использовать:** Когда нужно прочитать известный кусок данных как строку Nim — это наиболее распространённая высокоуровневая операция чтения.

```nim
import std/[asyncfile, asyncdispatch]

proc example() {.async.} =
  let f = openAsync("data.bin")
  defer: f.close()

  # Читаем первые 16 байт (может быть меньше, если файл короче)
  let header = await f.read(16)
  echo "Прочитано ", header.len, " байт: ", header

  # Читаем порциями по 4 КБ до конца файла
  while true:
    let chunk = await f.read(4096)
    if chunk.len == 0: break
    process(chunk)

waitFor example()
```

---

### `readBuffer`

```nim
proc readBuffer*(f: AsyncFile, buf: pointer, size: int): Future[int]
```

**Что делает:** Асинхронно читает **до** `size` байт непосредственно в сырой буфер памяти, на который указывает `buf`. Возвращает `Future[int]`, разрешающийся в количество фактически прочитанных байт. Возвращает `0`, если файловый указатель стоит на конце файла или за ним.

Это низкоуровневый аналог `read`. Он эффективнее в тех случаях, когда буфер уже выделен вами (нет лишнего выделения и копирования), или при работе с C-кодом, которому нужен интерфейс через `pointer`.

**Когда использовать:** В производительно-критичном коде, где буфер выделяется заранее, чтобы избежать выделения строки, которое делает `read`.

```nim
import std/[asyncfile, asyncdispatch]

proc example() {.async.} =
  let f = openAsync("data.bin")
  defer: f.close()

  var buf = alloc(1024)
  defer: dealloc(buf)

  let bytesRead = await f.readBuffer(buf, 1024)
  echo "В буфер прочитано ", bytesRead, " байт"

waitFor example()
```

---

### `readLine`

```nim
proc readLine*(f: AsyncFile): Future[string] {.async.}
```

**Что делает:** Асинхронно читает одну строку текста из файла, останавливаясь на `\n` (LF) или `\r\n` (CRLF). Возвращаемая строка **не** включает символ(ы) переноса строки. Возвращает пустую строку `""` при достижении конца файла.

Внутри вызывает `read(f, 1)` в цикле, читая по одному байту. Это удобно, но не слишком эффективно при очень длинных строках или высокой пропускной способности — в таких случаях лучше читать большими кусками и делить вручную.

**Когда использовать:** Для построчного чтения текстовых файлов, когда удобство важнее максимальной производительности.

```nim
import std/[asyncfile, asyncdispatch]

proc readAllLines() {.async.} =
  let f = openAsync("config.txt")
  defer: f.close()

  while true:
    let line = await f.readLine()
    if line.len == 0: break  # EOF
    echo "Строка: ", line

waitFor readAllLines()
```

---

### `readAll`

```nim
proc readAll*(f: AsyncFile): Future[string] {.async.}
```

**Что делает:** Читает **всё оставшееся содержимое** файла от текущей позиции до конца, накапливая всё в одну строку. Делает это вызовами `read(f, 4000)` до тех пор, пока не получит пустой результат (признак EOF).

Поскольку все данные накапливаются в памяти, подходит только для файлов, которые целиком помещаются в RAM. Для больших файлов предпочтительнее потоковая обработка через `readToStream` или ручное чтение порциями.

**Когда использовать:** Для небольших и средних конфигурационных файлов, шаблонов и других ресурсов, которые нужны целиком в памяти.

```nim
import std/[asyncfile, asyncdispatch, os]

proc loadConfig() {.async.} =
  let f = openAsync(getConfigDir() / "settings.json")
  defer: f.close()

  let json = await f.readAll()
  echo "Загружено ", json.len, " байт конфигурации"

waitFor loadConfig()
```

---

### `readToStream`

```nim
proc readToStream*(f: AsyncFile, fs: FutureStream[string]) {.async.}
```

**Что делает:** Читает файл порциями по 4000 байт и записывает каждую порцию в `FutureStream[string]` `fs` по мере поступления. По достижении конца файла вызывает `fs.complete()`, сигнализируя, что новых данных не будет. Файл при этом не закрывается.

`FutureStream` — это асинхронный канал «производитель–потребитель» Nim. Записывая в него данные здесь, вы позволяете потребителю на другом конце обрабатывать данные по мере их поступления, не дожидаясь загрузки всего файла.

**Когда использовать:** Когда нужно передать содержимое файла другому async-потребителю — например, в тело HTTP-ответа, конвейер сжатия или парсер — без буферизации всего файла.

```nim
import std/[asyncfile, asyncdispatch, asyncstreams]

proc streamFile(path: string): FutureStream[string] =
  result = newFutureStream[string]("streamFile")
  let fs = result
  asyncCheck (proc() {.async.} =
    let f = openAsync(path)
    defer: f.close()
    await f.readToStream(fs)
  )()

proc consumer() {.async.} =
  let stream = streamFile("large.log")
  while true:
    let (hasData, chunk) = await stream.read()
    if not hasData: break
    process(chunk)

waitFor consumer()
```

---

## Запись

### `write`

```nim
proc write*(f: AsyncFile, data: string): Future[void]
```

**Что делает:** Асинхронно записывает всё содержимое строки `data` в файл на текущую позицию файлового указателя. `Future[void]` разрешается, когда **все** байты были переданы ОС (но не обязательно сброшены на физический диск). Указатель продвигается на `data.len` байт.

Запись пустой строки (`""`) на POSIX — это no-op (вызывает `write` с размером 0) и завершается успешно.

**Когда использовать:** В подавляющем большинстве async-операций записи — это идиоматичный высокоуровневый API.

```nim
import std/[asyncfile, asyncdispatch, os]

proc writeExample() {.async.} =
  let f = openAsync(getTempDir() / "out.txt", fmWrite)
  defer: f.close()

  await f.write("Привет, асинхронный мир!\n")
  await f.write("Вторая строка.\n")

waitFor writeExample()
```

---

### `writeBuffer`

```nim
proc writeBuffer*(f: AsyncFile, buf: pointer, size: int): Future[void]
```

**Что делает:** Асинхронно записывает ровно `size` байт из сырого буфера памяти `buf` в файл. `Future[void]` разрешается, когда все байты записаны. Как и `write`, продвигает указатель файла на `size` байт.

Это низкоуровневый аналог `write`. Позволяет избежать выделения строки Nim, когда данные уже находятся в буфере, выделенном через C или вручную.

**Когда использовать:** В производительно-критичном коде или при C-интеропе, когда данные для записи уже доступны через `pointer`.

```nim
import std/[asyncfile, asyncdispatch]

proc writeRaw() {.async.} =
  let f = openAsync("raw.bin", fmWrite)
  defer: f.close()

  var data: array[8, byte] = [0x4e'u8, 0x49, 0x4d, 0x00, 0x01, 0x00, 0x00, 0x00]
  await f.writeBuffer(addr data[0], data.len)

waitFor writeRaw()
```

---

### `writeFromStream`

```nim
proc writeFromStream*(f: AsyncFile, fs: FutureStream[string]) {.async.}
```

**Что делает:** Зеркальная процедура для `readToStream` — на стороне записи. Читает строковые порции из `FutureStream[string]` `fs` по одной и немедленно записывает каждую в файл, освобождая память. Останавливается, когда `fs` сигнализирует о завершении (т.е. когда `hasValue` равно `false`). Файл при этом не закрывается.

Поскольку каждая порция записывается и отбрасывается до получения следующей, пиковое потребление памяти ограничено размером одной порции, независимо от полного размера файла.

**Когда использовать:** Для сохранения потоковых данных (сетевые загрузки, генерируемый контент, вывод пайпа) на диск без буферизации всей полезной нагрузки в RAM.

```nim
import std/[asyncfile, asyncdispatch, asyncstreams]

proc saveStream(path: string, source: FutureStream[string]) {.async.} =
  let f = openAsync(path, fmWrite)
  defer: f.close()
  await f.writeFromStream(source)

# Пример: сохранение сетевой загрузки прямо на диск
# let stream = httpClient.getBodyStream("https://example.com/large.zip")
# waitFor saveStream("large.zip", stream)
```

---

## Указатель файла и метаданные

### `getFilePos`

```nim
proc getFilePos*(f: AsyncFile): int64
```

**Что делает:** Возвращает текущую позицию чтения/записи файлового указателя в виде байтового смещения от начала файла. Первый байт файла находится на позиции `0`. Это **синхронный**, мгновенный вызов — он просто читает внутреннее поле `offset` и возвращает его.

**Когда использовать:** Чтобы сохранить текущую позицию перед seek'ом и вернуться к ней позже, или чтобы вычислить, сколько байт уже прочитано.

```nim
import std/[asyncfile, asyncdispatch]

proc example() {.async.} =
  let f = openAsync("data.txt")
  defer: f.close()

  let startPos = f.getFilePos()   # 0
  discard await f.read(100)
  let afterRead = f.getFilePos()  # 100
  echo "Прочитано ", afterRead - startPos, " байт"

waitFor example()
```

---

### `setFilePos`

```nim
proc setFilePos*(f: AsyncFile, pos: int64)
```

**Что делает:** Перемещает файловый указатель на байтовое смещение `pos` от начала файла. Первый байт — на позиции `0`. На POSIX-системах немедленно вызывает `lseek` (и возбуждает `OSError` при ошибке); на Windows смещение просто сохраняется и используется в следующем overlapped I/O вызове. После `setFilePos` следующая операция `read` или `write` будет выполняться начиная с позиции `pos`.

**Когда использовать:** Для реализации произвольного доступа (например, повторное чтение заголовка, переход к известной записи), или для сброса в начало после записи, чтобы прочитать то, что было написано.

```nim
import std/[asyncfile, asyncdispatch, os]

proc roundTrip() {.async.} =
  let f = openAsync(getTempDir() / "tmp.txt", fmReadWrite)
  defer: f.close()

  await f.write("Привет, Мир!")

  # Возврат в начало и повторное чтение
  f.setFilePos(0)
  let content = await f.readAll()
  assert content == "Привет, Мир!"

waitFor roundTrip()
```

---

### `getFileSize`

```nim
proc getFileSize*(f: AsyncFile): int64
```

**Что делает:** Возвращает полный размер файла в байтах как `int64`. Это **синхронный** вызов. На POSIX временно перемещается в конец файла для определения размера, а затем возвращается на исходную позицию — так что после вызова указатель файла не изменяется. На Windows использует Win32 API `GetFileSize` напрямую, без seek'а.

**Когда использовать:** Когда нужно заранее знать размер файла — например, для предварительного выделения буфера, отображения прогресса или проверки ожидаемого размера.

```nim
import std/[asyncfile, asyncdispatch]

proc example() {.async.} =
  let f = openAsync("archive.zip")
  defer: f.close()

  let size = f.getFileSize()
  echo "Размер файла: ", size, " байт"

  # Выделяем буфер точного размера
  var buf = newString(size)
  discard await f.readBuffer(addr buf[0], size.int)

waitFor example()
```

---

### `setFileSize`

```nim
proc setFileSize*(f: AsyncFile, length: int64)
```

**Что делает:** Усекает или расширяет файл ровно до `length` байт. Это **синхронный** вызов. Если `length` меньше текущего размера файла, файл усекается и лишние данные теряются. Если `length` больше, файл расширяется (содержимое новых байт определяется платформой — как правило, нули на большинстве систем, но это не гарантировано).

На POSIX вызывает `ftruncate`. На Windows перемещает указатель и вызывает `SetEndOfFile`. При ошибке возбуждает `OSError`.

**Когда использовать:** Для предварительного выделения места в файле перед записью (улучшает производительность, исключая многократное перераспределение), или для очистки лог-файла.

```nim
import std/[asyncfile, asyncdispatch]

proc preallocate() {.async.} =
  let f = openAsync("preallocated.bin", fmReadWrite)
  defer: f.close()

  # Резервируем 1 МБ заранее
  f.setFileSize(1024 * 1024)

  # Теперь пишем — без лишних перераспределений в ОС
  f.setFilePos(0)
  await f.write("данные заголовка здесь")

waitFor preallocate()
```

---

## Закрытие

### `close`

```nim
proc close*(f: AsyncFile)
```

**Что делает:** Снимает регистрацию файлового дескриптора с асинхронного цикла событий и закрывает лежащий в основе дескриптор ОС. Это **синхронный** вызов. Возбуждает `OSError`, если системный вызов `close` завершается ошибкой (может случиться, если буферизованные записи не были сброшены, хотя асинхронные записи завершаются до разрешения `Future`, так что это редко).

**Когда использовать:** Всегда, когда работа с `AsyncFile` закончена. Если не закрыть файл, произойдёт утечка файлового дескриптора ОС и регистрации в цикле событий, что в конечном счёте исчерпает системные ресурсы. Используйте `defer: f.close()` сразу после `openAsync`, чтобы файл всегда закрывался, даже при исключении.

```nim
import std/[asyncfile, asyncdispatch]

proc example() {.async.} =
  let f = openAsync("data.txt")
  defer: f.close()   # ← всегда будет закрыт, даже при исключении

  let data = await f.readAll()
  echo data

waitFor example()
```

---

## Справка по FileMode

Перечисление `FileMode` определено в `std/io`, но активно используется в `asyncfile`. Вот что означает каждое значение при передаче в `openAsync`:

| Значение | Файл существует | Файл отсутствует | Чтение | Запись | Позиция |
|---|---|---|---|---|---|
| `fmRead` | Открывает | Ошибка | ✓ | ✗ | Начало |
| `fmWrite` | Очищает до 0 | Создаёт | ✗ | ✓ | Начало |
| `fmAppend` | Открывает | Создаёт | ✗ | ✓ | Конец |
| `fmReadWrite` | Очищает до 0 | Создаёт | ✓ | ✓ | Начало |
| `fmReadWriteExisting` | Открывает | Ошибка | ✓ | ✓ | Начало |

**Типичный выбор:**
- Загрузка конфига или файла данных → `fmRead`
- Запись нового файла вывода → `fmWrite`
- Добавление в лог-файл → `fmAppend`
- Правка существующего файла на месте → `fmReadWriteExisting`
- Создание нового временного файла для чтения/записи → `fmReadWrite`

---

## Полный практический пример

Самодостаточная программа, демонстрирующая полный сценарий async-файлового ввода/вывода:

```nim
import std/[asyncfile, asyncdispatch, os, strutils]

proc processLog(inputPath, outputPath: string) {.async.} =
  let src = openAsync(inputPath, fmRead)
  defer: src.close()

  let dst = openAsync(outputPath, fmWrite)
  defer: dst.close()

  echo "Размер входного файла: ", src.getFileSize(), " байт"

  var lineNum = 0
  while true:
    let line = await src.readLine()
    if line.len == 0 and src.getFilePos() >= src.getFileSize():
      break   # настоящий EOF (не просто пустая строка)
    inc lineNum
    # Сохраняем только строки, содержащие "ОШИБКА"
    if "ОШИБКА" in line:
      await dst.write($lineNum & ": " & line & "\n")

  echo "Готово. Результат записан в ", outputPath

proc main() {.async.} =
  let tmp = getTempDir()
  let input  = tmp / "app.log"
  let output = tmp / "errors.log"

  # Записываем тестовые данные лога
  let w = openAsync(input, fmWrite)
  await w.write("INFO  сервис запущен\n")
  await w.write("ОШИБКА диск заполнен\n")
  await w.write("INFO  повторная попытка\n")
  await w.write("ОШИБКА таймаут соединения\n")
  w.close()

  await processLog(input, output)

  # Читаем и отображаем результат
  let r = openAsync(output)
  defer: r.close()
  echo await r.readAll()

waitFor main()
```

---

## Шпаргалка

| Задача | API |
|---|---|
| Открыть файл для async ввода/вывода | `openAsync(path, mode)` |
| Обернуть существующий дескриптор | `newAsyncFile(fd)` |
| Прочитать N байт как строку | `await f.read(n)` |
| Прочитать N байт в буфер | `await f.readBuffer(buf, n)` |
| Прочитать одну строку | `await f.readLine()` |
| Прочитать весь файл | `await f.readAll()` |
| Передать файл в поток | `await f.readToStream(fs)` |
| Записать строку | `await f.write(data)` |
| Записать сырой буфер | `await f.writeBuffer(buf, n)` |
| Записать из потока | `await f.writeFromStream(fs)` |
| Получить текущую позицию | `f.getFilePos()` |
| Перейти к позиции | `f.setFilePos(pos)` |
| Получить размер файла | `f.getFileSize()` |
| Усечь / расширить файл | `f.setFileSize(length)` |
| Закрыть файл | `f.close()` |
| Паттерн гарантированного закрытия | `defer: f.close()` |
