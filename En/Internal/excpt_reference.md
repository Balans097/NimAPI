# `excpt.nim` — Module Reference

> **Module:** `excpt` — Nim Runtime Library, Exception Handling Core  
> **Stability:** Internal / semi-public. Marked with `*` = exported. Several symbols are also tagged `compilerRtl`, meaning they are primarily called by Nim's code generator rather than end-user code. The public API most useful to application developers is `onUnhandledException`, `errorMessageWriter`, `getStackTraceEntries`, and `stackTraceAvailable`.

---

## Overview

`excpt.nim` is the heart of Nim's runtime exception machinery. Its responsibilities are:

- Maintaining the per-thread **call-frame stack** (used to generate stack traces).
- Maintaining the per-thread **current exception** chain.
- Writing formatted stack traces to stderr (or to a user-supplied hook).
- Raising, re-raising, and propagating exceptions to the nearest handler.
- Catching OS **signals** (SIGSEGV, SIGFPE, etc.) and converting them into readable crash messages.

Because it must work even when the heap is unavailable (e.g. very small embedded programs that never allocate), the file is written with extreme care about memory allocation. Many routines have zero heap dependencies.

---

## Exported Variables

---

### `errorMessageWriter`

```nim
var errorMessageWriter*: (proc(msg: string) {.tags: [WriteIOEffect], gcsafe, nimcall, raises: [].})
```

**What it does:**  
A replaceable callback that receives every error/diagnostic message the runtime wants to print (stack traces, crash reports, signal messages). When this variable is `nil` the runtime falls back to writing directly to `stderr`.

**Why you would set it:**  
You might want to redirect crash output to a log file, a GUI dialog, a network socket, or a test harness that captures output.

**Important:** The callback signature is strict — it must be `gcsafe`, accept no extra arguments, and never raise. This is because it can be called from signal handlers or GC-unsafe contexts.

```nim
# Redirect all runtime error messages to a file
var logFile = open("crash.log", fmWrite)

errorMessageWriter = proc(msg: string) =
  logFile.write(msg)
  logFile.flushFile()

# Now any unhandled exception or signal crash writes to crash.log
```

---

### `onUnhandledException`

```nim
var onUnhandledException*: (proc (errorMsg: string) {.nimcall, gcsafe.})
```

**What it does:**  
Called by the runtime when an exception propagates all the way to the top of a thread with no handler left to catch it. The `errorMsg` string is already fully formatted and contains the stack trace plus the exception message and type name.

When `nil` (the default), the runtime writes `errorMsg` to stderr and calls `quit(1)`.

**Why you would set it:**  
- Custom crash reporters (send a bug report to a server).
- GUI applications that must show a dialog instead of printing to stderr.
- Test frameworks that want to assert on unhandled exceptions.

**Stability:** Unstable API — may change between Nim versions.

```nim
onUnhandledException = proc(errorMsg: string) =
  # Show a GUI popup instead of writing to console
  showDialog("Fatal Error", errorMsg)
  quit(1)
```

```nim
# Capture the crash message in tests
var lastCrash = ""
onUnhandledException = proc(errorMsg: string) =
  lastCrash = errorMsg
  quit(1)
```

---

### `frameMsgBuf` *(conditional)*

```nim
when NimStackTraceMsgs:
  var frameMsgBuf* {.threadvar.}: string
```

**What it does:**  
A per-thread string buffer that accumulates per-frame debug messages when the compiler flag `-d:nimStackTraceMsgs` is active. Each stack frame can append additional context text (e.g. variable values) that will appear inline in the stack trace.

**When it exists:** Only compiled in when `NimStackTraceMsgs` is `true` (i.e. `-d:nimStackTraceMsgs` passed to the compiler).

**Typical usage:** Advanced debugging — append context to `frameMsgBuf` inside a procedure to have that information appear in the stack trace if the program crashes.

```nim
# Only valid when compiled with -d:nimStackTraceMsgs
frameMsgBuf.add("userId=" & $userId & " ")
```

---

## Frame-State Functions

These functions save and restore the complete per-thread exception/frame state. They are primarily used by the compiler when implementing coroutines, async/await, closures, and other constructs that need to suspend and resume execution context.

---

### `getFrameState`

```nim
proc getFrameState*(): FrameState {.compilerRtl, inl.}
```

**What it does:**  
Captures a snapshot of the current thread's entire exception-handling context into a `FrameState` tuple. Depending on the compilation mode (`gotoBasedExceptions` or not), the snapshot contains some or all of:

- `framePtr` — pointer to the top of the call-frame stack.
- `currException` — the currently active exception (if any).
- `excHandler` — the innermost active `try` block handler.
- `gcFramePtr` — the GC frame stack pointer.

**Analogy:** Think of it like `setjmp` — it takes a "photograph" of the control-flow state at a given moment.

```nim
# Save state before entering a coroutine body
let savedState = getFrameState()
# ... run other code ...
setFrameState(savedState)  # restore when we come back
```

---

### `setFrameState`

```nim
proc setFrameState*(state: FrameState) {.compilerRtl, inl.}
```

**What it does:**  
The inverse of `getFrameState`. Restores all thread-local exception/frame pointers from a previously saved `FrameState` snapshot.

**Warning:** Misusing this function (e.g. restoring a state that refers to stack frames that no longer exist) leads to undefined behavior. It exists for use by generated code, not general application logic.

```nim
let state = getFrameState()
doSomeWork()
setFrameState(state)  # undo any frame/exception changes made by doSomeWork
```

---

### `getFrame`

```nim
proc getFrame*(): PFrame {.compilerRtl, inl.}
```

**What it does:**  
Returns the `PFrame` pointer to the top of the current thread's call-frame linked list. Each frame holds the procedure name, source filename, and line number that appear in stack traces.

**Typical use:** Inspecting or walking the stack trace programmatically at runtime without triggering an exception.

```nim
let topFrame = getFrame()
if topFrame != nil:
  echo "Currently in: ", topFrame.procname
  echo "  at ", topFrame.filename, ":", topFrame.line
```

---

### `setFrame`

```nim
proc setFrame*(s: PFrame) {.compilerRtl, inl.}
```

**What it does:**  
Directly overwrites the thread-local `framePtr` with the given frame pointer `s`. Used by the compiler to restore the frame pointer when unwinding across coroutine or async boundaries.

**Caution:** Like `setFrameState`, this is a low-level operation. Setting an invalid frame pointer corrupts stack traces.

```nim
# Compiler-generated pattern for async/coroutine resumption:
let savedFrame = getFrame()
setFrame(resumptionFrame)
runResumedCode()
setFrame(savedFrame)
```

---

## GC Frame Functions *(non-`gotoBasedExceptions` only)*

These four functions manage a secondary linked list of **GC frames** — structures that tell the garbage collector which heap references are live on the current call stack. They only exist when `gotoBasedExceptions` is `false` (the traditional `setjmp`-based exception model).

---

### `getGcFrame`

```nim
proc getGcFrame*(): GcFrame {.compilerRtl, inl.}
```

Returns a pointer to the top of the GC-frame stack. Used when saving/restoring GC roots across coroutine or exception boundaries.

---

### `popGcFrame`

```nim
proc popGcFrame*() {.compilerRtl, inl.}
```

Removes the topmost GC frame from the stack (equivalent to `gcFramePtr = gcFramePtr.prev`). Called by compiler-generated code when a procedure returns.

---

### `setGcFrame`

```nim
proc setGcFrame*(s: GcFrame) {.compilerRtl, inl.}
```

Directly sets the GC frame stack pointer to `s`. Used during coroutine resumption to restore GC root visibility.

---

### `pushGcFrame`

```nim
proc pushGcFrame*(s: GcFrame) {.compilerRtl, inl.}
```

**What it does:**  
Pushes a new GC frame onto the stack. Before linking it in, the function **zeroes out all pointer slots** inside the frame (the memory immediately following the `GcFrameHeader`). This zeroing is critical: it ensures the GC never sees stale/garbage pointers if a GC cycle happens before those slots are properly initialized.

```nim
# Compiler-generated at the start of any proc that holds ref-type locals:
var myGcFrame: MyGcFrameType
pushGcFrame(addr myGcFrame)
# ... proc body ...
popGcFrame()
```

---

## Stack Trace Functions

---

### `stackTraceAvailable`

```nim
proc stackTraceAvailable*(): bool
```

**What it does:**  
Returns `true` if the current build is capable of producing meaningful stack traces. The answer depends on compile-time flags:

| Condition | Result |
|---|---|
| Compiled with `NimStackTrace` (debug builds, default) | `true` if `framePtr != nil` |
| Compiled with `-d:nimStackTraceOverride` | always `true` |
| Compiled with `-d:nativeStackTrace` on Linux/macOS | `true` |
| Release builds without the above | `false` |

**Why this matters:** In production (release) builds, stack traces are usually disabled for performance. Calling `getStackTraceEntries()` in such a build would return an empty sequence. `stackTraceAvailable` lets you check this before attempting to collect trace data.

```nim
if stackTraceAvailable():
  let entries = getStackTraceEntries()
  for e in entries:
    echo e.filename, ":", e.line, " in ", e.procname
else:
  echo "(stack traces not available in this build)"
```

---

### `getStackTraceEntries` *(for an exception)*

```nim
proc getStackTraceEntries*(e: ref Exception): lent seq[StackTraceEntry]
```

**What it does:**  
Returns the stack trace that was captured **at the moment the exception `e` was raised**, as a sequence of `StackTraceEntry` records. Each entry contains:

- `procname` — the name of the procedure at that level.
- `filename` — the source file.
- `line` — the line number.
- `frameMsg` — optional per-frame message (when `NimStackTraceMsgs` is active).

The return type is `lent`, meaning it is a borrowed reference into the exception object — no copy is made.

**Key insight:** This trace was frozen at raise-time, not at the point you call `getStackTraceEntries`. This means you can call this function from a `finally` block or from an error-reporting routine and still get the original raise location.

```nim
try:
  riskyOperation()
except IOError as e:
  let entries = getStackTraceEntries(e)
  for entry in entries:
    echo entry.filename, "(", entry.line, ") ", entry.procname
```

```nim
# Structured logging: attach trace to a log record
proc logException(e: ref Exception) =
  var record = newLogRecord()
  record.message = e.msg
  record.trace = getStackTraceEntries(e)  # attach the frozen trace
  submitLog(record)
```

---

### `getStackTraceEntries` *(current stack)*

```nim
proc getStackTraceEntries*(): seq[StackTraceEntry]
```

**What it does:**  
Captures the stack trace **right now** — at the point where this function is called — and returns it as a fresh `seq[StackTraceEntry]`. This is the non-exception variant: you don't need to be inside an exception handler to use it.

**Difference from the exception overload:** This creates a new `seq` and captures the live call stack. The exception overload returns an existing (frozen) trace stored inside the exception object.

**Note:** Returns an empty sequence when `stackTraceAvailable()` is `false`.

```nim
# Log where we are without raising anything
proc checkpoint(label: string) =
  let trace = getStackTraceEntries()
  debugLog(label, trace)

proc processOrder(id: int) =
  checkpoint("processOrder start")
  # ...
```

```nim
# Assert that a certain function is in the call stack
proc mustBeCalledFrom(expectedProc: string) =
  let trace = getStackTraceEntries()
  let found = trace.anyIt(it.procname == expectedProc)
  assert found, "Expected to be called from " & expectedProc
```

---

## Notes on Platform Variants

- **Windows GUI apps** (`-d:guiapp`): `writeToStdErr` is replaced by `MessageBoxA`, so crash messages appear as dialog boxes.
- **Genode**: Uses `echo` via the Genode LOG session since stderr is unavailable.
- **C++ backend** (`--cc:cpp`): Exception propagation uses native C++ `throw`/`catch`. `std::set_terminate` is installed to handle uncaught C++ exceptions.
- **`gotoBasedExceptions`**: An alternative exception model (used for ORC/ARC memory management) that replaces `setjmp`/`longjmp` with explicit error flags and `goto` — the GC frame functions and `excHandler` do not exist in this mode.
- **JS backend**: `getStackTraceEntries` is documented as not yet available.

---

## Quick Cheat-Sheet

| Symbol | Kind | Use case |
|---|---|---|
| `errorMessageWriter` | `var` | Redirect runtime crash output |
| `onUnhandledException` | `var` | Custom handler for uncaught exceptions |
| `frameMsgBuf` | `var` | Per-frame debug messages (debug builds) |
| `getFrameState` / `setFrameState` | proc | Save/restore full exception context |
| `getFrame` / `setFrame` | proc | Access/set current call-frame pointer |
| `getGcFrame` / `setGcFrame` / `pushGcFrame` / `popGcFrame` | proc | Manage GC root frames (non-goto mode) |
| `stackTraceAvailable` | proc | Check if traces work in this build |
| `getStackTraceEntries(e)` | proc | Get frozen trace from an exception |
| `getStackTraceEntries()` | proc | Capture live stack trace right now |
