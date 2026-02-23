# options — Module Reference

> **Nim standard library** | `import std/options`

## What Is an Option?

An **`Option[T]`** is a container that either holds exactly one value of type `T`, or holds nothing at all. It makes the concept of "a value that might be absent" explicit in the type system, instead of relying on conventions like returning `-1`, `""`, or `nil`.

```
Option[int]
├── some(42)   — contains the integer 42
└── none(int)  — contains nothing
```

**Why use it?** In Nim, `nil` only works for pointer and reference types. For value types like `int`, `float`, or `object`, there is no natural "absent" sentinel. `Option[T]` solves this uniformly for any type. It also forces callers to explicitly handle the "no value" case — the compiler will not let you silently use a value that might not be there.

**The golden rule:** Never call `get` on an `Option` without first checking `isSome`. If you call `get` on a `none`, an `UnpackDefect` is raised — and since it inherits from `Defect`, it should never be caught.

---

## Table of Contents

1. [Types](#types)
   - [Option\[T\]](#optiont)
   - [UnpackDefect](#unpackdefect)
2. [Constructors](#constructors)
   - [some](#some)
   - [none (typedesc)](#none-typedesc)
   - [none (generic)](#none-generic)
   - [option](#option)
3. [Checking for a Value](#checking-for-a-value)
   - [isSome](#issome)
   - [isNone](#isnone)
4. [Extracting the Value](#extracting-the-value)
   - [get (safe, read-only)](#get-safe-read-only)
   - [get (with default)](#get-with-default)
   - [get (mutable)](#get-mutable)
   - [unsafeGet](#unsafeget)
5. [Transforming Options](#transforming-options)
   - [map (side-effect)](#map-side-effect)
   - [map (transform)](#map-transform)
   - [flatMap](#flatmap)
   - [flatten](#flatten)
   - [filter](#filter)
6. [Operators and Utilities](#operators-and-utilities)
   - [==](#operator-)
   - [$](#operator--1)
7. [Pattern Matching](#pattern-matching)
8. [Complete Example](#complete-example)
9. [Quick Reference](#quick-reference)

---

## Types

### `Option[T]`

```nim
type Option*[T] = object
```

A generic value-type container. For pointer types (`ref`, `ptr`, `pointer`, `proc`, `iterator {.closure.}`), an absent value is represented internally as `nil` — no extra boolean is needed. For all other types, a hidden `has: bool` field tracks presence. This distinction is entirely internal; the public API behaves identically for both cases.

Because `Option[T]` is a value type (an `object`, not a `ref object`), assigning one `Option` to another copies it. There is no aliasing.

---

### `UnpackDefect`

```nim
type UnpackDefect* = object of Defect
```

Raised by `get` when called on a `none` value. Because it inherits from `Defect` (not `Exception`), it represents a **programming error** — the same category as index-out-of-bounds or nil dereference. You should not catch it; instead, check `isSome` before calling `get`.

---

## Constructors

### `some`

```nim
proc some*[T](val: sink T): Option[T]
```

Creates an `Option` that **contains** `val`. The value is moved into the option (`sink`), so no copy is made for move-eligible types.

For pointer types, `some` asserts at runtime that `val` is not `nil`. If you have a pointer that might be `nil`, use `option` instead.

```nim
let a = some(42)          # Option[int], has a value
let b = some("hello")     # Option[string], has a value
let c = some(@[1, 2, 3])  # Option[seq[int]], has a value

assert a.isSome
assert a.get == 42
```

---

### `none` (typedesc)

```nim
proc none*(T: typedesc): Option[T]
```

Creates an **empty** `Option` for the given type. The type must be passed explicitly because Nim cannot infer it from nothing.

```nim
let a = none(int)          # empty Option[int]
let b = none(string)       # empty Option[string]
let c = none(seq[float])   # empty Option[seq[float]]

assert a.isNone
```

---

### `none` (generic)

```nim
proc none*[T]: Option[T]
```

A generic alias for `none(T)`. Useful when the type parameter can be inferred from context (e.g., as a return value in a typed function), saving you from writing the type name twice.

```nim
proc safeDivide(a, b: int): Option[int] =
  if b == 0: return none[int]()   # or: return none(int)
  some(a div b)
```

---

### `option`

```nim
proc option*[T](val: sink T): Option[T]
```

A "smart" constructor for pointer types: converts `nil` to `none(T)` and any non-nil value to `some(val)`. For non-pointer types, it behaves identically to `some(val)`.

Use this when you receive a pointer that may legitimately be `nil` and you want to lift it into the `Option` world cleanly — without writing an `if` statement yourself.

```nim
type Node = ref object
  value: int

proc findNode(id: int): Node = nil  # returns nil when not found

# option() handles the nil → none conversion for you
let n = option(findNode(99))
assert n.isNone

let m = option(findNode(0))  # hypothetically returns a node
# m would be some(node)
```

```nim
# For non-pointer types, option and some are identical:
assert option(42) == some(42)
```

---

## Checking for a Value

Always check before extracting. These two predicates are the foundation of safe `Option` usage.

### `isSome`

```nim
proc isSome*[T](self: Option[T]): bool
```

Returns `true` if the option contains a value, `false` if it is empty. This is the standard guard before calling `get`.

```nim
let x = some(10)
let y = none(int)

assert x.isSome      # true
assert not y.isSome  # false

if x.isSome:
  echo x.get  # safe — we checked first
```

---

### `isNone`

```nim
proc isNone*[T](self: Option[T]): bool
```

The logical inverse of `isSome`. Returns `true` if the option is empty. Use whichever reads more naturally in your context.

```nim
let result = none(string)

if result.isNone:
  echo "Nothing was found."
```

---

## Extracting the Value

### `get` (safe, read-only)

```nim
proc get*[T](self: Option[T]): lent T
```

Returns the contained value as a `lent` (borrowed) reference. Raises `UnpackDefect` if the option is empty. The returned reference is valid as long as the option itself is alive.

This is the standard way to read a value after confirming it is present with `isSome`.

```nim
let x = some("hello")
echo x.get          # "hello"
echo x.get.len      # 5 — methods can be called on the result directly

let y = none(string)
# y.get  ← this would raise UnpackDefect at runtime
```

---

### `get` (with default)

```nim
proc get*[T](self: Option[T], otherwise: T): T
```

Returns the contained value if present, or `otherwise` if the option is empty. This eliminates the need for a separate `if isSome` check in many common situations and is the idiomatic way to provide a fallback value.

```nim
let found    = some(42)
let notFound = none(int)

echo found.get(0)     # 42   — value is present, default is ignored
echo notFound.get(0)  # 0    — option is empty, default is returned

# Useful inline when a default makes sense:
let name = getUserName().get("Anonymous")
```

---

### `get` (mutable)

```nim
proc get*[T](self: var Option[T]): var T
```

Returns the contained value as a mutable reference, allowing in-place modification. Raises `UnpackDefect` if the option is empty. The option must be a `var` for this overload to be selected.

```nim
var counter = some(0)
counter.get += 1       # increment in place
counter.get += 1
assert counter.get == 2

var empty = none(int)
# empty.get += 1  ← UnpackDefect
```

---

### `unsafeGet`

```nim
proc unsafeGet*[T](self: Option[T]): lent T
```

Returns the contained value **without checking** whether the option is actually `some`. If called on a `none`, the behaviour is undefined — in debug builds an assertion fires; in release builds memory corruption may occur.

Use this only in tight inner loops where you have already confirmed `isSome` and every nanosecond counts. In all other circumstances, use `get`.

```nim
let x = some(99)
# We have checked isSome externally — safe to unsafeGet here
if x.isSome:
  echo x.unsafeGet  # 99, no second check inside
```

---

## Transforming Options

These procedures let you work with `Option` values functionally — applying transformations, chaining operations, and filtering — all without ever manually unpacking the option.

### `map` (side-effect)

```nim
proc map*[T](self: Option[T], callback: proc (input: T))
```

Calls `callback` with the contained value if the option is `some`. Does nothing if it is `none`. The callback returns nothing (`void`) — this overload is for side effects only.

This is the "do something if present" pattern, expressed without an `if` statement.

```nim
var log: seq[string]

some("event").map(proc(s: string) = log.add(s))
assert log == @["event"]

none(string).map(proc(s: string) = log.add(s))
assert log == @["event"]   # unchanged — none, nothing happened
```

---

### `map` (transform)

```nim
proc map*[T, R](self: Option[T], callback: proc (input: T): R): Option[R]
```

Applies `callback` to the contained value and wraps the result in a new `Option`. If the option is `none`, returns `none(R)` without calling the callback at all. The type parameter `R` is inferred from the callback's return type.

This is the fundamental building block for transforming optional values without unpacking them.

```nim
let x = some(6)
let doubled = x.map(proc(n: int): int = n * 2)
assert doubled == some(12)

let y = none(int)
assert y.map(proc(n: int): int = n * 2) == none(int)

# Chaining transformations:
let result = some("  hello  ")
  .map(proc(s: string): string = s.strip)
  .map(proc(s: string): int = s.len)
assert result == some(5)
```

---

### `flatMap`

```nim
proc flatMap*[T, R](self: Option[T],
                    callback: proc (input: T): Option[R]): Option[R]
```

Like `map`, but the callback itself returns an `Option[R]` instead of a plain `R`. The result is automatically flattened — you get `Option[R]`, not `Option[Option[R]]`.

This is the key combinator for chaining multiple operations that each might fail. Instead of nested `if isSome` checks, you can write a clean pipeline.

```nim
proc parseInt(s: string): Option[int] =
  try: some(s.parseInt)
  except: none(int)

proc safeSqrt(x: int): Option[float] =
  if x >= 0: some(sqrt(float(x)))
  else: none(float)

# Chain: parse a string, then compute its square root
let r1 = some("16").flatMap(parseInt).flatMap(safeSqrt)
assert r1 == some(4.0)

let r2 = some("abc").flatMap(parseInt).flatMap(safeSqrt)
assert r2 == none(float)  # parse failed, chain short-circuits

let r3 = some("-4").flatMap(parseInt).flatMap(safeSqrt)
assert r3 == none(float)  # sqrt of negative fails
```

---

### `flatten`

```nim
proc flatten*[T](self: Option[Option[T]]): Option[T]
```

Removes one layer of nesting from an `Option[Option[T]]`. If the outer option is `some`, returns the inner option unchanged. If the outer option is `none`, returns `none(T)`.

You rarely need to call this directly — `flatMap` calls it internally. It becomes useful when you have code that independently produces a nested option structure.

```nim
let nested   = some(some(42))
let unwrapped = flatten(nested)
assert unwrapped == some(42)

let outerNone = none(Option[int])
assert flatten(outerNone) == none(int)

let innerNone = some(none(int))
assert flatten(innerNone) == none(int)
```

---

### `filter`

```nim
proc filter*[T](self: Option[T], callback: proc (input: T): bool): Option[T]
```

Applies a predicate to the contained value. If the option is `some` and the predicate returns `true`, the option is returned unchanged. If the predicate returns `false`, `none(T)` is returned. If the option is already `none`, it passes through unchanged.

Think of it as "keep the value only if it satisfies this condition."

```nim
proc isPositive(x: int): bool = x > 0

assert some(42).filter(isPositive)  == some(42)
assert some(-5).filter(isPositive)  == none(int)
assert none(int).filter(isPositive) == none(int)

# Combining with map:
let result = some("hello world")
  .filter(proc(s: string): bool = s.len <= 20)
  .map(proc(s: string): string = s.toUpperAscii)
assert result == some("HELLO WORLD")
```

---

## Operators and Utilities

### `==` operator

```nim
proc `==`*[T](a, b: Option[T]): bool
```

Two options are equal if both are `none`, or if both are `some` and their contained values are equal via `==`. A `some` and a `none` are never equal, even if the default value of `T` would equal the `some` content.

```nim
assert some(42)    == some(42)    # both some, equal values
assert none(int)   == none(int)   # both none
assert some(42)    != none(int)   # one some, one none
assert some(42)    != some(43)    # both some, different values
```

---

### `$` operator

```nim
proc `$`*[T](self: Option[T]): string
```

Returns a human-readable string representation. `some(x)` produces `"some(value)"` and `none(T)` produces `"none(TypeName)"`. Useful for debugging and `echo`.

```nim
echo some(42)          # some(42)
echo some("hi")        # some("hi")
echo none(int)         # none(int)
echo some(@[1, 2, 3])  # some(@[1, 2, 3])
```

---

## Pattern Matching

If you install the [`fusion`](https://github.com/nim-lang/fusion) package, you can use pattern matching on `Option` with the `fusion/matching` module. This requires `{.experimental: "caseStmtMacros".}`.

```nim
{.experimental: "caseStmtMacros".}
import fusion/matching

case some(42)
of Some(@value):
  echo "Got: ", value   # Got: 42
of None():
  echo "Nothing"

# Nested options also work:
assertMatch(some(some(none(int))), Some(Some(None())))
```

This is more expressive than `isSome`/`get` for complex destructuring, but requires a separate package.

---

## Complete Example

The following example shows a realistic pipeline: parsing user input from a web form, validating it step by step using `Option`, and responding appropriately — all without a single nested `if` or explicit `nil` check.

```nim
import std/options
import std/strutils

proc parseAge(s: string): Option[int] =
  ## Parse a string as an integer, return none if it fails.
  try: some(parseInt(s.strip))
  except ValueError: none(int)

proc validateAge(age: int): Option[int] =
  ## Accept only ages between 0 and 150.
  some(age).filter(proc(a: int): bool = a in 0..150)

proc ageCategory(age: int): string =
  if age < 18: "minor"
  elif age < 65: "adult"
  else: "senior"

proc processInput(rawAge: string): string =
  let category = parseAge(rawAge)
    .flatMap(validateAge)
    .map(ageCategory)
  category.get("invalid input")

echo processInput("34")    # adult
echo processInput("  17 ") # minor
echo processInput("200")   # invalid input  (failed validation)
echo processInput("abc")   # invalid input  (failed parsing)
echo processInput("-5")    # invalid input  (failed validation)
```

---

## Quick Reference

| Symbol | Signature | Description |
|---|---|---|
| `some` | `some(T)` → `Option[T]` | Wrap a value — creates a non-empty option |
| `none` | `none(typedesc)` or `none[T]()` → `Option[T]` | Create an empty option |
| `option` | `option(T)` → `Option[T]` | Smart constructor: nil → none, otherwise some |
| `isSome` | `Option[T]` → `bool` | True if the option contains a value |
| `isNone` | `Option[T]` → `bool` | True if the option is empty |
| `get` | `Option[T]` → `lent T` | Extract value; raises `UnpackDefect` if none |
| `get` | `Option[T], T` → `T` | Extract value or return a default |
| `get` | `var Option[T]` → `var T` | Extract mutable reference |
| `unsafeGet` | `Option[T]` → `lent T` | Extract without checking (undefined if none) |
| `map` | `Option[T], (T→void)` → `void` | Run a side-effect if some |
| `map` | `Option[T], (T→R)` → `Option[R]` | Transform the contained value |
| `flatMap` | `Option[T], (T→Option[R])` → `Option[R]` | Chain operations that may fail |
| `flatten` | `Option[Option[T]]` → `Option[T]` | Remove one layer of nesting |
| `filter` | `Option[T], (T→bool)` → `Option[T]` | Keep value only if predicate is true |
| `==` | `Option[T], Option[T]` → `bool` | Equality: both none, or both some with equal values |
| `$` | `Option[T]` → `string` | Human-readable representation |
