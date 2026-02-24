# `appdirs` Module Reference

> **Module purpose:** `appdirs` provides cross-platform helpers for locating the standard system directories that applications use to store data, configuration, caches, and temporary files. Instead of hard-coding paths like `~/.config` or `C:\Users\Alice\AppData`, you ask the module and it figures out the right place for the current OS and user.

---

## Table of Contents

1. [getHomeDir](#gethomedir)
2. [getDataDir](#getdatadir)
3. [getConfigDir](#getconfigdir)
4. [getCacheDir (user)](#getcachedir-user)
5. [getCacheDir (app)](#getcachedir-app)
6. [getTempDir](#gettempdir)
7. [Platform summary table](#platform-summary-table)

---

## `getHomeDir`

```nim
proc getHomeDir*(): Path
```

### What it does

Returns the **home directory** of the currently logged-in user. This is the personal root folder where the OS keeps everything that belongs to one user account.

This proc is intentionally a low-level building block. For processing paths that come from user config files (e.g. paths starting with `~`), the higher-level `expandTilde` proc wraps it for convenience.

### Platform behaviour

| OS | Typical result |
|----|---------------|
| Linux / BSD | `/home/alice/` |
| macOS | `/Users/alice/` |
| Windows | `C:\Users\Alice\` |

### When to use it

Use `getHomeDir` when you need to construct a user-specific path that does not belong to any of the more specialised categories (data, config, cache, temp). For example, storing a dotfile directly in the home directory — though this is considered old-fashioned; prefer `getConfigDir` for configuration.

### Example

```nim
import std/appdirs
import std/paths

let home = getHomeDir()
echo home                          # /home/alice/

let dotfile = home / Path(".myapp")
echo dotfile                       # /home/alice/.myapp
```

---

## `getDataDir`

```nim
proc getDataDir*(): Path
```

### What it does

Returns the directory where applications should store **persistent user data** — things that belong to the user and must survive between sessions, but are not configuration (e.g. a local database, downloaded assets, saved game state).

On Linux and BSD this proc follows the **XDG Base Directory Specification**: it first checks the `XDG_DATA_HOME` environment variable. If that variable is not set, it falls back to `~/.local/share`. This means power users can redirect data storage simply by setting an environment variable — no code changes required.

### Platform behaviour

| OS | Environment variable checked | Default fallback |
|----|------------------------------|-----------------|
| Linux / BSD | `XDG_DATA_HOME` | `~/.local/share/` |
| macOS | `XDG_DATA_HOME` | `~/Library/Application Support/` |
| Windows | — | `%APPDATA%\` |

### When to use it

Choose `getDataDir` for anything the user would be unhappy to lose: a local database, imported files, game save files, offline cached content (when it is canonical, not throwaway). Do not use it for temporary files or machine-local caches that can be regenerated.

### Example

```nim
import std/appdirs
import std/paths
import std/os

let dataRoot = getDataDir()
let appData  = dataRoot / Path("myapp")

createDir(appData)                          # ensure directory exists
let db = appData / Path("storage.db")
echo "Database lives at: ", db
# Linux:   /home/alice/.local/share/myapp/storage.db
# macOS:   /Users/alice/Library/Application Support/myapp/storage.db
# Windows: C:\Users\Alice\AppData\Roaming\myapp\storage.db
```

---

## `getConfigDir`

```nim
proc getConfigDir*(): Path
```

### What it does

Returns the directory where applications should store **user configuration files** — settings, preferences, keybindings, and similar. The path is user-specific, so each account on the machine has its own configuration.

On Linux and BSD the proc conforms to the XDG Base Directory Specification, checking `XDG_CONFIG_HOME` first and defaulting to `~/.config/`. On macOS the same XDG variable is checked, but there is no widely-used Apple-native equivalent, so `~/.config/` is used there too.

> **Trailing separator:** The returned path always ends with a directory separator (`/` on POSIX, `\` on Windows). Keep this in mind when concatenating paths manually; using the `/` operator from `std/paths` handles it correctly regardless.

### Platform behaviour

| OS | Environment variable checked | Default fallback |
|----|------------------------------|-----------------|
| Linux / BSD | `XDG_CONFIG_HOME` | `~/.config/` |
| macOS | `XDG_CONFIG_HOME` | `~/.config/` |
| Windows | — | `%APPDATA%\` |

### When to use it

Store anything the user deliberately edits to change how your application behaves: `config.toml`, `settings.json`, `keybindings.yaml`, etc.

### Example

```nim
import std/appdirs
import std/paths
import std/os

let cfgDir = getConfigDir() / Path("myapp")
createDir(cfgDir)

let settings = cfgDir / Path("settings.toml")
echo "Config file: ", settings
# Linux:   /home/alice/.config/myapp/settings.toml
# Windows: C:\Users\Alice\AppData\Roaming\myapp\settings.toml
```

---

## `getCacheDir` (user)

```nim
proc getCacheDir*(): Path
```

### What it does

Returns the directory where applications should store **cache data** for the current user. A cache is expendable: it can be deleted at any time and the application must be able to regenerate or re-download its contents without data loss.

The proc checks platform-specific environment variables to respect user or system administrator overrides, then falls back to well-known defaults.

### Platform behaviour

| OS | Resolution logic |
|----|-----------------|
| Linux / BSD | `XDG_CACHE_HOME` → `~/.cache/` |
| macOS | `XDG_CACHE_HOME` → `~/Library/Caches/` |
| Windows | `%LOCALAPPDATA%\` |

Note that on Windows, `LOCALAPPDATA` (not `APPDATA`) is used — this is intentional because caches are machine-local and should not roam between devices in a domain environment.

### When to use it

Use for thumbnail images, compiled shaders, network response caches, pre-processed data — anything that speeds things up but is safe to discard. Operating systems and users may clear cache directories without warning.

### Example

```nim
import std/appdirs
import std/paths

let cacheRoot = getCacheDir()
echo cacheRoot
# Linux:   /home/alice/.cache/
# macOS:   /Users/alice/Library/Caches/
# Windows: C:\Users\Alice\AppData\Local\
```

---

## `getCacheDir` (app)

```nim
proc getCacheDir*(app: Path): Path
```

### What it does

A convenience overload that returns a **cache subdirectory scoped to a specific application**. You pass the application name (or any identifier) as `app`, and the proc constructs the full path automatically.

On Windows an extra `"cache"` segment is appended (because Windows does not have the same convention as POSIX systems of using `~/.cache/<appname>` directly):

- **Windows:** `getCacheDir() / app / "cache"`
- **All other platforms:** `getCacheDir() / app`

### When to use it

This is the overload you will call most often. Instead of manually joining `getCacheDir()` and your app name, this single call does it for you and handles the Windows quirk transparently.

### Example

```nim
import std/appdirs
import std/paths
import std/os

let myCache = getCacheDir(Path("myapp"))
createDir(myCache)

echo myCache
# Linux:   /home/alice/.cache/myapp
# macOS:   /Users/alice/Library/Caches/myapp
# Windows: C:\Users\Alice\AppData\Local\myapp\cache

let thumbnail = myCache / Path("thumb_42.png")
echo "Storing thumbnail at: ", thumbnail
```

---

## `getTempDir`

```nim
proc getTempDir*(): Path
```

### What it does

Returns the **system temporary directory** — the place where applications put files they only need for the duration of a single run or a short operation. Temporary files are not guaranteed to survive a reboot and should never hold information the user needs long-term.

The resolution strategy differs by platform:

- **Windows:** calls the native `GetTempPath` WinAPI function.
- **POSIX (Linux, macOS, BSD):** checks the environment variables `TMPDIR`, `TEMP`, `TMP`, and `TEMPDIR` in that order.
- **Fallback (all platforms):** returns `/tmp` if none of the above yield a result.

You can override the result at **compile time** by passing `-d:tempDir=mypath` to the Nim compiler — useful for sandboxed build environments or hermetic testing.

> **Important:** `getTempDir` does **not** verify that the returned path actually exists on disk. Always check or create it before writing files.

### When to use it

Use for intermediate results of a computation, lock files, extracted archives before moving them to their final location, or any file where loss on crash is acceptable. Never store user data here.

### Example

```nim
import std/appdirs
import std/paths
import std/os

let tmp = getTempDir()
echo tmp
# Linux:   /tmp/
# macOS:   /tmp/   (or value of $TMPDIR, e.g. /var/folders/.../T/)
# Windows: C:\Users\Alice\AppData\Local\Temp\

# Create a uniquely-named temporary file
let workFile = tmp / Path("myapp_work_" & $getCurrentProcessId() & ".dat")
echo "Working file: ", workFile
```

---

## Platform summary table

A quick reference showing what each function returns on the three major platforms, assuming default environment variables (i.e. no XDG overrides set) and the user `alice`.

| Function | Linux | macOS | Windows |
|---|---|---|---|
| `getHomeDir()` | `/home/alice/` | `/Users/alice/` | `C:\Users\Alice\` |
| `getDataDir()` | `/home/alice/.local/share/` | `/Users/alice/Library/Application Support/` | `C:\Users\Alice\AppData\Roaming\` |
| `getConfigDir()` | `/home/alice/.config/` | `/home/alice/.config/` | `C:\Users\Alice\AppData\Roaming\` |
| `getCacheDir()` | `/home/alice/.cache/` | `/Users/alice/Library/Caches/` | `C:\Users\Alice\AppData\Local\` |
| `getCacheDir(Path("app"))` | `/home/alice/.cache/app` | `/Users/alice/Library/Caches/app` | `C:\Users\Alice\AppData\Local\app\cache` |
| `getTempDir()` | `/tmp/` | `/tmp/` (or `$TMPDIR`) | `C:\Users\Alice\AppData\Local\Temp\` |
