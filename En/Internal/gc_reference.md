# gc.nim — Complete Reference

> **Module**: `system/gc` — Nim's Runtime Library  
> **Author**: Andreas Rumpf  
> **Algorithm**: Deferred Reference Counting + Mark & Sweep cycle detection  
> **Scope**: This is an *internal* runtime module. Most of the functions here are called exclusively by the Nim compiler's code generator (`compilerproc`) or the runtime library (`rtl`). The handful of functions marked `*` (exported with an asterisk) form the **public GC API** visible to ordinary Nim programs.

---

## How the GC Works — A Mental Model

Before diving into individual functions it helps to understand the three interacting mechanisms this file implements.

### 1. Deferred Reference Counting (the normal path)

Every heap-allocated object (string, sequence, `ref` value, closure) is called a **cell**. Each cell carries a hidden header containing a reference counter and a pointer to the object's type descriptor. The reference counter is stored shifted left by 3 bits so the lowest three bits can hold a *colour* used by cycle detection.

When Nim code assigns, passes, or drops a reference, the compiler emits calls to `incRef`/`decRef`. If a counter falls to zero the cell is added to the **ZCT** (Zero Count Table) for deferred collection — it is not freed immediately because the object might still be reachable from a stack variable (stack references are deliberately *not* counted for performance).

### 2. ZCT Collection (the fast path)

`collectZCT` drains the ZCT. For every candidate it checks whether the real reference count is still zero. If so, it calls the object's finalizer (if any), decrements the reference counters of all child objects (which may add them to the ZCT in turn), and physically frees the memory. The whole process is effectively *iterative deep freeing* without recursion.

### 3. Mark & Sweep Cycle Detection (the slow path)

Reference counting alone cannot free reference cycles (A → B → A). When occupied heap memory exceeds `cycleThreshold`, `collectCycles` runs a mark-and-sweep pass: it marks every cell reachable from the stack, registers, and global roots, then sweeps away everything that was *not* marked.

### The Cell Header Layout

```
 ┌──────────────────────────────────────────────────┐
 │  Cell header  (hidden, at negative offset)       │
 │  ┌─────────────────────────────────┬──────────┐  │
 │  │  refcount (bits 3…63)  │ color (bits 0..2) │  │
 │  └─────────────────────────────────┴──────────┘  │
 │  typ: PNimType                                   │
 └──────────────────────────────────────────────────┘
 ┌──────────────────────────────────────────────────┐
 │  User data  ← pointer returned to Nim code      │
 └──────────────────────────────────────────────────┘
```

The colour bits encode the cell's role in cycle detection:

| Constant | Bits | Meaning |
|----------|------|---------|
| `rcBlack` | `000` | In use or free — not a candidate |
| `rcGray` | `001` | Possible member of a cycle |
| `rcWhite` | `010` | Confirmed member of a garbage cycle |
| `rcPurple` | `011` | Possible root of a cycle |

---

## Important Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `CycleIncrease` | `2` | Multiplicative factor by which thresholds grow after each collection |
| `InitialCycleThreshold` | `4 MB` (or `high(int)` if `nimCycleBreaker` is defined) | Occupied-memory level at which cycle collection is triggered |
| `InitialZctThreshold` | `500` | ZCT entry count at which a collection is triggered |
| `rcIncrement` | `0b1000` (8) | The step added/subtracted to the raw `refcount` field per reference |
| `rcShift` | `3` | Number of bits to shift right to get the actual reference count integer |
| `ZctFlag` | `0b100` | Bit set in `refcount` while a cell is in the ZCT |

---

## Key Internal Types

### `GcStat`

A statistics record accumulated throughout a program's lifetime.

| Field | Type | Description |
|-------|------|-------------|
| `stackScans` | `int` | How many times the C stack was scanned |
| `cycleCollections` | `int` | How many full mark-and-sweep cycles have run |
| `maxThreshold` | `int` | Highest value `cycleThreshold` has ever reached |
| `maxStackSize` | `int` | Largest observed C stack size |
| `maxStackCells` | `int` | Largest `decStack` seen during a collection |
| `cycleTableSize` | `int` | Peak size of the cycle-detection table |
| `maxPause` | `int64` | Longest observed GC pause in nanoseconds |

### `GcStack`

Describes one C stack (there is normally exactly one; coroutines may have several).

| Field | Available when | Description |
|-------|---------------|-------------|
| `bottom` | always | Pointer to the lowest used address of this stack |
| `pos` | `withRealTime` or `nimCoroutines` | Current stack pointer, used to limit scan depth |
| `bottomSaved` | `withRealTime` | Saved `bottom` during a `GC_step` call with a custom `stackSize` |
| `prev`, `next` | `nimCoroutines` | Doubly-linked list links for coroutine stacks |
| `maxStackSize` | `nimCoroutines` | Per-coroutine stack size maximum |

### `GcHeap`

The central GC state, stored as a per-thread variable (`gch`).

| Field | Description |
|-------|-------------|
| `stack` | The current C stack descriptor |
| `cycleThreshold` | Occupied-memory high-water mark that triggers cycle collection |
| `zctThreshold` | ZCT length that triggers a ZCT collection |
| `zct` | The Zero Count Table — cells whose refcount just hit zero |
| `decStack` | Cells found on the stack that need their refcount decremented after scanning |
| `tempStack` | Scratch stack used to eliminate recursion during marking |
| `recGcLock` | Re-entrancy counter; non-zero disables the GC (used by `GC_disable`) |
| `maxPause` | Maximum pause time (nanoseconds) for real-time mode |
| `region` | The underlying memory region (arena allocator) |
| `stat` | Accumulated statistics |
| `marked` | The cell-set built during mark-and-sweep |
| `additionalRoots` | Extra GC roots registered via `GC_ref` / `nimGCref` |
| `toDispose` | Thread-safe disposal list (multi-threaded builds) |
| `gcThreadId` | Zero-based index identifying which thread this heap belongs to |

### `WalkOp`

An enum used as the second argument to every function that traverses a cell's children.

| Value | Purpose |
|-------|---------|
| `waMarkGlobal` | Mark a cell during the global (backup) mark-and-sweep pass |
| `waMarkPrecise` | Push a cell onto `tempStack` during the precise mark pass |
| `waZctDecRef` | Decrement the refcount of a child as it is freed by ZCT collection |
| `waPush` | Push the child cell onto `tempStack` without further action |

### `Finalizer`

```nim
Finalizer = proc (self: pointer) {.nimcall, gcsafe, raises: [].}
```

A callback stored in a type's descriptor. Called by `prepareDealloc` immediately before a cell's memory is physically freed, while all child references are still valid.

---

## Public API (exported with `*`)

These are the only functions that ordinary Nim application code is expected to call.

---

### `GC_disable()`

**Increments** the re-entrancy lock counter `gch.recGcLock`. While this counter is greater than zero the GC will not collect, even if thresholds are exceeded. Calls can be nested; each `GC_disable` must be matched by exactly one `GC_enable`.

**When to use**: around a critical section where you need zero allocator activity (e.g., a lock-free data structure manipulation, performance measurement, or calling C code that is not GC-safe).

```nim
GC_disable()
# ... critical section; no GC collection will happen here ...
GC_enable()
```

> **Warning**: if you forget to call `GC_enable`, memory will accumulate indefinitely because nothing is ever freed.

---

### `GC_enable()`

**Decrements** `gch.recGcLock`. If the counter was already zero and `nimDoesntTrackDefects` is not defined, raises `AssertionDefect` to catch mismatched disable/enable pairs.

```nim
GC_disable()
try:
  riskyOperation()
finally:
  GC_enable()   # always re-enable, even on exception
```

---

### `GC_fullCollect()`

Forces an **immediate, complete collection** by:
1. Temporarily setting `cycleThreshold` to zero (so that `collectCT` will unconditionally run cycle detection).
2. Calling `collectCT`, which in turn drains the ZCT and runs `collectCycles`.
3. Restoring the original threshold.

Use this when you know you have just released a large object graph and want the memory back *now*, rather than waiting for the next automatic trigger.

```nim
# Release a large tree
bigTree = nil
# Force the memory back to the OS immediately
GC_fullCollect()
echo getOccupiedMem()   # should be much smaller now
```

---

### `GC_enableMarkAndSweep()`

Resets `gch.cycleThreshold` back to `InitialCycleThreshold` (4 MB), re-enabling cycle collection after it was suppressed. Has no effect if cycle collection was never disabled.

```nim
GC_disableMarkAndSweep()
# ... do work that cannot afford the cycle-collector pause ...
GC_enableMarkAndSweep()
```

---

### `GC_disableMarkAndSweep()`

Sets `gch.cycleThreshold` to `high(int) - 1`, making the cycle-collection condition never true. Reference counting still works; only the mark-and-sweep phase that breaks cycles is suppressed.

**When to use**: latency-sensitive sections, or when you know your object graph contains no cycles and you want to avoid the overhead of the mark-and-sweep pass.

```nim
GC_disableMarkAndSweep()
# Pure tree structures — no cycles possible
buildHugeTree()
GC_enableMarkAndSweep()
GC_fullCollect()
```

---

### `GC_getStatistics(): string`

Returns a human-readable multiline string reporting the current GC state and accumulated statistics. Useful for performance profiling and debugging memory issues.

```nim
echo GC_getStatistics()
```

**Example output**:
```
[GC] total memory: 8388608
[GC] occupied memory: 102400
[GC] stack scans: 14
[GC] stack cells: 3
[GC] cycle collections: 1
[GC] max threshold: 4194304
[GC] zct capacity: 64
[GC] max cycle table size: 0
[GC] max pause time [ms]: 0
[GC] max stack size: 16384
```

Each line begins with `[GC]` so it can be filtered from normal program output.

---

### `GC_setStrategy(strategy: GC_Strategy)`

A no-op stub retained for backward compatibility. The old Nim GC had multiple strategies; the current implementation ignores this setting.

```nim
GC_setStrategy(gcThroughput)  # silently ignored
```

---

### `GC_setMaxPause(MaxPauseInUs: int)`

*Available only when compiled with `-d:useRealtimeGC`.*

Sets the **maximum allowed GC pause** in microseconds. When set, `collectZCT` will interrupt itself and return `false` if the elapsed wall time exceeds this limit, allowing the calling thread to meet soft real-time deadlines.

```nim
GC_setMaxPause(500)   # GC may not pause for more than 500 µs at a time
```

After setting the pause limit, call `GC_step` regularly from your main loop to spread GC work across frames.

---

### `GC_step(us: int, strongAdvice = false, stackSize = -1)`

*Available only when compiled with `-d:useRealtimeGC`.*

Asks the GC to perform at most `us` microseconds of work in this call. The GC decides whether to actually run based on internal thresholds, unless `strongAdvice = true` forces a collection regardless.

`stackSize` can be used to tell the GC exactly how deep the current stack is (in bytes). This is useful in coroutine-like frameworks where the C stack size is known in advance and you want to limit scan depth. Pass `-1` to use the full stack.

```nim
while true:
  handleEvents()
  GC_step(1000)         # spend at most 1 ms on GC per frame
  renderFrame()
```

---

### `GC_collectZct()`

*Experimental, unstable — do not use in production.*

Directly invokes `collectCTBody`, the core ZCT drain + optional cycle collection routine, without checking any thresholds. Intended for testing the GC internals.

---

### `gcInvariant()`

*Exported for debugging tools.*

Asserts that the allocator's internal invariant holds (via `sysAssert(allocInv(...))`). When `traceGC` is enabled it also runs `markForDebug`. Used by Nim's test suite to verify GC consistency between operations.

```nim
gcInvariant()  # will crash with a debug message if the heap is corrupt
```

---

## Compiler-Facing Functions (`compilerproc`)

These functions are not called by user code directly — the Nim compiler emits calls to them as part of the code it generates. They are documented here because understanding them helps when reading generated C code or diagnosing subtle memory bugs.

---

### `nimGCref(p: pointer)`

**`GC_ref` equivalent for raw pointers.** Increments the refcount of the cell at `p` and adds it to `gch.additionalRoots` so that the mark-and-sweep phase will treat it as a root even if no Nim-typed reference to it exists. The higher-level `GC_ref` proc in the standard library calls this.

```nim
# Nim code:
var x: ref MyObj = new(MyObj)
GC_ref(x)   # internally calls nimGCref
```

---

### `nimGCunref(p: pointer)`

**`GC_unref` equivalent for raw pointers.** Removes `p` from `gch.additionalRoots` (linear scan) and decrements its refcount. Paired with `nimGCref`.

---

### `nimGCunrefNoCycle(p: pointer)`

Inline version of `decRef` with allocator invariant checks. Called by the compiler when it can prove at compile time that the object being released cannot be part of a reference cycle (e.g., a `string` or a simple non-recursive `ref`). Saves the overhead of the full `nimGCunref` path.

---

### `nimGCunrefRC1(p: pointer)`

Even thinner wrapper around `decRef`. Used in situations where allocator invariant checks are not needed (e.g., during certain finalization sequences).

---

### `asgnRef(dest: PPointer, src: pointer)`

**The standard reference assignment.** Called by the compiler whenever a heap-allocated reference is assigned to a heap-allocated location (`dest` is provably *not* on the stack):

1. If `src != nil`: increment `src`'s refcount.
2. If `dest[]` is not nil: decrement the old value's refcount.
3. Write `src` into `*dest`.

The "incRef before decRef" order is critical — if both pointers happen to be the same object (self-assignment), decrementing first would cause a premature free.

```c
// Generated C for:   node.child = newChild
asgnRef(&node->child, newChild);
```

---

### `asgnRefNoCycle(dest: PPointer, src: pointer)`

Deprecated alias for `asgnRef`, retained for compatibility with old generated code. Both are identical at runtime.

---

### `unsureAsgnRef(dest: PPointer, src: pointer)`

**The uncertain reference assignment.** Called when the compiler cannot determine at compile time whether `dest` is on the stack or the heap (the classic case is a `var ref T` parameter). It checks `isOnStack(dest)` at runtime:

- If `dest` **is on the stack**: skip all refcount updates (stack references are not counted), just write `src` into `*dest`.
- If `dest` **is on the heap**: perform the full `asgnRef` logic.

This is slightly more expensive than `asgnRef` due to the stack check, but the check is just two comparisons on typical platforms with contiguous stacks.

```c
// Generated C for:   setRef(&r)   where r might be stack or heap
unsureAsgnRef(&r, newObj(TNode_TI, sizeof(TNode)));
```

---

### `newObj(typ: PNimType, size: int): pointer`

Allocates `size` bytes of zeroed memory for an object of type `typ`, records it in the ZCT with refcount 0, and returns a pointer to the user-data area (i.e., the area *after* the hidden Cell header). The compiler emits this for every `new(T)` expression.

---

### `newObjNoInit(typ: PNimType, size: int): pointer`

Like `newObj` but skips the `zeroMem` call. Used by the compiler when it can prove that every field will be written before it is read (e.g., aggregate construction literals).

---

### `newObjRC1(typ: PNimType, size: int): pointer`

Like `newObj` but initialises the refcount to **1** instead of 0. The "RC1" suffix signals that the resulting pointer already owns one reference, so the compiler can skip the first `incRef` call at the point of first assignment.

---

### `newSeq(typ: PNimType, len: int): pointer`

Allocates a sequence (dynamic array) of `len` elements. Computes the total size as `GenericSeqSize + len × element_size`, calls `newObj`, then sets `len` and `reserved` in the sequence header.

---

### `newSeqRC1(typ: PNimType, len: int): pointer`

Same as `newSeq` but pre-charges the refcount to 1 (calls `newObjRC1`). Used in optimised assignment paths.

---

### `growObj(old: pointer, newsize: int): pointer`

Grows an existing sequence or string to `newsize` bytes. Allocates a fresh cell, copies the existing content, zero-fills the new tail, patches the old sequence header's `len` to 0 (so any lingering reference to `old` sees an empty sequence), and returns the new cell's user pointer.

This is called by the compiler when `add`, `setLen`, or `&` trigger a reallocation.

---

### `nimGCvisit(d: pointer, op: int)`

The generic child-visitation entry point used by GC marker functions. The compiler generates a per-type `marker` procedure that calls `nimGCvisit` for every traced field. The `op` argument is a `WalkOp` cast to `int` indicating the action to take (push to temp stack, decref, etc.).

---

### `extGetCellType(c: pointer): PNimType`

Returns the runtime type descriptor of the object at `c`. Used by debugging and profiling tools that need to identify an object's type from a raw pointer.

---

### `internRefcount(p: pointer): int`

Returns the current reference count of the object at `p`. Exported to C as `getRefcount`. Available to diagnostic tools; do not use in application logic (the value is not stable across GC points).

```nim
# Low-level diagnostic only:
echo internRefcount(addr someObj)
```

---

## Internal Helper Functions

These functions are not exported and not called by user code, but understanding them is essential for reading or modifying the GC itself.

---

### `cellToUsr(cell: PCell): pointer`

Converts an internal `PCell` pointer (pointing to the hidden header) into a user-facing pointer (pointing to the actual object data). This is simply `cell + sizeof(Cell)`.

### `usrToCell(usr: pointer): PCell`

The inverse of `cellToUsr`. Converts a user pointer back to the header. This is `usr - sizeof(Cell)`. Every compiler-facing function that receives a `pointer` from Nim code calls this first.

### `incRef(c: PCell)`

Adds `rcIncrement` (8) to `c.refcount`. Inline, no heap check at runtime in release builds.

### `decRef(c: PCell)`

Subtracts `rcIncrement` from `c.refcount`. If the result drops below `rcIncrement` (i.e., the true count is now zero), adds the cell to the ZCT via `rtlAddZCT`.

### `addZCT(s, c)`

Adds cell `c` to ZCT sequence `s` after setting its `ZctFlag` bit. The flag prevents double-insertion.

### `isOnStack(p: pointer): bool`

Checks whether pointer `p` falls within the bounds of the current C stack. On platforms with contiguous stacks this is two comparisons. Returns `true` if `p` is a stack address.

### `rawNewObj(typ, size, gch)`

The lowest-level allocator: reserves `size + sizeof(Cell)` bytes from `gch.region`, sets up the Cell header with refcount = `ZctFlag` (zero count, flagged for ZCT), and adds it to the ZCT. All public allocation procs (`newObj`, `newObjRC1`, etc.) call this.

### `collectZCT(gch): bool`

Drains the ZCT until it is empty or (in real-time mode) until the pause budget is exhausted. Returns `true` if the ZCT was fully drained, `false` if interrupted.

### `collectCycles(gch)`

Runs the full mark-and-sweep cycle collector: drains the ZCT first (to get clean colours), marks everything reachable from the stack, registers, and global roots, then sweeps everything not marked.

### `collectCT(gch)`

The main collection gateway. Decides whether conditions are met for a collection (ZCT size, occupied memory, or the `alwaysGC` debug flag), and if so calls `collectCTBody`. Also updates `zctThreshold` adaptively after each collection.

### `collectCTBody(gch)`

The actual body of a collection cycle:
1. Records max stack size.
2. Calls `markStackAndRegisters` to scan the C stack and CPU registers.
3. Calls `collectZCT`.
4. If ZCT was fully drained and memory exceeds `cycleThreshold`, calls `collectCycles`.
5. Calls `unmarkStackAndRegisters` to decrement counts boosted during scanning.
6. In real-time mode, records the duration as a pause-time sample.

### `markStackAndRegisters(gch)`

Walks every word on the C stack and in CPU registers, calling `gcMark` for each. Annotated with `CLANG_NO_SANITIZE_ADDRESS` to suppress false ASAN positives caused by conservative scanning of arbitrary memory words.

### `gcMark(gch, p)`

Called for each potential pointer found on the stack. Checks whether `p` looks like a valid heap address (above `PageSize`), then uses `interiorAllocatedPtr` to locate the start of any cell that contains `p` (interior pointer support). If found, increments that cell's refcount and records it in `decStack` so the increment is undone after scanning.

### `unmarkStackAndRegisters(gch)`

Decrements the refcount of every cell in `decStack` (the ones temporarily incremented during the stack scan), then clears `decStack`.

### `markS(gch, c)`

Marks a single cell `c` and all its transitive children by iteratively popping from `gch.tempStack`. Avoids C-stack recursion by using `tempStack` as an explicit worklist.

### `markGlobals(gch)`

Invokes all registered global marker functions and thread-local marker functions, then marks all `additionalRoots`. Called during `collectCycles` to ensure globally reachable cells are not swept.

### `sweep(gch)`

Iterates over every object in `gch.region`. For each cell not in `gch.marked` (and not currently in the ZCT), calls `freeCyclicCell` to run its finalizer and release its memory.

### `freeCyclicCell(gch, c)`

Runs `prepareDealloc(c)` (finalizer), then physically frees `c` with `rawDealloc`.

### `forAllChildren(cell, op)`

Dispatches `op` over all GC-traced children of `cell`. Uses the type's `marker` function if available (compiler-generated, fast path), otherwise falls back to the reflective `forAllChildrenAux` path.

### `forAllChildrenAux(dest, mt, op)`

Reflective child walker. Interprets the `PNimType` descriptor to find traced fields in objects, tuples, and arrays recursively.

### `forAllSlotsAux(dest, n, op)`

Walks the `TNimNode` tree of a type descriptor to visit each slot in an `object` or `tuple`, handling `nkSlot`, `nkList`, and `nkCase` (variant objects) nodes.

### `doOperation(p, op)`

The dispatch hub for `WalkOp`. Called once per child pointer during a traversal:
- `waZctDecRef` → `decRef(c)`
- `waPush` / `waMarkPrecise` → `add(gch.tempStack, c)`
- `waMarkGlobal` → `markS(gch, c)`

### `addNewObjToZCT(res, gch)`

Optimised ZCT insertion for freshly-allocated objects. Scans the last 8 entries of the ZCT for a slot already in the ZCT (refcount ≥ `rcIncrement` means it has been "promoted" and the slot can be reused). This cache-friendly search succeeds ~63% of the time and avoids appending to the ZCT unnecessarily.

### `initGC()`

Resets all GC state to initial values. Called once at program startup (and once per thread). Sets thresholds, zeroes statistics, initialises all internal sequences and the cell-set.

### `cellsetReset(s)`

Clears a `CellSet` by calling `deinit` then `init`.

---

## Conditional Compilation Flags

The behaviour of gc.nim is controlled by several compile-time flags:

| Flag | Effect |
|------|--------|
| `-d:useRealtimeGC` | Enables `GC_setMaxPause` and `GC_step` for incremental, time-limited collection |
| `-d:nimCycleBreaker` | Sets `InitialCycleThreshold = high(int)`, effectively disabling automatic cycle collection at startup |
| `-d:logGC` | Enables verbose per-cell logging to stderr via `writeCell` |
| `-d:useGcAssert` | Enables runtime `gcAssert` checks (expensive; for debugging the GC itself) |
| `-d:memProfiler` | Calls `nimProfile` after every allocation to record allocation sizes |
| `--threads:on` | Enables the `toDispose` shared list for cross-thread disposal |
| `--define:nimCoroutines` | Adds support for multiple stacks (one per coroutine) |
| `--define:traceGC` | Enables per-cell state transition tracing |
| `--define:leakDetector` | Records source filename and line number in each cell header |

---

## Quick Reference: Which Function Fires When

| User-level Nim action | Function called |
|-----------------------|-----------------|
| `new(T)` | `newObj` or `newObjRC1` |
| `@[...]` / `newSeq` | `newSeq` or `newSeqRC1` |
| `a = b` (both on heap) | `asgnRef` |
| `setRef(r)` with `var ref` | `unsureAsgnRef` |
| `add(s, x)` triggers realloc | `growObj` |
| scope exit / last ref dropped | `decRef` → ZCT add |
| ZCT size ≥ threshold | `collectZCT` |
| heap ≥ cycle threshold | `collectCycles` |
| `GC_fullCollect()` | `collectCT` with threshold = 0 |
| `GC_ref(x)` | `nimGCref` |
| `GC_unref(x)` | `nimGCunref` |
