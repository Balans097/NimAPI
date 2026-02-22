# std/strformat — Module Reference

> String interpolation and formatting for Nim, inspired by Python's f-strings.
> Everything is resolved at **compile time** via macros — no runtime parsing overhead.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [The `fmt` Template](#the-fmt-template)
3. [The `&` Operator](#the--operator)
4. [fmt vs. & — The Critical Difference](#fmt-vs----the-critical-difference)
5. [Format Specifier Syntax](#format-specifier-syntax)
6. [Alignment and Fill](#alignment-and-fill)
7. [Sign Control](#sign-control)
8. [Integer Formatting](#integer-formatting)
9. [Float Formatting](#float-formatting)
10. [String Formatting](#string-formatting)
11. [Debug Mode — `expr=`](#debug-mode--expr)
12. [Arbitrary Expressions in Braces](#arbitrary-expressions-in-braces)
13. [Custom Delimiters](#custom-delimiters)
14. [Extending Formatting — `formatValue`](#extending-formatting--formatvalue)
15. [`alignString`](#alignstring)
16. [`parseStandardFormatSpecifier`](#parsestandardformatspecifier)
17. [`StandardFormatSpecifier`](#standardformatspecifier)
18. [Escaping Braces](#escaping-braces)
19. [Limitations](#limitations)
20. [Quick-Reference Cheatsheet](#quick-reference-cheatsheet)

---

## Core Concepts

`std/strformat` turns string literals containing `{expression}` placeholders into efficient string-building code at compile time. The compiler rewrites each format string into a sequence of `add` calls on a pre-allocated buffer — there is no runtime format-string parsing.

The module exports two surface-level tools:

- **`fmt`** — a template/macro that takes a *raw string literal* (backslashes are not escape sequences).
- **`&`** — a unary operator that takes a *regular string literal* (backslashes work normally).

Both expand into the same internal machinery, but the escape behaviour of their respective string literals differs in an important way (see [fmt vs. &](#fmt-vs----the-critical-difference)).

---

## The `fmt` Template

```nim
template fmt*(pattern: static string): string
template fmt*(pattern: static string; openChar: static char, closeChar: static char): string
```

Interpolates `pattern` using variables and expressions visible in the current scope. `pattern` must be a **compile-time constant** (a literal or a `const`).

Because `fmt"..."` is a *generalized raw string literal*, backslash sequences such as `\n` are **not** interpreted — the backslash is kept as-is in the output.

```nim
let name = "Alice"
let score = 98.5

echo fmt"Player: {name}, Score: {score:.1f}"
# Player: Alice, Score: 98.5

const template_str = "Hello, {name}!"
echo template_str.fmt
# Hello, Alice!

# fmt uses a raw string literal — \n is literal backslash-n
echo fmt"{name}\n"
# Alice\n     ← NOT a newline!
```

---

## The `&` Operator

```nim
template `&`*(pattern: string{lit}): string
```

A unary prefix operator that is exactly equivalent to `pattern.fmt`. The key difference from `fmt"..."` is that `&"..."` takes a **regular string literal**, so escape sequences like `\n`, `\t`, `\"` are processed normally by the Nim lexer before the macro sees the string.

```nim
let name = "Alice"

echo &"{name}\n"
# Alice        ← followed by a real newline

echo &"Tab:\there"
# Tab:	here   ← real tab character
```

Use `&` when your format string needs to contain newlines, tabs, or other escape sequences interleaved with interpolation.

---

## fmt vs. & — The Critical Difference

This is the single most common source of confusion in the module.

| Feature | `fmt"..."` | `&"..."` |
|---|---|---|
| String literal kind | Raw (generalized raw string literal) | Regular |
| `\n` in the pattern | Literal `\n` (two characters) | Real newline |
| `\t` in the pattern | Literal `\t` (two characters) | Real tab |
| Interpolation | Yes | Yes |
| Compile-time | Yes | Yes |

```nim
let x = "hello"

# fmt — raw literal, \n stays as backslash-n
assert fmt"{x}\n" == "hello\\n"

# & — regular literal, \n becomes a real newline
assert &"{x}\n" == "hello\n"

# Workarounds when you need both newlines and fmt:
assert fmt"{x}{'\n'}"   == "hello\n"  # embed char literal
assert fmt("{x}\n")     == "hello\n"  # call syntax uses regular literal
assert "{x}\n".fmt      == "hello\n"  # method syntax uses regular literal
```

**Rule of thumb:** if your format string has escape sequences that you want to be interpreted as actual control characters, use `&`. If you are using `fmt` and need a newline, embed it as `{'\n'}` or use the call/method syntax.

---

## Format Specifier Syntax

Inside `{expression}`, an optional colon followed by a *format specifier* controls how the value is rendered:

```
{expression}           ← default formatting
{expression:specifier} ← custom formatting
```

The general grammar of a format specifier is:

```
[[fill]align][sign][#][0][minimumwidth][.precision][type]
```

Every component is optional. They are applied left to right. Below is a description of each component.

---

## Alignment and Fill

`[fill]align` controls how a value is padded when it is shorter than `minimumwidth`.

The **align** character can be:

| Character | Meaning | Default for |
|---|---|---|
| `<` | Left-align | strings |
| `>` | Right-align | numbers |
| `^` | Center | — |

The optional **fill** character (any single character before the align character) is used for padding instead of the default space.

```nim
# Right-align in a field of width 8 (default for numbers)
assert fmt"{42:8}"    == "      42"

# Left-align explicitly
assert fmt"{42:<8}"   == "42      "

# Center
assert fmt"{42:^8}"   == "   42   "

# Custom fill character
assert fmt"{'x':*^10}" == "xxxx'x'xxx"   # illustrative
assert fmt"{42:0>6}"  == "000042"        # zero fill, right-align

# Strings default to left-alignment
assert &"""{"abc":>6}""" == "   abc"     # right-align a string
assert &"""{"abc":<6}""" == "abc   "     # explicit left (default)
assert &"""{"abc":^6}""" == " abc  "     # center
```

---

## Sign Control

The `sign` component applies only to **numeric** values.

| Character | Meaning |
|---|---|
| `+` | Always show a sign (both positive and negative) |
| `-` | Show sign only for negative numbers (default) |
| ` ` (space) | Space before positive numbers, `-` before negative |

```nim
assert fmt"{42:+}"  == "+42"
assert fmt"{42:-}"  == "42"
assert fmt"{42: }"  == " 42"
assert fmt"{-42:+}" == "-42"
assert fmt"{-42: }" == "-42"
```

---

## Integer Formatting

The `type` character for integers selects the output base:

| Type | Meaning |
|---|---|
| `d` or (none) | Decimal (base 10) |
| `b` | Binary (base 2) |
| `o` | Octal (base 8) |
| `x` | Hexadecimal, lowercase |
| `X` | Hexadecimal, uppercase |

The `#` flag adds the base prefix (`0b`, `0o`, `0x`) automatically.

The `0` flag enables zero-padding (equivalent to `fill=0, align=>`).

```nim
let n = 255

assert fmt"{n}"      == "255"       # decimal (default)
assert fmt"{n:d}"    == "255"       # explicit decimal
assert fmt"{n:b}"    == "11111111"  # binary
assert fmt"{n:o}"    == "377"       # octal
assert fmt"{n:x}"    == "ff"        # hex lowercase
assert fmt"{n:X}"    == "FF"        # hex uppercase
assert fmt"{n:#x}"   == "0xff"      # hex with prefix
assert fmt"{n:#b}"   == "0b11111111"
assert fmt"{n:08}"   == "00000255"  # zero-padded to width 8
assert fmt"{n:>10}"  == "       255" # right-aligned in field of 10

# Negative numbers and zero-padding
assert fmt"{-12345:08}" == "-0012345"
assert fmt"{-1:03}"     == "-01"
assert fmt"{-1:3}"      == " -1"
```

---

## Float Formatting

The `type` character for floats selects the notation:

| Type | Meaning |
|---|---|
| `f` / `F` | Fixed-point decimal |
| `e` / `E` | Scientific (exponential) notation |
| `g` / `G` | General: fixed unless too large, then scientific |
| (none) | Like `g`, but always shows at least one decimal digit |

The `.precision` component sets the number of digits after the decimal point (for `f`, `e`) or the total number of significant digits (for `g`).

```nim
let f = 123.456

assert fmt"{f}"          == "123.456"
assert fmt"{f:.2f}"      == "123.46"        # 2 decimal places
assert fmt"{f:9.3f}"     == "  123.456"     # width 9, 3 decimals
assert fmt"{f:<9.4f}"    == "123.4560 "     # left-align, 4 decimals
assert fmt"{f:>9.0f}"    == "     123."     # 0 decimals
assert fmt"{f:e}"        == "1.234560e+02"  # scientific
assert fmt"{f:>13e}"     == " 1.234560e+02" # right-aligned scientific
assert fmt"{f:.5g}"      == "123.46"        # 5 significant digits

# Sign with floats
assert fmt"{1.5:+.1f}"  == "+1.5"
assert fmt"{-1.5:+.1f}" == "-1.5"

# Zero-padding with floats
assert fmt"{1.5:08.2f}" == "00001.50"
```

---

## String Formatting

For string values the `precision` component truncates the string to at most that many Unicode code points (not bytes). The valid type character is `s` or omitted.

```nim
let s = "Hello, 世界"

assert fmt"{s}"      == "Hello, 世界"
assert fmt"{s:.5}"   == "Hello"      # truncate to 5 codepoints
assert fmt"{s:>15}"  == "     Hello, 世界"  # right-align in 15
assert fmt"{s:<15}"  == "Hello, 世界     "  # left-align
assert fmt"{s:^15}"  == "  Hello, 世界   "  # center
```

---

## Debug Mode — `expr=`

Adding `=` after the expression name inside braces prints both the expression's source text and its value. This is invaluable for quick debugging without introducing separate `echo` statements.

```
{expr=}           → "expr=<value>"
{expr=:specifier} → "expr=<formatted value>"
```

The whitespace around `=` is preserved literally in the output, giving you fine-grained control over the separator's appearance.

```nim
let x = 42
let name = "Bob"

assert fmt"{x=}"          == "x=42"
assert fmt"{x = }"        == "x = 42"      # spaces around = are kept
assert fmt"{x=:08}"       == "x=00000042"  # specifier after =

let pi = 3.14159
assert fmt"{pi=:.2f}"     == "pi=3.14"

proc double(n: int): int = n * 2
assert fmt"{double(x) = }" == "double(x) = 84"
assert fmt"{x.double = }"  == "x.double = 84"
```

---

## Arbitrary Expressions in Braces

Any valid Nim expression can appear between the braces — not just variable names. This includes arithmetic, function calls, method calls, conditional expressions, and even `block` expressions.

```nim
let a = 10
let b = 3

assert fmt"{a + b}"        == "13"
assert fmt"{a * b - 1}"    == "29"
assert fmt"{a div b}"      == "3"

# Conditional expression
assert fmt"{(if a > b: \"yes\" else: \"no\")}" == "yes"

# Function call
import std/strutils
assert fmt"{'hello'.toUpperAscii}" == "HELLO"

# A colon inside parentheses does NOT need to be escaped
let x = 3.14
assert fmt"{(if x != 0: 1.0/x else: 0):.5}" == "0.31847"

# Block expression (multi-line logic in a format string)
assert fmt"""{(block:
  var res = ""
  for i in 1..5:
    res &= $i & " "
  res.strip)}""" == "1 2 3 4 5"
```

---

## Custom Delimiters

By default `fmt` uses `{` and `}` as the open and close delimiters. You can override both with static character arguments, which is useful when your content itself contains many curly braces (e.g. JSON, CSS).

```nim
let x = 7

# Using < and > as delimiters
assert "<x>".fmt('<', '>') == "7"
assert "<<<x>>>".fmt('<', '>') == "<7>"   # << >> are escape sequences

# Using backtick as both open and close delimiter
assert "`x`".fmt('`', '`') == "7"
```

To escape a delimiter, double it — `<<` produces a literal `<` when `<` is the open delimiter.

---

## Escaping Braces

To include a literal `{` or `}` in the output, double the character in the format string:

```nim
assert fmt"{{" == "{"
assert fmt"}}" == "}"
assert fmt"{{x}}" == "{x}"    # literal braces, not interpolation
assert fmt"{{{42}}}" == "{42}" # brace + value + brace
```

Inside a curly-brace expression, `{` and `}` that are part of Nim syntax (e.g. in a table constructor) must be escaped with a backslash:

```nim
# Passing a table literal to $() inside a format expression
assert fmt"""{ $(\{x:1,"world":2\}) }""" == """[("hello", 1), ("world", 2)]"""
```

---

## Extending Formatting — `formatValue`

`fmt` and `&` delegate all actual rendering to an open family of `formatValue` overloads. To make your own type formattable, define:

```nim
proc formatValue(result: var string; value: YourType; specifier: string)
```

and it will automatically be picked up by `&` and `fmt` when a value of `YourType` appears in a format string.

```nim
type Point = object
  x, y: float

proc formatValue(result: var string; p: Point; specifier: string) =
  result.add fmt"({p.x:.2f}, {p.y:.2f})"

let p = Point(x: 1.5, y: -3.0)
echo &"Point: {p}"
# Point: (1.50, -3.00)
```

Built-in overloads exist for `SomeInteger`, `SomeFloat`, `string`, `char`, and `cstring`. For any other type, `formatValue` falls back to calling `$` on the value and then formatting the resulting string.

---

## `alignString`

```nim
proc alignString*(s: string, minimumWidth: int; align = '\0'; fill = ' '): string
```

A utility proc that pads string `s` to at least `minimumWidth` characters using `fill`, according to the `align` flag (`<`, `>`, `^`, or `'\0'` for right-align). This is a low-level building block exposed for authors writing custom `formatValue` implementations that need to honour the standard alignment specifiers.

```nim
assert alignString("hi", 8)        == "      hi"   # default: right
assert alignString("hi", 8, '<')   == "hi      "   # left
assert alignString("hi", 8, '^')   == "   hi   "   # center
assert alignString("hi", 8, '<', '-') == "hi------"  # custom fill
```

---

## `parseStandardFormatSpecifier`

```nim
proc parseStandardFormatSpecifier*(s: string; start = 0;
                                   ignoreUnknownSuffix = false): StandardFormatSpecifier
```

Parses a format-specifier string (the part after the colon in `{expr:specifier}`) and returns a `StandardFormatSpecifier` object. The optional `start` parameter lets you begin parsing at an offset within `s`. If `ignoreUnknownSuffix` is `true`, any trailing characters after the recognized fields are silently ignored rather than raising a `ValueError`.

This proc is intended for authors writing custom `formatValue` procs that want to support the same specifier syntax as the built-in types — parse the specifier once, then use the resulting object's fields.

```nim
let spec = parseStandardFormatSpecifier(">10.3f")
assert spec.align     == '>'
assert spec.minimumWidth == 10
assert spec.precision == 3
assert spec.typ       == 'f'
```

---

## `StandardFormatSpecifier`

```nim
type StandardFormatSpecifier* = object
  fill*:         char  # padding character (default: space)
  align*:        char  # '<', '>', '^', or '\0'
  sign*:         char  # '+', '-', or ' '
  alternateForm*: bool # true if '#' was present
  padWithZero*:  bool  # true if '0' prefix was present
  minimumWidth*: int   # 0 means "no minimum"
  precision*:    int   # -1 means "not specified"
  typ*:          char  # 'f', 'g', 'e', 'd', 'x', etc., or '\0'
  endPosition*:  int   # index in the source string after the last parsed char
```

The structured representation of a parsed format specifier. All fields are exported and can be read directly after calling `parseStandardFormatSpecifier`. Custom `formatValue` implementations inspect these fields to reproduce the same alignment and number-formatting behaviour as the built-in types.

---

## Limitations

### Template arguments cannot be interpolated

Because `fmt`/`&` are macros, they expand at compile time and resolve identifiers from the lexical scope of the call site. Template parameters are substituted at a *different* expansion stage, after the macro has already frozen the identifiers it knows about. As a result, a template parameter name inside a format string will not resolve:

```nim
# This does NOT work:
template myTemplate(arg: untyped): untyped =
  echo &"--- {arg} ---"   # `arg` is not visible to & at macro-expansion time

# Workaround: bind to a local variable first
template myTemplate(arg: untyped): untyped =
  block:
    let arg1 {.inject.} = arg
    echo &"--- {arg1} ---"
```

The `{.inject.}` pragma is needed because templates are hygienic by default; the `block` wrapper prevents `arg1` from leaking into the caller's scope.

### Only string literals are accepted

`fmt` and `&` require a **compile-time constant** string. You cannot build a format string at runtime and pass it to `fmt`:

```nim
let pattern = "{x}"   # runtime variable
# echo fmt(pattern)   ← compile error: not a static string
```

For runtime format strings you need a different approach (e.g. manual substitution with `strutils.replace` or a purpose-built library).

---

## Quick-Reference Cheatsheet

### Specifier grammar recap

```
{expr}
{expr:[[fill]align][sign][#][0][width][.precision][type]}
{expr=}           ← debug: prints "expr=value"
{expr=:specifier} ← debug with format specifier
```

### Alignment

| Spec | Effect |
|---|---|
| `{v:10}` | Right-align in field of 10 (default for numbers) |
| `{v:<10}` | Left-align in field of 10 (default for strings) |
| `{v:^10}` | Center in field of 10 |
| `{v:*^10}` | Center, fill with `*` |
| `{v:010}` | Zero-pad to width 10 |

### Integers

| Spec | Example output |
|---|---|
| `{255:d}` or `{255}` | `255` |
| `{255:b}` | `11111111` |
| `{255:o}` | `377` |
| `{255:x}` | `ff` |
| `{255:X}` | `FF` |
| `{255:#x}` | `0xff` |
| `{-1:05}` | `-0001` |

### Floats

| Spec | Example output |
|---|---|
| `{1.5:.2f}` | `1.50` |
| `{1.5:e}` | `1.500000e+00` |
| `{1.5:.3g}` | `1.5` |
| `{1.5:+.1f}` | `+1.5` |
| `{1.5:08.2f}` | `00001.50` |

### Strings

| Spec | Example output |
|---|---|
| `{"hi":>10}` | `        hi` |
| `{"hi":<10}` | `hi        ` |
| `{"hello":.3}` | `hel` |

### Escaping and delimiters

| Pattern | Output |
|---|---|
| `fmt"{{" ` | `{` |
| `fmt"}}"` | `}` |
| `"x".fmt('<','>')` | uses `<` `>` as delimiters |

### Key proc summary

| Name | Purpose |
|---|---|
| `fmt"..."` | Format with raw string literal (no escape sequences) |
| `&"..."` | Format with regular string literal (escape sequences work) |
| `alignString(s, w, align, fill)` | Pad a string to a minimum width |
| `parseStandardFormatSpecifier(s)` | Parse a specifier string into a struct |
| `formatValue(result, value, spec)` | Low-level hook; override for custom types |
