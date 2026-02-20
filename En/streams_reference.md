# `std/streams` Module Reference (Nim)

> This module provides a **stream interface** and two concrete implementations:
> - `StringStream` — an in-memory stream backed by a `string`
> - `FileStream` — a stream backed by a `File` object
>
> Basic flow: **open → read/write → close**.
>
> ⚠️ `readData`, `peekData`, and `writeData` are unavailable on the compile-time VM, and require a `cast[ptr string]` on the JS backend. Use `readDataStr` as a portable alternative to `readData`.

---

## Table of Contents

1. [Types](#types)
2. [Creating Streams](#creating-streams)
3. [Stream Control](#stream-control)
4. [Positioning](#positioning)
5. [Writing Data](#writing-data)
6. [Low-Level Read and Write](#low-level-read-and-write)
7. [Reading Typed Values](#reading-typed-values)
8. [Peeking Without Advancing](#peeking-without-advancing)
9. [Reading Lines](#reading-lines)
10. [Iterator](#iterator)
11. [Read/Peek Quick Reference Table](#readpeek-quick-reference-table)

---

## Types

### `Stream`

```nim
type Stream* = ref StreamObj
```

The primary type used by all procedures in this module. It is a reference to a `StreamObj`.

---

### `StreamObj`

```nim
type StreamObj* = object of RootObj
```

The underlying stream object, containing function-pointer fields that define the stream's behaviour. **Do not use these fields directly** — they are exposed so that custom stream implementations can override them.

| Field | Type | Purpose |
|---|---|---|
| `closeImpl` | `proc(s: Stream)` | Close the stream |
| `atEndImpl` | `proc(s: Stream): bool` | Check end-of-stream |
| `setPositionImpl` | `proc(s: Stream, pos: int)` | Set byte position |
| `getPositionImpl` | `proc(s: Stream): int` | Get byte position |
| `readDataStrImpl` | `proc(s, buffer, slice): int` | Read into a string slice |
| `readLineImpl` | `proc(s, line): bool` | Read a text line |
| `readDataImpl` | `proc(s, buffer, bufLen): int` | Read into a raw buffer |
| `peekDataImpl` | `proc(s, buffer, bufLen): int` | Peek without advancing |
| `writeDataImpl` | `proc(s, buffer, bufLen)` | Write from a raw buffer |
| `flushImpl` | `proc(s: Stream)` | Flush write buffers |

---

### `StringStream`

```nim
type StringStream* = ref StringStreamObj
```

A stream that wraps an in-memory string. The `data` field is public and can be read directly after writing.

---

### `StringStreamObj`

```nim
type StringStreamObj* = object of StreamObj
  data*: string  ## The string backing this stream
```

---

### `FileStream`

```nim
type FileStream* = ref FileStreamObj
```

A stream that wraps a `File`. **Not available on the JS backend.**

---

### `FileStreamObj`

```nim
type FileStreamObj* = object of Stream
```

Contains a private `f: File` field.

---

## Creating Streams

### `newStringStream(s: string = ""): owned StringStream`

Creates a new stream from the string `s`. The string is copied into the stream. If `s` is omitted, an empty writable stream is created.

```nim
import std/streams

# Empty stream for writing
var ws = newStringStream()
ws.write("hello")
ws.setPosition(0)
echo ws.readAll()   # "hello"
ws.close()

# Stream for reading from an existing string
var rs = newStringStream("line one\nline two")
echo rs.readLine()  # "line one"
rs.close()
```

---

### `newFileStream(f: File): owned FileStream`

Creates a stream from an already-opened `File` object. Ownership of the file (opening/closing) remains with the caller until `close` is called on the stream.

> **Not available on the JS backend.**

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

Creates a stream by opening the named file. If the file cannot be opened, returns **`nil`** — no exception is raised.

> ⚠️ **Always check the return value for `nil`.** To get an exception on failure, use `openFileStream` instead.
>
> **Not available on the JS backend.**

| Parameter | Description |
|---|---|
| `filename` | Path to the file |
| `mode` | `fmRead`, `fmWrite`, `fmReadWrite`, `fmAppend`, etc. |
| `bufSize` | I/O buffer size (`-1` = system default) |

```nim
import std/streams

var strm = newFileStream("log.txt", fmWrite)
if not isNil(strm):
  strm.writeLine("Started")
  strm.close()
else:
  echo "Could not open file"
```

---

### `openFileStream(filename: string, mode: FileMode = fmRead, bufSize: int = -1): owned FileStream`

Creates a stream by opening the named file. If the file cannot be opened, **raises `IOError`**.

> **Not available on the JS backend.**

```nim
import std/streams

try:
  var strm = openFileStream("config.txt")
  echo strm.readLine()
  strm.close()
except IOError as e:
  echo "Error: ", e.msg
```

---

## Stream Control

### `close(s: Stream)`

Closes the stream and releases any associated resources. For a `FileStream`, the underlying file is also closed. Safe to call even if `s` is `nil`.

```nim
import std/streams

let strm = newStringStream("data")
defer: strm.close()
echo strm.readLine()
```

---

### `flush(s: Stream)`

Flushes any buffered writes to the underlying storage. Has no effect on `StringStream`. Important for `FileStream` when you need data written immediately without closing.

```nim
import std/streams

var strm = newFileStream("out.txt", fmWrite)
strm.write("hello")
strm.flush()   # Data is guaranteed to be on disk
strm.write("world")
strm.close()   # close also flushes
```

---

### `atEnd(s: Stream): bool`

Returns `true` if all data has been read (the stream is at its end).

```nim
import std/streams

var strm = newStringStream("abc")
echo strm.atEnd()          # false
discard strm.readAll()
echo strm.atEnd()          # true
strm.close()
```

---

## Positioning

### `setPosition(s: Stream, pos: int)`

Sets the current read/write position to byte offset `pos` from the beginning of the stream. For `StringStream`, the position is clamped to `[0, data.len]`.

```nim
import std/streams

var strm = newStringStream("Hello, world!")
strm.setPosition(7)
echo strm.readStr(5)   # "world"
strm.setPosition(0)
echo strm.readStr(5)   # "Hello"
strm.close()
```

---

### `getPosition(s: Stream): int`

Returns the current byte position in the stream.

```nim
import std/streams

var strm = newStringStream("Hello\nWorld")
echo strm.getPosition()     # 0
discard strm.readLine()
echo strm.getPosition()     # 6
strm.close()
```

---

## Writing Data

### `write[T](s: Stream, x: T)`

Generic write. Writes the binary representation of `x` to the stream. The number of bytes written equals `sizeof(x)`.

> **Not available on the JS backend.** Use `write(Stream, string)` instead.

```nim
import std/streams

var strm = newStringStream()
strm.write(42'i32)     # writes 4 bytes
strm.write(3.14'f64)   # writes 8 bytes
strm.setPosition(0)
echo strm.readInt32()    # 42
echo strm.readFloat64()  # 3.14
strm.close()
```

---

### `write(s: Stream, x: string)`

Writes the string `x` to the stream. No length prefix or null terminator is written.

```nim
import std/streams

var strm = newStringStream()
strm.write("Hello")
strm.write(", ")
strm.write("World!")
strm.setPosition(0)
echo strm.readAll()   # "Hello, World!"
strm.close()
```

---

### `write(s: Stream, args: varargs[string, `$`])`

Writes multiple values to the stream. Each value is converted to a string via `$` before writing.

```nim
import std/streams

var strm = newStringStream()
strm.write(1, 2, 3, " done")
strm.setPosition(0)
echo strm.readLine()   # "123 done"
strm.close()
```

---

### `writeLine(s: Stream, args: varargs[string, `$`])`

Writes one or more values to the stream followed by a newline `\n`.

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

## Low-Level Read and Write

### `readData(s: Stream, buffer: pointer, bufLen: int): int`

Reads up to `bufLen` bytes into the untyped `buffer`. Returns the actual number of bytes read (may be less than `bufLen` at end-of-stream).

> On the JS backend, `buffer` is treated as a `ptr string`.

```nim
import std/streams

var strm = newStringStream("abcde")
var buf: array[8, char]
let n = strm.readData(addr(buf), 8)
echo n       # 5
echo buf[0]  # 'a'
strm.close()
```

---

### `readDataStr(s: Stream, buffer: var string, slice: Slice[int]): int`

Reads data into the string `buffer` at byte positions specified by `slice`. Returns the number of bytes read. **Available on all backends**, including VM and JS — use this instead of `readData` for portable code.

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

Reads all remaining data from the stream into a new string.

```nim
import std/streams

var strm = newStringStream("The first line\nthe second line\nthe third line")
echo strm.readAll()
# The first line
# the second line
# the third line
echo strm.atEnd()   # true
strm.close()
```

---

### `peekData(s: Stream, buffer: pointer, bufLen: int): int`

Reads up to `bufLen` bytes into `buffer` **without advancing the stream position**. Returns the number of bytes read.

```nim
import std/streams

var strm = newStringStream("abcde")
var buf: array[6, char]
echo strm.peekData(addr(buf), 5)  # 5
echo buf[0]                        # 'a'
echo strm.getPosition()            # 0 — position unchanged
strm.close()
```

---

### `writeData(s: Stream, buffer: pointer, bufLen: int)`

Writes `bufLen` bytes from the untyped `buffer` into the stream.

```nim
import std/streams

var strm = newStringStream()
var buf = ['H', 'i', '!']
strm.writeData(addr(buf), 3)
strm.setPosition(0)
echo strm.readStr(3)   # "Hi!"
strm.close()
```

---

### `read[T](s: Stream, result: var T)`

Generic typed read. Reads `sizeof(T)` bytes into `result`. Raises `IOError` if fewer bytes are available.

> **Not available on the JS backend.**

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
echo a   # 255
echo b   # -1
strm.close()
```

---

### `peek[T](s: Stream, result: var T)`

Generic typed peek. Reads `sizeof(T)` bytes into `result` **without advancing the position**. Raises `IOError` if fewer bytes are available.

> **Not available on the JS backend.**

```nim
import std/streams

var strm = newStringStream()
strm.write(42'i32)
strm.setPosition(0)
var x: int32
strm.peek(x)
echo x                   # 42
echo strm.getPosition()  # 0
strm.close()
```

---

## Reading Typed Values

All procedures below read a specific binary type from the stream using the platform's native byte order. They raise `IOError` on failure.

> ⚠️ **Not available on the JS backend.** Use `readStr` for JS.

### Characters

#### `readChar(s: Stream): char`

Reads one byte as a `char`. Returns `'\0'` at end-of-stream instead of raising an exception.

```nim
import std/streams

var strm = newStringStream("Hi!")
echo strm.readChar()   # 'H'
echo strm.readChar()   # 'i'
echo strm.readChar()   # '!'
echo strm.readChar()   # '\x00' (EOF)
strm.close()
```

---

### Signed integers

| Procedure | Type | Size |
|---|---|---|
| `readInt8(s)` | `int8` | 1 byte |
| `readInt16(s)` | `int16` | 2 bytes |
| `readInt32(s)` | `int32` | 4 bytes |
| `readInt64(s)` | `int64` | 8 bytes |

```nim
import std/streams

var strm = newStringStream()
strm.write(100'i8)
strm.write(1000'i16)
strm.write(100000'i32)
strm.setPosition(0)
echo strm.readInt8()    # 100
echo strm.readInt16()   # 1000
echo strm.readInt32()   # 100000
strm.close()
```

---

### Unsigned integers

| Procedure | Type | Size |
|---|---|---|
| `readUint8(s)` | `uint8` | 1 byte |
| `readUint16(s)` | `uint16` | 2 bytes |
| `readUint32(s)` | `uint32` | 4 bytes |
| `readUint64(s)` | `uint64` | 8 bytes |

```nim
import std/streams

var strm = newStringStream()
strm.write(200'u8)
strm.write(60000'u16)
strm.setPosition(0)
echo strm.readUint8()    # 200
echo strm.readUint16()   # 60000
strm.close()
```

---

### Floating-point numbers

| Procedure | Type | Size |
|---|---|---|
| `readFloat32(s)` | `float32` | 4 bytes |
| `readFloat64(s)` | `float64` | 8 bytes |

```nim
import std/streams

var strm = newStringStream()
strm.write(1.5'f32)
strm.write(3.14'f64)
strm.setPosition(0)
echo strm.readFloat32()   # 1.5
echo strm.readFloat64()   # 3.14
strm.close()
```

---

### Booleans

#### `readBool(s: Stream): bool`

Reads 1 byte. Any non-zero value is returned as `true`.

```nim
import std/streams

var strm = newStringStream()
strm.write(true)
strm.write(false)
strm.setPosition(0)
echo strm.readBool()   # true
echo strm.readBool()   # false
strm.close()
```

---

### Fixed-length strings

#### `readStr(s: Stream, length: int): string`

Reads exactly `length` bytes from the stream and returns them as a string. If fewer bytes are available, a shorter string is returned.

```nim
import std/streams

var strm = newStringStream("abcde")
echo strm.readStr(2)   # "ab"
echo strm.readStr(2)   # "cd"
echo strm.readStr(2)   # "e"   (less than 2 bytes remain)
echo strm.readStr(2)   # ""    (EOF)
strm.close()
```

#### `readStr(s: Stream, length: int, str: var string)`

In-place overload: stores the result in the existing `str` variable.

---

## Peeking Without Advancing

All `peek` procedures read data **without changing the current stream position**. Their signatures mirror the corresponding `read` procedures.

### `peekChar(s: Stream): char`

```nim
import std/streams

var strm = newStringStream("AB")
echo strm.peekChar()   # 'A'
echo strm.peekChar()   # 'A' (position unchanged)
echo strm.readChar()   # 'A'
echo strm.peekChar()   # 'B'
strm.close()
```

---

### `peekBool(s: Stream): bool`

Peeks 1 byte as a `bool` without advancing.

---

### Signed integers (peek)

`peekInt8`, `peekInt16`, `peekInt32`, `peekInt64`

```nim
import std/streams

var strm = newStringStream()
strm.write(42'i32)
strm.setPosition(0)
echo strm.peekInt32()   # 42
echo strm.peekInt32()   # 42  (position still 0)
echo strm.readInt32()   # 42
strm.close()
```

---

### Unsigned integers (peek)

`peekUint8`, `peekUint16`, `peekUint32`, `peekUint64`

---

### Floating-point numbers (peek)

`peekFloat32`, `peekFloat64`

---

### Strings (peek)

#### `peekStr(s: Stream, length: int): string`

Reads `length` bytes **without advancing the position**.

```nim
import std/streams

var strm = newStringStream("abcde")
echo strm.peekStr(2)   # "ab"
echo strm.peekStr(2)   # "ab" (again)
echo strm.readStr(2)   # "ab"
echo strm.peekStr(2)   # "cd"
strm.close()
```

#### `peekStr(s: Stream, length: int, str: var string)`

In-place overload.

---

## Reading Lines

### `readLine(s: Stream, line: var string): bool`

Reads a text line into `line`. Lines may be delimited by `LF` or `CRLF`; the delimiter is **not** included in `line`. Returns `false` at EOF — in that case `line` is set to `""`.

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

Reads a line and returns it. Raises `IOError` at EOF.

> ⚠️ Less efficient than the `var string` overload.

```nim
import std/streams

var strm = newStringStream("A\nB\nC")
echo strm.readLine()   # "A"
echo strm.readLine()   # "B"
echo strm.readLine()   # "C"
# strm.readLine() now raises IOError
strm.close()
```

---

### `peekLine(s: Stream, line: var string): bool`

Reads a line into `line` **without advancing the position**. Uses `defer` + `setPosition` internally to restore the position.

```nim
import std/streams

var strm = newStringStream("First\nSecond")
var line = ""
echo strm.peekLine(line)   # true
echo line                   # "First"
echo strm.peekLine(line)   # true
echo line                   # "First" (position unchanged)
echo strm.readLine(line)   # true
echo line                   # "First"
strm.close()
```

---

### `peekLine(s: Stream): string`

Reads a line and returns it **without advancing the position**.

```nim
import std/streams

var strm = newStringStream("First\nSecond")
echo strm.peekLine()   # "First"
echo strm.peekLine()   # "First"
echo strm.readLine()   # "First"
echo strm.peekLine()   # "Second"
strm.close()
```

---

## Iterator

### `iterator lines(s: Stream): string`

Iterates over every line in the stream using `readLine`. A convenient replacement for a `while readLine` loop.

```nim
import std/streams

var strm = newStringStream("Line 1\nLine 2\nLine 3")
var collected: seq[string]
for line in strm.lines():
  collected.add(line)
echo collected   # @["Line 1", "Line 2", "Line 3"]
strm.close()
```

---

## Read/Peek Quick Reference Table

| Data type | Read (advances) | Peek (no advance) |
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
| fixed-length string | `readStr(s, n)` | `peekStr(s, n)` |
| text line | `readLine(s)` | `peekLine(s)` |
| generic `T` | `read[T](s, result)` | `peek[T](s, result)` |
| raw buffer | `readData(s, buf, n)` | `peekData(s, buf, n)` |
| string buffer | `readDataStr(s, buf, sl)` | — |
| all remaining | `readAll(s)` | — |

---

## Complete Examples

### Binary struct serialization

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
echo readPoint(strm)   # (x: 1.5, y: -2.0)
echo readPoint(strm)   # (x: 3.0, y: 4.5)
strm.close()
```

---

### Reading a CSV file line by line

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

### Safe file open with defer

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

### Writing and reading mixed types

```nim
import std/streams

var strm = newStringStream()
strm.write(42'u16)         # 2 bytes
strm.write(true)            # 1 byte
strm.writeLine("end")      # string + \n

strm.setPosition(0)
echo strm.readUint16()     # 42
echo strm.readBool()       # true
echo strm.readLine()       # "end"
strm.close()
```

---

### Using `peekChar` to dispatch on next token

```nim
import std/streams

var strm = newStringStream("42 hello true")

proc nextToken(s: Stream): string =
  result = ""
  while not s.atEnd() and s.peekChar() != ' ':
    result.add(s.readChar())
  if not s.atEnd(): discard s.readChar()  # consume space

echo nextToken(strm)   # "42"
echo nextToken(strm)   # "hello"
echo nextToken(strm)   # "true"
strm.close()
```

---

### Implementing a custom stream

```nim
import std/streams

# Example: a stream that counts bytes written
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
echo cs.bytesWritten   # 13
cs.close()
```
