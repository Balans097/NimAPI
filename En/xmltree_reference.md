# xmltree — Module Reference

> Nim standard library module for building and manipulating XML trees in memory.
> The module lets you **create**, **inspect**, **traverse**, **modify**, and **serialize** XML documents without ever touching a file or a parser — everything lives as a tree of `XmlNode` objects.

---

## Table of Contents

1. [Core Types](#core-types)
2. [Constants](#constants)
3. [Node Constructors](#node-constructors)
4. [Text & Tag Access](#text--tag-access)
5. [Inner Text](#inner-text)
6. [Tree Manipulation](#tree-manipulation)
7. [Indexing & Iteration](#indexing--iteration)
8. [Attributes](#attributes)
9. [Searching](#searching)
10. [Serialization](#serialization)
11. [Escaping](#escaping)
12. [Speed Hacks](#speed-hacks)
13. [Client Data](#client-data)
14. [The `<>` Macro](#the--macro)

---

## Core Types

### `XmlNode`

```nim
type XmlNode* = ref XmlNodeObj
```

The fundamental building block of the module. Every piece of an XML document — an element, a text run, a comment, a CDATA section, an entity reference — is represented as an `XmlNode`. Because it is a `ref` type, nodes are passed by reference; two variables can point to the same subtree.

---

### `XmlNodeKind`

```nim
type XmlNodeKind* = enum
  xnText,         # plain text content
  xnVerbatimText, # text inserted without escaping
  xnElement,      # a tag with optional children and attributes
  xnCData,        # <![CDATA[…]]> section
  xnEntity,       # &name; reference
  xnComment       # <!-- … -->
```

Every `XmlNode` has a `kind` field of this type. Knowing the kind is essential because many procedures only make sense for certain kinds (e.g., `tag` only works on `xnElement`). Calling a procedure on the wrong kind raises an assertion at runtime.

---

### `XmlAttributes`

```nim
type XmlAttributes* = StringTableRef
```

A case-sensitive string-to-string map that stores the attributes of an element node (`xnElement`). It is initialized lazily — elements start with `nil` attributes to save memory, and a non-nil value is only created when you actually set attributes.

---

## Constants

### `xmlHeader`

```nim
const xmlHeader* = "<?xml version=\"1.0\" encoding=\"UTF-8\" ?>\n"
```

The standard XML declaration line. When you want to produce a complete, standalone XML file (not just a fragment), prepend this string to the serialized tree.

```nim
let doc = newElement("root")
doc.add newText("hello")
echo xmlHeader & $doc
# <?xml version="1.0" encoding="UTF-8" ?>
# <root>hello</root>
```

---

## Node Constructors

These procedures create leaf or branch nodes of a specific kind.

---

### `newElement`

```nim
proc newElement*(tag: sink string): XmlNode
```

Creates an **element node** — the workhorse of XML. Elements have a tag name, optional attributes, and a list of children. After creation the children list is empty and attributes are `nil`.

```nim
let p = newElement("p")
p.add newText("Hello, world!")
echo $p
# <p>Hello, world!</p>
```

---

### `newText`

```nim
proc newText*(text: sink string): XmlNode
```

Creates a **text node**. When serialized, special characters (`<`, `>`, `&`, `"`, `'`) are automatically escaped so the output is always valid XML.

```nim
let t = newText("Price < 10 & tax > 0")
echo $t
# Price &lt; 10 &amp; tax &gt; 0
```

---

### `newVerbatimText`

```nim
proc newVerbatimText*(text: sink string): XmlNode
```

Like `newText`, but the content is written **without any escaping**. Use this when you have already-escaped or non-XML content that must be inserted verbatim — for example, an embedded HTML snippet or a pre-rendered string. Use with caution: if the content contains unescaped `<` or `&`, the resulting document will be malformed.

```nim
let raw = newVerbatimText("&already;&#x2013;escaped")
echo $raw
# &already;&#x2013;escaped   (no double-escaping)
```

---

### `newComment`

```nim
proc newComment*(comment: sink string): XmlNode
```

Creates an **XML comment** node. Comments are serialized as `<!-- text -->`. They are visible in the raw XML source but are ignored by XML processors.

```nim
let c = newComment("TODO: remove this section")
echo $c
# <!-- TODO: remove this section -->
```

---

### `newCData`

```nim
proc newCData*(cdata: sink string): XmlNode
```

Creates a **CDATA section**. The content is wrapped in `<![CDATA[` … `]]>` and is never escaped by the serializer. CDATA sections are traditionally used to embed blocks of text that contain many special characters (e.g., JavaScript or CSS inside an XML document).

```nim
let js = newCData("if (a < b && c > 0) { return true; }")
echo $js
# <![CDATA[if (a < b && c > 0) { return true; }]]>
```

---

### `newEntity`

```nim
proc newEntity*(entity: string): XmlNode
```

Creates an **entity reference** node. The name is wrapped in `&` … `;`. Common built-in entities (`&amp;`, `&lt;`, etc.) are handled by `newText`; `newEntity` is for named entities defined in a DTD or for custom entity references.

```nim
let e = newEntity("copy")
echo $e
# &copy;
```

---

### `newXmlTree`

```nim
proc newXmlTree*(tag: sink string,
                 children: openArray[XmlNode],
                 attributes: XmlAttributes = nil): XmlNode
```

A convenience constructor that creates an element node and populates it with children and attributes in one call. Equivalent to calling `newElement`, followed by `add` for each child, and setting `attrs`. Ideal for building trees in a single expression.

```nim
let attrs = {"lang": "en", "dir": "ltr"}.toXmlAttributes
let html  = newXmlTree("html", [
  newXmlTree("head", [newElement("title")]),
  newElement("body")
], attrs)
echo $html
# <html dir="ltr" lang="en">
#   <head>
#     <title />
#   </head>
#   <body />
# </html>
```

> **Note:** Attribute order in the output is not guaranteed because `XmlAttributes` is a hash table internally.

---

## Text & Tag Access

### `text` (getter)

```nim
proc text*(n: XmlNode): lent string
```

Returns the text content of a node. Valid for `xnText`, `xnVerbatimText`, `xnComment`, `xnCData`, and `xnEntity` nodes. Calling it on an `xnElement` raises an assertion.

```nim
let c = newComment("draft")
echo c.text   # draft
```

---

### `text=` (setter)

```nim
proc `text=`*(n: XmlNode, text: sink string)
```

Replaces the text content of a leaf node. Valid for the same kinds as the getter.

```nim
var e = newEntity("nbsp")
e.text = "copy"
echo $e   # &copy;
```

---

### `tag` (getter)

```nim
proc tag*(n: XmlNode): lent string
```

Returns the tag name of an element node. The node must be `xnElement`.

```nim
let el = newElement("section")
echo el.tag   # section
```

---

### `tag=` (setter)

```nim
proc `tag=`*(n: XmlNode, tag: sink string)
```

Changes the tag name of an existing element node, leaving its children and attributes untouched.

```nim
var el = newElement("div")
el.tag = "section"
echo el.tag   # section
```

---

### `rawText`

```nim
proc rawText*(n: XmlNode): string
```

Returns the underlying text string by moving or shallow-copying it (depending on the GC mode). This is a **speed hack** intended for performance-critical serializers; it may consume the node's content. Prefer `text` for normal use.

---

### `rawTag`

```nim
proc rawTag*(n: XmlNode): string
```

Same as `rawText` but for the tag field. Again, a speed hack for specialized serializers.

---

## Inner Text

### `innerText`

```nim
proc innerText*(n: XmlNode): string
```

Recursively collects and concatenates all text and entity content within a node, ignoring comments, CDATA, and element tags themselves. This mirrors what a browser would display as the "visible text" of an HTML element.

```nim
var article = newElement("article")
article.add newElement("h1")          # no text, ignored
article.add newText("Hello ")
article.add newEntity("mdash")        # entity text included
article.add newText(" world")

echo innerText(article)   # Hello —— world
# (entity name "mdash" is concatenated, not the glyph itself)
```

---

## Tree Manipulation

### `add` (single child)

```nim
proc add*(father, son: XmlNode)
```

Appends a single child node to the end of `father`'s children list. `father` must be `xnElement`.

```nim
let ul = newElement("ul")
ul.add newXmlTree("li", [newText("Item 1")])
ul.add newXmlTree("li", [newText("Item 2")])
echo $ul
# <ul>
#   <li>Item 1</li>
#   <li>Item 2</li>
# </ul>
```

---

### `add` (multiple children)

```nim
proc add*(father: XmlNode, sons: openArray[XmlNode])
```

Appends an array or sequence of children at once. Functionally equivalent to calling the single-child `add` in a loop.

```nim
let row = newElement("tr")
row.add @[newXmlTree("td", [newText("A")]),
          newXmlTree("td", [newText("B")])]
echo $row
# <tr>
#   <td>A</td>
#   <td>B</td>
# </tr>
```

---

### `insert` (single child)

```nim
proc insert*(father, son: XmlNode, index: int)
```

Inserts a child at a specific position. All existing children at or after `index` are shifted right. If `index` is beyond the current length, the node is appended at the end.

```nim
var list = newElement("ol")
list.add newXmlTree("li", [newText("second")])
list.insert(newXmlTree("li", [newText("first")]), 0)
echo $list
# <ol>
#   <li>first</li>
#   <li>second</li>
# </ol>
```

---

### `insert` (multiple children)

```nim
proc insert*(father: XmlNode, sons: openArray[XmlNode], index: int)
```

Inserts an array of children at a specific position, preserving their relative order.

```nim
var list = newElement("ol")
list.add newXmlTree("li", [newText("third")])
list.insert([newXmlTree("li", [newText("first")]),
             newXmlTree("li", [newText("second")])], 0)
# <ol><li>first</li><li>second</li><li>third</li></ol>
```

---

### `delete` (by index)

```nim
proc delete*(n: XmlNode, i: Natural)
```

Removes the child at position `i`. Subsequent children shift left to fill the gap. Raises an out-of-bounds error if `i >= n.len`.

```nim
var nav = newElement("nav")
nav.add newXmlTree("a", [newText("Home")])
nav.add newXmlTree("a", [newText("About")])
nav.delete(0)   # removes "Home"
echo $nav
# <nav>
#   <a>About</a>
# </nav>
```

---

### `delete` (by slice)

```nim
proc delete*(n: XmlNode, slice: Slice[int])
```

Removes a contiguous range of children. For example, `n.delete(1..3)` removes children at indices 1, 2, and 3.

```nim
var row = newElement("tr")
for i in 1..5:
  row.add newXmlTree("td", [newText($i)])
row.delete(1..3)   # keep only td[0] and td[4]
```

---

### `replace` (single index)

```nim
proc replace*(n: XmlNode, i: Natural, replacement: openArray[XmlNode])
```

Replaces the child at position `i` with all nodes in `replacement`. The replacement can be a different number of nodes than 1, so one child can be expanded into many.

```nim
var root = newElement("root")
root.add newElement("placeholder")
root.replace(0, @[newElement("real"), newElement("content")])
echo $root
# <root>
#   <real />
#   <content />
# </root>
```

---

### `replace` (slice)

```nim
proc replace*(n: XmlNode, slice: Slice[int], replacement: openArray[XmlNode])
```

Replaces a range of children with a new set. The number of nodes in `replacement` does not need to equal the length of the slice.

```nim
var root = newElement("root")
root.add([newElement("a"), newElement("b"), newElement("c")])
root.replace(0..1, @[newElement("x")])
# children are now: x, c
```

---

### `clear`

```nim
proc clear*(n: var XmlNode)
```

Recursively removes **all** children from `n` and every descendant element. The node itself and its attributes survive; only the children are erased. Useful for reusing a tree skeleton with different content.

```nim
var form = newXmlTree("form",
  [newElement("input"), newElement("button")],
  {"action": "/submit"}.toXmlAttributes)
echo form.len   # 2
clear(form)
echo form.len   # 0
echo $form      # <form action="/submit" />
```

---

## Indexing & Iteration

### `len`

```nim
proc len*(n: XmlNode): int
```

Returns the number of direct children. For non-element nodes this is always 0.

```nim
let parent = newXmlTree("ul", [newElement("li"), newElement("li")])
echo parent.len   # 2
```

---

### `kind`

```nim
proc kind*(n: XmlNode): XmlNodeKind
```

Returns the kind of the node. Use this to branch on node type before calling kind-specific procedures.

```nim
proc describe(n: XmlNode) =
  case n.kind
  of xnElement: echo "element: ", n.tag
  of xnText:    echo "text: ",    n.text
  of xnComment: echo "comment: ", n.text
  else:         echo "other"
```

---

### `[]` (read)

```nim
proc `[]`*(n: XmlNode, i: int): XmlNode
```

Returns the `i`-th child (zero-based). Raises an assertion if `n` is not `xnElement`.

```nim
let parent = newXmlTree("ul", [newElement("li"), newElement("li")])
echo parent[0].tag   # li
```

---

### `[]` (write)

```nim
proc `[]`*(n: var XmlNode, i: int): var XmlNode
```

Returns a mutable reference to the `i`-th child so you can modify it in place.

---

### `items` (iterator)

```nim
iterator items*(n: XmlNode): XmlNode
```

Iterates over direct children of an element node in order. This is what powers `for child in parent:` loops.

```nim
let menu = newXmlTree("menu",
  [newXmlTree("item", [newText("File")]),
   newXmlTree("item", [newText("Edit")])])

for item in menu:
  echo item.innerText
# File
# Edit
```

---

### `mitems` (mutable iterator)

```nim
iterator mitems*(n: var XmlNode): var XmlNode
```

Like `items` but yields mutable references so you can modify each child without indexing.

```nim
var list = newXmlTree("ul",
  [newXmlTree("li", [newText("old")])])

for child in mitems(list):
  if child.kind == xnElement:
    child[0].text = "new"

echo $list   # <ul><li>new</li></ul>
```

---

## Attributes

### `toXmlAttributes`

```nim
proc toXmlAttributes*(keyValuePairs: varargs[tuple[key, val: string]]): XmlAttributes
```

Converts a sequence of `(key, value)` pairs into an `XmlAttributes` table. The idiomatic way to call it is via the `{}` literal syntax.

```nim
let attrs = {"class": "hero", "id": "banner"}.toXmlAttributes
let div = newElement("div")
div.attrs = attrs
echo $div
# <div class="hero" id="banner" />
```

---

### `attrs` (getter)

```nim
proc attrs*(n: XmlNode): XmlAttributes
```

Returns the `XmlAttributes` table of an element node, or `nil` if no attributes have been set yet. Always check for `nil` before iterating or indexing.

```nim
let el = newElement("span")
echo el.attrs == nil   # true — no attributes yet
```

---

### `attrs=` (setter)

```nim
proc `attrs=`*(n: XmlNode, attr: XmlAttributes)
```

Replaces the entire attribute table of an element. Passing `nil` clears all attributes.

```nim
var el = newElement("input")
el.attrs = {"type": "text", "name": "q"}.toXmlAttributes
echo $el
# <input name="q" type="text" />
```

---

### `attrsLen`

```nim
proc attrsLen*(n: XmlNode): int
```

Returns the number of attributes. Returns 0 both when attributes are `nil` and when the table is present but empty.

```nim
let el = newElement("br")
echo el.attrsLen   # 0
el.attrs = {"class": "wide"}.toXmlAttributes
echo el.attrsLen   # 1
```

---

### `attr`

```nim
proc attr*(n: XmlNode, name: string): string
```

Looks up a single attribute value by name. Returns an empty string if the attribute does not exist or if `attrs` is `nil`. Safe to call without checking for `nil` first.

```nim
let img = newElement("img")
img.attrs = {"src": "photo.png", "alt": "A photo"}.toXmlAttributes
echo img.attr("src")      # photo.png
echo img.attr("missing")  # (empty string)
```

---

## Searching

### `child`

```nim
proc child*(n: XmlNode, name: string): XmlNode
```

Searches the **direct** children of `n` for the first element whose tag matches `name`. Returns `nil` if none is found. The search is case-sensitive.

```nim
let root = newXmlTree("root", [
  newXmlTree("meta", []),
  newXmlTree("body", [newText("content")])
])
let body = root.child("body")
echo innerText(body)   # content
```

---

### `findAll` (in-place)

```nim
proc findAll*(n: XmlNode, tag: string,
              result: var seq[XmlNode],
              caseInsensitive = false)
```

Recursively searches the entire subtree rooted at `n` and **appends** all element nodes matching `tag` to the existing `result` sequence. Because it appends rather than replacing, you can call it multiple times to accumulate results from different subtrees.

```nim
var matches: seq[XmlNode]
root.findAll("td", matches)
echo matches.len   # all <td> elements anywhere in the tree
```

---

### `findAll` (returning seq)

```nim
proc findAll*(n: XmlNode, tag: string,
              caseInsensitive = false): seq[XmlNode]
```

A shorthand that creates and returns a new sequence. More convenient when you don't need to reuse an existing accumulator.

```nim
let allLinks = page.findAll("a")
for link in allLinks:
  echo link.attr("href")
```

The `caseInsensitive` flag makes the tag comparison ignore case — useful when processing HTML where tag names may be mixed case.

```nim
let allDivs = doc.findAll("div", caseInsensitive = true)
# matches <div>, <Div>, <DIV>, etc.
```

---

## Serialization

### `add` (string overload)

```nim
proc add*(result: var string, n: XmlNode,
          indent = 0, indWidth = 2, addNewLines = true)
```

Appends the serialized XML representation of `n` to the string `result`. This is the low-level building block used by `$`. The parameters give fine-grained control over formatting: `indent` sets the starting indentation level (number of spaces), `indWidth` controls how many additional spaces are added per nesting level, and `addNewLines` controls whether elements appear on separate lines.

```nim
var output = "<!-- generated -->\n"
output.add(myTree, indent = 0, indWidth = 4)
```

---

### `$` (stringify)

```nim
proc `$`*(n: XmlNode): string
```

Converts a node (or entire tree) to its XML string representation using default formatting (two-space indent, newlines enabled). No XML declaration (`<?xml … ?>`) is prepended; use `xmlHeader` separately if needed.

```nim
let tree = newXmlTree("note",
  [newXmlTree("to",   [newText("Alice")]),
   newXmlTree("from", [newText("Bob")]),
   newXmlTree("body", [newText("See you there!")])])
echo $tree
# <note>
#   <to>Alice</to>
#   <from>Bob</from>
#   <body>See you there!</body>
# </note>
```

---

## Escaping

### `escape`

```nim
proc escape*(s: string): string
```

Returns a new string with XML special characters replaced by their entity equivalents:

| Character | Becomes  |
|-----------|----------|
| `<`       | `&lt;`   |
| `>`       | `&gt;`   |
| `&`       | `&amp;`  |
| `"`       | `&quot;` |
| `'`       | `&apos;` |

Use this when you need to embed a plain string into an XML attribute value or text outside of a `newText` node.

```nim
let safe = escape("<script>alert('xss')</script>")
echo safe
# &lt;script&gt;alert(&apos;xss&apos;)&lt;/script&gt;
```

---

### `addEscaped`

```nim
proc addEscaped*(result: var string, s: string)
```

The in-place, allocation-efficient version of `escape`. Instead of returning a new string, it appends the escaped content directly to an existing `result` string. Prefer this in tight loops or when building large documents character by character.

```nim
var buf = "<description>"
buf.addEscaped("Price < 100 & discount > 0")
buf.add("</description>")
echo buf
# <description>Price &lt; 100 &amp; discount &gt; 0</description>
```

---

## Speed Hacks

### `rawText` and `rawTag`

These are performance-oriented accessors that either move or shallow-copy the underlying string field without any allocation. They are intended for custom, high-throughput serializers that know what they are doing. Under GC destructors, calling `rawText` or `rawTag` **moves** the content out of the node, leaving the node in an empty state. Do not use these in application code.

---

## Client Data

### `clientData` (getter)

```nim
proc clientData*(n: XmlNode): int
```

Returns an integer field that is entirely at the caller's disposal. The field is used internally by Nim's HTML parser and generator to store parser state. You may repurpose it for your own bookkeeping when building custom tools on top of `xmltree`.

---

### `clientData=` (setter)

```nim
proc `clientData=`*(n: XmlNode, data: int)
```

Sets the client data field.

```nim
let node = newElement("section")
node.clientData = 42
echo node.clientData   # 42
```

---

## The `<>` Macro

```nim
macro `<>`*(x: untyped): untyped
```

A compile-time macro that provides a concise, HTML-like syntax for building XML trees. Attributes are written as `key = value` expressions, and child nodes are passed as positional arguments.

The macro expands at compile time into the equivalent `newXmlTree` / `newStringTable` call, so there is zero runtime overhead compared to the verbose form.

**Syntax:**

```nim
<>tagName(attr1 = "val1", attr2 = "val2", childNode1, childNode2)
```

**Examples:**

```nim
# Simple element with an attribute
let link = <>a(href = "https://nim-lang.org", newText("Nim"))
echo $link
# <a href="https://nim-lang.org">Nim</a>

# Nested structure
let card = <>div(class = "card",
  <>h2(newText("Title")),
  <>p(newText("Body text.")))
echo $card
# <div class="card">
#   <h2>Title</h2>
#   <p>Body text.</p>
# </div>

# Self-closing element (no children)
let br = <>br()
echo $br
# <br />
```

> **Tip:** Attribute names containing hyphens (e.g., `data-lang`) are supported; the macro strips whitespace that would otherwise be introduced by the Nim tokenizer.

---

## Quick-Reference Summary

| Procedure / Macro | What it does |
|---|---|
| `newElement(tag)` | Create element node |
| `newText(s)` | Create escaped text node |
| `newVerbatimText(s)` | Create raw (unescaped) text node |
| `newComment(s)` | Create comment node |
| `newCData(s)` | Create CDATA section |
| `newEntity(s)` | Create entity reference |
| `newXmlTree(tag, children, attrs)` | Create element with children & attrs at once |
| `n.text` / `n.text=` | Get/set leaf text content |
| `n.tag` / `n.tag=` | Get/set element tag name |
| `innerText(n)` | Concatenate all text/entity content recursively |
| `n.add(son)` | Append one child |
| `n.add(sons)` | Append multiple children |
| `n.insert(son, i)` | Insert one child at position |
| `n.insert(sons, i)` | Insert multiple children at position |
| `n.delete(i)` | Remove child at index |
| `n.delete(slice)` | Remove range of children |
| `n.replace(i, nodes)` | Replace child at index |
| `n.replace(slice, nodes)` | Replace range of children |
| `clear(n)` | Remove all children recursively |
| `n.len` | Number of direct children |
| `n.kind` | Node kind (`XmlNodeKind`) |
| `n[i]` | Access i-th child |
| `for c in n` | Iterate children |
| `for c in mitems(n)` | Iterate children (mutable) |
| `toXmlAttributes({…})` | Create attribute table |
| `n.attrs` / `n.attrs=` | Get/set attribute table |
| `n.attrsLen` | Number of attributes |
| `n.attr(name)` | Get single attribute value |
| `n.child(name)` | Find first direct child by tag |
| `n.findAll(tag)` | Find all descendants by tag |
| `$n` | Serialize to string |
| `s.add(n)` | Append serialized node to string |
| `escape(s)` | Escape XML special chars |
| `addEscaped(result, s)` | Efficient in-place escaping |
| `xmlHeader` | Standard XML declaration constant |
| `<>tag(attrs, children)` | Macro: concise tree construction |
