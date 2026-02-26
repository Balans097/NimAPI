# `assign` Module Reference

> **Module:** `assign.nim` — Nim's Runtime Library  
> **Purpose:** Low-level engine for copying, initialising, and resetting Nim values of any complexity. Every assignment statement in your Nim program — `a = b` — eventually calls one of the procedures defined here when the type involved contains managed memory (strings, sequences, refs, objects with variant fields). This module sits at the heart of Nim's memory safety guarantees.

---

## Overview

In Nim, simple value types (integers, floats, plain `char`) can be copied with a single `copyMem`. But as soon as a type contains a `string`, a `seq`, a `ref`, or a variant (`case`) object, the runtime must do more work: it must deep-copy heap data, inform the garbage collector about new references, reset the previous occupant of memory correctly, and handle self-assignment safely.

`assign.nim` implements this logic generically, driven by Nim's runtime type information (RTTI, represented as `PNimType` / `TNimNode`). The compiler emits calls to these procedures automatically — you never call them from application code.

---

## Exported (Compiler-Facing) Procedures

---

### `genericAssign`

```nim
proc genericAssign(dest, src: pointer, mt: PNimType)
```

**What it does:**  
Performs a **deep copy** of the value at `src` into the memory at `dest`, guided by the runtime type descriptor `mt`. "Deep" means that every heap-allocated component (strings, sequences, nested refs) is independently duplicated on the heap — after the copy, `dest` and `src` share no heap data.

This is the procedure the compiler calls for a plain `=` assignment whenever the type requires non-trivial copying. It delegates entirely to `genericAssignAux` with `shallow = false`.

**How it works internally:**  
`genericAssign` dispatches on `mt.kind`:
- `tyString` — copies the string's heap buffer (via `copyString` / `nimAsgnStrV2`)
- `tySequence` — allocates a new payload, then recursively copies each element
- `tyObject` — walks the inheritance chain, copies each field slot, then writes the static type pointer into the object header
- `tyTuple` — copies each field slot
- `tyArray` — copies each element individually
- `tyRef` — copies the reference itself (pointer), informing the GC via `unsureAsgnRef`
- Everything else — raw `copyMem` (the type contains no managed pointers)

**Why it matters:**  
Without deep copy semantics, modifying a sequence in one variable would silently corrupt another that was assigned from the first. `genericAssign` ensures value semantics for all Nim types.

**Example (from the Nim programmer's perspective):**
```nim
var a = @[1, 2, 3]
var b = a           # compiler emits genericAssign here
b.add(4)
echo a  # @[1, 2, 3]  — a is unaffected; b got its own heap copy
echo b  # @[1, 2, 3, 4]
```

---

### `genericShallowAssign`

```nim
proc genericShallowAssign(dest, src: pointer, mt: PNimType)
```

**What it does:**  
Performs a **shallow copy** of the value at `src` into `dest`. For strings and sequences, instead of duplicating the heap payload, only the header (pointer + length) is copied — both `dest` and `src` end up sharing the same underlying heap buffer.

This is the procedure called when you use Nim's `shallowCopy` or `shallow` pragma on a type. It delegates to `genericAssignAux` with `shallow = true`.

**The critical trade-off:**  
Shallow assignment is dramatically faster for large sequences and strings (no heap allocation, no element-by-element copy), but it creates aliasing: mutating the sequence through one variable mutates the other as well. This is intentional and opt-in.

**When the runtime treats an assignment as shallow automatically:**  
A string or sequence whose `reserved` field has `seqShallowFlag` set will be copied shallowly even by `genericAssign`. This happens with string literals and compile-time constants — they live in read-only memory and must not be duplicated.

**Example:**
```nim
var a = @[1, 2, 3]
shallowCopy(b, a)  # compiler emits genericShallowAssign
# b.p points to the same payload as a.p
# Mutating b would also mutate a — use with care
```

---

### `genericSeqAssign`

```nim
proc genericSeqAssign(dest, src: pointer, mt: PNimType)
```

**What it does:**  
A specialised wrapper around `genericAssign` for sequence values that are passed by value (not by pointer) — situations where the compiler has a sequence value on the stack rather than behind a pointer. It takes the address of `src` before forwarding to `genericAssign`, bridging the impedance mismatch.

The source code comment `# ugly, but I like to stress the parser sometimes :-)` from Andreas Rumpf hints that this is a workaround for how the C code generator represents sequence values in some contexts.

**Why it matters:**  
Without this adapter, the generic assign machinery (which always works through pointers) could not accept a bare sequence value. This makes the dispatch from generated C code uniform.

**Example (transparent to the Nim programmer):**
```nim
proc take(s: seq[int]) =
  var local = s  # may emit genericSeqAssign depending on codegen context
  echo local
```

---

### `genericAssignOpenArray`

```nim
proc genericAssignOpenArray(dest, src: pointer, len: int, mt: PNimType)
```

**What it does:**  
Copies `len` elements from the open array starting at `src` into the memory starting at `dest`, element by element, using `genericAssign` for each element. The element type information is taken from `mt.base`.

An `openArray[T]` at runtime is simply a pointer + length. This procedure knows how to walk those elements and deep-copy each one.

**Why it matters:**  
When you assign or pass an `openArray` whose elements are managed types (e.g., `openArray[string]`), each string must be individually deep-copied. Raw `copyMem` over the whole block would duplicate the header pointers but not the heap data — exactly the unsafe aliasing that this function prevents.

**Example:**
```nim
proc copyStrings(dest: var seq[string], src: openArray[string]) =
  dest.setLen(src.len)
  # internally the compiler may emit genericAssignOpenArray
  # to copy each string from src into dest
  for i, s in src:
    dest[i] = s
```

---

### `objectInit`

```nim
proc objectInit(dest: pointer, typ: PNimType)
```

**What it does:**  
Initialises a freshly allocated object or array to its zero/default state, with one important extra step: it writes the correct **type pointer** into the object header. This type pointer is what powers Nim's `of` operator (runtime type checks) and dynamic dispatch.

It handles:
- `tyObject` — writes the type info pointer into the object header, then recursively initialises all fields via `objectInitAux`
- `tyTuple` — recursively initialises all fields
- `tyArray` — initialises each element recursively
- All other types — nothing to do (zero memory is already correct)

**Why it matters:**  
Simply zeroing memory is not enough for objects. A zeroed object header means "no type" — any subsequent `of` check or method dispatch would crash or give wrong results. `objectInit` ensures the object is in a valid, typed state from the very first moment.

**Example (transparent to Nim programmer, but essential):**
```nim
type Animal = ref object of RootObj
  name: string

type Dog = ref object of Animal
  breed: string

var d = new Dog
# The `new` call ends with objectInit, which writes Dog's type pointer
# into the header so that `d of Dog` returns true immediately
echo d of Dog     # true
echo d of Animal  # true (Dog inherits from Animal)
```

---

### `genericReset`

```nim
proc genericReset(dest: pointer, mt: PNimType)
```

**What it does:**  
Resets the value at `dest` to its zero/default state, correctly cleaning up any managed resources it holds. This is what Nim's `reset(x)` standard library procedure calls internally.

The reset logic per type kind:
- `tyRef` — calls `unsureAsgnRef(dest, nil)`, telling the GC the reference is gone
- `tyString` — frees the heap buffer (nimSeqsV2 path) or nil-assigns the GC pointer
- `tySequence` — frees the payload (nimSeqsV2 path) or nil-assigns the GC pointer
- `tyTuple` — recursively resets each field via `genericResetAux`
- `tyObject` — recursively resets each field, then **also zeroes the type pointer** in the object header (crucial for variant objects: resetting the discriminant must invalidate the old branch)
- `tyArray` — resets each element
- Everything else — `zeroMem` (raw zero fill)

**Why the type pointer must be zeroed for objects:**  
Variant (`case`) objects in Nim can change their active branch by reassigning the discriminant field. Before a new branch can be safely written, the fields of the old branch must be reset. `genericReset` handles this by zeroing the type pointer along with the data, leaving the object in a pristine, untyped state ready for re-initialisation.

**Example:**
```nim
var s = "hello, world"
reset(s)        # calls genericReset; heap buffer is freed
echo s == ""    # true — s is now the default empty string

var nums = @[1, 2, 3]
reset(nums)
echo nums.len   # 0 — sequence payload freed, len zeroed
```

---

### `FieldDiscriminantCheck`

```nim
proc FieldDiscriminantCheck(oldDiscVal, newDiscVal: int,
                            a: ptr array[0x7fff, ptr TNimNode],
                            L: int)
```

**What it does:**  
Enforces Nim's safety rule for **variant objects**: you may not assign a new discriminant value that would switch the object to a different `case` branch if the current branch is already occupied by non-zero data. Violating this rule would create a type confusion — fields from the old branch would be reinterpreted as fields of the new branch.

The check compares which branch corresponds to `oldDiscVal` and which to `newDiscVal` using `selectBranch`. If they differ, a `FieldDefect` is raised at runtime.

**The `nimOldCaseObjects` escape hatch:**  
With `-d:nimOldCaseObjects`, the check is slightly more lenient: it only raises if `oldDiscVal != 0` (non-zero old value) *and* the branch changes. Without that flag (modern Nim), changing branches always raises, even from zero — unless you first `reset` the object.

**Why it matters:**  
This guard is what makes variant objects memory-safe. Without it, code like the following would silently corrupt memory:

```nim
type Shape = object
  case kind: ShapeKind
  of Circle:
    radius: float     # 8 bytes at some offset
  of Rect:
    width, height: float  # different fields, same memory region

var s = Shape(kind: Circle, radius: 3.14)
s.kind = Rect  # ← FieldDiscriminantCheck fires here!
               # radius bits would be reinterpreted as width/height
```

**Correct pattern — reset before switching:**
```nim
reset(s)                              # clears radius, zeroes type info
s = Shape(kind: Rect, width: 10, height: 5)  # safe re-initialisation
```

---

## Internal Helpers (Not Exported, Essential to Understanding)

| Helper | Role |
|---|---|
| `genericAssignAux(dest, src, mt, shallow)` | Central recursive dispatcher: chooses copy strategy per type kind |
| `genericAssignAux(dest, src, n, shallow)` | Node-level dispatcher: walks `TNimNode` tree (slots, lists, case branches) |
| `deepSeqAssignImpl` | Template that allocates a new sequence payload and copies elements; avoids code duplication between the two sequence variants |
| `genericResetAux` | Node-level reset dispatcher mirroring `genericAssignAux` for the reset path |
| `objectInitAux` | Node-level init dispatcher mirroring `genericAssignAux` for the init path |
| `selectBranch(discVal, L, a)` | Given a discriminant value and the branch table, returns the active `TNimNode` branch (or the `else` branch) |

---

## Key Design Principles

**RTTI-driven dispatch:** All procedures accept `PNimType` (a pointer to a runtime type descriptor) and switch on `mt.kind`. The compiler generates and links these type descriptors for every type in your program. No generics, no templates at the call site — just a type-tagged pointer and a big `case` statement.

**GC cooperation via `unsureAsgnRef`:** Whenever a `ref` is stored into a new location, the GC must be notified so it can update its reference-counting or mark-and-sweep bookkeeping. `unsureAsgnRef` is the single safe write barrier for this purpose. The "unsure" means it handles both heap-allocated and stack-allocated destinations.

**Shallow flag in sequence headers:** The `seqShallowFlag` bit in the `reserved` field of a sequence header tells the runtime "this payload is not owned — don't copy it, don't free it." String literals use this so they can be cheaply assigned without heap allocation.

**Object header type pointer:** Every `object` value (heap or stack) begins with a pointer to its `PNimType` / `PNimTypeV2` descriptor. This is what makes `x of SomeType`, `method` dispatch, and `reset` all work correctly. Both `objectInit` and `genericReset` maintain this invariant.

**Variant object safety:** The `FieldDiscriminantCheck` + `genericResetAux` combination provides memory-safe discriminant reassignment. The rule is simple: reset first, then assign to the new branch. Nim enforces this at runtime (and the compiler can often detect violations statically too).
