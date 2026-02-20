# Nim `std/nativesockets` Module — Complete Reference

> Low-level, cross-platform sockets interface. This module maps closely to the BSD/POSIX and WinSock2 socket APIs.  
> For a higher-level, more ergonomic API use the `std/net` module instead.

---

## Table of Contents

1. [Overview](#overview)
2. [Types](#types)
3. [Constants](#constants)
4. [Socket Lifecycle](#socket-lifecycle)
5. [Address Resolution](#address-resolution)
6. [Byte Order Conversion](#byte-order-conversion)
7. [Socket Options](#socket-options)
8. [Address Inspection](#address-inspection)
9. [I/O Readiness — select](#io-readiness--select)
10. [Enum Conversion Utilities](#enum-conversion-utilities)
11. [Platform-Specific Notes](#platform-specific-notes)
12. [Quick Reference Table](#quick-reference-table)

---

## Overview

`std/nativesockets` wraps the underlying OS socket API uniformly across Windows (WinSock2) and POSIX systems (Linux, macOS, BSD, etc.). It re-exports the essential platform constants and low-level types so you can write code that compiles on all supported targets without conditional imports.

```nim
import std/nativesockets

let sock = createNativeSocket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
if sock == osInvalidSocket:
  quit("Could not create socket")
```

---

## Types

### `Port`

```nim
type Port* = distinct uint16
```

A strongly-typed wrapper around a 16-bit port number. Using a distinct type prevents accidental mixing of ports with other integers.

```nim
let httpPort = Port(80)
let httpsPort = Port(443)
echo $httpPort   # → "80"
```

Operators `==` and `$` are provided via borrow.

---

### `Domain`

```nim
type Domain* = enum
  AF_UNSPEC = 0   ## Unspecified; can be auto-detected by getaddrinfo
  AF_UNIX   = 1   ## Unix domain sockets (local IPC). Not available on Windows.
  AF_INET   = 2   ## IPv4
  AF_INET6  = …   ## IPv6 (value is OS-dependent: 10 on Linux, 30 on macOS, 23 on Windows)
```

Specifies the address family / protocol family for a socket.

---

### `SockType`

```nim
type SockType* = enum
  SOCK_STREAM    = 1  ## Reliable, connection-oriented byte stream (TCP)
  SOCK_DGRAM     = 2  ## Unreliable, connectionless datagrams (UDP)
  SOCK_RAW       = 3  ## Raw access to network layer protocols
  SOCK_SEQPACKET = 5  ## Reliable, connection-oriented sequenced packets
```

---

### `Protocol`

```nim
type Protocol* = enum
  IPPROTO_TCP    = 6   ## Transmission Control Protocol
  IPPROTO_UDP    = 17  ## User Datagram Protocol
  IPPROTO_IP          ## Internet Protocol (generic)
  IPPROTO_IPV6        ## IPv6
  IPPROTO_RAW         ## Raw packets (unsupported on Windows)
  IPPROTO_ICMP        ## Internet Control Message Protocol
  IPPROTO_ICMPV6      ## ICMPv6
```

---

### `Servent`

```nim
type Servent* = object
  name*:    string        ## Official service name
  aliases*: seq[string]   ## Alternative names
  port*:    Port          ## Port number
  proto*:   string        ## Protocol name (e.g. "tcp")
```

Returned by `getServByName` and `getServByPort`.

---

### `Hostent`

```nim
type Hostent* = object
  name*:     string        ## Official hostname
  aliases*:  seq[string]   ## Alternative hostnames
  addrtype*: Domain        ## Address family (AF_INET or AF_INET6)
  length*:   int           ## Length of each address in bytes
  addrList*: seq[string]   ## List of IP addresses as strings
```

Returned by `getHostByName` and `getHostByAddr`.

---

## Constants

### `IPPROTO_NONE`

```nim
const IPPROTO_NONE* = IPPROTO_IP
```

Use when the socket type requires a protocol value of zero (e.g. Unix domain sockets).

---

### `osInvalidSocket`

```nim
let osInvalidSocket*: SocketHandle
```

The sentinel value indicating an invalid or failed socket. On POSIX this is `-1`; on Windows it is `INVALID_SOCKET`.

---

### Re-exported socket-option constants

| Constant | Meaning |
|---|---|
| `SO_ERROR` | Retrieve and clear pending socket error |
| `SOL_SOCKET` | Socket-level option layer |
| `SOMAXCONN` | Maximum backlog queue length for `listen` |
| `SO_ACCEPTCONN` | Socket is accepting connections |
| `SO_BROADCAST` | Permit broadcast messages |
| `SO_DEBUG` | Enable debug recording |
| `SO_DONTROUTE` | Bypass routing table |
| `SO_KEEPALIVE` | Enable keep-alive probes |
| `SO_OOBINLINE` | Receive OOB data in-band |
| `SO_REUSEADDR` | Allow reuse of local address |
| `SO_REUSEPORT` | Allow multiple sockets on the same port |
| `MSG_PEEK` | Peek at data without consuming it |
| `SO_NOSIGPIPE` | *(macOS only)* Suppress SIGPIPE |

---

## Socket Lifecycle

### `createNativeSocket` (enum overload)

```nim
proc createNativeSocket*(
    domain:     Domain   = AF_INET,
    sockType:   SockType = SOCK_STREAM,
    protocol:   Protocol = IPPROTO_TCP,
    inheritable: bool    = defined(nimInheritHandles)
): SocketHandle
```

Creates a new OS socket using the high-level Nim enum types.  
Returns `osInvalidSocket` on failure — **no exception is raised**.

`inheritable` controls whether child processes spawned via `exec` inherit the file descriptor. The default follows the compile-time flag `-d:nimInheritHandles`.

```nim
let tcp4 = createNativeSocket()                            # TCP/IPv4
let udp4 = createNativeSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
let tcp6 = createNativeSocket(AF_INET6, SOCK_STREAM, IPPROTO_TCP)
```

---

### `createNativeSocket` (raw cint overload)

```nim
proc createNativeSocket*(
    domain:     cint,
    sockType:   cint,
    protocol:   cint,
    inheritable: bool = defined(nimInheritHandles)
): SocketHandle
```

Use this overload when you need a domain, type, or protocol value not present in the Nim enums. Arguments are passed directly to the OS `socket()` call.

---

### `close`

```nim
proc close*(socket: SocketHandle)
```

Closes the socket and releases the OS file descriptor. Calls `closesocket` on Windows and `close` on POSIX. The return value is discarded — call `getSockOptInt` with `SO_ERROR` beforehand if you need to inspect errors.

```nim
close(sock)
```

---

### `setInheritable`

```nim
proc setInheritable*(s: SocketHandle, inheritable: bool): bool
```

Sets whether the socket handle is inherited by child processes after `exec`. Returns `true` on success.

> **Availability:** Only present on platforms that support `setInheritable` for file handles. Test with `declared(setInheritable)` before calling.

---

### `bindAddr`

```nim
proc bindAddr*(socket: SocketHandle, name: ptr SockAddr, namelen: SockLen): cint
```

Binds `socket` to the address specified by `name`. Returns 0 on success, -1 on error (check `osLastError()`). This is a thin wrapper around the OS `bind()` call.

---

### `listen`

```nim
proc listen*(socket: SocketHandle, backlog = SOMAXCONN): cint
```

Marks `socket` as a passive socket that will accept incoming connections.  
`backlog` is the maximum number of pending connections in the queue.  
Returns 0 on success, -1 on error.

```nim
if listen(serverSock, 128) != 0:
  raiseOSError(osLastError())
```

---

### `accept`

```nim
proc accept*(fd: SocketHandle,
             inheritable = defined(nimInheritHandles)): (SocketHandle, string)
```

Accepts a new incoming connection on `fd`. Returns a tuple of the new client `SocketHandle` and the client's IP address string.  
Returns `(osInvalidSocket, "")` on error.

On Linux and BSDs, uses `accept4` with `SOCK_CLOEXEC` for atomic non-inheritable creation.

```nim
let (clientSock, clientAddr) = accept(serverSock)
if clientSock == osInvalidSocket:
  echo "Accept failed"
else:
  echo "Client connected from ", clientAddr
```

---

## Address Resolution

### `getAddrInfo` (with `AddrInfo` hints)

```nim
proc getAddrInfo*(address: string, port: Port, hints: AddrInfo): ptr AddrInfo
```

Resolves `address` and `port` using the OS `getaddrinfo()`. The `hints` structure controls address family, socket type, and protocol filtering.

> ⚠️ **The returned `ptr AddrInfo` must be freed by calling `freeAddrInfo`.**

---

### `getAddrInfo` (enum overload)

```nim
proc getAddrInfo*(
    address:  string,
    port:     Port,
    domain:   Domain   = AF_INET,
    sockType: SockType = SOCK_STREAM,
    protocol: Protocol = IPPROTO_TCP
): ptr AddrInfo
```

Convenience overload — builds the `AddrInfo` hints from Nim enums. On IPv6 with a non-BSD platform, sets `AI_V4MAPPED` automatically.

> ⚠️ **The returned `ptr AddrInfo` must be freed by calling `freeAddrInfo`.**

```nim
let ai = getAddrInfo("nim-lang.org", Port(80))
defer: freeAddrInfo(ai)
# use ai …
```

---

### `toKnownDomain`

```nim
proc toKnownDomain*(family: cint): Option[Domain]
```

Converts a raw platform `cint` address family value to the `Domain` enum wrapped in an `Option`. Returns `none(Domain)` for unknown families.

```nim
let dom = toKnownDomain(2)  # → some(AF_INET) on all platforms
```

---

### `getServByName` *(not available with `-d:nimNetLite`)*

```nim
proc getServByName*(name, proto: string): Servent
```

Looks up a service by name and protocol (e.g. `"http"`, `"tcp"`). On POSIX, reads `/etc/services`. Raises `OSError` if not found.

```nim
let svc = getServByName("http", "tcp")
echo svc.port   # → Port(80)
```

---

### `getServByPort` *(not available with `-d:nimNetLite`)*

```nim
proc getServByPort*(port: Port, proto: string): Servent
```

Looks up a service by port number and protocol. Raises `OSError` if not found.

```nim
let svc = getServByPort(Port(443), "tcp")
echo svc.name   # → "https"
```

---

### `getHostByName` *(not available with `-d:nimNetLite`)*

```nim
proc getHostByName*(name: string): Hostent
```

Resolves a hostname to IP addresses. Raises `OSError` on failure.

```nim
let h = getHostByName("localhost")
echo h.addrList  # → @["127.0.0.1"]
```

---

### `getHostByAddr` *(not available with `-d:nimNetLite`)*

```nim
proc getHostByAddr*(ip: string): Hostent
```

Reverse-resolves an IP address to a hostname. Raises `OSError` or `IOError` on failure.

```nim
let h = getHostByAddr("8.8.8.8")
echo h.name  # e.g. "dns.google"
```

---

### `getHostname` *(not available with `-d:nimNetLite`)*

```nim
proc getHostname*(): string
```

Returns the local machine's hostname (not the FQDN). Raises `OSError` on failure.

```nim
echo getHostname()  # e.g. "myworkstation"
```

---

### `getProtoByName`

```nim
proc getProtoByName*(name: string): int
```

Returns the numeric protocol code for the given protocol name (e.g. `"tcp"` → `6`). Raises `OSError` if not found.

```nim
echo getProtoByName("udp")  # → 17
```

---

### `makeUnixAddr` *(POSIX only, not available with `-d:nimNetLite`)*

```nim
proc makeUnixAddr*(path: string): Sockaddr_un
```

Creates a `Sockaddr_un` structure for the given Unix domain socket path. Raises `ValueError` if the path is too long.

```nim
let addr = makeUnixAddr("/tmp/my.sock")
```

---

## Byte Order Conversion

Network byte order is big-endian. These functions convert between host byte order and network byte order. On big-endian machines they are no-ops.

### `ntohl`

```nim
proc ntohl*(x: uint32): uint32
```

Converts a 32-bit value from **network** byte order to **host** byte order.

---

### `ntohs`

```nim
proc ntohs*(x: uint16): uint16
```

Converts a 16-bit value from **network** byte order to **host** byte order.

---

### `htonl`

```nim
template htonl*(x: uint32): untyped
```

Converts a 32-bit value from **host** byte order to **network** byte order. Implemented as an alias for `ntohl` (the operation is its own inverse).

---

### `htons`

```nim
template htons*(x: uint16): untyped
```

Converts a 16-bit value from **host** byte order to **network** byte order. Alias for `ntohs`.

```nim
let port = htons(8080'u16)   # → big-endian 8080
let back = ntohs(port)       # → 8080
```

---

## Socket Options

### `getSockOptInt`

```nim
proc getSockOptInt*(socket: SocketHandle, level, optname: int): int
```

Reads an integer socket option. `level` is typically `SOL_SOCKET`. Raises `OSError` on failure.

```nim
let err = getSockOptInt(sock, SOL_SOCKET, SO_ERROR)
if err != 0:
  echo "Pending socket error: ", err
```

---

### `setSockOptInt`

```nim
proc setSockOptInt*(socket: SocketHandle, level, optname, optval: int)
```

Sets an integer socket option. Raises `OSError` on failure.

```nim
setSockOptInt(sock, SOL_SOCKET, SO_REUSEADDR, 1)
setSockOptInt(sock, SOL_SOCKET, SO_KEEPALIVE, 1)
```

---

### `setBlocking`

```nim
proc setBlocking*(s: SocketHandle, blocking: bool)
```

Switches the socket between blocking and non-blocking mode. On Windows uses `ioctlsocket(FIONBIO)`; on POSIX uses `fcntl(F_SETFL, O_NONBLOCK)`. Raises `OSError` on failure.

```nim
setBlocking(sock, false)   # non-blocking (async-friendly)
setBlocking(sock, true)    # blocking (default)
```

---

## Address Inspection

### `getSockDomain`

```nim
proc getSockDomain*(socket: SocketHandle): Domain
```

Returns the address family (`AF_INET` or `AF_INET6`) of an existing socket by calling `getsockname`. Raises `OSError` or `IOError` if the family is unknown.

```nim
let domain = getSockDomain(sock)
```

---

### `getSockName` *(not available with `-d:nimNetLite`)*

```nim
proc getSockName*(socket: SocketHandle): Port
```

Returns the port number the socket is currently bound to. Raises `OSError` on failure.

```nim
let boundPort = getSockName(sock)
echo "Bound to port ", boundPort
```

---

### `getLocalAddr`

```nim
proc getLocalAddr*(socket: SocketHandle, domain: Domain): (string, Port)
```

Returns the socket's local IP address and port as a tuple. The `domain` argument must be `AF_INET` or `AF_INET6`. Raises `OSError` on failure.

```nim
let (ip, port) = getLocalAddr(sock, AF_INET)
echo ip, ":", port
```

---

### `getPeerAddr` *(not available with `-d:nimNetLite`)*

```nim
proc getPeerAddr*(socket: SocketHandle, domain: Domain): (string, Port)
```

Returns the remote peer's IP address and port. Analogous to POSIX `getpeername`. Raises `OSError` on failure.

```nim
let (peerIp, peerPort) = getPeerAddr(clientSock, AF_INET)
echo "Connected to ", peerIp, ":", peerPort
```

---

### `getAddrString` (ptr SockAddr overload)

```nim
proc getAddrString*(sockAddr: ptr SockAddr): string
```

Converts the address inside a raw `SockAddr` pointer to a human-readable string. Handles IPv4, IPv6, and Unix domain sockets. Raises `OSError` or `IOError` on unknown families.

---

### `getAddrString` (pre-allocated string overload) *(not available with `-d:nimNetLite`)*

```nim
proc getAddrString*(sockAddr: ptr SockAddr, strAddress: var string)
```

Writes the address string into `strAddress`. `strAddress` **must be pre-initialized to length 46** before the call.

```nim
var buf = newString(46)
getAddrString(addr mySockAddr, buf)
```

---

## I/O Readiness — select

### `selectRead`

```nim
proc selectRead*(readfds: var seq[SocketHandle], timeout = 500): int
```

Waits until one or more sockets in `readfds` have data available to read. Sockets that are **not** readable are removed from `readfds` in place. Returns the number of readable sockets, 0 on timeout, or -1 on error.

`timeout` is in **milliseconds**. Pass `-1` for an indefinite wait.

```nim
var fds = @[sock1, sock2]
let n = selectRead(fds, timeout = 1000)
if n > 0:
  for fd in fds:
    echo "Ready to read: ", fd
```

---

### `selectWrite`

```nim
proc selectWrite*(writefds: var seq[SocketHandle], timeout = 500): int
```

Waits until one or more sockets in `writefds` are ready to be written to. Sockets that are **not** writable are removed in place. Returns the count of writable sockets, 0 on timeout, -1 on error.

`timeout` is in **milliseconds**. Pass `-1` for an indefinite wait.

```nim
var fds = @[sock]
let n = selectWrite(fds, timeout = 2000)
if n > 0:
  echo "Socket is writable"
```

---

## Enum Conversion Utilities

### `toInt(Domain)`

```nim
proc toInt*(domain: Domain): cint
```

Converts a `Domain` enum value to the platform-native `cint` required by OS calls.

---

### `toInt(SockType)`

```nim
proc toInt*(typ: SockType): cint
```

Converts a `SockType` enum value to the platform-native `cint`.

---

### `toInt(Protocol)`

```nim
proc toInt*(p: Protocol): cint
```

Converts a `Protocol` enum value to the platform-native `cint`.

---

### `toSockType`

```nim
proc toSockType*(protocol: Protocol): SockType
```

Maps a `Protocol` to the appropriate `SockType`:
- `IPPROTO_TCP` → `SOCK_STREAM`
- `IPPROTO_UDP` → `SOCK_DGRAM`
- All others → `SOCK_RAW`

```nim
echo toSockType(IPPROTO_TCP)   # → SOCK_STREAM
echo toSockType(IPPROTO_UDP)   # → SOCK_DGRAM
echo toSockType(IPPROTO_ICMP)  # → SOCK_RAW
```

---

## Platform-Specific Notes

| Topic | Details |
|---|---|
| **Windows** | `wsaStartup` is called automatically at module load. The module exports `WSAEWOULDBLOCK`, `WSAECONNRESET`, `WSAECONNABORTED`, `WSAENETRESET`, `WSANOTINITIALISED`, `WSAENOTSOCK`, `WSAEINPROGRESS`, `WSAEINTR`, `WSAEDISCON`, `ERROR_NETNAME_DELETED`. |
| **POSIX** | Exports `fcntl`, `F_GETFL`, `O_NONBLOCK`, `F_SETFL`, `EAGAIN`, `EWOULDBLOCK`, `MSG_NOSIGNAL`, `EINTR`, `EINPROGRESS`, `ECONNRESET`, `EPIPE`, `ENETRESET`, `EBADF`, `Sockaddr_storage`, `Sockaddr_un`, `Sockaddr_un_path_length`. |
| **macOS** | `SO_NOSIGPIPE` is exported to suppress `SIGPIPE` instead of using `MSG_NOSIGNAL`. |
| **Linux / BSD** | `accept4` is used in `accept` and `createNativeSocket` for atomic `SOCK_CLOEXEC` handling. |
| **`-d:nimNetLite`** | Disables `getServByName`, `getServByPort`, `getHostByName`, `getHostByAddr`, `getHostname`, `getSockName`, `getPeerAddr`, `makeUnixAddr`, and the two-argument `getAddrString`. Intended for resource-constrained environments (FreeRTOS, Zephyr, NuttX). |
| **`AF_UNIX`** | Not supported on Windows. `AF_INET6` numeric value differs by OS (Linux: 10, macOS: 30, Windows: 23). |

---

## Quick Reference Table

| Symbol | Kind | Description |
|---|---|---|
| `Port` | type | Distinct `uint16` port number |
| `Domain` | enum | Address family (`AF_INET`, `AF_INET6`, `AF_UNIX`, `AF_UNSPEC`) |
| `SockType` | enum | Socket type (`SOCK_STREAM`, `SOCK_DGRAM`, `SOCK_RAW`, `SOCK_SEQPACKET`) |
| `Protocol` | enum | Protocol (`IPPROTO_TCP`, `IPPROTO_UDP`, …) |
| `Servent` | object | Service database entry |
| `Hostent` | object | Host database entry |
| `IPPROTO_NONE` | const | Alias for `IPPROTO_IP` (use for zero-protocol sockets) |
| `osInvalidSocket` | let | Sentinel for invalid socket handles |
| `createNativeSocket` | proc ×2 | Create a new socket (enum or raw cint overload) |
| `close` | proc | Close and release a socket |
| `setInheritable` | proc | Set child-process inheritance of socket |
| `bindAddr` | proc | Bind socket to an address |
| `listen` | proc | Mark socket as accepting connections |
| `accept` | proc | Accept incoming connection, return handle + client IP |
| `getAddrInfo` | proc ×2 | Resolve hostname+port to `AddrInfo` (must free!) |
| `toKnownDomain` | proc | Convert raw `cint` family to `Option[Domain]` |
| `getServByName` | proc | Look up service by name + protocol |
| `getServByPort` | proc | Look up service by port + protocol |
| `getHostByName` | proc | Resolve hostname to `Hostent` |
| `getHostByAddr` | proc | Reverse-resolve IP to `Hostent` |
| `getHostname` | proc | Local machine hostname |
| `getProtoByName` | proc | Protocol number by name |
| `makeUnixAddr` | proc | Build `Sockaddr_un` for Unix socket path |
| `ntohl` | proc | Network→host byte order, 32-bit |
| `ntohs` | proc | Network→host byte order, 16-bit |
| `htonl` | template | Host→network byte order, 32-bit |
| `htons` | template | Host→network byte order, 16-bit |
| `getSockOptInt` | proc | Read integer socket option |
| `setSockOptInt` | proc | Write integer socket option |
| `setBlocking` | proc | Toggle blocking/non-blocking mode |
| `getSockDomain` | proc | Get socket's address family |
| `getSockName` | proc | Get bound port number |
| `getLocalAddr` | proc | Get local address + port tuple |
| `getPeerAddr` | proc | Get remote peer address + port tuple |
| `getAddrString` | proc ×2 | Convert `SockAddr*` to string |
| `selectRead` | proc | Wait for readable sockets (select wrapper) |
| `selectWrite` | proc | Wait for writable sockets (select wrapper) |
| `toInt` | proc ×3 | Convert `Domain`/`SockType`/`Protocol` to `cint` |
| `toSockType` | proc | Map `Protocol` → `SockType` |
