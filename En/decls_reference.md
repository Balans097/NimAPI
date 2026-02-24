# `decls` Module Reference

> **Module purpose:** `decls` provides syntax sugar for declarations that are cumbersome to express in plain Nim. At the moment the module contains one macro — `byaddr` — which brings C++-style l-value references into Nim. More ergonomic declaration forms may be added here in the future.

---

## Table of Contents

1. [Background: what is an l-value reference?](#background-what-is-an-l-value-reference)
2. [byaddr](#byaddr)
3. [How byaddr works internally](#how-byaddr-works-internally)
4. [Limitations and stability](#limitations-and-stability)
5. [Practical patterns](#practical-patterns)

---

## Background: what is an l-value reference?

In most systems languages, a **reference** (or *alias*) is a name that refers directly to an existing memory location rather than holding its own copy of the data. Modifying the reference modifies the original.

In C++ this is written as:
```cpp
auto& a = someArray[0];
a += 100;  // modifies someArray[0] in place
```

Nim does not have a built-in reference-declaration syntax for local variables. The usual workarounds — passing `var` parameters, using explicit pointers — are either impossible in certain scopes or syntactically heavy. `byaddr` fills this gap by generating the pointer machinery automatically, hiding it behind a clean declaration syntax.

---

## `byaddr`

```nim
macro byaddr*(sect)
```

### What it does

`byaddr` is a **variable pragma macro** that turns a `let` or `var` binding into an alias — a name that transparently reads from and writes to the memory of the expression on the right-hand side. After the declaration, every use of the new name is exactly equivalent to accessing the original location directly.

The declaration syntax mirrors a normal Nim variable declaration, with `{.byaddr.}` placed between the name and the `=`:

```nim
var a {.byaddr.} = someExpression
```

The type annotation is optional but supported:

```nim
var b {.byaddr.}: int = someExpression
```

### Key properties

| Property | Behaviour |
|---|---|
| Modifying the alias | Modifies the original memory location |
| Type of the alias | The element type of the expression (not a pointer) |
| Address of the alias | Identical to the address of the original |
| Original expression | Evaluated once, at the point of declaration |
| Works with | Any l-value: array elements, sequence elements, object fields, dereferenced pointers |

### Usage syntax

```nim
import std/decls

# Form 1: type inferred
var a {.byaddr.} = expression

# Form 2: explicit type annotation
var b {.byaddr.}: SomeType = expression
```

> `byaddr` must appear as a pragma on a single variable declaration — not on a multi-binding `var` block and not on a `const` or `type` section.

### Examples

#### Example 1 — Alias into a sequence element

The most direct demonstration: modifying `a` modifies `s[0]` because they share the same address.

```nim
import std/decls

var s = @[10, 11, 12]
var a {.byaddr.} = s[0]

a += 100

assert s == @[110, 11, 12]   # s[0] changed through the alias
assert a is int               # the alias has the element type, not a pointer type
```

#### Example 2 — Two aliases, same address

Both `a` and `b` refer to `s[0]`, so their addresses are identical. This confirms that no copies are made.

```nim
import std/decls

var s = @[10, 11, 12]
var a {.byaddr.} = s[0]
var b {.byaddr.}: int = s[0]

assert a.addr == b.addr   # both point at s[0]
```

#### Example 3 — Alias into an object field

`byaddr` is not limited to sequences. It works with any l-value, including struct fields.

```nim
import std/decls

type Point = object
  x, y: float

var p = Point(x: 1.0, y: 2.0)
var px {.byaddr.} = p.x

px = 99.0
assert p.x == 99.0   # field was modified through the alias
```

#### Example 4 — Alias into a nested structure

Useful for avoiding repeated deep indexing in a tight block of code.

```nim
import std/decls

var grid = @[@[1, 2, 3], @[4, 5, 6]]

var cell {.byaddr.} = grid[1][2]   # alias to the element with value 6
cell = 999
assert grid[1][2] == 999
```

#### Example 5 — Alias with explicit type annotation

The optional type annotation is useful when the type should be explicit for readability or when you want the compiler to verify the expected type at the declaration site.

```nim
import std/decls

var values = [10.0, 20.0, 30.0]
var first {.byaddr.}: float = values[0]

first = 0.5
assert values[0] == 0.5
```

#### Example 6 — Avoiding repeated indexing in a loop body

A common real-world use: create a short alias to a deeply nested element at the top of a loop body to keep the rest of the body readable.

```nim
import std/decls

type Config = object
  thresholds: seq[int]

var cfg = Config(thresholds: @[10, 20, 30, 40])

for i in 0 ..< cfg.thresholds.len:
  var t {.byaddr.} = cfg.thresholds[i]
  t = t * 2   # modifies cfg.thresholds[i] directly

assert cfg.thresholds == @[20, 40, 60, 80]
```

---

## How `byaddr` works internally

Understanding the generated code demystifies the macro and explains both its power and its constraints.

When you write:
```nim
var a {.byaddr.} = s[0]
```

The macro expands this to approximately:
```nim
let tmp: ptr typeof(s[0]) = addr(s[0])
template a: untyped = tmp[]
```

Two things are happening:

1. **A `ptr` is captured.** `addr(s[0])` takes the address of the expression and stores it in an immutable `let` binding `tmp`. The expression is evaluated exactly once here. The type of `tmp` is `ptr T` where `T` is the element type.

2. **A template becomes the alias.** A zero-argument template named `a` is defined, which expands to `tmp[]` (dereferencing the pointer) every time `a` appears in code. Because it is a template — not a variable — it expands inline at every use site, making reads and writes transparently go through the pointer.

When you later write `a += 100`, the compiler sees `tmp[] += 100`, which is a direct write to the original memory. When you write `a.addr`, the compiler sees `addr(tmp[])`, which is identical to `addr(s[0])`.

The explicit-type form `var b {.byaddr.}: int = s[0]` generates:
```nim
let tmp: ptr int = addr(s[0])
template b: untyped = tmp[]
```

---

## Limitations and stability

### ⚠️ Experimental features

`byaddr` is built on two Nim features that are themselves marked experimental:

- **Nullary templates instantiated as symbols** — zero-argument templates that look and behave like variable names.
- **Variable macro pragmas** — the ability to use a macro as a pragma on a variable declaration.

Because of this, the behaviour of `byaddr` is **not guaranteed to be stable** across Nim versions. It works reliably in practice, but you should be aware of the dependency on experimental internals when deciding whether to use it in long-lived or widely distributed code.

### Unintended redefinition

The current implementation allows the same name to be declared with `{.byaddr.}` more than once in the same scope. This is an unintended side-effect of the template-based implementation, not a deliberate feature. Do not rely on it.

### r-values are not supported

`byaddr` requires an **l-value** on the right-hand side — an expression that has a stable memory address. Passing a temporary or an r-value (the result of a function that returns by value, a literal, an arithmetic expression) will fail at the `addr()` call:

```nim
# ❌ Will not compile — integer literal has no address
var x {.byaddr.} = 42

# ❌ Will not compile — function result is a temporary
var y {.byaddr.} = someProc()

# ✅ Works — array element is an l-value
var arr = [1, 2, 3]
var z {.byaddr.} = arr[0]
```

### Not a language-level reference

The alias created by `byaddr` is a template, not a first-class reference type. You cannot store it in a data structure, return it from a function, or pass it as a `var` parameter. Its scope is strictly lexical — it exists only in the block where it is declared.

### Single declaration only

The macro expects exactly one variable declaration per pragma invocation. Multi-name `var` blocks with a single `{.byaddr.}` are not supported.

---

## Practical patterns

### Pattern 1 — Clean in-place mutation of a complex location

```nim
import std/decls

type Matrix = seq[seq[float]]

proc normalize(m: var Matrix, row, col: int) =
  var cell {.byaddr.} = m[row][col]
  if cell < 0.0: cell = 0.0
  if cell > 1.0: cell = 1.0
  # No need to write m[row][col] each time
```

### Pattern 2 — Swapping aliases to compare two locations

```nim
import std/decls

var data = @[5, 3, 8, 1]

var a {.byaddr.} = data[0]
var b {.byaddr.} = data[2]

if a > b:
  let tmp = a   # copies the int value, not the alias
  a = b
  b = tmp

assert data == @[5, 3, 8, 1] or data[0] <= data[2]
```

### Pattern 3 — Readable access to deeply nested config

```nim
import std/decls

type
  Server = object
    host: string
    port: int
  AppConfig = object
    server: Server

var config = AppConfig(server: Server(host: "localhost", port: 8080))

block:
  var host {.byaddr.} = config.server.host
  var port {.byaddr.} = config.server.port

  host = "0.0.0.0"
  port = 443

assert config.server.host == "0.0.0.0"
assert config.server.port == 443
```
