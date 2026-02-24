# `sharedlist` Module Reference

> **Module:** `std/sharedlist`  
> **Language:** Nim  
> **Status:** ⚠️ **Deprecated — Unstable API.** The module carries `{.deprecated.}` and is explicitly marked as having an unstable API. Do not use it in new code. For thread-safe shared collections, prefer `std/sharedtables` or design around message passing with `std/channels`.  
> **Purpose:** A generic singly-linked list that lives in shared memory, designed to be accessible from multiple threads. Backed by a coarse-grained `Lock`.

---

## Overview

`SharedList[A]` is a thread-safe append-only-like list where all data is stored in **shared memory** (allocated with `allocShared0`), making it safe to access from any thread regardless of the GC. The structure is a singly-linked list of fixed-size **chunk nodes**, each holding up to `ElemsPerNode = 100` elements of type `A`.

### Storage model: chunked nodes

Elements are not stored one-per-node. Instead, each `SharedListNode[A]` is a chunk that holds an array of 100 elements. New nodes are allocated only when the current tail chunk is full. This design amortises allocation overhead — instead of one allocation per element, you get one allocation per 100 elements.

```
head ──► [node: d[0..99], dataLen=100] ──► [node: d[0..5], dataLen=6, next=nil] ◄── tail
```

Traversal always goes from `head` to `tail`. Elements within a node occupy `d[0..dataLen-1]`.

### Thread safety model

All exported procedures (except `init`) use the embedded `lock: Lock` via the internal `withLock` template, which acquires the lock before the operation and releases it after. The lock is a **coarse-grained** mutex — every operation on the list, including iteration, holds the single lock for its entire duration.

The notable exception is `iterAndMutate`, which deliberately releases and re-acquires the lock around each call to the user-supplied `action` procedure. This design choice — and its consequences — is the most important thing to understand about this module.

---

## Types

### `SharedList[A]`

```nim
type SharedList*[A] = object
  head, tail: SharedListNode[A]
  lock*: Lock
```

The public type. `A` is the element type. `head` and `tail` point to the first and last chunk nodes respectively; both are `nil` in an empty list. The `lock` field is exported (`*`) — in principle you could acquire it manually, but this is not recommended.

`SharedList` is a value type (`object`, not `ref`). It must be stored in shared memory if you want to share it between threads — embed it in a `ptr`-allocated struct or a global variable:

```nim
var sharedGlobal: SharedList[int]  # global — accessible from all threads
```

---

## Lifecycle functions

### `init`

```nim
proc init*[A](t: var SharedList[A])
```

#### What it does

Initialises a `SharedList` before first use. Calls `initLock` on the embedded lock and sets both `head` and `tail` to `nil`. **Must be called before any other operation** on the list — using a `SharedList` without calling `init` first leaves the lock uninitialised, which is undefined behaviour.

`init` does **not** acquire the lock (the lock is not yet initialised when `init` runs), so it must be called from a single thread before the list is shared.

#### Example

```nim
import std/sharedlist

var list: SharedList[int]
list.init()
```

---

### `deinitSharedList`

```nim
proc deinitSharedList*[A](t: var SharedList[A])
```

#### What it does

Tears down the list completely: calls `clear` to free all chunk nodes, then calls `deinitLock` to destroy the lock. After this call, the `SharedList` object must not be used again without a fresh `init`.

This is the correct counterpart to `init`. Always call it when the list is no longer needed to avoid memory leaks and lock resource leaks.

#### Example

```nim
list.deinitSharedList()
```

---

### `clear`

```nim
proc clear*[A](t: var SharedList[A])
```

#### What it does

Frees all chunk nodes, setting `head` and `tail` back to `nil`. The list becomes empty, but the lock is left intact and the `SharedList` object remains usable — you can `add` to it again after `clear`.

Acquires the lock for the entire traversal and deallocation.

Note: `clear` does **not** call destructors or finalizers on the elements stored in the chunks. If `A` contains managed pointers or other resources, you are responsible for cleaning them up before calling `clear`.

#### Example

```nim
list.clear()
# list is now empty but still usable
list.add(42)
```

---

## Mutation functions

### `add`

```nim
proc add*[A](x: var SharedList[A]; y: A)
```

#### What it does

Appends element `y` to the end of the list. Acquires the lock, then:

- If the list is empty (`tail == nil`): allocates a new chunk node, sets it as both `head` and `tail`, stores `y` in position 0.
- If the tail chunk is full (`dataLen == ElemsPerNode`, i.e. 100 elements): allocates a new chunk node, links it as the new `tail`, stores `y` in position 0.
- Otherwise: appends `y` to the existing tail chunk at position `dataLen` and increments `dataLen`.

Elements are stored in the order they are added. However, **`iterAndMutate` can disrupt this order** when elements are removed (see below).

#### Example

```nim
list.add(10)
list.add(20)
list.add(30)
```

---

## Iteration and query functions

### `items` (iterator)

```nim
iterator items*[A](x: var SharedList[A]): A
```

#### What it does

Iterates over all elements from `head` to `tail`, yielding each value in storage order. Acquires the lock for the entire duration of the iteration — no other thread can `add` or `clear` while this iterator is running.

Because the lock is held throughout, `items` provides a consistent snapshot of the list, but it **blocks all writers** for as long as the iteration takes.

#### Example

```nim
for val in list.items:
  echo val
```

---

### `iterAndMutate`

```nim
proc iterAndMutate*[A](x: var SharedList[A]; action: proc(x: A): bool)
```

#### What it does

Iterates over all elements and gives the caller the opportunity to remove each one. For every element, `action` is called with the element's value. If `action` returns `true`, the element is removed.

This is the most complex function in the module, and its locking behaviour requires careful attention.

**The removal strategy:** When an element at position `i` in node `n` is to be removed, it is replaced by the **last element in the tail node** (`t.d[t.dataLen - 1]`), and the tail's `dataLen` is decremented. This is a swap-with-last-of-tail operation. It is O(1) and avoids shifting elements, but it **does not preserve element order**. The module's own documentation warns of this.

**The lock release pattern:** For each element, the lock is **released** before calling `action`, and **re-acquired** after it returns. This is intentional: it allows `action` to call `list.add` without deadlocking (since `add` also tries to acquire the lock). However, this creates a window where another thread can modify the list between the release and re-acquire — the iteration is not atomic.

Specifically:
- Another thread can `add` elements to the tail while `action` is running.
- The iteration will visit those newly added elements if they end up in a node that hasn't been visited yet (because node traversal moves `head` to `tail` and new nodes are appended at the tail).
- The removal logic (swap-with-tail) can move an element from the tail to a position already past the current iterator position, meaning it won't be visited in the current pass.

There is also a known design limitation mentioned in the source comment: when the tail node's `dataLen` reaches 0 after removals, the tail node is not deallocated or unlinked. This is a TODO in the code, and means that empty tail nodes can accumulate after heavy removal activity.

#### Example

```nim
# Remove all even numbers:
list.iterAndMutate(proc(x: int): bool = x mod 2 == 0)
```

#### When to use it vs `items`

| | `items` | `iterAndMutate` |
|-|---------|-----------------|
| Removes elements? | No | Yes, when `action` returns `true` |
| Lock held during callback? | Yes (whole iteration) | No (released per element) |
| Order preserved? | Yes | No (after removals) |
| `action` can call `list.add`? | No (would deadlock) | Yes (lock released first) |
| Consistent snapshot? | Yes | No |

---

## Summary table

| Function | Acquires lock | Notes |
|----------|--------------|-------|
| `init` | No | Must be called first; lock not yet initialised |
| `add` | Yes, full duration | Allocates new chunk every 100 elements |
| `items` | Yes, full iteration | Blocks all writers; consistent snapshot |
| `iterAndMutate` | Yes, with per-element release | Non-atomic; allows `add` from `action`; disrupts order on removal |
| `clear` | Yes, full duration | Frees nodes; does not call element destructors |
| `deinitSharedList` | Via `clear` | Frees nodes + destroys lock; list must not be used after |

---

## Notes and gotchas

- **Deprecated.** This module is deprecated and has an explicitly unstable API. Use it only for maintenance of existing code.
- **`init` is mandatory.** Forgetting to call `init` before the first use leaves the lock uninitialised, which causes undefined behaviour (likely a crash or data race).
- **`A` must be safe in shared memory.** The element type `A` must not contain GC-managed pointers (refs, strings with GC, seqs), because those are not valid across thread boundaries in Nim's GC model. Use plain data, `ptr`-based types, or `cstring` instead.
- **`iterAndMutate` is non-atomic.** The lock release/reacquire around each `action` call means the list can change between element visits. Do not rely on seeing every element exactly once.
- **Empty tail nodes after removal.** When the tail node's `dataLen` reaches 0 due to removals in `iterAndMutate`, the empty node remains linked as the tail. Subsequent `add` calls will use it (since `dataLen < ElemsPerNode`), so it won't be allocated anew, but a zero-`dataLen` tail node is a transient state the module was not designed to handle fully.
- **Order is not guaranteed after `iterAndMutate`.** The swap-with-last-of-tail removal strategy moves elements. If element order matters, do not use `iterAndMutate` for removal.
- **`lock` is exported but should not be acquired manually.** Acquiring `t.lock` outside of the module's own functions while simultaneously calling module functions from another thread will deadlock.
- **Stack traces are disabled.** The module pushes `{.stackTrace: off.}`, so exceptions or errors inside shared list operations will produce less informative stack traces.
