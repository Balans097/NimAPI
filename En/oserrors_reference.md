# std/oserrors — Module Reference

> **Part of Nim's standard library** (`std/oserrors`).

## Overview

The `oserrors` module provides a thin but essential bridge between the operating system's error reporting mechanism and Nim's exception system. When you call OS-level functions — opening files, spawning processes, binding sockets — the OS records the reason for any failure as a numeric error code. This module gives you three things:

1. A typed wrapper around that numeric code (`OSErrorCode`), so the compiler stops you from accidentally mixing it with ordinary integers.
2. A way to read the most recently recorded OS error (`osLastError`).
3. Tools to translate a code into a human-readable message (`osErrorMsg`) or immediately raise a Nim exception from it (`newOSError`, `raiseOSError`).

On **POSIX** systems (Linux, macOS, BSD) the codes come from the C global `errno`. On **Windows** they come from `GetLastError()`. The module abstracts over both transparently.

---

## Type

### `OSErrorCode`

```nim
type OSErrorCode* = distinct int32
```

A **distinct** wrapper around a 32-bit signed integer that represents a numeric OS error code. Because it is `distinct`, Nim's type system treats it as completely separate from `int32` — you cannot accidentally pass a raw integer where an `OSErrorCode` is expected, or compare one against a plain number without an explicit cast.

The typical lifecycle of an `OSErrorCode`:

```
OS call fails
     │
     ▼
osLastError()          ← capture the code immediately
     │
     ├─► osErrorMsg()  ← turn it into a readable string
     │
     └─► raiseOSError() / newOSError()  ← turn it into an exception
```

---

## Operators on `OSErrorCode`

### `==`

```nim
proc `==`*(err1, err2: OSErrorCode): bool {.borrow.}
```

**Compares two `OSErrorCode` values for equality.**

Implemented via `borrow`, meaning it delegates directly to `int32`'s equality. Use this to check whether a captured error code matches a known constant.

**Example:**

```nim
import std/oserrors

let err = osLastError()
if err == OSErrorCode(0):
  echo "No error recorded."
```

---

### `$`

```nim
proc `$`*(err: OSErrorCode): string {.borrow.}
```

**Converts an `OSErrorCode` to its decimal string representation.**

This gives you the raw numeric code as a string — useful for low-level logging. It does **not** produce a human-readable error description; for that, use `osErrorMsg`.

**Example:**

```nim
import std/oserrors

let err = osLastError()
echo "Error code: ", $err        # e.g. "2"
echo "Description: ", osErrorMsg(err)  # e.g. "No such file or directory"
```

---

## Procedures

---

### `osLastError`

```nim
proc osLastError*(): OSErrorCode {.sideEffect.}
```

**Returns the error code set by the most recently failed OS call on the current thread.**

This is always the first thing you call after detecting that an OS operation has failed. The moment you call it, the code is captured and wrapped in an `OSErrorCode` that you can inspect or pass along. Delaying this call — even by one or two unrelated function calls — can cause you to lose the original error on Windows.

**Platform behaviour:**

| Platform | Source of the code |
|---|---|
| POSIX (Linux, macOS, …) | C global `errno` |
| Windows | `GetLastError()` |
| NimScript | always returns `0` (no OS access) |

> **Windows warning:** Some Windows API calls internally reset `GetLastError()` to `0` as a side effect, even when they succeed. Call `osLastError()` *immediately* after the failing call, before any other OS interaction. On POSIX this is not a concern because `errno` is only written on failure.

**Example:**

```nim
import std/oserrors

# Simulate an OS failure and capture its reason:
let code = osLastError()
if code != OSErrorCode(0):
  echo "OS error: ", osErrorMsg(code)
```

**Example — correct usage pattern (capture first, inspect later):**

```nim
import std/oserrors

let failed = someOsCall() < 0    # detect failure
let code = osLastError()         # capture immediately while errno/GetLastError is fresh
if failed:
  raiseOSError(code)
```

---

### `osErrorMsg`

```nim
proc osErrorMsg*(errorCode: OSErrorCode): string
```

**Converts a numeric OS error code into a human-readable English description string.**

Internally it calls `strerror()` on POSIX and `FormatMessageW()` on Windows, so the message text comes from the OS itself and reflects the platform's own wording. If the code is `0`, or if the OS cannot produce a message for the given code, an empty string `""` is returned (not an exception).

**Example:**

```nim
import std/oserrors

echo osErrorMsg(OSErrorCode(0))   # ""
echo osErrorMsg(OSErrorCode(1))   # "Operation not permitted"  (POSIX)
echo osErrorMsg(OSErrorCode(2))   # "No such file or directory"  (POSIX)
```

**Example — building a custom error message:**

```nim
import std/oserrors

proc checkResult(rc: int) =
  if rc < 0:
    let code = osLastError()
    let msg = osErrorMsg(code)
    echo "Failed (", $code, "): ", msg

checkResult(-1)
```

---

### `newOSError`

```nim
proc newOSError*(
  errorCode: OSErrorCode,
  additionalInfo = ""
): owned(ref OSError) {.noinline.}
```

**Creates and returns a new `OSError` exception object without raising it.**

This is the non-throwing counterpart of `raiseOSError`. Use it when you want to construct the exception, possibly inspect or augment it, and only raise it later — or pass it somewhere else entirely.

The exception's `msg` field is populated by `osErrorMsg(errorCode)`. If the resulting message is empty (code is `0` or unknown), the fallback text `"unknown OS error"` is used. The `errorCode` is also stored directly on the exception object so callers can inspect it programmatically.

If `additionalInfo` is provided and non-empty, it is appended to the message on a new line, prefixed with `"Additional info: "`. This is intentionally left without trailing punctuation so that IDE "jump to file" features work correctly when file paths are included.

**Example — creating without immediately raising:**

```nim
import std/oserrors

proc tryOpen(path: string): ref OSError =
  # ... attempt the operation ...
  let code = osLastError()
  if code != OSErrorCode(0):
    return newOSError(code, "while opening: " & path)
  return nil

let err = tryOpen("/nonexistent/path")
if err != nil:
  echo err.msg
```

**Example — appending extra context:**

```nim
import std/oserrors

let code = osLastError()
let err = newOSError(code, "path: /etc/shadow")
# err.msg might be:
# "Permission denied\nAdditional info: path: /etc/shadow"
```

---

### `raiseOSError`

```nim
proc raiseOSError*(errorCode: OSErrorCode, additionalInfo = "") {.noinline.}
```

**Creates an `OSError` exception from the given error code and immediately raises it.**

This is the most common way to surface an OS failure as a Nim exception. It is a one-liner equivalent of `raise newOSError(errorCode, additionalInfo)` and covers the vast majority of use cases where you detect a failure and want to propagate it up the call stack right away.

The `additionalInfo` parameter lets you attach contextual information — such as a file path or an operation name — that will appear in the exception message. This makes the error much easier to diagnose from a stack trace or log entry.

**Example — simplest use:**

```nim
import std/oserrors

proc mustSucceed(rc: cint) =
  if rc < 0:
    raiseOSError(osLastError())
```

**Example — with additional context:**

```nim
import std/oserrors

proc openFile(path: string): FileHandle =
  let handle = rawOpen(path)   # hypothetical raw call
  if handle < 0:
    raiseOSError(osLastError(), path)
  return handle
```

The resulting exception message might read:

```
No such file or directory
Additional info: /home/user/missing.txt
```

**Example — catching and inspecting the raised error:**

```nim
import std/oserrors

try:
  raiseOSError(OSErrorCode(13), "reading config")
except OSError as e:
  echo "Caught OSError: ", e.msg
  echo "Error code:     ", e.errorCode
```

---

## Typical Workflow

The four exported symbols are designed to be used together in a predictable pattern:

```
1. Make an OS call.
2. If it fails, immediately call osLastError() to capture the code.
3a. If you just want the message text:  osErrorMsg(code)
3b. If you want to raise an exception:  raiseOSError(code, extraInfo)
3c. If you want an exception object:    newOSError(code, extraInfo)
```

```nim
import std/oserrors, std/posix

proc readLink(path: string): string =
  var buf = newString(4096)
  let n = readlink(path.cstring, buf.cstring, buf.len.csize_t)
  if n < 0:
    raiseOSError(osLastError(), path)   # steps 2 + 3b combined
  buf.setLen(n)
  buf
```

---

## Summary Table

| Symbol | Kind | Purpose |
|--------|------|---------|
| `OSErrorCode` | type | Typed wrapper around an OS error number |
| `==` | operator | Compare two error codes |
| `$` | operator | Format a code as a decimal string |
| `osLastError()` | proc | Capture the most recent OS error code |
| `osErrorMsg(code)` | proc | Translate a code to a human-readable string |
| `newOSError(code, info)` | proc | Build an `OSError` exception object |
| `raiseOSError(code, info)` | proc | Build and immediately raise an `OSError` |

---

> **See also:** [`system.OSError`](https://nim-lang.org/docs/system.html#OSError) — the exception type raised by this module; [`std/os`](https://nim-lang.org/docs/os.html) — higher-level OS operations that use `oserrors` internally.
