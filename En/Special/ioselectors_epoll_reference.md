# Reference: `ioselectors` Module — epoll Backend (Nim)

> This file is the **Linux-specific implementation** of the portable `ioselectors` interface, built on top of the `epoll(7)` system call.  
> It provides high-performance I/O multiplexing: monitoring file descriptors, sockets, timers, POSIX signals, user-defined events, and process termination.

> ℹ️ **Platform:** Linux only. Does not compile on macOS, Windows, or other OSes.  
> Signal and process registration is **not available** on Android.

---

## Table of Contents

- [Data Types](#data-types)
- [Creating and Closing a Selector](#creating-and-closing-a-selector)
- [Registering Descriptors](#registering-descriptors)
- [Registering Special Event Sources](#registering-special-event-sources)
- [Updating and Unregistering](#updating-and-unregistering)
- [Waiting for Events](#waiting-for-events)
- [User Event Management](#user-event-management)
- [Data Access and Utilities](#data-access-and-utilities)
- [Event Types](#event-types)
- [How It Works](#how-it-works)
- [Quick Reference](#quick-reference)

---

## Data Types

### `Selector[T]`

The main structure — a wrapper around an `epoll` file descriptor. The type parameter `T` is an arbitrary user-defined type associated with each registered descriptor.

- When compiled **with thread support** (`hasThreadSupport`): `ptr SelectorImpl[T]` — allocated in shared memory.
- Without thread support: `ref SelectorImpl[T]` — a regular heap-allocated object.

The public field `count` holds the number of actively monitored descriptors.

```nim
var sel = newSelector[MyData]()
```

---

### `SelectEvent`

A pointer to a user-defined event backed by a Linux `eventfd`. Used for waking up a waiting event loop from another thread or an arbitrary piece of code.

```nim
var ev = newSelectEvent()
```

---

### `SignalFdInfo`

Maps to `struct signalfd_siginfo` from `<sys/signalfd.h>`. Filled in by the kernel when a signal arrives via `signalfd`. Key fields:

| Field | Type | Description |
|---|---|---|
| `ssi_signo` | `uint32` | Signal number |
| `ssi_pid` | `uint32` | PID of the sender |
| `ssi_uid` | `uint32` | UID of the sender |
| `ssi_code` | `int32` | Signal source code |
| `ssi_status` | `int32` | Exit code / signal number for `SIGCHLD` |

> Not available on Android.

---

## Creating and Closing a Selector

### `newSelector[T](): Selector[T]`

Creates a new epoll selector. Internally calls `epoll_create1(O_CLOEXEC)`. The initial descriptor table capacity is 1024 entries (automatically grown on demand). The maximum limit is determined by `getrlimit()`.

Raises `OSError` if the epoll file descriptor cannot be created.

```nim
type MyData = object
  name: string

var sel = newSelector[MyData]()
defer: sel.close()
```

---

### `close[T](s: Selector[T])`

Closes the epoll file descriptor and releases all associated resources. In multithreaded mode, also frees the shared memory allocation.

> Always call this when done to avoid file descriptor leaks.

```nim
sel.close()
```

---

## Registering Descriptors

### `registerHandle[T](s: Selector[T], fd: int | SocketHandle, events: set[Event], data: T)`

Registers an existing file descriptor or socket `fd` for the given `events`. The user data `data` is associated with the descriptor and returned with every matching event notification.

Valid events: `Event.Read`, `Event.Write`, or both. `EPOLLRDHUP` (remote connection hangup) is always added implicitly.

```nim
import std/net

var sock = newSocket()
sock.bindAddr(Port(8080))
sock.listen()

type ConnData = object
  id: int

var sel = newSelector[ConnData]()
sel.registerHandle(sock.getFd(), {Event.Read}, ConnData(id: 1))
```

---

### `updateHandle[T](s: Selector[T], fd: int | SocketHandle, events: set[Event])`

Changes the set of monitored events for an **already registered** descriptor. Only valid for plain file descriptors and sockets — not applicable to timers, signals, processes, user events, or Oneshot entries.

- If `events == {}` and the descriptor was active — removes it from epoll (`EPOLL_CTL_DEL`).
- If `events != {}` and the descriptor was inactive — adds it back (`EPOLL_CTL_ADD`).
- Otherwise — modifies the event mask (`EPOLL_CTL_MOD`).

```nim
# Switch from read to write monitoring:
sel.updateHandle(fd, {Event.Write})

# Temporarily suspend monitoring:
sel.updateHandle(fd, {})

# Resume:
sel.updateHandle(fd, {Event.Read})
```

---

## Registering Special Event Sources

### `registerTimer[T](s: Selector[T], timeout: int, oneshot: bool, data: T): int`

Registers a timer using `timerfd_create(CLOCK_MONOTONIC)`. Returns the `timerfd` file descriptor (for use with `unregister`).

- `timeout` — interval in **milliseconds**.
- `oneshot = true` — fires once; after firing it is marked `Event.Finished` and removed from active epoll monitoring (but not from the key table).
- `oneshot = false` — fires repeatedly at the given interval.

```nim
type TimerData = object
  label: string

# Repeating timer every 500 ms:
let timerFd = sel.registerTimer(500, false, TimerData(label: "tick"))

# One-shot timer after 2 seconds:
let onceFd = sel.registerTimer(2000, true, TimerData(label: "once"))
```

---

### `registerSignal[T](s: Selector[T], signal: int, data: T): int`

Registers a POSIX signal for monitoring via `signalfd`. Blocks the signal in the current thread's signal mask and redirects it through epoll. Returns the `signalfd` descriptor.

> ℹ️ **Not available on Android.**

```nim
import std/posix

type SigData = object
  signum: int

# Watch for SIGTERM:
let sigFd = sel.registerSignal(SIGTERM, SigData(signum: SIGTERM))
```

---

### `registerProcess[T](s: Selector, pid: int, data: T): int`

Registers monitoring for the termination of a child process with the given `pid`, using `SIGCHLD` and `signalfd`. Marked as `Event.Oneshot` — fires exactly once when the process exits. Returns the `signalfd` descriptor.

> ℹ️ **Not available on Android.**

```nim
type ProcData = object
  pid: int

let childPid = 12345
let procFd = sel.registerProcess(childPid, ProcData(pid: childPid))
```

---

### `registerEvent[T](s: Selector[T], ev: SelectEvent, data: T)`

Registers a user event `ev` (created with `newSelectEvent()`) in the selector. The event fires when `trigger(ev)` is called.

```nim
var ev = newSelectEvent()
sel.registerEvent(ev, MyData(id: 42))
```

---

## Updating and Unregistering

### `unregister[T](s: Selector[T], fd: int | SocketHandle)`

Unregisters descriptor `fd`. Handles all types correctly:
- Plain descriptors / sockets — removes from epoll.
- Timers — closes the `timerfd`.
- Signals / processes — unblocks the signal in the signal mask, closes the `signalfd`.

```nim
sel.unregister(sock.getFd())
sel.unregister(timerFd)
```

---

### `unregister[T](s: Selector[T], ev: SelectEvent)`

Unregisters a user event `ev`. Removes the `eventfd` descriptor from epoll. Does **not** close the underlying `ev` — call `ev.close()` separately.

```nim
sel.unregister(ev)
ev.close()
```

---

## Waiting for Events

### `select[T](s: Selector[T], timeout: int): seq[ReadyKey]`

Waits for events and returns a sequence of ready keys. Internally calls `epoll_wait` with the given timeout.

- `timeout > 0` — wait at most `timeout` milliseconds.
- `timeout == 0` — return immediately (non-blocking poll).
- `timeout == -1` — block indefinitely.

Returns at most 64 events per call. Interruption by a signal (`EINTR`) is handled gracefully — returns an empty list without raising an error.

```nim
while true:
  let events = sel.select(100)   # wait up to 100 ms
  for key in events:
    echo "Event on fd=", key.fd, " events=", key.events
```

---

### `selectInto[T](s: Selector[T], timeout: int, results: var openArray[ReadyKey]): int`

Like `select`, but writes results into a **caller-provided** buffer `results`. Returns the number of filled entries. Avoids heap allocation on every call, useful in hot loops.

```nim
var buf: array[64, ReadyKey]
while true:
  let n = sel.selectInto(100, buf)
  for i in 0 ..< n:
    processEvent(buf[i])
```

---

## User Event Management

### `newSelectEvent(): SelectEvent`

Creates a new user event object backed by `eventfd(0, O_CLOEXEC | O_NONBLOCK)`. Used for inter-thread or cross-component signalling.

```nim
var wakeup = newSelectEvent()
```

---

### `trigger(ev: SelectEvent)`

Fires the user event by writing `1` to the underlying `eventfd`. This causes `Event.User` to appear in the next `select` / `selectInto` call. Safe to call from a different thread.

```nim
# In a producer / other thread:
wakeup.trigger()

# In the main event loop after select():
for key in events:
  if Event.User in key.events:
    echo "Wake-up signal received"
```

---

### `close(ev: SelectEvent)`

Closes the `eventfd` file descriptor and frees associated memory. Call after `unregister(ev)`.

```nim
sel.unregister(wakeup)
wakeup.close()
```

---

## Data Access and Utilities

### `getData[T](s: Selector[T], fd: SocketHandle | int): var T`

Returns a mutable reference to the user data associated with the registered descriptor `fd`. Behaviour is undefined if `fd` is not registered.

```nim
var d = sel.getData(sock.getFd())
d.name = "updated"
```

---

### `setData[T](s: Selector[T], fd: SocketHandle | int, data: T): bool`

Replaces the user data for descriptor `fd`. Returns `true` if the descriptor is registered and the data was updated; `false` otherwise.

```nim
let ok = sel.setData(fd, MyData(id: 99))
assert ok
```

---

### `withData[T](s, fd, value, body)` — template (2 forms)

Safely accesses descriptor data — executes `body` only if `fd` is registered. The variable `value` inside `body` is a `ptr T` pointing directly to the stored data, allowing in-place modification.

**Form 1 — execute only if registered:**
```nim
sel.withData(fd, val):
  val.counter += 1
```

**Form 2 — with an else branch:**
```nim
sel.withData(fd, val):
  echo "fd registered: ", val.name
do:
  echo "fd not found"
```

---

### `contains[T](s: Selector[T], fd: SocketHandle | int): bool`

Returns `true` if descriptor `fd` is registered in the selector. Supports the `in` operator.

```nim
if fd in sel:
  echo "fd is active"

assert sel.contains(sock.getFd())
```

---

### `isEmpty[T](s: Selector[T]): bool` — template

Returns `true` when no descriptors are actively monitored (`s.count == 0`).

```nim
if sel.isEmpty():
  echo "No active descriptors"
```

---

### `getFd[T](s: Selector[T]): int`

Returns the raw epoll file descriptor (`epollFD`). Used for nested multiplexing — registering one epoll fd inside another selector.

```nim
let epollFd = sel.getFd()
```

---

## Event Types

The `set[Event]` type is used both when registering and in the `ReadyKey.events` field returned by `select`:

| Value | Description |
|---|---|
| `Event.Read` | Data available for reading (`EPOLLIN`) |
| `Event.Write` | Writing possible without blocking (`EPOLLOUT`) |
| `Event.Timer` | Timer has fired |
| `Event.Signal` | POSIX signal received (via `signalfd`) |
| `Event.Process` | Child process has terminated |
| `Event.User` | User event fired (via `eventfd`) |
| `Event.Error` | Error on descriptor (`EPOLLERR` / `EPOLLHUP`) |
| `Event.Oneshot` | Event fires only once, then removed from epoll |
| `Event.Finished` | Internal flag: Oneshot event already consumed |

### `ReadyKey`

The structure returned by `select` / `selectInto`:

```nim
type ReadyKey = object
  fd: int                  # file descriptor
  events: set[Event]       # set of events that occurred
  errorCode: OSErrorCode   # error code when Event.Error is set
```

---

## How It Works

### Lifecycle

```
newSelector()                   # epoll_create1()
  │
  ├─ registerHandle(fd, ...)    # epoll_ctl(EPOLL_CTL_ADD)
  ├─ registerTimer(ms, ...)     # timerfd_create() + epoll_ctl(ADD)
  ├─ registerSignal(sig, ...)   # signalfd() + epoll_ctl(ADD)
  ├─ registerProcess(pid, ...)  # signalfd(SIGCHLD) + epoll_ctl(ADD)
  ├─ registerEvent(ev, ...)     # eventfd already created, epoll_ctl(ADD)
  │
  └─ loop:
       select(timeout)          # epoll_wait()
         │
         ├─ Event.Read/Write  → handle I/O data
         ├─ Event.Timer       → read uint64 from timerfd
         ├─ Event.Signal      → read SignalFdInfo from signalfd
         ├─ Event.Process     → read SignalFdInfo, verify pid
         ├─ Event.User        → read uint64 from eventfd
         └─ Event.Error       → read SO_ERROR via getsockopt
  │
  ├─ unregister(fd)             # epoll_ctl(EPOLL_CTL_DEL), close fd
  └─ close()                    # close epollFD, free memory
```

### Automatic table growth

The internal descriptor table starts at 1024 entries and is doubled as needed (via the `checkFd` template). The upper bound is the system file descriptor limit from `getrlimit(RLIMIT_NOFILE)`.

### Oneshot events

When registered with `Event.Oneshot` (timers, processes), epoll uses the `EPOLLONESHOT` flag. After the first firing the descriptor is marked `Event.Finished` and removed from active monitoring, but **remains** in the key table so the user data is still accessible via `getData`. An explicit `unregister` call is needed to fully remove it.

---

## Quick Reference

### Operations table

| Operation | Function |
|---|---|
| Create selector | `newSelector[T]()` |
| Close selector | `close(s)` |
| Register fd / socket | `registerHandle(s, fd, events, data)` |
| Update fd events | `updateHandle(s, fd, events)` |
| Unregister fd | `unregister(s, fd)` |
| Register timer | `registerTimer(s, ms, oneshot, data)` |
| Register signal | `registerSignal(s, signal, data)` |
| Register process | `registerProcess(s, pid, data)` |
| Create user event | `newSelectEvent()` |
| Register user event | `registerEvent(s, ev, data)` |
| Fire user event | `trigger(ev)` |
| Close user event | `close(ev)` |
| Unregister user event | `unregister(s, ev)` |
| Wait for events → seq | `select(s, timeout)` |
| Wait for events → buffer | `selectInto(s, timeout, buf)` |
| Get fd data | `getData(s, fd)` |
| Set fd data | `setData(s, fd, data)` |
| Safe data access | `withData(s, fd, val): body` |
| Check registration | `contains(s, fd)` / `fd in s` |
| Any descriptors? | `isEmpty(s)` |
| Get epoll fd | `getFd(s)` |

### Typical event loop

```nim
import std/[ioselectors, net, os]

type ConnData = object
  id: int

var sel = newSelector[ConnData]()
defer: sel.close()

var server = newSocket()
server.setSockOpt(OptReuseAddr, true)
server.bindAddr(Port(8080))
server.listen()
sel.registerHandle(server.getFd(), {Event.Read}, ConnData(id: 0))

while true:
  let events = sel.select(1000)
  for key in events:
    if Event.Error in key.events:
      echo "Error on fd=", key.fd
      sel.unregister(key.fd)
    elif Event.Read in key.events:
      if key.fd == server.getFd().int:
        var client = server.accept()[0]
        sel.registerHandle(client.getFd(), {Event.Read}, ConnData(id: key.fd))
      else:
        # read client data
        discard
```

### Notes

- This file is an **internal implementation** used through `std/ioselectors`. For portable code, import `std/ioselectors` instead.
- `select` returns at most **64 events** per call (`MAX_EPOLL_EVENTS`).
- `EINTR` in `epoll_wait` is handled transparently — `select` returns 0 without raising.
- `registerSignal` and `registerProcess` are **unavailable** on Android.
- With `--threads:on`, `Selector` is allocated in shared memory (`ptr`) and may safely be used across threads.
