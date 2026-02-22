# Nim `std/asynchttpserver` Module Reference

> A high-performance asynchronous HTTP/1.1 server built on top of `std/asyncdispatch` and `std/asyncnet`.
>
> ⚠️ **Production warning:** This server is designed for **local development and testing only**. For production deployments, place it behind a reverse proxy such as nginx or Caddy. It lacks TLS, advanced connection management, and production-grade hardening.

---

## Table of Contents

1. [Architecture overview](#architecture-overview)
2. [Types](#types)
   - [Request](#request)
   - [AsyncHttpServer](#asynchttpserver)
3. [Creating the server](#creating-the-server)
   - [newAsyncHttpServer](#newasynchronousserver)
4. [Starting and stopping](#starting-and-stopping)
   - [listen](#listen)
   - [serve](#serve)
   - [close](#close)
5. [The request loop](#the-request-loop)
   - [shouldAcceptRequest](#shouldacceptrequest)
   - [acceptRequest](#acceptrequest)
   - [getPort](#getport)
6. [Responding to requests](#responding-to-requests)
   - [respond](#respond)
   - [sendHeaders](#sendheaders)
7. [Exported constants](#exported-constants)
   - [nimMaxDescriptorsFallback](#nimmaxdescriptorsfallback)
8. [Request lifecycle](#request-lifecycle)
9. [Complete examples](#complete-examples)
   - [Minimal server](#minimal-server)
   - [Custom server loop with error handling](#custom-server-loop-with-error-handling)
   - [Routing by method and path](#routing-by-method-and-path)
10. [Quick Cheat-Sheet](#quick-cheat-sheet)

---

## Architecture overview

The server's concurrency model relies entirely on Nim's async event loop. Each incoming TCP connection is accepted and immediately handed off to `processClient`, which runs as a free-flying async task (via `asyncCheck`). This means hundreds of connections can be in-flight simultaneously in a single thread, each suspended at an `await` while waiting for network I/O.

```
                ┌────────────────────────┐
                │   Your server loop     │
                │  (while true loop)     │
                └──────────┬─────────────┘
                           │ await acceptRequest(cb)
                           ▼
                ┌────────────────────────┐
                │  processClient (async) │  ← one per connection, runs concurrently
                │  ┌──────────────────┐  │
                │  │  processRequest  │  │  ← parses HTTP, reads body
                │  │  → your cb(req)  │  │  ← your handler is called here
                │  └──────────────────┘  │
                │  (keep-alive loop)     │
                └────────────────────────┘
```

The connection is kept alive for HTTP/1.1 by default and closed after the first request for HTTP/1.0, following standard RFC 7230 rules.

---

## Types

### `Request`

```nim
type Request* = object
  client*:    AsyncSocket                           # Raw socket to the connected client
  reqMethod*: HttpMethod                            # GET, POST, PUT, DELETE, …
  headers*:   HttpHeaders                           # All request headers
  protocol*:  tuple[orig: string, major, minor: int] # e.g. ("HTTP/1.1", 1, 1)
  url*:       Uri                                   # Parsed request URL
  hostname*:  string                                # IP address of the connecting client
  body*:      string                                # Request body (fully buffered)
```

**What it is:** A value object passed to your request handler callback. It contains everything you need to understand and respond to an HTTP request. All fields are public and readable.

Key notes:
- `body` is fully read and buffered before your callback is invoked. The server enforces `maxBody` bytes as a hard limit, responding with `413` if exceeded.
- `client` gives you direct socket access for advanced use cases (WebSocket upgrades, streaming responses). In normal usage you should use `respond` instead.
- `protocol.major` and `protocol.minor` let you distinguish HTTP/1.0 from HTTP/1.1 clients.
- `url` is a parsed `Uri` — you can access `url.path`, `url.query`, `url.hostname` etc. directly.
- `hostname` is the **client's IP address** as a string (not a DNS name).

```nim
proc handler(req: Request) {.async.} =
  echo req.reqMethod   # GET, POST, etc.
  echo req.url.path    # "/api/users"
  echo req.url.query   # "page=2&limit=10"
  echo req.hostname    # "192.168.1.42"
  echo req.body        # request body (for POST/PUT)

  for key, val in req.headers:
    echo key, ": ", val
```

---

### `AsyncHttpServer`

```nim
type AsyncHttpServer* = ref object
  # socket:   AsyncSocket  (internal)
  # reuseAddr: bool        (internal)
  # reusePort: bool        (internal)
  # maxBody:  int          (internal)
  # maxFDs:   int          (internal)
```

**What it is:** A reference-type object representing the running HTTP server. You create it with `newAsyncHttpServer`, bind it to a port with `listen`, and then pump requests with `acceptRequest` or `serve`. All fields are internal — configure the server at construction time via `newAsyncHttpServer`'s parameters.

---

## Creating the server

### `newAsyncHttpServer`

```nim
proc newAsyncHttpServer*(reuseAddr = true,
                         reusePort = false,
                         maxBody   = 8_388_608): AsyncHttpServer
```

**What it does:** Creates a new server instance and configures its socket options and body limit. Does **not** bind a port or start listening — call `listen` next.

**Parameters:**

- `reuseAddr` — when `true` (the default), sets `SO_REUSEADDR` on the socket. This allows the server to restart quickly without waiting for the OS to release the previous socket's TIME_WAIT state. Almost always desirable.
- `reusePort` — when `true`, sets `SO_REUSEPORT`, allowing multiple server processes to bind to the same port (useful for multi-process load balancing). Disabled by default. Not available on NuttX.
- `maxBody` — the maximum number of bytes the server will read for a request body. Defaults to 8 MiB (8,388,608 bytes). Requests with a `Content-Length` exceeding this limit receive a `413 Request Entity Too Large` response automatically.

```nim
import std/asynchttpserver

# Default: 8 MiB body limit, reuseAddr on
var server = newAsyncHttpServer()

# API server with a tighter body limit (1 MiB)
var apiServer = newAsyncHttpServer(maxBody = 1_048_576)

# Multi-process setup (socket shared between workers)
var workerServer = newAsyncHttpServer(reusePort = true)
```

---

## Starting and stopping

### `listen`

```nim
proc listen*(server: AsyncHttpServer;
             port:    Port;
             address  = "";
             domain   = AF_INET)
```

**What it does:** Binds the server's socket to the given `port` and `address`, then calls `listen()` on it — making the OS start queuing incoming connections. This procedure is **synchronous** (no `await` needed). After `listen` returns, the server is ready to accept connections; call `acceptRequest` or `serve` to begin processing them.

**Parameters:**

- `port` — the TCP port to bind. Use `Port(0)` to let the OS choose a free port automatically (useful in tests); retrieve the assigned port with `getPort`.
- `address` — the local IP address to bind to. Empty string (default) binds to all interfaces (`0.0.0.0` for IPv4). Use `"127.0.0.1"` to accept local connections only.
- `domain` — the socket address family. `AF_INET` (default) for IPv4, `AF_INET6` for IPv6.

```nim
import std/asynchttpserver
from std/nativesockets import AF_INET6

var server = newAsyncHttpServer()

# Bind to all interfaces on port 8080
server.listen(Port(8080))

# Bind to localhost only
server.listen(Port(8080), address = "127.0.0.1")

# IPv6
server.listen(Port(8080), domain = AF_INET6)

# OS-chosen port (good for tests)
server.listen(Port(0))
echo "Listening on port ", server.getPort.uint16
```

---

### `serve`

```nim
proc serve*(server:   AsyncHttpServer,
            port:     Port,
            callback: proc(request: Request): Future[void] {.closure, gcsafe.},
            address   = "";
            assumedDescriptorsPerRequest = -1;
            domain    = AF_INET) {.async.}
```

**What it does:** A convenience all-in-one coroutine that calls `listen` and then loops forever, calling `acceptRequest` for each incoming connection. Intended for quick prototypes where you don't need a custom loop.

**Important:** The module's own documentation recommends preferring `acceptRequest` with your own loop for production-quality code, because `serve` gives you no opportunity to handle errors from `acceptRequest` or log connection events.

`assumedDescriptorsPerRequest` controls the FD-limit guard:
- `-1` (default): **disabled** — the guard is not checked, connections are always accepted.
- `≥ 0`: the number of file descriptors assumed to be consumed per request. The server refuses to accept a new connection if `activeDescriptors() + assumedDescriptorsPerRequest ≥ maxFDs`, calling `poll()` to yield instead. Use `5` (the value used in examples) as a starting point.

```nim
import std/[asynchttpserver, asyncdispatch]

proc handler(req: Request) {.async.} =
  await req.respond(Http200, "Hello!")

var server = newAsyncHttpServer()
waitFor server.serve(Port(8080), handler)
```

---

### `close`

```nim
proc close*(server: AsyncHttpServer)
```

**What it does:** Closes the server's listening socket. This is a **synchronous** call. After `close`, no new connections will be accepted. Connections that are already in progress (being handled by `processClient`) are **not** interrupted — they run to completion.

**When to use it:** For graceful shutdown in response to a signal, a test teardown, or any other situation where you want to stop accepting new clients.

```nim
import std/[asynchttpserver, asyncdispatch, os]

var server = newAsyncHttpServer()
server.listen(Port(8080))

# Shut down cleanly when the user presses Ctrl-C
setControlCHook(proc() {.noconv.} =
  server.close()
  quit(0)
)
```

---

## The request loop

The recommended pattern for running the server is to write an explicit `while true` loop that calls `shouldAcceptRequest` before each `acceptRequest`. This gives you control over error handling, logging, and backpressure.

### `shouldAcceptRequest`

```nim
proc shouldAcceptRequest*(server: AsyncHttpServer;
                          assumedDescriptorsPerRequest = 5): bool {.inline.}
```

**What it does:** Returns `true` if it is safe to accept another connection right now. Internally it checks whether `activeDescriptors() + assumedDescriptorsPerRequest < server.maxFDs`. When the process is running low on file descriptors, accepting a new connection could cause the OS to refuse to open additional sockets or files — this guard prevents that.

- `assumedDescriptorsPerRequest` — how many file descriptors a single request is expected to consume (socket + potential log file, temp files, etc.). The default of `5` is conservative and works for most applications.
- Returns `true` unconditionally when `assumedDescriptorsPerRequest < 0`.

**When to use it:** Always call this before `acceptRequest` in your own loop. When it returns `false`, wait briefly (e.g. `await sleepAsync(500)`) to let open connections close before trying again.

```nim
while true:
  if server.shouldAcceptRequest():
    await server.acceptRequest(handler)
  else:
    await sleepAsync(500)  # back off, FDs are scarce
```

---

### `acceptRequest`

```nim
proc acceptRequest*(server:   AsyncHttpServer,
                    callback: proc(request: Request): Future[void] {.closure, gcsafe.})
                   {.async.}
```

**What it does:** Accepts a single incoming TCP connection and immediately launches `processClient` as a background async task (via `asyncCheck`). `processClient` handles HTTP parsing, reads the body, calls your `callback`, and manages keep-alive. `acceptRequest` itself returns as soon as the connection is handed off — it does **not** wait for the request to be handled.

**The callback signature:** Your handler must be:
```nim
proc myHandler(req: Request): Future[void] {.async.}
```
It may not raise exceptions directly — wrap any errors in a try/except and call `req.respond` with an appropriate error code.

**When to use it:** In your own `while true` server loop, paired with `shouldAcceptRequest`.

```nim
import std/[asynchttpserver, asyncdispatch]

proc handler(req: Request) {.async.} =
  try:
    await req.respond(Http200, "OK")
  except Exception as e:
    echo "Handler error: ", e.msg

proc main() {.async.} =
  var server = newAsyncHttpServer()
  server.listen(Port(8080))
  echo "Listening on port 8080"

  while true:
    if server.shouldAcceptRequest():
      await server.acceptRequest(handler)
    else:
      await sleepAsync(500)

waitFor main()
```

---

### `getPort`

```nim
proc getPort*(self: AsyncHttpServer): Port
```

**What it does:** Returns the port that the server is currently bound to. This is especially useful when you bound to `Port(0)` and need to discover what port the OS actually assigned.

Must be called **after** `listen` — before that, the socket has not been bound yet.

```nim
import std/asynchttpserver

var server = newAsyncHttpServer()
server.listen(Port(0))

let port = server.getPort
echo "Server is on port ", port.uint16   # e.g. "Server is on port 54321"
```

---

## Responding to requests

### `respond`

```nim
proc respond*(req:     Request,
              code:    HttpCode,
              content: string,
              headers: HttpHeaders = nil): Future[void]
```

**What it does:** Sends a complete HTTP response to the client. It assembles the status line, your custom headers, an automatic `Content-Length` header (if you did not provide one), and the body, then sends everything in a single write. Returns a `Future[void]` that completes when the data has been handed to the OS.

**Important:** `respond` does **not** close the client socket. The server manages connection lifetime (keep-alive vs. close) automatically based on the HTTP version and `Connection` headers. Never call `req.client.close()` from your callback unless you are doing a WebSocket upgrade.

**Parameters:**

- `code` — an `HttpCode` from `std/httpcore`. Common values: `Http200`, `Http201`, `Http400`, `Http404`, `Http500`. You can also use numeric literals: `HttpCode(418)`.
- `content` — the response body as a string. May be empty for responses like `204 No Content`.
- `headers` — optional `HttpHeaders` with response headers. If `nil`, only `Content-Length` is sent. If you supply headers without `Content-Length`, the server adds it automatically.

```nim
proc handler(req: Request) {.async.} =
  case req.url.path
  of "/":
    await req.respond(Http200, "Welcome!")

  of "/json":
    let headers = newHttpHeaders([("Content-Type", "application/json")])
    await req.respond(Http200, """{"ok": true}""", headers)

  of "/created":
    let headers = newHttpHeaders([
      ("Content-Type", "text/plain"),
      ("Location", "/resource/42")
    ])
    await req.respond(Http201, "Created", headers)

  else:
    await req.respond(Http404, "Not Found")
```

---

### `sendHeaders`

```nim
proc sendHeaders*(req: Request, headers: HttpHeaders): Future[void]
```

**What it does:** Sends **only** the given `HttpHeaders` to the client, without a status line or body. This is a low-level building block for scenarios where you need fine-grained control over what is sent — for example, when streaming a response in chunks, or when implementing a protocol upgrade (WebSocket handshake).

**When to use it:** Rarely — most handlers should use `respond`. Use `sendHeaders` when you want to start a response manually and then write the body directly to `req.client`.

```nim
proc streamingHandler(req: Request) {.async.} =
  # Send status line manually
  await req.client.send("HTTP/1.1 200 OK\r\n")

  # Send headers
  let h = newHttpHeaders([
    ("Content-Type", "text/event-stream"),
    ("Cache-Control", "no-cache"),
    ("Transfer-Encoding", "chunked"),
  ])
  await req.sendHeaders(h)
  await req.client.send("\r\n")  # blank line ends headers

  # Stream data directly
  for i in 0..9:
    await req.client.send("data: event " & $i & "\n\n")
    await sleepAsync(500)
```

---

## Exported constants

### `nimMaxDescriptorsFallback`

```nim
const nimMaxDescriptorsFallback* {.intdefine.} = 16_000
```

**What it is:** The fallback value used for the maximum number of open file descriptors when the system's `maxDescriptors()` call is unavailable. Defaults to 16,000.

You can override this at compile time:
```sh
nim c -d:nimMaxDescriptorsFallback=8000 myserver.nim
```

This constant feeds into `shouldAcceptRequest`'s guard. If your OS limit is lower than this fallback (e.g., the default `ulimit -n` of 1024 on many Linux systems), you may want to lower this value or increase `ulimit -n` in production.

---

## Request lifecycle

Understanding the full lifecycle helps you write correct handlers:

```
Client connects
    │
    ▼
listen() queues the connection
    │
    ▼
acceptRequest() accepts TCP socket
    │
    ▼
processClient() loop starts
    │
    ▼
processRequest() runs:
  1. Read request line (GET /path HTTP/1.1)   → 413 if > 8KB
  2. Read headers one by one                  → 413 if > 8KB/header
                                              → 400 if too many headers
  3. Handle Expect: 100-continue (POST only)
  4. Read body:
       Content-Length present → recv(N) bytes  → 413 if N > maxBody
       Transfer-Encoding: chunked              → reassemble chunks
       POST with neither                       → 411 Length Required
  5. Call your callback(request)
  6. Check Connection: close/upgrade
  7. HTTP/1.1 → keep connection alive
     HTTP/1.0 → close connection
    │
    ▼
Loop back or close socket
```

**What you must do in your callback:**
- Always call `await req.respond(...)` (or write to `req.client` directly). If you return without responding, the client will see a partial or empty response.
- Handle all exceptions inside the callback. An unhandled exception in your callback will be silently swallowed by `asyncCheck` — use `try/except` to log and respond with a 500.

---

## Complete examples

### Minimal server

```nim
import std/[asynchttpserver, asyncdispatch]

proc handler(req: Request) {.async.} =
  let headers = newHttpHeaders([("Content-Type", "text/plain; charset=utf-8")])
  await req.respond(Http200, "Hello, World!", headers)

proc main() {.async.} =
  var server = newAsyncHttpServer()
  server.listen(Port(8080))
  echo "Listening on http://localhost:8080"

  while true:
    if server.shouldAcceptRequest():
      await server.acceptRequest(handler)
    else:
      await sleepAsync(500)

waitFor main()
```

---

### Custom server loop with error handling

```nim
import std/[asynchttpserver, asyncdispatch, logging]

addHandler(newConsoleLogger())

proc handler(req: Request) {.async.} =
  try:
    info req.reqMethod, " ", req.url.path, " from ", req.hostname
    await req.respond(Http200, "OK")
  except Exception as e:
    error "Handler exception: ", e.msg
    try:
      await req.respond(Http500, "Internal Server Error")
    except: discard

proc main() {.async.} =
  var server = newAsyncHttpServer()
  server.listen(Port(8080))
  info "Server started on port 8080"

  while true:
    if server.shouldAcceptRequest():
      try:
        await server.acceptRequest(handler)
      except Exception as e:
        error "acceptRequest failed: ", e.msg
        await sleepAsync(100)  # brief pause before retrying
    else:
      warn "FD limit approaching, pausing..."
      await sleepAsync(500)

waitFor main()
```

---

### Routing by method and path

```nim
import std/[asynchttpserver, asyncdispatch, json, strutils]

type User = object
  id:   int
  name: string

var users = @[User(id: 1, name: "Alice"), User(id: 2, name: "Bob")]

proc handler(req: Request) {.async.} =
  let path = req.url.path

  if path == "/users" and req.reqMethod == HttpGet:
    let body = $ %* users.mapIt(%*{"id": it.id, "name": it.name})
    let headers = newHttpHeaders([("Content-Type", "application/json")])
    await req.respond(Http200, body, headers)

  elif path.startsWith("/users/") and req.reqMethod == HttpGet:
    try:
      let id = path[7..^1].parseInt
      for u in users:
        if u.id == id:
          let headers = newHttpHeaders([("Content-Type", "application/json")])
          await req.respond(Http200, $ %*{"id": u.id, "name": u.name}, headers)
          return
      await req.respond(Http404, "User not found")
    except ValueError:
      await req.respond(Http400, "Invalid user ID")

  elif path == "/users" and req.reqMethod == HttpPost:
    try:
      let data = parseJson(req.body)
      let newUser = User(id: users.len + 1, name: data["name"].getStr)
      users.add(newUser)
      let headers = newHttpHeaders([
        ("Content-Type", "application/json"),
        ("Location", "/users/" & $newUser.id),
      ])
      await req.respond(Http201, $ %*{"id": newUser.id}, headers)
    except JsonParsingError:
      await req.respond(Http400, "Invalid JSON")

  else:
    await req.respond(Http404, "Not Found")

proc main() {.async.} =
  var server = newAsyncHttpServer()
  server.listen(Port(8080))
  while true:
    if server.shouldAcceptRequest():
      await server.acceptRequest(handler)
    else:
      await sleepAsync(500)

waitFor main()
```

---

## Quick Cheat-Sheet

| Task | API |
|---|---|
| Create server | `newAsyncHttpServer(reuseAddr, reusePort, maxBody)` |
| Bind to port | `server.listen(Port(8080))` |
| OS-chosen port | `server.listen(Port(0))` |
| Find bound port | `server.getPort.uint16` |
| Check FD headroom | `server.shouldAcceptRequest()` |
| Accept one connection | `await server.acceptRequest(handler)` |
| All-in-one loop | `await server.serve(Port(8080), handler)` |
| Stop accepting | `server.close()` |
| Respond with body | `await req.respond(Http200, "body", headers)` |
| Respond without body | `await req.respond(Http204, "")` |
| Send headers only | `await req.sendHeaders(headers)` |
| Read request path | `req.url.path` |
| Read query string | `req.url.query` |
| Read request body | `req.body` |
| Read HTTP method | `req.reqMethod` (HttpGet, HttpPost, …) |
| Read client IP | `req.hostname` |
| Read a header | `req.headers["Content-Type"]` |
| Read header safely | `req.headers.getOrDefault("X-Key", "")` |
| Check HTTP version | `req.protocol.major`, `req.protocol.minor` |
| Adjust body size limit | `newAsyncHttpServer(maxBody = 1_048_576)` |
| Override max FD fallback | compile with `-d:nimMaxDescriptorsFallback=N` |
