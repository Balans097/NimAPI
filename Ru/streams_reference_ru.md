# streams — справочник модуля

> **Импорт:** `import std/streams`
> **Область применения:** абстрактный интерфейс потока ввода-вывода (чтение/запись), с двумя готовыми реализациями — поверх файла (`FileStream`) и поверх строки (`StringStream`).

Модуль решает одну задачу: дать единый API для чтения и записи данных, не зависящий
от того, откуда эти данные на самом деле берутся — из файла на диске или из строки
в памяти. Всё построено вокруг базового типа `Stream`, у которого есть набор полей-
процедур (`closeImpl`, `readDataImpl` и т. д.) — каждая конкретная реализация потока
(`FileStream`, `StringStream`) подставляет туда свои процедуры. Общие конвенции модуля:
— пара "read.../peek..." — `read...` читает и сдвигает позицию в потоке, `peek...`
читает то же самое, но возвращает позицию назад (просмотр без потребления); —
низкоуровневые процедуры (`readData`, `writeData`) работают с сырыми `pointer`-
буферами, а поверх них построены типизированные процедуры (`readInt32`, `readStr`,
`readLine` и т. п.).

---

## Оглавление

I. [Тип Stream и его устройство](#тип-stream-и-его-устройство)
   1. [`Stream` / `StreamObj`](#stream--streamobj)
II. [Управление стримом: закрытие, сброс, позиция](#управление-стримом-закрытие-сброс-позиция)
   1. [`close`](#close)
   2. [`flush`](#flush)
   3. [`atEnd`](#atend)
   4. [`setPosition`](#setposition)
   5. [`getPosition`](#getposition)
III. [Низкоуровневое чтение и запись через буфер](#низкоуровневое-чтение-и-запись-через-буфер)
   1. [`readData`](#readdata)
   2. [`peekData`](#peekdata)
   3. [`writeData`](#writedata)
IV. [Обобщённая запись и чтение произвольных типов](#обобщённая-запись-и-чтение-произвольных-типов)
   1. [`write[T]` (обобщённая запись)](#writet-обобщённая-запись)
   2. [`write` (строка)](#write-строка)
   3. [`write` (varargs)](#write-varargs)
   4. [`writeLine`](#writeline)
   5. [`read[T]` (обобщённое чтение)](#readt-обобщённое-чтение)
   6. [`peek[T]` (обобщённый просмотр)](#peekt-обобщённый-просмотр)
V. [Чтение и просмотр типизированных числовых значений](#чтение-и-просмотр-типизированных-числовых-значений)
   1. [`readChar` / `peekChar`](#readchar--peekchar)
   2. [`readBool` / `peekBool`](#readbool--peekbool)
   3. [`readInt8/16/32/64` / `peekInt8/16/32/64`](#readint81632643264--peekint81632643264)
   4. [`readUint8/16/32/64` / `peekUint8/16/32/64`](#readuint8163264--peekuint8163264)
   5. [`readFloat32/64` / `peekFloat32/64`](#readfloat3264--peekfloat3264)
VI. [Чтение строк заданной длины](#чтение-строк-заданной-длины)
   1. [`readStr` / `peekStr`](#readstr--peekstr)
VII. [Построчное чтение](#построчное-чтение)
   1. [`readLine` / `peekLine`](#readline--peekline)
   2. [`lines` (итератор)](#lines-итератор)
VIII. [StringStream — стрим поверх строки](#stringstream--стрим-поверх-строки)
   1. [`newStringStream`](#newstringstream)
IX. [FileStream — стрим поверх файла](#filestream--стрим-поверх-файла)
   1. [`newFileStream` (из открытого `File`)](#newfilestream-из-открытого-file)
   2. [`newFileStream` (из имени файла)](#newfilestream-из-имени-файла)
   3. [`openFileStream`](#openfilestream)
X. [Практические рецепты](#практические-рецепты)
XI. [Краткая таблица](#краткая-таблица)
XII. [Сводка: какую процедуру выбрать](#сводка-какую-процедуру-выбрать)

---

## Тип Stream и его устройство

### `Stream` / `StreamObj`

```nim
type
  Stream* = ref StreamObj
  StreamObj* = object of RootObj
    closeImpl*: proc (s: Stream)
    atEndImpl*: proc (s: Stream): bool
    setPositionImpl*: proc (s: Stream, pos: int)
    getPositionImpl*: proc (s: Stream): int
    readDataStrImpl*: proc (s: Stream, buffer: var string, slice: Slice[int]): int
    readLineImpl*: proc (s: Stream, line: var string): bool
    readDataImpl*: proc (s: Stream, buffer: pointer, bufLen: int): int
    peekDataImpl*: proc (s: Stream, buffer: pointer, bufLen: int): int
    writeDataImpl*: proc (s: Stream, buffer: pointer, bufLen: int)
    flushImpl*: proc (s: Stream)
```

**Что делает.** `Stream` — это единственный тип, с которым работают все процедуры
модуля: и `FileStream`, и `StringStream` в итоге приводятся к нему через наследование
(`object of RootObj` / `object of StreamObj`). Сам по себе `StreamObj` не хранит
данные потока — вместо этого он хранит набор указателей на процедуры (`closeImpl`,
`readDataImpl` и т. д.), которые каждая конкретная реализация подставляет своими.
Это классический паттерн "таблица виртуальных методов", реализованный вручную:
вызов `close(s)` внутри себя просто зовёт `s.closeImpl(s)`, а какая именно функция
там окажется — зависит от того, был ли `s` создан как `FileStream` или `StringStream`.
Поля помечены как публичные (`*`) специально для того, чтобы сторонние модули могли
подставлять свои реализации потоков, не входящие в стандартную библиотеку.

- **`closeImpl`, `atEndImpl`, `setPositionImpl`, `getPositionImpl`, `flushImpl`** —
  обязательные (для базовых операций) процедуры-реализации.
- **`readDataStrImpl`, `readLineImpl`, `readDataImpl`, `peekDataImpl`,
  `writeDataImpl`** — процедуры чтения/записи; часть из них (`readDataImpl`,
  `peekDataImpl`, `writeDataImpl`) недоступна в режиме компиляции (VM) и в JS-бэкенде,
  так как использует сырые указатели (`pointer`).

```nim
# Пример не самостоятельный: показывает, как выглядит "самодельный" стрим,
# если бы вы захотели реализовать свою собственную обёртку.
type
  MyStream = ref MyStreamObj
  MyStreamObj = object of StreamObj
    data: string
```

---

## Управление стримом: закрытие, сброс, позиция

### `close`

```nim
proc close*(s: Stream)
```

**Что делает.** Закрывает поток `s`, вызывая его `closeImpl`. Для `FileStream` это
означает закрытие файлового дескриптора, для `StringStream` — очистку внутренней
строки (`data` становится пустой строкой). Безопасен для повторного вызова и для
`nil`-потока: обе проверки (`isNil(s)`, `isNil(s.closeImpl)`) стоят перед вызовом,
поэтому `close(nil)` просто ничего не делает, а не падает.

- **`s: Stream`** — закрываемый поток; может быть `nil`.

```nim
var strm = newStringStream("The first line\nthe second line")
# ... что-то делаем с потоком ...
close(strm)

# close безопасен даже если поток не удалось открыть
var maybeStrm = newFileStream("несуществующий_файл.txt")
defer: close(maybeStrm)  # сработает, даже если maybeStrm == nil
```

---

### `flush`

```nim
proc flush*(s: Stream)
```

**Что делает.** Принудительно сбрасывает буферизованные, но ещё не записанные
данные потока `s` во "внешнюю среду" — например, из буфера `File` на диск. Пока
`flush` не вызван (или поток не закрыт), данные могут физически ещё не попасть
в файл, даже если `write` уже отработал.

- **`s: Stream`** — поток, буферы которого нужно сбросить.

```nim
var strm = newFileStream("output.txt", fmWrite)
write(strm, "hello")
# до flush() данные могут ещё не оказаться на диске
flush(strm)
# теперь гарантированно записаны
close(strm)
```

---

### `atEnd`

```nim
proc atEnd*(s: Stream): bool
```

**Что делает.** Проверяет, достигнут ли конец потока — то есть можно ли ещё что-то
прочитать. Возвращает `true`, когда все данные уже прочитаны.

- **`s: Stream`** — проверяемый поток (только для чтения по смыслу операции).

```nim
var strm = newStringStream("abc")
doAssert atEnd(strm) == false
discard readAll(strm)
doAssert atEnd(strm) == true
close(strm)
```

---

### `setPosition`

```nim
proc setPosition*(s: Stream, pos: int)
```

**Что делает.** Устанавливает текущую позицию чтения/записи в потоке на `pos`
(отсчёт от начала, с нуля). Для `StringStream` позиция обрезается (`clamp`) в
границы `0..len(data)`, для `FileStream` — соответствует смещению внутри файла.

- **`s: Stream`** — поток, у которого меняется позиция.
- **`pos: int`** — новая позиция (индекс байта от начала потока).

```nim
var strm = newStringStream("The first line\nthe second line")
setPosition(strm, 4)
doAssert readLine(strm) == "first line"
setPosition(strm, 0)
doAssert readLine(strm) == "The first line"
close(strm)
```

---

### `getPosition`

```nim
proc getPosition*(s: Stream): int
```

**Что делает.** Возвращает текущую позицию чтения/записи в потоке — то есть сколько
байт уже "пройдено" от начала.

- **`s: Stream`** — поток, позиция которого запрашивается.

```nim
var strm = newStringStream("The first line\nthe second line")
doAssert getPosition(strm) == 0
discard readLine(strm)
doAssert getPosition(strm) == 15
close(strm)
```

---

## Низкоуровневое чтение и запись через буфер

Три процедуры этого раздела — фундамент, на котором построены все остальные
типизированные операции чтения/записи модуля (`readInt32`, `readStr`, `write` и т. д.).
Работают напрямую с `pointer` и сырым количеством байт, поэтому недоступны в VM
(на этапе компиляции) без обхода через `readDataStr`.

### `readData`

```nim
proc readData*(s: Stream, buffer: pointer, bufLen: int): int
```

**Что делает.** Читает из потока `s` не более `bufLen` байт в буфер `buffer` и
сдвигает позицию потока вперёд на столько байт, сколько реально удалось прочитать.
Возвращает фактическое число прочитанных байт — оно может быть меньше `bufLen`,
если поток закончился раньше.

- **`s: Stream`** — источник чтения.
- **`buffer: pointer`** — указатель на область памяти для записи прочитанных данных;
  вызывающий код отвечает за то, чтобы там было выделено достаточно места.
- **`bufLen: int`** — сколько байт максимум нужно прочитать.

```nim
var strm = newStringStream("abcde")
var buffer: array[6, char]
doAssert readData(strm, addr(buffer), 1024) == 5  # прочитано меньше, чем просили
doAssert buffer == ['a', 'b', 'c', 'd', 'e', '\x00']
doAssert atEnd(strm) == true
close(strm)
```

---

### `peekData`

```nim
proc peekData*(s: Stream, buffer: pointer, bufLen: int): int
```

**Что делает.** То же самое, что `readData`, но не сдвигает позицию потока —
данные можно "подсмотреть" и прочитать снова тем же или другим вызовом.

- **`s: Stream`** — источник просмотра.
- **`buffer: pointer`** — буфер для записи просмотренных данных.
- **`bufLen: int`** — сколько байт максимум нужно просмотреть.

```nim
var strm = newStringStream("abcde")
var buffer: array[6, char]
doAssert peekData(strm, addr(buffer), 1024) == 5
doAssert buffer == ['a', 'b', 'c', 'd', 'e', '\x00']
doAssert atEnd(strm) == false  # позиция не сдвинулась
close(strm)
```

---

### `writeData`

```nim
proc writeData*(s: Stream, buffer: pointer, bufLen: int)
```

**Что делает.** Записывает в поток `s` ровно `bufLen` байт из `buffer`, сдвигая
позицию вперёд. В отличие от чтения, здесь нет "частичной" записи — либо
записываются все байты, либо (для `FileStream`) поднимается `IOError`.

- **`s: Stream`** — приёмник записи.
- **`buffer: pointer`** — указатель на данные для записи.
- **`bufLen: int`** — сколько байт записать.

```nim
var strm = newStringStream("")
var buffer = ['a', 'b', 'c', 'd', 'e']
writeData(strm, addr(buffer), sizeof(buffer))
doAssert atEnd(strm) == true
setPosition(strm, 0)
var buffer2: array[6, char]
doAssert readData(strm, addr(buffer2), sizeof(buffer2)) == 5
close(strm)
```

---

## Обобщённая запись и чтение произвольных типов

Этот раздел — удобная типизированная обёртка над `writeData`/`readData`: вместо
ручного вычисления `sizeof` и приведения указателей можно писать/читать значение
любого типа `T` одной процедурой.

### `write[T]` (обобщённая запись)

```nim
proc write*[T](s: Stream, x: T)
```

**Что делает.** Записывает значение `x` произвольного типа `T` в поток `s` "как есть" —
побайтово, через `sizeof(x)` байт. Недоступна для JS-бэкенда (используйте перегрузку
`write(Stream, string)`).

- **`s: Stream`** — приёмник записи.
- **`x: T`** — записываемое значение любого типа.

```nim
var strm = newStringStream("")
write(strm, "abcde")
setPosition(strm, 0)
doAssert readAll(strm) == "abcde"
close(strm)
```

---

### `write` (строка)

```nim
proc write*(s: Stream, x: string)
```

**Что делает.** Записывает строку `x` в поток "как есть" — без длины и без
завершающего нулевого байта.

- **`s: Stream`** — приёмник записи.
- **`x: string`** — записываемая строка.

```nim
var strm = newStringStream("")
write(strm, "THE FIRST LINE")
setPosition(strm, 0)
doAssert readLine(strm) == "THE FIRST LINE"
close(strm)
```

---

### `write` (varargs)

```nim
proc write*(s: Stream, args: varargs[string, `$`])
```

**Что делает.** Записывает одну или несколько строк подряд, без разделителей и
без завершающих нулей. Каждый аргумент неявно приводится к строке через `$`.

- **`s: Stream`** — приёмник записи.
- **`args: varargs[string, `$`]`** — произвольное число значений, каждое из
  которых будет преобразовано в строку и записано подряд.

```nim
var strm = newStringStream("")
write(strm, 1, 2, 3, 4)
setPosition(strm, 0)
doAssert readLine(strm) == "1234"
close(strm)
```

---

### `writeLine`

```nim
proc writeLine*(s: Stream, args: varargs[string, `$`])
```

**Что делает.** То же, что `write` с несколькими аргументами, но в конце
дописывает символ перевода строки `\n`.

- **`s: Stream`** — приёмник записи.
- **`args: varargs[string, `$`]`** — значения для записи перед переводом строки.

```nim
var strm = newStringStream("")
writeLine(strm, 1, 2)
writeLine(strm, 3, 4)
setPosition(strm, 0)
doAssert readAll(strm) == "12\n34\n"
close(strm)
```

---

### `read[T]` (обобщённое чтение)

```nim
proc read*[T](s: Stream, result: var T)
```

**Что делает.** Читает `sizeof(T)` байт из потока в переменную `result`. Если
прочитано меньше байт, чем нужно для заполнения `T` (поток закончился раньше),
поднимает `IOError`.

- **`s: Stream`** — источник чтения.
- **`result: var T`** — переменная, в которую записывается прочитанное значение;
  тип `T` определяет, сколько байт будет прочитано.

```nim
var strm = newStringStream("012")
var i: int8
read(strm, i)
doAssert i == 48  # код символа '0'
close(strm)
```

---

### `peek[T]` (обобщённый просмотр)

```nim
proc peek*[T](s: Stream, result: var T)
```

**Что делает.** То же, что `read[T]`, но без сдвига позиции потока — значение
можно "подсмотреть" и прочитать заново.

- **`s: Stream`** — источник просмотра.
- **`result: var T`** — переменная для записи просмотренного значения.

```nim
var strm = newStringStream("012")
var i: int8
peek(strm, i)
doAssert i == 48
read(strm, i)
doAssert i == 48  # то же самое значение читается ещё раз
close(strm)
```

---

## Чтение и просмотр типизированных числовых значений

Все процедуры этого раздела построены по одному и тому же шаблону поверх `read[T]`
и `peek[T]`: пара "читает и сдвигает позицию" / "читает без сдвига". Различаются
они только конкретным типом и, соответственно, числом читаемых байт.

### `readChar` / `peekChar`

```nim
proc readChar*(s: Stream): char
proc peekChar*(s: Stream): char
```

**Что делает.** Читает (или просматривает) один символ из потока. Если поток уже
закончился, вместо исключения возвращается специальный символ-маркер `'\0'` —
единственная процедура чтения в модуле, которая обозначает конец данных значением,
а не исключением.

- **`s: Stream`** — источник чтения.

```nim
var strm = newStringStream("12\n3")
doAssert readChar(strm) == '1'
doAssert readChar(strm) == '2'
doAssert readChar(strm) == '\n'
doAssert readChar(strm) == '3'
doAssert readChar(strm) == '\x00'  # конец потока — маркер, не исключение
close(strm)
```

---

### `readBool` / `peekBool`

```nim
proc readBool*(s: Stream): bool
proc peekBool*(s: Stream): bool
```

**Что делает.** Читает (или просматривает) один байт и интерпретирует его как
`bool`: любое ненулевое значение — `true`, ноль — `false`. В отличие от `readChar`,
при выходе за конец потока поднимает `IOError`, а не возвращает специальное значение.

- **`s: Stream`** — источник чтения.

```nim
var strm = newStringStream()
write(strm, true)
write(strm, false)
flush(strm)
setPosition(strm, 0)
doAssert readBool(strm) == true
doAssert readBool(strm) == false
doAssertRaises(IOError): discard readBool(strm)  # данные закончились
close(strm)
```

---

### `readInt8/16/32/64` / `peekInt8/16/32/64`

```nim
proc readInt8*(s: Stream): int8
proc readInt16*(s: Stream): int16
proc readInt32*(s: Stream): int32
proc readInt64*(s: Stream): int64
proc peekInt8*(s: Stream): int8
proc peekInt16*(s: Stream): int16
proc peekInt32*(s: Stream): int32
proc peekInt64*(s: Stream): int64
```

**Что делает.** Восемь процедур с идентичным поведением, отличающиеся только
разрядностью читаемого целого числа со знаком (1, 2, 4 или 8 байт). `read...`
сдвигает позицию потока на соответствующее число байт, `peek...` — нет. При
недостатке данных поднимается `IOError`.

- **`s: Stream`** — источник чтения; разрядность выбирается именем процедуры,
  а не аргументом.

```nim
var strm = newStringStream()
write(strm, 1'i32)
write(strm, 2'i32)
flush(strm)
setPosition(strm, 0)
doAssert readInt32(strm) == 1'i32
doAssert readInt32(strm) == 2'i32
doAssertRaises(IOError): discard readInt32(strm)
close(strm)

# peek не сдвигает позицию — то же значение читается ещё раз
var strm2 = newStringStream()
write(strm2, 1'i16)
flush(strm2)
setPosition(strm2, 0)
doAssert peekInt16(strm2) == 1'i16
doAssert readInt16(strm2) == 1'i16  # значение то же самое
close(strm2)
```

---

### `readUint8/16/32/64` / `peekUint8/16/32/64`

```nim
proc readUint8*(s: Stream): uint8
proc readUint16*(s: Stream): uint16
proc readUint32*(s: Stream): uint32
proc readUint64*(s: Stream): uint64
proc peekUint8*(s: Stream): uint8
proc peekUint16*(s: Stream): uint16
proc peekUint32*(s: Stream): uint32
proc peekUint64*(s: Stream): uint64
```

**Что делает.** Полный аналог предыдущего раздела, но для целых чисел без знака.
Поведение на границах (конец потока → `IOError`) и семантика `read`/`peek`
идентичны `readInt.../peekInt...`.

- **`s: Stream`** — источник чтения; разрядность выбирается именем процедуры.

```nim
var strm = newStringStream()
write(strm, 1'u8)
write(strm, 2'u8)
flush(strm)
setPosition(strm, 0)
doAssert readUint8(strm) == 1'u8
doAssert readUint8(strm) == 2'u8
doAssertRaises(IOError): discard readUint8(strm)
close(strm)
```

---

### `readFloat32/64` / `peekFloat32/64`

```nim
proc readFloat32*(s: Stream): float32
proc readFloat64*(s: Stream): float64
proc peekFloat32*(s: Stream): float32
proc peekFloat64*(s: Stream): float64
```

**Что делает.** Те же принципы, что и для целых чисел, но для чисел с плавающей
точкой одинарной (4 байта) или двойной (8 байт) точности.

- **`s: Stream`** — источник чтения; разрядность выбирается именем процедуры.

```nim
var strm = newStringStream()
write(strm, 1'f64)
write(strm, 2'f64)
flush(strm)
setPosition(strm, 0)
doAssert readFloat64(strm) == 1'f64
doAssert readFloat64(strm) == 2'f64
doAssertRaises(IOError): discard readFloat64(strm)
close(strm)
```

---

## Чтение строк заданной длины

### `readStr` / `peekStr`

```nim
proc readStr*(s: Stream, length: int): string
proc readStr*(s: Stream, length: int, str: var string)
proc peekStr*(s: Stream, length: int): string
proc peekStr*(s: Stream, length: int, str: var string)
```

**Что делает.** Читает (или просматривает) ровно `length` байт как строку.
Если в потоке осталось меньше данных, чем запрошено, возвращается укороченная
строка — исключение не поднимается (в отличие от типизированных числовых
процедур). Версия с параметром `str: var string` пишет результат в уже
существующую строку, не создавая новую; версия без него возвращает новую строку.

- **`s: Stream`** — источник чтения.
- **`length: int`** — сколько байт запросить.
- **`str: var string`** *(опционально)* — существующая строка-приёмник; если не
  указана, процедура выделяет новую строку сама.

```nim
var strm = newStringStream("abcde")
doAssert readStr(strm, 2) == "ab"
doAssert readStr(strm, 2) == "cd"
doAssert readStr(strm, 2) == "e"    # осталась только 1 буква — укороченный результат
doAssert readStr(strm, 2) == ""     # поток пуст
close(strm)

var strm2 = newStringStream("abcde")
doAssert peekStr(strm2, 2) == "ab"
doAssert peekStr(strm2, 2) == "ab"  # повторный просмотр — то же самое
doAssert readStr(strm2, 2) == "ab"  # а теперь сдвигаем позицию по-настоящему
doAssert peekStr(strm2, 2) == "cd"
close(strm2)
```

---

## Построчное чтение

### `readLine` / `peekLine`

```nim
proc readLine*(s: Stream, line: var string): bool
proc readLine*(s: Stream): string
proc peekLine*(s: Stream, line: var string): bool
proc peekLine*(s: Stream): string
```

**Что делает.** Читает одну строку текста, разделённую `LF` (`\n`) или `CRLF`
(`\r\n`); сам символ(ы) перевода строки в результат не попадают. Версия с
`line: var string` и возвращаемым `bool` — низкоуровневая: `false` означает, что
поток уже закончился и новых данных не появилось (при этом `line` не изменяется).
Версия, возвращающая `string` напрямую, менее эффективна и поднимает `IOError`
на конце потока вместо возврата признака конца. `peekLine` — то же самое, что
`readLine`, но восстанавливает позицию потока после чтения (реализовано через
`getPosition`/`setPosition` вокруг вызова `readLine`, а не отдельным механизмом).

- **`s: Stream`** — источник чтения.
- **`line: var string`** *(для булевой формы)* — строка, в которую записывается
  прочитанная строка; не должна быть `nil`.

```nim
var strm = newStringStream("The first line\nthe second line\nthe third line")
var line = ""
doAssert readLine(strm, line) == true
doAssert line == "The first line"
doAssert readLine(strm, line) == true
doAssert line == "the second line"
close(strm)

# форма без параметра line — поднимает исключение на конце потока
var strm2 = newStringStream("only line")
doAssert readLine(strm2) == "only line"
doAssertRaises(IOError): discard readLine(strm2)
close(strm2)

# peekLine не сдвигает позицию
var strm3 = newStringStream("first\nsecond")
doAssert peekLine(strm3) == "first"
doAssert peekLine(strm3) == "first"  # снова та же строка
doAssert readLine(strm3) == "first"  # а теперь сдвигаем по-настоящему
doAssert peekLine(strm3) == "second"
close(strm3)
```

---

### `lines` (итератор)

```nim
iterator lines*(s: Stream): string
```

**Что делает.** Последовательно проходит по всем строкам потока, используя под
капотом `readLine`. Удобная обёртка для цикла `for line in lines(strm)` вместо
ручного `while readLine(strm, line)`.

- **`s: Stream`** — поток, по строкам которого нужно пройтись.

```nim
var strm = newStringStream("The first line\nthe second line\nthe third line")
var collected: seq[string]
for line in lines(strm):
  add(collected, line)
doAssert collected == @["The first line", "the second line", "the third line"]
close(strm)
```

---

## StringStream — стрим поверх строки

### `newStringStream`

```nim
proc newStringStream*(s: sink string = ""): owned StringStream
```

**Что делает.** Создаёт новый поток, который читает и пишет прямо в строку `s`
(или в новую пустую строку, если аргумент не передан). Все операции записи меняют
публичное поле `data` этого потока — то есть в любой момент можно посмотреть на
`StringStream.data`, чтобы увидеть накопленный результат, не дожидаясь `close`.
Реализация читает и пишет через `copyMem` напрямую в буфер строки — отсюда
эффективность по сравнению, например, с построчной конкатенацией.

- **`s: sink string`** — начальное содержимое потока; передаётся по владению
  (`sink`), то есть строка "забирается" в поток без лишнего копирования.

```nim
var strm = newStringStream("The first line\nthe second line\nthe third line")
doAssert readLine(strm) == "The first line"
doAssert readLine(strm) == "the second line"
doAssert readLine(strm) == "the third line"
close(strm)

# запись и последующее чтение обратно
var strm2 = newStringStream("")
writeLine(strm2, "hello")
doAssert strm2.data == "hello\n"  # data — публичное поле, доступно напрямую
close(strm2)
```

---

## FileStream — стрим поверх файла

**Примечание.** Все три процедуры этого раздела недоступны для JS-бэкенда — только
для нативной компиляции.

### `newFileStream` (из открытого `File`)

```nim
proc newFileStream*(f: File): owned FileStream
```

**Что делает.** Оборачивает уже открытый файл `f` (тип `File` из модуля `io`/`syncio`)
в поток. Полезно, когда файл нужно открыть с нестандартными параметрами через
`open`, а дальше работать с ним через единый интерфейс `Stream`.

- **`f: File`** — заранее открытый файл.

```nim
var f: File
if open(f, "somefile.txt", fmRead, -1):
  var strm = newFileStream(f)
  var line = ""
  while readLine(strm, line):
    echo line
  close(strm)
```

---

### `newFileStream` (из имени файла)

```nim
proc newFileStream*(filename: string, mode: FileMode = fmRead,
                     bufSize: int = -1): owned FileStream
```

**Что делает.** Открывает файл `filename` в режиме `mode` и сразу оборачивает
его в поток. **Если файл не удалось открыть, возвращает `nil`** — это нужно
проверять явно (`if not isNil(strm)`), иначе последующие операции с `nil`-потоком
приведут к падению. Именно из-за этой особенности в документации модуля
рекомендуется `openFileStream` для случаев, где ошибка открытия должна быть
обработана как исключение, а не как `nil`.

- **`filename: string`** — путь к файлу.
- **`mode: FileMode`** — режим доступа (`fmRead`, `fmWrite` и т. д.), по умолчанию
  `fmRead`.
- **`bufSize: int`** — размер буфера ввода-вывода; `-1` означает буфер по умолчанию.

```nim
var strm = newFileStream("somefile.txt", fmWrite)
if not isNil(strm):
  writeLine(strm, "The first line")
  writeLine(strm, "the second line")
  writeLine(strm, "the third line")
  close(strm)
```

---

### `openFileStream`

```nim
proc openFileStream*(filename: string, mode: FileMode = fmRead,
                      bufSize: int = -1): owned FileStream
```

**Что делает.** Делает то же самое, что и `newFileStream(filename, ...)`, но при
неудаче не возвращает `nil`, а поднимает `IOError` — удобнее в коде, где ошибка
открытия файла должна прерывать выполнение через исключение, а не через ручную
проверку на `nil` на каждом шаге.

- **`filename: string`** — путь к файлу.
- **`mode: FileMode`** — режим доступа, по умолчанию `fmRead`.
- **`bufSize: int`** — размер буфера, `-1` — значение по умолчанию.

```nim
try:
  var strm = openFileStream("somefile.txt")
  echo readLine(strm)
  close(strm)
except:
  write(stderr, getCurrentExceptionMsg())
```

---

## Практические рецепты

### Разбор строки в двоичный конфиг и обратно

Комбинация `writeLine`/`write`/`readInt32`/`readStr` — типичная схема
сериализации простого бинарного формата поверх `StringStream`.

```nim
# Записываем "запись": целое число + строка фиксированной длины.
var buf = newStringStream("")
write(buf, 42'i32)
write(buf, "abcd")  # ровно 4 байта, без длины и терминатора
flush(buf)

# Читаем обратно в том же порядке, в котором писали.
setPosition(buf, 0)
let recordId = readInt32(buf)
let payload = readStr(buf, 4)
doAssert recordId == 42'i32
doAssert payload == "abcd"
close(buf)
```

---

### Построчная фильтрация большого файла без загрузки его целиком

Использует `lines` вместе с `newFileStream`/`openFileStream`, чтобы не держать
весь файл в памяти сразу.

```nim
try:
  var input = openFileStream("access.log")
  var output = newStringStream("")
  for line in lines(input):
    if contains(line, "ERROR"):
      writeLine(output, line)
  close(input)
  echo readAll(output)  # только строки с ошибками
  close(output)
except IOError:
  write(stderr, "не удалось прочитать access.log\n")
```

---

### Просмотр заголовка без "потребления" потока

Комбинация `peekStr`/`peekChar` с последующим полноценным чтением — типичный
паттерн для распознавания формата файла по первым байтам ("магическому числу")
перед тем, как решить, каким парсером его читать дальше.

```nim
var strm = newStringStream("PNG\x89data...")
let header = peekStr(strm, 3)  # позиция не сдвинулась
if header == "PNG":
  discard readStr(strm, 3)      # теперь по-настоящему пропускаем заголовок
  let rest = readAll(strm)
  doAssert rest == "\x89data..."
close(strm)
```

---

### Копирование содержимого одного потока в другой порциями

Использует низкоуровневые `readData`/`writeData` напрямую — пригодится, когда
нужен полный контроль над размером буфера копирования (например, для больших
файлов, которые не хочется читать через `readAll` целиком).

```nim
proc copyStream(src, dst: Stream, chunkSize = 4096) =
  var chunk = newString(chunkSize)
  while true:
    let n = readData(src, addr(chunk[0]), chunkSize)
    if n == 0:
      break
    writeData(dst, addr(chunk[0]), n)

var source = newStringStream("немного данных для копирования")
var target = newStringStream("")
copyStream(source, target)
doAssert target.data == source.data
close(source)
close(target)
```

---

## Краткая таблица

| Задача | Сдвигает позицию | Что вернёт / куда пишет |
|---|---|---|
| Закрыть поток | — | `close` |
| Сбросить буфер записи | — | `flush` |
| Проверить конец данных | нет | `atEnd` → `bool` |
| Узнать/задать позицию | задаёт | `getPosition` / `setPosition` |
| Прочитать сырые байты в буфер | да | `readData` → число байт |
| Просмотреть сырые байты | нет | `peekData` → число байт |
| Записать сырые байты | да | `writeData` |
| Записать значение произвольного типа | да | `write[T]` / `write(string)` / `write(varargs)` |
| Записать строку(и) с переводом строки | да | `writeLine` |
| Прочитать/просмотреть значение произвольного типа | read: да, peek: нет | `read[T]` / `peek[T]` |
| Прочитать/просмотреть один символ | read: да, peek: нет | `readChar` / `peekChar` (конец → `'\0'`) |
| Прочитать/просмотреть bool или число (int/uint/float) | read: да, peek: нет | `readBool`/`readInt*`/`readUint*`/`readFloat*` и `peek`-варианты (конец → `IOError`) |
| Прочитать/просмотреть строку заданной длины | read: да, peek: нет | `readStr` / `peekStr` (укорачивается на конце, без исключения) |
| Прочитать/просмотреть одну строку текста | read: да, peek: нет | `readLine` / `peekLine` |
| Пройтись по всем строкам | да | `lines` (итератор) |
| Стрим поверх строки в памяти | — | `newStringStream` |
| Стрим поверх файла (без исключения при ошибке) | — | `newFileStream` (может вернуть `nil`) |
| Стрим поверх файла (с исключением при ошибке) | — | `openFileStream` |

---

## Сводка: какую процедуру выбрать

- Нужно прочитать содержимое файла построчно → `openFileStream` + `lines`.
- Нужно писать бинарные данные в память и потом отдать их как строку →
  `newStringStream("")` + `write`/`writeLine` + поле `.data`.
- Нужно узнать формат/заголовок данных, не трогая позицию → `peekStr`/`peekChar`/`peek[T]`.
- Нужно прочитать ровно N байт как строку, не заботясь о переполнении →
  `readStr(s, N)` — вернёт укороченную строку вместо исключения.
- Нужно читать типизированные числа (сериализованные через `write`) → `readInt*`
  /`readUint*`/`readFloat*` в том же порядке, в каком они были записаны.
- Нужно открыть файл и не проверять `nil` вручную → `openFileStream` (кинет
  `IOError`); если `nil` — это ожидаемый и обрабатываемый случай, берите
  `newFileStream`.
- Нужно скопировать поток целиком за один вызов → `readAll`.
- Нужно скопировать поток по частям с контролем размера буфера →
  `readData`/`writeData` в цикле (см. рецепт "Копирование содержимого").
- Нужно гарантировать, что данные реально попали на диск прямо сейчас →
  `flush`, не дожидаясь `close`.
