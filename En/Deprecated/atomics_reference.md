# Reference: `atomics` Module (Nim)

> The `atomics` module provides **atomic types and operations** for lock-free algorithms.  
> By default it uses C11 atomic primitives; compiling with `-d:nimUseCppAtomics` switches to C++11 `std::atomic`.

> ⚠️ **Unstable API** — signatures may change in future Nim versions.

---

## Table of Contents

- [Data Types](#data-types)
- [Memory Ordering (MemoryOrder)](#memory-ordering-memoryorder)
- [Atomic — Access Operations](#atomic--access-operations)
- [Atomic — Compare and Exchange](#atomic--compare-and-exchange)
- [Atomic — Numerical Operations](#atomic--numerical-operations)
- [Atomic — Operators and Convenience Wrappers](#atomic--operators-and-convenience-wrappers)
- [AtomicFlag — Operations](#atomicflag--operations)
- [Memory Fences](#memory-fences)
- [Choosing a Memory Order](#choosing-a-memory-order)
- [Quick Reference](#quick-reference)

---

## Data Types

### `Atomic[T]`

An atomic object with underlying type `T`. Guarantees that read and write operations cannot be torn by other threads.

For trivial types (`SomeNumber`, `bool`, `enum`, `ptr`, `pointer`) up to 8 bytes, hardware atomic instructions are used. For all other types, a spin-lock backed by `AtomicFlag` is used instead.

```nim
var counter: Atomic[int]
var flag: Atomic[bool]
var p: Atomic[ptr MyObj]
```

---

### `AtomicFlag`

An atomic boolean flag. The only type guaranteed to be lock-free on all platforms by the C11 standard. Starts in the cleared (`false`) state.

```nim
var flag: AtomicFlag
```

---

## Memory Ordering (MemoryOrder)

`MemoryOrder` specifies the **constraints on how non-atomic memory operations may be reordered** around an atomic operation. Stricter orders provide stronger synchronisation guarantees but may incur higher runtime cost.

All operations in this module default to `moSequentiallyConsistent` — the safest and strictest option.

### `moRelaxed`

No ordering constraints. Only the atomicity of the operation itself and ordering relative to other atomic operations on the **same variable** in the **same thread** are guaranteed.

Use for: statistics counters where exact ordering is irrelevant.

```nim
var c: Atomic[int]
c.store(1, moRelaxed)
let v = c.load(moRelaxed)
```

---

### `moConsume`

A weakened form of `moAcquire` based on data dependency. **Not recommended** — its semantics are being revised in the standard. Prefer `moAcquire`.

---

### `moAcquire`

Applied to **load** operations. No reads or writes in the current thread may be reordered **before** this operation. Forms an "acquire fence".

Typical pairing: `load(moAcquire)` in the consumer + `store(moRelease)` in the producer.

```nim
# Consumer thread:
let ready = flag.load(moAcquire)
if ready:
  let data = sharedData.load(moRelaxed)  # visible after acquire
```

---

### `moRelease`

Applied to **store** operations. No reads or writes in the current thread may be reordered **after** this operation. Forms a "release fence".

```nim
# Producer thread:
sharedData.store(42, moRelaxed)
flag.store(true, moRelease)   # guarantees sharedData visibility
```

---

### `moAcquireRelease`

For **read-modify-write** (RMW) operations: acts as both `moAcquire` and `moRelease` simultaneously. Used with `exchange`, `compareExchange`, `fetchAdd`, etc.

```nim
let old = counter.exchange(0, moAcquireRelease)
```

---

### `moSequentiallyConsistent`

The strictest order. Acts as `moAcquire` for loads, `moRelease` for stores, and `moAcquireRelease` for RMW operations. Additionally guarantees that **all threads** observe the same total ordering of all `moSequentiallyConsistent` operations.

**Default value** for all operations in this module.

```nim
var loc: Atomic[int]
loc.store(42)       # moSequentiallyConsistent
let v = loc.load()  # moSequentiallyConsistent
```

---

## Atomic — Access Operations

### `load[T](location: var Atomic[T]; order: MemoryOrder = moSequentiallyConsistent): T`

Atomically reads and returns the current value of the atomic object.

```nim
var loc: Atomic[int]
loc.store(42)

let v1 = loc.load()               # moSequentiallyConsistent
let v2 = loc.load(moRelaxed)      # no fence
let v3 = loc.load(moAcquire)      # with acquire fence
assert v1 == 42
```

---

### `store[T](location: var Atomic[T]; desired: T; order: MemoryOrder = moSequentiallyConsistent)`

Atomically writes `desired` into the atomic object.

> Valid orders: `moRelaxed`, `moRelease`, `moSequentiallyConsistent`.

```nim
var loc: Atomic[int]
loc.store(10)
loc.store(20, moRelaxed)
loc.store(30, moRelease)
assert loc.load() == 30
```

---

### `exchange[T](location: var Atomic[T]; desired: T; order: MemoryOrder = moSequentiallyConsistent): T`

Atomically replaces the value with `desired` and **returns the old value**. This is a read-modify-write (RMW) operation.

```nim
var loc: Atomic[int]
loc.store(7)

let old = loc.exchange(99)
assert old == 7
assert loc.load() == 99
```

---

## Atomic — Compare and Exchange

### `compareExchange[T](location, expected, desired, order): bool`  
### `compareExchange[T](location, expected, desired, success, failure): bool`

**CAS (Compare-And-Swap)** — atomically compares the atomic object's value with `expected`:
- If **equal**: writes `desired`, returns `true`. `expected` is unchanged.
- If **not equal**: loads the current value into `expected`, returns `false`. The object is not modified.

The two-order variant allows specifying separate memory orders for the success (`success`) and failure (`failure`) cases.

```nim
var loc: Atomic[int]
loc.store(7)

var expected = 7
let ok = loc.compareExchange(expected, 100, moRelaxed, moRelaxed)
assert ok == true
assert loc.load() == 100
assert expected == 7      # expected unchanged on success

# Failure case:
var expected2 = 0         # wrong expected value
let ok2 = loc.compareExchange(expected2, 200, moRelaxed, moRelaxed)
assert ok2 == false
assert expected2 == 100   # expected2 updated with actual value
assert loc.load() == 100  # object unchanged

# Lock-free update pattern:
var counter: Atomic[int]
counter.store(0)
var current = counter.load(moRelaxed)
while not counter.compareExchange(current, current + 1, moRelaxed, moRelaxed):
  discard   # current is automatically refreshed; retry
```

---

### `compareExchangeWeak[T](location, expected, desired, order): bool`  
### `compareExchangeWeak[T](location, expected, desired, success, failure): bool`

Weak variant of CAS. Identical to `compareExchange`, but may **fail spuriously** — it can return `false` even when the values are equal. On some architectures (ARM) this compiles to more efficient code.

Use inside retry loops, where a spurious failure simply causes one extra iteration:

```nim
var loc: Atomic[int]
loc.store(5)
var expected = loc.load(moRelaxed)
while not loc.compareExchangeWeak(expected, expected * 2, moRelaxed, moRelaxed):
  discard
assert loc.load() == 10
```

> **`compareExchange` vs `compareExchangeWeak`**: use `compareExchange` for a single one-shot check. Use `compareExchangeWeak` inside retry loops — it can be faster on LL/SC architectures (ARM, RISC-V).

---

## Atomic — Numerical Operations

All numerical operations are only available for `SomeInteger` types. Each atomically performs the operation and returns the **value before the change**.

### `fetchAdd[T: SomeInteger](location, value, order): T`

Atomically adds `value` and returns the **previous** value.

```nim
var c: Atomic[int]
c.store(5)
assert c.fetchAdd(3) == 5   # was 5, now 8
assert c.load() == 8
assert c.fetchAdd(2) == 8   # was 8, now 10
```

---

### `fetchSub[T: SomeInteger](location, value, order): T`

Atomically subtracts `value` and returns the **previous** value.

```nim
var c: Atomic[int]
c.store(10)
assert c.fetchSub(3) == 10  # was 10, now 7
assert c.load() == 7
```

---

### `fetchAnd[T: SomeInteger](location, value, order): T`

Atomically applies bitwise AND with `value` and returns the **previous** value.

```nim
var flags: Atomic[int]
flags.store(0b1111)
let old = flags.fetchAnd(0b1010)
assert old == 0b1111
assert flags.load() == 0b1010
```

---

### `fetchOr[T: SomeInteger](location, value, order): T`

Atomically applies bitwise OR with `value` and returns the **previous** value.

```nim
var flags: Atomic[int]
flags.store(0b0101)
let old = flags.fetchOr(0b1010)
assert old == 0b0101
assert flags.load() == 0b1111
```

---

### `fetchXor[T: SomeInteger](location, value, order): T`

Atomically applies bitwise XOR with `value` and returns the **previous** value.

```nim
var flags: Atomic[int]
flags.store(0b1100)
let old = flags.fetchXor(0b1010)
assert old == 0b1100
assert flags.load() == 0b0110
```

---

## Atomic — Operators and Convenience Wrappers

### `atomicInc[T: SomeInteger](location: var Atomic[T]; value: T = 1)`

Atomically increments the value by `value` (default: 1). A wrapper around `fetchAdd` that discards the return value.

```nim
var c: Atomic[int]
c.store(0)
c.atomicInc()      # +1
c.atomicInc(5)     # +5
assert c.load() == 6
```

---

### `atomicDec[T: SomeInteger](location: var Atomic[T]; value: T = 1)`

Atomically decrements the value by `value` (default: 1). A wrapper around `fetchSub`.

```nim
var c: Atomic[int]
c.store(10)
c.atomicDec()      # -1 → 9
c.atomicDec(4)     # -4 → 5
assert c.load() == 5
```

---

### `+=[T: SomeInteger](location: var Atomic[T]; value: T)`

Atomic add-and-assign operator. Equivalent to `atomicInc`.

```nim
var c: Atomic[int]
c.store(0)
c += 3
c += 7
assert c.load() == 10
```

---

### `-=[T: SomeInteger](location: var Atomic[T]; value: T)`

Atomic subtract-and-assign operator. Equivalent to `atomicDec`.

```nim
var c: Atomic[int]
c.store(10)
c -= 3
assert c.load() == 7
```

---

## AtomicFlag — Operations

### `testAndSet(location: var AtomicFlag; order: MemoryOrder = moSequentiallyConsistent): bool`

Atomically sets the flag to `true` and returns its **previous** value.

- Returns `false` — the flag **was not set**; it is now set.
- Returns `true` — the flag **was already set**.

Classic use: implementing a spin-lock:

```nim
var flag: AtomicFlag

# First call: flag was clear → now set, returns false
assert not flag.testAndSet()

# Second call: flag already set → returns true
assert flag.testAndSet()

# Spin-lock pattern:
var lock: AtomicFlag
while lock.testAndSet(moAcquire): discard   # spin until acquired
# --- critical section ---
lock.clear(moRelease)                        # release
```

---

### `clear(location: var AtomicFlag; order: MemoryOrder = moSequentiallyConsistent)`

Atomically resets the flag to `false`.

> Valid orders: `moRelaxed`, `moRelease`, `moSequentiallyConsistent`.

```nim
var flag: AtomicFlag
discard flag.testAndSet()    # set
assert flag.testAndSet()     # confirm set
flag.clear(moRelaxed)        # clear
assert not flag.testAndSet() # no longer set
```

---

## Memory Fences

### `fence(order: MemoryOrder)`

Establishes a **thread memory fence** without performing an atomic operation. Enforces the required visibility ordering for ordinary (non-atomic) variables between threads.

```nim
# Producer thread:
sharedData = 42         # ordinary write
fence(moRelease)        # fence: everything above is visible before the fence

# Consumer thread:
fence(moAcquire)        # fence: everything below happens after the fence
let v = sharedData      # reads 42
```

---

### `signalFence(order: MemoryOrder)`

A **compiler-only fence** (signal fence). Prevents the compiler from reordering operations across this point but inserts no CPU memory-ordering instructions. Used to prevent compiler optimisations in signal handlers.

```nim
signalFence(moAcquireRelease)  # compiler barrier only
```

---

## Choosing a Memory Order

| Scenario | load | store | RMW |
|---|---|---|---|
| Maximum safety (default) | `moSequentiallyConsistent` | `moSequentiallyConsistent` | `moSequentiallyConsistent` |
| Producer-consumer handoff | `moAcquire` | `moRelease` | — |
| Independent statistics counters | `moRelaxed` | `moRelaxed` | `moRelaxed` |
| Mutex acquire | — | — | `moAcquire` |
| Mutex release | — | `moRelease` | — |

> **Safety rule**: start with `moSequentiallyConsistent` (the default). Relax the order only with a solid understanding of the memory model and a confirmed need for optimisation.

---

## Quick Reference

### Atomic[T]

| Operation | Function | Returns |
|---|---|---|
| Read | `load(order)` | `T` — current value |
| Write | `store(desired, order)` | — |
| Swap | `exchange(desired, order)` | `T` — old value |
| Strong CAS | `compareExchange(expected, desired, ...)` | `bool` — success |
| Weak CAS | `compareExchangeWeak(expected, desired, ...)` | `bool` — success |
| Add | `fetchAdd(value, order)` | `T` — old value |
| Subtract | `fetchSub(value, order)` | `T` — old value |
| Bitwise AND | `fetchAnd(value, order)` | `T` — old value |
| Bitwise OR | `fetchOr(value, order)` | `T` — old value |
| Bitwise XOR | `fetchXor(value, order)` | `T` — old value |
| Increment | `atomicInc(value)` / `+= value` | — |
| Decrement | `atomicDec(value)` / `-= value` | — |

### AtomicFlag

| Operation | Function | Returns |
|---|---|---|
| Set and read | `testAndSet(order)` | `bool` — value **before** setting |
| Clear | `clear(order)` | — |

### Memory Order Strength (weakest to strongest)

```
moRelaxed  <  moConsume  <  moAcquire / moRelease  <  moAcquireRelease  <  moSequentiallyConsistent
```

### Notes

- API is marked **unstable**: signatures may change.
- Default backend: C11 atomics. Use `-d:nimUseCppAtomics` for C++ `std::atomic`.
- `AtomicFlag` is the only type **guaranteed** to be lock-free on all platforms.
- `Atomic[T]` for non-trivial types (not `SomeNumber`/`bool`/`ptr`) is implemented via a spin-lock on `AtomicFlag`.
- All `fetch*` functions return the value **before** the modification.
