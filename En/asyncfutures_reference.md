# Nim `std/asyncfutures` Module Reference

> The foundational building block of Nim's async system. This module defines what a `Future` *is* and all the operations you can perform on one: creating, completing, failing, inspecting, composing, and reacting to futures. It is automatically available whenever you import `std/asyncdispatch`, so you rarely need to import it directly.

---

## Table of Contents

1. [Core concept: what is a Future?](#core-concept-what-is-a-future)
2. [Types](#types)
   - [FutureBase](#futurebase)
   - [Future\[T\]](#futuret)
   - [FutureVar\[T\]](#futurevart)
   - [FutureError](#futureerror)
3. [Creating futures](#creating-futures)
   - [newFuture](#newfuture)
   - [newFutureVar](#newfuturevar)
4. [Completing and failing futures](#completing-and-failing-futures)
   - [complete (Future\[T\])](#complete-futuret)
   - [complete (Future\[void\])](#complete-futurevoid)
   - [complete (FutureVar\[T\])](#complete-futurevart)
   - [fail](#fail)
5. [Reading results](#reading-results)
   - [read](#read)
   - [readError](#readerror)
   - [mget](#mget)
6. [Inspecting state](#inspecting-state)
   - [finished](#finished)
   - [failed](#failed)
7. [Callbacks](#callbacks)
   - [addCallback (untyped)](#addcallback-untyped)
   - [addCallback (typed)](#addcallback-typed)
   - [callback= (setter)](#callback-setter)
   - [clearCallbacks](#clearcallbacks)
8. [Composing futures](#composing-futures)
   - [and operator](#and-operator)
   - [or operator](#or-operator)
   - [all](#all)
9. [Fire-and-forget](#fire-and-forget)
   - [asyncCheck](#asynccheck)
10. [FutureVar utilities](#futurevar-utilities)
    - [clean](#clean)
11. [callSoon infrastructure](#callsoon-infrastructure)
    - [callSoon](#callsoon)
    - [getCallSoonProc](#getcallsoonproc)
    - [setCallSoonProc](#setcallsoonproc)
12. [Future logging](#future-logging)
    - [isFutureLoggingEnabled](#isfutureloggingenabled)
    - [getFuturesInProgress](#getfuturesinprogress)
13. [Complete practical example](#complete-practical-example)
14. [Quick Cheat-Sheet](#quick-cheat-sheet)

---

## Core concept: what is a Future?

A `Future[T]` is a placeholder for a value of type `T` that does not exist yet. It represents an asynchronous operation that is currently in progress and will produce either a result or an error at some later point.

Think of it like a ticket at a restaurant: you hand over your order (start the async operation), receive a numbered ticket (a `Future`), and can come back later to pick up your food (the result). Meanwhile you are free to do other things.

```
State machine of a Future:
─────────────────────────
           ┌──────────┐
           │ pending  │  ← just created, not yet done
           └────┬─────┘
                │
       ┌────────┴────────┐
       ▼                 ▼
  ┌─────────┐       ┌──────────┐
  │completed│       │  failed  │  ← both are "finished"
  └─────────┘       └──────────┘
```

A `Future` is **finished** when it has been either completed (has a value) or failed (has an exception). You check this with `finished` and `failed`. You resolve it with `complete` or `fail`. You read the result with `read` (which will re-raise the exception if the future failed).

---

## Types

### `FutureBase`

```nim
type FutureBase* = ref object of RootObj
  error*:           ref Exception  ## The stored error, if any
  errorStackTrace*: string         ## Stack trace at the time of failure
  # (callbacks, finished, and debug fields are internal)
```

**What it is:** The untyped base class shared by all futures, regardless of their result type. It carries the error, the finished flag, and the callback list. You rarely work with `FutureBase` directly in application code — use the typed `Future[T]` instead. It is most useful in APIs that need to accept or return futures of any type (e.g., a generic timeout utility).

```nim
proc onDone(f: FutureBase) =
  if f.failed:
    echo "Error: ", f.error.msg
  else:
    echo "Succeeded (value type unknown at this level)"
```

---

### `Future[T]`

```nim
type Future*[T] = ref object of FutureBase
  # value: T  (internal)
```

**What it is:** The typed workhorse of the async system. `T` is the type of value the future will eventually hold when completed. `Future[void]` is used for operations that signal completion without producing a value (analogous to a `proc` with no return value).

This is the type you encounter most often — returned by every `async` proc and by most I/O operations in `asyncdispatch`, `asyncfile`, `asyncnet`, etc.

```nim
import std/asyncdispatch

proc fetchNumber(): Future[int] {.async.} =
  await sleepAsync(100)
  return 42

let f: Future[int] = fetchNumber()
```

---

### `FutureVar[T]`

```nim
type FutureVar*[T] = distinct Future[T]
```

**What it is:** A `distinct` wrapper around `Future[T]` designed for **reuse**. A regular `Future[T]` can only be completed once — trying to complete it again is a programming error. `FutureVar[T]` allows you to `clean` it and reuse it for subsequent operations, avoiding repeated heap allocations in hot loops. It is a low-level optimization and rarely needed in everyday code.

Key difference: `FutureVar` has a `mget` proc that lets you inspect the stored value even before the future is finished.

---

### `FutureError`

```nim
type FutureError* = object of Defect
  cause*: FutureBase  ## The Future that caused this error
```

**What it is:** A `Defect` raised when you attempt to complete a future that has already been completed. This indicates a logic error in your program — a future was resolved twice. The `cause` field points to the offending future so you can inspect it in the debugger or error handler.

This only applies in non-release builds; in `--define:release` builds the double-completion check is skipped for performance.

---

## Creating futures

### `newFuture`

```nim
proc newFuture*[T](fromProc: string = "unspecified"): owned(Future[T])
```

**What it does:** Creates and returns a new, unfinished `Future[T]`. The `fromProc` string is a human-readable label (conventionally the name of the proc that owns this future) that appears in error messages and debug output, making it much easier to trace which operation a hanging or failing future came from.

**When to use it:** Whenever you implement a low-level async operation manually — that is, when you cannot simply `await` something else and need to construct a future from scratch, register a callback with the OS, and resolve the future when the callback fires.

```nim
import std/asyncdispatch

proc delay(ms: int): Future[void] =
  var fut = newFuture[void]("delay")
  # After ms milliseconds, complete the future
  addTimer(ms, oneshot = true, cb = proc(fd: AsyncFD): bool =
    fut.complete()
    return true
  )
  return fut

proc main() {.async.} =
  echo "waiting..."
  await delay(500)
  echo "done after 500ms"

waitFor main()
```

---

### `newFutureVar`

```nim
proc newFutureVar*[T](fromProc = "unspecified"): owned(FutureVar[T])
```

**What it does:** Creates a reusable `FutureVar[T]`. Internally allocates a `Future[T]` once, and wraps it. After a `FutureVar` is completed, you can call `clean` on it to reset the finished/error state and reuse the same object for the next operation.

**When to use it:** In performance-critical async loops where you perform the same type of operation many times and want to avoid heap allocation on each iteration. For most application code, plain `newFuture` is clearer and simpler.

```nim
import std/asyncdispatch

var fv = newFutureVar[string]("reusable-reader")

for i in 0..<1000:
  fv.clean()              # reset state, ready to be used again
  fv.complete("chunk-" & $i)
  echo fv.read()          # "chunk-0", "chunk-1", ...
```

---

## Completing and failing futures

These are the operations that transition a future from **pending** to **finished**. They also trigger any registered callbacks.

### `complete` (Future[T])

```nim
proc complete*[T](future: Future[T], val: sink T)
```

**What it does:** Marks `future` as successfully finished and stores `val` as its result. All callbacks registered with `addCallback` are scheduled to run via `callSoon`. Attempting to complete an already-finished future raises `FutureError` (in non-release builds).

**When to use it:** At the point in your code where an async operation succeeds and has a result to deliver.

```nim
import std/asyncdispatch

proc resolveWithAnswer(): Future[int] =
  result = newFuture[int]("resolveWithAnswer")
  result.complete(42)  # immediately resolved
```

---

### `complete` (Future[void])

```nim
proc complete*(future: Future[void])
```

**What it does:** Marks a `Future[void]` as successfully finished. No value is stored — this just signals that the operation completed without error. The same callback scheduling rules apply.

**When to use it:** When your async operation doesn't produce a meaningful return value — e.g., a write operation that just needs to signal it's done.

```nim
import std/asyncdispatch

proc signalDone(): Future[void] =
  result = newFuture[void]("signalDone")
  result.complete()
```

---

### `complete` (FutureVar[T])

```nim
proc complete*[T](future: FutureVar[T])
proc complete*[T](future: FutureVar[T], val: sink T)
```

**What it does:** Two overloads for completing a `FutureVar`. The first (no value) marks it done, relying on the value already having been written to via `mget`. The second stores a new value and marks it done, overwriting any previously stored value. Both fire callbacks.

---

### `fail`

```nim
proc fail*[T](future: Future[T], error: ref Exception)
```

**What it does:** Marks `future` as finished with an error. The `error` exception is stored inside the future and will be re-raised when someone calls `read` on this future (or when the callbacks inspect `future.failed`). The stack trace at the point of failure is also captured and stored.

**When to use it:** When an async operation encounters an unrecoverable error and cannot produce a result.

```nim
import std/asyncdispatch

proc mightFail(value: int): Future[int] =
  result = newFuture[int]("mightFail")
  if value < 0:
    result.fail(newException(ValueError, "Value must be non-negative"))
  else:
    result.complete(value * 2)

proc main() {.async.} =
  try:
    let x = await mightFail(-1)
  except ValueError as e:
    echo "Caught: ", e.msg

waitFor main()
```

---

## Reading results

### `read`

```nim
proc read*[T](future: Future[T] | FutureVar[T]): lent T
proc read*(future: Future[void] | FutureVar[void])
```

**What it does:** Retrieves the value stored inside a finished future. If the future failed, `read` re-raises the stored exception (with an injected async traceback in non-release builds). If the future is not yet finished, `read` raises a `ValueError` — it does **not** block or wait.

**When to use it:** After you have confirmed (via `finished`) that the future is done, or in a callback where you know the future has completed. In `async` procs you normally use `await` instead, which calls `read` automatically.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[string]("example")
  f.complete("hello")

  # Safe to call read() because we know it's finished
  echo f.read()   # "hello"

waitFor main()
```

---

### `readError`

```nim
proc readError*[T](future: Future[T]): ref Exception
```

**What it does:** Retrieves the stored exception from a failed future. If the future has no error (it either succeeded or is still pending), raises a `ValueError`. Unlike `read`, this does not re-raise the error — it returns it as a value so you can inspect it.

**When to use it:** When you have explicitly checked `future.failed` and want to examine or log the exception without re-raising it.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[void]("example")
  f.fail(newException(IOError, "disk full"))

  if f.failed:
    let err = f.readError()
    echo "Error type:    ", err.name
    echo "Error message: ", err.msg

waitFor main()
```

---

### `mget`

```nim
proc mget*[T](future: FutureVar[T]): var T
```

**What it does:** Returns a **mutable** reference to the value currently stored inside a `FutureVar`. Unlike `read`, this does not check whether the future is finished — it gives you direct write access to the internal value even while the future is still pending. This is the mechanism by which you populate a `FutureVar` incrementally before completing it.

**When to use it:** Only with `FutureVar`, in low-level code where you are building up a result in-place (e.g., an async read-into-buffer pattern).

```nim
import std/asyncdispatch

var fv = newFutureVar[seq[int]]("accumulate")
mget(fv).add(1)
mget(fv).add(2)
mget(fv).add(3)
fv.complete()

echo fv.read()   # @[1, 2, 3]
```

---

## Inspecting state

### `finished`

```nim
proc finished*(future: FutureBase | FutureVar): bool
```

**What it does:** Returns `true` if the future has been completed (either successfully or with an error). Returns `false` if the operation is still in progress. Note: `true` does not mean success — you must also check `failed` to distinguish the two outcomes.

**When to use it:** Before calling `read` outside of an `async` proc, or to poll the status of a future in a non-blocking way.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[int]("example")
  echo f.finished   # false — still pending

  f.complete(7)
  echo f.finished   # true — now done
  echo f.failed     # false — completed successfully

waitFor main()
```

---

### `failed`

```nim
proc failed*(future: FutureBase): bool
```

**What it does:** Returns `true` if the future finished with an error (i.e., `fail` was called on it). Returns `false` if the future is still pending or completed successfully.

**Important:** `failed` being `false` does not mean the future is finished — it could still be pending. Always check `finished` first if you need to know whether it's done at all.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[void]("example")
  f.fail(newException(ValueError, "oops"))

  echo f.finished   # true
  echo f.failed     # true — it failed

  let g = newFuture[void]("example2")
  g.complete()

  echo g.finished   # true
  echo g.failed     # false — it succeeded

waitFor main()
```

---

## Callbacks

Callbacks are procedures registered on a future that are scheduled to run when that future finishes. This is the low-level mechanism underlying `await`. Most application code should use `await` directly; callbacks are for building combinators or integrating with non-async code.

### `addCallback` (untyped)

```nim
proc addCallback*(future: FutureBase, cb: proc() {.closure, gcsafe.})
```

**What it does:** Registers `cb` to be called when `future` finishes. Multiple callbacks can be added — they are stored in an internal linked list and all called (via `callSoon`) when the future completes. If the future is **already finished** when you call `addCallback`, `cb` is scheduled immediately via `callSoon`. The callback receives no arguments.

**When to use it:** When you need to react to a future completing but don't have direct access to its type parameter, or when building generic async combinators.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[int]("example")

  f.addCallback proc() =
    echo "Future finished! Failed: ", f.failed

  f.complete(10)  # callback is scheduled here

waitFor main()
```

---

### `addCallback` (typed)

```nim
proc addCallback*[T](future: Future[T], cb: proc(future: Future[T]) {.closure, gcsafe.})
```

**What it does:** Same as the untyped overload, but the callback receives the completed `Future[T]` as its argument, making it easy to call `read` or inspect the result type-safely.

**When to use it:** Prefer this over the untyped overload whenever you know the future's type and want the callback to access the result.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[string]("greet")

  f.addCallback proc(fut: Future[string]) =
    if not fut.failed:
      echo "Result: ", fut.read()   # safe here — fut is finished

  f.complete("Hello!")

waitFor main()
```

---

### `callback=` (setter)

```nim
proc `callback=`*(future: FutureBase, cb: proc() {.closure, gcsafe.})
proc `callback=`*[T](future: Future[T], cb: proc(future: Future[T]) {.closure, gcsafe.})
```

**What it does:** **Replaces** all existing callbacks with a single new one. Internally calls `clearCallbacks` then `addCallback`. If the future is already finished, the new callback is scheduled immediately.

**When to use it:** When you want exactly one callback, replacing any previously registered ones. The module's own documentation recommends preferring `addCallback` or `then` over this setter, as silently discarding existing callbacks can lead to subtle bugs.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let f = newFuture[void]("example")
  f.addCallback proc() = echo "first"

  # This replaces the "first" callback — it will never run
  f.callback = proc() = echo "second"

  f.complete()   # only "second" prints

waitFor main()
```

---

### `clearCallbacks`

```nim
proc clearCallbacks*(future: FutureBase)
```

**What it does:** Removes all callbacks registered on `future`. After this call, completing or failing the future will not invoke any callbacks. This is primarily used internally by `callback=` to implement replacement semantics.

**When to use it:** Rarely in application code. Useful when you want to "detach" a future from its downstream consumers — for example, implementing a cancellation mechanism.

---

## Composing futures

These combinators let you express dependencies between multiple futures in a declarative way, without manual callback wiring.

### `and` operator

```nim
proc `and`*[T, Y](fut1: Future[T], fut2: Future[Y]): Future[void]
```

**What it does:** Returns a new `Future[void]` that completes only when **both** `fut1` and `fut2` have completed. If either future fails, the returned future immediately fails with the same error. The result values of `fut1` and `fut2` are discarded — use `all` if you need them.

**When to use it:** When two independent operations must both finish before you proceed, and you don't need their return values.

```nim
import std/asyncdispatch

proc main() {.async.} =
  let a = sleepAsync(200)
  let b = sleepAsync(300)

  await (a and b)   # waits ~300ms (the longer of the two)
  echo "Both done"

waitFor main()
```

---

### `or` operator

```nim
proc `or`*[T, Y](fut1: Future[T], fut2: Future[Y]): Future[void]
```

**What it does:** Returns a new `Future[void]` that completes as soon as **either** `fut1` or `fut2` completes (whichever finishes first). If the first to complete has an error, the returned future fails with that error. The "winning" future's value is discarded — use this primarily for race or timeout patterns.

**When to use it:** To implement timeouts (`operation or timeoutFuture`), or to take the first available result from two competing sources.

```nim
import std/asyncdispatch

proc withTimeout[T](fut: Future[T], ms: int): Future[void] =
  return fut or sleepAsync(ms)

proc main() {.async.} =
  let op = sleepAsync(1000)
  await withTimeout(op, 200)
  if op.finished:
    echo "completed in time"
  else:
    echo "timed out"

waitFor main()
```

---

### `all`

```nim
proc all*[T](futs: varargs[Future[T]]): auto
```

**What it does:** Returns a future that completes when **all** futures in `futs` complete. The return type depends on `T`:

- If `T` is `void` → returns `Future[void]`
- If `T` is anything else → returns `Future[seq[T]]` containing the results in the same order as the input futures

If any future fails, the returned future immediately fails with that error (already-completed results from other futures are discarded). If `futs` is empty, the returned future is immediately complete.

**When to use it:** When you have a collection of parallel operations of the same type and need all of their results.

```nim
import std/asyncdispatch

proc fetchNumber(n: int): Future[int] {.async.} =
  await sleepAsync(n * 10)
  return n * n

proc main() {.async.} =
  let results = await all(fetchNumber(1), fetchNumber(2), fetchNumber(3))
  echo results   # @[1, 4, 9]

waitFor main()
```

---

## Fire-and-forget

### `asyncCheck`

```nim
proc asyncCheck*[T](future: Future[T])
```

**What it does:** Attaches a callback to `future` that will **raise** the future's error as an unhandled exception if it fails. This makes "fire and forget" async calls safe: instead of silently swallowing errors (what `discard` would do), `asyncCheck` ensures that errors propagate and crash the program, making bugs visible.

**Never use `discard` on a `Future`.** A discarded future's errors vanish silently. Use `asyncCheck` for futures you don't await but still care about completing without error.

**When to use it:** When you launch an async operation that should run to completion in the background, but you don't need its return value and don't want to `await` it.

```nim
import std/asyncdispatch

proc backgroundTask() {.async.} =
  await sleepAsync(100)
  echo "background done"
  # if an exception occurs here, asyncCheck will propagate it

proc main() {.async.} =
  asyncCheck backgroundTask()    # launched in background — no await
  echo "main continues immediately"
  await sleepAsync(200)          # give background task time to finish

waitFor main()
```

---

## FutureVar utilities

### `clean`

```nim
proc clean*[T](future: FutureVar[T])
```

**What it does:** Resets the `finished` flag and clears any stored error on `future`, returning it to a "pending" state. The stored value (if any) is **not** cleared — it remains as-is until overwritten. This allows the same `FutureVar` object to be reused for a new operation without a fresh heap allocation.

**When to use it:** Always call `clean` before reusing a `FutureVar` for a new operation. Failing to do so will cause `checkFinished` to raise `FutureError` because the future is still in a "finished" state from the previous operation.

```nim
import std/asyncdispatch

var fv = newFutureVar[int]("counter")

for i in 1..5:
  fv.clean()          # reset for next use
  fv.complete(i * 10)
  echo fv.read()      # 10, 20, 30, 40, 50
```

---

## callSoon infrastructure

`callSoon` is the bridge between `asyncfutures` and the event loop. When a future completes, its callbacks are not called synchronously — they are *scheduled* via `callSoon` to run on the next tick of the event loop. This prevents deep call stacks and ensures predictable ordering.

### `callSoon`

```nim
proc callSoon*(cbproc: proc() {.gcsafe.})
```

**What it does:** Schedules `cbproc` to be called "soon" — either on the next dispatcher tick if the event loop is running, or immediately if no dispatcher has been set up yet. This deferred-execution guarantee is what prevents async callback chains from causing stack overflows.

**When to use it:** Rarely in application code. Useful when you need to defer execution of some logic until the event loop's next pass, without attaching it to a specific future.

---

### `getCallSoonProc`

```nim
proc getCallSoonProc*(): (proc(cbproc: proc()) {.gcsafe.})
```

**What it does:** Returns the current implementation of `callSoon` — the procedure that the event dispatcher injected. Useful for debugging or for custom async backends that need to inspect the scheduler.

---

### `setCallSoonProc`

```nim
proc setCallSoonProc*(p: (proc(cbproc: proc()) {.gcsafe.}))
```

**What it does:** Replaces the `callSoon` implementation. This is called internally by `asyncdispatch` when its event loop is initialized, wiring the `callSoon` mechanism into the dispatcher's queue. Application code should not call this unless implementing a custom async backend.

---

## Future logging

Future logging is a debugging facility that tracks which futures are currently in progress. Enabled by compiling with `-d:futureLogging`.

### `isFutureLoggingEnabled`

```nim
const isFutureLoggingEnabled* = defined(futureLogging)
```

**What it is:** A compile-time constant that is `true` when future logging is enabled. Use it to conditionally compile logging-aware code.

---

### `getFuturesInProgress`

```nim
proc getFuturesInProgress*(): var Table[FutureInfo, int]
```

**What it does:** Returns a mutable reference to the global table that tracks how many instances of each future (keyed by its stack trace and origin proc name) are currently pending. Available only when compiled with `-d:futureLogging`.

**When to use it:** For diagnosing hangs or memory growth by inspecting which futures are stuck. The table maps `FutureInfo` (stack trace + proc name) to a count of outstanding instances.

```nim
when defined(futureLogging):
  import std/asyncfutures
  let inProgress = getFuturesInProgress()
  for info, count in inProgress:
    if count > 0:
      echo info.fromProc, ": ", count, " pending"
```

---

## Complete practical example

A self-contained example that demonstrates creating, composing, and handling futures at multiple levels:

```nim
import std/asyncdispatch

# Low-level: manually creating and resolving a future
proc resolveAfterDelay(value: int, ms: int): Future[int] =
  var fut = newFuture[int]("resolveAfterDelay")
  addTimer(ms, oneshot = true, cb = proc(fd: AsyncFD): bool =
    fut.complete(value)
    return true
  )
  return fut

# Mid-level: combining futures
proc sumTwoAsync(a, b: int): Future[int] {.async.} =
  # Run both concurrently using `all`
  let results = await all(
    resolveAfterDelay(a, 100),
    resolveAfterDelay(b, 150)
  )
  return results[0] + results[1]

# Error path
proc mayFail(shouldFail: bool): Future[string] =
  result = newFuture[string]("mayFail")
  if shouldFail:
    result.fail(newException(ValueError, "intentional failure"))
  else:
    result.complete("success!")

# Top-level async proc
proc main() {.async.} =
  # Concurrent computation
  let sum = await sumTwoAsync(10, 32)
  echo "10 + 32 = ", sum   # 42

  # Error handling
  let f = mayFail(true)
  echo "failed? ", f.failed   # true
  let err = f.readError()
  echo "error: ", err.msg     # "intentional failure"

  # Composing with or (race)
  let winner = resolveAfterDelay(1, 50) or resolveAfterDelay(2, 200)
  await winner
  echo "race done"

  # Fire-and-forget background task
  asyncCheck (proc() {.async.} =
    await sleepAsync(10)
    echo "background finished"
  )()
  await sleepAsync(50)   # let background task run

waitFor main()
```

---

## Quick Cheat-Sheet

| Task | API |
|---|---|
| Create a new future | `newFuture[T]("procName")` |
| Create a reusable future | `newFutureVar[T]("procName")` |
| Mark future as successful (with value) | `future.complete(value)` |
| Mark future as successful (void) | `future.complete()` |
| Mark future as failed | `future.fail(exception)` |
| Read the result | `future.read()` |
| Read the error | `future.readError()` |
| Mutable access (FutureVar only) | `mget(futureVar)` |
| Check if done (success or error) | `future.finished` |
| Check if done with error | `future.failed` |
| React when done (typed) | `future.addCallback proc(f: Future[T]) = ...` |
| React when done (untyped) | `future.addCallback proc() = ...` |
| Replace all callbacks | `future.callback = proc() = ...` |
| Remove all callbacks | `future.clearCallbacks()` |
| Wait for both | `fut1 and fut2` |
| Wait for either | `fut1 or fut2` |
| Wait for all (with results) | `await all(fut1, fut2, fut3)` |
| Fire-and-forget safely | `asyncCheck future` |
| Reset FutureVar for reuse | `futureVar.clean()` |
| Defer execution to next tick | `callSoon(proc() = ...)` |
