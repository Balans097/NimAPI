# `wrapnils` Module Reference

> Safe navigation through nil pointers and case-object fields — without explicit nil checks.  
> **Status:** Experimental module, unstable API.

---

## The problem `wrapnils` solves

Deeply nested access to reference types in Nim requires defensive nil checks at every level:

```nim
# Without wrapnils — verbose nil-guard ladder
var result: string
if obj != nil and obj.child != nil and obj.child.inner != nil:
  result = obj.child.inner.name
```

With case objects the situation is similar: accessing a field that belongs to a different discriminant branch raises a `FieldDefect` at runtime.

`wrapnils` provides two macros — `?.` and `??.` — that transparently wrap any chain of field accesses and pointer dereferences, catching both hazards at compile time with zero boilerplate.

---

## Exported symbols

---

### `?.` — safe navigation returning a default value

```nim
macro `?.`*(a: typed): auto
```

**What it does:** Transforms the expression `a` into a version that evaluates safely even when intermediate values in the access chain are `nil` or when a case-object field has the wrong discriminant. If any such condition is encountered the macro immediately stops evaluating the chain and returns `default(T)`, where `T` is the static type of `a`.

**Key properties:**

- The return type is exactly the same as the return type of the original expression `a`. No wrapper type is introduced.
- `default(T)` means: `""` for `string`, `0` for numeric types, `nil` for reference types, `false` for `bool`, etc.
- **lvalue semantics are preserved** — you can take the address of the result with `.addr` and write back through the pointer.
- The macro is a compile-time transformation; the generated code consists of ordinary `if`/`break` statements with no runtime overhead beyond the nil checks themselves.

**Syntax:** Place `?.` directly before the expression. Parentheses can be used to control the scope of nil-protection.

---

#### Basic nil safety

```nim
import std/wrapnils

type Node = ref object
  value: string
  next: Node

var head: Node   # nil

# Without wrapnils this would panic:
# echo head.next.value  # raises NilAccessDefect

echo ?.head.next.value   # → ""  (default(string))
```

---

#### Scoping with parentheses

By default `?.` protects the entire chain to its right. Use parentheses to narrow the protected scope — everything outside the parentheses is evaluated normally (and can therefore still raise):

```nim
import std/wrapnils

type Foo = ref object
  x1: string
  x2: Foo
  x3: ref int

var f2 = Foo(x1: "a")
f2.x2 = f2   # circular: f2.x2 points back to f2

# ?.() protects only f2.x2.x2 — the chain is guaranteed non-nil there
# The outer .x3[] is evaluated normally after the protected part resolves
assert ?.(f2.x2.x2).x3[] == 0

# Contrast: terminating ?. early at the closing parenthesis
# The result of (?.f2.x2.x2) is of type Foo (a ref), which may be nil
assert (?.f2.x2.x2).x3 == nil   # outer .x3 access not protected
```

---

#### Constructors inside the expression

Any valid Nim expression can appear as the argument:

```nim
import std/wrapnils

type Foo = ref object
  x1: string

assert ?.Foo(x1: "a").x1 == "a"   # constructor — never nil, works fine
```

---

#### lvalue semantics — writing through the safe result

Because `?.` preserves lvalue semantics you can obtain a mutable pointer and write through it:

```nim
import std/wrapnils

type B = object
  b0: int
  case cond: bool
  of false: discard
  of true:
    b1: float

var b = B(cond: true, b1: 4.5)

# Take address of b.b1 safely; p is nil if the field wasn't accessible
if (let p = ?.b.b1.addr; p != nil):
  p[] = 4.7

assert b.b1 == 4.7
```

---

#### Case-object field safety

Accessing a case-object field whose discriminant is set to a different branch raises `FieldDefect` in normal Nim. `?.` catches this at the first such access and returns `default(T)`:

```nim
import std/wrapnils

type B = object
  b0: int
  case cond: bool
  of false: discard
  of true:
    b1: float

var b = B(cond: false, b0: 3)

# Normal access raises:
# discard b.b1  # FieldDefect!

assert ?.b.b1 == 0.0   # safe — returns default(float)

b = B(cond: true, b1: 4.5)
assert ?.b.b1 == 4.5   # now the field is accessible, real value returned
```

---

#### The default-value ambiguity

`?.` always returns `default(T)` on failure. This is a limitation: if `default(T)` is also a legitimate value in your domain (e.g. `0` for an integer counter, `""` for a string that might genuinely be empty), you cannot tell from the return value alone whether the chain succeeded or failed. Use `??.` when that distinction matters.

```nim
import std/wrapnils

type Foo = ref object
  x1: ref int

var f1 = Foo(x1: int.new)   # x1 points to a zero-valued int

assert ?.f1.x1[] == 0   # succeeded, value is 0
# but also:
var f2: Foo
assert ?.f2.x1[] == 0   # failed (f2 is nil), default is also 0
# The two cases are indistinguishable with ?.
```

---

### `??.` — safe navigation returning an `Option`

```nim
macro `??.`*(a: typed): Option
```

**What it does:** Identical transformation to `?.`, but wraps the result in `std/options.Option[T]` instead of returning the bare value. Returns `some(value)` when the chain evaluated successfully, and `none(T)` when a nil or wrong-discriminant condition was encountered.

**When to use `??.` instead of `?.`:**

- When `default(T)` is a valid value in your domain and you need to distinguish "successfully got a zero" from "chain was broken".
- When the caller needs to decide whether to handle the absence explicitly (e.g. show an error, use a fallback).

```nim
import std/wrapnils, std/options

type Foo = ref object
  x1: ref int
  x2: int

var f1 = Foo(x1: int.new, x2: 2)   # x1 → 0, x2 = 2

# x1[] is 0 — valid, but indistinguishable from failure with ?.
assert (??.f1.x1[]).get == 0    # value is 0
assert (??.f1.x1[]).isSome      # but the chain was valid!

assert (??.f1.x2).get == 2

var f2: Foo   # nil
assert not (??.f2.x1[]).isSome  # f2 was nil → none
```

**Comparison:**

```nim
assert ?.f2.x1[] == 0              # ?.  — returns 0, ambiguous
assert not (??.f2.x1[]).isSome     # ??. — explicitly says "no value"
```

---

### `fakeDot` — chain `Option` values with field access

```nim
template fakeDot*(a: Option, b): untyped
```

**What it does:** Allows accessing a field `b` on an `Option`-wrapped value `a` while propagating `none` — analogous to optional chaining in other languages. If `a` is `none`, the result is `none` of the appropriate type. If `a` is `some` but the inner value is a nil reference, the result is also `none`. Otherwise returns `some(a.get.b)`.

**Why "fakeDot":** In an ideal world this would be spelled `a.b` directly on an `Option`; since Nim does not support that transparently, `fakeDot(a, b)` is the explicit spelling.

```nim
import std/wrapnils, std/options

type Inner = ref object
  value: int

type Outer = ref object
  inner: Inner

var o = Outer(inner: Inner(value: 42))
let optOuter = option(o)

let result = fakeDot(optOuter, inner)   # Option[Inner]
assert result.isSome
assert fakeDot(result, value).get == 42
```

---

### `[]` (index operator for `Option`) — two overloads

#### `[](a: Option[T], i: I): auto`

```nim
func `[]`*[T, I](a: Option[T], i: I): auto {.inline.}
```

**What it does:** Applies the index operator `[i]` to the value inside `a`. Returns `some(a.get[i])` if `a` is `some`; returns `none` if `a` is `none`. Importantly, if `a` is `some` but the container is empty, an `IndexDefect` is raised normally — this operator does not suppress bounds errors, only propagates absence.

```nim
import std/wrapnils, std/options

let s = option(@[10, 20, 30])
assert s[1].get == 20

let empty: Option[seq[int]] = none(seq[int])
assert not empty[0].isSome   # none propagates
```

---

#### `[](a: Option[U]): auto`

```nim
func `[]`*[U](a: Option[U]): auto {.inline.}
```

**What it does:** Dereferences the pointer or reference inside `a` (equivalent to `a.get[]`). Returns `some(a.get[])` if `a` is `some` and the inner pointer is not nil; returns `none` otherwise. Allows dereferencing nil-safe optional pointers without extra checks.

```nim
import std/wrapnils, std/options

var x = new int
x[] = 99
let optPtr = option(x)

assert optPtr[].get == 99   # dereferences through the Option

let nilPtr: ref int = nil
let optNil = option(nilPtr)
assert not optNil[].isSome  # inner pointer is nil → none
```

---

## Combining `??.` and the `[]`/`fakeDot` helpers

These helpers exist so that you can build longer chains on `Option` values after `??.` has introduced the `Option` wrapper:

```nim
import std/wrapnils, std/options

type Node = ref object
  data: seq[int]
  next: Node

var head = Node(data: @[1, 2, 3], next: Node(data: @[4, 5, 6]))

# Get the second element of head.next.data safely:
let result = (??.head.next.data)[1]
assert result.get == 5

var nilHead: Node
assert not ((??.nilHead.next.data)[1]).isSome
```

---

## `?.` vs `??.` — decision guide

| Situation | Use |
|-----------|-----|
| `default(T)` is never a valid domain value (e.g. `nil` pointer, always-positive count) | `?.` |
| `default(T)` can be a valid value; you need to tell success from failure | `??.` |
| You want to keep chaining field access on the result | `??.` + `fakeDot` + `[]` |
| You just want the value or a safe fallback with no `Option` overhead | `?.` |

---

## Quick cheat-sheet

| Symbol | Kind | Returns | Nil/FieldDefect | Notes |
|--------|------|---------|-----------------|-------|
| `?.expr` | macro | `T` (same type as `expr`) | `default(T)` | Ambiguous when `default(T)` is valid |
| `??.expr` | macro | `Option[T]` | `none(T)` | Unambiguous; requires `std/options` |
| `fakeDot(opt, field)` | template | `Option[FieldType]` | `none(...)` | Field access on an `Option` |
| `opt[i]` | func | `Option[ElemType]` | `none(...)` | Index into an `Option`-wrapped container |
| `opt[]` | func | `Option[DerefType]` | `none(...)` | Dereference an `Option`-wrapped pointer |
