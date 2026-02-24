# `compilesettings` Module Reference

> **Module purpose:** `compilesettings` lets you interrogate the Nim compiler about its own configuration **at compile time**. While your program is being compiled, you can ask "where is the output directory?", "which backend is being used?", "what search paths are active?" — and bake the answers directly into constants, static assertions, or conditional compilation logic. All queries happen exclusively during compilation; there is no runtime overhead whatsoever.

---

## Table of Contents

1. [Type: SingleValueSetting](#type-singlevaluesetting)
2. [Type: MultipleValueSetting](#type-multiplevaluesetting)
3. [querySetting](#querysetting)
4. [querySettingSeq](#querysettingseq)
5. [SingleValueSetting field reference](#singlevaluesetting-field-reference)
6. [MultipleValueSetting field reference](#multiplevaluesetting-field-reference)
7. [Practical patterns](#practical-patterns)

---

## Type: `SingleValueSetting`

```nim
type SingleValueSetting* {.pure.} = enum
  arguments, outFile, outDir, nimcacheDir, projectName, projectPath,
  projectFull, command, commandLine, linkOptions, compileOptions,
  ccompilerPath, backend, libPath, gc, mm
```

### What it is

An enumeration that names every compiler setting which resolves to **a single string**. Each member corresponds to one piece of information the compiler holds internally — a directory path, a flag list, a backend name, etc. You pass one of these members to `querySetting` to retrieve the corresponding value.

The enum is marked `{.pure.}`, so members must be qualified: `SingleValueSetting.outDir`, not just `outDir`.

> **Compatibility note:** New values are only ever appended at the end of this enum. This preserves binary compatibility across Nim compiler versions — a binary compiled with an older Nim can still read the enum without misinterpreting existing members.

---

## Type: `MultipleValueSetting`

```nim
type MultipleValueSetting* {.pure.} = enum
  nimblePaths, searchPaths, lazyPaths, commandArgs, cincludes, clibs
```

### What it is

An enumeration that names every compiler setting which resolves to **a sequence of strings** — typically because the underlying setting naturally holds multiple values (several search paths, several linked libraries, etc.). You pass one of these members to `querySettingSeq` to retrieve the corresponding `seq[string]`.

Like `SingleValueSetting`, this enum is `{.pure.}` and only grows at the end.

---

## `querySetting`

```nim
proc querySetting*(setting: SingleValueSetting): string {.compileTime, noSideEffect.}
```

### What it does

Returns the current value of a single-valued compiler setting as a `string`. The call is resolved entirely by the compiler's virtual machine (VM) during compilation — the Nim documentation describes this as "implemented in the vmops", meaning the compiler intercepts the call and returns the answer directly without executing any Nim code.

Because the proc is `compileTime`, it cannot be called at runtime. It is only meaningful inside `const` declarations, `static` blocks, `when` branches, macros, and other compile-time contexts.

The `noSideEffect` pragma confirms that querying a setting has no observable effect on program state — it is a pure read.

### The `compileTime` constraint in practice

If you try to call `querySetting` from an ordinary runtime proc, the compiler will reject it. The value must always be captured into a `const`:

```nim
import std/compilesettings

# ✅ Correct — evaluated at compile time
const outputDir = querySetting(SingleValueSetting.outDir)
const backend   = querySetting(SingleValueSetting.backend)

# ❌ Wrong — cannot call a compileTime proc at runtime
proc show() =
  echo querySetting(SingleValueSetting.outDir)  # compile error
```

### Examples

```nim
import std/compilesettings

# 1. Embed the nimcache path into the binary for diagnostics
const nimcache = querySetting(SingleValueSetting.nimcacheDir)
echo "Compilation cache was at: ", nimcache

# 2. Assert that a specific backend was used
const be = querySetting(SingleValueSetting.backend)
static:
  assert be == "c", "This library only supports the C backend, got: " & be

# 3. Embed the full project path — useful in generated code or error messages
const projectFull = querySetting(SingleValueSetting.projectFull)
echo "Compiled from: ", projectFull

# 4. Expose the C compiler path for debugging build issues
const cc = querySetting(SingleValueSetting.ccompilerPath)
echo "C compiler used: ", cc

# 5. Conditional compilation based on memory manager
const memMgr = querySetting(SingleValueSetting.mm)
when memMgr == "orc":
  echo "Running with ORC memory management"
else:
  echo "Running with: ", memMgr
```

---

## `querySettingSeq`

```nim
proc querySettingSeq*(setting: MultipleValueSetting): seq[string] {.compileTime, noSideEffect.}
```

### What it does

Returns the current value of a multi-valued compiler setting as a `seq[string]`. Like `querySetting`, the call is intercepted by the compiler VM and has no runtime cost. All the same constraints apply: it is `compileTime`-only and must be used in a `const`, `static`, macro, or similar compile-time context.

The result is an ordinary Nim `seq[string]` at compile time, so you can iterate it, check its length, index into it, or pass it to other compile-time procedures.

### Examples

```nim
import std/compilesettings

# 1. Capture and embed the active search paths
const paths = querySettingSeq(MultipleValueSetting.searchPaths)
static:
  echo "Search paths at compile time:"
  for p in paths:
    echo "  ", p

# 2. Verify a required Nimble path is present
const nimblePaths = querySettingSeq(MultipleValueSetting.nimblePaths)
static:
  var found = false
  for p in nimblePaths:
    if "mypackage" in p:
      found = true
  assert found, "mypackage nimble path not found — is it installed?"

# 3. Inspect C include paths for build diagnostics
const cincludes = querySettingSeq(MultipleValueSetting.cincludes)
const clibs     = querySettingSeq(MultipleValueSetting.clibs)
static:
  echo "C includes (", cincludes.len, "):"
  for p in cincludes: echo "  -I", p
  echo "C libs (", clibs.len, "):"
  for l in clibs: echo "  -l", l

# 4. Embed the count of Nimble paths as a runtime constant
const nimblePathCount = querySettingSeq(MultipleValueSetting.nimblePaths).len
echo "Nimble paths active during compilation: ", nimblePathCount
```

---

## `SingleValueSetting` field reference

A detailed description of every member of the enum.

---

### `arguments`
⚠️ *Experimental*

The arguments passed to the program after the `-r` flag when invoking the compiler as `nim r myfile.nim arg1 arg2`. Rarely needed in library code; useful in meta-build tooling.

---

### `outFile`
⚠️ *Experimental*

The name of the output file being produced (e.g. `myapp` or `myapp.exe`). Does not include the directory — combine with `outDir` for a full path.

---

### `outDir`

The directory where the compiled output file will be written. Controlled by the `--outDir` compiler switch. Useful when your build generates additional files (assets, manifests, configuration) that should land next to the binary.

```nim
const outputDir = querySetting(SingleValueSetting.outDir)
# Use outputDir to write a companion file next to the binary at build time
```

---

### `nimcacheDir`

The path to the `nimcache` directory — the folder where Nim stores intermediate C/C++ source files, object files, and other artefacts of the compilation process. Useful for diagnostics and for custom build steps that need to inspect or clean generated C code.

```nim
const nimcache = querySetting(SingleValueSetting.nimcacheDir)
static: echo "Intermediate C files are in: ", nimcache
```

---

### `projectName`

The name of the project (source file) being compiled, without extension or directory. For a file `src/myapp.nim`, this would be `"myapp"`. Handy for embedding version strings, module names, or generating file names.

```nim
const name = querySetting(SingleValueSetting.projectName)
const versionString = name & " v1.0"
```

---

### `projectPath`
⚠️ *Experimental*

Some path associated with the project being compiled. The exact value is considered experimental and may differ from `projectFull`. Prefer `projectFull` for stable behaviour.

---

### `projectFull`

The **full absolute path** to the main source file being compiled. This is the most reliable way to know where the source lives, useful for embedding source references in generated code or error messages.

```nim
const src = querySetting(SingleValueSetting.projectFull)
static: echo "Building: ", src
```

---

### `command`
⚠️ *Experimental*

The Nim compiler subcommand being executed: `"c"`, `"cpp"`, `"doc"`, `"js"`, etc. Allows compile-time branching based on what the compiler was asked to do.

```nim
const cmd = querySetting(SingleValueSetting.command)
when cmd == "doc":
  # include extra documentation-only declarations
  discard
```

---

### `commandLine`
⚠️ *Experimental*

The full command line string that was passed to the Nim compiler. Rarely needed in application code; most useful in advanced meta-programming and build-system integration.

---

### `linkOptions`

Additional options that will be passed to the linker. These come from `--passL` flags or `.passl` pragmas. Useful for verifying that expected linker flags are active.

```nim
const lo = querySetting(SingleValueSetting.linkOptions)
static: echo "Linker flags: ", lo
```

---

### `compileOptions`

Additional options that will be passed to the C/C++ compiler. These come from `--passC` flags or `.passc` pragmas. Useful for diagnosing unexpected compiler flags.

```nim
const co = querySetting(SingleValueSetting.compileOptions)
static: echo "C compiler flags: ", co
```

---

### `ccompilerPath`

The filesystem path to the C or C++ compiler executable that Nim will invoke. Useful when your build needs to know exactly which toolchain is active (e.g. to call it directly for a custom step).

```nim
const cc = querySetting(SingleValueSetting.ccompilerPath)
static: echo "Toolchain: ", cc
```

---

### `backend`

The compilation backend in use. Common values are `"c"`, `"cpp"`, `"objc"`, and `"js"`. Both `nim doc --backend:js` and `nim js` result in `backend = "js"`. This is the canonical way to branch at compile time based on the target environment.

```nim
const be = querySetting(SingleValueSetting.backend)
when be == "js":
  # JavaScript-specific imports and logic
  import std/jsffi
```

---

### `libPath`

The absolute path to the Nim standard library (`--lib`). Available since Nim 1.5.1. Useful when build tooling needs to locate stdlib source files directly.

```nim
const stdlib = querySetting(SingleValueSetting.libPath)
static: echo "Standard library at: ", stdlib
```

---

### `gc`
⛔ *Deprecated* — use `mm` instead.

---

### `mm`

The memory management strategy selected for this build. Common values include `"orc"`, `"arc"`, `"refc"`, `"boehm"`, `"none"`. Use this to write libraries that adapt their behaviour or raise a meaningful error at compile time if they require a specific memory manager.

```nim
const memMgr = querySetting(SingleValueSetting.mm)
static:
  assert memMgr in ["orc", "arc"],
    "This library requires ARC or ORC, but got: " & memMgr
```

---

## `MultipleValueSetting` field reference

---

### `nimblePaths`

The list of directories that Nimble has registered as package roots. Each entry is a path under which Nimble packages are installed. Useful for verifying that expected dependencies are reachable.

---

### `searchPaths`

The full list of directories in which the compiler will look for imported modules (`--path` / `-p` flags and defaults). Inspecting this at compile time lets you verify the module resolution environment.

---

### `lazyPaths`
⚠️ *Experimental*

Additional paths that are searched lazily (on demand). The semantics are considered experimental.

---

### `commandArgs`

The list of arguments passed to the Nim compiler invocation itself (not to the program via `-r`). Useful in macros that need to react to specific compiler flags.

---

### `cincludes`

The list of `#include` search directories passed to the C/C++ compiler (`--cincludes` / `-p` for C). Each entry corresponds to a `-I<path>` flag. Useful when your Nim code wraps C libraries and you need to verify that the right headers are on the include path.

---

### `clibs`

The list of libraries that will be linked into the final binary via the C linker. Each entry typically corresponds to a `-l<name>` flag. Useful for diagnosing missing or unexpected link-time dependencies.

---

## Practical patterns

### Pattern 1 — Backend-specific module imports

```nim
import std/compilesettings

const be = querySetting(SingleValueSetting.backend)

when be == "js":
  import std/jsffi
  proc alert(msg: cstring) {.importjs: "alert(#)".}
else:
  proc alert(msg: string) = echo msg

alert("Hello from " & be & " backend!")
```

### Pattern 2 — Enforcing build requirements at compile time

```nim
import std/compilesettings

# Enforce memory manager
const mm = querySetting(SingleValueSetting.mm)
static:
  assert mm == "orc",
    "MyLib requires --mm:orc. Currently active: " & mm

# Enforce that a required Nimble package path is present
const np = querySettingSeq(MultipleValueSetting.nimblePaths)
static:
  assert np.len > 0, "No Nimble paths found — are packages installed?"
```

### Pattern 3 — Embedding build metadata into the binary

```nim
import std/compilesettings

const BuildInfo = (
  project:  querySetting(SingleValueSetting.projectName),
  backend:  querySetting(SingleValueSetting.backend),
  mm:       querySetting(SingleValueSetting.mm),
  outDir:   querySetting(SingleValueSetting.outDir),
)

proc printBuildInfo*() =
  echo "Project : ", BuildInfo.project
  echo "Backend : ", BuildInfo.backend
  echo "Memory  : ", BuildInfo.mm
  echo "OutDir  : ", BuildInfo.outDir
```

### Pattern 4 — Diagnostic dump at compile time

```nim
import std/compilesettings

static:
  echo "=== Compile-time diagnostics ==="
  echo "Project  : ", querySetting(SingleValueSetting.projectFull)
  echo "Backend  : ", querySetting(SingleValueSetting.backend)
  echo "MM       : ", querySetting(SingleValueSetting.mm)
  echo "Nimcache : ", querySetting(SingleValueSetting.nimcacheDir)
  let sp = querySettingSeq(MultipleValueSetting.searchPaths)
  echo "Search paths (", sp.len, "):"
  for p in sp: echo "  ", p
```
