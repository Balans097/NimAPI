# `strbasics` Module Reference

> **Module purpose:** `strbasics` provides a small set of **high-performance, in-place string operations** for Nim. All exported procedures modify a string directly in memory rather than allocating and returning a new one, which eliminates heap allocations and makes them suitable for hot paths and performance-critical code.
>
> ⚠️ **Experimental API.** The interface may change in future Nim versions.

---

## Core Philosophy: In-Place Mutation

The standard library's `strutils` module tends to return *new* strings — convenient, but costly when called in tight loops. `strbasics` takes the opposite approach: every exported procedure mutates its first argument (`var string`) directly. The caller owns the string and no extra allocation occurs.

This design fits naturally with Nim's `dup` helper from `std/sugar`, which makes it easy to get a modified *copy* when you do need one (see examples below).

---

## Internal constant: `whitespaces`

```nim
const whitespaces = {' ', '\t', '\v', '\r', '\l', '\f'}
```

This is the default set of characters considered "whitespace" by `strip`. It is not exported but it is the default value for the `chars` parameter in `strip`. It covers:

| Character | Description |
|---|---|
| `' '` | Space |
| `'\t'` | Horizontal tab |
| `'\v'` | Vertical tab |
| `'\r'` | Carriage return |
| `'\l'` | Line feed (`\n`) |
| `'\f'` | Form feed |

---

## Procedure Reference

---

### `add`

```nim
proc add*(x: var string, y: openArray[char])
```

#### What it does

Appends all characters from `y` to the end of `x`, growing `x` in place. This is an overload of the built-in `add` that accepts an `openArray[char]` as the source — meaning it works with strings, sequences of char, array slices, and any other char-compatible collection without extra conversion.

#### Performance note

The procedure is written to allow future `memcpy`-based optimisation (currently using a simple loop). The key constraint enabling this optimisation is the **no-alias rule**: `y` must not be a view into `x` itself. Appending a slice of the same string to itself is undefined behaviour.

#### Parameter summary

| Parameter | Type | Description |
|---|---|---|
| `x` | `var string` | The string being extended (mutated in place) |
| `y` | `openArray[char]` | The characters to append; must not overlap with `x` |

#### Examples

```nim
import strbasics

# Append a string literal (implicitly converted to openArray[char])
var s = "Hello"
s.add(", world!")
assert s == "Hello, world!"

# Append a seq[char]
var t = "Nim"
let suffix: seq[char] = @[' ', 'r', 'o', 'c', 'k', 's']
t.add(suffix)
assert t == "Nim rocks"

# Append a slice of another string (safe — different variable, no overlap)
var base = "foo"
let extra = "barBaz"
base.add(extra.toOpenArray(0, 2))  # appends "bar"
assert base == "foobar"
```

#### When to prefer this over `&=`

The built-in `&=` only accepts a `string` on the right-hand side. `add` here accepts any `openArray[char]`, so you can append slices, arrays, or `seq[char]` directly without constructing an intermediate string.

---

### `setSlice`

```nim
func setSlice*(s: var string, slice: Slice[int])
```

#### What it does

Replaces the content of `s` with the substring defined by `slice` — **in place**, without allocating a new string. Think of it as a destructive, zero-allocation version of `substr` or `s[a..b]`.

After the call, `s` contains exactly the characters that were at positions `slice.a` through `slice.b` (inclusive).

#### Preconditions (enforced by `assert`)

| Condition | What happens if violated |
|---|---|
| `slice.a >= 0` | `AssertionDefect` at runtime |
| `slice.b <= s.high` | `AssertionDefect` at runtime |

#### Special case: empty result

If `slice.a > slice.b` (an inverted or zero-length range), `s` is set to an empty string (`""`) without raising an error. This makes it easy to clear a string conditionally.

#### Parameter summary

| Parameter | Type | Description |
|---|---|---|
| `s` | `var string` | The string to slice in place |
| `slice` | `Slice[int]` | The inclusive index range to keep |

#### Examples

```nim
import strbasics, std/sugar

var a = "Hello, Nim!"

# Keep only "Nim" (indices 7, 8, 9)
doAssert a.dup(setSlice(7 .. 9)) == "Nim"

# Keep a single character
doAssert a.dup(setSlice(0 .. 0)) == "H"

# Keep the whole string (no-op effectively)
doAssert a.dup(setSlice(0 .. 10)) == "Hello, Nim!"

# Inverted range → empty string (no error)
doAssert a.dup(setSlice(1 .. 0)).len == 0

# Out-of-range high index → AssertionDefect
doAssertRaises(AssertionDefect):
  discard a.dup(setSlice(1 .. 11))
```

#### Using `dup` for non-destructive copies

`dup` (from `std/sugar`) copies the string, applies the mutation to the copy, and returns it — leaving the original unchanged. This is the idiomatic way to use `setSlice` (and `strip`) when you need a new value without losing the original.

```nim
import std/sugar, strbasics

var original = "Hello, Nim!"
let shortened = original.dup(setSlice(7 .. 9))
assert original == "Hello, Nim!"  # unchanged
assert shortened == "Nim"
```

#### Implementation detail

When running on native targets with `nimSeqsV2`, `setSlice` uses `moveMem` (essentially `memmove`) to shift the relevant bytes to the front of the buffer, then calls `setLen` to trim the length. This means the operation is **O(n)** in the length of the kept slice, with no allocation.

---

### `strip`

```nim
func strip*(a: var string, leading = true, trailing = true,
            chars: set[char] = whitespaces) {.inline.}
```

#### What it does

Removes characters from the beginning and/or end of `a` **in place**. By default it removes all whitespace characters from both sides. All three behaviours (leading only, trailing only, both sides) are controlled by boolean parameters.

This is the in-place counterpart to `strutils.strip`, which returns a new string. `strbasics.strip` mutates the original and performs no allocation.

#### Parameter summary

| Parameter | Type | Default | Description |
|---|---|---|---|
| `a` | `var string` | — | The string to strip (mutated in place) |
| `leading` | `bool` | `true` | Strip matching characters from the start |
| `trailing` | `bool` | `true` | Strip matching characters from the end |
| `chars` | `set[char]` | `whitespaces` | The set of characters to remove |

When both `leading` and `trailing` are `false`, the string is left completely unchanged.

#### Examples

```nim
import strbasics

# Default: strip whitespace from both ends
var a = "  vhellov   "
strip(a)
assert a == "vhellov"

# Strip trailing whitespace only
a = "  vhellov   "
a.strip(leading = false)
assert a == "  vhellov"

# Strip leading whitespace only
a = "  vhellov   "
a.strip(trailing = false)
assert a == "vhellov   "

# Strip a custom set of characters from both ends
var c = "blaXbla"
c.strip(chars = {'b', 'a'})
assert c == "laXbl"

# Expand the custom set
c = "blaXbla"
c.strip(chars = {'b', 'a', 'l'})
assert c == "X"
```

#### Using `dup` for non-destructive stripping

```nim
import std/sugar, strbasics

let original = "  hello  "
let stripped = original.dup(strip)
assert original == "  hello  "  # unchanged
assert stripped == "hello"
```

#### How it works internally

`strip` is a thin `{.inline.}` wrapper that calls the private `stripSlice` helper to compute which index range to keep, then delegates to `setSlice` to perform the in-place trimming. No characters are ever written — the function only moves a start pointer and shrinks the length, making it very cache-friendly.

---

## Private helper: `stripSlice` (not exported)

```nim
func stripSlice(s: openArray[char], leading = true, trailing = true,
                chars: set[char] = whitespaces): Slice[int]
```

Not part of the public API, but useful to understand the module's internals. It computes and returns the `Slice[int]` of the input that remains after stripping — without actually modifying anything. `strip` calls this to determine what range `setSlice` should keep.

```nim
assert stripSlice(" abc  ") == 1 .. 3
# Positions: 0=' ', 1='a', 2='b', 3='c', 4=' ', 5=' '
# Stripped range is indices 1 through 3 → "abc"
```

---

## Summary Table

| Procedure | In-place | Allocates | Purpose |
|---|---|---|---|
| `add` | ✅ | No (extends existing buffer) | Append `openArray[char]` to a string |
| `setSlice` | ✅ | No | Trim string to a specific index range |
| `strip` | ✅ | No | Remove leading/trailing characters |

---

## Common Patterns

### Pattern 1 — High-throughput line processing

```nim
import strbasics

proc processLines(data: string) =
  var line = ""
  for ch in data:
    if ch == '\n':
      line.strip()          # trim in place, no allocation
      if line.len > 0:
        echo line
      line.setLen(0)        # reuse the buffer
    else:
      line.add([ch])
```

### Pattern 2 — Building a string from parts without intermediates

```nim
import strbasics

var result = ""
for part in ["Hello", ", ", "world", "!"]:
  result.add(part.toOpenArray(0, part.high))
assert result == "Hello, world!"
```

### Pattern 3 — Non-destructive use with `dup`

```nim
import strbasics, std/sugar

let raw = "  |data|  "
let clean = raw.dup do (s: var string):
  s.strip()
  s.setSlice(1 .. s.high - 1)  # remove surrounding pipes
assert clean == "data"
assert raw == "  |data|  "     # original untouched
```

---

## Notes

- All procedures accept `var string` — you must have a mutable variable; string literals cannot be passed directly.
- `setSlice` uses `assert` (not `doAssert` or a softer check), so violations always abort in debug *and* release builds unless assertions are explicitly disabled.
- The `{.inline.}` on `strip` means the compiler will typically inline the call, reducing function call overhead in loops.
- The module is part of Nim's standard library but marked **experimental** — avoid relying on it in stable library code without pinning your Nim version.
