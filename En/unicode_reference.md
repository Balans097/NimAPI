# `unicode` Module Reference

> **Nim Standard Library — UTF-8 / Unicode Support**
> Compatible with **Unicode v12.0.0**.

---

## Overview

The `unicode` module is the standard way to work with **Unicode text** in Nim. It operates on UTF-8 encoded strings — the same `string` type used everywhere in Nim — but treats them as sequences of **Runes** (Unicode code points) rather than raw bytes.

The central insight you need before using this module is the difference between three distinct "lengths" of a string:

```
s = "añyóng"

s.len      == 8   # raw bytes (a=1, ñ=2, y=1, ó=2, n=1, g=1)
s.runeLen  == 6   # Unicode code points (characters as humans count them)
s.graphemeLen(1) == 2  # bytes the grapheme at byte-index 1 occupies (ñ)
```

Because UTF-8 is a variable-width encoding, naive byte indexing into a Unicode string yields garbage for non-ASCII text. This module provides all the tools you need to work correctly with any language.

```nim
import std/unicode
```

---

## The `Rune` Type

```nim
type Rune* = distinct int32
```

A `Rune` holds a single Unicode **code point** — an integer in the range U+0000 to U+10FFFF. It is a distinct type from `int32`, so you cannot accidentally mix it with plain integers.

Every character visible on screen may be composed of one *base* Rune followed by zero or more *combining* Runes (diacritics, modifiers). For example, `ñ` in NFD form is the base rune `n` (U+006E) followed by the combining tilde (U+0303).

**Creating Runes:**

```nim
let r1 = "ñ".runeAt(0)     # from a string at byte index 0
let r2 = Rune(0x00F1)      # from a code point value
let r3 = 'n'.Rune          # from a char (ASCII only)
```

---

## Measurement and Navigation

### `runeLen`

```nim
proc runeLen*(s: openArray[char]): int
proc runeLen*(s: string): int
```

Returns the number of **Unicode code points** (Runes) in `s`. This is the "human character count" — not the byte length.

```nim
let s = "añyóng"
assert s.len     == 8  # bytes
assert s.runeLen == 6  # runes
assert "€".runeLen == 1  # one rune, but 3 bytes
```

---

### `runeLenAt`

```nim
proc runeLenAt*(s: openArray[char], i: Natural): int
proc runeLenAt*(s: string, i: Natural): int
```

Returns how many **bytes** the Rune starting at **byte index** `i` occupies (1–6). This is the key tool for advancing a byte pointer by exactly one code point.

```nim
let s = "añyóng"
assert s.runeLenAt(0) == 1  # 'a'  is ASCII → 1 byte
assert s.runeLenAt(1) == 2  # 'ñ'  is U+00F1 → 2 bytes
assert s.runeLenAt(3) == 1  # 'y'  is ASCII → 1 byte
assert s.runeLenAt(4) == 2  # 'ó'  is U+00F3 → 2 bytes
```

---

### `size`

```nim
proc size*(r: Rune): int
```

Returns how many bytes the Rune `r` occupies when encoded as UTF-8 (1–6).

```nim
let runes = toRunes("aá€𐍈")
assert runes[0].size == 1  # ASCII
assert runes[1].size == 2  # Latin extended
assert runes[2].size == 3  # Euro sign U+20AC
assert runes[3].size == 4  # Gothic letter U+10348
```

---

### `graphemeLen`

```nim
proc graphemeLen*(s: openArray[char]; i: Natural): Natural
proc graphemeLen*(s: string; i: Natural): Natural
```

Returns the number of **bytes** belonging to the grapheme cluster that starts at byte index `i`. A grapheme cluster is what a human perceives as a single "character" — a base Rune plus any following combining Runes.

```nim
# "añyóng" — ñ and ó each consist of a base + combining rune in NFD,
# but here they are stored as precomposed forms (NFC):
let s = "añyóng"
assert s.graphemeLen(0) == 1  # 'a'
assert s.graphemeLen(1) == 2  # 'ñ' (U+00F1, precomposed)
assert s.graphemeLen(3) == 1  # 'y'

# With a combining character:
let t = "e\u0301"  # 'e' + combining acute accent
assert t.graphemeLen(0) == 3  # 'e' + combining = 3 bytes total
```

---

### `runeOffset`

```nim
proc runeOffset*(s: openArray[char], pos: Natural, start: Natural = 0): int
proc runeOffset*(s: string, pos: Natural, start: Natural = 0): int
```

Translates **rune index** `pos` (counting from the start, or from `start` byte offset) to a **byte index** in `s`. Returns `-1` if `pos` exceeds the number of runes.

> ⚠️ This is O(n) — it must scan from the beginning. For repeated access, convert to `seq[Rune]` first.

```nim
let s = "añyóng"
# Rune 0='a'(byte 0), 1='ñ'(byte 1), 2='y'(byte 3), 3='ó'(byte 4), ...
assert s.runeOffset(0) == 0
assert s.runeOffset(1) == 1
assert s.runeOffset(2) == 3
assert s.runeOffset(3) == 4
assert s.runeOffset(6) == -1  # out of range
```

---

### `runeReverseOffset`

```nim
proc runeReverseOffset*(s: openArray[char], rev: Positive): (int, int)
proc runeReverseOffset*(s: string, rev: Positive): (int, int)
```

Counts `rev` code points **from the end** of `s` (1-based: `rev=1` is the last rune). Returns a tuple `(byteOffset, totalRuneLen)`. If `rev` is larger than the rune count, `byteOffset` is negative.

> ⚠️ O(n) — same caveat as `runeOffset`.

```nim
let s = "añyóng"  # 6 runes
let (offset, total) = s.runeReverseOffset(1)  # last rune 'g'
assert total == 6
# offset points to byte where 'g' starts
```

---

### `lastRune`

```nim
proc lastRune*(s: openArray[char]; last: int): (Rune, int)
proc lastRune*(s: string; last: int): (Rune, int)
```

Returns the last Rune in `s[0..last]` and its byte length. Useful for scanning backwards without converting the whole string.

```nim
let s = "añ"
let (rune, byteLen) = s.lastRune(s.high)
assert $rune == "ñ"
assert byteLen == 2
```

---

## Reading Runes

### `runeAt`

```nim
proc runeAt*(s: openArray[char], i: Natural): Rune
proc runeAt*(s: string, i: Natural): Rune
```

Returns the Rune starting at **byte index** `i`. This is the low-level, fast accessor — `i` is a *byte* position, not a rune position.

```nim
let s = "añyóng"
assert s.runeAt(0) == "a".runeAt(0)  # byte 0 → 'a'
assert s.runeAt(1) == "ñ".runeAt(0)  # byte 1 → 'ñ' (2 bytes long)
assert s.runeAt(3) == "y".runeAt(0)  # byte 3 → 'y'
# Calling runeAt(2) gives you the second byte of 'ñ' — a continuation byte.
# Use runeLenAt to advance correctly.
```

---

### `runeAtPos`

```nim
proc runeAtPos*(s: openArray[char], pos: int): Rune
proc runeAtPos*(s: string, pos: int): Rune
```

Returns the Rune at **rune index** `pos` (0-based, counted in code points). More convenient than `runeAt` but O(n).

> ⚠️ Slow for repeated access — prefer iterating with `runes` or converting to `seq[Rune]`.

```nim
let s = "añyóng"
assert s.runeAtPos(0) == "a".runeAt(0)
assert s.runeAtPos(1) == "ñ".runeAt(0)
assert s.runeAtPos(2) == "y".runeAt(0)
```

---

### `runeStrAtPos`

```nim
proc runeStrAtPos*(s: openArray[char], pos: Natural): string
proc runeStrAtPos*(s: string, pos: Natural): string
```

Returns the Rune at rune index `pos` as a **UTF-8 string** (rather than a `Rune` value). Useful when you need the string form directly.

> ⚠️ O(n) — same caveat as `runeAtPos`.

```nim
let s = "añyóng"
assert s.runeStrAtPos(1) == "ñ"
assert s.runeStrAtPos(3) == "ó"
```

---

### `fastRuneAt` (template)

```nim
template fastRuneAt*(s: openArray[char] or string, i: int, result: untyped, doInc = true)
```

The **fastest** way to read a single Rune from position `i`. Decodes the Rune into `result` and, if `doInc` is true, advances `i` by the number of bytes consumed. Invalid byte sequences yield the Unicode replacement character U+FFFD.

This template is inline — no function call overhead — and is used internally by all other rune-reading procedures. Use it in performance-critical scanning loops.

```nim
var s = "añy"
var i = 0
var r: Rune

fastRuneAt(s, i, r)          # r = 'a', i = 1
fastRuneAt(s, i, r)          # r = 'ñ', i = 3
fastRuneAt(s, i, r)          # r = 'y', i = 4

# Read without advancing:
i = 1
fastRuneAt(s, i, r, doInc = false)  # r = 'ñ', i still = 1
```

---

## Substring Extraction

### `runeSubStr`

```nim
proc runeSubStr*(s: openArray[char], pos: int, len: int = int.high): string
proc runeSubStr*(s: string, pos: int, len: int = int.high): string
```

Returns a substring beginning at **rune index** `pos`, spanning `len` runes. Both `pos` and `len` may be **negative**, in which case they count from the end of the string. Omitting `len` means "to the end of the string".

```nim
let s = "Hänsel  ««: 10,00€"

assert s.runeSubStr(0, 2)   == "Hä"     # runes 0..1
assert s.runeSubStr(10, 1)  == ":"      # rune 10
assert s.runeSubStr(-6)     == "10,00€" # last 6 runes to end
assert s.runeSubStr(10)     == ": 10,00€" # from rune 10 to end
assert s.runeSubStr(12, 5)  == "10,00"  # 5 runes from rune 12
assert s.runeSubStr(-6, 3)  == "10,"    # 3 runes starting 6 from end
```

---

## Conversion Between Rune and String

### `toUTF8`

```nim
proc toUTF8*(c: Rune): string
```

Converts a single Rune to its UTF-8 string representation.

```nim
let r = "ñ".runeAt(0)
assert r.toUTF8 == "ñ"
assert Rune(0x20AC).toUTF8 == "€"
```

---

### `$` (Rune → string)

```nim
proc `$`*(rune: Rune): string
```

An alias for `toUTF8`. Allows `echo rune` and string interpolation to work naturally.

```nim
let r = Rune(0x03B1)  # α
echo $r    # "α"
echo "Letter: " & $r  # "Letter: α"
```

---

### `$` (seq[Rune] → string)

```nim
proc `$`*(runes: seq[Rune]): string
```

Converts a sequence of Runes back to a UTF-8 string. The reverse operation of `toRunes`.

```nim
let runes = @[Rune('H'), "ä".runeAt(0), Rune('i')]
assert $runes == "Häi"
```

---

### `add`

```nim
proc add*(s: var string; c: Rune)
```

Appends a single Rune `c` to string `s`. More efficient than `s &= $c` as it avoids an intermediate allocation.

```nim
var s = "abc"
s.add("ä".runeAt(0))
s.add(Rune(0x20AC))  # Euro sign
assert s == "abcä€"
```

---

### `toRunes`

```nim
proc toRunes*(s: openArray[char]): seq[Rune]
proc toRunes*(s: string): seq[Rune]
```

Decodes the entire string into a `seq[Rune]`. Once converted, rune indexing is O(1) — useful when you need to index or modify many runes.

```nim
let runes = toRunes("aáä")
assert runes[0] == "a".runeAt(0)
assert runes[1] == "á".runeAt(0)
assert runes[2] == "ä".runeAt(0)
assert runes.len == 3

# Round-trip
assert $runes == "aáä"
```

---

### `fastToUTF8Copy` (template)

```nim
template fastToUTF8Copy*(c: Rune, s: var string, pos: int, doInc = true)
```

Copies the UTF-8 bytes of Rune `c` into the **preallocated** string `s` at byte position `pos`. If `doInc` is true, `pos` is advanced past the written bytes. This is the high-performance building block for string construction — used internally by `toUTF8`, `add`, and conversion procs.

```nim
var buf = newString(3)  # preallocate for one 3-byte rune (€)
var pos = 0
fastToUTF8Copy(Rune(0x20AC), buf, pos)
assert buf == "€"
assert pos == 3
```

---

## Validation

### `validateUtf8`

```nim
proc validateUtf8*(s: openArray[char]): int
proc validateUtf8*(s: string): int
```

Scans `s` for invalid UTF-8 byte sequences. Returns the **byte index** of the first invalid byte, or `-1` if the string is valid. Also catches overlong encodings (a security concern in old parsers).

```nim
assert validateUtf8("hello") == -1           # valid ASCII
assert validateUtf8("añyóng") == -1          # valid UTF-8
assert validateUtf8("hello\xFF") == 5        # 0xFF at byte 5 is invalid
```

---

## Iterators

### `runes`

```nim
iterator runes*(s: openArray[char]): Rune
iterator runes*(s: string): Rune
```

Iterates over all Runes in `s`. This is the **idiomatic, efficient** way to visit every code point — prefer it over indexed access.

```nim
let s = "añy"
for r in s.runes:
  echo $r   # prints: a  ñ  y

# Collect into a sequence:
import std/sequtils
let runes = toSeq(s.runes)
```

---

### `utf8`

```nim
iterator utf8*(s: openArray[char]): string
iterator utf8*(s: string): string
```

Like `runes`, but yields each code point as a **string** (its UTF-8 byte sequence) rather than as a `Rune` value.

```nim
for ch in "añy".utf8:
  echo ch, " (", ch.len, " bytes)"
# a (1 bytes)
# ñ (2 bytes)
# y (1 bytes)
```

---

### `split` (iterator, multiple separators)

```nim
iterator split*(s: openArray[char], seps: openArray[Rune] = unicodeSpaces,
                maxsplit: int = -1): string
iterator split*(s: string, seps: openArray[Rune] = unicodeSpaces,
                maxsplit: int = -1): string
```

Splits `s` at any Rune found in `seps`, yielding substrings. The default separator set is `unicodeSpaces` — all Unicode whitespace characters, not just ASCII space. `maxsplit` limits the number of splits (-1 = unlimited).

```nim
import std/sequtils

# Default: split on Unicode whitespace
assert toSeq("hello world\t是".split) == @["hello", "world", "是"]

# Custom separators
assert toSeq(split("añyóng:hÃllo;是", ";:".toRunes)) ==
  @["añyóng", "hÃllo", "是"]
```

---

### `split` (iterator, single separator)

```nim
iterator split*(s: openArray[char], sep: Rune, maxsplit: int = -1): string
iterator split*(s: string, sep: Rune, maxsplit: int = -1): string
```

Like the multi-separator version, but splits on a single `sep` Rune. Consecutive separators produce empty strings between them.

```nim
import std/sequtils

let parts = toSeq(split("a;;b;c", ";".runeAt(0)))
assert parts == @["a", "", "b", "c"]
```

---

### `splitWhitespace` (iterator)

```nim
iterator splitWhitespace*(s: openArray[char]): string
iterator splitWhitespace*(s: string): string
```

Splits `s` on any Unicode whitespace, equivalent to `split(s, unicodeSpaces)`. Multiple consecutive whitespace characters are treated as a single separator.

```nim
for word in "  hello\t世界  ".splitWhitespace:
  echo word   # "hello", "世界"
```

---

## Procedures Returning Sequences

### `split` (proc, multiple separators)

```nim
proc split*(s: openArray[char], seps: openArray[Rune] = unicodeSpaces,
            maxsplit: int = -1): seq[string]
proc split*(s: string, seps: openArray[Rune] = unicodeSpaces,
            maxsplit: int = -1): seq[string]
```

Same as the `split` iterator but returns a `seq[string]` instead of yielding.

```nim
let parts = "a,b,c".split(",".toRunes)
assert parts == @["a", "b", "c"]
```

---

### `split` (proc, single separator)

```nim
proc split*(s: openArray[char], sep: Rune, maxsplit: int = -1): seq[string]
proc split*(s: string, sep: Rune, maxsplit: int = -1): seq[string]
```

```nim
let parts = "a::b:c".split(":".runeAt(0))
assert parts == @["a", "", "b", "c"]
```

---

### `splitWhitespace` (proc)

```nim
proc splitWhitespace*(s: openArray[char]): seq[string]
proc splitWhitespace*(s: string): seq[string]
```

Returns a `seq[string]` of words split on Unicode whitespace.

```nim
assert "  hello\t世界  ".splitWhitespace == @["hello", "世界"]
```

---

## Case Conversion (single Rune)

### `toLower` (Rune)

```nim
proc toLower*(c: Rune): Rune
```

Returns the lowercase form of Rune `c` according to Unicode case-folding tables. If `c` has no lowercase form (e.g., digits, punctuation), `c` is returned unchanged.

> Prefer `toLower` over `toUpper` for case-insensitive comparisons — Unicode defines more reliable lowercase mappings.

```nim
assert toLower("A".runeAt(0)) == "a".runeAt(0)
assert toLower("Γ".runeAt(0)) == "γ".runeAt(0)
assert toLower("1".runeAt(0)) == "1".runeAt(0)  # unchanged
```

---

### `toUpper` (Rune)

```nim
proc toUpper*(c: Rune): Rune
```

Returns the uppercase form of Rune `c`.

```nim
assert toUpper("a".runeAt(0)) == "A".runeAt(0)
assert toUpper("γ".runeAt(0)) == "Γ".runeAt(0)
```

---

### `toTitle` (Rune)

```nim
proc toTitle*(c: Rune): Rune
```

Converts `c` to its **titlecase** form. Titlecase is a distinct Unicode category used for certain digraph characters (e.g., `ǲ` → `ǲ` stays as-is in most cases; `dz` → `Dz`). For most runes, `toTitle` behaves the same as `toUpper`.

```nim
assert toTitle("a".runeAt(0)) == "A".runeAt(0)
```

---

## Case Conversion (whole string)

### `toUpper` (string)

```nim
proc toUpper*(s: openArray[char]): string
proc toUpper*(s: string): string
```

Returns a new string with every Rune in `s` converted to uppercase.

```nim
assert toUpper("abγ") == "ABΓ"
assert toUpper("héllo") == "HÉLLO"
```

---

### `toLower` (string)

```nim
proc toLower*(s: openArray[char]): string
proc toLower*(s: string): string
```

Returns a new string with every Rune converted to lowercase.

```nim
assert toLower("ABΓ") == "abγ"
assert toLower("HÉLLO") == "héllo"
```

---

### `swapCase`

```nim
proc swapCase*(s: openArray[char]): string
proc swapCase*(s: string): string
```

Returns a new string where uppercase Runes become lowercase and vice versa. Runes without a case (digits, punctuation, CJK) are left unchanged.

```nim
assert swapCase("Αlpha Βeta Γamma") == "αLPHA βETA γAMMA"
assert swapCase("Hello, World!") == "hELLO, wORLD!"
```

---

### `capitalize`

```nim
proc capitalize*(s: openArray[char]): string
proc capitalize*(s: string): string
```

Converts the **first rune** of `s` to uppercase and leaves the rest of the string unchanged.

```nim
assert capitalize("βeta") == "Βeta"
assert capitalize("hello world") == "Hello world"  # only first rune
```

---

### `title`

```nim
proc title*(s: openArray[char]): string
proc title*(s: string): string
```

Returns a new string where the **first rune of each whitespace-separated word** is converted to uppercase (title case). The rest of each word is left unchanged.

```nim
assert title("αlpha βeta γamma") == "Αlpha Βeta Γamma"
assert title("the quick brown fox") == "The Quick Brown Fox"
```

---

## Classification (single Rune)

### `isLower`

```nim
proc isLower*(c: Rune): bool
```

Returns `true` if `c` is a Unicode lowercase letter.

> Prefer `isLower` over `isUpper` for performance (same internal reason as `toLower`).

```nim
assert isLower("a".runeAt(0))
assert isLower("γ".runeAt(0))
assert not isLower("A".runeAt(0))
assert not isLower("1".runeAt(0))
```

---

### `isUpper`

```nim
proc isUpper*(c: Rune): bool
```

Returns `true` if `c` is a Unicode uppercase letter.

```nim
assert isUpper("A".runeAt(0))
assert isUpper("Γ".runeAt(0))
assert not isUpper("a".runeAt(0))
```

---

### `isAlpha` (Rune)

```nim
proc isAlpha*(c: Rune): bool
```

Returns `true` if `c` is a Unicode **alphabetic** character — any letter in any script, not just Latin.

```nim
assert isAlpha("a".runeAt(0))   # Latin
assert isAlpha("α".runeAt(0))   # Greek
assert isAlpha("中".runeAt(0))  # CJK
assert not isAlpha("1".runeAt(0))
assert not isAlpha(" ".runeAt(0))
```

---

### `isTitle`

```nim
proc isTitle*(c: Rune): bool
```

Returns `true` if `c` is a Unicode titlecase code point (a narrow category of ligature characters, e.g., `ǅ`).

---

### `isWhiteSpace`

```nim
proc isWhiteSpace*(c: Rune): bool
```

Returns `true` if `c` is a Unicode whitespace character. Covers all Unicode spaces — not just ASCII space, tab, and newline, but also non-breaking space, ideographic space, etc.

```nim
assert isWhiteSpace(" ".runeAt(0))
assert isWhiteSpace("\t".runeAt(0))
assert isWhiteSpace("\u00A0".runeAt(0))   # non-breaking space
assert not isWhiteSpace("a".runeAt(0))
```

---

### `isCombining`

```nim
proc isCombining*(c: Rune): bool
```

Returns `true` if `c` is a Unicode **combining character** (a diacritic or modifier that attaches to the preceding base character). Ranges covered: U+0300–U+036F, U+1AB0–U+1AFF, U+1DC0–U+1DFF, U+20D0–U+20FF, U+FE20–U+FE2F. Optimised to return `false` immediately for ASCII.

```nim
assert isCombining(Rune(0x0301))  # combining acute accent (◌́)
assert not isCombining("a".runeAt(0))
```

---

## Classification (whole string)

### `isAlpha` (string)

```nim
proc isAlpha*(s: openArray[char]): bool
proc isAlpha*(s: string): bool
```

Returns `true` if every Rune in `s` is alphabetic, and `s` is non-empty.

```nim
assert isAlpha("añyóng")
assert isAlpha("αβγ")
assert not isAlpha("añyóng123")
assert not isAlpha("")
```

---

### `isSpace` (string)

```nim
proc isSpace*(s: openArray[char]): bool
proc isSpace*(s: string): bool
```

Returns `true` if every Rune in `s` is Unicode whitespace, and `s` is non-empty.

```nim
assert isSpace("\t\n \r\f\v")
assert not isSpace("  a  ")
assert not isSpace("")
```

---

## Comparison

### `cmpRunesIgnoreCase`

```nim
proc cmpRunesIgnoreCase*(a, b: openArray[char]): int
proc cmpRunesIgnoreCase*(a, b: string): int
```

Compares two UTF-8 strings **case-insensitively** using Unicode case-folding. Returns `0` if equal, a negative value if `a < b`, and a positive value if `a > b`.

```nim
assert cmpRunesIgnoreCase("abc", "ABC") == 0
assert cmpRunesIgnoreCase("αβγ", "ΑΒΓ") == 0
assert cmpRunesIgnoreCase("a", "b") < 0
```

---

### Rune comparison operators

```nim
proc `==`*(a, b: Rune): bool
proc `<%`*(a, b: Rune): bool   # less than (unsigned code point comparison)
proc `<=%`*(a, b: Rune): bool  # less than or equal (unsigned)
```

Compare two Runes by their Unicode code point value. The `<%` and `<=%` operators use **unsigned** comparison, which matches Unicode ordering for valid code points.

```nim
let a = "ú".runeAt(0)  # U+00FA
let b = "ü".runeAt(0)  # U+00FC
assert a <% b
assert a <=% b
assert a == "ú".runeAt(0)
```

---

## String Manipulation

### `reversed`

```nim
proc reversed*(s: openArray[char]): string
proc reversed*(s: string): string
```

Returns the string `s` with all Runes in reversed order. **Combining characters are treated correctly** — they remain attached to their base character and move with it, so grapheme clusters are not broken apart.

```nim
assert reversed("Hello") == "olleH"
assert reversed("先秦兩漢") == "漢兩秦先"
# Combining characters stay with their base:
assert reversed("as⃝df̅") == "f̅ds⃝a"
assert reversed("a⃞b⃞c⃞") == "c⃞b⃞a⃞"
```

---

### `strip`

```nim
proc strip*(s: openArray[char], leading = true, trailing = true,
            runes: openArray[Rune] = unicodeSpaces): string
proc strip*(s: string, leading = true, trailing = true,
            runes: openArray[Rune] = unicodeSpaces): string
```

Removes leading and/or trailing Runes from `s`. By default strips all Unicode whitespace from both ends. Pass a custom `runes` array to strip specific characters. Set `leading = false` or `trailing = false` to strip only one end.

```nim
assert strip("\t áñyóng   ") == "áñyóng"
assert strip("\t áñyóng   ", leading = false) == "\t áñyóng"
assert strip("\t áñyóng   ", trailing = false) == "áñyóng   "

# Strip custom runes
let cutRunes = "*".toRunes
assert strip("**hello**", runes = cutRunes) == "hello"
```

---

### `repeat`

```nim
proc repeat*(c: Rune, count: Natural): string
```

Returns a string consisting of `count` copies of Rune `c`.

```nim
assert repeat("ñ".runeAt(0), 5) == "ñññññ"
assert repeat(Rune(0x2665), 3) == "♥♥♥"
```

---

### `align`

```nim
proc align*(s: openArray[char], count: Natural, padding = ' '.Rune): string
proc align*(s: string, count: Natural, padding = ' '.Rune): string
```

**Right-aligns** `s` in a field of `count` runes by prepending `padding` characters. If `s.runeLen >= count`, returns `s` unchanged. Padding is measured in **runes**, not bytes — so multibyte padding characters work correctly.

```nim
assert align("abc", 6) == "   abc"
assert align("Åge", 5) == "  Åge"
assert align("×", 4, '_'.Rune) == "___×"
assert align("1232", 6, '#'.Rune) == "##1232"
```

---

### `alignLeft`

```nim
proc alignLeft*(s: openArray[char], count: Natural, padding = ' '.Rune): string
proc alignLeft*(s: string, count: Natural, padding = ' '.Rune): string
```

**Left-aligns** `s` in a field of `count` runes by appending `padding` characters.

```nim
assert alignLeft("abc", 6) == "abc   "
assert alignLeft("Åge", 5) == "Åge  "
assert alignLeft("×", 4, '_'.Rune) == "×___"
```

---

### `translate`

```nim
proc translate*(s: openArray[char], replacements: proc(key: string): string): string
proc translate*(s: string, replacements: proc(key: string): string): string
```

Splits `s` into whitespace-separated words, calls `replacements(word)` for each, and reassembles the result preserving original whitespace. The callback receives each word as a UTF-8 string and should return the replacement.

```nim
proc wordToNumber(s: string): string =
  case s
  of "one": "1"
  of "two": "2"
  else: s

assert translate("one two three", wordToNumber) == "1 2 three"

# Practical: translate words to another language
proc en2fr(w: string): string =
  case w
  of "hello": "bonjour"
  of "world": "monde"
  else: w

assert translate("hello world", en2fr) == "bonjour monde"
```

---

## Performance Guide

| Scenario | Best approach |
|----------|--------------|
| Count characters in a string | `s.runeLen` |
| Iterate over all characters | `for r in s.runes` |
| Random access by character index | `toRunes(s)` then index with `[]` |
| Build a string from runes | `var s = ""; s.add(rune)` in a loop |
| Advance by one rune in a scanning loop | `fastRuneAt(s, i, r)` |
| Slice by character positions | `s.runeSubStr(pos, len)` |
| Check if input is valid UTF-8 | `s.validateUtf8 == -1` |
| Case-insensitive comparison | `cmpRunesIgnoreCase(a, b)` |
| Reverse a string preserving graphemes | `s.reversed` |

---

## Quick Reference

```
Goal                                  Procedure / Iterator
─────────────────────────────────────────────────────────────────────
Rune count of string                  runeLen(s)
Byte length of rune at byte i         runeLenAt(s, i)
Byte length of rune value             size(r)
Bytes for grapheme at byte i          graphemeLen(s, i)
Rune at byte index i                  runeAt(s, i)
Rune at rune index pos (slow)         runeAtPos(s, pos)
Rune as string at rune index pos      runeStrAtPos(s, pos)
Rune index → byte offset              runeOffset(s, pos)
Rune from end → byte offset + count   runeReverseOffset(s, rev)
Last rune and its byte length         lastRune(s, last)
Fast rune decode + advance            fastRuneAt(s, i, r)
Rune → UTF-8 string                   toUTF8(r) / $r
Add rune to string                    s.add(r)
String → seq of runes                 toRunes(s)
Seq of runes → string                 $runes
Validate UTF-8                        validateUtf8(s)
Iterate runes                         for r in s.runes
Iterate runes as strings              for ch in s.utf8
Substring by rune positions           runeSubStr(s, pos, len)
Lowercase rune / string               toLower(c) / toLower(s)
Uppercase rune / string               toUpper(c) / toUpper(s)
Titlecase rune                        toTitle(c)
Swap case                             swapCase(s)
Capitalize first rune                 capitalize(s)
Title-case each word                  title(s)
Is lowercase letter                   isLower(c)
Is uppercase letter                   isUpper(c)
Is alphabetic letter                  isAlpha(c) / isAlpha(s)
Is whitespace                         isWhiteSpace(c) / isSpace(s)
Is combining character                isCombining(c)
Compare ignoring case                 cmpRunesIgnoreCase(a, b)
Reverse rune-by-rune                  reversed(s)
Strip whitespace / custom runes       strip(s)
Repeat a rune                         repeat(c, n)
Right-align by rune width             align(s, count)
Left-align by rune width              alignLeft(s, count)
Split on whitespace                   splitWhitespace(s)
Split on separator rune(s)            split(s, sep) / split(s, seps)
Replace words via callback            translate(s, proc)
```
