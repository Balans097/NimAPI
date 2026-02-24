# `lexbase` Module Reference

> **Module:** `std/lexbase`  
> **Purpose:** A base object and buffer-management infrastructure for building efficient lexers (tokenisers) that read from a `Stream`.

---

## Overview

Writing a lexer from scratch requires careful handling of two concerns that have nothing to do with the actual grammar being parsed: buffering input efficiently, and tracking line and column numbers for error messages. `lexbase` handles both, so that concrete lexers (like those in `std/json`, `std/xmlparser`, etc.) can focus exclusively on token recognition.

### The core idea: sentinel-based buffering

Reading input one character at a time from a stream would be prohibitively slow because every read is a system call. `lexbase` instead reads a large chunk of input into an in-memory buffer (`buf`) at once. The lexer then scans through this buffer at full memory speed.

The clever part is the **sentinel** mechanism. Instead of checking `bufpos < bufLen` on every character (an extra branch inside the innermost loop), `lexbase` places a special terminator character at a chosen position in the buffer — the *sentinel*. The sentinel is always one of the `refillChars` (by default, a newline character). When the scanner reaches the sentinel, it knows it must either refill the buffer from the stream or handle a newline — two events that both require special treatment anyway. This means **the hot-path character loop needs zero extra boundary checks**.

```
buf: "Example Text\n ha!"   bufLen = 17
      ^pos = 0     ^sentinel = 12 ('\n')
```

When `bufpos` reaches the sentinel position, a refill procedure is called: it copies the unconsumed tail of the buffer to the front, reads a new chunk from the stream, and places the next sentinel.

### Relationship to concrete lexers

`BaseLexer` is defined as `object of RootObj`, meaning you create your own lexer type by inheriting from it:

```nim
import std/lexbase, std/streams

type
  MyLexer = object of BaseLexer
    token: string   # your extra fields
```

All the public fields of `BaseLexer` (`buf`, `bufpos`, `lineNumber`, etc.) are directly accessible in your subtype.

---

## Exported Constants

---

### `EndOfFile`

```nim
const EndOfFile* = '\0'
```

The character value used as the end-of-file marker inside the buffer. When the stream is exhausted, `lexbase` writes `'\0'` at the current buffer position so that lexer loops can detect EOF using the same character-comparison mechanism they use for everything else — without a separate boolean flag or length check.

```nim
# Typical usage in a hand-written lexer loop:
while L.buf[L.bufpos] != EndOfFile:
  processChar(L.buf[L.bufpos])
  inc(L.bufpos)
```

---

### `NewLines`

```nim
const NewLines* = {'\c', '\L'}
```

A character set containing the two newline characters:

| Character | Literal | Meaning |
|---|---|---|
| `'\c'` | CR, `\r`, 0x0D | Carriage return (used in old Mac OS and as part of Windows `\r\n`) |
| `'\L'` | LF, `\n`, 0x0A | Line feed (Unix newline; second byte of Windows `\r\n`) |

This set is the default value for the `refillChars` parameter of `open`. It is also useful in lexer character-classification expressions and loop termination conditions.

```nim
# Check whether a character is any kind of line ending
if L.buf[L.bufpos] in NewLines:
  # handle end of line
```

---

## Exported Types

---

### `BaseLexer`

```nim
type BaseLexer* = object of RootObj
  bufpos*:     int    # current read position in buf
  buf*:        string # the internal buffer
  lineNumber*: int    # current 1-based line number
  offsetBase*: int    # cumulative character offset of buf[0] from stream start
  # (private: input, sentinel, lineStart, refillChars)
```

The base type for all lexers built on this module. Inherit from it; do not instantiate it directly.

#### Public fields

| Field | Type | Description |
|---|---|---|
| `bufpos` | `int` | The index into `buf` of the character about to be consumed. Advance it as you read characters. |
| `buf` | `string` | The raw character buffer. Read characters from it; do not resize it manually. |
| `lineNumber` | `int` | The current line number, starting at 1. Updated by `handleCR` and `handleLF`. |
| `offsetBase` | `int` | The stream offset of `buf[0]`. Add `bufpos` to get the absolute character offset from the start of input: `offsetBase + bufpos`. |

#### Private fields (not directly accessible)

| Field | Purpose |
|---|---|
| `input` | The underlying `Stream` being read. |
| `sentinel` | Index of the sentinel character in `buf`. |
| `lineStart` | Index in `buf` where the current line began; used to compute the column number. |
| `refillChars` | The set of characters that trigger a buffer refill check; defaults to `NewLines`. |

---

## Exported Procedures

---

### `open`

```nim
proc open*(L: var BaseLexer, input: Stream, bufLen: int = 8192;
           refillChars: set[char] = NewLines)
```

#### What it does

Initialises a `BaseLexer` (or any type derived from it) and prepares it to read from the given `Stream`. This is the first call you must make before using any other `BaseLexer` procedure. It:

1. Stores the stream reference.
2. Allocates the internal buffer to `bufLen` characters.
3. Performs the first buffer fill from the stream.
4. Detects and silently skips a UTF-8 BOM (`\xEF\xBB\xBF`) at the very start of the input, so downstream parsers do not see it.
5. Sets `lineNumber` to 1 (lines are 1-based).

#### Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `L` | `var BaseLexer` | — | The lexer object to initialise. Must be declared by the caller. |
| `input` | `Stream` | — | Any `Stream` (file stream, string stream, memory stream, …). Must not be `nil`. |
| `bufLen` | `int` | `8192` | Size of the internal buffer in characters. Larger values reduce the number of I/O refill calls; the minimum useful value is 1. |
| `refillChars` | `set[char]` | `NewLines` | Characters that act as sentinel candidates. The buffer refill is triggered at one of these positions. Normally you keep the default; custom values are useful for formats where newlines have no special meaning but other characters (e.g., `\0`, `>`) do. |

#### Examples

```nim
import std/lexbase, std/streams

var L: BaseLexer
let s = newStringStream("hello\nworld")
L.open(s)
# L is now ready; L.buf holds the content, L.lineNumber == 1
```

```nim
import std/lexbase, std/streams

# Reading from a file
var L: BaseLexer
let fs = openFileStream("data.csv")
L.open(fs, bufLen = 32768)  # larger buffer for big files
defer: L.close()
```

```nim
import std/lexbase, std/streams

# Custom refill characters: useful for a format where '>' is the only
# structurally significant delimiter and newlines are irrelevant
var L: BaseLexer
L.open(newStringStream("<tag>value>"), refillChars = {'>'})
```

---

### `close`

```nim
proc close*(L: var BaseLexer)
```

#### What it does

Closes the `BaseLexer` and its underlying `Stream`. After calling `close`, the `buf` contents are no longer meaningful and no further reading should be performed. Always call this when you are done lexing, or use `defer: L.close()` immediately after `open` to guarantee cleanup.

#### Example

```nim
import std/lexbase, std/streams

var L: BaseLexer
L.open(newStringStream("data"))
defer: L.close()
# ... lex tokens ...
```

---

### `handleCR`

```nim
proc handleCR*(L: var BaseLexer, pos: int): int
```

#### What it does

Must be called whenever the scanner encounters a carriage-return character (`'\c'`, `\r`) in the buffer. It performs three tasks:

1. **Increments `lineNumber`** — the CR marks the end of a line.
2. **Triggers a buffer refill if necessary** — because CR is one of the `refillChars` and thus acts as a sentinel.
3. **Consumes a following LF if present** — Windows line endings are `\r\n`. If the next character after the CR is `\n`, `handleCR` skips it automatically so that the lexer sees a single logical line ending regardless of the platform convention.
4. **Updates `lineStart`** — records where the new line begins, enabling correct column number computation.

Returns the buffer position from which scanning should **resume** after the line ending.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `pos` | `int` | The position of the `'\c'` character in `buf`. `L.buf[pos]` must equal `'\c'`. |

#### Return value

The new `bufpos` value, pointing to the first character of the next line.

#### Example

```nim
# Inside a lexer scanning loop:
case L.buf[L.bufpos]
of '\c':
  L.bufpos = handleCR(L, L.bufpos)
  # L.bufpos now points to the first char of the next line
  # L.lineNumber has been incremented
of '\L':
  L.bufpos = handleLF(L, L.bufpos)
else:
  inc(L.bufpos)
```

---

### `handleLF`

```nim
proc handleLF*(L: var BaseLexer, pos: int): int
```

#### What it does

Must be called whenever the scanner encounters a line-feed character (`'\L'`, `\n`) in the buffer. It is the Unix counterpart to `handleCR`. It:

1. **Increments `lineNumber`**.
2. **Triggers a buffer refill if necessary**.
3. **Updates `lineStart`** to point to the start of the new line.

Unlike `handleCR`, it does **not** look ahead for a second newline character — a standalone `\n` is a complete Unix line ending on its own.

Returns the buffer position from which scanning should resume.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `pos` | `int` | The position of the `'\L'` character in `buf`. `L.buf[pos]` must equal `'\L'`. |

#### Return value

The new `bufpos` value, pointing to the first character of the next line.

#### Example

```nim
of '\L':
  L.bufpos = handleLF(L, L.bufpos)
  # L.lineNumber++, L.lineStart updated, bufpos on next line's first char
```

---

### `handleRefillChar`

```nim
proc handleRefillChar*(L: var BaseLexer, pos: int): int
```

#### What it does

A generalised version of `handleCR` / `handleLF` for **non-newline refill characters**. If you configured `open` with a custom `refillChars` set that includes characters other than `'\c'` and `'\L'`, call this procedure when the scanner hits one of those characters. It triggers the buffer refill but does **not** increment `lineNumber` or update `lineStart` — those are purely line-tracking concerns.

This is useful for formats (like binary protocols or certain DSLs) where a specific delimiter character should act as a sentinel without implying a new line.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `pos` | `int` | The position of the refill character. `L.buf[pos]` must be a member of `L.refillChars`. |

#### Return value

The new `bufpos` from which scanning should resume.

#### Example

```nim
import std/lexbase, std/streams

# Suppose we opened with refillChars = {'|', '\c', '\L'}
# When we hit '|', we refill but don't count a new line:
of '|':
  L.bufpos = handleRefillChar(L, L.bufpos)
  # buffer has been refilled; bufpos is past the '|'
```

---

### `getColNumber`

```nim
proc getColNumber*(L: BaseLexer, pos: int): int
```

#### What it does

Returns the **0-based column number** of the character at buffer position `pos` within the current line. Column 0 is the leftmost character. The result is computed as `abs(pos - L.lineStart)`.

This is the key function for generating accurate error messages: after detecting a problem at `L.bufpos`, call `getColNumber(L, L.bufpos)` to find out which column the bad token starts at.

Note that the column is measured in buffer positions (characters), not in visual display columns. For ASCII-only input this is identical to the visual column. For multi-byte UTF-8 sequences, the visual column may differ.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `pos` | `int` | Any buffer position, typically `L.bufpos`. |

#### Return value

0-based column index.

#### Example

```nim
import std/lexbase, std/streams

var L: BaseLexer
L.open(newStringStream("hello world\ngoodbye"))
# Simulate scanning to position 6 ('w' of "world")
L.bufpos = 6

echo getColNumber(L, L.bufpos)  # → 6 (0-based)
```

```nim
# In an error-reporting procedure:
proc lexError(L: BaseLexer, msg: string) =
  let line = L.lineNumber
  let col  = getColNumber(L, L.bufpos) + 1  # convert to 1-based for display
  echo msg & " at line " & $line & ", column " & $col
```

---

### `getCurrentLine`

```nim
proc getCurrentLine*(L: BaseLexer, marker: bool = true): string
```

#### What it does

Returns the **full text of the line currently being scanned**, as a string ending with `"\n"`. Optionally appends a second line with a caret (`^`) positioned under `L.bufpos` — visually pointing to the exact character being processed when the function is called.

This is designed specifically for error messages. The output looks like:

```
let x = @bad_token
         ^
```

The function reads forward from `L.lineStart` until it hits `'\c'`, `'\L'`, or `EndOfFile`, collecting all characters in between. It does not modify any state.

#### Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `marker` | `bool` | `true` | When `true`, appends a `^` indicator line under the current `bufpos` position. Pass `false` to get only the line text. |

#### Return value

A `string` containing the current source line (plus `"\n"`), and — if `marker` is `true` — a second line of spaces followed by `"^\n"`.

#### Examples

```nim
import std/lexbase, std/streams

var L: BaseLexer
L.open(newStringStream("let x = 42\necho x"))
L.bufpos = 8  # point to '4' in "42"

echo getCurrentLine(L)
# Output:
# let x = 42
#         ^
```

```nim
# Minimal error reporter for a hand-written lexer:
proc reportError(L: BaseLexer, msg: string) =
  echo "Error (line ", L.lineNumber, ", col ",
       getColNumber(L, L.bufpos) + 1, "): ", msg
  echo getCurrentLine(L)   # shows the line with a ^ marker

# If the lexer hits an unexpected '#' at column 5 of line 3:
# Error (line 3, col 5): unexpected character '#'
# foo = #bar
#     ^
```

```nim
# Without the marker — useful when you want just the line text:
let lineText = getCurrentLine(L, marker = false)
echo "Offending line: ", lineText
```

---

## Complete Minimal Lexer Example

The following shows how all the pieces fit together in a real, if tiny, lexer that tokenises a stream into words and counts lines:

```nim
import std/lexbase, std/streams

type
  WordLexer = object of BaseLexer

proc initWordLexer(src: string): WordLexer =
  result.open(newStringStream(src))

proc nextToken(L: var WordLexer): string =
  result = ""
  # Skip whitespace (but handle newlines properly)
  while true:
    case L.buf[L.bufpos]
    of ' ', '\t':
      inc(L.bufpos)
    of '\c':
      L.bufpos = handleCR(L, L.bufpos)
    of '\L':
      L.bufpos = handleLF(L, L.bufpos)
    of EndOfFile:
      return ""
    else:
      break

  # Collect a word
  while L.buf[L.bufpos] notin {' ', '\t', '\c', '\L', EndOfFile}:
    result.add(L.buf[L.bufpos])
    inc(L.bufpos)

var lex = initWordLexer("hello world\nfoo bar")
defer: lex.close()

var tok = lex.nextToken()
while tok.len > 0:
  echo "line ", lex.lineNumber, ": ", tok
  tok = lex.nextToken()

# Output:
# line 1: hello
# line 1: world
# line 2: foo
# line 2: bar
```

---

## Design Notes and Pitfalls

**1. Always assign the return value of handle* procedures**  
`handleCR`, `handleLF`, and `handleRefillChar` all return the new `bufpos`. Forgetting to write `L.bufpos = handleCR(L, L.bufpos)` means the buffer refill happens but `bufpos` is not updated — causing the lexer to re-read the same position forever.

**2. Sentinel position is not stable across refills**  
After a buffer refill, `buf` contents shift: the tail of the old buffer becomes the start of the new one. Any stored `int` index into `buf` that you computed before the refill is now invalid. The only index that survives a refill correctly is `bufpos` itself (which `fillBaseLexer` resets to 0).

**3. `offsetBase + bufpos` gives the absolute stream offset**  
If you need to record the stream position of a token start (e.g., for re-reading or error reporting), save `L.offsetBase + L.bufpos` at the beginning of the token, not just `L.bufpos`.

**4. `lineNumber` is only updated by `handleCR` / `handleLF`**  
If your lexer skips newlines without calling one of these handlers, `lineNumber` will be wrong and `getCurrentLine` will report incorrect positions.

**5. `getColNumber` is 0-based; error messages are usually 1-based**  
Add 1 to the result of `getColNumber` before displaying it to users: `getColNumber(L, L.bufpos) + 1`.

**6. UTF-8 BOM is skipped automatically**  
`open` silently advances past a leading `\xEF\xBB\xBF` sequence. If you are implementing a lexer for a format where the BOM is meaningful, be aware that `BaseLexer` has already consumed it by the time your lexer code runs.
