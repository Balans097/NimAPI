# Справочник модуля `std/parsesql` (Nim)

> Модуль реализует высокопроизводительный парсер SQL, поддерживающий синтаксис PostgreSQL и стандарт ANSI SQL. Результатом работы является **абстрактное синтаксическое дерево** (AST) типа `SqlNode`.

> ⚠️ **Внимание: нестабильный API.** Модуль помечен как устаревший (`deprecated`). Рекомендуется использовать `nimble install parsesql` и импортировать `pkg/parsesql`.

---

## Содержание

1. [Предупреждение об устаревании](#предупреждение-об-устаревании)
2. [Ключевые типы](#ключевые-типы)
   - [SqlNodeKind](#sqlnodekind)
   - [SqlNode](#sqlnode)
   - [SqlParseError](#sqlparseerror)
   - [SqlParser и SqlLexer](#sqlparser-и-sqllexer)
3. [Создание узлов AST](#создание-узлов-ast)
4. [Работа с узлами AST](#работа-с-узлами-ast)
5. [Парсинг SQL](#парсинг-sql)
6. [Рендеринг обратно в SQL](#рендеринг-обратно-в-sql)
7. [Отладка и визуализация дерева](#отладка-и-визуализация-дерева)
8. [Таблица SqlNodeKind с пояснениями](#таблица-sqlnodekind-с-пояснениями)
9. [Практические примеры](#практические-примеры)

---

## Предупреждение об устаревании

```nim
# Использование встроенного модуля вызовет предупреждение:
import std/parsesql  # deprecated

# Правильный способ:
# nimble install parsesql
import pkg/parsesql
```

---

## Ключевые типы

### `SqlNodeKind`

Перечисление, описывающее все возможные виды узлов SQL-дерева.

```nim
type SqlNodeKind* = enum
  nkNone,             # пустой/отсутствующий узел
  nkIdent,            # идентификатор (имя таблицы, столбца и т.д.)
  nkQuotedIdent,      # идентификатор в кавычках: "MyTable"
  nkStringLit,        # строковый литерал: 'hello'
  nkBitStringLit,     # битовая строка: B'0101'
  nkHexStringLit,     # шестнадцатеричная строка: x'FF'
  nkIntegerLit,       # целочисленный литерал: 42
  nkNumericLit,       # числовой литерал с плавающей точкой: 3.14
  nkPrimaryKey,       # PRIMARY KEY
  nkForeignKey,       # FOREIGN KEY
  nkNotNull,          # NOT NULL
  nkNull,             # NULL
  nkStmtList,         # список операторов (корень дерева)
  nkDot,              # оператор точки: table.column
  nkDotDot,           # двойная точка: schema..table
  nkPrefix,           # префиксная операция: -x, NOT x
  nkInfix,            # инфиксная операция: a + b, a AND b
  nkCall,             # вызов функции: COUNT(*)
  nkPrGroup,          # сгруппированное выражение в скобках: (a, b)
  nkColumnReference,  # ссылка на столбец с указанием таблицы
  nkReferences,       # REFERENCES table(col)
  nkDefault,          # DEFAULT value / DEFAULT VALUES
  nkCheck,            # CHECK (expr)
  nkConstraint,       # CONSTRAINT name CHECK (expr)
  nkUnique,           # UNIQUE
  nkIdentity,         # IDENTITY
  nkColumnDef,        # определение столбца: name type constraints
  nkInsert,           # INSERT INTO
  nkUpdate,           # UPDATE
  nkDelete,           # DELETE FROM
  nkSelect,           # SELECT
  nkSelectDistinct,   # SELECT DISTINCT
  nkSelectColumns,    # список столбцов в SELECT
  nkSelectPair,       # выражение [AS alias] в SELECT
  nkAsgn,             # присваивание в UPDATE: col = val
  nkFrom,             # FROM
  nkFromItemPair,     # элемент FROM [AS alias]
  nkJoin,             # JOIN (все виды)
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
  nkColumnList,       # список столбцов в INSERT
  nkValueList,        # VALUES (...)
  nkWhere,            # WHERE
  nkCreateTable,      # CREATE TABLE
  nkCreateTableIfNotExists,   # CREATE TABLE IF NOT EXISTS
  nkCreateType,       # CREATE TYPE
  nkCreateTypeIfNotExists,    # CREATE TYPE IF NOT EXISTS
  nkCreateIndex,      # CREATE INDEX
  nkCreateIndexIfNotExists,   # CREATE INDEX IF NOT EXISTS
  nkEnumDef           # ENUM ('a', 'b', ...)
```

**«Листовые» узлы** (`LiteralNodes`) хранят строковое значение в поле `strVal`:
`nkIdent`, `nkQuotedIdent`, `nkStringLit`, `nkBitStringLit`, `nkHexStringLit`, `nkIntegerLit`, `nkNumericLit`.

Все остальные узлы хранят дочерние узлы в поле `sons: seq[SqlNode]`.

---

### `SqlNode`

Ссылочный тип, представляющий один узел синтаксического дерева.

```nim
type
  SqlNode* = ref SqlNodeObj
  SqlNodeObj* = object
    case kind*: SqlNodeKind
    of LiteralNodes:
      strVal*: string   # значение листового узла
    else:
      sons*: seq[SqlNode]  # дочерние узлы
```

Доступ к полям:

```nim
let node = parseSql("SELECT id FROM users")
# node.kind == nkStmtList
# node[0].kind == nkSelect
# node[0][0].kind == nkSelectColumns
# node[0][0][0].kind == nkSelectPair
# node[0][0][0][0].kind == nkIdent
# node[0][0][0][0].strVal == "id"
```

---

### `SqlParseError`

Исключение, бросаемое при синтаксической ошибке во входном SQL.

```nim
type SqlParseError* = object of ValueError
```

```nim
try:
  let ast = parseSql("SELECT FROM")  # ошибка — нет столбцов
except SqlParseError as e:
  echo "Ошибка разбора: ", e.msg
```

---

### `SqlParser` и `SqlLexer`

Внутренние объекты-парсер и лексер. Обычно используются косвенно через процедуры `parseSql`. `SqlLexer` наследует `BaseLexer` из `std/lexbase` и предоставляет доступ к исходному файлу в целях диагностики.

```nim
type
  SqlLexer* = object of BaseLexer
    # filename: string (внутреннее поле)

  SqlParser* = object of SqlLexer
    # tok: Token (внутреннее)
    # considerTypeParams: bool
```

`considerTypeParams` — если `true`, параметры типов (например, `VARCHAR(255)`) включаются в AST как узлы `nkCall`; если `false` (по умолчанию) — параметры считываются, но в дерево не добавляются.

---

## Создание узлов AST

### `newNode(k: SqlNodeKind): SqlNode`

Создаёт пустой узел заданного вида. Для листовых узлов `strVal = ""`, для остальных `sons = @[]`.

```nim
import std/parsesql

let selectNode = newNode(nkSelect)
let identNode  = newNode(nkIdent)
```

---

### `newNode(k: SqlNodeKind, s: string): SqlNode`

Создаёт листовой узел с заданным строковым значением.

```nim
let col = newNode(nkIdent, "user_id")
echo col.strVal   # user_id
echo col.kind     # nkIdent

let lit = newNode(nkIntegerLit, "42")
echo lit.strVal   # 42
```

---

### `newNode(k: SqlNodeKind, sons: seq[SqlNode]): SqlNode`

Создаёт узел с заданным списком дочерних узлов.

```nim
let children = @[newNode(nkIdent, "a"), newNode(nkIdent, "b")]
let group = newNode(nkPrGroup, children)
echo group.len   # 2
```

---

## Работа с узлами AST

### `len(n: SqlNode): int`

Возвращает количество дочерних узлов. Для листовых узлов (`LiteralNodes`) всегда возвращает `0`.

```nim
let ast = parseSql("SELECT a, b FROM t")
let select = ast[0]              # nkSelect
let columns = select[0]          # nkSelectColumns
echo columns.len                 # 2 (столбца a и b)

let ident = newNode(nkIdent, "x")
echo ident.len                   # 0 (листовой узел)
```

---

### `[](n: SqlNode, i: int): SqlNode`

Возвращает `i`-й дочерний узел (индексация с нуля).

```nim
let ast = parseSql("SELECT id, name FROM users")
let select = ast[0]
let cols   = select[0]    # nkSelectColumns
echo cols[0][0].strVal    # id   (nkSelectPair → nkIdent)
echo cols[1][0].strVal    # name
```

---

### `[](n: SqlNode, i: BackwardsIndex): SqlNode`

Возвращает дочерний узел с конца через оператор `^`.

```nim
let ast = parseSql("SELECT a, b, c FROM t")
let cols = ast[0][0]   # nkSelectColumns
echo cols[^1][0].strVal  # c (последний столбец)
```

---

### `add(father, n: SqlNode)`

Добавляет дочерний узел `n` к узлу `father`. Узел `father` не должен быть листовым.

```nim
var select = newNode(nkSelect)
var cols = newNode(nkSelectColumns)
cols.add(newNode(nkSelectPair, @[newNode(nkIdent, "id")]))
select.add(cols)
echo select.len   # 1
```

---

## Парсинг SQL

### `parseSql(input: string, filename = "", considerTypeParams = false): SqlNode`

Разбирает SQL из строки и возвращает корень AST (`nkStmtList`). При синтаксической ошибке бросает `SqlParseError`. `filename` используется только в сообщениях об ошибках.

```nim
import std/parsesql

# Простой SELECT
let ast = parseSql("SELECT id, name FROM users WHERE id = 1")
echo ast.kind         # nkStmtList
echo ast[0].kind      # nkSelect

# Несколько операторов разделяются точкой с запятой
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
echo ddl[0].kind         # nkCreateTable
echo ddl[0][0].strVal    # users (имя таблицы)

# considerTypeParams = true — VARCHAR(255) попадёт в AST как nkCall
let ddlTyped = parseSql(
  "CREATE TABLE t (name VARCHAR(255))",
  considerTypeParams = true
)
let colType = ddlTyped[0][1][1]  # nkCall
echo colType.kind                 # nkCall
echo colType[0].strVal            # VARCHAR
echo colType[1].strVal            # 255

# Ошибка разбора
try:
  discard parseSql("SELECT FROM WHERE")
except SqlParseError as e:
  echo e.msg   # input(1, 8) Error: expression expected
```

---

### `parseSql(input: Stream, filename: string, considerTypeParams = false): SqlNode`

Разбирает SQL из потока `Stream`. Поток закрывается автоматически после разбора. Удобно для чтения больших файлов.

```nim
import std/[parsesql, streams]

let s = newStringStream("SELECT * FROM orders")
let ast = parseSql(s, "orders.sql")
echo ast[0].kind   # nkSelect

# Из файла
import std/streams
let fs = newFileStream("schema.sql", fmRead)
if fs != nil:
  let schema = parseSql(fs, "schema.sql")
  echo schema.len   # количество операторов
```

---

## Рендеринг обратно в SQL

### `renderSql(n: SqlNode, upperCase = false): string`

Преобразует AST обратно в SQL-строку. При `upperCase = true` ключевые слова записываются в верхнем регистре.

```nim
import std/parsesql

let ast = parseSql("select id, name from users where id = 1")

# Нижний регистр (по умолчанию)
echo renderSql(ast)
# select id,name from users where id = 1

# Верхний регистр
echo renderSql(ast, upperCase = true)
# SELECT id,name FROM users WHERE id = 1

# Roundtrip: парсинг → рендеринг
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

> **Замечание:** `renderSql` не гарантирует посимвольного воспроизведения оригинала — пробелы и переносы строк могут отличаться. Функция предназначена для семантической корректности, не для форматирования.

---

### `$(n: SqlNode): string`

Псевдоним для `renderSql(n)` с `upperCase = false`. Вызывается автоматически при `echo`.

```nim
import std/parsesql

let ast = parseSql("SELECT 1 + 2")
echo ast          # select 1+2
echo $ast[0]      # select 1+2
```

---

## Отладка и визуализация дерева

### `treeRepr(s: SqlNode): string`

Возвращает строковое представление дерева в отступном формате. Полезно для отладки и изучения структуры AST.

```nim
import std/parsesql

let ast = parseSql("SELECT id, name FROM users WHERE age > 18")
echo treeRepr(ast)
```

Пример вывода:
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
# Пример для JOIN
let ast2 = parseSql("""
  SELECT u.name, o.total
  FROM users u
  JOIN orders o ON u.id = o.user_id
""")
echo treeRepr(ast2)
```

---

## Таблица `SqlNodeKind` с пояснениями

| Kind                       | SQL-конструкция             | `sons` / `strVal`                                      |
|----------------------------|-----------------------------|--------------------------------------------------------|
| `nkStmtList`               | (корень)                    | `sons`: все операторы                                  |
| `nkSelect`                 | `SELECT ...`                | `sons[0]`=columns, `sons[1..]`=from/where/etc.         |
| `nkSelectDistinct`         | `SELECT DISTINCT ...`       | то же, что nkSelect                                    |
| `nkSelectColumns`          | список столбцов             | `sons`: nkSelectPair или nkIdent(`*`)                  |
| `nkSelectPair`             | `expr [AS alias]`           | `sons[0]`=expr, `sons[1]`=alias (опционально)          |
| `nkFrom`                   | `FROM ...`                  | `sons`: nkFromItemPair                                 |
| `nkFromItemPair`           | `table [AS alias]`          | `sons[0]`=table, `sons[1]`=alias (опционально)         |
| `nkWhere`                  | `WHERE expr`                | `sons[0]`=expr                                         |
| `nkGroup`                  | `GROUP BY ...`              | `sons`: выражения                                      |
| `nkOrder`                  | `ORDER BY ...`              | `sons`: выражения / nkDesc                             |
| `nkDesc`                   | `expr DESC`                 | `sons[0]`=expr                                         |
| `nkHaving`                 | `HAVING ...`                | `sons`: выражения                                      |
| `nkLimit`                  | `LIMIT n`                   | `sons[0]`=expr                                         |
| `nkOffset`                 | `OFFSET n`                  | `sons[0]`=expr                                         |
| `nkJoin`                   | `[type] JOIN ... ON/USING`  | `sons[0]`=type, `sons[1]`=left, `sons[2]`=right, `sons[3]`=cond |
| `nkNaturalJoin`            | `NATURAL JOIN`              | то же, что nkJoin                                      |
| `nkUsing`                  | `USING (cols)`              | `sons`: nkIdent                                        |
| `nkUnion`                  | `UNION`                     | нет дочерних                                           |
| `nkIntersect`              | `INTERSECT`                 | нет дочерних                                           |
| `nkExcept`                 | `EXCEPT`                    | нет дочерних                                           |
| `nkInsert`                 | `INSERT INTO`               | `sons[0]`=table, `sons[1]`=cols, `sons[2]`=values     |
| `nkUpdate`                 | `UPDATE ... SET`            | `sons[0]`=table, `sons[1..n-2]`=nkAsgn, `sons[n-1]`=where |
| `nkDelete`                 | `DELETE FROM`               | `sons[0]`=table, `sons[1]`=where                       |
| `nkAsgn`                   | `col = val`                 | `sons[0]`=col, `sons[1]`=val                           |
| `nkColumnList`             | `(col1, col2, ...)`         | `sons`: nkIdent                                        |
| `nkValueList`              | `VALUES (...)`              | `sons`: выражения                                      |
| `nkCreateTable`            | `CREATE TABLE`              | `sons[0]`=name, `sons[1..]`=column defs               |
| `nkCreateTableIfNotExists` | `CREATE TABLE IF NOT EXISTS`| то же                                                  |
| `nkColumnDef`              | `name type constraints`     | `sons[0]`=name, `sons[1]`=type, `sons[2..]`=constraints |
| `nkPrimaryKey`             | `PRIMARY KEY`               | `sons`: (опционально) столбцы                          |
| `nkForeignKey`             | `FOREIGN KEY`               | `sons`: столбцы + nkReferences                         |
| `nkReferences`             | `REFERENCES table(col)`     | `sons[0]`=column reference                             |
| `nkNotNull`                | `NOT NULL`                  | нет дочерних                                           |
| `nkNull`                   | `NULL`                      | нет дочерних                                           |
| `nkUnique`                 | `UNIQUE`                    | `sons`: (опционально) столбцы                          |
| `nkDefault`                | `DEFAULT expr`              | `sons[0]`=expr                                         |
| `nkCheck`                  | `CHECK (expr)`              | `sons[0]`=expr                                         |
| `nkConstraint`             | `CONSTRAINT name CHECK`     | `sons[0]`=name, `sons[1]`=expr                         |
| `nkIdentity`               | `IDENTITY`                  | нет дочерних                                           |
| `nkCreateIndex`            | `CREATE INDEX`              | `sons[0]`=index name, `sons[1]`=table, `sons[2..]`=cols |
| `nkCreateType`             | `CREATE TYPE`               | `sons[0]`=name, `sons[1]`=type def                     |
| `nkEnumDef`                | `ENUM ('a', 'b')`           | `sons`: nkStringLit                                    |
| `nkInfix`                  | `a OP b`                    | `sons[0]`=op, `sons[1]`=left, `sons[2]`=right          |
| `nkPrefix`                 | `OP a` (unary)              | `sons[0]`=op, `sons[1]`=operand                        |
| `nkCall`                   | `func(args)`                | `sons[0]`=name, `sons[1..]`=args                       |
| `nkDot`                    | `table.column`              | `sons[0]`=table, `sons[1]`=column                      |
| `nkDotDot`                 | `schema..table`             | `sons[0]`=schema, `sons[1]`=table                      |
| `nkPrGroup`                | `(expr, ...)`               | `sons`: выражения                                      |
| `nkIdent`                  | идентификатор               | `strVal`=имя                                           |
| `nkQuotedIdent`            | `"идентификатор"`           | `strVal`=имя без кавычек                               |
| `nkStringLit`              | `'строка'`                  | `strVal`=содержимое                                    |
| `nkIntegerLit`             | `42`                        | `strVal`="42"                                          |
| `nkNumericLit`             | `3.14`                      | `strVal`="3.14"                                        |
| `nkBitStringLit`           | `B'0101'`                   | `strVal`="0101"                                        |
| `nkHexStringLit`           | `x'FF'`                     | `strVal`="FF"                                          |

---

## Практические примеры

### Обход AST: извлечь все таблицы из SELECT

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

### Извлечь имена столбцов из CREATE TABLE

```nim
import std/parsesql

proc getColumns(ddl: string): seq[tuple[name, typ: string]] =
  let ast = parseSql(ddl)
  let createNode = ast[0]  # nkCreateTable
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

### Построение AST вручную и рендеринг

```nim
import std/parsesql

# Строим: SELECT id, name FROM users WHERE id = 1

var stmt = newNode(nkSelect)

# Столбцы
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

### Roundtrip: нормализация SQL

```nim
import std/parsesql

proc normalizeSql(sql: string): string =
  ## Нормализует SQL: убирает лишние пробелы, приводит ключевые слова к верхнему регистру.
  try:
    let ast = parseSql(sql)
    return renderSql(ast, upperCase = true)
  except SqlParseError as e:
    return "ОШИБКА: " & e.msg

echo normalizeSql("  select   id,name   from   users  ")
# SELECT id,name FROM users

echo normalizeSql("insert into t (a,b) values (1,'hello')")
# INSERT INTO t(a,b) VALUES(1,'hello')

echo normalizeSql("create table x (id integer primary key)")
# CREATE TABLE x(id INTEGER PRIMARY KEY);
```

---

### Проверка типа оператора на верхнем уровне

```nim
import std/parsesql

proc classifySql(sql: string): string =
  try:
    let ast = parseSql(sql)
    if ast.len == 0: return "пустой"
    case ast[0].kind
    of nkSelect, nkSelectDistinct: "SELECT"
    of nkInsert:                   "INSERT"
    of nkUpdate:                   "UPDATE"
    of nkDelete:                   "DELETE"
    of nkCreateTable, nkCreateTableIfNotExists: "CREATE TABLE"
    of nkCreateIndex, nkCreateIndexIfNotExists: "CREATE INDEX"
    of nkCreateType,  nkCreateTypeIfNotExists:  "CREATE TYPE"
    else: "ДРУГОЙ (" & $ast[0].kind & ")"
  except SqlParseError:
    "ОШИБКА СИНТАКСИСА"

echo classifySql("SELECT 1")                        # SELECT
echo classifySql("INSERT INTO t VALUES (1)")         # INSERT
echo classifySql("CREATE TABLE t (id INT)")          # CREATE TABLE
echo classifySql("DELETE FROM t WHERE id = 5")       # DELETE
```

---

### Поддерживаемые SQL-конструкции

Модуль поддерживает следующие операторы и синтаксис:

**DML (Data Manipulation Language):**
- `SELECT [DISTINCT] ... FROM ... [JOIN ...] [WHERE ...] [GROUP BY ...] [HAVING ...] [ORDER BY ...] [LIMIT ...] [OFFSET ...]`
- `INSERT INTO table [(cols)] VALUES (...) | DEFAULT VALUES`
- `UPDATE table SET col = val [, ...] [WHERE ...]`
- `DELETE FROM table [WHERE ...]`

**DDL (Data Definition Language):**
- `CREATE TABLE [IF NOT EXISTS] name (...)`
- `CREATE INDEX [IF NOT EXISTS] name ON table (...)`
- `CREATE TYPE [IF NOT EXISTS] name AS ...`

**Типы данных:**
- Любые идентификаторы (`INTEGER`, `VARCHAR`, `TEXT`, `NUMERIC`, и т.д.)
- Параметризованные типы: `VARCHAR(255)`, `DECIMAL(10, 2)` — обрабатываются с учётом флага `considerTypeParams`
- `ENUM ('val1', 'val2', ...)`

**Ограничения столбцов:**
- `PRIMARY KEY`, `FOREIGN KEY ... REFERENCES`
- `NOT NULL`, `NULL`, `UNIQUE`
- `DEFAULT expr`
- `CHECK (expr)`
- `CONSTRAINT name CHECK (expr)`
- `IDENTITY`

**JOIN-конструкции:**
- `INNER JOIN`, `LEFT [OUTER] JOIN`, `RIGHT [OUTER] JOIN`, `FULL [OUTER] JOIN`
- `CROSS JOIN`, `NATURAL JOIN`
- `ON expr`, `USING (cols)`

**Множественные операции:**
- `UNION`, `INTERSECT`, `EXCEPT`
