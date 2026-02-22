# std/strscans — Module Reference

> Pattern-based string scanning for Nim. Provides `scanf` and `scanp` macros
> for extracting structured data from strings — often simpler and faster than
> regular expressions, with no runtime overhead since patterns are compiled at
> compile time.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [`scanf` — Pattern Macro](#scanf--pattern-macro)
   - [Built-in Format Tokens](#built-in-format-tokens)
   - [Literal Matching](#literal-matching)
   - [Full-match vs. Prefix-match](#full-match-vs-prefix-match)
   - [No Backtracking](#no-backtracking)
3. [`scanTuple` — Tuple-returning Variant](#scantuple--tuple-returning-variant)
4. [User-defined Matchers](#user-defined-matchers)
   - [`${proc}` — Binding Matcher](#proc--binding-matcher)
   - [`$[proc]` — Skipping Matcher](#proc--skipping-matcher)
   - [Passing Arguments to Matchers](#passing-arguments-to-matchers)
5. [`scanp` — Grammar-based Macro](#scanp--grammar-based-macro)
   - [Grammar Operators](#grammar-operators)
   - [`$_` and Bind Actions (`->`)](#_-and-bind-actions---)
   - [Calling Nim Procs Inside `scanp`](#calling-nim-procs-inside-scanp)
   - [Custom Input Types](#custom-input-types)
6. [Helper Templates for Custom Types](#helper-templates-for-custom-types)
7. [Combining `scanf` and `scanp`](#combining-scanf-and-scanp)
8. [Quick-Reference Summary](#quick-reference-summary)

---

## Core Concepts

`std/strscans` gives you two complementary scanning tools:

**`scanf`** works similarly to C's `scanf`: you supply a *pattern string* with `$`-prefixed tokens interspersed with literal text, and it tries to match the input from left to right, binding captured values to your variables. The pattern is a compile-time constant, so the macro rewrites it into efficient inline code — there is no pattern interpreter at runtime.

**`scanp`** expresses patterns using Nim expression syntax with PEG/EBNF-like operators (`*`, `+`, `?`, `~`, `{n,m}`, `->`, etc.). It is more powerful and flexible but also more verbose. It also takes a mutable index variable, making it ideal for incremental scanning in a loop.

Both macros perform **no backtracking**: once a part of the pattern has consumed input, that consumption is never undone. This makes them predictable and fast, but means you need to design your patterns carefully.

---

## `scanf` — Pattern Macro

```nim
macro scanf*(input: string; pattern: static[string]; results: varargs[typed]): bool
```

Tries to match `input` against `pattern`. Returns `true` if the match succeeds (i.e. the pattern is satisfied starting from the beginning of `input`). Captured values are written into the `results` variables in the order they appear in the pattern.

```nim
var x, y, z: int
if scanf("(1,2,4)", "($i,$i,$i)", x, y, z):
  echo x, " ", y, " ", z   # 1 2 4
```

The pattern is a **compile-time constant** — either a string literal or a `const`. Nim resolves it at compile time and generates the corresponding parsing calls directly, so there is no runtime overhead from interpreting the format string.

---

### Built-in Format Tokens

Each token begins with `$` followed by a letter. The token consumes a substring from the input and — where applicable — writes a parsed value into the next variable in the `results` list.

| Token | Captures | Description |
|---|---|---|
| `$i` | `int` | Decimal integer. Uses `parseutils.parseInt`. |
| `$b` | `int` | Binary integer (e.g. `0b1010`). Uses `parseutils.parseBin`. |
| `$o` | `int` | Octal integer (e.g. `0o17`). Uses `parseutils.parseOct`. |
| `$h` | `int` | Hexadecimal integer (e.g. `0xFF`). Uses `parseutils.parseHex`. |
| `$f` | `float` | Floating-point number. Uses `parseutils.parseFloat`. |
| `$w` | `string` | ASCII identifier: `[A-Za-z_][A-Za-z_0-9]*`. |
| `$c` | `char` | Single ASCII character. |
| `$s` | — | Skips zero or more whitespace characters. Never fails. |
| `$$` | — | Matches a literal `$` character. |
| `$.` | — | Succeeds only if the end of the input has been reached. Must appear last. |
| `$*` | `string` | Matches until the literal token *after* `$*` is found. Zero-length match is allowed. |
| `$+` | `string` | Same as `$*`, but requires at least one character. |
| `${proc}` | depends | Calls a user-defined matching proc. See [User-defined Matchers](#user-defined-matchers). |
| `$[proc]` | — | Calls a user-defined *skipping* proc (no variable binding). See below. |

```nim
var year, month, day: int
var name: string
var temp: float

# Parse "2024-03-15 London 18.5"
if scanf("2024-03-15 London 18.5", "$i-$i-$i $w$s$f",
         year, month, day, name, temp):
  echo year, "-", month, "-", day   # 2024-3-15
  echo name, " ", temp              # London 18.5
```

---

### `$*` and `$+` — Match Until a Delimiter

These two tokens capture everything up to the next *literal* string in the pattern. They use `parseutils.parseUntil` internally.

```nim
var proto, host, path: string

# Parse a simple URL: "https://example.com/index.html"
if scanf("https://example.com/index.html", "$*://$*$+", proto, host, path):
  echo proto   # https
  echo host    # example.com
  echo path    # /index.html
```

`$*` accepts an empty match (the delimiter is found immediately), while `$+` requires at least one character before the delimiter. If no delimiter follows in the pattern, the token matches to the end of the string.

---

### Literal Matching

Any part of the pattern that does not start with `$` is matched verbatim against the input. The match is case-sensitive. If the literal text is not found at the current position, the whole `scanf` returns `false`.

```nim
var n: int
# The literal "item:" must be present
if scanf("item:42", "item:$i", n):
  echo n   # 42
```

---

### Full-match vs. Prefix-match

By default `scanf` returns `true` as soon as the pattern is satisfied, even if there are characters left in the input. Append `$.` to the pattern to require that the entire input is consumed.

```nim
var n: int

# Prefix match — succeeds even with trailing characters
if scanf("42abc", "$i", n): echo "prefix ok: ", n   # 42

# Full match — requires nothing left after the number
if scanf("42", "$i$.", n): echo "full ok: ", n       # 42
if scanf("42abc", "$i$.", n): echo "never"           # does NOT print
```

`$.` must be the very last element of the pattern. Placing it elsewhere is a compile-time error.

---

### No Backtracking

Once `scanf` has consumed input for a token, it does not backtrack if a later token fails. If a bound variable was partially written, it keeps its new value — not the original. This is rarely a problem in practice, but keep it in mind when designing patterns with overlapping possibilities:

```nim
# FRAGILE: if $* matches too much, the literal "x" will not be found
# and scanf returns false, but the variable 'a' may have been modified.
var a, b: string
discard scanf("helloworld", "$*x$+", a, b)
# Safe pattern: bind to a temp first, then validate
```

---

## `scanTuple` — Tuple-returning Variant

```nim
macro scanTuple*(input: untyped; pattern: static[string];
                 matcherTypes: varargs[untyped]): untyped
```

Works identically to `scanf`, but instead of requiring pre-declared variables it returns a tuple. The first element of the tuple is a `bool` indicating success; the remaining elements are the captured values, inferred from the pattern tokens.

```nim
let (ok, year, month, day, time) =
  scanTuple("1000-01-01 00:00:00", "$i-$i-$i$s$+")

if ok:
  echo year, "-", month, "-", day   # 1000-1-1
  echo time                          # 00:00:00
```

**Type inference:** the tuple element types are determined automatically from the pattern (`$i` → `int`, `$f` → `float`, `$w`/`$*`/`$+` → `string`, `$c` → `char`).

**User-defined matchers:** if the pattern includes `${yourMatcher()}`, the tuple cannot infer the captured type automatically. Provide the expected types as additional arguments to `scanTuple`:

```nim
let (ok, val) = scanTuple(line, "${myMatcher()}", int)
```

---

## User-defined Matchers

One of the most powerful features of `scanf` is that you can extend it with ordinary Nim procs. Two syntactic forms exist:

---

### `${proc}` — Binding Matcher

```
${procName}
${procName(extraArg1, extraArg2, ...)}
```

The proc is called with the input string, the variable to fill, and the current position. It must return the number of characters consumed (0 means no match).

**Required signature:**

```nim
proc myMatcher(input: string; result: var T; start: int): int
```

where `T` is the type of the variable that will receive the captured value.

```nim
proc parseWord(input: string; word: var string; start: int): int =
  result = 0
  while start + result < input.len and input[start + result] != ' ':
    word.add input[start + result]
    inc result
  # return 0 means "no match"; any positive value means "consumed N chars"

var w: string
if scanf("hello world", "${parseWord} world", w):
  echo w   # hello
```

---

### `$[proc]` — Skipping Matcher

```
$[procName]
$[procName(extraArg1, extraArg2, ...)]
```

Like `${proc}`, but does **not** bind any variable — the matched characters are simply consumed and discarded. The proc still returns the number of characters consumed (0 to skip nothing).

**Required signature:**

```nim
proc mySkipper(input: string; start: int): int
# or with extra args:
proc mySkipper(input: string; start: int; extraArg: SomeType): int
```

```nim
proc skipSeps(input: string; start: int;
              seps: set[char] = {':', '-', '.'}): int =
  while start + result < input.len and input[start + result] in seps:
    inc result

var key, value: string
# Skip any combination of ':', '-', '.' between key and value
if scanf("key---value", "$w$[skipSeps]$w", key, value):
  echo key, " -> ", value   # key -> value
```

---

### Passing Arguments to Matchers

Extra arguments can be passed inside the `{}` or `[]` after the proc name. They are appended to the call *after* the mandatory `input`, `result`/`start` arguments.

```nim
proc ndigits(input: string; intVal: var int; start: int; n: int): int =
  ## Match exactly n decimal digits, return 0 if fewer are available.
  var x = 0
  var i = 0
  while i < n and i + start < input.len and input[i + start] in {'0'..'9'}:
    x = x * 10 + input[i + start].ord - '0'.ord
    inc i
  if i == n:
    intVal = x
    result = n
  # else result stays 0 → no match

var year, month, day: int
# Matches exactly yyyy-mm-dd and nothing more
if scanf("2024-03-15", "${ndigits(4)}-${ndigits(2)}-${ndigits(2)}$.",
         year, month, day):
  echo year, month, day   # 2024 3 15
```

---

## `scanp` — Grammar-based Macro

```nim
macro scanp*(input, idx: typed; pattern: varargs[untyped]): bool
```

A more expressive alternative to `scanf`. Instead of a string pattern, `scanp` takes Nim expressions using PEG/EBNF-inspired operators. It modifies `idx` in place to track the current position, making it ideal for incremental scanning inside a loop.

```nim
var idx = 0
var content = "hello world\nfoo bar\n"
var line = ""
while scanp(content, idx, +(~{'\L', '\0'} -> line.add($_)), '\L'):
  echo line
  line = ""
```

---

### Grammar Operators

| Syntax | Meaning |
|---|---|
| `'c'` | Match a single character literal |
| `{'a'..'z'}` | Match a character in a set/range |
| `"str"` | Match the literal string `str` |
| `(E)` | Grouping |
| `*E` | Zero or more of E |
| `+E` | One or more of E |
| `?E` | Zero or one of E (optional) |
| `~E` | Negative predicate: succeed if E does *not* match (and consume nothing) |
| `E{n,m}` | Match E at least `n` and at most `m` times |
| `E{n}` | Match E exactly `n` times |
| `a ^* b` | Shorthand for `?(a *(b a))` — list of `a` separated by `b` |
| `a ^+ b` | Shorthand for `(a *(b a))` — non-empty list of `a` separated by `b` |
| `E -> action` | Execute `action` for each character consumed by E |
| `$_` | Inside an action: refers to the currently matched character |
| `$input` | Inside an action: the whole input |
| `$index` | Inside an action: the current index into the input |
| `procName` | Call proc as a non-terminal (returns chars consumed) |

Patterns are passed as comma-separated Nim expressions — the comma acts as *sequence* (concatenation). All elements must match in order.

---

### `$_` and Bind Actions (`->`)

The `->` infix operator binds a *side effect action* to a sub-expression. `$_` inside the action refers to the currently matched character. This is how you accumulate characters into a string:

```nim
var word = ""
var idx = 0
# Collect all characters until a space or end-of-string
discard scanp("hello world", idx,
              +(~{' ', '\0'} -> word.add($_)))
echo word   # hello
```

Multiple actions can be chained. The action runs once for each character the sub-expression consumes.

---

### Calling Nim Procs Inside `scanp`

Regular Nim procs can be called as *non-terminals* inside `scanp`. The proc signature must be:

```nim
proc myProc(input: InputType; start: int): int
# Returns number of chars consumed, or 0 for no match
```

When called inside `scanp`, the result is passed to the built-in `success` template: any non-zero return is a match, zero is a failure.

```nim
proc skipUntilHref(s: string; until: string; unless: char; start: int): int =
  # returns chars to skip, or 0 if `unless` was found first
  ...

var href = ""
var idx, old = 0
while idx < html.len:
  old = idx
  if scanp(html, idx, "<a",
           skipUntilHref($input, "href=", '>', $index),
           {'\'', '"'},
           *(~{'\'', '"'} -> href.add($_))):
    echo href
    href = ""
  idx = old + 1
```

---

### Custom Input Types

By default `atom` and `nxt` are defined for `string`. You can overload these templates for other input types (e.g. `Stream`, a custom tokenizer) to make `scanp` work on them:

```nim
import std/streams

template atom(input: Stream; idx: int; c: char): bool =
  peekChar(input) == c

template atom(input: Stream; idx: int; s: set[char]): bool =
  peekChar(input) in s

template nxt(input: Stream; idx: int; step: int = 1) =
  inc(idx, step)
  setPosition(input, idx)
```

---

## Helper Templates for Custom Types

These templates are exported by the module to support custom input types with `scanp`. They are also used internally by the string implementation.

### `atom` (string overloads)

```nim
template atom*(input: string; idx: int; c: char): bool
template atom*(input: string; idx: int; s: set[char]): bool
```

Tests whether the character at position `idx` in `input` matches `c` or belongs to the set `s`. At end-of-string (`idx == input.len`), `'\0'` is matched. These are the primitives that character and set literals in `scanp` expand into.

### `hasNxt`

```nim
template hasNxt*(input: string; idx: int): bool
```

Returns `true` if there are more characters to consume (`idx < input.len`). Used by the `*` and `+` operators in `scanp` to guard their loop.

### `nxt`

```nim
template nxt*(input: string; idx: int; step: int = 1)
```

Advances `idx` by `step` positions. Called after each successful atom match to move the position forward.

### `success`

```nim
template success*(x: int): bool
```

Returns `true` if `x != 0`. This is how proc-based non-terminals report success to `scanp` — a proc returns the number of characters it consumed, and `success` converts that to a boolean condition.

---

## Combining `scanf` and `scanp`

The two macros complement each other well. A common pattern is to use `scanp` for iterative, character-level scanning (e.g., collecting a candidate substring), then use `scanf` to parse and validate the structure of that candidate:

```nim
iterator parseIps(soup: string): string =
  const digits = {'0'..'9'}
  var a, b, c, d: int
  var buf = ""
  var idx = 0
  while idx < soup.len:
    # scanp: collect a candidate that looks like it could be an IP
    if scanp(soup, idx,
             (`digits`{1,3}, '.', `digits`{1,3}, '.',
              `digits`{1,3}, '.', `digits`{1,3}) -> buf.add($_)):
      # scanf: validate ranges of each octet
      if buf.scanf("$i.$i.$i.$i", a, b, c, d):
        if a in 0..254 and b in 0..254 and c in 0..254 and d in 0..254:
          yield buf
    buf.setLen(0)
    inc idx
```

---

## Quick-Reference Summary

### `scanf` pattern tokens

| Token | Captured type | Meaning |
|---|---|---|
| `$i` | `int` | Decimal integer |
| `$b` | `int` | Binary integer |
| `$o` | `int` | Octal integer |
| `$h` | `int` | Hexadecimal integer |
| `$f` | `float` | Floating-point number |
| `$w` | `string` | ASCII identifier |
| `$c` | `char` | Single character |
| `$s` | — | Skip whitespace (never fails) |
| `$$` | — | Match literal `$` |
| `$.` | — | Assert end of input (must be last) |
| `$*` | `string` | Match until next literal (0+ chars) |
| `$+` | `string` | Match until next literal (1+ chars) |
| `${proc}` | depends | User-defined binding matcher |
| `$[proc]` | — | User-defined skipping matcher |

### `scanp` operators

| Syntax | Meaning |
|---|---|
| `'c'` | Single char literal |
| `{'a'..'z'}` | Char set / range |
| `"str"` | String literal |
| `*E` | Zero or more |
| `+E` | One or more |
| `?E` | Optional |
| `~E` | Not predicate |
| `E{n,m}` | n to m repetitions |
| `a ^* b` | Separated list (0+) |
| `a ^+ b` | Separated list (1+) |
| `E -> action` | Execute action per char |
| `$_` | Current matched char |

### Macro signatures

```nim
macro scanf*(input: string; pattern: static[string];
             results: varargs[typed]): bool

macro scanTuple*(input: untyped; pattern: static[string];
                 matcherTypes: varargs[untyped]): untyped  # → tuple

macro scanp*(input, idx: typed; pattern: varargs[untyped]): bool
```

### User-defined matcher signatures

```nim
# Binding matcher — used with ${procName}
proc myMatcher(input: string; value: var T; start: int): int

# Binding matcher with extra args — used with ${procName(arg1)}
proc myMatcher(input: string; value: var T; start: int; arg1: A): int

# Skipping matcher — used with $[procName]
proc mySkipper(input: string; start: int): int

# scanp non-terminal
proc myProc(input: string; start: int): int
```

### Key behavioral notes

- Patterns are compile-time constants — no runtime format parsing.
- `scanf` performs **prefix matching** by default; append `$.` for a full match.
- **No backtracking**: once input is consumed, it is not restored on failure.
- `scanp` takes a mutable `idx` and advances it, enabling incremental scanning in loops.
- Both macros expand to plain Nim parsing calls — no interpreter overhead.
