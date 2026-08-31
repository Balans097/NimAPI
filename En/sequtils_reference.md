# sequtils — module reference

> **Import:** `import std/sequtils`
> **Scope:** functional operations over `openArray`-based types (`seq`, `array`, `string`) — creation, transformation, filtering, searching, folding, and combining sequences.

The module is not limited to `seq` — despite the name, every procedure applies equally to `array` and `string`, since both fall under `openArray`. The library is consistently built around several "mutating / non-mutating" pairs (for example, `filter`/`keepIf`, `map`/`apply`): the non-mutating variant returns a new sequence without touching the original, while the mutating variant works "in place" and requires a `var` parameter. Many functions have counterparts with the `It` suffix (`mapIt`, `filterIt`, `anyIt`...) — these are templates that, instead of an explicit `proc` argument, use an implicitly injected `it` variable, removing the need to write out `proc(x: T): S = ...` in full each time.

---

## Table of contents

I. [Creating and generating sequences](#creating-and-generating-sequences)
  1. [`concat`](#concat)
  2. [`cycle`](#cycle)
  3. [`repeat`](#repeat)
  4. [`newSeqWith`](#newseqwith)
  5. [`toSeq`](#toseq)

II. [Transforming elements](#transforming-elements)
  1. [`map`](#map)
  2. [`mapIt`](#mapit)
  3. [`apply` (variant A, in-place mutating function with no return value)](#apply-variant-a)
  4. [`apply` (variant B, function T → T)](#apply-variant-b)
  5. [`apply` (variant C, side effect only)](#apply-variant-c)
  6. [`applyIt`](#applyit)
  7. [`mapLiterals`](#mapliterals)

III. [Filtering](#filtering)
  1. [`filter` (proc)](#filter-proc)
  2. [`filter` (iterator)](#filter-iterator)
  3. [`filterIt`](#filterit)
  4. [`keepIf`](#keepif)
  5. [`keepItIf`](#keepitif)

IV. [Searching and condition checks](#searching-and-condition-checks)
  1. [`findIt`](#findit)
  2. [`all` / `allIt`](#all--allit)
  3. [`any` / `anyIt`](#any--anyit)
  4. [`countIt`](#countit)
  5. [`count`](#count)

V. [Aggregation and folding](#aggregation-and-folding)
  1. [`foldl` (without an initial value)](#foldl-without-an-initial-value)
  2. [`foldl` (with an initial value)](#foldl-with-an-initial-value)
  3. [`foldr`](#foldr)
  4. [`min` / `max` (with a comparator)](#min--max-with-a-comparator)
  5. [`minIndex` / `maxIndex`](#minindex--maxindex)
  6. [`minmax`](#minmax)

VI. [In-place sequence modification](#in-place-sequence-modification)
  1. [`addUnique`](#addunique)
  2. [`deduplicate`](#deduplicate)
  3. [`delete`](#delete)
  4. [`insert`](#insert)

VII. [Combining and splitting sequences](#combining-and-splitting-sequences)
  1. [`zip`](#zip)
  2. [`unzip`](#unzip)
  3. [`distribute`](#distribute)

VIII. [Iterators for functional style](#iterators-for-functional-style)
  1. [`items` (for a closure iterator)](#items-for-a-closure-iterator)

IX. [Practical recipes](#practical-recipes)
  1. [Top-N leaders by value](#top-n-leaders-by-value)
  2. [Removing duplicates while preserving order](#removing-duplicates-while-preserving-order)
  3. [Round-robin distribution of tasks across workers](#round-robin-distribution-of-tasks-across-workers)
  4. [Word frequency analysis](#word-frequency-analysis)
  5. [Combining two parallel lists into a report](#combining-two-parallel-lists-into-a-report)
  6. [Rolling statistics over a numeric log](#rolling-statistics-over-a-numeric-log)

X. [Quick reference table](#quick-reference-table)

XI. [Summary: which procedure to choose](#summary-which-procedure-to-choose)

---

## Creating and generating sequences

### `concat`

```nim
func concat*[T](seqs: varargs[seq[T]]): seq[T]
```

**What it does.** Joins several sequences of the same type into one new sequence. All elements of all arguments are copied in order into the resulting `seq`, preserving both the order of the arguments and the order of elements within each. Passing one empty sequence, or none at all, yields an empty `seq`.

**Implementation notes.** Before copying, the function sums the lengths of all input sequences in a single pass (`inc(L, len(seqitm))`) and allocates memory for the result of the exact needed size once (`newSeq(result, L)`). This avoids the repeated reallocations that would occur with successive `add` calls: the complexity stays linear O(n) in the total number of elements, but with a single allocation instead of potentially O(log n) reallocations of a growing dynamic array.

- **Parameters:**
  - `seqs: varargs[seq[T]]` — any number of sequences of the same type `T`, joined in the order listed.

```nim
let
  s1 = @[1, 2, 3]
  s2 = @[4, 5]
  s3: seq[int] = @[]
  total = concat(s1, s2, s3)
echo total  # prints @[1, 2, 3, 4, 5]

let empty = concat[int]()  # edge case: no arguments
echo empty  # prints @[]
```

---

### `cycle`

```nim
func cycle*[T](s: openArray[T], n: Natural): seq[T]
```

**What it does.** Returns a new sequence in which the contents of `s` are repeated whole `n` times in a row (all of `s` — then all of `s` again, `n` times). At `n = 0` the result is an empty `seq`. For an empty `s` and any `n`, the result is also empty.

- **Parameters:**
  - `s: openArray[T]` — the immutable source sequence/array/string, repeated as a whole.
  - `n: Natural` — number of repetitions (0 or more).

```nim
let s = @[1, 2, 3]
echo cycle(s, 3)  # prints @[1, 2, 3, 1, 2, 3, 1, 2, 3]
echo cycle(s, 0)  # edge case: prints @[]

let empty: seq[int] = @[]
echo cycle(empty, 5)  # edge case: prints @[]
```

---

### `repeat`

```nim
proc repeat*[T](x: T, n: Natural): seq[T]
```

**What it does.** Returns a new sequence made of `n` copies of the same value `x` (unlike `cycle`, which repeats an entire sequence, `repeat` repeats a single element). At `n = 0` — an empty `seq`.

- **Parameters:**
  - `x: T` — the value to be duplicated.
  - `n: Natural` — how many copies to produce (0 or more).

```nim
echo repeat(5, 3)     # prints @[5, 5, 5]
echo repeat("ok", 0)  # edge case: prints @[]

# Practical scenario: a fixed-length separator string
let separator = repeat('-', 20)
echo len(separator)  # prints 20
```

---

### `newSeqWith`

```nim
template newSeqWith*(len: int, init: untyped): untyped
```

**What it does.** Creates a new sequence of length `len`, where each element is initialized by a separate evaluation of the expression `init` — that is, `init` is evaluated afresh for *each* position. This is what distinguishes `newSeqWith` from `repeat`: if `init` is, say, `newSeq[bool](3)`, then each nested sequence is its own independent object, not n references to the same one. That's exactly why `newSeqWith` is the standard way to build a "2D" sequence (a seq of seqs).

- **Parameters:**
  - `len: int` — the required length of the resulting sequence.
  - `init: untyped` — an expression evaluated afresh for each element (may use `rand`, constructors for nested `seq`s, and so on).

```nim
# Creates a seq of 5 bool seqs, each of length 3 — independent nested seqs
var seq2D = newSeqWith(5, newSeq[bool](3))
echo len(seq2D)       # prints 5
echo len(seq2D[0])    # prints 3

seq2D[0][0] = true
echo seq2D[1][0]     # prints false — independent initialization, not a shared reference
```

---

### `toSeq`

```nim
template toSeq*(iter: untyped): untyped
```

**What it does.** Turns an arbitrary iterable object (a range `1..5`, a `set`, an iterator, an `openArray`) into a concrete `seq`. This is the most common first step in functional-style chains: a range on its own does not support `mapIt`/`filterIt`, while a `seq` obtained via `toSeq` does.

**Implementation notes.** `toSeq` is not a single implementation but a dispatcher of three branches, chosen via `when compiles(...)` right at compile time, with no involvement from the calling code: the first branch handles ordinary typed containers (ranges, arrays, sets) — if the argument has a known `len`, the result is allocated at the exact needed size in a single O(n) pass; the second branch handles named closure iterators, whose `len` is not known in advance, so accumulation happens through incremental `add`, which may require several reallocations; the third branch is a fallback path for untyped inline iterators such as `toSeq(myInlineIterator(3))` that don't fit the first two.

- **Parameters:**
  - `iter: untyped` — any expression that can be iterated in a `for` loop (a range, a set, an `openArray`, a named or inline iterator).

```nim
let
  myRange = 1..5
  mySet: set[int8] = {5'i8, 3, 1}
echo toSeq(myRange)  # prints @[1, 2, 3, 4, 5]
echo toSeq(mySet)    # prints @[1, 3, 5] — a set yields sorted order

# Edge case: an empty range yields an empty sequence
echo toSeq(1..0)  # prints @[]

# Practical scenario: an inline iterator turned into a seq for a further mapIt/filterIt chain
iterator evens(n: int): int =
  for i in 0..<n:
    if i mod 2 == 0:
      yield i
echo toSeq(evens(10))  # prints @[0, 2, 4, 6, 8]
```

---

## Transforming elements

### `map`

```nim
proc map*[T, S](s: openArray[T], op: proc (x: T): S {.closure.}): seq[S]
```

**What it does.** Returns a new sequence of the same length as `s`, where each element is the result of applying `op` to the corresponding element of the source sequence. The source `s` is not changed. The result type `S` may differ from the source element type `T` — this is the standard way to convert one type to another element-wise.

- **Parameters:**
  - `s: openArray[T]` — the immutable source sequence.
  - `op: proc (x: T): S` — the function converting one element of type `T` into an element of type `S`.

```nim
let
  a = @[1, 2, 3, 4]
  b = map(a, proc(x: int): string = $x)
echo b  # prints @["1", "2", "3", "4"]
```

---

### `mapIt`

```nim
template mapIt*(s: typed, op: untyped): untyped
```

**What it does.** The same thing as `map`, but instead of an explicit `proc(x: T): S = ...`, it uses an implicitly injected `it` variable referring to the current element. This removes the need to write out the converter function's signature every time. Like `map`, it does not change the source sequence.

- **Parameters:**
  - `s: typed` — the source sequence/array/string.
  - `op: untyped` — an expression using the `it` variable (the current element).

```nim
let
  nums = @[1, 2, 3, 4]
  strings = mapIt(nums, $(4 * it))
echo strings  # prints @["4", "8", "12", "16"]
```

---

### `apply` (variant A)

```nim
proc apply*[T](s: var openArray[T], op: proc (x: var T) {.closure.})
```

**What it does.** Changes `s` in place: for each element it calls `op`, passing the element by `var` reference, so `op` mutates the element directly inside its body (returning nothing). Requires `s` to be declared as `var`.

- **Parameters:**
  - `s: var openArray[T]` — the mutable sequence whose elements will be replaced in place.
  - `op: proc (x: var T)` — a procedure that mutates the passed element `x` directly.

```nim
var a = @["1", "2", "3", "4"]
apply(a, proc(x: var string) = x &= "42")
echo a  # prints @["142", "242", "342", "442"]
```

---

### `apply` (variant B)

```nim
proc apply*[T](s: var openArray[T], op: proc (x: T): T {.closure.})
```

**What it does.** Also changes `s` in place, but unlike variant A, `op` does not mutate the argument directly — instead it returns a new value of the same type `T`, which replaces the element `s[i]`. Requires the input and output of `op` to be of the same type, otherwise the assignment `s[i] = op(s[i])` won't compile.

- **Parameters:**
  - `s: var openArray[T]` — the mutable sequence.
  - `op: proc (x: T): T` — a function taking an element and returning its new value of the same type.

```nim
var a = @["1", "2", "3", "4"]
apply(a, proc(x: string): string = x & "42")
echo a  # prints @["142", "242", "342", "442"]
```

---

### `apply` (variant C)

```nim
proc apply*[T](s: openArray[T], op: proc (x: T) {.closure.})
```

**What it does.** Unlike variants A and B, this mutates and returns nothing at all — this overload exists purely for the side effect of `op` (for example, accumulating into an outer variable, writing to a file, logging). That's exactly why `s` here doesn't require `var`.

- **Parameters:**
  - `s: openArray[T]` — the immutable sequence being walked by `op`.
  - `op: proc (x: T)` — a procedure with no return value, called for its side effect.

```nim
var message: string
apply([0, 1, 2, 3, 4], proc(item: int) = addInt(message, item))
echo message  # prints "01234"
```

---

### `applyIt`

```nim
template applyIt*(varSeq, op: untyped)
```

**What it does.** A convenient wrapper around the mutating `apply` (variant B) that, like `mapIt`, uses an implicit `it` variable instead of an explicit function. The expression `op` must return a value of the same type as the elements of `varSeq`, since the result of `op` is assigned back into `varSeq[i]`.

- **Parameters:**
  - `varSeq: untyped` — the mutable sequence (must be `var`).
  - `op: untyped` — an expression with the `it` variable, returning the element's new value.

```nim
var nums = @[1, 2, 3, 4]
applyIt(nums, it * 3)
echo nums  # prints @[3, 6, 9, 12]
```

---

### `mapLiterals`

```nim
macro mapLiterals*(constructor, op: untyped; nested = true): untyped
```

**What it does.** Applies a function/operator `op` not to the elements of a sequence at run time, but to every "atomic" literal (a number, a string) directly in the AST of a constructor (an array, a tuple) at compile time. Useful for converting the types of literals inside nested constructs without manually wrapping every number.

- **Parameters:**
  - `constructor: untyped` — an AST constructor (for example, an array or tuple literal) inside which atomic literals are searched for.
  - `op: untyped` — a function or operator applied to every literal found (for example, `int`, `` `$` ``).
  - `nested: bool` — if `true` (the default), literals are replaced at every level of nesting; if `false` — only at the top level.

```nim
let x = mapLiterals([0.1, 1.2, 2.3, 3.4], int)
echo x  # prints [0, 1, 2, 3] — literals converted to int

let a = mapLiterals((1.2, (2.3, 3.4), 4.8), int)
let b = mapLiterals((1.2, (2.3, 3.4), 4.8), int, nested = false)
echo a  # prints (1, (2, 3), 4) — literals replaced at every level
echo b  # prints (1, (2.3, 3.4), 4) — only the top-level literal replaced
```

---

## Filtering

### `filter` (proc)

```nim
proc filter*[T](s: openArray[T], pred: proc(x: T): bool {.closure.}): seq[T]
```

**What it does.** Returns a new sequence containing only the elements of `s` for which `pred` returned `true`. Order is preserved, the source `s` is not changed. If no element qualifies, the result is an empty `seq`.

- **Parameters:**
  - `s: openArray[T]` — the immutable source sequence.
  - `pred: proc(x: T): bool` — the predicate deciding whether to keep an element.

```nim
let
  colors = @["red", "yellow", "black"]
  short = filter(colors, proc(x: string): bool = len(x) < 6)
  withY = filter(colors, proc(x: string): bool = contains(x, 'y'))
echo short  # prints @["red", "black"]
echo withY  # prints @["yellow"]
```

---

### `filter` (iterator)

```nim
iterator filter*[T](s: openArray[T], pred: proc(x: T): bool {.closure.}): T
```

**What it does.** Functionally the same as the `filter` proc — it walks `s` and yields, one at a time, only the elements for which `pred` is true — but it does not build an intermediate `seq`; instead it generates values "lazily", one at a time, inside a `for` loop. The mutating/non-mutating distinction doesn't apply here — this is a choice between "materialize the whole seq at once" and "yield one element per loop iteration".

- **Parameters:**
  - `s: openArray[T]` — the immutable source sequence.
  - `pred: proc(x: T): bool` — the selection predicate.

```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
var evens = newSeq[int]()
for n in filter(numbers, proc (x: int): bool = x mod 2 == 0):
  add(evens, n)
echo evens  # prints @[4, 8, 4]
```

---

### `filterIt`

```nim
template filterIt*(s, pred: untyped): untyped
```

**What it does.** A version of `filter` with an implicit `it` variable instead of an explicit predicate function. Returns a new sequence, the source `s` is not changed.

- **Parameters:**
  - `s: untyped` — the source sequence/array/string.
  - `pred: untyped` — a boolean expression using the `it` variable.

```nim
let
  temperatures = @[-272.15, -2.0, 24.5, 44.31, 99.9, -113.44]
  acceptable = filterIt(temperatures, it < 50 and it > -10)
echo acceptable  # prints @[-2.0, 24.5, 44.31]

let empty: seq[int] = @[]
echo filterIt(empty, it > 0)  # edge case: prints @[]
```

---

### `keepIf`

```nim
proc keepIf*[T](s: var seq[T], pred: proc(x: T): bool {.closure.})
```

**What it does.** The mutating counterpart of `filter`: instead of creating a new sequence, it rebuilds `s` in place, keeping only the elements that satisfy `pred`, and truncating the length to the new element count.

**Implementation notes.** Uses the classic "two pointers moving over one array" pattern: `pos` is where to write the next matching element, `i` is what's currently being read. When an element matches and `pos != i`, it is moved back to position `pos` (via `move` when `gcDestructors` is enabled, otherwise `shallowCopy`); after the pass, `setLen(s, pos)` is called, dropping the tail. Complexity is O(n), with no extra memory for a copy of the sequence.

- **Parameters:**
  - `s: var seq[T]` — the mutable sequence (rebuilt in place).
  - `pred: proc(x: T): bool` — the selection predicate.

```nim
var floats = @[13.0, 12.5, 5.8, 2.0, 6.1, 9.9, 10.1]
keepIf(floats, proc(x: float): bool = x > 10)
echo floats  # prints @[13.0, 12.5, 10.1]
```

---

### `keepItIf`

```nim
template keepItIf*(varSeq: seq, pred: untyped)
```

**What it does.** A version of `keepIf` with an implicit `it` variable. Changes `varSeq` in place using the same "two pointers" approach as `keepIf`.

- **Parameters:**
  - `varSeq: seq` — the mutable sequence (must be `var`).
  - `pred: untyped` — a boolean expression using the `it` variable.

```nim
var candidates = @["foo", "bar", "baz", "foobar"]
keepItIf(candidates, len(it) == 3 and it[0] == 'b')
echo candidates  # prints @["bar", "baz"]
```

---

## Searching and condition checks

### `findIt`

```nim
template findIt*(s, predicate: untyped): int
```

**What it does.** Returns the index of the first element of `s` satisfying `predicate`, or `-1` if no such element exists. Unlike `find` (from another module), the predicate here is an expression using the `it` variable, not a value to compare against.

- **Parameters:**
  - `s: untyped` — the source sequence/array/string.
  - `predicate: untyped` — a boolean expression using the `it` variable.

```nim
echo findIt(@[3, 2, 1], it == 2)   # prints 1
echo findIt(@[3, 2, 1], it == 99)  # edge case, not found: prints -1
```

---

### `all` / `allIt`

```nim
proc all*[T](s: openArray[T], pred: proc(x: T): bool {.closure.}): bool
template allIt*(s, pred: untyped): bool
```

**What it does.** Checks that *every single* element of `s` satisfies the predicate. On an empty sequence it returns `true` (a degenerate case: the "for all" condition is vacuously true when there are no elements). `allIt` is the same algorithm but with an implicit `it` variable instead of an explicit function.

- **Parameters:**
  - `s: openArray[T]` (for `all`) / `s: untyped` (for `allIt`) — the sequence being checked.
  - `pred` — the predicate (an explicit function for `all`, an expression with `it` for `allIt`).

```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
echo all(numbers, proc (x: int): bool = x < 10)  # prints true
echo allIt(numbers, it < 9)                       # prints false

let empty: seq[int] = @[]
echo allIt(empty, it > 1000)  # edge case: prints true
```

---

### `any` / `anyIt`

```nim
proc any*[T](s: openArray[T], pred: proc(x: T): bool {.closure.}): bool
template anyIt*(s, pred: untyped): bool
```

**What it does.** Checks whether *at least one* element of `s` satisfies the predicate. On an empty sequence it's always `false`. `anyIt` is implemented via `findIt` — as soon as a matching index is found (not `-1`), the result is `true`.

- **Parameters:**
  - `s` — the sequence being checked.
  - `pred` — the predicate (an explicit function for `any`, an expression with `it` for `anyIt`).

```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
echo any(numbers, proc (x: int): bool = x > 8)  # prints true
echo anyIt(numbers, it > 9)                      # prints false

let empty: seq[int] = @[]
echo anyIt(empty, true)  # edge case: prints false
```

---

### `countIt`

```nim
template countIt*(s, pred: untyped): int
```

**What it does.** Counts how many elements of `s` satisfy the predicate `pred` (an expression with the `it` variable). Unlike `count`, the comparison here is an arbitrary condition, not equality against a specific value.

- **Parameters:**
  - `s: untyped` — the sequence being checked, or any iterable object.
  - `pred: untyped` — a boolean expression using the `it` variable.

```nim
let numbers = @[-3, -2, -1, 0, 1, 2, 3, 4, 5, 6]
echo countIt(numbers, it < 0)  # prints 3
```

---

### `count`

```nim
func count*[T](s: openArray[T], x: T): int
```

**What it does.** Counts how many times the specific value `x` occurs in `s` (comparison via `==`). This is a special case of `countIt` for comparing against a fixed value, but it's implemented separately and works without a predicate expression.

- **Parameters:**
  - `s: openArray[T]` — the sequence/array/string being checked.
  - `x: T` — the value being searched for.

```nim
let
  a = @[1, 2, 2, 3, 2, 4, 2]
  b = "abracadabra"
echo count(a, 2)   # prints 4
echo count(a, 99)  # edge case, not found: prints 0
echo count(b, 'r') # prints 2
```

---

## Aggregation and folding

### `foldl` (without an initial value)

```nim
template foldl*(sequence, operation: untyped): untyped
```

**What it does.** Folds the sequence left to right into a single value, using the variables `a` (accumulator) and `b` (current element) in `operation`. Requires at least one element — on an empty sequence there will be an assertion failure in a debug build (undefined behavior in a release build). For a single-element sequence, returns that element without applying `operation`.

**Implementation notes.** `a` and `b` are injected on every iteration via `{.inject.}`, which lets `operation` be written as an ordinary expression (`a + b`) rather than as a function with explicit parameters. For non-associative operations, the parenthesization is "left-associative": for `1, 2, 3` this is `((1 - 2) - 3)`.

- **Parameters:**
  - `sequence: untyped` — a non-empty sequence.
  - `operation: untyped` — an expression using the variables `a` (accumulated result) and `b` (the next element).

```nim
let numbers = @[5, 9, 11]
echo foldl(numbers, a + b)  # prints 25, since ((5+9)+11)
echo foldl(numbers, a - b)  # prints -15, since ((5-9)-11)
```

---

### `foldl` (with an initial value)

```nim
template foldl*(sequence, operation, first): untyped
```

**What it does.** The same as `foldl` without an initial value, but with an explicit starting value `first`, which becomes the first `a`. This allows folding a sequence into a type different from its element type (for example, numbers → string), and also lets an empty sequence be handled correctly (the result is simply `first`).

- **Parameters:**
  - `sequence: untyped` — a sequence (may be empty).
  - `operation: untyped` — an expression using `a` (accumulator) and `b` (current element).
  - `first` — the accumulator's starting value, which determines the result type.

```nim
let
  numbers = @[0, 8, 1, 5]
  digits = foldl(numbers, a & chr(b + ord('0')), "")
echo digits  # prints "0815" — folding numbers into a string
```

---

### `foldr`

```nim
template foldr*(sequence, operation: untyped): untyped
```

**What it does.** Folds the sequence right to left (unlike `foldl`). Requires at least one element. For a single element, returns it without applying `operation`. For non-associative operations, the parentheses are right-associative: for `1, 2, 3` — `(1 - (2 - 3))`.

- **Parameters:**
  - `sequence: untyped` — a non-empty sequence.
  - `operation: untyped` — an expression using `a` and `b`, where at each step `b` is the already-accumulated (right-hand) result and `a` is the next element to its left.

```nim
let numbers = @[5, 9, 11]
echo foldr(numbers, a - b)  # prints 7, since (5-(9-(11)))
```

---

### `min` / `max` (with a comparator)

```nim
proc min*[T](x: openArray[T], cmp: proc(a, b: T): int): T
proc max*[T](x: openArray[T], cmp: proc(a, b: T): int): T
```

**What it does.** Returns the minimum/maximum element of `x`, determining order via an arbitrary comparison function `cmp` (as in `sort`: a negative value means `a < b`, a positive value means `a > b`). Useful when there's no natural `<` operator for type `T`, or when a non-standard comparison criterion is needed (for example, string length rather than alphabetical order).

- **Parameters:**
  - `x: openArray[T]` — a non-empty sequence.
  - `cmp: proc(a, b: T): int` — a function comparing two elements.

```nim
let words = @["foo", "bar", "hello"]
echo min(words, proc (a, b: string): int = len(a) - len(b))  # prints "foo"
echo max(words, proc (a, b: string): int = len(a) - len(b))  # prints "hello"
```

---

### `minIndex` / `maxIndex`

```nim
func minIndex*[T](s: openArray[T]): int
func minIndex*[T](s: openArray[T], cmp: proc(a, b: T): int): int
func maxIndex*[T](s: openArray[T]): int
func maxIndex*[T](s: openArray[T], cmp: proc(a, b: T): int): int
```

**What it does.** Returns not the minimum/maximum element itself, but its *index* in `s`. The variant without `cmp` requires `T` to have a `<` operator; the variant with `cmp` works like `min`/`max` with a comparator. When there are several equal minimums/maximums, the index of the first one encountered is returned.

- **Parameters:**
  - `s: openArray[T]` — a non-empty sequence.
  - `cmp: proc(a, b: T): int` — an optional comparison function.

```nim
let c = [2, -7, 8, -5]
echo minIndex(c)  # prints 1 (value -7)
echo maxIndex(c)  # prints 2 (value 8)

let s1 = @["foo", "bar", "hello"]
echo minIndex(s1, proc (a, b: string): int = len(a) - len(b))  # prints 0
```

---

### `minmax`

```nim
func minmax*[T](x: openArray[T]): (T, T)
func minmax*[T](x: openArray[T], cmp: proc(a, b: T): int): (T, T)
```

**What it does.** Returns a pair (minimum, maximum) in one go, doing a single pass over `x` instead of two separate calls to `min`/`max`. The variant without `cmp` requires `T` to have a `<` operator; the variant with `cmp` uses a custom comparison.

**Implementation notes.** A single pass maintains two invariants at once: the current minimum and the current maximum; each element is compared at most twice (`elif`, not two independent `if`s), which is twice as efficient as calling `min(x)` and `max(x)` sequentially.

- **Parameters:**
  - `x: openArray[T]` — a non-empty sequence.
  - `cmp: proc(a, b: T): int` — an optional comparison function.

```nim
let a = [2, -7, 8, -5]
echo minmax(a)  # prints (-7, 8)
```

---

## In-place sequence modification

### `addUnique`

```nim
func addUnique*[T](s: var seq[T], x: sink T)
```

**What it does.** Adds `x` to the end of `s`, but only if such an element isn't already there (checked via `==` against every existing element). Calling it again with an already-present value has no effect.

- **Parameters:**
  - `s: var seq[T]` — the mutable sequence the element is added to.
  - `x: sink T` — the candidate value to add.

```nim
var a = @[1, 2, 3]
addUnique(a, 4)
addUnique(a, 4)  # adding the same value again — no effect
echo a  # prints @[1, 2, 3, 4]
```

---

### `deduplicate`

```nim
func deduplicate*[T](s: openArray[T], isSorted: bool = false): seq[T]
```

**What it does.** Returns a new sequence without repeated elements, preserving the order of first appearance. Does not change `s`. The `isSorted = true` flag enables a faster algorithm, applicable only if `s` is already sorted (in which case it's enough to compare neighboring elements rather than search through the whole accumulated result so far).

**Implementation notes.** At `isSorted = false`, `result.contains(itm)` is called for every element, giving quadratic O(n²) complexity in the worst case (for each of the n elements — a check against everything accumulated so far). At `isSorted = true`, the algorithm compares an element only against the previous one (`prev`), giving linear O(n) complexity — but the result will be incorrect if the input isn't actually sorted, since duplicates separated by other values won't be detected.

- **Parameters:**
  - `s: openArray[T]` — the source sequence.
  - `isSorted: bool` — enables the fast algorithm for pre-sorted data (defaults to `false`).

```nim
let dup1 = @[1, 1, 3, 4, 2, 2, 8, 1, 4]
echo deduplicate(dup1)  # prints @[1, 3, 4, 2, 8]

let dup2 = @["a", "a", "c", "d", "d"]
echo deduplicate(dup2, isSorted = true)  # prints @["a", "c", "d"]
```

---

### `delete`

```nim
func delete*[T](s: var seq[T]; slice: Slice[int])
```

**What it does.** Deletes from `s` the elements falling within the range `slice` (inclusive on both ends), shifting the remaining tail left. If `slice`'s bounds fall outside `s`, an `IndexDefect` exception is raised. An empty slice (for example, `1..<1`) is not an error but a degenerate case with no deletion.

**Implementation notes.** Elements after the deleted range are shifted left in place (`move` under `gcDestructors`, otherwise `shallowCopy`), after which `s` is truncated to the new length via `setLen`. The operation's cost is linear O(n) in the number of shifted elements — that is, from the deletion position to the end of the sequence, not from the size of the deleted range itself.

- **Parameters:**
  - `s: var seq[T]` — the mutable sequence.
  - `slice: Slice[int]` — the inclusive index range to delete.

```nim
var a = @[10, 11, 12, 13, 14]
doAssertRaises(IndexDefect): delete(a, 4..5)  # error case: out of bounds
echo a  # prints @[10, 11, 12, 13, 14] — unchanged

delete(a, 4..4)
echo a  # prints @[10, 11, 12, 13]
delete(a, 1..2)
echo a  # prints @[10, 13]
delete(a, 1..<1)  # edge case: empty slice
echo a  # prints @[10, 13] — unchanged
```

---

### `insert`

```nim
func insert*[T](dest: var seq[T], src: openArray[T], pos = 0)
```

**What it does.** Inserts all elements of `src` into `dest` starting at position `pos`, shifting `dest`'s existing elements from `pos` onward further to the right. Changes `dest` in place, does not touch `src`. `pos` defaults to `0` (insert at the start).

**Implementation notes.** First `dest` is expanded to its final length (`setLen`), then the existing tail is shifted right *from the end*, so as not to overwrite data not yet copied, and only then are `src`'s elements written into the freed window. This order (right-to-left for the shift, left-to-right for the insertion) is the standard trick for avoiding overwriting data that's still needed for reading.

- **Parameters:**
  - `dest: var seq[T]` — the mutable destination sequence.
  - `src: openArray[T]` — the elements to insert.
  - `pos: int` — the insertion position (defaults to 0).

```nim
var dest = @[1, 1, 1, 1, 1, 1, 1, 1]
let src = @[2, 2, 2, 2, 2, 2]
insert(dest, src, 3)
echo dest  # prints @[1, 1, 1, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1]

# Edge case: pos omitted — inserts at the start
var d2 = @[3, 4]
insert(d2, @[1, 2])
echo d2  # prints @[1, 2, 3, 4]

# Practical scenario: inserting a block of new tracks into the middle of a playlist
var playlist = @["track1", "track2", "track5"]
insert(playlist, @["track3", "track4"], 2)
echo playlist  # prints @["track1", "track2", "track3", "track4", "track5"]
```

---

## Combining and splitting sequences

### `zip`

```nim
proc zip*[S, T](s1: openArray[S], s2: openArray[T]): seq[(S, T)]
```

**What it does.** Returns a sequence of tuples `(s1[i], s2[i])` for each `i`. If the sequences differ in length, the extra elements of the longer one are dropped — the result's length equals the smaller of the two.

- **Parameters:**
  - `s1: openArray[S]` — the first sequence.
  - `s2: openArray[T]` — the second sequence (may be of a different type).

```nim
let
  short = @[1, 2, 3]
  long = @[6, 5, 4, 3, 2, 1]
  words = @["one", "two", "three"]
echo zip(short, long)   # prints @[(1, 6), (2, 5), (3, 4)] — the extra tail of long is dropped
echo zip(short, words)  # prints @[(1, "one"), (2, "two"), (3, "three")]
```

---

### `unzip`

```nim
proc unzip*[S, T](s: openArray[(S, T)]): (seq[S], seq[T])
```

**What it does.** The reverse operation to `zip`: splits a sequence of two-element tuples into a pair of separate sequences — all the first components into one, all the second components into the other.

- **Parameters:**
  - `s: openArray[(S, T)]` — a sequence of pairs.

```nim
let zipped = @[(1, 'a'), (2, 'b'), (3, 'c')]
echo unzip(zipped)  # prints (@[1, 2, 3], @['a', 'b', 'c'])
```

---

### `distribute`

```nim
func distribute*[T](s: seq[T], num: Positive, spread = true): seq[seq[T]]
```

**What it does.** Splits `s` into `num` nested sub-sequences. This is partly the reverse operation to `concat` (for some inputs). If `s` is empty, the result is `num` empty sub-sequences. If `num < 2`, the result is one sub-sequence equal to the whole of `s`.

**Implementation notes.** The `spread` flag determines how the remainder of dividing `len(s)` by `num` is distributed. At `spread = true`, the remainder is evenly "spread" across all sub-sequences (the first few get one extra element each) — handy, for example, for evenly handing out work to a thread pool. At `spread = false`, the entire remainder goes to the first sub-sequence, while the rest are filled strictly evenly.

- **Parameters:**
  - `s: seq[T]` — the source sequence.
  - `num: Positive` — how many parts to split into (≥ 1).
  - `spread: bool` — whether to evenly distribute the remainder (`true`, the default) or hand it entirely to the first part (`false`).

```nim
let numbers = @[1, 2, 3, 4, 5, 6, 7]
echo distribute(numbers, 3)         # prints @[@[1, 2, 3], @[4, 5], @[6, 7]]
echo distribute(numbers, 3, false)  # prints @[@[1, 2, 3], @[4, 5, 6], @[7]]
```

---

## Iterators for functional style

### `items` (for a closure iterator)

```nim
iterator items*[T](xs: iterator: T): T
```

**What it does.** Lets you iterate, via an ordinary `for`, over values yielded by a closure iterator (`iterator: T`, rather than an ordinary `for`-compatible object). On its own this might not seem particularly useful, but it's exactly this `items` overload that makes it possible to apply `mapIt`, `filterIt`, `allIt`, `anyIt`, and similar templates directly to closure iterators, not only to `seq`/`array`/`string`.

- **Parameters:**
  - `xs: iterator: T` — a closure iterator yielding values of type `T`.

```nim
iterator countUp3(n: int): int {.closure.} =
  var i = 0
  while i < n:
    yield i
    inc i

var acc: seq[int] = @[]
for x in items(countUp3(4)):
  add(acc, x)
echo acc  # prints @[0, 1, 2, 3]
```

---

## Practical recipes

### Top-N leaders by value

Find the top three players by score, without sorting the whole list — it's enough to repeatedly pull out the maximum and remove it.

```nim
type Player = tuple[name: string, score: int]

var players = @[
  ("Anna", 42), ("Bob", 87), ("Carl", 15),
  ("Dina", 87), ("Egor", 63)
]

proc topN(s: seq[Player], n: int): seq[Player] =
  var pool = s
  for i in 0 ..< min(n, len(pool)):
    let idx = maxIndex(pool, proc(a, b: Player): int = a.score - b.score)
    add(result, pool[idx])
    delete(pool, idx..idx)

let leaders = topN(players, 3)
echo leaders  # prints the top 3 by descending score: Bob, Dina, Egor
```

---

### Removing duplicates while preserving order

Collect unique email addresses from a request log, preserving the order of first appearance.

```nim
let log = @[
  "a@example.com", "b@example.com", "a@example.com",
  "c@example.com", "b@example.com"
]

let uniqueEmails = deduplicate(log)
echo uniqueEmails  # prints @["a@example.com", "b@example.com", "c@example.com"]
```

---

### Round-robin distribution of tasks across workers

Hand out a set of tasks to a fixed number of workers as evenly as possible (ignoring task weight), for further parallel processing.

```nim
let tasks = toSeq(1..10)
let perWorker = distribute(tasks, 3)  # 3 workers

for i, chunk in perWorker:
  echo "worker " & $i & " gets: " & $chunk
# worker 0 gets: @[1, 2, 3, 4]
# worker 1 gets: @[5, 6, 7]
# worker 2 gets: @[8, 9, 10]
```

---

### Word frequency analysis

Count how many words longer than three characters appear in a sentence, and print just those, converted to upper case.

```nim
import std/strutils

let sentence = "the quick brown fox jumps over the lazy dog"
let words = split(sentence, " ")

let longWords = filterIt(words, len(it) > 3)
let upperLongWords = mapIt(longWords, toUpperAscii(it))
let longWordCount = countIt(words, len(it) > 3)

echo upperLongWords  # prints @["QUICK", "BROWN", "JUMPS", "OVER", "LAZY"]
echo longWordCount   # prints 5
```

---

### Combining two parallel lists into a report

The `zip` + `filterIt` + `mapIt` combo: pair up two parallel lists, select the ones that match a condition, and turn them into ready-made report text.

```nim
import std/strformat

proc failureReport(names: openArray[string], results: openArray[bool]): seq[string] =
  let paired = zip(names, results)
  let failures = filterIt(paired, not it[1])
  mapIt(failures, fmt"{it[0]}: failed")

let
  names = @["test_a", "test_b", "test_c"]
  results = @[true, false, false]
echo failureReport(names, results)  # prints @["test_b: failed", "test_c: failed"]
```

---

### Rolling statistics over a numeric log

The `minmax`, `foldl`, and `allIt` combo: minimum, maximum, sum, and an "all values within the allowed range" check, in the fewest passes possible.

```nim
type LogStats = object
  minimum, maximum, total: float
  withinRange: bool

proc collectStats(values: openArray[float], lowerBound, upperBound: float): LogStats =
  let (lo, hi) = minmax(values)
  let total = foldl(values, a + b, 0.0)
  result = LogStats(
    minimum: lo,
    maximum: hi,
    total: total,
    withinRange: allIt(values, it >= lowerBound and it <= upperBound))

let
  readings = @[18.2, 19.5, 21.0, 17.8, 20.3]
  stats = collectStats(readings, 15.0, 25.0)
echo stats.minimum      # prints 17.8
echo stats.withinRange  # prints true
```

---

## Quick reference table

| Task | Mutates the argument | Returns a new seq/value |
|---|---|---|
| Join several seqs | no | `concat` |
| Repeat a whole seq n times | no | `cycle` |
| Repeat one element n times | no | `repeat` |
| Create a seq with an individually initialized element | no | `newSeqWith` |
| Turn a range/set/iterator into a seq | no | `toSeq` |
| Transform every element (explicit function) | no | `map` |
| Transform every element (via `it`) | no | `mapIt` |
| Mutate every element by reference | yes | `apply` (variant A) |
| Replace every element with a function's result | yes | `apply` (variant B) |
| Walk elements for a side effect | no | `apply` (variant C) |
| Replace every element with an `it`-expression | yes | `applyIt` |
| Replace literals in an AST constructor at compile time | no | `mapLiterals` |
| Keep only matching elements (explicit function) | no | `filter` (proc) |
| Keep only matching elements lazily, no intermediate seq | no | `filter` (iterator) |
| Keep only matching elements (via `it`) | no | `filterIt` |
| Keep only matching elements in place | yes | `keepIf` |
| Keep only matching elements in place (via `it`) | yes | `keepItIf` |
| Find the index of the first matching element | no | `findIt` |
| Check that every element matches | no | `all` / `allIt` |
| Check that at least one element matches | no | `any` / `anyIt` |
| Count elements by condition | no | `countIt` |
| Count occurrences of a specific value | no | `count` |
| Fold left to right (first element as accumulator) | no | `foldl` |
| Fold left to right (starting value of another type) | no | `foldl` (with `first`) |
| Fold right to left | no | `foldr` |
| Find min/max with a custom comparison | no | `min` / `max` |
| Find the index of the min/max element | no | `minIndex` / `maxIndex` |
| Get both min and max in a single pass | no | `minmax` |
| Add an element if it isn't already there | yes | `addUnique` |
| Remove repeated elements | no | `deduplicate` |
| Delete a range of elements | yes | `delete` |
| Insert a set of elements at a position | yes | `insert` |
| Stitch two seqs into a seq of pairs | no | `zip` |
| Split a seq of pairs into two seqs | no | `unzip` |
| Split a seq into N parts | no | `distribute` |
| Iterate a closure iterator in `for`/`It` templates | no | `items` |

---

## Summary: which procedure to choose

- Need to turn a range, set, or iterator into a `seq` → use `toSeq`.
- Need to create a `seq` of `N` independently initialized elements (including a "2D" seq) → use `newSeqWith`; if it's the same element repeated — `repeat`; if it's a whole seq repeated — `cycle`.
- Need to join several sequences into one → use `concat`.
- Need to transform a seq element-wise without touching the source → use `mapIt` (or `map`, if the transformation is already a standalone function).
- Need to change a seq's elements in place → use `applyIt`/`apply`, depending on whether you're mutating the element by reference or returning a new value of the same type.
- Need to keep only part of the elements, returning a new seq → use `filterIt` (or `filter`, if the predicate is already a ready-made function); for a lazy pass with no intermediate seq — the `filter` iterator.
- Need to keep only part of the elements in place, without creating a new seq → use `keepItIf`/`keepIf`.
- Need to find the index of the first matching element → use `findIt`.
- Need to check a "for all"/"at least one" condition → use `allIt`/`anyIt`.
- Need to count elements by condition → use `countIt`; if the condition is just equality to a value → `count`.
- Need to fold a seq into a single value → use `foldl` (left to right, with no starting value or with a starting value of another type) or `foldr` (right to left).
- Need to find the minimum/maximum with a non-standard comparison → use `min`/`max` with `cmp`; if you specifically need the index → `minIndex`/`maxIndex`; if you need both at once in a single pass → `minmax`.
- Need to add an element without duplicating it → use `addUnique`; need to remove existing duplicates → `deduplicate` (with `isSorted = true` if the data is definitely sorted).
- Need to delete or insert a range of elements inside a seq → use `delete`/`insert`.
- Need to stitch two sequences into pairs or split pairs back apart → use `zip`/`unzip`.
- Need to split a seq into roughly equal parts, for example to distribute work → use `distribute`.
- Need to iterate a closure iterator in a `for` loop or in `mapIt`/`filterIt` templates → use the `items` overload for iterators.
