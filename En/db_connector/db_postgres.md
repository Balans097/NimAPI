
# 📘 Documentation for the `db_postgres` Package (Nim)

## 🔍 Introduction
`db_postgres` is a high-level wrapper over the native `libpq` library for the PostgreSQL RDBMS in the Nim programming language. The module provides a unified API for working with relational data, supports parameterized queries (`?` for standard queries and `$1, $2...` for prepared statements), safe and fast iterators, and automatic type casting to string representation.

> **Note:** This module does not implement ORM mechanisms. To maintain maximum proximity to the `libpq` API and simplify serialization, all rows are returned as `seq[string]`, and `NULL` values are converted to empty strings `""`.

## 📦 Installation
To use the module, install the `db_connector` package:
```bash
nimble install db_connector
```

## 🧱 Core Data Types

| Type | Description | Purpose |
|------|-------------|---------|
| `DbConn` | `PPGconn` | Handle for an active connection to a PostgreSQL server. |
| `Row` | `seq[string]` | Represents a single query result row. All values are cast to strings. |
| `InstantRow` | `object { res: PPGresult }` | Lightweight handle containing a pointer to the internal result buffer. Allows on-demand column extraction without memory allocations. |
| `SqlPrepared` | `distinct string` | String identifier for a server-prepared SQL query. |
| `DbType` | Structure | Contains column type metadata: `kind` (type category), `size`, `name` (type name in DB), `tableName`, etc. |
| `DbColumns` | `seq[DbType]` | Collection of metadata for all columns returned by a query. |

## ⚙️ Basic API

### 🔌 Connection Management
- `open(connection, user, password, database): DbConn` — opens a connection. Supports standard connection strings (key=value) in the `database` parameter.
- `close(db: DbConn)` — safely closes the connection and frees `libpq` resources.
- `setEncoding(db: DbConn, encoding: string): bool` — sets the client connection encoding.

### 📝 Query Execution
- `exec(db, query, args)` — executes a query. Throws `DbError` on failure.
- `tryExec(db, query, args): bool` — executes a query, returns `true` on success.
- `execAffectedRows(db, query, args): int64` — executes a query and returns the number of modified rows (for `UPDATE`/`DELETE`/`INSERT ... RETURNING`).

### 🔍 Data Fetching
- `fastRows(db, query, args): iterator Row` — fast iterator. **Unlike SQLite, it is safe for PostgreSQL** and does not block the connection on interruption.
- `rows(db, query, args): iterator Row` — safe iterator (allocates all rows to memory before iteration).
- `instantRows(db, query, args): iterator InstantRow` — returns a row handle for lazy access.
- `instantRows(db, columns, query, args): iterator InstantRow` — additionally populates `columns` with metadata about column types.
- `getRow(db, query, args): Row` — returns the first row of the result.
- `getAllRows(db, query, args): seq[Row]` — returns all rows as a sequence.
- `getValue(db, query, args): string` — returns the value of the first column of the first row.

### 🛠 Prepared Statements
- `prepare(db, stmtName, query, nParams): SqlPrepared` — compiles the query on the server. Parameters are specified as `$1, $2...`.
- `exec(db, stmtName, args)` / `tryExec(db, stmtName, args): bool` — executes a prepared query.

### 📊 Insertion & ID Retrieval
- `tryInsertID(db, query, args): int64` — executes an `INSERT` and returns the `rowid`. Automatically appends `RETURNING id` to the query. Only works if the primary key is named `id`.
- `insertID(db, query, args): int64` — same as above, but throws `DbError` on failure.
- `tryInsert(db, query, pkName, args): int64` — allows specifying an arbitrary primary key name (`pkName`).
- `insert(db, query, pkName, args): int64` — same as above with exception throwing.

### 🛡 Utilities
- `dbQuote(s: string): string` — escapes single quotes (`'` → `''`) and null bytes (`\0` → `\\0`).
- `dbError(db: DbConn)` — explicitly throws an exception with the error message from `pqErrorMessage`.

---

## 💻 Usage Examples

### 1. Connection, Table Creation, and Data Insertion
```nim
import
  db_connector/db_postgres
  std/math
  std/assertions

let
  # Open connection to the local PostgreSQL server.
  # Parameters: host, user, password, database name.
  db = open("localhost", "postgres", "secret", "company_db")

# Drop table if it exists to ensure a clean state.
exec(db, sql"DROP TABLE IF EXISTS employees")

# Create a table with a typical enterprise structure.
exec(db, sql"""
  CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    department VARCHAR(50),
    salary NUMERIC(10, 2)
  )
""")

# Insert a single record using automatic parameter substitution (?).
exec(db, sql"INSERT INTO employees (full_name, department, salary) VALUES (?, ?, ?)",
     "Alice Smith", "Engineering", 95000.00)

# Batch insertion wrapped inside a transaction for performance and atomicity.
exec(db, sql"BEGIN")
for i in 1 .. 5:
  let
    empName = "Employee_" & $i
    dept = "Support"
    pay = 50000.0 + (float(i) * 2000.0)
  exec(db, sql"INSERT INTO employees (full_name, department, salary) VALUES (?, ?, ?)",
       empName, dept, pay)
exec(db, sql"COMMIT")

# Retrieve the auto-generated ID of the last inserted row.
# The module automatically appends "RETURNING id" to the query.
let
  lastId = tryInsertID(db, sql"INSERT INTO employees (full_name, department, salary) VALUES (?, ?, ?)",
                       "Bob Jones", "HR", 65000.00)

# Verify that the insertion succeeded (ID should be positive).
assert(lastId > 0, "Insertion error: negative or zero ID returned")

# Safely close the database connection and release libpq resources.
close(db)
```

### 2. Data Fetching, `InstantRow`, and Column Metadata
```nim
import
  db_connector/db_postgres
  std/strutils
  std/assertions

let
  db = open("localhost", "postgres", "secret", "company_db")

echo("--- Standard safe iterator (rows) ---")
# The 'rows' iterator loads all data into memory first, allowing safe loop breaks.
for row in rows(db, sql"SELECT id, full_name, salary FROM employees WHERE salary > ?", 70000.00):
  let
    # Row is a seq[string]; explicit type conversion is required.
    uid = parseInt(row[0])
    name = row[1]
    pay = parseFloat(row[2])
  echo("ID: " & $uid & " | Employee: " & name & " | Salary: " & $pay)

echo("\n--- Lazy iterator (instantRows) ---")
# 'instantRows' streams data directly from libpq without full memory allocation.
for row in instantRows(db, sql"SELECT full_name, department FROM employees"):
  let
    # Access columns by index. Data points directly to the PGresult buffer.
    name = row[0]
    dept = row[1]
  # Check if department string is not empty before printing.
  if len(dept) > 0:
    echo(name & " works in department: " & dept)

echo("\n--- Retrieving column metadata ---")
var
  # DbColumns will hold type information for each selected column.
  cols: DbColumns

# The iterator initializes 'cols' on the first iteration. We discard the rows.
for _ in instantRows(db, cols, sql"SELECT * FROM employees LIMIT 0"):
  discard

# Iterate through the metadata structure to inspect table schema.
for col in cols:
  echo("Column: " & col.name & " | PostgreSQL Type: " & col.typ.name &
       " | Table: " & col.tableName)

echo("\n--- Retrieving a single value ---")
let
  # getValue extracts only the first cell of the first row.
  firstEmp = getValue(db, sql"SELECT full_name FROM employees ORDER BY id LIMIT 1")
echo("First employee: " & firstEmp)

# Always close the connection when done.
close(db)
```

### 3. Prepared Statements and Arbitrary Primary Key Handling
```nim
import
  db_connector/db_postgres
  std/assertions

let
  # Connect via Unix socket. This bypasses TCP overhead and boosts performance on Linux.
  db = open("/var/run/postgresql", "postgres", "secret", "company_db")

# Prepare a named query on the server.
# PostgreSQL requires $1, $2, $3 notation for prepared statement parameters.
let
  updateStmt = prepare(db, "update_salary_by_id", sql"""
    UPDATE employees SET salary = $1 WHERE id = $2
  """, 2)

# Execute the prepared query with string arguments.
# The server handles type casting automatically.
exec(db, updateStmt, "115000.00", "1")

# Verify the update by fetching the new salary.
let
  newSalary = getValue(db, sql"SELECT salary FROM employees WHERE id = 1")
assert(newSalary == "115000.00", "Salary was not updated correctly")

# Example using an explicit primary key name (fallback if PK isn't 'id').
let
  customPkId = tryInsert(db, sql"INSERT INTO employees (full_name, department, salary) VALUES (?, ?, ?)",
                         "id", "Charlie Brown", "Finance", 72000.00)
if customPkId < 0:
  echo("Insertion failed or table lacks a PK named 'id'")

# Count rows affected by the most recent DML statement.
let
  affectedCount = execAffectedRows(db, sql"UPDATE employees SET department = 'Executive' WHERE salary > ?", 100000.00)
echo("Records updated: " & $affectedCount)

# Explicitly remove the prepared statement from the server's memory.
exec(db, sql"DEALLOCATE update_salary_by_id")

# Close the database connection.
close(db)
```

---

## 📖 Detailed Architecture Notes

### Why `seq[string]` for Rows?
The source code explicitly implements `Row` as `seq[string]`. This design choice serves three purposes:
1. **Nativity:** It aligns closest with the native `libpq` API (`PQgetvalue` returns `char*`), minimizing client-side type conversion overhead.
2. **Abstraction:** PostgreSQL supports dozens of specific types (`JSONB`, `UUID`, `INET`, `RANGE`, arbitrary-precision `NUMERIC`). Casting everything to strings hides this complexity from the user.
3. **Forwarding Convenience:** If data simply needs to be logged, passed to another query, or serialized to JSON, the string representation is a universal format.

### How `InstantRow` Works
`InstantRow` does not copy data into Nim's memory on each iteration. Instead, it holds a pointer to the internal `PGresult` buffer. The `row[col]` operation directly calls `PQgetvalue(res, 0, col)`. This saves memory and speeds up reading massive datasets, but requires data access to occur **strictly inside the loop body**. After the iterator finishes or `PGresult` is cleared, the buffer is freed, and accessing `row` afterward leads to undefined behavior.

### Prepared Statements (`SqlPrepared`)
In PostgreSQL, prepared statements are stored server-side and identified by a string name (`distinct string`). When calling `prepare(db, "stmt_name", sql"$1, $2", 2)`, the server caches the execution plan.
Each call to `exec(db, stmtName, args)` automatically:
1. Resets the state of the previous execution.
2. Substitutes arguments in binary or text format.
3. Executes the query via `PQexecPrepared`.
To explicitly remove the query from the server, use standard SQL: `exec(db, sql"DEALLOCATE stmt_name")`.

### Security & Escaping
The module supports two parameterization mechanisms:
- **`SqlQuery` (`?`)**: Automatically converted to a safe format via `dbFormat` and `dbQuote`.
- **`SqlPrepared` (`$1, $2...`)**: Uses PostgreSQL's native protocol, completely eliminating injection risks and reducing DB parser load.

If you must manually insert a string into a dynamic query, use `dbQuote(str)`:
```nim
let
  safeName = dbQuote(userInput)
exec(db, sql"SELECT * FROM employees WHERE full_name = " & safeName)
```
The `dbQuote` function correctly handles single quotes (`'` → `''`) and null terminators (`\0` → `\\0`), which is crucial for binary data in text fields.

---

## ⚠️ Important Notes

1. **`fastRows` Safety for PostgreSQL:** Unlike SQLite, the `fastRows` iterator for PostgreSQL is **safe**. The `libpq` engine uses `PGRES_SINGLE_TUPLE` mode, which correctly handles loop interruption (`break`) without blocking the connection.
2. **Encoding:** `setEncoding` invokes `PQsetClientEncoding`. The change applies only to the current connection and does not affect existing table data.
3. **ID Retrieval (`tryInsertID` / `insertID`):** The module automatically appends `RETURNING id` to the query. This works **only** if the primary key column is named `id`. For arbitrary names, use `tryInsert(db, query, "my_pk_name", args)`.
4. **Parameter Types in `SqlPrepared`:** The server expects strict type matching during query preparation. All arguments are passed as strings and automatically converted by the PostgreSQL server according to the target column type. For explicit casting, use SQL casts (`$1::int`).
5. **Errors:** All procedures with the `try` prefix return success/failure status or `-1` without throwing exceptions. Other procedures call `dbError(db)` on failure, which throws `ref DbError` with the message from `PQerrorMessage`.
6. **Unix Sockets:** For maximum performance on Linux, it is recommended to pass the socket path to the `connection` parameter instead of `localhost`. This bypasses the TCP network stack and can accelerate operations by 30%–175%.

## 🏁 Conclusion
The `db_postgres` package provides a balanced, high-performance interface to PostgreSQL. Its strict separation of fast/safe iterators, support for server-side prepared statements, automatic column metadata handling, and built-in Unix socket optimization make it suitable for both administration scripts and high-load web applications in Nim.



