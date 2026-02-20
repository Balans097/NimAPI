# Nim `asyncnet` Module — Complete Reference

> **Module:** `std/asyncnet`  
> **Purpose:** High-level asynchronous socket API built on top of `asyncdispatch`. Provides buffered, non-blocking TCP/UDP/Unix socket operations with optional SSL support.

---

## Asynchronous IO Architecture

Nim's async IO is built in layers (top to bottom):

```
asyncnet        ← this module (high-level, buffered sockets)
async/await     ← coroutine transformation
asyncdispatch   ← event loop, Futures, IOCP/epoll/kqueue
selectors       ← OS-level select/epoll/kqueue abstraction
```

All procedures are **single-threaded** and **fully non-blocking**. `async` procs return a `Future[T]` and can be `await`-ed.

### SSL

Enable SSL by compiling with `-d:ssl`. Create an `SslContext` via `net.newContext`, then call `wrapSocket` on your socket.

---

## Table of Contents

1. [Types](#types)
2. [Socket Creation](#socket-creation)
3. [Connection](#connection)
4. [Sending Data](#sending-data)
5. [Receiving Data](#receiving-data)
6. [Server Operations](#server-operations)
7. [Socket Options & Info](#socket-options--info)
8. [SSL Operations](#ssl-operations)
9. [Unix Domain Sockets](#unix-domain-sockets)
10. [Closing Sockets](#closing-sockets)
11. [Complete Examples](#complete-examples)

---

## Types

### `AsyncSocket`

```nim
type AsyncSocket* = ref AsyncSocketDesc
```

The main type representing an asynchronous socket. It is a reference type. Internally holds:

| Field | Description |
|-------|-------------|
| `fd` | Underlying OS socket file descriptor (`SocketHandle`) |
| `closed` | Whether the socket has been closed |
| `isBuffered` | Whether read buffering is enabled |
| `buffer` | Internal read buffer (size `BufferSize`) |
| `currPos` | Current read position in buffer |
| `bufLen` | Amount of valid data in buffer |
| `isSsl` | Whether SSL is active |
| `domain` | Address family (`AF_INET`, `AF_INET6`) |
| `sockType` | Socket type (`SOCK_STREAM`, `SOCK_DGRAM`) |
| `protocol` | Protocol (`IPPROTO_TCP`, `IPPROTO_UDP`) |

> **SSL-only fields** (available when compiled with `-d:ssl`):  
> `sslHandle`, `sslContext`, `bioIn`, `bioOut`, `sslNoShutdown`

---

## Socket Creation

### `newAsyncSocket` (from AsyncFD)

```nim
proc newAsyncSocket*(fd: AsyncFD,
                     domain: Domain = AF_INET,
                     sockType: SockType = SOCK_STREAM,
                     protocol: Protocol = IPPROTO_TCP,
                     buffered = true,
                     inheritable = defined(nimInheritHandles)): owned(AsyncSocket)
```

Creates an `AsyncSocket` wrapping an **existing** file descriptor. Non-blocking mode is enabled implicitly.

> **Important:** This does **not** register `fd` with the async dispatcher. If you created `fd` via `newAsyncNativeSocket`, it is already registered.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `fd` | An existing `AsyncFD` (must not be `osInvalidSocket`) |
| `domain` | Address family. Default: `AF_INET` |
| `sockType` | Socket type. Default: `SOCK_STREAM` |
| `protocol` | Protocol. Default: `IPPROTO_TCP` |
| `buffered` | Enable read buffering. Default: `true` |
| `inheritable` | Allow child processes to inherit this fd. Default: platform-defined |

**Example:**
```nim
import std/[asyncnet, asyncdispatch, nativesockets]

let rawFd = createAsyncNativeSocket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
let sock = newAsyncSocket(rawFd, AF_INET, SOCK_STREAM, IPPROTO_TCP)
```

---

### `newAsyncSocket` (create new fd)

```nim
proc newAsyncSocket*(domain: Domain = AF_INET,
                     sockType: SockType = SOCK_STREAM,
                     protocol: Protocol = IPPROTO_TCP,
                     buffered = true,
                     inheritable = defined(nimInheritHandles)): owned(AsyncSocket)
```

Creates a new asynchronous socket **with a fresh file descriptor**. This is the most common way to create a socket.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `domain` | Address family. Default: `AF_INET` (IPv4) |
| `sockType` | Socket type. Default: `SOCK_STREAM` (TCP) |
| `protocol` | Protocol. Default: `IPPROTO_TCP` |
| `buffered` | Enable read buffering. Default: `true` |
| `inheritable` | Allow child processes to inherit fd. Default: platform-defined |

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

# Default TCP/IPv4 socket:
let sock = newAsyncSocket()

# UDP/IPv6 socket:
let udpSock = newAsyncSocket(AF_INET6, SOCK_DGRAM, IPPROTO_UDP)

# Non-buffered socket:
let rawSock = newAsyncSocket(buffered = false)
```

---

### `newAsyncSocket` (raw cint overload)

```nim
proc newAsyncSocket*(domain, sockType, protocol: cint,
                     buffered = true,
                     inheritable = defined(nimInheritHandles)): owned(AsyncSocket)
```

Same as above but accepts raw `cint` values for maximum interoperability with C socket constants. Useful when you have integer constants from external C libraries.

**Example:**
```nim
import std/asyncnet

# Using raw C integer constants:
let sock = newAsyncSocket(2.cint, 1.cint, 6.cint)
# 2 = AF_INET, 1 = SOCK_STREAM, 6 = IPPROTO_TCP
```

---

## Connection

### `dial`

```nim
proc dial*(address: string, port: Port,
           protocol = IPPROTO_TCP,
           buffered = true): owned(Future[AsyncSocket]) {.async.}
```

Establishes a connection to `address:port` and returns a ready-to-use `AsyncSocket`. Automatically tries all IP addresses returned by DNS resolution, making it transparent for both IPv4 and IPv6.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `address` | Hostname or IP address string |
| `port` | Destination port |
| `protocol` | Transport protocol. Default: `IPPROTO_TCP` |
| `buffered` | Enable buffering. Default: `true` |

**Returns:** `Future[AsyncSocket]` — resolves when connected.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  # Connects to example.com, works with IPv4 and IPv6 automatically:
  let sock = await dial("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  let response = await sock.recv(4096)
  echo response
  sock.close()

waitFor main()
```

---

### `connect`

```nim
proc connect*(socket: AsyncSocket, address: string, port: Port) {.async.}
```

Connects an **already created** socket to a server at `address:port`. If SSL is active, performs the SSL handshake immediately after the TCP connection is established.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | An unconnected `AsyncSocket` |
| `address` | Server hostname or IP address |
| `port` | Server port |

**Returns:** `Future[void]` — resolves on successful connection.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("httpbin.org", Port(80))
  echo "Connected!"
  
  await sock.send("GET /get HTTP/1.0\r\nHost: httpbin.org\r\n\r\n")
  let data = await sock.recv(2048)
  echo data
  sock.close()

waitFor main()
```

---

### `connectUnix`

```nim
proc connectUnix*(socket: AsyncSocket, path: string): owned(Future[void])
```

Connects to a Unix domain socket at `path`. **POSIX only** (Linux, macOS, BSD).

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | An unconnected `AsyncSocket` |
| `path` | Filesystem path to the Unix socket |

**Returns:** `Future[void]` — resolves when connected.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket(AF_UNIX, SOCK_STREAM, 0)
  await sock.connectUnix("/var/run/myapp.sock")
  await sock.send("hello\n")
  let reply = await sock.recvLine()
  echo "Got: ", reply
  sock.close()

waitFor main()
```

---

## Sending Data

### `send` (string)

```nim
proc send*(socket: AsyncSocket, data: string,
           flags = {SocketFlag.SafeDisconn}) {.async.}
```

Sends a string to the socket. The future completes only after **all** data has been sent. Handles SSL transparently.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A connected `AsyncSocket` |
| `data` | String data to send |
| `flags` | Socket flags. Default: `{SafeDisconn}` |

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  
  # Send HTTP request:
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  echo "Request sent"
  sock.close()

waitFor main()
```

---

### `send` (raw pointer)

```nim
proc send*(socket: AsyncSocket, buf: pointer, size: int,
           flags = {SocketFlag.SafeDisconn}) {.async.}
```

Sends `size` bytes from a raw memory buffer. Useful for zero-copy scenarios or interop with C structures.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A connected `AsyncSocket` |
| `buf` | Pointer to data buffer |
| `size` | Number of bytes to send |
| `flags` | Socket flags. Default: `{SafeDisconn}` |

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("127.0.0.1", Port(9000))
  
  var message = "binary data"
  await sock.send(addr message[0], message.len)
  sock.close()

waitFor main()
```

---

### `sendTo`

```nim
proc sendTo*(socket: AsyncSocket, address: string, port: Port, data: string,
             flags = {SocketFlag.SafeDisconn}): owned(Future[void]) {.async.}
```

Sends a datagram to a specific `address:port`. Designed for **UDP sockets** — cannot be used on TCP sockets. Tries all DNS-resolved addresses until one succeeds.

> Available since Nim 1.3.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A UDP `AsyncSocket` (not TCP) |
| `address` | Destination hostname or IP |
| `port` | Destination port |
| `data` | Data to send |
| `flags` | Socket flags. Default: `{SafeDisconn}` |

**Raises:** `IOError` if address cannot be resolved; `OSError` on system error.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var udpSock = newAsyncSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
  
  await udpSock.sendTo("127.0.0.1", Port(9999), "ping")
  echo "Datagram sent"
  udpSock.close()

waitFor main()
```

---

## Receiving Data

### `recv`

```nim
proc recv*(socket: AsyncSocket, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[string]) {.async.}
```

Reads **up to** `size` bytes from the socket and returns them as a string.

- **Buffered mode:** attempts to fill the entire `size` in `BufferSize` chunks.
- **Unbuffered mode:** returns whatever the OS provides in one system call.
- Returns `""` if the socket is disconnected and no data is available.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A connected `AsyncSocket` |
| `size` | Maximum number of bytes to read |
| `flags` | Socket flags. Default: `{SafeDisconn}` |

**Returns:** `Future[string]` — the received data, possibly shorter than `size`.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  
  # Read up to 4096 bytes:
  let data = await sock.recv(4096)
  if data.len == 0:
    echo "Disconnected"
  else:
    echo "Received ", data.len, " bytes"
    echo data[0..min(200, data.len-1)]
  
  sock.close()

waitFor main()
```

---

### `recvInto`

```nim
proc recvInto*(socket: AsyncSocket, buf: pointer, size: int,
               flags = {SocketFlag.SafeDisconn}): owned(Future[int]) {.async.}
```

Reads **up to** `size` bytes from the socket directly into a pre-allocated buffer `buf`. Returns the number of bytes actually read. Useful for zero-copy or fixed-buffer protocols.

- **Buffered mode:** reads `BufferSize` chunks until `size` bytes are accumulated.
- **Unbuffered mode:** single OS read, returns immediately.
- Returns `0` if disconnected and no data is available.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A connected `AsyncSocket` |
| `buf` | Pointer to pre-allocated receive buffer |
| `size` | Size of buffer / maximum bytes to read |
| `flags` | Socket flags. Default: `{SafeDisconn}` |

**Returns:** `Future[int]` — number of bytes read.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  
  var buf: array[1024, char]
  let bytesRead = await sock.recvInto(addr buf[0], buf.len)
  echo "Read ", bytesRead, " bytes"
  echo cast[string](buf[0..<bytesRead])
  
  sock.close()

waitFor main()
```

---

### `recvLine`

```nim
proc recvLine*(socket: AsyncSocket,
               flags = {SocketFlag.SafeDisconn},
               maxLength = MaxLineLength): owned(Future[string]) {.async.}
```

Reads a complete line (terminated by `\r\L` or `\L`) from the socket.

- The `\r\L` terminator is **stripped** from the result.
- If only `\r\L` is received (empty line), the result **is** `"\r\L"`.
- Returns `""` if disconnected (mid-line data is discarded).
- `maxLength` protects against DoS attacks with very long lines.

> **Warning:** The `Peek` flag is not yet implemented.  
> **Warning:** On unbuffered sockets, assumes `\r\L` line endings.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A connected buffered `AsyncSocket` |
| `flags` | Socket flags. Default: `{SafeDisconn}` |
| `maxLength` | Maximum characters per line (`MaxLineLength` by default) |

**Returns:** `Future[string]` — one line without the terminator.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("towel.blinkenlights.nl", Port(23))
  
  # Read lines from a telnet connection:
  while true:
    let line = await sock.recvLine()
    if line == "":
      echo "Connection closed"
      break
    echo "> ", line
  
  sock.close()

waitFor main()
```

---

### `recvLineInto`

```nim
proc recvLineInto*(socket: AsyncSocket, resString: FutureVar[string],
                   flags = {SocketFlag.SafeDisconn},
                   maxLength = MaxLineLength) {.async.}
```

Low-allocation version of `recvLine`. Reads a line into a pre-existing `FutureVar[string]`, avoiding string allocation on each call. Ideal for high-throughput servers.

> **Warning:** The `Peek` flag is not yet implemented.  
> **Warning:** On unbuffered sockets, assumes `\r\L` line endings.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A connected `AsyncSocket` |
| `resString` | Pre-allocated `FutureVar[string]` to write into |
| `flags` | Socket flags. Default: `{SafeDisconn}` |
| `maxLength` | Maximum characters per line |

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc processClient(client: AsyncSocket) {.async.} =
  # Reuse a single FutureVar across many reads:
  var lineVar = newFutureVar[string]("processClient")
  while true:
    lineVar.mget() = ""
    await client.recvLineInto(lineVar)
    let line = lineVar.mget()
    if line == "": break
    echo "Got: ", line

waitFor processClient(newAsyncSocket())
```

---

### `recvFrom` (with FutureVars)

```nim
proc recvFrom*(socket: AsyncSocket, data: FutureVar[string], size: int,
               address: FutureVar[string], port: FutureVar[Port],
               flags = {SocketFlag.SafeDisconn}): owned(Future[int]) {.async.}
```

Receives a single UDP datagram into pre-allocated `FutureVar` buffers. Returns the number of bytes received. The sender's address and port are written into `address` and `port`.

> Available since Nim 1.3.

**Requirements:**
- `data` must be pre-initialized to `size` length.
- `address` must be pre-initialized to length 46 (max IPv6 string length).

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A UDP `AsyncSocket` |
| `data` | Pre-initialized `FutureVar[string]` for payload |
| `size` | Expected datagram size |
| `address` | Pre-initialized `FutureVar[string]` (len=46) for sender IP |
| `port` | `FutureVar[Port]` for sender port |
| `flags` | Socket flags |

**Returns:** `Future[int]` — actual bytes received.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var udpSock = newAsyncSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
  udpSock.bindAddr(Port(9999))
  
  var data = newFutureVar[string]()
  var address = newFutureVar[string]()
  var port = newFutureVar[Port]()
  
  data.mget().setLen(1024)
  address.mget().setLen(46)
  
  let bytesRead = await udpSock.recvFrom(data, 1024, address, port)
  echo "Got ", bytesRead, " bytes from ", address.mget(), ":", port.mget()
  echo "Data: ", data.mget()
  
  udpSock.close()

waitFor main()
```

---

### `recvFrom` (simple tuple overload)

```nim
proc recvFrom*(socket: AsyncSocket, size: int,
               flags = {SocketFlag.SafeDisconn}):
              owned(Future[tuple[data: string, address: string, port: Port]]) {.async.}
```

Simpler overload of `recvFrom` that allocates internally and returns a named tuple. More convenient but slightly less efficient than the `FutureVar` version.

> Available since Nim 1.3.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A UDP `AsyncSocket` |
| `size` | Maximum datagram size to receive |
| `flags` | Socket flags |

**Returns:** `Future[tuple[data, address, port]]`

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var udpSock = newAsyncSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
  udpSock.bindAddr(Port(9999))
  
  # Simple receive loop:
  while true:
    let (data, address, port) = await udpSock.recvFrom(1024)
    echo "From ", address, ":", port, " -> ", data
  
  udpSock.close()

waitFor main()
```

---

## Server Operations

### `bindAddr`

```nim
proc bindAddr*(socket: AsyncSocket, port = Port(0), address = "") {.tags: [ReadIOEffect].}
```

Binds the socket to a local `address:port`. If `address` is `""`, binds to all interfaces (`INADDR_ANY` / `::` for IPv6).

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | An unbound `AsyncSocket` |
| `port` | Local port to bind. `Port(0)` = OS chooses a free port |
| `address` | Local address string. `""` = all interfaces |

**Raises:** `OSError` on failure; `ValueError` for unknown domain with no address.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

var server = newAsyncSocket()
server.setSockOpt(OptReuseAddr, true)

# Bind to all interfaces on port 8080:
server.bindAddr(Port(8080))

# Bind to a specific interface:
server.bindAddr(Port(8080), "127.0.0.1")

# Let the OS assign a port:
server.bindAddr(Port(0))
let (_, assignedPort) = server.getLocalAddr()
echo "Listening on port: ", assignedPort
```

---

### `listen`

```nim
proc listen*(socket: AsyncSocket, backlog = SOMAXCONN) {.tags: [ReadIOEffect].}
```

Marks the socket as a passive server socket that accepts incoming connections. Must be called after `bindAddr` and before `accept`.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A bound `AsyncSocket` |
| `backlog` | Maximum number of pending connections in the queue. Default: `SOMAXCONN` |

**Raises:** `OSError` on failure.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

var server = newAsyncSocket()
server.setSockOpt(OptReuseAddr, true)
server.bindAddr(Port(8080))
server.listen()
echo "Server is listening..."
```

---

### `accept`

```nim
proc accept*(socket: AsyncSocket,
             flags = {SocketFlag.SafeDisconn}): owned(Future[AsyncSocket])
```

Waits for and accepts a single incoming connection. Returns the client socket only (without the address). If you also need the client's IP address, use `acceptAddr` instead.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A listening `AsyncSocket` |
| `flags` | Socket flags. Default: `{SafeDisconn}` |

**Returns:** `Future[AsyncSocket]` — the connected client socket.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc serve() {.async.} =
  var server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(8080))
  server.listen()
  
  echo "Waiting for connections..."
  while true:
    let client = await server.accept()
    echo "Client connected"
    asyncCheck (proc() {.async.} =
      let line = await client.recvLine()
      await client.send("Echo: " & line & "\r\n")
      client.close()
    )()

waitFor serve()
```

---

### `acceptAddr`

```nim
proc acceptAddr*(socket: AsyncSocket,
                 flags = {SocketFlag.SafeDisconn},
                 inheritable = defined(nimInheritHandles)):
      owned(Future[tuple[address: string, client: AsyncSocket]])
```

Waits for and accepts a single incoming connection. Returns both the client socket **and** the remote address string as a tuple.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | A listening `AsyncSocket` |
| `flags` | Socket flags. Default: `{SafeDisconn}` |
| `inheritable` | Whether the client socket fd is inheritable by child processes |

**Returns:** `Future[tuple[address: string, client: AsyncSocket]]`

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc serve() {.async.} =
  var server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(8080))
  server.listen()
  
  while true:
    let (address, client) = await server.acceptAddr()
    echo "Connection from: ", address
    asyncCheck (proc() {.async.} =
      await client.send("Hello from server\r\n")
      client.close()
    )()

waitFor serve()
```

---

### `bindUnix`

```nim
proc bindUnix*(socket: AsyncSocket, path: string) {.tags: [ReadIOEffect].}
```

Binds a Unix domain socket to a filesystem `path`. **POSIX only** (Linux, macOS, BSD). Call `listen` afterwards to start accepting connections.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | An unbound `AsyncSocket` (must use `AF_UNIX` domain) |
| `path` | Filesystem path for the socket file |

**Raises:** `OSError` on failure.

**Example:**
```nim
import std/[asyncnet, asyncdispatch, os]

proc serve() {.async.} =
  let sockPath = "/tmp/myapp.sock"
  if fileExists(sockPath): removeFile(sockPath)
  
  var server = newAsyncSocket(AF_UNIX, SOCK_STREAM, 0)
  server.bindUnix(sockPath)
  server.listen()
  echo "Unix socket server listening at ", sockPath
  
  let client = await server.accept()
  let msg = await client.recvLine()
  echo "Got: ", msg
  client.close()
  server.close()

waitFor serve()
```

---

## Socket Options & Info

### `getLocalAddr`

```nim
proc getLocalAddr*(socket: AsyncSocket): (string, Port)
```

Returns the local address and port the socket is bound to. Wraps `getsockname`.

**Returns:** Tuple `(address: string, port: Port)`.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

var sock = newAsyncSocket()
sock.bindAddr(Port(0))  # OS picks a port
let (addr, port) = sock.getLocalAddr()
echo "Bound to: ", addr, ":", port
```

---

### `getPeerAddr`

```nim
proc getPeerAddr*(socket: AsyncSocket): (string, Port)
```

Returns the remote address and port of the connected peer. Wraps `getpeername`. **Not available** when compiled with `nimNetLite`.

**Returns:** Tuple `(address: string, port: Port)`.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  let (peerAddr, peerPort) = sock.getPeerAddr()
  echo "Connected to: ", peerAddr, ":", peerPort
  sock.close()

waitFor main()
```

---

### `getSockOpt`

```nim
proc getSockOpt*(socket: AsyncSocket, opt: SOBool, level = SOL_SOCKET): bool {.tags: [ReadIOEffect].}
```

Retrieves a boolean socket option. Commonly used options include `OptReuseAddr`, `OptReusePort`, `OptNoDelay`, `OptKeepAlive`.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | The socket to query |
| `opt` | The option to read (a `SOBool` enum value) |
| `level` | Option level. Default: `SOL_SOCKET` |

**Returns:** `true` if the option is set.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

var sock = newAsyncSocket()
echo "ReuseAddr: ", sock.getSockOpt(OptReuseAddr)
echo "KeepAlive: ", sock.getSockOpt(OptKeepAlive)
sock.close()
```

---

### `setSockOpt`

```nim
proc setSockOpt*(socket: AsyncSocket, opt: SOBool, value: bool,
                 level = SOL_SOCKET) {.tags: [WriteIOEffect].}
```

Sets a boolean socket option.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `socket` | The socket to configure |
| `opt` | The option to set (a `SOBool` enum value) |
| `value` | `true` to enable, `false` to disable |
| `level` | Option level. Default: `SOL_SOCKET` |

**Common options:**

| Option | Description |
|--------|-------------|
| `OptReuseAddr` | Allow reuse of local address (avoids "address already in use") |
| `OptReusePort` | Allow multiple sockets to bind the same port |
| `OptNoDelay` | Disable Nagle's algorithm (TCP only) |
| `OptKeepAlive` | Enable TCP keepalive |
| `OptBroadcast` | Allow sending broadcast datagrams (UDP) |

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

var server = newAsyncSocket()
server.setSockOpt(OptReuseAddr, true)   # avoid "already in use" errors
server.setSockOpt(OptKeepAlive, true)   # detect dead connections
server.bindAddr(Port(8080))
server.listen()
```

---

### `getFd`

```nim
proc getFd*(socket: AsyncSocket): SocketHandle
```

Returns the raw OS file descriptor (`SocketHandle`) of the socket. Useful for low-level interop with OS APIs or native socket libraries.

**Returns:** `SocketHandle`

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

var sock = newAsyncSocket()
let fd = sock.getFd()
echo "Socket file descriptor: ", fd.int
```

---

### `isClosed`

```nim
proc isClosed*(socket: AsyncSocket): bool
```

Returns `true` if the socket has been closed via `close()`.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

var sock = newAsyncSocket()
echo "Before close: ", sock.isClosed()  # false
sock.close()
echo "After close: ", sock.isClosed()   # true
```

---

### `isSsl`

```nim
proc isSsl*(socket: AsyncSocket): bool
```

Returns `true` if the socket has been wrapped with an SSL context via `wrapSocket` or `wrapConnectedSocket`.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

var sock = newAsyncSocket()
echo "Is SSL: ", sock.isSsl()  # false

# After wrapSocket(ctx, sock):
# echo "Is SSL: ", sock.isSsl()  # true
sock.close()
```

---

### `hasDataBuffered`

```nim
proc hasDataBuffered*(s: AsyncSocket): bool
```

Returns `true` if the socket has unread data in its internal read buffer. Useful to check whether you can read without blocking (i.e., buffered data is immediately available).

> Available since Nim 1.5.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  
  # Read some data to populate the buffer:
  discard await sock.recv(100)
  
  if sock.hasDataBuffered():
    echo "More data available in buffer"
  sock.close()

waitFor main()
```

---

## SSL Operations

> All SSL procs require compiling with `-d:ssl`.

### `wrapSocket`

```nim
proc wrapSocket*(ctx: SslContext, socket: AsyncSocket)
```

Converts an `AsyncSocket` into an SSL socket using the provided `SslContext`. Must be called **before** connecting or accepting. Sets up internal BIO memory buffers for OpenSSL.

> **Disclaimer:** SSL support may have security vulnerabilities. Use carefully.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `ctx` | An `SslContext` created via `net.newContext()` |
| `socket` | The socket to wrap |

**Example:**
```nim
import std/[asyncnet, asyncdispatch, net]

proc main() {.async.} =
  let ctx = newContext()  # from std/net
  var sock = newAsyncSocket()
  wrapSocket(ctx, sock)
  await sock.connect("example.com", Port(443))
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  let data = await sock.recv(4096)
  echo data
  sock.close()

waitFor main()
```

---

### `wrapConnectedSocket`

```nim
proc wrapConnectedSocket*(ctx: SslContext, socket: AsyncSocket,
                          handshake: SslHandshakeType,
                          hostname: string = "")
```

Wraps an **already connected** socket in SSL and performs the SSL handshake immediately. Use `handshakeAsClient` for outgoing connections, `handshakeAsServer` for accepted connections.

If `hostname` is provided and is not an IP address, sets the SNI (Server Name Indication) extension for TLS 1.0+.

> **Disclaimer:** SSL support may have security vulnerabilities. Use carefully.

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `ctx` | An `SslContext` |
| `socket` | An already TCP-connected socket |
| `handshake` | `handshakeAsClient` or `handshakeAsServer` |
| `hostname` | SNI hostname (client only, optional) |

**Example:**
```nim
import std/[asyncnet, asyncdispatch, net]

proc main() {.async.} =
  let ctx = newContext()
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(443))
  
  # Upgrade to SSL on already-connected socket:
  wrapConnectedSocket(ctx, sock, handshakeAsClient, "example.com")
  
  await sock.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  let resp = await sock.recv(4096)
  echo resp[0..200]
  sock.close()

waitFor main()
```

---

### `getPeerCertificates`

```nim
proc getPeerCertificates*(socket: AsyncSocket): seq[Certificate]
```

Returns the certificate chain presented by the peer after a completed SSL handshake. The chain is ordered from leaf to root. Returns an empty sequence if the socket is not SSL or the handshake has not been completed/verified.

> Available since Nim 1.1.

**Returns:** `seq[Certificate]` — peer certificate chain, leaf first.

**Example:**
```nim
import std/[asyncnet, asyncdispatch, net]

proc main() {.async.} =
  let ctx = newContext()
  var sock = newAsyncSocket()
  wrapSocket(ctx, sock)
  await sock.connect("example.com", Port(443))
  
  let certs = sock.getPeerCertificates()
  echo "Certificate chain length: ", certs.len
  for cert in certs:
    echo "  Subject: ", cert.subject
  
  sock.close()

waitFor main()
```

---

### `sslHandle`

```nim
proc sslHandle*(self: AsyncSocket): SslPtr
```

Returns the raw OpenSSL `SSL*` handle. Useful for advanced OpenSSL interop (e.g., querying cipher, session info, or calling OpenSSL functions directly).

**Returns:** `SslPtr` (an `SSL*` pointer from OpenSSL).

**Example:**
```nim
import std/[asyncnet, asyncdispatch, net, openssl]

proc main() {.async.} =
  let ctx = newContext()
  var sock = newAsyncSocket()
  wrapSocket(ctx, sock)
  await sock.connect("example.com", Port(443))
  
  let ssl = sock.sslHandle()
  # Can now call OpenSSL functions directly on `ssl`
  sock.close()

waitFor main()
```

---

## Unix Domain Sockets

### Summary

| Proc | Direction | Platform |
|------|-----------|----------|
| `bindUnix` | Server | POSIX only |
| `connectUnix` | Client | POSIX only |

See their individual sections above for full documentation and examples.

---

## Closing Sockets

### `close`

```nim
proc close*(socket: AsyncSocket)
```

Closes the socket and releases all associated resources. If the socket is already closed, this is a no-op (safe to call multiple times).

For SSL sockets, performs a proper `SSL_shutdown` before closing unless the SSL handshake was never completed.

**Example:**
```nim
import std/[asyncnet, asyncdispatch]

proc main() {.async.} =
  var sock = newAsyncSocket()
  await sock.connect("example.com", Port(80))
  
  # ... do work ...
  
  sock.close()
  echo "Socket closed: ", sock.isClosed()  # true
  
  # Safe to call again:
  sock.close()  # no-op

waitFor main()
```

---

## Complete Examples

### Echo Server

```nim
import std/[asyncnet, asyncdispatch]

proc processClient(client: AsyncSocket) {.async.} =
  while true:
    let line = await client.recvLine()
    if line.len == 0:
      break
    await client.send(line & "\r\n")
  client.close()

proc serve() {.async.} =
  var server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(7777))
  server.listen()
  echo "Echo server listening on port 7777"
  
  while true:
    let client = await server.accept()
    asyncCheck processClient(client)

waitFor serve()
```

---

### Chat Server

```nim
import std/[asyncnet, asyncdispatch]

var clients: seq[AsyncSocket] = @[]

proc processClient(client: AsyncSocket) {.async.} =
  while true:
    let line = await client.recvLine()
    if line.len == 0:
      clients.keepIf(proc(c: AsyncSocket): bool = c != client)
      break
    for c in clients:
      if c != client:
        await c.send(line & "\r\n")
  client.close()

proc serve() {.async.} =
  var server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(12345))
  server.listen()
  echo "Chat server on port 12345"
  
  while true:
    let client = await server.accept()
    clients.add(client)
    asyncCheck processClient(client)

waitFor serve()
```

---

### HTTP Client (minimal)

```nim
import std/[asyncnet, asyncdispatch, strutils]

proc httpGet(host: string, port: Port, path: string): Future[string] {.async.} =
  let sock = await dial(host, port)
  await sock.send("GET " & path & " HTTP/1.0\r\nHost: " & host & "\r\n\r\n")
  
  result = ""
  while true:
    let chunk = await sock.recv(4096)
    if chunk.len == 0: break
    result.add chunk
  sock.close()

proc main() {.async.} =
  let response = await httpGet("example.com", Port(80), "/")
  let lines = response.splitLines()
  echo lines[0]  # HTTP status line

waitFor main()
```

---

### UDP Echo Server

```nim
import std/[asyncnet, asyncdispatch]

proc serve() {.async.} =
  var sock = newAsyncSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
  sock.bindAddr(Port(9999))
  echo "UDP server listening on port 9999"
  
  while true:
    let (data, address, port) = await sock.recvFrom(1024)
    echo "From ", address, ":", port, " -> ", data
    # Echo back:
    await sock.sendTo(address, port, data)

waitFor serve()
```

---

## Platform Availability Summary

| Proc | Linux | macOS | Windows | Notes |
|------|:-----:|:-----:|:-------:|-------|
| `newAsyncSocket` | ✓ | ✓ | ✓ | All overloads |
| `dial` | ✓ | ✓ | ✓ | |
| `connect` | ✓ | ✓ | ✓ | |
| `connectUnix` | ✓ | ✓ | ✗ | POSIX only |
| `bindAddr` | ✓ | ✓ | ✓ | |
| `bindUnix` | ✓ | ✓ | ✗ | POSIX only |
| `listen` | ✓ | ✓ | ✓ | |
| `accept` | ✓ | ✓ | ✓ | |
| `acceptAddr` | ✓ | ✓ | ✓ | |
| `send` (string) | ✓ | ✓ | ✓ | |
| `send` (pointer) | ✓ | ✓ | ✓ | |
| `sendTo` | ✓ | ✓ | ✓ | UDP; since 1.3 |
| `recv` | ✓ | ✓ | ✓ | |
| `recvInto` | ✓ | ✓ | ✓ | |
| `recvLine` | ✓ | ✓ | ✓ | |
| `recvLineInto` | ✓ | ✓ | ✓ | |
| `recvFrom` | ✓ | ✓ | ✓ | UDP; since 1.3 |
| `getLocalAddr` | ✓ | ✓ | ✓ | |
| `getPeerAddr` | ✓ | ✓ | ✓ | Not with nimNetLite |
| `getSockOpt` | ✓ | ✓ | ✓ | |
| `setSockOpt` | ✓ | ✓ | ✓ | |
| `getFd` | ✓ | ✓ | ✓ | |
| `isClosed` | ✓ | ✓ | ✓ | |
| `isSsl` | ✓ | ✓ | ✓ | |
| `hasDataBuffered` | ✓ | ✓ | ✓ | since 1.5 |
| `close` | ✓ | ✓ | ✓ | |
| `wrapSocket` | ✓ | ✓ | ✓ | requires `-d:ssl` |
| `wrapConnectedSocket` | ✓ | ✓ | ✓ | requires `-d:ssl` |
| `getPeerCertificates` | ✓ | ✓ | ✓ | requires `-d:ssl`; since 1.1 |
| `sslHandle` | ✓ | ✓ | ✓ | requires `-d:ssl` |

---

*Generated from Nim's standard library `std/asyncnet` module.*
