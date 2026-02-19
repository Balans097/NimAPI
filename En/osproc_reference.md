# `osproc` Module Reference (Nim)

> Nim standard library module for launching and managing OS processes.  
> Provides facilities for spawning child processes, controlling their I/O streams, waiting for completion, and running commands in parallel.

---

## Table of Contents

1. [Overview](#overview)
2. [Types](#types)
   - [ProcessOption](#processoption)
   - [Process](#process)
3. [Launching Processes](#launching-processes)
   - [startProcess](#startprocess)
   - [execProcess](#execprocess)
   - [execCmd](#execcmd)
   - [execCmdEx](#execcmdex)
   - [execProcesses](#execprocesses)
4. [Process Control](#process-control)
   - [close](#close)
   - [suspend](#suspend)
   - [resume](#resume)
   - [terminate](#terminate)
   - [kill](#kill)
5. [Process State](#process-state)
   - [running](#running)
   - [processID](#processid)
   - [waitForExit](#waitforexit)
   - [peekExitCode](#peekexitcode)
   - [hasData](#hasdata)
6. [I/O Streams](#io-streams)
   - [inputStream](#inputstream)
   - [outputStream](#outputstream)
   - [errorStream](#errorstream)
   - [peekableOutputStream](#peekableoutputstream)
   - [peekableErrorStream](#peekableerrorstream)
   - [inputHandle](#inputhandle)
   - [outputHandle](#outputhandle)
   - [errorHandle](#errorhandle)
7. [Reading Output](#reading-output)
   - [lines (iterator)](#lines-iterator)
   - [readLines](#readlines)
8. [System Information](#system-information)
   - [countProcessors](#countprocessors)
9. [Re-exported Shell Utilities](#re-exported-shell-utilities)
10. [Practical Examples](#practical-examples)
11. [Quick Reference](#quick-reference)

---

## Overview

The `osproc` module provides advanced facilities for spawning and communicating with child processes. It lets you:
- Launch executables with arguments and a custom environment
- Read from and write to a child process's `stdin`/`stdout`/`stderr` streams
- Suspend, resume, or terminate processes
- Run many processes in parallel
- Retrieve exit codes

**Related modules:** `os`, `streams`, `memfiles`

---

## Types

### `ProcessOption`

```nim
type ProcessOption* = enum
  poEchoCmd,          ## Echo the command before execution
  poUsePath,          ## Search for the executable via PATH
  poEvalCommand,      ## Pass command directly to the shell
  poStdErrToStdOut,   ## Merge stderr into stdout
  poParentStreams,     ## Inherit the parent's I/O streams
  poInteractive,      ## Optimise buffering for interactive/UI apps
  poDaemon            ## Windows: no window; Unix: run as daemon (WIP)
```

Flags passed to `startProcess` to control the child process's behavior.

> ⚠️ `poEvalCommand` passes the command directly to `sh`/`cmd.exe`. Only use with trusted input!  
> ⚠️ On Windows, `poUsePath` is enabled by default.

---

### `Process`

```nim
type Process* = ref ProcessObj
```

An object representing a running OS process. Returned by `startProcess`.  
You **must** call `close` when finished with a process.

---

## Launching Processes

### `startProcess`

```nim
proc startProcess*(command: string, workingDir: string = "",
    args: openArray[string] = [], env: StringTableRef = nil,
    options: set[ProcessOption] = {poStdErrToStdOut}): owned(Process)
```

Starts a process and returns a `Process` object. Never returns `nil` — raises `OSError` on failure.

**Parameters:**
- `command` — path to the executable
- `workingDir` — working directory (`""` = current directory)
- `args` — command-line arguments, **not** including the executable name
- `env` — environment variables (`nil` = inherit from parent)
- `options` — set of `ProcessOption` flags

> ⚠️ You cannot pass `args` when using `poEvalCommand`.  
> ⚠️ Always call `close(p)` when done.

```nim
import std/[osproc, os]

# Simple launch
var p = startProcess("ls", args = ["-la"], options = {poUsePath})
echo p.outputStream.readAll()
p.close()

# Custom environment
import std/strtabs
var env = newStringTable({"MY_VAR": "hello"})
var p2 = startProcess("printenv", env = env, options = {poUsePath})
echo p2.outputStream.readAll()
p2.close()

# Custom working directory
var p3 = startProcess("pwd", workingDir = "/tmp", options = {poUsePath})
echo p3.outputStream.readAll()
p3.close()
```

---

### `execProcess`

```nim
proc execProcess*(command: string, workingDir: string = "",
    args: openArray[string] = [], env: StringTableRef = nil,
    options: set[ProcessOption] = {poStdErrToStdOut, poUsePath, poEvalCommand}): string
```

Convenience wrapper: runs a command and returns its entire output as a string. Blocks until completion.

> ⚠️ `poEvalCommand` is enabled by default for backward compatibility. Always pass `options` explicitly!

```nim
import std/osproc

# Shell invocation (poEvalCommand active by default)
let out1 = execProcess("ls -la /tmp")
echo out1

# Preferred — no shell:
let out2 = execProcess("ls", args = ["-la", "/tmp"], options = {poUsePath})
echo out2

# Compile a Nim file:
let result = execProcess("nim", args = ["c", "-r", "hello.nim"],
                          options = {poUsePath})
echo result
```

---

### `execCmd`

```nim
proc execCmd*(command: string): int
```

Runs `command` through the system shell and returns its exit code. stdin/stdout/stderr are inherited from the parent process (output goes directly to the terminal).

> Equivalent to the C `system()` function.

```nim
import std/osproc

let code = execCmd("echo Hello, World!")
echo "Exit code: ", code   # 0

let errCode = execCmd("ls /nonexistent")
echo "Error code: ", errCode  # != 0
```

---

### `execCmdEx`

```nim
proc execCmdEx*(command: string,
                options: set[ProcessOption] = {poStdErrToStdOut, poUsePath},
                env: StringTableRef = nil,
                workingDir = "",
                input = ""): tuple[output: string, exitCode: int]
```

Convenience wrapper that returns both the captured output and the exit code in a single tuple. If `input` is provided, it is written to the child's stdin.

> ⚠️ May block if `input.len` exceeds the OS pipe buffer size.

```nim
import std/osproc

# Basic usage
let (output, code) = execCmdEx("echo hello")
echo output    # "hello\n"
echo code      # 0

# Error check
let (_, errCode) = execCmdEx("ls /nonexistent")
echo errCode   # non-zero

# Custom environment (Posix only)
when defined(posix):
  import std/strtabs
  let (out2, _) = execCmdEx("echo $FOO",
                              env = newStringTable({"FOO": "bar"}))
  echo out2  # "bar\n"

# Pipe data to stdin
var res = execCmdEx("nim r --hints:off -", options = {}, input = "echo 3*4")
echo res.output    # "12\n"
echo res.exitCode  # 0

# Custom working directory
when defined(posix):
  let (pwd, _) = execCmdEx("echo $PWD", workingDir = "/")
  echo pwd  # "/\n"
```

---

### `execProcesses`

```nim
proc execProcesses*(cmds: openArray[string],
    options = {poStdErrToStdOut, poParentStreams},
    n = countProcessors(),
    beforeRunEvent: proc(idx: int) = nil,
    afterRunEvent: proc(idx: int, p: Process) = nil): int
```

Executes a list of commands `cmds` **in parallel**, running up to `n` processes simultaneously. Returns the highest absolute exit code across all processes.

**Parameters:**
- `cmds` — commands to run (each is passed via `poEvalCommand`)
- `options` — flags applied to every process
- `n` — maximum concurrent processes (default = number of processors)
- `beforeRunEvent` — callback invoked before starting the `idx`-th command
- `afterRunEvent` — callback invoked after the `idx`-th command finishes

```nim
import std/osproc

let commands = ["echo one", "echo two", "echo three", "echo four"]

let maxCode = execProcesses(commands, n = 2,
  beforeRunEvent = proc(i: int) =
    echo "Starting: ", commands[i],
  afterRunEvent = proc(i: int, p: Process) =
    echo "Finished: ", commands[i]
)
echo "Max exit code: ", maxCode
```

---

## Process Control

### `close`

```nim
proc close*(p: Process)
```

Releases all resources associated with process `p`. **Always call this when finished.**

> ⚠️ If the process is still running, it will be forcibly terminated. This may cause zombie processes.

```nim
var p = startProcess("sleep", args = ["5"], options = {poUsePath})
# ... do work ...
p.close()
```

---

### `suspend`

```nim
proc suspend*(p: Process)
```

Suspends execution of process `p`.  
On Unix — sends `SIGSTOP`. On Windows — calls `SuspendThread`.

```nim
var p = startProcess("some_long_task", options = {poUsePath})
p.suspend()
# ... do something else ...
p.resume()
p.close()
```

---

### `resume`

```nim
proc resume*(p: Process)
```

Resumes a suspended process `p`.  
On Unix — sends `SIGCONT`. On Windows — calls `ResumeThread`.

```nim
p.suspend()
# ... pause ...
p.resume()
```

---

### `terminate`

```nim
proc terminate*(p: Process)
```

Sends a graceful stop signal to process `p`. On Unix sends `SIGTERM` (the process can catch it and clean up), on Windows calls `TerminateProcess()`.

```nim
var p = startProcess("my_server", options = {poUsePath})
# ...
p.terminate()
discard p.waitForExit()
p.close()
```

---

### `kill`

```nim
proc kill*(p: Process)
```

Forcibly destroys process `p`. On Unix sends `SIGKILL` (cannot be caught or ignored). On Windows, this is an alias for `terminate`.

```nim
var p = startProcess("stubborn_process", options = {poUsePath})
if p.running:
  p.kill()
p.close()
```

---

## Process State

### `running`

```nim
proc running*(p: Process): bool
```

Returns `true` if process `p` is still executing. Does not block.

```nim
var p = startProcess("sleep", args = ["2"], options = {poUsePath})
echo p.running  # true
discard p.waitForExit()
echo p.running  # false
p.close()
```

---

### `processID`

```nim
proc processID*(p: Process): int
```

Returns the PID (process identifier) of the running process `p`.

```nim
var p = startProcess("sleep", args = ["10"], options = {poUsePath})
echo "PID: ", p.processID
p.terminate()
p.close()
```

---

### `waitForExit`

```nim
proc waitForExit*(p: Process, timeout: int = -1): int
```

Blocks the current thread until process `p` finishes. Returns the exit code.

**Parameters:**
- `timeout` — timeout in **milliseconds** (`-1` = wait indefinitely)

On Unix, if the process was killed by a signal, returns `128 + signal_number`.

> ⚠️ Do not use with processes that have buffered output streams without `poParentStreams` — may deadlock if the output buffer fills up.

```nim
var p = startProcess("ls", args = ["-la"],
                     options = {poUsePath, poStdErrToStdOut})
echo p.outputStream.readAll()
let code = p.waitForExit()
echo "Code: ", code
p.close()

# With timeout
var p2 = startProcess("sleep", args = ["10"], options = {poUsePath})
let code2 = p2.waitForExit(timeout = 1000)  # wait 1 second
echo code2
p2.close()
```

---

### `peekExitCode`

```nim
proc peekExitCode*(p: Process): int
```

Non-blocking. Returns `-1` if the process is still running, otherwise its exit code.  
On Unix, if killed by a signal, returns `128 + signal_number`.

```nim
var p = startProcess("sleep", args = ["5"], options = {poUsePath})
while p.peekExitCode() == -1:
  echo "Still running..."
  # ... do other work ...
echo "Exited with: ", p.peekExitCode()
p.close()
```

---

### `hasData`

```nim
proc hasData*(p: Process): bool
```

Returns `true` if there is data available to read from the process's output stream (non-blocking check). Useful for polling output without blocking.

```nim
var p = startProcess("ping", args = ["localhost", "-c", "3"],
                     options = {poUsePath})
while p.running or p.hasData:
  if p.hasData:
    echo p.outputStream.readLine()
p.close()
```

---

## I/O Streams

> ⚠️ **None of the returned streams or handles should be closed manually** — they are closed automatically when `close(p)` is called.

### `inputStream`

```nim
proc inputStream*(p: Process): Stream
```

Returns a stream for **writing** to the child process's stdin.

```nim
import std/[osproc, streams]

var p = startProcess("cat", options = {poUsePath})
p.inputStream.write("Hello from parent\n")
p.inputStream.flush()
# signal EOF to cat:
p.inputStream.close()
echo p.outputStream.readAll()
p.close()
```

---

### `outputStream`

```nim
proc outputStream*(p: Process): Stream
```

Returns a stream for **reading** the child process's stdout. Does not support peek/write/setOption — use `peekableOutputStream` for that.

```nim
import std/[osproc, streams]

var p = startProcess("echo", args = ["hello world"], options = {poUsePath})
let out = p.outputStream.readAll()
echo out  # "hello world\n"
p.close()
```

---

### `errorStream`

```nim
proc errorStream*(p: Process): Stream
```

Returns a stream for **reading** the child process's stderr. Does not support peek/write/setOption — use `peekableErrorStream` for that.

```nim
import std/[osproc, streams]

var p = startProcess("ls", args = ["/nonexistent"], options = {poUsePath})
discard p.waitForExit()
let err = p.errorStream.readAll()
echo "Error: ", err
p.close()
```

---

### `peekableOutputStream`

```nim
proc peekableOutputStream*(p: Process): Stream
```

Like `outputStream`, but the returned stream supports the peek operation (reading ahead without consuming data). Available since Nim 1.3.

```nim
import std/[osproc, streams]

var p = startProcess("echo", args = ["test"], options = {poUsePath})
let s = p.peekableOutputStream
# can peek before reading
discard p.waitForExit()
p.close()
```

---

### `peekableErrorStream`

```nim
proc peekableErrorStream*(p: Process): Stream
```

Like `errorStream`, but supports peek. Available since Nim 1.3.

---

### `inputHandle`

```nim
proc inputHandle*(p: Process): FileHandle
```

Returns the low-level `FileHandle` for the process's stdin. Useful for direct OS-level file descriptor operations.

---

### `outputHandle`

```nim
proc outputHandle*(p: Process): FileHandle
```

Returns the `FileHandle` for the process's stdout.

---

### `errorHandle`

```nim
proc errorHandle*(p: Process): FileHandle
```

Returns the `FileHandle` for the process's stderr.

```nim
import std/osproc

var p = startProcess("echo", args = ["hello"], options = {poUsePath})
echo "stdin fd:  ", p.inputHandle
echo "stdout fd: ", p.outputHandle
echo "stderr fd: ", p.errorHandle
p.close()
```

---

## Reading Output

### `lines` (iterator)

```nim
iterator lines*(p: Process, keepNewLines = false): string
```

Convenience iterator for reading stdout from a background process line by line. Automatically calls `waitForExit` when the stream ends. Available since Nim 1.3.

**Parameters:**
- `keepNewLines` — if `true`, newline characters are preserved in each yielded string

```nim
import std/osproc

# Run two processes in parallel and read their output
const opts = {poUsePath, poDaemon, poStdErrToStdOut}
var ps: seq[Process]
for prog in ["ls /tmp", "ls /etc"]:
  ps.add startProcess("sh", args = ["-c", prog], options = {poUsePath})

for p in ps:
  var i = 0
  for line in p.lines:
    echo line
    i.inc
    if i > 100: break
  p.close()
```

---

### `readLines`

```nim
proc readLines*(p: Process): (seq[string], int)
```

Reads all of the process's stdout and returns a tuple of `(lines, exitCode)`. Waits for the process to finish. Available since Nim 1.3.

```nim
import std/osproc

const opts = {poUsePath, poDaemon, poStdErrToStdOut}
var ps: seq[Process]
for prog in ["ls /tmp", "ls /etc"]:
  ps.add startProcess("sh", args = ["-c", prog], options = {poUsePath})

for p in ps:
  let (lines, exitCode) = p.readLines
  if exitCode != 0:
    echo "Error output:"
    for line in lines: echo line
  else:
    echo "Success: ", lines.len, " lines"
  p.close()
```

---

## System Information

### `countProcessors`

```nim
proc countProcessors*(): int
```

Returns the number of processors/cores the system has. Returns `0` if the count cannot be detected. Delegates to `cpuinfo.countProcessors` internally.

```nim
import std/osproc

echo "CPU cores: ", countProcessors()

# Run one process per core:
let n = max(1, countProcessors())
echo "Parallel processes: ", n
```

---

## Re-exported Shell Utilities

`osproc` re-exports the following functions from the `os` module:

| Function | Description |
|----------|-------------|
| `quoteShell(s)` | Escapes a string for safe shell use (platform-dependent) |
| `quoteShellWindows(s)` | Escapes a string for `cmd.exe` |
| `quoteShellPosix(s)` | Escapes a string for a Unix shell |

```nim
import std/osproc

let arg = "hello world"
echo quoteShell(arg)         # 'hello world' (Unix) or "hello world" (Windows)
echo quoteShellPosix(arg)    # 'hello world'
echo quoteShellWindows(arg)  # "hello world"

# Safe command construction:
let cmd = "myapp " & quoteShell("/path/with spaces/file.txt")
let code = execCmd(cmd)
```

---

## Practical Examples

### Example 1: Capture stdout and stderr separately

```nim
import std/[osproc, streams]

var p = startProcess("ls", args = ["/nonexistent", "/tmp"],
                     options = {poUsePath})
let stdout_out = p.outputStream.readAll()
let stderr_out = p.errorStream.readAll()
let code = p.waitForExit()

echo "stdout:\n", stdout_out
echo "stderr:\n", stderr_out
echo "Exit code: ", code
p.close()
```

---

### Example 2: Two-way communication (stdin → stdout)

```nim
import std/[osproc, streams]

# Launch Python for a computation
var p = startProcess("python3", options = {poUsePath})
p.inputStream.write("print(2 ** 32)\n")
p.inputStream.flush()
p.inputStream.close()  # signal EOF

let result = p.outputStream.readAll()
echo result.strip()  # "4294967296"
discard p.waitForExit()
p.close()
```

---

### Example 3: Non-blocking process monitoring

```nim
import std/[osproc, os]

var p = startProcess("ping", args = ["-c", "5", "localhost"],
                     options = {poUsePath, poStdErrToStdOut})

while p.running:
  if p.hasData:
    let line = p.outputStream.readLine()
    echo ">> ", line
  else:
    sleep(50)

let code = p.waitForExit()
echo "Exited: ", code
p.close()
```

---

### Example 4: Parallel data collection

```nim
import std/osproc

# Ping multiple hosts simultaneously
let hosts = ["google.com", "github.com", "nim-lang.org"]
var procs = newSeq[Process](hosts.len)

for i, host in hosts:
  procs[i] = startProcess("ping",
    args = ["-c", "1", host],
    options = {poUsePath, poStdErrToStdOut})

for i, p in procs:
  let code = p.waitForExit()
  if code == 0:
    echo hosts[i], " — reachable"
  else:
    echo hosts[i], " — unreachable"
  p.close()
```

---

### Example 5: Using `execProcesses` with callbacks

```nim
import std/osproc

let tasks = [
  "echo task_1",
  "sleep 1 && echo task_2",
  "echo task_3"
]

let maxCode = execProcesses(tasks, n = 2,
  beforeRunEvent = proc(i: int) =
    echo "[START] ", tasks[i],
  afterRunEvent = proc(i: int, p: Process) =
    echo "[DONE]  ", tasks[i]
)
echo "Highest exit code: ", maxCode
```

---

### Example 6: Safe launch with `poUsePath` instead of `poEvalCommand`

```nim
import std/osproc

# BAD: vulnerable to shell injection
let bad = execProcess("ls " & userInput)

# GOOD: arguments passed separately
let good = execProcess("ls", args = [userInput], options = {poUsePath})
echo good
```

---

## Quick Reference

### Choosing a launch function

| Function | Blocks? | Returns | When to use |
|----------|---------|---------|-------------|
| `execCmd` | Yes | Exit code | Just need the code; output goes to terminal |
| `execProcess` | Yes | Output string | Need all output as a single string |
| `execCmdEx` | Yes | `(output, code)` | Need both output and exit code |
| `startProcess` | No | `Process` | Full control, background tasks |
| `execProcesses` | Yes | Max code | Parallel execution of a list of commands |

### Process management

| Goal | Code |
|------|------|
| Start | `var p = startProcess("cmd", args=["a"], options={poUsePath})` |
| Wait for completion | `discard p.waitForExit()` |
| Wait with timeout | `discard p.waitForExit(timeout=5000)` |
| Check if alive | `p.running` |
| Peek at exit code | `p.peekExitCode()` |
| Suspend | `p.suspend()` |
| Resume | `p.resume()` |
| Graceful stop | `p.terminate()` |
| Force kill | `p.kill()` |
| Free resources | `p.close()` |
| Get PID | `p.processID` |

### Streams

| Stream | Direction | Function |
|--------|-----------|----------|
| stdin | → process | `p.inputStream` |
| stdout | ← process | `p.outputStream` |
| stderr | ← process | `p.errorStream` |
| stdout (peek) | ← process | `p.peekableOutputStream` |
| stderr (peek) | ← process | `p.peekableErrorStream` |
| stdin fd | → process | `p.inputHandle` |
| stdout fd | ← process | `p.outputHandle` |
| stderr fd | ← process | `p.errorHandle` |

### Common option sets

```nim
# Capture output, find via PATH
{poUsePath, poStdErrToStdOut}

# Background / daemon process
{poUsePath, poDaemon, poStdErrToStdOut}

# Parallel batch execution
{poStdErrToStdOut, poParentStreams}

# Shell invocation (trusted input only!)
{poUsePath, poEvalCommand}
```
