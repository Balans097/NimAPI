# std/sha1 — Module Reference

> **Part of Nim's standard library** (`std/sha1`).

---

> ⚠️ **DEPRECATED**
>
> This module is deprecated. The recommended replacement is the `checksums` package from Nimble:
>
> ```
> nimble install checksums
> ```
>
> Then import with:
>
> ```nim
> import checksums/sha1
> ```
>
> The API in `checksums/sha1` is identical to this module. All code written against `std/sha1` can be migrated by changing only the import line.

---

## Overview

`std/sha1` implements [SHA-1 (Secure Hash Algorithm 1)](https://en.wikipedia.org/wiki/SHA-1), a cryptographic hash function that takes an input of arbitrary length and produces a fixed-size 160-bit (20-byte) digest, typically represented as a 40-character uppercase hexadecimal string.

### Important security note

SHA-1 is **cryptographically broken** and must not be used for security-sensitive purposes such as digital signatures, certificate validation, or password hashing. Practical collision attacks against SHA-1 have been demonstrated. For security-critical hashing, use SHA-256 or SHA-3.

SHA-1 remains appropriate for non-security uses: checksumming files to detect accidental corruption, comparing data identity in build systems or caches, and generating deterministic identifiers where collision resistance under adversarial conditions is not required.

### Two usage styles

The module offers two layers of API:

**High-level (one-shot):** `secureHash` and `secureHashFile` do everything in a single call. Use these when you have the entire input available at once.

**Low-level (streaming):** `newSha1State` → `update` → `finalize`. Use this when data arrives in chunks — network streams, large files read in pieces, incremental construction of input.

---

## Types

### `Sha1Digest`

```nim
type Sha1Digest* = array[0 .. 19, uint8]
```

A raw 20-byte (160-bit) array holding the binary SHA-1 digest. This is the direct output of the hash computation. You will rarely work with this type directly — `SecureHash` wraps it and provides a more ergonomic interface.

---

### `SecureHash`

```nim
type SecureHash* = distinct Sha1Digest
```

A `distinct` wrapper over `Sha1Digest`. Being `distinct` means the compiler will not let you accidentally mix a `SecureHash` with a raw byte array — they must be compared with the provided `==` operator and converted to/from strings with `$` and `parseSecureHash`. This is the type returned by `secureHash`, `secureHashFile`, and `parseSecureHash`, and the type you will use in the majority of code.

---

### `Sha1State`

```nim
type Sha1State* = object
  count: int
  state: array[5, uint32]
  buf:   array[64, byte]
```

The mutable state object for streaming (incremental) hashing. It holds the running SHA-1 computation: a byte counter, the five 32-bit working variables, and a 64-byte block buffer. You create one with `newSha1State`, feed it data with `update`, and extract the final digest with `finalize`. After `finalize` is called the state must not be reused — create a fresh one for the next hash.

---

## Low-Level Streaming API

These three procedures form a pipeline. Use them when input arrives incrementally.

---

### `newSha1State`

```nim
proc newSha1State*(): Sha1State
```

**Creates and returns a freshly initialised `Sha1State`.**

The state is set to the five SHA-1 initial hash values mandated by the standard (`0x67452301`, `0xEFCDAB89`, `0x98BADCFE`, `0x10325476`, `0xC3D2E1F0`). The byte counter is reset to zero. This is the starting point of every streaming hash computation.

If you use the high-level `secureHash` or `secureHashFile`, you never need to call this directly — they call it internally.

**Example:**

```nim
import std/sha1

var state = newSha1State()
state.update("first chunk")
state.update(" second chunk")
let digest: Sha1Digest = state.finalize()
```

---

### `update`

```nim
proc update*(ctx: var Sha1State, data: openArray[char])
```

**Feeds a chunk of data into the running SHA-1 computation.**

You can call `update` any number of times with chunks of any size. The algorithm buffers data internally in 64-byte blocks; whenever a complete block accumulates it is processed immediately and the buffer is cleared. The final partial block is held until `finalize` is called.

The hash of a single string fed all at once is identical to the hash of the same string fed as many small pieces — the chunking is invisible to the result.

After `finalize` has been called, calling `update` again produces undefined results. Create a new `Sha1State` if you need to hash something else.

**Example — hashing a large file in chunks:**

```nim
import std/sha1

proc hashStream(f: File): SecureHash =
  var state = newSha1State()
  var buf = newString(8192)
  while true:
    let n = readChars(f, buf)
    if n == 0: break
    buf.setLen(n)
    state.update(buf)
    if n < 8192: break
  SecureHash(state.finalize())
```

**Example — feeding multiple pieces:**

```nim
import std/sha1

var state = newSha1State()
state.update("Hello")
state.update(", ")
state.update("World")
let h = SecureHash(state.finalize())
assert h == parseSecureHash("0A4D55A8D778E5022FAB701977C5D840BBC486D0")
# Same result as secureHash("Hello, World")
```

---

### `finalize`

```nim
proc finalize*(ctx: var Sha1State): Sha1Digest
```

**Completes the SHA-1 computation and returns the 20-byte digest.**

`finalize` performs the mandatory SHA-1 padding: it appends a `1` bit, then enough zero bytes to bring the total length to 56 bytes mod 64, and finally encodes the original message length as a 64-bit big-endian integer to fill the last block. This padding is required by the SHA-1 specification and ensures the digest correctly represents the complete input regardless of its length.

The returned value is a `Sha1Digest` (raw `array[20, uint8]`). Wrap it in `SecureHash(...)` to get the typed hash you can stringify and compare.

**Example:**

```nim
import std/sha1

var state = newSha1State()
state.update("Hello World")
let raw: Sha1Digest = state.finalize()
let hash = SecureHash(raw)
echo $hash   # "0A4D55A8D778E5022FAB701977C5D840BBC486D0"
```

---

## High-Level One-Shot API

---

### `secureHash`

```nim
proc secureHash*(str: openArray[char]): SecureHash
```

**Hashes a string (or any `openArray[char]`) in a single call and returns its `SecureHash`.**

This is the most convenient entry point for hashing data that is already fully in memory. Internally it creates a `Sha1State`, calls `update` once with the entire input, and returns `SecureHash(state.finalize())`. There is no reason to use the streaming API if your data is already a string or array.

**Example:**

```nim
import std/sha1

let hash = secureHash("Hello World")
assert $hash == "0A4D55A8D778E5022FAB701977C5D840BBC486D0"

let nameHash = secureHash("John Doe")
assert $nameHash == "AE6E4D1209F17B460503904FAD297B31E9CF6362"
```

**Example — using as a cache key:**

```nim
import std/sha1, std/tables

var cache: Table[string, string]

proc cachedProcess(input: string): string =
  let key = $secureHash(input)
  if key in cache:
    return cache[key]
  result = expensiveOperation(input)
  cache[key] = result
```

---

### `secureHashFile`

```nim
proc secureHashFile*(filename: string): SecureHash
```

**Hashes the entire contents of a file and returns its `SecureHash`.**

The file is read in 8192-byte chunks using the streaming API internally, so arbitrarily large files are handled without loading them fully into memory. The file is opened and closed automatically. If the file cannot be opened, a standard Nim I/O exception is raised.

**Example — basic file checksum:**

```nim
import std/sha1

let hash = secureHashFile("myproject.nim")
echo $hash
```

**Example — verifying a file against a known hash:**

```nim
import std/sha1

let
  computed = secureHashFile("downloaded.zip")
  expected = parseSecureHash("10DFAEBF6BFDBC7939957068E2EFACEC4972933C")

if computed == expected:
  echo "File integrity OK."
else:
  echo "MISMATCH — file may be corrupted or tampered."
```

---

## Conversion and Comparison

---

### `$`

```nim
proc `$`*(self: SecureHash): string
```

**Converts a `SecureHash` to its canonical 40-character uppercase hexadecimal string.**

Each of the 20 bytes is rendered as two hex digits, always uppercase, always with a leading zero if the byte value is less than 16. The result is always exactly 40 characters long.

**Example:**

```nim
import std/sha1

let hash = secureHash("Hello World")
let s = $hash
assert s == "0A4D55A8D778E5022FAB701977C5D840BBC486D0"
assert s.len == 40
```

---

### `parseSecureHash`

```nim
proc parseSecureHash*(hash: string): SecureHash
```

**Parses a 40-character hexadecimal string into a `SecureHash`.**

This is the inverse of `$`. Use it to reconstruct a `SecureHash` from a hash string that was previously stored in a file, database, configuration, or received over a network. Both uppercase and lowercase hex digits are accepted by `parseHexInt` internally.

The input string must be exactly 40 characters long and contain only valid hexadecimal digits. No validation is performed — passing a malformed string will produce a garbage `SecureHash` or a runtime error.

**Example:**

```nim
import std/sha1

let stored = "0A4D55A8D778E5022FAB701977C5D840BBC486D0"
let hash = parseSecureHash(stored)
assert hash == secureHash("Hello World")
```

---

### `==`

```nim
proc `==`*(a, b: SecureHash): bool
```

**Returns `true` if two `SecureHash` values are identical — i.e., they represent the same digest.**

Two hashes are equal if and only if every one of their 20 bytes matches. In practice this means the inputs that were hashed produced the same 160-bit output.

> **Note:** This comparison is **not** constant-time. It is not safe to use for secret verification (e.g., comparing a password hash against a stored hash in an authentication flow). For that use case, use a dedicated constant-time comparison function.

**Example:**

```nim
import std/sha1

let a = secureHash("Hello World")
let b = secureHash("Goodbye World")
let c = parseSecureHash("0A4D55A8D778E5022FAB701977C5D840BBC486D0")

assert a != b   # different inputs → different hashes
assert a == c   # same digest from two different sources
```

---

### `isValidSha1Hash`

```nim
proc isValidSha1Hash*(s: string): bool
```

**Returns `true` if `s` looks like a valid SHA-1 hex digest string.**

Validates two properties: the string must be exactly 40 characters long, and every character must be a hexadecimal digit (`0`–`9`, `a`–`f`, `A`–`F`). This is a format check only — it does not verify that the string corresponds to any particular input.

Use this before calling `parseSecureHash` to guard against malformed input and avoid runtime errors.

**Example:**

```nim
import std/sha1

assert isValidSha1Hash("0A4D55A8D778E5022FAB701977C5D840BBC486D0") == true
assert isValidSha1Hash("0a4d55a8d778e5022fab701977c5d840bbc486d0") == true  # lowercase OK
assert isValidSha1Hash("ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ") == false  # non-hex
assert isValidSha1Hash("0A4D55")  == false  # too short
assert isValidSha1Hash("")        == false  # empty
```

**Example — safe parsing with validation:**

```nim
import std/sha1

proc tryParseHash(s: string): SecureHash =
  if not isValidSha1Hash(s):
    raise newException(ValueError, "Not a valid SHA-1 hash: " & s)
  parseSecureHash(s)
```

---

## Typical Workflows

### Workflow 1 — Hash a string and compare to a known value

```nim
import std/sha1

let hash = secureHash("my data")
echo $hash
```

### Workflow 2 — Verify a downloaded file

```nim
import std/sha1

let expected = parseSecureHash("DEADBEEFDEADBEEFDEADBEEFDEADBEEFDEADBEEF")
let actual   = secureHashFile("file.bin")

assert actual == expected, "Integrity check failed!"
```

### Workflow 3 — Incremental hashing of a stream

```nim
import std/sha1

var state = newSha1State()
state.update(headerBytes)
state.update(bodyBytes)
state.update(trailerBytes)
let hash = SecureHash(state.finalize())
echo $hash
```

### Workflow 4 — Safe round-trip through a string representation

```nim
import std/sha1

let original = secureHash("some input")
let asString = $original                   # serialize
let restored = parseSecureHash(asString)   # deserialize
assert original == restored
```

---

## Summary Table

| Symbol | Signature | Purpose |
|--------|-----------|---------|
| `Sha1Digest` | `array[20, uint8]` | Raw binary 160-bit digest |
| `SecureHash` | `distinct Sha1Digest` | Typed hash value for comparison and display |
| `Sha1State` | `object` | Mutable state for incremental hashing |
| `newSha1State()` | `() → Sha1State` | Initialise a new streaming hash computation |
| `update(ctx, data)` | `(var Sha1State, openArray[char]) → void` | Feed a data chunk into the computation |
| `finalize(ctx)` | `(var Sha1State) → Sha1Digest` | Complete the computation and extract the digest |
| `secureHash(str)` | `openArray[char] → SecureHash` | One-shot: hash a string |
| `secureHashFile(name)` | `string → SecureHash` | One-shot: hash a file by path |
| `$hash` | `SecureHash → string` | Format as 40-char uppercase hex |
| `parseSecureHash(s)` | `string → SecureHash` | Parse a 40-char hex string |
| `a == b` | `(SecureHash, SecureHash) → bool` | Compare two digests |
| `isValidSha1Hash(s)` | `string → bool` | Validate format before parsing |

---

> **See also:** [`checksums/sha1`](https://github.com/nim-lang/checksums) — the recommended replacement for this deprecated module; [`std/md5`](https://nim-lang.org/docs/md5.html) — MD5 checksum algorithm; [`std/hashes`](https://nim-lang.org/docs/hashes.html) — fast non-cryptographic hashing of Nim types; [`std/base64`](https://nim-lang.org/docs/base64.html) — Base64 encoding and decoding.
