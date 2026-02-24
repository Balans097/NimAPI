# exitprocs — Module Reference

## Overview

The `std/exitprocs` module lets you register **cleanup callbacks that run automatically when the program exits**. This is the Nim equivalent of C's `atexit` facility, with added support for ordinary Nim closures alongside raw no-calling-convention callbacks, and with companion utilities for reading and setting the process exit code.

All registered callbacks are called in **last-in, first-out (LIFO) order** — the last registered procedure runs first. Registration is protected by a lock, making it safe to call `addExitProc` from multiple threads. The actual dispatch also runs under the same lock.

**Platform coverage:**

| Target | Hook mechanism |
|---|---|
| Native (Linux, macOS, Windows, …) | C standard library `atexit()` |
| Node.js | `process.on('exit', …)` |
| Browser JS | `window.onbeforeunload` |

**Important limitation.** When a program terminates due to an **unhandled exception**, exit handlers are intentionally *not* called automatically by the runtime. If you need them to run even in the exceptional-exit case, you must call them manually (for example by wrapping your `main` body in a `try/finally`).

---

## `addExitProc` — closure overload

```nim
proc addExitProc*(cl: proc() {.closure.})
```

### Description

Registers a **closure procedure** to be called when the program exits. A closure is any ordinary Nim `proc` or lambda that may capture variables from its surrounding scope. This is the overload you will use in the vast majority of cases.

The first time `addExitProc` is called (with either overload), it registers the internal dispatcher `callClosures` with the platform's exit mechanism. Subsequent calls only append to the internal list — no additional platform registration is needed.

Procedures are stored in a sequence and called in **reverse registration order** (LIFO) when the program exits. This mirrors the behaviour of C's `atexit`, and it means that resources acquired later (registered later) are freed before resources acquired earlier (registered earlier) — a natural RAII-like discipline.

### Parameters

| Name | Type | Description |
|---|---|---|
| `cl` | `proc() {.closure.}` | The cleanup procedure to register |

### Examples

**Basic cleanup — closing a file:**

```nim
import std/exitprocs

let f = open("output.log", fmWrite)
addExitProc(proc() =
  f.close()
  echo "Log file closed.")

# f is automatically closed when the program exits, even if
# the code path to an explicit close() is never reached.
```

**Capturing state in a closure — logging elapsed time:**

```nim
import std/exitprocs, std/times

let startTime = now()

addExitProc(proc() =
  let elapsed = now() - startTime
  echo "Total runtime: ", elapsed)

# … rest of the program …
```

**LIFO order — teardown in reverse of setup:**

```nim
import std/exitprocs

addExitProc(proc() = echo "1: connect database")   # registered first
addExitProc(proc() = echo "2: open cache")          # registered second
addExitProc(proc() = echo "3: start server")        # registered third

# On exit, the output will be:
# 3: start server
# 2: open cache
# 1: connect database
# i.e. in reverse order — the server is torn down before the cache,
# the cache before the database.
```

**Multiple registrations — each call adds independently:**

```nim
import std/exitprocs

for i in 1..3:
  let n = i  # capture the current value
  addExitProc(proc() = echo "Cleanup step ", n)

# On exit:
# Cleanup step 3
# Cleanup step 2
# Cleanup step 1
```

---

## `addExitProc` — noconv overload

```nim
proc addExitProc*(cl: proc() {.noconv.})
```

### Description

Registers a procedure with the **`{.noconv.}`** calling convention. This overload exists for interoperability with C code or low-level system callbacks where the calling convention must exactly match the platform ABI — for example, functions obtained via `importc` or callbacks passed through C libraries.

In application code you will almost never need this overload. Prefer the closure overload unless you are specifically working with `noconv` callbacks obtained from a C interface.

The semantics — LIFO order, lock-protected registration, same dispatcher — are identical to the closure overload.

### Parameters

| Name | Type | Description |
|---|---|---|
| `cl` | `proc() {.noconv.}` | A no-calling-convention cleanup procedure to register |

### Example

```nim
import std/exitprocs

proc myCleanup() {.noconv.} =
  # Could be a C-imported function or a low-level shutdown routine
  echo "Low-level shutdown complete."

addExitProc(myCleanup)
```

---

## `getProgramResult`

```nim
proc getProgramResult*(): int
```

**Availability:** Native targets and Node.js only. Not available on NimScript or browser JS.

### Description

Returns the **current exit code** that the process will report to the operating system when it terminates normally. The default exit code is `0` (success). If `setProgramResult` has been called, this procedure returns the value that was set.

This is useful inside exit handlers registered with `addExitProc` when the handler needs to make a decision based on whether the program succeeded or failed — for example, to skip certain cleanup steps on error, or to log a different message depending on the exit status.

### Return value

An `int` holding the current program exit code. `0` means success; any non-zero value conventionally signals an error.

### Examples

**Reading the exit code inside an exit handler:**

```nim
import std/exitprocs

addExitProc(proc() =
  let code = getProgramResult()
  if code == 0:
    echo "Program finished successfully."
  else:
    echo "Program exited with error code: ", code)

# Somewhere later:
setProgramResult(1)   # signal failure
```

**Conditional cleanup based on exit status:**

```nim
import std/exitprocs

var tmpFile = "/tmp/workfile.tmp"

addExitProc(proc() =
  if getProgramResult() != 0:
    echo "Leaving ", tmpFile, " for post-mortem inspection."
  else:
    removeFile(tmpFile)
    echo "Cleaned up temporary file.")
```

---

## `setProgramResult`

```nim
proc setProgramResult*(a: int)
```

**Availability:** Native targets and Node.js only. Not available on NimScript or browser JS.

### Description

Sets the **exit code** that the process will report to the operating system when it terminates. Calling this procedure does not immediately exit the program; it only records the intended exit code. The code is actually delivered when the process ends, either naturally (reaching the end of `main`) or via `quit`.

On native targets this writes to the `programResult` global variable. On Node.js it sets `process.exitCode`. Both mechanisms are honoured by the platform's normal exit path, so exit handlers registered with `addExitProc` still run before the code is delivered.

The conventional value for success is `0`; any non-zero value signals failure. Specific non-zero codes can carry domain-specific meaning (for example, `1` for general error, `2` for misuse of command syntax, `127` for command not found in shell conventions).

### Parameters

| Name | Type | Description |
|---|---|---|
| `a` | `int` | The exit code to set |

### Examples

**Signalling failure without immediately exiting:**

```nim
import std/exitprocs

proc processFile(path: string): bool =
  # … returns false on error …
  false

if not processFile("data.csv"):
  setProgramResult(1)   # mark as failed; cleanup still runs on exit
  # do NOT quit here — let exit handlers run normally
```

**Using exit codes to communicate results to a shell script:**

```nim
import std/exitprocs

addExitProc(proc() =
  echo "Final exit code will be: ", getProgramResult())

let ok = runPipeline()
if not ok:
  setProgramResult(2)   # specific code for pipeline failure
```

**Resetting to success after a recoverable condition:**

```nim
import std/exitprocs

setProgramResult(1)  # tentatively mark as error

if recover():
  setProgramResult(0)  # recovery succeeded — report success after all
```

---

## Interaction between the procedures and unhandled exceptions

A subtle but important aspect of `addExitProc` is its behaviour when the program exits due to an unhandled exception:

```
Normal exit  →  exit handlers ARE called automatically
Unhandled exception  →  exit handlers are NOT called automatically
```

To guarantee that exit handlers run regardless of how the program terminates, wrap the top-level code in `try/finally`:

```nim
import std/exitprocs

var db: Database

addExitProc(proc() = db.close())

try:
  db = openDatabase("app.db")
  runApp()
finally:
  # This runs even on unhandled exceptions, before the process exits.
  # The exit proc will also run on normal exit — so db.close() may
  # run twice; make it idempotent or use a flag.
  discard
```

A cleaner pattern when the handler must run in all cases:

```nim
import std/exitprocs

proc cleanup() =
  echo "Shutting down…"
  # release resources

addExitProc(cleanup)   # handles normal exits

try:
  runApp()
except:
  cleanup()            # handles exceptional exits explicitly
  raise               # re-raise so the runtime still reports the error
```

---

## Thread safety

The internal sequence of registered callbacks is protected by `gFunsLock`. This means:

- Calling `addExitProc` from multiple threads concurrently is safe.
- The dispatcher `callClosures` (which runs at exit) also acquires the lock before iterating and clears the list afterwards, preventing double execution.

However, the *callbacks themselves* are not individually synchronised. If a registered callback accesses shared state, that state must be protected by its own synchronisation mechanism.

---

## Summary table

| Symbol | Available on | Description |
|---|---|---|
| `addExitProc(cl: proc(){.closure.})` | All targets | Register a closure to call on exit |
| `addExitProc(cl: proc(){.noconv.})` | All targets | Register a noconv proc to call on exit |
| `getProgramResult()` | Native + Node.js | Read the current exit code |
| `setProgramResult(a)` | Native + Node.js | Set the exit code (without exiting) |
