# Nim `std/parsesql` — Module Reference

> **Module:** `std/parsesql`  
> **Status:** Unstable API — interface may change between Nim releases.  
> **Scope:** Parses PostgreSQL syntax and the SQL ANSI standard.

`parsesql` provides a complete SQL pipeline in three stages: **lexing** (tokenising raw text), **parsing** (building an AST from tokens), and **rendering** (converting the AST back to a SQL string). The public surface is intentionally small — most of the complexity lives in the internal lexer and parser — but understanding the AST node types is essential for working with the output.

---

## Table of Contents

1. [Types](#types)
   - [SqlLexer](#sqllexer)
   - [SqlParseError](#sqlparseerror)
   - [SqlNodeKind](#sqlnodekind)
   - [SqlNode and SqlNodeObj](#sqlnode-and-sqlnodeobj)
   - [SqlParser](#sqlparser)
2. [Constructing AST Nodes](#constructing-ast-nodes)
   - [newNode](#newnode)
   - [add](#add)
   - [len](#len)
   - [[] — index operator](#---index-operator)
3. [Parsing](#parsing)
   - [parseSql (from string)](#parsesql-from-string)
   - [parseSql (from Stream)](#parsesql-from-stream)
4. [Rendering](#rendering)
   - [renderSql](#rendersql)
   - [$](#-dollar-operator)
   - [treeRepr](#treerepr)
5. [SqlNodeKind Reference](#sqlnodekind-reference)
6. [Complete workflow example](#complete-workflow-example)

---

## Types

### `SqlLexer`

```nim
type SqlLexer* = object of BaseLexer
```

The tokeniser. It extends `BaseLexer` (from `std/lexbase`) with the SQL-specific logic for recognising all SQL token types: identifiers, quoted identifiers, string constants (plain, escape, dollar-quoted, bit, and hex), integers, numerics, operators, and punctuation.

`SqlLexer` is not normally used directly. It is embedded inside `SqlParser`, which is in turn created and managed automatically by `parseSql`. It is exposed publicly mainly to allow users to extend or subclass the parser.

---

### `SqlParseError`

```nim
type SqlParseError* = object of ValueError
```

Raised when `parseSql` encounters a syntax error. The error message includes the filename (if one was given), the line number, and the column number, in the format:

```
filename(line, column) Error: description
```

When parsing a raw string (no file), the filename portion is empty. Always wrap calls to `parseSql` in a `try/except SqlParseError` block if you need to handle malformed input gracefully.

```nim
try:
  let ast = parseSql("SELECT * FORM users")  # "FORM" is not a keyword
except SqlParseError as e:
  echo "Syntax error: ", e.msg
```

---

### `SqlNodeKind`

```nim
type SqlNodeKind* = enum
  nkNone, nkIdent, nkQuotedIdent,
  nkStringLit, nkBitStringLit, nkHexStringLit,
  nkIntegerLit, nkNumericLit,
  nkPrimaryKey, nkForeignKey, nkNotNull, nkNull,
  nkStmtList, nkDot, nkDotDot, nkPrefix, nkInfix,
  nkCall, nkPrGroup, nkColumnReference,
  nkReferences, nkDefault, nkCheck, nkConstraint,
  nkUnique, nkIdentity, nkColumnDef,
  nkInsert, nkUpdate, nkDelete,
  nkSelect, nkSelectDistinct, nkSelectColumns, nkSelectPair,
  nkAsgn, nkFrom, nkFromItemPair,
  nkJoin, nkNaturalJoin, nkUsing,
  nkGroup, nkLimit, nkOffset, nkHaving, nkOrder, nkDesc,
  nkUnion, nkIntersect, nkExcept,
  nkColumnList, nkValueList, nkWhere,
  nkCreateTable, nkCreateTableIfNotExists,
  nkCreateType, nkCreateTypeIfNotExists,
  nkCreateIndex, nkCreateIndexIfNotExists,
  nkEnumDef
```

The complete enumeration of all node kinds in the SQL AST. Every `SqlNode` has a `kind` field of this type. There are two broad families:

**Leaf nodes** (collectively called `LiteralNodes`): `nkIdent`, `nkQuotedIdent`, `nkStringLit`, `nkBitStringLit`, `nkHexStringLit`, `nkIntegerLit`, `nkNumericLit`. These carry their textual value in the `strVal` field and have no children (`len` is always 0).

**Interior nodes**: everything else. These carry their logical sub-structure in the `sons: seq[SqlNode]` field.

See the [SqlNodeKind Reference](#sqlnodekind-reference) section for a description of every value and its children.

---

### `SqlNode` and `SqlNodeObj`

```nim
type
  SqlNode* = ref SqlNodeObj
  SqlNodeObj* = object
    case kind*: SqlNodeKind
    of LiteralNodes:
      strVal*: string
    else:
      sons*: seq[SqlNode]
```

A node in the SQL abstract syntax tree. `SqlNode` is a reference type — copying a `SqlNode` variable copies the reference, not the node itself.

**Accessing data:**
- For leaf nodes, read `node.strVal` to get the token text.
- For interior nodes, iterate over `node.sons` or use `node[i]` to access children.
- Always check `node.kind` before accessing `strVal` or `sons`, because the variant object makes only one field valid at a time.

```nim
let ast = parseSql("SELECT id FROM users")
# ast.kind == nkStmtList
# ast[0].kind == nkSelect
# ast[0][0].kind == nkSelectColumns
# ast[0][0][0].kind == nkSelectPair
# ast[0][0][0][0].kind == nkIdent
# ast[0][0][0][0].strVal == "id"
```

---

### `SqlParser`

```nim
type SqlParser* = object of SqlLexer
  tok: Token
  considerTypeParams: bool
```

The parser object that embeds the lexer. Normally created and destroyed automatically inside `parseSql`. The `considerTypeParams` flag (accessible via `parseSql`'s parameter of the same name) controls whether column type parameters such as the `255` in `VARCHAR(255)` are retained in the AST or silently discarded.

When `considerTypeParams = false` (default), `VARCHAR(255)` yields a simple `nkIdent` node with `strVal = "varchar"`. When `true`, it yields an `nkCall` node whose first child is the type name and whose remaining children are the size arguments.

---

## Constructing AST Nodes

These procs let you build or manipulate an AST manually, which is useful for code generation, AST transformations, or building SQL programmatically.

### `newNode`

```nim
proc newNode*(k: SqlNodeKind): SqlNode
proc newNode*(k: SqlNodeKind, s: string): SqlNode
proc newNode*(k: SqlNodeKind, sons: seq[SqlNode]): SqlNode
```

Creates a new `SqlNode` of kind `k`. Three overloads:

- **No extra arguments:** creates an empty leaf or an interior node with no children yet. For literal kinds this sets `strVal = ""`; for others `sons = @[]`.
- **With `s: string`:** creates a leaf node with `strVal = s`. Only meaningful for `LiteralNodes`.
- **With `sons: seq[SqlNode]`:** creates an interior node with the given children already attached.

```nim
let ident = newNode(nkIdent, "users")       # nkIdent, strVal = "users"
let select = newNode(nkSelect)              # nkSelect, no children yet
let infix = newNode(nkInfix, @[
  newNode(nkIdent, "="),
  newNode(nkIdent, "id"),
  newNode(nkIntegerLit, "42")
])
```

---

### `add`

```nim
proc add*(father, n: SqlNode)
```

Appends `n` as the last child of `father`. `father` must be an interior node (not a `LiteralNode`) — adding children to a leaf node would access `sons` on a variant field that doesn't exist.

```nim
let selectCols = newNode(nkSelectColumns)
selectCols.add(newNode(nkSelectPair, @[newNode(nkIdent, "*")]))
```

---

### `len`

```nim
proc len*(n: SqlNode): int
```

Returns the number of children of `n`. For leaf (`LiteralNodes`) nodes, always returns 0 regardless of whether `sons` would be accessible. For interior nodes, returns `n.sons.len`.

```nim
let ast = parseSql("SELECT a, b FROM t")
echo ast.len          # 1 (one statement)
echo ast[0].len       # several children: SelectColumns, From, ...
echo ast[0][0][0].len # 0 if it's an nkIdent leaf
```

---

### `[]` — index operator

```nim
proc `[]`*(n: SqlNode; i: int): SqlNode
proc `[]`*(n: SqlNode; i: BackwardsIndex): SqlNode
```

Accesses the `i`-th child of `n` by index (zero-based). The `BackwardsIndex` overload supports the `n[^1]` syntax (last child). These are simple wrappers around `n.sons[i]` and `n.sons[n.len - int(i)]` respectively; they will panic on out-of-range access.

```nim
let ast = parseSql("SELECT x FROM t")
let firstStmt = ast[0]       # the nkSelect node
let lastChild = ast[^1]      # same if there's only one statement
```

---

## Parsing

### `parseSql` (from string)

```nim
proc parseSql*(input: string, filename = "", considerTypeParams = false): SqlNode
```

The primary entry point. Parses the SQL text in `input` and returns the root of the AST — always an `nkStmtList` node, even for a single statement.

**Parameters:**
- `input` — the SQL text to parse.
- `filename` — used only in error messages; pass a file path if you read the SQL from disk, or leave empty.
- `considerTypeParams` — when `true`, column/type size parameters like `VARCHAR(255)` are represented as `nkCall` nodes in the AST. When `false` (default), the size is parsed but discarded.

**Returns:** `SqlNode` with `kind == nkStmtList`, whose children are the individual parsed statements.

**Raises:** `SqlParseError` for any syntax error.

```nim
import std/parsesql

let ast = parseSql("SELECT id, name FROM users WHERE active = 1")
echo ast.kind             # nkStmtList
echo ast.len              # 1
echo ast[0].kind          # nkSelect

# Multiple statements separated by semicolons:
let multi = parseSql("INSERT INTO t VALUES (1); SELECT * FROM t;")
echo multi.len            # 2
```

---

### `parseSql` (from Stream)

```nim
proc parseSql*(input: Stream, filename: string, considerTypeParams = false): SqlNode
```

Same as the string overload but reads from a `std/streams.Stream`. Useful when parsing large SQL files without reading the entire file into memory first.

```nim
import std/[parsesql, streams]

let f = newFileStream("schema.sql", fmRead)
defer: f.close()
let ast = parseSql(f, "schema.sql")
```

---

## Rendering

### `renderSql`

```nim
proc renderSql*(n: SqlNode, upperCase = false): string
```

Converts an `SqlNode` (typically the root `nkStmtList` returned by `parseSql`, but any sub-node works) back into a SQL string. The output is compact — no extra indentation or newlines — but syntactically correct and round-trippable.

**`upperCase`:** when `true`, all SQL keywords (`SELECT`, `FROM`, `WHERE`, etc.) are emitted in upper case. When `false` (default), keywords are lower case.

The renderer handles every `SqlNodeKind` in the grammar and applies the following conventions:
- Plain identifiers are emitted as-is unless they contain non-printable characters, in which case they are double-quoted.
- Quoted identifiers are always double-quoted, with internal `"` doubled.
- String literals are single-quoted with control characters escaped as `\xNN`.
- Bit string literals are prefixed `b'...'`; hex string literals `x'...'`.
- Identifiers that are reserved keywords (currently `select`, `from`, `where`, `group`, `limit`, `offset`, `having`, `count`) are automatically double-quoted when emitted as identifiers to avoid ambiguity.

```nim
let ast = parseSql("select id , name from users where id=42")
echo renderSql(ast)
# → "select id,name from users where id = 42"

echo renderSql(ast, upperCase = true)
# → "SELECT id,name FROM users WHERE id = 42"
```

Round-trip fidelity: parsing and re-rendering produces functionally equivalent SQL, but whitespace and letter case of literals are normalised. Comments are stripped (the lexer discards them).

---

### `$` (dollar operator)

```nim
proc `$`*(n: SqlNode): string
```

An alias for `renderSql(n)` with `upperCase = false`. Makes `SqlNode` compatible with `echo`, string interpolation, and any proc expecting a `$`-able type.

```nim
let ast = parseSql("DELETE FROM logs WHERE age > 30")
echo ast          # uses $, same as renderSql(ast)
echo $ast[0]      # render just the DELETE node
```

---

### `treeRepr`

```nim
proc treeRepr*(s: SqlNode): string
```

Returns a human-readable, indented tree representation of the AST — one node per line, with children indented by two spaces per level. For leaf nodes, the `strVal` is included on the same line after the kind name. This is invaluable for debugging: use it to understand exactly what structure `parseSql` produces for a given query.

```nim
let ast = parseSql("SELECT a + 1 FROM t")
echo treeRepr(ast)
```

Example output:
```
nkStmtList
  nkSelect
    nkSelectColumns
      nkSelectPair
        nkInfix
          nkIdent +
          nkIdent a
          nkIntegerLit 1
    nkFrom
      nkFromItemPair
        nkIdent t
```

---

## SqlNodeKind Reference

This section describes every `SqlNodeKind` value, what SQL construct it represents, and how its children are arranged.

### Leaf nodes (LiteralNodes)

These all have `strVal` and no children.

| Kind | Represents | `strVal` contents |
|---|---|---|
| `nkNone` | Placeholder / absent optional clause | `""` |
| `nkIdent` | Unquoted identifier or keyword-as-ident | The identifier text, lowercased by the parser |
| `nkQuotedIdent` | Double-quoted identifier `"name"` | The identifier text without quotes |
| `nkStringLit` | Single-quoted string `'abc'`, escape string `e'abc'`, or dollar-quoted `$$abc$$` | The decoded string content |
| `nkBitStringLit` | Bit string constant `B'0101'` | The bit digits without prefix/quotes |
| `nkHexStringLit` | Hex string constant `X'1F'` | The hex digits without prefix/quotes |
| `nkIntegerLit` | Integer literal `42` | The literal text |
| `nkNumericLit` | Floating-point or scientific literal `3.14`, `1e5` | The literal text |

---

### Constraint and column-level nodes

| Kind | Children | Represents |
|---|---|---|
| `nkPrimaryKey` | Ident children (table-level) or none (column-level) | `PRIMARY KEY` |
| `nkForeignKey` | Ident list + `nkReferences` child | `FOREIGN KEY ... REFERENCES ...` |
| `nkNotNull` | none | `NOT NULL` |
| `nkNull` | none | `NULL` |
| `nkUnique` | none (column-level) or ident list | `UNIQUE` |
| `nkIdentity` | none | `IDENTITY` |
| `nkDefault` | expression child | `DEFAULT <expr>` |
| `nkCheck` | expression child | `CHECK (<expr>)` |
| `nkConstraint` | `nkIdent` name + expression | `CONSTRAINT name CHECK <expr>` |
| `nkReferences` | `nkColumnReference` child | `REFERENCES table(col)` |
| `nkColumnReference` | table name + one or more column names | Column reference with optional parenthesised list |
| `nkColumnDef` | name + type + zero or more constraint nodes | A single column definition in `CREATE TABLE` |

---

### Expression nodes

| Kind | Children | Represents |
|---|---|---|
| `nkPrefix` | operator `nkIdent` + operand | Unary prefix expression: `-x`, `NOT x` |
| `nkInfix` | operator `nkIdent` + left + right | Binary infix expression: `a + b`, `x = 42` |
| `nkCall` | function name + zero or more argument nodes | Function call: `count(*)`, `substr(s, 1, 3)` |
| `nkPrGroup` | one or more expressions | Parenthesised expression group `(a, b)` |
| `nkDot` | left + right | Qualified name: `schema.table` or `table.column` |
| `nkDotDot` | left + right | Dotdot notation (rare PostgreSQL extension) |
| `nkAsgn` | left + right | Assignment `col = expr` (used in `UPDATE SET`) |

**Operator precedence** (from lowest to highest as parsed):
`OR` → `AND` → `BETWEEN` → user-defined operators → comparisons/`IS`/`LIKE`/`IN` → `+`/`-` → `*`/`/`/`%`

---

### Statement nodes

#### SELECT

| Kind | Children | Represents |
|---|---|---|
| `nkSelect` | `nkSelectColumns` + optional clauses below | `SELECT` statement |
| `nkSelectDistinct` | same as `nkSelect` | `SELECT DISTINCT` statement |
| `nkSelectColumns` | one or more `nkSelectPair` or `nkIdent` (`*`) | The column list |
| `nkSelectPair` | expr + optional alias expr | `expr [AS alias]` in the column list |
| `nkFrom` | one or more `nkFromItemPair` or join nodes | `FROM` clause |
| `nkFromItemPair` | table/subquery + optional alias | A single item in `FROM`, `FROM t AS x` |
| `nkJoin` | join-type `nkIdent` + left + right + ON/USING | An explicit join: `LEFT JOIN t ON ...` |
| `nkNaturalJoin` | join-type `nkIdent` + left + right | `NATURAL [type] JOIN` |
| `nkUsing` | one or more column `nkIdent`s | `USING (col1, col2)` |
| `nkWhere` | single expression | `WHERE <expr>` |
| `nkGroup` | one or more expressions | `GROUP BY expr, ...` |
| `nkOrder` | one or more expr or `nkDesc` | `ORDER BY expr [DESC], ...` |
| `nkDesc` | single expression child | Wraps an `ORDER BY` expression with `DESC` |
| `nkHaving` | one or more expressions | `HAVING expr, ...` |
| `nkLimit` | single expression | `LIMIT expr` |
| `nkOffset` | single expression | `OFFSET expr` |
| `nkUnion` | none | `UNION` keyword marker |
| `nkIntersect` | none | `INTERSECT` keyword marker |
| `nkExcept` | none | `EXCEPT` keyword marker |

#### INSERT

| Kind | Children | Represents |
|---|---|---|
| `nkInsert` | table name + `nkColumnList`/`nkNone` + `nkValueList`/`nkDefault` | `INSERT INTO table [(cols)] VALUES (...)` or `INSERT INTO table DEFAULT VALUES` |
| `nkColumnList` | one or more column `nkIdent`s | The `(col1, col2)` list after the table name |
| `nkValueList` | one or more expressions | The `VALUES (expr, ...)` list |

#### UPDATE

| Kind | Children | Represents |
|---|---|---|
| `nkUpdate` | table name + one or more `nkAsgn` + `nkWhere`/`nkNone` | `UPDATE table SET col=expr [WHERE ...]` |

#### DELETE

| Kind | Children | Represents |
|---|---|---|
| `nkDelete` | table name + `nkWhere`/`nkNone` | `DELETE FROM table [WHERE ...]` |

#### DDL (Data Definition)

| Kind | Children | Represents |
|---|---|---|
| `nkCreateTable` | table name + column/constraint nodes | `CREATE TABLE name (...)` |
| `nkCreateTableIfNotExists` | same | `CREATE TABLE IF NOT EXISTS name (...)` |
| `nkCreateType` | type name + `nkEnumDef` or data type | `CREATE TYPE name AS ...` |
| `nkCreateTypeIfNotExists` | same | `CREATE TYPE IF NOT EXISTS name AS ...` |
| `nkCreateIndex` | index name + table name + column names | `CREATE INDEX name ON table (cols)` |
| `nkCreateIndexIfNotExists` | same | `CREATE INDEX IF NOT EXISTS name ON ...` |
| `nkEnumDef` | one or more `nkStringLit` children | `ENUM ('val1', 'val2', ...)` |

#### Top level

| Kind | Children | Represents |
|---|---|---|
| `nkStmtList` | one or more statement nodes | The root node; holds all parsed statements |

---

## Complete Workflow Example

```nim
import std/parsesql

# 1. Parse
let sql = """
  CREATE TABLE users (
    id   INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    role VARCHAR(20) DEFAULT 'viewer'
  );
  SELECT u.id, u.name
  FROM   users AS u
  WHERE  u.role = 'admin'
  ORDER  BY u.name DESC
  LIMIT  10;
"""

let ast = parseSql(sql, filename = "example.sql", considerTypeParams = false)

# 2. Inspect the tree
echo treeRepr(ast)

# 3. Traverse: count SELECT nodes
proc countSelects(n: SqlNode): int =
  if n == nil: return 0
  if n.kind == nkSelect or n.kind == nkSelectDistinct:
    result = 1
  if n.kind notin {nkIdent, nkQuotedIdent, nkStringLit,
                   nkBitStringLit, nkHexStringLit,
                   nkIntegerLit, nkNumericLit}:
    for child in n.sons:
      result += countSelects(child)

echo "SELECT statements: ", countSelects(ast)  # 1

# 4. Modify: change LIMIT value
let selectNode = ast[1]  # second statement
for child in selectNode.sons:
  if child.kind == nkLimit:
    child.sons[0] = newNode(nkIntegerLit, "25")  # change LIMIT 10 → 25

# 5. Render
echo renderSql(ast, upperCase = true)

# 6. Handle syntax errors
try:
  discard parseSql("SELECT FORM users")
except SqlParseError as e:
  echo e.msg  # example.sql(1, 8) Error: ...
```

---

## Quick Reference

| Task | API |
|---|---|
| Parse SQL string to AST | `parseSql(sql)` or `parseSql(sql, filename)` |
| Parse SQL stream to AST | `parseSql(stream, filename)` |
| Convert AST to SQL string | `renderSql(node)` or `$node` |
| Render with uppercase keywords | `renderSql(node, upperCase = true)` |
| Debug-print the AST structure | `treeRepr(node)` |
| Create a new node | `newNode(kind)`, `newNode(kind, strVal)`, `newNode(kind, sons)` |
| Add a child node | `father.add(child)` |
| Count children | `node.len` |
| Access i-th child | `node[i]`, `node[^1]` |
| Read leaf value | `node.strVal` (only for `LiteralNodes`) |
| Check node type | `node.kind == nkSelect` etc. |
| Catch syntax errors | `try: parseSql(...) except SqlParseError: ...` |
| Retain type parameters (e.g. VARCHAR(255)) | `parseSql(sql, considerTypeParams = true)` |
