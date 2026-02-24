# genasts — Module Reference

## Overview

The `std/genasts` module provides a cleaner, safer alternative to `quote do:` for generating AST (Abstract Syntax Tree) nodes inside Nim macros. Its central idea is **explicit variable capture**: instead of relying on hygiene mechanisms that can be fragile or surprising, you declare exactly which runtime values from the macro body should be baked into the generated code, and under what names they should appear there.

### Why this exists — the problem with `quote do:`

When writing macros, the most common tool for producing AST is `quote do:`. It works well for simple cases, but has a well-known pitfall: any local variable in the macro body that is mentioned inside the `quote do:` block is captured *implicitly* via backtick substitution (`` `varName` ``). Forgetting a backtick means the compiler looks for a symbol with that name in the *caller's* scope — which may silently succeed with the wrong symbol, or fail with a confusing error.

`genAst` makes capture explicit and mandatory. You cannot accidentally refer to a macro-local variable inside the template body without first listing it in the capture list. This makes macro code easier to reason about and less prone to subtle hygiene bugs.

### How `genAst` works internally

`genAst` is a macro that, given a capture list and a body block, constructs a **local template** whose parameters correspond to the captured variables, immediately calls it with `getAst`, and returns the resulting `NimNode`. The template approach is what allows value capture without the fragility of `quote do:`.

---

## `GenAstOpt`

```nim
type GenAstOpt* = enum
  kDirtyTemplate
  kNoNewLit
```

### Description

An enum type used as a `static set[GenAstOpt]` argument to `genAstOpt`. Each value enables a specific behaviour modification. The set is empty by default (via `genAst`), which is the recommended setting for most use cases.

### Values

**`kDirtyTemplate`**

By default `genAst` generates a *hygienic* template (the default for templates in Nim). This means symbols introduced inside the template body are automatically `gensym`'d and cannot accidentally shadow names in the caller's scope — the safe choice.

Setting `kDirtyTemplate` switches to a *dirty* template (`{.dirty.}`), which does **not** apply automatic hygienic renaming. Symbols in the template body are resolved in the *caller's* scope rather than the template's own scope. This is occasionally necessary as a workaround for edge-case compiler bugs (for example, issue #8220 relating to `strformat`), but should be avoided in general because it reintroduces the exact hygiene hazards that `genAst` was designed to prevent.

Use `kDirtyTemplate` only when you have a specific, documented reason for needing dirty semantics.

**`kNoNewLit`**

By default, when you capture a variable `a` (or `x = expr`), `genAst` automatically wraps the captured value in `newLit(…)` (specifically in the internal `newLitMaybe` helper, which skips `newLit` for types, procs, iterators, funcs, and `NimNode` values). This converts the runtime value into a compile-time literal node, which is what you almost always want when generating code that embeds a value.

Setting `kNoNewLit` disables this wrapping. The captured expression is passed to the template as-is, without any `newLit` conversion. Use this when the captured value is already a `NimNode` or when you need to pass a symbol reference rather than a literal.

---

## `genAstOpt`

```nim
macro genAstOpt*(options: static set[GenAstOpt], args: varargs[untyped]): untyped
```

### Description

The core macro of this module. Accepts a compile-time set of options, a list of capture specifications, and a final block of Nim code, and returns a `NimNode` representing that code with the captured values substituted in.

In practice most code uses the `genAst` convenience wrapper instead of calling `genAstOpt` directly.

### Capture list syntax

Each entry in the capture list (all arguments except the final block) may be one of two forms:

| Form | Meaning |
|---|---|
| `a` | Capture the macro-local variable `a` under the same name `a` in the generated code |
| `a = expr` | Evaluate `expr` in the macro body and make it available as `a` in the generated code |

Unless `kNoNewLit` is set, each captured value is passed through `newLit` (via `newLitMaybe`) to convert it to an AST literal node. This is skipped automatically for types, proc/func/iterator values, and `NimNode` values.

### What is automatically available without capture

- **`{.inject.}` procs and templates** defined locally in the macro body are visible inside the `genAst` block without capture, because they are naturally injected into the caller's scope by Nim's template mechanism.
- **`{.gensym.}` variables** (all `var`, `let`, and `const` bindings inside a macro body are implicitly `gensym`'d) must be captured explicitly to be usable inside the block.

### Parameters

| Name | Type | Description |
|---|---|---|
| `options` | `static set[GenAstOpt]` | Set of option flags; use `{}` for defaults |
| `args` | `varargs[untyped]` | Capture list entries followed by the body block as the last argument |

### Return value

A `NimNode` — the AST of the body block with captured values substituted.

### Examples

**Minimal example — no captures, pure code generation:**

```nim
import std/[macros, genasts]

macro addOne(x: typed): untyped =
  genAst(x):
    x + 1

assert addOne(41) == 42
```

**Capturing a macro-local computed value:**

```nim
import std/[macros, genasts]

macro doubled(x: typed): untyped =
  # We compute something inside the macro and want to bake it into the output.
  let factor = 2
  genAst(x, factor):
    x * factor   # `factor` here is 2, baked in as a literal

assert doubled(21) == 42
```

**Renaming on capture (`a = expr` form):**

```nim
import std/[macros, genasts]

macro check2(cond: bool): untyped =
  # cond is the entire expression (e.g. `a * 2 == a + 3`)
  # We capture it under its own name, plus extract LHS and RHS separately.
  assert cond.kind == nnkInfix
  result = genAst(cond, s = repr(cond), lhs = cond[1], rhs = cond[2]):
    if not cond:
      raiseAssert "'$#' failed: lhs='$#', rhs='$#'" % [s, $lhs, $rhs]

let a = 3
check2 a * 2 == a + 3   # passes
# check2 a * 2 < a + 1  # would print: 'a * 2 < a + 1' failed: lhs='6', rhs='4'
```

**Forwarding a `static` macro parameter as a static value:**

```nim
import std/[macros, genasts]

macro makeConst(val: static int): untyped =
  genAst(val):
    const generated = val   # val is baked in as a literal integer
    generated

assert makeConst(99) == 99
```

**Using `kDirtyTemplate` as a workaround (advanced):**

```nim
import std/[macros, genasts]

macro withDirty(x: typed): untyped =
  genAstOpt({kDirtyTemplate}, x):
    # Dirty mode: names resolve in caller scope.
    # Use only when you have a specific hygiene reason.
    x + 1

assert withDirty(10) == 11
```

**Using `kNoNewLit` when the captured value is already a `NimNode`:**

```nim
import std/[macros, genasts]

macro wrapNode(body: untyped): untyped =
  # body is already a NimNode; we do not want newLit wrapping.
  genAstOpt({kNoNewLit}, body):
    body   # passed through as-is, not wrapped in newLit

wrapNode(echo "hello")
```

---

## `genAst`

```nim
template genAst*(args: varargs[untyped]): untyped
```

### Description

A convenience wrapper around `genAstOpt` that uses an empty options set (`{}`). This is the version you should reach for by default. It uses a hygienic template (not dirty), and automatically wraps captured values in `newLit` via `newLitMaybe`.

The call `genAst(captures..., body)` is exactly equivalent to `genAstOpt({}, captures..., body)`.

### Parameters

| Name | Type | Description |
|---|---|---|
| `args` | `varargs[untyped]` | Capture list entries followed by the body block |

### Return value

A `NimNode` — the AST of the body block with captured values substituted.

### Examples

**A practical reimplementation of `assert` for illustration:**

```nim
import std/[macros, genasts]

macro myAssert(cond: bool): untyped =
  genAst(cond, msg = "Assertion failed: " & repr(cond)):
    if not cond:
      raise newException(AssertionDefect, msg)

let x = 5
myAssert x > 3   # passes
# myAssert x > 10  # raises AssertionDefect: "Assertion failed: x > 10"
```

**Generating a typed wrapper procedure:**

```nim
import std/[macros, genasts]

macro makeLogger(prefix: static string): untyped =
  genAst(prefix):
    proc log(msg: string) =
      echo prefix, ": ", msg

makeLogger("INFO")
log("Server started")   # prints: INFO: Server started
```

**Capturing multiple values with renaming:**

```nim
import std/[macros, genasts]

macro between(val, lo, hi: typed): untyped =
  genAst(val, lo, hi, errMsg = "value out of range"):
    if val < lo or val > hi:
      raise newException(RangeDefect, errMsg)
    val

let result = between(5, 1, 10)   # 5 is in range, returned
# between(15, 1, 10)             # raises RangeDefect
```

---

## `genAst` vs `quote do:` — a comparison

| Aspect | `quote do:` | `genAst` |
|---|---|---|
| Variable capture | Implicit via `` `varName` `` backtick syntax | Explicit via capture list |
| Forgetting to capture | Resolves in caller scope — may silently use wrong symbol | Compile-time error: symbol not found |
| Hygiene | Hygienic by default | Hygienic by default |
| Embedding a computed value | `` `val` `` where `val` is a `NimNode` | `val` in capture list (auto-`newLit`'d) |
| Embedding a literal value | Must manually call `newLit` and assign to a `NimNode` | Automatic via `newLitMaybe` |
| Local inject procs | Accessible without backtick | Accessible without capture |
| Clarity of intent | Low (any local var accessible via backtick) | High (capture list is an explicit contract) |

### When to use each

- Use **`genAst`** when you are building new macros, when the captured values are plain Nim values (ints, strings, booleans, etc.) that should be embedded as literals, or when you want the safest and most readable macro code.
- Use **`quote do:`** for quick, simple AST templates where you are working primarily with `NimNode` values already (no auto-`newLit` needed) and the capture semantics are clear.
- Use **`genAstOpt` with `kNoNewLit`** when the values you capture are `NimNode`s themselves and you do not want `newLit` wrapping, but still want the hygienic template structure of `genAst`.

---

## Summary table

| Symbol | Kind | Description |
|---|---|---|
| `GenAstOpt` | enum | Option flags for `genAstOpt` |
| `kDirtyTemplate` | enum value | Use a dirty (non-hygienic) template internally |
| `kNoNewLit` | enum value | Skip automatic `newLit` wrapping of captured values |
| `genAstOpt(options, args…)` | macro | Full-featured AST generator with explicit options |
| `genAst(args…)` | template | Convenience wrapper: `genAstOpt({}, args…)` |
