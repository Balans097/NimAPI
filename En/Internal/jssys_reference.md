# system/jssys — module reference

> **Import:** this file is not imported manually by user code — it is part of the `system` module that the Nim compiler pulls in automatically when compiling to the JavaScript backend (`nim js`). Most procedures are marked with the `compilerproc` pragma and are called by the code generator itself, not from ordinary programs.
> **Scope:** runtime support for the JS backend — exceptions and stack traces, Nim/JavaScript string conversion, set operations (`set`), overflow-checked integer arithmetic, deep value copying (`nimCopy`), index and object-type checks, and `echo` output.

The module consists of low-level procedures, each serving one specific Nim-language operation in the generated JS code (for example, `a + b` for `int` compiles to a call to `addInt`). The module's general convention: almost everything is implemented through `{.emit: "...".}` blocks — raw JavaScript insertions — while the Nim signature only describes the input/output types. A second convention is the paired "fast/full" version pattern: for instance, arithmetic has 32-bit (`addInt`) and 64-bit (`addInt64`) variants with an identical structure.

---

## Table of Contents

I. [Types and exception infrastructure](#i-types-and-exception-infrastructure)
   1. [`SafePoint` / `PSafePoint`](#1-safepoint--psafepoint)
   2. [`PJSError`](#2-pjserror)
II. [Working with the current exception](#ii-working-with-the-current-exception)
   1. [`getCurrentException`](#1-getcurrentexception)
   2. [`getCurrentExceptionMsg`](#2-getcurrentexceptionmsg)
   3. [`setCurrentException`](#3-setcurrentexception)
III. [Raising exceptions and stack traces](#iii-raising-exceptions-and-stack-traces)
   1. [`getStackTrace` / `writeStackTrace`](#1-getstacktrace--writestacktrace)
   2. [`raiseException`](#2-raiseexception)
   3. [`raiseDefect` / `reraiseException`](#3-raisedefect--reraiseexception)
   4. [`raiseOverflow` / `raiseDivByZero` / `raiseRangeError` / `raiseIndexError` / `raiseFieldError2`](#4-raiseoverflow--raisedivbyzero--raiserangeerror--raiseindexerror--raisefielderror2)
IV. [Converting strings between Nim and JavaScript](#iv-converting-strings-between-nim-and-javascript)
   1. [`nimBoolToStr` / `nimCharToStr`](#1-nimbooltostr--nimchartostr)
   2. [`makeNimstrLit`](#2-makenimstrlit)
   3. [`cstrToNimstr`](#3-cstrtonimstr)
   4. [`toJSStr`](#4-tojsstr)
V. [Set operations (`set`)](#v-set-operations-set)
   1. [`setConstr`](#1-setconstr)
   2. [`SetCard`](#2-setcard)
   3. [`SetEq` / `SetLe` / `SetLt`](#3-seteq--setle--setlt)
   4. [`SetMul` / `SetPlus` / `SetMinus` / `SetXor`](#4-setmul--setplus--setminus--setxor)
VI. [String comparison](#vi-string-comparison)
   1. [`cmpStrings` / `cmp`](#1-cmpstrings--cmp)
   2. [`eqStrings`](#2-eqstrings)
VII. [Overflow-checked arithmetic](#vii-overflow-checked-arithmetic)
   1. [`checkOverflowInt` and `addInt` / `subInt` / `mulInt` / `divInt` / `modInt`](#1-checkoverflowint-and-addint--subint--mulint--divint--modint)
   2. [64-bit variants](#2-64-bit-variants)
   3. [`negInt` / `absInt` / `nimMin` / `nimMax`](#3-negint--absint--nimmin--nimmax)
VIII. [Copying values (`nimCopy`)](#viii-copying-values-nimcopy)
   1. [`isFatPointer`](#1-isfatpointer)
   2. [`nimCopy` / `nimCopyAux`](#2-nimcopy--nimcopyaux)
   3. [`arrayConstr`](#3-arrayconstr)
IX. [Checking indices, ranges, and object types](#ix-checking-indices-ranges-and-object-types)
   1. [`chckIndx` / `chckRange`](#1-chckindx--chckrange)
   2. [`chckObj` / `isObj`](#2-chckobj--isobj)
   3. [`chckNilDisp`](#3-chcknildisp)
X. [Output (`echo`) and appending to a string](#x-output-echo-and-appending-to-a-string)
   1. [`rawEcho`](#1-rawecho)
   2. [`addChar` / `nimAddStrStr`](#2-addchar--nimaddstrstr)
XI. [Parsing floating-point numbers](#xi-parsing-floating-point-numbers)
   1. [`parseFloatNative`](#1-parsefloatnative)
   2. [`nimParseBiggestFloat`](#2-nimparsebiggestfloat)
XII. [Practical recipes](#xii-practical-recipes)
XIII. [Quick reference table](#xiii-quick-reference-table)
XIV. [Summary: which procedure to choose](#xiv-summary-which-procedure-to-choose)

---

## I. Types and exception infrastructure

### 1. `SafePoint` / `PSafePoint`

```nim
type
  PSafePoint = ptr SafePoint
  SafePoint {.compilerproc, final.} = object
    prev: PSafePoint
    exc: ref Exception
```

**What it does.** A node in the singly linked list of "safe points" — places in the stack that can be returned to while handling an exception (the analogue of a `try` frame). The `prev` field points to the previous safe point, `exc` holds the exception caught for the current block.

**Implementation walkthrough.** The list grows and collapses together with nested `try` blocks: entering a `try` prepends a new `SafePoint` to the list, leaving it removes that node. This gives O(1) for entering/leaving a block and O(nesting depth) in the worst case when searching for a handler.

**Parameters.**
* `prev: PSafePoint` — pointer to the outer (parent) `try` block; `nil` if the current block is the outermost one.
* `exc: ref Exception` — the exception active within this safe point.

---

### 2. `PJSError`

```nim
type
  PJSError = ref object
    columnNumber {.importc.}: int
    fileName {.importc.}: cstring
    lineNumber {.importc.}: int
    message {.importc.}: cstring
    stack {.importc.}: cstring
```

**What it does.** A Nim projection onto the built-in JavaScript error object (`Error`/`TypeError`/etc.) that lets Nim code read its fields (`message`, `fileName`, `stack`), even when the exception was thrown by the JS engine itself rather than by a Nim program.

**Implementation walkthrough.** All fields are marked `{.importc.}` — that is, the object has no layout of its own; the compiler simply accesses same-named fields on an existing JS object. Thanks to this, the same `lastJSError` pointer can be interpreted either as a `PJSError` (a native error) or as a `ref Exception` (a Nim error) — the choice depends on the result of `isNimException()`.

**Parameters.**
* `columnNumber: int` — the column number where the error occurred (if the engine provides it).
* `fileName: cstring` — the name of the file containing the error.
* `lineNumber: int` — the line number.
* `message: cstring` — the error message text.
* `stack: cstring` — the raw stack trace produced by the JS engine.

---

## II. Working with the current exception

### 1. `getCurrentException`

```nim
proc getCurrentException*(): ref Exception {.compilerRtl, gcsafe.}
```

**What it does.** Returns the currently "active" exception — the one presently being handled inside an `except` block. If there is no active exception, or it originated from inside the JS engine rather than from Nim (see `isNimException`), it returns `nil`.

**Implementation walkthrough.** The procedure keeps no state of its own — it reads the global variable `lastJSError` and simply reinterprets it as `ref Exception`, once `isNimException()` has confirmed that it is a Nim object and not a raw JS error.

**Parameters.** None.

**Examples.**

```nim
try:
  raise newException(ValueError, "something went wrong")
except ValueError:
  let e = getCurrentException()
  echo e.msg  # prints "something went wrong"
```

```nim
# Edge case: outside a try/except block returns nil
let e = getCurrentException()
doAssert e.isNil  # there is no active exception outside try/except
```

---

### 2. `getCurrentExceptionMsg`

```nim
proc getCurrentExceptionMsg*(): string
```

**What it does.** Returns the message text of the current exception as a plain Nim string — without requiring the caller to check whether it is a Nim exception or a native JavaScript error.

**Implementation walkthrough.** It works along two branches: if `isNimException()` is true, it simply reads the `.msg` field of the Nim exception; otherwise it fetches the `message` field of the raw JS object via `{.emit.}` (because the `PJSError` type isn't guaranteed to apply to an arbitrary `lastJSError` value). If there is no message at all, it returns an empty string rather than `nil`.

**Parameters.** None.

**Examples.**

```nim
try:
  raise newException(IOError, "file not found")
except IOError:
  echo getCurrentExceptionMsg()  # prints "file not found"
```

```nim
# Edge case: message is absent
try:
  raise newException(ValueError, "")
except ValueError:
  let msg = getCurrentExceptionMsg()
  echo len(msg)  # prints 0, not a nil-access error
```

---

### 3. `setCurrentException`

```nim
proc setCurrentException*(exc: ref Exception)
```

**What it does.** Forcibly sets the given exception as "current" at the runtime level. Used by internal compiler mechanisms (for example, when working with closure iterators), not by ordinary user code.

**Implementation walkthrough.** It simply overwrites the global variable `lastJSError`, casting `exc` to its internal `PJSError` representation. No consistency check against the `try/except` stack is performed — the caller is responsible for keeping state coherent.

**Parameters.**
* `exc: ref Exception` — the exception that becomes "current" for subsequent calls to `getCurrentException`/`getCurrentExceptionMsg`.

**Examples.**

```nim
let customExc = newException(ValueError, "manually set exception")
setCurrentException(customExc)
echo getCurrentExceptionMsg()  # prints "manually set exception"
```

---

## III. Raising exceptions and stack traces

### 1. `getStackTrace` / `writeStackTrace`

```nim
proc getStackTrace*(): string
proc getStackTrace*(e: ref Exception): string
proc writeStackTrace()
```

**What it does.** The first variant returns a text trace of the current call stack (built from `framePtr`). The second variant returns the trace already saved on a specific exception (the `e.trace` field, filled in at `raise` time). `writeStackTrace` is the same as the first variant, but immediately prints the result via `echo`, trimming the trailing newline.

**Implementation walkthrough.** The trace is built by walking the linked list of `CallFrame`s (the `prev` field), with the frame buffer capped at 64 entries (`array[0..63, TempFrame]`) — an optimization to avoid allocating a dynamic structure on every call; if there are more than 64 frames, the older (bottom) frames are collapsed into a string like `(N calls omitted) ...`. Complexity is O(stack depth), but never more than 64 iterations when building the buffer.

**Parameters.**
* `e: ref Exception` — the exception whose already-saved trace (`e.trace`) is returned without walking the stack again.

**Examples.**

```nim
proc levelB() =
  echo getStackTrace()  # prints a multi-line trace with the names levelA, levelB

proc levelA() =
  levelB()

levelA()
```

```nim
# Edge case: the stack is empty (framePtr == nil)
# rawWriteStackTrace then returns "No stack traceback available"
```

---

### 2. `raiseException`

```nim
proc raiseException(e: ref Exception, ename: cstring) {.compilerproc, asmNoStackFrame.}
```

**What it does.** The main entry point for the `raise` statement in generated JS code: assigns the exception a type name, and if there is no active handler (`excHandler == 0`) immediately treats it as unhandled and terminates the program; otherwise it saves a stack trace and throws the exception as a native JS `throw`.

**Implementation walkthrough.** The `excHandler` variable is a counter of `try`-block nesting; `excHandler == 0` means the exception was raised outside any `try` and there is nobody to catch it — so `unhandledException` is called, which prints a message and also does a `throw`, but with error text meant for the console/runtime environment.

**Parameters.**
* `e: ref Exception` — the exception object to raise.
* `ename: cstring` — the exception's type name (stored in `e.name` for later diagnostics).

**Examples.**

```nim
# Typical case — the compiler inserts a call to raiseException automatically for raise:
try:
  raise newException(KeyError, "key not found")
except KeyError as e:
  echo e.name  # prints "KeyError"
```

---

### 3. `raiseDefect` / `reraiseException`

```nim
proc raiseDefect()
proc reraiseException()
```

**What it does.** `raiseDefect` checks whether the current exception is a descendant of `Defect` (i.e. not meant to be caught by an ordinary `except`), and if so, forcibly re-throws it, bypassing normal handling. `reraiseException` implements the argument-less `raise` statement inside an `except` block — it re-throws the exception that was already caught; if there is nothing to re-throw, it raises `ReraiseDefect`.

**Implementation walkthrough.** Both procedures rely on `isNimException()` to distinguish a Nim exception from a raw JS-engine error — otherwise accessing fields such as `.name` would fail as accessing a non-existent property.

**Parameters.** None.

**Examples.**

```nim
try:
  try:
    raise newException(ValueError, "internal error")
  except ValueError:
    raise  # this is reraiseException — re-throws the same exception
except ValueError as e:
  echo e.msg  # prints "internal error"
```

```nim
# Error case: re-raising with no active exception
doAssertRaises(ReraiseDefect):
  reraiseException()
```

---

### 4. `raiseOverflow` / `raiseDivByZero` / `raiseRangeError` / `raiseIndexError` / `raiseFieldError2`

```nim
proc raiseOverflow() {.noreturn.}
proc raiseDivByZero() {.noreturn.}
proc raiseRangeError() {.noreturn.}
proc raiseIndexError(i, a, b: int) {.noreturn.}
proc raiseFieldError2(f: string, discVal: string) {.noreturn.}
```

**What it does.** Five "named entry points" for standard Nim runtime errors: integer overflow (`OverflowDefect`), division by zero (`DivByZeroDefect`), a value out of its allowed range (`RangeDefect`), an index out of a container's bounds (`IndexDefect`), and access to an unavailable field of a variant object (`FieldDefect`). None of them ever returns control to the caller (`noreturn`) — each either raises an exception or, if it isn't caught, terminates the program.

**Implementation walkthrough.** These are thin wrappers around `raise newException(...)`; all the substantial logic (message formatting) is delegated to `formatErrorIndexBound`/`formatFieldDefect` from `system/indexerrors`. The compiler inserts calls to these procedures automatically before arithmetic, indexing, and field access — there is no need to write them by hand in ordinary code.

**Parameters.**
* `i, a, b: int` (for `raiseIndexError`) — the actual index `i` and the allowed bounds `a..b`.
* `f, discVal: string` (for `raiseFieldError2`) — the name of the requested field and the current value of the variant's discriminant.

**Examples.**

```nim
# Typical case — inserted automatically by the compiler:
var a = [1, 2, 3]
doAssertRaises(IndexDefect):
  discard a[10]  # the compiler inserts a call to raiseIndexError(10, 0, 2)
```

---

## IV. Converting strings between Nim and JavaScript

### 1. `nimBoolToStr` / `nimCharToStr`

```nim
proc nimBoolToStr(x: bool): string {.compilerproc.}
proc nimCharToStr(x: char): string {.compilerproc.}
```

**What it does.** Converts the simplest scalar values to a Nim string representation: `bool` to `"true"`/`"false"`, `char` to a one-character string. Used by the compiler when interpolating values in `$x` and `echo`.

**Implementation walkthrough.** A straightforward implementation with no JS involved: for `bool`, a conditional branch; for `char`, allocating a string of length 1 via `newString(1)` and writing the character at that index.

**Parameters.**
* `x: bool` / `x: char` — the source value to convert.

**Examples.**

```nim
echo nimBoolToStr(true)   # prints "true"
echo nimCharToStr('Z')    # prints "Z"
```

---

### 2. `makeNimstrLit`

```nim
proc makeNimstrLit(c: cstring): string {.asmNoStackFrame, compilerproc.}
```

**What it does.** Turns a JavaScript string literal (`cstring`) into the Nim string representation used by the JS-backend code generator — an array of character codes. Used by the compiler when inserting string constants into generated code; it is not called directly from user programs.

**Implementation walkthrough.** It builds a new JS array and copies each source character's `charCodeAt` into it — that is, it works at the level of 16-bit UTF-16 code units, without accounting for surrogate pairs. Full Unicode support (in particular, characters outside the BMP) is handled by the more elaborate `cstrToNimstr` procedure.

**Parameters.**
* `c: cstring` — the source JavaScript string whose character codes are to be transferred into the Nim representation.

---

### 3. `cstrToNimstr`

```nim
proc cstrToNimstr(c: cstring): string {.asmNoStackFrame, compilerproc.}
```

**What it does.** Converts a JS string into a Nim string with full re-encoding into UTF-8, including characters outside the Basic Multilingual Plane (BMP) that are encoded in UTF-16 as surrogate pairs.

**Implementation walkthrough.** It implements manual UTF-8 encoding on the fly: code points below 128 become a single byte as-is; below 2048, two bytes using the standard bit masks (`>> 6 | 192` and `& 63 | 128`); for surrogate pairs (the `0xD800..0xDFFF` range), the code point is assembled from the two 16-bit units using the formula `65536 + ((high & 1023) << 10 | (low & 1023))` and then split into four UTF-8 bytes. This is the classic bit-mask UTF-8 encoding algorithm, reimplemented by hand because not every JS environment at the time this module was written could be relied on for encoding support.

**Parameters.**
* `c: cstring` — the source JavaScript string (UTF-16) to be re-encoded into Nim's UTF-8 representation.

**Examples.**

```nim
# Practical scenario: obtaining a string from the browser's DOM/API and the
# reverse conversion into a Nim string is performed automatically by the
# compiler when assigning cstring to string — no manual call is needed.
```

---

### 4. `toJSStr`

```nim
proc toJSStr(s: string): cstring {.compilerproc.}
```

**What it does.** The reverse conversion: turns a Nim string (a byte array in UTF-8) into a JavaScript string, correctly decoding multi-byte UTF-8 sequences back into code points.

**Implementation walkthrough.** For ASCII bytes (`< 128`), `String.fromCharCode` is used directly. For the remaining bytes, a sequence of `%XX` escapes (in the style of `encodeURIComponent`) is accumulated until a byte that starts the next character is reached, and the whole accumulated sequence is then passed to `decodeURIComponent`, which restores the original Unicode character. If `decodeURIComponent` throws (an invalid byte sequence), the function falls back to plain concatenation of the escapes without decoding — a safeguard against crashing on corrupted data.

**Parameters.**
* `s: string` — the Nim string (a UTF-8 byte sequence) to be represented as a JavaScript string.

**Examples.**

```nim
let s = "Hello"
echo toJSStr(s)  # returns the cstring "Hello", suitable for passing to a JS API
```

```nim
# Edge case: empty string
doAssert len($toJSStr("")) == 0
```

---

## V. Set operations (`set`)

### 1. `setConstr`

```nim
proc setConstr() {.varargs, asmNoStackFrame, compilerproc.}
```

**What it does.** Builds the representation of a `set` literal as a JS object whose keys are the elements of the set. Called by the compiler when compiling constructs such as `{1, 2, 5..8}`.

**Implementation walkthrough.** Each argument is either a single element or a pair `[first, last]` describing a range (the compiler passes ranges as a nested two-element array). For a range, every value from `first` to `last` inclusive is written into the result as a key with value `true`; a set's representation here is essentially a hash-set backed by a JS object, not a bit vector.

**Parameters.** Accepts an arbitrary number of arguments (`varargs`) — individual set-element values or two-element arrays `[first, last]` for ranges.

**Examples.**

```nim
# The compiler turns a set literal into a call to setConstr:
let s = {1, 2, 5..8}
echo SetCard(cast[int](s))  # prints 6 (1, 2, 5, 6, 7, 8)
```

---

### 2. `SetCard`

```nim
proc SetCard(a: int): int {.compilerproc, asmNoStackFrame.}
```

**What it does.** Returns the cardinality of a set — the number of elements it contains. The `int` parameter type is fictitious (an actual set object is passed) — a quirk of how sets are represented in this backend.

**Implementation walkthrough.** A simple `for...in` walk over the keys of the JS object, counting them — O(size of the set), since sets here are not a fixed-size bit vector but a sparse hash structure.

**Parameters.**
* `a` — the set whose cardinality is computed (fictitious `int` type).

**Examples.**

```nim
let s = {2, 4, 6}
echo SetCard(cast[int](s))  # prints 3
```

```nim
# Edge case: empty set
let empty: set[0..7] = {}
echo SetCard(cast[int](empty))  # prints 0
```

---

### 3. `SetEq` / `SetLe` / `SetLt`

```nim
proc SetEq(a, b: int): bool {.compilerproc, asmNoStackFrame.}
proc SetLe(a, b: int): bool {.compilerproc, asmNoStackFrame.}
proc SetLt(a, b: int): bool {.compilerproc.}
```

**What it does.** Implements set-comparison operators: `SetEq` — equality (`a == b`), `SetLe` — non-strict inclusion (`a` is a subset of `b`, the `<=` operator), `SetLt` — strict inclusion (`a <= b` and `a != b`, the `<` operator).

**Implementation walkthrough.** `SetEq` checks inclusion in both directions — if every element of `a` is in `b` and vice versa, the sets are equal. `SetLe` performs only one of these two checks — only "every element of `a` is in `b`". `SetLt` doesn't duplicate the logic but is expressed in terms of the already-defined `SetLe` and `SetEq` — an idiomatic reuse of simpler operations to build a more complex one.

**Parameters.**
* `a, b` — the sets being compared (fictitious `int` type).

**Examples.**

```nim
let x = {1, 2, 3}
let y = {1, 2, 3, 4}
echo SetLe(cast[int](x), cast[int](y))  # prints true — x is a subset of y
echo SetLt(cast[int](x), cast[int](y))  # prints true — the inclusion is strict
echo SetEq(cast[int](x), cast[int](y))  # prints false — the sets are not equal
```

---

### 4. `SetMul` / `SetPlus` / `SetMinus` / `SetXor`

```nim
proc SetMul(a, b: int): int {.compilerproc, asmNoStackFrame.}
proc SetPlus(a, b: int): int {.compilerproc, asmNoStackFrame.}
proc SetMinus(a, b: int): int {.compilerproc, asmNoStackFrame.}
proc SetXor(a, b: int): int {.compilerproc, asmNoStackFrame.}
```

**What it does.** Four set-theoretic operations implementing the `*` (intersection), `+` (union), `-` (difference), and `xor` (symmetric difference) operators for the `set` type.

**Implementation walkthrough.** All four are non-mutating, returning-a-new-value operations: an empty `result` object is created and then populated according to its own rule — `SetMul` keeps elements of `a` that are also in `b`; `SetPlus` unions all keys from both; `SetMinus` keeps elements of `a` that are not in `b`; `SetXor` keeps elements present in exactly one of the two sets. All four operations run in O(size of `a` + size of `b`).

**Parameters.**
* `a, b` — the operands of the set-theoretic operation (fictitious `int` type).

**Examples.**

```nim
let x = {1, 2, 3}
let y = {2, 3, 4}
echo SetCard(cast[int](SetMul(cast[int](x), cast[int](y))))    # prints 2 ({2, 3})
echo SetCard(cast[int](SetPlus(cast[int](x), cast[int](y))))   # prints 4 ({1,2,3,4})
echo SetCard(cast[int](SetMinus(cast[int](x), cast[int](y))))  # prints 1 ({1})
echo SetCard(cast[int](SetXor(cast[int](x), cast[int](y))))    # prints 2 ({1, 4})
```

---

## VI. String comparison

### 1. `cmpStrings` / `cmp`

```nim
proc cmpStrings(a, b: string): int {.asmNoStackFrame, compilerproc.}
proc cmp(x, y: string): int
```

**What it does.** Returns a negative number, zero, or a positive number depending on whether string `a` is less than, equal to, or greater than string `b` in lexicographic order — the standard three-way comparator contract.

**Implementation walkthrough.** `cmpStrings` is a direct character-by-character comparison with an early exit on the first mismatch; if the common part matches, the shorter string is considered "smaller" (lengths are compared). `cmp` is a language-level wrapper: at compile time (`nimvm`, i.e. inside the compile-time VM) it uses the built-in `==`/`<` operators for strings, while in compiled JS code it delegates to `cmpStrings` — this is needed because `cmpStrings` (with its `{.emit.}` body) is not available during compilation.

**Parameters.**
* `a, b` / `x, y: string` — the strings being compared.

**Examples.**

```nim
echo cmp("apple", "banana")  # prints a negative number ("apple" < "banana")
echo cmp("banana", "apple")  # prints a positive number
echo cmp("nim", "nim")       # prints 0
```

---

### 2. `eqStrings`

```nim
proc eqStrings(a, b: string): bool {.asmNoStackFrame, compilerproc.}
```

**What it does.** Checks two strings for equality. Implements the `==` operator for `string` in generated code.

**Implementation walkthrough.** It separately handles the case where one of the strings is `nil` (represented in the JS backend as `null` rather than an empty array of codes) while the other is an empty string: by Nim convention they are considered equal. This is a subtlety that distinguishes the behavior from a "naive" `a === b` comparison, which would give `false` for a pair of `null` and `[]`.

**Parameters.**
* `a, b: string` — the strings being compared.

**Examples.**

```nim
echo eqStrings("hello", "hello")  # prints true
```

```nim
# Edge case: a nil string and an empty string are considered equal
var a: string
let b = ""
echo eqStrings(a, b)  # prints true
```

---

## VII. Overflow-checked arithmetic

### 1. `checkOverflowInt` and `addInt` / `subInt` / `mulInt` / `divInt` / `modInt`

```nim
proc checkOverflowInt(a: int) {.asmNoStackFrame, compilerproc.}
proc addInt(a, b: int): int {.asmNoStackFrame, compilerproc.}
proc subInt(a, b: int): int {.asmNoStackFrame, compilerproc.}
proc mulInt(a, b: int): int {.asmNoStackFrame, compilerproc.}
proc divInt(a, b: int): int {.asmNoStackFrame, compilerproc.}
proc modInt(a, b: int): int {.asmNoStackFrame, compilerproc.}
```

**What it does.** Implements the `+`, `-`, `*`, `div`, `mod` arithmetic operators for a 32-bit `int` the way Nim semantics require: with overflow checking (raising `OverflowDefect`) and division-by-zero checking (raising `DivByZeroDefect`) — unlike JavaScript's "raw" operators, which silently work with double-precision floats.

**Implementation walkthrough.** JavaScript has no native 32-bit integers — numbers are represented as `double`, so after every operation the result must be explicitly checked against the `[-2147483648, 2147483647]` bounds via `checkOverflowInt`. For `div`/`mod`, an additional classic edge case is checked — integer division overflow when dividing `INT_MIN` by `-1`, which doesn't fit in the `int32` range. Rounding toward zero is ensured via `Math.trunc` rather than `Math.floor`, to match Nim's semantics (unlike Python-style floor division).

**Parameters.**
* `a: int` (in `checkOverflowInt`) — the value whose bounds are checked.
* `a, b: int` (in the rest) — the operands of the arithmetic operation.

**Examples.**

```nim
echo addInt(2, 3)   # prints 5
echo divInt(7, 2)    # prints 3 (rounding toward zero via Math.trunc)
echo modInt(-7, 2)   # prints -1 (the result's sign follows the dividend)
```

```nim
# Error case: overflowing the upper bound of int32
doAssertRaises(OverflowDefect):
  discard addInt(2147483647, 1)
```

```nim
# Error case: division by zero
doAssertRaises(DivByZeroDefect):
  discard divInt(10, 0)
```

---

### 2. 64-bit variants

```nim
proc checkOverflowInt64(a: int64) {.asmNoStackFrame, compilerproc.}
proc addInt64(a, b: int64): int64 {.asmNoStackFrame, compilerproc.}
proc subInt64(a, b: int64): int64 {.asmNoStackFrame, compilerproc.}
proc mulInt64(a, b: int64): int64 {.asmNoStackFrame, compilerproc.}
proc divInt64(a, b: int64): int64 {.asmNoStackFrame, compilerproc.}
proc modInt64(a, b: int64): int64 {.asmNoStackFrame, compilerproc.}
```

**What it does.** A complete counterpart of the `addInt`/`subInt`/`mulInt`/`divInt`/`modInt` group, but for the 64-bit `int64` — with the same overflow- and division-by-zero-checking semantics.

**Implementation walkthrough.** The only difference from the 32-bit versions is the JS type used: instead of ordinary `double` numbers, native `BigInt` is used (literals with the `n` suffix, e.g. `9223372036854775807n`), since `double` cannot exactly represent the whole `int64` range. Overflow bounds and the special `INT64_MIN / -1` case are checked by the same principles as the 32-bit version, just with `int64`-range values.

**Parameters.** Same as the `int` variant group, but with type `int64`.

**Examples.**

```nim
echo addInt64(1000000000000'i64, 1'i64)  # prints 1000000000001
```

```nim
# Error case: int64 overflow
doAssertRaises(OverflowDefect):
  discard addInt64(9223372036854775807'i64, 1'i64)
```

---

### 3. `negInt` / `absInt` / `nimMin` / `nimMax`

```nim
proc negInt(a: int): int {.compilerproc.}
proc negInt64(a: int64): int64 {.compilerproc.}
proc absInt(a: int): int {.compilerproc.}
proc absInt64(a: int64): int64 {.compilerproc.}
proc nimMin(a, b: int): int {.compilerproc.}
proc nimMax(a, b: int): int {.compilerproc.}
```

**What it does.** `negInt`/`negInt64` — unary minus; `absInt`/`absInt64` — absolute value; `nimMin`/`nimMax` — minimum and maximum of a pair of values.

**Implementation walkthrough.** Unlike the previous group, there is no overflow check here (and hence no `{.emit.}`-based implementation — it is expressed in plain Nim code): within 32-/64-bit bounds, unary minus and absolute value cannot go out of range other than through a separately checked addition/multiplication, and `min`/`max` cannot overflow at all, since it simply returns one of the input arguments without computation.

**Parameters.**
* `a: int` / `a: int64` — the value whose negation or absolute value is computed.
* `a, b: int` — the pair of values to compare.

**Examples.**

```nim
echo negInt(5)       # prints -5
echo absInt(-12)     # prints 12
echo nimMin(3, 7)    # prints 3
echo nimMax(3, 7)    # prints 7
```

---

## VIII. Copying values (`nimCopy`)

### 1. `isFatPointer`

```nim
proc isFatPointer(ti: PNimType): bool
```

**What it does.** Determines whether a pointer of the given type is "fat" — that is, represented in JS not as a single value but as a pair `[object, offset]` (needed for pointers into fields of composite types, not just to the start of an object).

**Implementation walkthrough.** It checks the kind of the base type (`ti.base.kind`): if the base type is an object, array, tuple, open array, set, `var`, or another pointer, an ordinary reference is deemed sufficient; for everything else (primarily primitives), a fat pointer is required so that a specific field/element can be addressed.

**Parameters.**
* `ti: PNimType` — runtime type information (RTTI) for the pointer's type.

---

### 2. `nimCopy` / `nimCopyAux`

```nim
proc nimCopy(dest, src: JSRef, ti: PNimType): JSRef
proc nimCopyAux(dest, src: JSRef, n: ptr TNimNode)
```

**What it does.** `nimCopy` implements deep (value) copying of Nim values according to their type — this is what lies behind assignment `a = b` for value-semantics types (objects, tuples, arrays, sequences, strings, sets), as opposed to plain reference copying, which is what "raw" JavaScript would do on `a = b`. `nimCopyAux` is a helper that walks the fields of a composite type (object/tuple) using its `TNimNode` tree.

**Implementation walkthrough.** The implementation is a large `case` on the type's kind (`ti.kind`):
- pointers/`ref`/`var` are copied as a reference, or, if it's a "fat" pointer, as a new `[obj, offset]` pair;
- sets (`tySet`) are copied by transferring all keys into an existing or new JS object, clearing `dest`'s old contents first;
- objects and tuples are copied field by field via `nimCopyAux`, which recursively walks the type's structure (`nkSlot` — a single slot, `nkList` — a list of slots, `nkCase` — the variant part, walking the active branch);
- arrays, open arrays, and sequences are copied element by element, reusing the existing `dest` buffer if its length already matches — this avoids unnecessary allocations when copying repeatedly into the same destination;
- strings are copied via `slice(0)` — a shallow copy, but sufficient for strings (immutable code sequences).

The overall invariant of the whole procedure: `dest` is reused whenever possible (rather than recreated), which matters for performance when copying in a loop.

**Parameters.**
* `dest: JSRef` — the destination the data is copied into (may be `nil`, in which case a new object is created).
* `src: JSRef` — the source being copied.
* `ti: PNimType` — the RTTI description of the value's type, which determines which `case` branch applies.
* `n: ptr TNimNode` (for `nimCopyAux`) — the field-tree node processed at this step of the recursion.

**Examples.**

```nim
# Typical case — the compiler inserts nimCopy on object assignment:
type Point = object
  x, y: int

var a = Point(x: 1, y: 2)
var b = a  # under the hood — nimCopy(nil, a, ...)
b.x = 99
echo a.x  # prints 1 — a is unchanged, the copy was deep
```

---

### 3. `arrayConstr`

```nim
proc arrayConstr(len: int, value: JSRef, typ: PNimType): JSRef {.asmNoStackFrame, compilerproc.}
```

**What it does.** Builds a new array of length `len` where each element is an independent copy of `value` (rather than `len` references to the same object). Used by the compiler for constructs such as `array[5, Point]`, where an initial value needs to be replicated into every slot.

**Implementation walkthrough.** A naive "fill with the same value" implementation would be incorrect for composite types — every slot would point at the same object, and mutating one would "leak" into the rest. That's why `nimCopy(null, value, typ)` is called on every iteration, creating an independent copy.

**Parameters.**
* `len: int` — the length of the array to create.
* `value: JSRef` — the sample value copied into each slot.
* `typ: PNimType` — the RTTI type of the value, passed to `nimCopy` to choose the copying strategy.

**Examples.**

```nim
type Point = object
  x, y: int

var arr = arrayConstr(3, Point(x: 1, y: 1), typeof(Point))
# each element of arr is an independent copy of Point(x: 1, y: 1)
```

---

## IX. Checking indices, ranges, and object types

### 1. `chckIndx` / `chckRange`

```nim
proc chckIndx(i, a, b: int): int {.compilerproc.}
proc chckRange(i, a, b: int): int {.compilerproc.}
```

**What it does.** Both check that `i` lies within the bounds `a..b` and return `i` unchanged if the check passes. They differ only in which exception they raise on violation: `chckIndx` raises `IndexDefect` (for container indexing), `chckRange` raises `RangeDefect` (for conversion to a range type like `range[0..10]`).

**Implementation walkthrough.** The compiler inserts a call to one of these functions before every operation for which the corresponding check is enabled (`--boundChecks`/`--rangeChecks`), wrapping the original index expression. Returning `i` itself on success allows the call to be used "transparently", substituted right in place of the original index in the generated code.

**Parameters.**
* `i: int` — the value being checked.
* `a, b: int` — the allowed lower and upper bounds (inclusive).

**Examples.**

```nim
var arr = [10, 20, 30]
echo arr[chckIndx(1, 0, 2)]  # prints 20 — this is how the compiler checks arr[1]
```

```nim
# Error case: index out of bounds
doAssertRaises(IndexDefect):
  discard chckIndx(5, 0, 2)
```

---

### 2. `chckObj` / `isObj`

```nim
proc chckObj(obj, subclass: PNimType)
proc isObj(obj, subclass: PNimType): bool
```

**What it does.** Checks the "is a subtype of" relationship between object type RTTIs. `isObj` returns the boolean result of the check; `chckObj` performs the same check but, instead of returning a boolean, raises `ObjectConversionDefect` if the check fails — this is what lies behind the object type-cast operator `Foo(x)`.

**Implementation walkthrough.** Both procedures walk up the `base` chain from `obj` through the inheritance hierarchy, comparing against `subclass` at each step; there's a fast path for an exact match that skips walking the hierarchy. Complexity is O(depth of the inheritance hierarchy).

**Parameters.**
* `obj: PNimType` — the RTTI of the value's actual (dynamic) type.
* `subclass: PNimType` — the RTTI of the type the check/cast is performed against.

**Examples.**

```nim
type
  Animal = ref object of RootObj
  Dog = ref object of Animal

let a: Animal = Dog()
echo isObj(a.type, Dog.type)  # prints true — a's dynamic type is Dog (illustrative)
```

```nim
# Error case: casting to an incompatible type raises ObjectConversionDefect
```

---

### 3. `chckNilDisp`

```nim
proc chckNilDisp(p: JSRef) {.compilerproc.}
```

**What it does.** Checks that the virtual-call dispatcher (`p`, a pointer to the method table) is not `nil` before dynamic method dispatch — that is, it guards against calling a method on an object whose dispatcher was not properly initialized.

**Implementation walkthrough.** A simple `nil` check that raises `NilAccessDefect` if the dispatcher is missing — this prevents a less-understandable error deep inside the JS engine from accessing an `undefined` field.

**Parameters.**
* `p: JSRef` — the dispatcher pointer whose validity is being checked.

---

## X. Output (`echo`) and appending to a string

### 1. `rawEcho`

```nim
proc rawEcho() {.compilerproc.}
```

**What it does.** The low-level implementation of the `echo` statement — prints the given arguments (already converted to strings via `toJSStr`), separating them and terminating the line with a newline, depending on the runtime environment.

**Implementation walkthrough.** The procedure has three alternative implementations, selected via conditional compilation for a specific runtime environment:
- under the `kwin` flag — output through the global `print` function (KWin/Plasma environment);
- by default (browser/Node.js) — via `console.log`;
- under the `nimOldEcho` flag — the old way of writing directly into the DOM, appending text nodes and `<br>` to `<body>`, with an explicit check that `<body>` already exists (otherwise an exception is raised — output before the DOM is ready is not possible).

As an exception to the general prefix-call rule, `echo` itself is used without parentheses or a dot — this is the formatting convention of this reference (see the "Code formatting requirements" section in the template).

**Parameters.** Accepts an arbitrary number of arguments (`varargs`) — the values to print, already serialized into `cstring`.

**Examples.**

```nim
echo "Value: ", 42  # prints "Value: 42" to the console
```

---

### 2. `addChar` / `nimAddStrStr`

```nim
proc addChar(x: string, c: char) {.compilerproc, asmNoStackFrame.}
proc nimAddStrStr(x, y: string) {.compilerproc, asmNoStackFrame.}
```

**What it does.** `addChar` appends a single character to the end of string `x` in place (a mutating operation). `nimAddStrStr` appends all characters of string `y` to the end of `x`, also mutating `x` in place. Both implement the `&=`/built-in `add` operator for strings at the backend level.

**Implementation walkthrough.** In the JS backend, a string is represented as an array of character codes, so "appending" is simply a `push` onto the existing array, with no need to recreate the string as a whole; `nimAddStrStr` does this in a loop over all elements of `y`. Complexity is O(1) amortized and O(length of `y`), respectively.

**Parameters.**
* `x: string` — the destination string, mutated in place.
* `c: char` — the character being appended.
* `y: string` — the string whose contents are appended to `x`.

**Examples.**

```nim
var s = "Hello"
add(s, ", world!")
echo s  # prints the string "Hello, world!"
```

---

## XI. Parsing floating-point numbers

### 1. `parseFloatNative`

```nim
proc parseFloatNative(a: openarray[char]): float
```

**What it does.** Parses a character slice as a floating-point number, using JavaScript's built-in `Number` constructor — that is, the actual parsing is delegated entirely to the JS engine rather than implemented by hand in Nim.

**Implementation walkthrough.** The characters are first assembled into an ordinary Nim string, then converted to `cstring` and passed to `Number(...)`. This is the simplest way to obtain a `float`, but it gives Nim no control over syntax details (for example, support for `_` as a digit-group separator) — that is handled by the calling procedure `nimParseBiggestFloat`, which already cleans up the string before passing it here.

**Parameters.**
* `a: openarray[char]` — the character slice representing the number.

---

### 2. `nimParseBiggestFloat`

```nim
proc nimParseBiggestFloat(s: openarray[char], number: var BiggestFloat): int {.compilerproc.}
```

**What it does.** A full parse of a floating-point number from a character slice `s`, supporting a sign, the special values `nan`/`inf`, digit-group separators (`_`), and exponential notation. Writes the result to `number` (an out-parameter) and returns the number of characters consumed; returns `0` if parsing fails (the string is not a valid number).

**Implementation walkthrough.** The function walks `s` character by character, accumulating a "cleaned" version of the number in `buf` (stripping `_` separators along the way) while advancing the index `i`. The special values `nan`/`inf`/`-inf` are recognized from their first letters without going through `parseFloatNative`. For ordinary numbers, the integer part is read first, then (if there is a dot) the fractional part, then (if there is an `e`/`E`) the exponent; at each step the code checks that the string still looks like a valid number after that step, otherwise parsing is aborted with a result of `0`. The final cleaned string is passed to `parseFloatNative`.

**Parameters.**
* `s: openarray[char]` — the source character slice to parse.
* `number: var BiggestFloat` — an out-parameter: the result is written here on a successful parse.

**Examples.**

```nim
var num: float
let consumed = nimParseBiggestFloat("3.14", num)
echo num       # prints 3.14
echo consumed  # prints 4 — all 4 characters were consumed
```

```nim
# Practical scenario: digit-group separators and an exponent
var big: float
discard nimParseBiggestFloat("1_000.5e2", big)
echo big  # prints 100050.0
```

```nim
# Edge case: a special value
var n: float
discard nimParseBiggestFloat("nan", n)
echo n  # prints nan
```

---

## XII. Practical recipes

### Safely reading a failure message of unknown origin

```nim
proc describeFailure(action: proc()): string =
  ## Runs action and returns a human-readable description of the failure,
  ## regardless of whether a Nim exception or a raw JS error was thrown.
  try:
    action()
    result = "completed without errors"
  except:
    result = "error: " & getCurrentExceptionMsg()

echo describeFailure(proc() = raise newException(ValueError, "invalid data"))
# prints "error: invalid data"
```

### Diagnosing a failure with a full stack trace

```nim
proc withTrace(action: proc()) =
  ## Catches an exception, prints its message, and the stack trace saved
  ## on the exception — handy for logging in production code.
  try:
    action()
  except:
    let e = getCurrentException()
    echo "Failure: ", e.msg
    echo getStackTrace(e)

proc risky() = raise newException(IOError, "resource unavailable")
withTrace(proc() = risky())
```

### An integer accumulator with explicit overflow protection

```nim
proc safeAccumulate(values: openarray[int]): int =
  ## Sums the values using addInt directly, to get a controlled exception
  ## instead of a silently corrupted result.
  result = 0
  for v in values:
    result = addInt(result, v)

doAssertRaises(OverflowDefect):
  discard safeAccumulate([2147483647, 1])
```

### Comparing access sets with a readable diagnostic

```nim
proc describeAccessDiff(required, granted: set[0..31]): string =
  ## Shows which rights are missing, using SetMinus directly — as if
  ## explicitly explaining the difference to a user.
  let missing = required - granted
  if SetCard(cast[int](missing)) == 0:
    result = "all rights granted"
  else:
    result = "missing " & $SetCard(cast[int](missing)) & " right(s)"

echo describeAccessDiff({1, 2, 5}, {1, 5})  # prints "missing 1 right(s)"
```

### Robust parsing of numeric settings from a string config

```nim
proc readNumericSetting(raw: string, default: float): float =
  ## Parses a setting value; on an invalid format (0 characters consumed)
  ## returns the default value.
  var value: float
  let consumed = nimParseBiggestFloat(raw, value)
  result = if consumed == 0: default else: value

echo readNumericSetting("2_500.75", 0.0)  # prints 2500.75
echo readNumericSetting("not a number", 10.0)  # prints 10.0 — parsing failed
```

---

## XIII. Quick reference table

| Task | Mutates argument | Returns new seq/value |
|---|---|---|
| Read the current exception's message | no | `getCurrentExceptionMsg` |
| Get the current exception object | no | `getCurrentException` |
| Forcibly set the "current" exception | yes (global state) | `setCurrentException` |
| Get a stack trace (current / saved) | no | `getStackTrace` |
| Check / raise a range error | no | `chckRange` |
| Check / raise an index error | no | `chckIndx` |
| Overflow-checked add/subtract/multiply (32-bit) | no | `addInt` / `subInt` / `mulInt` |
| Same for 64-bit numbers | no | `addInt64` / `subInt64` / `mulInt64` |
| Integer division and remainder with checks | no | `divInt` / `modInt` |
| Cardinality of a set | no | `SetCard` |
| Intersection / union / difference / symmetric difference of sets | no | `SetMul` / `SetPlus` / `SetMinus` / `SetXor` |
| Compare strings (lexicographically) | no | `cmp` / `cmpStrings` |
| Check strings for equality | no | `eqStrings` |
| Deep-copy a value of any type | yes (`dest`, if given) | `nimCopy` |
| Create an array of N independent copies of a value | no | `arrayConstr` |
| Check "is a subtype of" | no | `isObj` (boolean) / `chckObj` (raises) |
| Append a character/string to the end of a string | yes | `addChar` / `nimAddStrStr` |
| Parse a floating-point number from a string | no | `nimParseBiggestFloat` |

---

## XIV. Summary: which procedure to choose

- Need the error text inside an `except` block → use `getCurrentExceptionMsg`.
- Need the exception object itself (e.g. to check its type) → use `getCurrentException`.
- Need to log not just the message but also the call path → use `getStackTrace(e)`.
- Need to re-raise an exception without creating a new one → that's the plain `raise` statement (backed by `reraiseException`).
- Need arithmetic that's guaranteed not to "stay silent" on overflow → use `addInt`/`subInt`/`mulInt`/`divInt`/`modInt` (or their `Int64` variants) directly if you're writing low-level code.
- Need to know how many elements a set has → use `SetCard`.
- Need to understand the difference between two sets of rights/flags → use `SetMinus` (what's in one but not the other) or `SetXor` (what differs either way).
- Need to copy a composite value so that changing the copy doesn't affect the original → that's what `nimCopy` does, inserted by the compiler on assignment — no need to call it by hand unless you're writing code tightly integrated with the runtime.
- Need to turn a string from a config file/user input into a number without crashing on garbage data → use `nimParseBiggestFloat` and check the returned character count against `0`.
- Need to check that an object is an instance of the right subtype before casting → use `isObj` for a soft check, or rely on the built-in type cast (`chckObj` under the hood) for a check that raises.
