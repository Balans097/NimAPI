# Cpuload Module — Reference

> ⚠️ **Unstable API.** The module is part of Nim's standard library but is explicitly marked as having an unstable interface. Signatures and behaviour may change between Nim versions.

---

## What This Module Does

`cpuload` answers a single question that a thread pool needs to ask periodically: given the current system load, should the pool **grow** (add a thread), **shrink** (retire a thread), or **stay the same**?

This is not a general-purpose CPU monitoring library. It is a narrow, purposeful advisor designed to be called from inside the thread pool scheduling loop — specifically from `threadpool`'s `nimSpawn3`. Its entire design is oriented around one decision point: is spawning more work likely to help right now, or is the system already saturated?

The module is intentionally minimal. It exposes two types and one procedure. Everything else is platform-specific internals.

> **Current platform coverage:** The adaptive logic is fully implemented only on **Windows** and **Linux**. On all other platforms, `advice` always returns `doNothing`, meaning the pool neither grows nor shrinks based on load — only the minimum pool size keeps threads alive.

---

## Core Design: Stateful, Incremental Sampling

The key design constraint is that `advice` is called **inside a hot scheduling loop** — a loop that runs every time a new task needs to be dispatched and no worker is immediately free. This loop must be fast.

To avoid expensive OS calls on every iteration, `advice` uses a **call counter** (`state.calls`) to skip the heavy measurement most of the time. The check only actually runs when `calls` is below `maxPoolSize` (the "warm-up" phase) or when `(calls and 127) == 0` (a periodic check roughly every 128 calls). This is controlled by the caller (`threadpool`'s `nimSpawn3`), not by `cpuload` itself.

On Windows, the measurement is incremental: `ThreadPoolState` stores the CPU time readings from the previous call, and `advice` computes the **delta** since last time. This means the first call after state initialisation is always a no-op for the decision (it just records the baseline), and the second call produces the first real measurement.

---

## Exported Types

### `ThreadPoolAdvice`

```nim
type ThreadPoolAdvice* = enum
  doNothing,
  doCreateThread,
  doShutdownThread
```

#### Description

An enumeration representing the three possible recommendations that `advice` can return. It is the output type of the entire module — everything else exists to produce one of these three values.

| Value | Meaning |
|-------|---------|
| `doNothing` | The current pool size is appropriate. Neither grow nor shrink. |
| `doCreateThread` | System load is low enough that adding a worker thread would likely improve throughput. |
| `doShutdownThread` | Too many threads are active relative to available work; retiring one would reduce contention and overhead. |

The consumer of this advice (`threadpool`) is not obligated to follow it exactly. It checks additional conditions (e.g. whether `currentPoolSize < maxPoolSize`) before acting on `doCreateThread`, and similarly for `doShutdownThread`. The advice is a hint, not a command.

---

### `ThreadPoolState`

```nim
type ThreadPoolState* = object
  when defined(windows):
    prevSysKernel, prevSysUser, prevProcKernel, prevProcUser: FILETIME
  calls*: int
```

#### Description

A small state object that `advice` uses to carry information between calls. It must be created by the caller, initialised to its zero value (which Nim's `default(ThreadPoolState)` or a `var` declaration at module level both provide), and passed by `var` reference to every call to `advice`.

The object has two layers:

**`calls`** (exported, all platforms): a simple counter that increments by one on every call to `advice`. The caller (or the module itself) uses this count to decide how frequently the expensive OS measurement should actually run.

**`prevSys*` / `prevProc*`** (Windows only, not exported): four `FILETIME` values that record the kernel-time and user-time of the system and of this process as of the last measurement. Used to compute CPU time deltas between consecutive calls.

On non-Windows platforms, the `when defined(windows)` block is absent, so `ThreadPoolState` contains only `calls`. The object remains valid and must still be passed to `advice`; it just carries no timing data.

**Lifetime:** a single `ThreadPoolState` instance should live for the entire lifetime of the thread pool. Reinitialising it (e.g. assigning a fresh `default(ThreadPoolState)`) resets the call counter and discards the Windows timing baseline, which causes the next Windows measurement to be skipped (the first call after init is always baseline-only).

---

## Exported Procedure

### `advice`

```nim
proc advice*(s: var ThreadPoolState): ThreadPoolAdvice
```

#### Description

Inspects the current system CPU load and returns a `ThreadPoolAdvice` value indicating whether the thread pool should grow, shrink, or stay the same. The state object `s` is updated on every call (at minimum, `s.calls` is incremented).

#### How it works per platform

**Windows:**

Uses Win32's `GetSystemTimes` and `GetProcessTimes` to obtain CPU time in 100-nanosecond intervals:
- `GetSystemTimes` returns three `FILETIME` values: idle time, kernel time (includes idle), and user time.
- `GetProcessTimes` returns four `FILETIME` values for the current process: creation time, exit time, kernel time, and user time.

On the first call (`s.calls == 0`), the function records the current values as the baseline and returns `doNothing`.

On subsequent calls it computes:
- `sysTotal` = (sysKernel − prevSysKernel) + (sysUser − prevSysUser) — total CPU time the system spent (across all cores) since the last measurement.
- `procTotal` = (procKernel − prevProcKernel) + (procUser − prevProcUser) — CPU time consumed by **this process** since the last measurement.

If `procTotal / sysTotal < 0.85` (this process used less than 85% of total CPU time), it returns `doCreateThread`. The 85% threshold exists because practical measurements show that true 100% utilisation is rarely reported even when all cores are busy.

If either time snapshot fails (the Win32 calls return 0), the function returns `doNothing` safely.

**Linux:**

Reads `/proc/loadavg`, which the Linux kernel updates approximately every 5 seconds. The file has the format:

```
<1-min> <5-min> <15-min> <running>/<total> <last-pid>
```

The function reads the `<running>/<total>` fields: `busy` is the number of processes currently runnable (on CPU or in the run queue), and `total` is the total number of processes/threads in the system.

The decision uses `busy - 1` (subtracting 1 to account for the current process itself) compared against the number of logical processors (`countProcessors()`):

| Condition | Returned advice |
|-----------|----------------|
| `busy - 1 < cpus` | `doCreateThread` — runnable tasks fewer than cores; cores are underutilised |
| `busy - 1 >= cpus * 2` | `doShutdownThread` — runnable tasks at least double the cores; system is overloaded |
| otherwise | `doNothing` — load is in the acceptable range |

If `/proc/loadavg` cannot be opened, the function returns `doNothing` safely.

**All other platforms:**

Always returns `doNothing`. The pool neither grows nor shrinks based on load. Only the minimum pool size configured via `setMinPoolSize` keeps threads alive.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `s` | `var ThreadPoolState` | The state object. Must persist between calls. Always mutated: at minimum `s.calls` is incremented. |

#### Return value

A `ThreadPoolAdvice` value. See the type documentation above for the meaning of each value.

#### Examples

```nim
import cpuload

var state: ThreadPoolState
# state is zero-initialised; state.calls == 0

# First call on Windows: records baseline, returns doNothing
# First call on Linux: reads /proc/loadavg immediately
echo advice(state)   # → doNothing (Windows baseline) or a real value (Linux)
echo state.calls     # → 1

# Subsequent calls produce real recommendations
for i in 1..10:
  let rec = advice(state)
  case rec
  of doNothing:       echo "Pool size OK"
  of doCreateThread:  echo "Consider adding a worker"
  of doShutdownThread: echo "Consider retiring a worker"
```

```nim
# Typical usage pattern mirroring what threadpool does internally:
import cpuload, os

var state: ThreadPoolState

proc schedulingLoop() =
  while true:
    # ... try to dispatch task to existing workers ...

    # Only check load periodically (every ~128 calls or during warmup)
    if state.calls < maxPoolSize or (state.calls and 127) == 0:
      case advice(state)
      of doCreateThread:
        if currentPoolSize < maxPoolSize:
          activateNewWorker()
      of doShutdownThread:
        if currentPoolSize > minPoolSize:
          signalWorkerToStop()
      of doNothing:
        discard

    os.sleep(10)
```

#### Caveats

**The Linux `/proc/loadavg` delay.** The kernel updates `/proc/loadavg` roughly every 5 seconds. Between updates, repeated calls to `advice` on Linux read a stale value. This means Linux load decisions have an inherent 5-second lag. For a thread pool, this is generally acceptable — thread creation and destruction are themselves slow operations.

**The Windows first-call skip.** On Windows, the very first call after initialising a `ThreadPoolState` always returns `doNothing` because there is no delta to compute yet. This is by design: the baseline must be established before a meaningful comparison can be made.

**Not a standalone monitoring tool.** `advice` is designed to be called from a specific scheduling context inside `threadpool`. Using it in isolation (e.g. to build a CPU dashboard) will work technically, but the design assumptions (short intervals between calls, the pool-size constraints checked by the caller) will not hold, and the advice may be misleading.

**Platform gap.** macOS, BSD, and other platforms always receive `doNothing`. On these systems, the pool size is governed entirely by `minPoolSize` and `maxPoolSize` and by whether workers happen to be available.

---

## Relationship to `threadpool`

`cpuload` is a private dependency of `threadpool`. The full interaction looks like this (from `threadpool.nim`'s `nimSpawn3`):

```nim
# Inside nimSpawn3 (simplified):
if state.calls < maxPoolSize or (state.calls and 127) == 0:
  if tryAcquire(stateLock):
    case advice(state)          # <-- cpuload.advice called here
    of doNothing: discard
    of doCreateThread:
      if currentPoolSize < maxPoolSize:
        activateWorkerThread(currentPoolSize)
        atomicInc currentPoolSize
    of doShutdownThread:
      if currentPoolSize > minPoolSize:
        workersData[currentPoolSize-1].shutdown = true
    release(stateLock)
```

Key observations:
- `advice` is only called under `stateLock`, so it is never called concurrently from multiple threads.
- The caller applies its own guards (`currentPoolSize < maxPoolSize`, etc.) before acting on the advice.
- `state` is a module-level `var` in `threadpool`, persistent for the entire program lifetime.
