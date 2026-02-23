# `httpclient` Module Reference

> **Nim Standard Library — HTTP Client**
> Part of Nim's runtime library. © 2026 Nim Contributors.
> API status: **Unstable** in parts; core API is stable.

---

## Overview

`httpclient` provides a full-featured HTTP/1.1 client for both **synchronous** and **asynchronous** usage. A single import is all you need to make GET, POST, PUT, PATCH, DELETE, and HEAD requests, handle redirects, work with multipart form data, stream large files, track download progress, and connect through proxies.

The module exposes two client types that share an identical API surface:

- `HttpClient` — blocking, suitable for scripts and simple tools.
- `AsyncHttpClient` — non-blocking, built on `asyncdispatch`, suitable for servers and concurrent code.

```nim
import std/httpclient
```

For HTTPS, also compile with `-d:ssl`.

> ⚠️ **Security note:** URI parsers in this module do not detect malicious URIs. Always validate untrusted input before constructing request URLs.

---

## Types

### `Response` / `AsyncResponse`

The object returned by every request procedure. Both types expose identical fields and accessor procs.

```nim
type Response* = ref object
  version*: string      # HTTP version: "1.0" or "1.1"
  status*: string       # Status line, e.g. "200 OK"
  headers*: HttpHeaders # All response headers
  bodyStream*: Stream   # Raw body stream (sync)

type AsyncResponse* = ref object
  version*: string
  status*: string
  headers*: HttpHeaders
  bodyStream*: FutureStream[string]  # Raw body stream (async)
```

### `Proxy`

Holds proxy connection details. Constructed via `newProxy`.

### `MultipartData`

Holds form fields and file entries for `multipart/form-data` requests. Constructed via `newMultipartData`.

### `ProgressChangedProc`

Callback type for download progress reporting:

```nim
type ProgressChangedProc[ReturnType] =
  proc (total, progress, speed: BiggestInt): ReturnType {.closure, gcsafe.}
```

For `HttpClient` the return type is `void`; for `AsyncHttpClient` it is `Future[void]`.

### `ProtocolError`

Raised when the server's response does not conform to HTTP/1.x.

### `HttpRequestError`

Raised by `getContent`, `postContent`, and all other `*Content` procedures when the server returns a 4xx or 5xx status code.

---

## Constants

### `defUserAgent`

```nim
const defUserAgent* = "Nim-httpclient/" & NimVersion
```

The default `User-Agent` header value sent with every request. You can override it per-client via `newHttpClient(userAgent = "MyApp/1.0")`.

---

## Client Construction

### `newHttpClient`

```nim
proc newHttpClient*(
  userAgent    = defUserAgent,
  maxRedirects = 5,
  sslContext   = getDefaultSSL(),
  proxy        : Proxy = nil,
  timeout      = -1,
  headers      = newHttpHeaders()
): HttpClient
```

#### What it does

Creates a synchronous HTTP client. The client maintains a persistent TCP connection — subsequent requests to the same host reuse it without reconnecting. You must call `close()` when done.

#### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `userAgent` | `"Nim-httpclient/X.Y.Z"` | Value of the `User-Agent` request header. |
| `maxRedirects` | `5` | Maximum number of automatic redirects to follow. Set to `0` to disable. |
| `sslContext` | system default | OpenSSL context for HTTPS. Controls certificate verification mode. |
| `proxy` | `nil` | Optional HTTP or SOCKS5 proxy. |
| `timeout` | `-1` (no timeout) | Socket timeout in **milliseconds**. Applies to individual socket operations, not the total request. |
| `headers` | empty | Default headers sent with every request from this client. |

#### Example

```nim
import std/httpclient

# Minimal: one-liner GET
let html = newHttpClient().getContent("http://example.com")

# Configured client
let client = newHttpClient(
  userAgent    = "MyBot/2.0",
  timeout      = 5_000,   # 5-second socket timeout
  maxRedirects = 3
)
defer: client.close()
echo client.getContent("https://api.example.com/data")
```

---

### `newAsyncHttpClient`

```nim
proc newAsyncHttpClient*(
  userAgent    = defUserAgent,
  maxRedirects = 5,
  sslContext   = getDefaultSSL(),
  proxy        : Proxy = nil,
  headers      = newHttpHeaders()
): AsyncHttpClient
```

#### What it does

Creates an **asynchronous** HTTP client for use inside `async` procedures. Accepts the same parameters as `newHttpClient`, except `timeout` is not yet supported for async clients.

One `AsyncHttpClient` instance handles **one request at a time**. To make parallel requests, create multiple client instances.

#### Example

```nim
import std/[asyncdispatch, httpclient]

proc fetchAll(): Future[void] {.async.} =
  # Two clients → two parallel requests
  let c1 = newAsyncHttpClient()
  let c2 = newAsyncHttpClient()
  defer:
    c1.close()
    c2.close()

  let (r1, r2) = await all(
    c1.getContent("https://api.example.com/users"),
    c2.getContent("https://api.example.com/posts")
  )
  echo r1
  echo r2

waitFor fetchAll()
```

---

## Client Management

### `close`

```nim
proc close*(client: HttpClient | AsyncHttpClient)
```

#### What it does

Closes the underlying TCP socket and marks the client as disconnected. Should always be called when you are finished with a client — use `defer` to guarantee it even if an exception is raised.

```nim
let client = newHttpClient()
defer: client.close()   # runs even if the body raises
let body = client.getContent("https://example.com")
```

---

### `getSocket`

```nim
proc getSocket*(client: HttpClient): Socket
proc getSocket*(client: AsyncHttpClient): AsyncSocket
```

#### What it does

Returns the underlying network socket. Useful for inspecting low-level connection details such as local and remote addresses.

```nim
let client = newHttpClient()
defer: client.close()
discard client.get("http://example.com")

if client.connected:
  echo "Local:  ", client.getSocket.getLocalAddr
  echo "Remote: ", client.getSocket.getPeerAddr
```

---

## Response Accessors

These procedures work on both `Response` and `AsyncResponse`.

### `code`

```nim
proc code*(response: Response | AsyncResponse): HttpCode
```

Parses and returns the numeric HTTP status code from the response's `status` string. Raises `ValueError` if the status string is malformed.

```nim
let resp = client.get("https://httpbin.org/status/404")
echo resp.code        # Http404
echo resp.code == Http200  # false
```

---

### `contentType`

```nim
proc contentType*(response: Response | AsyncResponse): string
```

Returns the value of the `Content-Type` response header, or an empty string if it is absent.

```nim
let resp = client.get("https://httpbin.org/json")
echo resp.contentType   # "application/json"
```

---

### `contentLength`

```nim
proc contentLength*(response: Response | AsyncResponse): int
```

Returns the value of the `Content-Length` header as an integer. Returns `-1` if the header is not present. Raises `ValueError` if the header value is not a valid integer.

```nim
let resp = client.get("https://example.com")
echo resp.contentLength   # e.g. 1256, or -1 for chunked responses
```

---

### `lastModified`

```nim
proc lastModified*(response: Response | AsyncResponse): DateTime
```

Parses the `Last-Modified` response header and returns a `DateTime` in UTC. Raises `ValueError` if the header is absent or cannot be parsed as an HTTP date.

```nim
let resp = client.get("https://example.com")
echo resp.lastModified   # e.g. 2024-01-15T10:30:00Z
```

---

### `body` (sync)

```nim
proc body*(response: Response): string
```

Reads and returns the entire response body as a string. The result is **cached**: calling `body` a second time returns the cached string without re-reading the stream.

```nim
let resp = client.get("https://httpbin.org/get")
echo resp.body   # full JSON string
```

---

### `body` (async)

```nim
proc body*(response: AsyncResponse): Future[string] {.async.}
```

Async counterpart: reads and caches the full body, returning a `Future[string]`.

```nim
let resp = await client.get("https://httpbin.org/get")
echo await resp.body
```

---

## Request Procedures

All request procedures come in two flavours:

- `verb(client, url)` → returns a `Response`/`AsyncResponse`. You can inspect status, headers, and body separately.
- `verbContent(client, url)` → returns the body string directly, **and raises `HttpRequestError`** on 4xx/5xx.

### `request`

```nim
proc request*(
  client     : HttpClient | AsyncHttpClient,
  url        : Uri | string,
  httpMethod : HttpMethod | string = HttpGet,
  body       = "",
  headers    : HttpHeaders = nil,
  multipart  : MultipartData = nil
): Future[Response | AsyncResponse] {.multisync.}
```

#### What it does

The universal request procedure — all other verb shortcuts (`get`, `post`, etc.) delegate to this. Sends an HTTP request using any method, with an optional body, per-request headers, and multipart data. Follows redirects automatically up to `client.maxRedirects`.

The `headers` parameter **overrides** `client.headers` for this single request only — they are not persisted.

The URL must not contain newline characters; violation raises `AssertionDefect`.

Passing a `string` for `httpMethod` is deprecated since Nim 1.5 — use the `HttpMethod` enum values (`HttpGet`, `HttpPost`, etc.).

#### Redirect behaviour

| Status code | Method after redirect | Body after redirect |
|------------|----------------------|---------------------|
| 301, 302, 303 | Changed to `GET` (unless already GET/HEAD) | Stripped |
| 307, 308 | Unchanged | Unchanged |

When a redirect crosses to a different hostname, the `Authorization` and `Host` headers are automatically removed to prevent credential leakage.

#### Example

```nim
import std/[httpclient, json]

let client = newHttpClient()
defer: client.close()

# POST with JSON body and per-request Content-Type header
let body = $(%*{"name": "Alice", "score": 42})
let resp = client.request(
  "https://api.example.com/users",
  httpMethod = HttpPost,
  body       = body,
  headers    = newHttpHeaders({"Content-Type": "application/json"})
)
echo resp.status    # "201 Created"
echo resp.body
```

---

### `head`

```nim
proc head*(client: HttpClient | AsyncHttpClient,
           url: Uri | string): Future[Response | AsyncResponse] {.multisync.}
```

Sends an HTTP `HEAD` request. The server returns headers but **no body**. Useful for checking whether a resource exists or obtaining its metadata without downloading it.

```nim
let resp = client.head("https://example.com")
echo resp.code          # Http200
echo resp.contentLength # size in bytes, without downloading
```

---

### `get`

```nim
proc get*(client: HttpClient | AsyncHttpClient,
          url: Uri | string): Future[Response | AsyncResponse] {.multisync.}
```

Sends an HTTP `GET` request and returns the full response object, giving you access to status, headers, and the body stream.

```nim
let resp = client.get("https://httpbin.org/get")
if resp.code == Http200:
  echo resp.body
```

---

### `getContent`

```nim
proc getContent*(client: HttpClient | AsyncHttpClient,
                 url: Uri | string): Future[string] {.multisync.}
```

Sends a `GET` request and returns **only the body** as a string. Raises `HttpRequestError` on any 4xx or 5xx response, making error handling simple.

```nim
try:
  let html = client.getContent("https://example.com")
  echo html
except HttpRequestError as e:
  echo "Server error: ", e.msg
```

---

### `delete`

```nim
proc delete*(client: HttpClient | AsyncHttpClient,
             url: Uri | string): Future[Response | AsyncResponse] {.multisync.}
```

Sends an HTTP `DELETE` request. Returns the full response.

```nim
let resp = client.delete("https://api.example.com/items/42")
echo resp.status   # "204 No Content"
```

---

### `deleteContent`

```nim
proc deleteContent*(client: HttpClient | AsyncHttpClient,
                    url: Uri | string): Future[string] {.multisync.}
```

Sends a `DELETE` request and returns the response body. Raises `HttpRequestError` on 4xx/5xx.

---

### `post`

```nim
proc post*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[Response | AsyncResponse] {.multisync.}
```

Sends an HTTP `POST` request with an optional plain-text body or multipart form data. Returns the full response.

```nim
let resp = client.post("https://httpbin.org/post", body = "hello=world")
echo resp.body
```

---

### `postContent`

```nim
proc postContent*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[string] {.multisync.}
```

Sends a `POST` request and returns the body string. Raises `HttpRequestError` on error.

```nim
let result = client.postContent(
  "https://api.example.com/echo",
  body = "ping"
)
echo result  # "pong"
```

---

### `put`

```nim
proc put*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[Response | AsyncResponse] {.multisync.}
```

Sends an HTTP `PUT` request. Semantically used to **replace** an existing resource in full.

```nim
let resp = client.put(
  "https://api.example.com/items/1",
  body = """{"name":"Updated"}"""
)
echo resp.status
```

---

### `putContent`

```nim
proc putContent*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[string] {.multisync.}
```

Sends a `PUT` request and returns the response body. Raises `HttpRequestError` on error.

---

### `patch`

```nim
proc patch*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[Response | AsyncResponse] {.multisync.}
```

Sends an HTTP `PATCH` request. Used to **partially update** a resource.

```nim
let resp = client.patch(
  "https://api.example.com/items/1",
  body = """{"status":"archived"}"""
)
echo resp.code
```

---

### `patchContent`

```nim
proc patchContent*(
  client    : HttpClient | AsyncHttpClient,
  url       : Uri | string,
  body      = "",
  multipart : MultipartData = nil
): Future[string] {.multisync.}
```

Sends a `PATCH` request and returns the response body. Raises `HttpRequestError` on error.

---

### `downloadFile`

```nim
proc downloadFile*(client: HttpClient, url: Uri | string, filename: string)
proc downloadFile*(client: AsyncHttpClient, url: Uri | string, filename: string): Future[void]
```

#### What it does

Downloads the resource at `url` and saves it directly to `filename` on disk. The body is **streamed** — it is never fully buffered in memory, making this procedure safe for arbitrarily large files.

Raises `HttpRequestError` on 4xx/5xx. Raises `IOError` if the file cannot be opened for writing.

The progress callback (`onProgressChanged`) works normally during a download.

#### Example

```nim
# Synchronous
let client = newHttpClient()
defer: client.close()
client.downloadFile("https://example.com/large.zip", "large.zip")

# Asynchronous
proc dlAsync() {.async.} =
  let client = newAsyncHttpClient()
  defer: client.close()
  await client.downloadFile("https://example.com/large.zip", "large.zip")

waitFor dlAsync()
```

---

## Proxy Support

### `newProxy`

```nim
proc newProxy*(url: Uri): Proxy
proc newProxy*(url: string): Proxy
```

#### What it does

Creates a `Proxy` object from a URL. The URL may include credentials for basic authentication (`http://user:pass@host:port`). Supported schemes:

- `http://` — plain HTTP proxy.
- `https://` — HTTPS proxy (requires `-d:ssl`).
- `socks5h://` — SOCKS5 proxy with **proxy-side** DNS resolution.

The two-argument overloads `newProxy(url, auth)` are **deprecated** — embed credentials in the URL instead.

#### Example

```nim
import std/httpclient

# Plain proxy
let client = newHttpClient(proxy = newProxy("http://proxy.company.com:8080"))

# Proxy with credentials
let client2 = newHttpClient(proxy = newProxy("http://alice:s3cret@proxy.company.com:8080"))

# SOCKS5 proxy (DNS resolved on the proxy side)
let client3 = newHttpClient(proxy = newProxy("socks5h://proxy.company.com:1080"))

# Read proxy from environment
import std/os
let proxyUrl = getEnv("http_proxy", getEnv("https_proxy"))
if proxyUrl.len > 0:
  let client4 = newHttpClient(proxy = newProxy(proxyUrl))
```

---

## Multipart Form Data

Multipart form data is used for HTML form submissions that include file uploads (`enctype="multipart/form-data"`).

### `newMultipartData` (empty)

```nim
proc newMultipartData*(): MultipartData
```

Creates an empty `MultipartData` container. Fields and files are added afterwards using `add`, `[]=`, and `addFiles`.

---

### `newMultipartData` (with entries)

```nim
proc newMultipartData*(xs: MultipartEntries): MultipartData
```

Creates a `MultipartData` pre-filled with simple text fields.

```nim
var data = newMultipartData({"action": "login", "format": "json"})
```

---

### `add` (single field)

```nim
proc add*(p: MultipartData, name, content: string,
          filename    = "",
          contentType = "",
          useStream   = true)
```

#### What it does

Adds a single entry to `p`. When `filename` is non-empty, the entry is treated as a file upload — the `Content-Disposition` header will include `filename=`, and `contentType` sets the part's `Content-Type`. Raises `ValueError` if any of the string arguments contain newline characters (which would corrupt the MIME boundary framing).

When `useStream` is `true` (the default) and the entry is a file, the file is streamed from disk during the request. When `false`, it is read into memory immediately.

```nim
var data = newMultipartData()
data.add("username", "Alice")
data.add("avatar", "/home/alice/photo.png", "photo.png", "image/png")
```

---

### `add` (multiple fields)

```nim
proc add*(p: MultipartData, xs: MultipartEntries): MultipartData {.discardable.}
```

Batch-adds a list of text fields. Returns `p` for chaining.

```nim
data.add({"field1": "value1", "field2": "value2"})
```

---

### `[]=` (text field)

```nim
proc `[]=`*(p: MultipartData, name, content: string)
```

Shorthand for adding a plain text field. Does not set a filename or content type.

```nim
data["username"] = "Alice"
data["language"] = "Nim"
```

---

### `[]=` (file tuple)

```nim
proc `[]=`*(p: MultipartData, name: string,
            file: tuple[name, contentType, content: string])
```

Adds a file entry with manually specified filename, MIME type, and content (as a string). Useful when the file content is already in memory.

```nim
data["document"] = ("report.html", "text/html",
                    "<html><body>Hello</body></html>")
```

---

### `addFiles`

```nim
proc addFiles*(
  p         : MultipartData,
  xs        : openArray[tuple[name, file: string]],
  mimeDb    = newMimetypes(),
  useStream = true
): MultipartData {.discardable.}
```

#### What it does

Reads files from disk and adds them as file parts. The MIME type is **inferred automatically** from the file extension using `mimeDb`. When `useStream` is `true` the files are read in chunks during the request rather than loaded into memory — suitable for large files.

Raises `IOError` if a file cannot be opened. Pass a pre-constructed `mimeDb` to avoid creating a new MIME database on every call.

```nim
import std/[httpclient, mimetypes]

let mimes = newMimetypes()
var data = newMultipartData()
data.addFiles(
  {"photo": "images/avatar.jpg", "resume": "docs/cv.pdf"},
  mimeDb = mimes
)
let resp = client.post("https://upload.example.com/profile", multipart = data)
```

---

### `$` (stringify)

```nim
proc `$`*(data: MultipartData): string
```

Converts `MultipartData` to a human-readable string showing each part's name, filename, content type, and content. Useful for debugging.

```nim
echo $data
# ------ 0 ------
# name="username"
#
# Alice
# ------ 1 ------
# name="avatar"; filename="photo.png"
# Content-Type: image/png
# ...
```

---

## Progress Reporting

Attach a callback to `client.onProgressChanged` to receive download progress updates once per second.

```nim
import std/[asyncdispatch, httpclient]

proc onProgress(total, progress, speed: BiggestInt) {.async.} =
  if total > 0:
    echo progress, " / ", total, " bytes  (", speed div 1024, " KB/s)"
  else:
    echo progress, " bytes  (total unknown)"

proc downloadWithProgress() {.async.} =
  let client = newAsyncHttpClient()
  client.onProgressChanged = onProgress
  defer: client.close()
  await client.downloadFile("https://example.com/large.iso", "large.iso")

waitFor downloadWithProgress()
```

> ⚠️ `total` may be `0` when the server does not send a `Content-Length` header (e.g. chunked transfer encoding).

To remove the callback: `client.onProgressChanged = nil`.

---

## SSL/TLS

HTTPS is activated automatically when the URL scheme is `https://`. You must compile with `-d:ssl`:

```sh
nim c -d:ssl myapp.nim
```

Certificate verification is **on by default** (`CVerifyPeer`). To customise:

```nim
import std/[net, httpclient]

# Verify certificates (default)
let client = newHttpClient(sslContext = newContext(verifyMode = CVerifyPeer))

# Disable verification (not recommended for production)
let clientNoVerify = newHttpClient(sslContext = newContext(verifyMode = CVerifyNone))

# Use environment variables SSL_CERT_FILE / SSL_CERT_DIR
let clientEnv = newHttpClient(sslContext = newContext(verifyMode = CVerifyPeerUseEnvVars))
```

---

## Timeouts

Only `HttpClient` (synchronous) supports timeouts. The timeout is measured in **milliseconds** and applies to individual **socket operations**, not the total request duration. If the server keeps sending data, the timeout will not trigger; it only fires when a single read/write stalls.

```nim
# Raise TimeoutError if any socket operation takes longer than 3 seconds
let client = newHttpClient(timeout = 3_000)
```

---

## Quick Reference

```
Task                                      Procedure
──────────────────────────────────────────────────────────────────────
Fetch a page body (raises on error)       getContent(client, url)
Fetch with full response inspection       get(client, url)
Check resource existence / metadata       head(client, url)
Submit a form / JSON body                 post(client, url, body)
Submit a form / JSON body, get body       postContent(client, url, body)
Replace a resource                        put(client, url, body)
Partial update                            patch(client, url, body)
Delete a resource                         delete(client, url)
Upload files                              post(client, url, multipart=data)
Download a file to disk                   downloadFile(client, url, path)
Custom HTTP method / full control         request(client, url, httpMethod)
```
