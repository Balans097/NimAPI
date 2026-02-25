# `with` Module Reference

> Provides the `with` macro for convenient function call chaining on a single object.  

---

## The problem `with` solves

When you need to call several procedures or perform several assignments on the same variable, you end up writing the variable name over and over:

```nim
# Without with — repetitive and noisy
var s = "Hello"
s.add(", ")
s.add("world")
s.add("!")
```

For deeply nested fields or long variable names, this becomes even more tedious. The `with` macro eliminates the repetition by making one variable the implicit first argument for an entire block of calls.

---

## Exported symbols

### `with` — call chaining macro

```nim
macro with*(arg: typed; calls: varargs[untyped]): untyped
```

**What it does:** Takes a target expression `arg` and a block of `calls`, then rewrites every statement in that block so that `arg` is automatically inserted as the first argument (or as the receiver) of each call. The result is a `StmtList` — a plain sequence of statements, just as if you had written them out in full by hand.

Under the hood, `with` delegates the actual transformation to `underscoredCalls` from the private `std/private/underscored_calls` module. That helper looks for a `_` placeholder in each call; if no `_` is present, it prepends `arg` as the first argument automatically.

> **Caution:** `arg` is re-evaluated for every statement in the block. If `arg` is an expression with side effects (e.g. a function call), those side effects will occur multiple times. Bind the result to a plain variable first if that matters.

---

## Usage patterns

### Pattern 1 — calling mutating procedures on a variable

The most common use. Every call in the block receives the variable as its first argument:

```nim
import std/with

var s = "yay"
with s:
  add "abc"   # expands to: s.add("abc")
  add "efg"   # expands to: s.add("efg")

doAssert s == "yayabcefg"
```

This is exactly equivalent to:

```nim
s.add("abc")
s.add("efg")
```

---

### Pattern 2 — compound assignment operators

Operator calls like `+=` and `-=` are treated as calls too and get `arg` prepended:

```nim
import std/with

var a = 44
with a:
  += 4    # expands to: a += 4
  -= 5    # expands to: a -= 5

doAssert a == 43
```

---

### Pattern 3 — field assignments on object types

Plain assignments to fields are also rewritten, making object initialisation tidy:

```nim
import std/with

type Config = object
  host: string
  port: int
  debug: bool

var cfg = Config()
with cfg:
  host = "localhost"   # expands to: cfg.host = "localhost"
  port = 8080          # expands to: cfg.port = 8080
  debug = true         # expands to: cfg.debug = true
```

---

### Pattern 4 — nesting for sub-objects

`with` blocks can be nested, each introducing a new implicit receiver for its inner scope:

```nim
import std/with

var foo = (bar: 1, qux: (baz: 2))
with foo:
  bar = 2          # foo.bar = 2
  with qux:        # now the implicit receiver is foo.qux
    baz = 3        # foo.qux.baz = 3

doAssert foo.bar == 2
doAssert foo.qux.baz == 3
```

---

### Pattern 5 — mixing calls and assignments freely

A single `with` block can contain any combination of calls, operators, and assignments:

```nim
import std/with

type Builder = object
  parts: seq[string]
  separator: string

proc add(b: var Builder, part: string) = b.parts.add(part)
proc setSep(b: var Builder, sep: string) = b.separator = sep

var b = Builder()
with b:
  add "first"
  add "second"
  setSep ", "
  parts.add "third"   # direct field access also works

echo b.parts    # → @["first", "second", "third"]
echo b.separator # → ", "
```

---

## How the expansion works

`with` is purely a compile-time transformation — it generates no extra runtime code beyond the statements themselves. Given:

```nim
with someVar:
  foo(x)
  bar
```

The macro produces:

```nim
someVar.foo(x)
someVar.bar
```

If you use the `_` placeholder explicitly, it controls exactly where `arg` is inserted:

```nim
with someVar:
  foo(_, extraArg)   # _ is replaced by someVar
  # → someVar.foo(someVar, extraArg)  -- use with care
```

Using `_` explicitly is an advanced pattern; for typical chaining you never need it.

---

## The side-effect caveat in detail

Because every statement in the block independently receives `arg`, and `arg` is not stored in a temporary variable by the macro itself, any expression passed as `arg` is re-evaluated each time:

```nim
proc makeObj(): MyObj =
  echo "created!"    # this side effect runs once per statement in the block
  result = MyObj()

with makeObj():      # DO NOT do this with side-effectful expressions
  field1 = 1
  field2 = 2
# "created!" is printed TWICE
```

**Safe pattern:** assign first, then use `with`:

```nim
var obj = makeObj()   # side effect runs exactly once
with obj:
  field1 = 1
  field2 = 2
```

---

## Comparison with similar approaches

| Approach | Nim spelling | Characteristic |
|----------|-------------|----------------|
| Manual chaining | `s.add("a"); s.add("b")` | Explicit, verbose |
| `result` convention | implicit `result` in procs | Works only inside procedures |
| `with` macro | `with s: add "a"; add "b"` | Concise, works anywhere, pure compile-time |
| Method chaining (fluent API) | `s.add("a").add("b")` | Requires return-self convention in each proc |

`with` is the most general option because it requires no changes to the called procedures — any existing proc that accepts the target type as its first argument works immediately.

---

## Quick cheat-sheet

| Feature | Supported |
|---------|-----------|
| Mutating procedure calls | ✓ |
| Compound operators (`+=`, `-=`, …) | ✓ |
| Field assignments | ✓ |
| Nesting (`with` inside `with`) | ✓ |
| Explicit `_` placeholder | ✓ |
| Side-effect-free `arg` only | ⚠ caution required |
| Runtime overhead | none (compile-time only) |
