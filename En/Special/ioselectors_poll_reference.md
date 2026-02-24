# `ioselectors_poll` Module Reference

> **Module:** `std/ioselectors` (POSIX poll backend)  
> **Language:** Nim  
> **Platform:** Any POSIX system where `poll()` is available — Linux, macOS, BSDs, Solaris, Zephyr RTOS, and others. This backend is also the fallback when a more capable backend (epoll, kqueue) is not available.  
> **Purpose:** Implements the `ioselectors` API on top of the POSIX `poll()` system call, enabling I/O event multiplexing — watching multiple file descriptors simultaneously without busy-polling.

---

## Background: what is poll() and how does it differ from kqueue/epoll?

`poll()` is the most portable of the three major Unix I/O multiplexing interfaces. Unlike kqueue (BSD) or epoll (Linux), which maintain state *inside the kernel* and let you add/remove descriptors incrementally, `poll()` works differently:

- You maintain a **flat array of `pollfd` structures** in user space, one entry per watched descriptor.
- On every call to `poll()`, you pass the **entire array** to the kernel. The kernel scans it, marks which entries have activity, and returns.
- The kernel retains **no state** between calls. Every `poll()` invocation is a fresh scan.

This design has important consequences:

| Property | `poll()` | `epoll`/`kqueue` |
|----------|----------|-----------------|
| State location | User space array | Kernel |
| Scalability | O(n) — kernel scans all fds each call | O(1) — kernel reports only ready fds |
| Portability | Universal POSIX | OS-specific |
| Max descriptors | No hard limit (unlike `select`) | No hard limit |
| Event types | Read, Write, Error only | Read, Write, Timer, Signal, Process, Vnode, User |

For small numbers of descriptors (dozens to low hundreds), `poll()` is perfectly adequate. For thousands of simultaneous connections, epoll or kqueue is preferable.

### What event types does this backend support?

Because `poll()` only understands three event conditions (`POLLIN`, `POLLOUT`, `POLLERR`/`POLLHUP`/`POLLNVAL`), this backend only meaningfully supports:

- `Event.Read` — data available to read
- `Event.Write` — write buffer has space
- `Event.User` — manually triggered via `SelectEvent`
- `Event.Error` — error or hangup on the descriptor

Timer, Signal, Process, and Vnode events are **not implemented** in this backend. Attempting to use them would require platform-specific extensions not present in `poll()`.

### Two implementations of `SelectEvent`

The module has a compile-time branch for the backing mechanism of user-triggerable events:

- **Default (pipe-based):** `newSelectEvent` creates an OS pipe. `trigger` writes 8 bytes to the write end; `poll` fires on `POLLIN` at the read end. Two file descriptors are consumed.
- **`eventfd`-based** (when `defined(zephyr)` or `-d:nimPollHasEventFds`):** Uses Linux's `eventfd` syscall instead. A single file descriptor serves as both the read and write end, halving fd consumption. Enabled for Zephyr RTOS automatically, or manually with the define flag.

---

## Types

### `Selector[T]`

```nim
# single-threaded build:
type Selector*[T] = ref SelectorImpl[T]
# thread-safe build (--threads:on):
type Selector*[T] = ptr SelectorImpl[T]
```

The central object. It maintains two parallel arrays:

- `fds: seq/ptr SharedArray[SelectorKey[T]]` — indexed by OS file descriptor number, holds registration metadata and user data. Entries not in use have `ident == InvalidIdent`.
- `pollfds: seq/ptr SharedArray[TPollFd]` — a **compact, densely packed** array of `pollfd` structs that is passed directly to the `poll()` syscall. This array is kept contiguous with no gaps: removing an entry shifts all subsequent entries left.

The two arrays serve different purposes. `fds` enables O(1) lookup by fd number. `pollfds` is the actual argument to `poll()` — it must be dense and contain only active entries, because `poll()` scans every element in it.

`pollcnt` tracks how many entries are currently in `pollfds`. `count` tracks how many descriptors are registered overall (they are identical in this backend, but the field is part of the shared `ioselectors` contract).

In a thread-safe build, `Selector` is a raw `ptr` into shared memory and a `Lock` protects all mutations to `pollfds` and `fds`.

### `SelectEvent`

```nim
type SelectEvent* = ptr SelectEventImpl
# SelectEventImpl = object
#   rfd: cint   # read end (or the single eventfd)
#   wfd: cint   # write end (same as rfd when using eventfd)
```

A manually triggerable notification. On pipe-based systems, `rfd ≠ wfd`. On eventfd-based systems, `rfd == wfd` (a single fd that acts as both source and sink).

Lives in shared memory so it can be safely handed between threads.

---

## Lifecycle functions

### `newSelector`

```nim
proc newSelector*[T](): Selector[T]
```

#### What it does

Creates and initialises a fresh selector. The steps are simpler than the kqueue backend because `poll()` needs no kernel object:

1. Queries the process's maximum allowed file descriptor count via `maxDescriptors()`.
2. Pre-allocates `fds` and `pollfds` to that size. `fds` is indexed by fd number; `pollfds` starts empty (all entries unused).
3. Marks every `fds` entry as `InvalidIdent`.
4. In thread-safe builds: allocates both arrays in shared memory and initialises a `Lock`.

Note that unlike the kqueue backend, **no kernel object is created here**. The `poll()` syscall receives everything it needs per-call, so there is nothing to open or close at the selector level.

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

In a **thread-safe build**, destroys the lock and frees the three shared-memory allocations (`fds`, `pollfds`, the `SelectorImpl` itself).

In a **single-threaded build**, this procedure does nothing. The Nim GC reclaims the `ref`-typed selector automatically when it goes out of scope. There is no kernel resource to release (unlike kqueue's `kqFD`).

This asymmetry means you should still call `close` unconditionally for correctness in mixed codebases and for forward compatibility.

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

Creates a triggerable event. The implementation depends on the platform:

**Standard path (pipe):**  
Creates an OS pipe with `posix.pipe`. Sets both ends non-blocking. Allocates a `SelectEventImpl` in shared memory. `rfd` = read end, `wfd` = write end.

**`eventfd` path** (`defined(zephyr)` or `-d:nimPollHasEventFds`):  
Calls `eventfd(0, O_NONBLOCK)` to create a single file descriptor that supports both reading and writing. Stores the same fd in both `rfd` and `wfd`. This saves one file descriptor compared to the pipe approach, which matters on resource-constrained systems like Zephyr RTOS.

#### Example

```nim
let wakeup = newSelectEvent()
defer: wakeup.close()
```

---

### `trigger`

```nim
proc trigger*(ev: SelectEvent)
```

#### What it does

Signals the event by writing 8 bytes (a `uint64` value of `1`) to `ev.wfd`. On a pipe, this makes `poll()` report `POLLIN` on the read end. On an `eventfd`, writing increments the internal counter, also triggering `POLLIN`.

This call is safe from any thread. It is the standard mechanism for waking a thread blocked in `select`/`selectInto`.

Raises `IOSelectorsError` if the write fails.

#### Example

```nim
# From a worker thread:
wakeupEvent.trigger()

# In the event loop thread, select returns with Event.User:
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

Closes the file descriptor(s) underlying the event:

- **Pipe path**: closes both `rfd` and `wfd` separately.
- **`eventfd` path**: closes `rfd` only (since `rfd == wfd`).

Then frees the shared-memory block. Always call `unregister` on the selector *before* calling `close` on the event.

Raises `IOSelectorsError` if any `close` syscall fails.

---

## Registration functions

### `registerHandle`

```nim
proc registerHandle*[T](s: Selector[T], fd: int | SocketHandle,
                        events: set[Event], data: T)
```

#### What it does

Registers an existing OS file descriptor with the selector. Stores a `SelectorKey` in `fds[fd]` and, if `events` is non-empty, calls `pollAdd` to append a new `pollfd` entry to the end of `pollfds` and increments `pollcnt` and `count`.

Only `Event.Read` and `Event.Write` are meaningful here. Passing other event types stores them in the key metadata but they will never fire, because `poll()` has no way to report them.

Requires that the descriptor is not already registered (`fds[fd].ident == InvalidIdent`). Violating this causes a `doAssert` failure.

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

Changes the events being watched for an already-registered I/O handle. Handles three distinct transitions:

- **`{}` → non-empty**: the fd was registered but not actively watched. Calls `pollAdd` to insert it into `pollfds`.
- **non-empty → non-empty (changed)**: calls `pollUpdate`, which performs a linear scan of `pollfds` to find and update the entry in-place.
- **non-empty → `{}`**: calls `pollRemove`, which finds the entry by linear scan and removes it by shifting subsequent entries left.

Like the kqueue backend, this is only valid for plain I/O handles. It rejects any descriptor registered with Timer, Signal, Process, Vnode, User, Oneshot, or Error event types.

#### Example

```nim
sel.registerHandle(sock, {Event.Read}, myData)

# When we have data to send:
sel.updateHandle(sock, {Event.Read, Event.Write})

# After write buffer drains:
sel.updateHandle(sock, {Event.Read})
```

---

### `registerEvent`

```nim
proc registerEvent*[T](s: Selector[T], ev: SelectEvent, data: T)
```

#### What it does

Registers a `SelectEvent` with the selector. Stores `{Event.User}` in the key's event set in `fds`, then calls `pollAdd` with `{Event.User, Event.Read}` — the `POLLIN` flag is what `poll()` actually monitors; `Event.User` is the Nim-level marker.

When `selectInto` later sees `POLLIN` on this fd, it checks whether `Event.User` is in the key's events and, if so, reads 8 bytes from the pipe/eventfd to drain it before reporting `Event.User` to the caller.

#### Example

```nim
let wakeup = newSelectEvent()
sel.registerEvent(wakeup, data = "wakeup-chan")
```

---

## Unregistration

### `unregister` (fd)

```nim
proc unregister*[T](s: Selector[T], fd: int | SocketHandle)
```

#### What it does

Removes a file descriptor from the selector. Marks `fds[fd].ident = InvalidIdent`, clears the event set, and calls `pollRemove`, which performs a **linear scan** of `pollfds` to find the entry and remove it by shifting all entries after it one position to the left.

This shifting is the O(n) cost of removal in the poll backend, where n is `pollcnt`. In the epoll and kqueue backends, removal is O(1) because the kernel maintains state.

---

### `unregister` (SelectEvent)

```nim
proc unregister*[T](s: Selector[T], ev: SelectEvent)
```

#### What it does

Removes a `SelectEvent` from the selector. Asserts that the event is currently registered and that `Event.User` is in its key. Clears the key and calls `pollRemove` to remove the pipe's read fd from `pollfds`.

---

## Waiting for events

### `selectInto`

```nim
proc selectInto*[T](s: Selector[T], timeout: int,
                    results: var openArray[ReadyKey]): int
```

#### What it does

The heart of the module. Calls `posix.poll()` with the current `pollfds` array and `pollcnt`, blocks until events arrive or `timeout` elapses, then translates the kernel's results into `ReadyKey` values.

**Timeout semantics:**

| Value | Meaning |
|-------|---------|
| `0` | Return immediately (non-blocking check) |
| `> 0` | Wait at most this many milliseconds |
| `-1` | Wait indefinitely |

**After `poll()` returns**, the code scans `pollfds[0..pollcnt)` looking for entries where `revents != 0`. For each such entry:

- `POLLIN` → `Event.Read`. But if the key has `Event.User` set, it reads 8 bytes from the fd to drain the pipe/eventfd counter. If the read gets `EAGAIN`, someone else already consumed the event, so this entry is skipped. Otherwise `Event.User` replaces `Event.Read` in the result.
- `POLLOUT` → `Event.Write`
- `POLLERR`, `POLLHUP`, or `POLLNVAL` → `Event.Error`

After processing an entry, `pollfds[i].revents` is cleared to `0`.

`EINTR` (interrupted by signal) causes `poll()` to return -1; the proc handles this by returning `0` rather than raising, matching POSIX convention.

The scan stops when either all `pollcnt` entries are checked, all `count` ready entries are consumed, or `maxres` (the size of `results`) is reached.

#### A critical difference from epoll/kqueue

In epoll and kqueue, the kernel returns *only the ready descriptors*. In poll, the kernel marks all `revents` fields but returns a count of how many are non-zero. The user-space code must still **scan the entire `pollfds` array** to find which ones fired. This scan is O(`pollcnt`), not O(number of ready events). For large arrays, this is noticeable.

#### Example

```nim
var ready: array[64, ReadyKey]
while true:
  let n = sel.selectInto(timeout = 500, ready)
  for i in 0 ..< n:
    let key = ready[i]
    if Event.Read in key.events:
      echo "fd ", key.fd, " is readable"
    if Event.Error in key.events:
      echo "fd ", key.fd, " has an error"
```

---

### `select`

```nim
proc select*[T](s: Selector[T], timeout: int): seq[ReadyKey]
```

#### What it does

A convenience wrapper that allocates and returns a `seq[ReadyKey]`. Internally pre-allocates `MAX_POLL_EVENTS` (64) slots, calls `selectInto`, and trims the result to the actual count.

Use `selectInto` in hot loops where heap allocations matter. Use `select` for simplicity.

#### Example

```nim
let ready = sel.select(timeout = 1000)
for key in ready:
  echo "fd: ", key.fd, "  events: ", key.events
```

---

## Data access functions

### `getData`

```nim
proc getData*[T](s: Selector[T], fd: SocketHandle | int): var T
```

#### What it does

Returns a mutable reference to the user data `T` stored for the given descriptor in `fds[fd].data`. The reference points directly into the table, so modifications are reflected immediately.

Returns the default value of `T` (zero-initialised) if `fd` is not registered, rather than raising. Use `contains` to check registration first when the default value could be misleading.

#### Example

```nim
sel.registerHandle(sock, {Event.Read}, 0)
sel.getData(sock) += 1   # increment counter
```

---

### `setData`

```nim
proc setData*[T](s: Selector[T], fd: SocketHandle | int, data: T): bool
```

#### What it does

Replaces the stored user data for a registered descriptor. Returns `true` if the descriptor was registered and data was updated; `false` otherwise.

#### Example

```nim
if not sel.setData(sock, newState):
  echo "sock not registered"
```

---

### `withData` (found branch)

```nim
template withData*[T](s: Selector[T], fd: SocketHandle | int,
                      value, body: untyped)
```

#### What it does

Executes `body` with `value` bound to a pointer to the stored user data, if `fd` is registered. If `fd` is not registered, `body` is skipped. This combines the registration check and data access in a single, safe step.

#### Example

```nim
sel.withData(sock, statePtr):
  statePtr.bytesRead += n
```

---

### `withData` (found + not-found branches)

```nim
template withData*[T](s: Selector[T], fd: SocketHandle | int,
                      value, body1, body2: untyped)
```

#### What it does

Two-branch version: `body1` runs if `fd` is registered; `body2` runs otherwise. Equivalent to an `if/else` around `contains`, but cleaner because the data pointer is already available in `body1`.

#### Example

```nim
sel.withData(sock, statePtr):
  statePtr.bytesRead += n
do:
  echo "fd not in selector"
```

---

## Query functions

### `isEmpty`

```nim
template isEmpty*[T](s: Selector[T]): bool
```

#### What it does

Returns `true` when `s.count == 0` — no descriptors are actively registered. A common event-loop termination condition: if nothing is being watched, there is no point calling `select`.

#### Example

```nim
while not sel.isEmpty:
  let ready = sel.select(timeout = -1)
  # handle events...
```

---

### `contains`

```nim
proc contains*[T](s: Selector[T], fd: SocketHandle | int): bool {.inline.}
```

#### What it does

Returns `true` if `fd` is currently registered. Checks `fds[fd.int].ident != InvalidIdent` — O(1).

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

Always returns `-1`.

Unlike kqueue (which has a real kernel queue fd) or epoll (which has an epoll instance fd), `poll()` has no persistent kernel object. There is nothing to return. The `-1` value signals "not applicable" and distinguishes this backend from the others if you ever need to branch on it.

#### Example

```nim
let fd = sel.getFd()
assert fd == -1  # always true for the poll backend
```

---

## Internal templates (not public, but worth knowing)

### `pollAdd`

Appends a new `pollfd` entry at position `pollfds[pollcnt]` and increments both `pollcnt` and `count`. Protected by `withPollLock` in thread-safe builds. Called by `registerHandle` and `registerEvent`.

### `pollUpdate`

Performs a **linear scan** of `pollfds[0..pollcnt)` to find the entry with the matching `fd`, then updates its `events` field in-place. Called by `updateHandle` when switching between non-empty event sets.

### `pollRemove`

Performs a **linear scan** to find the entry with matching `fd`. Removes it by:
- If it's the last entry: zeroes the slot.
- Otherwise: shifts all subsequent entries one position to the left, keeping the array dense.

Decrements both `pollcnt` and `count`. Called by `updateHandle` (when new events is `{}`) and `unregister`.

The shifting behaviour is what keeps `pollfds` compact, which is required because `poll()` is told to scan exactly `pollcnt` entries — any gap would waste a syscall iteration.

---

## Complete example: a minimal event loop

```nim
import std/[ioselectors, nativesockets, posix]

type ConnData = object
  id: int

let sel = newSelector[ConnData]()
defer: sel.close()

# Watch stdin for input
sel.registerHandle(0, {Event.Read}, ConnData(id: 0))

# Manually triggerable wakeup from another thread
let wakeup = newSelectEvent()
sel.registerEvent(wakeup, ConnData(id: -1))

# Spawn a thread that triggers wakeup after 1 second
# (not shown for brevity)

while not sel.isEmpty:
  let ready = sel.select(timeout = 2000)
  if ready.len == 0:
    echo "timeout — nothing ready"
    break
  for key in ready:
    if Event.Read in key.events and key.fd == 0:
      echo "stdin has data"
      sel.unregister(0)
    if Event.User in key.events:
      echo "woken by another thread"
      sel.unregister(wakeup)
      wakeup.close()
```

---

## Comparison: poll backend vs kqueue backend

| Aspect | poll backend | kqueue backend |
|--------|-------------|---------------|
| Kernel object | None | kqueue fd (one per Selector) |
| `close` (Selector) | No-op in single-threaded build | Must close kqFD and dummy socket |
| Registration | Appends to user-space array | Sends `EV_ADD` kevent to kernel |
| Removal | Linear scan + left-shift in array | Sends `EV_DELETE` kevent to kernel |
| Update | Linear scan + in-place update | Sends `EV_ADD` (new events) + `EV_DELETE` (old events) |
| Waiting | Passes full array every call | Kernel stores state, reports only ready fds |
| Scalability | O(n) per poll call | O(ready events) per kevent call |
| Event types | Read, Write, User, Error | Read, Write, User, Timer, Signal, Process, Vnode, Error |
| `getFd()` | Always `-1` | Returns kqueue fd |
| Timer/Signal/Process | Not supported | Natively supported via EVFILT_* |
| `SelectEvent` backing | Pipe or `eventfd` | Pipe only |
| Thread safety | Lock around `pollfds` mutations | Lock around kevent change buffer |

---

## Notes and gotchas

- **poll() passes the full array every call.** Even if only one descriptor out of a hundred is ready, the kernel still scans all hundred. For large numbers of connections, prefer the epoll or kqueue backend.
- **`pollRemove` shifts entries.** Removing a descriptor from the middle of `pollfds` is O(n) in `pollcnt`. In contrast, the `fds` array is never compacted — entries are just marked `InvalidIdent` in place.
- **No timer, signal, or process events.** These are fundamental `poll()` limitations. If you need them, the epoll (Linux) or kqueue (BSD) backend is required.
- **`getFd()` returns `-1`.** This is intentional and documents the fact that the poll backend has no persistent kernel object. Code that needs to nest selectors or expose the selector fd externally cannot use this backend.
- **The `count` and `pollcnt` fields are always equal.** Unlike backends where registered-but-inactive descriptors can exist, every registered handle in the poll backend has a corresponding `pollfds` entry.
- **`EINTR` is silently handled.** A `poll()` interrupted by a signal returns `0` events instead of raising. This is correct — handle the signal through normal channels.
- **Thread safety covers only `pollfds` mutations.** The `withPollLock` template protects `pollAdd`, `pollUpdate`, and `pollRemove`, as well as the `selectInto` wait. It does not protect `fds` entries independently; the design assumes that `fds` mutations happen in coordination with `pollfds` mutations.
- **`eventfd` mode halves fd usage.** On Zephyr or with `-d:nimPollHasEventFds`, each `SelectEvent` uses one fd instead of two. The `close` implementation accounts for this by conditionally skipping the second `close` call.
