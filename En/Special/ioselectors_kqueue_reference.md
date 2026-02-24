# `ioselectors_kqueue` Module Reference

> **Module:** `std/ioselectors` (BSD kqueue backend)  
> **Language:** Nim  
> **Platform:** macOS, FreeBSD, NetBSD, OpenBSD, DragonFly BSD  
> **Purpose:** Implements the `ioselectors` API on top of the BSD kernel's `kqueue`/`kevent` mechanism, enabling efficient I/O event multiplexing — watching many file descriptors simultaneously without polling.

---

## Background: what is kqueue?

Before looking at the API, it's worth understanding the engine underneath.

**kqueue** is a BSD kernel interface for event notification. Rather than asking "is this socket ready yet?" in a loop (polling), your program registers interest in certain events with the kernel and then calls `kevent()` to *block* until one or more of those events actually occur. The kernel does the waiting; your process sleeps and consumes no CPU.

Compared to POSIX `select`/`poll`, kqueue can watch an almost unlimited number of descriptors at once, and it can watch far more than just sockets — it handles timers, signals, child process termination, and filesystem changes through the same unified interface.

This module is the BSD implementation of Nim's `std/ioselectors` API. You don't normally import it directly; `std/ioselectors` selects the right backend automatically based on the target OS.

### How changes get submitted to the kernel

Every function that registers or removes interest queues a `KEvent` change descriptor internally (via the internal `modifyKQueue` template). Those changes are sent to the kernel in one of two modes:

- **Immediate mode** (default): changes are flushed to the kernel via `kevent()` immediately after being queued, via `flushKQueue`.
- **Cache mode** (compile flag `-d:kqcache`): changes accumulate and are submitted to the kernel in a single batch call inside `selectInto`. This reduces syscall overhead when many registrations happen before the next wait.

---

## Types

### `Selector[T]`

```nim
# single-threaded build:
type Selector*[T] = ref SelectorImpl[T]
# thread-safe build (--threads:on):
type Selector*[T] = ptr SelectorImpl[T]
```

The central object. It wraps a kqueue file descriptor and maintains a table mapping every watched file descriptor to a `SelectorKey[T]` that stores the registered events and a user-supplied data value of type `T`. The type parameter `T` lets you attach any Nim value to a watched descriptor — connection state, a callback, a pointer to your own context, etc.

In a thread-safe build (`--threads:on`), `Selector` is a raw `ptr` into shared memory and all access to the pending-changes buffer is protected by a `Lock`.

### `SelectEvent`

```nim
type SelectEvent* = ptr SelectEventImpl
```

A manually triggerable event. Internally it is an OS pipe (a read end and a write end). When `trigger` writes a byte to the write end, kqueue fires on the read end. Because it lives in shared memory (`allocShared0`), the same `SelectEvent` can be shared between threads, making it the standard way to wake a sleeping `select`/`selectInto` call from another thread.

---

## Lifecycle functions

### `newSelector`

```nim
proc newSelector*[T](): owned(Selector[T])
```

#### What it does

Creates and initialises a fresh selector. Internally it:

1. Queries the kernel via `sysctl` for the maximum number of file descriptors the process is allowed to open, then pre-allocates the `fds` table to that size. This is why looking up a watched descriptor is O(1) — it's a direct array index.
2. Opens a `kqueue` file descriptor with the kernel call `kqueue()`.
3. Creates a dummy TCP socket (`usock`) whose sole purpose is to be duplicated with `F_DUPFD_CLOEXEC` by `getUnique` later. Because timer, signal, and process events have no real file descriptor in kqueue, the module fabricates unique fd-like integers by duplicating this socket. The duplicated integers are never connected to any real I/O; they exist purely as keys into the `fds` table.
4. In a thread-safe build, allocates everything in shared memory and initialises a mutex.

#### Return value

An `owned` `Selector[T]` ready to use. Call `close` when done.

#### Example

```nim
import std/ioselectors

type ConnState = object
  id: int

let sel = newSelector[ConnState]()
defer: sel.close()
```

---

### `close` (Selector)

```nim
proc close*[T](s: Selector[T])
```

#### What it does

Releases all resources held by the selector:

- Closes the kqueue file descriptor (`kqFD`).
- Closes the dummy socket used for generating unique indices.
- In thread-safe builds: destroys the lock and frees shared memory.

Raises an `IOSelectorsError` if either `close` syscall fails. You are responsible for unregistering all descriptors *before* calling this, or at least for closing the underlying OS resources yourself, since `close` on the selector does not automatically unregister watchers.

#### Example

```nim
let sel = newSelector[int]()
# ... use sel ...
sel.close()
```

---

### `newSelectEvent`

```nim
proc newSelectEvent*(): SelectEvent
```

#### What it does

Creates a `SelectEvent` backed by an OS pipe. Both ends of the pipe are set to non-blocking mode. The event is allocated in shared memory so that it can safely be passed between threads.

The typical pattern: one thread calls `select`/`selectInto` and blocks; another thread calls `trigger` on this event to unblock it.

#### Example

```nim
import std/ioselectors

let wakeup = newSelectEvent()
defer: wakeup.close()
```

---

### `trigger`

```nim
proc trigger*(ev: SelectEvent)
```

#### What it does

Signals the event by writing 8 bytes (a `uint64` value of `1`) to the write end of the internal pipe. This makes the selector's `kevent()` call return with a `{Event.User}` result for the descriptor associated with this event.

`trigger` is safe to call from any thread. It is designed to be called from a different thread than the one blocked in `select`/`selectInto`.

Raises `IOSelectorsError` if the write fails for any reason other than a full pipe buffer.

#### Example

```nim
# in a worker thread:
wakeupEvent.trigger()

# in the main loop thread, selectInto returns with Event.User:
let ready = sel.select(timeout = -1)
for key in ready:
  if Event.User in key.events:
    echo "woken up by another thread"
```

---

### `close` (SelectEvent)

```nim
proc close*(ev: SelectEvent)
```

#### What it does

Closes both the read and write ends of the underlying pipe, then frees the shared memory block. Always call this when the `SelectEvent` is no longer needed. Make sure to call `unregister` on the selector *first*, before closing the event.

Raises `IOSelectorsError` if either `close` syscall fails.

---

## Registration functions

All `register*` functions attach a file descriptor (or a virtual identifier) to the selector and tell the kernel which events to watch for. They all:

- Validate that the descriptor number is within bounds (`checkFd`).
- Store a `SelectorKey` in the `fds` table with the event set and user data.
- Queue one or more `KEvent` change descriptors.
- Flush changes to the kernel immediately (unless `-d:kqcache` is set).
- Increment `s.count`.

### `registerHandle`

```nim
proc registerHandle*[T](s: Selector[T], fd: int | SocketHandle,
                        events: set[Event], data: T)
```

#### What it does

Registers an existing file descriptor — a socket, a pipe end, `stdin`, any readable/writable OS resource — with the selector. You specify which events matter: `{Event.Read}`, `{Event.Write}`, or both.

This is the workhorse for network servers: you register every accepted client socket here so the selector can tell you which ones have data waiting or are ready for a write.

`events` may be `{}` to register the descriptor without actually watching any events yet; use `updateHandle` later to activate watching.

#### Parameters

| Parameter | Description |
|-----------|-------------|
| `fd` | The OS file descriptor or socket handle to watch |
| `events` | `{Event.Read}` and/or `{Event.Write}` |
| `data` | User value stored alongside the descriptor |

#### Example

```nim
import std/[ioselectors, nativesockets]

let sel = newSelector[string]()
let sock = createNativeSocket()
sock.setBlocking(false)
sel.registerHandle(sock, {Event.Read}, "client-1")
```

---

### `updateHandle`

```nim
proc updateHandle*[T](s: Selector[T], fd: int | SocketHandle,
                      events: set[Event])
```

#### What it does

Changes the set of events being watched for an already-registered file descriptor. You can add `Event.Read` or `Event.Write`, remove one of them, switch between them, or pass `{}` to stop watching events without unregistering.

Only works on plain I/O handles (sockets, pipes). It explicitly forbids descriptors registered with timer, signal, process, vnode, user, oneshot, or error event types — those have their own registration procedures and cannot be updated this way.

This is how you implement write-readiness watching selectively: register with `{Event.Read}` initially, then call `updateHandle` to add `Event.Write` only when you have data to send, and remove it again once the write buffer is drained.

#### Example

```nim
# Initially watching reads only
sel.registerHandle(sock, {Event.Read}, myData)

# Later, when we have data to send:
sel.updateHandle(sock, {Event.Read, Event.Write})

# After the write drains:
sel.updateHandle(sock, {Event.Read})
```

---

### `registerTimer`

```nim
proc registerTimer*[T](s: Selector[T], timeout: int, oneshot: bool,
                       data: T): int {.discardable.}
```

#### What it does

Registers a kernel timer that fires every `timeout` milliseconds. No real file descriptor is needed — the module creates a synthetic unique integer identifier (by duplicating the internal dummy socket) and uses kqueue's `EVFILT_TIMER`.

- `oneshot = false`: the timer fires repeatedly until unregistered. Each firing appears as `Event.Timer` in the select results.
- `oneshot = true`: the timer fires exactly once. After firing, `Event.Finished` is added to the key's event set and `s.count` is decremented internally, but the key stays in `fds` so you can still read the associated `data`. Call `unregister` to clean it up.

Returns the synthetic file descriptor integer, which you need if you want to call `unregister` later.

#### Example

```nim
# Repeating timer: fires every 500ms
let timerFd = sel.registerTimer(500, oneshot = false, data = "heartbeat")

# Oneshot timer: fires once after 2 seconds
let onceFd = sel.registerTimer(2000, oneshot = true, data = "startup-delay")
```

---

### `registerSignal`

```nim
proc registerSignal*[T](s: Selector[T], signal: int,
                        data: T): int {.discardable.}
```

#### What it does

Registers a POSIX signal number so that the selector reports it as `Event.Signal` instead of delivering it to a signal handler. Internally:

1. The signal is **blocked** in the process signal mask (so the default handler won't run).
2. The signal disposition is set to `SIG_IGN` (Linux-compatible "eating" semantics).
3. kqueue's `EVFILT_SIGNAL` is used to deliver the signal as a normal event.

This converts the notoriously awkward signal-handling model into the same event loop you use for I/O, eliminating signal-unsafe callbacks entirely.

Returns the synthetic fd integer for later `unregister`.

#### Example

```nim
import std/posix
let sigFd = sel.registerSignal(SIGTERM.int, data = "got-sigterm")
```

---

### `registerProcess`

```nim
proc registerProcess*[T](s: Selector[T], pid: int,
                         data: T): int {.discardable.}
```

#### What it does

Registers a child process by its PID so that the selector fires `Event.Process` when that process exits (`NOTE_EXIT`). This is always a oneshot event: once the process exits, the key is marked `Event.Finished` and `s.count` is decremented, but the key remains in `fds` until you call `unregister`.

This lets you integrate child process reaping into the same event loop as I/O, without `SIGCHLD` handlers.

Returns the synthetic fd integer for later `unregister`.

#### Example

```nim
let childPid = startProcess("worker")
let procFd = sel.registerProcess(childPid.processID, data = "worker-proc")
```

---

### `registerEvent`

```nim
proc registerEvent*[T](s: Selector[T], ev: SelectEvent, data: T)
```

#### What it does

Registers a `SelectEvent` (a manually triggerable event, backed by a pipe) with the selector. Once registered, any call to `ev.trigger()` will cause `selectInto`/`select` to return a `{Event.User}` result for this descriptor.

Unlike `registerHandle`, this uses the pipe's read end fd directly as the key — no synthetic identifier needed.

#### Example

```nim
let wakeup = newSelectEvent()
sel.registerEvent(wakeup, data = "wakeup-signal")
```

---

### `registerVnode`

```nim
proc registerVnode*[T](s: Selector[T], fd: cint, events: set[Event], data: T)
```

#### What it does

Registers a file descriptor (typically an open file or directory) for filesystem change notifications via kqueue's `EVFILT_VNODE`. You specify which kinds of changes to watch:

| Event | kqueue flag | Meaning |
|-------|-------------|---------|
| `Event.VnodeDelete` | `NOTE_DELETE` | File was deleted |
| `Event.VnodeWrite` | `NOTE_WRITE` | File data was written |
| `Event.VnodeExtend` | `NOTE_EXTEND` | File was extended (grew) |
| `Event.VnodeAttrib` | `NOTE_ATTRIB` | Metadata changed (chmod, chown…) |
| `Event.VnodeLink` | `NOTE_LINK` | Link count changed |
| `Event.VnodeRename` | `NOTE_RENAME` | File was renamed |
| `Event.VnodeRevoke` | `NOTE_REVOKE` | Access was revoked |

The `EV_CLEAR` flag is always set, so the event fires once per change rather than continuously.

This is how you build a file watcher without polling — open the target file, pass its fd here, and the selector will wake you on any of the changes you specified.

#### Example

```nim
import std/posix
let fileFd = posix.open("config.json", O_RDONLY)
sel.registerVnode(fileFd, {Event.VnodeWrite, Event.VnodeDelete},
                  data = "config-watcher")
```

---

## Unregistration

### `unregister` (fd)

```nim
proc unregister*[T](s: Selector[T], fd: int | SocketHandle)
```

#### What it does

Removes a file descriptor from the selector. The exact cleanup depends on what type of event it was registered with:

- **Read/Write**: removes `EVFILT_READ` and/or `EVFILT_WRITE` from kqueue.
- **Timer**: removes `EVFILT_TIMER` and closes the synthetic fd (unless already `Finished`).
- **Signal**: unblocks the signal, restores `SIG_DFL`, removes `EVFILT_SIGNAL`, closes the synthetic fd.
- **Process**: removes `EVFILT_PROC` (unless already `Finished`), closes the synthetic fd.
- **Vnode**: removes `EVFILT_VNODE`.
- **User**: removes `EVFILT_READ` (the pipe-based fd).

In all cases, the `SelectorKey` entry is cleared and `s.count` is decremented (unless it was already decremented at `Finished` time).

---

### `unregister` (SelectEvent)

```nim
proc unregister*[T](s: Selector[T], ev: SelectEvent)
```

#### What it does

Removes a `SelectEvent` from the selector. Removes the `EVFILT_READ` filter for the pipe's read end and clears the key. Always call this before calling `ev.close()`.

---

## Waiting for events

### `selectInto`

```nim
proc selectInto*[T](s: Selector[T], timeout: int,
                    results: var openArray[ReadyKey]): int
```

#### What it does

Blocks until one or more registered events occur, or until `timeout` elapses, and writes results into the caller-supplied `results` array. This is the zero-allocation variant: you own the buffer, so no heap allocation happens per call.

The return value is the number of ready events written into `results`. Indices `0 ..< return_value` are valid; the rest of `results` is untouched.

**Timeout semantics:**

| Value | Meaning |
|-------|---------|
| `0` | Return immediately (non-blocking check) |
| `> 0` | Wait at most this many milliseconds |
| `-1` | Wait indefinitely |

If the call is interrupted by a signal (`EINTR`), it returns `0` instead of raising.

In cache mode (`-d:kqcache`), any pending registration changes are submitted to the kernel *atomically in the same `kevent()` call* that waits for results, reducing syscall count.

**Event demultiplexing:** for each returned `KEvent`, the proc examines the filter field and translates it back to the appropriate Nim `Event` set:

- `EVFILT_READ` → `Event.Read` (or `Event.User` if the fd was registered as a user event — it reads from the pipe to drain it)
- `EVFILT_WRITE` → `Event.Write`
- `EVFILT_TIMER` → `Event.Timer`
- `EVFILT_VNODE` → `Event.Vnode` + fine-grained vnode events
- `EVFILT_SIGNAL` → `Event.Signal`
- `EVFILT_PROC` → `Event.Process`
- `EV_ERROR` flag → `Event.Error` with error code
- `EV_EOF` flag → `Event.Error` with `ECONNRESET` (for sockets whose remote end closed)

#### Example

```nim
var ready: array[64, ReadyKey]
while true:
  let n = sel.selectInto(timeout = 1000, ready)
  for i in 0 ..< n:
    let key = ready[i]
    if Event.Read in key.events:
      echo "fd ", key.fd, " has data"
    if Event.Error in key.events:
      echo "fd ", key.fd, " error: ", key.errorCode
```

---

### `select`

```nim
proc select*[T](s: Selector[T], timeout: int): seq[ReadyKey]
```

#### What it does

Convenience wrapper around `selectInto` that allocates and returns a `seq[ReadyKey]`. Internally it pre-allocates a sequence of `MAX_KQUEUE_EVENTS` (64) slots, calls `selectInto`, then trims the sequence to the actual count.

Use `selectInto` in hot loops where allocations matter. Use `select` when convenience trumps performance.

#### Example

```nim
let ready = sel.select(timeout = 500)
for key in ready:
  echo "event on fd: ", key.fd, " events: ", key.events
```

---

## Data access functions

### `getData`

```nim
proc getData*[T](s: Selector[T], fd: SocketHandle | int): var T
```

#### What it does

Returns a mutable reference to the user data `T` associated with the given file descriptor. The returned reference points directly into the `fds` table, so modifying it modifies the stored value in place.

If `fd` is not registered, the behaviour is undefined (no bounds check on the data itself, only on the fd range). Use `contains` to check first.

#### Example

```nim
type Conn = object
  bytesSent: int

sel.registerHandle(sock, {Event.Write}, Conn(bytesSent: 0))
sel.getData(sock).bytesSent += 1024
```

---

### `setData`

```nim
proc setData*[T](s: Selector[T], fd: SocketHandle | int, data: T): bool
```

#### What it does

Replaces the user data for a registered descriptor. Returns `true` if the descriptor was registered and the data was updated; `false` if the descriptor is not found (not registered).

#### Example

```nim
let updated = sel.setData(sock, Conn(bytesSent: 0))
if not updated:
  echo "sock was not registered"
```

---

### `withData` (found branch)

```nim
template withData*[T](s: Selector[T], fd: SocketHandle | int,
                      value, body: untyped)
```

#### What it does

Executes `body` with `value` bound as a pointer to the user data for `fd`, if `fd` is registered. If `fd` is not registered, `body` is not executed. This is the idiomatic way to access user data without a separate `contains` check — the registration check and data access are one atomic step.

`value` is a mutable pointer (`var T`), so you can modify the stored data in place.

#### Example

```nim
sel.withData(sock, connPtr):
  connPtr.bytesSent += 512
  echo "updated bytes sent"
```

---

### `withData` (found + not-found branches)

```nim
template withData*[T](s: Selector[T], fd: SocketHandle | int,
                      value, body1, body2: untyped)
```

#### What it does

Two-branch version: `body1` runs (with `value` bound) if `fd` is registered; `body2` runs if it is not. Mirrors an `if/else` pattern while bundling the registration check and data access together.

#### Example

```nim
sel.withData(sock, connPtr):
  connPtr.bytesSent += 512
do:
  echo "fd not registered — ignoring"
```

---

## Query functions

### `isEmpty`

```nim
template isEmpty*[T](s: Selector[T]): bool
```

#### What it does

Returns `true` if no descriptors are currently registered and active in the selector (`s.count == 0`). Useful as a loop exit condition: if nothing is being watched, there is no point calling `select`.

Note: `count` is decremented for oneshot events (timers, process exits) when they fire, even before you call `unregister`. So `isEmpty` can become `true` while those keys still exist in `fds`.

#### Example

```nim
while not sel.isEmpty:
  let ready = sel.select(timeout = -1)
  # process events...
```

---

### `contains`

```nim
proc contains*[T](s: Selector[T], fd: SocketHandle | int): bool {.inline.}
```

#### What it does

Returns `true` if the given file descriptor is currently registered in the selector. The check is a direct array lookup — O(1).

#### Example

```nim
if sock in sel:
  sel.updateHandle(sock, {Event.Read, Event.Write})
```

---

### `getFd`

```nim
proc getFd*[T](s: Selector[T]): int
```

#### What it does

Returns the OS file descriptor of the kqueue instance itself (`kqFD`). This is rarely needed in normal use, but can be useful if you want to nest selectors (watch one kqueue from another), pass the kqueue fd to a library that expects a raw fd, or examine it in debugging.

#### Example

```nim
echo "kqueue fd = ", sel.getFd()
```

---

## Event types quick reference

| `Event` value | Source | Meaning in `ReadyKey.events` |
|---------------|--------|------------------------------|
| `Event.Read` | `EVFILT_READ` | Data available to read |
| `Event.Write` | `EVFILT_WRITE` | Buffer has space, write won't block |
| `Event.Timer` | `EVFILT_TIMER` | Timer interval elapsed |
| `Event.Signal` | `EVFILT_SIGNAL` | POSIX signal received |
| `Event.Process` | `EVFILT_PROC` | Child process exited |
| `Event.Vnode` | `EVFILT_VNODE` | Filesystem change occurred |
| `Event.VnodeDelete` | `NOTE_DELETE` | Watched file was deleted |
| `Event.VnodeWrite` | `NOTE_WRITE` | Watched file was written |
| `Event.VnodeExtend` | `NOTE_EXTEND` | Watched file grew |
| `Event.VnodeAttrib` | `NOTE_ATTRIB` | Watched file metadata changed |
| `Event.VnodeLink` | `NOTE_LINK` | Hard link count changed |
| `Event.VnodeRename` | `NOTE_RENAME` | Watched file was renamed |
| `Event.VnodeRevoke` | `NOTE_REVOKE` | File access was revoked |
| `Event.User` | pipe read | `trigger()` was called |
| `Event.Error` | `EV_ERROR`/`EV_EOF` | Error on the descriptor |
| `Event.Finished` | internal | Oneshot event already fired (not returned to user) |

---

## Complete example: a minimal event loop

```nim
import std/[ioselectors, nativesockets, posix]

type Client = object
  id: int

let sel = newSelector[Client]()
defer: sel.close()

# Watch stdin for input
sel.registerHandle(0, {Event.Read}, Client(id: 0))

# A 1-second repeating heartbeat timer
let hbFd = sel.registerTimer(1000, oneshot = false, Client(id: -1))

# Watch for SIGINT so we can exit cleanly
let sigFd = sel.registerSignal(SIGINT.int, Client(id: -2))

while not sel.isEmpty:
  let ready = sel.select(timeout = 5000)
  for key in ready:
    if Event.Read in key.events and key.fd == 0:
      echo "stdin has data"
    if Event.Timer in key.events:
      echo "heartbeat"
    if Event.Signal in key.events:
      echo "SIGINT received, shutting down"
      sel.unregister(hbFd)
      sel.unregister(sigFd)
      sel.unregister(0)
      break
```

---

## Compile-time flags

| Flag | Effect |
|------|--------|
| `--threads:on` | Switches to shared-memory allocation and adds a `Lock` around the change buffer |
| `-d:kqcache` | Enables change caching: changes are batched and submitted in the same `kevent()` call as the wait |

---

## Notes and gotchas

- **Synthetic fds for timers/signals/processes.** These events have no real OS fd. The module duplicates an internal dummy socket with `F_DUPFD_CLOEXEC` to get unique integers usable as `fds` table keys. Those synthetic integers are closed by `unregister`, not by the caller.
- **Always `unregister` before `close`.** Closing an OS resource while it's still registered in the selector leaves stale entries in the `fds` table and may cause `doAssert` failures or silently wrong behaviour on future registrations of the same fd number.
- **`MAX_KQUEUE_EVENTS = 64`** limits how many events a single `select`/`selectInto` call can return. In a very high-throughput server you may need to loop `selectInto` until `result < maxres`, or switch to `selectInto` with a larger buffer.
- **`EV_EOF` on sockets** is translated to `Event.Error` with `ECONNRESET`. The module itself notes this is a simplification (the fflags field can carry other error codes) and may be refined in future.
- **Oneshot events linger in `fds`.** After firing, a oneshot timer or process event has `Event.Finished` set and is no longer counted, but it occupies a slot in `fds` until you call `unregister`. Not calling `unregister` is a resource leak.
- **`EINTR` is silently ignored.** If `kevent()` is interrupted by a signal while waiting, `selectInto` returns `0` rather than raising. This is correct behaviour — the signal will show up as an `Event.Signal` result on the next call if it was registered.
