# hashcommon — Reference Manual

## Overview

`hashcommon.nim` is an **internal `include` file** shared by `std/tables`, `std/sets`, and `std/sharedtables`. It is not imported directly — it is textually pasted into the including module at compile time via Nim's `include` directive, making all its definitions available as if they were written inline.

The file implements the **low-level open-addressing hash table engine** that powers every hash-based container in Nim's standard library. Understanding it demystifies why those containers behave the way they do: how slots are probed, when a rehash is triggered, how zero hashes are handled, and how a single lookup function doubles as both a "find" and a "find insertion point" routine.

### Open addressing in one paragraph

An open-addressing hash table is a flat array of slots. To look up a key, you hash it to get a starting slot index, then walk forward (probe) until you either find the key or find an empty slot (meaning the key is absent). There are no pointers, no separate linked chains — everything lives in one contiguous array. The tradeoff is that the array must never be too full: if load factor gets too high, collisions become expensive. The growth and rehash logic in this file manages that constraint.

---

## Constants

### `growthFactor`

```nim
const growthFactor = 2
```

**What it is:** the multiplier applied to the internal data array's length every time a rehash is triggered. When the table decides it needs more room, the array is doubled in size.

**Why 2:** doubling is the canonical choice for open-addressing tables. It keeps the amortised cost of insertions O(1) — each element is moved O(log n) times over its lifetime. Smaller factors would reduce peak memory waste but increase rehash frequency; larger factors would rehash less often but consume more memory.

**Practical implication:** in the worst case a `Table` or `HashSet` uses roughly twice the memory needed to hold its current elements. The exact ratio depends on when `mustRehash` fires.

---

## Internal procedures and templates

### `isEmpty`

```nim
proc isEmpty(hcode: Hash): bool {.inline.}
```

**Purpose:** tests whether a slot in the data array is unoccupied. Returns `true` if and only if `hcode == 0`.

**The encoding contract:** the hash table encodes "empty slot" as a stored `hcode` value of exactly zero. Real keys can never have a hash of zero — the `genHashImpl` template remaps any zero hash to a non-zero constant before it reaches the table. This means a single integer field (`hcode`) carries two roles at once: the actual hash of the key (when non-zero) and the "empty" sentinel (when zero). No extra boolean "occupied" field is needed, saving memory.

---

### `isFilled`

```nim
proc isFilled(hcode: Hash): bool {.inline.}
```

**Purpose:** the logical complement of `isEmpty`. Returns `true` if the slot contains a real key (i.e. `hcode != 0`).

Used in probe loops: the loop walks slots while `isFilled` — stopping at the first empty slot means the key cannot be further along the probe sequence (the probe sequence is unbroken).

---

### `nextTry`

```nim
proc nextTry(h, maxHash: Hash): Hash {.inline.}
```

**Purpose:** advances the probe index by one step, wrapping around at the array boundary.

**Formula:** `(h + 1) and maxHash`

This is **linear probing** — the simplest possible collision resolution strategy. After a collision at slot `h`, the next candidate is slot `h + 1`. The `and maxHash` part replaces a modulo operation: because the array length is always a power of two, `maxHash = len - 1` has all lower bits set, and `and maxHash` is equivalent to `mod len` but faster.

**Why linear probing:** it is cache-friendly. Sequential slots are adjacent in memory, so probing is a sequential scan that the CPU prefetcher handles well. The downside is *primary clustering* — runs of occupied slots grow, increasing average probe length. For typical key distributions this is still faster in practice than chained or quadratic alternatives because of cache effects.

**Example walk-through:**

```
Array length = 8, maxHash = 7
Key hashes to slot 5, slot 5 is occupied.
nextTry(5, 7) = (5 + 1) & 7 = 6   → check slot 6
nextTry(6, 7) = (6 + 1) & 7 = 7   → check slot 7
nextTry(7, 7) = (7 + 1) & 7 = 0   → wrap around to slot 0
```

---

### `mustRehash`

```nim
proc mustRehash[T](t: T): bool {.inline.}
```

**Purpose:** decides whether the table is too full to accommodate another insertion without degrading performance. Returns `true` if a rehash should be triggered before the next element is inserted.

**Condition (either triggers rehash):**
- `t.dataLen < t.counter + t.counter div 2` — the array has fewer slots than 1.5× the current element count (load factor above ~67%)
- `t.dataLen - t.counter < 4` — fewer than 4 empty slots remain (guards against degenerate small tables)

**Why ~67% load factor:** open-addressing performance degrades as load factor approaches 1. At 67% the average number of probes for a lookup is still small (around 1.5–2 for random keys with linear probing). Above that the probe length grows superlinearly. The 4-slot minimum guard handles small tables where percentage-based thresholds would leave too little room.

**The `assert`:** `mustRehash` asserts that `dataLen > counter` before checking. This is a safety invariant — the table must never be completely full. If it were, a lookup for a missing key would loop forever (no empty slot to stop at).

**Practical example:**

```
Table with dataLen=8, counter=4:
  counter + counter div 2 = 4 + 2 = 6
  dataLen (8) < 6? No.
  dataLen - counter = 4. 4 < 4? No.
  → mustRehash = false, safe to insert.

Table with dataLen=8, counter=6:
  counter + counter div 2 = 6 + 3 = 9
  dataLen (8) < 9? Yes.
  → mustRehash = true, rehash before inserting.
```

---

### `slotsNeeded`

```nim
proc slotsNeeded(count: Natural): int {.inline.}
```

**Purpose:** given a desired number of elements `count`, returns the minimum array length (number of slots) that will keep the load factor below `mustRehash`'s threshold. Used when initialising a table with a known capacity hint.

**Formula:** `nextPowerOfTwo(count div 2 + count + 4)`

This simplifies to `nextPowerOfTwo(count * 3 / 2 + 4)`, which is exactly the inverse of the `mustRehash` threshold — it computes the smallest power-of-two array length `L` such that `mustRehash` would return `false` for `counter = count`.

**Why a power of two:** the `nextTry` probe step uses `and maxHash` (a bitmask) instead of modulo, which only works when the array length is a power of two.

**Example:**

```nim
slotsNeeded(10)
  = nextPowerOfTwo(10 div 2 + 10 + 4)
  = nextPowerOfTwo(5 + 10 + 4)
  = nextPowerOfTwo(19)
  = 32

# A table pre-sized for 10 elements gets 32 slots,
# giving comfortable headroom before the first rehash.
```

**Synchronisation note:** the comment in the source stresses that `slotsNeeded` and `mustRehash` must stay in sync. If one is changed, the other must be updated to match, otherwise a table initialised via `initTable(n)` might trigger an immediate rehash on the very first insertion.

---

### `genHashImpl` (template)

```nim
template genHashImpl(key, hc: typed)
```

**Purpose:** computes the hash of `key`, stores it in `hc`, and remaps the result if it happens to be zero. This is the single point where the "hcode 0 = empty" encoding is enforced.

**What it does:**
1. Calls `hash(key)` and stores the result in `hc`.
2. If `hc == 0` (almost never for real keys), replaces it with a fixed non-zero constant:
   - On 16-bit platforms: `31415`
   - On all others: `314159265` (a truncated π × 10⁸)

**Why remap zero:** the slot emptiness encoding uses `hcode == 0` as the sentinel. If a real key hashed to zero and was stored, the slot would look empty and the key would be invisible to all lookups. Remapping to a fixed constant is safe because the actual key is also stored in the slot — a false collision between the remapped hash and some other key's hash will be resolved by the subsequent key equality check.

**Why almost never zero:** a good hash function distributes evenly across all possible `Hash` values. The probability of any particular output (including zero) is approximately 1/2^64 on 64-bit systems. The branch is deliberately described as "almost never taken" to hint to the branch predictor that it should predict "not taken" every time, keeping the common path fast.

---

### `genHash` (template)

```nim
template genHash(key: typed): Hash
```

**Purpose:** a convenience wrapper around `genHashImpl` that returns the hash as an expression rather than storing it in a caller-supplied variable. Used when the hash result is needed inline without a pre-declared variable.

**Example (conceptual):**
```nim
let h = genHash(myKey)   # equivalent to:
# var h: Hash
# genHashImpl(myKey, h)
```

---

### `rawGetKnownHCImpl` (template)

```nim
template rawGetKnownHCImpl() {.dirty.}
```

**Purpose:** the core slot-lookup loop. Given a key and its already-computed hash `hc`, searches the data array and returns either the index of the found slot (≥ 0) or a negative value encoding the index where the key should be inserted if it were to be added.

**The `{.dirty.}` pragma:** dirty templates are expanded without hygienic renaming — variables inside the template (`h`) share the caller's namespace. This allows the template to be `include`-d into both `rawGet` and `rawGetKnownHC` without code duplication, while still mutating a shared local variable.

**Return value encoding:**
- `≥ 0` — the slot index where the key was found. The caller can read or update `t.data[result]`.
- `< 0` — the key is absent. The insertion index is `(-1 - result)`, i.e. `-1 - rawGetKnownHCImpl()`. This dual-purpose return eliminates a second pass through the probe sequence when inserting: the lookup has already found the best slot.

**Probe loop logic:**

```
h = hc and maxHash(t)          # starting slot (fast modulo)
while slot h is filled:
    if hcode matches AND key matches:
        return h               # found
    h = nextTry(h, maxHash(t)) # linear probe forward
return -1 - h                  # not found; h is first empty slot seen
```

**Two-stage comparison (hcode then key):** the loop first compares the stored `hcode` with the query `hc` before comparing keys. This short-circuits most probe steps for absent keys: hash collisions are rare, so the expensive key equality check (which may involve string comparison, struct comparison, etc.) is usually skipped. For present keys, the hcode comparison adds one cheap integer comparison per probe step — a worthwhile trade for non-trivial key types.

**Empty table guard:** if `t.dataLen == 0` the function returns `-1` immediately, avoiding a division-by-zero or infinite loop in the `and maxHash(t)` computation.

---

### `rawGetKnownHC`

```nim
proc rawGetKnownHC[X, A](t: X, key: A, hc: Hash): int {.inline.}
```

**Purpose:** a typed procedure wrapper around `rawGetKnownHCImpl`. Takes a pre-computed hash `hc` (already remapped from zero if necessary) and performs the probe loop. Used when the hash has already been computed by the caller and storing it in a variable for later reuse is beneficial — for example, during an insert that follows a failed lookup.

**Parameters:**
- `t` — the table or set (any type that has `data`, `dataLen`, and appropriate slot structure)
- `key` — the key to look up
- `hc` — the pre-computed, zero-remapped hash of `key`

**Returns:** same dual-purpose integer as `rawGetKnownHCImpl`.

---

### `rawGetImpl` (template)

```nim
template rawGetImpl() {.dirty.}
```

**Purpose:** the "compute hash then probe" composite step. Calls `genHashImpl` to compute and store the hash in the caller's `hc` variable, then immediately runs `rawGetKnownHCImpl`. Used by `rawGet` to combine both steps in one call when the hash hasn't been computed yet.

---

### `rawGet`

```nim
proc rawGet[X, A](t: X, key: A, hc: var Hash): int {.inline, outParamsAt: [3].}
```

**Purpose:** the primary lookup entry point for callers that do not yet have the hash. Computes the hash, stores it in `hc` (an output parameter), and returns the slot index (or negative insertion index).

**Parameters:**
- `t` — the table or set
- `key` — the key to look up
- `hc` — output: receives the computed, zero-remapped hash of `key`

**The `outParamsAt: [3]` pragma:** tells the compiler that argument 3 (`hc`) is a pure output — it is written before being read. This enables certain optimisations and suppresses "uninitialised variable" warnings at call sites.

**Why returning `hc` matters:** after a failed lookup (key not found), the caller now has the hash ready. If it then wants to insert the key, it can call `rawGetKnownHC(t, key, hc)` directly — reusing the already-computed hash — instead of recomputing it. This pattern appears throughout `tableimpl` for efficient insert-or-update operations.

**Usage pattern (conceptual):**
```nim
var hc: Hash
var idx = rawGet(t, key, hc)
if idx >= 0:
    # key found at t.data[idx]
else:
    # key absent; insert at -1 - idx
    # hc already computed, reuse it:
    rawInsert(t, t.data, key, val, hc, -1 - idx)
```

---

## How the pieces fit together

The following diagram shows the call flow for a typical `Table` lookup and insert:

```
t[key] = val
  │
  ├─ rawGet(t, key, hc)
  │     ├─ genHashImpl(key, hc)       # compute & zero-remap hash
  │     └─ rawGetKnownHCImpl()        # probe loop → slot index or negative
  │
  ├─ if slot found (idx ≥ 0):
  │     t.data[idx].val = val         # update in place
  │
  └─ if not found (idx < 0):
        mustRehash(t)?
          yes → enlarge(t)            # double array, reinsert all
                rawGetKnownHC(t, key, hc)  # re-probe in new array
          no  → use -1 - idx          # insert at the slot found during lookup
        rawInsert(t, t.data, key, val, hc, insertIdx)
```

The elegance is that a single probe pass simultaneously answers "is the key present?" and "where would I insert it if not?" — never probing twice.
