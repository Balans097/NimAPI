# strutils.nim — Module Reference

> **Import:** `import std/strutils`
> **Scope:** additional procedures, iterators and templates for working with `string` values — character classification, case conversion, splitting/joining, searching, replacing, trimming/padding, number ↔ string conversions, escaping, and floating-point/size formatting.

The `system` module already provides the core string primitives (`$`, `&`, `add`, `in`/`contains`, `notin`). `strutils` builds on top of those primitives and provides everything else you typically need when working with text in Nim.

Most procedures here support **method call syntax**, so `s.split(',').join("-")` reads naturally as a pipeline.

---

## Table of Contents

1. [Overview table of all functions](#overview-table-of-all-functions)
2. [Constants, types and helpers](#constants-types-and-helpers)
3. [Main reference](#main-reference)
   - [Character classification](#character-classification)
   - [Case conversion](#case-conversion)
   - [Normalization and comparison](#normalization-and-comparison)
   - [Splitting](#splitting)
   - [Joining](#joining)
   - [Number ↔ string conversions](#number--string-conversions)
   - [Parsing](#parsing)
   - [Repeating, padding and alignment](#repeating-padding-and-alignment)
   - [Indentation](#indentation)
   - [Deleting and trimming](#deleting-and-trimming)
   - [Prefix / suffix operations](#prefix--suffix-operations)
   - [Searching](#searching)
   - [Counting and membership](#counting-and-membership)
   - [Replacing](#replacing)
   - [Escaping](#escaping)
   - [Identifier validation](#identifier-validation)
   - [Floating point formatting](#floating-point-formatting)
   - [Size and engineering notation formatting](#size-and-engineering-notation-formatting)
   - [String interpolation (`%` and `format`)](#string-interpolation--and-format)
   - [Tokenizing](#tokenizing)
4. [Recommendations: which function to use when](#recommendations-which-function-to-use-when)

---

## Overview table of all functions

| Name | Kind | Signature (short) | Purpose |
|---|---|---|---|
| `isAlphaAscii` | func | `(c: char): bool` | Is `c` an ASCII letter? |
| `isAlphaNumeric` | func | `(c: char): bool` | Is `c` a letter or digit? |
| `isDigit` | func | `(c: char): bool` | Is `c` a digit `0-9`? |
| `isSpaceAscii` | func | `(c: char): bool` | Is `c` whitespace? |
| `isLowerAscii` | func | `(c: char): bool` | Is `c` lowercase ASCII letter? |
| `isUpperAscii` | func | `(c: char): bool` | Is `c` uppercase ASCII letter? |
| `toLowerAscii` | func | `(c: char\|s: string)` | Convert to lower case (ASCII only) |
| `toUpperAscii` | func | `(c: char\|s: string)` | Convert to upper case (ASCII only) |
| `capitalizeAscii` | func | `(s: string): string` | Uppercase first character |
| `nimIdentNormalize` | func | `(s: string): string` | Normalize as Nim identifier |
| `normalize` | func | `(s: string): string` | Lowercase + remove `_` |
| `cmpIgnoreCase` | func | `(a, b: string): int` | Case-insensitive compare |
| `cmpIgnoreStyle` | func | `(a, b: string): int` | Style-insensitive compare |
| `split` (iterator/func) | iterator/func | `(s, sep\|seps\|sepStr, maxsplit)` | Split into substrings |
| `rsplit` (iterator/func) | iterator/func | `(s, sep\|seps\|sepStr, maxsplit)` | Split from the right |
| `splitLines` (iterator/func) | iterator/func | `(s, keepEol)` | Split into lines |
| `splitWhitespace` (iterator/func) | iterator/func | `(s, maxsplit)` | Split on whitespace, trimming ends |
| `toBin` | func | `(x: BiggestInt, len)` | Integer → binary string |
| `toOct` | func | `(x: BiggestInt, len)` | Integer → octal string |
| `toHex` | func | `(x: SomeInteger \| string)` | Integer/bytes → hex string |
| `toOctal` | func | `(c: char): string` | Char → 3-digit octal |
| `fromBin` / `fromOct` / `fromHex` | func | `[T](s: string): T` | Parse binary/octal/hex into integer |
| `intToStr` | func | `(x: int, minchars)` | Int → decimal string, zero-padded |
| `parseInt` / `parseBiggestInt` / `parseUInt` / `parseBiggestUInt` | func | `(s: string)` | Parse decimal integers |
| `parseFloat` | func | `(s: string): float` | Parse a float |
| `parseBinInt` / `parseOctInt` / `parseHexInt` | func | `(s: string): int` | Parse binary/octal/hex with prefixes |
| `parseHexStr` | func | `(s: string): string` | Hex text → byte string |
| `parseBool` | func | `(s: string): bool` | Parse `"yes"/"true"/"1"`, etc. |
| `parseEnum` | func | `[T](s: string[, default: T]): T` | Parse string into enum value |
| `repeat` | func | `(c: char\|s: string, n)` | Repeat a char/string `n` times |
| `spaces` | func | `(n: Natural): string` | `n` spaces |
| `align` | func | `(s, count, padding)` | Right-align (pad on the left) |
| `alignLeft` | func | `(s, count, padding)` | Left-align (pad on the right) |
| `center` | func | `(s, width, fillChar)` | Center a string |
| `indent` | func | `(s, count, padding)` | Indent every line |
| `unindent` | func | `(s, count, padding)` | Remove indentation |
| `indentation` | func | `(s: string): Natural` | Common indentation of all lines |
| `dedent` | func | `(s, count)` | Remove common indentation (spaces only) |
| `delete` | func | `(s: var string, slice\|first,last)` | Remove a slice of characters |
| `startsWith` / `endsWith` | func | `(s, prefix\|suffix: char\|string)` | Prefix/suffix test |
| `continuesWith` | func | `(s, substr, start)` | Does `s` contain `substr` at `start`? |
| `removePrefix` / `removeSuffix` | func | `(s: var string, chars\|c\|str)` | Strip prefix/suffix in place |
| `addSep` | func | `(dest: var string, sep, startLen)` | Conditionally append a separator |
| `allCharsInSet` | func | `(s, theSet): bool` | Are all chars in the set? |
| `abbrev` | func | `(s, possibilities): int` | Match unambiguous abbreviation |
| `join` | func | `(a: openArray, sep)` | Join elements into a string |
| `initSkipTable` / `find` (with table) | func | `(sub)` / `(a, s, sub, start, last)` | Boyer–Moore–Horspool search |
| `find` | func | `(s, sub: char\|set[char]\|string, start, last)` | Find substring/char/charset |
| `rfind` | func | `(s, sub, start, last)` | Find from the right |
| `count` | func | `(s, sub: char\|set[char]\|string)` | Count occurrences |
| `countLines` | func | `(s): int` | Count lines |
| `contains` | func | `(s, sub\|chars): bool` | Membership test |
| `replace` | func | `(s, sub, by)` | Replace all occurrences |
| `replaceWord` | func | `(s, sub, by)` | Replace whole-word occurrences |
| `multiReplace` | func | `(s, replacements)` | Multiple replacements in one pass |
| `insertSep` | func | `(s, sep, digits)` | Insert thousands separators |
| `escape` / `unescape` | func | `(s, prefix, suffix)` | Escape/unescape special characters |
| `validIdentifier` | func | `(s): bool` | Is `s` a valid identifier? |
| `formatBiggestFloat` / `formatFloat` | func | `(f, format, precision, decimalSep)` | Format floating point numbers |
| `trimZeros` | func | `(x: var string, decimalSep)` | Strip trailing zeros |
| `formatSize` | func | `(bytes, decimalSep, prefix, includeSpace)` | Human-readable byte sizes |
| `formatEng` | func | `(f, precision, trim, siPrefix, unit, ...)` | Engineering notation |
| `addf` | func | `(s: var string, formatstr, a)` | Efficient `add(s, formatstr % a)` |
| `%` | func | `(formatstr, a)` | String interpolation operator |
| `format` | func | `(formatstr, a)` | Same as `%` with auto-stringify |
| `strip` | func | `(s, leading, trailing, chars)` | Trim leading/trailing characters |
| `stripLineEnd` | func | `(s: var string)` | Remove one line-ending sequence |
| `tokenize` | iterator | `(s, seps)` | Yield `(token, isSep)` pairs |
| `isEmptyOrWhitespace` | func | `(s): bool` | Is `s` empty or all whitespace? |

---

## Constants, types and helpers

### `Whitespace*: set[char]`
All characters counted as whitespace: space, tab, vertical tab, carriage return, line feed, form feed — i.e. `{' ', '\t', '\v', '\r', '\l', '\f'}`.

### `Letters*: set[char]`
The set of ASCII letters: `{'A'..'Z', 'a'..'z'}`.

### `UppercaseLetters*: set[char]`
`{'A'..'Z'}` — uppercase ASCII letters.

### `LowercaseLetters*: set[char]`
`{'a'..'z'}` — lowercase ASCII letters.

### `PunctuationChars*: set[char]`
All ASCII punctuation characters: `{'!'..'/', ':'..'@', '['..'`', '{'..'~'}`.

### `Digits*: set[char]`
`{'0'..'9'}`.

### `HexDigits*: set[char]`
`{'0'..'9', 'A'..'F', 'a'..'f'}`.

### `IdentChars*: set[char]`
Characters that may appear in a Nim identifier: `{'a'..'z', 'A'..'Z', '0'..'9', '_'}`.

### `IdentStartChars*: set[char]`
Characters an identifier may *start* with: `{'a'..'z', 'A'..'Z', '_'}`.

### `Newlines*: set[char]`
`{'\13', '\10'}` — carriage return and line feed, the characters a newline sequence can start with.

### `PrintableChars*: set[char]`
Union of `Letters + Digits + PunctuationChars + Whitespace`.

### `AllChars*: set[char]`
All 256 possible byte values, `{'\x00'..'\xFF'}`. Mostly useful for building *inverted* sets, e.g. `AllChars - Digits` to find the first non-digit character with `find`.

### `SkipTable* = array[char, int]`
Precomputed jump table used by the Boyer–Moore–Horspool substring search (see `initSkipTable` and the table-based `find` overload). Build it once with `initSkipTable(sub)` and reuse it to search the same substring `sub` repeatedly in different strings — this avoids recomputing the table on every call.

### `FloatFormatMode* = enum`
Controls how `formatFloat`/`formatBiggestFloat` render numbers:
- `ffDefault` — the shortest representation that round-trips (similar to `%g` in C).
- `ffDecimal` — fixed-point decimal notation, `precision` digits after the point.
- `ffScientific` — scientific notation with an `e` exponent, `precision` significant digits.

### `BinaryPrefixMode* = enum`
Controls which unit prefixes `formatSize` uses:
- `bpIEC` — binary IEC/ISO prefixes (`Ki`, `Mi`, `Gi`, … with base 1024), e.g. `"4KiB"`.
- `bpColloquial` — everyday SI-style names reused for base-1024 values (`k`, `M`, `G`, …), e.g. `"4kB"`.

### Module exports from `std/unicode`
`toLower` and `toUpper` (Unicode-aware case conversion) are re-exported from `std/unicode` for convenience, complementing the ASCII-only `toLowerAscii`/`toUpperAscii` defined here.

---

## Main reference

### Character classification

#### `isAlphaAscii(c: char): bool`
Returns `true` if `c` is an ASCII letter (`a-z` or `A-Z`). Only ASCII is checked — for full Unicode support use the `unicode` module.
```nim
doAssert isAlphaAscii('e') == true
doAssert isAlphaAscii('E') == true
doAssert isAlphaAscii('8') == false
```

#### `isAlphaNumeric(c: char): bool`
Returns `true` if `c` is an ASCII letter or digit (`a-z`, `A-Z`, `0-9`).
```nim
doAssert isAlphaNumeric('n') == true
doAssert isAlphaNumeric('8') == true
doAssert isAlphaNumeric(' ') == false
```

#### `isDigit(c: char): bool`
Returns `true` if `c` is one of `0-9`.
```nim
doAssert isDigit('n') == false
doAssert isDigit('8') == true
```

#### `isSpaceAscii(c: char): bool`
Returns `true` if `c` is one of the characters in `Whitespace`.
```nim
doAssert isSpaceAscii('n') == false
doAssert isSpaceAscii(' ') == true
doAssert isSpaceAscii('\t') == true
```

#### `isLowerAscii(c: char): bool`
Returns `true` if `c` is a lowercase ASCII letter. Only checks ASCII characters — use the `unicode` module for full Unicode support.
```nim
doAssert isLowerAscii('e') == true
doAssert isLowerAscii('E') == false
doAssert isLowerAscii('7') == false
```

#### `isUpperAscii(c: char): bool`
Returns `true` if `c` is an uppercase ASCII letter. Only checks ASCII characters.
```nim
doAssert isUpperAscii('e') == false
doAssert isUpperAscii('E') == true
doAssert isUpperAscii('7') == false
```

---

### Case conversion

#### `toLowerAscii(c: char): char`
Returns the lowercase version of `c`. Works only for `A-Z`; other characters are returned unchanged. For Unicode characters use `unicode.toLower`.
```nim
doAssert toLowerAscii('A') == 'a'
doAssert toLowerAscii('e') == 'e'
```
Implementation note: for an uppercase letter, the conversion is done by flipping bit 5 (`xor 0b0010_0000`), which is the standard ASCII trick for case conversion since uppercase and lowercase letters differ by exactly that bit.

#### `toLowerAscii(s: string): string`
Converts every character of `s` to lower case (ASCII `A-Z` only); other characters pass through unchanged.
```nim
doAssert toLowerAscii("FooBar!") == "foobar!"
```
See also: `normalize` — also removes underscores.

#### `toUpperAscii(c: char): char`
Returns the uppercase version of `c`. Works only for `a-z`. For Unicode characters use `unicode.toUpper`.
```nim
doAssert toUpperAscii('a') == 'A'
doAssert toUpperAscii('E') == 'E'
```

#### `toUpperAscii(s: string): string`
Converts every character of `s` to upper case (ASCII `a-z` only).
```nim
doAssert toUpperAscii("FooBar!") == "FOOBAR!"
```

#### `capitalizeAscii(s: string): string`
Converts only the **first** character of `s` to upper case (ASCII `a-z` only). An empty string returns an empty string. If the first character is not a letter, it is left unchanged.
```nim
doAssert capitalizeAscii("foo") == "Foo"
doAssert capitalizeAscii("-bar") == "-bar"
```

---

### Normalization and comparison

#### `nimIdentNormalize(s: string): string`
Normalizes `s` the way the Nim compiler normalizes identifiers for comparison: converts to lower case and removes all underscores **except** the first character of the string. The first character is preserved as-is (only case-folded).

> ⚠️ Backticks (`` ` ``) are *not* handled specially — they remain as-is, and spaces are preserved. For an alternative that handles backticks, see `nimIdentBackticksNormalize` in `dochelpers`.

```nim
doAssert nimIdentNormalize("Foo_bar") == "Foobar"
```

#### `normalize(s: string): string`
Converts `s` to lower case and removes **all** underscores (including the first character, unlike `nimIdentNormalize`). This must **not** be used to normalize Nim identifier names — use `nimIdentNormalize` for that.
```nim
doAssert normalize("Foo_bar") == "foobar"
doAssert normalize("Foo Bar") == "foo bar"
```

#### `cmpIgnoreCase(a, b: string): int`
Case-insensitive string comparison. Returns `0` if equal, a negative value if `a < b`, a positive value if `a > b` (same convention as `system.cmp`).
```nim
doAssert cmpIgnoreCase("FooBar", "foobar") == 0
doAssert cmpIgnoreCase("bar", "Foo") < 0
doAssert cmpIgnoreCase("Foo5", "foo4") > 0
```

#### `cmpIgnoreStyle(a, b: string): int`
Semantically equivalent to `cmp(normalize(a), normalize(b))`, but implemented without allocating temporary strings — i.e. it is "style-insensitive": case and underscores are ignored when comparing. This must **not** be used to compare Nim identifier names (which follow `nimIdentNormalize` rules, where the first character's underscore matters); use `macros.eqIdent` for identifiers.
```nim
doAssert cmpIgnoreStyle("foo_bar", "FooBar") == 0
doAssert cmpIgnoreStyle("foo_bar_5", "FooBar4") > 0
```

> Both comparison functions are compiled with bounds/overflow checks disabled (`{.push checks: off.}`) because they are hot paths used internally by the Nim compiler.

---

### Splitting

Nim provides both **iterator** and **func** (sequence-returning) variants for every split operation. The iterators are more memory-efficient for streaming/`for` loops; the func variants return a `seq[string]` and are convenient when you need the whole result at once (e.g. for indexing or `len`).

#### `split(s: string, sep: char, maxsplit: int = -1): string` *(iterator)*
Splits `s` on every occurrence of the character `sep`. Empty fields between consecutive separators (or at the start/end of the string) are preserved as empty strings.
```nim
for word in split(";;this;is;an;;example;;;", ';'):
  writeLine(stdout, word)
# ""
# ""
# "this"
# "is"
# "an"
# ""
# "example"
# ""
# ""
# ""
```
`maxsplit` limits the number of splits performed (default `-1` = unlimited); once the limit is reached, the rest of the string is yielded as the final piece.

See also: `rsplit` (split from the right), `splitLines`, `splitWhitespace`, and the `seq`-returning `split` func.

#### `split(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): string` *(iterator)*
Splits `s` wherever a maximal run of characters from `seps` occurs — i.e. a *group* of consecutive separator characters is treated as one delimiter.
```nim
for word in split("this\lis an\texample"):
  writeLine(stdout, word)
# "this"
# "is"
# "an"
# "example"

for word in split("this:is;an$example", {';', ':', '$'}):
  writeLine(stdout, word)
# same output as above

let date = "2012-11-20T22:08:08.398990"
let separators = {' ', '-', ':', 'T'}
for number in split(date, separators):
  writeLine(stdout, number)
# "2012"
# "11"
# "20"
# "22"
# "08"
# "08.398990"
```
> **Note:** an empty separator set returns the original string unchanged (interpreted as "split by no element").

#### `split(s: string, sep: string, maxsplit: int = -1): string` *(iterator)*
Splits `s` on every occurrence of the substring `sep` (the whole string `sep` acts as one delimiter).
```nim
for word in split("thisDATAisDATAcorrupted", "DATA"):
  writeLine(stdout, word)
# "this"
# "is"
# "corrupted"
```
> **Note:** an empty separator string `""` returns the original string unchanged.

#### `rsplit(s: string, sep: char | set[char] | string, maxsplit: int = -1[, keepSeparators: bool]): string` *(iterator)*
Behaves exactly like the corresponding `split` iterator, but processes the string from the **right** (end) towards the start. This only matters when `maxsplit` is set: with `rsplit`, the splits closest to the *end* of the string are performed first, so the leftover "unsplit" portion ends up at the **beginning**.
```nim
for piece in "foo:bar".rsplit(':'):
  echo piece
# "bar"
# "foo"

for piece in "foo bar".rsplit(Whitespace):
  echo piece
# "bar"
# "foo"

for piece in "foothebar".rsplit("the"):
  echo piece
# "bar"
# "foo"
```
The string-separator overload also accepts `keepSeparators: bool` (default `false`).
> **Note:** an empty separator (empty set or empty string) returns the original string unchanged.

#### `splitLines(s: string, keepEol = false): string` *(iterator)*
Splits `s` into its lines. All three newline conventions are recognized: `LF` (`\n`), `CR` (`\r`), and `CRLF` (`\r\n`) — each counts as **one** line terminator. By default the line terminator is stripped from each yielded line; pass `keepEol = true` to keep it attached.
```nim
for line in splitLines("\nthis\nis\nan\n\nexample\n"):
  writeLine(stdout, line)
# ""
# "this"
# "is"
# "an"
# ""
# "example"
# ""
```
See also: `splitWhitespace`, `countLines` (more efficient if you only need the count).

#### `splitWhitespace(s: string, maxsplit: int = -1): string` *(iterator)*
Splits `s` on runs of whitespace, **also stripping leading and trailing whitespace**. If `maxsplit` is positive, at most that many splits are performed, with any remaining text (including its surrounding whitespace) returned as the final piece verbatim.
```nim
let s = "  foo \t bar  baz  "
for ms in [-1, 1, 2, 3]:
  echo "------ maxsplit = ", ms, ":"
  for item in s.splitWhitespace(maxsplit=ms):
    echo '"', item, '"'
# ------ maxsplit = -1:
# "foo"
# "bar"
# "baz"
# ------ maxsplit = 1:
# "foo"
# "bar  baz  "
# ------ maxsplit = 2:
# "foo"
# "bar"
# "baz  "
# ------ maxsplit = 3:
# "foo"
# "bar"
# "baz"
```

#### `split(s: string, sep: char, maxsplit: int = -1): seq[string]`
The `seq`-returning equivalent of the `split` iterator over a single character.
```nim
doAssert "a,b,c".split(',') == @["a", "b", "c"]
doAssert "".split(' ') == @[""]
```

#### `split(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): seq[string]`
The `seq`-returning equivalent of the `split` iterator over a character set.
```nim
doAssert "a,b;c".split({',', ';'}) == @["a", "b", "c"]
doAssert "".split({' '}) == @[""]
doAssert "empty seps return unsplit s".split({}) == @["empty seps return unsplit s"]
```

#### `split(s: string, sep: string, maxsplit: int = -1): seq[string]`
The `seq`-returning equivalent of the `split` iterator over a string separator. This is the most commonly used overload for CSV-like parsing.
```nim
doAssert "a,b,c".split(",") == @["a", "b", "c"]
doAssert "a man a plan a canal panama".split("a ") == @["", "man ", "plan ", "canal panama"]
doAssert "".split("Elon Musk") == @[""]
doAssert "a  largely    spaced sentence".split(" ") == @["a", "", "largely", "", "", "", "spaced", "sentence"]
doAssert "a  largely    spaced sentence".split(" ", maxsplit = 1) == @["a", " largely    spaced sentence"]
doAssert "empty sep returns unsplit s".split("") == @["empty sep returns unsplit s"]
```
Note how splitting on a literal `" "` keeps empty strings for consecutive spaces — if you want to ignore repeated whitespace, use `splitWhitespace` instead.

#### `rsplit(s: string, sep: char, maxsplit: int = -1): seq[string]`
The `seq`-returning equivalent of `rsplit` over a single character, with results in the **original left-to-right order** (internally collected in reverse and then reversed back).

A common use case is path-like manipulation where you want to peel off the last component(s):
```nim
var tailSplit = rsplit("Root#Object#Method#Index", '#', maxsplit=1)
# tailSplit == @["Root#Object#Method", "Index"]
```

#### `rsplit(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): seq[string]`
The `seq`-returning equivalent of `rsplit` over a character set, in original order.
```nim
var tailSplit = rsplit("Root#Object#Method#Index", {'#'}, maxsplit=1)
# tailSplit == @["Root#Object#Method", "Index"]
```
> **Note:** an empty separator set returns the original string unchanged.

#### `rsplit(s: string, sep: string, maxsplit: int = -1): seq[string]`
The `seq`-returning equivalent of `rsplit` over a string separator, in original order.
```nim
doAssert "a  largely    spaced sentence".rsplit(" ", maxsplit = 1) == @["a  largely    spaced", "sentence"]
doAssert "a,b,c".rsplit(",") == @["a", "b", "c"]
doAssert "a man a plan a canal panama".rsplit("a ") == @["", "man ", "plan ", "canal panama"]
doAssert "".rsplit("Elon Musk") == @[""]
doAssert "a  largely    spaced sentence".rsplit(" ") == @["a", "", "largely", "", "", "", "spaced", "sentence"]
doAssert "empty sep returns unsplit s".rsplit("") == @["empty sep returns unsplit s"]
```
> **Note:** an empty separator string returns the original string unchanged.

#### `splitLines(s: string, keepEol = false): seq[string]`
The `seq`-returning equivalent of the `splitLines` iterator.

#### `splitWhitespace(s: string, maxsplit: int = -1): seq[string]`
The `seq`-returning equivalent of the `splitWhitespace` iterator.

---

### Tokenizing

#### `tokenize(s: string, seps: set[char] = Whitespace): tuple[token: string, isSep: bool]` *(iterator)*
Tokenizes `s` into alternating substrings of separator-characters and non-separator characters, yielding `(token, isSep)` pairs. Unlike `split`, **both** the separator runs and the content runs are yielded — nothing is discarded.
```nim
for word in tokenize("  this is an  example  "):
  writeLine(stdout, word)
# ("  ", true)
# ("this", false)
# (" ", true)
# ("is", false)
# (" ", true)
# ("an", false)
# ("  ", true)
# ("example", false)
# ("  ", true)
```
Useful when you need to reconstruct or rewrite a string while preserving the exact original whitespace/separator layout.

---

### Joining

#### `join(a: openArray[string], sep: string = ""): string`
Concatenates all strings in `a`, inserting `sep` between consecutive elements (but not before the first or after the last).
```nim
doAssert join(["A", "B", "Conclusion"], " -> ") == "A -> B -> Conclusion"
```
The implementation pre-computes the total required capacity and allocates the result string once, making it efficient even for large arrays.

#### `join[T: not string](a: openArray[T], sep: string = ""): string`
Generic overload for arrays of any non-string type `T`: converts each element to a string via `$` and joins them with `sep`.
```nim
doAssert join([1, 2, 3], " -> ") == "1 -> 2 -> 3"
```

---

### Number ↔ string conversions

#### `toBin(x: BiggestInt, len: Positive): string`
Converts `x` to its binary representation as a string **exactly** `len` characters long (zero-padded on the left, or truncated to the lowest `len` bits if `x` doesn't fit). No `0b` prefix is added.
```nim
let
  a = 29
  b = 257
doAssert a.toBin(8) == "00011101"
doAssert b.toBin(8) == "00000001"   # truncated — 257 doesn't fit in 8 bits
doAssert b.toBin(9) == "100000001"
```

#### `toOct(x: BiggestInt, len: Positive): string`
Converts `x` to its octal representation, exactly `len` characters long, with no `0o` prefix. Do not confuse with `toOctal(c: char)`, which converts a single *character's ordinal value*.
```nim
let
  a = 62
  b = 513
doAssert a.toOct(3) == "076"
doAssert b.toOct(3) == "001"   # truncated to 3 octal digits
doAssert b.toOct(5) == "01001"
```

#### `toHex[T: SomeInteger](x: T, len: Positive): string`
Converts `x` to its hexadecimal representation, exactly `len` characters long. `x` is treated as **unsigned** (so a negative number is shown via its two's-complement bit pattern). No `0x` prefix is added.
```nim
let
  a = 62'u64
  b = 4097'u64
doAssert a.toHex(3) == "03E"
doAssert b.toHex(3) == "001"     # truncated to 3 hex digits
doAssert b.toHex(4) == "1001"
doAssert toHex(62, 3) == "03E"
doAssert toHex(-8, 6) == "FFFFF8" # two's-complement representation
```

#### `toHex[T: SomeInteger](x: T): string`
Shortcut for `toHex(x, T.sizeof * 2)` — i.e. produces a string whose length matches the full bit-width of `T` in hex digits (2 hex digits per byte).
```nim
doAssert toHex(1984'i64) == "00000000000007C0"
doAssert toHex(1984'i16) == "07C0"
```

#### `toHex(s: string): string`
Converts a byte/string value to its hexadecimal representation. The output is exactly twice as long as the input. No `0x` prefix.
```nim
let
  a = "1"
  b = "A"
  c = "\0\255"
doAssert a.toHex() == "31"
doAssert b.toHex() == "41"
doAssert c.toHex() == "00FF"
```
See also: `parseHexStr` for the inverse operation (hex text → bytes).

#### `toOctal(c: char): string`
Converts the ordinal value of character `c` to its octal representation, always exactly 3 characters long (possibly with leading zeros, no `0o` prefix). Not to be confused with `toOct(x: BiggestInt, len)`.
```nim
doAssert toOctal('1') == "061"
doAssert toOctal('A') == "101"
doAssert toOctal('a') == "141"
doAssert toOctal('!') == "041"
```

#### `fromBin[T: SomeInteger](s: string): T`
Parses a binary integer literal from `s` into type `T`. Accepts optional `0b`/`0B` prefix; underscores in `s` are ignored (so digit-grouping like `0b_0100_1000` is allowed). Raises `ValueError` if `s` is not a valid binary literal.
**Does not check for overflow** — if the value is too large for `T`, only the rightmost bits that fit are kept (silent truncation/wraparound).
```nim
let s = "0b_0100_1000_1000_1000_1110_1110_1001_1001"
doAssert fromBin[int](s) == 1216933529
doAssert fromBin[int8](s) == 0b1001_1001'i8
doAssert fromBin[int8](s) == -103'i8        # wraps as signed 8-bit
doAssert fromBin[uint8](s) == 153
doAssert s.fromBin[:int16] == 0b1110_1110_1001_1001'i16
doAssert s.fromBin[:uint64] == 1216933529'u64
```

#### `fromOct[T: SomeInteger](s: string): T`
Parses an octal integer literal from `s` into type `T`. Accepts optional `0o`/`0O` prefix; underscores are ignored. Raises `ValueError` for an invalid literal. **No overflow checking** — truncates to the rightmost digits that fit `T`.
```nim
let s = "0o_123_456_777"
doAssert fromOct[int](s) == 21913087
doAssert fromOct[int8](s) == 0o377'i8
doAssert fromOct[int8](s) == -1'i8
doAssert fromOct[uint8](s) == 255'u8
doAssert s.fromOct[:int16] == 24063'i16
doAssert s.fromOct[:uint64] == 21913087'u64
```

#### `fromHex[T: SomeInteger](s: string): T`
Parses a hexadecimal integer literal from `s` into type `T`. Accepts optional `0x`, `0X`, or `#` prefix; underscores are ignored. Raises `ValueError` for an invalid literal. **No overflow checking** — truncates to the rightmost digits that fit `T`.
```nim
let s = "0x_1235_8df6"
doAssert fromHex[int](s) == 305499638
doAssert fromHex[int8](s) == 0xf6'i8
doAssert fromHex[int8](s) == -10'i8
doAssert fromHex[uint8](s) == 246'u8
doAssert s.fromHex[:int16] == -29194'i16
doAssert s.fromHex[:uint64] == 305499638'u64
```

#### `intToStr(x: int, minchars: Positive = 1): string`
Converts `x` to its decimal string representation, left-padded with `'0'` characters until the result is at least `minchars` long. The minus sign (for negative numbers) is added *after* the zero-padding of the absolute value.
```nim
doAssert intToStr(1984) == "1984"
doAssert intToStr(1984, 6) == "001984"
```

#### `insertSep(s: string, sep = '_', digits = 3): string`
Inserts the separator character `sep` every `digits` characters, counting from the **right** end of `s` (i.e. like grouping the digits of a number into thousands). Although it works on any string, it is intended for strings that represent numbers — an optional non-digit prefix (e.g. a `-` sign) is preserved and not counted toward the grouping.
```nim
doAssert insertSep("1000000") == "1_000_000"
```

---

### Parsing

#### `parseInt(s: string): int`
Parses a decimal integer from `s`. The **entire** string must be a valid integer (including an optional sign), or `ValueError` is raised.
```nim
doAssert parseInt("-0042") == -42
```

#### `parseBiggestInt(s: string): BiggestInt`
Same as `parseInt`, but returns `BiggestInt` (the largest signed integer type), for values that may not fit in a plain `int`.

#### `parseUInt(s: string): uint`
Parses a decimal **unsigned** integer from `s`. The entire string must be valid, or `ValueError` is raised.

#### `parseBiggestUInt(s: string): BiggestUInt`
Same as `parseUInt`, but returns `BiggestUInt`.

#### `parseFloat(s: string): float`
Parses a decimal floating-point value from `s`. The entire string must be a valid float, or `ValueError` is raised. The special values `nan`, `inf`, `-inf` are recognized case-insensitively.
```nim
doAssert parseFloat("3.14") == 3.14
doAssert parseFloat("inf") == 1.0/0
```

#### `parseBinInt(s: string): int`
Parses a binary integer from `s`. Optional `0b`/`0B` prefix; underscores are ignored. Raises `ValueError` if invalid or empty.
```nim
let
  a = "0b11_0101"
  b = "111"
doAssert a.parseBinInt() == 53
doAssert b.parseBinInt() == 7
```

#### `parseOctInt(s: string): int`
Parses an octal integer from `s`. Optional `0o`/`0O` prefix; underscores are ignored. Raises `ValueError` if invalid or empty.

#### `parseHexInt(s: string): int`
Parses a hexadecimal integer from `s`. Optional `0x`, `0X`, or `#` prefix; underscores are ignored. Raises `ValueError` if invalid or empty.

#### `parseHexStr(s: string): string`
Converts a hex-encoded string (each byte represented by exactly two hex digits, case-insensitive) into the corresponding raw byte string. Raises `ValueError` if `s` has odd length or contains an invalid hex digit.
```nim
let
  a = "41"
  b = "3161"
  c = "00ff"
doAssert parseHexStr(a) == "A"
doAssert parseHexStr(b) == "1a"
doAssert parseHexStr(c) == "\0\255"
```
See also: `toHex(s: string)` for the reverse operation.

#### `parseBool(s: string): bool`
Parses a "truthy"/"falsy" textual value (case-insensitively, and with underscores ignored via `normalize`):
- `true` for: `y`, `yes`, `true`, `1`, `on`.
- `false` for: `n`, `no`, `false`, `0`, `off`.
- Anything else raises `ValueError`.
```nim
let a = "n"
doAssert parseBool(a) == false
```

#### `parseEnum[T: enum](s: string): T`
Parses `s` into an enum value of type `T`, comparing names in a *style-insensitive* way (case-insensitive and underscore-insensitive, except the first letter remains case-sensitive — same rules as `cmpIgnoreStyle`/`nimIdentNormalize`). Raises `ValueError` if `s` doesn't match any enum field. This is a compile-time error if `T` has multiple fields that normalize to the same string.
```nim
type
  MyEnum = enum
    first = "1st",
    second,
    third = "3rd"

doAssert parseEnum[MyEnum]("1_st") == first
doAssert parseEnum[MyEnum]("second") == second
doAssertRaises(ValueError):
  echo parseEnum[MyEnum]("third")
```

#### `parseEnum[T: enum](s: string, default: T): T`
Same as above, but returns `default` instead of raising `ValueError` when `s` doesn't match any field.
```nim
type
  MyEnum = enum
    first = "1st",
    second,
    third = "3rd"

doAssert parseEnum[MyEnum]("1_st") == first
doAssert parseEnum[MyEnum]("second") == second
doAssert parseEnum[MyEnum]("last", third) == third
```

---

### Repeating, padding and alignment

#### `repeat(c: char, count: Natural): string`
Returns a string of `count` copies of the character `c`.
```nim
let a = 'z'
doAssert a.repeat(5) == "zzzzz"
```

#### `repeat(s: string, n: Natural): string`
Returns `s` concatenated with itself `n` times (i.e. `n` copies of `s` back-to-back, no separator).
```nim
doAssert "+ foo +".repeat(3) == "+ foo ++ foo ++ foo +"
```

#### `spaces(n: Natural): string`
Returns a string consisting of `n` space characters. Implemented as `repeat(' ', n)`. Handy for left-aligning text manually.
```nim
let
  width = 15
  text1 = "Hello user!"
  text2 = "This is a very long string"
doAssert text1 & spaces(max(0, width - text1.len)) & "|" == "Hello user!    |"
doAssert text2 & spaces(max(0, width - text2.len)) & "|" == "This is a very long string|"
```
See also: `align`, `alignLeft`, `indent`, `center`.

#### `align(s: string, count: Natural, padding = ' '): string`
**Right-aligns** `s` by inserting `padding` characters *before* it until the total length is `count`. If `s.len >= count`, `s` is returned unchanged (never truncated). For left alignment, use `alignLeft`.
```nim
assert align("abc", 4) == " abc"
assert align("a", 0) == "a"
assert align("1232", 6) == "  1232"
assert align("1232", 6, '#') == "##1232"
```

#### `alignLeft(s: string, count: Natural, padding = ' '): string`
**Left-aligns** `s` by appending `padding` characters *after* it until the total length is `count`. If `s.len >= count`, `s` is returned unchanged. For right alignment, use `align`.
```nim
assert alignLeft("abc", 4) == "abc "
assert alignLeft("a", 0) == "a"
assert alignLeft("1232", 6) == "1232  "
assert alignLeft("1232", 6, '#') == "1232##"
```

#### `center(s: string, width: int, fillChar: char = ' '): string`
Centers `s` within a string of length `width`, using `fillChar` for padding on both sides. If the total padding needed is odd, the **extra** padding character goes on the right (i.e. left padding = `(width - s.len) div 2`). If `width <= s.len`, `s` is returned unchanged.
```nim
let a = "foo"
doAssert a.center(2) == "foo"
doAssert a.center(5) == " foo "
doAssert a.center(6) == " foo  "
```

---

### Indentation

#### `indent(s: string, count: Natural, padding: string = " "): string`
Prepends `count` copies of `padding` to **every line** of `s` (lines are obtained via `splitLines`, then rejoined with `\n`).
> ⚠️ This does **not** preserve the original newline characters of `s` — all line breaks in the result are `\n`, regardless of whether the input used `\r\n`, `\r`, or `\n`.
```nim
doAssert indent("First line\c\l and second line.", 2) == "  First line\l   and second line."
```

#### `unindent(s: string, count: Natural = int.high, padding: string = " "): string`
Removes up to `count` occurrences of `padding` from the start of **every line** of `s`. If a line has fewer than `count` leading copies of `padding`, only as many as are actually present are removed. The default `count = int.high` means "remove all leading `padding` from every line".
> ⚠️ Like `indent`, this does not preserve the original newline characters — the result is joined with `\n`.
```nim
let x = """
  Hello
    There
""".unindent()

doAssert x == "Hello\nThere\n"
```
See also: `dedent` for removing only the indentation *common* to all lines.

#### `indentation(s: string): Natural`
Returns the number of leading space characters that **all** non-blank lines of `s` have in common (lines consisting only of whitespace are ignored when computing the minimum). Returns `0` if `s` is empty or has no non-blank lines. *(Available since Nim 1.3.)*

#### `dedent(s: string, count: Natural = indentation(s)): string`
Like `unindent`, but the default `count` is the value returned by `indentation(s)` — i.e. by default it strips only the indentation that is **common to every line**, preserving any *additional* (relative) indentation. Only supports space (`" "`) as padding.
> ⚠️ Does not preserve original newline characters (result uses `\n`). *(Available since Nim 1.3.)*
```nim
let x = """
  Hello
    There
""".dedent()

doAssert x == "Hello\n  There\n"
```
Compare with the `unindent` example above: `dedent` keeps the extra two-space indentation of `"There"` relative to `"Hello"`, whereas `unindent()` (with its default of removing *all* leading spaces) strips it entirely.

---

### Deleting and trimming

#### `delete(s: var string, slice: Slice[int])`
Removes the characters at indices `s[slice]` **in place**, shifting subsequent characters left. Raises `IndexDefect` if the slice contains out-of-range indices (when bound checks are enabled). This is the string analog of `sequtils.delete` for sequences.
```nim
var a = "abcde"
doAssertRaises(IndexDefect): a.delete(4..5)
assert a == "abcde"
a.delete(4..4)
assert a == "abcd"
a.delete(1..2)
assert a == "ad"
a.delete(1..<1)  # empty slice — no-op
assert a == "ad"
```

#### `delete(s: var string, first, last: int)` — **deprecated**
Older two-integer-argument form of `delete`. Deletes characters at positions `first..last` (inclusive). **Deprecated** — use `delete(s, first..last)` (the `Slice[int]` overload) instead. Unlike the slice version, this form clamps `last` to `s.len` rather than raising on out-of-range values.
```nim
var a = "abracadabra"
a.delete(4, 5)
doAssert a == "abradabra"
a.delete(1, 6)
doAssert a == "ara"
a.delete(2, 999)
doAssert a == "ar"
```

#### `strip(s: string, leading = true, trailing = true, chars: set[char] = Whitespace): string`
Returns a copy of `s` with leading and/or trailing characters from `chars` removed (default `chars` is `Whitespace`). Control which ends are trimmed with `leading`/`trailing`; if both are `false`, `s` is returned unchanged.
```nim
let a = "  vhellov   "
let b = strip(a)
doAssert b == "vhellov"

doAssert a.strip(leading = false) == "  vhellov"
doAssert a.strip(trailing = false) == "vhellov   "

doAssert b.strip(chars = {'v'}) == "hello"
doAssert b.strip(leading = false, chars = {'v'}) == "vhello"

let c = "blaXbla"
doAssert c.strip(chars = {'b', 'a'}) == "laXbl"
doAssert c.strip(chars = {'b', 'a', 'l'}) == "X"
```
See also: `strbasics.strip` for an in-place version; `stripLineEnd` for removing just a trailing newline.

#### `stripLineEnd(s: var string)`
Removes **one** trailing line-ending sequence from `s` in place: `\r\n`, `\n`, `\r`, `\v`, or `\f` (at most one such instance, even if `s` ends with several newlines). Also known as "chomp". Useful after reading a line via `osproc.execCmdEx` or similar APIs that may leave a trailing newline.
```nim
var s = "foo\n\n"
s.stripLineEnd
doAssert s == "foo\n"   # only ONE trailing \n removed
s = "foo\r\n"
s.stripLineEnd
doAssert s == "foo"     # \r\n counts as a single sequence
```

---

### Prefix / suffix operations

#### `startsWith(s: string, prefix: char): bool`
Returns `true` if `s` is non-empty and its first character equals `prefix`.
```nim
let a = "abracadabra"
doAssert a.startsWith('a') == true
doAssert a.startsWith('b') == false
```

#### `startsWith(s, prefix: string): bool`
Returns `true` if `s` starts with the string `prefix`. An empty `prefix` always returns `true`.
```nim
let a = "abracadabra"
doAssert a.startsWith("abra") == true
doAssert a.startsWith("bra") == false
```

#### `endsWith(s: string, suffix: char): bool`
Returns `true` if `s` is non-empty and its last character equals `suffix`.
```nim
let a = "abracadabra"
doAssert a.endsWith('a') == true
doAssert a.endsWith('b') == false
```

#### `endsWith(s, suffix: string): bool`
Returns `true` if `s` ends with the string `suffix`. An empty `suffix` always returns `true`.
```nim
let a = "abracadabra"
doAssert a.endsWith("abra") == true
doAssert a.endsWith("dab") == false
```

#### `continuesWith(s, substr: string, start: Natural): bool`
Returns `true` if, starting at index `start` in `s`, the following characters match `substr` exactly (i.e. `s[start ..< start+substr.len] == substr`). An empty `substr` always returns `true`.
```nim
let a = "abracadabra"
doAssert a.continuesWith("ca", 4) == true
doAssert a.continuesWith("ca", 5) == false
doAssert a.continuesWith("dab", 6) == true
```

#### `removePrefix(s: var string, chars: set[char] = Newlines)`
Removes, **in place**, all leading characters of `s` that belong to `chars` (default `Newlines`). Repeated/different characters from the set are all stripped until a character outside the set is found.
```nim
var userInput = "\r\n*~Hello World!"
userInput.removePrefix
doAssert userInput == "*~Hello World!"
userInput.removePrefix({'~', '*'})
doAssert userInput == "Hello World!"

var otherInput = "?!?Hello!?!"
otherInput.removePrefix({'!', '?'})
doAssert otherInput == "Hello!?!"
```

#### `removePrefix(s: var string, c: char)`
Removes, in place, all leading occurrences of the single character `c`. Equivalent to `removePrefix(s, chars = {c})`.
```nim
var ident = "pControl"
ident.removePrefix('p')
doAssert ident == "Control"
```

#### `removePrefix(s: var string, prefix: string)`
Removes, in place, **one** occurrence of `prefix` from the start of `s`, but only if `s` actually starts with `prefix` (and `prefix` is non-empty). Unlike the `chars`/`char` overloads, this does **not** repeat — only the first match is removed.
```nim
var answers = "yesyes"
answers.removePrefix("yes")
doAssert answers == "yes"
```

#### `removeSuffix(s: var string, chars: set[char] = Newlines)`
Removes, in place, all trailing characters of `s` that belong to `chars` (default `Newlines`).
```nim
var userInput = "Hello World!*~\r\n"
userInput.removeSuffix
doAssert userInput == "Hello World!*~"
userInput.removeSuffix({'~', '*'})
doAssert userInput == "Hello World!"

var otherInput = "Hello!?!"
otherInput.removeSuffix({'!', '?'})
doAssert otherInput == "Hello"
```

#### `removeSuffix(s: var string, c: char)`
Removes, in place, all trailing occurrences of the single character `c`. Equivalent to `removeSuffix(s, chars = {c})`.
```nim
var table = "users"
table.removeSuffix('s')
doAssert table == "user"

var dots = "Trailing dots......."
dots.removeSuffix('.')
doAssert dots == "Trailing dots"
```

#### `removeSuffix(s: var string, suffix: string)`
Removes, in place, **one** occurrence of `suffix` from the end of `s`, but only if `s` actually ends with `suffix`. Does not repeat.
```nim
var answers = "yeses"
answers.removeSuffix("es")
doAssert answers == "yes"
```

---

### Searching

#### `initSkipTable(a: var SkipTable, sub: string)`
Initializes an existing `SkipTable` `a` for searching the substring `sub`, using the Boyer–Moore–Horspool preprocessing step. Use this (or `initSkipTable(sub)`) once, then reuse the table across multiple `find(a, s, sub, ...)` calls against the same `sub` for better performance.

#### `initSkipTable(sub: string): SkipTable`
Returns a new, freshly initialized `SkipTable` for `sub`. Convenience wrapper over the `var`-parameter overload.

#### `find(a: SkipTable, s, sub: string, start: Natural = 0, last = -1): int`
Searches for `sub` inside `s[start..last]` using a precomputed `SkipTable` (Boyer–Moore–Horspool algorithm). If `last < 0`, it defaults to `s.high`. Returns the index of the first match (relative to `s[0]`), or `-1` if `sub` is not found. An empty `sub` matches immediately at `start`. Searching is case-sensitive.

#### `find(s: string, sub: char, start: Natural = 0, last = -1): int`
Searches for the character `sub` in `s[start..last]`. If `last` is unspecified or negative, defaults to `s.high`. Returns the index of the first match relative to `s[0]` (subtract `start` yourself if you need a `start`-relative index), or `-1` if not found. Case-sensitive. On native targets this uses the C library's `memchr` for speed when possible.
See also: `rfind`, `replace(s, sub: char, by: char)`.

#### `find(s: string, chars: set[char], start: Natural = 0, last = -1): int`
Searches for the first character in `s[start..last]` that belongs to the set `chars`. Returns its index (relative to `s[0]`), or `-1` if none of `s[start..last]` is in `chars`.
See also: `rfind`, `multiReplace`. A common trick is `find(s, AllChars - Digits)` to find the first *non-digit* character.

#### `find(s, sub: string, start: Natural = 0, last = -1): int`
Searches for the substring `sub` in `s[start..last]`. If `last` is unspecified or negative, defaults to `s.high`. Returns the index of the first match relative to `s[0]`, or `-1` if not found. Case-sensitive. Internally dispatches to single-character search for length-1 `sub`, to the platform's `memmem` when available (Linux/BSD/macOS, and only when `last < 0`), or otherwise to the Boyer–Moore–Horspool `SkipTable` search.
See also: `rfind`, `replace(s, sub: string, by: string)`.

#### `rfind(s: string, sub: char, start: Natural = 0, last = -1): int`
Searches for `sub` in `s[start..last]`, but scans from the **end** towards `start` (i.e. finds the *last* occurrence). If `last` is unspecified, defaults to `s.high`. Returns the index relative to `s[0]`, or `-1` if not found.

#### `rfind(s: string, chars: set[char], start: Natural = 0, last = -1): int`
Like `find(s, chars, ...)`, but scans from the end towards `start`, returning the index of the *last* matching character, or `-1` if none found.

#### `rfind(s, sub: string, start: Natural = 0, last = -1): int`
Like `find(s, sub, ...)`, but scans from the end towards `start`, returning the index of the *last* occurrence of `sub`, or `-1` if not found. For an empty `sub`, returns `max(start, last)` (or `max(start, s.len)` if `last < 0`) — i.e. the rightmost valid position.

---

### Counting and membership

#### `count(s: string, sub: char): int`
Counts how many times the character `sub` occurs in `s`.

#### `count(s: string, subs: set[char]): int`
Counts how many characters of `s` belong to the set `subs`. `subs` must be non-empty (checked via `doAssert`).

#### `count(s: string, sub: string, overlapping: bool = false): int`
Counts non-overlapping occurrences of the substring `sub` in `s` by default. If `overlapping = true`, occurrences may overlap (the search advances by 1 character after each match instead of by `sub.len`). `sub` must be non-empty (checked via `doAssert`).

#### `countLines(s: string): int`
Returns the number of lines in `s`. Equivalent to `len(splitLines(s))` but much more efficient, since it doesn't build any intermediate strings/sequences — it just scans `s` once counting newline sequences (`\r`, `\n`, `\r\n`, each counted once) and adds 1. A line may be empty.
```nim
doAssert countLines("First line\l and second line.") == 2
```

#### `contains(s, sub: string): bool`
Returns `true` if `sub` occurs anywhere in `s`. Equivalent to `find(s, sub) >= 0`. This is the proc behind Nim's `in`/`notin` operators for strings (e.g. `"sub" in s`).

#### `contains(s: string, chars: set[char]): bool`
Returns `true` if any character of `s` belongs to `chars`. Equivalent to `find(s, chars) >= 0`.

#### `allCharsInSet(s: string, theSet: set[char]): bool`
Returns `true` if **every** character of `s` belongs to `theSet`. An empty string returns `true` (vacuously true).
```nim
doAssert allCharsInSet("aeea", {'a', 'e'}) == true
doAssert allCharsInSet("", {'a', 'e'}) == true
```

#### `isEmptyOrWhitespace(s: string): bool`
Returns `true` if `s` is empty or consists entirely of characters from `Whitespace`. Implemented via `allCharsInSet(s, Whitespace)`.

---

### Replacing

#### `replace(s, sub: string, by = ""): string`
Returns a copy of `s` with **every** occurrence of `sub` replaced by `by`. If `sub` is empty, `s` is returned unchanged. Single-character `sub` uses a fast char-search path; longer `sub` uses the Boyer–Moore–Horspool `SkipTable`.
See also: `find`, `replace(s, sub, by: char)` for single characters, `replaceWord`, `multiReplace` for several substrings/characters at once.

#### `replace(s: string, sub, by: char): string`
Returns a copy of `s` with every occurrence of the character `sub` replaced by the character `by`. An optimized special case of the string-based `replace` for single characters (result has the same length as `s`).

#### `replaceWord(s, sub: string, by = ""): string`
Like `replace`, but only replaces occurrences of `sub` that are surrounded by **word boundaries** — i.e. the character immediately before and immediately after the match (if any) must *not* be a "word character" (`'a'..'z'`, `'A'..'Z'`, `'0'..'9'`, `'_'`, or any byte `'\128'..'\255'`). This is comparable to `\b` in regular expressions. If `sub` is empty, `s` is returned unchanged.

#### `multiReplace(s: string, replacements: varargs[(string, string)]): string`
Performs multiple substring replacements in a **single left-to-right pass** over `s`, which is more efficient than chaining several `replace` calls. Rules:
- At each position, the **first matching** replacement pair (in argument order) is applied if multiple could match.
- After a replacement is applied, scanning resumes **after** the matched (original) text — overlaps with the *replacement text* are not considered, and substrings spanning a replacement boundary are not matched.
- If the result is no longer than the input, only a single memory allocation is needed.
```nim
# Swapping occurrences of 'a' and 'b':
doAssert multireplace("abba", [("a", "b"), ("b", "a")]) == "baab"

# The second replacement ("ab") is matched and performed first, the scan then
# continues from 'c', so the "bc" replacement is never matched and thus skipped.
doAssert multireplace("abc", [("bc", "x"), ("ab", "_b")]) == "_bc"
```

#### `multiReplace(s: openArray[char], replacements: varargs[(set[char], char)]): string`
Single-pass **character**-level replacement: for each character of `s`, the first `(set[char], char)` rule whose set contains that character determines its replacement; if no rule matches, the character is kept as-is. Useful for sanitizing strings (e.g. stripping characters that are illegal in filenames).
```nim
const WinSanitationRules = [
  ({'\0'..'\31'}, ' '),
  ({'"'}, '\''),
  ({'/', '\\', ':', '|'}, '-'),
  ({'*', '?', '<', '>'}, '_'),
]
const file = "a/file:with?invalid*chars.txt"
doAssert file.multiReplace(WinSanitationRules) == "a-file-with_invalid_chars.txt"
```

---

### Escaping

#### `escape(s: string, prefix = "\"", suffix = "\""): string`
Returns an escaped representation of `s`, wrapped in `prefix`/`suffix` (both default to `"` so the result looks like a Nim string literal). The escaping rules:
- Bytes `'\0'..'\31'` and `'\127'..'\255'` become `\xHH` (two-digit uppercase hex).
- `\` becomes `\\`.
- `'` becomes `\'`.
- `"` becomes `\"`.
- All other characters are copied as-is.

> Note: this escaping scheme is **different** from `system.addEscapedChar` (which produces Nim-source-compatible escapes like `\n`, `\t`, etc.) — `escape` always uses `\xHH` for non-printable/high bytes.

See also: `unescape` for the inverse operation.

#### `unescape(s: string, prefix = "\"", suffix = "\""): string`
Reverses `escape`: removes `prefix` from the start and `suffix` from the end of `s` (raising `ValueError` if they are not present), and converts `\xHH`, `\\`, `\'`, `\"` sequences back to their original bytes. Any other `\X` sequence is passed through literally as `\X`.

---

### Identifier validation

#### `validIdentifier(s: string): bool`
Returns `true` if `s` is a syntactically valid identifier: the first character must be in `IdentStartChars` (a letter or `_`), and every subsequent character must be in `IdentChars` (letters, digits, or `_`). An empty string is not a valid identifier.
```nim
doAssert "abc_def08".validIdentifier
```

---

### Abbreviation matching

#### `abbrev(s: string, possibilities: openArray[string]): int`
Finds the index of the entry in `possibilities` for which `s` is an unambiguous prefix:
- Returns the index of the matching item if exactly one item starts with `s`.
- Returns `-1` if **no** item starts with `s`.
- Returns `-2` if **multiple** items start with `s` (ambiguous) — *unless* `s` is itself an exact match for one of them, in which case that exact match wins regardless of other prefix matches.
```nim
doAssert abbrev("fac", ["college", "faculty", "industry"]) == 1
doAssert abbrev("foo", ["college", "faculty", "industry"]) == -1 # Not found
doAssert abbrev("fac", ["college", "faculty", "faculties"]) == -2 # Ambiguous
doAssert abbrev("college", ["college", "colleges", "industry"]) == 0
```

---

### Other helpers

#### `addSep(dest: var string, sep = ", ", startLen: Natural = 0)`
Appends `sep` to `dest` **only if** `dest.len > startLen`. Shorthand for:
```nim
if dest.len > startLen: add(dest, sep)
```
Designed for building delimiter-separated output incrementally, where the separator must be skipped before the very first element.
```nim
var arr = "["
for x in items([2, 3, 5, 7, 11]):
  addSep(arr, startLen = len("["))
  add(arr, $x)
add(arr, "]")
doAssert arr == "[2, 3, 5, 7, 11]"
```

---

### Floating point formatting

#### `formatBiggestFloat(f: BiggestFloat, format: FloatFormatMode = ffDefault, precision: range[-1..32] = 16, decimalSep = '.'): string`
Converts `f` to a string according to `format`:
- `ffDecimal` — `precision` is the number of digits **after the decimal point**.
- `ffScientific` — `precision` is the maximum number of **significant digits**.
- `ffDefault` — uses the shortest representation that round-trips.

The default `precision = 16` is the maximum number of meaningful digits after the decimal point for `BiggestFloat`. Pass `precision = -1` to let the implementation choose a "nice" formatting. `decimalSep` controls which character is used for the decimal point (useful for locales that use `,`).
```nim
let x = 123.456
doAssert x.formatBiggestFloat() == "123.4560000000000"
doAssert x.formatBiggestFloat(ffDecimal, 4) == "123.4560"
doAssert x.formatBiggestFloat(ffScientific, 2) == "1.23e+02"
```

#### `formatFloat(f: float, format: FloatFormatMode = ffDefault, precision: range[-1..32] = 16, decimalSep = '.'): string`
Same as `formatBiggestFloat`, but for the plain `float` type (and the default precision documentation refers to `float`'s meaningful digits). In practice it simply forwards to `formatBiggestFloat`.
```nim
let x = 123.456
doAssert x.formatFloat() == "123.4560000000000"
doAssert x.formatFloat(ffDecimal, 4) == "123.4560"
doAssert x.formatFloat(ffScientific, 2) == "1.23e+02"
```

#### `trimZeros(x: var string, decimalSep = '.')`
Removes trailing zeros from a formatted floating-point string `x`, **in place**. Operates only on the fractional part (after `decimalSep`); if there's an exponent (`e`), trailing zeros are trimmed only up to the exponent marker. If trimming would leave nothing after `decimalSep`, the separator itself is also removed.
```nim
var x = "123.456000000"
x.trimZeros()
doAssert x == "123.456"
```

---

### Size and engineering notation formatting

#### `formatSize(bytes: int64, decimalSep = '.', prefix = bpIEC, includeSpace = false): string`
Formats `bytes` as a human-readable size with a binary unit suffix, rounded to 3 significant decimal digits (with trailing zeros trimmed). By default uses IEC/ISO binary prefixes (`Ki`, `Mi`, `Gi`, … — powers of 1024 with the "i" suffix to indicate binary, not decimal). Pass `prefix = bpColloquial` to instead use everyday names (`k`, `M`, `G`, …) for the same 1024-based magnitudes. Set `includeSpace = true` to insert a space between the number and the unit, as recommended by the SI standard.
```nim
doAssert formatSize((1'i64 shl 31) + (300'i64 shl 20)) == "2.293GiB"
doAssert formatSize((2.234*1024*1024).int) == "2.233MiB"
doAssert formatSize(4096, includeSpace = true) == "4 KiB"
doAssert formatSize(4096, prefix = bpColloquial, includeSpace = true) == "4 kB"
doAssert formatSize(4096) == "4KiB"
doAssert formatSize(5_378_934, prefix = bpColloquial, decimalSep = ',') == "5,129MB"
```

#### `formatEng(f: BiggestFloat, precision: range[0..32] = 10, trim: bool = true, siPrefix: bool = false, unit: string = "", decimalSep = '.', useUnitSpace = false): string`
Formats `f` using **engineering notation**: the exponent (if any) is always a multiple of 3, and the significand is kept in the range `-1000.0 < significand < 1000.0`. Values with `-1000.0 < f < 1000.0` are shown with no exponent at all.

- `precision` — number of digits shown after the decimal point (or, if `trim = true`, the *maximum* number of digits shown).
- `trim` (default `true`) — strips trailing zeros from the fractional part, producing the shortest exact representation (up to `precision` digits). If `false`, always shows exactly `precision` digits.
- `siPrefix` (default `false`) — if `true`, replaces the `eN` exponent with the corresponding SI unit prefix letter (`k`, `M`, `G`, `T`, `P`, `E` for positive exponents; `m`, `u`, `n`, `p`, `f`, `a` for negative ones — note `u` is used instead of `μ` per ISO 2955). Numbers whose exponent falls outside the range covered by these prefixes (roughly `1e-18` to `1e18`) still use `eN` even if `siPrefix` is true.
- `unit` — a unit string appended to the result.
- `useUnitSpace` — if `true`, a space is inserted before the unit even when there's no SI prefix (the exact placement of the space depends on whether an exponent/prefix is present).
- `decimalSep` — character used as the decimal separator.

```nim
formatEng(0, 2, trim=false) == "0.00"
formatEng(0, 2) == "0"
formatEng(0.053, 0) == "53e-3"
formatEng(52731234, 2) == "52.73e6"
formatEng(-52731234, 2) == "-52.73e6"

formatEng(4100, siPrefix=true, unit="V") == "4.1 kV"
formatEng(4.1, siPrefix=true, unit="V") == "4.1 V"
formatEng(4.1, siPrefix=true) == "4.1"          # Note: no trailing space without unit
formatEng(4100, siPrefix=true) == "4.1 k"
formatEng(4.1, siPrefix=true, unit="") == "4.1 "      # Space with unit=""
formatEng(4100, siPrefix=true, unit="") == "4.1 k"
formatEng(4100) == "4.1e3"
formatEng(4100, unit="V") == "4.1e3 V"
formatEng(4100, unit="", useUnitSpace=true) == "4.1e3 " # Space with useUnitSpace=true
```

---

### String interpolation (`%` and `format`)

#### `addf(s: var string, formatstr: string, a: varargs[string, `$`])`
The efficient, in-place equivalent of `add(s, formatstr % a)` — appends the interpolated result of `formatstr` directly to `s` without an intermediate allocation for the result of `%`. This proc implements the substitution logic described below; `%` and `format` are thin wrappers around it.

#### `% (formatstr: string, a: openArray[string]): string`
The **substitution operator**. Scans `formatstr` for placeholders and replaces them using values from `a`. Supported placeholder forms:

- **`$N`** (and `$-N`) — positional reference. `$1` refers to `a[0]`, `$2` to `a[1]`, etc. (1-based indexing). `$-1` refers to `a[a.high]` (counting from the end).
- **`$#`** — refers to the "next" positional argument, incrementing an internal counter each time it's used. Lets you avoid manually numbering sequential placeholders.
- **`$$`** — produces a literal `$` in the output.
- **`${...}`** — braces form. If the content between braces is purely numeric, it behaves like `$N`/`$-N` (positional). Otherwise, the content is treated as a *named* placeholder (see below).
- **`$identifier`** (a run of `[A-Za-z0-9_\128-\255]` characters after `$`) — named placeholder. In this case, `a` is interpreted as a flat list of alternating **key, value** pairs: elements at even indices (0, 2, 4, …) are keys and odd indices (1, 3, 5, …) are the corresponding values. The placeholder name is compared against the keys using `cmpIgnoreStyle` (case- and underscore-insensitive).

If `formatstr` is malformed (e.g. references an out-of-range index, or a named placeholder that doesn't exist), `ValueError` is raised.

```nim
"$1 eats $2." % ["The cat", "fish"]
# "The cat eats fish."

"$# eats $#." % ["The cat", "fish"]
# "The cat eats fish."

"$animal eats $food." % ["animal", "The cat", "food", "fish"]
# "The cat eats fish."
```

#### `% (formatstr, a: string): string`
Shortcut for `formatstr % [a]` — interpolates a *single* string value as `$1`/`$#`.

#### `format(formatstr: string, a: varargs[string, `$`]): string`
Same as `formatstr % a`, but additionally supports **automatic stringification**: arguments that are not already strings are converted via `$` before substitution. Use this when passing non-string values (numbers, objects with a `$` proc, etc.) directly.

---

## Recommendations: which function to use when

- **Splitting input text:**
  - Use `splitWhitespace` when you want to tokenize free-form text and don't care about runs of multiple spaces/tabs — leading/trailing whitespace is also discarded.
  - Use `split(s, ",")` (or another fixed delimiter string/char) for structured data such as CSV-like fields, where empty fields between consecutive delimiters matter and must be preserved.
  - Use `split(s, seps: set[char])` when several different *single-character* delimiters should all be treated as equivalent (e.g. `{',', ';'}`).
  - Use `rsplit` with `maxsplit = 1` when you need to split off only the **last** component (e.g. file extension, last path segment) and keep the rest of the string intact.
  - Use `splitLines` for line-based processing; prefer `countLines` if you only need the number of lines (it's faster and avoids allocations).
  - Use `tokenize` when you need to **reconstruct** the original string exactly (including all separator runs), e.g. for a text formatter or pretty-printer.

- **Joining text back together:**
  - Use `join` for both `seq[string]` (direct overload) and sequences of other types (generic `join[T: not string]`, which stringifies via `$`).

- **Padding / aligning text for display:**
  - Use `align` for right-aligned (e.g. numeric columns) output, `alignLeft` for left-aligned (e.g. text columns), and `center` for centered headers/titles.
  - Use `spaces` when you need raw padding without a target column width computed by `align`/`alignLeft`.
  - Use `indent`/`unindent`/`dedent` for block-level reindentation of multi-line text; prefer `dedent` over `unindent` when you want to preserve *relative* indentation between lines (e.g. nested code blocks), and `unindent` when you want to strip a fixed amount/all leading padding from every line.

- **Searching and replacing:**
  - Use `find`/`rfind` for locating a single occurrence (first or last) and getting its index.
  - Use `contains` (or the `in`/`notin` operators) when you only need a yes/no membership test, not the position.
  - Use `count` when you need the number of occurrences, not their positions.
  - Use `replace` for simple "replace all" operations; use `replaceWord` if matches must be whole words; use `multiReplace` (string or char-set variant) when you need **several** replacements applied in a single efficient pass, especially for sanitizing input against multiple forbidden substrings/characters at once.
  - If you will repeatedly search for the **same** substring across many different strings, precompute a `SkipTable` with `initSkipTable` once and reuse it via `find(table, s, sub, ...)` for better performance.

- **Trimming and cleanup:**
  - Use `strip` for general leading/trailing character removal (default: whitespace).
  - Use `stripLineEnd` specifically to remove a single trailing newline sequence (e.g. after reading a line from a process or file).
  - Use `removePrefix`/`removeSuffix` for targeted, in-place removal of a known prefix/suffix (string, single character, or character set).

- **Case and identifier handling:**
  - Use `toLowerAscii`/`toUpperAscii`/`capitalizeAscii` for ASCII-only case changes; use the re-exported `unicode.toLower`/`unicode.toUpper` for full Unicode support.
  - Use `cmpIgnoreCase` for simple case-insensitive comparisons.
  - Use `cmpIgnoreStyle`/`normalize` for "style-insensitive" comparisons of human-facing config-style strings (case- and underscore-insensitive) — but **never** for comparing Nim identifiers; for that, use `nimIdentNormalize` (compiler-style normalization) or `macros.eqIdent`.
  - Use `validIdentifier` to check whether a string is syntactically usable as an identifier before, e.g., generating code.

- **Numbers and strings:**
  - Use `parseInt`/`parseFloat`/`parseBool`/`parseEnum` and friends to parse user-supplied or config text into typed values — they raise `ValueError` on bad input, so wrap calls in `try/except` (or pre-validate) when input may be untrusted.
  - Use `parseBinInt`/`parseOctInt`/`parseHexInt` (returning `int`) for simple cases, or `fromBin`/`fromOct`/`fromHex` (generic over the target integer type) when you need a specific width (e.g. `uint8`, `int16`) — remember these do **not** check for overflow.
  - Use `toBin`/`toOct`/`toHex`/`toOctal`/`intToStr` to render numbers in a fixed-width textual form (e.g. for hex dumps, binary visualizations, zero-padded IDs).
  - Use `insertSep` to add thousands separators for human-readable display of large numbers.
  - Use `parseHexStr`/`toHex(s: string)` to convert between raw bytes and their hex-text representation (e.g. for hashes, binary data debugging).

- **Floating point / sizes:**
  - Use `formatFloat`/`formatBiggestFloat` for general numeric formatting with explicit precision/notation control; combine with `trimZeros` if you want to drop trailing zeros from a fixed-precision result.
  - Use `formatSize` for displaying file sizes, memory usage, transfer rates, etc., in human-readable binary units.
  - Use `formatEng` for scientific/engineering contexts where SI prefixes (k, M, G, m, µ, n, …) or strict multiple-of-3 exponents are expected (e.g. electronics, measurement data).
  - For more elaborate or locale-aware formatting needs, also consider the `strformat` module (string interpolation with format specifiers).

- **Building strings with substitution:**
  - Use `%`/`format` for simple templated text where placeholders (`$1`, `$#`, `$name`) are filled from a list of strings/values; use `addf` if you're appending the result directly into an existing string buffer to avoid extra allocations.
  - For more complex or type-safe interpolation, prefer the `strformat` module's `&"..."` syntax.
