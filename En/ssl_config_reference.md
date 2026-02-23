# ssl_config — Module Reference

## Overview

The `ssl_config` module exposes three compile-time string constants containing **OpenSSL-compatible TLS cipher suite lists**, sourced directly from [Mozilla's Server Side TLS guidelines](https://wiki.mozilla.org/Security/Server_Side_TLS) (configuration file version 5.4, generated 2020-06-03).

The module's only purpose is to give Nim programs a convenient, authoritative starting point for configuring TLS without having to copy-paste or maintain raw cipher strings manually. All three constants can be passed directly wherever an OpenSSL cipher list string is expected — for example, to `newContext` in Nim's `net` or `asyncnet` modules.

### The three compatibility tiers

Mozilla defines three named compatibility profiles that reflect a deliberate trade-off between security and the breadth of clients you need to support:

| Constant | Profile | Oldest supported clients |
|---|---|---|
| `CiphersModern` | **Modern** — maximum security, narrow client support | Firefox 63, Chrome 70, Android 10, Java 11 |
| `CiphersIntermediate` | **Intermediate** — strong security, broad modern client support | Firefox 27, Chrome 31, Android 4.4.2, Java 8u31 |
| `CiphersOld` | **Old** — widest legacy support at acceptable risk | Firefox 1, IE 8 on XP, Android 2.3, Java 6 |

The profiles are cumulative: `CiphersOld` is a superset of `CiphersIntermediate`, which is itself a superset of `CiphersModern`. Each step down the list adds ciphers that support older clients but carry a higher security cost.

---

## Exported Constants

---

### `CiphersModern`

```nim
const CiphersModern* = "TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256"
```

The strictest profile. Contains **only TLS 1.3 cipher suites**, which means it implicitly requires TLS 1.3 — no TLS 1.0, 1.1, or 1.2 connections will be negotiated using these ciphers alone.

**What the ciphers mean:**

- `TLS_AES_128_GCM_SHA256` — AES-128 in Galois/Counter Mode with SHA-256 for authentication. Fast on hardware with AES-NI.
- `TLS_AES_256_GCM_SHA384` — AES-256 in GCM with SHA-384. Higher key strength at modest extra cost.
- `TLS_CHACHA20_POLY1305_SHA256` — ChaCha20 stream cipher with Poly1305 MAC. Excellent performance on devices without AES hardware acceleration (mobile, embedded).

**Oldest clients supported:**

Firefox 63, Android 10.0, Chrome 70, Edge 75, Java 11, OpenSSL 1.1.1, Opera 57, Safari 12.1.

**When to use:** public-facing services where you control or know your audience — APIs consumed by modern apps, internal microservices, developer tooling. Not suitable if you need to support any client predating late 2018.

**Example:**

```nim
import net, ssl_config

let ctx = newContext(cipherList = CiphersModern)
```

---

### `CiphersIntermediate`

```nim
const CiphersIntermediate* = "TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:" &
  "TLS_CHACHA20_POLY1305_SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:" &
  "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:" &
  "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:" &
  "ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:" &
  "DHE-RSA-AES256-GCM-SHA384"
```

*(Shown split for readability; the actual constant is a single colon-separated string.)*

The **recommended default** for most servers. Adds TLS 1.2 ECDHE and DHE cipher suites on top of the TLS 1.3 suites from `CiphersModern`, enabling negotiation with clients that do not yet support TLS 1.3 while still excluding all weak, export, and static-RSA ciphers.

**Key cipher families added over Modern:**

- `ECDHE-{ECDSA,RSA}-AES{128,256}-GCM-SHA{256,384}` — Elliptic-curve Diffie-Hellman ephemeral key exchange with AES-GCM. Provides **forward secrecy**: past sessions cannot be decrypted even if the server's long-term key is later compromised.
- `ECDHE-{ECDSA,RSA}-CHACHA20-POLY1305` — ECDHE key exchange with ChaCha20-Poly1305. Good on mobile clients.
- `DHE-RSA-AES{128,256}-GCM-SHA{256,384}` — Classic Diffie-Hellman (non-EC) ephemeral key exchange with AES-GCM. Covers clients with ECDHE issues.

**Oldest clients supported:**

Firefox 27, Android 4.4.2, Chrome 31, Edge (all versions), IE 11 on Windows 7, Java 8u31, OpenSSL 1.0.1, Opera 20, Safari 9.

**When to use:** the right choice for the vast majority of public web servers, REST APIs, and network services. Mozilla themselves describe this as *"the recommended configuration for a general-purpose server."*

**Example:**

```nim
import net, ssl_config

let ctx = newContext(cipherList = CiphersIntermediate)
```

---

### `CiphersOld`

```nim
const CiphersOld* = "TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:" &
  "TLS_CHACHA20_POLY1305_SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:..." &
  "...:DES-CBC3-SHA"
```

*(Shown truncated; the actual constant contains 28 colon-separated cipher entries.)*

The most permissive profile. Adds a wide range of older TLS 1.2 and even TLS 1.0/1.1 cipher suites, including ciphers without forward secrecy (`AES128-SHA`, `AES256-SHA`, `DES-CBC3-SHA`) and suites using SHA-1 for authentication. This is the only profile that allows connections from very old clients such as IE 8 on Windows XP or Android 2.3.

**Additional cipher families over Intermediate:**

- `DHE-RSA-CHACHA20-POLY1305` — DHE (non-EC) with ChaCha20-Poly1305.
- `ECDHE-{ECDSA,RSA}-AES{128,256}-SHA{256,384}` — ECDHE with SHA-1/SHA-2 HMAC instead of GCM (TLS 1.2 CBC mode).
- `ECDHE-{ECDSA,RSA}-AES{128,256}-SHA` — ECDHE with SHA-1 authentication.
- `DHE-RSA-AES{128,256}-SHA256` — DHE with SHA-2 HMAC.
- `AES{128,256}-{GCM-SHA256,SHA256,SHA}` — Static RSA key exchange (no forward secrecy).
- `DES-CBC3-SHA` — Triple-DES with CBC and SHA-1. Very old, included only for IE 8 on Windows XP.

**Oldest clients supported:**

Firefox 1, Android 2.3, Chrome 1, Edge 12, IE 8 on Windows XP, Java 6, OpenSSL 0.9.8, Opera 5, Safari 1.

**When to use:** only when you have a concrete business requirement to serve legacy clients that cannot be updated. Mozilla cautions that this profile has *"known weaknesses"* and should be used only when there is no alternative. The presence of `DES-CBC3-SHA` and static RSA ciphers is a deliberate concession to compatibility, not a security recommendation.

**Example:**

```nim
import net, ssl_config

# Serving an intranet application that must support very old embedded clients:
let ctx = newContext(cipherList = CiphersOld)
```

---

## Choosing the Right Profile

The decision tree is straightforward:

1. **Do all your clients support TLS 1.3?** → Use `CiphersModern`.
2. **Do your clients include anything from the last ~10 years?** → Use `CiphersIntermediate`. This is the correct default for almost every public service.
3. **Do you have a hard requirement to support clients from before 2013 (IE 8, Android 2.x, Java 6)?** → Use `CiphersOld`, and document why.

When in doubt, **`CiphersIntermediate` is always the safest starting point.** It covers modern clients fully, provides forward secrecy for all connections, and excludes the genuinely weak ciphers that `CiphersOld` adds.

---

## Practical Example — HTTPS server with selectable profile

```nim
import asynchttpserver, asyncnet, net, ssl_config

proc makeServerContext(profile: string): SslContext =
  let ciphers = case profile
    of "modern":       CiphersModern
    of "old":          CiphersOld
    else:              CiphersIntermediate   # safe default
  newContext(
    certFile   = "server.crt",
    keyFile    = "server.key",
    cipherList = ciphers
  )

let ctx = makeServerContext("intermediate")
```

---

## Important Notes

- **These constants are frozen at the 2020-06-03 snapshot** of Mozilla's guidelines (version 5.4). Mozilla updates its recommendations periodically; for a production system with high security requirements, check [ssl-config.mozilla.org](https://ssl-config.mozilla.org) for the current version and update your cipher list accordingly.
- The strings are **OpenSSL cipher list format**. They may not work verbatim with other TLS libraries (e.g. mbedTLS, wolfSSL) without translation.
- All three profiles **include the TLS 1.3 suites** (`TLS_AES_*` and `TLS_CHACHA20_*`). In OpenSSL 1.1.1+, TLS 1.3 cipher suites are controlled separately from TLS 1.2 ones, so having them in the string is harmless and ensures TLS 1.3 is always enabled when available.
- The module contains **no procedures** — it is a pure constant-definition module with zero runtime cost.
