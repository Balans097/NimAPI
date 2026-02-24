# `asyncstreams` Module Reference

> **Nim Standard Library — Asynchronous Stream Futures**
> API status: **Unstable** — subject to change.

---

## Overview

`asyncstreams` introduces a single data structure — `FutureStream[T]` — that bridges the gap between **producers** that generate data over time and **consumers** that read it asynchronously.

Think of `FutureStream` as a **typed async queue**: a producer writes values into it one by one (or in batches), and a consumer awaits them from the other end. The stream can be *completed* to signal that no more data will ever arrive, or *failed* to propagate an error to the consumer.

This is different from a plain `Future[T]`, which resolves exactly once with a single value. A `FutureStream[T]` can resolve many times — once per written value — making it the natural building block for:

- HTTP response body streaming
- File downloads chunked over a network
- Real-time data feeds
- Producer-consumer pipelines in async code

```nim
import std/asyncstreams
```

---

## Mental model: lifecycle of a FutureStream

```
newFutureStream()          # create an open, empty stream
        │
        ├─ write(v1) ──►  consumer reads v1 via read()
        ├─ write(v2) ──►  consumer reads v2 via read()
        ├─ write(v3) ──►  consumer reads v3 via read()
        │
        ├─ complete()      # signal "no more data"
        │   OR
        └─ fail(err)       # signal "something went wrong"

After complete() + all queued values read → finished() == true
After fail()                              → failed()   == true
```

---

## Type

### `FutureStream[T]`

```nim
type FutureStream*[T] = ref object
  queue:    Deque[T]           # internal FIFO buffer
  finished: bool               # true once complete() or fail() was called
  cb:       proc () {...}      # internal wakeup callback
  error*:   ref Exception      # non-nil if fail() was called
```

A reference type (garbage-collected). The internal queue is a `Deque[T]`, so values arrive and leave in **FIFO order** — first written is first read. The `error` field is the only public field; all interaction with the stream goes through the procedures described below.

---

## Procedures

### `newFutureStream`

```nim
proc newFutureStream*[T](fromProc = "unspecified"): FutureStream[T]
```

#### What it does

Creates a new, **open** and **empty** `FutureStream[T]`. The stream is ready to accept writes immediately. The optional `fromProc` string labels this stream in debug output — supply the name of the procedure that owns the stream to make error traces easier to read.

The internal callback (`cb`) starts as `nil` and is set automatically by `read` and `callback=` when a consumer registers interest.

#### Example

```nim
import std/[asyncdispatch, asyncstreams]

# A stream of string chunks — e.g., an HTTP body arriving in pieces
let bodyStream = newFutureStream[string]("downloadBody")

# A stream of sensor readings
let sensorStream = newFutureStream[float]("readSensors")
```

---

### `write`

```nim
proc write*[T](future: FutureStream[T], value: T): Future[void]
```

#### What it does

Enqueues `value` at the **back** of the stream's internal queue and immediately wakes up any waiting consumer. Returns a `Future[void]` that is already completed on success, so `await`ing it is optional but conventional.

If the stream has already been `complete`d or `fail`ed, `write` fails the returned future with a `ValueError` — you cannot add data to a closed stream.

#### Important detail

There is currently **no backpressure** mechanism. If the producer writes faster than the consumer reads, the queue grows without bound. A future version of the API is expected to add flow control (see the TODO comment in the source).

#### Example

```nim
proc producer(stream: FutureStream[string]) {.async.} =
  await stream.write("first chunk")
  await stream.write("second chunk")
  await stream.write("third chunk")
  stream.complete()   # signal: no more data

# Writing to a finished stream raises ValueError:
let s = newFutureStream[int]()
s.complete()
let fut = s.write(42)
assert fut.failed   # cannot write after complete()
```

---

### `read`

```nim
proc read*[T](future: FutureStream[T]): owned(Future[(bool, T)])
```

#### What it does

Returns a `Future` that resolves with a `(bool, T)` pair when data becomes available:

- `(true, value)` — a value was successfully dequeued from the front of the stream. The value is removed from the stream's internal queue.
- `(false, default(T))` — the stream was `complete`d and the queue is now empty; there will never be more data.

If the stream has **failed**, the returned future itself is failed with the stream's error — you must handle it with a `try/except` around the `await`.

`read` is designed to be used in a loop: keep reading until you get `false`, then stop. This is the canonical consumption pattern.

#### Example

```nim
proc consumer(stream: FutureStream[string]) {.async.} =
  while true:
    let (hasData, chunk) = await stream.read()
    if not hasData:
      break               # stream completed, no more chunks
    echo "Received: ", chunk

# With error handling:
proc safeConsumer(stream: FutureStream[string]) {.async.} =
  while true:
    try:
      let (hasData, chunk) = await stream.read()
      if not hasData: break
      echo chunk
    except Exception as e:
      echo "Stream error: ", e.msg
      break
```

---

### `complete`

```nim
proc complete*[T](future: FutureStream[T])
```

#### What it does

Signals the **end of data**. After `complete` is called:

- No more `write` calls are allowed (they will fail with `ValueError`).
- Any `read` calls that are still waiting are woken up.
- Once the queue drains completely, `finished()` returns `true`.

Calling `complete` on a stream that already has an error (`fail` was called) will raise an `AssertionDefect` — a stream can only be terminated once.

`complete` does **not** discard data already in the queue. The consumer will still read all previously written values before `read` returns `(false, ...)`.

#### Example

```nim
proc streamNumbers(s: FutureStream[int]) {.async.} =
  for i in 1..5:
    await s.write(i)
  s.complete()  # consumer will receive 1,2,3,4,5 then (false, 0)
```

---

### `fail`

```nim
proc fail*[T](future: FutureStream[T], error: ref Exception)
```

#### What it does

Terminates the stream with an **error**. The error is stored in `future.error` and the stream is marked as finished. Any consumer currently `await`ing a `read` will have their future failed with this error.

Use `fail` to propagate exceptions across async boundaries — for instance, when the underlying network connection drops while you are mid-stream.

Like `complete`, `fail` must only be called once; calling it on an already-finished stream raises `AssertionDefect`.

#### Example

```nim
proc riskyProducer(s: FutureStream[string]) {.async.} =
  try:
    await s.write("partial data")
    raise newException(IOError, "network dropped")
  except IOError as e:
    s.fail(e)   # propagate the error to the consumer

proc consumer(s: FutureStream[string]) {.async.} =
  try:
    while true:
      let (ok, chunk) = await s.read()
      if not ok: break
      echo chunk
  except IOError as e:
    echo "Producer failed: ", e.msg
```

---

### `finished`

```nim
proc finished*[T](future: FutureStream[T]): bool
```

#### What it does

Returns `true` **only when both conditions are true simultaneously**:

1. `complete()` or `fail()` has been called.
2. The internal queue is **empty** (all written data has been read).

This is a stronger guarantee than just checking whether `complete` was called — it tells you that the consumer has caught up and there is nothing left to process.

```nim
let s = newFutureStream[int]()
assert not s.finished   # open and empty — not finished

s.complete()
assert not s.finished   # complete called, but queue might still have data

# After all items are read, finished() becomes true
```

> **Tip:** Inside a `callback=` handler, always check `finished()` to decide whether to keep processing or to stop cleanly.

---

### `failed`

```nim
proc failed*[T](future: FutureStream[T]): bool
```

#### What it does

Returns `true` if `fail` was called on the stream — that is, if `future.error` is non-nil. Does not check whether the queue is empty.

```nim
let s = newFutureStream[string]()
assert not s.failed

s.fail(newException(IOError, "oops"))
assert s.failed
assert s.error.msg == "oops"
```

---

### `len`

```nim
proc len*[T](future: FutureStream[T]): int
```

#### What it does

Returns the number of values currently buffered in the internal queue — data that has been `write`n but not yet `read`. Useful for monitoring queue depth or implementing simple flow control manually.

```nim
let s = newFutureStream[int]()
assert s.len == 0

waitFor s.write(1)
waitFor s.write(2)
assert s.len == 2

discard waitFor s.read()   # reads and removes 1
assert s.len == 1
```

---

### `callback=`

```nim
proc `callback=`*[T](future: FutureStream[T],
    cb: proc (future: FutureStream[T]) {.closure, gcsafe.})
```

#### What it does

Registers a **low-level wakeup callback** that is invoked every time:

- New data is written to the stream (`write` was called).
- The stream is completed or failed (`complete` or `fail` was called).

If the stream already has queued data or is already finished at the time of assignment, `cb` is scheduled to be called immediately via `callSoon`.

The callback receives the `FutureStream` itself as an argument, so it can inspect `finished()`, `failed()`, and `len` to decide what to do.

> **Note:** In most cases you should use `read()` in a loop rather than setting a callback directly. The callback-based API is lower level and is used internally by `read` and by `httpclient`'s body parsing.

#### Example

```nim
import std/[asyncdispatch, asyncstreams]

let s = newFutureStream[int]()

s.callback = proc(fs: FutureStream[int]) =
  if fs.failed:
    echo "Error: ", fs.error.msg
    return
  while fs.len > 0:
    # We can't await here, so we peek at the queue directly via len
    echo "data available, queue depth: ", fs.len

waitFor s.write(10)
waitFor s.write(20)
s.complete()

runForever()  # let callbacks fire
```

---

## Complete usage example

The following example shows a producer and consumer running concurrently on the same event loop, connected by a `FutureStream`.

```nim
import std/[asyncdispatch, asyncstreams]

proc producer(stream: FutureStream[string]) {.async.} =
  let chunks = @["Hello, ", "async ", "world!"]
  for chunk in chunks:
    await stream.write(chunk)
    await sleepAsync(100)   # simulate network delay between chunks
  stream.complete()
  echo "Producer: done."

proc consumer(stream: FutureStream[string]) {.async.} =
  var assembled = ""
  while true:
    let (hasData, chunk) = await stream.read()
    if not hasData:
      break
    assembled.add(chunk)
    echo "Consumer: received '", chunk, "'"
  echo "Consumer: full message = '", assembled, "'"

proc main() {.async.} =
  let stream = newFutureStream[string]("example")
  await all(producer(stream), consumer(stream))

waitFor main()

# Output:
# Consumer: received 'Hello, '
# Consumer: received 'async '
# Consumer: received 'world!'
# Producer: done.
# Consumer: full message = 'Hello, async world!'
```

---

## Error propagation example

```nim
import std/[asyncdispatch, asyncstreams]

proc faultyProducer(stream: FutureStream[int]) {.async.} =
  await stream.write(1)
  await stream.write(2)
  stream.fail(newException(IOError, "connection lost"))

proc consumer(stream: FutureStream[int]) {.async.} =
  try:
    while true:
      let (ok, v) = await stream.read()
      if not ok: break
      echo "Got: ", v
  except IOError as e:
    echo "Stream failed: ", e.msg

proc main() {.async.} =
  let stream = newFutureStream[int]("faultyExample")
  await all(faultyProducer(stream), consumer(stream))

waitFor main()

# Output:
# Got: 1
# Got: 2
# Stream failed: connection lost
```

---

## Procedure summary

| Procedure | Signature | Description |
|-----------|-----------|-------------|
| `newFutureStream` | `(fromProc = "unspecified"): FutureStream[T]` | Create an open, empty stream. |
| `write` | `(future, value: T): Future[void]` | Enqueue a value; fails if stream is closed. |
| `read` | `(future): Future[(bool, T)]` | Dequeue the next value; `false` means stream ended. |
| `complete` | `(future)` | Signal end of data; no more writes allowed. |
| `fail` | `(future, error: ref Exception)` | Terminate with an error. |
| `finished` | `(future): bool` | `true` iff closed **and** queue is empty. |
| `failed` | `(future): bool` | `true` iff `fail` was called. |
| `len` | `(future): int` | Number of values currently buffered. |
| `callback=` | `(future, cb: proc(FutureStream[T]))` | Register a low-level wakeup callback. |

---

## Key invariants

- A stream can be written to until `complete` or `fail` is called.
- `complete` and `fail` are mutually exclusive and each may be called at most once.
- `finished()` is only `true` after closure **and** after the queue drains.
- `read` consumes values in FIFO order.
- There is no built-in backpressure; producers must regulate their own write rate if the consumer is slow.
