# Nim `std/asyncfile` Module Reference

> Asynchronous file I/O for Nim's `async`/`await` ecosystem. This module lets you read from and write to files without blocking the event loop — meaning other asynchronous tasks (network requests, timers, etc.) continue running while the file operation is in progress.
>
> Internally the module uses **IOCP** (I/O Completion Ports) on Windows and **non-blocking POSIX file descriptors** on Unix-like systems, so the mechanics differ by platform, but the public API is identical on all platforms.
>
> Requires `std/asyncdispatch` to be imported and an event loop to be running (via `waitFor` or `runForever`).

---

## Table of Contents

1. [Core concept: async file I/O](#core-concept-async-file-io)
2. [Types](#types)
   - [AsyncFile](#asyncfile)
3. [Opening and creating files](#opening-and-creating-files)
   - [openAsync](#openasync)
   - [newAsyncFile](#newasyncfile)
4. [Reading](#reading)
   - [read](#read)
   - [readBuffer](#readbuffer)
   - [readLine](#readline)
   - [readAll](#readall)
   - [readToStream](#readtostream)
5. [Writing](#writing)
   - [write](#write)
   - [writeBuffer](#writebuffer)
   - [writeFromStream](#writefromstream)
6. [File pointer and metadata](#file-pointer-and-metadata)
   - [getFilePos](#getfilepos)
   - [setFilePos](#setfilepos)
   - [getFileSize](#getfilesize)
   - [setFileSize](#setfilesize)
7. [Closing](#closing)
   - [close](#close)
8. [FileMode reference](#filemode-reference)
9. [Complete example](#complete-example)
10. [Quick Cheat-Sheet](#quick-cheat-sheet)

---

## Core concept: async file I/O

In synchronous Nim code, calling `file.read(1024)` blocks the current thread until the OS delivers the data. In an async program this would stall the entire event loop — no other coroutines would make progress while waiting.

`std/asyncfile` solves this by returning a `Future[T]` from every I/O call. The `Future` is resolved later, by the event loop, when the OS signals that the operation is complete. Your coroutine suspends at each `await` and resumes automatically once the data is ready.

```
Your async proc             Event loop              OS
─────────────────           ──────────              ──
await f.read(4096)  ──────► registers callback
(suspends here)             runs other tasks ◄──── data ready signal
(resumes here)  ◄────────── calls your callback
```

This means that for truly high-throughput scenarios (log processors, HTTP servers serving large files), `asyncfile` lets a single thread handle many concurrent file operations without spawning threads.

---

## Types

### `AsyncFile`

```nim
type AsyncFile* = ref object
  fd: AsyncFD    # Platform file descriptor registered with the event loop
  offset: int64  # Current read/write position (internal)
```

**What it is:** A reference type that wraps an open file. Because it is a `ref`, it can be freely passed around and stored without worrying about copying. The `fd` and `offset` fields are internal — you never touch them directly; instead you use `getFilePos`, `setFilePos`, and the read/write procedures.

An `AsyncFile` must always be closed with `close` when you are done with it. Nim's garbage collector will not close it for you.

---

## Opening and creating files

### `openAsync`

```nim
proc openAsync*(filename: string, mode = fmRead): AsyncFile
```

**What it does:** Opens (or creates) the file at `filename` with the given `mode` and registers its file descriptor with the async event loop. This is the standard entry point — almost every async file workflow starts here.

On failure (file not found, permission denied, etc.) it raises an `OSError` immediately — `openAsync` itself is not async, only the subsequent read/write operations are.

The `mode` parameter is the standard Nim `FileMode` enum. See the [FileMode reference](#filemode-reference) table for all values and their effects.

**When to use it:** Whenever you need to open a file for async reading or writing. Prefer this over `newAsyncFile` unless you are integrating with an existing low-level file descriptor.

```nim
import std/[asyncfile, asyncdispatch, os]

proc example() {.async.} =
  # Open for reading (default)
  let r = openAsync("data.txt")
  defer: r.close()

  # Open for writing — creates or truncates
  let w = openAsync("output.txt", fmWrite)
  defer: w.close()

  # Open for appending — never truncates
  let a = openAsync("log.txt", fmAppend)
  defer: a.close()

  # Read-write on existing file only
  let rw = openAsync("existing.db", fmReadWriteExisting)
  defer: rw.close()

waitFor example()
```

---

### `newAsyncFile`

```nim
proc newAsyncFile*(fd: AsyncFD): AsyncFile
```

**What it does:** Wraps an **already-open** `AsyncFD` (an async-aware file descriptor) into an `AsyncFile` object and registers `fd` with the event loop via `register`. This is a low-level escape hatch for when you obtain a file descriptor from somewhere outside this module — for instance, from C FFI code, from `socketpair`, or from `os.open`.

**When to use it:** Rarely. Most code should use `openAsync`. Use `newAsyncFile` only when you already hold an `AsyncFD` from another source.

```nim
import std/[asyncfile, asyncdispatch]

# Hypothetical: a C library returns a raw fd
let rawFd: cint = someExternalLibraryOpen("file.bin")
let f = newAsyncFile(rawFd.AsyncFD)
defer: f.close()
```

> **Note:** You are responsible for ensuring that `fd` is opened with non-blocking I/O flags on POSIX, or with `FILE_FLAG_OVERLAPPED` on Windows, before passing it to `newAsyncFile`. `openAsync` does this automatically.

---

## Reading

### `read`

```nim
proc read*(f: AsyncFile, size: int): Future[string]
```

**What it does:** Asynchronously reads **up to** `size` bytes from the current file pointer position and returns them as a `string`. The file pointer advances by the number of bytes actually read.

Two important edge cases:
- If the file pointer is already at or past the end of file, the returned `Future` resolves to an **empty string** `""`.
- The returned string may be **shorter than** `size` bytes if the end of file is reached mid-read. Always check `result.len`, not just whether it is empty.

`size` must be greater than zero (enforced by an `assert`).

**When to use it:** When you want to read a known chunk of data as a Nim string — the most common high-level read operation.

```nim
import std/[asyncfile, asyncdispatch]

proc example() {.async.} =
  let f = openAsync("data.bin")
  defer: f.close()

  # Read first 16 bytes as a string (may be shorter at EOF)
  let header = await f.read(16)
  echo "Read ", header.len, " bytes: ", header

  # Read in 4 KB chunks until EOF
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

**What it does:** Asynchronously reads **up to** `size` bytes directly into a raw memory buffer pointed to by `buf`. Returns a `Future[int]` that resolves to the number of bytes actually read. Returns `0` if the file pointer is at or past the end of file.

This is the low-level counterpart to `read`. It is more efficient when you already manage your own memory buffer (avoiding an extra allocation and copy), or when working with C interop where you need a `pointer`-based interface.

**When to use it:** In performance-critical code where you pre-allocate your read buffer and want to avoid the string allocation that `read` performs.

```nim
import std/[asyncfile, asyncdispatch]

proc example() {.async.} =
  let f = openAsync("data.bin")
  defer: f.close()

  var buf = alloc(1024)
  defer: dealloc(buf)

  let bytesRead = await f.readBuffer(buf, 1024)
  echo "Read ", bytesRead, " bytes into buffer"

waitFor example()
```

---

### `readLine`

```nim
proc readLine*(f: AsyncFile): Future[string] {.async.}
```

**What it does:** Asynchronously reads one line of text from the file, stopping at `\n` (LF) or `\r\n` (CRLF). The returned string does **not** include the line terminator. Returns an empty string `""` when the end of file is reached.

Internally it calls `read(f, 1)` in a loop, reading one byte at a time. This makes it convenient but not particularly efficient for files with very long lines or for high-throughput reading — in those cases, reading larger chunks and splitting manually will be faster.

**When to use it:** For reading line-by-line from text files where simplicity matters more than maximum throughput.

```nim
import std/[asyncfile, asyncdispatch]

proc readAllLines() {.async.} =
  let f = openAsync("config.txt")
  defer: f.close()

  while true:
    let line = await f.readLine()
    if line.len == 0: break  # EOF
    echo "Line: ", line

waitFor readAllLines()
```

---

### `readAll`

```nim
proc readAll*(f: AsyncFile): Future[string] {.async.}
```

**What it does:** Reads the **entire remaining content** of the file from the current file pointer position to the end, accumulating everything into a single string. It does this by calling `read(f, 4000)` repeatedly until an empty result signals EOF.

Because it accumulates all data in memory, it is only suitable for files that comfortably fit in RAM. For large files, prefer streaming with `readToStream` or manual chunked reading.

**When to use it:** For small-to-medium configuration files, templates, or other resources that you need entirely in memory at once.

```nim
import std/[asyncfile, asyncdispatch, os]

proc loadConfig() {.async.} =
  let f = openAsync(getConfigDir() / "settings.json")
  defer: f.close()

  let json = await f.readAll()
  echo json.len, " bytes of config loaded"

waitFor loadConfig()
```

---

### `readToStream`

```nim
proc readToStream*(f: AsyncFile, fs: FutureStream[string]) {.async.}
```

**What it does:** Reads the file in 4000-byte chunks and writes each chunk to the `FutureStream[string]` `fs` as it arrives. When the end of file is reached, it calls `fs.complete()` to signal that no more data will arrive. The file is not closed automatically.

A `FutureStream` is Nim's async producer–consumer channel. By writing to it here, you allow a consumer on the other end to process data as it streams, rather than waiting for the whole file to load.

**When to use it:** When you want to pipe file content into another async consumer — for example, an HTTP response body, a compression pipeline, or a parser — without buffering the entire file.

```nim
import std/[asyncfile, asyncdispatch, asyncstreams]

proc streamFile(path: string): FutureStream[string] =
  result = newFutureStream[string]("streamFile")
  let fs = result  # capture for the async proc
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

## Writing

### `write`

```nim
proc write*(f: AsyncFile, data: string): Future[void]
```

**What it does:** Asynchronously writes the entire contents of the string `data` to the file at the current file pointer position. The `Future[void]` resolves when **all** bytes have been flushed to the OS (though not necessarily to physical disk). The file pointer advances by `data.len` bytes.

Writing an empty string (`""`) is a no-op on POSIX (calls `write` with size 0) and completes successfully.

**When to use it:** For the vast majority of async write operations — it is the idiomatic, highest-level write API.

```nim
import std/[asyncfile, asyncdispatch, os]

proc writeExample() {.async.} =
  let f = openAsync(getTempDir() / "out.txt", fmWrite)
  defer: f.close()

  await f.write("Hello, async world!\n")
  await f.write("Second line.\n")

waitFor writeExample()
```

---

### `writeBuffer`

```nim
proc writeBuffer*(f: AsyncFile, buf: pointer, size: int): Future[void]
```

**What it does:** Asynchronously writes exactly `size` bytes from the raw memory buffer at `buf` to the file. The `Future[void]` resolves when all bytes have been written. Like `write`, it advances the file pointer by `size` bytes.

This is the low-level counterpart to `write`. It avoids allocating a Nim string when the data already lives in a C-allocated or manually managed buffer.

**When to use it:** In performance-critical or C-interop code where you already have a `pointer` to the data you want to write.

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

**What it does:** The write-side mirror of `readToStream`. Reads string chunks from the `FutureStream[string]` `fs` one at a time and writes each chunk to the file immediately, freeing it from memory. Stops when `fs` signals completion (i.e., when `hasValue` is `false`). The file is not closed automatically.

Because each chunk is written and then discarded before the next is fetched, peak memory usage is limited to the size of a single chunk, regardless of total file size.

**When to use it:** For saving streamed data (network downloads, generated content, pipe output) to disk without buffering the whole payload in RAM.

```nim
import std/[asyncfile, asyncdispatch, asyncstreams]

proc saveStream(path: string, source: FutureStream[string]) {.async.} =
  let f = openAsync(path, fmWrite)
  defer: f.close()
  await f.writeFromStream(source)

# Example: pipe a network download directly to disk
# let stream = httpClient.getBodyStream("https://example.com/large.zip")
# waitFor saveStream("large.zip", stream)
```

---

## File pointer and metadata

### `getFilePos`

```nim
proc getFilePos*(f: AsyncFile): int64
```

**What it does:** Returns the current read/write position of the file pointer as a byte offset from the start of the file. The very first byte of the file is at offset `0`. This is a **synchronous**, instant call — it just reads the internally tracked `offset` field and returns it.

**When to use it:** To save the current position before a seek so you can return to it later, or to calculate how many bytes have been read so far.

```nim
import std/[asyncfile, asyncdispatch]

proc example() {.async.} =
  let f = openAsync("data.txt")
  defer: f.close()

  let startPos = f.getFilePos()   # 0
  discard await f.read(100)
  let afterRead = f.getFilePos()  # 100
  echo "Read ", afterRead - startPos, " bytes"

waitFor example()
```

---

### `setFilePos`

```nim
proc setFilePos*(f: AsyncFile, pos: int64)
```

**What it does:** Moves the file pointer to byte offset `pos` from the start of the file. The first byte is at position `0`. On POSIX systems this calls `lseek` immediately (and raises `OSError` on failure); on Windows the offset is simply stored and used in the next overlapped I/O call. After calling `setFilePos`, the next `read` or `write` will operate from position `pos`.

**When to use it:** To implement random-access reading (e.g., re-reading a header, seeking to a known data record), or to reset to the beginning after writing so you can read back what was written.

```nim
import std/[asyncfile, asyncdispatch, os]

proc roundTrip() {.async.} =
  let f = openAsync(getTempDir() / "tmp.txt", fmReadWrite)
  defer: f.close()

  await f.write("Hello, World!")

  # Seek back to start and read it back
  f.setFilePos(0)
  let content = await f.readAll()
  assert content == "Hello, World!"

waitFor roundTrip()
```

---

### `getFileSize`

```nim
proc getFileSize*(f: AsyncFile): int64
```

**What it does:** Returns the total size of the file in bytes as an `int64`. This is a **synchronous** call. On POSIX, it temporarily seeks to the end of the file to determine its size, then seeks back to the original position — so the file pointer is unchanged after the call. On Windows it uses the `GetFileSize` Win32 API directly without seeking.

**When to use it:** When you need to know the file size upfront — for example, to pre-allocate a buffer, to report progress, or to validate that a file has the expected size.

```nim
import std/[asyncfile, asyncdispatch]

proc example() {.async.} =
  let f = openAsync("archive.zip")
  defer: f.close()

  let size = f.getFileSize()
  echo "File is ", size, " bytes"

  # Pre-allocate a buffer of exactly the right size
  var buf = newString(size)
  discard await f.readBuffer(addr buf[0], size.int)

waitFor example()
```

---

### `setFileSize`

```nim
proc setFileSize*(f: AsyncFile, length: int64)
```

**What it does:** Truncates or extends the file to exactly `length` bytes. This is a **synchronous** call. If `length` is less than the current file size, the file is truncated and the excess data is lost. If `length` is greater, the file is extended (the content of the new bytes is platform-defined — typically zeros on most systems, but this is not guaranteed).

On POSIX this calls `ftruncate`. On Windows it moves the file pointer and calls `SetEndOfFile`. Raises `OSError` on failure.

**When to use it:** To pre-allocate space for a file before writing (which can improve write performance by avoiding repeated reallocation), or to truncate a log file.

```nim
import std/[asyncfile, asyncdispatch]

proc preallocate() {.async.} =
  let f = openAsync("preallocated.bin", fmReadWrite)
  defer: f.close()

  # Reserve 1 MB upfront
  f.setFileSize(1024 * 1024)

  # Now write data — no repeated OS reallocations
  f.setFilePos(0)
  await f.write("header data here")

waitFor preallocate()
```

---

## Closing

### `close`

```nim
proc close*(f: AsyncFile)
```

**What it does:** Unregisters the file descriptor from the async event loop and closes the underlying OS file handle. This is a **synchronous** call. Raises `OSError` if the OS close call fails (which can happen if buffered writes were not flushed, though async writes complete before the `Future` resolves, so this is uncommon).

**When to use it:** Always, when you are done with an `AsyncFile`. Failing to close a file leaks both the OS file descriptor and the event loop registration, which can eventually exhaust system resources. Use `defer: f.close()` immediately after `openAsync` to ensure the file is always closed even if an exception is raised.

```nim
import std/[asyncfile, asyncdispatch]

proc example() {.async.} =
  let f = openAsync("data.txt")
  defer: f.close()   # ← always closed, even on exception

  let data = await f.readAll()
  echo data

waitFor example()
```

---

## FileMode reference

The `FileMode` enum is defined in `std/io` but used throughout `asyncfile`. Here is what each value means when passed to `openAsync`:

| Value | File exists | File absent | Read | Write | Position |
|---|---|---|---|---|---|
| `fmRead` | Opens it | Raises error | ✓ | ✗ | Start |
| `fmWrite` | Truncates to 0 | Creates new | ✗ | ✓ | Start |
| `fmAppend` | Opens it | Creates new | ✗ | ✓ | End |
| `fmReadWrite` | Truncates to 0 | Creates new | ✓ | ✓ | Start |
| `fmReadWriteExisting` | Opens it | Raises error | ✓ | ✓ | Start |

**Typical choices:**
- Loading a config or data file → `fmRead`
- Writing a new output file → `fmWrite`
- Adding to a log file → `fmAppend`
- In-place editing of an existing file → `fmReadWriteExisting`
- Creating a new temp file for read-write → `fmReadWrite`

---

## Complete example

A self-contained program demonstrating the full async file I/O workflow:

```nim
import std/[asyncfile, asyncdispatch, os, strutils]

proc processLog(inputPath, outputPath: string) {.async.} =
  let src = openAsync(inputPath, fmRead)
  defer: src.close()

  let dst = openAsync(outputPath, fmWrite)
  defer: dst.close()

  echo "Input size: ", src.getFileSize(), " bytes"

  var lineNum = 0
  while true:
    let line = await src.readLine()
    if line.len == 0 and src.getFilePos() >= src.getFileSize():
      break   # true EOF (not just a blank line)
    inc lineNum
    # Keep only lines that contain "ERROR"
    if "ERROR" in line:
      await dst.write($lineNum & ": " & line & "\n")

  echo "Done. Output written to ", outputPath

proc main() {.async.} =
  let tmp = getTempDir()
  let input  = tmp / "app.log"
  let output = tmp / "errors.log"

  # Write some sample log data
  let w = openAsync(input, fmWrite)
  await w.write("INFO  service started\n")
  await w.write("ERROR disk full\n")
  await w.write("INFO  retrying\n")
  await w.write("ERROR connection timeout\n")
  w.close()

  await processLog(input, output)

  # Read and display the result
  let r = openAsync(output)
  defer: r.close()
  echo await r.readAll()

waitFor main()
```

---

## Quick Cheat-Sheet

| Task | API |
|---|---|
| Open a file for async I/O | `openAsync(path, mode)` |
| Wrap existing fd | `newAsyncFile(fd)` |
| Read N bytes as string | `await f.read(n)` |
| Read N bytes into buffer | `await f.readBuffer(buf, n)` |
| Read one line | `await f.readLine()` |
| Read entire file | `await f.readAll()` |
| Pipe file to stream | `await f.readToStream(fs)` |
| Write a string | `await f.write(data)` |
| Write raw buffer | `await f.writeBuffer(buf, n)` |
| Write from stream | `await f.writeFromStream(fs)` |
| Get current position | `f.getFilePos()` |
| Seek to position | `f.setFilePos(pos)` |
| Get file size | `f.getFileSize()` |
| Truncate / extend file | `f.setFileSize(length)` |
| Close file | `f.close()` |
| Always-close pattern | `defer: f.close()` |
