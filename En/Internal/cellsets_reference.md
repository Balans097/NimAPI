# `cellsets.nim` — Module Reference

> **Part of Nim's Runtime Library (Garbage Collector internals)**  
> Internal GC module — never imported directly by user code.

---

## Overview

`cellsets.nim` implements `CellSet`: the **central data structure of Nim's garbage collector**. A `CellSet` is a set of memory pointers (called *cells*) that the GC uses to track all live heap allocations. Every time the GC needs to know whether a pointer belongs to the managed heap — or to enumerate all managed objects — it queries a `CellSet`.

The defining requirement is extreme performance. A GC that spends too much time tracking allocations is useless. This module therefore implements a bespoke hybrid structure rather than using a general-purpose container.

### The hybrid structure: hash table of bitsets

The design is documented in the module's opening comment and worth understanding before reading individual procedures:

```
Hash table
 ┌──────────────────────────────┐
 │ [hash slot 0] → PageDesc     │   PageDesc
 │ [hash slot 1] → nil          │  ┌──────────────────────────┐
 │ [hash slot 2] → PageDesc ────┼─►│ key   = page base address │
 │ ...                          │  │ bits  = [64 integers]     │
 └──────────────────────────────┘  │ next  = →PageDesc (list)  │
                                    └──────────────────────────┘
```

Memory is divided into fixed-size **pages** (defined by `PageSize` / `PageShift` from `bitmasks.nim`). For each page that contains at least one managed object, a `PageDesc` is created. The `PageDesc` contains a **bit vector**: one bit per possible cell position within the page. If bit *k* is set, then the cell at position *k* is a live managed object.

This design gives three valuable properties simultaneously:
- **O(1) insert, delete, and lookup** — a pointer is located in three steps: hash lookup for the page, then two bit operations.
- **Compact storage** — a page with 512 cells needs only 512 bits (64 bytes) of bit storage, not 512 pointers.
- **Fast traversal** — iterating all live objects is a tight loop over words in each bit vector, skipping zero words instantly.

### Compile-time variants

The module adapts to the active GC:

- **ARC/ORC/AtomicArc**: the `PCell` type alias is defined as `Cell` and `bitmasks.nim` is included for page constants.
- **Classic GC (refc, etc.)**: `cellseqs_v1` is included instead, which provides additional sequence-based cell tracking used by the older GC.

---

## Type Definitions

### `PageDesc`

```nim
type
  PageDesc {.final, pure.} = object
    next: PPageDesc    # intrusive linked list — all descriptors form a chain
    key:  uint         # page base address (cell address >> PageShift)
    bits: array[BitIndex, int]  # one bit per cell slot in this page
```

The core data record. Each `PageDesc` represents one OS memory page and stores which cell addresses within that page are currently live. The `bits` array is the actual bitset — each `int`-sized word contains `sizeof(int)*8` bits, one per potential cell position.

The `next` pointer is used for the **intrusive linked list** (`head` in `CellSet`), which allows the `elements` iterator to traverse all page descriptors without needing to scan the entire hash table.

`BitIndex` is the range `[0..UnitsPerPage-1]`, constraining array access to valid indices.

---

### `CellSet`

```nim
type
  CellSet {.final, pure.} = object
    counter: int        # number of PageDesc entries currently in the hash table
    max:     int        # hash table capacity minus one (always 2^n - 1)
    head:    PPageDesc  # head of the intrusive linked list of all PageDescs
    data:    PPageDescArray  # the hash table itself (open addressing)
```

The top-level container. Its hash table (`data`) maps page addresses to `PageDesc` records. The `head` linked list provides fast ordered traversal of all descriptors without scanning empty slots.

The capacity is always a power of two (`max = 2^n - 1`), which makes modular indexing a single bitwise AND operation.

---

## Lifecycle Procedures

### `init`

```nim
proc init(s: var CellSet)
```

Initialises a `CellSet` for use. Allocates the initial hash table of `InitCellSetSize` (1024) slots, all zeroed. Sets `counter` to zero and `head` to nil.

Must be called before any other operation on a `CellSet`. The initial table size of 1024 is a power of two, chosen to avoid early rehashing for typical programs — most GC-managed programs have well under 1024 live pages at any one time.

```nim
var cs: CellSet
init(cs)
# cs is now ready to use
```

---

### `deinit`

```nim
proc deinit(s: var CellSet)
```

Releases all memory owned by the `CellSet`. Walks the intrusive linked list from `head` and calls `dealloc` on every `PageDesc`, then deallocates the hash table array `data`. Resets all fields to safe zero/nil values.

This is always called when the GC shuts down or rebuilds its tracking structures. Page descriptors are *never* individually removed from the table during normal operation (see the architecture note above); `deinit` is the only point at which they are freed.

```nim
deinit(cs)
# All memory returned to the allocator; cs must not be used again
# without another call to init.
```

---

## Internal Hash Table Procedures

These procedures implement the raw hash table mechanics. They are not part of the public-facing interface of `CellSet`; they are the engine underneath.

---

### `nextTry`

```nim
proc nextTry(h, maxHash: int): int {.inline.}
```

Computes the **next probe position** for open-address hash collision resolution. Uses the formula `(5*h + 1) and maxHash`.

This is a well-known probe sequence. When `maxHash` is `2^n - 1`, starting from any initial `h` and repeatedly applying this formula visits **every slot in the table exactly once** before returning to `h`. This is the full-cycle property that guarantees insertion always terminates (as long as at least one slot is empty) and lookup terminates (as soon as a nil slot is found).

The constant `5` is chosen because it is coprime to any power of two, which is the mathematical requirement for full-cycle coverage.

```
# Visualisation for a table of 8 slots (maxHash = 7):
# Start at h=3: 3 → 0 → 1 → 6 → 7 → 4 → 5 → 2 → 3 (full cycle)
```

---

### `cellSetGet`

```nim
proc cellSetGet(t: CellSet, key: uint): PPageDesc
```

**Lookup** — finds the `PageDesc` for page address `key` in the hash table. Returns the descriptor if found, `nil` if the page is not tracked.

The probe loop terminates on `nil` (the slot is empty, so the key was never inserted). It never visits the same slot twice thanks to `nextTry`'s full-cycle property.

This is called by `contains`, `excl`, and `containsOrIncl` — any operation that needs to find an existing page without creating a new one.

```nim
# Used internally, e.g.:
let desc = cellSetGet(cs, pageAddress)
if desc != nil:
  # this page is known — check individual bits
```

---

### `cellSetRawInsert`

```nim
proc cellSetRawInsert(t: CellSet, data: PPageDescArray, desc: PPageDesc)
```

**Low-level insert** — places an existing `PageDesc` (with `desc.key` already set) into a given hash table array `data`. Does not modify `t.counter` or any list pointers. Uses `sysAssert` to verify the invariant that the descriptor is not already present and that the chosen slot is indeed empty.

Used exclusively by `cellSetEnlarge` during table rehashing, where descriptors from the old table need to be reinserted into the freshly allocated, larger table.

---

### `cellSetEnlarge`

```nim
proc cellSetEnlarge(t: var CellSet)
```

**Rehash** — doubles the capacity of the hash table when it becomes too full. Allocates a new table of twice the size, reinserts all existing `PageDesc` records using `cellSetRawInsert`, then frees the old table.

The resize threshold is checked in `cellSetPut`: the table is enlarged when either:
- The number of entries exceeds 2/3 of capacity (load factor > 0.67), or
- Fewer than 4 slots remain free.

This keeps the hash table sparse enough that probe chains stay short, maintaining O(1) expected lookup time.

---

### `cellSetPut`

```nim
proc cellSetPut(t: var CellSet, key: uint): PPageDesc
```

**Get-or-create** — the main write path. Finds the `PageDesc` for page `key` and returns it. If no descriptor exists for that page, allocates a new zeroed `PageDesc`, links it into the `head` list, inserts it into the hash table, increments `counter`, and returns it.

This is the only place where new `PageDesc` objects are created. All five steps happen atomically from the caller's perspective:
1. Check if key already exists → return it if so.
2. Check if table needs enlarging → enlarge if needed.
3. Allocate new `PageDesc` (zeroed via `alloc0`).
4. Prepend to the `head` intrusive linked list.
5. Insert into the hash table.

---

## Set Operation Procedures

These are the meaningful, higher-level interface of `CellSet`. They perform the address decomposition — splitting a pointer into its page number and bit index — and delegate to the hash table layer.

The address decomposition follows this scheme for any cell pointer `cell`:

```
page_key   = cast[uint](cell) >> PageShift        → which PageDesc to look up
bit_offset = (cast[uint](cell) mod PageSize) div MemAlign  → which bit within the page
word_index = bit_offset >> IntShift               → which int in bits[]
bit_index  = bit_offset and IntMask               → which bit within that int
```

---

### `contains`

```nim
proc contains(s: CellSet, cell: PCell): bool
```

Returns `true` if `cell` is currently a member of the set. Queries the hash table for the page descriptor, then tests the specific bit corresponding to `cell`'s offset within that page. Returns `false` immediately if no descriptor exists for the page (i.e. no cell from that page has ever been inserted).

This is a pure query — it never modifies the set.

```nim
if contains(liveSet, somePointer):
  # this pointer is tracked by the GC
```

---

### `incl`

```nim
proc incl(s: var CellSet, cell: PCell)
```

Adds `cell` to the set. Uses `cellSetPut` to find or create the page descriptor, then **sets** the appropriate bit. If the page descriptor already existed, only the bit changes. If it is new, the entire descriptor is freshly allocated and zeroed, then the single bit is set.

The name `incl` mirrors Nim's standard set `incl` operation.

```nim
incl(rootSet, newlyAllocatedPointer)
# The pointer is now tracked
```

---

### `excl`

```nim
proc excl(s: var CellSet, cell: PCell)
```

Removes `cell` from the set by **clearing** its bit. If no page descriptor exists for the cell's page, the call is a safe no-op — no assertion, no crash.

Importantly, `excl` **never deletes page descriptors** even if all bits in a page's descriptor become zero. The comment in the architecture section explains why: the GC rebuilds its structures periodically, so the overhead of removing empty descriptors is not worth the complexity.

```nim
excl(rootSet, freedPointer)
# The bit is cleared; the PageDesc remains
```

---

### `containsOrIncl`

```nim
proc containsOrIncl(s: var CellSet, cell: PCell): bool
```

**Atomic test-and-set.** Returns `true` if `cell` was already in the set; returns `false` if it was not (and adds it in that case). Combines a membership test and a conditional insert in a single, efficient operation.

This is more efficient than calling `contains` followed by `incl` separately, because when the page descriptor already exists, the bit is both tested and set in a single pass through `cellSetGet` — no second hash lookup is needed.

This operation is central to the GC's mark phase: during marking, each reachable pointer must be visited exactly once. `containsOrIncl` provides the precise "mark if not already marked, report if already marked" semantic that makes this efficient.

```nim
if not containsOrIncl(markedSet, reachablePtr):
  # First time we've seen this pointer — push its children for scanning
  pushChildren(reachablePtr)
# If it returned true, we've already processed this pointer — skip it
```

---

## Iterators

### `elements`

```nim
iterator elements(t: CellSet): PCell {.inline.}
```

Yields every cell pointer currently in the set. Traverses all page descriptors via the `head` linked list, then for each descriptor scans all words in the `bits` array. For each non-zero word, scans all set bits and reconstructs the original cell address.

The address reconstruction is the inverse of the decomposition:

```
cell_address = (page_key << PageShift) | (word_index << IntShift + bit_index) * MemAlign
```

**Critical constraint:** *modifying the set during iteration is undefined behaviour.* The iterator takes a snapshot of each `bits` word (`var w = r.bits[i]`) rather than reading live memory, which is both an optimisation and a safety measure — but it does not protect against structural changes to the page descriptor list itself.

This iterator is used by the GC to enumerate all live roots and all reachable objects during a collection cycle.

```nim
for cell in elements(liveSet):
  # process every tracked pointer exactly once
  markAndScan(cell)
```

---

### `elementsExcept`

```nim
iterator elementsExcept(t, s: CellSet): PCell {.inline.}
```

Yields every cell pointer that is in set `t` but **not** in set `s`. This is a set-difference iterator: `t \ s`.

The implementation is efficient: it does not build a temporary difference set. For each page descriptor in `t`, it looks up the corresponding descriptor in `s`. If found, the bits of `s`'s descriptor are subtracted from `t`'s using bitwise AND-NOT (`w and not ss.bits[i]`), producing a word with only the "in `t` but not in `s`" bits set. If `s` has no descriptor for that page, all bits of `t`'s descriptor are yielded.

This is used by the GC to find objects that were newly allocated since the last GC cycle — effectively computing `allocatedThisCycle \ alreadyTracked`.

```nim
for newCell in elementsExcept(currentAllocs, previousAllocs):
  # This pointer is new — add it to the root set
  incl(rootSet, newCell)
```

---

## Dead Code: `CellSetIter` and `next`

```nim
when false:
  type CellSetIter = object ...
  proc next(it: var CellSetIter): PCell = ...
  proc init(it: var CellSetIter; t: CellSet): PCell = ...
```

This block is guarded by `when false` and is never compiled. It represents an **explicit-state iterator** — an alternative to the `iterator` syntax that allows the iteration state to be stored in an object and advanced one step at a time, which is useful in contexts where a `for` loop cannot be used (e.g. coroutines or manual interleaving of two iterations).

It was likely preserved as reference for future use, possibly for the NIR backend or a future context where the `iterator` form cannot be used.

---

## Internal Constant: `InitCellSetSize`

```nim
const InitCellSetSize = 1024
```

The initial number of slots in the hash table array. Must be a power of two (comment enforces this). Chosen large enough to accommodate typical programs without an early rehash, but small enough to avoid wasteful allocation for short-lived programs.

---

## Summary Table

| Symbol | Kind | Role |
|---|---|---|
| `PageDesc` | type | Single page's bit vector + hash chain link |
| `CellSet` | type | The complete hybrid set (hash table + list) |
| `init` | proc | Allocate and zero-initialise a CellSet |
| `deinit` | proc | Free all memory owned by a CellSet |
| `nextTry` | proc (internal) | Open-addressing probe sequence (full-cycle 5h+1) |
| `cellSetGet` | proc (internal) | Hash table lookup by page key |
| `cellSetRawInsert` | proc (internal) | Insert descriptor into a table array (used during rehash) |
| `cellSetEnlarge` | proc (internal) | Double capacity and rehash all descriptors |
| `cellSetPut` | proc (internal) | Get-or-create page descriptor by key |
| `contains` | proc | Test membership of a cell pointer |
| `incl` | proc | Add a cell pointer to the set |
| `excl` | proc | Remove a cell pointer from the set |
| `containsOrIncl` | proc | Atomic test-and-set (core of GC mark phase) |
| `elements` | iterator | Yield all cell pointers in the set |
| `elementsExcept` | iterator | Yield cells in `t` but not in `s` (set difference) |
| `InitCellSetSize` | const | Initial hash table slot count (1024) |
