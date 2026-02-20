# Nim `std/htmlparser` Module — Complete Reference

> **Installation:** `nimble install htmlparser`
>
> An HTML parser that tolerates real-world ("wild") malformed HTML and produces an `XmlNode` tree compatible with `std/xmltree`.  
> The parser aims for browser-like error recovery rather than strict compliance. Behaviour may change in future versions as the definition of "wild HTML" evolves.

---

## Table of Contents

1. [Overview](#overview)
2. [Types](#types)
3. [Constants](#constants)
4. [Tag Identification](#tag-identification)
5. [HTML Entity Conversion](#html-entity-conversion)
6. [Parsing](#parsing)
7. [Loading from File](#loading-from-file)
8. [Working with the Result Tree](#working-with-the-result-tree)
9. [HtmlTag Enum — Full List](#htmltag-enum--full-list)
10. [Quick Reference Table](#quick-reference-table)

---

## Overview

`std/htmlparser` reads HTML from a string, stream, or file and returns a tree of `XmlNode` values. Because the parser is lenient by design, it can handle HTML that would be rejected by strict XML parsers — unclosed tags, missing quotes, empty attributes, and so on.

```nim
import std/htmlparser
import std/xmltree

# Parse an HTML string
let tree = parseHtml("<h1>Hello <b>World</b></h1>")
echo tree   # → <h1>Hello <b>World</b></h1>

# Load from a file and print as XHTML
echo loadHtml("page.html")
```

> **Note:** All tag names in the resulting tree are **lower-case**.  
> **Note:** The `clientData` field of every resulting `XmlNode` is used internally by the parser to cache `HtmlTag` values. Do not use `clientData` yourself.

---

## Types

### `HtmlTag`

```nim
type HtmlTag* = enum
  tagUnknown,     ## Any tag not in the known list
  tagA,           ## <a>
  tagAbbr,        ## <abbr>
  tagAcronym,     ## <acronym>
  tagAddress,     ## <address>
  # … (see full list at the end of this document)
  tagWbr          ## <wbr>
```

An enum covering every HTML tag known to the parser, in alphabetical order. Used to identify tags in a parsed tree without fragile string comparisons.

```nim
import std/htmlparser, std/xmltree

let tree = parseHtml("<p>Hello</p>")
for node in tree:
  if node.kind == xnElement and node.htmlTag == tagP:
    echo "Found a paragraph: ", node
```

---

## Constants

### `tagToStr`

```nim
const tagToStr*: array[…, string]
```

Maps each `HtmlTag` enum value (starting from `tagA`) to its lowercase HTML string name. Useful for converting enum values back to strings.

```nim
echo tagToStr[tagA.ord - 1]    # → "a"
echo tagToStr[tagDiv.ord - 1]  # → "div"
```

> **Note:** The array starts from `tagA` (index 0). `tagUnknown` has no entry.

---

### `InlineTags`

```nim
const InlineTags*: set[HtmlTag]
```

The set of all HTML tags considered **inline** elements (elements that flow within text content). Includes: `a`, `abbr`, `acronym`, `applet`, `b`, `basefont`, `bdo`, `big`, `br`, `button`, `cite`, `code`, `del`, `dfn`, `em`, `font`, `i`, `img`, `ins`, `input`, `iframe`, `kbd`, `label`, `map`, `object`, `q`, `s`, `samp`, `script`, `select`, `small`, `span`, `strike`, `strong`, `sub`, `sup`, `textarea`, `tt`, `u`, `var`, `wbr`.

```nim
if node.htmlTag in InlineTags:
  echo node.tag, " is an inline element"
```

---

### `BlockTags`

```nim
const BlockTags*: set[HtmlTag]
```

The set of all HTML tags considered **block-level** elements. Includes: `address`, `blockquote`, `center`, `del`, `dir`, `div`, `dl`, `fieldset`, `form`, `h1`–`h6`, `hr`, `ins`, `isindex`, `menu`, `noframes`, `noscript`, `ol`, `p`, `pre`, `table`, `ul`.

```nim
if node.htmlTag in BlockTags:
  echo node.tag, " is a block element"
```

---

### `SingleTags`

```nim
const SingleTags*: set[HtmlTag]
```

The set of **void / self-closing** tags that never have children. The parser does not expect a closing tag for these. Includes: `area`, `base`, `basefont`, `br`, `col`, `frame`, `hr`, `img`, `isindex`, `link`, `meta`, `param`, `source`, `wbr`.

```nim
if node.htmlTag in SingleTags:
  echo node.tag, " is a void element (no closing tag)"
```

---

## Tag Identification

### `htmlTag(n: XmlNode)`

```nim
proc htmlTag*(n: XmlNode): HtmlTag
```

Returns the `HtmlTag` enum value for a parsed `XmlNode`. The result is cached in `n.clientData` after the first call, so repeated calls are cheap.

```nim
import std/htmlparser, std/xmltree

let tree = parseHtml("<div><p>text</p></div>")
for node in tree:
  case node.htmlTag
  of tagDiv: echo "div found"
  of tagP:   echo "paragraph found"
  else: discard
```

---

### `htmlTag(s: string)`

```nim
proc htmlTag*(s: string): HtmlTag
```

Converts an arbitrary string to the corresponding `HtmlTag` enum value. The comparison is **case-insensitive**. Returns `tagUnknown` if the string is not a known HTML tag.

```nim
echo htmlTag("DIV")    # → tagDiv
echo htmlTag("span")   # → tagSpan
echo htmlTag("foobar") # → tagUnknown
```

---

## HTML Entity Conversion

### `runeToEntity`

```nim
proc runeToEntity*(rune: Rune): string
```

Converts a Unicode `Rune` to its **numeric HTML entity** string (`#NNN` form). Returns `""` for `Rune(0)` or any negative rune.

```nim
import std/unicode, std/htmlparser

echo runeToEntity(Rune(0))           # → ""
echo runeToEntity(Rune(-1))          # → ""
echo runeToEntity("Ü".runeAt(0))    # → "#220"
echo runeToEntity("∈".runeAt(0))    # → "#8712"
```

---

### `entityToRune`

```nim
proc entityToRune*(entity: string): Rune
```

Converts an HTML entity name or numeric reference to a `Rune`. Returns `Rune(0)` for unknown or invalid entities.

Supported formats:
- Named entities: `"gt"`, `"Uuml"`, `"amp"`, `"Sigma"`, etc.
- Decimal numeric: `"#63"`, `"#931"`, `"#0931"`
- Hexadecimal numeric: `"#x3A3"`, `"#x03A3"`, `"#X3a3"` (case-insensitive `x`)

The `&` and `;` delimiters are **not** included — pass only the content between them.

```nim
import std/unicode, std/htmlparser

echo entityToRune("")        # → Rune(0)
echo entityToRune("gt")      # → Rune('>')
echo entityToRune("Uuml")    # → Rune('Ü')
echo entityToRune("#63")     # → Rune('?')
echo entityToRune("#x3A3")   # → Rune('Σ')
```

---

### `entityToUtf8`

```nim
proc entityToUtf8*(entity: string): string
```

Converts an HTML entity name or numeric reference to its **UTF-8 string**. Returns `""` for unknown entities. This is a convenience wrapper over `entityToRune` + `toUTF8`.

The HTML parser already applies this conversion automatically during parsing; use this function when you need to decode entities manually.

```nim
echo entityToUtf8("")        # → ""
echo entityToUtf8("gt")      # → ">"
echo entityToUtf8("Uuml")    # → "Ü"
echo entityToUtf8("quest")   # → "?"
echo entityToUtf8("#63")     # → "?"
echo entityToUtf8("#931")    # → "Σ"
echo entityToUtf8("#x3A3")   # → "Σ"
echo entityToUtf8("#X3a3")   # → "Σ"
```

---

## Parsing

### `parseHtml(s, filename, errors)`

```nim
proc parseHtml*(s: Stream, filename: string,
                errors: var seq[string]): XmlNode
```

Parses HTML from the open `Stream` `s`. The `filename` parameter is used only in error messages. Each recoverable parsing error is appended as a string to `errors`.

Returns the root `XmlNode` of the parsed document. If the document has a single top-level element it is returned directly; otherwise a synthetic `"document"` wrapper element is returned.

```nim
import std/streams, std/htmlparser

var errors: seq[string]
let s = newStringStream("<p>Hello <b>World")
let tree = parseHtml(s, "my_doc.html", errors)
for e in errors: echo "Parse error: ", e
echo tree
```

---

### `parseHtml(s: Stream)`

```nim
proc parseHtml*(s: Stream): XmlNode
```

Parses HTML from a stream, silently discarding all parsing errors.

```nim
import std/streams, std/htmlparser

let tree = parseHtml(newStringStream("<ul><li>A<li>B<li>C"))
echo tree
```

---

### `parseHtml(html: string)`

```nim
proc parseHtml*(html: string): XmlNode
```

Parses HTML directly from a string. All parsing errors are ignored. This is the most convenient overload for quick use.

```nim
import std/htmlparser, std/xmltree

let tree = parseHtml("""
  <html>
    <body>
      <h1>Title</h1>
      <p>Paragraph with <a href="https://nim-lang.org">a link</a>.</p>
    </body>
  </html>
""")

for a in tree.findAll("a"):
  echo a.attrs["href"]
# → https://nim-lang.org
```

---

## Loading from File

### `loadHtml(path, errors)`

```nim
proc loadHtml*(path: string, errors: var seq[string]): XmlNode
```

Opens the file at `path`, parses its HTML content, and returns the `XmlNode` tree. Each parsing error is appended to `errors`. Raises `IOError` if the file cannot be opened.

```nim
import std/htmlparser

var errors: seq[string]
let tree = loadHtml("page.html", errors)
if errors.len > 0:
  echo "Warnings during parse:"
  for e in errors: echo "  ", e
echo tree
```

---

### `loadHtml(path: string)`

```nim
proc loadHtml*(path: string): XmlNode
```

Opens and parses an HTML file, discarding all parsing errors. Raises `IOError` if the file cannot be read.

```nim
import std/htmlparser
echo loadHtml("mydirty.html")
```

---

## Working with the Result Tree

The returned `XmlNode` is a standard `std/xmltree` node. All `xmltree` procedures work on it normally. Key operations:

```nim
import std/htmlparser, std/xmltree, std/strtabs

let tree = parseHtml("""
  <html><body>
    <a href="page.rst">ReST page</a>
    <a href="other.html">HTML page</a>
  </body></html>
""")

# Iterate over all <a> tags
for a in tree.findAll("a"):
  echo a.attrs.getOrDefault("href", "(no href)")

# Check the HtmlTag enum
for node in tree:
  if node.kind == xnElement:
    echo node.tag, " → ", node.htmlTag

# Modify attributes
for a in tree.findAll("a"):
  if a.attrs.hasKey("href") and a.attrs["href"].endsWith(".rst"):
    a.attrs["href"] = a.attrs["href"][0..^5] & ".html"

# Serialise back to HTML string
echo $tree
```

### Practical example — transforming hyperlinks

```nim
import std/htmlparser, std/xmltree, std/strtabs, std/os, std/strutils

proc transformHyperlinks() =
  let html = loadHtml("input.html")

  for a in html.findAll("a"):
    if a.attrs.hasKey("href"):
      let (dir, filename, ext) = splitFile(a.attrs["href"])
      if cmpIgnoreCase(ext, ".rst") == 0:
        a.attrs["href"] = dir / filename & ".html"

  writeFile("output.html", $html)
```

---

## HtmlTag Enum — Full List

| Enum value | HTML tag | Notes |
|---|---|---|
| `tagUnknown` | *(any unknown tag)* | |
| `tagA` | `<a>` | |
| `tagAbbr` | `<abbr>` | deprecated per module comment |
| `tagAcronym` | `<acronym>` | |
| `tagAddress` | `<address>` | |
| `tagApplet` | `<applet>` | deprecated |
| `tagArea` | `<area>` | void |
| `tagArticle` | `<article>` | |
| `tagAside` | `<aside>` | |
| `tagAudio` | `<audio>` | |
| `tagB` | `<b>` | |
| `tagBase` | `<base>` | void |
| `tagBdi` | `<bdi>` | |
| `tagBdo` | `<bdo>` | deprecated per module comment |
| `tagBasefont` | `<basefont>` | deprecated, void |
| `tagBig` | `<big>` | |
| `tagBlockquote` | `<blockquote>` | |
| `tagBody` | `<body>` | |
| `tagBr` | `<br>` | void |
| `tagButton` | `<button>` | |
| `tagCanvas` | `<canvas>` | |
| `tagCaption` | `<caption>` | |
| `tagCenter` | `<center>` | deprecated |
| `tagCite` | `<cite>` | |
| `tagCode` | `<code>` | |
| `tagCol` | `<col>` | void |
| `tagColgroup` | `<colgroup>` | |
| `tagCommand` | `<command>` | |
| `tagDatalist` | `<datalist>` | |
| `tagDd` | `<dd>` | |
| `tagDel` | `<del>` | |
| `tagDetails` | `<details>` | |
| `tagDfn` | `<dfn>` | |
| `tagDialog` | `<dialog>` | |
| `tagDiv` | `<div>` | |
| `tagDir` | `<dir>` | deprecated |
| `tagDl` | `<dl>` | |
| `tagDt` | `<dt>` | |
| `tagEm` | `<em>` | |
| `tagEmbed` | `<embed>` | |
| `tagFieldset` | `<fieldset>` | |
| `tagFigcaption` | `<figcaption>` | |
| `tagFigure` | `<figure>` | |
| `tagFont` | `<font>` | deprecated |
| `tagFooter` | `<footer>` | |
| `tagForm` | `<form>` | |
| `tagFrame` | `<frame>` | void |
| `tagFrameset` | `<frameset>` | deprecated |
| `tagH1`–`tagH6` | `<h1>`–`<h6>` | |
| `tagHead` | `<head>` | |
| `tagHeader` | `<header>` | |
| `tagHgroup` | `<hgroup>` | |
| `tagHtml` | `<html>` | |
| `tagHr` | `<hr>` | void |
| `tagI` | `<i>` | |
| `tagIframe` | `<iframe>` | deprecated per module comment |
| `tagImg` | `<img>` | void |
| `tagInput` | `<input>` | void |
| `tagIns` | `<ins>` | |
| `tagIsindex` | `<isindex>` | deprecated, void |
| `tagKbd` | `<kbd>` | |
| `tagKeygen` | `<keygen>` | |
| `tagLabel` | `<label>` | |
| `tagLegend` | `<legend>` | |
| `tagLi` | `<li>` | |
| `tagLink` | `<link>` | void |
| `tagMap` | `<map>` | |
| `tagMark` | `<mark>` | |
| `tagMenu` | `<menu>` | deprecated |
| `tagMeta` | `<meta>` | void |
| `tagMeter` | `<meter>` | |
| `tagNav` | `<nav>` | |
| `tagNobr` | `<nobr>` | deprecated |
| `tagNoframes` | `<noframes>` | deprecated |
| `tagNoscript` | `<noscript>` | |
| `tagObject` | `<object>` | |
| `tagOl` | `<ol>` | |
| `tagOptgroup` | `<optgroup>` | |
| `tagOption` | `<option>` | |
| `tagOutput` | `<output>` | |
| `tagP` | `<p>` | |
| `tagParam` | `<param>` | void |
| `tagPre` | `<pre>` | |
| `tagProgress` | `<progress>` | |
| `tagQ` | `<q>` | |
| `tagRp` | `<rp>` | |
| `tagRt` | `<rt>` | |
| `tagRuby` | `<ruby>` | |
| `tagS` | `<s>` | deprecated |
| `tagSamp` | `<samp>` | |
| `tagScript` | `<script>` | |
| `tagSection` | `<section>` | |
| `tagSelect` | `<select>` | |
| `tagSmall` | `<small>` | |
| `tagSource` | `<source>` | void |
| `tagSpan` | `<span>` | |
| `tagStrike` | `<strike>` | deprecated |
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
| `tagU` | `<u>` | deprecated |
| `tagUl` | `<ul>` | |
| `tagVar` | `<var>` | |
| `tagVideo` | `<video>` | |
| `tagWbr` | `<wbr>` | void |

---

## Quick Reference Table

| Symbol | Kind | Description |
|---|---|---|
| `HtmlTag` | enum | Identifies every known HTML tag; `tagUnknown` for unrecognised tags |
| `tagToStr` | const array | Maps `HtmlTag` enum values to lowercase tag name strings |
| `InlineTags` | const set | Set of inline-level HTML tags |
| `BlockTags` | const set | Set of block-level HTML tags |
| `SingleTags` | const set | Set of void/self-closing tags (no children, no closing tag) |
| `htmlTag(XmlNode)` | proc | Get the `HtmlTag` of a parsed node (result cached in `clientData`) |
| `htmlTag(string)` | proc | Convert a tag name string to `HtmlTag` (case-insensitive) |
| `runeToEntity` | proc | Convert `Rune` → numeric HTML entity string (`#NNN`) |
| `entityToRune` | proc | Convert HTML entity name or `#NNN`/`#xNN` reference → `Rune` |
| `entityToUtf8` | proc | Convert HTML entity name or reference → UTF-8 string |
| `parseHtml(Stream, string, var seq)` | proc | Parse HTML from stream; collect errors in seq |
| `parseHtml(Stream)` | proc | Parse HTML from stream; ignore errors |
| `parseHtml(string)` | proc | Parse HTML from string; ignore errors |
| `loadHtml(string, var seq)` | proc | Load and parse HTML file; collect errors |
| `loadHtml(string)` | proc | Load and parse HTML file; ignore errors |
