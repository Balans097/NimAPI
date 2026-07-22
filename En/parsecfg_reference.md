# parsecfg — module reference

> **Import:** `import std/parsecfg`
> **Purpose:** parsing and writing configuration files in a format close to Windows `.ini`, but with support for Nim string literals (ordinary, raw, and triple-quoted strings).

The module solves two related but distinct tasks. First, it offers a
low-level streaming (event-based) parser: `CfgParser` reads the input
stream token by token and, through `next`, hands back one event at a time
(`CfgEvent`) — a section start, a key-value pair, a `--key:value`
command-line-style option, or an error. This layer keeps nothing buffered
in memory as a whole and is well suited to large files or to custom
processing logic.

Second, built on top of the event parser is a table-based API: the `Config`
type (a table of tables) together with `loadConfig`/`writeConfig`/
`getSectionValue`/`setSectionKey`, which load the whole file into memory and
provide convenient "section → key → value" access. This is the layer to
reach for whenever full control over the parsing process isn't needed.

A module-wide convention: for keys of the form `--key:value` (written in
the file with a double dash), the table API stores the key name together
with its `"--"` prefix, to distinguish them from ordinary `key=value`
pairs. The empty string `""` used as a section name denotes the "common"
section — whatever is written in the file before the first `[section]`
header.

---

## Table of Contents

I. [Types and helpers](#types-and-helpers)
   1. [`CfgEventKind`](#cfgeventkind)
   2. [`CfgEvent`](#cfgevent)
   3. [`CfgParser`](#cfgparser)
   4. [`Config`](#config)
II. [Streaming parser — the low-level event API](#streaming-parser--the-low-level-event-api)
   1. [`open`](#open)
   2. [`next`](#next)
   3. [`close`](#close)
   4. [`getColumn`, `getLine`, `getFilename`](#getcolumn-getline-getfilename)
   5. [`errorStr`, `warningStr`, `ignoreMsg`](#errorstr-warningstr-ignoremsg)
III. [Table API — loading and building configuration](#table-api--loading-and-building-configuration)
   1. [`newConfig`](#newconfig)
   2. [`loadConfig` (from a stream)](#loadconfig-from-a-stream)
   3. [`loadConfig` (from a file)](#loadconfig-from-a-file)
   4. [`getSectionValue`](#getsectionvalue)
   5. [`setSectionKey`](#setsectionkey)
   6. [`delSection`](#delsection)
   7. [`delSectionKey`](#delsectionkey)
   8. [`sections` (iterator)](#sections-iterator)
IV. [Writing and serializing configuration](#writing-and-serializing-configuration)
   1. [`writeConfig` (to a stream)](#writeconfig-to-a-stream)
   2. [`writeConfig` (to a file)](#writeconfig-to-a-file)
   3. [`` `$` ``](#-)
V. [Practical recipes](#practical-recipes)
   1. [Building a configuration from scratch and saving it to a file](#building-a-configuration-from-scratch-and-saving-it-to-a-file)
   2. [Reading with default values](#reading-with-default-values)
   3. [Updating an existing file (round-trip)](#updating-an-existing-file-round-trip)
   4. [Iterating over all sections and keys](#iterating-over-all-sections-and-keys)
   5. [Custom event handling that ignores unknown entries](#custom-event-handling-that-ignores-unknown-entries)
VI. [Quick reference table](#quick-reference-table)
VII. [Summary: which procedure to use](#summary-which-procedure-to-use)

---

## Types and helpers

### `CfgEventKind`

```nim
type
  CfgEventKind* = enum
    cfgEof,          ## end of file reached
    cfgSectionStart, ## a `[section]` header has been recognized
    cfgKeyValuePair, ## a `key=value` pair has been recognized
    cfgOption,       ## a `--key=value` command-line option has been recognized
    cfgError         ## an error occurred during parsing
```

**What it does.** An enum of the event kinds `next` can return. It's the
"tag" of the `CfgEvent` variant object — the value of `kind` determines
which fields of the event are populated.

- **Values:**
  - `cfgEof` — the stream is exhausted, parsing is done;
  - `cfgSectionStart` — a `[name]` section header was encountered;
  - `cfgKeyValuePair` — an ordinary `key=value` or `key:value` pair was encountered;
  - `cfgOption` — a `--key:value` entry was encountered (outside a section it belongs to the "common" part of the file);
  - `cfgError` — a syntax error; parsing does not stop with an exception, the event simply reports it.

---

### `CfgEvent`

```nim
type
  CfgEvent* = object of RootObj
    case kind*: CfgEventKind
    of cfgEof: nil
    of cfgSectionStart:
      section*: string
    of cfgKeyValuePair, cfgOption:
      key*, value*: string
    of cfgError:
      msg*: string
```

**What it does.** The variant object-event returned by `next`. The set of
available fields depends on `kind`: `cfgSectionStart` only has `section`,
`cfgKeyValuePair`/`cfgOption` have a `key`/`value` pair, and `cfgError` has
`msg`. Accessing a field that doesn't belong to the current `kind` (for
example, `section` on an event with `kind == cfgError`) is a runtime error,
as with any Nim variant object.

- **Parameters/fields:**
  - `kind: CfgEventKind` — determines the active variant;
  - `section: string` — the section name (only for `cfgSectionStart`);
  - `key, value: string` — the key and value (for `cfgKeyValuePair`/`cfgOption`); `value == ""` if no value was given in the file;
  - `msg: string` — the error text, already formatted with the file name, line, and column (only for `cfgError`).

**Example:**

```nim
import std/streams

var p: CfgParser
open(p, newStringStream("[Package]\nname=hello\n"), "example.ini")
while true:
  var e = next(p)
  case e.kind
  of cfgEof: break
  of cfgSectionStart:
    echo "section: " & e.section  # prints: section: Package
  of cfgKeyValuePair:
    echo e.key & " = " & e.value  # prints: name = hello
  of cfgOption:
    echo "option: " & e.key
  of cfgError:
    echo e.msg
close(p)
```

---

### `CfgParser`

```nim
type
  CfgParser* = object of BaseLexer
```

**What it does.** An opaque parser object: it inherits from `BaseLexer`
(`std/lexbase`) and holds the current input buffer, read position, line
number, and a pre-read (lookahead) token. No fields are exposed publicly —
access to the state goes only through `getLine`, `getColumn`, and
`getFilename`.

**Implementation notes.** The parser is built as a one-token-lookahead
parser: right after `open`, and after every `next` call, the parser
already has the next token "sitting" in it, ready to be examined. This
simplifies `next` — there's no need to backtrack when deciding what
construct has started (a key-value pair or a section header); looking one
token ahead is enough.

- **Parameters:** no public fields; the object is declared as a `var` variable and passed by reference into every procedure of the module (`open`, `next`, `close`, and so on).

---

### `Config`

```nim
type
  Config* = OrderedTableRef[string, OrderedTableRef[string, string]]
```

**What it does.** An in-memory table representation of an entire
configuration file: the outer ordered table maps a section name to an
inner ordered table of "key → value". Insertion order of both sections
and keys is preserved — this matters when the table is later written back
out with `writeConfig`, so the file doesn't get shuffled on every save.

- **Parameters:**
  - outer key — the section name (`""` — the common section, everything before the first `[...]`);
  - inner key — the parameter name within a section (for `--options` it's stored together with the `"--"` prefix);
  - value — a string (the module performs no type conversion — all values are stored as text).

---

## Streaming parser — the low-level event API

### `open`

```nim
proc open*(c: var CfgParser, input: Stream, filename: string, lineOffset = 0)
```

**What it does.** Initializes parser `c` with the given input stream.
`filename` is only used in error/warning message text and has no effect
on parsing itself. `lineOffset` shifts the line numbering used in those
same messages — useful when the text being parsed is an insert inside
another document and the line numbers need to match the original
document.

**Implementation notes.** Calls `lexbase.open` to set up buffered stream
reading, resets any previous token state (`tkInvalid`, empty literal),
adds `lineOffset` to the line counter, and immediately reads the first
token via the internal `rawGetTok`. That's why, right after `open`, the
parser is already ready for the first `next` call — the lookahead token
is already in place.

- **Parameters:**
  - `c: var CfgParser` — the parser being initialized;
  - `input: Stream` — an already-open input stream;
  - `filename: string` — the file name for error messages;
  - `lineOffset: int` — line-numbering offset, default `0`.

**Example:**

```nim
import std/streams

var f = newFileStream("example.ini", fmRead)
doAssert f != nil, "failed to open the file"
var p: CfgParser
open(p, f, "example.ini")
close(p)
```

---

### `next`

```nim
proc next*(c: var CfgParser): CfgEvent
```

**What it does.** Returns the next parsing event and advances the parser.
This is the single procedure that drives parsing — it's called in a loop
until `cfgEof` is returned. On encountering a `[` token, the parser
expects a symbol followed by a closing `]`; if either of those doesn't
match, `cfgError` is returned, but parsing doesn't stop with an
exception — `next` can keep being called afterward.

**Implementation notes.** The kind of event to build is decided from the
current lookahead token `c.tok.kind`:

- `tkEof` → `cfgEof`;
- `tkDashDash` (`--`) → the next token is read and the result is built via
  the helper `getKeyValPair` with `kind = cfgOption`;
- `tkSymbol` → also `getKeyValPair`, but with `kind = cfgKeyValuePair`;
- `tkBracketLe` (`[`) → a `tkSymbol` is expected, then `tkBracketRi` (`]`);
  a mismatch at either step produces `cfgError` stating what was expected;
- any other token (`tkInvalid`, `tkEquals`, `tkColon`, a stray
  `tkBracketRi`) → `cfgError` "invalid token".

The common technique — reading a `key` plus an optional separator
(`=` or `:`) plus a `value` — lives in `getKeyValPair`: if no `=`/`:`
follows the key, the value stays an empty string (`value == ""`), which
matches what's documented for the `CfgEvent.value` field.

- **Parameters:** `c: var CfgParser` — the parser to pull an event from.

**Example (boundary case — a key without a value):**

```nim
import std/streams

var p: CfgParser
open(p, newStringStream("key_without_value\n"), "[stream]")
let e = next(p)
doAssert e.kind == cfgKeyValuePair
doAssert e.key == "key_without_value"
doAssert e.value == ""  # no value was given — it stays empty
close(p)
```

**Example (error case):**

```nim
import std/streams

var p: CfgParser
open(p, newStringStream("[section\n"), "[stream]")  # no closing ]
let e = next(p)
doAssert e.kind == cfgError
echo e.msg  # prints a formatted message with file name, line, and column
close(p)
```

---

### `close`

```nim
proc close*(c: var CfgParser)
```

**What it does.** Closes the parser and the input stream associated with
it. Called once after `next` has returned `cfgEof` (or after parsing is
aborted early).

- **Parameters:** `c: var CfgParser`.

---

### `getColumn`, `getLine`, `getFilename`

```nim
proc getColumn*(c: CfgParser): int
proc getLine*(c: CfgParser): int
proc getFilename*(c: CfgParser): string
```

**What they do.** Three simple getters for the parser's current position:
column, line, and the file name passed to `open`. Mostly used for custom
error messages — the standard `errorStr`/`warningStr` already use them
internally.

- **Parameters:** `c: CfgParser` (no `var` — read-only access to the state) in all three cases.

**Example:**

```nim
import std/streams

var p: CfgParser
open(p, newStringStream("a=b\n"), "cfg.ini")
discard next(p)
echo getFilename(p) & ":" & $getLine(p) & ":" & $getColumn(p)
close(p)
```

---

### `errorStr`, `warningStr`, `ignoreMsg`

```nim
proc errorStr*(c: CfgParser, msg: string): string
proc warningStr*(c: CfgParser, msg: string): string
proc ignoreMsg*(c: CfgParser, e: CfgEvent): string
```

**What they do.** They format a text message of the form
`filename(line, column) Error: text` (or `Warning:` respectively), using
the parser's current position. `ignoreMsg` is a specialized variant for
when calling code decides to ignore an event (for example, an unknown
section or an unsupported option): the wording depends on `e.kind` —
"section ignored", "key ignored", "command ignored"; for `cfgError` the
error text itself is returned, and for `cfgEof` an empty string.

- **Parameters:**
  - `errorStr`/`warningStr`: `c: CfgParser`, `msg: string` — arbitrary message text;
  - `ignoreMsg`: `c: CfgParser`, `e: CfgEvent` — the event being ignored.

**Example:** see the [«Custom event handling that ignores unknown entries»](#custom-event-handling-that-ignores-unknown-entries) recipe below — that's the main scenario `ignoreMsg` is used for.

---

## Table API — loading and building configuration

### `newConfig`

```nim
proc newConfig*(): Config
```

**What it does.** Creates an empty configuration table — the starting
point for building a settings file programmatically (as opposed to
reading one from disk).

- **Parameters:** none.

**Example:**

```nim
var dict = newConfig()
doAssert len(dict) == 0
```

---

### `loadConfig` (from a stream)

```nim
proc loadConfig*(stream: Stream, filename: string = "[stream]"): Config
```

**What it does.** Reads the entire stream through an internal `CfgParser`
and builds a `Config` in memory, grouping key-value pairs by the most
recently seen section.

**Implementation notes.** Internally this is the same
`while true: next(p)` loop shown in the low-level API examples, except
the result isn't printed but accumulated: the `curSection` variable holds
the name of the last section seen (initially `""` — the common section);
on `cfgKeyValuePair` the pair is placed into the current section's inner
table; on `cfgOption` it goes into the same table, but the key gets a
`"--"` prefix so it isn't confused with an ordinary key when written back
out later. An important detail: on `cfgError` the loop simply stops
(`break`) — no exception is raised, and the file will be loaded only up
to the error point, with no warning that this happened.

- **Parameters:**
  - `stream: Stream` — an open stream containing configuration content;
  - `filename: string` — the name used in error messages, defaults to `"[stream]"`.

**Example:**

```nim
import std/streams

let dict = loadConfig(newStringStream("[Package]\nname=hello\n"))
doAssert getSectionValue(dict, "Package", "name") == "hello"
```

---

### `loadConfig` (from a file)

```nim
proc loadConfig*(filename: string): Config
```

**What it does.** The same thing, but takes a file path on disk rather
than a ready-made stream.

**Implementation notes.** Under normal execution it opens the file
(`open`, `fmRead`), wraps it in a `FileStream`, and delegates to the
stream-based overload, closing the stream when done (`defer`). When
compiled for `nimvm` (for example, in NimScript), a workaround is used
instead: the file is read in full via `readFile` into a string and
wrapped in a `StringStream`, since low-level stream `open` through
`{.importc.}` isn't available in NimScript.

- **Parameters:** `filename: string` — the path to the configuration file.

**Example:**

```nim
let dict = loadConfig("config.ini")
echo getSectionValue(dict, "Package", "name")
```

---

### `getSectionValue`

```nim
proc getSectionValue*(dict: Config, section, key: string, defaultVal = ""): string
```

**What it does.** Returns the value of key `key` in section `section`; if
the section or the key doesn't exist, it returns `defaultVal` instead of
raising an error or an exception. This is the main way to safely read
values out of an already-loaded configuration.

- **Parameters:**
  - `dict: Config` — the configuration table;
  - `section: string` — the section name (`""` — the common section);
  - `key: string` — the key name;
  - `defaultVal: string` — the fallback value if the key isn't found; defaults to `""`.

**Example:**

```nim
let dict = loadConfig(newStringStream("[Package]\nname=hello\n"))
doAssert getSectionValue(dict, "Package", "name") == "hello"
doAssert getSectionValue(dict, "Package", "version", "0.1.0") == "0.1.0"  # no such key — the fallback was returned
```

---

### `setSectionKey`

```nim
proc setSectionKey*(dict: var Config, section, key, value: string)
```

**What it does.** Sets the value of a key in the given section. If the
section doesn't exist yet, it's created automatically — calling code
doesn't need to check for its presence beforehand.

- **Parameters:**
  - `dict: var Config` — the mutable configuration table;
  - `section, key, value: string` — the section, the key, and the new value.

**Example:**

```nim
var dict = newConfig()
setSectionKey(dict, "Package", "name", "hello")
setSectionKey(dict, "Package", "--threads", "on")  # a command-line-style option key
doAssert getSectionValue(dict, "Package", "name") == "hello"
```

---

### `delSection`

```nim
proc delSection*(dict: var Config, section: string)
```

**What it does.** Deletes an entire section along with all of its keys.
If no section with that name exists, nothing happens — a silent no-op,
same as `del` on plain Nim tables.

- **Parameters:** `dict: var Config`, `section: string`.

**Example:**

```nim
var dict = loadConfig(newStringStream("[Author]\nname=nim-lang\n"))
delSection(dict, "Author")
doAssert getSectionValue(dict, "Author", "name", "no author") == "no author"
```

---

### `delSectionKey`

```nim
proc delSectionKey*(dict: var Config, section, key: string)
```

**What it does.** Deletes a single key inside a section. If the section
has no keys left after the deletion, the section itself is removed too —
so the table doesn't accumulate empty entries.

**Implementation notes.** Before deleting, `dict[section].len == 1` is
checked: if the key being removed was the only one, `dict.del(section)`
is called (removing the whole section); otherwise the key is deleted from
the inner table directly. If the section or the key doesn't exist, the
call silently does nothing.

- **Parameters:** `dict: var Config`, `section: string`, `key: string`.

**Example:**

```nim
var dict = loadConfig(newStringStream("[Author]\nname=nim-lang\nwebsite=nim-lang.org\n"))
delSectionKey(dict, "Author", "website")
doAssert getSectionValue(dict, "Author", "website", "no website") == "no website"
doAssert getSectionValue(dict, "Author", "name") == "nim-lang"  # the section stays, since another key remains
```

---

### `sections` (iterator)

```nim
iterator sections*(dict: Config): lent string
```

**What it does.** Iterates over all section names in the table, including
the empty string `""` (the common section) if it holds at least one key.
The iteration order is insertion order, since `Config` is an ordered
table.

- **Parameters:** `dict: Config` — the table being iterated over.

**Example:**

```nim
let dict = loadConfig(newStringStream("charset=utf-8\n[Package]\nname=hello\n"))
for section in sections(dict):
  echo "section: \"" & section & "\""
# prints:
# section: ""
# section: "Package"
```

---

## Writing and serializing configuration

### `writeConfig` (to a stream)

```nim
proc writeConfig*(dict: Config, stream: Stream)
```

**What it does.** Writes the table's contents to a stream in `.ini`
format, section by section, in insertion order. The empty section `""`
is written without a `[...]` header. Comments are not supported — they're
dropped when a file is read, so, naturally, there's nowhere for them to
come from when writing (this is explicitly noted in the source
documentation of the procedure).

**Implementation notes.** Writing is essentially the reverse of parsing.
For each section name and each value, the module decides whether it needs
quoting using a single criterion: does the string contain only "safe"
characters (`SymChars` — letters, digits, `_`, space, `./\-`, and bytes
`\x80..\xFF`)? If so, it's written as-is; if not, the string is wrapped
in ordinary quotes `"..."`, and if it already contains a `"`, in triple
quotes `"""..."""` instead, to avoid colliding with the closing quote.
Keys with a `"--"` prefix are recognized by their first two characters and
written with a colon (`--key:value`) instead of `=`, as required by the
command-line-option syntax. Escaping of backslashes and line breaks
within a value is handled by the internal `replace` helper.

- **Parameters:**
  - `dict: Config` — the table being written;
  - `stream: Stream` — a stream open for writing.

**Example:**

```nim
import std/streams

var dict = newConfig()
setSectionKey(dict, "", "charset", "utf-8")
setSectionKey(dict, "Package", "name", "hello")
setSectionKey(dict, "Package", "--threads", "on")
let stream = newStringStream()
writeConfig(dict, stream)
echo stream.data
# prints:
# charset=utf-8
# [Package]
# name=hello
# --threads:on
```

---

### `writeConfig` (to a file)

```nim
proc writeConfig*(dict: Config, filename: string)
```

**What it does.** The same thing, but opens the file for writing itself
(mode `fmWrite`, meaning any previous file contents are fully overwritten)
and closes it when done.

- **Parameters:** `dict: Config`, `filename: string` — the destination file path.

**Example:**

```nim
var dict = loadConfig("config.ini")
setSectionKey(dict, "Author", "name", "nim-lang")
writeConfig(dict, "config.ini")
```

---

### `` `$` ``

```nim
proc `$`*(dict: Config): string
```

**What it does.** Returns the table's contents as a string in the same
format `writeConfig` uses — handy for debug printing or comparison in
tests without writing to disk.

**Implementation notes.** Wraps `writeConfig` around a `StringStream`,
then retrieves the accumulated data via `stream.data`.

- **Parameters:** `dict: Config`.

**Example:**

```nim
var dict = newConfig()
setSectionKey(dict, "Package", "name", "hello")
doAssert $dict == "[Package]\nname=hello\n"
```

---

## Practical recipes

### Building a configuration from scratch and saving it to a file

```nim
var dict = newConfig()
setSectionKey(dict, "", "charset", "utf-8")           # the common section
setSectionKey(dict, "Package", "name", "hello")
setSectionKey(dict, "Package", "--threads", "on")      # a command-line option
setSectionKey(dict, "Author", "name", "nim-lang")
setSectionKey(dict, "Author", "website", "nim-lang.org")
writeConfig(dict, "generated.ini")
```

### Reading with default values

```nim
let dict = loadConfig("config.ini")
let
  charset = getSectionValue(dict, "", "charset", "utf-8")
  threads = getSectionValue(dict, "Package", "--threads", "off")
  version = getSectionValue(dict, "Package", "version", "0.0.0")
echo charset & " / " & threads & " / " & version
```

### Updating an existing file (round-trip)

```nim
var dict = loadConfig("config.ini")
setSectionKey(dict, "Author", "name", "nim-lang")
delSectionKey(dict, "Author", "website")   # dropped a stale key
writeConfig(dict, "config.ini")            # rewrote the file in place
```

### Iterating over all sections and keys

```nim
let dict = loadConfig("config.ini")
for section in sections(dict):
  echo "[" & section & "]"
  for key, value in pairs(dict[section]):
    echo "  " & key & " = " & value
```

### Custom event handling that ignores unknown entries

```nim
import std/streams

const knownSections = ["Package", "Author"]

var p: CfgParser
open(p, newFileStream("config.ini"), "config.ini")
while true:
  var e = next(p)
  case e.kind
  of cfgEof: break
  of cfgSectionStart:
    if not contains(knownSections, e.section):
      echo ignoreMsg(p, e)  # a warning with file/line/column info
  of cfgKeyValuePair:
    echo e.key & " = " & e.value
  of cfgOption:
    echo ignoreMsg(p, e)   # options aren't supported in this scenario
  of cfgError:
    echo errorStr(p, e.msg)
close(p)
```

---

## Quick reference table

| Task | Mutates `dict` | Returns |
|---|---|---|
| Parse a file event by event | no (only the `CfgParser`) | a `CfgEvent` via `next` |
| Load an entire file into a table | no | a new `Config` (`loadConfig`) |
| Create an empty configuration | no | a new `Config` (`newConfig`) |
| Read a value with a fallback | no | `string` (`getSectionValue`) |
| Set/create a key | yes | — (`setSectionKey`) |
| Delete an entire section | yes | — (`delSection`) |
| Delete a single key (and the section if it becomes empty) | yes | — (`delSectionKey`) |
| Iterate over section names in order | no | `string` per iteration (`sections`) |
| Save to a stream/file | no | — (`writeConfig`) |
| Get the contents as a string | no | `string` (`` `$` ``) |

---

## Summary: which procedure to use

- Need to read a whole file and work with it like a dictionary → `loadConfig` + `getSectionValue`.
- Need to build a configuration programmatically and write it to disk → `newConfig` + `setSectionKey` + `writeConfig`.
- Need to read values that might be missing without extra checks → `getSectionValue` with the `defaultVal` parameter.
- Need to delete a single parameter without worrying about an emptied-out section → `delSectionKey`.
- Need to delete a section entirely → `delSection`.
- Need to iterate over all sections in file order → the `sections` iterator.
- Need full control over parsing (custom error formatting, partial loading, streaming without buffering the whole file) → the low-level `CfgParser` with `open`/`next`/`close`.
- Need to tell the user that an unrecognized directive was ignored → `ignoreMsg`.
- Need to build an error/warning message with line and column info manually → `errorStr`/`warningStr`.
