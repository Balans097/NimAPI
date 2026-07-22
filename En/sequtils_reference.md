# sequtils — Module Reference

> **Import:** `import std/sequtils`
> **Scope:** functional-style operations over `openArray` types — `seq`, `string`, and `array` (creation, mapping, filtering, folding, searching, and working with pairs of sequences).

The `sequtils` module doesn't introduce its own data types — it builds a
functional layer on top of `seq`, `string`, and `array`, adding
functional-programming-style operations: `map`/`filter`/`fold`, extremum
search, merging and splitting pairs of sequences, generation and
deduplication. A consistent naming convention runs through the module:
many procedures have an "expression" variant with an `-It` suffix
(`mapIt`, `filterIt`, `allIt`, `anyIt`, `countIt`, `keepItIf`) — these are
templates that, instead of taking a `proc`, accept an arbitrary
**expression** using an implicitly injected `it` variable. Another
convention is the pair of "non-mutating / in-place mutating" procedures:
`filter` returns a new sequence, `keepIf` mutates the original; `map`
returns a new sequence, `apply` mutates the original.

---

## Table of Contents

I. [Creating and Generating Sequences](#i-creating-and-generating-sequences)
   1. [`toSeq`](#1-toseq)
   2. [`repeat`](#2-repeat)
   3. [`newSeqWith`](#3-newseqwith)
   4. [`cycle`](#4-cycle)
   5. [`concat`](#5-concat)
   6. [`distribute`](#6-distribute)
   7. [`items` (over a closure iterator)](#7-items-over-a-closure-iterator)

II. [Mapping](#ii-mapping)
   1. [`map`](#1-map)
   2. [`mapIt`](#2-mapit)
   3. [`apply` (three variants)](#3-apply-three-variants)
   4. [`applyIt`](#4-applyit)
   5. [`mapLiterals`](#5-mapliterals)

III. [Filtering](#iii-filtering)
   1. [`filter` (proc)](#1-filter-proc)
   2. [`filter` (iterator)](#2-filter-iterator)
   3. [`filterIt`](#3-filterit)
   4. [`keepIf`](#4-keepif)
   5. [`keepItIf`](#5-keepitif)

IV. [Searching, Predicate Checks, and Aggregate Metrics](#iv-searching-predicate-checks-and-aggregate-metrics)
   1. [`findIt`](#1-findit)
   2. [`any` / `anyIt`](#2-any--anyit)
   3. [`all` / `allIt`](#3-all--allit)
   4. [`count` / `countIt`](#4-count--countit)
   5. [`min` / `max`](#5-min--max)
   6. [`minIndex` / `maxIndex`](#6-minindex--maxindex)
   7. [`minmax`](#7-minmax)

V. [Folding](#v-folding)
   1. [`foldl` (without a starting value)](#1-foldl-without-a-starting-value)
   2. [`foldl` (with a starting value)](#2-foldl-with-a-starting-value)
   3. [`foldr`](#3-foldr)

VI. [Working with Pairs of Sequences](#vi-working-with-pairs-of-sequences)
   1. [`zip`](#1-zip)
   2. [`unzip`](#2-unzip)

VII. [In-Place Composition Changes](#vii-in-place-composition-changes)
   1. [`addUnique`](#1-addunique)
   2. [`deduplicate`](#2-deduplicate)
   3. [`delete`](#3-delete)
   4. [`insert`](#4-insert)

VIII. [Practical Recipes](#viii-practical-recipes)
   1. [Top-N Leaders by Criterion](#1-top-n-leaders-by-criterion)
   2. [Deduplication Preserving First Occurrence](#2-deduplication-preserving-first-occurrence)
   3. [Round-Robin Task Distribution Across Workers](#3-round-robin-task-distribution-across-workers)
   4. [Merging Two Parallel Lists into a Report](#4-merging-two-parallel-lists-into-a-report)
   5. [Rolling Statistics over a Numeric Log](#5-rolling-statistics-over-a-numeric-log)

IX. [Quick Reference Table](#ix-quick-reference-table)

X. [Summary: Which Procedure to Choose](#x-summary-which-procedure-to-choose)

---

## I. Creating and Generating Sequences

### 1. `toSeq`

```nim
template toSeq*(iter: untyped): untyped
```

**What it does.** Turns any iterable thing — a range, a set, an
iterator, anything with `items` — into a plain `seq`. It's the only
procedure in the module explicitly designed to accept **iterators**
(including inline ones), not just `openArray`.

**Implementation notes.** `toSeq` isn't a single procedure but a
three-branch dispatcher selected at compile time via `when
compiles(...)`. `toSeq1` handles ordinary "typed" containers (`seq`,
`array`, `set`, ranges) — if the object has `len`, the result is
allocated at the right size up front (`newSeq[OutType](len(s2))`), saving
on reallocations; if `len` isn't available, elements accumulate via
`add`. `toSeq2` handles named closure iterators. The final branch is the
fallback for untyped inline iterators like `toSeq(myInlineIterator(3))`.
The branch is chosen without any input from the caller — it just writes
`toSeq(iter)`.

- **Parameters:**
  - `iter: untyped` — anything that supports traversal: a range
    (`1..5`), a set, a `seq`/`array`, a named or inline iterator.

**Examples:**

```nim
let
  range = 1..5
  numbers = toSeq(range)
assert numbers == @[1, 2, 3, 4, 5]

iterator evens(n: int): int =
  for i in 0..<n:
    if i mod 2 == 0:
      yield i

let result = toSeq(evens(10))
assert result == @[0, 2, 4, 6, 8]

# Edge case: an empty range yields an empty sequence.
let empty = toSeq(1..0)
assert empty == newSeq[int]()
```

---

### 2. `repeat`

```nim
proc repeat*[T](x: T, n: Natural): seq[T]
```

**What it does.** Builds a new sequence made of `n` copies of the same
value `x`. If `n == 0`, an empty sequence is returned — no exception is
raised.

**Implementation notes.** The implementation is trivial and linear: the
result is allocated at the exact size up front via `newSeq[T](n)` (a
single allocation, no subsequent `add` calls), then every index is
simply overwritten with a copy of `x`. Complexity is O(n) in both time
and memory.

- **Parameters:**
  - `x: T` — the value to replicate (copied `n` times).
  - `n: Natural` — how many copies to make; `Natural` guarantees
    negative values are rejected at the type level.

**Examples:**

```nim
let total = repeat(5, 3)
assert total == @[5, 5, 5]

# Edge case: n = 0 — an empty sequence.
let empty = repeat("x", 0)
assert empty == newSeq[string]()

# Practical scenario: building a separator string.
let separator = repeat('-', 20)
assert len(separator) == 20
```

---

### 3. `newSeqWith`

```nim
template newSeqWith*(len: int, init: untyped): untyped
```

**What it does.** Creates a new sequence of length `len`, where each
element is initialized by a separate evaluation of the `init`
expression — unlike `repeat`, `init` is evaluated fresh for **every**
index, which matters for non-primitive values (nested `seq`s, random
numbers) that can't simply be copied.

**Implementation notes.** The result type is inferred from
`typeof(init)`. For types that support byte-wise copying
(`supportsCopyMem`), `newSeqUninit` is used — a fast allocation that
skips zero-initializing memory, since it will be overwritten immediately
in the loop anyway; for other types, an ordinary `newSeq` is used. The
loop `for i in 0 ..< newLen` re-evaluates `init` on every iteration, so
`newSeqWith(5, newSeq[bool](3))` creates five **independent** inner
sequences, not five references to the same one.

- **Parameters:**
  - `len: int` — the required length of the resulting sequence.
  - `init: untyped` — an expression evaluated separately for each
    element; its result type sets the element type of the `seq`.

**Examples:**

```nim
# A "2D" sequence: five independent bool sequences of length 3.
var grid = newSeqWith(5, newSeq[bool](3))
assert len(grid) == 5
assert len(grid[0]) == 3
assert grid[4][2] == false

# Independence: mutating one nested seq doesn't affect the others.
grid[0][0] = true
assert grid[1][0] == false

# Practical scenario: a sequence of random numbers (rand(1.0) is
# evaluated afresh each time, so the values differ).
import std/random
var randomSeq = newSeqWith(20, rand(1.0))
assert len(randomSeq) == 20
```

---

### 4. `cycle`

```nim
func cycle*[T](s: openArray[T], n: Natural): seq[T]
```

**What it does.** Returns a new sequence in which the whole content of
`s` is repeated `n` times in a row. If `n == 0` (or `s` is empty), the
result is an empty sequence.

**Implementation notes.** The result is allocated up front in one shot —
`newSeq[T](n * len(s))` — then filled by two nested loops: the outer
loop repeats the "pass" `n` times, the inner loop copies the elements of
`s` one after another with a running counter `o`, incremented via
`unCheckedInc` (an overflow-check-free increment — a micro-optimization,
since `o` is guaranteed not to exceed the result's size). Complexity is
O(n × len(s)).

- **Parameters:**
  - `s: openArray[T]` — the source sequence pattern (not modified).
  - `n: Natural` — how many times to repeat the content of `s`.

**Examples:**

```nim
let
  s = @[1, 2, 3]
  total = cycle(s, 3)
assert total == @[1, 2, 3, 1, 2, 3, 1, 2, 3]

# Edge case: n = 0 — empty result regardless of s's content.
assert cycle(s, 0) == newSeq[int]()

# Practical scenario: a pattern for a texture or progress indicator.
let pattern = cycle(@['.', '.', 'o'], 4)
assert len(pattern) == 12
```

---

### 5. `concat`

```nim
func concat*[T](seqs: varargs[seq[T]]): seq[T]
```

**What it does.** Takes any number of sequences of the same type and
concatenates their elements into one new sequence — in the order the
arguments were passed. The reverse operation is
[`distribute`](#6-distribute), which splits one sequence into several.

**Implementation notes.** First, a single pass over `seqs` computes the
total length `L` of all input sequences, so the result can be allocated
in one shot (`newSeq(result, L)`) — with no further reallocations while
adding elements. A second pass then copies each sequence's elements in
turn, using an unchecked counter `i` (`unCheckedInc`), since it's
guaranteed not to exceed `L`.

- **Parameters:**
  - `seqs: varargs[seq[T]]` — any number of sequences of type `T`,
    passed as separate arguments.

**Examples:**

```nim
let
  s1 = @[1, 2, 3]
  s2 = @[4, 5]
  s3 = @[6, 7]
  total = concat(s1, s2, s3)
assert total == @[1, 2, 3, 4, 5, 6, 7]

# Edge case: empty sequences in the argument list simply contribute
# no elements.
assert concat(s1, newSeq[int](), s3) == @[1, 2, 3, 6, 7]

# Practical scenario: merging several pages of a paginated API result
# into a single list.
let
  page1 = @["a", "b"]
  page2 = @["c", "d", "e"]
  allRecords = concat(page1, page2)
assert len(allRecords) == 5
```

---

### 6. `distribute`

```nim
func distribute*[T](s: seq[T], num: Positive, spread = true): seq[seq[T]]
```

**What it does.** Splits sequence `s` into `num` sub-sequences. If
`num < 2`, the result is a single element, equal to the whole of `s`.
If `s` is empty, the result is `num` empty sub-sequences. The `spread`
parameter controls **how** the remainder of `len(s) div num` is
distributed: `spread = true` (the default) spreads the remainder evenly
across all sub-sequences (handy for distributing work across threads —
every worker gets a nearly equal load); `spread = false` gives the
entire remainder to the last sub-sequence, while the earlier ones get
exactly `1 + len(s) div num` elements each.

**Implementation notes.** The base step `stride = len(s) div num` and
remainder `extra = len(s) mod num` are computed once. When `spread =
false` (or there's no remainder), an "over-counting" algorithm is used:
if there's a remainder, `stride` is bumped up by one in advance, and the
upper bound of each chunk is taken as `min(len(s), first + stride)` — so
the last chunk automatically ends up shorter if the total length runs
out. When `spread = true`, an "under-counting" algorithm is used: the
base `stride` is left as is, and the remainder `extra` is **spent** one
element at a time on the first `extra` sub-sequences — hence the even
distribution of the surplus.

- **Parameters:**
  - `s: seq[T]` — the source sequence (not modified).
  - `num: Positive` — how many parts to split into; at `1`, the result
    is `s` itself wrapped in an extra `seq`.
  - `spread: bool` (default `true`) — spread the remainder evenly
    (`true`) or give it all to the last part (`false`).

**Examples:**

```nim
let numbers = @[1, 2, 3, 4, 5, 6, 7]
assert distribute(numbers, 3) == @[@[1, 2, 3], @[4, 5], @[6, 7]]
assert distribute(numbers, 3, false) == @[@[1, 2, 3], @[4, 5, 6], @[7]]

# Edge case: num greater than the length of s — some sub-sequences
# end up empty.
assert distribute(numbers, 6)[0] == @[1, 2]
assert distribute(numbers, 6)[1] == @[3]

# Practical scenario: distributing tasks across a thread pool —
# every worker gets roughly equal load.
let
  tasks = toSeq(1..10)
  perWorker = distribute(tasks, 4)
assert len(perWorker) == 4
```

---

### 7. `items` (over a closure iterator)

```nim
iterator items*[T](xs: iterator: T): T
```

**What it does.** A helper iterator that lets you walk a closure
iterator (`iterator: T`) exactly like an ordinary container — wrapping
`for x in xs(): yield x`. On its own it's rarely used directly; its
practical value is that it opens up templates like `mapIt`, `filterIt`,
`allIt`, and `anyIt` — which internally rely on `items(s)` — to closure
iterators as well.

**Implementation notes.** The implementation is a one-line pass: the
iterator calls the passed closure iterator `xs()` and forwards
(`yield`s) each of its values as its own. No additional data structures
are created.

- **Parameters:**
  - `xs: iterator: T` — a closure iterator (not an inline one) that
    yields values of type `T`.

**Examples:**

```nim
# Wrapping a closure iterator so it can be used with filterIt.
var counter = 0
let iter = iterator (): int {.closure.} =
  while counter < 5:
    yield counter
    inc(counter)

var collected: seq[int]
for x in items(iter):
  add(collected, x)
assert collected == @[0, 1, 2, 3, 4]
```

---

## II. Mapping

### 1. `map`

```nim
proc map*[T, S](s: openArray[T], op: proc (x: T): S {.closure.}): seq[S]
```

**What it does.** Returns a new sequence the same size as `s`, where
each element is the result of applying `op` to the corresponding element
of `s`. The source container is not modified. Since `op` can return a
type `S` different from `T`, `map` also serves as a way to convert a
container's element type (e.g. `seq[int]` into `seq[string]`).

**Implementation notes.** The result is allocated at the right size up
front (`newSeq(result, len(s))`), then a simple index loop calls
`op(s[i])` and writes the result into `result[i]`. No intermediate
accumulation via `add` is needed — a single O(n) pass.

- **Parameters:**
  - `s: openArray[T]` — the source container (not modified).
  - `op: proc (x: T): S {.closure.}` — the per-element transform
    function; it may change the type (`T` → `S`).

**Examples:**

```nim
let
  a = @[1, 2, 3, 4]
  b = map(a, proc(x: int): string = $x)
assert b == @["1", "2", "3", "4"]

# Edge case: an empty container — an empty result.
let empty: seq[int] = @[]
assert map(empty, proc(x: int): int = x * 2) == newSeq[int]()

# Practical scenario: converting a list of Celsius temperatures to
# Fahrenheit.
let
  celsius = @[0.0, 20.0, 37.0, 100.0]
  fahrenheit = map(celsius, proc(c: float): float = c * 9.0 / 5.0 + 32.0)
assert fahrenheit[2] == 98.6
```

---

### 2. `mapIt`

```nim
template mapIt*(s: typed, op: untyped): untyped
```

**What it does.** A syntactically lighter version of `map`: instead of
a separately declared `proc`, it uses an expression `op` where the
current element is available through the injected `it` variable.
Behavior (a result the same size as `s`, with a possibly transformed
type) is identical to `map`.

**Implementation notes.** The result type (`OutType`) is inferred by the
compiler by substituting a dummy `it` into the `op` expression. If the
resulting type is *not* a `proc` (the common case), the template avoids
creating closures inside the loop: if `s` has `len`, the result is
pre-allocated (`newSeq[OutType](len(s2))`) and filled by index;
otherwise it's accumulated via `add`. A special case is when `op`
itself constructs a `proc` (e.g. `mapIt([1, 2], (x: int) => it + x)`):
here `mapIt` avoids a manual loop and instead builds a helper function
`f` that closes over `it`, delegating the work to the ordinary `map` —
otherwise, every iteration would produce procs identical in code but
different in captured `it`, which is both less efficient and less
predictable.

- **Parameters:**
  - `s: typed` — the source container, supporting `items`.
  - `op: untyped` — an expression using the `it` variable, defining the
    per-element transform.

**Examples:**

```nim
let
  nums = @[1, 2, 3, 4]
  strings = mapIt(nums, $(4 * it))
assert strings == @["4", "8", "12", "16"]

# Edge case: a single element.
assert mapIt(@[7], it * it) == @[49]

# Practical scenario: mapIt returning a closure — mapIt delegates to
# map instead of producing a fresh proc on every iteration.
import std/sugar
let adders = mapIt([1, 2], (x: int) => it + x)
assert adders[0](10) == 11
assert adders[1](10) == 12
```

---

### 3. `apply` (three variants)

```nim
proc apply*[T](s: var openArray[T], op: proc (x: var T) {.closure.})
proc apply*[T](s: var openArray[T], op: proc (x: T): T {.closure.})
proc apply*[T](s: openArray[T], op: proc (x: T) {.closure.})
```

**What it does.** The in-place counterpart of `map`. The first variant
passes each element to `op` **by reference** (`var T`) — `op` mutates it
directly. The second variant takes an `op` that receives a `T` and
returns a new `T` — the result is written back to the same slot. The
third variant is for an `op` with no return value and no `var`
mutation; `s` itself here can even be non-mutable from the caller's
perspective (`openArray[T]` without `var`), since `apply` in this case
is used purely for `op`'s side effects (e.g. accumulating into an
outer variable).

**Implementation notes.** All three variants are a simple index loop
over `0 ..< len(s)`, differing only in what happens to `op`'s result:
the first calls `op(s[i])` without capturing a return value (mutation
already happened via `var T`); the second does `s[i] = op(s[i])`; the
third simply calls `op(s[i])`, ignoring both mutation and return value.

- **Parameters:**
  - `s: var openArray[T]` (variants 1–2) or `openArray[T]` (variant 3)
    — the container mutated in place (except in variant 3).
  - `op` — the function whose shape determines which of the three
    variants the compiler picks.

**Examples:**

```nim
# Variant 1: op mutates the element via var.
var a1 = @["1", "2", "3", "4"]
apply(a1, proc(x: var string) = x &= "42")
assert a1 == @["142", "242", "342", "442"]

# Variant 2: op takes and returns T.
var a2 = @["1", "2", "3", "4"]
apply(a2, proc(x: string): string = x & "42")
assert a2 == @["142", "242", "342", "442"]

# Variant 3: op returns nothing — used purely for its side effect.
var message: string
apply([0, 1, 2, 3, 4], proc(item: int) = addInt(message, item))
assert message == "01234"
```

---

### 4. `applyIt`

```nim
template applyIt*(varSeq, op: untyped)
```

**What it does.** A convenience wrapper around the mutating variant of
`apply` — no need to write a `proc`. The `op` expression uses the `it`
variable and must return a value of the same type as the container's
elements; that value overwrites the element.

**Implementation notes.** The loop runs over indices `low(varSeq) ..
high(varSeq)`; on each iteration `it` is declared as a `let` holding the
current element's value, `op` is evaluated, and the result is written
back into `varSeq[i]`. Since `it` is constant while `op` is evaluated,
the expression cannot rely on a "previously mutated" value — only on
the element's original value.

- **Parameters:**
  - `varSeq: untyped` — a mutable container (`var seq`/`var array`).
  - `op: untyped` — an expression using `it`, returning the new
    element value.

**Examples:**

```nim
var nums = @[1, 2, 3, 4]
applyIt(nums, it * 3)
assert nums[0] + nums[3] == 15

# Practical scenario: upper-casing a list of strings in place.
import std/strutils
var words = @["hello", "world"]
applyIt(words, toUpperAscii(it))
assert words == @["HELLO", "WORLD"]
```

---

### 5. `mapLiterals`

```nim
macro mapLiterals*(constructor, op: untyped; nested = true): untyped
```

**What it does.** Applies `op` not to a sequence's elements at
runtime, but to every atomic **literal** (`3`, `"abc"`, `1.2`) found in
the passed-in AST constructor — at compile time. Useful when you need to
convert every literal inside an array/tuple to a different type at
once, without manually spelling out each call. With `nested = true`
(the default), `op` is applied to literals at any nesting depth
(tuples inside tuples); with `nested = false`, only first-level
literals are touched, and nested structures are left alone.

**Implementation notes.** The macro recursively walks the passed-in
AST (`mapLitsImpl`): if the current node is a literal of the relevant
kind (`nnkLiterals` by default), it's wrapped in a call to `op`;
otherwise the node is copied (`copyNimNode`) and its children are
processed recursively — but only if `nested = true`, or the child is
itself a literal. Because this happens at compile time, `op` here can
be not only a `proc`, but also an arbitrary type converter such as
`int` or `$`.

- **Parameters:**
  - `constructor: untyped` — an AST expression: an array, a tuple, or a
    nested combination of these with literals.
  - `op: untyped` — a function or type converter applied to every
    matching literal.
  - `nested: bool` (default `true`) — whether to apply `op` at every
    nesting level or only at the first one.

**Examples:**

```nim
let x = mapLiterals([0.1, 1.2, 2.3, 3.4], int)
assert x is array[4, int]
assert x == [int(0.1), int(1.2), int(2.3), int(3.4)]

# nested = true (the default) — literals are replaced at any depth.
let a = mapLiterals((1.2, (2.3, 3.4), 4.8), int)
assert a == (1, (2, 3), 4)

# nested = false — only first-level literals.
let b = mapLiterals((1.2, (2.3, 3.4), 4.8), int, nested = false)
assert b == (1, (2.3, 3.4), 4)
```

---

## III. Filtering

### 1. `filter` (proc)

```nim
proc filter*[T](s: openArray[T], pred: proc(x: T): bool {.closure.}): seq[T]
```

**What it does.** Returns a new sequence containing only the elements
of `s` for which the predicate `pred` returned `true`. Order is
preserved, and the source container is not modified. The in-place
counterpart is [`keepIf`](#4-keepif).

**Implementation notes.** The result starts empty (`newSeq[T]()`), and
an index loop simply `add`s matching elements. Unlike `map`, the exact
result size can't be pre-allocated — the number of matching elements is
unknown up front — so dynamic appending is used (the `seq` may
reallocate internally).

- **Parameters:**
  - `s: openArray[T]` — the source container (not modified).
  - `pred: proc(x: T): bool {.closure.}` — the predicate: an element
    stays in the result if the predicate returns `true`.

**Examples:**

```nim
let
  colors = @["red", "yellow", "black"]
  f1 = filter(colors, proc(x: string): bool = len(x) < 6)
  f2 = filter(colors, proc(x: string): bool = contains(x, 'y'))
assert f1 == @["red", "black"]
assert f2 == @["yellow"]

# Edge case: no element matches — an empty result.
assert filter(colors, proc(x: string): bool = len(x) > 100) == newSeq[string]()

# Practical scenario: picking log lines by severity.
import std/strutils
let
  log = @["INFO: started", "ERROR: disk failure", "INFO: ready", "ERROR: timeout"]
  errors = filter(log, proc(line: string): bool = strutils.startsWith(line, "ERROR"))
assert len(errors) == 2
```

---

### 2. `filter` (iterator)

```nim
iterator filter*[T](s: openArray[T], pred: proc(x: T): bool {.closure.}): T
```

**What it does.** An iterator version of the same selection: instead of
building a whole new sequence, elements passing `pred` are yielded one
at a time as traversal proceeds. Useful when the result is consumed
right away in a loop and materializing an intermediate `seq` would be
wasteful.

**Implementation notes.** An index loop over `0 ..< len(s)` checks
`pred(s[i])` and does `yield s[i]` when the predicate holds — without
allocating any extra memory for the result inside the iterator itself
(only the consuming code spends memory, if it chooses to accumulate
values).

- **Parameters:**
  - `s: openArray[T]` — the source container (not modified).
  - `pred: proc(x: T): bool {.closure.}` — the selection predicate.

**Examples:**

```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
var evens = newSeq[int]()
for n in filter(numbers, proc (x: int): bool = x mod 2 == 0):
  add(evens, n)
assert evens == @[4, 8, 4]

# Practical scenario: processing on the fly without an intermediate
# seq — summing the even numbers without collecting them separately.
var evenSum = 0
for n in filter(numbers, proc (x: int): bool = x mod 2 == 0):
  inc(evenSum, n)
assert evenSum == 16
```

---

### 3. `filterIt`

```nim
template filterIt*(s, pred: untyped): untyped
```

**What it does.** The "expression" variant of `filter`: `pred` is an
expression using `it` rather than a `proc`. Returns a new sequence;
order and semantics match `filter`.

**Implementation notes.** The result's element type is taken from
`typeof(s[0])`, so `filterIt` requires that `s` support indexing at
least logically (via `items`, from which the first element is drawn for
type inference). The loop `for it {.inject.} in items(s)` checks `pred`
on each pass and `add`s matching `it` values to the result; at the end,
`result` is returned via `move` — avoiding an unnecessary copy of the
accumulated sequence when leaving the template.

- **Parameters:**
  - `s: untyped` — the source container.
  - `pred: untyped` — a predicate expression using `it`.

**Examples:**

```nim
let
  temperatures = @[-272.15, -2.0, 24.5, 44.31, 99.9, -113.44]
  acceptable = filterIt(temperatures, it < 50 and it > -10)
  notAcceptable = filterIt(temperatures, it > 50 or it < -10)
assert acceptable == @[-2.0, 24.5, 44.31]
assert notAcceptable == @[-272.15, 99.9, -113.44]

# Practical scenario: stripping vowels from a string (the module works
# on strings as openArray[char] too).
import std/strutils
let
  vowels = @['a', 'e', 'i', 'o', 'u']
  foo = "sequtils is an awesome module"
assert join(filterIt(foo, it notin vowels)) == "sqtls s n wsm mdl"
```

---

### 4. `keepIf`

```nim
proc keepIf*[T](s: var seq[T], pred: proc(x: T): bool {.closure.})
```

**What it does.** The in-place counterpart of `filter`: keeps in `s`
only the elements that pass `pred`, dropping the rest; `s` must be
`var`.

**Implementation notes.** Instead of building a new container, this
uses the "two pointers" idiom: `pos` is the write position for the next
matching element, `i` is the read position. Scanning `s` left to right,
on a predicate match the element is "compacted" into position `pos` (if
it differs from `i` — otherwise no rewrite is needed); where
destructors are supported, `move` is used, otherwise `shallowCopy` (an
older, non-deep-copying mechanism). Finally, `setLen(s, pos)` trims the
tail — the old elements past `pos` are no longer needed. This is the
classic "in-place partitioning" idiom: O(n) time, no extra memory for
the result.

- **Parameters:**
  - `s: var seq[T]` — the sequence, mutated in place.
  - `pred: proc(x: T): bool {.closure.}` — the selection predicate.

**Examples:**

```nim
var floats = @[13.0, 12.5, 5.8, 2.0, 6.1, 9.9, 10.1]
keepIf(floats, proc(x: float): bool = x > 10)
assert floats == @[13.0, 12.5, 10.1]

# Edge case: no element matches — the sequence becomes empty, but not nil.
var numbers = @[1, 2, 3]
keepIf(numbers, proc(x: int): bool = x > 100)
assert numbers == newSeq[int]()

# Practical scenario: dropping stale records without allocating a new seq.
var tasks = @[("A", true), ("B", false), ("C", true)]
keepIf(tasks, proc(x: (string, bool)): bool = x[1])
assert tasks == @[("A", true), ("C", true)]
```

---

### 5. `keepItIf`

```nim
template keepItIf*(varSeq: seq, pred: untyped)
```

**What it does.** The "expression" variant of `keepIf`: `pred` is an
expression using `it` rather than a `proc`. Semantics (in-place
mutation, order preserved) match `keepIf`.

**Implementation notes.** The implementation is the same "two pointers"
trick as `keepIf`, except `it` is declared as a `let` for each position
`i` before evaluating `pred`, letting you write `pred` as an ordinary
boolean expression instead of a separate `proc`.

- **Parameters:**
  - `varSeq: seq` — the mutable sequence.
  - `pred: untyped` — a predicate expression using `it`.

**Examples:**

```nim
var candidates = @["foo", "bar", "baz", "foobar"]
keepItIf(candidates, len(it) == 3 and it[0] == 'b')
assert candidates == @["bar", "baz"]

# Practical scenario: cleaning short/service tokens out of a list.
var tokens = @["a", "hello", "x", "world"]
keepItIf(tokens, len(it) > 1)
assert tokens == @["hello", "world"]
```

---

## IV. Searching, Predicate Checks, and Aggregate Metrics

### 1. `findIt`

```nim
template findIt*(s, predicate: untyped): int
```

**What it does.** Returns the index of the **first** element of `s` for
which `predicate` holds, or `-1` if there's no such element.

**Implementation notes.** A simple linear scan, `for it {.inject.} in
items(s)`, with a counter `i`: as soon as `predicate` holds, `i` is
saved into `res` and the loop breaks — no further elements are checked
(short-circuiting). If the loop finishes without a match, `res` stays
`-1`.

- **Parameters:**
  - `s: untyped` — a container supporting `items`.
  - `predicate: untyped` — an expression using `it`.

**Examples:**

```nim
assert findIt(@[3, 2, 1], it == 2) == 1

# Edge case: no match — result is -1.
assert findIt(@[3, 2, 1], it == 99) == -1

# Practical scenario: index of the first negative reading in a log.
let readings = @[1.5, 2.0, -0.5, 3.3]
assert findIt(readings, it < 0) == 2
```

---

### 2. `any` / `anyIt`

```nim
proc any*[T](s: openArray[T], pred: proc(x: T): bool {.closure.}): bool
template anyIt*(s, pred: untyped): bool
```

**What it does.** Checks whether `s` has at least one element
satisfying the predicate. `any` takes `pred` as a `proc`, `anyIt` — as
an expression using `it`.

**Implementation notes.** `any` is a direct linear scan with an early
exit: as soon as `pred(i)` holds, it immediately `return`s `true`; if
the loop finishes, it returns `false`. `anyIt` is implemented in terms
of the already-described `findIt`: its result is simply `findIt(s,
pred) != -1`, reusing the existing short-circuiting search instead of
duplicating the loop.

- **Parameters:**
  - `s: openArray[T]` (`any`) / `untyped` (`anyIt`) — the container.
  - `pred` / `pred: untyped` — the check predicate.

**Examples:**

```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
assert any(numbers, proc (x: int): bool = x > 8) == true
assert any(numbers, proc (x: int): bool = x > 9) == false

assert anyIt(numbers, it > 8) == true
assert anyIt(numbers, it > 9) == false

# Practical scenario: is there at least one overdue order in the list.
let orders = @[(id: 1, overdue: false), (id: 2, overdue: true)]
assert anyIt(orders, it.overdue) == true
```

---

### 3. `all` / `allIt`

```nim
proc all*[T](s: openArray[T], pred: proc(x: T): bool {.closure.}): bool
template allIt*(s, pred: untyped): bool
```

**What it does.** Checks whether **all** elements of `s` satisfy the
predicate. For an empty container, both return `true` ("the statement
holds for all elements of the empty set" — a standard logical
convention).

**Implementation notes.** `all` is a linear scan with an early exit: as
soon as an element is found for which `pred` is false, it immediately
`return`s `false`; if the whole container is scanned without such an
element, it returns `true`. `allIt` is implemented directly (unlike
`anyIt`, not in terms of `findIt`): `result` is initialized to `true`,
and as soon as `pred` is false for the current `it`, `result` becomes
`false` and the loop breaks.

- **Parameters:**
  - `s: openArray[T]` (`all`) / `untyped` (`allIt`) — the container.
  - `pred` / `pred: untyped` — the check predicate.

**Examples:**

```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
assert all(numbers, proc (x: int): bool = x < 10) == true
assert all(numbers, proc (x: int): bool = x < 9) == false

assert allIt(numbers, it < 10) == true
assert allIt(numbers, it < 9) == false

# Edge case: an empty container — the predicate is vacuously true.
let empty: seq[int] = @[]
assert allIt(empty, it > 1000) == true
```

---

### 4. `count` / `countIt`

```nim
func count*[T](s: openArray[T], x: T): int
template countIt*(s, pred: untyped): int
```

**What it does.** `count` counts how many times a specific value `x`
occurs in `s` (compared via `==`). `countIt` is the more general
variant: it counts how many elements satisfy an arbitrary predicate
expression `pred`.

**Implementation notes.** Both are simple linear scans accumulating a
counter: `count` compares each element to `x` (`unCheckedInc result` on
a match), `countIt` evaluates `pred` for each `it` and increments
`result` when it holds. The full scan always runs (no early exit),
since **all** matches need to be counted, not just the first.

- **Parameters:**
  - `s: openArray[T]` (`count`) / `untyped` (`countIt`, supporting
    `items`) — the container.
  - `x: T` — the value to compare exactly (`count` only).
  - `pred: untyped` — a predicate expression using `it` (`countIt`
    only).

**Examples:**

```nim
let
  a = @[1, 2, 2, 3, 2, 4, 2]
  b = "abracadabra"
assert count(a, 2) == 4
assert count(a, 99) == 0
assert count(b, 'r') == 2

let numbers = @[-3, -2, -1, 0, 1, 2, 3, 4, 5, 6]
assert countIt(numbers, it < 0) == 3

# Practical scenario: countIt works with arbitrary iterators too.
iterator iota(n: int): int =
  for i in 0..<n: yield i
assert countIt(iota(10), it < 2) == 2
```

---

### 5. `min` / `max`

```nim
proc min*[T](x: openArray[T], cmp: proc(a, b: T): int): T
proc max*[T](x: openArray[T], cmp: proc(a, b: T): int): T
```

**What it does.** Return the minimum (or maximum) element of `x`
according to a user-supplied comparator `cmp` — unlike `system.min`/
`system.max`, comparison is fully defined by the caller, so this works
for comparing by an arbitrary criterion (string length, a struct field,
etc.), not just via the built-in `<`.

**Implementation notes.** Both start with `result = x[0]` and scan the
remaining elements (`1..high(x)`), updating `result` whenever `cmp`
says the current element is "better." The convention is Nim's standard
comparator convention: `cmp(a, b) < 0` means "`a` is less than `b`."
Both require a non-empty `x` — on an empty container, accessing `x[0]`
raises `IndexDefect`.

- **Parameters:**
  - `x: openArray[T]` — a non-empty container.
  - `cmp: proc(a, b: T): int` — a comparator in Nim's standard form
    (negative/zero/positive).

**Examples:**

```nim
proc byLen(a, b: string): int = len(a) - len(b)

let words = @["hello", "i", "world"]
assert min(words, byLen) == "i"
assert max(words, byLen) == "hello"

# Practical scenario: the minimal element by a custom tuple field.
let ranges = @[2..4, 1..3, 6..10]
assert min(ranges, proc(a, b: Slice[int]): int = a.a - b.a) == 1..3
```

---

### 6. `minIndex` / `maxIndex`

```nim
func minIndex*[T](s: openArray[T]): int
func minIndex*[T](s: openArray[T], cmp: proc(a, b: T): int): int
func maxIndex*[T](s: openArray[T]): int
func maxIndex*[T](s: openArray[T], cmp: proc(a, b: T): int): int
```

**What it does.** Return not the minimum/maximum element itself, but
its **index** within `s`. The variant without `cmp` requires `T` to
support the `<` operator; the variant with `cmp` works the same way as
`min`/`max` above — comparison is fully user-defined.

**Implementation notes.** All four variants follow the same pattern:
`result` is initialized to index `0`, then a loop over `1..high(s)`
updates `result` whenever the current element turns out to be "better"
than the current candidate (via `<` or via `cmp`). The only difference
between `minIndex` and `maxIndex` is the direction of the comparison
(`s[i] < s[result]` versus `s[i] > s[result]`, or the corresponding
swap of `cmp`'s arguments).

- **Parameters:**
  - `s: openArray[T]` — a non-empty container.
  - `cmp: proc(a, b: T): int` (optional) — a custom comparator.

**Examples:**

```nim
let
  a = @[1, 2, 3, 4]
  b = @[6, 5, 4, 3]
  c = [2, -7, 8, -5]
  d = "ziggy"
assert minIndex(a) == 0
assert minIndex(b) == 3
assert minIndex(c) == 1
assert minIndex(d) == 2

assert maxIndex(a) == 3
assert maxIndex(b) == 0
assert maxIndex(c) == 2
assert maxIndex(d) == 0

# Practical scenario: the index of the shortest/longest word.
let s1 = @["foo", "bar", "hello"]
assert minIndex(s1, proc (a, b: string): int = len(a) - len(b)) == 0
assert maxIndex(s1, proc (a, b: string): int = len(a) - len(b)) == 2
```

---

### 7. `minmax`

```nim
func minmax*[T](x: openArray[T]): (T, T)
func minmax*[T](x: openArray[T], cmp: proc(a, b: T): int): (T, T)
```

**What it does.** Returns a (minimum, maximum) pair in a single pass
over `x` — where separate calls to `min` and `max` would require two
passes. The variant without `cmp` requires `<` for `T`; the variant
with `cmp` uses a custom comparator.

**Implementation notes.** Both `l` (minimum) and `h` (maximum) start at
`x[0]`, then a single loop over the remaining elements updates either
`l` or `h` depending on the comparison result (`elif`, meaning each
element is checked against only one of the two criteria per iteration,
not both). This yields O(n) with a single pass instead of O(2n) for
separate `min`+`max` calls.

- **Parameters:**
  - `x: openArray[T]` — a non-empty container.
  - `cmp: proc(a, b: T): int` (optional) — a custom comparator.

**Examples:**

```nim
let numbers = @[5, 1, 9, -3, 7]
assert minmax(numbers) == (-3, 9)

# Practical scenario: the range of string lengths in a single pass.
let words = @["a", "hello", "sun", "internationalization"]
let (shortest, longest) = minmax(words, proc(a, b: string): int = len(a) - len(b))
assert shortest == "a"
assert longest == "internationalization"
```

---

## V. Folding

### 1. `foldl` (without a starting value)

```nim
template foldl*(sequence, operation: untyped): untyped
```

**What it does.** Folds `sequence` left to right into a single value,
using `operation` — an expression with variables `a` (the accumulated
result) and `b` (the current element). The sequence must be non-empty:
the first element becomes the accumulator's starting value, and the
fold applies to the rest. For non-associative operations (subtraction,
say), the parenthesization is: `(((s[0]) op s[1]) op s[2]) op ...`.

**Implementation notes.** `result` is initialized to the first element
(`s[0]`), then a loop `for i in 1..<len(s)` declares `a` and `b` as
injected `let`s at each step (`a` is the current `result`, `b` is the
next element), evaluates `operation`, and writes the result back into
`result`. In debug builds non-emptiness is checked via `assert`, but in
release builds (without checks) there's no such guard — the caller must
guarantee non-emptiness.

- **Parameters:**
  - `sequence: untyped` — a non-empty container.
  - `operation: untyped` — an expression using variables `a` (the
    accumulator) and `b` (the current element).

**Examples:**

```nim
let
  numbers = @[5, 9, 11]
  addition = foldl(numbers, a + b)
  subtraction = foldl(numbers, a - b)
assert addition == 25, "Addition: (((5)+9)+11)"
assert subtraction == -15, "Subtraction: (((5)-9)-11)"

# Practical scenario: concatenating a list of strings.
let words = @["nim", "is", "cool"]
assert foldl(words, a & b) == "nimiscool"

# Edge case: a single element is returned without applying operation.
assert foldl(@[42], a + b) == 42
```

---

### 2. `foldl` (with a starting value)

```nim
template foldl*(sequence, operation, first): untyped
```

**What it does.** A variant of `foldl` with an explicit starting value
`first`. The key difference from the no-start version: the result type
comes from the type of `first`, not from the sequence's element type —
so this variant lets you fold a sequence into a **different** type
(e.g. folding a `seq[int]` into a `string`). It also works on an empty
sequence — it simply returns `first` unchanged.

**Implementation notes.** `result` is initialized to the value of
`first` (with an explicit `typeof(first)`), then the loop `for x in
items(sequence)` runs `operation` at each step with `a = result` and `b
= x`. Unlike the no-start version, there's no special handling of the
"first element" — the loop simply walks every element of `sequence`
from the very start.

- **Parameters:**
  - `sequence: untyped` — a container (may be empty).
  - `operation: untyped` — an expression using variables `a` (the
    accumulator) and `b` (the current element).
  - `first: untyped` — the accumulator's starting value; sets the
    result type.

**Examples:**

```nim
let
  numbers = @[0, 8, 1, 5]
  digits = foldl(numbers, a & chr(b + ord('0')), "")
assert digits == "0815"

# Edge case: an empty sequence — the result equals first.
let empty: seq[int] = @[]
assert foldl(empty, a + b, 100) == 100

# Practical scenario: summing string lengths (an int accumulator over
# a seq[string]).
let words = @["one", "two", "six"]
assert foldl(words, a + len(b), 0) == 9
```

---

### 3. `foldr`

```nim
template foldr*(sequence, operation: untyped): untyped
```

**What it does.** Folds `sequence` right to left — unlike `foldl`,
where the accumulator builds up from the left, here the fold starts at
the last element and moves toward the first. For non-associative
operations, parentheses nest in the opposite order:
`(s[0] op (s[1] op (... op (s[n-1]))))`. Requires a non-empty sequence
(checked via `assert` in debug builds); with a single element, it's
returned without applying `operation`.

**Implementation notes.** The sequence is materialized into `s` (the
source comments note a possible inefficiency here), then `result` is
initialized to the last element (`s[n - 1]`), and the loop runs **in
reverse** (`countdown(n - 2, 0)`): at each step `a` is the next (more
leftward) element `s[i]`, `b` is the already-accumulated `result` —
meaning `a` and `b` in `foldr` are reversed in meaning compared to
`foldl`.

- **Parameters:**
  - `sequence: untyped` — a non-empty container.
  - `operation: untyped` — an expression using variables `a` (the more
    leftward element) and `b` (the accumulated result to the right).

**Examples:**

```nim
let
  numbers = @[5, 9, 11]
  addition = foldr(numbers, a + b)
  subtraction = foldr(numbers, a - b)
assert addition == 25, "Addition: (5+(9+(11)))"
assert subtraction == 7, "Subtraction: (5-(9-(11)))"

# Practical scenario: right-to-left concatenation gives the same result
# as left-to-right for an associative operation.
let words = @["nim", "is", "cool"]
assert foldr(words, a & b) == "nimiscool"

# Edge case: a single element is returned unchanged.
assert foldr(@[42], a - b) == 42
```

---

## VI. Working with Pairs of Sequences

### 1. `zip`

```nim
proc zip*[S, T](s1: openArray[S], s2: openArray[T]): seq[(S, T)]
```

**What it does.** Builds a sequence of tuples, pairing up the elements
of `s1` and `s2` by index: the `i`-th element of the result is
`(s1[i], s2[i])`. If the input containers differ in length, the extra
elements of the longer one are ignored — the result's length is
`min(len(s1), len(s2))`. The two containers can have different element
types (`S` and `T`).

**Implementation notes.** `zip` is implemented via a private template
`zipImpl`, which substitutes the returned tuple's type at compile time
(a named tuple with fields `a`/`b` for older Nim versions, an unnamed
`(S, T)` for modern ones) — this is how the library preserves backward
compatibility without duplicating the logic. The implementation itself
is straightforward: `m = min(len(s1), len(s2))`, the result is allocated
up front for `m` elements, and a loop over `0 ..< m` collects the
pairs.

- **Parameters:**
  - `s1: openArray[S]` — the first container.
  - `s2: openArray[T]` — the second container (may have a different
    element type).

**Examples:**

```nim
let
  short = @[1, 2, 3]
  long = @[6, 5, 4, 3, 2, 1]
  words = @["one", "two", "three"]
  zip1 = zip(short, long)
  zip2 = zip(short, words)
assert zip1 == @[(1, 6), (2, 5), (3, 4)]
assert zip2 == @[(1, "one"), (2, "two"), (3, "three")]
assert zip1[2][0] == 3
assert zip2[1][1] == "two"

# Edge case: differing lengths — the longer container's tail is
# dropped.
assert len(zip(short, long)) == 3

# Practical scenario: pairing names and scores together for a report.
let
  names = @["Ann", "Bob", "Vera"]
  scores = @[91, 88, 95]
  paired = zip(names, scores)
assert paired[1] == ("Bob", 88)
```

---

### 2. `unzip`

```nim
proc unzip*[S, T](s: openArray[(S, T)]): (seq[S], seq[T])
```

**What it does.** The reverse of `zip`: takes a sequence of 2-element
tuples and splits it into a pair of separate sequences — one made of
the first fields of the tuples, the other of the second fields. For any
`s1`, `s2`, `unzip(zip(s1, s2)) == (s1, s2)` holds (accounting for
truncation at the shorter length, as with `zip`).

**Implementation notes.** Both output sequences are allocated up front
at the exact size (`newSeq[S](len(s))`, `newSeq[T](len(s))`), then a
single pass over `s` splits `s[i][0]` into the first, `s[i][1]` into
the second. O(n) complexity, no intermediate structures.

- **Parameters:**
  - `s: openArray[(S, T)]` — a sequence of pairs (not modified).

**Examples:**

```nim
let
  zipped = @[(1, 'a'), (2, 'b'), (3, 'c')]
  unzipped1 = @[1, 2, 3]
  unzipped2 = @['a', 'b', 'c']
assert unzip(zipped) == (unzipped1, unzipped2)
assert unzip(zip(unzipped1, unzipped2)) == (unzipped1, unzipped2)

# Practical scenario: splitting a list of (key, value) pairs into two
# parallel lists for separate processing.
let settings = @[("host", "localhost"), ("port", "8080")]
let (keys, values) = unzip(settings)
assert keys == @["host", "port"]
assert values == @["localhost", "8080"]
```

---

## VII. In-Place Composition Changes

### 1. `addUnique`

```nim
func addUnique*[T](s: var seq[T], x: sink T)
```

**What it does.** Appends `x` to the end of `s`, but only if that value
isn't already present (checked via `==`). If the element is already
there, the call does nothing.

**Implementation notes.** First, a linear scan `for i in 0..high(s)`
checks whether `x` is already present in `s`; on a match, it
immediately `return`s without appending. If no match is found, the
element is added via `add`, using `ensureMove(x)` wherever it's
declared (`when declared(ensureMove)`) — this lets `x` be moved into
the sequence instead of copied, since the parameter is declared `sink
T` (ownership may be transferred by the caller). The duplicate check is
O(n), so repeated calls to `addUnique` on a large `s` add up to
quadratic total complexity.

- **Parameters:**
  - `s: var seq[T]` — the sequence, mutated in place.
  - `x: sink T` — the value to add; the parameter may be moved into
    `s` instead of copied.

**Examples:**

```nim
var a = @[1, 2, 3]
addUnique(a, 4)
addUnique(a, 4)
assert a == @[1, 2, 3, 4]

# Edge case: adding an already-present element leaves order and
# contents unchanged — no duplicate is created.
addUnique(a, 2)
assert a == @[1, 2, 3, 4]

# Practical scenario: building a set of unique tags from an input
# stream.
var tags: seq[string] = @[]
for tag in @["python", "nim", "python", "rust"]:
  addUnique(tags, tag)
assert tags == @["python", "nim", "rust"]
```

---

### 2. `deduplicate`

```nim
func deduplicate*[T](s: openArray[T], isSorted: bool = false): seq[T]
```

**What it does.** Returns a new sequence with no repeated elements,
preserving the order of each value's first appearance. The `isSorted`
parameter is a performance hint: if `s` is known to be sorted, a faster
algorithm can be used, comparing only adjacent elements.

**Implementation notes.** With `isSorted = true` (the fast path), it's
enough to compare each element to the previous one (`prev`) — since the
sequence is sorted, all occurrences of a given value are adjacent, so
"differs from the immediately preceding element" is equivalent to "has
been seen before." This gives O(n). With `isSorted = false` (the
default), each element is checked via `contains` against the already
accumulated result — O(n) per insertion, i.e. O(n²) total in the worst
case, but it doesn't require pre-sorted data and doesn't reorder
elements.

- **Parameters:**
  - `s: openArray[T]` — the source container (not modified).
  - `isSorted: bool` (default `false`) — turns on the fast algorithm
    for already-sorted data.

**Examples:**

```nim
let
  dup1 = @[1, 1, 3, 4, 2, 2, 8, 1, 4]
  dup2 = @["a", "a", "c", "d", "d"]
  unique1 = deduplicate(dup1)
  unique2 = deduplicate(dup2, isSorted = true)
assert unique1 == @[1, 3, 4, 2, 8]
assert unique2 == @["a", "c", "d"]

# Edge case: an empty container — an empty result, no errors.
assert deduplicate(newSeq[int]()) == newSeq[int]()

# Practical scenario: unique IP addresses from a request log,
# preserving the order of first contact.
let log = @["10.0.0.1", "10.0.0.2", "10.0.0.1", "10.0.0.3"]
assert deduplicate(log) == @["10.0.0.1", "10.0.0.2", "10.0.0.3"]
```

---

### 3. `delete`

```nim
func delete*[T](s: var seq[T]; slice: Slice[int])
```

**What it does.** Removes from `s` the elements whose indices fall
within `slice` (inclusive on both ends), shifting the remaining
elements left. If `slice` extends past `s`'s bounds, `IndexDefect` is
raised (when `boundChecks` are enabled). An empty slice (`i..<i`) is a
valid, no-op call.

The deprecated variant `delete(s, first, last: Natural)` (with two
separate parameters instead of a `Slice[int]`) is marked
`{.deprecated.}` in favor of the slice form — new code should use
`delete(s, first..last)`.

**Implementation notes.** The removal happens without allocating new
memory: the elements after the removed range (`s[slice.b + 1 .. ^1]`)
are shifted into the freed slot via `move` (where destructors are
supported) or `shallowCopy` (without them), after which `setLen` trims
the sequence's tail. Compilation for JS is handled separately — instead
of a manual shift loop, it uses the native array method `splice`, which
is more efficient in that environment. Complexity is O(n) in the worst
case (the entire tail after the removed range shifts).

- **Parameters:**
  - `s: var seq[T]` — the sequence, mutated in place.
  - `slice: Slice[int]` — the inclusive index range to remove.

**Examples:**

```nim
var a = @[10, 11, 12, 13, 14]
doAssertRaises(IndexDefect): delete(a, 4..5)
assert a == @[10, 11, 12, 13, 14]

delete(a, 4..4)
assert a == @[10, 11, 12, 13]
delete(a, 1..2)
assert a == @[10, 13]

# Edge case: an empty slice doesn't change the sequence.
delete(a, 1..<1)
assert a == @[10, 13]

# Practical scenario: removing a range of stale entries from a buffer
# while preserving order of the remaining ones.
var buffer = @["a", "b", "c", "d", "e"]
delete(buffer, 1..2)
assert buffer == @["a", "d", "e"]
```

---

### 4. `insert`

```nim
func insert*[T](dest: var seq[T], src: openArray[T], pos = 0)
```

**What it does.** Inserts the elements of `src` into `dest`, starting
at position `pos`, shifting the existing elements `dest[pos..]` right.
`dest` and `src` must have the same element type; `dest` is mutated in
place, `src` is not.

**Implementation notes.** First, `dest` is grown to its final length
(`setLen(dest, i + 1)`, where `i` is the new last index), then the
existing tail `dest[pos..]` is shifted **right to left** (a `while j
>= pos` loop, decreasing indices) — this order is required to avoid
overwriting elements that haven't been copied yet when the read and
write ranges overlap. After shifting the tail, the freed range starting
at `pos` is filled in order with the elements of `src`. Complexity is
O(len(dest) + len(src)), with no extra container.

- **Parameters:**
  - `dest: var seq[T]` — the sequence, mutated in place (receiving the
    insertion).
  - `src: openArray[T]` — the elements to insert (not modified).
  - `pos: int` (default `0`) — the position at which `src`'s elements
    are inserted.

**Examples:**

```nim
var dest = @[1, 1, 1, 1, 1, 1, 1, 1]
let
  src = @[2, 2, 2, 2, 2, 2]
  outcome = @[1, 1, 1, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1]
insert(dest, src, 3)
assert dest == outcome

# Edge case: inserting at the start (pos defaults to 0).
var d2 = @[3, 4]
insert(d2, @[1, 2])
assert d2 == @[1, 2, 3, 4]

# Practical scenario: inserting a block of new items in the middle of
# a playlist buffer.
var playlist = @["track1", "track2", "track5"]
insert(playlist, @["track3", "track4"], 2)
assert playlist == @["track1", "track2", "track3", "track4", "track5"]
```

---

## VIII. Practical Recipes

### 1. Top-N Leaders by Criterion

A combination of a working copy and repeated `maxIndex` + `delete` —
useful when a full sort would be wasteful and n is much smaller than
the input.

```nim
proc topN(scores: openArray[int], n: Positive): seq[int] =
  ## Returns the n largest values without an external sort — suitable
  ## when n << len(scores).
  var
    working = toSeq(scores)
    leaders = newSeq[int]()

  for i in 0 ..< min(n, len(working)):
    let idx = maxIndex(working)
    add(leaders, working[idx])
    delete(working, idx..idx)

  leaders

let scores = @[42, 17, 88, 3, 56, 91, 10]
assert topN(scores, 3) == @[91, 88, 56]
```

---

### 2. Deduplication Preserving First Occurrence

`deduplicate` already solves this directly, but sometimes you also need
to count how many times each unique element occurred — that's where
the combination `deduplicate` + `count` comes in.

```nim
proc frequencies(s: openArray[string]): seq[(string, int)] =
  let uniqueValues = deduplicate(s)
  result = newSeq[(string, int)](len(uniqueValues))
  for i, value in uniqueValues:
    result[i] = (value, count(s, value))

let words = @["nim", "go", "nim", "rust", "go", "nim"]
assert frequencies(words) == @[("nim", 3), ("go", 2), ("rust", 1)]
```

---

### 3. Round-Robin Task Distribution Across Workers

A combination of `toSeq`, `distribute`, and `mapIt` — a typical
pattern when preparing work for a thread pool: turn a range into a
sequence, split it into nearly equal chunks, and wrap each chunk in a
convenient structure.

```nim
type WorkPacket = object
  workerIndex: int
  items: seq[int]

proc buildPackets(total: int, workers: Positive): seq[WorkPacket] =
  let tasks = toSeq(0 ..< total)
  let chunks = distribute(tasks, workers)
  result = newSeq[WorkPacket](len(chunks))
  for i, chunk in chunks:
    result[i] = WorkPacket(workerIndex: i, items: chunk)

let packets = buildPackets(10, 3)
assert len(packets) == 3
assert packets[0].items == @[0, 1, 2, 3]
```

---

### 4. Merging Two Parallel Lists into a Report

A combination of `zip` + `filterIt` + `mapIt`: pair up two parallel
lists, filter by a condition, and format the result as report text.

```nim
import std/strformat

proc failureReport(names: openArray[string],
                    results: openArray[bool]): seq[string] =
  let paired = zip(names, results)
  let failures = filterIt(paired, not it[1])
  mapIt(failures, fmt"{it[0]}: failed")

let
  names = @["test_a", "test_b", "test_c"]
  outcomes = @[true, false, false]
assert failureReport(names, outcomes) == @["test_b: failed", "test_c: failed"]
```

---

### 5. Rolling Statistics over a Numeric Log

A combination of `minmax`, `foldl`, and `allIt` — a quick way to check
the range and "health" of a numeric stream with a minimum of passes.

```nim
type LogStats = object
  minimum, maximum, total: float
  withinRange: bool

proc gatherStats(values: openArray[float],
                  lower, upper: float): LogStats =
  let (minVal, maxVal) = minmax(values)
  let total = foldl(values, a + b, 0.0)
  result = LogStats(
    minimum: minVal,
    maximum: maxVal,
    total: total,
    withinRange: allIt(values, it >= lower and it <= upper))

let
  readings = @[18.2, 19.5, 21.0, 17.8, 20.3]
  stats = gatherStats(readings, 15.0, 25.0)
assert stats.minimum == 17.8
assert stats.withinRange == true
```

---

## IX. Quick Reference Table

| Task | Mutates the argument | Returns a new seq/value |
|---|---|---|
| Transform every element | — | `map`, `mapIt` |
| Transform every element in place | `apply`, `applyIt` | — |
| Keep elements matching a condition | — | `filter` (proc/iterator), `filterIt` |
| Keep elements matching a condition in place | `keepIf`, `keepItIf` | — |
| Find the index of the first match | — | `findIt` |
| Check "at least one" / "all" | — | `any`/`anyIt`, `all`/`allIt` |
| Count occurrences | — | `count`, `countIt` |
| Find min/max by a comparator | — | `min`, `max` |
| Find the index of the min/max | — | `minIndex`, `maxIndex` |
| Find min and max in a single pass | — | `minmax` |
| Fold a sequence into a value | — | `foldl`, `foldr` |
| Pair up two sequences | — | `zip` |
| Split a sequence of pairs | — | `unzip` |
| Turn an iterable into a `seq` | — | `toSeq` |
| Replicate a single value | — | `repeat` |
| Repeat a whole sequence | — | `cycle` |
| Build a `seq` by evaluating an expression per index | — | `newSeqWith` |
| Concatenate several sequences | — | `concat` |
| Split a sequence into N parts | — | `distribute` |
| Add an element without duplicates | `addUnique` | — |
| Remove duplicates | — | `deduplicate` |
| Remove a range of elements | `delete` | — |
| Insert elements in the middle | `insert` | — |
| Apply a transform to AST literals | — | `mapLiterals` (macro) |

---

## X. Summary: Which Procedure to Choose

- Need a new sequence with transformed elements, leaving the original
  untouched → `map` (with a `proc`) or `mapIt` (with an `it`
  expression).
- Need to transform elements **in place** → `apply` (three forms based
  on `op`'s signature), or, more concisely, `applyIt`.
- Need to keep only matching elements, returning a new container →
  `filter` (the `iterator` version for consuming in a loop, the `proc`
  version for a materialized `seq`) or `filterIt`.
- Need to filter **in place**, without allocating new memory →
  `keepIf` or `keepItIf`.
- Need only the index of the first match → `findIt`.
- Need to know whether a condition holds for at least one / for all
  elements → `anyIt`/`any` and `allIt`/`all` respectively.
- Need to count occurrences of a specific value → `count`; of an
  arbitrary condition → `countIt`.
- Need a minimum/maximum by a non-standard criterion (not via `<`) →
  `min`/`max` with a comparator; need the index specifically →
  `minIndex`/`maxIndex`; need both at once in a single pass → `minmax`.
- Need to fold a sequence into a single value, left to right → `foldl`
  (no starting value, if the container is guaranteed non-empty; with a
  starting value, if you need a different accumulator type or the
  container may be empty); right to left → `foldr`.
- Need to pair two lists into a list of tuples → `zip`; split a list
  of pairs back into two lists → `unzip`.
- Need to turn a range/set/iterator into a `seq` → `toSeq`.
- Need to create a sequence of N identical copies → `repeat`; of N
  independently computed values (e.g. random ones) → `newSeqWith`;
  repeat the content of an existing sequence N times → `cycle`.
- Need to concatenate several sequences into one → `concat`; the
  reverse — split one into N parts → `distribute`.
- Need to add an element only if it's not already present →
  `addUnique`; remove all repeats → `deduplicate` (with `isSorted =
  true` if the data is already sorted — it's faster).
- Need to remove a range of elements in place → `delete`; insert a
  block of elements in the middle → `insert`.
- Need to apply a transform to literals inside an AST constructor at
  compile time → `mapLiterals`.
