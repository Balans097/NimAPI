# `browsers` Module Reference

> **Module:** `std/browsers`  
> **Stability:** Unstable API  
> **Purpose:** Open URLs or local files in the user's default web browser in a cross-platform way.

---

## Overview

The `browsers` module provides a minimal, cross-platform interface for launching the system's default web browser. It abstracts away the platform-specific commands (`ShellExecute` on Windows, `open` on macOS, `xdg-open` on Linux/BSD) so you can open a URL with a single call.

The module is intentionally slim — it exports one constant and two procedures (one of which is deprecated).

---

## Exported Symbols

---

### `osOpenCmd`

```nim
const osOpenCmd*: string
```

#### What it is

A compile-time constant that holds the name of the OS-level command used to open a file or URL with its associated application. You rarely need to use this directly, but it is exported so that other tools (build scripts, shell wrappers, etc.) can reference the correct command name without hard-coding it.

| Platform | Value |
|---|---|
| Windows | `"open"` |
| macOS / Mac OS X | `"open"` |
| Linux, BSD, and others | `"xdg-open"` |

#### Why it matters

If you are building a tool that needs to shell out to the system opener yourself (for example, to pass extra flags), reading `osOpenCmd` keeps your code portable without `when defined(...)` blocks scattered everywhere.

#### Example

```nim
import std/browsers

# Just inspect the constant at runtime (its value is fixed at compile time)
echo "This system opens URLs with: ", osOpenCmd
# Windows/macOS → "open"
# Linux         → "xdg-open"
```

```nim
import std/browsers, std/osproc

# Build a custom shell command using the same opener the module would use
let cmd = osOpenCmd & " https://nim-lang.org"
discard execShellCmd(cmd)
```

---

### `openDefaultBrowser(url: string)`

```nim
proc openDefaultBrowser*(url: string)
```

#### What it does

Opens the given URL (or local file path) in the user's default browser. **The call does not block** — it hands the URL to the OS and returns immediately; your program keeps running while the browser window opens in the background.

#### How it works on each platform

- **Windows** — calls `ShellExecuteW`, the standard Windows API for opening a document with its associated application.
- **macOS** — runs `open <url>` via the shell. The `open` command is a macOS built-in that honours the user's default browser setting.
- **Linux / BSD / other Unix** — first tries `xdg-open`, which is the freedesktop.org standard. If that fails (exit code ≠ 0), the procedure falls back to iterating over the colon-separated list of browser executables in the `BROWSER` environment variable and starts the first one it can find using `startProcess` (non-blocking).

#### Important caveats

1. **Empty URL is forbidden.** Passing an empty string triggers a `doAssert` failure and terminates the program. Use a placeholder URL such as `"about:blank"` if you need a blank page (though see the deprecated overload below).
2. **No exception on failure.** If the browser cannot be opened, the procedure returns silently. There is no return value or error flag. This is a deliberate design choice in the module — check the module's source comment: *"This proc doesn't raise an exception on error, beware."*
3. **Local paths are supported.** If the string does not contain `"://"`, the private `prepare` helper prepends `"file://"` and converts the path to an absolute one. However, `prepare` is called internally only by `openDefaultBrowserRaw`, and `openDefaultBrowser` passes the URL straight through. Effectively, if you pass a bare file path without a scheme the behaviour depends on how the OS opener handles it — for guaranteed local file opening, prefix with `file://` yourself.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `url` | `string` | A non-empty URL (`https://…`, `http://…`, `file://…`) or a path. Must not be empty. |

#### Examples

```nim
import std/browsers

# Open a website — the browser appears, the program continues immediately
openDefaultBrowser("https://nim-lang.org")
echo "Browser launched (or at least we tried)."
```

```nim
import std/browsers

# Open a local HTML report
openDefaultBrowser("file:///home/user/report.html")
```

```nim
import std/browsers, std/os

# Build a file:// URL from a relative path safely
let reportPath = "file://" & absolutePath("build/report.html")
openDefaultBrowser(reportPath)
```

```nim
import std/browsers

# BAD — this will crash with a doAssert error:
# openDefaultBrowser("")

# GOOD — always guard against empty strings if the URL is dynamic
let url = getEnv("TARGET_URL", "https://example.com")
if url.len > 0:
  openDefaultBrowser(url)
```

---

### `openDefaultBrowser()` *(deprecated)*

```nim
proc openDefaultBrowser*() {.deprecated: "not implemented, please open with a specific url instead".}
```

> **Status:** Deprecated since Nim 1.1. Do **not** use in new code.

#### What it was meant to do

The intent was to open the user's default browser to a blank page, following [IETF RFC 6694 §3](https://tools.ietf.org/html/rfc6694#section-3), which reserves `"about:blank"` for this purpose.

#### What it actually does (and why it is deprecated)

The implementation simply calls `openDefaultBrowserRaw("about:blank")`, but the results are inconsistent and surprising across platforms:

| Platform | Actual behaviour |
|---|---|
| **Windows** | Pops up a dialog asking which application should handle the `about:` protocol — it does **not** open the browser to a blank tab. |
| **macOS** | Runs `open "about:blank"`, which does open the default browser to a blank page correctly. |
| **Linux/BSD** | `xdg-open` associates `about:blank` with the `text/html` MIME type rather than `x-scheme-handler/http`, so it may open a plain HTML viewer instead of your default browser. |

Because the behaviour cannot be made consistent across platforms without significant extra work, and the use-case (opening a blank page) is considered marginal, the Nim team deprecated this overload and recommends always passing an explicit URL.

#### Migration

```nim
# Before (deprecated):
openDefaultBrowser()

# After — be explicit about what you want:
openDefaultBrowser("https://example.com")
# or, if you truly want a blank page and accept the platform quirks:
openDefaultBrowser("about:blank")
```

---

## Platform Summary Table

| Feature | Windows | macOS | Linux / BSD |
|---|---|---|---|
| Underlying mechanism | `ShellExecuteW` (WinAPI) | `open` command | `xdg-open`, then `$BROWSER` |
| Blocks caller? | No | No | No |
| Local file support | Yes (via OS association) | Yes | Yes (`xdg-open`) |
| Exception on failure? | No | No | No |

---

## Common Patterns

### Open a URL and log a warning if it looks suspicious

```nim
import std/browsers, std/strutils

proc safeLaunch(url: string) =
  if url.len == 0:
    echo "Warning: attempted to open an empty URL"
    return
  if not (url.startsWith("https://") or url.startsWith("http://") or
          url.startsWith("file://")):
    echo "Warning: URL has no recognised scheme: ", url
  openDefaultBrowser(url)

safeLaunch("https://nim-lang.org")
```

### Open a locally generated report after a build step

```nim
import std/browsers, std/os

let outFile = getCurrentDir() / "build" / "index.html"
if fileExists(outFile):
  openDefaultBrowser("file://" & outFile)
else:
  echo "Report not found at: ", outFile
```

---

## Notes on Error Handling

Because `openDefaultBrowser` does not raise exceptions and provides no return value, you cannot know programmatically whether the browser actually opened. If you need reliability guarantees, consider:

- Checking that the `xdg-open` / `open` binary exists before calling the proc (Linux/macOS).
- On Linux, checking that the `BROWSER` environment variable is set as a fallback.
- Wrapping the call with a timeout and then verifying a side-effect (e.g., a temporary file the browser would write) — though this is rarely worth the complexity.

For most GUI and developer-tooling use-cases, silent failure is acceptable.
