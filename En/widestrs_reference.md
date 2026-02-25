# `widestrs` Module Reference

> Nim support for C/C++ wide strings (UTF-16 encoded character sequences).  
> This module is part of Nim's runtime library and primarily serves as a bridge between Nim's native UTF-8 strings and the UTF-16 strings required by the Windows API and other C/C++ interfaces.

---

## Background: Why wide strings exist

Most of the world stores text in **UTF-8** — a variable-width encoding where ASCII characters take 1 byte and everything else takes 2–4 bytes. Nim's native `string` type is UTF-8.

Windows, however, was built around **UTF-16**, where every code point takes 2 bytes (or 4 bytes for characters outside the Basic Multilingual Plane, using *surrogate pairs*). The Windows API's "wide" (`W`) variants of functions — `CreateFileW`, `MessageBoxW`, etc. — all expect UTF-16 strings terminated by a null 16-bit word.

This module provides the types and conversion procedures needed to cross that boundary.

---

## Unicode concepts used in this module

| Term | Meaning |
|------|---------|
| **Code point** | A Unicode scalar value, e.g. U+0041 = `A`, U+1F600 = 😀 |
| **BMP** | Basic Multilingual Plane — code points U+0000 to U+FFFF, fit in one UTF-16 unit |
| **Surrogate pair** | Two 16-bit units (high + low) encoding a code point above U+FFFF |
| **Replacement character** | U+FFFD `<EFBFBD>` — substituted when an invalid sequence is encountered |

---

## Exported types

### `Utf16Char`

```nim
type Utf16Char* = distinct int16
```

**What it is:** A single UTF-16 code unit — a 16-bit integer treated as a distinct type so the compiler prevents accidental mixing with plain `int16`. Each `Utf16Char` represents either a BMP character directly or one half of a surrogate pair.

---

### `WideCString`

```nim
# nimv2 (ARC/ORC GC):
type WideCString* = ptr UncheckedArray[Utf16Char]

# Legacy GC:
type WideCString* = ref UncheckedArray[Utf16Char]
```

**What it is:** A raw pointer (or managed reference, depending on the GC) to a null-terminated array of `Utf16Char` values. This is the type you pass directly to C/C++ functions expecting `wchar_t*` or `LPWSTR`.

Because `WideCString` is a raw pointer in `nimv2`, it carries no ownership information and no automatic lifetime management — you should prefer `WideCStringObj` for most Nim-side work.

---

### `WideCStringObj`

```nim
# nimv2 only:
type WideCStringObj* = object
  bytes: int
  data: WideCString
```

**What it is:** An owning wrapper around a `WideCString`. It manages the memory of the underlying UTF-16 buffer — allocating on construction and freeing in its destructor. Copy is intentionally disabled (`=copy` is marked `{.error.}`); the object can only be moved.

In the legacy GC build, `WideCStringObj` is simply an alias for `WideCString` (a ref type), so the GC manages the lifetime.

**Key properties:**
- Supports `[]` and `[]=` for indexed access to individual `Utf16Char` elements.
- Implicitly converts to `WideCString` via the `toWideCString` converter, so you can pass it wherever a raw pointer is expected.
- Thread-safe allocation: uses `allocShared0`/`deallocShared` when compiled with `--threads:on`.

---

## Exported procedures

---

### `len` — get the length of a wide string

```nim
proc len*(w: WideCString): int
proc len*(w: WideCStringObj): int  # nimv2 only, calls the above
```

**What it does:** Counts the number of `Utf16Char` elements in the wide string, *not* counting the terminating null. This works by linearly scanning until a zero `int16` is found — there is no stored length field.

**Important caveat:** This counts UTF-16 *code units*, not Unicode code points. A single emoji or rare CJK character encoded as a surrogate pair counts as **2** units but represents only **1** character.

**Performance note:** Every call traverses the entire string. If you need the length more than once, store it.

**Example:**

```nim
# "Hi" in UTF-16 → 2 code units + null terminator
let w = newWideCString("Hi")
echo w.len   # → 2

# Emoji encoded as a surrogate pair → 2 code units for 1 character
let emoji = newWideCString("😀")
echo emoji.len   # → 2  (one surrogate pair)
```

---

### `newWideCString` — create a wide string

Three overloads; choose the one that matches your source data.

---

#### `newWideCString(size: int): WideCStringObj`

```nim
proc newWideCString*(size: int): WideCStringObj
```

**What it does:** Allocates an empty (zero-filled) wide string buffer large enough to hold `size` UTF-16 code units plus a null terminator. The buffer is zeroed, so the result is immediately a valid (empty) wide string.

**Use case:** Pre-allocating a buffer you will fill manually, for example when receiving data from a Windows API call that writes into a caller-provided buffer.

**Example:**

```nim
# Reserve space for up to 256 UTF-16 code units
var buf = newWideCString(256)
# Pass buf to a Windows API that fills the buffer, then convert back:
# let name = $buf
```

---

#### `newWideCString(source: cstring, L: int): WideCStringObj`

```nim
proc newWideCString*(source: cstring, L: int): WideCStringObj
```

**What it does:** Converts a UTF-8 `cstring` of known byte-length `L` into a freshly allocated UTF-16 `WideCStringObj`. Each UTF-8 code point is decoded and re-encoded as UTF-16:

- BMP characters (U+0000–U+FFFF, excluding surrogates) → one `Utf16Char`
- Characters above U+FFFF → a surrogate pair (two `Utf16Char` values)
- Invalid surrogates or out-of-range code points → U+FFFD replacement character

**Warning from the source:** `source` must already be allocated with at least `L` bytes. No bounds checking is performed beyond what the length parameter provides.

**Example:**

```nim
let s: cstring = "Привет"
let w = newWideCString(s, s.len)
echo w.len   # → 6  (6 Cyrillic characters, each one UTF-16 code unit)
```

---

#### `newWideCString(s: cstring): WideCStringObj`

```nim
proc newWideCString*(s: cstring): WideCStringObj
```

**What it does:** Convenience overload that determines the length automatically via `s.len`. Returns a null-wide-string (`nullWide`) if `s` is nil.

**Example:**

```nim
let w = newWideCString(cstring "Hello, 世界")
echo w.len  # → 8  (7 ASCII + 1 null BMP char... actually: H,e,l,l,o,,,<space>,世,界 = 9)
```

---

#### `newWideCString(s: string): WideCStringObj`

```nim
proc newWideCString*(s: string): WideCStringObj
```

**What it does:** The most convenient overload — accepts Nim's native `string` directly. Internally casts to `cstring` and calls the two-argument overload with `s.len`.

**This is the overload you will use most often.**

**Example:**

```nim
import widestrs

let greeting = "Héllo, wörld! 🌍"
let wide = newWideCString(greeting)

# wide is now a UTF-16 representation ready for Windows APIs
echo wide.len   # → 16 (15 BMP chars + 1 surrogate pair for 🌍 = 16 code units)

# Convert back to Nim string
let back = $wide
echo back       # → "Héllo, wörld! 🌍"
assert back == greeting
```

---

### `$` — convert a wide string back to a Nim string (UTF-8)

Four overloads; they all decode UTF-16 → UTF-8.

---

#### `$(w: WideCString, estimate: int, replacement: int): string`

```nim
proc `$`*(w: WideCString, estimate: int, replacement: int = 0xFFFD): string
```

**What it does:** The core conversion procedure. Reads `w` until a null terminator is found, decodes each UTF-16 code unit (handling surrogate pairs correctly), and writes the result as a UTF-8 Nim string.

**Parameters:**

- `w` — the wide string to convert.
- `estimate` — a hint for the initial string capacity. Using a good estimate avoids reallocations. The actual capacity allocated is `estimate + estimate / 4`.
- `replacement` — the code point to substitute when an invalid UTF-16 sequence is encountered (defaults to U+FFFD `<EFBFBD>`).

**Surrogate handling:**
- Valid surrogate pair (high followed by low) → decoded to the corresponding Unicode code point.
- Lone surrogate (high without a following low, or a low surrogate appearing unexpectedly) → replaced with `replacement`.

**Example:**

```nim
let w = newWideCString("Hello 🌍")
# Use a generous estimate to avoid reallocation
let s = `$`(w.data, 20)
echo s   # → "Hello 🌍"
```

---

#### `$(s: WideCString): string`

```nim
proc `$`*(s: WideCString): string
```

**What it does:** Shorthand that calls the above with `estimate = 80`. Suitable for most strings. If you know your string is very long, use the explicit-estimate overload for better performance.

**Example:**

```nim
let w = newWideCString("Nim is great!")
echo $w   # → "Nim is great!"
```

---

#### `$(s: WideCStringObj, estimate: int, replacement: int): string` *(nimv2 only)*

```nim
proc `$`*(s: WideCStringObj, estimate: int, replacement: int = 0xFFFD): string
```

**What it does:** Delegates to the `WideCString` overload using the object's internal data pointer. Identical semantics to the raw-pointer version.

---

#### `$(s: WideCStringObj): string` *(nimv2 only)*

```nim
proc `$`*(s: WideCStringObj): string
```

**What it does:** Shorthand `$` for `WideCStringObj`, using the default estimate of 80. The most ergonomic way to convert back to a Nim string.

**Example:**

```nim
import widestrs

let original = "Привет, мир! 🌏"
let wide = newWideCString(original)
let back = $wide
assert back == original
echo back   # → "Привет, мир! 🌏"
```

---

### `toWideCString` — implicit converter (nimv2 only)

```nim
converter toWideCString*(x: WideCStringObj): WideCString {.inline.}
```

**What it does:** Allows a `WideCStringObj` to be used anywhere a `WideCString` (raw pointer) is expected, without an explicit cast. This is what makes passing `WideCStringObj` to C FFI functions transparent.

**This is a converter, not a regular proc** — you never call it explicitly; the compiler inserts it automatically.

**Example:**

```nim
proc someWindowsApi(s: WideCString) = discard  # FFI stub

let w = newWideCString("test")
someWindowsApi(w)   # WideCStringObj silently converts to WideCString
```

---

## Full round-trip example

```nim
import widestrs

# --- UTF-8 → UTF-16 ---
let utf8 = "Héllo, 世界! 🎉"
let wide = newWideCString(utf8)

echo "UTF-16 code units: ", wide.len
# BMP chars: H é l l o ,   世 界 !  = 10
# Surrogate pair for 🎉                = 2
# Total: 12

# --- Inspect individual code units ---
# (raw int16 values, not printable directly)
echo cast[int16](wide[0])   # → 72  (= 'H')

# --- UTF-16 → UTF-8 ---
let roundTrip = $wide
assert roundTrip == utf8
echo roundTrip   # → "Héllo, 世界! 🎉"
```

---

## Windows FFI pattern

The primary real-world use case for this module is calling Windows API functions:

```nim
import widestrs

# Suppose we wrap CreateFileW from the Windows API:
proc CreateFileW(
  lpFileName: WideCString,
  dwDesiredAccess: uint32,
  # ... other params
): int {.importc, dynlib: "kernel32".}

proc openFile(path: string): int =
  let widePath = newWideCString(path)  # UTF-8 → UTF-16
  result = CreateFileW(widePath,       # WideCStringObj → WideCString automatically
                       0x80000000'u32)
```

---

## Quick cheat-sheet

| Symbol | Kind | Purpose |
|--------|------|---------|
| `Utf16Char` | type | A single UTF-16 code unit (distinct `int16`) |
| `WideCString` | type | Raw pointer to a null-terminated UTF-16 array |
| `WideCStringObj` | type | Owning, move-only wrapper around `WideCString` |
| `len(w)` | proc | Count UTF-16 code units (linear scan to null) |
| `newWideCString(size)` | proc | Allocate empty buffer for `size` code units |
| `newWideCString(cstring, int)` | proc | UTF-8 cstring + length → UTF-16 |
| `newWideCString(cstring)` | proc | UTF-8 cstring (auto-length) → UTF-16 |
| `newWideCString(string)` | proc | Nim string → UTF-16 (most common) |
| `$w` | proc | UTF-16 wide string → Nim UTF-8 string |
| `toWideCString` | converter | `WideCStringObj` → `WideCString` (implicit) |
