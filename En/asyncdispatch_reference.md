# std/asyncdispatch — module reference

> **Import:** `import std/asyncdispatch`
> **Scope:** asynchronous I/O (sockets, timers, signals, child processes), the event dispatcher, and the `Future`/`{.async.}`/`await` mechanism.

The module implements an event loop (the dispatcher) on top of which asynchronous operations are built. The dispatcher is polled by the `poll` procedure (or by something that calls `poll` for you — `waitFor`, `drain`, `runForever`). Every asynchronous operation is represented by a `Future[T]` object — a value that doesn't exist yet but may appear later; readiness is checked with `finished(fut)`, success or failure with `failed(fut)`.

Important cross-cutting conventions of the module:

- The dispatcher's implementation is platform-specific: on Windows it's IOCP (`CreateIoCompletionPort`), on Linux/BSD/macOS it's `select`/`epoll`/`kqueue` via `std/selectors`. The public API of the procedures (`recv`, `send`, `accept`, `connect`, `addRead`, `addWrite`, etc.) is the same on both branches, but their internal mechanics differ — this is called out in the "Implementation notes" subsection for each procedure.
- There are three main ways to handle a `Future`: `asyncCheck` (doesn't wait and doesn't return a value, only raises an exception on failure), `waitFor` (blocks the current thread until completion, called only from synchronous code), `await` (suspends the current `{.async.}` procedure without blocking the whole thread — used inside asynchronous procedures).
- `addRead`/`addWrite`/`addTimer`/`addProcess`/`addSignal` work in terms of callback functions (`Callback = proc (fd: AsyncFD): bool`), not `Future` — this is the low-level layer on top of which `recv`, `send`, `accept`, and other high-level procedures are built.

---

## Table of contents

1. [Types and basic concepts](#types-and-basic-concepts)
   1. [`AsyncFD`](#asyncfd)
   2. [`Callback`](#callback)
   3. [`AsyncEvent`](#asyncevent)
2. [The dispatcher: obtaining it and registering descriptors](#the-dispatcher-obtaining-it-and-registering-descriptors)
   1. [`getGlobalDispatcher`](#getglobaldispatcher)
   2. [`setGlobalDispatcher`](#setglobaldispatcher)
   3. [`hasPendingOperations`](#haspendingoperations)
   4. [`register`](#register)
   5. [`unregister`](#unregister)
   6. [`contains`](#contains)
3. [The event loop: polling and waiting](#the-event-loop-polling-and-waiting)
   1. [`poll`](#poll)
   2. [`drain`](#drain)
   3. [`runForever`](#runforever)
   4. [`waitFor`](#waitfor)
   5. [`callSoon`](#callsoon)
4. [Subscribing to file-descriptor events](#subscribing-to-file-descriptor-events)
   1. [`addRead`](#addread)
   2. [`addWrite`](#addwrite)
   3. [`addTimer`](#addtimer)
   4. [`addProcess`](#addprocess)
   5. [`addSignal`](#addsignal)
5. [`AsyncEvent` — user-defined events](#asyncevent--user-defined-events)
   1. [`newAsyncEvent`](#newasyncevent)
   2. [`trigger`](#trigger)
   3. [`close` (event)](#close-event)
   4. [`addEvent`](#addevent)
6. [Sockets: creation, accepting connections, exchanging data](#sockets-creation-accepting-connections-exchanging-data)
   1. [`createAsyncNativeSocket`](#createasyncnativesocket)
   2. [`closeSocket`](#closesocket)
   3. [`setInheritable`](#setinheritable)
   4. [`recv`](#recv)
   5. [`recvInto`](#recvinto)
   6. [`send`](#send)
   7. [`sendTo`](#sendto)
   8. [`recvFromInto`](#recvfrominto)
   9. [`acceptAddr`](#acceptaddr)
   10. [`accept`](#accept)
7. [Establishing outgoing connections](#establishing-outgoing-connections)
   1. [`dial`](#dial)
   2. [`connect`](#connect)
8. [Timing and waiting](#timing-and-waiting)
   1. [`sleepAsync`](#sleepasync)
   2. [`withTimeout`](#withtimeout)
9. [Streaming data](#streaming-data)
   1. [`readAll`](#readall)
10. [Resource diagnostics](#resource-diagnostics)
    1. [`activeDescriptors`](#activedescriptors)
    2. [`maxDescriptors`](#maxdescriptors)
11. [Practical recipes](#practical-recipes)
12. [Quick reference table](#quick-reference-table)
13. [Summary: which procedure to choose](#summary-which-procedure-to-choose)

---

## Types and basic concepts

### `AsyncFD`

```nim
type AsyncFD* = distinct int
```

**What it does.** A separate (`distinct`) wrapper type over an integer socket/file descriptor. `distinct int` guards against accidentally mixing an async descriptor with a plain `int` or `SocketHandle` — conversion is only possible through an explicit type cast.

**Implementation notes.** On Windows, an `AsyncFD` is registered in IOCP via `createIoCompletionPort`; on Unix-like systems, via a `selector` (`epoll`/`kqueue`/`select`, wrapped by `std/selectors`). The interface looks the same from the outside, which is why code written on top of `AsyncFD` is portable across platforms.

- **Parameters:** none — this is a type, not a procedure.

```nim
var fd: AsyncFD = createAsyncNativeSocket()
assert int(fd) >= 0  # the descriptor is a valid non-negative int
```

---

### `Callback`

```nim
type Callback* = proc (fd: AsyncFD): bool {.closure, gcsafe.}
```

**What it does.** The type of the callback function used by the low-level procedures `addRead`/`addWrite`/`addTimer`/`addProcess`/`addSignal`/`addEvent`. The return value controls the subscription: `true` means "unsubscribe this callback from observation", `false` means "keep the callback, call it again the next time the event fires".

**Implementation notes.** This convention (`bool` meaning "unsubscribe/keep") is used uniformly across all dispatchers: on the Unix branch the callback is stored in the `readList`/`writeList` inside the selector's per-descriptor data and removed from the list if it returns `true`; on the Windows branch, the equivalent role is played by re-registering `RegisterWaitForSingleObject`.

- **Parameters:**
  - `fd: AsyncFD` — the descriptor on which the event fired.

```nim
var callsLeft = 3
proc countdownCb(fd: AsyncFD): bool =
  dec(callsLeft)
  result = callsLeft <= 0  # true — stop observing after the third call
```

---

### `AsyncEvent`

```nim
when defined(windows):
  type AsyncEvent* = ptr AsyncEventImpl
else:
  type AsyncEvent* = distinct SelectEvent
```

**What it does.** A user-defined, manually triggerable event ("home-made" signal) not tied to a specific socket or timer. It is created once (`newAsyncEvent`), then subscribed to (`addEvent`), and triggered manually from anywhere in the code (`trigger`) — a typical scenario is a shutdown signal for the dispatcher coming from another thread.

**Implementation notes.** On Windows this is a named kernel object (`CreateEvent`) on which `RegisterWaitForSingleObject` is hung; on Unix it's a wrapper over `SelectEvent` from `std/selectors`, registered in the selector as just another observed descriptor. Both variants provide the same observable semantics: `trigger` wakes the dispatcher even if it is "sleeping" inside `poll`.

- **Parameters:** none — this is a type.

```nim
let stopEvent = newAsyncEvent()
proc onStop(fd: AsyncFD): bool =
  echo "shutdown signal received"
  result = true
addEvent(stopEvent, onStop)
trigger(stopEvent)
```

---

## The dispatcher: obtaining it and registering descriptors

### `getGlobalDispatcher`

```nim
proc getGlobalDispatcher*(): PDispatcher
```

**What it does.** Returns the dispatcher of the current thread, creating one on first access. The dispatcher is stored in a thread-local variable, so each thread has its own independent dispatcher and its own set of registered descriptors, timers, and pending callbacks.

**Implementation notes.** Lazy initialization via an `isNil` check — the "create on first use" idiom. A side effect of the internal call to `setGlobalDispatcher` is registering the `callSoon` function as the deferred-call handler for the `std/asyncfutures` module, so that `Future` callbacks are scheduled by the same dispatcher.

- **Parameters:** none.

```nim
let disp1 = getGlobalDispatcher()
let disp2 = getGlobalDispatcher()
assert disp1 == disp2  # a repeated call returns the same thread dispatcher
```

---

### `setGlobalDispatcher`

```nim
proc setGlobalDispatcher*(disp: PDispatcher)
```

**What it does.** Forcibly replaces the current thread's dispatcher with the one passed in. Rarely needed — for example, when writing a custom thread pool where each worker must explicitly create and install its own dispatcher before working with sockets.

**Implementation notes.** On the Windows branch, before the replacement there's an `assert callbacks.len == 0` check — the old dispatcher is assumed to be empty at the moment of replacement, otherwise pending calls would be lost. This is an explicit warning: replacing the dispatcher "on the fly" while it still has unfinished operations is a source of hard-to-track leaks and lost callbacks.

- **Parameters:**
  - `disp: PDispatcher` — the new dispatcher, usually created via `newDispatcher()`.

```nim
import std/asyncdispatch

let customDisp = newDispatcher()
setGlobalDispatcher(customDisp)
assert getGlobalDispatcher() == customDisp
```

---

### `hasPendingOperations`

```nim
proc hasPendingOperations*(): bool
```

**What it does.** Reports whether the global dispatcher has any unfinished work: registered descriptors, pending timers, or deferred callbacks. Used as the stopping condition for `poll`/`drain` loops — if there's no more work, polling the dispatcher further makes no sense (and calling `poll` on an empty dispatcher raises an exception).

**Implementation notes.** A simple check of three counters (`handles`/`timers`/`callbacks` on Windows, `selector`/`timers`/`callbacks` on Unix) with no side effects — O(1) on both platforms.

- **Parameters:** none.

```nim
assert hasPendingOperations() == false  # a fresh dispatcher is empty
discard sleepAsync(10)
assert hasPendingOperations() == true   # the timer added an unfinished operation
```

---

### `register`

```nim
proc register*(fd: AsyncFD)
```

**What it does.** Registers an already-created (usually native, "raw") descriptor with the current thread's dispatcher, making it usable by `addRead`/`addWrite`/`recv`/`send` and the other procedures of the module. Without registration the dispatcher knows nothing about the descriptor and cannot deliver events for it.

**Implementation notes.** On Windows, registration means binding the descriptor to the I/O completion port (`createIoCompletionPort`) and adding it to the `handles` set; on Unix, it means adding an entry to the `selector` with empty read/write callback lists. `createAsyncNativeSocket` calls `register` automatically, so an explicit call is mainly needed for descriptors obtained from elsewhere (e.g. accepted manually via `accept4`).

- **Parameters:**
  - `fd: AsyncFD` — the descriptor to register.

```nim
import std/nativesockets

let rawSocket = createNativeSocket(Domain.AF_INET, SockType.SOCK_STREAM, Protocol.IPPROTO_TCP)
setBlocking(rawSocket, false)
let fd = AsyncFD(rawSocket)
register(fd)
assert contains(getGlobalDispatcher(), fd)
```

---

### `unregister`

```nim
proc unregister*(fd: AsyncFD)
proc unregister*(ev: AsyncEvent)
```

**What it does.** Removes a descriptor (or a user event) from the dispatcher's observation. After the call, the dispatcher will no longer deliver any events for that `fd`/`ev`; the socket itself is not closed — `closeSocket` is the procedure that closes it while also unsubscribing it.

**Implementation notes.** On the Unix branch this is a direct call to the selector's `unregister`; on Windows there's no equivalent public procedure for `fd` in the shared part (removal happens inside `closeSocket`/on callback completion), reflecting a difference between the models: IOCP binds a descriptor to the port in an almost irreversible way, whereas `selectors` allows records to be added and removed cheaply.

- **Parameters:**
  - `fd: AsyncFD` **or** `ev: AsyncEvent` — the thing being unsubscribed.

```nim
let fd = createAsyncNativeSocket()
assert contains(getGlobalDispatcher(), fd)
unregister(fd)
assert not contains(getGlobalDispatcher(), fd)
```

---

### `contains`

```nim
proc contains*(disp: PDispatcher, fd: AsyncFD): bool
```

**What it does.** Checks whether descriptor `fd` is registered with dispatcher `disp`. Used as the `in` operator: `fd in disp`.

**Implementation notes.** On Unix — a check `fd.SocketHandle in disp.selector`, i.e. delegating to the selector's own `contains`; on Windows — `fd in disp.handles` (a `HashSet` lookup). Both are amortized O(1).

- **Parameters:**
  - `disp: PDispatcher` — the dispatcher to search in.
  - `fd: AsyncFD` — the sought descriptor.

```nim
let disp = getGlobalDispatcher()
let fd = createAsyncNativeSocket()
assert contains(disp, fd)
assert fd in disp  # equivalent infix form of the same call
```

---

## The event loop: polling and waiting

### `poll`

```nim
proc poll*(timeout = 500)
```

**What it does.** Polls the dispatcher once: waits for at least one event (a timer expiring, a socket becoming ready, a signal firing) for no longer than `timeout` milliseconds, and processes every event that has accumulated by that point. If the dispatcher has no registered operations at all, it raises `ValueError`.

**Implementation notes.** The whole body is a single line — `discard runOnce(timeout)`. `runOnce` calls the underlying system wait primitive (`epoll_wait`/`kqueue`/`GetQueuedCompletionStatus`) exactly once, so `poll` is "one tick" of the event loop, not the loop itself; a continuous loop is built by the calling code (see `runForever`, `waitFor`, `drain`).

- **Parameters:**
  - `timeout: int` — the maximum wait time in milliseconds (500 by default).

```nim
var fut = sleepAsync(10)
while not finished(fut):
  poll()          # each call is one tick of the loop, until the timer fires
assert finished(fut)
```

---

### `drain`

```nim
proc drain*(timeout = 500)
```

**What it does.** Unlike `poll`, which performs exactly one system poll, `drain` calls `runOnce` in a loop for as long as there is unfinished work (`hasPendingOperations`), while spending no more than `timeout` milliseconds in total. Useful when you want to "drain" all work currently available on the dispatcher with a single call instead of hand-rolling a `while` loop.

**Implementation notes.** Keeps track of elapsed time via `getMonoTime()` (monotonic timestamps, unaffected by system-clock changes) and, on each iteration, decreases the remaining `timeout - elapsed` passed into `runOnce`. The loop stops on either of two conditions: no more operations are pending, or the time budget has run out — i.e. `drain` does not guarantee that all operations will finish, only that it will "spend" the allotted time trying.

- **Parameters:**
  - `timeout: int` — the total time budget in milliseconds for processing all available events.

```nim
discard sleepAsync(5)
discard sleepAsync(5)
drain(100)                       # processes both timers in a single call
assert not hasPendingOperations()
```

---

### `runForever`

```nim
proc runForever*()
```

**What it does.** Runs an infinite `poll()` loop. Used as the "entry point" of a server that should never return on its own — the only way out is an exception or process termination.

**Implementation notes.** The body is literally `while true: poll()`. There is no extra logic: protection against a busy loop comes entirely from the fact that `poll` blocks internally in the system wait call until an event occurs or the timeout elapses.

- **Parameters:** none.

```nim
# The example is illustrative: runForever() never returns control,
# so in real code it is the last line of a program,
# called after all handlers have been registered.
proc startServer() =
  discard accept(serverSocket)  # register the wait for incoming connections
  runForever()
```

---

### `waitFor`

```nim
proc waitFor*[T](fut: Future[T]): T
```

**What it does.** Blocks the current thread, spinning `poll()` in a loop, until the given `Future` finishes, then returns its value (or raises an exception if the `Future` finished with an error). This is the "entry point" into async code from synchronous code — the typical call site is a `main` procedure, tests, or a CLI tool.

**Implementation notes.** The body is `while not finished(fut): poll()`, followed by `read(fut)`. An important restriction: `waitFor` must not be called inside an `{.async.}` procedure — there it would block the single thread in which the other asynchronous tasks are supposed to run, leading to a deadlock if the awaited `Future` depends on the progress of other asynchronous tasks.

- **Parameters:**
  - `fut: Future[T]` — the awaited `Future` of any type `T` (including `void`).

```nim
proc delayedAnswer(): owned(Future[int]) {.async.} =
  await sleepAsync(5)
  result = 42

let answer = waitFor(delayedAnswer())
assert answer == 42
```

---

### `callSoon`

```nim
proc callSoon*(cbproc: proc () {.gcsafe.})
```

**What it does.** Schedules `cbproc` to be called "as soon as possible" — as soon as control returns to the dispatcher again (i.e. on the next `poll` tick), but not immediately or synchronously at the call site. Useful for deferring code until the current call stack has unwound, avoiding deep recursion, or reusing the dispatcher as a "microtask" scheduler.

**Implementation notes.** Adds `cbproc` to the dispatcher's `callbacks` queue (a `Deque`); the queue is drained in `processPendingCallbacks` on every iteration of `runOnce`, and crucially **before** the system-poll timeout is computed (`adjustTimeout` immediately returns 0 if the queue isn't empty) — this guarantees that deferred calls are not "delayed" by waiting on network events.

- **Parameters:**
  - `cbproc: proc ()` — a procedure with no parameters and no return value, called asynchronously later.

```nim
var order: seq[int] = @[]
add(order, 1)
callSoon(proc () = add(order, 3))
add(order, 2)
poll()
assert order == @[1, 2, 3]  # the deferred call ran after the current code
```

---

## Subscribing to file-descriptor events

### `addRead`

```nim
proc addRead*(fd: AsyncFD, cb: Callback)
```

**What it does.** Subscribes callback `cb` to descriptor `fd` becoming ready for reading. This is a low-level building block: high-level procedures such as `recv` call `addRead` internally, wrapping the callback around completing a `Future`.

**Implementation notes.** On Unix, the callback is added to the `readList` of the selector's entry corresponding to `fd`, after which the descriptor's watched-event mask is updated (`updateHandle`). On Windows there is no single notion of "readiness for reading" in IOCP terms (there, events are already tied to a specific I/O operation), so the Windows implementation of `addRead` is a "hack" via `WSAEventSelect`, intended mainly for porting Unix libraries — the source explicitly warns about this: it conflicts with this module's own `recv`/`accept` on Windows if used on the same socket at the same time.

- **Parameters:**
  - `fd: AsyncFD` — the observed descriptor.
  - `cb: Callback` — called when readable; return `true` to stop observing.

```nim
let fd = createAsyncNativeSocket()
var triggered = false
proc onReadable(sock: AsyncFD): bool =
  triggered = true
  result = true
addRead(fd, onReadable)
```

---

### `addWrite`

```nim
proc addWrite*(fd: AsyncFD, cb: Callback)
```

**What it does.** The write-side counterpart of `addRead`, subscribing to write readiness. Used, for example, inside `send`/`connect` (Unix branch) to wait for the moment the socket is ready to accept the next chunk of data, or for a non-blocking `connect` to complete.

**Implementation notes.** On Unix — the same scheme with the `writeList` and the selector's event-mask update; on Windows — the same kind of `WSAEventSelect`-based "hack" as `addRead`, with the mask `FD_WRITE or FD_CONNECT or FD_CLOSE`.

- **Parameters:**
  - `fd: AsyncFD` — the observed descriptor.
  - `cb: Callback` — called when writable; return `true` to stop observing.

```nim
let fd = createAsyncNativeSocket()
proc onWritable(sock: AsyncFD): bool =
  result = true  # a one-shot readiness check, not observed further
addWrite(fd, onWritable)
```

---

### `addTimer`

```nim
proc addTimer*(timeout: int, oneshot: bool, cb: Callback)
```

**What it does.** Registers a timer that calls `cb` after `timeout` milliseconds. If `oneshot == true`, the event fires once; if `false`, it repeats periodically (every `timeout` milliseconds) until `cb` returns `true`.

**Implementation notes.** On Unix, where the selector's platform timers are available (`ioselSupportedPlatform`, i.e. Linux/BSD/macOS/Solaris), the timer is registered directly via `selector.registerTimer`. On Windows it's a `CreateEvent` plus `RegisterWaitForSingleObject` with the `WT_EXECUTEONLYONCE` flag set when `oneshot == true`. The dispatcher's shared, simpler timer mechanism (`processTimers`, based on a `HeapQueue`, used for example by `sleepAsync`) is a separate, simpler path that works through `Future`; `addTimer` is the low-level callback version for cases where a `Future` would be overkill.

- **Parameters:**
  - `timeout: int` — the interval in milliseconds, must be greater than 0.
  - `oneshot: bool` — `true` for a single firing, `false` for repeated firings.
  - `cb: Callback` — called when the timer expires.

```nim
var ticks = 0
proc onTick(fd: AsyncFD): bool =
  inc(ticks)
  result = ticks >= 3    # unsubscribe the timer after the third tick
addTimer(5, oneshot = false, cb = onTick)
drain(200)
assert ticks == 3
```

---

### `addProcess`

```nim
proc addProcess*(pid: int, cb: Callback)
```

**What it does.** Registers a callback that fires when the process with ID `pid` exits. Lets you asynchronously await a child process's completion without blocking the thread.

**Implementation notes.** On Windows — opening a process handle (`openProcess` with the `SYNCHRONIZE` flag) and `RegisterWaitForSingleObject` on that handle with the `WT_EXECUTEONLYONCE` flag (a process is either alive or dead — there are no repeated firings). On Unix, where `ioselSupportedPlatform` is available, registration goes through `selector.registerProcess`, relying on a platform-specific tracking mechanism (e.g. `pidfd` on modern Linux kernels, or `kqueue`'s `EVFILT_PROC` events on BSD/macOS).

- **Parameters:**
  - `pid: int` — the ID of the process to watch.
  - `cb: Callback` — called when the process exits.

```nim
import std/osproc

let p = startProcess("sleep", args = @["0"])
proc onExit(fd: AsyncFD): bool =
  echo "process has exited"
  result = true
addProcess(processID(p), onExit)
```

---

### `addSignal`

```nim
proc addSignal*(signal: int, cb: Callback)
```

**What it does.** Subscribes a callback to a POSIX signal (e.g. `SIGTERM`, `SIGUSR1`) — an asynchronous alternative to installing a classic signal handler. Available only on platforms with signal support in `std/selectors` (`ioselSupportedPlatform`); Windows has no such procedure at all, since Windows has no POSIX signals in that sense.

**Implementation notes.** Delegates registration to `selector.registerSignal`, which on Linux uses `signalfd`, and on BSD/macOS the `kqueue` filter `EVFILT_SIGNAL`: the signal is turned into an ordinary readiness event, handled by the same `runOnce` loop as sockets and timers.

- **Parameters:**
  - `signal: int` — the signal number (see `std/posix`, e.g. `SIGTERM`).
  - `cb: Callback` — called when the signal is received.

```nim
when defined(posix):
  import std/posix
  proc onTerm(fd: AsyncFD): bool =
    echo "SIGTERM received, shutting down"
    result = true
  addSignal(SIGTERM, onTerm)
```

---

## `AsyncEvent` — user-defined events

### `newAsyncEvent`

```nim
proc newAsyncEvent*(): AsyncEvent
```

**What it does.** Creates a new user-event object that isn't registered to anything yet. The event itself isn't observed by the dispatcher until you explicitly subscribe to it via `addEvent`.

**Implementation notes.** On Unix — a thin wrapper over `newSelectEvent()` from `std/selectors` (typically an `eventfd` on Linux, or an equivalent on BSD/macOS); on Windows — a named, manual-reset kernel object (`CreateEvent`), wrapped in the `AsyncEventImpl` structure.

- **Parameters:** none.

```nim
let ev = newAsyncEvent()
```

---

### `trigger`

```nim
proc trigger*(ev: AsyncEvent)
```

**What it does.** Puts event `ev` into the signaled state, waking the dispatcher (if it's currently waiting inside `poll`) and calling all callbacks subscribed to `ev` on the next loop tick.

**Implementation notes.** Thread-safe — the event can be triggered from another thread, which makes `AsyncEvent` the primary mechanism for cross-thread communication with a single-threaded dispatcher (for example, a "time to stop" signal from a thread handling Ctrl+C).

- **Parameters:**
  - `ev: AsyncEvent` — the event being triggered.

```nim
let ev = newAsyncEvent()
var fired = false
proc onFire(fd: AsyncFD): bool =
  fired = true
  result = true
addEvent(ev, onFire)
trigger(ev)
drain(100)
assert fired
```

---

### `close` (event)

```nim
proc close*(ev: AsyncEvent)
```

**What it does.** Releases the system resources associated with `ev` (closes the event's file descriptor on Unix, the kernel handle on Windows). The event must not be used after this call.

**Implementation notes.** Symmetric to `close` for sockets/files: an event is also a kernel-level resource that must be explicitly closed, or it will be held open until the process ends.

- **Parameters:**
  - `ev: AsyncEvent` — the event being closed.

```nim
let ev = newAsyncEvent()
close(ev)
```

---

### `addEvent`

```nim
proc addEvent*(ev: AsyncEvent, cb: Callback)
```

**What it does.** Subscribes `cb` to the firing of the user event `ev` — from this point on, the dispatcher calls `cb` every time `ev` is triggered via `trigger`.

**Implementation notes.** Registers `ev` as just another entry in the same selector/completion port as ordinary sockets — from the point of view of the `runOnce` loop, a user event is indistinguishable from a socket becoming ready to read, which is exactly what lets a single polling loop serve sockets, timers, signals, and manual events all at once.

- **Parameters:**
  - `ev: AsyncEvent` — the observed event.
  - `cb: Callback` — called when the event is triggered.

```nim
let shutdown = newAsyncEvent()
proc onShutdown(fd: AsyncFD): bool =
  echo "stopping the dispatcher"
  result = true
addEvent(shutdown, onShutdown)
```

---

## Sockets: creation, accepting connections, exchanging data

### `createAsyncNativeSocket`

```nim
proc createAsyncNativeSocket*(domain: cint, sockType: cint, protocol: cint,
                              inheritable = defined(nimInheritHandles)): AsyncFD
proc createAsyncNativeSocket*(domain: Domain = Domain.AF_INET,
                              sockType: SockType = SOCK_STREAM,
                              protocol: Protocol = IPPROTO_TCP,
                              inheritable = defined(nimInheritHandles)): AsyncFD
```

**What it does.** Creates a new native socket, switches it to non-blocking mode, and registers it with the current thread's dispatcher — i.e. replaces the manual sequence of "create a socket + `setBlocking(false)` + `register`". There are two overloads: a low-level one (raw `cint` constants) and a high-level one (typed enums `Domain`/`SockType`/`Protocol` from `std/nativesockets`).

**Implementation notes.** On macOS, the `SO_NOSIGPIPE` socket option is additionally set so that a write attempt on a closed connection returns an error instead of delivering a `SIGPIPE` signal to the process — a detail specific to the BSD-family sockets on Darwin. On a creation failure (`createNativeSocket` returned `osInvalidSocket`), the procedure silently returns `osInvalidSocket.AsyncFD` without raising — the calling code should check the result.

- **Parameters:**
  - `domain: Domain` — the address family (`AF_INET`, `AF_INET6`, etc.).
  - `sockType: SockType` — the socket type (`SOCK_STREAM` for TCP, `SOCK_DGRAM` for UDP).
  - `protocol: Protocol` — the transport protocol (`IPPROTO_TCP`, `IPPROTO_UDP`).
  - `inheritable: bool` — whether the descriptor can be inherited by child processes (not inherited by default).

```nim
let tcpSocket = createAsyncNativeSocket(Domain.AF_INET, SockType.SOCK_STREAM, Protocol.IPPROTO_TCP)
assert contains(getGlobalDispatcher(), tcpSocket)  # the socket is already registered with the dispatcher
```

---

### `closeSocket`

```nim
proc closeSocket*(socket: AsyncFD)
```

**What it does.** Closes the socket while simultaneously unregistering it from the dispatcher, unblocking any pending read/write callbacks on it (they are notified of the closure, so they can properly fail the associated `Future` instead of "hanging" forever).

**Implementation notes.** On Unix the order matters: first the `readList`/`writeList` are captured, then the socket is removed from the selector and closed at the OS level, and only afterward are the saved callbacks invoked — if even one of them doesn't return `true` (i.e. refuses to finish), this is treated as a usage error and raises an exception, since operations on an already-closed descriptor cannot continue.

- **Parameters:**
  - `socket: AsyncFD` — the socket to close.

```nim
let s = createAsyncNativeSocket()
closeSocket(s)
assert not contains(getGlobalDispatcher(), s)
```

---

### `setInheritable`

```nim
proc setInheritable*(fd: AsyncFD, inheritable: bool): bool
```

**What it does.** Controls whether a file descriptor will be inherited by child processes (analogous to the `FD_CLOEXEC`/`O_NOINHERIT` flag). Returns `true` on success. Not available on every platform — check `declared(setInheritable)` before use.

**Implementation notes.** A thin wrapper over the platform-specific `setInheritable` for `FileHandle`, hooked up via the `{.dirty.}` template `implementSetInheritable`, which only pulls in the implementation where the corresponding system procedure is actually declared (`when declared(setInheritable)`) — a technique that avoids duplicating the procedure by hand on every platform branch.

- **Parameters:**
  - `fd: AsyncFD` — the descriptor to modify.
  - `inheritable: bool` — `true` to allow inheritance.

```nim
when declared(setInheritable):
  let s = createAsyncNativeSocket()
  assert setInheritable(s, false)
```

---

### `recv`

```nim
proc recv*(socket: AsyncFD, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[string])
```

**What it does.** Reads **up to** `size` bytes from `socket`. The returned `Future` completes once at least some data has been read, or with an empty string `""` if the other side disconnected — the latter is **not** an error (the `Future` doesn't fail), but a normal end-of-stream signal that must be checked explicitly by comparing to `""`.

**Implementation notes.** On Unix the operation is implemented by registering a read-readiness callback (`addRead`): when it fires, a non-blocking system `recv` is performed, and the error codes `EINTR`/`EWOULDBLOCK`/`EAGAIN` are treated as "no data yet, keep waiting" (`result = false` in the callback — observation continues), unlike other errors, which fail the `Future`. On Windows the same operation is built around `WSARecv` and the `OVERLAPPED` structure, with the result arriving already-completed through the completion port — a "request it and wait for the finished result" model rather than "wait for readiness and read it yourself".

- **Parameters:**
  - `socket: AsyncFD` — the socket to read from (already registered).
  - `size: int` — the upper bound on the number of bytes to read.
  - `flags: set[SocketFlag]` — behavior flags, `SafeDisconn` by default (don't treat typical disconnection errors as fatal).

```nim
proc echoOnce(socket: AsyncFD) {.async.} =
  let data = await recv(socket, 1024)
  if data == "":
    echo "connection closed by the remote side"
  else:
    echo "received: ", data
```

---

### `recvInto`

```nim
proc recvInto*(socket: AsyncFD, buf: pointer, size: int,
               flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

**What it does.** The same as `recv`, but writes data directly into an already-allocated buffer `buf` (without allocating a new string) and returns not a string but the number of bytes actually read. A disconnection is signaled by the value `0`. Used where extra allocations matter — for example, in a loop that reuses the same buffer.

**Implementation notes.** Differs from `recv` only in where the result of the system call goes: instead of an intermediate `readBuffer` string, the data is copied straight into the caller-supplied `buf` — saving one memory copy compared to `recv`.

- **Parameters:**
  - `socket: AsyncFD` — the socket to read from.
  - `buf: pointer` — the destination buffer, must hold at least `size` bytes and remain valid (not collected by the GC) until the `Future` completes.
  - `size: int` — the maximum number of bytes to read.
  - `flags: set[SocketFlag]` — same as `recv`.

```nim
proc readFixed(socket: AsyncFD): Future[int] {.async.} =
  var buf = newString(64)
  result = await recvInto(socket, addr buf[0], 64)
```

---

### `send`

```nim
proc send*(socket: AsyncFD, buf: pointer, size: int,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])
proc send*(socket: AsyncFD, data: string,
           flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

**What it does.** Sends data to `socket`; the returned `Future` completes once **all** of the supplied data has been guaranteed to be sent. There's a low-level variant with a raw pointer and size (platform-dependent implementation), and a high-level variant with a `string` — shared across both platforms, built on top of the pointer variant.

**Implementation notes.** On Unix, the write loop explicitly tracks progress (`written`) and keeps calling the non-blocking system `send` until all `size` bytes have been sent — a partial write does not count as completion; the callback returns `false` and continues observing via `addWrite`. The string variant additionally solves a lifetime problem: the string `data` passed by the caller could be garbage-collected before the asynchronous write finishes, so a helper function `keepAlive` is used, which "asserts" the parameter as an escaping variable, forcing the compiler to place it into the callback's closure and keep it alive until the send completes.

- **Parameters:**
  - `socket: AsyncFD` — the socket to write to.
  - `buf: pointer` / `data: string` — the data being sent.
  - `size: int` — the size of the data in bytes (pointer variant only).
  - `flags: set[SocketFlag]` — behavior flags.

```nim
proc greet(socket: AsyncFD) {.async.} =
  await send(socket, "hello\n")
```

---

### `sendTo`

```nim
proc sendTo*(socket: AsyncFD, data: pointer, size: int, saddr: ptr SockAddr,
             saddrLen: SockLen, flags = {SocketFlag.SafeDisconn}): owned(Future[void])
```

**What it does.** Sends a datagram to the specified address `saddr` without a prior `connect` — the procedure for UDP sockets (`SOCK_DGRAM`), where every packet is addressed individually.

**Implementation notes.** The destination address is copied into its own fixed-size stack buffer (128 bytes — the size of `SOCKADDR_STORAGE`, large enough for both IPv4 and IPv6 addresses), so as not to depend on the lifetime of the `saddr` pointer passed by the caller: by the time the send actually happens (which on Unix happens asynchronously, when the socket becomes writable), the original `saddr` may already be invalid.

- **Parameters:**
  - `socket: AsyncFD` — the UDP socket.
  - `data: pointer`, `size: int` — the data to send and its size.
  - `saddr: ptr SockAddr`, `saddrLen: SockLen` — the recipient's address and its structure size.
  - `flags: set[SocketFlag]` — behavior flags.

```nim
proc sendDatagram(socket: AsyncFD, data: string, address: ptr SockAddr, addrLen: SockLen) {.async.} =
  await sendTo(socket, unsafeAddr data[0], len(data), address, addrLen)
```

---

### `recvFromInto`

```nim
proc recvFromInto*(socket: AsyncFD, data: pointer, size: int,
                   saddr: ptr SockAddr, saddrLen: ptr SockLen,
                   flags = {SocketFlag.SafeDisconn}): owned(Future[int])
```

**What it does.** Receives a single datagram into buffer `data`, while also writing the sender's address into `saddr`/`saddrLen`. The returned `Future` completes with the size of the received packet, in bytes.

**Implementation notes.** The counterpart to `sendTo` in the same UDP-exchange model; unlike `recv`/`recvInto`, it returns not only the data but also the source address — necessary information for a server serving many clients through a single UDP socket, where every incoming packet may come from a different peer.

- **Parameters:**
  - `socket: AsyncFD` — the UDP socket.
  - `data: pointer`, `size: int` — the receive buffer and its size.
  - `saddr: ptr SockAddr`, `saddrLen: ptr SockLen` — where to write the sender's address.
  - `flags: set[SocketFlag]` — behavior flags.

```nim
proc receiveDatagram(socket: AsyncFD): Future[int] {.async.} =
  var buf = newString(512)
  var address: Sockaddr_storage
  var addrLen = SockLen(sizeof(address))
  result = await recvFromInto(socket, addr buf[0], 512, cast[ptr SockAddr](addr address), addr addrLen)
```

---

### `acceptAddr`

```nim
proc acceptAddr*(socket: AsyncFD, flags = {SocketFlag.SafeDisconn},
                 inheritable = defined(nimInheritHandles)):
    owned(Future[tuple[address: string, client: AsyncFD]])
```

**What it does.** Accepts a single incoming connection on the listening `socket` and returns both the accepted client socket and the connecting peer's textual address — unlike `accept`, which returns only the socket itself.

**Implementation notes.** On Unix, `accept4` is used where it's declared (allowing `SOCK_CLOEXEC` to be set atomically, avoiding a race between `accept` and a subsequent call to set the "don't inherit" flag), falling back to plain `accept` plus a separate `setInheritable` call where `accept4` isn't available. The accepted client descriptor is automatically registered with the dispatcher (`register(client.AsyncFD)`) — the calling code doesn't need to do this itself.

- **Parameters:**
  - `socket: AsyncFD` — the listening socket.
  - `flags: set[SocketFlag]` — behavior flags.
  - `inheritable: bool` — whether the client descriptor is inherited by child processes.

```nim
proc acceptOne(server: AsyncFD) {.async.} =
  let (address, client) = await acceptAddr(server)
  echo "connection from ", address
```

---

### `accept`

```nim
proc accept*(socket: AsyncFD, flags = {SocketFlag.SafeDisconn},
             inheritable = defined(nimInheritHandles)): owned(Future[AsyncFD])
```

**What it does.** The same as `acceptAddr`, but returns only the client socket, discarding the address — a convenient shorthand for when the connecting peer's address isn't needed.

**Implementation notes.** Implemented on top of `acceptAddr`: it creates its own `Future[AsyncFD]`, attaches a callback to the `Future` returned by `acceptAddr` that, on success, extracts the `client` field from the tuple and completes the outer `Future` with it, and on failure propagates the error further. A typical example of a "thin wrapper" narrowing a more general procedure's interface.

- **Parameters:**
  - `socket: AsyncFD` — the listening socket.
  - `flags: set[SocketFlag]` — behavior flags.
  - `inheritable: bool` — whether the client descriptor is inherited.

```nim
proc acceptLoop(server: AsyncFD) {.async.} =
  while true:
    let client = await accept(server)
    echo "new connection: ", int(client)
```

---

## Establishing outgoing connections

### `dial`

```nim
proc dial*(address: string, port: Port,
           protocol: Protocol = IPPROTO_TCP): owned(Future[AsyncFD])
```

**What it does.** Establishes a connection to `address:port`, iterating on its own through every address DNS/`getaddrinfo` returned for that name (handling both IPv4 and IPv6), and returns an already-ready, registered socket. The preferred high-level way to connect to a remote host when it doesn't matter which specific IP address ends up being used.

**Implementation notes.** The address iteration is implemented in the `asyncAddrInfoLoop` template, shared between `dial` and `connect`: it walks the `AddrInfo` linked list, creating a socket of the appropriate domain for each address and trying `doConnect`; on failure with one address the socket is closed and the next address is tried, on success the attempts stop and the resulting `Future` completes with the socket found. This strategy (a simplified, sequential form of "happy eyeballs") spares the calling code from having to handle multiple DNS records manually.

- **Parameters:**
  - `address: string` — the hostname or IP address.
  - `port: Port` — the TCP/UDP port.
  - `protocol: Protocol` — the transport-layer protocol, `IPPROTO_TCP` by default.

```nim
proc fetchOnce() {.async.} =
  let socket = await dial("example.org", Port(80))
  await send(socket, "GET / HTTP/1.0\r\n\r\n")
```

---

### `connect`

```nim
proc connect*(socket: AsyncFD, address: string, port: Port,
              domain = Domain.AF_INET): owned(Future[void])
```

**What it does.** Establishes a connection on an already-created socket `socket` to a specific `domain` (unlike `dial`, which picks the domain and creates the socket itself). Useful when a socket already exists ahead of time (for example, with special options) and only needs to be connected.

**Implementation notes.** On Windows the socket must be pre-bound (`bindToDomain`) before connecting — a requirement of `ConnectEx`, which the plain `connect(2)` on Unix does not have; after that, the same address iteration via `asyncAddrInfoLoop` is used as in `dial`, but without creating a new socket (`shouldCreateFd = false`), since the socket has already been supplied by the calling code.

- **Parameters:**
  - `socket: AsyncFD` — a pre-created socket.
  - `address: string`, `port: Port` — the destination address and port.
  - `domain: Domain` — the address family, must match the domain the `socket` was created with.

```nim
proc connectManual() {.async.} =
  let socket = createAsyncNativeSocket()
  await connect(socket, "example.org", Port(80))
```

---

## Timing and waiting

### `sleepAsync`

```nim
proc sleepAsync*(ms: int | float): owned(Future[void])
```

**What it does.** Returns a `Future` that completes after `ms` milliseconds — an asynchronous analog of `sleep` that doesn't block the whole thread: while the timer is ticking, the dispatcher keeps serving other operations.

**Implementation notes.** Registers a pair `(firing time, Future)` in the dispatcher's priority queue (`HeapQueue`), ordered by firing time — the same timer mechanism processed by `processTimers` on every iteration of `runOnce`, only expressed through a `Future` rather than a low-level `Callback` (unlike `addTimer`).

- **Parameters:**
  - `ms: int | float` — the delay in milliseconds; supports fractional values via `float` for sub-millisecond precision (converted to nanoseconds).

```nim
let start = epochTime()
waitFor sleepAsync(20)
assert epochTime() - start >= 0.02 - 0.005  # at least ~20 ms have passed (with margin for error)
```

---

### `withTimeout`

```nim
proc withTimeout*[T](fut: Future[T], timeout: int): owned(Future[bool])
```

**What it does.** Wraps an arbitrary `fut` with a timeout: the returned `Future[bool]` completes with `true` if `fut` finished before `timeout` milliseconds, and `false` if the time ran out first. In neither case is the original `fut` forcibly cancelled — it continues to exist (and may still complete on its own later); `withTimeout` simply stops referencing it after the timeout.

**Implementation notes.** Runs two `Future`s in parallel — the original `fut` and `sleepAsync(timeout)` — and attaches its own callback to each; whichever of the two finishes first "wins": it completes the shared `retFuture` with the corresponding value and explicitly clears the losing side's callbacks (`clearCallbacks`) so as not to keep unnecessary closures alive once the result is no longer needed.

- **Parameters:**
  - `fut: Future[T]` — the operation being awaited.
  - `timeout: int` — the maximum wait time in milliseconds.

```nim
proc slowOp(): owned(Future[int]) {.async.} =
  await sleepAsync(50)
  result = 1

let completed = waitFor withTimeout(slowOp(), 10)
assert completed == false  # the operation did not finish within 10 ms
```

---

## Streaming data

### `readAll`

```nim
proc readAll*(future: FutureStream[string]): owned(Future[string]) {.async.}
```

**What it does.** Sequentially reads every value coming from a streaming `FutureStream[string]` as it arrives and concatenates it into a single string; completes once the stream signals the end of data.

**Implementation notes.** Written as an ordinary `{.async.}` procedure rather than through low-level callbacks: the loop calls `await read(future)`, which returns a `(hasValue, value)` pair — a pattern characteristic of `FutureStream` from `std/asyncstreams` (unlike `Future[T]`, which completes exactly once, a `FutureStream` can yield values repeatedly, so reading is expressed as an explicit loop with a stop condition on `hasValue == false`).

- **Parameters:**
  - `future: FutureStream[string]` — the source of streaming string data.

```nim
proc demo() {.async.} =
  var stream = newFutureStream[string]("demo")
  await write(stream, "hello, ")
  await write(stream, "world")
  complete(stream)
  let whole = await readAll(stream)
  assert whole == "hello, world"
```

---

## Resource diagnostics

### `activeDescriptors`

```nim
proc activeDescriptors*(): int {.inline.}
```

**What it does.** Returns the current number of active (dispatcher-registered) file descriptors. A cheap operation requiring no system call — useful for monitoring descriptor leaks in long-running servers.

**Implementation notes.** On Windows — the size of the dispatcher's `handles` set; on Unix — the counter `selector.count`, maintained by the selector itself. Both cases simply read an already-existing internal counter, hence the `{.inline.}` pragma.

- **Parameters:** none.

```nim
let before = activeDescriptors()
let s = createAsyncNativeSocket()
assert activeDescriptors() == before + 1
```

---

### `maxDescriptors`

```nim
proc maxDescriptors*(): int {.raises: OSError.}
```

**What it does.** Returns the maximum number of file descriptors the current process is allowed to have open at once — a system-imposed limit, not current usage (unlike `activeDescriptors`). Useful for sizing connection pools ahead of time, before the limit is actually hit in practice.

**Implementation notes.** Unlike `activeDescriptors`, this procedure **performs a system call** on every invocation: on most Unix platforms, `getrlimit(RLIMIT_NOFILE, ...)`, subtracting one reserved descriptor; on Windows, a hardcoded constant `16_700_000` is returned (the documented practical IOCP limit on the number of handles), since there's no notion of a POSIX-style per-process descriptor limit there. Available only on the platforms explicitly listed in the compile-time condition (Linux, Windows, macOS, BSD, Solaris, and several embedded OSes).

- **Parameters:** none.

```nim
let limit = maxDescriptors()
assert limit > 0
```

---

## Practical recipes

### An echo server built on `acceptAddr` + `recv` + `send`

Each new connection is served by its own independent asynchronous loop; the server keeps accepting new connections without waiting for previous ones to finish.

```nim
proc handleClient(client: AsyncFD) {.async.} =
  while true:
    let data = await recv(client, 1024)
    if data == "":
      closeSocket(client)
      break
    await send(client, data)  # echo the data back as-is

proc serve(server: AsyncFD) {.async.} =
  while true:
    let (address, client) = await acceptAddr(server)
    asyncCheck handleClient(client)  # start handling without awaiting its completion
```

---

### A timeout on a network operation

Combining `withTimeout` with an explicit result check is a typical pattern for operations that must not "hang" forever — for example, reading from a slow or stalled client.

```nim
proc recvWithLimit(socket: AsyncFD, timeoutMs: int): owned(Future[string]) {.async.} =
  let dataFut = recv(socket, 1024)
  let completed = await withTimeout(dataFut, timeoutMs)
  if completed:
    result = read(dataFut)
  else:
    raise newException(TimeoutError, "the operation did not finish in time")
```

---

### A periodic background task via `addTimer`

A low-level timer is convenient when a background task doesn't need a `Future` — for example, a periodic health check that doesn't return a value externally.

```nim
proc startHeartbeat(intervalMs: int) =
  proc tick(fd: AsyncFD): bool =
    echo "heartbeat"
    result = false  # false — keep the periodic firings going
  addTimer(intervalMs, oneshot = false, cb = tick)
```

---

### Gracefully stopping the dispatcher via an `AsyncEvent`

`AsyncEvent` is the only safe way to wake the dispatcher from another thread (for example, from a shutdown-signal handler), without resorting to unsafe tricks like forcibly stopping a thread.

```nim
var running = true
let stopEvent = newAsyncEvent()

proc onStopSignal(fd: AsyncFD): bool =
  running = false
  result = true

addEvent(stopEvent, onStopSignal)

proc mainLoop() =
  while running:
    poll(100)

# from another thread or a signal handler:
# trigger(stopEvent)
```

---

### Connecting to the first available address via `dial`

`dial` avoids manually iterating over DNS records — useful for clients that need to work equally well with hosts returning both IPv4 and IPv6 addresses.

```nim
proc connectResilient(host: string, port: int): owned(Future[AsyncFD]) {.async.} =
  result = await dial(host, Port(port))
  echo "connected, descriptor = ", int(result)
```

---

## Quick reference table

| Task | What to use |
|---|---|
| Get/create the thread's dispatcher | `getGlobalDispatcher` |
| Register an external descriptor | `register` |
| Check whether a descriptor is registered | `contains` |
| Unsubscribe a descriptor/event | `unregister` |
| One tick of the event loop | `poll` |
| Drain all available events within a time budget | `drain` |
| An infinite service loop | `runForever` |
| Await a `Future` from synchronous code | `waitFor` |
| Defer a call until the next tick | `callSoon` |
| Low-level subscription to read/write readiness | `addRead` / `addWrite` |
| A low-level timer without a `Future` | `addTimer` |
| Wait for a process to exit | `addProcess` |
| Wait for a POSIX signal | `addSignal` |
| A user-defined, manually triggered event | `newAsyncEvent` + `addEvent` + `trigger` |
| Create a socket and register it right away | `createAsyncNativeSocket` |
| Close a socket and unregister it | `closeSocket` |
| Read data from a TCP socket | `recv` / `recvInto` |
| Send data over a TCP socket | `send` |
| Exchange UDP datagrams | `sendTo` / `recvFromInto` |
| Accept an incoming connection | `accept` / `acceptAddr` |
| Connect to a host by name, iterating over addresses | `dial` |
| Connect an already-created socket | `connect` |
| An asynchronous delay | `sleepAsync` |
| Bound an operation by time | `withTimeout` |
| Collect all data from a `FutureStream[string]` | `readAll` |
| Number of descriptors in use now / the OS limit | `activeDescriptors` / `maxDescriptors` |

---

## Summary: which procedure to choose

- Just need to connect to a host by name without worrying about IPv4/IPv6 → use `dial`.
- Already have a ready-made socket with special options, just need to connect it → use `connect`.
- Need to accept a connection and know the client's address → use `acceptAddr`; if the address doesn't matter → `accept`.
- Need to read data with no extra allocations in a hot loop → use `recvInto` instead of `recv`.
- Need to bound an operation by time without forcibly cancelling it → wrap the `Future` in `withTimeout`.
- Need an asynchronous pause → `sleepAsync`, not a blocking `sleep`.
- Need to call async code from a synchronous entry point (`main`, tests) → `waitFor`, never `waitFor` inside an `{.async.}` procedure.
- Need to fire off an operation "and forget it", without awaiting or manually checking the result → `asyncCheck` (from `std/asyncfutures`), not a bare `discard`.
- Need to wake the dispatcher from another thread → an `AsyncEvent` (`newAsyncEvent`/`addEvent`/`trigger`), not attempts to write to variables directly from another thread.
- Need a periodic background call without creating a `Future` → `addTimer` with `oneshot = false`.
- Need to check whether descriptors are leaking → compare `activeDescriptors()` before and after operations; to size limits ahead of time → `maxDescriptors()`.
- Need to handle a process-termination signal (`SIGTERM`) asynchronously, without blocking handlers → `addSignal` (POSIX platforms only).
