# `assertions` Module Reference

> **Module purpose:** `assertions` provides the building blocks for runtime correctness checking in Nim. It gives you tools to declare invariants that must hold, to verify that certain code paths raise the exceptions you expect, and to customise what happens when an assertion fails — all with precise control over whether checks are compiled into your binary at all.

---

## Table of Contents

1. [raiseAssert](#raiseassert)
2. [failedAssertImpl](#failedassertimpl)
3. [assert](#assert)
4. [doAssert](#doassert)
5. [onFailedAssert](#onfailedassert)
6. [doAssertRaises](#doassertraises)
7. [Choosing the right tool](#choosing-the-right-tool)

---

## `raiseAssert`

```nim
proc raiseAssert*(msg: string) {.noinline, noreturn, nosinks.}
```

### What it does

The lowest-level entry point in the module. `raiseAssert` raises an `AssertionDefect` unconditionally, attaching `msg` as the human-readable reason. Execution never returns from this call — the `noreturn` pragma guarantees this, which lets the compiler reason that any code placed after a `raiseAssert` call is unreachable.

Being `noinline` means the function body is not duplicated at call sites, keeping binary size predictable even when many assertions are spread across the code base.

### When to use it directly

You will rarely call `raiseAssert` yourself. It is the common termination point that `assert`, `doAssert`, `failedAssertImpl`, and `doAssertRaises` all funnel into. However, you may call it explicitly when you want to mark a branch as logically impossible ("this should never happen") without wrapping it in a boolean condition:

```nim
import std/assertions

proc classify(x: int): string =
  if x > 0: "positive"
  elif x < 0: "negative"
  elif x == 0: "zero"
  else:
    # Mathematically unreachable, but the compiler doesn't know that.
    raiseAssert("impossible: integer is neither >, <, nor == 0")
```

---

## `failedAssertImpl`

```nim
proc failedAssertImpl*(msg: string) {.raises: [], tags: [].}
```

### What it does

`failedAssertImpl` is the **effect-system firewall** between your code and assertion failures. It calls `raiseAssert` internally, but its signature declares that it raises nothing (`raises: []`) and has no IO or other effects (`tags: []`). This allows the `assert` and `doAssert` templates to be used inside procedures that carry strict effect annotations, without forcing those procedures to declare `{.raises: [AssertionDefect].}`.

The philosophy here is that `AssertionDefect` represents a programming error, not a recoverable runtime condition. It is intentionally invisible to the effect system so that it cannot accidentally become part of a public API contract.

### Interaction with `onFailedAssert`

`failedAssertImpl` is a deliberate **extension point**. The `onFailedAssert` template works by locally shadowing this proc with a new template of the same name. This substitution mechanism is what makes custom failure handlers possible without any global state.

You will almost never call `failedAssertImpl` directly, but understanding its role is key to understanding `onFailedAssert`.

```nim
import std/assertions

# This compiles cleanly even though assert can raise:
proc pureComputation(x: int): int {.raises: [].} =
  doAssert x >= 0, "x must be non-negative"
  x * 2
```

---

## `assert`

```nim
template assert*(cond: untyped, msg = "")
```

### What it does

`assert` is the standard correctness check. It evaluates `cond` at runtime and, if `cond` is false, raises an `AssertionDefect` that includes the source location, the stringified expression that failed, and the optional `msg`.

The crucial behavioural detail is that **`assert` is compiled away entirely when `--assertions:off` or `-d:danger` is active**. No code is generated, no condition is evaluated, no side effects of `cond` occur. This makes `assert` appropriate for checks whose cost matters in production builds, but which you want active during development and testing.

### Key behaviours at a glance

| Scenario | Behaviour |
|---|---|
| `cond` is true | Nothing happens, execution continues normally |
| `cond` is false (default build) | Raises `AssertionDefect` with location + expression + `msg` |
| Built with `-d:danger` | The entire `assert` call vanishes from the binary |
| Built with `--assertions:off` | Same as `-d:danger` for assertions |

### Examples

```nim
import std/assertions

# Basic usage — check a logical invariant
let items = @[1, 2, 3]
assert items.len > 0

# With a helpful message that appears in the error
proc divide(a, b: float): float =
  assert b != 0.0, "division by zero is undefined"
  a / b

echo divide(10.0, 2.0)   # 5.0
# divide(10.0, 0.0)      # AssertionDefect: ... `b != 0.0` division by zero is undefined

# Assertions carry no overhead in danger mode:
# nim c -d:danger myfile.nim  →  the assert lines above produce zero instructions
```

---

## `doAssert`

```nim
template doAssert*(cond: untyped, msg = "")
```

### What it does

`doAssert` behaves identically to `assert` in every respect **except one**: it is **never compiled away**. The check is always present in the binary, regardless of `--assertions:off` or `-d:danger`. This makes `doAssert` appropriate for checks that must hold even in optimised or hardened production builds — particularly in test code and safety-critical validation.

### When to prefer `doAssert` over `assert`

- **In unit tests:** you want test logic to always execute, even when the test binary itself is built with `-d:release` or `-d:danger`.
- **For post-conditions on external input:** if you are validating data whose integrity matters at runtime (not just during development), `doAssert` makes the intent explicit.
- **When the condition has side effects you always want:** because `assert` can eliminate the condition entirely, any side effects inside it disappear too. `doAssert` preserves them unconditionally.

### Examples

```nim
import std/assertions

# Always checked — even in a -d:danger build
doAssert 2 + 2 == 4

proc parsePositive(s: string): int =
  result = parseInt(s)
  doAssert result > 0, "expected a positive integer, got: " & s

# In a test suite: doAssert guarantees the check runs
proc testAdder() =
  let sum = 1 + 1
  doAssert sum == 2, "basic arithmetic is broken"
```

---

## `onFailedAssert`

```nim
template onFailedAssert*(msg, code: untyped): untyped {.dirty.}
```

### What it does

`onFailedAssert` installs a **custom assertion failure handler** that applies to all `assert` and `doAssert` calls in the same scope from that point forward. When an assertion fails, instead of raising `AssertionDefect` through the default path, the `code` block is executed. Inside `code`, the parameter `msg` holds the fully-formatted failure message (source location + failed expression + any user message).

The mechanism is scope-local: the handler takes effect from the `onFailedAssert` call until the end of the enclosing block. It does not affect assertions in other scopes or modules. This lets different parts of a code base use different failure strategies without global state.

### Why this exists

The default `AssertionDefect` is an uncatchable defect by design. But testing frameworks, error-reporting libraries, and specialised subsystems sometimes need assertion failures to produce a different, catchable exception type — one that carries extra metadata like a stack trace, a log entry, or a correlation ID. `onFailedAssert` is the sanctioned hook for this.

### Examples

```nim
import std/assertions

# Example 1: convert assertion failure to a catchable error with line info
type RichError = object of CatchableError
  location: tuple[filename: string, line: int, column: int]

block:
  onFailedAssert(msg):
    raise (ref RichError)(
      msg: msg,
      location: instantiationInfo(-2)
    )

  try:
    doAssert 1 == 2, "one is not two"
  except RichError as e:
    echo "Caught at: ", e.location.filename, ":", e.location.line
    echo "Message:   ", e.msg

# Example 2: log and continue (for non-critical subsystems)
block:
  onFailedAssert(msg):
    echo "[WARN] Assertion failed: ", msg
    # no raise — execution continues

  doAssert false, "this will be logged but not crash"
  echo "still running"
```

---

## `doAssertRaises`

```nim
template doAssertRaises*(exception: typedesc, code: untyped)
```

### What it does

`doAssertRaises` verifies that a block of `code` raises the specified `exception` type (or a subtype of it). If the code raises the expected exception, the template succeeds silently. If the code raises a *different* exception, or if it completes without raising anything at all, `doAssertRaises` itself raises an `AssertionDefect` with a precise message explaining what went wrong.

Unlike `assert` and `doAssert`, this template is **always compiled in** — there is no flag to disable it. It is fundamentally a testing construct.

### Three possible outcomes

| What the wrapped code does | Result |
|---|---|
| Raises the expected exception (or a subtype) | ✅ Success — template completes normally |
| Raises a completely different exception | ❌ `AssertionDefect`: "expected raising X, instead raised Y" |
| Does not raise anything | ❌ `AssertionDefect`: "expected raising X, instead nothing was raised" |

### Working with exception hierarchies

Because Nim exceptions form a hierarchy, you can use a parent type to accept any of its children:

```nim
import std/assertions

# Accept any CatchableError subtype:
doAssertRaises(CatchableError):
  raise newException(ValueError, "too small")   # passes

# Must be the exact type or a subtype:
doAssertRaises(ValueError):
  raise newException(ValueError, "bad value")   # passes

# doAssertRaises(ValueError):
#   raise newException(IOError, "disk full")   # would fail — wrong type
```

### Examples

```nim
import std/assertions

# 1. Verify that a function raises on bad input
proc mustBePositive(n: int): int =
  doAssert n > 0, "n must be positive"
  n

doAssertRaises(AssertionDefect):
  discard mustBePositive(-1)

# 2. Verify parser rejects malformed input
import std/strutils

doAssertRaises(ValueError):
  discard parseInt("not a number")

# 3. Verify a hierarchy match
doAssertRaises(CatchableError):
  raise newException(KeyError, "key not found")

# 4. What failure looks like — if this code ran, it would produce:
# AssertionDefect: expected raising 'ValueError', instead nothing was raised by: echo "hi"
# doAssertRaises(ValueError):
#   echo "hi"
```

---

## Choosing the right tool

| I want to… | Use |
|---|---|
| Check an invariant that can be disabled in production | `assert` |
| Check an invariant that must never be disabled | `doAssert` |
| Signal a code path that is logically impossible | `raiseAssert` |
| Verify that a call raises the expected exception | `doAssertRaises` |
| Replace the default `AssertionDefect` with a custom error | `onFailedAssert` |

### The `-d:danger` mental model

Think of it as two layers of protection:

- **Development layer** (`assert`): active during `nim c` and `nim c -d:release`, gone in `nim c -d:danger`. Use for expensive checks, debug guards, and anything that would harm performance in a hot loop.
- **Permanent layer** (`doAssert`, `doAssertRaises`): always active. Use for test assertions, post-conditions on untrusted input, and invariants you are not willing to trade away for speed.
