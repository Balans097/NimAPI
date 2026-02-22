# `asyncmacro` Module Reference

> **Module:** `asyncmacro`  
> **Source:** Nim's Runtime Library — `(c) Copyright 2015 Dominik Picheta`  
> **Purpose:** Provides the `async`, `await`, and `multisync` macros that power Nim's asynchronous programming model on top of `asyncdispatch`.

---

## Overview

Nim does not have native coroutine syntax. Instead, asynchronous code is written in normal procedure style and then **transformed at compile time** by the macros in this module. When you write `proc fetchData(): Future[string] {.async.}`, the compiler never actually sees an ordinary procedure — the `async` macro rewrites it into a closure iterator that `asyncdispatch` can resume piece by piece each time an awaited `Future` completes.

Understanding this module helps you:
- Know what `async` and `await` really do (and why certain things don't compile).
- Use `multisync` to write a single procedure body that works in both sync and async contexts.
- Avoid common pitfalls such as `discardable` on async procs, or `await` outside async context.

---

## Exported Symbols

This module exports three public symbols intended for direct use:

| Symbol | Kind | Purpose |
|--------|------|---------|
| `async` | macro | Transforms a procedure into an async-capable closure iterator |
| `await` | template (×2) | Suspends the current async proc until a `Future` resolves |
| `multisync` | macro | Generates both sync and async versions of one procedure |

---

## `async`

### What it does

`async` is a **compile-time macro** that you attach to a procedure as a pragma. It performs a deep source-level transformation: the entire procedure body is rewritten into a **closure iterator** that yields control to the event loop whenever it encounters an `await` expression.

Under the hood, `async` does the following:
1. Verifies the return type is `Future[T]` (or empty, becoming `Future[void]`).
2. Rewrites `return` statements to call `complete()` on the hidden return-`Future`.
3. Wraps the body in a closure iterator that yields `FutureBase` objects.
4. Creates a callback (`createCb`) that re-enters the iterator each time a yielded `Future` finishes.
5. Allocates `newFuture[T]()`, kicks off the iterator, and returns that `Future` to the caller.

The result is that every async proc, from the outside, looks like a normal function that immediately hands back a `Future`. Internally, it is a state machine driven by the event loop.

### Signature

```nim
macro async*(prc: untyped): untyped
```

`async` can be applied to a single procedure, a method, a lambda, or a `do` block. It can also receive a `stmtList` of multiple procedures (when used as a block pragma).

### Rules and constraints

- The return type **must** be `Future[T]` or omitted (treated as `Future[void]`).
- The pragma `discardable` is **not allowed** — futures must be checked via `asyncCheck`, not silently discarded.
- Nested procedure/method definitions inside an async body are left untouched (not made async themselves).
- Compile-time debug output can be enabled with `-d:nimDumpAsync`.

### Examples

#### Basic async procedure

```nim
import asyncdispatch

proc greet(name: string): Future[string] {.async.} =
  # Simulate an I/O delay without blocking the thread
  await sleepAsync(100)
  return "Hello, " & name

# Calling greet() does NOT block; it returns a Future immediately.
let fut = greet("World")
echo waitFor fut   # → "Hello, World"
```

The `async` macro turns `greet` into a closure iterator. The `return "Hello, " & name` line is rewritten to call `complete(retFuture, "Hello, " & name)` on the hidden future before returning `nil` from the iterator.

#### Async procedure returning void

```nim
proc logEvent(msg: string): Future[void] {.async.} =
  await sleepAsync(10)
  echo "[LOG] " & msg

# Equivalent — omitting the return type also gives Future[void]:
proc logEvent2(msg: string) {.async.} =
  await sleepAsync(10)
  echo "[LOG] " & msg
```

#### Applying to a lambda

```nim
let handler = proc (data: string): Future[void] {.async.} =
  await sleepAsync(5)
  echo "received: " & data

asyncCheck handler("ping")
runForever()
```

#### Multiple async procs in one block (stmtList form)

```nim
{.async.}:
  proc fetchA(): Future[string] =
    return "A"
  proc fetchB(): Future[string] =
    return "B"
```

---

## `await`

### What it does

`await` is a **template** (actually two overloaded templates) that suspends the current async procedure until a `Future[T]` has completed, then unwraps and returns its value (or raises its exception).

There are two overloads:

1. **`await[T](f: Future[T]): auto`** — the real overload. Inside an async proc it compiles to a `yield` of the underlying `FutureBase`, which hands control back to the event loop. When the future finishes, the iterator is resumed and `f.read()` is called to retrieve the value (or re-raise the stored exception).

2. **`await(f: typed): untyped`** — a fallback that fires a `static: error` if `f` is not a `Future[T]`, giving a clear compile-time message rather than a confusing type error.

Using `await` outside an `async` proc triggers a compile-time error with a helpful suggestion to use `waitFor` instead.

### Signature

```nim
template await*[T](f: Future[T]): auto
template await*(f: typed): untyped   # fallback — always a compile error
```

### Rules and constraints

- **Must be inside an `async` proc.** Using `await` at the top level or in a plain proc gives: *"Can only 'await' inside a proc marked as 'async'. Use 'waitFor' instead."*
- The argument must be a `Future[T]`. Passing anything else hits the fallback template and fails at compile time.
- Awaiting a `nil` Future raises `AssertionDefect` at runtime (caught by the `createCb` machinery).

### Examples

#### Awaiting a value

```nim
proc download(url: string): Future[string] {.async.} =
  # getContent returns Future[string]
  let body = await getContent(url)
  return body.toUpperAscii()
```

`await getContent(url)` pauses `download` without blocking the OS thread. The event loop runs other tasks until `getContent` completes, then resumes `download` with the result.

#### Chaining awaits

```nim
proc pipeline(): Future[void] {.async.} =
  let raw  = await fetchRaw()       # wait for raw data
  let data = await parseAsync(raw)  # wait for parsing
  await saveAsync(data)             # wait for save
```

Each `await` is a suspension point; the procedure reads as sequential code but executes asynchronously.

#### Awaiting multiple futures concurrently with `asyncdispatch.all`

```nim
proc fetchBoth(): Future[void] {.async.} =
  # Launch both, then await the combined future
  let (a, b) = await all(fetchA(), fetchB())
  echo a, " ", b
```

#### Compile-time error examples

```nim
# Wrong: not inside async
let x = await someAsyncProc()  # ERROR: use waitFor instead

# Wrong: not a Future
proc doStuff() {.async.} =
  let x = await 42  # ERROR: await expects Future[T], got int
```

---

## `multisync`

### What it does

`multisync` solves a common Nim library design problem: you want to provide both a **blocking (synchronous)** and a **non-blocking (asynchronous)** version of the same procedure, but you don't want to maintain two copies of the logic.

The macro accepts a **single procedure** whose parameter types are written as a **type union** using `|` (or `or`), where one branch contains the word `async` (e.g. `Socket | AsyncSocket`). From that one definition it generates **two procs**:

- The **async version** applies the `async` macro and uses the `AsyncSocket`-side types.
- The **sync version** strips `Future[T]` down to plain `T`, replaces `await expr` with just `expr`, and uses the `Socket`-side types.

This means the body is written once, with `await` calls that simply disappear in the sync variant.

### Signature

```nim
macro multisync*(prc: untyped): untyped
```

### Rules and constraints

- Parameter types must use `|`/`or` to separate sync and async variants. The async variant must have `async` somewhere in its type name (case-insensitive normalization is applied).
- The return type should be `Future[T]`; the sync version will return plain `T`.
- `await` inside the body becomes a no-op in the sync version (the value passes through unchanged).

### Examples

#### Classic socket read

```nim
import asyncdispatch, asyncnet, net

proc readLine(socket: Socket | AsyncSocket): Future[string] {.multisync.} =
  # In the sync version:  socket.readLine()
  # In the async version: await socket.readLine()
  result = await socket.readLine()
```

After macro expansion, the compiler sees the equivalent of:

```nim
# Sync version (generated)
proc readLine(socket: Socket): string =
  result = socket.readLine()   # await stripped away

# Async version (generated)
proc readLine(socket: AsyncSocket): Future[string] {.async.} =
  result = await socket.readLine()
```

Callers of the sync version never touch a `Future`; callers of the async version work with `await` normally.

#### Multiple parameters with mixed types

```nim
proc transfer(src: Socket | AsyncSocket,
              dst: Socket | AsyncSocket,
              n: int): Future[void] {.multisync.} =
  let data = await src.recv(n)
  await dst.send(data)
```

Both `src` and `dst` get their types split independently for each generated version.

#### Why not just write two procs?

```nim
# Without multisync — two copies to maintain:
proc doWork(s: Socket): string =
  let line = s.readLine()
  process(line)

proc doWork(s: AsyncSocket): Future[string] {.async.} =
  let line = await s.readLine()
  process(line)  # bug: forgot to update the async copy!

# With multisync — one source of truth:
proc doWork(s: Socket | AsyncSocket): Future[string] {.multisync.} =
  let line = await s.readLine()
  process(line)
```

---

## Internal Symbols (non-exported, for reference)

These are not part of the public API but understanding them explains error messages and stack traces.

| Symbol | Role |
|--------|------|
| `asyncSingleProc` | Core transformation: rewrites one proc into an iterator + callback |
| `createCb` (template) | Generates the re-entry callback that drives the iterator |
| `processBody` | Recursive AST walker that rewrites `return` statements and tracks `try`/`finally` |
| `getFutureVarIdents` | Collects `FutureVar[T]` parameters so they are auto-completed on exit |
| `splitProc` | Splits a `multisync` proc into its sync and async halves |
| `NimAsyncContinueSuffix` | String appended to callback names; recognised by `asyncfutures` for nicer stack traces |

---

## Common Errors and Their Meanings

| Error message | Cause | Fix |
|---|---|---|
| `Expected return type of 'Future' got 'X'` | `async` proc returns a non-Future type | Change return type to `Future[X]` |
| `Cannot make async proc discardable` | `{.discardable.}` on an async proc | Use `asyncCheck` to discard the future |
| `await expects Future[T], got X` | Non-future passed to `await` | Pass a `Future[T]` value |
| `Can only 'await' inside a proc marked as 'async'` | `await` in a non-async context | Mark the proc `{.async.}` or use `waitFor` |
| `Async procedure (X) yielded nil` | `await` received a `nil` Future | Check that the async callee never returns `nil` |

---

## Quick-Reference Card

```nim
# Mark a proc async — returns Future automatically
proc myProc(): Future[int] {.async.} = ...

# Suspend and unwrap inside async
let val = await someAsyncProc()

# Generate sync + async twins from one body
proc myProc(s: Socket | AsyncSocket): Future[string] {.multisync.} =
  result = await s.readLine()
```
