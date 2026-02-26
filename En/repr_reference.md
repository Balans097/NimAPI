# `repr` Module Reference

> **Module:** `repr.nim` — Nim's Runtime Library  
> **Purpose:** Universal debugging tool. Provides string representations of arbitrary Nim values, including complex nested structures, pointers, and low-level types. The output is Nim-syntax-like and human-readable, making it invaluable during development and debugging.

---

## Overview

The `repr` module is the engine behind Nim's built-in `repr()` procedure. It knows how to traverse the runtime type information (RTTI) of any value and produce a descriptive, readable string — much like Python's `repr()` or Rust's `{:?}` formatter, but for Nim.

All functions in this module are primarily called **by the compiler** (marked `compilerproc` or `compilerRtl`), meaning you rarely call them directly. Instead, you call `repr(someValue)`, and the compiler dispatches to the appropriate function below based on the value's type.

---

## Exported Functions

---

### `reprInt`

```nim
proc reprInt(x: int64): string
```

**What it does:**  
Converts a 64-bit signed integer to its decimal string representation. This is the most fundamental building block — used directly and also called internally by other repr functions when they need to display an integer component (e.g., escape codes in strings).

**Why it matters:**  
All integer types (`int`, `int8`, `int16`, `int32`, `int64`) eventually route through this function (after widening to `int64`). It simply delegates to Nim's `$` operator on `int64`.

**Example:**
```nim
echo reprInt(42)      # "42"
echo reprInt(-1000)   # "-1000"
echo reprInt(0)       # "0"
```

---

### `reprFloat`

```nim
proc reprFloat(x: float): string
```

**What it does:**  
Converts a floating-point number to its string representation. Like `reprInt`, it wraps Nim's `$` operator, which produces a decimal representation with enough precision to round-trip the value.

**Why it matters:**  
Floating-point numbers have subtle representation issues. Nim's `$` for floats guarantees that `parseFloat(reprFloat(x)) == x`, preserving exactness. Useful to know when debugging float arithmetic.

**Example:**
```nim
echo reprFloat(3.14)      # "3.14"
echo reprFloat(1.0e10)    # "10000000000.0"
echo reprFloat(1.0/3.0)   # "0.3333333333333333"
```

---

### `reprPointer`

```nim
proc reprPointer(x: pointer): string
```

**What it does:**  
Formats a raw pointer address as a hexadecimal string using the platform's native `%p` format specifier (via C's `snprintf`). The result looks like `0x7ffd3a1c2b40` on a 64-bit system.

**Why it matters:**  
When debugging reference types, sequences, or raw pointers, seeing the actual memory address helps you identify aliasing (two variables pointing to the same object) or track object identity across `repr` calls. This function is also used internally to prefix sequences and ref values with their address.

**Example:**
```nim
var x = 42
echo reprPointer(addr x)  # e.g., "0x7ffee3b2a004"

# Two refs pointing to the same object will show identical addresses
var a: ref int = new int
var b = a
echo reprPointer(cast[pointer](a))  # same address as b
echo reprPointer(cast[pointer](b))  # same address as a
```

---

### `reprStr`

```nim
proc reprStr(s: string): string
```

**What it does:**  
Produces a debug representation of a Nim string. The output is always enclosed in double quotes, and special characters are escaped:
- `"` → `\"`
- `\` → `\\`
- newline (`\10`) → `\10"` followed by a real newline and `"` (splits across lines for readability)
- Other control characters or high bytes → `\NNN` (decimal escape)

Additionally, on non-nil strings with length > 0, the memory address of the underlying C string data is prepended, which helps identify string aliasing.

**Why it matters:**  
Unlike Nim's `$` operator, `reprStr` makes invisible characters visible. This is crucial when debugging strings that contain embedded nulls, escape sequences, or binary data.

**Example:**
```nim
echo reprStr("hello")         # 0x... "hello"
echo reprStr("say \"hi\"")    # 0x... "say \"hi\""
echo reprStr("line1\nline2")  # 0x... "line1\10"
                               #       "line2"
echo reprStr("")              # ""
```

---

### `reprBool`

```nim
proc reprBool(x: bool): string
```

**What it does:**  
Returns `"true"` or `"false"` for boolean values. Simple and direct.

**Why it matters:**  
When booleans appear inside complex structures (tuples, objects), `repr` uses this function to render them. Having explicit `"true"`/`"false"` strings (rather than `"1"`/`"0"`) keeps output readable.

**Example:**
```nim
echo reprBool(true)   # "true"
echo reprBool(false)  # "false"

# Inside a tuple:
type Flags = tuple[active: bool, count: int]
var f: Flags = (true, 5)
echo repr(f)  # [active = true, count = 5]
```

---

### `reprChar`

```nim
proc reprChar(x: char): string
```

**What it does:**  
Converts a single character to its repr string, always wrapped in single quotes. Special characters are escaped:
- `"` → `\"` inside the quotes
- `\` → `\\`
- Control characters (0–31) and high bytes (127–255) → `\NNN` decimal escape

**Why it matters:**  
Without escaping, printing a null byte or a tab character as a char literal would produce invisible or misleading output. `reprChar` makes the actual byte value unambiguous.

**Example:**
```nim
echo reprChar('A')     # "'A'"
echo reprChar('\n')    # "'\10'"
echo reprChar('\\')    # "'\\'"
echo reprChar('\0')    # "'\0'"
echo reprChar('"')     # "'\"'"
```

---

### `reprEnum`

```nim
proc reprEnum(e: int, typ: PNimType): string
```

**What it does:**  
Given the integer value of an enum and its RTTI (`PNimType`), returns the name of the corresponding enum variant as a string. 

Two lookup strategies are used:
1. **Direct indexing** — for enums without holes (sequential values), uses the ordinal as an array index for O(1) lookup.
2. **Linear search** — for enums with holes (non-sequential values), scans all variants to find a match.

If the integer doesn't match any variant, returns `"<value> (invalid data!)"`.

**Why it matters:**  
Enums are stored as integers at runtime. Without RTTI, you'd just see `2` instead of `"Green"`. This function bridges that gap, making enum values readable in debug output.

**Example:**
```nim
type Color = enum
  Red, Green, Blue

var c = Green
echo repr(c)  # "Green"

# Enum with holes:
type Status = enum
  Ok = 0, Warning = 10, Error = 100

var s = Warning
echo repr(s)  # "Warning"

# Invalid data example (hypothetical):
# reprEnum(99, statusType)  →  "99 (invalid data!)"
```

---

### `reprSet`

```nim
proc reprSet(p: pointer, typ: PNimType): string
```

**What it does:**  
Produces the repr string for a Nim `set`. The output is formatted as `{elem1, elem2, ...}` (or `{}` for an empty set). It reads the underlying bit-array (Nim sets are stored as bit fields) and reconstructs which elements are present, then displays each using `addSetElem`.

The element display format depends on the element type:
- Enum elements → enum variant names
- Bool elements → `true`/`false`
- Char elements → char literals
- Integer elements → decimal numbers

**Why it matters:**  
Nim sets are compact bitmaps at runtime — inspecting raw bytes tells you nothing useful. `reprSet` decodes the bitmap back into the set of values it represents.

**Example:**
```nim
type Fruit = {Apple, Banana, Cherry}
var basket: set[Fruit] = {Apple, Cherry}
echo repr(basket)  # "{Apple, Cherry}"

var bits: set[char] = {'a', 'b', 'z'}
echo repr(bits)    # "{'a', 'b', 'z'}"

var empty: set[int8] = {}
echo repr(empty)   # "{}"
```

---

### `reprOpenArray`

```nim
proc reprOpenArray(p: pointer, length: int, elemtyp: PNimType): string
```

**What it does:**  
Produces a repr string for an open array (Nim's `openArray[T]` — a parameter type that accepts both arrays and sequences). The output is formatted as `[elem0, elem1, ...]`, with each element rendered by the universal `reprAux` dispatcher.

An `openArray` is just a pointer + length at runtime, so this function receives those separately and walks the elements manually.

**Why it matters:**  
`openArray` parameters are a common pattern in Nim for writing flexible procedures. Being able to `repr` them during debugging is essential, even though they have no standalone type at runtime.

**Example:**
```nim
proc process(data: openArray[int]) =
  echo repr(data)  # uses reprOpenArray internally

process([1, 2, 3])    # "[1, 2, 3]"
process(@[10, 20])    # "[10, 20]"
```

---

### `reprAny`

```nim
proc reprAny(p: pointer, typ: PNimType): string
```

**What it does:**  
This is the **top-level dispatcher** — the function that the compiler calls when you write `repr(someValue)`. It accepts a raw pointer to any value and its RTTI, initializes the traversal state (`ReprClosure`), and calls the internal `reprAux` recursive dispatcher.

The `ReprClosure` context tracks:
- **`recdepth`**: recursion depth limit (default: unlimited, `-1`). Prevents infinite loops on cyclic references by outputting `"..."` when the limit is reached.
- **`indent`**: current indentation level for multi-line output.
- **`marked`**: set of already-visited GC cells (prevents infinite loops on cyclic `ref`/`ptr` graphs by showing each cell's content only once).

**Why it matters:**  
Everything flows through `reprAny`. It's the single entry point that makes `repr()` work for every type in Nim. Understanding it helps explain why cyclic structures don't cause infinite loops and why deeply nested structures are handled gracefully.

**Example:**
```nim
# You never call reprAny directly — use repr() instead:
type Node = ref object
  val: int
  next: Node

var n1 = Node(val: 1)
var n2 = Node(val: 2)
n1.next = n2
n2.next = n1  # cycle!

echo repr(n1)
# ref 0x... --> [val = 1,
# next = ref 0x... --> [val = 2,
# next = ref 0x... ]]   ← cycle detected, address shown but not expanded again
```

---

## Internal Helpers (Not Public, But Worth Knowing)

These are not exported for direct use but shape the behavior you observe:

| Helper | Role |
|---|---|
| `reprStrAux` | Core string escaping logic, shared by `reprStr` and the `tyCstring` branch |
| `reprArray` | Renders fixed-size arrays as `[e0, e1, ...]` |
| `reprSequence` | Renders sequences as `0x... @[e0, e1, ...]` (address + `@[...]`) |
| `reprRecord` | Renders tuples and objects as `[field = val, ...]` |
| `reprRef` | Renders `ref`/`ptr` with address and dereferenced content, cycle-safe |
| `reprAux` | Central recursive dispatcher over all type kinds |
| `addSetElem` | Renders a single set element depending on its type |
| `initReprClosure` | Sets up the traversal state (acquires heap lock in multithreaded mode) |
| `deinitReprClosure` | Tears down traversal state (releases heap lock) |

---

## Key Design Notes

**Cycle safety:** `reprAny` / `reprRef` use a `CellSet` (a set of GC cell pointers) to track visited objects. A `ref` or `ptr` that has already been visited is shown as just its address, not expanded again. This prevents infinite recursion on cyclic graphs.

**Thread safety:** When `hasThreadSupport` and `hasSharedHeap` are defined, `initReprClosure` acquires the heap lock for the duration of the repr operation. This is necessary because `CellSet` allocates memory internally.

**Compiler integration:** Functions marked `compilerproc` or `compilerRtl` are called directly by generated C code. The compiler knows their names and emits calls to them — you don't dispatch them at the Nim level.

**Legacy newline:** The `nimLegacyReprWithNewline` define makes `reprAny` append a `\n` at the end of the output, matching older Nim behavior (pre-PR #16034).
