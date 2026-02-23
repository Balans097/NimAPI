# Nim `typetraits` Module — Complete Reference

> **Module purpose:** Compile-time reflection for types. Every exported symbol in this module operates at compile time: it accepts type descriptors or typed expressions and returns facts about types — their names, arities, base types, generic parameters, and capabilities. Nothing in this module produces runtime overhead.
>
> **Import:** `import std/typetraits`
>
> **API stability:** Marked *unstable* — the public interface may change between Nim releases.
>
> **Core concept:** Most procs here accept `typedesc` — Nim's way of passing a type as a value. You write `name(int)` or `int.name` to query the type `int` itself, not an instance of it.

---

## Table of Contents

1. [Type aliases for enum classification](#type-aliases-for-enum-classification)
   - [`HoleyEnum`](#holeyenum)
   - [`OrdinalEnum`](#ordinalenum)
2. [Basic type introspection](#basic-type-introspection)
   - [`name`](#name)
   - [`arity`](#arity)
3. [Generic type introspection](#generic-type-introspection)
   - [`genericHead`](#generichead)
   - [`stripGenericParams`](#stripgenericparams)
   - [`genericParams`](#genericparams)
   - [`StaticParam`](#staticparam)
4. [Type capability queries](#type-capability-queries)
   - [`supportsCopyMem`](#supportscopymem)
   - [`hasDefaultValue`](#hasdefaultvalue)
5. [Structural type queries](#structural-type-queries)
   - [`isNamedTuple`](#isnamedtuple)
   - [`tupleLen` (type)](#tuplelen-type)
   - [`tupleLen` (value)](#tuplelen-value)
   - [`get`](#get)
   - [`enumLen`](#enumlen)
   - [`elementType`](#elementtype)
6. [Pointer and reference types](#pointer-and-reference-types)
   - [`pointerBase`](#pointerbase)
7. [Range types](#range-types)
   - [`rangeBase` (type)](#rangebase-type)
   - [`rangeBase` (value)](#rangebase-value)
8. [Distinct types](#distinct-types)
   - [`distinctBase` (type)](#distinctbase-type)
   - [`distinctBase` (value)](#distinctbase-value)
9. [Integer sign conversion](#integer-sign-conversion)
   - [`toUnsigned`](#tounsigned)
   - [`toSigned`](#tosigned)
10. [Closure inspection](#closure-inspection)
    - [`hasClosure`](#hasclosure)

---

## Type aliases for enum classification

### `HoleyEnum`

```nim
type HoleyEnum* = (not Ordinal) and enum
```

A **type class** that matches enums with *holes* — enum types where the underlying integer values are not contiguous (i.e. there are gaps between ordinal values). An enum is holey when you explicitly assign integer values that skip numbers.

Holey enums are **not** `Ordinal` because you cannot safely do arithmetic like `succ`/`pred` on them without jumping over undefined values.

```nim
type
  Status = enum
    Pending = 0
    Active  = 10   # gap: 1..9 are undefined → holey!
    Closed  = 20

  Priority = enum
    Low, Medium, High   # 0, 1, 2 — contiguous, NOT holey

assert Status   is HoleyEnum
assert Priority is OrdinalEnum
assert Priority isnot HoleyEnum
```

Use `HoleyEnum` in `when` branches or generic constraints to write code that handles both kinds of enums correctly, especially when serialising or iterating.

---

### `OrdinalEnum`

```nim
type OrdinalEnum* = Ordinal and enum
```

A **type class** that matches enums whose values are *contiguous* (no holes). These are ordinary `Ordinal` types, meaning `ord`, `succ`, `pred`, and `low`/`high` all work as expected and safe iteration is possible.

```nim
type Color = enum Red, Green, Blue   # 0, 1, 2 — contiguous

assert Color is OrdinalEnum
assert Color isnot HoleyEnum
assert int isnot OrdinalEnum   # int is Ordinal but not enum
```

---

## Basic type introspection

### `name`

```nim
proc name*(t: typedesc): string {.magic: "TypeTrait".}
```

**Returns the name of the type `t` as a string.** For generic types the full instantiated form is returned. This is an alias for `system.$` on a `typedesc` (available since Nim 0.20), so `name(int)` and `$int` are equivalent.

The string is the **canonical Nim name** of the type, not necessarily what you declared — for example, type aliases may be resolved.

```nim
import std/typetraits

doAssert name(int)          == "int"
doAssert name(string)       == "string"
doAssert name(seq[string])  == "seq[string]"
doAssert name(array[3, float]) == "array[0..2, float]"

type MyInt = distinct int
doAssert name(MyInt) == "MyInt"
```

**Practical use:** Generating error messages, debug output, or reflection-based serialisation where you need the type name at runtime.

---

### `arity`

```nim
proc arity*(t: typedesc): int {.magic: "TypeTrait".}
```

**Returns the number of generic (type) parameters** of a type — its *arity* in the mathematical sense. A plain type with no generic parameters has arity 0. A tuple's arity equals the number of its fields. A generic type's arity equals the number of type arguments it was instantiated with.

```nim
import std/typetraits

doAssert arity(int)                       == 0  # no generic params
doAssert arity(seq[string])               == 1  # one: the element type
doAssert arity(array[3, int])             == 2  # two: index type + element type
doAssert arity((int, int, float, string)) == 4  # four fields in the tuple
doAssert arity(Table[string, int])        == 2  # key + value
```

**Practical use:** Useful in generic procs that need to branch differently depending on how many type parameters a type carries.

---

## Generic type introspection

### `genericHead`

```nim
proc genericHead*(t: typedesc): typedesc {.magic: "TypeTrait".}
```

**Accepts an instantiated generic type and returns its uninstantiated (bare) constructor.** Given `Foo[int]`, it returns `Foo` — the generic type without any arguments applied. This is the *generic head* in the sense of type theory.

A **compile-time error** is produced if the supplied type is not generic — use `stripGenericParams` if you need a safe version that handles non-generics.

```nim
import std/typetraits

type Foo[T] = object

doAssert genericHead(Foo[int])        is Foo
doAssert genericHead(Foo[seq[string]]) is Foo

# Using it to write a concept that requires a generic type:
type Generic = concept f
  type _ = genericHead(typeof(f))

proc identity(a: Generic): typeof(a) = a
```

**Practical use:** Writing concepts or macros that need to reason about the bare generic container, independent of its type arguments. For example, checking "is this value in any `seq`?" regardless of the element type.

---

### `stripGenericParams`

```nim
proc stripGenericParams*(t: typedesc): typedesc {.magic: "TypeTrait".}
```

**Like `genericHead`, but safe for non-generic types.** If `t` is a generic instantiation, the bare generic type is returned. If `t` is not generic at all, it is returned unchanged. This is the "forgiving" version of `genericHead`.

```nim
import std/typetraits

type Foo[T] = object

doAssert stripGenericParams(Foo[string]) is Foo   # generic → stripped
doAssert stripGenericParams(int)         is int   # non-generic → unchanged
doAssert stripGenericParams(string)      is string
```

**Use instead of `genericHead`** when the input type might or might not be generic and you simply want the bare form in all cases without worrying about compile errors.

---

### `genericParams`

```nim
template genericParams*(T: typedesc): untyped
```

*(Available since Nim 1.1)*

**Returns the type arguments of a generic instantiation as a tuple of types.** Given `Foo[float, string]`, it returns a tuple type `(float, string)` that you can inspect with `is`, `get`, and other type-level operations.

Static (compile-time constant) parameters are wrapped in `StaticParam[value]` so they can be retrieved as a type-level value.

```nim
import std/typetraits

type Foo[T1, T2] = object

doAssert genericParams(Foo[float, string]) is (float, string)
doAssert genericParams(seq[int]).get(0) is int

type Bar[N: static float, T] = object
doAssert genericParams(Bar[1.0, string]) is (StaticParam[1.0], string)
doAssert genericParams(Bar[1.0, string]).get(0).value == 1.0

# Nested:
doAssert genericParams(seq[Bar[2.0, string]]).get(0) is Bar[2.0, string]
```

> **Array caveat:** For the built-in `array` type, the size parameter is always normalised to a `range` after type checking, so `genericParams(array[10, int])` gives `(StaticParam[10], int)` in a type expression, but `genericParams(typeof(a))` where `a: array[10, int]` gives `(range[0..9], int)`.

---

### `StaticParam`

```nim
type StaticParam*[value: static type] = object
```

A helper type used by `genericParams` to wrap **static (compile-time) generic parameters** so they can be represented in a tuple of types. You don't construct `StaticParam` yourself — you receive it as part of the tuple returned by `genericParams` and read its `.value` field.

```nim
import std/typetraits

type Arr = array[5, int]
let p = genericParams(Arr).get(0)   # StaticParam[5]
# p.value would be 5 — but note: p is a type, not a value,
# so you access .value in a type context:
doAssert genericParams(Arr).get(0) is StaticParam[5]
```

---

## Type capability queries

### `supportsCopyMem`

```nim
proc supportsCopyMem*(t: typedesc): bool {.magic: "TypeTrait".}
```

**Returns `true` if values of type `t` can be safely duplicated by copying raw bytes** (i.e. via `copyMem`). Types for which this is true are called *plain old data* (POD) or *blobs* in other languages. They contain no managed fields (no `string`, `seq`, `ref`, GC-tracked pointers, etc.) and no custom destructors.

```nim
import std/typetraits

doAssert supportsCopyMem(int)       == true
doAssert supportsCopyMem(float)     == true
doAssert supportsCopyMem(char)      == true
doAssert supportsCopyMem(string)    == false  # ref-counted data inside
doAssert supportsCopyMem(seq[int])  == false  # heap allocation inside

type Point = object
  x, y: float
doAssert supportsCopyMem(Point) == true   # plain struct, no managed fields
```

**Practical use:** Low-level serialisation, writing to memory-mapped files, zero-copy network buffers, or passing data to C libraries — any context where raw byte copies must be valid.

---

### `hasDefaultValue`

```nim
proc hasDefaultValue*(t: typedesc): bool {.magic: "TypeTrait".}
```

**Returns `true` if the type `t` can be default-initialised** (i.e. `default(t)` or `var x: t` is valid without providing an explicit initial value). Returns `false` for:
- `var T` (mutable reference — requires explicit binding)
- `not nil` constrained ref types (nil is not a valid default)
- Types with fields marked `{.requiresInit.}`

```nim
import std/typetraits

{.experimental: "strictNotNil".}
type
  NilableObject = ref object
    a: int
  Object = NilableObject not nil   # cannot be nil → no default

assert hasDefaultValue(int)           == true
assert hasDefaultValue(string)        == true
assert hasDefaultValue(NilableObject) == true   # nil is valid default here
assert hasDefaultValue(Object)        == false  # not nil → no nil default
assert hasDefaultValue(var string)    == false  # var ref has no default

type RequiresInit[T] = object
  a {.requiresInit.}: T
assert hasDefaultValue(RequiresInit[int]) == false
```

**Practical use:** Generic code that conditionally zero-initialises a value, or generic containers that need to know whether they can safely create a "blank" element.

---

## Structural type queries

### `isNamedTuple`

```nim
proc isNamedTuple*(T: typedesc): bool {.magic: "TypeTrait".}
```

**Returns `true` only for named tuples** — tuples where every field has an explicit name (declared with `tuple[name: Type, …]` syntax). Returns `false` for anonymous tuples `(Type, Type, …)` and for all non-tuple types.

```nim
import std/typetraits

doAssert not isNamedTuple(int)
doAssert not isNamedTuple((string, int))              # anonymous tuple
doAssert     isNamedTuple(tuple[name: string, age: int])  # named tuple

type Person = tuple[name: string, age: int]
doAssert isNamedTuple(Person)
```

**Practical use:** Generic serialisers that can emit field names for named tuples but must fall back to positional access for anonymous ones.

---

### `tupleLen` (type)

```nim
proc tupleLen*(T: typedesc[tuple]): int {.magic: "TypeTrait".}
```

*(Available since Nim 1.1)*

**Returns the number of fields** in the tuple type `T`. Works for both named and anonymous tuples. Accepts only tuple types — passing a non-tuple produces a compile error.

```nim
import std/typetraits

doAssert tupleLen((int, int, float, string))       == 4
doAssert tupleLen(tuple[name: string, age: int])   == 2
doAssert tupleLen(tuple[])                         == 0
```

---

### `tupleLen` (value)

```nim
template tupleLen*(t: tuple): int
```

*(Available since Nim 1.1)*

**Overload of `tupleLen` that accepts a tuple value** rather than a type. Simply delegates to `tupleLen(typeof(t))`. Convenient when you have a value at hand and don't want to write `tupleLen(typeof(myTuple))`.

```nim
import std/typetraits

let p = (1, "hello")
doAssert tupleLen(p) == 2

let person = (name: "Alice", age: 30)
doAssert tupleLen(person) == 2
```

---

### `get`

```nim
template get*(T: typedesc[tuple], i: static int): untyped
```

*(Available since Nim 1.1)*

**Returns the type of the `i`-th field** (zero-based) of tuple type `T`. This is a type-level operation — it gives you the *type*, not a value. Used together with `genericParams` to inspect individual generic arguments by position.

```nim
import std/typetraits

doAssert get((int, float, string), 0) is int
doAssert get((int, float, string), 1) is float
doAssert get((int, float, string), 2) is string

# Most useful with genericParams:
type Pair[A, B] = object
doAssert genericParams(Pair[int, string]).get(0) is int
doAssert genericParams(Pair[int, string]).get(1) is string
```

---

### `enumLen`

```nim
macro enumLen*(T: typedesc[enum]): int
```

**Returns the number of variants** in the enum type `T` as a compile-time integer. Works for both ordinal and holey enums. The result is a `static int` — it can be used in array sizes, static assertions, and other compile-time contexts.

```nim
import std/typetraits

type Direction = enum North, South, East, West

doAssert Direction.enumLen == 4

type Status = enum
  Pending = 0, Active = 10, Closed = 20   # holey enum

doAssert Status.enumLen == 3   # 3 variants, regardless of gaps

# Use the count to size a lookup table:
var names: array[Direction.enumLen, string]
```

---

### `elementType`

```nim
template elementType*(a: untyped): typedesc
```

*(Available since Nim 1.3.5)*

**Returns the element type of any iterable `a`** — anything you can use in a `for` loop: sequences, arrays, strings, custom iterators, ranges, sets, and more. This is a pure compile-time operation; `a` is never actually iterated.

```nim
import std/typetraits

doAssert elementType(@[1, 2, 3])    is int
doAssert elementType("hello")       is char
doAssert elementType({1, 2, 3})     is int   # set

iterator countdown3: int =
  yield 3; yield 2; yield 1

doAssert elementType(countdown3()) is int

# Useful in generic procs:
proc firstElement[C](c: C): elementType(c) =
  for x in c: return x
```

**Practical use:** Writing generic algorithms that need to know the element type of a container or iterator without requiring an explicit type parameter.

---

## Pointer and reference types

### `pointerBase`

```nim
template pointerBase*[T](_: typedesc[ptr T | ref T]): typedesc
```

**Returns the pointee type `T`** for `ptr T` and `ref T` types — one level of indirection removed. Does **not** recurse: `(ref ptr int).pointerBase` gives `ptr int`, not `int`.

```nim
import std/typetraits

assert (ref int).pointerBase    is int
assert (ptr float).pointerBase  is float

type A = ptr seq[float]
assert A.pointerBase is seq[float]

assert (ref A).pointerBase is A         # one level only: A = ptr seq[float]
assert (ref A).pointerBase isnot seq[float]

# Works with variable addresses too:
var s = "hello"
assert s[0].addr.typeof.pointerBase is char
```

**Practical use:** Generic code over pointer types that needs to operate on or declare the dereferenced type — for example, allocators, wrappers, or FFI bindings.

---

## Range types

### `rangeBase` (type)

```nim
proc rangeBase*(T: typedesc[range]): typedesc {.magic: "TypeTrait".}
```

**Returns the underlying base type of a `range` type.** A `range[lo..hi]` constrains values to a subset of another type; `rangeBase` unwraps that constraint and gives you the raw type underneath.

Only accepted for `range` types — passing a non-range produces a compile error.

```nim
import std/typetraits

type MyRange = range[0..5]
type MyEnum  = enum a, b, c
type EnumRange = range[b..c]

doAssert rangeBase(MyRange)   is int     # range of int → int
doAssert rangeBase(EnumRange) is MyEnum  # range of MyEnum → MyEnum
doAssert rangeBase(range['a'..'z']) is char
```

---

### `rangeBase` (value)

```nim
template rangeBase*[T: range](a: T): untyped
```

**Overload of `rangeBase` for values.** Converts a value of a range type to a value of the base type. Equivalent to `rangeBase(typeof(a))(a)` — a type-level cast that strips the range constraint.

```nim
import std/typetraits

type MyRange = range[0..5]
let x = MyRange(3)
doAssert rangeBase(x) is int
doAssert rangeBase(x) == 3          # same numeric value, base type

type MyEnum = enum a, b, c
type EnumRange = range[b..c]
let e: EnumRange = c
doAssert rangeBase(e) is MyEnum
doAssert rangeBase(e) == c
```

**Practical use:** Converting a range-typed value to its base for use in arithmetic, comparisons with plain-typed variables, or passing to procedures that don't accept the range type.

---

## Distinct types

### `distinctBase` (type)

```nim
proc distinctBase*(T: typedesc, recursive: static bool = true): typedesc {.magic: "TypeTrait".}
```

**Returns the underlying type beneath one or more `distinct` layers.** By default (`recursive = true`) it peels all layers recursively until a non-distinct type is reached. With `recursive = false` it peels exactly one layer.

For non-distinct types, the type itself is returned unchanged.

```nim
import std/typetraits

type MyInt      = distinct int
type MyOtherInt = distinct MyInt   # two layers deep

doAssert distinctBase(MyInt)              is int    # one layer → int
doAssert distinctBase(MyOtherInt)         is int    # two layers → int (recursive)
doAssert distinctBase(MyOtherInt, false)  is MyInt  # one layer only
doAssert distinctBase(int)                is int    # non-distinct → unchanged
```

**Practical use:** Generic utilities (parsers, serialisers, formatters) that receive a `distinct` type but need to operate on the underlying integer, string, etc.

---

### `distinctBase` (value)

```nim
template distinctBase*[T](a: T, recursive: static bool = true): untyped
```

*(Available since Nim 1.1)*

**Overload of `distinctBase` for values.** Strips the `distinct` wrapper from a value and returns the underlying value with the base type. If `T` is not `distinct`, the value is returned unchanged (without a self-conversion warning).

```nim
import std/typetraits

type MyInt      = distinct int
type MyOtherInt = distinct MyInt

doAssert 12.MyInt.distinctBase      == 12           # value: 12, type: int
doAssert 12.MyOtherInt.distinctBase == 12           # peels both layers
doAssert 12.MyOtherInt.distinctBase(false) is MyInt # one layer
doAssert 12.distinctBase            == 12           # non-distinct → unchanged
```

---

## Integer sign conversion

### `toUnsigned`

```nim
template toUnsigned*(T: typedesc[SomeInteger and not range]): untyped
```

**Returns the unsigned integer type with the same bit-width as `T`.** If `T` is already unsigned, it is returned unchanged. Range types are not supported and produce a compile error.

| Input type | Result |
|---|---|
| `int8` | `uint8` |
| `int16` | `uint16` |
| `int32` | `uint32` |
| `int64` | `uint64` |
| `int` | `uint` |
| `uint`, `uint8`, … | unchanged |

```nim
import std/typetraits

assert int8.toUnsigned   is uint8
assert int16.toUnsigned  is uint16
assert int.toUnsigned    is uint
assert uint.toUnsigned   is uint    # already unsigned — no change

# Not allowed for range types:
assert not compiles(toUnsigned(range[0..7]))
```

**Practical use:** Generic code that needs to reinterpret an integer as its unsigned counterpart without changing bit width — for example, bit manipulation utilities or low-level network encoding.

---

### `toSigned`

```nim
template toSigned*(T: typedesc[SomeInteger and not range]): untyped
```

**Returns the signed integer type with the same bit-width as `T`.** If `T` is already signed, it is returned unchanged. Range types are not supported.

| Input type | Result |
|---|---|
| `uint8` | `int8` |
| `uint16` | `int16` |
| `uint32` | `int32` |
| `uint64` | `int64` |
| `uint` | `int` |
| `int`, `int8`, … | unchanged |

```nim
import std/typetraits

assert uint8.toSigned   is int8
assert uint16.toSigned  is int16
assert uint.toSigned    is int
assert int8.toSigned    is int8    # already signed — no change

assert not compiles(toSigned(range[0..7]))
```

---

## Closure inspection

### `hasClosure`

```nim
proc hasClosure*(fn: NimNode): bool {.since: (1, 5, 1).}
```

*(Available since Nim 1.5.1)*

**Returns `true` if the procedure represented by the `NimNode` `fn` has a closure environment** — i.e. it captures variables from an enclosing scope. This proc is meant to be called from **macros** that accept `typed` arguments, because `fn` must be a fully resolved `nnkSym` node (not an untyped AST fragment).

```nim
import std/typetraits, std/macros

macro checkClosure(fn: typed): bool =
  newLit(hasClosure(fn))

proc pureProc(x: int): int = x + 1

var captured = 10
let closureProc = proc(x: int): int = x + captured

assert not checkClosure(pureProc)      # no captures → no closure
assert     checkClosure(closureProc)   # captures `captured` → has closure
```

**Practical use:** Code generators and macros that need to decide whether a callable requires a hidden environment pointer — relevant when generating C FFI callbacks, interfacing with C libraries that expect raw function pointers, or optimising dispatch.

---

## Complete usage example

```nim
import std/typetraits

# --- Basic introspection ---
doAssert name(seq[int])      == "seq[int]"
doAssert arity(Table[string, int]) == 2

# --- Enum classification ---
type
  Sparse = enum s0 = 0, s1 = 100, s2 = 200  # holey
  Dense  = enum d0, d1, d2                   # ordinal

assert Sparse is HoleyEnum
assert Dense  is OrdinalEnum
assert Dense.enumLen == 3

# --- Generic type introspection ---
type Box[T] = object
  value: T

doAssert genericHead(Box[string]) is Box
doAssert genericParams(Box[float]).get(0) is float

# --- Capability queries ---
type RawPoint = object
  x, y: float32

doAssert supportsCopyMem(RawPoint)        # safe to memcpy
doAssert not supportsCopyMem(seq[int])    # heap-allocated

# --- Distinct type utilities ---
type Meters   = distinct float
type Seconds  = distinct float

proc speed(d: Meters, t: Seconds): float =
  distinctBase(d) / distinctBase(t)

doAssert speed(Meters(100.0), Seconds(10.0)) == 10.0

# --- Sign conversion ---
type Storage = int32
type UStorage = Storage.toUnsigned   # uint32

# --- Range / pointer base ---
type SmallInt = range[0..15]
doAssert rangeBase(SmallInt) is int

doAssert (ref string).pointerBase is string

# --- Structural tuple queries ---
type Record = tuple[name: string, score: int]

doAssert isNamedTuple(Record)
doAssert tupleLen(Record) == 2
doAssert get(Record, 0)   is string
doAssert get(Record, 1)   is int

# --- Element type ---
doAssert elementType(@[1.0, 2.0]) is float
```

---

*Reference covers Nim standard library `std/typetraits` as found in the module source.*
