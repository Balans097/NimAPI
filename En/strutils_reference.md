# `std/strutils` Module Reference (Nim)

> The `strutils` module provides procedures, iterators, and templates for working with ASCII strings.
> Available for the JavaScript backend. For Unicode support, use `std/unicode`.

---

## Table of Contents

1. [Constants — Character Sets](#constants--character-sets)
2. [Character Predicates](#character-predicates)
3. [Case Conversion](#case-conversion)
4. [Normalization and Comparison](#normalization-and-comparison)
5. [Splitting Strings (split)](#splitting-strings-split)
6. [Splitting from the Right (rsplit)](#splitting-from-the-right-rsplit)
7. [Line and Whitespace Splitting](#line-and-whitespace-splitting)
8. [Converting Numbers to Strings](#converting-numbers-to-strings)
9. [Parsing Strings into Numbers and Booleans](#parsing-strings-into-numbers-and-booleans)
10. [Repetition and Alignment](#repetition-and-alignment)
11. [Indentation and Block Formatting](#indentation-and-block-formatting)
12. [Stripping (strip)](#stripping-strip)
13. [Prefix and Suffix Checks](#prefix-and-suffix-checks)
14. [Removing Prefixes and Suffixes](#removing-prefixes-and-suffixes)
15. [Searching in Strings](#searching-in-strings)
16. [Counting Occurrences](#counting-occurrences)
17. [String Replacement](#string-replacement)
18. [Joining Strings (join)](#joining-strings-join)
19. [Floating-Point Formatting](#floating-point-formatting)
20. [String Formatting (`%` and `format`)](#string-formatting--and-format)
21. [Miscellaneous Utilities](#miscellaneous-utilities)
22. [SkipTable — Optimized Search](#skiptable--optimized-search)
23. [tokenize Iterator](#tokenize-iterator)

---

## Constants — Character Sets

```nim
Whitespace*      = {' ', '\t', '\v', '\r', '\l', '\f'}
Letters*         = {'A'..'Z', 'a'..'z'}
UppercaseLetters* = {'A'..'Z'}
LowercaseLetters* = {'a'..'z'}
Digits*          = {'0'..'9'}
HexDigits*       = {'0'..'9', 'A'..'F', 'a'..'f'}
PunctuationChars* = {'!'..'/', ':'..'@', '['..'`', '{'..'~'}
IdentChars*      = {'a'..'z', 'A'..'Z', '0'..'9', '_'}
IdentStartChars* = {'a'..'z', 'A'..'Z', '_'}
Newlines*        = {'\13', '\10'}
PrintableChars*  = Letters + Digits + PunctuationChars + Whitespace
AllChars*        = {'\x00'..'\xFF'}
```

`AllChars` is useful for creating inverted sets to find invalid characters:

```nim
let invalid = AllChars - Digits
doAssert "01234".find(invalid) == -1
doAssert "01A34".find(invalid) == 2
```

---

## Character Predicates

### `isAlphaAscii`
```nim
func isAlphaAscii(c: char): bool
```
Returns `true` if the character is an ASCII letter (A–Z or a–z).

```nim
doAssert isAlphaAscii('e') == true
doAssert isAlphaAscii('E') == true
doAssert isAlphaAscii('8') == false
```

---

### `isAlphaNumeric`
```nim
func isAlphaNumeric(c: char): bool
```
Returns `true` if the character is an ASCII letter or digit (A–Z, a–z, 0–9).

```nim
doAssert isAlphaNumeric('n') == true
doAssert isAlphaNumeric('8') == true
doAssert isAlphaNumeric(' ') == false
```

---

### `isDigit`
```nim
func isDigit(c: char): bool
```
Returns `true` if the character is a decimal digit (0–9).

```nim
doAssert isDigit('n') == false
doAssert isDigit('8') == true
```

---

### `isSpaceAscii`
```nim
func isSpaceAscii(c: char): bool
```
Returns `true` if the character is an ASCII whitespace character.

```nim
doAssert isSpaceAscii('n') == false
doAssert isSpaceAscii(' ') == true
doAssert isSpaceAscii('\t') == true
```

---

### `isLowerAscii`
```nim
func isLowerAscii(c: char): bool
```
Returns `true` if the character is an ASCII lowercase letter.

```nim
doAssert isLowerAscii('e') == true
doAssert isLowerAscii('E') == false
doAssert isLowerAscii('7') == false
```

---

### `isUpperAscii`
```nim
func isUpperAscii(c: char): bool
```
Returns `true` if the character is an ASCII uppercase letter.

```nim
doAssert isUpperAscii('e') == false
doAssert isUpperAscii('E') == true
doAssert isUpperAscii('7') == false
```

---

### `isEmptyOrWhitespace`
```nim
func isEmptyOrWhitespace(s: string): bool
```
Returns `true` if the string is empty or contains only whitespace characters.

```nim
doAssert isEmptyOrWhitespace("") == true
doAssert isEmptyOrWhitespace("   ") == true
doAssert isEmptyOrWhitespace("  a ") == false
```

---

### `allCharsInSet`
```nim
func allCharsInSet(s: string, theSet: set[char]): bool
```
Returns `true` if every character of `s` is in `theSet`. An empty string returns `true`.

```nim
doAssert allCharsInSet("aeea", {'a', 'e'}) == true
doAssert allCharsInSet("", {'a', 'e'}) == true
doAssert allCharsInSet("aeba", {'a', 'e'}) == false
```

---

### `validIdentifier`
```nim
func validIdentifier(s: string): bool
```
Returns `true` if `s` is a valid Nim identifier (starts with `IdentStartChars`, followed by `IdentChars`).

```nim
doAssert "abc_def08".validIdentifier == true
doAssert "0abc".validIdentifier == false
doAssert "".validIdentifier == false
```

---

## Case Conversion

### `toLowerAscii`
```nim
func toLowerAscii(c: char): char
func toLowerAscii(s: string): string
```
Converts a character or string to lowercase (ASCII A–Z only).

```nim
doAssert toLowerAscii('A') == 'a'
doAssert toLowerAscii('e') == 'e'
doAssert toLowerAscii("FooBar!") == "foobar!"
```

---

### `toUpperAscii`
```nim
func toUpperAscii(c: char): char
func toUpperAscii(s: string): string
```
Converts a character or string to uppercase (ASCII a–z only).

```nim
doAssert toUpperAscii('a') == 'A'
doAssert toUpperAscii('E') == 'E'
doAssert toUpperAscii("FooBar!") == "FOOBAR!"
```

---

### `capitalizeAscii`
```nim
func capitalizeAscii(s: string): string
```
Converts only the first character to uppercase. Works for ASCII A–Z only.

```nim
doAssert capitalizeAscii("foo") == "Foo"
doAssert capitalizeAscii("-bar") == "-bar"
```

---

## Normalization and Comparison

### `normalize`
```nim
func normalize(s: string): string
```
Converts to lowercase and removes all `_` characters. Should NOT be used for Nim identifier normalization.

```nim
doAssert normalize("Foo_bar") == "foobar"
doAssert normalize("Foo Bar") == "foo bar"
```

---

### `nimIdentNormalize`
```nim
func nimIdentNormalize(s: string): string
```
Normalizes a string as a Nim identifier: converts to lowercase, removes `_` except for the first character.

```nim
doAssert nimIdentNormalize("Foo_bar") == "Foobar"
```

---

### `cmpIgnoreCase`
```nim
func cmpIgnoreCase(a, b: string): int
```
Compares two strings case-insensitively. Returns `0` if equal, `< 0` if `a < b`, `> 0` if `a > b`.

```nim
doAssert cmpIgnoreCase("FooBar", "foobar") == 0
doAssert cmpIgnoreCase("bar", "Foo") < 0
doAssert cmpIgnoreCase("Foo5", "foo4") > 0
```

---

### `cmpIgnoreStyle`
```nim
func cmpIgnoreStyle(a, b: string): int
```
Compares strings ignoring both case and underscore characters. Equivalent to `cmp(normalize(a), normalize(b))` but without allocation. Do NOT use for Nim identifier comparison.

```nim
doAssert cmpIgnoreStyle("foo_bar", "FooBar") == 0
doAssert cmpIgnoreStyle("foo_bar_5", "FooBar4") > 0
```

---

## Splitting Strings (split)

All `split` functions have three overloads: by character, character set, or string. Both iterator and `seq`-returning func variants are available.

### `split` (iterator)
```nim
iterator split(s: string, sep: char, maxsplit: int = -1): string
iterator split(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): string
iterator split(s: string, sep: string, maxsplit: int = -1): string
```
Splits string `s` into substrings. `maxsplit` limits the number of splits (-1 = unlimited). An empty separator returns the original string.

```nim
for word in split("this is an example"):
  echo word  # "this", "is", "an", "example"

for word in split("a,b,,c", ','):
  echo word  # "a", "b", "", "c"

for word in split("aXbYc", {'X', 'Y'}):
  echo word  # "a", "b", "c"
```

---

### `split` (func)
```nim
func split(s: string, sep: char, maxsplit: int = -1): seq[string]
func split(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): seq[string]
func split(s: string, sep: string, maxsplit: int = -1): seq[string]
```
Same as the iterator, but returns a `seq[string]`.

```nim
doAssert "a,b,c".split(',') == @["a", "b", "c"]
doAssert "".split(' ') == @[""]
doAssert "a,b;c".split({',', ';'}) == @["a", "b", "c"]
doAssert "a,b,c".split(",") == @["a", "b", "c"]
doAssert "a  spaced sentence".split(" ", maxsplit = 1) ==
  @["a", " spaced sentence"]
doAssert "empty seps return unsplit s".split({}) == @["empty seps return unsplit s"]
```

---

## Splitting from the Right (rsplit)

### `rsplit` (iterator)
```nim
iterator rsplit(s: string, sep: char, maxsplit: int = -1): string
iterator rsplit(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): string
iterator rsplit(s: string, sep: string, maxsplit: int = -1, keepSeparators: bool = false): string
```
Like `split` but scans from the **right**. Substrings are yielded in reverse order.

```nim
for piece in "foo:bar".rsplit(':'):
  echo piece  # "bar", then "foo"
```

---

### `rsplit` (func)
```nim
func rsplit(s: string, sep: char, maxsplit: int = -1): seq[string]
func rsplit(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): seq[string]
func rsplit(s: string, sep: string, maxsplit: int = -1): seq[string]
```
Like the iterator, but returns results in **original** (non-reversed) order.

```nim
doAssert "a  largely    spaced sentence".rsplit(" ", maxsplit = 1) ==
  @["a  largely    spaced", "sentence"]
doAssert "a,b,c".rsplit(",") == @["a", "b", "c"]
# Common use case: path manipulation
var tailSplit = rsplit("Root#Object#Method#Index", '#', maxsplit=1)
# @["Root#Object#Method", "Index"]
```

---

## Line and Whitespace Splitting

### `splitLines` (iterator and func)
```nim
iterator splitLines(s: string, keepEol = false): string
func splitLines(s: string, keepEol = false): seq[string]
```
Splits a string into lines. Supports CR, LF, and CRLF. If `keepEol = true`, the end-of-line characters are preserved.

```nim
assert splitLines("first line\nsecond line\nthird line") ==
  @["first line", "second line", "third line"]

for line in splitLines("\nfoo\nbar\n"):
  echo line  # "", "foo", "bar", ""
```

---

### `splitWhitespace` (iterator and func)
```nim
iterator splitWhitespace(s: string, maxsplit: int = -1): string
func splitWhitespace(s: string, maxsplit: int = -1): seq[string]
```
Splits by whitespace, stripping leading and trailing whitespace from each resulting token.

```nim
let s = "  foo \t bar  baz  "
doAssert s.splitWhitespace() == @["foo", "bar", "baz"]
doAssert s.splitWhitespace(maxsplit = 1) == @["foo", "bar  baz  "]
```

---

### `countLines`
```nim
func countLines(s: string): int
```
Returns the number of lines in the string. More efficient than `len(splitLines(s))`.

```nim
doAssert countLines("First line\l and second line.") == 2
doAssert countLines("a\nb\nc") == 3
```

---

## Converting Numbers to Strings

### `toBin`
```nim
func toBin(x: BiggestInt, len: Positive): string
```
Converts a number to its binary string representation of exactly `len` characters. No `0b` prefix is added.

```nim
doAssert 29.toBin(8) == "00011101"
doAssert 257.toBin(9) == "100000001"
```

---

### `toOct`
```nim
func toOct(x: BiggestInt, len: Positive): string
```
Converts a number to its octal string representation of exactly `len` characters. No `0o` prefix is added.

```nim
doAssert 62.toOct(3) == "076"
doAssert 513.toOct(5) == "01001"
```

---

### `toOctal`
```nim
func toOctal(c: char): string
```
Converts a character to its octal representation. Always returns a string of length 3.

```nim
doAssert toOctal('1') == "061"
doAssert toOctal('A') == "101"
doAssert toOctal('a') == "141"
```

---

### `toHex` (numbers)
```nim
func toHex[T: SomeInteger](x: T, len: Positive): string
func toHex[T: SomeInteger](x: T): string  # = toHex(x, T.sizeof * 2)
```
Converts a number to its hexadecimal representation. No `0x` prefix. Treats the value as unsigned.

```nim
doAssert 62'u64.toHex(3) == "03E"
doAssert toHex(-8, 6) == "FFFFF8"
doAssert toHex(1984'i64) == "00000000000007C0"
doAssert toHex(1984'i16) == "07C0"
```

---

### `toHex` (string)
```nim
func toHex(s: string): string
```
Converts a byte string to its hexadecimal representation. The output is twice the length of the input.

```nim
doAssert "1".toHex() == "31"
doAssert "A".toHex() == "41"
doAssert "\0\255".toHex() == "00FF"
```

---

### `intToStr`
```nim
func intToStr(x: int, minchars: Positive = 1): string
```
Converts an integer to its decimal string, zero-padded to at least `minchars` characters.

```nim
doAssert intToStr(1984) == "1984"
doAssert intToStr(1984, 6) == "001984"
doAssert intToStr(-42, 5) == "-0042"
```

---

### `insertSep`
```nim
func insertSep(s: string, sep = '_', digits = 3): string
```
Inserts separator `sep` every `digits` characters from the right — useful for formatting large numbers.

```nim
doAssert insertSep("1000000") == "1_000_000"
doAssert insertSep("1000000", ',') == "1,000,000"
```

---

## Parsing Strings into Numbers and Booleans

All parse functions raise `ValueError` on invalid input.

### `parseInt`
```nim
func parseInt(s: string): int
```
```nim
doAssert parseInt("-0042") == -42
```

### `parseBiggestInt`
```nim
func parseBiggestInt(s: string): BiggestInt
```

### `parseUInt`
```nim
func parseUInt(s: string): uint
```

### `parseBiggestUInt`
```nim
func parseBiggestUInt(s: string): BiggestUInt
```

### `parseFloat`
```nim
func parseFloat(s: string): float
```
Supports `NAN`, `INF`, `-INF` (case-insensitive).

```nim
doAssert parseFloat("3.14") == 3.14
doAssert parseFloat("inf") == 1.0/0
```

---

### `parseBinInt`
```nim
func parseBinInt(s: string): int
```
Parses a binary integer string. Supports `0b`, `0B` prefixes and `_` separators.

```nim
doAssert "0b11_0101".parseBinInt() == 53
doAssert "111".parseBinInt() == 7
```

---

### `parseOctInt`
```nim
func parseOctInt(s: string): int
```
Parses an octal integer string. Supports `0o`, `0O` prefixes and `_` separators.

---

### `parseHexInt`
```nim
func parseHexInt(s: string): int
```
Parses a hexadecimal integer string. Supports `0x`, `0X`, `#` prefixes and `_` separators.

---

### `fromBin` / `fromOct` / `fromHex`
```nim
func fromBin[T: SomeInteger](s: string): T
func fromOct[T: SomeInteger](s: string): T
func fromHex[T: SomeInteger](s: string): T
```
Generic parsers with explicit result type. No overflow checking — if the value doesn't fit, only the rightmost bits are returned.

```nim
doAssert fromBin[int]("0b_0100_1000") == 72
doAssert fromOct[int]("0o777") == 511
doAssert fromHex[int]("0xFF") == 255
doAssert fromHex[int8]("0xFF") == -1'i8   # silent overflow
```

---

### `parseHexStr`
```nim
func parseHexStr(s: string): string
```
Converts a hex-encoded string back to a byte string. Case-insensitive. The input length must be even.

```nim
doAssert parseHexStr("41") == "A"
doAssert parseHexStr("00ff") == "\0\255"
```

---

### `parseBool`
```nim
func parseBool(s: string): bool
```
Parses a boolean value. Accepts `y, yes, true, 1, on` → `true`; `n, no, false, 0, off` → `false`. Case-insensitive (normalizes via `normalize`).

```nim
doAssert parseBool("yes") == true
doAssert parseBool("False") == false
doAssert parseBool("on") == true
```

---

### `parseEnum`
```nim
func parseEnum[T: enum](s: string): T
func parseEnum[T: enum](s: string, default: T): T
```
Parses an enum value by name. Comparison is Nim-style (first letter case-sensitive, rest case-insensitive and ignoring `_`). On failure: first overload raises `ValueError`, second returns `default`.

```nim
type Color = enum red, green, blue

doAssert parseEnum[Color]("red") == red
doAssert parseEnum[Color]("GREEN") == green
doAssert parseEnum[Color]("unknown", blue) == blue
```

---

## Repetition and Alignment

### `repeat`
```nim
func repeat(c: char, count: Natural): string
func repeat(s: string, n: Natural): string
```
Repeats a character `count` times or a string `n` times.

```nim
doAssert 'z'.repeat(5) == "zzzzz"
doAssert "+ foo +".repeat(3) == "+ foo ++ foo ++ foo +"
```

---

### `spaces`
```nim
func spaces(n: Natural): string
```
Returns a string of `n` space characters. Useful for simple left-alignment.

```nim
let width = 15
let text = "Hello user!"
doAssert text & spaces(max(0, width - text.len)) & "|" == "Hello user!    |"
```

---

### `align`
```nim
func align(s: string, count: Natural, padding = ' '): string
```
Right-aligns `s` in a field of `count` characters, padding with `padding` on the left. If `s.len >= count`, returns `s` unchanged.

```nim
assert align("abc", 4) == " abc"
assert align("1232", 6) == "  1232"
assert align("1232", 6, '#') == "##1232"
assert align("toolong", 3) == "toolong"
```

---

### `alignLeft`
```nim
func alignLeft(s: string, count: Natural, padding = ' '): string
```
Left-aligns `s` in a field of `count` characters, padding with `padding` on the right.

```nim
assert alignLeft("abc", 4) == "abc "
assert alignLeft("1232", 6) == "1232  "
assert alignLeft("1232", 6, '#') == "1232##"
```

---

### `center`
```nim
func center(s: string, width: int, fillChar: char = ' '): string
```
Centers `s` in a field of `width` characters using `fillChar`. If there are an odd number of padding characters, the extra one goes on the right.

```nim
doAssert "foo".center(5) == " foo "
doAssert "foo".center(6) == " foo  "
doAssert "foo".center(2) == "foo"  # unchanged when width <= len
```

---

## Indentation and Block Formatting

### `indent`
```nim
func indent(s: string, count: Natural, padding: string = " "): string
```
Prepends `count * padding` to the beginning of every line in `s`.

```nim
doAssert indent("First line\nsecond line.", 2) == "  First line\n  second line."
doAssert indent("A\nB", 1, ">>") == ">>A\n>>B"
```

---

### `unindent`
```nim
func unindent(s: string, count: Natural = int.high, padding: string = " "): string
```
Removes up to `count` occurrences of `padding` from the beginning of every line.

```nim
let x = """
  Hello
    There
""".unindent()
doAssert x == "Hello\nThere\n"
```

---

### `indentation`
```nim
func indentation(s: string): Natural
```
Returns the number of spaces common to all non-whitespace-only lines in `s`.

```nim
doAssert indentation("  foo\n  bar") == 2
doAssert indentation("  foo\n    bar") == 2
```

---

### `dedent`
```nim
func dedent(s: string, count: Natural = indentation(s)): string
```
Removes the common leading indentation from all lines in `s` (spaces only). By default removes exactly the shared indentation. Unlike `unindent`, it respects the actual common indent.

```nim
let x = """
  Hello
    There
""".dedent()
doAssert x == "Hello\n  There\n"
```

---

### `delete`
```nim
func delete(s: var string, slice: Slice[int])
```
Deletes the characters `s[slice]` in-place. Raises `IndexDefect` if the slice is out of bounds.

```nim
var a = "abcde"
a.delete(1..2)
assert a == "ade"

a.delete(1..<1)  # empty slice — no-op
assert a == "ade"
```

---

## Stripping (strip)

### `strip`
```nim
func strip(s: string, leading = true, trailing = true,
           chars: set[char] = Whitespace): string
```
Removes all leading and/or trailing characters from the set `chars`. Both `leading` and `trailing` default to `true`.

```nim
doAssert strip("  vhellov   ") == "vhellov"
doAssert "  vhellov   ".strip(leading = false) == "  vhellov"
doAssert "  vhellov   ".strip(trailing = false) == "vhellov   "
doAssert "vhellov".strip(chars = {'v'}) == "hello"
```

---

### `stripLineEnd`
```nim
func stripLineEnd(s: var string)
```
Strips **one** line-ending suffix (`\r`, `\n`, `\r\n`, `\f`, `\v`) from the end of `s` in-place. Also known as `chomp`.

```nim
var s = "foo\n\n"
s.stripLineEnd
doAssert s == "foo\n"

s = "foo\r\n"
s.stripLineEnd
doAssert s == "foo"
```

---

## Prefix and Suffix Checks

### `startsWith`
```nim
func startsWith(s: string, prefix: char): bool
func startsWith(s, prefix: string): bool
```
Returns `true` if `s` starts with the given character or string. An empty `prefix` always returns `true`.

```nim
let a = "abracadabra"
doAssert a.startsWith('a') == true
doAssert a.startsWith("abra") == true
doAssert a.startsWith("bra") == false
```

---

### `endsWith`
```nim
func endsWith(s: string, suffix: char): bool
func endsWith(s, suffix: string): bool
```
Returns `true` if `s` ends with the given character or string.

```nim
let a = "abracadabra"
doAssert a.endsWith('a') == true
doAssert a.endsWith("abra") == true
doAssert a.endsWith("dab") == false
```

---

### `continuesWith`
```nim
func continuesWith(s, substr: string, start: Natural): bool
```
Returns `true` if `s` contains `substr` starting at position `start`.

```nim
let a = "abracadabra"
doAssert a.continuesWith("ca", 4) == true
doAssert a.continuesWith("ca", 5) == false
doAssert a.continuesWith("dab", 6) == true
```

---

## Removing Prefixes and Suffixes

### `removePrefix`
```nim
func removePrefix(s: var string, chars: set[char] = Newlines)
func removePrefix(s: var string, c: char)
func removePrefix(s: var string, prefix: string)
```
Removes all characters from `chars` / a single character / the first occurrence of `prefix` from the start of `s` in-place.

```nim
var s = "\r\n*~Hello World!"
s.removePrefix           # removes Newlines characters
doAssert s == "*~Hello World!"

s.removePrefix({'~', '*'})
doAssert s == "Hello World!"

var answers = "yesyes"
answers.removePrefix("yes")
doAssert answers == "yes"
```

---

### `removeSuffix`
```nim
func removeSuffix(s: var string, chars: set[char] = Newlines)
func removeSuffix(s: var string, c: char)
func removeSuffix(s: var string, suffix: string)
```
Same as `removePrefix` but operates on the end of the string.

```nim
var s = "Hello World!*~\r\n"
s.removeSuffix
doAssert s == "Hello World!*~"

var table = "users"
table.removeSuffix('s')
doAssert table == "user"

var answers = "yeses"
answers.removeSuffix("es")
doAssert answers == "yes"
```

---

## Searching in Strings

### `find`
```nim
func find(s: string, sub: char, start: Natural = 0, last = -1): int
func find(s: string, chars: set[char], start: Natural = 0, last = -1): int
func find(s, sub: string, start: Natural = 0, last = -1): int
func find(a: SkipTable, s, sub: string, start: Natural = 0, last = -1): int
```
Finds the first occurrence of a character, character set, or substring in the range `start..last`. Returns the index relative to `s[0]`, or -1 if not found. Case-sensitive.

```nim
doAssert "abcdef".find('c') == 2
doAssert "abcdef".find({'c', 'e'}) == 2
doAssert "abcdef".find("cd") == 2
doAssert "abcdef".find('z') == -1
doAssert "abcabc".find('c', start = 3) == 5
```

---

### `rfind`
```nim
func rfind(s: string, sub: char, start: Natural = 0, last = -1): int
func rfind(s: string, chars: set[char], start: Natural = 0, last = -1): int
func rfind(s, sub: string, start: Natural = 0, last = -1): int
```
Finds the **last** occurrence by scanning from right to left (from `last` down to `start`).

```nim
doAssert "abcabc".rfind('c') == 5
doAssert "abcabc".rfind("bc") == 4
```

---

### `contains`
```nim
func contains(s, sub: string): bool
func contains(s: string, chars: set[char]): bool
```
Returns `true` if the substring or any character from the set is present in `s`. Equivalent to `find(s, sub) >= 0`.

```nim
doAssert "hello world".contains("world") == true
doAssert "hello".contains({'x', 'y'}) == false
```

---

## Counting Occurrences

### `count`
```nim
func count(s: string, sub: char): int
func count(s: string, subs: set[char]): int
func count(s: string, sub: string, overlapping: bool = false): int
```
Counts occurrences of a character, character set, or substring. With `overlapping = true`, overlapping matches are counted.

```nim
doAssert "aababc".count('a') == 3
doAssert "aababc".count({'a', 'b'}) == 5
doAssert "aababc".count("ab") == 2
doAssert "aaa".count("aa") == 1              # no overlap
doAssert "aaa".count("aa", overlapping = true) == 2
```

---

## String Replacement

### `replace`
```nim
func replace(s, sub: string, by = ""): string
func replace(s: string, sub, by: char): string
```
Replaces **all** occurrences of `sub` with `by`. The character version is optimized.

```nim
doAssert "Hello World".replace("World", "Nim") == "Hello Nim"
doAssert "aabbcc".replace('b', 'x') == "aaxxcc"
doAssert "a,b,c".replace(",") == "abc"  # delete all commas
```

---

### `replaceWord`
```nim
func replaceWord(s, sub: string, by = ""): string
```
Replaces `sub` only when surrounded by word boundaries (similar to `\b` in regex).

```nim
# "bar" inside "foobar" is not replaced
doAssert "foo bar foobar".replaceWord("bar", "baz") == "foo baz foobar"
```

---

### `multiReplace` (strings)
```nim
func multiReplace(s: string, replacements: varargs[(string, string)]): string
```
Performs multiple replacements in a single pass. When multiple patterns match at the same position, the earliest one in the list is applied.

```nim
doAssert "abba".multiReplace([("a", "b"), ("b", "a")]) == "baab"
doAssert "abc".multiReplace([("bc", "x"), ("ab", "_b")]) == "_bc"
```

---

### `multiReplace` (character sets)
```nim
func multiReplace(s: openArray[char]; replacements: varargs[(set[char], char)]): string
```
Performs character-by-character replacement using character sets in a single pass. First matching rule wins.

```nim
const WinRules = [
  ({'\0'..'\31'}, ' '),
  ({'"'}, '\''),
  ({'/', '\\', ':', '|'}, '-'),
  ({'*', '?', '<', '>'}, '_'),
]
doAssert "a/file:with?invalid*chars.txt".multiReplace(WinRules) ==
  "a-file-with_invalid_chars.txt"
```

---

## Joining Strings (join)

### `join`
```nim
func join(a: openArray[string], sep: string = ""): string
proc join[T: not string](a: openArray[T], sep: string = ""): string
```
Concatenates all elements with `sep` between them. The second overload converts elements to strings using `$`.

```nim
doAssert join(["A", "B", "C"], " -> ") == "A -> B -> C"
doAssert join([1, 2, 3], ", ") == "1, 2, 3"
doAssert join(["a", "b", "c"]) == "abc"
```

---

## Floating-Point Formatting

### `FloatFormatMode`
```nim
type FloatFormatMode* = enum
  ffDefault    # shortest representation
  ffDecimal    # fixed-point notation
  ffScientific # scientific notation (e)
```

### `formatFloat`
```nim
func formatFloat(f: float, format: FloatFormatMode = ffDefault,
                 precision: range[-1..32] = 16; decimalSep = '.'): string
```
Converts a `float` to a string. With `precision == -1`, tries to choose the most compact form.

```nim
doAssert 123.456.formatFloat() == "123.4560000000000"
doAssert 123.456.formatFloat(ffDecimal, 4) == "123.4560"
doAssert 123.456.formatFloat(ffScientific, 2) == "1.23e+02"
```

---

### `formatBiggestFloat`
```nim
func formatBiggestFloat(f: BiggestFloat, format: FloatFormatMode = ffDefault,
                        precision: range[-1..32] = 16; decimalSep = '.'): string
```
Same as `formatFloat` but for the `BiggestFloat` type.

---

### `trimZeros`
```nim
func trimZeros(x: var string; decimalSep = '.')
```
Trims trailing zeros from a floating-point string representation, in-place.

```nim
var x = "123.456000000"
x.trimZeros()
doAssert x == "123.456"

var y = "1.20e+03"
y.trimZeros()
doAssert y == "1.2e+03"
```

---

### `formatSize`
```nim
func formatSize(bytes: int64, decimalSep = '.', prefix = bpIEC,
                includeSpace = false): string
```
Formats a byte count in human-readable form. `bpIEC` uses IEC standard (KiB, MiB...), `bpColloquial` uses SI-style names (kB, MB...).

```nim
doAssert formatSize(4096) == "4KiB"
doAssert formatSize(4096, includeSpace = true) == "4 KiB"
doAssert formatSize(4096, prefix = bpColloquial, includeSpace = true) == "4 kB"
doAssert formatSize((1'i64 shl 31) + (300'i64 shl 20)) == "2.293GiB"
```

---

### `formatEng`
```nim
func formatEng(f: BiggestFloat, precision: range[0..32] = 10,
               trim: bool = true, siPrefix: bool = false,
               unit: string = "", decimalSep = '.', useUnitSpace = false): string
```
Formats a number in engineering notation (exponent is always a multiple of 3). Supports SI prefix substitution.

```nim
doAssert formatEng(4100) == "4.1e3"
doAssert formatEng(4100, siPrefix = true, unit = "V") == "4.1 kV"
doAssert formatEng(0.053, 0) == "53e-3"
doAssert formatEng(52731234, 2) == "52.73e6"
```

---

## String Formatting (`%` and `format`)

### `%` (format operator)
```nim
func `%`(formatstr: string, a: openArray[string]): string
func `%`(formatstr, a: string): string
```
Interpolates a format string with values from `a`. Substitution tokens:
- `$1`, `$2`, ... — positional (1-based)
- `$#` — next argument in sequence
- `$$` — literal `$`
- `$name` / `${name}` — named variable (even indices are keys, odd indices are values)

```nim
doAssert "$1 eats $2." % ["The cat", "fish"] == "The cat eats fish."
doAssert "$# eats $#." % ["The cat", "fish"] == "The cat eats fish."
doAssert "$animal eats $food." % ["animal", "The cat", "food", "fish"] ==
  "The cat eats fish."
doAssert "value: $$42" % [] == "value: $42"
```

---

### `addf`
```nim
func addf(s: var string, formatstr: string, a: varargs[string, `$`])
```
More efficient equivalent of `add(s, formatstr % a)` — avoids creating an intermediate string.

```nim
var result = "Numbers: "
result.addf("$1 and $2", "42", "7")
doAssert result == "Numbers: 42 and 7"
```

---

### `format` (string formatting)
```nim
func format(formatstr: string, a: varargs[string, `$`]): string
```
Same as `formatstr % a` but with automatic stringification via `$`.

```nim
doAssert format("$1 + $2 = $3", 1, 2, 3) == "1 + 2 = 3"
```

---

## Miscellaneous Utilities

### `addSep`
```nim
func addSep(dest: var string, sep = ", ", startLen: Natural = 0)
```
Adds `sep` to `dest` only if `dest.len > startLen`. Useful for building comma-separated lists.

```nim
var arr = "["
for x in [2, 3, 5, 7, 11]:
  addSep(arr, startLen = len("["))
  arr.add($x)
arr.add("]")
doAssert arr == "[2, 3, 5, 7, 11]"
```

---

### `abbrev`
```nim
func abbrev(s: string, possibilities: openArray[string]): int
```
Returns the index of the first item in `possibilities` that starts with `s`. Returns `-1` if not found, `-2` if ambiguous (multiple matches).

```nim
doAssert abbrev("fac", ["college", "faculty", "industry"]) == 1
doAssert abbrev("foo", ["college", "faculty", "industry"]) == -1
doAssert abbrev("fac", ["college", "faculty", "faculties"]) == -2
doAssert abbrev("college", ["college", "colleges"]) == 0  # exact match
```

---

### `escape`
```nim
func escape(s: string, prefix = "\"", suffix = "\""): string
```
Escapes a string: control characters → `\xHH`, `\` → `\\`, `'` → `\'`, `"` → `\"`. Wraps the result in `prefix` / `suffix`.

```nim
doAssert escape("Hello\nWorld") == "\"Hello\\x0AWorld\""
doAssert escape("test", prefix = "", suffix = "") == "test"
```

---

### `unescape`
```nim
func unescape(s: string, prefix = "\"", suffix = "\""): string
```
The inverse of `escape`. Raises `ValueError` if `s` does not start with `prefix` or end with `suffix`.

```nim
doAssert unescape("\"Hello\\x0AWorld\"") == "Hello\nWorld"
```

---

## SkipTable — Optimized Search

`SkipTable` is a precomputed table for the Boyer–Moore–Horspool algorithm. Use it when searching for the same substring many times in different strings.

### `initSkipTable`
```nim
func initSkipTable(a: var SkipTable, sub: string)
func initSkipTable(sub: string): SkipTable
```
Initializes a skip table for the pattern `sub`.

### `find` (SkipTable)
```nim
func find(a: SkipTable, s, sub: string, start: Natural = 0, last = -1): int
```
Searches for `sub` in `s` using the precomputed table.

```nim
let needle = "example"
let table = initSkipTable(needle)
let haystack = "This is an example string"

doAssert table.find(haystack, needle) == 11
doAssert table.find(haystack, needle, start = 12) == -1
```

---

## tokenize Iterator

### `tokenize`
```nim
iterator tokenize(s: string, seps: set[char] = Whitespace): tuple[token: string, isSep: bool]
```
Splits a string into alternating tokens and separators, yielding `(token, isSep)` tuples. Unlike `split`, separators are **not** discarded — they are yielded as tokens with `isSep = true`.

```nim
for word in tokenize("  this is an  example  "):
  echo word
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
