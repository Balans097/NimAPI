# Справочник модуля `std/streams` (Nim)

> Модуль предоставляет **интерфейс потока** и две реализации:
> - `StringStream` — поток поверх строки в памяти
> - `FileStream` — поток поверх файла (`File`)
>
> Базовый порядок работы: **открыть → читать/писать → закрыть**.
>
> ⚠️ `readData`, `peekData`, `writeData` недоступны на VM-бэкенде и требуют `cast[ptr string]` на JS. Используйте `readDataStr` как универсальную замену.

---

## Содержание

1. [Типы](#типы)
2. [Создание потоков](#создание-потоков)
3. [Управление потоком](#управление-потоком)
4. [Позиционирование](#позиционирование)
5. [Запись данных](#запись-данных)
6. [Низкоуровневое чтение и запись](#низкоуровневое-чтение-и-запись)
7. [Чтение типизированных значений](#чтение-типизированных-значений)
8. [Просмотр без сдвига (peek)](#просмотр-без-сдвига-peek)
9. [Чтение строк](#чтение-строк)
10. [Итератор](#итератор)
11. [Таблица соответствий read/peek](#таблица-соответствий-readpeek)

---

## Типы

### `Stream`

```nim
type Stream* = ref StreamObj
```

Основной тип модуля. Все процедуры принимают `Stream`. Является ссылкой на `StreamObj`.

---

### `StreamObj`

```nim
type StreamObj* = object of RootObj
```

Объект потока с полями-функциями, через которые реализуется интерфейс. **Не использовать напрямую** — эти поля предназначены для реализаций потоков.

| Поле | Тип | Назначение |
|---|---|---|
| `closeImpl` | `proc(s: Stream)` | Закрыть поток |
| `atEndImpl` | `proc(s: Stream): bool` | Проверить конец |
| `setPositionImpl` | `proc(s: Stream, pos: int)` | Установить позицию |
| `getPositionImpl` | `proc(s: Stream): int` | Получить позицию |
| `readDataStrImpl` | `proc(s, buffer, slice): int` | Читать в строку |
| `readLineImpl` | `proc(s, line): bool` | Читать строку |
| `readDataImpl` | `proc(s, buffer, bufLen): int` | Читать в буфер |
| `peekDataImpl` | `proc(s, buffer, bufLen): int` | Смотреть без сдвига |
| `writeDataImpl` | `proc(s, buffer, bufLen)` | Писать из буфера |
| `flushImpl` | `proc(s: Stream)` | Сбросить буферы |

---

### `StringStream`

```nim
type StringStream* = ref StringStreamObj
```

Поток, инкапсулирующий строку в памяти. Поле `data: string` публично — строку можно читать напрямую после записи.

---

### `StringStreamObj`

```nim
type StringStreamObj* = object of StreamObj
  data*: string  ## Текущее содержимое потока
```

---

### `FileStream`

```nim
type FileStream* = ref FileStreamObj
```

Поток, инкапсулирующий файл `File`. **Недоступен на JS-бэкенде.**

---

### `FileStreamObj`

```nim
type FileStreamObj* = object of Stream
```

Объект файлового потока. Содержит непубличное поле `f: File`.

---

## Создание потоков

### `newStringStream(s: string = ""): owned StringStream`

Создаёт новый поток из строки `s`. Строка копируется внутрь потока. Если `s` не указана — создаётся пустой поток для записи.

```nim
import std/streams

# Пустой поток для записи
var ws = newStringStream()
ws.write("hello")
ws.setPosition(0)
echo ws.readAll()  # "hello"

# Поток для чтения из готовой строки
var rs = newStringStream("line one\nline two")
echo rs.readLine()  # "line one"
rs.close()
```

---

### `newFileStream(f: File): owned FileStream`

Создаёт поток из уже открытого объекта `File`. Управление файлом (открытие/закрытие) остаётся за вызывающим кодом до вызова `close`.

> **Недоступен на JS-бэкенде.**

```nim
import std/streams

var f: File
if open(f, "data.bin", fmRead):
  var strm = newFileStream(f)
  echo strm.readLine()
  strm.close()
```

---

### `newFileStream(filename: string, mode: FileMode = fmRead, bufSize: int = -1): owned FileStream`

Создаёт поток из файла по имени. Если файл не удаётся открыть — возвращает **`nil`** (исключение не бросается).

> ⚠️ **Всегда проверяйте возвращаемое значение на `nil`.** Для гарантированного исключения при ошибке используйте `openFileStream`.

> **Недоступен на JS-бэкенде.**

| Параметр | Описание |
|---|---|
| `filename` | Путь к файлу |
| `mode` | `fmRead`, `fmWrite`, `fmReadWrite`, `fmAppend` и т.д. |
| `bufSize` | Размер буфера (`-1` — системный по умолчанию) |

```nim
import std/streams

var strm = newFileStream("log.txt", fmWrite)
if not isNil(strm):
  strm.writeLine("Started")
  strm.close()
else:
  echo "Не удалось открыть файл"
```

---

### `openFileStream(filename: string, mode: FileMode = fmRead, bufSize: int = -1): owned FileStream`

Создаёт поток из файла по имени. Если файл не удаётся открыть — **бросает `IOError`**.

> **Недоступен на JS-бэкенде.**

```nim
import std/streams

try:
  var strm = openFileStream("config.txt")
  echo strm.readLine()
  strm.close()
except IOError as e:
  echo "Ошибка: ", e.msg
```

---

## Управление потоком

### `close(s: Stream)`

Закрывает поток. Для `FileStream` также закрывает файл. Безопасно вызывать, даже если `s` — `nil`.

```nim
import std/streams

let strm = newStringStream("data")
defer: strm.close()
echo strm.readLine()
```

---

### `flush(s: Stream)`

Сбрасывает буферы потока на физический носитель. Для `StringStream` ничего не делает. Особенно важно для `FileStream` при записи.

```nim
import std/streams

var strm = newFileStream("out.txt", fmWrite)
strm.write("hello")
strm.flush()  # Данные гарантированно записаны в файл
strm.write("world")
strm.close()  # close тоже сбрасывает буферы
```

---

### `atEnd(s: Stream): bool`

Возвращает `true`, если все данные прочитаны (достигнут конец потока).

```nim
import std/streams

var strm = newStringStream("abc")
echo strm.atEnd()  # false
discard strm.readAll()
echo strm.atEnd()  # true
strm.close()
```

---

## Позиционирование

### `setPosition(s: Stream, pos: int)`

Устанавливает текущую позицию чтения/записи в байтах от начала потока. Для `StringStream` позиция зажата в `[0, data.len]`.

```nim
import std/streams

var strm = newStringStream("Hello, world!")
strm.setPosition(7)
echo strm.readStr(5)  # "world"
strm.setPosition(0)
echo strm.readStr(5)  # "Hello"
strm.close()
```

---

### `getPosition(s: Stream): int`

Возвращает текущую байтовую позицию в потоке.

```nim
import std/streams

var strm = newStringStream("Hello\nWorld")
echo strm.getPosition()     # 0
discard strm.readLine()
echo strm.getPosition()     # 6
strm.close()
```

---

## Запись данных

### `write[T](s: Stream, x: T)`

Обобщённая процедура записи. Записывает двоичное представление значения `x` в поток. Размер записанных данных равен `sizeof(x)`.

> **Недоступен на JS-бэкенде.** Используйте `write(Stream, string)`.

```nim
import std/streams

var strm = newStringStream()
strm.write(42'i32)    # записывает 4 байта
strm.write(3.14'f64)  # записывает 8 байт
strm.setPosition(0)
echo strm.readInt32()    # 42
echo strm.readFloat64()  # 3.14
strm.close()
```

---

### `write(s: Stream, x: string)`

Записывает строку `x` в поток. Терминирующий ноль и длина **не записываются**.

```nim
import std/streams

var strm = newStringStream()
strm.write("Hello")
strm.write(", ")
strm.write("World!")
strm.setPosition(0)
echo strm.readAll()  # "Hello, World!"
strm.close()
```

---

### `write(s: Stream, args: varargs[string, `$`])`

Записывает несколько значений подряд, преобразуя каждое через `$` в строку.

```nim
import std/streams

var strm = newStringStream()
strm.write(1, 2, 3, " done")
strm.setPosition(0)
echo strm.readLine()  # "123 done"
strm.close()
```

---

### `writeLine(s: Stream, args: varargs[string, `$`])`

Записывает одно или несколько значений, после чего добавляет символ новой строки `\n`.

```nim
import std/streams

var strm = newStringStream()
strm.writeLine("First")
strm.writeLine(2, "nd line")
strm.setPosition(0)
echo strm.readAll()
# First
# 2nd line
strm.close()
```

---

## Низкоуровневое чтение и запись

### `readData(s: Stream, buffer: pointer, bufLen: int): int`

Читает до `bufLen` байт в нетипизированный буфер `buffer`. Возвращает фактическое число прочитанных байт.

> На JS-бэкенде `buffer` трактуется как `ptr string`.

```nim
import std/streams

var strm = newStringStream("abcde")
var buf: array[8, char]
let n = strm.readData(addr(buf), 8)
echo n        # 5
echo buf[0]   # 'a'
strm.close()
```

---

### `readDataStr(s: Stream, buffer: var string, slice: Slice[int]): int`

Читает данные в строку `buffer` в диапазон байт `slice`. Возвращает число прочитанных байт. Доступно на всех бэкендах, включая VM и JS.

```nim
import std/streams

var strm = newStringStream("abcde")
var buf = "12345"
let n = strm.readDataStr(buf, 0..3)
echo n    # 4
echo buf  # "abcd5"
strm.close()
```

---

### `readAll(s: Stream): string`

Читает все оставшиеся данные из потока в строку.

```nim
import std/streams

var strm = newStringStream("The first line\nthe second line\nthe third line")
echo strm.readAll()
# The first line
# the second line
# the third line
echo strm.atEnd()  # true
strm.close()
```

---

### `peekData(s: Stream, buffer: pointer, bufLen: int): int`

Читает до `bufLen` байт в буфер **без сдвига позиции** в потоке. Возвращает число прочитанных байт.

```nim
import std/streams

var strm = newStringStream("abcde")
var buf: array[6, char]
echo strm.peekData(addr(buf), 5)  # 5
echo buf[0]                        # 'a'
echo strm.getPosition()            # 0 — позиция не сдвинулась
strm.close()
```

---

### `writeData(s: Stream, buffer: pointer, bufLen: int)`

Записывает `bufLen` байт из нетипизированного буфера `buffer` в поток.

```nim
import std/streams

var strm = newStringStream()
var buf = ['H', 'i', '!']
strm.writeData(addr(buf), 3)
strm.setPosition(0)
echo strm.readStr(3)  # "Hi!"
strm.close()
```

---

### `read[T](s: Stream, result: var T)`

Обобщённая процедура чтения. Читает `sizeof(T)` байт в `result`. Бросает `IOError` при неполном чтении.

> **Недоступен на JS-бэкенде.**

```nim
import std/streams

var strm = newStringStream()
strm.write(255'u8)
strm.write(-1'i16)
strm.setPosition(0)
var a: uint8
var b: int16
strm.read(a)
strm.read(b)
echo a  # 255
echo b  # -1
strm.close()
```

---

### `peek[T](s: Stream, result: var T)`

Обобщённый просмотр. Читает `sizeof(T)` байт **без сдвига позиции**. Бросает `IOError` при неполном чтении.

> **Недоступен на JS-бэкенде.**

```nim
import std/streams

var strm = newStringStream()
strm.write(42'i32)
strm.setPosition(0)
var x: int32
strm.peek(x)
echo x                  # 42
echo strm.getPosition() # 0
strm.close()
```

---

## Чтение типизированных значений

Все процедуры ниже читают соответствующий тип из потока (двоичное представление, платформо-зависимый порядок байт). При неудаче бросают `IOError`.

> ⚠️ **Недоступны на JS-бэкенде.** Используйте `readStr` для JS.

### Символы

#### `readChar(s: Stream): char`

Читает один байт как `char`. При достижении конца потока возвращает `'\0'` (не бросает исключение).

```nim
import std/streams

var strm = newStringStream("Hi!")
echo strm.readChar()  # 'H'
echo strm.readChar()  # 'i'
echo strm.readChar()  # '!'
echo strm.readChar()  # '\x00' (конец)
strm.close()
```

---

### Целые числа со знаком

| Процедура | Тип | Размер |
|---|---|---|
| `readInt8(s)` | `int8` | 1 байт |
| `readInt16(s)` | `int16` | 2 байта |
| `readInt32(s)` | `int32` | 4 байта |
| `readInt64(s)` | `int64` | 8 байт |

```nim
import std/streams

var strm = newStringStream()
strm.write(100'i8)
strm.write(1000'i16)
strm.write(100000'i32)
strm.setPosition(0)
echo strm.readInt8()   # 100
echo strm.readInt16()  # 1000
echo strm.readInt32()  # 100000
strm.close()
```

---

### Целые числа без знака

| Процедура | Тип | Размер |
|---|---|---|
| `readUint8(s)` | `uint8` | 1 байт |
| `readUint16(s)` | `uint16` | 2 байта |
| `readUint32(s)` | `uint32` | 4 байта |
| `readUint64(s)` | `uint64` | 8 байт |

```nim
import std/streams

var strm = newStringStream()
strm.write(200'u8)
strm.write(60000'u16)
strm.setPosition(0)
echo strm.readUint8()   # 200
echo strm.readUint16()  # 60000
strm.close()
```

---

### Числа с плавающей точкой

| Процедура | Тип | Размер |
|---|---|---|
| `readFloat32(s)` | `float32` | 4 байта |
| `readFloat64(s)` | `float64` | 8 байт |

```nim
import std/streams

var strm = newStringStream()
strm.write(1.5'f32)
strm.write(3.14'f64)
strm.setPosition(0)
echo strm.readFloat32()  # 1.5
echo strm.readFloat64()  # 3.14
strm.close()
```

---

### Булевы значения

#### `readBool(s: Stream): bool`

Читает 1 байт. Любое ненулевое значение возвращается как `true`.

```nim
import std/streams

var strm = newStringStream()
strm.write(true)
strm.write(false)
strm.setPosition(0)
echo strm.readBool()  # true
echo strm.readBool()  # false
strm.close()
```

---

### Строки фиксированной длины

#### `readStr(s: Stream, length: int): string`

Читает ровно `length` байт из потока и возвращает как строку. Если данных меньше — возвращает укороченную строку.

```nim
import std/streams

var strm = newStringStream("abcde")
echo strm.readStr(2)  # "ab"
echo strm.readStr(2)  # "cd"
echo strm.readStr(2)  # "e"   (меньше 2 байт)
echo strm.readStr(2)  # ""    (конец)
strm.close()
```

#### `readStr(s: Stream, length: int, str: var string)`

Перегрузка «на месте»: записывает результат в переменную `str`.

---

## Просмотр без сдвига (peek)

Все `peek`-процедуры читают данные **без изменения позиции** в потоке. Их сигнатуры симметричны `read`-версиям.

### `peekChar(s: Stream): char`

```nim
import std/streams

var strm = newStringStream("AB")
echo strm.peekChar()    # 'A'
echo strm.peekChar()    # 'A' (позиция не сдвинулась)
echo strm.readChar()    # 'A'
echo strm.peekChar()    # 'B'
strm.close()
```

---

### `peekBool(s: Stream): bool`

Смотрит 1 байт как `bool` без сдвига позиции.

---

### Целые числа со знаком (peek)

`peekInt8`, `peekInt16`, `peekInt32`, `peekInt64`

```nim
import std/streams

var strm = newStringStream()
strm.write(42'i32)
strm.setPosition(0)
echo strm.peekInt32()   # 42
echo strm.peekInt32()   # 42  (снова — позиция не сдвинулась)
echo strm.readInt32()   # 42
strm.close()
```

---

### Целые числа без знака (peek)

`peekUint8`, `peekUint16`, `peekUint32`, `peekUint64`

---

### Числа с плавающей точкой (peek)

`peekFloat32`, `peekFloat64`

---

### Строки (peek)

#### `peekStr(s: Stream, length: int): string`

Читает `length` байт **без сдвига позиции**.

```nim
import std/streams

var strm = newStringStream("abcde")
echo strm.peekStr(2)  # "ab"
echo strm.peekStr(2)  # "ab" (снова)
echo strm.readStr(2)  # "ab"
echo strm.peekStr(2)  # "cd"
strm.close()
```

#### `peekStr(s: Stream, length: int, str: var string)`

Перегрузка «на месте».

---

## Чтение строк

### `readLine(s: Stream, line: var string): bool`

Читает строку в `line`. Разделитель — `LF` или `CRLF`, в результат **не включается**. Возвращает `false` при EOF. При `false` переменная `line` остаётся пустой.

```nim
import std/streams

var strm = newStringStream("First\r\nSecond\nThird")
var line = ""
while strm.readLine(line):
  echo line
# First
# Second
# Third
strm.close()
```

---

### `readLine(s: Stream): string`

Читает строку и возвращает её. При EOF бросает `IOError`.

> ⚠️ Менее эффективна, чем версия с `var string`.

```nim
import std/streams

var strm = newStringStream("A\nB\nC")
echo strm.readLine()  # "A"
echo strm.readLine()  # "B"
echo strm.readLine()  # "C"
# strm.readLine() теперь бросит IOError
strm.close()
```

---

### `peekLine(s: Stream, line: var string): bool`

Читает строку **без сдвига позиции**. Использует `defer` + `setPosition` для отката.

```nim
import std/streams

var strm = newStringStream("First\nSecond")
var line = ""
echo strm.peekLine(line)  # true
echo line                  # "First"
echo strm.peekLine(line)  # true
echo line                  # "First" (позиция не изменилась)
echo strm.readLine(line)  # true
echo line                  # "First"
strm.close()
```

---

### `peekLine(s: Stream): string`

Читает строку **без сдвига позиции** и возвращает её.

```nim
import std/streams

var strm = newStringStream("First\nSecond")
echo strm.peekLine()  # "First"
echo strm.peekLine()  # "First"
echo strm.readLine()  # "First"
echo strm.peekLine()  # "Second"
strm.close()
```

---

## Итератор

### `iterator lines(s: Stream): string`

Итерирует по всем строкам потока, используя `readLine`. Удобная замена цикла `while readLine`.

```nim
import std/streams

var strm = newStringStream("Line 1\nLine 2\nLine 3")
var collected: seq[string]
for line in strm.lines():
  collected.add(line)
echo collected  # @["Line 1", "Line 2", "Line 3"]
strm.close()
```

---

## Таблица соответствий read/peek

| Тип | `read` (со сдвигом) | `peek` (без сдвига) |
|---|---|---|
| `char` | `readChar(s)` | `peekChar(s)` |
| `bool` | `readBool(s)` | `peekBool(s)` |
| `int8` | `readInt8(s)` | `peekInt8(s)` |
| `int16` | `readInt16(s)` | `peekInt16(s)` |
| `int32` | `readInt32(s)` | `peekInt32(s)` |
| `int64` | `readInt64(s)` | `peekInt64(s)` |
| `uint8` | `readUint8(s)` | `peekUint8(s)` |
| `uint16` | `readUint16(s)` | `peekUint16(s)` |
| `uint32` | `readUint32(s)` | `peekUint32(s)` |
| `uint64` | `readUint64(s)` | `peekUint64(s)` |
| `float32` | `readFloat32(s)` | `peekFloat32(s)` |
| `float64` | `readFloat64(s)` | `peekFloat64(s)` |
| `string` (фикс. длина) | `readStr(s, n)` | `peekStr(s, n)` |
| `string` (строка) | `readLine(s)` | `peekLine(s)` |
| `T` (обобщённый) | `read[T](s, result)` | `peek[T](s, result)` |
| буфер | `readData(s, buf, n)` | `peekData(s, buf, n)` |
| строковый буфер | `readDataStr(s, buf, sl)` | — |
| все данные | `readAll(s)` | — |

---

## Полные примеры

### Бинарная сериализация структуры

```nim
import std/streams

type Point = object
  x, y: float32

proc writePoint(s: Stream, p: Point) =
  s.write(p.x)
  s.write(p.y)

proc readPoint(s: Stream): Point =
  result.x = s.readFloat32()
  result.y = s.readFloat32()

var strm = newStringStream()
writePoint(strm, Point(x: 1.5, y: -2.0))
writePoint(strm, Point(x: 3.0, y:  4.5))
strm.setPosition(0)
echo readPoint(strm)  # (x: 1.5, y: -2.0)
echo readPoint(strm)  # (x: 3.0, y: 4.5)
strm.close()
```

---

### Чтение CSV-файла построчно

```nim
import std/streams

let strm = newFileStream("data.csv")
if not isNil(strm):
  for line in strm.lines():
    let parts = line.split(',')
    echo parts
  strm.close()
```

---

### Безопасное открытие файла с defer

```nim
import std/streams

let strm = newFileStream("config.txt")
defer: strm.close()

if not isNil(strm):
  var line = ""
  while strm.readLine(line):
    echo line
```

---

### Запись и чтение смешанных типов

```nim
import std/streams

var strm = newStringStream()
strm.write(42'u16)          # 2 байта
strm.write(true)             # 1 байт
strm.writeLine("end")       # строка + \n

strm.setPosition(0)
echo strm.readUint16()      # 42
echo strm.readBool()        # true
echo strm.readLine()        # "end"
strm.close()
```

---

### Копирование потоков через StringStream

```nim
import std/streams

proc streamToString(s: Stream): string =
  ## Читает весь поток в строку с текущей позиции
  result = s.readAll()

var src = newStringStream("Hello, Stream world!")
let content = streamToString(src)
echo content        # "Hello, Stream world!"
echo src.atEnd()    # true
src.close()
```

---

### Реализация собственного потока

```nim
import std/streams

# Пример: поток с подсчётом байт записи
type CountingStream = ref object of StreamObj
  inner: Stream
  bytesWritten: int

proc newCountingStream(inner: Stream): CountingStream =
  result = CountingStream(inner: inner)
  result.writeDataImpl = proc(s: Stream, buf: pointer, len: int) =
    let cs = CountingStream(s)
    cs.inner.writeData(buf, len)
    cs.bytesWritten += len
  result.closeImpl = proc(s: Stream) =
    CountingStream(s).inner.close()

var inner = newStringStream()
var cs = newCountingStream(inner)
cs.write("Hello")
cs.write(", World!")
echo cs.bytesWritten  # 13
cs.close()
```
