# `std/parseopt` Module Reference — Nim Standard Library

Standard command-line argument parser for Nim.

## Table of Contents
- [Module Overview](#1-module-overview)
- [Supported Syntax](#2-supported-syntax)
- [Types and Enums](#3-types-and-enums)
- [Initialization: `initOptParser`](#4-initialization-initoptparser)
- [Iterator `getopt`](#5-iterator-getopt)
- [Manual Traversal: `next`](#6-manual-traversal-next)
- [Parameters `shortNoVal` and `longNoVal`](#7-parameters-shortnoval-and-longnoval)
- [Parser Modes (`CliMode`)](#8-parser-modes-climode)
- [Getting Remaining Arguments: `cmdLineRest` and `remainingArgs`](#9-getting-remaining-arguments-cmdlinerest-and-remainingargs)
- [Deprecated API](#10-deprecated-api)
- [Practical Examples](#11-practical-examples)
- [Quick Reference Table](#12-quick-reference-table)

---

## 1. Module Overview

The `std/parseopt` module implements a full-featured command-line argument parser. It supports short and long options, option arguments, short flag bundling, and several compatibility modes.

### Importing
```nim
import std/parseopt
```

### Related Modules

| Module | Purpose |
|--------|---------|
| `std/os` | Low-level access to arguments (`paramCount`, `paramStr`) |
| `std/parseutils` | Utilities for parsing tokens, numbers, identifiers |
| `std/strutils` | String operations (useful for processing option values) |
| `std/parsecfg` | Configuration file parser |

---

## 2. Supported Syntax

### Basic Formats

| Format | Type | Description |
|--------|------|-------------|
| `-a` | Short option without value | Flag |
| `-a:5` | Short option with value (via `:`) | Option with explicit delimiter |
| `-a=5` | Short option with value (via `=`) | Option with explicit delimiter |
| `-cde` | Bundled short options | Three flags `c`, `d`, `e` in one argument |
| `-fgh=5` | Bundled options with value on last | Flags `f`, `g`, then `h=5` |
| `--foo` | Long option without value | Flag |
| `--foo:bar` | Long option with value | Via `:` delimiter |
| `--foo=bar` | Long option with value | Via `=` delimiter |
| `file.txt` | Argument | Anything not starting with `-` |

### Special Cases

Values starting with a delimiter are valid:
```
--foo::     → option foo, value ":"
--foo=:     → option foo, value ":"
--foo:=     → option foo, value "="
--foo==     → option foo, value "="
```

The `--` delimiter is a special long option with an empty name (`key == ""`). Traditionally means "all following tokens are arguments". The parser does not handle `--` automatically — the programmer must catch this case and call `remainingArgs`.

Whitespace around delimiters (in `NimMode` and `LaxMode`):
```
--foo: bar   → option foo, value "bar"  (space after : is allowed)
--foo =bar   → option foo, value "bar"  (space before = is allowed)
```

---

## 3. Types and Enums

### `CmdLineKind` — Type of Recognized Token
```nim
type
  CmdLineKind* = enum
    cmdEnd,          ## End of argument stream
    cmdArgument,     ## Argument (not an option), e.g., filename
    cmdLongOption,   ## Long option: --foo
    cmdShortOption   ## Short option: -f
```

This enum is stored in the `kind` field of the `OptParser` object and returned by the `getopt` iterator.

| Value | When it occurs | Contents of `key` | Contents of `val` |
|-------|---------------|-------------------|-------------------|
| `cmdEnd` | Tokens exhausted | `" "` | `" "` |
| `cmdArgument` | Token without `-` | The argument itself | `" "` |
| `cmdLongOption` | Token `--foo` | `"foo"` | Value or `" "` |
| `cmdShortOption` | Token `-f` | `"f"` | Value or `" "` |

Special case for `--`: `kind = cmdLongOption`, `key = ""`, `val = ""`.

### `CliMode` — Parser Behavior Mode
```nim
type
  CliMode* = enum
    LaxMode,  ## Most flexible mode (POSIX-like)
    NimMode,  ## Standard Nim mode (default)
    GnuMode   ## GNU-compatible mode
```

Detailed mode descriptions are in [Section 8](#8-parser-modes-climode).

### `OptParser` — Parser Object
```nim
type
  OptParser* = object of RootObj
    kind*: CmdLineKind  ## Type of last recognized token
    key*:  string       ## Option name or argument text
    val*:  string       ## Option value (empty if not specified)
    # ... internal fields (pos, idx, cmds, rules, etc.)
```

Fields `kind`, `key`, and `val` are the only public fields of the object. The rest are implementation details and not directly accessible.

After each call to `next` or each iteration of `getopt`, these fields are updated.

---

## 4. Initialization: `initOptParser`

There are two public overloads: one accepting a string, and one accepting `seq[string]`.

### From String
```nim
proc initOptParser*(cmdline = "";
                    shortNoVal: set[char] = {};
                    longNoVal: seq[string] = @[];
                    mode: CliMode = NimMode): OptParser
```

The `cmdline` string is split into tokens according to shell quoting rules (respecting quotes). If `cmdline` is empty, real process arguments are read via `os.paramStr`.

```nim
import std/parseopt

# From string (convenient for tests)
var p1 = initOptParser("--left --debug:3 -l -r:2")

# From string with flags that take no values specified
var p2 = initOptParser("--left -lr",
                        shortNoVal = {'l', 'r'},
                        longNoVal = @["left"])

# From real command line
var p3 = initOptParser()

# With GNU mode
var p4 = initOptParser("--foo=bar -c val", mode = GnuMode)
```

### From `seq[string]`
```nim
proc initOptParser*(cmdline: seq[string];
                    shortNoVal: set[char] = {};
                    longNoVal: seq[string] = @[];
                    mode: CliMode = NimMode): OptParser
```

Accepts a pre-split list of tokens — this is more precise than a string because it doesn't require re-parsing shell quotes. Optimal for passing `commandLineParams()`.

```nim
import std/parseopt, std/os

# From real arguments — the most correct way
var p = initOptParser(commandLineParams())

# From explicit list (e.g., in tests)
var p2 = initOptParser(@["--output", "file.txt", "-v", "input.nim"])

# With no-value options
var p3 = initOptParser(
  @["--verbose", "-n", "10"],
  shortNoVal = {'v'},
  longNoVal = @["verbose", "help"]
)
```

### `initOptParser` Parameters Summary

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `cmdline` | `string` or `seq[string]` | `""` / `@[]` | Arguments to parse. Empty → real process arguments |
| `shortNoVal` | `set[char]` | `{}` | Short options that do **not** accept values |
| `longNoVal` | `seq[string]` | `@[]` | Long options that do **not** accept values |
| `mode` | `CliMode` | `NimMode` | Parser behavior mode |

---

## 5. Iterator `getopt`

The `getopt` iterator is the main and most convenient way to iterate over all tokens. It exists in two variants.

### From Existing `OptParser`
```nim
iterator getopt*(p: var OptParser): tuple[kind: CmdLineKind, key, val: string]
```

Iterates over tokens stored in an already-created `OptParser`. Automatically stops at `cmdEnd` — no need to check it inside a `for` loop. When using `case` with `cmdEnd`, use `assert(false)` or `discard`.

```nim
import std/parseopt

var p = initOptParser("--left --debug:3 -l -r:2")
for kind, key, val in p.getopt():
  case kind
  of cmdArgument:
    echo "Argument: ", key
  of cmdLongOption, cmdShortOption:
    if val == "":
      echo "Flag: ", key
    else:
      echo "Option: ", key, " = ", val
  of cmdEnd:
    assert false  # never reached
```

### Directly from Arguments
```nim
iterator getopt*(cmdline: seq[string] = @[];
                 shortNoVal: set[char] = {};
                 longNoVal: seq[string] = @[];
                 mode: CliMode = NimMode):
    tuple[kind: CmdLineKind, key, val: string]
```

Creates an `OptParser` internally and iterates immediately. Parameters are the same as for `initOptParser`. If `cmdline` is empty, real process arguments are used.

```nim
import std/parseopt

# Shortest way to process real arguments
for kind, key, val in getopt():
  case kind
  of cmdShortOption, cmdLongOption:
    echo key, " = ", val
  of cmdArgument:
    echo "file: ", key
  of cmdEnd: discard

# With explicit list and settings
for kind, key, val in getopt(
    @["--output=file.txt", "-v"],
    shortNoVal = {'v'},
    longNoVal = @["verbose"]):
  discard
```

### Full Example Handling All Token Types
```nim
import std/parseopt

let cmds = "-ab -e:5 --foo --bar=20 file.txt".parseCmdLine()
var output: seq[string] = @[]

for kind, key, val in getopt(cmds):
  case kind
  of cmdEnd: break
  of cmdShortOption, cmdLongOption:
    if val == "":
      output.add("Option: " & key)
    else:
      output.add("Option and value: " & key & ", " & val)
  of cmdArgument:
    output.add("Argument: " & key)

doAssert output == @[
   "Option: a",
   "Option: b",
   "Option and value: e, 5",
   "Option: foo",
   "Option and value: bar, 20",
   "Argument: file.txt"
]
```

---

## 6. Manual Traversal: `next`

```nim
proc next*(p: var OptParser)
```

Advances the parser by one token. After the call, fields `p.kind`, `p.key`, and `p.val` contain data about the current token. When arguments are exhausted, `p.kind` becomes `cmdEnd`.

Used when finer control over the parsing process is needed: e.g., breaking mid-stream, skipping a token, or handling `--` followed by retrieving remaining arguments.

```nim
import std/parseopt

var p = initOptParser("--left -r:2 file.txt")

p.next()
doAssert p.kind == cmdLongOption and p.key == "left"

p.next()
doAssert p.kind == cmdShortOption and p.key == "r" and p.val == "2"

p.next()
doAssert p.kind == cmdArgument and p.key == "file.txt"

p.next()
doAssert p.kind == cmdEnd   # end of stream
```

### Manual Loop with `next`
```nim
import std/parseopt

var p = initOptParser("--verbose -o output.txt data.csv")
while true:
  p.next()
  case p.kind
  of cmdEnd: break
  of cmdLongOption:
    echo "long: --", p.key, if p.val != "": "=" & p.val else: ""
  of cmdShortOption:
    echo "short: -", p.key, if p.val != "": "=" & p.val else: ""
  of cmdArgument:
    echo "argument: ", p.key
```

---

## 7. Parameters `shortNoVal` and `longNoVal`

The `shortNoVal` and `longNoVal` parameters are the key mechanism for controlling whether an option can accept a value without an explicit delimiter.

### Default Behavior (Empty `shortNoVal`/`longNoVal`)

When both parameters are empty (default):
- A short option accepts a value only via an explicit delimiter (`-k:val`, `-k=val`) or direct adjacency (`-kval` → value `val` for option `k`).
- `-j4` without a delimiter is parsed as two separate flags: `j` and `4`.
- `--foo bar` → `--foo` is a flag without a value, `bar` is a separate argument.

```nim
import std/parseopt

var p = initOptParser("-j4 --first bar")
# shortNoVal empty → -j and 4 are two different flags
for kind, key, val in p.getopt():
  echo kind, ": ", key, " = ", val
# Output:
# cmdShortOption: j =
# cmdShortOption: 4 =
# cmdLongOption:  first =
# cmdArgument:    bar =
```

### With Specified `shortNoVal`/`longNoVal`

When parameters are specified, the parser knows which options do not expect a value. Then:
- `-j4` with `shortNoVal = {'j'}`: `-j4` → flag `j`, then flag `4` (as before)
- `-j4` with `shortNoVal` **not** containing `j`: `-j4` → option `j`, value `"4"`
- `--foo bar` with `longNoVal = @["bar"]`: `--foo` is a flag, `bar` is an argument
- `--foo bar` with `longNoVal` **not** containing `"foo"`: `--foo` → option with value `bar` (next token)

```nim
import std/parseopt

var p = initOptParser("-j4 --first bar",
                      shortNoVal = {'c'},      # 'j' not in list → can accept value
                      longNoVal = @["second"]) # "first" not in list → can accept value
for kind, key, val in p.getopt():
  echo kind, ": ", key, " = ", val
# Output:
# cmdShortOption: j = 4
# cmdLongOption:  first = bar
```

### Enabling Next-Argument Value-Taking

⚠️ **Important**: Next-argument value-taking (value from the next token) is enabled **only** when `shortNoVal` or `longNoVal` is non-empty. If all your options accept values, pass a dummy element:

```nim
# All options accept values, but we want to enable --foo bar
var p = initOptParser(cmdline,
  shortNoVal = {'\0'},   # dummy element
  longNoVal = @[""])     # dummy element
```

### Parser Does Not Forbid Explicit Values for No-Val Options

If a user passes an explicit value (`--foo:bar`) for an option marked as `longNoVal`, the parser still recognizes that value — it does not raise an error. This allows detecting erroneous usage in application code:

```nim
import std/[sequtils, os, parseopt]

let cmds = "-n:9 --foo:bar".parseCmdLine()
let parsed = toSeq(cmds.getopt(shortNoVal = {'n'}, longNoVal = @["foo"]))

for (kind, key, val) in parsed:
  case kind
  of cmdShortOption, cmdLongOption:
    if key in ["n", "foo"] and val != "":
      echo "Error: option ", key, " does not accept values, but received: ", val
  else: discard

# parsed == @[(cmdShortOption, "n", "9"), (cmdLongOption, "foo", "bar")]
```

### Behavior Summary Table

| Syntax | `shortNoVal = {}` | `shortNoVal = {'j'}` | `shortNoVal` does not contain `j` |
|--------|-----------------|----------------------|-----------------------------------|
| `-j4` | flags `j`, `4` | flags `j`, `4` | option `j = "4"` |
| `-j:4` | option `j = "4"` | option `j = "4"` | option `j = "4"` |
| `-j 4` (LaxMode) | flag `j`, argument `4` | flag `j`, argument `4` | option `j = "4"` |

---

## 8. Parser Modes (`CliMode`)

⚠️ Modes `LaxMode` and `GnuMode` are experimental and may change in future Nim versions.

The parser mode is set via the `mode` parameter in `initOptParser` or `getopt`. `NimMode` is used by default.

### `NimMode` (Default)

Standard Nim mode. A balance between convenience and predictability.

**Characteristics:**
- Both delimiters `:` and `=` are allowed
- Whitespace before and after delimiter is allowed: `--foo: bar`, `--foo =bar`
- Short flag bundling: `-abc` → flags `a`, `b`, `c`
- Adjacent values for short options: `-kval` → option `k = "val"` (only if `k` not in `shortNoVal`)
- Next-argument (`-k val`) not supported by default (only with non-empty `shortNoVal`)
- Values starting with `-` are treated as new options

```nim
import std/parseopt

# NimMode (default)
for kind, key, val in getopt(@["--foo:bar", "--baz =qux", "-k5"]):
  echo key, " = ", val
# foo = bar
# baz = qux
# k = 5  (adjacent value)
```

### `LaxMode`

Most flexible mode. Combines `NimMode` with POSIX-like handling of short options.

**In addition to `NimMode`:**
- `-c val` is supported: next token becomes value of `-c`
- `-abc val`: flags `a`, `b`, then option `c = "val"`
- Values starting with `-` can be passed as option arguments: `-n -10`

```nim
import std/parseopt

# LaxMode: -c val works
for kind, key, val in getopt(
    @["-c", "hello", "--level", "5"],
    shortNoVal = {'\0'},  # enable next-arg
    longNoVal = @[""],
    mode = LaxMode):
  echo key, " = ", val
# c = hello
# level = 5
```

### `GnuMode`

Follows GNU getopt conventions. Stricter regarding delimiters.

**Characteristics:**
- Only `=` is a delimiter (`:` is **not** a delimiter)
- Whitespace around `=` is **not** allowed: `--foo =bar` → option `foo` without value, argument `=bar`
- `-c val` is supported (like LaxMode)
- Values starting with `-` are allowed as option arguments
- Colon `:` has no special meaning

```nim
import std/parseopt

# GnuMode: only = as delimiter
for kind, key, val in getopt(
    @["--foo=bar", "--baz:qux"],
    mode = GnuMode):
  echo key, " = ", val
# foo = bar
# baz:qux =    ← `:` not a delimiter → entire string is option name
```

### Mode Comparison Table

| Behavior | NimMode | LaxMode | GnuMode |
|----------|---------|---------|---------|
| Delimiter `:` | ✅ | ✅ | ❌ |
| Delimiter `=` | ✅ | ✅ | ✅ |
| Whitespace before delimiter | ✅ | ✅ | ❌ |
| Whitespace after delimiter | ✅ | ✅ | ❌ |
| `-kval` (adjacent value) | ✅ | ✅ | ✅ |
| `-k val` (next token) | ❌* | ✅* | ✅* |
| Values starting with `-` | ❌ | ✅ | ✅ |
| Bundling `-abc` | ✅ | ✅ | ✅ |

\* Requires non-empty `shortNoVal`/`longNoVal`

### Parsing Identical Strings in Different Modes
```nim
import std/parseopt

let cmdline = @["--foo:bar", "--baz", "=qux", "-c", "-10"]

for mode in [NimMode, LaxMode, GnuMode]:
  echo "=== ", mode, " ==="
  for kind, key, val in getopt(cmdline,
      shortNoVal = {'\0'}, longNoVal = @["baz"], mode = mode):
    echo "   ", kind, ": '", key, "' = '", val, "'"
```

---

## 9. Getting Remaining Arguments: `cmdLineRest` and `remainingArgs`

Both procedures are intended to retrieve tokens that have not yet been processed by the parser. A typical scenario is handling the `--` separator.

### `remainingArgs`
```nim
proc remainingArgs*(p: OptParser): seq[string]
```

Returns a `seq[string]` of unprocessed tokens. Preferred method — returns tokens in the form they were provided, unchanged.

```nim
import std/parseopt

var p = initOptParser("--left -r:2 -- foo.txt bar.txt")
while true:
  p.next()
  if p.kind == cmdLongOption and p.key == "":  # caught "--"
    break

let rest = p.remainingArgs
doAssert rest == @["foo.txt", "bar.txt"]

# Now rest can be used as a list of files
for f in rest:
  echo "Processing file: ", f
```

### `cmdLineRest`
```nim
proc cmdLineRest*(p: OptParser): string
```

Returns the remainder of the command line as a single string, with restored escaping (via `quoteShellCommand`). Available only on platforms where `quoteShellCommand` is defined (POSIX / Windows).

```nim
import std/parseopt

var p = initOptParser("--left -r:2 -- foo.txt bar.txt")
while true:
  p.next()
  if p.kind == cmdLongOption and p.key == "":
    break

echo p.cmdLineRest   # => "foo.txt bar.txt"
```

### When to Use Which:
- `remainingArgs` — for programmatic processing: returns structured `seq[string]`
- `cmdLineRest` — for passing the remainder to another program or shell command as a whole

---

## 10. Deprecated API

### `allowWhitespaceAfterColon` (deprecated)
```nim
proc initOptParser*(cmdline = ""; shortNoVal: set[char] = {};
                    longNoVal: seq[string] = @[];
                    allowWhitespaceAfterColon: bool): OptParser
  {.deprecated: "`allowWhitespaceAfterColon` is deprecated, use parser modes instead".}
```

Old version of `initOptParser` that controlled whitespace around delimiters. Replaced by the `mode` parameter:

| Old Call | New Equivalent |
|----------|---------------|
| `initOptParser(cmd, allowWhitespaceAfterColon = true)` | `initOptParser(cmd, mode = NimMode)` ← default |
| `initOptParser(cmd, allowWhitespaceAfterColon = false)` | `initOptParser(cmd, mode = GnuMode)` |

---

## 11. Practical Examples

### Minimal CLI with Two Options and One Argument
```nim
import std/parseopt

proc writeHelp() =
  echo """
Usage: mytool [options] <file>

Options:
  -v, --verbose     Verbose output
  -o, --output=FILE File for writing result (default: stdout)
  -h, --help        Show this help
"""

proc writeVersion() =
  echo "mytool v1.0.0"

var
  verbose  = false
  output   = ""
  filename = ""

for kind, key, val in getopt(shortNoVal = {'v', 'h'},
                              longNoVal = @["verbose", "help"]):
  case kind
  of cmdEnd: break
  of cmdArgument:
    filename = key
  of cmdShortOption, cmdLongOption:
    case key
    of "h", "help":    writeHelp(); quit(0)
    of "v", "verbose": verbose = true
    of "o", "output":  output = val
    else:
      echo "Unknown option: ", key
      quit(1)

if filename == "":
  writeHelp()
  quit(1)

echo "File: ", filename
echo "Verbose: ", verbose
echo "Output: ", if output == "": "(stdout)" else: output
```

### Default Values and Validation
```nim
import std/parseopt, std/strutils

# Default values
var
  host    = "localhost"
  port    = 8080
  debug   = false
  workers = 4

for kind, key, val in getopt(shortNoVal = {'d'},
                              longNoVal = @["debug"]):
  case kind
  of cmdEnd: break
  of cmdArgument:
    echo "Extra argument: ", key
  of cmdShortOption, cmdLongOption:
    case key
    of "h", "host":
      host = val
    of "p", "port":
      try:
        port = parseInt(val)
        if port notin 1..65535:
          raise newException(ValueError, "port out of range")
      except ValueError as e:
        echo "Error: invalid port: ", e.msg
        quit(1)
    of "d", "debug":
      debug = true
    of "w", "workers":
      workers = parseInt(val)
    else:
      echo "Unknown option: ", key
      quit(1)

echo "Server: ", host, ":", port
echo "Debug: ", debug
echo "Workers: ", workers
```

### Handling `--` and Positional Arguments After It
```nim
import std/parseopt

var
  options: seq[(string, string)] = @[]
  files:   seq[string] = @[]

var p = initOptParser()
while true:
  p.next()
  case p.kind
  of cmdEnd: break
  of cmdArgument:
    files.add p.key
  of cmdLongOption:
    if p.key == "":   # encountered "--"
      files.add p.remainingArgs()   # everything after "--" are files
      break
    options.add((p.key, p.val))
  of cmdShortOption:
    options.add((p.key, p.val))

echo "Options: ", options
echo "Files: ", files
```

### Parser with Subcommands (git-style)
```nim
import std/parseopt

type SubCommand = enum
  scNone, scBuild, scTest, scRun

var
  subcmd  = scNone
  verbose = false
  output  = ""
  args:   seq[string] = @[]

var p = initOptParser()
p.next()

# First token is subcommand
if p.kind == cmdArgument:
  case p.key
  of "build": subcmd = scBuild
  of "test":  subcmd = scTest
  of "run":   subcmd = scRun
  else:
    echo "Unknown command: ", p.key
    quit(1)
else:
  echo "Expected subcommand (build, test, run)"
  quit(1)

# Remaining tokens are subcommand options
for kind, key, val in p.getopt():
  case kind
  of cmdEnd: break
  of cmdArgument: args.add key
  of cmdShortOption, cmdLongOption:
    case key
    of "v", "verbose": verbose = true
    of "o", "output":  output = val
    else:
      echo "Unknown option: ", key

echo "Command: ", subcmd
echo "Verbose: ", verbose
echo "Output:  ", output
echo "Arguments: ", args
```

### Usage in Tests (Without Real Arguments)
```nim
import std/parseopt

proc parseMyArgs(cmdline: seq[string]): tuple[verbose: bool, files: seq[string]] =
  result = (verbose: false, files: @[])
  for kind, key, val in getopt(cmdline, shortNoVal = {'v'},
                                         longNoVal = @["verbose"]):
    case kind
    of cmdArgument:
      result.files.add key
    of cmdShortOption, cmdLongOption:
      if key in ["v", "verbose"]:
        result.verbose = true
    of cmdEnd: break

# Unit tests:
let r1 = parseMyArgs(@["-v", "file.txt"])
doAssert r1.verbose == true
doAssert r1.files == @["file.txt"]

let r2 = parseMyArgs(@["--verbose", "a.nim", "b.nim"])
doAssert r2.verbose == true
doAssert r2.files == @["a.nim", "b.nim"]

let r3 = parseMyArgs(@["file.txt"])
doAssert r3.verbose == false
```

### GNU Mode — Compatibility with `getopt_long`
```nim
import std/parseopt

# GNU-style: only =, no whitespace around delimiter, values may start with -
for kind, key, val in getopt(
    @["--output=result.txt", "--count=-5", "-v"],
    shortNoVal = {'v'},
    longNoVal = @["help"],
    mode = GnuMode):
  case kind
  of cmdShortOption, cmdLongOption:
    echo key, " => ", val
  of cmdArgument:
    echo "arg: ", key
  of cmdEnd: break
# output => result.txt
# count  => -5          ← value starts with -, allowed in GnuMode
# v      =>
```

---

## 12. Quick Reference Table

### Types and Enums

| Name | Kind | Description |
|------|------|-------------|
| `CmdLineKind` | `enum` | Type of recognized token |
| `cmdEnd` | value | End of arguments |
| `cmdArgument` | value | Positional argument |
| `cmdLongOption` | value | Long option `--foo` |
| `cmdShortOption` | value | Short option `-f` |
| `CliMode` | `enum` | Parser behavior mode |
| `NimMode` | value | Standard mode (default) |
| `LaxMode` | value | Flexible POSIX-like mode |
| `GnuMode` | value | GNU-compatible mode |
| `OptParser` | `object` | Parser object |
| `OptParser.kind` | field `CmdLineKind` | Type of last token |
| `OptParser.key` | field `string` | Option name / argument text |
| `OptParser.val` | field `string` | Option value (or `""`) |

### Procedures and Iterators

| Name | Signature | Description |
|------|-----------|-------------|
| `initOptParser` | `(string, ...) → OptParser` | Initialization from string |
| `initOptParser` | `(seq[string], ...) → OptParser` | Initialization from list |
| `next` | `(var OptParser)` | Parse next token |
| `getopt` | `(var OptParser) → iter` | Iterate existing parser |
| `getopt` | `(seq[string], ...) → iter` | Create parser and iterate |
| `remainingArgs` | `(OptParser) → seq[string]` | Unprocessed tokens (list) |
| `cmdLineRest` | `(OptParser) → string` | Unprocessed tokens (string) |

### Supported Syntax by Mode

| Syntax | NimMode | LaxMode | GnuMode |
|--------|---------|---------|---------|
| `--foo:bar` | ✅ | ✅ | ❌ |
| `--foo=bar` | ✅ | ✅ | ✅ |
| `--foo: bar` (space) | ✅ | ✅ | ❌ |
| `--foo =bar` (space) | ✅ | ✅ | ❌ |
| `--foo bar` (next-arg) | ✅* | ✅* | ✅* |
| `-kval` (adjacent) | ✅ | ✅ | ✅ |
| `-k val` (next-arg) | ❌ | ✅* | ✅* |
| `-k -10` (value with `-`) | ❌ | ✅* | ✅* |
| `-abc` (bundling) | ✅ | ✅ | ✅ |

\* Requires non-empty `shortNoVal`/`longNoVal`

---

*Document compiled from the source code of `std/parseopt` from the Nim standard library. Compatible with Nim 2.x.*
