# `strmisc` Module Reference

> **Module:** `std/strmisc`  
> **Language:** Nim  
> **Purpose:** A collection of less common but useful string utility routines, complementing the standard `strutils` module.

---

## Overview

The `strmisc` module provides three exported functions designed for specific string manipulation tasks that don't appear frequently enough to warrant inclusion in the main `strutils` module, but are nonetheless genuinely useful when you need them:

- **`expandTabs`** — replaces tab characters with spaces, respecting column alignment
- **`partition`** — splits a string into three parts around a separator
- **`rpartition`** — same as `partition`, but searches from the right side of the string

All three functions are pure (`func`), meaning they have no side effects and always produce the same output for the same input.

---

## `expandTabs`

```nim
func expandTabs*(s: string, tabSize: int = 8): string
```

### What it does

This function scans through a string and replaces every tab character (`\t`) with the appropriate number of spaces needed to reach the *next tab stop*. The result is a string that looks visually identical to the original when rendered in a fixed-width font, but uses only spaces for indentation.

The key concept here is **tab stops**: positions in a line that are multiples of `tabSize`. When the function encounters a tab, it doesn't blindly insert `tabSize` spaces — instead it inserts *just enough* spaces to advance the current column position to the next tab stop. This is exactly how a text editor or terminal handles tabs.

### How column tracking works

- A counter `pos` tracks the current column number, starting at `0`.
- Every character (including inserted spaces) increments `pos` by 1.
- A newline (`\n`) resets `pos` back to `0`, starting a fresh line.
- When a tab is hit: `spaces_to_insert = tabSize - (pos mod tabSize)`

This means a tab near the beginning of a line inserts more spaces than a tab that's already partway through a tab interval.

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `s` | `string` | — | The input string, may contain `\t` and `\n` |
| `tabSize` | `int` | `8` | The width between tab stops. If `≤ 0`, treated as `1` |

### Return value

A new string with all `\t` characters replaced by spaces. The original string is not modified.

### Examples

```nim
import std/strmisc

# A lone tab at column 0 with tabSize=4 → 4 spaces
doAssert expandTabs("\t", 4) == "    "

# Tab after "foo" (3 chars): next stop at 4, so 1 space
doAssert expandTabs("\tfoo\t", 4) == "    foo "
#                   ^^^^         4 spaces (col 0 → col 4)
#                       ^^^  " " 1 space  (col 3 → col 4 after "foo")

# Multi-line: newline resets column counter
doAssert expandTabs("a\tb\n\txy\t", 3) == "a  b\n   xy "
# Line 1: "a" is at col 1, tab → 2 spaces to reach col 3; "b" at col 3
# Line 2: col resets; tab → 3 spaces; "xy" at col 3; tab → 1 space to col 6? No — col 5, next stop at 6 → 1 space

# Default tabSize is 8
doAssert expandTabs("a\tb") == "a       b"
#                              "a" + 7 spaces + "b"
```

### Common use case

Normalising source code or configuration files that mix tabs and spaces before further processing, diffing, or display.

---

## `partition`

```nim
func partition*(s: string, sep: string, right: bool = false): (string, string, string)
```

### What it does

`partition` searches for the separator string `sep` inside `s` and splits the string into **exactly three parts**: the portion before the separator, the separator itself, and the portion after the separator. The result is always a 3-tuple, whether or not `sep` was found.

This is analogous to Python's `str.partition()` and `str.rpartition()` methods, unified into one function via the `right` flag.

### Why a 3-tuple instead of just splitting?

The key design choice is that the separator is preserved in the middle of the tuple. This makes it easy to tell *whether* the separator was found at all (the middle element will be empty if not found), and to reconstruct the original string if needed: `before & sep & after == s`.

### Search direction

- `right = false` (default): finds the **first** occurrence of `sep` (left-to-right search)
- `right = true`: finds the **last** occurrence of `sep` (right-to-left search)

### When `sep` is not found

| `right` value | Return value |
|--------------|--------------|
| `false` | `(s, "", "")` — the whole string goes in the first slot |
| `true` | `("", "", s)` — the whole string goes in the third slot |

The middle element being empty is the reliable signal that the separator was not present.

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `s` | `string` | — | The string to split |
| `sep` | `string` | — | The separator to search for |
| `right` | `bool` | `false` | If `true`, search from the right (last occurrence) |

### Return value

A tuple `(beforeSep, sep, afterSep)`. All three elements are strings.

### Examples

```nim
import std/strmisc

# Basic split on first ":"
doAssert partition("foo:bar:baz", ":") == ("foo", ":", "bar:baz")
#                                           ^^^   sep   ^^^^^^^
#                                           before       after

# Split on last ":" using right = true
doAssert partition("foo:bar:baz", ":", right = true) == ("foo:bar", ":", "baz")

# Separator not found (right = false): whole string in first slot, middle is ""
doAssert partition("foobar", ":") == ("foobar", "", "")

# Separator not found (right = true): whole string in third slot, middle is ""
doAssert partition("foobar", ":", right = true) == ("", "", "foobar")

# Practical: parse "key=value" config lines
let line = "host=localhost"
let (key, _, value) = partition(line, "=")
echo key    # "host"
echo value  # "localhost"

# Detect whether sep was found
let (_, sep, _) = partition("no-separator-here", "=")
if sep == "":
  echo "separator not found"
```

### Common use cases

- Parsing `key=value` or `key:value` configuration lines
- Extracting the file extension from a path by partitioning on the last `.`
- Splitting URLs into protocol and the rest by partitioning on `://`

---

## `rpartition`

```nim
func rpartition*(s: string, sep: string): (string, string, string)
```

### What it does

`rpartition` is a convenience wrapper around `partition(s, sep, right = true)`. It searches for the **last** occurrence of `sep` in `s` and returns the same 3-tuple format.

Having a separate function makes code more readable when you specifically want a right-to-left search — the intent is immediately clear without needing to pass a named argument.

### When `sep` is not found

Returns `("", "", s)` — the whole string ends up in the third slot, and the middle is empty.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `s` | `string` | The string to split |
| `sep` | `string` | The separator to search for (last occurrence) |

### Return value

A tuple `(beforeSep, sep, afterSep)`.

### Examples

```nim
import std/strmisc

# Finds the LAST ":", splitting off "baz"
doAssert rpartition("foo:bar:baz", ":") == ("foo:bar", ":", "baz")

# Not found: everything in the third slot
doAssert rpartition("foobar", ":") == ("", "", "foobar")

# Practical: get the file extension (last ".")
let filename = "archive.tar.gz"
let (base, _, ext) = rpartition(filename, ".")
echo base  # "archive.tar"
echo ext   # "gz"

# Practical: get the last segment of a URL path
let url = "https://example.com/docs/api/reference"
let (_, _, page) = rpartition(url, "/")
echo page  # "reference"
```

### `rpartition` vs `partition` at a glance

```
Input: "foo:bar:baz"   sep: ":"

partition  → ("foo",     ":", "bar:baz")   ← first ":"
rpartition → ("foo:bar", ":", "baz")       ← last  ":"
```

### Common use cases

- Extracting the final file extension (`.gz`, `.nim`, etc.)
- Getting the last path segment from a URL or filesystem path
- Any scenario where you care about the *rightmost* delimiter

---

## Summary Table

| Function | Searches from | Not found (before) | Not found (after) | Key use |
|----------|--------------|-------------------|-------------------|---------|
| `expandTabs(s, tabSize)` | — | — | — | Tabs → spaces with correct alignment |
| `partition(s, sep)` | Left (first) | `(s, "", "")` | — | Split on first separator |
| `partition(s, sep, right=true)` | Right (last) | — | `("", "", s)` | Split on last separator |
| `rpartition(s, sep)` | Right (last) | — | `("", "", s)` | Convenience alias for right partition |

---

## Notes

- All functions are `func` (pure functions) — safe to use in compile-time contexts.
- None of these functions modify the input string; they always return a new value.
- `partition` and `rpartition` always return a 3-tuple; check the middle element for `""` to detect a missing separator.
- `expandTabs` with `tabSize ≤ 0` silently uses `1` to avoid a division-by-zero error.
