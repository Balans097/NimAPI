# `wordwrap` Module Reference

> An algorithm for wrapping Unicode strings at word boundaries.  
> Part of Nim's standard library (`std/wordwrap`).

---

## What word wrapping is

Word wrapping is the process of breaking a long string into multiple lines so that each line does not exceed a given width. The key challenge is doing it *gracefully*: breaking at spaces or other separators whenever possible, and only breaking mid-word when there is truly no alternative (or when explicitly requested).

This module handles **Unicode text** correctly — line widths are measured in *grapheme clusters* (visible characters), not raw bytes. An accented letter like `é` or an emoji like `🌍` each count as **one** unit of width, regardless of how many bytes they occupy.

---

## Exported symbols

### `wrapWords` — wrap a Unicode string to a target line width

```nim
proc wrapWords*(
  s: string,
  maxLineWidth = 80,
  splitLongWords = true,
  seps: set[char] = Whitespace,
  newLine = "\n"
): string {.noSideEffect.}
```

**What it does:** Scans `s` from left to right, tracks how many grapheme columns have been used on the current line, and inserts `newLine` breaks at appropriate points to keep each line at or below `maxLineWidth` columns. Returns the wrapped string; the original is never modified.

---

## Parameters in depth

### `s: string`

The input text to wrap. May contain any valid UTF-8, including multi-byte characters, combining marks, and emoji. Line-width accounting is done in grapheme clusters via `std/unicode.graphemeLen`, so the visual width — not the byte count — drives all decisions.

---

### `maxLineWidth: int = 80`

The maximum number of grapheme columns allowed per line. The default of 80 matches the traditional terminal width.

A separator that falls exactly at the boundary does not cause a break by itself; only the next *word* (non-separator token) that would push the line over the limit triggers a break.

```nim
import std/wordwrap

# Default width of 80 — short sentence fits on one line
echo "Hello, world!".wrapWords()
# → "Hello, world!"

# Width of 10 — must break
echo "Hello, world!".wrapWords(10)
# → "Hello,\nworld!"

# Width of 20 — long run of digits gets split
echo "123456789012345678901234567890".wrapWords(20)
# → "12345678901234567890\n1234567890"
```

---

### `splitLongWords: bool = true`

Controls what happens when a single word (an unbroken non-separator token) is *longer* than `maxLineWidth` all by itself.

- **`true` (default):** The word is split mid-grapheme-cluster, inserting `newLine` every time the column counter reaches zero. The word is never left hanging off the end of the line.
- **`false`:** The oversized word is placed on its own line and allowed to exceed `maxLineWidth`. The line is not split mid-word under any circumstances.

```nim
import std/wordwrap

let longWord = "superlongwordthatexceedsthewidth"

# splitLongWords = true (default): broken ruthlessly mid-word
echo longWord.wrapWords(10)
# → "superlongw\nordthatexc\needsthewid\nth"

# splitLongWords = false: placed on its own line intact
echo longWord.wrapWords(10, splitLongWords = false)
# → "superlongwordthatexceedsthewidth"
# (no break at all — the word simply overflows)

# Realistic case: a normal sentence with one long technical term
echo "Hello Bob. Hello John.".wrapWords(13, false)
# → "Hello Bob.\nHello John."
```

---

### `seps: set[char] = Whitespace`

The set of characters that are treated as *separators* — the boundaries where line breaks are allowed to occur. By default this is `std/strutils.Whitespace`, which includes space, tab, newline (`\n`), carriage return (`\r`), form feed, and vertical tab.

**How separators are handled:**

- Newline and carriage return characters in the separator run are *discarded* — they don't appear in the output. Other separator characters (spaces, tabs) are collected and replayed before the following word, unless a line break is inserted instead.
- If a separator run consists only of newlines/carriage-returns (no spaces or tabs), a single space is synthesised and counted against the line budget.

You can pass a custom set to treat other characters (e.g. `;`, `/`, `-`) as valid break points:

```nim
import std/wordwrap

# Using semicolon as the only separator
echo "Hello Bob. Hello John.".wrapWords(13, true, {';'})
# → "Hello Bob. He\nllo John."
# (only ';' is a separator, so the space inside "Hello Bob." is NOT a break point;
#  the algorithm is forced to split mid-word instead)

# Allow both spaces and hyphens as break points
echo "long-hyphenated-compound-word".wrapWords(10, false, {' ', '-'})
# → "long-\nhyphenated-\ncompound-\nword"
```

---

### `newLine: string = "\n"`

The string inserted at each line break. Defaults to a Unix newline `"\n"`. Override with `"\r\n"` for Windows-style line endings, or with any other string (e.g. `"<br>"` for HTML output).

```nim
import std/wordwrap

echo "Hello world from Nim".wrapWords(11, newLine = "\r\n")
# → "Hello world\r\nfrom Nim"

echo "Hello world from Nim".wrapWords(11, newLine = "<br>")
# → "Hello world<br>from Nim"
```

---

## Algorithm walkthrough

Understanding the algorithm helps predict edge cases:

1. The string is scanned left-to-right, alternating between *separator runs* and *word runs*.
2. For each **separator run**: newlines and carriage returns are dropped; remaining characters are saved as `lastSep`. A purely newline/CR run synthesises a single space. The saved separator is subtracted from `spaceLeft`.
3. For each **word run** of grapheme-column width `wlen`:
   - If `wlen ≤ spaceLeft`: append `lastSep` then the word; subtract `wlen` from `spaceLeft`.
   - If `wlen > spaceLeft` and (`splitLongWords = false` or `wlen ≤ maxLineWidth`): insert `newLine`, place the word on the new line, reset `spaceLeft = maxLineWidth - wlen`.
   - If `wlen > spaceLeft` and `splitLongWords = true` and `wlen > maxLineWidth`: split the word grapheme-by-grapheme, inserting `newLine` whenever `spaceLeft` hits zero.

**Key consequence:** The separator immediately before a word that triggers a line break is *not* emitted — only the `newLine` is. This means you won't get a trailing space at the end of a line.

---

## Unicode correctness

Width is measured using `graphemeLen` from `std/unicode`, which respects Unicode grapheme cluster boundaries. This means:

- A combining accent (e.g. `e` + combining acute = `é`) counts as **1** column.
- A zero-width joiner sequence (e.g. family emoji 👨‍👩‍👧) counts as **1** column.
- A CJK character counts as **1** column (note: this module does *not* implement East Asian double-width accounting — if you need that, post-process with a dedicated library).

```nim
import std/wordwrap

# Cyrillic — each letter is one grapheme = one column
echo "Привет мир".wrapWords(7)
# → "Привет\nмир"
```

---

## Practical examples

```nim
import std/wordwrap

# ── Terminal output ──────────────────────────────────────────────────────────
let longText = "The quick brown fox jumps over the lazy dog near the riverbank."
echo longText.wrapWords(30)
# → "The quick brown fox jumps over\nthe lazy dog near the riverbank."

# ── HTML paragraph ───────────────────────────────────────────────────────────
let html = longText.wrapWords(40, newLine = "<br>\n")
echo "<p>" & html & "</p>"

# ── Windows-style line endings ───────────────────────────────────────────────
let winText = longText.wrapWords(40, newLine = "\r\n")

# ── Preserving intentional breaks ────────────────────────────────────────────
# If the input already contains newlines, they are treated as separators and
# normalised (collapsed into a single space). Pre-split text is NOT preserved.
echo "Line one\nLine two".wrapWords(40)
# → "Line one Line two"   (the \n became a space)

# ── Code path identifier with slashes ────────────────────────────────────────
echo "some/very/long/path/that/overflows".wrapWords(12, false, {'/'})
# → "some/\nvery/\nlong/\npath/\nthat/\noverflows"
```

---

## Quick cheat-sheet

| Parameter | Default | Effect |
|-----------|---------|--------|
| `maxLineWidth` | `80` | Maximum grapheme columns per line |
| `splitLongWords` | `true` | Split words longer than `maxLineWidth` mid-grapheme |
| `seps` | `Whitespace` | Characters that serve as break-point boundaries |
| `newLine` | `"\n"` | String inserted at each line break |

| Behaviour | Notes |
|-----------|-------|
| Width unit | Grapheme clusters (not bytes, not code points) |
| Separator at line end | Dropped — no trailing whitespace on lines |
| Input newlines | Treated as separators, normalised to a space |
| Long-word overflow (`splitLongWords=false`) | Word placed on own line, may exceed `maxLineWidth` |
| Side effects | None (`{.noSideEffect.}`) |
