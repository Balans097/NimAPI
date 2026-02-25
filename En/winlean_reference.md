# Nim `winlean.nim` — Windows API Wrapper Reference

> **About this module.** `winlean.nim` is Nim's thin, self-contained wrapper around the subset of the Windows API actually needed by the Nim runtime and standard library. Its name comes from the `WIN32_LEAN_AND_MEAN` preprocessor flag it passes to the C compiler, which strips out rarely-needed Win32 headers. Rather than importing the enormous `windows` module, the Nim runtime imports only this file. It is a semi-public module — it is part of Nim's `std` namespace and can be imported by user code on Windows, but it intentionally exposes only a curated, stable slice of Win32.

---

## Module Setup

```nim
import std/dynlib
{.passc: "-DWIN32_LEAN_AND_MEAN".}
```

`WIN32_LEAN_AND_MEAN` is passed to the C compiler for every file that includes this module. This tells `<windows.h>` to skip Winsock 1.x, multimedia APIs, and other large optional subsystems, significantly reducing compile time.

When `nimPreviewSlimSystem` is defined, `FileHandle` is imported from `std/syncio` and `widestrs` is imported for `WideCString` support.

---

## Primitive Type Aliases

These aliases map Windows SDK type names to Nim types. They exist so that Win32 API signatures can be declared exactly as documented in the Windows SDK.

| Nim name | Underlying type | Windows SDK equivalent | Notes |
|---|---|---|---|
| `WinChar` | `Utf16Char` | `WCHAR` | UTF-16 code unit; used in all `W`-suffixed API calls |
| `Handle` | `int` | `HANDLE` | Opaque OS handle; `INVALID_HANDLE_VALUE = Handle(-1)` |
| `LONG` | `int32` | `LONG` | 32-bit signed integer |
| `ULONG` | `int32` | `ULONG` | 32-bit unsigned integer (note: mapped to signed `int32` in Nim) |
| `PULONG` | `ptr int` | `PULONG` | Pointer to `ULONG` |
| `WINBOOL` | `int32` | `BOOL` | Win32 boolean: **non-zero = success**, zero = failure — the opposite of POSIX convention |
| `PBOOL` | `ptr WINBOOL` | `PBOOL` | Pointer to `WINBOOL` |
| `DWORD` | `int32` | `DWORD` | 32-bit unsigned value (mapped to signed `int32`; use bitwise operations carefully) |
| `PDWORD` | `ptr DWORD` | `PDWORD` | Pointer to `DWORD` |
| `LPINT` | `ptr int32` | `LPINT` | Pointer to `int32` |
| `ULONG_PTR` | `uint` | `ULONG_PTR` | Pointer-sized unsigned integer; used for completion keys and similar |
| `PULONG_PTR` | `ptr uint` | `PULONG_PTR` | Pointer to `ULONG_PTR` |
| `HDC` | `Handle` | `HDC` | Device context handle |
| `HGLRC` | `Handle` | `HGLRC` | OpenGL rendering context handle |
| `BYTE` | `uint8` | `BYTE` | Unsigned 8-bit value |
| `SockLen` | `cuint` | `socklen_t` | Socket address length |
| `SocketHandle` | `distinct int` | `SOCKET` | Distinct type preventing accidental mix-up with `Handle` |
| `WinSizeT` | `uint32` or `uint64` | `SIZE_T` | Pointer-sized unsigned; selects width based on `cpu32` define |

> **`WINBOOL` convention.** Windows `BOOL` is non-zero on success, which is the opposite of POSIX. Use `isSuccess(result)` rather than `result != 0` in your own code to make the intent explicit.

---

## Structures

### `SECURITY_ATTRIBUTES`

```nim
SECURITY_ATTRIBUTES* = object
  nLength*: int32
  lpSecurityDescriptor*: pointer
  bInheritHandle*: WINBOOL
```

Controls whether a newly created kernel object (pipe, file, process, thread) can be inherited by child processes. Set `bInheritHandle = 1` to allow inheritance. `lpSecurityDescriptor = nil` uses the caller's default security descriptor. `nLength` must always be set to `sizeof(SECURITY_ATTRIBUTES)`.

---

### `STARTUPINFO`

```nim
STARTUPINFO* = object
  cb*: int32              # Must be sizeof(STARTUPINFO)
  lpReserved*: cstring    # Must be nil
  lpDesktop*: cstring     # Desktop/window station name, or nil
  lpTitle*: cstring       # Console window title, or nil
  dwX*, dwY*: int32       # Window position (used if STARTF_USESHOWWINDOW)
  dwXSize*, dwYSize*: int32
  dwXCountChars*, dwYCountChars*: int32
  dwFillAttribute*: int32
  dwFlags*: int32         # Combination of STARTF_* flags
  wShowWindow*: int16
  cbReserved2*: int16
  lpReserved2*: pointer
  hStdInput*, hStdOutput*, hStdError*: Handle
```

Passed to `createProcessW`. Key fields: set `dwFlags = STARTF_USESTDHANDLES` and fill `hStdInput`/`hStdOutput`/`hStdError` to redirect standard I/O. Set `dwFlags = STARTF_USESHOWWINDOW` and `wShowWindow = SW_SHOWNORMAL` (or `0` to hide) to control the window state.

---

### `PROCESS_INFORMATION`

```nim
PROCESS_INFORMATION* = object
  hProcess*: Handle
  hThread*: Handle
  dwProcessId*: int32
  dwThreadId*: int32
```

Filled in by `createProcessW` on success. `hProcess` and `hThread` are open handles that **must be closed** with `closeHandle` when no longer needed. `dwProcessId` / `dwThreadId` are the numeric IDs that remain valid even after the handles are closed.

---

### `FILETIME`

```nim
FILETIME* = object  # CANNOT BE int64 BECAUSE OF ALIGNMENT
  dwLowDateTime*: DWORD
  dwHighDateTime*: DWORD
```

A 64-bit timestamp representing 100-nanosecond intervals since January 1, 1601 (UTC). Stored as two `DWORD` fields because Windows guarantees only 4-byte alignment for this struct — placing it in an `int64` would violate the layout on some architectures. Use `rdFileTime` to convert to a plain `int64`, and `toFILETIME` to go the other way.

---

### `BY_HANDLE_FILE_INFORMATION`

```nim
BY_HANDLE_FILE_INFORMATION* = object
  dwFileAttributes*: DWORD
  ftCreationTime*, ftLastAccessTime*, ftLastWriteTime*: FILETIME
  dwVolumeSerialNumber*: DWORD
  nFileSizeHigh*, nFileSizeLow*: DWORD
  nNumberOfLinks*: DWORD
  nFileIndexHigh*, nFileIndexLow*: DWORD
```

Returned by `getFileInformationByHandle`. The file index (`nFileIndexHigh`/`nFileIndexLow`) together with `dwVolumeSerialNumber` uniquely identify an open file and can be used to determine whether two handles refer to the same underlying file (the Windows equivalent of checking inode equality).

---

### `OSVERSIONINFO`

```nim
OSVERSIONINFO* = object
  dwOSVersionInfoSize*: DWORD
  dwMajorVersion*, dwMinorVersion*, dwBuildNumber*, dwPlatformId*: DWORD
  szCSDVersion*: array[0..127, WinChar]
```

Used with `getVersionExW` / `getVersionExA`. Always set `dwOSVersionInfoSize = sizeof(OSVERSIONINFO)` before calling. On Windows 8.1+ the version numbers are manifested (applications report older versions unless they include an appropriate compatibility manifest), so build number detection is more reliable.

---

### `WIN32_FIND_DATA`

```nim
WIN32_FIND_DATA* {.pure.} = object
  dwFileAttributes*: int32
  ftCreationTime*, ftLastAccessTime*, ftLastWriteTime*: FILETIME
  nFileSizeHigh*, nFileSizeLow*: int32
  dwReserved0, dwReserved1: int32     # private
  cFileName*: array[0..MAX_PATH - 1, WinChar]
  cAlternateFileName*: array[0..13, WinChar]
```

Filled by `findFirstFileW` and `findNextFileW`. `cFileName` is the file's long name (up to `MAX_PATH` wide chars). `cAlternateFileName` is the 8.3 short name (may be empty if short name generation is disabled on the volume). Use `rdFileSize` to combine `nFileSizeLow`/`nFileSizeHigh` into a single `int64`.

---

### `OVERLAPPED`

```nim
OVERLAPPED* {.pure, inheritable.} = object
  internal*: PULONG
  internalHigh*: PULONG
  offset*: DWORD
  offsetHigh*: DWORD
  hEvent*: Handle
```

The key structure for asynchronous (overlapped) I/O. Pass a pointer to an `OVERLAPPED` to `readFile`, `writeFile`, `WSARecv`, `WSASend`, etc. to make the call asynchronous. After submitting, use `getQueuedCompletionStatus` on an IOCP port, or poll `hasOverlappedIoCompleted`, to detect completion. `offset`/`offsetHigh` specify the file position for file I/O; for sockets they are ignored. `hEvent` can optionally hold a manual-reset event that gets signalled on completion.

`internal` and `internalHigh` are used by the OS and should not be written by application code; they hold the status code and transferred byte count while the operation is in flight.

---

### `GUID`

```nim
GUID* = object
  D1*: int32
  D2*: int16
  D3*: int16
  D4*: array[0..7, int8]
```

Standard globally unique identifier. Used with `WSAIoctl` to query extension function pointers (`WSAID_CONNECTEX`, `WSAID_ACCEPTEX`, `WSAID_GETACCEPTEXSOCKADDRS`).

---

### `OVERLAPPED`-derived callback types

```nim
POVERLAPPED* = ptr OVERLAPPED
POVERLAPPED_COMPLETION_ROUTINE* = proc(para1: DWORD, para2: DWORD,
                                       para3: POVERLAPPED) {.stdcall.}
```

`POVERLAPPED_COMPLETION_ROUTINE` is the type passed to extended send/receive functions. `para1` is the error code, `para2` is the number of bytes transferred, `para3` is the overlapped structure.

---

### Winsock address structures

| Type | Maps to | Purpose |
|---|---|---|
| `SockAddr` | `SOCKADDR` | Generic socket address (family + opaque data) |
| `Sockaddr_in` | `SOCKADDR_IN` | IPv4 address + port |
| `Sockaddr_in6` | `SOCKADDR_IN6` | IPv6 address + port + flow info + scope |
| `Sockaddr_storage` | `SOCKADDR_STORAGE` | Large enough to hold any socket address; safe for `cast` to a specific type |
| `InAddr` | `IN_ADDR` | Raw IPv4 address as `uint32` |
| `In6_addr` | `IN6_ADDR` | Raw IPv6 address as 16-byte array |
| `AddrInfo` | `ADDRINFOA` | Linked list node from `getaddrinfo`; holds resolved address + metadata |
| `TFdSet` | custom | Socket set for `select`; max `FD_SETSIZE` (64) sockets |
| `Timeval` | `struct timeval` | Timeout for `select`: `tv_sec` seconds + `tv_usec` microseconds |
| `WSAData` | `WSADATA` | Winsock version information filled by `wsaStartup` |
| `TWSABuf` | `WSABUF` | Scatter/gather buffer descriptor: `len` + `buf` pointer |
| `Servent` | `servent` | Service entry (name, port, protocol) |
| `Hostent` | `hostent` | Legacy host entry (name, address list) |

---

### WSA extension function pointer types

These proc types are retrieved at runtime via `WSAIoctl` + `SIO_GET_EXTENSION_FUNCTION_POINTER`:

| Type | Extension | Purpose |
|---|---|---|
| `WSAPROC_ACCEPTEX` | `AcceptEx` | Accept a connection and receive first data in one call |
| `WSAPROC_CONNECTEX` | `ConnectEx` | Connect and send first data in one call (requires bound socket) |
| `WSAPROC_GETACCEPTEXSOCKADDRS` | `GetAcceptExSockaddrs` | Parse local/remote addresses from an `AcceptEx` buffer |

---

### Security types

| Type | Purpose |
|---|---|
| `SID_IDENTIFIER_AUTHORITY` | 6-byte authority component of a Security Identifier |
| `SID` | Security Identifier object (variable length; use `PSID` in practice) |
| `PSID` | `ptr SID` |

---

### Fiber and callback types

```nim
LPFIBER_START_ROUTINE* = proc(param: pointer) {.stdcall.}
WAITORTIMERCALLBACK* = proc(para1: pointer, para2: int32) {.stdcall.}
```

`LPFIBER_START_ROUTINE` is the entry point for a fiber created with `CreateFiber`. `WAITORTIMERCALLBACK` is the callback registered with `registerWaitForSingleObject`.

---

## Constants

### Process and thread control

| Constant | Value | Meaning |
|---|---|---|
| `STARTF_USESHOWWINDOW` | 1 | `wShowWindow` field in `STARTUPINFO` is valid |
| `STARTF_USESTDHANDLES` | 256 | `hStdInput`/`hStdOutput`/`hStdError` in `STARTUPINFO` are valid |
| `HIGH_PRIORITY_CLASS` | 128 | High scheduling priority for `createProcessW` |
| `IDLE_PRIORITY_CLASS` | 64 | Background/idle priority |
| `NORMAL_PRIORITY_CLASS` | 32 | Default priority |
| `REALTIME_PRIORITY_CLASS` | 256 | Real-time priority (use with care) |
| `DETACHED_PROCESS` | 8 | New process has no console |
| `CREATE_NO_WINDOW` | 0x08000000 | Do not create a visible console window |
| `CREATE_UNICODE_ENVIRONMENT` | 1024 | Environment block is Unicode |
| `SW_SHOWNORMAL` | 1 | Show and activate the window normally |

### Wait return values

| Constant | Value | Meaning |
|---|---|---|
| `WAIT_OBJECT_0` | 0 | The object was signalled |
| `WAIT_TIMEOUT` | 0x102 | Timeout elapsed |
| `WAIT_FAILED` | 0xFFFFFFFF | Error — call `getLastError()` |
| `INFINITE` | -1 | Wait forever |
| `STILL_ACTIVE` | 0x103 | Process is still running (from `getExitCodeProcess`) |
| `MAXIMUM_WAIT_OBJECTS` | 64 | Maximum handles passed to `waitForMultipleObjects` |

### Standard handle constants

| Constant | Value | Meaning |
|---|---|---|
| `STD_INPUT_HANDLE` | -10 | Console standard input |
| `STD_OUTPUT_HANDLE` | -11 | Console standard output |
| `STD_ERROR_HANDLE` | -12 | Console standard error |
| `INVALID_HANDLE_VALUE` | `Handle(-1)` | Sentinel for failed handle operations |

### Pipe constants

| Constant | Meaning |
|---|---|
| `PIPE_ACCESS_DUPLEX` | Named pipe is bidirectional |
| `PIPE_ACCESS_INBOUND` | Data flows from client to server only |
| `PIPE_ACCESS_OUTBOUND` | Data flows from server to client only |
| `PIPE_NOWAIT` | Non-blocking mode (deprecated; prefer overlapped I/O) |
| `SYNCHRONIZE` | Required access right to wait on an object |
| `HANDLE_FLAG_INHERIT` | Handle is inheritable by child processes |

### File attributes (`FILE_ATTRIBUTE_*`)

| Constant | Meaning |
|---|---|
| `FILE_ATTRIBUTE_READONLY` | Read-only |
| `FILE_ATTRIBUTE_HIDDEN` | Hidden in Explorer |
| `FILE_ATTRIBUTE_SYSTEM` | OS system file |
| `FILE_ATTRIBUTE_DIRECTORY` | Entry is a directory |
| `FILE_ATTRIBUTE_ARCHIVE` | Marked for backup/archiving |
| `FILE_ATTRIBUTE_NORMAL` | No other attributes set |
| `FILE_ATTRIBUTE_TEMPORARY` | Temporary file; OS may keep in cache |
| `FILE_ATTRIBUTE_REPARSE_POINT` | Has a reparse point (symlink, mount point) |
| `MAX_PATH` | 260 — maximum path length in characters |

### File flags (`FILE_FLAG_*`)

| Constant | Meaning |
|---|---|
| `FILE_FLAG_OVERLAPPED` | Enable overlapped (asynchronous) I/O — required for IOCP |
| `FILE_FLAG_WRITE_THROUGH` | Write operations go to disk, not just cache |
| `FILE_FLAG_NO_BUFFERING` | Bypass OS cache; requires sector-aligned buffers |
| `FILE_FLAG_RANDOM_ACCESS` | Hint: random access pattern (disables read-ahead) |
| `FILE_FLAG_SEQUENTIAL_SCAN` | Hint: sequential access pattern (enables read-ahead) |
| `FILE_FLAG_DELETE_ON_CLOSE` | File is deleted when all handles are closed |
| `FILE_FLAG_BACKUP_SEMANTICS` | Required to open directories or files with backup intent |
| `FILE_FLAG_OPEN_REPARSE_POINT` | Open the reparse point itself, not its target |

### File access and sharing

| Constant | Meaning |
|---|---|
| `GENERIC_READ` | Read access |
| `GENERIC_WRITE` | Write access |
| `GENERIC_ALL` | All access |
| `FILE_SHARE_READ` | Allow simultaneous readers |
| `FILE_SHARE_WRITE` | Allow simultaneous writers |
| `FILE_SHARE_DELETE` | Allow renaming/deletion while open |

### File creation disposition

| Constant | Meaning |
|---|---|
| `CREATE_NEW` | Fail if file already exists |
| `CREATE_ALWAYS` | Always create; truncates existing file |
| `OPEN_EXISTING` | Fail if file does not exist |
| `OPEN_ALWAYS` | Open existing or create new |

### Memory mapping

| Constant | Meaning |
|---|---|
| `PAGE_READONLY` | Mapped region is read-only |
| `PAGE_READWRITE` | Mapped region is read/write |
| `PAGE_EXECUTE_READ` | Mapped region is executable + readable |
| `PAGE_EXECUTE_READWRITE` | Mapped region is executable + read/write |
| `FILE_MAP_READ` | Map with read access |
| `FILE_MAP_WRITE` | Map with write access |

### `MoveFileEx` flags

| Constant | Meaning |
|---|---|
| `MOVEFILE_REPLACE_EXISTING` | Replace destination if it exists |
| `MOVEFILE_COPY_ALLOWED` | Allow cross-volume move (copies + deletes) |
| `MOVEFILE_DELAY_UNTIL_REBOOT` | Schedule rename at next reboot |
| `MOVEFILE_WRITE_THROUGH` | Wait until flush before returning |

### Process access rights

| Constant | Meaning |
|---|---|
| `PROCESS_TERMINATE` | Right to terminate |
| `PROCESS_CREATE_THREAD` | Right to create threads |
| `PROCESS_VM_READ` / `PROCESS_VM_WRITE` | Right to read/write process memory |
| `PROCESS_QUERY_INFORMATION` | Right to query process info |
| `PROCESS_SUSPEND_RESUME` | Right to suspend/resume |
| `PROCESS_ALL_ACCESS` | (not defined here; use SDK directly) |

### Thread pool flags (`WT_*`)

Flags for `registerWaitForSingleObject`. Key values:

| Constant | Meaning |
|---|---|
| `WT_EXECUTEDEFAULT` | Use default thread pool thread |
| `WT_EXECUTEONLYONCE` | Callback fires once, then auto-unregisters |
| `WT_EXECUTELONGFUNCTION` | Hint: callback will run for a long time |
| `WT_EXECUTEINPERSISTENTTHREAD` | Use a dedicated persistent thread |

### Network socket event flags (`FD_*`)

Bits for `wsaEventSelect`. Combine with `or`:

| Constant | Meaning |
|---|---|
| `FD_READ` | Data is available to read |
| `FD_WRITE` | Send buffer has space |
| `FD_ACCEPT` | Incoming connection ready |
| `FD_CONNECT` | Connection completed |
| `FD_CLOSE` | Graceful close notification |
| `FD_OOB` | Out-of-band data |
| `FD_ALL_EVENTS` | All of the above |

### Winsock error codes

| Constant | Value | Meaning |
|---|---|---|
| `WSAEWOULDBLOCK` | 10035 | Non-blocking operation would block |
| `WSAEINPROGRESS` | 10036 | Blocking operation in progress |
| `WSAEINTR` | 10004 | Blocking call interrupted |
| `WSAECONNRESET` | 10054 | Connection reset by peer |
| `WSAECONNABORTED` | 10053 | Connection aborted |
| `WSAEADDRINUSE` | 10048 | Address already in use |
| `WSAETIMEDOUT` | 10060 | Connection timed out |
| `WSAESHUTDOWN` | 10058 | Cannot send after socket shutdown |
| `WSAEDISCON` | 10101 | Graceful shutdown in progress |
| `WSAENETRESET` | 10052 | Network reset |
| `WSANOTINITIALISED` | 10093 | `wsaStartup` not called |
| `WSAENOTSOCK` | 10038 | Not a socket |
| `ERROR_IO_PENDING` | 997 | Overlapped I/O operation pending (also `WSA_IO_PENDING`) |
| `STATUS_PENDING` | 0x103 | I/O operation not yet complete |

### Win32 error codes

| Constant | Value | Meaning |
|---|---|---|
| `NO_ERROR` | 0 | Success |
| `ERROR_FILE_NOT_FOUND` | 2 | File not found |
| `ERROR_PATH_NOT_FOUND` | 3 | Path not found |
| `ERROR_ACCESS_DENIED` | 5 | Permission denied |
| `ERROR_NO_MORE_FILES` | 18 | End of directory enumeration |
| `ERROR_LOCK_VIOLATION` | 33 | File region is locked |
| `ERROR_HANDLE_EOF` | 38 | Read past end of file |
| `ERROR_FILE_EXISTS` | 80 | File already exists |
| `ERROR_BAD_ARGUMENTS` | 165 | Invalid argument |
| `ERROR_NETNAME_DELETED` | 64 | Network name is no longer available |
| `INVALID_FILE_SIZE` | -1 | Returned by `getFileSize` on error |

### Winsock IOC codes

| Constant | Meaning |
|---|---|
| `IOC_OUT` | Output parameter is populated by the call |
| `IOC_IN` | Input parameter is consumed by the call |
| `IOC_INOUT` | Both |
| `IOC_WS2` | Winsock2-specific ioctl |
| `SIO_GET_EXTENSION_FUNCTION_POINTER` | Query for `AcceptEx`, `ConnectEx`, etc. |
| `SO_UPDATE_ACCEPT_CONTEXT` | Required after `AcceptEx` to finish socket inheritance |

### Socket address families

| Constant | Value | Meaning |
|---|---|---|
| `AF_UNSPEC` | 0 | Unspecified |
| `AF_INET` | 2 | IPv4 |
| `AF_INET6` | 23 | IPv6 |

### IOCP / socket address constants

| Constant | Meaning |
|---|---|
| `AI_V4MAPPED` | Return IPv4-mapped IPv6 addresses if no IPv6 found |
| `INADDR_ANY` | Bind to all local IPv4 interfaces |
| `INADDR_LOOPBACK` | IPv4 loopback (127.0.0.1) |
| `INADDR_BROADCAST` | Limited broadcast |
| `FD_SETSIZE` | Maximum sockets in a `TFdSet` (64 on Windows) |
| `MSG_PEEK` | Peek at incoming data without consuming it |

### Security constants

| Constant | Meaning |
|---|---|
| `SECURITY_NT_AUTHORITY` | Authority identifier for standard Windows SIDs |
| `SECURITY_BUILTIN_DOMAIN_RID` | 32 — the built-in domain |
| `DOMAIN_ALIAS_RID_ADMINS` | 544 — the Administrators alias |

### Fiber constants

| Constant | Value | Meaning |
|---|---|---|
| `FIBER_FLAG_FLOAT_SWITCH` | 0x01 | Save/restore floating-point state on fiber switch |

---

## Nim helper functions

These are pure Nim functions in `winlean.nim`, not Win32 imports.

### `isSuccess`

```nim
proc isSuccess*(a: WINBOOL): bool {.inline.}
```

Returns `a != 0`. Exists purely to document intent: Windows `BOOL` uses the opposite convention from POSIX, where `0` means success. Use `isSuccess(result)` instead of `result != 0` when checking Win32 call results.

```nim
let ok = closeHandle(h)
if not ok.isSuccess:
  raise newException(OSError, "CloseHandle failed: " & $getLastError())
```

---

### `rdFileTime`

```nim
proc rdFileTime*(f: FILETIME): int64
```

Combines the two `DWORD` halves of a `FILETIME` into a single `int64` representing 100-nanosecond intervals since January 1, 1601. The combination uses unsigned 32-bit casts to avoid sign-extension of the low word.

```nim
var info: BY_HANDLE_FILE_INFORMATION
discard getFileInformationByHandle(h, addr info)
let writeTime = rdFileTime(info.ftLastWriteTime)
```

---

### `rdFileSize`

```nim
proc rdFileSize*(f: WIN32_FIND_DATA): int64
```

Combines `nFileSizeLow` and `nFileSizeHigh` from a `WIN32_FIND_DATA` into a single `int64` file size. Same unsigned-cast technique as `rdFileTime`.

---

### `toFILETIME`

```nim
proc toFILETIME*(t: int64): FILETIME
```

Converts a 64-bit Windows timestamp (100-ns intervals since 1601) back to a `FILETIME` struct. Useful when calling `setFileTime`.

```nim
let ft = toFILETIME(someTimestamp)
discard setFileTime(h, nil, nil, ft.addr)
```

---

### `FD_ISSET` / `FD_SET` / `FD_ZERO`

```nim
proc FD_ISSET*(socket: SocketHandle, set: var TFdSet): cint
proc FD_SET*(socket: SocketHandle, s: var TFdSet)
proc FD_ZERO*(s: var TFdSet)
```

The standard `fd_set` manipulation macros from BSD sockets, implemented as Nim procedures. Windows does not export them from a header in a way Nim can bind directly, so they are re-implemented here.

- `FD_ZERO` clears the set (sets `fd_count` to 0).
- `FD_SET` adds a socket to the set (up to `FD_SETSIZE` sockets).
- `FD_ISSET` tests whether a socket is in the set (wraps `__WSAFDIsSet`).

---

### `hasOverlappedIoCompleted`

```nim
template hasOverlappedIoCompleted*(lpOverlapped): bool =
  (cast[uint](lpOverlapped.internal) != STATUS_PENDING)
```

Tests whether an overlapped I/O operation has completed by inspecting the `internal` field. This is the Nim equivalent of the `HasOverlappedIoCompleted` C macro. Returns `true` once the operation finishes (the OS writes the final status into `internal`).

---

### `WSAIORW`

```nim
template WSAIORW*(x, y): untyped = (IOC_INOUT or x or y)
```

Computes a bidirectional Winsock ioctl code. Used to build `SIO_GET_EXTENSION_FUNCTION_POINTER`.

---

### `inet_ntop` (compatibility wrapper)

```nim
proc inet_ntop*(family: cint, paddr: pointer, pStringBuffer: cstring,
                stringBufSize: int32): cstring {.stdcall.}
```

Converts a binary network address to its text representation (e.g., `"192.168.1.1"`). On Windows Vista and later, delegates to the native `inet_ntop` from `ws2_32.dll`. On Windows XP, falls back to `inet_ntop_emulated`, which uses `WSAAddressToStringA` internally. The real function pointer is resolved at module load time via `dynlib.loadLib`.

Parameters match POSIX: `family` is `AF_INET` or `AF_INET6`, `paddr` points to an `InAddr` or `In6_addr`, `pStringBuffer` receives the result, `stringBufSize` is the buffer capacity. Returns `nil` on failure.

---

## Windows API Functions

### Error handling

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `getLastError` | `(): int32` | kernel32 | Retrieve the last Win32 error code for the calling thread |
| `setLastError` | `(error: int32)` | kernel32 | Set the last error code (useful in emulation code) |
| `formatMessageW` | `(dwFlags, lpSource, dwMessageId, dwLanguageId: int32, lpBuffer: pointer, nSize: int32, arguments: pointer): int32` | kernel32 | Format a system error message from its code; pass `dwFlags = 0x1300` (FORMAT_MESSAGE_FROM_SYSTEM \| ALLOCATE_BUFFER \| IGNORE_INSERTS) for typical use |
| `localFree` | `(p: pointer)` | kernel32 | Free a buffer allocated by `formatMessageW` when `FORMAT_MESSAGE_ALLOCATE_BUFFER` was used |
| `wsaGetLastError` | `(): cint` | ws2_32 | Winsock-specific last error (equivalent to `getLastError` for socket operations) |

---

### Handle management

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `closeHandle` | `(hObject: Handle): WINBOOL` | kernel32 | Close any kernel object handle; must be called for every handle returned by kernel32 |
| `duplicateHandle` | `(hSourceProcessHandle, hSourceHandle, hTargetProcessHandle: Handle, lpTargetHandle: ptr Handle, dwDesiredAccess: DWORD, bInheritHandle: WINBOOL, dwOptions: DWORD): WINBOOL` | kernel32 | Duplicate a handle, optionally into another process; use `DUPLICATE_SAME_ACCESS` for `dwOptions` to copy access rights |
| `getHandleInformation` | `(hObject: Handle, lpdwFlags: ptr DWORD): WINBOOL` | kernel32 | Query handle flags (e.g., `HANDLE_FLAG_INHERIT`) |
| `setHandleInformation` | `(hObject: Handle, dwMask: DWORD, dwFlags: DWORD): WINBOOL` | kernel32 | Set handle flags |
| `getCurrentProcess` | `(): Handle` | kernel32 | Return a pseudo-handle for the current process; does not need to be closed |

---

### Process management

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `createProcessW` | `(lpApplicationName, lpCommandLine: WideCString, ..., lpStartupInfo: var STARTUPINFO, lpProcessInformation: var PROCESS_INFORMATION): WINBOOL` | kernel32 | Spawn a new process; fills `PROCESS_INFORMATION` on success |
| `terminateProcess` | `(hProcess: Handle, uExitCode: int): WINBOOL` | kernel32 | Forcibly terminate a process with the given exit code |
| `getExitCodeProcess` | `(hProcess: Handle, lpExitCode: var int32): WINBOOL` | kernel32 | Get exit code; returns `STILL_ACTIVE` if the process is running |
| `openProcess` | `(dwDesiredAccess: DWORD, bInheritHandle: WINBOOL, dwProcessId: DWORD): Handle` | kernel32 | Open a handle to an existing process by PID |
| `suspendThread` | `(hThread: Handle): int32` | kernel32 | Suspend a thread; returns previous suspend count |
| `resumeThread` | `(hThread: Handle): int32` | kernel32 | Resume a thread; returns previous suspend count |
| `getSystemTimes` | `(lpIdleTime, lpKernelTime, lpUserTime: var FILETIME): WINBOOL` | kernel32 | Get system-wide CPU time breakdown |
| `getProcessTimes` | `(hProcess: Handle; lpCreationTime, lpExitTime, lpKernelTime, lpUserTime: var FILETIME): WINBOOL` | kernel32 | Get per-process CPU times |

---

### Waiting and synchronisation

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `waitForSingleObject` | `(hHandle: Handle, dwMilliseconds: int32): int32` | kernel32 | Block until a handle is signalled or timeout expires; returns `WAIT_OBJECT_0`, `WAIT_TIMEOUT`, or `WAIT_FAILED` |
| `waitForMultipleObjects` | `(nCount: DWORD, lpHandles: PWOHandleArray, bWaitAll: WINBOOL, dwMilliseconds: DWORD): DWORD` | kernel32 | Wait on up to 64 handles simultaneously |
| `createEvent` | `(lpEventAttributes: ptr SECURITY_ATTRIBUTES, bManualReset: DWORD, bInitialState: DWORD, lpName: ptr Utf16Char): Handle` | kernel32 | Create a manual-reset or auto-reset event object |
| `setEvent` | `(hEvent: Handle): cint` | kernel32 | Signal an event |
| `registerWaitForSingleObject` | `(phNewWaitObject: ptr Handle, hObject: Handle, Callback: WAITORTIMERCALLBACK, Context: pointer, dwMilliseconds: ULONG, dwFlags: ULONG): bool` | kernel32 | Register a callback to fire when a handle is signalled (thread pool) |
| `unregisterWait` | `(WaitHandle: Handle): DWORD` | kernel32 | Cancel a registration made with `registerWaitForSingleObject` |
| `postQueuedCompletionStatus` | `(CompletionPort: Handle, dwNumberOfBytesTransferred: DWORD, dwCompletionKey: ULONG_PTR, lpOverlapped: pointer): bool` | kernel32 | Post a synthetic completion packet to an IOCP port |
| `sleep` | `(dwMilliseconds: int32)` | kernel32 | Suspend the calling thread |

---

### I/O Completion Ports (IOCP)

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `createIoCompletionPort` | `(FileHandle: Handle, ExistingCompletionPort: Handle, CompletionKey: ULONG_PTR, NumberOfConcurrentThreads: DWORD): Handle` | kernel32 | Create or associate a handle with an IOCP port; call with `FileHandle = INVALID_HANDLE_VALUE` and `ExistingCompletionPort = 0` to create a new port |
| `getQueuedCompletionStatus` | `(CompletionPort: Handle, lpNumberOfBytesTransferred: PDWORD, lpCompletionKey: PULONG_PTR, lpOverlapped: ptr POVERLAPPED, dwMilliseconds: DWORD): WINBOOL` | kernel32 | Dequeue a completed I/O notification; `lpOverlapped` is `nil` on timeout |
| `getOverlappedResult` | `(hFile: Handle, lpOverlapped: POVERLAPPED, lpNumberOfBytesTransferred: var DWORD, bWait: WINBOOL): WINBOOL` | kernel32 | Retrieve the result of a completed overlapped I/O operation |

---

### File I/O

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `createFileW` | `(lpFileName: WideCString, dwDesiredAccess, dwShareMode: DWORD, lpSecurityAttributes: pointer, dwCreationDisposition, dwFlagsAndAttributes: DWORD, hTemplateFile: Handle): Handle` | kernel32 | Open or create a file, directory, pipe, or device |
| `createFileA` | same but `lpFileName: cstring` | kernel32 | ANSI version of `createFileW` |
| `deleteFileW` / `deleteFileA` | `(pathName: WideCString/cstring): int32` | kernel32 | Delete a file |
| `readFile` | `(hFile: Handle, buffer: pointer, nNumberOfBytesToRead: int32, lpNumberOfBytesRead: ptr int32, lpOverlapped: pointer): WINBOOL` | kernel32 | Read from a file or pipe; pass `nil` for `lpOverlapped` for synchronous I/O |
| `writeFile` | `(hFile: Handle, buffer: pointer, nNumberOfBytesToWrite: int32, lpNumberOfBytesWritten: ptr int32, lpOverlapped: pointer): WINBOOL` | kernel32 | Write to a file or pipe |
| `flushFileBuffers` | `(hFile: Handle): WINBOOL` | kernel32 | Flush OS cache to disk |
| `setEndOfFile` | `(hFile: Handle): WINBOOL` | kernel32 | Truncate or extend a file to the current file pointer position |
| `setFilePointer` | `(hFile: Handle, lDistanceToMove: LONG, lpDistanceToMoveHigh: ptr LONG, dwMoveMethod: DWORD): DWORD` | kernel32 | Move the file pointer; `dwMoveMethod` is `FILE_BEGIN`, `FILE_CURRENT`, or `FILE_END` |
| `getFileSize` | `(hFile: Handle, lpFileSizeHigh: ptr DWORD): DWORD` | kernel32 | Get file size; combine returned `DWORD` with `*lpFileSizeHigh` for large files |
| `getFileInformationByHandle` | `(hFile: Handle, lpFileInformation: ptr BY_HANDLE_FILE_INFORMATION): WINBOOL` | kernel32 | Get detailed metadata including creation/modification times and hard link count |
| `setFileTime` | `(hFile: Handle, lpCreationTime, lpLastAccessTime, lpLastWriteTime: LPFILETIME): WINBOOL` | kernel32 | Set file timestamps; pass `nil` for any you do not want to change |
| `setFileAttributesW` | `(lpFileName: WideCString, dwFileAttributes: int32): WINBOOL` | kernel32 | Change file attributes |
| `getFileAttributesW` | `(lpFileName: WideCString): int32` | kernel32 | Query file attributes; returns `-1` on failure |

---

### Directory and path operations

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `createDirectoryW` | `(pathName: WideCString, security: pointer = nil): int32` | kernel32 | Create a single directory (not recursive) |
| `removeDirectoryW` | `(lpPathName: WideCString): int32` | kernel32 | Delete an empty directory |
| `getCurrentDirectoryW` | `(nBufferLength: int32, lpBuffer: WideCString): int32` | kernel32 | Get current working directory |
| `setCurrentDirectoryW` | `(lpPathName: WideCString): int32` | kernel32 | Set current working directory |
| `getFullPathNameW` | `(lpFileName: WideCString, nBufferLength: int32, lpBuffer: WideCString, lpFilePart: var WideCString): int32` | kernel32 | Resolve a path to its absolute form |
| `findFirstFileW` | `(lpFileName: WideCString, lpFindFileData: var WIN32_FIND_DATA): Handle` | kernel32 | Begin directory enumeration; returns search handle |
| `findNextFileW` | `(hFindFile: Handle, lpFindFileData: var WIN32_FIND_DATA): int32` | kernel32 | Advance to the next entry in a directory enumeration |
| `findClose` | `(hFindFile: Handle)` | kernel32 | Close a search handle from `findFirstFileW` |
| `createSymbolicLinkW` | `(lpSymlinkFileName, lpTargetFileName: WideCString, flags: DWORD): int32` | kernel32 | Create a symbolic link (requires `SeCreateSymbolicLinkPrivilege` or Developer Mode) |
| `createHardLinkW` | `(lpFileName, lpExistingFileName: WideCString, security: pointer = nil): int32` | kernel32 | Create a hard link |
| `moveFileW` | `(lpExistingFileName, lpNewFileName: WideCString): WINBOOL` | kernel32 | Rename a file |
| `moveFileExW` | `(lpExistingFileName, lpNewFileName: WideCString, flags: DWORD): WINBOOL` | kernel32 | Rename with flags (e.g., `MOVEFILE_REPLACE_EXISTING`) |
| `copyFileW` | `(lpExistingFileName, lpNewFileName: WideCString, bFailIfExists: WINBOOL): WINBOOL` | kernel32 | Copy a file |

---

### Memory-mapped files

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `createFileMappingW` | `(hFile: Handle, lpFileMappingAttributes: pointer, flProtect, dwMaximumSizeHigh, dwMaximumSizeLow: DWORD, lpName: pointer): Handle` | kernel32 | Create a file mapping object; pass `hFile = INVALID_HANDLE_VALUE` for anonymous (RAM-backed) mapping |
| `mapViewOfFileEx` | `(hFileMappingObject: Handle, dwDesiredAccess: DWORD, dwFileOffsetHigh, dwFileOffsetLow: DWORD, dwNumberOfBytesToMap: WinSizeT, lpBaseAddress: pointer): pointer` | kernel32 | Map a view of the file into process address space |
| `unmapViewOfFile` | `(lpBaseAddress: pointer): WINBOOL` | kernel32 | Unmap a previously mapped view |
| `flushViewOfFile` | `(lpBaseAddress: pointer, dwNumberOfBytesToFlush: DWORD): WINBOOL` | kernel32 | Flush dirty pages of a mapped view to disk |

---

### Pipes

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `createPipe` | `(hReadPipe, hWritePipe: var Handle, lpPipeAttributes: var SECURITY_ATTRIBUTES, nSize: int32): WINBOOL` | kernel32 | Create an anonymous pipe; use with `SECURITY_ATTRIBUTES.bInheritHandle` for child process I/O redirection |
| `createNamedPipe` | `(lpName: WideCString, dwOpenMode, dwPipeMode, nMaxInstances, ...: int32, lpSecurityAttributes: ptr SECURITY_ATTRIBUTES): Handle` | kernel32 | Create a named pipe server instance |
| `peekNamedPipe` | `(hNamedPipe: Handle, ...): bool` | kernel32 | Query available bytes without consuming them |

---

### Environment and module

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `getEnvironmentStringsW` | `(): WideCString` | kernel32 | Get the environment block as a double-null-terminated wide string |
| `freeEnvironmentStringsW` | `(para1: WideCString): int32` | kernel32 | Free the block returned by `getEnvironmentStringsW` |
| `setEnvironmentVariableW` | `(lpName, lpValue: WideCString): int32` | kernel32 | Set or delete an environment variable (`lpValue = nil` deletes) |
| `getCommandLineW` | `(): WideCString` | kernel32 | Get the full command line for the current process |
| `getModuleFileNameW` | `(handle: Handle, buf: WideCString, size: int32): int32` | kernel32 | Get the path of the current executable (pass `0` for `handle`) |

---

### Standard I/O handles

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `getStdHandle` | `(nStdHandle: int32): Handle` | kernel32 | Get the handle for stdin, stdout, or stderr using `STD_*_HANDLE` constants |
| `setStdHandle` | `(nStdHandle: int32, hHandle: Handle): WINBOOL` | kernel32 | Redirect a standard handle |
| `get_osfhandle` | `(fd: cint): Handle` | `<io.h>` | Convert a C runtime file descriptor to a Win32 `Handle` |

---

### Time

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `getSystemTimeAsFileTime` | `(lpSystemTimeAsFileTime: var FILETIME)` | kernel32 | Get current UTC time as `FILETIME` (millisecond precision) |
| `getSystemTimePreciseAsFileTime` | `(lpSystemTimeAsFileTime: var FILETIME)` | kernel32 | High-resolution version (100 ns precision); requires Windows 8 / Server 2012 |

---

### Version

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `getVersionExW` | `(lpVersionInfo: ptr OSVERSIONINFO): WINBOOL` | kernel32 | Get Windows version (deprecated on Windows 10; applications must be manifested) |
| `getVersionExA` | same, ANSI | kernel32 | ANSI version |
| `getVersion` | `(): DWORD` | kernel32 | Legacy packed version; low byte = major, high byte of low word = minor |

---

### Shell

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `shellExecuteW` | `(hwnd: Handle, lpOperation, lpFile, lpParameters, lpDirectory: WideCString, nShowCmd: int32): Handle` | shell32 | Open or run a file using the shell (equivalent of double-clicking); `lpOperation = "open"` or `"runas"` for elevation |

---

### Fibers

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `CreateFiber` | `(stackSize: int, fn: LPFIBER_START_ROUTINE, param: pointer): pointer` | kernel32 | Create a fiber with the given stack size and entry point |
| `CreateFiberEx` | `(stkCommit, stkReserve: int, flags: int32, fn: LPFIBER_START_ROUTINE, param: pointer): pointer` | kernel32 | Create a fiber with separate commit/reserve sizes; pass `FIBER_FLAG_FLOAT_SWITCH` to save FP state |
| `ConvertThreadToFiber` | `(param: pointer): pointer` | kernel32 | Convert the calling thread into a fiber |
| `ConvertThreadToFiberEx` | `(param: pointer, flags: int32): pointer` | kernel32 | Convert thread to fiber with flags |
| `DeleteFiber` | `(fiber: pointer)` | kernel32 | Delete a fiber and free its stack |
| `SwitchToFiber` | `(fiber: pointer)` | kernel32 | Switch execution to another fiber |
| `GetCurrentFiber` | `(): pointer` | `windows.h` (inline) | Return the current fiber pointer |

---

### Console

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `readConsoleInput` | `(hConsoleInput: Handle, lpBuffer: pointer, nLength: cint, lpNumberOfEventsRead: ptr cint): cint` | kernel32 | Read raw input events (key, mouse, window resize) from the console |

---

### Winsock initialisation

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `wsaStartup` | `(wVersionRequired: int16, WSData: ptr WSAData): cint` | ws2_32 | Initialise Winsock; must be called before any socket operation; request version `0x0202` for Winsock 2.2 |

---

### Name resolution

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `getaddrinfo` | `(nodename, servname: cstring, hints: ptr AddrInfo, res: var ptr AddrInfo): cint` | ws2_32 | Resolve a hostname/service to a linked list of `AddrInfo`; the modern replacement for `gethostbyname` |
| `freeAddrInfo` | `(ai: ptr AddrInfo)` | ws2_32 | Free the list returned by `getaddrinfo` |
| `getnameinfo` | `(a1: ptr SockAddr, a2: SockLen, ..., a7: cint): cint` | ws2_32 | Reverse-resolve a socket address to a hostname/service string |
| `gethostbyname` | `(name: cstring): ptr Hostent` | ws2_32 | Legacy hostname lookup (IPv4 only; use `getaddrinfo` instead) |
| `gethostbyaddr` | `(ip: ptr InAddr, len: cuint, theType: cint): ptr Hostent` | ws2_32 | Legacy reverse lookup |
| `gethostname` | `(hostname: cstring, len: cint): cint` | ws2_32 | Get the local machine hostname |
| `getservbyname` | `(name, proto: cstring): ptr Servent` | ws2_32 | Look up a service by name |
| `getservbyport` | `(port: cint, proto: cstring): ptr Servent` | ws2_32 | Look up a service by port number |
| `getprotobyname` | `(name: cstring): ptr Protoent` | ws2_32 | Look up a protocol by name (e.g., `"tcp"`) |
| `getprotobynumber` | `(proto: cint): ptr Protoent` | ws2_32 | Look up a protocol by number |

---

### Core socket operations

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `socket` | `(af, typ, protocol: cint): SocketHandle` | ws2_32 | Create a socket |
| `closesocket` | `(s: SocketHandle): cint` | ws2_32 | Close a socket (not `closeHandle`) |
| `bindSocket` | `(s: SocketHandle, name: ptr SockAddr, namelen: SockLen): cint` | ws2_32 | Bind a socket to a local address; note the Nim-level rename from `bind` to avoid keyword conflict |
| `listen` | `(s: SocketHandle, backlog: cint): cint` | ws2_32 | Mark a socket as passive (server) |
| `accept` | `(s: SocketHandle, a: ptr SockAddr, addrlen: ptr SockLen): SocketHandle` | ws2_32 | Accept an incoming connection |
| `connect` | `(s: SocketHandle, name: ptr SockAddr, namelen: SockLen): cint` | ws2_32 | Initiate a connection |
| `shutdown` | `(s: SocketHandle, how: cint): cint` | ws2_32 | Disable send and/or receive on a socket |

---

### Socket I/O

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `send` | `(s: SocketHandle, buf: pointer, len, flags: cint): cint` | ws2_32 | Send data; returns bytes sent or `-1` |
| `sendto` | `(s: SocketHandle, buf: pointer, len, flags: cint, to: ptr SockAddr, tolen: SockLen): cint` | ws2_32 | Send a datagram to a specific address |
| `recv` | `(s: SocketHandle, buf: pointer, len, flags: cint): cint` | ws2_32 | Receive data |
| `recvfrom` | `(s: SocketHandle, buf: cstring, len, flags: cint, fromm: ptr SockAddr, fromlen: ptr SockLen): cint` | ws2_32 | Receive a datagram and record the sender's address |
| `select` | `(nfds: cint, readfds, writefds, exceptfds: ptr TFdSet, timeout: ptr Timeval): cint` | ws2_32 | Multiplexed I/O readiness check; `nfds` is ignored on Windows but must be provided for POSIX compatibility |

---

### Socket options

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `getsockname` | `(s: SocketHandle, name: ptr SockAddr, namelen: ptr SockLen): cint` | ws2_32 | Get local address of a bound/connected socket |
| `getpeername` | `(s: SocketHandle, name: ptr SockAddr, namelen: ptr SockLen): cint` | ws2_32 | Get remote address of a connected socket |
| `getsockopt` | `(s: SocketHandle, level, optname: cint, optval: pointer, optlen: ptr SockLen): cint` | ws2_32 | Read a socket option |
| `setsockopt` | `(s: SocketHandle, level, optname: cint, optval: pointer, optlen: SockLen): cint` | ws2_32 | Write a socket option; use `SOL_SOCKET` for socket-level options, `IPPROTO_TCP` for TCP options |

---

### Overlapped socket I/O (Winsock2)

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `WSARecv` | `(s: SocketHandle, buf: ptr TWSABuf, bufCount: DWORD, bytesReceived, flags: PDWORD, lpOverlapped: POVERLAPPED, completionProc: POVERLAPPED_COMPLETION_ROUTINE): cint` | ws2_32 | Asynchronous receive with scatter buffers |
| `WSARecvFrom` | similar + `name`, `namelen` | ws2_32 | Asynchronous datagram receive |
| `WSASend` | `(s: SocketHandle, buf: ptr TWSABuf, bufCount: DWORD, bytesSent: PDWORD, flags: DWORD, lpOverlapped: POVERLAPPED, completionProc: POVERLAPPED_COMPLETION_ROUTINE): cint` | ws2_32 | Asynchronous send with gather buffers |
| `WSASendTo` | similar + `name`, `namelen` | ws2_32 | Asynchronous datagram send |
| `WSAIoctl` | `(s: SocketHandle, dwIoControlCode: DWORD, lpvInBuffer: pointer, cbInBuffer: DWORD, lpvOutBuffer: pointer, cbOutBuffer: DWORD, lpcbBytesReturned: PDWORD, lpOverlapped: POVERLAPPED, lpCompletionRoutine: POVERLAPPED_COMPLETION_ROUTINE): cint` | ws2_32 | Issue a socket I/O control operation; use with `SIO_GET_EXTENSION_FUNCTION_POINTER` to load `AcceptEx`/`ConnectEx` |

---

### WSA event-based networking

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `wsaCreateEvent` | `(): Handle` | ws2_32 | Create a Winsock event object (manual-reset) |
| `wsaCloseEvent` | `(hEvent: Handle): bool` | ws2_32 | Close a Winsock event |
| `wsaResetEvent` | `(hEvent: Handle): bool` | ws2_32 | Reset (unsignal) a Winsock event |
| `wsaEventSelect` | `(s: SocketHandle, hEventObject: Handle, lNetworkEvents: clong): cint` | ws2_32 | Associate a WSA event with socket network events; the event is signalled when any of the `FD_*` conditions occur |

---

### Legacy address conversion

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `inet_addr` | `(cp: cstring): uint32` | ws2_32 | Parse a dotted-decimal IPv4 string to `uint32`; returns `INADDR_NONE` on error |
| `inet_ntoa` | `(i: InAddr): cstring` | ws2_32 | Convert IPv4 address to dotted-decimal string; result lives in a static buffer — copy it immediately |

---

### Security

| Function | Signature | DLL | Purpose |
|---|---|---|---|
| `allocateAndInitializeSid` | `(pIdentifierAuthority: ptr SID_IDENTIFIER_AUTHORITY, nSubAuthorityCount: BYTE, nSubAuthority0..7: DWORD, pSid: ptr PSID): WINBOOL` | Advapi32 | Allocate and initialise a Security Identifier; supports up to 8 sub-authorities |
| `checkTokenMembership` | `(tokenHandle: Handle, sidToCheck: PSID, isMember: PBOOL): WINBOOL` | Advapi32 | Test whether a token is a member of a given SID (e.g., the Administrators group) |
| `freeSid` | `(pSid: PSID): PSID` | Advapi32 | Free a SID allocated by `allocateAndInitializeSid`; return value should be `nil` |

---

## Usage Patterns

### Check if running as administrator

```nim
import winlean

proc isAdmin(): bool =
  var authority = SID_IDENTIFIER_AUTHORITY(
    value: SECURITY_NT_AUTHORITY
  )
  var adminSid: PSID
  if allocateAndInitializeSid(addr authority, 2,
      SECURITY_BUILTIN_DOMAIN_RID.DWORD,
      DOMAIN_ALIAS_RID_ADMINS.DWORD,
      0, 0, 0, 0, 0, 0,
      addr adminSid) == 0:
    return false
  var isMember: WINBOOL
  discard checkTokenMembership(0.Handle, adminSid, addr isMember)
  discard freeSid(adminSid)
  return isMember.isSuccess
```

### Launch a child process with piped stdout

```nim
import winlean

var sa = SECURITY_ATTRIBUTES(
  nLength: sizeof(SECURITY_ATTRIBUTES).int32,
  bInheritHandle: 1
)
var readPipe, writePipe: Handle
discard createPipe(readPipe, writePipe, sa, 0)
# Make the write end non-inheritable so the child gets a clean copy
discard setHandleInformation(readPipe, HANDLE_FLAG_INHERIT, 0)

var si = STARTUPINFO(cb: sizeof(STARTUPINFO).int32,
  dwFlags: STARTF_USESTDHANDLES,
  hStdOutput: writePipe, hStdError: writePipe,
  hStdInput: getStdHandle(STD_INPUT_HANDLE))
var pi: PROCESS_INFORMATION

if createProcessW(nil, newWideCString("cmd.exe /c dir"),
                  nil, nil, 1, 0, nil, nil, si, pi).isSuccess:
  discard closeHandle(writePipe)  # Close our copy so EOF propagates
  # Read from readPipe ...
  discard closeHandle(pi.hProcess)
  discard closeHandle(pi.hThread)
discard closeHandle(readPipe)
```

### Memory-map a file

```nim
import winlean

let h = createFileW(newWideCString("data.bin"),
  GENERIC_READ.DWORD, FILE_SHARE_READ.DWORD, nil,
  OPEN_EXISTING.DWORD, FILE_ATTRIBUTE_NORMAL.DWORD, 0.Handle)
let mapping = createFileMappingW(h, nil, PAGE_READONLY.DWORD, 0, 0, nil)
let view = mapViewOfFileEx(mapping, FILE_MAP_READ.DWORD, 0, 0, 0, nil)
# use view as a pointer into the file contents ...
discard unmapViewOfFile(view)
discard closeHandle(mapping)
discard closeHandle(h)
```

### Resolve a hostname with `getaddrinfo`

```nim
import winlean

var hints = AddrInfo(ai_family: AF_UNSPEC, ai_socktype: SOCK_STREAM.cint)
var res: ptr AddrInfo
if getaddrinfo("example.com", "443", addr hints, res) == 0:
  var it = res
  while it != nil:
    # use it.ai_addr, it.ai_addrlen ...
    it = it.ai_next
  freeAddrInfo(res)
```
