# Nim `selectors` Module — Complete Reference

> **Module purpose:** High-level, efficient I/O multiplexing. Allows a single thread (or a thread-safe pool) to monitor many file descriptors, sockets, timers, signals, processes, and user-defined events simultaneously — without busy-waiting.
>
> **Underlying OS primitives:** `epoll` (Linux), `kqueue` (macOS/BSD), `poll` (Solaris/fallback), `select` (Windows). The right backend is chosen automatically; you can override it with `-d:nimIoselector=epoll|kqueue|poll|select`.
>
> **Thread safety:** Compile with `-d:threadsafe --threads:on` to get the thread-safe variant.

---

## Table of Contents

1. [Types and constants](#types-and-constants)
   - [`ioselSupportedPlatform`](#ioselSupportedPlatform)
   - [`Event`](#event)
   - [`ReadyKey`](#readykey)
   - [`SelectEvent`](#selectevent)
   - [`Selector[T]`](#selectort)
   - [`IOSelectorsException`](#ioselectorsexception)
2. [Selector lifecycle](#selector-lifecycle)
   - [`newSelector`](#newselector)
   - [`close` (Selector)](#close-selector)
   - [`getFd`](#getfd)
   - [`isEmpty`](#isempty)
   - [`maxDescriptors`](#maxdescriptors)
3. [Registering descriptors](#registering-descriptors)
   - [`registerHandle`](#registerhandle)
   - [`updateHandle`](#updatehandle)
   - [`registerTimer`](#registertimer)
   - [`registerSignal`](#registersignal)
   - [`registerProcess`](#registerprocess)
   - [`registerVnode`](#registervnode)
4. [User events](#user-events)
   - [`newSelectEvent`](#newselectevent)
   - [`trigger`](#trigger)
   - [`close` (SelectEvent)](#close-selectevent)
   - [`registerEvent`](#registerevent)
   - [`unregister` (SelectEvent)](#unregister-selectevent)
5. [Unregistering descriptors](#unregistering-descriptors)
   - [`unregister` (fd)](#unregister-fd)
6. [Waiting for events](#waiting-for-events)
   - [`select`](#select)
   - [`selectInto`](#selectinto)
7. [Accessing associated data](#accessing-associated-data)
   - [`getData`](#getdata)
   - [`setData`](#setdata)
   - [`withData` (1-branch)](#withdata-1-branch)
   - [`withData` (2-branch)](#withdata-2-branch)
   - [`contains`](#contains)

---

## Types and constants

### `ioselSupportedPlatform`

```nim
const ioselSupportedPlatform*: bool
```

A compile-time constant that is `true` when the current platform is **fully** supported (macOS, FreeBSD, NetBSD, OpenBSD, DragonFly, NuttX, Linux excluding Android/Emscripten). On partially-supported platforms (Windows, Solaris, Android) some features (signals, processes, vnodes) are unavailable.

Use it to conditionally compile platform-specific code:

```nim
when ioselSupportedPlatform:
  discard sel.registerSignal(SIGTERM, myData)
else:
  echo "Signal watching not available on this OS"
```

---

### `Event`

```nim
type Event* {.pure.} = enum
  Read, Write, Timer, Signal, Process, Vnode, User, Error,
  VnodeWrite, VnodeDelete, VnodeExtend, VnodeAttrib,
  VnodeLink, VnodeRename, VnodeRevoke
```

An enumeration describing **what happened** to a descriptor. Values are used both when registering (to declare interest) and when reading results (to check what fired).

| Value | Meaning |
|---|---|
| `Read` | Data is available to read (or a new connection is pending on a listening socket) |
| `Write` | Buffer space is available; writing will not block |
| `Timer` | A registered timer has elapsed |
| `Signal` | A Unix signal was delivered |
| `Process` | A watched process has exited |
| `Vnode` | A BSD/macOS filesystem event occurred (use specific `Vnode*` variants) |
| `User` | A user-defined `SelectEvent` was triggered |
| `Error` | An error occurred; inspect `ReadyKey.errorCode` |
| `VnodeWrite` … `VnodeRevoke` | Fine-grained BSD/macOS file-change notifications |

```nim
# Registering interest in both read and write:
sel.registerHandle(sock, {Event.Read, Event.Write}, myData)

# Checking what fired:
for key in sel.select(1000):
  if Event.Read in key.events:
    let data = sock.recv(buf, 1024)
```

---

### `ReadyKey`

```nim
type ReadyKey* = object
  fd*: int               # the descriptor that fired
  events*: set[Event]    # which events fired
  errorCode*: OSErrorCode # non-zero when Event.Error is present
```

A value returned by `select` / `selectInto`. It tells you **which descriptor** became ready and **what kind of readiness** was detected.

```nim
for key in sel.select(500):
  if Event.Error in key.events:
    echo "fd ", key.fd, " error: ", osErrorMsg(key.errorCode)
  elif Event.Read in key.events:
    echo "fd ", key.fd, " is readable"
```

---

### `SelectEvent`

```nim
type SelectEvent* = object
```

An opaque handle representing a **user-defined event** — a lightweight cross-thread (or cross-coroutine) signalling primitive. Create with `newSelectEvent`, fire with `trigger`, destroy with `close`.

---

### `Selector[T]`

```nim
type Selector*[T] = ref object
```

The central object of the module. `T` is the type of **application data** you attach to each registered descriptor. Using a concrete type (e.g. `Selector[MyConn]`) lets you avoid hash-map lookups — the data travels with the event.

```nim
type ConnData = object
  id: int
  buf: string

var sel = newSelector[ConnData]()
```

---

### `IOSelectorsException`

```nim
type IOSelectorsException* = object of CatchableError
```

Raised when an internal OS call fails (e.g. `epoll_create` returns an error). Catch it if you need to handle selector failures gracefully.

```nim
try:
  var sel = newSelector[int]()
except IOSelectorsException as e:
  echo "Could not create selector: ", e.msg
```

---

## Selector lifecycle

### `newSelector`

```nim
proc newSelector*[T](): Selector[T]
```

**Creates and initialises a new selector.** Internally this opens an `epoll`/`kqueue`/`poll`/`select` instance. No descriptors are registered yet.

The type parameter `T` is the type of per-descriptor data you will associate with each registered fd. Using `void` is valid if you need no extra data.

```nim
var sel = newSelector[string]()  # each fd will carry a string label
```

---

### `close` (Selector)

```nim
proc close*[T](s: Selector[T])
```

**Releases all OS resources** held by the selector (the epoll/kqueue fd, internal buffers, locks in threadsafe mode). After calling `close`, the selector must not be used.

```nim
var sel = newSelector[int]()
# ... use sel ...
sel.close()
```

---

### `getFd`

```nim
proc getFd*[T](s: Selector[T]): int
```

Returns the **underlying OS file descriptor** of the selector itself (the epoll or kqueue fd). Useful when you want to nest selectors or pass the selector fd to another multiplexer. Returns `-1` for `poll` and `select` backends, which have no single fd.

```nim
let fd = sel.getFd()
if fd >= 0:
  echo "Selector lives at fd ", fd
```

---

### `isEmpty`

```nim
template isEmpty*[T](s: Selector[T]): bool
```

Returns `true` when **no descriptors** are currently registered in the selector. Handy as a loop-exit condition when you drain descriptors one by one.

```nim
while not sel.isEmpty():
  let events = sel.select(100)
  for key in events:
    sel.unregister(key.fd)
```

---

### `maxDescriptors`

```nim
template maxDescriptors*(): int
```

*(Available on Windows, Linux, macOS, BSD, Solaris)*

Returns the **maximum number of file descriptors** the current process may have open, obtained via `getrlimit(RLIMIT_NOFILE)` (or a fixed constant on Windows). Useful for pre-allocating data structures.

```nim
echo "Process fd limit: ", maxDescriptors()
```

---

## Registering descriptors

### `registerHandle`

```nim
proc registerHandle*[T](s: Selector[T], fd: int | SocketHandle,
                        events: set[Event], data: T)
```

**Adds a file descriptor or socket** to the selector and declares which events to watch for (`Read`, `Write`, or both). The `data` value is stored internally and returned alongside every event for this fd — use it to hold connection state, buffers, or any per-fd context.

Each fd may be registered **at most once**. Attempting to register the same fd twice raises `IOSelectorsException`.

```nim
var sel = newSelector[string]()
let sock: SocketHandle = createSocket()
sel.registerHandle(sock, {Event.Read}, "client-1")

for key in sel.select(2000):
  if Event.Read in key.events:
    echo sel.getData(key.fd)  # prints "client-1"
```

---

### `updateHandle`

```nim
proc updateHandle*[T](s: Selector[T], fd: int | SocketHandle,
                      events: set[Event])
```

**Changes the event mask** of an already-registered fd without unregistering it. Useful for switching a connection between read-interest and write-interest as your application state changes (e.g. switch to `Write` once you have data to send, then back to `Read` when the send buffer is empty).

```nim
# Initially interested only in reads:
sel.registerHandle(sock, {Event.Read}, myData)

# Now we have data to send — switch to write:
sel.updateHandle(sock, {Event.Write})

# After sending, go back to reading:
sel.updateHandle(sock, {Event.Read})
```

---

### `registerTimer`

```nim
proc registerTimer*[T](s: Selector[T], timeout: int, oneshot: bool,
                       data: T): int {.discardable.}
```

**Registers a timer** that fires after `timeout` milliseconds. If `oneshot` is `true` the timer fires exactly once and then stops. If `false`, it fires repeatedly every `timeout` ms. Returns the internal fd used for this timer (needed if you want to unregister it early).

The `data` value is delivered with the `Timer` event, just like with `registerHandle`.

```nim
# One-shot 5-second timeout:
let timerFd = sel.registerTimer(5000, true, "deadline")

for key in sel.select(-1):   # block until something fires
  if Event.Timer in key.events:
    echo "Timer fired: ", sel.getData(key.fd)
    break
```

```nim
# Periodic heartbeat every 1 second:
discard sel.registerTimer(1000, false, "heartbeat")
```

---

### `registerSignal`

```nim
proc registerSignal*[T](s: Selector[T], signal: int,
                        data: T): int {.discardable.}
```

*(Not supported on Windows)*

**Intercepts a Unix signal** and delivers it as a `Signal` event through the selector, rather than interrupting your process asynchronously. This lets you handle signals safely inside your event loop.

Returns the internal fd for the signal (for later `unregister` if needed).

```nim
import std/posix

discard sel.registerSignal(SIGTERM.int, "got-sigterm")

for key in sel.select(-1):
  if Event.Signal in key.events:
    echo "Received signal: ", sel.getData(key.fd)
    break
```

---

### `registerProcess`

```nim
proc registerProcess*[T](s: Selector[T], pid: int,
                         data: T): int {.discardable.}
```

**Watches a child process** identified by `pid`. When the process exits, a `Process` event is delivered. This is an alternative to blocking on `waitpid` — you can handle process exit inside your event loop alongside I/O events.

```nim
import std/osproc

let child = startProcess("sleep", args=["3"])
discard sel.registerProcess(child.processID, "child-done")

for key in sel.select(-1):
  if Event.Process in key.events:
    echo "Child exited"
    child.close()
    break
```

---

### `registerVnode`

```nim
proc registerVnode*[T](s: Selector[T], fd: cint, events: set[Event],
                       data: T)
```

*(BSD and macOS only)*

**Monitors filesystem-level changes** on an open file descriptor using kqueue's `EVFILT_VNODE`. You select which specific changes interest you from the `Vnode*` event set:

| Event | Triggered when |
|---|---|
| `VnodeWrite` | Data was written to the file |
| `VnodeDelete` | The file was unlinked |
| `VnodeExtend` | The file grew in size |
| `VnodeAttrib` | File attributes (permissions, timestamps) changed |
| `VnodeLink` | The hard-link count changed |
| `VnodeRename` | The file was renamed |
| `VnodeRevoke` | The file was revoked |

```nim
when bsdPlatform or defined(macosx):
  import std/posix
  let fd = open("config.toml", O_RDONLY)
  sel.registerVnode(fd, {VnodeWrite, VnodeDelete}, "config-file")

  for key in sel.select(-1):
    if VnodeDelete in key.events:
      echo "Config file was deleted!"
```

---

## User events

User events let you wake up the selector from **another thread or coroutine** — without any OS signal or network traffic.

### `newSelectEvent`

```nim
proc newSelectEvent*(): SelectEvent
```

**Creates a new user-defined event object.** Internally this is usually a self-pipe or eventfd pair. You must register it with a selector via `registerEvent` before it can be used.

```nim
var ev = newSelectEvent()
```

---

### `trigger`

```nim
proc trigger*(ev: SelectEvent)
```

**Signals the event**, causing the selector that has `ev` registered to wake up and report a `User` event. Safe to call from any thread.

```nim
# In another thread or callback:
ev.trigger()
```

---

### `close` (SelectEvent)

```nim
proc close*(ev: SelectEvent)
```

**Releases OS resources** held by the event (the underlying pipe or eventfd). Call this after you have unregistered the event from the selector.

```nim
sel.unregister(ev)
ev.close()
```

---

### `registerEvent`

```nim
proc registerEvent*[T](s: Selector[T], ev: SelectEvent, data: T)
```

**Adds a user event to the selector's watch list.** After this call, every `trigger(ev)` will produce a `User` event in the next `select`/`selectInto` call.

```nim
var sel = newSelector[string]()
var ev  = newSelectEvent()
sel.registerEvent(ev, "wakeup-signal")

# From another thread:
ev.trigger()

# In event loop:
for key in sel.select(5000):
  if Event.User in key.events:
    echo sel.getData(key.fd)  # "wakeup-signal"
```

---

### `unregister` (SelectEvent)

```nim
proc unregister*[T](s: Selector[T], ev: SelectEvent)
```

**Removes a user event** from the selector. After unregistering, `trigger` will no longer cause the selector to report an event.

```nim
sel.unregister(ev)
ev.close()
```

---

## Unregistering descriptors

### `unregister` (fd)

```nim
proc unregister*[T](s: Selector[T], fd: int | SocketHandle | cint)
```

**Removes a file descriptor, socket, timer, signal, or process fd** from the selector. The associated `data` is discarded. After unregistering, no more events will be reported for this fd.

```nim
# Client disconnected — remove from selector:
sel.unregister(clientSock)
clientSock.close()
```

---

## Waiting for events

### `select`

```nim
proc select*[T](s: Selector[T], timeout: int): seq[ReadyKey]
```

**Blocks until at least one registered descriptor becomes ready**, then returns a `seq` of `ReadyKey` objects describing all ready descriptors.

- `timeout > 0` — wait at most that many **milliseconds**.
- `timeout = 0` — return immediately (non-blocking poll).
- `timeout = -1` — block **indefinitely** until at least one event fires.

`select` allocates a new `seq` on every call. If you need zero-allocation event loops, use `selectInto` instead.

```nim
while true:
  let ready = sel.select(1000)   # wait up to 1 second
  for key in ready:
    if Event.Read in key.events:
      handleRead(key.fd)
    if Event.Error in key.events:
      handleError(key.fd, key.errorCode)
```

---

### `selectInto`

```nim
proc selectInto*[T](s: Selector[T], timeout: int,
                    results: var openArray[ReadyKey]): int
```

**Same semantics as `select`**, but writes results into a **pre-allocated array** you provide, avoiding heap allocation. Returns the number of events written into `results`.

Use this in performance-critical loops where you want full control over allocation.

```nim
var buf: array[64, ReadyKey]

while true:
  let n = sel.selectInto(500, buf)
  for i in 0 ..< n:
    let key = buf[i]
    if Event.Write in key.events:
      handleWrite(key.fd)
```

---

## Accessing associated data

### `getData`

```nim
proc getData*[T](s: Selector[T], fd: SocketHandle | int): var T
```

**Returns a mutable reference** to the application data stored for `fd`. If `fd` is not registered, the default/zero value for `T` is returned (no exception). Because it returns `var T`, you can modify the data in place.

```nim
type Conn = object
  bytesRead: int

var sel = newSelector[Conn]()
sel.registerHandle(sock, {Event.Read}, Conn(bytesRead: 0))

for key in sel.select(-1):
  sel.getData(key.fd).bytesRead += 100   # modify in place
```

---

### `setData`

```nim
proc setData*[T](s: Selector[T], fd: SocketHandle | int, data: var T): bool
```

**Replaces the application data** stored for `fd` with a new value. Returns `true` on success, `false` if `fd` is not registered in the selector.

```nim
var updated = Conn(bytesRead: 999)
if not sel.setData(sock, updated):
  echo "fd not found in selector"
```

---

### `withData` (1-branch)

```nim
template withData*[T](s: Selector[T], fd: SocketHandle | int, value,
                      body: untyped)
```

**Safely retrieves the data for `fd`** into `value` and executes `body` — but **only if `fd` is registered**. If the fd is absent, the body is silently skipped. `value` is mutable and changes are persisted.

This is the idiomatic way to access per-fd data without checking `contains` manually.

```nim
sel.withData(key.fd, conn) do:
  conn.bytesRead += bytesReceived
  if conn.bytesRead > 1_000_000:
    sel.unregister(key.fd)
```

---

### `withData` (2-branch)

```nim
template withData*[T](s: Selector[T], fd: SocketHandle | int, value,
                      body1, body2: untyped)
```

**Two-branch version of `withData`**: `body1` runs when `fd` is registered (with `value` bound), `body2` runs when `fd` is not found. Use the second branch for error handling or logging.

```nim
sel.withData(key.fd, conn) do:
  echo "Handling fd ", key.fd, " bytes so far: ", conn.bytesRead
do:
  echo "Unknown fd: ", key.fd
  # fd disappeared between select() and here
```

---

### `contains`

```nim
proc contains*[T](s: Selector[T], fd: SocketHandle | int): bool {.inline.}
```

**Returns `true`** if `fd` is currently registered in the selector. A simple existence check — equivalent to the condition guard in `withData`, but explicit.

```nim
if sel.contains(sock):
  sel.unregister(sock)
  sock.close()
```

---

## Full working example

```nim
import std/selectors, std/nativesockets, std/net

type ConnData = object
  label: string

var sel = newSelector[ConnData]()

# Register a listening socket
let server = newSocket()
server.bindAddr(Port(8080))
server.listen()
setBlocking(server.getFd(), false)
sel.registerHandle(server.getFd().int, {Event.Read}, ConnData(label: "server"))

# Register a 10-second watchdog timer
discard sel.registerTimer(10_000, true, ConnData(label: "watchdog"))

# Register a user event for graceful shutdown
var shutdownEv = newSelectEvent()
sel.registerEvent(shutdownEv, ConnData(label: "shutdown"))

while not sel.isEmpty():
  for key in sel.select(500):
    sel.withData(key.fd, conn) do:
      case conn.label
      of "server":
        # Accept new connection
        let (client, _) = server.accept()
        setBlocking(client.getFd(), false)
        sel.registerHandle(client.getFd().int, {Event.Read},
                           ConnData(label: "client"))
      of "watchdog":
        echo "Watchdog fired — shutting down"
        shutdownEv.trigger()
      of "shutdown":
        sel.unregister(shutdownEv)
        shutdownEv.close()
        sel.unregister(server.getFd().int)
        server.close()
      of "client":
        if Event.Read in key.events:
          # read data ...
          sel.unregister(key.fd)
      else: discard

sel.close()
```

---

*Reference covers Nim standard library `std/selectors` as found in the module source. Platform-specific behaviour (epoll/kqueue/poll/select) is abstracted away by the module.*
