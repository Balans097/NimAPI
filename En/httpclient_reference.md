# `std/httpclient` Module Reference (Nim)

> This module implements a simple HTTP client for retrieving web pages and other data.  
> **Warning:** Validate untrusted inputs — URI parsers and getters do not detect malicious URIs.

---

## Table of Contents

1. [Types and Data Structures](#types-and-data-structures)
2. [Proxy](#proxy)
3. [MultipartData](#multipartdata)
4. [Creating Clients](#creating-clients)
5. [Client Management](#client-management)
6. [Response Object](#response-object)
7. [HTTP Request Methods](#http-request-methods)
8. [Constants](#constants)
9. [Exceptions](#exceptions)

---

## Types and Data Structures

### `Response`

Represents a synchronous HTTP server response.

```nim
type Response* = ref object
  version*: string       # HTTP version ("1.0" or "1.1")
  status*: string        # Status string, e.g. "200 OK"
  headers*: HttpHeaders  # Response headers
  bodyStream*: Stream    # Stream for reading the response body
```

### `AsyncResponse`

Represents an asynchronous HTTP server response.

```nim
type AsyncResponse* = ref object
  version*: string
  status*: string
  headers*: HttpHeaders
  bodyStream*: FutureStream[string]
```

### `MultipartData`

An object used to construct `multipart/form-data` request bodies.

### `MultipartEntries`

A type alias for `openArray[tuple[name, content: string]]` — a list of field name / value pairs.

### `Proxy`

Represents an HTTP or SOCKS5 proxy server.

```nim
type Proxy* = ref object
  url*: Uri
```

### `ProgressChangedProc`

The type of the progress callback. Called approximately once per second during a download.

```nim
type ProgressChangedProc*[ReturnType] =
  proc (total, progress, speed: BiggestInt): ReturnType {.closure, gcsafe.}
```

- `total` — expected total size in bytes (may be 0 if the server omits `Content-Length`).
- `progress` — bytes received so far.
- `speed` — bytes received in the last second.

---

## Proxy

### `newProxy(url: Uri): Proxy`
### `newProxy(url: string): Proxy`

Creates a new `Proxy` object. Authentication credentials can be embedded directly in the URL (`http://user:password@host`).

```nim
import std/httpclient

# No authentication
let proxy = newProxy("http://myproxy.example.com:8080")

# With authentication
let authProxy = newProxy("http://user:secret@myproxy.example.com:8080")

# SOCKS5 proxy with proxy-side DNS resolution
let socks5Proxy = newProxy("socks5h://user:secret@myproxy.example.com:1080")

let client = newHttpClient(proxy = proxy)
```

> **Deprecated overloads:** `newProxy(url, auth)` and `proc auth*(p: Proxy)` are deprecated. Embed credentials in the URL instead.

---

## MultipartData

### `newMultipartData(): MultipartData`

Constructs a new, empty `MultipartData` object.

```nim
var data = newMultipartData()
```

---

### `newMultipartData(xs: MultipartEntries): MultipartData`

Creates a `MultipartData` and fills it with the provided entries immediately.

```nim
var data = newMultipartData({"action": "login", "format": "json"})
```

---

### `add(p: MultipartData, name, content: string, filename = "", contentType = "", useStream = true)`

Adds an entry to `MultipartData`.

| Parameter | Description |
|---|---|
| `name` | The form field name |
| `content` | Field value, or a file path if `filename` is provided |
| `filename` | File name (makes this a file field) |
| `contentType` | MIME type of the file |
| `useStream` | `true` — file is streamed from disk; `false` — file is read into memory |

Raises `ValueError` if `name`, `filename`, or `contentType` contain newline characters.

```nim
var data = newMultipartData()
data.add("comment", "Hello!")
data.add("avatar", "/path/to/image.png", filename = "avatar.png",
         contentType = "image/png")
```

---

### `add(p: MultipartData, xs: MultipartEntries): MultipartData`

Adds a list of text fields. Returns `p` for chaining.

```nim
data.add({"username": "alice", "role": "admin"})
```

---

### `addFiles(p: MultipartData, xs: openArray[tuple[name, file: string]], mimeDb = newMimetypes(), useStream = true): MultipartData`

Adds files from the filesystem. MIME types are determined automatically from file extensions.

- `useStream = true` (default): files are streamed from disk during the request — memory-efficient.
- `useStream = false`: files are fully read into memory before sending.

Raises `IOError` if a file cannot be opened or read.

```nim
import std/[httpclient, mimetypes]

let mimes = newMimetypes()
var data = newMultipartData()
data.addFiles({"report": "report.pdf", "logo": "logo.png"}, mimeDb = mimes)

let client = newHttpClient()
defer: client.close()
echo client.postContent("https://example.com/upload", multipart = data)
```

---

### `p[name] = content: string`

Shorthand for adding a plain text field.

```nim
data["username"] = "alice"
```

---

### `p[name] = (filename, contentType, content): tuple`

Shorthand for adding a file field with explicit filename, content type, and content.

```nim
data["page"] = ("index.html", "text/html", "<html>...</html>")
```

---

### `$(data: MultipartData): string`

Converts `MultipartData` to a human-readable string for debugging.

```nim
echo $data
```

---

## Creating Clients

### `newHttpClient(...): HttpClient`

Creates a new synchronous HTTP client instance.

**Signature:**
```nim
proc newHttpClient*(
  userAgent = defUserAgent,
  maxRedirects = 5,
  sslContext = getDefaultSSL(),
  proxy: Proxy = nil,
  timeout = -1,
  headers = newHttpHeaders()
): HttpClient
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `userAgent` | `string` | `"Nim-httpclient/<version>"` | User-Agent header value |
| `maxRedirects` | `int` | `5` | Maximum redirects to follow; `0` to disable |
| `sslContext` | `SslContext` | `CVerifyPeer` | SSL/TLS context |
| `proxy` | `Proxy` | `nil` | HTTP or SOCKS5 proxy |
| `timeout` | `int` | `-1` (no limit) | Timeout in milliseconds |
| `headers` | `HttpHeaders` | empty | Global headers sent with every request |

```nim
import std/httpclient

# Simple client
let client = newHttpClient()
defer: client.close()
echo client.getContent("https://example.com")

# Client with 5-second timeout and custom headers
let client2 = newHttpClient(
  timeout = 5000,
  headers = newHttpHeaders({"Accept": "application/json"})
)
defer: client2.close()
```

---

### `newAsyncHttpClient(...): AsyncHttpClient`

Creates a new asynchronous HTTP client instance. Parameters are identical to `newHttpClient`, except `timeout` is not yet supported.

```nim
import std/[asyncdispatch, httpclient]

proc fetchPage(): Future[string] {.async.} =
  let client = newAsyncHttpClient()
  defer: client.close()
  return await client.getContent("https://example.com")

echo waitFor fetchPage()
```

> **Important:** A single `AsyncHttpClient` instance can only handle one request at a time. To send requests in parallel, create multiple client instances.

---

## Client Management

### `close(client: HttpClient | AsyncHttpClient)`

Closes any active network connections held by the client.

```nim
client.close()
```

Always close the client in a `try/finally` block or using `defer` to avoid resource leaks.

---

### `getSocket(client: HttpClient): Socket`
### `getSocket(client: AsyncHttpClient): AsyncSocket`

Returns the underlying network socket. Useful for inspecting connection details.

```nim
if client.connected:
  echo client.getSocket().getLocalAddr()
  echo client.getSocket().getPeerAddr()
```

---

### Field `onProgressChanged`

A callback invoked approximately once per second to report download progress.

```nim
import std/[asyncdispatch, httpclient]

proc onProgress(total, progress, speed: BiggestInt) {.async.} =
  echo "Downloaded ", progress, " of ", total, " bytes"
  echo "Speed: ", speed div 1024, " KB/s"

proc main() {.async.} =
  let client = newAsyncHttpClient()
  client.onProgressChanged = onProgress
  defer: client.close()
  discard await client.getContent("https://example.com/bigfile.zip")

waitFor main()

# To disable the callback:
# client.onProgressChanged = nil
```

---

### Field `headers`

`HttpHeaders` sent with every request from this client. Can be overridden per-request via the `headers` parameter of `request`.

```nim
client.headers = newHttpHeaders({
  "Authorization": "Bearer mytoken",
  "Accept": "application/json"
})
```

---

### Field `timeout`

Timeout in milliseconds for the synchronous client. `-1` means no timeout. Raises `TimeoutError` if exceeded.

> **Note:** The timeout applies to individual socket calls, not to the entire request. As long as the server is sending data, no exception will be raised.

```nim
let client = newHttpClient(timeout = 3000) # 3 seconds
```

---

## Response Object

### `code(response: Response | AsyncResponse): HttpCode`

Returns the numeric HTTP status code of the response.

Raises `ValueError` if the status string cannot be parsed.

```nim
let resp = client.get("https://example.com")
echo resp.code           # 200
echo resp.code == Http200  # true
```

---

### `contentType(response: Response | AsyncResponse): string`

Returns the value of the `Content-Type` response header.

```nim
echo resp.contentType    # "text/html; charset=utf-8"
```

---

### `contentLength(response: Response | AsyncResponse): int`

Returns the value of the `Content-Length` response header.  
Returns `-1` if the header is absent.

Raises `ValueError` if the header value is not a valid integer.

```nim
echo resp.contentLength  # 42315
```

---

### `lastModified(response: Response | AsyncResponse): DateTime`

Returns the resource's last modification time parsed from the `Last-Modified` header.

Raises `ValueError` if parsing fails.

```nim
echo resp.lastModified   # 2024-01-15T10:30:00+00:00
```

---

### `body(response: Response): string`
### `body(response: AsyncResponse): Future[string]`

Returns the response body as a string. The result is cached — subsequent calls do not re-read from the network.

```nim
# Synchronous
let resp = client.get("https://example.com")
echo resp.body

# Asynchronous
let resp = await client.get("https://example.com")
echo await resp.body
```

---

## HTTP Request Methods

> All methods work with both `HttpClient` (synchronous) and `AsyncHttpClient` (asynchronous).  
> Async versions return a `Future[...]`.

---

### `request(client, url, httpMethod, body, headers, multipart): Response | AsyncResponse`

The universal method for performing any HTTP request.

```nim
proc request*(
  client: HttpClient | AsyncHttpClient,
  url: Uri | string,
  httpMethod: HttpMethod | string = HttpGet,
  body = "",
  headers: HttpHeaders = nil,
  multipart: MultipartData = nil
): Future[Response | AsyncResponse]
```

| Parameter | Description |
|---|---|
| `url` | Request URL; must not contain `\r` or `\n` characters |
| `httpMethod` | Method enum: `HttpGet`, `HttpPost`, `HttpPut`, `HttpDelete`, `HttpPatch`, `HttpHead`, `HttpOptions`, `HttpTrace`, `HttpConnect` |
| `body` | Request body string |
| `headers` | Per-request headers (override `client.headers` for this request only) |
| `multipart` | Form data for multipart uploads |

```nim
import std/[httpclient, json]

let client = newHttpClient()
defer: client.close()

let body = $(%*{"name": "Alice", "age": 30})
let resp = client.request(
  "https://api.example.com/users",
  httpMethod = HttpPost,
  body = body,
  headers = newHttpHeaders({"Content-Type": "application/json"})
)
echo resp.status   # "201 Created"
echo resp.body
```

---

### `head(client, url): Response | AsyncResponse`

Performs a `HEAD` request. No response body is returned.

```nim
let resp = client.head("https://example.com/file.zip")
echo resp.contentLength  # Get file size without downloading
```

---

### `get(client, url): Response | AsyncResponse`

Performs a `GET` request and returns the full response object.

```nim
let resp = client.get("https://httpbin.org/get")
echo resp.code
echo resp.body
```

---

### `getContent(client, url): string | Future[string]`

Performs a `GET` request and returns the response body as a string.  
Raises `HttpRequestError` on 4xx or 5xx status codes.

```nim
let html = client.getContent("https://example.com")
echo html
```

---

### `delete(client, url): Response | AsyncResponse`

Performs a `DELETE` request.

```nim
let resp = client.delete("https://api.example.com/items/42")
echo resp.status
```

---

### `deleteContent(client, url): string | Future[string]`

Performs a `DELETE` request and returns the response body.  
Raises `HttpRequestError` on 4xx/5xx status codes.

```nim
echo client.deleteContent("https://api.example.com/items/42")
```

---

### `post(client, url, body, multipart): Response | AsyncResponse`

Performs a `POST` request.

```nim
# JSON request
client.headers = newHttpHeaders({"Content-Type": "application/json"})
let resp = client.post("https://api.example.com/data", body = """{"key":"value"}""")

# Multipart form upload
var data = newMultipartData()
data["file"] = ("report.csv", "text/csv", "a,b,c\n1,2,3")
let resp2 = client.post("https://example.com/upload", multipart = data)
```

---

### `postContent(client, url, body, multipart): string | Future[string]`

Performs a `POST` request and returns the response body.  
Raises `HttpRequestError` on 4xx/5xx status codes.

```nim
let result = client.postContent(
  "https://api.example.com/echo",
  body = "Hello!"
)
echo result
```

---

### `put(client, url, body, multipart): Response | AsyncResponse`

Performs a `PUT` request.

```nim
let resp = client.put(
  "https://api.example.com/items/1",
  body = """{"name":"Updated"}"""
)
```

---

### `putContent(client, url, body, multipart): string | Future[string]`

Performs a `PUT` request and returns the response body.  
Raises `HttpRequestError` on error status codes.

---

### `patch(client, url, body, multipart): Response | AsyncResponse`

Performs a `PATCH` request.

```nim
let resp = client.patch(
  "https://api.example.com/items/1",
  body = """{"status":"active"}"""
)
```

---

### `patchContent(client, url, body, multipart): string | Future[string]`

Performs a `PATCH` request and returns the response body.  
Raises `HttpRequestError` on error status codes.

---

### `downloadFile(client: HttpClient, url, filename: string)`
### `downloadFile(client: AsyncHttpClient, url, filename): Future[void]`

Downloads the resource at `url` and writes it to `filename` on disk. Data is streamed — the entire body is never loaded into memory at once.

Raises `HttpRequestError` on 4xx/5xx status codes.  
Raises `IOError` if the output file cannot be opened for writing.

```nim
# Synchronous
let client = newHttpClient()
defer: client.close()
client.downloadFile("https://example.com/data.zip", "data.zip")
echo "Download complete!"

# Asynchronous
import std/asyncdispatch

proc downloadAsync() {.async.} =
  let client = newAsyncHttpClient()
  defer: client.close()
  await client.downloadFile("https://example.com/data.zip", "data.zip")

waitFor downloadAsync()
```

---

## Constants

### `defUserAgent*: string`

The default User-Agent string. Format: `"Nim-httpclient/<NimVersion>"`.

```nim
echo defUserAgent  # "Nim-httpclient/2.x.x"
```

---

## Exceptions

| Type | Parent | When raised |
|---|---|---|
| `ProtocolError` | `IOError` | The server violated the HTTP protocol |
| `HttpRequestError` | `IOError` | The server returned a 4xx or 5xx status (in `getContent`, `postContent`, etc.) |

---

## Complete Examples

### Fetch a page (synchronous)

```nim
import std/httpclient

let client = newHttpClient()
try:
  echo client.getContent("https://example.com")
finally:
  client.close()
```

### Send JSON data (asynchronous)

```nim
import std/[asyncdispatch, httpclient, json]

proc sendJson() {.async.} =
  let client = newAsyncHttpClient()
  client.headers = newHttpHeaders({"Content-Type": "application/json"})
  defer: client.close()

  let body = $(%*{"message": "hello"})
  let resp = await client.post("https://httpbin.org/post", body = body)
  echo await resp.body

waitFor sendJson()
```

### Download with progress reporting

```nim
import std/[asyncdispatch, httpclient]

proc onProgress(total, progress, speed: BiggestInt) {.async.} =
  if total > 0:
    echo progress * 100 div total, "% | ", speed div 1024, " KB/s"

proc main() {.async.} =
  let client = newAsyncHttpClient()
  client.onProgressChanged = onProgress
  defer: client.close()
  await client.downloadFile("https://example.com/bigfile.bin", "output.bin")

waitFor main()
```

### Using a proxy with SSL

```nim
import std/[net, httpclient]

let proxy = newProxy("http://proxyuser:proxypass@proxy.example.com:3128")
let ctx = newContext(verifyMode = CVerifyPeer)
let client = newHttpClient(proxy = proxy, sslContext = ctx, timeout = 10_000)
defer: client.close()
echo client.getContent("https://example.com")
```

### Read proxy from environment variables

```nim
import std/[os, httpclient]

var proxyUrl = ""
try:
  if existsEnv("http_proxy"):
    proxyUrl = getEnv("http_proxy")
  elif existsEnv("https_proxy"):
    proxyUrl = getEnv("https_proxy")
except ValueError:
  echo "Failed to read proxy environment variable."

let client = if proxyUrl.len > 0:
  newHttpClient(proxy = newProxy(proxyUrl))
else:
  newHttpClient()
defer: client.close()
```

---

## SSL/TLS Support

SSL is activated automatically when using `https://` URLs.  
You must compile with the `ssl` flag: `nim c -d:ssl yourfile.nim`.

Certificate verification modes:

| Mode | Description |
|---|---|
| `CVerifyNone` | Certificates are not verified |
| `CVerifyPeer` | Certificates are verified (default) |
| `CVerifyPeerUseEnvVars` | Certificates verified + `SSL_CERT_FILE` / `SSL_CERT_DIR` env vars are consulted |

```nim
import std/[net, httpclient]
let client = newHttpClient(sslContext = newContext(verifyMode = CVerifyNone))
```

---

## Redirects

The client automatically follows redirects (301, 302, 303, 307, 308).

- **301 / 302 / 303:** Method is changed to `GET` (unless it was `GET` or `HEAD`); body is discarded.
- **307 / 308:** Method and body are preserved unchanged.
- When redirecting to a different domain, the `Host` and `Authorization` headers are removed.

```nim
let client = newHttpClient(maxRedirects = 0)   # disable redirects
let client2 = newHttpClient(maxRedirects = 10) # increase limit
```
