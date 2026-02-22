# `colors` Module Reference

> **Module:** `colors`  
> **Source:** Nim's Runtime Library — `(c) Copyright 2010 Andreas Rumpf`  
> **Purpose:** Color representation, arithmetic, parsing, and conversion. Provides the CSS/X11 named-color palette as compile-time constants.

---

## Overview

The `colors` module revolves around a single type — `Color` — which stores a color as a 24-bit RGB integer (the familiar `0xRRGGBB` format). Everything else in the module is built on top of that: operators for combining colors, helpers for decomposing them into channels, a way to scale brightness, a flexible user-defined mixing template, and string parsing in both `#RRGGBB` hex and CSS named-color forms.

A key design point is **saturated arithmetic**: addition and subtraction on colors clamp each channel to `[0, 255]` instead of wrapping around, which matches natural expectations when blending colors on screen.

---

## Type

### `Color`

```nim
type Color* = distinct int
```

A 24-bit color stored as a plain `int` with the layout `0x00RRGGBB`:

- Bits 23–16 → Red channel (0–255)
- Bits 15–8  → Green channel (0–255)
- Bits 7–0   → Blue channel (0–255)

Because `Color` is a `distinct int`, it cannot be accidentally mixed with ordinary integers. You must use the module's own operators and constructors to work with it. The alpha channel is not represented; this module handles opaque RGB only.

---

## Exported Symbols at a Glance

| Symbol | Kind | One-line summary |
|--------|------|-----------------|
| `==` | operator | Deep equality between two colors |
| `+` | operator | Saturated per-channel addition |
| `-` | operator | Saturated per-channel subtraction |
| `$` | operator | Convert color to `"#RRGGBB"` string |
| `rgb` | proc | Build a `Color` from three channel values |
| `extractRGB` | proc | Decompose a `Color` into `(r, g, b)` tuple |
| `intensity` | proc | Scale brightness by a float factor |
| `mix` | template | Apply a custom function to each channel pair |
| `parseColor` | proc | Parse `"name"` or `"#RRGGBB"` string → `Color` |
| `isColor` | proc | Check whether a string is a valid color |
| Named constants | `const` | ~140 CSS/X11 color names, e.g. `colRed`, `colSkyBlue` |

---

## Operators

### `==`

```nim
proc `==`*(a, b: Color): bool
```

Compares two colors for equality. Borrowed from the underlying `int` comparison, so it is a bitwise check of the full RGB value.

```nim
assert colFuchsia == Color(0xFF00FF)   # same bits → equal
assert colRed != colBlue               # different bits → not equal

let a = Color(0xFF00FF)
let b = colFuchsia
assert a == b   # named constant matches raw hex literal
```

---

### `+`  — Saturated Addition

```nim
proc `+`*(a, b: Color): Color
```

Adds two colors **channel by channel**. Each channel result is clamped to 255 if it would exceed that value — it never wraps. This makes the operator safe to use without worrying about overflow producing nonsense colors.

Think of it as "brightening" one color with another: adding a small orange tint to a dark blue will shift it slightly toward warm tones without blowing out any channel.

```nim
let dark  = Color(0xAA_00_FF)   # R=170 G=0   B=255
let tint  = Color(0x11_CC_CC)   # R=17  G=204 B=204
let mixed = dark + tint
# R: 170+17=187 (0xBB), G: 0+204=204 (0xCC), B: 255+204→clamped to 255 (0xFF)
assert mixed == Color(0xBB_CC_FF)

# Saturation in action:
let white = Color(0xFF_FF_FF)
assert white + white == white   # all channels already at max, nothing changes
```

---

### `-`  — Saturated Subtraction

```nim
proc `-`*(a, b: Color): Color
```

Subtracts two colors channel by channel. Each channel result is clamped to 0 if it would go negative. This is the complement of `+`: use it to darken or remove a tint.

```nim
let a      = Color(0xFF_33_FF)   # R=255 G=51  B=255
let b      = Color(0x11_FF_CC)   # R=17  G=255 B=204
let result = a - b
# R: 255-17=238 (0xEE), G: 51-255→clamped to 0 (0x00), B: 255-204=51 (0x33)
assert result == Color(0xEE_00_33)

# Underflow protection:
assert colBlack - colWhite == colBlack   # can't go below 0
```

---

### `$`  — String Conversion

```nim
proc `$`*(c: Color): string
```

Converts a `Color` value to its canonical uppercase hexadecimal string in the format `"#RRGGBB"`. Useful for outputting colors to HTML, CSS, or log messages. The `#` prefix is always present and the hex digits are always uppercase.

```nim
echo $colFuchsia      # → "#FF00FF"
echo $colBlack        # → "#000000"
echo $Color(0x1a2b3c) # → "#1A2B3C"
```

This is the inverse of `parseColor` for the `#RRGGBB` form.

---

## Constructors

### `rgb`

```nim
proc rgb*(r, g, b: range[0..255]): Color
```

The most readable way to construct a `Color` from separate channel values. Each argument is a `range[0..255]`, so passing a value outside that range is a compile-time or runtime error — you cannot accidentally construct an invalid color.

```nim
let c = rgb(255, 128, 0)   # orange: R=255, G=128, B=0
assert c == Color(0xFF_80_00)

let navy = rgb(0, 0, 128)
assert navy == colNavy

# Range enforced:
# rgb(300, 0, 0)  ← won't compile / raises RangeDefect at runtime
```

Use `rgb` when the channel values come from separate variables or calculations. Use the `Color(0xRRGGBB)` literal form when you already know the hex value.

---

## Decomposition

### `extractRGB`

```nim
proc extractRGB*(a: Color): tuple[r, g, b: range[0..255]]
```

Splits a `Color` back into its three individual channel values. The returned tuple uses named fields `r`, `g`, `b`, each typed as `range[0..255]`. This is the inverse of `rgb`.

Use this when you need to inspect or manipulate individual channels — for example, to convert to HSL, to compare brightness levels, or to format a color as `rgb(R, G, B)` for CSS output.

```nim
let c = Color(0xFF_00_FF)   # fuchsia
let (r, g, b) = extractRGB(c)
assert r == 255
assert g == 0
assert b == 255

# Accessing fields by name:
let ch = extractRGB(colSkyBlue)
echo ch.r, " ", ch.g, " ", ch.b   # → 135 206 235

# Round-trip: rgb ↔ extractRGB
let original = rgb(10, 200, 50)
let (ro, go, bo) = extractRGB(original)
assert rgb(ro, go, bo) == original
```

---

## Brightness

### `intensity`

```nim
proc intensity*(a: Color, f: float): Color
```

Scales every channel of color `a` by the factor `f`. The factor `f` is expected to be in the range `0.0` (completely dark — all channels become 0) to `1.0` (unchanged). Values slightly above 1.0 are tolerated because each channel is clamped to 255 if the multiplication would overflow, but passing very large values or negative values produces undefined-looking results.

Conceptually this models how much light illuminates a surface painted with that color: at `f = 0.5` you get the same hue but half as bright; at `f = 0.0` you always get black.

```nim
assert colWhite.intensity(0.5) == Color(0x80_80_80)   # mid-gray
assert colWhite.intensity(0.0) == colBlack             # total darkness
assert colWhite.intensity(1.0) == colWhite             # no change

let fuchsia = Color(0xFF_00_FF)
assert fuchsia.intensity(0.5) == Color(0x80_00_80)   # 255*0.5=127 → 0x7F rounds to 0x80

let teal = Color(0x00_42_CC)
assert teal.intensity(0.5) == Color(0x00_21_66)
```

> **Note:** Rounding is done via `toInt(toFloat(channel) * f)`, which truncates toward zero. For channel 255 at f=0.5: `toInt(255.0 * 0.5)` = `toInt(127.5)` = `127` (0x7F), not 128. The example in the source shows `0x80` because that particular input rounds up — always verify with the actual channel value.

---

## Custom Blending

### `mix`

```nim
template mix*(a, b: Color, fn: untyped): untyped
```

The most powerful and general color-combination tool in the module. `mix` calls your function `fn` on each pair of corresponding channels `(aR, bR)`, `(aG, bG)`, `(aB, bB)` and assembles the results into a new `Color`. Each result is clamped to `[0, 255]` regardless of what your function returns.

`fn` must accept two `int` arguments and return an `int`. It is evaluated three times — once per channel — inside the template. Because `fn` is an `untyped` argument, you can pass any callable: a named proc, a lambda, or even an inline template.

This covers use cases that `+` and `-` cannot: screen blending, multiply blending, difference, custom weighted averages, etc.

```nim
# Simple average (50/50 blend)
proc avg(x, y: int): int = (x + y) div 2
let blended = mix(colRed, colBlue, avg)
# R: (255+0)/2=127, G: (0+0)/2=0, B: (0+255)/2=127
assert blended == Color(0x7F_00_7F)

# "Screen" blend mode: 1 - (1-a)(1-b)  → expressed in integer arithmetic
proc screen(x, y: int): int =
  255 - ((255 - x) * (255 - y)) div 255
let s = mix(Color(0x80_80_80), Color(0x80_80_80), screen)
# screen(128,128) = 255 - (127*127)/255 ≈ 255 - 63 = 192
assert s == Color(0xBF_BF_BF)

# Negative result → clamped to 0
proc myMix(x, y: int): int = 2 * x - 3 * y
let a = Color(0x0A_28_14)
let b = Color(0x05_0A_03)
assert mix(a, b, myMix) == Color(0x05_32_1F)
```

---

## Parsing and Validation

### `parseColor`

```nim
proc parseColor*(name: string): Color
```

Converts a string to a `Color`. Two formats are accepted:

- **Named CSS colors** — case-insensitive, e.g. `"red"`, `"SkyBlue"`, `"GOLDENROD"`. Internally looked up via binary search in the sorted `colorNames` table.
- **Hex literal** — a string starting with `#` followed by up to 6 hex digits, e.g. `"#FF00FF"`, `"#0179fc"`. Parsed with `parseHexInt`.

Raises `ValueError` if the string is neither a recognised name nor a valid hex string.

```nim
assert parseColor("silver")  == Color(0xC0_C0_C0)
assert parseColor("Silver")  == Color(0xC0_C0_C0)   # case-insensitive
assert parseColor("SILVER")  == Color(0xC0_C0_C0)

assert parseColor("#0179fc") == Color(0x01_79_FC)
assert parseColor("#FF00FF") == colFuchsia

# Invalid input:
try:
  discard parseColor("#zzmmtt")
except ValueError as e:
  echo e.msg   # → "invalid hex string: #zzmmtt" (from parseHexInt)

try:
  discard parseColor("notacolor")
except ValueError as e:
  echo e.msg   # → "unknown color: notacolor"
```

> **Tip:** If you are unsure whether a string is valid, call `isColor` first to avoid catching exceptions in hot paths.

---

### `isColor`

```nim
proc isColor*(name: string): bool
```

A non-throwing validator that returns `true` if `name` is either a known CSS named color or a syntactically valid `#`-prefixed hex color string. It does not raise any exceptions — it simply returns `false` for anything unrecognisable.

The hex check validates that every character after `#` is a hex digit (`0–9`, `a–f`, `A–F`). An empty string returns `false`.

```nim
assert "silver".isColor       # known CSS name
assert "#0179fc".isColor      # valid hex
assert "#FFFFFF".isColor      # uppercase hex
assert not "#zzmmtt".isColor  # 'z' and 'm' are not hex digits
assert not "".isColor         # empty string
assert not "electricpink".isColor  # not in CSS palette

# Typical use: validate before parsing
let input = getUserInput()
if input.isColor:
  let c = parseColor(input)
  draw(c)
else:
  echo "Not a valid color: " & input
```

---

## Named Color Constants

The module exports approximately 140 CSS/X11 color constants, all prefixed with `col`. They are compile-time `const` values of type `Color`.

A few representative examples:

| Constant | Hex value | Approximate appearance |
|----------|-----------|----------------------|
| `colBlack` | `0x000000` | Black |
| `colWhite` | `0xFFFFFF` | White |
| `colRed` | `0xFF0000` | Pure red |
| `colGreen` | `0x008000` | CSS green (dark) |
| `colLime` | `0x00FF00` | Pure bright green |
| `colBlue` | `0x0000FF` | Pure blue |
| `colFuchsia` / `colMagenta` | `0xFF00FF` | Magenta (two aliases) |
| `colCyan` / `colAqua` | `0x00FFFF` | Cyan (two aliases) |
| `colGray` / `colGrey` | `0x808080` | Medium gray (two aliases) |
| `colCornflowerBlue` | `0x6495ED` | Soft blue |
| `colRebeccaPurple` | `0x663399` | Medium purple |
| `colTomato` | `0xFF6347` | Orange-red |
| `colGold` | `0xFFD700` | Bright yellow-gold |

Note that several CSS colors come in both American (`Gray`) and British (`Grey`) spellings — they are identical values exported under two names.

```nim
# Use named constants for readability
let background = colMidnightBlue
let highlight  = colGold
let shadow     = background.intensity(0.3)
```

---

## Common Patterns

### Building a palette from CSS names

```nim
let palette = ["tomato", "steelblue", "goldenrod", "mediumseagreen"]
for name in palette:
  let c = parseColor(name)
  echo name, " → ", $c
```

### Lightening and darkening

```nim
let base   = colSteelBlue                  # original
let light  = base.intensity(1.5)           # over-exposed (channels clamp to 255)
let dark   = base.intensity(0.4)           # darker shade
```

### User-defined screen blend

```nim
proc screen(a, b: int): int =
  255 - ((255 - a) * (255 - b)) div 255

let overlay = mix(colCornflowerBlue, colTomato, screen)
```

### Decompose and reconstruct with a gamma correction

```nim
proc gammaCorrect(c: Color, gamma: float): Color =
  let (r, g, b) = extractRGB(c)
  let f = proc (ch: int): int =
    toInt(pow(ch.float / 255.0, gamma) * 255.0)
  rgb(f(r), f(g), f(b))
```

---

## Error Reference

| Situation | Exception | Message pattern |
|-----------|-----------|-----------------|
| Unknown color name in `parseColor` | `ValueError` | `"unknown color: <name>"` |
| Invalid hex string in `parseColor` | `ValueError` | raised by `parseHexInt` |
| Channel value out of `[0..255]` in `rgb` | `RangeDefect` | raised by range check |
