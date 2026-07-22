# streams — module reference

> **Import:** `import std/streams`
> **Scope:** an abstract stream (read/write) interface, with two ready-made implementations — over a file (`FileStream`) and over a string (`StringStream`).

The module solves one problem: give a single API for reading and writing data,
regardless of where that data actually comes from — a file on disk or a string
in memory. Everything is built around the base `Stream` type, which holds a set
of procedure fields (`closeImpl`, `readDataImpl`, etc.) — each concrete stream
implementation (`FileStream`, `StringStream`) plugs its own procedures into
those slots. The module's common conventions: — a "read.../peek..." pair —
`read...` reads and advances the stream position, `peek...` reads the same data
but rewinds the position afterward (a look without consuming); — low-level
procedures (`readData`, `writeData`) work with raw `pointer` buffers, and the
typed procedures (`readInt32`, `readStr`, `readLine`, etc.) are built on top
of them.

---

## Table of Contents

I. [The Stream type and how it's built](#the-stream-type-and-how-its-built)
   1. [`Stream` / `StreamObj`](#stream--streamobj)
II. [Stream management: closing, flushing, position](#stream-management-closing-flushing-position)
   1. [`close`](#close)
   2. [`flush`](#flush)
   3. [`atEnd`](#atend)
   4. [`setPosition`](#setposition)
   5. [`getPosition`](#getposition)
III. [Low-level reads and writes through a buffer](#low-level-reads-and-writes-through-a-buffer)
   1. [`readData`](#readdata)
   2. [`peekData`](#peekdata)
   3. [`writeData`](#writedata)
IV. [Generic write and read for arbitrary types](#generic-write-and-read-for-arbitrary-types)
   1. [`write[T]` (generic write)](#writet-generic-write)
   2. [`write` (string)](#write-string)
   3. [`write` (varargs)](#write-varargs)
   4. [`writeLine`](#writeline)
   5. [`read[T]` (generic read)](#readt-generic-read)
   6. [`peek[T]` (generic peek)](#peekt-generic-peek)
V. [Reading and peeking typed numeric values](#reading-and-peeking-typed-numeric-values)
   1. [`readChar` / `peekChar`](#readchar--peekchar)
   2. [`readBool` / `peekBool`](#readbool--peekbool)
   3. [`readInt8/16/32/64` / `peekInt8/16/32/64`](#readint81632643264--peekint81632643264)
   4. [`readUint8/16/32/64` / `peekUint8/16/32/64`](#readuint8163264--peekuint8163264)
   5. [`readFloat32/64` / `peekFloat32/64`](#readfloat3264--peekfloat3264)
VI. [Reading strings of a given length](#reading-strings-of-a-given-length)
   1. [`readStr` / `peekStr`](#readstr--peekstr)
VII. [Line-oriented reading](#line-oriented-reading)
   1. [`readLine` / `peekLine`](#readline--peekline)
   2. [`lines` (iterator)](#lines-iterator)
VIII. [StringStream — a stream over a string](#stringstream--a-stream-over-a-string)
   1. [`newStringStream`](#newstringstream)
IX. [FileStream — a stream over a file](#filestream--a-stream-over-a-file)
   1. [`newFileStream` (from an open `File`)](#newfilestream-from-an-open-file)
   2. [`newFileStream` (from a filename)](#newfilestream-from-a-filename)
   3. [`openFileStream`](#openfilestream)
X. [Practical recipes](#practical-recipes)
XI. [Quick reference table](#quick-reference-table)
XII. [Summary: which procedure to pick](#summary-which-procedure-to-pick)

---

## The Stream type and how it's built

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

**What it does.** `Stream` is the single type that every procedure in the
module works with: both `FileStream` and `StringStream` ultimately reduce to
it through inheritance (`object of RootObj` / `object of StreamObj`).
`StreamObj` itself doesn't store any stream data — instead it stores a set of
procedure pointers (`closeImpl`, `readDataImpl`, etc.), which each concrete
implementation fills in with its own. This is a classic "virtual method
table" pattern, implemented by hand: calling `close(s)` internally just calls
`s.closeImpl(s)`, and which function actually ends up there depends on
whether `s` was created as a `FileStream` or a `StringStream`. The fields are
marked public (`*`) specifically so third-party modules can plug in their own
stream implementations that aren't part of the standard library.

- **`closeImpl`, `atEndImpl`, `setPositionImpl`, `getPositionImpl`,
  `flushImpl`** — the mandatory implementation procedures for basic
  operations.
- **`readDataStrImpl`, `readLineImpl`, `readDataImpl`, `peekDataImpl`,
  `writeDataImpl`** — read/write procedures; some of them (`readDataImpl`,
  `peekDataImpl`, `writeDataImpl`) are unavailable at compile time (in the VM)
  and on the JS backend, since they use raw `pointer`s.

```nim
# Not a runnable example on its own: shows what a "hand-rolled" stream
# looks like if you wanted to implement your own wrapper.
type
  MyStream = ref MyStreamObj
  MyStreamObj = object of StreamObj
    data: string
```

---

## Stream management: closing, flushing, position

### `close`

```nim
proc close*(s: Stream)
```

**What it does.** Closes the stream `s` by calling its `closeImpl`. For
`FileStream` this means closing the file descriptor; for `StringStream` it
clears the internal string (`data` becomes an empty string). Safe to call
again and safe on a `nil` stream: both checks (`isNil(s)`,
`isNil(s.closeImpl)`) happen before the call, so `close(nil)` simply does
nothing rather than crashing.

- **`s: Stream`** — the stream being closed; may be `nil`.

```nim
var strm = newStringStream("The first line\nthe second line")
# ... do something with the stream ...
close(strm)

# close is safe even if opening the stream failed
var maybeStrm = newFileStream("nonexistent_file.txt")
defer: close(maybeStrm)  # still works even if maybeStrm == nil
```

---

### `flush`

```nim
proc flush*(s: Stream)
```

**What it does.** Forces any buffered but not-yet-written data for stream `s`
out to the host environment — for example, from a `File`'s buffer to disk.
Until `flush` is called (or the stream is closed), data may not physically
have reached the file yet, even though `write` already ran.

- **`s: Stream`** — the stream whose buffers should be flushed.

```nim
var strm = newFileStream("output.txt", fmWrite)
write(strm, "hello")
# before flush() the data may not be on disk yet
flush(strm)
# now it's guaranteed to be written
close(strm)
```

---

### `atEnd`

```nim
proc atEnd*(s: Stream): bool
```

**What it does.** Checks whether the end of the stream has been reached —
that is, whether there's anything left to read. Returns `true` once all data
has been read.

- **`s: Stream`** — the stream being checked (conceptually read-only for this
  operation).

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

**What it does.** Sets the current read/write position in the stream to
`pos` (zero-based, counted from the start). For `StringStream` the position
is clamped into the `0..len(data)` range; for `FileStream` it corresponds to
an offset inside the file.

- **`s: Stream`** — the stream whose position is being changed.
- **`pos: int`** — the new position (byte index from the start of the
  stream).

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

**What it does.** Returns the current read/write position in the stream —
i.e. how many bytes have already been "passed" from the start.

- **`s: Stream`** — the stream whose position is being queried.

```nim
var strm = newStringStream("The first line\nthe second line")
doAssert getPosition(strm) == 0
discard readLine(strm)
doAssert getPosition(strm) == 15
close(strm)
```

---

## Low-level reads and writes through a buffer

The three procedures in this section are the foundation that every other
typed read/write operation in the module is built on (`readInt32`, `readStr`,
`write`, etc.). They work directly with `pointer` and a raw byte count, so
they're unavailable in the VM (at compile time) without going through
`readDataStr`.

### `readData`

```nim
proc readData*(s: Stream, buffer: pointer, bufLen: int): int
```

**What it does.** Reads at most `bufLen` bytes from stream `s` into `buffer`
and advances the stream's position by however many bytes were actually read.
Returns the actual number of bytes read — it can be smaller than `bufLen` if
the stream ended earlier.

- **`s: Stream`** — the source to read from.
- **`buffer: pointer`** — a pointer to the memory region the read data is
  written into; the caller is responsible for making sure enough space is
  allocated there.
- **`bufLen: int`** — the maximum number of bytes to read.

```nim
var strm = newStringStream("abcde")
var buffer: array[6, char]
doAssert readData(strm, addr(buffer), 1024) == 5  # read fewer than requested
doAssert buffer == ['a', 'b', 'c', 'd', 'e', '\x00']
doAssert atEnd(strm) == true
close(strm)
```

---

### `peekData`

```nim
proc peekData*(s: Stream, buffer: pointer, bufLen: int): int
```

**What it does.** The same as `readData`, but doesn't move the stream's
position — the data can be "peeked at" and read again by this or another
call.

- **`s: Stream`** — the source being peeked at.
- **`buffer: pointer`** — the buffer the peeked data is written into.
- **`bufLen: int`** — the maximum number of bytes to peek.

```nim
var strm = newStringStream("abcde")
var buffer: array[6, char]
doAssert peekData(strm, addr(buffer), 1024) == 5
doAssert buffer == ['a', 'b', 'c', 'd', 'e', '\x00']
doAssert atEnd(strm) == false  # the position did not move
close(strm)
```

---

### `writeData`

```nim
proc writeData*(s: Stream, buffer: pointer, bufLen: int)
```

**What it does.** Writes exactly `bufLen` bytes from `buffer` into stream `s`,
advancing the position forward. Unlike reading, there's no "partial" write
here — either all the bytes get written, or (for `FileStream`) an `IOError`
is raised.

- **`s: Stream`** — the destination being written to.
- **`buffer: pointer`** — a pointer to the data to write.
- **`bufLen: int`** — how many bytes to write.

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

## Generic write and read for arbitrary types

This section is a convenient typed wrapper over `writeData`/`readData`:
instead of manually computing `sizeof` and casting pointers, you can
write/read a value of any type `T` with a single procedure.

### `write[T]` (generic write)

```nim
proc write*[T](s: Stream, x: T)
```

**What it does.** Writes a value `x` of an arbitrary type `T` into stream `s`
"as is" — byte for byte, over `sizeof(x)` bytes. Unavailable on the JS
backend (use the `write(Stream, string)` overload instead).

- **`s: Stream`** — the destination being written to.
- **`x: T`** — the value of any type being written.

```nim
var strm = newStringStream("")
write(strm, "abcde")
setPosition(strm, 0)
doAssert readAll(strm) == "abcde"
close(strm)
```

---

### `write` (string)

```nim
proc write*(s: Stream, x: string)
```

**What it does.** Writes string `x` into the stream "as is" — with no length
field and no terminating zero byte.

- **`s: Stream`** — the destination being written to.
- **`x: string`** — the string being written.

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

**What it does.** Writes one or more strings in a row, with no separators and
no terminating zeros. Each argument is implicitly converted to a string via
`$`.

- **`s: Stream`** — the destination being written to.
- **`args: varargs[string, `$`]`** — any number of values, each converted to
  a string and written in sequence.

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

**What it does.** The same as `write` with multiple arguments, but appends a
newline character `\n` at the end.

- **`s: Stream`** — the destination being written to.
- **`args: varargs[string, `$`]`** — the values to write before the newline.

```nim
var strm = newStringStream("")
writeLine(strm, 1, 2)
writeLine(strm, 3, 4)
setPosition(strm, 0)
doAssert readAll(strm) == "12\n34\n"
close(strm)
```

---

### `read[T]` (generic read)

```nim
proc read*[T](s: Stream, result: var T)
```

**What it does.** Reads `sizeof(T)` bytes from the stream into the variable
`result`. If fewer bytes are read than needed to fill `T` (the stream ended
too early), it raises `IOError`.

- **`s: Stream`** — the source to read from.
- **`result: var T`** — the variable the read value is written into; the type
  `T` determines how many bytes are read.

```nim
var strm = newStringStream("012")
var i: int8
read(strm, i)
doAssert i == 48  # the character code of '0'
close(strm)
```

---

### `peek[T]` (generic peek)

```nim
proc peek*[T](s: Stream, result: var T)
```

**What it does.** The same as `read[T]`, but without moving the stream's
position — the value can be "peeked at" and read again afterward.

- **`s: Stream`** — the source being peeked at.
- **`result: var T`** — the variable the peeked value is written into.

```nim
var strm = newStringStream("012")
var i: int8
peek(strm, i)
doAssert i == 48
read(strm, i)
doAssert i == 48  # the same value is read again
close(strm)
```

---

## Reading and peeking typed numeric values

Every procedure in this section is built on the same pattern on top of
`read[T]` and `peek[T]`: a "reads and moves the position" / "reads without
moving it" pair. They only differ in the concrete type and, accordingly, the
number of bytes read.

### `readChar` / `peekChar`

```nim
proc readChar*(s: Stream): char
proc peekChar*(s: Stream): char
```

**What it does.** Reads (or peeks) a single character from the stream. If the
stream has already ended, instead of an exception a special marker character
`'\0'` is returned — the only reading procedure in the module that signals
the end of data with a value rather than an exception.

- **`s: Stream`** — the source to read from.

```nim
var strm = newStringStream("12\n3")
doAssert readChar(strm) == '1'
doAssert readChar(strm) == '2'
doAssert readChar(strm) == '\n'
doAssert readChar(strm) == '3'
doAssert readChar(strm) == '\x00'  # end of stream — a marker, not an exception
close(strm)
```

---

### `readBool` / `peekBool`

```nim
proc readBool*(s: Stream): bool
proc peekBool*(s: Stream): bool
```

**What it does.** Reads (or peeks) a single byte and interprets it as a
`bool`: any non-zero value is `true`, zero is `false`. Unlike `readChar`,
running past the end of the stream raises `IOError` rather than returning a
special value.

- **`s: Stream`** — the source to read from.

```nim
var strm = newStringStream()
write(strm, true)
write(strm, false)
flush(strm)
setPosition(strm, 0)
doAssert readBool(strm) == true
doAssert readBool(strm) == false
doAssertRaises(IOError): discard readBool(strm)  # the data ran out
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

**What it does.** Eight procedures with identical behavior, differing only in
the width of the signed integer being read (1, 2, 4, or 8 bytes). `read...`
advances the stream position by the corresponding number of bytes,
`peek...` doesn't. When there isn't enough data, `IOError` is raised.

- **`s: Stream`** — the source to read from; the width is picked by the
  procedure's name, not by an argument.

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

# peek does not move the position — the same value is read again
var strm2 = newStringStream()
write(strm2, 1'i16)
flush(strm2)
setPosition(strm2, 0)
doAssert peekInt16(strm2) == 1'i16
doAssert readInt16(strm2) == 1'i16  # the same value
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

**What it does.** A full analog of the previous section, but for unsigned
integers. Boundary behavior (end of stream → `IOError`) and the `read`/`peek`
semantics are identical to `readInt.../peekInt...`.

- **`s: Stream`** — the source to read from; the width is picked by the
  procedure's name.

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

**What it does.** The same principles as for integers, but for single- (4
bytes) or double-precision (8 bytes) floating-point numbers.

- **`s: Stream`** — the source to read from; the width is picked by the
  procedure's name.

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

## Reading strings of a given length

### `readStr` / `peekStr`

```nim
proc readStr*(s: Stream, length: int): string
proc readStr*(s: Stream, length: int, str: var string)
proc peekStr*(s: Stream, length: int): string
proc peekStr*(s: Stream, length: int, str: var string)
```

**What it does.** Reads (or peeks) exactly `length` bytes as a string. If less
data than requested remains in the stream, a shortened string is returned —
no exception is raised (unlike the typed numeric procedures). The version
with a `str: var string` parameter writes the result into an already-existing
string instead of allocating a new one; the version without it returns a new
string.

- **`s: Stream`** — the source to read from.
- **`length: int`** — how many bytes to request.
- **`str: var string`** *(optional)* — an existing destination string; if not
  supplied, the procedure allocates a new string itself.

```nim
var strm = newStringStream("abcde")
doAssert readStr(strm, 2) == "ab"
doAssert readStr(strm, 2) == "cd"
doAssert readStr(strm, 2) == "e"    # only 1 letter left — a shortened result
doAssert readStr(strm, 2) == ""     # the stream is empty
close(strm)

var strm2 = newStringStream("abcde")
doAssert peekStr(strm2, 2) == "ab"
doAssert peekStr(strm2, 2) == "ab"  # peeking again — the same result
doAssert readStr(strm2, 2) == "ab"  # now actually move the position
doAssert peekStr(strm2, 2) == "cd"
close(strm2)
```

---

## Line-oriented reading

### `readLine` / `peekLine`

```nim
proc readLine*(s: Stream, line: var string): bool
proc readLine*(s: Stream): string
proc peekLine*(s: Stream, line: var string): bool
proc peekLine*(s: Stream): string
```

**What it does.** Reads a single line of text, delimited by `LF` (`\n`) or
`CRLF` (`\r\n`); the newline character(s) themselves aren't included in the
result. The version with `line: var string` and a returned `bool` is
low-level: `false` means the stream has already ended and no new data
appeared (in that case `line` is left unchanged). The version that returns a
`string` directly is less efficient and raises `IOError` at the end of the
stream instead of returning an end-of-data flag. `peekLine` is the same as
`readLine`, but restores the stream's position afterward (implemented via
`getPosition`/`setPosition` wrapped around a call to `readLine`, not via a
separate mechanism).

- **`s: Stream`** — the source to read from.
- **`line: var string`** *(for the boolean form)* — the string the read line
  is written into; must not be `nil`.

```nim
var strm = newStringStream("The first line\nthe second line\nthe third line")
var line = ""
doAssert readLine(strm, line) == true
doAssert line == "The first line"
doAssert readLine(strm, line) == true
doAssert line == "the second line"
close(strm)

# the form without the line parameter raises at the end of the stream
var strm2 = newStringStream("only line")
doAssert readLine(strm2) == "only line"
doAssertRaises(IOError): discard readLine(strm2)
close(strm2)

# peekLine does not move the position
var strm3 = newStringStream("first\nsecond")
doAssert peekLine(strm3) == "first"
doAssert peekLine(strm3) == "first"  # the same line again
doAssert readLine(strm3) == "first"  # now actually move the position
doAssert peekLine(strm3) == "second"
close(strm3)
```

---

### `lines` (iterator)

```nim
iterator lines*(s: Stream): string
```

**What it does.** Walks sequentially through every line of the stream, using
`readLine` under the hood. A convenient wrapper for a `for line in
lines(strm)` loop instead of a manual `while readLine(strm, line)`.

- **`s: Stream`** — the stream whose lines are being walked.

```nim
var strm = newStringStream("The first line\nthe second line\nthe third line")
var collected: seq[string]
for line in lines(strm):
  add(collected, line)
doAssert collected == @["The first line", "the second line", "the third line"]
close(strm)
```

---

## StringStream — a stream over a string

### `newStringStream`

```nim
proc newStringStream*(s: sink string = ""): owned StringStream
```

**What it does.** Creates a new stream that reads and writes directly into
string `s` (or into a new empty string, if no argument is given). Every write
operation changes this stream's public `data` field — so at any moment you
can look at `StringStream.data` to see the accumulated result, without
waiting for `close`. The implementation reads and writes via `copyMem`
straight into the string's buffer — hence the efficiency compared to, say,
building the result up through line-by-line concatenation.

- **`s: sink string`** — the initial content of the stream; passed by
  ownership (`sink`), meaning the string is "taken over" by the stream
  without extra copying.

```nim
var strm = newStringStream("The first line\nthe second line\nthe third line")
doAssert readLine(strm) == "The first line"
doAssert readLine(strm) == "the second line"
doAssert readLine(strm) == "the third line"
close(strm)

# writing and then reading it back
var strm2 = newStringStream("")
writeLine(strm2, "hello")
doAssert strm2.data == "hello\n"  # data is a public field, accessible directly
close(strm2)
```

---

## FileStream — a stream over a file

**Note.** All three procedures in this section are unavailable on the JS
backend — native compilation only.

### `newFileStream` (from an open `File`)

```nim
proc newFileStream*(f: File): owned FileStream
```

**What it does.** Wraps an already-open file `f` (of type `File`, from the
`io`/`syncio` module) in a stream. Useful when a file needs to be opened with
non-standard parameters via `open`, and you want to work with it afterward
through the unified `Stream` interface.

- **`f: File`** — a file that's already been opened.

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

### `newFileStream` (from a filename)

```nim
proc newFileStream*(filename: string, mode: FileMode = fmRead,
                     bufSize: int = -1): owned FileStream
```

**What it does.** Opens file `filename` in mode `mode` and immediately wraps
it in a stream. **If the file can't be opened, it returns `nil`** — this must
be checked explicitly (`if not isNil(strm)`), otherwise subsequent operations
on the `nil` stream will crash. It's precisely because of this quirk that the
module's own documentation recommends `openFileStream` for cases where a
failed open should be handled as an exception rather than as `nil`.

- **`filename: string`** — the path to the file.
- **`mode: FileMode`** — the access mode (`fmRead`, `fmWrite`, etc.),
  defaulting to `fmRead`.
- **`bufSize: int`** — the I/O buffer size; `-1` means the default buffer.

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

**What it does.** Does the same thing as `newFileStream(filename, ...)`, but
on failure it doesn't return `nil` — it raises `IOError` instead. More
convenient in code where a failed file open should interrupt execution via an
exception rather than through a manual `nil` check at every step.

- **`filename: string`** — the path to the file.
- **`mode: FileMode`** — the access mode, defaulting to `fmRead`.
- **`bufSize: int`** — the buffer size, `-1` for the default value.

```nim
try:
  var strm = openFileStream("somefile.txt")
  echo readLine(strm)
  close(strm)
except:
  write(stderr, getCurrentExceptionMsg())
```

---

## Practical recipes

### Parsing a binary record into a string and back

A combination of `writeLine`/`write`/`readInt32`/`readStr` — a typical scheme
for serializing a simple binary format over a `StringStream`.

```nim
# Write a "record": an integer plus a fixed-length string.
var buf = newStringStream("")
write(buf, 42'i32)
write(buf, "abcd")  # exactly 4 bytes, no length prefix, no terminator
flush(buf)

# Read it back in the same order it was written.
setPosition(buf, 0)
let recordId = readInt32(buf)
let payload = readStr(buf, 4)
doAssert recordId == 42'i32
doAssert payload == "abcd"
close(buf)
```

---

### Line-by-line filtering of a large file without loading it whole

Uses `lines` together with `newFileStream`/`openFileStream`, so the whole
file doesn't need to be held in memory at once.

```nim
try:
  var input = openFileStream("access.log")
  var output = newStringStream("")
  for line in lines(input):
    if contains(line, "ERROR"):
      writeLine(output, line)
  close(input)
  echo readAll(output)  # only the error lines
  close(output)
except IOError:
  write(stderr, "failed to read access.log\n")
```

---

### Peeking at a header without "consuming" the stream

A combination of `peekStr`/`peekChar` followed by an actual read — a typical
pattern for recognizing a file's format from its first bytes (a "magic
number") before deciding which parser to use for the rest.

```nim
var strm = newStringStream("PNG\x89data...")
let header = peekStr(strm, 3)  # the position did not move
if header == "PNG":
  discard readStr(strm, 3)      # now actually skip the header
  let rest = readAll(strm)
  doAssert rest == "\x89data..."
close(strm)
```

---

### Copying one stream's contents into another in chunks

Uses the low-level `readData`/`writeData` directly — useful when you need
full control over the copy buffer size (for example, for large files you
don't want to read via `readAll` all at once).

```nim
proc copyStream(src, dst: Stream, chunkSize = 4096) =
  var chunk = newString(chunkSize)
  while true:
    let n = readData(src, addr(chunk[0]), chunkSize)
    if n == 0:
      break
    writeData(dst, addr(chunk[0]), n)

var source = newStringStream("some data to copy")
var target = newStringStream("")
copyStream(source, target)
doAssert target.data == source.data
close(source)
close(target)
```

---

## Quick reference table

| Task | Moves the position | What it returns / writes to |
|---|---|---|
| Close a stream | — | `close` |
| Flush the write buffer | — | `flush` |
| Check for end of data | no | `atEnd` → `bool` |
| Query/set the position | sets it | `getPosition` / `setPosition` |
| Read raw bytes into a buffer | yes | `readData` → byte count |
| Peek at raw bytes | no | `peekData` → byte count |
| Write raw bytes | yes | `writeData` |
| Write a value of an arbitrary type | yes | `write[T]` / `write(string)` / `write(varargs)` |
| Write string(s) with a newline | yes | `writeLine` |
| Read/peek a value of an arbitrary type | read: yes, peek: no | `read[T]` / `peek[T]` |
| Read/peek a single character | read: yes, peek: no | `readChar` / `peekChar` (end → `'\0'`) |
| Read/peek a bool or a number (int/uint/float) | read: yes, peek: no | `readBool`/`readInt*`/`readUint*`/`readFloat*` and their `peek` variants (end → `IOError`) |
| Read/peek a string of a given length | read: yes, peek: no | `readStr` / `peekStr` (shortened at the end, no exception) |
| Read/peek a single line of text | read: yes, peek: no | `readLine` / `peekLine` |
| Walk through every line | yes | `lines` (iterator) |
| A stream over an in-memory string | — | `newStringStream` |
| A stream over a file (no exception on failure) | — | `newFileStream` (may return `nil`) |
| A stream over a file (exception on failure) | — | `openFileStream` |

---

## Summary: which procedure to pick

- Need to read a file's contents line by line → `openFileStream` + `lines`.
- Need to write binary data in memory and then hand it back as a string →
  `newStringStream("")` + `write`/`writeLine` + the `.data` field.
- Need to identify a data format/header without touching the position →
  `peekStr`/`peekChar`/`peek[T]`.
- Need to read exactly N bytes as a string without worrying about
  overrunning → `readStr(s, N)` — returns a shortened string instead of
  raising an exception.
- Need to read typed numbers (serialized via `write`) → `readInt*`
  /`readUint*`/`readFloat*` in the same order they were written.
- Need to open a file without checking for `nil` by hand → `openFileStream`
  (raises `IOError`); if `nil` is an expected, handled case, use
  `newFileStream` instead.
- Need to copy an entire stream in one call → `readAll`.
- Need to copy a stream in chunks with control over buffer size →
  `readData`/`writeData` in a loop (see the "Copying stream contents"
  recipe).
- Need to guarantee data has actually hit disk right now → `flush`, without
  waiting for `close`.
