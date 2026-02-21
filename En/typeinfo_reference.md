# typeinfo.nim — Module Reference

> **Import:** `import std/typeinfo`  
> **Purpose:** Runtime type inspection and value manipulation for arbitrary Nim values through a uniform `Any` wrapper.

---

## Safety warning

> **This module is inherently unsafe.** `Any` stores a raw, untraced pointer to the wrapped value. The wrapper **must not outlive** the original value. Incorrect use leads to dangling pointers, memory corruption, and undefined behaviour.

For safer alternatives:
- Use the `macros` module and `getTypeImpl` for **compile-time** type inspection.
- Use generics for **type-safe** storage of arbitrary values at runtime.

---

## Architecture

The module's design is straightforward: a single `Any` struct holds a raw pointer to a value and a pointer to that value's runtime type descriptor (`PNimType`). All procedures operate on this pair — the type descriptor drives dispatch (which cast to apply, how large the value is, which fields exist) and the value pointer provides the actual data to read or write.

```
var myInt = 42
var a = myInt.toAny        # a.value → &myInt
                           # a.rawType → runtime descriptor for `int`
echo a.kind                # → akInt
echo a.getInt              # → 42
a.setBiggestInt(99)
echo myInt                 # → 99   (the original was modified!)
```

---

## Types

### `AnyKind`

```nim
type AnyKind* = enum
  akNone, akBool, akChar, akEnum, akArray, akObject, akTuple,
  akSet, akRange, akPtr, akRef, akSequence, akProc, akPointer,
  akString, akCString, akInt, akInt8, akInt16, akInt32, akInt64,
  akFloat, akFloat32, akFloat64, akFloat128,
  akUInt, akUInt8, akUInt16, akUInt32, akUInt64
```

Discriminates what Nim type an `Any` value represents. `akNone` indicates an invalid or uninitialised wrapper. The values correspond directly to Nim's internal type kind identifiers.

---

### `Any`

```nim
type Any* = object
  value: pointer        # raw pointer to the wrapped value
  rawTypePtr: pointer   # pointer to the runtime type descriptor
```

The universal wrapper. Two critical lifetime rules:

1. **`Any` must not outlive the variable it wraps.** It holds a raw pointer; when the source goes out of scope, the pointer dangles.
2. **Default assignment is a shallow copy** — both the original and the copy point to the same memory. To deep-copy the underlying value, use `assign`.

---

## Creating an `Any`

### `toAny`

```nim
proc toAny*[T](x: var T): Any
```

**What it does.** Wraps `x` in an `Any`. Captures the address of `x` directly — no copying occurs. The `Any` and `x` share the same memory, so mutations through the `Any` wrapper affect `x` and vice versa.

The parameter must be `var` because the wrapper may be used to modify the value. Pass a `let` or a literal and you will get a compiler error.

```nim
var count = 0
var a = count.toAny
assert a.kind == akInt
a.setBiggestInt(7)
assert count == 7      # original was modified through the wrapper
```

---

## Type inspection

### `kind`

```nim
proc kind*(x: Any): AnyKind
```

**What it does.** Returns the `AnyKind` of the wrapped value. The first thing you should check before calling any typed accessor — calling `getInt` on an `akFloat` value causes an assertion failure.

```nim
var s = "hello"
var a = s.toAny
assert a.kind == akString
```

---

### `size`

```nim
proc size*(x: Any): int
```

**What it does.** Returns the size in bytes of the wrapped type, as known to the runtime. Equivalent to `sizeof(T)` for the wrapped type.

---

### `baseTypeKind`

```nim
proc baseTypeKind*(x: Any): AnyKind
```

**What it does.** Returns the `AnyKind` of `x`'s **base type**. Relevant for containers and derived types: for a `seq[int]`, this returns `akInt`; for a `ptr T`, it returns the kind of `T`. Returns `akNone` if the type has no base type.

---

### `baseTypeSize`

```nim
proc baseTypeSize*(x: Any): int
```

**What it does.** Returns the size in bytes of the base type. Returns `0` if there is no base type.

---

## Lifetime and memory management

### `invokeNew`

```nim
proc invokeNew*(x: Any)
```

**What it does.** Performs `new(x)` at runtime — allocates a fresh heap object and stores its address through the `Any` wrapper. `x` must represent a `ref` type (`x.kind == akRef`). After this call, the `ref` variable that `x` wraps points to a newly allocated object of the appropriate type.

```nim
type Node = ref object
  val: int
var n: Node
var a = n.toAny
a.invokeNew           # equivalent to: new(n)
assert not n.isNil
```

---

### `invokeNewSeq`

```nim
proc invokeNewSeq*(x: Any, len: int)
```

**What it does.** Performs `newSeq(x, len)` at runtime — allocates a new sequence of `len` elements and stores it in the wrapped variable. `x` must represent a sequence (`x.kind == akSequence`). Any previous sequence value is replaced.

```nim
var s: seq[int]
var a = s.toAny
a.invokeNewSeq(5)     # equivalent to: newSeq(s, 5)
assert s.len == 5
```

---

### `extendSeq`

```nim
proc extendSeq*(x: Any)
```

**What it does.** Appends one zero-initialised element to the sequence wrapped by `x`. Equivalent to `s.setLen(s.len + 1)`. `x` must represent a sequence. The new element is zeroed — you will need to set it explicitly via `x[x.len - 1] = value`.

```nim
var s = @[1, 2, 3]
var a = s.toAny
a.extendSeq           # s is now @[1, 2, 3, 0]
assert a.len == 4
```

---

### `setObjectRuntimeType`

```nim
proc setObjectRuntimeType*(x: Any)
```

**What it does.** Initialises the runtime type field of an object. This must be called after manually allocating memory for an object that participates in method dispatch or has virtual methods, before any other operations are performed on it. `x` must represent an `object` type.

---

## Accessing elements of containers

### `[]` (index — read)

```nim
proc `[]`*(x: Any, i: int): Any
```

**What it does.** Returns an `Any` wrapping the `i`-th element of an array or sequence. The returned wrapper points into the *interior* of the original container — modifying it modifies the original. Raises `IndexDefect` if `i` is out of bounds. Raises `ValueError` if the sequence is nil.

```nim
var a = @[10, 20, 30]
var x = a.toAny
assert x[1].getInt == 20
```

---

### `[]=` (index — write)

```nim
proc `[]=`*(x: Any, i: int, y: Any)
```

**What it does.** Sets the `i`-th element of an array or sequence. The element type of `y` must match the element type of `x` (enforced by assertion). Raises `IndexDefect` on out-of-bounds access.

```nim
var a = @[10, 20, 30]
var x = a.toAny
var v = 99
x[1] = v.toAny        # a is now @[10, 99, 30]
```

---

### `len`

```nim
proc len*(x: Any): int
```

**What it does.** Returns the number of elements in an array or sequence. For arrays, this is the static compile-time length stored in the type descriptor. For sequences, it is the current runtime length. Returns `0` for a nil sequence.

```nim
var s = @[1, 2, 3]
var a = s.toAny
assert a.len == 3
```

---

### `[]` (field by name — read)

```nim
proc `[]`*(x: Any, fieldName: string): Any
```

**What it does.** Returns an `Any` wrapping the named field of an object or tuple. Field lookup is style-insensitive (the same rules as Nim's `eqIdent`). For objects with inheritance, the search walks up the parent chain. Raises `ValueError` if the field name does not exist.

```nim
type Point = object
  x, y: int
var p = Point(x: 3, y: 4)
var a = p.toAny
assert a["x"].getInt == 3
```

---

### `[]=` (field by name — write)

```nim
proc `[]=`*(x: Any, fieldName: string, value: Any)
```

**What it does.** Sets the named field of an object or tuple to `value`. The type of `value` must exactly match the field's type (enforced by assertion). Raises `ValueError` for an unknown field name.

```nim
var p = Point(x: 3, y: 4)
var a = p.toAny
var newY = 99
a["y"] = newY.toAny   # p.y is now 99
```

---

### `[]` (dereference — read)

```nim
proc `[]`*(x: Any): Any
```

**What it does.** Dereferences a `ptr` or `ref` wrapped by `x`, returning an `Any` for the pointed-to value. `x` must be of kind `akPtr` or `akRef`.

```nim
var n = new int
n[] = 42
var a = n.toAny
assert a[].getInt == 42
```

---

### `[]=` (dereference — write)

```nim
proc `[]=`*(x, y: Any)
```

**What it does.** Writes through a `ptr` or `ref` wrapper. `x` must be a `akPtr`/`akRef`, and `y`'s type must match the pointed-to type.

---

### `base`

```nim
proc base*(x: Any): Any
```

**What it does.** Returns an `Any` with the same value pointer but with the type descriptor replaced by the base type. Useful for accessing the parent object's fields when working with inheritance. The pointer is shared — no data is copied.

---

## Reading scalar values

Each getter asserts the correct kind before reading. Use `kind` or `baseTypeKind` to check before calling.

### Signed integers

```nim
proc getInt*(x: Any): int
proc getInt8*(x: Any): int8
proc getInt16*(x: Any): int16
proc getInt32*(x: Any): int32
proc getInt64*(x: Any): int64
```

Read the exact integer type. Use these when you know the precise width. Use `getBiggestInt` when the width is not known in advance.

---

### `getBiggestInt`

```nim
proc getBiggestInt*(x: Any): BiggestInt
```

**What it does.** Reads any integer-like value and widens it to `BiggestInt` (which is `int64`). Works for `int` through `int64`, `uint` through `uint32`, `bool`, `char`, `enum`, and small bitsets. Sign-extends signed types; zero-extends unsigned ones.

```nim
var e: SomeEnum = SomeEnum.Val
var a = e.toAny
echo a.getBiggestInt   # ordinal value of Val
```

---

### `setBiggestInt`

```nim
proc setBiggestInt*(x: Any, y: BiggestInt)
```

**What it does.** The universal integer/enum/char/bool setter. Truncates or sign-extends `y` to fit the wrapped type's width. Works for the same set of types as `getBiggestInt`.

---

### Unsigned integers

```nim
proc getUInt*(x: Any): uint
proc getUInt8*(x: Any): uint8
proc getUInt16*(x: Any): uint16
proc getUInt32*(x: Any): uint32
proc getUInt64*(x: Any): uint64
```

Read the exact unsigned integer type.

---

### `getBiggestUint`

```nim
proc getBiggestUint*(x: Any): uint64
```

**What it does.** Reads any unsigned integer value and widens it to `uint64`.

---

### `setBiggestUint`

```nim
proc setBiggestUint*(x: Any; y: uint64)
```

**What it does.** Sets any unsigned integer value, truncating `y` to the correct width.

---

### `getChar`

```nim
proc getChar*(x: Any): char
```

**What it does.** Reads a `char` value. Handles range types by stripping the range wrapper first.

---

### `getBool`

```nim
proc getBool*(x: Any): bool
```

**What it does.** Reads a `bool` value. Handles range types.

---

### Floats

```nim
proc getFloat*(x: Any): float
proc getFloat32*(x: Any): float32
proc getFloat64*(x: Any): float64
```

Read the exact float type.

---

### `getBiggestFloat`

```nim
proc getBiggestFloat*(x: Any): BiggestFloat
```

**What it does.** Reads any float value, widening to `BiggestFloat` (`float64`).

---

### `setBiggestFloat`

```nim
proc setBiggestFloat*(x: Any, y: BiggestFloat)
```

**What it does.** Sets any float value, narrowing from `BiggestFloat` as needed.

---

### `getString`

```nim
proc getString*(x: Any): string
```

**What it does.** Reads the `string` value. Returns an empty string for a nil string under the older GC.

---

### `setString`

```nim
proc setString*(x: Any, y: string)
```

**What it does.** Replaces the `string` value of the wrapped variable with `y`.

---

### `getCString`

```nim
proc getCString*(x: Any): cstring
```

**What it does.** Reads a `cstring` value. The returned `cstring` points into external C memory; the caller is responsible for its lifetime.

---

### `getPointer`

```nim
proc getPointer*(x: Any): pointer
```

**What it does.** Reads the raw pointer from a value of kind `akString`, `akCString`, `akProc`, `akRef`, `akPtr`, `akPointer`, or `akSequence` (the pointer-like group). This gives you the actual machine pointer rather than the Nim-level value.

---

### `setPointer`

```nim
proc setPointer*(x: Any, y: pointer)
```

**What it does.** Writes a raw pointer into `x`. For non-`pointer` kinds, uses `genericAssign` to perform a proper GC-aware copy of the pointed-to object; for bare `pointer` types, writes the address directly.

---

## Enum operations

### `getEnumOrdinal`

```nim
proc getEnumOrdinal*(x: Any, name: string): int
```

**What it does.** Given an enum-typed `Any` (used only for its type information), looks up the ordinal value of the enum field named `name`. Comparison is style-insensitive. Returns `low(int)` if the name is not found. Handles both contiguous enums and "holey" enums (those with explicit ordinal values).

```nim
type Color = enum Red = 0, Green = 2, Blue = 4
var c: Color
var a = c.toAny
assert a.getEnumOrdinal("Green") == 2
assert a.getEnumOrdinal("purple") == low(int)
```

---

### `getEnumField` (by ordinal)

```nim
proc getEnumField*(x: Any, ordinalValue: int): string
```

**What it does.** The reverse of `getEnumOrdinal` — given an ordinal value, returns the corresponding enum field name as a string. If the ordinal has no matching name (e.g. for holey enums with gaps), returns the decimal representation of the ordinal.

```nim
var c: Color
var a = c.toAny
assert a.getEnumField(2) == "Green"
```

---

### `getEnumField` (current value)

```nim
proc getEnumField*(x: Any): string
```

**What it does.** Reads the current ordinal value of the enum wrapped in `x` and returns its name. Combines `getBiggestInt` and `getEnumField(ordinal)`.

```nim
var c = Green
var a = c.toAny
assert a.getEnumField == "Green"
```

---

## Set operations

### `elements` (iterator)

```nim
iterator elements*(x: Any): int
```

**What it does.** Iterates over every element present in the bitset wrapped by `x`. Each yielded value is the ordinal of a set member. `x` must represent a `set` type. Works for sets of any size (up to the compiler's maximum).

```nim
type Flags = set[char]
var f: Flags = {'a', 'c', 'z'}
var a = f.toAny
for ord in a.elements:
  write(stdout, chr(ord))   # prints: acz
```

---

### `inclSetElement`

```nim
proc inclSetElement*(x: Any, elem: int)
```

**What it does.** Includes the element with ordinal value `elem` in the bitset wrapped by `x`. Equivalent to `incl(s, elem)` at the bit level. `x` must represent a `set` type.

```nim
var f: Flags = {}
var a = f.toAny
a.inclSetElement(ord('b'))
assert 'b' in f
```

---

## Object and tuple field iteration

### `fields` (iterator)

```nim
iterator fields*(x: Any): tuple[name: string, any: Any]
```

**What it does.** Iterates over every **active** field of an object or tuple, yielding the field name and an `Any` pointing into that field. For objects with variant (case) fields, only the fields of the currently active branch are yielded. For inherited objects, fields from all ancestor types are included. The `any` values in the tuples point directly into the wrapped object — modifying them modifies the original.

```nim
type Person = object
  name: string
  age: int

var p = Person(name: "Alice", age: 30)
var a = p.toAny
for fieldName, fieldAny in a.fields:
  echo fieldName, ": ", fieldAny.kind
# name: akString
# age: akInt
```

---

## Deep assignment

### `assign`

```nim
proc assign*(x, y: Any)
```

**What it does.** Performs a **deep copy** of the value wrapped by `y` into the memory wrapped by `x`. Both must represent the same type (enforced by assertion).

This is the correct way to copy values through `Any` wrappers. The default Nim assignment on `Any` values copies only the pointer pair — both `Any` objects end up pointing to the same memory, which is almost never what you want when copying the actual data.

```nim
var src = @[1, 2, 3]
var dst: seq[int]
assign(dst.toAny, src.toAny)   # dst is now a deep copy of src
```

---

## Nil checking

### `isNil`

```nim
proc isNil*(x: Any): bool
```

**What it does.** Returns `true` if the pointer wrapped by `x` is nil. Valid for `akCString`, `akRef`, `akPtr`, `akPointer`, and `akProc`. Raises an assertion error for other kinds.

```nim
var p: ptr int = nil
var a = p.toAny
assert a.isNil
```

---

## Range types

### `skipRange`

```nim
proc skipRange*(x: Any): Any
```

**What it does.** For a range-typed `Any` (kind `akRange`), returns a new `Any` with the same value pointer but with the type descriptor replaced by the underlying base type, stripping the range constraint. Most scalar getters call this automatically, so you rarely need it directly.

```nim
type SmallInt = range[0..10]
var n: SmallInt = 5
var a = n.toAny
assert a.kind == akRange
var base = a.skipRange
assert base.kind == akInt
```

---

## Quick reference

| Goal | Procedure |
|---|---|
| Wrap a value | `toAny` |
| Get type kind | `kind` |
| Get base type kind | `baseTypeKind` |
| Allocate ref | `invokeNew` |
| Allocate seq | `invokeNewSeq` |
| Extend seq by 1 | `extendSeq` |
| Index array/seq | `x[i]` / `x[i] = y` |
| Access field | `x["field"]` / `x["field"] = y` |
| Dereference ptr/ref | `x[]` / `x[] = y` |
| Iterate fields | `fields` iterator |
| Iterate set members | `elements` iterator |
| Add to set | `inclSetElement` |
| Read any integer | `getBiggestInt` |
| Write any integer | `setBiggestInt` |
| Read any uint | `getBiggestUint` |
| Write any uint | `setBiggestUint` |
| Read any float | `getBiggestFloat` |
| Write any float | `setBiggestFloat` |
| Read/write string | `getString` / `setString` |
| Enum name → ordinal | `getEnumOrdinal` |
| Ordinal → enum name | `getEnumField(ordinal)` |
| Current enum name | `getEnumField()` |
| Raw pointer read | `getPointer` |
| Raw pointer write | `setPointer` |
| Deep copy | `assign` |
| Nil check | `isNil` |
| Strip range wrapper | `skipRange` |
