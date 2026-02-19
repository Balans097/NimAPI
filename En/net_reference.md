# `std/net` Module Reference (Nim)

> A high-level cross-platform socket interface.
> Designed primarily for **blocking** sockets.
> For non-blocking asynchronous sockets, use `asyncnet` + `asyncdispatch`.
>
> SSL/TLS: compile with `-d:ssl` to enable SSL support.

---

## Table of Contents

1. [Constants](#constants)
2. [Types](#types)
   - [Socket, SocketImpl](#socket-socketimpl)
   - [IpAddress, IpAddressFamily](#ipaddress-ipaddressfamily)
   - [SOBool — Socket Options](#sobool--socket-options)
   - [SocketFlag](#socketflag)
   - [SSL-specific Types](#ssl-specific-types)
3. [Creating a Socket](#creating-a-socket)
4. [Working with IP Addresses](#working-with-ip-addresses)
5. [Special-Purpose Addresses](#special-purpose-addresses)
6. [Connecting (Client)](#connecting-client)
7. [Server: bind, listen, accept](#server-bind-listen-accept)
8. [Unix Sockets (POSIX)](#unix-sockets-posix)
9. [Sending Data](#sending-data)
10. [Receiving Data](#receiving-data)
11. [Closing a Socket](#closing-a-socket)
12. [Socket Options](#socket-options)
13. [Buffering and State](#buffering-and-state)
14. [Error Handling](#error-handling)
15. [SSL/TLS API](#ssltls-api)
16. [SockAddr Utilities](#sockaddr-utilities)
17. [Quick Recipes](#quick-recipes)

---

## Constants

```nim
BufferSize*: int = 4000
  ## Size of the internal buffer of a buffered socket.

MaxLineLength* = 1_000_000
  ## Maximum line length for readLine/recvLine.
```

---

## Types

### Socket, SocketImpl

```nim
type
  SocketImpl* = object
    fd: SocketHandle
    isBuffered: bool
    buffer: array[0..BufferSize, char]
    currPos: int
    bufLen: int
    # (SSL fields only with -d:ssl)
    lastError: OSErrorCode
    domain: Domain
    sockType: SockType
    protocol: Protocol

  Socket* = ref SocketImpl
```

`Socket` is the main type — a reference to `SocketImpl`. All operations are performed on this type.

---

### IpAddress, IpAddressFamily

```nim
type
  IpAddressFamily* {.pure.} = enum
    IPv6  ## IPv6 address
    IPv4  ## IPv4 address

  IpAddress* = object
    case family*: IpAddressFamily
    of IpAddressFamily.IPv6:
      address_v6*: array[0..15, uint8]
    of IpAddressFamily.IPv4:
      address_v4*: array[0..3, uint8]
```

Stores an IPv4 or IPv6 address as bytes. The `family` field determines which data field is active.

---

### SOBool — Socket Options

```nim
type SOBool* = enum
  OptAcceptConn   ## Socket is accepting connections
  OptBroadcast    ## Allow broadcast packets
  OptDebug        ## Enable debugging
  OptDontRoute    ## Don't use routing
  OptKeepAlive    ## Send keepalive packets
  OptOOBInline    ## Receive OOB data in the normal data stream
  OptReuseAddr    ## Allow address reuse
  OptReusePort    ## Allow multiple sockets on the same port
  OptNoDelay      ## Disable Nagle's algorithm
```

---

### SocketFlag

```nim
type SocketFlag* {.pure.} = enum
  Peek         ## Read without consuming from the buffer (MSG_PEEK)
  SafeDisconn  ## Do not raise an exception on disconnection errors
```

`SafeDisconn` allows graceful handling when the client closes the connection during `accept` or `send`.

---

### ReadLineResult

```nim
type ReadLineResult* = enum
  ReadFullLine      ## A complete line was read
  ReadPartialLine   ## Only part of a line was read
  ReadDisconnected  ## Connection was disconnected
  ReadNone          ## No data available
```

---

### SSL-specific Types

Only available when compiled with `-d:ssl`.

```nim
Certificate* = string  ## DER-encoded certificate

SslError*           # SSL exception (CatchableError)
TimeoutError*       # Timeout exception (CatchableError)

SslCVerifyMode* = enum
  CVerifyNone               ## Certificates are not verified
  CVerifyPeer               ## Certificates are verified
  CVerifyPeerUseEnvVars     ## Verified + SSL_CERT_FILE / SSL_CERT_DIR env vars used

SslProtVersion* = enum
  protSSLv2, protSSLv3, protTLSv1, protSSLv23

SslHandshakeType* = enum
  handshakeAsClient
  handshakeAsServer

SslAcceptResult* = enum
  AcceptNoClient = 0
  AcceptNoHandshake
  AcceptSuccess

SslContext* = ref object  ## SSL context
  context*: SslCtx

SslClientGetPskFunc* = proc(hint: string): tuple[identity: string, psk: string]
SslServerGetPskFunc* = proc(identity: string): string
```

---

## Creating a Socket

### `newSocket` (from SocketHandle)
```nim
proc newSocket*(fd: SocketHandle, domain: Domain = AF_INET,
    sockType: SockType = SOCK_STREAM,
    protocol: Protocol = IPPROTO_TCP, buffered = true): owned(Socket)
```
Creates a `Socket` object from an existing file descriptor. Used when interfacing with native APIs.

---

### `newSocket` (create new)
```nim
proc newSocket*(domain: Domain = AF_INET, sockType: SockType = SOCK_STREAM,
                protocol: Protocol = IPPROTO_TCP, buffered = true,
                inheritable = defined(nimInheritHandles)): owned(Socket)

proc newSocket*(domain, sockType, protocol: cint, buffered = true,
                inheritable = defined(nimInheritHandles)): owned(Socket)
```
Creates a new socket. Parameters:
- `domain` — address family (`AF_INET`, `AF_INET6`, `AF_UNIX`, etc.)
- `sockType` — type (`SOCK_STREAM` for TCP, `SOCK_DGRAM` for UDP)
- `protocol` — protocol (`IPPROTO_TCP`, `IPPROTO_UDP`, `IPPROTO_NONE`)
- `buffered` — use internal read buffer (recommended: `true`)
- `inheritable` — whether child processes inherit the file descriptor

Raises `OSError` on failure.

```nim
# Default TCP socket
let tcpSock = newSocket()

# UDP socket
let udpSock = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)

# IPv6 TCP
let sock6 = newSocket(AF_INET6, SOCK_STREAM, IPPROTO_TCP)
```

---

### `dial`
```nim
proc dial*(address: string, port: Port,
           protocol = IPPROTO_TCP, buffered = true): owned(Socket)
```
Establishes a connection to `address:port` and returns a socket ready to use. Unlike `newSocket` + `connect`, `dial` automatically iterates through all resolved IP addresses and works transparently with both IPv4 and IPv6. Raises `OSError` or `IOError` on failure.

```nim
let sock = dial("nim-lang.org", Port(80))
sock.send("GET / HTTP/1.0\r\nHost: nim-lang.org\r\n\r\n")
echo sock.recvLine()
sock.close()
```

---

## Working with IP Addresses

### `parseIpAddress`
```nim
proc parseIpAddress*(addressStr: string): IpAddress
```
Parses a string into an `IpAddress`. Supports IPv4 (strict RFC 6943 — no leading zeros) and IPv6. Raises `ValueError` on invalid input.

```nim
let ip4 = parseIpAddress("192.168.1.1")
let ip6 = parseIpAddress("::1")
let ip6full = parseIpAddress("2001:db8::1")
```

---

### `isIpAddress`
```nim
proc isIpAddress*(addressStr: string): bool
```
Returns `true` if the string is a valid IPv4 or IPv6 address. Never raises exceptions.

```nim
doAssert isIpAddress("192.168.1.1") == true
doAssert isIpAddress("::1") == true
doAssert isIpAddress("not-an-ip") == false
```

---

### `$` (IpAddress → string)
```nim
proc `$`*(address: IpAddress): string
```
Converts an `IpAddress` to its string representation. IPv6 addresses are formatted per RFC (with `::` compression of the largest zero run).

```nim
let ip = parseIpAddress("192.168.1.1")
echo $ip  # "192.168.1.1"

let ip6 = parseIpAddress("::1")
echo $ip6  # "::1"
```

---

### `==` (IpAddress)
```nim
proc `==`*(lhs, rhs: IpAddress): bool
```
Compares two `IpAddress` values. Addresses of different families are always unequal.

```nim
doAssert parseIpAddress("127.0.0.1") == parseIpAddress("127.0.0.1")
doAssert parseIpAddress("::1") != parseIpAddress("127.0.0.1")
```

---

### `toSockAddr` / `fromSockAddr`
```nim
proc toSockAddr*(address: IpAddress, port: Port, sa: var Sockaddr_storage, sl: var SockLen)
proc fromSockAddr*(sa: ..., sl: SockLen, address: var IpAddress, port: var Port)
```
Converts between `IpAddress`/`Port` and the low-level `Sockaddr_storage` structures used in system calls. Raises `ValueError` / `ObjectConversionDefect` on unknown address family.

---

### `getPrimaryIPAddr`
```nim
proc getPrimaryIPAddr*(dest = parseIpAddress("8.8.8.8")): IpAddress
```
Returns the local IP address used to reach external networks (typically the address assigned to eth0 or wlan0). No traffic is actually sent. Supports both IPv4 and IPv6. Raises `OSError` if external networking is unavailable.

```nim
echo getPrimaryIPAddr()  # e.g. "192.168.1.2"
```

---

## Special-Purpose Addresses

```nim
proc IPv4_any*(): IpAddress        ## 0.0.0.0 — all interfaces
proc IPv4_loopback*(): IpAddress   ## 127.0.0.1
proc IPv4_broadcast*(): IpAddress  ## 255.255.255.255

proc IPv6_any*(): IpAddress        ## :: — all interfaces
proc IPv6_loopback*(): IpAddress   ## ::1
```

```nim
# Listen on all interfaces
socket.bindAddr(Port(8080), $IPv4_any())

# Connect to local server
socket.connect($IPv4_loopback(), Port(3000))
```

---

## Connecting (Client)

### `connect` (without timeout)
```nim
proc connect*(socket: Socket, address: string, port = Port(0))
```
Connects the socket to `address:port`. `address` can be an IP address or hostname. Iterates through all resolved IPs if needed. Raises `OSError` on failure. For SSL sockets, performs the handshake automatically.

```nim
let socket = newSocket()
socket.connect("google.com", Port(80))
```

---

### `connect` (with timeout)
```nim
proc connect*(socket: Socket, address: string, port = Port(0), timeout: int)
```
Same as above but with a `timeout` in **milliseconds**. Raises `TimeoutError` if the connection is not established in time. Temporarily sets the socket to non-blocking mode internally.

```nim
let socket = newSocket()
socket.connect("example.com", Port(80), timeout = 5000)  # 5 seconds
```

---

## Server: bind, listen, accept

### `bindAddr`
```nim
proc bindAddr*(socket: Socket, port = Port(0), address = "")
```
Binds the socket to `address:port`. If `address` is `""`, binds to `ADDR_ANY` (all interfaces: `0.0.0.0` for IPv4, `::` for IPv6). `Port(0)` means the OS picks a free port automatically.

```nim
let server = newSocket()
server.bindAddr(Port(8080))
```

---

### `listen`
```nim
proc listen*(socket: Socket, backlog = SOMAXCONN)
```
Marks the socket as passively accepting connections. `backlog` is the maximum length of the pending connection queue. Raises `OSError` on failure.

```nim
server.listen()
```

---

### `acceptAddr`
```nim
proc acceptAddr*(server: Socket, client: var owned(Socket), address: var string,
                 flags = {SocketFlag.SafeDisconn},
                 inheritable = defined(nimInheritHandles))
```
Blocks until an incoming connection arrives. Sets `client` to the client socket and `address` to the client's address string. The client socket inherits the server's buffering settings.

The `SafeDisconn` flag (default) causes `accept` to retry automatically if the connecting client disconnects during the accept. For SSL servers, performs the SSL handshake automatically.

```nim
var client: Socket
var address = ""
server.acceptAddr(client, address)
echo "Client connected from: ", address
```

---

### `accept`
```nim
proc accept*(server: Socket, client: var owned(Socket),
             flags = {SocketFlag.SafeDisconn},
             inheritable = defined(nimInheritHandles))
```
Same as `acceptAddr` but does not return the client's address.

```nim
var client: Socket
server.accept(client)
client.send("Welcome!\n")
```

---

### TCP Server Example

```nim
let server = newSocket()
server.setSockOpt(OptReuseAddr, true)
server.bindAddr(Port(8080))
server.listen()

while true:
  var client: Socket
  var address = ""
  server.acceptAddr(client, address)
  echo "Client from: ", address
  client.send("HTTP/1.0 200 OK\r\n\r\nHello!\r\n")
  client.close()
```

---

## Unix Sockets (POSIX)

Available only on POSIX systems (Linux, macOS, BSD).

### `connectUnix`
```nim
proc connectUnix*(socket: Socket, path: string)
```
Connects to a Unix domain socket at `path`.

### `bindUnix`
```nim
proc bindUnix*(socket: Socket, path: string)
```
Binds a socket to the Unix domain socket path `path`.

```nim
let server = newSocket(AF_UNIX, SOCK_STREAM, IPPROTO_NONE)
server.bindUnix("/tmp/myapp.sock")
server.listen()

let client = newSocket(AF_UNIX, SOCK_STREAM, IPPROTO_NONE)
client.connectUnix("/tmp/myapp.sock")
```

---

## Sending Data

### `send` (low-level pointer)
```nim
proc send*(socket: Socket, data: pointer, size: int): int
```
Low-level send into a raw memory buffer. Returns the number of bytes sent. Does not guarantee all data is sent.

---

### `send` (string)
```nim
proc send*(socket: Socket, data: string,
           flags = {SocketFlag.SafeDisconn}, maxRetries = 100)
```
High-level string send. Handles interrupts and partial writes, retrying up to `maxRetries` times. Raises `OSError` on failure.

```nim
socket.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
```

---

### `&=` (send operator)
```nim
template `&=`*(socket: Socket; data: typed)
```
An alias for `send`. Convenient for chained sends.

```nim
socket &= "Hello, "
socket &= "World!\n"
```

---

### `trySend`
```nim
proc trySend*(socket: Socket, data: string): bool
```
A safe alternative to `send`. Returns `false` on failure instead of raising an exception.

```nim
if not socket.trySend("data"):
  echo "Failed to send"
```

---

### `sendTo` (UDP)
```nim
# Low-level: by address string + raw pointer
proc sendTo*(socket: Socket, address: string, port: Port, data: pointer,
             size: int, af: Domain = AF_INET, flags = 0'i32)

# High-level: by address string
proc sendTo*(socket: Socket, address: string, port: Port, data: string)

# By IpAddress (returns number of bytes sent)
proc sendTo*(socket: Socket, address: IpAddress, port: Port,
             data: string, flags = 0'i32): int {.discardable.}
```
Sends data to the specified address without a prior connection. Intended for connectionless (UDP) sockets. Raises an assertion error if used on a TCP socket.

```nim
let udpSock = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
udpSock.sendTo("192.168.0.1", Port(27960), "status\n")

let ip = parseIpAddress("192.168.0.1")
doAssert udpSock.sendTo(ip, Port(27960), "status\c\l") == 8
```

---

## Receiving Data

### `recv` (low-level pointer)
```nim
proc recv*(socket: Socket, data: pointer, size: int): int
proc recv*(socket: Socket, data: pointer, size: int, timeout: int): int
```
Low-level read into a raw memory buffer. For buffered sockets, attempts to read all `size` bytes. For unbuffered sockets, returns whatever the OS provides. Returns 0 on connection close, negative on error.

---

### `recv` (string, write to var)
```nim
proc recv*(socket: Socket, data: var string, size: int, timeout = -1,
           flags = {SocketFlag.SafeDisconn}): int
```
Reads up to `size` bytes into string `data`. Returns the actual number of bytes read. On disconnection returns 0 and sets `data = ""`. Raises `OSError` on error. Timeout is in milliseconds (-1 = no limit).

```nim
var data: string
let n = socket.recv(data, 1024, timeout = 5000)
echo "Received: ", n, " bytes"
```

---

### `recv` (string, return value)
```nim
proc recv*(socket: Socket, size: int, timeout = -1,
           flags = {SocketFlag.SafeDisconn}): string
```
Convenient overload returning a string directly. Returns `""` when the connection is closed.

```nim
let response = socket.recv(4096)
if response == "":
  echo "Connection closed"
```

---

### `readLine`
```nim
proc readLine*(socket: Socket, line: var string, timeout = -1,
               flags = {SocketFlag.SafeDisconn}, maxLength = MaxLineLength)
```
Reads a line from the socket. The trailing `\r\L` is stripped, unless the line consists solely of `\r\L`. On disconnection, sets `line = ""`. Timeout in ms. `maxLength` guards against DoS by truncating long lines.

```nim
var line: string
socket.readLine(line, timeout = 10_000)
echo "Line: ", line
```

---

### `recvLine`
```nim
proc recvLine*(socket: Socket, timeout = -1,
               flags = {SocketFlag.SafeDisconn},
               maxLength = MaxLineLength): string
```
Convenience wrapper around `readLine` that returns the string directly.

```nim
let line = socket.recvLine(timeout = 5000)
```

---

### `recvFrom` (UDP)
```nim
proc recvFrom[T: string | IpAddress](socket: Socket, data: var string, length: int,
               address: var T, port: var Port, flags = 0'i32): int
```
Receives a UDP packet. Writes the payload to `data`, the sender's address to `address` (string or `IpAddress`), and the port to `port`. Returns the number of bytes read. Raises an assertion error on TCP sockets.

**Warning:** No buffered implementation exists — data already in the socket buffer is not returned.

```nim
var data = ""
var address = ""
var port: Port
let udpSock = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
udpSock.bindAddr(Port(9999))
let n = udpSock.recvFrom(data, 1024, address, port)
echo "From ", address, ":", port, " → ", data
```

---

### `skip`
```nim
proc skip*(socket: Socket, size: int, timeout = -1)
```
Discards `size` bytes from the socket. Optional timeout in ms.

```nim
socket.skip(4)  # skip 4-byte header
```

---

## Closing a Socket

### `close`
```nim
proc close*(socket: Socket, flags = {SocketFlag.SafeDisconn})
```
Closes the socket. For SSL sockets, sends a `close_notify` alert to the peer first. If `SafeDisconn` is set, errors during the notification are silently ignored. Resets `fd` to `osInvalidSocket`.

```nim
socket.close()
```

---

## Socket Options

### `getSockOpt`
```nim
proc getSockOpt*(socket: Socket, opt: SOBool, level = SOL_SOCKET): bool
```
Returns the value of a boolean socket option.

```nim
let reuse = socket.getSockOpt(OptReuseAddr)
echo "ReuseAddr: ", reuse
```

---

### `setSockOpt`
```nim
proc setSockOpt*(socket: Socket, opt: SOBool, value: bool, level = SOL_SOCKET)
```
Sets a boolean socket option.

```nim
socket.setSockOpt(OptReuseAddr, true)
socket.setSockOpt(OptReusePort, true)
socket.setSockOpt(OptNoDelay, true, level = IPPROTO_TCP.cint)
socket.setSockOpt(OptKeepAlive, true)
```

---

### `toCInt`
```nim
proc toCInt*(opt: SOBool): cint
```
Converts a `SOBool` to the corresponding OS constant `cint` for use in native system calls.

---

### `getLocalAddr`
```nim
proc getLocalAddr*(socket: Socket): (string, Port)
```
Returns the local address and port of the socket. High-level wrapper for `getsockname`.

```nim
let (addr, port) = socket.getLocalAddr()
echo "Listening on: ", addr, ":", port
```

---

### `getPeerAddr`
```nim
proc getPeerAddr*(socket: Socket): (string, Port)
```
Returns the remote endpoint's address and port. High-level wrapper for `getpeername`.

```nim
let (peerAddr, peerPort) = socket.getPeerAddr()
echo "Connected to: ", peerAddr, ":", peerPort
```

---

### `getFd`
```nim
proc getFd*(socket: Socket): SocketHandle
```
Returns the native file descriptor of the socket for use in system calls.

---

### `isSsl`
```nim
proc isSsl*(socket: Socket): bool
```
Returns `true` if the socket is wrapped in SSL/TLS.

---

## Buffering and State

### `hasDataBuffered`
```nim
proc hasDataBuffered*(s: Socket): bool
```
Returns `true` if the socket's internal buffer contains unread data. Useful for select-like patterns without blocking.

```nim
if socket.hasDataBuffered:
  let line = socket.recvLine()
```

---

## Error Handling

### `socketError`
```nim
proc socketError*(socket: Socket, err: int = -1, async = false,
                  lastError = (-1).OSErrorCode,
                  flags: set[SocketFlag] = {})
```
Inspects the error code and raises the appropriate exception. For SSL sockets uses `SSL_get_error`; for plain sockets uses `osLastError`. With `async = true`, no exception is raised when no data is available yet.

---

### `getSocketError`
```nim
proc getSocketError*(socket: Socket): OSErrorCode
```
Returns the most recent error code. Falls back to the error stored in the socket object if `osLastError` has been reset.

---

### `isDisconnectionError`
```nim
proc isDisconnectionError*(flags: set[SocketFlag], lastError: OSErrorCode): bool
```
Returns `true` if the error code represents a disconnection error and `SafeDisconn` is in `flags`. Handles platform-specific codes (ECONNRESET, EPIPE, WSAECONNRESET, etc.).

---

### `toOSFlags`
```nim
proc toOSFlags*(socketFlags: set[SocketFlag]): cint
```
Converts a `set[SocketFlag]` to a bitmask for system calls. `SafeDisconn` is excluded from the mask.

---

## SSL/TLS API

All procedures below require `-d:ssl`.

### `newContext`
```nim
proc newContext*(protVersion = protSSLv23, verifyMode = CVerifyPeer,
                 certFile = "", keyFile = "", cipherList = CiphersIntermediate,
                 caDir = "", caFile = "", ciphersuites = CiphersModern): SslContext
```
Creates an SSL context. Parameters:
- `verifyMode` — certificate verification mode
- `certFile` / `keyFile` — certificate and private key files (required for servers)
- `cipherList` / `ciphersuites` — allowed cipher suites (TLS 1.2 and TLS 1.3)
- `caDir` / `caFile` — paths to CA root certificates

If `caDir` and `caFile` are not specified, known system paths are scanned. Raises `IOError` if no CA certs are found.

Generate a self-signed certificate:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout mykey.pem -out mycert.pem
```

```nim
# Client context
let ctx = newContext()

# Server context
let serverCtx = newContext(certFile = "mycert.pem", keyFile = "mykey.pem")
```

---

### `wrapSocket`
```nim
proc wrapSocket*(ctx: SslContext, socket: Socket)
```
Wraps an **unconnected** socket in an SSL context. The SSL session starts when the socket connects. Call this before `connect`.

```nim
let socket = newSocket()
let ctx = newContext()
wrapSocket(ctx, socket)
socket.connect("google.com", Port(443))
```

---

### `wrapConnectedSocket`
```nim
proc wrapConnectedSocket*(ctx: SslContext, socket: Socket,
                           handshake: SslHandshakeType, hostname: string = "")
```
Wraps an **already connected** socket in SSL and immediately performs the handshake. `hostname` is required for SNI and certificate name validation.

```nim
# Client side
ctx.wrapConnectedSocket(socket, handshakeAsClient, "example.com")

# Server side
ctx.wrapConnectedSocket(clientSock, handshakeAsServer)
```

---

### `destroyContext`
```nim
proc destroyContext*(ctx: SslContext)
```
Frees memory held by the SSL context.

---

### `sessionIdContext=`
```nim
proc `sessionIdContext=`*(ctx: SslContext, sidCtx: string)
```
Sets the session ID context, enabling TLS clients to resume sessions. Must be at most 32 characters. Should be unique per application. Only meaningful on the server side.

```nim
serverCtx.sessionIdContext = "myapp-v1"
```

---

### `getPeerCertificates`
```nim
proc getPeerCertificates*(sslHandle: SslPtr): seq[Certificate]
proc getPeerCertificates*(socket: Socket): seq[Certificate]
```
Returns the certificate chain received from the peer (DER-encoded). The handshake must have completed successfully; returns an empty sequence otherwise. The chain is ordered from leaf to root.

```nim
let certs = socket.getPeerCertificates()
echo "Certificates received: ", certs.len
```

---

### `gotHandshake`
```nim
proc gotHandshake*(socket: Socket): bool
```
Returns `true` if the SSL handshake completed successfully. Raises `SslError` if the socket is not an SSL socket.

---

### `raiseSSLError`
```nim
proc raiseSSLError*(s = "") {.raises: [SslError].}
```
Raises an `SslError`. If `s` is empty, the error message is taken from the OpenSSL error stack.

---

### `sslHandle`
```nim
proc sslHandle*(self: Socket): SslPtr
```
Returns the OpenSSL `SSL*` pointer for advanced interop with the `openssl` module.

---

### `getExtraData` / `setExtraData`
```nim
proc getExtraData*(ctx: SslContext, index: int): RootRef
proc setExtraData*(ctx: SslContext, index: int, data: RootRef)
```
Store and retrieve arbitrary `RootRef` data inside an `SslContext`. Properly managed through the GC reference count.

---

### PSK (Pre-Shared Key) Functions

```nim
proc `pskIdentityHint=`*(ctx: SslContext, hint: string)
  ## Sets the PSK identity hint sent to the client

proc `clientGetPskFunc=`*(ctx: SslContext, fun: SslClientGetPskFunc)
  ## Sets the client-side PSK callback

proc `serverGetPskFunc=`*(ctx: SslContext, fun: SslServerGetPskFunc)
  ## Sets the server-side PSK callback

proc clientGetPskFunc*(ctx: SslContext): SslClientGetPskFunc
proc serverGetPskFunc*(ctx: SslContext): SslServerGetPskFunc

proc getPskIdentity*(socket: Socket): string
  ## Returns the PSK identity provided by the client
```

Using PSK ciphers (no certificates needed):

```nim
# Server side
let ctx = newContext(verifyMode = CVerifyNone)
ctx.serverGetPskFunc = proc(identity: string): string =
  if identity == "user": return "secretkey"
  return ""
```

---

## SockAddr Utilities

### `toSockAddr`
```nim
proc toSockAddr*(address: IpAddress, port: Port, sa: var Sockaddr_storage, sl: var SockLen)
```
Converts `IpAddress` + `Port` to `Sockaddr_storage` for native system calls.

### `fromSockAddr`
```nim
proc fromSockAddr*(sa: ..., sl: SockLen, address: var IpAddress, port: var Port)
```
Reverse conversion. Raises `ValueError` / `ObjectConversionDefect` for unknown address families.

---

## Quick Recipes

### Simple TCP Client

```nim
import std/net

let socket = newSocket()
socket.connect("example.com", Port(80))
socket.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")

var response = ""
while true:
  let chunk = socket.recv(4096)
  if chunk == "": break
  response.add(chunk)

socket.close()
echo response
```

---

### HTTPS Client (SSL)

```nim
import std/net

let socket = newSocket()
let ctx = newContext()
wrapSocket(ctx, socket)
socket.connect("example.com", Port(443))
socket.send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
echo socket.recvLine()
socket.close()
```

---

### UDP Exchange

```nim
import std/net

# Sender
let sender = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
sender.sendTo("127.0.0.1", Port(9999), "Hello UDP!")

# Receiver
let receiver = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
receiver.bindAddr(Port(9999))
var data = ""
var fromAddr = ""
var fromPort: Port
receiver.recvFrom(data, 1024, fromAddr, fromPort)
echo "Received from ", fromAddr, ": ", data
```

---

### Multithreaded TCP Echo Server

```nim
import std/net, std/threads

proc handleClient(client: Socket) {.thread.} =
  let line = client.recvLine()
  client.send("Echo: " & line & "\r\n")
  client.close()

let server = newSocket()
server.setSockOpt(OptReuseAddr, true)
server.bindAddr(Port(8080))
server.listen()

while true:
  var client: Socket
  var address = ""
  server.acceptAddr(client, address)
  var t: Thread[Socket]
  createThread(t, handleClient, client)
  t.joinThread()
```

---

### SSL Server

```nim
import std/net

let serverCtx = newContext(certFile = "mycert.pem", keyFile = "mykey.pem")
serverCtx.sessionIdContext = "myserver"

let server = newSocket()
server.setSockOpt(OptReuseAddr, true)
server.bindAddr(Port(8443))
server.listen()

while true:
  var client: Socket
  var address = ""
  serverCtx.wrapSocket(server)  # ssl wrapping on server
  server.acceptAddr(client, address)  # handshake happens inside
  let line = client.recvLine()
  client.send("Echo: " & line & "\r\n")
  client.close()
```
