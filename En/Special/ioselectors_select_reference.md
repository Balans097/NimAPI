# `ioselectors_select` Module Reference

> **Module:** `std/ioselectors` (POSIX/Windows `select()` backend)  
> **Language:** Nim  
> **Platform:** **Windows** and any POSIX system — the only `ioselectors` backend that runs on both. Used as the ultimate fallback when epoll, kqueue, and poll are all unavailable.  
> **Purpose:** Implements the `ioselectors` API on top of the classic `select()` system call, providing cross-platform I/O multiplexing at the cost of the most severe scalability limitations of all the backends.

---

## Background: what is select() and why is it the last resort?

`select()` is the oldest of the four Unix I/O multiplexing interfaces, dating to 4.2BSD in 1983. It predates `poll()`, `epoll`, and `kqueue`, and its design reflects the limitations of the era:

- You maintain **three bit-set structures** (`fd_set`) in user space — one for reads, one for writes, one for exceptions.
- Before each call to `select()`, you **copy all three sets** to temporary copies and pass those to the kernel, because `select()` **overwrites** them with results.
- The kernel scans the sets and clears bits for descriptors that are not ready. What remains tells you which descriptors are ready.
- `select()` has a **hard limit** on the maximum file descriptor number it can handle: `FD_SETSIZE`, which is typically 1024 on POSIX and 64 on Windows (where it counts sockets, not fd numbers).

These constraints make `select()` the least scalable multiplexer, but also the most portable: it works on every POSIX system and on Windows without extra headers.

### Why this backend exists at all

The `select` backend is the cross-platform lifeline of `ioselectors`. On Windows, neither epoll nor kqueue exists; `select()` (via Winsock2) is the available mechanism. On obscure embedded POSIX targets where `poll()` might be absent, `select()` is the fallback. For any Nim program that needs to run on Windows and must use `ioselectors`, this is the backend that makes it possible.

### How the `fds` table works differently here

In the kqueue and poll backends, `fds` is indexed **directly by file descriptor number** — looking up fd 42 is O(1) because you read `fds[42]`. Here, because Windows socket handles can be arbitrary large integers (not small sequential numbers), the `fds` array is used as an **unordered list** scanned linearly. Every lookup — `getKey`, `setSelectKey`, `delKey`, `contains`, `getData`, `setData`, `withData` — is a **linear scan of all `FD_SETSIZE` slots**. This is a fundamental O(n) cost on every operation, not just on waiting.

---

## Types

### `Selector[T]`

```nim
# single-threaded build:
type Selector*[T] = ref SelectorImpl[T]
# thread-safe build (--threads:on):
type Selector*[T] = ptr SelectorImpl[T]
```

The central object. It contains three `FdSet` objects — `rSet`, `wSet`, `eSet` — which are the **master copies** of the read, write, and exception descriptor sets. These are updated incrementally as descriptors are registered and unregistered. When `selectInto` is called, they are copied to temporaries that `select()` can clobber, preserving the master state for the next call.

It also maintains `fds`, an unordered array of `SelectorKey[T]` of fixed size `FD_SETSIZE`, searched linearly for every lookup.

`maxFD` tracks the highest registered file descriptor number seen so far (POSIX only). This value is passed as the first argument to `select()` (`nfds = maxFD + 1`), which tells the kernel how far to scan the bit sets. On Windows, this argument is ignored by `select()`/Winsock, so `maxFD` is not tracked.

In thread-safe builds, all mutations to the sets and `fds` are protected by a `Lock` via `withSelectLock`.

### `SelectEvent`

```nim
type SelectEvent* = ptr SelectEventImpl
# SelectEventImpl = object
#   rsock: SocketHandle   # read socket
#   wsock: SocketHandle   # write socket
```

A manually triggerable event. The field names use `sock` rather than `fd` because on Windows, `select()` only works with sockets — not with arbitrary file descriptors. The implementation of `newSelectEvent` is therefore **entirely different** on Windows vs POSIX.

---

## Lifecycle functions

### `newSelector`

```nim
proc newSelector*[T](): Selector[T]
```

#### What it does

Creates and initialises a fresh selector:

1. In a thread-safe build, allocates the `SelectorImpl` and `fds` array in shared memory and initialises the lock. In a single-threaded build, allocates normally.
2. Marks every `fds` slot as `InvalidIdent`.
3. Zero-initialises all three `FdSet` structures with `FD_ZERO`, clearing every bit.

No kernel object is created — `select()`, like `poll()`, is stateless in the kernel. Everything lives in the user-space `SelectorImpl`.

#### Example

```nim
import std/ioselectors

let sel = newSelector[string]()
defer: sel.close()
```

---

### `close` (Selector)

```nim
proc close*[T](s: Selector[T])
```

#### What it does

In a **thread-safe build**: frees the `fds` shared array, frees the `SelectorImpl` itself, and destroys the lock. Note the order: `deinitLock` is called *after* the memory is freed, which means the lock must not be held at the time of calling.

In a **single-threaded build**: does nothing. The GC handles the `ref`-typed selector. There is no kernel resource to release.

#### Example

```nim
let sel = newSelector[int]()
sel.close()
```

---

### `newSelectEvent` — POSIX

```nim
# POSIX path (non-Windows):
proc newSelectEvent*(): SelectEvent
```

#### What it does (POSIX)

On POSIX, `select()` can monitor any file descriptor, not just sockets. The implementation follows the same approach as the poll backend: creates an OS pipe, sets both ends non-blocking, and stores them as `rsock` and `wsock` (the field names happen to use "sock" for uniformity with the Windows version, but they hold plain pipe fds cast to `SocketHandle`).

---

### `newSelectEvent` — Windows

```nim
# Windows path:
proc newSelectEvent*(): SelectEvent
```

#### What it does (Windows)

On Windows, `select()` only works with **Winsock sockets**, not with pipes or arbitrary file handles. Pipes cannot be added to an `fd_set` on Windows. This forces a more elaborate implementation:

1. Creates a temporary **listening socket** (`ssock`) bound to `127.0.0.1:0` (OS assigns the port).
2. Creates a **write socket** (`wsock`) and connects it to the same loopback address and port.
3. Calls `accept()` on the listening socket to get the **read socket** (`rsock`).
4. Closes the listening socket — it was only needed to broker the connection.
5. Sets both `rsock` and `wsock` to non-blocking mode with `ioctlsocket(FIONBIO)`.
6. Allocates a `SelectEventImpl` in shared memory.

The result is a pair of connected loopback TCP sockets. When `trigger` sends data on `wsock`, `rsock` becomes readable, which `select()` can detect normally.

This is more expensive than a pipe (three sockets created, one immediately closed, a full TCP handshake on loopback) but is the only portable mechanism available on Windows without additional APIs.

#### Example (works the same on both platforms)

```nim
let wakeup = newSelectEvent()
defer: wakeup.close()
```

---

### `trigger`

```nim
# POSIX:
proc trigger*(ev: SelectEvent)
# Windows:
proc trigger*(ev: SelectEvent)
```

#### What it does

Writes 8 bytes (`uint64 = 1`) to `ev.wsock`:

- **POSIX**: uses `posix.write` on the pipe's write fd.
- **Windows**: uses `winlean.send` on the write socket.

In both cases, this makes `ev.rsock` readable, which `select()` will report as `POLLIN`-equivalent activity.

Safe to call from any thread.

#### Example

```nim
# From a background thread:
wakeupEvent.trigger()
# The main thread's selectInto/select returns with Event.User.
```

---

### `close` (SelectEvent)

```nim
# POSIX:
proc close*(ev: SelectEvent)
# Windows:
proc close*(ev: SelectEvent)
```

#### What it does

Closes both sockets/fds and frees the shared memory:

- **POSIX**: `posix.close` on both `rsock` and `wsock`.
- **Windows**: `winlean.closesocket` on both.

Always call `unregister` *before* `close`.

---

## Registration functions

### `registerHandle`

```nim
proc registerHandle*[T](s: Selector[T], fd: int | SocketHandle,
                        events: set[Event], data: T)
```

#### What it does

Registers an OS file descriptor or socket with the selector. The lock is held for the entire operation.

1. Calls `setSelectKey` — a **linear scan** of `fds` to find a free slot (`InvalidIdent`) and store the key there. Raises if `fds` is full.
2. On POSIX, updates `maxFD` if `fd > s.maxFD`. On Windows, `maxFD` is not tracked.
3. If `Event.Read` is in `events`: calls `IOFD_SET(fd, rSet)` and increments `count`.
4. If `Event.Write` is in `events`: calls `IOFD_SET(fd, wSet)` **and** `IOFD_SET(fd, eSet)`, incrementing `count`. The exception set (`eSet`) is also armed for write registrations — on Windows, errors on writable sockets are reported via the exception set.

Note: `maxFD` is never decremented on unregister, so it can only grow. This is a minor conservatism: after unregistering a high-numbered fd, `select()` will still scan up to the old `maxFD + 1`, which is harmless but slightly wasteful.

#### Example

```nim
import std/[ioselectors, nativesockets]

let sel = newSelector[string]()
let sock = createNativeSocket()
sock.setBlocking(false)
sel.registerHandle(sock, {Event.Read}, "client-A")
```

---

### `updateHandle`

```nim
proc updateHandle*[T](s: Selector[T], fd: int | SocketHandle,
                      events: set[Event])
```

#### What it does

Changes the events being watched for an already-registered I/O handle. Acquires the lock, calls `getKey` (linear scan) to find the current key, then diffs the old event set against the new one:

- If `Read` was set and is now removed: `IOFD_CLR(fd, rSet)`, decrement `count`.
- If `Write` was set and is now removed: `IOFD_CLR(fd, wSet)`, `IOFD_CLR(fd, eSet)`, decrement `count`.
- If `Read` was not set and is now added: `IOFD_SET(fd, rSet)`, increment `count`.
- If `Write` was not set and is now added: `IOFD_SET(fd, wSet)`, `IOFD_SET(fd, eSet)`, increment `count`.

Forbidden for descriptors registered with Timer, Signal, Process, Vnode, User, Oneshot, or Error events.

#### Example

```nim
sel.registerHandle(sock, {Event.Read}, myData)

# Need to send data:
sel.updateHandle(sock, {Event.Read, Event.Write})

# Write buffer drained:
sel.updateHandle(sock, {Event.Read})
```

---

### `registerEvent`

```nim
proc registerEvent*[T](s: Selector[T], ev: SelectEvent, data: T)
```

#### What it does

Registers a `SelectEvent` (the read socket) with the selector. Stores `{Event.User}` in the key, adds `ev.rsock` to `rSet`, and increments `count`. On POSIX, also updates `maxFD`.

When `selectInto` later sees that `ev.rsock` is readable, it identifies the key as `Event.User` and reads 8 bytes to drain the pipe/socket before reporting `Event.User` to the caller.

#### Example

```nim
let wakeup = newSelectEvent()
sel.registerEvent(wakeup, data = "thread-wakeup")
```

---

## Unregistration

### `unregister` (fd)

```nim
proc unregister*[T](s: Selector[T], fd: SocketHandle | int)
```

#### What it does

Acquires the lock. Calls `getKey` (linear scan) to find the key. Removes the fd from whichever sets it was in:

- If `Read` or `User` was registered: `IOFD_CLR(fd, rSet)`, decrement `count`.
- If `Write` was registered: `IOFD_CLR(fd, wSet)`, `IOFD_CLR(fd, eSet)`, decrement `count`.

Then calls `delKey` (another linear scan) to clear the `fds` slot.

---

### `unregister` (SelectEvent)

```nim
proc unregister*[T](s: Selector[T], ev: SelectEvent)
```

#### What it does

Acquires the lock. Clears `ev.rsock` from `rSet`, decrements `count`, and calls `delKey` to clear the `fds` slot. Always call this before `ev.close()`.

---

## Waiting for events

### `selectInto`

```nim
proc selectInto*[T](s: Selector[T], timeout: int,
                    results: var openArray[ReadyKey]): int
```

#### What it does

The central waiting function. Its mechanics differ from the poll backend in a subtle but important way.

**Step 1 — copy the master sets.** Under the lock, copies `rSet`, `wSet`, `eSet` to local variables `rset`, `wset`, `eset`. This is essential because `select()` *modifies* the sets it receives, clearing bits for non-ready descriptors. If the master sets were passed directly, every call would destroy the registration state.

**Step 2 — call `select()`.** Passes the three copied sets along with `maxFD + 1` (POSIX) and the timeout. On Windows, `EINTR` is not silenced — any error raises immediately.

**Timeout semantics:**

| Value | Meaning |
|-------|---------|
| `0` | Non-blocking check |
| `> 0` | Wait up to this many milliseconds |
| `-1` | Wait indefinitely (passes `nil` as the timeout pointer) |

Note: `select()` uses microsecond resolution (`timeval`), while `poll()` uses milliseconds. The conversion `timeout * 1000` is done for `tv_usec`.

**Step 3 — scan `fds` for results.** After `select()` returns, the code scans `fds[0..FD_SETSIZE)` sequentially, checking each registered entry with `IOFD_ISSET` against the three result sets:

- `IOFD_ISSET(fd, rset)`: if the key is `Event.User`, reads 8 bytes via `recv` to drain the socket/pipe. If `EAGAIN`, someone else already consumed it — skip. Otherwise reports `Event.User`. If a plain read handle, reports `Event.Read`.
- `IOFD_ISSET(fd, wset)`: adds `Event.Write` to the result.
- If `wset` hit and also `IOFD_ISSET(fd, eset)`: adds `Event.Error`.

This scan is **O(`FD_SETSIZE`)** regardless of how many descriptors are actually ready — the code must check every registered entry because `select()` doesn't tell you *which* bit positions it set, only how many total bits remain.

**EINTR handling**: on POSIX, if `select()` returns -1 with `EINTR`, the function returns `0` (not an error). On Windows, any negative return raises immediately.

#### Example

```nim
var ready: array[64, ReadyKey]
while true:
  let n = sel.selectInto(timeout = 1000, ready)
  for i in 0 ..< n:
    if Event.Read in ready[i].events:
      echo "fd ", ready[i].fd, " is readable"
    if Event.Error in ready[i].events:
      echo "fd ", ready[i].fd, " has an error"
```

---

### `select`

```nim
proc select*[T](s: Selector[T], timeout: int): seq[ReadyKey]
```

#### What it does

Convenience wrapper. Allocates a `seq[ReadyKey]` of size `FD_SETSIZE` (not `MAX_POLL_EVENTS` like the other backends — here the result buffer is as large as the fd set itself), calls `selectInto`, and trims the result.

Note: because `FD_SETSIZE` is typically 1024 on POSIX, this allocates more than the 64-slot buffers in the epoll/kqueue/poll backends.

---

### `flush`

```nim
proc flush*[T](s: Selector[T]) = discard
```

#### What it does

A no-op. Exists to satisfy the `ioselectors` API contract — the kqueue backend has a non-trivial `flush` when change caching is disabled. Here there is nothing to flush; `select()` takes its input on every call.

---

## Data access functions

A critical distinction from the other backends: **all data access functions perform linear scans**. There is no O(1) array index by fd number. Every `getData`, `setData`, `contains`, and `withData` call walks up to `FD_SETSIZE` slots. This is the price of accommodating Windows socket handles, which are not guaranteed to be small sequential integers.

### `getData`

```nim
proc getData*[T](s: Selector[T], fd: SocketHandle | int): var T
```

#### What it does

Acquires the lock, scans `fds[0..FD_SETSIZE)` for a slot with `ident == fd`, and returns a mutable reference to its `.data`. If the fd is not found, returns (without a value — this is undefined behaviour if T is not default-initialised). Use `contains` to check first.

#### Example

```nim
sel.registerHandle(sock, {Event.Read}, 0)
sel.getData(sock) += 1
```

---

### `setData`

```nim
proc setData*[T](s: Selector[T], fd: SocketHandle | int, data: T): bool
```

#### What it does

Acquires the lock, scans for the fd, and if found, overwrites `.data`. Returns `true` on success, `false` if not found.

Note: there is a subtle bug in this implementation — the `while` loop increments `i` only after the assignment, but there is no `inc(i)` in the non-matching branch. The loop terminates only via `break` or when `i` reaches `FD_SETSIZE`, meaning on a non-matching entry it will loop forever. In practice this function is safe because the `break` statement is present in the matching branch, but the missing `inc(i)` for the non-matching case makes this a latent bug if the code is ever refactored. This is worth knowing.

#### Example

```nim
discard sel.setData(sock, newState)
```

---

### `contains`

```nim
proc contains*[T](s: Selector[T], fd: SocketHandle | int): bool {.inline.}
```

#### What it does

Acquires the lock and scans `fds` linearly for a slot with `ident == fd`. Returns `true` if found. O(`FD_SETSIZE`).

Unlike the other backends (which are O(1) via direct array indexing), this is a full scan every time.

#### Example

```nim
if sock in sel:
  sel.updateHandle(sock, {Event.Read, Event.Write})
```

---

### `withData` (found branch)

```nim
template withData*[T](s: Selector[T], fd: SocketHandle | int,
                      value, body: untyped)
```

#### What it does

Acquires the lock, scans for `fd`. If found, binds `value` to a pointer to `.data` and executes `body`. If not found, `body` is skipped. Because the lock is held for the duration of `body`, be careful not to call any other selector functions inside `body` (that would deadlock in the thread-safe build).

#### Example

```nim
sel.withData(sock, connPtr):
  connPtr.bytesSent += 512
```

---

### `withData` (found + not-found branches)

```nim
template withData*[T](s: Selector[T], fd: SocketHandle | int,
                      value, body1, body2: untyped)
```

#### What it does

Two-branch version: `body1` runs if found (with `value` bound), `body2` runs if not found. Same lock-held caveat applies.

#### Example

```nim
sel.withData(sock, connPtr):
  connPtr.bytesSent += 512
do:
  echo "fd not registered"
```

---

## Query functions

### `isEmpty`

```nim
template isEmpty*[T](s: Selector[T]): bool
```

Returns `true` when `s.count == 0`. No descriptors are being watched.

---

### `getFd`

```nim
proc getFd*[T](s: Selector[T]): int
```

Always returns `-1`. Like the poll backend, `select()` has no persistent kernel object to expose.

---

## Complete example: cross-platform event loop

```nim
import std/[ioselectors, nativesockets]

type ConnData = object
  id: int

let sel = newSelector[ConnData]()
defer: sel.close()

# Watch a socket for reads
let sock = createNativeSocket()
sock.setBlocking(false)
sel.registerHandle(sock, {Event.Read}, ConnData(id: 1))

# A cross-thread wakeup event (works on both Windows and POSIX)
let wakeup = newSelectEvent()
sel.registerEvent(wakeup, ConnData(id: 0))

while not sel.isEmpty:
  let ready = sel.select(timeout = 2000)
  for key in ready:
    if Event.Read in key.events:
      echo "socket fd ", key.fd, " has data"
    if Event.Write in key.events:
      echo "socket fd ", key.fd, " is writable"
    if Event.Error in key.events:
      echo "socket fd ", key.fd, " has an error"
    if Event.User in key.events:
      echo "woken by trigger()"
      sel.unregister(wakeup)
      wakeup.close()
      sel.unregister(sock)
```

---

## Comparison: select backend vs poll backend vs kqueue backend

| Aspect | select backend | poll backend | kqueue backend |
|--------|---------------|-------------|---------------|
| Platform | Windows + POSIX | POSIX only | BSD only |
| Kernel object | None | None | kqueue fd |
| fd table lookup | O(n) linear scan | O(1) direct index | O(1) direct index |
| `contains` | O(n) linear scan | O(1) | O(1) |
| `getData`/`setData` | O(n) linear scan | O(1) | O(1) |
| Max descriptors | `FD_SETSIZE` (hard) | Process ulimit | Process ulimit |
| FD_SETSIZE typical | 1024 POSIX / 64 Win | No such limit | No such limit |
| `select()` fd sets | Copied per call | Full array passed | Kernel state |
| Result scan after wait | O(`FD_SETSIZE`) | O(`pollcnt`) | O(ready events) |
| Error reporting | Write + exception set | POLLERR flag | EV_ERROR flag |
| `SelectEvent` (POSIX) | Pipe | Pipe or eventfd | Pipe |
| `SelectEvent` (Windows)| Loopback TCP sockets | N/A | N/A |
| Timer/Signal/Process | Not supported | Not supported | Natively supported |
| `getFd()` | Always -1 | Always -1 | kqueue fd |
| `flush()` | No-op | No-op | May flush kevent changes |

---

## Notes and gotchas

- **`FD_SETSIZE` is a hard ceiling.** On most POSIX systems it is 1024 (set at compile time, not runtime). On Windows with Winsock it is 64. If your process opens more file descriptors than this — even if you only register a few in the selector — `select()` itself may exhibit undefined behaviour when a descriptor number exceeds `FD_SETSIZE`. This is a kernel-level restriction, not just a Nim one.
- **Every lookup is O(n).** Unlike poll and kqueue backends where `fds[fd]` is a direct array index, here every `contains`, `getData`, `setData`, `withData`, and `getKey` scans up to `FD_SETSIZE` slots. For large `FD_SETSIZE` values and many short-lived connections, this adds up.
- **The fd sets are copied on every `selectInto` call.** Three full `FdSet` struct copies happen before each `select()` syscall, because `select()` destroys its inputs.
- **`maxFD` only grows, never shrinks.** After unregistering the highest-numbered fd, `select()` continues to receive `maxFD + 1` as `nfds`. This is safe but makes the kernel scan more bits than strictly necessary.
- **On Windows, `EINTR` is not caught.** Any negative return from `ioselect` on Windows raises immediately. On POSIX, `-1` with `EINTR` is silently returned as `0`.
- **On Windows, `select()` ignores the `nfds` argument.** The Windows implementation of `select()` ignores the first argument entirely (it uses socket handles, not fd numbers). `maxFD` is therefore not tracked in the Windows build.
- **`newSelectEvent` on Windows performs a TCP handshake.** Creating a wakeup event on Windows allocates three sockets, performs a `bind`/`listen`/`connect`/`accept` sequence on loopback, then discards the listener. This is expensive relative to a pipe but unavoidable without `eventfd` or `CreateEvent`.
- **Lock held during `body` in `withData`.** The templates acquire the selector lock before executing user code. Do not call other selector methods from inside `body` in thread-safe builds — this will deadlock.
- **`setData` has a latent loop bug.** The missing `inc(i)` in the non-matching iteration makes the loop exit only via `break` on match or by `i` being incremented... which doesn't happen on non-matches. In practice the function works correctly because the only exit on non-match is hitting `FD_SETSIZE`, but this is fragile.
