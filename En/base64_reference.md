# `base64` Module Reference

**Module:** `base64` (part of Nim's standard library)   
**Status:** ⚠️ Unstable API — interfaces may change between Nim versions.  
**Purpose:** Encode binary data to a Base64 ASCII string and decode it back, with support for standard, URL-safe, and MIME (email) variants.

---

## Overview

Base64 is a binary-to-text encoding scheme. It works by grouping every three input bytes (24 bits) into four 6-bit chunks, then mapping each chunk to one of 64 printable ASCII characters. The result is about one-third larger than the original but consists entirely of safe, printable characters that can travel through text-only channels such as email, JSON, URLs, and HTML attributes.

### The two alphabets

The module supports two character sets, chosen via the `safe` flag:

| Position | Standard (`safe=false`) | URL-safe (`safe=true`) |
|---|---|---|
| 62 | `+` | `-` |
| 63 | `/` | `_` |
| Padding | `=` | `=` |

The URL-safe alphabet (RFC 4648 §5) avoids `+` and `/`, which have special meanings in URLs and file paths. Both alphabets are otherwise identical.

### Padding

If the input length is not a multiple of 3, one or two `=` characters are appended to make the output length a multiple of 4:

| Input bytes mod 3 | Padding `=` characters |
|---|---|
| 0 | none |
| 1 | `==` |
| 2 | `=` |

---

## Exported API

| Symbol | Kind | Description |
|---|---|---|
| `encode[T](s, safe)` | proc | Encode `openArray[byte\|char]` to Base64 string |
| `encodeMime(s, lineLen, newLine, safe)` | proc | Encode string with line-wrapping for MIME/email |
| `initDecodeTable()` | proc | Build and return the 256-entry decode lookup table |
| `decode(s)` | proc | Decode a Base64 string back to its original bytes |

> The second overload of `encode` accepting `SomeInteger and not byte` is **deprecated** and not covered here.

---

## `encode`

```nim
proc encode*[T: byte|char](s: openArray[T], safe = false): string
```

### What it does

Converts any sequence of `byte` or `char` values into a Base64-encoded string. The input can be a `string`, a `seq`, an array literal, or any other `openArray`.

The `safe` parameter selects which of the two alphabets is used:
- `false` (default) — standard RFC 4648 §4 alphabet, using `+` and `/`.
- `true` — URL/filesystem-safe RFC 4648 §5 alphabet, using `-` and `_` instead.

The returned string is always purely ASCII. Its length is approximately `⌈len/3⌉ × 4`.

### When to use

Use `encode` for any data that needs to survive text-only transport: embedding images in HTML/CSS (`data:` URIs), sending binary payloads in JSON, storing binary blobs in databases that expect strings, generating HTTP Basic Auth headers, and so on. Use `safe = true` whenever the encoded string will appear in a URL query parameter or a file name.

### Examples

```nim
import base64

# Plain string
assert encode("Hello World") == "SGVsbG8gV29ybGQ="

# Input length divisible by 3 — no padding needed
assert encode("nim") == "bmlt"

# Input length mod 3 == 1 — two '=' padding chars
assert encode("a") == "YQ=="

# Input length mod 3 == 2 — one '=' padding char
assert encode("ab") == "YWI="

# Array of bytes (e.g. raw binary data)
assert encode([1'u8, 2, 3, 4, 5]) == "AQIDBAU="

# Array of chars
assert encode(['h', 'e', 'y']) == "aGV5"

# URL-safe alphabet: '/' becomes '_', '+' becomes '-'
assert encode("c\xf7>", safe = false) == "Y/c+"
assert encode("c\xf7>", safe = true)  == "Y_c-"

# Practical: embed a small binary as a data URI fragment
let png = [0x89'u8, 0x50, 0x4E, 0x47]  # PNG magic bytes
echo "data:image/png;base64," & encode(png)
```

---

## `encodeMime`

```nim
proc encodeMime*(s: string, lineLen = 75.Positive, newLine = "\r\n",
                 safe = false): string
```

### What it does

Encodes a string to Base64 and then wraps it into fixed-length lines separated by a configurable newline sequence. This matches the MIME email format specified in RFC 2045, which requires Base64 content to be broken into lines of at most 76 characters.

Internally the proc calls `encode` first to produce the full Base64 string, then slices it into segments of `lineLen` characters, joining them with `newLine`.

**Special cases:**
- If `s` is empty, an empty string is returned immediately.
- If the encoded result is already `≤ lineLen` characters, it is returned as-is with no line breaks inserted.
- If `newLine` is an empty string `""`, the result is returned without any wrapping (identical to calling `encode`).

**`lineLen`** — the maximum number of Base64 characters per line. Must be a `Positive` (≥ 1). Default is 75, which together with the two-character `\r\n` newline gives lines of 77 bytes — just within the MIME 76-character limit for the Base64 content itself.

**`newLine`** — the line separator string. Default `"\r\n"` (CRLF) as required by email standards. Use `"\n"` for Unix-style output or when embedding in contexts that do not need strict MIME compliance.

**`safe`** — same meaning as in `encode`.

### When to use

Use `encodeMime` whenever you are constructing an email body or MIME attachment. Many mail servers reject messages with Base64 lines longer than 76 characters. For all other use cases (APIs, data URIs, tokens) prefer plain `encode`.

### Examples

```nim
import base64

# Standard MIME encoding — CRLF, 75 chars per line
let mime = encodeMime("Hello, this is a test of MIME base64 encoding for email!")
# Each line is at most 75 Base64 characters followed by \r\n

# Custom line length and Unix newline — handy for readable output or testing
assert encodeMime("Hello World", 4, "\n") == "SGVs\nbG8g\nV29y\nbGQ="

# Empty input returns empty string
assert encodeMime("") == ""

# Result shorter than lineLen — no wrapping
assert encodeMime("Hi", 100) == "SGk="

# Disable wrapping entirely by passing an empty newline string
let noWrap = encodeMime("Hello World", 4, "")
assert noWrap == encode("Hello World")  # identical to plain encode

# URL-safe variant for MIME
let safeMime = encodeMime("c\xf7>", 4, "\n", safe = true)
echo safeMime  # Y_c- (short enough to fit on one line)
```

---

## `initDecodeTable`

```nim
proc initDecodeTable*(): array[256, char]
```

### What it does

Builds and returns a 256-element lookup table that maps every possible ASCII byte value to its 6-bit Base64 value. This table is used internally by `decode` to convert each Base64 character back to its numeric equivalent in O(1) time.

The table is computed **at compile time** (the result is stored in the constant `decodeTable`) and handles both alphabets simultaneously:

| Input character range | Mapped value |
|---|---|
| `A`–`Z` | 0–25 |
| `a`–`z` | 26–51 |
| `0`–`9` | 52–61 |
| `+` or `-` | 62 |
| `/` or `_` | 63 |
| Any other byte | 255 (= `invalidChar`, marks an illegal character) |

Note that both `+` and `-` map to 62, and both `/` and `_` map to 63. This means `decode` accepts input encoded with either the standard or the URL-safe alphabet without needing to know which one was used.

### When to use

You will rarely call this proc directly — it exists so the constant `decodeTable` can be initialised at compile time. However, it is exported, so you can call it if you need your own copy of the table (for example, to build a custom decoder with different error handling, or to inspect which characters are considered valid).

### Example

```nim
import base64

let table = initDecodeTable()

# Check that 'A' maps to 0 and 'a' maps to 26
echo int(table[ord('A')])   # 0
echo int(table[ord('Z')])   # 25
echo int(table[ord('a')])   # 26
echo int(table[ord('z')])   # 51
echo int(table[ord('0')])   # 52
echo int(table[ord('9')])   # 61
echo int(table[ord('+')])   # 62
echo int(table[ord('-')])   # 62  (same as '+')
echo int(table[ord('/')])   # 63
echo int(table[ord('_')])   # 63  (same as '/')
echo uint8(table[ord('!')])  # 255 — invalid character sentinel
```

---

## `decode`

```nim
proc decode*(s: string): string
```

### What it does

Converts a Base64-encoded string back to its original binary representation. Returns the decoded bytes as a Nim `string` (Nim strings are byte strings and can hold arbitrary binary data).

#### Whitespace handling

The decoder silently skips leading whitespace at the start of the string and inline `\n`, `\r`, and ` ` characters throughout the data. This means it can consume the multi-line output of `encodeMime` directly without pre-stripping newlines.

Trailing `=`, `\n`, `\r`, and ` ` characters are stripped before processing begins.

#### Alphabet agnosticism

Thanks to the decode table design (see `initDecodeTable`), `decode` accepts input encoded with either the standard alphabet (`+`, `/`) or the URL-safe alphabet (`-`, `_`) — or even a mix of both — without any flag or parameter.

#### Error handling

If any character that is not a valid Base64 character, whitespace, or padding `=` is encountered, a `ValueError` is raised with a detailed message including the offending character, its ordinal value, and its position in the input string.

```
ValueError: Invalid base64 format character `!` (ord 33) at location 4.
```

### When to use

Use `decode` whenever you receive Base64-encoded data and need the original bytes: reading MIME email attachments, consuming API responses that embed binary data as Base64, parsing `data:` URIs, and so on.

### Examples

```nim
import base64

# Basic round-trip
assert decode(encode("Hello World")) == "Hello World"

# Padding variants
assert decode("YQ==") == "a"    # 1 original byte
assert decode("YWI=") == "ab"   # 2 original bytes
assert decode("YWJj") == "abc"  # 3 original bytes, no padding

# Leading and inline whitespace is silently ignored
assert decode("  SGVsbG8gV29ybGQ=") == "Hello World"
assert decode("SGVs\r\nbG8g\r\nV29y\r\nbGQ=") == "Hello World"  # MIME lines

# URL-safe encoded input decoded without any flag
assert decode("Y_c-") == "c\xf7>"
assert decode("Y/c+") == "c\xf7>"   # standard alphabet — same result

# Decoding raw binary data
let original = [0x00'u8, 0xFF, 0x10, 0xAB]
let encoded  = encode(original)
let decoded  = decode(encoded)
assert decoded.len == 4
assert byte(decoded[0]) == 0x00
assert byte(decoded[1]) == 0xFF

# Error on invalid character
import std/unittest
expect ValueError:
  discard decode("SGVs!!GQ=")
```

---

## Full round-trip example

```nim
import base64, strutils

# --- Binary data (simulated small file) ---
let binaryData = "PK\x03\x04\x14\x00\x00\x00"  # ZIP local file header magic

# Encode for embedding in a JSON field
let encoded = encode(binaryData)
echo "Encoded:  ", encoded            # UEsDBAUAAAAA (no padding needed here)
echo "Length:   ", encoded.len, " chars for ", binaryData.len, " bytes"

# Decode back to verify integrity
let decoded = decode(encoded)
assert decoded == binaryData
echo "Round-trip OK"

# --- Email attachment ---
let attachment = "Dear user,\n\nPlease find the report attached.\n\nRegards"
let mime = encodeMime(attachment, lineLen = 76, newLine = "\r\n")
echo "\nMIME encoded attachment (first 80 chars):"
echo mime[0 .. min(79, mime.high)]

# Decode the MIME output directly (newlines are stripped automatically)
let recovered = decode(mime)
assert recovered == attachment
echo "MIME round-trip OK"

# --- URL-safe token (e.g. a password reset token) ---
let tokenBytes = "\xde\xad\xbe\xef\xca\xfe\xba\xbe"
let token = encode(tokenBytes, safe = true)
echo "\nURL-safe token: ", token      # 3q2-78r-ur6- (no +//)
assert decode(token) == tokenBytes   # decode handles both alphabets
```

---

## Common pitfalls

**Binary data in decoded strings.** Nim `string` is a byte buffer, not necessarily valid UTF-8. Passing a decoded binary string to procedures that expect UTF-8 text may produce garbled output or exceptions. Cast to `seq[byte]` if you need to treat the result as raw bytes.

**`encodeMime` line length vs MIME limit.** RFC 2045 mandates lines of at most 76 characters *including* the CRLF. With the default `lineLen = 75` and `newLine = "\r\n"` (2 chars), each transmitted line is 77 bytes, which is within the 76-character limit for the Base64 content portion only. If your mail library counts the CRLF against the 76-character limit, use `lineLen = 74`.

**URL-safe encoding and `=` padding.** Some URL contexts also require stripping the trailing `=` padding, since `=` is used as a query-string delimiter. `encode` and `encodeMime` do not strip padding — you must do this yourself if needed.

**Deprecated integer overload.** Calling `encode` with an array of `int`, `int32`, or other integer types (not `byte` or `char`) triggers a deprecation warning. Convert to `seq[byte]` or `array[N, byte]` first.

**This is an unstable API.** The module header warns that interfaces may change. Pin your Nim version if you depend on exact behaviour.
