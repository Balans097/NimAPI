# Nim `marshal` Module — Complete Reference

## Overview

The `marshal` module provides **serialization** (converting Nim data structures into a string) and **deserialization** (reconstructing Nim data structures from a string) using **JSON** as the wire format.

It works at both **runtime** and **compile-time** (`nimvm`), which is an unusual and powerful property. You can, for example, use `$$` and `to` inside macros or `static:` blocks.

### Key limitations to know upfront

- **Type information is not embedded in the output.** If an object's runtime type differs from its compile-time type (i.e. polymorphism via `ref` inheritance), only the compile-time fields will be serialized. See the example in the *Basic behaviour* section below.
- **Not supported on JavaScript or NimScript targets.** The module raises a compile-time error on those platforms; use alternative packages instead.
- Pointers, `proc` values, and `cstring` values are serialized as raw integer addresses, which are generally not portable across runs.

---

## Exported API

The module exports **four** public symbols:

| Symbol | Kind | Purpose |
|--------|------|---------|
| `load` | proc | Deserialize from a `Stream` into an existing variable |
| `store` | proc | Serialize a value into a `Stream` |
| `$$` | operator (proc) | Serialize a value to a `string` |
| `to` | proc | Deserialize a `string` into a new value of type `T` |

---

## `load`

```nim
proc load*[T](s: Stream, data: var T)
```

### What it does

Reads JSON text from the stream `s` and **populates** the variable `data` with the decoded values. The variable must already exist; `load` fills it in-place. Raises `IOError` if reading fails, and a parse error if the JSON does not match the expected structure for type `T`.

### When to use it

Use `load` when you already have a `Stream` — for example a file stream opened with `openFileStream`, or a `newStringStream` wrapping a string you received over a network. It is the stream-based counterpart of `to`.

### Parameters

| Parameter | Description |
|-----------|-------------|
| `s` | Any `Stream` (file, string, socket-backed, …) positioned at the start of valid JSON |
| `data` | A `var` variable of any supported Nim type; its fields are overwritten with the deserialized values |

### Example

```nim
import std/[marshal, streams]

var s = newStringStream("[1, 3, 5]")
var a: array[3, int]
load(s, a)
assert a == [1, 3, 5]
```

Reading an object from a JSON file on disk:

```nim
import std/[marshal, streams]

type Person = object
  name: string
  age:  int

var f = openFileStream("person.json")
var p: Person
load(f, p)
f.close()
echo p.name, " is ", p.age, " years old"
```

### Important notes

- The stream position is **not** reset automatically before reading. Make sure it points to the beginning of the JSON data (call `s.setPosition(0)` if needed).
- `data` is mutated; pass it as `var`.

---

## `store`

```nim
proc store*[T](s: Stream, data: sink T)
```

### What it does

Serializes `data` to JSON and **writes** the result into the stream `s`. It is the mirror image of `load`: whatever you `store`, you can later `load` back into a variable of the same type.

`data` is declared as `sink`, meaning ownership may be transferred for efficiency — do not rely on `data` being usable after the call if you are on ARC/ORC.

### When to use it

Use `store` when you need to write serialized data somewhere other than a plain string — into a file, a network buffer, or any custom stream.

### Parameters

| Parameter | Description |
|-----------|-------------|
| `s` | A writable `Stream` |
| `data` | The value to serialize; passed as `sink` (ownership may be consumed) |

### Example

```nim
import std/[marshal, streams]

var s = newStringStream("")
var a = [1, 3, 5]
store(s, a)
s.setPosition(0)
assert s.readAll() == "[1, 3, 5]"
```

Writing an object to a file:

```nim
import std/[marshal, streams]

type Config = object
  host: string
  port: int

let cfg = Config(host: "localhost", port: 8080)
var f = openFileStream("config.json", fmWrite)
store(f, cfg)
f.close()
# config.json now contains: {"host": "localhost", "port": 8080}
```

### Important notes

- After calling `store`, the stream position is **at the end** of the written data. Call `s.setPosition(0)` before reading back from the same stream.
- Because `data` is a `sink`, consider passing a copy if you need to keep using the original value under non-ARC GC modes.

---

## `$$` (double-dollar operator)

```nim
proc `$$`*[T](x: sink T): string
```

### What it does

Converts **any** Nim value `x` into its JSON string representation. This is the most convenient serialization entry point: no stream setup, just a function that returns a `string`.

It is the high-level, string-returning wrapper around `store`. Internally it creates a temporary `StringStream`, calls `storeAny`, and returns the accumulated string.

**Works at compile-time** (`static:` blocks, macros). The compile-time path is handled by the compiler's VM operations (`loadVM`), not the runtime stream path.

### Parameters

| Parameter | Description |
|-----------|-------------|
| `x` | Any Nim value to serialize; passed as `sink` |

### Return value

A `string` containing the JSON representation of `x`.

### Example — primitive types

```nim
import std/marshal

assert $$(42)          == "42"
assert $$(3.14)        == "3.14"
assert $$("hello")     == "\"hello\""
assert $$(true)        == "true"
```

### Example — object

```nim
import std/marshal

type Foo = object
  id:  int
  bar: string

let x = Foo(id: 1, bar: "baz")
let y = $$x
assert y == """{"id": 1, "bar": "baz"}"""
```

### Example — sequences and arrays

```nim
import std/marshal

let nums = @[10, 20, 30]
assert $$nums == "[10, 20, 30]"

let pair = (name: "Alice", score: 99)
assert $$pair == """{"Field0": "Alice", "Field1": 99}"""  # anonymous tuple fields
```

### Example — compile-time usage

```nim
import std/marshal

static:
  type Point = object
    x, y: int
  const p = Point(x: 3, y: 7)
  const s = $$p
  assert s == """{"x": 3, "y": 7}"""
```

### Polymorphism caveat

```nim
import std/marshal

type
  A = object of RootObj
  B = object of A
    f: int

let a: ref A = new(B)
assert $$a[] == "{}"   # compile-time type is A, so field f is invisible
```

The field `f` belongs to `B`, but the pointer is typed as `ref A`, so the marshaller sees only the `A` part of the object.

### Note on JSON module difference

`$$x` is **not** the same as `%x` from `std/json`. The `json` module's `%` operator produces a `JsonNode` tree and always embeds type metadata for discriminated unions. Use `$$` for fast round-trip serialization with `to`; use `%` / `jsonutils.toJson` when you need standard, human-friendly JSON.

---

## `to`

```nim
proc to*[T](data: string): T
```

### What it does

Parses a JSON `string` and **constructs** a new value of type `T`. This is the inverse of `$$`. The type `T` must be specified explicitly, either via the `[:T]` syntax or inferred from context.

Like `$$`, it works at **compile-time** as well as at runtime.

### Parameters

| Parameter | Description |
|-----------|-------------|
| `data` | A JSON string, typically produced by `$$` or `store` |

### Type parameter

| Parameter | Description |
|-----------|-------------|
| `T` | The target Nim type to decode into; must match the structure of the JSON |

### Return value

A freshly created value of type `T` with all fields populated from the JSON.

### Example — object round-trip

```nim
import std/marshal

type Foo = object
  id:  int
  bar: string

let json = """{"id": 1, "bar": "baz"}"""
let z = json.to[:Foo]

assert z.id  == 1
assert z.bar == "baz"
```

### Example — array and sequence

```nim
import std/marshal

let z = "[4, 5, 6]".to[:array[3, int]]
assert z == [4, 5, 6]

let s = "[1, 2, 3]".to[:seq[int]]
assert s == @[1, 2, 3]
```

### Example — enum

```nim
import std/marshal

type Direction = enum North, South, East, West

let d = to[Direction]("""\"North\"""")
assert d == North
```

### Example — nested objects

```nim
import std/marshal

type
  Address = object
    city: string
  Person = object
    name:    string
    address: Address

let json = """{"name": "Alice", "address": {"city": "Oslo"}}"""
let p = json.to[:Person]
assert p.address.city == "Oslo"
```

### Example — compile-time usage

```nim
import std/marshal

static:
  type Point = object
    x, y: int
  const p = """{"x": 10, "y": 20}""".to[:Point]
  assert p.x == 10
```

### Error handling

If the JSON structure does not match the expected type, a `JsonParsingError` (a subtype of `CatchableError`) is raised. Always wrap `to` in a `try/except` block in production code when the input is untrusted.

```nim
import std/marshal

try:
  let bad = "not json at all".to[:int]
except CatchableError as e:
  echo "Parse failed: ", e.msg
```

---

## Type support matrix

| Nim type | Serialized as |
|----------|--------------|
| `bool` | `true` / `false` |
| `char` (ASCII) | JSON string `"a"` |
| `char` (non-ASCII) | integer ordinal |
| `int`, `uint`, all variants | JSON integer |
| `float`, all variants | JSON float |
| `string` (valid UTF-8) | JSON string `"…"` |
| `string` (invalid UTF-8) | JSON array of byte ordinals |
| `array`, `seq` | JSON array `[…]` |
| `object`, `tuple` | JSON object `{…}` |
| `set` | JSON array of ordinals |
| `enum` | JSON string of the enum field name |
| `ref`, `ptr` (non-nil, first time) | `[address, value]` pair |
| `ref`, `ptr` (non-nil, repeated) | integer address |
| `ref`, `ptr` (nil) | `null` |
| `proc`, `pointer`, `cstring` | integer address |

---

## Full round-trip example

```nim
import std/marshal

type
  Tag  = enum tRed, tGreen, tBlue
  Item = object
    name:  string
    value: float
    tags:  seq[Tag]

let original = Item(name: "widget", value: 3.14, tags: @[tRed, tBlue])

# serialize
let json = $$original
echo json
# {"name": "widget", "value": 3.14, "tags": ["tRed", "tBlue"]}

# deserialize
let restored = json.to[:Item]
assert restored.name        == original.name
assert restored.value       == original.value
assert restored.tags        == original.tags
```

---

## See also

- [`std/json`](https://nim-lang.org/docs/json.html) — general-purpose JSON parsing and generation
- [`std/streams`](https://nim-lang.org/docs/streams.html) — stream types used by `load` and `store`
- [`std/jsonutils`](https://nim-lang.org/docs/jsonutils.html) — type-safe JSON helpers with hooks
