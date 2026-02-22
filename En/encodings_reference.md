# `encodings` Module Reference

> **Nim Standard Library — `std/encodings`**
> Routines for converting strings between different character encodings.
> Uses `iconv` on Unix/macOS/BSD and the Windows API on Windows.

---

## Table of Contents

1. [Overview & Mental Model](#overview--mental-model)
2. [Types](#types)
   - [EncodingConverter](#encodingconverter)
   - [EncodingError](#encodingerror)
3. [Procedures](#procedures)
   - [getCurrentEncoding](#getcurrentencoding)
   - [open](#open)
   - [close](#close)
   - [convert (converter object)](#convert-converter-object)
   - [convert (one-shot)](#convert-one-shot)
4. [Encoding Names Reference](#encoding-names-reference)
5. [Platform Notes & Caveats](#platform-notes--caveats)
6. [Complete Worked Examples](#complete-worked-examples)

---

## Overview & Mental Model

The `encodings` module solves one fundamental problem: **you have a string of bytes that was encoded in charset X, and you need it in charset Y**.

Every string in Nim is just a sequence of bytes. When text is encoded in Latin-1, KOI8-R, Shift-JIS, or any other encoding, those bytes have different meanings than they would in UTF-8. This module provides the machinery to reinterpret and re-encode those bytes correctly.

There are two usage modes:

```
Mode 1: One-shot conversion
───────────────────────────
  convert(myString, destEncoding, srcEncoding)
  Opens a converter, converts once, closes it. Convenient for occasional use.

Mode 2: Reusable converter
──────────────────────────
  var c = open(destEncoding, srcEncoding)
  c.convert(string1)
  c.convert(string2)   ← converter is reused, more efficient
  c.close()
```

Under the hood, the module delegates all actual conversion work to the operating system:
- On **Unix, macOS, BSD**: calls the POSIX `iconv` library.
- On **Windows**: calls `MultiByteToWideChar` / `WideCharToMultiByte` from `kernel32.dll`, routing through UTF-16 as an intermediate representation.

---

## Types

### `EncodingConverter`

```nim
# On Unix:
type EncodingConverter* = ptr ConverterObj

# On Windows:
type EncodingConverter* = object
  dest, src: CodePage
```

A handle to an open conversion session. It holds the source and destination encoding information and any internal state needed by the underlying system library. You must call `open()` to create one and `close()` to release it when done.

Treat this as an opaque handle — do not inspect its fields directly. The internal representation differs between platforms.

---

### `EncodingError`

```nim
type EncodingError* = object of ValueError
```

An exception type that signals problems related to character encoding. It is raised in two situations:

1. **When opening a converter**: if one or both of the encoding names you provided are unknown or unsupported (e.g. a typo in `"UTF-8"` or requesting `"UTF-32BE"` on Windows).
2. **On Windows**: if you attempt to convert from `unicodeFFFE` (UTF-16 big-endian), `utf-32`, or `utf-32BE` — encodings that the Windows API's `MultiByteToWideChar` cannot handle as source.

Because `EncodingError` inherits from `ValueError`, you can catch it with `except ValueError` or more specifically with `except EncodingError`.

---

## Procedures

### `getCurrentEncoding`

```nim
proc getCurrentEncoding*(uiApp = false): string
```

**What it does:** Returns the name of the character encoding currently in use by the system.

The behaviour differs by platform:

- **On Unix/macOS/BSD**: always returns `"UTF-8"`, regardless of `uiApp`. Modern Unix systems have standardised on UTF-8, so `iconv` does not need to know the "current" encoding.
- **On Windows**: queries the Windows API. If `uiApp` is `false` (the default), it returns the encoding of the **console** (the code page used by the command prompt window). If `uiApp` is `true`, it returns the **ANSI code page** used by GUI applications — the encoding in which Windows stores filenames and UI text.

**Parameter:**
- `uiApp` — Windows-only flag. `false` → console code page (`GetConsoleCP`). `true` → ANSI/UI code page (`GetACP`).

```nim
import std/encodings

let enc = getCurrentEncoding()
echo enc  # "UTF-8" on Linux/macOS; e.g. "windows-1252" on a Western-locale Windows system

# On Windows, distinguish between UI and console encodings:
let uiEnc   = getCurrentEncoding(uiApp = true)   # e.g. "windows-1252"
let consEnc = getCurrentEncoding(uiApp = false)  # e.g. "ibm850"
```

**Practical use case:** When you read a file whose encoding you don't know, but you trust it was created on the same machine, `getCurrentEncoding()` gives you a reasonable guess for `srcEncoding`.

---

### `open`

```nim
proc open*(destEncoding = "UTF-8", srcEncoding = "CP1252"): EncodingConverter
```

**What it does:** Creates and returns a reusable `EncodingConverter` configured to translate strings **from** `srcEncoding` **to** `destEncoding`. This is the factory function for the converter object.

The defaults (`"UTF-8"` destination, `"CP1252"` source) reflect a common scenario: converting legacy Western-European text files to modern UTF-8.

**Parameters:**
- `destEncoding` — the encoding you want the output string to be in.
- `srcEncoding` — the encoding the input string is currently in.

**Raises `EncodingError`** if either encoding name is unrecognised by the underlying system (`iconv` on Unix, the Windows code-page table on Windows).

```nim
import std/encodings

# A converter from KOI8-R (old Russian encoding) to UTF-8
var c = open("UTF-8", "KOI8-R")

let russianKoi8 = "\xf0\xd2\xc9\xd7\xc5\xd4"  # "Привет" in KOI8-R bytes
echo c.convert(russianKoi8)  # "Привет"

c.close()
```

```nim
# Convert Shift-JIS Japanese text (common in older Windows/Japanese files) to UTF-8
var toUtf8 = open("UTF-8", "shift_jis")
defer: toUtf8.close()

let sjisBytes = "\x82\xb1\x82\xf1\x82\xc9\x82\xbf\x82\xcd"  # こんにちは in Shift-JIS
echo toUtf8.convert(sjisBytes)  # こんにちは
```

---

### `close`

```nim
proc close*(c: EncodingConverter)
```

**What it does:** Releases all resources held by the converter. On Unix this calls `iconv_close()` to free the library's internal state. On Windows there are no OS-level resources to release (the converter is a plain value object), so this is a no-op, but calling it is still good practice for portability and clarity.

Always call `close()` when you are done with a converter — especially in long-running programs, to avoid `iconv` handle leaks. Using `defer` is the idiomatic Nim pattern:

```nim
var c = open("UTF-8", "iso-8859-2")
defer: c.close()  # guaranteed to run even if an exception is raised

let result = c.convert(myInput)
```

---

### `convert` (converter object)

```nim
proc convert*(c: EncodingConverter, s: string): string
```

**What it does:** Converts the string `s` from the converter's source encoding to its destination encoding, and returns the result as a new string. This is the core operation of the module.

You can call this on the same converter any number of times — the converter is stateless between calls (each call is independent). This makes it efficient to re-use a single converter to process many strings in a loop.

**On Unix (iconv):** Unknown or unrepresentable characters are passed through as-is (the raw byte is copied to the output) rather than raising an error. The output buffer is grown dynamically if needed. A final flush call ensures multi-byte sequences at the end of input are completed correctly.

**On Windows:** Delegates to `MultiByteToWideChar` (source → UTF-16) then `WideCharToMultiByte` (UTF-16 → destination). The empty string is handled as a special case, since the Windows API returns 0 (which is also its error code) for zero-length input.

```nim
import std/encodings

var fromLatin2 = open("UTF-8", "iso-8859-2")
defer: fromLatin2.close()

# Process a list of strings — the converter is created once
let sentences = [
  "\xe8l\xf3vek",  # "člóvek" in ISO-8859-2
  "Dob\xfd den",   # "Dobrý den" in ISO-8859-2
]

for s in sentences:
  echo fromLatin2.convert(s)
```

---

### `convert` (one-shot)

```nim
proc convert*(s: string, destEncoding = "UTF-8", srcEncoding = "CP1252"): string
```

**What it does:** A convenience procedure that opens a converter, converts `s`, closes the converter, and returns the result — all in one call. This is the simplest way to do a single conversion.

The defaults match `open()`: destination `"UTF-8"`, source `"CP1252"` (Windows Western European).

**Raises `EncodingError`** if either encoding name is unrecognised.

**Performance note:** Because this opens and closes a converter on every call, it incurs setup/teardown overhead each time. For converting a single string, this is negligible. For converting hundreds or thousands of strings, use the reusable `EncodingConverter` from `open()` instead.

```nim
import std/encodings

# Simple one-liner: convert a CP1252-encoded string to UTF-8
let utf8 = convert("\xe9l\xe8ve")  # élève in CP1252 → UTF-8

# Explicit encoding names
let koi8text = "\xf0\xd2\xc9\xd7\xc5\xd4"
echo convert(koi8text, destEncoding = "UTF-8", srcEncoding = "KOI8-R")  # Привет

# Round-trip: UTF-8 → ISO-8859-1 → UTF-8
let original = "Héllo"
let latin1   = convert(original, "iso-8859-1", "UTF-8")
let restored = convert(latin1,   "UTF-8",      "iso-8859-1")
assert original == restored
```

---

## Encoding Names Reference

Encoding names are **case-insensitive** and **hyphens/underscores are interchangeable** when looked up on Windows (e.g. `"UTF-8"`, `"utf-8"`, `"utf_8"` are all equivalent). On Unix, `iconv` also accepts many aliases, but exact behaviour depends on the installed iconv version.

Below is a selection of the most commonly needed encodings:

| Name(s) | Description |
|---|---|
| `UTF-8` | Unicode, variable-width, universally used |
| `UTF-16` | Unicode, little-endian (Windows native wide-char) |
| `UTF-32` | Unicode, little-endian 32-bit (**not supported on Windows**) |
| `UTF-32BE` | Unicode, big-endian 32-bit (**not supported on Windows**) |
| `unicodeFFFE` | UTF-16, big-endian (**not supported on Windows as source**) |
| `us-ascii` | 7-bit ASCII |
| `iso-8859-1` | Latin-1, Western European |
| `iso-8859-2` | Latin-2, Central European (Czech, Polish, Slovak…) |
| `iso-8859-5` | Cyrillic |
| `iso-8859-7` | Greek |
| `iso-8859-8` | Hebrew |
| `windows-1250` / `cp-1250` | Windows Central European |
| `windows-1251` / `cp-1251` | Windows Cyrillic |
| `windows-1252` / `cp-1252` | Windows Western European (default srcEncoding) |
| `windows-1253` / `cp-1253` | Windows Greek |
| `koi8-r` | KOI8-R, Russian (Unix tradition) |
| `koi8-u` | KOI8-U, Ukrainian |
| `cp866` | DOS Cyrillic |
| `ibm850` | DOS Western European (OEM Latin-1) |
| `shift_jis` | Japanese (Windows/Shift-JIS) |
| `gb2312` / `gbk` | Simplified Chinese |
| `big5` | Traditional Chinese |
| `euc-jp` | Japanese EUC |
| `euc-kr` | Korean EUC |
| `ks_c_5601-1987` | Korean (Unified Hangul Code) |
| `iso-2022-jp` | Japanese (JIS, 7-bit) |

On Windows you can also use the numeric code page number as a string: `"1252"`, `"850"`, etc.

---

## Platform Notes & Caveats

**UTF-16BE, UTF-32, UTF-32BE on Windows**

The Windows API routes all conversions through UTF-16 Little-Endian as an intermediate. Encodings that are themselves wide-character formats in big-endian or 32-bit layouts cannot be used as the *source* of a conversion via `MultiByteToWideChar`. Attempting to do so raises `EncodingError` with a message explaining the limitation. As the destination these encodings are similarly unsupported. If you need these conversions on Windows, implement them manually or use a third-party library.

**Unknown/unmappable characters on Unix**

On Unix, if a byte sequence in the input cannot be mapped to the destination encoding, the raw byte is copied verbatim to the output (the character is "skipped" rather than replaced with a substitute). This means the output might contain invalid sequences if the source encoding is wrong. There is no error raised.

**Empty string input on Windows**

The Windows `MultiByteToWideChar` function returns `0` both for success with zero-length output and for errors. The module handles the empty string as a special case (`if s.len == 0: return ""`) before calling the API, so empty input always produces empty output without raising an error.

**iconv availability on Unix**

On Linux and macOS, `iconv` is part of the standard C library. On BSD systems the module links against `-liconv` explicitly (OpenBSD). On Haiku it loads `libiconv.so` dynamically. If `iconv` is not available on your target platform, the module will fail to compile or link.

---

## Complete Worked Examples

### Example 1: Read a legacy-encoded text file and print it as UTF-8

A common scenario: you have a file written by an old Windows application that saved text in CP1252 (Western European) or another system encoding, and you need to display or process it as UTF-8.

```nim
import std/[encodings, streams]

proc readAsUtf8(filename: string, srcEncoding: string): string =
  var f = newFileStream(filename, fmRead)
  if f == nil:
    raise newException(IOError, "Cannot open " & filename)
  defer: f.close()

  let raw = f.readAll()
  result = convert(raw, destEncoding = "UTF-8", srcEncoding = srcEncoding)

# Read a file saved in Windows Cyrillic encoding (CP1251)
let text = readAsUtf8("document.txt", "windows-1251")
echo text
```

---

### Example 2: Batch-convert many strings efficiently

When you have a large number of strings to convert, creating a reusable converter avoids the overhead of opening and closing the OS library handle on every call.

```nim
import std/encodings

proc convertAll(strings: seq[string], src, dest: string): seq[string] =
  var c = open(dest, src)
  defer: c.close()
  result = newSeq[string](strings.len)
  for i, s in strings:
    result[i] = c.convert(s)

let raw = @[
  "\xcf\xf0\xe8\xe2\xe5\xf2",   # "Привет" in CP1251
  "\xc4\xee\xe1\xf0\xfb\xe9",   # "Добрый" in CP1251
  "\xe4\xe5\xed\xfc",            # "день"   in CP1251
]

let utf8Strings = convertAll(raw, "windows-1251", "UTF-8")
for s in utf8Strings:
  echo s
# Привет
# Добрый
# день
```

---

### Example 3: Detect and handle the current system encoding

Write a program that reads from standard input and outputs UTF-8, regardless of the current system encoding.

```nim
import std/[encodings, os]

let sysEnc = getCurrentEncoding()
echo "System encoding: ", sysEnc

if sysEnc == "UTF-8":
  # Already UTF-8, just pass through
  echo stdin.readAll()
else:
  var c = open("UTF-8", sysEnc)
  defer: c.close()
  echo c.convert(stdin.readAll())
```

---

### Example 4: Safe conversion with error handling

Demonstrate catching `EncodingError` for bad encoding names.

```nim
import std/encodings

proc safeConvert(s, dest, src: string): string =
  try:
    result = convert(s, dest, src)
  except EncodingError as e:
    echo "Encoding error: ", e.msg
    result = s  # fall back to returning the original

# Valid conversion
echo safeConvert("hello", "UTF-8", "us-ascii")  # hello

# Invalid encoding name → EncodingError is caught
echo safeConvert("hello", "UTF-8", "notarealencoding")
# Encoding error: cannot create encoding converter from notarealencoding to UTF-8
# hello  ← fallback
```

---

### Example 5: Round-trip verification

Useful for testing that your source encoding assumption is correct — convert there and back, and verify the result matches the original.

```nim
import std/encodings

proc roundTripCheck(original: string, intermediate: string): bool =
  let encoded  = convert(original, intermediate, "UTF-8")
  let restored = convert(encoded,  "UTF-8",      intermediate)
  result = (restored == original)

let text = "Héllo Wörld"
echo roundTripCheck(text, "iso-8859-1")  # true  — all chars fit in Latin-1
echo roundTripCheck(text, "us-ascii")    # false — é, ö are not ASCII
```
