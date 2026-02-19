# `std/sequtils` Module Reference (Nim)

> Although this module has `seq` in its name, it implements operations not only for the `seq` type, but for all containers under the `openArray` umbrella: sequences (`seq`), strings (`string`), and arrays (`array`).

---

## Table of Contents

1. [concat](#concat)
2. [addUnique](#addunique)
3. [count](#count)
4. [cycle](#cycle)
5. [repeat](#repeat)
6. [deduplicate](#deduplicate)
7. [min / max (with comparator)](#min--max-with-comparator)
8. [minIndex / maxIndex](#minindex--maxindex)
9. [minmax](#minmax)
10. [findIt](#findit)
11. [zip](#zip)
12. [unzip](#unzip)
13. [distribute](#distribute)
14. [map](#map)
15. [apply](#apply)
16. [filter (iterator and proc)](#filter-iterator-and-proc)
17. [keepIf](#keepif)
18. [delete](#delete)
19. [insert](#insert)
20. [filterIt](#filterit)
21. [keepItIf](#keepitif)
22. [countIt](#countit)
23. [all / allIt](#all--allit)
24. [any / anyIt](#any--anyit)
25. [toSeq](#toseq)
26. [foldl](#foldl)
27. [foldr](#foldr)
28. [mapIt](#mapit)
29. [applyIt](#applyit)
30. [newSeqWith](#newseqwith)
31. [mapLiterals](#mapliterals)
32. [items (for closure iterators)](#items-for-closure-iterators)

---

## concat

```nim
func concat*[T](seqs: varargs[seq[T]]): seq[T]
```

**Description:** Combines several sequences into a single new sequence. All input sequences must be of the same type.

**Example:**
```nim
let
  s1 = @[1, 2, 3]
  s2 = @[4, 5]
  s3 = @[6, 7]
  total = concat(s1, s2, s3)
assert total == @[1, 2, 3, 4, 5, 6, 7]
```

> **Reverse operation:** `distribute` — splits a sequence into several sub-sequences.

---

## addUnique

```nim
func addUnique*[T](s: var seq[T], x: sink T)
```

**Description:** Adds element `x` to sequence `s` only if it is not already present. Uses the `==` operator for comparison.

**Example:**
```nim
var a = @[1, 2, 3]
a.addUnique(4)
a.addUnique(4)  # Second call does nothing
assert a == @[1, 2, 3, 4]
```

---

## count

```nim
func count*[T](s: openArray[T], x: T): int
```

**Description:** Returns the number of occurrences of element `x` in container `s`.

**Example:**
```nim
let
  a = @[1, 2, 2, 3, 2, 4, 2]
  b = "abracadabra"
assert count(a, 2) == 4
assert count(a, 99) == 0
assert count(b, 'r') == 2
```

---

## cycle

```nim
func cycle*[T](s: openArray[T], n: Natural): seq[T]
```

**Description:** Returns a new sequence with the elements of container `s` repeated `n` times. `n` must be a non-negative number.

**Example:**
```nim
let
  s = @[1, 2, 3]
  total = s.cycle(3)
assert total == @[1, 2, 3, 1, 2, 3, 1, 2, 3]
```

---

## repeat

```nim
proc repeat*[T](x: T, n: Natural): seq[T]
```

**Description:** Returns a new sequence with the single element `x` repeated `n` times. Unlike `cycle`, it works with one element rather than a container.

**Example:**
```nim
let total = repeat(5, 3)
assert total == @[5, 5, 5]
```

---

## deduplicate

```nim
func deduplicate*[T](s: openArray[T], isSorted: bool = false): seq[T]
```

**Description:** Returns a new sequence without duplicate elements. Setting the optional `isSorted` argument to `true` (default: `false`) enables a faster algorithm for already-sorted input.

**Example:**
```nim
let
  dup1    = @[1, 1, 3, 4, 2, 2, 8, 1, 4]
  dup2    = @["a", "a", "c", "d", "d"]
  unique1 = deduplicate(dup1)
  unique2 = deduplicate(dup2, isSorted = true)
assert unique1 == @[1, 3, 4, 2, 8]
assert unique2 == @["a", "c", "d"]
```

> **Note:** Without `isSorted = true`, the order of the first occurrence of each unique element is preserved. With `isSorted = true`, it works like a `unique` algorithm — faster, but requires sorted input.

---

## min / max (with comparator)

```nim
proc min*[T](x: openArray[T], cmp: proc(a, b: T): int): T
proc max*[T](x: openArray[T], cmp: proc(a, b: T): int): T
```

**Description:** Finds the minimum or maximum element of an array using a custom comparison function. `cmp` should return a negative number, zero, or a positive number (analogous to C's `qsort`).

**Example:**
```nim
import std/sugar

let words = @["apple", "fig", "banana"]
# Minimum by string length
let shortest = words.min((a, b) => a.len - b.len)
assert shortest == "fig"

# Maximum by string length
let longest = words.max((a, b) => a.len - b.len)
assert longest == "banana"
```

---

## minIndex / maxIndex

```nim
func minIndex*[T](s: openArray[T]): int
func minIndex*[T](s: openArray[T], cmp: proc(a, b: T): int): int

func maxIndex*[T](s: openArray[T]): int
func maxIndex*[T](s: openArray[T], cmp: proc(a, b: T): int): int
```

**Description:** Returns the index of the minimum or maximum element. Overloads without a comparator require type `T` to support the `<` operator. Overloads with a comparator accept a custom comparison function.

**Example (without comparator):**
```nim
let
  a = @[1, 2, 3, 4]
  b = @[6, 5, 4, 3]
  c = [2, -7, 8, -5]
assert minIndex(a) == 0   # minimum is 1, at index 0
assert minIndex(b) == 3   # minimum is 3, at index 3
assert minIndex(c) == 1   # minimum is -7, at index 1
assert maxIndex(c) == 2   # maximum is 8, at index 2
```

**Example (with comparator):**
```nim
import std/sugar

let s1 = @["foo", "bar", "hello"]
# Index of the shortest string
assert minIndex(s1, (a, b) => a.len - b.len) == 0
# Index of the longest string
assert maxIndex(s1, (a, b) => a.len - b.len) == 2
```

---

## minmax

```nim
func minmax*[T](x: openArray[T]): (T, T)
func minmax*[T](x: openArray[T], cmp: proc(a, b: T): int): (T, T)
```

**Description:** Returns a `(minimum, maximum)` tuple in a single pass over the container. More efficient than calling `min` and `max` separately.

**Example:**
```nim
let
  a = @[3, 1, 4, 1, 5, 9, 2, 6]
  (lo, hi) = minmax(a)
assert lo == 1
assert hi == 9
```

---

## findIt

```nim
template findIt*(s, predicate: untyped): int
```

**Description:** Iterates through a container and returns the index of the first element satisfying the predicate. Returns `-1` if no element is found. The predicate is written as an expression using the variable `it`.

**Example:**
```nim
let a = @[10, 20, 30, 40, 50]
assert findIt(a, it > 25) == 2    # first element > 25 is 30, at index 2
assert findIt(a, it > 100) == -1  # not found
```

---

## zip

```nim
proc zip*[S, T](s1: openArray[S], s2: openArray[T]): seq[(S, T)]
```

**Description:** Combines two containers into a sequence of tuples. If the containers have different lengths, the extra elements of the longer container are discarded.

**Example:**
```nim
let
  short = @[1, 2, 3]
  long  = @[6, 5, 4, 3, 2, 1]
  words = @["one", "two", "three"]
  zip1  = zip(short, long)
  zip2  = zip(short, words)
assert zip1 == @[(1, 6), (2, 5), (3, 4)]
assert zip2 == @[(1, "one"), (2, "two"), (3, "three")]
```

---

## unzip

```nim
proc unzip*[S, T](s: openArray[(S, T)]): (seq[S], seq[T])
```

**Description:** The reverse of `zip`. Takes a sequence of 2-element tuples and returns a tuple of two sequences.

**Example:**
```nim
let
  zipped        = @[(1, 'a'), (2, 'b'), (3, 'c')]
  (nums, chars) = zipped.unzip()
assert nums  == @[1, 2, 3]
assert chars == @['a', 'b', 'c']
```

---

## distribute

```nim
func distribute*[T](s: seq[T], num: Positive, spread = true): seq[seq[T]]
```

**Description:** Splits sequence `s` into `num` sub-sequences. Acts as a partial reverse of `concat`.

- `spread = true` (default): the remainder is distributed evenly across sub-sequences — ideal for multithreaded workloads.
- `spread = false`: earlier sub-sequences are filled to capacity; the remainder goes into the last one.

**Example:**
```nim
let numbers = @[1, 2, 3, 4, 5, 6, 7]
assert numbers.distribute(3)        == @[@[1, 2, 3], @[4, 5], @[6, 7]]
assert numbers.distribute(3, false) == @[@[1, 2, 3], @[4, 5, 6], @[7]]
```

---

## map

```nim
proc map*[T, S](s: openArray[T], op: proc(x: T): S): seq[S]
```

**Description:** Applies function `op` to every element of the container and returns a new sequence with the results. Does not modify the original container. Can be used to transform the element type.

**Example:**
```nim
let
  a = @[1, 2, 3, 4]
  b = map(a, proc(x: int): string = $x)
assert b == @["1", "2", "3", "4"]
```

**With syntactic sugar from the `sugar` module:**
```nim
import std/sugar

let a = @[1, 2, 3, 4]
let b = a.map(x => x * x)
assert b == @[1, 4, 9, 16]
```

---

## apply

```nim
# Variant 1: takes var T — modifies the element directly
proc apply*[T](s: var openArray[T], op: proc(x: var T))

# Variant 2: takes T, returns T — replaces each element
proc apply*[T](s: var openArray[T], op: proc(x: T): T)

# Variant 3: takes T, returns nothing — for side effects only
proc apply*[T](s: openArray[T], op: proc(x: T))
```

**Description:** Applies function `op` to every element of the container. Unlike `map`, the first two variants work in-place, modifying the original container.

**Example (variant 1 — modify via var):**
```nim
var a = @["1", "2", "3", "4"]
apply(a, proc(x: var string) = x &= "42")
assert a == @["142", "242", "342", "442"]
```

**Example (variant 2 — replace via return value):**
```nim
var a = @["1", "2", "3", "4"]
apply(a, proc(x: string): string = x & "42")
assert a == @["142", "242", "342", "442"]
```

**Example (variant 3 — side effects only):**
```nim
var message: string
apply([0, 1, 2, 3, 4], proc(item: int) = message.addInt item)
assert message == "01234"
```

---

## filter (iterator and proc)

### Iterator
```nim
iterator filter*[T](s: openArray[T], pred: proc(x: T): bool): T
```

**Description:** Iterates over container `s` and **yields** only the elements that satisfy predicate `pred`. Useful when you do not need to create an intermediate sequence.

**Example:**
```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
var evens = newSeq[int]()
for n in filter(numbers, proc(x: int): bool = x mod 2 == 0):
  evens.add(n)
assert evens == @[4, 8, 4]
```

### Proc
```nim
proc filter*[T](s: openArray[T], pred: proc(x: T): bool): seq[T]
```

**Description:** Returns a new sequence containing only the elements that satisfy the predicate.

**Example:**
```nim
let
  colors = @["red", "yellow", "black"]
  f1 = filter(colors, proc(x: string): bool = x.len < 6)
  f2 = filter(colors, proc(x: string): bool = x.contains('y'))
assert f1 == @["red", "black"]
assert f2 == @["yellow"]
```

---

## keepIf

```nim
proc keepIf*[T](s: var seq[T], pred: proc(x: T): bool)
```

**Description:** Modifies sequence `s` in-place, keeping only the elements that satisfy the predicate. An in-place alternative to `filter` that avoids allocating a new sequence.

**Example:**
```nim
var floats = @[13.0, 12.5, 5.8, 2.0, 6.1, 9.9, 10.1]
keepIf(floats, proc(x: float): bool = x > 10)
assert floats == @[13.0, 12.5, 10.1]
```

---

## delete

```nim
func delete*[T](s: var seq[T]; slice: Slice[int])
```

**Description:** Deletes the elements `s[slice]` from sequence `s`, raising `IndexDefect` if the slice contains out-of-range indices. Shifts all elements after the deleted range — an O(n) operation.

**Example:**
```nim
var a = @[10, 11, 12, 13, 14]
a.delete(4..4)    # Delete the last element
assert a == @[10, 11, 12, 13]
a.delete(1..2)    # Delete elements at indices 1 and 2
assert a == @[10, 13]
a.delete(1..<1)   # Empty slice — nothing happens
assert a == @[10, 13]
```

> **Deprecated overload:** `delete(s, first, last)` (two separate indices) is marked deprecated — use slices instead.

---

## insert

```nim
func insert*[T](dest: var seq[T], src: openArray[T], pos = 0)
```

**Description:** Inserts elements from `src` into `dest` starting at position `pos`. Modifies `dest` in-place. The element types of `src` and `dest` must match.

**Example:**
```nim
var dest = @[1, 1, 1, 1, 1, 1, 1, 1]
let src  = @[2, 2, 2, 2, 2, 2]
dest.insert(src, 3)
assert dest == @[1, 1, 1, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1]
```

---

## filterIt

```nim
template filterIt*(s, pred: untyped): untyped
```

**Description:** Template version of `filter`. Returns a new sequence containing elements that satisfy the predicate. The predicate is written as an expression using the variable `it`, with no need to define an anonymous function.

**Example:**
```nim
let
  temperatures  = @[-272.15, -2.0, 24.5, 44.31, 99.9, -113.44]
  acceptable    = temperatures.filterIt(it < 50 and it > -10)
  notAcceptable = temperatures.filterIt(it > 50 or it < -10)
assert acceptable    == @[-2.0, 24.5, 44.31]
assert notAcceptable == @[-272.15, 99.9, -113.44]
```

---

## keepItIf

```nim
template keepItIf*(varSeq: seq, pred: untyped)
```

**Description:** Template version of `keepIf`. Modifies the sequence in-place, keeping only elements that satisfy the predicate. The predicate uses the variable `it`.

**Example:**
```nim
var candidates = @["foo", "bar", "baz", "foobar"]
candidates.keepItIf(it.len == 3 and it[0] == 'b')
assert candidates == @["bar", "baz"]
```

---

## countIt

```nim
template countIt*(s, pred: untyped): int
```

**Description:** Returns the count of elements in the container that satisfy the predicate. The predicate uses the variable `it`. Also works with closure iterators.

**Example:**
```nim
let numbers = @[-3, -2, -1, 0, 1, 2, 3, 4, 5, 6]
assert numbers.countIt(it < 0) == 3
assert numbers.countIt(it mod 2 == 0) == 5
```

---

## all / allIt

### all
```nim
proc all*[T](s: openArray[T], pred: proc(x: T): bool): bool
```

**Description:** Checks whether **all** elements of the container satisfy the predicate. Returns `true` only if the predicate holds for every element. Returns `false` at the first mismatch.

**Example:**
```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
assert all(numbers, proc(x: int): bool = x < 10) == true
assert all(numbers, proc(x: int): bool = x < 9)  == false
```

### allIt
```nim
template allIt*(s, pred: untyped): bool
```

**Description:** Template version of `all` using the variable `it`.

**Example:**
```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
assert numbers.allIt(it < 10) == true
assert numbers.allIt(it < 9)  == false
```

---

## any / anyIt

### any
```nim
proc any*[T](s: openArray[T], pred: proc(x: T): bool): bool
```

**Description:** Checks whether **at least one** element of the container satisfies the predicate. Returns `true` at the first match.

**Example:**
```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
assert any(numbers, proc(x: int): bool = x > 8) == true
assert any(numbers, proc(x: int): bool = x > 9) == false
```

### anyIt
```nim
template anyIt*(s, pred: untyped): bool
```

**Description:** Template version of `any` using the variable `it`.

**Example:**
```nim
let numbers = @[1, 4, 5, 8, 9, 7, 4]
assert numbers.anyIt(it > 8) == true
assert numbers.anyIt(it > 9) == false
```

---

## toSeq

```nim
template toSeq*(iter: untyped): untyped
```

**Description:** Converts any iterable (range, set, iterator, etc.) into a `seq`.

**Example:**
```nim
let
  myRange = 1..5
  mySet: set[int8] = {5'i8, 3, 1}
  mySeq1 = toSeq(myRange)
  mySeq2 = toSeq(mySet)
assert mySeq1 == @[1, 2, 3, 4, 5]
assert mySeq2 == @[1'i8, 3, 5]  # sets are stored in sorted order
```

**With an inline iterator:**
```nim
iterator evens(n: int): int =
  for i in 0..<n:
    if i mod 2 == 0: yield i

assert toSeq(evens(10)) == @[0, 2, 4, 6, 8]
```

---

## foldl

### Without a starting value
```nim
template foldl*(sequence, operation: untyped): untyped
```

**Description:** Folds a sequence from left to right using the expression `operation` with variables `a` (accumulator) and `b` (current element). The sequence must have at least one element. The result type matches the element type.

**Example:**
```nim
let
  numbers       = @[5, 9, 11]
  addition      = foldl(numbers, a + b)   # ((5 + 9) + 11) = 25
  subtraction   = foldl(numbers, a - b)   # ((5 - 9) - 11) = -15
  words         = @["nim", "is", "cool"]
  concatenation = foldl(words, a & b)     # "nimiscool"
assert addition == 25
assert subtraction == -15
assert concatenation == "nimiscool"
```

### With a starting value
```nim
template foldl*(sequence, operation, first): untyped
```

**Description:** A version of `foldl` with a starting value `first`. The result type is determined by the type of `first`, allowing accumulation into a different type than the sequence elements.

**Example:**
```nim
let
  numbers = @[0, 8, 1, 5]
  # Convert numbers to a string of digits
  digits = foldl(numbers, a & (chr(b + ord('0'))), "")
assert digits == "0815"
```

---

## foldr

```nim
template foldr*(sequence, operation: untyped): untyped
```

**Description:** Folds a sequence from right to left. Variables `a` and `b`: `a` is the current left element, `b` is the right accumulator. Parenthesization is `(1 op (2 op (3)))`.

**Example:**
```nim
let
  numbers       = @[5, 9, 11]
  addition      = foldr(numbers, a + b)   # (5 + (9 + 11)) = 25
  subtraction   = foldr(numbers, a - b)   # (5 - (9 - 11)) = 7
  words         = @["nim", "is", "cool"]
  concatenation = foldr(words, a & b)     # "nimiscool"
assert addition == 25
assert subtraction == 7    # Different from foldl!
assert concatenation == "nimiscool"
```

> **Note:** For non-associative operations (subtraction, division), `foldl` and `foldr` produce different results.

---

## mapIt

```nim
template mapIt*(s: typed, op: untyped): untyped
```

**Description:** Template version of `map`. Applies expression `op` to every element of the container and returns a new sequence. Uses the variable `it` to refer to the current element.

**Example:**
```nim
let
  nums    = @[1, 2, 3, 4]
  strings = nums.mapIt($(4 * it))
assert strings == @["4", "8", "12", "16"]
```

**Method chaining:**
```nim
let result = @[1, 2, 3, 4, 5]
  .mapIt(it * 2)
  .filterIt(it mod 4 != 0)
assert result == @[2, 6, 10]
```

---

## applyIt

```nim
template applyIt*(varSeq, op: untyped)
```

**Description:** Template version of `apply`. Modifies each element of the sequence in-place using the variable `it`. The expression `op` must return the same type as the sequence element.

**Example:**
```nim
var nums = @[1, 2, 3, 4]
nums.applyIt(it * 3)
assert nums == @[3, 6, 9, 12]
assert nums[0] + nums[3] == 15
```

---

## newSeqWith

```nim
template newSeqWith*(len: int, init: untyped): untyped
```

**Description:** Creates a new sequence of length `len`, evaluating the expression `init` for each element. Unlike `newSeq`, the `init` expression is evaluated fresh for every index — essential for creating nested sequences where each inner sequence must be an independent object.

**Example (nested sequences):**
```nim
var seq2D = newSeqWith(5, newSeq[bool](3))
assert seq2D.len == 5
assert seq2D[0].len == 3
# Each inner sequence is a separate object
seq2D[0][0] = true
assert seq2D[1][0] == false  # other rows are unaffected
```

**Example (random numbers):**
```nim
import std/random
var seqRand = newSeqWith(20, rand(1.0))
# Each element is a fresh call to rand()
assert seqRand[0] != seqRand[1]  # very likely to differ
```

---

## mapLiterals

```nim
macro mapLiterals*(constructor, op: untyped; nested = true): untyped
```

**Description:** Applies operation `op` to all atomic literals (numbers, strings) in the constructor AST. Useful for type-casting elements in arrays and tuples.

- `nested = true` (default): the operation is applied recursively to nested structures.
- `nested = false`: only literals at the top level are transformed.

**Example (type casting):**
```nim
let x = mapLiterals([0.1, 1.2, 2.3, 3.4], int)
assert x is array[4, int]
assert x == [int(0.1), int(1.2), int(2.3), int(3.4)]
# Result: [0, 1, 2, 3]
```

**Example (nested tuples):**
```nim
let a = mapLiterals((1.2, (2.3, 3.4), 4.8), int)
let b = mapLiterals((1.2, (2.3, 3.4), 4.8), int, nested = false)
assert a == (1, (2, 3), 4)     # nested=true: converts everywhere
assert b == (1, (2.3, 3.4), 4) # nested=false: top level only
```

**Example (converting to string):**
```nim
let c = mapLiterals((1, (2, 3), 4, (5, 6)), `$`)
assert c == ("1", ("2", "3"), "4", ("5", "6"))
```

---

## items (for closure iterators)

```nim
iterator items*[T](xs: iterator: T): T
```

**Description:** Allows iterating over the elements of a closure iterator. This enables closure iterators to be used with the `mapIt`, `filterIt`, `allIt`, `anyIt`, and other templates that rely on `items`.

**Example:**
```nim
iterator countdown3(): int {.closure.} =
  yield 3
  yield 2
  yield 1

let asSeq = toSeq(countdown3)
assert asSeq == @[3, 2, 1]

# Now usable with It-templates:
assert countdown3.allIt(it > 0)
assert countdown3.anyIt(it == 2)
```

---

## Quick comparison: proc vs template (`It`-versions)

| Task | Proc / func version | Template (`It`-version) |
|------|--------------------|-----------------------|
| Transform elements | `map(s, proc(x: T): S = ...)` | `s.mapIt(expr with it)` |
| Modify elements in-place | `apply(s, proc(x: T): T = ...)` | `s.applyIt(expr with it)` |
| Filter to new sequence | `filter(s, proc(x: T): bool = ...)` | `s.filterIt(expr with it)` |
| Filter in-place | `keepIf(s, proc(x: T): bool = ...)` | `s.keepItIf(expr with it)` |
| Find index | *(no direct equivalent)* | `findIt(s, expr with it)` |
| Count matches | *(no direct equivalent)* | `countIt(s, expr with it)` |
| Check all | `all(s, proc(x: T): bool = ...)` | `s.allIt(expr with it)` |
| Check any | `any(s, proc(x: T): bool = ...)` | `s.anyIt(expr with it)` |

`It`-templates are more concise and require no explicit type annotations. Proc-versions are more flexible: they can be passed as values and reused across call sites.

---

## Comprehensive example

```nim
import std/sequtils, std/sugar

# Build a sequence from 1 to 10, double each element,
# keep only those not divisible by 6
let
  foo = toSeq(1..10).map(x => x * 2).filter(x => x mod 6 != 0)
  bar = toSeq(1..10).mapIt(it * 2).filterIt(it mod 6 != 0)

assert foo == bar
assert foo == @[2, 4, 8, 10, 14, 16, 20]

assert foo.any(x => x > 17)        # is there an element > 17?
assert not bar.allIt(it < 20)      # not all elements are < 20
assert foo.foldl(a + b) == 74      # sum of all elements
```
