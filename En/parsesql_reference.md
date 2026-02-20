# `std/parsesql` Module Reference (Nim)

> A high-performance SQL parser supporting PostgreSQL syntax and the ANSI SQL standard. The result of parsing is an **abstract syntax tree** (AST) rooted at a `SqlNode`.

> ⚠️ **Unstable API.** This module is marked as deprecated. Use `nimble install parsesql` and import `pkg/parsesql` instead.

---

## Table of Contents

1. [Deprecation Notice](#deprecation-notice)
2. [Core Types](#core-types)
   - [SqlNodeKind](#sqlnodekind)
   - [SqlNode](#sqlnode)
   - [SqlParseError](#sqlparseerror)
   - [SqlParser and SqlLexer](#sqlparser-and-sqllexer)
3. [Creating AST Nodes](#creating-ast-nodes)
4. [Working with AST Nodes](#working-with-ast-nodes)
5. [Parsing SQL](#parsing-sql)
6. [Rendering Back to SQL](#rendering-back-to-sql)
7. [Debugging and Tree Visualization](#debugging-and-tree-visualization)
8. [SqlNodeKind Reference Table](#sqlnodekind-reference-table)
9. [Practical Examples](#practical-examples)

---

## Deprecation Notice

```nim
# Importing the built-in module will produce a deprecation warning:
import std/parsesql  # deprecated

# Correct approach:
# nimble install parsesql
import pkg/parsesql
```

---

## Core Types

### `SqlNodeKind`

An enum describing all possible kinds of SQL AST nodes.

```nim
type SqlNodeKind* = enum
  nkNone,             # empty / absent node
  nkIdent,            # plain identifier (table name, column name, etc.)
  nkQuotedIdent,      # quoted identifier: "MyTable"
  nkStringLit,        # string literal: 'hello'
  nkBitStringLit,     # bit string: B'0101'
  nkHexStringLit,     # hex string: x'FF'
  nkIntegerLit,       # integer literal: 42
  nkNumericLit,       # float literal: 3.14
  nkPrimaryKey,       # PRIMARY KEY
  nkForeignKey,       # FOREIGN KEY
  nkNotNull,          # NOT NULL
  nkNull,             # NULL
  nkStmtList,         # statement list (tree root)
  nkDot,              # dot operator: table.column
  nkDotDot,           # double dot: schema..table
  nkPrefix,           # unary prefix: -x, NOT x
  nkInfix,            # binary infix: a + b, a AND b
  nkCall,             # function call: COUNT(*)
  nkPrGroup,          # parenthesized group: (a, b)
  nkColumnReference,  # column reference with explicit table
  nkReferences,       # REFERENCES table(col)
  nkDefault,          # DEFAULT value / DEFAULT VALUES
  nkCheck,            # CHECK (expr)
  nkConstraint,       # CONSTRAINT name CHECK (expr)
  nkUnique,           # UNIQUE
  nkIdentity,         # IDENTITY
  nkColumnDef,        # column definition: name type constraints
  nkInsert,           # INSERT INTO
  nkUpdate,           # UPDATE
  nkDelete,           # DELETE FROM
  nkSelect,           # SELECT
  nkSelectDistinct,   # SELECT DISTINCT
  nkSelectColumns,    # column list in SELECT
  nkSelectPair,       # expression [AS alias] in SELECT
  nkAsgn,             # assignment in UPDATE: col = val
  nkFrom,             # FROM clause
  nkFromItemPair,     # FROM item [AS alias]
  nkJoin,             # JOIN (all types)
  nkNaturalJoin,      # NATURAL JOIN
  nkUsing,            # USING (col)
  nkGroup,            # GROUP BY
  nkLimit,            # LIMIT
  nkOffset,           # OFFSET
  nkHaving,           # HAVING
  nkOrder,            # ORDER BY
  nkDesc,             # DESC
  nkUnion,            # UNION
  nkIntersect,        # INTERSECT
  nkExcept,           # EXCEPT
  nkColumnList,       # column list in INSERT
  nkValueList,        # VALUES (...)
  nkWhere,            # WHERE
  nkCreateTable,      # CREATE TABLE
  nkCreateTableIfNotExists,
  nkCreateType,       # CREATE TYPE
  nkCreateTypeIfNotExists,
  nkCreateIndex,      # CREATE INDEX
  nkCreateIndexIfNotExists,
  nkEnumDef           # ENUM ('a', 'b', ...)
```

**Leaf nodes** (`LiteralNodes`) store their value in the `strVal` field:
`nkIdent`, `nkQuotedIdent`, `nkStringLit`, `nkBitStringLit`, `nkHexStringLit`, `nkIntegerLit`, `nkNumericLit`.

All other nodes store child nodes in `sons: seq[SqlNode]`.

---

### `SqlNode`

A reference type representing a single node in the SQL syntax tree.

```nim
type
  SqlNode* = ref SqlNodeObj
  SqlNodeObj* = object
    case kind*: SqlNodeKind
    of LiteralNodes:
      strVal*: string       # value for leaf nodes
    else:
      sons*: seq[SqlNode]   # child nodes
```

Accessing fields:

```nim
let node = parseSql("SELECT id FROM users")
# node.kind       == nkStmtList
# node[0].kind    == nkSelect
# node[0][0].kind == nkSelectColumns
# node[0][0][0].kind  == nkSelectPair
# node[0][0][0][0].kind   == nkIdent
# node[0][0][0][0].strVal == "id"
```

---

### `SqlParseError`

Exception raised when a syntax error is encountered.

```nim
type SqlParseError* = object of ValueError
```

```nim
try:
  let ast = parseSql("SELECT FROM")   # error: missing column list
except SqlParseError as e:
  echo "Parse error: ", e.msg
```

---

### `SqlParser` and `SqlLexer`

Internal parser and lexer objects. Normally used indirectly through `parseSql`. `SqlLexer` inherits from `BaseLexer` (`std/lexbase`) and tracks file position for error messages.

```nim
type
  SqlLexer* = object of BaseLexer
    # filename: string  (internal)

  SqlParser* = object of SqlLexer
    # tok: Token        (internal)
    # considerTypeParams: bool
```

`considerTypeParams` — when `true`, type parameters such as `VARCHAR(255)` are included in the AST as `nkCall` nodes; when `false` (default), they are parsed and discarded.

---

## Creating AST Nodes

### `newNode(k: SqlNodeKind): SqlNode`

Creates an empty node of the given kind. Leaf nodes get `strVal = ""`, branch nodes get `sons = @[]`.

```nim
import std/parsesql

let selectNode = newNode(nkSelect)
let identNode  = newNode(nkIdent)
```

---

### `newNode(k: SqlNodeKind, s: string): SqlNode`

Creates a leaf node with the given string value.

```nim
let col = newNode(nkIdent, "user_id")
echo col.strVal   # user_id
echo col.kind     # nkIdent

let lit = newNode(nkIntegerLit, "42")
echo lit.strVal   # 42
```

---

### `newNode(k: SqlNodeKind, sons: seq[SqlNode]): SqlNode`

Creates a branch node with the given list of children.

```nim
let children = @[newNode(nkIdent, "a"), newNode(nkIdent, "b")]
let group = newNode(nkPrGroup, children)
echo group.len   # 2
```

---

## Working with AST Nodes

### `len(n: SqlNode): int`

Returns the number of child nodes. Always returns `0` for leaf nodes (`LiteralNodes`).

```nim
let ast = parseSql("SELECT a, b FROM t")
let select  = ast[0]      # nkSelect
let columns = select[0]   # nkSelectColumns
echo columns.len          # 2 (columns a and b)

let ident = newNode(nkIdent, "x")
echo ident.len            # 0 (leaf node)
```

---

### `[](n: SqlNode, i: int): SqlNode`

Returns the `i`-th child node (zero-based).

```nim
let ast = parseSql("SELECT id, name FROM users")
let cols = ast[0][0]    # nkSelectColumns
echo cols[0][0].strVal  # id   (nkSelectPair → nkIdent)
echo cols[1][0].strVal  # name
```

---

### `[](n: SqlNode, i: BackwardsIndex): SqlNode`

Returns a child node from the end using the `^` operator.

```nim
let ast = parseSql("SELECT a, b, c FROM t")
let cols = ast[0][0]   # nkSelectColumns
echo cols[^1][0].strVal   # c (last column)
```

---

### `add(father, n: SqlNode)`

Appends child node `n` to `father`. `father` must not be a leaf node.

```nim
var select = newNode(nkSelect)
var cols = newNode(nkSelectColumns)
cols.add(newNode(nkSelectPair, @[newNode(nkIdent, "id")]))
select.add(cols)
echo select.len   # 1
```

---

## Parsing SQL

### `parseSql(input: string, filename = "", considerTypeParams = false): SqlNode`

Parses SQL from a string and returns the AST root (`nkStmtList`). Raises `SqlParseError` on syntax errors. `filename` is used only in error messages.

```nim
import std/parsesql

# Simple SELECT
let ast = parseSql("SELECT id, name FROM users WHERE id = 1")
echo ast.kind       # nkStmtList
echo ast[0].kind    # nkSelect

# Multiple statements separated by semicolons
let multi = parseSql("""
  SELECT a FROM t1;
  SELECT b FROM t2;
""")
echo multi.len   # 2

# CREATE TABLE
let ddl = parseSql("""
  CREATE TABLE users (
    id   INTEGER PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    age  INTEGER DEFAULT 0
  );
""")
echo ddl[0].kind       # nkCreateTable
echo ddl[0][0].strVal  # users (table name)

# considerTypeParams = true — VARCHAR(255) appears in AST as nkCall
let ddlTyped = parseSql(
  "CREATE TABLE t (name VARCHAR(255))",
  considerTypeParams = true
)
let colType = ddlTyped[0][1][1]   # nkCall
echo colType.kind                  # nkCall
echo colType[0].strVal             # VARCHAR
echo colType[1].strVal             # 255

# Syntax error
try:
  discard parseSql("SELECT FROM WHERE")
except SqlParseError as e:
  echo e.msg   # input(1, 8) Error: expression expected
```

---

### `parseSql(input: Stream, filename: string, considerTypeParams = false): SqlNode`

Parses SQL from a `Stream`. The stream is closed automatically after parsing. Useful for large files.

```nim
import std/[parsesql, streams]

let s = newStringStream("SELECT * FROM orders")
let ast = parseSql(s, "orders.sql")
echo ast[0].kind   # nkSelect

# From a file
let fs = newFileStream("schema.sql", fmRead)
if fs != nil:
  let schema = parseSql(fs, "schema.sql")
  echo schema.len   # number of SQL statements
```

---

## Rendering Back to SQL

### `renderSql(n: SqlNode, upperCase = false): string`

Converts an AST back into a SQL string. When `upperCase = true`, keywords are uppercased.

```nim
import std/parsesql

let ast = parseSql("select id, name from users where id = 1")

# Lowercase (default)
echo renderSql(ast)
# select id,name from users where id = 1

# Uppercase keywords
echo renderSql(ast, upperCase = true)
# SELECT id,name FROM users WHERE id = 1

# Roundtrip: parse → render
let original = "INSERT INTO orders (id, total) VALUES (1, 99.99)"
let roundtrip = renderSql(parseSql(original))
echo roundtrip
# insert into orders(id,total) values(1,99.99)

# CREATE TABLE
let ddl = parseSql("""
  CREATE TABLE products (
    id   INTEGER PRIMARY KEY,
    name TEXT NOT NULL
  )
""")
echo renderSql(ddl, upperCase = true)
# CREATE TABLE products(id INTEGER PRIMARY KEY,name TEXT NOT NULL);
```

> **Note:** `renderSql` does not guarantee character-for-character reproduction of the original — whitespace and line breaks may differ. The function aims for semantic correctness, not formatting preservation.

---

### `$(n: SqlNode): string`

Alias for `renderSql(n)` with `upperCase = false`. Called automatically by `echo`.

```nim
import std/parsesql

let ast = parseSql("SELECT 1 + 2")
echo ast        # select 1+2
echo $ast[0]    # select 1+2
```

---

## Debugging and Tree Visualization

### `treeRepr(s: SqlNode): string`

Returns an indented string representation of the AST. Useful for debugging and understanding the tree structure.

```nim
import std/parsesql

let ast = parseSql("SELECT id, name FROM users WHERE age > 18")
echo treeRepr(ast)
```

Example output:
```
nkStmtList
  nkSelect
    nkSelectColumns
      nkSelectPair
        nkIdent id
      nkSelectPair
        nkIdent name
    nkFrom
      nkFromItemPair
        nkIdent users
    nkWhere
      nkInfix
        nkIdent >
        nkIdent age
        nkIntegerLit 18
```

```nim
# JOIN example
let ast2 = parseSql("""
  SELECT u.name, o.total
  FROM users u
  JOIN orders o ON u.id = o.user_id
""")
echo treeRepr(ast2)
```

---

## `SqlNodeKind` Reference Table

| Kind                       | SQL construct                | `sons` / `strVal`                                           |
|----------------------------|------------------------------|-------------------------------------------------------------|
| `nkStmtList`               | (root)                       | `sons`: all statements                                      |
| `nkSelect`                 | `SELECT ...`                 | `sons[0]`=columns, `sons[1..]`=from/where/etc.              |
| `nkSelectDistinct`         | `SELECT DISTINCT ...`        | same as nkSelect                                            |
| `nkSelectColumns`          | column list                  | `sons`: nkSelectPair or nkIdent(`*`)                        |
| `nkSelectPair`             | `expr [AS alias]`            | `sons[0]`=expr, `sons[1]`=alias (optional)                  |
| `nkFrom`                   | `FROM ...`                   | `sons`: nkFromItemPair                                      |
| `nkFromItemPair`           | `table [AS alias]`           | `sons[0]`=table, `sons[1]`=alias (optional)                 |
| `nkWhere`                  | `WHERE expr`                 | `sons[0]`=expr                                              |
| `nkGroup`                  | `GROUP BY ...`               | `sons`: expressions                                         |
| `nkOrder`                  | `ORDER BY ...`               | `sons`: expressions / nkDesc                                |
| `nkDesc`                   | `expr DESC`                  | `sons[0]`=expr                                              |
| `nkHaving`                 | `HAVING ...`                 | `sons`: expressions                                         |
| `nkLimit`                  | `LIMIT n`                    | `sons[0]`=expr                                              |
| `nkOffset`                 | `OFFSET n`                   | `sons[0]`=expr                                              |
| `nkJoin`                   | `[type] JOIN ... ON/USING`   | `sons[0]`=type, `sons[1]`=left, `sons[2]`=right, `sons[3]`=cond |
| `nkNaturalJoin`            | `NATURAL JOIN`               | same as nkJoin                                              |
| `nkUsing`                  | `USING (cols)`               | `sons`: nkIdent                                             |
| `nkUnion`                  | `UNION`                      | no children                                                 |
| `nkIntersect`              | `INTERSECT`                  | no children                                                 |
| `nkExcept`                 | `EXCEPT`                     | no children                                                 |
| `nkInsert`                 | `INSERT INTO`                | `sons[0]`=table, `sons[1]`=cols, `sons[2]`=values          |
| `nkUpdate`                 | `UPDATE ... SET`             | `sons[0]`=table, `sons[1..n-2]`=nkAsgn, `sons[n-1]`=where |
| `nkDelete`                 | `DELETE FROM`                | `sons[0]`=table, `sons[1]`=where                            |
| `nkAsgn`                   | `col = val`                  | `sons[0]`=col, `sons[1]`=val                                |
| `nkColumnList`             | `(col1, col2, ...)`          | `sons`: nkIdent                                             |
| `nkValueList`              | `VALUES (...)`               | `sons`: expressions                                         |
| `nkCreateTable`            | `CREATE TABLE`               | `sons[0]`=name, `sons[1..]`=column defs                    |
| `nkCreateTableIfNotExists` | `CREATE TABLE IF NOT EXISTS` | same                                                        |
| `nkColumnDef`              | `name type constraints`      | `sons[0]`=name, `sons[1]`=type, `sons[2..]`=constraints    |
| `nkPrimaryKey`             | `PRIMARY KEY`                | `sons`: (optional) column names                             |
| `nkForeignKey`             | `FOREIGN KEY`                | `sons`: columns + nkReferences                              |
| `nkReferences`             | `REFERENCES table(col)`      | `sons[0]`=column reference                                  |
| `nkNotNull`                | `NOT NULL`                   | no children                                                 |
| `nkNull`                   | `NULL`                       | no children                                                 |
| `nkUnique`                 | `UNIQUE`                     | `sons`: (optional) columns                                  |
| `nkDefault`                | `DEFAULT expr`               | `sons[0]`=expr                                              |
| `nkCheck`                  | `CHECK (expr)`               | `sons[0]`=expr                                              |
| `nkConstraint`             | `CONSTRAINT name CHECK`      | `sons[0]`=name, `sons[1]`=expr                              |
| `nkIdentity`               | `IDENTITY`                   | no children                                                 |
| `nkCreateIndex`            | `CREATE INDEX`               | `sons[0]`=index name, `sons[1]`=table, `sons[2..]`=cols    |
| `nkCreateType`             | `CREATE TYPE`                | `sons[0]`=name, `sons[1]`=type def                          |
| `nkEnumDef`                | `ENUM ('a', 'b')`            | `sons`: nkStringLit                                         |
| `nkInfix`                  | `a OP b`                     | `sons[0]`=op, `sons[1]`=left, `sons[2]`=right              |
| `nkPrefix`                 | `OP a` (unary)               | `sons[0]`=op, `sons[1]`=operand                             |
| `nkCall`                   | `func(args)`                 | `sons[0]`=name, `sons[1..]`=args                            |
| `nkDot`                    | `table.column`               | `sons[0]`=table, `sons[1]`=column                           |
| `nkDotDot`                 | `schema..table`              | `sons[0]`=schema, `sons[1]`=table                           |
| `nkPrGroup`                | `(expr, ...)`                | `sons`: expressions                                         |
| `nkIdent`                  | plain identifier             | `strVal`=name                                               |
| `nkQuotedIdent`            | `"identifier"`               | `strVal`=name without quotes                                |
| `nkStringLit`              | `'string'`                   | `strVal`=content                                            |
| `nkIntegerLit`             | `42`                         | `strVal`="42"                                               |
| `nkNumericLit`             | `3.14`                       | `strVal`="3.14"                                             |
| `nkBitStringLit`           | `B'0101'`                    | `strVal`="0101"                                             |
| `nkHexStringLit`           | `x'FF'`                      | `strVal`="FF"                                               |

---

## Practical Examples

### Traversing the AST: collect all table names from a SELECT

```nim
import std/parsesql

proc collectTables(node: SqlNode, tables: var seq[string]) =
  if node == nil: return
  if node.kind == nkFromItemPair and node.len >= 1:
    let src = node[0]
    if src.kind in {nkIdent, nkQuotedIdent}:
      tables.add(src.strVal)
  for i in 0 ..< node.len:
    collectTables(node[i], tables)

let ast = parseSql("""
  SELECT u.name, o.total
  FROM users u
  JOIN orders o ON u.id = o.user_id
""")

var tables: seq[string]
collectTables(ast, tables)
echo tables   # @["users", "orders"]
```

---

### Extracting column names from CREATE TABLE

```nim
import std/parsesql

proc getColumns(ddl: string): seq[tuple[name, typ: string]] =
  let ast = parseSql(ddl)
  let createNode = ast[0]   # nkCreateTable
  for i in 1 ..< createNode.len:
    let child = createNode[i]
    if child.kind == nkColumnDef:
      let colName = child[0].strVal
      let colType = child[1].strVal
      result.add((colName, colType))

let cols = getColumns("""
  CREATE TABLE employees (
    id         INTEGER PRIMARY KEY,
    first_name VARCHAR NOT NULL,
    salary     NUMERIC DEFAULT 0,
    dept_id    INTEGER REFERENCES departments(id)
  )
""")

for col in cols:
  echo col.name, " : ", col.typ
# id : INTEGER
# first_name : VARCHAR
# salary : NUMERIC
# dept_id : INTEGER
```

---

### Building an AST manually and rendering it

```nim
import std/parsesql

# Build: SELECT id, name FROM users WHERE id = 1

var stmt = newNode(nkSelect)

# Columns
var cols = newNode(nkSelectColumns)
cols.add(newNode(nkSelectPair, @[newNode(nkIdent, "id")]))
cols.add(newNode(nkSelectPair, @[newNode(nkIdent, "name")]))
stmt.add(cols)

# FROM
var frm = newNode(nkFrom)
frm.add(newNode(nkFromItemPair, @[newNode(nkIdent, "users")]))
stmt.add(frm)

# WHERE id = 1
var where = newNode(nkWhere)
var infix = newNode(nkInfix)
infix.add(newNode(nkIdent, "="))
infix.add(newNode(nkIdent, "id"))
infix.add(newNode(nkIntegerLit, "1"))
where.add(infix)
stmt.add(where)

var root = newNode(nkStmtList)
root.add(stmt)

echo renderSql(root, upperCase = true)
# SELECT id,name FROM users WHERE id = 1
```

---

### Roundtrip: SQL normalization

```nim
import std/parsesql

proc normalizeSql(sql: string): string =
  ## Normalizes SQL: removes extra whitespace, uppercases keywords.
  try:
    let ast = parseSql(sql)
    return renderSql(ast, upperCase = true)
  except SqlParseError as e:
    return "ERROR: " & e.msg

echo normalizeSql("  select   id,name   from   users  ")
# SELECT id,name FROM users

echo normalizeSql("insert into t (a,b) values (1,'hello')")
# INSERT INTO t(a,b) VALUES(1,'hello')

echo normalizeSql("create table x (id integer primary key)")
# CREATE TABLE x(id INTEGER PRIMARY KEY);
```

---

### Classifying the type of a SQL statement

```nim
import std/parsesql

proc classifySql(sql: string): string =
  try:
    let ast = parseSql(sql)
    if ast.len == 0: return "empty"
    case ast[0].kind
    of nkSelect, nkSelectDistinct:              "SELECT"
    of nkInsert:                                "INSERT"
    of nkUpdate:                                "UPDATE"
    of nkDelete:                                "DELETE"
    of nkCreateTable, nkCreateTableIfNotExists: "CREATE TABLE"
    of nkCreateIndex, nkCreateIndexIfNotExists: "CREATE INDEX"
    of nkCreateType,  nkCreateTypeIfNotExists:  "CREATE TYPE"
    else: "OTHER (" & $ast[0].kind & ")"
  except SqlParseError:
    "SYNTAX ERROR"

echo classifySql("SELECT 1")                        # SELECT
echo classifySql("INSERT INTO t VALUES (1)")         # INSERT
echo classifySql("CREATE TABLE t (id INT)")          # CREATE TABLE
echo classifySql("DELETE FROM t WHERE id = 5")       # DELETE
```

---

### Supported SQL Constructs

The module supports the following statements and syntax:

**DML (Data Manipulation Language):**
- `SELECT [DISTINCT] ... FROM ... [JOIN ...] [WHERE ...] [GROUP BY ...] [HAVING ...] [ORDER BY ...] [LIMIT ...] [OFFSET ...]`
- `INSERT INTO table [(cols)] VALUES (...) | DEFAULT VALUES`
- `UPDATE table SET col = val [, ...] [WHERE ...]`
- `DELETE FROM table [WHERE ...]`

**DDL (Data Definition Language):**
- `CREATE TABLE [IF NOT EXISTS] name (...)`
- `CREATE INDEX [IF NOT EXISTS] name ON table (...)`
- `CREATE TYPE [IF NOT EXISTS] name AS ...`

**Data Types:**
- Any identifier (`INTEGER`, `VARCHAR`, `TEXT`, `NUMERIC`, etc.)
- Parameterized types: `VARCHAR(255)`, `DECIMAL(10, 2)` — handled per the `considerTypeParams` flag
- `ENUM ('val1', 'val2', ...)`

**Column Constraints:**
- `PRIMARY KEY`, `FOREIGN KEY ... REFERENCES`
- `NOT NULL`, `NULL`, `UNIQUE`
- `DEFAULT expr`
- `CHECK (expr)`
- `CONSTRAINT name CHECK (expr)`
- `IDENTITY`

**JOIN Types:**
- `INNER JOIN`, `LEFT [OUTER] JOIN`, `RIGHT [OUTER] JOIN`, `FULL [OUTER] JOIN`
- `CROSS JOIN`, `NATURAL JOIN`
- `ON expr`, `USING (cols)`

**Set Operations:**
- `UNION`, `INTERSECT`, `EXCEPT`

**Operator Precedence (high to low):**

| Operators                        | Notes                                |
|----------------------------------|--------------------------------------|
| `.`                              | table/column separator               |
| `[ ]`                            | array element selection              |
| unary `-`                        | negation                             |
| `* / %`                          | multiplication, division, modulo     |
| `+ -`                            | addition, subtraction                |
| `=  <  >  >=  <=  <>  !=`       | comparison                           |
| `IS`, `LIKE`, `IN`              | predicate operators                  |
| `AND`                            | logical conjunction                  |
| `OR`                             | logical disjunction                  |
| `BETWEEN`                        | range containment                    |
