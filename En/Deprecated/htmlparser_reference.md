# `htmlparser` Module Reference (Nim)

> Nim standard library module for parsing real-world HTML documents.  
> Parses lenient, imperfect HTML and returns an `XmlNode` tree.
>
> ⚠️ **Deprecated module.** This module is now marked `deprecated`.  
> Recommended replacement: `nimble install htmlparser`, then import `pkg/htmlparser`.

---

## Table of Contents

1. [Overview](#overview)
2. [Types and Constants](#types-and-constants)
3. [Parsing HTML](#parsing-html)
   - [parseHtml (Stream + errors)](#parsehtmls-stream-filename-string-errors-var-seqstring-xmlnode)
   - [parseHtml (Stream)](#parsehtmls-stream-xmlnode)
   - [parseHtml (string)](#parsehtmlhtml-string-xmlnode)
   - [loadHtml (file + errors)](#loadhtmlpath-string-errors-var-seqstring-xmlnode)
   - [loadHtml (file)](#loadhtmlpath-string-xmlnode)
4. [Working with Tags](#working-with-tags)
   - [htmlTag (XmlNode)](#htmltagnxmlnode-htmltag)
   - [htmlTag (string)](#htmltags-string-htmltag)
5. [Working with HTML Entities](#working-with-html-entities)
   - [runeToEntity](#runetoentityrune-rune-string)
   - [entityToRune](#entitytoruneentity-string-rune)
   - [entityToUtf8](#entitytoutf8entity-string-string)
6. [The `tagToStr` Constant](#the-tagtostr-constant)
7. [The `HtmlTag` Enum](#the-htmltag-enum)
8. [Practical Examples](#practical-examples)
9. [Quick Reference](#quick-reference)

---

## Overview

`htmlparser` parses HTML documents — including malformed, real-world ones — and returns an `XmlNode` tree from `std/xmltree`. All tags in the resulting tree are **lowercase**.

The resulting `XmlNode` uses the `clientData` field internally. Client code must not modify it.

```nim
import std/htmlparser
import std/xmltree

let html = parseHtml("<html><body><p>Hello!</p></body></html>")
echo html  # prints normalised XML
```

---

## Types and Constants

### `HtmlTag`

```nim
type HtmlTag* = enum
  tagUnknown, tagA, tagAbbr, tagAcronym, tagAddress, ...
```

An enumeration of every supported HTML tag, in alphabetical order. Both current and deprecated tags are included (deprecated ones are noted in the docs).

Special value:
- `tagUnknown` — an unrecognised or custom tag

### `tagToStr`

```nim
const tagToStr* = ["a", "abbr", "acronym", ...]
```

An array of tag name strings parallel to the `HtmlTag` enum (starting from `tagA`). Use it to convert an enum value back to its lowercase tag name string.

```nim
echo tagToStr[tagDiv.ord - 1]  # "div"
```

> Index 0 corresponds to `tagA`, not `tagUnknown`.

---

## Parsing HTML

### `parseHtml(s: Stream, filename: string, errors: var seq[string]): XmlNode`

```nim
proc parseHtml*(s: Stream, filename: string,
                errors: var seq[string]): XmlNode
```

Parses HTML from stream `s`. The `filename` string is used only in error messages. Every parsing error is appended to `errors`.

**Parameters:**
- `s` — input `Stream` containing HTML
- `filename` — name used in diagnostics (e.g. the file path)
- `errors` — mutable sequence that collects error messages

**Returns:** the root `XmlNode` of the document.

```nim
import std/[htmlparser, xmltree, streams]

var errors: seq[string] = @[]
let stream = newStringStream("<html><body><p>Hello</p></body></html>")
let doc = parseHtml(stream, "test.html", errors)

if errors.len > 0:
  for e in errors:
    echo "Error: ", e
else:
  echo "Parse successful"

echo doc
```

---

### `parseHtml(s: Stream): XmlNode`

```nim
proc parseHtml*(s: Stream): XmlNode
```

Simplified overload: parses HTML from stream `s`, silently ignoring all errors. Uses `"unknown_html_doc"` as the internal filename.

```nim
import std/[htmlparser, streams]

let s = newStringStream("<div class='box'><p>Hello</p></div>")
let doc = parseHtml(s)
echo doc
```

---

### `parseHtml(html: string): XmlNode`

```nim
proc parseHtml*(html: string): XmlNode
```

The most convenient overload: accepts an HTML string directly. All errors are silently ignored.

```nim
import std/htmlparser

let doc = parseHtml("""
  <html>
    <head><title>Test</title></head>
    <body>
      <h1>Heading</h1>
      <p>A paragraph with <b>bold</b> text.</p>
    </body>
  </html>
""")

echo doc
```

---

### `loadHtml(path: string, errors: var seq[string]): XmlNode`

```nim
proc loadHtml*(path: string, errors: var seq[string]): XmlNode
```

Reads and parses an HTML file at `path`. Parsing errors are appended to `errors`. Raises `IOError` if the file cannot be opened.

**Parameters:**
- `path` — path to the HTML file
- `errors` — mutable sequence for error collection

```nim
import std/htmlparser

var errors: seq[string] = @[]
let doc = loadHtml("index.html", errors)

for e in errors:
  echo "Warning: ", e

echo "Document loaded"
```

---

### `loadHtml(path: string): XmlNode`

```nim
proc loadHtml*(path: string): XmlNode
```

Simplified version: loads and parses an HTML file, ignoring all errors.

```nim
import std/htmlparser

let doc = loadHtml("page.html")
echo doc
```

---

## Working with Tags

### `htmlTag(n: XmlNode): HtmlTag`

```nim
proc htmlTag*(n: XmlNode): HtmlTag
```

Returns the HTML tag of node `n` as an `HtmlTag` enum value. The result is cached in the node's `clientData` field for fast repeated access.

```nim
import std/[htmlparser, xmltree]

let doc = parseHtml("<div><p>Text</p></div>")
for node in doc:
  if node.kind == xnElement:
    case node.htmlTag
    of tagDiv: echo "Found a div"
    of tagP:   echo "Found a paragraph"
    else: discard
```

---

### `htmlTag(s: string): HtmlTag`

```nim
proc htmlTag*(s: string): HtmlTag
```

Converts the string `s` to an `HtmlTag` enum value. Case-insensitive. Returns `tagUnknown` if the tag is not recognised.

```nim
import std/htmlparser

echo htmlTag("div")     # tagDiv
echo htmlTag("DIV")     # tagDiv  (case-insensitive)
echo htmlTag("p")       # tagP
echo htmlTag("custom")  # tagUnknown
```

---

## Working with HTML Entities

### `runeToEntity(rune: Rune): string`

```nim
proc runeToEntity*(rune: Rune): string
```

Converts a Unicode `Rune` to its numeric HTML entity string of the form `#N`.  
Returns an empty string if `rune.ord <= 0`.

```nim
import std/[htmlparser, unicode]

echo runeToEntity(Rune(0))         # ""
echo runeToEntity(Rune(-1))        # ""
echo runeToEntity("Ü".runeAt(0))   # "#220"
echo runeToEntity("∈".runeAt(0))   # "#8712"
echo runeToEntity("©".runeAt(0))   # "#169"
```

**Use case:** escaping characters when building raw HTML strings.

```nim
import std/[htmlparser, unicode]

proc escapeChar(c: char): string =
  let entity = runeToEntity(Rune(c.ord))
  if entity.len > 0: "&" & entity & ";"
  else: $c
```

---

### `entityToRune(entity: string): Rune`

```nim
proc entityToRune*(entity: string): Rune
```

Converts an HTML entity name or numeric code to a Unicode `Rune`.  
Returns `Rune(0)` if the entity is unknown or the input is malformed.

**Accepted input formats** (no surrounding `&` or `;`):
- Named: `"Uuml"`, `"gt"`, `"amp"`, `"quest"`, etc.
- Decimal: `"#220"`, `"#63"`
- Hexadecimal: `"#x000DC"`, `"#X3a3"` (the `x` prefix is case-insensitive)

```nim
import std/[htmlparser, unicode]

echo entityToRune("")        # Rune(0) — too short
echo entityToRune("a")       # Rune(0) — unknown
echo entityToRune("gt")      # Rune for '>'
echo entityToRune("Uuml")    # Rune for 'Ü'
echo entityToRune("quest")   # Rune for '?'
echo entityToRune("#63")     # Rune for '?'
echo entityToRune("#x003F")  # Rune for '?'
```

---

### `entityToUtf8(entity: string): string`

```nim
proc entityToUtf8*(entity: string): string
```

Converts an HTML entity to its UTF-8 string representation. A convenience wrapper over `entityToRune` for when you need a string rather than a code point.  
Returns `""` if the entity is unknown.

> **Note:** The HTML parser already converts entities to UTF-8 automatically during parsing. This function is for manual decoding outside the parser.

```nim
import std/htmlparser

echo entityToUtf8("")       # ""
echo entityToUtf8("a")      # ""
echo entityToUtf8("gt")     # ">"
echo entityToUtf8("Uuml")   # "Ü"
echo entityToUtf8("quest")  # "?"
echo entityToUtf8("#63")    # "?"
echo entityToUtf8("Sigma")  # "Σ"
echo entityToUtf8("#931")   # "Σ"
echo entityToUtf8("#x3A3")  # "Σ"
echo entityToUtf8("#X3a3")  # "Σ"
```

---

## The `HtmlTag` Enum

Complete list of `HtmlTag` values (alphabetical order):

| Value | HTML tag | Note |
|-------|----------|------|
| `tagUnknown` | — | Unrecognised tag |
| `tagA` | `<a>` | |
| `tagAbbr` | `<abbr>` | Deprecated |
| `tagAcronym` | `<acronym>` | |
| `tagAddress` | `<address>` | |
| `tagApplet` | `<applet>` | Deprecated |
| `tagArea` | `<area>` | |
| `tagArticle` | `<article>` | |
| `tagAside` | `<aside>` | |
| `tagAudio` | `<audio>` | |
| `tagB` | `<b>` | |
| `tagBase` | `<base>` | |
| `tagBdi` | `<bdi>` | |
| `tagBdo` | `<bdo>` | Deprecated |
| `tagBasefont` | `<basefont>` | Deprecated |
| `tagBig` | `<big>` | |
| `tagBlockquote` | `<blockquote>` | |
| `tagBody` | `<body>` | |
| `tagBr` | `<br>` | |
| `tagButton` | `<button>` | |
| `tagCanvas` | `<canvas>` | |
| `tagCaption` | `<caption>` | |
| `tagCenter` | `<center>` | Deprecated |
| `tagCite` | `<cite>` | |
| `tagCode` | `<code>` | |
| `tagCol` | `<col>` | |
| `tagColgroup` | `<colgroup>` | |
| `tagCommand` | `<command>` | |
| `tagDatalist` | `<datalist>` | |
| `tagDd` | `<dd>` | |
| `tagDel` | `<del>` | |
| `tagDetails` | `<details>` | |
| `tagDfn` | `<dfn>` | |
| `tagDialog` | `<dialog>` | |
| `tagDiv` | `<div>` | |
| `tagDir` | `<dir>` | Deprecated |
| `tagDl` | `<dl>` | |
| `tagDt` | `<dt>` | |
| `tagEm` | `<em>` | |
| `tagEmbed` | `<embed>` | |
| `tagFieldset` | `<fieldset>` | |
| `tagFigcaption` | `<figcaption>` | |
| `tagFigure` | `<figure>` | |
| `tagFont` | `<font>` | Deprecated |
| `tagFooter` | `<footer>` | |
| `tagForm` | `<form>` | |
| `tagFrame` | `<frame>` | |
| `tagFrameset` | `<frameset>` | Deprecated |
| `tagH1`–`tagH6` | `<h1>`–`<h6>` | |
| `tagHead` | `<head>` | |
| `tagHeader` | `<header>` | |
| `tagHgroup` | `<hgroup>` | |
| `tagHtml` | `<html>` | |
| `tagHr` | `<hr>` | |
| `tagI` | `<i>` | |
| `tagIframe` | `<iframe>` | Deprecated |
| `tagImg` | `<img>` | |
| `tagInput` | `<input>` | |
| `tagIns` | `<ins>` | |
| `tagIsindex` | `<isindex>` | Deprecated |
| `tagKbd` | `<kbd>` | |
| `tagKeygen` | `<keygen>` | |
| `tagLabel` | `<label>` | |
| `tagLegend` | `<legend>` | |
| `tagLi` | `<li>` | |
| `tagLink` | `<link>` | |
| `tagMap` | `<map>` | |
| `tagMark` | `<mark>` | |
| `tagMenu` | `<menu>` | Deprecated |
| `tagMeta` | `<meta>` | |
| `tagMeter` | `<meter>` | |
| `tagNav` | `<nav>` | |
| `tagNobr` | `<nobr>` | Deprecated |
| `tagNoframes` | `<noframes>` | Deprecated |
| `tagNoscript` | `<noscript>` | |
| `tagObject` | `<object>` | |
| `tagOl` | `<ol>` | |
| `tagOptgroup` | `<optgroup>` | |
| `tagOption` | `<option>` | |
| `tagOutput` | `<output>` | |
| `tagP` | `<p>` | |
| `tagParam` | `<param>` | |
| `tagPre` | `<pre>` | |
| `tagProgress` | `<progress>` | |
| `tagQ` | `<q>` | |
| `tagRp` | `<rp>` | |
| `tagRt` | `<rt>` | |
| `tagRuby` | `<ruby>` | |
| `tagS` | `<s>` | Deprecated |
| `tagSamp` | `<samp>` | |
| `tagScript` | `<script>` | |
| `tagSection` | `<section>` | |
| `tagSelect` | `<select>` | |
| `tagSmall` | `<small>` | |
| `tagSource` | `<source>` | |
| `tagSpan` | `<span>` | |
| `tagStrike` | `<strike>` | Deprecated |
| `tagStrong` | `<strong>` | |
| `tagStyle` | `<style>` | |
| `tagSub` | `<sub>` | |
| `tagSummary` | `<summary>` | |
| `tagSup` | `<sup>` | |
| `tagTable` | `<table>` | |
| `tagTbody` | `<tbody>` | |
| `tagTd` | `<td>` | |
| `tagTextarea` | `<textarea>` | |
| `tagTfoot` | `<tfoot>` | |
| `tagTh` | `<th>` | |
| `tagThead` | `<thead>` | |
| `tagTime` | `<time>` | |
| `tagTitle` | `<title>` | |
| `tagTr` | `<tr>` | |
| `tagTrack` | `<track>` | |
| `tagTt` | `<tt>` | |
| `tagU` | `<u>` | Deprecated |
| `tagUl` | `<ul>` | |
| `tagVar` | `<var>` | |
| `tagVideo` | `<video>` | |
| `tagWbr` | `<wbr>` | |

---

## Practical Examples

### Example 1: Extract all hyperlinks

```nim
import std/[htmlparser, xmltree, strtabs]

let html = loadHtml("page.html")

for a in html.findAll("a"):
  if a.attrs != nil and a.attrs.hasKey("href"):
    echo a.attrs["href"]
```

---

### Example 2: Rewrite link extensions (from the official docs)

```nim
import std/[htmlparser, xmltree, strtabs, os, strutils]

proc transformHyperlinks() =
  let html = loadHtml("input.html")

  for a in html.findAll("a"):
    if a.attrs != nil and a.attrs.hasKey("href"):
      let (dir, filename, ext) = splitFile(a.attrs["href"])
      if cmpIgnoreCase(ext, ".rst") == 0:
        a.attrs["href"] = dir / filename & ".html"

  writeFile("output.html", $html)

transformHyperlinks()
```

---

### Example 3: Filtering nodes by `HtmlTag`

```nim
import std/[htmlparser, xmltree]

let doc = parseHtml("""
  <html><body>
    <h1>Heading</h1>
    <p>Paragraph 1</p>
    <p>Paragraph 2</p>
    <ul>
      <li>Item 1</li>
      <li>Item 2</li>
    </ul>
  </body></html>
""")

proc collectByTag(node: XmlNode, tag: HtmlTag, result: var seq[XmlNode]) =
  if node.kind == xnElement and node.htmlTag == tag:
    result.add(node)
  for child in node:
    collectByTag(child, tag, result)

var paragraphs: seq[XmlNode] = @[]
collectByTag(doc, tagP, paragraphs)

for p in paragraphs:
  echo p.innerText
```

---

### Example 4: Manual entity decoding

```nim
import std/htmlparser

let entities = ["gt", "lt", "amp", "Uuml", "Sigma", "#169", "#x1F600"]
for e in entities:
  let decoded = entityToUtf8(e)
  if decoded.len > 0:
    echo "&", e, "; → ", decoded
  else:
    echo "&", e, "; → unknown"
```

---

### Example 5: Parsing with error collection

```nim
import std/[htmlparser, streams]

let brokenHtml = """
  <html>
    <body>
      <p>Unclosed paragraph
      <div>Wrongly nested</p></div>
    </body>
  </html>
"""

var errors: seq[string] = @[]
let doc = parseHtml(newStringStream(brokenHtml), "broken.html", errors)

echo "Parse errors: ", errors.len
for e in errors:
  echo "  - ", e
echo "Document parsed anyway:"
echo doc
```

---

### Example 6: Tag membership check from a string

```nim
import std/htmlparser

proc isFormElement(tagName: string): bool =
  htmlTag(tagName) in {tagInput, tagSelect, tagTextarea, tagButton}

echo isFormElement("input")    # true
echo isFormElement("div")      # false
echo isFormElement("TEXTAREA") # true  (case-insensitive)
```

---

## Quick Reference

| Goal | Function |
|------|----------|
| Parse an HTML string | `parseHtml(htmlString)` |
| Parse HTML from a stream | `parseHtml(stream)` |
| Parse with error capture | `parseHtml(stream, filename, errors)` |
| Load HTML from a file | `loadHtml("path.html")` |
| Load file with error capture | `loadHtml("path.html", errors)` |
| Node tag as enum | `node.htmlTag` |
| String → tag enum | `htmlTag("div")` → `tagDiv` |
| Rune → HTML entity string | `runeToEntity(rune)` → `"#220"` |
| Entity → Rune | `entityToRune("Uuml")` |
| Entity → UTF-8 string | `entityToUtf8("Uuml")` → `"Ü"` |
| Tag enum → string | `tagToStr[tagDiv.ord - 1]` |

### Entity input formats for `entityToRune` / `entityToUtf8`

| Format | Example | Notes |
|--------|---------|-------|
| Named | `"Uuml"` | No surrounding `&` or `;` |
| Decimal | `"#220"` | Prefixed with `#` |
| Hexadecimal | `"#xDC"` or `"#XDC"` | Prefix `#x` or `#X` (case-insensitive) |
