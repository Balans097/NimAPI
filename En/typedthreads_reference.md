# `typedthreads` Module Reference

> **Module purpose:** `typedthreads` is the core low-level thread management module in Nim's standard library. It provides the `Thread[TArg]` type and the procedures needed to create, inspect, synchronise, and bind threads to CPU cores. All threads in Nim programs ultimately rest on the primitives defined here.
>
> 💡 **Recommendation from the module itself:** For application-level concurrency, prefer higher-level libraries such as [`malebolgia`](https://github.com/araq/malebolgia), [`taskpools`](https://github.com/status-im/nim-taskpools), or [`weave`](https://github.com/mratsim/weave). Use `typedthreads` directly when you need fine-grained control or are building your own concurrency abstraction.

---

## Critical Memory Safety Rules

Before using threads in Nim, two rules must be understood:

**Rule 1 — No sharing of GC-managed memory across threads (except with `arc`/`orc`/`boehm`).**
Nim's default garbage collector (`orc`) supports shared heaps, but older strategies like `refc` use per-thread local heaps. Passing `ref` values, `seq`, `string`, or other GC-managed objects between threads compiled with `refc` is **undefined behaviour**. Use global variables, raw pointers, or memory allocated with `allocShared`.

**Rule 2 — Thread arguments are copied.**
The `param` argument passed to `createThread` is copied into the thread's storage. Mutations to the argument inside the thread do not affect the original. To share mutable state, pass a `ptr` to shared memory.

---

## Type Reference

### `Thread[TArg]`

```nim
type Thread[TArg] = object  # re-exported from std/private/threadtypes
```

Represents a single OS thread. It is a generic type parameterised by `TArg` — the type of data passed to the thread's entry procedure. `TArg` can be `void` if the thread needs no argument.

A `Thread` value must be declared as a `var` (mutable). It should be treated as an opaque handle: do not copy or move it after `createThread` has been called on it.

```nim
var t: Thread[int]                    # thread receiving a single int
var t2: Thread[tuple[a, b: int]]      # thread receiving a tuple
var t3: Thread[void]                  # thread with no argument
```

`Thread` is exported from `std/private/threadtypes` — you do not need to import that module separately.

---

## Procedure Reference

---

### `createThread` (with argument)

```nim
proc createThread*[TArg](t: var Thread[TArg],
                          tp: proc (arg: TArg) {.thread, nimcall.},
                          param: TArg)
```

#### What it does

Creates a new OS thread, stores its handle in `t`, and immediately starts executing `tp(param)` concurrently. The call returns as soon as the thread is successfully launched — it does not wait for `tp` to finish.

`param` is **copied** into the thread's private storage before the thread begins. The entry procedure `tp` receives this copy as its argument.

If the OS cannot create the thread (e.g. due to resource exhaustion), `ResourceExhaustedError` is raised.

#### The `{.thread.}` pragma

The entry procedure `tp` **must** be annotated with `{.thread.}`. This pragma:
- Initialises the thread-local GC state (stack bottom, GC roots) when the thread starts.
- Marks the procedure as having the correct calling convention for the platform (`stdcall` on Windows, `noconv` on POSIX).
- Prevents the procedure from being called as a normal Nim proc (the compiler enforces this).

Forgetting `{.thread.}` on `tp` is a compile-time error.

#### Parameter summary

| Parameter | Type | Description |
|---|---|---|
| `t` | `var Thread[TArg]` | The thread object to initialise. Must be declared as `var`. |
| `tp` | `proc (arg: TArg) {.thread, nimcall.}` | The entry procedure. Must carry `{.thread.}`. |
| `param` | `TArg` | The argument passed to `tp`. Copied into thread storage. |

#### Examples

Spawning a thread that receives a plain value:

```nim
import std/typedthreads

proc worker(n: int) {.thread.} =
  echo "Thread received: ", n

var t: Thread[int]
createThread(t, worker, 42)
joinThread(t)
```

Spawning multiple threads with tuples as arguments:

```nim
import std/typedthreads, std/locks

var L: Lock
initLock(L)

proc rangeEcho(interval: tuple[a, b: int]) {.thread.} =
  for i in interval.a .. interval.b:
    withLock L: echo i

var threads: array[5, Thread[tuple[a, b: int]]]
for i in 0 .. high(threads):
  createThread(threads[i], rangeEcho, (i * 10, i * 10 + 5))
joinThreads(threads)
deinitLock(L)
```

Sharing mutable data via a pointer (requires `arc` or `orc`):

```nim
import std/typedthreads, std/locks

var L: Lock
initLock(L)

proc accumulate(s: ptr seq[int]) {.thread.} =
  withLock L:
    for i in 0 ..< 10:
      s[].add(i)

var data = newSeq[int]()
var threads: array[3, Thread[ptr seq[int]]]
for i in 0 .. high(threads):
  createThread(threads[i], accumulate, data.addr)
joinThreads(threads)
echo data
deinitLock(L)
```

---

### `createThread` (void, no argument)

```nim
proc createThread*(t: var Thread[void], tp: proc () {.thread, nimcall.})
```

#### What it does

Convenience overload for threads that need no argument at all. Equivalent to `createThread[void](t, tp)` but avoids having to write the `[void]` type parameter explicitly.

#### Example

```nim
import std/typedthreads

proc background() {.thread.} =
  echo "Running in background with no argument"

var t: Thread[void]
createThread(t, background)
joinThread(t)
```

---

### `joinThread`

```nim
proc joinThread*[TArg](t: Thread[TArg]) {.inline.}
```

#### What it does

Blocks the calling thread until thread `t` has finished executing. After `joinThread` returns, all writes made by `t` are visible to the caller (a happens-before guarantee is established).

Internally this calls `WaitForSingleObject` on Windows and `pthread_join` on POSIX.

#### When to use it

Always join threads you have created before they (or their associated data) go out of scope. Failing to join a thread that is still running when its `Thread` object is destroyed is undefined behaviour.

#### Example

```nim
import std/typedthreads

proc slowWork(n: int) {.thread.} =
  for i in 0 ..< n: discard

var t: Thread[int]
createThread(t, slowWork, 1_000_000)
# ... do other things here ...
joinThread(t)  # wait for slowWork to complete
echo "Thread finished"
```

---

### `joinThreads`

```nim
proc joinThreads*[TArg](t: varargs[Thread[TArg]])
```

#### What it does

Waits for **every** thread in the `varargs` list to finish. Equivalent to calling `joinThread` on each thread individually, but on Windows it uses `WaitForMultipleObjects` in batches of up to 64 threads, which can be more efficient than sequential joins.

#### Windows batch limit

The Windows API `WaitForMultipleObjects` handles at most 64 handles at a time (`MAXIMUM_WAIT_OBJECTS = 64`). `joinThreads` automatically batches larger arrays, so you can safely pass arrays of any size.

#### Example

```nim
import std/typedthreads

proc task(id: int) {.thread.} =
  echo "Task ", id, " done"

var workers: array[10, Thread[int]]
for i in 0 .. high(workers):
  createThread(workers[i], task, i)

joinThreads(workers)  # waits for all 10 threads
echo "All tasks complete"
```

---

### `running`

```nim
proc running*[TArg](t: Thread[TArg]): bool {.inline.}
```

#### What it does

Returns `true` if thread `t` is currently running (i.e. its entry procedure has been set and has not yet returned). Returns `false` if the thread has not been started or has already finished.

The check is not a synchronisation point — it is a lightweight poll of the thread's internal `dataFn` field. Because of scheduling and memory visibility, a result of `false` does not guarantee the thread's side effects are visible to the caller; use `joinThread` for that guarantee.

#### Example

```nim
import std/typedthreads

proc longTask(n: int) {.thread.} =
  for i in 0 ..< n: discard

var t: Thread[int]
createThread(t, longTask, 100_000_000)

if running(t):
  echo "Thread is still going..."

joinThread(t)
echo "Now it is done. running = ", running(t)  # false
```

---

### `handle`

```nim
proc handle*[TArg](t: Thread[TArg]): SysThread {.inline.}
```

#### What it does

Returns the underlying OS thread handle (`SysThread`) for thread `t`. On Windows this is a `HANDLE`; on POSIX it is a `pthread_t`.

This is an **escape hatch** for interoperating with platform-specific threading APIs not covered by this module — for example, setting real-time scheduling priorities, querying thread IDs via OS calls, or passing the handle to a third-party C library.

Under normal circumstances you should not need `handle`. If you find yourself reaching for it, consider whether a higher-level library covers your use case.

#### Example

```nim
import std/typedthreads

proc work(n: int) {.thread.} =
  for i in 0 ..< n: discard

var t: Thread[int]
createThread(t, work, 1_000_000)

let h = handle(t)
# h is now a SysThread (HANDLE on Windows, pthread_t on POSIX)
# pass to platform-specific OS calls if needed...

joinThread(t)
```

---

### `pinToCpu`

```nim
proc pinToCpu*[Arg](t: var Thread[Arg]; cpu: Natural)
```

#### What it does

Sets the **CPU affinity** of thread `t` to the logical CPU core identified by `cpu` (zero-indexed). After this call, the OS scheduler will only run the thread on that specific core.

Pinning a thread to a core can improve cache locality (the thread's working set stays on one core's L1/L2 cache) and reduce jitter in latency-sensitive applications. Used carelessly, it can also interfere with the OS scheduler's load balancing and hurt throughput.

#### Platform behaviour

| Platform | Mechanism | Notes |
|---|---|---|
| Windows | `SetThreadAffinityMask` | `cpu` maps to a single bit in the affinity mask |
| Linux | `pthread_setaffinity_np` via `cpuset` | Full `cpu_set_t` manipulation |
| macOS | No-op | macOS does not expose thread affinity at this level |
| Genode | No-op (hint emitted) | Affinity is set at thread creation on Genode, not after |

#### Example

```nim
import std/typedthreads

proc worker(n: int) {.thread.} =
  for i in 0 ..< n: discard

var t: Thread[int]
createThread(t, worker, 500_000_000)
pinToCpu(t, 0)   # pin to core 0
joinThread(t)
```

> ⚠️ Only use `pinToCpu` if you have measured that cache effects or scheduling jitter are a real problem. Gratuitous pinning often *hurts* performance by preventing the OS from moving threads to idle cores.

---

## Thread Stack Configuration

Stack sizes are controlled by internal constants. On resource-constrained targets (Zephyr, FreeRTOS, NuttX, 8-bit/16-bit CPUs) they can be overridden at compile time:

| Compile-time define | Default | Description |
|---|---|---|
| `-d:nimThreadStackSize=N` | `8192` | Total stack size in bytes for embedded/RTOS targets |
| `-d:nimThreadStackGuard=N` | `128` | Guard page size in bytes for embedded/RTOS targets |

On standard desktop/server platforms the stack size is approximately **2 MB** per thread on 64-bit systems (1 MB on 32-bit), minus a 4 KB guard page.

---

## Memory Management and Threads

| Strategy | Shared heap? | Passing `ref`/`seq`/`string`? | Notes |
|---|---|---|---|
| `orc` (default) | ✅ Yes | ✅ Safe | Reference-counted with cycle detection |
| `arc` | ✅ Yes | ✅ Safe | Reference-counted, no cycle detection |
| `boehm` | ✅ Yes | ✅ Safe | Conservative GC |
| `refc` | ❌ No | ❌ Unsafe | Local heaps per thread; use `ptr` + `allocShared` |

---

## Complete Example

```nim
import std/typedthreads, std/locks

var
  results: seq[int]
  L: Lock

initLock(L)

proc computeRange(bounds: tuple[lo, hi: int]) {.thread.} =
  var localSum = 0
  for i in bounds.lo .. bounds.hi:
    localSum += i
  withLock L:
    results.add(localSum)

const nThreads = 4
var threads: array[nThreads, Thread[tuple[lo, hi: int]]]

for i in 0 ..< nThreads:
  createThread(threads[i], computeRange, (lo: i * 250, hi: (i + 1) * 250 - 1))
  pinToCpu(threads[i], i)

joinThreads(threads)
deinitLock(L)

echo "Partial sums: ", results
```

---

## Summary Table

| Procedure | Purpose | Blocking? |
|---|---|---|
| `createThread` | Launch a new thread with a typed argument | No |
| `createThread` (void) | Launch a thread with no argument | No |
| `joinThread` | Wait for one thread to finish | Yes |
| `joinThreads` | Wait for all threads in an array to finish | Yes |
| `running` | Poll whether a thread is still executing | No |
| `handle` | Get the raw OS thread handle | No |
| `pinToCpu` | Bind a thread to a specific CPU core | No |

---

## Notes

- `Thread` objects must not be copied after `createThread` is called. Treat them as unique, non-movable handles for the lifetime of the thread.
- There is no public `destroyThread` procedure. The implementation exists in source but is disabled (`when false`) because forceful thread termination is inherently unsafe. Design threads to exit naturally.
- For inter-thread communication, use `Channel` from `system` or a higher-level primitive. Never read or write a global mutable variable from multiple threads without a `Lock` — doing so is a data race.
- This module is not available when compiling with `--os:any`. Attempting to import it under that target produces a compile-time error.
