# Reference: `threadpool` Module (Nim)

> The `threadpool` module implements the `parallel` and `spawn` statements for parallel task execution via a managed thread pool.

> ⚠️ **Deprecated.** Use one of the Nimble packages instead:  
> **[malebolgia](https://github.com/araq/malebolgia)**, **[taskpools](https://github.com/status-im/nim-taskpools)**, or **[weave](https://github.com/mratsim/weave)**.

> ⚠️ **Unstable API** — signatures may change in future Nim versions.

> ℹ️ Requires `--threads:on` to compile.

---

## Table of Contents

- [Constants](#constants)
- [Data Types](#data-types)
- [Spawning Tasks](#spawning-tasks)
- [FlowVar — Awaiting Results](#flowvar--awaiting-results)
- [Thread Pool Management](#thread-pool-management)
- [Synchronisation](#synchronisation)
- [How It Works](#how-it-works)
- [Quick Reference](#quick-reference)

---

## Constants

### `MaxThreadPoolSize`

Maximum number of threads in the pool. Default: `256`. Can be overridden at compile time with `-d:MaxThreadPoolSize=N`.

```nim
echo MaxThreadPoolSize   # 256
```

---

### `MaxDistinguishedThread`

Maximum number of "distinguished" threads available to `pinnedSpawn`. Default: `32`. Override with `-d:MaxDistinguishedThread=N`.

```nim
echo MaxDistinguishedThread   # 32
```

---

## Data Types

### `FlowVarBase`

An untyped base class for `FlowVar[T]`. Used by functions that work with any flow variable regardless of result type (e.g. `blockUntilAny`, `isReady`).

---

### `FlowVar[T]`

A **data flow variable** — a reference object representing the pending result of an asynchronously executing task. Returned by `spawn` when the spawned procedure has a non-`void` return type.

The value is not immediately available; it becomes accessible once the background thread finishes. Use the `^` operator or `awaitAndThen` to retrieve it.

```nim
proc double(x: int): int = x * 2

# spawn returns FlowVar[int]:
let fv: FlowVar[int] = spawn double(21)

# block until ready and read the result:
let result = ^fv
assert result == 42
```

---

### `ThreadId`

A thread identifier for `pinnedSpawn`. Range: `0 .. MaxDistinguishedThread - 1`.

```nim
let tid: ThreadId = 0
pinnedSpawn tid, myProc(args)
```

---

## Spawning Tasks

### `spawn(call: sink typed)`

Always dispatches `call` to a worker thread from the pool. The calling thread **never** executes the task itself.

`call` must be a procedure call `p(...)` where `p` is marked `gcsafe`. The return type of `p` must be either `void` or compatible with `FlowVar[T]`.

```nim
proc compute(n: int): int {.gcsafe.} =
  result = n * n

# Void task:
proc printHello() {.gcsafe.} =
  echo "Hello from thread!"

spawn printHello()

# Task with result:
let fv = spawn compute(10)
echo ^fv   # 100

# Multiple tasks:
var results: array[4, FlowVar[int]]
for i in 0..3:
  results[i] = spawn compute(i)
for i in 0..3:
  echo ^results[i]
```

---

### `pinnedSpawn(id: ThreadId; call: sink typed)`

Dispatches the task to **exactly** the thread identified by `id`. Guarantees that the task runs on the specified "distinguished" thread.

Useful when a task must interact with thread-local resources (e.g. an OpenGL context bound to a specific thread).

```nim
proc renderFrame(data: RenderData) {.gcsafe.} =
  # Must run on the thread that owns the GPU context
  discard

let gpuThread: ThreadId = 0
pinnedSpawn gpuThread, renderFrame(myData)
```

---

### `spawnX(call)` — template

Dispatches the task to a new thread **if** a free CPU core is available (i.e. `preferSpawn()` returns `true`). Otherwise executes the call **in the calling thread**.

Caution: this can introduce unpredictable latency in the calling thread. Prefer explicit `spawn` in most cases.

```nim
spawnX myProc(arg)
# Equivalent to:
if preferSpawn(): spawn myProc(arg) else: myProc(arg)
```

---

### `parallel(body: untyped)`

A parallel section operator. The `body` must be written in a special DSL — a restricted subset of Nim that the compiler can analyse for data races.

See the [Nim experimental manual — parallel & spawn](https://nim-lang.org/docs/manual_experimental.html#parallel-amp-spawn) for the full DSL specification.

```nim
parallel:
  for i in 0 ..< 4:
    spawn myProc(i)
```

---

## FlowVar — Awaiting Results

### `^[T](fv: FlowVar[T]): T`

The FlowVar dereference operator. **Blocks** the calling thread until the result is available, then returns it.

```nim
proc fib(n: int): int {.gcsafe.} =
  if n <= 1: n else: fib(n-1) + fib(n-2)

let f = spawn fib(30)
# ... do other work here ...
let result = ^f   # block here if not yet ready
echo result       # 832040
```

---

### `awaitAndThen[T](fv: FlowVar[T]; action: proc(x: T) {.closure.})`

Blocks until `fv` is ready, then passes the result to `action`. Can be more efficient than `^` because Nim's parameter passing semantics may allow avoiding a copy of `T`.

```nim
let fv = spawn computeData(42)
fv.awaitAndThen do (x: int):
  echo "Result: ", x
  process(x)
```

---

### `unsafeRead*[T](fv: FlowVar[ref T]): ptr T`

Blocks until `fv` is ready and returns a **raw pointer** to the `ref T` value. Unsafe — the caller is responsible for the lifetime of the pointed-to object.

```nim
type Data = ref object
  value: int

proc makeData(n: int): Data {.gcsafe.} =
  Data(value: n * 2)

let fv = spawn makeData(10)
let p = unsafeRead(fv)
echo p.value   # 20
```

---

### `blockUntil(fv: var FlowVarBaseObj)`

Blocks the calling thread until the value of `fv` becomes available. Normally you do not need to call this directly — `^` and `awaitAndThen` call it automatically.

```nim
var fvBase: FlowVarBase = spawn myProc()
blockUntil(fvBase[])
```

---

### `blockUntilAny(flowVars: openArray[FlowVarBase]): int`

Waits for **any one** of the given `flowVars` to become ready. Returns the **index** of the first FlowVar whose value arrived. Returns `-1` if no FlowVar could be waited on.

> ⚠️ Each FlowVar supports only **one** concurrent call to `blockUntilAny`. Results in non-deterministic behaviour — use with care.

```nim
let f1 = spawn slowTask(1)
let f2 = spawn slowTask(2)
let f3 = spawn slowTask(3)

let idx = blockUntilAny([FlowVarBase(f1), FlowVarBase(f2), FlowVarBase(f3)])
echo "First to finish: index ", idx
```

---

### `isReady(fv: FlowVarBase): bool`

Returns `true` if the result is already available **without blocking**. If `true`, `^fv` will return immediately.

```nim
let fv = spawn longComputation()

# Non-blocking poll:
while not isReady(fv):
  echo "Not ready yet, doing other work..."
  doOtherWork()

echo "Result: ", ^fv
```

---

## Thread Pool Management

### `setMinPoolSize(size: range[1..MaxThreadPoolSize])`

Sets the **minimum** number of threads kept alive in the pool. Default: `4`. The pool will never shrink below this value.

```nim
setMinPoolSize(2)
```

---

### `setMaxPoolSize(size: range[1..MaxThreadPoolSize])`

Sets the **maximum** number of threads the pool may grow to. Default: `MaxThreadPoolSize` (256). If the current pool exceeds the new maximum, excess worker threads are marked for shutdown.

```nim
setMaxPoolSize(8)   # cap pool at 8 threads
```

---

### `preferSpawn(): bool`

Quickly checks whether a `spawn` is likely worthwhile at this moment (i.e. whether a worker thread is currently idle). Returns `true` if spawning may help. Used internally by the `spawnX` template.

```nim
if preferSpawn():
  spawn myTask()
else:
  myTask()   # run synchronously
```

---

## Synchronisation

### `sync()`

A simple barrier: blocks the calling thread until **all** tasks dispatched via `spawn` have finished. Checks that every worker thread in the pool is idle.

> For finer-grained synchronisation, use explicit barriers from the `locks` module.

```nim
for i in 0 ..< 10:
  spawn worker(i)

sync()   # wait for all 10 tasks
echo "All tasks finished"
```

---

## How It Works

### Task lifecycle

```
Calling thread              Thread pool
      │                          │
      │  spawn myProc(args)      │
      ├─────────────────────────►│  A worker picks up the task
      │                          │  and executes myProc(args)
      │  let fv = spawn ...      │
      │                          │  result is written into FlowVar
      │  let r = ^fv   ──────────┼──► (blocks if not yet ready)
      │◄─────────────────────────┤  signal: value is ready
      │  r == result             │
```

### Automatic pool scaling

The thread pool adapts its size dynamically:
- Starts with one thread per logical CPU core.
- Automatically adds threads when all workers are busy (up to `maxPoolSize`).
- Shuts down excess threads when load is low (down to `minPoolSize`).

### Requirements for spawned procedures

A procedure passed to `spawn` must:
- Be marked `{.gcsafe.}` (must not capture globally mutable state without synchronisation).
- Have return type `void` or one compatible with `FlowVar[T]`.

```nim
# Correct:
proc safeWorker(x: int): string {.gcsafe.} =
  $x & " processed"

# Wrong — GC-unsafe proc causes a compiler error:
var globalData: seq[int]
proc unsafeWorker(x: int) =   # not gcsafe
  globalData.add(x)
```

---

## Quick Reference

### Operations table

| Operation | Description |
|---|---|
| `spawn myProc(args)` | Dispatch task to pool; always runs on a background thread |
| `pinnedSpawn id, myProc(args)` | Dispatch to a specific distinguished thread |
| `spawnX myProc(args)` | Dispatch if a free thread exists, otherwise run synchronously |
| `parallel: ...` | Parallel block (DSL) |
| `^fv` | Block and retrieve result from FlowVar |
| `fv.awaitAndThen(action)` | Block and pass result to a closure |
| `unsafeRead(fv)` | Get a raw pointer to a `FlowVar[ref T]` result |
| `blockUntil(fv)` | Explicitly wait for a FlowVarBase to become ready |
| `blockUntilAny(fvs)` | Wait for the first ready FlowVar in an array |
| `isReady(fv)` | Non-blocking check whether a FlowVar is ready |
| `sync()` | Barrier: wait for all spawned tasks to finish |
| `setMinPoolSize(n)` | Set minimum pool size (default: 4) |
| `setMaxPoolSize(n)` | Set maximum pool size (default: 256) |
| `preferSpawn()` | Check if a free worker is available |

### Typical pattern: parallel map

```nim
import std/threadpool

proc processItem(x: int): int {.gcsafe.} =
  x * x + 1

const N = 8
var futures: array[N, FlowVar[int]]

# Dispatch all tasks:
for i in 0 ..< N:
  futures[i] = spawn processItem(i)

# Collect results:
var results: array[N, int]
for i in 0 ..< N:
  results[i] = ^futures[i]

echo results   # [1, 2, 5, 10, 17, 26, 37, 50]
```

### Notes

- The module is **deprecated**; for new projects use `malebolgia`, `taskpools`, or `weave`.
- Requires the `--threads:on` compiler flag.
- The initial pool size equals the number of logical CPU cores.
- `FlowVar` has a `=destroy` destructor that automatically cleans up associated resources when it goes out of scope.
- `blockUntilAny` produces **non-deterministic** behaviour and should be used with care.
- The pool's polling interval is controlled by the compile-time define `threadpoolWaitMs` (default: 100 ms).
