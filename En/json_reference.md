# `std/json` Module Reference (Nim)

> A high-performance JSON parser and generator for Nim.  
> The central type is `JsonNode` — a variant object that can represent any JSON value.

---

## Table of Contents

1. [Data Types](#data-types)
2. [Node Constructors](#node-constructors)
3. [The `%` Constructor and `%*` Macro](#the--constructor-and--macro)
4. [Reading Values](#reading-values)
5. [Accessing and Traversing Nodes](#accessing-and-traversing-nodes)
6. [Modifying Nodes](#modifying-nodes)
7. [Checking and Searching](#checking-and-searching)
8. [Equality and Hashing](#equality-and-hashing)
9. [Iterators](#iterators)
10. [Serialization](#serialization)
11. [String Escaping](#string-escaping)
12. [Parsing](#parsing)
13. [Deserializing into Nim Types](#deserializing-into-nim-types)

---

## Data Types

### `JsonNodeKind`

An enum describing all possible JSON node kinds:

| Value     | JSON type        |
|-----------|------------------|
| `JNull`   | `null`           |
| `JBool`   | `true` / `false` |
| `JInt`    | integer number   |
| `JFloat`  | floating point   |
| `JString` | string           |
| `JObject` | object `{}`      |
| `JArray`  | array `[]`       |

### `JsonNode`

A `ref` type holding a JSON value of any kind. The `kind` field is publicly accessible:

```nim
import std/json

let node = parseJson("42")
echo node.kind  # JInt
```

---

## Node Constructors

### `newJString(s: string): JsonNode`

Creates a `JString` node.

```nim
let s = newJString("Hello")
echo s        # "Hello"
echo s.kind   # JString
```

---

### `newJInt(n: BiggestInt): JsonNode`

Creates a `JInt` node.

```nim
let i = newJInt(42)
echo i        # 42
echo i.kind   # JInt
```

---

### `newJFloat(n: float): JsonNode`

Creates a `JFloat` node.

```nim
let f = newJFloat(3.14)
echo f        # 3.14
echo f.kind   # JFloat
```

---

### `newJBool(b: bool): JsonNode`

Creates a `JBool` node.

```nim
let b = newJBool(true)
echo b        # true
echo b.kind   # JBool
```

---

### `newJNull(): JsonNode`

Creates a `JNull` node.

```nim
let n = newJNull()
echo n        # null
echo n.kind   # JNull
```

---

### `newJObject(): JsonNode`

Creates an empty `JObject` node.

```nim
let obj = newJObject()
obj["key"] = newJInt(1)
echo obj      # {"key":1}
```

---

### `newJArray(): JsonNode`

Creates an empty `JArray` node.

```nim
let arr = newJArray()
arr.add(newJInt(1))
arr.add(newJInt(2))
echo arr      # [1,2]
```

---

## The `%` Constructor and `%*` Macro

### `%(s: string): JsonNode`

Wraps a string in a `JString` node.

```nim
let node = %"hello"
echo node   # "hello"
```

---

### `%(n: int): JsonNode`

Wraps an integer in a `JInt` node.

```nim
let node = %100
echo node   # 100
```

---

### `%(n: float): JsonNode`

Wraps a `float` in a `JFloat` node. Special values (`NaN`, `Inf`, `-Inf`) are stored as `JString`.

```nim
echo %3.14       # 3.14
echo %NaN        # "nan"
echo %Inf        # "inf"
echo %(-Inf)     # "-inf"
```

---

### `%(b: bool): JsonNode`

Wraps a `bool` in a `JBool` node.

```nim
let node = %true
echo node   # true
```

---

### `%[T](elements: openArray[T]): JsonNode`

Creates a `JArray` from a Nim array or sequence.

```nim
let arr = %[1, 2, 3]
echo arr    # [1,2,3]

let words = %["hello", "world"]
echo words  # ["hello","world"]
```

---

### `%[T: object](o: T): JsonNode`

Serializes a Nim object into a `JObject`. Fields are preserved in declaration order.

```nim
type Point = object
  x, y: int

let p = Point(x: 10, y: 20)
echo %p     # {"x":10,"y":20}
```

---

### `%(o: ref object): JsonNode`

Serializes a ref object. Returns `JNull` if the reference is `nil`.

```nim
type Node = ref object
  val: int

let n: Node = nil
echo %n     # null
```

---

### `%(o: enum): JsonNode`

Serializes an enum value as a `JString`.

```nim
type Color = enum Red, Green, Blue
echo %Green   # "Green"
```

---

### `%[T](table: Table[string, T]|OrderedTable[string, T]): JsonNode`

Creates a `JObject` from a table.

```nim
import std/tables
let t = {"a": 1, "b": 2}.toTable
echo %t   # {"a":1,"b":2}
```

---

### `%[T](opt: Option[T]): JsonNode`

Returns `JNull` if the `Option` is empty; otherwise wraps the contained value.

```nim
import std/options
echo %some(42)    # 42
echo %none(int)   # null
```

---

### `macro %*(x: untyped): untyped`

A convenience macro for building JSON structures directly using Nim syntax. Applies `%` to every element automatically.

```nim
import std/json

var name = "Alice"
let age = 30

let data = %* {
  "user": name,
  "age": age,
  "scores": [10, 20, 30]
}
echo data
# {"user":"Alice","age":30,"scores":[10,20,30]}

# Nested structures
var j = %* {"a": {"b": {"c": 42}}}
echo j{"a", "b", "c"}   # 42
```

---

## Reading Values

All `get`-procedures are safe: they return the default value (never raise) when the node is `nil` or of the wrong kind.

### `getStr(n: JsonNode, default = ""): string`

Returns the string value of a `JString` node.

```nim
let j = parseJson("""{"name": "Bob"}""")
echo j["name"].getStr()          # Bob
echo j{"missing"}.getStr("?")   # ?
```

---

### `getInt(n: JsonNode, default = 0): int`

Returns the integer value of a `JInt` node.

```nim
let j = parseJson("""{"n": 7}""")
echo j["n"].getInt()       # 7
echo j{"x"}.getInt(-1)    # -1
```

---

### `getBiggestInt(n: JsonNode, default = 0): BiggestInt`

Like `getInt`, but returns `BiggestInt` — suitable for very large integers.

```nim
let j = parseJson("9999999999999999")
echo j.getBiggestInt()   # 9999999999999999
```

---

### `getFloat(n: JsonNode, default = 0.0): float`

Returns the float value. Accepts both `JFloat` and `JInt` nodes.

```nim
let j = parseJson("""{"pi": 3.14, "n": 2}""")
echo j["pi"].getFloat()    # 3.14
echo j["n"].getFloat()     # 2.0  (auto-converted from JInt)
echo j{"x"}.getFloat(1.0) # 1.0
```

---

### `getBool(n: JsonNode, default = false): bool`

Returns the boolean value of a `JBool` node.

```nim
let j = parseJson("""{"flag": true}""")
echo j["flag"].getBool()        # true
echo j{"miss"}.getBool(true)   # true
```

---

### `getFields(n: JsonNode, default = ...): OrderedTable[string, JsonNode]`

Returns the ordered field table of a `JObject` node.

```nim
let j = parseJson("""{"a": 1, "b": 2}""")
let fields = j.getFields()
for k, v in fields:
  echo k, " = ", v
# a = 1
# b = 2
```

---

### `getElems(n: JsonNode, default = @[]): seq[JsonNode]`

Returns the elements of a `JArray` node as a sequence.

```nim
let j = parseJson("[10, 20, 30]")
let elems = j.getElems()
echo elems.len    # 3
echo elems[1]     # 20
```

---

## Accessing and Traversing Nodes

### `[](node: JsonNode, name: string): JsonNode`

Gets a field from a `JObject` by name. Raises `KeyError` if the key does not exist.

```nim
let j = parseJson("""{"x": 5}""")
echo j["x"]   # 5
# j["y"]  -> KeyError!
```

---

### `[](node: JsonNode, index: int): JsonNode`

Gets an element from a `JArray` by index.

```nim
let j = parseJson("[10, 20, 30]")
echo j[0]   # 10
echo j[2]   # 30
```

---

### `[](node: JsonNode, index: BackwardsIndex): JsonNode`

Accesses an array element from the end using the `^` operator.

```nim
let j = parseJson("[1, 2, 3, 4, 5]")
echo j[^1]   # 5
echo j[^2]   # 4
```

---

### `[][U, V](a: JsonNode, x: HSlice[U, V]): JsonNode`

Slice operation for `JArray` (inclusive range). Returns a new `JArray`.

```nim
let arr = %[0, 1, 2, 3, 4, 5]
echo arr[2..4]    # [2,3,4]
echo arr[2..^2]   # [2,3,4]
echo arr[^4..^2]  # [2,3,4]
```

---

### `{}(node: JsonNode, keys: varargs[string]): JsonNode`

Safe traversal by a chain of keys. Returns `nil` if any key is missing (no exception).

```nim
let j = parseJson("""{"a": {"b": {"c": 99}}}""")
echo j{"a", "b", "c"}         # 99
echo j{"a", "x"}              # (nil — no exception)
echo j{"a", "x"}.getInt(-1)   # -1
```

---

### `{}(node: JsonNode, index: varargs[int]): JsonNode`

Safe traversal by a chain of array indices. Returns `nil` if any index is out of bounds.

```nim
let j = parseJson("[[1,2],[3,4]]")
echo j{0, 1}   # 2
echo j{5, 0}   # (nil)
```

---

### `getOrDefault(node: JsonNode, key: string): JsonNode`

Returns a field from the object or `nil` if the node is not a `JObject` / the key is absent.

```nim
let j = parseJson("""{"a": 1}""")
echo j.getOrDefault("a")    # 1
echo j.getOrDefault("b")    # (nil)
```

---

### `{}=(node: JsonNode, keys: varargs[string], value: JsonNode)`

Sets a value by a chain of keys, creating intermediate objects as needed (autovivification).

```nim
var j = newJObject()
j{"x", "y", "z"} = %42
echo j   # {"x":{"y":{"z":42}}}
```

---

### `len(n: JsonNode): int`

Returns the number of elements in a `JArray` or the number of pairs in a `JObject`. Returns `0` for all other node kinds.

```nim
let arr = parseJson("[1,2,3]")
echo arr.len   # 3

let obj = parseJson("""{"a":1,"b":2}""")
echo obj.len   # 2

let s = parseJson("\"hello\"")
echo s.len     # 0
```

---

## Modifying Nodes

### `add(father, child: JsonNode)`

Appends an element to a `JArray`.

```nim
var arr = newJArray()
arr.add(%1)
arr.add(%"two")
echo arr   # [1,"two"]
```

---

### `add(obj: JsonNode, key: string, val: JsonNode)`

Sets a field in a `JObject` (equivalent to `obj[key] = val`).

```nim
var obj = newJObject()
obj.add("name", %"Alice")
echo obj   # {"name":"Alice"}
```

---

### `[]=(obj: JsonNode, key: string, val: JsonNode)`

Assignment operator for `JObject` fields.

```nim
var obj = newJObject()
obj["age"] = %25
echo obj   # {"age":25}
```

---

### `delete(obj: JsonNode, key: string)`

Deletes a field from a `JObject`. Raises `KeyError` if the key is not found.

```nim
var j = parseJson("""{"a": 1, "b": 2}""")
j.delete("a")
echo j   # {"b":2}
```

---

### `copy(p: JsonNode): JsonNode`

Performs a deep copy of a node.

```nim
let original = parseJson("""{"x": [1,2,3]}""")
var clone = original.copy()
clone["x"][0] = %99
echo original   # {"x":[1,2,3]}  — original is unchanged
echo clone      # {"x":[99,2,3]}
```

---

## Checking and Searching

### `hasKey(node: JsonNode, key: string): bool`

Checks whether a key exists in a `JObject`.

```nim
let j = parseJson("""{"a": 1}""")
echo j.hasKey("a")   # true
echo j.hasKey("b")   # false
```

---

### `contains(node: JsonNode, key: string): bool`

Checks whether a key exists in a `JObject`. Enables the `in` operator.

```nim
let j = parseJson("""{"x": 1}""")
echo "x" in j   # true
echo "y" in j   # false
```

---

### `contains(node: JsonNode, val: JsonNode): bool`

Checks whether a value exists in a `JArray`.

```nim
let arr = parseJson("[1, 2, 3]")
echo arr.contains(%2)   # true
echo arr.contains(%9)   # false
```

---

## Equality and Hashing

### `==(a, b: JsonNode): bool`

Compares two nodes for equality. For objects, key order does not matter.

```nim
let a = parseJson("""{"x": 1, "y": 2}""")
let b = parseJson("""{"y": 2, "x": 1}""")
echo a == b   # true

echo parseJson("null") == newJNull()   # true
```

---

### `hash(n: JsonNode): Hash`

Computes a hash for a node. Allows `JsonNode` to be used as a key in tables and sets.

```nim
import std/tables
var t = initTable[JsonNode, string]()
t[%42] = "answer"
echo t[%42]   # answer
```

---

## Iterators

### `items(node: JsonNode): JsonNode`

Iterates over elements of a `JArray`.

```nim
let arr = parseJson("[10, 20, 30]")
for item in arr:
  echo item
# 10
# 20
# 30
```

---

### `mitems(node: var JsonNode): var JsonNode`

Iterates over elements of a `JArray` with mutable access.

```nim
var arr = parseJson("[1, 2, 3]")
for item in arr.mitems:
  item = %(item.getInt * 10)
echo arr   # [10,20,30]
```

---

### `pairs(node: JsonNode): tuple[key: string, val: JsonNode]`

Iterates over key–value pairs of a `JObject`. Order is preserved (OrderedTable).

```nim
let j = parseJson("""{"b": 2, "a": 1}""")
for k, v in j:
  echo k, ": ", v
# b: 2
# a: 1
```

---

### `keys(node: JsonNode): string`

Iterates over the keys of a `JObject`.

```nim
let j = parseJson("""{"x": 1, "y": 2}""")
for k in j.keys:
  echo k
# x
# y
```

---

### `mpairs(node: var JsonNode): tuple[key: string, val: var JsonNode]`

Iterates over key–value pairs of a `JObject` with mutable value access.

```nim
var j = parseJson("""{"a": 1, "b": 2}""")
for k, v in j.mpairs:
  v = %(v.getInt * 100)
echo j   # {"a":100,"b":200}
```

---

### `parseJsonFragments(s: Stream, filename = "", rawIntegers = false, rawFloats = false): JsonNode`

An iterator that reads multiple JSON values from a stream, separated by whitespace. Closes the stream when done.

```nim
import std/[json, streams]

let s = newStringStream("""{"a":1} {"b":2} [1,2,3]""")
for node in parseJsonFragments(s):
  echo node
# {"a":1}
# {"b":2}
# [1,2,3]
```

---

## Serialization

### `toUgly(result: var string, node: JsonNode)`

Serializes a node into compact JSON without any formatting. Faster than `pretty`.

```nim
var output = ""
let j = parseJson("""{"a": 1, "b": [1,2]}""")
toUgly(output, j)
echo output   # {"a":1,"b":[1,2]}
```

---

### `pretty(node: JsonNode, indent = 2): string`

Returns a formatted JSON string with indentation and newlines. Similar to Python's `json.dumps(..., indent=2)`.

```nim
let j = %* {"name": "Isaac", "books": ["Robot Dreams"],
             "details": {"age": 35, "pi": 3.1415}}
echo pretty(j)
# {
#   "name": "Isaac",
#   "books": [
#     "Robot Dreams"
#   ],
#   "details": {
#     "age": 35,
#     "pi": 3.1415
#   }
# }

# Custom indent width:
echo pretty(j, indent = 4)
```

---

### `$(node: JsonNode): string`

String conversion operator — compact (single-line) JSON. Used implicitly by `echo`.

```nim
let j = %* {"x": 1, "y": [2, 3]}
echo $j   # {"x":1,"y":[2,3]}
echo j    # same result (echo calls $)
```

---

## String Escaping

### `escapeJsonUnquoted(s: string): string`

### `escapeJsonUnquoted(s: string; result: var string)`

Escapes special characters in a string for JSON **without** surrounding quotes. The two-argument version appends to an existing string.

```nim
echo escapeJsonUnquoted("line1\nline2")   # line1\nline2
echo escapeJsonUnquoted("tab\there")      # tab\there
```

---

### `escapeJson(s: string): string`

### `escapeJson(s: string; result: var string)`

Escapes a string for JSON **with** surrounding double quotes.

```nim
echo escapeJson("""say "hello"""")   # "say \"hello\""
echo escapeJson("a\nb")              # "a\nb"
```

---

## Parsing

### `parseJson(buffer: string; rawIntegers = false, rawFloats = false): JsonNode`

Parses JSON from a string. Raises `JsonParsingError` if there is extra data after the JSON value.

- `rawIntegers = true` — integers are stored as `JString` instead of `JInt`.
- `rawFloats = true` — floats are stored as `JString` instead of `JFloat`.

```nim
let j = parseJson("""{"key": 42, "list": [1, 2, 3]}""")
echo j["key"].getInt()      # 42
echo j["list"][1].getInt()  # 2

# rawIntegers example:
let j2 = parseJson("12345678901234567890", rawIntegers = true)
echo j2.kind    # JString (number too large for BiggestInt)
echo j2.getStr  # 12345678901234567890
```

---

### `parseJson(s: Stream, filename = "", rawIntegers = false, rawFloats = false): JsonNode`

Parses JSON from a `Stream`. Closes the stream when done.

```nim
import std/[json, streams]

let s = newStringStream("""[1, 2, 3]""")
let j = parseJson(s)
echo j[0].getInt()   # 1
```

---

### `parseFile(filename: string): JsonNode`

Parses JSON from a file. Raises `IOError` if the file cannot be opened.

```nim
# Assuming data.json contains {"answer": 42}
let j = parseFile("data.json")
echo j["answer"].getInt()   # 42
```

---

## Deserializing into Nim Types

### `to[T](node: JsonNode, t: typedesc[T]): T`

Unmarshals a `JsonNode` into an arbitrary Nim type. Supports objects, tuples, sequences, arrays, tables, `Option`, enums, and nested structures.

Known limitations:
- Heterogeneous arrays are not supported.
- Object variants with sets are not supported.
- `not nil` annotations are not supported.

```nim
import std/[json, options]

type
  Address = object
    city: string
    zip: int

  User = object
    name: string
    age: int
    address: Address
    nickname: Option[string]

let json = parseJson("""
{
  "name": "Alice",
  "age": 30,
  "address": {"city": "London", "zip": 10001}
}
""")

let user = json.to(User)
echo user.name             # Alice
echo user.age              # 30
echo user.address.city     # London
echo user.nickname.isSome  # false
```

```nim
# Deserializing a seq
let arr = parseJson("[10, 20, 30]")
let nums = arr.to(seq[int])
echo nums   # @[10, 20, 30]

# Deserializing an enum
type Color = enum Red, Green, Blue
let c = parseJson(""""Green"""").to(Color)
echo c   # Green

# Deserializing a table
import std/tables
let tbl = parseJson("""{"a": 1, "b": 2}""").to(Table[string, int])
echo tbl["a"]   # 1
```

---

## Quick Reference Cheat Sheet

```nim
import std/json

# Parsing
let j = parseJson("""{"name":"Nim","ver":2,"tags":["fast","safe"]}""")

# Reading values
echo j["name"].getStr()          # Nim
echo j["ver"].getInt()           # 2
echo j["tags"][0].getStr()       # fast
echo j{"missing"}.getStr("?")   # ?  (safe — no exception)

# Building JSON
var data = %* {
  "scores": [95, 87, 100],
  "active": true,
  "ratio": 0.99
}
data["extra"] = %* {"note": "ok"}

# Output
echo data           # compact JSON
echo pretty(data)   # formatted JSON

# Deserializing
type Config = object
  scores: seq[int]
  active: bool

let cfg = data.to(Config)
echo cfg.active   # true
```
