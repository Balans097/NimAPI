# Справочник модуля `memfiles`

> **Стандартная библиотека Nim — `std/memfiles`**
> Файловый ввод-вывод через отображение в память (POSIX `mmap` / Windows `MapViewOfFile`).
> Быстрая итерация по строкам и записям с разделителями без копирования данных.
> Поддерживается только на Windows и POSIX-системах.

---

## Содержание

1. [Обзор и принцип работы](#обзор-и-принцип-работы)
2. [Типы](#типы)
   - [MemFile](#memfile)
   - [MemSlice](#memslice)
   - [MemMapFileStream](#memmapfilestream)
3. [Жизненный цикл MemFile](#жизненный-цикл-memfile)
   - [open](#open)
   - [close](#close)
   - [flush](#flush)
   - [resize](#resize)
4. [Отображение подобластей](#отображение-подобластей)
   - [mapMem](#mapmem)
   - [unmapMem](#unmapmem)
5. [Итерация по строкам и записям](#итерация-по-строкам-и-записям)
   - [memSlices](#memslices)
   - [lines (с буфером)](#lines-с-буфером)
   - [lines (простой)](#lines-простой)
6. [Утилиты MemSlice](#утилиты-memslice)
   - [== (равенство)](#-равенство)
   - [$ (в строку)](#-в-строку)
7. [MemMapFileStream](#memmapfilestream-1)
   - [newMemMapFileStream](#newmemmapfilestream)
8. [Платформенные особенности](#платформенные-особенности)
9. [Полные рабочие примеры](#полные-рабочие-примеры)

---

## Обзор и принцип работы

**Файлы, отображённые в память** — один из самых быстрых способов работы с файлами. Вместо того чтобы читать данные в буфер через `read()`, операционная система отображает байты файла непосредственно в виртуальное адресное пространство вашего процесса. После этого вы обращаетесь к ним через обычный указатель — ровно как если бы весь файл был массивом в оперативной памяти.

```
Традиционный ввод-вывод:         Отображение в память:
──────────────────────────        ──────────────────────────────────
Диск → буфер ядра                 Диск ⟶ страничный кеш ОС
  → пользовательский буфер (копия!)  ↑ доступен по адресу напрямую
  → ваша переменная (копия!)         └─ ваш указатель MemFile.mem
```

Ключевые преимущества:
- **Нулевое копирование при чтении**: данные никогда не копируются из пространства ядра в пользовательское.
- **Ленивая загрузка**: ОС подгружает только те страницы, к которым вы реально обращаетесь.
- **Запись на месте**: сохранение значения в `mem[i]` немедленно изменяет файл (после `flush`).
- **Быстрая итерация по строкам**: итератор `memSlices` использует `memchr` — один вызов C — для поиска разделителей без какого-либо копирования байт.

Типичный сценарий использования:

```
open(...)        → получаем MemFile, .mem указывает в тело файла
  │
  ├─ чтение:  обращаемся к .mem через cast/индексирование,
  │            или итерируем через lines/memSlices
  ├─ запись:  приводим .mem к нужному типу, пишем напрямую
  ├─ flush:   принудительно сбрасываем «грязные» страницы на диск
  └─ close:   записываем ожидающие изменения, освобождаем дескрипторы ОС
```

Модуль также оборачивает `MemFile` в стандартный интерфейс `Stream` через `MemMapFileStream`, что позволяет передавать файл, отображённый в память, в любой API, принимающий `Stream`.

---

## Типы

### `MemFile`

```nim
type MemFile* = object
  mem*:  pointer   ## указатель в начало отображённой области; чтение и запись — через него
  size*: int       ## количество байт в отображённой области

  # Только Windows (публичные):
  fHandle*:   Handle
  mapHandle*: Handle
  wasOpened*: bool

  # Только POSIX (публичное):
  handle*: cint
```

Центральный объект модуля. После успешного вызова `open`, `mem` указывает на начало байт файла в виртуальной памяти, а `size` содержит количество отображённых байт.

Чтение и запись в файл происходят через `mem` как через сырой указатель. Например:

```nim
# Трактуем весь файл как массив байт
let bytes = cast[ptr UncheckedArray[byte]](f.mem)
echo bytes[0]            # первый байт файла
bytes[0] = 0x42'u8       # запись в файл (только если открыт с правом записи)
```

Поля с дескрипторами платформы (`fHandle`, `mapHandle`, `handle`) открыты для продвинутых сценариев — прямых вызовов ОС, операций `ioctl` и т.д. В обычных случаях они не нужны.

> **Важно:** Никогда не устанавливайте `mem` и `size` вручную. Ими управляют процедуры `open`, `resize` и `close`.

---

### `MemSlice`

```nim
type MemSlice* = object
  data*: pointer
  size*: int
```

Лёгкий, «не-владеющий» вид на область `MemFile`. Содержит только сырой указатель и длину — ничего больше. Именно это возвращает итератор `memSlices` на каждом шаге.

`MemSlice` — **не** Nim-строка. Он не завершается нулём. Он не владеет своей памятью. Он действителен только пока открыт родительский `MemFile`. Если вам нужна постоянная копия, вызовите `$ms` для преобразования в обычную строку.

---

### `MemMapFileStream`

```nim
type
  MemMapFileStream*    = ref MemMapFileStreamObj
  MemMapFileStreamObj* = object of Stream
    mf:   MemFile
    mode: FileMode
    pos:  int
```

`ref`-тип, наследующий `Stream` и оборачивающий `MemFile`. После создания через `newMemMapFileStream` он ведёт себя как любой другой поток — поддерживает все стандартные операции из `std/streams`: `readLine`, `write`, `setPosition`, `atEnd`, `flush`, `close` и т.д.

---

## Жизненный цикл MemFile

### `open`

```nim
proc open*(
  filename:    string,
  mode:        FileMode = fmRead,
  mappedSize:  int      = -1,
  offset:      int      = 0,
  newFileSize: int      = -1,
  allowRemap:  bool     = false,
  mapFlags:    cint     = cint(-1)
): MemFile
```

**Что делает:** Открывает файл и отображает его в виртуальную память. Возвращает инициализированный `MemFile`, у которого `mem` указывает на начало отображённой области, а `size` содержит количество отображённых байт. При ошибке выбрасывает `OSError`.

**Параметры подробно:**

- **`filename`** — путь к файлу.

- **`mode`** — режим открытия файла. Основные значения:
  - `fmRead` (по умолчанию) — отображение только для чтения. Запись через `mem` вызовет segfault.
  - `fmReadWrite` — отображение для чтения и записи. Изменения через `mem` записываются на диск при `flush`/`close`.
  - `fmWrite` — используется вместе с `newFileSize` для *создания нового файла*. Файл не должен существовать при таком сочетании.
  - `fmAppend` **не поддерживается** и вызывает `IOError`.

- **`mappedSize`** — сколько байт отобразить, начиная с `offset`. `-1` (по умолчанию) — отображать весь файл (или весь `newFileSize` для нового файла). Положительное значение отображает только часть большого файла.

- **`offset`** — смещение в байтах от начала файла, с которого начинается отображение. **Должен быть кратен размеру страницы ОС** (обычно 4096 байт). Начать отображение с произвольного байта нельзя.

- **`newFileSize`** — если > 0, создаёт новый файл ровно такого размера и отображает его. Допустимо только при `mode != fmRead`. Не влияет на уже существующие файлы.

- **`allowRemap`** — если `true`, дескриптор файла ОС остаётся открытым после создания отображения, чтобы впоследствии можно было вызвать `resize`. Если `false` (по умолчанию), дескриптор закрывается после отображения для экономии ресурсов. Устанавливайте в `true` только если планируете вызывать `resize`.

- **`mapFlags`** — продвинутый параметр: переопределить стандартные флаги `mmap`/`MapViewOfFile`. По умолчанию на POSIX используется `MAP_SHARED`. Неправильные флаги могут привести к сбою `open` или непредсказуемому поведению.

```nim
import std/memfiles

# 1. Открыть существующий файл только для чтения (самый частый случай)
var f = memfiles.open("data.txt")
defer: f.close()
# f.mem указывает на байты файла, f.size = длина файла в байтах

# 2. Открыть для чтения и записи — изменения на месте
var g = memfiles.open("config.bin", mode = fmReadWrite)
defer: g.close()

# 3. Создать новый файл 4 КБ (ОС заполнит нулями)
var h = memfiles.open("/tmp/scratch.bin", mode = fmReadWrite, newFileSize = 4096)
defer: h.close()

# 4. Отобразить только первые 512 байт большого файла
var part = memfiles.open("bigfile.dat", mode = fmRead, mappedSize = 512)
defer: part.close()

# 5. Отобразить 512 байт, начиная с байта 4096 (одна страница от начала)
var mid = memfiles.open("bigfile.dat", mode = fmRead,
                         mappedSize = 512, offset = 4096)
defer: mid.close()
```

---

### `close`

```nim
proc close*(f: var MemFile)
```

**Что делает:** Снимает отображение виртуальной памяти, сбрасывает все «грязные» (изменённые) страницы на диск и закрывает все дескрипторы ОС. После `close` поле `f.mem` становится `nil`, а `f.size` — `0`. При ошибке выбрасывает `OSError`.

Всегда вызывайте `close` по завершении работы. Идиоматичный Nim-паттерн — `defer: f.close()` сразу после `open`.

```nim
var f = memfiles.open("log.txt")
defer: f.close()  # гарантированно выполнится, даже при исключении
# ... работаем с f ...
```

Если файл был открыт с правом записи, именно `close` гарантирует, что данные безопасно записаны в файловую систему. Не завершайте процесс резко до вызова `close`, если важна целостность данных.

---

### `flush`

```nim
proc flush*(f: var MemFile; attempts: Natural = 3)
```

**Что делает:** Принудительно записывает все изменённые страницы отображённой области в файл на диске. При ошибке после `attempts` попыток выбрасывает `OSError`.

На POSIX вызывает `msync(MS_SYNC | MS_INVALIDATE)`, на Windows — `FlushViewOfFile`.

Логика повторных попыток нужна потому, что могут возникать временные ошибки (`EBUSY` на POSIX, `ERROR_LOCK_VIOLATION` на Windows), когда другие процессы читают те же страницы. По умолчанию 3 попытки покрывают большинство реальных ситуаций.

**Когда использовать:** Когда вы записали данные через `mem` и хотите гарантировать их сохранность до продолжения — например, перед вызовом `resize`, перед созданием контрольной точки, или периодически в долго работающем процессе, ведущем запись в большой отображённый файл.

```nim
var f = memfiles.open("output.bin", mode = fmReadWrite)
defer: f.close()

let data = cast[ptr UncheckedArray[byte]](f.mem)
data[0] = 0xFF

# Убедиться, что байт 0 на диске, прежде чем продолжить:
f.flush()
```

---

### `resize`

```nim
proc resize*(f: var MemFile, newFileSize: int) {.raises: [IOError, OSError].}
```

**Что делает:** Изменяет размер файла на диске *и* отображение в памяти до `newFileSize` байт. После вызова `f.mem` **может указывать на другой адрес** (ОС может переместить отображение), а `f.size` обновляется. Выбрасывает `IOError`, если `newFileSize < 1` или файл был открыт без `allowRemap = true`.

На Linux используется `mremap` — расширение отображения на месте, которое может быть более чем в 100 раз быстрее по сравнению со снятием и повторным созданием отображения на других POSIX-системах. На других POSIX и на Windows отображение уничтожается и пересоздаётся.

**Предусловия:**
- Файл должен быть открыт с `allowRemap = true`.
- Весь файл должен быть отображён с нулевого смещения (`offset = 0`, `mappedSize = -1`).
- `newFileSize` должен быть ≥ 1.

После `resize` все указатели, вычисленные из старого значения `f.mem`, **недействительны**. Пересчитайте все указатели из нового значения `f.mem`.

```nim
var f = memfiles.open("/tmp/grow.bin",
                      mode        = fmReadWrite,
                      newFileSize = 1024,
                      allowRemap  = true)
defer: f.close()

echo f.size  # 1024

f.flush()
f.resize(2048)
echo f.size  # 2048
# f.mem может теперь указывать на другой адрес — пересчитываем производные указатели
```

---

## Отображение подобластей

Эти две процедуры позволяют создавать дополнительные отображения в уже открытый `MemFile` — например, для скользящего доступа к разным частям большого файла. Они полезны только когда при `open` было передано `allowRemap = true`.

### `mapMem`

```nim
proc mapMem*(
  m:          var MemFile,
  mode:       FileMode = fmRead,
  mappedSize: int      = -1,
  offset:     int      = 0,
  mapFlags:   cint     = cint(-1)
): pointer
```

**Что делает:** Создаёт *дополнительное* отображение части файла `m` и возвращает сырой указатель на него. Это отображение независимо от того, что уже хранится в `m.mem`. При ошибке выбрасывает `OSError`. Режим `fmAppend` не поддерживается.

- `mappedSize = -1` — отображать весь файл (на POSIX требуется `mappedSize > 0`; на Windows `-1` означает «до конца файла»).
- `offset` должен быть выровнен по границе страницы.

Полученный указатель необходимо впоследствии передать в `unmapMem` с тем же значением `size`, иначе произойдёт утечка ресурсов ОС.

```nim
var f = memfiles.open("bigfile.bin", mode = fmRead, allowRemap = true)
defer: f.close()

# Отобразить 512 байт, начиная со второй страницы (байт 8192 при 4K-страницах)
let p = f.mapMem(mode = fmRead, mappedSize = 512, offset = 8192)
defer: f.unmapMem(p, 512)

let arr = cast[ptr UncheckedArray[byte]](p)
echo arr[0]  # первый байт второй страницы
```

---

### `unmapMem`

```nim
proc unmapMem*(f: var MemFile, p: pointer, size: int)
```

**Что делает:** Освобождает дополнительное отображение, созданное через `mapMem`. Записывает «грязные» страницы обратно в файловую систему, если отображение было доступно для записи. При ошибке выбрасывает `OSError`.

`size` **должен быть ровно тем же значением**, что было передано как `mappedSize` в `mapMem`. Несовпадающий размер приводит к неопределённому поведению (POSIX `munmap` либо отклонит вызов, либо уберёт не тот диапазон).

*(Пример использования — см. пример `mapMem` выше.)*

---

## Итерация по строкам и записям

Эти итераторы — один из самых быстрых способов обработки текстового файла строка за строкой в любом языке: они используют `memchr` для поиска разделителей без копирования данных.

### `memSlices`

```nim
iterator memSlices*(
  mfile: MemFile,
  delim: char = '\l',
  eat:   char = '\r'
): MemSlice {.inline.}
```

**Что делает:** Возвращает по одному `MemSlice` на каждую запись с разделителем в файле. Каждый срез — это сырая пара `(указатель, размер)`, ссылающаяся непосредственно в отображённую память файла — **никакого копирования не происходит**. Разделители в результирующий срез не включаются.

**Параметры:**
- `delim` — основной разделитель записей (по умолчанию: `'\l'` = `\n`). Каждое вхождение этого байта завершает запись.
- `eat` — необязательный символ, обрезаемый с конца записи, если он там присутствует (по умолчанию: `'\r'`). Это обрабатывает Windows-окончания `\r\n` побайтно: `\n` — разделитель, `\r` — «съедается». Передайте `eat = '\0'` для отключения обрезки и строго `delim`-разделения.

**Что обрабатывается автоматически:**
- Unix-окончания `\n`.
- Windows-окончания `\r\n` (по умолчанию `\r` «съедается»).
- Последняя строка без завершающего разделителя возвращается как обычная запись.
- Смешанные окончания строк внутри одного файла.

**Что НЕ обрабатывается:**
- Окончания только `\r` (старый Mac OS 9) — используйте `delim='\r', eat='\0'`.

**Безопасность:** Возвращаемый `MemSlice` не имеет проверки границ, нулевого терминатора и не является Nim-строкой. Используйте семантику C `mem*`-функций: читать можно `slice.size` байт начиная с `slice.data`, но не за их пределами.

```nim
import std/memfiles

var f = memfiles.open("data.txt")
defer: f.close()

var lineCount = 0
for slice in memSlices(f):
  # slice.data: указатель в тело файла
  # slice.size: количество байт в этой строке (без разделителя)
  inc lineCount

  # Безопасный доступ к первому символу:
  if slice.size > 0:
    let first = cast[cstring](slice.data)[0]
    if first != '#':  # пропустить строки-комментарии
      echo $slice     # преобразовать в Nim-строку только когда нужно
echo "Всего строк: ", lineCount
```

```nim
# Итерация по записям, разделённым нулевым байтом (например, вывод `find -print0`)
for slice in memSlices(f, delim = '\0', eat = '\0'):
  echo $slice
```

---

### `lines` (с буфером)

```nim
iterator lines*(
  mfile: MemFile,
  buf:   var string,
  delim: char = '\l',
  eat:   char = '\r'
): string {.inline.}
```

**Что делает:** Итерирует по строкам так же, как `memSlices`, но копирует каждую строку в переданную строку-буфер `buf` и возвращает её. Это исключает выделение новой строки на каждой итерации — один и тот же объект-строка переиспользуется каждый раз.

Это наиболее производительный вариант, когда вам нужны настоящие Nim-строки (а не сырые указатели `MemSlice`), и вы итерируете по большому количеству строк.

```nim
import std/memfiles

var f = memfiles.open("data.txt")
defer: f.close()

var buf = newStringOfCap(256)  # заранее выделить разумную ёмкость
for line in lines(f, buf):
  # `line` — тот же объект, что и `buf` — не сохраняйте его между итерациями!
  echo line
```

> **Важно:** `buf` изменяется на каждой итерации. Если вам нужно сохранить строку после выполнения тела цикла — скопируйте её: `let saved = line` или `let saved = buf`.

---

### `lines` (простой)

```nim
iterator lines*(
  mfile: MemFile,
  delim: char = '\l',
  eat:   char = '\r'
): string {.inline.}
```

**Что делает:** Удобный вариант `lines` — аргумент-буфер не нужен. Внутри выделяет и переиспользует один строковый буфер. Возвращает каждую строку как Nim-строку.

Используйте этот вариант, когда важна простота, а не микро-оптимизация выделений памяти.

```nim
import std/memfiles

var f = memfiles.open("log.txt")
defer: f.close()

for line in lines(f):
  echo line
```

---

## Утилиты MemSlice

### `==` (равенство)

```nim
proc `==`*(x, y: MemSlice): bool
```

**Что делает:** Возвращает `true`, если `x` и `y` имеют одинаковый `size` и одинаковое содержимое байт (сравнение через `equalMem`). Сравнение побайтовое, чувствительное к регистру, независимое от кодировки.

```nim
var f = memfiles.open("data.txt")
defer: f.close()

var slices: seq[MemSlice]
for s in memSlices(f):
  slices.add(s)

if slices.len >= 2:
  echo slices[0] == slices[1]  # сравнение по содержимому
```

---

### `$` (в строку)

```nim
proc `$`*(ms: MemSlice): string {.inline.}
```

**Что делает:** Копирует байты `ms` в новую Nim-строку и возвращает её. Это выход из мира нулевого копирования: когда нужно сохранить строку, передать её в API, ожидающий `string`, или использовать после закрытия `MemFile` — вызовите `$`.

```nim
var f = memfiles.open("names.txt")
defer: f.close()

var names: seq[string]
for slice in memSlices(f):
  names.add($slice)  # копируем в постоянную строку
# f теперь можно закрыть; names остаётся действительным
```

---

## `MemMapFileStream`

### `newMemMapFileStream`

```nim
proc newMemMapFileStream*(
  filename: string,
  mode:     FileMode = fmRead,
  fileSize: int      = -1
): MemMapFileStream
```

**Что делает:** Открывает файл как отображение в память и оборачивает его в `MemMapFileStream` — стандартный Nim `Stream`. После создания поддерживает все обычные операции с потоком: `readLine`, `writeLine`, `write`, `read`, `setPosition`, `getPosition`, `atEnd`, `flush`, `close` и т.д.

**Параметры:**
- `filename` — путь к файлу.
- `mode` — `fmRead` (по умолчанию) или `fmReadWrite`. Создание нового файла — только при `fmReadWrite` вместе с `fileSize`.
- `fileSize` — если > 0, создаёт новый файл такого размера. Допустимо только при `mode != fmRead`. Аналог `newFileSize` в `open`.

При невозможности открыть файл выбрасывает `OSError`.

**Когда использовать:** Когда вам нужна скорость отображения в память, но код написан под абстракцию `Stream` — например, для передачи в парсер, сериализатор или любую библиотеку, принимающую `Stream`.

```nim
import std/[memfiles, streams]

# Читаем файл через интерфейс Stream
var s = newMemMapFileStream("config.txt")
defer: s.close()

while not s.atEnd():
  echo s.readLine()
```

```nim
# Создаём новый файл и пишем в него через интерфейс Stream
var s = newMemMapFileStream("/tmp/out.bin", mode = fmReadWrite, fileSize = 1024)
defer: s.close()

s.write("Привет, мир с отображением в память!")
s.setPosition(0)
echo s.readLine()
```

---

## Платформенные особенности

**`fmAppend` не поддерживается.** Передача `mode = fmAppend` в `open` или `mapMem` вызывает `IOError`. Отображение в память не имеет понятия «указатель добавления» — позицию записи вы управляете вручную через `mem`.

**`offset` должен быть выровнен по границе страницы.** ОС отображает память блоками размером с страницу (обычно 4096 байт на x86, 16384 на Apple Silicon). Передача невыровненного `offset` приведёт к ошибке `OSError` в `open` или `mapMem`.

**`resize` требует `allowRemap = true`.** Если при `open` не передавалось `allowRemap = true`, дескриптор файла закрывается после отображения, и `resize` не сможет его переоткрыть. Вызов `resize` в этом случае выбрасывает `IOError`.

**После `resize` все указатели из старого отображения недействительны.** Это касается и любых значений, вычисленных как `cast[int](f.mem) + смещение`. Пересчитайте всё из нового значения `f.mem`.

**Дескрипторы Windows.** Поля `fHandle`, `mapHandle` и `wasOpened` существуют только на Windows. На POSIX есть только `handle`. Код, обращающийся к этим полям, должен использовать защиту `when defined(windows)` / `when defined(posix)`.

**Ограничения размера файла.** Поле `size: int` может переполниться на 32-битных платформах для файлов от 2 до 4 ГБ, поскольку там `int` 32-битный. На 64-битных платформах это не проблема.

**Оптимизация `mremap` на Linux.** На Linux `resize` использует `mremap` с флагом `MREMAP_MAYMOVE`, что означает расширение на месте (или рядом), и это драматически быстрее стратегии снятия + повторного создания отображения, применяемой на других POSIX-системах.

---

## Полные рабочие примеры

### Пример 1: Подсчёт строк в большом файле — максимально быстрый метод

```nim
import std/memfiles

proc countLines(path: string): int =
  var f = memfiles.open(path)
  defer: f.close()
  for _ in memSlices(f):
    inc result

echo countLines("/var/log/syslog")
```

---

### Пример 2: Чтение CSV-файла с пропуском строк-комментариев

```nim
import std/memfiles

proc parseCSV(path: string): seq[seq[string]] =
  var f = memfiles.open(path)
  defer: f.close()

  for slice in memSlices(f):
    if slice.size == 0: continue
    if cast[cstring](slice.data)[0] == '#': continue  # пропускаем комментарии

    let line = $slice
    result.add(line.split(','))

let rows = parseCSV("data.csv")
for row in rows:
  echo row
```

---

### Пример 3: Запись структурированных данных в новый отображённый файл

```nim
import std/memfiles

type Header = object
  magic:   uint32
  version: uint32
  count:   int64

var f = memfiles.open("/tmp/mydata.bin",
                      mode        = fmReadWrite,
                      newFileSize = 4096)
defer: f.close()

# Записываем заголовок в начало файла
let hdr = cast[ptr Header](f.mem)
hdr.magic   = 0xDEADBEEF'u32
hdr.version = 1'u32
hdr.count   = 42'i64

# Записываем сырые байты дальше
let body = cast[ptr UncheckedArray[byte]](
             cast[int](f.mem) + sizeof(Header))
for i in 0 ..< 16:
  body[i] = byte(i)

f.flush()
echo "Записано"
```

---

### Пример 4: Динамическое расширение журнального файла через `resize`

```nim
import std/memfiles

const PageSize = 4096

var f = memfiles.open("/tmp/growlog.bin",
                      mode        = fmReadWrite,
                      newFileSize = PageSize,
                      allowRemap  = true)
defer: f.close()

var writePos = 0

proc appendEntry(f: var MemFile, data: string, pos: var int) =
  let needed = pos + data.len
  if needed > f.size:
    f.flush()
    # Увеличиваем хотя бы на одну страницу
    let newSize = ((needed div PageSize) + 1) * PageSize
    f.resize(newSize)
    # f.mem может измениться после resize!

  let dst = cast[ptr UncheckedArray[byte]](cast[int](f.mem) + pos)
  for i, c in data:
    dst[i] = byte(c)
  inc(pos, data.len)

appendEntry(f, "Первая запись\n",  writePos)
appendEntry(f, "Вторая запись\n",  writePos)
echo "Размер журнала: ", f.size
```

---

### Пример 5: Использование `MemMapFileStream` с построчным парсером

```nim
import std/[memfiles, streams, strutils]

proc parseCfg(path: string): seq[(string, string)] =
  var s = newMemMapFileStream(path)
  defer: s.close()

  while not s.atEnd():
    let line = s.readLine().strip()
    if line.len == 0 or line[0] == ';': continue  # пропускаем пустые/комментарии
    let eq = line.find('=')
    if eq > 0:
      result.add((line[0 ..< eq].strip(), line[eq+1 .. ^1].strip()))

for (key, val) in parseCfg("settings.ini"):
  echo key, " = ", val
```

---

### Пример 6: Нестандартный разделитель — записи через нулевой байт

```nim
import std/memfiles

# Обрабатываем вывод `find /path -print0`
# Записи разделены нулевыми байтами; символ "съедания" не нужен.
var f = memfiles.open("find_output.bin")
defer: f.close()

for slice in memSlices(f, delim = '\0', eat = '\0'):
  if slice.size > 0:
    echo $slice  # каждый путь к файлу
```
