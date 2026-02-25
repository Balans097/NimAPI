# `sysrand` Module Reference

> **Module:** `std/sysrand`  
> **Language:** Nim  
> **Purpose:** Cryptographically secure random bytes sourced directly from the operating system — suitable for key generation, nonces, tokens, salts, and other security-sensitive purposes.

> ⚠️ **Security notice:** This module has not been audited by cryptography professionals. It is provided as-is. For production systems handling sensitive data, request an independent security audit before relying on this module.

---

## Core Idea: Cryptographic vs. Statistical Randomness

Most general-purpose random number generators — including Nim's `std/random` — are *pseudorandom*: they produce deterministic sequences from a seed. This is perfectly fine for simulations, games, shuffling, or any use that only needs statistical unpredictability. But for security purposes (passwords, session tokens, encryption keys, salt values), determinism is a vulnerability: if an attacker can determine or influence the seed, they can predict all future output.

`std/sysrand` solves this by delegating to the operating system's own entropy source. The OS accumulates true randomness from hardware events — keyboard timing, disk interrupts, network packet timing, hardware noise — and provides it through a system call. The result is non-deterministic and, critically, **not seeded by anything a program controls**, making it suitable for cryptographic use.

---

## Platform Implementations

The module automatically selects the best available OS primitive at compile time. You do not choose — the right one is used for your target:

| Platform | Underlying mechanism |
|---|---|
| **Windows** | `BCryptGenRandom` — Windows cryptographic API, uses the system RNG |
| **Linux** | `getrandom(2)` syscall — blocks until entropy pool is initialized |
| **macOS / iOS** | `SecRandomCopyBytes` — Apple Security framework |
| **OpenBSD** | `getentropy(2)` — high-quality, max 256 bytes per call |
| **FreeBSD** | `getrandom(2)` — similar to Linux, max 256 bytes per call |
| **Zephyr RTOS** | `sys_csrand_get` — hardware RNG for embedded targets |
| **Browser (JS)** | `window.crypto.getRandomValues` — Web Crypto API, max 65536 bytes per call |
| **Node.js** | `crypto.randomFillSync` — Node.js crypto module |
| **Other POSIX** | `/dev/urandom` — read from the kernel's random device |

### Linux: opting out of `getrandom`

On Linux, if you are targeting kernels older than 3.17 (which do not have the `getrandom` syscall), compile with:

```
nim c -d:nimNoGetRandom yourprogram.nim
```

This falls back to reading from `/dev/urandom`, which works on all POSIX-compatible systems.

### Why `/dev/urandom` and not `/dev/random`?

The POSIX fallback uses `/dev/urandom` rather than `/dev/random`. On modern Linux kernels there is practically no security difference between the two once the entropy pool is initialized — `/dev/random` would block indefinitely waiting for more hardware entropy, while `/dev/urandom` does not. The fallback also verifies that the opened file is a character device (`S_ISCHR`) before reading, to guard against an attacker replacing it with a regular file.

---

## Exported Procedures

There are exactly two exported procedures, both named `urandom`. They differ only in how you provide the output buffer.

---

### `urandom` — fill an existing buffer

```nim
proc urandom*(dest: var openArray[byte]): bool
```

**What it does:** Fills every byte of the `dest` buffer with cryptographically secure random bytes sourced from the OS. Returns `true` if the operation succeeded, `false` if the OS call failed.

This is the in-place variant. You supply a pre-allocated buffer — any `array`, `seq`, or `openArray` of `byte` — and the procedure fills it. No new allocation occurs inside the call.

**Empty buffer:** If `dest` has length 0, `urandom` returns `true` immediately without making any OS call.

**Error handling:** The boolean return value must be checked. On failure, `dest` may be partially filled — do not use any of its contents if `false` is returned. The failure cases are rare but possible: insufficient OS entropy (early in boot), OS resource exhaustion, or a system call interrupted in an unrecoverable way.

On JavaScript targets, errors are silently ignored (JS crypto APIs do not provide an error indication in the same way) and `true` is always returned.

```nim
import std/sysrand

# Fill a fixed-size array (stack-allocated, no heap):
var key: array[32, byte]
if not urandom(key):
  raise newException(IOError, "Failed to generate random key")
echo "Key bytes: ", key

# Fill a seq:
var token = newSeq[byte](16)
if not urandom(token):
  raise newException(IOError, "Failed to generate token")

# Zero-length — always succeeds:
var empty: array[0, byte]
assert urandom(empty) == true
```

**When to prefer this overload:** When you already have a buffer that needs filling, when you want to avoid an extra allocation, when writing into an existing data structure's fields, or when integrating with a library that provides its own buffer.

---

### `urandom` — allocate and return a new sequence

```nim
proc urandom*(size: Natural): seq[byte]
```

**What it does:** Allocates a new `seq[byte]` of length `size`, fills it entirely with cryptographically secure random bytes from the OS, and returns it.

Unlike the in-place overload, this version **raises `OSError`** on failure rather than returning a boolean. If the returned `seq` is non-empty, every byte in it is trustworthy random data.

**Zero size:** Requesting `urandom(0)` returns an empty `seq[byte]` immediately without any OS call.

**Error handling:** Because errors are signaled via exception, you do not need to check a return value. Simply catch `OSError` if you need to handle failures gracefully. In most applications, an OS random source failure is unrecoverable and crashing is the right response.

```nim
import std/sysrand

# Generate a 32-byte (256-bit) encryption key:
let key = urandom(32)
assert key.len == 32

# Generate a 128-bit session token:
let token = urandom(16)

# Generate a UUID-like nonce (16 bytes, then format as needed):
let nonce = urandom(16)

# Zero length:
assert urandom(0).len == 0

# Two calls will almost certainly differ:
assert urandom(32) != urandom(32)
```

**When to prefer this overload:** When you want the simplest possible API, when you need the bytes in a fresh `seq` anyway, and when exceptions are your preferred error-handling style.

---

## Security Properties and Guarantees

### What `urandom` guarantees

- **Non-determinism:** The bytes cannot be predicted by observing previous output, even if the attacker knows the algorithm.
- **Uniform distribution:** Each byte value 0–255 is equally likely.
- **Independent calls:** Successive calls produce statistically independent outputs.
- **OS entropy pool:** The source is seeded by the OS from hardware events and is not influenced by anything your program does.

### What `urandom` does NOT guarantee

- **Audit:** The Nim wrapper code has not been reviewed by cryptographers. The underlying OS primitives have been, but the glue code has not.
- **Quantum resistance:** Like all classical cryptographic sources, this provides bits that a sufficiently powerful quantum computer could potentially exploit in certain algorithms. The raw bytes themselves are fine; what you build with them may or may not be quantum-resistant.
- **Forward secrecy of state:** If the OS entropy pool state is compromised (e.g. on a virtual machine with a cloned snapshot), past and future outputs could be correlated.

---

## Practical Usage Patterns

### Generating a cryptographic key

```nim
import std/sysrand

proc generateAesKey(): array[32, byte] =
  ## Generate a 256-bit AES key.
  if not urandom(result):
    raise newException(IOError, "OS random source unavailable")
```

### Generating a URL-safe token

```nim
import std/sysrand, std/base64

proc generateToken(byteLen = 24): string =
  ## Generates a base64url-encoded random token.
  ## 24 random bytes → 32 characters of base64.
  let bytes = urandom(byteLen)
  result = encode(bytes, safe = true)

echo generateToken()  # e.g. "aB3kZ9mQpR2xYvLwNsT4cU7d"
```

### Generating a random salt for password hashing

```nim
import std/sysrand

proc generateSalt(): array[16, byte] =
  ## 128-bit salt for use with a password hashing function.
  if not urandom(result):
    raise newException(IOError, "Cannot generate salt")
```

### Filling part of a larger structure

```nim
import std/sysrand

type Packet = object
  nonce: array[12, byte]   # 96-bit nonce for AES-GCM
  payload: seq[byte]

proc initPacket(data: seq[byte]): Packet =
  result.payload = data
  if not urandom(result.nonce):
    raise newException(IOError, "Nonce generation failed")
```

---

## Comparison: `sysrand` vs `random`

| Property | `std/sysrand` | `std/random` |
|---|---|---|
| Source | OS hardware entropy | Algorithmic PRNG (xoshiro256) |
| Seedable | No | Yes |
| Deterministic | No | Yes (given same seed) |
| Suitable for crypto | Yes (with audit caveat) | **No** |
| Suitable for simulations | Overkill | Yes |
| Speed | Slower (system call) | Fast |
| API | Bytes only | Rich (integers, floats, shuffle…) |
| Blocking risk | Possible on boot (Linux) | Never |

Use `std/sysrand` whenever the randomness will be used for security. Use `std/random` for everything else.

---

## Error Handling Summary

| Overload | On success | On failure |
|---|---|---|
| `urandom(dest: var openArray[byte])` | Returns `true` | Returns `false` |
| `urandom(size: Natural): seq[byte]` | Returns filled `seq` | Raises `OSError` |

Always check the return value of the first overload. Never use partial buffer contents on failure. The second overload's exception-based design means failure is impossible to silently ignore.
