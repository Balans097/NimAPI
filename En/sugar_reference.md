# Nim `sugar` Module — Complete Reference

> **Module purpose:** Syntactic sugar built on top of Nim's macro system. Provides concise notation for anonymous functions, procedure types, collection comprehensions, variable capture in closures, copy-and-modify idioms, and expression debugging — all without adding runtime overhead.
>
> **Import:** `import std/sugar`
>
> **Note on semicolons:** The `=>` and `->` macros parse parameter lists using parentheses and commas. Semicolons **cannot** be used to separate parameters.

---

## Table of Contents

1. [`=>` — Anonymous procedure (lambda)](#--anonymous-procedure-lambda)
2. [`->` — Procedure type notation](#--procedure-type-notation)
3. [`dump` — Debug-print an expression](#dump--debug-print-an-expression)
4. [`dumpToString` — Stringify an expression after semantic analysis](#dumptostring--stringify-an-expression-after-semantic-analysis)
5. [`capture` — Capture loop variables in closures](#capture--capture-loop-variables-in-closures)
6. [`dup` — Copy-and-modify idiom](#dup--copy-and-modify-idiom)
7. [`collect` (with init) — Collection comprehension](#collect-with-init--collection-comprehension)
8. [`collect` (without init) — Collection comprehension (auto-inferred)](#collect-without-init--collection-comprehension-auto-inferred)

---

## `=>` — Anonymous procedure (lambda)

```nim
macro `=>`*(p, b: untyped): untyped
```

### What it does

`=>` is the **lambda / anonymous procedure** operator. It lets you write inline functions without the full `proc` keyword ceremony. The left side is the parameter list; the right side is the body.

The macro expands to a full `nnkLambda` node, so the result is a first-class procedure value that can be assigned, passed as an argument, stored in a variable, or called immediately.

### Parameter list syntax

| Form | Meaning |
|---|---|
| `x => body` | Single untyped parameter named `x` |
| `(x, y) => body` | Multiple untyped parameters; types inferred |
| `(x: int, y: int) => body` | Explicitly typed parameters |
| `(x, y: int) => body` | Both `x` and `y` typed as `int` (trailing type applies to group) |
| `() => body` | No parameters (returns void or auto) |
| `(x, y) -> int => body` | Parameters with explicit **return type** via `->` |
| `(x: int) {.noSideEffect.} => body` | Pragma on the lambda |

### Examples

```nim
import std/sugar

# Simplest form — passed directly as a callback
proc apply(f: (int) -> int, x: int): int = f(x)
assert apply(x => x * x, 5) == 25
```

```nim
# Multiple parameters, types inferred
proc passTwoAndTwo(f: (int, int) -> int): int = f(2, 2)
assert passTwoAndTwo((x, y) => x + y) == 4
```

```nim
# Explicit types and explicit return type
let divide: (float, float) -> float = (a: float, b: float) -> float => a / b
assert divide(10.0, 4.0) == 2.5
```

```nim
# Lambda with pragma — stored in a struct field
type Bot = object
  call: (string {.noSideEffect.} -> string)

var bot = Bot()
bot.call = (name: string) {.noSideEffect.} => "Hello, " & name & "!"
assert bot.call("Alice") == "Hello, Alice!"
```

```nim
# Zero-argument lambda (returns void)
let greet = () => (echo "hi")
greet()
```

```nim
# Multi-statement body using a block
let process = (x: int) =>
  (
    let doubled = x * 2
    doubled + 1
  )
assert process(3) == 7
```

### Important notes

- The **return type** is `auto` by default (inferred). To make it explicit, put `-> ReturnType` between the parameter list and `=>`.
- Pragmas (e.g. `{.noSideEffect.}`, `{.gcsafe.}`) go between the parameter list and `=>`, after a space: `(x: int) {.noSideEffect.} => x + 1`.
- For zero-parameter lambdas that return `void`, wrap the body in `(discard)` or use a block: `() => (discard)`.

---

## `->` — Procedure type notation

```nim
macro `->`*(p, b: untyped): untyped
```

### What it does

`->` is the **proc type** operator. It creates a `proc (…): ReturnType` type expression from a concise notation. It is used in **type positions** — function signatures, type aliases, object fields — wherever you need to declare the type of a callable.

Without `->`, declaring a callback parameter type requires verbose syntax:

```nim
# Without sugar:
proc run(f: proc (x, y: int): int): int = f(1, 2)

# With sugar:
proc run(f: (int, int) -> int): int = f(1, 2)
```

### Syntax

```
(ParamType1, ParamType2, …) -> ReturnType
```

Named parameters are also supported using colon syntax:

```
(name: ParamType) -> ReturnType
```

Pragmas attach to the proc type:

```
(ParamType {.pragma.} -> ReturnType)
```

### Examples

```nim
import std/sugar

# Basic callback parameter type
proc transform(xs: seq[int], f: (int) -> int): seq[int] =
  result = newSeq[int](xs.len)
  for i, x in xs: result[i] = f(x)

assert transform(@[1, 2, 3], x => x * 10) == @[10, 20, 30]
```

```nim
# Multi-argument proc type
proc passTwoAndTwo(f: (int, int) -> int): int = f(2, 2)
assert passTwoAndTwo((x, y) => x + y) == 4
```

```nim
# Proc type with pragma
proc passOne(f: (int {.noSideEffect.} -> int)): int = f(1)
assert passOne(x {.noSideEffect.} => x + 1) == 2
```

```nim
# Type alias using ->
type Transformer = (string) -> string

let upper: Transformer = s => s.toUpperAscii()
assert upper("hello") == "HELLO"
```

```nim
# Object field with proc type
type Handler = object
  handle: (string, int) -> bool

var h = Handler(handle: (s, n) => s.len == n)
assert h.handle("abc", 3)
```

### Relationship with `=>`

`->` and `=>` are complementary:
- `->` appears in **type positions** to describe what a callable looks like.
- `=>` appears in **value positions** to create a callable that matches such a type.

```nim
proc run(f: (int) -> int): int = f(42)   # -> in type position
assert run(x => x + 1) == 43             # => in value position
```

---

## `dump` — Debug-print an expression

```nim
macro dump*(x: untyped): untyped
```

### What it does

`dump` prints both the **source text** of an expression and its **runtime value** to stderr, on a single line in the format `expression = value`. It uses `debugEcho` internally, so it goes to stderr and works even when stdout is redirected.

This is a quick inspection tool: you wrap any expression in `dump(…)` and immediately see what it evaluates to, without having to write a separate `echo` with string formatting.

### Example

```nim
import std/sugar

let x = 10
let y = 20
dump(x + y)      # prints: x + y = 30
dump(x * y - 5)  # prints: x * y - 5 = 195

var s = @[3, 1, 2]
dump(s)          # prints: s = @[3, 1, 2]
```

### When to use `dump` vs `dumpToString`

| Feature | `dump` | `dumpToString` |
|---|---|---|
| Output destination | stderr (via `debugEcho`) | Returns a `string` |
| Expands templates/macros | No — shows source text | Yes — shows expanded form |
| Works with statements | No | Yes |
| Useful for | Quick print debugging | Assertions, logging, testing |

---

## `dumpToString` — Stringify an expression after semantic analysis

```nim
macro dumpToString*(x: untyped): string
```

### What it does

`dumpToString` returns a `string` that describes both the **source form** and the **expanded/evaluated form** of an expression, separated by ` = `. Unlike `dump`, it performs full semantic analysis first — so intermediate template and macro calls are expanded and their result is shown, not the original call site.

The returned string follows this format:
- For expressions with a value: `"source: expanded = value"`
- For statements (void): `"source: expanded"`

Because it returns a `string`, you can use it in assertions, write it to a log, or inspect it at compile time.

### Examples

```nim
import std/sugar

const a = 1
let x = 10

# Constant folded — both source and value visible:
assert dumpToString(a + 2) == "a + 2: 3 = 3"

# Mix of const and runtime — expansion shows partial folding:
assert dumpToString(a + x) == "a + x: 1 + x = 11"

# Template call — expansion reveals what the template produces:
template square(n): untyped = n * n
assert dumpToString(square(x)) == "square(x): x * x = 100"

# Works with statements (void result):
import std/strutils
assert "failedAssertImpl" in dumpToString(assert true)
```

```nim
# Practical use: rich assertion messages
proc checkPositive(n: int) =
  assert n > 0, dumpToString(n > 0)

checkPositive(5)   # passes silently
# checkPositive(-1)  # would fail with a descriptive message
```

### Compile-time safety

`dumpToString` requires the expression to be **valid Nim** — it does not compile if the expression contains undefined symbols:

```nim
assert not compiles dumpToString(1 + nonexistent)  # true — it won't compile
```

---

## `capture` — Capture loop variables in closures

```nim
macro capture*(locals: varargs[typed], body: untyped): untyped
```

*(Available since Nim 1.1)*

### What it does

`capture` solves the classic **closure-over-loop-variable** problem. In Nim (as in many languages), a closure created inside a loop captures a reference to the loop variable — not its value at the time the closure was created. By the time the closure runs, the loop variable may have a different value (or be out of scope entirely).

`capture` works by generating an **immediately-invoked lambda** that receives the listed variables as value parameters, binding each variable's current value at the point of the `capture` call. The `body` runs inside that lambda, so it sees the frozen values.

### Syntax

```nim
capture varA, varB, …:
  body
```

### The problem it solves

```nim
import std/sugar

# WITHOUT capture — all closures share the same `i`
var closures: seq[() -> int]
for i in 0..2:
  closures.add(() => i)      # captures reference to i
# After the loop, i == 3 (or undefined), so:
# closures[0]() may return 3, not 0!

# WITH capture — each closure gets its own frozen copy of i
var closures2: seq[() -> int]
for i in 0..2:
  capture i:
    closures2.add(() => i)   # i is a fresh value parameter here

assert closures2[0]() == 0
assert closures2[1]() == 1
assert closures2[2]() == 2
```

### Multiple variables

```nim
import std/sugar, std/strformat

var myClosure: () -> string
for i in 5..7:
  for j in 7..9:
    if i * j == 42:
      capture i, j:
        myClosure = () => fmt"{i} * {j} = 42"

assert myClosure() == "6 * 7 = 42"
```

### Important constraints

- The variable name **`result`** is forbidden as a capture argument (it has special meaning inside procedures).
- Only plain identifiers can be captured — arbitrary expressions are not supported.
- The `body` runs immediately (the generated lambda is called at the point of the `capture` statement), so it is not deferred.

---

## `dup` — Copy-and-modify idiom

```nim
macro dup*[T](arg: T, calls: varargs[untyped]): T
```

*(Available since Nim 1.2)*

### What it does

`dup` turns **in-place mutation** procedures into **pure, copy-returning** operations without writing an explicit `var copy = original; mutate(copy); return copy` pattern every time.

It creates a fresh mutable copy of `arg`, applies a sequence of calls to that copy, and returns the modified copy — leaving the original unchanged. The original stays intact.

This is particularly useful with standard library procedures like `sort`, `insert`, `reverse`, `add`, etc., which mutate their first argument in place.

### Syntax

```nim
original.dup(call1, call2, …)

# Block form (equivalent):
dup original:
  call1
  call2
```

In each call, the copy is passed as the **first argument** by default. You can explicitly mark its position with an underscore `_`:

```nim
original.dup(addQuoted(_, "foo"))   # _ = the copy
original.dup(addQuoted("foo"))      # same — _ is optional when first
```

### Examples

```nim
import std/sugar, std/algorithm

let a = @[3, 1, 4, 1, 5, 9]

# Sort a copy, leave original unchanged:
let sorted = a.dup(sort)
assert a == @[3, 1, 4, 1, 5, 9]     # original untouched
assert sorted == @[1, 1, 3, 4, 5, 9]

# Chain multiple in-place operations:
var aCopy = a
aCopy.insert(10)
assert a.dup(insert(10), sort) == sorted(aCopy)
```

```nim
# Works with strings and operators:
let s1 = "abc"
let s2 = "xyz"
assert s1 & s2 == s1.dup(&= s2)
```

```nim
# Block form with custom mutating proc:
proc makePalindrome(s: var string) =
  for i in countdown(s.len - 2, 0):
    s.add(s[i])

let c = "xyz"
let d = dup c:
  makePalindrome          # "xyzyx"
  sort(_, SortOrder.Descending)  # "zyyxx"
  makePalindrome          # "zyyxxxyyz"
assert d == "zyyxxxyyz"
```

```nim
# Underscore for explicit argument placement:
assert "".dup(addQuoted(_, "foo")) == "\"foo\""
assert "".dup(addQuoted("foo"))    == "\"foo\""  # _ optional here
```

### Key properties

- The **original is never modified** — `dup` always works on a fresh copy.
- Multiple calls are applied **in order**, each to the same running copy.
- The `_` placeholder lets you target any argument position, not just the first.
- The return type `T` is the same as the input type — no type conversion occurs.

---

## `collect` (with init) — Collection comprehension

```nim
macro collect*(init, body: untyped): untyped
```

*(Available since Nim 1.1)*

### What it does

`collect` is Nim's **list/set/table comprehension**. It evaluates a loop body and accumulates results into a collection, inferring the element type automatically. The `init` argument specifies **what kind of collection** to build and how to initialise it. The `body` is a loop (possibly with conditions) whose last expression determines what gets added.

The syntax of the final expression in the body determines the collection type:

| Body syntax | Collection built | Operation |
|---|---|---|
| `value` (plain expression) | `seq[T]` | `add` |
| `{value}` (curly set) | `HashSet[T]` | `incl` |
| `{key: value}` (curly table) | `Table[K, V]` | `[]=` |

### Examples

```nim
import std/sugar, std/sets, std/tables

let data = @["bird", "word"]

# seq — with newSeq:
let k = collect(newSeq):
  for i, d in data.pairs:
    if i mod 2 == 0: d          # plain expression → seq.add
assert k == @["bird"]

# seq — with pre-allocated capacity:
let x = collect(newSeqOfCap(4)):
  for i, d in data.pairs:
    if i mod 2 == 0: d
assert x == @["bird"]

# HashSet — {value} syntax:
let y = collect(initHashSet()):
  for d in data.items: {d}      # {d} → HashSet.incl
assert y == data.toHashSet

# Table — {key: value} syntax:
let z = collect(initTable(2)):
  for i, d in data.pairs: {i: d}  # {i: d} → Table[]=
assert z == {0: "bird", 1: "word"}.toTable
```

```nim
# Filtering and transforming in one expression:
let numbers = @[1, 2, 3, 4, 5, 6, 7, 8]
let evenSquares = collect(newSeq):
  for n in numbers:
    if n mod 2 == 0: n * n
assert evenSquares == @[4, 16, 36, 64]
```

```nim
# Nested loops — cartesian product:
let pairs = collect(newSeq):
  for x in 1..3:
    for y in 1..3:
      if x != y: (x, y)
```

### Notes

- The element type is inferred from the expression — you do not specify it in `init`.
- Conditional expressions with `if` are fine; only iterations where the condition is true produce a value.
- For `Table`, both the key type and value type are inferred separately.

---

## `collect` (without init) — Collection comprehension (auto-inferred)

```nim
macro collect*(body: untyped): untyped
```

*(Available since Nim 1.5)*

### What it does

This is the **no-argument overload** of `collect`. It does exactly the same thing as the two-argument form but infers the collection constructor automatically from the body's final expression syntax — you do not need to pass `newSeq`, `initHashSet()`, or `initTable()` explicitly.

The same body syntax rules apply:

| Body syntax | Collection built |
|---|---|
| Plain expression | `seq[T]` |
| `{value}` | `HashSet[T]` |
| `{key: value}` | `Table[K, V]` |

### Examples

```nim
import std/sugar, std/sets, std/tables

let data = @["bird", "word"]

# seq — no init needed:
let k = collect:
  for i, d in data.pairs:
    if i mod 2 == 0: d
assert k == @["bird"]

# HashSet — auto-detected from {d}:
let n = collect:
  for d in data.items: {d}
assert n == data.toHashSet

# Table — auto-detected from {i: d}:
let m = collect:
  for i, d in data.pairs: {i: d}
assert m == {0: "bird", 1: "word"}.toTable
```

```nim
# Practical example — word frequency map:
let words = @["apple", "banana", "apple", "cherry", "banana", "apple"]
# (This builds a Table counting occurrences)
# Note: for counting, a manual approach is clearer, but collect works
# for straightforward key-value mappings:
let firstSeen = collect:
  for i, w in words.pairs: {w: i}
# firstSeen maps each word to its last seen index (later entries overwrite)
```

### When to prefer the no-init form

Use the no-init form when:
- You do not need a specific initial capacity hint.
- The collection type is obvious from the body.

Use `collect(init, body)` when:
- You want to pre-allocate with `newSeqOfCap`, `initTable(capacity)`, etc.
- The extra hint improves performance for large collections.

---

## Complete usage example

```nim
import std/sugar, std/algorithm, std/sets, std/tables, std/strformat

# --- Lambdas and proc types ---
proc applyAll(xs: seq[int], f: (int) -> int): seq[int] =
  collect:
    for x in xs: f(x)

assert applyAll(@[1, 2, 3], x => x * x) == @[1, 4, 9]

# --- dup: sort without mutating original ---
let raw = @[5, 3, 8, 1, 9, 2]
let ordered = raw.dup(sort)
assert raw == @[5, 3, 8, 1, 9, 2]  # untouched
assert ordered == @[1, 2, 3, 5, 8, 9]

# --- capture: safe closures in loops ---
var actions: seq[() -> string]
for i in 0..2:
  capture i:
    actions.add(() => fmt"action {i}")
assert actions[0]() == "action 0"
assert actions[2]() == "action 2"

# --- collect: comprehensions ---
let nums = @[1..10].concat
let oddSquares = collect:
  for n in nums:
    if n mod 2 != 0: n * n
assert oddSquares == @[1, 9, 25, 49, 81]

let wordSet = collect:
  for w in @["nim", "is", "fun", "nim"]: {w}
assert wordSet == ["nim", "is", "fun"].toHashSet

# --- dumpToString: descriptive assertions ---
template double(x): untyped = x + x
let v = 7
assert dumpToString(double(v)) == "double(v): v + v = 14"
```

---

*Reference covers Nim standard library `std/sugar` as found in the module source.*
