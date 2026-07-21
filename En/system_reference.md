# system — module reference

> **Import:** the module is implicitly imported by the compiler, you never
> write `import system` yourself, and you cannot declare your own module
> named `system`.
> **Scope:** core types (`seq`, `string`, `array`, `openArray`, ranges),
> memory and ownership management (`addr`, `new`, `move`, the
> `=destroy`/`=copy`/`=sink` hooks), type introspection (`typeof`, `is`,
> `of`, `sizeof`), basic operations on strings and sequences (`add`, `len`,
> `setLen`, `substr`, `&`), the exception hierarchy and program
> termination (`quit`), and elementary output (`echo`).

The `system` module is the foundation the rest of Nim stands on: the types
`int`, `string`, `seq[T]`, `array` are declared right here (in the included
files `system/basic_types` and `system/hti`), and most procedures are
tagged with the `magic` pragma — meaning the real work is done by the
compiler itself, and the declaration in `system.nim` is essentially a typed
signature for documentation and overload resolution.

General conventions worth keeping in mind while reading this reference:

- Many procedures are declared as **families of overloads** for different
  types — for example, `len` is declared separately for `string`, `seq[T]`,
  `array`, `openArray`, `cstring`. In this reference such families are
  covered together under one subsection rather than one type at a time.
  Many procedures also have a "special form": `high`/`low` accepting either
  a value or a `typedesc` — the same rule applies there.
- Some declarations are hidden behind `when defined(...)` — they only exist
  on specific backends/builds (JS, ARC/ORC, standalone mode, etc.). Such
  spots are marked "conditional compilation" in this reference.
- All examples in this reference are deliberately rewritten in prefix form
  (`len(a)`, not `a.len()`), even though the original Nim documentation
  uses dot notation throughout — that is the only difference from the
  originals.

---

## Table of Contents

I. [Types and Metaprogramming](#i-types-and-metaprogramming)
&nbsp;&nbsp;1. [`static`, `type` — meta-types](#1-static-type--meta-types)
&nbsp;&nbsp;2. [`typeof`](#2-typeof)
&nbsp;&nbsp;3. [`or`, `and`, `not` for `typedesc`](#3-or-and-not-for-typedesc)
&nbsp;&nbsp;4. [Container and support types: `Ordinal`, `range`, `array`, `openArray`, `varargs`, `seq`, `set`, `UncheckedArray`, `sink`, `lent`](#4-container-and-support-types)
&nbsp;&nbsp;5. [`is` and `of`](#5-is-and-of)
&nbsp;&nbsp;6. [`sizeof`, `alignof`, `offsetOf`](#6-sizeof-alignof-offsetof)
&nbsp;&nbsp;7. [`default`, `zeroDefault`, `reset`](#7-default-zerodefault-reset)

II. [Ordinal Types, Ranges, and Indexing](#ii-ordinal-types-ranges-and-indexing)
&nbsp;&nbsp;1. [`high`, `low`](#1-high-low)
&nbsp;&nbsp;2. [`Natural`, `Positive`](#2-natural-positive)
&nbsp;&nbsp;3. [`..` — range constructor, `HSlice`/`Slice`](#3----range-constructor-hsliceslice)
&nbsp;&nbsp;4. [`contains`/`in`/`notin` for a range](#4-containsinnotin-for-a-range)
&nbsp;&nbsp;5. [`[]`, `[]=` — basic indexing](#5---basic-indexing)

III. [Memory and Ownership](#iii-memory-and-ownership)
&nbsp;&nbsp;1. [`addr`, `unsafeAddr`](#1-addr-unsafeaddr)
&nbsp;&nbsp;2. [`new`, `unsafeNew`](#2-new-unsafenew)
&nbsp;&nbsp;3. [`move`, `wasMoved`, `ensureMove`](#3-move-wasmoved-ensuremove)
&nbsp;&nbsp;4. [Lifecycle hooks: `=destroy`, `=copy`, `=sink`, `=dup`, `=trace`](#4-lifecycle-hooks)
&nbsp;&nbsp;5. [`isNil`](#5-isnil)
&nbsp;&nbsp;6. [`shallowCopy` (deprecated)](#6-shallowcopy-deprecated)

IV. [Sequences: Creation and Modification](#iv-sequences-creation-and-modification)
&nbsp;&nbsp;1. [`newSeq`, `newSeqOfCap`](#1-newseq-newseqofcap)
&nbsp;&nbsp;2. [`len` (overload family)](#2-len-overload-family)
&nbsp;&nbsp;3. [`add`, `del`, `delete`, `insert`, `pop`](#3-add-del-delete-insert-pop)
&nbsp;&nbsp;4. [`setLen`, `setLenUninit`](#4-setlen-setlenuninit)
&nbsp;&nbsp;5. [`find`, `contains` for `openArray`](#5-find-contains-for-openarray)
&nbsp;&nbsp;6. [`@` — array to seq](#6----array-to-seq)
&nbsp;&nbsp;7. [`swap`](#7-swap)

V. [Strings](#v-strings)
&nbsp;&nbsp;1. [`len`, `substr`](#1-len-substr)
&nbsp;&nbsp;2. [`&`, `add`, `&=` — concatenation](#2----add----concatenation)
&nbsp;&nbsp;3. [`newString`, `newStringOfCap`](#3-newstring-newstringofcap)
&nbsp;&nbsp;4. [`addEscapedChar`, `addQuoted`](#4-addescapedchar-addquoted)
&nbsp;&nbsp;5. [`toOpenArray`, `toOpenArrayByte`, `toOpenArrayChar`](#5-toopenarray-toopenarraybyte-toopenarraychar)

VI. [Numbers](#vi-numbers)
&nbsp;&nbsp;1. [`toFloat`, `toBiggestFloat`, `toInt`, `toBiggestInt`](#1-tofloat-tobiggestfloat-toint-tobiggestint)
&nbsp;&nbsp;2. [`/` for `int`](#2--for-int)
&nbsp;&nbsp;3. [`abs`](#3-abs)
&nbsp;&nbsp;4. [`ord`, `chr`](#4-ord-chr)
&nbsp;&nbsp;5. [`cmp`](#5-cmp)

VII. [Comparing Tuples and Objects](#vii-comparing-tuples-and-objects)
&nbsp;&nbsp;1. [`==`, `<=`, `<` for `tuple`/`object`](#1----for-tupleobject)

VIII. [Exceptions and Program Termination](#viii-exceptions-and-program-termination)
&nbsp;&nbsp;1. [Exception hierarchy: `Exception`, `Defect`, `CatchableError`](#1-exception-hierarchy)
&nbsp;&nbsp;2. [`newException`](#2-newexception)
&nbsp;&nbsp;3. [`quit`](#3-quit)
&nbsp;&nbsp;4. [`instantiationInfo`](#4-instantiationinfo)
&nbsp;&nbsp;5. [`procCall`](#5-proccall)

IX. [Output and Debugging](#ix-output-and-debugging)
&nbsp;&nbsp;1. [`echo`, `debugEcho`](#1-echo-debugecho)
&nbsp;&nbsp;2. [`repr` for `HSlice`](#2-repr-for-hslice)
&nbsp;&nbsp;3. [`locals`](#3-locals)

X. [Miscellaneous Utilities](#x-miscellaneous-utilities)
&nbsp;&nbsp;1. [`likely`, `unlikely`](#1-likely-unlikely)
&nbsp;&nbsp;2. [`once`, `closureScope`, `disarm`](#2-once-closurescope-disarm)
&nbsp;&nbsp;3. [`finished`](#3-finished)
&nbsp;&nbsp;4. [`arrayWith`, `arrayWithDefault`](#4-arraywith-arraywithdefault)

XI. [Practical Recipes](#xi-practical-recipes)
&nbsp;&nbsp;1. [Safely building a seq of a fixed size](#1-safely-building-a-seq-of-a-fixed-size)
&nbsp;&nbsp;2. [Removing an element without preserving order vs. preserving order](#2-removing-an-element-without-preserving-order-vs-preserving-order)
&nbsp;&nbsp;3. [A custom exception type with context](#3-a-custom-exception-type-with-context)
&nbsp;&nbsp;4. [Moving instead of copying in an accumulation loop](#4-moving-instead-of-copying-in-an-accumulation-loop)
&nbsp;&nbsp;5. [Working with a string slice without extra allocations](#5-working-with-a-string-slice-without-extra-allocations)

XII. [Quick Reference Table](#xii-quick-reference-table)

XIII. [Summary: Which Procedure to Choose](#xiii-summary-which-procedure-to-choose)

---

## I. Types and Metaprogramming

### 1. `static`, `type` — meta-types

```nim
type
  `static`*[T] {.magic: "Static".}
  `type`*[T] {.magic: "Type".}
```

**What it does.** These aren't ordinary types but compiler-level
constructs. `static[T]` describes values that must be evaluated at compile
time (used as a constraint on a generic parameter: `x: static int` means
"any integer known at compile time"). `type[T]` describes "the type of a
type itself" — what the expression `type(x)` returns.

- **Parameters:** `T` — the type parameter being described by the
  meta-type; `static`/`type` are never created directly, but rather used
  as constraints (`proc f(x: static int)`) or through the coercion syntax
  `static(x)` / `type(x)`.

```nim
proc repeatChar(c: static char, n: int): string =
  # `c` is guaranteed to be known at compile time — the compiler can
  # inline or check it right here
  result = newString(n)
  for i in 0 ..< n:
    result[i] = c

let s = repeatChar('x', 5)
assert s == "xxxxx"

assert type(42) is int          # `type(x)` — the type of expression `x`
assert type("hello") is string
```

---

### 2. `typeof`

```nim
proc typeof*(x: untyped; mode = typeOfIter): typedesc
```

**What it does.** Returns the type of expression `x`, computed by the
compiler without actually evaluating `x`. The `mode` parameter controls how
ambiguous calls are interpreted: if there's both a regular procedure and an
iterator with the same call signature, `typeOfIter` (the default) prefers
the interpretation "this is an iterator call", while `typeOfProc` prefers
"this is a regular procedure call". The existence of two modes is a direct
consequence of the fact that in Nim a name can be overloaded between `proc`
and `iterator` with identical call syntax.

- **Parameters:**
  - `x: untyped` — an arbitrary expression whose type needs to be
    determined; the expression is not evaluated.
  - `mode: TypeOfMode` — `typeOfProc` or `typeOfIter`, determines which
    interpretation takes priority under ambiguity; defaults to
    `typeOfIter`.

```nim
proc myFoo(): float = 0.0
iterator myFoo(): string = yield "abc"

assert typeof(myFoo()) is string           # default mode: as an iterator
assert typeof(myFoo(), typeOfProc) is float # explicitly as a regular call
```

---

### 3. `or`, `and`, `not` for `typedesc`

```nim
proc `or`*(a, b: typedesc): typedesc
proc `and`*(a, b: typedesc): typedesc
proc `not`*(a: typedesc): typedesc
```

**What it does.** Builds what are called "type meta-classes" — constructs
like `int or float`, `Ordinal and not enum`, used in generic type-parameter
constraints (`proc f[T: int or float](x: T)`). No actual "union type"
appears at runtime — this is purely a compile-time overload-resolution
mechanism.

- **Parameters:** `a`, `b: typedesc` — the types participating in the
  union, intersection, or negation of the constraint.

```nim
proc describe[T: int or string](x: T): string =
  when T is int:
    "integer: " & $x
  else:
    "string: " & x

assert describe(5) == "integer: 5"
assert describe("hi") == "string: hi"
```

---

### 4. Container and Support Types

```nim
type
  Ordinal*[T] {.magic: Ordinal.}
  range*[T] {.magic: "Range".}
  array*[I, T] {.magic: "Array".}
  openArray*[T] {.magic: "OpenArray".}
  varargs*[T] {.magic: "Varargs".}
  seq*[T] {.magic: "Seq".}
  set*[T] {.magic: "Set".}
  UncheckedArray*[T] {.magic: "UncheckedArray".}
  sink*[T] {.magic: "BuiltinType".}
  lent*[T] {.magic: "BuiltinType".}
```

**What it does.** These are not ordinary type declarations (there's no
body — only compiler magic), but the points through which the compiler
registers its built-in containers in Nim's type system, so that ordinary
generic and overload-resolution rules can apply to them.

- `Ordinal[T]` — a generalization over "ordinal" types: integers, `bool`,
  `char`, enumerations, and their subranges. Used as a constraint (`high`,
  `low` accept `Ordinal`).
- `range[T]` — a constructor for range types (`range[0..255]`).
- `array[I, T]` — a fixed-length array indexed by type `I` (usually a
  `range` or `Ordinal`).
- `openArray[T]` — an "open array": a pointer to data plus a length,
  without owning the memory; a common parameter type for procedures that
  accept both `seq[T]`, `array`, and a slice.
- `varargs[T]` — like `openArray[T]`, but lets the caller pass arguments
  comma-separated instead of as an explicit array.
- `seq[T]` — a growable sequence that owns its own memory.
- `set[T]` — a bit set; `T` must be an ordinal type with a small value
  range.
- `UncheckedArray[T]` — an array with no bounds checking; used together
  with manual memory management (`ptr UncheckedArray[T]`) when you need to
  explicitly disable checks for performance or C interoperability.
- `sink[T]` / `lent[T]` — not types in the usual sense, but ownership
  annotations for parameters: `sink T` says "this procedure may take
  ownership of this value", `lent T` — "this procedure only borrows the
  value temporarily, it does not take ownership".

```nim
proc sumOpen(a: openArray[int]): int =
  # openArray accepts both array and seq the same way
  result = 0
  for x in a:
    result += x

let arr = [1, 2, 3]
let s = @[1, 2, 3]
assert sumOpen(arr) == 6
assert sumOpen(s) == 6

type Digit = range[0..9]
var d: Digit = 7
assert d == 7
```

---

### 5. `is` and `of`

```nim
proc `is`*[T, S](x: T, y: S): bool
proc `of`*[T, S](x: T, y: typedesc[S]): bool
```

**What it does.** `is` checks whether an expression's type matches a given
type or constraint, fully evaluated at compile time (costs nothing at
runtime). `of` checks whether an object in a dynamic inheritance hierarchy
is actually an instance of `S` or one of its descendants; unlike `is`, for
object hierarchies this is a runtime check, since the fact of inheritance
is only known from data stored in the object itself.

- **Parameters:**
  - `x: T` — the value or expression being checked.
  - `y: S` / `y: typedesc[S]` — the type or constraint being compared
    against.

```nim
assert 5 is int
assert "abc" is string
assert not (5 is string)

type
  Animal = ref object of RootObj
  Dog = ref object of Animal

let a: Animal = Dog()
assert a of Dog     # runtime check of the dynamic type
assert a of Animal
```

---

### 6. `sizeof`, `alignof`, `offsetOf`

```nim
proc sizeof*[T](x: T): int
proc sizeof*(x: typedesc): int
proc alignof*[T](x: T): int
proc alignof*(x: typedesc): int
template offsetOf*[T](t: typedesc[T]; member: untyped): int
template offsetOf*[T](value: T; member: untyped): int
```

**What it does.** `sizeof` returns the size of a value or type in bytes —
the actual physical size in memory, not the logical capacity
(`sizeof(seq[int])` returns the size of the `seq`'s internal bookkeeping
structure, not the total size of its elements). `alignof` returns a type's
required alignment in bytes. `offsetOf` returns the byte offset of a
specific field within an object/struct — useful when interoperating with C
code, where the binary field layout matters.

- **Parameters:**
  - `x: T` / `x: typedesc` — the value or type whose size or alignment is
    being computed.
  - `t: typedesc[T]` / `value: T` — the object's type or value, `member`
    is the field name (passed as `untyped`, i.e. unevaluated).

```nim
assert sizeof('A') == 1
assert sizeof(int) == 8      # on a 64-bit platform

type Point = object
  x, y: int32
  active: bool

assert offsetOf(Point, x) == 0
```

---

### 7. `default`, `zeroDefault`, `reset`

```nim
func zeroDefault*[T](_: typedesc[T]): T
proc default*[T](_: typedesc[T]): T
proc reset*[T](obj: var T)
```

**What it does.** `zeroDefault(T)` returns a representation of type `T`
made of binary zeros, completely ignoring any user-defined field defaults
(`= someValue` in the type's definition). `default(T)` does the same, but
**does take into account** user-defined field defaults, if any are
specified in the object's definition. `reset(obj)` returns the variable
`obj` to its initial ("zero"/default) state — equivalent to
`obj = default(typeof(obj))`, but without creating an intermediate
temporary value.

- **Parameters:**
  - `_: typedesc[T]` — the type for which a default value is built; the
    parameter itself is unused, it only determines `T`.
  - `obj: var T` — the mutable variable to be reset.

```nim
type Config = object
  retries: int = 3      # a user-defined default value
  verbose: bool

assert default(Config).retries == 3       # takes `= 3` into account
assert zeroDefault(Config).retries == 0   # ignores it, pure zeros

var counter = 42
reset(counter)
assert counter == 0
```

---

## II. Ordinal Types, Ranges, and Indexing

### 1. `high`, `low`

```nim
proc high*[T: Ordinal|enum|range](x: typedesc[T]): T
proc high*[T](x: openArray[T]): int
proc high*[I, T](x: array[I, T]): I
proc high*(x: string): int
proc low*[T: Ordinal|enum|range](x: typedesc[T]): T
proc low*[T](x: openArray[T]): int
proc low*[I, T](x: array[I, T]): I
proc low*(x: string): int
```

**What it does.** `high`/`low` return, respectively, the largest and
smallest permissible value or index. There are two fundamentally different
families of overloads:

1. **By type** (`high(int)`, `high(SomeEnum)`) — returns the bound of the
   type's own range of values. For instance, `high(int)` is the platform's
   `INT_MAX`.
2. **By container value** (`high(myArray)`, `high(mySeq)`, `high(myStr)`)
   — returns the largest permissible **index** of that specific
   container, which for `seq`/`string` equals `len(x) - 1`, not some
   absolute maximum value.

The older overloads `high(value: Ordinal)` / `low(value: Ordinal)`
(accepting a specific value of an ordinal type, rather than the type
itself) are marked `deprecated` since version 1.4 — you should use
`high(typedesc)`/`low(typedesc)` instead.

For empty arrays, `high` returns `-1` (and `low` returns `0`), which is why
`for i in low(a)..high(a)` safely "doesn't run at all" instead of ending up
with a range of negative length.

- **Parameters:**
  - `x: typedesc[T]` — the type for which the value-range bound is taken
    (`T` is an ordinal type, `enum`, or `range`).
  - `x: openArray[T]` / `x: array[I, T]` / `x: string` — the container for
    which the index bound is taken.

```nim
assert high(int) == 9223372036854775807
assert low(int) == -9223372036854775808

var s = @[10, 20, 30]
assert low(s) == 0
assert high(s) == 2        # len(s) - 1, not an absolute maximum

var empty: seq[int] = @[]
assert high(empty) == -1   # an empty sequence — an empty range

for i in low(s) .. high(s):
  echo s[i]
```

---

### 2. `Natural`, `Positive`

```nim
type
  Natural* = range[0..high(int)]
  Positive* = range[1..high(int)]
```

**What it does.** These aren't separate procedures but two ready-made
range types built on top of `range` and `int`: `Natural` allows 0 and
positive numbers, `Positive` allows only strictly positive numbers. They
are used primarily for documentation and static contract-checking of
procedures (for example, `newSeq[T](len: Natural)` — the length can't be
negative, and this is guaranteed by the type rather than by a runtime
check inside the procedure).

```nim
proc repeatWord(word: string, times: Positive): string =
  result = ""
  for i in 1 .. times:
    add(result, word)

assert repeatWord("go", 3) == "gogogo"
# repeatWord("go", 0) — won't compile with a constant value,
# or will fail at runtime if passed a variable outside the range
```

---

### 3. `..` — Range Constructor, `HSlice`/`Slice`

```nim
type
  HSlice*[T, U] = object
    a*: T
    b*: U
  Slice*[T] = HSlice[T, T]

proc `..`*[T, U](a: sink T, b: sink U): HSlice[T, U]
proc `..`*[T](b: sink T): HSlice[int, T]
```

**What it does.** The `..` operator builds a range value `[a, b]`
(inclusive on both ends) — the `HSlice[T, U]` type simply stores a pair of
bounds `a`/`b`, with no expansion into a sequence of values happening at
construction time; that's done lazily when iterating `for i in a..b`. The
single-parameter variant `..b` builds the range `0..b` — a shorthand
convenient for indexing from the start of a container. `Slice[T]` is a
special case of `HSlice` where both bounds share the same type.

- **Parameters:**
  - `a: sink T`, `b: sink U` — the lower and upper bound of the range;
    `sink` means the procedure may take ownership of the value if that's
    cheaper than copying.

```nim
let r = 2 .. 5
assert r.a == 2
assert r.b == 5

var s = @[10, 20, 30, 40, 50]
assert s[1 .. 3] == @[20, 30, 40]
assert s[.. 2] == @[10, 20, 30]   # shorthand: 0..2
```

---

### 4. `contains`/`in`/`notin` for a Range

```nim
proc contains*[U, V, W](s: HSlice[U, V], value: W): bool
template `in`*(x, y: untyped): untyped = contains(y, x)
template `notin`*(x, y: untyped): untyped = not contains(y, x)
```

**What it does.** Checks whether `value` falls within the bounds of range
`s` (inclusive). The `in`/`notin` templates are simply syntactic sugar
that swap the operands: `x in y` expands to `contains(y, x)`. That's why
`in` works not just with ranges but with anything that has `contains`
defined for it (sets, tables, strings, and so on) — this is a general
language mechanism, not something specific to ranges.

- **Parameters:**
  - `s: HSlice[U, V]` — the range in which membership is being checked.
  - `value: W` — the value being checked.

```nim
let r = 1 .. 10
assert contains(r, 5)
assert 5 in r
assert 11 notin r
```

---

### 5. `[]`, `[]=` — Basic Indexing

```nim
proc `[]`*[I: Ordinal; T](a: T; i: I): T
proc `[]=`*[I: Ordinal; T, S](a: T; i: I; x: sink S)
```

**What it does.** This is the `ArrGet`/`ArrPut` compiler magic — the basic
mechanism for indexed reads and writes that underlies the `a[i]` and
`a[i] = x` syntax for `array`, `openArray`, `seq`, `string`, `cstring`,
and tuples. There's no point calling `` `[]`(a, i) `` directly instead of
`a[i]` — this is an internal layer through which the compiler implements
indexing; user code always uses the square-bracket syntax.

- **Parameters:**
  - `a: T` — the container being indexed.
  - `i: I` — an index of an ordinal type.
  - `x: sink S` — the value being written (for `[]=`).

```nim
var arr = [1, 2, 3]
assert arr[0] == 1
arr[1] = 99
assert arr == [1, 99, 3]
```

---

## III. Memory and Ownership

### 1. `addr`, `unsafeAddr`

```nim
proc `addr`*[T](x: T): ptr T
proc unsafeAddr*[T](x: T): ptr T
```

**What it does.** `addr(x)` returns a "raw" (non-GC-traced) pointer to the
memory where `x` is stored. It works for `let` variables and parameters —
this was specifically designed for convenient interop with C: a library
needs a pointer, but the source value was declared immutable.
`unsafeAddr` is a deprecated alias for `addr`, kept only for backward
compatibility.

- **Parameters:** `x: T` — the value whose address is being taken; `T`
  must reside in addressable memory (not, say, a temporary result of a
  computation held in a register).

```nim
var buf: seq[char] = @['a', 'b', 'c']
let p = addr(buf[1])
assert p[] == 'b'
p[] = 'z'
assert buf == @['a', 'z', 'c']
```

---

### 2. `new`, `unsafeNew`

```nim
proc new*[T](a: var ref T, finalizer: proc (x: T) {.nimcall.})
proc unsafeNew*[T](a: var ref T, size: Natural)
```

**What it does.** The overload of `new` with a finalizer creates an object
of type `T` and stores a traced (garbage-collector-managed) reference to
it in `a`; when the garbage collector frees the object, `finalizer` is
called. An important subtlety: the finalizer is tied to the **type**, not
to a specific object — it will be called for every object of that type,
not only for the one passed to `new`. `unsafeNew` is a lower-level variant
that lets you explicitly specify the memory size allocated for the object;
used for optimizations when the standard `sizeof(T)`-based size isn't
appropriate (for example, an object with a "trailing" variable-length
array).

- **Parameters:**
  - `a: var ref T` — the reference variable that will receive the new
    object.
  - `finalizer: proc (x: T) {.nimcall.}` — the procedure called by the GC
    when the object's memory is freed; must not retain a reference to
    `x`.
  - `size: Natural` (for `unsafeNew`) — the explicit size of the allocated
    memory.

```nim
type Resource = object
  id: int

var r: ref Resource
new(r, proc (x: Resource) = echo "resource released: ", x.id)
r.id = 7
assert r.id == 7
```

---

### 3. `move`, `wasMoved`, `ensureMove`

```nim
proc move*[T](x: var T): T
proc wasMoved*[T](obj: var T)
proc ensureMove*[T](x: T): T
```

**What it does.** `move(x)` takes the value of `x` outright: it returns it
as the result and puts the source variable `x` back into its initial
(zeroed) state via `wasMoved`, so that `x`'s destructor doesn't do
anything redundant once it goes out of scope (ideally the compiler
eliminates the destructor call entirely). `wasMoved(obj)` on its own is
just a reset of the variable to binary zero without returning a value — a
signal to the compiler "this variable no longer owns the resource".
`ensureMove(x)` is a compile-time check: it guarantees that `x` is
actually moved (rather than silently copied) into its new location,
otherwise it's a compile error.

- **Parameters:**
  - `x: var T` (for `move`) / `obj: var T` (for `wasMoved`) — the variable
    whose contents are being taken or reset.
  - `x: T` (for `ensureMove`) — the expression whose move must be
    guaranteed by the compiler.

```nim
proc buildBigString(): string =
  var x = "Hello"
  let y = ensureMove(x)   # compile error if `x` were used again afterward
  assert y == "Hello"

buildBigString()

var big = @[1, 2, 3, 4, 5]
let moved = move(big)
assert moved == @[1, 2, 3, 4, 5]
assert big.len == 0        # `big` was reset after the move
```

---

### 4. Lifecycle Hooks

```nim
proc `=destroy`*[T](x: var T)
proc `=copy`*[T](dest: var T; src: T)
proc `=sink`*[T](x: var T; y: T)
proc `=dup`*[T](x: T): T
proc `=trace`*[T](x: var T; env: pointer)
```

**What it does.** These are not procedures meant for direct use in user
code, but extension points of the ARC/ORC memory-management system: the
compiler itself calls `=destroy` when a variable goes out of scope,
`=copy` on an explicit assignment, `=sink` on a move (when the source is
no longer needed), `=dup` when an independent copy is required, and
`=trace` during garbage collection while traversing the reference graph.
The standard implementations in `system` are stubs/default behavior;
custom types may **override** them (for example, to close a file
descriptor in `=destroy`), and the compiler then uses the custom version
for that type instead of the system one.

- **Parameters:** each hook receives variables/values of the type whose
  behavior is being overridden; writing them by hand for your own types
  is only necessary when the standard (byte-wise/member-wise) behavior
  isn't enough (ownership of an external resource, manual memory
  management, and so on).

```nim
type FileHandle = object
  fd: int

proc `=destroy`(x: var FileHandle) =
  if x.fd != 0:
    echo "closing descriptor ", x.fd
    x.fd = 0

proc openFake(id: int): FileHandle =
  result = FileHandle(fd: id)

block:
  var f = openFake(42)
  assert f.fd == 42
  # `=destroy` is called automatically when the block exits
```

---

### 5. `isNil`

```nim
proc isNil*[T](x: ref T): bool
proc isNil*[T](x: ptr T): bool
proc isNil*(x: pointer): bool
proc isNil*(x: cstring): bool
proc isNil*[T: proc | iterator {.closure.}](x: T): bool
```

**What it does.** Checks whether a reference value (a traced reference, a
raw pointer, a `cstring`, or a closure) points to nothing at all — a
single, uniform way to check for "emptiness" across all of Nim's reference
categories.

- **Parameters:** `x` — the value being checked, of any of the listed
  reference types.

```nim
var p: ptr int = nil
assert isNil(p)

var r: ref int
assert isNil(r)
new(r)
assert not isNil(r)
```

---

### 6. `shallowCopy` (Deprecated)

```nim
proc shallowCopy*[T](x: var T, y: T)
```

**What it does.** Assigns `y` to `x`, but for `seq`/`string` (and types
containing them) — without deep-copying the data, unlike a regular `=`.
Only available under GC models other than ARC/ORC (when `arcLikeMem` is
off), and its use requires caution: mutating one of the "aliases" after
`shallowCopy` shows up in the other one too, which breaks the usual
expectations from assignment.

- **Parameters:** `x: var T` — the destination, `y: T` — the source,
  whose internal data is not copied but shared with `x`.

```nim
# available only without ARC/ORC; in modern projects using ARC/ORC
# this procedure is simply unavailable — use a regular assignment
# or `move` instead if you need to avoid a copy.
```

---

## IV. Sequences: Creation and Modification

### 1. `newSeq`, `newSeqOfCap`

```nim
proc newSeq*[T](s: var seq[T], len: Natural)
proc newSeq*[T](len = 0.Natural): seq[T]
proc newSeqOfCap*[T](cap: Natural): seq[T]
```

**What it does.** The first two overloads create a sequence of a given
**length**, filled with zeroed element values — this is more efficient
than `@[]` followed by `len` calls to `add`, because the memory is
allocated all at once, without intermediate reallocations. An important
subtlety: after `newSeq(s, 3)`, `s` already has 3 elements (zeroed), and
you should access them via `s[i] = ...` rather than `add(s, ...)` —
otherwise the elements would be appended *after* the existing three.
`newSeqOfCap` is the opposite case: the length stays at zero, but memory
is pre-reserved for `cap` elements, so subsequent `add` calls won't
reallocate the buffer until that capacity is exceeded.

- **Parameters:**
  - `s: var seq[T]` — the sequence that will be (re)created.
  - `len: Natural` — the desired length of the new sequence.
  - `cap: Natural` — the reserved capacity, without changing the length.

```nim
var names: seq[string]
newSeq(names, 3)
assert len(names) == 3
names[0] = "first"
names[1] = "second"
names[2] = "third"

var buf = newSeqOfCap[int](100)
assert len(buf) == 0
add(buf, 1)
assert len(buf) == 1   # capacity 100 is already allocated, no reallocation yet
```

---

### 2. `len` (Overload Family)

```nim
func len*[TOpenArray: openArray|varargs](x: TOpenArray): int
func len*(x: string): int
proc len*(x: cstring): int
func len*(x: (type array)|array): int
func len*[T](x: seq[T]): int
proc len*[U: Ordinal; V: Ordinal](x: HSlice[U, V]): int
```

**What it does.** Returns the number of elements in a container. For
`array` this is essentially `high(T) - low(T) + 1`, known at compile time.
For `string`/`seq`, it's the current length in elements (for a string —
in bytes, not Unicode characters: for that there's `runeLen` in
`std/unicode`). For `cstring`, this is an O(n) operation at runtime
(except when compiling to JS, where the string already stores its
length), since a C string doesn't store its length explicitly — it's
terminated by a null byte instead. For `HSlice`/a range — the count of
integer values covered by the range.

- **Parameters:** `x` — a container of the corresponding type.

```nim
assert len("abc") == 3
assert len(@[1, 2, 3]) == 3
assert len([1, 2, 3, 4]) == 4
assert len(1 .. 10) == 10
```

---

### 3. `add`, `del`, `delete`, `insert`, `pop`

```nim
proc add*(x: var string, y: char)
proc add*(x: var string, y: string)
proc del*[T](x: var seq[T], i: Natural)
proc insert*[T](x: var seq[T], item: sink T, i = 0.Natural)
proc delete*[T](x: var seq[T], i: Natural)
proc pop*[T](s: var seq[T]): T
```

**What it does.** `add` appends an element/string to the end. `del(x, i)`
removes the element at index `i` **without preserving order**: the
element "from the end" of the sequence is moved into the removed
position — this is O(1) instead of O(n), but the order of the remaining
elements changes. `delete(x, i)`, on the other hand, removes while
**preserving order**, shifting all subsequent elements one position to
the left — this is O(n), but the list stays in the same order as before.
`insert(x, item, i)` inserts an element at position `i`, shifting
subsequent elements to the right (defaults to `i = 0`, i.e. insertion at
the start). `pop(s)` removes and returns the **last** element of the
sequence — O(1), since it requires no shifting of the remaining elements.

- **Parameters:**
  - `x: var string` / `x: var seq[T]` — the mutable container.
  - `y: char` / `y: string` — the data being added.
  - `i: Natural` — the element's index.
  - `item: sink T` — the element being inserted.
  - `s: var seq[T]` — the sequence from which the last element is
    extracted.

```nim
var s = @[1, 2, 3, 4, 5]

del(s, 1)
assert s == @[1, 5, 3, 4]        # 2 removed, 5 "from the end" took its place

var ordered = @[1, 2, 3, 4, 5]
delete(ordered, 1)
assert ordered == @[1, 3, 4, 5]  # order preserved, shifted

insert(ordered, 99, 1)
assert ordered == @[1, 99, 3, 4, 5]

var stack = @[1, 2, 3]
let top = pop(stack)
assert top == 3
assert stack == @[1, 2]
```

---

### 4. `setLen`, `setLenUninit`

```nim
proc setLen*[T](s: var seq[T], newlen: Natural)
proc setLen*(s: var string, newlen: Natural)
func setLenUninit*(s: var string, newlen: Natural)
```

**What it does.** Changes the length of `s` to `newlen`: on growth, new
elements are filled with the zero value of type `T` (or zero bytes for a
string); on shrinking, the extra elements at the end are dropped (without
freeing the underlying allocated memory — capacity stays the same).
`setLenUninit` is a variant for strings that does **not** zero out the new
bytes on growth: it's faster, but requires the calling code to fill the
new bytes itself before reading them — otherwise there will be "garbage"
left over from memory.

- **Parameters:** `s` — the mutable `seq`/`string`, `newlen: Natural` —
  the new length.

```nim
var s = @[1, 2, 3]
setLen(s, 5)
assert s == @[1, 2, 3, 0, 0]     # new elements are zeros

setLen(s, 2)
assert s == @[1, 2]              # the extra elements are dropped

var str = "abc"
setLen(str, 5)
assert str[0..2] == "abc"        # the first 3 bytes are untouched
```

---

### 5. `find`, `contains` for `openArray`

```nim
proc find*[T, S](a: T, item: S): int
proc contains*[T](a: openArray[T], item: T): bool
```

**What it does.** `find` returns the index of the first occurrence of
`item` in `a`, or `-1` if the element isn't found — a linear O(n) search.
`contains` is built on top of `find` and simply checks that the result is
non-negative; it's `contains` that makes it possible to write `item in a`
for sequences, thanks to the general `in` → `contains` mechanism.

- **Parameters:** `a` — the container to search, `item` — the value being
  searched for.

```nim
let a = @[10, 20, 30]
assert find(a, 20) == 1
assert find(a, 99) == -1
assert contains(a, 30)
assert 99 notin a
```

---

### 6. `@` — Array to Seq

```nim
proc `@`*[IDX, T](a: sink array[IDX, T]): seq[T]
```

**What it does.** The prefix operator `@` converts a fixed-size array
value into a dynamic sequence — this is exactly the operation behind the
familiar `@[1, 2, 3]` literal (the compiler first builds an `array`, then
applies `@`).

- **Parameters:** `a: sink array[IDX, T]` — the source array; `sink`
  allows the array's memory to be reused instead of copied, if the caller
  no longer needs the array.

```nim
let arr = [1, 2, 3]
let s = `@`(arr)
assert s == @[1, 2, 3]
assert s is seq[int]
```

---

### 7. `swap`

```nim
proc swap*[T](a, b: var T)
```

**What it does.** Swaps the contents of `a` and `b` without an
intermediate "by-value" copy into a third variable — implemented as an
exchange of the internal representation (for `seq`/`string`/`ref` this
literally means swapping pointers and lengths, O(1), not an element-wise
copy).

- **Parameters:** `a`, `b: var T` — two mutable variables of the same
  type.

```nim
var x = @[1, 2, 3]
var y = @[9, 8]
swap(x, y)
assert x == @[9, 8]
assert y == @[1, 2, 3]
```

---

## V. Strings

### 1. `len`, `substr`

```nim
func len*(x: string): int
proc substr*(s: string; first, last: int): string
proc substr*(s: string, first = 0): string
```

**What it does.** `len` — the length of a string in bytes (see also
section IV.2). `substr` returns a **new** string — a copy of the slice
from `first` to `last` inclusive (or to the end of the string, if `last`
isn't given). Out-of-bounds indices are handled gracefully, without
raising exceptions: a `first` index greater than the string's length
yields an empty string, and a negative `first` is silently clamped to
0 — so `substr` never raises an exception due to invalid bounds, unlike
the `[]` slice operator.

- **Parameters:**
  - `s: string` — the source string.
  - `first: int` — the starting index of the slice (defaults to 0).
  - `last: int` — the ending index of the slice, inclusive (defaults to
    the end of the string).

```nim
let a = "abcdefgh"
assert substr(a, 2) == "cdefgh"     # from index 2 to the end
assert substr(a, 100) == ""         # `first` out of bounds — empty, no error
assert substr(a, -1) == "abcdefgh"  # negative `first` clamped to 0
assert substr(a, 2, 4) == "cde"
```

---

### 2. `&`, `add`, `&=` — Concatenation

```nim
proc `&`*(x: string, y: char): string
proc `&`*(x, y: char): string
proc `&`*(x, y: string): string
proc `&`*(x: char, y: string): string
proc add*(x: var string, y: char)
proc add*(x: var string, y: string)
proc `&=`*(x: var string, y: string)
```

**What it does.** `&` builds a **new** string out of two operands
(strings and/or characters) — used when the source strings shouldn't be
modified. `add` appends a character or string to the end of an existing
string **in place**, without creating a new one — more efficient than
`s = s & extra`, since it reuses the already-allocated buffer if there's
enough capacity. `&=` is the "concatenate in place" operator, essentially
a synonym for `add`, but in the form of a compound assignment.

- **Parameters:** `x`, `y` — the strings/characters being combined; for
  `add`/`&=`, `x: var string` is modified in place.

```nim
assert "abc" & "def" == "abcdef"
assert 'a' & "bc" == "abc"

var s = "Hello"
add(s, ", ")
add(s, "world!")
assert s == "Hello, world!"

var t = "foo"
`&=`(t, "bar")
assert t == "foobar"
```

---

### 3. `newString`, `newStringOfCap`

```nim
proc newString*(len: Natural): string
proc newStringOfCap*(cap: Natural): string
```

**What it does.** `newString(len)` creates a string of a given length,
filled with zero bytes — the string counterpart of `newSeq`.
`newStringOfCap` creates a zero-length string with a reserved buffer
capacity — so subsequent `add` calls won't cause reallocations until that
capacity is exceeded.

- **Parameters:** `len`/`cap: Natural` — the length or the reserved
  capacity.

```nim
var s = newString(5)
assert len(s) == 5

var builder = newStringOfCap(256)
assert len(builder) == 0
add(builder, "the start of a string")
```

---

### 4. `addEscapedChar`, `addQuoted`

```nim
proc addEscapedChar*(s: var string, c: char)
proc addQuoted*[T](s: var string, x: T)
```

**What it does.** `addEscapedChar` appends character `c` to `s`, escaping
non-printable/special characters (a newline becomes `\n`, a backslash
becomes `\\`, and so on) — the same principle used when generating a
`repr` representation. `addQuoted` is a more general procedure: it
appends a "quoted" (Nim-literal-style) representation of an arbitrary
value `x` of type `T`, using `addEscapedChar` internally for
character/string data.

- **Parameters:** `s: var string` — the accumulation buffer, `c: char` —
  the character to escape, `x: T` — an arbitrary value to render in
  quoted form.

```nim
var s = ""
addEscapedChar(s, '\n')
assert s == "\\n"

var buf = ""
addQuoted(buf, "hi\nthere")
assert buf == "\"hi\\nthere\""
```

---

### 5. `toOpenArray`, `toOpenArrayByte`, `toOpenArrayChar`

```nim
proc toOpenArray*[T](x: seq[T]; first, last: int): openArray[T]
proc toOpenArray*(x: string; first, last: int): openArray[char]
proc toOpenArrayByte*(x: string; first, last: int): openArray[byte]
proc toOpenArrayChar*(x: openArray[byte]; first, last: int): openArray[char]
```

**What it does.** Builds a **non-owning slice** (a `view`) of the source
data from `first` to `last` inclusive — unlike `substr`/a `[]` slice, this
involves **no copying**: `openArray` is simply a pointer plus a length
over the source container's memory. This is cheaper than passing a copy
of the slice when the data is only needed for reading inside the called
procedure. `toOpenArrayByte`/`toOpenArrayChar` give the same slice but
treat the bytes as `byte`/`char` respectively — convenient when working
with binary data laid over strings.

- **Parameters:** `x` — the source data, `first`, `last: int` — the
  slice's bounds, inclusive.

```nim
proc sumBytes(a: openArray[byte]): int =
  result = 0
  for b in a:
    result += int(b)

let data = "abcde"
let view = toOpenArrayByte(data, 1, 3)
assert sumBytes(view) == int('b') + int('c') + int('d')
```

---

## VI. Numbers

### 1. `toFloat`, `toBiggestFloat`, `toInt`, `toBiggestInt`

```nim
proc toFloat*(i: int): float
proc toBiggestFloat*(i: BiggestInt): BiggestFloat
proc toInt*(f: float): int
proc toBiggestInt*(f: BiggestFloat): BiggestInt
```

**What it does.** Explicit conversions between integers and
floating-point numbers (in addition to the regular conversion syntax like
`float(i)`/`int(f)`). `toInt`/`toBiggestInt` round to the nearest integer
(rounding, not truncating the fractional part the way a direct type
conversion `int(f)` does).

- **Parameters:** `i` — an integer, `f` — a floating-point number.

```nim
assert toFloat(5) == 5.0
assert toInt(2.7) == 3      # rounding, not truncation
assert toInt(2.4) == 2
```

---

### 2. `/` for `int`

```nim
proc `/`*(x, y: int): float
```

**What it does.** The division operator `int / int` returns a `float`,
not integer division — designed so that `5 / 2` gives `2.5`, not `2` as it
would under the usual integer-division semantics in other languages. For
integer division in Nim there are separate operators, `div`/`mod`.

- **Parameters:** `x`, `y: int` — the dividend and divisor.

```nim
assert `/`(5, 2) == 2.5
assert `/`(4, 2) == 2.0
```

---

### 3. `abs`

```nim
proc abs*[T: float64 | float32](x: T): T
func abs*(x: int): int
func abs*(x: int8): int8
func abs*(x: int16): int16
func abs*(x: int32): int32
func abs*(x: int64): int64
```

**What it does.** Returns the absolute value of a number — separate
overloads per integer type are needed so the result stays the same type
as the input (rather than implicitly widening to `int64`).

- **Parameters:** `x` — a number of any of the supported numeric types.

```nim
assert abs(-5) == 5
assert abs(5) == 5
assert abs(-3.14) == 3.14
```

---

### 4. `ord`, `chr`

```nim
func ord*[T: Ordinal|enum](x: T): int
func chr*(u: range[0..255]): char
```

**What it does.** `ord` returns the integer ordinal value of an ordinal
type or enum (for `char` — the character code, for `enum` — the variant
number). `chr` is the inverse operation: it builds a character from a
`0..255` code.

- **Parameters:** `x` — a value of an ordinal type, `u` — a character
  code in the range `0..255`.

```nim
assert ord('A') == 65
assert chr(65) == 'A'

type Color = enum red, green, blue
assert ord(green) == 1
```

---

### 5. `cmp`

```nim
proc cmp*[T](x, y: T): int
proc cmp*(x, y: string): int
```

**What it does.** A universal three-way comparison function: returns a
negative number if `x < y`, zero on equality, and a positive number if
`x > y`. Used as a basic building block for sorting and other algorithms
that need a single comparator rather than separate `<` and `==`. The
`string` overload is separate because comparing strings is usually done
via a specialized character-by-character algorithm rather than the
generic `<`/`==` implementation for a generic `T`.

- **Parameters:** `x`, `y: T` — the two values of the same type being
  compared.

```nim
assert cmp(1, 2) < 0
assert cmp(2, 2) == 0
assert cmp(3, 2) > 0
assert cmp("abc", "abd") < 0
```

---

## VII. Comparing Tuples and Objects

### 1. `==`, `<=`, `<` for `tuple`/`object`

```nim
proc `==`*[T: tuple|object](x, y: T): bool
proc `<=`*[T: tuple](x, y: T): bool
proc `<`*[T: tuple](x, y: T): bool
```

**What it does.** The standard (compiler-generated for any `tuple`/
`object`, unless explicitly overridden) comparison implementation: `==`
compares objects/tuples **field by field, across every field**. For
tuples, `<=`/`<` compare fields **lexicographically** — the same way
strings are compared character by character: the first field that
differs determines the result. Note that for `object`, the operators
`<`/`<=` are **not** generated automatically — they're defined only for
`tuple`, since objects have no single "natural" field ordering implied by
the user.

- **Parameters:** `x`, `y: T` — the two tuples or objects of the same
  type being compared.

```nim
assert (1, "a") == (1, "a")
assert (1, "a") != (1, "b")

assert (1, 2) < (1, 3)     # the first field is equal, the second decides
assert (1, 5) < (2, 0)     # the first field already decides everything
```

---

## VIII. Exceptions and Program Termination

### 1. Exception Hierarchy

```nim
type
  Exception* = object of RootObj
    parent*: ref Exception
    name*: cstring
    msg*: string
    trace*: seq[StackTraceEntry]
  Defect* = object of Exception
  CatchableError* = object of Exception
```

**What it does.** The root of Nim's exception types is `Exception`. Two
branches with different meanings inherit from it: `Defect` is for errors
that signal programming bugs and generally aren't meant to be caught in
normal control flow (index mismatches, division by zero, and so on — in a
release build such errors may be compiled as uncatchable, leading to an
abrupt termination); `CatchableError` is the base class for exceptions
that are expected to be caught in user code (`IOError`, `ValueError`, etc.
from other modules inherit from it directly). The `trace` field holds the
call stack at the moment the exception was created — used when printing
the message for an unhandled exception.

- **Fields:** `parent` — the exception wrapped by the current one (a
  chain of causes), `name` — the exception type's name (filled in
  automatically at `raise`), `msg` — the message text, `trace` — a
  snapshot of the call stack.

```nim
type MyError = object of CatchableError
  code: int

try:
  raise newException(MyError, "something went wrong")
except MyError as e:
  assert e.msg == "something went wrong"
  assert e.name == "MyError"
```

---

### 2. `newException`

```nim
template newException*(exceptn: typedesc, message: string): untyped
```

**What it does.** Creates and returns a new instance of the given
exception type with the `msg` field already filled in — a shorthand for
manually writing `var e = new(exceptn); e.msg = message`.

- **Parameters:** `exceptn: typedesc` — the exception type (a descendant
  of `Exception`), `message: string` — the message text.

```nim
type ConfigError = object of ValueError

proc loadConfig(path: string) =
  if len(path) == 0:
    raise newException(ConfigError, "config path is empty")

try:
  loadConfig("")
except ConfigError as e:
  assert e.msg == "config path is empty"
```

---

### 3. `quit`

```nim
proc quit*(errormsg: string, errorcode = QuitFailure) {.noreturn.}
```

**What it does.** Prints `errormsg` and immediately terminates the
program with the return code `errorcode` — unlike an exception, `quit`
does not unwind the stack or call the destructors of objects still alive
at the time of the call (a deliberate trade-off: abrupt termination should
be fast and unconditional).

- **Parameters:** `errormsg: string` — the message printed before exiting,
  `errorcode` — the process's return code (defaults to `QuitFailure`).

```nim
# quit("critical configuration error") — would terminate the process
# immediately; not invoked in this reference's examples so as not to
# interrupt the assert checks
```

---

### 4. `instantiationInfo`

```nim
proc instantiationInfo*(index = -1, fullPaths = false): tuple[
  filename: string, line: int, column: int]
```

**What it does.** Returns information about the call site (file, line,
column) — but not of the current procedure, rather of the place where the
construct that called it was instantiated (used mainly inside templates
and macros, so error messages point at the user code that invoked the
template rather than at the template's own body).

- **Parameters:** `index` — how far "up" the instantiation stack to
  climb (`-1` — the immediate caller), `fullPaths` — whether to print the
  full file path.

```nim
template checkPositive(x: int) =
  let info = instantiationInfo()
  if x <= 0:
    echo "check failed at line ", info.line

# checkPositive(-1) will report the call site's line, not a line
# inside the template's own body
```

---

### 5. `procCall`

```nim
proc procCall*(x: untyped) {.compileTime.}
```

**What it does.** Inside an overridden method, calls the parent type's
version of the method — analogous to `super()` in object-oriented
languages. Only works for procedures using Nim's type-based dispatch
mechanism (`method`), not for regular `proc`.

- **Parameters:** `x: untyped` — the expression calling the parent's
  method (for example, `procCall(baseMethod(self))`).

```nim
type
  Animal = ref object of RootObj
  Dog = ref object of Animal

method speak(a: Animal): string {.base.} = "..."
method speak(d: Dog): string =
  # call the Animal implementation and extend it
  procCall(speak(Animal(d))) & " Woof!"
```

---

## IX. Output and Debugging

### 1. `echo`, `debugEcho`

```nim
proc echo*(x: varargs[typed, `$`]) {.magic: "Echo", gcsafe, sideEffect.}
proc debugEcho*(x: varargs[typed, `$`]) {.magic: "Echo", noSideEffect.}
```

**What it does.** `echo` prints the string representation (`$`) of each
argument to standard output, separated by spaces, with a trailing
newline. `debugEcho` does the same thing but is marked `noSideEffect` —
this lets you insert debug output even in places that formally forbid
side effects (for example, inside a `func`), without changing the
enclosing procedure's effect signature; intended purely for temporary
debugging, not for production code.

- **Parameters:** `x: varargs[typed, `$`]` — any number of values of any
  type for which a `$` conversion to a string is defined.

```nim
echo "x = ", 42, ", done: ", true
```

---

### 2. `repr` for `HSlice`

```nim
proc repr*[T, U](x: HSlice[T, U]): string
```

**What it does.** Builds a detailed string representation of a range in
the form `a..b`, using `repr` for each bound — differs from `$` in that
`repr` generally shows a value's internal structure in more detail than
the "human-readable" `$` representation does.

- **Parameters:** `x: HSlice[T, U]` — the range to represent as a string.

```nim
let r = 1 .. 5
assert repr(r) == "1 .. 5"
```

---

### 3. `locals`

```nim
proc locals*(): RootObj {.magic: "Plugin".}
```

**What it does.** Returns a pseudo-object containing all local variables
visible at the call site — a service feature for debugging tools and the
REPL; not intended for use in ordinary application code, since its result
has an unstable, compiler-specific representation.

```nim
proc demo() =
  let a = 1
  let b = "two"
  let vars = locals()
  # `vars` contains `a` and `b` in a form suitable for debug printing
```

---

## X. Miscellaneous Utilities

### 1. `likely`, `unlikely`

```nim
template likely*(val: bool): bool
template unlikely*(val: bool): bool
```

**What it does.** Hints to the compiler/processor about the likely
outcome of a condition — used to arrange "cold" branches of code (error
handling, rare cases) so that the main, "hot" execution path is predicted
better by the processor. Semantically, `likely(val)` and `val` are
equivalent — this only affects the generated machine code, not the
program's logic.

- **Parameters:** `val: bool` — the condition for which a probability
  hint is given.

```nim
proc process(x: int): string =
  if unlikely(x < 0):
    return "error: negative value"
  "ok: " & $x

assert process(5) == "ok: 5"
```

---

### 2. `once`, `closureScope`, `disarm`

```nim
template once*(body: untyped): untyped
template closureScope*(body: untyped): untyped
template disarm*(x: typed)
```

**What it does.** `once(body)` guarantees that `body` will run exactly
once over the entire lifetime of the program, even if the surrounding
code (say, a loop body) executes multiple times — implemented via a
hidden static flag that tracks "already ran". `closureScope(body)` wraps
`body` in a separate scope for closures — a typical problem without it:
closures created inside a loop all "see" the same loop variable at its
final value; `closureScope` gives each iteration its own copy of the
variables it needs. `disarm(x)` "defuses" an object before it's manually
freed, so that its destructor doesn't later try to free it a second time.

- **Parameters:** `body: untyped` — a block of code, `x: typed` — the
  variable to defuse before freeing.

```nim
for i in 1 .. 3:
  once:
    echo "this only prints once, when i == 1"

var closures: seq[proc(): int]
for i in 0 .. 2:
  closureScope:
    let captured = i
    add(closures, proc(): int = captured)

assert closures[0]() == 0
assert closures[1]() == 1
assert closures[2]() == 2
```

---

### 3. `finished`

```nim
proc finished*[T: iterator {.closure.}](x: T): bool
```

**What it does.** Checks whether a closure iterator (`closure iterator`)
has finished its work — that is, whether its values have been exhausted.
Useful when manually driving an iterator (not via `for`), when you need
to explicitly check whether it's worth requesting the next value.

- **Parameters:** `x` — a closure iterator.

```nim
iterator countTo(n: int): int {.closure.} =
  for i in 1 .. n:
    yield i

var it = countTo
while not finished(it):
  let v = it()
  if not finished(it):
    echo v
```

---

### 4. `arrayWith`, `arrayWithDefault`

```nim
proc arrayWith*[T](y: T, size: static int): array[size, T]
proc arrayWithDefault*[T](size: static int): array[size, T]
```

**What it does.** `arrayWith(y, size)` builds a fixed-size array of size
`size`, filled with `size` copies of value `y` (each element gets its own
independent copy via `=dup`, not the same shared reference).
`arrayWithDefault(size)` builds an array of the same size, filled with
`default(T)` for each element.

- **Parameters:** `y: T` — the fill value, `size: static int` — the
  array's size, known at compile time.

```nim
let filled = arrayWith(7, 4)
assert filled == [7, 7, 7, 7]

let zeros = arrayWithDefault[int](3)
assert zeros == [0, 0, 0]
```

---

## XI. Practical Recipes

### 1. Safely Building a seq of a Fixed Size

If the length of the result is known up front, `newSeq` plus indexed
writes is more efficient than repeated `add` calls, and `newSeqOfCap` is
the right choice when only an upper bound on the size is known:

```nim
proc squares(n: Natural): seq[int] =
  newSeq(result, n)
  for i in 0 ..< n:
    result[i] = i * i

assert squares(5) == @[0, 1, 4, 9, 16]

proc collectEven(upTo: int): seq[int] =
  result = newSeqOfCap[int](upTo div 2 + 1)
  for i in 0 .. upTo:
    if i mod 2 == 0:
      add(result, i)

assert collectEven(10) == @[0, 2, 4, 6, 8, 10]
```

---

### 2. Removing an Element Without Preserving Order vs. Preserving Order

```nim
proc removeFast[T](s: var seq[T], i: Natural) =
  # order doesn't matter — use O(1) removal
  del(s, i)

proc removeStable[T](s: var seq[T], i: Natural) =
  # order matters — use O(n) removal with a shift
  delete(s, i)

var unordered = @[1, 2, 3, 4, 5]
removeFast(unordered, 1)
assert unordered == @[1, 5, 3, 4]

var ordered = @[1, 2, 3, 4, 5]
removeStable(ordered, 1)
assert ordered == @[1, 3, 4, 5]
```

---

### 3. A Custom Exception Type With Context

Combining the exception hierarchy with `newException` to carry additional
context about an error, not just message text:

```nim
type
  ParseError = object of CatchableError
    lineNumber: int

proc parseLine(line: string, num: int) =
  if len(line) == 0:
    var e = newException(ParseError, "empty input line")
    e.lineNumber = num
    raise e

try:
  parseLine("", 42)
except ParseError as e:
  assert e.lineNumber == 42
  assert e.msg == "empty input line"
```

---

### 4. Moving Instead of Copying in an Accumulation Loop

When large intermediate values (strings, `seq`) need to be shuffled from
one variable to another without unnecessary copying:

```nim
proc drainInto(source: var seq[string], sink_: var seq[string]) =
  while len(source) > 0:
    let item = move(pop(source))
    add(sink_, item)

var src = @["a", "b", "c"]
var dst: seq[string] = @[]
drainInto(src, dst)
assert dst == @["c", "b", "a"]
assert len(src) == 0
```

---

### 5. Working With a String Slice Without Extra Allocations

When you only need to read part of a string (say, pass it to a procedure
that sums up bytes), `toOpenArrayByte` avoids the copy that `substr`
would make:

```nim
proc checksum(data: openArray[byte]): int =
  result = 0
  for b in data:
    result += int(b)

let payload = "header:body:footer"
let bodyView = toOpenArrayByte(payload, 7, 10)  # "body" with no copy
assert checksum(bodyView) == checksum(@[byte('b'), byte('o'), byte('d'), byte('y')])
```

---

## XII. Quick Reference Table

| Task | Mutates the argument | Returns a new value |
|---|---|---|
| Find the length | — | `len(x)` |
| Bounds of valid indices | — | `low(x)`, `high(x)` |
| Bounds of a type's values | — | `low(T)`, `high(T)` |
| Append an element/character to the end | `add(x, y)` | — |
| Remove without preserving order | `del(x, i)` | — |
| Remove while preserving order | `delete(x, i)` | — |
| Insert at an arbitrary position | `insert(x, item, i)` | — |
| Extract the last element | `pop(s)` (mutates and returns) | — |
| Change the length | `setLen(x, n)` | — |
| Create a seq of a given length | — | `newSeq[T](n)` |
| Reserve capacity without length | — | `newSeqOfCap[T](cap)` |
| Copy of part of a string | — | `substr(s, a, b)` |
| Non-owning slice (no copy) | — | `toOpenArray(x, a, b)` |
| Take an address | — | `addr(x)` |
| Take a value without copying | resets the source | `move(x)` |
| Check a type at compile time | — | `x is T` |
| Check an object's dynamic type | — | `x of T` |
| Check a reference for "emptiness" | — | `isNil(x)` |
| Build a default value (respecting field defaults) | — | `default(T)` |
| Build a default value (pure zeros) | — | `zeroDefault(T)` |
| Swap the contents of two variables | `swap(a, b)` | — |
| Check membership in a range/container | — | `contains(s, v)` / `v in s` |
| Terminate the program abruptly | terminates the process | `quit(msg)` |
| Create an exception with a message | — | `newException(T, msg)` |

---

## XIII. Summary: Which Procedure to Choose

- Need the length of a container → `len(x)`.
- Need the bounds for a manual loop `for i in ...` → `low(x) .. high(x)`,
  rather than `0 ..< len(x)` — this works correctly even for arrays with
  non-standard index types.
- Need to build a sequence of known length → `newSeq[T](n)`, then assign
  by index — faster than `add` in a loop.
- Only need to reserve memory for future growth via `add` →
  `newSeqOfCap[T](cap)`.
- Need to remove an element and order doesn't matter → `del(x, i)` (O(1)).
- Need to remove an element and order matters → `delete(x, i)` (O(n)).
- Need to read a piece of a string without copying it →
  `toOpenArray`/`toOpenArrayByte`, not `substr`.
- Need a copy of a piece of a string that can be freely modified →
  `substr(s, first, last)`.
- Need to pass a large value onward and the source variable is no longer
  needed → `move(x)`, rather than a regular assignment.
- Need to check a type at compile time (regular types, generic code) →
  `x is T`.
- Need to check an object's dynamic type in an inheritance hierarchy →
  `x of T`.
- Need a "default" value that respects user-defined `= value` defaults in
  the object's definition → `default(T)`; if you specifically need pure
  binary zeros, ignoring those defaults → `zeroDefault(T)`.
- Need your own exception with extra fields → declare a type descending
  from `CatchableError`, create it via `newException`, and fill in extra
  fields as needed before `raise`.
- Need to print a value for debugging without breaking `noSideEffect` on
  the enclosing procedure → `debugEcho`, not `echo`.
