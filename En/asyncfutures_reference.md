# `asyncfutures.nim` — Module Reference

> **Import:** `import std/asyncfutures`
>
> **Scope:** Asynchronous programming — low-level Future primitives for coordinating concurrent operations in a single-threaded event loop without blocking the OS thread.

The `asyncfutures` module is the foundational layer of Nim's async ecosystem. It defines the `Future[T]` type — a placeholder for a value that does not yet exist — together with the machinery for completing, failing, and composing Futures. The module does **not** run an event loop itself; that is the job of `asyncdispatch`. Instead it provides the primitives that the dispatcher and user code both build on: creation, completion, callback registration, composition operators (`and`, `or`, `all`), and a pluggable scheduler hook (`callSoon`).

Everything in `asyncdispatch`, the `async`/`await` transformation macros, and any third-party async library for Nim ultimately rests on the types and procedures defined here.

---

## Table of Contents

1. [API Overview](#api-overview)
   - [Types](#types-overview)
   - [Functions & Operators](#functions--operators-overview)
   - [Constants](#constants-overview)
2. [Constants, Types, and Auxiliary Constructs](#constants-types-and-auxiliary-constructs)
   - [`isFutureLoggingEnabled`](#isfutureloggingenabled)
   - [`NimAsyncContinueSuffix`](#nimasynccontinuesuffix)
   - [`FutureBase`](#futurebase)
   - [`Future[T]`](#futuret)
   - [`FutureVar[T]`](#futurevart)
   - [`FutureError`](#futureerror)
   - [`FutureInfo`](#futureinfo-futurelogging-only)
   - [`CallbackList` (internal)](#callbacklist-internal)
   - [`CallbackFunc` (internal)](#callbackfunc-internal)
3. [Function Reference](#function-reference)
   - [`newFuture`](#newfuturet)
   - [`newFutureVar`](#newfuturevart)
   - [`clean`](#cleant)
   - [`complete` — Future[T]](#completet--futuret)
   - [`complete` — Future[void]](#complete--futurevoid)
   - [`complete` — FutureVar (no value)](#complete--futurevar-no-value)
   - [`complete` — FutureVar (with value)](#complete--futurevar-with-value)
   - [`fail`](#failt)
   - [`read`](#readt)
   - [`readError`](#readerrort)
   - [`mget`](#mgett)
   - [`finished`](#finished)
   - [`failed`](#failed)
   - [`asyncCheck`](#asynccheckt)
   - [`addCallback` (no-arg)](#addcallback-no-arg)
   - [`addCallback` (typed)](#addcallback-typed)
   - [`callback=` (no-arg)](#callback-no-arg)
   - [`callback=` (typed)](#callback-typed)
   - [`clearCallbacks`](#clearcallbacks)
   - [`callSoon`](#callsoon)
   - [`getCallSoonProc`](#getcallsoonproc)
   - [`setCallSoonProc`](#setcallsoonproc)
   - [`and`](#and)
   - [`or`](#or)
   - [`all`](#allt)
   - [`getFuturesInProgress`](#getfuturesinprogress-futurelogging-only)
4. [Diagnostics and Debugging](#diagnostics-and-debugging)
5. [Thread Safety](#thread-safety)
6. [Complete Example](#complete-example)

---

## API Overview

### Types Overview

| Type | Kind | Description |
|---|---|---|
| `FutureBase` | `ref object` | Untyped base for all Futures. Holds callbacks, status, error. |
| `Future[T]` | `ref object of FutureBase` | Typed Future holding a value of type `T`. |
| `FutureVar[T]` | `distinct Future[T]` | Reusable Future wrapper to avoid repeated allocations. |
| `FutureError` | `object of Defect` | Raised when a Future is completed more than once. |
| `FutureInfo` | `object` | Debug record (only with `-d:futureLogging`). |

### Functions & Operators Overview

| Procedure | Signature | Description |
|---|---|---|
| `newFuture` | `[T](fromProc = "unspecified"): owned Future[T]` | Create a new unfinished Future. |
| `newFutureVar` | `[T](fromProc = "unspecified"): owned FutureVar[T]` | Create a new reusable FutureVar. |
| `clean` | `[T](future: FutureVar[T])` | Reset a FutureVar so it can be reused. |
| `complete` | `[T](future: Future[T], val: sink T)` | Complete a typed Future with a value. |
| `complete` | `(future: Future[void])` | Complete a void Future. |
| `complete` | `[T](future: FutureVar[T])` | Complete a FutureVar (keeps existing value). |
| `complete` | `[T](future: FutureVar[T], val: sink T)` | Complete a FutureVar with a new value. |
| `fail` | `[T](future: Future[T], error: ref Exception)` | Complete a Future with an error. |
| `read` | `[T](future: Future[T] \| FutureVar[T]): lent T` | Read the value of a finished Future. |
| `readError` | `[T](future: Future[T]): ref Exception` | Return the stored exception without rethrowing. |
| `mget` | `[T](future: FutureVar[T]): var T` | Mutable access to the stored value (no finish check). |
| `finished` | `(future: FutureBase \| FutureVar): bool` | True if the Future has completed (success or failure). |
| `failed` | `(future: FutureBase): bool` | True if the Future completed with an error. |
| `asyncCheck` | `[T](future: Future[T])` | Install a callback that raises if the Future failed. |
| `addCallback` | `(future: FutureBase, cb: proc() {.closure, gcsafe.})` | Append a no-arg callback. |
| `addCallback` | `[T](future: Future[T], cb: proc(future: Future[T]))` | Append a typed callback receiving the Future. |
| `callback=` | `(future: FutureBase, cb: proc())` | Replace all callbacks with one no-arg callback. |
| `callback=` | `[T](future: Future[T], cb: proc(future: Future[T]))` | Replace all callbacks with one typed callback. |
| `clearCallbacks` | `(future: FutureBase)` | Remove all registered callbacks. |
| `callSoon` | `(cbproc: proc() {.gcsafe.})` | Schedule a procedure for the next dispatcher tick. |
| `getCallSoonProc` | `(): proc(cbproc: proc()) {.gcsafe.}` | Get the current `callSoon` implementation. |
| `setCallSoonProc` | `(p: proc(cbproc: proc()) {.gcsafe.})` | Replace the `callSoon` implementation. |
| `and` | `[T,Y](fut1: Future[T], fut2: Future[Y]): Future[void]` | Complete when **both** Futures finish. |
| `or` | `[T,Y](fut1: Future[T], fut2: Future[Y]): Future[void]` | Complete when **either** Future finishes. |
| `all` | `[T](futs: varargs[Future[T]]): auto` | Complete when **all** Futures in a list finish. |
| `getFuturesInProgress` | `(): var Table[FutureInfo, int]` | *(futureLogging)* Live table of in-progress Futures. |

### Constants Overview

| Constant | Type | Value | Description |
|---|---|---|---|
| `isFutureLoggingEnabled` | `bool` | `defined(futureLogging)` | Whether the Future diagnostic log is active. |
| `NimAsyncContinueSuffix` | `string` | `"NimAsyncContinue"` | Internal suffix used by the async macro machinery. |

---

## Constants, Types, and Auxiliary Constructs

### `isFutureLoggingEnabled`

```nim
const isFutureLoggingEnabled* = defined(futureLogging)
```

A compile-time boolean that is `true` when the program is built with `-d:futureLogging`. When true, every `newFuture` call registers the new Future in a thread-local table (`futuresInProgress`), and every `complete`/`fail` call removes it. This lets you inspect which Futures are still pending at any point during execution — useful for hunting Future leaks in long-running servers.

The constant also gates the definition of `FutureInfo`, `getFuturesInProgress`, and the `logFutureStart`/`logFutureFinish` internal helpers. None of those symbols exist at all when the flag is absent.

---

### `NimAsyncContinueSuffix`

```nim
const NimAsyncContinueSuffix* = "NimAsyncContinue"
```

A string sentinel used internally by the `async`/`await` macro transformation. When the compiler rewrites an `async proc` into a state-machine continuation, the generated helper procedures are named with this suffix appended. User code should never construct or match this string directly. It is exported only so that tooling (debuggers, stack-trace formatters) can recognise and filter these generated names.

---

### `FutureBase`

```nim
type
  FutureBase* = ref object of RootObj
    callbacks: CallbackList        # linked list of pending callbacks
    finished: bool                 # true after complete() or fail()
    error*: ref Exception          # non-nil when failed
    errorStackTrace*: string       # stack trace captured at fail() time
    # debug-only fields (absent in -d:release):
    stackTrace: seq[StackTraceEntry]
    id: int
    fromProc: string
```

`FutureBase` is the untyped root of the Future type hierarchy. It exists so that code which does not need to know the value type — callback machinery, the `and`/`or` operators, `asyncCheck`, error introspection — can work with any Future through a single, uniform interface.

**Public fields:**

- `error` — holds the `ref Exception` passed to `fail()`. Reading it is safe at any time; it is `nil` on an unfinished or successfully completed Future.
- `errorStackTrace` — the stack trace string captured when `fail()` was called. The `$` operator on `seq[StackTraceEntry]` formats this, filtering out internal Nim async frames.

**Debug-only fields** (stripped when `-d:release` is set):

- `stackTrace` — the call stack at the moment `newFuture` was called. Printed in `FutureError` messages and the async traceback injected by `injectStacktrace`.
- `id` — a monotonically increasing integer (`currentID`) assigned at creation. Unique per thread per run.
- `fromProc` — the string passed to `newFuture("procName")`. Identifies the owning procedure in error messages.

---

### `Future[T]`

```nim
type
  Future*[T] = ref object of FutureBase
    value: T
```

The primary typed Future. Carries a stored value of type `T` alongside the inherited status, callbacks, and error of `FutureBase`. Because it is a `ref object`, assignment copies the reference (not the object), and passing it around is O(1).

`value` is private; it can be read only through `read()` or `mget()` (the latter for `FutureVar[T]` only). Writing to `value` directly from user code is not possible — it is set exclusively by `complete()`.

When `T` is `void`, `Future[void]` is used for operations that signal completion without producing a value. `complete()` on a `Future[void]` accepts no argument; `read()` on it performs only the error check and returns `void`.

---

### `FutureVar[T]`

```nim
type
  FutureVar*[T] = distinct Future[T]
```

`FutureVar[T]` is a `distinct` wrapper around `Future[T]`. The key difference is that it is explicitly designed for reuse: after calling `complete()`, you call `clean()` to reset the `finished` flag, and then the same heap object can be completed again. This eliminates the allocation and GC pressure of creating a new `Future[T]` for every operation iteration.

The `distinct` type prevents accidental mixing with `Future[T]`. Procedures that accept `FutureVar[T]` — `clean`, `mget`, and the `FutureVar`-specific `complete` overloads — are separate from the `Future[T]` ones.

**Typical lifecycle:**

```
newFutureVar → [loop: mget → fill → complete → read → clean → repeat]
```

`FutureVar` is most useful in tight I/O loops (e.g., reading from a socket in fixed-size chunks) where the overhead of allocation matters.

---

### `FutureError`

```nim
type
  FutureError* = object of Defect
    cause*: FutureBase
```

Raised by the internal `checkFinished` guard when `complete()` or `fail()` is called on an already-finished Future. The `cause` field holds the offending Future so you can inspect its `id`, `fromProc`, and captured `stackTrace`.

The full error message includes:
- The Future's numeric ID and the `fromProc` string.
- The stack trace from the moment `newFuture` was called (creation site).
- The string value if `T` is `string` (to aid debugging).
- The stack trace from the second (erroneous) completion site.

`FutureError` is a `Defect`, not an `Exception`. Defects in Nim represent programming errors that are not expected to be caught with `try/except` in production code.

**This guard is entirely absent in `-d:release` builds.** Double-completion in release mode results in silent corruption of the Future state or, at best, redundant callback invocations.

---

### `FutureInfo` *(futureLogging only)*

```nim
when isFutureLoggingEnabled:
  type
    FutureInfo* = object
      stackTrace*: seq[StackTraceEntry]
      fromProc*: string
```

A composite key used in the `futuresInProgress` table. Two Futures are considered the "same kind" if they share the same creation stack trace and `fromProc` string. The table counts how many Futures of each kind are currently in flight.

`FutureInfo` implements a custom `hash` that combines the hashes of `stackTrace` entries (each hashed from `procname`, `line`, and `filename`) and `fromProc`.

---

### `CallbackList` *(internal)*

```nim
type
  CallbackList = object
    function: CallbackFunc
    next: owned(ref CallbackList)
```

A singly-linked list of callbacks embedded directly in `FutureBase`. The first callback is stored **inline** in the `FutureBase` object (no heap allocation for the common single-callback case). Additional callbacks are heap-allocated via `owned ref`.

When `callbacks.call()` is invoked (inside `complete`/`fail`), it walks the list calling `callSoon(fn)` for each non-nil function, then zeroes out both `function` and `next`. This allows the GC to collect the entire callback chain immediately after it fires, rather than waiting for the Future's own lifetime to end.

The in-order traversal guarantees callbacks are called in the order they were registered (FIFO).

---

### `CallbackFunc` *(internal)*

```nim
type
  CallbackFunc = proc() {.closure, gcsafe.}
```

The concrete type for every callback stored in `CallbackList`. The `{.closure.}` pragma means the procedure may capture variables from its enclosing scope. The `{.gcsafe.}` pragma asserts that captured references are safe to hand to the GC across threads (which matters for the `callSoon` interface when a multi-threaded dispatcher is used). User code never names this type directly; it is inferred from lambda literals passed to `addCallback` or `callback=`.

---

## Function Reference

---

### `newFuture[T]`

```nim
proc newFuture*[T](fromProc: string = "unspecified"): owned(Future[T])
```

Creates and returns a new, unfinished `Future[T]`.

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `fromProc` | `string` | `"unspecified"` | Name of the owning procedure. Appears in debug messages. |

**Returns:** `owned Future[T]` — the caller takes sole ownership.

**Behaviour:**

1. Allocates a new `Future[T]` on the heap (`new(result)` via `setupFutureBase`).
2. Sets `finished = false`.
3. In non-`release` builds: captures the current call stack via `getStackTraceEntries()`, stores it in `stackTrace`, assigns the next `currentID` value, and records `fromProc`.
4. If `isFutureLoggingEnabled`: calls `logFutureStart`, incrementing the counter for this Future's `FutureInfo` key in `futuresInProgress`.

**Implementation notes:** The `setupFutureBase` template (not exported) is the actual implementation. It is a `template` rather than a `proc` to ensure that `getStackTraceEntries()` captures the **caller's** stack, not a stack from inside `newFuture`. This is critical: if it were a `proc`, the stack trace would always point into `newFuture` itself, which would be useless.

The `owned` return type participates in Nim's experimental ownership system. In practice, for most code this means "the caller is responsible for this object's lifetime" — the Future should be stored in a variable or returned, not discarded.

```nim
import std/asyncfutures

let f = newFuture[int]("computeSum")
assert not f.finished
assert f.error == nil

let g = newFuture[string]()   # fromProc defaults to "unspecified"
```

---

### `newFutureVar[T]`

```nim
proc newFutureVar*[T](fromProc = "unspecified"): owned(FutureVar[T])
```

Creates a new `FutureVar[T]` — a reusable Future wrapper.

Internally calls `newFuture[T](fromProc)` and wraps the result in `FutureVar[T]` via a `typeof(result)(fo)` cast. Both the underlying `Future[T]` and the `FutureVar[T]` wrapper share the same heap object; the `distinct` wrapper is zero-cost at runtime.

When `isFutureLoggingEnabled`, `logFutureStart` is called on the underlying `Future[T]` (cast from `FutureVar[T]`).

```nim
import std/asyncfutures

var fv = newFutureVar[seq[byte]]("socket.readChunk")
# fv can be reused many times with clean()
```

---

### `clean[T]`

```nim
proc clean*[T](future: FutureVar[T])
```

Resets a `FutureVar[T]` so it can be completed again.

Sets `Future[T](future).finished = false` and `Future[T](future).error = nil`.

**What is NOT reset:**

- `value` — the previously stored value remains in place. This is intentional: you can inspect the old value after `clean()`, and the next `complete(val)` will simply overwrite it.
- `callbacks` — the callback list is not reset. Any callbacks that were not consumed (because they were registered after the Future completed) remain. In practice, `clean()` is called after all callbacks have already fired and been cleared by `callbacks.call()`, so the list is already empty.
- Debug fields (`stackTrace`, `id`, `fromProc`) — unchanged. The Future retains its original identity for diagnostic purposes.

**Why only `FutureVar`?** Resetting a regular `Future[T]` would be a logic error: any code that holds a reference to it and already observed `finished == true` would silently see it become unfinished again. `FutureVar` is a deliberate, explicit signal that reuse is intended.

```nim
var fv = newFutureVar[int]("example")
fv.complete(1); echo fv.read()   # 1
fv.clean()
assert not fv.finished
fv.complete(2); echo fv.read()   # 2
```

---

### `complete[T]` — `Future[T]`

```nim
proc complete*[T](future: Future[T], val: sink T)
```

Successfully completes `future`, storing `val` as its result.

**Steps (via `completeImpl`):**

1. Calls `checkFinished(future)` — raises `FutureError` in debug builds if already finished.
2. Asserts `future.error == nil` (a sanity guard; should always hold at this point).
3. Stores `val` into `future.value` (using `sink` semantics — ownership is transferred, not copied, when the compiler can arrange it).
4. Sets `future.finished = true`.
5. Calls `future.callbacks.call()` — walks the callback list and schedules each via `callSoon`.
6. If logging: calls `logFutureFinish`, decrementing the counter in `futuresInProgress`.

After `complete`, `future.finished == true`, `future.error == nil`, and `future.value` holds `val`. Any call to `read()` will return (a `lent` reference to) `val`.

```nim
let f = newFuture[string]("greet")
f.complete("hello, world")
assert f.finished and not f.failed
assert f.read() == "hello, world"
```

---

### `complete` — `Future[void]`

```nim
proc complete*(future: Future[void], val = Future[void].default)
```

Completes a `Future[void]`. The `val` parameter is a dummy with a default value and is never used; it exists only to satisfy the `completeImpl` generic. In practice, always call this as `future.complete()` with no argument.

```nim
let f = newFuture[void]("signal")
f.complete()
assert f.finished
```

---

### `complete` — `FutureVar` (no value)

```nim
proc complete*[T](future: FutureVar[T])
```

Completes a `FutureVar[T]` using the value already stored in `mget()`. Calls `checkFinished`, sets `finished = true`, fires callbacks. No value parameter: this overload is meant for the `mget → fill → complete` pattern where you mutate the buffer in place and then signal completion.

```nim
var fv = newFutureVar[array[4, byte]]("readFixed")
fv.mget() = [0xDE'u8, 0xAD, 0xBE, 0xEF]
fv.complete()    # signal that the buffer is ready
```

---

### `complete` — `FutureVar` (with value)

```nim
proc complete*[T](future: FutureVar[T], val: sink T)
```

Completes a `FutureVar[T]`, overwriting its stored value with `val`. Equivalent to `fv.mget() = val; fv.complete()` but in a single call.

```nim
var fv = newFutureVar[string]("line")
fv.complete("first line")
echo fv.read()    # "first line"
fv.clean()
fv.complete("second line")
echo fv.read()    # "second line"
```

---

### `fail[T]`

```nim
proc fail*[T](future: Future[T], error: ref Exception)
```

Completes `future` in the error state.

**Steps:**

1. `checkFinished(future)` — guards against double-completion in debug builds.
2. Sets `future.finished = true`.
3. Sets `future.error = error`.
4. Captures the stack trace:
   - If `getStackTrace(error)` is non-empty (the exception was already raised and caught), that trace is used.
   - Otherwise, `getStackTrace()` from the current call site is used.
   This preserves the original raise site when an exception is caught and re-wrapped, rather than replacing it with the `fail()` call site.
5. Fires callbacks via `future.callbacks.call()`.
6. If logging: calls `logFutureFinish`.

After `fail`, `future.finished == true`, `future.failed == true`, and any call to `read()` will rethrow `future.error` (with an augmented async traceback in debug builds).

```nim
let f = newFuture[int]("readPort")
try:
  raise newException(OSError, "connection refused")
except OSError as e:
  f.fail(e)       # original OSError traceback is preserved

assert f.failed
assert f.error of OSError
```

---

### `read[T]`

```nim
proc read*[T](future: Future[T] | FutureVar[T]): lent T
proc read*(future: Future[void] | FutureVar[void])
```

Returns the value of a finished Future.

**Return type:** `lent T` — a borrowed reference to the internal `value` field, no copy. For `void`, nothing is returned.

**Behaviour:**

- If `future.finished` and `future.error == nil`: returns (a borrow of) `future.value`.
- If `future.finished` and `future.error != nil`: calls `injectStacktrace(future)` in debug builds to append the async traceback to the exception message, then re-raises `future.error`.
- If `not future.finished`: raises `ValueError("Future still in progress.")`.

**`injectStacktrace` detail:** The function checks whether the `"\nAsync traceback:\n"` header is already present in the exception message (to avoid duplication across multiple `read()` calls on the same failed Future). If absent, it appends:
- The formatted async stack trace from `getStackTraceEntries(future.error)`.
- `"Exception message: <original message>\n"` as a footer.

The `readImpl` template is used internally. It uses a `{.cursor.}` local to avoid an extra reference-count operation on the Future.

```nim
let f = newFuture[float]("compute")
f.complete(3.14)
let x: float = f.read()     # lent float — no copy
assert x == 3.14

# Reading an unfinished Future:
let g = newFuture[int]("pending")
try:
  discard g.read()
except ValueError as e:
  assert e.msg == "Future still in progress."
```

---

### `readError[T]`

```nim
proc readError*[T](future: Future[T]): ref Exception
```

Returns the exception stored in `future` without rethrowing it.

Raises `ValueError("No error in future.")` if `future.error == nil`.

This is the "inspect without raise" complement to `read()`. It is useful when you need the exception object itself — to log it, wrap it in another exception, or inspect its type — without triggering the exception-handling machinery.

```nim
let f = newFuture[int]("net")
f.fail(newException(TimeoutError, "read timed out"))

let err = f.readError()
assert err of TimeoutError
echo err.msg     # "read timed out"
# err was NOT raised — we just inspected it
```

---

### `mget[T]`

```nim
proc mget*[T](future: FutureVar[T]): var T
```

Returns a `var` (mutable) reference to the value stored inside a `FutureVar[T]`. Unlike `read()`, it does **not** check `finished`, does not raise on an unfinished Future, and does not re-raise errors. It is a raw accessor to the storage slot.

The typical use is to prepare an output buffer **before** signalling completion, avoiding an extra allocation. Because the returned reference is `var`, you can index into it, pass it to `copyMem`, or assign a slice — all without touching the Future status.

**Only available on `FutureVar[T]`**, not `Future[T]`. This restriction is enforced by the type system: `FutureVar` is a `distinct` type, so `mget` is not reachable through a plain `Future[T]`.

```nim
var fv = newFutureVar[seq[byte]]("recv")
let buf = addr fv.mget()          # take a pointer before completion
# ... OS fills buf with received bytes ...
fv.mget().setLen(bytesRead)       # trim to actual size
fv.complete()
```

---

### `finished`

```nim
proc finished*(future: FutureBase | FutureVar): bool
```

Returns `true` if `future` has finished — either successfully or with an error.

For `FutureVar`, the procedure casts to `FutureBase` before checking, since `FutureVar` is a `distinct` type that does not inherit `FutureBase` methods directly.

**Does not distinguish between success and failure.** Use `failed()` or check `future.error != nil` to tell them apart.

```nim
let f = newFuture[int]("x")
assert not f.finished
f.complete(0)
assert f.finished   # true regardless of error/success
```

---

### `failed`

```nim
proc failed*(future: FutureBase): bool
```

Returns `true` if and only if `future.error != nil`.

**Caution:** Returns `false` for an **unfinished** Future (because `error` is nil until `fail()` is called). Always pair with a `finished` check when the distinction matters:

```nim
if fut.finished and fut.failed:
  handleError(fut.error)
elif fut.finished:
  handleValue(fut.read())
else:
  # still pending
```

---

### `asyncCheck[T]`

```nim
proc asyncCheck*[T](future: Future[T])
```

Installs a callback that re-raises the Future's error if it failed. Used as a safe alternative to `discard` for Futures whose value is intentionally unused.

**Why not `discard`?**

```nim
discard someAsyncProc()   # ← error is silently swallowed forever
asyncCheck someAsyncProc() # ← error surfaces at the next dispatcher tick
```

**Implementation:** Uses `future.callback = asyncCheckCallback` (the assignment form, not `addCallback`), which clears any prior callbacks. The `asyncCheckCallback` closure captures `future` and, when invoked, calls `injectStacktrace(future)` before re-raising. This means the error — together with the full async traceback — propagates to the dispatcher's unhandled-exception handler.

**Do not use `asyncCheck` if you are going to `await` the Future later** — `callback=` clears prior callbacks, including those set by the `await` machinery.

```nim
proc backgroundTask(): Future[void] =
  result = newFuture[void]("backgroundTask")
  result.fail(newException(IOError, "disk full"))

asyncCheck backgroundTask()   # IOError will not be silently lost
```

---

### `addCallback` (no-arg)

```nim
proc addCallback*(future: FutureBase, cb: proc() {.closure, gcsafe.})
```

Appends `cb` to the Future's callback list. If the Future is already finished, `cb` is dispatched immediately via `callSoon(cb)` rather than being added to the list.

`assert cb != nil` guards against nil callbacks; passing nil is a programming error and will raise in all build modes.

**List mechanics (`CallbackList.add`):**

- If `callbacks.function` is nil (empty list): assign `cb` directly — no allocation.
- Else: allocate a new `ref CallbackList` node, set its `function`, and walk to the tail of the chain to append. This O(n) walk is intentional: callback lists are expected to be very short (typically 1 or 2 entries).

After `future.callbacks.call()` fires (inside `complete`/`fail`), the list is cleared (`nil`). Callbacks registered **after** the list fires but before the Future object is collected will be dispatched immediately (the Future is finished at that point).

```nim
let f = newFuture[int]("x")
f.addCallback proc() =
  echo "done (no-arg)"
f.addCallback proc() =
  echo "also done"
f.complete(1)
# Both fire in FIFO order: "done (no-arg)", then "also done"
```

---

### `addCallback` (typed)

```nim
proc addCallback*[T](future: Future[T],
                     cb: proc(future: Future[T]) {.closure, gcsafe.})
```

Convenience overload that wraps `cb` in a no-arg closure:

```nim
future.addCallback(
  proc() = cb(future)
)
```

The inner closure captures `future` by reference. Because `future` is a `ref object`, no copy is made. The typed overload is more convenient than the no-arg form when the callback needs to inspect the Future's value or error:

```nim
f.addCallback proc(fut: Future[string]) =
  if fut.failed:
    echo "error: ", fut.error.msg
  else:
    echo "value: ", fut.read()
```

---

### `callback=` (no-arg)

```nim
proc `callback=`*(future: FutureBase, cb: proc() {.closure, gcsafe.})
```

Replaces **all** existing callbacks with `cb`. Calls `clearCallbacks` first, then `addCallback`. If the Future is already finished, `cb` is dispatched immediately via `callSoon`.

**When to use instead of `addCallback`:** Only when you need to guarantee exactly one callback will run, and you explicitly want to discard any previously registered ones. Used internally by `asyncCheck` and the `and`/`or` operators.

**Warning:** Using `callback=` on a Future that already has `await`-registered callbacks will break the `await` suspension — the continuation will never be called. Do not mix `callback=` with `await` on the same Future.

---

### `callback=` (typed)

```nim
proc `callback=`*[T](future: Future[T],
    cb: proc(future: Future[T]) {.closure, gcsafe.})
```

Typed form of `callback=`. Implemented as:

```nim
future.callback = proc() = cb(future)
```

---

### `clearCallbacks`

```nim
proc clearCallbacks*(future: FutureBase)
```

Sets `callbacks.function = nil` and `callbacks.next = nil`, dropping the entire callback chain. The GC can then collect all allocated callback nodes.

This is a low-level escape hatch. Clearing callbacks on a Future that is about to be awaited will cause the await to hang forever. Use with care.

---

### `callSoon`

```nim
proc callSoon*(cbproc: proc() {.gcsafe.})
```

Schedules `cbproc` for execution "soon" — on the next tick of the active event dispatcher.

**Behaviour:**

- If `callSoonProc` is nil (no dispatcher is running): calls `cbproc()` **synchronously and immediately**. This allows Future machinery (including `complete`, `fail`, and callbacks) to function correctly before `asyncdispatch.runForever` / `waitFor` starts.
- If `callSoonProc` is set: delegates to it. In `asyncdispatch`, this enqueues `cbproc` in the dispatcher's ready queue for execution at the start of the next event-loop iteration.

**Thread-local:** `callSoonProc` is a `{.threadvar.}`. Each OS thread has its own independent copy, enabling per-thread dispatchers.

The indirection through `callSoonProc` is what decouples `asyncfutures` from `asyncdispatch`: the futures module never imports the dispatcher; instead the dispatcher injects itself by calling `setCallSoonProc` at startup.

```nim
callSoon proc() =
  echo "This runs on the next dispatcher tick"
```

---

### `getCallSoonProc`

```nim
proc getCallSoonProc*(): (proc(cbproc: proc()) {.gcsafe.})
```

Returns the current value of `callSoonProc` for the calling thread. Useful for saving and restoring the scheduler (e.g., in test frameworks that swap in a synchronous scheduler).

---

### `setCallSoonProc`

```nim
proc setCallSoonProc*(p: (proc(cbproc: proc()) {.gcsafe.}))
```

Replaces the `callSoon` implementation for the calling thread. Called by `asyncdispatch` when its event loop initialises. Can be called by test code or alternative dispatchers to inject a custom scheduler.

**Example — synchronous test scheduler:**

```nim
import std/asyncfutures
import std/deques

var queue: Deque[proc()]

setCallSoonProc proc(cb: proc()) =
  queue.addLast(cb)

proc runSync() =
  while queue.len > 0:
    queue.popFirst()()

let f = newFuture[int]("test")
f.addCallback proc() = echo "fired: ", f.read()
f.complete(42)
runSync()   # prints "fired: 42"
```

---

### `and`

```nim
proc `and`*[T, Y](fut1: Future[T], fut2: Future[Y]): Future[void]
```

Returns a `Future[void]` that completes when **both** `fut1` and `fut2` have finished.

**Error semantics:** If either Future fails, the returned Future fails with the same error immediately (whichever fails first). The other Future's result is ignored.

**Implementation details:**

Each of `fut1` and `fut2` receives a callback via `future.callback = ...` (**not** `addCallback`). This means any previously set callbacks on `fut1` or `fut2` are **silently discarded**. For Futures that are going to be combined with `and`, do not register callbacks before the `and` call.

Inside each callback, a double-completion guard checks `if not retFuture.finished` before acting. This handles the race where both Futures complete "simultaneously" in the same dispatcher tick — only the first callback to run will complete `retFuture`.

The returned Future does **not** carry values; to access the individual values of `fut1` and `fut2`, read them directly after the combined Future completes.

```nim
let a = newFuture[int]("a")
let b = newFuture[string]("b")

let both = a and b
both.addCallback proc() =
  echo a.read(), " ", b.read()   # "1 hello"

a.complete(1)
b.complete("hello")
```

---

### `or`

```nim
proc `or`*[T, Y](fut1: Future[T], fut2: Future[Y]): Future[void]
```

Returns a `Future[void]` that completes as soon as **either** `fut1` or `fut2` finishes.

**Error semantics:** If the first Future to finish has failed, the returned Future fails with that error. The second Future's outcome is ignored entirely.

**Implementation:**

An internal generic `cb[X]` closure is bound to both Futures. The guard `if not retFuture.finished` ensures only the first completion wins. Like `and`, this uses `future.callback = ...`, which overwrites pre-existing callbacks.

**Classic timeout pattern:**

```nim
import std/asyncfutures, std/asyncdispatch

proc withTimeout[T](fut: Future[T], ms: int): Future[void] =
  result = fut or sleepAsync(ms)
```

```nim
let request = httpGetAsync("https://example.com")
let done = request or sleepAsync(5000)
await done
if request.finished:
  echo request.read()
else:
  echo "timed out"
```

---

### `all[T]`

```nim
proc all*[T](futs: varargs[Future[T]]): auto
```

Returns a Future that completes when every Future in `futs` has finished.

**Return type depends on `T`:**

| `T` | Return type | Value on success |
|---|---|---|
| `void` | `Future[void]` | (none) |
| anything else | `Future[seq[T]]` | Values in the same order as `futs` |

**Empty input:** Returns an immediately-completed Future (`complete()` is called before the procedure returns).

**Error semantics:** As soon as any Future in `futs` fails, the returned Future fails with that error. Already-computed values in `retValues` are discarded.

**Implementation details (non-void branch):**

The index-capture problem: a naive `for i, fut in futs: fut.addCallback proc() = retValues[i] = ...` would have all callbacks closing over the **same** loop variable `i` (its final value). The module solves this with a nested `setCallback(i: int)` procedure:

```nim
for i, fut in futs:
  proc setCallback(i: int) =
    fut.addCallback proc(f: Future[T]) =
      retValues[i] = f.read()   # i is now a fresh copy per iteration
      ...
  setCallback(i)
```

Each call to `setCallback` creates a new stack frame with its own `i`, so each callback closes over a different value.

The `completedFutures` counter is incremented inside each callback. When it reaches `len(retValues)`, `retFuture.complete(retValues)` is called.

```nim
let f1 = newFuture[int]("f1")
let f2 = newFuture[int]("f2")
let f3 = newFuture[int]("f3")

let combined = all(f1, f2, f3)   # Future[seq[int]]
combined.addCallback proc() =
  echo combined.read()   # @[10, 20, 30]

f3.complete(30)
f1.complete(10)
f2.complete(20)          # last one triggers completion
```

---

### `getFuturesInProgress` *(futureLogging only)*

```nim
when isFutureLoggingEnabled:
  proc getFuturesInProgress*(): var Table[FutureInfo, int]
```

Returns a mutable reference to the thread-local `futuresInProgress` table. Each entry maps a `FutureInfo` (creation stack trace + `fromProc`) to the count of currently in-flight Futures with that signature.

Incremented by `logFutureStart` (called from `newFuture`/`newFutureVar`) and decremented by `logFutureFinish` (called from `complete`/`fail`). A count that grows without bound indicates a Future that is never completed — a Future leak.

```nim
# Compile with: nim c -d:futureLogging myapp.nim
when isFutureLoggingEnabled:
  import std/tables
  for info, count in getFuturesInProgress():
    if count > 0:
      echo "[LEAK] ", info.fromProc, " has ", count, " pending Future(s)"
      for entry in info.stackTrace:
        echo "  ", entry.filename, "(", entry.line, ") ", entry.procname
```

---

## Diagnostics and Debugging

### Compilation Flags

| Flag | Effect |
|---|---|
| *(none / debug)* | `checkFinished` active; `stackTrace`, `id`, `fromProc` fields present; full async tracebacks injected. |
| `-d:release` | `checkFinished` removed; debug fields absent; `injectStacktrace` is a no-op; maximum performance. |
| `-d:futureLogging` | Activates `FutureInfo` table tracking. Can be combined with `-d:release`. |
| `-d:nimStackTraceOverride` | Enables external debug symbol resolution in `$`-formatted stack traces. |
| `-d:nimPreviewSlimSystem` | Requires explicit import of `std/objectdollar` (for `StackTraceEntry`) and `std/assertions`. |

### Async Traceback Injection

When a failed Future's error is re-raised by `read()`, the module enriches the exception message with an **async traceback** section in non-release builds:

```
IOError: connection refused
Async traceback:
  mymodule.nim(42) fetchData
  mymodule.nim(78) processRequest
Exception message: connection refused
```

The `injectStacktrace` procedure:

1. Checks if `"\nAsync traceback:\n"` is already in the message (idempotent — calling `read()` twice on the same failed Future does not duplicate the section).
2. Strips the existing message down to the original (pre-injection) text.
3. Calls `$` on `getStackTraceEntries(future.error)` to format the trace, which:
   - Applies `addDebuggingInfo` if `-d:nimStackTraceOverride` is set.
   - Filters duplicate entries via `seenEntries` (a `HashSet`).
   - Filters internal Nim async frames via `isInternal` (entries from `asyncdispatch.nim`, `asyncfutures.nim`, `threadimpl.nim`).
   - Skips entries with negative line numbers (reraise markers).
4. Appends the formatted trace and a footer.

### Stack Trace Formatting

The `$` operator on `seq[StackTraceEntry]` is exported and formats a trace as a multi-line string. Each line has the form:

```
  filename.nim(lineNumber) procName
```

Internal async frames are automatically excluded, so the trace shows only application-level call sites.

---

## Thread Safety

`asyncfutures` is designed for a **single-threaded event loop** model. Key thread-safety considerations:

- **`callSoonProc` is `{.threadvar.}`** — each OS thread has an independent scheduler. Two threads can each have their own dispatcher without interfering.
- **`currentID` is a plain module-level `var`** (not threadvar, only present in non-release). In a multi-threaded program with concurrent Future creation in non-release builds, this counter is not thread-safe. This is acceptable because it is debug-only.
- **`futuresInProgress` is `{.threadvar.}`** — the leak-detection table is per-thread.
- **Accessing the same `Future[T]` from multiple threads concurrently is not safe.** There are no locks or atomic operations in `FutureBase`. The `{.gcsafe.}` annotations on callbacks assert GC safety, not data-race freedom. Keep Futures on the thread that created them.

---

## Complete Example

```nim
import std/asyncfutures

# ── 1. Basic Future lifecycle ────────────────────────────────────────────────
block basicLifecycle:
  let f = newFuture[int]("basicLifecycle")
  assert not f.finished

  f.addCallback proc(fut: Future[int]) =
    echo "value = ", fut.read()   # value = 99

  f.complete(99)
  assert f.finished and not f.failed

# ── 2. Failure and error inspection ─────────────────────────────────────────
block failurePath:
  let f = newFuture[string]("failurePath")
  f.fail(newException(IOError, "disk full"))

  assert f.failed
  assert f.error of IOError
  assert f.readError().msg == "disk full"

  try:
    discard f.read()
  except IOError as e:
    echo "caught: ", e.msg    # caught: disk full

# ── 3. FutureVar — reusable Future ─────────────────────────────────────────
block reusePattern:
  var fv = newFutureVar[seq[byte]]("reusePattern")
  for chunk in 1..3:
    fv.clean()
    fv.mget() = @[byte(chunk * 10)]
    fv.complete()
    echo fv.read()   # @[10], @[20], @[30]

# ── 4. Parallel composition: and ────────────────────────────────────────────
block andCompose:
  let a = newFuture[int]("a")
  let b = newFuture[float]("b")
  let both = a and b
  both.addCallback proc() =
    echo a.read(), " ", b.read()   # 7 3.14
  a.complete(7)
  b.complete(3.14)

# ── 5. Race: or ─────────────────────────────────────────────────────────────
block orRace:
  let fast = newFuture[void]("fast")
  let slow = newFuture[void]("slow")
  let race = fast or slow
  race.addCallback proc() = echo "winner!"
  fast.complete()   # "winner!" fires; slow is ignored

# ── 6. Aggregation: all ─────────────────────────────────────────────────────
block allAggregate:
  let tasks = [newFuture[int]("t0"),
               newFuture[int]("t1"),
               newFuture[int]("t2")]
  let agg = all(tasks)
  agg.addCallback proc() =
    echo agg.read()   # @[100, 200, 300]
  tasks[2].complete(300)
  tasks[0].complete(100)
  tasks[1].complete(200)

# ── 7. asyncCheck — surface errors from discarded Futures ───────────────────
block checkPattern:
  proc riskyOp(): Future[void] =
    result = newFuture[void]("riskyOp")
    result.fail(newException(ValueError, "bad input"))

  asyncCheck riskyOp()
  # Without a running dispatcher, the callback fires immediately.
  # In a real async program the ValueError would propagate to the dispatcher.

# ── 8. Custom callSoon for testing ──────────────────────────────────────────
block customScheduler:
  import std/deques
  var q: Deque[proc()]
  setCallSoonProc proc(cb: proc()) = q.addLast(cb)

  let f = newFuture[int]("custom")
  f.addCallback proc() = echo "scheduled: ", f.read()
  f.complete(42)          # callback is enqueued, not called yet

  while q.len > 0:
    q.popFirst()()        # prints "scheduled: 42"

  # Restore default (nil → immediate execution)
  setCallSoonProc nil
```

---

*Based on the source of `asyncfutures.nim` from the Nim standard library
