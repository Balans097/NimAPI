# `jsformdata` Module Reference

> **Module:** `std/jsformdata` (Nim standard library)  
> **Target:** JavaScript backend **only** — using this module with a native Nim backend is a fatal compile-time error.  
> **Purpose:** Nim bindings for the browser's [`FormData`](https://developer.mozilla.org/en-US/docs/Web/API/FormData) API, enabling construction and manipulation of form-encoded data for submission via `fetch()` or `XMLHttpRequest`.

---

## Background: What Is FormData?

`FormData` is a browser-native data structure that represents a set of key-value pairs, mirroring the data a user would submit through an HTML `<form>` element with `enctype="multipart/form-data"`. It is the standard way to:

- Upload files alongside other fields to a server.
- Send structured form data via `fetch()` without manually serializing to JSON or URL-encoded strings.
- Pre-populate or programmatically modify form data before submission.

**Critical behavioral difference from a plain dictionary:** `FormData` is a **multimap** — it deliberately allows multiple values under the same key. This is intentional and matches how HTML checkboxes and multi-select fields work. The operations `add` and `[]=` (via `put`) have distinct semantics because of this:

- `add` (→ `FormData.append`) **adds** a new entry, even if the key already exists. All entries for that key accumulate.
- `[]=` (→ `FormData.set`) **replaces** all existing entries for that key with a single new one.

Understanding this distinction is the single most important thing to know before using this module.

---

## The `FormData` Type

```nim
type FormData* = ref object of JsRoot
```

A ref object wrapping the JavaScript `FormData` object. Being a `ref` type, it is passed by reference and garbage collected. All mutations are performed in place on the underlying JavaScript object.

---

## Exported Functions

---

### `newFormData`

```nim
func newFormData*(): FormData
```

**What it does**

Creates a new, empty `FormData` object. This corresponds to `new FormData()` in JavaScript. The resulting object has zero entries and is ready to receive data through `add`, `[]=`, or `put`.

**Example**

```nim
import std/jsformdata

let data = newFormData()
assert data.len == 0
```

---

### `add` (without filename)

```nim
func add*(self: FormData; name: cstring; value: SomeNumber | bool | cstring | Blob)
```

**What it does**

Appends a new key-value pair to the `FormData`, calling JavaScript's `FormData.append(name, value)`. If an entry with `name` already exists, the new entry is added **alongside it** — the existing entry is not replaced. Both entries coexist in insertion order.

This is the key behavioral distinction from `[]=`: `add` accumulates, `[]=` replaces.

Accepted value types: any Nim number (`int`, `float`, etc. via `SomeNumber`), `bool`, `cstring`, or a DOM `Blob`.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data.add("color", "red".cstring)
data.add("color", "blue".cstring)   # does NOT replace "red"

let colors = data.getAll("color")
assert colors == @["red".cstring, "blue".cstring]  # both are present
```

---

### `add` (with filename)

```nim
func add*(self: FormData; name: cstring; value: SomeNumber | bool | cstring | Blob; filename: cstring)
```

**What it does**

The three-argument variant of `add`, which attaches a `filename` hint to the entry. When `value` is a `Blob` or `File`, this filename is sent to the server as the suggested filename in the `Content-Disposition` header of that multipart entry. For non-Blob values, the `filename` argument is technically accepted but has no practical effect on most servers.

Like the two-argument version, this **appends** without replacing existing entries for the same key.

**Example**

```nim
import std/[jsformdata, dom]

let data = newFormData()
let blob: Blob = ...   # obtained from a file input or constructed
data.add("upload", blob, "report.pdf".cstring)
```

---

### `delete`

```nim
func delete*(self: FormData; name: cstring)
```

**What it does**

Removes **all** entries whose key matches `name`, calling JavaScript's `FormData.delete(name)`. This is an all-or-nothing operation: if you added three entries under `"color"`, all three are removed. There is no way to delete only one entry from a group of duplicates using this function — to achieve that, you would need to `getAll`, filter, `delete` all, and re-`add` the ones you want to keep.

If no entry with `name` exists, this is a no-op — no error is raised.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data.add("tag", "alpha".cstring)
data.add("tag", "beta".cstring)
data.add("tag", "gamma".cstring)

data.delete("tag")       # removes ALL three "tag" entries
assert not data.hasKey("tag")
```

---

### `getAll`

```nim
func getAll*(self: FormData; name: cstring): seq[cstring]
```

**What it does**

Returns a `seq[cstring]` containing **all values** associated with the given key, in insertion order. This is the correct way to read from a key that may have multiple values (added via `add`). If the key does not exist, an empty seq is returned.

Compare with `[]`, which returns only the **first** value for a key.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data.add("fruit", "apple".cstring)
data.add("fruit", "banana".cstring)
data.add("fruit", "cherry".cstring)

let fruits = data.getAll("fruit")
assert fruits.len == 3
assert fruits[0] == "apple".cstring
assert fruits[2] == "cherry".cstring

# A key that doesn't exist:
assert data.getAll("vegetable") == @[]
```

---

### `hasKey`

```nim
func hasKey*(self: FormData; name: cstring): bool
```

**What it does**

Returns `true` if at least one entry with the given key exists in the `FormData`, calling JavaScript's `FormData.has(name)`. Returns `false` otherwise. This is a safe existence check that does not raise even if the key is absent.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data["username"] = "alice".cstring

assert data.hasKey("username") == true
assert data.hasKey("password") == false
```

---

### `keys`

```nim
func keys*(self: FormData): seq[cstring]
```

**What it does**

Returns a `seq[cstring]` of all keys in the `FormData`, in insertion order. If the same key appears multiple times (via `add`), it appears multiple times in the returned sequence. Wraps `Array.from(formData.keys())`.

This behavior differs from what you might expect if you think of `FormData` as a dictionary — iterating keys gives you the full multi-valued picture, not a deduplicated set.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data.add("a", "1".cstring)
data.add("b", "2".cstring)
data.add("a", "3".cstring)   # duplicate key

let ks = data.keys()
assert ks == @["a".cstring, "b".cstring, "a".cstring]
# "a" appears twice because there are two entries with that key
```

---

### `values`

```nim
func values*(self: FormData): seq[cstring]
```

**What it does**

Returns a `seq[cstring]` of all values in the `FormData`, in insertion order — one value per entry. Wraps `Array.from(formData.values())`. The values are returned in the same positional order as the corresponding entries in `keys()` and `pairs()`.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data.add("x", "10".cstring)
data.add("y", "20".cstring)

let vs = data.values()
assert vs == @["10".cstring, "20".cstring]
```

---

### `pairs`

```nim
func pairs*(self: FormData): seq[tuple[key, val: cstring]]
```

**What it does**

Returns a `seq` of `(key, val)` tuples for every entry in the `FormData`, in insertion order. Wraps `Array.from(formData.entries())`. This is the most complete view of the data: unlike `keys()` alone or `values()` alone, each tuple keeps the key and value together, and all duplicates are visible.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data.add("name", "Alice".cstring)
data.add("role", "admin".cstring)
data.add("role", "editor".cstring)

for (k, v) in data.pairs():
  echo $k, " = ", $v
# name = Alice
# role = admin
# role = editor
```

---

### `put` (with filename)

```nim
func put*(self: FormData; name: cstring; value: SomeNumber | bool | cstring | Blob; filename: cstring)
```

**What it does**

Sets (replaces) the entry for `name` to `value`, with an associated `filename`, calling JavaScript's `FormData.set(name, value, filename)`. All existing entries for `name` are removed and replaced by this single new entry. The `filename` is attached as a content-disposition hint, useful for `Blob` file uploads.

This is the three-argument replacement counterpart to `[]=`.

**Example**

```nim
import std/[jsformdata, dom]

let data = newFormData()
let blob: Blob = ...
data.put("attachment", blob, "final_report.pdf".cstring)
# Any previous "attachment" entries are gone; only this one remains
```

---

### `[]=` — set (replace)

```nim
func `[]=`*(self: FormData; name: cstring; value: SomeNumber | bool | cstring | Blob)
```

**What it does**

Sets the entry for `name` to `value`, calling JavaScript's `FormData.set(name, value)`. If any entries for `name` already exist, they are **all removed** and replaced by this single new entry. This is the **replace** operation, in contrast to `add` which is the **append** operation.

Use `[]=` when you want exactly one value for a key. Use `add` when you want to accumulate multiple values.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data.add("status", "draft".cstring)
data.add("status", "pending".cstring)   # now has 2 "status" entries

data["status"] = "published".cstring    # replaces BOTH with one entry

assert data.getAll("status") == @["published".cstring]
```

---

### `[]` — get (first value)

```nim
func `[]`*(self: FormData; name: cstring): cstring
```

**What it does**

Returns the **first** value associated with `name`, calling JavaScript's `FormData.get(name)`. If no entry with `name` exists, returns `nil` (JavaScript `null`). If multiple entries exist under the same key, only the first one is returned — use `getAll` to retrieve all of them.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data.add("lang", "en".cstring)
data.add("lang", "fr".cstring)

assert data["lang"] == "en".cstring    # first value only
assert data.getAll("lang").len == 2    # both are accessible via getAll

# Missing key returns nil:
assert data["missing"] == nil
```

---

### `clear`

```nim
func clear*(self: FormData)
```

**What it does**

Removes all entries from the `FormData`, leaving it empty. This is a convenience function implemented in Nim (not a direct mapping to a native JavaScript method, since `FormData` has no `clear()` in the web spec). Internally it calls `keys()` on the current entries and calls `delete` on each unique key, resulting in a completely empty object.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data["a"] = "1".cstring
data["b"] = "2".cstring
data.add("c", "3".cstring)

assert data.len == 3
data.clear()
assert data.len == 0
assert not data.hasKey("a")
```

---

### `toCstring`

```nim
func toCstring*(self: FormData): cstring
```

**What it does**

Serializes the `FormData` object to a JSON `cstring` by calling JavaScript's `JSON.stringify()` on it. Useful for debugging and logging — lets you quickly inspect the contents without iterating through `pairs()`.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data["user"] = "bob".cstring
echo $data.toCstring()
```

---

### `$`

```nim
func `$`*(self: FormData): string
```

**What it does**

Nim's stringify operator for `FormData`. Returns a Nim `string` with the JSON representation of the object. A thin wrapper over `toCstring`.

**Example**

```nim
import std/jsformdata

let data = newFormData()
data["x"] = "42".cstring
echo $data
```

---

### `len`

```nim
func len*(self: FormData): int
```

**What it does**

Returns the total number of **entries** in the `FormData`. Because duplicate keys are allowed, `len` counts individual entries, not unique keys. A `FormData` with key `"color"` added three times has `len == 3`, not `len == 1`.

Implemented by enumerating all entries via `Array.from(formData.entries()).length`.

**Example**

```nim
import std/jsformdata

let data = newFormData()
assert data.len == 0

data.add("tag", "nim".cstring)
data.add("tag", "js".cstring)
data.add("tag", "web".cstring)
assert data.len == 3   # 3 entries, all under key "tag"

data.delete("tag")
assert data.len == 0
```

---

## `add` vs `[]=`: The Central Distinction

This is the most common source of confusion when using `FormData`. A side-by-side comparison:

```nim
import std/jsformdata

let data = newFormData()

# --- add (append) ---
data.add("color", "red".cstring)
data.add("color", "blue".cstring)
assert data.getAll("color") == @["red".cstring, "blue".cstring]  # both exist
assert data.len == 2

# --- []= (set/replace) ---
data["color"] = "green".cstring    # removes "red" AND "blue", adds "green"
assert data.getAll("color") == @["green".cstring]                # only one
assert data.len == 1
```

**Rule of thumb:**
- Use `[]=` when a key should have **exactly one value** (like a text input field).
- Use `add` when a key can have **multiple values** (like a multi-select or checkbox group).

---

## Practical Examples

### Building a login form payload

```nim
import std/[jsformdata, jsfetch, asyncjs]
from std/httpcore import HttpPost

proc login(username, password: cstring) {.async.} =
  let data = newFormData()
  data["username"] = username
  data["password"] = password

  let opts = newFetchOptions(metod = HttpPost, body = nil)
  # FormData is passed directly to fetch body:
  let resp = await fetch("https://api.example.com/login".cstring, opts)
  echo $resp.status
```

### Uploading a file with metadata

```nim
import std/[jsformdata, dom]

let data = newFormData()
data["description"] = "Monthly report".cstring
data["category"] = "finance".cstring
# Add blob with a filename hint:
let file: Blob = ...  # e.g. from a file input element
data.add("file", file, "report_march.pdf".cstring)
```

### Inspecting all entries

```nim
import std/jsformdata

let data = newFormData()
data.add("sizes", "S".cstring)
data.add("sizes", "M".cstring)
data.add("sizes", "L".cstring)
data["color"] = "blue".cstring

echo "Keys: ", $data.keys()    # @["sizes", "sizes", "sizes", "color"]
echo "Len: ", data.len         # 4

for (k, v) in data.pairs():
  echo $k, " → ", $v
# sizes → S
# sizes → M
# sizes → L
# color → blue
```

---

## Summary Table

| Symbol | Kind | JS equivalent | Purpose |
|--------|------|--------------|---------|
| `FormData` | type | `FormData` | Multi-map of key-value form entries |
| `newFormData()` | func | `new FormData()` | Create an empty `FormData` |
| `add(name, value)` | func | `.append(name, value)` | **Append** — adds alongside existing entries |
| `add(name, value, filename)` | func | `.append(name, value, filename)` | Append with filename hint (for Blob uploads) |
| `delete(name)` | func | `.delete(name)` | Remove **all** entries for a key |
| `getAll(name)` | func | `.getAll(name)` | Get all values for a key as `seq` |
| `hasKey(name)` | func | `.has(name)` | Check if any entry with key exists |
| `keys()` | func | `Array.from(.keys())` | All keys in order (with duplicates) |
| `values()` | func | `Array.from(.values())` | All values in order |
| `pairs()` | func | `Array.from(.entries())` | All (key, value) tuples in order |
| `put(name, value, filename)` | func | `.set(name, value, filename)` | **Replace** all entries for key, with filename |
| `data[name] = value` | func | `.set(name, value)` | **Replace** all entries for key |
| `data[name]` | func | `.get(name)` | Get **first** value for key (`nil` if absent) |
| `clear()` | func | *(Nim convenience)* | Remove all entries |
| `toCstring()` | func | `JSON.stringify()` | JSON-serialize to `cstring` |
| `$` | func | *(Nim)* | JSON-serialize to `string` |
| `len` | func | `Array.from(.entries()).length` | Total entry count (including duplicates) |
