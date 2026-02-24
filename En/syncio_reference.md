# `syncio` Module Reference

> **Module purpose:** `syncio` implements synchronous, buffered file I/O — every operation blocks until it completes. It is a thin, high-fidelity wrapper around C's standard `<stdio.h>` library, exposed through an idiomatic Nim API. The module covers opening and closing files, reading and writing data in various forms (bytes, characters, lines, strings, typed values), navigating within a file, querying file metadata, and managing the standard streams (`stdin`, `stdout`, `stderr`).

---

## Table of Contents

1. [Types](#types)
   - [File](#file)
   - [FileMode](#filemode)
   - [FileSeekPos](#fileseekpos)
   - [FileHandle](#filehandle)
2. [Standard streams](#standard-streams)
   - [stdin, stdout, stderr](#stdin-stdout-stderr)
   - [stdmsg](#stdmsg)
3. [Opening and closing](#opening-and-closing)
   - [open (filename)](#open-filename)
   - [open (returning File)](#open-returning-file)
   - [open (FileHandle)](#open-filehandle)
   - [reopen](#reopen)
   - [close](#close)
4. [High-level file helpers](#high-level-file-helpers)
   - [readFile](#readfile)
   - [writeFile (string)](#writefile-string)
   - [writeFile (bytes)](#writefile-bytes)
   - [readLines](#readlines)
5. [Reading](#reading)
   - [readAll](#readall)
   - [readLine (mutating)](#readline-mutating)
   - [readLine (returning)](#readline-returning)
   - [readChar](#readchar)
   - [readBuffer](#readbuffer)
   - [readBytes](#readbytes)
   - [readChars](#readchars)
6. [Writing](#writing)
   - [write (overload family)](#write-overload-family)
   - [writeLine](#writeline)
   - [writeBuffer](#writebuffer)
   - [writeBytes](#writebytes)
   - [writeChars](#writechars)
   - [&= operator](#-operator)
7. [Iteration](#iteration)
   - [lines (filename)](#lines-filename)
   - [lines (File)](#lines-file)
8. [Position and size](#position-and-size)
   - [getFilePos](#getfilepos)
   - [setFilePos](#setfilepos)
   - [getFileSize](#getfilesize)
   - [endOfFile](#endoffile)
9. [Handles and inheritance](#handles-and-inheritance)
   - [getFileHandle](#getfilehandle)
   - [getOsFileHandle](#getosfilehandle)
   - [setInheritable](#setinheritable)
10. [Buffer control](#buffer-control)
    - [flushFile](#flushfile)
    - [setStdIoUnbuffered](#setstdiounbuffered)
11. [Error behaviour summary](#error-behaviour-summary)
12. [Practical patterns](#practical-patterns)

---

## Types

### `File`

```nim
type File* = ptr CFile
```

A handle to an open file. It is a pointer to C's opaque `FILE` struct, so it is `nil` when uninitialised or after an open failure. Most procs in this module take a `File` as their first argument.

Because `File` is a pointer, assignment copies only the handle — both variables then refer to the same underlying OS file. Always close exactly once.

---

### `FileMode`

```nim
type FileMode* = enum
  fmRead, fmWrite, fmReadWrite, fmReadWriteExisting, fmAppend
```

Controls how a file is opened. The five modes cover all standard POSIX/C combinations:

| Mode | File must exist? | Clears existing content? | Readable? | Writable? |
|---|---|---|---|---|
| `fmRead` | Yes | No | ✅ | ❌ |
| `fmWrite` | No (creates) | **Yes** | ❌ | ✅ |
| `fmReadWrite` | No (creates) | **Yes** | ✅ | ✅ |
| `fmReadWriteExisting` | Yes | No | ✅ | ✅ |
| `fmAppend` | No (creates) | No | ❌ | ✅ (end only) |

> **Warning:** `fmWrite` and `fmReadWrite` silently erase the contents of any existing file. Use `fmReadWriteExisting` or `fmAppend` to preserve data.

---

### `FileSeekPos`

```nim
type FileSeekPos* = enum
  fspSet,   ## absolute position from start of file
  fspCur,   ## position relative to current position
  fspEnd    ## position relative to end of file
```

Used as the `relativeTo` argument of `setFilePos`. Maps directly to POSIX `SEEK_SET`, `SEEK_CUR`, and `SEEK_END`. Negative offsets with `fspEnd` point backwards from the end.

---

### `FileHandle`

```nim
# POSIX:
type FileHandle* = cint
# Windows:
type FileHandle* = int   # HANDLE, convertible to winlean.Handle
```

A low-level OS file descriptor number (POSIX) or Windows HANDLE. Used when interoperating with OS-level APIs. Not the same as `File` — a `FileHandle` is just an integer, not a buffered stream.

---

## Standard streams

### `stdin`, `stdout`, `stderr`

```nim
var stdin*:  File   ## standard input
var stdout*: File   ## standard output
var stderr*: File   ## standard error
```

Pre-opened `File` values pointing to the process's standard streams, imported directly from C's `<stdio.h>`. On Windows, the module additionally switches all three to binary mode at startup, and console applications are configured to use UTF-8 (code page 65001).

These can be passed wherever a `File` is expected:

```nim
import std/syncio

write(stdout, "Hello, world\n")
let line = readLine(stdin)
write(stderr, "Warning: " & line & "\n")
```

---

### `stdmsg`

```nim
template stdmsg*: File
```

Expands to `stderr` by default, or to `stdout` if the program was compiled with `-d:useStdoutAsStdmsg`. Used internally by the runtime for diagnostic messages. Useful when you want a single symbol that your users can redirect at compile time.

```nim
import std/syncio

write(stdmsg, "This goes to stderr unless -d:useStdoutAsStdmsg\n")
```

---

## Opening and closing

### `open` (filename)

```nim
proc open*(f: var File, filename: string,
           mode: FileMode = fmRead,
           bufSize: int = -1): bool
```

Tries to open the file at `filename`. Returns `true` on success and fills `f`; returns `false` on failure without raising an exception. This is the non-throwing overload — prefer it when you want to handle open failures yourself.

`bufSize` controls the C-level I/O buffer:
- `-1` (default): use the system default buffer size
- `0`: disable buffering (unbuffered I/O, every byte goes to the OS immediately)
- `> 0`: use a buffer of exactly this many bytes

The resulting file handle is marked non-inheritable by child processes (unless `-d:nimInheritHandles` is set).

```nim
import std/syncio

var f: File
if open(f, "data.txt"):
  let content = readAll(f)
  close(f)
  echo content
else:
  echo "Could not open data.txt"
```

---

### `open` (returning `File`)

```nim
proc open*(filename: string,
           mode: FileMode = fmRead,
           bufSize: int = -1): File
```

Opens a file and **returns** the `File` handle directly. Raises `IOError` if the file cannot be opened. This is the throwing overload — it is more concise when you know the file should exist and an exception on failure is the right behaviour.

```nim
import std/syncio

let f = open("config.toml")   # raises IOError if missing
defer: close(f)
echo readAll(f)
```

---

### `open` (FileHandle)

```nim
proc open*(f: var File, filehandle: FileHandle,
           mode: FileMode = fmRead): bool
```

Wraps an existing OS file descriptor (from a system call, `pipe`, or `socket`) in a buffered `File`. Returns `true` on success. The handle is marked non-inheritable after wrapping. Useful for integrating low-level OS handles into Nim's buffered I/O layer.

```nim
import std/syncio

# Example: wrap file descriptor 3 (already open by parent process)
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

Redirects an already-open `File` to a different file on disk without closing and reopening separately. The classic use case is redirecting `stdin`, `stdout`, or `stderr` to a file, which is the POSIX/C idiom for I/O redirection inside a process.

```nim
import std/syncio

# Redirect stdout to a log file for everything that follows
if reopen(stdout, "app.log", fmWrite):
  echo "This goes to app.log, not the terminal"
```

---

### `close`

```nim
proc close*(f: File)
```

Closes the file, flushing any buffered data to the OS. After `close`, the `File` handle is invalid and must not be used again.

If compiled with `-d:nimPreviewCheckedClose`, `close` raises `IOError` on failure (e.g. if a write flush fails). Without that flag the error is silently ignored — the default, safe-for-most-uses behaviour.

```nim
import std/syncio

var f: File
if open(f, "output.txt", fmWrite):
  write(f, "Hello")
  close(f)   # flushes and releases the handle
```

---

## High-level file helpers

These procs open, operate on, and close a file in a single call — the most convenient way to handle simple file tasks.

### `readFile`

```nim
proc readFile*(filename: string): string
```

Reads the entire contents of the file at `filename` and returns it as a `string`. Opens the file, reads everything, closes it. Raises `IOError` if the file cannot be opened or read. Available at compile time via `staticRead`.

```nim
import std/syncio

let src = readFile("main.nim")
echo src.len, " bytes"
```

---

### `writeFile` (string)

```nim
proc writeFile*(filename, content: string)
```

Creates or overwrites the file at `filename` with `content`. Opens with `fmWrite` (erasing any previous contents), writes the entire string, then closes. Raises `IOError` on failure.

```nim
import std/syncio

writeFile("greeting.txt", "Hello, world!")
```

---

### `writeFile` (bytes)

```nim
proc writeFile*(filename: string, content: openArray[byte])
```

Same as the string overload but accepts raw bytes. Useful when writing binary data that should not be interpreted as text.

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

Reads exactly `n` lines from the file at `filename` and returns them as a `seq[string]`. Newline characters are stripped. Raises `IOError` if the file cannot be opened, or `EOFError` if it contains fewer than `n` lines.

```nim
import std/syncio

# Read a two-line header
let header = readLines("data.csv", 2)
echo header[0]   # first line
echo header[1]   # second line
```

---

## Reading

### `readAll`

```nim
proc readAll*(file: File): string
```

Reads everything from the current position to the end of `file` and returns it as a `string`. For regular files the file size is read first so a single allocation is made. For streams of unknown length (like `stdin`) it reads in chunks. Raises `IOError` on error.

It is an error to call `readAll` when the file position is not at the beginning — use `setFilePos` first if needed.

```nim
import std/syncio

let f = open("big.log")
defer: close(f)
let contents = readAll(f)
echo contents.len, " bytes read"
```

---

### `readLine` (mutating)

```nim
proc readLine*(f: File, line: var string): bool
```

Reads one line of text from `f` into `line`, stripping the trailing `LF` or `CRLF`. Returns `true` if a line was successfully read, `false` if the end of the file has been reached (in which case `line` is unchanged).

This overload reuses the `line` string's existing allocation, making it efficient in tight loops.

On Windows, when reading from an interactive console, `Ctrl+Z` is treated as EOF.

```nim
import std/syncio

let f = open("names.txt")
defer: close(f)

var line: string
while readLine(f, line):
  echo "Name: ", line
```

---

### `readLine` (returning)

```nim
proc readLine*(f: File): string
```

Reads one line from `f` and returns it. Raises `EOFError` if the end of the file has already been reached before the call. Use the mutating overload in performance-sensitive loops; use this one for clarity when reading a small number of lines.

```nim
import std/syncio

let f = open("config.txt")
defer: close(f)

let firstLine = readLine(f)
echo "First line: ", firstLine
```

---

### `readChar`

```nim
proc readChar*(f: File): char
```

Reads a single byte from `f` and returns it as a `char`. Raises `EOFError` at end of file. Not suitable for performance-sensitive code — use `readBuffer` or `readAll` when reading many bytes.

```nim
import std/syncio

let f = open("tiny.bin")
defer: close(f)

let first = readChar(f)
echo "First byte: ", ord(first)
```

---

### `readBuffer`

```nim
proc readBuffer*(f: File, buffer: pointer, len: Natural): int
```

The lowest-level read operation. Reads up to `len` bytes from `f` into the memory pointed to by `buffer`. Returns the number of bytes actually read, which may be less than `len` at end of file but never more. Uses `fread` directly.

```nim
import std/syncio

var buf: array[1024, byte]
let f = open("raw.bin")
defer: close(f)

let n = readBuffer(f, addr buf[0], buf.len)
echo n, " bytes read"
```

---

### `readBytes`

```nim
proc readBytes*(f: File, a: var openArray[int8|uint8], start, len: Natural): int
```

Reads `len` bytes from `f` into the slice `a[start ..< start+len]`. Returns the number of bytes actually read. A typed convenience wrapper around `readBuffer` for `int8`/`uint8` arrays.

```nim
import std/syncio

var buf = newSeq[uint8](256)
let f = open("data.bin")
defer: close(f)

let n = readBytes(f, buf, 0, 128)
echo n, " bytes read into first 128 slots"
```

---

### `readChars`

```nim
proc readChars*(f: File, a: var openArray[char]): int
```

Reads up to `a.len` bytes from `f` into the `char` buffer `a`. Returns the number of bytes actually read. Equivalent to `readBuffer` but typed for character arrays.

```nim
import std/syncio

var buf: array[64, char]
let f = open("text.txt")
defer: close(f)

let n = readChars(f, buf)
echo n, " chars read"
```

---

## Writing

### `write` overload family

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

Writes a single value — or any number of values via the `varargs` overload — to `f`. All overloads raise `IOError` on failure. The `varargs` overload converts each argument with `$` first, making it usable with any type that has `$` defined.

On Windows, string output routes through `fprintf` to correctly handle UTF-8 content including embedded null characters.

```nim
import std/syncio

let f = open("out.txt", fmWrite)
defer: close(f)

write(f, "Count: ")
write(f, 42)
write(f, "\n")
write(f, true)
write(f, "\n")

# varargs overload — all printed in sequence
write(f, "x=", 3.14, " done\n")
```

---

### `writeLine`

```nim
proc writeLine*[Ty](f: File, x: varargs[Ty, `$`])
```

Writes each element of `x` (converting with `$`) to `f`, then writes a single `"\n"`. Equivalent to calling `write` followed by `write(f, "\n")`, but more readable when printing complete lines.

```nim
import std/syncio

let f = open("log.txt", fmWrite)
defer: close(f)

writeLine(f, "Starting up")
writeLine(f, "Value: ", 99)
writeLine(f, "Done")
```

---

### `writeBuffer`

```nim
proc writeBuffer*(f: File, buffer: pointer, len: Natural): int
```

Writes exactly `len` bytes from the memory pointed to by `buffer` to `f`. Returns the number of bytes actually written, which may be less than `len` on error. The lowest-level write — use it when working with raw memory.

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

Writes `a[start ..< start+len]` to `f`. Returns bytes written. Typed convenience wrapper around `writeBuffer` for `int8`/`uint8` arrays.

```nim
import std/syncio

let payload: seq[uint8] = @[1'u8, 2, 3, 4, 5]
let f = open("payload.bin", fmWrite)
defer: close(f)

discard writeBytes(f, payload, 1, 3)  # writes bytes [2, 3, 4]
```

---

### `writeChars`

```nim
proc writeChars*(f: File, a: openArray[char], start, len: Natural): int
```

Writes `a[start ..< start+len]` to `f`. Returns bytes written. Identical to `writeBytes` but typed for `char` arrays — useful when working with fixed character buffers.

```nim
import std/syncio

var buf = ['H', 'e', 'l', 'l', 'o']
let f = open("chars.txt", fmWrite)
defer: close(f)

discard writeChars(f, buf, 0, 5)
```

---

### `&=` operator

```nim
template `&=`*(f: File, x: typed)
```

A shorthand alias for `write(f, x)`. Lets you use the familiar `&=` append operator on a `File` handle, which reads naturally when building output incrementally.

```nim
import std/syncio

let f = open("result.txt", fmWrite)
defer: close(f)

f &= "Hello"
f &= ", "
f &= "world!"
```

---

## Iteration

### `lines` (filename)

```nim
iterator lines*(filename: string): string
```

Opens the file at `filename`, iterates over every line (stripping newline characters), then closes the file automatically. Raises `IOError` if the file cannot be opened. The internal read buffer is 8 000 bytes, making it efficient for line-by-line processing of large files.

```nim
import std/syncio

var count = 0
for line in lines("access.log"):
  if "ERROR" in line:
    inc count
echo count, " error lines"
```

---

### `lines` (File)

```nim
iterator lines*(f: File): string
```

Same as the filename overload but operates on an already-open `File`. Does not close `f` when iteration ends — the caller is responsible for closing. Useful when you need to perform other operations on `f` before or after iterating.

```nim
import std/syncio

let f = open("data.csv")
defer: close(f)

# Skip header
discard readLine(f)

# Iterate over data rows
for row in f.lines:
  echo row
```

---

## Position and size

### `getFilePos`

```nim
proc getFilePos*(f: File): int64
```

Returns the current read/write position in bytes from the start of the file (zero-based). Raises `IOError` if the position cannot be retrieved (e.g. for non-seekable streams like pipes).

```nim
import std/syncio

let f = open("data.bin")
defer: close(f)

discard readChar(f)
echo getFilePos(f)   # 1 — one byte past the start
```

---

### `setFilePos`

```nim
proc setFilePos*(f: File, pos: int64, relativeTo: FileSeekPos = fspSet)
```

Moves the file pointer to `pos` bytes relative to `relativeTo`. Raises `IOError` on failure. After a successful `setFilePos`, the next read or write will happen at the new position.

```nim
import std/syncio

let f = open("data.bin")
defer: close(f)

setFilePos(f, 0, fspEnd)          # jump to end
let size = getFilePos(f)

setFilePos(f, -4, fspEnd)         # 4 bytes before end
let tail = readAll(f)

setFilePos(f, 0)                  # back to the beginning (fspSet is default)
```

---

### `getFileSize`

```nim
proc getFileSize*(f: File): int64
```

Returns the total size of `f` in bytes. Internally it seeks to the end, reads the position, then seeks back — so it temporarily moves the file pointer. The original position is restored before returning.

```nim
import std/syncio

let f = open("image.png")
defer: close(f)

echo getFileSize(f), " bytes"
```

---

### `endOfFile`

```nim
proc endOfFile*(f: File): bool
```

Returns `true` if `f`'s read position is at the end of the file. Internally reads one byte and pushes it back with `ungetc`, so the file position is unchanged. Slightly cheaper than `getFilePos == getFileSize` for this check.

```nim
import std/syncio

let f = open("stream.bin")
defer: close(f)

while not endOfFile(f):
  let ch = readChar(f)
  process(ch)
```

---

## Handles and inheritance

### `getFileHandle`

```nim
proc getFileHandle*(f: File): FileHandle
```

Returns the C library's file descriptor number for `f` (the integer passed to `fileno` / `_fileno` on Windows). On Windows this is **not** the Win32 `HANDLE` — use `getOsFileHandle` for that.

```nim
import std/syncio

let f = open("test.txt")
defer: close(f)

echo "fd = ", getFileHandle(f)   # e.g. 3
```

---

### `getOsFileHandle`

```nim
proc getOsFileHandle*(f: File): FileHandle
```

Returns the true OS-level handle: on POSIX this is the same integer as `getFileHandle`; on Windows it is the actual Win32 `HANDLE` obtained from `_get_osfhandle`. Use this when calling OS APIs that require the raw handle.

```nim
import std/syncio

let f = open("test.txt")
defer: close(f)

let handle = getOsFileHandle(f)
# pass `handle` to a Win32 API or POSIX syscall
```

---

### `setInheritable`

```nim
proc setInheritable*(f: FileHandle, inheritable: bool): bool
```

Controls whether the OS file handle `f` is inherited by child processes when this process calls `fork`/`exec` (POSIX) or `CreateProcess` (Windows). Returns `true` on success.

By default, all files opened by `syncio` are already marked non-inheritable. Use `setInheritable(handle, true)` only when you specifically need a child process to inherit an open file.

Not available on all platforms — check with `declared(setInheritable)`.

```nim
import std/syncio

let f = open("shared.txt")
defer: close(f)

# Make this file inheritable before spawning a child
if declared(setInheritable):
  discard setInheritable(getOsFileHandle(f), true)
```

---

## Buffer control

### `flushFile`

```nim
proc flushFile*(f: File)
```

Forces any data in `f`'s write buffer to be sent to the OS. This does **not** guarantee the data has been written to physical storage — for that, use OS-level `fsync`. Useful before a crash-sensitive operation, before reading a file you just wrote, or when piping output to another process that needs to see it immediately.

```nim
import std/syncio

write(stdout, "Enter your name: ")
flushFile(stdout)   # ensure the prompt appears before blocking on input
let name = readLine(stdin)
```

---

### `setStdIoUnbuffered`

```nim
proc setStdIoUnbuffered*()
```

Disables the C-level buffers on `stdin`, `stdout`, and `stderr` (`setvbuf(f, nil, _IONBF, 0)`). After this call, every `write` goes directly to the OS without accumulating in a buffer. Useful for real-time logging, interactive tools, or when output must be immediately visible to a monitoring process.

Note that unbuffered I/O is significantly slower for large volumes of data — disable buffering only when latency matters more than throughput.

```nim
import std/syncio

setStdIoUnbuffered()   # all stdout/stderr writes now immediate
echo "Line 1"          # appears instantly even if stdout is redirected
echo "Line 2"
```

---

## Error behaviour summary

| Operation | On failure |
|---|---|
| `open(f, filename, ...)` | Returns `false`, no exception |
| `open(filename, ...)` → `File` | Raises `IOError` |
| `reopen` | Returns `false` |
| `close` (default) | Silently ignores errors |
| `close` (`-d:nimPreviewCheckedClose`) | Raises `IOError` |
| `readLine(f, line)` → `bool` | Returns `false` at EOF; raises `IOError` on error |
| `readLine(f)` → `string` | Raises `EOFError` at EOF; raises `IOError` on error |
| `readChar` | Raises `EOFError` at EOF; raises `IOError` on error |
| `readAll`, `readFile`, `readLines` | Raise `IOError` |
| `write*`, `writeLine`, `writeFile` | Raise `IOError` |
| `setFilePos`, `getFilePos` | Raise `IOError` |

---

## Practical patterns

### Pattern 1 — Safe open with `defer`

```nim
import std/syncio

let f = open("data.txt")   # raises IOError if missing
defer: close(f)            # guaranteed to close, even on exception

for line in f.lines:
  echo line
```

---

### Pattern 2 — Non-throwing open with manual error handling

```nim
import std/syncio

var f: File
if not open(f, "optional.cfg"):
  echo "Config not found, using defaults"
else:
  defer: close(f)
  echo readAll(f)
```

---

### Pattern 3 — Appending to a log file

```nim
import std/syncio
import std/times

proc logEvent(filename, message: string) =
  var f: File
  if open(f, filename, fmAppend):
    defer: close(f)
    writeLine(f, $now(), " ", message)

logEvent("events.log", "Server started")
logEvent("events.log", "Connection accepted")
```

---

### Pattern 4 — Reading a binary file in chunks

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

### Pattern 5 — In-place file seek for tail reading

```nim
import std/syncio

proc tail(filename: string, bytes: int64): string =
  let f = open(filename)
  defer: close(f)

  let size = getFileSize(f)
  let offset = if size < bytes: 0'i64 else: size - bytes
  setFilePos(f, offset)
  result = readAll(f)

echo tail("big.log", 512)   # last 512 bytes
```

---

### Pattern 6 — Redirect stdout to a file and back

```nim
import std/syncio

# Temporarily redirect stdout
if reopen(stdout, "capture.txt", fmWrite):
  echo "This goes to capture.txt"
  flushFile(stdout)
  discard reopen(stdout, "/dev/tty", fmWrite)   # restore (POSIX)

echo "This goes to the terminal again"
```
