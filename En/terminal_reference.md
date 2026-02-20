# Nim `terminal` Module — Complete Reference

> **Module:** `std/terminal`  
> **Purpose:** Control the terminal/console — colors, styles, cursor movement, screen erasure, terminal size, password input, and more.  
> **Implementation:** ANSI escape sequences on UNIX/macOS; Windows Console API on Windows.

---

## ⚠️ Important Notes

- **Style changes are persistent** — they survive after your program exits. Use `exitprocs.addExitProc(resetAttributes)` to restore defaults on exit.
- **Hidden cursor** — if you call `hideCursor`, always call `showCursor` before quitting.
- **Two categories of styling functions:**
  - `styledWriteLine`, `styledEcho`, `styledWrite` — **temporary** effect, auto-reset after the call.
  - `setForegroundColor`, `setBackgroundColor`, `setStyle` — **persistent** effect, must manually call `resetAttributes`.

---

## Table of Contents

1. [Constants](#constants)
2. [Types](#types)
3. [Terminal Size](#terminal-size)
4. [Cursor Position](#cursor-position)
5. [Cursor Visibility](#cursor-visibility)
6. [Cursor Movement](#cursor-movement)
7. [Screen & Line Erasure](#screen--line-erasure)
8. [Text Styles](#text-styles)
9. [Colors](#colors)
10. [True Color (24-bit)](#true-color-24-bit)
11. [ANSI Code Helpers](#ansi-code-helpers)
12. [Styled Output](#styled-output)
13. [Input](#input)
14. [Utility](#utility)
15. [stdout Shorthand Templates](#stdout-shorthand-templates)
16. [Platform Availability Summary](#platform-availability-summary)
17. [Complete Examples](#complete-examples)

---

## Constants

### `ansiResetCode`

```nim
const ansiResetCode* = "\e[0m"
```

The ANSI escape sequence string that resets all terminal attributes (color, brightness, style) to defaults. Can be written directly to a file or embedded in strings.

**Example:**
```nim
import std/terminal
echo "colored text" & ansiResetCode & "normal text"
```

---

## Types

### `Style`

```nim
type Style* = enum
  styleBright = 1,        ## bright/bold text
  styleDim,               ## dim text
  styleItalic,            ## italic (may show as reverse on some terminals)
  styleUnderscore,        ## underlined text
  styleBlink,             ## blinking text
  styleBlinkRapid,        ## rapid blinking (not widely supported)
  styleReverse,           ## swap foreground and background colors
  styleHidden,            ## hidden/invisible text
  styleStrikethrough      ## strikethrough text
```

Enum of text style modifiers for terminal output. Used with `setStyle`, `styledWrite`, `styledWriteLine`, and `styledEcho`.

---

### `ForegroundColor`

```nim
type ForegroundColor* = enum
  fgBlack = 30,   ## black
  fgRed,          ## red
  fgGreen,        ## green
  fgYellow,       ## yellow
  fgBlue,         ## blue
  fgMagenta,      ## magenta
  fgCyan,         ## cyan
  fgWhite,        ## white
  fg8Bit,         ## 256-color (not supported — use `enableTrueColors` instead)
  fgDefault       ## reset to terminal's default foreground color
```

Standard 8-color foreground color enum. Values are ANSI color codes (30–39).

---

### `BackgroundColor`

```nim
type BackgroundColor* = enum
  bgBlack = 40,   ## black
  bgRed,          ## red
  bgGreen,        ## green
  bgYellow,       ## yellow
  bgBlue,         ## blue
  bgMagenta,      ## magenta
  bgCyan,         ## cyan
  bgWhite,        ## white
  bg8Bit,         ## 256-color (not supported — use `enableTrueColors` instead)
  bgDefault       ## reset to terminal's default background color
```

Standard 8-color background color enum. Values are ANSI color codes (40–49).

---

### `TerminalCmd`

```nim
type TerminalCmd* = enum
  resetStyle,   ## reset all text attributes
  fgColor,      ## mark that next Color argument is a foreground true color
  bgColor       ## mark that next Color argument is a background true color
```

Special command tokens used as arguments in `styledWrite`, `styledWriteLine`, and `styledEcho` to control behavior inline. `fgColor` and `bgColor` act as "mode selectors" before a `Color` value argument.

**Example:**
```nim
import std/terminal
styledEcho styleBright, fgGreen, "[OK]", resetStyle, " Done"
```

---

## Terminal Size

### `terminalWidth`

```nim
proc terminalWidth*(): int
```

Returns the terminal width in character columns. Falls back gracefully:

- **POSIX:** checks `$COLUMNS` env var → `ioctl` on stdin/stdout/stderr → controlling terminal → defaults to `80`
- **Windows:** queries `GetConsoleScreenBufferInfo` on stdin/stdout/stderr → defaults to `80`

**Returns:** Width in columns. Never returns 0 (minimum 80).

**Example:**
```nim
import std/terminal
echo "Terminal width: ", terminalWidth(), " columns"
```

---

### `terminalHeight`

```nim
proc terminalHeight*(): int
```

Returns the terminal height in rows. Falls back gracefully:

- **POSIX:** checks `$LINES` env var → `ioctl` on stdin/stdout/stderr → controlling terminal → defaults to `0`
- **Windows:** queries `GetConsoleScreenBufferInfo` → defaults to `0`

**Returns:** Height in rows. Returns `0` if undetermined.

**Example:**
```nim
import std/terminal
let h = terminalHeight()
if h > 0:
  echo "Terminal height: ", h, " rows"
else:
  echo "Could not determine height"
```

---

### `terminalSize`

```nim
proc terminalSize*(): tuple[w, h: int]
```

Returns both terminal width and height as a named tuple. Convenience wrapper combining `terminalWidth()` and `terminalHeight()`.

**Returns:** `(w: int, h: int)` — width and height.

**Example:**
```nim
import std/terminal
let (w, h) = terminalSize()
echo "Size: ", w, "×", h

# Responsive layout:
let halfWidth = terminalWidth() div 2
```

---

### `terminalWidthIoctl`

```nim
# POSIX:
proc terminalWidthIoctl*(fds: openArray[int]): int
# Windows:
proc terminalWidthIoctl*(handles: openArray[Handle]): int
```

Low-level helper that queries terminal width using `ioctl` (POSIX) or `GetConsoleScreenBufferInfo` (Windows) on the given file descriptors/handles. Returns `0` if none respond.

**Example:**
```nim
import std/terminal
# Try specific file descriptors:
let w = terminalWidthIoctl([1, 2])  # stdout and stderr
echo "Width: ", w
```

---

### `terminalHeightIoctl`

```nim
# POSIX:
proc terminalHeightIoctl*(fds: openArray[int]): int
# Windows:
proc terminalHeightIoctl*(handles: openArray[Handle]): int
```

Low-level helper that queries terminal height from specific file descriptors/handles. Returns `0` if none respond.

**Example:**
```nim
import std/terminal
let h = terminalHeightIoctl([0, 1, 2])
echo "Height: ", h
```

---

## Cursor Position

### `getCursorPos`

```nim
proc getCursorPos*(): tuple[x, y: int]
  {.raises: [ValueError, IOError, OSError].}
```

Returns the current cursor position as a `(x, y)` tuple (0-indexed).

- **POSIX:** writes the ANSI `\e[6n` escape to stdout and reads the response from stdin using raw mode. Requires a real terminal (not a pipe).
- **Windows:** calls `GetConsoleScreenBufferInfo`.

**Returns:** `(x, y)` — column and row of cursor (0-based).

**Raises:**
- `ValueError` — if the terminal response is malformed or incomplete
- `IOError` — on I/O failure
- `OSError` — on Windows API failure

**Example:**
```nim
import std/terminal
let (x, y) = getCursorPos()
echo "Cursor at column ", x, ", row ", y
```

---

### `setCursorPos`

```nim
proc setCursorPos*(f: File, x, y: int)
template setCursorPos*(x, y: int)    # stdout shorthand
```

Moves the cursor to absolute position `(x, y)`. Origin `(0, 0)` is the upper-left corner of the terminal.

**Parameters:**
- `f` — output file (e.g. `stdout`, `stderr`)
- `x` — column (0-based)
- `y` — row (0-based)

**Example:**
```nim
import std/terminal
setCursorPos(0, 0)          # move to top-left (stdout)
stdout.setCursorPos(10, 5)  # move to column 10, row 5
```

---

### `setCursorXPos`

```nim
proc setCursorXPos*(f: File, x: int)
template setCursorXPos*(x: int)    # stdout shorthand
```

Moves the cursor to column `x` on the current row. The row (Y) position is unchanged.

**Parameters:**
- `f` — output file
- `x` — column (0-based)

**Example:**
```nim
import std/terminal
stdout.setCursorXPos(0)   # move to beginning of current line
setCursorXPos(20)         # move to column 20 on stdout
```

---

### `setCursorYPos` *(Windows only)*

```nim
proc setCursorYPos*(f: File, y: int)         # Windows only
template setCursorYPos*(x: int)              # stdout shorthand, Windows only
```

Moves the cursor to row `y` while keeping the column position. **Windows only** — not supported on UNIX.

**Parameters:**
- `f` — output file
- `y` — row (0-based)

**Example:**
```nim
import std/terminal
when defined(windows):
  stdout.setCursorYPos(10)
```

---

## Cursor Visibility

### `hideCursor`

```nim
proc hideCursor*(f: File)
template hideCursor*()    # stdout shorthand
```

Makes the terminal cursor invisible. The cursor still moves but is not drawn.

> **Important:** Always restore cursor visibility before exiting. Use `showCursor` or register `showCursor` with `exitprocs.addExitProc`.

**Example:**
```nim
import std/[terminal, exitprocs]
hideCursor()
addExitProc(showCursor)   # restore on exit

# ... draw animation ...

showCursor()
```

---

### `showCursor`

```nim
proc showCursor*(f: File)
template showCursor*()    # stdout shorthand
```

Makes the terminal cursor visible again after it was hidden.

**Example:**
```nim
import std/terminal
hideCursor()
# ... do work ...
showCursor()
```

---

## Cursor Movement

All movement procs have two forms: one accepting a `File` for explicit output target, and one shorthand template that defaults to `stdout`.

### `cursorUp`

```nim
proc cursorUp*(f: File, count = 1)
template cursorUp*(count = 1)    # stdout shorthand
```

Moves the cursor up by `count` rows. The column position is unchanged.

**Parameters:**
- `count` — number of rows to move up (default: 1)

**Example:**
```nim
import std/terminal
stdout.cursorUp(3)         # move 3 rows up
cursorUp()                 # move 1 row up on stdout
```

---

### `cursorDown`

```nim
proc cursorDown*(f: File, count = 1)
template cursorDown*(count = 1)    # stdout shorthand
```

Moves the cursor down by `count` rows.

**Parameters:**
- `count` — number of rows to move down (default: 1)

**Example:**
```nim
import std/terminal
cursorDown(2)
write(stdout, "Two rows below original position")
```

---

### `cursorForward`

```nim
proc cursorForward*(f: File, count = 1)
template cursorForward*(count = 1)    # stdout shorthand
```

Moves the cursor forward (right) by `count` columns.

**Parameters:**
- `count` — number of columns to move right (default: 1)

**Example:**
```nim
import std/terminal
write(stdout, ">>")
cursorForward(5)
write(stdout, "<<")    # printed 5 spaces to the right
```

---

### `cursorBackward`

```nim
proc cursorBackward*(f: File, count = 1)
template cursorBackward*(count = 1)    # stdout shorthand
```

Moves the cursor backward (left) by `count` columns.

**Parameters:**
- `count` — number of columns to move left (default: 1)

**Example:**
```nim
import std/terminal
write(stdout, "Hello!")
cursorBackward(6)         # go back 6 columns
write(stdout, "World!")   # overwrite "Hello!"
```

---

## Screen & Line Erasure

### `eraseLine`

```nim
proc eraseLine*(f: File)
template eraseLine*()    # stdout shorthand
```

Erases the entire current line (from column 0 to end) and moves the cursor to the beginning of the line (`x = 0`). Useful in progress bar and spinner patterns.

**Example:**
```nim
import std/[terminal, os]
for i in 1..10:
  write(stdout, "Progress: " & $i & "/10")
  stdout.flushFile()
  sleep(200)
  eraseLine()
write(stdout, "Done!\n")
```

---

### `eraseScreen`

```nim
proc eraseScreen*(f: File)
template eraseScreen*()    # stdout shorthand
```

Clears the entire screen using the current background color and moves the cursor to the home position (upper-left).

**Example:**
```nim
import std/terminal
eraseScreen()
setCursorPos(0, 0)
write(stdout, "Fresh screen!")
```

---

## Text Styles

### `setStyle`

```nim
proc setStyle*(f: File, style: set[Style])
template setStyle*(style: set[Style])    # stdout shorthand
```

Applies one or more text styles to `f`. The effect persists until overridden or `resetAttributes` is called.

**Parameters:**
- `style` — a `set[Style]` (can combine multiple styles)

> **Note:** On Windows, only `styleBright`, `styleBlink`, `styleReverse`, and `styleUnderscore` are supported; the rest are silently ignored.

**Example:**
```nim
import std/terminal
stdout.setStyle({styleBright, styleUnderscore})
write(stdout, "Bold and underlined")
resetAttributes()

setStyle({styleItalic, styleBlink})
write(stdout, "Italic blinking")
resetAttributes()
```

---

### `writeStyled`

```nim
proc writeStyled*(txt: string, style: set[Style] = {styleBright})
```

Writes `txt` to stdout with the given `style`, then **automatically restores** the previous style. Unlike `setStyle`, this is safe to call without a subsequent `resetAttributes`.

**Parameters:**
- `txt` — string to write
- `style` — styles to apply (default: `{styleBright}`)

**Example:**
```nim
import std/terminal
write(stdout, "Normal → ")
writeStyled("Bold Text", {styleBright})
write(stdout, " → Normal again\n")

writeStyled("[ERROR]", {styleBright, styleReverse})
```

---

### `resetAttributes` (for File)

```nim
proc resetAttributes*(f: File)
```

Resets all text attributes (color, brightness, style) on file `f` to terminal defaults. On POSIX, writes `ansiResetCode`. On Windows, restores the original console attribute word captured at startup.

**Example:**
```nim
import std/terminal
stdout.setForegroundColor(fgRed)
write(stdout, "Red text")
stdout.resetAttributes()
write(stdout, "Normal text\n")
```

---

### `resetAttributes` (stdout)

```nim
proc resetAttributes*() {.noconv.}
```

Resets all attributes on `stdout`. Recommended to register as an exit proc:

```nim
import std/[terminal, exitprocs]
addExitProc(resetAttributes)
```

**Example:**
```nim
import std/terminal
setForegroundColor(fgBlue)
write(stdout, "Blue text")
resetAttributes()
```

---

### `ansiStyleCode`

```nim
proc ansiStyleCode*(style: int): string
template ansiStyleCode*(style: Style): string
template ansiStyleCode*(style: static[Style]): string   # compile-time optimized
```

Returns the raw ANSI escape string for a given style code or `Style` enum value. Useful for embedding style codes directly in strings.

**Example:**
```nim
import std/terminal
echo ansiStyleCode(styleBright)          # "\e[1m"
echo ansiStyleCode(styleUnderscore)      # "\e[4m"
echo ansiStyleCode(1)                    # "\e[1m"
```

---

## Colors

### `setForegroundColor` (named color)

```nim
proc setForegroundColor*(f: File, fg: ForegroundColor, bright = false)
template setForegroundColor*(fg: ForegroundColor, bright = false)  # stdout
```

Sets the terminal's foreground (text) color. The effect persists until `resetAttributes` is called.

**Parameters:**
- `f` — output file
- `fg` — a `ForegroundColor` enum value
- `bright` — if `true`, uses the bright/high-intensity variant (adds 60 to ANSI code)

**Example:**
```nim
import std/terminal
stdout.setForegroundColor(fgRed)
write(stdout, "Red text\n")

stdout.setForegroundColor(fgGreen, bright = true)
write(stdout, "Bright green text\n")

stdout.setForegroundColor(fgDefault)    # restore default
```

---

### `setBackgroundColor` (named color)

```nim
proc setBackgroundColor*(f: File, bg: BackgroundColor, bright = false)
template setBackgroundColor*(bg: BackgroundColor, bright = false)  # stdout
```

Sets the terminal's background color. The effect persists until `resetAttributes`.

**Parameters:**
- `f` — output file
- `bg` — a `BackgroundColor` enum value
- `bright` — if `true`, uses the bright/high-intensity variant

**Example:**
```nim
import std/terminal
stdout.setBackgroundColor(bgBlue)
stdout.setForegroundColor(fgWhite)
write(stdout, "White on Blue\n")
stdout.resetAttributes()

setBackgroundColor(bgYellow, bright = true)
write(stdout, "Text on bright yellow\n")
resetAttributes()
```

---

### `setForegroundColor` (true color)

```nim
proc setForegroundColor*(f: File, color: Color)
template setForegroundColor*(color: Color)    # stdout shorthand
```

Sets the foreground color to an RGB true color (`Color` from `std/colors`). Only works after `enableTrueColors()` is called and the terminal supports it.

**Parameters:**
- `color` — a `Color` value from `std/colors` (e.g. `colRed`, `rgb(255, 128, 0)`)

**Example:**
```nim
import std/[terminal, colors]
enableTrueColors()
stdout.setForegroundColor(colDodgerBlue)
write(stdout, "Dodger blue text\n")
stdout.resetAttributes()
disableTrueColors()
```

---

### `setBackgroundColor` (true color)

```nim
proc setBackgroundColor*(f: File, color: Color)
template setBackgroundColor*(color: Color)    # stdout shorthand
```

Sets the background color to an RGB true color. Only works after `enableTrueColors()`.

**Example:**
```nim
import std/[terminal, colors]
enableTrueColors()
stdout.setBackgroundColor(colLimeGreen)
write(stdout, "Text on lime green background\n")
stdout.resetAttributes()
```

---

## True Color (24-bit)

### `enableTrueColors`

```nim
proc enableTrueColors*()
```

Enables 24-bit true color (RGB) support for the terminal. Must be called before using true color variants of `setForegroundColor` and `setBackgroundColor`.

- **POSIX:** checks the `$COLORTERM` environment variable for `"truecolor"` or `"24bit"`.
- **Windows:** checks OS version (≥ Windows 10 build 10586) or `$ANSICON_DEF`, then enables `ENABLE_VIRTUAL_TERMINAL_PROCESSING`.

**Example:**
```nim
import std/[terminal, colors]
enableTrueColors()
if isTrueColorSupported():
  stdout.setForegroundColor(rgb(255, 128, 0))
  write(stdout, "Orange text via true color!\n")
  stdout.resetAttributes()
disableTrueColors()
```

---

### `disableTrueColors`

```nim
proc disableTrueColors*()
```

Disables 24-bit true color support. On Windows, removes `ENABLE_VIRTUAL_TERMINAL_PROCESSING` from the console mode flags.

**Example:**
```nim
import std/terminal
enableTrueColors()
# ... colored output ...
disableTrueColors()
```

---

### `isTrueColorSupported`

```nim
proc isTrueColorSupported*(): bool
```

Returns `true` if the terminal supports true color **and** `enableTrueColors()` has been called. Returns `false` before `enableTrueColors()` is called even on supporting terminals.

**Example:**
```nim
import std/[terminal, colors]
enableTrueColors()
if isTrueColorSupported():
  echo "True color is available!"
  stdout.setForegroundColor(colDeepPink)
  write(stdout, "Deep pink!\n")
  stdout.resetAttributes()
else:
  echo "Falling back to 8-color mode"
  stdout.setForegroundColor(fgRed)
  write(stdout, "Red\n")
  stdout.resetAttributes()
```

---

## ANSI Code Helpers

These procs return raw ANSI escape strings without writing them to any file. Useful for building styled strings, logging, or non-terminal output.

### `ansiForegroundColorCode` (named color)

```nim
proc ansiForegroundColorCode*(fg: ForegroundColor, bright = false): string
template ansiForegroundColorCode*(fg: static[ForegroundColor],
                                  bright: static[bool] = false): string
```

Returns the ANSI escape string for the given foreground color.

**Example:**
```nim
import std/terminal
echo ansiForegroundColorCode(fgRed)             # "\e[31m"
echo ansiForegroundColorCode(fgGreen, bright=true)  # "\e[92m"

# Build a colored string:
let s = ansiForegroundColorCode(fgBlue) & "Blue text" & ansiResetCode
echo s
```

---

### `ansiForegroundColorCode` (true color)

```nim
proc ansiForegroundColorCode*(color: Color): string
template ansiForegroundColorCode*(color: static[Color]): string
```

Returns the ANSI true-color (24-bit) escape string for `color` as a foreground color code. Format: `\e[38;2;R;G;Bm`.

**Example:**
```nim
import std/[terminal, colors]
let code = ansiForegroundColorCode(colCornflowerBlue)
echo code & "Cornflower blue" & ansiResetCode
```

---

### `ansiBackgroundColorCode` (true color)

```nim
proc ansiBackgroundColorCode*(color: Color): string
template ansiBackgroundColorCode*(color: static[Color]): string
```

Returns the ANSI true-color escape string for `color` as a background color code. Format: `\e[48;2;R;G;Bm`.

**Example:**
```nim
import std/[terminal, colors]
let bgCode = ansiBackgroundColorCode(colGold)
let fgCode = ansiForegroundColorCode(fgBlack)
echo bgCode & fgCode & "Black on gold" & ansiResetCode
```

---

## Styled Output

### `styledWrite`

```nim
macro styledWrite*(f: File, m: varargs[typed]): untyped
```

A macro similar to `write`, but understands terminal style arguments. Non-string arguments of type `Style`, `set[Style]`, `ForegroundColor`, `BackgroundColor`, `Color`, or `TerminalCmd` are translated into the appropriate terminal operations instead of being written as text.

- Style arguments **before** a string affect only that immediately following string.
- After the macro call, attributes are **automatically reset**.

**Accepted argument types:**
| Type | Effect |
|------|--------|
| `string` | Written to `f` directly |
| `Style` | Calls `setStyle(f, {style})` |
| `set[Style]` | Calls `setStyle(f, style)` |
| `ForegroundColor` | Calls `setForegroundColor(f, color)` |
| `BackgroundColor` | Calls `setBackgroundColor(f, color)` |
| `Color` | Sets true foreground or background color (based on preceding `fgColor`/`bgColor` cmd) |
| `TerminalCmd` | `resetStyle` resets attrs; `fgColor`/`bgColor` set mode for next `Color` |

**Example:**
```nim
import std/terminal
stdout.styledWrite(fgRed, "Error: ", resetStyle, "something went wrong")
stdout.styledWrite({styleBright, styleUnderscore}, "Important!")
stdout.styledWrite(fgGreen, "green ", fgBlue, "blue ", "no color")
```

---

### `styledWriteLine`

```nim
template styledWriteLine*(f: File, args: varargs[untyped])
```

Calls `styledWrite(f, args)` and appends a newline `"\n"` at the end. The most commonly used styled output procedure.

**Example:**
```nim
import std/terminal

proc error(msg: string) =
  styledWriteLine(stderr, fgRed, "Error: ", resetStyle, msg)

proc success(msg: string) =
  styledWriteLine(stdout, fgGreen, styleBright, "✓ ", resetStyle, msg)

error("File not found")
success("Build complete")

stdout.styledWriteLine(bgCyan, fgBlack, "  INFO  ", resetStyle, " Application started")
```

---

### `styledEcho`

```nim
template styledEcho*(args: varargs[untyped])
```

Convenience shorthand for `stdout.styledWriteLine(args)`. Echoes styled output to stdout with an automatic newline.

**Example:**
```nim
import std/terminal

# Multiple styles in one call:
styledEcho styleBright, fgGreen, "[PASS]", resetStyle, fgGreen, " Test passed!"
styledEcho fgRed, "[FAIL]", resetStyle, " Test failed"
styledEcho {styleBright, styleUnderscore}, "Section Header"

# Progress bar:
styledEcho fgYellow, "Loading: ", fgGreen, "████████░░", resetStyle, " 80%"
```

---

## Input

### `getch`

```nim
proc getch*(): char
```

Reads a single character from the terminal without echoing it to the screen. Blocks until a key is pressed. Temporarily sets raw mode on POSIX.

**Returns:** The character corresponding to the key pressed.

**Platform behavior:**
- **POSIX:** saves terminal mode, sets raw mode, reads one byte, restores mode.
- **Windows:** uses `ReadConsoleInput` to wait for a key-down event.

> **Note:** Special keys (arrows, F-keys, etc.) generate multiple bytes on POSIX (escape sequences). Only the first byte is returned by `getch`.

**Example:**
```nim
import std/terminal
write(stdout, "Press any key to continue...")
let ch = getch()
echo "\nYou pressed: ", ch.repr

# Menu navigation:
write(stdout, "Press 'q' to quit, 'h' for help: ")
let key = getch()
case key
of 'q': echo "\nQuitting..."
of 'h': echo "\nHelp: ..."
else: echo "\nUnknown key"
```

---

### `readPasswordFromStdin` (with var string)

```nim
proc readPasswordFromStdin*(prompt: string, password: var string): bool
  {.tags: [ReadIOEffect, WriteIOEffect].}
```

Reads a password from stdin without echoing it. Displays `prompt`, reads input, then restores echo mode.

**Parameters:**
- `prompt` — prompt string displayed to the user
- `password` — output variable where the read password is stored (must not be nil)

**Returns:** `true` on success; `false` at end-of-file.

**Platform behavior:**
- **Windows:** uses `CONIN$` with `ENABLE_ECHO_INPUT` flag cleared.
- **POSIX:** clears the `ECHO` flag in termios settings.

**Example:**
```nim
import std/terminal
var pass: string
let ok = readPasswordFromStdin("Enter password: ", pass)
if ok:
  echo "Got password of length: ", pass.len
else:
  echo "EOF reached"
```

---

### `readPasswordFromStdin` (returns string)

```nim
proc readPasswordFromStdin*(prompt = "password: "): string
```

Convenience overload that returns the password as a string directly.

**Parameters:**
- `prompt` — prompt string (default: `"password: "`)

**Returns:** The entered password string.

**Example:**
```nim
import std/terminal
let password = readPasswordFromStdin()
let adminPass = readPasswordFromStdin("Admin password: ")

if password == "secret":
  echo "Access granted"
```

---

## Utility

### `isatty`

```nim
proc isatty*(f: File): bool
```

Returns `true` if `f` is connected to a real terminal device (TTY), `false` if it is a pipe, file redirect, or other non-terminal stream.

**Platform:** Uses POSIX `isatty()` or Windows `_isatty()`.

**Example:**
```nim
import std/terminal

if isatty(stdout):
  echo "Running in a terminal — colors enabled"
  setForegroundColor(fgGreen)
  write(stdout, "Colored output\n")
  resetAttributes()
else:
  echo "Output is redirected — skipping colors"
```

---

## stdout Shorthand Templates

The module provides convenient `template` wrappers for all major procs, defaulting their output to `stdout`. This means you can call them without an explicit `f` argument:

| Shorthand | Equivalent |
|-----------|------------|
| `hideCursor()` | `hideCursor(stdout)` |
| `showCursor()` | `showCursor(stdout)` |
| `setCursorPos(x, y)` | `setCursorPos(stdout, x, y)` |
| `setCursorXPos(x)` | `setCursorXPos(stdout, x)` |
| `setCursorYPos(y)` *(Windows)* | `setCursorYPos(stdout, y)` |
| `cursorUp(n)` | `cursorUp(stdout, n)` |
| `cursorDown(n)` | `cursorDown(stdout, n)` |
| `cursorForward(n)` | `cursorForward(stdout, n)` |
| `cursorBackward(n)` | `cursorBackward(stdout, n)` |
| `eraseLine()` | `eraseLine(stdout)` |
| `eraseScreen()` | `eraseScreen(stdout)` |
| `setStyle(s)` | `setStyle(stdout, s)` |
| `setForegroundColor(fg)` | `setForegroundColor(stdout, fg)` |
| `setBackgroundColor(bg)` | `setBackgroundColor(stdout, bg)` |
| `setForegroundColor(c)` | `setForegroundColor(stdout, c)` |
| `setBackgroundColor(c)` | `setBackgroundColor(stdout, c)` |
| `resetAttributes()` | `resetAttributes(stdout)` |
| `styledEcho(...)` | `stdout.styledWriteLine(...)` |

---

## Platform Availability Summary

| Feature / Proc | Linux | macOS | Windows | Notes |
|---------------|:-----:|:-----:|:-------:|-------|
| `terminalWidth` | ✓ | ✓ | ✓ | |
| `terminalHeight` | ✓ | ✓ | ✓ | |
| `terminalSize` | ✓ | ✓ | ✓ | |
| `terminalWidthIoctl` | ✓ | ✓ | ✓ | Different signature |
| `terminalHeightIoctl` | ✓ | ✓ | ✓ | Different signature |
| `getCursorPos` | ✓ | ✓ | ✓ | POSIX needs real TTY |
| `setCursorPos` | ✓ | ✓ | ✓ | |
| `setCursorXPos` | ✓ | ✓ | ✓ | |
| `setCursorYPos` | ✗ | ✗ | ✓ | Windows only |
| `hideCursor` | ✓ | ✓ | ✓ | |
| `showCursor` | ✓ | ✓ | ✓ | |
| `cursorUp` | ✓ | ✓ | ✓ | |
| `cursorDown` | ✓ | ✓ | ✓ | |
| `cursorForward` | ✓ | ✓ | ✓ | |
| `cursorBackward` | ✓ | ✓ | ✓ | |
| `eraseLine` | ✓ | ✓ | ✓ | |
| `eraseScreen` | ✓ | ✓ | ✓ | |
| `setStyle` | ✓ | ✓ | ✓ | Limited styles on Windows |
| `writeStyled` | ✓ | ✓ | ✓ | |
| `resetAttributes` | ✓ | ✓ | ✓ | |
| `ansiStyleCode` | ✓ | ✓ | ✓ | |
| `setForegroundColor` | ✓ | ✓ | ✓ | |
| `setBackgroundColor` | ✓ | ✓ | ✓ | |
| `setForegroundColor(Color)` | ✓ | ✓ | ✓ | Needs `enableTrueColors` |
| `setBackgroundColor(Color)` | ✓ | ✓ | ✓ | Needs `enableTrueColors` |
| `ansiForegroundColorCode` | ✓ | ✓ | ✓ | |
| `ansiBackgroundColorCode` | ✓ | ✓ | ✓ | |
| `enableTrueColors` | ✓ | ✓ | ✓ | Win10+ build 10586 |
| `disableTrueColors` | ✓ | ✓ | ✓ | |
| `isTrueColorSupported` | ✓ | ✓ | ✓ | |
| `styledWrite` | ✓ | ✓ | ✓ | |
| `styledWriteLine` | ✓ | ✓ | ✓ | |
| `styledEcho` | ✓ | ✓ | ✓ | |
| `getch` | ✓ | ✓ | ✓ | |
| `readPasswordFromStdin` | ✓ | ✓ | ✓ | |
| `isatty` | ✓ | ✓ | ✓ | |
| `ansiResetCode` | ✓ | ✓ | ✓ | const |

---

## Complete Examples

### Progress Bar

```nim
import std/[terminal, os, strutils]

proc progressBar(value, total: int) =
  let pct = value * 100 div total
  let filled = value * 40 div total
  let bar = "#".repeat(filled) & "-".repeat(40 - filled)

  eraseLine()
  stdout.styledWrite(fgCyan, "[", fgGreen, bar, fgCyan, "] ",
                     fgYellow, $pct & "%", resetStyle)
  stdout.flushFile()

hideCursor()
addExitProc(proc() = showCursor(); resetAttributes())

for i in 1..100:
  progressBar(i, 100)
  sleep(30)
  if i < 100: cursorUp()

echo ""
styledEcho fgGreen, styleBright, "Complete!"
```

---

### Colored Logger

```nim
import std/terminal

type LogLevel = enum DEBUG, INFO, WARN, ERROR

proc log(level: LogLevel, msg: string) =
  let (label, color) = case level
    of DEBUG: ("[DEBUG]", fgCyan)
    of INFO:  ("[INFO] ", fgGreen)
    of WARN:  ("[WARN] ", fgYellow)
    of ERROR: ("[ERROR]", fgRed)

  if isatty(stderr):
    stderr.styledWriteLine(color, styleBright, label, resetStyle, " ", msg)
  else:
    stderr.writeLine(label & " " & msg)

log(INFO, "Server started on port 8080")
log(WARN, "Memory usage above 80%")
log(ERROR, "Connection refused")
```

---

### Text Editor Status Bar

```nim
import std/terminal

proc drawStatusBar(filename: string, line, col, total: int) =
  let w = terminalWidth()
  let (h, _) = (terminalHeight(), 0)

  # Save position and move to last line:
  setCursorPos(0, h - 1)
  stdout.setBackgroundColor(bgWhite)
  stdout.setForegroundColor(fgBlack)

  let info = " " & filename & "  Ln " & $line & ", Col " & $col &
             "  " & $total & " lines "
  let padding = " ".repeat(max(0, w - info.len))
  write(stdout, info & padding)
  resetAttributes()

drawStatusBar("main.nim", 42, 15, 300)
```

---

### Interactive Menu

```nim
import std/terminal

proc menu(items: seq[string]): int =
  hideCursor()
  var selected = 0
  while true:
    for i, item in items:
      eraseLine()
      if i == selected:
        stdout.styledWriteLine(fgBlack, bgWhite, " > " & item & " ", resetStyle)
      else:
        stdout.writeLine("   " & item)
    cursorUp(items.len)

    let key = getch()
    case key
    of 'k', 'A': selected = max(0, selected - 1)          # up
    of 'j', 'B': selected = min(items.high, selected + 1)  # down
    of '\r', '\n':
      cursorDown(items.len)
      showCursor()
      return selected
    else: discard

let choice = menu(@["New File", "Open File", "Settings", "Quit"])
echo "You chose: ", choice
```

---

### True Color Gradient

```nim
import std/[terminal, colors]

enableTrueColors()
if isTrueColorSupported():
  let w = terminalWidth()
  for x in 0..<w:
    let r = x * 255 div w
    let b = 255 - r
    stdout.setBackgroundColor(rgb(r.uint8, 0, b.uint8))
    write(stdout, " ")
  resetAttributes()
  echo ""
else:
  echo "True color not supported"
disableTrueColors()
```

---

*Generated from Nim's standard library `std/terminal` module.*
