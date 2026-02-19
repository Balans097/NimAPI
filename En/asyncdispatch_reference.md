# `asyncdispatch` Module Reference (Nim)

> Nim standard library module for asynchronous I/O.  
> Implements an event dispatcher, the `Future[T]` type, and the `async`/`await` macro.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Types](#types)
3. [Event Dispatcher](#event-dispatcher)
4. [Sockets: Receiving Data](#sockets-receiving-data)
5. [Sockets: Sending Data](#sockets-sending-data)
6. [Sockets: Connecting & Accepting](#sockets-connecting--accepting)
7. [Timers & Delays](#timers--delays)
8. [Events, Processes, Signals](#events-processes-signals)
9. [Event Loop Control](#event-loop-control)
10. [Helper Functions](#helper-functions)
11. [Quick Reference](#quick-reference)

---

## Core Concepts

### Future and the Proactor Pattern

`Future[T]` holds a value that may not be available yet but will be at some point. When an asynchronous operation finishes, the Future is either *completed* with a value or *failed* with an exception.

```nim
import asyncdispatch

proc main() {.async.} =
  let fut = sleepAsync(100)   # returns Future[void]
  await fut                   # suspend without blocking the thread
  echo "100 ms have passed"

waitFor main()
```

### async / await

A procedure annotated with `{.async.}` must return `Future[T]` or nothing (implicitly `Future[void]`).  
Inside such a procedure, `await` suspends execution until the awaited Future completes, without blocking the OS thread.

```nim
proc fetchData(): Future[string] {.async.} =
  await sleepAsync(50)
  return "data"

proc main() {.async.} =
  let data = await fetchData()
  echo data   # "data"

waitFor main()
```

### Exception Handling

```nim
proc main() {.async.} =
  try:
    let data = await someAsyncOp()
    echo data
  except IOError as e:
    echo "Error: ", e.msg

waitFor main()
```

---

## Types

| Type | Description |
|------|-------------|
| `AsyncFD` | Asynchronous file descriptor (wraps `int`/`cint`) |
| `PDispatcher` | Event dispatcher (thread-local global) |
| `Callback` | `proc (fd: AsyncFD): bool {.closure, gcsafe.}` — return `true` to remove the handler |
| `AsyncEvent` | Thread-safe event object (like a semaphore/event handle) |

---

## Event Dispatcher

### `newDispatcher`

```nim
proc newDispatcher*(): owned PDispatcher
```

Creates a new dispatcher instance. Usually not needed directly — use `getGlobalDispatcher` instead.

```nim
let disp = newDispatcher()
setGlobalDispatcher(disp)
```

---

### `setGlobalDispatcher`

```nim
proc setGlobalDispatcher*(disp: sink PDispatcher)
```

Sets the provided dispatcher as the global dispatcher for the current thread.

```nim
setGlobalDispatcher(newDispatcher())
```

---

### `getGlobalDispatcher`

```nim
proc getGlobalDispatcher*(): PDispatcher
```

Returns the global dispatcher for the current thread. Creates one automatically if it doesn't exist yet.

```nim
let p = getGlobalDispatcher()
echo p.timers.len
```

---

### `getIoHandler`

```nim
# Windows:
proc getIoHandler*(disp: PDispatcher): Handle
# Unix:
proc getIoHandler*(disp: PDispatcher): Selector[AsyncData]
```

Returns the underlying I/O handler — an IO Completion Port handle on Windows or a selector on Unix. Useful for integration with external libraries.

---

### `register`

```nim
proc register*(fd: AsyncFD)
```

Registers a file descriptor with the global dispatcher. Must be called before using `fd` with any async operations.

```nim
let sock = createAsyncNativeSocket()
register(sock)
```

---

### `unregister` (fd)

```nim
proc unregister*(fd: AsyncFD)
```

Unregisters a file descriptor from the dispatcher.

```nim
unregister(mySocket)
```

---

### `contains`

```nim
proc contains*(disp: PDispatcher, fd: AsyncFD): bool
```

Checks whether `fd` is registered with dispatcher `disp`.

```nim
if getGlobalDispatcher().contains(myFd):
  echo "Descriptor is active"
```

---

### `hasPendingOperations`

```nim
proc hasPendingOperations*(): bool
```

Returns `true` if the global dispatcher has any pending work: registered handles, timers, or deferred callbacks.

```nim
while hasPendingOperations():
  poll()
```

---

### `closeSocket`

```nim
proc closeSocket*(socket: AsyncFD)
```

Closes a socket and unregisters it from the dispatcher in one step. Preferred over calling `close` and `unregister` separately.

```nim
closeSocket(clientSock)
```

---

### `callSoon`

```nim
proc callSoon*(cbproc: proc () {.gcsafe.}) {.gcsafe.}
```

Schedules `cbproc` to be called on the next iteration of the event loop. The callback is non-blocking.

```nim
callSoon(proc () = echo "called soon")
poll()
```

---

### `activeDescriptors`

```nim
proc activeDescriptors*(): int {.inline.}
```

Returns the current number of active file descriptors in the event loop. This is a cheap operation — no system call is made.

```nim
echo "Active connections: ", activeDescriptors()
```

---

### `maxDescriptors`

```nim
proc maxDescriptors*(): int {.raises: OSError.}
```

Returns the maximum number of file descriptors for the current process (requires a system call). Supported on Windows, Linux, macOS, BSD, Solaris.

```nim
echo "Max descriptors: ", maxDescriptors()
```

---

### `setInheritable`

```nim
proc setInheritable*(fd: AsyncFD, inheritable: bool): bool
```

Controls whether a file handle can be inherited by child processes. Availability is platform-dependent — check with `declared(setInheritable)`.

```nim
when declared(setInheritable):
  discard myFd.setInheritable(false)
```

---

## Sockets: Receiving Data

### `recv`

```nim
proc recv*(socket: AsyncFD, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[string])
```

Reads **up to** `size` bytes from `socket`. The Future completes when data is available, when a partial read occurs, or when the socket disconnects (result is `""`).

> ⚠️ The `Peek` flag is not supported on Windows.

```nim
proc handler() {.async.} =
  let data = await socket.recv(1024)
  if data.len == 0:
    echo "Connection closed"
  else:
    echo "Received: ", data

waitFor handler()
```

---

### `recvInto`

```nim
proc recvInto*(socket: AsyncFD, buf: pointer, size: int,
               flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

Reads **up to** `size` bytes from `socket` into an existing buffer `buf`. The Future resolves to the number of bytes read (0 on disconnect).

```nim
proc handler() {.async.} =
  var buf = newString(4096)
  let bytesRead = await socket.recvInto(addr buf[0], buf.len)
  buf.setLen(bytesRead)
  echo "Read ", bytesRead, " bytes"

waitFor handler()
```

---

### `recvFromInto`

```nim
proc recvFromInto*(socket: AsyncFD, data: pointer, size: int,
                   saddr: ptr SockAddr, saddrLen: ptr SockLen,
                   flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

Receives a UDP datagram from `socket` into `data`. The sender's address is written to `saddr`/`saddrLen`. The Future returns the size of the received packet.

```nim
proc udpHandler() {.async.} =
  var buf = newString(65535)
  var sender: Sockaddr_storage
  var senderLen = sizeof(sender).SockLen
  let size = await socket.recvFromInto(
    addr buf[0], buf.len,
    cast[ptr SockAddr](addr sender), addr senderLen
  )
  echo "UDP packet of ", size, " bytes"

waitFor udpHandler()
```

---

## Sockets: Sending Data

### `send` (from buffer)

```nim
proc send*(socket: AsyncFD, buf: pointer, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

Sends `size` bytes from buffer `buf` to `socket`. The Future completes once all data has been sent.

> ⚠️ If `buf` points to a GC-managed object, use `GC_ref`/`GC_unref` to prevent premature collection.

```nim
proc sendRaw() {.async.} =
  var data = "Hello"
  GC_ref(data)
  await socket.send(unsafeAddr data[0], data.len)
  GC_unref(data)

waitFor sendRaw()
```

---

### `send` (string)

```nim
proc send*(socket: AsyncFD, data: string,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

High-level overload: sends a `string` to `socket`. Safer than the pointer variant since memory management is handled automatically.

```nim
proc sendMsg() {.async.} =
  await socket.send("Hello, World!\n")

waitFor sendMsg()
```

---

### `sendTo`

```nim
proc sendTo*(socket: AsyncFD, data: pointer, size: int,
             saddr: ptr SockAddr, saddrLen: SockLen,
             flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

Sends a UDP datagram `data` to the destination `saddr`. The Future completes after the send.

```nim
proc udpSend() {.async.} =
  var msg = "ping"
  var dest: Sockaddr_in
  # ... fill in dest ...
  await socket.sendTo(
    addr msg[0], msg.len,
    cast[ptr SockAddr](addr dest), sizeof(dest).SockLen
  )

waitFor udpSend()
```

---

## Sockets: Connecting & Accepting

### `createAsyncNativeSocket`

```nim
proc createAsyncNativeSocket*(
  domain: Domain = Domain.AF_INET,
  sockType: SockType = SOCK_STREAM,
  protocol: Protocol = IPPROTO_TCP,
  inheritable = defined(nimInheritHandles)
): AsyncFD
```

Creates a native non-blocking socket and automatically registers it with the dispatcher. An overload accepting raw `cint` parameters is also available for non-standard protocols.

```nim
let sock = createAsyncNativeSocket()
# sock is ready for async operations
```

---

### `connect`

```nim
proc connect*(socket: AsyncFD, address: string, port: Port,
              domain = Domain.AF_INET): owned(Future[void])
```

Asynchronously connects an existing `socket` to `address:port`.

```nim
proc main() {.async.} =
  let sock = createAsyncNativeSocket()
  await sock.connect("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\n\r\n")
  let resp = await sock.recv(4096)
  echo resp

waitFor main()
```

---

### `dial`

```nim
proc dial*(address: string, port: Port,
           protocol: Protocol = IPPROTO_TCP): owned(Future[AsyncFD])
```

Creates a socket, resolves all IP addresses (both IPv4 and IPv6) for `address`, and connects to the first successful one. Returns an `AsyncFD` ready to send and receive.

```nim
proc main() {.async.} =
  let sock = await dial("example.com", Port(80))
  await sock.send("GET / HTTP/1.0\r\n\r\n")
  echo await sock.recv(4096)

waitFor main()
```

---

### `acceptAddr`

```nim
proc acceptAddr*(socket: AsyncFD,
                 flags = {SocketFlag.SafeDisconn},
                 inheritable = defined(nimInheritHandles)):
    owned(Future[tuple[address: string, client: AsyncFD]])
```

Accepts an incoming connection on `socket`. The Future returns a tuple with the client's IP address string and its socket. The client socket is automatically registered with the dispatcher.

```nim
proc server(listener: AsyncFD) {.async.} =
  while true:
    let (address, client) = await listener.acceptAddr()
    echo "Connected: ", address
    asyncCheck handleClient(client)

waitFor server(listenerSocket)
```

---

### `accept`

```nim
proc accept*(socket: AsyncFD,
             flags = {SocketFlag.SafeDisconn},
             inheritable = defined(nimInheritHandles)): owned(Future[AsyncFD])
```

Simplified version of `acceptAddr` that returns only the client socket without the remote address.

```nim
proc server(listener: AsyncFD) {.async.} =
  let client = await listener.accept()
  await client.send("Welcome!\n")

waitFor server(listenerSocket)
```

---

## Timers & Delays

### `sleepAsync`

```nim
proc sleepAsync*(ms: int | float): owned(Future[void])
```

Suspends the current async procedure for `ms` milliseconds. Accepts either `int` or `float`.

```nim
proc main() {.async.} =
  echo "Start"
  await sleepAsync(1000)    # 1 second
  await sleepAsync(500.0)   # 0.5 seconds
  echo "End"

waitFor main()
```

---

### `withTimeout`

```nim
proc withTimeout*[T](fut: Future[T], timeout: int): owned(Future[bool])
```

Returns a Future that completes with `true` if `fut` finishes before `timeout` milliseconds, or `false` if the timeout elapses first.

```nim
proc main() {.async.} =
  let longOp = sleepAsync(5000)
  let completed = await longOp.withTimeout(1000)
  if completed:
    echo "Finished in time"
  else:
    echo "Timed out!"

waitFor main()
```

---

### `addTimer`

```nim
proc addTimer*(timeout: int, oneshot: bool, cb: Callback)
```

Registers a low-level callback `cb` to be called after `timeout` ms.  
- `oneshot = true` — fires once  
- `oneshot = false` — fires repeatedly every `timeout` ms  

The callback should return `true` to remove itself, or `false` to keep firing.

> Available on platforms with `ioselSupportedPlatform` (Linux, macOS, BSD) and on Windows.

```nim
var count = 0
addTimer(500, false, proc(fd: AsyncFD): bool =
  inc count
  echo "Tick #", count
  return count >= 5  # stop after 5 ticks
)
runForever()
```

---

## Events, Processes, Signals

### `newAsyncEvent`

```nim
proc newAsyncEvent*(): AsyncEvent
```

Creates a new thread-safe `AsyncEvent` object. It is not automatically registered with the dispatcher — use `addEvent` to attach a callback.

```nim
let ev = newAsyncEvent()
```

---

### `trigger`

```nim
proc trigger*(ev: AsyncEvent)
```

Sets event `ev` to the signaled state. Safe to call from another thread.

```nim
ev.trigger()
```

---

### `addEvent`

```nim
proc addEvent*(ev: AsyncEvent, cb: Callback)
```

Registers callback `cb` to be called when `ev` transitions to the signaled state.

```nim
let ev = newAsyncEvent()
addEvent(ev, proc(fd: AsyncFD): bool =
  echo "Event triggered!"
  return true  # remove handler
)
ev.trigger()
poll()
```

---

### `unregister` (ev)

```nim
proc unregister*(ev: AsyncEvent)
```

Unregisters the event from the dispatcher.

```nim
unregister(ev)
```

---

### `close` (ev)

```nim
proc close*(ev: AsyncEvent)
```

Closes and frees all resources associated with the event.

```nim
ev.close()
```

---

### `addProcess`

```nim
proc addProcess*(pid: int, cb: Callback)
```

Registers callback `cb` to be called when the process with PID `pid` exits.

> Available on Windows and on Unix platforms with `ioselSupportedPlatform`.

```nim
addProcess(childPid, proc(fd: AsyncFD): bool =
  echo "Process exited"
  return true
)
```

---

### `addSignal` (Unix only)

```nim
proc addSignal*(signal: int, cb: Callback)
```

Registers callback `cb` to be called when signal `signal` is received.

> Available only on platforms with `ioselSupportedPlatform` (Linux, macOS, BSD).

```nim
import posix
addSignal(SIGTERM.int, proc(fd: AsyncFD): bool =
  echo "Received SIGTERM, shutting down..."
  return true
)
```

---

### `addRead`

```nim
proc addRead*(fd: AsyncFD, cb: Callback)
```

Watches `fd` for read readiness and calls `cb`. On Windows this uses a non-IOCP mechanism — prefer `recv`/`acceptAddr` unless adapting a Unix-style library.

```nim
addRead(myFd, proc(fd: AsyncFD): bool =
  echo "Data available to read"
  return true  # true = stop watching
)
```

---

### `addWrite`

```nim
proc addWrite*(fd: AsyncFD, cb: Callback)
```

Watches `fd` for write readiness and calls `cb`. On Windows this does not use IOCP.

```nim
addWrite(myFd, proc(fd: AsyncFD): bool =
  echo "Socket ready to write"
  return true
)
```

---

## Event Loop Control

### `poll`

```nim
proc poll*(timeout = 500)
```

Performs **one** pass of the event loop: waits for ready events and dispatches them. Raises `ValueError` if there are no registered operations.

```nim
# Manual event loop:
while hasPendingOperations():
  poll(100)
```

---

### `drain`

```nim
proc drain*(timeout = 500)
```

Processes **all** available events within `timeout` ms. Unlike `poll`, it does not stop after a single pass.

```nim
drain(2000)  # handle events for up to 2 seconds
```

---

### `runForever`

```nim
proc runForever*()
```

Starts an infinite event loop. Used when the application must run indefinitely (e.g. servers). Never returns.

```nim
runForever()
```

---

### `waitFor`

```nim
proc waitFor*[T](fut: Future[T]): T
```

**Blocks** the current thread until `fut` completes, spinning the event loop. Used to call async code from a synchronous context. **Never call inside an async procedure!**

```nim
proc fetchPage(): Future[string] {.async.} =
  await sleepAsync(100)
  return "page content"

let result = waitFor fetchPage()
echo result
```

---

### `readAll`

```nim
proc readAll*(future: FutureStream[string]): owned(Future[string]) {.async.}
```

Reads all string data from a `FutureStream` until it is completed and returns it as a single concatenated string.

```nim
proc main() {.async.} =
  let stream = newFutureStream[string]()
  # ... write to stream ...
  stream.complete()
  let allData = await readAll(stream)
  echo allData

waitFor main()
```

---

## Helper Functions

### `newCustom` (Windows only)

```nim
proc newCustom*(): CustomRef
```

Creates a new `OVERLAPPED`-based structure for use with IOCP on Windows. Used when implementing custom async operations at the WinSock level.

---

### `scheduleCallbacks` (Genode only)

```nim
proc scheduleCallbacks*(): bool {.discardable.}
```

Schedules processing of the `callSoon` queue on the Genode platform. Equivalent to a non-blocking `poll()` — useful for RPC servers that need to dispatch deferred work after retiring a request.

---

## Quick Reference

| Goal | Function |
|------|----------|
| Run async from sync code | `waitFor myFut` |
| Wait inside async proc | `await myFut` |
| Check without waiting | `asyncCheck myFut` |
| Delay execution | `await sleepAsync(ms)` |
| Add a timeout | `await myFut.withTimeout(ms)` |
| Create a socket | `createAsyncNativeSocket()` |
| Connect to host | `await dial(host, port)` |
| Accept a connection | `await listener.acceptAddr()` |
| Read data | `await socket.recv(size)` |
| Send data | `await socket.send(data)` |
| Infinite server loop | `runForever()` |
| Single event loop pass | `poll()` |
| All events (with drain) | `drain()` |
| Count active connections | `activeDescriptors()` |

---

### Known Limitations

- The effect system (`raises: []`) does not work with async procedures.
- Mutable parameters (`var T`) are not supported in async procs — use `ref T` instead.
- `waitFor` must never be called inside an async procedure.
- Never `discard` a Future directly — use `asyncCheck` to safely ignore results.
