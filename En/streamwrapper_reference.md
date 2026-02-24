# `streamwrapper` Module Reference

> **Module:** `std/streamwrapper`  
> **Purpose:** Wrap a forward-only (pipe-like) stream so that `peek*` operations work correctly on it, while making any illegal operation (writing, seeking) raise a runtime error.

---

## Overview

Nim's `std/streams` module defines the `Stream` abstraction with a rich set of operations: reading, writing, peeking, seeking, and flushing. However, not every concrete stream implementation supports all of these. A **pipe stream** — the output stream of a child process, for example — is fundamentally forward-only and write-protected: you can only read from it in the direction data arrives, and you cannot rewind.

The problem with using such a stream directly is that `peek*` operations (which "look ahead" without advancing the read position) are not supported by forward-only streams. Pipes have no built-in notion of a position or a look-ahead buffer — once a byte is read from a pipe, it is consumed and gone.

`streamwrapper` solves this with a thin, transparent wrapper that adds a software look-ahead buffer on top of any pipe-like stream. The wrapper intercepts `peek*` calls, reads ahead from the underlying stream, stores the extra bytes in an internal `Deque[char]`, and serves future `read*` calls from that deque before touching the underlying stream again.

At the same time, the wrapper explicitly disables all operations that are meaningless or dangerous on a pipe — seeking (`setPosition`, `getPosition`), writing (`writeData`), and flushing (`flush`) — by setting their implementation pointers to `nil`. Calling those on a wrapped stream produces a Nim runtime assertion failure, which is far more informative than silent data corruption or a mysterious OS error.

### The look-ahead buffer in detail

```
Underlying pipe stream: → [D] [E] [F] [G] [H] …
                                  ↑ next unconsumed byte

PipeOutStream internal Deque: [ ]  (empty until a peek is performed)

After peekChar():
  Deque: ['D']       ← stored here, NOT consumed from logical position
  peek returns: 'D'

After readChar():
  Deque: []          ← 'D' popped from front
  read returns: 'D'  ← same byte, position advanced by 1
```

The deque is a FIFO — `popFirst` serves bytes in the order they arrived. When a `read` requests more bytes than the deque holds, the deque is drained first and then the remainder is filled directly from the underlying stream.

---

## Exported Types

---

### `PipeOutStream[T]`

```nim
type PipeOutStream*[T] = ref object of T
  buffer: Deque[char]
  baseReadLineImpl: typeof(StreamObj.readLineImpl)
  baseReadDataImpl: typeof(StreamObj.readDataImpl)
```

#### What it is

A generic reference type that inherits from any stream type `T` (where `T` must itself be a subtype of `Stream`, since `PipeOutStream` is used exclusively to wrap streams). The `[T]` parameter specifies the concrete stream type being wrapped — typically `Stream` itself, but it could also be `FileStream`, `StringStream`, or any custom stream type that is pipe-like.

Because `PipeOutStream[T]` inherits from `T`, the wrapped object is a valid `T` and can be passed anywhere a `T` (or `Stream`) is expected. There is no type-erasure: all of the wrapped stream's vtable entries are copied and then selectively overridden.

#### Fields

All fields are private (not exported). They are described here for conceptual understanding:

| Field | Type | Purpose |
|---|---|---|
| `buffer` | `Deque[char]` | The look-ahead buffer. Bytes that have been physically read from the underlying stream for a peek operation but not yet logically consumed by a read operation live here. |
| `baseReadLineImpl` | proc reference | Saved pointer to the original `readLineImpl` of the wrapped stream, so the wrapper can delegate to it when the deque is exhausted. `nil` if the wrapped stream had no `readLineImpl`. |
| `baseReadDataImpl` | proc reference | Saved pointer to the original `readDataImpl` of the wrapped stream. Always non-nil (asserted in `newPipeOutStream`). |

#### Disabled operations

The following `Stream` operations are set to `nil` at construction time and will raise a runtime error if called:

| Operation | Why disabled |
|---|---|
| `setPosition` | Pipes have no seek capability. |
| `getPosition` | Pipe position is not tracked. |
| `writeData` | The output stream of a process is read-only from the caller's side. |
| `flush` | Nothing to flush on a read-only pipe. |

---

## Exported Procedures

---

### `newPipeOutStream`

```nim
proc newPipeOutStream*[T](s: sink (ref T)): owned PipeOutStream[T]
```

#### What it does

Wraps an existing stream `s` of type `ref T` into a `PipeOutStream[T]`, enabling `peek*` operations and disabling illegal operations. This is the only constructor for `PipeOutStream` and the only procedure you need to call directly — everything else is automatic.

The function performs these steps:

1. **Asserts** that `s.readDataImpl != nil` — the stream must be readable.
2. **Allocates** a new `PipeOutStream[T]` object on the heap.
3. **Copies all fields** from `s` into the new object (including all vtable function pointers). This is a shallow bitwise copy of the base `T` fields — the wrapper starts life as an identical twin of the original.
4. **Moves** `s` into an invalidated state (`wasMoved`) so the caller no longer owns it. The wrapper now exclusively owns the underlying stream state.
5. **Installs replacement implementations** for `readLineImpl`, `readDataImpl`, `readDataStrImpl`, and `peekDataImpl` — these are the wrapper's own procedures that mediate access through the deque buffer.
6. **Saves the originals** in `baseReadLineImpl` and `baseReadDataImpl` for delegation.
7. **Sets to `nil`**: `setPositionImpl`, `getPositionImpl`, `writeDataImpl`, `flushImpl`.

#### The `sink` parameter

The parameter `s: sink (ref T)` uses Nim's `sink` modifier. This means:
- The caller relinquishes ownership of `s` upon calling `newPipeOutStream`.
- After the call, `s` should not be used.
- The compiler may optimise away the copy of the `ref` smart pointer.

In practice, pass the stream directly: `p.outputStream().newPipeOutStream()`. Do not keep a reference to the original stream object and read from both — the wrapper owns the underlying I/O resource exclusively after this call.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `s` | `sink (ref T)` | The pipe-like stream to wrap. Must have a non-nil `readDataImpl`. Ownership is transferred to the wrapper. |

#### Return value

An `owned PipeOutStream[T]` — a heap-allocated wrapper object ready for reading and peeking. The caller owns the result and is responsible for closing it.

#### Examples

```nim
import std/[osproc, streamwrapper]

# Wrap the stdout pipe of a child process
var p = startProcess("echo", args = ["hello"], options = {poUsePath})
var outStream = p.outputStream().newPipeOutStream()

# Now we can peek without consuming
echo outStream.peekChar()  # → 'h' (first char of "hello\n")
echo outStream.peekChar()  # → 'h' again — position did not advance

# Now read — consumes from the buffer
echo outStream.readChar()  # → 'h' — now the position advances

p.close()
```

```nim
import std/[osproc, streamwrapper, streams]

# Read all output from a command, demonstrating peek + read interplay
var p = startProcess("ls", options = {poUsePath})
var s = p.outputStream().newPipeOutStream()

while not s.atEnd():
  # Peek at the next character without consuming it
  let c = s.peekChar()
  if c == '\n':
    discard s.readChar()  # consume the newline
    echo "-- end of line --"
  else:
    write(stdout, s.readChar())  # consume and print

p.close()
```

```nim
import std/[osproc, streamwrapper, streams]

# Reading lines — works because posReadLine drains the deque first
var p = startProcess("echo", args = ["-e", "line1\nline2"], options = {poUsePath})
var s = p.outputStream().newPipeOutStream()

var line: string
while s.readLine(line):
  echo "Got: ", line

p.close()
```

```nim
import std/[osproc, streamwrapper]

# Demonstrate that write operations raise a runtime error
var p = startProcess("cat", options = {poUsePath})
var s = p.outputStream().newPipeOutStream()

try:
  s.write("oops")  # this will raise — writeDataImpl is nil
except AssertionDefect:
  echo "Correctly prevented: writing to a read-only pipe stream"

p.close()
```

```nim
import std/[osproc, streamwrapper]

# Using peek to implement a simple look-ahead parser
# that decides which branch to take without consuming input
var p = startProcess("printf", args = ["%s", "42 hello"], options = {poUsePath})
var s = p.outputStream().newPipeOutStream()

let first = s.peekChar()
if first >= '0' and first <= '9':
  echo "Starts with a digit: will parse a number"
else:
  echo "Starts with a letter: will parse a word"
# The read position is still at the very first character
p.close()
```

---

## Internal Implementation Details

This section describes the four private procedures that back the wrapper's vtable. They are not exported, but understanding them clarifies how the deque interacts with reads and peeks.

### `posReadData` — buffered read

When a read of `bufLen` bytes is requested:
1. At most `min(buffer.len, bufLen)` bytes are popped from the front of the deque and placed into the destination.
2. If `bufLen` exceeds what the deque held, the remaining bytes are read directly from the underlying stream via `baseReadDataImpl`.

This ensures the deque drains in FIFO order and the underlying stream is only touched for bytes not already buffered.

### `posPeekData` — non-consuming read

When a peek of `bufLen` bytes is requested:
1. At most `min(buffer.len, bufLen)` bytes are copied from the deque (using indexed access, not `popFirst` — they stay in the deque).
2. If more bytes are needed, they are read from the underlying stream via `baseReadDataImpl` and appended to the end of the deque with `addLast`, so future reads consume them in order.

The critical difference from `posReadData`: peek never removes anything from the deque.

### `posReadLine` — line reading with deque awareness

Line reading is more complex because newline detection requires examining individual characters:
1. Characters are popped one-by-one from the deque and added to `line`.
2. A `'\c'` (CR) causes one more character to be consumed from the underlying stream (handling `\r\n` — the next `\n` is discarded).
3. A `'\L'` (LF) terminates the line.
4. A `'\0'` (null byte) terminates the line if any characters were accumulated; otherwise it signals EOF.
5. Once the deque is exhausted, the remainder of the line is read from the underlying stream using `baseReadLineImpl` and appended.

### `posReadDataStr` — string-slice read

A thin wrapper over `posReadData` that extracts the address of a string slice and delegates, providing the `readDataStrImpl` vtable entry needed by some `Stream` callers.

---

## Design Notes and Pitfalls

**1. Do not use the original stream after wrapping**  
`newPipeOutStream` uses `wasMoved` to invalidate the original object `s`. Reading from the original stream after passing it to `newPipeOutStream` is undefined behaviour — bytes may be silently lost from the logical stream because the wrapper's deque is unaware of them.

**2. The type parameter `T` must be a stream type**  
`PipeOutStream[T]` inherits from `T`, so `T` must be `Stream` or a concrete subtype of `Stream`. Passing an unrelated type will produce a compile-time error because `PipeOutStream` accesses `StreamObj` vtable fields on `T`.

**3. `nil` vtable entries raise a runtime error, not a compile-time error**  
Calling `setPosition`, `getPosition`, `write`, or `flush` on a `PipeOutStream` will not be caught by the compiler. The error is detected at runtime when the `nil` function pointer is dereferenced. Use `when` or wrapper guards in code that must work with both seekable and non-seekable streams.

**4. The buffer is unbounded**  
Every byte requested by a peek but not yet consumed by a read accumulates in the `Deque`. If your code repeatedly peeks large amounts without reading them, memory usage grows without bound. In practice this is rarely a problem because peek operations on streams are typically small (one character, one line), but be aware of it in unusual patterns.

**5. `peekChar` / `readChar` are inherited from `Stream`**  
`PipeOutStream` does not define `peekChar` or `readChar` directly. These procedures are defined on `Stream` in `std/streams` and call the underlying `peekDataImpl` / `readDataImpl` vtable entries. Because `newPipeOutStream` installs `posPeekData` and `posReadData` into those slots, `peekChar` and `readChar` on a `PipeOutStream` automatically go through the deque.

**6. Closing the stream**  
Call `close(s)` on the `PipeOutStream` when done. This delegates to the base stream's `closeImpl`, which closes the underlying pipe and releases OS resources. The deque is garbage-collected automatically.

**7. Thread safety**  
`PipeOutStream` provides no internal locking. Do not read from or peek at the same `PipeOutStream` from multiple threads concurrently.
