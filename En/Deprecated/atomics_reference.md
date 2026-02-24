# Atomics Module — Reference

> ⚠️ **Unstable API.** The module is part of Nim's standard library but is explicitly marked as having an unstable interface. Signatures and behaviour may change between Nim versions.

> 🔧 **Compilation requirement.** Requires `--threads:on`. By default the module uses C11 atomic primitives. To switch to C++ `std::atomic`, compile with `-d:nimUseCppAtomics`.

---

## What This Module Does

Modern multi-core programs need a way to share data between threads **without locks**. The naive approach — just reading and writing a plain variable from multiple threads — is unsafe: the CPU and compiler are both free to reorder memory operations in ways that are invisible to other cores, leading to data races and undefined behaviour.

This module provides **atomic operations**: hardware-level read-modify-write instructions that are guaranteed to be indivisible and to be seen consistently by all threads. It also exposes **memory ordering** controls that let you fine-tune exactly how much synchronisation you pay for — from the bare minimum (`moRelaxed`) to a full global sequencing guarantee (`moSequentiallyConsistent`).

The two central types are `Atomic[T]` — a generic container that wraps any small value type and makes every operation on it atomic — and `AtomicFlag` — a minimal single-bit boolean flag that is guaranteed lock-free on every conforming platform.

### How the implementation is selected

The module has three code paths selected at compile time:

- **C++ with `std::atomic`** (`-d:nimUseCppAtomics`): operations map directly to `std::atomic<T>` methods. The cleanest path.
- **C11 atomics (GCC/Clang)**: operations map to `_Atomic` types and `atomic_*_explicit` free functions. Used by default for the C backend.
- **MSVC**: C11 is not available, so the module falls back to `_Interlocked*` intrinsics.

For types that do not fit neatly into a hardware atomic (non-trivial types larger than 8 bytes), the module automatically falls back to a spin-lock implemented using `AtomicFlag`.

---

## The Memory Ordering Model

Every operation in this module accepts an optional `order: MemoryOrder` parameter. Understanding it is essential to using the module correctly.

### `MemoryOrder`

```nim
type MemoryOrder* = enum
  moRelaxed
  moConsume
  moAcquire
  moRelease
  moAcquireRelease
  moSequentiallyConsistent
```

Memory ordering controls what guarantees are made about the visibility of **other** memory operations (not just the atomic one itself) relative to this atomic operation. It is a contract between your code and the CPU/compiler about which optimisations are forbidden.

---

#### `moRelaxed`

The weakest ordering. The only guarantee is that the atomic operation itself is indivisible — it either completes fully or not at all. The CPU and compiler are free to reorder this operation relative to any non-atomic loads and stores, and even relative to other atomic operations on different locations.

**When to use:** Counters and statistics where you only care about the final value, not about the order in which increments happen relative to other work. For example, a thread-safe hit counter.

```nim
var counter: Atomic[int]

# Threads increment in any order; we only care about the total
counter.atomicInc(1)            # moRelaxed would suffice here
echo counter.load(moRelaxed)
```

---

#### `moConsume`

A weakened form of acquire that only prevents reordering of operations that **data-depend** on the loaded value. Currently discouraged by the C++ standards committee because compilers struggle to implement it correctly. Prefer `moAcquire` in all practical code.

---

#### `moAcquire`

Applied to a **load**. Guarantees that no memory reads or writes in the current thread that appear *after* this load in source order will be moved by the CPU or compiler to execute *before* it.

In other words: everything the loading thread does after the acquire load is guaranteed to see all the writes that were done before the matching release store in the publishing thread.

**When to use:** The "consumer" side of a producer-consumer pair. The consumer does an acquire load on a flag to check whether data is ready, then reads the data.

```nim
# Consumer thread
while not ready.load(moAcquire): discard  # spin
echo data  # guaranteed to see the producer's write to 'data'
```

---

#### `moRelease`

Applied to a **store**. Guarantees that no memory reads or writes in the current thread that appear *before* this store in source order will be moved to execute *after* it.

**When to use:** The "producer" side of a producer-consumer pair. The producer writes data, then does a release store on a flag to signal readiness.

```nim
# Producer thread
data = 42                    # write data first
ready.store(true, moRelease) # then signal; 'data' write cannot move past this
```

---

#### `moAcquireRelease`

For read-modify-write operations (like `exchange`, `fetchAdd`, etc.) that need to act as both a release for the writes before them and an acquire for the reads after them. Use on operations that both read a value and write a new one atomically.

---

#### `moSequentiallyConsistent`

The strongest ordering. In addition to the acquire/release guarantees, it enforces a single total ordering of all `moSequentiallyConsistent` operations across all threads. Every thread observes them in the same sequence.

This is the **default** for all operations in this module. It is the safest choice and the hardest to misuse. On most hardware (x86) it has negligible performance cost; on weakly-ordered architectures (ARM, POWER) it can require a full memory barrier instruction.

**When to use:** When in doubt. Also when you need all threads to agree on a single global order of events — for example, implementing a mutex or a barrier from scratch.

---

## Exported Types

### `Atomic[T]`

```nim
type Atomic*[T] = object
```

A generic atomic container. `T` can be any type, but the implementation differs:

- For **trivial types** (`SomeNumber`, `bool`, `enum`, `ptr`, `pointer`) whose size is 1, 2, 4, or 8 bytes: the value is stored in an internal field backed by a hardware atomic integer of the matching width. All operations compile to a single CPU instruction.
- For **non-trivial types** (structs, objects, anything larger than 8 bytes): the value is stored in a plain field alongside an `AtomicFlag` that acts as a spin-lock. Operations acquire this lock, perform the work, and release it. This is still correct but not truly lock-free.

An `Atomic[T]` should always be declared as a `var` at module level, as a heap-allocated object field, or passed by `var` reference. Never copy it — copying an atomic object is not meaningful.

---

### `AtomicFlag`

```nim
type AtomicFlag* = object
```

A single boolean flag whose two operations — `testAndSet` and `clear` — are guaranteed to be lock-free on every platform that conforms to C11/C++11. Unlike `Atomic[bool]`, an `AtomicFlag` has no `load` operation; you can only atomically set it and atomically clear it.

`AtomicFlag` is the building block for spin-locks, and it is used internally by `Atomic[T]` for non-trivial types.

A freshly declared `AtomicFlag` is in an unspecified state. The C standard guarantees it starts cleared when zero-initialised (e.g. as a global variable), but this is not guaranteed for stack-allocated flags. Prefer global or heap-allocated flags, or clear them explicitly before first use.

---

## Access Operations

These operations read and/or write the value stored in an `Atomic[T]`.

---

### `load`

```nim
proc load*[T](location: var Atomic[T]; order: MemoryOrder = moSequentiallyConsistent): T
```

#### Description

Atomically reads and returns the current value of the atomic object. No other thread can write a partial value — you either see the value before a concurrent store or the value after it, never a torn mix of bytes from both.

The `order` parameter must be one of `moRelaxed`, `moConsume`, `moAcquire`, or `moSequentiallyConsistent`. Store-only orderings (`moRelease`, `moAcquireRelease`) are invalid for loads — passing them is undefined behaviour.

#### Examples

```nim
var x: Atomic[int]
x.store(42)

echo x.load()              # → 42  (sequentially consistent)
echo x.load(moAcquire)     # → 42  (acquire; pairs with a release store)
echo x.load(moRelaxed)     # → 42  (relaxed; cheapest, no ordering guarantee)
```

---

### `store`

```nim
proc store*[T](location: var Atomic[T]; desired: T; order: MemoryOrder = moSequentiallyConsistent)
```

#### Description

Atomically replaces the current value of the atomic object with `desired`. No other thread can see a partial write.

The `order` parameter must be one of `moRelaxed`, `moRelease`, or `moSequentiallyConsistent`. Load-only orderings (`moAcquire`, `moConsume`) are invalid for stores.

#### Examples

```nim
var flag: Atomic[bool]

flag.store(true)                 # sequentially consistent
flag.store(false, moRelease)     # release; pairs with an acquire load in another thread
flag.store(true, moRelaxed)      # cheapest; use only when ordering doesn't matter
```

---

### `exchange`

```nim
proc exchange*[T](location: var Atomic[T]; desired: T; order: MemoryOrder = moSequentiallyConsistent): T
```

#### Description

Atomically replaces the value of the atomic object with `desired` and returns the **old** value — the one that was there just before the replacement. The read and the write happen as a single indivisible step.

This is a read-modify-write operation, so any `MemoryOrder` value is valid for `order`.

#### Examples

```nim
var state: Atomic[int]
state.store(10)

let old = state.exchange(20)
echo old          # → 10  (the previous value)
echo state.load() # → 20  (the new value)
```

```nim
# A common pattern: take ownership of a shared pointer by swapping it with nil
var sharedPtr: Atomic[ptr MyObj]
let mine = sharedPtr.exchange(nil)
if mine != nil:
  # We own it now; process and free it
  mine[].process()
```

---

### `compareExchange`

```nim
proc compareExchange*[T](location: var Atomic[T]; expected: var T; desired: T;
                          order: MemoryOrder = moSequentiallyConsistent): bool

proc compareExchange*[T](location: var Atomic[T]; expected: var T; desired: T;
                          success, failure: MemoryOrder): bool
```

#### Description

The cornerstone of lock-free programming. Performs the following atomically:

1. Read the current value of `location`.
2. Compare it with `expected`.
3. **If they are equal:** replace the value with `desired`, return `true`. The `expected` variable is unchanged.
4. **If they are not equal:** write the current value of `location` into `expected` (so the caller learns what the actual value is), return `false`. The atomic object is unchanged.

This is the "strong" variant, meaning it **never fails spuriously** — if the values are equal, the exchange always succeeds. See `compareExchangeWeak` for a variant that may fail spuriously but can be cheaper in loops on some architectures.

The two-`MemoryOrder` overload lets you specify separate orderings for the success case (which is a full read-modify-write) and the failure case (which is just a load).

**This operation is how you implement any lock-free data structure.** The typical usage pattern is a retry loop: read the current value, compute the new value based on it, attempt the CAS, and retry if it fails because another thread changed the value concurrently.

#### Examples

```nim
var counter: Atomic[int]
counter.store(5)

# Try to change 5 → 10
var expected = 5
let ok = counter.compareExchange(expected, 10)
echo ok        # → true  (exchange succeeded)
echo expected  # → 5     (unchanged on success)
echo counter.load() # → 10

# Now try again; but counter is 10, not 5 anymore
var expected2 = 5
let ok2 = counter.compareExchange(expected2, 99)
echo ok2        # → false  (failed; values didn't match)
echo expected2  # → 10     (updated to actual value)
echo counter.load() # → 10 (unchanged)
```

```nim
# Classic lock-free increment using CAS loop
proc lockFreeInc(counter: var Atomic[int]) =
  var current = counter.load(moRelaxed)
  while not counter.compareExchange(current, current + 1, moRelaxed, moRelaxed):
    discard  # current is updated on each failure; just retry
```

---

### `compareExchangeWeak`

```nim
proc compareExchangeWeak*[T](location: var Atomic[T]; expected: var T; desired: T;
                              order: MemoryOrder = moSequentiallyConsistent): bool

proc compareExchangeWeak*[T](location: var Atomic[T]; expected: var T; desired: T;
                              success, failure: MemoryOrder): bool
```

#### Description

Identical in semantics to `compareExchange`, with one difference: it is allowed to **fail spuriously** — even when the current value matches `expected`, the operation may return `false`. On some architectures (particularly ARM with its load-linked/store-conditional instructions), the weak variant maps to a single hardware instruction and is cheaper than the strong variant in a retry loop.

Use `compareExchangeWeak` instead of `compareExchange` **whenever you would use the operation inside a loop anyway**, because the loop already handles spurious failures.

Use `compareExchange` (the strong variant) when you need to distinguish "values didn't match" from "something else went wrong", i.e. when you cannot use a loop.

#### Examples

```nim
# Prefer weak variant inside a retry loop — potentially more efficient
proc lockFreeAdd(location: var Atomic[int]; delta: int) =
  var current = location.load(moRelaxed)
  while not location.compareExchangeWeak(current, current + delta, moRelaxed, moRelaxed):
    discard  # spurious or real failure; either way, retry with updated current
```

---

## Numerical Operations

These operations are only available for `Atomic[T]` where `T` is `SomeInteger`. They all return the **original value** that existed in the atomic object *before* the operation — not the result. This "fetch-then-operate" semantics is why they are named `fetch*`.

---

### `fetchAdd`

```nim
proc fetchAdd*[T: SomeInteger](location: var Atomic[T]; value: T;
                                order: MemoryOrder = moSequentiallyConsistent): T
```

#### Description

Atomically adds `value` to the atomic integer and returns what was there **before** the addition. This is a single indivisible read-modify-write operation — no other thread can observe the intermediate state.

#### Examples

```nim
var n: Atomic[int]
n.store(10)

echo n.fetchAdd(3)  # → 10  (returns old value)
echo n.load()       # → 13  (new value)
echo n.fetchAdd(3)  # → 13
echo n.load()       # → 16
```

---

### `fetchSub`

```nim
proc fetchSub*[T: SomeInteger](location: var Atomic[T]; value: T;
                                order: MemoryOrder = moSequentiallyConsistent): T
```

#### Description

Atomically subtracts `value` from the atomic integer and returns the **original value**.

#### Examples

```nim
var n: Atomic[int]
n.store(10)

echo n.fetchSub(4)  # → 10  (original)
echo n.load()       # → 6
```

---

### `fetchAnd`

```nim
proc fetchAnd*[T: SomeInteger](location: var Atomic[T]; value: T;
                                order: MemoryOrder = moSequentiallyConsistent): T
```

#### Description

Atomically replaces the atomic integer with its **bitwise AND** with `value`, and returns the original value. Useful for atomically clearing specific bits in a flags word.

#### Examples

```nim
var flags: Atomic[uint32]
flags.store(0b1111_1111'u32)

# Clear the lowest two bits atomically
let before = flags.fetchAnd(0b1111_1100'u32)
echo before.toBin(8)      # → "11111111"
echo flags.load.toBin(8)  # → "11111100"
```

---

### `fetchOr`

```nim
proc fetchOr*[T: SomeInteger](location: var Atomic[T]; value: T;
                               order: MemoryOrder = moSequentiallyConsistent): T
```

#### Description

Atomically replaces the atomic integer with its **bitwise OR** with `value`, and returns the original value. Useful for atomically setting specific bits in a flags word.

#### Examples

```nim
var flags: Atomic[uint32]
flags.store(0b0000_0000'u32)

# Set bit 3 atomically
discard flags.fetchOr(0b0000_1000'u32)
echo flags.load.toBin(8)  # → "00001000"
```

---

### `fetchXor`

```nim
proc fetchXor*[T: SomeInteger](location: var Atomic[T]; value: T;
                                order: MemoryOrder = moSequentiallyConsistent): T
```

#### Description

Atomically replaces the atomic integer with its **bitwise XOR** with `value`, and returns the original value. Useful for atomically toggling specific bits.

#### Examples

```nim
var flags: Atomic[uint32]
flags.store(0b1010_1010'u32)

# Toggle all bits
discard flags.fetchXor(0b1111_1111'u32)
echo flags.load.toBin(8)  # → "01010101"
```

---

## Convenience Arithmetic Operations

These are thin wrappers around `fetchAdd` / `fetchSub` that discard the return value when you only care about the side effect, not the old value.

---

### `atomicInc`

```nim
proc atomicInc*[T: SomeInteger](location: var Atomic[T]; value: T = 1)
```

#### Description

Atomically increments the atomic integer by `value` (default 1). The old value is discarded. This is exactly equivalent to `discard location.fetchAdd(value)` but reads more clearly when you don't need the previous value.

#### Examples

```nim
var counter: Atomic[int]
counter.store(0)

counter.atomicInc()    # increment by 1
counter.atomicInc(5)   # increment by 5
echo counter.load()    # → 6
```

---

### `atomicDec`

```nim
proc atomicDec*[T: SomeInteger](location: var Atomic[T]; value: T = 1)
```

#### Description

Atomically decrements the atomic integer by `value` (default 1). The old value is discarded.

#### Examples

```nim
var counter: Atomic[int]
counter.store(10)

counter.atomicDec()    # → 9
counter.atomicDec(3)   # → 6
echo counter.load()    # → 6
```

---

### `+=` operator

```nim
proc `+=`*[T: SomeInteger](location: var Atomic[T]; value: T)
```

#### Description

Syntactic sugar for `atomicInc`. Allows the familiar `x += n` syntax on atomic integers. Internally calls `fetchAdd` and discards the result.

#### Examples

```nim
var counter: Atomic[int]
counter.store(0)
counter += 10
counter += 5
echo counter.load()  # → 15
```

---

### `-=` operator

```nim
proc `-=`*[T: SomeInteger](location: var Atomic[T]; value: T)
```

#### Description

Syntactic sugar for `atomicDec`. Allows the familiar `x -= n` syntax on atomic integers. Internally calls `fetchSub` and discards the result.

#### Examples

```nim
var counter: Atomic[int]
counter.store(20)
counter -= 8
echo counter.load()  # → 12
```

---

## Flag Operations

---

### `testAndSet`

```nim
proc testAndSet*(location: var AtomicFlag; order: MemoryOrder = moSequentiallyConsistent): bool
```

#### Description

Atomically sets the `AtomicFlag` to `true` and returns its **previous** value. This is a read-modify-write operation and is guaranteed to be lock-free.

The semantics are: "was the flag already set?" — if the return value is `false`, the flag was clear and you just set it (you "won the race"); if the return value is `true`, the flag was already set by someone else.

`testAndSet` is the fundamental building block for a spin-lock: loop while `testAndSet` returns `true` (the lock is held by someone), and stop when it returns `false` (you've acquired the lock).

#### Examples

```nim
var flag: AtomicFlag

echo flag.testAndSet()   # → false  (was clear; now set)
echo flag.testAndSet()   # → true   (was already set)
echo flag.testAndSet()   # → true   (still set)
```

```nim
# Minimal spin-lock
var lock: AtomicFlag

proc acquire(lock: var AtomicFlag) =
  while lock.testAndSet(moAcquire): discard  # spin until we set it

proc release(lock: var AtomicFlag) =
  lock.clear(moRelease)
```

---

### `clear`

```nim
proc clear*(location: var AtomicFlag; order: MemoryOrder = moSequentiallyConsistent)
```

#### Description

Atomically resets the `AtomicFlag` to `false`. This is the counterpart to `testAndSet` and is guaranteed to be lock-free. After `clear`, the next call to `testAndSet` by any thread will return `false` and set the flag.

The `order` parameter should be `moRelease` or `moSequentiallyConsistent` when using the flag as a spin-lock, to ensure that all memory writes performed while holding the lock are visible to the next acquirer.

#### Examples

```nim
var flag: AtomicFlag

discard flag.testAndSet()  # set it
flag.clear()               # clear it
echo flag.testAndSet()     # → false  (was clear again)
```

---

## Memory Barrier Operations

These operations do not act on any specific atomic object. Instead, they insert barriers that control how the compiler and CPU are allowed to reorder memory operations globally.

---

### `fence`

```nim
proc fence*(order: MemoryOrder)
```

#### Description

Inserts a **hardware memory fence** — a CPU instruction (or set of instructions) that enforces ordering constraints on all memory accesses, both atomic and non-atomic, that cross the fence boundary.

A fence is a "standalone" ordering constraint not tied to any particular memory location. It is more heavyweight than attaching an ordering to an atomic operation, but is sometimes necessary when coordinating with non-atomic code or with hardware devices.

The meaning of `order` is the same as for atomic operations:
- `moAcquire`: prevents loads from being moved above the fence.
- `moRelease`: prevents stores from being moved below the fence.
- `moAcquireRelease`: combines both.
- `moSequentiallyConsistent`: full barrier; all threads see the same ordering.

Fences do not prevent the compiler from caching values in registers. For interaction with memory-mapped hardware registers, use `volatile` (not available directly in Nim's standard library).

#### Examples

```nim
# Producer thread
data[0] = 1
data[1] = 2
fence(moRelease)     # all stores above are visible before any store below
readyFlag.store(true, moRelaxed)

# Consumer thread
while not readyFlag.load(moRelaxed): discard
fence(moAcquire)     # all loads below see the writes the producer made before its release fence
echo data[0]         # → 1
echo data[1]         # → 2
```

---

### `signalFence`

```nim
proc signalFence*(order: MemoryOrder)
```

#### Description

Like `fence`, but **does not emit CPU instructions** — it only prevents the **compiler** from reordering memory accesses across the fence boundary. It has no effect on hardware reordering between CPU cores.

`signalFence` is intended specifically for synchronisation between a thread and a signal handler (Unix signals) that runs on the same thread. Because a signal handler interrupts the thread and runs in its context, hardware memory ordering is not an issue — only compiler reordering is. Using `signalFence` instead of `fence` in this scenario avoids unnecessary (and expensive) hardware barrier instructions.

#### Examples

```nim
# Synchronising with a signal handler on the same thread
var signalFlag: Atomic[bool]

proc mySignalHandler() {.noconv.} =
  signalFlag.store(true, moRelaxed)

# In main thread, check the flag set by the signal handler
signalFence(moAcquire)  # prevent compiler from caching the read above this point
if signalFlag.load(moRelaxed):
  echo "Signal received"
```

---

## Compile-time Configuration

| Flag | Effect |
|------|--------|
| `--threads:on` | Required for meaningful use. Without it, atomics provide no real concurrency guarantees. |
| `-d:nimUseCppAtomics` | Use C++ `std::atomic` instead of C11 primitives. Only meaningful with the C++ backend. |

---

## Choosing the Right Memory Order — Quick Reference

| Scenario | Load order | Store order |
|----------|-----------|-------------|
| Counter/statistics, don't care about ordering | `moRelaxed` | `moRelaxed` |
| Publish data: writer finishes writing, then signals | — | `moRelease` |
| Consume data: reader checks signal, then reads | `moAcquire` | — |
| Read-modify-write in a lock-free data structure | `moAcquireRelease` | `moAcquireRelease` |
| Need all threads to agree on a single total order | `moSequentiallyConsistent` | `moSequentiallyConsistent` |
| Unsure / correctness first | `moSequentiallyConsistent` | `moSequentiallyConsistent` |

---

## Common Patterns

**Thread-safe reference counting:**
```nim
var refCount: Atomic[int]
refCount.store(1)

proc retain() = refCount.atomicInc()

proc release(): bool =
  # fetchSub returns the old value; if it was 1, we just hit zero
  result = refCount.fetchSub(1, moAcquireRelease) == 1
```

**Spin-lock from AtomicFlag:**
```nim
var lock: AtomicFlag

template withSpinLock(body: untyped) =
  while lock.testAndSet(moAcquire): discard
  try: body
  finally: lock.clear(moRelease)

withSpinLock:
  sharedData.modify()
```

**One-time initialisation flag:**
```nim
var initialised: Atomic[bool]

proc ensureInit() =
  var expected = false
  if initialised.compareExchange(expected, true, moAcquireRelease):
    # We were first; do the actual initialisation
    setup()
  else:
    # Someone else is initialising or already done
    while not initialised.load(moAcquire): discard
```
