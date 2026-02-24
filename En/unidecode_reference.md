# Unidecode Module — Reference

> **Module origin:** Nim standard library, ported from Python's [Unidecode](https://pypi.org/project/Unidecode/) by Tomaz Solc, which itself derives from the Perl module [Text::Unidecode](https://metacpan.org/pod/Text::Unidecode) by Sean M. Burke.

---

## What This Module Does

`unidecode` solves a very practical problem: how do you represent a Unicode string — Chinese characters, German umlauts, Arabic letters, Cyrillic text — using only the 128 printable characters of ASCII?

The module answers this question by transliterating each Unicode character into the closest ASCII approximation. The transformation is **lossy** (one-way) and **deterministic**: the same input always yields the same output, but different Unicode strings can produce the same ASCII result. Despite the loss of information, a human reader can usually still guess the original from the context.

A pre-built translation table (`unidecode.dat`) maps every Unicode code point up to `0xFFFF` to its ASCII counterpart. By default this file is embedded directly into the compiled binary, so no external file needs to be distributed alongside your program.

---

## Exported Procedures

### `unidecode`

```nim
proc unidecode*(s: string): string
```

#### Description

Takes any UTF-8 encoded string `s` and returns a new string that contains only ASCII characters (code points 0–127). The function walks through every Unicode rune (character) in the input:

- If the rune's code point is **already within the ASCII range** (≤ 127), it is copied to the result unchanged.
- If the rune's code point is **above 127**, the function looks it up in the translation table and appends the corresponding ASCII sequence. If no approximation exists for a particular code point (e.g. some rare or private-use characters), nothing is appended for that rune — it is silently skipped.

The result is always a valid, plain ASCII string.

#### Important nuances

- **One-way transformation.** There is no inverse function. `"Ä"` → `"A"`, but you cannot recover `"Ä"` from `"A"` programmatically.
- **Spaces in transliterations.** Some multi-character scripts (e.g. Chinese) use spaces as separators between words in their ASCII representation. This can produce trailing or extra spaces in the output.
- **Silent omission.** Characters with no known transliteration are dropped without raising an error. Do not rely on the output length being proportional to the input length.
- **Only BMP characters.** The table covers Unicode code points up to `0xFFFF` (the Basic Multilingual Plane). Code points above this range are silently discarded.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `s` | `string` | A UTF-8 encoded input string of any length. |

#### Return value

A `string` containing only ASCII characters (printable or otherwise).

#### Examples

```nim
import unidecode

# Basic Latin letters with diacritics → closest ASCII letters
doAssert unidecode("Äußerst") == "Ausserst"
# "Ä" → "A", "ü" → "u", "ß" → "ss", "e","r","s","t" pass through as-is

# Chinese characters → Pinyin romanisation with spaces
doAssert unidecode("北京") == "Bei Jing "
# Each character maps to its Pinyin syllable; note the trailing space

# Pure ASCII input is returned unchanged
doAssert unidecode("Hello, world!") == "Hello, world!"

# Cyrillic
echo unidecode("Привет")   # → "Privet"

# Japanese (hiragana)
echo unidecode("にほんご")  # → "nihongo"

# Arabic
echo unidecode("مرحبا")    # → "mrHb'" (approximate romanisation)

# Mixed script
echo unidecode("Zürich 北京 Москва")
# → "Zurich Bei Jing  Moskva"
```

#### When to use

- Generating **URL slugs** from Unicode titles.
- Creating **file names** that are safe on ASCII-only file systems.
- **Indexing or searching** text in systems that do not support Unicode collation.
- Producing **human-readable identifiers** from foreign-language input.

---

### `loadUnidecodeTable`

```nim
proc loadUnidecodeTable*(datafile = "unidecode.dat")
```

#### Description

Loads the translation table from an external file on disk, instead of using the version that is compiled into the binary. This procedure is only **meaningful** (and only **does anything**) when the program was compiled with the flag:

```
--define:noUnidecodeTable
```

Without that flag, the translation table is embedded at compile time as a compile-time constant, and calling `loadUnidecodeTable` is a no-op — the call compiles and runs without error, but has no effect whatsoever.

When `--define:noUnidecodeTable` **is** set, the table is not embedded and the module starts with an empty sequence. In that case you **must** call `loadUnidecodeTable` before any call to `unidecode`, or `unidecode` will silently produce incorrect (empty) output for all non-ASCII input.

#### Threading requirement

`loadUnidecodeTable` must be called from the **main thread** before any other thread calls `unidecode`. Because the translation table is a module-level variable shared across threads, initialising it from a worker thread while other threads might be reading it would be a data race. Nim marks the variable `shared` specifically to allow safe concurrent reads after the initial load.

#### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `datafile` | `string` | `"unidecode.dat"` | Path to the data file. Can be relative or absolute. |

#### Return value

None (`void`).

#### Examples

```nim
# --- Compile with: nim c --define:noUnidecodeTable myapp.nim ---

import unidecode

# Must be called once at program start, on the main thread:
loadUnidecodeTable()                        # uses default path "unidecode.dat"
# or with an explicit path:
loadUnidecodeTable("/opt/data/unidecode.dat")

echo unidecode("Привет")  # → "Privet"  (works only after table is loaded)
```

```nim
# If compiled WITHOUT --define:noUnidecodeTable, the call is harmless but useless:
import unidecode

loadUnidecodeTable()  # no-op; table is already embedded in the binary
echo unidecode("Ångström")  # → "Angstrom"  (works regardless)
```

#### When to use

Use `loadUnidecodeTable` (together with `--define:noUnidecodeTable`) when:

- You want to **reduce binary size** by not embedding the ~1 MB data table.
- You need to **ship or update** the translation table independently of the executable.
- You are building a system where the data file path is determined at **runtime** (e.g. from a configuration file).

---

## Compile-time Flags

| Flag | Effect |
|------|--------|
| *(none)* | The translation table is compiled into the binary. No external file needed. `loadUnidecodeTable` is a no-op. |
| `--define:noUnidecodeTable` | The translation table is **not** compiled in. You must call `loadUnidecodeTable` before using `unidecode`. |

---

## Limitations Summary

- Only Unicode code points up to `U+FFFF` are covered.
- The transformation is strictly one-way.
- Characters with no transliteration entry are silently dropped.
- Transliterations are approximate — the result is human-readable, not linguistically precise.
- Output length is unpredictable relative to input length.
