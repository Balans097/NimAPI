# logging.nim — Module Reference

> **Import:** `import std/logging`  
> **JavaScript backend:** `ConsoleLogger` only; file-based loggers and `defaultFilename` are not available.

This module provides a lightweight, extensible logging system. Its design revolves around three concepts that work together:

1. **Levels** — a severity scale that controls which messages matter.
2. **Loggers** — objects that know *where* to write (console, file, rolling file) and *at what threshold* they care.
3. **Handlers** — a thread-local registry that lets a single `log` call fan out to every logger at once.

---

## Architecture overview

```
Your code
   │
   ├── logger.log(lvlInfo, "msg")   ← direct call to one specific logger
   │
   └── log(lvlInfo, "msg")          ← template: fans out to ALL registered handlers
           │
           ├── ConsoleLogger  (stdout / stderr)
           ├── FileLogger     (plain file)
           └── RollingFileLogger (rotating files)
```

A message reaches a logger only if its level passes **two independent gates**:
- The **global log filter** (`setLogFilter`) — a thread-local floor that applies to every handler.
- The **logger's own `levelThreshold`** — set per-logger at creation time.

---

## Types

### `Level`

```nim
type Level* = enum
  lvlAll, lvlDebug, lvlInfo, lvlNotice, lvlWarn, lvlError, lvlFatal, lvlNone
```

The severity ladder, from lowest (`lvlAll`) to highest (`lvlNone`). Each value has a conventional meaning:

| Value | Meaning |
|---|---|
| `lvlAll` | Special: enables every message. Use as a `levelThreshold` to accept everything. |
| `lvlDebug` | Developer-only detail, usually stripped in production. |
| `lvlInfo` | Routine operational events — normal, unremarkable. |
| `lvlNotice` | Normal but significant events the operator should be aware of. |
| `lvlWarn` | Something unexpected happened; the app still works but warrants attention. |
| `lvlError` | A recoverable failure — the app continues but with degraded behaviour. |
| `lvlFatal` | An unrecoverable failure. The app is about to exit. Logging `fatal` does **not** quit — that remains the programmer's responsibility. |
| `lvlNone` | Special: suppresses everything. Use as a `levelThreshold` to silence a logger completely. |

---

### `Logger`

```nim
type Logger* = ref object of RootObj
  levelThreshold*: Level
  fmtStr*: string
```

The abstract base class for all loggers. To create a custom logger, inherit from this type and override the `log` method. Two fields are always present and publicly writable:

- **`levelThreshold`** — messages below this level are silently discarded by this logger.
- **`fmtStr`** — the format string prepended to every message (see [Format strings](#format-strings)).

---

### `ConsoleLogger`

```nim
type ConsoleLogger* = ref object of Logger
  useStderr*: bool
  flushThreshold*: Level
```

Writes messages to the terminal. On the JS backend, uses the browser's `console.*` methods. Two extra fields:

- **`useStderr`** — when `true`, writes to `stderr` instead of `stdout`.
- **`flushThreshold`** — messages at or above this level cause the output buffer to be flushed immediately. By default only `lvlError` and above trigger a flush.

---

### `FileLogger` *(not available on JS)*

```nim
type FileLogger* = ref object of Logger
  file*: File
  flushThreshold*: Level
```

Writes messages to a file. The `file` field is the underlying `File` handle and can be inspected or replaced directly if needed.

---

### `RollingFileLogger` *(not available on JS)*

```nim
type RollingFileLogger* = ref object of FileLogger
  # (rotation state is internal)
```

Extends `FileLogger` with automatic log rotation: once the current file exceeds `maxLines` lines, it is renamed to `<name>.1` (previous `.1` becomes `.2`, and so on) and a fresh file is opened. This prevents log files from growing unboundedly.

---

## Constants

### `LevelNames`

```nim
const LevelNames*: array[Level, string] =
  ["DEBUG", "DEBUG", "INFO", "NOTICE", "WARN", "ERROR", "FATAL", "NONE"]
```

A lookup table from `Level` to its display name. `lvlAll` maps to `"DEBUG"` — when all levels are active, the effective display name is `DEBUG`. Used internally by format string substitution and available for custom loggers.

---

### `defaultFmtStr`

```nim
const defaultFmtStr* = "$levelname "
```

The format string used when none is specified. Produces output like `INFO some message`. The trailing space separates the prefix from the message body.

---

### `verboseFmtStr`

```nim
const verboseFmtStr* = "$levelid, [$datetime] -- $appname: "
```

A richer preset that includes the single-letter level identifier, a full timestamp, and the application name. Produces output like `I, [2024-01-15T10:32:04] -- myapp: some message`. Pass it as `fmtStr` when creating any logger.

---

## Format strings

Format strings are text templates where `$variable` placeholders are replaced with live values before the user message is appended. Any characters outside a placeholder are copied verbatim.

| Placeholder | Replaced with |
|---|---|
| `$date` | Current date (`YYYY-MM-DD`) |
| `$time` | Current time (`HH:MM:SS`) |
| `$datetime` | `$date` + `T` + `$time` |
| `$app` | Full path of the executable (`os.getAppFilename()`) |
| `$appname` | Base name of the executable (without extension or directory) |
| `$appdir` | Directory containing the executable |
| `$levelid` | First character of the level name (`D`, `I`, `N`, `W`, `E`, `F`) |
| `$levelname` | Full level name (`DEBUG`, `INFO`, `NOTICE`, `WARN`, `ERROR`, `FATAL`) |

`$app`, `$appname`, and `$appdir` are not available on the JavaScript backend.

```nim
# Custom prefix: [10:32:04] - INFO: 
var logger = newConsoleLogger(fmtStr = "[$time] - $levelname: ")
logger.log(lvlInfo, "server started on port 8080")
# Output: [10:32:04] - INFO: server started on port 8080
```

---

## Creating loggers

### `newConsoleLogger`

```nim
proc newConsoleLogger*(
    levelThreshold = lvlAll,
    fmtStr = defaultFmtStr,
    useStderr = false,
    flushThreshold = defaultFlushThreshold
): ConsoleLogger
```

**What it does.** Creates a `ConsoleLogger`. By default accepts all messages, uses the default format string, writes to `stdout`, and only flushes on `lvlError` and above.

The most common starting point for any Nim program that needs logging. In production you would typically raise `levelThreshold` to `lvlInfo` or `lvlWarn`.

```nim
# Minimal — log everything to stdout
var log = newConsoleLogger()

# Only warnings and above, sent to stderr, always flushed
var errLog = newConsoleLogger(
    levelThreshold = lvlWarn,
    useStderr = true,
    flushThreshold = lvlAll
)

# With verbose timestamps
var verboseLog = newConsoleLogger(fmtStr = verboseFmtStr)
```

---

### `newFileLogger` (from file handle) *(not JS)*

```nim
proc newFileLogger*(
    file: File,
    levelThreshold = lvlAll,
    fmtStr = defaultFmtStr,
    flushThreshold = defaultFlushThreshold
): FileLogger
```

**What it does.** Creates a `FileLogger` wrapping an already-open `File` handle. Use this overload when you want full control over how the file was opened — for example, to share a file handle with other parts of the program, or to use a special mode.

```nim
let f = open("app.log", fmWrite)
var logger = newFileLogger(f, levelThreshold = lvlInfo)
```

---

### `newFileLogger` (from filename) *(not JS)*

```nim
proc newFileLogger*(
    filename = defaultFilename(),
    mode: FileMode = fmAppend,
    levelThreshold = lvlAll,
    fmtStr = defaultFmtStr,
    bufSize: int = -1,
    flushThreshold = defaultFlushThreshold
): FileLogger
```

**What it does.** The more convenient overload — opens the file internally. By default the file is opened for appending, so restarting the application does not wipe previous log entries.

`bufSize` controls the I/O buffer:
- `-1` — use the OS/runtime default (usually 4–8 KB).
- `0` — unbuffered: every write goes straight to the OS. Safe, but can be slow.
- `> 0` — fixed buffer size in bytes.

```nim
# Append to "app.log", only log errors
var errLog = newFileLogger("app.log", levelThreshold = lvlError)

# Overwrite on each run, verbose format, unbuffered
var devLog = newFileLogger("dev.log", mode = fmWrite,
                           fmtStr = verboseFmtStr, bufSize = 0)
```

---

### `newRollingFileLogger` *(not JS)*

```nim
proc newRollingFileLogger*(
    filename = defaultFilename(),
    mode: FileMode = fmReadWrite,
    levelThreshold = lvlAll,
    fmtStr = defaultFmtStr,
    maxLines: Positive = 1000,
    bufSize: int = -1,
    flushThreshold = defaultFlushThreshold
): RollingFileLogger
```

**What it does.** Creates a `RollingFileLogger`. Once the active log file accumulates `maxLines` lines, it is rotated: `app.log` → `app.log.1`, the previous `app.log.1` → `app.log.2`, and so on. A new empty `app.log` is then created. Old rotated files are never automatically deleted.

This is the right choice for long-running services where disk space is a concern and you want to keep a bounded number of lines per file without manual log management.

```nim
# Default: rotate every 1000 lines
var logger = newRollingFileLogger("service.log")

# Compact rotation every 200 lines, errors only
var errorLog = newRollingFileLogger(
    "errors.log",
    levelThreshold = lvlError,
    maxLines = 200
)
```

---

### `defaultFilename` *(not JS)*

```nim
proc defaultFilename*(): string
```

**What it does.** Returns the default log file path used by `newFileLogger` and `newRollingFileLogger` when no `filename` argument is given. It is constructed as the path of the running executable with the extension replaced by `.log`. For example, if the executable is `/usr/bin/myapp`, this returns `/usr/bin/myapp.log`.

```nim
echo defaultFilename()   # e.g. /home/user/project/myapp.log
```

---

## Logging methods

Every logger type has a `log` method. Calling it directly targets **only that one logger** — the registered handler list is not consulted. This matters when you want to funnel a specific message to a specific destination without broadcasting it.

### `log` (ConsoleLogger)

```nim
method log*(logger: ConsoleLogger, level: Level, args: varargs[string, `$`])
```

**What it does.** Formats and writes a message to the console, provided `level` passes both the global filter and the logger's own `levelThreshold`. Multiple `args` are concatenated. Any value whose type has a `$` proc can be passed — the conversion happens automatically.

On the JS backend, the appropriate `console.*` method is selected per level (`console.debug`, `console.warn`, etc.).

```nim
var log = newConsoleLogger()
log.log(lvlInfo,  "Server started on port ", 8080)
log.log(lvlError, "Failed to open file: ", filename)
```

---

### `log` (FileLogger) *(not JS)*

```nim
method log*(logger: FileLogger, level: Level, args: varargs[string, `$`])
```

**What it does.** Same semantics as the ConsoleLogger version, but writes to the wrapped file. Flushes immediately if `level >= logger.flushThreshold`.

```nim
var log = newFileLogger("app.log")
log.log(lvlWarn, "Disk usage above 80%")
```

---

### `log` (RollingFileLogger) *(not JS)*

```nim
method log*(logger: RollingFileLogger, level: Level, args: varargs[string, `$`])
```

**What it does.** Same as `FileLogger.log`, but also checks whether rotation is needed before writing. If the current file has reached `maxLines`, rotation is performed transparently and then the message is written to the fresh file.

```nim
var log = newRollingFileLogger("rolling.log", maxLines = 500)
log.log(lvlInfo, "tick ", tickCount)   # rotates silently when needed
```

---

### `log` (base Logger)

```nim
method log*(logger: Logger, level: Level, args: varargs[string, `$`])
```

**What it does.** The base-class implementation — does nothing. Exists so that custom loggers can inherit from `Logger` and only override this method to add their own behaviour. You would never call this on a plain `Logger` instance directly.

```nim
type SyslogLogger = ref object of Logger

method log*(logger: SyslogLogger, level: Level, args: varargs[string, `$`]) =
  if level >= logger.levelThreshold:
    sendToSyslog(substituteLog(logger.fmtStr, level, args))
```

---

## Handler registry

The handler system is the multi-logger broadcasting mechanism. Registering a logger with `addHandler` means that every subsequent call to the top-level `log` template (and the level-specific shortcuts) will automatically dispatch to it.

> **Thread safety note:** both the handler list and the global log filter are **thread-local variables**. If you need logging in multiple threads, call `addHandler` and `setLogFilter` separately inside each thread.

### `addHandler`

```nim
proc addHandler*(handler: Logger)
```

**What it does.** Appends `handler` to the current thread's list of registered loggers. After this call, every `log(...)` template invocation in this thread will dispatch to `handler` (subject to level filtering).

If the same logger is added multiple times, it will receive the same message multiple times — there is no deduplication.

```nim
var consoleLog = newConsoleLogger()
var fileLog    = newFileLogger("app.log", levelThreshold = lvlWarn)

addHandler(consoleLog)   # see everything in the terminal
addHandler(fileLog)      # write warnings+ to disk
```

---

### `removeHandler`

```nim
proc removeHandler*(handler: Logger)
```

**What it does.** Removes the first occurrence of `handler` from the registered handler list. If the logger was added multiple times, call this proc once per registration to fully remove it.

```nim
addHandler(debugLog)
# ... do some debug-heavy work ...
removeHandler(debugLog)   # silence the debug logger for the rest of the run
```

---

### `getHandlers`

```nim
proc getHandlers*(): seq[Logger]
```

**What it does.** Returns a snapshot of the currently registered handler list for the calling thread. Useful for inspection, testing, or conditional registration.

```nim
if debugLog notin getHandlers():
    addHandler(debugLog)
```

---

## Global filter

### `setLogFilter`

```nim
proc setLogFilter*(lvl: Level)
```

**What it does.** Sets the global minimum level for the current thread. Any message whose level is below `lvl` will be silently dropped before it even reaches a logger's own `levelThreshold` check. The default is `lvlAll`, meaning nothing is filtered globally.

This is the quickest way to silence entire categories of output at runtime without modifying individual loggers — for example, switching from verbose debug output to warnings-only before entering a performance-critical section.

```nim
setLogFilter(lvlWarn)   # from now on, debug and info are globally suppressed
warn("this will appear")
info("this will NOT appear")
```

---

### `getLogFilter`

```nim
proc getLogFilter*(): Level
```

**What it does.** Returns the current global log filter level for the calling thread.

```nim
echo "Current filter: ", getLogFilter()   # e.g. WARN
```

---

## Logging templates

These templates broadcast to **all registered handlers** at once (unlike calling `log` on a specific logger). They check the global filter first, and each handler then applies its own `levelThreshold`.

### `log` (template)

```nim
template log*(level: Level, args: varargs[string, `$`])
```

**What it does.** The master broadcast template. Fans the message out to every registered handler. All level-specific templates below delegate to this one.

```nim
log(lvlInfo, "Worker started, thread id: ", threadId)
```

---

### `debug`, `info`, `notice`, `warn`, `error`, `fatal`

```nim
template debug* (args: varargs[string, `$`])
template info*  (args: varargs[string, `$`])
template notice*(args: varargs[string, `$`])
template warn*  (args: varargs[string, `$`])
template error* (args: varargs[string, `$`])
template fatal* (args: varargs[string, `$`])
```

**What they do.** Convenience wrappers around `log` with a fixed level. Calling `error("msg")` is exactly equivalent to `log(lvlError, "msg")`. Using these instead of the generic `log` template makes call sites more readable and makes it easy to grep for specific severity categories.

`fatal` deserves a special note: it logs the message, but it does **not** terminate the program. After calling `fatal`, add your own `quit` or `raise` as appropriate for your application.

```nim
debug("cache miss for key: ", key)
info("user ", userId, " logged in")
notice("config file reloaded")
warn("response time exceeded threshold: ", elapsed, "ms")
error("database query failed: ", errMsg)
fatal("out of memory — cannot continue")
quit(1)
```

---

## Low-level utility

### `substituteLog`

```nim
proc substituteLog*(frmt: string, level: Level,
                    args: varargs[string, `$`]): string
```

**What it does.** The formatting engine used internally by all loggers. Takes a format string, expands all `$variable` placeholders with their current values for the given `level`, then appends the concatenated `args`. Returns the final string ready to be written.

You will rarely call this directly unless you are building a custom logger. It is documented here because understanding it demystifies how format strings and message construction work.

```nim
echo substituteLog("[$levelname] ", lvlWarn, "disk full")
# Output: [WARN] disk full

echo substituteLog("$levelid|", lvlError, "code=", 500)
# Output: E|code=500
```

---

## Complete examples

### Minimal console logging

```nim
import std/logging

var log = newConsoleLogger(levelThreshold = lvlInfo)
addHandler(log)

info("Application started")
warn("Config file not found, using defaults")
error("Failed to connect to database")
```

### Multiple handlers with different thresholds

```nim
import std/logging

# Everything visible in the terminal during development
addHandler(newConsoleLogger())

# Only warnings and above go to a persistent file
addHandler(newFileLogger("app.log", levelThreshold = lvlWarn))

# A rotating file for all messages, cycling every 5000 lines
addHandler(newRollingFileLogger("trace.log", maxLines = 5000))

info("Server ready")      # → console + rolling file only
warn("High memory usage") # → all three
error("Request timeout")  # → all three
```

### Custom logger

```nim
import std/logging

type NetworkLogger = ref object of Logger
  endpoint: string

method log*(logger: NetworkLogger, level: Level,
            args: varargs[string, `$`]) =
  if level >= logger.levelThreshold:
    let msg = substituteLog(logger.fmtStr, level, args)
    sendHttpPost(logger.endpoint, msg)   # hypothetical

var netLog = NetworkLogger(
    endpoint: "https://logs.example.com/ingest",
    levelThreshold: lvlError,
    fmtStr: defaultFmtStr
)
addHandler(netLog)

error("Payment gateway unreachable")   # sent over HTTP
```

### Adjusting verbosity at runtime

```nim
import std/logging

addHandler(newConsoleLogger())

setLogFilter(lvlDebug)
debug("verbose startup diagnostics...")

setLogFilter(lvlWarn)
debug("this will be silently dropped now")
warn("but this will appear")
```

### Multi-thread logging (one registration per thread)

```nim
import std/logging, std/threadpool

proc workerThread() =
    # Must register handlers in each thread separately
    addHandler(newConsoleLogger(levelThreshold = lvlInfo))
    setLogFilter(lvlInfo)
    info("Worker thread started")

spawn workerThread()
sync()
```
