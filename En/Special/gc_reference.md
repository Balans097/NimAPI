# gc.nim — Module Reference

> **Import:** not imported separately — `lib/system/gc.nim` is pulled in via
> an `include` directive straight into the `system` module when compiling
> with the `refc` backend garbage collector. User code only ever touches a
> small set of `GC_*` procedures, which are available automatically (the
> `system` module is implicitly imported into every Nim program).
> **Scope:** the runtime implementation of Nim's garbage collector, based on
> *Deferred Reference Counting* with a background cycle detector running a
> mark&sweep pass.

> **A note on this reference.** Unlike `std/algorithm`, `gc.nim` is not a
> library module with a public API — it's an internal part of the compiler's
> runtime. The overwhelming majority of procedures here are either
> `{.compilerproc.}` (called only by generated C code) or simply not exported
> (no `*`) and operate on private types `PCell`/`GcHeap` via pointers offset
> from a cell header. There is no such thing as "a typical user call" for
> these — no such call exists. So:
> - for procedures in the **public API** (section VI) below, real, working
>   code examples are given — the way an ordinary Nim user would actually
>   write them;
> - for **internal mechanisms** (sections II–V), instead of "call examples"
>   you'll find **algorithm traces** — a step-by-step illustration of what
>   happens inside the GC while a program runs (creating an object, `x = y`,
>   leaving a scope, etc.). This is not code you can run on its own, but a
>   breakdown of what actually executes underneath an ordinary program.

The core idea of the module: every *cell* is a header holding a `refcount`
field and a pointer to the type, placed right before the user data (before a
`ref`, `seq`, or `string`). Reference assignments increase and decrease
`refcount` via `incRef`/`decRef`. Cells whose counter drops to zero land in
the ZCT (*zero count table*) and get freed in batches. Because plain
reference counting can't catch reference cycles, a periodic mark&sweep pass
(the "cycle collector") runs on top of it, marking objects reachable from the
stack and globals and cleaning up everything unreachable.

---

## Table of Contents

I. [Types and Global State](#i-types-and-global-state)
   1. [`WalkOp`](#walkop)
   2. [`GcHeap` and `gch`](#gcheap-and-gch)
   3. [`refcount` Bit Fields (Cell Color)](#refcount-bit-fields-cell-color)

II. [Reference Counting](#ii-reference-counting)
   1. [`incRef` / `decRef`](#incref--decref)
   2. [`asgnRef`](#asgnref)
   3. [`unsureAsgnRef`](#unsureasgnref)
   4. [`nimGCref` / `nimGCunref`](#nimgcref--nimgcunref)

III. [Object Allocation](#iii-object-allocation)
   1. [`rawNewObj` / `newObj` / `newObjNoInit`](#rawnewobj--newobj--newobjnoinit)
   2. [`newObjRC1` / `newSeqRC1`](#newobjrc1--newseqrc1)
   3. [`growObj`](#growobj)

IV. [Walking the Object Graph](#iv-walking-the-object-graph)
   1. [`forAllChildren` / `forAllChildrenAux`](#forallchildren--forallchildrenaux)
   2. [`markS` / `markGlobals`](#marks--markglobals)
   3. [`gcMark` / `markStackAndRegisters`](#gcmark--markstackandregisters)

V. [The Collection Cycle](#v-the-collection-cycle)
   1. [`collectZCT`](#collectzct)
   2. [`collectCTBody` / `collectCT`](#collectctbody--collectct)
   3. [`collectCycles` / `sweep`](#collectcycles--sweep)

VI. [Public GC Control API](#vi-public-gc-control-api)
   1. [`GC_disable` / `GC_enable`](#gc_disable--gc_enable)
   2. [`GC_fullCollect`](#gc_fullcollect)
   3. [`GC_enableMarkAndSweep` / `GC_disableMarkAndSweep`](#gc_enablemarkandsweep--gc_disablemarkandsweep)
   4. [`GC_getStatistics`](#gc_getstatistics)
   5. [`GC_step` / `GC_setMaxPause`](#gc_step--gc_setmaxpause)

VII. [Practical Recipes](#vii-practical-recipes)
   1. [Diagnosing Leaks via GC Statistics](#1-diagnosing-leaks-via-gc-statistics)
   2. [Disabling the GC During a Critical Section](#2-disabling-the-gc-during-a-critical-section)
   3. [Soft Real-Time GC for Interactive Programs](#3-soft-real-time-gc-for-interactive-programs)

VIII. [Quick Reference Table](#viii-quick-reference-table)

IX. [Summary: What to Pick](#ix-summary-what-to-pick)

---

## I. Types and Global State

### `WalkOp`

```nim
type
  WalkOp = enum
    waMarkGlobal,    # part of the backup/debug mark&sweep
    waMarkPrecise,   # part of the backup/debug mark&sweep
    waZctDecRef, waPush
```

**What it does.** An enum of the operation codes to apply to a child
reference while walking an object. The same walking function
(`forAllChildren`) is reused both for decref-ing when removing a cell from
the ZCT and for marking during cycle search — the actual action is selected
through this enum rather than by separate walker functions for each case.

**Implementation notes.** This is the classic "one graph walker, many
operations" pattern: instead of four copy-pasted versions of the
field-tree-walk (`forAllChildrenAux`, `forAllSlotsAux`), there's a single
version that ends with a `case op` (see `doOperation`) deciding what to do
with each discovered child reference — decrement its refcount
(`waZctDecRef`), queue it for marking (`waPush`/`waMarkPrecise`), or mark it
recursively right away (`waMarkGlobal`).

**List of values:**
- `waMarkGlobal` — mark a reachable object immediately (used when walking
  global variables and additional roots);
- `waMarkPrecise` — don't mark right away, push onto `tempStack` instead
  (avoids deep recursion when walking large graphs);
- `waZctDecRef` — decrement the child object's refcount (when a parent is
  being freed out of the ZCT);
- `waPush` — simply add the cell to `tempStack` without marking.

---

### `GcHeap` and `gch`

```nim
type
  GcHeap {.final, pure.} = object
    stack: GcStack
    cycleThreshold: int
    zctThreshold: int
    zct: CellSeq             # the zero count table
    decStack: CellSeq        # stack cells that need to be decref'd again
    tempStack: CellSeq       # temporary stack for recursion elimination
    recGcLock: int           # lock against re-entering via finalizers
    region: MemRegion        # the GC-managed memory region
    stat: GcStat
    marked: CellSet
    additionalRoots: CellSeq # dummy roots for GC_ref/unref
    gcThreadId: int

var
  gch {.rtlThreadVar.}: GcHeap
```

**What it does.** `GcHeap` is the entire collector state for one thread:
thresholds, three helper cell lists (the ZCT, the decref-stack, the temporary
walk stack), the set of marked cells, and its own managed memory region
(`MemRegion`). The `gch` variable is the global (more precisely, thread-local
— `{.rtlThreadVar.}`) instance of this state; nearly every procedure in the
file reads and mutates it.

**Implementation notes.** The split into `zct` / `decStack` / `tempStack` is
not an accident — it's a way to avoid deep recursion over large object
graphs: wherever a naive implementation would recurse into itself for every
child field, here child cells are pushed onto an array (`tempStack`), and a
`while` loop processes it iteratively (see `markS`). The `recGcLock` field is
a counter of nested `GC_disable()` calls; while it's above zero, `collectCT`
won't start a collection — this guards against re-entering the collector
from inside a finalizer.

**Key fields:**
- `zct: CellSeq` — the queue of cells whose refcount hit zero, awaiting
  release;
- `decStack: CellSeq` — cells found on the stack during the last scan, whose
  temporary incref needs to be rolled back;
- `cycleThreshold: int` / `zctThreshold: int` — the occupied-memory and
  ZCT-size thresholds that trigger the next collection phase once exceeded;
- `marked: CellSet` — the set of cells marked reachable in the current
  mark&sweep pass;
- `recGcLock: int` — the nesting counter for `GC_disable()` calls.

---

### `refcount` Bit Fields (Cell Color)

```nim
const
  rcIncrement = 0b1000 # so that lowest 3 bits are not touched
  rcBlack = 0b000  # cell is colored black; in use or free
  rcGray = 0b001   # possible member of a cycle
  rcWhite = 0b010  # member of a garbage cycle
  rcPurple = 0b011 # possible root of a cycle
  ZctFlag = 0b100  # in ZCT
  rcShift = 3      # shift by rcShift to get the reference counter
  colorMask = 0b011

template color(c): untyped = c.refCount and colorMask
template setColor(c, col) =
  when col == rcBlack:
    c.refcount = c.refcount and not colorMask
  else:
    c.refcount = c.refcount and not colorMask or col
```

**What it does.** The `refcount` field of every cell packs two things into a
single machine word: the reference counter itself (in the high bits, above
`rcShift`) and a "color"/flag in the low three bits. This is classic bit
packing for memory economy — a separate `color: byte` field on every cell
would be wasteful.

**Implementation notes.** `rcIncrement = 0b1000` is chosen so that adding or
subtracting one from the counter (`c.refcount +% rcIncrement`) never touches
the low three bits — those change only explicitly, via `setColor`/`ZctFlag`.
The `rcBlack/rcGray/rcWhite/rcPurple` colors are declared here but only
actively used by the debug/alternative mark&sweep path (see the `when false`
blocks) — the production path only checks whether `ZctFlag` is set or not.

**List of constants:**
- `rcIncrement` — the amount by which one `incref`/`decref` changes
  `refcount`;
- `ZctFlag` — the bit meaning "this cell is currently in the ZCT";
- `rcShift` — how many bits to shift `refcount` by to get the actual
  reference count (`internRefcount`);
- `colorMask` — the mask for extracting/setting the three-bit color.

---

## II. Reference Counting

### `incRef` / `decRef`

```nim
proc incRef(c: PCell) {.inline.}
proc decRef(c: PCell) {.inline.}
```

**What it does.** `incRef` increases a cell's refcount by `rcIncrement`.
`decRef` decreases it and, if the result drops below `rcIncrement` (i.e. the
logical count reached zero), puts the cell into the ZCT via `rtlAddZCT` —
the actual memory release doesn't happen at this point; it's deferred until
`collectZCT`.

**Implementation notes.** This deferral is exactly what gives the algorithm
its name — *Deferred* Reference Counting: `decRef` is an O(1) operation with
no recursive side effects, unlike "naive" refcounting where `decRef` would
immediately recurse into decref-ing all child objects. The heavy lifting is
moved into `collectZCT`, where it's easier to batch and (under
`withRealTime`) interrupt on a timeout.

**List of parameters:**
- `c: PCell` — a pointer to the cell header (not to the user data — use
  `usrToCell`/`cellToUsr` to convert between the two).

**Trace** (what happens on `a = b`, where `a`, `b: ref T`):

```nim
# Roughly the pseudo-C the compiler generates for "a = b":
# asgnRef(addr a, cast[pointer](b))
#
# Inside asgnRef:
#   1. incRef(usrToCell(b))   # b now has one more reference
#   2. decRef(usrToCell(a))   # a's old value loses one reference
#      # if refcount(old_a) reaches 0 after this —
#      # the cell is put into the ZCT and will be freed later, in collectZCT
#   3. a = b                  # the pointer itself is reassigned
```

---

### `asgnRef`

```nim
proc asgnRef(dest: PPointer, src: pointer) {.compilerproc, inline.}
```

**What it does.** This is the very procedure that compiler-generated C code
calls on every assignment of a reference value (`ref`, `seq`, `string`) on
the heap. It updates the refcounts of both the old and new value and only
then performs the actual pointer assignment.

**Implementation notes.** The order of operations is deliberately fixed by a
comment in the source ("BUGFIX: first incRef then decRef!") — increment the
new value first, then decrement the old one. If the order were reversed and
`dest` happened to equal `src` (i.e. `a = a`), the decrement could
prematurely zero out the object's only reference before the increment
restored the count — and it would be wrongly pushed into the ZCT.

**List of parameters:**
- `dest: PPointer` — the address of the pointer field being written to
  (guaranteed not to be on the stack — checked by
  `gcAssert(not isOnStack(dest), ...)`);
- `src: pointer` — the new value (may be `nil`).

---

### `unsureAsgnRef`

```nim
proc unsureAsgnRef(dest: PPointer, src: pointer) {.compilerproc.}
```

**What it does.** A variant of `asgnRef` for cases where the compiler
*cannot statically determine* whether `dest` lives on the stack or on the
heap — this happens with `var` parameters (see the example in the file's
introductory comment: `setRef(r)` vs. `setRef(r.left)`). The counters are
only updated if, at runtime, `dest` turns out not to be on the stack.

**Implementation notes.** References on the stack aren't counted at all by
this collector (for performance — that's the main tradeoff the algorithm
states up front in the file header), so updating the refcount for a stack
address would be not just wasteful but wrong. The `isOnStack` check is cheap
on most systems with a continuous stack — it's just two pointer comparisons
against the stack bounds.

**List of parameters:**
- `dest: PPointer` — the destination address, for which it's unknown at
  compile time whether it's stack or heap;
- `src: pointer` — the new value.

---

### `nimGCref` / `nimGCunref`

```nim
proc nimGCref(p: pointer) {.compilerproc.}
proc nimGCunref(p: pointer) {.compilerproc.}
```

**What it does.** This is the runtime implementation behind the public
`GC_ref`/`GC_unref` procedures in `system`: it adds the object to the
`additionalRoots` list ("external roots") and bumps its refcount so the
object won't be collected even without any ordinary references to it — for
example, when it's held only by a C array. `nimGCunref` removes the object
from `additionalRoots` and drops this artificial reference.

**Edge-case behavior.** `nimGCunref` linearly searches for the cell in
`additionalRoots` and, if found, replaces it with the last element of the
list (the classic "swap-with-last" trick for removing from an unordered
array in O(1) without shifting the tail).

**List of parameters:**
- `p: pointer` — a pointer to the user data (not to the cell header — the
  procedure calls `usrToCell(p)` internally).

---

## III. Object Allocation

### `rawNewObj` / `newObj` / `newObjNoInit`

```nim
proc rawNewObj(typ: PNimType, size: int, gch: var GcHeap): pointer
proc newObj(typ: PNimType, size: int): pointer {.compilerRtl, noinline, raises: [].}
proc newObjNoInit(typ: PNimType, size: int): pointer {.compilerRtl, raises: [].}
```

**What it does.** `rawNewObj` is the heart of the allocator: before
allocating, it checks the thresholds and, if needed, runs a collection
(`collectCT`), then actually allocates memory for the cell header plus the
user data, zeroes the `refcount` field, sets the `ZctFlag` (the count is
zero, but the cell is still alive until the first assignment), and
immediately puts the cell into the ZCT. `newObj` additionally zeroes the
user data (`zeroMem`); `newObjNoInit` is a faster variant that skips
zeroing, for places where initialization will happen right afterward
anyway.

**Implementation notes.** Note the order: `collectCT(gch)` is called
*before* allocation, not after — meaning the decision to collect is made
based on the heap's state as of the moment of the request, not after the
fact. A newly created object immediately goes into the ZCT with
`refcount = 0` — this lets an "not yet assigned to anything" object be
treated uniformly with an object whose count genuinely dropped to zero: both
paths to release go through the same `collectZCT`.

**List of parameters:**
- `typ: PNimType` — the RTTI type descriptor (needed for the collector to
  later walk child references);
- `size: int` — the size of the user data in bytes (excluding the cell
  header — `sizeof(Cell)` is added internally);
- `gch: var GcHeap` — the heap to allocate in (in the single-threaded case,
  almost always the global `gch`).

**Trace** (creating a `ref T` in user code):

```nim
# User code:
#   var r = new(T)
#
# The compiler generates roughly:
#   r = newObj(T_typeInfo, sizeof(T))
#
# Inside newObj -> rawNewObj:
#   1. collectCT(gch)                 # a background collection may run here
#   2. rawAlloc(...)                  # the actual memory allocation
#   3. res.refcount = ZctFlag         # count = 0, but flagged "in the ZCT"
#   4. addNewObjToZCT(res, gch)       # queued for its "first collection"
#   5. result = cellToUsr(res)        # the user gets the address AFTER the header
```

---

### `newObjRC1` / `newSeqRC1`

```nim
proc newObjRC1(typ: PNimType, size: int): pointer {.compilerRtl, noinline, raises: [].}
proc newSeqRC1(typ: PNimType, len: int): pointer {.compilerRtl, raises: [].}
```

**What it does.** An optimized allocation path for the case where the
compiler statically knows the newly created object will be immediately
assigned to exactly one reference (the typical `var x = ...` case). In that
case `refcount` can be set to `1` right away, and the object doesn't need to
go through the ZCT at all — saving one "put into ZCT → immediately removed
on first assignment" round trip.

**Implementation notes.** This is an example of an optimization driven by
the compiler's data-flow analysis: the ordinary `newObj` pessimistically
assumes a new object might sit around unassigned for a while (say, returned
from a procedure and not stored by the caller), so it must go through the
ZCT. `newObjRC1` is only used where that analysis has proven exactly one
assignment will follow immediately.

**List of parameters:**
- `typ: PNimType` — the type descriptor;
- `size` / `len: int` — the size in bytes (`newObjRC1`) or the sequence
  length (`newSeqRC1`).

---

### `growObj`

```nim
proc growObj(old: pointer, newsize: int, gch: var GcHeap): pointer
```

**What it does.** The implementation of `seq`/`string` growth when adding
elements beyond current capacity: allocates a new, larger block, copies the
old data into it, and zeroes the tail. The old cell isn't freed immediately
— its length is simply zeroed out (`cast[PGenericSeq](old).len = 0`) so the
collector doesn't attempt to walk the "stolen" child references twice.

**Implementation notes.** Copying `oldsize + sizeof(Cell)` in a single
`copyMem` captures both the cell header (type, count, etc.) and all the
data at once — this is more robust than copying fields individually, because
it automatically carries over any future header fields without needing to
touch this code. Zeroing the old object's length is a guard against double
walking: if the GC managed to scan the old cell before it's physically
freed, it would see the same child pointers as in the new one and count
them twice.

**List of parameters:**
- `old: pointer` — a pointer to the user data of the old sequence/string;
- `newsize: int` — the newly required size, in bytes;
- `gch: var GcHeap` — the heap being used.

---

## IV. Walking the Object Graph

### `forAllChildren` / `forAllChildrenAux`

```nim
proc forAllChildren(cell: PCell, op: WalkOp)
proc forAllChildrenAux(dest: pointer, mt: PNimType, op: WalkOp)
```

**What it does.** For a single cell, `forAllChildren` finds all its child
references (`ref` fields, `seq` elements, fields of objects and tuples) and
runs operation `op` on each one (see `WalkOp`). If the type has its own
marker (`cell.typ.marker` — a compiler-generated walking function specific
to that type), it's used; otherwise a generic RTTI-driven walk
(`forAllChildrenAux`/`forAllSlotsAux`) recurses down the field tree: objects
and tuples are walked by their list of slots, arrays element by element,
and `ref`/`seq`/`string` are treated as leaves of the tree, where the
operation is applied directly.

**Implementation notes.** A type flagged `ntfNoRefs` in `mt.flags` (no
GC pointers inside it at all) is skipped entirely without descending into
its fields — an important optimization: the walk doesn't waste time on
structures known in advance to contain no references.

**List of parameters:**
- `cell: PCell` / `dest: pointer` — the cell or raw data address being
  walked;
- `mt: PNimType` — the RTTI type descriptor for that data;
- `op: WalkOp` — what to do with each discovered child reference.

---

### `markS` / `markGlobals`

```nim
proc markS(gch: var GcHeap, c: PCell)
proc markGlobals(gch: var GcHeap) {.raises: [].}
```

**What it does.** `markS` marks cell `c` as reachable (adds it to
`gch.marked`) and iteratively (via `tempStack`, not recursion) walks and
marks everything reachable from it. `markGlobals` is the entry point for
marking "external" roots: the program's global variables
(`globalMarkers`/`threadLocalMarkers` — lists of marker functions the
compiler generates one per module) and objects added manually via `GC_ref`
(`additionalRoots`).

**Implementation notes.** Using an explicit stack (`tempStack`) instead of
recursive calls to `forAllChildren` is a deliberate choice against
overflowing the system stack on deep data structures (a long linked list, a
deep tree). It's the classic "recursion elimination" technique: instead of
`markChild(x)` inside the function body, you do `add(tempStack, x)`, and an
outer `while` processes what's been accumulated.

**List of parameters:**
- `gch: var GcHeap` — the heap holding the `marked` set and the helper
  `tempStack`;
- `c: PCell` — the root to start marking from (for `markS`).

---

### `gcMark` / `markStackAndRegisters`

```nim
proc gcMark(gch: var GcHeap, p: pointer) {.inline.}
proc markStackAndRegisters(gch: var GcHeap) {.noinline, cdecl.}
```

**What it does.** `markStackAndRegisters` scans the entire current call
stack (and the registers saved on function entry) looking for values that
look like pointers into the GC heap, calling `gcMark` for each candidate
address. `gcMark` checks whether address `p` genuinely points inside a
GC-managed object (`interiorAllocatedPtr`), and if so, temporarily increfs
it and remembers it in `decStack`, so the count can be rolled back after the
stack scan (see `unmarkStackAndRegisters`).

**Edge-case behavior.** The stack is scanned conservatively: any machine
word that looks like an address inside the GC heap is treated as a
potential reference, even if it's actually, say, an integer that just
happens to look like a pointer — hence the term "conservative collector" as
applied to stack scanning. False positives are safe (an object simply
doesn't get collected this cycle a bit longer than strictly necessary),
false negatives are not, so the algorithm chooses to err on the side of
caution.

**List of parameters:**
- `gch: var GcHeap` — the heap, with its `stack` and `decStack` fields;
- `p: pointer` — a candidate address found while scanning the stack (for
  `gcMark`).

---

## V. The Collection Cycle

### `collectZCT`

```nim
proc collectZCT(gch: var GcHeap): bool
```

**What it does.** Processes the zero count table (ZCT): for every cell whose
`refcount` genuinely reached zero, it calls the finalizer (if any),
recursively decrefs all of its children (via
`forAllChildren(c, waZctDecRef)` — which may push new cells into the same
ZCT while this very function is still running), and physically frees the
memory. Returns `false` only if the collection was cut short by a timeout in
`withRealTime` mode (see "soft real-time" below).

**Implementation notes.** A cell isn't taken from the front or back of the
list, but always from index `0`, with the last element then moved into its
place (`gch.zct.d[0] = gch.zct.d[L-1]`) — again the "swap-with-last" trick
for O(1) removal from an array without shifting the remaining elements. A
comment in the source explicitly warns that since freeing one cell can add
its children to the ZCT, this is effectively "deep" freeing, and the whole
point of batching (`workPackage = 100`) is to keep that chain of releases
from blocking the thread for too long in time-limited mode.

**List of parameters:**
- `gch: var GcHeap` — the heap, with its `zct` field and (under
  `withRealTime`) `maxPause`.

---

### `collectCTBody` / `collectCT`

```nim
proc collectCTBody(gch: var GcHeap) {.raises: [].}
proc collectCT(gch: var GcHeap)
```

**What it does.** `collectCT` is the "gatekeeper": it checks whether the
thresholds are exceeded (`zctThreshold` for the ZCT or `cycleThreshold` for
occupied memory) and whether `recGcLock` is held, and only then calls
`collectCTBody`. `collectCTBody` is one full collection phase: it scans the
stack, collects the ZCT, and, if occupied memory is still above
`cycleThreshold` afterward, additionally runs cycle search
(`collectCycles`).

**Implementation notes.** The cycle-collection threshold is recalculated
adaptively after every run: `cycleThreshold = max(InitialCycleThreshold,
occupiedMem * CycleIncrease)` — i.e. the next cycle search won't happen
until occupied memory grows by a factor of `CycleIncrease` (2) from its
current level. This balances "searching for cycles too often" (expensive)
against "never searching" (memory leaks on cyclic structures).

**List of parameters:**
- `gch: var GcHeap` — the entire heap (thresholds, ZCT, statistics).

---

### `collectCycles` / `sweep`

```nim
proc collectCycles(gch: var GcHeap) {.raises: [].}
proc sweep(gch: var GcHeap)
```

**What it does.** `collectCycles` is a full pass of cyclic garbage
detection: it first flushes the ZCT (so that the ZCT "color" doesn't get
confused with the marking pass), resets the `marked` set, marks everything
reachable from the stack (`decStack`) and from globals (`markGlobals`) as
reachable, and then calls `sweep`, which walks **every object in the
managed memory region** and frees the ones that didn't end up in `marked`.

**Edge-case behavior.** An object that's unmarked but still carries the
`ZctFlag` (which can happen if it was created while a finalizer was
running) is not physically freed — only the flag is cleared, so it isn't
lost in the next cycle.

**List of parameters:**
- `gch: var GcHeap` — the heap (uses `zct`, `marked`, `decStack`, `region`).

---

## VI. Public GC Control API

Below are the procedures you can *actually* call from an ordinary Nim
program (available directly, with no explicit `import`, as part of the
`system` module). The examples in this section are real, runnable code.

### `GC_disable` / `GC_enable`

```nim
proc GC_disable()
proc GC_enable()
```

**What it does.** `GC_disable` temporarily suspends the collector: it
increments the nesting counter `recGcLock`, which makes `collectCT` stop
triggering a collection even if the thresholds are exceeded. `GC_enable`
decrements that counter back; calls can be nested (two `GC_disable` calls
require two `GC_enable` calls).

**List of parameters:** none.

**Example:**

```nim
proc criticalSection() =
  GC_disable()
  var buf = newSeq[byte](1024)
  # no GC pause can happen here, guaranteed
  for i in 0..<len(buf):
    buf[i] = byte(i mod 256)
  GC_enable()
  echo "done, buffer length: ", len(buf) # prints "done, buffer length: 1024"
```

---

### `GC_fullCollect`

```nim
proc GC_fullCollect()
```

**What it does.** Forces a full collection cycle, including cycle search,
regardless of current thresholds — it temporarily zeroes `cycleThreshold`,
calls `collectCT`, then restores the previous threshold.

**List of parameters:** none.

**Example:**

```nim
type
  Node = ref object
    next: Node

var a = Node()
var b = Node()
a.next = b
b.next = a       # an artificial reference cycle
a = nil
b = nil          # plain refcounting alone won't free this pair by itself
GC_fullCollect()  # forcibly finds and collects the cycle
echo "cycle handled"  # prints "cycle handled"
```

---

### `GC_enableMarkAndSweep` / `GC_disableMarkAndSweep`

```nim
proc GC_enableMarkAndSweep()
proc GC_disableMarkAndSweep()
```

**What it does.** Turns just the cycle-search phase (mark&sweep) on or off,
without touching ordinary reference counting. `GC_disableMarkAndSweep` sets
`cycleThreshold` to the highest possible value, so the "time to search for
cycles" condition practically never triggers.

**Implementation notes.** This is not a full GC stop (like `GC_disable`) —
ordinary, acyclic objects keep getting freed through the ZCT as usual; only
the more expensive check for cyclic structures is turned off.

**List of parameters:** none.

**Example:**

```nim
GC_disableMarkAndSweep()
# in this section of the program, reference cycles will NOT be found and
# collected, but ordinary, acyclic objects keep being collected as usual
GC_enableMarkAndSweep()
echo "cycle search re-enabled" # prints "cycle search re-enabled"
```

---

### `GC_getStatistics`

```nim
proc GC_getStatistics(): string
```

**What it does.** Returns a multi-line summary of the collector's state:
total and occupied memory, number of stack scans, the peak number of cells
found on the stack, the number of full cycle collections, the peak
threshold used, ZCT capacity, and, if `withRealTime` is enabled, the longest
recorded pause.

**List of parameters:** none.

**Example:**

```nim
echo GC_getStatistics()
# prints roughly:
# [GC] total memory: 1048576
# [GC] occupied memory: 32768
# [GC] stack scans: 12
# [GC] stack cells: 4
# [GC] cycle collections: 1
# [GC] max threshold: 4194304
# [GC] zct capacity: 512
# [GC] max cycle table size: 0
# [GC] max pause time [ms]: 0
```

---

### `GC_step` / `GC_setMaxPause`

```nim
proc GC_setMaxPause(MaxPauseInUs: int)
proc GC_step(us: int, strongAdvice = false, stackSize = -1) {.noinline.}
```

**What it does.** Both procedures are only available when the runtime is
built with `-d:useRealtimeGC`. `GC_setMaxPause` sets a global limit on the
duration of a single collection pause, in microseconds — if exceeded,
`collectZCT` cuts itself short early and resumes from the same point on the
next step. `GC_step` explicitly performs one collection "step" lasting no
more than `us` microseconds; `strongAdvice = true` forces the step to run
even if the normal thresholds haven't been reached yet — i.e. this is manual
control over *when* time is spent on GC, instead of waiting for it to
trigger automatically in the middle of a frame.

**List of parameters:**
- `MaxPauseInUs: int` — the maximum duration of a single pause, in
  microseconds (for `GC_setMaxPause`);
- `us: int` — the time budget for this particular step, in microseconds;
- `strongAdvice: bool` — run the step even if the thresholds haven't been
  reached yet;
- `stackSize: int` — if ≥ 0, temporarily limits the portion of the stack
  scanned to this size (a typical scenario: a game loop calling `GC_step`
  every frame between draws).

**Example** (a typical game/interactive loop):

```nim
GC_setMaxPause(1_000) # no more than 1 ms per collection pause
while running:
  updateGame()
  renderFrame()
  GC_step(1_000, strongAdvice = true) # give the collector a small time
                                       # budget every frame instead of one
                                       # big pause somewhere in the middle
```

---

## VII. Practical Recipes

### 1. Diagnosing Leaks via GC Statistics

```nim
proc reportBefore() =
  echo "before the operation:"
  echo GC_getStatistics()

proc heavyOperation() =
  var data = newSeq[seq[int]](1000)
  for i in 0..<len(data):
    data[i] = newSeq[int](1000)
  discard data

reportBefore()
heavyOperation()
GC_fullCollect() # run a full cycle so the statistics are honest
echo "after the operation and a full collection:"
echo GC_getStatistics()
```

Comparing `occupied memory` before and after `GC_fullCollect()` at two
similar points in the program is the simplest way to spot memory not being
returned to the collector (a telltale sign of reference cycles that
somehow aren't found, or references being accidentally held by global
state).

### 2. Disabling the GC During a Critical Section

```nim
proc renderAudioBuffer(samples: int): seq[float32] =
  GC_disable()          # no GC pause should interrupt this computation
  result = newSeq[float32](samples)
  for i in 0..<samples:
    result[i] = float32(i) / float32(samples)
  GC_enable()
```

This trick only works for short, predictable-memory-footprint sections —
while the GC is disabled, the ZCT keeps growing, and that's worth
controlling explicitly (for example, don't allocate an unbounded number of
new objects inside such a section).

### 3. Soft Real-Time GC for Interactive Programs

```nim
# compiled with -d:useRealtimeGC
GC_setMaxPause(500) # a hard pause budget: 0.5 ms

proc gameLoop() =
  while true:
    handleInput()
    updatePhysics()
    draw()
    GC_step(500, strongAdvice = false, stackSize = 128 * 1024)
```

Here `strongAdvice = false` means "only run a step if the
`zctThreshold`/`cycleThreshold` thresholds have already been exceeded" —
i.e. `GC_step` is called every frame, but real work only happens when it's
genuinely due, bounded by a 500-microsecond budget.

---

## VIII. Quick Reference Table

| Task                                                    | What to use                            |
|-----------------------------------------------------------|------------------------------------------|
| Assign a reference when it's known where it lives (heap/stack) | `asgnRef` (generated by the compiler) |
| Assign through a `var` parameter of unknown nature          | `unsureAsgnRef`                        |
| Artificially keep an object alive (external root)            | `GC_ref` / `GC_unref` (`nimGCref`/`nimGCunref`) |
| Allocate a new `ref`/`seq`/`string`                          | `newObj` / `newSeq` (generated by the compiler) |
| Grow a `seq`/`string` beyond its capacity                    | `growObj`                               |
| Suspend collection for a critical section                    | `GC_disable` / `GC_enable`              |
| Force-collect reference cycles immediately                   | `GC_fullCollect`                        |
| Disable only cycle search, keep ordinary refcounting          | `GC_disableMarkAndSweep`               |
| Inspect the collector's running statistics                   | `GC_getStatistics`                      |
| Bound the length of a single GC pause (real-time collection)  | `GC_setMaxPause` + `GC_step`            |

---

## IX. Summary: What to Pick

- Need to temporarily guarantee no GC pauses in a short section →
  `GC_disable()` / `GC_enable()`.
- Suspect a leak from reference cycles → `GC_fullCollect()`, then compare
  `GC_getStatistics()` before/after.
- Writing a game or other interactive loop with a hard per-frame time
  budget → build the runtime with `-d:useRealtimeGC`, set
  `GC_setMaxPause`, call `GC_step` every frame.
- Want to keep ordinary memory cleanup but skip the cost of cycle search →
  `GC_disableMarkAndSweep()` / `GC_enableMarkAndSweep()`.
- Want to understand what happens "under the hood" on `a = b`, `new(T)`, or
  a growing `seq` → sections II–V of this reference (these are internal
  mechanisms, not a public API — they cannot be called directly from user
  code).
