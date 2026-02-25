# `jsfetch` Module Reference

> **Module:** `std/jsfetch` (Nim standard library)  
> **Target:** JavaScript backend **only** — using this module with a native Nim backend is a fatal compile-time error.  
> **Purpose:** Idiomatic Nim bindings for the browser [Fetch API](https://developer.mozilla.org/docs/Web/API/Fetch_API), enabling HTTP requests from Nim code compiled to JavaScript.  
> **Re-exports:** `std/jsformdata`, `std/jsheaders`

---

## Background: The Fetch API in Nim

The browser's Fetch API is the modern standard for making HTTP requests from JavaScript. It is Promise-based (asynchronous), replaces the older `XMLHttpRequest`, and is supported in all modern browsers and Node.js.

This module maps the Fetch API to Nim idioms: JavaScript Promises become Nim `Future[T]` values from `std/asyncjs`, raw `cstring` options are replaced by typed enums, and request configuration is expressed through a structured `FetchOptions` object. The result is HTTP networking code that looks natural in Nim while compiling cleanly to standard JavaScript `fetch()` calls.

**Key concepts to understand before using this module:**

- All network calls are **asynchronous** — they return `Future[Response]` and must be `await`-ed inside an `{.async.}` proc.
- The module works in browsers and in Node.js (with the `node-fetch` polyfill or Node.js 18+).
- `GET` and `HEAD` requests silently drop any `body` you provide — this matches the HTTP specification and is enforced by `newFetchOptions`.

---

## Exported Types

---

### `FetchOptions`

```nim
type FetchOptions* = ref object of JsRoot
  keepalive*: bool
  metod* {.importjs: "method".}: cstring
  body*, integrity*, referrer*, mode*, credentials*, cache*, redirect*, referrerPolicy*: cstring
  headers*: Headers
```

**What it is**

A ref object holding all configurable parameters of a `fetch()` call, corresponding directly to the JavaScript [`RequestInit`](https://developer.mozilla.org/en-US/docs/Web/API/fetch#options) dictionary. You typically do not construct this by hand — use `newFetchOptions` or `unsafeNewFetchOptions` instead.

**Fields**

| Field | Type | Description |
|-------|------|-------------|
| `metod` | `cstring` | HTTP method (`"GET"`, `"POST"`, etc.). Named `metod` (not `method`) because `method` is a Nim keyword. Maps to JavaScript `method`. |
| `body` | `cstring` | Request body. Automatically `nil` for `GET` and `HEAD` when using `newFetchOptions`. |
| `mode` | `cstring` | CORS mode — one of the `FetchModes` enum values. |
| `credentials` | `cstring` | How cookies and auth headers are handled — one of `FetchCredentials`. |
| `cache` | `cstring` | Cache behaviour — one of `FetchCaches`. |
| `redirect` | `cstring` | How redirects are handled — one of `FetchRedirects`. |
| `referrer` | `cstring` | Referrer URL or `"client"` to use the default. |
| `referrerPolicy` | `cstring` | Referrer policy — one of `FetchReferrerPolicies`. |
| `integrity` | `cstring` | Subresource integrity hash, or `""` to skip verification. |
| `keepalive` | `bool` | If `true`, the request persists beyond page unload (useful for analytics beacons). |
| `headers` | `Headers` | Request headers, from `std/jsheaders`. |

---

### `Response`

```nim
type Response* = ref object of JsRoot
  bodyUsed*, ok*, redirected*: bool
  typ* {.importjs: "type".}: cstring
  url*, statusText*: cstring
  status*: cint
  headers*: Headers
  body*: cstring
```

**What it is**

Represents the response to a `fetch()` call, wrapping the browser's [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) object. The fields are read-only in practice (modifying them has no effect on the underlying JS object).

**Fields**

| Field | Type | Description |
|-------|------|-------------|
| `ok` | `bool` | `true` if `status` is in the 200–299 range. The most common field to check first. |
| `status` | `cint` | HTTP status code (e.g. `200`, `404`, `500`). |
| `statusText` | `cstring` | Human-readable status message (e.g. `"OK"`, `"Not Found"`). |
| `url` | `cstring` | The final URL after any redirects. |
| `redirected` | `bool` | `true` if the response is the result of a redirect. |
| `headers` | `Headers` | Response headers. |
| `body` | `cstring` | Raw body as `cstring`. For structured access, use `.text()`, `.json()`, or `.formData()`. |
| `bodyUsed` | `bool` | `true` if the body has already been read (body streams can only be consumed once). |
| `typ` | `cstring` | Response type: `"basic"`, `"cors"`, `"opaque"`, etc. Named `typ` because `type` is a Nim keyword. |

---

### `Request`

```nim
type Request* = ref object of JsRoot
  bodyUsed*, ok*, redirected*: bool
  typ* {.importjs: "type".}: cstring
  url*, statusText*: cstring
  status*: cint
  headers*: Headers
  body*: cstring
```

**What it is**

Represents an HTTP request object, wrapping the browser's [`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request). Shares the same field layout as `Response`. Used when you want to construct and inspect a request independently of sending it — for example, to clone it or inspect its headers before dispatching.

---

### `FetchModes`

```nim
type FetchModes* = enum
  fmCors = "cors"
  fmNoCors = "no-cors"
  fmSameOrigin = "same-origin"
```

Controls how the browser handles cross-origin requests:

| Value | Meaning |
|-------|---------|
| `fmCors` | Standard CORS — the server must include appropriate `Access-Control-Allow-*` headers. Default for most requests. |
| `fmNoCors` | Restricts the request to a safe subset of methods and headers. The response will be "opaque" — you cannot read its body or status. Useful for fire-and-forget requests to third-party endpoints. |
| `fmSameOrigin` | Fails if the request crosses an origin boundary. Use this when you intentionally want to assert same-origin. |

---

### `FetchCredentials`

```nim
type FetchCredentials* = enum
  fcInclude = "include"
  fcSameOrigin = "same-origin"
  fcOmit = "omit"
```

Controls whether cookies, HTTP authentication, and TLS client certificates are sent:

| Value | Meaning |
|-------|---------|
| `fcInclude` | Always send credentials, including cross-origin requests. |
| `fcSameOrigin` | Send credentials only for same-origin requests. Default. |
| `fcOmit` | Never send credentials. |

---

### `FetchCaches`

```nim
type FetchCaches* = enum
  fchDefault = "default"
  fchNoStore = "no-store"
  fchReload = "reload"
  fchNoCache = "no-cache"
  fchForceCache = "force-cache"
```

Controls interaction with the browser's HTTP cache:

| Value | Meaning |
|-------|---------|
| `fchDefault` | Standard cache behaviour: use cached response if fresh, revalidate if stale. |
| `fchNoStore` | Never read from or write to the cache. Always goes to the network. |
| `fchReload` | Always goes to the network, but stores the result in cache. |
| `fchNoCache` | Always revalidates with the server, even if the cached response is fresh. |
| `fchForceCache` | Use the cached response even if stale; only goes to network if there is no cached entry. |

---

### `FetchRedirects`

```nim
type FetchRedirects* = enum
  frFollow = "follow"
  frError = "error"
  frManual = "manual"
```

Controls what happens when the server responds with a redirect:

| Value | Meaning |
|-------|---------|
| `frFollow` | Automatically follow redirects (up to the browser's limit). Default. |
| `frError` | Reject the `Future` with an error if a redirect is encountered. |
| `frManual` | Do not follow the redirect; return an "opaque redirect" response for manual handling. |

---

### `FetchReferrerPolicies`

```nim
type FetchReferrerPolicies* = enum
  frpNoReferrer = "no-referrer"
  frpNoReferrerWhenDowngrade = "no-referrer-when-downgrade"
  frpOrigin = "origin"
  frpOriginWhenCrossOrigin = "origin-when-cross-origin"
  frpUnsafeUrl = "unsafe-url"
```

Controls what information is sent in the `Referer` HTTP header:

| Value | Meaning |
|-------|---------|
| `frpNoReferrer` | No `Referer` header is sent at all. |
| `frpNoReferrerWhenDowngrade` | Send the full URL for same-protocol requests; omit for HTTPS→HTTP downgrades. Default. |
| `frpOrigin` | Send only the origin (scheme + host + port), never the path. |
| `frpOriginWhenCrossOrigin` | Send full URL for same-origin; only origin for cross-origin. |
| `frpUnsafeUrl` | Always send the full URL. Can leak private paths to third-party sites — use with caution. |

---

## Exported Functions and Procedures

---

### `newFetchOptions`

```nim
func newFetchOptions*(
  metod = HttpGet;
  body: cstring = nil;
  mode = fmCors;
  credentials = fcSameOrigin;
  cache = fchDefault;
  referrerPolicy = frpNoReferrerWhenDowngrade;
  keepalive = false;
  redirect = frFollow;
  referrer = "client".cstring;
  integrity = "".cstring;
  headers: Headers = newHeaders()
): FetchOptions
```

**What it does**

The **safe, recommended** constructor for `FetchOptions`. All parameters have sensible defaults matching the Fetch API specification, so you only need to specify what differs from the defaults.

The key safety behavior: if `metod` is `HttpGet` or `HttpHead`, the `body` is automatically set to `nil` regardless of what you pass. This matches the HTTP specification (GET and HEAD requests must not have a body) and prevents a class of subtle bugs.

All enum parameters (`mode`, `credentials`, `cache`, `referrerPolicy`, `redirect`) are converted to their string values automatically — you work with type-safe Nim enums and never hand-write the strings yourself.

**Parameters**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `metod` | `HttpGet` | HTTP method from `std/httpcore`. |
| `body` | `nil` | Request body. Ignored for GET/HEAD. |
| `mode` | `fmCors` | CORS mode. |
| `credentials` | `fcSameOrigin` | Credential policy. |
| `cache` | `fchDefault` | Cache policy. |
| `referrerPolicy` | `frpNoReferrerWhenDowngrade` | Referrer policy. |
| `keepalive` | `false` | Keep request alive beyond page unload. |
| `redirect` | `frFollow` | Redirect policy. |
| `referrer` | `"client"` | Referrer value. |
| `integrity` | `""` | Subresource integrity hash. |
| `headers` | `newHeaders()` | Request headers. |

**Example: simple GET (all defaults)**

```nim
import std/jsfetch

let opts = newFetchOptions()   # GET, cors, same-origin credentials, default cache
```

**Example: POST with JSON body**

```nim
import std/[jsfetch, jsheaders]
from std/httpcore import HttpPost

var h = newHeaders()
h["Content-Type"] = "application/json"

let opts = newFetchOptions(
  metod = HttpPost,
  body = """{"username": "alice", "password": "secret"}""".cstring,
  headers = h
)
```

**Example: POST — body is automatically dropped for GET**

```nim
from std/httpcore import HttpGet

let opts = newFetchOptions(
  metod = HttpGet,
  body = "this will be nil".cstring   # silently set to nil
)
assert opts.body == nil
```

---

### `unsafeNewFetchOptions`

```nim
proc unsafeNewFetchOptions*(
  metod, body, mode, credentials, cache, referrerPolicy: cstring;
  keepalive: bool;
  redirect = "follow".cstring;
  referrer = "client".cstring;
  integrity = "".cstring;
  headers: Headers = newHeaders()
): FetchOptions
```

**What it does**

An alternative constructor for `FetchOptions` that accepts raw `cstring` values for all fields instead of typed enums. It performs **no validation** — whatever strings you pass go directly into the resulting object. Passing an invalid string (e.g. `"PSOT"` instead of `"POST"`) will result in a runtime error from the browser, not a compile-time error from Nim.

This exists as an escape hatch for cases where the typed constructor is too restrictive — for example, if the Fetch API gains new option values that are not yet reflected in the Nim enums.

**When to prefer `newFetchOptions` instead:** almost always. Use `unsafeNewFetchOptions` only when you specifically need to pass a raw string value not covered by the existing enums, or when integrating with code that already deals in raw strings.

**Example**

```nim
import std/jsfetch

let opts = unsafeNewFetchOptions(
  metod = "POST".cstring,
  body = """{"key": "value"}""".cstring,
  mode = "no-cors".cstring,
  credentials = "omit".cstring,
  cache = "no-cache".cstring,
  referrerPolicy = "no-referrer".cstring,
  keepalive = false
)
assert opts.metod == "POST".cstring
assert opts.mode == "no-cors".cstring
```

---

### `newResponse`

```nim
func newResponse*(body: cstring | FormData): Response
```

**What it does**

Constructs a new `Response` object with the given body, directly calling JavaScript's `new Response(body)`. This does **not** make any network request — it creates a synthetic response in memory, identical to what `new Response(...)` does in JavaScript.

The primary use case is in service workers and testing scenarios, where you want to fabricate a response to return from a cache or intercept handler without actually going to the network.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | `cstring` or `FormData` | The body content for the synthetic response. |

**Example**

```nim
import std/jsfetch

# Synthetic text response:
let r = newResponse("-. .. --".cstring)

# Synthetic response from form data:
import std/jsformdata
var fd = newFormData()
fd.append("field", "value")
let r2 = newResponse(fd)
```

---

### `newRequest` (URL only)

```nim
func newRequest*(url: cstring): Request
```

**What it does**

Constructs a `Request` object for the given URL with no additional options, calling `new Request(url)` in JavaScript. Does not send anything to the network.

Useful when you want to pass a `Request` object directly to `fetch()`, or inspect/clone it before dispatching.

**Example**

```nim
import std/jsfetch

let req = newRequest("https://api.example.com/data".cstring)
```

---

### `newRequest` (URL + options)

```nim
func newRequest*(url: cstring; fetchOptions: FetchOptions): Request
```

**What it does**

Constructs a `Request` object combining a URL with a `FetchOptions` configuration, calling `new Request(url, fetchOptions)`. Equivalent in effect to calling `fetch(url, fetchOptions)` — use this variant when you need a reusable, self-contained request object (for example, to clone it and send multiple times).

**Example**

```nim
import std/jsfetch
from std/httpcore import HttpPost

let opts = newFetchOptions(metod = HttpPost, body = "data".cstring)
let req = newRequest("https://api.example.com/submit".cstring, opts)
```

---

### `clone`

```nim
func clone*(self: Response | Request): Response
```

**What it does**

Creates an identical copy of a `Response` or `Request` object. This is necessary because body streams can only be read **once** — after calling `.text()`, `.json()`, or `.formData()` on a `Response`, its body is consumed and cannot be read again. Cloning before reading lets you keep a pristine copy.

Note the return type is always `Response` regardless of input — this matches the underlying JavaScript API, where `Request.clone()` also returns a `Response` in Nim's binding.

**Example**

```nim
import std/[jsfetch, asyncjs]

proc handleResponse(resp: Response) {.async.} =
  let copy = resp.clone()           # preserve original
  let text = await resp.text()      # consumes resp.body
  # resp.bodyUsed is now true — but copy is still fresh
  let text2 = await copy.text()     # reads from the clone
```

---

### `text`

```nim
proc text*(self: Response): Future[cstring]
```

**What it does**

Reads the response body as a plain text `cstring`. This is an asynchronous operation — the body is streamed from the network, so reading it returns a `Future`. Wraps `Response.prototype.text()`.

**Important:** The body can only be read once. After `await resp.text()`, `resp.bodyUsed` becomes `true` and further reads will fail. Use `clone()` if you need to read the body multiple times.

**Example**

```nim
import std/[jsfetch, asyncjs]

proc fetchText() {.async.} =
  let resp = await fetch("https://example.com/data.txt".cstring)
  if resp.ok:
    let content = await resp.text()
    echo $content
```

---

### `json`

```nim
proc json*(self: Response): Future[JsObject]
```

**What it does**

Reads the response body and parses it as JSON, returning the parsed value as a `JsObject`. Wraps `Response.prototype.json()`. The parsing is done by the JavaScript engine — if the body is not valid JSON, the returned `Future` will be rejected with a `SyntaxError`.

`JsObject` from `std/jsffi` is a loosely-typed JavaScript object. You can access its properties using `[]` indexing or convert it to a typed Nim object with `to()`.

**Example**

```nim
import std/[jsfetch, asyncjs, jsffi]

proc fetchJson() {.async.} =
  let resp = await fetch("https://api.github.com/users/nim-lang".cstring)
  if resp.ok:
    let data = await resp.json()
    echo $data["login"]   # access property on JsObject
```

---

### `formData`

```nim
proc formData*(self: Response): Future[FormData]
```

**What it does**

Reads the response body and parses it as `multipart/form-data` or `application/x-www-form-urlencoded`, returning a `FormData` object. Wraps `Response.prototype.formData()`. Useful when a server endpoint returns form-encoded data rather than JSON.

**Example**

```nim
import std/[jsfetch, asyncjs, jsformdata]

proc fetchForm() {.async.} =
  let resp = await fetch("https://example.com/form-endpoint".cstring)
  if resp.ok:
    let fd = await resp.formData()
    echo $fd.get("field-name")
```

---

### `fetch` (simple GET)

```nim
proc fetch*(url: cstring | Request): Future[Response]
```

**What it does**

Sends an HTTP `GET` request to `url` with all default options and returns a `Future[Response]`. This is the simplest possible way to make a network request. Wraps JavaScript's `fetch(url)`.

The `Future` resolves to a `Response` when the response headers arrive (not when the full body is downloaded). The `Future` is rejected only for network errors (no connection, DNS failure) — HTTP error codes like 404 and 500 still resolve successfully; check `response.ok` or `response.status` to detect them.

**Example**

```nim
import std/[jsfetch, asyncjs]

proc loadData() {.async.} =
  let resp = await fetch("https://httpbin.org/get".cstring)
  assert resp.ok
  assert resp.status == 200.cint
  let body = await resp.text()
  echo $body
```

---

### `fetch` (with options)

```nim
proc fetch*(url: cstring | Request; options: FetchOptions): Future[Response]
```

**What it does**

Sends an HTTP request to `url` with the configuration specified in `options`. This is the full-featured variant — use it for anything other than a plain GET. Wraps JavaScript's `fetch(url, options)`.

**Example: POST with JSON**

```nim
import std/[jsfetch, jsheaders, asyncjs]
from std/httpcore import HttpPost

proc postJson() {.async.} =
  var h = newHeaders()
  h["Content-Type"] = "application/json"

  let opts = newFetchOptions(
    metod = HttpPost,
    body = """{"name": "nim"}""".cstring,
    headers = h
  )
  let resp = await fetch("https://httpbin.org/post".cstring, opts)
  assert resp.ok
  let data = await resp.json()
  echo $data
```

**Example: chaining with `.then()` (Promise-style)**

```nim
import std/[jsfetch, asyncjs, jsconsole, jsffi]
from std/sugar import `=>`

proc example() {.async.} =
  await fetch("https://api.github.com/users/torvalds".cstring)
    .then((response: Response) => response.json())
    .then((json: JsObject) => console.log(json))
    .catch((err: Error) => console.log("Request Failed", err))

discard example()
```

---

### `toCstring`

```nim
func toCstring*(self: Request | Response | FetchOptions): cstring
```

**What it does**

Serializes any of the three main types to a JSON `cstring` by calling JavaScript's `JSON.stringify()` on the underlying object. Useful for debugging, logging, or passing the configuration as a string to other JavaScript interop code.

**Example**

```nim
import std/jsfetch

let opts = newFetchOptions()
echo $opts.toCstring()
# {"method":"GET","mode":"cors","credentials":"same-origin",...}
```

---

### `$`

```nim
func `$`*(self: Request | Response | FetchOptions): string
```

**What it does**

Nim's stringify operator for `Request`, `Response`, and `FetchOptions`. Returns a Nim `string` containing the JSON representation of the object. Implemented as `$toCstring(self)` — a thin wrapper around `toCstring`.

**Example**

```nim
import std/jsfetch

let opts = newFetchOptions()
echo $opts    # prints JSON representation
```

---

## Complete Usage Examples

### Minimal GET request

```nim
import std/[jsfetch, asyncjs]

proc getData() {.async.} =
  let resp = await fetch("https://httpbin.org/get".cstring)
  if resp.ok:
    let text = await resp.text()
    echo $text
  else:
    echo "Error: ", $resp.status

discard getData()
```

### POST with JSON body and custom headers

```nim
import std/[jsfetch, jsheaders, asyncjs, jsffi]
from std/httpcore import HttpPost

proc createUser() {.async.} =
  var headers = newHeaders()
  headers["Content-Type"] = "application/json"
  headers["Authorization"] = "Bearer my-token"

  let opts = newFetchOptions(
    metod = HttpPost,
    body = """{"name": "Alice", "email": "alice@example.com"}""".cstring,
    headers = headers
  )

  let resp = await fetch("https://api.example.com/users".cstring, opts)
  case resp.status
  of 201.cint:
    let user = await resp.json()
    echo "Created: ", $user["id"]
  of 400.cint:
    echo "Bad request: ", $resp.statusText
  else:
    echo "Unexpected status: ", $resp.status

discard createUser()
```

### Reading the same response body twice (using `clone`)

```nim
import std/[jsfetch, asyncjs]

proc inspectResponse() {.async.} =
  let resp = await fetch("https://example.com/data".cstring)
  let copy = resp.clone()

  let raw = await resp.text()
  echo "Raw body length: ", raw.len

  # resp is now consumed, but copy is still usable:
  let again = await copy.text()
  echo "Copy: ", $again

discard inspectResponse()
```

---

## Summary Table

| Symbol | Kind | Purpose |
|--------|------|---------|
| `FetchOptions` | type | Configuration for a `fetch()` call |
| `Response` | type | Result of a `fetch()` call |
| `Request` | type | Self-contained HTTP request object |
| `FetchModes` | enum | CORS mode options (`cors`, `no-cors`, `same-origin`) |
| `FetchCredentials` | enum | Credential policy (`include`, `same-origin`, `omit`) |
| `FetchCaches` | enum | Cache policy (5 options) |
| `FetchRedirects` | enum | Redirect policy (`follow`, `error`, `manual`) |
| `FetchReferrerPolicies` | enum | Referrer policy (5 options) |
| `newFetchOptions` | func | **Safe** options constructor with typed enums and defaults |
| `unsafeNewFetchOptions` | proc | **Unsafe** options constructor with raw strings |
| `newResponse` | func | Create a synthetic `Response` (no network call) |
| `newRequest` (×2) | func | Create a `Request` object, optionally with options |
| `clone` | func | Copy a `Response`/`Request` before consuming its body |
| `text` | proc | Read response body as `Future[cstring]` |
| `json` | proc | Read response body as `Future[JsObject]` |
| `formData` | proc | Read response body as `Future[FormData]` |
| `fetch` (×2) | proc | Send the HTTP request; returns `Future[Response]` |
| `toCstring` | func | JSON-serialize to `cstring` |
| `$` | func | JSON-serialize to `string` |
