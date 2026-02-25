# openssl.nim — Complete Function Reference

> **Module**: `std/openssl` (Nim's Runtime Library)  
> **Purpose**: A thin Nim wrapper around the OpenSSL C library (`libssl` + `libcrypto`). Covers TLS context management, certificate handling, cryptographic digests, HMAC, RSA operations, MD5, BIO I/O, and ALPN/SNI extensions.  
> **Minimum supported version**: OpenSSL ≥ 1.1.0 (dynamic by default).  
> **Build flag**: `-d:ssl` must be passed to the Nim compiler whenever this module is used.

---

## Table of Contents

1. [Key Types](#key-types)
2. [Important Constants](#important-constants)
3. [Library Initialisation](#library-initialisation)
4. [SSL/TLS Method Selection](#ssltls-method-selection)
5. [SSL Context (SslCtx) Management](#ssl-context-sslctx-management)
6. [SSL Connection (SslPtr) Management](#ssl-connection-sslptr-management)
7. [Certificate & Private Key Loading](#certificate--private-key-loading)
8. [BIO — Buffered I/O Abstraction](#bio--buffered-io-abstraction)
9. [Error Handling](#error-handling)
10. [SNI — Server Name Indication](#sni--server-name-indication)
11. [ECDH & Cipher Configuration](#ecdh--cipher-configuration)
12. [PSK — Pre-Shared Key Callbacks](#psk--pre-shared-key-callbacks)
13. [ALPN — Application Layer Protocol Negotiation](#alpn--application-layer-protocol-negotiation)
14. [X.509 Certificate Inspection](#x509-certificate-inspection)
15. [X.509 Certificate Store](#x509-certificate-store)
16. [EVP Digest Functions](#evp-digest-functions)
17. [HMAC](#hmac)
18. [RSA Operations](#rsa-operations)
19. [PEM Key Reading](#pem-key-reading)
20. [EVP Signing (DigestSign)](#evp-signing-digestsign)
21. [MD5 Functions](#md5-functions)
22. [Version & Memory Utilities](#version--memory-utilities)

---

## Key Types

| Type | Description |
|------|-------------|
| `SslPtr` | Opaque pointer that stands in for `SSL*` in the C API. Represents a live TLS connection. |
| `SslCtx` | An alias for `SslPtr` used specifically as a **context** (`SSL_CTX*`). Holds shared settings for many connections. |
| `PSSL_METHOD` | Pointer to an SSL/TLS method descriptor (e.g. TLS 1.2, TLS 1.3). |
| `BIO` | OpenSSL's generic "buffered I/O" object. Can wrap a socket, a memory buffer, an SSL connection, etc. |
| `PX509` | Pointer to an X.509 certificate structure. |
| `PX509_NAME` | Pointer to an X.509 distinguished name (subject or issuer). |
| `EVP_MD` | Pointer to a message-digest algorithm descriptor (SHA-256, etc.). |
| `EVP_MD_CTX` | Context object that holds the in-progress state of a digest computation. |
| `EVP_PKEY` | Pointer to a generic public/private key. |
| `EVP_PKEY_CTX` | Context for operations on `EVP_PKEY` keys (signing, encryption). |
| `PRSA` | Pointer to an RSA key structure. |
| `PaddingType` | Enum for RSA padding schemes: `RSA_PKCS1_PADDING`, `RSA_PKCS1_OAEP_PADDING`, `RSA_NO_PADDING`, etc. |
| `MD5_CTX` | Structure holding the rolling state for an incremental MD5 hash. |
| `pem_password_cb` | Callback type used when reading password-protected PEM keys. |
| `PskClientCallback` | Callback type for PSK client identity/key provisioning. |
| `PskServerCallback` | Callback type for PSK server identity verification. |

---

## Important Constants

### SSL Error Codes (returned by `SSL_get_error`)

| Constant | Value | Meaning |
|----------|-------|---------|
| `SSL_ERROR_NONE` | 0 | No error. |
| `SSL_ERROR_SSL` | 1 | A fatal SSL/TLS protocol error. Check the error queue. |
| `SSL_ERROR_WANT_READ` | 2 | Non-blocking I/O: must wait for data to become readable. |
| `SSL_ERROR_WANT_WRITE` | 3 | Non-blocking I/O: must wait for the socket to become writable. |
| `SSL_ERROR_SYSCALL` | 5 | OS-level I/O error; check `errno`. |
| `SSL_ERROR_ZERO_RETURN` | 6 | The TLS connection was cleanly closed by the peer. |

### SSL Mode Flags (used with `SSLCTXSetMode`)

| Constant | Meaning |
|----------|---------|
| `SSL_MODE_ENABLE_PARTIAL_WRITE` | Allow `SSL_write` to return with fewer bytes than requested (partial write). |
| `SSL_MODE_ACCEPT_MOVING_WRITE_BUFFER` | Allow retry with a different buffer pointer (content must be identical). |
| `SSL_MODE_AUTO_RETRY` | Automatically retry on `WANT_READ`/`WANT_WRITE` in blocking mode. |

### SSL Option Flags (used with `SSL_CTX_ctrl`)

| Constant | Meaning |
|----------|---------|
| `SSL_OP_NO_SSLv2` | Disable SSLv2. |
| `SSL_OP_NO_SSLv3` | Disable SSLv3. |
| `SSL_OP_NO_TLSv1` | Disable TLSv1.0. |
| `SSL_OP_NO_TLSv1_1` | Disable TLSv1.1. |

### Certificate Verification Results

`X509_V_OK` (0) means success. Any non-zero value is an error, for example:

| Constant | Meaning |
|----------|---------|
| `X509_V_ERR_CERT_HAS_EXPIRED` | The certificate has expired. |
| `X509_V_ERR_DEPTH_ZERO_SELF_SIGNED_CERT` | Leaf certificate is self-signed. |
| `X509_V_ERR_CERT_REVOKED` | Certificate has been revoked. |

---

## Library Initialisation

### `SSL_library_init(): cint`

Initialises the OpenSSL library. Must be called **once** before using any other SSL function. On OpenSSL ≥ 1.1.0 this delegates to `OPENSSL_init_ssl`; on older versions it calls the original `SSL_library_init`. The return value can be discarded.

```nim
discard SSL_library_init()
```

### `SSL_load_error_strings()`

Registers human-readable error strings so that `ERR_error_string` can produce meaningful messages. Removed in OpenSSL 1.1.0, so this wrapper becomes a no-op on newer versions.

```nim
SSL_load_error_strings()  # safe to call on any version
```

### `ERR_load_BIO_strings()`

Loads error strings specific to BIO operations. Like `SSL_load_error_strings`, this is a no-op on OpenSSL ≥ 1.1.0.

### `OpenSSL_add_all_algorithms()`

Registers all available digest and cipher algorithms in the internal lookup table. Required before using algorithm-by-name lookups. A no-op on OpenSSL ≥ 1.1.0, but harmless to call.

```nim
SSL_library_init()
SSL_load_error_strings()
OpenSSL_add_all_algorithms()
```

### `getOpenSSLVersion(): culong`

Returns the OpenSSL library version as a packed unsigned long in the format `0xMNN00PP0L` (Major, Minor, patch, etc.). Returns 0 if the library is not available.

```nim
let v = getOpenSSLVersion()
echo v.toHex   # e.g. "10101000" for OpenSSL 1.1.1
```

### `CRYPTO_malloc_init()`

Replaces OpenSSL's internal memory allocator with Nim's `allocShared`/`deallocShared`. Call this early when mixing Nim's GC with OpenSSL to avoid cross-allocator frees. A no-op on platforms where it does not apply.

---

## SSL/TLS Method Selection

These functions return a `PSSL_METHOD` that you pass to `SSL_CTX_new`. They determine which protocol versions are negotiated.

### `TLS_method(): PSSL_METHOD`

The modern, recommended method. Supports SSLv3, TLSv1.0, TLSv1.1, TLSv1.2, and TLSv1.3 (whichever OpenSSL supports). The negotiated version is the highest mutually supported one.

```nim
let ctx = SSL_CTX_new(TLS_method())
```

### `TLS_client_method(): PSSL_METHOD`

Same as `TLS_method()` but explicitly for client-side connections.

### `TLS_server_method(): PSSL_METHOD`

Same as `TLS_method()` but explicitly for server-side connections.

### `SSLv23_method(): PSSL_METHOD`

Compatibility alias for `TLS_method()`. In older OpenSSL the name "SSLv23" was used to mean "negotiate the best available version."

### `SSLv23_client_method(): PSSL_METHOD`

Compatibility alias for `TLS_client_method()`.

### `TLSv1_method(): PSSL_METHOD`

Force TLSv1.0 specifically. Avoid unless you must talk to a legacy peer; TLSv1.0 is deprecated.

### `SSLv2_method() / SSLv3_method(): PSSL_METHOD`

Force SSLv2 or SSLv3. Both are **cryptographically broken**. These are provided only for historic compatibility with very old systems.

---

## SSL Context (SslCtx) Management

An `SslCtx` is a factory and configuration object. You create one context, configure certificates, keys, verification settings, and ciphers on it, then spawn individual `SslPtr` connections from it.

### `SSL_CTX_new(meth: PSSL_METHOD): SslCtx`

Creates a new SSL context using the given method. Returns `nil` on failure.

```nim
let ctx = SSL_CTX_new(TLS_client_method())
if ctx.isNil:
  raise newException(Exception, "Failed to create SSL context")
```

### `SSL_CTX_free(ctx: SslCtx)`

Destroys a context and frees all associated memory. Call when done with a context.

```nim
SSL_CTX_free(ctx)
```

### `SSL_CTX_set_verify(s: SslCtx, mode: int, cb: proc)`

Controls how the peer's certificate is verified.  
- `mode = SSL_VERIFY_NONE` — do not verify (dangerous for clients!).  
- `mode = SSL_VERIFY_PEER` — request and verify the peer certificate.  
- `cb` — optional custom verification callback; pass `nil` to use the default.

```nim
# Require peer certificate verification, default callback
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, nil)
```

### `SSL_CTX_load_verify_locations(ctx, CAfile, CApath): cint`

Tells the context where to find trusted CA certificates. `CAfile` is a PEM file with one or more certificates; `CApath` is a directory of hash-named certificates. Pass `nil` for either to ignore it. Returns 1 on success.

```nim
let ok = SSL_CTX_load_verify_locations(ctx, "/etc/ssl/certs/ca-certificates.crt", nil)
```

### `SSL_CTX_set_cipher_list(s: SslCtx, ciphers: cstring): cint`

Sets the list of allowed TLS 1.0–1.2 cipher suites in OpenSSL's colon-separated notation. Returns 1 if at least one cipher is valid.

```nim
discard SSL_CTX_set_cipher_list(ctx, "HIGH:!aNULL:!MD5")
```

### `SSL_CTX_set_ciphersuites(ctx: SslCtx, str: cstring): cint`

Sets the list of allowed **TLS 1.3** cipher suites. Syntax is different from `set_cipher_list` — use names like `TLS_AES_256_GCM_SHA384`.

```nim
discard SSL_CTX_set_ciphersuites(ctx, "TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256")
```

### `SSL_CTX_ctrl(ctx, cmd, larg, parg): clong`

Low-level control function used internally for many SSL context settings. Prefer the higher-level helpers (`SSLCTXSetMode`, `SSL_CTX_set_ecdh_auto`, etc.) over calling this directly.

### `SSLCTXSetMode(ctx: SslCtx, mode: int): int`

Convenience wrapper around `SSL_CTX_ctrl` that sets one or more mode bits. Returns the new combined mode flags.

```nim
discard SSLCTXSetMode(ctx, SSL_MODE_AUTO_RETRY)
```

### `SSL_CTX_set_session_id_context(context, sid_ctx, sid_ctx_len)`

Sets a session ID context string used to identify sessions created by this context. Important for server-side session resumption when multiple contexts are in use.

### `SSL_CTX_get_ex_new_index / SSL_CTX_set_ex_data / SSL_CTX_get_ex_data`

These three functions allow you to attach arbitrary user data to an `SslCtx` using an index:

1. `SSL_CTX_get_ex_new_index` — register a new index slot.
2. `SSL_CTX_set_ex_data` — store a `pointer` in that slot.
3. `SSL_CTX_get_ex_data` — retrieve the stored pointer.

```nim
let idx = SSL_CTX_get_ex_new_index(0, nil, nil, nil, nil)
var myData = "hello"
discard SSL_CTX_set_ex_data(ctx, idx, addr myData)
let p = SSL_CTX_get_ex_data(ctx, idx)
```

---

## SSL Connection (SslPtr) Management

### `SSL_new(context: SslCtx): SslPtr`

Creates a new SSL connection object from a context. The connection is not yet associated with a socket.

```nim
let ssl = SSL_new(ctx)
```

### `SSL_free(ssl: SslPtr)`

Destroys a connection object and releases its memory.

### `SSL_set_fd(ssl: SslPtr, fd: SocketHandle): cint`

Associates the SSL object with an OS socket file descriptor. After this call, `SSL_connect` or `SSL_accept` can be used to perform the TLS handshake.

```nim
discard SSL_set_fd(ssl, socket.fd)
```

### `SSL_connect(ssl: SslPtr): cint`

Performs the client-side TLS handshake. Returns 1 on success, ≤ 0 on error (use `SSL_get_error` to interpret).

```nim
let ret = SSL_connect(ssl)
if ret != 1:
  let err = SSL_get_error(ssl, ret.cint)
  echo "Handshake failed: ", err
```

### `SSL_accept(ssl: SslPtr): cint`

Performs the server-side TLS handshake. Returns 1 on success.

### `SSL_do_handshake(ssl: SslPtr): cint` *(via `sslDoHandshake`)*

Generic handshake that works for both client and server depending on the connection's state.

### `SSL_read(ssl: SslPtr, buf: pointer, num: int): cint`

Reads decrypted application data from the connection into `buf`. Returns the number of bytes read, 0 on clean close, or negative on error.

```nim
var buf = newString(4096)
let n = SSL_read(ssl, buf.cstring, buf.len.cint)
if n > 0:
  echo buf[0..<n]
```

### `SSL_write(ssl: SslPtr, buf: cstring, num: int): cint`

Encrypts and sends application data. Returns number of bytes written or ≤ 0 on error.

```nim
let msg = "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n"
discard SSL_write(ssl, msg, msg.len)
```

### `SSL_pending(ssl: SslPtr): cint`

Returns the number of bytes already decrypted and buffered inside OpenSSL, ready to be read without another round-trip to the network.

### `SSL_shutdown(ssl: SslPtr): cint`

Sends the TLS `close_notify` alert to begin a clean shutdown. May need to be called twice (first call sends, second call waits for the peer's reply). Returns 0 when the first `close_notify` was sent, 1 when both sides have closed cleanly, <0 on error.

### `SSL_set_shutdown / SSL_get_shutdown`

Manually set or query the shutdown state flags (`SSL_SENT_SHUTDOWN`, `SSL_RECEIVED_SHUTDOWN`).

### `SSL_get_error(s: SslPtr, ret_code: cint): cint`

Translates the raw return value of `SSL_connect`, `SSL_read`, `SSL_write`, etc. into a meaningful error code. Always call this immediately after a failed SSL operation.

```nim
let err = SSL_get_error(ssl, ret.cint)
case err
of SSL_ERROR_WANT_READ:  echo "need more data"
of SSL_ERROR_ZERO_RETURN: echo "connection closed"
else: echo "error ", err
```

### `SSL_get_verify_result(ssl: SslPtr): int`

Returns the result of the peer certificate verification performed during the handshake. Compare against `X509_V_OK` (0).

```nim
if SSL_get_verify_result(ssl) != X509_V_OK:
  raise newException(Exception, "Certificate verification failed")
```

### `SSL_get_SSL_CTX(ssl: SslPtr): SslCtx`

Returns the context associated with this connection.

### `SSL_set_SSL_CTX(ssl: SslPtr, ctx: SslCtx): SslCtx`

Replaces the context of an existing SSL connection. Useful for virtual hosting (SNI callbacks).

### `SSL_get0_verified_chain(ssl: SslPtr): PSTACK`

Returns the verified certificate chain after a successful handshake, as an `PSTACK` (OpenSSL stack). Useful for inspecting intermediate certificates.

### `SSL_in_init(ssl: SslPtr): cint`

Returns non-zero if the TLS handshake is still in progress.

### `sslSetConnectState(s: SslPtr)`

Puts the SSL object into **client** mode before calling `sslDoHandshake`. Used with BIO-based non-blocking I/O.

### `sslSetAcceptState(s: SslPtr)`

Puts the SSL object into **server** mode before calling `sslDoHandshake`.

### `sslRead / sslPeek / sslWrite`

Lower-level aliases for `SSL_read` / a non-consuming read / `SSL_write`, used internally by Nim's networking layer. They accept a `cstring` buffer instead of a raw pointer.

### `sslSetBio(ssl: SslPtr, rbio, wbio: BIO)`

Connects two BIO objects to an SSL handle: `rbio` for reading and `wbio` for writing. Useful when using memory BIOs for non-blocking I/O without raw sockets.

```nim
let rbio = bioNew(bioSMem())
let wbio = bioNew(bioSMem())
sslSetBio(ssl, rbio, wbio)
```

---

## Certificate & Private Key Loading

### `SSL_CTX_use_certificate_file(ctx, filename, typ): cint`

Loads a single certificate from `filename`. `typ` is `SSL_FILETYPE_PEM` (1) or `SSL_FILETYPE_ASN1` (2).

```nim
discard SSL_CTX_use_certificate_file(ctx, "server.crt", SSL_FILETYPE_PEM)
```

### `SSL_CTX_use_certificate_chain_file(ctx, filename): cint`

Loads a certificate *and* any additional chain certificates from a PEM file. Prefer this over `use_certificate_file` for production servers.

```nim
discard SSL_CTX_use_certificate_chain_file(ctx, "fullchain.pem")
```

### `SSL_CTX_use_PrivateKey_file(ctx, filename, typ): cint`

Loads a private key from `filename` (PEM or DER format).

```nim
discard SSL_CTX_use_PrivateKey_file(ctx, "server.key", SSL_FILETYPE_PEM)
```

### `SSL_CTX_check_private_key(ctx: SslCtx): cint`

Verifies that the private key loaded into `ctx` matches the certificate. Returns 1 if they match. Always call this after loading both.

```nim
if SSL_CTX_check_private_key(ctx) != 1:
  raise newException(Exception, "Key/certificate mismatch")
```

---

## BIO — Buffered I/O Abstraction

BIO (Basic I/O) is OpenSSL's I/O abstraction layer. It can represent a socket, an SSL stream, or an in-memory buffer.

### `bioNew(b: PBIO_METHOD): BIO`

Allocates a new BIO of the given type.

### `bioFreeAll(b: BIO)`

Frees a BIO and all BIOs chained to it.

### `bioSMem(): PBIO_METHOD`

Returns the BIO method for an in-memory buffer. Use with `bioNew` to create an in-memory BIO.

```nim
let memBio = bioNew(bioSMem())  # in-memory buffer BIO
```

### `bioCtrlPending(b: BIO): cint`

Returns the number of bytes waiting to be read from the BIO.

### `bioRead(b: BIO, Buf: cstring, length: cint): cint`

Reads `length` bytes from the BIO into `Buf`. Returns the number of bytes read.

### `bioWrite(b: BIO, Buf: cstring, length: cint): cint`

Writes `length` bytes from `Buf` to the BIO.

### `BIO_new_mem_buf(data: pointer, len: cint): BIO`

Creates a read-only memory BIO over an existing in-memory buffer. Useful for feeding PEM data from a string rather than a file.

```nim
let pemData = "-----BEGIN CERTIFICATE-----\n..."
let bio = BIO_new_mem_buf(pemData.cstring, pemData.len.cint)
```

### `BIO_new_ssl_connect(ctx: SslCtx): BIO`

Creates a BIO chain containing both an SSL BIO and a connect BIO. The result is a high-level object you can use to open an SSL connection without managing raw sockets.

```nim
let bio = BIO_new_ssl_connect(ctx)
```

### `BIO_ctrl(bio, cmd, larg, arg): int`

Low-level control function. Prefer the wrappers below.

### `BIO_get_ssl(bio, ssl: ptr SslPtr): int`

Extracts the `SslPtr` from a BIO chain (e.g. one created with `BIO_new_ssl_connect`).

### `BIO_set_conn_hostname(bio, name): int`

Sets the hostname and port for a connect BIO, in `"hostname:port"` form.

```nim
discard BIO_set_conn_hostname(bio, "example.com:443")
```

### `BIO_do_connect(bio) / BIO_do_handshake(bio): int`

Triggers TCP connection and TLS handshake on a connect BIO chain.

```nim
if BIO_do_connect(bio) <= 0:
  ERR_print_errors_fp(stderr)
```

### `BIO_read(b, data, length): cint`

Standard BIO read (direct C-level import, vs. camelCase `bioRead`).

### `BIO_write(b, data, length): cint`

Standard BIO write.

### `BIO_free(b: BIO): cint`

Frees a single BIO object.

---

## Error Handling

### `ERR_get_error(): culong`

Pops and returns the oldest error code from the OpenSSL error queue. Returns 0 when the queue is empty. Loop until 0 to drain all errors.

```nim
var code = ERR_get_error()
while code != 0:
  echo ERR_error_string(code, nil)
  code = ERR_get_error()
```

### `ERR_peek_last_error(): culong`

Returns the most recent error code **without** removing it from the queue.

### `ERR_error_string(e: culong, buf: cstring): cstring`

Converts an error code into a human-readable string. Pass `nil` for `buf` to use an internal static buffer.

```nim
echo ERR_error_string(ERR_get_error(), nil)
```

### `ERR_print_errors_fp(fp: File)`

Dumps the entire error queue to a file (typically `stderr`). Convenient for debugging.

```nim
ERR_print_errors_fp(stderr)
```

### `ErrClearError()`

Clears all errors from the error queue. Call before an operation if you want to check for *new* errors only.

### `ErrFreeStrings()`

Frees memory used by loaded error strings. Call at program shutdown.

### `ErrRemoveState(pid: cint)`

Removes the error state for a thread identified by `pid`. Used for thread cleanup.

---

## SNI — Server Name Indication

SNI allows a TLS client to declare which hostname it wants to connect to inside the ClientHello, so a server can select the right certificate before the handshake completes.

### `SSL_set_tlsext_host_name(ssl: SslPtr, name: cstring): int`

Sets the SNI hostname on the client side. Call before `SSL_connect`. Returns 1 on success.

```nim
discard SSL_set_tlsext_host_name(ssl, "example.com")
```

### `SSL_get_servername(ssl: SslPtr, typ: cint): cstring`

On the **server** side, retrieves the hostname sent by the client in the ClientHello. Typically called inside an SNI callback. May return `nil` if SNI was not used.

```nim
let host = SSL_get_servername(ssl, TLSEXT_NAMETYPE_host_name)
```

### `SSL_CTX_set_tlsext_servername_callback(ctx, cb): int`

Registers a callback invoked when a client hello is received with an SNI hostname. Inside the callback you can call `SSL_set_SSL_CTX` to switch to a different context with a different certificate.

```nim
proc sniCb(ssl: SslPtr, cb_id: int, arg: pointer): int {.cdecl.} =
  let host = SSL_get_servername(ssl, TLSEXT_NAMETYPE_host_name)
  if host == "other.example.com":
    discard SSL_set_SSL_CTX(ssl, otherCtx)
  return SSL_TLSEXT_ERR_OK

discard SSL_CTX_set_tlsext_servername_callback(ctx, sniCb)
```

### `SSL_CTX_set_tlsext_servername_arg(ctx, arg): int`

Sets a user data pointer that will be passed to the SNI callback registered above.

---

## ECDH & Cipher Configuration

### `SSL_CTX_set_ecdh_auto(ctx, onoff: cint): cint`

Enables or disables automatic ECDH curve selection. On OpenSSL ≥ 1.1.0 this is always on, so the function returns 1 without doing anything.

```nim
discard SSL_CTX_set_ecdh_auto(ctx, 1)
```

### `OPENSSL_sk_num(stack: PSTACK): int`

Returns the number of elements in an OpenSSL generic stack (e.g. a certificate chain).

### `OPENSSL_sk_value(stack: PSTACK, index: int): pointer`

Returns the element at `index` from an OpenSSL stack as a raw pointer. Cast to the appropriate type.

### `OPENSSL_config(configName: cstring)`

Loads an OpenSSL configuration file. Pass `nil` for the default (`openssl.cnf`).

---

## PSK — Pre-Shared Key Callbacks

PSK (Pre-Shared Key) TLS allows authentication without certificates, using a shared secret agreed upon out-of-band.

### `SSL_CTX_set_psk_client_callback(ctx, callback)`

Registers a callback invoked on the **client** side when a PSK cipher suite is negotiated. The callback must fill in `identity` and `psk` buffers and return the PSK length.

```nim
proc myPskClient(ssl: SslPtr; hint: cstring; identity: cstring;
                 maxIdentLen: cuint; psk: ptr uint8;
                 maxPskLen: cuint): cuint {.cdecl.} =
  copyMem(identity, "client1", 8)
  let secret = "mysecret"
  copyMem(psk, secret.cstring, secret.len)
  return secret.len.cuint

SSL_CTX_set_psk_client_callback(ctx, myPskClient)
```

### `SSL_CTX_set_psk_server_callback(ctx, callback)`

Same but for the **server** side. The callback receives the identity string and must fill in the PSK buffer.

### `SSL_CTX_use_psk_identity_hint(ctx, hint: cstring): cint`

Sets a hint string that the server sends to the client to help it choose the right PSK identity. Returns 1 on success.

### `SSL_get_psk_identity(ssl: SslPtr): cstring`

Retrieves the PSK identity string chosen by the client after a PSK handshake.

---

## ALPN — Application Layer Protocol Negotiation

ALPN allows client and server to agree on the application protocol (e.g. `"h2"` for HTTP/2, `"http/1.1"`) during the TLS handshake.

### `SSL_CTX_set_alpn_protos(ctx, protos, protos_len): cint`

Sets the list of protocols the **client** is willing to use. `protos` is a wire-format byte string where each protocol is prefixed by its length. Returns 0 on success (note: inverted from most OpenSSL functions!).

```nim
# "h2" in wire format: \x02h2
let protos = "\x02h2\x08http/1.1"
discard SSL_CTX_set_alpn_protos(ctx, protos, protos.len.cuint)
```

### `SSL_set_alpn_protos(ssl, protos, protos_len): cint`

Per-connection version of the above; overrides the context-level setting for one specific SSL object.

### `SSL_CTX_set_alpn_select_cb(ctx, cb, arg)`

Server-side callback for ALPN. OpenSSL calls this with the client's list and the callback selects a protocol. Typically used with `SSL_select_next_proto`.

### `SSL_get0_alpn_selected(ssl, data, len)`

After the handshake, retrieves the negotiated ALPN protocol. `data` will point into OpenSSL's internal buffer (do not free).

```nim
var proto: cstring
var protoLen: cuint
SSL_get0_alpn_selected(ssl, addr proto, addr protoLen)
echo "Negotiated: ", proto[0..<protoLen.int]
```

### `SSL_select_next_proto(out_proto, outlen, server, server_len, client, client_len): cint`

Helper that performs the RFC 7301 intersection of server and client protocol lists. Returns `OPENSSL_NPN_NEGOTIATED` (1) if a match was found.

### `SSL_CTX_set_next_protos_advertised_cb / SSL_CTX_set_next_proto_select_cb / SSL_get0_next_proto_negotiated`

Older NPN (Next Protocol Negotiation) variants, superseded by ALPN but still present for compatibility with older clients.

---

## X.509 Certificate Inspection

### `SSL_get_peer_certificate(ssl: SslCtx): PX509`

Returns the peer's certificate (presented during the handshake). Returns `nil` if no certificate was provided. On OpenSSL 3 this delegates to `SSL_get1_peer_certificate`.

```nim
let cert = SSL_get_peer_certificate(ssl)
if cert.isNil:
  echo "No certificate"
```

### `X509_get_subject_name(a: PX509): PX509_NAME`

Returns the subject `PX509_NAME` of a certificate (the identity of the certificate holder).

### `X509_get_issuer_name(a: PX509): PX509_NAME`

Returns the issuer `PX509_NAME` (the identity of the CA that signed the certificate).

### `X509_NAME_oneline(a: PX509_NAME, buf: cstring, size: cint): cstring`

Converts a `PX509_NAME` to a human-readable one-line string like `/C=US/O=Example/CN=example.com`.

```nim
let subject = X509_NAME_oneline(X509_get_subject_name(cert), nil, 0)
echo subject
```

### `X509_NAME_get_text_by_NID(subject, NID, buf, size): cint`

Extracts the text of a specific field from a name using its NID (Numeric Identifier). For example, NID 13 is `CN` (common name).

### `X509_check_host(cert, name, namelen, flags, peername): cint`

Checks whether the certificate is valid for the given hostname. Returns 1 on match, 0 on no match, -1 on error. This is the correct, RFC-compliant hostname verification function.

```nim
if X509_check_host(cert, "example.com", 11, 0, nil) != 1:
  raise newException(Exception, "Hostname mismatch")
```

### `X509_free(cert: PX509)`

Frees a certificate object. Call this after using a certificate returned by `SSL_get_peer_certificate`.

### `d2i_X509(b: string): PX509`

Decodes a DER/BER-encoded byte string into an in-memory X.509 certificate struct.

```nim
let certBytes = readFile("cert.der")
let cert = d2i_X509(certBytes)
```

### `i2d_X509(cert: PX509): string`

Encodes an in-memory X.509 certificate to a DER byte string.

```nim
let der = i2d_X509(cert)
writeFile("cert.der", der)
```

---

## X.509 Certificate Store

### `X509_STORE_new(): PX509_STORE`

Creates a new, empty certificate store.

### `X509_STORE_free(v: PX509_STORE)`

Frees a certificate store and decrements reference counts of all stored certificates.

### `X509_STORE_add_cert(ctx, x: PX509): cint`

Adds a certificate to the store. Returns 1 on success.

```nim
let store = X509_STORE_new()
discard X509_STORE_add_cert(store, cert)
```

### `X509_STORE_lock / X509_STORE_unlock`

Thread-safe locking/unlocking of a store.

### `X509_STORE_up_ref(v: PX509_STORE): cint`

Increments the reference count of a store. Needed when sharing a store across multiple contexts.

### `X509_STORE_set_flags / set_purpose / set_trust`

Fine-tune store behaviour: set extended validation flags, restrict the intended purpose of certificates, or set the trust model.

### `X509_OBJECT_new() / X509_OBJECT_free`

Allocate and free a generic X.509 store object (wraps certificates, CRLs, etc.).

---

## EVP Digest Functions

EVP (Envelope) is OpenSSL's high-level abstraction for cryptographic operations. Digest algorithms are represented as `EVP_MD` values.

### Algorithm Descriptors

| Function | Algorithm |
|----------|-----------|
| `EVP_md_null()` | No-op (empty digest) |
| `EVP_md5()` | MD5 (128-bit, **do not use for security**) |
| `EVP_sha1()` | SHA-1 (160-bit, deprecated for security) |
| `EVP_sha224()` | SHA-224 |
| `EVP_sha256()` | SHA-256 ✓ recommended |
| `EVP_sha384()` | SHA-384 |
| `EVP_sha512()` | SHA-512 |
| `EVP_ripemd160()` | RIPEMD-160 |
| `EVP_whirlpool()` | Whirlpool |

### `EVP_MD_size(md: EVP_MD): cint`

Returns the output size in bytes of a digest algorithm.

```nim
echo EVP_MD_size(EVP_sha256())  # → 32
```

### `EVP_MD_CTX_create(): EVP_MD_CTX`

Allocates a new digest context. On Linux this maps to `EVP_MD_CTX_new`; on macOS/Windows it maps to `EVP_MD_CTX_create`.

### `EVP_MD_CTX_destroy(ctx: EVP_MD_CTX)`

Frees a digest context. Maps to `EVP_MD_CTX_free` on Linux.

### `EVP_DigestInit_ex(ctx, typ, engine): cint`

Initialises a digest context for the given algorithm. Pass `nil` for `engine` to use the default.

```nim
let ctx = EVP_MD_CTX_create()
discard EVP_DigestInit_ex(ctx, EVP_sha256(), nil)
```

### `EVP_DigestUpdate(ctx, data: pointer, len: cuint): cint`

Feeds more data into the running digest. Can be called multiple times.

```nim
let msg = "hello world"
discard EVP_DigestUpdate(ctx, msg.cstring, msg.len.cuint)
```

### `EVP_DigestFinal_ex(ctx, buffer: pointer, size: ptr cuint): cint`

Finalises the digest and writes the result into `buffer`. Sets `size` to the number of bytes written.

```nim
var hash: array[32, uint8]
var hashLen: cuint
discard EVP_DigestFinal_ex(ctx, addr hash[0], addr hashLen)
```

---

## HMAC

HMAC (Hash-based Message Authentication Code) combines a secret key with a hash function to authenticate data.

### `HMAC(evp_md, key, key_len, d, n, md, md_len): cstring`

One-shot HMAC computation.  
- `evp_md` — hash algorithm (e.g. `EVP_sha256()`).  
- `key` / `key_len` — the secret key.  
- `d` / `n` — the data to authenticate.  
- `md` — output buffer (at least `EVP_MAX_MD_SIZE` bytes); pass `nil` for an internal static buffer.  
- `md_len` — set to the number of bytes written.

```nim
var outLen: cuint
let secret = "mysecret"
let data = "message"
let result = HMAC(EVP_sha256(),
                  secret.cstring, secret.len.cint,
                  data, data.len.csize_t,
                  nil, addr outLen)
# result points to a 32-byte HMAC-SHA256
```

---

## RSA Operations

### `RSA_size(rsa: PRSA): cint`

Returns the RSA modulus size in bytes (key size / 8). The output buffer for encryption must be at least this large.

### `RSA_public_encrypt(flen, fr, to, rsa, padding): cint`

Encrypts `flen` bytes from `fr` using the RSA public key, storing the result in `to`. Returns the ciphertext length or −1 on error.

```nim
var cipher = newString(RSA_size(rsa))
let n = RSA_public_encrypt(msg.len.cint, cast[ptr uint8](msg.cstring),
                            cast[ptr uint8](cipher.cstring), rsa, RSA_PKCS1_OAEP_PADDING)
```

### `RSA_private_decrypt(flen, fr, to, rsa, padding): cint`

Decrypts ciphertext `fr` using the RSA private key.

### `RSA_private_encrypt(flen, fr, to, rsa, padding): cint`

Signs (private-key encrypts) a message. Used in RSA PKCS#1 v1.5 signing (raw; prefer EVP signing for new code).

### `RSA_public_decrypt(flen, fr, to, rsa, padding): cint`

Verifies (public-key decrypts) a signature.

### `RSA_verify(kind, origMsg, origMsgLen, signature, signatureLen, rsa): cint`

Verifies an RSA signature against the original message. `kind` is the NID of the digest used. Returns 1 on valid, 0 on invalid.

### `RSA_free(rsa: PRSA)`

Frees an RSA key structure.

---

## PEM Key Reading

### `PEM_read_bio_PrivateKey(bp, x, cb, u): EVP_PKEY`

Reads a PEM-encoded private key from a BIO. Supports RSA, EC, and other key types. `cb` is a password callback for encrypted keys; pass `nil` for unencrypted.

```nim
let bio = BIO_new_mem_buf(pemKey.cstring, pemKey.len.cint)
let pkey = PEM_read_bio_PrivateKey(bio, nil, nil, nil)
```

### `PEM_read_bio_RSA_PUBKEY / PEM_read_RSA_PUBKEY`

Reads an RSA public key from a BIO or FILE pointer, in the `PUBLIC KEY` PEM format (SubjectPublicKeyInfo).

### `PEM_read_bio_RSAPublicKey / PEM_read_RSAPublicKey`

Reads an RSA public key in the older `RSA PUBLIC KEY` PEM format (PKCS#1).

### `PEM_read_bio_RSAPrivateKey / PEM_read_RSAPrivateKey`

Reads an RSA private key in PEM format from a BIO or FILE pointer.

### `EVP_PKEY_free(p: EVP_PKEY)`

Frees an `EVP_PKEY` object and its underlying key material.

---

## EVP Signing (DigestSign)

These functions perform a combined hash-then-sign operation using an `EVP_MD_CTX`.

### `EVP_DigestSignInit(ctx, pctx, typ, e, pkey): cint`

Initialises the context for signing with the given private key and digest algorithm. `pctx` (optional) receives a pointer to the key context for setting padding etc.

```nim
let ctx = EVP_MD_CTX_create()
discard EVP_DigestSignInit(ctx, nil, EVP_sha256(), nil, pkey)
```

### `EVP_DigestUpdate(ctx, data, len)` *(shared with hashing)*

Feeds data into the sign-then-hash pipeline.

### `EVP_DigestSignFinal(ctx, data: pointer, len: ptr csize_t): cint`

Finalises the signature. Call with `data = nil` first to determine the required buffer size, then call again with an allocated buffer.

```nim
var sigLen: csize_t
discard EVP_DigestSignFinal(ctx, nil, addr sigLen)
var sig = newString(sigLen.int)
discard EVP_DigestSignFinal(ctx, sig.cstring, addr sigLen)
```

### `EVP_PKEY_CTX_new(pkey, e): EVP_PKEY_CTX`

Creates a key context for the given `EVP_PKEY`. Used for setting RSA padding mode.

### `EVP_PKEY_CTX_free(pkeyCtx: EVP_PKEY_CTX)`

Frees a key context.

### `EVP_PKEY_sign_init(c: EVP_PKEY_CTX): cint`

Initialises the key context for a signing operation.

### `EVP_MD_CTX_cleanup(ctx: EVP_MD_CTX): cint`

Cleans the state of an MD context without freeing it, so it can be reused.

---

## MD5 Functions

> **Warning**: MD5 is cryptographically broken. Use it only for non-security purposes such as checksums or cache keys.

### `md5_File(file: string): string`

High-level convenience function. Reads the file at `file` and returns its MD5 digest as a 32-character lowercase hex string (equivalent to `md5sum` output).

```nim
echo md5_File("photo.jpg")  # e.g. "d8e8fca2dc0f896fd7cb4cb0031ba249"
```

### `md5_Str(str: string): string`

Returns the MD5 digest of a string as a 32-character lowercase hex string.

```nim
echo md5_Str("hello")  # "5d41402abc4b2a76b9719d911017c592"
```

### `md5_Init(c: var MD5_CTX): cint`

Initialises an `MD5_CTX` for a new incremental hash computation.

### `md5_Update(c: var MD5_CTX, data: pointer, len: csize_t): cint`

Feeds a chunk of data into the running MD5 computation.

### `md5_Final(md: cstring, c: var MD5_CTX): cint`

Finalises the computation and writes the 16-byte binary digest into `md`.

### `md5(d: ptr uint8, n: csize_t, md: ptr uint8): ptr uint8`

One-shot MD5: hashes `n` bytes at `d` and writes the result to `md`. Returns `md`.

### `md5_Transform(c: var MD5_CTX, b: ptr uint8)`

Processes one 512-bit block directly. Low-level; rarely needed outside the library itself.

---

## Version & Memory Utilities

### `getOpenSSLVersion(): culong`

Returns the OpenSSL version packed as an unsigned long. See [Library Initialisation](#library-initialisation).

### `CRYPTO_malloc_init()`

Replaces OpenSSL's allocator with Nim's GC-aware allocator. See [Library Initialisation](#library-initialisation).

### `useOpenssl3*: bool`

Compile-time constant set to `true` when building against OpenSSL 3.x (via `-d:useOpenssl3` or when the version string starts with `'3'`).

### `DLLSSLName* / DLLUtilName*`

Compile-time constants with the platform-specific library names (e.g. `"libssl.so.3"`, `"libcrypto-1_1-x64.dll"`). Useful when you need to load additional symbols with `dynlib`.

---

## Quick-Start: TLS Client Example

```nim
import openssl

# 1. Initialise
discard SSL_library_init()
SSL_load_error_strings()
OpenSSL_add_all_algorithms()

# 2. Create context
let ctx = SSL_CTX_new(TLS_client_method())
discard SSL_CTX_load_verify_locations(ctx, "/etc/ssl/certs/ca-certificates.crt", nil)
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, nil)

# 3. Create SSL object and connect it to a socket fd
let ssl = SSL_new(ctx)
discard SSL_set_fd(ssl, mySocket.fd)
discard SSL_set_tlsext_host_name(ssl, "example.com")

# 4. Handshake
if SSL_connect(ssl) != 1:
  ERR_print_errors_fp(stderr)
  quit(1)

# 5. Verify
if SSL_get_verify_result(ssl) != X509_V_OK:
  echo "Certificate verification failed"
  quit(1)

# 6. Send/receive
discard SSL_write(ssl, "GET / HTTP/1.0\r\n\r\n", 18)
var buf = newString(4096)
let n = SSL_read(ssl, buf.cstring, buf.len.cint)
echo buf[0..<n]

# 7. Shutdown
discard SSL_shutdown(ssl)
SSL_free(ssl)
SSL_CTX_free(ctx)
```
