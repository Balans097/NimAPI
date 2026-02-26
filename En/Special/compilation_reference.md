# `compilation.nim` — Module Reference

> **Part of Nim's Standard Library (`system` module)**  
> All symbols are available without any import — they are injected by the compiler.

---

## Overview

`compilation.nim` exposes the **compile-time introspection and metaprogramming interface** of the Nim compiler. Every symbol in this module is evaluated at compile time, not at runtime — they let your code ask the compiler questions about itself: *What version of Nim is this? Is this flag defined? Can this type do that operation? What does this source file embed?*

This is the backbone of Nim's conditional compilation and build-time code generation. You use these symbols whenever you write `when defined(...)`, embed a file into a binary as a constant, shell out to an external tool during build, or write documentation examples that are automatically tested.

Two important things to note up front:

1. **Nothing in this module runs at runtime.** Every procedure and constant here is resolved by the compiler during the compilation pass. The resulting binary contains no trace of these calls — only their outputs (a `bool`, a `string`, a computed constant).
2. **Many of these are "magic" symbols.** The `{.magic: "...".}` pragma means the compiler has hardcoded knowledge of what these do. Their Nim bodies (if any) are dummies; the compiler substitutes its own logic.

---

## Version Constants

### `NimMajor*`

```nim
const NimMajor* {.intdefine.}: int = 2
```

The **major** version number of the Nim compiler being used. The `{.intdefine.}` pragma means this value can be overridden at compile time with `-d:NimMajor=X`, but in practice you should never do that — it is set correctly by the compiler itself.

Use this together with `NimMinor` and `NimPatch` to guard code that requires a minimum Nim version:

```nim
when (NimMajor, NimMinor, NimPatch) >= (2, 0, 0):
  echo "Running on Nim 2.x or newer"
else:
  echo "Running on Nim 1.x"
```

Tuple comparison in Nim is lexicographic, so this one-liner correctly tests the full three-part version.

---

### `NimMinor*`

```nim
const NimMinor* {.intdefine.}: int = 2
```

The **minor** version number. By Nim convention: **even** minor numbers are stable releases; **odd** minor numbers are development builds. So `2.0`, `2.2`, `2.4` are releases, while `2.1`, `2.3` are in-progress development versions.

```nim
when NimMinor mod 2 == 1:
  {.warning: "You are building with a development version of Nim.".}
```

---

### `NimPatch*`

```nim
const NimPatch* {.intdefine.}: int = 8
```

The **patch** version number. Same parity convention as `NimMinor`: even = release, odd = development.

```nim
# Require at least 2.0.4 for a specific bugfix:
when (NimMajor, NimMinor, NimPatch) < (2, 0, 4):
  {.error: "This library requires Nim >= 2.0.4".}
```

---

## Compile-Time State

### `nimvm*`

```nim
let nimvm* {.magic: "Nimvm", compileTime.}: bool = false
```

A special boolean that is **`true` inside the Nim VM** (i.e. inside `static:` blocks, `const` initialisers, and compile-time procedure calls) and **`false` in normal compiled code**. It can only be used in `when` expressions — never in runtime `if` statements.

This is the standard way to write code that behaves differently depending on whether it is running at compile time or at runtime:

```nim
proc serialize(x: int): string =
  when nimvm:
    # Compile-time path: no OS calls available, use a simple approach
    $x
  else:
    # Runtime path: can use full formatting
    formatInt(x, padding = 10)
```

The `{.profiler: off.}` pragma around its declaration ensures it does not appear in profiler output.

---

### `isMainModule*`

```nim
const isMainModule* {.magic: "IsMainModule".}: bool = false
```

Is `true` **only in the module that was passed directly to the compiler** — i.e. the entry point. In any module that was imported, it is `false`.

This is Nim's idiomatic equivalent of Python's `if __name__ == "__main__":`. Its most common use is embedding self-tests directly inside a library module, so they run when you compile the module standalone but are silently skipped when the module is imported:

```nim
# In mylib.nim:
proc add(a, b: int): int = a + b

when isMainModule:
  assert add(2, 3) == 5
  echo "All tests passed"
  # This block is completely absent from the binary
  # when mylib is imported by another module.
```

---

### `CompileDate*`

```nim
const CompileDate* {.magic: "CompileDate".}: string = "0000-00-00"
```

The **UTC date** at which the binary was compiled, as a `YYYY-MM-DD` string. Set by the compiler at build time. The default `"0000-00-00"` is never seen in practice — it is the placeholder the compiler replaces.

```nim
const buildInfo = "Built on: " & CompileDate
echo buildInfo   # e.g. "Built on: 2025-07-15"
```

Useful for embedding build metadata in executables, version strings, or log output.

---

### `CompileTime*`

```nim
const CompileTime* {.magic: "CompileTime".}: string = "00:00:00"
```

The **UTC time** at which the binary was compiled, as an `HH:MM:SS` string.

```nim
const versionBanner = "v1.0 compiled " & CompileDate & " at " & CompileTime
```

Together, `CompileDate` and `CompileTime` let you embed a precise, reproducible build timestamp into any binary. Note that using them makes builds non-reproducible (the binary changes every time), which is a deliberate trade-off.

---

## Compile-Time Introspection Procedures

### `defined`

```nim
proc defined*(x: untyped): bool {.magic: "Defined", noSideEffect, compileTime.}
```

Checks whether a compile-time **symbol** `x` has been defined via the `-d:x` compiler flag, a `{.define.}` pragma, or is a built-in platform/OS symbol.

This is the primary mechanism for **conditional compilation** in Nim. The symbol name is not a string — it is passed as a bare identifier directly to the compiler:

```nim
when defined(release):
  # Optimised build — skip expensive checks
  discard
else:
  # Debug build — full validation
  doExpensiveCheck()
```

The compiler provides many built-in symbols you can test without any `-d:` flag:

| Symbol | True when… |
|---|---|
| `windows` | Compiling for Windows |
| `linux` | Compiling for Linux |
| `macosx` | Compiling for macOS |
| `posix` | Compiling for any POSIX system |
| `amd64` | 64-bit x86 architecture |
| `i386` | 32-bit x86 architecture |
| `arm` | ARM architecture |
| `release` | `-d:release` was passed |
| `debug` | No release/danger flag |
| `danger` | `-d:danger` was passed |
| `gcArc` | Using ARC memory management |
| `gcOrc` | Using ORC memory management |
| `nimscript` | Running inside NimScript |

```nim
when defined(linux) and defined(amd64):
  echo "64-bit Linux"

# Custom flags:
# Compile with: nim c -d:myFeature app.nim
when defined(myFeature):
  includeMyFeature()
```

---

### `declared`

```nim
proc declared*(x: untyped): bool {.magic: "Declared", noSideEffect, compileTime.}
```

Checks whether the identifier `x` has been **declared anywhere visible** from the current point (i.e. in any imported module or in any enclosing scope). `x` must be an identifier or a qualified name like `module.symbol`.

The canonical use case is writing code that gracefully adapts to what a library provides, without hard compilation failures if a symbol is missing:

```nim
when not declared(strutils.toUpper):
  proc toUpper(s: string): string =
    # Provide our own fallback implementation
    result = s
    for c in result.mitems: c = c.toUpperAscii()
```

Another common pattern is feature detection in cross-platform code:

```nim
when declared(os.getEnv):
  let home = os.getEnv("HOME")
else:
  let home = ""
```

---

### `declaredInScope`

```nim
proc declaredInScope*(x: untyped): bool {.magic: "DeclaredInScope", noSideEffect, compileTime.}
```

Like `declared`, but **stricter**: only returns `true` if `x` is declared in the **current lexical scope**, not in any enclosing scope or imported module. `x` must be a simple (unqualified) identifier.

This is useful in macros and generic code where you want to check whether a particular local binding has been established at the exact point of evaluation:

```nim
block:
  when declaredInScope(result):
    echo "we are inside a proc, 'result' is available"

# Outside a proc:
when not declaredInScope(result):
  echo "no implicit 'result' here"
```

---

### `compiles`

```nim
proc compiles*(x: untyped): bool {.magic: "Compiles", noSideEffect, compileTime.}
```

Attempts to **type-check and compile the expression `x`** without emitting any actual code, and returns `true` if it succeeds without errors, `false` otherwise. The expression is never executed.

This is Nim's primary tool for **duck-typing checks** — testing whether a type supports a certain operation before committing to using it:

```nim
type Animal = concept a
  when compiles(a.speak()):
    true

proc makeNoise[T](x: T) =
  when compiles(x.speak()):
    x.speak()
  elif compiles(x.sound):
    echo x.sound
  else:
    echo "..."
```

Practical use with generics — check before you call, not after:

```nim
proc tryAdd[T](a, b: T): T =
  when compiles(a + b):
    a + b
  else:
    {.error: "Type " & $T & " does not support '+'".}
```

The body of `compiles` is `discard` — it is a dummy; the compiler's magic handles everything.

---

### `astToStr`

```nim
proc astToStr*[T](x: T): string {.magic: "AstToStr", noSideEffect.}
```

Converts the **source code AST of expression `x`** into a string, as it appears in the source. This is purely a debugging and diagnostic tool — it lets you print what the compiler sees, not what the value of `x` is at runtime.

```nim
let a = 42
echo astToStr(a)          # "a"
echo astToStr(1 + 2 * 3)  # "1 + 2 * 3"
echo astToStr(isMainModule) # "isMainModule"
```

This is most useful inside macros and `{.error.}` or `{.warning.}` pragmas, where you want to include the user's original expression text in a diagnostic message:

```nim
macro mustBePositive(x: int): untyped =
  if x.intVal <= 0:
    error("Expected positive value, got: " & astToStr(x))
```

---

## Documentation Utility

### `runnableExamples`

```nim
proc runnableExamples*(rdoccmd = "", body: untyped) {.magic: "RunnableExamples".}
```

Marks a block of code as a **runnable documentation example**. This has two distinct behaviours depending on context:

- **In normal builds (debug/release):** the body is completely ignored and compiled away — zero runtime overhead.
- **During `nim doc` documentation generation:** each `runnableExamples` block is extracted, placed in its own temporary `.nim` file, compiled, and executed. If it fails, the documentation build fails. This ensures documentation examples are always correct and up to date.

The optional `rdoccmd` parameter passes extra compiler flags to use when building that specific example:

```nim
proc myProc*(x: int): string =
  ## Converts an integer to a string.
  ##
  runnableExamples:
    # Basic usage — compiled and run during `nim doc`
    assert myProc(42) == "42"
    assert myProc(0)  == "0"

  runnableExamples "-d:myFlag":
    # Only compiled when -d:myFlag is set
    when defined(myFlag):
      assert myProc(1) == "1"

  runnableExamples "-r:off":
    # Compiled but NOT run (useful for examples that open browsers,
    # launch processes, or have side effects):
    import std/browsers
    openDefaultBrowser("https://nim-lang.org")

  $x
```

The examples are put into their own module scope, so they cannot accidentally reference non-exported symbols — this catches documentation bugs where an example works only because of internal access.

---

## Compile-Time Option Queries

### `compileOption` (on/off)

```nim
proc compileOption*(option: string): bool {.magic: "CompileOption", noSideEffect.}
```

Queries the current value of a **boolean (`on`/`off`) compiler option** by name. Returns `true` if the option is currently `on`. The option name is the lowercase version of the compiler switch (without `--`).

This is useful when you want to adapt code based on what optimisation or safety flags are active at the call site — options can even be pushed and popped locally with `{.push.}` / `{.pop.}`:

```nim
proc safeDivide(a, b: float): float =
  when compileOption("infchecks"):
    if b == 0.0: raise newException(DivByZeroDefect, "division by zero")
  a / b
```

Testing scope-local option changes:

```nim
static: doAssert not compileOption("floatchecks")

{.push floatChecks: on.}
static: doAssert compileOption("floatchecks")
{.pop.}

static: doAssert not compileOption("floatchecks")
```

Commonly queried boolean options: `assertions`, `boundchecks`, `floatchecks`, `infchecks`, `nanchecks`, `overflowchecks`, `rangechecks`, `stacktrace`, `linetrace`, `threads`.

---

### `compileOption` (enum)

```nim
proc compileOption*(option, arg: string): bool {.magic: "CompileOptionArg", noSideEffect.}
```

Queries a **multi-value (enum) compiler option** — checks whether the option named `option` is currently set to the value `arg`. Both arguments are strings.

```nim
when compileOption("opt", "size"):
  echo "Optimising for binary size"

when compileOption("opt", "speed"):
  echo "Optimising for speed"

when compileOption("gc", "orc"):
  echo "Using ORC garbage collector"
```

This is the right tool when `compileOption(option)` (on/off) would be insufficient because the option has more than two possible values. Commonly queried enum options: `opt` (`none`, `speed`, `size`), `gc` (`refc`, `arc`, `orc`, `boehm`, `go`, `none`), `backend` (`c`, `cpp`, `js`, `objc`), `exceptionSystem` (`setjmp`, `cpp`, `goto`, `quirky`).

---

## Path Introspection

### `currentSourcePath`

```nim
template currentSourcePath*: string = instantiationInfo(-1, true).filename
```

Expands to the **full filesystem path of the source file** where this template is called, at compile time. Because it is a template (not a proc), it is instantiated at the call site — so the path is always the calling file's path, never `compilation.nim` itself.

```nim
echo currentSourcePath
# e.g. "/home/user/myproject/mylib.nim"

# Get the directory of the current source file:
import std/os
const thisDir = currentSourcePath.parentDir()
const dataFile = thisDir / "data" / "config.json"
```

The primary use is constructing **compile-time paths relative to the source file**, so that embedded resources are found correctly regardless of where the compiler is invoked from. This is more robust than using a hardcoded absolute path or relying on the working directory.

---

## File and Process Embedding

### `staticRead` / `slurp`

```nim
proc staticRead*(filename: string): string {.magic: "Slurp".}
proc slurp*(filename: string): string {.magic: "Slurp".}
```

Reads a file **at compile time** and returns its contents as a `string` constant. The file is embedded directly into the compiled binary — the resulting program has no runtime dependency on the file. `slurp` is simply an alias for `staticRead`; they are identical.

The filename is resolved relative to the **source file** that contains the call (not the working directory at compile time).

```nim
const htmlTemplate = staticRead("templates/index.html")
const privateKey    = slurp("certs/server.pem")
const wordList      = staticRead("data/words.txt")
```

The entire file becomes a compile-time string constant. The maximum file size you can embed is limited by the free memory available on your build machine — for very large files, this can be a practical constraint.

```nim
# Combined with currentSourcePath for robustness:
const schema = staticRead(currentSourcePath.parentDir() / "schema.sql")
```

---

### `staticExec` / `gorge`

```nim
proc staticExec*(command: string, input = "", cache = ""): string {.magic: "StaticExec".}
proc gorge*(command: string, input = "", cache = ""): string {.magic: "StaticExec".}
```

Executes an **external shell command at compile time** and captures its combined stdout+stderr output as a string. `gorge` is an alias for `staticExec`; they are identical.

This is a powerful tool for embedding dynamic build information or running code generators as part of compilation:

```nim
const gitRevision  = staticExec("git rev-parse --short HEAD")
const buildHost    = staticExec("hostname")
const platformInfo = staticExec("uname -srm")

const versionString = "v1.0 rev:" & gitRevision & " built on " & buildHost
```

**The `input` parameter** passes a string to the command's stdin:

```nim
const processed = staticExec("awk '{print toupper($0)}'", input = "hello world")
# processed == "HELLO WORLD\n"
```

**The `cache` parameter** enables caching: the output is stored in `nimcache/` and reused on subsequent compilations as long as `command & input & cache` produces the same combined key. Use a version string as the cache key to invalidate it when the tool changes:

```nim
const optimisedDFA = staticExec("dfaoptimizer", "my_grammar", "tool_v2.1")
# Re-runs dfaoptimizer only when the input or version key changes.
# Use --forceBuild to bypass the cache entirely.
```

Can also be used inside pragmas:

```nim
{.passc: staticExec("pkg-config --cflags openssl").}
{.passl: staticExec("pkg-config --libs openssl").}
```

---

### `gorgeEx`

```nim
proc gorgeEx*(command: string, input = "", cache = ""): tuple[output: string, exitCode: int]
```

Like `staticExec` / `gorge`, but also returns the **exit code** of the command. Useful when you need to detect failure rather than assuming the command always succeeds.

```nim
let (output, code) = gorgeEx("git rev-parse --short HEAD")
const gitRevision =
  if code == 0: output.strip()
  else: "unknown"
```

```nim
let (libFlags, err) = gorgeEx("pkg-config --libs libpng")
when err != 0:
  {.error: "libpng not found — please install it.".}
else:
  {.passl: libFlags.}
```

The `{.noinit.}` pragma on the return tuple means the compiler will not zero-initialise it before the assignment — a minor performance note that has no effect on correctness.

---

## Summary Table

| Symbol | Kind | What it provides |
|---|---|---|
| `NimMajor` | `const int` | Nim compiler major version |
| `NimMinor` | `const int` | Nim compiler minor version (odd = dev) |
| `NimPatch` | `const int` | Nim compiler patch version (odd = dev) |
| `nimvm` | `let bool` | `true` inside Nim VM / compile-time context |
| `isMainModule` | `const bool` | `true` only in the directly-compiled entry module |
| `CompileDate` | `const string` | UTC date of compilation (`YYYY-MM-DD`) |
| `CompileTime` | `const string` | UTC time of compilation (`HH:MM:SS`) |
| `defined` | proc | Is a `-d:symbol` or built-in symbol defined? |
| `declared` | proc | Is an identifier visible from here? |
| `declaredInScope` | proc | Is an identifier in the current scope? |
| `compiles` | proc | Can an expression be type-checked without errors? |
| `astToStr` | proc | Source text of an expression as a string |
| `runnableExamples` | proc | Marks code as a doc-tested example |
| `compileOption` (1 arg) | proc | Is a boolean compiler option currently `on`? |
| `compileOption` (2 args) | proc | Is an enum compiler option set to a specific value? |
| `currentSourcePath` | template | Filesystem path of the current source file |
| `staticRead` / `slurp` | proc | Embed a file's contents as a compile-time string |
| `staticExec` / `gorge` | proc | Run a shell command at compile time, capture output |
| `gorgeEx` | proc | Like `staticExec`, but also returns the exit code |
