# `nimprof` Module Reference

**Module:** `nimprof` (part of Nim's standard library)  
**Purpose:** Embedded stack-trace profiler — measures where your program spends its time (CPU profiling) or which call sites allocate the most memory (memory profiling).

---

## Overview

`nimprof` is an *embedded* profiler, meaning it runs inside your own program rather than as a separate tool. You activate it purely through compiler flags; no external agent or OS tool is needed.

The profiler intercepts execution at regular intervals, captures the current stack trace, and tallies how many times each unique trace was observed. When the program exits it writes a human-readable report to `profile_results.txt`.

### Activation

```sh
# CPU / time profiling (default mode)
nim c --profiler:on --stackTrace:on myapp.nim

# Memory / allocation profiling
nim c --profiler:on --stackTrace:on -d:memProfiler myapp.nim
```

> **Important:** Simply importing `nimprof` without passing `--profiler:on` to the compiler is a compile-time error. The module enforces this with a `{.error.}` pragma.

### How it works internally

1. The Nim runtime calls a lightweight *hook* function on every procedure entry (or periodically, depending on the interval setting).
2. Each hook call hashes the current stack trace and stores it in a fixed-size hash table (`ProfileData`, 64 K slots).
3. When the table is full, the least-frequent entry is evicted to make room for new ones (LRU-like eviction).
4. On program exit, `writeProfile` sorts entries by hit count, formats them, and saves the file.

---

## Exported API

The module exports exactly **three** symbols for public use:

| Symbol | Kind | Mode |
|---|---|---|
| `setSamplingFrequency` | proc | CPU profiling only |
| `disableProfiling` | proc | both modes |
| `enableProfiling` | proc | both modes |

---

## `setSamplingFrequency`

```nim
proc setSamplingFrequency*(intervalInUs: int)
```

### What it does

Controls *how often* the profiler samples the call stack.

- The parameter `intervalInUs` is expressed in **microseconds** (µs).
- Internally the value is converted to nanoseconds and stored as the target interval between samples.
- A small correction constant (`tickCountCorrection = 50_000 ns`) is subtracted to compensate for the overhead of reading the system timer.
- **Default:** 5 000 µs (= 5 ms).

**Special case — `intervalInUs <= 0`:**  
Passing zero or a negative value switches the profiler from *time-based* sampling to *instruction-count-based* sampling. In this mode the hook fires on every procedure call rather than waiting for the timer. This produces more data but significantly increases overhead and is better suited to very short-running programs where the time-based sampler would collect nothing at all.

### When to call it

Call `setSamplingFrequency` **before** the code you want to measure. Calling it after the hot section has already executed will not retroactively change what was captured.

### Examples

```nim
import nimprof

# Example 1 — coarser sampling: one sample every 20 ms.
# Reduces overhead; good when the program runs for many seconds.
setSamplingFrequency(20_000)

# Example 2 — finer sampling: one sample every 1 ms.
# More detail, higher overhead; good for short benchmarks.
setSamplingFrequency(1_000)

# Example 3 — instruction-count mode.
# Every procedure call is counted. Very high overhead.
setSamplingFrequency(0)

runMyHotLoop()
```

> **Tip:** If `profile_results.txt` shows very few entries, your program is finishing faster than one sampling interval. Try `setSamplingFrequency(1_000)` or even `setSamplingFrequency(0)`.

> **Note:** This proc is **not available** when the module is compiled with `-d:memProfiler`. Memory profiling uses a fixed sampling interval of 1-in-50 000 allocations, which is not configurable through this API.

---

## `disableProfiling`

```nim
proc disableProfiling*()
```

### What it does

Temporarily **suspends** data collection by detaching the profiling hook from the Nim runtime. After this call, procedure entries are no longer intercepted and no new stack traces are recorded — until `enableProfiling` is called again.

Internally the procedure uses an atomic counter (`disabled`) so that nested enable/disable pairs are balanced correctly even in multi-threaded code.

### When to use it

Use `disableProfiling` to **exclude** sections of code that are not interesting for your current analysis. Common scenarios:

- Startup/teardown boilerplate that would skew the results.
- I/O-heavy sections (reading config files, logging) where you only care about CPU work.
- Library initialization that you do not own.

### Example

```nim
import nimprof

disableProfiling()           # don't count program startup
loadConfiguration("app.cfg") # irrelevant initialization
connectToDatabase()          # slow but not what we're measuring
enableProfiling()            # start measuring from here

runCriticalAlgorithm()       # ← this is what we care about

disableProfiling()           # stop before teardown
closeDatabase()
```

> **Warning:** Calls to `disableProfiling` and `enableProfiling` are reference-counted. Every `disableProfiling` call must be matched by an `enableProfiling` call to restore the hook. Unbalanced calls will leave the profiler permanently disabled for the rest of the run.

---

## `enableProfiling`

```nim
proc enableProfiling*()
```

### What it does

Re-attaches the profiling hook after a call to `disableProfiling`, resuming data collection. If the internal counter indicates that profiling should be active, the hook is reinstalled into `system.profilingRequestedHook`.

The check `atomicInc(disabled) >= 0` ensures thread safety: the hook is only reinstalled when the counter transitions back to non-negative, preventing races between threads that might call `enableProfiling` concurrently.

### When to use it

Always pair with `disableProfiling`. You can nest multiple enable/disable pairs around different code regions, though nesting is rarely needed.

### Example

```nim
import nimprof

# Profile only a specific phase of a multi-phase pipeline.
disableProfiling()
phase1_dataIngest()          # not interesting

enableProfiling()
phase2_transform()           # ← measure this

disableProfiling()
phase3_outputToDisk()        # I/O, not interesting

enableProfiling()
phase4_postProcess()         # ← measure this too

disableProfiling()           # clean exit before teardown
```

---

## Output — `profile_results.txt`

The report is written automatically when the program exits (via `addExitProc`). You do not need to call anything.

### Format

```
total executions of each stack trace:
Entry: 1/42 Calls: 3812/10000 = 38.12% [sum: 3812; 3812/10000 = 38.12%]
  mymodule.nim: hotFunction  3812/10000 = 38.12%
  main.nim: main  10000/10000 = 100.00%
Entry: 2/42 Calls: 2100/10000 = 21.00% [sum: 5912; 5912/10000 = 59.12%]
  ...
```

- **Entry N/M** — this is the N-th most frequent trace out of M unique traces recorded.
- **Calls X/Y = Z%** — this trace was sampled X times out of Y total samples.
- **sum** — cumulative samples across all entries printed so far.
- Each indented line is one frame in the stack, from innermost to outermost.
- The percentage after the function name shows how often *that function* appeared across **all** traces (not just this entry's trace), giving a flat-profile view.
- At most 100 entries are written even if more were collected.

---

## Full working example

```nim
# compile with:
#   nim c --profiler:on --stackTrace:on -r example.nim

import nimprof

proc fib(n: int): int =
  if n < 2: return n
  return fib(n - 1) + fib(n - 2)

proc heavyWork() =
  for i in 1..30:
    discard fib(i)

proc lightWork() =
  for i in 1..1_000_000:
    discard i * i

setSamplingFrequency(500)   # sample every 0.5 ms for finer detail

disableProfiling()
echo "Starting benchmark..."
enableProfiling()

heavyWork()
lightWork()

disableProfiling()
echo "Done. Check profile_results.txt"
```

After running, `profile_results.txt` will show `fib` dominating the top entries, reflecting its recursive and therefore frequently sampled nature.

---

## Caveats and limitations

- The hash table holds at most ~96 K unique stack traces. When full, rare traces are silently evicted.
- Stack traces are capped at 21 frames. Very deep call stacks are truncated.
- The profiler measures *sampling hits*, not wall-clock time directly. A function called briefly but very often may appear higher than one called rarely but for a long time — unless time-based sampling is used.
- Thread safety is achieved via a global lock (`profilingLock`). In highly parallel programs this lock can become a bottleneck.
- `setSamplingFrequency` is not available in memory-profiler mode (`-d:memProfiler`).
