# std/socketstreams — Module Reference

> **Part of Nim's standard library** (`std/socketstreams`).

## Overview

`std/socketstreams` bridges two independently useful abstractions: **sockets** (from `std/net`) and **streams** (from `std/streams`). A stream in Nim is a uniform interface for sequential reading and writing — the same code that reads from a file stream can work with any other stream implementation. This module gives you stream objects backed by a network socket, so all the stream-level helpers (`readLine`, `readStr`, `peekStr`, `write`, `setPosition`, `getPosition`, `flush`, `close`, and so on) work transparently over a socket connection.

The module provides **two distinct stream types** with different contracts:

| Type | Writes? | Reads? | Where reads come from |
|---|---|---|---|
| `ReadSocketStream` | No | Yes | Live socket (`recv`) + internal buffer |
| `WriteSocketStream` | Yes | Yes | Internal buffer only (never from the socket) |

Understanding the **internal buffer** is key to using both types correctly:

- On a `ReadSocketStream`, incoming bytes are received from the socket *once* and stored in the buffer. After that, seeking backwards re-reads from the buffer — the socket is not contacted again. The buffer grows as new data arrives.
- On a `WriteSocketStream`, writes go into the buffer, reads come back out of it, and `flush` is the explicit action that sends buffered bytes over the socket. The socket itself is never read.

---

## Types

### `ReadSocketStreamObj` / `ReadSocketStream`

```nim
type
  ReadSocketStream* = ref ReadSocketStreamObj
  ReadSocketStreamObj* = object of StreamObj
    data: Socket     # the underlying socket
    pos:  int        # current read/write position in buf
    buf:  seq[byte]  # internal accumulation buffer
```

A reference-counted stream object that wraps a `Socket` for **reading**. All bytes received from the socket accumulate in `buf`, indexed by `pos`. Once data is in the buffer it stays there until `resetStream` is called — which means `setPosition(0)` followed by `readLine` will return data from the buffer without touching the socket at all.

`ReadSocketStream` does **not** support `write` or `flush` — attempting them will raise an error because `writeDataImpl` is not set.

---

### `WriteSocketStreamObj` / `WriteSocketStream`

```nim
type
  WriteSocketStream* = ref WriteSocketStreamObj
  WriteSocketStreamObj* = object of ReadSocketStreamObj
    lastFlush: int   # index of the first byte not yet sent
```

Inherits from `ReadSocketStreamObj` and adds a write capability. The extra field `lastFlush` is the high-water mark: bytes from `0` up to `lastFlush - 1` have already been sent via the socket and are immutable. Trying to overwrite them raises an `IOError`. Bytes from `lastFlush` onwards can still be written.

Unlike `ReadSocketStream`, reads on a `WriteSocketStream` are **entirely local** — they read back from `buf`, not from the socket.

`atEnd` on a `WriteSocketStream` returns `true` when `pos == buf.len`, meaning the position has reached the end of what has been written so far.

---

## Constructor Procedures

---

### `newReadSocketStream`

```nim
proc newReadSocketStream*(s: Socket): owned ReadSocketStream
```

**Creates a new `ReadSocketStream` wrapping the given socket.**

The returned stream owns its stream object (not the socket — you retain ownership of `s` and are responsible for not closing it independently while the stream is in use). The internal buffer starts empty and `pos` starts at `0`.

Wires up the following stream operations: `close`, `atEnd` (always `false`), `setPosition`, `getPosition`, `readData`, `peekData`, `readDataStr`. Write operations are **not** wired up.

**Example — reading a response over UDP:**

```nim
import std/[net, socketstreams]

var socket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
socket.sendTo("127.0.0.1", Port(12345), "PING")

var stream = newReadSocketStream(socket)
echo stream.readLine()     # receives from socket, stores in buffer
stream.setPosition(0)
echo stream.readLine()     # re-reads from buffer — no socket call
stream.close()
```

---

### `newWriteSocketStream`

```nim
proc newWriteSocketStream*(s: Socket): owned WriteSocketStream
```

**Creates a new `WriteSocketStream` wrapping the given socket.**

The buffer starts empty, `pos` is `0`, and `lastFlush` is `0` (nothing has been sent yet). Wires up: `close`, `atEnd`, `setPosition`, `getPosition`, `writeData`, `readData`, `peekData`, `flush`. The `readDataStr` operation is **not** wired up.

**Example — constructing and sending a packet:**

```nim
import std/[net, socketstreams]

var socket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
socket.connect("127.0.0.1", Port(12345))

var stream = newWriteSocketStream(socket)
stream.write("NOM")
stream.setPosition(1)
echo stream.peekStr(2)    # "OM"  — reads from buffer
stream.write("I")         # overwrites position 1 with 'I'
stream.setPosition(0)
echo stream.readStr(3)    # "NIM" — reads from buffer
stream.flush()            # sends "NIM" over the socket
```

---

## `resetStream`

```nim
proc resetStream*(s: ReadSocketStream)
proc resetStream*(s: WriteSocketStream)
```

**Clears the internal buffer and resets the stream position to `0`.**

Two overloaded versions — one for each stream type. On a `WriteSocketStream`, `resetStream` also resets `lastFlush` to `0`.

Calling `resetStream` does **not** affect the underlying socket in any way — no data is sent, no connection is closed, the socket remains open and usable. It is purely a buffer management operation.

After `resetStream`:
- The next read on a `ReadSocketStream` will receive fresh data from the socket (the buffer is empty, so there is nothing to serve from cache).
- The next write on a `WriteSocketStream` starts filling the buffer from the beginning, and the next `flush` will send from position `0`.

**Example — re-reading from socket after clearing buffer:**

```nim
import std/[net, socketstreams]

var socket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
var stream = newReadSocketStream(socket)

socket.sendTo("127.0.0.1", Port(12345), "REQUEST 1")
echo stream.readLine()    # reads from socket into buffer

stream.resetStream()      # buffer cleared; pos = 0

socket.sendTo("127.0.0.1", Port(12345), "REQUEST 2")
echo stream.readLine()    # reads fresh from socket again
```

---

## Behavioural Details

### ReadSocketStream — buffer semantics

The buffer grows lazily as data arrives. When you call a read or peek operation and the requested range extends beyond the current buffer, the stream calls `recv` on the underlying socket to fill in the missing bytes. If the socket returns fewer bytes than requested (e.g. a short UDP datagram), the returned count reflects what was actually available.

Seeking backwards (`setPosition` to a position already in the buffer) never touches the socket — the data is served entirely from the buffer. This makes it safe to implement parsers that need to backtrack: you pay the socket round-trip only once per byte.

`atEnd` on a `ReadSocketStream` always returns `false`. Since a TCP stream or UDP socket can always receive more data, there is no reliable way to know when "the end" has arrived without application-level framing.

### WriteSocketStream — immutability after flush

`flush` marks the current buffer length as the new `lastFlush`. Everything before that point is **immutable**: if you move the position back into the already-sent region and try to write, an `IOError` is raised immediately with the message `"Unable to write into buffer that has already been sent"`. You can still *read* (peek) from the sent region — it remains in the buffer — but you cannot change it.

This prevents subtle bugs where in-flight data is silently overwritten. If you need a completely fresh slate, call `resetStream`.

### WriteSocketStream — reading past the end

On a `WriteSocketStream`, attempting to read or peek past the current end of the buffer raises an `IOError`: `"Unable to read past end of write buffer"`. There is no socket fallback — reads only come from what has been explicitly written into the buffer.

---

## Inherited Stream Operations

`ReadSocketStream` and `WriteSocketStream` both inherit the full `std/streams` interface. The following operations (provided by `std/streams`) work transparently on both types, subject to the constraints described above:

| Operation | ReadSocketStream | WriteSocketStream |
|---|---|---|
| `readLine()` | ✔ reads from socket/buffer | ✘ (no socket reads) |
| `readStr(n)` | ✔ | ✔ reads from buffer |
| `peekStr(n)` | ✔ | ✔ peeks buffer |
| `write(x)` | ✘ not supported | ✔ writes to buffer |
| `flush()` | ✘ no-op / not wired | ✔ sends buffer over socket |
| `setPosition(n)` | ✔ | ✔ |
| `getPosition()` | ✔ | ✔ |
| `atEnd()` | always `false` | `true` when pos == buf.len |
| `close()` | ✔ closes the socket | ✔ closes the socket |
| `resetStream(s)` | ✔ clears buffer, pos=0 | ✔ clears buffer, pos=0, lastFlush=0 |

---

## Complete Usage Examples

### ReadSocketStream — seek-and-reread pattern

```nim
import std/[net, socketstreams]

var socket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
var stream = newReadSocketStream(socket)

socket.sendTo("127.0.0.1", Port(12345), "SOME REQUEST")

let line = stream.readLine()   # receives from socket
echo line

stream.setPosition(0)
echo stream.readLine()         # served from buffer; socket not called

stream.resetStream()           # buffer cleared
echo stream.readLine()         # receives from socket again

stream.close()                 # closes the underlying socket
```

### WriteSocketStream — build, inspect, send

```nim
import std/[net, socketstreams]

var socket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
socket.connect("127.0.0.1", Port(12345))

var sendStream = newWriteSocketStream(socket)

sendStream.write("NOM")         # buffer: [N, O, M], pos = 3
sendStream.setPosition(1)
echo sendStream.peekStr(2)      # "OM" — peek at positions 1-2

sendStream.write("I")           # buffer: [N, I, M], pos = 2
sendStream.setPosition(0)
echo sendStream.readStr(3)      # "NIM", pos = 3

echo sendStream.getPosition()   # 3

sendStream.flush()              # sends "NIM" over the socket; lastFlush = 3

sendStream.setPosition(1)
sendStream.write("I")           # ERROR: position 1 is before lastFlush (3)
```

---

## Summary Table

| Symbol | Kind | Purpose |
|--------|------|---------|
| `ReadSocketStream` | ref object | Read-only stream backed by a socket |
| `WriteSocketStream` | ref object | Read/write stream with buffered-then-flush semantics |
| `newReadSocketStream(s)` | proc | Wrap a socket in a ReadSocketStream |
| `newWriteSocketStream(s)` | proc | Wrap a socket in a WriteSocketStream |
| `resetStream(s: ReadSocketStream)` | proc | Clear buffer, reset pos to 0 |
| `resetStream(s: WriteSocketStream)` | proc | Clear buffer, reset pos and lastFlush to 0 |

---

> **See also:** [`std/streams`](https://nim-lang.org/docs/streams.html) for the full stream interface (all operations inherited by socket streams); [`std/net`](https://nim-lang.org/docs/net.html) for the `Socket` type and socket creation.
