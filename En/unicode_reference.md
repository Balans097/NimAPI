# `std/unicode` Module Reference (Nim)

> This module provides support for the **UTF-8** encoding and Unicode v12.0.0.  
> Related modules: `strutils`, `unidecode`, `encodings`.

---

## Table of Contents

1. [Types](#types)
2. [Encoding and Decoding](#encoding-and-decoding)
3. [Length and Positioning](#length-and-positioning)
4. [Rune Access](#rune-access)
5. [Validation](#validation)
6. [Rune Classification](#rune-classification)
7. [Case Conversion](#case-conversion)
8. [String Operations](#string-operations)
9. [Iterators](#iterators)
10. [Comparison Operators](#comparison-operators)

---

## Types

### `Rune`

```nim
type Rune* = distinct int32
```

A type that holds a single Unicode code point. The underlying storage type is `int32`.

A single `Rune` may combine with other runes to form a visible character (grapheme cluster) on screen.

```nim
import std/unicode

let r = "ñ".runeAt(0)
echo r           # ñ
echo r.ord       # 241
echo r.toUTF8    # "ñ"
```

---

## Encoding and Decoding

### `toUTF8(c: Rune): string`

Converts a rune into its UTF-8 string representation.

```nim
import std/unicode

let r = "ó".runeAt(0)
echo r.toUTF8       # "ó"
echo r.toUTF8.len   # 2 (bytes)
```

---

### `$(rune: Rune): string`

An alias for `toUTF8`. Enables direct use of runes in string concatenation.

```nim
import std/unicode

let r = "ñ".runeAt(0)
echo $r              # "ñ"
echo "Hello, " & $r  # "Hello, ñ"
```

---

### `$(runes: seq[Rune]): string`

Converts a sequence of runes back into a string. The reverse of `toRunes`.

```nim
import std/unicode

let someString = "öÑ"
let runes = toRunes(someString)
echo $runes      # "öÑ"
assert $runes == someString
```

---

### `add(s: var string; c: Rune)`

Appends a rune to the end of a string.

```nim
import std/unicode

var s = "abc"
let c = "ä".runeAt(0)
s.add(c)
echo s   # "abcä"
```

---

### `fastToUTF8Copy(c: Rune, s: var string, pos: int, doInc = true)` *(template)*

Low-level template: copies the UTF-8 bytes of rune `c` into the preallocated string `s` starting at byte position `pos`.

- If `doInc = true` (default), `pos` is incremented by the number of bytes written.
- For maximum efficiency, preallocate `s` with enough capacity before calling.

```nim
import std/unicode

let r = "ñ".runeAt(0)
var s = newString(0)
var pos = 0
fastToUTF8Copy(r, s, pos)
echo s   # "ñ"
```

---

### `fastRuneAt(s, i, result, doInc = true)` *(template)*

Low-level template: reads the rune from string `s` at byte position `i` and stores it in `result`.

- If `doInc = true`, `i` is advanced by the number of bytes consumed.
- Returns the replacement rune `U+FFFD` for invalid UTF-8 sequences.

```nim
import std/unicode

let s = "añyóng"
var i = 1  # byte position of 'ñ'
var r: Rune
fastRuneAt(s, i, r)
echo r   # ñ
echo i   # 3 (advanced by 2 bytes)
```

---

## Length and Positioning

### `runeLen(s): int`

Returns the number of runes in the string (not bytes).

```nim
import std/unicode

let a = "añyóng"
echo a.len      # 8 (bytes)
echo a.runeLen  # 6 (runes)
```

---

### `runeLenAt(s, i: Natural): int`

Returns the number of bytes taken by the rune starting at byte position `i`.

```nim
import std/unicode

let a = "añyóng"
echo a.runeLenAt(0)  # 1 (ASCII 'a')
echo a.runeLenAt(1)  # 2 ('ñ' — 2-byte character)
echo a.runeLenAt(3)  # 1 ('y')
```

---

### `size(r: Rune): int`

Returns the number of bytes the rune `r` occupies in its UTF-8 encoding.

```nim
import std/unicode

let runes = toRunes("aá")
echo size(runes[0])  # 1 ('a' — ASCII)
echo size(runes[1])  # 2 ('á')
```

---

### `runeOffset(s, pos: Natural, start: Natural = 0): int`

Returns the byte position of the rune at rune-index `pos`. The optional `start` parameter provides a starting byte offset. Returns `-1` if the string ends before rune `pos` is reached.

> ⚠️ **Slow operation.** Prefer iterators or `seq[Rune]` conversion for repeated access.

```nim
import std/unicode

let a = "añyóng"
echo a.runeOffset(0)  # 0
echo a.runeOffset(1)  # 1
echo a.runeOffset(3)  # 4
echo a.runeOffset(4)  # 6
```

---

### `runeReverseOffset(s, rev: Positive): (int, int)`

Returns a tuple `(byteOffset, totalRunes)` where `byteOffset` is the byte position of the rune counted `rev` positions from the end (1-based). If there are not enough runes, `byteOffset` will be negative.

> ⚠️ **Slow operation.**

```nim
import std/unicode

let a = "añyóng"
let (offset, total) = a.runeReverseOffset(1)
echo offset  # byte offset of the last rune
echo total   # total rune count: 6
```

---

### `graphemeLen(s, i: Natural): Natural`

Returns the number of bytes belonging to the grapheme cluster starting at byte index `i`, including any following combining code units.

```nim
import std/unicode

let a = "añyóng"
echo a.graphemeLen(1)  # 2 (ñ = base + combining mark)
echo a.graphemeLen(3)  # 1 (y — plain ASCII)
echo a.graphemeLen(4)  # 2 (ó)
```

---

### `lastRune(s, last: int): (Rune, int)`

Returns the tuple `(rune, byteLen)` for the last rune in the substring `s[0..last]`.

```nim
import std/unicode

let a = "añyóng"
let (r, size) = a.lastRune(a.high)
echo r     # g
echo size  # 1
```

---

## Rune Access

### `runeAt(s, i: Natural): Rune`

Returns the rune at **byte index** `i`. Does not validate that `i` points to the start of a rune — it may point into the middle of a multi-byte character.

```nim
import std/unicode

let a = "añyóng"
echo a.runeAt(0)  # a
echo a.runeAt(1)  # ñ  (start of 2-byte character)
echo a.runeAt(2)  # ñ  (second byte of same character)
echo a.runeAt(3)  # y
```

---

### `runeAtPos(s, pos: int): Rune`

Returns the rune at **rune index** `pos` (not byte index).

> ⚠️ **Slow operation** — walks the string from the beginning on every call.

```nim
import std/unicode

let a = "añyóng"
echo a.runeAtPos(0)  # a
echo a.runeAtPos(1)  # ñ
echo a.runeAtPos(3)  # ó
```

---

### `runeStrAtPos(s, pos: Natural): string`

Returns the rune at rune index `pos` as a UTF-8 string.

> ⚠️ **Slow operation.**

```nim
import std/unicode

let a = "añyóng"
echo a.runeStrAtPos(1)  # "ñ"
echo a.runeStrAtPos(3)  # "ó"
```

---

### `runeSubStr(s, pos: int, len: int = int.high): string`

Returns a substring starting at rune index `pos` with `len` runes.

- Negative `pos` and `len` count from the end of the string.
- Omitting `len` means "to the end of the string."

```nim
import std/unicode

let s = "Hänsel  ««: 10,00€"
echo s.runeSubStr(0, 2)   # "Hä"
echo s.runeSubStr(10, 1)  # ":"
echo s.runeSubStr(-6)     # "10,00€"
echo s.runeSubStr(10)     # ": 10,00€"
echo s.runeSubStr(12, 5)  # "10,00"
echo s.runeSubStr(-6, 3)  # "10,"
```

---

### `toRunes(s): seq[Rune]`

Converts a string into a sequence of runes. Enables O(1) rune-indexed access.

```nim
import std/unicode

let a = toRunes("aáä")
echo a[0]  # a
echo a[1]  # á
echo a[2]  # ä
echo a.len # 3
```

---

## Validation

### `validateUtf8(s): int`

Checks whether the string contains valid UTF-8 data.

- Returns `-1` if the string is valid.
- Returns the byte index of the first invalid byte otherwise.

```nim
import std/unicode

echo validateUtf8("añyóng")   # -1 (valid)
echo validateUtf8("abc\xff")  # 3  (bad byte at index 3)
echo validateUtf8("")         # -1
```

---

## Rune Classification

### `isLower(c: Rune): bool`

Returns `true` if the rune is a lowercase letter.

> `isLower` is slightly more efficient than `isUpper` and should be preferred when both would work.

```nim
import std/unicode

echo "a".runeAt(0).isLower  # true
echo "A".runeAt(0).isLower  # false
echo "α".runeAt(0).isLower  # true (Greek lowercase)
```

---

### `isUpper(c: Rune): bool`

Returns `true` if the rune is an uppercase letter.

```nim
import std/unicode

echo "A".runeAt(0).isUpper  # true
echo "Γ".runeAt(0).isUpper  # true (Greek uppercase)
echo "a".runeAt(0).isUpper  # false
```

---

### `isAlpha(c: Rune): bool`

Returns `true` if the rune is an alphabetic character (letter from any script).

```nim
import std/unicode

echo "a".runeAt(0).isAlpha  # true
echo "ñ".runeAt(0).isAlpha  # true
echo "1".runeAt(0).isAlpha  # false
echo " ".runeAt(0).isAlpha  # false
```

---

### `isAlpha(s): bool`

Returns `true` if **all** characters in the string are alphabetic. An empty string returns `false`.

```nim
import std/unicode

echo "añyóng".isAlpha  # true
echo "abc123".isAlpha  # false
echo "".isAlpha        # false
```

---

### `isTitle(c: Rune): bool`

Returns `true` if the rune is a Unicode titlecase character — a special category where the code point is simultaneously considered both upper and lower case (e.g. `ǅ`).

```nim
import std/unicode

echo "A".runeAt(0).isTitle  # false (regular uppercase)
```

---

### `isWhiteSpace(c: Rune): bool`

Returns `true` if the rune is a Unicode whitespace character (space, tab, newline, etc.).

```nim
import std/unicode

echo " ".runeAt(0).isWhiteSpace   # true
echo "\t".runeAt(0).isWhiteSpace  # true
echo "a".runeAt(0).isWhiteSpace   # false
```

---

### `isSpace(s): bool`

Returns `true` if **all** characters in the string are whitespace. An empty string returns `false`.

```nim
import std/unicode

echo "\t\n \r".isSpace  # true
echo "  a  ".isSpace    # false
echo "".isSpace         # false
```

---

### `isCombining(c: Rune): bool`

Returns `true` if the rune is a Unicode combining character (diacritical marks, superscripts, etc.).

Optimized to return `false` immediately for ASCII code points.

```nim
import std/unicode

# U+0301 — COMBINING ACUTE ACCENT
let combining = Rune(0x0301)
echo combining.isCombining        # true
echo "a".runeAt(0).isCombining    # false
```

---

## Case Conversion

### `toLower(c: Rune): Rune`

Converts a rune to lowercase.

```nim
import std/unicode

echo "A".runeAt(0).toLower  # a
echo "Γ".runeAt(0).toLower  # γ
echo "a".runeAt(0).toLower  # a (already lowercase)
```

---

### `toUpper(c: Rune): Rune`

Converts a rune to uppercase.

> Prefer `toLower` over `toUpper` when possible — it is slightly more efficient.

```nim
import std/unicode

echo "a".runeAt(0).toUpper  # A
echo "γ".runeAt(0).toUpper  # Γ
```

---

### `toTitle(c: Rune): Rune`

Converts a rune to titlecase form.

```nim
import std/unicode

let r = "a".runeAt(0).toTitle
echo r  # A (for most characters, same result as toUpper)
```

---

### `toLower(s): string`

Returns a new string with all runes converted to lowercase.

```nim
import std/unicode

echo toLower("ABΓ")       # "abγ"
echo "ÄÖÜ".toLower        # "äöü"
```

---

### `toUpper(s): string`

Returns a new string with all runes converted to uppercase.

```nim
import std/unicode

echo toUpper("abγ")       # "ABΓ"
echo "hello world".toUpper  # "HELLO WORLD"
```

---

### `swapCase(s): string`

Returns a new string with the case of each rune swapped.

```nim
import std/unicode

echo swapCase("Αlpha Βeta Γamma")  # "αLPHA βETA γAMMA"
echo swapCase("Hello")             # "hELLO"
```

---

### `capitalize(s): string`

Returns a new string with the first rune converted to uppercase, all others unchanged.

```nim
import std/unicode

echo capitalize("βeta")   # "Βeta"
echo capitalize("hello")  # "Hello"
echo capitalize("")        # ""
```

---

### `title(s): string`

Returns a new string in title case: the first letter of each whitespace-separated word is uppercased.

```nim
import std/unicode

echo title("αlpha βeta γamma")  # "Αlpha Βeta Γamma"
echo title("hello world")       # "Hello World"
```

---

## String Operations

### `repeat(c: Rune, count: Natural): string`

Returns a string consisting of `count` repetitions of rune `c`.

```nim
import std/unicode

let n = "ñ".runeAt(0)
echo n.repeat(5)  # "ñññññ"
echo n.repeat(0)  # ""
```

---

### `reversed(s): string`

Returns a copy of `s` with runes in reverse order. Combining characters are handled correctly — they remain attached to their base characters.

```nim
import std/unicode

echo reversed("Reverse this!")  # "!siht esreveR"
echo reversed("先秦兩漢")        # "漢兩秦先"
echo reversed("as⃝df̅")         # "f̅ds⃝a"  (combining handled correctly)
echo reversed("a⃞b⃞c⃞")        # "c⃞b⃞a⃞"
```

---

### `translate(s, replacements: proc(key: string): string): string`

Replaces each whitespace-delimited word in `s` by passing it through the `replacements` callback.

```nim
import std/unicode

proc wordToNumber(s: string): string =
  case s
  of "one": "1"
  of "two": "2"
  else: s

let a = "one two three four"
echo a.translate(wordToNumber)  # "1 2 three four"
```

---

### `strip(s, leading = true, trailing = true, runes = unicodeSpaces): string`

Removes the specified runes from the beginning and/or end of `s`.

- `leading = true` — strip from the beginning.
- `trailing = true` — strip from the end.
- `runes` — the set of runes to strip; defaults to all Unicode whitespace.

```nim
import std/unicode

let a = "\táñyóng   "
echo a.strip                    # "áñyóng"
echo a.strip(leading = false)   # "\táñyóng"
echo a.strip(trailing = false)  # "áñyóng   "

# Custom rune stripping
let b = "***hello***"
echo b.strip(runes = @["*".runeAt(0)])  # "hello"
```

---

### `align(s, count: Natural, padding = ' '.Rune): string`

Right-aligns the string to a rune-length of `count`, prepending `padding` characters. If the string is already `>= count` runes long, it is returned unchanged.

```nim
import std/unicode

echo align("abc", 4)               # " abc"
echo align("1232", 6)              # "  1232"
echo align("1232", 6, '#'.Rune)   # "##1232"
echo align("Åge", 5)              # "  Åge"
echo align("×", 4, '_'.Rune)     # "___×"
```

---

### `alignLeft(s, count: Natural, padding = ' '.Rune): string`

Left-aligns the string to a rune-length of `count`, appending `padding` characters after it.

```nim
import std/unicode

echo alignLeft("abc", 4)              # "abc "
echo alignLeft("1232", 6)            # "1232  "
echo alignLeft("1232", 6, '#'.Rune)  # "1232##"
echo alignLeft("Åge", 5)             # "Åge  "
echo alignLeft("×", 4, '_'.Rune)    # "×___"
```

---

### `cmpRunesIgnoreCase(a, b): int`

Compares two UTF-8 strings case-insensitively.

- Returns `0` if `a == b`.
- Returns `< 0` if `a < b` lexicographically.
- Returns `> 0` if `a > b` lexicographically.

```nim
import std/unicode

echo cmpRunesIgnoreCase("Hello", "hello")  # 0
echo cmpRunesIgnoreCase("abc", "abd")      # < 0
echo cmpRunesIgnoreCase("Ä", "ä")         # 0
```

---

### `split(s, seps: openArray[Rune] = unicodeSpaces, maxsplit = -1): seq[string]`

Splits the string by any rune in `seps`. Returns a sequence of substrings.

- `seps` — separator runes; defaults to all Unicode whitespace.
- `maxsplit` — maximum number of splits; `-1` means unlimited.

```nim
import std/unicode

# By whitespace
echo "hello world  nim".split()
# @["hello", "world", "nim"]

# By custom separators
echo split("añyóng:hÃllo;是", ";:".toRunes)
# @["añyóng", "hÃllo", "是"]

# Date splitting
echo split("2012-11-20T22:08", " -:T".toRunes)
# @["2012", "11", "20", "22", "08"]
```

---

### `split(s, sep: Rune, maxsplit = -1): seq[string]`

Splits the string by a single rune separator.

```nim
import std/unicode

echo split("a;b;;c", ";".runeAt(0))
# @["a", "b", "", "c"]
```

---

### `splitWhitespace(s): seq[string]`

Splits the string at Unicode whitespace boundaries. Equivalent to `split()` with default arguments but returns `seq[string]` directly.

```nim
import std/unicode

echo splitWhitespace("hello  world\tnim")
# @["hello", "world", "nim"]
```

---

## Iterators

### `iterator runes(s): Rune`

Iterates over each rune of the string in order.

```nim
import std/unicode

for r in runes("añyóng"):
  echo r
# a
# ñ
# y
# ó
# n
# g
```

---

### `iterator utf8(s): string`

Iterates over each rune of the string, yielding each one as a UTF-8 string.

```nim
import std/unicode

for c in utf8("añyóng"):
  echo c, " (", c.len, " byte(s))"
# a (1 byte(s))
# ñ (2 byte(s))
# y (1 byte(s))
# ó (2 byte(s))
# n (1 byte(s))
# g (1 byte(s))
```

---

### `iterator split(s, seps, maxsplit)` / `iterator split(s, sep, maxsplit)`

Iterator versions of `split`. Do not allocate an intermediate sequence — more memory-efficient for large strings.

```nim
import std/unicode

for part in split("a:b:c", ":".runeAt(0)):
  echo part
# a
# b
# c
```

---

### `iterator splitWhitespace(s)`

Iterator version of `splitWhitespace`.

```nim
import std/unicode

for word in splitWhitespace("hello   world"):
  echo word
# hello
# world
```

---

## Comparison Operators

### `==(a, b: Rune): bool`

Checks whether two runes have the same Unicode code point.

```nim
import std/unicode

echo "a".runeAt(0) == "a".runeAt(0)  # true
echo "a".runeAt(0) == "b".runeAt(0)  # false
```

---

### `<%(a, b: Rune): bool`

Unsigned less-than comparison of rune code points.

```nim
import std/unicode

let a = "ú".runeAt(0)
let b = "ü".runeAt(0)
echo a <% b  # true
```

---

### `<=%(a, b: Rune): bool`

Unsigned less-than-or-equal comparison of rune code points.

```nim
import std/unicode

let a = "ú".runeAt(0)
let b = "ü".runeAt(0)
echo a <=% b  # true
echo b <=% b  # true
```

---

## Complete Examples

### Counting characters in a multilingual string

```nim
import std/unicode

let text = "Hello, мир! 你好"
echo "Bytes: ", text.len      # 20+
echo "Runes: ", text.runeLen  # 13 characters
```

---

### Iterating and filtering by character type

```nim
import std/unicode

let s = "Hello, Unicode! 42"
var letters, spaces, other: int

for r in runes(s):
  if r.isAlpha: inc letters
  elif r.isWhiteSpace: inc spaces
  else: inc other

echo "Letters: ", letters
echo "Spaces:  ", spaces
echo "Other:   ", other
```

---

### Reversing a string with combining marks

```nim
import std/unicode

# Combining characters stay attached to their base characters
let s = "as⃝df̅"
echo reversed(s)  # "f̅ds⃝a"
```

---

### Building a column-aligned table

```nim
import std/unicode

let words = ["Åge", "先秦", "a", "Unicode"]
for w in words:
  echo alignLeft(w, 12) & "|" & align($w.runeLen, 5)
# Åge         |    3
# 先秦          |    2
# a           |    1
# Unicode     |    7
```

---

### Case-insensitive search across scripts

```nim
import std/unicode

proc containsCI(text, pattern: string): bool =
  toLower(text).find(toLower(pattern)) >= 0

echo containsCI("Привет МИР", "мир")  # true
echo containsCI("Ärger", "ärger")    # true
```

---

### Safe position-based rune access

```nim
import std/unicode

let s = "Hänsel"

# RECOMMENDED — convert once, then index in O(1)
let runes = toRunes(s)
echo runes[1]           # ä
echo runes.len          # 6

# SLOWER — for one-off accesses
echo s.runeAtPos(1)     # ä
echo s.runeSubStr(1, 3) # "äns"
```

---

### Validating user input

```nim
import std/unicode

proc checkInput(s: string) =
  let badByte = validateUtf8(s)
  if badByte >= 0:
    echo "Invalid UTF-8 at byte ", badByte
  elif not s.isAlpha:
    echo "Input must contain only letters"
  else:
    echo "Valid: ", s

checkInput("hello")     # Valid: hello
checkInput("abc\xff")   # Invalid UTF-8 at byte 3
checkInput("hello123")  # Input must contain only letters
```
