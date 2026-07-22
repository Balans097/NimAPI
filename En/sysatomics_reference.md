# system/atomics — Module Reference

> **Import:** `import system/atomics`
> **Scope:** low-level atomic memory operations for multithreaded code — atomic load/store, exchange, compare-and-swap (CAS), increment/decrement, bitwise operations, and memory fences.

The module is a compiler-specific wrapper over the GCC/Clang atomic intrinsics (`__atomic_*`) and the MSVC ones (`_Interlocked*`), plus a handful of platform-independent procedures built on top of them (`atomicInc`, `atomicDec`, `cas`, `fence`, `cpuRelax`). The module's general convention: almost every procedure takes a `mem: AtomMemModel` parameter that specifies the memory model (fence) the operation runs under — from `ATOMIC_RELAXED` (no ordering guarantees at all) to `ATOMIC_SEQ_CST` (a full fence, the safest and slowest option). Several procedures exist in two forms: the "N" form (`atomicLoadN`, `atomicStoreN`, ...) works with the value directly, while the plain form (`atomicLoad`, `atomicStore`, ...) works through a pointer to the second operand/result (the "generic" variant of the intrinsic). The module is considered low-level: application code should almost always prefer `std/atomics` instead — this file is the foundation `std/atomics` is built on.

---

## Table of Contents

I. [Types and Constants](#i-types-and-constants)
&nbsp;&nbsp;&nbsp;1. [`AtomType`](#1-atomtype)
&nbsp;&nbsp;&nbsp;2. [`AtomMemModel` and the `ATOMIC_*` constants](#2-atommemmodel-and-the-atomic_-constants)

II. [Atomic Load and Store](#ii-atomic-load-and-store)
&nbsp;&nbsp;&nbsp;1. [`atomicLoadN`](#1-atomicloadn)
&nbsp;&nbsp;&nbsp;2. [`atomicLoad`](#2-atomicload)
&nbsp;&nbsp;&nbsp;3. [`atomicStoreN`](#3-atomicstoren)
&nbsp;&nbsp;&nbsp;4. [`atomicStore`](#4-atomicstore)

III. [Atomic Exchange and Compare-Exchange](#iii-atomic-exchange-and-compare-exchange)
&nbsp;&nbsp;&nbsp;1. [`atomicExchangeN`](#1-atomicexchangen)
&nbsp;&nbsp;&nbsp;2. [`atomicExchange`](#2-atomicexchange)
&nbsp;&nbsp;&nbsp;3. [`atomicCompareExchangeN`](#3-atomiccompareexchangen)
&nbsp;&nbsp;&nbsp;4. [`atomicCompareExchange`](#4-atomiccompareexchange)
&nbsp;&nbsp;&nbsp;5. [`cas`](#5-cas)

IV. [Arithmetic and Bitwise Atomic Operations](#iv-arithmetic-and-bitwise-atomic-operations)
&nbsp;&nbsp;&nbsp;1. [`atomic*Fetch` (returns the new value)](#1-atomicfetch-returns-the-new-value)
&nbsp;&nbsp;&nbsp;2. [`atomicFetch*` (returns the old value)](#2-atomicfetch-returns-the-old-value)
&nbsp;&nbsp;&nbsp;3. [`atomicInc`, `atomicDec`](#3-atomicinc-atomicdec)
&nbsp;&nbsp;&nbsp;4. [`addAndFetch`](#4-addandfetch)

V. [Flags, Fences, and Helper Procedures](#v-flags-fences-and-helper-procedures)
&nbsp;&nbsp;&nbsp;1. [`atomicTestAndSet`, `atomicClear`](#1-atomictestandset-atomicclear)
&nbsp;&nbsp;&nbsp;2. [`atomicThreadFence`, `atomicSignalFence`, `fence`](#2-atomicthreadfence-atomicsignalfence-fence)
&nbsp;&nbsp;&nbsp;3. [`cpuRelax`](#3-cpurelax)
&nbsp;&nbsp;&nbsp;4. [`atomicAlwaysLockFree`, `atomicIsLockFree`](#4-atomicalwayslockfree-atomicislockfree)

VI. [Practical Recipes](#vi-practical-recipes)
&nbsp;&nbsp;&nbsp;1. [Atomic reference counter](#1-atomic-reference-counter)
&nbsp;&nbsp;&nbsp;2. [One-time lazy initialization (double-checked locking)](#2-one-time-lazy-initialization-double-checked-locking)
&nbsp;&nbsp;&nbsp;3. [Spinlock on `atomicTestAndSet`](#3-spinlock-on-atomictestandset)
&nbsp;&nbsp;&nbsp;4. [Lock-free stack (Treiber stack) on `cas`](#4-lock-free-stack-treiber-stack-on-cas)

VII. [Quick Reference Table](#vii-quick-reference-table)

VIII. [Summary: Which Procedure to Choose](#viii-summary-which-procedure-to-choose)

---

## I. Types and Constants

### 1. `AtomType`

```nim
type
  AtomType* = SomeNumber|pointer|ptr|char|bool
```

**What it does.** This is not a type in the usual sense but a type class — a constraint the compiler checks when instantiating the module's generic procedures. Any procedure of the form `proc atomicXxx*[T: AtomType](...)` only agrees to work with numbers (`SomeNumber` — all integer and floating-point types), untyped pointers (`pointer`), typed pointers (`ptr`), characters (`char`), and booleans (`bool`). The point of the constraint: the processor's atomic intrinsics can only operate on values of a fixed, small size (1/2/4/8/16 bytes) that can be copied with a simple bitwise copy — atomic operations don't directly apply to arbitrary-sized structs, seqs, or strings.

- **Parameters:** none — this is a type constraint, not a procedure.

**Example:**

```nim
# NOTE: AtomType is used implicitly, as a generic-parameter constraint
var counter: int = 0
discard atomicAddFetch(addr(counter), 1, ATOMIC_SEQ_CST) # int satisfies SomeNumber -> AtomType
```

---

### 2. `AtomMemModel` and the `ATOMIC_*` constants

```nim
type AtomMemModel* = distinct cint

var ATOMIC_RELAXED* {.importc: "__ATOMIC_RELAXED", nodecl.}: AtomMemModel
var ATOMIC_CONSUME* {.importc: "__ATOMIC_CONSUME", nodecl.}: AtomMemModel
var ATOMIC_ACQUIRE* {.importc: "__ATOMIC_ACQUIRE", nodecl.}: AtomMemModel
var ATOMIC_RELEASE* {.importc: "__ATOMIC_RELEASE", nodecl.}: AtomMemModel
var ATOMIC_ACQ_REL* {.importc: "__ATOMIC_ACQ_REL", nodecl.}: AtomMemModel
var ATOMIC_SEQ_CST* {.importc: "__ATOMIC_SEQ_CST", nodecl.}: AtomMemModel
```

**What it does.** `AtomMemModel` wraps the integer code for the memory model that the C compiler passes to its `__atomic_*` intrinsics. The six constants are six levels of fence "strictness":

- `ATOMIC_RELAXED` — no ordering guarantees relative to other operations; only the atomicity of the operation itself is guaranteed;
- `ATOMIC_CONSUME` — ordering only along the data-dependency chain (in practice many compilers treat it the same as `ATOMIC_ACQUIRE`);
- `ATOMIC_ACQUIRE` — a "don't hoist code above this point" fence, used for reads;
- `ATOMIC_RELEASE` — a "don't sink code below this point" fence, used for writes;
- `ATOMIC_ACQ_REL` — both fences at once, for read-modify-write operations;
- `ATOMIC_SEQ_CST` — a full fence with a single global order across all threads; the safest default choice unless there's a reason to pick a weaker model for performance.

Analogy: if the `ATOMIC_ACQUIRE`/`ATOMIC_RELEASE` fences are a barrier gate that lets traffic through in only one direction (forbidding "peeking" into the future or "arriving late" from the past), then `ATOMIC_SEQ_CST` is a fully closed intersection with a single traffic light shared by all threads at once.

- **Parameters:** constants, take no arguments.

**Example:**

```nim
var flag: bool = false
atomicStoreN(addr(flag), true, ATOMIC_RELEASE)      # publish the result of the work
let v = atomicLoadN(addr(flag), ATOMIC_ACQUIRE)     # we see it no earlier than it was published
echo v # prints true
```

---

## II. Atomic Load and Store

### 1. `atomicLoadN`

```nim
proc atomicLoadN*[T: AtomType](p: ptr T, mem: AtomMemModel): T
```

**What it does.** Atomically reads the value at address `p` and returns it. Unlike a plain dereference `p[]`, it guarantees the read won't be split into parts by the compiler or the processor, and that it's ordered relative to other threads according to `mem`. Valid models: `ATOMIC_RELAXED`, `ATOMIC_SEQ_CST`, `ATOMIC_ACQUIRE`, `ATOMIC_CONSUME` — models implying a write (`ATOMIC_RELEASE`, `ATOMIC_ACQ_REL`) don't make sense for a read.

- `p: ptr T` — pointer to the cell being read, not mutated.
- `mem: AtomMemModel` — memory model for the read operation.

**Example:**

```nim
var x: int = 42
let y = atomicLoadN(addr(x), ATOMIC_SEQ_CST)
echo y # prints 42
```

---

### 2. `atomicLoad`

```nim
proc atomicLoad*[T: AtomType](p, ret: ptr T, mem: AtomMemModel)
```

**What it does.** The generic (non-"N") variant of the load: instead of returning the value via `result`, it writes the value read into the address `ret`. The compiler uses this when `T` is too large to return efficiently by value, or when generating code with no dedicated return register.

- `p: ptr T` — the address to read from, not mutated.
- `ret: ptr T` — address where the atomically-read value is written (mutated).
- `mem: AtomMemModel` — memory model for the read.

**Example:**

```nim
var source: int = 7
var dest: int
atomicLoad(addr(source), addr(dest), ATOMIC_ACQUIRE)
echo dest # prints 7
```

---

### 3. `atomicStoreN`

```nim
proc atomicStoreN*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel)
```

**What it does.** Atomically writes `val` to the address `p`. Valid models are `ATOMIC_RELAXED`, `ATOMIC_SEQ_CST`, `ATOMIC_RELEASE` (models implying a read don't apply to a store).

- `p: ptr T` — destination address (mutated).
- `val: T` — value being written, not mutated.
- `mem: AtomMemModel` — memory model of the store.

**Example:**

```nim
var ready: bool = false
atomicStoreN(addr(ready), true, ATOMIC_RELEASE)
echo ready # prints true
```

---

### 4. `atomicStore`

```nim
proc atomicStore*[T: AtomType](p, val: ptr T, mem: AtomMemModel)
```

**What it does.** The generic variant of a store: the value is passed not by value but through the pointer `val`. Semantically equivalent to `atomicStoreN(p, val[], mem)`, but matches the signature of the `__atomic_store` intrinsic (as opposed to `__atomic_store_n`).

- `p: ptr T` — destination address (mutated).
- `val: ptr T` — address of the source value, not mutated by the call itself.
- `mem: AtomMemModel` — memory model of the store.

**Example:**

```nim
var dest: int
let src = 99
atomicStore(addr(dest), unsafeAddr(src), ATOMIC_SEQ_CST)
echo dest # prints 99
```

---

## III. Atomic Exchange and Compare-Exchange

### 1. `atomicExchangeN`

```nim
proc atomicExchangeN*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**What it does.** Atomically writes `val` to address `p` and returns whatever value was there *before* the write — the "exchange" happens as a single indivisible action, with no window in which another thread could slip in between reading the old value and writing the new one.

- **Implementation notes.** The key difference from the sequence "`atomicLoadN` + `atomicStoreN`" is the atomicity of the whole pair of operations at once: a competing thread will either see the state "before" or "after", but can never observe something in between. All six memory models are allowed.

- `p: ptr T` — the cell's address (mutated).
- `val: T` — the new value, not mutated.
- `mem: AtomMemModel` — memory model of the operation.

**Example:**

```nim
var lockFlag: int = 0
let old = atomicExchangeN(addr(lockFlag), 1, ATOMIC_ACQUIRE)
echo old # prints 0 — before acquiring, the lock was free
```

---

### 2. `atomicExchange`

```nim
proc atomicExchange*[T: AtomType](p, val, ret: ptr T, mem: AtomMemModel)
```

**What it does.** The generic version of exchange: the new value is read from `val`, and the previous value at `p` is stored into `ret`, instead of being returned via `result`.

- `p: ptr T` — the cell's address (mutated).
- `val: ptr T` — address of the new value, not mutated.
- `ret: ptr T` — address where the previous value is stored (mutated).
- `mem: AtomMemModel` — memory model of the operation.

**Example:**

```nim
var cell: int = 5
let newVal = 10
var oldVal: int
atomicExchange(addr(cell), unsafeAddr(newVal), addr(oldVal), ATOMIC_SEQ_CST)
echo oldVal, " ", cell # prints 5 10
```

---

### 3. `atomicCompareExchangeN`

```nim
proc atomicCompareExchangeN*[T: AtomType](p, expected: ptr T, desired: T,
  weak: bool, success_memmodel: AtomMemModel, failure_memmodel: AtomMemModel): bool
```

**What it does.** This is the fundamental CAS (compare-and-swap) operation: it compares the contents at `p` with the contents of `expected[]`; if they're equal, it atomically writes `desired` to `p` and returns `true`; if they're not equal, it writes the actual (changed) value from `p` back into `expected[]` and returns `false`. The comparison and the write happen as a single indivisible operation.

- **Implementation notes.** The `weak` parameter permits a "spurious failure" — on some architectures (e.g., ARM with LL/SC instructions), a weak CAS can return `false` even when the values actually matched, because the implementation doesn't guarantee atomicity in the rare case the load-linked/store-conditional chain gets interrupted. The weak variant is cheaper and is typically used inside a retry loop, where an occasional spurious "failure" simply leads to another iteration. The strong variant (`weak = false`) never fails spuriously, but can be slower — use it if the CAS runs only once, outside a loop. `failure_memmodel` cannot be stricter than `success_memmodel`, and cannot be `ATOMIC_RELEASE`/`ATOMIC_ACQ_REL` (there's no point in a "publish" fence on the failure path, where nothing is written).

- `p: ptr T` — the cell being compared and potentially mutated (mutated on success).
- `expected: ptr T` — the expected value on input; on failure it's overwritten with the actual value from `p` (mutated).
- `desired: T` — the value written on a match, not mutated.
- `weak: bool` — `true` allows spurious failure (for retry loops), `false` is strict semantics.
- `success_memmodel: AtomMemModel` — memory model on a successful exchange.
- `failure_memmodel: AtomMemModel` — memory model on failure.

**Example:**

```nim
var value: int = 10
var expected: int = 10
let ok = atomicCompareExchangeN(addr(value), addr(expected), 20,
  false, ATOMIC_SEQ_CST, ATOMIC_SEQ_CST)
echo ok, " ", value # prints true 20 — the value matched and was replaced

var expectedFail: int = 999   # a deliberately wrong expectation
let failed = atomicCompareExchangeN(addr(value), addr(expectedFail), 30,
  false, ATOMIC_SEQ_CST, ATOMIC_SEQ_CST)
echo failed, " ", expectedFail # prints false 20 — expectedFail received the actual value
```

---

### 4. `atomicCompareExchange`

```nim
proc atomicCompareExchange*[T: AtomType](p, expected, desired: ptr T,
  weak: bool, success_memmodel: AtomMemModel, failure_memmodel: AtomMemModel): bool
```

**What it does.** Fully mirrors the semantics of `atomicCompareExchangeN`, but the desired value `desired` is also passed by address rather than by value — this matches the `__atomic_compare_exchange` intrinsic (as opposed to `__atomic_compare_exchange_n`) and is used by the compiler for large types that are cheaper to pass by pointer.

- `p: ptr T` — the cell being compared and potentially mutated (mutated on success).
- `expected: ptr T` — address of the expected value; overwritten on failure (mutated).
- `desired: ptr T` — address of the desired value, not mutated by the call.
- `weak: bool` — see `atomicCompareExchangeN`.
- `success_memmodel: AtomMemModel` — memory model on success.
- `failure_memmodel: AtomMemModel` — memory model on failure.

**Example:**

```nim
var value: int = 1
var expected: int = 1
let desired = 2
let ok = atomicCompareExchange(addr(value), addr(expected), unsafeAddr(desired),
  false, ATOMIC_SEQ_CST, ATOMIC_SEQ_CST)
echo ok, " ", value # prints true 2
```

---

### 5. `cas`

```nim
proc cas*[T: bool|int|ptr](p: ptr T; oldValue, newValue: T): bool
```

**What it does.** A platform-independent compare-and-swap wrapper: if the value at address `p` equals `oldValue`, writes `newValue` there and returns `true`; otherwise leaves memory unchanged and returns `false`. Unlike `atomicCompareExchangeN`, there's no memory-model parameter here (the strictest one, `ATOMIC_SEQ_CST`, or the corresponding `_Interlocked*`/`__sync_bool_compare_and_swap` intrinsic, is always used), and it only works with `bool`, `int`, `ptr` — the minimal set sufficient for most lock-free data structures.

- **Implementation notes.** The procedure has several mutually exclusive implementations, selected at compile time via `when`: on MSVC — through `_InterlockedCompareExchange{8,32,64}` depending on `sizeof(T)`; on tcc — through an inline assembly snippet with the `cmpxchg`/`lock` instruction; when `atomicCompareExchangeN` is available — it delegates to it with `weak = false`; otherwise — a direct `importc` binding to `__sync_bool_compare_and_swap` (an older, but widely portable GCC intrinsic). The end result is the same contract for the caller, regardless of platform.

- `p: ptr T` — the cell being compared and potentially mutated (mutated on success).
- `oldValue: T` — the expected current value, not mutated.
- `newValue: T` — the value to write on a match, not mutated.

**Example:**

```nim
var counter: int = 0
if cas(addr(counter), 0, 1):
  echo "acquired the counter, now: ", counter # prints: acquired the counter, now: 1
if not cas(addr(counter), 0, 2):
  echo "re-acquire failed, value already: ", counter
  # prints: re-acquire failed, value already: 1
```

---

## IV. Arithmetic and Bitwise Atomic Operations

### 1. `atomic*Fetch` (returns the new value)

```nim
proc atomicAddFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicSubFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicOrFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicAndFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicXorFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicNandFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**What it does.** A family of six read-modify-write procedures: each atomically applies an operation (`+`, `-`, `or`, `and`, `xor`, `nand` — bitwise AND followed by negation) to the value at `p` and the parameter `val`, stores the result back at `p`, and **returns the new** (already modified) value. All six memory models are valid for all six operations.

- **Implementation notes.** Each procedure is a thin wrapper over the like-named GCC intrinsic (`__atomic_add_fetch`, etc.), which on most architectures compiles down to a single atomic RMW processor instruction (e.g., `lock xadd` on x86 for addition) with no explicit CAS loop — meaning there's no risk of "losing a race" and having to retry.

- `p: ptr T` — the cell being mutated (mutated).
- `val: T` — the operation's second operand, not mutated.
- `mem: AtomMemModel` — memory model of the operation.

**Example:**

```nim
var flags: int = 0b0110
let newFlags = atomicOrFetch(addr(flags), 0b1000, ATOMIC_SEQ_CST)
echo newFlags # prints 14 (0b1110) — the new value after OR
```

---

### 2. `atomicFetch*` (returns the old value)

```nim
proc atomicFetchAdd*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicFetchSub*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicFetchOr*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicFetchAnd*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicFetchXor*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicFetchNand*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**What it does.** The mirror family to the previous one: the same six operations (`+`, `-`, `or`, `and`, `xor`, `nand`), but the value returned is the one that was at `p` **before** the operation was applied. Memory is mutated exactly the same way as in the `atomic*Fetch` family — the only difference is what's returned.

- **Implementation notes.** This distinction matters in practice: for example, an atomic task counter where you need to know "was I the last one" (the value before decrement was 1) requires `atomicFetchSub`, whereas getting the resulting count of remaining tasks after decrementing is a job for `atomicSubFetch`.

- `p: ptr T` — the cell being mutated (mutated).
- `val: T` — the operation's second operand, not mutated.
- `mem: AtomMemModel` — memory model of the operation.

**Example:**

```nim
var counter: int = 5
let oldValue = atomicFetchAdd(addr(counter), 3, ATOMIC_SEQ_CST)
echo oldValue, " ", counter # prints 5 8 — the old value and the cell's new state
```

---

### 3. `atomicInc`, `atomicDec`

```nim
proc atomicInc*(memLoc: var int, x: int = 1): int {.discardable.}
proc atomicDec*(memLoc: var int, x: int = 1): int {.discardable.}
```

**What it does.** Platform-independent atomic increment/decrement of an integer by `x` (default 1), returning the new value after the change. In a single-threaded build (`hasThreadSupport = false`) they degrade to a plain, non-atomic `inc`/`dec` — atomicity isn't needed where there are no threads.

- **Implementation notes.** The implementation is chosen at compile time via `when`: on GCC/Clang with thread support, `atomicAddFetch`/`atomicSubFetch` is used with `ATOMIC_SEQ_CST` (for decrement, if `atomicSubFetch` isn't available on the given platform, it's emulated via `atomicAddFetch` with a negative `x`); on MSVC — via `addAndFetch` (a wrapper over `_InterlockedExchangeAdd`) with a manual correction of the result, since `_InterlockedExchangeAdd` originally returns the *old* value, not the new one; otherwise (no thread support) — plain `inc`/`dec`. Marked `{.discardable.}`, so the returned value doesn't have to be used if only the fact of the change matters.

- `memLoc: var int` — the variable being mutated (mutated).
- `x: int` — the amount of change, defaults to `1`, not mutated.

**Example:**

```nim
var refCount: int = 1
let afterInc = atomicInc(refCount)
echo afterInc # prints 2

let afterDec = atomicDec(refCount)
echo afterDec # prints 1
```

---

### 4. `addAndFetch`

```nim
proc addAndFetch*(p: ptr int, val: int): int
```

**What it does.** An internal platform procedure (unrelated to the `AtomType` type class — it works specifically with `int`), used as a building block for `atomicInc`/`atomicDec` on MSVC and in single-threaded builds without atomic intrinsic support. Atomically adds `val` to the value at `p` and returns the new value.

- **Implementation notes.** On MSVC it's implemented via `_InterlockedExchangeAdd`/`_InterlockedExchangeAdd64` (which in the original C API return the *old* value — hence the manual correction `inc(result, x)`/`dec(result, x)` in the calling code of `atomicInc`/`atomicDec`). In the "neither GCC nor VCC" branch (e.g., a build with no thread support at all) it's a trivial non-atomic `inc(p[], val); result = p[]`, since there's no thread contention in such an environment.

- `p: ptr int` — the cell being mutated (mutated).
- `val: int` — the amount to add, not mutated.

**Example:**

```nim
var total: int = 100
let newTotal = addAndFetch(addr(total), 50)
echo newTotal # prints 150
```

---

## V. Flags, Fences, and Helper Procedures

### 1. `atomicTestAndSet`, `atomicClear`

```nim
proc atomicTestAndSet*(p: pointer, mem: AtomMemModel): bool
proc atomicClear*(p: pointer, mem: AtomMemModel)
```

**What it does.** These work with a single-byte "atomic flag" at address `p`: `atomicTestAndSet` atomically sets the byte to some implementation-defined nonzero "set" value and returns `true` if the byte was already set *before* the call (i.e., the flag was busy); `atomicClear` atomically resets the byte to `0`. The pair is used as a minimal locking primitive — "busy/free" — with no `AtomType` involved, since it operates at the level of a raw byte (`pointer`) rather than a typed value.

- **Implementation notes.** This is a direct wrapper over `__atomic_test_and_set`/`__atomic_clear` — the same intrinsics C++'s `std::atomic_flag` is built on. Only the atomicity of the flag byte itself is guaranteed, not of an arbitrary structure.

- `p: pointer` — address of the flag byte (mutated).
- `mem: AtomMemModel` — memory model of the operation (`atomicClear` only allows `ATOMIC_RELAXED`, `ATOMIC_SEQ_CST`, `ATOMIC_RELEASE`).

**Example:**

```nim
var flagByte: byte = 0
let wasSet = atomicTestAndSet(addr(flagByte), ATOMIC_ACQUIRE)
echo wasSet # prints false — the flag was free, now it's set
atomicClear(addr(flagByte), ATOMIC_RELEASE)
```

---

### 2. `atomicThreadFence`, `atomicSignalFence`, `fence`

```nim
proc atomicThreadFence*(mem: AtomMemModel)
proc atomicSignalFence*(mem: AtomMemModel)
template fence*()
```

**What it does.** All three are memory fences with no attachment to a particular cell. `atomicThreadFence` synchronizes memory visibility between threads according to `mem`, without itself performing any atomic read/write — it's a "pure" fence. `atomicSignalFence` is a narrower fence: synchronization not with other threads, but with signal handlers in the same thread (relevant only in the context of OS signal handling). `fence` is a template shorthand for the strictest variant: `atomicThreadFence(ATOMIC_SEQ_CST)`.

- **Implementation notes.** A fence with no address is a way of telling the compiler and the processor "don't reorder instructions around this point," even though no specific variable is explicitly read or written. Useful when several atomic operations have already been performed with a weaker model (`ATOMIC_RELAXED`) for speed, and a fence is needed once at the end — a single "synchronization point" instead of strengthening the model on every operation individually.

- `mem: AtomMemModel` — the required fence memory model; all six models are allowed.

**Example:**

```nim
var payload: int = 0
var published: bool = false

payload = 42                                    # prepare the data (plain write)
atomicThreadFence(ATOMIC_RELEASE)               # fence: nothing above will "leak" below
atomicStoreN(addr(published), true, ATOMIC_RELAXED)
echo payload, " ", published # prints 42 true
```

---

### 3. `cpuRelax`

```nim
proc cpuRelax*()
```

**What it does.** A hint to the processor inside a busy-wait / spin loop: "it's okay to briefly yield the pipeline here," which reduces power consumption and lets a neighboring logical core (SMT/Hyper-Threading) work more efficiently, without affecting the program's correctness.

- **Implementation notes.** On x86/amd64 it compiles down to the `pause` instruction via an inline assembly snippet (or to MSVC's `YieldProcessor` intrinsic) — by itself it doesn't free the core or switch threads, it merely reduces power draw during the short pause and lowers the penalty when exiting a speculative loop. On platforms with no dedicated instruction it falls back to a compiler fence (`asm volatile ("" ::: "memory")`), which prevents the compiler from "optimizing away" an empty wait loop.

- **Parameters:** none.

**Example:**

```nim
var lockFlag: int = 1 # imagine the lock is already held by someone
var attempts = 0
while lockFlag != 0 and attempts < 3:
  cpuRelax()   # yield the processor while waiting
  inc(attempts)
echo attempts # prints 3 — the wait loop ran the specified number of times
```

---

### 4. `atomicAlwaysLockFree`, `atomicIsLockFree`

```nim
proc atomicAlwaysLockFree*(size: int, p: pointer): bool
proc atomicIsLockFree*(size: int, p: pointer): bool
```

**What it does.** Diagnostic procedures answering the question "will atomic operations on an object of `size` bytes be implemented without locks (a mutex) on the target architecture." `atomicAlwaysLockFree` is resolved **at compile time** (and `size` must be a compile-time constant) — used to statically choose a faster code path. `atomicIsLockFree` may, if necessary, perform a runtime check (calling `__atomic_is_lock_free`) when the answer isn't known statically — for example, when lock-freedom depends on the actual alignment of the pointer `p`, known only at runtime.

- **Implementation notes.** Most architectures only guarantee lock-free operations for sizes not exceeding a machine word (typically 8 bytes on 64-bit systems), and only given proper alignment. The `p` parameter is optional in meaning (can be `nil`) and serves as a hint to the compiler about alignment — if it's unknown, the worst case is assumed.

- `size: int` — the object's size in bytes; must be known at compile time for `atomicAlwaysLockFree`.
- `p: pointer` — an optional pointer to the object (for alignment estimation), can be `nil`.

**Example:**

```nim
when atomicAlwaysLockFree(sizeof(int), nil):
  echo "atomic operations on int are lock-free on this platform"
else:
  echo "atomic operations on int may use a lock"
```

---

## VI. Practical Recipes

### 1. Atomic reference counter

```nim
type RefCounted = object
  count: int

proc retain(r: var RefCounted) =
  discard atomicInc(r.count)

proc release(r: var RefCounted): bool =
  # NOTE: atomicFetchSub returns the value BEFORE the decrement —
  # if it was 1, this call is the one that zeroed out the counter
  result = atomicFetchSub(addr(r.count), 1, ATOMIC_ACQ_REL) == 1

var obj = RefCounted(count: 1)
retain(obj)                 # count: 2
echo release(obj)           # prints false — references are still remaining
echo release(obj)           # prints true  — the last reference was dropped, safe to free the resource
```

---

### 2. One-time lazy initialization (double-checked locking)

```nim
var
  initialized: bool = false
  sharedValue: int = 0

proc ensureInitialized() =
  if atomicLoadN(addr(initialized), ATOMIC_ACQUIRE):
    return # already initialized — fast path, no locking
  # slow path: only one thread will actually run the initialization,
  # since cas atomically "claims" the false -> true transition
  if cas(addr(initialized), false, true):
    sharedValue = 12345                     # heavy initialization
    atomicThreadFence(ATOMIC_RELEASE)       # publish sharedValue for other threads

ensureInitialized()
echo sharedValue # prints 12345
```

---

### 3. Spinlock on `atomicTestAndSet`

```nim
type SpinLock = object
  flagByte: byte

proc acquire(lock: var SpinLock) =
  while atomicTestAndSet(addr(lock.flagByte), ATOMIC_ACQUIRE):
    cpuRelax() # wait without hammering the memory bus with extra write attempts

proc release(lock: var SpinLock) =
  atomicClear(addr(lock.flagByte), ATOMIC_RELEASE)

var lock: SpinLock
acquire(lock)
echo "critical section"
release(lock)
```

---

### 4. Lock-free stack (Treiber stack) on `cas`

```nim
type
  Node = ref object
    value: int
    next: Node
  LockFreeStack = object
    head: Node

proc push(s: var LockFreeStack, value: int) =
  var newNode = Node(value: value)
  var oldHead = s.head
  newNode.next = oldHead
  # retry as long as the stack's head doesn't match the one we expected:
  # if another thread changed the head between reading oldHead and the cas,
  # the cas fails and newNode.next gets refreshed to the actual head
  while not cas(addr(s.head), oldHead, newNode):
    oldHead = s.head
    newNode.next = oldHead

var stack: LockFreeStack
push(stack, 1)
push(stack, 2)
echo stack.head.value # prints 2 — the last pushed element is on top
```

---

## VII. Quick Reference Table

| Task | Procedure | Returns |
|---|---|---|
| Atomically read a value | `atomicLoadN` / `atomicLoad` | the value read |
| Atomically write a value | `atomicStoreN` / `atomicStore` | nothing |
| Write and learn what was there before | `atomicExchangeN` / `atomicExchange` | old value |
| Compare against an expected value and replace (may fail) | `atomicCompareExchangeN` / `atomicCompareExchange` | success `bool` |
| Simple cross-platform CAS for `bool`/`int`/`ptr` | `cas` | success `bool` |
| Arithmetic/bitwise op, need the result AFTER | `atomicAddFetch`, `atomicSubFetch`, `atomicOrFetch`, `atomicAndFetch`, `atomicXorFetch`, `atomicNandFetch` | new value |
| Arithmetic/bitwise op, need the result BEFORE | `atomicFetchAdd`, `atomicFetchSub`, `atomicFetchOr`, `atomicFetchAnd`, `atomicFetchXor`, `atomicFetchNand` | old value |
| Atomic ±1 (or by `x`) for `int`, cross-platform | `atomicInc` / `atomicDec` | new value |
| A single-byte busy flag (spinlock) | `atomicTestAndSet` / `atomicClear` | `bool` (was it busy) / nothing |
| Memory fence with no specific cell | `atomicThreadFence` / `fence` | nothing |
| Fence relative to signal handlers only | `atomicSignalFence` | nothing |
| Yield the processor in a busy-wait loop | `cpuRelax` | nothing |
| Check whether operations will be lock-free | `atomicAlwaysLockFree` / `atomicIsLockFree` | `bool` |

---

## VIII. Summary: Which Procedure to Choose

- Just need to safely read/write a variable from multiple threads → `atomicLoadN` / `atomicStoreN`.
- Need to know the previous value when overwriting (e.g., implementing a "first call" flag) → `atomicExchangeN`.
- Need a conditional replacement, "if it's still what I expect" → `cas` (for `bool`/`int`/`ptr`, no explicit memory model) or `atomicCompareExchangeN` (if you need control over the memory model and `weak`).
- Need a CAS inside a retry loop → `atomicCompareExchangeN` with `weak = true`; outside a loop, one-off → `weak = false`.
- Need to atomically add/subtract a number and immediately know the new total → `atomicAddFetch`/`atomicSubFetch` (or the platform-independent `atomicInc`/`atomicDec` for `int`).
- Need to know the value before the change (e.g., "was I the one who decremented the counter to zero") → `atomicFetchSub`.
- Need a simple lock/busy flag without a full mutex → `atomicTestAndSet` + `atomicClear`, paired with `cpuRelax` in the wait loop.
- Need to place "synchronization points" without tying them to a specific variable → `atomicThreadFence` or the shorthand `fence()`.
- Need to optimize a spin-wait loop without changing its logic → add `cpuRelax()` inside the loop.
- Need to pick a faster code path at compile time depending on the platform's lock-freedom → `atomicAlwaysLockFree`.
