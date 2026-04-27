
# 📘 Documentation for the `db_sqlite` Package

## 🔍 Introduction
`db_sqlite` is a high-level wrapper over the SQLite library for the Nim programming language. The module provides a unified API for working with relational data, supports parameter substitution, prepared statements, binary data (`BLOB`) handling, and efficient row iteration.

> **Note:** This module does not implement ORM mechanisms. To maintain simplicity and stay as close as possible to the native SQLite API, all rows are returned as `seq[string]`, and `NULL` values are converted to empty strings `""`.

## 📦 Installation
To use the module, you need to install the `db_connector` package:
```bash
nimble install db_connector
```

## 🧱 Core Data Types

| Type | Description | Purpose |
|------|-------------|---------|
| `DbConn` | `PSqlite3` | Handle for an active database connection. |
| `Row` | `seq[string]` | Represents a single result row. All values are cast to strings. |
| `InstantRow` | `PStmt` | Lightweight handle allowing on-demand extraction of column values inside an iterator. |
| `SqlPrepared` | `distinct PStmt` | Identifier for a precompiled SQL query. |
| `DbType` | Structure | Contains column type metadata: `kind` (type), `size`, `name`, `primaryKey`, etc. |
| `DbColumns` | `seq[DbType]` | Collection of metadata for all columns returned by a query. |

## ⚙️ Basic API

### 🔌 Connection Management
- `open(connection, user, password, database): DbConn` — opens a connection. For SQLite, only `connection` (file path) and `database` (usually empty) are used.
- `close(db: DbConn)` — safely closes the connection and frees resources.

### 📝 Query Execution
- `exec(db, query, args)` — executes a query. Throws a `DbError` exception on failure.
- `tryExec(db, query, args): bool` — executes a query, returns `true` on success, `false` on failure.
- `execAffectedRows(db, query, args): int64` — executes a query and returns the number of modified rows (for `UPDATE`/`DELETE`).

### 🔍 Data Fetching
- `fastRows(db, query, args): iterator Row` — fast iterator. **Warning:** Breaking the loop (`break`) may block subsequent queries until the connection is closed.
- `rows(db, query, args): iterator Row` — safe iterator (slower, but allows `break`).
- `instantRows(db, query, args): iterator InstantRow` — returns a row handle for lazy column access.
- `getRow(db, query, args): Row` — returns the first row of the result. If empty, returns a `Row` of empty strings.
- `getAllRows(db, query, args): seq[Row]` — returns all result rows as a sequence.
- `getValue(db, query, args): string` — returns the value of the first column of the first row.

### 🛠 Prepared Statements
- `prepare(db, query: string): SqlPrepared` — compiles an SQL query for repeated execution.
- `finalize(stmt: SqlPrepared)` — frees resources of a prepared statement.
- `bindParam(ps, paramIdx, val)` — binds a value to a specific parameter index (`?`).
- `bindNull(ps, paramIdx)` — binds a `NULL` value.
- `bindParams(ps, args...)` — macro for bulk parameter binding. Automatically infers types.

### 📊 Insertion & ID Retrieval
- `tryInsertID(db, query, args): int64` — executes an `INSERT` and returns the `rowid`. Returns `-1` on error.
- `insertID(db, query, args): int64` — same as above, but throws `DbError` on failure.

### 🛡 Utilities
- `dbQuote(s: string): string` — escapes single quotes for safe insertion into `VARCHAR` fields.
- `dbError(db: DbConn)` — explicitly throws an exception with the SQLite error message.
- `setEncoding(db, encoding: string): bool` — sets the database encoding (only works before the first table is created).

---

## 💻 Usage Examples

### 1. Connection, Table Creation, and Data Insertion
```nim
import
  db_connector/db_sqlite
  std/math
  std/assertions

let
  # Open connection. For SQLite, only the filename is required.
  db = open("app_data.db", "", "", "")

# Drop table if it exists
exec(db, sql"DROP TABLE IF EXISTS users")

# Create table
exec(db, sql"""
  CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    age INT,
    balance DECIMAL(10, 2)
  )
""")

# Insert single record with automatic parameter substitution
exec(db, sql"INSERT INTO users (username, age, balance) VALUES (?, ?, ?)",
     "alice", 30, 150.50)

# Batch insertion within a transaction
exec(db, sql"BEGIN")
for i in 1 .. 10:
  let
    name = "user_" & $i
    age = i * 5
    bal = sqrt(float(i))
  exec(db, sql"INSERT INTO users (username, age, balance) VALUES (?, ?, ?)",
       name, age, bal)
exec(db, sql"COMMIT")

# Get the ID of the last inserted row
let
  lastId = tryInsertID(db, sql"INSERT INTO users (username, age, balance) VALUES (?, ?, ?)",
                       "bob", 25, 0.0)

# Verify result
assert(lastId > 0, "Insertion failed: negative ID returned")

# Close connection
close(db)
```

### 2. Data Fetching and `InstantRow` Usage
```nim
import
  db_connector/db_sqlite
  std/strutils
  std/assertions

let
  db = open("app_data.db", "", "", "")

echo("--- Standard iterator (rows) ---")
for row in rows(db, sql"SELECT id, username, age FROM users WHERE age > ?", 10):
  # Access by index, type conversion via standard functions
  let
    uid = parseInt(row[0])
    uname = row[1]
    uage = parseInt(row[2])
  echo("ID: " & $uid & " | Name: " & uname & " | Age: " & $uage)

echo("\n--- Lazy iterator (instantRows) ---")
for row in instantRows(db, sql"SELECT username, balance FROM users"):
  # row[col] returns the string directly from SQLite memory
  let
    name = row[0]
    money = row[1]
  # Skip rows with zero balance
  if parseFloat(money) > 0.0:
    echo(name & " has balance: " & money)

echo("\n--- Retrieve a single value ---")
let
  firstUser = getValue(db, sql"SELECT username FROM users ORDER BY id LIMIT 1")
echo("First user: " & firstUser)

echo("\n--- Retrieve column metadata ---")
var
  cols: DbColumns
for _ in instantRows(db, cols, sql"SELECT * FROM users LIMIT 0"):
  discard # Iterator body not required, we only need cols initialization

for col in cols:
  echo("Column: " & col.name & " | Type: " & col.typ.name &
       " | Primary Key: " & $col.primaryKey)

close(db)
```

### 3. Binary Data (`BLOB`) and Prepared Statements
```nim
import
  db_connector/db_sqlite
  std/random
  std/assertions

let
  db = open("blob_test.db", "", "", "")

# Create table with a BLOB field
exec(db, sql"DROP TABLE IF EXISTS media")
exec(db, sql"CREATE TABLE media (id INTEGER PRIMARY KEY, data BLOB)")

# Generate test binary data (byte array)
var
  originalData = newSeq[byte](128)
randomize()
for i in 0 .. len(originalData) - 1:
  originalData[i] = byte(rand(255))

# Prepare insert statement
let
  insertStmt = prepare(db, "INSERT INTO media (id, data) VALUES (?, ?)")

# Bind parameters manually
bindParam(insertStmt, 1, 1'i32)
bindParam(insertStmt, 2, originalData, true) # true = SQLite copies the data

# Execute
exec(db, insertStmt)
finalize(insertStmt)

# Prepare select statement
let
  selectStmt = prepare(db, "SELECT data FROM media WHERE id = ?")
bindParam(selectStmt, 1, 1'i32)

# Extract BLOB
var
  retrievedData = getValue(db, selectStmt)

# Verify data identity
assert(len(retrievedData) == len(originalData), "Data size mismatch")
for i in 0 .. len(originalData) - 1:
  assert(retrievedData[i] == originalData[i], "Data corrupted at index " & $i)

echo("Binary data successfully saved and read.")

finalize(selectStmt)
close(db)
```

---

## 📖 Detailed Architecture Notes

### Why `seq[string]` for Rows?
The source code explicitly defines `Row` as `seq[string]`. This design choice is made for three reasons:
1. **Nativity:** It is closest to the native SQLite API (`char**`), minimizing type conversion overhead.
2. **Abstraction:** Databases can return dozens of types (`INTEGER`, `REAL`, `BLOB`, `TEXT`, custom extensions). Casting everything to strings hides this complexity from the user.
3. **Forwarding Convenience:** If data simply needs to be logged, passed to another query, or serialized to JSON, the string representation is a universal format.

### How `InstantRow` Works
`InstantRow` does not copy data into Nim's memory on each iteration. Instead, it holds a pointer to the internal `sqlite3_stmt` buffer. The `row[col]` operation calls `column_text(stmt, col)` directly. This saves memory and speeds up reading large datasets, but requires data access to happen **strictly inside the loop body**. After the iterator finishes, the buffer is freed, and accessing `row` afterward leads to undefined behavior.

### Prepared Statements (`SqlPrepared`)
Used for repeatedly executing the same query with different parameters.
```nim
let
  stmt = prepare(db, "UPDATE items SET price = ? WHERE id = ?")
```
Each call to `exec(db, stmt, ...)` or `bindParam(stmt, idx, val)` resets the state (`reset`) and clears bindings (`clear_bindings`), ensuring operation isolation. You **must** call `finalize(stmt)` after use, otherwise a memory leak will occur.

### Security & Escaping
The module supports safe parameter substitution via `?`:
```nim
# ✅ Safe
exec(db, sql"SELECT * FROM users WHERE name = ?", "O'Connor")

# ❌ Dangerous (SQL injection risk)
exec(db, sql"SELECT * FROM users WHERE name = '" & userInput & "'")
```
If you must manually insert a string into a query, use `dbQuote(str)`:
```nim
let
  safeName = dbQuote(userInput)
exec(db, sql"SELECT * FROM users WHERE name = " & safeName)
```

---

## ⚠️ Important Notes

1. **Breaking `fastRows`:** Never use `break` inside `fastRows`. SQLite leaves the read transaction open, and the next query will fail with `unable to close due to ...`. Use `rows` for safe interruption.
2. **Encoding:** `setEncoding` must be called **before** creating the first table. After database initialization, the SQLite engine silently ignores encoding changes.
3. **Parameter Types:** When using `bindParam` or `bindParams`, the macro automatically selects the overloaded function (`int32`, `int64`, `float64`, `string`, `openArray[byte]`). For `NULL`, use the macro `bindParams(stmt, nil)` or explicitly call `bindNull(stmt, idx)`.
4. **Errors:** All procedures with the `try` prefix return success/failure status without throwing exceptions. Other procedures call `dbError(db)` on failure, which throws `ref DbError` with the message from `sqlite3.errmsg(db)`.

## 🏁 Conclusion
The `db_sqlite` package provides a balance between performance and ease of use. Its strict separation of fast/safe iterators, binary data support, and prepared statements make it suitable for both utility scripts and high-load server applications in Nim.



