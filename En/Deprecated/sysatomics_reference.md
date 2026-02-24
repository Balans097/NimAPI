# `sysatomics` Module Reference

> **Module:** `sysatomics` (internal runtime module; the modern public-facing equivalent is `std/atomics`)  
> **Language:** Nim  
> **Purpose:** Low-level atomic operations for lock-free concurrent programming — atomic loads, stores, exchanges, compare-and-swap, arithmetic, fences, and CPU hints.

> ⚠️ **Deprecation notice:** When compiling with `-d:nimPreviewSlimSystem`, this module emits a deprecation warning. New code should prefer `std/atomics`.

---

## Why Atomics?

In a multi-threaded program, ordinary reads and writes to shared variables are not safe — the compiler and CPU are both free to reorder instructions, cache values in registers, and partially complete writes. The result is data races, torn reads, and subtly broken logic that is nearly impossible to debug.

Atomic operations are hardware-guaranteed to be **indivisible**: no other thread can observe the operation in a partially-completed state. They are the foundation of all lock-free data structures, reference counting, flags, counters, and synchronization primitives.

---

## Platform Coverage

The module provides two distinct implementations:

- **GCC / Clang / LLVM-GCC / Nintendo Switch** — uses the C11-style `__atomic_*` built-ins, which give full control over the memory ordering model.
- **MSVC / Clang-CL (Windows)** — uses the `Interlocked*` family of Windows intrinsics. The memory model constants exist for API compatibility, but the underlying barrier behaviour is mapped to Windows conventions.
- **Other / single-threaded** — fallback implementations that degrade gracefully to ordinary operations when no thread support is present.

---

## Exported Type: `AtomType`

```nim
type AtomType* = SomeNumber | pointer | ptr | char | bool
```

A type-class constraint used by all generic atomic procedures. Only these categories of types are valid arguments. Larger composite types (objects, strings, sequences) cannot be used atomically through this API — they do not fit in a CPU register and cannot be exchanged indivisibly.

---

## Memory Model Constants (`AtomMemModel`)

Understanding memory ordering is the most important concept in lock-free programming. These constants tell the CPU and compiler how much freedom they have to reorder surrounding operations around an atomic instruction.

> **Available on GCC/Clang only.** On MSVC, equivalent constants exist for API compatibility, but the underlying implementation maps them all to appropriate Windows barriers.

### `ATOMIC_RELAXED`

The weakest ordering. The atomic operation itself is indivisible, but the compiler and CPU are free to reorder **any** other loads and stores around it freely. Use this only when you need atomicity for correctness (e.g. a counter that is never used for synchronization), and you do not need to observe or establish any happens-before relationship with other threads.

```
Thread A:              Thread B:
flag.store(RELAXED)    if flag.load(RELAXED): ...
data = 42              # data might still be 0 here!
```

### `ATOMIC_CONSUME`

A weak acquire-like ordering tied to **data dependencies**. If thread B reads a value that was released by thread A with a pointer/data dependency chain, the CPU must make the pointed-to data visible. In practice, most compilers currently map this to `ACQUIRE` for safety.

### `ATOMIC_ACQUIRE`

For **loads** (reads). No read or write in the current thread that comes after this load in program order may be reordered to appear before it. Combined with a `RELEASE` store in another thread, this establishes a synchronization point: everything thread A wrote before its `RELEASE` store is visible to thread B after its `ACQUIRE` load.

```
Thread A (producer):         Thread B (consumer):
data = 42                    while not ready.load(ACQUIRE): discard
ready.store(RELEASE)         echo data  # guaranteed to be 42
```

### `ATOMIC_RELEASE`

For **stores** (writes). No read or write in the current thread that comes before this store in program order may be reordered to appear after it. Pairs with `ACQUIRE` loads in other threads to form a happens-before edge.

### `ATOMIC_ACQ_REL`

Both acquire and release semantics simultaneously. Used for **read-modify-write** operations (like exchange or compare-and-swap) that both read an old value and write a new one, and need to synchronize in both directions.

### `ATOMIC_SEQ_CST`

The strongest ordering. A full memory barrier in both directions. All sequentially consistent atomic operations in all threads appear to execute in a single, globally agreed-upon total order. This is the safest choice and the default used by `atomicInc`, `atomicDec`, and `cas`. It is also the slowest on architectures with weak memory models (ARM, POWER).

---

## Load Operations

### `atomicLoadN`

```nim
proc atomicLoadN*[T: AtomType](p: ptr T, mem: AtomMemModel): T
```

**What it does:** Atomically reads and returns the value stored at `p`. No other thread can write to `*p` in a way that results in a torn (partially-written) read.

This is the value-returning ("N" = "native/scalar") version. It is the most convenient form for loading primitive values.

**Valid memory models:** `ATOMIC_RELAXED`, `ATOMIC_CONSUME`, `ATOMIC_ACQUIRE`, `ATOMIC_SEQ_CST`.

```nim
var counter: int = 0
let val = atomicLoadN(counter.addr, ATOMIC_ACQUIRE)
# val now holds whatever was last atomically stored to counter,
# and all writes that happened before the matching RELEASE store
# are now visible to this thread.
```

---

### `atomicLoad`

```nim
proc atomicLoad*[T: AtomType](p, ret: ptr T, mem: AtomMemModel)
```

**What it does:** The generic (pointer-to-pointer) version of `atomicLoadN`. Atomically reads the value at `p` and writes it into `*ret`. Useful when you need to work with the result via a pointer rather than by value, or for types where passing by value is awkward.

```nim
var source: int = 99
var dest: int
atomicLoad(source.addr, dest.addr, ATOMIC_SEQ_CST)
# dest == 99
```

---

## Store Operations

### `atomicStoreN`

```nim
proc atomicStoreN*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel)
```

**What it does:** Atomically writes `val` to the location pointed to by `p`. The write is indivisible — no other thread will observe a partial write.

**Valid memory models:** `ATOMIC_RELAXED`, `ATOMIC_RELEASE`, `ATOMIC_SEQ_CST`.  
(`ACQUIRE` and `CONSUME` are not valid for stores — those orderings only make sense for reads.)

```nim
var ready: bool = false
# ... set up shared data ...
atomicStoreN(ready.addr, true, ATOMIC_RELEASE)
# Any thread that subsequently loads `ready` with ACQUIRE
# is guaranteed to see all writes made before this store.
```

---

### `atomicStore`

```nim
proc atomicStore*[T: AtomType](p, val: ptr T, mem: AtomMemModel)
```

**What it does:** The generic pointer-to-pointer version of `atomicStoreN`. Atomically stores the value at `*val` into `*p`.

```nim
var target: int = 0
var newVal: int = 42
atomicStore(target.addr, newVal.addr, ATOMIC_RELEASE)
```

---

## Exchange Operations

### `atomicExchangeN`

```nim
proc atomicExchangeN*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**What it does:** Atomically writes `val` to `*p` and returns the **old** value that was previously stored there. The read and write happen as a single indivisible unit — no other thread can slip a write in between.

This is the fundamental swap primitive. It is useful for implementing spinlocks (swap in `true`, get back `false` → you acquired the lock) and for transferring ownership.

**Valid memory models:** all.

```nim
var lock: bool = false

# Attempt to acquire a spinlock:
let wasLocked = atomicExchangeN(lock.addr, true, ATOMIC_ACQ_REL)
if not wasLocked:
  echo "Lock acquired!"
  # ... critical section ...
  atomicStoreN(lock.addr, false, ATOMIC_RELEASE)
```

---

### `atomicExchange`

```nim
proc atomicExchange*[T: AtomType](p, val, ret: ptr T, mem: AtomMemModel)
```

**What it does:** Generic pointer version of `atomicExchangeN`. Atomically stores `*val` into `*p` and writes the old value of `*p` into `*ret`.

---

## Compare-and-Swap (CAS)

Compare-and-swap is the most powerful primitive in lock-free programming. It is the building block for nearly all lock-free algorithms: queues, stacks, counters, reference counting.

### `atomicCompareExchangeN`

```nim
proc atomicCompareExchangeN*[T: AtomType](
    p, expected: ptr T,
    desired: T,
    weak: bool,
    success_memmodel: AtomMemModel,
    failure_memmodel: AtomMemModel): bool
```

**What it does:** Compares the value at `*p` to `*expected`:

- If they are **equal**: writes `desired` into `*p`, returns `true`. The `success_memmodel` governs the memory ordering of this successful exchange.
- If they are **not equal**: writes the current value of `*p` back into `*expected` (so you can inspect what actually was there), returns `false`. The `failure_memmodel` governs the ordering — it must not be `RELEASE` or `ACQ_REL`, and must be no stronger than `success_memmodel`.

The `weak` flag: a `weak` CAS may spuriously fail even when the values compare equal (on some architectures, for efficiency). Always use `weak = false` unless you are in a loop that will retry on failure anyway (where spurious failures are harmless and `weak` can be faster).

**When to use it:** Implementing lock-free updates where you read-compute-write and must ensure no other thread modified the value between your read and your write. The canonical pattern is a retry loop.

```nim
var counter: int = 0

# Atomically increment counter by 1, no matter what other threads are doing.
proc incrementSafely(p: ptr int) =
  while true:
    let old = atomicLoadN(p, ATOMIC_RELAXED)
    let newVal = old + 1
    if atomicCompareExchangeN(p, old.unsafeAddr, newVal, false,
                               ATOMIC_SEQ_CST, ATOMIC_RELAXED):
      break  # success — our new value is in place
    # else: another thread changed p first; retry with fresh old value
```

---

### `atomicCompareExchange`

```nim
proc atomicCompareExchange*[T: AtomType](
    p, expected, desired: ptr T,
    weak: bool,
    success_memmodel: AtomMemModel,
    failure_memmodel: AtomMemModel): bool
```

**What it does:** Identical to `atomicCompareExchangeN`, but `desired` is passed as a pointer rather than by value. Use this when the desired value is large or already lives behind a pointer.

---

## Arithmetic and Bitwise — "Fetch-then-Op" vs "Op-then-Fetch"

There are two families of arithmetic atomics. The distinction is purely about **what value is returned**:

| Family | Returns | Naming pattern |
|---|---|---|
| `atomicFetch*` | The **old** value (before the operation) | `atomicFetchAdd`, `atomicFetchSub`, … |
| `atomic*Fetch` | The **new** value (after the operation) | `atomicAddFetch`, `atomicSubFetch`, … |

All operations in both families accept any `mem` model and apply the operation atomically. No intermediate state is observable by other threads.

### `atomicFetchAdd` / `atomicAddFetch`

```nim
proc atomicFetchAdd*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T  # returns OLD
proc atomicAddFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T  # returns NEW
```

**What they do:** Add `val` to `*p` atomically.

```nim
var hits: int = 0

# Count a page view; get the count BEFORE this increment (for sequencing):
let before = atomicFetchAdd(hits.addr, 1, ATOMIC_RELAXED)

# Count a page view; get the count AFTER this increment:
let after = atomicAddFetch(hits.addr, 1, ATOMIC_RELAXED)
```

---

### `atomicFetchSub` / `atomicSubFetch`

```nim
proc atomicFetchSub*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicSubFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**What they do:** Subtract `val` from `*p` atomically. The classic use-case is reference counting: when `atomicSubFetch` returns 0, the current thread knows it was the last to release a reference.

```nim
var refCount: int = 3

let remaining = atomicSubFetch(refCount.addr, 1, ATOMIC_ACQ_REL)
if remaining == 0:
  echo "Last reference released — free the object."
```

---

### `atomicFetchOr` / `atomicOrFetch`

```nim
proc atomicFetchOr*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicOrFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**What they do:** Atomically bitwise-OR `val` into `*p`. Useful for setting individual bits in a shared bitmask without needing a lock.

```nim
var flags: int = 0b0000
const FLAG_READY = 0b0001

discard atomicOrFetch(flags.addr, FLAG_READY, ATOMIC_RELEASE)
# flags now has FLAG_READY bit set, regardless of concurrent writes to other bits
```

---

### `atomicFetchAnd` / `atomicAndFetch`

```nim
proc atomicFetchAnd*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicAndFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**What they do:** Atomically bitwise-AND `val` into `*p`. Useful for clearing specific bits without affecting others.

```nim
var flags: int = 0b1111
const CLEAR_BUSY = 0b1101  # clears bit 1

discard atomicAndFetch(flags.addr, CLEAR_BUSY, ATOMIC_RELEASE)
```

---

### `atomicFetchXor` / `atomicXorFetch`

```nim
proc atomicFetchXor*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicXorFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**What they do:** Atomically bitwise-XOR `val` into `*p`. XOR with a mask toggles bits: if the bit was 0 it becomes 1, and vice versa.

```nim
var toggleState: int = 0
const TOGGLE_BIT = 0b0001

discard atomicXorFetch(toggleState.addr, TOGGLE_BIT, ATOMIC_SEQ_CST)
# bit 0 is now 1
discard atomicXorFetch(toggleState.addr, TOGGLE_BIT, ATOMIC_SEQ_CST)
# bit 0 is back to 0
```

---

### `atomicFetchNand` / `atomicNandFetch`

```nim
proc atomicFetchNand*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicNandFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**What they do:** Atomically compute `NOT ((*p) AND val)` and store the result back. NAND is a universal gate and can express any boolean function, but practical uses are rare. Occasionally found in specialized hardware driver code or bitfield manipulation.

> ⚠️ Note: GCC's `__atomic_nand_fetch` was not properly implemented (it did `~*p & val` rather than `~(*p & val)`) before GCC 4.4. Verify behaviour if targeting very old compilers.

---

## Test-and-Set / Clear

### `atomicTestAndSet`

```nim
proc atomicTestAndSet*(p: pointer, mem: AtomMemModel): bool
```

**What it does:** Atomically reads the byte at `p`, sets it to a non-zero "set" value (implementation-defined, typically `1`), and returns `true` if the byte was already "set" before the operation, `false` otherwise.

This is the traditional primitive for implementing spinlocks: if `atomicTestAndSet` returns `false`, you have successfully set the flag and acquired the lock. If it returns `true`, the flag was already set by someone else and you must wait and retry.

Note that `p` is untyped `pointer` — it works on raw bytes, not on a typed `bool` or `int`.

```nim
var spinlock: byte = 0

# Busy-wait spinlock acquire:
while atomicTestAndSet(spinlock.addr, ATOMIC_ACQUIRE):
  cpuRelax()  # hint CPU that we are spinning

# ... critical section ...
atomicClear(spinlock.addr, ATOMIC_RELEASE)
```

---

### `atomicClear`

```nim
proc atomicClear*(p: pointer, mem: AtomMemModel)
```

**What it does:** Atomically writes `0` to the byte at `p`. This is the counterpart to `atomicTestAndSet` — used to release a lock or reset a flag that was set with `atomicTestAndSet`.

**Valid memory models:** `ATOMIC_RELAXED`, `ATOMIC_RELEASE`, `ATOMIC_SEQ_CST`.

---

## Memory Fences

Fences are synchronization barriers that do not operate on any specific memory location — they constrain the reordering of *all* memory operations relative to the fence instruction.

### `atomicThreadFence`

```nim
proc atomicThreadFence*(mem: AtomMemModel)
```

**What it does:** Inserts a memory fence between threads. All memory operations (loads and stores) that appear before the fence in program order are prevented from being reordered past the fence, and vice versa — to the degree specified by `mem`. This is a cross-thread synchronization barrier.

```nim
# Ensure all prior writes are visible before any subsequent reads:
atomicThreadFence(ATOMIC_SEQ_CST)
```

---

### `atomicSignalFence`

```nim
proc atomicSignalFence*(mem: AtomMemModel)
```

**What it does:** Like `atomicThreadFence`, but only synchronizes between a **thread and signal handlers running in the same thread**. It does not emit any CPU barrier instructions — it only constrains the compiler's reordering. Much cheaper than `atomicThreadFence`, but useless for inter-thread synchronization.

---

### `fence` (template / proc)

```nim
template fence*()  # GCC: equivalent to atomicThreadFence(ATOMIC_SEQ_CST)
proc fence*()      # MSVC: calls _ReadWriteBarrier
```

**What it does:** A convenience full-barrier. Equivalent to the strongest possible `atomicThreadFence`. Use this as a simple, portable way to insert a complete memory barrier without having to think about which model to pass.

---

## Lock-Free Capability Queries

### `atomicAlwaysLockFree`

```nim
proc atomicAlwaysLockFree*(size: int, p: pointer): bool
```

**What it does:** Returns `true` if atomic operations on objects of `size` bytes are **always** lock-free on the current target architecture, as determined at **compile time**. The result is a compile-time constant.

"Lock-free" here means the CPU has native atomic instructions for this size and does not need to use internal library locks. Objects of 1, 2, 4, and 8 bytes are typically always lock-free on modern architectures. 16-byte objects (e.g. 128-bit integers or two pointers) are lock-free on some x86-64 with `CMPXCHG16B` but not universally.

`p` is an optional alignment hint (pass `nil` for typical alignment).

```nim
echo atomicAlwaysLockFree(sizeof(int), nil)   # true on most platforms
echo atomicAlwaysLockFree(16, nil)             # may be false
```

---

### `atomicIsLockFree`

```nim
proc atomicIsLockFree*(size: int, p: pointer): bool
```

**What it does:** Like `atomicAlwaysLockFree`, but may emit a **runtime** call to determine the answer. Returns `true` if lock-free atomic instructions will be used for objects of `size` bytes. If the answer cannot be determined at compile time, the runtime routine `__atomic_is_lock_free` is called.

Use `atomicAlwaysLockFree` when you need a compile-time constant; use `atomicIsLockFree` when a runtime check is acceptable.

---

## High-Level Convenience Procedures

These procedures are available on all platforms and automatically select the correct underlying implementation (GCC intrinsics, MSVC Interlocked, or sequential fallback).

### `atomicInc`

```nim
proc atomicInc*(memLoc: var int, x: int = 1): int {.discardable.}
```

**What it does:** Atomically adds `x` to `memLoc` and returns the new value. The default `x = 1` makes it a simple atomic increment. Uses `ATOMIC_SEQ_CST` for maximum safety.

This is the simplest atomic primitive available in this module — no memory model choice, no pointer arithmetic, just a safe in-place increment of a shared integer.

```nim
var visitors: int = 0

# In any thread:
let myCount = atomicInc(visitors)
echo "Visitor number: ", myCount
```

---

### `atomicDec`

```nim
proc atomicDec*(memLoc: var int, x: int = 1): int {.discardable.}
```

**What it does:** Atomically subtracts `x` from `memLoc` and returns the new value. The default `x = 1` makes it a simple atomic decrement. Uses `ATOMIC_SEQ_CST`.

```nim
var activeConnections: int = 0

atomicInc(activeConnections)   # connection opened
# ... handle connection ...
let remaining = atomicDec(activeConnections)
if remaining == 0:
  echo "All connections closed."
```

---

### `addAndFetch`

```nim
proc addAndFetch*(p: ptr int, val: int): int
```

**What it does:** Atomically adds `val` to `*p` and returns the **old** value (before the addition). This is a lower-level primitive used internally by `atomicInc`/`atomicDec` on MSVC.

On single-threaded or unsupported builds, it falls back to a plain increment with no atomicity.

---

### `cas`

```nim
proc cas*[T: bool|int|ptr](p: ptr T; oldValue, newValue: T): bool
```

**What it does:** A convenient, portable compare-and-swap. Atomically checks whether `*p == oldValue`; if so, writes `newValue` into `*p` and returns `true`. If not, returns `false` and leaves `*p` unchanged.

Internally uses `atomicCompareExchangeN` on GCC/Clang with `ATOMIC_SEQ_CST`, `Interlocked` functions on MSVC, inline assembly on TCC, or the legacy `__sync_bool_compare_and_swap` on older GCC.

Accepts only `bool`, `int`, and pointer types (all pointer-sized or smaller on most platforms).

```nim
var ticket: int = 0

# Only one thread succeeds in changing ticket from 0 to 1:
if cas(ticket.addr, 0, 1):
  echo "This thread got ticket 1."
else:
  echo "Another thread got there first."
```

---

## CPU Spin Hint

### `cpuRelax`

```nim
proc cpuRelax*()
```

**What it does:** Emits a CPU hint instruction indicating that the current thread is in a busy-wait spin loop. On x86/x86-64, this emits the `PAUSE` instruction, which reduces power consumption, prevents pipeline hazards, and can improve the performance of other simultaneous hardware threads (hyperthreading).

On other architectures, it may emit a compiler memory barrier (`asm volatile("" ::: "memory")`) to prevent the compiler from optimizing away the spin loop entirely, or use `YieldProcessor` on Windows.

This procedure has no effect on program correctness — it is purely a performance and power-consumption optimization. However, omitting it from spin loops on x86 can significantly reduce performance of the spinning thread and all threads sharing the same physical core.

```nim
var flag: bool = false

# Spin until another thread sets the flag:
while not atomicLoadN(flag.addr, ATOMIC_ACQUIRE):
  cpuRelax()  # be a good citizen while waiting

echo "Flag was set!"
```

---

## Quick Reference: Which Procedure to Use?

| Goal | Recommended procedure |
|---|---|
| Read a shared variable | `atomicLoadN` |
| Write a shared variable | `atomicStoreN` |
| Swap a value, get the old one | `atomicExchangeN` |
| Conditional update (lock-free loop) | `cas` or `atomicCompareExchangeN` |
| Increment a shared counter | `atomicInc` or `atomicFetchAdd` |
| Decrement, detect when zero | `atomicSubFetch` (returns new value) |
| Set/clear individual bits | `atomicOrFetch` / `atomicAndFetch` |
| Acquire a spinlock | `atomicTestAndSet`, then `atomicClear` to release |
| Full memory barrier | `fence()` |
| Spin-wait efficiently | `cpuRelax()` inside the loop |
| Check if type is lock-free | `atomicAlwaysLockFree` |

## Memory Model Quick Guide

| Model | Use for | Direction |
|---|---|---|
| `ATOMIC_RELAXED` | Counters with no sync need | Any |
| `ATOMIC_ACQUIRE` | Loads that begin a critical section | Load |
| `ATOMIC_RELEASE` | Stores that end a critical section | Store |
| `ATOMIC_ACQ_REL` | Read-modify-write (CAS, exchange) | Both |
| `ATOMIC_SEQ_CST` | Default safe choice; global order | Any |
