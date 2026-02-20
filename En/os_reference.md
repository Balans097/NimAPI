# Nim `os` Module — Complete Reference

> **Module:** `std/os`  
> **Purpose:** Basic operating system facilities — environment variables, directories, paths, file info, shell commands, and more.

---

## Table of Contents

1. [Constants](#constants)
2. [Types](#types)
3. [Path Manipulation](#path-manipulation)
4. [Shell Quoting](#shell-quoting)
5. [Executable Search](#executable-search)
6. [File Timestamps](#file-timestamps)
7. [File Size & Info](#file-size--info)
8. [File Permissions](#file-permissions)
9. [Hard Links](#hard-links)
10. [Process & System](#process--system)
11. [Application Info](#application-info)
12. [Hidden Files & Filename Validation](#hidden-files--filename-validation)
13. [Deprecated Aliases](#deprecated-aliases)

---

## Constants

### `invalidFilenameChars`

```nim
const invalidFilenameChars*: set[char]
```

A set of characters that may produce invalid filenames across Linux, Windows, and macOS.

| Platform | Banned characters |
|----------|------------------|
| Linux    | `/` |
| macOS    | `:` |
| Windows  | All others in the set |

**Value:** `{'/', '\\', ':', '*', '?', '"', '<', '>', '|', '^', '\0'}`

**Example:**
```nim
import std/os

let name = "my:file*name"
for ch in name:
  if ch in invalidFilenameChars:
    echo "Invalid character: ", ch
# Output:
# Invalid character: :
# Invalid character: *
```

---

### `invalidFilenames`

```nim
const invalidFilenames*: array[...]
```

A list of filenames that are reserved/invalid on Windows (and possibly other platforms). Includes `CON`, `PRN`, `AUX`, `NUL`, `COM0`–`COM9`, `LPT0`–`LPT9`.

**Example:**
```nim
import std/os, std/strutils

let name = "CON"
for invalid in invalidFilenames:
  if cmpIgnoreCase(name, invalid) == 0:
    echo name, " is a reserved filename!"
# Output: CON is a reserved filename!
```

---

### `ExeExts`

```nim
const ExeExts*: array[...]
```

Platform-specific file extensions for executables.

- **Windows:** `["exe", "cmd", "bat"]`
- **POSIX:** `[""]`

**Example:**
```nim
import std/os
echo ExeExts  # on Linux: [""]
              # on Windows: ["exe", "cmd", "bat"]
```

---

## Types

### `FileInfo`

```nim
type FileInfo* = object
  id*: tuple[device: DeviceId, file: FileId]
  kind*: PathComponent
  size*: BiggestInt
  permissions*: set[FilePermission]
  linkCount*: BiggestInt
  lastAccessTime*: times.Time
  lastWriteTime*: times.Time
  creationTime*: times.Time
  blockSize*: int
  isSpecial*: bool
```

Contains information associated with a file object. Returned by `getFileInfo`.

| Field | Description |
|-------|-------------|
| `id` | Device and file ID tuple |
| `kind` | `pcFile`, `pcDir`, `pcLinkToFile`, `pcLinkToDir` |
| `size` | File size in bytes |
| `permissions` | Set of `FilePermission` values |
| `linkCount` | Number of hard links |
| `lastAccessTime` | Last access timestamp |
| `lastWriteTime` | Last modification timestamp |
| `creationTime` | Creation timestamp (not reliable on all systems) |
| `blockSize` | Preferred I/O block size |
| `isSpecial` | True for non-regular files (FIFOs, devices) on Unix |

---

### `DeviceId` / `FileId`

```nim
when defined(windows):
  type DeviceId* = int32
  type FileId* = int64
elif defined(posix):
  type DeviceId* = Dev
  type FileId* = Ino
```

Platform-specific types for device and inode identifiers.

---

## Path Manipulation

### `expandTilde`

```nim
proc expandTilde*(path: string): string
```

Expands `~` or `~/...` to the full home directory path. Returns `path` unchanged if it does not start with `~`. On Windows both `~/` and `~\` are handled.

**Parameters:**
- `path` — the path string, possibly starting with `~`

**Returns:** Expanded absolute path string.

**Example:**
```nim
import std/os

echo expandTilde("~/documents")      # e.g. /home/user/documents
echo expandTilde("~")                # e.g. /home/user/
echo expandTilde("/absolute/path")   # /absolute/path (unchanged)
echo expandTilde("relative/path")    # relative/path (unchanged)
```

---

## Shell Quoting

### `quoteShellWindows`

```nim
proc quoteShellWindows*(s: string): string
```

Quotes a string so it can be safely passed to the Windows API / `cmd.exe`. Based on Python's `subprocess.list2cmdline`. Handles spaces, tabs, backslashes, and double-quotes.

**Parameters:**
- `s` — the argument string to quote

**Returns:** Safely quoted string for Windows shell.

**Example:**
```nim
import std/os

echo quoteShellWindows("hello world")   # "hello world"
echo quoteShellWindows("no-spaces")     # no-spaces
echo quoteShellWindows(`with"quote`)    # "with\"quote"
echo quoteShellWindows("")              # ""
```

---

### `quoteShellPosix`

```nim
proc quoteShellPosix*(s: string): string
```

Quotes a string so it can be safely passed to a POSIX shell. Safe characters (alphanumeric, `%+-./_ :=@`) are left unquoted; everything else is wrapped in single quotes.

**Parameters:**
- `s` — the argument string to quote

**Returns:** Safely quoted string for POSIX shell.

**Example:**
```nim
import std/os

echo quoteShellPosix("hello world")    # 'hello world'
echo quoteShellPosix("safe-string")    # safe-string
echo quoteShellPosix("")               # ''
echo quoteShellPosix("it's here")      # 'it'"'"'s here'
```

---

### `quoteShell`

```nim
proc quoteShell*(s: string): string
```

Quotes a string for the current platform's shell. Calls `quoteShellWindows` on Windows, `quoteShellPosix` on POSIX. Available on Windows, POSIX, and Nintendo Switch targets.

**Parameters:**
- `s` — argument to quote

**Returns:** Platform-appropriate quoted string.

**Example:**
```nim
import std/os

let arg = "hello world"
echo quoteShell(arg)
# Linux/macOS: 'hello world'
# Windows:     "hello world"
```

---

### `quoteShellCommand`

```nim
proc quoteShellCommand*(args: openArray[string]): string
```

Concatenates and quotes multiple shell arguments into a single command string ready for shell execution.

**Parameters:**
- `args` — sequence of argument strings

**Returns:** Space-separated, individually quoted command string.

**Example:**
```nim
import std/os

# On POSIX:
echo quoteShellCommand(["ls", "-la", "my dir"])
# ls -la 'my dir'

# On Windows:
echo quoteShellCommand(["dir", "", "C:\\Program Files"])
# dir "" "C:\Program Files"
```

---

## Executable Search

### `findExe`

```nim
proc findExe*(exe: string, followSymlinks: bool = true;
              extensions: openArray[string] = ExeExts): string
```

Searches for an executable named `exe` first in the current working directory, then in directories listed in the `PATH` environment variable. Automatically appends platform-specific extensions if the name has none.

**Parameters:**
- `exe` — executable name to find (without or with extension)
- `followSymlinks` — if `true` (default), resolves symlinks to the real file
- `extensions` — list of extensions to try; defaults to `ExeExts`

**Returns:** Full path to the executable, or `""` if not found.

**Not available on:** NimScript / JS targets.

**Example:**
```nim
import std/os

let pythonPath = findExe("python3")
if pythonPath.len > 0:
  echo "Found Python at: ", pythonPath
else:
  echo "Python not found in PATH"

# Search without following symlinks:
let path = findExe("gcc", followSymlinks = false)
echo path
```

---

## File Timestamps

### `getLastModificationTime`

```nim
proc getLastModificationTime*(file: string): times.Time
```

Returns the file's last modification time.

**Parameters:**
- `file` — path to the file

**Returns:** `times.Time` value.

**Raises:** `OSError` if the file cannot be accessed.

**Example:**
```nim
import std/os, std/times

let t = getLastModificationTime("myfile.txt")
echo t.format("yyyy-MM-dd HH:mm:ss")
```

---

### `getLastAccessTime`

```nim
proc getLastAccessTime*(file: string): times.Time
```

Returns the time when the file was last read or written.

**Parameters:**
- `file` — path to the file

**Returns:** `times.Time` value.

**Raises:** `OSError` if the file cannot be accessed.

**Example:**
```nim
import std/os, std/times

let t = getLastAccessTime("myfile.txt")
echo "Last accessed: ", t.format("yyyy-MM-dd HH:mm:ss")
```

---

### `getCreationTime`

```nim
proc getCreationTime*(file: string): times.Time
```

Returns the file's creation time.

> **Note (POSIX):** On POSIX systems this may actually reflect the time when the file's attributes were last changed (`ctime`), not the actual creation time.

**Parameters:**
- `file` — path to the file

**Returns:** `times.Time` value.

**Raises:** `OSError` if the file cannot be accessed.

**Example:**
```nim
import std/os, std/times

let t = getCreationTime("myfile.txt")
echo "Created: ", t.format("yyyy-MM-dd")
```

---

### `fileNewer`

```nim
proc fileNewer*(a, b: string): bool
```

Returns `true` if file `a` has a later modification time than file `b`.

**Parameters:**
- `a` — path to the first file
- `b` — path to the second file

**Returns:** `true` if `a` is newer than `b`.

**Example:**
```nim
import std/os

if fileNewer("output.bin", "source.nim"):
  echo "Output is up-to-date"
else:
  echo "Need to recompile"
```

---

### `setLastModificationTime`

```nim
proc setLastModificationTime*(file: string, t: times.Time)
```

Sets the last modification time of a file.

**Parameters:**
- `file` — path to the file
- `t` — desired modification time

**Raises:** `OSError` on failure.

**Example:**
```nim
import std/os, std/times

let newTime = now().toTime()
setLastModificationTime("myfile.txt", newTime)
echo "Timestamp updated"
```

---

## File Size & Info

### `getFileSize`

```nim
proc getFileSize*(file: string): BiggestInt
```

Returns the size of the file in bytes.

**Parameters:**
- `file` — path to the file

**Returns:** File size as `BiggestInt`.

**Raises:** `OSError` on failure.

**Example:**
```nim
import std/os

let size = getFileSize("myfile.txt")
echo "File size: ", size, " bytes"

# Human-readable
if size < 1024:
  echo size, " B"
elif size < 1024 * 1024:
  echo size div 1024, " KB"
else:
  echo size div (1024*1024), " MB"
```

---

### `getFileInfo` (by handle)

```nim
proc getFileInfo*(handle: FileHandle): FileInfo
```

Retrieves `FileInfo` for the file represented by a system file handle.

**Raises:** `OSError` if the handle is invalid.

**Example:**
```nim
import std/os

var f: File
if open(f, "myfile.txt"):
  let info = getFileInfo(f.getFileHandle())
  echo "Size: ", info.size
  close(f)
```

---

### `getFileInfo` (by File)

```nim
proc getFileInfo*(file: File): FileInfo
```

Retrieves `FileInfo` for an open `File` object.

**Raises:** `IOError` if `file` is nil; `OSError` on system error.

**Example:**
```nim
import std/os

var f: File
if open(f, "myfile.txt"):
  let info = getFileInfo(f)
  echo "Block size: ", info.blockSize
  echo "Is directory: ", info.kind == pcDir
  close(f)
```

---

### `getFileInfo` (by path)

```nim
proc getFileInfo*(path: string, followSymlink = true): FileInfo
```

Retrieves `FileInfo` for the file at `path`. When `followSymlink` is `true` (default), symlinks are followed. When `false`, info about the symlink itself is returned.

**Parameters:**
- `path` — file system path
- `followSymlink` — whether to follow symbolic links

**Raises:** `OSError` if path does not exist or access is denied.

**Example:**
```nim
import std/os

let info = getFileInfo("/etc/hosts")
echo "Kind: ", info.kind         # pcFile
echo "Size: ", info.size
echo "Permissions: ", info.permissions

# For a symlink, get info about the link itself:
let symlinkInfo = getFileInfo("/etc/localtime", followSymlink = false)
echo "Is symlink: ", symlinkInfo.kind in {pcLinkToFile, pcLinkToDir}
```

---

### `sameFileContent`

```nim
proc sameFileContent*(path1, path2: string): bool
```

Returns `true` if both files have identical binary content. Uses buffered reading for efficiency.

**Parameters:**
- `path1` — path to the first file
- `path2` — path to the second file

**Returns:** `true` if content is byte-for-byte identical.

**Example:**
```nim
import std/os

if sameFileContent("original.txt", "backup.txt"):
  echo "Files are identical"
else:
  echo "Files differ"
```

---

## File Permissions

### `inclFilePermissions`

```nim
proc inclFilePermissions*(filename: string, permissions: set[FilePermission])
```

Adds the given permissions to a file's existing permission set. Equivalent to:
```nim
setFilePermissions(filename, getFilePermissions(filename) + permissions)
```

**Parameters:**
- `filename` — path to the file
- `permissions` — set of `FilePermission` values to add

**Example:**
```nim
import std/os

# Make a file executable by the owner:
inclFilePermissions("myscript.sh", {fpUserExec})

# Add group read permission:
inclFilePermissions("shared.cfg", {fpGroupRead})
```

---

### `exclFilePermissions`

```nim
proc exclFilePermissions*(filename: string, permissions: set[FilePermission])
```

Removes the given permissions from a file's existing permission set. Equivalent to:
```nim
setFilePermissions(filename, getFilePermissions(filename) - permissions)
```

**Parameters:**
- `filename` — path to the file
- `permissions` — set of `FilePermission` values to remove

**Example:**
```nim
import std/os

# Remove write permission for others:
exclFilePermissions("public.html", {fpOthersWrite})

# Make file read-only for everyone:
exclFilePermissions("readonly.cfg", {fpUserWrite, fpGroupWrite, fpOthersWrite})
```

---

## Hard Links

### `createHardlink`

```nim
proc createHardlink*(src, dest: string)
```

Creates a hard link at `dest` pointing to `src`. Both paths then refer to the same file data.

> **Warning:** Some operating systems restrict hard link creation to root/administrators.

**Parameters:**
- `src` — existing file path
- `dest` — path where the hard link will be created

**Raises:** `OSError` on failure.

**Example:**
```nim
import std/os

createHardlink("original.dat", "hardlink.dat")

# Both now point to the same data:
echo sameFileContent("original.dat", "hardlink.dat")  # true

# Verify link count increased:
let info = getFileInfo("original.dat")
echo "Link count: ", info.linkCount  # 2
```

---

## Process & System

### `sleep`

```nim
proc sleep*(milsecs: int)
```

Pauses execution for `milsecs` milliseconds. A negative value returns immediately.

**Parameters:**
- `milsecs` — number of milliseconds to sleep

**Example:**
```nim
import std/os

echo "Starting..."
sleep(2000)    # sleep for 2 seconds
echo "Done!"

# Negative value is safe and does nothing:
sleep(-100)
```

---

### `execShellCmd`

```nim
proc execShellCmd*(command: string): int
```

Executes a shell command and waits for it to complete. Returns the shell exit code (0 on success).

> **Note:** For more control over process I/O, use `osproc.execProcess`.

**Parameters:**
- `command` — shell command string (e.g. `"ls -la"`)

**Returns:** Exit code of the shell process.

**Example:**
```nim
import std/os

let code = execShellCmd("echo Hello from shell")
echo "Exit code: ", code   # 0

# Check if a command succeeds:
if execShellCmd("which python3") == 0:
  echo "python3 is available"

# Running a failing command:
let err = execShellCmd("false")
echo "Error code: ", err   # non-zero
```

---

### `exitStatusLikeShell`

```nim
proc exitStatusLikeShell*(status: cint): cint
```

Converts a raw exit status from `c_system` (the C `system()` call) into a shell-style exit code. On POSIX, if the process was killed by a signal, returns `128 + signal_number`.

**Parameters:**
- `status` — raw status from C `system()` call

**Returns:** Shell-style exit code.

**Example:**
```nim
import std/os

# Usually used internally, but can be useful for wrapping C system calls:
let rawStatus: cint = 0
echo exitStatusLikeShell(rawStatus)  # 0
```

---

### `isAdmin`

```nim
proc isAdmin*: bool
```

Returns `true` if the current process is running with administrator/root privileges.

- **Windows:** checks membership in the Administrators local group
- **POSIX:** checks if effective user ID (`euid`) is 0

**Example:**
```nim
import std/os

if isAdmin():
  echo "Running as administrator/root"
else:
  echo "Running as regular user"
  echo "Some operations may require elevated privileges"
```

---

### `getCurrentProcessId`

```nim
proc getCurrentProcessId*(): int
```

Returns the current process ID (PID).

**Returns:** Integer PID of the current process.

**Example:**
```nim
import std/os

echo "Current PID: ", getCurrentProcessId()
# e.g. Current PID: 12345
```

---

## Application Info

### `expandFilename`

```nim
proc expandFilename*(filename: string): string
```

Returns the full, absolute path of an existing file. Follows symbolic links to resolve the true path.

**Parameters:**
- `filename` — relative or absolute path to an existing file

**Returns:** Absolute resolved path.

**Raises:** `OSError` if the file does not exist or cannot be accessed.

**Example:**
```nim
import std/os

# From a script in /home/user/projects/:
echo expandFilename("../data/file.txt")
# e.g. /home/user/data/file.txt

echo expandFilename("./main.nim")
# e.g. /home/user/projects/main.nim
```

---

### `getAppFilename`

```nim
proc getAppFilename*(): string
```

Returns the absolute path of the currently running application's executable. Resolves symbolic links. Returns an empty string if the path cannot be determined.

**Returns:** Absolute path string, or `""` on failure.

**Example:**
```nim
import std/os

let exe = getAppFilename()
echo "This application: ", exe
# e.g. /usr/local/bin/myapp
```

---

### `getAppDir`

```nim
proc getAppDir*(): string
```

Returns the directory containing the current application's executable. Equivalent to `splitFile(getAppFilename()).dir`.

**Returns:** Directory path string.

**Example:**
```nim
import std/os

let dir = getAppDir()
echo "Application directory: ", dir

# Load a config file relative to the executable:
let config = dir / "config.ini"
if fileExists(config):
  echo "Config found at: ", config
```

---

### `getCurrentCompilerExe`

```nim
proc getCurrentCompilerExe*(): string {.compileTime.}
```

Returns the path of the currently running Nim compiler (or nimble) at **compile time**. Useful in macros or nimscript to invoke the compiler programmatically.

**Returns:** Compile-time path to the Nim compiler executable.

**Example:**
```nim
import std/os

const nimPath = getCurrentCompilerExe()
echo "Compiler: ", nimPath  # evaluated at compile time
```

---

## Hidden Files & Filename Validation

### `isHidden`

```nim
proc isHidden*(path: string): bool
```

Determines whether the given path refers to a hidden file or directory.

- **POSIX:** returns `true` if the last path component starts with `.` and is not `.` or `..`
- **Windows:** returns `true` if the file exists and has the `FILE_ATTRIBUTE_HIDDEN` attribute set

> **Note:** Paths are not normalized before checking.

**Parameters:**
- `path` — file or directory path

**Returns:** `true` if hidden.

**Example:**
```nim
import std/os

# POSIX examples:
echo isHidden(".bashrc")          # true
echo isHidden("/home/user/.ssh")  # true (last component is .ssh)
echo isHidden(".foo/bar")         # false (bar is not hidden)
echo isHidden(".")                # false
echo isHidden("..")               # false
echo isHidden(".hidden/")         # true (trailing slash OK)
echo isHidden("visible.txt")      # false
```

---

### `isValidFilename`

```nim
func isValidFilename*(filename: string, maxLen = 259.Positive): bool
```

Returns `true` if `filename` is valid for cross-platform use (Windows, Linux, macOS). Checks against `invalidFilenameChars`, `invalidFilenames`, and maximum length.

> **Warning:** Only checks the filename component, not full paths.

**Parameters:**
- `filename` — filename string to validate (just the name, no path)
- `maxLen` — maximum allowed length (default 259, the Windows `MAX_PATH` limit minus one)

**Returns:** `true` if valid for cross-platform use.

**Rules that fail validation:**
- Leading or trailing whitespace
- Ends with a dot `.`
- Contains any `invalidFilenameChars` (`/ \ : * ? " < > | ^ \0`)
- Matches a Windows reserved name (`CON`, `AUX`, `NUL`, `COM1`–`COM9`, `LPT1`–`LPT9`, etc.) case-insensitively
- Empty string
- Exceeds `maxLen`

**Example:**
```nim
import std/os

echo isValidFilename("myFile.txt")    # true
echo isValidFilename(" myFile.txt")   # false (leading space)
echo isValidFilename("myFile.txt ")   # false (trailing space)
echo isValidFilename("myFile.")       # false (ends with dot)
echo isValidFilename("con.txt")       # false (reserved name)
echo isValidFilename("my:file")       # false (colon invalid on Mac/Win)
echo isValidFilename("")              # false (empty)
echo isValidFilename("foo/bar")       # false (contains slash)

# Custom max length:
echo isValidFilename("a".repeat(260), maxLen = 100)  # false
```

---

## Deprecated Aliases

### `existsFile` *(deprecated)*

```nim
template existsFile*(args: varargs[untyped]): untyped {.deprecated: "use fileExists".}
```

Use `fileExists` instead.

---

### `existsDir` *(deprecated)*

```nim
template existsDir*(args: varargs[untyped]): untyped {.deprecated: "use dirExists".}
```

Use `dirExists` instead.

---

## Re-exported Modules

The `os` module re-exports the following sub-modules, making all their symbols available:

| Module | Contents |
|--------|----------|
| `std/private/ospaths2` | Path joining, splitting, normalization (`joinPath`, `splitPath`, `splitFile`, `changeFileExt`, `addFileExt`, `parentDir`, `isAbsolute`, etc.) |
| `std/private/osfiles` | File operations (`copyFile`, `moveFile`, `removeFile`, `fileExists`, `getFilePermissions`, `setFilePermissions`) |
| `std/private/osdirs` | Directory operations (`createDir`, `removeDir`, `copyDir`, `dirExists`, `walkDir`, `walkDirRec`, `getCurrentDir`, `setCurrentDir`) |
| `std/private/ossymlinks` | Symlink operations (`createSymlink`, `expandSymlink`, `symlinkExists`) |
| `std/private/osappdirs` | Application directories (`getHomeDir`, `getConfigDir`, `getTempDir`, `getAppDir`) |
| `std/cmdline` | Command-line arguments (`paramCount`, `paramStr`, `commandLineParams`) |
| `std/oserrors` | OS error types (`OSError`, `osLastError`, `raiseOSError`) |
| `std/envvars` | Environment variables (`getEnv`, `putEnv`, `delEnv`, `existsEnv`, `envPairs`) |
| `std/private/osseps` | Path separators (`DirSep`, `AltSep`, `PathSep`, `ExeExt`) |

---

## Platform Availability Summary

| Function | Linux | macOS | Windows | NimScript | JS |
|----------|:-----:|:-----:|:-------:|:---------:|:--:|
| `expandTilde` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `quoteShellWindows` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `quoteShellPosix` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `quoteShell` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `quoteShellCommand` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `findExe` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getLastModificationTime` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getLastAccessTime` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getCreationTime` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `fileNewer` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `setLastModificationTime` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getFileSize` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getFileInfo` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `sameFileContent` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `inclFilePermissions` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `exclFilePermissions` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `createHardlink` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `sleep` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `execShellCmd` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `isAdmin` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getCurrentProcessId` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `expandFilename` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getAppFilename` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getAppDir` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getCurrentCompilerExe` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `isHidden` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `isValidFilename` | ✓ | ✓ | ✓ | ✓ | ✓ |

---

*Generated from Nim's standard library `std/os` module.*
