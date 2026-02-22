# `parsejson` Module Reference

> **Module:** `parsejson`  
> **Purpose:** A low-level, streaming, event-driven JSON parser. Used internally by the `json` module, but usable on its own for memory-efficient processing of large JSON documents without building a full parse tree.

---

## Overview

`parsejson` implements a **pull parser**: you repeatedly call `next` to advance the parser one event at a time and inspect the result through `kind`, `str`, `getInt`, and `getFloat`. This is the same pattern as SAX in XML or `encoding/json`'s `Decoder.Token` in Go — you consume only as much of the document as you need, and you never pay for a full in-memory tree.

The parser sits on top of `lexbase.BaseLexer`, which provides buffered line-by-line input, correct CR/LF/CRLF handling, and line/column tracking. As a bonus the parser tolerates `//` and `/* */` comments in its input — technically non-standard JSON, but useful when parsing config files.

### Typical usage pattern

```
open → loop { next → inspect kind → read value } → close
```

---

## Types

### `JsonEventKind`

```nim
type JsonEventKind* = enum
  jsonError,        ## a parse error occurred
  jsonEof,          ## the input is exhausted
  jsonString,       ## a string value was parsed
  jsonInt,          ## an integer literal was parsed
  jsonFloat,        ## a floating-point literal was parsed
  jsonTrue,         ## the literal `true`
  jsonFalse,        ## the literal `false`
  jsonNull,         ## the literal `null`
  jsonObjectStart,  ## the `{` token
  jsonObjectEnd,    ## the `}` token
  jsonArrayStart,   ## the `[` token
  jsonArrayEnd      ## the `]` token
```

Every call to `next` puts the parser into one of these states. Reading `kind` tells you which state you are in. The events map directly to JSON's structural tokens: you receive `jsonObjectStart` when `{` is encountered, then a sequence of key-value events, then `jsonObjectEnd` when `}` is reached.

---

### `TokKind`

```nim
type TokKind* = enum
  tkError, tkEof, tkString, tkInt, tkFloat,
  tkTrue, tkFalse, tkNull,
  tkCurlyLe, tkCurlyRi,     ## { }
  tkBracketLe, tkBracketRi, ## [ ]
  tkColon, tkComma
```

The raw lexical token as returned by `getTok`. `JsonEventKind` is the higher-level semantic event derived from the token after the state machine in `next` processes it. Most users only need `JsonEventKind`; `TokKind` is relevant when you call `getTok` or `eat` directly.

---

### `JsonError`

```nim
type JsonError* = enum
  errNone,              ## no error
  errInvalidToken,      ## invalid character sequence
  errStringExpected,    ## a string was expected
  errColonExpected,     ## ':' was expected
  errCommaExpected,     ## ',' was expected
  errBracketRiExpected, ## ']' was expected
  errCurlyRiExpected,   ## '}' was expected
  errQuoteExpected,     ## '"' or "'" was expected
  errEOC_Expected,      ## '*/' was expected (unterminated block comment)
  errEofExpected,       ## EOF was expected but more tokens found
  errExprExpected       ## an expression was expected
```

When `kind` is `jsonError`, the internal `err` field holds one of these values. Call `errorMsg` to get the full human-readable description including filename, line, and column.

---

### `JsonParser`

```nim
type JsonParser* = object of BaseLexer
  a*: string          ## accumulation buffer: holds the raw text of the current token
  tok*: TokKind       ## the most recently lexed raw token
  kind: JsonEventKind ## current semantic event (read via kind())
  err: JsonError      ## current error (read via errorMsg())
  state: seq[ParserState]
  filename: string
  rawStringLiterals: bool
```

The parser object. **Must be declared as `var`** because all mutating procs take it by `var`. The fields `a` and `tok` are public — `a` contains the raw text of the current string, integer, or float token; `tok` is the last raw token kind.

Inheriting from `BaseLexer` gives the parser automatic buffered I/O, correct newline normalisation, and the `lineNumber`/`bufpos` machinery used by `getLine` and `getColumn`.

---

### `JsonKindError`

```nim
type JsonKindError* = object of ValueError
```

Raised by the `to` macro in the `json` module when the JSON kind does not match the expected Nim type. Not raised directly by `parsejson` itself, but declared here as part of the shared public API.

---

### `JsonParsingError`

```nim
type JsonParsingError* = object of ValueError
```

Raised by `raiseParseErr`. Carries the formatted error message produced by `errorMsgExpected`, including filename, line, and column.

---

### `errorMessages`

```nim
const errorMessages*: array[JsonError, string]
```

A compile-time array mapping each `JsonError` variant to its plain English description. Indexed by `JsonError`. Useful if you need to print an error without going through `errorMsg` (e.g. when you already have the `JsonError` value but not the parser object).

```nim
echo errorMessages[errColonExpected]   # → "':' expected"
```

---

## Lifecycle

### `open`

```nim
proc open*(my: var JsonParser, input: Stream, filename: string;
           rawStringLiterals = false)
```

Initialises the parser with a `Stream` and a filename string. The filename is used only in error messages — it can be any descriptive label, not necessarily a real file path.

**`rawStringLiterals`** controls how string tokens are stored in `my.a`:
- `false` (default): escape sequences are decoded into their actual characters, surrounding quotes are stripped. `"hello\nworld"` → `hello` + newline + `world`.
- `true`: the string is stored verbatim including its surrounding `"` characters and all `\` escape sequences unchanged. `"hello\nworld"` → `"hello\nworld"`. This is useful for round-tripping JSON without re-encoding strings.

```nim
import std/streams

var p: JsonParser
let s = newStringStream("""{"name": "Alice", "age": 30}""")
p.open(s, "inline")
defer: p.close()
```

---

### `close`

```nim
proc close*(my: var JsonParser) {.inline.}
```

Closes the parser and releases the underlying `BaseLexer` resources (and thereby the associated stream). Always call `close` when done, or use `defer`.

```nim
p.open(s, "data.json")
defer: p.close()
```

---

## Driving the Parser

### `next`

```nim
proc next*(my: var JsonParser)
```

Advances the parser to the **next semantic event** and updates `my.kind`. This is the central control proc — the parser does nothing until you call `next`. After each call, read `kind()` to learn what was found, then call `str`, `getInt`, or `getFloat` as appropriate.

`next` is implemented as a state machine with a stack of `ParserState` values. Each structural token (`{`, `}`, `[`, `]`) pushes or pops a state. The stack lets the parser correctly handle arbitrarily nested JSON.

```nim
p.next()
while p.kind != jsonEof:
  case p.kind
  of jsonObjectStart: echo "{"
  of jsonObjectEnd:   echo "}"
  of jsonArrayStart:  echo "["
  of jsonArrayEnd:    echo "]"
  of jsonString:      echo "string: ", p.str
  of jsonInt:         echo "int: ",    p.getInt
  of jsonFloat:       echo "float: ",  p.getFloat
  of jsonTrue:        echo "true"
  of jsonFalse:       echo "false"
  of jsonNull:        echo "null"
  of jsonError:
    echo p.errorMsg
    break
  of jsonEof: break
  p.next()
```

---

### `getTok`

```nim
proc getTok*(my: var JsonParser): TokKind
```

Lexes the **next raw token** from the input, skipping whitespace and comments first, and returns its `TokKind`. Also updates `my.tok` and `my.a`. Unlike `next`, this does **not** update `my.kind` or run the state machine — it operates one level below `next` at the pure lexical level.

Use `getTok` when you are implementing custom token-level parsing logic, such as in `eat`, or when integrating `parsejson` with another parser that needs direct token access.

```nim
let tok = p.getTok()
if tok == tkString:
  echo "raw string token: ", p.a
```

---

## Reading Values

### `kind`

```nim
proc kind*(my: JsonParser): JsonEventKind {.inline.}
```

Returns the current event type set by the most recent call to `next`. Always call `next` first; reading `kind` before any `next` call returns the initial value `jsonError`.

```nim
p.next()
if p.kind == jsonString:
  echo p.str
```

---

### `str`

```nim
proc str*(my: JsonParser): string {.inline.}
```

Returns the text content of the current token. Valid only when `kind` is one of `jsonString`, `jsonInt`, or `jsonFloat`. Asserts in debug mode if called in any other state.

- For `jsonString`: the decoded string value (or the raw quoted string if `rawStringLiterals` was `true`).
- For `jsonInt` / `jsonFloat`: the number as it appeared in the source text, e.g. `"42"` or `"3.14e-2"`.

```nim
p.next()
assert p.kind == jsonString
echo p.str   # the actual string value, unquoted
```

---

### `getInt`

```nim
proc getInt*(my: JsonParser): BiggestInt {.inline.}
```

Parses and returns the current token's text as a `BiggestInt`. Valid only when `kind == jsonInt`. Asserts otherwise. Internally calls `parseBiggestInt(my.a)`.

`BiggestInt` is `int64` on 64-bit targets. For values that fit in a regular `int`, just cast: `int(p.getInt())`.

```nim
p.next()
assert p.kind == jsonInt
let n = p.getInt()   # e.g. 42
```

---

### `getFloat`

```nim
proc getFloat*(my: JsonParser): float {.inline.}
```

Parses and returns the current token's text as a `float`. Valid only when `kind == jsonFloat`. Asserts otherwise. Internally calls `parseFloat(my.a)`.

```nim
p.next()
assert p.kind == jsonFloat
let f = p.getFloat()   # e.g. 3.14
```

---

## Position Information

### `getColumn`

```nim
proc getColumn*(my: JsonParser): int {.inline.}
```

Returns the **1-based column number** of the current parser position — the column of the character currently being examined in the input. Useful for error messages and diagnostics.

```nim
if p.kind == jsonError:
  echo "at column ", p.getColumn
```

---

### `getLine`

```nim
proc getLine*(my: JsonParser): int {.inline.}
```

Returns the **1-based line number** of the current parser position. Newlines in the input (CR, LF, CRLF) are all correctly tracked by the underlying `BaseLexer`.

```nim
echo "line: ", p.getLine, ", col: ", p.getColumn
```

---

### `getFilename`

```nim
proc getFilename*(my: JsonParser): string {.inline.}
```

Returns the filename string that was passed to `open`. Purely informational — it can be any label you chose.

```nim
echo p.getFilename   # whatever you passed as filename to open()
```

---

## Error Handling

### `errorMsg`

```nim
proc errorMsg*(my: JsonParser): string
```

Returns a formatted error string for the current `jsonError` event. Format: `"filename(line, col) Error: message"`. Only valid when `kind == jsonError`; asserts otherwise.

```nim
p.next()
if p.kind == jsonError:
  echo p.errorMsg
  # e.g. → "data.json(3, 12) Error: ':' expected"
```

---

### `errorMsgExpected`

```nim
proc errorMsgExpected*(my: JsonParser, e: string): string
```

Builds an error message in the same `"filename(line, col) Error: <e> expected"` format, but lets you supply the `e` text yourself instead of deriving it from the parser's internal error. Used by `raiseParseErr` to format custom expectation messages.

```nim
if p.kind != jsonString:
  echo p.errorMsgExpected("key string")
  # → "data.json(1, 2) Error: key string expected"
```

---

### `raiseParseErr`

```nim
proc raiseParseErr*(p: JsonParser, msg: string) {.noinline, noreturn.}
```

Raises a `JsonParsingError` with a message built by `errorMsgExpected(p, msg)`. Marked `noreturn`, so the compiler knows execution does not continue past this call. Use it to signal structural expectation failures in custom parsing logic.

```nim
if p.kind != jsonObjectStart:
  p.raiseParseErr("object")
  # raises JsonParsingError: "data.json(1, 1) Error: object expected"
```

---

### `eat`

```nim
proc eat*(p: var JsonParser, tok: TokKind)
```

Asserts that the **current raw token** (`p.tok`) matches `tok`, then advances the lexer by calling `getTok`. If the token does not match, calls `raiseParseErr` with the expected token's name.

This is the standard "consume and move on" helper for building recursive-descent parsers on top of `parsejson`. It works at the `TokKind` level, not the `JsonEventKind` level.

```nim
# Consume the opening brace of a JSON object:
discard p.getTok()       # lex the first token
p.eat(tkCurlyLe)         # assert it's '{' and advance
```

---

### `parseEscapedUTF16`

```nim
proc parseEscapedUTF16*(buf: cstring, pos: var int): int
```

Parses exactly 4 hexadecimal characters from `buf` starting at `pos`, decoding them as a UTF-16 code unit. Returns the decoded integer value, or `-1` on failure. Advances `pos` past the 4 hex digits on success.

This is a public utility used internally by the string parser to handle `\uXXXX` escape sequences, including surrogate pair decoding. It is exposed so that code layered on top of `parsejson` can reuse the same escape parsing logic.

```nim
var pos = 0
let codeUnit = parseEscapedUTF16(cstring("1F600"), pos)
# pos is now 4, codeUnit is 0x1F60 (first 4 hex digits)
```

Note: surrogate pairs (`\uD800\uDC00` and similar) are assembled into the correct Unicode code point inside the private `parseString` proc; `parseEscapedUTF16` itself only reads one 4-hex-digit group.

---

## Complete Usage Example

### Streaming a large JSON array

```nim
import std/[parsejson, streams]

proc countInts(json: string): int =
  var p: JsonParser
  p.open(newStringStream(json), "input")
  defer: p.close()

  p.next()
  if p.kind != jsonArrayStart:
    p.raiseParseErr("array")

  p.next()
  while p.kind != jsonArrayEnd:
    if p.kind == jsonError:
      echo p.errorMsg
      break
    if p.kind == jsonInt:
      inc result
    p.next()

echo countInts("[1, 2, \"skip\", 3, null, 4]")   # → 4
```

### Extracting a single field without a full parse tree

```nim
proc findName(json: string): string =
  var p: JsonParser
  p.open(newStringStream(json), "input")
  defer: p.close()

  var depth = 0
  var expectValue = false
  p.next()
  while p.kind notin {jsonEof, jsonError}:
    case p.kind
    of jsonObjectStart: inc depth
    of jsonObjectEnd:   dec depth
    of jsonString:
      if expectValue and depth == 1:
        return p.str
      expectValue = (depth == 1 and p.str == "name")
    else:
      expectValue = false
    p.next()

let data = """{"id": 1, "name": "Alice", "role": "admin"}"""
echo findName(data)   # → Alice
```

### Using `rawStringLiterals` for round-tripping

```nim
var p: JsonParser
p.open(newStringStream("""{"key": "hello\nworld"}"""), "rt", rawStringLiterals = true)
defer: p.close()

p.next()   # jsonObjectStart
p.next()   # jsonString  →  key name
p.next()   # jsonString  →  value, still has escape sequences
echo p.str   # → "hello\nworld"  (with the quotes and backslash)
```

---

## Event Sequence Reference

Understanding what `next` produces for common JSON structures:

**Scalar value** `42`:
```
jsonInt   → getInt() == 42
jsonEof
```

**Object** `{"a": 1}`:
```
jsonObjectStart
jsonString  → str() == "a"      (the key)
jsonInt     → getInt() == 1     (the value)
jsonObjectEnd
jsonEof
```

**Array** `[true, null]`:
```
jsonArrayStart
jsonTrue
jsonNull
jsonArrayEnd
jsonEof
```

**Nested** `{"x": [1, 2]}`:
```
jsonObjectStart
jsonString   → "x"
jsonArrayStart
jsonInt      → 1
jsonInt      → 2
jsonArrayEnd
jsonObjectEnd
jsonEof
```

---

## Error Reference

| Error constant | Meaning | Typical cause |
|---------------|---------|---------------|
| `errNone` | No error | — |
| `errInvalidToken` | Unexpected character or bad escape | Malformed `\uXXXX`, unknown token |
| `errStringExpected` | A string was required | Object key is not a quoted string |
| `errColonExpected` | `:` missing between key and value | `{"a" 1}` instead of `{"a": 1}` |
| `errCommaExpected` | `,` missing between elements | `[1 2]` instead of `[1, 2]` |
| `errBracketRiExpected` | `]` missing | Array not closed |
| `errCurlyRiExpected` | `}` missing | Object not closed |
| `errQuoteExpected` | Unterminated string | String literal hit EOF |
| `errEOC_Expected` | Block comment not closed | `/* ...` without `*/` |
| `errEofExpected` | Extra tokens after the top-level value | `42 43` (two top-level values) |
| `errExprExpected` | An expression was expected | Object value is missing |

| Exception type | Raised by | Inherits from |
|---------------|-----------|---------------|
| `JsonParsingError` | `raiseParseErr` | `ValueError` |
| `JsonKindError` | `json.to` macro | `ValueError` |
