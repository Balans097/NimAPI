# std/outparams — Module Reference

> **Part of Nim's standard library** (`std/outparams`).

## Overview

This module provides a single macro — `outParamsAt` — whose entire purpose is **cross-version compatibility**: it lets you annotate procedure parameters as `out` in a way that compiles correctly on both Nim 2.x (which introduced the `out` parameter kind) and Nim 1.x (which does not know about `out` at all).

### The problem it solves

Nim 2.0 introduced a distinction between `var` parameters and `out` parameters:

- A **`var` parameter** says "this argument may be read *and* written by the procedure."
- An **`out` parameter** says "this argument will be *only written* by the procedure — its value before the call is irrelevant and the caller must not read it before the procedure returns."

The `out` distinction matters especially when the experimental `strictDefs` mode is active: the compiler can prove that a variable is definitely initialized after a call that takes it as `out`, allowing you to write:

```nim
{.experimental: "strictDefs".}
var x: int           # not yet initialized — reading x here is a compile error
myProc(x)            # x is an out-parameter, so after this call x is initialized
echo x               # OK: compiler knows x was set by myProc
```

The trouble is that writing `proc myProc(x: out int)` in source code **does not compile on Nim 1.x**, which has no `out` type constructor at all. The `outParamsAt` macro bridges this gap: on Nim 1.x it leaves every parameter as `var` (the closest available approximation), and on Nim 2.x it silently rewrites the selected parameters from `var T` to `out T` at compile time.

---

## Macro

### `outParamsAt`

```nim
macro outParamsAt*(positions: static openArray[int]; n: untyped): untyped
```

**Annotates selected `var` parameters of a procedure as `out` parameters in a version-portable manner.**

The macro is applied as a pragma on a procedure declaration. It takes one argument — an array of 1-based parameter positions — and rewrites those parameters from `var T` to `out T` on compilers that support `out`, leaving the declaration completely unchanged on compilers that do not.

#### Parameters

| Parameter | Type | Description |
|---|---|---|
| `positions` | `static openArray[int]` | 1-based indices of the parameters to rewrite. Index `1` refers to the first parameter, `2` to the second, and so on. Index `0` is the return type — never pass `0` here. |
| `n` | `untyped` | The procedure declaration the pragma is attached to. Provided automatically by the Nim pragma machinery; you never write this yourself. |

#### Return value

The (possibly rewritten) procedure declaration AST node. Because it is a pragma macro that returns `untyped`, it is transparent: from the outside the procedure looks and behaves normally.

#### How the rewriting works

Internally the macro inspects the compile-time flag `nimHasOutParams` (set automatically by Nim 2.x):

- **On Nim 1.x** (`nimHasOutParams` is *not* defined): the macro simply returns the original node unchanged. Every annotated parameter stays `var T`.
- **On Nim 2.x** (`nimHasOutParams` *is* defined): for each index in `positions`, the macro locates the corresponding parameter node in the AST, verifies it is currently typed as `var T`, and replaces the `nnkVarTy` node with an `nnkOutTy` node wrapping the same inner type `T`. The result is as if you had written `out T` directly in the source.

#### Usage pattern

```nim
import std/outparams

proc myProc(result1: var int; result2: var string) {.outParamsAt: [1, 2].} =
  result1 = 42
  result2 = "hello"
```

Only the parameters at positions `1` and `2` are candidates for rewriting; any other parameters left out of the list remain `var` (or whatever they were).

---

## Examples

### Basic — single out parameter

```nim
import std/outparams

proc parseInt(s: string; value: var int): bool {.outParamsAt: [2].} =
  ## Parses `s` as an integer, writing the result into `value`.
  ## `value` is an out parameter: its previous content does not matter.
  try:
    value = system.parseInt(s)
    result = true
  except ValueError:
    result = false
```

On Nim 2.x this compiles to the equivalent of `proc parseInt(s: string; value: out int): bool`. On Nim 1.x it compiles to the original `var int` signature — the behaviour is the same either way, but the Nim 2.x compiler can additionally prove that `value` is set on the `true` branch.

---

### Multiple out parameters

```nim
import std/outparams

proc splitPath(full: string; dir: var string; name: var string) {.outParamsAt: [2, 3].} =
  ## Splits a file path into its directory and filename components.
  ## Both `dir` and `name` are write-only outputs; they need not be
  ## initialized before the call.
  let slash = full.rfind('/')
  if slash < 0:
    dir = ""
    name = full
  else:
    dir = full[0..slash]
    name = full[slash+1..^1]

var d, n: string
splitPath("/home/user/file.txt", d, n)
echo d   # "/home/user/"
echo n   # "file.txt"
```

---

### Interaction with `strictDefs`

The most compelling reason to use `outParamsAt` is to work seamlessly with `strictDefs`, which enforces that every variable is definitely initialized before use:

```nim
{.experimental: "strictDefs".}
import std/outparams

proc computeHash(data: string; hash: var uint64) {.outParamsAt: [2].} =
  hash = 0xcbf29ce484222325'u64
  for ch in data:
    hash = hash xor uint64(ord(ch))
    hash *= 0x100000001b3'u64

proc process(data: string) =
  var h: uint64          # uninitialized — strictDefs tracks this
  computeHash(data, h)   # after this call, h is provably initialized (out param)
  echo h                 # OK on Nim 2.x; no "possibly uninitialized" warning
```

Without `outParamsAt`, `h` would be typed as `var uint64`, and `strictDefs` could not prove it was initialized by the call — it would require you to write `var h: uint64 = 0` as a workaround.

---

### What happens on each compiler version

```nim
import std/outparams

# Source code you write:
proc fill(x: var int) {.outParamsAt: [1].} =
  x = 99

# On Nim 1.x — compiled as:
proc fill(x: var int) = x = 99      # unchanged

# On Nim 2.x — compiled as:
proc fill(x: out int) = x = 99      # var → out rewrite applied
```

Both compile and run identically; the difference is only in what the type system can *prove* about callers.

---

## Important constraints

**The annotated parameters must already be `var T`.**  
The macro calls `expectKind nnkVarTy` on each listed parameter before rewriting. If you accidentally list a non-`var` parameter, you will get a compile-time error explaining the mismatch.

**Positions are 1-based.**  
Parameter index `1` is the first parameter in the procedure's formal parameter list. This aligns with Nim's `params` AST convention where index `0` is the return type. Passing `0` would attempt to rewrite the return type, which is nonsensical — never do this.

**Only applies to the declaration, not the body.**  
`outParamsAt` rewrites the type in the signature. Inside the procedure body you still use the parameter as if it were `var` — you can assign to it freely. The `out` distinction only affects what the *caller's* compiler can conclude about initialization.

---

## Summary

| Aspect | Detail |
|--------|--------|
| **What it is** | A compatibility pragma-macro |
| **Effect on Nim 1.x** | None — procedure is left exactly as written |
| **Effect on Nim 2.x** | Selected `var T` parameters are rewritten to `out T` |
| **Argument format** | 1-based array of parameter positions, e.g. `[1]`, `[2, 3]` |
| **Requirements** | Listed parameters must be `var T` in the source |
| **Main benefit** | Enables `strictDefs` initialization proofs without sacrificing 1.x compatibility |

---

> **See also:** Nim manual section on [out parameters](https://nim-lang.org/docs/manual.html) and the [`strictDefs` experimental feature](https://nim-lang.org/docs/manual_experimental.html).
