# Nim `system` Module — complete reference

> The `system` module is Nim's **implicit standard library**. It is automatically imported into every Nim file — you never need to write `import system`. Everything documented here is available out of the box. Many procs are marked `magic`, meaning the compiler handles them with special optimisation or code generation rather than ordinary Nim code.

---

## Table of Contents

1. [Type System Utilities](#1-type-system-utilities)
2. [Memory & Allocation](#2-memory--allocation)
3. [Ordinal & Bounds](#3-ordinal--bounds)
4. [Sequence Operations](#4-sequence-operations)
5. [String Operations](#5-string-operations)
6. [Arithmetic & Conversion](#6-arithmetic--conversion)
7. [Comparison & Containment](#7-comparison--containment)
8. [Control Flow & Program Lifecycle](#8-control-flow--program-lifecycle)
9. [I/O & Output](#9-io--output)
10. [Exception Handling](#10-exception-handling)
11. [Move Semantics & Lifecycle Hooks](#11-move-semantics--lifecycle-hooks)
12. [Low-Level & Unsafe Utilities](#12-low-level--unsafe-utilities)
13. [Metaprogramming & Debugging](#13-metaprogramming--debugging)
14. [Important Types & Constants](#14-important-types--constants)

---

## 1. Type System Utilities

### `typeof(x)`
```
proc typeof(x: untyped; mode = typeOfIter): typedesc
```

Returns the **compile-time type** of expression `x`. This is the standard way to ask "what type does this expression have?" without evaluating `x`. The optional `mode` parameter controls how overloaded names that could be either an iterator or a procedure are resolved.

**Why it matters:** When writing generic code, you often need to derive a type from an expression rather than specify it manually. `typeof` makes this possible.

```nim
var n = 42
echo typeof(n)          # int

proc foo(): float = 1.0
echo typeof(foo())      # float (typeOfProc perspective)
```

---

### `is` / `isnot`
```
proc `is`[T, S](x: T, y: S): bool
template `isnot`(x, y: untyped): untyped
```

`is` checks whether value `x` has the **same type** as `y` (which can be a type or a value). `isnot` is the negation. These are evaluated at **compile time** inside `when` branches, making them the primary tool for type-based dispatch in generic code.

```nim
assert 42 is int
assert "hello" isnot int

proc process[T](x: T) =
  when T is string:
    echo "Got a string: ", x
  else:
    echo "Got something else"
```

---

### `of`
```
proc `of`[T, S](x: T, y: typedesc[S]): bool
```

Checks whether an object or reference `x` is **an instance of** (or inherits from) type `S`. This is Nim's equivalent of `instanceof` in Java or dynamic type checks in other OO languages. Unlike `is`, this is a **runtime** check for object variants.

```nim
type
  Animal = ref object of RootObj
  Dog    = ref object of Animal

var a: Animal = Dog()
assert a of Animal   # true — Dog IS-A Animal
assert a of Dog      # true — runtime type is Dog
```

---

### `default[T]`
```
proc default[T](_: typedesc[T]): T
```

Returns the **default value** for type `T`. For integers this is `0`, for strings `""`, for sequences `@[]`, etc. Unlike `zeroDefault`, this respects any `defaultVal` annotations on object fields.

```nim
assert default(int)    == 0
assert default(string) == ""
assert default(bool)   == false

type Point = object
  x, y: float
assert default(Point) == Point(x: 0.0, y: 0.0)
```

---

### `zeroDefault[T]`
```
func zeroDefault[T](_: typedesc[T]): T
```

Similar to `default`, but always returns the **binary zero** representation, completely ignoring any default field values declared in object types. Useful in very low-level code where you need raw zeroed memory regardless of semantics.

```nim
type Config = object
  retries: int = 3   # default field value
assert zeroDefault(Config).retries == 0   # ignores the default
assert default(Config).retries     == 3   # respects the default
```

---

### `sizeof[T]`
```
proc sizeof[T](x: T): int
proc sizeof(x: typedesc): int
```

Returns the **size in bytes** of a value or type. Works with both values and type descriptors. Primarily useful when interfacing with C or doing low-level memory work.

```nim
echo sizeof(int)     # 8 on 64-bit systems
echo sizeof(char)    # 1
echo sizeof(float64) # 8
```

---

### `alignof[T]`
```
proc alignof[T](x: T): int
proc alignof(x: typedesc): int
```

Returns the **alignment requirement** (in bytes) of a type. Alignment is the boundary at which the type must start in memory. Necessary when manually constructing memory layouts or wrapping C structs.

```nim
echo alignof(int)    # typically 8 on 64-bit
echo alignof(char)   # 1
```

---

### `offsetOf`
```
template offsetOf[T](t: typedesc[T]; member: untyped): int
```

Returns the **byte offset** of a named field within a struct/object. This is the Nim equivalent of C's `offsetof` macro and is indispensable when you need to know exactly where a field sits in memory.

```nim
type Vec = object
  x: float32
  y: float32

echo offsetOf(Vec, x)  # 0
echo offsetOf(Vec, y)  # 4
```

---

## 2. Memory & Allocation

### `new[T]`
```
proc new[T](a: var ref T)
proc new(t: typedesc): auto
proc new[T](a: var ref T, finalizer: proc(x: ref T))
```

Allocates a new heap object of type `T` and stores a **traced (GC-managed) reference** in `a`. The traced reference ensures the garbage collector knows about the allocation. The optional `finalizer` callback runs just before the object is freed, but cannot prevent freeing.

```nim
type Node = object
  value: int
  next: ref Node

var n: ref Node
new(n)
n.value = 42
echo n.value   # 42
```

---

### `unsafeNew[T]`
```
proc unsafeNew[T](a: var ref T, size: Natural)
```

Like `new`, but allocates exactly `size` bytes instead of `sizeof(T)`. This allows allocating objects whose actual size is larger than the declared type — useful for C interop or variable-length types. **Use with extreme care.**

---

### `wasMoved[T]`
```
proc wasMoved[T](obj: var T)
```

Resets `obj` to binary zero to signal that its value has been **logically moved** elsewhere. After calling this, the object's destructor should be a no-op. Used internally by the move-semantics machinery.

---

### `deepCopy[T]`
```
proc deepCopy[T](x: out T, y: T)
proc deepCopy[T](y: T): T
```

Performs a **recursive, deep copy** of `y` into `x`, traversing reference chains. This is the opposite of a shallow copy, which only copies the top-level pointer. Required for use with `spawn` in parallel code. For ARC/ORC memory management, deep-copy support must be explicitly enabled with `--deepcopy:on`.

```nim
var a = @[@[1, 2], @[3, 4]]
var b = deepCopy(a)
b[0][0] = 99
assert a[0][0] == 1   # a is unaffected — true deep copy
```

---

## 3. Ordinal & Bounds

### `high` / `low`
```
proc high[T: Ordinal|enum|range](x: typedesc[T]): T
proc low[T: Ordinal|enum|range](x: typedesc[T]): T
proc high[T](x: openArray[T]): int
proc low[T](x: openArray[T]): int
proc high[I, T](x: array[I, T]): I
proc low[I, T](x: array[I, T]): I
proc high(x: string): int
proc low(x: string): int
```

`high` returns the **maximum valid value or index**; `low` returns the **minimum**. They work on types, arrays, sequences, strings, and other containers. These are the canonical way to write portable range loops.

```nim
echo high(int)      # 9223372036854775807
echo low(int8)      # -128

var s = @[10, 20, 30]
for i in low(s)..high(s):
  echo s[i]         # 10, 20, 30

echo high("hello")  # 4 (last index)
```

---

### `ord[T]`
```
func ord[T: Ordinal|enum](x: T): int
```

Converts an ordinal value (integer, bool, character, or enum) to its underlying `int` representation. Essential when you need the numeric value of an enum member or character.

```nim
echo ord('A')     # 65
echo ord(true)    # 1
echo ord(false)   # 0

type Color = enum red, green, blue
echo ord(green)   # 1
```

---

### `chr`
```
func chr(u: range[0..255]): char
```

Converts an integer in the range `0..255` to the corresponding `char`. This is the inverse of `ord` for characters. Raises `RangeDefect` if the value is out of range.

```nim
echo chr(65)    # A
echo chr(10)    # newline character
```

---

## 4. Sequence Operations

### `newSeq[T]`
```
proc newSeq[T](s: var seq[T], len: Natural)
proc newSeq[T](len = 0.Natural): seq[T]
```

Creates a new sequence of length `len` with all elements **zero-initialized**. This is more efficient than growing a sequence incrementally when the final size is known ahead of time.

```nim
var data = newSeq[int](5)
assert data.len == 5
data[0] = 100
# data == @[100, 0, 0, 0, 0]
```

---

### `newSeqOfCap[T]`
```
proc newSeqOfCap[T](cap: Natural): seq[T]
```

Creates an empty sequence (length 0) that has **pre-allocated capacity** for `cap` elements. Use this when you know roughly how many elements you will add but don't want to initialize them yet. Prevents repeated reallocations during growth.

```nim
var s = newSeqOfCap[string](100)
assert s.len == 0          # length is still 0
for i in 0..<50:
  s.add($i)               # no reallocation needed
```

---

### `newSeqUninit[T]`
```
func newSeqUninit[T](len: Natural): seq[T]
```

Like `newSeq` but the elements are **not initialized**. Only available for types without managed memory (plain value types like `int`, `float`, etc.). Use when you will assign every element immediately and want to avoid the cost of zero-initialization.

```nim
var buf = newSeqUninit[byte](1024)
# Fill immediately — do NOT read uninitialized slots
for i in 0..<buf.len:
  buf[i] = 0xFF.byte
```

---

### `setLen`
```
proc setLen[T](s: var seq[T], newlen: Natural)
proc setLen(s: var string, newlen: Natural)
```

Resizes a sequence or string. If `newlen` is greater than the current length, **new slots are zero-initialized**. If smaller, the sequence or string is **truncated**. Does not release the underlying allocation.

```nim
var x = @[1, 2, 3]
x.setLen(5)
assert x == @[1, 2, 3, 0, 0]

x.setLen(2)
assert x == @[1, 2]
```

---

### `add`
```
proc add[T](x: var seq[T], y: sink T)
proc add[T](x: var seq[T], y: openArray[T])
proc add(x: var string, y: char)
proc add(x: var string, y: string)
```

Appends an element or a collection to a sequence or string **in place**. This is the idiomatic way to grow a container one element at a time. Nim uses this name consistently for all container types.

```nim
var s: seq[int]
s.add(1)
s.add(2)
s.add([3, 4, 5])
assert s == @[1, 2, 3, 4, 5]

var t = "hello"
t.add(' ')
t.add("world")
assert t == "hello world"
```

---

### `del[T]`
```
proc del[T](x: var seq[T], i: Natural)
```

Removes the element at index `i` from the sequence in **O(1)** time by swapping it with the last element and then shrinking the length. **This does not preserve order.** If order matters, use `delete` instead.

```nim
var a = @[10, 20, 30, 40, 50]
a.del(1)
# The element at index 1 is replaced by the last element
assert a == @[10, 50, 30, 40]
```

---

### `delete[T]`
```
proc delete[T](x: var seq[T], i: Natural)
```

Removes the element at index `i` by shifting all subsequent elements one position left. This is **O(n)** and **preserves the original order**, unlike `del`.

```nim
var a = @[10, 20, 30, 40, 50]
a.delete(1)
assert a == @[10, 30, 40, 50]   # order preserved
```

---

### `insert[T]`
```
proc insert[T](x: var seq[T], item: sink T, i = 0.Natural)
```

Inserts `item` at position `i`, shifting all subsequent elements to the right. An O(n) operation. By default inserts at the beginning (position 0).

```nim
var a = @[1, 3, 5]
a.insert(99, 1)
assert a == @[1, 99, 3, 5]
```

---

### `pop[T]`
```
proc pop[T](s: var seq[T]): T
```

Removes and returns the **last element** of the sequence, treating it as a stack. Raises `IndexDefect` if the sequence is empty.

```nim
var stack = @[10, 20, 30]
let top = stack.pop()
assert top  == 30
assert stack == @[10, 20]
```

---

### `find[T]`
```
proc find[T, S](a: T, item: S): int
```

Searches container `a` for `item` and returns the **first index** where it is found, or **-1** if not found.

```nim
var a = @[5, 10, 15, 20]
assert a.find(15) == 2
assert a.find(99) == -1
```

---

### `contains[T]`
```
proc contains[T](a: openArray[T], item: T): bool
```

Returns `true` if `item` is present in `a`. This backs the `in` operator, so `x in a` and `a.contains(x)` are equivalent.

```nim
var a = @[1, 3, 5]
assert 3 in a
assert 99 notin a
```

---

### `@` (array/openArray to seq)
```
proc `@`[IDX, T](a: sink array[IDX, T]): seq[T]
proc `@`[T](a: openArray[T]): seq[T]
```

Converts an array (or openArray) to a sequence by copying all elements. This is how you write sequence literals: `@[1, 2, 3]` is syntactic sugar for converting the array literal `[1, 2, 3]` to a `seq[int]`.

```nim
let arr = [10, 20, 30]
let s   = @arr
assert s == @[10, 20, 30]
assert s is seq[int]
```

---

### `len`
```
func len[T](x: seq[T]): int
func len(x: string): int
func len[T](x: openArray[T]): int
func len(x: array): int
proc len[U, V](x: HSlice[U, V]): int
```

Returns the **number of elements** (or characters) in a container. For `HSlice`, returns `max(0, b - a + 1)`.

```nim
echo @[1, 2, 3].len   # 3
echo "hello".len      # 5
echo (2..5).len       # 4
```

---

## 5. String Operations

### `&` (string concatenation)
```
proc `&`(x, y: string): string
proc `&`(x: string, y: char): string
proc `&`(x, y: char): string
```

Creates a **new string** that is the concatenation of its arguments. All combinations of string and char are supported.

```nim
let s = "Hello" & ", " & "world" & '!'
assert s == "Hello, world!"
```

---

### `&=`
```
proc `&=`(x: var string, y: string)
```

Appends `y` to `x` **in place**, without creating a new string. More efficient than `x = x & y` because it avoids an extra allocation.

```nim
var s = "Hello"
s &= ", world"
assert s == "Hello, world"
```

---

### `newString`
```
proc newString(len: Natural): string
```

Allocates a new string of exactly `len` characters. The content is zero-initialized. You then fill it character by character via the index operator. Exists for optimization when the string length is known in advance.

```nim
var s = newString(5)
s[0] = 'h'; s[1] = 'e'; s[2] = 'l'; s[3] = 'l'; s[4] = 'o'
assert s == "hello"
```

---

### `newStringOfCap`
```
proc newStringOfCap(cap: Natural): string
```

Returns an empty string (`len == 0`) with pre-allocated internal capacity for `cap` bytes. Use to avoid reallocations when building a string of roughly known size.

---

### `newStringUninit`
```
proc newStringUninit(len: Natural): string
```

Like `newString` but the character content is **uninitialized** (garbage). Must assign every position before reading. Useful in performance-critical code that fills every byte immediately.

---

### `insert` (string)
```
proc insert(x: var string, item: string, i = 0.Natural)
```

Inserts `item` into string `x` at byte position `i`, shifting the rest of the string right. O(n) operation.

```nim
var s = "abcdef"
s.insert("XY", 2)
assert s == "abXYcdef"
```

---

### `substr`
```
proc substr(s: string; first, last: int): string
proc substr(s: string; first = 0): string
proc substr(a: openArray[char]): string
```

Extracts a **substring** of `s` from index `first` to `last` inclusive. Clamps both indices safely: negative `first` becomes 0, `last` beyond the string end is clamped to `high(s)`. Returns `""` when `first > last`.

```nim
let s = "Hello, world!"
echo s.substr(7, 11)   # world
echo s.substr(7)       # world!
echo s.substr(-5, 3)   # Hell  (negative first clamped to 0)
```

---

### `addEscapedChar`
```
proc addEscapedChar(s: var string, c: char)
```

Appends character `c` to `s` with **escape sequences** applied for special characters (`\n`, `\t`, `\\`, `\"`, `\'`, and non-printable bytes as `\xHH`). Used internally by `repr` and useful when building debug output.

```nim
var s = ""
s.addEscapedChar('\n')
s.addEscapedChar('A')
assert s == "\\nA"
```

---

### `addQuoted[T]`
```
proc addQuoted[T](s: var string, x: T)
```

Appends `x` to `s` with quoting appropriate to `x`'s type: strings get double quotes, chars get single quotes, and special characters are escaped. Numbers are appended as-is.

```nim
var s = ""
s.addQuoted("hi")
s.add(", ")
s.addQuoted('c')
assert s == "\"hi\", 'c'"
```

---

## 6. Arithmetic & Conversion

### `abs`
```
proc abs[T: float64|float32](x: T): T
func abs(x: int): int
# also int8, int16, int32, int64 variants
```

Returns the **absolute value** of `x`. For integers, raises an overflow exception if `x == low(T)` (since `-low(int)` is not representable). For floats, handles `NaN` and `-0.0` correctly.

```nim
echo abs(-42)    # 42
echo abs(-3.14)  # 3.14
echo abs(0)      # 0
```

---

### `toFloat` / `toBiggestFloat`
```
proc toFloat(i: int): float
proc toBiggestFloat(i: BiggestInt): BiggestFloat
```

Converts an integer to a floating-point number. Same as a type cast `float(i)` but may raise `ValueError` on platforms where the conversion is not lossless (rare in practice).

```nim
let x: int = 7
echo x.toFloat / 2.0   # 3.5
```

---

### `toInt` / `toBiggestInt`
```
proc toInt(f: float): int
proc toBiggestInt(f: BiggestFloat): BiggestInt
```

Converts a float to an integer, **rounding half away from zero** (0.5 → 1, -0.5 → -1). This is different from a type cast, which truncates toward zero.

```nim
echo toInt(2.5)    # 3
echo toInt(-2.5)   # -3
echo toInt(2.4)    # 2
```

---

### `/` (integer division to float)
```
proc `/`(x, y: int): float
```

Divides two integers and returns a **float** result. This is distinct from `div`, which does integer (floor) division. The result of `7 / 2` is `3.5`, not `3`.

```nim
echo 7 / 2     # 3.5
echo 10 / 4    # 2.5
```

---

### `swap[T]`
```
proc swap[T](a, b: var T)
```

Exchanges the values of `a` and `b` **in place**. Often more efficient than a manual three-step swap, especially for reference types.

```nim
var x = 10
var y = 20
swap(x, y)
assert x == 20
assert y == 10
```

---

### `cmp`
```
proc cmp[T](x, y: T): int
proc cmp(x, y: string): int
```

Returns a negative value if `x < y`, zero if `x == y`, and a positive value if `x > y`. The generic version uses `==` and `<`. The string version uses C's `strcmp` and is more efficient. Ideal for use with sorting functions.

```nim
import std/algorithm
var nums = @[3, 1, 4, 1, 5]
nums.sort(cmp[int])
assert nums == @[1, 1, 3, 4, 5]
```

---

## 7. Comparison & Containment

### `contains` (slice)
```
proc contains[U, V, W](s: HSlice[U, V], value: W): bool
```

Checks if `value` lies within the inclusive range `[s.a, s.b]`. Enables the `in` operator on ranges.

```nim
assert 3 in 1..5       # true
assert 6 notin 1..5    # true
```

---

### `..` (slice constructor)
```
proc `..`[T, U](a: T, b: U): HSlice[T, U]
```

Constructs an inclusive range `[a, b]`. The resulting `HSlice` is used for array slicing, set constructors, and case statements.

```nim
let r = 2..7
echo r.a   # 2
echo r.b   # 7

let arr = [10, 20, 30, 40, 50]
echo arr[1..3]   # @[20, 30, 40]
```

---

### `isNil`
```
proc isNil[T](x: ref T): bool
proc isNil[T](x: ptr T): bool
proc isNil(x: pointer): bool
proc isNil(x: cstring): bool
```

Fast nil check for reference types, pointers, and closures. Often faster than comparing with `nil` directly since the compiler can sometimes lower it to a single instruction.

```nim
var p: ref int
assert p.isNil

new(p)
assert not p.isNil
```

---

## 8. Control Flow & Program Lifecycle

### `quit`
```
proc quit(errorcode: int = QuitSuccess)
proc quit(errormsg: string, errorcode = QuitFailure)
```

Terminates the program immediately with the given exit code (or first prints a message to stderr). Runs registered exit procedures before exiting. **Bypasses** `defer`, exception handlers, and destructors — use sparingly, especially in library code.

```nim
if criticalFileNotFound:
  quit("Cannot find required config file", QuitFailure)
```

---

### `newException`
```
template newException(exceptn: typedesc, message: string;
                      parentException: ref Exception = nil): untyped
```

Constructs a new exception object of the given type with its `msg` field set to `message`. The idiomatic way to raise exceptions in Nim.

```nim
raise newException(ValueError, "input must be positive")
```

---

### `getCurrentException`
```
proc getCurrentException(): ref Exception
```

Inside an `except` block, returns the **currently active exception object**, or `nil` if there is none. Use this to inspect the exception, chain exceptions, or re-raise.

```nim
try:
  raise newException(IOError, "disk full")
except IOError:
  let e = getCurrentException()
  echo e.msg   # "disk full"
```

---

### `getCurrentExceptionMsg`
```
proc getCurrentExceptionMsg(): string
```

Shortcut for `getCurrentException().msg`. Returns `""` if no exception is active.

---

### `setControlCHook`
```
proc setControlCHook(hook: proc() {.noconv.})
```

Overrides the default Ctrl+C behaviour of your application. The provided handler runs inside a C signal handler, where most Nim operations (including `echo`, `string`, and exceptions) are **unsafe**. Only use with atomic operations or flags.

```nim
import std/atomics
var shouldStop: Atomic[bool]

proc onCtrlC() {.noconv.} =
  shouldStop.store(true)

setControlCHook(onCtrlC)

while not shouldStop.load():
  doWork()
```

---

### `once`
```
template once(body: untyped)
```

Executes `body` **exactly once** across all calls to the surrounding code, regardless of how many times the surrounding scope is entered. Uses a global boolean flag under the hood.

```nim
proc render() =
  once:
    initGraphicsSubsystem()
  drawFrame()
```

---

### `closureScope`
```
template closureScope(body: untyped)
```

Wraps `body` in an immediately-invoked anonymous procedure, forcing all local variables to be captured **by value at the point of closure creation**. This solves the classic loop-closure variable capture problem.

```nim
var callbacks: seq[proc()]
for i in 0..3:
  closureScope:
    let j = i       # j is captured at THIS iteration's value
    callbacks.add(proc() = echo j)

for cb in callbacks: cb()   # prints 0, 1, 2, 3 (not all 3)
```

---

### `likely` / `unlikely`
```
template likely(val: bool): bool
template unlikely(val: bool): bool
```

Hints to the CPU branch predictor whether a boolean condition is expected to be `true` (`likely`) or `false` (`unlikely`) most of the time. On native targets these compile down to `__builtin_expect`. No effect on JS or NimScript backends.

```nim
for value in data:
  if likely(value >= 0):
    process(value)
  else:
    handleNegative(value)
```

---

### `procCall`
```
proc procCall(x: untyped)
```

Suppresses dynamic dispatch for method calls, forcing **static binding** — equivalent to `super.method()` in other OO languages. Only valid at compile time.

```nim
type
  Base = ref object of RootObj
  Child = ref object of Base

method greet(a: Base) {.base.} = echo "Hello from Base"
method greet(a: Child) =
  procCall greet(Base(a))   # call Base's implementation
  echo "...and Child"
```

---

## 9. I/O & Output

### `echo`
```
proc echo(x: varargs[typed, `$`])
```

Writes all arguments to stdout separated by nothing, then adds a newline and flushes the buffer. Each argument is converted to a string via its `$` operator. Thread-safe. Works on all backends including JavaScript.

```nim
echo "Count: ", 42, ", flag: ", true
# → Count: 42, flag: true
```

---

### `debugEcho`
```
proc debugEcho(x: varargs[typed, `$`])
```

Identical to `echo` but declared as having **no side effects**. This allows it to be called inside `func` (pure functions) and procs marked `{.noSideEffect.}`, where `echo` would be disallowed. Intended exclusively for debugging.

```nim
func pureComputation(x: int): int =
  debugEcho "computing with x = ", x   # allowed inside func
  x * x
```

---

### `writeStackTrace`
```
proc writeStackTrace()
```

Writes the current call stack to **stderr**. Only useful in debug builds (compiled without `-d:release`).

---

### `getStackTrace`
```
proc getStackTrace(): string
proc getStackTrace(e: ref Exception): string
```

Returns the current stack trace as a string (or the trace captured when exception `e` was raised). Debug builds only.

---

## 10. Exception Handling

### Exception Hierarchy (Types)

The key exception types exported by `system`:

| Type | Description |
|---|---|
| `Exception` | Root of all exceptions |
| `Defect` | Unrecoverable runtime errors (e.g. out-of-bounds) |
| `CatchableError` | Base for recoverable exceptions |
| `IOError` | Input/output failures |
| `ValueError` | Invalid values (e.g. bad format) |
| `IndexDefect` | Array/sequence index out of range |
| `RangeDefect` | Value outside a declared range |
| `OverflowDefect` | Integer arithmetic overflow |
| `DivByZeroDefect` | Division by zero |
| `NilAccessDefect` | Dereferencing `nil` |
| `ObjectConversionDefect` | Invalid object type cast |

```nim
try:
  var a = @[1, 2, 3]
  echo a[10]   # raises IndexDefect
except IndexDefect as e:
  echo "Caught: ", e.msg
```

---

### `globalRaiseHook` / `localRaiseHook`
```
var globalRaiseHook: proc(e: ref Exception): bool
var localRaiseHook {.threadvar.}: proc(e: ref Exception): bool
```

Hook procedures called for **every `raise` statement**. If the hook returns `false`, the exception is swallowed. `globalRaiseHook` is program-wide; `localRaiseHook` is per-thread. Intended for advanced diagnostics or logging frameworks.

---

### `outOfMemHook`
```
var outOfMemHook: proc()
```

Called by the runtime when a memory allocation fails. By default the program terminates. Set this to a custom procedure to raise a Nim exception instead (you must pre-allocate the exception object before OOM actually occurs).

---

## 11. Move Semantics & Lifecycle Hooks

### `move[T]`
```
proc move[T](x: var T): T
```

Transfers the value of `x` to the return value and resets `x` to its zero state. After `move`, `x` is in a valid but unspecified (zeroed) state. More efficient than copying when ownership transfer is intended.

```nim
var source = @[1, 2, 3, 4, 5]
var dest   = move(source)
assert source.len == 0      # source was reset
assert dest  == @[1, 2, 3, 4, 5]
```

---

### `ensureMove[T]`
```
proc ensureMove[T](x: T): T
```

Like `move` but **fails at compile time** if the compiler cannot guarantee an actual move (i.e., would fall back to a copy). Useful when you want to be certain no accidental copying happens.

```nim
var s = "hello world"
let t = ensureMove(s)    # compile error if copy would occur
```

---

### `=destroy` / `=copy` / `=sink` / `=dup` / `=trace`

These are **lifecycle hooks** that users can override for custom types to plug into Nim's ARC/ORC memory management system:

- `=destroy` — called when an object goes out of scope
- `=copy` — called on assignment (deep copy)
- `=sink` — called on move-assignment (ownership transfer)
- `=dup` — called to duplicate an object
- `=trace` — called by the cycle collector to trace pointers

```nim
type Resource = object
  handle: int

proc `=destroy`(r: Resource) =
  if r.handle != 0:
    closeHandle(r.handle)

proc `=copy`(dest: var Resource, src: Resource) =
  dest.handle = dupHandle(src.handle)
```

---

### `reset[T]`
```
proc reset[T](obj: var T)
```

Resets `obj` to its default value (calling its destructor if applicable). After `reset`, the object is in the same state as a freshly declared variable.

```nim
var s = @[1, 2, 3]
reset(s)
assert s.len == 0   # s is now @[]
```

---

## 12. Low-Level & Unsafe Utilities

### `addr[T]`
```
proc addr[T](x: T): ptr T
```

Takes the **memory address** of `x` and returns an untraced (raw) pointer to it. Unlike `ref`, a `ptr` is not tracked by the garbage collector. Never store the resulting pointer beyond the lifetime of `x`.

```nim
var n = 42
let p = addr(n)
p[] = 100
assert n == 100
```

---

### `toOpenArray`
```
proc toOpenArray[T](x: seq[T]; first, last: int): openArray[T]
proc toOpenArray[T](x: openArray[T]; first, last: int): openArray[T]
proc toOpenArray[I,T](x: array[I,T]; first, last: I): openArray[T]
proc toOpenArray(x: string; first, last: int): openArray[char]
```

Creates a **non-owning view (slice)** of a container between indices `first` and `last` inclusive. Avoids copying the data — the slice refers directly to the original storage. Useful for passing sub-ranges to procedures that accept `openArray[T]`.

```nim
proc sum(a: openArray[int]): int =
  for x in a: result += x

let s = @[0, 1, 2, 3, 4, 5]
echo sum(s.toOpenArray(1, 4))   # sums 1+2+3+4 = 10
```

---

### `cstringArrayToSeq`
```
proc cstringArrayToSeq(a: cstringArray, len: Natural): seq[string]
proc cstringArrayToSeq(a: cstringArray): seq[string]
```

Converts a C-style null-terminated array of C strings into a Nim `seq[string]`. The second form auto-detects the length by scanning for the terminating `nil` pointer.

```nim
# Typically used when wrapping C APIs that return char**
let args = cstringArrayToSeq(rawArgv, rawArgc)
```

---

### `allocCStringArray` / `deallocCStringArray`
```
proc allocCStringArray(a: openArray[string]): cstringArray
proc deallocCStringArray(a: cstringArray)
```

Allocates a null-terminated C-style `char**` array from a Nim `seq[string]` (or array). You must call `deallocCStringArray` when done.

```nim
let args = @["program", "--verbose", "file.txt"]
let cArgs = allocCStringArray(args)
runCProgram(cArgs)
deallocCStringArray(cArgs)
```

---

### `rawProc` / `rawEnv`
```
proc rawProc[T: proc{.closure.}](x: T): pointer
proc rawEnv[T: proc{.closure.}](x: T): pointer
```

Extracts the raw function pointer and captured environment pointer from a closure. Used when passing Nim closures to C callbacks that expect a `void*` environment parameter.

---

### `getTypeInfo[T]`
```
proc getTypeInfo[T](x: T): pointer
```

Returns a pointer to the runtime type information (RTTI) of `x`. Used internally by the `typeinfo` module and certain reflection scenarios. Ordinary code should use the `typeinfo` module instead.

---

### `prepareMutation`
```
proc prepareMutation(s: var string)
```

Signals that you are about to mutate a string via a raw pointer (using `addr`). In ARC/ORC mode, string literals are copy-on-write and this proc ensures the string has its own writable copy before mutation. Omitting it can cause `SIGBUS` or `SIGSEGV`.

---

## 13. Metaprogramming & Debugging

### `repr[T]`
```
proc repr[T](x: T): string
```

Returns a detailed string representation of any Nim value, including its memory addresses for references. Handles circular data structures correctly. Indispensable for debugging complex data.

```nim
var data = @[@[1, 2], @[3, 4]]
echo repr(data)
# something like: 0x...[0x...[1, 2], 0x...[3, 4]]
```

---

### `instantiationInfo`
```
proc instantiationInfo(index = -1, fullPaths = false):
     tuple[filename: string, line: int, column: int]
```

Returns the **filename, line number, and column** of the template call site at compile time. Available only inside templates. Used to implement assertion-style macros that report the caller's location.

```nim
template myAssert(cond: bool) =
  if not cond:
    let info = instantiationInfo()
    echo "Assertion failed at ", info.filename, ":", info.line
```

---

### `locals`
```
proc locals(): RootObj
```

Returns a **tuple of all local variables** in the current scope at the call site. The actual return type is a generated tuple, not `RootObj`. Useful for debug dumps.

```nim
proc demo() =
  var x = 10
  var name = "Nim"
  for k, v in fieldPairs(locals()):
    echo k, " = ", v
# prints:
# x = 10
# name = Nim
```

---

### `finished[T]`
```
proc finished[T: iterator{.closure.}](x: T): bool
```

Checks whether a first-class closure iterator `x` has **finished iterating** (has no more values to yield).

```nim
iterator countUp(n: int): int {.closure.} =
  for i in 1..n: yield i

var it = countUp
discard it(3)
# After all values are yielded, finished(it) == true
```

---

### `arrayWith[T]`
```
proc arrayWith[T](y: T, size: static int): array[size, T]
```

Creates a fixed-size array where **every element** is a copy of `y`. The size must be known at compile time.

```nim
let grid = arrayWith(0, 5)
assert grid == [0, 0, 0, 0, 0]
```

---

### `arrayWithDefault[T]`
```
proc arrayWithDefault[T](size: static int): array[size, T]
```

Creates a fixed-size array where every element is the **default value** of type `T`.

```nim
let flags = arrayWithDefault[bool](4)
assert flags == [false, false, false, false]
```

---

## 14. Important Types & Constants

### Numeric Constants
| Constant | Value | Meaning |
|---|---|---|
| `Inf` | `+∞` (float64) | IEEE positive infinity |
| `NegInf` | `-∞` (float64) | IEEE negative infinity |
| `NaN` | `NaN` (float64) | IEEE Not-a-Number |
| `QuitSuccess` | `0` | Exit code for success |
| `QuitFailure` | `1` | Exit code for failure |
| `NimVersion` | e.g. `"2.0.0"` | Current Nim version string |

### Compile-Time Constants
| Constant | Type | Meaning |
|---|---|---|
| `hostOS` | `string` | OS name at compile time (e.g. `"linux"`) |
| `hostCPU` | `string` | CPU name at compile time (e.g. `"amd64"`) |
| `cpuEndian` | `Endianness` | `littleEndian` or `bigEndian` |
| `appType` | `string` | `"console"`, `"gui"`, or `"lib"` |

### Key Types
| Type | Description |
|---|---|
| `Natural` | `int` restricted to `0..high(int)` |
| `Positive` | `int` restricted to `1..high(int)` |
| `byte` | Alias for `uint8` |
| `RootObj` | Base of Nim's OO hierarchy |
| `RootRef` | `ref RootObj` |
| `NimNode` | AST node type used in macros |
| `HSlice[T,U]` | Inclusive range `[a..b]` |
| `Slice[T]` | `HSlice[T,T]` — homogeneous range |
| `Endianness` | Enum: `littleEndian` or `bigEndian` |

---

*This reference covers the public API of `system.nim` as found in the Nim runtime library. Internal, compiler-only, and deprecated items have been omitted for clarity.*
