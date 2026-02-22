# Nim `std/parseopt` Module Reference

> The standard Nim command line parser. Implements parsing of command line arguments into three kinds of tokens: short options, long options, and plain arguments. Conforms to conventional Unix CLI conventions.

---

## Table of Contents

1. [Concepts and Supported Syntax](#concepts-and-supported-syntax)
2. [Types](#types)
   - [CmdLineKind](#cmdlinekind)
   - [OptParser](#optparser)
3. [initOptParser (string)](#initoptparser-string)
4. [initOptParser (seq\[string\])](#initoptparser-seqstring)
5. [next](#next)
6. [cmdLineRest](#cmdlinerest)
7. [remainingArgs](#remainingargs)
8. [getopt (iterator over OptParser)](#getopt-iterator-over-optparser)
9. [getopt (standalone iterator)](#getopt-standalone-iterator)
10. [shortNoVal and longNoVal explained](#shortnoval-and-longnoval-explained)
11. [Complete practical example](#complete-practical-example)
12. [Quick Cheat-Sheet](#quick-cheat-sheet)

---

## Concepts and Supported Syntax

Before diving into the API, it is worth understanding what the parser recognises as valid input.

### Token kinds

Every element on the command line is classified as one of three things:

| Kind | Example input | Meaning |
|---|---|---|
| Short option | `-v`, `-e:5`, `-e=5`, `-abc` | A single-letter flag, optionally with a value |
| Long option | `--verbose`, `--output:file`, `--output=file` | A word-length flag, optionally with a value |
| Argument | `file.txt`, `src/` | Anything that does not begin with `-` |

### Short option clusters

Multiple short options can be clustered into a single token: `-abc` is equivalent to `-a -b -c`. Only the last letter in a cluster can carry a value; `-e:5` sets option `e` to `5`.

### The `--` separator

The bare `--` token is parsed as a long option whose **name is the empty string** (`key == ""`). By convention this signals "everything that follows is a plain argument, not an option". The parser itself does **not** enforce this convention automatically — you must check for it in your own code and call `remainingArgs` (or `cmdLineRest`) to collect the rest.

### Value separators

Values can be attached to options using either `:` or `=`:
- `--output:file.txt`
- `--output=file.txt`
- `-o:file.txt`
- `-o=file.txt`

If the value itself begins with `:` or `=`, double it or alternate the separators:
- `--delim::` → value is `:`
- `--delim=:` → value is `:`

---

## Types

### `CmdLineKind`

```nim
type CmdLineKind* = enum
  cmdEnd,         ## No more tokens; end of command line
  cmdArgument,    ## A plain argument (e.g. a filename)
  cmdLongOption,  ## A long option (--foo)
  cmdShortOption  ## A short option (-f)
```

**What it is:** An enumeration that describes the type of the current token. After each call to `next`, the parser's `kind` field is set to one of these four values.

`cmdEnd` is the sentinel — when you see it, stop processing. The three active values map directly to the three syntactic forms described above.

```nim
import std/parseopt

var p = initOptParser("--verbose -n 42 report.txt")
p.next()
assert p.kind == cmdLongOption   # --verbose
p.next()
assert p.kind == cmdShortOption  # -n
p.next()
assert p.kind == cmdShortOption  # 42 is treated as value of -n... see below
```

---

### `OptParser`

```nim
type OptParser* = object of RootObj
  kind*: CmdLineKind  # Kind of the most recently parsed token
  key*:  string       # Option name or argument text
  val*:  string       # Option value, or "" if none was given
  # (internal fields are not exported)
```

**What it is:** The parser's state object. It holds everything needed to walk through the command line token by token. You never construct it with `OptParser(...)` directly — always use `initOptParser`.

The three exported fields you read after each `next()` call:

- `kind` — what kind of token was just parsed
- `key` — the option name (for options) or the raw text (for arguments)
- `val` — the value string if the option had one; `""` otherwise

```nim
import std/parseopt

var p = initOptParser("--output=report.pdf -v input.md")
p.next()
echo p.kind  # cmdLongOption
echo p.key   # "output"
echo p.val   # "report.pdf"
```

---

## `initOptParser` (string)

```nim
proc initOptParser*(cmdline = "",
                    shortNoVal: set[char] = {},
                    longNoVal: seq[string] = @[];
                    allowWhitespaceAfterColon = true): OptParser
```

**What it does:** Creates and returns a new `OptParser` from a **single string** that is first split into tokens (the same way a shell would split a command line). If `cmdline` is the empty string `""`, the parser reads the actual process command line via the `os` module instead. Raises `ValueError` if the platform cannot provide the real command line.

The optional parameters are explained in detail in the [shortNoVal and longNoVal](#shortnoval-and-longnoval-explained) section.

`allowWhitespaceAfterColon` controls whether `--key: value` (space between `:` and the value) is accepted. It defaults to `true`.

**When to use it:** When you have a command line as a single string — for example, in tests, or when reading a command from a config file — and want to parse it the same way as real `argv`.

```nim
import std/parseopt

# Parse a literal string (typical in tests)
var p = initOptParser("--left --debug:3 -l -r:2")

# Parse the actual process arguments (production code)
var q = initOptParser()

# Parse with NoVal hints
var r = initOptParser("--verbose -j4",
                      shortNoVal = {'v'},
                      longNoVal  = @["verbose"])
```

---

## `initOptParser` (seq[string])

```nim
proc initOptParser*(cmdline: seq[string],
                    shortNoVal: set[char] = {},
                    longNoVal: seq[string] = @[];
                    allowWhitespaceAfterColon = true): OptParser
```

**What it does:** Same purpose as the string overload, but accepts a **sequence of strings** where each element is already a separate token. This is the more natural form when working with `os.commandLineParams()` (which returns a `seq[string]`) because no shell-splitting step is needed. If `cmdline` is empty (`@[]`), it falls back to the real process command line.

**When to use it:** In production programs where you receive `argv` as `seq[string]` — which is the common case.

```nim
import std/parseopt, std/os

# Use the real command line directly as a seq — no re-splitting needed
var p = initOptParser(commandLineParams())

# Or pass an explicit list, useful in unit tests
var q = initOptParser(@["--output", "report.pdf", "-v", "input.md"])
```

---

## `next`

```nim
proc next*(p: var OptParser)
```

**What it does:** Advances the parser by **one token** and updates `p.kind`, `p.key`, and `p.val` to describe that token. When the end of the command line is reached, sets `p.kind = cmdEnd` and returns immediately on all subsequent calls.

This is the fundamental operation of the module — `getopt` is simply a loop around `next`. You use `next` directly when you need fine-grained control: for example, to stop parsing in the middle, to handle `--` specially, or to inspect `remainingArgs` at a particular point.

**Important:** `next` mutates the parser in place (`var OptParser`), so the parser must be declared as `var`.

```nim
import std/parseopt

var p = initOptParser("--left -r:2 file.txt")

p.next()
assert p.kind == cmdLongOption and p.key == "left" and p.val == ""

p.next()
assert p.kind == cmdShortOption and p.key == "r" and p.val == "2"

p.next()
assert p.kind == cmdArgument and p.key == "file.txt"

p.next()
assert p.kind == cmdEnd   # Nothing left; further calls stay here
```

**Handling `--` (end-of-options marker):**

```nim
import std/parseopt

var p = initOptParser("--verbose -- foo.txt bar.txt")
while true:
  p.next()
  if p.kind == cmdEnd: break
  if p.kind == cmdLongOption and p.key == "":
    # "--" found: everything that follows is a raw argument
    echo "Remaining: ", p.remainingArgs  # @["foo.txt", "bar.txt"]
    break
  echo p.key
```

---

## `cmdLineRest`

```nim
proc cmdLineRest*(p: OptParser): string
```

**What it does:** Returns all tokens that have **not yet been consumed** by the parser, reassembled into a single shell-quoted string. This is essentially the not-yet-parsed tail of the command line in a form that can be passed to another process or interpreted by a shell.

> **Platform note:** This procedure is only available on platforms where `quoteShellCommand` is declared (Windows and POSIX). It is not available in NimScript.

**When to use it:** When your program delegates some arguments to a sub-process. After processing your own options, hand off the remainder as a ready-to-use shell string.

```nim
import std/parseopt

var p = initOptParser("--left -r:2 -- foo.txt bar.txt")
while true:
  p.next()
  if p.kind == cmdLongOption and p.key == "":  # Hit "--"
    break
  if p.kind == cmdEnd: break

# Everything after "--" as a shell-ready string
echo p.cmdLineRest  # "foo.txt bar.txt"
```

---

## `remainingArgs`

```nim
proc remainingArgs*(p: OptParser): seq[string]
```

**What it does:** Returns a `seq[string]` of all tokens that have **not yet been consumed** by the parser. Unlike `cmdLineRest`, this gives you the raw individual strings, not a single shell-quoted one. This is easier to work with in Nim code.

**When to use it:** When you want to collect leftover arguments as a list — for example, all positional file arguments that appear after a `--` separator, or when you stop parsing early and need to pass the rest somewhere.

```nim
import std/parseopt

var p = initOptParser("--left -r:2 -- foo.txt bar.txt")
while true:
  p.next()
  if p.kind == cmdLongOption and p.key == "":  # Hit "--"
    break
  if p.kind == cmdEnd: break

let rest = p.remainingArgs
assert rest == @["foo.txt", "bar.txt"]

for f in rest:
  echo "Will process: ", f
```

---

## `getopt` (iterator over OptParser)

```nim
iterator getopt*(p: var OptParser): tuple[kind: CmdLineKind, key, val: string]
```

**What it does:** A convenience iterator that wraps `next` in a loop. It resets the parser to the beginning (`pos = 0`, `idx = 0`) and then repeatedly calls `next`, yielding each `(kind, key, val)` tuple until `cmdEnd` is reached — at which point it stops without yielding. You never see `cmdEnd` in the loop body.

**When to use it:** When you want to iterate over a pre-built `OptParser` using a `for` loop instead of a manual `while true` / `next` / `break` loop. Slightly more expressive than the manual form.

**Important difference from the standalone `getopt`:** This overload takes an existing `OptParser` by `var` reference, so it **resets** the parser each time it is called. If you have already advanced the parser with `next`, calling this iterator will restart from the beginning.

```nim
import std/parseopt

var p = initOptParser("--output=report.pdf -v input.md")

for kind, key, val in p.getopt():
  case kind
  of cmdLongOption, cmdShortOption:
    echo "Option: ", key, if val != "": " = " & val else: ""
  of cmdArgument:
    echo "File: ", key
  of cmdEnd:
    discard  # never reached here
```

Output:
```
Option: output = report.pdf
Option: v
File: input.md
```

---

## `getopt` (standalone iterator)

```nim
iterator getopt*(cmdline: seq[string] = @[],
                 shortNoVal: set[char] = {},
                 longNoVal: seq[string] = @[]):
         tuple[kind: CmdLineKind, key, val: string]
```

**What it does:** A self-contained convenience iterator that creates an `OptParser` internally and immediately iterates over it. When `cmdline` is empty (`@[]`), it reads the real process command line. You never need to create an `OptParser` explicitly when using this form.

**When to use it:** In straightforward programs where you process command line arguments exactly once, from start to finish, without needing to pause, backtrack, or inspect `remainingArgs`. It is the most concise option for the common case.

```nim
import std/parseopt

var output = "a.out"
var verbose = false
var files: seq[string]

for kind, key, val in getopt():
  case kind
  of cmdShortOption, cmdLongOption:
    case key
    of "o", "output":  output  = val
    of "v", "verbose": verbose = true
  of cmdArgument:
    files.add(key)
  of cmdEnd:
    discard  # never reached inside getopt loop

echo output, " | ", verbose, " | ", files
```

**With explicit argument list (useful in tests):**

```nim
import std/parseopt

for kind, key, val in getopt(@["--output=out.txt", "-v", "main.nim"]):
  echo kind, " | ", key, " | ", val
```

---

## `shortNoVal` and `longNoVal` explained

These two optional parameters to `initOptParser` (and the standalone `getopt`) teach the parser which options **do not accept a value**. This unlocks additional input syntax that would otherwise be ambiguous.

### The problem: ambiguous clusters and space-separated values

Consider the command line `-j4`. Without any hints, the parser has no way to know if this means "option `j` with value `4`" or "options `j` and `4`". Similarly, `--jobs 4` — is `4` the value of `--jobs`, or a separate argument?

By default, the parser takes the safe, conservative interpretation:
- `-j4` → short option `j`, then short option `4` (two separate options)
- `--jobs 4` → long option `jobs` (no value), then argument `4`

### The solution: declare which options take no value

Once you tell the parser which options **never** have values, it can infer that all *other* options **do** have values, enabling the natural syntax:

```nim
import std/parseopt

# Without hints: -j4 is two options; --first bar is one option + one argument
var p1 = initOptParser("-j4 --first bar")
for kind, key, val in p1.getopt():
  echo kind, " | ", key, " | ", val
# cmdShortOption | j  |
# cmdShortOption | 4  |
# cmdLongOption  | first |
# cmdArgument    | bar |

# With hints: 'j' is not in shortNoVal, so -j4 means j=4
#             "first" is not in longNoVal, so --first bar means first=bar
var p2 = initOptParser("-j4 --first bar",
                       shortNoVal = {'v'},      # 'v' takes no value; 'j' does
                       longNoVal  = @["verbose"]) # "verbose" takes no value; "first" does
for kind, key, val in p2.getopt():
  echo kind, " | ", key, " | ", val
# cmdShortOption | j     | 4
# cmdLongOption  | first | bar
```

### When to use it

Use `shortNoVal` and `longNoVal` when your CLI uses:
- `-j8` style (make-style parallelism count)
- `--jobs 8` style (GNU-style space-separated value)

Keep these sets/sequences **in sync with your actual option definitions**. If you add a new flag-only option but forget to add it to `shortNoVal`/`longNoVal`, the parser will unexpectedly consume the next token as its value.

---

## Complete practical example

A realistic program that combines all the features:

```nim
import std/parseopt, std/os

# Defaults
var
  outputFile  = "a.out"
  verbosity   = 0
  jobs        = 1
  inputFiles: seq[string]

proc writeHelp() =
  echo """Usage: mytool [options] <files>
Options:
  -h, --help          Show this help
  -v, --verbose       Increase verbosity (repeatable)
  -j<N>, --jobs <N>   Number of parallel jobs (default: 1)
  -o, --output <file> Output file (default: a.out)
  --                  Stop option processing
"""

var p = initOptParser(commandLineParams(),
                      shortNoVal = {'h', 'v'},
                      longNoVal  = @["help", "verbose"])

while true:
  p.next()
  case p.kind
  of cmdEnd: break
  of cmdLongOption and p.key == "":   # bare "--"
    for f in p.remainingArgs: inputFiles.add(f)
    break
  of cmdShortOption, cmdLongOption:
    case p.key
    of "h", "help":    writeHelp(); quit(0)
    of "v", "verbose": inc verbosity
    of "j", "jobs":    jobs = p.val.parseInt
    of "o", "output":  outputFile = p.val
    else:
      echo "Unknown option: ", p.key; quit(1)
  of cmdArgument:
    inputFiles.add(p.key)

echo "Output:    ", outputFile
echo "Verbosity: ", verbosity
echo "Jobs:      ", jobs
echo "Files:     ", inputFiles
```

---

## Quick Cheat-Sheet

| Task | Code |
|---|---|
| Create parser from string | `initOptParser("--foo bar")` |
| Create parser from seq | `initOptParser(commandLineParams())` |
| Create parser for real argv | `initOptParser()` |
| Advance one token | `p.next()` |
| Check token type | `p.kind` → `cmdShortOption`, `cmdLongOption`, `cmdArgument`, `cmdEnd` |
| Read option name | `p.key` |
| Read option value | `p.val` (empty string if no value) |
| Iterate conveniently | `for kind, key, val in p.getopt()` |
| Iterate without explicit parser | `for kind, key, val in getopt()` |
| Get unparsed args as seq | `p.remainingArgs` |
| Get unparsed args as string | `p.cmdLineRest` |
| Enable `-j4` syntax | `shortNoVal = {'h', 'v'}` (list flags that take no value) |
| Enable `--jobs 4` syntax | `longNoVal = @["help", "verbose"]` |
| Detect `--` separator | `p.kind == cmdLongOption and p.key == ""` |
