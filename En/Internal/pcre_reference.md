# `pcre.nim` — Module Reference

> **Module:** `pcre` — Nim binding to the PCRE (Perl Compatible Regular Expressions) C library  
> **PCRE version:** 8.36 (2014-09-26)  
> **Binding mode:** Dynamic library by default (`libpcre.so.3/.1`, `libpcre.dylib`, `pcre64.dll`/`pcre32.dll`). Use `-d:usePcreHeader` to bind against `<pcre.h>` at compile time instead.  
> **Audience:** This is a **low-level, direct C-API binding**. For everyday regex work in Nim, the higher-level `std/re` module (which uses this binding internally) is a better starting point. Use `pcre.nim` directly when you need JIT compilation, DFA matching, callouts, custom character tables, or per-match resource limits.

---

## Module Architecture at a Glance

```
Version constants       PCRE_MAJOR, PCRE_MINOR, PCRE_DATE
Option flags            CASELESS, MULTILINE, UTF8, PARTIAL_SOFT, …
Error codes             ERROR_NOMATCH, ERROR_BADUTF8, …
INFO / CONFIG selectors INFO_CAPTURECOUNT, CONFIG_JIT, …
STUDY / EXTRA flags     STUDY_JIT_COMPILE, EXTRA_MATCH_LIMIT, …
Opaque types            Pcre, JitStack
Data structures         ExtraData, CalloutBlock
Callback type           JitCallback
C function bindings     compile, exec, study, jit_exec, fullinfo, …
```

The typical workflow is:

```
compile() ──► [optional] study() ──► exec() / jit_exec() / dfa_exec()
                                           │
                              fullinfo() ◄─┤ (query pattern metadata)
                  get_substring() / get_named_substring() ◄─┘ (extract captures)
```

---

## Version Constants

```nim
PCRE_MAJOR*    = 8
PCRE_MINOR*    = 36
PCRE_PRERELEASE* = true
PCRE_DATE*     = "2014-09-26"
```

These identify the PCRE header version the binding was written against. They are compile-time constants — they do **not** change to match the version of the PCRE library installed on the target machine. To get the runtime library version, call `version()`.

```nim
echo PCRE_MAJOR, ".", PCRE_MINOR  # "8.36" — header version baked in at compile time
echo version()                     # e.g. "8.45 2021-06-15" — actual installed library
```

---

## Option Flag Constants

All flags are `int` bitmasks combined with `or`. PCRE tags each flag with the contexts where it is valid:

| Tag | Valid for |
|---|---|
| `C1` | `compile()` only — saved into the compiled pattern |
| `C2` | `compile()` behaviour only — not saved |
| `C3` | `compile()`, `exec()`, `dfa_exec()` |
| `C4` | `compile()`, `exec()`, `dfa_exec()`, `study()` |
| `C5` | `compile()`, `exec()`, `study()` |
| `E` | `exec()` at match time |
| `D` | `dfa_exec()` at match time |
| `J` | Compatible with JIT execution |

---

### Compile-time pattern behaviour flags

---

#### `CASELESS` — `0x00000001` · C1

Makes the entire pattern case-insensitive. ASCII letters match both cases. When combined with `UTF8` and `UCP`, Unicode case-folding rules apply.

```nim
# Matches "Hello", "HELLO", "hElLo", …
let re = compile("hello", CASELESS, addr err, addr errOff, nil)
```

---

#### `MULTILINE` — `0x00000002` · C1

Changes `^` and `$` to match at the start and end of **each line** (separated by `\n`) rather than just at the boundaries of the whole subject string. Equivalent to Python's `re.MULTILINE`.

```nim
# Matches "foo" at the start of any line, not just the very beginning
let re = compile("^foo", MULTILINE, addr err, addr errOff, nil)
```

---

#### `DOTALL` — `0x00000004` · C1

Makes `.` match **every** character including `\n`. By default, `.` stops at newline boundaries. This mode is sometimes called "single-line mode" in other regex flavours.

```nim
# Matches "a\nb" — the dot crosses the newline
let re = compile("a.b", DOTALL, addr err, addr errOff, nil)
```

---

#### `EXTENDED` — `0x00000008` · C1

Unquoted whitespace and `# …` comments in the pattern are ignored. Makes it possible to write readable, self-documenting patterns.

```nim
let pat = r"""
  \d{4}   # year
  -
  \d{2}   # month
  -
  \d{2}   # day
"""
let re = compile(pat, EXTENDED, addr err, addr errOff, nil)
```

---

#### `ANCHORED` — `0x00000010` · C4, E, D

Forces matching to start only at `startoffset` (or the very beginning of the subject when `startoffset` is 0). Equivalent to prepending `\A` to the pattern.

---

#### `DOLLAR_ENDONLY` — `0x00000020` · C2

Makes `$` match only at the absolute end of the subject string, ignoring a trailing `\n`. Without this flag, `$` also matches just before a final newline.

---

#### `EXTRA` — `0x00000040` · C1

Enables stricter syntax checking: unrecognised escape sequences like `\q` become compile-time errors rather than being passed through. A PCRE-specific strictness mode beyond Perl compatibility.

---

#### `UNGREEDY` — `0x00000200` · C1

Globally inverts quantifier greediness. `*`, `+`, `?` become non-greedy (lazy) by default; `*?`, `+?`, `??` become greedy. Saves appending `?` to every quantifier in patterns where non-greedy is the norm.

---

#### `NO_AUTO_CAPTURE` — `0x00001000` · C1

Disables the automatic numbering of plain `(…)` groups — they become non-capturing. Named groups `(?P<name>…)` / `(?<name>…)` still work and are still numbered. Useful to reduce ovector memory requirements.

---

#### `AUTO_CALLOUT` — `0x00004000` · C1

Inserts automatic callout points (number 255) before every item in the pattern. Used for step-by-step debugging of pattern execution via callout functions.

---

#### `FIRSTLINE` — `0x00040000` · C3

Requires the match to begin in the first line of the subject. The engine won't attempt matches starting after the first newline.

---

#### `DUPNAMES` — `0x00080000` · C1

Permits multiple capture groups to share the same name. Without this flag, duplicate names are a compile error. Useful in alternation patterns where logically equivalent groups appear in different branches.

```nim
# Both branches name their group "word"
let re = compile(r"(?P<word>\w+)|(?P<word>\d+)", DUPNAMES, addr err, addr errOff, nil)
```

---

#### `JAVASCRIPT_COMPAT` — `0x02000000` · C5

Adjusts a handful of behaviours to match JavaScript regex semantics — for example, `\u{HHHH}` syntax support and slightly different empty-match rules.

---

### UTF mode flags

```nim
UTF8*  = 0x00000800  # C4 — treat subject as UTF-8
UTF16* = 0x00000800  # C4 — synonym for Pcre16
UTF32* = 0x00000800  # C4 — synonym for Pcre32
```

All three constants share the same bit value. The active encoding is determined by which opaque type you use (`Pcre` for UTF-8, `Pcre16`, `Pcre32`). When enabled, character classes, `.`, `\w`, etc. operate on Unicode code points rather than raw bytes.

```nim
NO_UTF8_CHECK*  = 0x00002000  # C1 E D J
NO_UTF16_CHECK* = 0x00002000  # synonym
NO_UTF32_CHECK* = 0x00002000  # synonym
```

Skips the UTF validity check on the subject string. Use only when you can guarantee that the input is valid UTF — malformed input with this flag set may cause undefined behaviour.

---

#### `UCP` — `0x20000000` · C3

Makes `\w`, `\d`, `\s`, `\b`, and POSIX classes (`[:alpha:]` etc.) use Unicode property data rather than ASCII tables. Requires UTF mode to be active. Without `UCP`, `\w` matches only `[a-zA-Z0-9_]` even in UTF-8 mode.

---

### Match-time flags (E / D / J)

These are passed to `exec()`, `dfa_exec()`, or `jit_exec()` to adjust per-call matching behaviour.

---

#### `NOTBOL` — `0x00000080` · E D J

Signals that the start of `subject` is **not** a line beginning. `^` will not match at position 0. Essential for correct matching when searching within a substring of a larger text.

---

#### `NOTEOL` — `0x00000100` · E D J

Signals that the end of `subject` is **not** a line ending. `$` will not match at the final position. Mirrors `NOTBOL` for the end-of-line anchor.

---

#### `NOTEMPTY` — `0x00000400` · E D J

Rejects zero-length matches. Prevents infinite loops when iterating through all matches in a string using patterns that can match empty strings (e.g. `a*`).

---

#### `NOTEMPTY_ATSTART` — `0x10000000` · E D J

Like `NOTEMPTY` but only rejects an empty match at `startoffset`. Empty matches at later positions are still allowed. Useful in iterative matching loops.

---

#### `PARTIAL_SOFT` / `PARTIAL` — `0x00008000` · E D J

Enables **soft partial matching**. If the pattern consumes the entire subject without fully succeeding — but could succeed with more input — the engine returns `ERROR_PARTIAL` instead of `ERROR_NOMATCH`. A full match always takes precedence over a partial one.

```nim
# Is "abc" the start of a valid match for r"\d{4}-\d{2}"?
let rc = exec(re, nil, "20", 2, 0, PARTIAL_SOFT, addr ov[0], 30)
if rc == ERROR_PARTIAL:
  echo "Needs more input to decide"
```

---

#### `PARTIAL_HARD` — `0x08000000` · E D J

Like `PARTIAL_SOFT` but a partial match takes precedence over a full match. Use when you want to detect whether the pattern *could continue* past the end of the subject — useful for streaming parsers.

---

#### `NO_START_OPTIMIZE` / `NO_START_OPTIMISE` — `0x04000000` · C2 E D

Disables PCRE's start-position optimisations (first-character bitmap, anchoring inference). Normally never needed, but ensures callouts are triggered even at positions the engine would otherwise skip.

---

### Newline flags — C3 E D

Control what byte sequence counts as a newline. Affects `^`, `$`, `.`, and `\R`.

| Flag | Newline sequence |
|---|---|
| `NEWLINE_CR` | `\r` only |
| `NEWLINE_LF` | `\n` only (library default) |
| `NEWLINE_CRLF` | `\r\n` sequence only |
| `NEWLINE_ANY` | Any Unicode newline |
| `NEWLINE_ANYCRLF` | `\r`, `\n`, or `\r\n` |

---

### `\R` sequence flags — C3 E D

| Flag | What `\R` matches |
|---|---|
| `BSR_ANYCRLF` | `\r`, `\n`, or `\r\n` |
| `BSR_UNICODE` | All Unicode-defined newlines |

---

### Overlapping-bit pairs

Some pairs of flags share a bit. PCRE distinguishes them by context (compile vs. exec/dfa):

```nim
NEVER_UTF*       = 0x00010000  # C1: forbid UTF mode in compile()
DFA_SHORTEST*    = 0x00010000  # D:  DFA returns only shortest match

NO_AUTO_POSSESS* = 0x00020000  # C1: disable auto-possessification
DFA_RESTART*     = 0x00020000  # D:  resume DFA after partial match
```

---

## Error Codes

Negative return values from `exec()`, `dfa_exec()`, `jit_exec()`, `compile()`, and information functions. Non-negative values from `exec()` indicate success (number of captured substrings found).

| Constant | Value | When it occurs |
|---|---|---|
| `ERROR_NOMATCH` | -1 | Pattern did not match at all |
| `ERROR_NULL` | -2 | A required pointer argument was `nil` |
| `ERROR_BADOPTION` | -3 | Unrecognised option bit was set |
| `ERROR_BADMAGIC` | -4 | Compiled pattern has wrong magic number — version mismatch? |
| `ERROR_UNKNOWN_OPCODE` | -5 | Unknown opcode in compiled pattern (alias `ERROR_UNKNOWN_NODE`) |
| `ERROR_NOMEMORY` | -6 | Memory allocation failed inside PCRE |
| `ERROR_NOSUBSTRING` | -7 | Requested capture group number does not exist |
| `ERROR_MATCHLIMIT` | -8 | `ExtraData.match_limit` was exceeded |
| `ERROR_CALLOUT` | -9 | A callout function returned a non-zero (abort) value |
| `ERROR_BADUTF8` | -10 | Malformed UTF-8/16/32 in subject (alias `ERROR_BADUTF16`, `ERROR_BADUTF32`) |
| `ERROR_BADUTF8_OFFSET` | -11 | `startoffset` falls inside a multi-byte character |
| `ERROR_PARTIAL` | -12 | Partial match found when `PARTIAL_SOFT`/`PARTIAL_HARD` was set |
| `ERROR_BADPARTIAL` | -13 | Pattern cannot be used for partial matching |
| `ERROR_INTERNAL` | -14 | Internal PCRE error — should never occur |
| `ERROR_BADCOUNT` | -15 | `ovecsize` was not a multiple of 3 |
| `ERROR_RECURSIONLIMIT` | -21 | `ExtraData.match_limit_recursion` was exceeded |
| `ERROR_BADNEWLINE` | -23 | Conflicting newline options |
| `ERROR_BADOFFSET` | -24 | `startoffset` is negative or past the end of the subject |
| `ERROR_SHORTUTF8` | -25 | `startoffset` lands inside an incomplete UTF character |
| `ERROR_RECURSELOOP` | -26 | Recursive pattern call detected an infinite loop |
| `ERROR_JIT_STACKLIMIT` | -27 | The JIT stack overflowed |
| `ERROR_BADMODE` | -28 | 8/16/32-bit mode mismatch between pattern and exec call |
| `ERROR_BADLENGTH` | -32 | A negative `length` was passed to `exec()` |
| `ERROR_UNSET` | -33 | The requested named/numbered substring was not set in this match |

```nim
let rc = exec(re, nil, subject, subject.len.cint, 0, 0, addr ov[0], 30)
case rc
of ERROR_NOMATCH:    echo "no match"
of ERROR_PARTIAL:    echo "partial match — need more input"
of ERROR_MATCHLIMIT: echo "pattern too complex for this input"
else:
  if rc < 0: echo "PCRE error: ", rc
  else:      echo "found ", rc, " captured substrings"
```

---

## UTF Validity Error Codes

`UTF8_ERR0..UTF8_ERR22`, `UTF16_ERR0..UTF16_ERR4`, and `UTF32_ERR0..UTF32_ERR3` are secondary error codes populated inside `fullinfo()` when a UTF validity check fails. `ERR0` always means "no error". The higher numbers identify the specific kind of malformed byte sequence. Consult the PCRE documentation for the precise meaning of each code.

---

## `INFO_*` Constants — `fullinfo()` selector

Passed as the `what` argument to `fullinfo()` to query a specific property of a compiled pattern. The corresponding data type of `where` depends on the selector.

| Constant | Value | Type of `where` | What it returns |
|---|---|---|---|
| `INFO_OPTIONS` | 0 | `ptr cint` | Compile-time option bitmask |
| `INFO_SIZE` | 1 | `ptr csize_t` | Size of the compiled pattern in bytes |
| `INFO_CAPTURECOUNT` | 2 | `ptr cint` | Number of capture groups |
| `INFO_BACKREFMAX` | 3 | `ptr cint` | Highest back-reference number |
| `INFO_FIRSTCHAR` / `INFO_FIRSTBYTE` | 4 | `ptr cint` | First character value if the match is anchored to a literal |
| `INFO_FIRSTTABLE` | 5 | `ptr pointer` | Pointer to the 256-bit first-character bitmap |
| `INFO_LASTLITERAL` | 6 | `ptr cint` | Value of the literal that must be present anywhere in a match |
| `INFO_NAMEENTRYSIZE` | 7 | `ptr cint` | Size of each entry in the named-group table |
| `INFO_NAMECOUNT` | 8 | `ptr cint` | Number of named capture groups |
| `INFO_NAMETABLE` | 9 | `ptr pointer` | Pointer to the named-group table |
| `INFO_STUDYSIZE` | 10 | `ptr csize_t` | Size of study data |
| `INFO_MINLENGTH` | 15 | `ptr cint` | Minimum length of any match |
| `INFO_JIT` | 16 | `ptr cint` | 1 if JIT-compiled, 0 otherwise |
| `INFO_JITSIZE` | 17 | `ptr csize_t` | Size of the JIT compiled code |
| `INFO_MAXLOOKBEHIND` | 18 | `ptr cint` | Maximum lookbehind assertion length |
| `INFO_MATCHLIMIT` | 23 | `ptr clong` | Match limit embedded via `(*LIMIT_MATCH=N)` |
| `INFO_RECURSIONLIMIT` | 24 | `ptr clong` | Recursion limit embedded via `(*LIMIT_RECURSION=N)` |
| `INFO_MATCH_EMPTY` | 25 | `ptr cint` | 1 if the pattern can match an empty string |

```nim
var capCount: cint
discard fullinfo(re, nil, INFO_CAPTURECOUNT, addr capCount)
echo "This pattern has ", capCount, " capture groups"

var jitDone: cint
discard fullinfo(re, extra, INFO_JIT, addr jitDone)
if jitDone == 0:
  echo "Warning: JIT compilation was requested but did not succeed"
```

---

## `CONFIG_*` Constants — `config()` selector

Passed to `config()` to query the **library's** build-time configuration (not a specific pattern).

| Constant | Value | What it returns |
|---|---|---|
| `CONFIG_UTF8` | 0 | 1 if UTF-8 support is compiled in |
| `CONFIG_NEWLINE` | 1 | Default newline code (integer) |
| `CONFIG_LINK_SIZE` | 2 | Internal link size (2, 3, or 4) |
| `CONFIG_POSIX_MALLOC_THRESHOLD` | 3 | POSIX malloc threshold |
| `CONFIG_MATCH_LIMIT` | 4 | Default match limit |
| `CONFIG_STACKRECURSE` | 5 | 1 if the matching engine recurses on the C stack |
| `CONFIG_UNICODE_PROPERTIES` | 6 | 1 if Unicode property (`\p{}`) support is compiled in |
| `CONFIG_MATCH_LIMIT_RECURSION` | 7 | Default recursion match limit |
| `CONFIG_BSR` | 8 | Default `\R` behaviour (0 = any Unicode, 1 = CR/LF only) |
| `CONFIG_JIT` | 9 | 1 if JIT support is compiled in |
| `CONFIG_UTF16` | 10 | 1 if UTF-16 support is compiled in |
| `CONFIG_JITTARGET` | 11 | JIT target CPU string (pointer to `cstring`) |
| `CONFIG_UTF32` | 12 | 1 if UTF-32 support is compiled in |
| `CONFIG_PARENS_LIMIT` | 13 | Maximum parenthesis nesting depth |

```nim
var jitAvail: cint
discard config(CONFIG_JIT, addr jitAvail)
if jitAvail == 0:
  echo "This PCRE build has no JIT — study() with STUDY_JIT_COMPILE will be a no-op"
```

---

## `STUDY_*` Constants — `study()` option flags

Passed as the `options` argument to `study()`.

| Constant | Value | Effect |
|---|---|---|
| `STUDY_JIT_COMPILE` | 0x0001 | JIT-compile the pattern for full matching |
| `STUDY_JIT_PARTIAL_SOFT_COMPILE` | 0x0002 | JIT-compile for `PARTIAL_SOFT` matching |
| `STUDY_JIT_PARTIAL_HARD_COMPILE` | 0x0004 | JIT-compile for `PARTIAL_HARD` matching |
| `STUDY_EXTRA_NEEDED` | 0x0008 | Always allocate an `ExtraData` block even if study produces no data |

Combine with `or` to produce multiple JIT variants in one call:

```nim
let extra = study(re,
  STUDY_JIT_COMPILE or STUDY_JIT_PARTIAL_SOFT_COMPILE,
  addr studyErr)
```

---

## `EXTRA_*` Constants — `ExtraData.flags` bitmask

Set the corresponding bit in `ExtraData.flags` for every field you populate before passing `ExtraData` to `exec()`.

| Constant | Value | Corresponding field |
|---|---|---|
| `EXTRA_STUDY_DATA` | 0x0001 | `study_data` — set automatically by `study()` |
| `EXTRA_MATCH_LIMIT` | 0x0002 | `match_limit` |
| `EXTRA_CALLOUT_DATA` | 0x0004 | `callout_data` |
| `EXTRA_TABLES` | 0x0008 | `tables` |
| `EXTRA_MATCH_LIMIT_RECURSION` | 0x0010 | `match_limit_recursion` |
| `EXTRA_MARK` | 0x0020 | `mark` (output) |
| `EXTRA_EXECUTABLE_JIT` | 0x0040 | `executable_jit` — set automatically by `study()` with JIT |

```nim
# Protect against catastrophic backtracking on untrusted input:
var extra = ExtraData()
extra.flags = EXTRA_MATCH_LIMIT or EXTRA_MATCH_LIMIT_RECURSION
extra.match_limit = 100_000
extra.match_limit_recursion = 5_000
discard exec(re, addr extra, subject, subject.len.cint, 0, 0, addr ov[0], 30)
```

---

## Types

---

### `Pcre`, `Pcre16`, `Pcre32`

```nim
type
  Pcre*   = object
  Pcre16* = object
  Pcre32* = object
```

Opaque, zero-size object types used **only as pointer targets** (`ptr Pcre`). They represent a compiled regular expression pattern. You never inspect or construct them directly — `compile()` produces a `ptr Pcre` and all other functions consume it. `Pcre` is for 8-bit (byte or UTF-8) patterns; `Pcre16`/`Pcre32` are for the 16- and 32-bit PCRE variants (whose functions are not included in this binding file).

The compiler prevents you from accidentally mixing pointers to different widths because the types are distinct, even though they have the same internal layout.

---

### `JitStack`, `JitStack16`, `JitStack32`

```nim
type
  JitStack*   = object
  JitStack16* = object
  JitStack32* = object
```

Opaque handle for a **JIT execution stack**. The JIT engine needs a separate call stack distinct from the system C stack. Allocate with `jit_stack_alloc()`, associate with a pattern via `assign_jit_stack()`, and free with `jit_stack_free()`.

---

### `ExtraData`

```nim
type
  ExtraData* = object
    flags*:                  clong
    study_data*:             pointer
    match_limit*:            clong
    callout_data*:           pointer
    tables*:                 pointer
    match_limit_recursion*:  clong
    mark*:                   pointer
    executable_jit*:         pointer
```

**What it is:**  
An extensible parameter block passed to matching functions to supply optional per-match configuration and receive additional output. Every field is opt-in: you must set the corresponding `EXTRA_*` bit in `flags` for each field you use. Fields you do not flag are ignored by PCRE.

**Field descriptions:**

- **`flags`** — Bitmask of `EXTRA_*` constants declaring which fields are active. Always fill this before passing the struct.
- **`study_data`** — Opaque pointer populated by `study()`. Do not modify; it points to the analysed pattern data.
- **`match_limit`** — Maximum number of recursive backtracking steps. Prevents catastrophic backtracking on pathological inputs. Flag: `EXTRA_MATCH_LIMIT`.
- **`callout_data`** — Arbitrary user pointer forwarded verbatim into `CalloutBlock.callout_data` during callout invocations. Flag: `EXTRA_CALLOUT_DATA`.
- **`tables`** — Pointer to locale-specific character tables created by `maketables()`. Flag: `EXTRA_TABLES`.
- **`match_limit_recursion`** — Separate limit on recursive call depth (distinct from `match_limit`). Flag: `EXTRA_MATCH_LIMIT_RECURSION`.
- **`mark`** — *Output field.* After a successful match, PCRE writes the pointer to the last `(*MARK:name)` verb here. Flag: `EXTRA_MARK`.
- **`executable_jit`** — Pointer to JIT-compiled code, set by `study()`. Do not touch.

```nim
# Set a custom match limit to guard against ReDoS:
var extra = ExtraData()
extra.flags = EXTRA_MATCH_LIMIT
extra.match_limit = 50_000
let rc = exec(re, addr extra, subject, subject.len.cint, 0, 0, addr ov[0], 30)
if rc == ERROR_MATCHLIMIT:
  echo "Input rejected: pattern too complex"
```

---

### `CalloutBlock`

```nim
type
  CalloutBlock* = object
    version*         : cint
    callout_number*  : cint
    offset_vector*   : ptr cint
    subject*         : cstring
    subject_length*  : cint
    start_match*     : cint
    current_position*: cint
    capture_top*     : cint
    capture_last*    : cint
    callout_data*    : pointer
    pattern_position*: cint
    next_item_length*: cint
    mark*            : pointer
```

**What it is:**  
A live snapshot of the matcher's internal state, delivered to your callout function each time the engine reaches a `(?C)` point in the pattern (or at every item with `AUTO_CALLOUT`). Callouts let you observe, log, or abort matching as it progresses.

**Key fields:**

- **`version`** — Struct layout version (0, 1, or 2). Higher versions add more fields at the end.
- **`callout_number`** — The number from `(?Cn)`, or 255 for automatic callouts.
- **`subject`** / **`subject_length`** — The full subject string being matched.
- **`start_match`** — Byte offset of the start of the current match attempt.
- **`current_position`** — Current byte offset of the engine within the subject.
- **`capture_top`** — One past the highest capture group currently set.
- **`capture_last`** — Most recently closed capture group number.
- **`callout_data`** — The pointer from `ExtraData.callout_data`, passed through unchanged.
- **`pattern_position`** / **`next_item_length`** — Position in the pattern and length of the next pattern item — useful for tracing.
- **`mark`** — Current `(*MARK:name)` value (present from version 2).

A callout function returns `0` to continue matching or non-zero to abort with `ERROR_CALLOUT`. In C you set `pcre_callout` globally; from Nim you can do this via `{.emit:…}`.

---

### `JitCallback`

```nim
type JitCallback* = proc (a: pointer): ptr JitStack {.cdecl.}
```

**What it is:**  
A C-compatible (`cdecl`) callback invoked by the JIT engine just before each match attempt to obtain the `JitStack` to use. The `a` parameter is whatever `data` you passed to `assign_jit_stack()`.

The main use case is **per-thread JIT stacks**: store a `ptr JitStack` in a thread-local variable and return it from the callback. This avoids contention and makes JIT matching safe across threads.

```nim
var myThreadStack {.threadvar.}: ptr JitStack

proc getStack(data: pointer): ptr JitStack {.cdecl.} =
  if myThreadStack == nil:
    myThreadStack = jit_stack_alloc(32768, 1 shl 20)
  return myThreadStack

assign_jit_stack(extra, getStack, nil)
```

---

### `PPcre`, `PJitStack` *(deprecated)*

```nim
type
  PPcre*     {.deprecated.} = ptr Pcre
  PJitStack* {.deprecated.} = ptr JitStack
```

Legacy aliases from an earlier version of the binding. Use `ptr Pcre` and `ptr JitStack` directly.

---

## Functions

All functions are imported from the PCRE shared library with C calling convention. Nim names map to C names by the rule `pcre_<nimName>` (e.g. Nim's `compile` → C's `pcre_compile`).

---

### `compile`

```nim
proc compile*(pattern: cstring,
              options: cint,
              errptr: ptr cstring,
              erroffset: ptr cint,
              tableptr: pointer): ptr Pcre
```

**What it does:**  
Compiles a regex pattern string into an internal representation that can be reused for multiple matches. This is the primary entry point for using PCRE — compile once, match many times.

**Parameters:**
- `pattern` — Null-terminated regex string.
- `options` — Bitwise OR of compile-time flags (`CASELESS`, `MULTILINE`, `UTF8`, etc.).
- `errptr` — On failure: filled with a static, human-readable error message.
- `erroffset` — On failure: filled with the byte offset in `pattern` where the error was detected.
- `tableptr` — Custom locale tables from `maketables()`, or `nil` for the built-in ASCII tables.

**Returns:** A non-nil `ptr Pcre` on success. `nil` on failure — inspect `errptr` and `erroffset`.

**Ownership:** The compiled pattern is not automatically freed. `std/re` manages lifetime via Nim's GC. For direct use, decrement the reference count with `refcount(re, -1)` or call C's `pcre_free()` directly.

```nim
var err: cstring
var errOff: cint
let re = compile(r"(\d{4})-(\d{2})-(\d{2})", 0, addr err, addr errOff, nil)
if re == nil:
  quit("Pattern error at position " & $errOff & ": " & $err)
```

---

### `compile2`

```nim
proc compile2*(pattern: cstring,
               options: cint,
               errorcodeptr: ptr cint,
               errptr: ptr cstring,
               erroffset: ptr cint,
               tableptr: pointer): ptr Pcre
```

**What it does:**  
Identical to `compile()` but also populates `errorcodeptr` with a numeric error code on failure. Useful when you need to distinguish error types programmatically (e.g. UTF validity failure vs. syntax error) without parsing the human-readable string.

```nim
var errCode: cint
var err: cstring
var errOff: cint
let re = compile2(r"[\x{FFFFFF}]", UTF8,
                  addr errCode, addr err, addr errOff, nil)
if re == nil:
  echo "Error code ", errCode, ": ", err
```

---

### `exec`

```nim
proc exec*(code: ptr Pcre,
           extra: ptr ExtraData,
           subject: cstring,
           length: cint,
           startoffset: cint,
           options: cint,
           ovector: ptr cint,
           ovecsize: cint): cint
```

**What it does:**  
The core match function. Tries to match the compiled pattern against `subject` starting from `startoffset`. Uses a backtracking NFA engine.

**Parameters:**
- `code` — Compiled pattern from `compile()`.
- `extra` — Optional `ExtraData` for limits, callout data, tables, or `nil`.
- `subject` — The string to search.
- `length` — Length in bytes (not code points for UTF-8). Pass `-1` to have PCRE call `strlen`.
- `startoffset` — Byte offset to begin matching at. Use `0` for the start.
- `options` — Per-call flags: `NOTBOL`, `NOTEOL`, `NOTEMPTY`, `PARTIAL_SOFT`, etc.
- `ovector` — Output array of `cint` pairs: `[start0, end0, start1, end1, …]`. Pair 0 = overall match; pair 1 = first capture group; pair N = Nth capture. Size must be a multiple of 3.
- `ovecsize` — Number of elements in `ovector`. Minimum safe value: `(captureCount + 1) * 3`.

**Returns:**
- `> 0` — Match found. Value = number of capture pairs filled. `0` means the ovector was too small to hold all captures.
- `ERROR_NOMATCH (-1)` — No match.
- Other negative — Error code.

```nim
var ov: array[30, cint]   # space for 9 capture groups + whole match
let subj = "Date: 2024-03-15"
let rc = exec(re, nil, subj, subj.len.cint, 0, 0, addr ov[0], 30)
if rc > 0:
  echo "full match: ", subj[ov[0]..<ov[1]]
  echo "year:  ", subj[ov[2]..<ov[3]]
  echo "month: ", subj[ov[4]..<ov[5]]
  echo "day:   ", subj[ov[6]..<ov[7]]
```

**Iterating all matches:**

```nim
var start = 0
while start <= subj.len:
  let rc = exec(re, nil, subj, subj.len.cint, start.cint, 0, addr ov[0], 30)
  if rc < 0: break
  echo subj[ov[0]..<ov[1]]
  # Advance past the match; if empty match, move forward by 1 to avoid infinite loop
  start = if ov[1] > ov[0]: ov[1] else: ov[1] + 1
```

---

### `jit_exec`

```nim
proc jit_exec*(code: ptr Pcre,
               extra: ptr ExtraData,
               subject: cstring,
               length: cint,
               startoffset: cint,
               options: cint,
               ovector: ptr cint,
               ovecsize: cint,
               jstack: ptr JitStack): cint
```

**What it does:**  
Identical in semantics to `exec()` but dispatches to JIT-compiled machine code for the match, making it significantly faster (commonly 5–20×) on patterns that were studied with `STUDY_JIT_COMPILE`. Falls back silently to interpreted matching if JIT is unavailable or compilation failed.

The extra `jstack` parameter specifies the JIT execution stack directly for this call. Pass `nil` to use the stack registered via `assign_jit_stack()`.

```nim
# Full JIT pipeline:
var studyErr: cstring
let extra = study(re, STUDY_JIT_COMPILE, addr studyErr)
let jstack = jit_stack_alloc(32768, 1 shl 20)
assign_jit_stack(extra, nil, jstack)

let rc = jit_exec(re, extra, subj, subj.len.cint,
                  0, 0, addr ov[0], 30, jstack)

# Cleanup:
jit_stack_free(jstack)
free_study(extra)
```

---

### `dfa_exec`

```nim
proc dfa_exec*(code: ptr Pcre,
               extra: ptr ExtraData,
               subject: cstring,
               length: cint,
               startoffset: cint,
               options: cint,
               ovector: ptr cint,
               ovecsize: cint,
               workspace: ptr cint,
               wscount: cint): cint
```

**What it does:**  
An alternative matcher using a **DFA (Deterministic Finite Automaton)**. Key differences from `exec()`:

- **No backtracking.** Immune to catastrophic backtracking ("ReDoS"). Runtime is always O(n) in subject length.
- **Finds all matches** at the current position simultaneously (the ovector contains all of them, longest first), not just the first.
- **No capture groups.** Only `ovector[0..1]` (the overall match boundary) is meaningful; all other pairs are unused.
- Generally slower than JIT `exec()` for typical patterns.

**Extra parameters:**
- `workspace` — Scratch array of `cint` for the DFA algorithm's internal state. At least 20 elements; larger for complex patterns.
- `wscount` — Length of `workspace`.

```nim
var ov: array[30, cint]
var ws: array[200, cint]
let rc = dfa_exec(re, nil, subj, subj.len.cint,
                  0, 0, addr ov[0], 30, addr ws[0], 200)
if rc > 0:
  # ov[0..1] = longest match, ov[2..3] = second-longest, …
  for i in 0 ..< rc:
    echo "match ", i, ": ", subj[ov[i*2]..<ov[i*2+1]]
```

---

### `study`

```nim
proc study*(code: ptr Pcre,
            options: cint,
            errptr: ptr cstring): ptr ExtraData
```

**What it does:**  
Analyses a compiled pattern and produces an `ExtraData` block containing optimisation data. The most important use is requesting **JIT compilation**, which translates the pattern into native machine code and can make matching 5–100× faster.

**Parameters:**
- `code` — A compiled pattern from `compile()`.
- `options` — `STUDY_*` flags. Use `STUDY_JIT_COMPILE` to enable JIT.
- `errptr` — Set to an error message if the study itself fails.

**Returns:** `ptr ExtraData` on success. Returns `nil` without error if studying produced no useful data and `STUDY_EXTRA_NEEDED` was not set — this is normal on simple patterns. Returns `nil` with `errptr` set on actual failure.

**Lifecycle:** Must be freed with `free_study()` when the pattern is no longer needed.

```nim
var studyErr: cstring
let extra = study(re, STUDY_JIT_COMPILE, addr studyErr)
if studyErr != nil:
  echo "study failed: ", studyErr
elif extra != nil:
  var jitDone: cint
  discard fullinfo(re, extra, INFO_JIT, addr jitDone)
  echo if jitDone == 1: "JIT active" else: "JIT not available on this platform"
```

---

### `free_study`

```nim
proc free_study*(extra: ptr ExtraData)
```

**What it does:**  
Frees the `ExtraData` block returned by `study()`, including any embedded JIT-compiled code. Always pair with `study()`.

```nim
free_study(extra)
```

---

### `fullinfo`

```nim
proc fullinfo*(code: ptr Pcre,
               extra: ptr ExtraData,
               what: cint,
               where: pointer): cint
```

**What it does:**  
Queries a compiled pattern for metadata. Pass an `INFO_*` constant as `what` and a pointer to an appropriately typed variable as `where`. Returns `0` on success or a negative error code.

```nim
# How many capture groups does this pattern have?
var nCaptures: cint
discard fullinfo(re, nil, INFO_CAPTURECOUNT, addr nCaptures)
var ov = newSeq[cint]((nCaptures + 1) * 3)

# Can this pattern match an empty string? (important for loop safety)
var canEmpty: cint
discard fullinfo(re, nil, INFO_MATCH_EMPTY, addr canEmpty)
if canEmpty == 1:
  echo "Warning: this pattern can match empty strings"
```

---

### `config`

```nim
proc config*(what: cint, where: pointer): cint
```

**What it does:**  
Queries the PCRE **library's** build-time configuration — not a specific pattern. Use `CONFIG_*` constants as `what`. Useful for feature detection before attempting operations that depend on specific PCRE compile options.

```nim
var hasUcp: cint
discard config(CONFIG_UNICODE_PROPERTIES, addr hasUcp)
if hasUcp == 0:
  echo "\\p{} Unicode properties not available — recompile PCRE with --enable-unicode-properties"
```

---

### `get_substring`

```nim
proc get_substring*(subject: cstring,
                    ovector: ptr cint,
                    stringcount: cint,
                    stringnumber: cint,
                    stringptr: cstringArray): cint
```

**What it does:**  
Allocates a new null-terminated C string containing the text of capture group `stringnumber` (0 = overall match, 1 = first group, etc.) and writes its address into `stringptr`. `stringcount` is the value returned by `exec()`. Returns the length of the string or a negative error code.

Must be freed with `free_substring()`.

```nim
var sub: cstringArray
let len = get_substring(subj, addr ov[0], rc, 1, addr sub)
if len >= 0:
  echo "group 1: ", sub
  free_substring(sub)
```

> In practice, extracting substrings directly from `subject` using `ov` indices is usually cleaner in Nim: `subject[ov[2]..<ov[3]]`.

---

### `get_named_substring`

```nim
proc get_named_substring*(code: ptr Pcre,
                          subject: cstring,
                          ovector: ptr cint,
                          stringcount: cint,
                          stringname: cstring,
                          stringptr: cstringArray): cint
```

**What it does:**  
Like `get_substring()` but retrieves a capture group by **name** (from `(?P<name>…)` or `(?<name>…)` syntax). Returns the substring length or a negative error.

```nim
var yearStr: cstringArray
discard get_named_substring(re, subj, addr ov[0], rc, "year", addr yearStr)
echo "year: ", yearStr
free_substring(yearStr)
```

---

### `copy_substring`

```nim
proc copy_substring*(subject: cstring,
                     ovector: ptr cint,
                     stringcount: cint,
                     stringnumber: cint,
                     buffer: cstring,
                     buffersize: cint): cint
```

**What it does:**  
Like `get_substring()` but copies into a **caller-owned buffer** instead of allocating. No `free_substring()` call needed. Returns `ERROR_NOMEMORY` if the buffer is too small.

```nim
var buf: array[256, char]
let rc2 = copy_substring(subj, addr ov[0], rc, 0,
                         cast[cstring](addr buf[0]), 256)
if rc2 >= 0:
  echo "match: ", cast[cstring](addr buf[0])
```

---

### `copy_named_substring`

```nim
proc copy_named_substring*(code: ptr Pcre,
                           subject: cstring,
                           ovector: ptr cint,
                           stringcount: cint,
                           stringname: cstring,
                           buffer: cstring,
                           buffersize: cint): cint
```

**What it does:**  
Combines `copy_substring` and `get_named_substring`: retrieves a named capture group into a caller-provided buffer with no allocation.

---

### `get_substring_list`

```nim
proc get_substring_list*(subject: cstring,
                         ovector: ptr cint,
                         stringcount: cint,
                         listptr: ptr cstringArray): cint
```

**What it does:**  
Allocates a null-terminated array of C strings containing **all** captured substrings (index 0 = whole match). Convenient for inspecting all captures in one call. Must be freed with `free_substring_list()`.

```nim
var list: cstringArray
discard get_substring_list(subj, addr ov[0], rc, addr list)
var i = 0
while list[i] != nil:
  echo "group ", i, ": ", list[i]
  inc i
free_substring_list(list)
```

---

### `free_substring`

```nim
proc free_substring*(stringptr: cstring)
```

Frees a string allocated by `get_substring()` or `get_named_substring()`.

---

### `free_substring_list`

```nim
proc free_substring_list*(stringptr: cstringArray)
```

Frees the list allocated by `get_substring_list()`.

---

### `get_stringnumber`

```nim
proc get_stringnumber*(code: ptr Pcre, name: cstring): cint
```

**What it does:**  
Translates a capture group **name** to its **number**. Returns the number (≥ 1) on success, or a negative error code if the name is not found in the pattern.

```nim
let yearIdx = get_stringnumber(re, "year")
if yearIdx > 0:
  echo "year: ", subj[ov[yearIdx*2]..<ov[yearIdx*2+1]]
```

---

### `get_stringtable_entries`

```nim
proc get_stringtable_entries*(code: ptr Pcre,
                              name: cstring,
                              first: cstringArray,
                              last: cstringArray): cint
```

**What it does:**  
When `DUPNAMES` is active, multiple groups can share a name. This function returns pointers to the first and last entries in the internal name table for that name, enabling iteration over all groups with that name. Returns the entry size on success.

---

### `maketables`

```nim
proc maketables*(): pointer
```

**What it does:**  
Generates a complete set of character classification tables using the C library's current locale (as set by `setlocale`). The resulting pointer can be passed to `compile()` as `tableptr` to make case-folding and character classes locale-aware. Returns `nil` on allocation failure.

Ownership is caller's responsibility; free with C's `free()`.

```nim
let tables = maketables()
let re = compile(r"\w+", CASELESS, addr err, addr errOff, tables)
```

---

### `refcount`

```nim
proc refcount*(code: ptr Pcre, adjust: cint): cint
```

**What it does:**  
Adjusts the compiled pattern's reference count by `adjust` and returns the new count. Pass `+1` to increment, `-1` to decrement (freeing the pattern when count reaches 0), `0` to query without changing. Used by higher-level wrappers that implement shared ownership of compiled patterns.

---

### `version`

```nim
proc version*(): cstring
```

**What it does:**  
Returns a static string from the PCRE library describing its runtime version and date, e.g. `"8.45 2021-06-15"`. Use this to verify that the installed shared library matches what your code expects.

```nim
echo "Bound against header: ", PCRE_MAJOR, ".", PCRE_MINOR
echo "Runtime library:      ", version()
```

---

### `pattern_to_host_byte_order`

```nim
proc pattern_to_host_byte_order*(code: ptr Pcre,
                                 extra: ptr ExtraData,
                                 tables: pointer): cint
```

**What it does:**  
Byte-swaps a compiled pattern that was serialised on a machine with different endianness and loaded on the current host. Returns `0` if swapping was needed and succeeded, `ERROR_BADENDIANNESS` if the pattern is already in host byte order (i.e. no swapping needed), or another error on failure. Relevant only in cross-platform serialisation/caching workflows.

---

### `jit_stack_alloc`

```nim
proc jit_stack_alloc*(startsize: cint, maxsize: cint): ptr JitStack
```

**What it does:**  
Allocates a JIT execution stack. The stack starts at `startsize` bytes and can grow on demand up to `maxsize` bytes. Returns `nil` on allocation failure.

**Sizing guidance:** `32 * 1024` start / `1 shl 20` (1 MB) max is a good default. Patterns with deep recursion or very long input may need larger stacks.

```nim
let stack = jit_stack_alloc(32 * 1024, 1 shl 20)
if stack == nil: quit("Failed to allocate JIT stack")
```

---

### `jit_stack_free`

```nim
proc jit_stack_free*(stack: ptr JitStack)
```

Frees a JIT stack allocated by `jit_stack_alloc()`. Always pair with `jit_stack_alloc()`.

---

### `assign_jit_stack`

```nim
proc assign_jit_stack*(extra: ptr ExtraData,
                       callback: JitCallback,
                       data: pointer)
```

**What it does:**  
Associates a JIT stack (or a stack-providing callback) with the `ExtraData` for a JIT-compiled pattern. Two modes:

- **Static stack:** Pass `nil` as `callback` and the `ptr JitStack` as `data`. PCRE will always use this one stack. Simple but not thread-safe if the same `extra` is used from multiple threads.
- **Dynamic callback:** Pass a `JitCallback` proc as `callback` and an arbitrary context pointer as `data`. PCRE calls the callback before each match to obtain the stack. Ideal for per-thread stacks.

```nim
# Static — fine for single-threaded use:
assign_jit_stack(extra, nil, cast[pointer](stack))

# Dynamic — correct for multi-threaded use:
assign_jit_stack(extra, getThreadStack, nil)
```

---

### `jit_free_unused_memory`

```nim
proc jit_free_unused_memory*()
```

**What it does:**  
Releases memory held by the JIT machine code allocator that is no longer referenced by any compiled pattern. Useful in long-running programs that compile many patterns over time, to prevent the JIT executable memory pool from growing unboundedly.

---

## Complete Usage Examples

### Minimum viable match

```nim
import pcre

var err: cstring
var errOff: cint
let re = compile(r"(\w+)\s+(\w+)", 0, addr err, addr errOff, nil)
if re == nil:
  quit("compile error at " & $errOff & ": " & $err)

var ov: array[9, cint]  # (2 groups + whole match) * 3
let subj = "hello world"
let rc = exec(re, nil, subj, subj.len.cint, 0, 0, addr ov[0], 9)
if rc > 0:
  echo "match:   ", subj[ov[0]..<ov[1]]
  echo "word 1:  ", subj[ov[2]..<ov[3]]
  echo "word 2:  ", subj[ov[4]..<ov[5]]
```

### JIT-accelerated matching in a loop

```nim
var studyErr: cstring
let extra = study(re, STUDY_JIT_COMPILE, addr studyErr)
let jstack = jit_stack_alloc(32768, 1 shl 20)
assign_jit_stack(extra, nil, cast[pointer](jstack))

var start = 0
while start <= subj.len:
  let rc = jit_exec(re, extra, subj, subj.len.cint,
                    start.cint, 0, addr ov[0], 9, jstack)
  if rc < 0: break
  echo subj[ov[0]..<ov[1]]
  start = if ov[1] > ov[0]: ov[1] else: ov[1] + 1

jit_stack_free(jstack)
free_study(extra)
```

### ReDoS protection with match limits

```nim
var extra = ExtraData()
extra.flags = EXTRA_MATCH_LIMIT or EXTRA_MATCH_LIMIT_RECURSION
extra.match_limit            = 100_000
extra.match_limit_recursion  =   5_000

let rc = exec(re, addr extra, untrustedInput, untrustedInput.len.cint,
              0, 0, addr ov[0], 9)
case rc
of ERROR_MATCHLIMIT:    echo "Input rejected: match limit reached"
of ERROR_NOMATCH:       echo "No match"
else: discard
```

### Named capture groups

```nim
let datePat = compile(r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})",
                      0, addr err, addr errOff, nil)
var ov: array[12, cint]
let rc = exec(datePat, nil, "2024-03-15", 10, 0, 0, addr ov[0], 12)
if rc > 0:
  let yearN = get_stringnumber(datePat, "year")
  echo "year: ", "2024-03-15"[ov[yearN*2]..<ov[yearN*2+1]]
```
