# `isolation` Module Reference

> **Module:** `std/isolation` (Nim standard library)  
> **Status:** ⚠️ Experimental — the interface may change in future Nim versions.  
> **Purpose:** Safe construction of isolated data subgraphs that can be transferred between threads and channels without data races.

---

## Background: Why Isolation Matters in Concurrent Code

When you write concurrent Nim programs — using `threads`, `channels`, or structured concurrency primitives — you face a fundamental safety challenge: if two threads simultaneously access the same piece of heap memory, you have a data race, which is undefined behavior.

The conventional solution is to ensure that data sent between threads is *isolated*: no other thread holds a reference to it. But the compiler cannot verify this by default; the programmer is responsible for not breaking the invariant. One mistake — sending a value while keeping a reference to it — leads to subtle, hard-to-reproduce bugs.

The `isolation` module formalizes this constraint at the type level. The `Isolated[T]` wrapper makes the "this data belongs to exactly one owner" contract machine-checked:

- **Copying is forbidden at compile time** — you cannot accidentally create a second reference.
- **Moving is the only way to transfer** — the original is destroyed in the process.
- **Isolation itself is verified by the compiler** (when using `isolate`) — the compiler proves that the value truly has no external aliases.

This design draws from the concept of *linear types* in type theory, applied practically to Nim's ownership and destructor model.

---

## Exported Symbols

---

### `Isolated[T]`

```nim
type Isolated*[T] {.sendable.} = object
  value: T
```

**What it is**

`Isolated[T]` is a generic wrapper type that enforces move-only semantics on any value of type `T`. It carries the `.sendable` pragma, which tells the Nim compiler that this type is safe to send across thread boundaries via channels.

The internal `value` field is deliberately private. The only ways to get data into an `Isolated[T]` are through `isolate` or `unsafeIsolate`, and the only way to get data back out is through `extract`. This controlled interface ensures the isolation guarantee is never silently bypassed.

**The three lifecycle hooks**

The module defines all three memory management hooks for `Isolated[T]`, and understanding them explains the wrapper's fundamental behavior:

`=copy` is **deleted** (marked `{.error.}`). Any attempt to copy an `Isolated[T]` value is a compile-time error. This is the cornerstone guarantee: you cannot create a second owner of the data.

`=sink` **delegates to the inner value's sink**. Moving an `Isolated[T]` transfers ownership of the contained value to the destination; the source is left in a destroyed state.

`=destroy` **delegates to the inner value's destructor**. When an `Isolated[T]` goes out of scope, it properly cleans up the contained value, preventing memory leaks.

**Example: the copy prohibition in action**

```nim
import std/isolation

let a = isolate(@[1, 2, 3])
# let b = a          # compile error: '=copy' is not allowed for Isolated[T]
let b = move(a)      # this is fine — ownership transfers, a is now empty
```

---

### `isolate`

```nim
func isolate*[T](value: sink T): Isolated[T] {.magic: "Isolate".}
```

**What it does**

`isolate` wraps a value in `Isolated[T]` with a **compile-time safety check**: the Nim compiler verifies that `value` forms a true isolated subgraph — meaning no existing variable outside this call holds a reference to the same heap data.

The `{.magic: "Isolate".}` pragma means the compiler does not just execute the function body as written; it intercepts the call and performs a special alias analysis before proceeding. If the check fails, you get a compile-time error, not a runtime crash.

The `sink` parameter annotation means the function takes ownership of `value` from the call site. The original variable is consumed: after `isolate(x)`, `x` is no longer valid.

**What "isolated subgraph" means**

A subgraph is the transitive closure of all heap objects reachable from a value. For a `seq[string]`, this is the seq's buffer plus every string's buffer. For an object with ref fields, it includes all referenced objects. The value is isolated when *nothing else* points into this subgraph.

Simple values like integers and stack-allocated objects are always isolated — they have no heap references at all. The interesting case is heap-allocated structures (refs, seqs, strings, closures) where aliasing is possible.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `value` | `sink T` | The value to wrap. Ownership is transferred into `Isolated[T]`. Must form an isolated subgraph — verified at compile time. |

**Returns:** `Isolated[T]` wrapping the value.

**Example: safe isolation of a freshly created value**

```nim
import std/isolation

# A newly created seq has no aliases — isolation check passes:
let iso = isolate(@[10, 20, 30])

# An integer is trivially isolated:
let isoInt = isolate(42)

# A fresh object is isolated:
type Config = object
  timeout: int
  retries: int

let isoConfig = isolate(Config(timeout: 5000, retries: 3))
```

**Example: what the compiler rejects**

```nim
import std/isolation

var data = @[1, 2, 3]
var alias = data          # now `data` and `alias` share the same seq buffer

# isolate(data)           # compile error: `data` is not isolated — `alias` aliases it
```

The compiler catches this because after `var alias = data`, two variables refer to the same underlying buffer. Wrapping `data` in `Isolated` while `alias` exists would violate the "exactly one owner" invariant.

**Example: sending isolated data to a thread**

```nim
import std/isolation, std/tasks

# Prepare data on the main thread:
let payload = isolate(@["hello", "world"])

# Transfer to another thread — no copy happens, ownership moves:
channel.send(payload)
# `payload` is now invalid — the channel owns the data
```

---

### `unsafeIsolate`

```nim
func unsafeIsolate*[T](value: sink T): Isolated[T]
```

**What it does**

`unsafeIsolate` does exactly what `isolate` does at the *type level* — it wraps a value in `Isolated[T]` — but it **skips the compile-time alias check**. The programmer is solely responsible for ensuring the value is genuinely isolated.

This is the escape hatch for situations where the compiler's analysis is too conservative and rejects a value that is actually safe to isolate. This can happen with complex data structures or patterns the alias analysis cannot reason about statically.

**When to use it**

Use `unsafeIsolate` only when:
1. You have manually verified that no other variable aliases the data being wrapped.
2. `isolate` rejects the value due to a limitation in the compiler's analysis, not due to an actual aliasing problem.
3. You understand that the responsibility for correctness shifts entirely to you.

If you use `unsafeIsolate` incorrectly — wrapping a value that is actually aliased — the result is a data race, which is undefined behavior. There is no runtime check to catch this.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `value` | `sink T` | The value to wrap. Ownership transfers. No isolation verification is performed. |

**Returns:** `Isolated[T]` wrapping the value.

**Example: using unsafeIsolate when you know it's safe**

```nim
import std/isolation

proc buildData(): seq[int] =
  result = newSeq[int](1000)
  for i in 0..<1000:
    result[i] = i * i
  # result is a local variable about to be returned;
  # after the return, the caller is the sole owner

# The returned seq has no aliases — we know this by construction,
# even if the compiler is uncertain:
let iso = unsafeIsolate(buildData())
```

**Contrast with `isolate`:**

```nim
# isolate would verify statically:
let iso1 = isolate(buildData())       # fine if compiler can prove it

# unsafeIsolate skips verification entirely:
let iso2 = unsafeIsolate(buildData()) # fine only if YOU know it's safe
```

---

### `extract`

```nim
func extract*[T](src: var Isolated[T]): T
```

**What it does**

`extract` retrieves the value stored inside an `Isolated[T]`, consuming the wrapper in the process. Internally it uses `move`, so the value is transferred out of `src` without copying. After `extract` returns, `src` is in a destroyed/empty state — accessing it further would be incorrect.

This is the mirror of `isolate`/`unsafeIsolate`: those put data *in*, `extract` takes data *out*. Together, the three functions form the complete lifecycle of an isolated value: **wrap → transfer → unwrap**.

`src` is a `var` parameter, not a `sink`, because the function needs to mutate `src` (move out of it) while leaving the `Isolated[T]` shell itself behind for proper cleanup by its destructor.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `src` | `var Isolated[T]` | The isolated wrapper to extract from. Must be a mutable variable. The contained value is moved out; `src` becomes empty afterward. |

**Returns:** `T` — the value that was inside the wrapper.

**Example: full lifecycle**

```nim
import std/isolation

# Step 1: wrap (on the producer side, e.g. main thread)
var iso = isolate(@["a", "b", "c"])

# Step 2: transfer (e.g. send through a channel — ownership moves)
# channel.send(iso)
# ... on the consumer side, receive gives back an Isolated[T] ...

# Step 3: unwrap (on the consumer side, e.g. worker thread)
let data = extract(iso)
echo data   # @["a", "b", "c"]
# iso is now empty — its destructor will run but has nothing to clean up
```

**Example: extract in a worker thread context**

```nim
import std/isolation

proc worker(iso: var Isolated[seq[int]]) =
  let data = extract(iso)
  var sum = 0
  for x in data: sum += x
  echo "Sum: ", sum

var payload = isolate(@[1, 2, 3, 4, 5])
worker(payload)
# payload is now exhausted
```

---

## The Complete Picture: Data Flow Between Threads

The three key functions map directly onto three stages of a producer-consumer pipeline:

```
[Producer thread]                    [Consumer thread]

 raw value
     │
     ▼
 isolate(value)          ──────────────────────────────►  Isolated[T]  (in channel)
 Isolated[T]                  channel.send(iso)                │
                                                               ▼
                                                          extract(iso)
                                                               │
                                                               ▼
                                                           raw value T
```

At every step, the type system prevents accidental sharing:
- You cannot copy `Isolated[T]` — only move it.
- You cannot read the data without calling `extract`, which destroys the wrapper.
- The `{.sendable.}` pragma permits the type to cross thread boundaries in channel operations.

---

## Design Notes

**Why is `=copy` a compile error and not just suppressed?** Making it a hard error means the compiler tells you immediately when you accidentally try to copy. A suppressed or missing `=copy` could silently fall back to a bitwise copy in some backends, which would duplicate pointers and cause double-frees or data races. The `{.error.}` pragma makes the invariant unambiguous.

**Why does `extract` take `var` instead of `sink`?** If `extract` took a `sink`, the wrapper object would be destroyed by the calling code after the call, which would invoke `=destroy` on an already-moved-from value. Taking `var` lets `extract` move the internal value out while leaving the shell intact for its own destructor to handle gracefully.

**The relationship to Nim's ownership model.** `Isolated[T]` is an application of Nim's ARC/ORC destructor infrastructure. It does not require a garbage collector, and it works efficiently in `--gc:arc` and `--gc:orc` builds. The move semantics are zero-cost: no extra allocation or copying occurs when transferring ownership.

---

## Summary Table

| Symbol | Kind | Purpose |
|--------|------|---------|
| `Isolated[T]` | type | Move-only wrapper; marks data as thread-safe to send |
| `isolate(value)` | func | Wrap with **compile-time** isolation check |
| `unsafeIsolate(value)` | func | Wrap **without** isolation check — programmer's responsibility |
| `extract(src)` | func | Unwrap and take ownership of the contained value |
| `=copy` | deleted hook | Prevents all copying — compile error if attempted |
| `=sink` | lifecycle hook | Enables efficient move of `Isolated[T]` values |
| `=destroy` | lifecycle hook | Cleans up contained value when wrapper goes out of scope |
