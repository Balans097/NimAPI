# `parsexml` Module Reference

> **Nim Standard Library — `std/parsexml`**
> A high-performance, error-tolerant streaming parser for XML and HTML documents.
> Only UTF-8 encoding is supported.

---

## Table of Contents

1. [Overview & Mental Model](#overview--mental-model)
2. [Types](#types)
   - [XmlEventKind](#xmleventkind)
   - [XmlErrorKind](#xmlerrorkind)
   - [XmlParseOption](#xmlparseoption)
   - [XmlParser](#xmlparser)
3. [Lifecycle Procedures](#lifecycle-procedures)
   - [open](#open)
   - [close](#close)
4. [Driving the Parser](#driving-the-parser)
   - [next](#next)
   - [kind](#kind)
5. [Reading Event Data](#reading-event-data)
   - [charData](#chardata)
   - [elementName](#elementname)
   - [entityName](#entityname)
   - [attrKey](#attrkey)
   - [attrValue](#attrvalue)
   - [piName](#piname)
   - [piRest](#pirest)
6. [Position & Diagnostics](#position--diagnostics)
   - [getLine](#getline)
   - [getColumn](#getcolumn)
   - [getFilename](#getfilename)
   - [errorMsg (xmlError event)](#errormsg-xmlerror-event)
   - [errorMsg (custom text)](#errormsg-custom-text)
   - [errorMsgExpected](#errormsgexpected)
7. [Low-Level / Speed Hacks](#low-level--speed-hacks)
   - [rawData](#rawdata)
   - [rawData2](#rawdata2)
8. [Complete Worked Examples](#complete-worked-examples)

---

## Overview & Mental Model

`parsexml` is a **pull parser** (also called an *event-based* or *streaming* parser). Unlike a DOM parser, it does **not** build an in-memory tree of the document. Instead, you call `next()` in a loop, and the parser advances to the next *event*. You then inspect `kind` to find out what happened, and call the appropriate accessor to read the associated data.

```
┌─────────────────────────────────────────────┐
│  Your code drives the parser with next()     │
│                                              │
│  loop:                                       │
│    next(x)          ← advance one token      │
│    case x.kind      ← what did we hit?       │
│    of xmlCharData:  ← text between tags      │
│      use x.charData                          │
│    of xmlElementStart:                       │
│      use x.elementName                       │
│    of xmlEof: break ← done                   │
└─────────────────────────────────────────────┘
```

Key design decisions you should know:

- **No tag-matching is performed.** The parser will not tell you if a `<div>` is missing its `</div>`. You are responsible for tracking open/close tags if you need that check.
- **Error-tolerant.** The parser is designed to handle "wild HTML" — malformed documents common on the web. It attempts to recover rather than crash.
- **Tags with attributes split into multiple events.** `<a href="x">` produces three consecutive events: `xmlElementOpen` → `xmlAttribute` → `xmlElementClose`. A tag with no attributes produces a single `xmlElementStart` event.

---

## Types

### `XmlEventKind`

An enumeration describing every possible event the parser can emit. After each call to `next()`, the parser's `kind` will be one of these values.

| Value | Triggered by | Example in XML/HTML |
|---|---|---|
| `xmlError` | A parse error occurred | malformed input |
| `xmlEof` | End of input stream reached | — |
| `xmlCharData` | Plain text between tags | `Hello, world!` |
| `xmlWhitespace` | Pure whitespace (only with `reportWhitespace` option) | `  ` |
| `xmlComment` | An XML/HTML comment (only with `reportComments`) | `<!-- note -->` |
| `xmlPI` | Processing instruction | `<?xml version="1.0"?>` |
| `xmlElementStart` | An opening tag **without** attributes | `<br>`, `<p>` |
| `xmlElementEnd` | A closing tag | `</p>` |
| `xmlElementOpen` | Opening of a tag **with** attributes (tag name only, `>` not yet seen) | `<a ` |
| `xmlAttribute` | A key=value attribute pair | `href="https://..."` |
| `xmlElementClose` | The closing `>` of an element that had attributes | `>` |
| `xmlCData` | CDATA section content | `<![CDATA[ raw ]]>` |
| `xmlEntity` | An unrecognised named entity | `&nbsp;` |
| `xmlSpecial` | Any `<!...>` construct that is not CDATA or a comment | `<!DOCTYPE html>` |

> **Important:** The distinction between `xmlElementStart` and `xmlElementOpen`+`xmlAttribute`+`xmlElementClose` is crucial. If you only ever look for `xmlElementStart`, you will miss elements that carry attributes.

---

### `XmlErrorKind`

An enumeration listing the specific parse errors the parser can detect. You do not typically need to inspect this directly — use `errorMsg()` to get a human-readable message.

| Value | Meaning |
|---|---|
| `errNone` | No error |
| `errEndOfCDataExpected` | `]]>` missing at end of CDATA |
| `errNameExpected` | A tag or attribute name was expected |
| `errSemicolonExpected` | `;` missing after entity reference |
| `errQmGtExpected` | `?>` missing at end of processing instruction |
| `errGtExpected` | `>` missing at end of tag |
| `errEqExpected` | `=` missing between attribute name and value |
| `errQuoteExpected` | `"` or `'` missing around attribute value |
| `errEndOfCommentExpected` | `-->` missing at end of comment |
| `errAttributeValueExpected` | Attribute value is empty or missing |

---

### `XmlParseOption`

A set of flags you can pass to `open()` to control parser behaviour.

| Flag | Effect |
|---|---|
| `reportWhitespace` | Emit `xmlWhitespace` events for runs of whitespace. By default, whitespace between tags is silently skipped. |
| `reportComments` | Emit `xmlComment` events and populate `charData` with the comment text. By default, comments are silently discarded. |
| `allowUnquotedAttribs` | Accept attribute values without surrounding quotes, as often seen in old or hand-written HTML (e.g. `<td width=100>`). |
| `allowEmptyAttribs` | Accept attributes with no value at all (e.g. `<input disabled>`). |

---

### `XmlParser`

The parser object. Declare it as a `var`, initialise it with `open()`, drive it with `next()`, and release resources with `close()`.

```nim
var x: XmlParser
open(x, stream, "myfile.xml")
# ... use x ...
close(x)
```

You should treat `XmlParser` as an opaque type and only access its state through the documented procedures and templates.

---

## Lifecycle Procedures

### `open`

```nim
proc open*(my: var XmlParser; input: Stream; filename: string;
           options: set[XmlParseOption] = {})
```

**What it does:** Initialises the parser and binds it to an input `Stream`. Must be called before any other procedure. The `filename` argument is used only for error message formatting — it does not need to be a real path.

**Parameters:**
- `input` — Any `Stream` object (file stream, string stream, etc.).
- `filename` — A label used in error messages (e.g. the actual filename, or `"<stdin>"`).
- `options` — A set of `XmlParseOption` flags (default: none).

```nim
import std/[streams, parsexml]

# Parse from a file
var s = newFileStream("data.xml", fmRead)
var x: XmlParser
open(x, s, "data.xml")

# Parse from a string in memory
var s2 = newStringStream("<root><item>hello</item></root>")
var x2: XmlParser
open(x2, s2, "<string>")

# Enable HTML-friendly options
var s3 = newFileStream("page.html", fmRead)
var x3: XmlParser
open(x3, s3, "page.html", {allowUnquotedAttribs, allowEmptyAttribs})
```

---

### `close`

```nim
proc close*(my: var XmlParser)
```

**What it does:** Closes the parser and releases the underlying stream. Always call `close()` when you are finished, even if you stopped parsing early (e.g. after finding what you needed).

```nim
x.close()
```

---

## Driving the Parser

### `next`

```nim
proc next*(my: var XmlParser)
```

**What it does:** Advances the parser to the next event. This is the engine of the pull parser. Each call moves the parser forward by exactly one token. After the call, inspect `my.kind` to learn what was parsed, and use the appropriate accessor to retrieve the associated data.

`next()` silently discards comments and whitespace unless the corresponding options (`reportComments`, `reportWhitespace`) were passed to `open()`. The very first `<?xml ... ?>` processing instruction is also silently skipped.

```nim
var s = newStringStream("<msg lang=\"en\">Hello</msg>")
var x: XmlParser
open(x, s, "")

x.next()  # → xmlElementOpen  (tag has an attribute)
echo x.elementName  # "msg"

x.next()  # → xmlAttribute
echo x.attrKey    # "lang"
echo x.attrValue  # "en"

x.next()  # → xmlElementClose  (the '>' of <msg lang="en">)

x.next()  # → xmlCharData
echo x.charData  # "Hello"

x.next()  # → xmlElementEnd
echo x.elementName  # "msg"

x.next()  # → xmlEof

x.close()
```

---

### `kind`

```nim
proc kind*(my: XmlParser): XmlEventKind
```

**What it does:** Returns the type of the most recently parsed event. This is the central discriminator — almost every use of the parser involves checking `kind` first.

```nim
x.next()
case x.kind
of xmlElementStart: echo "open tag: ", x.elementName
of xmlElementEnd:   echo "close tag: ", x.elementName
of xmlCharData:     echo "text: ", x.charData
of xmlEof:          echo "done"
else:               discard
```

---

## Reading Event Data

These templates and procedures let you read the data associated with the current event. **Each accessor is only valid when `kind` matches the documented events.** In debug builds, calling an accessor with the wrong `kind` raises an assertion error. In release builds the behaviour is undefined.

---

### `charData`

```nim
template charData*(my: XmlParser): string
```

**Valid for:** `xmlCharData`, `xmlWhitespace`, `xmlComment`, `xmlCData`, `xmlSpecial`

**What it does:** Returns the textual content of the current event. For `xmlCharData` this is the plain text between tags. For `xmlComment` it is the comment text (without the `<!--` / `-->` delimiters). For `xmlCData` it is the raw data inside `<![CDATA[...]]>`. For `xmlSpecial` it is the raw content of `<!...>` constructs like DOCTYPE declarations.

```nim
# Input: <p>Hello <b>world</b></p>
# After next() lands on xmlCharData:
echo x.charData  # "Hello "

# Input: <!-- this is a comment -->
# (parser opened with reportComments option)
# After next() lands on xmlComment:
echo x.charData  # " this is a comment "

# Input: <![CDATA[<raw & text>]]>
# After next() lands on xmlCData:
echo x.charData  # "<raw & text>"
```

---

### `elementName`

```nim
template elementName*(my: XmlParser): string
```

**Valid for:** `xmlElementStart`, `xmlElementEnd`, `xmlElementOpen`

**What it does:** Returns the name of the element — the tag name, without angle brackets. For `xmlElementStart` and `xmlElementEnd` this is the full tag name. For `xmlElementOpen` it is the tag name of an element whose attribute list has not yet been fully parsed.

```nim
# Input: <title>Nim</title>
x.next()  # xmlElementStart
echo x.elementName  # "title"

x.next()  # xmlCharData → "Nim"

x.next()  # xmlElementEnd
echo x.elementName  # "title"
```

```nim
# Input: <a href="https://nim-lang.org">
x.next()  # xmlElementOpen (tag has attributes)
echo x.elementName  # "a"
# don't call charData here — wrong event type!
```

---

### `entityName`

```nim
template entityName*(my: XmlParser): string
```

**Valid for:** `xmlEntity`

**What it does:** Returns the name of an unrecognised named entity — the text between `&` and `;`. The standard XML entities (`&lt;`, `&gt;`, `&amp;`, `&apos;`, `&quot;`) and numeric character references (`&#65;`, `&#x41;`) are decoded silently and appear as `xmlCharData`, never as `xmlEntity`. Only truly unknown entities (like `&nbsp;`, `&copy;`) surface as `xmlEntity` events.

```nim
# Input: <p>Copyright &copy; 2024</p>
# After landing on xmlEntity:
echo x.entityName  # "copy"
# You can then emit "&copy;" or look it up in an HTML entity table.
```

---

### `attrKey`

```nim
template attrKey*(my: XmlParser): string
```

**Valid for:** `xmlAttribute`

**What it does:** Returns the attribute name (the left side of `key="value"`).

```nim
# Input: <img src="photo.png" alt="A photo">
x.next()  # xmlElementOpen → "img"
x.next()  # xmlAttribute
echo x.attrKey    # "src"
echo x.attrValue  # "photo.png"
x.next()  # xmlAttribute
echo x.attrKey    # "alt"
echo x.attrValue  # "A photo"
x.next()  # xmlElementClose
```

---

### `attrValue`

```nim
template attrValue*(my: XmlParser): string
```

**Valid for:** `xmlAttribute`

**What it does:** Returns the attribute value (the right side of `key="value"`), with surrounding quotes stripped and standard entities decoded. Always read `attrKey` and `attrValue` together on the same `xmlAttribute` event.

*(See the example for `attrKey` above.)*

---

### `piName`

```nim
template piName*(my: XmlParser): string
```

**Valid for:** `xmlPI`

**What it does:** Returns the name of a processing instruction — the identifier immediately after `<?`. The standard `<?xml ... ?>` declaration at the top of a document is silently consumed by `next()` and never surfaces as an event.

```nim
# Input: <?xml-stylesheet type="text/css" href="style.css"?>
# After next() lands on xmlPI:
echo x.piName  # "xml-stylesheet"
echo x.piRest  # " type=\"text/css\" href=\"style.css\""
```

---

### `piRest`

```nim
template piRest*(my: XmlParser): string
```

**Valid for:** `xmlPI`

**What it does:** Returns everything after the processing instruction name, up to the closing `?>`. This is the raw, unparsed remainder of the instruction — you must parse it yourself if you need structured data from it.

*(See the example for `piName` above.)*

---

## Position & Diagnostics

### `getLine`

```nim
proc getLine*(my: XmlParser): int
```

**What it does:** Returns the 1-based line number in the input where the parser is currently positioned. Useful for error messages.

---

### `getColumn`

```nim
proc getColumn*(my: XmlParser): int
```

**What it does:** Returns the 0-based column number in the input where the parser is currently positioned.

---

### `getFilename`

```nim
proc getFilename*(my: XmlParser): string
```

**What it does:** Returns the filename string that was passed to `open()`. This is used internally by `errorMsg()` to produce well-formatted error messages.

---

### `errorMsg` (xmlError event)

```nim
proc errorMsg*(my: XmlParser): string
```

**Valid for:** `xmlError` events only.

**What it does:** Returns a human-readable error message describing the parse error that caused the `xmlError` event. The message is formatted as:

```
filename(line, column) Error: description
```

This is the primary way to report parser errors. Always call this when you encounter `xmlError` — do not try to format error messages manually.

```nim
while true:
  x.next()
  case x.kind
  of xmlError:
    echo x.errorMsg()   # e.g. "data.xml(12, 5) Error: '>' expected"
    x.next()            # try to recover and continue
  of xmlEof: break
  else: discard
```

---

### `errorMsg` (custom text)

```nim
proc errorMsg*(my: XmlParser; msg: string): string
```

**What it does:** Formats an arbitrary `msg` string into the same `filename(line, column) Error: msg` format used by the parser itself. Use this when you detect a semantic error in client code (e.g. unexpected nesting) and want it to look consistent with parser errors.

```nim
if x.elementName != "root":
  echo x.errorMsg("expected <root> as document root")
  # → "schema.xml(1, 1) Error: expected <root> as document root"
```

---

### `errorMsgExpected`

```nim
proc errorMsgExpected*(my: XmlParser; tag: string): string
```

**What it does:** A convenience wrapper around `errorMsg`. Formats a message of the form `<tag> expected` using the standard error format. Intended specifically for the case where you expected a particular closing tag but found something else — a situation the parser cannot check for you.

```nim
x.next()  # we expect </title> here
if x.kind != xmlElementEnd or x.elementName != "title":
  echo x.errorMsgExpected("/title")
  # → "index.html(8, 3) Error: </title> expected"
```

---

## Low-Level / Speed Hacks

These procedures expose the parser's internal string buffers directly by reference. They exist purely for performance-sensitive code that wants to avoid string copies. Under normal circumstances, prefer the named template accessors.

### `rawData`

```nim
proc rawData*(my: var XmlParser): lent string
```

**What it does:** Returns a reference to the parser's primary internal string buffer (`a`). This holds the same value as `charData`, `elementName`, `entityName`, `attrKey`, and `piName` depending on context. Because this is a `lent` (borrowed) reference, you must not hold onto it past the next call to `next()`.

---

### `rawData2`

```nim
proc rawData2*(my: var XmlParser): lent string
```

**What it does:** Returns a reference to the parser's secondary internal string buffer (`b`). This holds the same value as `attrValue` and `piRest` depending on context. Same lifetime caveats as `rawData`.

---

## Complete Worked Examples

### Example 1: Extract all text from an XML document

```nim
import std/[streams, parsexml]

var s = newStringStream("""
  <doc>
    <section>First section text.</section>
    <!-- ignored comment -->
    <section>Second section text.</section>
  </doc>
""")

var x: XmlParser
open(x, s, "<string>")

while true:
  x.next()
  case x.kind
  of xmlCharData:
    let text = x.charData.strip()
    if text.len > 0:
      echo text
  of xmlEof: break
  of xmlError: echo x.errorMsg()
  else: discard

x.close()
# Output:
# First section text.
# Second section text.
```

---

### Example 2: Collect all attributes of a specific element

This example collects every attribute of every `<meta>` tag in an HTML document — a common need when scraping HTML.

```nim
import std/[streams, parsexml, strutils, tables]

proc collectMetaTags(html: string): seq[Table[string, string]] =
  var s = newStringStream(html)
  var x: XmlParser
  open(x, s, "<html>", {allowUnquotedAttribs, allowEmptyAttribs})

  var inMeta = false
  var current: Table[string, string]

  while true:
    x.next()
    case x.kind
    of xmlElementOpen:
      if cmpIgnoreCase(x.elementName, "meta") == 0:
        inMeta = true
        current = initTable[string, string]()
    of xmlAttribute:
      if inMeta:
        current[x.attrKey] = x.attrValue
    of xmlElementClose:
      if inMeta:
        result.add(current)
        inMeta = false
    of xmlEof: break
    else: discard

  x.close()

let html = """
  <html><head>
    <meta name="description" content="A test page">
    <meta charset="UTF-8">
  </head></html>
"""

for tag in collectMetaTags(html):
  echo tag
```

---

### Example 3: Error-resilient parsing loop (recommended pattern)

```nim
import std/[streams, parsexml]

var s = newFileStream("data.xml", fmRead)
var x: XmlParser
open(x, s, "data.xml")

while true:
  x.next()
  case x.kind
  of xmlEof:
    break                        # clean exit
  of xmlError:
    stderr.writeLine x.errorMsg()  # report and try to continue
  of xmlElementStart:
    echo "START: ", x.elementName
  of xmlElementEnd:
    echo "END:   ", x.elementName
  of xmlElementOpen:
    echo "OPEN:  ", x.elementName
  of xmlAttribute:
    echo "  ATTR: ", x.attrKey, " = ", x.attrValue
  of xmlElementClose:
    echo "  >"
  of xmlCharData:
    let t = x.charData.strip()
    if t.len > 0: echo "TEXT:  ", t
  of xmlCData:
    echo "CDATA: ", x.charData
  of xmlComment:
    discard  # ignored unless reportComments was set
  of xmlEntity:
    echo "ENTITY: &", x.entityName, ";"
  of xmlPI:
    echo "PI: ", x.piName, " | ", x.piRest
  of xmlSpecial:
    echo "SPECIAL: ", x.charData
  of xmlWhitespace:
    discard  # ignored unless reportWhitespace was set

x.close()
```

---

### Example 4: Verify a closing tag (client-side tag matching)

Since the parser does not check tag pairing, here is a simple pattern for doing it yourself:

```nim
import std/[streams, parsexml]

proc parseTitle(x: var XmlParser): string =
  # Called when we have just consumed xmlElementStart for "title"
  x.next()
  while x.kind == xmlCharData:
    result.add(x.charData)
    x.next()
  if x.kind == xmlElementEnd and x.elementName == "title":
    discard  # all good
  else:
    echo x.errorMsgExpected("/title")

var s = newStringStream("<html><head><title>Hello World</title></head></html>")
var x: XmlParser
open(x, s, "")

while true:
  x.next()
  case x.kind
  of xmlElementStart:
    if x.elementName == "title":
      echo "Title: ", parseTitle(x)
  of xmlEof: break
  else: discard

x.close()
# Output:
# Title: Hello World
```
