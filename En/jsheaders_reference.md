# `jsheaders` Module Reference

> **Module:** `std/jsheaders` (Nim standard library)  
> **Target:** JavaScript backend **only** — using this module with a native Nim backend is a fatal compile-time error.  
> **Purpose:** Nim bindings for the browser's [`Headers`](https://developer.mozilla.org/en-US/docs/Web/API/Headers) API, used to construct and manipulate HTTP header collections for use with `fetch()` and related APIs.

---

## Background: The Headers API

In the browser's Fetch API, HTTP headers are not represented as plain dictionaries — they are managed through a dedicated `Headers` object. This design reflects a quirk of the HTTP specification: **header names are case-insensitive**, and **multiple values under the same header name are allowed** and have special merge semantics.

For example, if a response carries three `Set-Cookie` headers, they are all distinct and must all be preserved. At the same time, sending duplicate `Content-Type` headers is an error. The `Headers` API manages this tension by providing two separate write operations with different semantics — `append` (allows duplicates) and `set` (enforces uniqueness) — which this module exposes as `add` and `[]=` respectively.

**Critical behavior to understand before using this module:**

- `add` (→ `Headers.append`) **accumulates** values — existing entries for the same key are preserved and the new value is added alongside them.
- `[]=` (→ `Headers.set`) **replaces** — all existing entries for the key are removed and replaced by the single new value.
- `[]` (→ `Headers.get`) retrieves all values for a key, **joined by `", "`** (comma-space) into a single string. This is defined by the HTTP specification, not by this module.
- Header names are **automatically normalized to lowercase** by the browser's `Headers` implementation.

---

## The `Headers` Type

```nim
type Headers* = ref object of JsRoot
```

A ref object wrapping the JavaScript `Headers` object. Passed by reference, garbage collected. All mutations operate in place on the underlying JavaScript object.

---

## Exported Functions

---

### `newHeaders`

```nim
func newHeaders*(): Headers
```

**What it does**

Creates a new, empty `Headers` object, calling `new Headers()` in JavaScript. The resulting object has zero entries and is ready to be populated.

`Headers` objects are used both for outgoing requests (passed into `newFetchOptions` or `newRequest`) and are present on incoming `Response` objects after a `fetch()` call completes.

**Example**

```nim
import std/jsheaders

let h = newHeaders()
assert h.len == 0
```

---

### `add`

```nim
func add*(self: Headers; key: cstring; value: cstring)
```

**What it does**

Appends a new `key: value` entry to the `Headers`, calling JavaScript's `Headers.append(key, value)`. If an entry for `key` already exists, the new value is added **alongside** the existing one — the existing entry is not replaced. Both coexist and will be joined when retrieved via `[]`.

This is the header equivalent of "add another value to this field". Use it for headers that legitimately carry multiple values (e.g. `Accept`, `Set-Cookie`). For headers that must be unique (e.g. `Content-Type`, `Authorization`), use `[]=` instead to avoid unintended accumulation.

Header names are case-insensitive — `"Content-Type"` and `"content-type"` refer to the same header, and the browser normalises them to lowercase.

**Example**

```nim
import std/jsheaders

let h = newHeaders()
h.add("Accept", "text/html".cstring)
h.add("Accept", "application/json".cstring)   # does NOT replace text/html

assert h["Accept"] == "text/html, application/json".cstring
# Multiple values are joined with ", " — this is HTTP spec behavior
```

---

### `delete`

```nim
func delete*(self: Headers; key: cstring)
```

**What it does**

Removes **all** entries whose key matches `key`, calling JavaScript's `Headers.delete(key)`. Because `add` can create multiple entries for the same key, this operation removes the entire group at once. There is no single-entry removal.

If `key` does not exist, this is a no-op — no error is raised.

**Example**

```nim
import std/jsheaders

let h = newHeaders()
h.add("Accept", "text/html".cstring)
h.add("Accept", "application/json".cstring)   # 2 Accept entries

h.delete("Accept")   # removes BOTH entries
assert not h.hasKey("Accept")
assert h.len == 0
```

---

### `hasKey`

```nim
func hasKey*(self: Headers; key: cstring): bool
```

**What it does**

Returns `true` if at least one entry with the given key exists, calling JavaScript's `Headers.has(key)`. The check is case-insensitive — `"content-type"` and `"Content-Type"` are treated as the same header.

**Example**

```nim
import std/jsheaders

let h = newHeaders()
h["Authorization"] = "Bearer token123".cstring

assert h.hasKey("Authorization") == true
assert h.hasKey("authorization") == true    # case-insensitive
assert h.hasKey("Content-Type") == false
```

---

### `keys`

```nim
func keys*(self: Headers): seq[cstring]
```

**What it does**

Returns a `seq[cstring]` of all header names in the `Headers` object, in insertion order, all lowercased. Wraps `Array.from(headers.keys())`.

Note that the browser's `Headers` implementation automatically normalizes header names to lowercase. So even if you added `"Content-Type"`, the key returned here will be `"content-type"`.

Unlike `FormData.keys()`, duplicate header names in `Headers` are **merged** by the browser at the storage level — so even if you `add` the same key twice, it still appears as one key in `keys()`. The multiple values are accessible only through `[]` (which joins them).

**Example**

```nim
import std/jsheaders

let h = newHeaders()
h["Content-Type"] = "application/json".cstring
h["Authorization"] = "Bearer abc".cstring

assert h.keys() == @["content-type".cstring, "authorization".cstring]
# Note: normalized to lowercase
```

---

### `values`

```nim
func values*(self: Headers): seq[cstring]
```

**What it does**

Returns a `seq[cstring]` of all header values in insertion order, one per key. Wraps `Array.from(headers.values())`. If a key had multiple values added via `add`, they appear as a single comma-joined string in this sequence (the browser merges them at storage).

**Example**

```nim
import std/jsheaders

let h = newHeaders()
h["Content-Type"] = "application/json".cstring
h.add("Accept", "text/html".cstring)
h.add("Accept", "application/json".cstring)

let vs = h.values()
# "Accept" multi-values are stored merged:
assert "text/html" in $vs[1]
assert "application/json" in $vs[1]
```

---

### `entries`

```nim
func entries*(self: Headers): seq[tuple[key, value: cstring]]
```

**What it does**

Returns a `seq` of `(key, value)` tuples for all entries in the `Headers`, in insertion order, with keys lowercased. Wraps `Array.from(headers.entries())`. This is the most complete view of the headers — keys and values together. Multi-value keys appear as a single entry with a comma-joined value string.

**Example**

```nim
import std/jsheaders

let h = newHeaders()
h["Content-Type"] = "application/json".cstring
h["Accept"] = "text/html".cstring

let e = h.entries()
assert e == @[
  ("content-type".cstring, "application/json".cstring),
  ("accept".cstring, "text/html".cstring)
]
```

---

### `[]` — get (all values, comma-joined)

```nim
func `[]`*(self: Headers; key: cstring): cstring
```

**What it does**

Returns the value associated with `key`, calling JavaScript's `Headers.get(key)`. The lookup is case-insensitive.

**The critical detail:** if the same header name was added multiple times via `add`, all values are returned as a **single `cstring`**, with individual values joined by `", "` (comma followed by space). This is mandated by the HTTP specification — headers with the same name are semantically equivalent to a single header with a comma-separated value list (with the exception of `Set-Cookie`, which the browser handles specially).

If `key` does not exist, returns `nil`.

**Example**

```nim
import std/jsheaders

let h = newHeaders()
h.add("Accept", "text/html".cstring)
h.add("Accept", "application/json".cstring)
h.add("Accept", "image/webp".cstring)

assert h["Accept"] == "text/html, application/json, image/webp".cstring
# All three values joined with ", "

assert h["X-Missing"] == nil   # non-existent key
```

---

### `[]=` — set (replace)

```nim
func `[]=`*(self: Headers; key: cstring; value: cstring)
```

**What it does**

Sets the header `key` to `value`, calling JavaScript's `Headers.set(key, value)`. If any entries for `key` already exist (including duplicates created via `add`), they are **all removed** and replaced by this single entry. Subsequent reads of `h[key]` will return exactly this `value`.

This is the correct operation for headers that must appear only once — `Content-Type`, `Authorization`, `Cache-Control`, etc.

**Example**

```nim
import std/jsheaders

let h = newHeaders()
h.add("X-Custom", "first".cstring)
h.add("X-Custom", "second".cstring)   # now 2 values

assert h["X-Custom"] == "first, second".cstring

h["X-Custom"] = "only".cstring        # replaces BOTH
assert h["X-Custom"] == "only".cstring
```

---

### `clear`

```nim
func clear*(self: Headers)
```

**What it does**

Removes all entries from the `Headers` object, leaving it empty. This is a convenience function implemented in Nim — the web `Headers` specification has no `clear()` method. Internally it iterates all current keys and calls `delete` on each, resulting in an empty object.

**Example**

```nim
import std/jsheaders

let h = newHeaders()
h["Content-Type"] = "application/json".cstring
h["Authorization"] = "Bearer abc".cstring
h["Accept"] = "text/html".cstring

assert h.len == 3
h.clear()
assert h.len == 0
assert h.entries() == @[]
```

---

### `toCstring`

```nim
func toCstring*(self: Headers): cstring
```

**What it does**

Serializes the `Headers` object to a JSON `cstring`. Internally calls `JSON.stringify(Array.from(headers.entries()))`, producing a JSON array of `[key, value]` pairs. Useful for logging, debugging, and inspecting headers at a glance.

The output format is a JSON array of two-element arrays — for example: `[["content-type","application/json"],["authorization","Bearer xyz"]]`.

**Example**

```nim
import std/jsheaders

let h = newHeaders()
h["key"] = "value".cstring
h["other"] = "another".cstring

assert h.toCstring() == """[["key","value"],["other","another"]]""".cstring
```

---

### `$`

```nim
func `$`*(self: Headers): string
```

**What it does**

Nim's stringify operator for `Headers`. Returns a Nim `string` with the JSON array representation of all entries. Thin wrapper over `toCstring`.

**Example**

```nim
import std/jsheaders

let h = newHeaders()
h["Content-Type"] = "application/json".cstring
echo $h   # [["content-type","application/json"]]
```

---

### `len`

```nim
func len*(self: Headers): int
```

**What it does**

Returns the number of **distinct header names** currently in the `Headers` object. Because the browser's `Headers` implementation merges values for the same key into a single entry (accessible via the comma-joined `[]`), `len` counts unique keys, not individual `add` calls.

This is an important distinction from `FormData.len`, where duplicates are counted separately. In `Headers`, adding the same key three times with `add` results in `len == 1`, because internally they are stored as one key with a merged value.

Implemented as `Array.from(headers.entries()).length`.

**Example**

```nim
import std/jsheaders

let h = newHeaders()
assert h.len == 0

h.add("Accept", "text/html".cstring)
h.add("Accept", "application/json".cstring)   # duplicate key
assert h.len == 1   # still 1 — merged into a single entry

h["Authorization"] = "Bearer xyz".cstring
assert h.len == 2   # 2 distinct header names
```

---

## `add` vs `[]=`: Side-by-Side

The same pattern as `FormData`, but with one crucial additional detail: the `Headers` object **merges** duplicate values into a comma-joined string rather than storing them as separate entries. This affects what `[]` returns and what `len` counts.

```nim
import std/jsheaders

let h = newHeaders()

# --- add ---
h.add("Accept", "text/html".cstring)
h.add("Accept", "image/webp".cstring)
assert h["Accept"] == "text/html, image/webp".cstring  # joined, not a list
assert h.len == 1                                       # ONE merged entry

# --- []= ---
h["Accept"] = "application/json".cstring   # replaces the merged entry
assert h["Accept"] == "application/json".cstring
assert h.len == 1                          # still 1, now with a single value
```

**Comparison with `FormData`:**

| Behavior | `Headers` | `FormData` |
|----------|-----------|------------|
| Duplicate key storage | Merged into one comma-joined value | Stored as separate entries |
| `len` with 3 `add` calls | `1` | `3` |
| Retrieve all values | `[]` returns comma-joined `cstring` | `getAll` returns `seq[cstring]` |
| `keys()` with duplicates | Key appears once | Key appears multiple times |

---

## Practical Examples

### Setting headers for a JSON API request

```nim
import std/[jsheaders, jsfetch, asyncjs]
from std/httpcore import HttpPost

proc callApi(token: cstring; payload: cstring) {.async.} =
  let h = newHeaders()
  h["Content-Type"] = "application/json"
  h["Authorization"] = "Bearer " & $token
  h["Accept"] = "application/json"

  let opts = newFetchOptions(metod = HttpPost, body = payload, headers = h)
  let resp = await fetch("https://api.example.com/data".cstring, opts)
  echo "Status: ", $resp.status
```

### Inspecting response headers

```nim
import std/[jsfetch, asyncjs]

proc checkHeaders() {.async.} =
  let resp = await fetch("https://httpbin.org/get".cstring)
  let h = resp.headers

  if h.hasKey("Content-Type"):
    echo "Content-Type: ", $h["Content-Type"]

  echo "All headers:"
  for (k, v) in h.entries():
    echo "  ", $k, ": ", $v
```

### Building an Accept header with multiple MIME types

```nim
import std/jsheaders

let h = newHeaders()
h.add("Accept", "text/html".cstring)
h.add("Accept", "application/xhtml+xml".cstring)
h.add("Accept", "application/xml;q=0.9".cstring)
h.add("Accept", "*/*;q=0.8".cstring)

echo $h["Accept"]
# "text/html, application/xhtml+xml, application/xml;q=0.9, */*;q=0.8"
```

---

## Summary Table

| Symbol | Kind | JS equivalent | Purpose |
|--------|------|--------------|---------|
| `Headers` | type | `Headers` | HTTP header collection |
| `newHeaders()` | func | `new Headers()` | Create an empty `Headers` object |
| `add(key, value)` | func | `.append(key, value)` | **Append** — adds value, merges with existing |
| `delete(key)` | func | `.delete(key)` | Remove the entry for key (all merged values) |
| `hasKey(key)` | func | `.has(key)` | Check if key exists (case-insensitive) |
| `keys()` | func | `Array.from(.keys())` | All header names, lowercased, in order |
| `values()` | func | `Array.from(.values())` | All header values (merged), in order |
| `entries()` | func | `Array.from(.entries())` | All `(key, value)` tuples in order |
| `h[key]` | func | `.get(key)` | Get value (multiple values comma-joined) |
| `h[key] = value` | func | `.set(key, value)` | **Replace** — overwrites all values for key |
| `clear()` | func | *(Nim convenience)* | Remove all entries |
| `toCstring()` | func | `JSON.stringify(Array.from(.entries()))` | JSON-serialize to `cstring` |
| `$` | func | *(Nim)* | JSON-serialize to `string` |
| `len` | func | `Array.from(.entries()).length` | Count of distinct header names |
