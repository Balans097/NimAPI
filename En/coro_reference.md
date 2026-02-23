# `coro` Module Reference

**Module:** `coro` (part of Nim's standard library)  
**Status:** ⚠️ Unstable API — interfaces may change between Nim versions.  
**Purpose:** Cooperative multitasking through stackful coroutines — multiple execution contexts that explicitly yield control to one another within a single OS thread.

---

## Overview

A **coroutine** is a function that can pause its own execution at an arbitrary point (`suspend`) and later be resumed exactly where it left off, with its entire call stack intact. Unlike OS threads, coroutines are *cooperative*: the scheduler only switches context when the currently running coroutine explicitly suspends. There is no preemption.

This model is well-suited for:
- State machines written as straight-line code instead of explicit state variables.
- I/O pipelines where one stage waits while another produces data.
- Game logic loops where each entity "sleeps" between ticks.
- Any scenario where you want the clarity of sequential code but need to interleave multiple independent tasks.

### How execution flows

```
main thread
│
├── start(taskA)   ← schedules taskA, does not run it yet
├── start(taskB)   ← schedules taskB, does not run it yet
└── run()          ← enters the scheduler loop
      │
      ├── switches to taskA → taskA runs until suspend()
      │          taskA suspends ────────────────────────┐
      ├── switches to taskB → taskB runs until suspend() │
      │          taskB suspends ─────────────────────────┤
      ├── back to taskA (if sleep expired) ←─────────────┘
      │   ...
      └── exits when all coroutines have returned
```

### Compiler flag required

```sh
nim c -d:nimCoroutines myapp.nim
```

Without `-d:nimCoroutines` the module refuses to compile.

### Context-switching backends

| Backend | Platform | Flag |
|---|---|---|
| `ucontext` | Linux, macOS, most POSIX *(default)* | `-d:nimCoroutinesUcontext` |
| `setjmp` | x86/x86-64 POSIX, Haiku, OpenBSD | `-d:nimCoroutinesSetjmp` |
| `fibers` | Windows *(only option)* | automatic |

The backend is selected automatically from the platform. You only need flags to override the default on POSIX.

---

## Exported API

| Symbol | Kind | Description |
|---|---|---|
| `CoroutineRef` | ref object | Safe handle to a running coroutine |
| `start(c, stacksize)` | proc | Schedule a coroutine for execution |
| `run()` | proc | Start the scheduler; blocks until all coroutines finish |
| `suspend(sleepTime)` | proc | Yield execution; optionally sleep for `sleepTime` seconds |
| `alive(c)` | proc | Check whether a coroutine is still running |
| `wait(c, interval)` | proc | Block current coroutine until another one finishes |

---

## `CoroutineRef`

```nim
type CoroutineRef* = ref object
  coro: CoroutinePtr   # private; nil when coroutine has finished
```

### What it is

`CoroutineRef` is the only handle the public API gives you to a coroutine. It is a `ref object`, so it participates in Nim's garbage collection: you can store it in a variable, pass it around, and the GC will keep it alive as long as you hold a reference.

The internal `coro` field points to the actual low-level coroutine structure. When the coroutine finishes and the scheduler cleans it up, `coro` is set to `nil`. All public procedures (`alive`, `wait`) check for `nil` first and fail gracefully instead of crashing.

This design decouples the *logical handle* (which you hold) from the *physical memory* (which the scheduler manages). You never need to free a `CoroutineRef` manually.

### Example

```nim
import coro

let handle: CoroutineRef = start(proc() =
  echo "I am a coroutine"
)

run()
# After run() returns, handle.coro is nil — the coroutine is gone.
echo alive(handle)   # false
```

---

## `start`

```nim
proc start*(c: proc(), stacksize: int = defaultStackSize): CoroutineRef {.discardable.}
```

### What it does

Registers a zero-argument, zero-result procedure `c` as a new coroutine and adds it to the scheduler's queue. The coroutine does **not** begin running immediately — it only runs when the scheduler loop (`run`) transfers control to it.

**`c`** — the body of the coroutine. It is a plain `proc()` closure, so it may capture variables from its enclosing scope.

**`stacksize`** — the size of the coroutine's private call stack in bytes. The default is 512 KB (`512 * 1024`). Each coroutine gets its own stack; choose a size large enough for the deepest call chain you expect inside the coroutine. Stack overflow from too-small a value is undefined behaviour.

**Return value** — a `CoroutineRef` that can be passed to `alive` or `wait`. The return value is marked `discardable`, so you may ignore it if you only care that the coroutine runs and not about tracking its lifetime.

`start` calls `initialize()` on first use if the scheduler context has not been set up yet, so you do not need to initialise anything manually before calling `start`.

### When to use

Call `start` once per concurrent task, before calling `run`. You can also call `start` from *inside* a running coroutine to dynamically spawn additional tasks; the new coroutine will be picked up by the scheduler on the next cycle.

### Examples

```nim
import coro

# Example 1 — basic scheduling, handle discarded
start(proc() = echo "task one")
start(proc() = echo "task two")
run()

# Example 2 — capturing variables from the outer scope
var results: seq[int]
for i in 1..3:
  let id = i   # capture by value
  start(proc() =
    suspend(float(id) * 0.001)  # stagger start times
    results.add(id)
  )
run()
echo results  # order depends on timing, but contains 1, 2, 3

# Example 3 — keeping the handle to check completion later
let worker = start(proc() =
  for i in 1..5:
    echo "working: ", i
    suspend(0.1)
)
run()
echo "worker finished: ", not alive(worker)

# Example 4 — custom stack size for a deeply recursive coroutine
start(proc() =
  # deep recursion needs more stack space
  discard heavilyRecursiveOperation()
, stacksize = 2 * 1024 * 1024)  # 2 MB
run()
```

---

## `run`

```nim
proc run*()
```

### What it does

Enters the scheduler loop and drives all registered coroutines to completion. `run` **blocks the calling thread** until every coroutine created with `start` has returned (or been cleaned up).

#### The scheduling algorithm

The scheduler maintains a doubly-linked list of coroutines. On each iteration it:

1. Looks at the current coroutine's remaining sleep time (`sleepTime - elapsed`).
2. If the sleep has expired (or there was no sleep), switches execution into that coroutine.
3. When the coroutine calls `suspend`, control returns here.
4. If the coroutine has finished (`state == CORO_FINISHED`), it is removed from the list and its memory is freed.
5. Moves to the next coroutine in the list (round-robin).
6. If all coroutines are sleeping, calls `os.sleep` for the shortest remaining delay before checking again.

The algorithm is **non-preemptive**: a coroutine runs uninterrupted until it calls `suspend` or returns.

#### What happens to memory

When a coroutine finishes, the scheduler:
- Sets `CoroutineRef.coro` to `nil` so existing handles know the coroutine is gone.
- Removes the stack from the GC's tracked stack list (non-ARC/ORC builds).
- Frees the fiber object (Windows) or the allocated stack memory (POSIX).
- Frees the coroutine structure itself.

`run` returns once the coroutine list is empty.

### When to use

Call `run` exactly once per program (or once per logical coroutine "session"). All `start` calls for the initial batch of tasks should happen before `run`. Tasks that dynamically spawn children can call `start` from within their own body — the children will be processed in subsequent scheduler cycles.

### Example

```nim
import coro

proc producer() =
  for i in 1..5:
    echo "produce ", i
    suspend()

proc consumer() =
  for i in 1..5:
    echo "consume ", i
    suspend()

start(producer)
start(consumer)
run()
# Prints interleaved produce/consume lines, then returns.
```

---

## `suspend`

```nim
proc suspend*(sleepTime: float = 0)
```

### What it does

Pauses the currently running coroutine and hands control back to the scheduler. The coroutine will not be resumed until at least `sleepTime` seconds have elapsed. Passing `0` (the default) yields immediately and allows the scheduler to run the next coroutine, but the current coroutine is eligible to run again on the very next scheduler cycle.

Internally, `suspend` saves the current execution context and calls `switchTo` targeting the scheduler's own "loop" coroutine. The scheduler loop then picks the next coroutine to run.

`suspend` may only be called from *inside* a coroutine (i.e. from code running under `run`). Calling it from the main thread outside `run` is undefined behaviour.

### The `sleepTime` parameter in depth

`sleepTime` is a `float` measured in **seconds**. The scheduler checks elapsed time using a high-resolution tick counter. The actual sleep is not exact — the coroutine becomes *eligible* to run after `sleepTime` seconds, but will only actually run when the scheduler reaches it in the round-robin cycle. Short `sleepTime` values (e.g. `0.001`) create high-frequency coroutines; large values (e.g. `5.0`) create low-frequency ones.

Passing a negative value has the same effect as passing `0`.

### When to use

- Call `suspend()` (no argument) at every natural "yield point" in cooperative tasks — between iterations of a processing loop, after each unit of work, etc.
- Call `suspend(seconds)` when a coroutine must wait for a real-time event: polling a sensor every 100 ms, animating at 60 fps, retrying a failed operation after a back-off delay.
- Call `suspend()` inside a wait loop when you want to check a condition repeatedly without busy-waiting.

### Examples

```nim
import coro

# Example 1 — yield on every iteration (maximum interleaving)
proc taskA() =
  for i in 1..5:
    echo "A: ", i
    suspend()   # give taskB a chance to run

proc taskB() =
  for i in 1..5:
    echo "B: ", i
    suspend()

start(taskA)
start(taskB)
run()
# Output interleaves: A:1, B:1, A:2, B:2, ...

# Example 2 — timed sleep (coroutine wakes up after ~0.5 s)
proc timedTask() =
  echo "starting"
  suspend(0.5)   # sleep 500 ms
  echo "resumed after 0.5 s"

start(timedTask)
run()

# Example 3 — polling loop with back-off
proc poller(ready: ptr bool) =
  while not ready[]:
    suspend(0.05)   # check every 50 ms
  echo "condition met!"
```

---

## `alive`

```nim
proc alive*(c: CoroutineRef): bool = c.coro != nil and c.coro.state != CORO_FINISHED
```

### What it does

Returns `true` if the coroutine represented by `c` is still scheduled and has not yet returned; `false` if it has finished or if `c.coro` is `nil` (meaning the scheduler has already cleaned up the coroutine's memory).

The check is a simple two-condition expression:
- `c.coro != nil` — the scheduler has not freed the structure yet.
- `c.coro.state != CORO_FINISHED` — the coroutine's function has not returned.

`alive` does **not** resume the coroutine or interact with the scheduler in any way. It is a pure, side-effect-free read.

### When to use

- As a guard in a wait loop you write yourself (when you need custom inter-check logic beyond what `wait` provides).
- To check after `run()` returns whether a specific coroutine completed normally.
- To implement a "fire and forget" pattern where you only occasionally poll whether background work is still going.

### Examples

```nim
import coro

let h = start(proc() =
  suspend(0.2)
  echo "done"
)

# From inside another coroutine, poll manually:
start(proc() =
  while alive(h):
    echo "still running..."
    suspend(0.05)
  echo "it finished"
)

run()

# After run() returns:
echo alive(h)   # false — scheduler has cleaned up
```

---

## `wait`

```nim
proc wait*(c: CoroutineRef, interval = 0.01)
```

### What it does

Suspends the *calling* coroutine and repeatedly checks whether coroutine `c` has finished, polling every `interval` seconds. Returns only after `c` is no longer alive.

Internally it is a thin loop:

```nim
while alive(c):
  suspend(interval)
```

`wait` must be called from *inside* a coroutine (it calls `suspend` internally). Calling it from the main thread outside a `run` context will hang.

**`interval`** — the polling frequency in seconds. The default `0.01` (10 ms) is a sensible balance between responsiveness and CPU usage. Use a smaller value if you need low-latency join semantics; use a larger value if the waited-for task is expected to run for many seconds.

### When to use

Use `wait` whenever one coroutine depends on the result or side-effects of another and must not proceed until the other has finished. It is the coroutine equivalent of joining a thread.

Do **not** call `wait(c)` on your own `CoroutineRef` — that would create an infinite loop.

### Examples

```nim
import coro

# Example 1 — sequential dependency between two coroutines
proc setup() =
  echo "setup running..."
  suspend(0.1)
  echo "setup done"

proc mainTask(setupRef: CoroutineRef) =
  wait(setupRef)         # block until setup finishes
  echo "main running after setup"

let s = start(setup)
start(proc() = mainTask(s))
run()
# Guaranteed order: "setup done" before "main running after setup"

# Example 2 — fan-out / fan-in pattern
proc worker(id: int) =
  suspend(float(id) * 0.05)
  echo "worker ", id, " done"

proc coordinator() =
  var handles: seq[CoroutineRef]
  for i in 1..4:
    handles.add(start(proc() = worker(i)))
  for h in handles:
    wait(h)            # wait for each worker in order
  echo "all workers finished"

start(coordinator)
run()
```

---

## Full worked example

```nim
# compile: nim c -d:nimCoroutines -r pipeline.nim

import coro

# A simple producer → transformer → consumer pipeline.
# Each stage runs as a coroutine and communicates through a shared seq.

var buffer: seq[int]
var transformBuffer: seq[int]
var done = false

proc producer() =
  for i in 1..10:
    buffer.add(i)
    echo "produced: ", i
    suspend(0.01)   # yield after each item
  done = true

proc transformer() =
  var consumed = 0
  while not done or consumed < buffer.len:
    if consumed < buffer.len:
      let val = buffer[consumed] * buffer[consumed]  # square it
      transformBuffer.add(val)
      inc consumed
    suspend(0.005)

proc consumer() =
  var printed = 0
  while not done or printed < transformBuffer.len:
    if printed < transformBuffer.len:
      echo "consumed: ", transformBuffer[printed]
      inc printed
    suspend(0.005)

let p = start(producer)
let t = start(transformer)
let c = start(consumer)

# A watchdog coroutine that waits for all stages and reports
start(proc() =
  wait(p)
  wait(t)
  wait(c)
  echo "Pipeline complete."
)

run()
```

---

## Caveats and limitations

**Single-threaded.** All coroutines created within one `run` call share a single OS thread. You cannot run coroutines across threads — `ctx` is a thread-local variable, so each thread has its own independent scheduler and coroutine list.

**Cooperative scheduling only.** A coroutine that never calls `suspend` will block all other coroutines indefinitely, starving them. Always call `suspend` in any long-running or looping coroutine.

**Stack size is fixed at creation time.** You cannot grow a coroutine's stack after `start`. If a coroutine overflows its stack, behaviour is undefined (likely a crash). For call-heavy coroutines, pass a larger `stacksize` to `start`.

**Exceptions in coroutines.** Unhandled exceptions inside a coroutine are caught by `runCurrentTask`, which prints a message and a stack trace, then marks the coroutine as finished. The exception does **not** propagate to the caller or to other coroutines.

**`wait` must be called inside `run`.** Calling `wait` (or `suspend`) outside a `run` context is undefined behaviour because there is no scheduler loop to return control to.

**setjmp backend platform restriction.** The `setjmp` backend only supports x86 and x86-64. On other architectures, use `ucontext` (POSIX) or `fibers` (Windows).

**Unstable API.** This module is explicitly marked as having an unstable API. Pin your Nim version if you depend on it.
