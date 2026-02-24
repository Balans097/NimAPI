# effecttraits — Module Reference

## Overview

The `effecttraits` module gives macro authors **programmatic access to the effect system metadata** that the Nim compiler infers for every procedure, function, converter, iterator, or method. With it you can ask, at compile time and inside a macro body, questions like:

- *Which exceptions can this function raise?*
- *Which effect tags does it carry?*
- *Which tags does it explicitly forbid?*
- *Is it GC-safe?*
- *Does it have no side effects?*

This unlocks a family of powerful meta-programming patterns: generating wrapper code that automatically propagates effects, enforcing custom effect policies, building introspection utilities, and more — all without modifying the procedures being inspected.

**Availability:** Nim 1.4 and later. You can guard code at the call site with:

```nim
when defined(nimHasEffectTraitsModule):
  import std/effecttraits
```

**Critical constraint shared by all five procedures:** The `fn` argument must be a **resolved symbol node** (`nnkSym`). This means the macro that calls any of these procedures must accept **`typed`** parameters, not `untyped` ones. A `typed` parameter tells the compiler to resolve the argument — look up its symbol in the symbol table — before passing it to the macro. Passing an unresolved `untyped` node will cause a compile-time assertion failure.

**Backend note:** All five procedures delegate to compiler-internal VM operations defined in `compiler/vmops.nim`. Their Nim-level bodies are intentionally unreachable stubs; the real implementation lives inside the compiler itself and is invoked only during macro expansion.

---

## `getRaisesList`

```nim
proc getRaisesList*(fn: NimNode): NimNode
```

### Description

Returns the **`.raises` effect list** that the compiler has inferred (or that was explicitly annotated) for the procedure `fn`. The returned node is a `NimNode` representing a bracketed list of exception types — the same data structure that appears in the procedure's pragma section when you write `.raises: [IOError, ValueError]`.

If the compiler has determined that `fn` raises nothing, the returned node is an empty list. If the raises effects are unconstrained (the procedure was never annotated and inference could not limit them), the list reflects that unconstrained state.

The primary use case is generating wrapper procedures that must carry the same `.raises` contract as the wrapped function, or asserting at compile time that a callback passed to a generic framework does not raise certain exceptions.

### Parameters

| Name | Type | Description |
|---|---|---|
| `fn` | `NimNode` | A resolved symbol node (`nnkSym`) of the procedure to inspect |

### Return value

A `NimNode` — specifically a node that can be embedded directly into a generated pragma or used for further AST inspection. Its exact shape mirrors what the compiler stores internally for `.raises` annotations.

### Example

```nim
import std/macros
import std/effecttraits

# A procedure with a constrained raises list
proc riskyOp() {.raises: [IOError].} =
  raise newException(IOError, "disk full")

# A procedure that raises nothing
proc safeOp() {.raises: [].} =
  discard

macro inspectRaises(fn: typed): untyped =
  let raises = getRaisesList(fn)
  echo "raises list AST: ", raises.treeRepr
  result = newEmptyNode()

inspectRaises(riskyOp)
# Output (at compile time):
# raises list AST: BracketExpr
#   Sym "IOError"

inspectRaises(safeOp)
# Output (at compile time):
# raises list AST: BracketExpr   ← empty bracket, no children
```

A more practical use — asserting that a callback is safe before embedding it in a critical section:

```nim
macro requireNoRaises(fn: typed): untyped =
  let raises = getRaisesList(fn)
  if raises.len > 0:
    error("Callback must not raise any exceptions, but raises: " & raises.repr, fn)
  result = newEmptyNode()

proc safe() {.raises: [].} = discard
proc unsafe() {.raises: [IOError].} = discard

requireNoRaises(safe)    # OK
requireNoRaises(unsafe)  # Compile-time error
```

---

## `getTagsList`

```nim
proc getTagsList*(fn: NimNode): NimNode
```

### Description

Returns the **`.tags` effect list** associated with `fn`. Tags are user-defined or standard effect markers that label a procedure as having certain observable behaviours — for example `ReadIOEffect`, `WriteIOEffect`, `ReadDirEffect`, or any custom tag type you define. They are declared with `.tags: [TagA, TagB]` on a procedure.

Tags serve as a documentation and enforcement mechanism: a caller annotated with a restricted tag set cannot call a procedure that introduces a tag not in that set. `getTagsList` lets macros inspect this set and make decisions based on it.

### Parameters

| Name | Type | Description |
|---|---|---|
| `fn` | `NimNode` | A resolved symbol node (`nnkSym`) of the procedure to inspect |

### Return value

A `NimNode` containing the list of tag types, analogous in structure to the `.raises` list.

### Example

```nim
import std/macros
import std/effecttraits

type ReadOnlyEffect = object  # custom tag

proc readConfig() {.tags: [ReadIOEffect].} =
  discard

proc pureCompute() {.tags: [].} =
  discard

macro checkTags(fn: typed): untyped =
  let tags = getTagsList(fn)
  echo fn.repr, " has ", tags.len, " tag(s): ", tags.repr
  result = newEmptyNode()

checkTags(readConfig)
# readConfig has 1 tag(s): [ReadIOEffect]

checkTags(pureCompute)
# pureCompute has 0 tag(s): []
```

A practical pattern — enforcing that only "read-only" procedures are registered in a read-only handler table:

```nim
macro registerReadOnly(fn: typed): untyped =
  let tags = getTagsList(fn)
  for tag in tags:
    if tag.repr == "WriteIOEffect":
      error("Read-only handler must not have WriteIOEffect", fn)
  # proceed with registration code generation...
  result = newEmptyNode()
```

---

## `getForbidsList`

```nim
proc getForbidsList*(fn: NimNode): NimNode
```

### Description

Returns the **`.forbids` effect list** of `fn`. The `.forbids` pragma is the counterpart to `.tags`: while `.tags` declares which effects a procedure *introduces*, `.forbids` declares which effects it *refuses to allow in its callees*. A procedure annotated with `.forbids: [WriteIOEffect]` will cause the compiler to error if it calls any procedure that carries `WriteIOEffect` in its tag list.

`getForbidsList` lets macro code read this list and make meta-decisions: for instance, verifying that a callback being wrapped obeys the same restrictions as the wrapper's own policy.

### Parameters

| Name | Type | Description |
|---|---|---|
| `fn` | `NimNode` | A resolved symbol node (`nnkSym`) of the procedure to inspect |

### Return value

A `NimNode` listing the forbidden effect types.

### Example

```nim
import std/macros
import std/effecttraits

proc strictReader() {.forbids: [WriteIOEffect].} =
  discard

proc relaxedReader() =
  discard

macro showForbids(fn: typed): untyped =
  let forbids = getForbidsList(fn)
  echo fn.repr, " forbids: ", forbids.repr
  result = newEmptyNode()

showForbids(strictReader)
# strictReader forbids: [WriteIOEffect]

showForbids(relaxedReader)
# relaxedReader forbids: []
```

Combined inspection — checking that a plug-in function is compatible with a host that forbids writes:

```nim
macro ensureCompatible(plugin: typed): untyped =
  # Host forbids WriteIOEffect; plugin must not introduce it
  let tags = getTagsList(plugin)
  for tag in tags:
    if tag.repr == "WriteIOEffect":
      error("Plugin introduces WriteIOEffect, which the host forbids", plugin)
  result = newEmptyNode()
```

---

## `isGcSafe`

```nim
proc isGcSafe*(fn: NimNode): bool
```

### Description

Returns `true` if the compiler has determined that `fn` is **GC-safe** — meaning it does not access any GC-managed heap memory through global or closure variables in a way that would be unsafe in a multi-threaded context. This corresponds to the `{.gcsafe.}` pragma.

GC safety is a prerequisite for passing a procedure to a new thread via `createThread` or `spawn`. Inspecting this property in a macro lets you gate thread-related code generation on whether the supplied callback is actually safe to hand to another thread, producing a clear compile-time diagnostic rather than a linker or runtime error.

### Parameters

| Name | Type | Description |
|---|---|---|
| `fn` | `NimNode` | A resolved symbol node (`nnkSym`) of the procedure to inspect |

### Return value

`true` if `fn` is GC-safe, `false` otherwise.

### Example

```nim
import std/macros
import std/effecttraits

proc threadSafe() {.gcsafe.} =
  discard

proc notThreadSafe() =
  var x = @[1, 2, 3]  # captures GC heap; not gcsafe in all contexts
  discard x

macro spawnSafe(fn: typed): untyped =
  if not isGcSafe(fn):
    error("Cannot spawn: procedure is not GC-safe", fn)
  # generate spawn call...
  result = newEmptyNode()

spawnSafe(threadSafe)    # OK
# spawnSafe(notThreadSafe) # ← compile-time error
```

This is especially useful in frameworks that abstract over `createThread` or work with thread pools — you can reject unsafe callbacks early with a descriptive message rather than letting the compiler's own less-targeted error surface.

---

## `hasNoSideEffects`

```nim
proc hasNoSideEffects*(fn: NimNode): bool
```

### Description

Returns `true` if `fn` is marked or inferred as having **no side effects** — that is, for the same inputs it always produces the same output and does not modify any observable external state. This corresponds to the `{.noSideEffect.}` pragma and is what makes a `func` different from a `proc` in Nim.

Knowing at macro-expansion time whether a function is pure opens up optimisation and correctness guarantees: you might generate a memoised wrapper only for pure functions, enforce that mathematical combinators receive only pure arguments, or produce a compile-time table of values for pure functions over a small domain.

### Parameters

| Name | Type | Description |
|---|---|---|
| `fn` | `NimNode` | A resolved symbol node (`nnkSym`) of the procedure to inspect |

### Return value

`true` if `fn` has no side effects, `false` otherwise.

### Example

```nim
import std/macros
import std/effecttraits

func square(x: int): int = x * x          # func → noSideEffect
proc printSquare(x: int) = echo x * x     # proc → has side effects

macro memoize(fn: typed): untyped =
  if not hasNoSideEffects(fn):
    error("memoize requires a pure (noSideEffect) function", fn)
  # generate memoization wrapper...
  result = newEmptyNode()

memoize(square)       # OK — square is pure
# memoize(printSquare) # ← compile-time error
```

Another use — generating a compile-time lookup table by calling a pure function across a range of inputs:

```nim
macro buildTable(fn: typed): untyped =
  if not hasNoSideEffects(fn):
    error("Table generation requires a pure function", fn)
  # Safe to call fn at compile time for each input value and bake results
  # into a constant array literal.
  result = newEmptyNode()
```

---

## The `typed` vs `untyped` requirement — a deeper look

Every procedure in this module calls `expectKind fn, nnkSym` as its first action. This assertion fires if `fn` is anything other than a resolved symbol node. The only reliable way to guarantee a resolved symbol is to declare the macro parameter as `typed`:

```nim
# CORRECT: fn is resolved before the macro body runs
macro inspect(fn: typed): untyped =
  echo getRaisesList(fn).repr
  result = newEmptyNode()

# WRONG: fn arrives as a raw AST identifier, not a symbol
macro inspect(fn: untyped): untyped =
  echo getRaisesList(fn).repr  # ← runtime assertion failure during macro expansion
  result = newEmptyNode()
```

The `typed` designation causes the compiler to perform **semantic analysis** on the argument — name resolution, overload resolution, type checking — before the macro sees it. The result is a `nnkSym` node that carries a full, compiler-populated symbol entry, from which the effect information can be read.

---

## Summary table

| Procedure | Returns | What it answers |
|---|---|---|
| `getRaisesList(fn)` | `NimNode` (list) | Which exceptions can `fn` raise? |
| `getTagsList(fn)` | `NimNode` (list) | Which effect tags does `fn` carry? |
| `getForbidsList(fn)` | `NimNode` (list) | Which effect tags does `fn` forbid in callees? |
| `isGcSafe(fn)` | `bool` | Is `fn` safe to call from another thread? |
| `hasNoSideEffects(fn)` | `bool` | Is `fn` a pure function? |

All five require `fn` to be a `nnkSym` node, obtainable only through a `typed` macro parameter.
