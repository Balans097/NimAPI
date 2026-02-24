# `cstrutils` Module Reference

> **Module:** `std/cstrutils`  
> **Purpose:** Utility functions for working with `cstring` values directly, without converting them to `string`, in order to avoid heap allocations.

---

## Overview

In Nim, `cstring` is the "C string" type — a raw, null-terminated pointer to a character sequence. It is the native string representation used by C libraries and by JavaScript runtimes. Operations on `cstring` are low-level: the standard `string` module does not cover them, because `string` and `cstring` are distinct types.

The challenge with `cstring` is that converting it to a `string` (e.g., `$myCstring`) triggers a heap allocation, which copies all the bytes into a freshly allocated Nim string object. In hot paths — tight loops, FFI callbacks, embedded systems, or high-throughput servers — this overhead is undesirable.

`cstrutils` solves this by providing common string predicates and comparisons that operate directly on `cstring` values, paying no allocation cost whatsoever.

### When to prefer `cstrutils` over `strutils`

| Situation | Use |
|---|---|
| Working with data received from C FFI (`const char*`) | `cstrutils` |
| Writing a JavaScript backend and want native JS string methods | `cstrutils` |
| Need to avoid any allocation in a hot path | `cstrutils` |
| Working with ordinary Nim `string` values | `strutils` |
| Need richer operations (split, replace, strip, etc.) | Convert to `string`, use `strutils` |

### Cross-platform notes

All four functions in this module work across three execution contexts:

- **Native (C backend)** — hand-rolled loops operating on raw memory, zero allocations.
- **JavaScript backend** — `startsWith` and `endsWith` delegate to the native `String.prototype.startsWith` / `String.prototype.endsWith` JavaScript methods; `cmpIgnoreStyle` and `cmpIgnoreCase` use a shared implementation.
- **`nimvm` (compile-time evaluation)** — shared implementation used so the functions can be called in `static:` blocks and `const` expressions.

---

## Exported Symbols

---

### `startsWith`

```nim
func startsWith*(s, prefix: cstring): bool
```

#### What it does

Returns `true` if the `cstring` `s` begins with the exact byte sequence given by `prefix`. The comparison is **case-sensitive** and **byte-exact**.

#### Edge cases

- If `prefix` is an empty string (`cstring""`), the function always returns `true`, regardless of what `s` contains. This is consistent with the mathematical definition: every string is a prefix of itself, and the empty string is a prefix of every string.
- If `s` itself is empty but `prefix` is not, the function returns `false`.

#### How it works (native backend)

The implementation walks both strings in lockstep, character by character. The moment `prefix` reaches its null terminator (`'\0'`), all prefix characters have been matched and the function returns `true`. If any character differs before that point, it returns `false` immediately. There is no length calculation, no allocation, and no branching overhead beyond the per-character comparison.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `s` | `cstring` | The string to examine. |
| `prefix` | `cstring` | The prefix to look for at the beginning of `s`. |

#### Examples

```nim
import std/cstrutils

# Basic positive match
assert startsWith(cstring"Hello, World", cstring"Hello")

# No match — prefix appears in the middle, not at the start
assert not startsWith(cstring"Hello, World", cstring"World")

# Empty prefix always matches
assert startsWith(cstring"anything", cstring"")
assert startsWith(cstring"", cstring"")

# Case sensitivity: "hello" ≠ "Hello"
assert not startsWith(cstring"hello, world", cstring"Hello")
```

```nim
import std/cstrutils

# Practical pattern: routing an HTTP request path without allocation
proc routeRequest(path: cstring) =
  if startsWith(path, cstring"/api/"):
    echo "API route"
  elif startsWith(path, cstring"/static/"):
    echo "Static file"
  else:
    echo "Unknown route"

routeRequest(cstring"/api/users")   # → API route
routeRequest(cstring"/static/app.js") # → Static file
```

---

### `endsWith`

```nim
func endsWith*(s, suffix: cstring): bool
```

#### What it does

Returns `true` if the `cstring` `s` ends with the exact byte sequence given by `suffix`. The comparison is **case-sensitive** and **byte-exact**.

#### Edge cases

- An empty `suffix` always yields `true` — the empty string is a suffix of every string.
- If `suffix` is longer than `s`, the function returns `false`.

#### How it works (native backend)

The implementation first calculates the lengths of both `s` and `suffix`. It then starts comparing from the position `len(s) - len(suffix)` onward — that is, from the point in `s` where a match would have to begin if the suffix is present. This avoids scanning the entire string and goes straight to the candidate region. The unsigned comparison `i + j <% slen` is used to handle edge cases involving zero-length strings safely.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `s` | `cstring` | The string to examine. |
| `suffix` | `cstring` | The suffix to look for at the end of `s`. |

#### Examples

```nim
import std/cstrutils

# Basic positive match
assert endsWith(cstring"image.png", cstring".png")

# No match
assert not endsWith(cstring"image.png", cstring".jpg")

# Empty suffix always matches
assert endsWith(cstring"anything", cstring"")

# Case sensitive
assert not endsWith(cstring"file.PNG", cstring".png")
```

```nim
import std/cstrutils

# Practical pattern: file type detection without allocation
proc describeFile(name: cstring) =
  if endsWith(name, cstring".nim"):
    echo "Nim source file"
  elif endsWith(name, cstring".c") or endsWith(name, cstring".h"):
    echo "C source or header"
  elif endsWith(name, cstring".json"):
    echo "JSON data"
  else:
    echo "Unknown file type"

describeFile(cstring"main.nim")    # → Nim source file
describeFile(cstring"bindings.h")  # → C source or header
```

---

### `cmpIgnoreCase`

```nim
func cmpIgnoreCase*(a, b: cstring): int
```

#### What it does

Compares two `cstring` values **ignoring ASCII letter case**. The return value follows the standard three-way comparison convention used throughout Nim and C:

| Return value | Meaning |
|---|---|
| `0` | `a` and `b` are equal when case is ignored |
| `< 0` | `a` comes before `b` lexicographically (case-insensitive) |
| `> 0` | `a` comes after `b` lexicographically (case-insensitive) |

#### What "ignoring case" means here

Only ASCII letters (`A`–`Z` / `a`–`z`) are folded. The function calls `toLowerAscii` on each character before comparing. Non-ASCII bytes are compared by their raw byte values. This is **not** Unicode-aware — for Unicode case folding, you would need a different library.

#### How it works (native backend)

The function walks both strings in parallel, converting each character to lowercase ASCII before comparing. If characters differ, it returns their difference immediately. If both characters are the null terminator at the same time, the strings are equal and it returns `0`.

#### When to use it

- Comparing command-line flags, HTTP headers, or configuration keys where case should not matter but Unicode complexity is not needed.
- Sorting a list of `cstring` values case-insensitively.
- Checking user input against a known keyword.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `a` | `cstring` | First string to compare. |
| `b` | `cstring` | Second string to compare. |

#### Examples

```nim
import std/cstrutils

# Equal despite different casing
assert cmpIgnoreCase(cstring"hello", cstring"HeLLo") == 0
assert cmpIgnoreCase(cstring"NIM", cstring"nim") == 0

# Lexicographic order: "echo" < "hello" ('e' < 'h')
assert cmpIgnoreCase(cstring"echo", cstring"hello") < 0

# "yellow" > "hello" ('y' > 'h')
assert cmpIgnoreCase(cstring"yellow", cstring"hello") > 0
```

```nim
import std/cstrutils, std/algorithm

# Sorting a seq[cstring] case-insensitively
var words = @[cstring"Zebra", cstring"apple", cstring"Mango", cstring"banana"]
words.sort(proc(a, b: cstring): int = cmpIgnoreCase(a, b))
# Result order: apple, banana, Mango, Zebra
for w in words: echo w
```

```nim
import std/cstrutils

# Matching an HTTP method regardless of casing
proc handleMethod(meth: cstring) =
  if cmpIgnoreCase(meth, cstring"GET") == 0:
    echo "GET request"
  elif cmpIgnoreCase(meth, cstring"POST") == 0:
    echo "POST request"

handleMethod(cstring"get")   # → GET request
handleMethod(cstring"Post")  # → POST request
```

---

### `cmpIgnoreStyle`

```nim
func cmpIgnoreStyle*(a, b: cstring): int
```

#### What it does

Compares two `cstring` values while ignoring both **ASCII letter case** and **underscore characters**. This is semantically equivalent to calling `cmp(normalize($a), normalize($b))` from `strutils`, but operates directly on `cstring` without any allocation.

The return value uses the same three-way convention as `cmpIgnoreCase`:

| Return value | Meaning |
|---|---|
| `0` | `a` and `b` are equal under style-insensitive comparison |
| `< 0` | `a` comes before `b` |
| `> 0` | `a` comes after `b` |

#### What "ignoring style" means

Nim's identifier style rules allow mixing `camelCase` and `snake_case` names interchangeably for identifiers. `cmpIgnoreStyle` embodies this rule: underscores are skipped entirely, and letters are compared case-insensitively. So `"helloWorld"`, `"hello_world"`, `"Hello_World"`, and `"HELLO_WORLD"` all compare as equal.

#### Important warning: do not use for Nim identifier comparison

The module doc explicitly warns: **do not use `cmpIgnoreStyle` to compare Nim identifier names in macros or code generation**. Use `macros.eqIdent` for that, because `eqIdent` handles additional edge cases specific to Nim's identifier rules.

#### How it works (native backend)

Before comparing each character, the function skips any `_` characters on both sides independently. Then it converts both characters to lowercase ASCII and subtracts their ordinal values. If the difference is non-zero or the character is the null terminator, the loop ends.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `a` | `cstring` | First string to compare. |
| `b` | `cstring` | Second string to compare. |

#### Examples

```nim
import std/cstrutils

# Underscores and case are both ignored
assert cmpIgnoreStyle(cstring"hello_world", cstring"helloWorld") == 0
assert cmpIgnoreStyle(cstring"hello", cstring"H_e_L_Lo") == 0
assert cmpIgnoreStyle(cstring"MY_CONST", cstring"myConst") == 0

# Strings that differ in actual letters are still not equal
assert cmpIgnoreStyle(cstring"foo", cstring"bar") != 0
```

```nim
import std/cstrutils

# Comparing configuration keys that might come in snake_case or camelCase
proc lookupConfig(key: cstring): string =
  let knownKeys = [cstring"max_connections", cstring"server_port", cstring"log_level"]
  let labels    = ["MaxConnections", "ServerPort", "LogLevel"]
  for i, k in knownKeys:
    if cmpIgnoreStyle(key, k) == 0:
      return labels[i]
  return "unknown"

echo lookupConfig(cstring"maxConnections")  # → MaxConnections
echo lookupConfig(cstring"SERVER_PORT")     # → ServerPort
echo lookupConfig(cstring"logLevel")        # → LogLevel
```

```nim
import std/cstrutils

# Verifying that two naming conventions refer to the same concept
let snakeCase  = cstring"user_account_id"
let camelCase  = cstring"userAccountId"
let screamCase = cstring"USER_ACCOUNT_ID"

assert cmpIgnoreStyle(snakeCase, camelCase)  == 0
assert cmpIgnoreStyle(snakeCase, screamCase) == 0
```

---

## Function Comparison Table

| Function | Ignores case | Ignores `_` | Returns | Allocation-free |
|---|---|---|---|---|
| `startsWith` | No | No | `bool` | Yes |
| `endsWith` | No | No | `bool` | Yes |
| `cmpIgnoreCase` | Yes | No | `int` (-/0/+) | Yes |
| `cmpIgnoreStyle` | Yes | Yes | `int` (-/0/+) | Yes |

---

## Common Pitfalls

**1. Case sensitivity of `startsWith` and `endsWith`**  
These two functions are strictly case-sensitive. `startsWith(cstring"Hello", cstring"hello")` returns `false`. If you need case-insensitive prefix/suffix checking, convert to `string` and use `strutils`, or do a manual `cmpIgnoreCase` on the relevant substring.

**2. Non-ASCII characters**  
All four functions operate on raw bytes. Case folding is ASCII-only (`A`–`Z`). If your `cstring` contains UTF-8 multi-byte sequences, the functions will not apply Unicode case rules — `"ñ"` and `"Ñ"` will not compare as equal.

**3. `cmpIgnoreStyle` is not for Nim identifier comparison**  
Despite sounding ideal for it, the module explicitly advises against using `cmpIgnoreStyle` to compare Nim identifiers in macro code. Use `macros.eqIdent` for that purpose.

**4. Null safety**  
`cstring` in Nim can be `nil`. Passing `nil` to any of these functions is undefined behaviour on the native backend (it will dereference a null pointer). Always ensure your `cstring` values are non-nil before calling these functions.

**5. The return value of comparison functions is not clamped to -1/0/1**  
`cmpIgnoreCase` and `cmpIgnoreStyle` return the raw difference of character ordinal values, which can be any integer, not just -1, 0, or 1. When using the result in a sort comparator, this is fine. When comparing against specific integers (other than 0), be aware that only `== 0` reliably means "equal".
