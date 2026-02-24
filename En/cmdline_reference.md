# Cmdline Module — Reference

---

## What This Module Does

`cmdline` provides the standard facilities for reading command-line arguments passed to a Nim program at launch. It answers three questions a program typically needs to ask:

- How many arguments were passed? → `paramCount`
- What is argument number *N*? → `paramStr`
- Give me all arguments as a list → `commandLineParams`

Additionally, it provides `parseCmdLine` — a low-level parser that splits a raw command-line string into tokens according to platform-specific quoting rules. This is useful when you have a command line stored in a string rather than received from the OS directly.

---

## Availability and Platform Constraints

Unlike most standard library modules, several procedures in `cmdline` are **conditionally compiled** and may not exist on every target. This is a fundamental property of the module, not an edge case.

| Procedure | When it may be absent |
|-----------|----------------------|
| `paramCount` | When building a **dynamic library** (`--app:lib`) on Posix. No portable way to obtain argv from a shared object exists. Also absent on Genode (raises `OSError` instead). |
| `paramStr` | Same conditions as `paramCount`. |
| `commandLineParams` | Same conditions as the two above; also triggers a **compile-time error** rather than a runtime error when used in an unsupported library context. |

The correct way to handle this in portable code is to guard usage with `declared()`:

```nim
when declared(commandLineParams):
  let args = commandLineParams()
else:
  # fallback for dynamic library context
  let args: seq[string] = @[]
```

---

## How Arguments Are Numbered

The indexing convention follows the **C argv** model, but with one difference from C:

- Index `0`: the executable itself (its path as passed by the OS). **Do not use `paramStr(0)` to get the executable path** — use `getAppFilename()` from `std/os` instead, which is more reliable.
- Index `1` through `paramCount()`: the actual arguments the user typed.
- `paramCount()` returns `0` if no arguments were provided (not `-1`, unlike C's `argc - 1`).

So the valid range for `paramStr` when retrieving user arguments is `1..paramCount()`. Passing any integer outside the range `0..paramCount()` raises `IndexDefect`.

---

## Windows Implementation Note

On Windows, Nim does **not** rely on the C runtime's `argv` array. Instead it calls `GetCommandLineW()` and runs the result through its own `parseCmdLine` parser. This is necessary because Nim supports GUI applications (`WinMain` entry point) that have no access to a pre-parsed argv. As a beneficial side effect, the argument parsing behaviour on Windows is always consistent regardless of which C compiler built the binary.

The parsed result is cached per-thread in `ownArgv` after the first call to `paramCount` or `paramStr`, so subsequent calls on the same thread are free.

---

## Exported Procedures

---

### `parseCmdLine`

```nim
proc parseCmdLine*(c: string): seq[string]
```

#### Description

Takes a raw command-line string `c` and splits it into a sequence of individual argument tokens. The splitting rules differ between Windows and Posix and are applied at compile time based on the target platform.

This procedure is **pure** (marked `noSideEffect`) — it has no interaction with the OS or any global state. It works entirely from the input string.

**When is this useful?** When you have a command line in a string that did not come directly from `os.getAppFilename()` or the OS — for example, a command line stored in a configuration file, a database, or received over a network. For parsing the actual program's own arguments, `commandLineParams` is simpler and more direct.

#### Parsing rules — Windows

The Windows rules implement the [Microsoft C runtime parsing conventions](https://msdn.microsoft.com/en-us/library/17w5ykft.aspx):

- Arguments are separated by spaces or tabs.
- The caret (`^`) is **not** an escape character at this level (it is processed by `cmd.exe` before the program receives the string).
- Text inside double quotes is treated as a single argument, regardless of whitespace inside.
- A backslash immediately before a `"` is an escape for the quote. The quote is treated as a literal `"` character, not a delimiter.
- Backslashes are literal everywhere **except** immediately before a `"`. When preceding a `"`:
  - An even number of backslashes `\\...\\` produces half as many literal backslashes, and the `"` is a delimiter.
  - An odd number of backslashes `\\...\` produces `(N-1)/2` literal backslashes, and the `"` becomes a literal quote character.

#### Parsing rules — Posix

- Arguments are separated by whitespace (spaces, tabs, newlines).
- Text inside single quotes `'...'` is a single argument; no escape processing occurs inside single quotes.
- Text inside double quotes `"..."` is a single argument; no escape processing occurs inside double quotes on this implementation (unlike a real shell, which processes `\"` inside double quotes).
- No backslash escaping outside quotes.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `c` | `string` | The raw command-line string to parse. |

#### Return value

`seq[string]` — each element is one parsed argument token, with quoting and escaping resolved. The executable name is included if present in `c` (as the first element).

#### Examples

```nim
import cmdline

# Basic splitting
let args = parseCmdLine("hello world foo")
assert args == @["hello", "world", "foo"]

# Quoted string containing spaces → single token
let args2 = parseCmdLine("""say "hello world" now""")
assert args2 == @["say", "hello world", "now"]

# Empty string
let args3 = parseCmdLine("")
assert args3 == @[]

# Multiple spaces are collapsed (they are just delimiters)
let args4 = parseCmdLine("a   b   c")
assert args4 == @["a", "b", "c"]
```

```nim
# Windows: backslash-quote escaping
when defined(windows):
  # \" → literal "
  let a = parseCmdLine("""say \"quoted\" word""")
  assert a == @["say", "\"quoted\"", "word"]

  # \\" → one backslash, then end of quoted section
  let b = parseCmdLine("""path\\" extra""")
  # One pair of backslashes → one literal backslash; " closes no quote (none open)
  assert b[0] == "path\\"
```

```nim
# Posix: single quotes preserve spaces verbatim
when not defined(windows):
  let a = parseCmdLine("echo 'hello world'")
  assert a == @["echo", "hello world"]

  # Double quotes also group
  let b = parseCmdLine("""echo "hello world" """)
  assert b == @["echo", "hello world"]
```

---

### `paramCount`

```nim
proc paramCount*(): int {.tags: [ReadIOEffect].}
```

#### Description

Returns the number of command-line arguments passed to the program, **not counting** the program name (index 0). If the program was invoked with no arguments, returns `0`.

This is equivalent to `argc - 1` in C.

After calling this, you can safely iterate `for i in 1..paramCount()` and call `paramStr(i)` on each index.

**Availability:** Not defined when building a Posix dynamic library (`--app:lib`). Use `declared(paramCount)` to check.

#### Return value

A non-negative integer. `0` means no arguments were provided.

#### Examples

```nim
# Invoked as: ./myapp foo bar baz
import cmdline

echo paramCount()   # → 3
```

```nim
# Invoked as: ./myapp  (no arguments)
import cmdline

echo paramCount()   # → 0
```

```nim
# Portable guard
when declared(paramCount):
  echo "Got ", paramCount(), " arguments"
else:
  echo "Running as a library; no argv available"
```

---

### `paramStr`

```nim
proc paramStr*(i: int): string {.tags: [ReadIOEffect].}
```

#### Description

Returns the `i`-th command-line argument as a string. The valid range for user-supplied arguments is `1..paramCount()`. Passing an index outside `0..paramCount()` raises `IndexDefect`.

Calling `paramStr(0)` is technically possible and returns an OS-specific string that usually contains the executable path as passed on the command line. However, this is unreliable — it may be an empty string, a relative path, or an absolute path, depending on how the program was invoked. Always use `getAppFilename()` from `std/os` to get the executable path.

**Availability:** Not defined when building a Posix dynamic library (`--app:lib`). Use `declared(paramStr)` to check.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `i` | `int` | The argument index. Valid user argument range: `1..paramCount()`. |

#### Return value

The argument string at position `i`.

#### Exceptions

`IndexDefect` is raised if `i < 0` or `i > paramCount()`.

#### Examples

```nim
# Invoked as: ./myapp Alice 42 true
import cmdline

echo paramStr(1)   # → "Alice"
echo paramStr(2)   # → "42"
echo paramStr(3)   # → "true"
```

```nim
import cmdline

# Iterating all arguments
for i in 1..paramCount():
  echo "Arg ", i, ": ", paramStr(i)
```

```nim
import cmdline

# Safe access with bounds check
proc getArg(i: int): string =
  if i >= 1 and i <= paramCount():
    paramStr(i)
  else:
    ""
```

---

### `commandLineParams`

```nim
proc commandLineParams*(): seq[string]
```

#### Description

Returns all command-line arguments as a sequence of strings, **excluding** the executable name (index 0). This is the most convenient way to access all arguments at once.

Internally it simply calls `paramStr(i)` for `i` in `1..paramCount()` and collects the results. It is entirely equivalent to building that sequence manually, but more readable.

**Do not use this to get the executable path.** For that, call `getAppFilename()` from `std/os`.

**Availability:** Not defined when building a Posix dynamic library (`--app:lib`). In that context, using this proc triggers a **compile-time error** with the message `"commandLineParams() unsupported by dynamic libraries"`. Use `declared(commandLineParams)` to guard against this.

#### Return value

`seq[string]` — the list of arguments the user passed, in order. Empty sequence if no arguments were given.

#### Examples

```nim
# Invoked as: ./myapp --verbose input.txt output.txt
import cmdline

let args = commandLineParams()
echo args          # → @["--verbose", "input.txt", "output.txt"]
echo args.len      # → 3
```

```nim
import cmdline

# Simple argument presence check
let args = commandLineParams()
if "--help" in args:
  echo "Usage: myapp [options] <input>"
  quit(0)
```

```nim
import cmdline

# Portable usage
when declared(commandLineParams):
  let args = commandLineParams()
  for arg in args:
    echo "Processing: ", arg
else:
  echo "No command-line access in library mode"
```

```nim
import cmdline, strutils

# Parsing a simple key=value argument list
let args = commandLineParams()
for arg in args:
  let parts = arg.split('=', maxsplit = 1)
  if parts.len == 2:
    echo "Key: ", parts[0], "  Value: ", parts[1]
  else:
    echo "Flag: ", arg
```

---

## Choosing the Right Tool

| Need | Tool |
|------|------|
| All arguments as a list | `commandLineParams()` |
| Specific argument by index | `paramStr(i)` |
| Number of arguments | `paramCount()` |
| Parse a command-line string from a config file / network / database | `parseCmdLine(s)` |
| Parse flags like `--verbose`, `-o file`, named options | `std/parseopt` |
| Get the executable's own path | `std/os.getAppFilename()` |

---

## Interaction with `std/parseopt`

`commandLineParams` and `parseCmdLine` give you raw strings. If your program accepts options in the conventional Unix style (`--flag`, `-f value`, `--key=value`), use `std/parseopt` instead, which builds on top of these raw accessors and handles the option syntax for you.

```nim
import std/parseopt

var p = initOptParser(commandLineParams())
for kind, key, val in p.getopt():
  case kind
  of cmdLongOption:  echo "Long option: ", key, " = ", val
  of cmdShortOption: echo "Short option: ", key
  of cmdArgument:    echo "Argument: ", key
  of cmdEnd:         break
```
