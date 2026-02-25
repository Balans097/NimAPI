# logic.nim — Module Reference

> **Scope:** This module extends Nim's built-in boolean toolkit with formal-logic operators intended for use inside **contract pragmas** such as `.ensures`, `.requires`, and `.invariant`. These operators are *not* general-purpose runtime functions — they are special *magic* constructs understood by the Nim compiler's formal verification and specification layer. Using them outside of contract context is either a no-op or undefined.

---

## Table of Contents

1. [`->` — Logical Implication](#--logical-implication)
2. [`<->` — Logical Equivalence (Biconditional)](#---logical-equivalence-biconditional)
3. [`forall` — Universal Quantifier](#forall--universal-quantifier)
4. [`exists` — Existential Quantifier](#exists--existential-quantifier)
5. [`old` — Pre-state Value Capture](#old--pre-state-value-capture)

---

## `->` — Logical Implication

```nim
proc `->`*(a, b: bool): bool {.magic: "Implies".}
```

### What it means

The implication operator reads: **"if `a` then `b`"**. It returns `false` only when `a` is `true` and `b` is `false` — that is, the only way an implication can be violated is if the premise holds but the conclusion does not.

| `a`   | `b`   | `a -> b` |
|-------|-------|----------|
| false | false | **true** |
| false | true  | **true** |
| true  | false | **false** |
| true  | true  | **true** |

The intuition: *a promise is only broken when you had every opportunity to keep it and didn't.*

### When to use it

Use `->` inside postconditions (`.ensures`) or invariants when one condition should **conditionally guarantee** another. It lets you express *conditional* correctness: "only when X is the case, Y must hold."

### Example

```nim
proc safeDivide(a, b: int): int {.ensures: (b != 0) -> (result * b == a).} =
  if b == 0: 0 else: a div b
```

Here the postcondition says: *if the divisor is non-zero, then the result times `b` must equal `a`*. When `b` is zero we make no promise about the result at all — the implication's premise is false, so it is automatically satisfied.

---

## `<->` — Logical Equivalence (Biconditional)

```nim
proc `<->`*(a, b: bool): bool {.magic: "Iff".}
```

### What it means

The biconditional reads: **"`a` if and only if `b`"**. Both sides must have the same truth value for the expression to be `true`. It is the symmetric, two-directional version of implication: `(a -> b) and (b -> a)`.

| `a`   | `b`   | `a <-> b` |
|-------|-------|-----------|
| false | false | **true**  |
| false | true  | **false** |
| true  | false | **false** |
| true  | true  | **true**  |

### When to use it

Use `<->` when two properties are *equivalent by definition* — whenever one holds, the other must too, with no exceptions in either direction. Common in invariants that tie two flags or states together.

### Example

```nim
type Stack[T] = object
  data: seq[T]
  isEmpty: bool

proc push[T](s: var Stack[T]; val: T)
    {.ensures: s.isEmpty <-> (s.data.len == 0).} =
  s.data.add(val)
  s.isEmpty = s.data.len == 0
```

The postcondition asserts that the `isEmpty` flag and the actual emptiness of `data` are *always in sync* — neither can be true without the other.

---

## `forall` — Universal Quantifier

```nim
proc forall*(args: varargs[untyped]): bool {.magic: "Forall".}
```

### What it means

`forall` expresses the mathematical **∀ (for all)** quantifier. It asserts that a property holds for *every* element in a collection or range. Think of it as a specification-level `all()` check that the verifier can reason about symbolically — rather than actually iterating at runtime.

The conventional calling convention mirrors mathematical notation:

```nim
forall(x, collection, predicate(x))
```

- **`x`** — a bound variable (name chosen by the author)
- **`collection`** — the domain to quantify over
- **`predicate(x)`** — a boolean expression that must hold for every `x`

### When to use it

Use `forall` in `.ensures` or `.invariant` pragmas when you need to guarantee something about *all* elements of a data structure — e.g., every element is positive, a sorted invariant holds everywhere, no null pointers exist.

### Example

```nim
proc sortAscending(arr: var seq[int])
    {.ensures: forall(i, 0..<arr.len-1, arr[i] <= arr[i+1]).} =
  # sorting implementation
  arr.sort()
```

The postcondition reads: *for all valid indices `i`, the element at `i` is less than or equal to the element at `i+1`* — a complete, formal statement of the ascending-sort property.

---

## `exists` — Existential Quantifier

```nim
proc exists*(args: varargs[untyped]): bool {.magic: "Exists".}
```

### What it means

`exists` expresses the mathematical **∃ (there exists)** quantifier. It asserts that *at least one* element in a collection satisfies a given property. It is the dual of `forall`: where `forall` demands universal truth, `exists` only requires a single witness.

```nim
exists(x, collection, predicate(x))
```

### When to use it

Use `exists` when a function's correctness guarantee is *existential* — the function must produce or leave behind at least one element with a certain property, but need not say which one or how many.

### Example

```nim
proc findPositive(arr: seq[int]): int
    {.requires: exists(x, arr, x > 0),
      ensures:  result > 0.} =
  for v in arr:
    if v > 0: return v
  0
```

The precondition uses `exists` to demand that the caller only invokes this procedure when at least one positive element is present. The postcondition then guarantees the returned value is positive.

---

## `old` — Pre-state Value Capture

```nim
proc old*[T](x: T): T {.magic: "Old".}
```

### What it means

`old(x)` captures the **value of `x` at the moment the procedure was called** — before any modifications made by the procedure body. This is indispensable in postconditions, where you want to relate the *final* state to the *initial* state. Without `old`, a postcondition cannot reference what a mutable variable looked like before the call.

The `[T]` type parameter means `old` works on any type: integers, strings, sequences, objects — anything.

### When to use it

Use `old` exclusively inside `.ensures` pragmas when the correctness of a side-effecting procedure is expressed as a *change* or *relationship* between before and after. If your postcondition only talks about the result or the final state in isolation, `old` is not needed. But as soon as you say "the list grew by one" or "the counter increased", you need to anchor the before-state with `old`.

### Example

```nim
proc appendItem[T](s: var seq[T]; item: T)
    {.ensures: s.len == old(s.len) + 1 and s[^1] == item.} =
  s.add(item)
```

The postcondition makes two promises:
1. The sequence is exactly one element longer than it was before the call (`old(s.len) + 1`).
2. The last element is the item that was appended.

Without `old(s.len)`, there would be no way to express "one longer than before" — only "some length."

---

## Summary Table

| Operator / Proc | Formal meaning | Typical pragma |
|-----------------|----------------|----------------|
| `a -> b`        | If `a` then `b` (implication) | `.ensures`, `.invariant` |
| `a <-> b`       | `a` iff `b` (equivalence) | `.ensures`, `.invariant` |
| `forall(x, S, P(x))` | ∀ x ∈ S: P(x) | `.ensures`, `.requires` |
| `exists(x, S, P(x))` | ∃ x ∈ S: P(x) | `.ensures`, `.requires` |
| `old(x)`        | Value of `x` before the call | `.ensures` only |

---

> **Note:** All five constructs are compiler *magic* — they have no conventional runtime implementation. They exist to let Nim's formal specification and verification tooling reason about program correctness. Treat this module as a **specification language embedded in Nim**, not as a runtime utility library.
