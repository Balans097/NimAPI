
# Variables and Data Types in Nim: A Comprehensive Reference

Nim has a strict yet flexible type system and memory management model. For developers coming from C, C++, Python, or Rust, some Nim concepts may feel unfamiliar.

Below is a detailed explanation of the differences between the various ways of declaring variables, object types, pointers, and other related concepts ("etc.").


---

## 1. Variable declarations: `let`, `var`, `const`

These three keywords define a variable's **mutability** and **time of evaluation**.

### `let` (immutable variable)
* **Essence:** Declares a variable whose value **cannot be changed** after initialization (single assignment).
* **Time of evaluation:** Evaluated at runtime, but the compiler may optimize it into a constant if the value is known in advance.
* **Analog:** `const` in C++ (for local variables), `final` in Java, `let` in JavaScript/Rust.

```nim
let x = 10
# x = 20  # Compile error: cannot reassign a let-variable
```

**A simple example of the key feature — the compiler catches an attempted mutation at compile time, not at runtime:**

```nim
let userName = "Maria"
# userName = "Diana"    # <-- intentionally commented out:
                         #     if uncommented, this produces a compile error
                         #     "userName" cannot be assigned to
echo userName            # "Maria" — a single assignment, read-only afterwards
```

If several `let` declarations follow one another, use the block form instead of repeating the keyword:

```nim
let
  width = 1920   # screen width, px
  height = 1080  # screen height, px
  ratio = width / height  # can reference a let declared earlier in the same block
```

### `var` (mutable variable)
* **Essence:** Declares a classic mutable variable. Its value can be changed while the program runs.
* **Time of evaluation:** At runtime.
* **Analog:** Ordinary variables in C/C++/Python.

```nim
var y = 10
y = 20 # Allowed
```

**Example of the key feature — a var can be declared without an initial value if the type is given explicitly:**

```nim
var counter: int  # no initializer — Nim fills it with the default value (0 for int)
echo counter        # 0
counter = counter + 1
echo counter        # 1
```

Block form for several `var` declarations:

```nim
var
  attempts: int      # attempt counter, defaults to 0
  isRunning: bool    # process active flag, defaults to false
  logLine: string    # accumulating log-line buffer, defaults to ""
```

### `const` (compile-time constant)
* **Essence:** The value must be **computable at compile time**. You cannot use the results of procedure calls (unless marked `static`), values of other variables, and so on.
* **Feature:** A `const` doesn't occupy memory like a variable; the compiler simply substitutes its value (inlining).

```nim
const Pi = 3.14159
const MaxSize = 100
# const runtimeVal = getCurrentTime() # Error: cannot be evaluated at compile time
```

**A simple example of the key feature — a const can be used directly in a type declaration (e.g. as an array size), while a var cannot:**

```nim
const BufferSize = 256   # known in advance, at compile time

type
  Buffer = object
    data: array[BufferSize, byte]  # OK: BufferSize is const, the compiler knows the size ahead of time

# var runtimeSize = 256
# type BadBuffer = object
#   data: array[runtimeSize, byte]  # Error: a var cannot be used as an array size
```

Block form for several `const` declarations:

```nim
const
  AppName = "Monolit"     # application name
  AppVersion = "1.3"      # current version
  MaxThreads = 4          # limit on the number of processing threads
```

---

## 2. Objects: `object` and `ref object`

This is a fundamental Nim distinction, defining **memory semantics** (copy vs. reference) and **allocation** (stack vs. heap).

### `object` (Value / Struct)
* **Essence:** A value type. When assigned or passed to a function, it is **copied** in full.
* **Memory:** Usually placed on the stack (or inline inside another object).
* **Analog:** `struct` in C/C++/Rust.
* **Memory management:** Requires no garbage collection (GC); cleaned up automatically when it goes out of scope.

```nim
type
  Point = object
    x, y: int

var p1 = Point(x: 1, y: 2)
var p2 = p1  # A full data copy happens here!
p2.x = 10
echo p1.x    # Prints 1 (p1 is unchanged)
```

### `ref object` (Reference / Class)
* **Essence:** A reference type. When assigned, only the **reference** (pointer) is copied, not the data itself.
* **Memory:** Data is always allocated on the **heap**.
* **Analog:** Classes in Java/C#/Python, `class` in C++.
* **Memory management:** Tracked by the garbage collector (ARC/ORC in modern Nim).

```nim
type
  Node = ref object
    value: int
    next: Node

var n1 = Node(value: 1)
var n2 = n1  # Only the reference is copied!
n2.value = 99
echo n1.value # Prints 99 (n1 and n2 point to the same data)
```

**A clear illustration of the difference through a procedure** (without `var` in the parameters — see section 5):

```nim
proc tryModify(p: Point) =
  var localP = p
  localP.x = 999   # Only the local copy inside the procedure changes

proc tryModifyRef(n: Node) =
  n.value = 999    # Here the data that n points to changes — it's shared!

var p = Point(x: 1, y: 1)
tryModify(p)
echo p.x            # 1 — the object outside is unchanged

var n = Node(value: 1)
tryModifyRef(n)
echo n.value         # 999 — the ref object changed, even though the parameter was passed "by value"
```

This is a key source of confusion for beginners: even without `var` in the procedure signature, a `ref object` allows changing data from the outside, because only the reference is copied, not the object itself on the heap.

**One more simple example — a `nil` check, which is meaningful only for `ref object` (a plain `object` has no such notion):**

```nim
type
  Task = ref object
    title: string

var t: Task  # an uninitialized ref-variable — nil by default
echo isNil(t)  # true — the reference points to nothing

t = Task(title: "Review the contractor's HSE plan")
echo isNil(t)  # false — t now points to real data on the heap

# t.title will only work after checking/assignment — otherwise accessing
# a field on a nil reference will crash the program at runtime
```

---

## 3. Pointers and addresses: `addr`, `ptr`, `ref`

These concepts deal with physical memory addresses and references.

### `addr` (address-of operator)
* **Essence:** This is an **operator** (like `&` in C++) that returns the physical address of a variable in memory.
* **Returns:** A value of type `ptr[T]`.
* **Rule:** You can only take the address of a `var` (or of a `let`, if you don't intend to modify the data at that address).

```nim
var num = 42
let numAddr = addr num # numAddr has type ptr int
echo numAddr[]         # Prints 42 (dereferencing the pointer)
```

**A simple example of the key feature — through a `ptr` you can modify the original variable, bypassing ordinary assignment:**

```nim
var original = 5
let p = addr original
p[] = 100          # dereference the ptr and write a new value at that address
echo original       # 100 — the original variable changed through the pointer
```

### `ptr` (unmanaged pointer)
* **Essence:** A "raw" pointer, analogous to pointers in C. **Not tracked by the garbage collector**.
* **Use:** Interfacing with C (FFI), writing high-performance code, working with `addr`.
* **Danger:** Can lead to memory leaks or a segfault if the memory is freed manually while you still dereference the pointer.

```nim
type
  CPointer = ptr int

var x = 10
var p: CPointer = addr x
```

### `ref` (managed reference)
* **Essence:** This is not an operator, but a **type modifier**. It creates a reference to an object that **is tracked by the garbage collector**.
* **Use:** Used only when declaring types (`ref object`). You cannot write `ref int` (for that there's `ptr int` or `var`).

```nim
# Correct:
type MyRef = ref MyObject

# Incorrect:
# var x: ref int  # Error!
```

---

## 4. Interfacing with C and `struct`

**Important:** Nim **has no `struct` keyword**.
The role of structs in Nim is fully played by `object`.

However, if you need to work with structs from C (through FFI), you use `object` with the `{.importc.}` and `{.bycopy.}` pragmas.

```nim
# What a C "struct" looks like in Nim:
type
  CStruct {.importc: "my_c_struct", bycopy.} = object
    field1: cint
    field2: cstring

# The {.bycopy.} pragma tells the compiler to pass this object
# by value (like a struct in C), rather than via a hidden pointer.
```

---

## 5. Procedure parameters: `var` in the signature

The `var` keyword is used not only for declaring variables, but also for **passing parameters by reference**.

* **Without `var`:** The parameter is passed by value (copied). *Note: the Nim compiler automatically optimizes passing large objects by reference if it sees you don't modify them, but semantically it's still a copy.*
* **With `var`:** The parameter is passed by reference. Modifying the parameter inside the procedure changes the original variable.

```nim
proc incrementByValue(x: int) =
  x = x + 1 # Only the local copy changes

proc incrementByRef(x: var int) =
  x = x + 1 # The original variable changes!

var myNum = 5
incrementByValue(myNum)
echo myNum # 5

incrementByRef(myNum)
echo myNum # 6
```

**A simple example of the key feature — a `var` parameter also works with an object's fields, which is convenient for mutator procedures:**

```nim
type
  Counter = object
    value: int

proc bump(c: var Counter) =
  # without "var" here, we'd have to return a new Counter and reassign it
  c.value = c.value + 1

var counter = Counter(value: 0)
bump(counter)
bump(counter)
echo counter.value  # 2 — both mutations were applied to the original
```

---

## 6. Other important types

### `seq` (sequences / dynamic arrays)
* **Essence:** A dynamic array.
* **Semantics:** Formally a value type (copied on assignment), but thanks to the **Copy-on-Write** optimization in older versions, or the strict rules in Nim 2.0, it behaves predictably.

```nim
var s: seq[int] = @[1, 2, 3]
add(s, 4)
```

**A simple example of the key feature — a `seq` is copied in full on ordinary assignment (unlike a `ref object`):**

```nim
var original = @[1, 2, 3]
var copy = original   # a full copy of the contents, not a reference
add(copy, 4)
echo len(original)     # 3 — the original is unaffected
echo len(copy)         # 4 — only copy changed
```

Useful functional-style calls for `seq` (in the prefix form, not the dot form):

```nim
var numbers = @[5, 3, 1, 4, 2]
echo len(numbers)        # 5 — the length of the sequence
echo contains(numbers, 3) # true — checking whether an element is present
delete(numbers, 0)        # deletes the element at that index — now @[3, 1, 4, 2]
```

### `string` (strings)
* **Essence:** In Nim, strings are **mutable** sequences of characters (unlike Python/Java, where they are immutable).
* **Semantics:** Strings are passed by value, but internally they are represented as byte arrays with a null terminator (for C compatibility). In Nim 2.0, strings have strict value semantics and don't use Copy-on-Write, which makes them thread-safe.

```nim
var str = "Hello"
add(str, " World") # Strings are mutable!
```

**A simple example of the key feature — commonly used string procedures in the functional form (from `std/strutils`):**

```nim
import std/strutils

let path = "  /home/maria/Monolit.nim  "
echo strip(path)               # trims whitespace from both ends
echo startsWith(path, "  /home") # true — prefix check, without .startsWith()
echo toUpperAscii("nim")         # "NIM"
echo split("a,b,c", ',')         # @["a", "b", "c"]
```

---

## 7. Arrays: `array` (static, fixed size)

* **Essence:** A fixed-length array whose length is known **at compile time**. Unlike `seq`, the size cannot be changed after creation.
* **Memory:** Placed on the stack (if it's a local variable), with no heap access and no GC.
* **Analog:** `T arr[N]` in C/C++, `[T; N]` in Rust.
* **Semantics:** A value type — copied in full on assignment, like `object`.

```nim
var a: array[3, int] = [1, 2, 3]
var b = a        # A full copy, not a reference
b[0] = 100
echo a[0]         # Prints 1 (a is unchanged)

# Indexing doesn't have to start at 0:
var days: array[1..7, string] = ["Mon","Tue","Wed","Thu","Fri","Sat","Sun"]
echo days[1]      # "Mon"
```

**`array` vs `seq` — when to use which:**

| | `array[N, T]` | `seq[T]` |
| :--- | :--- | :--- |
| Size | Fixed at compile time | Dynamic (can grow/shrink) |
| Memory | Stack (usually) | Heap |
| Copying | Always full | Full (value), but GC-managed |
| When to use | Size is known in advance, maximum speed is needed | Size changes at runtime |

---

## 8. Tuples: `tuple`

* **Essence:** A group of values of different types. Unlike `object`, it doesn't require a separate `type` declaration, though one is possible.
* **Semantics:** A value type, copied in full, like `object`.
* **Difference from `object`:** Two tuples with the same set of fields are considered the same type (structural equivalence), rather than by name.

```nim
# An unnamed (positional) tuple:
var point = (1, 2)
echo point[0]        # 1

# A named tuple — reads like an object, but remains a tuple:
var p2 = (x: 1, y: 2)
echo p2.x             # 1
p2.x = 10             # Tuple fields can be changed if the tuple itself is a var

# Two tuples with the same field structure are considered the same type:
type Coord = tuple[x, y: int]
var c: Coord = (x: 5, y: 5)
```

**A simple example of the key feature — a convenient way to return several values from a procedure and unpack them immediately:**

```nim
proc minMax(numbers: seq[int]): tuple[minVal, maxVal: int] =
  # the procedure returns a named tuple — two values in one call
  result = (minVal: numbers[0], maxVal: numbers[0])
  for n in numbers:
    if n < result.minVal:
      result.minVal = n
    if n > result.maxVal:
      result.maxVal = n

let
  data = @[7, 2, 9, 4]
  (lo, hi) = minMax(data)  # unpack the tuple straight into two separate variables

echo lo   # 2
echo hi   # 9
```

---

## 9. Sets and enums: `set` and `enum`

### `enum` (enumeration)
* **Essence:** A named set of integer constants. In Nim, unlike C, `enum` is a full-fledged separate type, not just an `int`.

```nim
type
  Color = enum
    cRed, cGreen, cBlue

var favorite = cGreen
echo ord(favorite)   # 1 (the ordinal value)
```

**A simple example of the key feature — an `enum` can be safely used in a `case`, and the compiler checks that all variants are handled:**

```nim
type
  HseStatus = enum
    hsCompliant, hsMinorIssue, hsCritical

proc describe(status: HseStatus): string =
  case status
  of hsCompliant: "Compliant with requirements"
  of hsMinorIssue: "Minor issue"
  of hsCritical: "Critical violation"
  # if you forget one of the enum variants, the compiler raises an error:
  # "not all cases are covered"

echo describe(hsCritical)  # "Critical violation"
```

### `set` (bit set)
* **Essence:** A compact set of values of an **ordinal type** (enum, char, a range of small ints). Implemented as a bitmask — very fast membership and union operations.
* **Constraint:** The base type must be small (usually up to 2^16 possible values), otherwise the compiler will refuse to build the set.

```nim
type
  Direction = enum
    dNorth, dEast, dSouth, dWest

var allowed: set[Direction] = {dNorth, dEast}
incl(allowed, dSouth)
echo dWest in allowed   # false
echo dNorth in allowed  # true
```

**A simple example of the key feature — a `set` supports set algebra (union, intersection) directly via operators:**

```nim
type
  Weekday = enum
    moMon, moTue, moWed, moThu, moFri, moSat, moSun

let
  workDays: set[Weekday] = {moMon, moTue, moWed, moThu, moFri}
  weekend: set[Weekday] = {moSat, moSun}
  allDays = workDays + weekend      # union of sets
  noOverlap = workDays * weekend    # intersection — empty in this case

echo card(allDays)    # 7 — card() returns the number of elements in the set
echo card(noOverlap)  # 0 — weekdays and the weekend don't overlap
```

---

## Quick-reference summary table

| Concept | Keyword / Type | Semantics | Memory | C/C++ analog |
| :--- | :--- | :--- | :--- | :--- |
| **Immutability** | `let` | Value | Stack / Register | `const T x` (local) |
| **Mutability** | `var` | Value | Stack / Heap | `T x` |
| **Constant** | `const` | Compile-time evaluation | None (inline) | `#define` / `constexpr` |
| **Struct** | `object` | Value (copied) | Stack | `struct` |
| **Class** | `ref object` | Reference (pointer copied) | Heap (GC) | `class` (with GC) |
| **Address-of** | `addr` | Operator | Returns an address | `&x` |
| **Raw pointer** | `ptr` | Reference (no GC) | Any | `T*` |
| **Smart reference** | `ref` | Reference (with GC) | Heap | `std::shared_ptr` (roughly) |
| **Pass by reference**| `var` (in parameters) | Reference | Depends on the variable | `T&` |
| **Static array** | `array[N, T]` | Value (copied) | Stack | `T arr[N]` |
| **Dynamic array** | `seq[T]` | Value (with GC) | Heap | `std::vector<T>` |
| **Tuple** | `tuple` | Value (structural equivalence) | Stack | `std::tuple` / anonymous `struct` |
| **Enumeration** | `enum` | Value (integer) | Stack | `enum` |
| **Bit set** | `set[T]` | Value | Stack | `std::bitset` (roughly) |

### Summary for decision-making:
1. Use **`let`** by default for all variables.
2. Use **`var`** only when the value genuinely needs to change.
3. Use **`object`** when the data is small and it's reasonable to copy it (coordinates, colors, configurations).
4. Use **`ref object`** when the data is large, has a complex lifecycle, or forms graphs/trees (tree nodes, game state).
5. Use **`ptr`** and **`addr`** only for low-level programming or FFI (interfacing with C).
6. Use **`array`** when the collection size is known in advance and doesn't change; use **`seq`** when the size is dynamic.
7. Use **`tuple`** when you need to quickly group a few values without declaring a separate type (e.g. to return 2–3 values from a procedure); use **`object`** when the structure is important enough to have its own name and methods.
8. Use **`enum`** for named sets of states instead of "magic" integers; use **`set`** for compact storage and fast checking of combinations of enum values.
9. Write procedure calls in the functional form — **`len(x)`, `close(f)`, `add(s, v)`** — rather than `x.len()`, `f.close()`, `s.add(v)`. Reserve dot notation for field access (`p.x`) and indexing (`a[0]`) only.
10. Declare several `let`/`var`/`const` in a row as a block (the keyword once, followed by indented lines), rather than repeating the keyword on each line.
11. Write `import` comma-separated on one line, or group logically related modules using `std/[...]`.
