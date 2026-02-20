# Nim Concurrency Primitives — Module Reference

> **Compilation requirements:** All modules require `--threads:on`.  
> `channels` additionally requires `--mm:arc`, `--mm:atomicArc`, or `--mm:orc`.  
> `atomics` is deprecated — prefer `std/atomics`.

This reference covers all seven concurrency modules. The sections are ordered from low-level primitives to high-level coordination patterns, mirroring the layers of a typical concurrent system.

---

## Table of Contents

1. [atomics — Lock-free atomic operations](#atomics)
2. [channels — Thread-safe typed message queues](#channels)
3. [semaphore — Counter-based access control](#semaphore)
4. [barrier — Phase synchronization for a fixed thread group](#barrier)
5. [rwlock — Readers-writer lock](#rwlock)
6. [once — One-time initialisation](#once)
7. [waitgroups — Wait for a dynamic set of workers](#waitgroups)
8. [Choosing the Right Primitive](#choosing-the-right-primitive)

---

## atomics

> **Module:** `atomics`  
> ⚠️ **Deprecated** — this is the older `std/atomics`-like interface. New code should `import std/atomics`.  
> Requires: `--threads:on`

Atomic operations act on individual memory locations without using a lock. They are the building blocks of every lock-free algorithm. The key guarantee is that no thread can ever observe the object in a half-modified state: a read or write either happens completely or not at all, from every other thread's perspective.

### Understanding Memory Ordering

Every atomic operation accepts an optional `order: Ordering` parameter that controls what non-atomic reads and writes around it may be reordered by the CPU or compiler. Getting this right is the hardest part of lock-free programming.

```
Relaxed  ──────────────────────────────────────────  Weakest (fastest)
Consume  (discouraged — use Acquire instead)
Acquire  (for loads: nothing moves before this)
Release  (for stores: nothing moves after this)
AcqRel   (Acquire + Release for read-modify-write)
SeqCst   ──────────────────────────────────────────  Strongest (default)
```

### Type: `Ordering`

```nim
type Ordering* {.pure.} = enum
  Relaxed, Consume, Acquire, Release, AcqRel, SeqCst
```

Specifies the visibility and ordering constraints that the CPU must honour around a single atomic operation. When in doubt, leave the parameter at its default `SeqCst` — it is the safest and easiest to reason about, at some performance cost.

| Value | Typical use |
|---|---|
| `Relaxed` | Independent counters where ordering relative to other data doesn't matter (e.g., statistics) |
| `Acquire` | A load that "acquires" ownership — pairs with a `Release` store elsewhere |
| `Release` | A store that publishes data — pairs with an `Acquire` load elsewhere |
| `AcqRel` | Read-modify-write operations (e.g., `fetchAdd` on a shared resource) |
| `SeqCst` | Default. All threads agree on a single total ordering of all `SeqCst` operations |

---

### Type: `Atomic[T]`

```nim
type Atomic*[T: AtomType] = distinct T
```

A wrapper around a value of type `T` that ensures all accesses are atomic. `T` must be an `AtomType` — in practice this means integers, booleans, and pointer-sized types. You cannot atomically operate on complex structs.

```nim
var counter: Atomic[int]
var flag: Atomic[bool]
```

---

### `load`

```nim
proc load*[T](location: var Atomic[T]; order: Ordering = SeqCst): T
```

Reads the current value of the atomic object and returns it. The `order` parameter must be `Relaxed`, `Consume`, `Acquire`, or `SeqCst` — it cannot be `Release` or `AcqRel` (those apply to stores/RMWs, not loads).

```nim
var x: Atomic[int]
x.store(42)
echo x.load()           # 42, with full SeqCst ordering
echo x.load(Relaxed)    # 42, cheapest read when you don't need ordering
```

---

### `store`

```nim
proc store*[T](location: var Atomic[T]; desired: T; order: Ordering = SeqCst)
```

Atomically writes `desired` into the atomic object. Valid orderings: `Relaxed`, `Release`, `SeqCst`.

```nim
var flag: Atomic[bool]
flag.store(true)           # publish with SeqCst
flag.store(false, Release) # release: signals that preceding writes are visible
```

---

### `exchange`

```nim
proc exchange*[T](location: var Atomic[T]; desired: T; order: Ordering = SeqCst): T
```

Atomically replaces the current value with `desired` and returns the **old** value. Useful for taking ownership of a resource.

```nim
var x: Atomic[int]
x.store(10)
let old = x.exchange(99)
assert old == 10
assert x.load() == 99
```

---

### `compareExchange`

```nim
proc compareExchange*[T](location: var Atomic[T]; expected: var T; desired: T;
    order: Ordering = SeqCst): bool

proc compareExchange*[T](location: var Atomic[T]; expected: var T; desired: T;
    success, failure: Ordering): bool
```

The fundamental building block of lock-free algorithms. Atomically does the following as one indivisible step:

1. Compare the current value of `location` with `expected`.
2. **If they match:** write `desired` into `location`, return `true`. `expected` is unchanged.
3. **If they don't match:** load the current value of `location` into `expected`, return `false`. `location` is unchanged.

The two-ordering overload lets you specify separate orderings for the success and failure paths, which can be more efficient.

```nim
var x: Atomic[int]
x.store(5)

var expected = 5
if x.compareExchange(expected, 100):
  echo "swapped: x is now 100"
else:
  echo "failed: x was", expected, "not 5"

# Typical CAS loop pattern — spin until we succeed:
var cur = x.load(Relaxed)
while not x.compareExchange(cur, cur * 2, Relaxed, Relaxed):
  discard  # cur was updated with the actual value; retry
```

---

### `compareExchangeWeak`

```nim
proc compareExchangeWeak*[T](location: var Atomic[T]; expected: var T;
    desired: T; order: Ordering = SeqCst): bool

proc compareExchangeWeak*[T](location: var Atomic[T]; expected: var T;
    desired: T; success, failure: Ordering): bool
```

Same semantics as `compareExchange`, but is allowed to **spuriously fail** even when the current value equals `expected`. This maps directly to the hardware's `LL/SC` (load-linked/store-conditional) instruction on architectures like ARM, which can occasionally fail for microarchitectural reasons unrelated to contention.

In a retry loop, `compareExchangeWeak` is often faster than the strong variant because it avoids an extra comparison on those architectures. Use the strong `compareExchange` when a single attempt is made (no loop).

```nim
var cur = x.load(Relaxed)
while not x.compareExchangeWeak(cur, cur + 1, AcqRel, Relaxed):
  discard  # spurious or real failure; cur updated; retry
```

---

### `fetchAdd`

```nim
proc fetchAdd*[T: SomeInteger](location: var Atomic[T]; value: T;
    order: Ordering = SeqCst): T
```

Atomically adds `value` to the integer and returns the **original** (pre-addition) value. This is the canonical way to generate unique IDs or indices across threads without a lock.

```nim
var idGen: Atomic[int]
let myId = idGen.fetchAdd(1)  # returns 0, 1, 2, ... per caller
```

---

### `fetchSub`

```nim
proc fetchSub*[T: SomeInteger](location: var Atomic[T]; value: T;
    order: Ordering = SeqCst): T
```

Atomically subtracts `value` and returns the original value. Commonly used in reference-counting: when the returned value reaches 1 (meaning it just became 0 after the subtraction), the caller is responsible for cleanup.

```nim
var refCount: Atomic[int]
refCount.store(3)
if refCount.fetchSub(1) == 1:  # was 1, now 0
  echo "last reference — free the object"
```

---

### `fetchAnd`

```nim
proc fetchAnd*[T: SomeInteger](location: var Atomic[T]; value: T;
    order: Ordering = SeqCst): T
```

Atomically replaces the value with `location AND value` and returns the original. Used to atomically clear specific bits.

```nim
var flags: Atomic[uint32]
flags.store(0b1111'u32)
let before = flags.fetchAnd(0b1100'u32)  # clear bits 0 and 1
assert flags.load() == 0b1100'u32
```

---

### `fetchOr`

```nim
proc fetchOr*[T: SomeInteger](location: var Atomic[T]; value: T;
    order: Ordering = SeqCst): T
```

Atomically replaces the value with `location OR value` and returns the original. Used to atomically set specific bits.

```nim
var flags: Atomic[uint32]
flags.store(0b0000'u32)
discard flags.fetchOr(0b0011'u32)  # set bits 0 and 1
assert flags.load() == 0b0011'u32
```

---

### `fetchXor`

```nim
proc fetchXor*[T: SomeInteger](location: var Atomic[T]; value: T;
    order: Ordering = SeqCst): T
```

Atomically replaces the value with `location XOR value` and returns the original. Used to atomically toggle specific bits.

```nim
var flags: Atomic[uint32]
flags.store(0b1010'u32)
discard flags.fetchXor(0b1100'u32)
assert flags.load() == 0b0110'u32
```

---

### `atomicInc`

```nim
proc atomicInc*[T: SomeInteger](location: var Atomic[T]; value: T = 1)
```

Atomically increments `location` by `value` (default 1). A convenience wrapper around `fetchAdd` that discards the return value. Use when you don't need to know the value before the increment.

```nim
var hits: Atomic[int]
hits.atomicInc()       # += 1
hits.atomicInc(5)      # += 5
```

---

### `atomicDec`

```nim
proc atomicDec*[T: SomeInteger](location: var Atomic[T]; value: T = 1)
```

Atomically decrements `location` by `value` (default 1). Convenience wrapper around `fetchSub`.

```nim
var remaining: Atomic[int]
remaining.store(10)
remaining.atomicDec()
```

---

### `+=` and `-=`

```nim
proc `+=`*[T: SomeInteger](location: var Atomic[T]; value: T)
proc `-=`*[T: SomeInteger](location: var Atomic[T]; value: T)
```

Operator syntax for `atomicInc` and `atomicDec`. Particularly readable in sequential code dealing with atomic counters.

```nim
var score: Atomic[int]
score += 10
score -= 3
echo score.load()  # 7
```

---

## channels

> **Module:** `channels`  
> ⚠️ **Experimental** — interface may change.  
> Requires: `--threads:on --mm:arc` (or `--mm:atomicArc` or `--mm:orc`)

A channel is a typed, thread-safe FIFO queue. It enables the "communicating sequential processes" (CSP) style of concurrency: instead of sharing memory protected by a lock, threads own their data exclusively and pass ownership through the channel. The `Isolated[T]` wrapper (from `std/isolation`) enforces this at the type level — data passed through a channel is **moved**, not copied.

`Chan[T]` is a reference-counted shared handle. You can copy it freely and pass copies to threads — all copies refer to the same underlying queue. The queue is freed when the last copy is destroyed.

---

### `newChan`

```nim
proc newChan*[T](elements: Positive = 30): Chan[T]
```

Creates and initialises a new channel with a buffer capacity of `elements` messages. This is the **only** way to create a valid `Chan[T]`. The internal queue is allocated in shared memory so it can be accessed from any thread.

`elements` is the maximum number of messages the queue can hold simultaneously. If all slots are occupied:
- `send` blocks until a receiver frees a slot.
- `trySend` returns `false` immediately.

```nim
var ch = newChan[string]()         # capacity 30
var smallCh = newChan[int](4)      # capacity 4 — strict back-pressure
```

---

### `send`

```nim
proc send*[T](c: Chan[T], src: sink Isolated[T])
template send*[T](c: Chan[T]; src: T)
```

Sends a message into the channel. **Blocks** if the channel is full, waiting until a receiver removes a message to make room. The data is **moved** into the channel — do not use `src` after sending.

The template form wraps `src` in `isolate()` automatically, which verifies at compile time that the value has no references to thread-local data. Use the template form for simple values and small objects.

```nim
var ch = newChan[string]()

# From a worker thread:
ch.send("hello")            # template form — most convenient

# For objects that cannot be copied, use explicit isolation:
import std/isolation
ch.send(isolate(myString))
```

---

### `recv`

```nim
proc recv*[T](c: Chan[T], dst: var T)
proc recv*[T](c: Chan[T]): T
```

Receives a message from the channel. **Blocks** if the channel is empty, waiting until a sender puts a message in. Two forms are provided:

- **In-place form** (`dst: var T`): writes directly into an existing variable.
- **Return form**: allocates a new value and returns it. Slightly more ergonomic but requires an extra move in some cases.

```nim
var ch = newChan[string]()
ch.send("world")

var msg: string
ch.recv(msg)          # in-place form
echo msg              # "world"

let msg2 = ch.recv()  # return form
```

---

### `recvIso`

```nim
proc recvIso*[T](c: Chan[T]): Isolated[T]
```

Like `recv`, but returns the value wrapped in `Isolated[T]`. Useful when you intend to forward the received value into another channel or another isolation boundary without unwrapping it first.

```nim
var ch = newChan[string]()
ch.send("data")
let iso = ch.recvIso()
# can be forwarded directly to another channel:
# anotherChan.send(iso)
```

---

### `trySend`

```nim
proc trySend*[T](c: Chan[T], src: sink Isolated[T]): bool
template trySend*[T](c: Chan[T], src: T): bool
```

Attempts to send a message without blocking. Returns `true` if the message was placed in the channel, `false` if the channel was full. The data is moved on success; on failure the caller retains ownership of `src`.

In high-contention scenarios, consider an **exponential backoff** loop to reduce cache-line thrashing.

```nim
var ch = newChan[int](2)
assert ch.trySend(1) == true
assert ch.trySend(2) == true
assert ch.trySend(3) == false  # full
```

---

### `tryTake`

```nim
proc tryTake*[T](c: Chan[T], src: var Isolated[T]): bool
```

Non-blocking send variant that operates directly on an `Isolated[T]` variable, moving it without any copying. Suitable for types that cannot be copied (i.e., those with a deleted `=copy` hook). Returns `true` on success.

```nim
import std/isolation
var ch = newChan[string](5)
var iso = isolate("non-copyable data")
if ch.tryTake(iso):
  echo "sent"
```

---

### `tryRecv`

```nim
proc tryRecv*[T](c: Chan[T], dst: var T): bool
```

Attempts to receive a message without blocking. Returns `true` and fills `dst` with the received value if a message was available; returns `false` and leaves `dst` unchanged otherwise.

The canonical pattern for a polling consumer: do useful work between attempts rather than spin-waiting.

```nim
var ch = newChan[string]()
var msg: string
while not ch.tryRecv(msg):
  doOtherWork()
echo "got:", msg
```

---

### `peek`

```nim
proc peek*[T](c: Chan[T]): int
```

Returns an **estimate** of the number of messages currently in the channel. Because other threads may be sending or receiving concurrently, the value may be outdated by the time you read it. Do not use `peek` for control flow that must be correct (use `tryRecv` instead). It is useful for monitoring, logging, and back-pressure heuristics.

```nim
var ch = newChan[int](10)
ch.send(1); ch.send(2)
echo ch.peek()  # probably 2, but not guaranteed
```

---

## semaphore

> **Module:** `semaphore`  
> Requires: `--threads:on`

A semaphore is a generalised lock. Where a mutex allows exactly one thread at a time, a semaphore allows **up to N threads simultaneously**, controlled by a counter. When the counter reaches zero, `wait` blocks until another thread calls `signal`, restoring a unit.

Two canonical uses:
- **Counting semaphore** (N > 1): limit concurrent access to a pool of resources (e.g., database connections, worker slots).
- **Binary semaphore** (N = 1): a simple signalling mechanism — one thread waits, another signals. Unlike a mutex, the thread that signals need not be the one that waited.

---

### Type: `Semaphore`

```nim
type Semaphore* = object
```

Holds the counter, a mutex, and a condition variable. Cannot be copied or moved — it must stay at a fixed memory address once created.

---

### `createSemaphore`

```nim
proc createSemaphore*(count: Natural = 0): Semaphore
```

Creates a semaphore initialised to `count`. This is the only valid way to obtain a `Semaphore`.

- `count = 0` — the first thread to call `wait` will block immediately. Use this as a one-shot notification: the waiting thread is released only after some other thread calls `signal`.
- `count = N` — up to N threads can `wait` and proceed without blocking. Use this to limit concurrency to N simultaneous operations.

```nim
var pool = createSemaphore(3)  # allow at most 3 concurrent workers
var gate = createSemaphore(0)  # barrier: wait until someone signals
```

---

### `wait` (Semaphore)

```nim
proc wait*(s: var Semaphore)
```

Decrements the semaphore counter by 1. If the counter is already 0, the calling thread blocks until another thread calls `signal`. Once unblocked, the thread decrements the counter and proceeds.

```nim
# Worker: claim a slot from the pool
pool.wait()
try:
  doExpensiveWork()
finally:
  pool.signal()  # release the slot
```

---

### `signal`

```nim
proc signal*(s: var Semaphore)
```

Increments the semaphore counter by 1 and wakes one waiting thread (if any). Signal can be called from a different thread than the one that called `wait` — this is what distinguishes a semaphore from a mutex.

```nim
# After finishing a task, release one slot back to the pool:
pool.signal()

# Signal a waiting thread that an event has occurred:
gate.signal()
```

---

## barrier

> **Module:** `barrier`  
> Requires: `--threads:on`

A barrier is a meeting point for a fixed, pre-declared number of threads. Every thread that calls `wait` on the barrier blocks until **all** of the declared number of threads have called `wait`. At that point, all threads are released simultaneously and the barrier automatically **resets**, ready for the next round. This is called a *cyclic barrier*.

Barriers are ideal for phased algorithms where every thread must complete phase N before any thread starts phase N+1.

---

### Type: `Barrier`

```nim
type Barrier* = object
```

Holds an internal counter, a lock, and a condition variable. Cannot be copied or moved.

---

### `createBarrier`

```nim
proc createBarrier*(parties: Natural): Barrier
```

Creates a barrier that requires `parties` threads to call `wait` before any are released. `parties` must match the exact number of participating threads and must not change after creation.

```nim
var b = createBarrier(4)  # 4 threads must all arrive before any continue
```

---

### `wait` (Barrier)

```nim
proc wait*(b: var Barrier)
```

Signals that the calling thread has reached the barrier and then blocks until all other threads have done the same. The last thread to arrive resets the barrier and wakes all waiting threads. Every thread then returns from `wait` and continues.

The barrier's cyclic nature means you can call `wait` in a loop, synchronising on every iteration without needing to recreate the barrier.

```nim
var b = createBarrier(10)

proc phaseWorker(id: int) =
  for phase in 0..<100:
    doPhaseWork(id, phase)
    b.wait()  # ensure everyone finishes before the next phase begins
```

---

## rwlock

> **Module:** `rwlock`  
> Requires: `--threads:on`

A readers-writer lock (or shared-exclusive lock) distinguishes between two types of access:

- **Readers** hold a *shared* lock. Any number of readers may hold the lock simultaneously.
- **Writers** hold an *exclusive* lock. While a writer holds the lock, no readers or other writers may proceed.

This is perfect for data structures that are read frequently but modified rarely. A plain mutex would serialise all readers unnecessarily; an RwLock allows all concurrent reads to proceed in parallel, serialising only when a write occurs.

The implementation is *writer-preferring*: new readers block if a writer is waiting, preventing writer starvation.

---

### Type: `RwLock`

```nim
type RwLock* = object
```

Holds a condition variable, a mutex, reader count, waiting-writer count, and a writer-active flag. Cannot be copied or moved.

---

### `createRwLock`

```nim
proc createRwLock*(): RwLock
```

Creates and initialises a readers-writer lock. The only valid way to create an `RwLock`.

```nim
var rw = createRwLock()
var sharedData = 0
```

---

### `beginRead`

```nim
proc beginRead*(rw: var RwLock)
```

Acquires the shared (read) lock. Blocks if a writer is currently active or waiting. Once acquired, other readers may also hold the lock concurrently.

Always pair with `endRead`. If an exception might occur between the two, use the `readWith` template instead.

```nim
rw.beginRead()
let snapshot = sharedData  # safe to read alongside other readers
rw.endRead()
```

---

### `endRead`

```nim
proc endRead*(rw: var RwLock)
```

Releases the shared (read) lock. If this was the last active reader, any waiting writer is woken.

---

### `beginWrite`

```nim
proc beginWrite*(rw: var RwLock)
```

Acquires the exclusive (write) lock. Blocks until all current readers have finished and any active writer has released. Once acquired, the calling thread is the sole holder.

Always pair with `endWrite`. If an exception might occur, use `writeWith`.

```nim
rw.beginWrite()
sharedData += 1  # exclusive access — no readers or other writers possible
rw.endWrite()
```

---

### `endWrite`

```nim
proc endWrite*(rw: var RwLock)
```

Releases the exclusive (write) lock and wakes all waiting threads (readers and writers), letting them compete for the lock.

---

### `readWith`

```nim
template readWith*(a: RwLock, body: untyped)
```

Convenience template: acquires the read lock, executes `body`, and releases the lock in a `finally` block so the lock is always released even if `body` raises an exception. Equivalent to a `try/finally` wrapping `beginRead`/`endRead`, but more concise and harder to get wrong.

```nim
readWith(rw):
  echo sharedData  # guaranteed read access; lock released on exit
```

---

### `writeWith`

```nim
template writeWith*(a: RwLock, body: untyped)
```

Same as `readWith` but for the exclusive write lock.

```nim
writeWith rw:
  sharedData += 1  # exclusive; lock released on exit even if exception thrown
```

---

## once

> **Module:** `once`  
> Requires: `--threads:on`

`Once` ensures that a block of code runs **exactly once** across all threads, no matter how many threads call it concurrently. All other threads that reach the `once` block while the first thread is still executing it will block and wait. Once the block completes, all threads proceed with the guarantee that every side-effect of the block is fully visible.

This is the canonical pattern for lazy, thread-safe singleton initialisation.

**Exception behaviour:** If the executing thread's `body` raises an exception, the `Once` is reset to its initial state. The next thread to call `once` will attempt the body again. This means "try until someone succeeds", which is the correct semantic for initialisation that may fail transiently.

---

### Type: `Once`

```nim
type Once* = object
```

Holds a small state machine (`Unset`/`Pending`/`Complete`), a mutex, and a condition variable. Cannot be copied or moved.

---

### `createOnce`

```nim
proc createOnce*(): Once
```

Creates and initialises a `Once` object in the `Unset` state. The only valid way to obtain a `Once`.

```nim
var o = createOnce()
```

---

### `once` (template)

```nim
template once*(o: Once, body: untyped)
```

Executes `body` exactly once. The logic is:

1. If the `Once` is already `Complete`, skip the body immediately and return.
2. If another thread is currently executing the body (`Pending`), block until it finishes, then also skip and return.
3. If the `Once` is `Unset`, this thread executes the body:
   - On success: mark as `Complete`, wake all waiting threads.
   - On exception: reset to `Unset`, wake all waiting threads, re-raise the exception.

```nim
var o = createOnce()
var expensiveResource: ptr MyType = nil

proc getResource(): ptr MyType =
  once(o):
    expensiveResource = allocAndInitialise()
  result = expensiveResource
```

No matter how many threads call `getResource()` simultaneously, `allocAndInitialise()` is called exactly once.

---

## waitgroups

> **Module:** `waitgroups`  
> Requires: `--threads:on`

A `WaitGroup` lets a coordinator thread wait until a dynamic set of worker threads have all signalled completion. Unlike a `Barrier` (which requires knowing the exact thread count up front), a `WaitGroup` allows you to add workers one at a time and wait for all of them.

The flow is:
1. Call `enter(n)` to announce that n workers are about to start.
2. Spawn the workers. Each worker calls `leave()` when done.
3. The coordinator calls `wait()`, which blocks until the internal counter reaches zero.

---

### Type: `WaitGroup`

```nim
type WaitGroup* = object
```

Holds an internal counter (the number of outstanding workers), a mutex, and a condition variable. Cannot be copied or moved.

---

### `createWaitGroup`

```nim
proc createWaitGroup*(): WaitGroup
```

Creates a `WaitGroup` with an internal counter of 0. The only valid way to create a `WaitGroup`.

```nim
var wg = createWaitGroup()
```

---

### `enter`

```nim
proc enter*(b: var WaitGroup; delta: Natural = 1)
```

Increases the outstanding-worker counter by `delta` (default 1). Call `enter` **before** spawning the corresponding workers, so that `wait` cannot return prematurely if a worker finishes before the coordinator calls `wait`.

```nim
wg.enter(10)        # announce 10 workers are coming
for i in 0..<10:
  createThread(threads[i], worker, i)
```

---

### `leave`

```nim
proc leave*(b: var WaitGroup)
```

Decrements the counter by 1. Should be called by each worker when its task is complete. If the counter reaches 0, any thread blocked in `wait` is woken immediately.

```nim
proc worker(i: int) =
  doWork(i)
  wg.leave()  # signal completion
```

---

### `wait` (WaitGroup)

```nim
proc wait*(b: var WaitGroup)
```

Blocks the calling thread until the counter reaches 0, meaning all workers that called `enter` have also called `leave`. If the counter is already 0 when `wait` is called, it returns immediately.

```nim
wg.wait()             # block until all 10 workers are done
for x in results:
  echo x              # safe to read — all workers have finished
```

---

## Choosing the Right Primitive

| Problem | Best fit |
|---|---|
| Shared integer counter between threads | `Atomic[int]` |
| Unique ID generation | `fetchAdd` |
| Reference counting | `fetchSub` |
| Pass typed data between threads with ownership transfer | `Chan[T]` |
| Limit concurrent access to N resources | `Semaphore(N)` |
| Signal from one thread to another (no shared data needed) | `Semaphore(0)` |
| Synchronise N threads between fixed phases | `Barrier(N)` |
| Read-heavy shared data structure | `RwLock` |
| Thread-safe lazy singleton / one-time initialisation | `Once` |
| Wait for a dynamic group of workers to finish | `WaitGroup` |

### Key behavioural summary

| Primitive | Blocks caller? | Reusable? | Direction |
|---|---|---|---|
| `Atomic` | Never | Yes | N/A — direct memory access |
| `Chan.send` | Yes (if full) | Yes | Producer → Consumer |
| `Chan.trySend` | Never | Yes | Producer → Consumer |
| `Semaphore.wait` | Yes (if counter = 0) | Yes | Any → Any |
| `Barrier.wait` | Yes (until all arrive) | Yes (cyclic) | All ↔ All |
| `RwLock.beginRead` | Only if writer active/waiting | Yes | — |
| `RwLock.beginWrite` | Yes (until no readers/writers) | Yes | — |
| `Once.once` | Only if body in progress | No (one-shot) | — |
| `WaitGroup.wait` | Yes (until counter = 0) | Yes (reset by `enter`) | Coordinator ← Workers |
