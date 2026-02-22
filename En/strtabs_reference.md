# `strtabs` Module Reference

> **Module:** `strtabs`  
> **Purpose:** An efficient hash table mapping strings to strings, with built-in support for case-sensitive, case-insensitive, and style-insensitive key comparison. Also provides a powerful string substitution operator `%`.

---

## Overview

`strtabs` is a specialised string-to-string hash table — simpler and lighter than the general-purpose `tables` module, but with one unique feature: the **comparison mode** is baked into the table itself. You choose at creation time whether keys are compared exactly, without regard to letter case, or in Nim's "style-insensitive" mode where underscores and case are both ignored. This makes it ideal for configuration files, environment-variable lookups, template expansion, and any scenario where key spelling is flexible.

The module also ships a `%` operator that performs `${key}` substitution against a string table, turning the table into a lightweight template engine.

---

## Types

### `StringTableMode`

```nim
type StringTableMode* = enum
  modeCaseSensitive,    ## keys matched exactly: "Name" ≠ "name"
  modeCaseInsensitive,  ## ASCII letters compared without case: "Name" == "name"
  modeStyleInsensitive  ## case AND underscores ignored: "firstName" == "first_name"
```

The mode is chosen once at table creation and determines how every key lookup and insertion works internally. It cannot be changed after the fact without calling `clear`.

`modeStyleInsensitive` follows Nim's own identifier comparison rules: all ASCII letters are folded to lowercase, and underscore characters are stripped before comparison. This makes `"myKey"`, `"my_key"`, `"MyKey"`, and `"MY_KEY"` all equivalent.

---

### `StringTableRef`

```nim
type StringTableRef* = ref StringTableObj
```

A reference-counted pointer to the table object. All public functions take or return `StringTableRef`. Because it is a `ref`, the table is passed by reference everywhere — mutations are always visible to all holders.

---

### `FormatFlag`

```nim
type FormatFlag* = enum
  useEnvironment,  ## fall back to an environment variable if key not found
  useEmpty,        ## use "" instead of raising ValueError for missing keys
  useKey           ## leave "$key" in the output if key not found
```

A set of `FormatFlag` values is passed to the `%` operator to control its behaviour when a substitution key is not present in the table. The flags are combinable: you can use `{useEnvironment, useEmpty}` at the same time.

`useEnvironment` is ignored on the JS, NimScript, and Standalone targets.

---

## Exported Symbols at a Glance

| Symbol | Kind | Summary |
|--------|------|---------|
| `newStringTable` (mode only) | proc | Create an empty table with a given mode |
| `newStringTable` (flat pairs) | proc | Create from alternating key/value strings |
| `newStringTable` (tuple pairs) | proc | Create from `(key, val)` tuples or a `{k:v}` literal |
| `mode` | proc | Read the comparison mode of a table |
| `len` | proc | Number of key-value pairs currently stored |
| `[]` | proc | Retrieve a value by key; raises `KeyError` if absent |
| `[]=` | proc | Insert or overwrite a key-value pair |
| `getOrDefault` | proc | Retrieve a value, returning a default if key absent |
| `hasKey` | proc | Test whether a key exists |
| `contains` | proc | Alias for `hasKey`; enables `in` / `notin` |
| `del` | proc | Remove a key from the table |
| `clear` (with mode) | proc | Empty the table, optionally changing mode |
| `clear` (no args) | proc | Empty the table, keeping the existing mode |
| `pairs` | iterator | Yield every `(key, value)` pair |
| `keys` | iterator | Yield every key |
| `values` | iterator | Yield every value |
| `$` | proc | String representation `{key: value, ...}` |
| `%` | proc | Template substitution with `${key}` / `$key` syntax |

---

## Creating Tables

### `newStringTable(mode)`

```nim
proc newStringTable*(mode: StringTableMode): owned(StringTableRef)
```

Creates an empty table with the specified comparison mode. This is the minimal constructor — use it when you want to fill the table manually afterward.

The returned table is `owned`, so Nim's ARC/ORC memory management will track it automatically.

```nim
var t = newStringTable(modeCaseSensitive)
t["host"] = "localhost"
t["port"] = "8080"
```

---

### `newStringTable(keyValuePairs, mode)`

```nim
proc newStringTable*(keyValuePairs: varargs[string],
                     mode: StringTableMode): owned(StringTableRef)
```

Creates a table pre-populated from a flat list of alternating key and value strings. The `mode` argument is **required** and must come last. Keys and values are interleaved: argument 0 is a key, argument 1 its value, argument 2 the next key, and so on. An odd total count simply ignores the trailing key.

```nim
var t = newStringTable("host", "localhost", "port", "8080", modeCaseSensitive)
assert t["host"] == "localhost"
assert t["port"] == "8080"
```

This overload is convenient when you are building the pair list programmatically or from a config parser that yields a flat string sequence.

---

### `newStringTable(keyValuePairs, mode)` — tuple overload

```nim
proc newStringTable*(keyValuePairs: varargs[tuple[key, val: string]],
    mode: StringTableMode = modeCaseSensitive): owned(StringTableRef)
```

The most ergonomic constructor for static data. Accepts either an array of `(key, val)` tuples or a Nim table-constructor literal `{"key": "value", ...}`. The `mode` parameter is optional and defaults to `modeCaseSensitive`.

```nim
# From a table literal (most common usage):
var cfg = {"host": "localhost", "port": "8080"}.newStringTable

# With an explicit mode:
var ci = {"name": "Alice"}.newStringTable(modeCaseInsensitive)
assert ci["NAME"] == "Alice"

# From an explicit tuple array:
var t = newStringTable([("a", "1"), ("b", "2")])
```

---

## Reading the Mode

### `mode`

```nim
proc mode*(t: StringTableRef): StringTableMode
```

Returns the comparison mode the table was created with. Useful when a function receives a `StringTableRef` from an external source and needs to know how key lookups will behave.

```nim
var t = newStringTable(modeStyleInsensitive)
assert t.mode == modeStyleInsensitive
```

---

## Querying Size

### `len`

```nim
proc len*(t: StringTableRef): int
```

Returns the number of key-value pairs currently in the table. An empty table returns `0`. Runs in O(1).

```nim
var t = {"a": "1", "b": "2"}.newStringTable
assert t.len == 2
t.del("a")
assert t.len == 1
```

---

## Accessing Values

### `[]` — Index read

```nim
proc `[]`*(t: StringTableRef, key: string): var string
```

Looks up `key` and returns a **mutable reference** to its value. If `key` is absent, `KeyError` is raised. Because the return is `var string`, you can modify the value in place:

```nim
var t = {"score": "10"}.newStringTable
echo t["score"]       # → "10"
t["score"] = "20"     # same as []=, but also via the var ref:
t["score"].add("!")   # mutate in place → "20!"
```

Use `getOrDefault` or `hasKey` before `[]` if the key might be absent.

---

### `getOrDefault`

```nim
proc getOrDefault*(t: StringTableRef; key: string, default: string = ""): string
```

Looks up `key` and returns its value. If `key` is not found, returns `default` (empty string by default) instead of raising an exception. This is the **safe** alternative to `[]` for optional keys.

```nim
var t = {"name": "John", "city": "Monaco"}.newStringTable

assert t.getOrDefault("name")              == "John"
assert t.getOrDefault("occupation")        == ""         # absent → ""
assert t.getOrDefault("occupation", "N/A") == "N/A"      # custom default
assert t.getOrDefault("name", "Fallback")  == "John"     # present → actual value
```

---

### `hasKey`

```nim
proc hasKey*(t: StringTableRef, key: string): bool
```

Returns `true` if `key` exists in the table. The comparison respects the table's mode.

```nim
var t = {"name": "John"}.newStringTable(modeCaseInsensitive)
assert t.hasKey("name")
assert t.hasKey("NAME")    # case-insensitive table: both match
assert not t.hasKey("age")
```

---

### `contains`

```nim
proc contains*(t: StringTableRef, key: string): bool
```

Alias for `hasKey`, enabling the idiomatic `in` / `notin` operators. Prefer `in` for clarity in conditions.

```nim
var t = {"name": "John", "city": "Monaco"}.newStringTable
assert "name" in t
assert "occupation" notin t

if "city" in t:
  echo t["city"]   # → Monaco
```

---

## Modifying the Table

### `[]=` — Insert or update

```nim
proc `[]=`*(t: StringTableRef, key, val: string)
```

Inserts a new `(key, value)` pair, or overwrites the value if `key` already exists. The table grows automatically (by a factor of 2) when load becomes too high.

```nim
var t = newStringTable(modeCaseSensitive)
t["lang"]  = "Nim"
t["lang"]  = "Nim 2"   # overwrites
t["year"]  = "2024"

assert t["lang"] == "Nim 2"
assert t.len == 2
```

---

### `del`

```nim
proc del*(t: StringTableRef, key: string)
```

Removes `key` and its associated value from the table. If `key` is absent, the call is a no-op (no exception is raised). The deletion algorithm uses Knuth's Algorithm 6.4R to preserve open-addressing probe chains — all remaining pairs stay correctly reachable.

```nim
var t = {"a": "1", "b": "2", "c": "3"}.newStringTable
t.del("b")
assert t.len == 2
assert "b" notin t
assert "a" in t   # other pairs are unaffected
t.del("z")        # missing key — silently ignored
```

---

### `clear` (with mode)

```nim
proc clear*(s: StringTableRef, mode: StringTableMode)
```

Empties the table and resets it to its initial (small) capacity. You can simultaneously change the comparison mode, which is useful when re-using a table for a different domain.

```nim
var t = {"name": "John", "city": "Monaco"}.newStringTable
assert t.len == 2

t.clear(modeCaseInsensitive)
assert t.len == 0
assert t.mode == modeCaseInsensitive

# Now we can refill it with a different mode:
t["LANG"] = "Nim"
assert t["lang"] == "Nim"   # case-insensitive now
```

---

### `clear` (without mode)

```nim
proc clear*(s: StringTableRef)
```

Empties the table while keeping its existing comparison mode. Equivalent to `clear(s, s.mode)`.

```nim
var t = {"a": "1"}.newStringTable(modeStyleInsensitive)
t.clear()
assert t.len == 0
assert t.mode == modeStyleInsensitive   # unchanged
```

---

## Iterating

All three iterators skip empty internal slots transparently. The iteration order is **not guaranteed** to match insertion order.

### `pairs`

```nim
iterator pairs*(t: StringTableRef): tuple[key, value: string]
```

Yields every `(key, value)` pair in the table.

```nim
var t = {"a": "1", "b": "2", "c": "3"}.newStringTable
for key, val in t.pairs:
  echo key, " → ", val
# output order is unspecified; all three pairs will appear
```

---

### `keys`

```nim
iterator keys*(t: StringTableRef): string
```

Yields every key in the table. Use when you only need the key names, not the associated values.

```nim
var t = {"host": "localhost", "port": "8080"}.newStringTable
var ks: seq[string]
for k in t.keys:
  ks.add(k)
assert "host" in ks
assert "port" in ks
```

---

### `values`

```nim
iterator values*(t: StringTableRef): string
```

Yields every value in the table. Use when you only need the stored values.

```nim
var t = {"x": "10", "y": "20"}.newStringTable
var total = 0
for v in t.values:
  total += v.parseInt
assert total == 30
```

---

## String Representation

### `$`

```nim
proc `$`*(t: StringTableRef): string
```

Returns a human-readable representation of the table in `{key: value, key: value}` format. An empty table produces the special token `{:}` (to distinguish it from an empty sequence `{}`). The iteration order of pairs within the string is unspecified.

```nim
var t = {"name": "John", "city": "Monaco"}.newStringTable
echo t   # e.g. → {name: John, city: Monaco}

var empty = newStringTable(modeCaseSensitive)
echo empty   # → {:}
```

---

## Template Substitution

### `%`

```nim
proc `%`*(f: string, t: StringTableRef, flags: set[FormatFlag] = {}): string
```

Performs `${key}` and `$key` substitution in the string `f`, looking up each key in the table `t`. This turns a `StringTableRef` into a lightweight template engine.

**Substitution syntax recognised by `%`:**

| Pattern | Effect |
|---------|--------|
| `${key}` | Replace with the value for `key` |
| `$key` | Replace with the value for `key` (key ends at first non-identifier character) |
| `$$` | Literal `$` character |
| Any other `$x` | Passed through unchanged |

A key consists of ASCII letters, digits, `_`, and high bytes (`\x80`–`\xFF`).

**Behaviour when a key is missing** (controlled by `flags`):

| Flag | Missing key behaviour |
|------|-----------------------|
| *(none)* | Raises `ValueError` |
| `useEmpty` | Substitutes `""` |
| `useKey` | Leaves `$key` or `${key}` literally in the output |
| `useEnvironment` | Looks up an OS environment variable; falls back per other flags |

```nim
var t = {"name": "John", "city": "Monaco"}.newStringTable

# Basic substitution:
assert "${name} lives in ${city}" % t == "John lives in Monaco"
assert "$name lives in $city"    % t == "John lives in Monaco"

# Literal dollar:
assert "$$name is $name" % t == "$name is John"

# Missing key — default raises ValueError:
try:
  discard "$missing" % t
except ValueError as e:
  echo e.msg   # → "format string: key not found: missing"

# Missing key — use empty string:
assert "$missing" % (t, {useEmpty}) == ""

# Missing key — keep substitution token:
assert "$missing here" % (t, {useKey}) == "$missing here"

# Fall back to environment variables (non-JS targets only):
import std/os
putEnv("USER", "alice")
var empty = newStringTable(modeCaseSensitive)
assert "$USER" % (empty, {useEnvironment}) == "alice"
```

---

## Comparison Mode in Depth

The mode affects **both reading and writing**. When you set a key under one spelling in a style-insensitive table, looking it up with any equivalent spelling will find it.

```nim
var t = newStringTable(modeStyleInsensitive)
t["first_name"] = "John"

assert t["firstName"]  == "John"   # underscore stripped, case folded
assert t["FIRST_NAME"] == "John"
assert t["firstname"]  == "John"
assert t["FirstName"]  == "John"

# Only one entry exists — all spellings map to the same slot:
assert t.len == 1
```

For `modeCaseInsensitive`, only ASCII case is ignored; underscores are significant:

```nim
var ci = newStringTable(modeCaseInsensitive)
ci["LastName"] = "Doe"
assert ci["lastname"] == "Doe"
assert ci["LASTNAME"] == "Doe"
# "last_name" would NOT match — underscore matters in case-insensitive mode
```

---

## Common Patterns

### Configuration file parsing

```nim
var cfg = newStringTable(modeCaseInsensitive)
for line in configFile.lines:
  let parts = line.split('=', 1)
  if parts.len == 2:
    cfg[parts[0].strip] = parts[1].strip

let host = cfg.getOrDefault("host", "127.0.0.1")
let port = cfg.getOrDefault("port", "80")
```

### Template rendering

```nim
let tmpl = "Dear ${name},\n\nYour order #${id} is ready in ${city}."
var vars = {"name": "Alice", "id": "42", "city": "Monaco"}.newStringTable
echo tmpl % vars
```

### Safe iteration with deletion

Do **not** delete keys while iterating with `pairs`, `keys`, or `values`. Instead, collect the keys first:

```nim
var t = {"a": "keep", "b": "remove", "c": "keep"}.newStringTable
var toRemove: seq[string]
for key, val in t.pairs:
  if val == "remove":
    toRemove.add(key)
for key in toRemove:
  t.del(key)
assert t.len == 2
```

---

## Error Reference

| Situation | Exception | Message |
|-----------|-----------|---------|
| `t[key]` — key absent | `KeyError` | `"key not found: <key>"` |
| `"$key" % t` — key absent, no flags | `ValueError` | `"format string: key not found: <key>"` |
| Odd number of arguments to flat-pair constructor | *(silent)* | trailing key is silently ignored |
