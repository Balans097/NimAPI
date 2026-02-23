# Nim `base64` Module — Complete Reference

## Overview

The `base64` module implements **Base64 encoding and decoding** for Nim. Base64 is a binary-to-text encoding scheme that represents arbitrary binary data using only 64 printable ASCII characters: `A–Z`, `a–z`, `0–9`, and two symbol characters (either `+/` in standard mode or `-_` in URL-safe mode).

The core mechanic is simple: every 3 bytes of binary input (24 bits) are regrouped into 4 Base64 digits of 6 bits each, then each 6-bit value is mapped to a printable character. When the input length is not a multiple of 3, `=` padding characters are appended to make the output length a multiple of 4.

```
Input bytes:   H (0x48)      e (0x65)      l (0x6C)
Binary:        01001000      01100101      01101100
Regroup 6-bit: 010010  000110  010101  101100
Base64 digit:  18      6       21      44
Char:          S       G       V       s
```

### Why use Base64?

- **Email attachments** — MIME encoding requires that binary data (images, files) be represented as ASCII text. Base64 is the standard mechanism.
- **JSON and XML payloads** — embedding binary blobs (images, cryptographic keys, certificates) in text-based formats.
- **URLs** — the URL-safe variant replaces characters that have special meaning in URLs (`+` → `-`, `/` → `_`).
- **Data URIs** — embedding images directly in HTML/CSS.

### API stability note

The module is marked **Unstable API**, meaning signatures or behaviour may change in future Nim versions without a deprecation period.

---

## Exported API — Quick Reference

| Symbol | Kind | Description |
|--------|------|-------------|
| `encode` | proc | Encode a `byte` or `char` array to a Base64 string |
| `encodeMime` | proc | Encode a string with line-wrapping (MIME format) |
| `decode` | proc | Decode a Base64 string back to its original bytes |
| `initDecodeTable` | proc | Build the decoding lookup table (compile-time) |

---

## Alphabets

The module defines two Base64 alphabets as compile-time constants:

**Standard alphabet** (`safe = false`, the default):

```
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
a b c d e f g h i j k l m n o p q r s t u v w x y z
0 1 2 3 4 5 6 7 8 9 + /
```

**URL-safe alphabet** (`safe = true`, RFC 4648 §5):

```
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
a b c d e f g h i j k l m n o p q r s t u v w x y z
0 1 2 3 4 5 6 7 8 9 - _
```

The only difference is the last two characters: `+` and `/` are replaced by `-` and `_`. The URL-safe variant can be used directly in URLs, filenames, and HTTP query parameters without percent-encoding.

The decode table treats both `+` and `-` as value 62, and both `/` and `_` as value 63, so `decode` accepts output from either alphabet without any extra flags.

---

## `encode`

```nim
proc encode*[T: byte|char](s: openArray[T], safe = false): string
```

### What it does

Encodes the input `s` — an array or sequence of `byte` or `char` values — into a Base64 string and returns it. The result length is always a multiple of 4 (padding `=` signs are added as needed).

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `s` | `openArray[byte]` or `openArray[char]` | The data to encode. Accepts arrays, sequences, strings (via `openArray`), and any slice thereof. |
| `safe` | `bool` (default `false`) | If `true`, use the URL-safe alphabet (`-` and `_` instead of `+` and `/`). |

### Return value

A `string` containing the Base64-encoded representation of `s`. The string contains only ASCII characters and is always 4 bytes longer per 3 bytes of input (plus padding).

### Examples

**String encoding:**

```nim
import std/base64

assert encode("Hello World") == "SGVsbG8gV29ybGQ="
assert encode("")            == ""
assert encode("a")           == "YQ=="   # 1 byte → 2 chars + "=="
assert encode("ab")          == "YWI="   # 2 bytes → 3 chars + "="
assert encode("abc")         == "YWJj"   # 3 bytes → 4 chars, no padding
```

The length of the padding reveals how many bytes the last group contained: `==` means 1 byte, `=` means 2 bytes, no padding means a multiple of 3 bytes.

**Character array encoding:**

```nim
import std/base64

assert encode(['n', 'i', 'm']) == "bmlt"
assert encode(@['n', 'i', 'm']) == "bmlt"   # seq works too
```

**Byte array encoding:**

```nim
import std/base64

assert encode([1'u8, 2, 3, 4, 5]) == "AQIDBAU="
assert encode([0'u8, 0, 0])       == "AAAA"
```

**URL-safe encoding:**

```nim
import std/base64

# The bytes 0x63, 0xf7, 0x3e contain bits that map to '+' and '/'
# in standard Base64, but '-' and '_' in safe mode:
assert encode("c\xf7>", safe = false) == "Y/c+"
assert encode("c\xf7>", safe = true)  == "Y_c-"
```

**Encoding image data for a data URI:**

```nim
import std/base64

# Suppose you have raw PNG bytes:
let pngBytes: seq[byte] = @[0x89'u8, 0x50, 0x4e, 0x47]  # PNG header
let b64 = encode(pngBytes)
let dataUri = "data:image/png;base64," & b64
```

### Deprecated overload

There is a deprecated overload for `openArray[SomeInteger and not byte]` (i.e. `int`, `int32`, etc.). It still works but emits a deprecation warning. Use `byte` or `char` arrays instead:

```nim
# Deprecated — emits warning:
# encode([1, 2, 3])

# Correct:
encode([1'u8, 2, 3])
```

### How padding works

| Input bytes remaining | Padding appended | Example |
|-----------------------|-----------------|---------|
| 0 (exact multiple of 3) | None | `"abc"` → `"YWJj"` |
| 1 | `==` | `"a"` → `"YQ=="` |
| 2 | `=` | `"ab"` → `"YWI="` |

---

## `encodeMime`

```nim
proc encodeMime*(s: string, lineLen = 75.Positive, newLine = "\r\n",
                 safe = false): string
```

### What it does

Encodes a string into Base64 with **line-length limiting**, as required by the MIME email standard (RFC 2045). MIME specifies that Base64-encoded email content must be split into lines no longer than 76 characters, separated by `\r\n`. This function wraps the plain `encode` result at `lineLen` characters, inserting `newLine` between each chunk.

The function first computes the full Base64 encoding via `encode`, then slices it into lines. If the encoded result is already short enough to fit in one line (length ≤ `lineLen`) or if `newLine` is empty, the raw Base64 string is returned unchanged.

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `s` | `string` | — | The string to encode |
| `lineLen` | `Positive` | `75` | Maximum number of Base64 characters per line |
| `newLine` | `string` | `"\r\n"` | Line separator inserted between chunks |
| `safe` | `bool` | `false` | Use URL-safe alphabet if `true` |

### Return value

A `string` of Base64 characters broken into lines of at most `lineLen` characters, separated by `newLine`. The last line is not terminated with `newLine`.

### Examples

**Standard MIME encoding:**

```nim
import std/base64

let mime = encodeMime("Hello World")
# Result split at 75 chars (the whole string fits in one line here):
assert mime == "SGVsbG8gV29ybGQ="
```

**Short line length for demonstration:**

```nim
import std/base64

assert encodeMime("Hello World", 4, "\n") == "SGVs\nbG8g\nV29y\nbGQ="
```

Every 4 Base64 characters are followed by a newline, except the last group.

**Encoding a long binary payload for an email:**

```nim
import std/base64

let fileData = readFile("document.pdf")
let mimePayload = encodeMime(fileData)
# mimePayload is ready to be used as the body of a MIME part:
# Content-Transfer-Encoding: base64
# <blank line>
# <mimePayload>
```

**Custom line separator:**

```nim
import std/base64

# Unix line endings with 60-char lines:
let unix = encodeMime("some binary data here", lineLen = 60, newLine = "\n")
```

### Relationship to `encode`

`encodeMime` is a thin wrapper around `encode`. It calls `encode(s, safe)` first, then post-processes the result by inserting line separators. If you do not need line-wrapping, use `encode` directly — it is faster.

---

## `decode`

```nim
proc decode*(s: string): string
```

### What it does

Decodes a Base64 string `s` back into the original binary data and returns it as a `string` (Nim's `string` is a byte sequence, so it can hold arbitrary binary content).

The decoder is **tolerant** in several ways:
- Leading and embedded whitespace (`\n`, `\r`, ` `) is silently skipped, making it suitable for decoding multi-line MIME payloads directly.
- Trailing `=` padding characters, spaces, newlines, and carriage returns are stripped before processing.
- Both the standard alphabet (`+`, `/`) and the URL-safe alphabet (`-`, `_`) are accepted in the same input — `decode` handles both without any flag.

If a character in `s` is not a valid Base64 character and is not whitespace or padding, `decode` raises `ValueError` with a message that includes the invalid character, its ordinal value, and its position.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `s` | `string` | A Base64-encoded string (with optional whitespace and padding) |

### Return value

A `string` containing the decoded binary data. For text that was encoded as UTF-8, this is directly usable as a string. For arbitrary binary data, treat it as a byte sequence.

### Examples

**Basic round-trip:**

```nim
import std/base64

let original = "Hello World"
let encoded  = encode(original)
let decoded  = decode(encoded)
assert decoded == original
```

**Decoding with leading whitespace:**

```nim
import std/base64

assert decode("  SGVsbG8gV29ybGQ=") == "Hello World"
```

**Decoding a multi-line MIME payload:**

```nim
import std/base64

let mime = "SGVs\r\nbG8g\r\nV29y\r\nbGQ="
assert decode(mime) == "Hello World"
```

Embedded `\r\n` sequences from MIME line breaks are automatically skipped.

**Decoding URL-safe Base64:**

```nim
import std/base64

# URL-safe encoded by a different tool (uses - and _ instead of + and /):
let urlSafe = "Y_c-"
assert decode(urlSafe) == "c\xf7>"
```

**Decoding binary data:**

```nim
import std/base64

let b64 = encode([0x89'u8, 0x50, 0x4e, 0x47])   # PNG magic bytes
let raw = decode(b64)
assert raw[0] == '\x89'
assert raw[1] == 'P'
```

**Error handling for invalid input:**

```nim
import std/base64

try:
  let _ = decode("SGVs!G8=")   # '!' is not a valid Base64 character
except ValueError as e:
  echo e.msg
  # Invalid base64 format character `!` (ord 33) at location 4.
```

### Performance notes

The decoder pre-allocates the output buffer once based on the input length (`(size * 3 / 4) + 6`) to avoid repeated reallocations. The main loop processes 4 input characters at a time in a tight inner loop, with whitespace skipping deferred to the start of each group. Final resizing with `setLen` releases unused bytes.

---

## `initDecodeTable`

```nim
proc initDecodeTable*(): array[256, char]
```

### What it does

Constructs and returns the 256-entry lookup table used internally by `decode` to convert Base64 characters back to their 6-bit values. The table is computed **at compile time** (called in a `const` block) and stored as a module-level constant, so there is zero runtime cost for table initialisation.

You will almost never call this directly. It is exported mainly for use cases where you want to embed the decode table in generated code, inspect it, or build a custom decoder on top of it.

### Encoding in the table

| Character range | Stored value |
|----------------|-------------|
| `A`–`Z` | 0–25 |
| `a`–`z` | 26–51 |
| `0`–`9` | 52–61 |
| `+` and `-` (both) | 62 |
| `/` and `_` (both) | 63 |
| All other characters | 255 (`invalidChar`) |

Note that both `+`/`-` and `/`/`_` map to the same values, which is why `decode` transparently handles both standard and URL-safe encoded inputs.

### Example

```nim
import std/base64

let t = initDecodeTable()
echo int(t[ord('A')])   # 0
echo int(t[ord('Z')])   # 25
echo int(t[ord('a')])   # 26
echo int(t[ord('+')])   # 62
echo int(t[ord('-')])   # 62  ← same as '+'
echo int(t[ord('/')])   # 63
echo int(t[ord('_')])   # 63  ← same as '/'
echo int(t[ord('!')])   # 255 ← invalid
```

---

## Complete Practical Examples

### Round-trip encode and decode

```nim
import std/base64

proc roundTrip(data: string) =
  let enc = encode(data)
  let dec = decode(enc)
  assert dec == data, "Round-trip failed for: " & data

roundTrip("Hello, World!")
roundTrip("")
roundTrip("a")
roundTrip("\x00\x01\x02\xFF")   # binary data
roundTrip("Unicode: café naïve résumé")
```

---

### Embedding a file as a Base64 constant

```nim
import std/base64

# Embed a small resource at compile time:
const smallIcon = staticRead("icon.png")
const iconBase64 = encode(smallIcon)
# iconBase64 is a compile-time constant string

proc getDataUri(): string =
  "data:image/png;base64," & iconBase64
```

---

### MIME email attachment

```nim
import std/base64

proc buildMimePart(filename, contentType, data: string): string =
  result = ""
  result &= "Content-Type: " & contentType & "\r\n"
  result &= "Content-Disposition: attachment; filename=\"" & filename & "\"\r\n"
  result &= "Content-Transfer-Encoding: base64\r\n"
  result &= "\r\n"
  result &= encodeMime(data)
  result &= "\r\n"

let csvData = "name,value\nalice,42\nbob,99\n"
echo buildMimePart("report.csv", "text/csv", csvData)
```

---

### URL-safe token generation

```nim
import std/base64
import std/random

proc generateToken(byteLen: int = 24): string =
  ## Generates a URL-safe Base64 token from random bytes.
  ## byteLen=24 produces a 32-character token.
  var bytes = newSeq[byte](byteLen)
  for i in 0 ..< byteLen:
    bytes[i] = byte(rand(255))
  encode(bytes, safe = true)

randomize()
echo generateToken()   # e.g. "xK3mV-R_tQpLwZ8sNfYcAj2u"
```

---

### Decoding a Base64 JSON field

```nim
import std/base64
import std/json

let payload = """{"name": "document", "data": "SGVsbG8gV29ybGQ="}"""
let j = parseJson(payload)

let rawData = decode(j["data"].getStr())
echo rawData   # "Hello World"
```

---

## Error Conditions

| Situation | Error type | Message format |
|-----------|-----------|----------------|
| Invalid Base64 character in `decode` input | `ValueError` | `Invalid base64 format character \`X\` (ord N) at location P.` |

There is no error for:
- Trailing `=` padding (silently stripped)
- Embedded or leading whitespace (silently skipped)
- Missing padding (handled gracefully for the last 2–3 characters)

---

## See Also

- [`std/hashes`](https://nim-lang.org/docs/hashes.html) — hash computation for Nim types
- [`std/md5`](https://nim-lang.org/docs/md5.html) — MD5 checksum algorithm
- [`std/sha1`](https://nim-lang.org/docs/sha1.html) — SHA-1 checksum algorithm
- [RFC 4648](https://tools.ietf.org/html/rfc4648) — The Base16, Base32, and Base64 Data Encodings standard
- [RFC 2045](https://tools.ietf.org/html/rfc2045) — MIME Part One: Format of Internet Message Bodies
