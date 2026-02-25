# `importutils` Module Reference

> **Module:** `std/importutils` (Nim standard library)  
> **Status:** ⚠️ Experimental API — subject to change in future Nim versions.  
> **Purpose:** Utilities related to import and symbol resolution, with a focus on controlled access to otherwise inaccessible (private) fields of types.

---

## Background: What Problem Does This Module Solve?

In Nim, types and their fields can be public (exported with `*`) or private (not exported). Private fields are normally inaccessible from outside the module that defines the type — this is a deliberate encapsulation mechanism.

However, there are legitimate situations where you need to reach into a type's private fields:

- **Testing:** you want to assert on internal state without exposing a public accessor just for tests.
- **Interoperability:** you are working with a third-party type that didn't expose what you need.
- **Debugging and inspection tools:** you need to serialize or print the full state of an object.
- **FFI and low-level code:** you are bridging between Nim and external code where full field access is necessary.

The `importutils` module provides exactly one tool for this: `privateAccess`.

---

## Exported Functions

---

### `privateAccess`

```nim
proc privateAccess*(t: typedesc) {.magic: "PrivateAccess".}
```

**What it does**

`privateAccess` grants access to the private fields of a given type within the **current lexical scope**. Once called, any private fields of `t` that would normally trigger a compile-time error become readable and writable — but only inside the block where the call was made.

This is a **compile-time construct**, not a runtime one. The `{.magic: "PrivateAccess".}` pragma means it is handled directly by the Nim compiler as a special built-in, not by ordinary code generation. There is zero runtime overhead.

The scope-limiting behavior is a deliberate safety feature: the access grant does not "leak" beyond the enclosing block. As soon as execution logically exits that block, the compiler once again treats private fields as inaccessible.

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `t` | `typedesc` | The type whose private fields you want to access. You can pass either a type name directly (e.g. `Goo`) or derive the type from an existing variable (e.g. `myVar.type`). |

**Returns:** Nothing. The effect is purely a compile-time permission grant.

---

### How Scope Works

The key concept to internalize is that `privateAccess` is **scoped**. Consider this structure:

```nim
block outerBlock:
  # f.f0 is NOT accessible here

  block innerBlock:
    privateAccess(MyType)
    # f.f0 IS accessible here
    # nested blocks inside also inherit the access:
    block deeperBlock:
      # f.f0 is STILL accessible here
  
  # f.f0 is NOT accessible here again
```

This mirrors how `var` declarations work in Nim: a declaration inside a `block` is visible until the block ends. `privateAccess` follows the same scoping rules.

---

### Two Ways to Pass the Type

**From a variable** — useful when you already have an instance and want to unlock its type:

```nim
var f = initFoo()
privateAccess(f.type)   # f.type resolves to Foo at compile time
```

**From a type name directly** — useful for generic types or when you don't have an instance yet:

```nim
privateAccess(Goo)           # concrete type
privateAccess(Goo[float])    # instantiated generic
```

Both forms are equivalent in effect.

---

### Example: Accessing Private Fields of a Concrete Type

```nim
# Assume another module defines:
# type
#   Foo = object
#     f0: int  # private — not exported
# proc initFoo*(): auto = Foo()

import std/importutils
import mymodule  # provides initFoo

var f = initFoo()

# Without privateAccess, this would be a compile error:
# f.f0 = 1  # Error: undeclared field 'f0'

block:
  privateAccess(f.type)   # grant access to Foo's private fields
  f.f0 = 1                # now this compiles and runs fine
  assert f.f0 == 1        # read access works too

# Back outside the block — access is revoked at compile time:
# f.f0 = 2  # Error again: undeclared field 'f0'
assert not compiles(f.f0)
```

The `assert not compiles(f.f0)` line is a compile-time assertion — it verifies that the access *truly* does not work outside the block. This pattern is used in the module's own tests to document the scoping guarantee.

---

### Example: Accessing Private Fields of a Generic Type

Generic types work the same way — pass the type name (with or without type parameters):

```nim
# Assume:
# type
#   Goo*[T] = object
#     g0: int  # private

import std/importutils

privateAccess(Goo)                          # unlocks Goo for any T
assert Goo[float](g0: 1).g0 == 1           # construct and read a private field

privateAccess(Goo[int])                     # alternatively, specific instantiation
var x = Goo[int](g0: 42)
assert x.g0 == 42
```

Note that even though `Goo` itself is exported (has `*`), its field `g0` is not. `privateAccess` bridges that gap for generic types just as cleanly as for concrete ones.

---

### Example: Using `privateAccess` in Tests

A very common and idiomatic use case is white-box testing, where you want to verify internal state without polluting the public API with test-only accessors:

```nim
# tests/test_foo.nim
import std/importutils
import mymodule

proc testFooInternals() =
  var f = initFoo()
  doSomethingWith(f)

  block checkInternalState:
    privateAccess(Foo)
    # Verify that the internal counter was updated correctly:
    assert f.f0 == 42, "Expected f0 to be 42 after processing"

testFooInternals()
```

The test file is a separate module, so normally it cannot see `f0`. With `privateAccess`, it can — but only inside the test block, keeping the rest of the code clean.

---

## Important Notes and Caveats

**It is a tool for deliberate, supervised escape hatches — not a way to ignore encapsulation carelessly.** Using `privateAccess` in production application logic is a design smell. The private field you are accessing may change, be renamed, or be removed in a future version of the library, and there will be no API deprecation notice because it was never part of the public contract.

**The scope is lexical, not dynamic.** The access grant does not follow function calls made from within the block. If you call `someProc()` from inside the block, `someProc`'s body does not automatically gain private access. Only the code *textually* within the block benefits.

**No runtime overhead.** `privateAccess` is entirely a compile-time directive. It produces no executable code and has no effect on performance.

**Experimental status.** The module header marks this API as experimental and subject to change. In particular, more symbol-resolution utilities may be added in the future (see the module's own comments about `whichModule`, `getCurrentPkgDir`, and similar planned features).

---

## Summary

| Aspect | Detail |
|--------|--------|
| What it unlocks | Private (unexported) fields of any Nim object type |
| When the access applies | Only within the lexical block where `privateAccess` was called |
| Works with generics? | Yes — pass type name with or without type parameters |
| Runtime cost | Zero — purely compile-time |
| Intended use cases | Testing, debugging, interop, low-level utilities |
| Risk of misuse | Breaks encapsulation; use only where truly needed |
| API stability | Experimental — may change |
