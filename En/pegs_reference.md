# `pegs` Module Reference (Nim)

> Nim standard library module for **PEG** (Parsing Expression Grammar) matching.  
> Supports Unicode, superoperators for speed, an embedded DSL, and event-driven parsing.

---

## Table of Contents

1. [Overview and PEG Syntax](#overview-and-peg-syntax)
2. [Types](#types)
3. [Creating PEG Objects from a String](#creating-peg-objects-from-a-string)
4. [PEG Constructors (DSL)](#peg-constructors-dsl)
   - [Terminals](#terminals)
   - [Operators (combinators)](#operators-combinators)
   - [Predicates](#predicates)
   - [Captures and Back-references](#captures-and-back-references)
   - [Anchors and Special Symbols](#anchors-and-special-symbols)
   - [Shorthand Templates](#shorthand-templates)
   - [Non-terminals](#non-terminals)
5. [Peg Accessors](#peg-accessors)
6. [NonTerminal Accessors](#nonterminal-accessors)
7. [The `Captures` Type](#the-captures-type)
8. [Search and Matching](#search-and-matching)
9. [Replacement](#replacement)
10. [String Splitting](#string-splitting)
11. [Low-level Matching](#low-level-matching)
12. [Event Parser](#event-parser)
13. [Utilities](#utilities)
14. [Quick Reference](#quick-reference)

---

## Overview and PEG Syntax

PEG (Parsing Expression Grammar) is a formal system for describing languages — an alternative to regular expressions and context-free grammars. Unlike regexps, PEG matching is deterministic (no ambiguity). This module uses no memoization but applies superoperators and symbol inlining to achieve competitive performance.

### PEG String Syntax (for `peg` / `parsePeg`)

| Element | Syntax | Description |
|---------|--------|-------------|
| Literal | `'hello'` | Exact match |
| Case-insensitive | `i'hello'` | Ignores letter case |
| Style-insensitive | `y'hello'` | Ignores `_` / `-` and case |
| Any character | `.` | Any single byte |
| Any rune | `_` | Any Unicode character |
| Character set | `[a-zA-Z]` | Ranges / individual chars |
| Negated set | `[^0-9]` | Excluded characters |
| Group | `(a b c)` | Sequence |
| Ordered choice | `a / b` | Try left first |
| Optional | `a?` | 0 or 1 times |
| Greedy repetition | `a*` | 0 or more times |
| Positive repetition | `a+` | 1 or more times |
| And-predicate | `&a` | Look-ahead (no consumption) |
| Not-predicate | `!a` | Negative look-ahead |
| Search | `@a` | Skip ahead to first match |
| Capture | `{a}` | Save matched substring |
| Back-reference | `$1` | Reference to 1st capture |
| Non-terminal | `Name` | Named rule reference |
| Rule definition | `Name <- a b c` | Defines a non-terminal |
| Start anchor | `^` | Beginning of input |
| End anchor | `$` | End of input (`!.`) |
| Newline | `\n` | CR, LF, or CR+LF |
| Unicode letter | `\letter` | Any Unicode letter |
| Unicode lower | `\lower` | Lowercase Unicode letter |
| Unicode upper | `\upper` | Uppercase Unicode letter |
| Unicode title | `\title` | Title-case Unicode character |
| Unicode whitespace | `\white` | Unicode whitespace character |

**Escape sequences:**

| Sequence | Meaning |
|----------|---------|
| `\t` | Tab |
| `\n` | Newline |
| `\r` / `\c` | Carriage return |
| `\\` | Backslash |
| `\xHH` | Character with hex code HH |

**Constant:** `MaxSubpatterns* = 20` — maximum number of capturable sub-patterns.

---

## Types

```nim
type
  PegKind* = enum
    pkEmpty, pkAny, pkAnyRune, pkNewLine,
    pkLetter, pkLower, pkUpper, pkTitle, pkWhitespace,
    pkTerminal, pkTerminalIgnoreCase, pkTerminalIgnoreStyle,
    pkChar, pkCharChoice, pkNonTerminal,
    pkSequence, pkOrderedChoice,
    pkGreedyRep, pkGreedyRepChar, pkGreedyRepSet, pkGreedyAny,
    pkOption, pkAndPredicate, pkNotPredicate,
    pkCapture, pkBackRef, pkBackRefIgnoreCase, pkBackRefIgnoreStyle,
    pkSearch, pkCapturedSearch,
    pkRule, pkList, pkStartAnchor

  NonTerminalFlag* = enum
    ntDeclared, ntUsed

  Peg* = object          ## A node in a PEG expression tree
  NonTerminal* = ref object  ## A named non-terminal symbol

  Captures* = object     ## Capture results from a match
```

---

## Creating PEG Objects from a String

### `peg`

```nim
func peg*(pattern: string): Peg
```

Creates a `Peg` from a pattern string. Designed to be used as a **raw string modifier**:

```nim
import std/pegs

let p = peg"{\ident} \s* '=' \s* {.*}"
if "name = Alice" =~ p:
  echo matches[0]  # "name"
  echo matches[1]  # "Alice"
```

---

### `parsePeg`

```nim
func parsePeg*(pattern: string, filename = "pattern",
               line = 1, col = 0): Peg
```

Extended version of `peg`: accepts a filename and line/column for precise error reporting. Tracks line and column numbers within `pattern`.

```nim
import std/pegs

let p = parsePeg("{\\ident}", "config.peg", line = 10, col = 3)
```

---

### `escapePeg`

```nim
func escapePeg*(s: string): string
```

Escapes string `s` so that it is matched **literally** when used as a PEG pattern (all special characters are escaped).

```nim
import std/pegs

let raw = "hello (world)"
let escaped = escapePeg(raw)  # -> "'hello (world)'"
let p = peg(escaped)
echo "hello (world)" =~ p     # true
```

---

### `$` (convert to string)

```nim
func `$`*(r: Peg): string
```

Converts a `Peg` back to its PEG string representation. Useful for debugging.

```nim
import std/pegs

let p = sequence(term("ab"), *charSet({'0'..'9'}))
echo $p  # prints the PEG string representation
```

---

## PEG Constructors (DSL)

### Terminals

#### `term(t: string): Peg`

```nim
func term*(t: string): Peg
```

Creates a PEG that matches the exact string `t`. If `t` is one character long, a more efficient `pkChar` node is created.

```nim
let p = term("hello")
echo "hello world".find(p)  # 0
```

---

#### `term(t: char): Peg`

```nim
func term*(t: char): Peg
```

Creates a PEG matching the exact character `t`. The null character `'\0'` is not allowed.

```nim
let p = term('!')
```

---

#### `termIgnoreCase(t: string): Peg`

```nim
func termIgnoreCase*(t: string): Peg
```

Creates a PEG matching `t` case-insensitively.

```nim
let p = termIgnoreCase("hello")
echo "HELLO".match(p)  # true
echo "Hello".match(p)  # true
```

---

#### `termIgnoreStyle(t: string): Peg`

```nim
func termIgnoreStyle*(t: string): Peg
```

Creates a PEG matching `t` with style and case ignored: `_` and `-` are treated as equivalent, and case is ignored.

```nim
let p = termIgnoreStyle("helloWorld")
echo "hello_world".match(p)  # true
echo "Hello-World".match(p)  # true
```

---

#### `charSet(s: set[char]): Peg`

```nim
func charSet*(s: set[char]): Peg
```

Creates a PEG matching any single character in the set `s`. The null character `'\0'` is not allowed.

```nim
let p = charSet({'a'..'z', 'A'..'Z'})
echo "hello".match(p)  # true (first char matched)
```

---

### Operators (Combinators)

#### `/` — ordered choice

```nim
func `/`*(a: varargs[Peg]): Peg
```

Creates an **ordered choice**: tries alternatives left to right, returns the first match.

```nim
let p = term("cat") / term("dog") / term("bird")
echo "dog is here".find(p)  # 0, matched "dog"
```

---

#### `sequence` — sequence

```nim
func sequence*(a: varargs[Peg]): Peg
```

Creates a **sequence**: all PEGs must match consecutively.

```nim
let p = sequence(term("hello"), term(" "), term("world"))
echo "hello world".match(p)  # true
```

---

#### `?` — optional

```nim
func `?`*(a: Peg): Peg
```

Matches `a` **0 or 1 times**. If `a` is already repetitive (`*`, `?`) returns `a` unchanged.

```nim
let p = sequence(term("color"), ?term("u"), term("r"))
echo "color".find(peg"'color' 'u'? 'r'")  # -1
echo "colour".match(peg"'color' 'u'? 'r'")  # true
```

---

#### `*` — greedy repetition (0 or more)

```nim
func `*`*(a: Peg): Peg
```

Matches `a` **0 or more times** (greedy).

```nim
let p = *charSet({'0'..'9'})
echo "12345abc".matchLen(p)  # 5
```

---

#### `+` — greedy positive repetition (1 or more)

```nim
func `+`*(a: Peg): Peg
```

Matches `a` **1 or more times**. Equivalent to `sequence(a, *a)`.

```nim
let digits = +charSet({'0'..'9'})
echo "42 rest".matchLen(digits)  # 2
```

---

#### `!*` — search

```nim
func `!*`*(a: Peg): Peg
```

**Search**: skips characters until `a` matches. Corresponds to the `@a` syntax in PEG strings.

```nim
let p = !*term("end")
echo "some text end".matchLen(p)  # length up to "end"
```

---

#### `` !*\ `` — captured search

```nim
func `!*\`*(a: Peg): Peg
```

**Captured search**: like `!*` but captures the skipped text. Corresponds to `{@}a` in PEG strings.

---

### Predicates

#### `&` — and-predicate (lookahead)

```nim
func `&`*(a: Peg): Peg
```

Checks that the following text matches `a` **without consuming** input.

```nim
# Match "a" only if followed by "b"
let p = sequence(&term("ab"), term("a"))
echo "ab".match(p)  # true, only "a" consumed
echo "ac".match(p)  # false
```

---

#### `!` — not-predicate (negative lookahead)

```nim
func `!`*(a: Peg): Peg
```

Checks that the following text does **not** match `a`, without consuming input.

```nim
# Match any character that is not a digit
let p = sequence(!charSet({'0'..'9'}), any())
echo "a5".matchLen(p)  # 1 (only 'a' consumed)
```

---

### Captures and Back-references

#### `capture`

```nim
func capture*(a: Peg = Peg(kind: pkEmpty)): Peg
```

Creates a **capturing group**: saves the matched fragment into the captures array. Without an argument, creates an empty capture.

```nim
import std/pegs

var caps: array[0..19, string]
let p = sequence(capture(+charSet({'a'..'z'})), term("="),
                 capture(+charSet({'a'..'z'})))
discard "key=val".match(p, caps)
echo caps[0]  # "key"
echo caps[1]  # "val"
```

---

#### `backref`

```nim
func backref*(index: range[1..MaxSubpatterns],
              reverse: bool = false): Peg
```

Creates a **back-reference** to the `index`-th capture (1-based). `reverse = true` counts from the end of the capture list.

```nim
# Find a doubled word
let p = sequence(capture(+charSet({'a'..'z'})), term(" "), backref(1))
echo "hello hello".match(p)  # true
echo "hello world".match(p)  # false
```

---

#### `backrefIgnoreCase`

```nim
func backrefIgnoreCase*(index: range[1..MaxSubpatterns],
                        reverse: bool = false): Peg
```

Back-reference with case-insensitive matching.

```nim
let p = sequence(capture(+charSet({'a'..'z'})), term(" "), backrefIgnoreCase(1))
echo "hello HELLO".match(p)  # true
```

---

#### `backrefIgnoreStyle`

```nim
func backrefIgnoreStyle*(index: range[1..MaxSubpatterns],
                         reverse: bool = false): Peg
```

Back-reference with style-insensitive matching.

---

### Anchors and Special Symbols

#### `any` — any character

```nim
func any*: Peg
```

Matches **any single byte** (analogous to `.` in regexp).

```nim
let p = *any()   # consumes the entire string
```

---

#### `anyRune` — any Unicode character

```nim
func anyRune*: Peg
```

Matches **any Unicode codepoint** (rune), regardless of the number of UTF-8 bytes.

```nim
let p = *anyRune()
```

---

#### `newLine` — newline

```nim
func newLine*: Peg
```

Matches `\r\n`, `\n`, or `\r`.

```nim
let lines = split("a\nb\r\nc", newLine())
```

---

#### `startAnchor` — start of input

```nim
func startAnchor*: Peg
```

Matches only at the very beginning of the input (analogous to `^`).

```nim
let p = sequence(startAnchor(), term("start"))
echo "start here".match(p)  # true
echo "not start".match(p)   # false
```

---

#### `endAnchor` — end of input

```nim
func endAnchor*: Peg
```

Matches only at the end of the input (equivalent to `!any()`).

```nim
let p = sequence(*any(), endAnchor())
echo "anything".match(p)  # true
```

---

#### `unicodeLetter`, `unicodeLower`, `unicodeUpper`, `unicodeTitle`, `unicodeWhitespace`

```nim
func unicodeLetter*: Peg    # \letter — any Unicode letter
func unicodeLower*: Peg     # \lower  — lowercase Unicode letter
func unicodeUpper*: Peg     # \upper  — uppercase Unicode letter
func unicodeTitle*: Peg     # \title  — title-case Unicode character
func unicodeWhitespace*: Peg # \white — Unicode whitespace character
```

```nim
let p = +unicodeLetter()
echo "Привет123".matchLen(p)  # byte length of "Привет"
```

---

### Shorthand Templates

These templates expand to `charSet` calls:

| Template | Expands to | Description |
|----------|------------|-------------|
| `letters` | `charSet({'A'..'Z', 'a'..'z'})` | ASCII letters |
| `digits` | `charSet({'0'..'9'})` | Decimal digits |
| `whitespace` | `charSet({' ', '\9'..'\13'})` | ASCII whitespace |
| `identChars` | `charSet({'a'..'z','A'..'Z','0'..'9','_'})` | Identifier characters |
| `identStartChars` | `charSet({'a'..'z','A'..'Z','_'})` | Identifier start characters |
| `ident` | `sequence(identStartChars, *identChars)` | Standard identifier |
| `natural` | `+digits` | Natural number (`\d+`) |

```nim
import std/pegs

let p = sequence(ident, *sequence(whitespace, ident))
echo "foo bar baz".match(p)  # true
```

---

### Non-terminals

#### `newNonTerminal`

```nim
func newNonTerminal*(name: string, line, column: int): NonTerminal
```

Creates a non-terminal symbol for use in named-rule grammars.

---

#### `nonterminal`

```nim
func nonterminal*(n: NonTerminal): Peg
```

Creates a `Peg` node referencing non-terminal `n`. If the symbol is declared and small enough, its rule is inlined for optimization.

```nim
import std/pegs

# Grammars are best built via parsePeg/peg:
let grammar = peg"""
  Expr    <- Term ('+' Term)*
  Term    <- Factor ('*' Factor)*
  Factor  <- [0-9]+ / '(' Expr ')'
"""
```

---

## Peg Accessors

Functions for reading internal data of a `Peg` node. Only applicable to the matching `kind` variant.

| Function | Returns | Applicable to |
|----------|---------|---------------|
| `kind(p)` | `PegKind` | Any `Peg` |
| `term(p)` | `string` | `pkTerminal`, `pkTerminalIgnoreCase`, `pkTerminalIgnoreStyle` |
| `ch(p)` | `char` | `pkChar`, `pkGreedyRepChar` |
| `charChoice(p)` | `ref set[char]` | `pkCharChoice`, `pkGreedyRepSet` |
| `nt(p)` | `NonTerminal` | `pkNonTerminal` |
| `index(p)` | `int` | `pkBackRef`..`pkBackRefIgnoreStyle` |

```nim
let p = term("hello")
echo p.kind   # pkTerminal
echo p.term   # "hello"
```

---

#### `items(p: Peg): Peg` (iterator)

```nim
iterator items*(p: Peg): Peg
```

Iterates over the child nodes of a `Peg` tree node.

```nim
let p = sequence(term("a"), term("b"), term("c"))
for child in p:
  echo $child
```

---

#### `pairs(p: Peg): (int, Peg)` (iterator)

```nim
iterator pairs*(p: Peg): (int, Peg)
```

Iterates over child nodes together with their indices.

```nim
for i, child in peg"('a' / 'b' / 'c')":
  echo i, ": ", $child
```

---

## NonTerminal Accessors

| Function | Returns | Description |
|----------|---------|-------------|
| `name(nt)` | `string` | Name of the non-terminal |
| `line(nt)` | `int` | Declaration line number |
| `col(nt)` | `int` | Declaration column number |
| `flags(nt)` | `set[NonTerminalFlag]` | Flags: `ntDeclared`, `ntUsed` |
| `rule(nt)` | `Peg` | The PEG rule for this symbol |

---

## The `Captures` Type

```nim
type Captures* = object
```

Stores capture results after a `rawMatch` call. Holds up to `MaxSubpatterns` (20) positions.

#### `bounds(c: Captures, i: range[0..MaxSubpatterns-1]): tuple[first, last: int]`

```nim
func bounds*(c: Captures, i: range[0..MaxSubpatterns-1]): tuple[first, last: int]
```

Returns the byte positions `[first..last]` of the `i`-th capture (0-indexed).

```nim
import std/pegs

var c: Captures
let s = "hello world"
let p = peg"{\ident} \s+ {\ident}"
discard rawMatch(s, p, 0, c)
echo c.bounds(0)  # (0, 4)  — "hello"
echo c.bounds(1)  # (6, 10) — "world"
```

---

## Search and Matching

### `=~` — match operator

```nim
template `=~`*(s: string, pattern: Peg): bool
```

Convenience operator: calls `match` with an **implicitly declared** `matches` array accessible in the `if` body. Each `elif` branch gets its own independent `matches`.

```nim
import std/pegs

let line = "key = value"
if line =~ peg"{\ident} \s* '=' \s* {.*}":
  echo "Key: ", matches[0]    # "key"
  echo "Value: ", matches[1]  # "value"
elif line =~ peg"{'#'.*}":
  echo "Comment: ", matches[0]
else:
  echo "Syntax error"
```

---

### `match` — match from a position

```nim
func match*(s: string, pattern: Peg,
            matches: var openArray[string], start = 0): bool
func match*(s: string, pattern: Peg, start = 0): bool
```

Returns `true` if `s[start..]` matches `pattern`. The overload with `matches` fills it with captured substrings.

```nim
import std/pegs

var caps = newSeq[string](20)
if "2024-01-15".match(peg"{\d+} '-' {\d+} '-' {\d+}", caps):
  echo caps[0]  # "2024"
  echo caps[1]  # "01"
  echo caps[2]  # "15"
```

---

### `matchLen` — match length

```nim
func matchLen*(s: string, pattern: Peg,
               matches: var openArray[string], start = 0): int
func matchLen*(s: string, pattern: Peg, start = 0): int
```

Like `match`, but returns the **length of the match** (or -1). A length of 0 is possible; a suffix of `s` may remain unmatched.

```nim
import std/pegs

let n = "123abc".matchLen(peg"\d+")
echo n  # 3
```

---

### `find` — search in string

```nim
func find*(s: string, pattern: Peg,
           matches: var openArray[string], start = 0): int
func find*(s: string, pattern: Peg, start = 0): int
```

Returns the **starting position** of the first match in `s` (from `start`), or -1. The overload with `matches` fills captured substrings.

```nim
import std/pegs

echo "foo 42 bar".find(peg"\d+")  # 4
```

---

### `findBounds` — match bounds

```nim
func findBounds*(s: string, pattern: Peg,
                 matches: var openArray[string],
                 start = 0): tuple[first, last: int]
```

Returns `(first, last)` byte positions of the first match, or `(-1, 0)`. Fills `matches` with captures.

```nim
import std/pegs

var caps = newSeq[string](20)
let bounds = "text: hello world".findBounds(peg"{\w+}", caps)
echo bounds  # (first: 6, last: 10) — "hello"
echo caps[0]  # "hello"
```

---

### `findAll` — all matches

```nim
iterator findAll*(s: string, pattern: Peg, start = 0): string
func findAll*(s: string, pattern: Peg, start = 0): seq[string]
```

Iterates over / returns all matching **substrings**. Returns `@[]` if nothing matches.

```nim
import std/pegs

for word in "one two three".findAll(peg"\ident"):
  echo word  # "one", "two", "three"

let nums = "a1 b22 c333".findAll(peg"\d+")
echo nums  # @["1", "22", "333"]
```

---

### `contains` — membership check

```nim
func contains*(s: string, pattern: Peg, start = 0): bool
func contains*(s: string, pattern: Peg,
               matches: var openArray[string], start = 0): bool
```

Equivalent to `find(s, pattern, start) >= 0`. Returns `true` if the pattern is found anywhere in `s`.

```nim
import std/pegs

echo "hello123".contains(peg"\d+")  # true
echo "abcdef".contains(peg"\d+")    # false
```

---

### `startsWith` — prefix check

```nim
func startsWith*(s: string, prefix: Peg, start = 0): bool
```

Returns `true` if `s` starts with the pattern `prefix` (from position `start`).

```nim
import std/pegs

echo "hello world".startsWith(peg"\ident")  # true
echo "123 world".startsWith(peg"\ident")    # false
```

---

### `endsWith` — suffix check

```nim
func endsWith*(s: string, suffix: Peg, start = 0): bool
```

Returns `true` if `s` ends with the pattern `suffix`.

```nim
import std/pegs

echo "file.nim".endsWith(peg"'.' \ident")  # true
echo "file.txt".endsWith(peg"'.nim'")      # false
```

---

## Replacement

### `replace` — replacement without captures

```nim
func replace*(s: string, sub: Peg, by = ""): string
```

Replaces all occurrences of `sub` in `s` with the string `by`. Captures in `by` are not supported.

```nim
import std/pegs

echo "hello   world".replace(peg"\s+", " ")  # "hello world"
echo "aaa123bbb".replace(peg"\d+", "NUM")     # "aaaNU​Mbbb"
```

---

### `replacef` — replacement with capture formatting

```nim
func replacef*(s: string, sub: Peg, by: string): string
```

Replaces occurrences of `sub`, treating `by` as a format string. Captures are accessible via `$1`, `$2`, ..., and `$#` (like `strutils.%`).

```nim
import std/pegs

let result = "var1=key; var2=key2".replacef(
  peg"{\ident} '=' {\ident}",
  "$1<-$2$2"
)
echo result  # "var1<-keykey; var2<-key2key2"
```

---

### `replace` with callback

```nim
func replace*(s: string, sub: Peg,
              cb: proc(match, cnt: int,
                       caps: openArray[string]): string): string
```

Replaces occurrences of `sub` by calling `cb` for each match:
- `match` — index of the current match (starting at 0)
- `cnt` — number of captures in this match
- `caps` — array of captured substrings

```nim
import std/pegs

func handler(m, n: int, c: openArray[string]): string =
  if m > 0: result.add(", ")
  result.add(case n:
    of 2: c[0].toLower & ": '" & c[1] & "'"
    of 1: c[0].toLower & ": ''"
    else: "")

let s = "Var1=key1;var2=Key2;   VAR3"
echo s.replace(peg"{\ident}('='{\ident})* ';'* \s*", handler)
# var1: 'key1', var2: 'Key2', var3: ''
```

---

### `parallelReplace` — parallel replacement

```nim
func parallelReplace*(s: string,
                      subs: varargs[tuple[pattern: Peg, repl: string]]): string
```

Applies multiple substitutions **in parallel** (simultaneously, not sequentially). The first matching rule at each position wins; other rules are not checked for the same position.

```nim
import std/pegs

let result = "hello world 123".parallelReplace(
  (peg"\ident", "WORD"),
  (peg"\d+",    "NUM")
)
echo result  # "WORD WORD NUM"
```

---

### `transformFile` (non-JS only)

```nim
proc transformFile*(infile, outfile: string,
                    subs: varargs[tuple[pattern: Peg, repl: string]])
```

Reads `infile`, applies `parallelReplace`, and writes the result to `outfile`. Raises `IOError` on failure. **Not available in the JS backend.**

```nim
import std/pegs

transformFile("input.txt", "output.txt",
  (peg"'TODO'", "DONE"),
  (peg"\d{4}", "YEAR")
)
```

---

## String Splitting

### `split` — split by PEG separator

```nim
iterator split*(s: string, sep: Peg): string
func split*(s: string, sep: Peg): seq[string]
```

Splits string `s` at every occurrence of the PEG separator `sep`. The function variant returns a `seq[string]`.

```nim
import std/pegs

for word in split("one,,two,,,three", peg",+"):
  echo word  # "one", "two", "three"

let parts = split("00232this02939is39an22example111", peg"\d+")
echo parts  # @["this", "is", "an", "example"]
```

---

## Low-level Matching

### `rawMatch`

```nim
func rawMatch*(s: string, p: Peg, start: int,
               c: var Captures): int
```

The **low-level** matching function — the PEG interpreter engine. Every other PEG operation ultimately calls this. Returns the match length, or -1 on failure.

- `s` — the string to analyse
- `p` — the PEG pattern
- `start` — starting position for the match
- `c` — object that receives capture positions

```nim
import std/pegs

var c: Captures
let s = "hello 42"
let p = peg"\ident \s+ \d+"
let len = rawMatch(s, p, 0, c)
echo len  # 8 (entire string)

# Access captures via bounds:
let p2 = peg"{\ident} \s+ {\d+}"
var c2: Captures
discard rawMatch(s, p2, 0, c2)
echo s[c2.bounds(0).first .. c2.bounds(0).last]  # "hello"
echo s[c2.bounds(1).first .. c2.bounds(1).last]  # "42"
```

---

## Event Parser

### `eventParser`

```nim
template eventParser*(pegAst, handlers: untyped): (proc(s: string): int)
```

Generates an **event-driven parser proc** from a PEG AST and handler code blocks. Returns a `proc(s: string): int` — call it with the string to parse. Returns -1 on failure or the total match length.

Handler blocks for each `PegKind`:
- `enter:` — executed when entering an element (has `s`, `p`, `start` in scope)
- `leave:` — executed when leaving (has `s`, `p`, `start`, `length`; `length = -1` on failure)

```nim
import std/[strutils, pegs]

let pegAst = """
  Expr    <- Sum
  Sum     <- Product (('+' / '-') Product)*
  Product <- Value (('*' / '/') Value)*
  Value   <- [0-9]+ / '(' Expr ')'
""".peg

var valStack: seq[float] = @[]
var opStack = ""

let parseExpr = pegAst.eventParser:
  pkNonTerminal:
    leave:
      if length > 0:
        let matchStr = s.substr(start, start + length - 1)
        case p.nt.name
        of "Value":
          try: valStack.add matchStr.parseFloat
          except ValueError: discard
        of "Sum", "Product":
          try: discard matchStr.parseFloat
          except ValueError:
            if valStack.len > 1 and opStack.len > 0:
              let b = valStack.pop
              let a = valStack.pop
              valStack.add case opStack[^1]:
                of '+': a + b
                of '-': a - b
                of '*': a * b
                else: a / b
              opStack.setLen opStack.high
  pkChar:
    leave:
      if length == 1 and valStack.len > 0:
        let c = s[start]
        if c in {'+', '-', '*', '/'}:
          opStack.add c

let len = parseExpr("(5+3)/2-7*22")
echo valStack[^1]  # computed result
```

---

## Utilities

### `$` — string representation

```nim
func `$`*(r: Peg): string
```

Converts a `Peg` to a human-readable PEG string. Useful for debugging.

```nim
import std/pegs

let p = sequence(+letters, *sequence(term(" "), +letters))
echo $p  # something like "([A-Za-z]+ (' ' [A-Za-z]+)*)"
```

---

## Quick Reference

### Building a pattern

| Goal | Code |
|------|------|
| From string (convenient) | `peg"\\ident '=' {.*}"` |
| From string with position | `parsePeg(s, "file.peg", 1, 0)` |
| Escape a literal string | `escapePeg("a+b")` |
| Any byte | `any()` |
| Any Unicode char | `anyRune()` |
| Character set | `charSet({'a'..'z'})` |
| Literal string | `term("hello")` |
| Case-insensitive | `termIgnoreCase("hello")` |
| Sequence | `sequence(a, b, c)` |
| Ordered choice | `a / b / c` |
| 0 or 1 | `?a` |
| 0 or more | `*a` |
| 1 or more | `+a` |
| Not-predicate | `!a` |
| And-predicate | `&a` |
| Capture | `capture(a)` |
| Back-reference | `backref(1)` |
| Search | `!*a` |
| Start anchor | `startAnchor()` |
| End anchor | `endAnchor()` |

### Matching

| Goal | Code |
|------|------|
| Quick check | `s =~ peg"..."` |
| Boolean match | `s.match(p)` |
| With captures | `s.match(p, caps)` |
| Match length | `s.matchLen(p)` |
| Find position | `s.find(p)` |
| Find bounds | `s.findBounds(p, caps)` |
| All matches | `s.findAll(p)` |
| Contains? | `s.contains(p)` |
| Starts with? | `s.startsWith(p)` |
| Ends with? | `s.endsWith(p)` |

### Transformation

| Goal | Code |
|------|------|
| Replace | `s.replace(p, "new")` |
| Replace with captures | `s.replacef(p, "$1-$2")` |
| Replace with callback | `s.replace(p, myProc)` |
| Parallel replace | `s.parallelReplace([(p1,"a"),(p2,"b")])` |
| Transform a file | `transformFile("in","out", subs)` |
| Split | `s.split(p)` |
| Low-level engine | `rawMatch(s, p, 0, captures)` |
