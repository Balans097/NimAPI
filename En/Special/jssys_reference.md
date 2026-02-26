# `jssys.nim` — Module Reference

> **Module:** `jssys` — Nim Runtime Library, JavaScript Backend System Layer  
> **Target:** JavaScript only (`--backend:js` / `--target:js`). This file is not compiled on native targets.  
> **Stability:** Most of the module consists of `compilerproc` symbols — internal helpers called exclusively by Nim's JS code generator. Only the five symbols marked with `*` form the user-facing API documented here.

---

## Overview

`jssys.nim` is the JavaScript-backend counterpart of the native `excpt.nim` / `system.nim` combination. When you compile a Nim program to JavaScript, this module provides:

- The **exception runtime**: raising, re-raising, and catching exceptions; wrapping native JS errors as Nim exceptions.
- **Stack trace** collection and formatting (when `-d:nimStackTrace` is active).
- Low-level **arithmetic** helpers with overflow/division-by-zero checks (called by generated JS code, not directly by users).
- **String and set** primitives adapted for the JavaScript memory model (Nim strings are `Array` of character codes in JS).
- A thin **console bridge** (`log`).

Because JavaScript has no pointer arithmetic, no `setjmp`, and a completely different memory model than C, every primitive in this file is a thin Nim wrapper around either inline JavaScript (via `{.emit: …}`) or a JS built-in imported with `{.importc: …}`.

---

## Architecture note: how exceptions work on the JS backend

On the JS backend there is no `setjmp`/`longjmp`. Exception handling uses native JavaScript `try/catch`. The compiler-generated JS code wraps blocks in `try { … } catch(e) { lastJSError = e; … }`. The global variable `lastJSError` always holds the most recently caught JS-level error object.

When the caught object is a Nim exception (it carries an `m_type` property set by Nim's code generator), `lastJSError` is simply cast to `ref Exception`. When it is a *native* JS `Error` (thrown by the JS environment itself, e.g. a `TypeError`), the module reads its `.message` property and presents it as a string.

This design means that `getCurrentException` and `getCurrentExceptionMsg` work correctly regardless of whether the exception originated in Nim or in JavaScript.

---

## Exported Functions

---

### `log`

```nim
proc log*(s: cstring) {.importc: "console.log", varargs, nodecl.}
```

**What it does:**  
A direct binding to the browser / Node.js `console.log` function. Accepts a `cstring` as the first argument and is declared `varargs`, so additional JavaScript arguments can be passed through when calling from inline JS or generated code.

**Why it is exported:**  
It is the lowest-level output primitive available on the JS backend — `echo` and `writeStackTrace` both ultimately call into `console.log`. Exposing it with `*` lets Nim code in the same compilation unit call `log` directly without going through `echo`'s formatting overhead.

**Practical use:** Useful when you need to print something during early initialisation, before Nim's string machinery is fully set up, or when you deliberately want raw `console.log` semantics (e.g. passing JS objects directly to the browser's DevTools console).

```nim
# Direct console output — no Nim string conversion overhead
log("Hello from Nim")

# Also works from Nim because the proc is varargs at the JS level:
log(cstring"debug value:")
```

> **Note:** For normal program output, prefer `echo`. Use `log` only when you need the raw JS primitive or when you are writing `{.emit:…}` code that needs a guaranteed-available output path.

---

### `getCurrentException`

```nim
proc getCurrentException*(): ref Exception {.compilerRtl, gcsafe.}
```

**What it does:**  
Returns the exception that is *currently being handled* — i.e. the exception that caused the innermost active `except` block to be entered. If the caught object is a genuine Nim exception (it has an `m_type` field), it is cast to `ref Exception` and returned. If no exception is active, returns `nil`.

**JS-specific behaviour:**  
On the JS backend, the "current exception" is not stored in a thread-local linked list as on native targets. Instead, it is read directly from `lastJSError` — the JS-level variable that holds the last caught JavaScript error object. Only objects that carry Nim's `m_type` marker are returned; pure JS `Error` objects are treated as non-Nim exceptions and this function returns `nil` for them (use `getCurrentExceptionMsg` in that case to get their message).

**Standard usage:** This function is implicitly called when you use `getCurrentException()` inside an `except` block. You rarely need to call it directly; the idiomatic way to get the exception object is the `as` clause: `except IOError as e`.

```nim
try:
  riskyOperation()
except:
  let e = getCurrentException()
  if e != nil:
    echo "Nim exception: ", e.msg
  else:
    echo "Native JS exception: ", getCurrentExceptionMsg()
```

```nim
# Useful when writing a generic error logger that works across all exception types:
proc logAnyException() =
  let e = getCurrentException()
  if e != nil:
    echo "[", e.name, "] ", e.msg
  else:
    echo "[JS Error] ", getCurrentExceptionMsg()
```

---

### `getCurrentExceptionMsg`

```nim
proc getCurrentExceptionMsg*(): string
```

**What it does:**  
Returns the human-readable message of the currently active exception as a Nim `string`. It handles both kinds of exceptions that can arise on the JS backend:

1. **Nim exceptions** — returns `e.msg`, the string set when the exception was created with `newException(…, "message")`.
2. **Native JS errors** — reads the `.message` property of the JavaScript `Error` object (e.g. a `TypeError: cannot read property 'x' of undefined` thrown by the JS runtime).

Returns an empty string `""` when no exception is active.

**Why this is more robust than `getCurrentException().msg`:**  
When a JS engine throws a native error (not a Nim-raised exception), `getCurrentException()` returns `nil`, so accessing `.msg` on it would crash. `getCurrentExceptionMsg()` safely covers both paths and always gives you *something* printable.

```nim
try:
  callJSAPIThatMightThrow()
except:
  # Works whether the error is from Nim or from the JS engine itself
  echo "Error: ", getCurrentExceptionMsg()
```

```nim
# Wrapping JS API calls with uniform error handling:
proc safeCall(f: proc()) =
  try:
    f()
  except:
    let msg = getCurrentExceptionMsg()
    reportError(msg)  # send to error tracker
```

---

### `setCurrentException`

```nim
proc setCurrentException*(exc: ref Exception)
```

**What it does:**  
Manually sets the "current exception" to `exc` by casting it to `PJSError` and storing it in `lastJSError`. This makes the given exception visible to subsequent calls to `getCurrentException()` and `getCurrentExceptionMsg()`.

**When you would call this:**  
Rarely in application code. The primary use cases are:

- **Custom exception propagation** across async / coroutine boundaries, where you need to manually ferry an exception object from one execution context to another.
- **Testing** — inject a synthetic exception into the runtime state to test error-handling code paths without actually raising and catching an exception.
- **Interop** — if you receive a `ref Exception` from a callback and want to make it the "current" exception before entering an error-handling routine that calls `getCurrentException()`.

**Important caveat:** This does *not* raise the exception. The exception is merely stored in the runtime's "last caught" slot. It will appear as the current exception to any code that reads `getCurrentException()` until the next `try/catch` cycle overwrites `lastJSError`.

```nim
# Propagate an exception captured in a callback into the calling context:
var savedExc: ref Exception

proc asyncCallback() =
  try:
    doWork()
  except Exception as e:
    savedExc = e  # capture before callback returns

# Later, in the main flow:
if savedExc != nil:
  setCurrentException(savedExc)
  handleCurrentException()
```

```nim
# Unit test: verify that your handler reads the message correctly
let fakeExc = newException(ValueError, "test message")
setCurrentException(fakeExc)
assert getCurrentExceptionMsg() == "test message"
```

---

### `getStackTrace` *(no argument)*

```nim
proc getStackTrace*(): string
```

**What it does:**  
Returns the current call-stack as a formatted multi-line string, representing the sequence of procedure calls that led to the point where `getStackTrace()` was invoked.

**Format** (when stack traces are enabled):
```
Traceback (most recent call last)
myfile.nim(42) at myProc
myfile.nim(10) at main
```

Returns `"No stack traceback available\n"` when stack traces are disabled (i.e. the program was compiled without `-d:nimStackTrace`, which is the default for release builds via `--opt:speed`).

**JS-specific note:** The stack trace is built from Nim's own `CallFrame` linked list, *not* from the JavaScript engine's native `Error.stack`. This means you get Nim source-level location information (file, line, procedure name) rather than JS-level information. On the other hand, it only works when Nim's stack-trace instrumentation is active.

```nim
# Log where we are right now without raising anything:
echo getStackTrace()
```

```nim
# Attach a snapshot of the call stack to a diagnostic report:
proc createReport(msg: string): string =
  result = msg & "\n" & getStackTrace()
```

---

### `getStackTrace` *(with exception)*

```nim
proc getStackTrace*(e: ref Exception): string
```

**What it does:**  
Returns the stack trace that was captured **at the moment exception `e` was raised**, stored inside the exception object as `e.trace`. This is the frozen trace from raise-time, not from the current call site.

Returns an empty string `""` if `e` is `nil` or if no trace was stored (release builds).

**Key distinction from the no-argument variant:**  
- `getStackTrace()` — snapshot of *where you are right now*.  
- `getStackTrace(e)` — snapshot of *where the exception was thrown*.

This distinction matters whenever you call `getStackTrace` from a `finally` block, an error-reporting routine, or any place that is many stack frames removed from the original `raise` site.

```nim
try:
  riskyOperation()
except IOError as e:
  # Print where the error *originated*, not where we caught it:
  echo getStackTrace(e)
```

```nim
# Structured error reporting — attach the original raise location:
proc formatError(e: ref Exception): string =
  result = "Error: " & e.msg & "\n"
  let trace = getStackTrace(e)
  if trace.len > 0:
    result &= "Stack trace:\n" & trace
```

---

## `compilerproc` Symbols — Overview

The table below lists the internal helpers that are *not* part of the user-facing API but are referenced by names visible in generated JS output. Understanding them helps when debugging generated JavaScript.

| Symbol | Purpose |
|---|---|
| `raiseException` | Called by generated code on every `raise` statement |
| `reraiseException` | Called by generated code on bare `raise` (re-raise) |
| `raiseOverflow` | Raised when integer arithmetic overflows |
| `raiseDivByZero` | Raised on integer division or modulo by zero |
| `raiseRangeError` | Raised on value-out-of-range (subrange types) |
| `raiseIndexError` | Raised on out-of-bounds array/seq access |
| `raiseFieldError2` | Raised on invalid discriminant field access in variant objects |
| `addInt` / `subInt` / `mulInt` / `divInt` / `modInt` | Checked 32-bit integer arithmetic |
| `addInt64` … `modInt64` | Checked 64-bit integer arithmetic (uses JS `BigInt`) |
| `cmpStrings` / `eqStrings` | String comparison (Nim strings are `Array` of char codes in JS) |
| `nimCopy` | Deep value-copy semantics for Nim's value types |
| `cstrToNimstr` / `toJSStr` | Conversion between JS native strings and Nim's internal char-array format |
| `SetCard`, `SetEq`, `SetLe`, `SetLt`, `SetMul`, `SetPlus`, `SetMinus`, `SetXor` | Set operations on Nim `set` types (represented as JS objects) |
| `nimParseBiggestFloat` | Float parsing with full Nim semantics (NaN, Inf, underscores, sign) |
| `chckIndx` / `chckRange` / `chckObj` / `isObj` | Runtime bounds and type checks |

---

## Quick Cheat-Sheet

| Symbol | Kind | When to use |
|---|---|---|
| `log` | proc | Raw `console.log` — low-level or early-init output |
| `getCurrentException` | proc | Get the active Nim exception object inside `except` |
| `getCurrentExceptionMsg` | proc | Get the error message string, works for JS errors too |
| `setCurrentException` | proc | Manually inject an exception into the runtime state |
| `getStackTrace()` | proc | Snapshot of the current call stack as a string |
| `getStackTrace(e)` | proc | Retrieve the frozen stack trace stored inside exception `e` |
