# SharedTable — Reference Manual

> **⚠ Deprecated:** This module is marked `{.deprecated.}` in Nim's standard library. It is preserved for legacy code. For new projects consider alternatives such as `std/tables` combined with explicit synchronisation primitives, or a purpose-built concurrent map.

---

## Overview

`sharedtables` provides a **generic hash table that lives in shared memory**, making it safe to use across threads. The entire table is protected by a **single mutual-exclusion lock**, so all operations are serialised — only one thread can read or write at a time.

### Key design rules

- **Use only non-GC types** for keys (`A`) and values (`B`). Because the table lives in shared memory (allocated with `allocShared0`), the Nim garbage collector is unaware of it. Storing GC-managed types (strings, `seq`, `ref` objects, etc.) will cause memory corruption or crashes.
- **Always call `init` first.** The table is a plain `object` — no automatic initialisation happens. Forgetting `init` means the lock and data pointer are uninitialised.
- **Always call `deinitSharedTable` when done.** Failing to do so leaks the shared memory allocation and the OS lock resource.
- **Lock contention is intentional.** The module's own documentation says: if contention is so high that you need a lock-free table, you're doing it wrong. Redesign the algorithm first.

### Type signature

```nim
type SharedTable*[A, B] = object
```

`A` is the key type, `B` is the value type. Both must be copyable and hashable (have a `hash` proc).

---

## Exported API

### `init`

```nim
proc init*[A, B](t: var SharedTable[A, B], initialSize = 32)
```

**Purpose:** Allocates the internal storage array in shared memory and initialises the lock. This is the constructor — nothing else may be called before `init`.

`initialSize` is a *hint* for the initial number of slots. The library rounds it up to the next power-of-two that keeps the load factor healthy. A good initial size avoids early rehashing and can improve performance when the approximate number of elements is known in advance.

**Example**

```nim
var t: SharedTable[int32, int32]
t.init(64)          # hint: room for ~64 elements before first rehash
# ... use the table ...
deinitSharedTable(t)
```

---

### `deinitSharedTable`

```nim
proc deinitSharedTable*[A, B](t: var SharedTable[A, B])
```

**Purpose:** Releases the shared-memory allocation and destroys the lock. This is the destructor — call it exactly once when the table is no longer needed. After this call the table object is invalid and must not be used.

**Example**

```nim
var t: SharedTable[int32, float32]
t.init()
# ... use t ...
deinitSharedTable(t)   # clean up
```

---

### `[]=` (assignment / insert or update)

```nim
proc `[]=`*[A, B](t: var SharedTable[A, B], key: A, val: B)
```

**Purpose:** Inserts a new key-value pair, or **replaces** the existing value if the key is already present. This is the standard "upsert" operation — the table always ends up with exactly one entry for `key`.

The lock is held for the duration of the operation, so the update is atomic from the perspective of other threads.

**Example**

```nim
var t: SharedTable[int32, int32]
t.init()
t[1'i32] = 100'i32     # insert
t[1'i32] = 200'i32     # replace — now t[1] == 200
deinitSharedTable(t)
```

---

### `add`

```nim
proc add*[A, B](t: var SharedTable[A, B], key: A, val: B)
```

**Purpose:** Inserts a new pair **without checking for an existing entry**. If the key already exists the table will contain **two entries with the same key** — a so-called "multimap" entry. Standard lookup (`mget`, `withValue`) will find only one of them (whichever open-addressing resolves first), making the other permanently unreachable.

Use `add` only when you are certain the key is not already present, or when you deliberately want a multimap-like structure and have a custom traversal strategy. In almost all other situations `[]=` is the right choice.

**Example**

```nim
var t: SharedTable[int32, int32]
t.init()
t.add(1'i32, 10'i32)
t.add(1'i32, 20'i32)   # ⚠ duplicate key — second value is unreachable via mget
deinitSharedTable(t)
```

---

### `del`

```nim
proc del*[A, B](t: var SharedTable[A, B], key: A)
```

**Purpose:** Removes the entry with the given key from the table. If the key does not exist, the call is silently a no-op — no exception is raised.

**Example**

```nim
var t: SharedTable[int32, int32]
t.init()
t[5'i32] = 99'i32
t.del(5'i32)             # entry removed
t.del(99'i32)            # key not present — no error
deinitSharedTable(t)
```

---

### `len`

```nim
proc len*[A, B](t: var SharedTable[A, B]): int
```

**Purpose:** Returns the current number of key-value pairs stored in the table. The lock is acquired to read the internal counter, so the result is consistent. Note that in a concurrent system the count may have changed by the time the caller uses it — treat the result as a snapshot.

**Example**

```nim
var t: SharedTable[int32, int32]
t.init()
t[1'i32] = 1'i32
t[2'i32] = 2'i32
assert t.len() == 2
deinitSharedTable(t)
```

---

### `mget`

```nim
proc mget*[A, B](t: var SharedTable[A, B], key: A): var B
```

**Purpose:** Returns a **mutable reference** to the value stored at `key`. If `key` is absent a `KeyError` exception is raised.

Because `mget` returns a reference (`var B`) into the table's internal storage, the caller must treat it as a snapshot — writing through the reference after the lock has been released is **not thread-safe** and can cause data races. For safe in-place modification under the lock, prefer `withValue` or `withKey`.

**Example**

```nim
var t: SharedTable[int32, int32]
t.init()
t[7'i32] = 42'i32
let v = t.mget(7'i32)   # v == 42
# t.mget(99'i32)         # would raise KeyError
deinitSharedTable(t)
```

---

### `mgetOrPut`

```nim
proc mgetOrPut*[A, B](t: var SharedTable[A, B], key: A, val: B): var B
```

**Purpose:** Looks up `key`. If found, returns a mutable reference to the existing value. If not found, inserts `(key, val)` and returns a mutable reference to the freshly inserted value.

The whole operation (lookup + optional insert) is performed under the lock. **However**, the returned reference is to internal storage and becomes unsafe the moment another thread modifies or rehashes the table. The module's own documentation flags this as *inherently unsafe in the context of multi-threading*. Use it only when you are certain no concurrent modification can happen (e.g. during a single-threaded initialisation phase), or immediately copy the returned value.

**Example**

```nim
var t: SharedTable[int32, int32]
t.init()
# Key 3 absent → inserts (3, 0) and returns ref to 0
var val = t.mgetOrPut(3'i32, 0'i32)
assert val == 0
# Key 3 now exists → returns ref to existing value
var val2 = t.mgetOrPut(3'i32, 99'i32)
assert val2 == 0   # original value, 99 was not used
deinitSharedTable(t)
```

---

### `hasKeyOrPut`

```nim
proc hasKeyOrPut*[A, B](t: var SharedTable[A, B], key: A, val: B): bool
```

**Purpose:** Returns `true` if `key` already exists in the table, and does **not** modify the table. Returns `false` and inserts `(key, val)` if the key was absent. The check and the potential insert are a single atomic operation under the lock.

This is a convenient "insert-if-absent" primitive — useful for initialising default values on first access without a separate existence check followed by a write.

**Example**

```nim
var t: SharedTable[int32, int32]
t.init()
assert t.hasKeyOrPut(1'i32, 10'i32) == false  # absent → inserts 10
assert t.hasKeyOrPut(1'i32, 20'i32) == true   # present → no change
assert t.mget(1'i32) == 10'i32                # value is still 10
deinitSharedTable(t)
```

---

### `withValue` (two-arm form — key found only)

```nim
template withValue*[A, B](t: var SharedTable[A, B], key: A,
                           value, body: untyped)
```

**Purpose:** Looks up `key` under the lock. If found, injects a pointer (`value`) to the stored value and executes `body` with the lock still held. If the key is absent, `body` is skipped. The lock is released when `body` finishes (via a `try/finally`).

This is the **safest way to read or modify a value in place** because the lock guarantees exclusive access for the entire duration of `body`. The injected `value` variable is of type `ptr B` — dereference with `value[]`.

**Example**

```nim
var t: SharedTable[int32, int32]
t.init()
t[5'i32] = 100'i32

t.withValue(5'i32, v):
  echo v[]        # prints 100
  v[] = 200       # safe in-place update while lock is held

t.withValue(99'i32, v):
  echo "not called — key absent"

deinitSharedTable(t)
```

---

### `withValue` (three-arm form — key found / not found)

```nim
template withValue*[A, B](t: var SharedTable[A, B], key: A,
                           value, body1, body2: untyped)
```

**Purpose:** Same as the two-arm form but accepts a second body (`body2`) that is executed when the key is **absent**. This lets you handle both the "found" and "not found" cases atomically within a single lock acquisition.

The `do:` syntax is used to supply the second branch.

**Example**

```nim
var t: SharedTable[int32, int32]
t.init()
t[1'i32] = 10'i32

# Key present: increment value
t.withValue(1'i32, v):
  inc v[]
do:
  t[1'i32] = 1'i32   # not reached here, but shows the pattern

# Key absent: insert default
t.withValue(2'i32, v):
  inc v[]
do:
  t[2'i32] = 0'i32   # initialise, then increment next time

assert t.mget(1'i32) == 11'i32
deinitSharedTable(t)
```

---

### `withKey`

```nim
proc withKey*[A, B](t: var SharedTable[A, B], key: A,
                    mapper: proc(key: A, val: var B, pairExists: var bool))
```

**Purpose:** The most powerful and flexible mutation primitive. It calls `mapper` **under the lock**, passing the current key, the current (or zero-initialised) value, and a boolean indicating whether the key exists. The mapper can:

- **Read** `val` and `pairExists` without changing anything.
- **Modify** `val` to update an existing value or set the value for a new entry (set `pairExists = true` to confirm insertion).
- **Delete** the entry by setting `pairExists = false`.

Because the entire read-modify-write cycle runs under the lock, there are no race windows — this is impossible to replicate safely with a combination of `mget` + `[]=`.

The mapper should be **short and non-blocking** — it holds the global table lock for its entire execution, blocking all other threads.

**Example — atomic decrement with auto-delete**

```nim
var t: SharedTable[int32, int32]
t.init()
t[1'i32] = 3'i32

# Decrement; remove the key when the counter reaches zero
t.withKey(1'i32) do (k: int32, v: var int32, exists: var bool):
  if exists:
    dec v
    if v <= 0:
      exists = false   # signals deletion

assert t.len() == 1    # counter was 3 → now 2, key still present

deinitSharedTable(t)
```

**Example — insert only if absent**

```nim
t.withKey(42'i32) do (k: int32, v: var int32, exists: var bool):
  if not exists:
    v = 100'i32
    exists = true      # confirm insertion
```

---

## Thread-safety summary

| Operation | Atomic? | Returns live reference? |
|---|---|---|
| `[]=` | ✅ yes | — |
| `del` | ✅ yes | — |
| `add` | ✅ yes | — |
| `len` | ✅ yes (snapshot) | — |
| `hasKeyOrPut` | ✅ yes | — |
| `withValue` (both forms) | ✅ yes | `ptr B` valid inside body only |
| `withKey` | ✅ yes | — |
| `mget` | ⚠ lock released before return | `var B` — unsafe after lock |
| `mgetOrPut` | ⚠ lock released before return | `var B` — unsafe after lock |

---

## Complete working example

```nim
import std/sharedtables, std/os

var hits: SharedTable[int32, int32]
hits.init(16)

proc worker(id: int32) {.thread.} =
  for i in 0'i32 ..< 1000'i32:
    hits.withKey(id) do (k: int32, v: var int32, exists: var bool):
      inc v
      if not exists: exists = true

var threads: array[4, Thread[int32]]
for i in 0 ..< 4:
  createThread(threads[i], worker, int32(i))
joinThreads(threads)

for i in 0'i32 ..< 4'i32:
  echo "worker ", i, " → ", hits.mget(i), " hits"

deinitSharedTable(hits)
```
