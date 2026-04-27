# 📘 Documentation for the `db_mysql` Package (Nim)

## 🔍 Introduction
`db_mysql` is a high-level wrapper over the native `libmysqlclient` library for the MySQL RDBMS in the Nim programming language. The module provides a unified API for working with relational data, supports parameterized queries via the `?` marker, safe and fast iterators, and automatic casting of all values to string representation.

> **Note:** This module does not implement ORM mechanisms. To maintain maximum proximity to the MySQL C-API and simplify serialization, all rows are returned as `seq[string]`, and `NULL` values are converted to empty strings `""`. Unlike `db_postgres` and `db_sqlite`, this module does not support server-side prepared statements (`SqlPrepared`), relying instead on client-side parameterization.

## 📦 Installation
To use the module, install the `db_connector` package:
```bash
nimble install db_connector
```

## 🧱 Core Data Types

| Type | Description | Purpose |
|------|-------------|---------|
| `DbConn` | `distinct PMySQL` | Handle for an active connection to a MySQL server. |
| `Row` | `seq[string]` | Represents a single query result row. All values are cast to strings. |
| `InstantRow` | `object { row: cstringArray, len: int }` | Lightweight handle containing a pointer to the result string array. Allows on-demand column extraction without memory allocations. |
| `DbType` | Structure | Contains column type metadata: `kind` (category), `size`, `name` (type name), `notNull`, `primaryKey`, `tableName`, etc. |
| `DbColumns` | `seq[DbType]` | Collection of metadata for all columns returned by a query. |

## ⚙️ Basic API

### 🔌 Connection Management
- `open(connection, user, password, database): DbConn` — opens a connection. Supports specifying the port via `host:port` syntax in the `connection` parameter.
- `close(db: DbConn)` — safely closes the connection and frees `libmysqlclient` resources.
- `setEncoding(connection: DbConn, encoding: string): bool` — sets the client connection encoding.

### 📝 Query Execution
- `exec(db, query, args)` — executes a query. Throws `DbError` on failure.
- `tryExec(db, query, args): bool` — executes a query, returns `true` on success, `false` on error.
- `execAffectedRows(db, query, args): int64` — executes a query and returns the number of modified rows (for `UPDATE`/`DELETE`).

### 🔍 Data Fetching
- `fastRows(db, query, args): iterator Row` — fast iterator. **Warning:** Breaking the loop (`break`) will cause a `Commands out of sync` error on the next query.
- `rows(db, query, args): iterator Row` — safe iterator (loads all rows into memory before iteration, allows `break`).
- `instantRows(db, query, args): iterator InstantRow` — returns a row handle for lazy access.
- `instantRows(db, columns, query, args): iterator InstantRow` — additionally populates `columns` with metadata about column types.
- `getRow(db, query, args): Row` — returns the first row of the result.
- `getAllRows(db, query, args): seq[Row]` — returns all rows as a sequence.
- `getValue(db, query, args): string` — returns the value of the first column of the first row.

### 📊 Insertion & ID Retrieval
- `tryInsertId(db, query, args): int64` — executes an `INSERT` and returns the `AUTO_INCREMENT` ID via `LAST_INSERT_ID()`. Returns `-1` on error.
- `insertId(db, query, args): int64` — same as above, but throws `DbError` on failure.
- `tryInsert(db, query, pkName, args): int64` — alias for `tryInsertId`. The `pkName` parameter is ignored but kept for compatibility with the common `db_common` API.
- `insert(db, query, pkName, args): int64` — alias for `insertId`.

### 🛡 Utilities
- `dbQuote(s: string): string` — escapes special characters for MySQL (`'`, `"`, `\`, `\0`, `\b`, `\t`, `\n`, `\r`, `\x1a`).
- `dbError(db: DbConn)` — explicitly throws an exception with the error message from `mysql.error()`.

---

## 💻 Usage Examples

### 1. Connection, Table Creation, and Data Insertion
```nim
import
  db_connector/db_mysql
  std/math
  std/assertions

let
  # Open connection to the local MySQL server.
  # Parameters: host[:port], user, password, database name.
  db = open("localhost", "root", "secret", "store_db")

# Drop the table if it exists to ensure a clean state
exec(db, sql"DROP TABLE IF EXISTS products")

# Create a table with a typical e-commerce structure
exec(db, sql"""
  CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    price DECIMAL(10, 2),
    stock_count INT DEFAULT 0
  )
""")

# Insert a single record with automatic parameter substitution via ?
exec(db, sql"INSERT INTO products (title, price, stock_count) VALUES (?, ?, ?)",
     "Widget Alpha", 19.99, 100)

# Batch insertion inside a transaction for improved performance
exec(db, sql"START TRANSACTION")
for i in 1 .. 5:
  let
    prodName = "Item_" & $i
    prodPrice = sqrt(float(i)) * 10.0
    prodStock = i * 20
  exec(db, sql"INSERT INTO products (title, price, stock_count) VALUES (?, ?, ?)",
       prodName, prodPrice, prodStock)
exec(db, sql"COMMIT")

# Retrieve the ID of the last inserted row.
# In MySQL, the module uses the system function LAST_INSERT_ID(),
# so automatic RETURNING clause injection is not required.
let
  lastId = tryInsertId(db, sql"INSERT INTO products (title, price, stock_count) VALUES (?, ?, ?)",
                       "Widget Beta", 29.50, 45)

# Verify that the insertion succeeded
assert(lastId > 0, "Insertion error: invalid ID returned")

# Close the connection and free libmysqlclient resources
close(db)
```

### 2. Data Fetching, `InstantRow`, and Column Metadata
```nim
import
  db_connector/db_mysql
  std/strutils
  std/assertions

let
  db = open("localhost", "root", "secret", "store_db")

echo("--- Safe iterator (rows) ---")
# 'rows' loads the entire result into Nim memory, which is safe for loops with 'break'
for row in rows(db, sql"SELECT id, title, price FROM products WHERE price > ?", 20.0):
  let
    uid = parseInt(row[0])
    name = row[1]
    cost = parseFloat(row[2])
  echo("ID: " & $uid & " | Product: " & name & " | Price: " & $cost)

echo("\n--- Lazy iterator (instantRows) ---")
# 'instantRows' does not allocate strings, pointing directly to the MySQL C API buffer
for row in instantRows(db, sql"SELECT title, price FROM products"):
  let
    title = row[0]
    priceStr = row[1]
  # Skip items without a price or with a zero price
  if len(priceStr) > 0 and parseFloat(priceStr) > 0.0:
    echo(title & " is available for: " & priceStr)

echo("\n--- Retrieving column metadata ---")
var
  # Structure to store information about field types and properties
  cols: DbColumns
# The iterator initializes 'cols' on the first call. We discard the rows.
for _ in instantRows(db, cols, sql"SELECT * FROM products LIMIT 0"):
  discard

for col in cols:
  echo("Column: " & col.name & " | MySQL Type: " & col.typ.name &
       " | Table: " & col.tableName & " | Primary Key: " & $col.primaryKey)

echo("\n--- Retrieving a single value ---")
let
  firstProduct = getValue(db, sql"SELECT title FROM products ORDER BY id LIMIT 1")
echo("First product in catalog: " & firstProduct)

# Always close the connection when finished
close(db)
```

### 3. Change Counting, Manual Escaping, and Error Handling
```nim
import
  db_connector/db_mysql
  std/assertions

let
  db = open("localhost", "root", "secret", "store_db")

echo("--- Counting rows affected by a query ---")
let
  affectedCount = execAffectedRows(db, sql"UPDATE products SET stock_count = stock_count - 1 WHERE id > ?", 0)
echo("Items decremented: " & $affectedCount)

echo("\n--- Safe manual escaping ---")
let
  userInput = "O'Reilly & Sons \"Special\" Ltd."
  # dbQuote replaces single/double quotes and control characters
  # according to MySQL escaping standards
  safeInput = dbQuote(userInput)
# Dynamic query assembly (use only when ? is not suitable)
exec(db, sql"INSERT INTO products (title, price, stock_count) VALUES (" & safeInput & ", 0.0, 10)")

echo("\n--- Error handling ---")
try:
  # Attempt to query a non-existent table
  exec(db, sql"SELECT * FROM non_existent_table")
except DbError:
  let
    errMsg = getCurrentExceptionMsg()
  echo("Caught expected exception: " & errMsg)

# Close the connection
close(db)
```

---

## 📖 Detailed Architecture Notes

### Why `seq[string]` for Rows?
The source code explicitly implements `Row` as `seq[string]`. This design choice serves three purposes:
1. **Nativity:** Aligns closest with the native MySQL C-API (`MYSQL_ROW` is an array of `char*`), minimizing client-side type conversion overhead.
2. **Abstraction:** MySQL supports dozens of specific types (`DECIMAL`, `DATETIME`, `JSON`, `ENUM`, `SET`, `BLOB`, etc.). Casting everything to strings hides this complexity from the user and simplifies driver logic.
3. **Forwarding Convenience:** If data simply needs to be logged, passed to another query, or serialized to JSON/XML, the string representation is a universal format.

### How `InstantRow` Works
`InstantRow` does not copy data into Nim's memory on each iteration. Instead, it stores a `row: cstringArray` pointer to the internal buffer returned by `mysql_fetch_row()`. The `row[col]` operation directly calls `$row.row[col]`. This saves memory and speeds up reading massive datasets, but requires data access to occur **strictly inside the loop body**. After the iterator finishes, the buffer is freed by `mysql_free_result()`, and accessing `row` afterward will lead to undefined behavior or a segmentation fault.

### Parameterization & Security (`?`)
`db_mysql` does not support server-side prepared statements. All parameters are substituted client-side:
1. The query is formatted via `dbFormat`.
2. Each argument is converted to a string (`$`), then escaped via `dbQuote`.
3. The final SQL query is sent to `mysql_real_query`.
This completely eliminates SQL injection risks when using `?`, but requires memory allocation for each query string. For mass operations in high-load systems, manually using `START TRANSACTION` / `COMMIT` is recommended, as shown in the examples.

### `AUTO_INCREMENT` ID Retrieval
Unlike PostgreSQL (`RETURNING id`) or SQLite (`last_insert_rowid()`), MySQL provides the system function `LAST_INSERT_ID()`. The module automatically calls it after a successful `INSERT`. Since the value is tied to the connection session, it is safe even under high concurrency. The `pkName` parameter in `tryInsert` / `insert` functions is preserved solely for compatibility with the generalized `db_common` interface and is ignored in this module.

---

## ⚠️ Important Notes

1. **Danger of breaking `fastRows`:** Never use `break` inside `fastRows`. The MySQL client library leaves the query result in an "unfetched" state. The next query call will fail with `Commands out of sync`. Use `rows` for safe interruption.
2. **MySQL-specific `dbQuote`:** The function correctly escapes not only single (`'`) but also double quotes (`"`), backslashes (`\`), and control characters (`\0`, `\b`, `\t`, `\n`, `\r`, `\x1a`). This is crucial when working with binary data accidentally placed in text fields or during migrations from other RDBMS.
3. **No `SqlPrepared` support:** This module does not implement prepared statements. If server-side query parsing performance is critical, consider using the direct C-API via `mysql_stmt_*` or switching to `db_postgres`/`db_sqlite`.
4. **Encoding:** `setEncoding` calls `mysql_set_character_set`. The change applies only to the current connection and affects how the client interprets incoming/outgoing data. Existing table data is not automatically converted.
5. **Errors:** All procedures with the `try` prefix return success/failure status or `-1` without throwing exceptions. Other procedures call `dbError(db)` on failure, which throws `ref DbError` with the message from `mysql.error()`.
6. **Type Compatibility:** When substituting parameters via `?`, all values are automatically converted to strings. If you pass `nil` or an empty string, ensure the target column allows `NULL`, otherwise MySQL will return a `Column cannot be null` error.

## 🏁 Conclusion
The `db_mysql` package provides a reliable and easy-to-use interface to MySQL. Its strict separation of fast/safe iterators, automatic column metadata handling (`DbType`), built-in escaping according to MySQL standards, and transparent `AUTO_INCREMENT` processing make it suitable for both administration scripts and high-load web applications in Nim.
