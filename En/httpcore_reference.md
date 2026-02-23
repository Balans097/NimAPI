# Nim `httpcore` Module — Complete Reference

> **Module purpose:** Shared HTTP primitives used by both `httpclient` and `asynchttpserver`. Defines the types, constants, and utility functions that form the vocabulary of HTTP in Nim: status codes, methods, versions, and headers.
>
> **API stability:** Marked *unstable* — the public interface may change between Nim releases.
>
> **Key design decisions:**
> - `HttpCode` is a `distinct` integer, preventing accidental mixing with plain `int`.
> - `HttpHeaders` stores all keys case-insensitively by default (or in Title-Case when explicitly requested), mirroring the HTTP/1.1 specification that header names are case-insensitive.
> - A header key can have **multiple values** — the storage type is `seq[string]` per key — which is required by the HTTP spec (e.g. multiple `Set-Cookie` lines).

---

## Table of Contents

1. [Types](#types)
   - [`HttpHeaders`](#httpheaders)
   - [`HttpHeaderValues`](#httpheadervalues)
   - [`HttpCode`](#httpcode)
   - [`HttpVersion`](#httpversion)
   - [`HttpMethod`](#httpmethod)
2. [Constants](#constants)
   - [`httpNewLine`](#httpnewline)
   - [`headerLimit`](#headerlimit)
   - [HTTP status code constants](#http-status-code-constants)
3. [HttpHeaders — construction](#httpheaders--construction)
   - [`newHttpHeaders` (empty)](#newhttpheaders-empty)
   - [`newHttpHeaders` (from pairs)](#newhttpheaders-from-pairs)
4. [HttpHeaders — reading](#httpheaders--reading)
   - [`[]` (single key)](#-single-key)
   - [`[]` (key + index)](#-key--index)
   - [`getOrDefault`](#getordefault)
   - [`hasKey`](#haskey)
   - [`contains` (HttpHeaderValues)](#contains-httpheadervalues)
   - [`len`](#len)
   - [`pairs`](#pairs)
   - [`$` (HttpHeaders)](#-httpheaders)
5. [HttpHeaders — writing](#httpheaders--writing)
   - [`[]=` (single value)](#-single-value)
   - [`[]=` (seq of values)](#-seq-of-values)
   - [`add`](#add)
   - [`del`](#del)
   - [`clear`](#clear)
6. [HttpHeaders — internal](#httpheaders--internal)
   - [`toCaseInsensitive`](#tocaseinsensitive)
7. [HttpCode — utilities](#httpcode--utilities)
   - [`$` (HttpCode)](#-httpcode)
   - [`==` (HttpCode)](#-httpcode-1)
   - [`is1xx`](#is1xx)
   - [`is2xx`](#is2xx)
   - [`is3xx`](#is3xx)
   - [`is4xx`](#is4xx)
   - [`is5xx`](#is5xx)
8. [HttpVersion — utilities](#httpversion--utilities)
   - [`==` (protocol tuple vs HttpVersion)](#-protocol-tuple-vs-httpversion)
9. [HttpMethod — utilities](#httpmethod--utilities)
   - [`contains` (set of HttpMethod)](#contains-set-of-httpmethod)
10. [Parsing](#parsing)
    - [`parseHeader`](#parseheader)
11. [Converter](#converter)
    - [`toString`](#tostring)

---

## Types

### `HttpHeaders`

```nim
type HttpHeaders* = ref object
  table*: TableRef[string, seq[string]]
  isTitleCase: bool
```

The central container for HTTP headers. It wraps a hash table that maps header names (strings) to **lists** of values (`seq[string]`), because the HTTP specification allows the same header name to appear multiple times (e.g. multiple `Set-Cookie` or `Accept-Encoding` lines).

Two normalisation modes are available:
- **Lowercase mode** (default) — all keys are stored as lowercase. Lookups are always lowercase-normalised too, making access reliably case-insensitive.
- **Title-Case mode** (`titleCase=true`) — keys are stored in HTTP Title-Case (e.g. `Content-Type`, `X-My-Header`). Useful when the downstream server or proxy is non-standard and cares about capitalisation.

The `table` field is exported, giving you direct access to the raw storage when you need to iterate or do bulk operations.

```nim
let h = newHttpHeaders()
h["Content-Type"] = "application/json"
h["content-type"] = "text/html"   # same key — replaces!
echo h["CONTENT-TYPE"]             # "text/html"  (case-insensitive)
```

---

### `HttpHeaderValues`

```nim
type HttpHeaderValues* = distinct seq[string]
```

A thin wrapper around `seq[string]` returned by the single-key `[]` operator. The `distinct` prevents accidental use as a plain `seq`. The built-in **converter** `toString` automatically extracts the first value when the result is passed to a context expecting a plain `string`, making common single-value access ergonomic:

```nim
let h = newHttpHeaders({"Content-Length": "42"})
let s: string = h["Content-Length"]   # converter fires: s == "42"
```

To access all values, cast back to `seq[string]`:

```nim
let all = seq[string](h["Accept"])
```

---

### `HttpCode`

```nim
type HttpCode* = distinct range[0 .. 599]
```

A type-safe HTTP status code. The underlying integer is constrained to `0..599` at the type level. Using `distinct` means you cannot accidentally pass a bare integer where an `HttpCode` is expected or vice versa.

Named constants (e.g. `Http200`, `Http404`) are provided for all standard codes. You can also construct one directly: `HttpCode(418)`.

```nim
let code = Http404
echo code           # "404 Not Found"
echo code.int       # 404
echo code.is4xx     # true
```

---

### `HttpVersion`

```nim
type HttpVersion* = enum
  HttpVer11,   # HTTP/1.1
  HttpVer10    # HTTP/1.0
```

Enumerates the two HTTP versions recognised by this module. Used internally by `httpclient` and `asynchttpserver` to track which protocol version is in use for a connection.

```nim
let ver = HttpVer11
```

---

### `HttpMethod`

```nim
type HttpMethod* = enum
  HttpHead, HttpGet, HttpPost, HttpPut, HttpDelete,
  HttpTrace, HttpOptions, HttpConnect, HttpPatch
```

All nine standard HTTP request methods. Each variant's string representation (e.g. `"GET"`, `"POST"`) matches the wire format exactly, so `$HttpGet == "GET"`.

| Variant | Wire string | Typical use |
|---|---|---|
| `HttpGet` | `GET` | Retrieve a resource |
| `HttpPost` | `POST` | Submit data, create resource |
| `HttpPut` | `PUT` | Replace a resource |
| `HttpPatch` | `PATCH` | Partially modify a resource |
| `HttpDelete` | `DELETE` | Remove a resource |
| `HttpHead` | `HEAD` | Like GET but no response body |
| `HttpOptions` | `OPTIONS` | Query supported methods |
| `HttpConnect` | `CONNECT` | Open a TCP tunnel (proxies) |
| `HttpTrace` | `TRACE` | Echo request back (debugging) |

```nim
echo $HttpPost   # "POST"
let m = parseEnum[HttpMethod]("DELETE")  # HttpDelete
```

---

## Constants

### `httpNewLine`

```nim
const httpNewLine* = "\c\L"
```

The HTTP line terminator: carriage return followed by line feed (`\r\n`). HTTP/1.x requires this exact two-byte sequence to end every header line and the blank line separating headers from body. Always use this constant rather than `"\n"` when building raw HTTP messages.

```nim
let raw = "HTTP/1.1 200 OK" & httpNewLine &
          "Content-Length: 0" & httpNewLine &
          httpNewLine
```

---

### `headerLimit`

```nim
const headerLimit* = 10_000
```

The maximum number of **bytes** that header parsing will process. This guard prevents memory exhaustion attacks (HTTP request smuggling, slow header floods). If a header block exceeds this limit, the parser in `asynchttpserver` stops reading and rejects the request.

---

### HTTP status code constants

```nim
const Http200* = HttpCode(200)
const Http404* = HttpCode(404)
# ... and so on for every standard code
```

Named constants are provided for all registered HTTP status codes from 100 to 511. They exist purely for readability — `Http404` is clearer than `HttpCode(404)` in routing or response logic.

```nim
if response.code == Http301:
  let location = response.headers["Location"]
  redirect(location)
```

The full list covers 1xx (Informational), 2xx (Success), 3xx (Redirection), 4xx (Client Error) and 5xx (Server Error) families, including WebDAV extensions (102, 207, 208, 423, 424, 507, 508), Early Hints (103), Delta Encoding (226), and others.

---

## HttpHeaders — construction

### `newHttpHeaders` (empty)

```nim
func newHttpHeaders*(titleCase = false): HttpHeaders
```

**Creates an empty `HttpHeaders` object.** The internal table has no entries. Pass `titleCase = true` when you need keys to be stored and serialised in HTTP Title-Case rather than lowercase.

```nim
let h = newHttpHeaders()                   # lowercase keys
let h2 = newHttpHeaders(titleCase = true)  # Title-Case keys
h2["content-type"] = "text/plain"
echo h2["content-type"]  # stored as "Content-Type" internally
```

---

### `newHttpHeaders` (from pairs)

```nim
func newHttpHeaders*(keyValuePairs: openArray[tuple[key: string, val: string]],
                     titleCase = false): HttpHeaders
```

**Creates a pre-populated `HttpHeaders` object** from an array of `(key, value)` pairs. Duplicate keys are handled correctly: if the same key appears multiple times in the input array, all values are accumulated in order — no value is silently dropped.

```nim
let h = newHttpHeaders({
  "Accept": "text/html",
  "Accept": "application/json",   # second Accept — both are kept
  "Content-Type": "text/plain"
})
echo seq[string](h["Accept"])   # @["text/html", "application/json"]
```

---

## HttpHeaders — reading

### `[]` (single key)

```nim
func `[]`*(headers: HttpHeaders, key: string): HttpHeaderValues
```

**Returns all values for the given header key** as an `HttpHeaderValues` (a wrapped `seq[string]`). The lookup is always case-insensitive, regardless of how the key was originally stored. If the key does not exist, a `KeyError` is raised.

When the result is used in a `string` context, the implicit **converter** returns only the first value. To access all values explicitly, cast to `seq[string]` or use the two-argument form below.

```nim
let h = newHttpHeaders({"Accept": "text/html", "Accept": "application/json"})
let first: string = h["Accept"]             # "text/html" via converter
let all = seq[string](h["Accept"])          # @["text/html", "application/json"]
```

---

### `[]` (key + index)

```nim
func `[]`*(headers: HttpHeaders, key: string, i: int): string
```

**Returns the `i`-th value** (zero-based) for the given header key. Raises `KeyError` if the key doesn't exist, and `IndexDefect` if `i` is out of range.

```nim
let h = newHttpHeaders({"Accept": "text/html", "Accept": "application/json"})
echo h["Accept", 0]   # "text/html"
echo h["Accept", 1]   # "application/json"
```

---

### `getOrDefault`

```nim
func getOrDefault*(headers: HttpHeaders, key: string,
                   default = @[""].HttpHeaderValues): HttpHeaderValues
```

**Safe version of `[]`** — returns the values for `key` if it exists, or `default` if it does not. Unlike `[]`, it never raises an exception for a missing key. The default is `@[""]` (a single empty string wrapped in `HttpHeaderValues`), which converts to an empty string via the `toString` converter.

```nim
let h = newHttpHeaders()
let ct: string = h.getOrDefault("Content-Type")   # "" (no exception)
let custom = h.getOrDefault("X-My-Header", @["fallback"].HttpHeaderValues)
```

---

### `hasKey`

```nim
func hasKey*(headers: HttpHeaders, key: string): bool
```

**Returns `true`** if the header with the given key exists (case-insensitive). Use this before accessing with `[]` when you cannot afford an exception.

```nim
if headers.hasKey("Authorization"):
  let token = headers["Authorization"]
  authenticate(token)
```

---

### `contains` (HttpHeaderValues)

```nim
func contains*(values: HttpHeaderValues, value: string): bool
```

**Checks whether a specific string appears among the values** of an `HttpHeaderValues` collection. The comparison is **case-insensitive**, which matches HTTP semantics (e.g. `"gzip"` and `"GZIP"` are the same encoding token).

```nim
let h = newHttpHeaders({"Accept-Encoding": "gzip, deflate"})
# Note: the header is stored as two separate values after parsing
if "gzip" in h["Accept-Encoding"]:
  echo "Client accepts gzip"
```

---

### `len`

```nim
func len*(headers: HttpHeaders): int
```

**Returns the number of distinct header keys** currently stored. Note this counts keys, not total values — a header with three `Accept` values still contributes 1 to `len`.

```nim
let h = newHttpHeaders({"A": "1", "B": "2"})
echo h.len   # 2
```

---

### `pairs`

```nim
iterator pairs*(headers: HttpHeaders): tuple[key, value: string]
```

**Iterates over every individual key-value pair**, yielding one `(key, value)` tuple at a time. If a key has multiple values, it is yielded once per value. This is the right iterator to use when serialising headers to the wire format.

```nim
let h = newHttpHeaders({"Accept": "text/html", "Accept": "application/json",
                         "Content-Type": "text/plain"})
for key, val in h.pairs:
  echo key, ": ", val
# Accept: text/html
# Accept: application/json
# Content-Type: text/plain
```

---

### `$` (HttpHeaders)

```nim
func `$`*(headers: HttpHeaders): string
```

**Converts the headers object to its string representation** — identical to calling `$` on the underlying `TableRef`. Useful for debugging; the output is Nim table syntax, not a valid HTTP header block.

```nim
let h = newHttpHeaders({"X-Foo": "bar"})
echo h   # {"x-foo": @["bar"]}
```

---

## HttpHeaders — writing

### `[]=` (single value)

```nim
proc `[]=`*(headers: HttpHeaders, key, value: string)
```

**Sets the header to exactly one value**, completely replacing any existing values for that key. If you need to keep existing values and append a new one, use `add` instead.

```nim
let h = newHttpHeaders()
h["Content-Type"] = "text/html"
h["Content-Type"] = "application/json"  # replaces previous
echo h["Content-Type"]  # "application/json"
```

---

### `[]=` (seq of values)

```nim
proc `[]=`*(headers: HttpHeaders, key: string, value: seq[string])
```

**Sets the header to a list of values** all at once, replacing any existing entries. If the `value` sequence is **empty**, the header key is deleted from the table entirely — this is a convenient way to remove a header.

```nim
let h = newHttpHeaders()
h["Accept"] = @["text/html", "application/json"]
echo h["Accept", 0]   # "text/html"

h["Accept"] = @[]     # deletes the "Accept" header
echo h.hasKey("Accept")  # false
```

---

### `add`

```nim
proc add*(headers: HttpHeaders, key, value: string)
```

**Appends a value** to the list of values for `key`. If the key does not yet exist, it is created with this single value. Unlike `[]=`, existing values are preserved. This is how you represent multi-value headers like `Set-Cookie`.

```nim
let h = newHttpHeaders()
h.add("Set-Cookie", "session=abc; HttpOnly")
h.add("Set-Cookie", "tracking=xyz; Secure")
echo seq[string](h["Set-Cookie"])
# @["session=abc; HttpOnly", "tracking=xyz; Secure"]
```

---

### `del`

```nim
proc del*(headers: HttpHeaders, key: string)
```

**Removes all values** associated with the given key. Does nothing if the key does not exist (no exception). The lookup is case-insensitive.

```nim
let h = newHttpHeaders({"Authorization": "Bearer token123"})
h.del("Authorization")
echo h.hasKey("Authorization")  # false
```

---

### `clear`

```nim
proc clear*(headers: HttpHeaders)
```

**Removes all entries** from the headers object, leaving it in the same state as a freshly constructed empty `HttpHeaders`. The object itself is still valid and can be reused.

```nim
let h = newHttpHeaders({"X-A": "1", "X-B": "2"})
h.clear()
echo h.len   # 0
```

---

## HttpHeaders — internal

### `toCaseInsensitive`

```nim
func toCaseInsensitive*(headers: HttpHeaders, s: string): string
```

**Normalises a key string** according to the header object's mode: returns a lowercase version in the default mode, or a Title-Case version when `isTitleCase` is `true`. Documented as "for internal usage only" — you generally won't call this directly, but understanding it clarifies why header access is case-insensitive.

---

## HttpCode — utilities

### `$` (HttpCode)

```nim
func `$`*(code: HttpCode): string
```

**Converts an `HttpCode` to its full HTTP status line**, including the numeric code and the standard reason phrase. This is the format expected on the wire and in log messages.

```nim
echo $Http200   # "200 OK"
echo $Http404   # "404 Not Found"
echo $Http418   # "418 I'm a teapot"
echo $HttpCode(999)  # "999"  (unknown codes yield just the number)
```

Unknown codes (not in the standard table) are rendered as just their numeric string with no reason phrase.

---

### `==` (HttpCode)

```nim
func `==`*(a, b: HttpCode): bool
```

**Equality comparison** between two `HttpCode` values (borrowed from the underlying integer). Lets you write natural comparisons like `response.code == Http200`.

```nim
doAssert Http200 == HttpCode(200)
doAssert Http404 != Http500
```

---

### `is1xx`

```nim
func is1xx*(code: HttpCode): bool
```

**Returns `true`** when the code is in the 100–199 range (Informational responses). These indicate that the server has received the request and the client should continue.

```nim
doAssert is1xx(Http100)   # Continue
doAssert is1xx(Http103)   # Early Hints
doAssert not is1xx(Http200)
```

---

### `is2xx`

```nim
func is2xx*(code: HttpCode): bool
```

**Returns `true`** when the code is in the 200–299 range (Successful responses). The most important family: the request was received, understood, and accepted.

```nim
if response.code.is2xx:
  processBody(response.body)
```

---

### `is3xx`

```nim
func is3xx*(code: HttpCode): bool
```

**Returns `true`** when the code is in the 300–399 range (Redirection). The client must take additional action — typically follow the `Location` header to a new URL.

```nim
if response.code.is3xx:
  let newUrl = response.headers["Location"]
  followRedirect(newUrl)
```

---

### `is4xx`

```nim
func is4xx*(code: HttpCode): bool
```

**Returns `true`** when the code is in the 400–499 range (Client errors). The request contains bad syntax or cannot be fulfilled. The problem is on the client side.

```nim
if response.code.is4xx:
  echo "Request error: ", $response.code
```

---

### `is5xx`

```nim
func is5xx*(code: HttpCode): bool
```

**Returns `true`** when the code is in the 500–599 range (Server errors). The server failed to fulfil a valid request. The problem is on the server side.

```nim
if response.code.is5xx:
  retryWithBackoff()
```

---

## HttpVersion — utilities

### `==` (protocol tuple vs HttpVersion)

```nim
func `==`*(protocol: tuple[orig: string, major, minor: int],
           ver: HttpVersion): bool
```

**Compares a parsed protocol descriptor tuple** (as returned by Nim's HTTP parsers in the form `(orig: "HTTP/1.1", major: 1, minor: 1)`) against an `HttpVersion` enum value. Lets you write readable version checks without manually comparing major/minor integers.

```nim
let proto = (orig: "HTTP/1.1", major: 1, minor: 1)
doAssert proto == HttpVer11
doAssert not (proto == HttpVer10)
```

---

## HttpMethod — utilities

### `contains` (set of HttpMethod)

```nim
func contains*(methods: set[HttpMethod], x: string): bool
```

**Checks whether a string method name** (e.g. `"GET"`, `"POST"`) belongs to a given set of `HttpMethod` values. This enables the `in` operator with string method names on the left-hand side, which is useful when routing requests.

```nim
let readOnly = {HttpGet, HttpHead, HttpOptions}
if "POST" in readOnly:
  echo "unexpected"
if "GET" in readOnly:
  echo "GET is read-only"   # prints this
```

---

## Parsing

### `parseHeader`

```nim
func parseHeader*(line: string): tuple[key: string, value: seq[string]]
```

**Parses a single raw HTTP header line** into a `(key, value)` tuple. The key is everything before the first `:`, and the values are the comma-separated tokens that follow (after stripping leading whitespace). The `Cookie` header is treated specially: it is kept as a single unsplit value because cookies use `;` as separator, not `,`.

Intended for internal use by `asynchttpserver` and `httpclient`. You would only call this directly if you are implementing a low-level HTTP parser.

```nim
let h = parseHeader("Content-Type: text/html, application/xhtml+xml")
echo h.key           # "Content-Type"
echo h.value         # @["text/html", "application/xhtml+xml"]

let c = parseHeader("Cookie: session=abc; user=john")
echo c.key           # "Cookie"
echo c.value         # @["session=abc; user=john"]  (not split)
```

---

## Converter

### `toString`

```nim
converter toString*(values: HttpHeaderValues): string
```

**Implicit conversion** from `HttpHeaderValues` to `string` that extracts the first value. This converter is what makes single-value header access ergonomic — you can assign `headers["Content-Type"]` directly to a `string` variable without an explicit cast. However, be aware that if a header has multiple values, all but the first are silently ignored by this path.

```nim
let h = newHttpHeaders({"Content-Length": "128"})
let len: string = h["Content-Length"]   # converter fires automatically
# Equivalent (explicit) form:
let len2 = seq[string](h["Content-Length"])[0]
```

---

## Complete usage example

```nim
import std/httpcore

# Build response headers
let headers = newHttpHeaders({
  "Content-Type": "application/json",
  "Cache-Control": "no-store"
}, titleCase = true)

headers.add("Set-Cookie", "session=abc; HttpOnly; Secure")
headers.add("Set-Cookie", "lang=en")

# Read headers
echo headers["Content-Type"]          # "application/json"
echo headers["Set-Cookie", 1]         # "lang=en"
echo headers.len                       # 3 distinct keys

# Iterate for wire serialisation
for key, val in headers.pairs:
  echo key & ": " & val

# Status code handling
proc handleResponse(code: HttpCode) =
  if code.is2xx:
    echo "Success: ", $code
  elif code.is3xx:
    echo "Redirect: ", $code
  elif code.is4xx:
    echo "Client error: ", $code
  elif code.is5xx:
    echo "Server error: ", $code

handleResponse(Http200)   # Success: 200 OK
handleResponse(Http301)   # Redirect: 301 Moved Permanently
handleResponse(Http503)   # Server error: 503 Service Unavailable

# Method routing
let safeMethods = {HttpGet, HttpHead, HttpOptions}
if "DELETE" in safeMethods:
  echo "not reached"
if "GET" in safeMethods:
  echo "GET is safe"
```

---

*Reference covers Nim standard library `std/httpcore` as found in the module source.*
