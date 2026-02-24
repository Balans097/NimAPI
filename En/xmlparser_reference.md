# `xmlparser` Module Reference

> **Module:** `std/xmlparser`  
> **Language:** Nim  
> **Purpose:** Parses an XML document from a string, stream, or file and builds an in-memory `XmlNode` tree that can be traversed and queried with `std/xmltree`.

---

## Overview

The `xmlparser` module is the bridge between raw XML text and Nim's structured `XmlNode` tree. Its job is simple but essential: take XML from wherever it lives (a string literal, a network stream, a file on disk) and hand back a tree of nodes you can navigate programmatically.

### Dependencies you should know about

| Module | Role |
|--------|------|
| `std/streams` | Abstracts the data source (file, string, network…) |
| `std/parsexml` | Low-level tokeniser; produces events like "element opened", "attribute seen" |
| `std/xmltree` | Defines `XmlNode` and the tree structure you get back |
| `std/strtabs` | Used internally to store element attributes as a string table |

### Two philosophies for error handling

Every public function comes in **two variants** that differ only in how errors are reported:

- **`errors: var seq[string]` variant** — soft errors. Parsing problems are collected into a sequence of human-readable messages that the caller inspects afterward. Parsing continues as far as possible; you get a (potentially partial) tree back even when errors occur.
- **Raising variant** — strict errors. If any errors accumulate, `raiseInvalidXml` is called immediately after parsing, throwing an `XmlError` exception. The partial tree is discarded.

Choose the soft variant when you want to show diagnostics, tolerate minor malformedness, or log problems without crashing. Choose the raising variant when invalid XML is always a programming error or a fatal condition.

---

## `XmlError`

```nim
type XmlError* = object of ValueError
  errors*: seq[string]
```

### What it is

`XmlError` is the exception type raised whenever the strict (non-`errors` parameter) variants of `parseXml` or `loadXml` encounter malformed XML. It inherits from `ValueError`, which in turn inherits from `CatchableError`, so it participates in Nim's normal `try/except` mechanism.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `msg` | `string` | (inherited) The first error message — what you see in a default exception handler |
| `errors` | `seq[string]` | **All** error messages collected during the parse run, not just the first one |

### Why `errors` is a sequence

A real XML document can contain multiple problems: a missing closing tag, a malformed attribute, an unexpected character. The `XmlError.errors` sequence preserves every issue found, so a caller catching the exception can present a complete diagnostic report rather than just the first failure.

### Example

```nim
import std/xmlparser

try:
  let tree = parseXml("<root><unclosed>")
except XmlError as e:
  echo "First error: ", e.msg
  echo "All errors:"
  for err in e.errors:
    echo "  - ", err
```

---

## `parseXml` (stream + filename + errors)

```nim
proc parseXml*(s: Stream, filename: string,
               errors: var seq[string],
               options: set[XmlParseOption] = {reportComments}): XmlNode
```

### What it does

This is the **foundational overload** — all other `parseXml` and `loadXml` variants ultimately delegate to this one. It opens an `XmlParser` on the given stream, drives it token by token, builds an `XmlNode` tree, and appends any problems it encounters to `errors`. It always closes the parser when done.

The `filename` string is purely diagnostic: it appears in error messages (like `"myfile.xml(12, 5) error: …"`), helping the caller pinpoint problems in the original source. When you parse a string rather than a file, the other overloads pass `"unknown_xml_doc"` here.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `s` | `Stream` | Any readable stream: `FileStream`, `StringStream`, a socket stream, etc. |
| `filename` | `string` | Name used in error messages to identify the source |
| `errors` | `var seq[string]` | Accumulates human-readable error strings; pre-existing content is preserved |
| `options` | `set[XmlParseOption]` | Parser options from `std/parsexml`. Default: `{reportComments}` |

### Return value

The root `XmlNode` of the parsed tree, or `nil` if no root element was found at all (for example, an empty stream).

### `XmlParseOption` options

These come from `std/parsexml`. The most relevant ones:

| Option | Effect |
|--------|--------|
| `reportComments` | Include comment nodes in the tree (default, on) |
| `reportWhitespace` | Include whitespace-only text nodes |
| `allowUnquotedAttribs` | Accept attributes without quotes (lenient mode) |
| `allowParentClose` | Accept tags closed by a parent element (lenient mode) |

### Example

```nim
import std/[xmlparser, streams]

let xml = """
  <library>
    <book id="1">
      <title>Moby Dick</title>
    </book>
  </library>
"""

var errors: seq[string] = @[]
let stream = newStringStream(xml)
let tree = parseXml(stream, "inline_doc", errors)

if errors.len > 0:
  for e in errors: echo "Parse error: ", e
else:
  echo tree.tag        # "library"
```

---

## `parseXml` (stream, raising)

```nim
proc parseXml*(s: Stream,
               options: set[XmlParseOption] = {reportComments}): XmlNode
```

### What it does

A convenience wrapper around the foundational overload. Parses from a stream without requiring the caller to manage an error sequence. Instead, if any errors accumulate, it raises `XmlError`. The filename passed internally is `"unknown_xml_doc"`.

Use this when you control the stream directly and want exceptions for bad XML.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `s` | `Stream` | Any readable stream |
| `options` | `set[XmlParseOption]` | Parser options. Default: `{reportComments}` |

### Return value

The root `XmlNode`, or raises `XmlError`.

### Example

```nim
import std/[xmlparser, streams]

let s = newStringStream("<note><body>Hello</body></note>")
let tree = parseXml(s)   # raises XmlError if malformed
echo tree[0].innerText   # "Hello"
```

---

## `parseXml` (string, raising)

```nim
proc parseXml*(str: string,
               options: set[XmlParseOption] = {reportComments}): XmlNode
```

### What it does

The most ergonomic overload: takes a plain Nim `string` containing XML and returns the parsed tree, raising `XmlError` on any error. Internally it wraps the string in a `StringStream` and delegates to the stream overload.

This is the right choice for parsing XML that lives in string literals, API responses already loaded into memory, or template output — situations where opening a file or managing a stream would be unnecessary ceremony.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `str` | `string` | The XML content as a Nim string |
| `options` | `set[XmlParseOption]` | Parser options. Default: `{reportComments}` |

### Return value

The root `XmlNode`, or raises `XmlError`.

### Example

```nim
import std/xmlparser

# Simplest possible use
let tree = parseXml("<config><timeout>30</timeout></config>")
echo tree.tag                    # "config"
echo tree[0].tag                 # "timeout"
echo tree[0].innerText           # "30"

# With attributes
let xml = """<user id="42" role="admin"/>"""
let user = parseXml(xml)
echo user.attr("id")             # "42"
echo user.attr("role")           # "admin"

# Error handling
try:
  let bad = parseXml("<broken>")
except XmlError as e:
  echo "XML error: ", e.msg
```

---

## `loadXml` (path + errors)

```nim
proc loadXml*(path: string,
              errors: var seq[string],
              options: set[XmlParseOption] = {reportComments}): XmlNode
```

### What it does

Opens the file at `path` for reading and parses it as XML, collecting any errors into the provided sequence. This is the file-based counterpart to the stream+errors overload of `parseXml`.

If the file cannot be opened (it doesn't exist, permissions are wrong, etc.), an `IOError` is raised immediately — before any XML parsing begins. This is distinct from `XmlError`: file I/O problems and XML validity problems are reported through different mechanisms.

The filename in error messages will be the actual `path` string, giving you accurate source locations.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | `string` | Filesystem path to the XML file |
| `errors` | `var seq[string]` | Accumulates error messages; safe to reuse across multiple calls |
| `options` | `set[XmlParseOption]` | Parser options. Default: `{reportComments}` |

### Return value

The root `XmlNode`, or `nil` if the file is empty or has no root element. Errors are appended to `errors`.

### Example

```nim
import std/xmlparser

var errors: seq[string] = @[]
let tree = loadXml("config.xml", errors)

if errors.len > 0:
  echo "Loaded with warnings:"
  for e in errors: echo "  ", e
  # decide whether to proceed despite errors
else:
  echo "Loaded cleanly: ", tree.tag
```

---

## `loadXml` (path, raising)

```nim
proc loadXml*(path: string,
              options: set[XmlParseOption] = {reportComments}): XmlNode
```

### What it does

The strict variant of file loading. Reads and parses the XML file at `path`, and raises `XmlError` if any parsing errors occur. Like the soft variant, it raises `IOError` if the file cannot be opened.

This is the right choice when loading a configuration file, data file, or any resource where malformed XML means the program simply cannot continue.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | `string` | Filesystem path to the XML file |
| `options` | `set[XmlParseOption]` | Parser options. Default: `{reportComments}` |

### Return value

The root `XmlNode`, or raises `XmlError` / `IOError`.

### Example

```nim
import std/xmlparser

# Load a required config file — bad XML is fatal
try:
  let config = loadXml("settings.xml")
  echo "Root tag: ", config.tag
except IOError as e:
  echo "File not found or unreadable: ", e.msg
  quit(1)
except XmlError as e:
  echo "Invalid XML in settings.xml:"
  for err in e.errors:
    echo "  ", err
  quit(1)
```

---

## Choosing the right overload

The module exposes four public functions arranged along two axes:

```
                    Source
                 ┌──────────┬──────────┐
                 │  string  │   file   │
   ┌─────────────┼──────────┼──────────┤
E  │ soft errors │parseXml* │ loadXml* │   * stream+filename+errors variant
r  │  (seq)      │(stream)  │          │     for parseXml
r  ├─────────────┼──────────┼──────────┤
o  │   raises    │parseXml  │ loadXml  │
r  │  XmlError   │(string)  │          │
s  └─────────────┴──────────┴──────────┘
```

**Quick decision guide:**

- Parsing a string literal or in-memory API response → `parseXml(str)`
- Loading a required config or data file → `loadXml(path)`
- Want to show all warnings but continue → use the `errors: var seq[string]` variant
- XML error = fatal → use the raising variant (no `errors` parameter)

---

## What nodes does the parser produce?

The parser recognises all standard XML constructs and maps them to `XmlNode` types from `std/xmltree`:

| XML construct | Example | `XmlNode` produced |
|--------------|---------|-------------------|
| Element with children | `<tag>…</tag>` | `newElement("tag")` |
| Self-closing element | `<br/>` | `newElement("br")` |
| Text content | `hello world` | `newText("hello world")` |
| Comment | `<!-- note -->` | `newComment(" note ")` |
| CDATA section | `<![CDATA[…]]>` | `newCData("…")` |
| Entity reference | `&amp;` | `newEntity("amp")` |
| Attributes | `id="42"` | stored in `node.attrs` string table |
| Processing instructions | `<?xml … ?>` | **silently ignored** |
| XML special (`<!…>`) | `<!DOCTYPE …>` | **silently ignored** |

> **Note:** Processing instructions (`<?…?>`) and special declarations (`<!DOCTYPE …>`) are currently **not** represented in the output tree — they are silently skipped by the parser.

---

## Notes and gotchas

- **`nil` root is possible.** If the stream or file is empty, or contains only whitespace/comments/PIs with no actual element, `parseXml` and `loadXml` return `nil`. Always guard against this before accessing `.tag` or children.
- **Comments are included by default.** The default option `{reportComments}` means comment nodes appear in the tree. Pass `{}` or omit `reportComments` from options if you want a comment-free tree.
- **Partial trees on error.** The soft (`errors` parameter) variants may return a partial tree alongside errors — nodes parsed before the error was hit are retained. The strict (raising) variants still build the partial tree internally but discard it when the exception is raised.
- **Entities are preserved, not resolved.** `&amp;` becomes an entity node, not the character `&`. Use `std/xmltree` utilities to work with entity content.
- **File stream is closed automatically.** `loadXml` opens a `FileStream` internally; the parser closes it on completion (or error).
