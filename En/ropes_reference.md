# ropes — Module Reference

> **Nim standard library** | `import std/ropes`

## What Is a Rope?

A **rope** is a tree-shaped data structure for representing strings. Instead of storing one contiguous block of characters, it builds a binary tree whose leaves hold ordinary Nim strings. Inner nodes carry only a combined `length`; they hold no characters themselves.

```
       [inner, len=11]
      /               \
[leaf "Hello, "]   [leaf "Nim!"]
```

The key insight is that **concatenation is O(1)**: joining two ropes creates a single new inner node that points to both trees — no characters are copied. The resulting string is materialised only when you call `$` or write to a file, at which point the tree is traversed once (O(n)).

This makes ropes ideal for **building large strings incrementally** — think code generators, template engines, log formatters, or any situation where you concatenate hundreds or thousands of pieces. With ordinary strings every `&` copies the left-hand side, leading to O(n²) behaviour overall. With ropes the same work is O(n).

**Trade-off:** Random character access (`r[i]`) walks the tree and is O(n) in the worst case, whereas for plain strings it is O(1). Ropes are not a drop-in replacement for strings; they are a specialised tool.

**`nil` is the empty rope.** There is no separate "empty" sentinel value — a `nil` `Rope` reference represents the empty string throughout the API.

---

## Table of Contents

1. [Types](#types)
   - [Rope](#rope)
2. [Constructors](#constructors)
   - [rope(string)](#ropestring)
   - [rope(int)](#ropeint)
   - [rope(float)](#ropefloat)
3. [Concatenation](#concatenation)
   - [& (Rope, Rope)](#-rope-rope)
   - [& (Rope, string)](#-rope-string)
   - [& (string, Rope)](#-string-rope)
   - [& (openArray[Rope])](#-openarrayrope)
   - [add(Rope, Rope)](#addrope-rope)
   - [add(Rope, string)](#addrope-string)
4. [Formatting](#formatting)
   - [% (format string)](#-format-string)
   - [addf](#addf)
5. [Access and Inspection](#access-and-inspection)
   - [len](#len)
   - [\[\] (character at index)](#-character-at-index)
   - [leaves (iterator)](#leaves-iterator)
   - [items (iterator)](#items-iterator)
6. [Output](#output)
   - [$ (to string)](#-to-string)
   - [write(File, Rope)](#writefile-rope)
   - [write(Stream, Rope)](#writestream-rope)
7. [File Comparison](#file-comparison)
   - [equalsFile(Rope, File)](#equalsfilrope-file)
   - [equalsFile(Rope, string)](#equalsfilrope-string)
8. [Leaf Cache](#leaf-cache)
   - [enableCache](#enablecache)
   - [disableCache](#disablecache)
9. [Complete Example](#complete-example)
10. [Quick Reference](#quick-reference)

---

## Types

### `Rope`

```nim
type Rope* {.acyclic.} = ref object
```

The rope itself. It is a reference type — assignment copies the reference, not the data. Because the structure is immutable, subtrees can be safely shared between multiple rope values without copying.

The **empty rope is `nil`**. All procedures in this module handle `nil` gracefully (e.g., `len(nil) == 0`, `nil & r == r`).

---

## Constructors

### `rope(string)`

```nim
proc rope*(s: string = ""): Rope
```

Converts a Nim string into a rope leaf. An empty string returns `nil` (the empty rope).

When the leaf cache is enabled (see [`enableCache`](#enablecache)), identical strings share the same leaf node in memory instead of allocating a new one.

```nim
let r = rope("Hello, world!")
echo $r  # Hello, world!

let empty = rope("")
echo empty == nil  # true — empty rope is nil
```

---

### `rope(int)`

```nim
proc rope*(i: BiggestInt): Rope
```

Converts an integer to its decimal string representation and wraps it in a rope. Accepts any integer type that fits in `BiggestInt`.

```nim
let r = rope(2024)
echo $r  # 2024

# Useful when building strings that mix text and numbers:
let msg = rope("Year: ") & rope(2024)
echo $msg  # Year: 2024
```

---

### `rope(float)`

```nim
proc rope*(f: BiggestFloat): Rope
```

Converts a floating-point number to its string representation and wraps it in a rope. The formatting follows Nim's default `$` conversion for floats.

```nim
let r = rope(3.14159)
echo $r  # 3.14159

let line = rope("Pi ≈ ") & rope(3.14159)
echo $line  # Pi ≈ 3.14159
```

---

## Concatenation

### `&` (Rope, Rope)

```nim
proc `&`*(a, b: Rope): Rope
```

Creates a new inner node whose left child is `a` and right child is `b`. **No characters are copied** — this is a pure pointer operation and runs in O(1). If either operand is `nil`, the other is returned directly (no new node is allocated).

```nim
let r = rope("Hello, ") & rope("world!")
echo $r  # Hello, world!

# nil is the identity element for &
let r2 = nil & rope("Nim")
echo $r2  # Nim
```

---

### `&` (Rope, string)

```nim
proc `&`*(a: Rope, b: string): Rope
```

Convenience overload — converts `b` to a rope then concatenates. Equivalent to `a & rope(b)`.

```nim
let r = rope("Hello, ") & "world!"
echo $r  # Hello, world!
```

---

### `&` (string, Rope)

```nim
proc `&`*(a: string, b: Rope): Rope
```

The mirror of the above — converts `a` to a rope then concatenates. Equivalent to `rope(a) & b`.

```nim
let r = "Hello, " & rope("world!")
echo $r  # Hello, world!
```

---

### `&` (openArray[Rope])

```nim
proc `&`*(a: openArray[Rope]): Rope
```

Concatenates an array or sequence of ropes left-to-right into a single rope. The prefix `&` operator syntax (without a left operand) is used to call this overload.

```nim
let parts = [rope("one"), rope(", "), rope("two"), rope(", "), rope("three")]
let r = &parts
echo $r  # one, two, three
```

---

### `add(Rope, Rope)`

```nim
proc add*(a: var Rope, b: Rope)
```

Mutates `a` in place by appending `b`. Equivalent to `a = a & b`. Useful when building a rope incrementally in a loop without introducing a separate result variable.

```nim
var r = rope("Lines:\n")
for i in 1..3:
  r.add(rope("  line ") & rope(i) & rope("\n"))
echo $r
# Lines:
#   line 1
#   line 2
#   line 3
```

---

### `add(Rope, string)`

```nim
proc add*(a: var Rope, b: string)
```

Same as `add(Rope, Rope)` but accepts a plain string on the right-hand side. Converts `b` to a rope internally.

```nim
var r = rope("Hello")
r.add(", world!")
echo $r  # Hello, world!
```

---

## Formatting

### `%` (format string)

```nim
proc `%`*(frmt: string, args: openArray[Rope]): Rope
```

Substitution operator — works like Nim's string `%` but produces a `Rope`. Supports three placeholder styles:

| Syntax | Meaning |
|---|---|
| `$1`, `$2`, … | Positional: insert `args[0]`, `args[1]`, … |
| `$#` | Sequential: each `$#` takes the next argument in order |
| `${1}`, `${2}`, … | Bracketed positional (same as `$1` but unambiguous) |
| `$$` | Literal dollar sign `$` |

> **Note:** The `$identifier` and `${identifier}` named-variable syntaxes are **not** supported. Only numeric indices work.

```nim
# Positional
let r1 = "$1 + $2 = $3" % [rope("1"), rope("2"), rope("3")]
echo $r1  # 1 + 2 = 3

# Sequential — cleaner when order matches
let r2 = "$# $# $#" % [rope("Nim"), rope("is"), rope("great")]
echo $r2  # Nim is great

# Mixing literal $ with positional
let r3 = "$$1 costs $1 USD" % [rope("5")]
echo $r3  # $1 costs 5 USD
```

---

### `addf`

```nim
proc addf*(c: var Rope, frmt: string, args: openArray[Rope])
```

Shortcut for `c.add(frmt % args)`. Appends a formatted result directly to an existing rope without creating a named intermediate variable. Useful inside loops or builders.

```nim
var report = rope("")
report.addf("User: $1\n", [rope("Alice")])
report.addf("Score: $1\n", [rope("95")])
echo $report
# User: Alice
# Score: 95
```

---

## Access and Inspection

### `len`

```nim
proc len*(a: Rope): int
```

Returns the total character length of the rope — the sum of all leaf string lengths. This is stored as a cached integer in every node, so it runs in **O(1)**. `len(nil) == 0`.

```nim
let r = rope("Hello") & rope(", ") & rope("world!")
echo r.len  # 13
echo len(nil)  # 0
```

---

### `[]` (character at index)

```nim
proc `[]`*(r: Rope, i: int): char
```

Returns the character at position `i` (zero-based). Returns `'\0'` if `i` is out of range (`i < 0` or `i >= r.len`) or if the rope is `nil`.

**Performance warning:** This operation traverses the tree and is **O(n) in the worst case**. If you need to read many individual characters, convert the rope to a string first with `$` and index the string instead.

```nim
let r = rope("Hello, Nim!")
echo r[0]   # H
echo r[7]   # N
echo r[50]  # '\0'  (out of range)
```

---

### `leaves` (iterator)

```nim
iterator leaves*(r: Rope): string
```

Yields each leaf string of the rope in left-to-right order, without materialising the full string. This is the most efficient way to process a rope's content when you need to handle it chunk by chunk (e.g., writing to a socket, hashing, streaming output).

The number of leaves equals the number of `rope(string)` calls used to build the rope, minus any sharing introduced by the cache.

```nim
let r = rope("Hello") & rope(", ") & rope("world!")
for leaf in r.leaves:
  stdout.write(leaf)  # writes leaf strings one by one
echo ""
# Output: Hello, world!

# Useful for hashing without allocating a full string:
import std/md5
var ctx: MD5Context
md5Init(ctx)
for leaf in r.leaves:
  md5Update(ctx, leaf)
```

---

### `items` (iterator)

```nim
iterator items*(r: Rope): char
```

Yields every individual character of the rope in order. Implemented by iterating over leaves and then over characters within each leaf. Convenient for character-by-character processing, but creates more iterator overhead than working with leaves directly.

```nim
let r = rope("abc") & rope("def")
var count = 0
for ch in r:
  if ch == 'a' or ch == 'e': inc count
echo count  # 2
```

---

## Output

### `$` (to string)

```nim
proc `$`*(r: Rope): string
```

Flattens the rope into a standard Nim string. Traverses all leaves in order and concatenates them. This is the only time in a rope's lifecycle when O(n) memory allocation and copying occurs. The resulting string has the exact same content as if all the pieces had been concatenated with `&` from the start.

```nim
let r = rope("Hello") & rope(", ") & rope("world!")
let s: string = $r
echo s       # Hello, world!
echo s.len   # 13
```

---

### `write(File, Rope)`

```nim
proc write*(f: File, r: Rope)
```

Writes the rope's content to an open `File` handle, leaf by leaf, without allocating the full concatenated string. This is the most memory-efficient way to output a large rope to a file.

```nim
let output = rope("Line 1\n") & rope("Line 2\n") & rope("Line 3\n")
var f = open("output.txt", fmWrite)
write(f, output)  # writes all leaves without constructing the full string
close(f)
```

---

### `write(Stream, Rope)`

```nim
proc write*(s: Stream, r: Rope)
```

Writes the rope's content to any `Stream` (file stream, string stream, network stream, etc.), leaf by leaf.

```nim
import std/streams
let r = rope("chunk1") & rope("chunk2")
var strm = newStringStream()
write(strm, r)
strm.setPosition(0)
echo strm.readAll()  # chunk1chunk2
```

---

## File Comparison

These procedures are not available on JavaScript and NimScript targets.

### `equalsFile(Rope, File)`

```nim
proc equalsFile*(r: Rope, f: File): bool
```

Returns `true` if the byte content of the already-open file `f` is identical to the content of rope `r`. The comparison is done incrementally using a 1 KB buffer — neither the full rope string nor the full file content needs to be loaded into memory at once. Efficient for verifying that a generated rope matches a reference file.

```nim
let r = rope("expected content\n")
var f = open("reference.txt", fmRead)
if equalsFile(r, f):
  echo "Match!"
close(f)
```

---

### `equalsFile(Rope, string)`

```nim
proc equalsFile*(r: Rope, filename: string): bool
```

Convenience overload — opens the file at `filename`, compares, then closes it. Returns `false` if the file does not exist.

```nim
let generated = rope("line one\n") & rope("line two\n")
if generated.equalsFile("expected.txt"):
  echo "Output matches expected file."
else:
  echo "Mismatch — writing new expected file."
  var f = open("expected.txt", fmWrite)
  write(f, generated)
  close(f)
```

---

## Leaf Cache

The cache is a per-thread splay tree of rope leaves. When enabled, calling `rope("some string")` first searches the cache for an existing leaf with the same data. If found, the existing node is reused; no new allocation occurs. This reduces memory usage when the same strings are turned into rope leaves many times — for example, repeated keywords in a code generator.

**Trade-off:** Cache lookups involve a tree traversal and string comparison, so runtime throughput decreases. Enable the cache only when memory pressure is more important than speed.

### `enableCache`

```nim
proc enableCache*()
```

Enables the leaf cache for the current thread. After this call, `rope(string)` will look up and potentially reuse existing leaf nodes.

```nim
enableCache()
let a = rope("keyword")
let b = rope("keyword")  # reuses the same leaf node as `a`
# a and b point to the same underlying Rope object
```

---

### `disableCache`

```nim
proc disableCache*()
```

Disables the cache and sets the cache tree to `nil`, allowing the garbage collector to reclaim its memory. New calls to `rope(string)` will allocate fresh nodes again.

```nim
disableCache()
# The GC can now collect all previously cached leaf nodes
```

---

## Complete Example

The following example demonstrates building a small HTML page as a rope, inspecting it, and writing it to a file — all without ever constructing the full intermediate string.

```nim
import std/ropes

proc tag(name: string, content: Rope): Rope =
  rope("<" & name & ">") & content & rope("</" & name & ">")

proc li(text: string): Rope =
  tag("li", rope(text))

var body: Rope
body.add(tag("h1", rope("Rope Demo")))

var list: Rope
for item in ["apples", "bananas", "cherries"]:
  list.add(li(item))
body.add(tag("ul", list))

let page = tag("html", tag("body", body))

echo "Length: ", page.len

# Write directly to file — no full string allocation needed
var f = open("page.html", fmWrite)
write(f, page)
close(f)

# Verify
if page.equalsFile("page.html"):
  echo "File written correctly."

# Inspect structure via leaves
echo "Leaf count:"
var n = 0
for _ in page.leaves: inc n
echo n
```

---

## Quick Reference

| Symbol | Signature | Cost | Description |
|---|---|---|---|
| `rope` | `rope(string)` → `Rope` | O(1) | Wrap string in a rope leaf |
| `rope` | `rope(int)` → `Rope` | O(1) | Integer → rope |
| `rope` | `rope(float)` → `Rope` | O(1) | Float → rope |
| `len` | `len(Rope)` → `int` | **O(1)** | Cached total length |
| `&` | `Rope & Rope` → `Rope` | **O(1)** | Concatenate two ropes |
| `&` | `Rope & string` → `Rope` | **O(1)** | Concatenate rope and string |
| `&` | `string & Rope` → `Rope` | **O(1)** | Concatenate string and rope |
| `&` | `&openArray[Rope]` → `Rope` | O(k) | Concatenate array of ropes |
| `add` | `add(var Rope, Rope)` | **O(1)** | Append rope in place |
| `add` | `add(var Rope, string)` | **O(1)** | Append string in place |
| `%` | `string % openArray[Rope]` → `Rope` | O(fmt) | Format with substitution |
| `addf` | `addf(var Rope, string, openArray[Rope])` | O(fmt) | Format and append in place |
| `[]` | `Rope[int]` → `char` | **O(n)** | Character at index |
| `leaves` | `iterator leaves(Rope)` → `string` | O(n) | Iterate over leaf strings |
| `items` | `iterator items(Rope)` → `char` | O(n) | Iterate over characters |
| `$` | `$(Rope)` → `string` | **O(n)** | Flatten to Nim string |
| `write` | `write(File, Rope)` | O(n) | Write to file (no alloc) |
| `write` | `write(Stream, Rope)` | O(n) | Write to stream (no alloc) |
| `equalsFile` | `equalsFile(Rope, File)` → `bool` | O(n) | Compare with open file |
| `equalsFile` | `equalsFile(Rope, string)` → `bool` | O(n) | Compare with file by path |
| `enableCache` | `enableCache()` | — | Enable leaf deduplication |
| `disableCache` | `disableCache()` | — | Disable cache, free memory |
