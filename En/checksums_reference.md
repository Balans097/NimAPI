# Nim Checksums & Password Hashing — Module Reference

> **Package:** `checksums` (install via `nimble install checksums`)
> **Umbrella module:** `docutils` — imports `md5`, `sha1`, `sha2`, `sha3`, and `bcrypt` in one shot.

This reference covers every exported symbol from all five modules. Sections are ordered from the most basic (MD5) to the most security-hardened (bcrypt) to give a natural reading order. Within each section, functions appear in the order you would typically call them.

---

## Table of Contents

1. [MD5](#md5)
2. [SHA-1](#sha-1)
3. [SHA-2](#sha-2)
4. [SHA-3 and SHAKE](#sha-3-and-shake)
5. [bcrypt](#bcrypt)
6. [Choosing the Right Algorithm](#choosing-the-right-algorithm)

---

## MD5

> **Module:** `md5`

MD5 produces a 128-bit (16-byte) digest. It is fast and widely used for **non-security** purposes such as checksumming files for accidental corruption, cache keys, and legacy interoperability. **Do not use MD5 where security matters** — collisions can be manufactured in seconds on consumer hardware.

The module works at compile time and in JavaScript targets, which is unusual and can be handy for metaprogramming.

---

### Type: `MD5Digest`

```nim
type MD5Digest* = array[0..15, uint8]
```

A 16-element byte array holding the raw MD5 output. You can convert it to a lowercase hex string with the `$` operator.

---

### Type: `MD5Context`

```nim
type MD5Context* {.final.} = object
```

Holds the incremental state when hashing data in chunks. You normally create one, feed data into it, and then finalize it. The `toMD5` and `getMD5` convenience procedures manage this lifecycle for you automatically.

---

### `md5Init`

```nim
proc md5Init*(c: var MD5Context)
```

Resets (or initialises) an `MD5Context` to its starting state. Call this before you begin feeding data when using the low-level streaming API.

```nim
var ctx: MD5Context
md5Init(ctx)
```

> **Note:** If you use `toMD5` or `getMD5`, you never need to call `md5Init` yourself.

---

### `md5Update`

```nim
proc md5Update*(c: var MD5Context, input: openArray[uint8])
proc md5Update*(c: var MD5Context, input: cstring, len: int)
```

Feeds a chunk of data into an already-initialised `MD5Context`. Can be called any number of times — each call continues where the previous one left off. This is the key to hashing large inputs without loading everything into memory at once.

```nim
var ctx: MD5Context
md5Init(ctx)
md5Update(ctx, cast[openArray[uint8]]("Hello, "))
md5Update(ctx, cast[openArray[uint8]]("World!"))
# The internal state now reflects "Hello, World!" as a whole
```

---

### `md5Final`

```nim
proc md5Final*(c: var MD5Context, digest: var MD5Digest)
```

Applies the MD5 padding, finalises the computation, and writes the 16-byte result into `digest`. After calling this the context is in an undefined state and must not be used again without re-initialising.

```nim
var ctx: MD5Context
var d: MD5Digest
md5Init(ctx)
md5Update(ctx, cast[openArray[uint8]]("abc"))
md5Final(ctx, d)
echo $d  # "900150983cd24fb0d6963f7d28e17f72"
```

---

### `toMD5`

```nim
proc toMD5*(s: string): MD5Digest
```

The one-call convenience wrapper: initialises a context, feeds `s` into it, finalises, and returns the raw `MD5Digest`. Use this whenever you have the whole string available at once.

```nim
let digest = toMD5("abc")
assert $digest == "900150983cd24fb0d6963f7d28e17f72"
```

---

### `getMD5`

```nim
proc getMD5*(s: string): string
```

Identical to `toMD5` but immediately converts the result to a lowercase hex string. The most ergonomic choice for the common "give me a hex checksum" use case.

```nim
assert getMD5("abc") == "900150983cd24fb0d6963f7d28e17f72"
assert getMD5("") == "d41d8cd98f00b204e9800998ecf8427e"  # empty string has a well-known hash
```

---

### `$` (MD5Digest → string)

```nim
proc `$`*(d: MD5Digest): string
```

Converts a raw `MD5Digest` byte array into its lowercase hexadecimal string representation (32 characters).

```nim
let d = toMD5("nim")
echo $d  # lowercase hex, e.g. "aadf27b62af0a7f8bf6abef5f7b01b2e"
```

---

### `==` (MD5Digest)

```nim
proc `==`*(D1, D2: MD5Digest): bool
```

Compares two `MD5Digest` values byte-by-byte. Returns `true` if every byte matches. Note that this is **not** a constant-time comparison — do not use it to compare secrets against user input in security-sensitive code.

```nim
assert toMD5("abc") == toMD5("abc")
assert toMD5("abc") != toMD5("xyz")
```

---

## SHA-1

> **Module:** `sha1`

SHA-1 produces a 160-bit (20-byte) digest. It has been **formally deprecated since 2011** because practical collision attacks exist. Nevertheless, SHA-1 is still widely used for git commit identifiers, legacy checksums, and protocol compatibility. The module itself is not deprecated — it will remain available.

---

### Type: `Sha1Digest`

```nim
type Sha1Digest* = array[0 .. 19, uint8]
```

Raw 20-byte SHA-1 output.

---

### Type: `SecureHash`

```nim
type SecureHash* = distinct Sha1Digest
```

A type alias over `Sha1Digest` that gives it its own identity for type-safety. The `$` and `==` operators are defined specifically for `SecureHash`.

---

### Type: `Sha1State`

```nim
type Sha1State* = object
```

Mutable object holding the streaming SHA-1 computation state. Needed for the low-level `update`/`finalize` API.

---

### `newSha1State`

```nim
proc newSha1State*(): Sha1State
```

Creates and returns a freshly initialised `Sha1State`. This is the entry point for the incremental (streaming) hashing path.

```nim
var state = newSha1State()
```

> `secureHash` calls this internally, so you only need it for streaming use.

---

### `update` (SHA-1)

```nim
proc update*(ctx: var Sha1State, data: openArray[char])
```

Feeds a chunk of characters into the running SHA-1 computation. Can be called repeatedly for streaming. The order of calls matters — each call appends to the running message.

```nim
var state = newSha1State()
state.update("Hello, ")
state.update("World!")
# equivalent to hashing "Hello, World!" in one shot
```

---

### `finalize` (SHA-1)

```nim
proc finalize*(ctx: var Sha1State): Sha1Digest
```

Completes the SHA-1 padding and compression and returns the raw 20-byte `Sha1Digest`. After finalisation the state should be discarded.

```nim
var state = newSha1State()
state.update("Hello World")
let raw: Sha1Digest = state.finalize()
```

---

### `secureHash`

```nim
proc secureHash*(str: openArray[char]): SecureHash
```

The one-liner: hashes `str` in a single call and returns a `SecureHash`. This is the most common entry point.

```nim
let h = secureHash("Hello World")
assert $h == "0A4D55A8D778E5022FAB701977C5D840BBC486D0"
```

---

### `secureHashFile`

```nim
proc secureHashFile*(filename: string): SecureHash
```

Opens a file, reads it in 8 KB chunks (streaming), and returns its SHA-1 hash. Memory-efficient for large files.

```nim
let fileHash = secureHashFile("mydata.bin")
```

---

### `parseSecureHash`

```nim
proc parseSecureHash*(hash: string): SecureHash
```

Parses a 40-character uppercase hex string back into a `SecureHash` value. Useful when you have a stored hash (e.g. from a database or config file) and want to compare it against a freshly computed one using `==`.

```nim
let stored = parseSecureHash("0A4D55A8D778E5022FAB701977C5D840BBC486D0")
assert secureHash("Hello World") == stored
```

---

### `$` (SecureHash → string)

```nim
proc `$`*(self: SecureHash): string
```

Returns the uppercase 40-character hex string for a `SecureHash`.

```nim
echo $secureHash("nim")  # e.g. "AE6E4D1209F17B460503904FAD297B31E9CF6362" — well-known value
```

---

### `==` (SecureHash)

```nim
proc `==`*(a, b: SecureHash): bool
```

Compares two `SecureHash` values. Like the MD5 variant, this is **not** constant-time.

---

### `isValidSha1Hash`

```nim
proc isValidSha1Hash*(s: string): bool
```

Returns `true` if `s` is exactly 40 characters long and consists entirely of hexadecimal digits (`0-9`, `a-f`, `A-F`). A quick format-validation gate — it does **not** verify that the string corresponds to any real data.

```nim
assert isValidSha1Hash("0A4D55A8D778E5022FAB701977C5D840BBC486D0") == true
assert isValidSha1Hash("not-a-hash") == false
assert isValidSha1Hash("0A4D") == false  # too short
```

---

## SHA-2

> **Module:** `sha2`

SHA-2 is the current standard family for general-purpose cryptographic hashing. The module implements four variants:

| Variant | Output | Internal word | Typical use |
|---------|--------|---------------|-------------|
| SHA-224 | 28 bytes | 32-bit | When 256-bit headroom is needed but output must fit tight budgets |
| SHA-256 | 32 bytes | 32-bit | General purpose; most common choice |
| SHA-384 | 48 bytes | 64-bit | Truncated SHA-512 for extra security without full 512-bit output |
| SHA-512 | 64 bytes | 64-bit | Highest security; slower on 32-bit platforms |

The module offers two layers of API. The **static** layer (`ShaStateStatic`) is preferred because the compiler enforces the correct buffer size and returns a properly typed array. The **unchecked** layer (`ShaState`) trades safety for flexibility (runtime-selected algorithm, external buffer).

---

### Type: `ShaInstance`

```nim
type ShaInstance* = enum
  Sha_224, Sha_256, Sha_384, Sha_512
```

An enum identifying which SHA-2 variant to use. Passed to `initSha` or `secureHash` at runtime when the algorithm is determined dynamically.

---

### Types: `ShaDigest_224/256/384/512`

```nim
type
  ShaDigest_224* = array[28, char]
  ShaDigest_256* = array[32, char]
  ShaDigest_384* = array[48, char]
  ShaDigest_512* = array[64, char]
```

Fixed-size arrays holding the raw output of each variant. Because the size is encoded in the type, the compiler can catch mismatches at compile time.

---

### Type: `ShaStateStatic[instance]`

```nim
type ShaStateStatic*[instance: static ShaInstance] = distinct ShaState
```

A statically-typed SHA-2 computation state. The `instance` type parameter is embedded in the type itself, so `digest()` can return the exact `ShaDigest_NNN` array without any runtime dispatch. **Prefer this over `ShaState` when possible.**

---

### Type: `ShaState`

```nim
type ShaState* = distinct ShaContext
```

The unchecked variant. Algorithm is selected at runtime; the user is responsible for passing a large-enough buffer to `digest`.

---

### `digestLength` (SHA-2)

```nim
func digestLength*(instance: ShaInstance): int
```

Returns the number of output bytes for a given `ShaInstance` at runtime. Useful when allocating a buffer before calling the unchecked `digest`.

```nim
assert digestLength(Sha_256) == 32
assert digestLength(Sha_512) == 64
```

---

### `initSha_224` / `initSha_256` / `initSha_384` / `initSha_512`

```nim
func initSha_224*(): ShaStateStatic[Sha_224]
func initSha_256*(): ShaStateStatic[Sha_256]
func initSha_384*(): ShaStateStatic[Sha_384]
func initSha_512*(): ShaStateStatic[Sha_512]
```

Create a fresh, statically-typed SHA-2 state ready to accept data. These are the recommended starting points for most use cases. Each returns a state bound to a specific digest size — the compiler will check all subsequent operations against it.

```nim
var hasher = initSha_256()
```

---

### `initSha`

```nim
func initSha*(instance: ShaInstance): ShaState
```

Creates an unchecked SHA state selected by a runtime value. Use this when the algorithm is not known at compile time (e.g., user configuration, protocol negotiation).

```nim
let algo = Sha_512  # could come from user input
var hasher = initSha(algo)
```

---

### `initShaStateStatic`

```nim
func initShaStateStatic*(instance: static ShaInstance): ShaStateStatic[instance]
```

Generic form of the four `initSha_NNN` helpers. Useful when writing generic code parameterised over a `ShaInstance`.

---

### `update` (SHA-2)

```nim
proc update*[instance: static ShaInstance](state: var ShaStateStatic[instance]; data: openArray[char])
proc update*(state: var ShaState; data: openArray[char])
```

Feeds a chunk of data into the hasher. Can be called any number of times, in any chunk sizes — the result is identical to concatenating all chunks and hashing them at once. This is the method you call in a streaming loop.

```nim
var hasher = initSha_256()
hasher.update("The quick brown fox ")
hasher.update("jumps over the lazy dog")
```

---

### `digest` (SHA-2, static)

```nim
proc digest*(state: var ShaStateStatic[Sha_224]): ShaDigest_224
proc digest*(state: var ShaStateStatic[Sha_256]): ShaDigest_256
proc digest*(state: var ShaStateStatic[Sha_384]): ShaDigest_384
proc digest*(state: var ShaStateStatic[Sha_512]): ShaDigest_512
```

Finalises the hash and returns the appropriately-sized digest array. The return type is determined by which `ShaStateStatic` you have. After calling `digest`, the state has been consumed — do not call `update` again.

```nim
var hasher = initSha_256()
hasher.update("The quick brown fox jumps over the lazy dog")
let d = hasher.digest()
assert $d == "d7a8fbb307d7809469ca9abcb0082e4f8d5651e46d3cdb762d02d0bf37c9e592"
```

---

### `digest` (SHA-2, unchecked)

```nim
proc digest*(state: var ShaState; dest: var openArray[char]): int
```

Finalises the hash and writes the result into the caller-supplied `dest` buffer. Returns the number of bytes actually written. If `dest` is smaller than the digest, output is silently truncated — so make sure to allocate enough space (use `digestLength` to determine the right size).

```nim
var hasher = initSha(Sha_512)
hasher.update("hello")
var buf = newString(64)
let written = hasher.digest(buf)
assert written == 64
```

---

### `secureHash` (SHA-2, static)

```nim
proc secureHash*(instance: static ShaInstance; data: openArray[char]): auto
```

One-shot static convenience: initialise, update once, digest. The return type is automatically the correct `ShaDigest_NNN` for the chosen instance. Use this when you have the full data in memory and don't need streaming.

```nim
let d = secureHash(Sha_256, "Hello World")
echo $d
```

---

### `secureHash` (SHA-2, dynamic)

```nim
proc secureHash*(instance: ShaInstance; data: openArray[char]): seq[char]
```

Same idea but with a runtime-chosen instance. Returns a `seq[char]` of the appropriate length.

```nim
let algo = Sha_384
let d = secureHash(algo, "Hello World")
assert d.len == 48
```

---

### `$` (SHA-2 digest)

The `$` operator for SHA-2 digest arrays is re-exported from `sha_utils`. It converts any `ShaDigest_NNN` to a lowercase hexadecimal string.

```nim
let d = secureHash(Sha_256, "abc")
echo $d  # "ba7816bf8f01cfea414140de5dae2ec73b00361bbef0469348423f656b8f3d3"
```

---

## SHA-3 and SHAKE

> **Module:** `sha3`

SHA-3 is based on the **Keccak** sponge construction, making it structurally unrelated to MD5/SHA-1/SHA-2 despite sharing the "SHA" name. This independence is a significant advantage: weaknesses in SHA-2 do not carry over to SHA-3.

The module implements two API families:

- **SHA-3** — fixed-size digests (224 / 256 / 384 / 512 bits), same interface pattern as SHA-2.
- **SHAKE** — extendable-output functions (XOF). You squeeze out as many bytes as you need, in any number of reads. Three security strengths: 128, 256, and 512 bits.

---

### Type: `Sha3Instance`

```nim
type Sha3Instance* = enum
  Sha3_224, Sha3_256, Sha3_384, Sha3_512
```

Selects the SHA-3 variant.

---

### Types: `Sha3Digest_224/256/384/512`

Fixed-size arrays, analogous to SHA-2's digest types.

---

### Type: `Sha3StateStatic[instance]`

Statically typed SHA-3 computation state. Returned by `initSha3_NNN` helpers. Recommended for most use.

---

### Type: `Sha3State`

Unchecked SHA-3 state, runtime-selected instance.

---

### `digestLength` (SHA-3)

```nim
func digestLength*(instance: Sha3Instance): int
```

Returns the output byte count for the given SHA-3 instance: 28, 32, 48, or 64.

---

### `initSha3_224` / `initSha3_256` / `initSha3_384` / `initSha3_512`

```nim
func initSha3_224*(): Sha3StateStatic[Sha3_224]
func initSha3_256*(): Sha3StateStatic[Sha3_256]
func initSha3_384*(): Sha3StateStatic[Sha3_384]
func initSha3_512*(): Sha3StateStatic[Sha3_512]
```

Create a fresh statically-typed SHA-3 hasher. Recommended starting points.

```nim
var hasher = initSha3_256()
```

---

### `initSha3`

```nim
func initSha3*(instance: Sha3Instance): Sha3State
```

Creates an unchecked SHA-3 state at runtime. Use when the variant is not a compile-time constant.

---

### `initSha3StateStatic`

```nim
func initSha3StateStatic*(instance: static Sha3Instance): Sha3StateStatic[instance]
```

Generic form of the `initSha3_NNN` helpers.

---

### `update` (SHA-3)

```nim
proc update*[instance: static Sha3Instance](state: var Sha3StateStatic[instance]; data: openArray[char])
proc update*(state: var Sha3State; data: openArray[char])
```

Feeds data into the hasher incrementally. Same semantics as the SHA-2 `update`.

```nim
var hasher = initSha3_256()
hasher.update("The quick brown fox ")
hasher.update("jumps over the lazy dog")
```

---

### `digest` (SHA-3, static)

```nim
proc digest*(state: var Sha3StateStatic[Sha3_224]): Sha3Digest_224
proc digest*(state: var Sha3StateStatic[Sha3_256]): Sha3Digest_256
proc digest*(state: var Sha3StateStatic[Sha3_384]): Sha3Digest_384
proc digest*(state: var Sha3StateStatic[Sha3_512]): Sha3Digest_512
```

Finalises and returns the typed digest array.

```nim
var hasher = initSha3_256()
hasher.update("The quick brown fox jumps over the lazy dog")
let d = hasher.digest()
assert $d == "69070dda01975c8c120c3aada1b282394e7f032fa9cf32f4cb2259a0897dfc04"
```

---

### `digest` (SHA-3, unchecked)

```nim
proc digest*(state: var Sha3State; dest: var openArray[char]): int
```

Same as SHA-2 unchecked `digest`: writes to caller-supplied buffer, returns bytes written, truncates if buffer is too small.

---

### `secureHash` (SHA-3, static)

```nim
proc secureHash*(instance: static Sha3Instance; data: openArray[char]): auto
```

One-shot hashing with a compile-time chosen SHA-3 instance.

```nim
let d = secureHash(Sha3_512, "Hello World")
echo $d
```

---

### `secureHash` (SHA-3, dynamic)

```nim
proc secureHash*(instance: Sha3Instance; data: openArray[char]): seq[char]
```

One-shot hashing with a runtime-chosen SHA-3 instance. Returns `seq[char]`.

---

### Type: `ShakeInstance`

```nim
type ShakeInstance* = enum
  Shake128   # 128-bit security strength
  Shake256   # 256-bit security strength
  Shake512   # 512-bit (Keccak proposal, not formally in SHA-3 standard)
```

Selects the SHAKE XOF security strength.

---

### Type: `ShakeState`

```nim
type ShakeState* = distinct KeccakState
```

Holds the mutable SHAKE computation state. Unlike SHA-3, SHAKE does not have a static variant because its output length is inherently variable.

---

### `digestLength` (SHAKE)

```nim
func digestLength*(instance: ShakeInstance): int
```

Returns the **default** chunk size for the SHAKE instance: 16, 32, or 64 bytes. This is informational — `shakeOut` can write any number of bytes regardless.

---

### `initShake`

```nim
func initShake*(instance: ShakeInstance): ShakeState
```

Creates a fresh SHAKE state for the given instance. This is the sole entry point for all SHAKE operations.

```nim
var xof = initShake(Shake128)
```

---

### `update` (SHAKE)

```nim
proc update*(state: var ShakeState; data: openArray[char])
```

Feeds data into the SHAKE absorb phase. Call as many times as needed before calling `finalize`.

```nim
xof.update("The quick brown fox ")
xof.update("jumps over the lazy dog")
```

---

### `finalize` (SHAKE)

```nim
proc finalize*(state: var ShakeState)
```

Signals the end of input. **Must be called before any call to `shakeOut`.** Once called, you can no longer feed more data with `update`.

```nim
xof.finalize()
```

---

### `shakeOut`

```nim
proc shakeOut*(state: var ShakeState; dest: var openArray[char])
```

Squeezes bytes out of the XOF into `dest`. Can be called repeatedly after `finalize` — each call continues from where the previous one left off, producing a continuous stream. The total output is the same regardless of how you split it across calls.

This is the defining feature of XOFs: you can request exactly as many bytes as your protocol needs, and the stream is deterministic.

```nim
var xof = initShake(Shake128)
xof.update("secret")
xof.finalize()

var key: array[32, char]
var iv:  array[16, char]

xof.shakeOut(key)  # first 32 bytes
xof.shakeOut(iv)   # next 16 bytes
# key and iv are derived from the same source material
```

---

## bcrypt

> **Module:** `bcrypt`

bcrypt is a **password hashing** algorithm, not a general-purpose checksum. It is designed to be intentionally slow through a tunable cost factor, making brute-force attacks expensive even with dedicated hardware. It also incorporates a per-password salt to defeat precomputed lookup tables (rainbow tables).

The module implements the `2b` subversion (current OpenBSD standard) and can verify `2a` and `2y` (PHP) hashes for compatibility.

**bcrypt is the right choice for storing user passwords.** Do not use MD5, SHA-1, or even SHA-256 for passwords — those are fast, which is exactly what you don't want.

---

### Type: `CostFactor`

```nim
type CostFactor* = range[4..31]
```

An integer in the range `[4, 31]` representing the log₂ of the number of bcrypt iterations. A cost of `n` means 2ⁿ rounds of key expansion are performed. Each unit increase roughly doubles the computation time.

- **4** — minimum allowed; essentially no cost delay (only for testing).
- **10–12** — common production values as of 2020s hardware. Aim for ~100–300 ms hash time on your server.
- **14+** — for high-security scenarios where latency budgets allow it.

---

### Type: `Salt`

```nim
type Salt = object  # opaque
```

An opaque object carrying the random 16-byte salt bytes, the cost factor, and the bcrypt subversion character (`'a'`, `'b'`, or `'y'`). Never construct a `Salt` directly — use `generateSalt` or `parseSalt`. The `$` operator converts a `Salt` to its `$2b$NN$...` string form.

---

### Type: `SaltedHash`

```nim
type SaltedHash = object  # opaque
```

An opaque object holding a completed bcrypt hash, which includes the salt, cost factor, and 24 hashed bytes. The `$` operator converts it to the canonical `$2b$NN$...` string that you store in a database.

---

### `generateSalt`

```nim
proc generateSalt*(cost: CostFactor): Salt {.raises: ResourceExhaustedError.}
```

Generates a new random 16-byte salt using the operating system's CSPRNG (via `std/sysrand`). Always uses the `2b` subversion. The `cost` parameter controls how expensive subsequent hashing will be.

Call this once per password, at registration time. **Never reuse a salt across different passwords.**

```nim
let salt = generateSalt(12)  # ~300ms on modern hardware — adjust for your server
```

> **Error:** Raises `ResourceExhaustedError` if the OS cannot produce enough entropy (extremely rare).

---

### `parseSalt`

```nim
proc parseSalt*(salt: string): Salt {.raises: ValueError.}
```

Parses a `Salt` from either a bcrypt salt preamble (`$2b$NN$<22chars>`) or a full bcrypt hash string. This is the bridge between the string world (database storage) and the `Salt` object.

```nim
# From a preamble only
let s1 = parseSalt("$2b$12$LzUyyYdKBoEy9V4NTvxDH.")

# From a full stored hash — useful when you want to re-extract the salt
let s2 = parseSalt("$2b$12$LzUyyYdKBoEy9V4NTvxDH.O11KQP30/Zyp5pQAQ.0Cy89WnkD5Jjy")

assert $s1 == $s2  # same salt either way
```

> **Error:** Raises `ValueError` for unsupported subversions, out-of-range cost factors, or malformed salt data.

---

### `bcrypt`

```nim
proc bcrypt*(password: openArray[char]; salt: Salt): SaltedHash
```

The core hashing function. Takes a password and a salt, runs the Blowfish-based EKS expansion for 2^cost rounds, and returns a `SaltedHash`. Converting the result with `$` gives the canonical string to store.

Key implementation details:
- Passwords longer than **72 bytes** are silently truncated (behaviour of `2b`/`2y`).
- The `2a` subversion replicates an old OpenBSD wraparound bug for backward compatibility.

```nim
let salt   = generateSalt(10)
let hashed = bcrypt("correct horse battery staple", salt)
let stored = $hashed  # store this in your database
echo stored  # "$2b$10$..."
```

> **Security warning:** If you accept a `salt` from untrusted input, a malicious user could supply a salt with cost factor 31, causing a denial-of-service by forcing your server to compute an astronomically expensive hash. Always generate salts yourself.

---

### `verify`

```nim
proc verify*(password: openArray[char]; knownGood: string): bool
```

The standard login check. Re-hashes `password` using the salt embedded in `knownGood` and then compares the result. Returns `true` if the password matches.

This is the **correct** way to authenticate a password — do not try to extract and compare hash bytes manually.

```nim
let stored = "$2b$10$LzUyyYdKBoEy9V4NTvxDH.O11KQP30/Zyp5pQAQ.0Cy89WnkD5Jjy"

if verify(userInputPassword, stored):
  echo "Access granted"
else:
  echo "Access denied"
```

> **Security warning:** The same caution about untrusted `knownGood` strings applies here as to `bcrypt` itself — an attacker-controlled cost factor in the hash string could cause a DoS.

---

## Choosing the Right Algorithm

| Use case | Recommended |
|---|---|
| Password storage | **bcrypt** (cost ≥ 10) |
| General-purpose cryptographic integrity | **SHA-256** or **SHA-3-256** |
| High security / future-proofing | **SHA-512** or **SHA-3-512** |
| Variable-length key derivation | **SHAKE256** |
| Legacy protocol compatibility | SHA-1 (read-only; don't create new uses) |
| File checksums, non-security use | MD5 or SHA-1 |

Never use MD5 or SHA-1 for new security-sensitive code. For password storage specifically, also consider Argon2 (not in this package) for resistance against GPU-based attacks.
