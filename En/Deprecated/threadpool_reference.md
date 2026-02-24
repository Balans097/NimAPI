# Threadpool Module — Reference

> ⚠️ **Deprecation notice.** This module is officially deprecated. The Nim team recommends using the Nimble packages **[malebolgia](https://github.com/araq/malebolgia)**, **[taskpools](https://github.com/status-im/nim-taskpools)**, or **[weave](https://github.com/mratsim/weave)** instead. The API is marked as unstable.

> 🔧 **Compilation requirement.** The module requires `--threads:on` to be passed to the compiler. Without this flag, compilation fails with an explicit error.

---

## What This Module Does

`threadpool` implements Nim's high-level task-based parallelism. Rather than managing threads manually (creating, joining, synchronising), you hand individual function calls off to a pool of worker threads and receive back special objects called **FlowVars** that represent the future results of those calls.

The central idea is **data-flow parallelism**: a spawned task runs concurrently, and whoever needs its result simply waits on the FlowVar. The pool itself manages how many threads exist, growing and shrinking within configurable bounds in response to system load.

---

## Core Concepts

### The Thread Pool

At program startup the module automatically calls `setup()`, which detects the number of CPU cores and starts that many worker threads (capped at `MaxThreadPoolSize`). These threads form the pool. Tasks submitted via `spawn` are distributed among them.

The pool size is elastic: it can grow up to `maxPoolSize` when tasks are piling up, and shrink back toward `minPoolSize` when the system is idle.

### FlowVar

A `FlowVar[T]` is a typed placeholder for a value that does not exist yet — it will be produced by a worker thread some time in the future. You can think of it as a single-use channel or a promise. Once the worker finishes, it writes the result into the FlowVar; any thread waiting on that FlowVar is then unblocked.

### Distinguished Threads

A small set of threads (up to `MaxDistinguishedThread`, default 32) can be "pinned": when you use `pinnedSpawn`, the task is sent to a specific, fixed worker thread identified by a `ThreadId`. This is useful when some state is thread-local and must not migrate between threads.

---

## Exported API

---

### `spawn`

```nim
proc spawn*(call: sink typed) {.magic: "Spawn".}
```

#### Description

`spawn` is the primary way to submit work to the thread pool. It takes a single procedure call expression and arranges for that call to execute on a worker thread — never on the calling thread. The compiler rewrites the call into an internal dispatch via `nimSpawn3`.

If the procedure returns a value, `spawn` evaluates to a `FlowVar[T]` holding the eventual result. If the procedure returns `void`, no FlowVar is produced.

The procedure being spawned **must** be annotated `{.gcsafe.}` (or be inferred as such), because it will run in a separate thread that does not share GC roots with the caller.

The dispatching strategy is eager: `spawn` always places the task in the pool and never runs it inline on the calling thread (for that behaviour, see `spawnX`).

#### When all workers are busy

If no worker thread is immediately free, the implementation enters a spin loop. During this loop it may:
- Grow the pool by activating a new thread (if below `maxPoolSize`).
- If the caller is itself a worker thread, run the task inline to avoid deadlock.
- Block on a semaphore until a worker becomes available.

#### Parameters

| Parameter | Description |
|-----------|-------------|
| `call` | A procedure call expression `p(arg1, arg2, ...)`. The procedure must be `gcsafe`. |

#### Return value

`FlowVar[T]` if `p` returns `T`; nothing if `p` returns `void`.

#### Examples

```nim
import threadpool

proc heavyComputation(n: int): int {.gcsafe.} =
  result = 0
  for i in 1..n: result += i

# Spawn a task; get back a FlowVar
let fv = spawn heavyComputation(1_000_000)

# Do other work here while the task runs...

# Block until the result is ready and read it
echo ^fv   # → 500000500000
```

```nim
import threadpool

proc printMessage(msg: string) {.gcsafe.} =
  echo msg

# Void procedure — no FlowVar is returned
spawn printMessage("Hello from a worker thread!")
sync()
```

---

### `spawnX`

```nim
template spawnX*(call)
```

#### Description

`spawnX` is a smarter, adaptive version of `spawn`. It first calls `preferSpawn()` to check whether any worker thread is currently free. If one is free, it spawns the task on that thread (`spawn call`). If no thread is free, it runs the call **inline on the calling thread** instead.

This makes `spawnX` a performance optimisation for situations where the work granularity is small enough that the overhead of scheduling a task on a busy pool outweighs the benefit of parallel execution. Instead of making the producer wait, it just does the work itself.

The trade-off is that you lose the guarantee that the call will execute in parallel. If the pool is saturated, `spawnX` degrades to sequential execution. This means `spawnX` is unsuitable for cases where you need a result asynchronously or need to guarantee non-blocking submission.

#### Examples

```nim
import threadpool

proc processChunk(data: int): int {.gcsafe.} =
  result = data * data

var results: array[10, FlowVar[int]]

for i in 0..<10:
  # Spawn only if a thread is free; otherwise compute inline
  results[i] = spawnX processChunk(i)

sync()
for fv in results:
  echo ^fv
```

---

### `pinnedSpawn`

```nim
proc pinnedSpawn*(id: ThreadId; call: sink typed) {.magic: "Spawn".}
```

#### Description

Works like `spawn`, but guarantees that the call executes on the specific "distinguished" worker thread identified by `id`. Distinguished threads are a separate fixed pool (up to `MaxDistinguishedThread` = 32 by default) that exist alongside the main dynamic pool.

Because a given `id` always maps to the same thread, you can safely store thread-local state in that thread (e.g. an OpenGL context, a database connection, or any non-GC-safe resource that must not move between OS threads) and always dispatch related work to it.

The call blocks if the chosen distinguished thread is currently busy, waiting for it to become free.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `ThreadId` | An integer in the range `0 .. MaxDistinguishedThread-1`. Identifies which distinguished thread to use. |
| `call` | typed | A `gcsafe` procedure call expression. |

#### Examples

```nim
import threadpool

# Thread 0 "owns" a resource that must not be shared
proc renderFrame(frameId: int) {.gcsafe.} =
  echo "Rendering frame ", frameId, " on dedicated thread"

# Always dispatches to distinguished thread 0
pinnedSpawn 0, renderFrame(1)
pinnedSpawn 0, renderFrame(2)
sync()
```

---

### `parallel`

```nim
proc parallel*(body: untyped) {.magic: "Parallel".}
```

#### Description

`parallel` introduces a **parallel block** — a structured region of code in which a restricted subset of Nim statements can be parallelised by the compiler. Inside a `parallel` block you can use `spawn` and iterate over ranges with implicit parallelism.

The compiler analyses the body of the block, proves data independence between iterations, and generates the necessary synchronisation automatically. When the `parallel` block ends, all spawned tasks within it are guaranteed to have completed — you do not need to call `sync()` explicitly.

The body must conform to a specific DSL described in the [experimental manual](https://nim-lang.org/docs/manual_experimental.html#parallel-amp-spawn). Not all Nim constructs are valid inside `parallel`.

#### Examples

```nim
import threadpool

proc processItem(x: int): int {.gcsafe.} = x * x

var data = @[1, 2, 3, 4, 5, 6, 7, 8]
var results = newSeq[int](data.len)

parallel:
  for i in 0 ..< data.len:
    results[i] = spawn processItem(data[i])

# All tasks are done here; no sync() needed
echo results  # → @[1, 4, 9, 16, 25, 36, 49, 64]
```

---

### `sync`

```nim
proc sync*()
```

#### Description

`sync` is a simple **global barrier**: it blocks the calling thread until every worker thread in the pool has finished its current task and returned to the idle state.

Internally it polls all worker threads every `threadpoolWaitMs` milliseconds (default 100 ms, configurable via `--define:threadpoolWaitMs=N`) until all are marked ready. The polling approach (rather than blocking on a semaphore) is intentional: it avoids a race condition where a worker might shut down before signalling the semaphore.

**Use `sync` when:**
- You've spawned multiple `void` tasks and need to know they've all finished.
- You want a checkpoint after a batch of spawned work before proceeding.

**Don't use `sync` when:**
- You need the return value of a spawned task — use `^` or `awaitAndThen` instead.
- You're inside a `parallel` block — the block handles synchronisation itself.

#### Examples

```nim
import threadpool

proc writeToLog(msg: string) {.gcsafe.} =
  # Imagine this writes to a file or database
  echo "[LOG] ", msg

spawn writeToLog("Task A")
spawn writeToLog("Task B")
spawn writeToLog("Task C")

sync()  # Wait for all three log writes to complete
echo "All tasks done."
```

---

### `^` (hat / dereference)

```nim
proc `^`*[T](fv: FlowVar[T]): T
```

#### Description

The `^` operator is the standard way to **extract the result** from a `FlowVar[T]`. It blocks the calling thread until the worker has produced the value, then returns it.

For value types (integers, floats, plain objects), the value is returned from an internal field (`blob`). For heap-allocated types (`string`, `seq`, `ref`), a deep copy is performed to safely transfer ownership from the worker thread's GC heap to the caller's. In Nim's ARC/ORC memory model (`nimV2`), deep-copying is avoided and the value is moved directly.

After `^` returns, the FlowVar is marked as consumed (`finished` is called internally). Do not call `^` twice on the same FlowVar.

#### Examples

```nim
import threadpool

proc fibonacci(n: int): int {.gcsafe.} =
  if n <= 1: return n
  return fibonacci(n-1) + fibonacci(n-2)

let fv = spawn fibonacci(30)

# The calling thread is free to do other things here
echo "Waiting for fibonacci..."

let result = ^fv   # Blocks until the result is ready
echo "fibonacci(30) = ", result   # → 832040
```

```nim
import threadpool

proc buildString(n: int): string {.gcsafe.} =
  result = "Result: " & $n

let fv = spawn buildString(42)
let s = ^fv   # Deep copy ensures safe ownership transfer
echo s        # → "Result: 42"
```

---

### `awaitAndThen`

```nim
proc awaitAndThen*[T](fv: FlowVar[T]; action: proc (x: T) {.closure.})
```

#### Description

`awaitAndThen` is an alternative to `^` that avoids an extra copy for large value types. It blocks until the FlowVar's value is ready, then calls `action` with the value **passed by reference semantics** (leveraging Nim's parameter-passing rules). When `action` returns, the FlowVar is finalised.

For types like `string` or `seq`, using `awaitAndThen` can be more efficient than `^` because the value does not need to be deep-copied into the caller's frame; instead the action receives direct access to the worker's result.

**Limitation:** `awaitAndThen` cannot be used with `FlowVar[ref T]` — this is enforced as a compile-time error. For reference types, use `unsafeRead` instead.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `fv` | `FlowVar[T]` | The flow variable to await. |
| `action` | `proc (x: T) {.closure.}` | A closure called with the result once it's ready. |

#### Examples

```nim
import threadpool

proc buildReport(n: int): string {.gcsafe.} =
  result = "Report for item " & $n & ": " & $( n * n)

let fv = spawn buildReport(7)

awaitAndThen(fv) do (report: string):
  # 'report' is available here without an extra deep copy
  echo report   # → "Report for item 7: 49"
```

---

### `unsafeRead`

```nim
proc unsafeRead*[T](fv: FlowVar[ref T]): ptr T
```

#### Description

`unsafeRead` is the escape hatch for `FlowVar[ref T]` — flow variables that hold reference types. The regular `^` operator performs a deep copy for reference types, which is often undesirable. `unsafeRead` blocks until the value is ready, then returns a **raw pointer** directly into the worker thread's heap-allocated object.

This is called **unsafe** for a reason: once you have the pointer, you are bypassing the garbage collector's tracking. The object the pointer refers to must remain alive for as long as you use the pointer. In practice, this means you should consume the pointer immediately and not store it beyond the current scope.

Use `unsafeRead` only when you need zero-copy access to a large heap-allocated result and you understand the lifetime implications.

#### Examples

```nim
import threadpool

type BigData = ref object
  values: seq[int]

proc computeBigData(): BigData {.gcsafe.} =
  result = BigData(values: @[1, 2, 3, 4, 5])

let fv = spawn computeBigData()
let p = unsafeRead(fv)   # ptr BigData — no deep copy
echo p[].values           # → @[1, 2, 3, 4, 5]
# Do NOT store 'p' after this point
```

---

### `blockUntil` (for FlowVarBase)

```nim
proc blockUntil*(fv: var FlowVarBaseObj)
```

#### Description

`blockUntil` is the low-level waiting primitive that underlies `^`, `awaitAndThen`, and `unsafeRead`. It blocks the calling thread until the underlying semaphore of the FlowVar is signalled (i.e., until the worker thread has written a result).

In most application code you will not call this directly — use `^` or `awaitAndThen` instead. `blockUntil` is primarily useful when working with `FlowVarBase` (the untyped base class) directly, or when building higher-level abstractions.

Calling `blockUntil` on a FlowVar that has already been awaited is a no-op (the `awaited` flag guards against double-blocking).

---

### `blockUntilAny`

```nim
proc blockUntilAny*(flowVars: openArray[FlowVarBase]): int
```

#### Description

`blockUntilAny` takes an array of `FlowVarBase` objects and blocks until **any one of them** has a value ready. It returns the **index** of the FlowVar that became ready first.

This is the threadpool equivalent of `select` or `poll` in networking: instead of waiting for a specific result, you wait for whichever of a set of concurrent tasks finishes first.

**Important caveats:**
- If a FlowVar is already ready when `blockUntilAny` is called, its index is returned immediately without blocking.
- Each FlowVar can only participate in **one** `blockUntilAny` call at a time. If you call `blockUntilAny([a, b])` and `blockUntilAny([b, c])` concurrently, the second call treats `b` as if it were not in its list (conflict detection).
- If all FlowVars are in conflict (already being waited on by someone else), `-1` is returned.
- The documentation explicitly warns that this leads to **non-deterministic behaviour** and recommends avoiding it.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `flowVars` | `openArray[FlowVarBase]` | The set of flow variables to watch. Must be upcast from `FlowVar[T]`. |

#### Return value

The index of the first FlowVar that became ready, or `-1` if none could be waited on.

#### Examples

```nim
import threadpool

proc slowTask(id, ms: int): int {.gcsafe.} =
  sleep(ms)
  result = id

let fv0: FlowVarBase = spawn slowTask(0, 300)
let fv1: FlowVarBase = spawn slowTask(1, 100)
let fv2: FlowVarBase = spawn slowTask(2, 200)

let first = blockUntilAny([fv0, fv1, fv2])
echo "First to finish: task ", first   # → likely 1 (100ms)
```

---

### `isReady`

```nim
proc isReady*(fv: FlowVarBase): bool
```

#### Description

`isReady` is a **non-blocking** check that returns `true` if the FlowVar already has a value available, and `false` if the worker is still computing it. Calling `^` or `awaitAndThen` on a FlowVar for which `isReady` returns `true` will return immediately without blocking.

This is useful for polling patterns where you want to check multiple FlowVars in a loop and process whichever ones have finished, without committing to waiting on any one of them.

Note that `isReady` returning `false` at one moment does not mean it will still be `false` a moment later. It is a snapshot query.

#### Examples

```nim
import threadpool

proc longTask(n: int): int {.gcsafe.} =
  sleep(500)
  result = n * 2

let fv = spawn longTask(21)

while not isReady(fv):
  echo "Still computing..."
  sleep(100)

echo "Result: ", ^fv   # → 42
```

---

### `setMinPoolSize`

```nim
proc setMinPoolSize*(size: range[1..MaxThreadPoolSize])
```

#### Description

Sets the minimum number of threads the pool will maintain. The default is `4`. The pool will not shrink below this value even when threads are idle.

Increasing the minimum is useful in latency-sensitive applications where you want threads to always be ready without the overhead of creating them on demand. Decreasing it is useful in memory-constrained environments.

Must be called before spawning tasks to have a meaningful effect on the initial pool shape.

#### Examples

```nim
import threadpool

setMinPoolSize(8)   # Keep at least 8 threads alive at all times
spawn myTask()
```

---

### `setMaxPoolSize`

```nim
proc setMaxPoolSize*(size: range[1..MaxThreadPoolSize])
```

#### Description

Sets the maximum number of threads the pool is allowed to grow to. The default is `MaxThreadPoolSize` (256). The pool will never exceed this number of threads.

If you call `setMaxPoolSize` with a value smaller than the current pool size, the excess threads are flagged for shutdown. They will not be forcibly terminated — instead, each is allowed to finish its current task and then exit cleanly.

Use this to limit resource consumption in environments where many concurrent threads would be harmful (e.g., constrained memory, shared hosting, or to prevent over-subscription on a system with few cores).

#### Examples

```nim
import threadpool

# On a 4-core machine, cap the pool to avoid over-subscription
setMaxPoolSize(4)
```

---

### `preferSpawn`

```nim
proc preferSpawn*(): bool
```

#### Description

`preferSpawn` is a quick, non-blocking hint that tells you whether spawning a new task is likely to result in actual parallelism. It returns `true` if at least one worker thread is currently idle (i.e., the global readiness semaphore counter is positive), and `false` if all workers appear to be busy.

You rarely need to call this directly. It is the mechanism behind the `spawnX` template. Use it if you are implementing your own adaptive dispatch logic.

#### Examples

```nim
import threadpool

proc myWork(n: int): int {.gcsafe.} = n * n

if preferSpawn():
  let fv = spawn myWork(10)
  echo ^fv
else:
  echo myWork(10)  # Run inline; pool is busy
```

---

## Exported Constants and Types

### `MaxThreadPoolSize`

```nim
const MaxThreadPoolSize* {.intdefine.} = 256
```

The hard upper limit on how many worker threads the pool can ever contain. Can be overridden at compile time with `--define:MaxThreadPoolSize=N`.

---

### `MaxDistinguishedThread`

```nim
const MaxDistinguishedThread* {.intdefine.} = 32
```

The maximum number of "distinguished" (pinned) threads available for use with `pinnedSpawn`. Can be overridden at compile time with `--define:MaxDistinguishedThread=N`.

---

### `ThreadId`

```nim
type ThreadId* = range[0..MaxDistinguishedThread-1]
```

A range type used to identify a distinguished thread for `pinnedSpawn`. Valid values are `0` to `MaxDistinguishedThread - 1`.

---

### `FlowVarBase`

```nim
type FlowVarBase* = ref FlowVarBaseObj
```

The untyped base class for all `FlowVar[T]` instances. Used in APIs that need to work with FlowVars regardless of their result type, such as `blockUntilAny` and `isReady`.

---

### `FlowVar[T]`

```nim
type FlowVar*[T] = ref FlowVarObj[T]
```

A typed data-flow variable. Returned by `spawn` when the spawned procedure has a non-void return type. Provides a typed interface to retrieve the result via `^` or `awaitAndThen`. Inherits from `FlowVarBase`.

---

## Compile-time Configuration

| Flag | Default | Effect |
|------|---------|--------|
| `--threads:on` | *(required)* | Must be set. Without it, the module refuses to compile. |
| `--define:MaxThreadPoolSize=N` | `256` | Maximum total threads in the pool. |
| `--define:MaxDistinguishedThread=N` | `32` | Maximum pinned/distinguished threads. |
| `--define:threadpoolWaitMs=N` | `100` | Polling interval (ms) used by `sync` and shutdown logic. |
| `--define:nimRecursiveSpawn` | *(off)* | Enables recursive spawn: a worker thread that is waiting can execute the pending task itself, preventing certain deadlocks in recursive parallel algorithms. |
| `--define:nimPinToCpu` | *(off)* | Enables CPU affinity pinning: each worker thread is pinned to a specific CPU core. |

---

## Common Patterns and Pitfalls

**Pattern: fan-out / fan-in**
Spawn many tasks, collect all FlowVars into a sequence, then read them all.

```nim
import threadpool

proc square(n: int): int {.gcsafe.} = n * n

var fvs = newSeq[FlowVar[int]](10)
for i in 0..<10:
  fvs[i] = spawn square(i)

var results = newSeq[int](10)
for i in 0..<10:
  results[i] = ^fvs[i]

echo results   # → @[0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

**Pitfall: reading a FlowVar twice**
Calling `^` a second time on the same FlowVar is undefined behaviour. The internal state is consumed on the first read.

**Pitfall: GC safety**
All procedures passed to `spawn` or `pinnedSpawn` must be `{.gcsafe.}`. Sharing mutable GC-managed data between threads without explicit locking will cause memory corruption or crashes.

**Pitfall: `sync` is a blunt instrument**
`sync()` waits for every thread in the pool to become idle. If another part of your program has also spawned work, `sync` will wait for that too. Prefer reading results via FlowVars when you need fine-grained synchronisation.
