# Справочник модуля `syncio`

> **Назначение модуля:** `syncio` реализует синхронный буферизованный файловый ввод-вывод — каждая операция блокирует выполнение до своего завершения. Это тонкая, высокоточная обёртка над стандартной библиотекой C `<stdio.h>`, представленная через идиоматичный API Nim. Модуль охватывает открытие и закрытие файлов, чтение и запись данных в различных формах (байты, символы, строки, типизированные значения), навигацию внутри файла, запрос метаданных файла и управление стандартными потоками (`stdin`, `stdout`, `stderr`).

---

## Содержание

1. [Типы](#типы)
   - [File](#file)
   - [FileMode](#filemode)
   - [FileSeekPos](#fileseekpos)
   - [FileHandle](#filehandle)
2. [Стандартные потоки](#стандартные-потоки)
   - [stdin, stdout, stderr](#stdin-stdout-stderr)
   - [stdmsg](#stdmsg)
3. [Открытие и закрытие](#открытие-и-закрытие)
   - [open (имя файла)](#open-имя-файла)
   - [open (возвращающий File)](#open-возвращающий-file)
   - [open (FileHandle)](#open-filehandle)
   - [reopen](#reopen)
   - [close](#close)
4. [Высокоуровневые помощники для файлов](#высокоуровневые-помощники-для-файлов)
   - [readFile](#readfile)
   - [writeFile (строка)](#writefile-строка)
   - [writeFile (байты)](#writefile-байты)
   - [readLines](#readlines)
5. [Чтение](#чтение)
   - [readAll](#readall)
   - [readLine (мутирующий)](#readline-мутирующий)
   - [readLine (возвращающий)](#readline-возвращающий)
   - [readChar](#readchar)
   - [readBuffer](#readbuffer)
   - [readBytes](#readbytes)
   - [readChars](#readchars)
6. [Запись](#запись)
   - [Семейство перегрузок write](#семейство-перегрузок-write)
   - [writeLine](#writeline)
   - [writeBuffer](#writebuffer)
   - [writeBytes](#writebytes)
   - [writeChars](#writechars)
   - [Оператор &=](#оператор-)
7. [Итерация](#итерация)
   - [lines (имя файла)](#lines-имя-файла)
   - [lines (File)](#lines-file)
8. [Позиция и размер](#позиция-и-размер)
   - [getFilePos](#getfilepos)
   - [setFilePos](#setfilepos)
   - [getFileSize](#getfilesize)
   - [endOfFile](#endoffile)
9. [Дескрипторы и наследование](#дескрипторы-и-наследование)
   - [getFileHandle](#getfilehandle)
   - [getOsFileHandle](#getosfilehandle)
   - [setInheritable](#setinheritable)
10. [Управление буфером](#управление-буфером)
    - [flushFile](#flushfile)
    - [setStdIoUnbuffered](#setstdiounbuffered)
11. [Сводная таблица поведения при ошибках](#сводная-таблица-поведения-при-ошибках)
12. [Практические паттерны](#практические-паттерны)

---

## Типы

### `File`

```nim
type File* = ptr CFile
```

Дескриптор открытого файла. Это указатель на непрозрачную структуру C `FILE`, поэтому он равен `nil`, когда не инициализирован или после неудачного открытия. Большинство процедур модуля принимают `File` в качестве первого аргумента.

Поскольку `File` является указателем, присваивание копирует только дескриптор — обе переменные затем ссылаются на один и тот же файл ОС. Всегда закрывайте ровно один раз.

---

### `FileMode`

```nim
type FileMode* = enum
  fmRead, fmWrite, fmReadWrite, fmReadWriteExisting, fmAppend
```

Управляет тем, как открывается файл. Пять режимов охватывают все стандартные комбинации POSIX/C:

| Режим | Файл должен существовать? | Очищает содержимое? | Чтение? | Запись? |
|---|---|---|---|---|
| `fmRead` | Да | Нет | ✅ | ❌ |
| `fmWrite` | Нет (создаёт) | **Да** | ❌ | ✅ |
| `fmReadWrite` | Нет (создаёт) | **Да** | ✅ | ✅ |
| `fmReadWriteExisting` | Да | Нет | ✅ | ✅ |
| `fmAppend` | Нет (создаёт) | Нет | ❌ | ✅ (только в конец) |

> **Предупреждение:** `fmWrite` и `fmReadWrite` молча уничтожают содержимое любого существующего файла. Используйте `fmReadWriteExisting` или `fmAppend` для сохранения данных.

---

### `FileSeekPos`

```nim
type FileSeekPos* = enum
  fspSet,   ## абсолютная позиция от начала файла
  fspCur,   ## позиция относительно текущей
  fspEnd    ## позиция относительно конца файла
```

Используется в качестве аргумента `relativeTo` функции `setFilePos`. Напрямую отображается на POSIX `SEEK_SET`, `SEEK_CUR` и `SEEK_END`. Отрицательные смещения с `fspEnd` указывают назад от конца файла.

---

### `FileHandle`

```nim
# POSIX:
type FileHandle* = cint
# Windows:
type FileHandle* = int   # HANDLE, конвертируется в winlean.Handle
```

Низкоуровневый числовой дескриптор файла ОС (POSIX) или Windows HANDLE. Используется при взаимодействии с API уровня ОС. Не то же самое, что `File` — `FileHandle` является просто числом, а не буферизованным потоком.

---

## Стандартные потоки

### `stdin`, `stdout`, `stderr`

```nim
var stdin*:  File   ## стандартный ввод
var stdout*: File   ## стандартный вывод
var stderr*: File   ## стандартный поток ошибок
```

Предварительно открытые значения `File`, указывающие на стандартные потоки процесса, импортированные напрямую из C `<stdio.h>`. На Windows модуль дополнительно переключает все три потока в бинарный режим при запуске, а консольные приложения настраиваются на использование UTF-8 (кодовая страница 65001).

Они могут передаваться туда, где ожидается `File`:

```nim
import std/syncio

write(stdout, "Привет, мир\n")
let line = readLine(stdin)
write(stderr, "Предупреждение: " & line & "\n")
```

---

### `stdmsg`

```nim
template stdmsg*: File
```

По умолчанию раскрывается в `stderr`, или в `stdout`, если программа скомпилирована с `-d:useStdoutAsStdmsg`. Используется внутри среды выполнения для диагностических сообщений. Полезен, когда нужен единый символ, который пользователи могут перенаправить на этапе компиляции.

```nim
import std/syncio

write(stdmsg, "Это идёт в stderr, если не задан -d:useStdoutAsStdmsg\n")
```

---

## Открытие и закрытие

### `open` (имя файла)

```nim
proc open*(f: var File, filename: string,
           mode: FileMode = fmRead,
           bufSize: int = -1): bool
```

Пытается открыть файл по пути `filename`. При успехе возвращает `true` и заполняет `f`; при неудаче возвращает `false` без выброса исключения. Это перегрузка без исключений — используйте её, когда хотите самостоятельно обрабатывать ошибки открытия.

`bufSize` управляет буфером ввода-вывода уровня C:
- `-1` (по умолчанию): использовать размер буфера по умолчанию системы
- `0`: отключить буферизацию (небуферизованный ввод-вывод, каждый байт немедленно передаётся ОС)
- `> 0`: использовать буфер именно такого размера в байтах

Полученный дескриптор файла помечается как не наследуемый дочерними процессами (если не задан флаг `-d:nimInheritHandles`).

```nim
import std/syncio

var f: File
if open(f, "data.txt"):
  let content = readAll(f)
  close(f)
  echo content
else:
  echo "Не удалось открыть data.txt"
```

---

### `open` (возвращающий `File`)

```nim
proc open*(filename: string,
           mode: FileMode = fmRead,
           bufSize: int = -1): File
```

Открывает файл и **возвращает** дескриптор `File` напрямую. Бросает `IOError`, если файл не может быть открыт. Это перегрузка с исключением — она более лаконична, когда вы знаете, что файл должен существовать, и исключение при неудаче является правильным поведением.

```nim
import std/syncio

let f = open("config.toml")   # бросает IOError, если файл отсутствует
defer: close(f)
echo readAll(f)
```

---

### `open` (FileHandle)

```nim
proc open*(f: var File, filehandle: FileHandle,
           mode: FileMode = fmRead): bool
```

Оборачивает существующий дескриптор файла ОС (полученный от системного вызова, `pipe` или `socket`) в буферизованный `File`. Возвращает `true` при успехе. Дескриптор помечается как не наследуемый после обёртывания. Полезен для интеграции низкоуровневых дескрипторов ОС в буферизованный слой ввода-вывода Nim.

```nim
import std/syncio

# Пример: обернуть файловый дескриптор 3 (уже открытый родительским процессом)
var f: File
if open(f, FileHandle(3)):
  echo readLine(f)
  close(f)
```

---

### `reopen`

```nim
proc reopen*(f: File, filename: string, mode: FileMode = fmRead): bool
```

Перенаправляет уже открытый `File` на другой файл на диске без отдельного закрытия и открытия. Классический сценарий использования — перенаправление `stdin`, `stdout` или `stderr` в файл, что является идиомой POSIX/C для перенаправления ввода-вывода внутри процесса.

```nim
import std/syncio

# Перенаправить stdout в лог-файл для всего последующего вывода
if reopen(stdout, "app.log", fmWrite):
  echo "Это идёт в app.log, а не в терминал"
```

---

### `close`

```nim
proc close*(f: File)
```

Закрывает файл, сбрасывая все буферизованные данные в ОС. После `close` дескриптор `File` недействителен и не должен использоваться снова.

При компиляции с `-d:nimPreviewCheckedClose` функция `close` бросает `IOError` при неудаче (например, если сброс буфера записи завершился ошибкой). Без этого флага ошибка молча игнорируется — поведение по умолчанию, безопасное для большинства случаев.

```nim
import std/syncio

var f: File
if open(f, "output.txt", fmWrite):
  write(f, "Привет")
  close(f)   # сбрасывает буфер и освобождает дескриптор
```

---

## Высокоуровневые помощники для файлов

Эти процедуры открывают файл, работают с ним и закрывают его в одном вызове — наиболее удобный способ решения простых файловых задач.

### `readFile`

```nim
proc readFile*(filename: string): string
```

Читает всё содержимое файла по пути `filename` и возвращает его как `string`. Открывает файл, читает всё, закрывает. Бросает `IOError`, если файл не может быть открыт или прочитан. Доступно во время компиляции через `staticRead`.

```nim
import std/syncio

let src = readFile("main.nim")
echo src.len, " байт"
```

---

### `writeFile` (строка)

```nim
proc writeFile*(filename, content: string)
```

Создаёт или перезаписывает файл по пути `filename` содержимым `content`. Открывает с режимом `fmWrite` (уничтожая предыдущее содержимое), записывает всю строку, затем закрывает. Бросает `IOError` при неудаче.

```nim
import std/syncio

writeFile("greeting.txt", "Привет, мир!")
```

---

### `writeFile` (байты)

```nim
proc writeFile*(filename: string, content: openArray[byte])
```

То же, что строковая перегрузка, но принимает сырые байты. Полезно при записи бинарных данных, которые не должны интерпретироваться как текст.

```nim
import std/syncio

let data: seq[byte] = @[0xFF'u8, 0xFE, 0x00, 0x41]
writeFile("bom.bin", data)
```

---

### `readLines`

```nim
proc readLines*(filename: string, n: Natural): seq[string]
```

Читает ровно `n` строк из файла по пути `filename` и возвращает их как `seq[string]`. Символы новой строки удаляются. Бросает `IOError`, если файл не может быть открыт, или `EOFError`, если он содержит меньше `n` строк.

```nim
import std/syncio

# Прочитать двухстрочный заголовок
let header = readLines("data.csv", 2)
echo header[0]   # первая строка
echo header[1]   # вторая строка
```

---

## Чтение

### `readAll`

```nim
proc readAll*(file: File): string
```

Читает всё от текущей позиции до конца `file` и возвращает как `string`. Для обычных файлов сначала считывается размер файла, что позволяет сделать единственное выделение памяти. Для потоков неизвестной длины (например, `stdin`) читает по частям. Бросает `IOError` при ошибке.

Вызывать `readAll`, когда позиция файла не находится в начале, является ошибкой — при необходимости предварительно используйте `setFilePos`.

```nim
import std/syncio

let f = open("big.log")
defer: close(f)
let contents = readAll(f)
echo contents.len, " байт прочитано"
```

---

### `readLine` (мутирующий)

```nim
proc readLine*(f: File, line: var string): bool
```

Читает одну строку текста из `f` в `line`, удаляя завершающий `LF` или `CRLF`. Возвращает `true`, если строка успешно прочитана, `false` если достигнут конец файла (в этом случае `line` не изменяется).

Эта перегрузка повторно использует существующее выделение памяти строки `line`, что делает её эффективной в плотных циклах.

На Windows при чтении из интерактивной консоли `Ctrl+Z` трактуется как EOF.

```nim
import std/syncio

let f = open("names.txt")
defer: close(f)

var line: string
while readLine(f, line):
  echo "Имя: ", line
```

---

### `readLine` (возвращающий)

```nim
proc readLine*(f: File): string
```

Читает одну строку из `f` и возвращает её. Бросает `EOFError`, если конец файла уже достигнут до вызова. Используйте мутирующую перегрузку в производительно-чувствительных циклах; используйте эту для ясности при чтении небольшого числа строк.

```nim
import std/syncio

let f = open("config.txt")
defer: close(f)

let firstLine = readLine(f)
echo "Первая строка: ", firstLine
```

---

### `readChar`

```nim
proc readChar*(f: File): char
```

Читает один байт из `f` и возвращает его как `char`. Бросает `EOFError` в конце файла. Не подходит для производительно-чувствительного кода — используйте `readBuffer` или `readAll` при чтении большого количества байт.

```nim
import std/syncio

let f = open("tiny.bin")
defer: close(f)

let first = readChar(f)
echo "Первый байт: ", ord(first)
```

---

### `readBuffer`

```nim
proc readBuffer*(f: File, buffer: pointer, len: Natural): int
```

Низкоуровневая операция чтения. Читает до `len` байт из `f` в память, на которую указывает `buffer`. Возвращает фактическое количество прочитанных байт, которое может быть меньше `len` в конце файла, но никогда не больше. Использует `fread` напрямую.

```nim
import std/syncio

var buf: array[1024, byte]
let f = open("raw.bin")
defer: close(f)

let n = readBuffer(f, addr buf[0], buf.len)
echo n, " байт прочитано"
```

---

### `readBytes`

```nim
proc readBytes*(f: File, a: var openArray[int8|uint8], start, len: Natural): int
```

Читает `len` байт из `f` в срез `a[start ..< start+len]`. Возвращает фактическое количество прочитанных байт. Типизированная обёртка над `readBuffer` для массивов `int8`/`uint8`.

```nim
import std/syncio

var buf = newSeq[uint8](256)
let f = open("data.bin")
defer: close(f)

let n = readBytes(f, buf, 0, 128)
echo n, " байт прочитано в первые 128 слотов"
```

---

### `readChars`

```nim
proc readChars*(f: File, a: var openArray[char]): int
```

Читает до `a.len` байт из `f` в буфер `char` `a`. Возвращает фактическое количество прочитанных байт. Эквивалентно `readBuffer`, но типизировано для символьных массивов.

```nim
import std/syncio

var buf: array[64, char]
let f = open("text.txt")
defer: close(f)

let n = readChars(f, buf)
echo n, " символов прочитано"
```

---

## Запись

### Семейство перегрузок `write`

```nim
proc write*(f: File, s: string)
proc write*(f: File, c: cstring)
proc write*(f: File, c: char)
proc write*(f: File, i: int)
proc write*(f: File, i: BiggestInt)
proc write*(f: File, r: float32)
proc write*(f: File, r: BiggestFloat)
proc write*(f: File, b: bool)
proc write*(f: File, a: varargs[string, `$`])
```

Записывает одно значение — или любое количество значений через перегрузку `varargs` — в `f`. Все перегрузки бросают `IOError` при неудаче. Перегрузка `varargs` предварительно преобразует каждый аргумент через `$`, делая её пригодной для любого типа, у которого определён `$`.

На Windows строковый вывод маршрутизируется через `fprintf` для корректной обработки UTF-8 контента, включая встроенные нулевые символы.

```nim
import std/syncio

let f = open("out.txt", fmWrite)
defer: close(f)

write(f, "Счётчик: ")
write(f, 42)
write(f, "\n")
write(f, true)
write(f, "\n")

# Перегрузка varargs — всё печатается последовательно
write(f, "x=", 3.14, " готово\n")
```

---

### `writeLine`

```nim
proc writeLine*[Ty](f: File, x: varargs[Ty, `$`])
```

Записывает каждый элемент `x` (преобразуя через `$`) в `f`, затем записывает один символ `"\n"`. Эквивалентно вызову `write` с последующим `write(f, "\n")`, но более читаемо при выводе полных строк.

```nim
import std/syncio

let f = open("log.txt", fmWrite)
defer: close(f)

writeLine(f, "Запуск")
writeLine(f, "Значение: ", 99)
writeLine(f, "Завершено")
```

---

### `writeBuffer`

```nim
proc writeBuffer*(f: File, buffer: pointer, len: Natural): int
```

Записывает ровно `len` байт из памяти, на которую указывает `buffer`, в `f`. Возвращает фактическое количество записанных байт, которое может быть меньше `len` при ошибке. Самый низкоуровневый метод записи — используйте его при работе с сырой памятью.

```nim
import std/syncio

var data = [0x48'u8, 0x65, 0x6C, 0x6C, 0x6F]  # "Hello"
let f = open("raw.bin", fmWrite)
defer: close(f)

let written = writeBuffer(f, addr data[0], data.len)
assert written == 5
```

---

### `writeBytes`

```nim
proc writeBytes*(f: File, a: openArray[int8|uint8], start, len: Natural): int
```

Записывает `a[start ..< start+len]` в `f`. Возвращает количество записанных байт. Типизированная обёртка над `writeBuffer` для массивов `int8`/`uint8`.

```nim
import std/syncio

let payload: seq[uint8] = @[1'u8, 2, 3, 4, 5]
let f = open("payload.bin", fmWrite)
defer: close(f)

discard writeBytes(f, payload, 1, 3)  # записывает байты [2, 3, 4]
```

---

### `writeChars`

```nim
proc writeChars*(f: File, a: openArray[char], start, len: Natural): int
```

Записывает `a[start ..< start+len]` в `f`. Возвращает количество записанных байт. Идентично `writeBytes`, но типизировано для массивов `char` — полезно при работе с фиксированными символьными буферами.

```nim
import std/syncio

var buf = ['П', 'р', 'и', 'в', 'е', 'т']
let f = open("chars.txt", fmWrite)
defer: close(f)

discard writeChars(f, buf, 0, 6)
```

---

### Оператор `&=`

```nim
template `&=`*(f: File, x: typed)
```

Краткое обозначение для `write(f, x)`. Позволяет использовать знакомый оператор `&=` на дескрипторе `File`, что естественно читается при инкрементальном построении вывода.

```nim
import std/syncio

let f = open("result.txt", fmWrite)
defer: close(f)

f &= "Привет"
f &= ", "
f &= "мир!"
```

---

## Итерация

### `lines` (имя файла)

```nim
iterator lines*(filename: string): string
```

Открывает файл по пути `filename`, итерирует по каждой строке (удаляя символы новой строки), затем автоматически закрывает файл. Бросает `IOError`, если файл не может быть открыт. Внутренний буфер чтения составляет 8 000 байт, что обеспечивает эффективную построчную обработку больших файлов.

```nim
import std/syncio

var count = 0
for line in lines("access.log"):
  if "ОШИБКА" in line:
    inc count
echo count, " строк с ошибками"
```

---

### `lines` (File)

```nim
iterator lines*(f: File): string
```

То же, что перегрузка с именем файла, но работает на уже открытом `File`. Не закрывает `f` по завершении итерации — ответственность за закрытие лежит на вызывающем коде. Полезно, когда нужно выполнить другие операции с `f` до или после итерации.

```nim
import std/syncio

let f = open("data.csv")
defer: close(f)

# Пропустить заголовок
discard readLine(f)

# Итерировать по строкам данных
for row in f.lines:
  echo row
```

---

## Позиция и размер

### `getFilePos`

```nim
proc getFilePos*(f: File): int64
```

Возвращает текущую позицию чтения/записи в байтах от начала файла (с нуля). Бросает `IOError`, если позиция не может быть получена (например, для непозиционируемых потоков, таких как каналы).

```nim
import std/syncio

let f = open("data.bin")
defer: close(f)

discard readChar(f)
echo getFilePos(f)   # 1 — один байт после начала
```

---

### `setFilePos`

```nim
proc setFilePos*(f: File, pos: int64, relativeTo: FileSeekPos = fspSet)
```

Перемещает указатель файла на `pos` байт относительно `relativeTo`. Бросает `IOError` при неудаче. После успешного `setFilePos` следующая операция чтения или записи произойдёт в новой позиции.

```nim
import std/syncio

let f = open("data.bin")
defer: close(f)

setFilePos(f, 0, fspEnd)          # прыжок в конец
let size = getFilePos(f)

setFilePos(f, -4, fspEnd)         # 4 байта до конца
let tail = readAll(f)

setFilePos(f, 0)                  # обратно в начало (fspSet по умолчанию)
```

---

### `getFileSize`

```nim
proc getFileSize*(f: File): int64
```

Возвращает общий размер `f` в байтах. Внутренне перемещается в конец, считывает позицию, затем возвращается назад — временно перемещая указатель файла. Исходная позиция восстанавливается перед возвратом.

```nim
import std/syncio

let f = open("image.png")
defer: close(f)

echo getFileSize(f), " байт"
```

---

### `endOfFile`

```nim
proc endOfFile*(f: File): bool
```

Возвращает `true`, если позиция чтения `f` находится в конце файла. Внутренне читает один байт и возвращает его обратно через `ungetc`, поэтому позиция файла не изменяется. Несколько дешевле, чем `getFilePos == getFileSize` для этой проверки.

```nim
import std/syncio

let f = open("stream.bin")
defer: close(f)

while not endOfFile(f):
  let ch = readChar(f)
  process(ch)
```

---

## Дескрипторы и наследование

### `getFileHandle`

```nim
proc getFileHandle*(f: File): FileHandle
```

Возвращает числовой дескриптор файла библиотеки C для `f` (целое число, передаваемое в `fileno` / `_fileno` на Windows). На Windows это **не** Win32 `HANDLE` — для этого используйте `getOsFileHandle`.

```nim
import std/syncio

let f = open("test.txt")
defer: close(f)

echo "fd = ", getFileHandle(f)   # например, 3
```

---

### `getOsFileHandle`

```nim
proc getOsFileHandle*(f: File): FileHandle
```

Возвращает настоящий дескриптор уровня ОС: на POSIX это то же целое число, что и `getFileHandle`; на Windows это фактический Win32 `HANDLE`, полученный из `_get_osfhandle`. Используйте его при вызове API ОС, требующих сырого дескриптора.

```nim
import std/syncio

let f = open("test.txt")
defer: close(f)

let handle = getOsFileHandle(f)
# передайте `handle` в Win32 API или системный вызов POSIX
```

---

### `setInheritable`

```nim
proc setInheritable*(f: FileHandle, inheritable: bool): bool
```

Управляет тем, наследуется ли дескриптор файла ОС `f` дочерними процессами при вызове `fork`/`exec` (POSIX) или `CreateProcess` (Windows). Возвращает `true` при успехе.

По умолчанию все файлы, открытые через `syncio`, уже помечены как не наследуемые. Используйте `setInheritable(handle, true)` только когда специально нужно, чтобы дочерний процесс унаследовал открытый файл.

Не гарантированно доступно на всех платформах — проверяйте через `declared(setInheritable)`.

```nim
import std/syncio

let f = open("shared.txt")
defer: close(f)

# Сделать файл наследуемым перед запуском дочернего процесса
if declared(setInheritable):
  discard setInheritable(getOsFileHandle(f), true)
```

---

## Управление буфером

### `flushFile`

```nim
proc flushFile*(f: File)
```

Принудительно отправляет все данные из буфера записи `f` в ОС. Это **не** гарантирует запись данных на физический носитель — для этого используйте `fsync` уровня ОС. Полезно перед критически важной операцией, перед чтением только что записанного файла или при передаче вывода другому процессу, которому он нужен немедленно.

```nim
import std/syncio

write(stdout, "Введите ваше имя: ")
flushFile(stdout)   # убедиться, что подсказка появилась до блокировки на вводе
let name = readLine(stdin)
```

---

### `setStdIoUnbuffered`

```nim
proc setStdIoUnbuffered*()
```

Отключает буферы уровня C для `stdin`, `stdout` и `stderr` (`setvbuf(f, nil, _IONBF, 0)`). После этого вызова каждая операция `write` идёт напрямую в ОС без накопления в буфере. Полезно для журналирования в реальном времени, интерактивных инструментов или когда вывод должен быть немедленно виден мониторинговому процессу.

Обратите внимание, что небуферизованный ввод-вывод значительно медленнее при больших объёмах данных — отключайте буферизацию только когда задержка важнее пропускной способности.

```nim
import std/syncio

setStdIoUnbuffered()   # все записи в stdout/stderr теперь немедленны
echo "Строка 1"        # появляется сразу, даже если stdout перенаправлен
echo "Строка 2"
```

---

## Сводная таблица поведения при ошибках

| Операция | При неудаче |
|---|---|
| `open(f, filename, ...)` | Возвращает `false`, исключений нет |
| `open(filename, ...)` → `File` | Бросает `IOError` |
| `reopen` | Возвращает `false` |
| `close` (по умолчанию) | Молча игнорирует ошибки |
| `close` (`-d:nimPreviewCheckedClose`) | Бросает `IOError` |
| `readLine(f, line)` → `bool` | Возвращает `false` в конце файла; бросает `IOError` при ошибке |
| `readLine(f)` → `string` | Бросает `EOFError` в конце файла; бросает `IOError` при ошибке |
| `readChar` | Бросает `EOFError` в конце файла; бросает `IOError` при ошибке |
| `readAll`, `readFile`, `readLines` | Бросают `IOError` |
| `write*`, `writeLine`, `writeFile` | Бросают `IOError` |
| `setFilePos`, `getFilePos` | Бросают `IOError` |

---

## Практические паттерны

### Паттерн 1 — Безопасное открытие с `defer`

```nim
import std/syncio

let f = open("data.txt")   # бросает IOError, если файл отсутствует
defer: close(f)            # гарантированно закроет, даже при исключении

for line in f.lines:
  echo line
```

---

### Паттерн 2 — Открытие без исключений с ручной обработкой ошибок

```nim
import std/syncio

var f: File
if not open(f, "optional.cfg"):
  echo "Конфигурация не найдена, используются настройки по умолчанию"
else:
  defer: close(f)
  echo readAll(f)
```

---

### Паттерн 3 — Дозапись в лог-файл

```nim
import std/syncio
import std/times

proc logEvent(filename, message: string) =
  var f: File
  if open(f, filename, fmAppend):
    defer: close(f)
    writeLine(f, $now(), " ", message)

logEvent("events.log", "Сервер запущен")
logEvent("events.log", "Соединение принято")
```

---

### Паттерн 4 — Чтение бинарного файла по частям

```nim
import std/syncio

proc processChunks(filename: string, chunkSize = 4096) =
  let f = open(filename)
  defer: close(f)

  var buf = newSeq[byte](chunkSize)
  while true:
    let n = readBuffer(f, addr buf[0], chunkSize)
    if n == 0: break
    process(buf.toOpenArray(0, n - 1))
```

---

### Паттерн 5 — Позиционирование для чтения хвоста файла

```nim
import std/syncio

proc tail(filename: string, bytes: int64): string =
  let f = open(filename)
  defer: close(f)

  let size = getFileSize(f)
  let offset = if size < bytes: 0'i64 else: size - bytes
  setFilePos(f, offset)
  result = readAll(f)

echo tail("big.log", 512)   # последние 512 байт
```

---

### Паттерн 6 — Перенаправление stdout в файл и обратно

```nim
import std/syncio

# Временно перенаправить stdout
if reopen(stdout, "capture.txt", fmWrite):
  echo "Это идёт в capture.txt"
  flushFile(stdout)
  discard reopen(stdout, "/dev/tty", fmWrite)   # восстановить (POSIX)

echo "Это снова идёт в терминал"
```
