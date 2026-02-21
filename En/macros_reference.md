# macros.nim — Module Reference

> **Import:** `import std/macros`  
> **Purpose:** Compile-time access to and transformation of Nim's Abstract Syntax Tree (AST). Everything in this module runs *at compile time*, inside `macro` definitions.

---

## What is a macro in Nim?

A Nim macro is a procedure that runs at **compile time**, receives parts of the program's source code as a tree of `NimNode` objects, and returns a new `NimNode` tree that replaces the original code. This is how Nim enables zero-overhead DSLs, code generation, and static reflection without any runtime cost.

The central type is `NimNode`: every piece of Nim syntax — a literal, a variable name, a call expression, a whole `proc` definition — is represented as a `NimNode` with a specific `NimNodeKind`.

---

## Core types

### `NimNodeKind`

An enum with ~150 values that classify every syntactic construct in Nim. A few commonly encountered values:

| Kind | What it represents |
|---|---|
| `nnkIdent` | An identifier, e.g. `foo` |
| `nnkSym` | A resolved symbol (after semantic analysis) |
| `nnkIntLit` | An integer literal |
| `nnkStrLit` | A string literal |
| `nnkCall` | A procedure call `f(a, b)` |
| `nnkInfix` | An infix call `a + b` |
| `nnkStmtList` | A sequence of statements |
| `nnkProcDef` | A `proc` definition |
| `nnkEmpty` | An absent/optional node (used as placeholder) |

**Node sets** (useful constants):

| Constant | Contents |
|---|---|
| `nnkLiterals` | All literal node kinds (`nnkCharLit..nnkNilLit`) |
| `nnkCallKinds` | All call-like nodes |
| `RoutineNodes` | All routine definition kinds (proc, func, method, …) |
| `AtomicNodes` | Leaf nodes that have no children |
| `CallNodes` | Same as `nnkCallKinds` |

---

### `NimTypeKind`

Classifies types as seen by the compiler (e.g. `ntyInt`, `ntyObject`, `ntySeq`). Obtained via `typeKind`.

---

### `NimSymKind`

Classifies what a resolved symbol refers to (e.g. `nskProc`, `nskVar`, `nskType`, `nskField`). Obtained via `symKind`.

---

### `LineInfo`

```nim
type LineInfo* = object
  filename*: string
  line*, column*: int
```

Represents a position in source code. Returned by `lineInfoObj`. Can be converted to `"file(line, col)"` with `$`.

---

### `BindSymRule`

Controls how `bindSym` resolves overloaded symbols:

| Value | Behaviour |
|---|---|
| `brClosed` | Only current-scope symbols; default |
| `brOpen` | Open for overloads, matching generic binding rules |
| `brForceOpen` | Always returns an open choice node |

---

## Inspecting nodes

### `len`

```nim
proc len*(n: NimNode): int
```

Returns the number of direct children of `n`. Leaf nodes (literals, identifiers) have `len == 0`.

```nim
# nnkCall for "echo(a, b)" has 3 children: echo, a, b
assert callNode.len == 3
```

---

### `[]` / `[]=` — child access

```nim
proc `[]`*(n: NimNode, i: int): NimNode
proc `[]`*(n: NimNode, i: BackwardsIndex): NimNode   # n[^1]
proc `[]`*[T,U](n: NimNode, x: HSlice[T,U]): seq[NimNode]
proc `[]=`*(n: NimNode, i: int, child: NimNode)
proc `[]=`*(n: NimNode, i: BackwardsIndex, child: NimNode)
```

Read or replace a child by index. Negative indexing (`n[^1]`) accesses from the end. Slice indexing returns a `seq[NimNode]`.

```nim
let call = quote do: foo(1, 2)
assert $call[0] == "foo"   # procedure name
assert call[1].intVal == 1
call[2] = newLit(99)       # replace second argument
```

---

### `kind`

```nim
proc kind*(n: NimNode): NimNodeKind
```

Returns the kind of `n`. The most fundamental discriminant — always check `kind` before reading `intVal`, `strVal`, or `floatVal`.

```nim
if n.kind == nnkStrLit:
  echo "string: ", n.strVal
```

---

### `intVal`, `floatVal`, `strVal`

```nim
proc intVal*(n: NimNode): BiggestInt
proc floatVal*(n: NimNode): BiggestFloat
proc strVal*(n: NimNode): string
```

Read the payload of a literal or identifier node. Each is only valid for the appropriate `kind`:

- `intVal` — integer and character literals, enum field symbols
- `floatVal` — float literals
- `strVal` — string literals, identifiers (`nnkIdent`), symbols (`nnkSym`), comments

```nim
let n = newLit(42)
assert n.intVal == 42

let id = ident("myProc")
assert id.strVal == "myProc"
```

---

### `strVal=`

```nim
proc `strVal=`*(n: NimNode, val: string)
```

Sets the string content of a string literal or comment node. Not valid for `nnkIdent` or `nnkSym` — create a new node with `ident(string)` instead.

---

### `intVal=` / `floatVal=`

```nim
proc `intVal=`*(n: NimNode, val: BiggestInt)
proc `floatVal=`*(n: NimNode, val: BiggestFloat)
```

Mutate the numeric payload of a literal node in-place.

---

### `symKind`

```nim
proc symKind*(symbol: NimNode): NimSymKind
```

For a node of kind `nnkSym`, returns what the symbol refers to. Useful for distinguishing a `nskProc` symbol from a `nskType` or `nskVar`.

```nim
if sym.symKind == nskProc:
  echo "This is a procedure symbol"
```

---

### `boolVal`

```nim
proc boolVal*(n: NimNode): bool
```

Extracts the boolean value from an `nnkIntLit` node or from the bound symbol `true`/`false`. Avoids writing `n.intVal != 0` directly.

---

### `last`

```nim
proc last*(node: NimNode): NimNode
```

Returns the last child of `node`. Equivalent to `node[^1]`. Useful when traversing statement lists where the last element is the result.

---

### `==` (structural equality)

```nim
proc `==`*(a, b: NimNode): bool
```

Two nodes are equal if they are **structurally identical** — same kind, same payload, same children. Two independently constructed nodes with the same content will compare as equal.

---

### `sameType`

```nim
proc sameType*(a, b: NimNode): bool
```

Returns `true` if `a` and `b` represent the same type, including across aliases (e.g. a `type MyInt = int` alias is the same type as `int`).

---

### `eqIdent`

```nim
proc eqIdent*(a: string; b: string): bool
proc eqIdent*(a: NimNode; b: string): bool
proc eqIdent*(a: string; b: NimNode): bool
proc eqIdent*(a: NimNode; b: NimNode): bool
```

**Style-insensitive** identifier comparison: `eqIdent("myProc", "myproc")` is `true`. Also handles `nnkPostfix` (export marker `*`) and backtick-quoted identifiers by unwrapping them automatically. Always prefer `eqIdent` over `$node == name` for identifier comparison.

```nim
if n.eqIdent("result"):
  echo "Found the result variable"
```

---

### `or` (nil / empty fallback)

```nim
template `or`*(x, y: NimNode): NimNode
```

Evaluates `x`; if it is `nil` or `nnkEmpty`, returns `y` instead. Chains multiple "try this or that" lookups elegantly.

```nim
let body = node.body or newStmtList()
```

---

## Type inspection

### `getType`

```nim
proc getType*(n: NimNode): NimNode
proc getType*(n: typedesc): NimNode
```

Returns the compiler's representation of `n`'s type as a `NimNode`. The result is itself a node tree you can traverse. Recursive types are pre-flattened to avoid infinite recursion; call `getType` again on a result node to follow type aliases.

```nim
macro showType(x: typed): untyped =
  echo x.getType.repr
showType(3.14)  # prints: float64
```

---

### `typeKind`

```nim
proc typeKind*(n: NimNode): NimTypeKind
```

Given a node obtained via `getType`, returns which category of type it is (`ntyInt`, `ntyObject`, `ntySeq`, etc.).

---

### `getTypeInst`

```nim
proc getTypeInst*(n: NimNode): NimNode
proc getTypeInst*(n: typedesc): NimNode
```

Returns the type **as it was written at the declaration site**, preserving aliases. Useful when you need to know that a variable was declared as `Vec4f` rather than the underlying `Vec[4, float32]`.

```nim
# If declared as "var a: Vec4f", getTypeInst returns the Vec4f alias node.
# getTypeImpl would return the fully-expanded object definition.
```

---

### `getTypeImpl`

```nim
proc getTypeImpl*(n: NimNode): NimNode
proc getTypeImpl*(n: typedesc): NimNode
```

Returns the type **after all aliases are expanded**, showing the concrete implementation. Unlike `getTypeInst`, all intermediate aliases are gone — you see the final `object`, `tuple`, etc. definition.

---

### `getImpl`

```nim
proc getImpl*(symbol: NimNode): NimNode
```

Returns the complete AST of a symbol's declaration — a `proc`'s full body, a `type`'s full definition, etc. Returns `nil` for symbols without accessible source (e.g. built-in magic symbols).

```nim
macro showImpl(x: typed): untyped =
  echo x.getImpl.treeRepr
```

---

### `getImplTransformed`

```nim
proc getImplTransformed*(symbol: NimNode): NimNode
```

For a typed `proc`, returns the AST **after the compiler's transformation pass** — after `defer` rewriting, iterator transformation, etc. Useful for debugging the compiler's internal transformations. (Available since Nim 1.3.5.)

---

### `isInstantiationOf`

```nim
proc isInstantiationOf*(instanceProcSym, genProcSym: NimNode): bool
```

Checks whether `instanceProcSym` is a concrete instantiation of the generic procedure `genProcSym`. Useful to verify proc symbols against generic templates returned by `bindSym`.

---

### `signatureHash`

```nim
proc signatureHash*(n: NimNode): string
```

Returns a stable hash string derived from a symbol's type, owning module, and other signature factors. This is the same identifier the backend uses for mangled symbol names — useful for generating stable unique names.

---

### `symBodyHash`

```nim
proc symBodyHash*(s: NimNode): string
```

A deeper hash than `signatureHash`: includes the implementation body and recursively hashes all procs and variables used inside it. Useful for cache-invalidation checks in code-generation tools.

---

### `isExported`

```nim
proc isExported*(n: NimNode): bool
```

Returns `true` if the symbol represented by `n` is exported (i.e. marked with `*`).

---

### `getSize`, `getAlign`, `getOffset`

```nim
proc getSize*(arg: NimNode): int
proc getAlign*(arg: NimNode): int
proc getOffset*(arg: NimNode): int
```

Compile-time equivalents of `sizeof`, `alignof`, and `offsetof`. Return a negative value if the compiler cannot determine the size/alignment/offset. `getOffset` takes a single field-symbol node rather than a type + field pair.

```nim
macro checkSize(T: typedesc): untyped =
  let sz = getType(T).getSize
  echo "Size of type: ", sz
```

---

## Building nodes

### `newNimNode`

```nim
proc newNimNode*(kind: NimNodeKind, lineInfoFrom: NimNode = nil): NimNode
```

The fundamental node constructor. Creates a new, childless node of the given kind. `lineInfoFrom` can be any existing node — its source location is copied so that compiler errors generated from this node point to a meaningful location.

```nim
let call = newNimNode(nnkCall)
call.add(ident("echo"), newLit("hello"))
```

---

### `newTree`

```nim
proc newTree*(kind: NimNodeKind, children: varargs[NimNode]): NimNode
```

A more convenient `newNimNode` that immediately adds children. The idiomatic way to build composite AST nodes.

```nim
# Equivalent to writing `a + b` in code
let sum = newTree(nnkInfix, ident("+"), a, b)

# Or using the kind as a constructor (also works):
let sum2 = nnkInfix.newTree(ident("+"), a, b)
```

---

### `newLit`

```nim
proc newLit*(c: char): NimNode
proc newLit*(i: int | int8 | int16 | int32 | int64): NimNode
proc newLit*(i: uint | uint8 | uint16 | uint32 | uint64): NimNode
proc newLit*(b: bool): NimNode
proc newLit*(s: string): NimNode
proc newLit*(f: float32 | float64): NimNode
proc newLit*(arg: enum): NimNode
proc newLit*(arg: object): NimNode
proc newLit*(arg: ref object): NimNode
proc newLit*[N,T](arg: array[N,T]): NimNode
proc newLit*[T](arg: seq[T]): NimNode
proc newLit*[T](s: set[T]): NimNode
proc newLit*[T: tuple](arg: T): NimNode
```

Converts a compile-time Nim value into its AST literal representation. Works recursively for compound types (objects, arrays, seqs, tuples). The produced node, when injected into generated code, will reproduce the original value.

```nim
let n = newLit(42)       # nnkIntLit with intVal = 42
let s = newLit("hello")  # nnkStrLit
let a = newLit([1, 2, 3]) # nnkBracket with three nnkIntLit children
```

---

### `ident`

```nim
proc ident*(name: string): NimNode
```

Creates an `nnkIdent` node from a string. This is the right way to introduce a fresh name into generated code. It does **not** bind to any symbol — the compiler resolves it at the call site of the generated code.

```nim
let varName = ident("myCounter")
let stmt = newVarStmt(varName, newLit(0))
# Generates: var myCounter = 0
```

---

### `newIdentNode`

```nim
proc newIdentNode*(i: string): NimNode
```

Alias for `ident(string)`. Prefer `ident`.

---

### `bindSym`

```nim
proc bindSym*(ident: string | NimNode,
              rule: BindSymRule = brClosed): NimNode
```

Creates a node that is **immediately bound** to a specific symbol in the current scope, not just a name. Unlike `ident`, the resulting node carries the resolved symbol identity, so overloading and name shadowing at the call site do not affect which symbol is used. Essential when you want to guarantee that generated code calls a specific procedure regardless of local overloads.

```nim
# Generates code that calls the `echo` symbol from system, not any shadowed version
let echoSym = bindSym("echo")
result = newCall(echoSym, newLit("safe"))
```

---

### `genSym`

```nim
proc genSym*(kind: NimSymKind = nskLet; ident = ""): NimNode
```

Generates a **hygienically unique** symbol guaranteed not to clash with any name in the surrounding code. Use this for temporaries inside macros to avoid variable capture bugs. The optional `ident` is a hint for the debugger but does not affect uniqueness.

```nim
let tmp = genSym(nskVar, "temp")
result = newVarStmt(tmp, expr)
result.add newAssignment(output, tmp)
```

---

### `newCall`

```nim
proc newCall*(theProc: NimNode | string, args: varargs[NimNode]): NimNode
```

Shortcut for constructing an `nnkCall` node: the first argument is the callee (a node or a string name), followed by the argument nodes.

```nim
# Generates: echo("hello", x)
let call = newCall("echo", newLit("hello"), ident("x"))
```

---

### `newStrLitNode`

```nim
proc newStrLitNode*(s: string): NimNode
```

Creates an `nnkStrLit` node. Equivalent to `newLit(s)`.

---

### `newIntLitNode`

```nim
proc newIntLitNode*(i: BiggestInt): NimNode
```

Creates an `nnkIntLit` node. Equivalent to `newLit(i)`.

---

### `newFloatLitNode`

```nim
proc newFloatLitNode*(f: BiggestFloat): NimNode
```

Creates an `nnkFloatLit` node.

---

### `newCommentStmtNode`

```nim
proc newCommentStmtNode*(s: string): NimNode
```

Creates an `nnkCommentStmt` node containing the given string. Useful for embedding documentation comments into generated code.

---

### `newNilLit`

```nim
proc newNilLit*(): NimNode
```

Creates a `nil` literal node (`nnkNilLit`).

---

### `newEmptyNode`

```nim
proc newEmptyNode*(): NimNode
```

Creates an `nnkEmpty` node — the placeholder used for absent optional fields in the AST (e.g. a missing return type, a missing default value). Many `newXxx` helpers use `newEmptyNode()` internally.

---

### `newStmtList`

```nim
proc newStmtList*(stmts: varargs[NimNode]): NimNode
```

Creates an `nnkStmtList` node containing the given statements. The bread-and-butter of macro return values.

```nim
result = newStmtList(
  newVarStmt(ident("x"), newLit(1)),
  newCall("echo", ident("x"))
)
# Generates:
#   var x = 1
#   echo x
```

---

### `newProc`

```nim
proc newProc*(name = newEmptyNode();
              params: openArray[NimNode] = [newEmptyNode()];
              body: NimNode = newStmtList();
              procType = nnkProcDef;
              pragmas: NimNode = newEmptyNode()): NimNode
```

High-level constructor for procedure definitions. `params[0]` is the return type; subsequent elements are `nnkIdentDefs` parameter definitions. `procType` selects which kind of routine to create (proc, func, method, iterator, etc.).

```nim
# Generates: proc double(x: int): int = x * 2
let p = newProc(
  name   = ident("double"),
  params = [ident("int"),
            newIdentDefs(ident("x"), ident("int"))],
  body   = newTree(nnkInfix, ident("*"), ident("x"), newLit(2))
)
```

---

### `newIfStmt`

```nim
proc newIfStmt*(branches: varargs[tuple[cond, body: NimNode]]): NimNode
```

Creates an `nnkIfStmt` node from condition-body pairs. Each tuple becomes one `elif` branch.

```nim
let check = newIfStmt(
  (cond: ident("flag"), body: newCall("echo", newLit("on")))
)
# Generates: if flag: echo "on"
```

---

### `newEnum`

```nim
proc newEnum*(name: NimNode, fields: openArray[NimNode],
              public, pure: bool): NimNode
```

Creates a complete `type … = enum` section. `name` must be `nnkIdent`. Fields can be plain `nnkIdent` nodes or `nnkEnumFieldDef` nodes (for explicit values).

```nim
let e = newEnum(ident("Color"), [ident("Red"), ident("Green"), ident("Blue")],
                public = true, pure = false)
# Generates: type Color* = enum Red, Green, Blue
```

---

### `newVarStmt`, `newLetStmt`, `newConstStmt`

```nim
proc newVarStmt*(name, value: NimNode): NimNode
proc newLetStmt*(name, value: NimNode): NimNode
proc newConstStmt*(name, value: NimNode): NimNode
```

Create single-name variable, let, or const declarations. The type is inferred; use `newNimNode(nnkVarSection)` manually if you need explicit types.

```nim
newVarStmt(ident("count"), newLit(0))  # var count = 0
newLetStmt(ident("pi"), newLit(3.14))  # let pi = 3.14
```

---

### `newAssignment`

```nim
proc newAssignment*(lhs, rhs: NimNode): NimNode
```

Creates an `nnkAsgn` node: `lhs = rhs`.

---

### `newDotExpr`

```nim
proc newDotExpr*(a, b: NimNode): NimNode
```

Creates `a.b` — an `nnkDotExpr` node for field access or method calls.

```nim
newDotExpr(ident("obj"), ident("field"))  # obj.field
```

---

### `newColonExpr`

```nim
proc newColonExpr*(a, b: NimNode): NimNode
```

Creates `a: b` — an `nnkExprColonExpr`, used in named arguments and object constructors.

---

### `newIdentDefs`

```nim
proc newIdentDefs*(name, kind: NimNode;
                   default = newEmptyNode()): NimNode
```

Creates a three-child `nnkIdentDefs` node: `name: kind = default`. Used as parameter and variable declarations inside `nnkFormalParams` and `nnkVarSection`. Either `kind` or `default` may be `newEmptyNode()`.

```nim
newIdentDefs(ident("x"), ident("int"))           # x: int
newIdentDefs(ident("y"), newEmptyNode(), newLit(0)) # y = 0
```

---

### `newBlockStmt`

```nim
proc newBlockStmt*(label, body: NimNode): NimNode
proc newBlockStmt*(body: NimNode): NimNode
```

Creates a `block: …` statement, with an optional label.

---

### `newPar`

```nim
proc newPar*(exprs: NimNode): NimNode
```

Creates a parenthesised expression node. Note: do not use `nnkPar` for tuple construction — use `nnkTupleConstr` instead.

---

### `nestList`

```nim
proc nestList*(op: NimNode; pack: NimNode): NimNode
proc nestList*(op: NimNode; pack: NimNode; init: NimNode): NimNode
```

Folds a list of nodes into a right-associated tree of binary calls: `[a, b, c]` → `op(a, op(b, c))`. With `init`, produces `op(a, op(b, op(c, init)))`. This is the AST equivalent of a fold/reduce.

```nim
# Generate: a and b and c
let combined = nestList(ident("and"), conditions)
```

---

### `prefix`, `postfix`, `infix`

```nim
proc prefix*(node: NimNode; op: string): NimNode   # op node
proc postfix*(node: NimNode; op: string): NimNode  # node op
proc infix*(a: NimNode; op: string; b: NimNode): NimNode  # a op b
```

Convenience constructors for operator expressions.

```nim
let negated = prefix(ident("x"), "not")   # not x
let exported = postfix(ident("foo"), "*")  # foo* (export marker)
let added = infix(ident("a"), "+", ident("b")) # a + b
```

---

### `unpackPrefix`, `unpackPostfix`, `unpackInfix`

```nim
proc unpackPrefix*(node: NimNode): tuple[node: NimNode; op: string]
proc unpackPostfix*(node: NimNode): tuple[node: NimNode; op: string]
proc unpackInfix*(node: NimNode): tuple[left: NimNode; op: string; right: NimNode]
```

The reverse of the construction helpers — decompose an operator node back into its parts.

```nim
let (left, op, right) = infix(a, "+", b).unpackInfix
assert op == "+"
```

---

### `add`

```nim
proc add*(father, child: NimNode): NimNode {.discardable.}
proc add*(father: NimNode, children: varargs[NimNode]): NimNode {.discardable.}
```

Appends one or more children to `father`. Returns `father` so calls can be chained.

```nim
let list = newStmtList()
list.add(newCall("echo", newLit("a")))
list.add(newCall("echo", newLit("b")))
```

---

### `del`

```nim
proc del*(father: NimNode, idx = 0, n = 1)
```

Deletes `n` consecutive children starting at index `idx`.

---

### `insert`

```nim
proc insert*(a: NimNode; pos: int; b: NimNode)
```

Inserts node `b` into `a` at position `pos`, shifting subsequent children right. If `pos` is past the end, empty nodes are inserted to fill the gap.

---

### `copyNimNode`

```nim
proc copyNimNode*(n: NimNode): NimNode
```

Shallow copy: copies `n` itself but **not** its children. The new node has the same kind and payload but zero children.

---

### `copyNimTree`

```nim
proc copyNimTree*(n: NimNode): NimNode
```

Deep copy: copies `n` and all its children recursively. Use this when you need to modify a subtree without affecting the original.

---

### `copy`

```nim
proc copy*(node: NimNode): NimNode
```

Alias for `copyNimTree`.

---

### `copyChildrenTo`

```nim
proc copyChildrenTo*(src, dest: NimNode)
```

Deep-copies all children of `src` and appends them to `dest`. Useful for merging subtrees.

---

## Routine (proc/func/method) accessors

These procs read and write the named sub-nodes of a routine definition node. They work on any node in `RoutineNodes`.

### `name` / `name=`

```nim
proc name*(someProc: NimNode): NimNode
proc `name=`*(someProc: NimNode; val: NimNode)
```

Gets or sets the routine's name node. Automatically unwraps `nnkPostfix` (export marker) and `nnkAccQuoted` (backtick names) to return the bare identifier.

---

### `params` / `params=`

```nim
proc params*(someProc: NimNode): NimNode
proc `params=`*(someProc: NimNode; params: NimNode)
```

Gets or replaces the `nnkFormalParams` node of a routine (or proc type). `params[0]` is the return type; subsequent children are `nnkIdentDefs` nodes.

---

### `pragma` / `pragma=`

```nim
proc pragma*(someProc: NimNode): NimNode
proc `pragma=`*(someProc: NimNode; val: NimNode)
```

Gets or replaces the `nnkPragma` node of a routine.

---

### `addPragma`

```nim
proc addPragma*(someProc, pragma: NimNode)
```

Appends a single pragma to a routine's pragma list, creating the `nnkPragma` node if it does not exist yet.

```nim
myProc.addPragma(ident("noSideEffect"))
```

---

### `body` / `body=`

```nim
proc body*(someProc: NimNode): NimNode
proc `body=`*(someProc: NimNode, val: NimNode)
```

Gets or replaces the body of a routine, `block`, `while`, or `for` node. For routines, this is the statement list at index 6 of the AST.

---

### `basename` / `basename=`

```nim
proc basename*(a: NimNode): NimNode
proc `basename=`*(a: NimNode; val: string)
```

Extracts or sets the identifier from a node that may be wrapped in `nnkPrefix`, `nnkPostfix`, or `nnkPragmaExpr`. For example, from `foo*` (a public export) it returns the `foo` identifier.

---

### `hasArgOfName`

```nim
proc hasArgOfName*(params: NimNode; name: string): bool
```

Searches an `nnkFormalParams` node to check whether a parameter with the given name exists.

---

### `addIdentIfAbsent`

```nim
proc addIdentIfAbsent*(dest: NimNode, ident: string)
```

Appends `ident` to `dest` (an `nnkPragma` or similar list) only if it is not already present. Prevents duplicates when conditionally adding pragmas.

---

## Iteration

### `items`

```nim
iterator items*(n: NimNode): NimNode
```

Iterates over the direct children of `n`.

```nim
for child in callNode:
  echo child.kind
```

---

### `pairs`

```nim
iterator pairs*(n: NimNode): (int, NimNode)
```

Iterates with index over the direct children of `n`.

---

### `children`

```nim
iterator children*(n: NimNode): NimNode
```

Equivalent to `items`. A more explicit alias for readability.

---

### `findChild`

```nim
template findChild*(n: NimNode; cond: untyped): NimNode
```

Returns the first child of `n` satisfying `cond`, or `nil` if none is found. The template injects `it` as the current child.

```nim
# Find the first exported parameter
let exported = findChild(params, it.kind == nnkPostfix)
```

---

## Quasi-quoting and code generation

### `quote`

```nim
proc quote*(bl: typed, op = "``"): NimNode
```

**Quasi-quoting**: write Nim code as-is and get back its AST. Backtick expressions inside the quoted block interpolate `NimNode` values from the surrounding scope.

This is the most readable way to produce complex ASTs: write what you want the generated code to look like, then splice in the variable parts.

```nim
macro assert2(cond: untyped): untyped =
  let msg = newLit("Assertion failed: " & cond.repr)
  result = quote do:
    if not `cond`:
      raise newException(AssertionError, `msg`)
```

A custom interpolation operator can be supplied to avoid conflicts with Nim operators in the quoted block:

```nim
result = quote("@") do:
  let value = @expr + 1   # @ is the interpolation operator here
```

---

### `getAst`

```nim
proc getAst*(macroOrTemplate: untyped): NimNode
```

Expands a macro or template call and returns the resulting AST node, without injecting it into the surrounding code. Useful for inspecting or further transforming the output of another macro.

---

## Validation

### `expectKind`

```nim
proc expectKind*(n: NimNode, k: NimNodeKind)
proc expectKind*(n: NimNode; k: set[NimNodeKind])
```

Asserts that `n` has the expected kind (or is one of a set of kinds). If not, compilation aborts with a descriptive error message pointing to `n`'s source location. The fundamental guard in any macro that validates its input.

```nim
proc myMacroHelper(n: NimNode) =
  n.expectKind(nnkProcDef)
  # safe to read n[6] (body) now
```

---

### `expectLen`

```nim
proc expectLen*(n: NimNode, len: int)
proc expectLen*(n: NimNode, min, max: int)
```

Asserts that `n` has exactly `len` children, or between `min` and `max`. Aborts with an error otherwise.

---

### `expectMinLen`

```nim
proc expectMinLen*(n: NimNode, min: int)
```

Asserts that `n` has at least `min` children.

---

### `expectIdent`

```nim
proc expectIdent*(n: NimNode, name: string)
```

Asserts that `eqIdent(n, name)` holds. Use this to check that a specific keyword or name appears where expected in the input AST.

---

## Compile-time diagnostics

### `error`

```nim
proc error*(msg: string, n: NimNode = nil)
```

Aborts compilation with an error message. The optional `n` provides source location for the error arrow in the compiler output. This is how macros report usage mistakes to their callers.

```nim
if n.kind != nnkProcDef:
  error("Expected a proc definition", n)
```

---

### `warning`

```nim
proc warning*(msg: string, n: NimNode = nil)
```

Emits a compiler warning without aborting.

---

### `hint`

```nim
proc hint*(msg: string, n: NimNode = nil)
```

Emits a compiler hint (informational message).

---

## AST visualisation and debugging

These tools are invaluable when writing macros: they show you exactly what AST structure a piece of code produces, which is what your macro must replicate or process.

### `treeRepr`

```nim
proc treeRepr*(n: NimNode): string
```

Converts a node tree to an indented, human-readable text form. Shows kinds and values. Use with `echo` inside a macro or with `dumpTree`.

```
StmtList
  VarSection
    IdentDefs
      Ident "x"
      Empty
      IntLit 42
```

---

### `lispRepr`

```nim
proc lispRepr*(n: NimNode; indented = false): string
```

S-expression form of the AST. More compact than `treeRepr`; parentheses denote nesting.

```
(StmtList (VarSection (IdentDefs (Ident "x") Empty (IntLit 42))))
```

---

### `astGenRepr`

```nim
proc astGenRepr*(n: NimNode): string
```

Converts the AST back to the **Nim code that would construct it** using `newTree`, `newLit`, etc. Copy-paste the output as a starting point for a macro.

```nim
nnkVarSection.newTree(
  nnkIdentDefs.newTree(
    newIdentNode("x"),
    newEmptyNode(),
    newLit(42)
  )
)
```

---

### `dumpTree`

```nim
macro dumpTree*(s: untyped): untyped
```

Prints the `treeRepr` of its argument **at compile time**. The essential first step when writing a new macro: wrap the code you want to process with `dumpTree` to see what AST it produces.

```nim
dumpTree:
  for i in 0 ..< 10:
    echo i
```

---

### `dumpLisp`

```nim
macro dumpLisp*(s: untyped): untyped
```

Prints the `lispRepr` at compile time.

---

### `dumpAstGen`

```nim
macro dumpAstGen*(s: untyped): untyped
```

Prints the `astGenRepr` at compile time. Use it to get a copy-paste starting point for building the same AST manually.

---

### `expandMacros`

```nim
macro expandMacros*(body: typed): untyped
```

Prints the expanded (post-macro) code of `body` at compile time, without altering what gets compiled. Useful for seeing what another macro produced.

```nim
expandMacros:
  dump(x + y)   # prints expansion of `dump` at compile time
```

---

## Source location

### `lineInfo`

```nim
proc lineInfo*(arg: NimNode): string
```

Returns `"filename(line, col)"` for the given node's source location. Use in error messages to point the user to the relevant source.

---

### `lineInfoObj`

```nim
proc lineInfoObj*(n: NimNode): LineInfo
```

Returns the structured `LineInfo` object (filename, line, column) for `n`.

---

### `copyLineInfo`

```nim
proc copyLineInfo*(arg: NimNode, info: NimNode)
```

Copies the source location from `info` onto `arg`. Use this to make generated nodes point to the correct user source location.

---

### `setLineInfo`

```nim
proc setLineInfo*(arg: NimNode, file: string, line: int, column: int)
proc setLineInfo*(arg: NimNode, lineInfo: LineInfo)
```

Explicitly sets the source location of a node. When using `quote`, add line info **after** the quote block, as `quote` overwrites it.

---

## Parsing strings into AST

### `parseExpr`

```nim
proc parseExpr*(s: string; filename: string = ""): NimNode
```

Parses a Nim expression string into an AST node. Raises `ValueError` on parse errors. Useful for dynamic code generation from string templates, though `quote` is usually cleaner.

```nim
let n = parseExpr("1 + 2 * 3")
```

---

### `parseStmt`

```nim
proc parseStmt*(s: string; filename: string = ""): NimNode
```

Like `parseExpr` but accepts one or more statements.

---

### `toStrLit`

```nim
proc toStrLit*(n: NimNode): NimNode
```

Converts a `NimNode` to its source-code `repr` string and wraps that in a string-literal node. Useful for embedding a stringified version of a node into generated code.

---

## `$` — node to string

```nim
proc `$`*(node: NimNode): string
```

Extracts the string name from identifier, symbol, string-literal, quoted, and postfix nodes. For a postfix export marker (`foo*`), appends `"*"`. Raises an error for unsuitable node kinds. Use `strVal` for literal nodes; use `$` for identifier/symbol nodes.

---

## Pragma introspection

### `hasCustomPragma`

```nim
macro hasCustomPragma*(n: typed, cp: typed{nkSym}): untyped
```

Returns `true` at compile time if `n` (a field, proc, or type) carries the custom pragma `cp`. The check works even if the pragma has arguments.

```nim
template myTag() {.pragma.}
type Foo = object
  tagged {.myTag.}: int

var o: Foo
assert o.tagged.hasCustomPragma(myTag)
```

---

### `getCustomPragmaVal`

```nim
macro getCustomPragmaVal*(n: typed, cp: typed{nkSym}): untyped
```

Retrieves the **value** of a custom pragma with arguments. If the pragma takes a single argument, returns that value directly. Multiple arguments are returned as a named tuple. Raises a compile error if the pragma is not found.

```nim
template key(s: string) {.pragma.}
type Foo = object
  field {.key: "f".}: int

var o: Foo
assert o.field.getCustomPragmaVal(key) == "f"
```

---

## Miscellaneous

### `unpackVarargs`

```nim
macro unpackVarargs*(callee: untyped; args: varargs[untyped]): untyped
```

Calls `callee` with each element of `args` as a separate argument. Solves the problem of forwarding `varargs` through an intermediate template or macro, including the edge case of an empty `varargs`.

```nim
template forward(fn: typed; args: varargs[untyped]): untyped =
  unpackVarargs(fn, args)
```

---

### `getProjectPath`

```nim
proc getProjectPath*(): string
```

Returns the directory of the **root file** being compiled (the file passed to the `nim` command). Contrast with `currentSourcePath` which returns the file containing the call site. Useful for locating adjacent resource files in a project.

---

### `extractDocCommentsAndRunnables`

```nim
proc extractDocCommentsAndRunnables*(n: NimNode): NimNode
```

Given a routine body node, returns an `nnkStmtList` containing only the leading doc-comment (`## …`) and `runnableExamples` nodes, stopping at the first statement that is neither. Used by macro-based code transformers that need to preserve documentation.

---

### `nodeID`

```nim
proc nodeID*(n: NimNode): int
```

Returns the internal compiler ID of `n` when the compiler is built with `-d:useNodeids`. Returns `-1` otherwise. Intended solely for compiler debugging.

---

## Workflow: writing a macro

```
1. Wrap sample input in dumpTree:  see the AST your macro will receive.
2. Write validation (expectKind, expectLen) at the top of your macro.
3. Build output using quote do: … for readable code,
   or newTree / newCall / newLit for precise control.
4. Use genSym for any temporaries to avoid hygiene issues.
5. Use bindSym instead of ident for symbols that must refer
   to a specific definition regardless of call-site overloads.
6. Use expandMacros or echo result.treeRepr to inspect output.
```

### Minimal worked example

```nim
import std/macros

macro repeat*(n: static int, body: untyped): untyped =
  ## Repeats `body` exactly `n` times at compile time.
  result = newStmtList()
  for _ in 0 ..< n:
    result.add(body.copyNimTree)  # deep copy — each iteration is independent

repeat(3):
  echo "hello"
# Expands to:
#   echo "hello"
#   echo "hello"
#   echo "hello"
```

### Inspecting generated code

```nim
import std/macros

dumpTree:
  if x > 0: echo "positive"

# Prints at compile time:
# IfStmt
#   ElifBranch
#     Infix
#       Ident ">"
#       Ident "x"
#       IntLit 0
#     StmtList
#       Command
#         Ident "echo"
#         StrLit "positive"
```
