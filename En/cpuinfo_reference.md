# Cpuinfo Module — Reference

---

## What This Module Does

`cpuinfo` solves a single, focused problem: how many processor cores does the machine running this program actually have? The answer is needed by almost every program that distributes work across threads — you want to create roughly as many worker threads as there are cores, no more and no less.

The module exposes exactly one procedure, `countProcessors`, and hides behind it the full complexity of querying the operating system on every platform Nim supports. Each OS has its own API for this, and those APIs differ substantially in calling conventions, required headers, and the meaning of the returned value.

---

## Understanding What "Number of Processors" Means

The term is deliberately broad. What `countProcessors` returns is the number of **logical processors** visible to the operating system — the count of execution units the scheduler can assign threads to independently. On a machine with hyperthreading or SMT (Simultaneous Multi-Threading), each physical core appears as two logical processors. On a machine without hyperthreading, logical and physical cores are the same thing.

For most parallel workloads, the logical processor count is the right number to use when sizing a thread pool. It is the number of threads that can physically run in parallel at any given moment.

---

## Platform Implementation Details

The implementation is entirely selected at compile time via `when defined(...)` branches. There is no runtime platform detection. Understanding which branch runs on your target helps predict edge cases.

| Platform | API used | Notes |
|----------|----------|-------|
| **Windows** | `GetSystemInfo` (Win32) | Reads `dwNumberOfProcessors` from the `SYSTEM_INFO` struct. Returns logical processors. |
| **macOS** | `sysctlbyname("hw.logicalcpu", ...)` | Queries the `hw.logicalcpu` sysctl key, which is an alias for `hw.activecpu`. Falls back to `sysctl([CTL_HW, HW_NCPU])` if the first call fails. |
| **BSD** (FreeBSD, OpenBSD, NetBSD) | `sysctl([CTL_HW, HW_NCPU])` | Uses the classic `HW_NCPU` sysctl constant. |
| **Linux / POSIX** (default) | `sysconf(SC_NPROCESSORS_ONLN)` | Returns only **online** processors — CPUs that are currently enabled and available to the scheduler. Processors that exist in hardware but have been taken offline (e.g. for power saving) are not counted. |
| **HP-UX** | `mpctl(MPC_GETNUMSPUS, ...)` | Uses HP-UX's multiprocessor control interface. |
| **IRIX** | `sysconf(_SC_NPROC_ONLN)` | SGI IRIX variant of the POSIX sysconf call. |
| **Haiku** | `get_system_info` | Reads `cpu_count` from the `system_info` struct from `<OS.h>`. |
| **Genode** | `env.cpu().affinity_space().total()` | Queries the Genode environment's CPU affinity space. |
| **Node.js** | `require("os").cpus().length` | Uses the Node.js `os` module. |
| **Browser / Deno** | `navigator.hardwareConcurrency` | Uses the Web API property available in browsers and Deno. |

---

## Exported Procedure

### `countProcessors`

```nim
proc countProcessors*(): int
```

#### Description

Queries the operating system and returns the number of logical processors (CPU cores) available on the current machine. The call delegates immediately to a platform-specific internal implementation selected at compile time.

The function is stateless and has no side effects. It queries the OS directly on every call without caching, though in practice the OS typically reads the value from a kernel data structure and the call is very fast.

#### Return value

A non-negative integer:
- A **positive** integer — the number of logical processors the OS reports as available.
- **Zero** — the count could not be determined. This happens either because the platform-specific query returned a negative value (which is normalised to zero) or because the OS call failed. Zero should be treated as "unknown", not as "no processors".

The function **never returns a negative number**.

#### Parameters

None.

#### Examples

```nim
import cpuinfo

let cores = countProcessors()
echo "Logical processors: ", cores
# → e.g. "Logical processors: 8" on a quad-core with hyperthreading
```

```nim
import cpuinfo

# Sizing a thread pool — the most common use case
let poolSize = max(1, countProcessors())
# max(1, ...) guards against the zero/unknown case
```

```nim
import cpuinfo, threadpool

setMaxPoolSize(countProcessors())
```

```nim
import cpuinfo

# Deciding whether parallel processing is worth the overhead
let n = countProcessors()
if n >= 2:
  echo "Running in parallel mode across ", n, " cores"
else:
  echo "Single-core or unknown; running sequentially"
```

#### Edge cases and caveats

**Containers and virtual machines.** In Docker containers and other cgroup-constrained environments on Linux, `sysconf(SC_NPROCESSORS_ONLN)` typically returns the number of logical processors of the **host** machine, not the number allocated to the container. The cgroup CPU quota (e.g. `--cpus=2`) does not reduce the value returned by this call. Code that needs container-aware CPU counts must read cgroup files directly or use a library that does so.

**CPU hotplug.** On Linux, processors can be taken online or offline at runtime. `SC_NPROCESSORS_ONLN` reflects the count at the moment of the call. If CPUs are dynamically offlined after program start, subsequent calls may return a lower number.

**Zero return.** If `countProcessors()` returns zero, the safest course is to fall back to a thread count of 1 (purely sequential execution). A pool of zero threads would be a programmer error.

**Hyperthreading.** The returned value counts logical threads, not physical cores. On a 4-core Intel CPU with hyperthreading, the return value is typically 8. Whether to use the full logical count or half of it for CPU-bound work is a performance tuning question — there is no single right answer.

**NUMA systems.** On multi-socket NUMA machines, `countProcessors` returns the total count across all sockets. It does not provide topology information (which core is on which socket). For NUMA-aware programming, use platform-specific APIs.

---

## Compile-time Configuration

There are no compile-time flags that modify the behaviour of `countProcessors` itself. The platform branch is selected automatically by the Nim compiler based on the target OS.

The only relevant compile flag is `--threads:on`, which is required when using the result of `countProcessors` to size a thread pool (e.g. with `threadpool`'s `setMaxPoolSize`), but is not required to import and call `countProcessors` itself.

---

## Relation to Other Modules

`countProcessors` is used internally by `threadpool` during its `setup()` call (which runs automatically at program start) to determine the initial pool size:

```nim
# Inside threadpool.nim (simplified):
proc setup() =
  let p = countProcessors()
  currentPoolSize = min(p, MaxThreadPoolSize)
  for i in 0..<currentPoolSize:
    activateWorkerThread(i)
```

This means that in most programs using `threadpool`, you do not need to call `countProcessors` yourself — it has already been called for you.
