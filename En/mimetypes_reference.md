# `std/mimetypes` Module Reference (Nim)

> A MIME types database: converts file extensions to MIME types and vice versa, and supports registration of custom types.

---

## Table of Contents

1. [Data Types](#data-types)
2. [The `mimes` Constant](#the-mimes-constant)
3. [Functions](#functions)
   - [newMimetypes](#newmimetypes)
   - [getMimetype](#getmimetype)
   - [getExt](#getext)
   - [register](#register)
4. [Behavior and Nuances](#behavior-and-nuances)
5. [Built-in Type Categories](#built-in-type-categories)
6. [Practical Examples](#practical-examples)

---

## Data Types

### `MimeDB`

An object that stores the extension → MIME type mapping table.

```nim
type
  MimeDB* = object
    mimes: OrderedTable[string, string]
```

The `mimes` field is private — the database is accessed exclusively through the exported functions. The insertion order is preserved (using `OrderedTable`), which matters for `getExt` — it returns the **first** extension found for a given MIME type.

---

## The `mimes` Constant

```nim
const mimes* = { "ez": "application/andrew-inset", ... }
```

A built-in constant table containing over **700 extension → MIME type pairs**. It is exported publicly and can be used directly when raw access to the data is needed without creating a `MimeDB`.

Covered categories: `application/`, `audio/`, `chemical/`, `font/`, `image/`, `message/`, `model/`, `text/`, `video/`, `x-conference/`.

Notably includes Nim file support:
```nim
"nim"    → "text/nim"
"nimble" → "text/nimble"
"nimf"   → "text/nim"
"nims"   → "text/nim"
```

```nim
import std/mimetypes

# Using the constant directly
for ext, mime in mimes:
  if mime == "image/png":
    echo ext   # png
```

---

## Functions

### `newMimetypes`

```nim
func newMimetypes*(): MimeDB
```

Creates a new MIME type database pre-populated with all built-in entries from the `mimes` constant.

**Returns:** an initialized `MimeDB` object.

The function is a `func` (no side effects in Nim's sense), although it internally uses `{.cast(noSideEffect).}` for the table initialization conversion.

```nim
import std/mimetypes

var db = newMimetypes()

# The database is ready to use immediately
echo db.getMimetype("html")  # text/html
echo db.getMimetype("mp3")   # audio/mpeg
echo db.getMimetype("png")   # image/png
```

---

### `getMimetype`

```nim
func getMimetype*(mimedb: MimeDB, ext: string, default = "text/plain"): string
```

Returns the MIME type corresponding to the given file extension.

**Parameters:**
- `mimedb` — the MIME database
- `ext` — the file extension (with or without a leading dot)
- `default` — the value returned when the extension is not found in the database (default: `"text/plain"`)

**Behavior:**
- The extension is **lowercased** before lookup (case-insensitive).
- A leading dot in the extension is **silently stripped** — `".html"` and `"html"` produce the same result.
- An empty string or unknown extension returns `default`.

```nim
import std/mimetypes

var db = newMimetypes()

# Basic usage
echo db.getMimetype("html")   # text/html
echo db.getMimetype("mp4")    # video/mp4
echo db.getMimetype("png")    # image/png
echo db.getMimetype("json")   # application/json
echo db.getMimetype("pdf")    # application/pdf

# Case-insensitive
echo db.getMimetype("MP4")    # video/mp4
echo db.getMimetype("HTML")   # text/html
echo db.getMimetype("PNG")    # image/png

# Leading dot is stripped automatically
echo db.getMimetype(".html")  # text/html
echo db.getMimetype(".MP4")   # video/mp4

# Unknown extension → default
echo db.getMimetype("xyz123")        # text/plain  (default)
echo db.getMimetype("INVALID")       # text/plain
echo db.getMimetype("")              # text/plain

# Custom default value
echo db.getMimetype("xyz", "application/octet-stream")
# application/octet-stream

# Nim files
echo db.getMimetype("nim")    # text/nim
echo db.getMimetype("nimble") # text/nimble

# Using with a full file path via splitFile
import std/os
let (_, _, ext) = splitFile("/data/image.JPEG")
echo db.getMimetype(ext)      # image/jpeg
```

---

### `getExt`

```nim
func getExt*(mimedb: MimeDB, mimetype: string, default = "txt"): string
```

Returns the file extension corresponding to the given MIME type.

**Parameters:**
- `mimedb` — the MIME database
- `mimetype` — the MIME type to look up
- `default` — the value returned when the type is not found (default: `"txt"`)

**Behavior:**
- The MIME type is **lowercased** before lookup.
- The returned extension has **no leading dot**.
- Since one MIME type can map to multiple extensions, the **first** one encountered in the `OrderedTable` insertion order is returned.
- An empty string or unknown type returns `default`.

```nim
import std/mimetypes

var db = newMimetypes()

# Basic usage
echo db.getExt("text/html")          # html
echo db.getExt("video/mp4")          # mp4
echo db.getExt("image/png")          # png
echo db.getExt("application/json")   # json
echo db.getExt("audio/mpeg")         # mpga  (first in the database for this type)

# Case-insensitive
echo db.getExt("TEXT/HTML")          # html
echo db.getExt("VIDEO/MP4")          # mp4

# Unknown type → default
echo db.getExt("INVALID/NONEXISTENT")  # txt
echo db.getExt("")                     # txt

# Custom default value
echo db.getExt("application/x-unknown", "bin")
# bin

# One type, multiple extensions — first registered is returned
echo db.getExt("image/jpeg")  # jpeg (not jpg — jpeg comes first in the table)
echo db.getExt("audio/ogg")   # oga  (not ogg or spx)
```

---

### `register`

```nim
func register*(mimedb: var MimeDB, ext: string, mimetype: string)
```

Registers a new extension → MIME type mapping in the database. Can add new types or override existing ones.

**Parameters:**
- `mimedb` — the MIME database (mutable, `var`)
- `ext` — the file extension (without a leading dot)
- `mimetype` — the MIME type string

**Behavior:**
- Both arguments are **lowercased** before being stored.
- If the extension already exists, its value is **overwritten**.
- An empty string (after `strip`) in either `ext` or `mimetype` triggers an `AssertionDefect`.
- Do not include a leading dot in `ext` — it will become part of the key and break lookups.

```nim
import std/mimetypes

var db = newMimetypes()

# Add a new type
db.register(ext = "fakext", mimetype = "text/fakelang")
echo db.getMimetype("fakext")   # text/fakelang
echo db.getMimetype("FAKEXT")   # text/fakelang  (case-insensitive)
echo db.getExt("text/fakelang") # fakext

# Override an existing type
db.register(ext = "json", mimetype = "application/json5")
echo db.getMimetype("json")     # application/json5

# Input case does not matter — everything is stored lowercase
db.register(ext = "MYEXT", mimetype = "APPLICATION/MY-TYPE")
echo db.getMimetype("myext")    # application/my-type
echo db.getMimetype("MYEXT")    # application/my-type

# Errors on empty arguments (AssertionDefect):
# db.register(ext = "", mimetype = "text/plain")    # error!
# db.register(ext = "txt", mimetype = "")           # error!
# db.register(ext = "  ", mimetype = "text/plain")  # error! (whitespace only)

# Adding types for a project
db.register("toml",    "application/toml")
db.register("graphql", "application/graphql")
db.register("wasm",    "application/wasm")
db.register("avif",    "image/avif")
db.register("heic",    "image/heic")
```

---

## Behavior and Nuances

### Case-insensitivity

All functions lowercase extensions and MIME types before lookup or storage, making the entire API case-insensitive.

```nim
var db = newMimetypes()
# All of the following return the same result:
doAssert db.getMimetype("mp4")  == "video/mp4"
doAssert db.getMimetype("MP4")  == "video/mp4"
doAssert db.getMimetype("Mp4")  == "video/mp4"
doAssert db.getMimetype(".MP4") == "video/mp4"
```

### Leading Dot Handling

`getMimetype` handles extensions both with and without a leading dot. `register` expects an extension **without** a dot — a leading dot would become part of the lookup key and break the search.

```nim
var db = newMimetypes()

# Correct:
echo db.getMimetype(".html")  # text/html  ← dot is stripped automatically
db.register("newext", "text/new")  # no dot

# Incorrect (dot becomes part of the key):
# db.register(".newext", "text/new")
# db.getMimetype("newext")  →  "text/plain"  (not found!)
```

### Multiple Extensions per MIME Type

Many MIME types in the database correspond to several extensions. `getExt` returns only the first one found.

```nim
var db = newMimetypes()

# image/jpeg maps to: jpeg, jpg, jpe
echo db.getExt("image/jpeg")  # jpeg

# audio/mpeg maps to: mpga, mp2, mp2a, mp3, m2a, m3a
echo db.getExt("audio/mpeg")  # mpga

# text/plain maps to: txt, text, conf, def, list, log, in
echo db.getExt("text/plain")  # txt
```

### `getExt` is a Linear Scan

`getExt` performs a **linear scan** of the entire table since it searches by value rather than by key. With 700+ entries this is practically imperceptible, but for hot paths in high-throughput code consider caching the results.

---

## Built-in Type Categories

The module includes extensions for the following main categories:

| Category        | Example extensions                                          |
|-----------------|-------------------------------------------------------------|
| `application/`  | `pdf`, `zip`, `json`, `xml`, `doc`, `xls`, `ppt`, `jar`, `apk`, `epub` |
| `audio/`        | `mp3`, `ogg`, `wav`, `flac`, `aac`, `m4a`, `opus`, `weba`  |
| `chemical/`     | `cif`, `cml`, `xyz`                                         |
| `font/`         | `ttf`, `otf`, `woff`, `woff2`, `ttc`                       |
| `image/`        | `png`, `jpeg`, `gif`, `webp`, `svg`, `bmp`, `ico`, `tiff`, `psd` |
| `message/`      | `eml`, `mime`                                               |
| `model/`        | `dae`, `wrl`, `x3d`, `obj`, `stl`                          |
| `text/`         | `html`, `css`, `js`, `csv`, `txt`, `xml`, `c`, `java`, `nim` |
| `video/`        | `mp4`, `avi`, `mkv`, `webm`, `mov`, `ogv`, `flv`, `wmv`   |
| `x-conference/` | `ice`                                                       |

---

## Practical Examples

### HTTP Server: Set Content-Type by file extension

```nim
import std/[mimetypes, os]

var db = newMimetypes()

proc getContentType(filename: string): string =
  let (_, _, ext) = splitFile(filename)
  # ext already contains the dot; getMimetype strips it
  db.getMimetype(ext, default = "application/octet-stream")

echo getContentType("index.html")    # text/html
echo getContentType("style.CSS")     # text/css
echo getContentType("app.bundle.js") # text/javascript
echo getContentType("data.UNKNOWN")  # application/octet-stream
echo getContentType("archive.tar.gz")# application/octet-stream (gz not found)
```

### Validate uploaded files by MIME type

```nim
import std/mimetypes

var db = newMimetypes()

const allowedImages = ["image/jpeg", "image/png", "image/gif", "image/webp"]

proc isAllowedImage(ext: string): bool =
  db.getMimetype(ext) in allowedImages

echo isAllowedImage("jpg")  # true
echo isAllowedImage("png")  # true
echo isAllowedImage("exe")  # false
echo isAllowedImage("SVG")  # false (svg = image/svg+xml, not in the list)
```

### Initialize with extra types for modern formats

```nim
import std/mimetypes

proc newAppMimetypes(): MimeDB =
  result = newMimetypes()
  # Formats absent from the standard database:
  result.register("avif",    "image/avif")
  result.register("heic",    "image/heic")
  result.register("heif",    "image/heif")
  result.register("jxl",     "image/jxl")
  result.register("wasm",    "application/wasm")
  result.register("toml",    "application/toml")
  result.register("graphql", "application/graphql+json")
  result.register("ndjson",  "application/x-ndjson")
  result.register("jsonld",  "application/ld+json")

var db = newAppMimetypes()
echo db.getMimetype("avif")    # image/avif
echo db.getMimetype("wasm")    # application/wasm
echo db.getMimetype("graphql") # application/graphql+json
```

### Determine the media kind of a file

```nim
import std/mimetypes

var db = newMimetypes()

proc mediaKind(filename: string): string =
  let mime = db.getMimetype(filename[filename.rfind('.')+1..^1])
  if mime.startsWith("image/"): "image"
  elif mime.startsWith("video/"): "video"
  elif mime.startsWith("audio/"): "audio"
  elif mime.startsWith("text/"): "text"
  else: "binary"

echo mediaKind("photo.jpg")    # image
echo mediaKind("clip.mp4")     # video
echo mediaKind("song.flac")    # audio
echo mediaKind("readme.txt")   # text
echo mediaKind("archive.zip")  # binary
```

### Use the raw `mimes` constant directly

```nim
import std/mimetypes

# Count extensions per category
var counts: array[0..9, int]
let cats = ["application", "audio", "chemical", "font",
            "image", "message", "model", "text", "video", "x-conference"]

for ext, mime in mimes:
  for i, cat in cats:
    if mime.startsWith(cat & "/"):
      inc counts[i]
      break

for i, cat in cats:
  echo cat, ": ", counts[i]
```

### Derive a file extension from a Content-Type response header

```nim
import std/mimetypes

var db = newMimetypes()

proc extFromContentType(contentType: string): string =
  ## Strips parameters like "; charset=utf-8" then looks up the extension.
  var mime = contentType
  let semi = mime.find(';')
  if semi >= 0:
    mime = mime[0..<semi].strip()
  db.getExt(mime, "bin")

echo extFromContentType("image/png")                # png
echo extFromContentType("text/html; charset=utf-8") # html
echo extFromContentType("application/json")         # json
echo extFromContentType("application/unknown")      # bin
```

### Build a reverse lookup table (all extensions per MIME type)

```nim
import std/[mimetypes, tables]

# Build a one-to-many reverse map: MIME type → all its extensions
var reverseMap = initTable[string, seq[string]]()
for ext, mime in mimes:
  reverseMap.mgetOrPut(mime, @[]).add(ext)

# Query all extensions for a given MIME type
echo reverseMap.getOrDefault("image/jpeg")
# @["jpeg", "jpg", "jpe"]

echo reverseMap.getOrDefault("audio/mpeg")
# @["mpga", "mp2", "mp2a", "mp3", "m2a", "m3a"]

echo reverseMap.getOrDefault("text/plain")
# @["txt", "text", "conf", "def", "list", "log", "in"]
```
