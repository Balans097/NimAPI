# asyncdispatch.nim — module reference

> **Import:** `import std/asyncdispatch`
> **Scope:** Nim's asynchronous I/O implementation — the event-loop dispatcher, the `Future[T]` type, and support for the `{.async.}` macro / `await`. Re-exports `std/asyncfutures` (the `Future`, `FutureStream` types and operations on them) and `std/asyncstreams`, plus `Port` and `SocketFlag` from `std/net`/`std/nativesockets`.

The `asyncdispatch` module is the heart of asynchronous programming in Nim. It does not implement specific protocols (HTTP, TCP servers, etc.) — that's the job of higher-level modules (`asyncnet`, `asynchttpserver`, and so on). Instead, `asyncdispatch` provides the three foundations everything else is built on:

1. **The dispatcher (event loop)** — a global object that polls the operating system (via `epoll` on Linux, IO Completion Ports on Windows, `select`/`kqueue` on other platforms) for completed I/O operations, fired timers, and signals.
2. **`Future[T]`** — a "promise" of a value that will become available later. A future has three states: *pending*, *completed successfully* (holding a value of type `T`), or *completed with an error* (holding an exception).
3. **The `{.async.}` macro and `await`** — syntactic sugar that turns code written in the familiar sequential style into a chain of callbacks over futures, without you having to write those callbacks by hand.

---

## Contents

1. [Concepts: Future, async/await, asynchronous procedures](#concepts-future-asyncawait-asynchronous-procedures)
2. [Three ways to work with a future: `asyncCheck`, `waitFor`, `await`](#three-ways-to-work-with-a-future)
3. [Exception handling](#exception-handling)
4. [Core types](#core-types)
5. [Managing the dispatcher](#managing-the-dispatcher)
6. [The event loop: `poll`, `drain`, `runForever`, `waitFor`](#the-event-loop)
7. [Timers and timeouts: `sleepAsync`, `addTimer`, `withTimeout`](#timers-and-timeouts)
8. [Asynchronous sockets](#asynchronous-sockets)
9. [Connecting and address resolution: `connect`, `dial`](#connecting-and-address-resolution)
10. [Events, signals, and processes](#events-signals-and-processes)
11. [Descriptor diagnostics](#descriptor-diagnostics)
12. [Full example: echo server](#full-example-echo-server)
13. [Quick reference table](#quick-reference-table)

---

## Concepts: Future, async/await, asynchronous procedures

### `Future[T]`

A `Future[T]` is an object that **at the moment** may not hold a value of type `T`, but will receive one (or an error) at some point in the future. Any asynchronous function that performs I/O returns a `Future` immediately and **does not block** the thread of execution — but it also doesn't return the actual result right away. The request for the operation is made, and its result will appear later, once the dispatcher processes the corresponding event.

A callback can be attached to a future via `addCallback`, which will be called automatically once it completes:

```nim
var future = socket.recv(100)
future.addCallback(
  proc () =
    echo(future.read)
)
```

A future's state can be inspected with:

- `finished(fut)` — has the operation finished (successfully or with an error)?
- `failed(fut)` — did it finish specifically with an error?
- `read(fut)` — return the value (or, if there's an error, raise it as an exception).

### Asynchronous procedures (`{.async.}`)

A procedure marked with the `{.async.}` pragma must return `Future[T]` or have no return type at all (in which case `Future[void]` is assumed). Inside such a procedure, the `await` keyword is available:

```nim
proc fetchAndPrint(socket: AsyncFD) {.async.} =
  let data = await socket.recv(100)   # suspends execution until the data arrives
  echo "Received: ", data
```

`await fut`:

- if `fut` hasn't completed yet — suspends the current asynchronous procedure (but **not** the thread!) and hands control back to the dispatcher, which may run other asynchronous procedures in the meantime;
- once `fut` completes, execution of the current procedure resumes from the same point;
- if `fut` completed with an error — `await` **re-raises that exception** at the call site, so it can be caught with an ordinary `try/except`.

`await` can be used:

- as part of an expression in a variable declaration: `var data = await socket.recv(100)`;
- as a standalone expression for a `Future[void]`: `await socket.send("hello")`;
- directly on a future object: `await someFuture`.

> ⚠️ **Limitation.** Asynchronous procedures **do not support mutable (`var`) parameters**, such as `var int`. If you need to pass a value by reference, use a `ref` type (`ref int`, etc.) instead. Additionally, the effect system (`{.raises: [].}`) does not work correctly with asynchronous procedures.

---

## Three ways to work with a future

| Procedure       | Calling context           | Blocking?                                      |
|-----------------|----------------------------|--------------------------------------------------|
| `asyncCheck`    | non-async and async        | no                                                 |
| `waitFor`       | non-async only             | blocks the current thread                         |
| `await`         | async only                 | suspends the current procedure (not the thread)  |

- **`asyncCheck`** — "fire and forget", except that if the future completes with an error, the exception will be raised (rather than silently dropped). Use it when you don't care about the result but don't want to lose an error. It does not wait for completion — for that, use `waitFor` or `await`.
- **`waitFor`** — runs `poll()` in a loop until the given `Future` completes, **blocking the current thread**. Used to call asynchronous code from a synchronous context (e.g. from `main`). **Never** use `waitFor` inside a `{.async.}` procedure — it would start a **second**, nested event loop, which can lead to unexpected ordering, double-processing of events, or deadlocks.
- **`await`** — usable **only** inside `{.async.}` procedures; suspends the current procedure, letting the dispatcher do other work, until the future completes.

> ⚠️ **Never `discard` a future directly** — a future may contain an error, and `discard` will silently drop it. Instead of `discard someAsyncProc()`, use `asyncCheck someAsyncProc()`. `await` already checks for errors on its own, so it's safe to discard its result.

```nim
# Wrong — any error will be lost:
discard sock.send("data")

# Right — an error, if any, will be raised:
asyncCheck sock.send("data")

# In an async context — await checks for errors automatically:
proc worker() {.async.} =
  await sock.send("data")
```

---

## Exception handling

Inside `{.async.}` procedures, exceptions are handled the same way as in ordinary code — with `try`/`except`:

```nim
proc fetchSafely(sock: AsyncFD) {.async.} =
  try:
    let data = await sock.recv(100)
    echo("Received ", data)
  except:
    echo "Error while receiving data: ", getCurrentExceptionMsg()
```

An alternative approach is to use `yield` instead of `await`. Unlike `await`, `yield` does **not** automatically raise the exception, and the error state must be checked manually via `failed`:

```nim
proc fetchOrIgnore(sock: AsyncFD) {.async.} =
  var future = sock.recv(100)
  yield future
  if future.failed:
    echo "The operation failed, ignoring it"
  else:
    echo "Received: ", future.read
```

`yield` is useful when you want to explicitly decide how to handle an error without interrupting execution via an exception — for example, when processing many concurrent futures in a loop, where one of them failing shouldn't abort processing of the rest.

---

## Core types

### `AsyncFD`

```nim
type AsyncFD* = distinct int
```

An asynchronous file/socket descriptor. It's a "wrapper" around an ordinary OS-level descriptor (`SocketHandle`/`FileHandle` on Unix, `Handle` on Windows) that has been **registered with the dispatcher**. Most procedures in the module (`recv`, `send`, `accept`, `addRead`, etc.) take an `AsyncFD`, not a "raw" descriptor — registration is required so the dispatcher can track events on that descriptor.

### `Callback`

```nim
type Callback* = proc (fd: AsyncFD): bool {.closure, gcsafe.}
```

The callback type used by the low-level functions `addRead`, `addWrite`, `addTimer`, `addEvent`, `addSignal`, and `addProcess`. The return value tells the dispatcher whether to **keep** watching:

- `true` — stop watching (e.g. the event was one-shot);
- `false` — keep receiving notifications for this event.

### `AsyncEvent`

An opaque (`ptr`) type representing an event object that can be "triggered" (`trigger`) from any thread — this is the only thread-safe way to "wake up" the event loop from another thread.

### `PDispatcher` / `PDispatcherBase`

The dispatcher type. It holds a timer queue, a queue of deferred calls (`callbacks`, populated via `callSoon`), and, on Unix, a `selector` — a wrapper around `epoll`/`kqueue`/`select`; on Windows, an `ioPort` (IO Completion Port) and a set of registered descriptors, `handles`.

Generally there's no need to work with `PDispatcher` directly — `getGlobalDispatcher`/`setGlobalDispatcher` exist for that purpose.

---

## Managing the dispatcher

### `getGlobalDispatcher`

```nim
proc getGlobalDispatcher*(): PDispatcher
```

**What it does.** Returns the event dispatcher for the **current thread** (the dispatcher is stored in a `{.threadvar.}`, i.e. there's a separate one per thread). If no dispatcher has been created yet, one is created automatically on first call. In most programs you never call this directly — it's used internally by `recv`, `send`, `poll`, and so on.

```nim
let disp = getGlobalDispatcher()
echo "Dispatcher handle: ", disp.getIoHandler()
```

### `setGlobalDispatcher`

```nim
proc setGlobalDispatcher*(disp: sink PDispatcher)
```

**What it does.** Replaces the current thread's dispatcher with the one given. Useful in the rare cases where you need to completely "recreate" the event loop — for example in tests, to isolate state between runs — or when manually constructing a dispatcher via `newDispatcher()`. Raises an assertion error if the current (old) dispatcher still has unprocessed `callbacks`.

```nim
import std/asyncdispatch

# Create a "clean" dispatcher, e.g. before running a new test
setGlobalDispatcher(newDispatcher())
assert getGlobalDispatcher().callbacks.len == 0
```

### `register` / `unregister` / `contains`

```nim
proc register*(fd: AsyncFD)
proc unregister*(fd: AsyncFD)
proc contains*(disp: PDispatcher, fd: AsyncFD): bool
```

**What it does.** `register` adds the descriptor `fd` to the current thread's dispatcher — after this, the dispatcher begins tracking events on it (on Windows this means associating it with the IO Completion Port; on Unix, adding it to the `selector`). `unregister` stops watching the descriptor **without** closing it (closing is a separate operation; for sockets that's `closeSocket`). `contains` (used via the `in` operator) lets you check whether a descriptor is registered with a given dispatcher.

Most high-level procedures (`createAsyncNativeSocket`, `acceptAddr`) already call `register` for you — doing it manually is mostly needed when wrapping "raw" descriptors from third-party libraries.

```nim
import std/[asyncdispatch, nativesockets]

let raw = createNativeSocket()
let fd = raw.AsyncFD
register(fd)
assert fd in getGlobalDispatcher()

unregister(fd)
assert fd notin getGlobalDispatcher()
```

### `callSoon`

```nim
proc callSoon*(cbproc: proc () {.gcsafe.})
```

**What it does.** Schedules `cbproc` to be called as soon as possible — it will run as soon as control returns to the event loop (i.e. after the current "slice" of execution, but before the next iteration's timer/IO processing). It's a low-level primitive that, in particular, the `Future` implementation from `asyncfutures` is built on (future callbacks are always invoked via `callSoon`, to avoid deep recursion).

```nim
import std/asyncdispatch

var executed = false
callSoon(proc () = executed = true)
assert not executed
poll(0)               # runs the code scheduled via callSoon
assert executed
```

---

## The event loop

### `poll`

```nim
proc poll*(timeout = 500)
```

**What it does.** Runs **one** pass of the event loop: a single call to the underlying OS primitive (`epoll_wait`, `kqueue`, `GetQueuedCompletionStatus`, etc.) with a `timeout` in milliseconds, and processes any expired timers and deferred `callSoon` calls. If no descriptors, timers, or deferred calls are registered, it raises `ValueError` ("No handles or timers registered in dispatcher").

`poll` is the "driving force" of all asynchronous code: if nothing ever calls `poll` (directly or via `waitFor`/`runForever`), no asynchronous operation will ever complete, no matter how physically ready it is.

```nim
import std/asyncdispatch

var fut = sleepAsync(10)
while not fut.finished:
  poll()   # without this, fut would never complete
echo "Done"
```

### `drain`

```nim
proc drain*(timeout = 500)
```

**What it does.** Unlike `poll`, which performs exactly one pass, `drain` **keeps processing events** as long as they're available, until the overall `timeout` has elapsed or there are no more pending operations (`hasPendingOperations() == false`). Handy for "flushing" all accumulated events before terminating a program or before switching dispatchers.

```nim
import std/asyncdispatch

# Wait for all accumulated operations to be processed (but no longer than 1 second)
drain(1000)
```

### `runForever`

```nim
proc runForever*()
```

**What it does.** An infinite `while true: poll()` loop. Used in server applications that should run "forever", serving incoming connections. There's no "clean" way to exit this loop from within the process — typically an exception, `quit`, or an OS signal is used.

```nim
import std/[asyncdispatch, asyncnet]

proc serve() {.async.} =
  let server = newAsyncSocket()
  server.setSockOpt(OptReuseAddr, true)
  server.bindAddr(Port(8080))
  server.listen()
  while true:
    let client = await server.accept()
    asyncCheck handleClient(client)   # handleClient is your own {.async.} procedure

asyncCheck serve()
runForever()
```

### `waitFor`

```nim
proc waitFor*[T](fut: Future[T]): T
```

**What it does.** Runs `poll()` in a loop until `fut` completes, then returns `fut.read` (i.e. either the value of type `T`, or re-raises the stored exception if the future completed with an error). This is the **entry point** for running asynchronous code from a synchronous program — for example, from a non-`{.async.}` `proc main()`.

```nim
import std/asyncdispatch

proc fetchData(): owned(Future[string]) {.async.} =
  await sleepAsync(100)
  return "result"

let result = waitFor fetchData()
assert result == "result"
```

> ⚠️ Never call `waitFor` from inside another `{.async.}` procedure: it would start a **second**, nested `poll` loop, which can lead to unexpected execution order, double-processing of events, or deadlocks. Inside async code, always use `await`.

### `hasPendingOperations`

```nim
proc hasPendingOperations*(): bool
```

**What it does.** Returns `true` if the global dispatcher has at least one registered descriptor, an active timer, or a deferred (`callSoon`) call. Used internally by `drain`, and can also serve as the condition for your own loops: "keep working as long as there's something to process".

```nim
import std/asyncdispatch

while hasPendingOperations():
  poll()
echo "All operations completed"
```

---

## Timers and timeouts

### `sleepAsync`

```nim
proc sleepAsync*(ms: int | float): owned(Future[void])
```

**What it does.** Returns a `Future[void]` that completes after `ms` milliseconds (both integer and floating-point values are supported — for the latter, internal precision is increased to nanoseconds). This is the asynchronous, **non-blocking** counterpart to `os.sleep` — while waiting, the dispatcher can do other work.

```nim
import std/asyncdispatch

proc demo() {.async.} =
  echo "Start"
  await sleepAsync(100)   # 100 ms pause, without blocking other tasks
  echo "100 ms have passed"

waitFor demo()
```

### `addTimer`

```nim
proc addTimer*(timeout: int, oneshot: bool, cb: Callback)
```

**What it does.** A low-level counterpart to `sleepAsync` that works via `Callback` instead of `Future`. Registers a timer with period `timeout` ms:

- `oneshot = true` — fires **once**;
- `oneshot = false` — fires **periodically**, every `timeout` ms, until `cb` returns `true`.

Useful when you need periodic polling/heartbeat behavior without the overhead of creating a new `Future` on every iteration.

```nim
import std/asyncdispatch

var ticks = 0
addTimer(50, oneshot = false, cb = proc (fd: AsyncFD): bool =
  inc ticks
  echo "tick ", ticks
  result = ticks >= 3   # return true after the 3rd tick to stop the timer
)

while ticks < 3:
  poll()
```

### `withTimeout`

```nim
proc withTimeout*[T](fut: Future[T], timeout: int): owned(Future[bool])
```

**What it does.** Wraps `fut` in a new `Future[bool]` that completes with whichever of two events happens **first**:

- if `fut` completes first — the result is `true` (the original `fut` itself can still be read separately via `fut.read`/`fut.error` depending on its outcome);
- if `timeout` milliseconds elapse first — the result is `false`, and the original `fut` **keeps running** in the background (it is not cancelled — `Future` in Nim has no cancellation mechanism at all), but its further callbacks are dropped by this wrapper.

This is the basic building block for adding timeouts to operations that don't natively support them (e.g. `recv`).

```nim
import std/asyncdispatch

proc demo() {.async.} =
  let fut = sleepAsync(1000)            # a "slow" operation taking 1 second
  if await withTimeout(fut, 100):       # wait at most 100 ms
    echo "completed in time"
  else:
    echo "timed out!"

waitFor demo()  # prints "timed out!"
```

---

## Asynchronous sockets

This part of the module provides low-level asynchronous operations on `AsyncFD`. In application code, `std/asyncnet` (the `AsyncSocket` object wrapper) is more commonly used, but the procedures from `asyncdispatch` are its foundation.

### `createAsyncNativeSocket`

```nim
proc createAsyncNativeSocket*(domain: Domain = Domain.AF_INET,
                              sockType: SockType = SOCK_STREAM,
                              protocol: Protocol = IPPROTO_TCP,
                              inheritable = defined(nimInheritHandles)): AsyncFD

proc createAsyncNativeSocket*(domain: cint, sockType: cint,
                              protocol: cint,
                              inheritable = defined(nimInheritHandles)): AsyncFD
```

**What it does.** Creates a new non-blocking ("asynchronous") socket, sets it to non-blocking mode (`setBlocking(false)`), and **automatically registers** it with the current thread's dispatcher (`register`). Returns `osInvalidSocket.AsyncFD` if creating the socket at the OS level fails. The first overload takes the "friendly" enum types `Domain`/`SockType`/`Protocol` from `std/nativesockets`; the second takes "raw" `cint` values for compatibility with low-level code.

The `inheritable` parameter controls whether the socket will be inherited by child processes (default: no, matching common practice for server sockets).

```nim
import std/[asyncdispatch, nativesockets, net]

let sock = createAsyncNativeSocket(Domain.AF_INET, SockType.SOCK_STREAM, IPPROTO_TCP)
assert sock in getGlobalDispatcher()
```

### `recv`

```nim
proc recv*(socket: AsyncFD, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[string])
```

**What it does.** Reads **up to** `size` bytes from `socket`. The future completes once: all requested data has been read, part of the data has been read, or the socket has disconnected (in the latter case, with the value `""`). The `SocketFlag.Peek` flag is **not supported on Windows**. The `SafeDisconn` flag (enabled by default) suppresses the typical "errors" caused by a disconnect, turning them into a normal completion with an empty string rather than an exception — this simplifies handling clients that simply closed the connection.

```nim
import std/asyncdispatch

proc echoOnce(sock: AsyncFD) {.async.} =
  let data = await recv(sock, 1024)
  if data.len == 0:
    echo "Client disconnected"
  else:
    echo "Received: ", data
```

### `recvInto`

```nim
proc recvInto*(socket: AsyncFD, buf: pointer, size: int,
               flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

**What it does.** Same as `recv`, but writes data directly into a pre-allocated buffer `buf` (at least `size` bytes), and returns `Future[int]` — the **number** of bytes read (`0` on disconnect). Avoids an extra string allocation if you already have a buffer (e.g. from a buffer pool).

```nim
import std/asyncdispatch

proc readChunk(sock: AsyncFD): owned(Future[int]) =
  var buf = newString(4096)
  recvInto(sock, addr buf[0], buf.len)
```

### `send`

```nim
proc send*(socket: AsyncFD, data: string,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])

proc send*(socket: AsyncFD, buf: pointer, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

**What it does.** Sends data to `socket`; the future completes once **all** the data has been sent. The `string` overload is the most convenient for application code; the `pointer`/`size` overload works with raw memory.

> ⚠️ For the `pointer` overload: if `buf` points into a GC-managed object, you must keep it alive yourself (`GC_ref`/`GC_unref`) for the duration of the async operation — otherwise the garbage collector might free the memory before the send completes. The `string` overload does this for you automatically.

```nim
import std/asyncdispatch

proc reply(sock: AsyncFD) {.async.} =
  await send(sock, "HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nOK")
```

### `sendTo` / `recvFromInto`

```nim
proc sendTo*(socket: AsyncFD, data: pointer, size: int, saddr: ptr SockAddr,
             saddrLen: SockLen,
             flags = {SocketFlag.SafeDisconn}): owned(Future[void])

proc recvFromInto*(socket: AsyncFD, data: pointer, size: int,
                   saddr: ptr SockAddr, saddrLen: ptr SockLen,
                   flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

**What it does.** Versions for **datagram** (UDP) sockets, which don't require an established connection. `sendTo` sends data to a specific address `saddr`; `recvFromInto` receives a single datagram, writing the data into `data` and filling in `saddr`/`saddrLen` with the sender's address. In application code it's usually more convenient to go through `std/asyncnet` (`sendTo`/`recvFrom` with the `IpAddress` type), but for working directly with `SockAddr` — for example with non-standard socket domains — these procedures are the working low-level tool.

### `acceptAddr` / `accept`

```nim
proc acceptAddr*(socket: AsyncFD, flags = {SocketFlag.SafeDisconn},
                 inheritable = defined(nimInheritHandles)):
    owned(Future[tuple[address: string, client: AsyncFD]])

proc accept*(socket: AsyncFD,
             flags = {SocketFlag.SafeDisconn},
             inheritable = defined(nimInheritHandles)): owned(Future[AsyncFD])
```

**What it does.** Accepts a new incoming connection on the listening socket `socket`. `acceptAddr` returns **both** the client socket (`AsyncFD`, already automatically registered with the dispatcher) **and** the client's address as a string. `accept` is a simplified wrapper that returns only the client socket (it's implemented via `acceptAddr`, discarding the address).

If the connecting client disconnects during `accept` and the `SafeDisconn` flag is set (the default) — no error is raised, and `accept` is automatically retried.

```nim
import std/asyncdispatch

proc serverLoop(server: AsyncFD) {.async.} =
  while true:
    let (address, client) = await acceptAddr(server)
    echo "Connection from ", address
    asyncCheck handleClient(client)
```

### `closeSocket`

```nim
proc closeSocket*(socket: AsyncFD)
```

**What it does.** Closes the socket at the OS level **and** removes its registration from the dispatcher (a combination of `close` + `unregister`). After this call, `socket` must not be used in any further operations of this module.

```nim
import std/asyncdispatch

proc cleanup(sock: AsyncFD) =
  closeSocket(sock)
  assert sock notin getGlobalDispatcher()
```

### `setInheritable`

```nim
proc setInheritable*(fd: AsyncFD, inheritable: bool): bool
```

**What it does.** Controls whether the descriptor `fd` will be inherited by child processes (via `fork`/`exec` on Unix, or `CreateProcess` on Windows). Returns `true` on success. Availability on a given platform can be checked with `declared(setInheritable)`. Useful if a child process needs to be handed an already-open socket.

---

## Connecting and address resolution

### `connect`

```nim
proc connect*(socket: AsyncFD, address: string, port: Port,
              domain = Domain.AF_INET): owned(Future[void])
```

**What it does.** Establishes a connection to `address:port` using an already-created socket `socket`, with the given `domain` (address family — IPv4/IPv6). This is the "low-level" variant: you must pick a `domain` that matches the one the socket was created with (on Unix, a mismatch triggers an `assert`).

### `dial`

```nim
proc dial*(address: string, port: Port,
           protocol: Protocol = IPPROTO_TCP): owned(Future[AsyncFD])
```

**What it does.** A high-level alternative to `connect`: it **itself** performs DNS resolution of `address` (via `getAddrInfo`) and iterates over **all** the addresses returned (both IPv4 **and** IPv6), trying to connect to each in turn until one succeeds. It returns an already-connected `AsyncFD`, registered with the dispatcher and ready to use. Unlike `connect`, it doesn't require the socket to already exist, nor does it require you to know in advance which IP version will be used — `dial` creates a socket of the appropriate type itself.

```nim
import std/[asyncdispatch, net]

proc fetchExample() {.async.} =
  let sock = await dial("example.com", Port(80))
  await send(sock, "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
  let response = await recv(sock, 4096)
  echo response
  closeSocket(sock)

waitFor fetchExample()
```

> **`dial` vs. `connect` + `createAsyncNativeSocket`.** If the server's address could be either IPv4 or IPv6 (the typical case for domain names), and you don't care which specific socket ends up being created — use `dial`. If you're working with an already-existing socket of a specific domain (e.g. a Unix domain socket, or a socket already accepted via `acceptAddr`), use `connect`.

---

## Events, signals, and processes

This group of procedures lets you plug arbitrary OS notification sources into the event loop — custom events (for waking up from another thread), Unix signals, and the exit of external processes.

### `newAsyncEvent` / `trigger` / `close` / `addEvent`

```nim
proc newAsyncEvent*(): AsyncEvent
proc trigger*(ev: AsyncEvent)
proc close*(ev: AsyncEvent)
proc addEvent*(ev: AsyncEvent, cb: Callback)
```

**What it does.** `newAsyncEvent` creates a new event object (a thread-safe synchronization primitive). `addEvent` registers a callback `cb` that will be called when the event becomes signaled. `trigger` puts the event into the signaled state — **it can be safely called from any thread**, including one that didn't start the event loop, which makes `AsyncEvent` the primary mechanism for "waking up" the dispatcher from the outside. `close` releases the event's resources.

```nim
import std/[asyncdispatch, os]

let ev = newAsyncEvent()
var notified = false
addEvent(ev, proc (fd: AsyncFD): bool =
  notified = true
  result = true   # stop watching after the first signal
)

# From another thread (here, synchronously for the sake of the example):
trigger(ev)

while not notified:
  poll()
close(ev)
```

### `addSignal` *(Unix only)*

```nim
proc addSignal*(signal: int, cb: Callback)
```

**What it does.** Registers `cb` to be called when the process receives the Unix signal `signal` (e.g. `SIGTERM`, `SIGUSR1`). Allows signals to be handled as part of the event loop, without interrupting asynchronous code in the way the OS's standard signal-handling mechanism would.

```nim
import std/[asyncdispatch, posix]

addSignal(SIGUSR1.int, proc (fd: AsyncFD): bool =
  echo "Received SIGUSR1"
  result = false   # keep listening for the signal
)
```

### `addProcess`

```nim
proc addProcess*(pid: int, cb: Callback)
```

**What it does.** Registers `cb` to be called when the process with ID `pid` exits. Useful for asynchronously waiting on child processes without a blocking `os.waitForExit`.

### `addRead` / `addWrite` *(low-level, mainly Unix; limited on Windows)*

```nim
proc addRead*(fd: AsyncFD, cb: Callback)
proc addWrite*(fd: AsyncFD, cb: Callback)
```

**What it does.** Register a callback that's called when `fd` becomes ready for reading/writing, respectively. This is the module's lowest level — used to adapt third-party descriptors that are "synchronous by nature" (pipes, devices, third-party libraries) to the event loop. `cb` should return `true` to stop watching, or `false` to keep receiving notifications.

> ⚠️ **On Windows**, this is not the "native" IOCP mechanism but an emulation via `RegisterWaitForSingleObject` — only use it if you genuinely need to (typically when porting Unix-oriented libraries). If you use `addRead`/`addWrite` on Windows for a socket, **don't mix** this with `recv`/`send`/`accept` from this same module — use the low-level `nativesockets.recv`/`nativesockets.send`/`nativesockets.accept` instead.

---

## Descriptor diagnostics

### `activeDescriptors`

```nim
proc activeDescriptors*(): int {.inline.}
```

**What it does.** Returns the current number of active file descriptors tracked by the **current thread's** dispatcher. This is a cheap operation that makes no system calls (on Windows it reads the size of the `handles` set; on Unix, the `selector`'s count).

```nim
import std/asyncdispatch

echo "Active descriptors: ", activeDescriptors()
```

### `maxDescriptors`

```nim
proc maxDescriptors*(): int {.raises: OSError.}
```

**What it does.** Returns the **maximum** number of file descriptors the current process is allowed to open (the system limit — `RLIMIT_NOFILE` on Unix; an approximate constant on Windows). Unlike `activeDescriptors`, this involves a system call. Useful for sizing connection pools: if `activeDescriptors()` is close to `maxDescriptors()`, it's time to start refusing new connections rather than crashing with `EMFILE`.

```nim
import std/asyncdispatch

let limit = maxDescriptors()
if activeDescriptors() > limit - 100:
  echo "Approaching the descriptor limit!"
```

### `getFuturesInProgress` and tracking "stuck" futures

When compiled with `-d:futureLogging`, the `asyncfutures` module (re-exported here) keeps track of every incomplete `Future`. The `getFuturesInProgress` procedure (from `asyncfutures`) returns a list of them along with the stack traces captured at the moment each one was created — an invaluable tool when diagnosing memory leaks caused by "forgotten" futures that never complete.

```sh
nim c -d:futureLogging --threads:on myapp.nim
```

---

## Full example: echo server

This example ties together the module's main procedures: `dial`/`acceptAddr` for network I/O, `recv`/`send` for data exchange, `asyncCheck`/`runForever` for driving the event loop, and `sleepAsync` to demonstrate a non-blocking delay.

```nim
import std/[asyncdispatch, nativesockets, net]

proc handleClient(client: AsyncFD) {.async.} =
  defer: closeSocket(client)
  while true:
    let line = await recv(client, 1024)
    if line.len == 0:
      echo "Client disconnected"
      break
    await sleepAsync(10)              # simulate a small processing delay
    await send(client, "echo: " & line)

proc serve(port: Port) {.async.} =
  let server = createAsyncNativeSocket()
  server.SocketHandle.setSockOptInt(SOL_SOCKET, SO_REUSEADDR, 1)
  server.SocketHandle.bindAddr(port)
  server.SocketHandle.listen()

  while true:
    let (address, client) = await acceptAddr(server)
    echo "New connection from ", address
    asyncCheck handleClient(client)

asyncCheck serve(Port(7777))
runForever()
```

---

## Quick reference table

| Task | Procedure(s) |
|---|---|
| Run async code from a synchronous `main` | `waitFor` |
| Run a "background" task without losing errors | `asyncCheck` |
| Wait for another future inside `{.async.}` | `await` |
| One pass of the event loop | `poll` |
| Process all accumulated events | `drain` |
| Infinite event loop (server) | `runForever` |
| Non-blocking delay | `sleepAsync` |
| Periodic callback-based timer | `addTimer` |
| Bound a future with a timeout | `withTimeout` |
| Create an asynchronous socket | `createAsyncNativeSocket` |
| Connect by hostname (IPv4/IPv6) | `dial` |
| Connect to an already-known address | `connect` |
| Accept a connection | `acceptAddr` / `accept` |
| Read/write socket data | `recv`, `recvInto`, `send` |
| UDP exchange | `sendTo`, `recvFromInto` |
| Close a socket and remove its registration | `closeSocket` |
| Register a "raw" descriptor | `register` / `unregister` / `in` |
| Wake the event loop from another thread | `newAsyncEvent` + `trigger` + `addEvent` |
| Wait for an OS signal (Unix) | `addSignal` |
| Wait for a process to exit | `addProcess` |
| Defer a call with no delay | `callSoon` |
| How many descriptors are open / allowed | `activeDescriptors`, `maxDescriptors` |
| Check whether there's unfinished work | `hasPendingOperations` |
