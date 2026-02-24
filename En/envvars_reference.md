# envvars — Module Reference

## Overview

The `std/envvars` module provides a complete, cross-platform interface for reading, writing, and iterating over **environment variables** — the key/value string pairs that every operating system process inherits from its parent and passes to its children.

The module works uniformly across all Nim backends and targets:

| Target | Implementation |
|---|---|
| Native (Linux, macOS, Windows, etc.) | C standard library (`getenv`, `setenv`, `unsetenv`, `environ`) |
| Windows-specific | Wide-string API (`_wgetenv`, `SetEnvironmentVariableW`) for correct Unicode handling |
| Node.js | `process.env` JavaScript object |
| Nim VM (`nimvm`) | Compiler-internal VM operations |

**Effect system integration.** Two effect types are exported so that the Nim effect system can track environment access in procedure signatures:

- `ReadEnvEffect` — applied to procedures that **read** environment variables. Descends from `ReadIOEffect`.
- `WriteEnvEffect` — applied to procedures that **write or delete** environment variables. Descends from `WriteIOEffect`.

Callers that are themselves annotated with restrictive effect sets will receive a compile-time error if they call a procedure tagged with an effect they do not allow. This is the standard Nim mechanism for controlling IO side effects.

---

## `getEnv`

```nim
proc getEnv*(key: string, default = ""): string {.tags: [ReadEnvEffect].}
```

### Description

Reads the value of the environment variable named `key` and returns it as a string. If no variable with that name exists in the current process environment, the `default` value is returned instead (which is `""` if you do not supply one).

One important subtlety: an environment variable can exist and legitimately hold an **empty string** as its value. In that case `getEnv` returns `""`, which is indistinguishable from the case where the variable does not exist at all. When this distinction matters — for example, when validating required configuration — use `existsEnv` first to confirm presence before calling `getEnv`, or check both together.

### Parameters

| Name | Type | Description |
|---|---|---|
| `key` | `string` | Name of the environment variable to read |
| `default` | `string` | Value to return if the variable is absent. Defaults to `""`. |

### Return value

The string value of the variable, or `default` if it does not exist.

### Examples

**Reading a well-known variable:**

```nim
import std/envvars

let home = getEnv("HOME")
echo "Home directory: ", home
# On Linux/macOS: /home/username   (or whatever the shell set)
# On Windows: "" unless HOME is set; use USERPROFILE instead
```

**Providing a fallback for optional configuration:**

```nim
import std/envvars

let host = getEnv("DB_HOST", "localhost")
let port = getEnv("DB_PORT", "5432")
echo "Connecting to ", host, ":", port
# If DB_HOST is unset → "localhost"
# If DB_HOST=myserver → "myserver"
```

**The empty-string ambiguity and how to handle it:**

```nim
import std/envvars

# Suppose APP_TOKEN is not set at all.
# Both of these return "":
echo getEnv("APP_TOKEN")           # ""
echo getEnv("APP_TOKEN", "")       # ""

# To tell whether the variable actually exists:
if existsEnv("APP_TOKEN"):
  let token = getEnv("APP_TOKEN")
  # token could still be "" if someone did: export APP_TOKEN=
else:
  echo "APP_TOKEN is not set"
```

**Node.js backend — same API, backed by `process.env`:**

```nim
# Compiled with: nim js -d:nodejs myapp.nim
import std/envvars
echo getEnv("PATH")   # reads process.env["PATH"]
```

---

## `existsEnv`

```nim
proc existsEnv*(key: string): bool {.tags: [ReadEnvEffect].}
```

### Description

Checks whether an environment variable named `key` is present in the current process environment. Returns `true` if the variable exists (regardless of its value, including an empty string), and `false` if it does not exist at all.

This is the correct way to test for the *presence* of a variable when its value might legitimately be an empty string. Comparing `getEnv(key) != ""` would falsely report a missing variable and a present-but-empty variable the same way.

### Parameters

| Name | Type | Description |
|---|---|---|
| `key` | `string` | Name of the environment variable to check |

### Return value

`true` if the variable exists, `false` otherwise.

### Examples

**Simple presence check:**

```nim
import std/envvars

if existsEnv("CI"):
  echo "Running in a CI environment"
else:
  echo "Running locally"
```

**Guarding required configuration:**

```nim
import std/envvars

const required = ["DATABASE_URL", "SECRET_KEY", "REDIS_URL"]

for varName in required:
  if not existsEnv(varName):
    raise newException(ValueError, "Required environment variable not set: " & varName)

echo "All required variables present, starting up…"
```

**Distinguishing absent from empty:**

```nim
import std/envvars

putEnv("EMPTY_VAR", "")    # variable exists but holds ""

echo existsEnv("EMPTY_VAR")          # true  — the variable IS there
echo getEnv("EMPTY_VAR") == ""       # true  — but its value is empty
echo existsEnv("NONEXISTENT_VAR")    # false — this one really is absent
```

---

## `putEnv`

```nim
proc putEnv*(key, val: string) {.tags: [WriteEnvEffect].}
```

### Description

Sets the environment variable named `key` to the string `val`, creating it if it does not already exist, or overwriting it if it does. The change is visible immediately to all subsequent calls to `getEnv` and `existsEnv` within the same process. Child processes spawned after the call will inherit the new value.

If the operation fails for any reason, `OSError` is raised with an OS-level error message. On Windows, additional validation is performed: the key must be non-empty and must not contain the `=` character (both of which are illegal in Windows environment variable names).

**Thread-safety note.** Environment variables are a process-wide global resource. Calling `putEnv` from multiple threads concurrently, or calling it while another thread is iterating with `envPairs`, is undefined behaviour on most platforms. Synchronise access externally if needed.

### Parameters

| Name | Type | Description |
|---|---|---|
| `key` | `string` | Name of the environment variable to set |
| `val` | `string` | Value to assign |

### Raises

`OSError` if the underlying OS call fails.

### Examples

**Creating a new variable:**

```nim
import std/envvars

putEnv("MY_APP_MODE", "production")
assert getEnv("MY_APP_MODE") == "production"
```

**Overwriting an existing variable:**

```nim
import std/envvars

putEnv("LOG_LEVEL", "info")
echo getEnv("LOG_LEVEL")   # info

putEnv("LOG_LEVEL", "debug")
echo getEnv("LOG_LEVEL")   # debug
```

**Setting a variable for a child process (conceptually):**

```nim
import std/envvars, std/osproc

putEnv("WORKER_THREADS", "8")
# Any process started after this point inherits WORKER_THREADS=8
let p = startProcess("myworker")
```

**Error handling on Windows — invalid key:**

```nim
import std/envvars

try:
  putEnv("", "value")        # empty key → OSError on Windows
except OSError as e:
  echo "Cannot set variable: ", e.msg

try:
  putEnv("KEY=NAME", "v")    # '=' in key → OSError on Windows
except OSError as e:
  echo "Cannot set variable: ", e.msg
```

---

## `delEnv`

```nim
proc delEnv*(key: string) {.tags: [WriteEnvEffect].}
```

### Description

Removes the environment variable named `key` from the current process environment. After this call, `existsEnv(key)` returns `false` and `getEnv(key)` returns `""` (or whatever default the caller supplies). Child processes spawned after the call will not inherit the deleted variable.

If the operation fails, `OSError` is raised. On Windows, the key must be non-empty and must not contain `=`; violating either constraint raises `OSError` immediately, before any OS call is made.

Deleting a variable that does not exist is not an error on POSIX systems (the underlying `unsetenv` call is a no-op). On Windows, however, attempting to delete a non-existent variable via the `_putenv` path may produce an error depending on the runtime version — treat delete-of-absent as a conditionally safe operation and wrap in `existsEnv` if portability matters.

### Parameters

| Name | Type | Description |
|---|---|---|
| `key` | `string` | Name of the environment variable to delete |

### Raises

`OSError` if the underlying OS call fails.

### Examples

**Deleting a variable:**

```nim
import std/envvars

putEnv("TEMP_FLAG", "1")
assert existsEnv("TEMP_FLAG")

delEnv("TEMP_FLAG")
assert not existsEnv("TEMP_FLAG")
assert getEnv("TEMP_FLAG") == ""
```

**Safe delete — check first for maximum portability:**

```nim
import std/envvars

if existsEnv("SESSION_TOKEN"):
  delEnv("SESSION_TOKEN")
```

**Cleaning up test fixtures:**

```nim
import std/envvars

proc withEnv(key, val: string, body: proc()) =
  let hadBefore = existsEnv(key)
  let oldVal = getEnv(key)
  putEnv(key, val)
  try:
    body()
  finally:
    if hadBefore:
      putEnv(key, oldVal)   # restore original
    else:
      delEnv(key)           # remove what we added

withEnv("FEATURE_FLAG", "1"):
  doTest()
```

---

## `envPairs` (iterator)

```nim
iterator envPairs*(): tuple[key, value: string] {.tags: [ReadEnvEffect].}
```

### Description

Iterates over **all environment variables** in the current process environment, yielding each one as a `(key, value)` tuple of strings. The iteration order is not guaranteed and depends on the operating system and the order in which variables were set. Every variable present at the moment the iterator is called is visited exactly once.

`envPairs` works across all supported targets: native binaries, Node.js (`process.env`), and the Nim VM. On native targets the implementation reads directly from the OS environment block — on POSIX systems this is the `environ` global array; on Windows it uses the wide-string `GetEnvironmentStringsW` API.

**Snapshot semantics.** Modifying the environment (via `putEnv` or `delEnv`) while iterating is undefined behaviour. The iterator reflects the environment state at the point it begins; take a snapshot into a `seq` if you need to modify while iterating.

### Yields

`tuple[key, value: string]` — the name and value of each environment variable.

### Examples

**Printing all environment variables:**

```nim
import std/envvars

for key, value in envPairs():
  echo key, "=", value
# PATH=/usr/local/bin:/usr/bin:/bin
# HOME=/home/alice
# TERM=xterm-256color
# … (all variables in the environment)
```

**Finding variables with a given prefix:**

```nim
import std/envvars

for key, value in envPairs():
  if key.startsWith("APP_"):
    echo key, " → ", value
# APP_MODE → production
# APP_PORT → 8080
```

**Collecting into a table for random-access lookup:**

```nim
import std/envvars, std/tables

var env = initTable[string, string]()
for key, value in envPairs():
  env[key] = value

echo env.getOrDefault("HOME", "(not set)")
```

**Taking a safe snapshot before modifying the environment:**

```nim
import std/envvars, std/sequtils

# Collect first, then modify — avoids undefined behaviour
let snapshot = toSeq(envPairs())

for (key, _) in snapshot:
  if key.startsWith("LEGACY_"):
    delEnv(key)
```

**Counting total variables and total value bytes:**

```nim
import std/envvars

var count = 0
var totalBytes = 0
for key, value in envPairs():
  inc count
  inc totalBytes, key.len + value.len + 1  # +1 for the '=' separator

echo "Total variables: ", count
echo "Total data bytes: ", totalBytes
```

---

## Effect types

### `ReadEnvEffect`

```nim
type ReadEnvEffect* = object of ReadIOEffect
```

A compile-time effect marker. It is automatically applied (via `{.tags: [ReadEnvEffect].}`) to `getEnv`, `existsEnv`, and `envPairs`. Any procedure that calls one of these will also acquire `ReadEnvEffect` through effect propagation unless it is explicitly suppressed.

Use this type in your own procedure annotations when you want to communicate that a procedure reads from the environment, or when you want to restrict callers via `.forbids`:

```nim
import std/envvars

proc configValue(key: string): string {.tags: [ReadEnvEffect].} =
  getEnv(key, "default")
```

### `WriteEnvEffect`

```nim
type WriteEnvEffect* = object of WriteIOEffect
```

A compile-time effect marker applied to `putEnv` and `delEnv`. Propagates to any procedure that calls either of them.

```nim
import std/envvars

proc applyConfig() {.tags: [WriteEnvEffect].} =
  putEnv("LOG_LEVEL", "warn")
  putEnv("MAX_CONN", "100")
```

---

## Summary table

| Symbol | Kind | Effect tag | Description |
|---|---|---|---|
| `getEnv(key, default)` | proc | `ReadEnvEffect` | Read a variable; return default if absent |
| `existsEnv(key)` | proc | `ReadEnvEffect` | Check whether a variable exists |
| `putEnv(key, val)` | proc | `WriteEnvEffect` | Set (create or overwrite) a variable |
| `delEnv(key)` | proc | `WriteEnvEffect` | Delete a variable |
| `envPairs()` | iterator | `ReadEnvEffect` | Iterate over all variables as `(key, value)` tuples |
| `ReadEnvEffect` | type | — | Effect tag for environment reads |
| `WriteEnvEffect` | type | — | Effect tag for environment writes |
