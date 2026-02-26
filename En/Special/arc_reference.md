# `arc` Module Reference

> **Module:** `arc.nim` — Nim's Runtime Library  
> **Purpose:** Core implementation of Nim's **ARC** (Automatic Reference Counting) memory management runtime, with optional extensions for **ORC** (cycle collector on top of ARC) and **Atomic ARC** (thread-safe reference counting). Every `ref` allocation, every copy of a `ref`, and every `ref` going out of scope passes through the procedures defined here.

---

## Overview

ARC is Nim's deterministic memory management strategy, introduced as a modern alternative to the classic garbage collector. It works by embedding a reference count in a small header (`RefHeader`) placed immediately before every heap-allocated object. When the count reaches zero the object is destroyed immediately — no GC pause, no background thread, no unpredictable latency.

This module defines:
- The `RefHeader` layout and the `Cell` alias for a pointer to it.
- Low-level allocation procedures called directly by compiler-generated code.
- Reference count manipulation (increment, decrement, check for last reference).
- The `=dispose` operator for `owned ref`.
- Utility procedures for debugging, thread safety, and runtime type checks.

All procedures marked `compilerRtl` are called by compiler-generated code — you never invoke them from Nim source directly. The handful of `*`-exported procedures are part of the public API and intended for library authors and advanced users.

---

## Internal Types and Constants

### `RefHeader`

```nim
type RefHeader = object
  rc: int
  # (ORC only)       rootIdx: int
  # (debug/IDs)      refId: int
  # (ORC + leak det) filename: cstring; line: int
```

The header that lives **just before** every heap-allocated `ref` object in memory. Its core field is `rc` — the packed reference count.

The reference count is not stored as a bare integer. The lowest 3 bits (ARC) or 4 bits (ORC) are reserved for runtime flags, and the actual count lives in the upper bits. This packing lets flag-bits and the count coexist in a single word without extra memory.

| Mode | `rcIncrement` | `rcMask` | `rcShift` | Reserved bits |
|---|---|---|---|---|
| ARC | `0b1000` (8) | `0b111` | 3 | 3 lowest bits |
| ORC | `0b10000` (16) | `0b1111` | 4 | 4 lowest bits |

**Optional fields** (compiled in only when the corresponding define is active):
- `rootIdx` — ORC only. Index into the cycle-root candidate list, enabling O(1) removal of a potential cycle root without doubly-linked lists.
- `refId` — Debug/ID mode. A monotonically increasing integer that uniquely identifies this allocation, making it easy to set a conditional breakpoint like "break when object #42 is allocated."
- `filename` / `line` — ORC leak detector. Records the source location of the allocation for leak reports.

### `Cell`

```nim
type Cell = ptr RefHeader
```

A typed alias for "a pointer to the header of a ref object." Used throughout the module to make header manipulation readable.

### `head` template

```nim
template head(p: pointer): Cell
```

Given a user-visible pointer `p` (pointing at the object data), steps back in memory by `sizeof(RefHeader)` bytes to reach the `RefHeader`. This is the fundamental bridge between the user's view of a ref and the runtime's view of its header.

```
Memory layout:
[ RefHeader | ... object data ... ]
^            ^
Cell         p  (what the programmer holds)
head(p) ──►  Cell
```

---

## Exported Procedures (Public API)

---

### `isUniqueRef`

```nim
proc isUniqueRef*[T](x: ref T): bool {.inline.}
```

**What it does:**  
Returns `true` if `x` is the **only** living reference to the object — that is, the internal reference count is exactly zero. In ARC's "start at zero" convention, `rc = 0` means exactly one owner exists; `rc` is only incremented when a second copy is made. So `rc == 0` genuinely means "unique."

**What it is good for:**

*Thread-safety verification.* Before transferring a `ref` to another thread, you can assert that no other thread holds a copy:
```nim
assert isUniqueRef(myObj), "object must be uniquely owned before thread transfer"
channel.send(myObj)
```

*In-place mutation optimisation.* Some APIs can take a faster path when the object is unshared:
```nim
if isUniqueRef(buf):
  mutateInPlace(buf)   # safe: nobody else sees the change
else:
  buf = buf.deepCopy() # must copy first
  mutateInPlace(buf)
```

*Ownership contracts as checked assertions.* `assert isUniqueRef(x)` turns an informal ownership requirement into a runtime-verified invariant.

**Important caveat:**  
The compiler and optimiser are free to eliminate or reorder reference count operations. The result of `isUniqueRef` can vary across compiler versions and optimisation levels. Never use it for correctness-critical logic — only for debugging and assertions.

---

### `GC_ref`

```nim
proc GC_ref*[T](x: ref T)
```

**What it does:**  
Manually increments the reference count of `x` by one. If `x` is `nil`, does nothing.

Under ARC/ORC there is no classical garbage collector, so `GC_ref` is a compatibility shim for code written for Nim's older GC-based runtimes. Internally it calls `nimIncRef`.

**When you need it:**  
When interfacing with C code or another runtime that keeps a raw pointer to a Nim `ref` object without going through normal Nim variable semantics. If external code holds a raw pointer alive beyond the lifetime of all Nim variables pointing to that object, you must manually bump the count so ARC doesn't destroy the object prematurely.

```nim
var obj: ref MyStruct = new MyStruct
GC_ref(obj)                          # manually bump; rc is now 1
passToCCallback(cast[pointer](obj))  # C code may outlive 'obj' variable
# ... C callback finishes ...
GC_unref(obj)                        # matches GC_ref; object freed when all refs drop
```

---

### `GC_unref`

```nim
proc GC_unref*[T](x: ref T)
```

**What it does:**  
Manually decrements the reference count of `x` by one, triggering destruction if the count reaches zero. The exact counterpart of `GC_ref`. Internally calls `=destroy` on a cursor (non-owning view) of `x`, which engages the standard ARC teardown path.

**Warning:** calling `GC_unref` without a prior matching `GC_ref` will over-decrement the count, causing premature destruction and a use-after-free. Always use `GC_ref` and `GC_unref` in matched pairs.

```nim
GC_ref(obj)
passToCCallback(cast[pointer](obj))
GC_unref(obj)   # matches the GC_ref above
```

---

### `=dispose` (operator)

```nim
template `=dispose`*[T](x: owned(ref T))
```

**What it does:**  
Immediately and unconditionally deallocates the object pointed to by `x`, bypassing the reference count. This is the ARC-era replacement for calling `dealloc` on an owned ref.

When you hold an `owned ref` — exclusive ownership proven by the compiler — you can release it at a precisely chosen moment with `=dispose` instead of waiting for the enclosing scope to end.

**Safety guard** (when `nimOwnedEnabled` is defined): before deallocating, checks that `rc < rcIncrement` (no other owners). If the check fails, the program aborts with a fatal message — protecting against disposing an object while live aliases exist.

```nim
var buf: owned(ref BigBuffer) = new BigBuffer
fillBuffer(buf)
processBuffer(buf)
`=dispose`(buf)    # explicitly free right here; don't wait for scope end
                   # any use of 'buf' after this is undefined behaviour
```

---

### `GC_fullCollect`

```nim
template GC_fullCollect*
```

**What it does:**  
A **no-op** under `--mm:arc`. Present purely for source-level compatibility with code written for Nim's older mark-and-sweep or reference-counting GC runtimes, where this call would trigger an immediate collection pass.

Under ARC, memory is reclaimed the instant the reference count hits zero — there is no deferred collection to trigger. Under ORC, cycle collection is managed automatically.

```nim
# Legacy code compiles unchanged under ARC:
GC_fullCollect()  # silently does nothing
```

---

### `setupForeignThreadGc` / `tearDownForeignThreadGc`

```nim
template setupForeignThreadGc*
template tearDownForeignThreadGc*
```

**What they do:**  
Both are **no-ops** under ARC/ORC. In Nim's older GC runtimes every thread that interacted with GC-managed data had to register and deregister itself with the collector. Under ARC that bookkeeping is unnecessary — reference counting is per-object and optionally atomic, not per-thread.

```nim
# Legacy threading code compiles unchanged:
setupForeignThreadGc()
doWorkWithNimObjects()
tearDownForeignThreadGc()
# All three lines are no-ops under --mm:arc
```

---

### `deallocatedRefId` (debug-only)

```nim
proc deallocatedRefId*(p: pointer): int
```

**Available when:** compiled with `-d:nimArcDebug`

**What it does:**  
Checks whether the object at raw pointer `p` has **already been freed**, and returns its unique `refId` if so (use-after-free detected), or `0` if the object is still alive.

Under `nimArcDebug`, `nimRawDispose` does **not** actually release memory — instead it adds the cell to a `freedCells` set. This intentional "memory leak during debug" means the memory address remains valid and readable after logical destruction, so `deallocatedRefId` can definitively tell you "yes, you are holding a dangling pointer to object #N."

```nim
# Compile with: nim c -d:nimArcDebug myfile.nim

var p: ref int = new int
let raw = cast[pointer](p)
p = nil   # drops last reference; object is "freed" (added to freedCells)

let id = deallocatedRefId(raw)
if id != 0:
  echo "Use-after-free! Object refId = ", id  # prints the object's unique ID
```

---

## Compiler-Internal Procedures

These are called exclusively by compiler-generated code. Understanding them is useful when reading generated C output, studying ARC behaviour, or debugging memory issues at the lowest level.

---

### `nimNewObj`

```nim
proc nimNewObj(size, alignment: int): pointer {.compilerRtl.}
```

**What it does:**  
Allocates `size + hdrSize` bytes of **zero-initialised** memory with the specified alignment, then returns a pointer past the header (to the object data). The `rc` field of the new header is `0` — one owner, no copies yet.

This is called for every `new T` or `ref T` allocation where memory must start zeroed, which is the default for all `object` types in Nim.

```
After nimNewObj:
[ RefHeader (rc=0) | zeroed object data ]
                    ^
                    returned pointer (what the programmer's ref holds)
```

**Debug features:** When `nimArcDebug` or `nimArcIds` is defined, assigns a unique `refId` from the global counter `gRefId`. If `refId` matches the compile-time constant `traceId`, prints a stack trace — a targeted allocation breakpoint without needing an interactive debugger.

```nim
# Your Nim code:
var x: ref MyObj = new MyObj
# Compiler emits approximately:
#   x = nimNewObj(sizeof(MyObj), alignOf(MyObj))
```

---

### `nimNewObjUninit`

```nim
proc nimNewObjUninit(size, alignment: int): pointer {.compilerRtl.}
```

**What it does:**  
Identical to `nimNewObj` but allocates **uninitialised** memory. The compiler uses this only when data-flow analysis proves every field will be written before it is read — so zero-filling would be wasted work. The `rc` and `rootIdx` header fields are still explicitly zeroed, as the runtime's own invariants require them.

**Why it matters:** For large objects or high-allocation-frequency code paths, skipping the `memset` to zero can be a meaningful performance gain. The compiler makes this decision automatically; the programmer does not need to opt in.

---

### `nimIncRef`

```nim
proc nimIncRef(p: pointer) {.compilerRtl, inl.}
```

**What it does:**  
Increments the reference count of the object at `p` by one `rcIncrement`. Called every time a new copy of a `ref` is made — when a `ref` is assigned to another variable, passed as a function argument, or stored in a data structure.

Under `--mm:atomicArc` with thread support uses an atomic increment (`atomicInc`). Otherwise uses plain integer arithmetic.

---

### `nimDecWeakRef`

```nim
proc nimDecWeakRef(p: pointer) {.compilerRtl, inl.}
```

**What it does:**  
Decrements the reference count of the object at `p` by one `rcIncrement` **without** triggering destruction, even if the count reaches zero. Used for weak references — references that observe an object without owning it, so they must not keep it alive past the point where all owning references drop.

After a weak-ref decrement you must verify the object is still alive before using it, because the owning references may have already destroyed it.

---

### `nimDecRefIsLast`

```nim
proc nimDecRefIsLast(p: pointer): bool {.compilerRtl, inl.}
```

**What it does:**  
Decrements the reference count of `p` and returns `true` if this decrement brought the count to zero — meaning this was the **last** reference and the object should now be destroyed.

The compiler generates a call to this procedure at every point where a `ref` goes out of scope or is overwritten. The surrounding generated code pattern is:

```c
// compiler-generated C:
if (nimDecRefIsLast(p)) {
    nimDestroyAndDispose(p);
}
```

Under Atomic ARC the decrement uses `atomicDec` and checks for the sentinel value `-rcIncrement` (the result of decrementing below zero in the packed encoding).

---

### `nimRawDispose`

```nim
proc nimRawDispose(p: pointer, alignment: int) {.compilerRtl.}
```

**What it does:**  
Unconditionally frees the memory backing the object at `p`, including its `RefHeader`. Computes `hdrSize = align(sizeof(RefHeader), alignment)`, steps back that many bytes to find the start of the original allocation, and calls `alignedDealloc`.

Under `nimArcDebug` — instead of freeing — adds the cell to `freedCells` to enable use-after-free detection.
Under `nimOwnedEnabled` — asserts `rc < rcIncrement` before freeing, catching bugs where disposal happens while aliases still exist.

---

### `nimDestroyAndDispose`

```nim
proc nimDestroyAndDispose(p: pointer) {.compilerRtl, quirky, raises: [].}
```

**What it does:**  
The two-phase teardown executed when a `ref` object's reference count hits zero:

1. **Destroy** — reads the object's `PNimTypeV2` type info and calls the type's `destructor` hook (if any). The destructor is responsible for recursively decrementing refs held by the object's own fields, triggering their teardowns in turn.
2. **Dispose** — calls `nimRawDispose` to release the raw memory.

The two phases are separated because the destructor may read `self` and dereference other objects during its execution. Freeing the memory before the destructor finishes would be a use-after-free.

---

### `isObjDisplayCheck`

```nim
proc isObjDisplayCheck(source: PNimTypeV2, targetDepth: int16, token: uint32): bool {.compilerRtl, inl.}
```

**What it does:**  
The fast O(1) runtime implementation of Nim's `of` operator (type membership test). Every type in the inheritance hierarchy has a `display` array and a `depth` field in its `PNimTypeV2` descriptor. To check `x of T`:

1. `targetDepth` is the compile-time-known depth of `T` in the hierarchy (root = 0).
2. `token` is `T`'s unique identity token.
3. The check reads `source.display[targetDepth]` and compares it to `token`.

If `source` is a subtype of `T`, then `T`'s token will be stored at index `targetDepth` in `source`'s display array — a single bounds check plus one array read, regardless of how deep the hierarchy is.

```nim
type
  Animal = ref object of RootObj
  Dog    = ref object of Animal
  Cat    = ref object of Animal

var d: Animal = Dog()
echo d of Dog     # true  — isObjDisplayCheck(Dog's type, Dog.depth, Dog.token)
echo d of Animal  # true  — Dog's display also contains Animal's token
echo d of Cat     # false — Cat's token is not in Dog's display
```

---

### `nimGetVTable`

```nim
proc nimGetVTable(p: pointer, index: int): pointer {.compilerRtl, inline, raises: [].}
```

**Available when:** compiled with `-d:gcDestructors`

**What it does:**  
Retrieves the function pointer at position `index` in the virtual method table (vtable) of the object at `p`. The vtable is embedded in the `PNimTypeV2` type descriptor, which sits at offset 0 of every object.

This is the mechanism behind Nim's `method` dispatch. The compiler translates a virtual method call into a `nimGetVTable` lookup followed by an indirect function call through the retrieved pointer.

---

## Key Design Principles

**Header-before-data layout.** The `RefHeader` sits immediately before the user-visible object data in memory. The single `head(p)` template navigates between the two views. Non-ref (stack or embedded) objects carry no header at all — zero overhead for the common case.

**Packed reference count.** The `rc` field stores both the reference count (upper bits) and status flags (lower bits) in a single `int`. Incrementing and decrementing by `rcIncrement` (not by 1) leaves the flag bits untouched, so counting and flag operations are always independent.

**"Start at zero" count convention.** A freshly allocated object has `rc = 0`, meaning one owner. This is an important optimisation: the overwhelmingly common case of a single-owner `ref` never needs to touch `rc` at all. `rc` only becomes non-zero when a second owner is created.

**Atomic vs non-atomic.** Under `--mm:atomicArc` with `hasThreadSupport`, `increment` and `decrement` use CPU atomic instructions (`atomicInc`/`atomicDec`). Otherwise they are plain integer operations — faster but unsafe to share across threads. Use `--mm:atomicArc` only when you genuinely share `ref` objects between threads.

**ORC cycle extension.** When compiled with `--mm:orc`, the `rootIdx` field in `RefHeader` enables the cycle collector to find and remove potential cycle roots in O(1). The cycle collector (implemented in `orc.nim`, conditionally included) handles what pure ARC cannot: cyclic object graphs where no reference count ever reaches zero through normal decrement alone.
