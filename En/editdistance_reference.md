# editdistance — module reference

## Overview

The `editdistance` module provides two procedures for computing the **edit distance** (also known as **Levenshtein distance**) between two strings. Edit distance is the minimum number of single-character operations — **insertions**, **deletions**, and **substitutions** — needed to transform one string into another.

The module offers two variants of the same algorithm:

| Procedure | String model | When to use |
|---|---|---|
| `editDistance` | Unicode runes | Any language, emoji, Cyrillic, CJK, etc. |
| `editDistanceAscii` | Raw bytes (ASCII) | Pure ASCII input; faster due to simpler iteration |

Both procedures are **pure** (marked `noSideEffect`) and use a memory-efficient, linear-space implementation of the Levenshtein algorithm. The algorithm also applies **prefix/suffix stripping** as an optimisation: identical leading and trailing characters are skipped before the main computation begins.

---

## `editDistance`

```nim
proc editDistance*(a, b: string): int
```

### Description

Computes the Levenshtein distance between strings `a` and `b` treating them as **sequences of Unicode runes** (code points), not raw bytes. This is the correct choice for any text that may contain multi-byte characters — accented letters, Cyrillic, Arabic, Chinese, Japanese, Korean, emoji, and so on.

The result is the number of **rune-level** operations (not byte-level operations) required to transform `a` into `b`. For ASCII-only text the result is identical to that of `editDistanceAscii`, but for multibyte text the two functions will diverge because `editDistanceAscii` counts bytes while `editDistance` counts characters.

The procedure is symmetric: `editDistance(a, b) == editDistance(b, a)`. Internally, the shorter string is always placed in position `a` to minimise allocations.

### Parameters

| Name | Type | Description |
|---|---|---|
| `a` | `string` | First input string (UTF-8 encoded) |
| `b` | `string` | Second input string (UTF-8 encoded) |

### Return value

A non-negative `int` equal to the minimum number of rune-level edit operations separating `a` from `b`. Returns `0` when the strings are identical.

### Algorithm notes

- Strips the common **prefix** (identical leading runes) before the main loop.
- Strips the common **suffix** (identical trailing runes) after the prefix pass.
- Runs the standard Levenshtein DP with a single one-dimensional row (linear memory, O(min(|a|,|b|)) space).
- Handles several **special cases** directly: one string empty → return the length of the other; one string is a single rune → scan the longer string for that rune and decide in O(n).

### Examples

**Basic substitution — one character changed:**

```nim
import editdistance

echo editDistance("Kitten", "Bitten")   # → 1  ("K" → "B")
echo editDistance("cat", "car")         # → 1  ("t" → "r")
```

**Multiple operations:**

```nim
echo editDistance("saturday", "sunday")  # → 3
# sat-ur-day → s-un-day:
#   delete "a", substitute "t"→"n", delete "r"
```

**Identical strings — always 0:**

```nim
echo editDistance("hello", "hello")    # → 0
```

**Empty string — distance equals the length of the other:**

```nim
echo editDistance("", "nim")           # → 3
echo editDistance("nim", "")           # → 3  (symmetric)
```

**Unicode / multibyte characters — runes are counted, not bytes:**

```nim
# "кот" (cat in Russian) → "код" (code in Russian): one substitution
echo editDistance("кот", "код")        # → 1

# Each Cyrillic character is 2 bytes, but the distance is still 1 rune
echo editDistance("café", "cave")      # → 2  ("f"→"v", "é"→"e")
```

**Emoji — each emoji counts as one rune:**

```nim
echo editDistance("😀😂", "😀😎")   # → 1
```

---

## `editDistanceAscii`

```nim
proc editDistanceAscii*(a, b: string): int
```

### Description

Computes the Levenshtein distance between strings `a` and `b` treating them as **sequences of bytes**. Each byte is one "character" for the purposes of this function. This is correct and efficient for pure ASCII text, and is meaningfully faster than `editDistance` because it avoids all Unicode decoding overhead.

**Do not use this function with non-ASCII input.** A multibyte UTF-8 character (e.g. a Cyrillic letter encoded as 2 bytes) will be treated as two separate "characters", producing distances that do not correspond to any human-visible character count.

Like `editDistance`, the procedure is symmetric and internally places the shorter string first.

### Parameters

| Name | Type | Description |
|---|---|---|
| `a` | `string` | First input string (ASCII) |
| `b` | `string` | Second input string (ASCII) |

### Return value

A non-negative `int` equal to the minimum number of byte-level edit operations separating `a` from `b`. Returns `0` when the strings are identical.

### Algorithm notes

Identical in structure to `editDistance` but replaces rune iteration (`fastRuneAt`, `runeAt`) with direct byte indexing. This makes the inner loop significantly cheaper for ASCII workloads. The prefix/suffix stripping optimisation is also present.

### Examples

**Basic substitution — ASCII only:**

```nim
import editdistance

echo editDistanceAscii("Kitten", "Bitten")   # → 1
echo editDistanceAscii("hello", "hallo")     # → 1  ("e" → "a")
```

**Typical use-case — spell checking / fuzzy search over ASCII tokens:**

```nim
let dictionary = ["apple", "apply", "aptly", "appeal"]
let query = "appel"

for word in dictionary:
  echo word, ": ", editDistanceAscii(query, word)
# apple:  1
# apply:  2
# aptly:  3
# appeal: 2
```

**Why NOT to use with Unicode:**

```nim
# "кот" is 6 bytes in UTF-8 (2 bytes per Cyrillic letter)
# editDistanceAscii treats it as 6 characters, not 3 runes!
echo editDistanceAscii("кот", "код")   # → 2  (WRONG: expected 1)
echo editDistance("кот", "код")        # → 1  (correct)
```

---

## Choosing between the two procedures

```
Is your input guaranteed to be ASCII only?
    │
    ├─ Yes → use editDistanceAscii   (faster, simpler)
    │
    └─ No  → use editDistance        (correct for any Unicode text)
```

A practical rule: if you are working with identifiers, file names, command-line tokens, or English prose, `editDistanceAscii` is fine. For user-facing text in any language, always prefer `editDistance`.

---

## Common applications

- **Spell checking** — suggest the dictionary word with the smallest edit distance to a misspelled query.
- **Fuzzy string search** — rank candidates by how many edits they require.
- **Diff / change detection** — measure how similar two versions of a string are.
- **Bioinformatics** — compare DNA or protein sequences (ASCII alphabets map naturally to `editDistanceAscii`).
- **OCR post-processing** — correct recognition errors by proximity to known words.

---

## Complexity

| | Time | Space |
|---|---|---|
| Both procedures | O(|a| × |b|) worst case | O(min(|a|,|b|)) |

The linear space comes from the single-row DP approach; only one row of the classic matrix is kept in memory at a time.
