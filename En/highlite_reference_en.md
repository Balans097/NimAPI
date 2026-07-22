# highlite — module reference

> **Import:** `import std/highlite`
> **Scope:** token-by-token parsing of source code in several programming
> languages — Nim, C, C++, C#, Java, Python — as well as YAML and
> Cmd/Console sessions — for syntax highlighting.

The module does not build a parse tree and does not validate the code — it
only cuts the input string into a sequence of tokens (keyword, identifier,
number, string literal, comment, operator, etc.), returning for each token its
`TokenClass` and its bounds within the source buffer. The module's general
convention: the same `GeneralTokenizer` object is initialized once with
`initGeneralTokenizer`, and then `getNextToken` is called in a loop, advancing
the internal position on every step and filling in the `kind`, `start`,
`length` fields. The specific language is passed as a separate
`lang: SourceLanguage` parameter on every call — this lets the same tokenizer
be reused for several nested languages (for example, for a `Cmd` command
inside a `Console` session).

---

## Table of contents

I. [Types and constants](#types-and-constants)
&nbsp;&nbsp;&nbsp;1. [`SourceLanguage`](#sourcelanguage)
&nbsp;&nbsp;&nbsp;2. [`TokenClass`](#tokenclass)
&nbsp;&nbsp;&nbsp;3. [`GeneralTokenizer`](#generaltokenizer)
&nbsp;&nbsp;&nbsp;4. [Lookup tables](#lookup-tables)

II. [Resolving a language from a string](#resolving-a-language-from-a-string)
&nbsp;&nbsp;&nbsp;1. [`getSourceLanguage`](#getsourcelanguage)

III. [Initializing the tokenizer](#initializing-the-tokenizer)
&nbsp;&nbsp;&nbsp;1. [`initGeneralTokenizer` (from `cstring`)](#initgeneraltokenizer-from-cstring)
&nbsp;&nbsp;&nbsp;2. [`initGeneralTokenizer` (from `string`)](#initgeneraltokenizer-from-string)
&nbsp;&nbsp;&nbsp;3. [`deinitGeneralTokenizer`](#deinitgeneraltokenizer)

IV. [Retrieving tokens](#retrieving-tokens)
&nbsp;&nbsp;&nbsp;1. [`getNextToken`](#getnexttoken)
&nbsp;&nbsp;&nbsp;2. [`tokenize`](#tokenize)

V. [Practical recipes](#practical-recipes)
&nbsp;&nbsp;&nbsp;1. [Breaking a line of code into colored fragments](#breaking-a-line-of-code-into-colored-fragments)
&nbsp;&nbsp;&nbsp;2. [Counting comment lines in a file](#counting-comment-lines-in-a-file)
&nbsp;&nbsp;&nbsp;3. [Extracting all identifiers and keywords](#extracting-all-identifiers-and-keywords)
&nbsp;&nbsp;&nbsp;4. [Resolving the language from a file extension and highlighting on the fly](#resolving-the-language-from-a-file-extension-and-highlighting-on-the-fly)
&nbsp;&nbsp;&nbsp;5. [Highlighting a command with its output (Console)](#highlighting-a-command-with-its-output-console)

VI. [Quick reference table](#quick-reference-table)

VII. [Summary: which procedure to pick](#summary-which-procedure-to-pick)

---

## Types and constants

### `SourceLanguage`

```nim
type
  SourceLanguage* = enum
    langNone, langNim, langCpp, langCsharp, langC, langJava,
    langYaml, langPython, langCmd, langConsole
```

**What it does.** An enumeration of the supported parsing languages.
`langNone` means "no language was resolved" and is used as the default value
for an uninitialized tokenizer; it must not be passed directly to
`getNextToken` (see the section on `getNextToken`).

**Implementation notes.** The order of the enum values is not arbitrary — it
must line up element-for-element with the row order in the
`sourceLanguageToStr` and `sourceLanguageToAlpha` constants (see "Lookup
tables"): the enum value is used as an array index, so any mismatch in order
silently breaks the mapping between a language's name and its enum value.

- **Parameters:** the type takes no parameters — it is a set of named
  constants.

```nim
var lang: SourceLanguage
lang = langPython
echo lang        # prints langPython
echo ord(lang)    # prints 7 — its ordinal position in the enum
```

---

### `TokenClass`

```nim
type
  TokenClass* = enum
    gtEof, gtNone, gtWhitespace, gtDecNumber, gtBinNumber, gtHexNumber,
    gtOctNumber, gtFloatNumber, gtIdentifier, gtKeyword, gtStringLit,
    gtLongStringLit, gtCharLit, gtEscapeSequence,
    gtOperator, gtPunctuation, gtComment, gtLongComment, gtRegularExpression,
    gtTagStart, gtTagEnd, gtKey, gtValue, gtRawData, gtAssembler,
    gtPreprocessor, gtDirective, gtCommand, gtRule, gtHyperlink, gtLabel,
    gtReference, gtPrompt, gtProgramOutput, gtProgram, gtOption, gtOther
```

**What it does.** Enumerates every token class that `getNextToken` can
return. Not every language uses all of the values: for example,
`gtTagStart`/`gtLabel`/`gtReference` occur only when parsing YAML, while
`gtProgram`/`gtOption`/`gtPrompt` occur only when parsing `Cmd`/`Console`.
`gtEof` marks the end of the buffer and terminates any parsing loop; `gtNone`
means "a token of an undetermined class" (for example, a single character
that doesn't fit anything else).

**Implementation notes.** Having one shared set of classes for every language
lets you write a single piece of highlighting code without branching by
language: the caller reacts to a `TokenClass`, not to which internal
`langNextToken` procedure produced it.

- **Parameters:** the type takes no parameters.

```nim
let tokenClass = gtKeyword
case tokenClass
of gtKeyword:
  echo "This is a language keyword"
of gtComment, gtLongComment:
  echo "This is a comment"
else:
  echo "Some other token class"
```

---

### `GeneralTokenizer`

```nim
type
  GeneralTokenizer* = object of RootObj
    kind*: TokenClass
    start*, length*: int
    buf: cstring
    pos: int
    state: TokenClass
    lang: SourceLanguage
```

**What it does.** Holds all the state of parsing a single buffer: a pointer
to the buffer itself (`buf`), the current read position (`pos`), the class
and bounds of the last token found (`kind`, `start`, `length`), plus `state` —
the internal "carried-over" parsing state between calls (for example, "we are
inside a multi-line comment" or "we are inside a string literal, and an
escape sequence hasn't finished yet").

**Implementation notes.** Only three fields are exposed publicly (via `*`):
`kind`, `start`, `length` — this is all a consumer of the module needs after
each `getNextToken` call. The `buf`, `pos`, `state`, `lang` fields are kept
private: together they form an internal line-parsing state machine, and its
transitions (for example, "enter a string — leave it on an escape sequence —
return to the string") are not meant to be read or written directly from
outside the module.

- **Fields:**
  - `kind*: TokenClass` — the class of the token found by the last call to `getNextToken`;
  - `start*, length*: int` — the start and length of the found token within the source buffer (not within a copied string!);
  - `buf: cstring` (private) — the buffer being parsed;
  - `pos: int` (private) — the current read position;
  - `state: TokenClass` (private) — the "suspended" state between tokens;
  - `lang: SourceLanguage` (private) — the language used on the last `getNextToken` call.

```nim
var g: GeneralTokenizer
initGeneralTokenizer(g, "let x = 1")
getNextToken(g, langNim)
echo g.kind    # prints gtKeyword ("let" is a keyword)
echo g.start   # prints 0
echo g.length  # prints 3
```

---

### Lookup tables

```nim
const
  sourceLanguageToStr*: array[SourceLanguage, string] = ["none",
    "Nim", "C++", "C#", "C", "Java", "Yaml", "Python", "Cmd", "Console"]
  sourceLanguageToAlpha*: array[SourceLanguage, string] = ["none",
    "Nim", "cpp", "csharp", "C", "Java", "Yaml", "Python", "Cmd", "Console"]
  tokenClassToStr*: array[TokenClass, string] = ["Eof", "None", "Whitespace",
    "DecNumber", "BinNumber", "HexNumber", "OctNumber", "FloatNumber",
    "Identifier", "Keyword", "StringLit", "LongStringLit", "CharLit",
    "EscapeSequence", "Operator", "Punctuation", "Comment", "LongComment",
    "RegularExpression", "TagStart", "TagEnd", "Key", "Value", "RawData",
    "Assembler", "Preprocessor", "Directive", "Command", "Rule", "Hyperlink",
    "Label", "Reference", "Prompt", "ProgramOutput",
    "program", "option", "Other"]
```

**What it does.** Three arrays, indexed by the enumerations themselves
(rather than by plain integers), giving a human-readable string name for
each `SourceLanguage` or `TokenClass` value. `sourceLanguageToStr` gives a
"pretty" display name ("C++", "C#"), while `sourceLanguageToAlpha` gives a
letters-only variant suitable, for example, as a CSS class name or as part of
an RST/HTML role ("cpp", "csharp").

**Implementation notes.** Because the arrays are declared as
`array[SourceLanguage, string]` and `array[TokenClass, string]`, the Nim
compiler itself checks at compile time that the number of strings in the
initializer exactly matches the number of enum values — a mismatch (a missing
or an extra row) fails to compile rather than showing up as a runtime bug.

- **Parameters:** constants, indexed by the corresponding enum value.

```nim
echo sourceLanguageToStr[langCsharp]    # prints "C#"
echo sourceLanguageToAlpha[langCsharp]  # prints "csharp"
echo tokenClassToStr[gtKeyword]         # prints "Keyword"
```

---

## Resolving a language from a string

### `getSourceLanguage`

```nim
proc getSourceLanguage*(name: string): SourceLanguage
```

**What it does.** Given an arbitrarily typed language name (comparison is
case-insensitive and ignores dashes/underscores, via `cmpIgnoreStyle`),
returns the matching `SourceLanguage` value. If the name is found in neither
the "pretty" (`sourceLanguageToStr`) nor the "letters-only"
(`sourceLanguageToAlpha`) arrays, `langNone` is returned.

**Implementation notes.** The search is linear (O(n) over the number of
languages, but there are only about ten of them, so this is negligible) and
checks both lookup arrays in turn for each language — first the "pretty"
name, then the "letters-only" one. Iteration deliberately starts after
`langNone` (`succ(low(SourceLanguage))`), since `langNone` is itself the
"nothing found" result, not something you can type in as a name.

- **Parameters:**
  - `name: string` — the language name in any case or spelling ("C++", "c++", "CPP", "cpp" are all recognized).

```nim
echo getSourceLanguage("Nim")     # prints langNim
echo getSourceLanguage("c++")     # prints langCpp — matched via sourceLanguageToAlpha
echo getSourceLanguage("C#")      # prints langCsharp
echo getSourceLanguage("pascal")  # prints langNone — the language isn't supported
```

A practical use case — picking the highlighting language from a file
extension — is shown in the "Practical recipes" section (recipe 4).

---

## Initializing the tokenizer

### `initGeneralTokenizer` (from `cstring`)

```nim
proc initGeneralTokenizer*(g: var GeneralTokenizer, buf: cstring)
```

**What it does.** Resets all of the tokenizer's state in `g` and binds it to
the `buf` buffer. After the call, the tokenizer is ready for the first call
to `getNextToken`. The buffer is not copied — `g` only stores a reference to
the given `cstring`, so it must remain alive (not freed or reused) for the
entire lifetime of the tokenizer.

**Implementation notes.** `kind` and `state` are set to `low(TokenClass)`
(that is, `gtEof`), `lang` is set to `low(SourceLanguage)` (that is,
`langNone`), and `start`/`length`/`pos` are all reset to zero. Working with
`cstring` rather than `string` matters for the internal parsing: procedures
like `nimNextToken` read the buffer character by character by index all the
way up to the terminating null byte, which would be unsafe for a Nim string
slice without a guaranteed trailing `\0`.

- **Parameters:**
  - `g: var GeneralTokenizer` — the tokenizer to initialize (mutable);
  - `buf: cstring` — the buffer to parse; must outlive `g`.

```nim
var g: GeneralTokenizer
let buffer: cstring = "proc main() = discard"
initGeneralTokenizer(g, buffer)
getNextToken(g, langNim)
echo g.kind  # prints gtKeyword ("proc")
```

---

### `initGeneralTokenizer` (from `string`)

```nim
proc initGeneralTokenizer*(g: var GeneralTokenizer, buf: string)
```

**What it does.** A convenience wrapper around the previous overload: it
takes an ordinary Nim string and converts it to `cstring` itself before
calling through. For the vast majority of use cases, this is the variant the
caller actually wants.

**Implementation notes.** The implementation is literally a single
delegating call: `initGeneralTokenizer(g, cstring(buf))`. An important
implicit contract in Nim applies here: converting a `string` to `cstring`
does not allocate new memory — it is only valid while the original `buf`
string is alive; if the string goes out of scope before the tokenizer does,
subsequent calls to `getNextToken` will read freed memory.

- **Parameters:**
  - `g: var GeneralTokenizer` — the tokenizer to initialize (mutable);
  - `buf: string` — the source code to parse; must outlive the whole parsing loop.

```nim
var g: GeneralTokenizer
var source = "for i in 0..10: echo i"
initGeneralTokenizer(g, source)
getNextToken(g, langNim)
echo g.kind  # prints gtKeyword ("for")

# edge case — an empty string
var empty: GeneralTokenizer
initGeneralTokenizer(empty, "")
getNextToken(empty, langNim)
echo empty.kind  # prints gtEof — there are no tokens, the buffer ends immediately
```

---

### `deinitGeneralTokenizer`

```nim
proc deinitGeneralTokenizer*(g: var GeneralTokenizer)
```

**What it does.** Formally releases any resources tied to the tokenizer. In
practice the current implementation does nothing (`discard`) — the procedure
exists for symmetry with `initGeneralTokenizer` and as a future extension
point (should the type ever need resources that require explicit release —
for example, GC-unmanaged memory).

**Implementation notes.** An "empty" deinitializer like this is a common
pattern in Nim libraries: caller code that calls `deinitGeneralTokenizer`
after use will keep working unchanged even if a future version of the module
makes the procedure do something real.

- **Parameters:**
  - `g: var GeneralTokenizer` — the tokenizer being deinitialized.

```nim
var g: GeneralTokenizer
initGeneralTokenizer(g, "x")
getNextToken(g, langNim)
deinitGeneralTokenizer(g)  # currently no more than a formality
```

---

## Retrieving tokens

### `getNextToken`

```nim
proc getNextToken*(g: var GeneralTokenizer, lang: SourceLanguage)
```

**What it does.** The module's main workhorse: parses exactly one next token
starting at the current `g.pos`, writes the result into `g.kind`, `g.start`,
`g.length`, and advances `g.pos` past the found token. It must be called in a
loop until `g.kind` equals `gtEof`. Passing `lang = langNone` is a usage
error — it triggers `assert false` (an abnormal termination when assertions
are enabled).

**Implementation notes.** The procedure is a dispatcher (`case lang of ...`)
that delegates the actual parsing to one of the module's internal,
non-public (no `*`) per-language procedures: `nimNextToken`, `clikeNextToken`
(used for C/C++/C#/Java through thin wrappers
`cNextToken`/`cppNextToken`/`csharpNextToken`/`javaNextToken`, which simply
pass in a different keyword list and flags such as "has a preprocessor" or
"supports nested comments"), `yamlNextToken`, `pythonNextToken` (a thin
wrapper around `nimNextToken` with a Python keyword list), and `cmdNextToken`
(used both for `Cmd` and, with a "dollar prompt" flag, for `Console`). All of
these internal procedures share the same underlying idea, similar to a
traffic light with a memory: if a previous call left a multi-line literal or
comment unclosed, that fact is recorded in `g.state`, and the next call
checks `state` first rather than starting to read from scratch — this is how
constructs that can span multiple calls are parsed (for example, an escaped
sequence inside a string as its own `gtEscapeSequence` token).

- **Parameters:**
  - `g: var GeneralTokenizer` — a tokenizer whose buffer has already been initialized (mutable);
  - `lang: SourceLanguage` — the language whose rules govern parsing the next token; must not be `langNone`.

```nim
var g: GeneralTokenizer
initGeneralTokenizer(g, "let x = 42 # the answer")
while true:
  getNextToken(g, langNim)
  if g.kind == gtEof:
    break
  echo tokenClassToStr[g.kind]
# prints, in order:
# Keyword, Whitespace, Identifier, Whitespace, Operator, Whitespace,
# DecNumber, Whitespace, Comment

# error case — no language given
var g2: GeneralTokenizer
initGeneralTokenizer(g2, "x")
doAssertRaises(AssertionDefect):
  getNextToken(g2, langNone)
```

---

### `tokenize`

```nim
proc tokenize*(text: string, lang: SourceLanguage): seq[(string, TokenClass)]
```

**What it does.** A high-level wrapper over the combination of
"`initGeneralTokenizer` plus a `getNextToken` loop": it parses the whole
`text` string at once and returns a ready-made sequence of pairs — "(token
substring, token class)". Handy when you don't need step-by-step control
over parsing (as in the example above) and just want the full list of
tokens.

**Implementation notes.** Internally the procedure only keeps track of
`prevPos` — the end position of the previous token — and on each iteration
slices out `text[prevPos ..< g.pos]`, that is, exactly the substring of the
source text that corresponds to the token found by the last `getNextToken`
call. Note that, unlike working with a `GeneralTokenizer` directly, calling
code does not need to compute the slice from `g.start`/`g.length` itself —
that work is already done. Complexity is O(n) in the length of the text,
since every character of the input buffer is visited within exactly one
token.

- **Parameters:**
  - `text: string` — the full source code to parse;
  - `lang: SourceLanguage` — the parsing language; must not be `langNone`.

```nim
let tokens = tokenize("var y = 1", langNim)
for (tokenText, tokenClass) in tokens:
  echo "'", tokenText, "' -> ", tokenClassToStr[tokenClass]
# prints:
# 'var' -> Keyword
# ' ' -> Whitespace
# 'y' -> Identifier
# ' ' -> Whitespace
# '=' -> Operator
# ' ' -> Whitespace
# '1' -> DecNumber

# edge case — an empty string
echo len(tokenize("", langNim))  # prints 0 — there are no tokens at all
```

---

## Practical recipes

### Breaking a line of code into colored fragments

A typical highlighting task: map each token to a CSS class and assemble the
resulting HTML.

```nim
proc highlightToHtml(code: string, lang: SourceLanguage): string =
  var g: GeneralTokenizer
  initGeneralTokenizer(g, code)
  result = ""
  while true:
    getNextToken(g, lang)
    if g.kind == gtEof:
      break
    let fragment = substr(code, g.start, g.start + g.length - 1)
    add(result, "<span class=\"tok-")
    add(result, tokenClassToStr[g.kind])
    add(result, "\">")
    add(result, fragment)
    add(result, "</span>")

echo highlightToHtml("if x: echo x", langNim)
```

---

### Counting comment lines in a file

A combination of `tokenize` and counting newline characters inside tokens of
class `gtComment`/`gtLongComment`.

```nim
proc countCommentLines(text: string, lang: SourceLanguage): int =
  let tokens = tokenize(text, lang)
  result = 0
  for (fragment, class) in tokens:
    if class in {gtComment, gtLongComment}:
      inc(result, 1 + count(fragment, '\n'))

echo countCommentLines("# a single line\n# another one\nx = 1", langPython)
```

---

### Extracting all identifiers and keywords

Useful for building a name index (for example, "go to definition" in an
editor).

```nim
proc collectNames(text: string, lang: SourceLanguage): tuple[identifiers, keywords: seq[string]] =
  let tokens = tokenize(text, lang)
  for (fragment, class) in tokens:
    case class
    of gtIdentifier: add(result.identifiers, fragment)
    of gtKeyword: add(result.keywords, fragment)
    else: discard

let (idents, keywords) = collectNames("proc foo(bar: int) = discard", langNim)
echo idents    # prints @["foo", "bar", "int"]
echo keywords  # prints @["proc", "discard"]
```

---

### Resolving the language from a file extension and highlighting on the fly

A combination of `getSourceLanguage` with ordinary string logic for
determining a file's extension.

```nim
proc languageFromFileName(path: string): SourceLanguage =
  let parts = split(path, '.')
  if len(parts) < 2:
    return langNone
  case parts[^1]
  of "nim": getSourceLanguage("Nim")
  of "py": getSourceLanguage("Python")
  of "cpp", "cc", "cxx": getSourceLanguage("C++")
  of "cs": getSourceLanguage("C#")
  of "java": getSourceLanguage("Java")
  of "yaml", "yml": getSourceLanguage("Yaml")
  else: langNone

echo languageFromFileName("main.cpp")     # prints langCpp
echo languageFromFileName("script.rb")    # prints langNone — the language isn't supported
```

---

### Highlighting a command with its output (Console)

`Console` is a language for interactive sessions: lines starting with a `$`
prompt are parsed as a command (`Cmd`), the rest as program output.

```nim
let session = """$ echo "hello"
hello"""
let tokens = tokenize(session, langConsole)
for (fragment, class) in tokens:
  if class == gtProgramOutput:
    echo "program output: ", fragment
  elif class == gtPrompt:
    echo "prompt: ", fragment
```

---

## Quick reference table

| Task                                                              | Mutates `g` | Returns                          |
|--------------------------------------------------------------------|:---:|--------------------------------------|
| Resolve a `SourceLanguage` from an arbitrary name                   | —   | `SourceLanguage`                     |
| Start parsing a buffer (`cstring`)                                  | yes | (only initializes `g`)               |
| Start parsing a buffer (`string`)                                   | yes | (only initializes `g`)               |
| Formally finish working with the tokenizer                         | yes | (no effect in the current version)   |
| Parse a single next token, given the language                      | yes | (result is in `g.kind/start/length`) |
| Parse the entire string at once and get the full token list        | no (uses its own internal `g`) | `seq[(string, TokenClass)]` |
| Get a display string for a language or token class                 | —   | `string` (via `sourceLanguageToStr`/`sourceLanguageToAlpha`/`tokenClassToStr`) |

---

## Summary: which procedure to pick

- Need to resolve a `SourceLanguage` from a string entered by a user or read
  from configuration → `getSourceLanguage`.
- Need to parse code in one go and get a ready-made list of tokens without
  writing a loop or slicing yourself → `tokenize`.
- Need step-by-step parsing with full control (for example, to stop parsing
  before reaching the end, or to process tokens one at a time without
  accumulating the whole list in memory) → `initGeneralTokenizer` plus a
  `getNextToken` loop until `gtEof`.
- The buffer already exists as a `cstring` (for example, obtained from C
  interop) → `initGeneralTokenizer(g, buf: cstring)`; otherwise (an ordinary
  Nim string) use the `string` overload.
- Need to display a language or token class name to a human (logs, debugging,
  a CSS class) → `sourceLanguageToStr`/`sourceLanguageToAlpha` or
  `tokenClassToStr`, not `$lang`/`$kind` (the latter gives the enum value's
  programmatic name, like `langCpp`, not "C++").
- Need to quickly check or display the supported languages → `SourceLanguage`
  together with `sourceLanguageToStr` as a reference table (see the "Lookup
  tables" section).
