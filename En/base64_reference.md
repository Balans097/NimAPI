# base64 — Module Reference

> **Import:** `import std/base64`
> **Scope:** encoding arbitrary binary data (or strings) into an ASCII-compatible text format (Base64), and decoding it back.

The module implements the classic Base64 scheme: every three bytes of input data (24 bits) are converted into four characters of the output alphabet (6 bits per character). The module supports both the standard alphabet (RFC 4648, characters `+` and `/`) and the URL-safe/filesystem-safe variant (characters `-` and `_`), as well as line-wrapped MIME encoding for e-mail. A general convention of the module: the `safe: bool` parameter in every encoding procedure selects the alphabet, while decoding always detects the alphabet automatically from the characters themselves.

---

## Table of Contents

I. [Helper Facilities](#helper-facilities)
&nbsp;&nbsp;&nbsp;1. [`initDecodeTable`](#initdecodetable)

II. [Encoding Data](#encoding-data)
&nbsp;&nbsp;&nbsp;1. [`encode` (from byte/char)](#encode-from-byte-char)
&nbsp;&nbsp;&nbsp;2. [`encode` (deprecated integer variant)](#encode-deprecated-integer-variant)
&nbsp;&nbsp;&nbsp;3. [`encodeMime`](#encodemime)

III. [Decoding Data](#decoding-data)
&nbsp;&nbsp;&nbsp;1. [`decode`](#decode)

IV. [Practical Recipes](#practical-recipes)
&nbsp;&nbsp;&nbsp;1. [HTTP Basic Auth header](#recipe-basic-auth)
&nbsp;&nbsp;&nbsp;2. [URL-safe token](#recipe-url-token)
&nbsp;&nbsp;&nbsp;3. [Encoding a binary file for embedding in JSON](#recipe-json-embedding)
&nbsp;&nbsp;&nbsp;4. [MIME attachment for e-mail](#recipe-mime-attachment)
&nbsp;&nbsp;&nbsp;5. [Safely decoding external data](#recipe-safe-decoding)

V. [Quick Reference Table](#quick-reference-table)

VI. [Summary: which procedure to use](#summary-which-procedure-to-use)

---

## Helper Facilities

### `initDecodeTable`

```nim
proc initDecodeTable*(): array[256, char]
```

**What it does.** Builds the reverse-mapping table: for each of the 256 possible byte values, it computes which 6-bit base64 code that value corresponds to. The result is used as a static lookup table (`decodeTable`) during decoding — instead of re-checking, for every character of the input string, which alphabet range (`A-Z`, `a-z`, `0-9`, `+`/`-`, `/`/`_`) it belongs to, the decoder simply indexes into the ready-made array by `ord(character)`.

**Implementation notes.** The table is pre-filled with `invalidChar` (255) by default — this serves as a marker for "not a valid base64 character". Then, for each character range (`A-Z`, `a-z`, `0-9`), the code is computed via an offset from the ASCII code point, and the characters `+`/`-` and `/`/`_` from both alphabets (standard and URL-safe) are mapped to the same codes, 62 and 63 respectively — so the decoder reads data encoded with either alphabet equally well, without needing to be told which one was used. The procedure is called once at compile time (see the `decodeTable` constant below), so building the table costs nothing at runtime.

- No parameters.
- Returns `array[256, char]` — a mapping table "character code → 6-bit value", where 255 means "the character is not part of the base64 alphabet".

**Example:**

```nim
const decodeTable = initDecodeTable()
echo ord(decodeTable[ord('A')])  # prints 0 — 'A' is the first character of the alphabet
echo ord(decodeTable[ord('/')])  # prints 63 — the last character of the standard alphabet
echo ord(decodeTable[ord(' ')])  # prints 255 — a space is not part of the base64 alphabet
```

---

## Encoding Data

### `encode` (from byte/char)

```nim
proc encode*[T: byte|char](s: openArray[T], safe = false): string
```

**What it does.** Encodes the sequence of bytes or characters `s` into a base64 string. If `safe` is `true`, the URL-safe and filesystem-safe alphabet is used (`-` and `_` instead of `+` and `/`); otherwise, the standard RFC 4648 alphabet is used. An empty input array produces an empty string.

**Implementation notes.** Data is processed in groups of three bytes: each group is "packed" into a 24-bit integer (three consecutive `shl` operations shift the bytes into the high positions of a 32-bit buffer), and that number is then "unpacked" back into four 6-bit chunks via `shr` and an `and 63` mask, with each chunk turned into a character through the alphabet table. If the length of `s` is not a multiple of three, the last incomplete group (1 or 2 bytes) is padded with `=` characters — this is the standard base64 padding mechanism that lets the decoder know how many "real" bytes were in the last group. Complexity is O(n) in the length of the input; the result buffer is pre-allocated in a single `setLen` call to avoid incremental string growth.

- `s: openArray[T]` — a sequence of bytes (`byte`) or characters (`char`) to encode; not modified.
- `safe: bool` — if `true`, the URL-safe alphabet (`-`, `_`) is used; defaults to `false` (standard alphabet `+`, `/`).

**Examples:**

```nim
# typical case — encoding a string
echo encode("Hello World")  # prints "SGVsbG8gV29ybGQ="

# encoding an array of integers (bytes) and characters
echo encode([1'u8, 2, 3])          # prints "AQID"
echo encode(['h', 'e', 'y'])       # prints "aGV5"

# edge case — empty input
echo encode("")  # prints ""

# difference between the standard and URL-safe alphabet on the same data
echo encode("c\xf7>", safe = false)  # prints "Y/c+"
echo encode("c\xf7>", safe = true)   # prints "Y_c-"

# practical scenario: encoding credentials for basic HTTP authentication
let credentials = "user:secret"
echo encode(credentials)  # prints "dXNlcjpzZWNyZXQ="
```

---

### `encode` (deprecated integer variant)

```nim
proc encode*[T: SomeInteger and not byte](s: openArray[T], safe = false): string
  {.deprecated: "use `byte` or `char` instead".}
```

**What it does.** The same base64 encoding as the main `encode` procedure, but it accepts a sequence of arbitrary integer types (`int`, `uint16`, etc.) rather than only `byte`/`char`. Marked as deprecated: the Nim compiler emits a warning when it is used.

**Implementation notes.** It uses the same internal `encodeImpl` template as the main variant — the implementation is identical; only the type constraint on parameter `T` in the signature differs. Since elements of an integer type wider than `byte` are implicitly truncated to a byte when packed, using types wider than `byte` can lead to data loss or unexpected results — which is why this variant is considered undesirable.

- `s: openArray[T]` — a sequence of integers of any type except `byte`; not modified.
- `safe: bool` — same meaning as in the main `encode` variant.

**Example:**

```nim
# compiles, but produces a deprecation warning
{.push warning[Deprecated]: off.}
echo encode([1, 2, 3])  # prints "AQID" — same result as the byte-based variant
{.pop.}
```

---

### `encodeMime`

```nim
proc encodeMime*(s: string, lineLen = 75.Positive, newLine = "\r\n",
                 safe = false): string
```

**What it does.** Encodes the string `s` into base64, but splits the result into fixed-length lines of `lineLen` characters, separated by the `newLine` sequence — exactly as required by the MIME format (RFC 2045) for e-mail attachments. If the input string is empty, an empty string is returned without allocating any space for line breaks.

**Implementation notes.** Regular encoding is performed first via `encode`, and the result is then "sliced" into chunks of `lineLen` characters with `newLine` inserted between them. The output buffer is allocated once via `newString` at the exact required size (the length of the encoded string plus however many line breaks that length requires), after which copying is done through the internal `cpy` template, which advances the indices `i` (position in the result) and `j`/`k` (positions in the sources) without any intermediate string concatenation.

- `s: string` — the source string to encode; not modified.
- `lineLen: Positive` — the maximum length of a single line of encoded output (excluding line-break characters); defaults to 75, as prescribed by MIME.
- `newLine: string` — the sequence of characters separating output lines; defaults to `"\r\n"` (the e-mail standard).
- `safe: bool` — the alphabet choice, as in `encode`.

**Examples:**

```nim
# typical case — a short line length for illustration
echo encodeMime("Hello World", 4, "\n")
# prints:
# SGVs
# bG8g
# V29y
# bGQ=

# edge case — empty string
echo encodeMime("")  # prints ""

# edge case — the result is shorter than lineLen: no line break is added
echo encodeMime("Hi", 75, "\n")  # prints "SGk="

# practical scenario — encoding an e-mail body with the defaults (75 characters, CRLF)
let body = "The full text of the message that needs to be sent as a MIME attachment."
let mimeBody = encodeMime(body)
echo mimeBody
```

---

## Decoding Data

### `decode`

```nim
proc decode*(s: string): string
```

**What it does.** Decodes the string `s`, encoded in base64 (using either of the two alphabets — standard or URL-safe), back into the original bytes/string. Leading whitespace characters are skipped, and trailing whitespace and padding `=` characters are discarded automatically. An empty input string produces an empty result. If the input string contains a character that is not part of the base64 alphabet, the procedure raises `ValueError`, identifying the character and its position.

**Implementation notes.** Decoding proceeds in groups of four input characters, each of which is turned into a 6-bit code via the `decodeTable`; the four 6-bit codes (24 bits) are reassembled into three bytes using `shl`/`shr`/`or`. The main loop — the "hot path" — processes 4-character blocks without accounting for whitespace inside them, for speed; a separate whitespace check (`\n`, `\r`, space) runs before each block, which allows base64 strings split across multiple lines to be decoded correctly (for example, the output of `encodeMime`). The last incomplete group (2 or 3 significant characters) is handled in a separate branch after the main loop — this corresponds to cases where the original data was not a multiple of three bytes and received one or two `=` characters during encoding.

- `s: string` — a base64-formatted string (standard or URL-safe alphabet, possibly containing whitespace/line breaks and trailing `=`); not modified.
- Exception: `ValueError`, if the string contains a character outside the base64 alphabet.

**Examples:**

```nim
# typical case
echo decode("SGVsbG8gV29ybGQ=")  # prints "Hello World"

# leading whitespace is skipped
echo decode("  SGVsbG8gV29ybGQ=")  # prints "Hello World"

# edge case — empty string
echo decode("")  # prints ""

# decoding output produced with the URL-safe alphabet — the table understands both alphabets equally
echo decode("Y_c-")  # prints "c\xf7>"

# error case — an invalid character raises an exception
doAssertRaises(ValueError):
  discard decode("not_base64!")

# practical scenario — decoding a multi-line MIME attachment
let mimeEncoded = "SGVs\nbG8g\nV29y\nbGQ="
echo decode(mimeEncoded)  # prints "Hello World"
```

---

## Practical Recipes

### HTTP Basic Auth header
<a id="recipe-basic-auth"></a>

The HTTP Basic Auth scheme requires the username and password to be joined with a colon and base64-encoded, then sent in the `Authorization` header.

```nim
proc buildBasicAuthHeader(username, password: string): string =
  let credentials = username & ":" & password
  result = "Basic " & encode(credentials)

echo buildBasicAuthHeader("admin", "s3cr3t")
# prints "Basic YWRtaW46czNjcjN0"
```

---

### URL-safe token
<a id="recipe-url-token"></a>

If the encoded value needs to be passed as part of a URL (for example, a password-reset token), the URL-safe alphabet is used — it contains no `+` or `/` characters that would require additional escaping in a URL.

```nim
import std/random

proc generateUrlSafeToken(byteLen: int): string =
  var bytes = newSeq[byte](byteLen)
  for i in 0 ..< byteLen:
    bytes[i] = byte(rand(255))
  result = encode(bytes, safe = true)

let token = generateUrlSafeToken(16)
echo token  # e.g., "3F9k_Q-1mZaC7bXpNvR8sw"
```

---

### Encoding a binary file for embedding in JSON
<a id="recipe-json-embedding"></a>

Binary data (for example, the contents of a small image) cannot be placed directly into a JSON string — it is first encoded as base64.

```nim
import std/[json, os]

proc fileToJsonField(path: string): JsonNode =
  let raw = readFile(path)
  let payload = encode(raw)
  result = %*{"filename": extractFilename(path), "data": payload}

let node = fileToJsonField("logo.png")
echo pretty(node)
```

---

### MIME attachment for e-mail
<a id="recipe-mime-attachment"></a>

A classic task: preparing an e-mail attachment per MIME — with line wrapping at 76 characters (the de-facto standard used by many mail clients) and CRLF line breaks.

```nim
proc buildMimeAttachment(content: string): string =
  result = encodeMime(content, lineLen = 76, newLine = "\r\n")

let attachment = buildMimeAttachment("The contents of the attachment, which can be arbitrarily long.")
echo attachment
```

---

### Safely decoding external data
<a id="recipe-safe-decoding"></a>

Data arriving from outside the program (from the network, from a user) cannot be decoded blindly — the input string may not be valid base64, in which case `decode` raises an exception that must be caught.

```nim
proc tryDecodeBase64(s: string): tuple[ok: bool, value: string] =
  try:
    result = (ok: true, value: decode(s))
  except ValueError:
    result = (ok: false, value: "")

let (ok1, value1) = tryDecodeBase64("SGVsbG8=")
echo ok1, " ", value1  # prints "true Hello"

let (ok2, value2) = tryDecodeBase64("not valid data!!!")
echo ok2  # prints "false"
```

---

## Quick Reference Table

| Task | Modifies argument | Returns |
|---|---|---|
| Encode bytes/characters as base64 | no | a new `string` |
| Encode as URL-safe base64 | no | a new `string` (parameter `safe = true`) |
| Encode long text for MIME/e-mail | no | a new `string`, split into lines |
| Decode a base64 string back into data | no | a new `string` |
| Obtain the decode table manually | no | `array[256, char]` |

---

## Summary: which procedure to use

- Need to encode bytes, characters, or a string as base64 → use `encode`.
- Need the result to be safe for direct use in a URL or filename without escaping → use `encode(..., safe = true)`.
- Need to prepare a long base64 text for an e-mail attachment (MIME) with line breaks → use `encodeMime`.
- Need to encode a sequence of integers other than `byte` → use `encode`, converting the data to `byte` or `char` first (the deprecated integer variant is not recommended).
- Need to recover the original data from a base64 string (in either alphabet) → use `decode`.
- Need to build the character-to-6-bit-code mapping table yourself (for example, for a custom implementation) → use `initDecodeTable`.
