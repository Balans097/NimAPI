# std/stackframes — Module Reference

> **Part of Nim's standard library** (`std/stackframes`).

## Overview

`std/stackframes` is a low-level diagnostic module that gives you direct, zero-overhead access to the runtime call stack frame that Nim maintains for each active procedure invocation. It is designed for two specific use cases:

1. **Introspection** — finding out the name of the currently executing C/C++ function at compile time, or obtaining a pointer to the current stack frame for custom stack-walking code.
2. **Annotating stack frames with messages** — attaching arbitrary string notes to the current frame so that they appear alongside the procedure name in a stack trace when an exception propagates.

All three exported symbols are **templates**, not procedures. This is intentional: templates are inlined at the call site and therefore avoid the cost of an extra function call and a new stack frame. For stack introspection code that is meant to describe its immediate caller, this distinction matters a great deal — a procedure version of `getPFrame` would return a pointer to *its own* frame, not the caller's.

### Compile-time guards

Two module-level constants drive conditional compilation throughout the module:

```nim
const NimStackTrace     = compileOption("stacktrace")
const NimStackTraceMsgs = compileOption("stacktraceMsgs")
```

- `NimStackTrace` is `true` when the program is compiled with `--stacktrace:on` (the default in debug builds).
- `NimStackTraceMsgs` is `true` when compiled with `--stacktraceMsgs:on`.

Whenever stack tracing is disabled these templates either return an empty/zero value or become complete no-ops — they never produce a compile error regardless of configuration. This means you can freely use them in library code without imposing a compile flag requirement on your users.

---

## Templates

---

### `procName`

```nim
template procName*(): string
```

**Returns the name of the current C or C++ function as a string.**

This template emits a reference to the compiler built-in `__func__` (C99 / C++11 standard), which is a `const char*` holding the unmangled name of the enclosing function as the compiler sees it at that exact point. The result is converted to a Nim `string` and returned.

**Availability:** Only works when the Nim backend is `c` or `cpp` (i.e. when compiled with `-b:c` or `-b:cpp`, which is the default). On other backends the template body is absent and calling it will produce a compile error — guard with `when defined(c) or defined(cpp)` if you need to be portable.

**What the name looks like:** For a Nim procedure `proc myProc(x: int)` the C backend generates a C function whose name follows Nim's name mangling scheme, for example `myProc__module_NimInt`. The exact name depends on the module, overload suffix, and Nim version. It is the *C-level* name, not the Nim-level name.

**Typical use — low-level logging and diagnostics:**

```nim
import std/stackframes

proc doSomething() =
  when defined(c) or defined(cpp):
    echo "Currently in C function: ", procName()
```

**Example — recording the origin of a debug message:**

```nim
import std/stackframes

proc criticalSection() =
  when defined(c) or defined(cpp):
    let fn = procName()
    debugLog("[" & fn & "] entering critical section")
    # ... work ...
    debugLog("[" & fn & "] leaving critical section")
```

---

### `getPFrame`

```nim
template getPFrame*(): PFrame
```

**Returns a `PFrame` pointer to the current procedure's stack frame, without the overhead of a function call.**

`PFrame` is Nim's internal type for a stack frame record. Each procedure invocation that is tracked by the stack-trace system has one `FR_` local variable of this type on the C stack. `getPFrame` directly takes the address of that local variable via `{.emit: "`framePtr` = &FR_;".}` and returns it.

The critical advantage over `getFrame()` (from `std/system`) is that `getFrame()` is a *function* — calling it pushes a new frame onto the stack and returns the frame for `getFrame` itself, not for your code. `getPFrame` is a template, so after inlining `&FR_` refers to the frame of the *call site*, which is what you almost always want.

**When stack tracing is disabled** (`--stacktrace:off`): `FR_` does not exist, so the template evaluates to a zero-initialised block result. The returned value should not be dereferenced in that case.

**Typical use — custom stack walkers and profilers:**

```nim
import std/stackframes

proc profiledOperation() =
  let frame = getPFrame()
  when frame != nil:
    echo "Frame procname: ", frame.procname
    echo "Frame filename: ", frame.filename
    echo "Frame line:     ", frame.line
```

**Example — passing the current frame to a custom diagnostic sink:**

```nim
import std/stackframes

proc logWithFrame(msg: string) =
  let f = getPFrame()
  myDiagnosticSink(f, msg)

proc compute(x: int): int =
  logWithFrame("starting computation")
  result = x * x
```

---

### `setFrameMsg`

```nim
template setFrameMsg*(msg: string, prefix = " ")
```

**Attaches a string message to the current stack frame so it appears in exception stack traces.**

When an exception is raised and propagates up the call stack, Nim's stack trace mechanism normally shows only the procedure name, file, and line number for each frame. `setFrameMsg` lets you embed additional runtime context — variable values, state flags, loop counters — directly into the trace at the point they matter. This can dramatically reduce the time spent figuring out *why* a particular execution path led to an exception.

The message is appended to a global `frameMsgBuf` string at the current frame's offset. `setFrameMsg` can be called **multiple times** within the same frame — each call appends `prefix + msg` to the frame's message region. The `prefix` parameter defaults to a single space and is intended to separate successive messages visually.

**Double compile guard:** This template is a **complete no-op** unless *both* `--stacktrace:on` and `--stacktraceMsgs:on` are active. Under any other combination it compiles to nothing and has zero runtime cost. You can therefore call it freely in hot paths and simply disable it in release builds.

**Typical use — annotating a frame with a loop variable:**

```nim
import std/stackframes

proc processItems(items: seq[string]) =
  for i, item in items:
    setFrameMsg("item=" & item & " i=" & $i)
    processOneItem(item)   # if this raises, the trace shows which item failed
```

**Example — multiple annotations in one frame:**

```nim
import std/stackframes

proc connect(host: string, port: int) =
  setFrameMsg("host=" & host)
  setFrameMsg("port=" & $port)
  # If connection fails, the stack trace will show both annotations
  rawConnect(host, port)
```

**Example — annotating with computed state:**

```nim
import std/stackframes

proc deserialize(data: openArray[byte]) =
  let version = data[0]
  let length  = (int(data[1]) shl 8) or int(data[2])
  setFrameMsg("v=" & $version & " len=" & $length)
  # Any parse error downstream now carries version and length in the trace
  parseBody(data[3..^1], length)
```

---

## How stack frame messages appear in traces

When an exception propagates with both `--stacktrace` and `--stacktraceMsgs` active, the output looks like:

```
Traceback (most recent call last)
mymodule.nim(42)         processItems  item=broken_value i=7
mymodule.nim(19)         deserialize   v=2 len=512
system.nim(...)          ...
Error: unhandled exception: unexpected token [ParseError]
```

Without `setFrameMsg`, the same trace would show only procedure names and line numbers, with no hint about *which* item or *what* version caused the failure.

---

## Compile flag quick reference

| Flag | Effect on this module |
|---|---|
| `--stacktrace:on` (default in debug) | Enables `getPFrame` and `setFrameMsg`; enables `FR_` locals |
| `--stacktrace:off` (default in release) | `getPFrame` returns nil/zero; `setFrameMsg` is a no-op |
| `--stacktraceMsgs:on` | Enables `setFrameMsg` message appending |
| `--stacktraceMsgs:off` (default) | `setFrameMsg` is a no-op even with `--stacktrace:on` |
| `-b:c` or `-b:cpp` (default) | Enables `procName` |
| Other backends | `procName` body is absent |

---

## Summary Table

| Symbol | Kind | Purpose | Active when |
|--------|------|---------|-------------|
| `procName()` | template | C-level function name of the current scope | C/C++ backend only |
| `getPFrame()` | template | Pointer to current stack frame (`PFrame`) | `--stacktrace:on` |
| `setFrameMsg(msg, prefix)` | template | Attach runtime message to current frame's trace output | `--stacktrace:on` **and** `--stacktraceMsgs:on` |

---

> **See also:** [`system.getFrame`](https://nim-lang.org/docs/system.html#getFrame) for a function-based (higher-overhead) frame accessor; [`system.PFrame`](https://nim-lang.org/docs/system.html) for the definition of the stack frame record type; the Nim compiler flags `--stacktrace` and `--stacktraceMsgs` in the [compiler user guide](https://nim-lang.org/docs/nimc.html).
