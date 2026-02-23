# parsecsv — Module Reference

> **Nim standard library** | `import std/parsecsv`

A simple, high-performance CSV (Comma-Separated Values) parser. It reads CSV data line by line, exposing each row as a sequence of strings. The parser is stream-based, which means it does not load the entire file into memory — suitable for large files.

---

## Table of Contents

1. [Types](#types)
   - [CsvRow](#csvrow)
   - [CsvParser](#csvparser)
   - [CsvError](#csverror)
2. [Procedures](#procedures)
   - [open (from Stream)](#open-from-stream)
   - [open (from file path)](#open-from-file-path)
   - [close](#close)
   - [readRow](#readrow)
   - [readHeaderRow](#readheaderrow)
   - [rowEntry](#rowentry)
   - [processedRows](#processedrows)

---

## Types

### `CsvRow`

```nim
type CsvRow* = seq[string]
```

A type alias for `seq[string]`. Represents a single parsed row of a CSV file — an ordered list of string fields. Each element corresponds to one column value in that row.

---

### `CsvParser`

```nim
type CsvParser* = object of BaseLexer
  row*: CsvRow
  headers*: seq[string]
```

The central object of the module. It holds the internal parsing state and exposes two public fields:

- **`row`** — the last successfully read row, as a `CsvRow` (i.e., `seq[string]`). Updated on every call to `readRow`.
- **`headers`** — the list of column names read by `readHeaderRow`. Used as keys when calling `rowEntry`.

You must always initialize a `CsvParser` by calling one of the `open` procedures before using it, and finalize it with `close`.

---

### `CsvError`

```nim
type CsvError* = object of IOError
```

An exception raised when a parsing error occurs (e.g., an unexpected character instead of a separator, or an unterminated quoted field). Inherits from `IOError`. Error messages include the filename, line number, and column number when available.

---

## Procedures

---

### `open` (from Stream)

```nim
proc open*(self: var CsvParser, input: Stream, filename: string,
           separator = ',', quote = '"', escape = '\0',
           skipInitialSpace = false)
```

Initializes the parser to read from an existing `Stream` object. The `filename` parameter is used only for error messages — it does not affect file access.

**Parameters:**

| Parameter | Default | Description |
|---|---|---|
| `separator` | `','` | The character that separates fields within a row. |
| `quote` | `'"'` | The character used to wrap fields that contain special characters (separator, newline, etc.). Set to `'\0'` to disable quoting entirely. |
| `escape` | `'\0'` | When set to a non-null character (e.g., `'\\'`), it acts as an escape prefix: the character following it is treated literally. When `'\0'` and quoting is enabled, two consecutive quote characters are treated as a single literal quote (RFC 4180 style). |
| `skipInitialSpace` | `false` | If `true`, whitespace characters immediately after a separator are ignored. Useful for human-friendly CSVs like `a, b, c`. |

**When to use:** When you already have a `Stream` (e.g., a `StringStream` for in-memory data, or a stream obtained from a network source).

```nim
import std/[parsecsv, streams]

var strm = newStringStream("name,age\nAlice,30\nBob,25")
var p: CsvParser
p.open(strm, "data.csv")

while p.readRow():
  echo p.row[0], " is ", p.row[1]

p.close()
strm.close()
# Output:
# name is age
# Alice is 30
# Bob is 25
```

---

### `open` (from file path)

```nim
proc open*(self: var CsvParser, filename: string,
           separator = ',', quote = '"', escape = '\0',
           skipInitialSpace = false)
```

A convenience overload that opens the file at the given path and creates the stream internally. All other parameters behave identically to the stream-based `open`.

**When to use:** The most common case — you have a CSV file path and just want to start reading.

```nim
import std/parsecsv

var p: CsvParser
p.open("employees.csv", separator = ';')

while p.readRow():
  for field in p.row:
    stdout.write(field & "\t")
  echo ""

p.close()
```

**Note:** If the file cannot be opened, `CsvError` is raised immediately.

---

### `close`

```nim
proc close*(self: var CsvParser) {.inline.}
```

Closes the parser and releases the underlying input stream. Always call `close` when you are done parsing — this frees the file handle and internal buffer.

```nim
var p: CsvParser
p.open("data.csv")
# ... read data ...
p.close()  # ← do not forget this
```

**Note:** After `close`, the `CsvParser` object should not be used without calling `open` again.

---

### `readRow`

```nim
proc readRow*(self: var CsvParser, columns = 0): bool
```

Reads and parses the next non-blank row from the input. Populates `self.row` with the fields of that row and returns `true`. Returns `false` when there are no more rows (end of file). **Blank lines are silently skipped.**

**Parameter `columns`:** If greater than 0, the parser enforces that every row has exactly this many columns. If a row has a different count, `CsvError` is raised. This is a useful safeguard for strict data formats.

```nim
import std/[parsecsv, streams]

# A CSV with a blank line in the middle — it is automatically skipped
var strm = newStringStream("1,2,3\n\n4,5,6")
var p: CsvParser
p.open(strm, "test.csv")

while p.readRow():
  echo p.row  # prints @["1","2","3"] then @["4","5","6"]

p.close()
strm.close()
```

**Enforcing column count:**

```nim
# Raises CsvError if any row does not have exactly 3 columns
while p.readRow(columns = 3):
  processRow(p.row)
```

---

### `readHeaderRow`

```nim
proc readHeaderRow*(self: var CsvParser)
```

Reads the first row of the file and stores it in `self.headers`. After this call, `self.row` also contains the header values. Subsequent calls to `readRow` will move to the data rows, while `self.headers` remains unchanged — acting as a persistent column name registry.

This procedure exists to support named column access via `rowEntry`. Without calling `readHeaderRow` first, `rowEntry` will not work correctly.

```nim
import std/[parsecsv, streams]

var strm = newStringStream("city,country,population\nBerlin,Germany,3600000\nTokyo,Japan,13960000")
var p: CsvParser
p.open(strm, "cities.csv")

p.readHeaderRow()
echo p.headers  # @["city", "country", "population"]

while p.readRow():
  echo p.rowEntry("city"), " (", p.rowEntry("country"), ")"

p.close()
strm.close()
# Output:
# Berlin (Germany)
# Tokyo (Japan)
```

---

### `rowEntry`

```nim
proc rowEntry*(self: var CsvParser, entry: string): var string
```

Returns a reference to the value in the current row (`self.row`) that corresponds to the column named `entry` in `self.headers`. This is the primary way to access fields by name rather than by index.

**Requires:** `readHeaderRow` must have been called before using this procedure. Otherwise `self.headers` is empty and every lookup will raise `KeyError`.

**Raises:** `KeyError` if `entry` does not match any name in `self.headers`.

Because the result is `var string` (a mutable reference), you can also assign to it directly — though this is rarely needed.

```nim
import std/[parsecsv, streams]

let csv = "product,price,qty\nApple,0.99,100\nBanana,0.45,200"
var strm = newStringStream(csv)
var p: CsvParser
p.open(strm, "shop.csv")
p.readHeaderRow()

while p.readRow():
  let name  = p.rowEntry("product")
  let price = p.rowEntry("price")
  let qty   = p.rowEntry("qty")
  echo name, ": $", price, " x ", qty

p.close()
strm.close()
# Output:
# Apple: $0.99 x 100
# Banana: $0.45 x 200
```

**Accessing a nonexistent column:**

```nim
try:
  echo p.rowEntry("discount")
except KeyError as e:
  echo "Column not found: ", e.msg
```

---

### `processedRows`

```nim
proc processedRows*(self: var CsvParser): int {.inline.}
```

Returns the total number of times `readRow` has been called, regardless of whether the call returned `true` (a real row was read) or `false` (end of file was reached). The counter starts at 0 and is incremented by 1 on every `readRow` call, even past EOF.

**Use cases:**
- Progress tracking while reading large files.
- Debugging or logging (e.g., "failed on row N").
- Verifying that a file has the expected number of data rows.

```nim
import std/[parsecsv, streams]

var strm = newStringStream("a,b\n1,2\n3,4")
var p: CsvParser
p.open(strm, "test.csv")

while p.readRow():
  echo "Read row #", p.processedRows(), ": ", p.row

# After the loop, readRow returned false once more — counter is 3
echo "Total readRow calls: ", p.processedRows()

p.close()
strm.close()
# Output:
# Read row #1: @["a", "b"]
# Read row #2: @["1", "2"]
# Read row #3: @["3", "4"]
# Total readRow calls: 3
```

> **Important:** `processedRows` counts *calls to `readRow`*, not the number of data rows in the file. If the file has blank lines (which are skipped), those do not add to the counter. However, each `readRow` call that returns `false` (at EOF) does increment the counter.

---

## Complete Usage Example

The following example demonstrates the full workflow — opening a file, reading headers, iterating over data rows by name, and handling errors.

```nim
import std/parsecsv

let content = """id,name,score
1,Alice,95
2,Bob,87
3,Carol,91
"""
writeFile("results.csv", content)

var p: CsvParser
p.open("results.csv")
p.readHeaderRow()

echo "Columns: ", p.headers

while p.readRow():
  echo p.rowEntry("name"), " scored ", p.rowEntry("score")

echo "Rows processed: ", p.processedRows()
p.close()

# Output:
# Columns: @["id", "name", "score"]
# Alice scored 95
# Bob scored 87
# Carol scored 91
# Rows processed: 5
```

---

## Quick Reference Table

| Procedure | Purpose |
|---|---|
| `open(parser, stream, filename, ...)` | Initialize parser from an existing stream |
| `open(parser, filename, ...)` | Initialize parser directly from a file path |
| `close(parser)` | Release the parser and its stream |
| `readRow(parser, columns?)` | Read the next non-blank row → `bool` |
| `readHeaderRow(parser)` | Read first row and store it as `headers` |
| `rowEntry(parser, name)` | Get current row's value by column name |
| `processedRows(parser)` | Get total number of `readRow` calls so far |
