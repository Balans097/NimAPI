# `std/parseutils` Module Reference (Nim)

> A collection of helpers for lexical parsing of strings: tokens, numbers, identifiers, character skipping, string interpolation, and human-readable size parsing.

**Return value convention:** most procedures return the **number of characters consumed**, or `0` on a parse failure. This makes it easy to build hand-rolled chained parsers by advancing a position variable by the returned amount.

---

## Table of Contents

1. [Data Types](#data-types)
2. [Radix Number Parsing](#radix-number-parsing)
3. [Integer Parsing](#integer-parsing)
4. [Float Parsing](#float-parsing)
5. [Human-Readable Size Parsing](#human-readable-size-parsing)
6. [Identifiers and Characters](#identifiers-and-characters)
7. [Skipping Characters](#skipping-characters)
8. [Capturing Tokens](#capturing-tokens)
9. [String Interpolation](#string-interpolation)
10. [Practical Recipes](#practical-recipes)

---

## Data Types

### `InterpolatedKind`

An enum used by the `interpolatedFragments` iterator to describe the type of each fragment:

| Value       | Description                            |
|-------------|----------------------------------------|
| `ikStr`     | Plain string part                      |
| `ikDollar`  | Escaped `$$` → `$`                     |
| `ikVar`     | Variable reference `$name`             |
| `ikExpr`    | Expression `${expr}`                   |

---

## Radix Number Parsing

All three procedures (`parseBin`, `parseOct`, `parseHex`) share the same signature and behavior:

- Accept `openArray[char]` or `string` + `start`.
- Write the result into `number: var T` where `T: SomeInteger`.
- Return the number of characters consumed, or `0` on failure.
- **Do not check for overflow** — on overflow, only the last fitting bits are stored without raising an error.
- Support the `_` separator character (silently ignored).
- Automatically recognize prefixes: `0b`/`0B` (binary), `0o`/`0O` (octal), `0x`/`0X` or `#` (hex).

---

### `parseBin[T](s, number, maxLen=0)` / `parseBin[T](s, number, start=0, maxLen=0)`

Parses a binary (base-2) integer.

**Parameters:**
- `s` — input string or `openArray[char]`
- `number` — output variable
- `start` — starting position (string overload only)
- `maxLen` — maximum characters to parse (`0` = until end)

```nim
import std/parseutils

var num: int
doAssert parseBin("0100_1110", num) == 9
doAssert num == 0b0100_1110   # 78

# With 0b prefix
doAssert parseBin("0b1010", num) == 6
doAssert num == 10

# Error: non-binary digit
doAssert parseBin("2101", num) == 0

# int8 — only the last 8 bits are stored (no overflow error)
var b8: int8
doAssert parseBin("0b_0100_1110_0110_1001_1110_1101", b8) == 32
doAssert b8 == 0b1110_1101'i8

# maxLen: parse at most 9 chars starting from position 3
doAssert parseBin("0b_0100_1110_0110_1001_1110_1101", b8, 3, 9) == 9
doAssert b8 == 0b0100_1110'i8
```

---

### `parseOct[T](s, number, maxLen=0)` / `parseOct[T](s, number, start=0, maxLen=0)`

Parses an octal (base-8) integer. Recognizes the `0o`/`0O` prefix.

```nim
import std/parseutils

var num: int
doAssert parseOct("0o23464755", num) == 10
doAssert num == 5138925

# Error: 8 is not an octal digit
doAssert parseOct("8", num) == 0

# int8 overflow — no error
var n8: int8
doAssert parseOct("0o_1464_755", n8) == 11
doAssert n8 == -19

# With start and maxLen
doAssert parseOct("0o_1464_755", n8, 3, 3) == 3
doAssert n8 == 102
```

---

### `parseHex[T](s, number, maxLen=0)` / `parseHex[T](s, number, start=0, maxLen=0)`

Parses a hexadecimal (base-16) integer. Accepts `0x`/`0X` and `#` prefixes. Case-insensitive for `a`–`f`.

```nim
import std/parseutils

var num: int
doAssert parseHex("4E_69_ED", num) == 8
doAssert num == 5138925

# # prefix
doAssert parseHex("#ABC", num) == 4
doAssert num == 0xABC

# Error
doAssert parseHex("X", num) == 0

# uint8
var u8: uint8
doAssert parseHex("0xFF", u8) == 4
doAssert u8 == 255

# Limit: 2 hex digits starting at position 3
var n8: int8
doAssert parseHex("0x_4E_69_ED", n8, 3, 2) == 2
doAssert n8 == 0x4E'i8
```

---

## Integer Parsing

### `parseBiggestInt(s, number): int`

**Signatures:**
```nim
proc parseBiggestInt(s: openArray[char], number: var BiggestInt): int
proc parseBiggestInt(s: string, number: var BiggestInt, start = 0): int
```

Parses a signed integer into `BiggestInt`. Supports `+`/`-` sign and `_` separators. Raises `ValueError` on overflow.

```nim
import std/parseutils

var res: BiggestInt
doAssert parseBiggestInt("9223372036854775807", res) == 19
doAssert res == 9223372036854775807   # int64.high

doAssert parseBiggestInt("-2024_05_09", res) == 11
doAssert res == -20240509

# From position 7
doAssert parseBiggestInt("-2024_05_02", res, 7) == 4
doAssert res == 502

# No number found — returns 0, number unchanged
doAssert parseBiggestInt("abc", res) == 0
```

---

### `parseInt(s, number): int`

**Signatures:**
```nim
proc parseInt(s: openArray[char], number: var int): int
proc parseInt(s: string, number: var int, start = 0): int
```

Like `parseBiggestInt` but stores into `int`. Raises `ValueError` if the value exceeds the platform's `int` range.

```nim
import std/parseutils

var res: int
doAssert parseInt("-2024_05_02", res) == 11
doAssert res == -20240502

doAssert parseInt("  42", res) == 0   # leading space is not valid
doAssert parseInt("42  ", res) == 2   # trailing space stops parsing
doAssert res == 42

# From position 1
doAssert parseInt("-100abc", res, 1) == 3
doAssert res == 100
```

---

### `parseSaturatedNatural(s, b): int`

**Signatures:**
```nim
proc parseSaturatedNatural(s: openArray[char], b: var int): int
proc parseSaturatedNatural(s: string, b: var int, start = 0): int
```

Parses a non-negative integer without any risk of overflow exceptions. On overflow, `b` is set to `high(int)`. Recommended over `parseInt` whenever overflow safety matters.

```nim
import std/parseutils

var res = 0
discard parseSaturatedNatural("848", res)
doAssert res == 848

# Overflow → high(int)
discard parseSaturatedNatural("999999999999999999999999", res)
doAssert res == high(int)

# Negative sign not accepted
doAssert parseSaturatedNatural("-5", res) == 0
```

---

### `parseBiggestUInt(s, number): int`

**Signatures:**
```nim
proc parseBiggestUInt(s: openArray[char], number: var BiggestUInt): int
proc parseBiggestUInt(s: string, number: var BiggestUInt, start = 0): int
```

Parses an unsigned integer into `BiggestUInt`. Strings starting with `-` raise `ValueError`.

```nim
import std/parseutils

var res: BiggestUInt
doAssert parseBiggestUInt("1111111111111111111", res) == 19
doAssert res == 1111111111111111111'u64

doAssert parseBiggestUInt("12", res) == 2
doAssert res == 12
```

---

### `parseUInt(s, number): int`

**Signatures:**
```nim
proc parseUInt(s: openArray[char], number: var uint): int
proc parseUInt(s: string, number: var uint, start = 0): int
```

Like `parseBiggestUInt` but stores into `uint`.

```nim
import std/parseutils

var res: uint
doAssert parseUInt("3450", res) == 4
doAssert res == 3450

# From position 2: parses "50"
doAssert parseUInt("3450", res, 2) == 2
doAssert res == 50
```

---

## Float Parsing

### `parseBiggestFloat(s, number): int`

**Signatures:**
```nim
proc parseBiggestFloat(s: openArray[char], number: var BiggestFloat): int
proc parseBiggestFloat(s: string, number: var BiggestFloat, start = 0): int
```

Parses a floating-point number into `BiggestFloat` (`float64`). Returns `0` on error.

```nim
import std/parseutils

var res: float
discard parseBiggestFloat("3.14159", res)
doAssert res == 3.14159
```

---

### `parseFloat(s, number): int`

**Signatures:**
```nim
proc parseFloat(s: openArray[char], number: var float): int
proc parseFloat(s: string, number: var float, start = 0): int
```

Parses a floating-point number into `float`. Supports scientific notation (`1e3`, `2.5E-4`).

```nim
import std/parseutils

var res: float
doAssert parseFloat("32", res) == 2
doAssert res == 32.0

doAssert parseFloat("32.57", res) == 5
doAssert res == 32.57

# From position 3: parses "57" → 57.0
doAssert parseFloat("32.57", res, 3) == 2
doAssert res == 57.0

doAssert parseFloat("1.5e3", res) == 5
doAssert res == 1500.0

# Error
doAssert parseFloat("abc", res) == 0
```

---

## Human-Readable Size Parsing

### `parseSize(s, size, alwaysBin=false): int`

**Signature:**
```nim
func parseSize(s: openArray[char], size: var int64, alwaysBin = false): int
```

Parses a "human-readable" data size string with metric or binary unit prefixes. Returns the number of characters consumed or `0` on error. The value is rounded to the nearest integer. Saturates to `int64.high` on overflow.

**Supported prefixes** (case-insensitive, except `i`):

| Prefix    | Metric             | Binary (`...i`)      |
|-----------|--------------------|----------------------|
| `k` / `K` | 10³ = 1,000        | 2¹⁰ = 1,024          |
| `m` / `M` | 10⁶ = 1,000,000    | 2²⁰ = 1,048,576      |
| `g` / `G` | 10⁹                | 2³⁰                  |
| `t` / `T` | 10¹²               | 2⁴⁰                  |
| `p` / `P` | 10¹⁵               | 2⁵⁰                  |
| `e` / `E` | 10¹⁸               | 2⁶⁰                  |

An optional trailing `B`/`b` is accepted and ignored.  
`alwaysBin = true` forces all prefixes to be treated as binary.

```nim
import std/parseutils

var res: int64

# Decimal metric
doAssert parseSize("10.5 MB", res) == 7
doAssert res == 10_500_000

# Binary (suffix 'i')
doAssert parseSize("64 mib", res) == 6
doAssert res == 67108864   # 64 * 2^20

# Forced binary mode; '/' stops the parse
doAssert parseSize("1G/h", res, alwaysBin = true) == 2
doAssert res == 1073741824   # 2^30

# Plain number, no unit
doAssert parseSize("1024", res) == 4
doAssert res == 1024

# Error: negative not accepted
doAssert parseSize("-5 MB", res) == 0
```

---

## Identifiers and Characters

### `parseIdent(s, ident): int` / `parseIdent(s): string`

**Signatures:**
```nim
proc parseIdent(s: openArray[char], ident: var string): int
proc parseIdent(s: openArray[char]): string
proc parseIdent(s: string, ident: var string, start = 0): int
proc parseIdent(s: string, start = 0): string
```

Parses a Nim identifier (first character from `{'a'..'z','A'..'Z','_'}`, then `{'a'..'z','A'..'Z','0'..'9','_'}`).

- The `var ident` overload writes into the variable and returns the length. On failure, `ident` is **not changed**.
- The no-`var` overload returns the identifier string, or `""` on failure.

```nim
import std/parseutils

var res: string

doAssert parseIdent("Hello World", res) == 5
doAssert res == "Hello"

# From position 6
doAssert parseIdent("Hello World", res, 6) == 5
doAssert res == "World"

# Space is not a valid start
doAssert parseIdent("Hello World", res, 5) == 0
# res is still "World" — unchanged on failure

# String-returning overload
doAssert parseIdent("_myVar123 rest") == "_myVar123"
doAssert parseIdent("123abc") == ""   # starts with digit — not an identifier
```

---

### `parseChar(s, c): int`

**Signatures:**
```nim
proc parseChar(s: openArray[char], c: var char): int
proc parseChar(s: string, c: var char, start = 0): int
```

Reads a single character into `c`. Returns `1` on success, `0` if the position is past the end of the string.

```nim
import std/parseutils

var c: char

doAssert "nim".parseChar(c, 0) == 1
doAssert c == 'n'

doAssert "nim".parseChar(c, 2) == 1
doAssert c == 'm'

# Past end of string
doAssert "nim".parseChar(c, 3) == 0
# c is unchanged
```

---

## Skipping Characters

### `skipWhitespace(s): int`

**Signatures:**
```nim
proc skipWhitespace(s: openArray[char]): int
proc skipWhitespace(s: string, start = 0): int
```

Skips whitespace characters (`' '`, `'\t'`, `'\v'`, `'\r'`, `'\n'`, `'\f'`). Returns the count of skipped characters.

```nim
import std/parseutils

doAssert skipWhitespace("Hello World") == 0
doAssert skipWhitespace(" Hello World") == 1
doAssert skipWhitespace("Hello World", 5) == 1   # the space between words
doAssert skipWhitespace("Hello  World", 5) == 2  # two spaces
doAssert skipWhitespace("\t\n  data") == 4
```

---

### `skip(s, token): int`

**Signatures:**
```nim
proc skip(s, token: openArray[char]): int
proc skip(s, token: string, start = 0): int
```

Checks whether `token` matches at the current position in `s`. Returns `token.len` on match, `0` on mismatch. Case-sensitive.

```nim
import std/parseutils

doAssert skip("2019-01-22", "2019") == 4
doAssert skip("2019-01-22", "19") == 0      # does not match at position 0
doAssert skip("2019-01-22", "19", 2) == 2   # matches at position 2
doAssert skip("CAPlow", "CAP") == 3
doAssert skip("CAPlow", "cap") == 0         # case matters
```

---

### `skipIgnoreCase(s, token): int`

**Signatures:**
```nim
proc skipIgnoreCase(s, token: openArray[char]): int
proc skipIgnoreCase(s, token: string, start = 0): int
```

Like `skip` but comparison is case-insensitive (ASCII only).

```nim
import std/parseutils

doAssert skipIgnoreCase("CAPlow", "cap") == 3
doAssert skipIgnoreCase("CAPlow", "CAP") == 3
doAssert skipIgnoreCase("Hello", "HELL") == 4
doAssert skipIgnoreCase("Hello", "world") == 0
```

---

### `skipUntil(s, until: set[char]): int`

**Signatures:**
```nim
proc skipUntil(s: openArray[char], until: set[char]): int
proc skipUntil(s: string, until: set[char], start = 0): int
```

Skips characters until any character from `until` is found or the end is reached.

```nim
import std/parseutils

doAssert skipUntil("Hello World", {'W', 'e'}) == 1   # 'e' at position 1
doAssert skipUntil("Hello World", {'W'}) == 6         # 'W' at position 6
doAssert skipUntil("Hello World", {'z'}) == 11        # not found — went to end
```

---

### `skipUntil(s, until: char): int`

**Signatures:**
```nim
proc skipUntil(s: openArray[char], until: char): int
proc skipUntil(s: string, until: char, start = 0): int
```

Like the set version but searches for a single specific character.

```nim
import std/parseutils

doAssert skipUntil("Hello World", 'o') == 4     # first 'o'
doAssert skipUntil("Hello World", 'o', 4) == 0  # already at 'o'
doAssert skipUntil("Hello World", 'W') == 6
doAssert skipUntil("Hello World", 'z') == 11    # not found
```

---

### `skipWhile(s, toSkip): int`

**Signatures:**
```nim
proc skipWhile(s: openArray[char], toSkip: set[char]): int
proc skipWhile(s: string, toSkip: set[char], start = 0): int
```

Skips characters for as long as they are members of `toSkip`.

```nim
import std/parseutils
from std/strutils import Digits

doAssert skipWhile("Hello World", {'H', 'e'}) == 2
doAssert skipWhile("Hello World", {'e'}) == 0  # 'H' is not in the set

# Extract a number from a string
let s = "2019 school start"
let yearLen = skipWhile(s, Digits)   # == 4
echo s[0 ..< yearLen]   # "2019"

# From a given position
doAssert skipWhile("Hello World", {'W', 'o', 'r'}, 6) == 3  # "Wor"
```

---

## Capturing Tokens

### `parseUntil(s, token, until: set[char]): int`

**Signatures:**
```nim
proc parseUntil(s: openArray[char], token: var string, until: set[char]): int
proc parseUntil(s: string, token: var string, until: set[char], start = 0): int
```

Reads characters into `token` until any character from the `until` set is encountered, or the end is reached.

```nim
import std/parseutils

var tok: string

doAssert parseUntil("Hello World", tok, {'W', 'o', 'r'}) == 4
doAssert tok == "Hell"

doAssert parseUntil("Hello World", tok, {'W', 'r'}) == 6
doAssert tok == "Hello "

# From position 3
doAssert parseUntil("Hello World", tok, {'W', 'r'}, 3) == 3
doAssert tok == "lo "
```

---

### `parseUntil(s, token, until: char): int`

**Signatures:**
```nim
proc parseUntil(s: openArray[char], token: var string, until: char): int
proc parseUntil(s: string, token: var string, until: char, start = 0): int
```

Reads characters into `token` until a specific delimiter character is found.

```nim
import std/parseutils

var tok: string

doAssert parseUntil("Hello World", tok, 'W') == 6
doAssert tok == "Hello "

doAssert parseUntil("Hello World", tok, 'o') == 4
doAssert tok == "Hell"

# From position 2
doAssert parseUntil("Hello World", tok, 'o', 2) == 2
doAssert tok == "ll"

# Practical: manual CSV field parsing
let csv = "alice,30,engineer"
var field: string
var pos = 0
pos += parseUntil(csv, field, ',', pos); echo field; inc pos  # alice
pos += parseUntil(csv, field, ',', pos); echo field; inc pos  # 30
pos += parseUntil(csv, field, ',', pos); echo field           # engineer
```

---

### `parseUntil(s, token, until: string): int`

**Signatures:**
```nim
proc parseUntil(s: openArray[char], token: var string, until: string): int
proc parseUntil(s: string, token: var string, until: string, start = 0): int
```

Reads characters into `token` until the substring `until` is found.

```nim
import std/parseutils

var tok: string

doAssert parseUntil("Hello World", tok, "Wor") == 6
doAssert tok == "Hello "

doAssert parseUntil("Hello World", tok, "Wor", 2) == 4
doAssert tok == "llo "

# Practical: extract HTML tag content
let html = "<b>Bold text</b>"
var content: string
var p = skip(html, "<b>")
discard parseUntil(html, content, "</b>", p)
echo content   # Bold text
```

---

### `parseWhile(s, token, validChars): int`

**Signatures:**
```nim
proc parseWhile(s: openArray[char], token: var string, validChars: set[char]): int
proc parseWhile(s: string, token: var string, validChars: set[char], start = 0): int
```

Reads characters into `token` for as long as they belong to `validChars`.

```nim
import std/parseutils
from std/strutils import Digits, Letters

var tok: string

doAssert parseWhile("Hello World", tok, {'W', 'o', 'r'}) == 0  # 'H' not in set
doAssert parseWhile("Hello World", tok, {'W', 'o', 'r'}, 6) == 3
doAssert tok == "Wor"

# Extract a word
doAssert parseWhile("hello123 rest", tok, Letters + Digits) == 8
doAssert tok == "hello123"

# Extract a numeric part
doAssert parseWhile("  42px", tok, Digits, 2) == 2
doAssert tok == "42"
```

---

### `captureBetween(s, first, second='\0'): string`

**Signatures:**
```nim
proc captureBetween(s: openArray[char], first: char, second = '\0'): string
proc captureBetween(s: string, first: char, second = '\0', start = 0): string
```

Finds the first occurrence of `first`, then returns everything up to `second`. If `second == '\0'`, the function searches for a second occurrence of `first`.

```nim
import std/parseutils

# Up to the second 'e'
doAssert captureBetween("Hello World", 'e') == "llo World"

# Between 'e' and 'r'
doAssert captureBetween("Hello World", 'e', 'r') == "llo Wo"

# From position 6: between 'l' and the next stop
doAssert captureBetween("Hello World", 'l', start = 6) == "d"

# Practical: extract parenthesized content
doAssert captureBetween("func(arg1, arg2)", '(', ')') == "arg1, arg2"

# Extract quoted string
doAssert captureBetween("""say "hello" now""", '"') == "hello"
```

---

## String Interpolation

### `interpolatedFragments(s): (InterpolatedKind, string)`

**Signatures:**
```nim
iterator interpolatedFragments(s: openArray[char]): tuple[kind: InterpolatedKind, value: string]
iterator interpolatedFragments(s: string): tuple[kind: InterpolatedKind, value: string]
```

Tokenizes a string into fragments for interpolation, recognizing `$var`, `${expr}`, and `$$` patterns. Raises `ValueError` on an unclosed `${` or an invalid `$` usage.

| Pattern      | Kind       | `value` content          |
|--------------|------------|--------------------------|
| `$name`      | `ikVar`    | `name`                   |
| `${expr}`    | `ikExpr`   | `expr` (without `{}`)    |
| `$$`         | `ikDollar` | `$`                      |
| plain text   | `ikStr`    | the text itself          |

```nim
import std/parseutils

var parts: seq[tuple[kind: InterpolatedKind, value: string]]
for k, v in interpolatedFragments("  $this is ${an  example}  $$"):
  parts.add (k, v)

doAssert parts == @[
  (ikStr,    "  "),
  (ikVar,    "this"),
  (ikStr,    " is "),
  (ikExpr,   "an  example"),
  (ikStr,    "  "),
  (ikDollar, "$")
]

# Minimal template engine
import std/tables
let tpl = "Hello, $name! You have $count messages."
let vars = {"name": "Alice", "count": "5"}.toTable
var result = ""
for kind, val in interpolatedFragments(tpl):
  case kind
  of ikStr:    result.add val
  of ikVar:    result.add vars.getOrDefault(val, "???")
  of ikExpr:   result.add vars.getOrDefault(val, "???")
  of ikDollar: result.add "$"
echo result   # Hello, Alice! You have 5 messages.
```

---

## Practical Recipes

### Chained hand-written parser

```nim
import std/parseutils

proc parseDate(s: string): tuple[y, m, d: int] =
  var pos = 0
  var num: int
  pos += parseInt(s, num, pos); result.y = num
  pos += skip(s, "-", pos)
  pos += parseInt(s, num, pos); result.m = num
  pos += skip(s, "-", pos)
  pos += parseInt(s, num, pos); result.d = num

let date = parseDate("2024-03-15")
echo date   # (y: 2024, m: 3, d: 15)
```

### Filtering log lines by date pattern

```nim
import std/parseutils

let logs = @["2019-01-10: OK_", "2019-01-11: FAIL_", "2019-01: aaaa"]
var outp: seq[string]

for log in logs:
  var res: string
  if parseUntil(log, res, ':') == 10:   # YYYY-MM-DD == 10 chars
    outp.add(res & " - " & captureBetween(log, ' ', '_'))

doAssert outp == @["2019-01-10 - OK", "2019-01-11 - FAIL"]
```

### Extracting a number token from a string

```nim
import std/parseutils
from std/strutils import Digits, parseInt

let input1 = "2019 school start"
let input2 = "3 years back"
let startYear = input1[0 ..< skipWhile(input1, Digits)]  # "2019"
let yearsBack = input2[0 ..< skipWhile(input2, Digits)]  # "3"
let examYear = parseInt(startYear) + parseInt(yearsBack)
doAssert "Examination is in " & $examYear == "Examination is in 2022"
```

### Parsing a `key = value` config line

```nim
import std/parseutils

proc parseKV(s: string): tuple[key, value: string] =
  var pos = 0
  pos += skipWhitespace(s, pos)
  pos += parseIdent(s, result.key, pos)
  pos += skipWhitespace(s, pos)
  pos += skip(s, "=", pos)
  pos += skipWhitespace(s, pos)
  discard parseUntil(s, result.value, {'\n', '#'}, pos)

let kv = parseKV("  timeout = 30  # seconds")
echo kv.key    # timeout
echo kv.value  # 30  
```

### Safe integer parsing without exceptions

```nim
import std/parseutils

proc tryParseInt(s: string): (bool, int) =
  var b = 0
  let consumed = parseSaturatedNatural(s, b)
  result = (consumed > 0, b)

let (ok, val) = tryParseInt("42abc")
# ok == true, val == 42 (stops at 'a')

let (ok2, val2) = tryParseInt("abc")
# ok2 == false, val2 == 0
```
