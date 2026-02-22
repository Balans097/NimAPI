# std/unittest — Module Reference

> Nim's built-in framework for writing and running unit tests.
> Tests are declared with simple templates, output is printed after each test,
> and the process exits with code `1` if any test failed.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Running Tests from the Command Line](#running-tests-from-the-command-line)
3. [Types](#types)
4. [Global Variables](#global-variables)
5. [Test Structure](#test-structure)
   - [`suite`](#suite)
   - [`test`](#test)
   - [`setup` and `teardown`](#setup-and-teardown)
6. [Assertions and Checks](#assertions-and-checks)
   - [`check`](#check)
   - [`require`](#require)
   - [`expect`](#expect)
   - [`fail`](#fail)
   - [`skip`](#skip)
   - [`checkpoint`](#checkpoint)
7. [Output Formatters](#output-formatters)
   - [OutputFormatter (base)](#outputformatter-base)
   - [ConsoleOutputFormatter](#consoleoutputformatter)
   - [JUnitOutputFormatter](#junitoutputformatter)
8. [Compile-time and Environment Configuration](#compile-time-and-environment-configuration)
9. [Limitations](#limitations)
10. [Quick-Reference Summary](#quick-reference-summary)

---

## Core Concepts

`std/unittest` is built around three ideas:

**Tests are named blocks of code.** A `test "name": body` block runs its body, catches any unhandled exceptions, and records whether the test passed, failed, or was skipped.

**Suites group related tests.** A `suite "name": body` block wraps multiple tests, provides shared `setup` and `teardown` hooks that run around each individual test, and adds the suite name to the output.

**Assertions are non-fatal by default.** A failing `check` prints a diagnostic message and marks the test as failed, but execution of the current test continues. This lets you see all failures in a single run rather than stopping at the first. Use `require` if you need a hard stop.

---

## Running Tests from the Command Line

After compiling (or using `nim c -r`), you can pass test names or patterns as command-line arguments to run only a subset of tests.

### Run all tests (default)

```sh
nim c -r mytest.nim
```

### Run specific tests by exact name

```sh
nim c -r mytest.nim "my test name" "another test"
```

### Run an entire suite

Append `::` to the suite name:

```sh
nim c -r mytest.nim "my suite name::"
```

### Filter with glob patterns

A single `*` acts as a wildcard that matches any sequence of characters. The `::` separator divides a suite pattern from a test pattern. Tests matching **any** of the given arguments are run.

```sh
nim c -r mytest.nim "fast_suite::mytest1" "fast_suite::mytest2"
nim c -r mytest.nim "fast_suite::mytest*"
nim c -r mytest.nim "auth*::" "crypto::hashing*"

# Run suites starting with 'bug #' and standalone tests starting with '#'
nim c -r mytest.nim 'bug #*::' '::#*'
```

If no arguments are given, all tests are run.

---

## Types

### `TestStatus`

```nim
type TestStatus* = enum
  OK,
  FAILED,
  SKIPPED
```

The outcome of a single test case after it has finished running. `OK` means no assertion failed and no unhandled exception was raised. `FAILED` means at least one `check` or `require` failed, or an unhandled exception escaped from the test body. `SKIPPED` means `skip()` was called inside the test.

---

### `OutputLevel`

```nim
type OutputLevel* = enum
  PRINT_ALL,       ## Print every test result
  PRINT_FAILURES,  ## Print only failed tests
  PRINT_NONE       ## Suppress all test output
```

Controls the verbosity of the console formatter. The default is `PRINT_ALL`. Can be set at compile time with `-d:nimUnittestOutputLevel:PRINT_FAILURES` (for example) or at runtime via the deprecated `NIMTEST_OUTPUT_LVL` environment variable.

---

### `TestResult`

```nim
type TestResult* = object
  suiteName*: string   ## Name of the enclosing suite, or "" if none
  testName*:  string   ## Name of the test case
  status*:    TestStatus
```

A value-type snapshot of a test's outcome, passed to formatter methods when a test ends. Formatters receive this object and use it to write output in whatever format they support.

---

### `OutputFormatter`

```nim
type OutputFormatter* = ref object of RootObj
```

The abstract base class for all output formatters. Custom formatters inherit from this type and override its methods. The module ships two concrete implementations: `ConsoleOutputFormatter` and `JUnitOutputFormatter`.

---

### `ConsoleOutputFormatter`

```nim
type ConsoleOutputFormatter* = ref object of OutputFormatter
```

The default formatter. Writes human-readable test results to the terminal, optionally with colour. Color support is auto-detected from `isatty(stdout)` and can be overridden at compile time with `-d:nimUnittestColor:on|off|auto`.

---

### `JUnitOutputFormatter`

```nim
type JUnitOutputFormatter* = ref object of OutputFormatter
```

A formatter that writes results in JUnit-compatible XML format to any `Stream`. Useful for integration with CI tools (Jenkins, GitLab CI, GitHub Actions, etc.) that understand JUnit XML reports.

---

## Global Variables

### `abortOnError`

```nim
var abortOnError* {.threadvar.}: bool
```

When set to `true`, any test failure immediately calls `quit(1)` instead of continuing. Teardown blocks are **not** executed when this happens. The default is `false`, meaning failing tests are recorded and execution continues. Can be enabled at compile time with `-d:nimUnittestAbortOnError:on`.

```nim
abortOnError = true   # set at runtime
```

---

## Test Structure

### `suite`

```nim
template suite*(name, body) {.dirty.}
```

Declares a named group of related tests. The `body` can contain `setup`, `teardown`, and `test` blocks, as well as any ordinary Nim code (which runs once when the suite block is entered, before any tests).

Suites add a header line like `[Suite] my suite name` to the output and indent test results under it. Nesting suites is possible, but a failure in a nested test does not propagate up to mark the parent as failed.

```nim
suite "arithmetic":
  echo "this runs once, before any tests in the suite"

  setup:
    let x = 10   # runs before each test

  teardown:
    echo "cleanup"   # runs after each test

  test "addition":
    check x + 5 == 15

  test "subtraction":
    check x - 3 == 7

  echo "this also runs once, after all tests"
```

Output:

```
[Suite] arithmetic
  [OK] addition
  [OK] subtraction
```

---

### `test`

```nim
template test*(name, body) {.dirty.}
```

Declares a single test case with a human-readable name. The body is executed when the test runs. Any unhandled exception caught during the body automatically fails the test and prints the exception message and stack trace.

Tests can be defined at the top level (outside any suite) or inside a `suite` block.

```nim
test "basic truth":
  check 1 + 1 == 2

test "string operations":
  let s = "hello"
  check s.len == 5
  check s.toUpperAscii() == "HELLO"
```

A test succeeds if it reaches the end of its body without any `check` failing and without any unhandled exception.

---

### `setup` and `teardown`

`setup` and `teardown` are sub-templates that can appear **inside a `suite` block**. They define code that runs before and after **each individual test** within that suite, respectively.

```nim
suite "database tests":
  var db: DatabaseConnection

  setup:
    db = openTestDatabase()     # runs before every test
    db.beginTransaction()

  teardown:
    db.rollback()               # runs after every test
    db.close()

  test "insert record":
    db.insert(Record(id: 1))
    check db.count() == 1

  test "delete record":
    db.insert(Record(id: 2))
    db.delete(2)
    check db.count() == 0
```

`setup` code is executed first in each test, before the test body. `teardown` is always executed after the test body, even if the test fails (it is registered with `defer`). A suite can have at most one `setup` and one `teardown` block. Inner suites inherit the parent's setup/teardown but can override them locally.

---

## Assertions and Checks

### `check`

```nim
macro check*(conditions: untyped): untyped
```

The primary assertion macro. Evaluates a boolean condition (or a block of conditions) and, if it is false, prints a diagnostic message showing the source of the failure and the values of sub-expressions, then marks the test as failed. Execution of the current test **continues** after a failing `check`.

**Single condition:**

```nim
check 2 + 2 == 4
check myList.len > 0
check "hello".startsWith("he")
```

**Block of conditions (each is checked independently):**

```nim
check:
  result != nil
  result.name == "Alice"
  result.age >= 18
```

**What makes `check` special:** when a comparison like `a == b` fails, `check` prints not just "check failed" but the actual runtime values of both `a` and `b`. For example, if `x = 5` and you write `check x == 10`, the output will include something like `x was 5`. This is achieved by rewriting the expression at compile time using macros.

**Important limitation:** because `check` assigns sub-expressions to temporary variables internally, mixed-type comparisons like `check 4.0 == 2 + 2` may fail to compile. Use `doAssert` for such cases, or ensure both sides have the same type.

---

### `require`

```nim
template require*(conditions: untyped)
```

Identical to `check` in every way, except that a failure immediately terminates the entire program (calls `quit(1)`) regardless of the current `abortOnError` setting. Teardown blocks are **not** run after `require` fails.

Use `require` for preconditions that make all subsequent test code meaningless if they are not met — for example, verifying that a file exists before trying to read it.

```nim
test "parse config file":
  require fileExists("config.json")   # stop the whole run if this fails
  let cfg = parseFile("config.json")
  check cfg["key"].getStr == "value"
```

---

### `expect`

```nim
macro expect*(exceptions: varargs[typed], body: untyped): untyped
```

Verifies that executing `body` raises one of the specified exception types. The test passes if the body raises an exception that matches any of the listed types. The test fails if the body completes without raising, or raises a different exception type.

```nim
test "out of bounds raises":
  let v = @[1, 2, 3]
  expect IndexDefect:
    discard v[10]

test "multiple acceptable exceptions":
  expect IOError, OSError:
    discard readFile("/nonexistent/path")

test "parsing invalid input":
  expect ValueError:
    discard parseInt("not a number")
```

When `Exception` is included in the exception list, `expect` uses a simpler form that accepts any exception without checking its specific type — useful for broad catch-all tests.

---

### `fail`

```nim
template fail*
```

Manually marks the current test as failed. Prints any accumulated checkpoints, then either quits (if `abortOnError` is true) or continues. After `fail`, the checkpoints list is cleared.

`fail` is rarely needed directly in application test code — it is the internal primitive that `check` and `require` call when an assertion does not hold. It can be useful in custom assertion helpers or for signalling failure based on complex logic that cannot be expressed as a simple boolean:

```nim
test "custom failure":
  checkpoint("About to check something complex")
  if not complexCondition():
    checkpoint("complexCondition returned false")
    fail
```

---

### `skip`

```nim
template skip*
```

Marks the current test as `SKIPPED` instead of `OK` or `FAILED`. The test body **continues executing** after `skip` is called (unlike a `return`). Use it when a test cannot meaningfully run in the current environment — for example, when a required external resource is unavailable.

```nim
test "GPU rendering":
  if not isGPUAvailable():
    skip()
  # code below still runs, but the test is recorded as SKIPPED
  let result = renderFrame()
  check result.width == 1920
```

Because execution continues, it is common to place a `return`-equivalent after `skip` by wrapping the remaining body in an `if` or by exiting the test block.

---

### `checkpoint`

```nim
proc checkpoint*(msg: string)
```

Records a diagnostic message that will be printed if the current test subsequently fails. Checkpoints are accumulated in a list; when `fail` is triggered (by a failing `check`, unhandled exception, or direct call), all checkpoints recorded since the last clear are printed before the failure message.

Checkpoints give context to a failure — they work like breadcrumbs that let you trace what happened leading up to the point where things went wrong.

```nim
test "complex multi-step process":
  checkpoint("Step 1: initializing")
  let ctx = initialize()

  checkpoint("Step 2: processing with ctx=" & $ctx.id)
  let result = process(ctx)

  checkpoint("Step 3: validating result")
  check result.valid        # if this fails, all three checkpoints are printed
  check result.value == 42
```

Checkpoints are automatically cleared after each test ends (whether it passes or fails).

---

## Output Formatters

Formatters are the output back-end of the test runner. Multiple formatters can be active simultaneously — for instance, printing to the console and writing JUnit XML at the same time.

### `addOutputFormatter`

```nim
proc addOutputFormatter*(formatter: OutputFormatter)
```

Adds a formatter to the active formatter list. All subsequent test lifecycle events (suite start, test start, failure, test end, suite end) will be dispatched to this formatter.

```nim
let f = newJUnitOutputFormatter(openFileStream("results.xml", fmWrite))
addOutputFormatter(f)
# run tests...
f.close()
```

---

### `delOutputFormatter`

```nim
proc delOutputFormatter*(formatter: OutputFormatter)
```

Removes a specific formatter from the active list. Useful when a formatter is only needed for a portion of the test run.

---

### `resetOutputFormatters`

```nim
proc resetOutputFormatters*
```

Clears the entire formatter list. After this call, no output is produced until a new formatter is added. This is useful in tests of the test framework itself, or when you want to switch formatters entirely mid-run.

---

### `newConsoleOutputFormatter`

```nim
proc newConsoleOutputFormatter*(
  outputLevel: OutputLevel = PRINT_ALL,
  colorOutput = true
): ConsoleOutputFormatter
```

Creates a new console formatter with the specified verbosity and color settings. Normally you do not need to call this directly — the module creates a default console formatter automatically on first use. Use this when you want a differently-configured console formatter.

```nim
let fmt = newConsoleOutputFormatter(PRINT_FAILURES, colorOutput = false)
addOutputFormatter(fmt)
```

---

### `defaultConsoleFormatter`

```nim
proc defaultConsoleFormatter*(): ConsoleOutputFormatter
```

Creates the console formatter that the module would use by default — respecting compile-time defines for output level and color, as well as the deprecated environment variables. The module calls this internally on first use; you would call it explicitly only if you needed to reset to the default formatter after replacing it.

---

### OutputFormatter — virtual methods

These methods form the lifecycle protocol that all formatters must implement. They are called by the test runner at the appropriate moments. Override them in a custom formatter:

```nim
method suiteStarted*(formatter: OutputFormatter, suiteName: string) {.base.}
method testStarted*(formatter: OutputFormatter, testName: string) {.base.}
method failureOccurred*(formatter: OutputFormatter, checkpoints: seq[string], stackTrace: string) {.base.}
method testEnded*(formatter: OutputFormatter, testResult: TestResult) {.base.}
method suiteEnded*(formatter: OutputFormatter) {.base.}
```

| Method | When called |
|---|---|
| `suiteStarted` | When a `suite` block begins |
| `testStarted` | When a `test` block begins |
| `failureOccurred` | When `fail` is called (after a `check` fails or an exception is caught) |
| `testEnded` | When a `test` block finishes (always, even on failure) |
| `suiteEnded` | When a `suite` block finishes |

`failureOccurred` receives the accumulated checkpoints and, if the failure was caused by an exception, the exception's stack trace. The base implementations do nothing — each concrete formatter overrides the methods it cares about.

**Example — custom formatter:**

```nim
type MyFormatter = ref object of OutputFormatter
  passed, failed: int

method testEnded*(f: MyFormatter, r: TestResult) =
  if r.status == OK: inc f.passed
  else: inc f.failed

let f = MyFormatter()
addOutputFormatter(f)
# ... run tests ...
echo f.passed, " passed, ", f.failed, " failed"
```

---

### `newJUnitOutputFormatter`

```nim
proc newJUnitOutputFormatter*(stream: Stream): JUnitOutputFormatter
```

Creates a formatter that writes JUnit-compatible XML to `stream`. The stream is **not** closed automatically; you must call `formatter.close()` after all tests are done to flush the closing XML tags and close the stream.

```nim
import std/[unittest, streams]

let xmlOut = newFileStream("results.xml", fmWrite)
let jfmt = newJUnitOutputFormatter(xmlOut)
addOutputFormatter(jfmt)

suite "my suite":
  test "passes": check 1 == 1
  test "fails":  check 1 == 2

jfmt.close()   # writes </testsuites> and closes the file
```

---

### `close` (JUnitOutputFormatter)

```nim
proc close*(formatter: JUnitOutputFormatter)
```

Finalizes the JUnit XML report by writing the closing `</testsuites>` tag and closing the underlying stream. Must be called after the last test has run.

---

## `disableParamFiltering`

```nim
proc disableParamFiltering*
```

By default, the framework reads command-line arguments at test startup and uses them as test-name filters. Calling `disableParamFiltering()` before the first test runs turns this off, so all tests run regardless of command-line arguments. Useful when embedding the test runner inside a larger application that has its own argument handling.

```nim
disableParamFiltering()
# now all tests run even if there are CLI arguments
```

---

## Compile-time and Environment Configuration

| Mechanism | Effect |
|---|---|
| `-d:nimUnittestOutputLevel:PRINT_ALL\|PRINT_FAILURES\|PRINT_NONE` | Set default output verbosity |
| `-d:nimUnittestColor:auto\|on\|off` | Set console color output |
| `-d:nimUnittestAbortOnError:on\|off` | Set default for `abortOnError` |
| `abortOnError = true` (runtime) | Same as above but at runtime |
| `addOutputFormatter(jfmt)` | Add JUnit or custom formatter |

Deprecated (still supported but may be removed in future versions):

| Environment Variable | Effect |
|---|---|
| `NIMTEST_OUTPUT_LVL` | `PRINT_ALL`, `PRINT_FAILURES`, or `PRINT_NONE` |
| `NIMTEST_COLOR` | `always` or `never` |
| `NIMTEST_NO_COLOR` | If set, disables color |
| `NIMTEST_ABORT_ON_ERROR` | If set, enables abort-on-error |

---

## Limitations

### Mixed-type comparisons in `check`

Because `check` rewrites comparisons to capture sub-expression values, it assigns each operand to a temporary variable. This can break when the operands have different types that cannot be implicitly converted:

```nim
check 4.0 == 2 + 2   # may fail to compile — type mismatch
doAssert 4.0 == 2 + 2  # works fine
```

**Workaround:** ensure both sides of any operator in a `check` expression have the same type, or use `doAssert` for mixed-type checks.

### Template arguments are not interpolated inside `check`

For the same compile-time reason, `check` cannot see template parameters by name — the string inside the format macro does not see the template's expanded context. This is the same limitation as in `strformat`. Bind the template argument to a local variable first:

```nim
template myAssert(val: int): untyped =
  let v = val
  check v > 0   # use v, not val
```

### `skip` does not halt test body execution

Calling `skip()` sets the test status to `SKIPPED` but does not exit the test body. If you want to skip all remaining assertions, place them in an `if` block conditioned on the skip criterion, or restructure the test to return early.

### Teardown is not run after `require` fails

If `require` fails, the program exits immediately. Any `teardown` blocks are bypassed. Use `check` instead of `require` if teardown cleanup is important for correctness.

---

## Quick-Reference Summary

### Test structure

```nim
suite "suite name":
  setup:
    # runs before each test in this suite

  teardown:
    # runs after each test in this suite

  test "test name":
    # test body

test "standalone test":
  # outside any suite
```

### Assertions

| Template / Macro | Behaviour on failure |
|---|---|
| `check expr` | Prints diagnostic, marks test FAILED, continues |
| `check: expr1; expr2` | Checks each independently |
| `require expr` | Quits the program immediately |
| `expect ExcType: body` | Fails if body does not raise ExcType |
| `fail` | Manually mark test as FAILED |
| `skip` | Mark test as SKIPPED (body continues) |
| `checkpoint("msg")` | Record a breadcrumb (printed on failure) |

### Formatters

| Proc | Purpose |
|---|---|
| `addOutputFormatter(f)` | Add a formatter |
| `delOutputFormatter(f)` | Remove a formatter |
| `resetOutputFormatters()` | Clear all formatters |
| `newConsoleOutputFormatter(level, color)` | Create console formatter |
| `defaultConsoleFormatter()` | Create default console formatter |
| `newJUnitOutputFormatter(stream)` | Create JUnit XML formatter |
| `jfmt.close()` | Finalize JUnit XML report |

### Configuration

| Option | Purpose |
|---|---|
| `abortOnError = true` | Quit on first failure |
| `disableParamFiltering()` | Ignore CLI test filters |
| `-d:nimUnittestOutputLevel:PRINT_FAILURES` | Show only failures |
| `-d:nimUnittestColor:off` | Disable colored output |
