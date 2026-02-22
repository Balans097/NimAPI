# `memfiles` Module Reference

> **Nim Standard Library — `std/memfiles`**
> Memory-mapped file I/O (POSIX `mmap` / Windows `MapViewOfFile`).
> Also provides fast, zero-copy iteration over lines and delimited records.
> Supported on Windows and POSIX systems only.

---

## Table of Contents

1. [Overview & Mental Model](#overview--mental-model)
2. [Types](#types)
   - [MemFile](#memfile)
   - [MemSlice](#memslice)
   - [MemMapFileStream](#memmapfilestream)
3. [MemFile Lifecycle](#memfile-lifecycle)
   - [open](#open)
   - [close](#close)
   - [flush](#flush)
   - [resize](#resize)
4. [Sub-range Mapping](#sub-range-mapping)
   - [mapMem](#mapmem)
   - [unmapMem](#unmapmem)
5. [Line & Record Iteration](#line--record-iteration)
   - [memSlices](#memslices)
   - [lines (with buffer)](#lines-with-buffer)
   - [lines (simple)](#lines-simple)
6. [MemSlice Utilities](#memslice-utilities)
   - [== (equality)](#-equality)
   - [$ (to string)](#-to-string)
7. [MemMapFileStream](#memmapfilestream-1)
   - [newMemMapFileStream](#newmemmapfilestream)
8. [Platform Notes](#platform-notes)
9. [Complete Worked Examples](#complete-worked-examples)

---

## Overview & Mental Model

**Memory-mapped files** are one of the fastest ways to do file I/O. Instead of reading data into a buffer with `read()`, the operating system maps the file's bytes directly into your process's virtual address space. You then access those bytes through an ordinary pointer — exactly as if the entire file were an in-memory array.

```
Traditional I/O:              Memory-mapped I/O:
─────────────────             ─────────────────────────────────
Disk → kernel buffer          Disk ⟶ OS page cache
  → user buffer (copy!)         ↑ directly addressable
  → your variable (copy!)       └─ your MemFile.mem pointer
```

Key benefits:
- **Zero-copy reads**: no data is ever copied from kernel to user space.
- **Lazy loading**: the OS loads only the pages you actually touch.
- **In-place writes**: storing to `mem[i]` modifies the file immediately (after `flush`).
- **Fast line iteration**: the `memSlices` iterator uses `memchr` — a single C call — to scan for delimiters without ever copying a byte.

Typical usage pattern:

```
open(...)        → get a MemFile with .mem pointing into the file
  │
  ├─ read:  cast/index .mem directly, or iterate with lines/memSlices
  ├─ write: cast .mem to the type you want, write to it
  ├─ flush: force dirty pages to disk
  └─ close: write back any pending changes, release OS handles
```

The module also wraps a `MemFile` inside the standard `Stream` interface via `MemMapFileStream`, so you can pass a memory-mapped file to any API that accepts a `Stream`.

---

## Types

### `MemFile`

```nim
type MemFile* = object
  mem*:  pointer   ## pointer into the mapped region; use this to read/write
  size*: int       ## number of bytes in the mapped region

  # Windows only (public):
  fHandle*:   Handle
  mapHandle*: Handle
  wasOpened*: bool

  # POSIX only (public):
  handle*: cint
```

The central object of the module. After a successful `open`, `mem` points to the beginning of the file's bytes in virtual memory, and `size` holds the number of mapped bytes.

You read from and write to the file by using `mem` as a raw pointer. For example:

```nim
# treat the whole file as an array of bytes
let bytes = cast[ptr UncheckedArray[byte]](f.mem)
echo bytes[0]           # first byte of the file
bytes[0] = 0x42'u8      # write to the file (only if opened writable)
```

The platform-specific handle fields (`fHandle`, `mapHandle`, `handle`) are exposed for advanced use cases — direct OS calls, custom `ioctl` operations, etc. Under normal circumstances you should not need them.

> **Important:** Never set `mem` or `size` manually. They are managed by `open`, `resize`, and `close`.

---

### `MemSlice`

```nim
type MemSlice* = object
  data*: pointer
  size*: int
```

A lightweight, non-owning view into a region of a `MemFile`. It holds a bare pointer and a length — nothing more. This is what `memSlices` yields on each iteration.

`MemSlice` is **not** a Nim string. It is not null-terminated. It does not own its memory. It is valid only as long as the parent `MemFile` is open. If you need a persistent copy, call `$ms` to convert it to a proper Nim string.

---

### `MemMapFileStream`

```nim
type
  MemMapFileStream*    = ref MemMapFileStreamObj
  MemMapFileStreamObj* = object of Stream
    mf:   MemFile
    mode: FileMode
    pos:  int
```

A `ref`-typed `Stream` subtype that wraps a `MemFile`. Once created via `newMemMapFileStream`, it behaves like any other `Stream` — you can use it with all APIs from `std/streams` (e.g., `readLine`, `write`, `setPosition`, `atEnd`, etc.).

---

## MemFile Lifecycle

### `open`

```nim
proc open*(
  filename:    string,
  mode:        FileMode = fmRead,
  mappedSize:  int      = -1,
  offset:      int      = 0,
  newFileSize: int      = -1,
  allowRemap:  bool     = false,
  mapFlags:    cint     = cint(-1)
): MemFile
```

**What it does:** Opens a file and maps it into virtual memory. Returns a fully initialised `MemFile` with `mem` pointing to the start of the mapped region and `size` set to the number of mapped bytes. Raises `OSError` on failure.

**Parameters in detail:**

- **`filename`** — path to the file.

- **`mode`** — how the file is opened. Common values:
  - `fmRead` (default) — read-only mapping. Writing through `mem` will segfault.
  - `fmReadWrite` — read-write mapping. Changes to `mem` are written back on `flush`/`close`.
  - `fmWrite` — used together with `newFileSize` to *create* a new file. The file must not already exist with this combination.
  - `fmAppend` is **not supported** and will raise `IOError`.

- **`mappedSize`** — how many bytes to map, starting at `offset`. `-1` (default) maps the entire file (or the full `newFileSize` for a new file). Use a positive value to map only a portion of a large file.

- **`offset`** — byte offset into the file where the mapping starts. **Must be a multiple of the OS page size** (typically 4096 bytes). You cannot start a mapping at an arbitrary byte.

- **`newFileSize`** — if > 0, creates a brand-new file of exactly this many bytes and maps it. Only valid when `mode` is not `fmRead`. Has no effect on existing files.

- **`allowRemap`** — if `true`, the underlying OS file handle is kept open after mapping so that `resize` can be called later. If `false` (default), the handle is closed after mapping to conserve resources. Only set this when you know you will call `resize`.

- **`mapFlags`** — advanced: override the default `mmap`/`MapViewOfFile` flags. The default on POSIX is `MAP_SHARED`. Incorrect flags may cause `open` to fail or produce unexpected behaviour.

```nim
import std/memfiles

# 1. Open an existing file read-only (most common use case)
var f = memfiles.open("data.txt")
defer: f.close()
# f.mem now points to the file's bytes, f.size = file length in bytes

# 2. Open read-write to modify an existing file in place
var g = memfiles.open("config.bin", mode = fmReadWrite)
defer: g.close()

# 3. Create a new 4 KB file (will be zero-filled by the OS)
var h = memfiles.open("/tmp/scratch.bin", mode = fmReadWrite, newFileSize = 4096)
defer: h.close()

# 4. Map only the first 512 bytes of a large file
var part = memfiles.open("bigfile.dat", mode = fmRead, mappedSize = 512)
defer: part.close()

# 5. Map a 512-byte chunk starting at byte 4096 (one page in)
var mid = memfiles.open("bigfile.dat", mode = fmRead,
                         mappedSize = 512, offset = 4096)
defer: mid.close()
```

---

### `close`

```nim
proc close*(f: var MemFile)
```

**What it does:** Unmaps the virtual memory region, flushes any dirty (modified) pages back to disk, and closes all OS-level file handles. After `close`, `f.mem` is set to `nil` and `f.size` to `0`. Raises `OSError` if any of the OS calls fail.

Always call `close` when you are done. The idiomatic Nim pattern is `defer: f.close()` immediately after `open`.

```nim
var f = memfiles.open("log.txt")
defer: f.close()  # guaranteed, even if an exception is raised
# ... use f ...
```

If you opened the file with write access, `close` is where the data is guaranteed to be safely written to the file system. Do not terminate your process abruptly before `close` if data integrity matters.

---

### `flush`

```nim
proc flush*(f: var MemFile; attempts: Natural = 3)
```

**What it does:** Forces all modified pages in the mapped region to be written to the underlying file on disk. Raises `OSError` if flushing fails after `attempts` retries.

On POSIX, this calls `msync(MS_SYNC | MS_INVALIDATE)`. On Windows, it calls `FlushViewOfFile`.

The retry logic exists because transient errors (`EBUSY` on POSIX, `ERROR_LOCK_VIOLATION` on Windows) can occur when other processes are reading the same pages. The default of 3 attempts covers most real-world cases.

**When to use it:** When you have written data through `mem` and want to ensure durability before proceeding — for example, before calling `resize`, before checkpointing, or periodically in a long-running process that writes to a large mapped file.

```nim
var f = memfiles.open("output.bin", mode = fmReadWrite)
defer: f.close()

let data = cast[ptr UncheckedArray[byte]](f.mem)
data[0] = 0xFF

# Make sure byte 0 is on disk before continuing:
f.flush()
```

---

### `resize`

```nim
proc resize*(f: var MemFile, newFileSize: int) {.raises: [IOError, OSError].}
```

**What it does:** Changes both the on-disk file size and the in-memory mapping to `newFileSize` bytes. After this call, `f.mem` may point to a **different address** (the OS may have remapped the region), and `f.size` is updated. Raises `IOError` if `newFileSize < 1` or if the file was not opened with `allowRemap = true`.

On Linux, this uses `mremap` — an in-place remap that can be over 100× faster than the unmapping + re-mapping cycle used on other POSIX systems. On other POSIX and on Windows, the mapping is torn down and rebuilt.

**Preconditions:**
- The file must have been opened with `allowRemap = true`.
- The entire file must be mapped at offset 0 (i.e., `offset = 0`, `mappedSize = -1`).
- `newFileSize` must be ≥ 1.

After `resize`, any existing pointers you computed from the old `f.mem` value are **invalid**. Recompute all pointers from the new `f.mem`.

```nim
var f = memfiles.open("/tmp/grow.bin",
                      mode        = fmReadWrite,
                      newFileSize = 1024,
                      allowRemap  = true)
defer: f.close()

echo f.size  # 1024

# Write some data, then grow the file:
f.flush()
f.resize(2048)
echo f.size  # 2048
# f.mem may now be at a different address — recompute any derived pointers
```

---

## Sub-range Mapping

These two procedures let you create additional mapped views into an already-open `MemFile` — for example, mapping a different region as you slide through a large file. They are only useful when `allowRemap = true` was passed to `open`.

### `mapMem`

```nim
proc mapMem*(
  m:          var MemFile,
  mode:       FileMode = fmRead,
  mappedSize: int      = -1,
  offset:     int      = 0,
  mapFlags:   cint     = cint(-1)
): pointer
```

**What it does:** Creates an *additional* mapping of a portion of `m`'s file and returns a raw pointer to it. The mapping is independent of the one already stored in `m.mem`. Raises `OSError` on failure. `fmAppend` mode is not supported.

- `mappedSize = -1` maps the whole file (POSIX requires `mappedSize > 0`; on Windows `-1` maps to end-of-file).
- `offset` must be page-aligned.

You must eventually call `unmapMem` on the returned pointer with the same `size` you requested, or you will leak OS resources.

```nim
var f = memfiles.open("bigfile.bin", mode = fmRead, allowRemap = true)
defer: f.close()

# Map a 512-byte view starting at page 2 (byte 8192 on a 4K-page system)
let p = f.mapMem(mode = fmRead, mappedSize = 512, offset = 8192)
defer: f.unmapMem(p, 512)

let arr = cast[ptr UncheckedArray[byte]](p)
echo arr[0]  # first byte of the second page
```

---

### `unmapMem`

```nim
proc unmapMem*(f: var MemFile, p: pointer, size: int)
```

**What it does:** Releases the additional memory mapping created by `mapMem`. Writes back any dirty pages to the file system if the mapping was writable. Raises `OSError` on failure.

`size` **must be exactly the same value** that was passed as `mappedSize` to `mapMem`. Mismatched sizes produce undefined behaviour (POSIX `munmap` will either reject the call or unmap the wrong range).

*(See the `mapMem` example above for usage.)*

---

## Line & Record Iteration

These iterators are among the fastest ways to process a text file line by line in any language — they use `memchr` to scan for delimiters without copying data.

### `memSlices`

```nim
iterator memSlices*(
  mfile: MemFile,
  delim: char = '\l',
  eat:   char = '\r'
): MemSlice {.inline.}
```

**What it does:** Yields one `MemSlice` per delimited record in the file. Each slice is a raw `(pointer, size)` pair pointing directly into the file's mapped memory — **no copying occurs**. Delimiters are not included in the yielded slice.

**Parameters:**
- `delim` — the primary record separator (default: `'\l'` = `\n`). Each occurrence of this byte ends a record.
- `eat` — an optional character that is *trimmed* from the end of the record if present (default: `'\r'`). This handles `\r\n` Windows line endings on a line-by-line basis: the `\n` is the delimiter, and the `\r` is eaten. Pass `eat = '\0'` to disable trimming and be strictly `delim`-only.

**What it handles automatically:**
- Unix `\n` endings.
- Windows `\r\n` endings (default behaviour eats the `\r`).
- A final line with no trailing delimiter is yielded as a normal record.
- Mixed line endings within the same file work correctly.

**What it does NOT handle:**
- Old Mac OS 9 `\r`-only endings (use `delim='\r', eat='\0'` for those).

**Safety:** The returned `MemSlice` has no bounds checking, no null terminator, and is not a Nim string. Use C `mem*` semantics — you can read `slice.size` bytes starting at `slice.data`, but nothing beyond that.

```nim
import std/memfiles

var f = memfiles.open("data.txt")
defer: f.close()

var lineCount = 0
for slice in memSlices(f):
  # slice.data: pointer into the file
  # slice.size: number of bytes in this line (excluding delimiter)
  inc lineCount

  # Access first character safely:
  if slice.size > 0:
    let first = cast[cstring](slice.data)[0]
    if first != '#':  # skip comment lines
      echo $slice     # convert to Nim string only when needed
echo "Total lines: ", lineCount
```

```nim
# Iterate over null-byte-delimited records (e.g. `find -print0` output)
for slice in memSlices(f, delim = '\0', eat = '\0'):
  echo $slice
```

---

### `lines` (with buffer)

```nim
iterator lines*(
  mfile: MemFile,
  buf:   var string,
  delim: char = '\l',
  eat:   char = '\r'
): string {.inline.}
```

**What it does:** Iterates over lines just like `memSlices`, but copies each line into the caller-supplied `buf` string and yields it. This avoids allocating a new string on every iteration — the same string object is reused each time.

This is the highest-performance option when you need actual Nim strings (rather than raw `MemSlice` pointers) and you are iterating over many lines.

```nim
import std/memfiles

var f = memfiles.open("data.txt")
defer: f.close()

var buf = newStringOfCap(256)  # pre-allocate a reasonable capacity
for line in lines(f, buf):
  # `line` is the same object as `buf` — do not store it between iterations!
  echo line
```

> **Important:** `buf` is modified in place on each iteration. If you need to keep a line after the loop body executes, you must copy it: `let saved = line` or `let saved = buf`.

---

### `lines` (simple)

```nim
iterator lines*(
  mfile: MemFile,
  delim: char = '\l',
  eat:   char = '\r'
): string {.inline.}
```

**What it does:** The convenient version of `lines` — no buffer argument needed. Internally allocates and reuses a single string buffer. Yields each line as a Nim string.

Use this when simplicity matters more than micro-optimising allocations.

```nim
import std/memfiles

var f = memfiles.open("log.txt")
defer: f.close()

for line in lines(f):
  echo line
```

---

## MemSlice Utilities

### `==` (equality)

```nim
proc `==`*(x, y: MemSlice): bool
```

**What it does:** Returns `true` if `x` and `y` have the same `size` and the same byte content (compared with `equalMem`). The comparison is byte-for-byte, case-sensitive, and encoding-agnostic.

```nim
var f = memfiles.open("data.txt")
defer: f.close()

var slices: seq[MemSlice]
for s in memSlices(f):
  slices.add(s)

# Compare two slices by content:
if slices.len >= 2:
  echo slices[0] == slices[1]
```

---

### `$` (to string)

```nim
proc `$`*(ms: MemSlice): string {.inline.}
```

**What it does:** Copies the bytes of `ms` into a freshly allocated Nim string and returns it. This is the escape hatch from the zero-copy world: when you need to store a line, pass it to a `string`-accepting API, or keep it past the lifetime of the `MemFile`, call `$`.

```nim
var f = memfiles.open("names.txt")
defer: f.close()

var names: seq[string]
for slice in memSlices(f):
  names.add($slice)  # copy into a persistent string
# f can now be closed; names remains valid
```

---

## `MemMapFileStream`

### `newMemMapFileStream`

```nim
proc newMemMapFileStream*(
  filename: string,
  mode:     FileMode = fmRead,
  fileSize: int      = -1
): MemMapFileStream
```

**What it does:** Opens `filename` as a memory-mapped file and wraps it in a `MemMapFileStream` — a standard Nim `Stream`. Once created, you can use all the usual `Stream` operations on it: `readLine`, `writeLine`, `write`, `read`, `setPosition`, `getPosition`, `atEnd`, `flush`, `close`, etc.

**Parameters:**
- `filename` — path to the file.
- `mode` — `fmRead` (default) or `fmReadWrite`. `fmWrite` creates a new file only when combined with `fileSize`.
- `fileSize` — if > 0, creates a new file of this size. Only valid when `mode` is not `fmRead`. Equivalent to `newFileSize` in `open`.

Raises `OSError` if the file cannot be opened.

**When to use it:** When you want the speed of memory mapping but your code is written against the `Stream` abstraction — for example, passing to a parser, a serialiser, or any library that accepts `Stream`.

```nim
import std/[memfiles, streams]

# Read a file through the Stream interface
var s = newMemMapFileStream("config.txt")
defer: s.close()

while not s.atEnd():
  echo s.readLine()
```

```nim
# Create and write to a new file through the Stream interface
var s = newMemMapFileStream("/tmp/out.bin", mode = fmReadWrite, fileSize = 1024)
defer: s.close()

s.write("Hello, memory-mapped world!")
s.setPosition(0)
echo s.readLine()
```

---

## Platform Notes

**`fmAppend` is unsupported.** Passing `mode = fmAppend` to `open` or `mapMem` raises `IOError`. Memory-mapped I/O has no concept of an append pointer — you must manage write positions manually through `mem`.

**`offset` must be page-aligned.** The OS maps memory in page-sized chunks (typically 4096 bytes on x86, 16384 on Apple Silicon). Passing a non-page-aligned `offset` will cause `open` or `mapMem` to fail with an `OSError`.

**`resize` requires `allowRemap = true`.** If you did not pass `allowRemap = true` to `open`, the OS file handle is closed after mapping, and `resize` cannot reopen it. Calling `resize` in this case raises `IOError`.

**After `resize`, all pointers from the old mapping are invalid.** This includes anything you computed as `cast[int](f.mem) + offset`. Recompute everything from the new `f.mem` value.

**Windows-specific handles.** The `fHandle`, `mapHandle`, and `wasOpened` fields are Windows-only. On POSIX, only `handle` exists. Code that reads these fields must use `when defined(windows)` / `when defined(posix)` guards.

**File size limits.** `size: int` may overflow on 32-bit platforms for files between 2 GiB and 4 GiB, because `int` is 32-bit there. On 64-bit platforms this is not an issue.

**Linux `mremap` optimisation.** On Linux, `resize` uses `mremap` with `MREMAP_MAYMOVE`, which is in-place (or near-in-place) and dramatically faster than the unmapping + re-mapping strategy used on other POSIX systems.

---

## Complete Worked Examples

### Example 1: Count lines in a large file — fastest possible method

```nim
import std/memfiles

proc countLines(path: string): int =
  var f = memfiles.open(path)
  defer: f.close()
  for _ in memSlices(f):
    inc result

echo countLines("/var/log/syslog")
```

---

### Example 2: Read a CSV file, skip comment lines, collect fields

```nim
import std/memfiles

proc parseCSV(path: string): seq[seq[string]] =
  var f = memfiles.open(path)
  defer: f.close()

  for slice in memSlices(f):
    if slice.size == 0: continue
    if cast[cstring](slice.data)[0] == '#': continue  # skip comments

    let line = $slice
    result.add(line.split(','))

let rows = parseCSV("data.csv")
for row in rows:
  echo row
```

---

### Example 3: Write structured data to a new memory-mapped file

```nim
import std/memfiles

type Header = object
  magic:   uint32
  version: uint32
  count:   int64

var f = memfiles.open("/tmp/mydata.bin",
                      mode        = fmReadWrite,
                      newFileSize = 4096)
defer: f.close()

# Write a header at the start of the file
let hdr = cast[ptr Header](f.mem)
hdr.magic   = 0xDEADBEEF'u32
hdr.version = 1'u32
hdr.count   = 42'i64

# Write raw bytes further in
let body = cast[ptr UncheckedArray[byte]](
             cast[int](f.mem) + sizeof(Header))
for i in 0 ..< 16:
  body[i] = byte(i)

f.flush()
echo "Written"
```

---

### Example 4: Grow a log file on the fly with `resize`

```nim
import std/memfiles

const PageSize = 4096

var f = memfiles.open("/tmp/growlog.bin",
                      mode        = fmReadWrite,
                      newFileSize = PageSize,
                      allowRemap  = true)
defer: f.close()

var writePos = 0

proc appendEntry(f: var MemFile, data: string, pos: var int) =
  let needed = pos + data.len
  if needed > f.size:
    f.flush()
    # Grow by at least one page
    let newSize = ((needed div PageSize) + 1) * PageSize
    f.resize(newSize)
    # f.mem may have changed after resize!

  let dst = cast[ptr UncheckedArray[byte]](cast[int](f.mem) + pos)
  for i, c in data:
    dst[i] = byte(c)
  inc(pos, data.len)

appendEntry(f, "First entry\n",  writePos)
appendEntry(f, "Second entry\n", writePos)
echo "Log size: ", f.size
```

---

### Example 5: Use `MemMapFileStream` with a line-oriented parser

```nim
import std/[memfiles, streams, strutils]

proc parseCfg(path: string): seq[(string, string)] =
  var s = newMemMapFileStream(path)
  defer: s.close()

  while not s.atEnd():
    let line = s.readLine().strip()
    if line.len == 0 or line[0] == ';': continue  # skip blank/comment lines
    let eq = line.find('=')
    if eq > 0:
      result.add((line[0 ..< eq].strip(), line[eq+1 .. ^1].strip()))

for (key, val) in parseCfg("settings.ini"):
  echo key, " = ", val
```

---

### Example 6: Custom delimiter — NUL-separated records

```nim
import std/memfiles

# Process output of `find /path -print0`
# Records are separated by NUL bytes; no "eat" character needed.
var f = memfiles.open("find_output.bin")
defer: f.close()

for slice in memSlices(f, delim = '\0', eat = '\0'):
  if slice.size > 0:
    echo $slice  # each file path
```
