# NimScript Module Reference

> **Module:** `nimscript`  
> **Part of:** Nim's Runtime Library  
> **Purpose:** Provides the standard API for `.nims` configuration scripts and Nimble package files. Lets you automate builds, manage files, control compiler settings, and define tasks — all in Nim itself.

---

## Table of Contents

1. [Constants](#constants)
2. [Script Mode](#script-mode)
3. [Compiler Control](#compiler-control)
4. [Configuration Keys](#configuration-keys)
5. [Project & Path Information](#project--path-information)
6. [File System Operations](#file-system-operations)
7. [Directory Navigation](#directory-navigation)
8. [External Process Execution](#external-process-execution)
9. [Environment Variables](#environment-variables)
10. [Standard Input](#standard-input)
11. [String Utilities](#string-utilities)
12. [Tasks](#tasks)
13. [Nimble Package Metadata](#nimble-package-metadata)

---

## Constants

### `buildOS`

```nim
const buildOS*: string
```

**What it is:** The operating system on which the *build itself is running* — the machine that is compiling your code. This can differ from `system.hostOS` when you are cross-compiling (e.g., building a Windows binary from a Linux machine).

**Why it matters:** Use this inside `.nims` scripts to apply platform-specific compiler flags or file paths *based on where the build runs*, not where the output will run.

```nim
# Apply Linux-specific flags only when building on Linux
if buildOS == "linux":
  switch("passC", "-march=native")
```

---

### `buildCPU`

```nim
const buildCPU*: string
```

**What it is:** The CPU architecture of the build machine. Like `buildOS`, it may differ from `system.hostCPU` in cross-compilation scenarios.

```nim
if buildCPU == "amd64":
  switch("passC", "-mavx2")
```

---

## Script Mode

### `ScriptMode` (type)

```nim
type ScriptMode* {.pure.} = enum
  Silent   # No output about operations
  Verbose  # Print each operation as it runs
  Whatif   # Dry-run: print operations but do NOT execute them
```

**What it is:** An enum that controls how file system functions (`mkDir`, `rmDir`, `cpFile`, etc.) behave — whether they are silent, verbose, or perform a dry run.

### `mode` (variable)

```nim
var mode*: ScriptMode
```

**What it is:** A global variable you set to change the logging behaviour of all NimScript file operations. Defaults to `Silent`.

```nim
# See exactly what your script is doing
mode = ScriptMode.Verbose
mkDir "build/output"
# prints: [NimScript] mkDir: build/output

# Test your script without touching the file system
mode = ScriptMode.Whatif
rmDir "dist"
# prints: [NimScript] rmDir: dist  (but does NOT delete anything)
```

> **Tip:** `Whatif` mode is invaluable for debugging destructive operations like `rmDir` or `mvFile` before running them for real.

---

## Compiler Control

### `switch`

```nim
proc switch*(key: string, val = "")
```

**What it is:** The primary way to pass compiler flags from a `.nims` script. Every command-line switch you would normally pass to `nim c` can be set programmatically here. The `key` is the flag name, and `val` is its value.

```nim
switch("opt", "speed")          # -d equivalent: optimize for speed
switch("checks", "on")          # enable all runtime checks
switch("define", "myFeature")   # define a symbol
```

---

### `--` (template, two forms)

```nim
template `--`*(key, val: untyped)
template `--`*(key: untyped)
```

**What it is:** A syntactic shortcut for `switch`. Instead of writing `switch("key", "value")`, you write `--key:value`. This makes `.nims` files read almost like a regular `nim.cfg` configuration file, but with the full power of Nim code around it.

```nim
--opt:speed
--path:"./src"
--hint:"[Conf]:off"   # disable the Conf hint
--listCmd             # single-argument form (no value)
```

---

### `warning`

```nim
proc warning*(name: string; val: bool)
```

**What it is:** Enables or disables a named compiler warning. Pass `true` to turn it on, `false` to silence it. Internally calls `switch` with the appropriate `"on"` / `"off"` suffix.

```nim
warning("UnusedImport", false)   # suppress "unused import" warnings
warning("Deprecated", true)      # make sure deprecation warnings are shown
```

---

### `hint`

```nim
proc hint*(name: string; val: bool)
```

**What it is:** Same idea as `warning`, but for compiler *hints* (informational messages, less severe than warnings).

```nim
hint("Processing", false)   # hide "Processing file..." lines
hint("Conf", false)         # hide config-file loading messages
```

---

### `patchFile`

```nim
proc patchFile*(package, filename, replacement: string)
```

**What it is:** Redirects the compiler away from a standard file in a package and toward a replacement file of your choosing. This is useful for substituting stdlib or third-party modules with patched versions without modifying the originals.

**Important note:** If `replacement` is a relative path, it is resolved relative to the `.nims` file itself — Nim's `--path` is **not** consulted.

```nim
# Replace asyncdispatch with a local patched version
patchFile("stdlib", "asyncdispatch", "patches/asyncdispatch_patched")
```

---

### `getCommand`

```nim
proc getCommand*(): string
```

**What it is:** Returns the Nim compiler command currently being executed — for example `"c"`, `"js"`, `"cpp"`, `"build"`, `"help"`. Useful when you need to conditionally behave differently depending on how the compiler was invoked.

```nim
if getCommand() == "js":
  switch("define", "useNodeJs")
```

---

### `setCommand`

```nim
proc setCommand*(cmd: string; project = "")
```

**What it is:** Overrides the compiler command that will continue after the NimScript finishes. You can optionally provide a `project` file path to compile. This is how `task` definitions change what the compiler does next.

```nim
# Force compilation via the C backend regardless of what was invoked
setCommand "c"

# Compile a specific file
setCommand("c", "src/main.nim")
```

---

### `cppDefine`

```nim
proc cppDefine*(define: string)
```

**What it is:** Informs Nim that a given identifier is a C preprocessor `#define`. Nim will then always mangle the name correctly when generating C/C++ code, preventing name clashes.

```nim
cppDefine("SOME_C_MACRO")
```

---

## Configuration Keys

These three procs read and write Nim's internal configuration key-value store. Keys follow the dotted notation used in `nim.cfg` (e.g., `"gcc.options.always"`).

### `put`

```nim
proc put*(key, value: string)
```

**What it is:** Sets a configuration entry. Equivalent to writing a line in `nim.cfg`.

```nim
put("gcc.options.always", "-fno-strict-aliasing")
```

---

### `get`

```nim
proc get*(key: string): string
```

**What it is:** Reads a configuration entry. Returns an empty string if the key does not exist.

```nim
let opts = get("gcc.options.always")
echo "Current GCC options: ", opts
```

---

### `exists`

```nim
proc exists*(key: string): bool
```

**What it is:** Tests whether a configuration key has been set. Useful for conditional logic before reading a value.

```nim
if exists("gcc.options.always"):
  echo "Custom GCC options are set"
```

---

## Project & Path Information

### `projectName`

```nim
proc projectName*(): string
```

**What it is:** Returns the name of the project being compiled (the main `.nim` file's name, without extension or path).

```nim
echo "Building: ", projectName()  # e.g. "myapp"
```

---

### `projectDir`

```nim
proc projectDir*(): string
```

**What it is:** Returns the absolute path to the directory that contains the project's main file.

```nim
let outDir = projectDir() / "build"
mkDir outDir
```

---

### `projectPath`

```nim
proc projectPath*(): string
```

**What it is:** Returns the full absolute path to the project's main `.nim` file, including the filename.

```nim
echo "Main file: ", projectPath()  # e.g. /home/user/myproject/src/main.nim
```

---

### `thisDir`

```nim
proc thisDir*(): string
```

**What it is:** Returns the directory where the currently executing `.nims` script lives. This is the right anchor for constructing paths relative to the script itself.

```nim
# Reference a helper file next to the .nims script
let helper = thisDir() / "helpers" / "codegen.nim"
```

---

### `nimcacheDir`

```nim
proc nimcacheDir*(): string
```

**What it is:** Returns the path to the `nimcache` directory where the compiler stores generated C/C++ files and object files for the current build.

```nim
echo "Cache lives at: ", nimcacheDir()
```

---

### `getCurrentDir`

```nim
proc getCurrentDir*(): string
```

**What it is:** Returns the process's current working directory at the time of the call. Unlike `thisDir()` and `projectDir()`, this reflects the *runtime* working directory, which can be changed with `cd`.

```nim
echo "Working in: ", getCurrentDir()
```

---

### `findExe`

```nim
proc findExe*(bin: string): string
```

**What it is:** Searches for an executable by name — first in the current working directory, then in every directory listed in the `PATH` environment variable. Returns an empty string `""` if not found.

```nim
let git = findExe("git")
if git == "":
  echo "git is not installed or not in PATH"
else:
  exec git & " pull"
```

---

## File System Operations

All file system operations respect the `mode` variable. In `Verbose` mode they print what they are doing; in `Whatif` mode they print the operation but do not execute it.

### `listDirs`

```nim
proc listDirs*(dir: string): seq[string]
```

**What it is:** Returns a list of all immediate subdirectories inside `dir`. The listing is **not recursive** — only the first level of subdirectories is returned.

```nim
for d in listDirs("src"):
  echo "Found subdirectory: ", d
```

---

### `listFiles`

```nim
proc listFiles*(dir: string): seq[string]
```

**What it is:** Returns a list of all files directly inside `dir`. Not recursive.

```nim
for f in listFiles("assets"):
  echo "Asset: ", f
```

---

### `mkDir`

```nim
proc mkDir*(dir: string) {.raises: [OSError].}
```

**What it is:** Creates the directory `dir`, including all intermediate parent directories (like `mkdir -p`). If the directory already exists, no error is raised — this makes it safe to call unconditionally.

```nim
mkDir "build/release/bin"   # creates all three levels if needed
```

---

### `rmDir`

```nim
proc rmDir*(dir: string, checkDir = false) {.raises: [OSError].}
```

**What it is:** Removes a directory and all its contents recursively. When `checkDir` is `true`, raises `OSError` if the directory does not exist; when `false` (default), silently succeeds on a missing directory.

```nim
rmDir "build"          # clean the build directory; no error if missing
rmDir("build", true)   # error if build doesn't exist
```

---

### `rmFile`

```nim
proc rmFile*(file: string) {.raises: [OSError].}
```

**What it is:** Deletes a single file. Raises `OSError` if deletion fails.

```nim
rmFile "build/old_output.o"
```

---

### `cpFile`

```nim
proc cpFile*(`from`, to: string) {.raises: [OSError].}
```

**What it is:** Copies the file at path `from` to path `to`. The destination directory must already exist.

```nim
cpFile("README.md", "dist/README.md")
```

---

### `cpDir`

```nim
proc cpDir*(`from`, to: string) {.raises: [OSError].}
```

**What it is:** Recursively copies the entire directory tree at `from` to `to`.

```nim
cpDir("templates", "build/templates")
```

---

### `mvFile`

```nim
proc mvFile*(`from`, to: string) {.raises: [OSError].}
```

**What it is:** Moves (renames) a file. If `from` and `to` are on different filesystems, the file is copied and then the original is deleted.

```nim
mvFile("app.log", "logs/app_2024.log")
```

---

### `mvDir`

```nim
proc mvDir*(`from`, to: string) {.raises: [OSError].}
```

**What it is:** Moves an entire directory. Useful for renaming or relocating build output.

```nim
mvDir("build/tmp", "build/release")
```

---

### `fileExists`

```nim
proc fileExists*(filename: string): bool {.tags: [ReadIOEffect].}
```

**What it is:** Returns `true` if `filename` exists and is a regular file (not a directory).

```nim
if not fileExists("config.json"):
  echo "Warning: config.json not found, using defaults"
```

---

### `dirExists`

```nim
proc dirExists*(dir: string): bool {.tags: [ReadIOEffect].}
```

**What it is:** Returns `true` if `dir` exists and is a directory.

```nim
if not dirExists("build"):
  mkDir "build"
```

---

## Directory Navigation

### `cd`

```nim
proc cd*(dir: string) {.raises: [OSError].}
```

**What it is:** Permanently changes the current working directory for the remainder of the script's execution. For a temporary change that automatically restores the previous directory, prefer `withDir`.

```nim
cd "src"
exec "nim c main.nim"
cd ".."   # you must manually go back
```

---

### `withDir`

```nim
template withDir*(dir: string; body: untyped): untyped
```

**What it is:** A template that temporarily changes to `dir`, executes `body`, and then **automatically restores** the original directory — even if an exception is raised inside `body`. This is the idiomatic and safe way to run code in a different directory.

```nim
withDir "subproject":
  # current dir is now subproject/
  exec "nim c src/main.nim"
# current dir is restored to whatever it was before
```

> **Prefer `withDir` over `cd`** whenever the directory change is meant to be local to a block of code. It prevents accidentally leaving the script in the wrong directory after an error.

---

## External Process Execution

### `exec` (command only)

```nim
proc exec*(command: string) {.raises: [OSError].}
```

**What it is:** Runs an external shell command. If the command exits with a non-zero code, an `OSError` is raised immediately. The command runs relative to the current working directory.

> If you need the exit code and/or captured output instead of raising on failure, use `system.gorgeEx` directly.

```nim
exec "nim c -r src/tests.nim"
exec "git add ."
exec "git commit -m 'automated release'"
```

---

### `exec` (command + input + cache)

```nim
proc exec*(command: string, input: string, cache = "") {.raises: [OSError].}
```

**What it is:** Like the single-argument `exec`, but additionally feeds `input` to the process's standard input and supports an optional `cache` key for `gorgeEx` caching. Output from the command is echoed to stdout.

> **Warning:** This variant runs relative to the NimScript module path, which may affect relative-path resolution inside the command. Prefer `gorgeEx` directly when you need fine-grained control.

```nim
exec("python3 process.py", "some input data")
```

---

### `selfExec`

```nim
proc selfExec*(command: string) {.raises: [OSError].}
```

**What it is:** Runs a Nim compiler command using the **same `nim` executable** that is currently running this script. This guarantees that you invoke the exact same compiler version, rather than whatever `nim` is first on PATH. The `command` must **not** include `"nim "` at the start.

```nim
# Equivalent to: nim doc src/mylib.nim
selfExec "doc src/mylib.nim"

# Build with a specific backend
selfExec "c -d:release src/main.nim"
```

---

## Environment Variables

### `getEnv`

```nim
proc getEnv*(key: string; default = ""): string
```

**What it is:** Reads the value of environment variable `key`. If the variable is not set, returns `default` (empty string by default).

```nim
let buildType = getEnv("BUILD_TYPE", "debug")
if buildType == "release":
  switch("opt", "speed")
```

---

### `existsEnv`

```nim
proc existsEnv*(key: string): bool
```

**What it is:** Returns `true` if the environment variable `key` is defined (even if its value is an empty string).

```nim
if existsEnv("CI"):
  echo "Running in CI environment"
```

---

### `putEnv`

```nim
proc putEnv*(key, val: string)
```

**What it is:** Sets the environment variable `key` to `val` for the current process and any child processes it spawns.

```nim
putEnv("NIM_CACHE", nimcacheDir())
exec "some_tool_that_reads_NIM_CACHE"
```

---

### `delEnv`

```nim
proc delEnv*(key: string)
```

**What it is:** Removes the environment variable `key` from the process environment.

```nim
delEnv("TMPDIR")   # force tools to use the system default temp dir
```

---

## Standard Input

### `readLineFromStdin`

```nim
proc readLineFromStdin*(): string {.raises: [IOError].}
```

**What it is:** Reads a single line from standard input, blocking until a newline `\n` is received or stdin is closed (EOF). Raises `EOFError` (a subtype of `IOError`) at EOF.

```nim
echo "Enter your name:"
let name = readLineFromStdin()
echo "Hello, ", name
```

---

### `readAllFromStdin`

```nim
proc readAllFromStdin*(): string {.raises: [IOError].}
```

**What it is:** Reads everything from standard input until EOF and returns it as a single string. Useful for processing piped input.

```nim
let data = readAllFromStdin()
echo "Received ", data.len, " bytes"
```

---

## String Utilities

### `cmpic`

```nim
proc cmpic*(a, b: string): int
```

**What it is:** Compares two strings case-insensitively. Returns `0` if equal, a negative number if `a < b`, and a positive number if `a > b`. Useful for portable string comparisons in scripts that may run on both case-sensitive (Linux) and case-insensitive (Windows/macOS) file systems.

```nim
assert cmpic("Hello", "hello") == 0
assert cmpic("abc", "ABC") == 0

if cmpic(getCommand(), "release") == 0:
  switch("opt", "speed")
```

---

### `toExe`

```nim
proc toExe*(filename: string): string
```

**What it is:** On Windows, appends `.exe` to `filename`. On all other platforms, returns `filename` unchanged. Use this whenever you construct a path to an executable in a cross-platform script.

```nim
let binary = "build" / toExe("myapp")
# On Windows: "build\myapp.exe"
# On Linux/macOS: "build/myapp"
exec binary
```

---

### `toDll`

```nim
proc toDll*(filename: string): string
```

**What it is:** Converts a base library name to the platform-appropriate shared library filename. On Windows: `name.dll`. On POSIX systems: `libname.so`.

```nim
echo toDll("mylib")
# Windows: "mylib.dll"
# Linux:   "libmylib.so"
```

---

## Command-Line Parameters

These are useful when a `.nims` file is invoked as a standalone script (e.g., `nim myScript.nims arg1 arg2`).

### `paramCount`

```nim
proc paramCount*(): int
```

**What it is:** Returns the number of command-line arguments passed to the script. Does not count the script name itself.

```nim
echo "Got ", paramCount(), " arguments"
```

---

### `paramStr`

```nim
proc paramStr*(i: int): string
```

**What it is:** Returns the `i`-th command-line argument. Index `0` typically refers to the script name or the first real argument depending on context — check the Nim docs for your specific invocation.

```nim
if paramCount() >= 1:
  let target = paramStr(1)
  echo "Building target: ", target
```

---

## Tasks

### `task`

```nim
template task*(name: untyped; description: string; body: untyped): untyped
```

**What it is:** Defines a named build task that can be invoked as `nim <name>` from the command line. The `description` is shown in the help output (`nim help`). Pass an empty string as `description` to create a hidden task.

For every task named `foo`, a procedure `fooTask()` is generated, allowing you to call one task from another.

```nim
task build, "Compile the project with the C backend":
  setCommand "c"

task clean, "Remove build artifacts":
  rmDir "build"
  rmDir "nimcache"

task release, "Build an optimized release binary":
  switch("opt", "speed")
  switch("define", "release")
  buildTask()   # call the build task from within release
```

Run with:
```sh
nim build myproject.nims
nim clean myproject.nims
nim release myproject.nims
nim help myproject.nims    # lists all tasks with descriptions
```

---

## Nimble Package Metadata

These variables and procedures are used inside `.nimble` files to describe a package. They are only available outside of Nimble's own environment (Nimble provides its own implementations).

### Variables

| Variable | Type | Purpose |
|---|---|---|
| `packageName` | `string` | The package name (defaults to the `.nims` filename) |
| `version` | `string` | Package version string (e.g., `"1.2.3"`) |
| `author` | `string` | Package author's name |
| `description` | `string` | Short human-readable description |
| `license` | `string` | SPDX license identifier (e.g., `"MIT"`) |
| `srcDir` | `string` | Directory containing source files |
| `binDir` | `string` | Directory where compiled binaries go |
| `backend` | `string` | Default compilation backend (`"c"`, `"cpp"`, `"js"`) |
| `bin` | `seq[string]` | List of binary entry points to compile |
| `skipDirs` | `seq[string]` | Directories to exclude from the package |
| `skipFiles` | `seq[string]` | Files to exclude from the package |
| `skipExt` | `seq[string]` | File extensions to exclude |
| `installDirs` | `seq[string]` | Extra directories to include when installing |
| `installFiles` | `seq[string]` | Extra files to include when installing |
| `installExt` | `seq[string]` | Extra extensions to include when installing |
| `requiresData` | `seq[string]` | Raw list of dependency strings (read/write) |

### `requires`

```nim
proc requires*(deps: varargs[string])
```

**What it is:** Declares the package dependencies. Each string follows Nimble's version constraint syntax.

```nim
# In a .nimble file:
packageName = "mylib"
version     = "0.5.0"
author      = "Alice"
description = "A useful library"
license     = "MIT"
srcDir      = "src"

requires "nim >= 1.6.0"
requires "jsony >= 1.1.5", "chronicles >= 0.10.0"
```

---

### `selfExe` *(deprecated)*

```nim
proc selfExe*(): string {.deprecated.}
```

**What it is:** Returns the path to the currently running `nim` or `nimble` executable. **Deprecated since Nim v1.7** — use `getCurrentCompilerExe` instead.

---

*This reference covers all exported symbols from `nimscript.nim` as of Nim's standard library.*
