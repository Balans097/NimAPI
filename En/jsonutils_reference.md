# `jsonutils` Module Reference

> **Module:** `std/jsonutils`  
> **Language:** Nim  
> **Purpose:** Hookable, type-safe serialization and deserialization between arbitrary Nim types and JSON, without requiring manual converters for most common types.

---

## Core Idea

`jsonutils` is built around two central functions: `toJson` (Nim → JSON) and `jsonTo` / `fromJson` (JSON → Nim). They handle objects, tuples, sequences, sets, tables, options, enums, and primitive types automatically via compile-time type introspection.

The key differentiator from the basic `json` module is **extensibility via hooks**: if you define a `toJsonHook` or `fromJsonHook` for your type, those hooks are picked up automatically — no registration, no inheritance required. This allows you to customize serialization for third-party types without touching their source code.

---

## Exported Types

### `Joptions`

```nim
type Joptions* = object
  allowExtraKeys*: bool
  allowMissingKeys*: bool
```

A configuration object that controls how strictly `fromJson` / `jsonTo` enforce the shape of incoming JSON when deserializing into Nim objects. Both fields default to `false`, meaning strict matching by default.

**`allowExtraKeys`** — when `true`, the JSON object is allowed to contain keys that have no matching field in the Nim type. These extra keys are silently ignored. Useful when consuming JSON from external APIs that may evolve and add new fields you don't yet care about.

**`allowMissingKeys`** — when `true`, the Nim type is allowed to have fields for which no corresponding key exists in the JSON. Missing fields retain their default (zero) value. Useful for partial updates or optional data.

```nim
import std/[jsonutils, json]

type Config = object
  host: string
  port: int
  debug: bool

let j = parseJson("""{"host": "localhost", "port": 8080, "extra": "ignored"}""")

# Strict (default) — would fail because "extra" is unexpected:
# let c = jsonTo(j, Config)

# Lenient — extra JSON keys are allowed:
let c = jsonTo(j, Config, Joptions(allowExtraKeys: true))
assert c.host == "localhost"
```

---

### `EnumMode`

```nim
type EnumMode* = enum
  joptEnumOrd     # serialize as integer ordinal (default)
  joptEnumSymbol  # serialize as the internal symbol name
  joptEnumString  # serialize as the string representation ($)
```

Controls how enum values are written to JSON by `toJson`. This only affects serialization direction (Nim → JSON). Deserialization (`fromJson`) accepts both integers and strings and picks the right one automatically.

| Mode | Example output for `Color.Red` |
|---|---|
| `joptEnumOrd` | `0` |
| `joptEnumSymbol` | `"Red"` |
| `joptEnumString` | `"Red"` |

The difference between `joptEnumSymbol` and `joptEnumString` matters when `$` is overloaded for the enum — `joptEnumSymbol` always uses the declared identifier name, while `joptEnumString` uses whatever `$` produces.

```nim
import std/[jsonutils, json]

type Direction = enum dNorth, dSouth, dEast, dWest

let d = dEast
echo toJson(d)                                               # 2
echo toJson(d, ToJsonOptions(enumMode: joptEnumString))      # "dEast"
echo toJson(d, ToJsonOptions(enumMode: joptEnumSymbol))      # "dEast"
```

---

### `JsonNodeMode`

```nim
type JsonNodeMode* = enum
  joptJsonNodeAsRef    # return the JsonNode ref as-is (default)
  joptJsonNodeAsCopy   # deep-copy the JsonNode
  joptJsonNodeAsObject # treat JsonNode as a plain ref object
```

Controls what happens when `toJson` encounters a value that is itself a `JsonNode`. Since `JsonNode` is already JSON, wrapping it again would be wrong in most cases.

- **`joptJsonNodeAsRef`** (default): the original `JsonNode` reference is returned directly. Fast, but mutations to the result will affect the original.
- **`joptJsonNodeAsCopy`**: a deep copy is made. Safe for independent mutation.
- **`joptJsonNodeAsObject`**: the `JsonNode` is treated as an ordinary Nim ref object and serialized field-by-field (rarely needed; mainly for introspection).

---

### `ToJsonOptions`

```nim
type ToJsonOptions* = object
  enumMode*: EnumMode
  jsonNodeMode*: JsonNodeMode
```

A bundle of all options that control the behavior of `toJson`. Create one with `initToJsonOptions()` and customize the fields as needed.

---

## Procedures and Iterators

### `initToJsonOptions`

```nim
proc initToJsonOptions*(): ToJsonOptions
```

**What it does:** Returns a `ToJsonOptions` value pre-filled with sensible defaults: `enumMode = joptEnumOrd` and `jsonNodeMode = joptJsonNodeAsRef`. This is the options object `toJson` uses when you don't pass one explicitly.

**When to use it:** As a starting point when you want to customize only one aspect of serialization without specifying every field from scratch.

```nim
import std/jsonutils

var opts = initToJsonOptions()
opts.enumMode = joptEnumString   # override just the enum mode
```

---

### `toJson`

```nim
proc toJson*[T](a: T, opt = initToJsonOptions()): JsonNode
```

**What it does:** Converts any Nim value `a` into a `JsonNode`. This is the main serialization entry point. It handles:

- **Primitives** (`bool`, integers, floats, `string`, `cstring`, `char`) — mapped to their natural JSON counterparts. Notably, special float values `NaN`, `Inf`, and `-Inf` are serialized as JSON strings (`"nan"`, `"inf"`, `"-inf"`) because JSON does not support them as numbers.
- **Enums** — serialized according to `opt.enumMode` (ordinal int by default).
- **Objects and named tuples** — become JSON objects `{}` with field names as keys.
- **Anonymous tuples** — become JSON arrays `[]`.
- **Sequences, arrays, sets** — become JSON arrays `[]`.
- **`ref` and `ptr`** — a `nil` pointer becomes `null`; a non-nil pointer is dereferenced and its contents serialized.
- **`distinct` types** — the underlying base type is serialized.
- **`JsonNode`** — handled according to `opt.jsonNodeMode`.
- **Custom hooks**: if `toJsonHook(a)` or `toJsonHook(a, opt)` is in scope for type `T`, it is called instead. This is how you plug in custom logic.

**When to use it:** Any time you need to produce JSON from a Nim data structure — for saving to disk, sending over a network, logging, or testing.

```nim
import std/[jsonutils, json]

type Person = object
  name: string
  age: int

let p = Person(name: "Alice", age: 30)
let j = toJson(p)
assert $j == """{"name":"Alice","age":30}"""

# Sequences and nesting work automatically:
let people = @[Person(name: "Bob", age: 25), Person(name: "Carol", age: 40)]
echo $toJson(people)
# [{"name":"Bob","age":25},{"name":"Carol","age":40}]

# Special float handling:
assert $toJson(Inf)  == "\"inf\""
assert $toJson(0.5)  == "0.5"
```

---

### `fromJson`

```nim
proc fromJson*[T](a: var T, b: JsonNode, opt = Joptions())
```

**What it does:** Deserializes a `JsonNode` `b` into an existing Nim variable `a` **in-place**. This is the mutable, in-place sibling of `jsonTo`. Both do the same work; `fromJson` is useful when you already have a variable you want to populate (for example, in a hot loop, to avoid repeated allocation).

The procedure applies in reverse all the same type rules that `toJson` follows. It also checks for `fromJsonHook` in scope first, delegating to custom logic when available.

Validation errors raise `ValueError` with a message describing exactly which field or condition failed.

**When to use it:** When you have a pre-existing variable to fill, or when you are implementing a `fromJsonHook` for a custom type and need to call the default deserialization for individual fields.

```nim
import std/[jsonutils, json]

type Config = object
  host: string
  port: int

var c: Config
fromJson(c, parseJson("""{"host":"0.0.0.0","port":443}"""))
assert c.host == "0.0.0.0"
assert c.port == 443
```

---

### `jsonTo`

```nim
proc jsonTo*(b: JsonNode, T: typedesc, opt = Joptions()): T
```

**What it does:** Deserializes a `JsonNode` into a **new** value of type `T` and returns it. This is the functional, return-value counterpart to `fromJson`. Internally it just calls `fromJson` on a freshly created default value and returns the result.

**When to use it:** The most convenient way to deserialize JSON when you want a clean new value rather than mutating an existing one. Pairs naturally with `toJson` to form a round-trip.

```nim
import std/[jsonutils, json]

type Point = object
  x, y: float

let j = parseJson("""{"x": 1.5, "y": -3.0}""")
let p = jsonTo(j, Point)
assert p.x == 1.5
assert p.y == -3.0

# Full round-trip:
assert p.toJson.jsonTo(Point) == p
```

---

### `fromJsonHook` for `Table` and `OrderedTable`

```nim
proc fromJsonHook*[K: string|cstring, V](
    t: var (Table[K, V] | OrderedTable[K, V]),
    jsonNode: JsonNode, opt = Joptions())
```

**What it does:** Teaches `fromJson` how to deserialize a JSON object into a Nim `Table` or `OrderedTable` whose keys are `string` or `cstring`. Each JSON key becomes a table key; each JSON value is deserialized into type `V`.

This hook is called automatically by `fromJson` — you never invoke it directly. Its existence is what makes `fromJson(myTable, jsonNode)` work at all.

The incoming `jsonNode` **must** be of kind `JObject`; otherwise an assertion fires.

```nim
import std/[jsonutils, json, tables]

var scores: Table[string, int]
fromJson(scores, parseJson("""{"alice": 95, "bob": 87}"""))
assert scores["alice"] == 95
assert scores["bob"] == 87
```

---

### `toJsonHook` for `Table` and `OrderedTable`

```nim
proc toJsonHook*[K: string|cstring, V](
    t: (Table[K, V] | OrderedTable[K, V]),
    opt = initToJsonOptions()): JsonNode
```

**What it does:** Teaches `toJson` how to serialize a `Table` or `OrderedTable` with `string`/`cstring` keys into a JSON object. Each key-value pair in the table becomes a key-value pair in the JSON object `{}`.

For tables with non-string keys (e.g. `Table[int, string]`), this hook does not apply. In that case, convert the table to a sequence of tuples first and serialize that.

```nim
import std/[jsonutils, json, tables]

let prices = {"apple": 1.2, "banana": 0.5}.toTable
echo $toJson(prices)
# {"apple": 1.2, "banana": 0.5}   (key order is not guaranteed for Table)

# Non-string keys — convert to seq of tuples first:
import std/sugar
let ids = {42: "foo", 99: "bar"}.toTable
let pairs = collect: (for k, v in ids: (k, v))
echo $toJson(pairs)
# [[42,"foo"],[99,"bar"]]
```

---

### `fromJsonHook` for `HashSet` and `OrderedSet`

```nim
proc fromJsonHook*[A](s: var SomeSet[A], jsonNode: JsonNode, opt = Joptions())
```

**What it does:** Teaches `fromJson` how to deserialize a JSON array into a Nim `HashSet` or `OrderedSet`. Each element of the JSON array is deserialized into type `A` and added to the set.

The incoming `jsonNode` **must** be of kind `JArray`; otherwise an assertion fires. Duplicate values in the JSON array are silently collapsed (as expected for a set).

```nim
import std/[jsonutils, json, sets]

var tags: HashSet[string]
fromJson(tags, parseJson("""["nim", "json", "nim"]"""))
assert tags == ["nim", "json"].toHashSet  # duplicate removed
```

---

### `toJsonHook` for `HashSet` and `OrderedSet`

```nim
proc toJsonHook*[A](s: SomeSet[A], opt = initToJsonOptions()): JsonNode
```

**What it does:** Teaches `toJson` how to serialize a `HashSet` or `OrderedSet` into a JSON array. Each element in the set becomes an element in the JSON array.

Note that `HashSet` has no guaranteed iteration order, so the order of elements in the resulting JSON array is non-deterministic. `OrderedSet` preserves insertion order.

```nim
import std/[jsonutils, json, sets]

let os = ["first", "second", "third"].toOrderedSet
echo $toJson(os)
# ["first","second","third"]  — order preserved
```

---

### `fromJsonHook` for `Option[T]`

```nim
proc fromJsonHook*[T](self: var Option[T], jsonNode: JsonNode, opt = Joptions())
```

**What it does:** Teaches `fromJson` how to deserialize a JSON value into a Nim `Option[T]`. The mapping is simple and intuitive:

- JSON `null` → `none[T]()` (an absent value)
- Any other JSON value → `some(deserializedValue)` (a present value of type `T`)

This makes `Option` a natural fit for nullable JSON fields.

```nim
import std/[jsonutils, json, options]

var maybePort: Option[int]

fromJson(maybePort, parseJson("8080"))
assert maybePort == some(8080)

fromJson(maybePort, parseJson("null"))
assert maybePort == none[int]()
```

---

### `toJsonHook` for `Option[T]`

```nim
proc toJsonHook*[T](self: Option[T], opt = initToJsonOptions()): JsonNode
```

**What it does:** Teaches `toJson` how to serialize a Nim `Option[T]` into JSON. The mapping mirrors `fromJsonHook`:

- `some(value)` → the JSON representation of `value`
- `none[T]()` → JSON `null`

```nim
import std/[jsonutils, json, options]

echo $toJson(some("hello"))   # "hello"
echo $toJson(none[string]())  # null

type User = object
  name: string
  nickname: Option[string]

let u = User(name: "Alice", nickname: none[string]())
echo $toJson(u)
# {"name":"Alice","nickname":null}
```

---

### `fromJsonHook` for `StringTableRef`

```nim
proc fromJsonHook*(a: var StringTableRef, b: JsonNode)
```

**What it does:** Teaches `fromJson` how to deserialize a JSON object into a `StringTableRef`. The expected JSON format stores the table in two parts:

```json
{
  "mode": 0,
  "table": {"key1": "value1", "key2": "value2"}
}
```

The `"mode"` field is an integer corresponding to the `StringTableMode` enum (`modeCaseSensitive`, `modeCaseInsensitive`, `modeStyleInsensitive`). The `"table"` field holds the actual key-value pairs.

```nim
import std/[jsonutils, json, strtabs]

var t: StringTableRef
fromJsonHook(t, parseJson("""{"mode": 0, "table": {"HOST": "localhost"}}"""))
assert t["HOST"] == "localhost"
```

---

### `toJsonHook` for `StringTableRef`

```nim
proc toJsonHook*(a: StringTableRef): JsonNode
```

**What it does:** Teaches `toJson` how to serialize a `StringTableRef` into JSON. The output follows the two-field format that `fromJsonHook` expects: a `"mode"` field (as a string name of the mode) and a `"table"` field containing all key-value pairs.

```nim
import std/[jsonutils, json, strtabs]

let t = newStringTable("lang", "nim", "version", "2", modeCaseSensitive)
let j = toJsonHook(t)
echo $j
# {"mode":"modeCaseSensitive","table":{"lang":"nim","version":"2"}}
```

---

## Writing Your Own Hooks

The hook system is what makes `jsonutils` extensible. If you define either of these procedures for your type in any scope that is visible to the call site, `toJson` and `fromJson` will prefer them over the built-in logic:

```nim
proc toJsonHook*(a: MyType): JsonNode = ...
proc toJsonHook*(a: MyType, opt: ToJsonOptions): JsonNode = ...  # with options

proc fromJsonHook*(a: var MyType, b: JsonNode) = ...
proc fromJsonHook*(a: var MyType, b: JsonNode, opt: Joptions) = ...  # with options
```

The version accepting `opt` takes precedence over the one without, so prefer that signature when you want to pass options down to nested calls.

**Example:** Serializing a custom `Date` type as a string:

```nim
import std/[jsonutils, json, strformat]

type Date = object
  year, month, day: int

proc toJsonHook*(d: Date): JsonNode =
  newJString(fmt"{d.year:04d}-{d.month:02d}-{d.day:02d}")

proc fromJsonHook*(d: var Date, b: JsonNode) =
  let parts = b.getStr().split('-')
  d = Date(year: parts[0].parseInt, month: parts[1].parseInt, day: parts[2].parseInt)

let date = Date(year: 2024, month: 3, day: 15)
let j = toJson(date)
assert $j == "\"2024-03-15\""
assert j.jsonTo(Date) == date
```

---

## Round-Trip Guarantee

For types that are fully supported and have no custom hooks, `toJson` and `jsonTo` form a round-trip:

```nim
let original = (name: "Nim", version: (major: 2, minor: 0))
assert original.toJson.jsonTo(typeof(original)) == original
```

---

## Handling Floats: Special Values

Standard JSON does not define representations for `NaN`, positive infinity, or negative infinity. `jsonutils` serializes them as strings to avoid data loss, and parses them back symmetrically:

```nim
import std/[jsonutils, json]

let values = [NaN, Inf, -Inf, 0.0, 1.5]
let j = toJson(values)
echo $j
# ["nan","inf","-inf",0.0,1.5]

let back = j.jsonTo(typeof(values))
assert back[1] == Inf
```
