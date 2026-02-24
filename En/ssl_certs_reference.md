# `ssl_certs` Module Reference

> **Module:** `std/ssl_certs`  
> **Purpose:** Locate SSL/TLS Certificate Authority (CA) certificate files and directories on the host system, so that OpenSSL-based connections can verify server identities.

---

## Overview

When a Nim program opens an HTTPS connection (or any TLS connection), the underlying OpenSSL library must verify that the server's certificate is signed by a trusted Certificate Authority. To do that, it needs a file or directory containing trusted CA root certificates — a *trust store*.

The problem is that the trust store lives in different places on every operating system and Linux distribution. `ssl_certs` solves this by encoding the canonical search paths for every supported platform and providing a single iterator, `scanSSLCertificates`, that walks those paths and yields the ones that actually exist on the current machine.

This module is used internally by Nim's `httpclient` and `net` modules to automatically wire up CA certificates. You can also use it directly if you are building custom TLS connections.

### What the module finds

The iterator yields two kinds of paths:

- **Single bundle files** — a `.pem` or `.crt` file that contains all trusted CA certificates concatenated together (the most common form on Linux and macOS).
- **Certificate directories** — a directory where each certificate is stored as a separate file, named by its hash (the OpenSSL hash directory format used on some Linux distributions).

### Environment variable overrides

When called with `useEnvVars = true`, the iterator respects two standard environment variables before falling back to compiled-in paths:

| Variable | Effect |
|---|---|
| `SSL_CERT_FILE` | Use this single file as the sole certificate source. The iterator yields it and stops. |
| `SSL_CERT_DIR` | Scan all files inside this directory. The iterator yields each file found. |

These variables are the standard mechanism for pointing OpenSSL-based programs at a non-default trust store — for example, inside a container, a virtual environment, or a corporate network with a custom CA.

---

## Platform-Specific Certificate Search Paths

The module compiles in a different list of candidate paths depending on the target OS. Each path is probed at runtime; non-existent paths are silently skipped. Paths are checked in the order listed.

### macOS

| Path | Source |
|---|---|
| `/etc/ssl/cert.pem` | System-maintained bundle |
| `/System/Library/OpenSSL/certs/cert.pem` | Legacy OpenSSL location |

### Linux

| Path | Distributions / notes |
|---|---|
| `/etc/ssl/certs/ca-certificates.crt` | Debian, Ubuntu, Arch, SUSE, Gentoo, NetBSD |
| `/etc/ssl/ca-bundle.pem` | OpenSUSE |
| `/etc/pki/tls/certs/ca-bundle.crt` | Red Hat 5+, Fedora, CentOS |
| `/usr/share/ssl/certs/ca-bundle.crt` | Red Hat 4 |
| `/etc/pki/tls/certs` | Fedora/RHEL (directory) |
| `/data/data/com.termux/files/usr/etc/tls/cert.pem` | Termux (Android) |
| `/system/etc/security/cacerts` | Android (directory) |

### BSD (FreeBSD, OpenBSD, NetBSD)

| Path | Source |
|---|---|
| `/etc/ssl/certs/ca-certificates.crt` | Debian-derived BSDs, NetBSD |
| `/usr/local/share/certs/ca-root-nss.crt` | FreeBSD (`security/ca-root-nss` package) |
| `/etc/ssl/cert.pem` | OpenBSD, FreeBSD symlink |
| `/usr/local/share/certs` | FreeBSD (directory) |
| `/etc/openssl/certs` | NetBSD (directory) |

### Windows

On Windows, the module does **not** use the Windows Certificate Store. Instead it looks for a file named `cacert.pem`:

1. First in the directory of the running executable (`getAppDir()`).
2. Then in every directory listed in the `PATH` environment variable (stripping surrounding quotes from quoted entries).

The `cacert.pem` file is typically the [Mozilla CA bundle](https://curl.se/docs/caextract.html) distributed alongside many Windows OpenSSL builds (e.g. via curl, Git for Windows, or manually downloaded). If none is found, no certificates are yielded.

### Haiku

Uses the native `find_paths_etc` Haiku API (from `<FindDirectory.h>`) to locate `ssl/CARootCertificates.pem` in the system data directory. Memory returned by the API is freed automatically via `defer`.

---

## Exported Symbols

---

### `scanSSLCertificates`

```nim
iterator scanSSLCertificates*(useEnvVars = false): string
```

#### What it does

An iterator that yields the paths of SSL/TLS CA certificate files (and, for hash-directory-format stores, also the directory path itself followed by the individual file paths) that exist on the current system. The caller receives one path per `yield` and can pass it to OpenSSL or any TLS API that accepts a certificate bundle path.

The iterator does **not** validate that the yielded files are syntactically correct PEM/DER data — it only checks file/directory existence. Feeding a corrupt certificate file to OpenSSL will produce an error from OpenSSL itself.

#### Decision tree

The iterator follows this logic in order, stopping at the first branch that applies:

```
useEnvVars = true AND SSL_CERT_FILE is set?
  → yield SSL_CERT_FILE, stop.

useEnvVars = true AND SSL_CERT_DIR is set?
  → yield every file inside SSL_CERT_DIR/*, stop.

Windows (no env vars matched)?
  → look for cacert.pem in app dir, then in PATH dirs.

Haiku?
  → call find_paths_etc, yield results.

Everything else (Linux, macOS, BSD)?
  → for each path in the compiled-in certificatePaths list:
      if it ends with .pem or .crt and the file exists → yield it.
      if it is a directory:
        if it contains any *.0 files → yield the directory path itself.
        yield every file in the directory.
```

#### Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `useEnvVars` | `bool` | `false` | When `true`, checks `SSL_CERT_FILE` and `SSL_CERT_DIR` environment variables before falling back to compiled-in paths. When `false`, environment variables are ignored entirely. |

#### Yield value

A `string` — a file system path to a certificate file or (for hash-directory stores) a directory path followed by individual certificate file paths within that directory.

#### Examples

```nim
import std/ssl_certs

# Print all CA certificate sources found on this system
for path in scanSSLCertificates():
  echo path
# Example output on Ubuntu:
# /etc/ssl/certs/ca-certificates.crt
```

```nim
import std/ssl_certs

# Respect environment variable overrides — useful in containers or CI
for path in scanSSLCertificates(useEnvVars = true):
  echo path
# If SSL_CERT_FILE=/etc/my-custom-ca.pem is set:
# /etc/my-custom-ca.pem
```

```nim
import std/ssl_certs

# Find the first available certificate bundle and stop
var firstCert = ""
for path in scanSSLCertificates(useEnvVars = true):
  firstCert = path
  break

if firstCert.len > 0:
  echo "Using CA bundle: ", firstCert
else:
  echo "Warning: no CA certificates found on this system!"
```

```nim
import std/ssl_certs, std/net

# Wiring up a TLS context manually using the first available CA bundle
# (this is roughly what httpclient does internally)
let ctx = newContext(caFile = block:
  var f = ""
  for path in scanSSLCertificates(useEnvVars = true):
    f = path
    break
  f
)
```

```nim
import std/ssl_certs

# Collect all found certificate paths into a sequence
var certs: seq[string]
for path in scanSSLCertificates():
  certs.add(path)

echo "Found ", certs.len, " certificate source(s)"
for c in certs:
  echo "  ", c
```

```nim
import std/ssl_certs, std/os

# Environment variable override pattern — useful for testing with a local CA
putEnv("SSL_CERT_FILE", "/tmp/test-ca.pem")
for path in scanSSLCertificates(useEnvVars = true):
  echo path   # → /tmp/test-ca.pem
```

---

## Common Use Cases and Patterns

### Using with `httpclient`

Nim's `httpclient` calls `scanSSLCertificates` automatically when creating a default SSL context. You normally do not need to call the iterator yourself unless you are creating a custom `SslContext`:

```nim
import std/[httpclient, net, ssl_certs]

# Manual context setup — locate certs, then configure
let sslCtx = newContext()
for certPath in scanSSLCertificates(useEnvVars = true):
  sslCtx.wrapSocket(...)  # platform-specific wiring
  break
```

### Diagnosing TLS certificate errors

If your program fails with a TLS verification error and you suspect missing CA certificates, iterate `scanSSLCertificates` to see what the library finds:

```nim
import std/ssl_certs

echo "Certificate sources visible to Nim on this system:"
var count = 0
for p in scanSSLCertificates(useEnvVars = true):
  echo "  [", count, "] ", p
  inc count
if count == 0:
  echo "  (none found — install a CA bundle or set SSL_CERT_FILE)"
```

### Containers and minimal environments

Docker images based on `alpine`, `scratch`, or other minimal bases often lack CA certificates. The recommended fix is to install the `ca-certificates` package in your `Dockerfile`. Alternatively, mount the host's bundle and point to it:

```sh
SSL_CERT_FILE=/host-certs/ca-bundle.pem ./your_nim_binary
```

---

## Design Notes and Pitfalls

**1. `useEnvVars = false` by default**  
Environment variables are intentionally ignored unless the caller opts in. This prevents user environment from silently affecting programs that do not expect it. Always pass `useEnvVars = true` in long-running services and tools where the operator should be able to override the trust store.

**2. The iterator may yield zero paths**  
On a freshly installed minimal system (or inside a container without CA packages), none of the compiled-in paths may exist. The iterator will silently yield nothing. Always check whether you received at least one path before proceeding with TLS operations.

**3. On Windows, no system certificate store is used**  
The module does not integrate with the Windows Certificate Store (`CertOpenSystemStore` / `CertEnumCertificatesInStore`). This commented-out code exists in the source but is not compiled. You must provide a `cacert.pem` file alongside the executable or somewhere on `PATH`, or set `SSL_CERT_FILE`. This is a known limitation.

**4. Directory entries: both the directory path and file paths may be yielded**  
For hash-directory-format stores (e.g. `/etc/pki/tls/certs` on Fedora/RHEL), the iterator first yields the directory path itself (if any `*.0` files are present), then yields each individual file inside the directory. Downstream consumers should handle both a bare directory string and individual file strings gracefully.

**5. File existence is checked; content is not**  
The iterator checks `fileExists` and `dirExists` but does not open, parse, or validate the certificate data. A zero-byte file or a corrupt PEM will be yielded and then cause an error only when OpenSSL tries to load it.

**6. `SSL_CERT_DIR` and `SSL_CERT_FILE` are mutually exclusive**  
When `useEnvVars = true`, `SSL_CERT_FILE` takes absolute priority. If it is set, `SSL_CERT_DIR` is never consulted, even if it is also set. If you need both a directory and a file, concatenate them into a single bundle file and point `SSL_CERT_FILE` at it.
