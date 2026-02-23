# md5 — Module Reference

> **⚠️ Deprecation notice:** This module is deprecated in recent versions of Nim.
> The recommended replacement is the `checksums` package:
> ```
> nimble install checksums
> ```
> Then use `import checksums/md5` instead.

---

## Overview

The `md5` module provides tools for computing **MD5 checksums** — fixed-length 128-bit (16-byte) fingerprints of arbitrary data. MD5 is a widely known hash function defined in [RFC 1321](https://www.ietf.org/rfc/rfc1321.txt).

**Key properties:**
- The same input always produces the same 128-bit digest.
- Even a tiny change in input produces a completely different digest.
- Works at **compile time** and in **JavaScript** targets.
- MD5 is **not cryptographically secure** against collision attacks and should not be used for security-critical tasks (password hashing, digital signatures). It remains useful for checksums, data integrity checks, and legacy compatibility.

---

## Exported Types

### `MD5Digest`

```nim
type MD5Digest* = array[0..15, uint8]
```

Represents an MD5 checksum as a raw array of 16 bytes. This is the core result type returned by `toMD5`. You can convert it to a human-readable hex string with the `$` operator, or compare two digests with `==`.

---

### `MD5Context`

```nim
type MD5Context* {.final.} = object
  ...
```

An opaque object that holds the **incremental hashing state**. It is used together with `md5Init`, `md5Update`, and `md5Final` when you need to hash data arriving in multiple chunks (e.g., reading a large file piece by piece). You never need to inspect its fields directly.

---

## Exported Procedures

---

### `toMD5`

```nim
proc toMD5*(s: string): MD5Digest
```

**The simplest way to compute an MD5 checksum.** Takes an entire string and returns an `MD5Digest`.

Internally it creates and manages an `MD5Context` for you — you do not need to call `md5Init`, `md5Update`, or `md5Final` yourself.

**Example:**

```nim
import md5

let digest = toMD5("hello, world")
echo digest          # prints the raw array representation
echo $digest         # prints: e4d7f1b4ed2e42d15898f4b27b019da4
```

**When to use:** whenever you have the complete data in memory as a `string` and just want the digest in one call.

---

### `$` (MD5Digest to string)

```nim
proc `$`*(d: MD5Digest): string
```

Converts an `MD5Digest` (16 raw bytes) into the familiar **32-character lowercase hexadecimal string**. This is the standard human-readable form of an MD5 hash.

**Example:**

```nim
import md5

let digest = toMD5("abc")
echo $digest   # 900150983cd24fb0d6963f7d28e17f72
```

**When to use:** any time you need to print, log, compare as text, or store an MD5 value as a string.

---

### `getMD5`

```nim
proc getMD5*(s: string): string
```

A **convenience shortcut** that computes the MD5 of a string and immediately returns the hex string result. It is exactly equivalent to writing `$toMD5(s)`, but saves you from having to do the conversion manually.

**Example:**

```nim
import md5

echo getMD5("abc")          # 900150983cd24fb0d6963f7d28e17f72
echo getMD5("abc") ==
     getMD5("abc")          # true — same input, same result
echo getMD5("abc") ==
     getMD5("ABC")          # false — MD5 is case-sensitive
```

**When to use:** whenever you need the checksum directly as a `string` and the data is already fully in memory.

---

### `==` (MD5Digest equality)

```nim
proc `==`*(D1, D2: MD5Digest): bool
```

Compares two `MD5Digest` values **byte by byte** and returns `true` only if all 16 bytes are identical. This is the standard way to verify that two pieces of data have the same checksum.

**Example:**

```nim
import md5

let a = toMD5("hello")
let b = toMD5("hello")
let c = toMD5("world")

echo a == b   # true  — identical input
echo a == c   # false — different input
```

**When to use:** when you want to verify data integrity by comparing two `MD5Digest` values without converting them to strings first. More efficient than converting both to strings and comparing those.

---

### `md5Init`

```nim
proc md5Init*(c: var MD5Context) {.raises: [], tags: [], gcsafe.}
```

**Initialises an `MD5Context`**, resetting it to the standard MD5 starting state. This must always be the first call in the streaming API workflow. The procedure sets the four internal 32-bit state words to the magic constants defined by the MD5 specification and zeroes the internal counters and buffer.

**Example:**

```nim
import md5

var ctx: MD5Context
md5Init(ctx)   # ctx is now ready to accept data
```

**When to use:** always, as the mandatory first step when using the low-level `md5Init` / `md5Update` / `md5Final` API. If you use `toMD5` or `getMD5`, this is called for you automatically.

---

### `md5Update` (openArray variant)

```nim
proc md5Update*(c: var MD5Context, input: openArray[uint8]) {.raises: [], tags: [], gcsafe.}
```

**Feeds a chunk of bytes into the ongoing MD5 computation.** The context accumulates the data internally, processing full 64-byte blocks as they become available. You can call this as many times as needed, with chunks of any size.

**Example:**

```nim
import md5

var ctx: MD5Context
var digest: MD5Digest

md5Init(ctx)
md5Update(ctx, [0x61'u8, 0x62'u8, 0x63'u8])  # bytes for "abc"
md5Final(ctx, digest)

echo $digest   # 900150983cd24fb0d6963f7d28e17f72
```

**When to use:** when your data is already in byte-array form, or when you are building a streaming solution and reading chunks of binary data.

---

### `md5Update` (cstring variant)

```nim
proc md5Update*(c: var MD5Context, input: cstring, len: int) {.raises: [], tags: [], gcsafe.}
```

An overload of `md5Update` that accepts a `cstring` (C-style pointer to characters) and an explicit byte length. It delegates to the `openArray[uint8]` variant internally by slicing the cstring. Useful when interfacing with C libraries or when working with raw string pointers.

**Example:**

```nim
import md5

var ctx: MD5Context
var digest: MD5Digest
let raw: cstring = "hello"

md5Init(ctx)
md5Update(ctx, raw, 5)   # explicitly pass the length
md5Final(ctx, digest)

echo $digest   # 5d41402abc4b2a76b9719d911017c592
```

**When to use:** when you have a `cstring` and its length, typically when working with C FFI or low-level buffers.

---

### `md5Final`

```nim
proc md5Final*(c: var MD5Context, digest: var MD5Digest) {.raises: [], tags: [], gcsafe.}
```

**Finalises the MD5 computation** and writes the 16-byte result into `digest`. This procedure applies the MD5 padding (a `0x80` byte followed by zeros, then the total bit-length as a 64-bit little-endian integer) as specified by RFC 1321, performs the last compression rounds, and extracts the final state into the output array. After calling `md5Final`, the context is cleared and should not be used further without a fresh `md5Init`.

**Example:**

```nim
import md5

var ctx: MD5Context
var digest: MD5Digest

md5Init(ctx)
md5Update(ctx, [0x61'u8])  # "a"
md5Update(ctx, [0x62'u8])  # "b" — fed in two separate chunks
md5Update(ctx, [0x63'u8])  # "c"
md5Final(ctx, digest)

# Same result as toMD5("abc") — order and chunking don't matter
echo $digest   # 900150983cd24fb0d6963f7d28e17f72
```

**When to use:** as the mandatory final step of the streaming API, after all data has been fed through `md5Update`.

---

## Usage Patterns

### Pattern 1 — One-shot (most common)

Use this when the data is fully available as a `string`.

```nim
import md5

# Returns hex string directly
let checksum = getMD5("The quick brown fox jumps over the lazy dog")
echo checksum   # 9e107d9d372bb6826bd81d3542a419d6

# Returns MD5Digest, useful when you need binary comparison
let digest = toMD5("The quick brown fox jumps over the lazy dog")
if digest == toMD5("The quick brown fox jumps over the lazy dog"):
  echo "Data is intact"
```

---

### Pattern 2 — Streaming (chunked data)

Use this when data arrives in pieces — for example, reading a file in blocks.

```nim
import md5

proc hashChunks(chunks: seq[string]): string =
  var ctx: MD5Context
  var digest: MD5Digest
  md5Init(ctx)
  for chunk in chunks:
    md5Update(ctx, chunk.toOpenArrayByte(0, chunk.high))
  md5Final(ctx, digest)
  return $digest

let parts = @["hel", "lo, ", "world"]
echo hashChunks(parts)
# Same as getMD5("hello, world") — e4d7f1b4ed2e42d15898f4b27b019da4
```

---

### Pattern 3 — Compile-time checksum

Because the module works at compile time, you can compute checksums in `static` blocks or `const` expressions.

```nim
import md5

const compiledHash = getMD5("my compile-time constant")
echo compiledHash
```

---

## Summary Table

| Procedure / Operator | Input | Output | Use case |
|---|---|---|---|
| `toMD5(s)` | `string` | `MD5Digest` | One-shot, need binary digest |
| `getMD5(s)` | `string` | `string` | One-shot, need hex string |
| `$digest` | `MD5Digest` | `string` | Convert digest to hex string |
| `d1 == d2` | `MD5Digest, MD5Digest` | `bool` | Compare two digests |
| `md5Init(ctx)` | `var MD5Context` | *(side effect)* | Begin streaming session |
| `md5Update(ctx, data)` | `var MD5Context`, bytes | *(side effect)* | Feed a chunk of data |
| `md5Final(ctx, digest)` | `var MD5Context`, `var MD5Digest` | *(side effect)* | Finish and extract result |
