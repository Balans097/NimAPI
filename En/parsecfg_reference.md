# `parsecfg` Module Reference

> **Nim Standard Library — `std/parsecfg`**
> High-performance parser for INI-style configuration files, plus a complete
> read/write API for structured config data.
> Supports both a streaming pull-parser interface and a high-level table-based interface.

---

## Table of Contents

1. [Overview & Mental Model](#overview--mental-model)
2. [Config File Syntax](#config-file-syntax)
3. [Types](#types)
   - [CfgEventKind](#cfgeventkind)
   - [CfgEvent](#cfgevent)
   - [CfgParser](#cfgparser)
   - [Config](#config)
4. [Low-Level Pull Parser API](#low-level-pull-parser-api)
   - [open](#open)
   - [close](#close)
   - [next](#next)
   - [getLine](#getline)
   - [getColumn](#getcolumn)
   - [getFilename](#getfilename)
   - [errorStr](#errorstr)
   - [warningStr](#warningstr)
   - [ignoreMsg](#ignoremsg)
5. [High-Level Config Table API](#high-level-config-table-api)
   - [newConfig](#newconfig)
   - [loadConfig (from stream)](#loadconfig-from-stream)
   - [loadConfig (from file)](#loadconfig-from-file)
   - [getSectionValue](#getsectionvalue)
   - [setSectionKey](#setsectionkey)
   - [delSectionKey](#deletesectionkey)
   - [delSection](#deletesection)
   - [sections](#sections)
   - [writeConfig (to stream)](#writeconfig-to-stream)
   - [writeConfig (to file)](#writeconfig-to-file)
   - [$ (to string)](#-to-string)
6. [Which API to Use?](#which-api-to-use)
7. [Complete Worked Examples](#complete-worked-examples)

---

## Overview & Mental Model

The `parsecfg` module provides two distinct ways to work with configuration files:

**1. Pull Parser (low-level)** — stream-oriented, event-by-event processing. You call `next()` in a loop and receive one `CfgEvent` at a time. Memory-efficient for huge files; lets you react to each token as it is parsed.

**2. Config Table (high-level)** — the entire file is loaded into an `OrderedTableRef` that maps sections to key-value pairs. Simple, familiar, and sufficient for almost all real applications.

```
Config file on disk
        │
        │  loadConfig(filename)        ← high-level: one call loads everything
        ▼
  Config (table)
  ├─ getSectionValue(section, key)
  ├─ setSectionKey(section, key, val)
  ├─ delSectionKey / delSection
  ├─ writeConfig(filename)
  └─ $dict  →  string

        │
        │  open(parser, stream, ...)   ← low-level: manual event loop
        ▼
  CfgParser
  └─ loop: next(parser) → CfgEvent
       ├─ cfgSectionStart  → e.section
       ├─ cfgKeyValuePair  → e.key, e.value
       ├─ cfgOption        → e.key, e.value
       ├─ cfgError         → e.msg
       └─ cfgEof           → break
```

The `Config` type is an alias for `OrderedTableRef[string, OrderedTableRef[string, string]]` — a table of sections, each holding an ordered table of key-value strings. Section insertion order and key insertion order are preserved.

---

## Config File Syntax

The parser understands a superset of the Windows `.ini` format. Key points:

**Sections** are declared with `[SectionName]` on their own line. Section names are trimmed of surrounding whitespace. The empty section `""` is the implicit global section before the first `[...]` header.

**Key-value pairs** use `=` or `:` as the delimiter:
```ini
name = hello
port: 8080
```

**Command-line options** start with `--` and use `:` as their delimiter. They appear as `cfgOption` events (stored with the `--` prefix in the `Config` table):
```ini
--threads:on
--opt:speed
```

**String literals** follow Nim conventions:
- Bare symbols: `value`, `nim-lang.org`, `some/path`
- Quoted strings: `"hello world"` — support escape sequences (`\n`, `\t`, `\x41`, etc.)
- Raw strings: `r"C:\Users\nim"` — backslashes are literal, no escape processing
- Triple-quoted strings: `"""multi-line value"""` — span multiple lines

**Comments** begin with `#` or `;` and extend to end of line:
```ini
# this is a comment
; so is this
key = value  # inline comments NOT supported — the # is part of the value
```

**Inline comments are not supported.** Everything after the `=` or `:` to the end of the token is the value, including `#` characters.

**Whitespace** around keys, values, and section names is stripped.

**Keys without values** are valid — `value` is set to `""`.

A complete example:
```ini
charset = "utf-8"

[Package]
name = "hello"
--threads:on

[Author]
name = "nim-lang"
website = "nim-lang.org"
```

---

## Types

### `CfgEventKind`

```nim
type CfgEventKind* = enum
  cfgEof,          ## end of file reached
  cfgSectionStart, ## a [section] header was parsed
  cfgKeyValuePair, ## a key=value or key:value pair was parsed
  cfgOption,       ## a --key:value command-line option was parsed
  cfgError         ## a parse error occurred (no exception is raised)
```

The discriminator for `CfgEvent`. Every call to `next()` returns an event of one of these five kinds. Errors do not raise exceptions — they arrive as `cfgError` events so the caller can decide how to handle them.

---

### `CfgEvent`

```nim
type CfgEvent* = object of RootObj
  case kind*: CfgEventKind
  of cfgEof:         (nothing)
  of cfgSectionStart:
    section*: string
  of cfgKeyValuePair, cfgOption:
    key*:   string
    value*: string   ## "" if no value was given
  of cfgError:
    msg*: string
```

A variant object describing a single parsed token. The fields available depend on `kind`:

| `kind` | Accessible fields | What they contain |
|---|---|---|
| `cfgEof` | — | nothing; signals the end of input |
| `cfgSectionStart` | `section` | name of the section (without `[` `]`) |
| `cfgKeyValuePair` | `key`, `value` | the key and its value; `value = ""` if none given |
| `cfgOption` | `key`, `value` | the option name without `--`; `value` may be `""` |
| `cfgError` | `msg` | formatted error string with file, line, column |

---

### `CfgParser`

```nim
type CfgParser* = object of BaseLexer
```

The parser object for the low-level pull-parser API. Declare it as `var`, initialise with `open()`, drive with `next()`, release with `close()`. Its internal state is managed by the module — do not access its fields directly.

---

### `Config`

```nim
type Config* = OrderedTableRef[string, OrderedTableRef[string, string]]
```

The data structure used by the high-level API. It is a reference type (heap-allocated), so you can pass it around without copying. Sections and keys preserve their insertion order.

- The **outer table** maps section names (`string`) to inner tables.
- The **inner table** maps key names to string values.
- The **global section** (keys before the first `[...]` header) is stored under the empty string key `""`.
- `--option` keys are stored with their `--` prefix: `dict["Section"]["--threads"]`.

---

## Low-Level Pull Parser API

Use this API when you need to process a config file as a stream, react to individual events, or handle files too large to load entirely.

### `open`

```nim
proc open*(c: var CfgParser, input: Stream, filename: string, lineOffset = 0)
```

**What it does:** Initialises the parser and binds it to an input `Stream`. After `open`, the parser is positioned just before the first token; calling `next()` fetches the first event.

**Parameters:**
- `input` — any `Stream` object: `newFileStream`, `newStringStream`, etc.
- `filename` — used only for error message formatting; it does not need to be a real path.
- `lineOffset` — shifts the reported line number by this amount. Useful when the config content is embedded inside a larger file and you want error messages to show the absolute line.

```nim
import std/[streams, parsecfg]

var s = newFileStream("app.cfg", fmRead)
var p: CfgParser
open(p, s, "app.cfg")
# p is now ready; call next(p) to start reading events
```

```nim
# Parse a config string embedded in code
var s = newStringStream("[db]\nhost=localhost\nport=5432\n")
var p: CfgParser
open(p, s, "<embedded>")
```

---

### `close`

```nim
proc close*(c: var CfgParser)
```

**What it does:** Closes the parser and its associated stream. Always call `close` when done — the idiomatic pattern is `defer: p.close()` immediately after `open`.

```nim
open(p, stream, "config.ini")
defer: p.close()
```

---

### `next`

```nim
proc next*(c: var CfgParser): CfgEvent
```

**What it does:** Advances the parser to the next event and returns it. This is the main driver of the pull-parser loop. Each call consumes exactly one logical token from the input.

The loop pattern is always the same: call `next`, dispatch on `e.kind`, break when `cfgEof`.

```nim
while true:
  var e = next(p)
  case e.kind
  of cfgEof:          break
  of cfgSectionStart: echo "section: ", e.section
  of cfgKeyValuePair: echo e.key, " = ", e.value
  of cfgOption:       echo "--", e.key, ": ", e.value
  of cfgError:        echo "error: ", e.msg
```

**Important:** The parser does **not** raise exceptions on malformed input. Syntax errors arrive as `cfgError` events. If you receive `cfgError`, it is safe to continue calling `next()` to attempt recovery — or you can break out of the loop immediately.

---

### `getLine`

```nim
proc getLine*(c: CfgParser): int
```

**What it does:** Returns the 1-based line number at the parser's current position. Useful when you want to include position information in your own error or warning messages.

---

### `getColumn`

```nim
proc getColumn*(c: CfgParser): int
```

**What it does:** Returns the 0-based column number at the parser's current position.

---

### `getFilename`

```nim
proc getFilename*(c: CfgParser): string
```

**What it does:** Returns the `filename` string that was passed to `open`. Used internally by `errorStr` and `warningStr` to produce consistent, well-formatted messages.

---

### `errorStr`

```nim
proc errorStr*(c: CfgParser, msg: string): string
```

**What it does:** Formats `msg` into the standard error message format:

```
filename(line, column) Error: msg
```

Use this when you detect a *semantic* error in your own code — for example, a required key is missing — and want the message to look consistent with parser-generated errors.

```nim
if not dict.hasKey("db"):
  echo p.errorStr("required section [db] is missing")
  # → "config.ini(42, 1) Error: required section [db] is missing"
```

---

### `warningStr`

```nim
proc warningStr*(c: CfgParser, msg: string): string
```

**What it does:** Like `errorStr`, but formats the message with `Warning:` instead of `Error:`:

```
filename(line, column) Warning: msg
```

Use for non-fatal issues — unknown keys that are being ignored, deprecated settings, etc.

```nim
echo p.warningStr("unknown key '" & key & "' will be ignored")
# → "config.ini(15, 1) Warning: unknown key 'legacy_mode' will be ignored"
```

---

### `ignoreMsg`

```nim
proc ignoreMsg*(c: CfgParser, e: CfgEvent): string
```

**What it does:** Returns a standard "this entry is being ignored" warning message for any `CfgEvent`. It delegates to `warningStr` internally, so the message includes file, line, and column. For `cfgError` events, it returns `e.msg` unchanged. For `cfgEof`, it returns `""`.

This is a convenience shortcut for the common pattern where you want to silently skip an unknown section or key but still log a warning about it.

```nim
while true:
  var e = next(p)
  case e.kind
  of cfgEof: break
  of cfgSectionStart:
    if e.section notin ["Package", "Author"]:
      echo p.ignoreMsg(e)  # "config.ini(8, 1) Warning: section ignored: Unknown"
  of cfgKeyValuePair:
    if e.key == "legacy":
      echo p.ignoreMsg(e)  # "config.ini(9, 1) Warning: key ignored: legacy"
  else: discard
```

---

## High-Level Config Table API

This API loads the entire config into memory as a nested ordered table. For most applications, this is all you need.

### `newConfig`

```nim
proc newConfig*(): Config
```

**What it does:** Creates and returns an empty `Config` table. Use this as the starting point when you want to build a config from scratch in code and then write it to disk.

```nim
import std/parsecfg

var cfg = newConfig()
cfg.setSectionKey("", "charset", "utf-8")
cfg.setSectionKey("Server", "host", "localhost")
cfg.setSectionKey("Server", "port", "8080")
```

---

### `loadConfig` (from stream)

```nim
proc loadConfig*(stream: Stream, filename: string = "[stream]"): Config
```

**What it does:** Parses a config from an open `Stream` and returns the result as a `Config` table. `filename` is optional and used only for error messages. Parsing stops at the first `cfgError` event.

Use this when you already have a `Stream` — for example, a string stream created from a config string embedded in your code, a network connection, or a decompressed in-memory buffer.

```nim
import std/[streams, parsecfg]

let cfgText = """
[server]
host = localhost
port = 8080
"""
let cfg = loadConfig(newStringStream(cfgText))
echo cfg.getSectionValue("server", "host")  # "localhost"
```

---

### `loadConfig` (from file)

```nim
proc loadConfig*(filename: string): Config
```

**What it does:** Opens the file at `filename`, parses its contents, and returns the result as a `Config`. This is the most common entry point for reading a real config file. Works in both compiled code and NimScript (the NimScript path uses `readFile` internally).

```nim
import std/parsecfg

let cfg = loadConfig("settings.ini")
echo cfg.getSectionValue("Database", "host")
echo cfg.getSectionValue("Database", "port", "5432")  # default if not found
```

---

### `getSectionValue`

```nim
proc getSectionValue*(dict: Config, section, key: string, defaultVal = ""): string
```

**What it does:** Returns the value associated with `key` in the given `section`. If the section does not exist, or the key does not exist within the section, returns `defaultVal` (empty string by default). Never raises an exception.

- To access the **global section** (keys before the first `[...]`), pass `section = ""`.
- To access a `--option` entry, pass its full name including `--`: `key = "--threads"`.

```nim
let cfg = loadConfig("app.ini")

# Read with no default (returns "" if missing)
let host = cfg.getSectionValue("Database", "host")

# Read with an explicit default
let port = cfg.getSectionValue("Database", "port", "5432")

# Read from the global section
let charset = cfg.getSectionValue("", "charset")

# Read a command-line option stored in the config
let threads = cfg.getSectionValue("Build", "--threads")
```

---

### `setSectionKey`

```nim
proc setSectionKey*(dict: var Config, section, key, value: string)
```

**What it does:** Sets `key` to `value` in the given `section`. If the section does not yet exist, it is created. If the key already exists, its value is overwritten. Preserves the existing insertion order for both sections and keys.

- Use `section = ""` for the global section.
- Use a key starting with `--` to store a command-line option entry.

```nim
var cfg = loadConfig("app.ini")
cfg.setSectionKey("Database", "host", "db.example.com")
cfg.setSectionKey("Database", "port", "5432")
cfg.setSectionKey("", "version", "2")     # global key
cfg.setSectionKey("Build", "--opt", "speed")  # --option
cfg.writeConfig("app.ini")
```

---

### `delSectionKey`

```nim
proc delSectionKey*(dict: var Config, section, key: string)
```

**What it does:** Removes a single key from a section. If removing this key would leave the section empty, the section itself is also removed from the table. Does nothing if the section or key does not exist.

```nim
var cfg = loadConfig("app.ini")
cfg.delSectionKey("Author", "website")   # remove one key
cfg.delSectionKey("Legacy", "old_key")   # safe even if section doesn't exist
cfg.writeConfig("app.ini")
```

---

### `delSection`

```nim
proc delSection*(dict: var Config, section: string)
```

**What it does:** Removes an entire section and all its keys from the config table. Does nothing if the section does not exist.

```nim
var cfg = loadConfig("app.ini")
cfg.delSection("DeprecatedSettings")  # remove the whole section
cfg.writeConfig("app.ini")
```

---

### `sections`

```nim
iterator sections*(dict: Config): lent string
```

**What it does:** Iterates over all section names in the `Config`, in insertion order. Yields each section name as a `lent string` (a borrowed reference — do not store it past the loop body without copying it first).

The global section `""` is included if it exists in the table.

```nim
let cfg = loadConfig("app.ini")

for section in cfg.sections:
  echo "Found section: ", section

# If you need to store sections:
var sectionNames: seq[string]
for s in cfg.sections:
  sectionNames.add(s)  # copy via add is safe
```

---

### `writeConfig` (to stream)

```nim
proc writeConfig*(dict: Config, stream: Stream)
```

**What it does:** Serialises the entire `Config` table to an open `Stream` in valid INI format. The output is suitable for reading back with `loadConfig`. Sections are written in insertion order; keys within each section are also in insertion order.

> **Note:** Comments present in the original file are **not preserved**. Because the `Config` type only stores key-value data, any `#` or `;` comments from the source file are lost when you load and re-write the config.

String values that contain special characters or spaces are automatically quoted or triple-quoted as needed. `--option` keys are written with the `--` prefix and `:` as their delimiter.

```nim
import std/[streams, parsecfg]

var cfg = newConfig()
cfg.setSectionKey("Server", "host", "localhost")
cfg.setSectionKey("Server", "port", "8080")

var s = newStringStream()
cfg.writeConfig(s)
echo s.data
# [Server]
# host=localhost
# port=8080
```

---

### `writeConfig` (to file)

```nim
proc writeConfig*(dict: Config, filename: string)
```

**What it does:** Writes the `Config` to a file, creating or overwriting it. This is the companion to `loadConfig(filename)` and the natural way to persist changes.

```nim
var cfg = loadConfig("settings.ini")
cfg.setSectionKey("App", "version", "2.0")
cfg.writeConfig("settings.ini")  # overwrite in place
```

---

### `$` (to string)

```nim
proc `$`*(dict: Config): string
```

**What it does:** Serialises the `Config` to a Nim string in INI format. Equivalent to calling `writeConfig` on a `newStringStream()` and reading the result. Useful for debugging, logging, or diffing config state in tests.

```nim
var cfg = newConfig()
cfg.setSectionKey("", "charset", "utf-8")
cfg.setSectionKey("Package", "name", "myapp")
echo $cfg
# charset=utf-8
# [Package]
# name=myapp
```

---

## Which API to Use?

| Scenario | Recommended API |
|---|---|
| Read a config file, access a few values | `loadConfig(filename)` + `getSectionValue` |
| Create a new config in code, write to disk | `newConfig()` + `setSectionKey` + `writeConfig` |
| Modify an existing config file | `loadConfig` + `setSectionKey`/`delSectionKey` + `writeConfig` |
| Process a very large config file without loading all of it | pull-parser: `open` + `next` loop + `close` |
| Validate a config file structure or check for unknown keys | pull-parser: `open` + `next` loop, use `ignoreMsg`/`warningStr` |
| Embed a config string directly in code | `loadConfig(newStringStream(...))` |
| List all sections in a config | `sections` iterator |

---

## Complete Worked Examples

### Example 1: Full pull-parser loop with all event types

```nim
import std/[streams, parsecfg]

let cfgText = """
charset = "utf-8"
[Package]
name = hello
--threads:on
[Author]
name = nim-lang
website = nim-lang.org
"""

var s = newStringStream(cfgText)
var p: CfgParser
open(p, s, "<string>")
defer: p.close()

while true:
  let e = next(p)
  case e.kind
  of cfgEof:
    echo "--- end of config ---"
    break
  of cfgSectionStart:
    echo "[", e.section, "]"
  of cfgKeyValuePair:
    echo "  ", e.key, " = ", e.value
  of cfgOption:
    echo "  --", e.key, ": ", e.value
  of cfgError:
    echo "ERROR: ", e.msg
    break  # stop on first error; or continue to attempt recovery
```

Output:
```
  charset = utf-8
[Package]
  name = hello
  --threads: on
[Author]
  name = nim-lang
  website = nim-lang.org
--- end of config ---
```

---

### Example 2: Build a config programmatically and write to disk

```nim
import std/parsecfg

var cfg = newConfig()

# Global keys (no section)
cfg.setSectionKey("", "charset", "utf-8")

# [Package] section
cfg.setSectionKey("Package", "name", "myapp")
cfg.setSectionKey("Package", "version", "1.0.0")
cfg.setSectionKey("Package", "--threads", "on")

# [Database] section
cfg.setSectionKey("Database", "host", "localhost")
cfg.setSectionKey("Database", "port", "5432")
cfg.setSectionKey("Database", "password", r"s3cr3t\pass")  # contains backslash

cfg.writeConfig("generated.ini")
echo $cfg
```

---

### Example 3: Load, modify, and save an existing config

```nim
import std/parsecfg

# Load
var cfg = loadConfig("app.ini")

# Read current values
let oldHost = cfg.getSectionValue("Database", "host", "localhost")
echo "Current host: ", oldHost

# Modify
cfg.setSectionKey("Database", "host", "db.production.example.com")
cfg.setSectionKey("Database", "port", "5432")

# Remove a deprecated key
cfg.delSectionKey("Legacy", "oldSetting")

# Remove an entire section that is no longer needed
cfg.delSection("Debug")

# Save back
cfg.writeConfig("app.ini")
```

---

### Example 4: Validate a config against a required schema

Using the pull-parser to check that required sections and keys are present:

```nim
import std/[streams, parsecfg, sets]

proc validateConfig(path: string): bool =
  let required = toHashSet(["host", "port", "password"])
  var found: HashSet[string]
  var inDB = false

  var s = newFileStream(path, fmRead)
  var p: CfgParser
  open(p, s, path)
  defer: p.close()

  while true:
    let e = next(p)
    case e.kind
    of cfgEof: break
    of cfgError:
      echo p.errorStr(e.msg)
      return false
    of cfgSectionStart:
      inDB = (e.section == "Database")
    of cfgKeyValuePair:
      if inDB: found.incl(e.key)
    else: discard

  let missing = required - found
  if missing.len > 0:
    for key in missing:
      echo p.warningStr("required key '" & key & "' missing from [Database]")
    return false
  return true

if validateConfig("app.ini"):
  echo "Config is valid"
else:
  echo "Config validation failed"
```

---

### Example 5: List all sections and their key counts

```nim
import std/parsecfg

let cfg = loadConfig("app.ini")

for section in cfg.sections:
  let label = if section == "": "<global>" else: "[" & section & "]"
  # Count keys in this section
  var count = 0
  if cfg.hasKey(section):
    for _ in cfg[section].pairs:
      inc count
  echo label, ": ", count, " key(s)"
```

---

### Example 6: Round-trip test — write, read back, verify

```nim
import std/parsecfg

var original = newConfig()
original.setSectionKey("", "encoding", "utf-8")
original.setSectionKey("Net", "host", "example.com")
original.setSectionKey("Net", "port", "443")
original.setSectionKey("Net", "--tls", "on")

# Write to a temp file
original.writeConfig("/tmp/roundtrip.ini")

# Read back
let loaded = loadConfig("/tmp/roundtrip.ini")

# Verify
assert loaded.getSectionValue("", "encoding") == "utf-8"
assert loaded.getSectionValue("Net", "host") == "example.com"
assert loaded.getSectionValue("Net", "port") == "443"
assert loaded.getSectionValue("Net", "--tls") == "on"

echo "Round-trip successful"
```
