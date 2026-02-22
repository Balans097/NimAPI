# Nim `std/uri` Module Reference

> Implements URI parsing as specified by **RFC 3986**.
> A URI (Uniform Resource Identifier) uniquely identifies a resource on the internet.
> A URL (Uniform Resource Locator) is a subset of URIs that additionally specifies how to access the resource.
>
> ⚠️ **Security note:** The parsers in this module do **not** perform security validation. Always validate and sanitize URIs in security-sensitive contexts yourself.

---

## Table of Contents

1. [Types](#types)
2. [uriParseError](#uriparseerror)
3. [initUri](#inituri)
4. [parseUri (in-place)](#parseuri-in-place)
5. [parseUri (return value)](#parseuri-return-value)
6. [encodeUrl](#encodeurl)
7. [decodeUrl](#decodeurl)
8. [encodeQuery](#encodequery)
9. [decodeQuery](#decodequery)
10. [isAbsolute](#isabsolute)
11. [combine (two URIs)](#combine-two-uris)
12. [combine (varargs)](#combine-varargs)
13. [/ operator](#-operator)
14. [? operator](#-operator-1)
15. [$ operator](#-operator-2)
16. [getDataUri](#getdatauri)

---

## Types

### `Uri`

The central type of this module. It represents a parsed URI broken down into its individual components according to the RFC 3986 standard URI structure:

```
scheme://username:password@hostname:port/path?query#anchor
```

```nim
type Uri* = object
  scheme*:   string   # "https", "ftp", "mailto", ...
  username*: string   # user info before '@'
  password*: string   # password after ':' in user info
  hostname*: string   # domain or IP address
  port*:     string   # port number as string (e.g. "8080")
  path*:     string   # resource path (e.g. "/docs/index.html")
  query*:    string   # everything after '?' (e.g. "q=nim&lang=en")
  anchor*:   string   # fragment after '#' (e.g. "section-3")
  opaque*:   bool     # true for URIs like "mailto:user@example.com"
  isIpv6*:   bool     # true when hostname is an IPv6 address
```

**Example — what gets parsed into what:**
```nim
import std/uri

let u = parseUri("https://alice:secret@example.com:443/path?q=1#top")
assert u.scheme   == "https"
assert u.username == "alice"
assert u.password == "secret"
assert u.hostname == "example.com"
assert u.port     == "443"
assert u.path     == "/path"
assert u.query    == "q=1"
assert u.anchor   == "top"
```

---

### `Url`

A `distinct string` type that wraps a URL string to make it type-safe and prevent accidental mixing with plain strings.

```nim
type Url* = distinct string
```

---

### `UriParseError`

An exception type (inherits from `ValueError`) raised when the URI parser encounters invalid input.

```nim
type UriParseError* = object of ValueError
```

---

## `uriParseError`

```nim
proc uriParseError*(msg: string) {.noreturn.}
```

**What it does:** Raises a `UriParseError` exception with the given message. This is a convenience helper used internally by the module and available for custom validation code. Because it is marked `{.noreturn.}`, the compiler knows that control never returns from this call — which helps with control flow analysis.

**When to use it:** When you need to signal that a URI string failed your own validation logic, using the same exception type that the module itself uses.

```nim
import std/uri

proc validateScheme(uri: Uri) =
  if uri.scheme notin ["http", "https", "ftp"]:
    uriParseError("Unsupported scheme: " & uri.scheme)

try:
  validateScheme(parseUri("smtp://mail.example.com"))
except UriParseError as e:
  echo e.msg  # "Unsupported scheme: smtp"
```

---

## `initUri`

```nim
func initUri*(isIpv6 = false): Uri
```

**What it does:** Creates a blank `Uri` object with all string fields set to `""` and boolean fields set to `false`. The optional `isIpv6` parameter pre-sets the `isIpv6` flag so that when you later convert the URI back to a string with `$`, the hostname will be correctly wrapped in square brackets `[...]` as required by RFC 3986.

**When to use it:** Use `initUri` when you want to build a URI programmatically, field by field, instead of parsing it from a string.

```nim
import std/uri

# Build an IPv6 URI manually
var u = initUri(isIpv6 = true)
u.scheme   = "tcp"
u.hostname = "2001:db8::1"
u.port     = "9000"

echo $u  # tcp://[2001:db8::1]:9000
```

---

## `parseUri` (in-place)

```nim
func parseUri*(uri: string, result: var Uri)
```

**What it does:** Parses the URI string `uri` into an **existing** `Uri` variable passed by reference. The target variable is **cleared** before parsing begins, so any previously stored data is discarded. This overload avoids allocating a new `Uri` object on every call and is therefore more efficient in tight loops.

**When to use it:** When you parse many URIs in sequence and want to reuse the same variable to reduce allocations.

```nim
import std/uri

var res = initUri()

parseUri("https://nim-lang.org/docs/manual.html", res)
assert res.scheme   == "https"
assert res.hostname == "nim-lang.org"
assert res.path     == "/docs/manual.html"

# Reuse the same variable — it gets cleared automatically
parseUri("ftp://files.example.com", res)
assert res.scheme   == "ftp"
assert res.hostname == "files.example.com"
assert res.path     == ""
```

---

## `parseUri` (return value)

```nim
func parseUri*(uri: string): Uri
```

**What it does:** Parses the URI string `uri` and **returns** a new `Uri` object. This is the most convenient form for one-off parsing. Internally it calls `initUri()` and then the in-place overload.

**When to use it:** For straightforward, readable code where performance is not the primary concern.

```nim
import std/uri

let u = parseUri("ftp://Alice:s3cr3t@files.example.com/data")
assert u.scheme   == "ftp"
assert u.username == "Alice"
assert u.password == "s3cr3t"
assert u.hostname == "files.example.com"
assert u.path     == "/data"
```

**Relative URIs** (without a scheme) are also handled:

```nim
let rel = parseUri("/docs/index.html")
assert rel.scheme   == ""
assert rel.hostname == ""
assert rel.path     == "/docs/index.html"
```

---

## `encodeUrl`

```nim
func encodeUrl*(s: string, usePlus = true): string
```

**What it does:** Percent-encodes a string according to RFC 3986. Characters that are "unreserved" (`a-z`, `A-Z`, `0-9`, `-`, `.`, `_`, `~`) pass through unchanged. Every other character is replaced by `%XX` where `XX` is its two-digit hexadecimal code point. Spaces are a special case: when `usePlus = true` (the default), they become `+`; when `usePlus = false`, they become `%20`.

**When to use it:** Before placing arbitrary user input into a URL query string, path segment, or any other URL component where special characters would break the URL structure.

```nim
import std/uri

# Colons and slashes must be encoded when used as data, not as URL structure
echo encodeUrl("https://nim-lang.org")
# → "https%3A%2F%2Fnim-lang.org"

# Spaces become '+' by default (form encoding style)
echo encodeUrl("hello world")
# → "hello+world"

# Spaces become '%20' with usePlus = false (strict RFC 3986 style)
echo encodeUrl("hello world", usePlus = false)
# → "hello%20world"
```

---

## `decodeUrl`

```nim
func decodeUrl*(s: string, decodePlus = true): string
```

**What it does:** The inverse of `encodeUrl`. Replaces each `%XX` sequence with the character it represents. When `decodePlus = true` (the default), `+` characters are also converted to spaces. If a `%XX` sequence contains non-hexadecimal characters, it is left as-is rather than raising an error.

**When to use it:** When reading URL-encoded data — for example, query parameter values received from an HTTP request.

```nim
import std/uri

echo decodeUrl("hello+world")           # → "hello world"
echo decodeUrl("hello+world", false)    # → "hello+world"  ('+' is kept)
echo decodeUrl("caf%C3%A9")             # → "café"
echo decodeUrl("abc%xyz")               # → "abc%xyz"  (invalid sequence, kept intact)
```

---

## `encodeQuery`

```nim
func encodeQuery*(query: openArray[(string, string)],
                  usePlus = true,
                  omitEq  = true,
                  sep     = '&'): string
```

**What it does:** Takes a sequence of `(key, value)` pairs and assembles them into a URL query string. Each key and value is URL-encoded via `encodeUrl`. The pairs are joined by the separator character (default `&`). If a value is an empty string and `omitEq = true` (the default), the `=` sign is omitted, producing `key` rather than `key=`.

**When to use it:** When you have structured data that needs to become the query portion of a URL — for example, building a search request or a form submission URL.

```nim
import std/uri

echo encodeQuery({"q": "nim programming", "lang": "en"})
# → "q=nim+programming&lang=en"

# Empty value: '=' is omitted by default
echo encodeQuery({"debug": ""})
# → "debug"

# Keep the '=' even for empty values
echo encodeQuery({"debug": ""}, omitEq = false)
# → "debug="

# Use ';' as separator (alternative convention)
echo encodeQuery({"a": "1", "b": "2"}, sep = ';')
# → "a=1;b=2"
```

---

## `decodeQuery`

```nim
iterator decodeQuery*(data: string, sep = '&'): tuple[key, value: string]
```

**What it does:** An iterator that parses a URL query string and yields `(key, value)` pairs one at a time. Both keys and values are URL-decoded on the fly. The separator character (default `&`) determines where one pair ends and the next begins. Handles edge cases like missing values, empty keys, and `=` signs embedded inside values gracefully.

**When to use it:** When you receive a query string from a URL (for example, in a web server handler) and need to extract individual parameter values.

```nim
import std/uri

let qs = "name=Alice&city=New+York&verified"

for key, value in decodeQuery(qs):
  echo key, " → ", value
# name   → Alice
# city   → New York
# verified → ""

# Collect into a table
import std/tables
var params = newTable[string, string]()
for k, v in decodeQuery(qs):
  params[k] = v
```

---

## `isAbsolute`

```nim
func isAbsolute*(uri: Uri): bool
```

**What it does:** Returns `true` if the URI is absolute — meaning it has a non-empty `scheme` **and** either a non-empty `hostname` or a non-empty `path`. A relative URI (one without a scheme, like `/docs/index.html` or `../img/logo.png`) returns `false`.

**When to use it:** To decide whether a URI can stand on its own or needs to be resolved against a base URI.

```nim
import std/uri

assert parseUri("https://nim-lang.org").isAbsolute       # true
assert parseUri("ftp://files.example.com/x").isAbsolute  # true
assert not parseUri("/relative/path").isAbsolute          # false — no scheme
assert not parseUri("just-a-name").isAbsolute             # false
```

---

## `combine` (two URIs)

```nim
func combine*(base: Uri, reference: Uri): Uri
```

**What it does:** Merges a `reference` URI into a `base` URI following the resolution algorithm defined in [RFC 3986 §5.2.2](https://tools.ietf.org/html/rfc3986#section-5.2.2). The rules, in brief, are:

- If `reference` has its own scheme, it completely replaces `base`.
- If `reference` has an authority (hostname), it replaces `base`'s authority and path.
- If `reference` has an absolute path (starts with `/`), it replaces `base`'s path.
- If `reference` has a relative path, it is merged with `base`'s path by replacing everything after the last `/` in `base`'s path.

Dot segments (`.` and `..`) in the resulting path are removed.

**When to use it:** When resolving hyperlinks relative to a document's URL — exactly what a web browser does.

```nim
import std/uri

let base = parseUri("https://nim-lang.org/foo/bar")

# Absolute reference path replaces everything after the host
let a = combine(base, parseUri("/baz"))
assert a.path == "/baz"

# Relative reference is merged with base path (last segment "bar" is replaced)
let b = combine(base, parseUri("baz"))
assert b.path == "/foo/baz"

# Trailing slash on base means "baz" is appended
let c = combine(parseUri("https://nim-lang.org/foo/bar/"), parseUri("baz"))
assert c.path == "/foo/bar/baz"
```

---

## `combine` (varargs)

```nim
func combine*(uris: varargs[Uri]): Uri
```

**What it does:** A convenience overload that applies `combine` repeatedly to a list of URIs from left to right. Equivalent to chaining `combine(combine(a, b), c)` etc.

**When to use it:** When building a URI from several parts in sequence.

```nim
import std/uri

let result = combine(
  parseUri("https://nim-lang.org/"),
  parseUri("docs/"),
  parseUri("manual.html")
)
assert result.hostname == "nim-lang.org"
assert result.path     == "/docs/manual.html"
```

---

## `/` operator

```nim
func `/`*(x: Uri, path: string): Uri
```

**What it does:** Appends a path segment to the path of URI `x`. This operator is deliberately lenient about leading and trailing slashes — it normalizes them so that you never end up with double slashes (`//`) or a missing slash between segments. Unlike `combine`, you pass a plain string (not a `Uri`) on the right-hand side.

**When to use it:** For the most common case of building a URL by appending path segments. Prefer this over `combine` when you just want to extend a path.

```nim
import std/uri

let base = parseUri("https://nim-lang.org")

echo $(base / "docs" / "manual.html")
# → "https://nim-lang.org/docs/manual.html"

# Slash handling is automatic — no double slashes regardless of input
echo $(base / "/docs/")
# → "https://nim-lang.org/docs/"
echo $(parseUri("https://example.com/a/b") / "c")
# path = "/a/b/c"
echo $(parseUri("https://example.com/a/b/") / "c")
# path = "/a/b/c"  (trailing slash on base makes no difference here)
```

---

## `?` operator

```nim
func `?`*(u: Uri, query: openArray[(string, string)]): Uri
```

**What it does:** Attaches a query string to a URI. The `(key, value)` pairs are encoded with `encodeQuery` and set as the `query` field of the returned URI. This operator is designed to chain naturally with the `/` operator.

**When to use it:** As the final step of building a request URL when you need to add query parameters.

```nim
import std/uri

let url = parseUri("https://api.example.com") / "search" ? {"q": "Nim", "page": "2"}
echo $url
# → "https://api.example.com/search?q=Nim&page=2"
```

---

## `$` operator

```nim
func `$`*(u: Uri): string
```

**What it does:** Converts a `Uri` object back into its canonical string representation. It reassembles all the fields in the correct order, inserting the appropriate punctuation (`://`, `@`, `:`, `/`, `?`, `#`) between them. Fields that are empty strings are simply skipped. IPv6 hostnames are automatically wrapped in `[...]`.

**When to use it:** Any time you need to turn a `Uri` back into a string — for printing, sending over a network, or storing.

```nim
import std/uri

let u = parseUri("https://alice@example.com:8080/path?q=1#sec")
assert $u == "https://alice@example.com:8080/path?q=1#sec"

# Works for manually built URIs too
var v = initUri(isIpv6 = true)
v.scheme   = "https"
v.hostname = "::1"
v.port     = "443"
echo $v   # → "https://[::1]:443"
```

---

## `getDataUri`

```nim
proc getDataUri*(data, mime: string, encoding = "utf-8"): string
```

**What it does:** Encodes arbitrary binary or text data as a [Data URI](https://en.wikipedia.org/wiki/Data_URI_scheme) according to RFC 2397. The result is a self-contained URI of the form:

```
data:<mime>;charset=<encoding>;base64,<base64data>
```

The data is Base64-encoded (not the URL-safe variant, per RFC 2397). Data URIs are commonly used to embed images, fonts, or other resources directly into HTML or CSS without a separate HTTP request.

**When to use it:** To inline small assets (icons, SVGs, fonts) into HTML, or to embed binary content in JSON/XML without a separate file reference.

```nim
import std/uri

# Embed a small text payload
echo getDataUri("Hello, World!", "text/plain")
# → "data:text/plain;charset=utf-8;base64,SGVsbG8sIFdvcmxkIQ=="

# Embed an SVG inline in HTML
let svg = "<svg>...</svg>"
let dataUri = getDataUri(svg, "image/svg+xml")
# → "data:image/svg+xml;charset=utf-8;base64,PHN2Zz4uLi48L3N2Zz4="

# Nim itself
echo getDataUri("Nim", "text/plain")
# → "data:text/plain;charset=utf-8;base64,Tmlt"
```

---

## Quick Cheat-Sheet

| Task | API |
|---|---|
| Parse a URI string | `parseUri("https://...")` |
| Build a URI object | `initUri()` + set fields |
| Append a path | `uri / "segment"` |
| Add query params | `uri ? {"key": "value"}` |
| Resolve relative URI | `combine(base, reference)` |
| URI → string | `$uri` |
| Encode for URL | `encodeUrl(s)` |
| Decode from URL | `decodeUrl(s)` |
| Build query string | `encodeQuery(pairs)` |
| Parse query string | `for k, v in decodeQuery(qs)` |
| Check if absolute | `uri.isAbsolute` |
| Data URI | `getDataUri(data, mime)` |
