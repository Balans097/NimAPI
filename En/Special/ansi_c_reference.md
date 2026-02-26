# `ansi_c.nim` — Module Reference

> **Part of Nim's Runtime Library**  
> Compiled with `{.push hints:off, stack_trace:off, profiler:off, raises:[].}`

---

## Overview

`ansi_c.nim` is the **glue layer between Nim's runtime and the C standard library**. It imports a carefully selected set of C functions — memory operations, string utilities, I/O primitives, signal handling, and heap allocation — into Nim's namespace under a consistent `c_` prefix to avoid any ambiguity with Nim's own standard library symbols.

This module is **not intended for application-level use**. It underpins the runtime itself: the garbage collector, the exception mechanism, the I/O system, and the crash handler all call into these wrappers. Understanding this module gives you a clear view of what is happening "one level below" ordinary Nim code.

Every procedure is `{.inline.}` or directly maps to a C function, so there is **zero overhead** beyond the C function itself. No stack traces, no profiler hooks, no exception annotations — this module intentionally lives outside all of those facilities.

---

## Module-Level Pragmas

```nim
{.push hints:off, stack_trace:off, profiler:off, raises:[].}
```

All symbols in this file are compiled with these pragmas active:

- **`hints:off`** — suppresses Nim compiler hints, keeping the runtime build clean.
- **`stack_trace:off`** — these procedures do not appear in Nim stack traces. Intentional: they are C-level and would pollute debug output.
- **`profiler:off`** — excluded from Nim's profiler. Raw C calls are not worth profiling at the Nim level.
- **`raises:[]`** — the compiler is told these procedures raise no Nim exceptions. This is crucial: the runtime uses them precisely in places where raising would be impossible or dangerous (e.g. inside the GC or crash handler).

---

## Type Definitions

### `C_JmpBuf*`

```nim
type C_JmpBuf* = ...  # platform-dependent
```

A Nim wrapper for C's `jmp_buf` — the buffer used by `setjmp`/`longjmp` to save and restore CPU register state. Its concrete definition is chosen at compile time:

| Condition | Definition |
|---|---|
| `-d:nimBuiltinSetjmp` | `array[5, pointer]` — compiler built-in, no C header |
| Linux + amd64 | `object` with `array[200 div sizeof(clong), clong]` abi field — matches Linux x86-64 ABI exactly |
| Everything else | Opaque `object` imported from `<setjmp.h>` |

`C_JmpBuf` is used exclusively by `c_setjmp` and `c_longjmp`. Application code should never need to touch it directly.

---

### `CFilePtr*`

```nim
type
  CFile {.importc: "FILE", header: "<stdio.h>", incompleteStruct.} = object
  CFilePtr* = ptr CFile
```

A pointer to C's `FILE` structure — the fundamental handle for all C-level I/O. Nim's own `File` type wraps this internally. `CFilePtr` appears in all the I/O procedures in this module.

---

## Global Variables

### `cstdin*`, `cstdout*`, `cstderr*`

```nim
var cstderr* {.importc: stderrName, header: "<stdio.h>".}: CFilePtr
var cstdout* {.importc: stdoutName, header: "<stdio.h>".}: CFilePtr
var cstdin*  {.importc: stdinName,  header: "<stdio.h>".}: CFilePtr
```

Direct Nim bindings to C's three standard streams. The actual C symbol name is chosen at compile time (`__stderrp`/`__stdoutp`/`__stdinp` on macOS/FreeBSD/DragonFly where `stdio.h` uses macros; `stderr`/`stdout`/`stdin` everywhere else).

These are used throughout the runtime for low-level output — for example, in crash reporters and the `rawWrite` helpers — precisely because they bypass all of Nim's higher-level buffering and exception machinery.

---

## Signal Constants

These integer constants identify POSIX/C signals. They are defined as compile-time constants on known platforms or imported from the C headers on unknown ones.

| Constant | Typical value | Meaning |
|---|---|---|
| `SIGABRT*` | 6 (POSIX), 22 (Windows) | Abnormal program termination (`abort()`) |
| `SIGFPE*`  | 8 | Floating-point error (divide by zero, overflow) |
| `SIGILL*`  | 4 | Illegal instruction |
| `SIGINT*`  | 2 | Interactive interrupt (Ctrl+C) |
| `SIGSEGV*` | 11 | Segmentation fault (invalid memory access) |
| `SIGTERM*` | 15 | Termination request |
| `SIGPIPE*` | 13 (POSIX) / 7 (Haiku) | Broken pipe (write to closed socket/pipe) |
| `SIGBUS*`  | 10 (macOS) / 30 (Haiku) | Bus error (misaligned memory access) |
| `SIG_DFL*` | 0 / nil | Restore default signal handler |

> **Note:** `SIGTERM` is not exported on Windows (`SIGTERM` without `*`). `SIGPIPE` and `SIGBUS` are POSIX-only and not available on Windows at all.

---

## Memory Procedures

These are direct wrappers of the C `<string.h>` memory functions. All operate on raw `pointer` arguments — no type safety, no bounds checking.

---

### `c_memchr`

```nim
proc c_memchr*(s: pointer, c: cint, n: csize_t): pointer
```

Searches the first `n` bytes starting at `s` for the **first occurrence** of byte value `c`. Returns a pointer to the found byte, or `nil` if not found.

This is the C-level equivalent of a byte-scan. It is used by the runtime when scanning raw memory for a specific byte value.

```nim
var buf = [byte(0), 1, 2, 42, 3]
let p = c_memchr(addr buf[0], 42, csize_t(buf.len))
# p points to buf[3], which holds 42
```

---

### `c_memcmp`

```nim
proc c_memcmp*(a, b: pointer, size: csize_t): cint {.noSideEffect.}
```

Compares `size` bytes of memory at `a` with memory at `b` **byte by byte** (unsigned comparison). Returns:
- `0` if the regions are identical,
- a negative value if `a` is lexicographically less than `b`,
- a positive value if `a` is greater.

Marked `noSideEffect` because it only reads memory and produces a deterministic result. Used in the runtime for comparing raw data blocks — for example in certain GC write-barrier checks and type descriptor comparisons.

```nim
var x = [byte(1), 2, 3]
var y = [byte(1), 2, 3]
assert c_memcmp(addr x[0], addr y[0], 3) == 0  # identical

var z = [byte(1), 2, 4]
assert c_memcmp(addr x[0], addr z[0], 3) < 0   # x < z
```

---

### `c_memcpy`

```nim
proc c_memcpy*(a, b: pointer, size: csize_t): pointer {.discardable.}
```

Copies exactly `size` bytes from source `b` to destination `a`. The source and destination regions **must not overlap** — for overlapping regions, use `c_memmove`. Returns `a`.

The return value is marked `discardable` because callers almost always ignore it (it is the same as `a`). This is the fastest raw byte copy available in C; on modern CPUs it compiles to SIMD-optimised memory moves.

```nim
var src = [byte(10), 20, 30)
var dst: array[3, byte]
c_memcpy(addr dst[0], addr src[0], 3)
# dst is now [10, 20, 30]
```

---

### `c_memmove`

```nim
proc c_memmove*(a, b: pointer, size: csize_t): pointer {.discardable.}
```

Copies `size` bytes from `b` to `a`, **correctly handling overlapping regions**. This is the safe alternative to `c_memcpy` when source and destination may alias each other. Slightly more expensive than `c_memcpy` because it must handle forward and backward copying depending on direction.

```nim
var buf = [byte(1), 2, 3, 4, 5]
# Shift elements right by one position (overlapping)
c_memmove(addr buf[1], addr buf[0], 4)
# buf is now [1, 1, 2, 3, 4]
```

---

### `c_memset`

```nim
proc c_memset*(p: pointer, value: cint, size: csize_t): pointer {.discardable.}
```

Fills the first `size` bytes of memory starting at `p` with the byte value of `value` (only the lowest 8 bits are used). Returns `p`. Used extensively by the runtime to zero-initialise freshly allocated memory blocks.

```nim
var buf: array[8, byte]
c_memset(addr buf[0], 0, 8)   # zero out
# buf = [0, 0, 0, 0, 0, 0, 0, 0]

c_memset(addr buf[0], 0xFF, 4)  # fill first 4 bytes with 0xFF
# buf = [255, 255, 255, 255, 0, 0, 0, 0]
```

---

## String Procedures

### `c_strcmp`

```nim
proc c_strcmp*(a, b: cstring): cint {.noSideEffect.}
```

Compares two null-terminated C strings **lexicographically** (by character code). Returns `0` if equal, negative if `a < b`, positive if `a > b`. Both strings must be valid (non-nil, null-terminated).

```nim
assert c_strcmp("apple", "apple") == 0
assert c_strcmp("apple", "banana") < 0
assert c_strcmp("zebra", "ant") > 0
```

---

### `c_strlen`

```nim
proc c_strlen*(a: cstring): csize_t {.noSideEffect.}
```

Returns the number of bytes in null-terminated string `a`, **not counting** the terminating `\0`. This is the C-level string length measurement used throughout Nim's runtime when it needs to determine the length of a `cstring` before processing it.

```nim
assert c_strlen("hello") == 5
assert c_strlen("") == 0
```

---

### `c_strstr`

```nim
proc c_strstr*(haystack, needle: cstring): cstring {.noSideEffect.}
```

Searches for the **first occurrence** of substring `needle` inside `haystack`. Returns a pointer to the beginning of the found substring within `haystack`, or `nil` if not found.

```nim
let result = c_strstr("hello world", "world")
# result points to the 'w' in "world" within "hello world"

let missing = c_strstr("hello world", "xyz")
# missing == nil
```

---

## Control Flow Procedures

### `c_abort`

```nim
proc c_abort*() {.noSideEffect, noreturn.}
```

Calls C's `abort()`, which **immediately terminates the program abnormally**. It raises `SIGABRT` and — depending on the platform and configuration — produces a core dump. Marked `noreturn`: the compiler knows execution never continues past this call.

Used by Nim's runtime as the last resort when the program reaches an unrecoverable state (e.g. an assertion failure in the memory manager itself, or a fatal internal error where even raising a Nim exception is unsafe).

```nim
# In the runtime:
if internalInvariantBroken:
  c_abort()  # program ends here, unconditionally
```

---

### `c_setjmp`

```nim
proc c_setjmp*(jmpb: C_JmpBuf): cint
```

Saves the current CPU register state (stack pointer, instruction pointer, callee-saved registers) into `jmpb` and returns `0`. If execution is later transferred back here via `c_longjmp`, it returns the non-zero value passed to `c_longjmp` instead.

This is one of the most subtle functions in all of C: it **returns twice** — once when called normally (returns `0`), and again when `c_longjmp` fires (returns non-zero). It is used by Nim's exception mechanism to implement try/catch at the C level without relying on C++ exceptions.

The actual C function called depends on the compilation flags:

| Flag | C function |
|---|---|
| `-d:nimSigSetjmp` | `sigsetjmp(jmpb, 0)` — saves signal mask too |
| `-d:nimBuiltinSetjmp` | `__builtin_setjmp` — GCC/Clang compiler intrinsic |
| `-d:nimRawSetjmp` (Windows) | `setjmp` or `_setjmp(jmpb, NULL)` |
| `-d:nimRawSetjmp` (POSIX) | `_setjmp` — does not save signal mask |
| Default | `setjmp` — standard ANSI C |

```nim
var buf: C_JmpBuf
let code = c_setjmp(buf)
if code == 0:
  echo "First time through — normal execution"
  # ... do work, maybe call c_longjmp(buf, 1) somewhere deep
else:
  echo "Jumped back! Return value was: ", code
```

---

### `c_longjmp`

```nim
proc c_longjmp*(jmpb: C_JmpBuf, retval: cint)
```

Restores CPU state from `jmpb` (previously saved by `c_setjmp`) and causes `c_setjmp` to return `retval`. If `retval` is `0`, it is treated as `1` by the C standard to distinguish the jump return from the initial call. **Execution does not return** from `c_longjmp` itself.

This is the runtime's non-local exit mechanism. When Nim raises an exception and C++ exceptions are not available (e.g. when compiled with `-d:nimRawSetjmp`), the exception mechanism uses `c_longjmp` to unwind the stack back to the nearest enclosing `c_setjmp` call.

```nim
var buf: C_JmpBuf
if c_setjmp(buf) == 0:
  echo "Setting up..."
  c_longjmp(buf, 42)   # jump back to setjmp
  echo "Never reached"
else:
  echo "Returned from longjmp"  # this executes, prints 42 as code
```

> **Warning:** All stack-allocated variables modified between `c_setjmp` and `c_longjmp` must be declared `volatile` in C to have defined behaviour. In Nim this is handled by the compiler.

---

## Signal Procedures

### `c_signal`

```nim
proc c_signal*(sign: cint, handler: CSighandlerT): CSighandlerT {.discardable.}
```

Installs `handler` as the signal handler for signal number `sign`. Returns the previous handler. Use `SIG_DFL` as the handler to restore the default OS behaviour.

Used by Nim's runtime to install crash handlers (for `SIGSEGV`, `SIGFPE`, etc.) so that a segfault or floating-point error can be turned into a Nim exception or a clean crash report rather than a silent OS termination.

The handler type `CSighandlerT` is `proc(a: cint) {.noconv.}` — a C-convention callback taking the signal number.

```nim
proc myHandler(sig: cint) {.noconv.} =
  c_fprintf(cstderr, "Caught signal %d\n", sig)
  c_abort()

c_signal(SIGSEGV, myHandler)  # install crash handler
```

---

### `c_raise`

```nim
proc c_raise*(sign: cint): cint
```

Sends signal `sign` to the **current process**. Returns `0` on success, non-zero on failure. This is the programmatic equivalent of pressing Ctrl+C (for `SIGINT`) or triggering any other signal from within the program itself.

```nim
# Simulate an interrupt from within the program:
discard c_raise(SIGINT)
```

---

## I/O Procedures

### `c_fprintf`

```nim
proc c_fprintf*(f: CFilePtr, frmt: cstring): cint {.varargs, discardable.}
```

The C `fprintf` function: formats a string according to `frmt` (printf-style format specifiers) and writes it to file handle `f`. Additional arguments follow `frmt` in C varargs style. Returns the number of characters written (usually discarded).

Used throughout the runtime for diagnostic output — especially in crash handlers and internal assertions where Nim's high-level I/O cannot be trusted.

```nim
c_fprintf(cstderr, "Error: value is %d\n", 42)
c_fprintf(cstdout, "Hello, %s!\n", cstring("world"))
```

---

### `c_printf`

```nim
proc c_printf*(frmt: cstring): cint {.varargs, discardable.}
```

Like `c_fprintf` but always writes to `stdout`. A thin wrapper around C's `printf`. Used sparingly in the runtime for debug output.

```nim
c_printf("Debug: x = %d, y = %f\n", 10, 3.14)
```

---

### `c_fputs`

```nim
proc c_fputs*(c: cstring, f: CFilePtr): cint {.discardable.}
```

Writes the string `c` (without its null terminator) to file `f`. No formatting — just raw string output. Returns a non-negative value on success, `EOF` on error. Faster than `c_fprintf` when no formatting is needed.

```nim
c_fputs("hello\n", cstdout)
c_fputs("error message", cstderr)
```

---

### `c_fputc`

```nim
proc c_fputc*(c: char, f: CFilePtr): cint {.discardable.}
```

Writes a **single character** `c` to file `f`. Returns the character written as an unsigned `cint`, or `EOF` on error. The most granular I/O primitive in this module.

```nim
c_fputc('\n', cstdout)   # write a newline
c_fputc('X', cstderr)    # write 'X' to stderr
```

---

### `c_sprintf`

```nim
proc c_sprintf*(buf, frmt: cstring): cint {.varargs, noSideEffect.}
```

Formats a string into buffer `buf` according to `frmt`. **No bounds checking** — the buffer must be large enough to hold the result. Marked `noSideEffect` because it only writes to its explicit output buffer.

> **Security note:** The comment in the source says *"we use it only in a way that cannot lead to security issues"*. In practice, buffer overflows via `sprintf` are a classic C security vulnerability. In the runtime it is only called with controlled format strings and pre-sized buffers.

```nim
var buf: array[64, char]
discard c_sprintf(cast[cstring](addr buf[0]), "%d + %d = %d", 1, 2, 3)
# buf contains "1 + 2 = 3\0"
```

---

### `c_snprintf`

```nim
proc c_snprintf*(buf: cstring, n: csize_t, frmt: cstring): cint {.varargs, noSideEffect.}
```

Like `c_sprintf`, but **safe**: writes at most `n - 1` characters into `buf` and always null-terminates. Returns the number of characters that *would have been written* if `n` were large enough — so if the return value ≥ `n`, the output was truncated. This is the preferred formatting primitive.

```nim
var buf: array[16, char]
let written = c_snprintf(cast[cstring](addr buf[0]), 16, "Value: %d", 12345)
# buf = "Value: 12345\0", written = 12
```

---

### `c_fwrite`

```nim
proc c_fwrite*(buf: pointer, size, n: csize_t, f: CFilePtr): csize_t
```

Writes `n` elements of `size` bytes each from `buf` to file `f`. Returns the number of elements successfully written (may be less than `n` on error). This is the workhorse of raw binary I/O — no formatting, no null-terminator assumptions.

```nim
var data = [byte(65), 66, 67]  # "ABC"
let written = c_fwrite(addr data[0], 1, 3, cstdout)
# prints: ABC (3 bytes written)
```

---

### `c_fflush`

```nim
proc c_fflush*(f: CFilePtr): cint
```

Flushes the write buffer of file `f` to the underlying OS. Returns `0` on success, `EOF` on error. Passing `nil` flushes **all open streams**. Without flushing, output written via `c_fprintf` or `c_fwrite` may remain in a user-space buffer and never actually appear on screen or in a file.

```nim
c_fprintf(cstdout, "About to crash...\n")
discard c_fflush(cstdout)   # ensure the message is visible before a crash
c_abort()
```

---

## Heap Allocation Procedures

These are Nim wrappers for C's dynamic memory allocation functions. On Zephyr RTOS without `libc` malloc, they are remapped to `k_malloc`/`k_calloc`/`k_free`/`k_realloc` from `<kernel.h>`. On all other platforms they call the standard `<stdlib.h>` functions.

---

### `c_malloc`

```nim
proc c_malloc*(size: csize_t): pointer
```

Allocates `size` bytes of uninitialised heap memory. Returns a pointer to the block, or `nil` on failure. The returned memory has **no guaranteed content** — it may contain arbitrary garbage from previous allocations.

```nim
let p = c_malloc(csize_t(64))
if p != nil:
  c_memset(p, 0, csize_t(64))  # zero it out manually
  # ... use memory ...
  c_free(p)
```

---

### `c_calloc`

```nim
proc c_calloc*(nmemb, size: csize_t): pointer
```

Allocates `nmemb * size` bytes of heap memory, **zero-initialised**. The two-argument form exists to allow overflow detection by the C runtime. Returns `nil` on failure.

Prefer this over `c_malloc` + `c_memset` when you need zeroed memory, as `calloc` can leverage OS-level zero pages and be faster in practice.

```nim
let p = c_calloc(csize_t(10), csize_t(sizeof(int)))
# p points to 10 zero-filled int-sized slots
defer: c_free(p)
```

---

### `c_free`

```nim
proc c_free*(p: pointer)
```

Releases memory previously allocated by `c_malloc`, `c_calloc`, or `c_realloc`. Passing `nil` is safe and does nothing. Passing any other invalid pointer (already freed, not from the allocator) is **undefined behaviour** and will likely corrupt the heap.

```nim
let p = c_malloc(csize_t(32))
# ... use p ...
c_free(p)
# After this, p is a dangling pointer — never use it again
```

---

### `c_realloc`

```nim
proc c_realloc*(p: pointer, newsize: csize_t): pointer
```

Resizes the memory block at `p` to `newsize` bytes. The block may be moved to a new address; the old pointer `p` must not be used after the call. Returns the new pointer, or `nil` if the resize failed (in which case the original block is **not freed**). If `p` is `nil`, behaves like `c_malloc`.

> **Zephyr note:** On Zephyr without libc, `c_realloc` is implemented manually: it allocates a new block, copies the old data, and frees the old pointer. It cannot shrink allocations efficiently.

```nim
var p = c_malloc(csize_t(16))
p = c_realloc(p, csize_t(64))  # grow the block
if p == nil:
  echo "Realloc failed"
else:
  defer: c_free(p)
```

---

## High-Level Nim Helpers (compilerproc)

These two procedures are not raw C imports — they are Nim-level wrappers that the **compiler itself** emits calls to. They are marked `{.compilerproc.}`, meaning the Nim code generator knows their names and inserts calls to them directly.

---

### `rawWriteString`

```nim
proc rawWriteString*(f: CFilePtr, s: cstring, length: int) {.compilerproc, nonReloadable, inline.}
```

Writes exactly `length` bytes of string `s` to file `f`, then flushes. Uses `c_fwrite` + `c_fflush`. Does not throw exceptions — by design, this must work even in contexts where the exception system is broken or unavailable (e.g. during a fatal crash).

This is the procedure the compiler calls when it needs to output a string to a C file handle unconditionally and immediately.

```nim
rawWriteString(cstderr, "fatal error\n", 12)
# Writes exactly 12 bytes to stderr and flushes
```

---

### `rawWrite`

```nim
proc rawWrite*(f: CFilePtr, s: cstring) {.compilerproc, nonReloadable, inline.}
```

Like `rawWriteString`, but determines the length automatically using `c_strlen`. Writes the entire null-terminated string `s` to `f` and flushes. Also exception-safe by design.

The difference from `rawWriteString` is purely a matter of whether the caller already knows the length. When it does, `rawWriteString` avoids the extra `strlen` call.

```nim
rawWrite(cstdout, "Hello from the runtime\n")
```

---

## Summary Table

| Procedure | C origin | Header | Purpose |
|---|---|---|---|
| `c_memchr` | `memchr` | `<string.h>` | Find byte in memory |
| `c_memcmp` | `memcmp` | `<string.h>` | Compare memory regions |
| `c_memcpy` | `memcpy` | `<string.h>` | Copy non-overlapping memory |
| `c_memmove` | `memmove` | `<string.h>` | Copy potentially-overlapping memory |
| `c_memset` | `memset` | `<string.h>` | Fill memory with a byte value |
| `c_strcmp` | `strcmp` | `<string.h>` | Compare C strings |
| `c_strlen` | `strlen` | `<string.h>` | Measure C string length |
| `c_strstr` | `strstr` | `<string.h>` | Find substring in C string |
| `c_abort` | `abort` | `<stdlib.h>` | Unconditional program termination |
| `c_setjmp` | `setjmp` (variant) | `<setjmp.h>` | Save execution context |
| `c_longjmp` | `longjmp` (variant) | `<setjmp.h>` | Restore execution context |
| `c_signal` | `signal` | `<signal.h>` | Install signal handler |
| `c_raise` | `raise` | `<signal.h>` | Send signal to current process |
| `c_fprintf` | `fprintf` | `<stdio.h>` | Formatted write to file |
| `c_printf` | `printf` | `<stdio.h>` | Formatted write to stdout |
| `c_fputs` | `fputs` | `<stdio.h>` | Write string to file |
| `c_fputc` | `fputc` | `<stdio.h>` | Write single char to file |
| `c_sprintf` | `sprintf` | `<stdio.h>` | Format string into buffer (unsafe) |
| `c_snprintf` | `snprintf` | `<stdio.h>` | Format string into buffer (safe) |
| `c_fwrite` | `fwrite` | `<stdio.h>` | Write binary data to file |
| `c_fflush` | `fflush` | `<stdio.h>` | Flush file write buffer |
| `c_malloc` | `malloc` / `k_malloc` | `<stdlib.h>` | Allocate uninitialised heap memory |
| `c_calloc` | `calloc` / `k_calloc` | `<stdlib.h>` | Allocate zero-initialised heap memory |
| `c_free` | `free` / `k_free` | `<stdlib.h>` | Free heap memory |
| `c_realloc` | `realloc` / `k_realloc` | `<stdlib.h>` | Resize heap memory block |
| `rawWriteString` | — (Nim helper) | — | Write string+length to file, flush |
| `rawWrite` | — (Nim helper) | — | Write cstring to file, flush |
