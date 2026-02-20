# Nim `std/htmlgen` Module — Complete Reference

> A compile-time HTML and XML code generator.  
> Every HTML tag is exposed as a macro that produces a `string` at compile time — no runtime allocations for the tag structure itself.

**Recommended import style:**

```nim
from std/htmlgen import nil
# then use fully qualified names:
let s = htmlgen.p("Hello, ", htmlgen.b("world"))
```

This avoids name collisions with common Nim identifiers like `div`, `object`, `var`, `template`, `s`, `i`, etc.

---

## Table of Contents

1. [How It Works](#how-it-works)
2. [Exported Constants](#exported-constants)
3. [Core Procedure](#core-procedure)
4. [HTML Tag Macros](#html-tag-macros)
   - [Document Structure](#document-structure)
   - [Metadata & Head](#metadata--head)
   - [Sectioning](#sectioning)
   - [Text Content](#text-content)
   - [Inline Text Semantics](#inline-text-semantics)
   - [Forms & Inputs](#forms--inputs)
   - [Tables](#tables)
   - [Embedded Content & Media](#embedded-content--media)
   - [Scripting & Templating](#scripting--templating)
   - [Interactive Elements](#interactive-elements)
   - [Ruby Annotations](#ruby-annotations)
   - [Legacy / Deprecated](#legacy--deprecated)
5. [MathML Tag Macros](#mathml-tag-macros)
6. [Attribute Rules](#attribute-rules)
7. [Complete Tag Reference Table](#complete-tag-reference-table)

---

## How It Works

Each macro accepts:
- **Named arguments** (`attr = "value"`) → become HTML attributes
- **Positional arguments** (strings, other macro calls) → become child content

```nim
# Syntax:  tagName(attr1="val", attr2="val", child1, child2, ...)
echo a(href="https://nim-lang.org", "Nim")
# → <a href="https://nim-lang.org">Nim</a>

echo p(class="intro", "Hello ", b("world"))
# → <p class="intro">Hello <b>world</b></p>
```

**Leaf elements** (void elements like `<br>`, `<img>`, `<input>`, etc.) cannot have child content — only attributes — and produce self-closing tags:

```nim
echo br()
# → <br />

echo img(src="photo.jpg", alt="A photo")
# → <img src="photo.jpg" alt="A photo" />
```

The module performs **compile-time attribute validation**: passing an unknown attribute for a tag causes a compile error.  
`data-*` attributes are always allowed on any element.

```nim
echo p(`data-value`="42", "text")
# → <p data-value="42">text</p>
```

---

## Exported Constants

### `coreAttr`

```nim
const coreAttr*: string
```

Space-separated string listing the HTML core (global) attributes:  
`accesskey`, `class`, `contenteditable`, `dir`, `hidden`, `id`, `lang`, `spellcheck`, `style`, `tabindex`, `title`, `translate`.

---

### `eventAttr`

```nim
const eventAttr*: string
```

Space-separated string listing all standard HTML event handler attributes:  
`onabort`, `onblur`, `oncancel`, `oncanplay`, `oncanplaythrough`, `onchange`, `onclick`, `oncuechange`, `ondblclick`, `ondurationchange`, `onemptied`, `onended`, `onerror`, `onfocus`, `oninput`, `oninvalid`, `onkeydown`, `onkeypress`, `onkeyup`, `onload`, `onloadeddata`, `onloadedmetadata`, `onloadstart`, `onmousedown`, `onmouseenter`, `onmouseleave`, `onmousemove`, `onmouseout`, `onmouseover`, `onmouseup`, `onmousewheel`, `onpause`, `onplay`, `onplaying`, `onprogress`, `onratechange`, `onreset`, `onresize`, `onscroll`, `onseeked`, `onseeking`, `onselect`, `onshow`, `onstalled`, `onsubmit`, `onsuspend`, `ontimeupdate`, `ontoggle`, `onvolumechange`, `onwaiting`.

---

### `ariaAttr`

```nim
const ariaAttr*: string
```

Contains `role` — the HTML ARIA role attribute.

---

### `commonAttr`

```nim
const commonAttr*: string = coreAttr & eventAttr & ariaAttr
```

Combination of all three above. Used internally as the default allowed attribute set for most tags.

---

## Core Procedure

### `xmlCheckedTag`

```nim
proc xmlCheckedTag*(argsList: NimNode, tag: string,
    optAttr = "", reqAttr = "", isLeaf = false): NimNode
```

The underlying building block used by all tag macros. Use this to define **custom XML/HTML tags** not already present in the module.

Parameters:
- `argsList` — the macro's argument list (`NimNode`)
- `tag` — the tag name string (e.g. `"section"`)
- `optAttr` — space-separated list of optional attributes
- `reqAttr` — space-separated list of required attributes (compile error if missing)
- `isLeaf` — `true` produces a self-closing tag (`<tag />`), `false` produces `<tag>...</tag>`

```nim
import std/[htmlgen, macros]

macro myCustomTag*(e: varargs[untyped]): untyped =
  result = xmlCheckedTag(e, "x-custom", "color size" & commonAttr)

echo myCustomTag(color="red", "Hello")
# → <x-custom color="red">Hello</x-custom>
```

---

## HTML Tag Macros

All macros share the same signature:

```nim
macro tagName*(e: varargs[untyped]): untyped
```

They return a `string`.

---

### Document Structure

#### `html`
Generates `<html>`. Optional attribute: `xmlns`.
```nim
echo html(xmlns="http://www.w3.org/1999/xhtml", head(""), body("content"))
# → <html xmlns="http://www.w3.org/1999/xhtml"><head></head><body>content</body></html>
```

#### `head`
Generates `<head>`. Wraps metadata elements.
```nim
echo head(title("My Page"), meta(charset="utf-8"))
```

#### `body`
Generates `<body>`. Extra attributes: `onafterprint`, `onbeforeprint`, `onbeforeunload`, `onhashchange`, `onmessage`, `onoffline`, `ononline`, `onpagehide`, `onpageshow`, `onpopstate`, `onstorage`, `onunload`.
```nim
echo body(onload="init()", p("Hello"))
```

---

### Metadata & Head

#### `title`
Generates `<title>`.
```nim
echo title("My Page")  # → <title>My Page</title>
```

#### `base`
Leaf element. Optional attributes: `href`, `target`.
```nim
echo base(href="https://example.com/", target="_blank")
# → <base href="https://example.com/" target="_blank" />
```

#### `link`
Leaf element. Optional attributes: `href`, `crossorigin`, `rel`, `media`, `hreflang`, `type`, `sizes`.
```nim
echo link(rel="stylesheet", href="style.css")
# → <link rel="stylesheet" href="style.css" />
```

#### `meta`
Leaf element. Optional attributes: `name`, `http-equiv`, `content`, `charset`.
```nim
echo meta(charset="utf-8")
# → <meta charset="utf-8" />
echo meta(name="description", content="My site")
# → <meta name="description" content="My site" />
```

#### `style`
Generates `<style>`. Optional attributes: `media`, `type`.
```nim
echo style(type="text/css", "body { margin: 0 }")
```

---

### Sectioning

#### `article`
Generates `<article>`.

#### `aside`
Generates `<aside>`.

#### `footer`
Generates `<footer>`.

#### `header`
Generates `<header>`.

#### `h1` .. `h6`
Heading levels 1–6.
```nim
echo h1("Main Title")    # → <h1>Main Title</h1>
echo h2("Subtitle")      # → <h2>Subtitle</h2>
```

#### `main`
Generates `<main>`.

#### `nav`
Generates `<nav>`.

#### `section`
Generates `<section>`.

---

### Text Content

#### `blockquote`
Generates `<blockquote>`. Optional attribute: `cite`.
```nim
echo blockquote(cite="https://example.com", "A wise quote.")
```

#### `dd`
Generates `<dd>` (description list definition).

#### `div`
Generates `<div>`. Note: must be called as `` `div` `` or via qualified name to avoid the Nim keyword.
```nim
echo `div`(class="container", p("text"))
```

#### `dl`
Generates `<dl>` (description list).

#### `dt`
Generates `<dt>` (description term).

#### `figcaption`
Generates `<figcaption>`.

#### `figure`
Generates `<figure>`.
```nim
echo figure(img(src="cat.jpg", alt="Cat"), figcaption("A cat"))
```

#### `hr`
Leaf element, no arguments.
```nim
echo hr()  # → <hr />
```

#### `li`
Generates `<li>`. Optional attribute: `value`.

#### `ol`
Generates `<ol>`. Optional attributes: `reversed`, `start`, `type`.
```nim
echo ol(start="3", li("Third"), li("Fourth"))
```

#### `p`
Generates `<p>`.
```nim
echo p(class="note", "This is a paragraph.")
```

#### `pre`
Generates `<pre>`.

#### `ul`
Generates `<ul>`.
```nim
echo ul(li("Apple"), li("Banana"), li("Cherry"))
# → <ul><li>Apple</li><li>Banana</li><li>Cherry</li></ul>
```

---

### Inline Text Semantics

#### `a`
Generates `<a>`. Optional attributes: `href`, `target`, `download`, `rel`, `hreflang`, `type`.
```nim
echo a(href="https://nim-lang.org", target="_blank", "Nim Language")
# → <a href="https://nim-lang.org" target="_blank">Nim Language</a>
```

#### `abbr`
Generates `<abbr>` (abbreviation).
```nim
echo abbr(title="HyperText Markup Language", "HTML")
```

#### `address`
Generates `<address>`.

#### `b`
Generates `<b>` (bold/attention).

#### `bdi`
Generates `<bdi>` (bidirectional isolation).

#### `bdo`
Generates `<bdo>` (bidirectional override).

#### `br`
Leaf element, no content.
```nim
echo br()  # → <br />
```

#### `cite`
Generates `<cite>`.

#### `code`
Generates `<code>`.
```nim
echo code("echo \"Hello\"")
```

#### `data`
Generates `<data>`. Optional attribute: `value`.

#### `dfn`
Generates `<dfn>` (definition).

#### `em`
Generates `<em>` (emphasis).

#### `i`
Generates `<i>`. Note: use `` `i` `` if it conflicts.

#### `kbd`
Generates `<kbd>` (keyboard input).
```nim
echo kbd("Ctrl+C")
```

#### `mark`
Generates `<mark>`.

#### `q`
Generates `<q>` (inline quotation). Optional attribute: `cite`.

#### `s`
Generates `<s>` (strikethrough). Use `` `s` ``.

#### `samp`
Generates `<samp>` (sample output).

#### `small`
Generates `<small>`.

#### `span`
Generates `<span>`.
```nim
echo span(class="highlight", "important text")
```

#### `strong`
Generates `<strong>`.

#### `sub`
Generates `<sub>` (subscript).

#### `sup`
Generates `<sup>` (superscript).
```nim
echo p("H", sub("2"), "O and x", sup("2"))
```

#### `time`
Generates `<time>`. Optional attribute: `datetime`.
```nim
echo time(datetime="2024-01-01", "New Year's Day")
```

#### `u`
Generates `<u>` (underline/unarticulated annotation).

#### `var`
Generates `<var>`. Use `` `var` ``.

#### `wbr`
Leaf element. Word break opportunity.
```nim
echo wbr()  # → <wbr />
```

---

### Forms & Inputs

#### `form`
Generates `<form>`. Optional attributes: `accept-charset`, `action`, `autocomplete`, `enctype`, `method`, `name`, `novalidate`, `target`.
```nim
echo form(action="/submit", `method`="post",
  input(`type`="text", name="username"),
  button(`type`="submit", "Send")
)
```

#### `button`
Generates `<button>`. Optional attributes: `autofocus`, `disabled`, `form`, `formaction`, `formenctype`, `formmethod`, `formnovalidate`, `formtarget`, `menu`, `name`, `type`, `value`.

#### `datalist`
Generates `<datalist>`.

#### `fieldset`
Generates `<fieldset>`. Optional attributes: `disabled`, `form`, `name`.

#### `input`
Leaf element. Optional attributes: `accept`, `alt`, `autocomplete`, `autofocus`, `checked`, `dirname`, `disabled`, `form`, `formaction`, `formenctype`, `formmethod`, `formnovalidate`, `formtarget`, `height`, `inputmode`, `list`, `max`, `maxlength`, `min`, `minlength`, `multiple`, `name`, `pattern`, `placeholder`, `readonly`, `required`, `size`, `src`, `step`, `type`, `value`, `width`.
```nim
echo input(`type`="email", name="email", placeholder="you@example.com", required="required")
# → <input type="email" name="email" placeholder="you@example.com" required="required" />
```

#### `keygen`
Generates `<keygen>`. Optional attributes: `autofocus`, `challenge`, `disabled`, `form`, `keytype`, `name`.

#### `label`
Generates `<label>`. Optional attributes: `form`, `for`.
```nim
echo label(`for`="email", "Email address:")
```

#### `legend`
Generates `<legend>`.

#### `meter`
Generates `<meter>`. Optional attributes: `value`, `min`, `max`, `low`, `high`, `optimum`.
```nim
echo meter(value="70", min="0", max="100", "70%")
```

#### `optgroup`
Generates `<optgroup>`. Required attribute: `label`. Optional: `disabled`.

#### `option`
Generates `<option>`. Optional attributes: `disabled`, `label`, `selected`, `value`.

#### `output`
Generates `<output>`. Optional attributes: `for`, `form`, `name`.

#### `progress`
Generates `<progress>`. Optional attributes: `value`, `max`.
```nim
echo progress(value="40", max="100")
```

#### `select`
Generates `<select>`. Optional attributes: `autofocus`, `disabled`, `form`, `multiple`, `name`, `required`, `size`.
```nim
echo select(name="color",
  option(value="red", "Red"),
  option(value="blue", "Blue")
)
```

#### `textarea`
Generates `<textarea>`. Optional attributes: `autocomplete`, `autofocus`, `cols`, `dirname`, `disabled`, `form`, `inputmode`, `maxlength`, `minlength`, `name`, `placeholder`, `readonly`, `required`, `rows`, `wrap`.

---

### Tables

#### `caption`
Generates `<caption>`.

#### `col`
Leaf element. Optional attribute: `span`.

#### `colgroup`
Generates `<colgroup>`. Optional attribute: `span`.

#### `table`
Generates `<table>`. Optional attributes: `border`, `sortable`.
```nim
echo table(
  thead(tr(th("Name"), th("Age"))),
  tbody(
    tr(td("Alice"), td("30")),
    tr(td("Bob"),   td("25"))
  )
)
```

#### `tbody`
Generates `<tbody>`.

#### `td`
Generates `<td>`. Optional attributes: `colspan`, `rowspan`, `headers`.

#### `tfoot`
Generates `<tfoot>`.

#### `th`
Generates `<th>`. Optional attributes: `colspan`, `rowspan`, `headers`, `abbr`, `scope`, `axis`, `sorted`.

#### `thead`
Generates `<thead>`.

#### `tr`
Generates `<tr>`.

---

### Embedded Content & Media

#### `area`
Leaf element. Required attribute: `alt`. Optional attributes: `coords`, `download`, `href`, `hreflang`, `rel`, `shape`, `target`, `type`.

#### `audio`
Generates `<audio>`. Optional attributes: `src`, `crossorigin`, `preload`, `autoplay`, `mediagroup`, `loop`, `muted`, `controls`.
```nim
echo audio(src="sound.mp3", controls="controls")
```

#### `canvas`
Generates `<canvas>`. Optional attributes: `width`, `height`.

#### `embed`
Leaf element. Optional attributes: `src`, `type`, `height`, `width`.

#### `iframe`
Generates `<iframe>`. Optional attributes: `src`, `srcdoc`, `name`, `sandbox`, `width`, `height`, `loading`.

#### `img`
Leaf element. Required attributes: `src`, `alt`. Optional: `crossorigin`, `usemap`, `ismap`, `height`, `width`, `loading`.
```nim
echo img(src="logo.png", alt="Logo", width="200", height="80")
# → <img src="logo.png" alt="Logo" width="200" height="80" />
```

#### `map`
Generates `<map>`. Optional attribute: `name`.

#### `object`
Generates `<object>`. Use `` `object` ``. Optional attributes: `data`, `type`, `typemustmatch`, `name`, `usemap`, `form`, `width`, `height`.

#### `param`
Leaf element. Required attributes: `name`, `value`.

#### `picture`
Generates `<picture>`.

#### `portal`
Generates `<portal>`. Optional attributes: `width`, `height`, `type`, `src`, `disabled`.

#### `source`
Leaf element. Required attribute: `src`. Optional: `type`.
```nim
echo video(
  source(src="video.webm", `type`="video/webm"),
  source(src="video.mp4",  `type`="video/mp4"),
  "Your browser does not support video."
)
```

#### `track`
Leaf element. Required attribute: `src`. Optional: `kind`, `srclang`, `label`, `default`.

#### `video`
Generates `<video>`. Optional attributes: `src`, `crossorigin`, `poster`, `preload`, `autoplay`, `mediagroup`, `loop`, `muted`, `controls`, `width`, `height`.

---

### Scripting & Templating

#### `noscript`
Generates `<noscript>`.

#### `script`
Generates `<script>`. Optional attributes: `src`, `type`, `charset`, `async`, `defer`, `crossorigin`.
```nim
echo script(src="app.js", defer="defer")
```

#### `slot`
Generates `<slot>`.

#### `template`
Generates `<template>`. Use `` `template` ``.

---

### Interactive Elements

#### `details`
Generates `<details>`. Optional attribute: `open`.
```nim
echo details(open="open",
  summary("Click to expand"),
  p("Hidden content here.")
)
```

#### `dialog`
Generates `<dialog>`. Optional attribute: `open`.

#### `menu`
Generates `<menu>`.

#### `summary`
Generates `<summary>`.

---

### Ruby Annotations

#### `rb`
Generates `<rb>` (ruby base).

#### `rp`
Generates `<rp>` (ruby parenthesis fallback).

#### `rt`
Generates `<rt>` (ruby text).

#### `rtc`
Generates `<rtc>` (ruby text container).

#### `ruby`
Generates `<ruby>`.
```nim
echo ruby("漢", rp("("), rt("Kan"), rp(")"))
```

---

### Legacy / Deprecated

These tags exist in the module for completeness but are deprecated in HTML5. Avoid in new projects.

#### `big`
Generates `<big>`.

#### `center`
Generates `<center>`.

#### `marquee`
Generates `<marquee>`. Attributes: `behavior`, `bgcolor`, `direction`, `height`, `hspace`, `loop`, `scrollamount`, `scrolldelay`, `truespeed`, `vspace`, `width`, `onbounce`, `onfinish`, `onstart`.

#### `tt`
Generates `<tt>` (teletype text).

---

## MathML Tag Macros

MathML is part of HTML5 (ISO/IEC 40314:2015). All MathML macros accept `mathbackground`, `mathcolor`, `href`, and `commonAttr` as common attributes.

```nim
echo math(
  semantics(
    mrow(
      msup(mi("x"), mn("42"))
    )
  )
)
# → <math><semantics><mrow><msup><mi>x</mi><mn>42</mn></msup></mrow></semantics></math>
```

| Macro | HTML element | Extra attributes |
|---|---|---|
| `math` | `<math>` | `overflow` |
| `maction` | `<maction>` | `actiontype selection` |
| `menclose` | `<menclose>` | `notation` |
| `merror` | `<merror>` | — |
| `mfenced` | `<mfenced>` | `open close separators` |
| `mfrac` | `<mfrac>` | `linethickness numalign` |
| `mglyph` | `<mglyph>` | `src valign` |
| `mi` | `<mi>` | `mathsize mathvariant` |
| `mlabeledtr` | `<mlabeledtr>` | `columnalign groupalign rowalign` |
| `mmultiscripts` | `<mmultiscripts>` | `subscriptshift superscriptshift` |
| `mn` | `<mn>` | `mathsize mathvariant` |
| `mo` | `<mo>` | `fence form largeop lspace mathsize mathvariant movablelimits rspace separator stretchy symmetric` |
| `mover` | `<mover>` | `accent` |
| `mpadded` | `<mpadded>` | `depth lspace voffset` |
| `mphantom` | `<mphantom>` | — |
| `mroot` | `<mroot>` | — |
| `mrow` | `<mrow>` | — |
| `ms` | `<ms>` | `lquote mathsize mathvariant rquote` |
| `mspace` | `<mspace>` | `linebreak` |
| `msqrt` | `<msqrt>` | — |
| `mstyle` | `<mstyle>` | `decimalpoint displaystyle infixlinebreakstyle scriptlevel scriptminsize scriptsizemultiplier` |
| `msub` | `<msub>` | `subscriptshift` |
| `msubsup` | `<msubsup>` | `subscriptshift superscriptshift` |
| `msup` | `<msup>` | `superscriptshift` |
| `mtable` | `<mtable>` | `align alignmentscope columnalign columnlines columnspacing columnwidth displaystyle equalcolumns equalrows frame framespacing groupalign rowalign rowlines rowspacing side width` |
| `mtd` | `<mtd>` | `columnalign columnspan groupalign rowalign rowspan` |
| `mtext` | `<mtext>` | `mathsize mathvariant` |
| `munder` | `<munder>` | `accentunder align` |
| `munderover` | `<munderover>` | `accentunder accent align` |
| `semantics` | `<semantics>` | `definitionURL encoding cd src` |
| `annotation` | `<annotation>` | `definitionURL encoding cd src` |
| `` `annotation-xml` `` | `<annotation-xml>` | `definitionURL encoding cd src` |

---

## Attribute Rules

| Rule | Details |
|---|---|
| Unknown attribute | Compile-time error |
| `data-*` attributes | Always allowed on any element |
| Required attributes | Missing → compile-time error |
| Nim keyword tags | Wrap in backticks: `` `div` ``, `` `var` ``, `` `object` ``, `` `template` ``, `` `method` `` |
| Hyphenated attributes | Wrap in backticks: `` `accept-charset` ``, `` `http-equiv` ``, `` `annotation-xml` `` |

---

## Complete Tag Reference Table

| Macro | Tag | Leaf? | Required Attrs | Notable Optional Attrs |
|---|---|:---:|---|---|
| `a` | `<a>` | | | `href target download rel hreflang type` |
| `abbr` | `<abbr>` | | | |
| `address` | `<address>` | | | |
| `area` | `<area>` | ✓ | `alt` | `coords href shape target` |
| `article` | `<article>` | | | |
| `aside` | `<aside>` | | | |
| `audio` | `<audio>` | | | `src controls autoplay loop muted` |
| `b` | `<b>` | | | |
| `base` | `<base>` | ✓ | | `href target` |
| `bdi` | `<bdi>` | | | |
| `bdo` | `<bdo>` | | | |
| `big` | `<big>` | | | *(deprecated)* |
| `blockquote` | `<blockquote>` | | | `cite` |
| `body` | `<body>` | | | page lifecycle events |
| `br` | `<br>` | ✓ | | |
| `button` | `<button>` | | | `type name value disabled` |
| `canvas` | `<canvas>` | | | `width height` |
| `caption` | `<caption>` | | | |
| `center` | `<center>` | | | *(deprecated)* |
| `cite` | `<cite>` | | | |
| `code` | `<code>` | | | |
| `col` | `<col>` | ✓ | | `span` |
| `colgroup` | `<colgroup>` | | | `span` |
| `data` | `<data>` | | | `value` |
| `datalist` | `<datalist>` | | | |
| `dd` | `<dd>` | | | |
| `del` | `<del>` | | | `cite datetime` |
| `details` | `<details>` | | | `open` |
| `dfn` | `<dfn>` | | | |
| `dialog` | `<dialog>` | | | `open` |
| `` `div` `` | `<div>` | | | |
| `dl` | `<dl>` | | | |
| `dt` | `<dt>` | | | |
| `em` | `<em>` | | | |
| `embed` | `<embed>` | ✓ | | `src type width height` |
| `fieldset` | `<fieldset>` | | | `disabled form name` |
| `figcaption` | `<figcaption>` | | | |
| `figure` | `<figure>` | | | |
| `footer` | `<footer>` | | | |
| `form` | `<form>` | | | `action method enctype name` |
| `h1`–`h6` | `<h1>`–`<h6>` | | | |
| `head` | `<head>` | | | |
| `header` | `<header>` | | | |
| `hr` | `<hr>` | ✓ | | |
| `html` | `<html>` | | | `xmlns` |
| `i` | `<i>` | | | |
| `iframe` | `<iframe>` | | | `src srcdoc name sandbox width height loading` |
| `img` | `<img>` | ✓ | `src alt` | `width height loading crossorigin` |
| `input` | `<input>` | ✓ | | `type name value placeholder required` |
| `ins` | `<ins>` | | | `cite datetime` |
| `kbd` | `<kbd>` | | | |
| `keygen` | `<keygen>` | | | `name keytype disabled` |
| `label` | `<label>` | | | `for form` |
| `legend` | `<legend>` | | | |
| `li` | `<li>` | | | `value` |
| `link` | `<link>` | ✓ | | `rel href type media` |
| `main` | `<main>` | | | |
| `map` | `<map>` | | | `name` |
| `mark` | `<mark>` | | | |
| `marquee` | `<marquee>` | | | *(deprecated)* |
| `menu` | `<menu>` | | | |
| `meta` | `<meta>` | ✓ | | `charset name content http-equiv` |
| `meter` | `<meter>` | | | `value min max low high optimum` |
| `nav` | `<nav>` | | | |
| `noscript` | `<noscript>` | | | |
| `` `object` `` | `<object>` | | | `data type name width height` |
| `ol` | `<ol>` | | | `reversed start type` |
| `optgroup` | `<optgroup>` | | `label` | `disabled` |
| `option` | `<option>` | | | `value selected disabled label` |
| `output` | `<output>` | | | `for form name` |
| `p` | `<p>` | | | |
| `param` | `<param>` | ✓ | `name value` | |
| `picture` | `<picture>` | | | |
| `portal` | `<portal>` | | | `src type width height disabled` |
| `pre` | `<pre>` | | | |
| `progress` | `<progress>` | | | `value max` |
| `q` | `<q>` | | | `cite` |
| `rb` | `<rb>` | | | |
| `rp` | `<rp>` | | | |
| `rt` | `<rt>` | | | |
| `rtc` | `<rtc>` | | | |
| `ruby` | `<ruby>` | | | |
| `s` | `<s>` | | | |
| `samp` | `<samp>` | | | |
| `script` | `<script>` | | | `src type async defer crossorigin` |
| `section` | `<section>` | | | |
| `select` | `<select>` | | | `name multiple size disabled` |
| `slot` | `<slot>` | | | |
| `small` | `<small>` | | | |
| `source` | `<source>` | ✓ | `src` | `type` |
| `span` | `<span>` | | | |
| `strong` | `<strong>` | | | |
| `style` | `<style>` | | | `media type` |
| `sub` | `<sub>` | | | |
| `summary` | `<summary>` | | | |
| `sup` | `<sup>` | | | |
| `table` | `<table>` | | | `border sortable` |
| `tbody` | `<tbody>` | | | |
| `td` | `<td>` | | | `colspan rowspan headers` |
| `` `template` `` | `<template>` | | | |
| `textarea` | `<textarea>` | | | `name rows cols placeholder required` |
| `tfoot` | `<tfoot>` | | | |
| `th` | `<th>` | | | `colspan rowspan scope abbr sorted` |
| `thead` | `<thead>` | | | |
| `time` | `<time>` | | | `datetime` |
| `title` | `<title>` | | | |
| `tr` | `<tr>` | | | |
| `track` | `<track>` | ✓ | `src` | `kind srclang label default` |
| `tt` | `<tt>` | | | *(deprecated)* |
| `u` | `<u>` | | | |
| `ul` | `<ul>` | | | |
| `` `var` `` | `<var>` | | | |
| `video` | `<video>` | | | `src controls width height poster` |
| `wbr` | `<wbr>` | ✓ | | |
