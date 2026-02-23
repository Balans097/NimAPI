# cgi — Module Reference

> **Nim standard library** | `import std/cgi`

## What Is CGI?

**CGI (Common Gateway Interface)** is the original protocol for web servers to execute programs and return dynamic content. When a browser submits a form or hits a URL, the web server (Apache, Nginx, etc.) launches your executable and communicates with it through **environment variables** and **standard I/O**:

- The web server sets environment variables like `REQUEST_METHOD`, `QUERY_STRING`, `HTTP_HOST`, and so on before launching your program.
- For POST requests, the form body is piped to your program's `stdin`.
- Your program writes the HTTP response (starting with headers) to `stdout`.

This module provides everything needed to write CGI scripts in Nim: reading and decoding form data, accessing all standard CGI environment variables, working with cookies, encoding output safely, and debugging.

The module also re-exports `encodeUrl` and `decodeUrl` from `std/uri`, so you get URL encoding without an extra import.

---

## Table of Contents

1. [Types](#types)
   - [CgiError](#cgierror)
   - [RequestMethod](#requestmethod)
2. [Input — Reading Form Data](#input--reading-form-data)
   - [decodeData (from string)](#decodedata-from-string)
   - [decodeData (from environment)](#decodedata-from-environment)
   - [readData (from environment)](#readdata-from-environment)
   - [readData (from string)](#readdata-from-string)
   - [validateData](#validatedata)
3. [Output — Sending a Response](#output--sending-a-response)
   - [writeContentType](#writecontenttype)
   - [writeErrorMessage](#writeerrormessage)
   - [setStackTraceStdout](#setstacktracestdout)
   - [xmlEncode](#xmlencode)
4. [Cookies](#cookies)
   - [setCookie](#setcookie)
   - [getCookie](#getcookie)
   - [existsCookie](#existscookie)
5. [CGI Environment Variables](#cgi-environment-variables)
6. [Error Handling](#error-handling)
   - [cgiError](#cgierror-1)
7. [Testing and Debugging](#testing-and-debugging)
   - [setTestData](#settestdata)
8. [Re-exported from std/uri](#re-exported-from-stduri)
9. [Complete Example](#complete-example)
10. [Quick Reference](#quick-reference)

---

## Types

### `CgiError`

```nim
type CgiError* = object of IOError
```

The exception type raised when something goes wrong at the CGI protocol level — for example, the client uses an HTTP method that your script does not allow, or `stdin` cannot be read. Inherits from `IOError`, so it can be caught with `except IOError` if needed.

---

### `RequestMethod`

```nim
type RequestMethod* = enum
  methodNone,   ## REQUEST_METHOD environment variable is not set
  methodPost,   ## client used POST
  methodGet     ## client used GET
```

An enumeration representing the three possible states of the HTTP request method as seen by the CGI environment. Used as a filter in `decodeData` and `readData` to restrict which HTTP methods your script accepts.

- `methodNone` — the script is being run outside a web server (e.g., on the command line). Useful for testing.
- `methodPost` — the browser submitted a form with `method="POST"`.
- `methodGet` — the browser submitted a form with `method="GET"`, or followed a URL with a query string.

---

## Input — Reading Form Data

This is the most important part of any CGI script: getting the data the user submitted.

### `decodeData` (from string)

```nim
iterator decodeData*(data: string): tuple[key, value: string]
```

Decodes a URL-encoded query string (e.g., `name=Alice&age=30`) and yields `(key, value)` pairs one by one. The string is the raw encoded data — typically the contents of `QUERY_STRING` or a POST body you have read yourself.

URL encoding is handled automatically: `%20` becomes a space, `+` becomes a space, `%2F` becomes `/`, and so on.

```nim
let qs = "city=New+York&country=US&pop=8336817"
for key, val in decodeData(qs):
  echo key, " = ", val
# city = New York
# country = US
# pop = 8336817
```

---

### `decodeData` (from environment)

```nim
iterator decodeData*(allowedMethods: set[RequestMethod] =
    {methodNone, methodPost, methodGet}): tuple[key, value: string]
```

The most commonly used input iterator. It automatically reads from the right place — `stdin` for POST requests, `QUERY_STRING` for GET requests — decodes the data, and yields `(key, value)` pairs.

The `allowedMethods` parameter acts as a security gate: if the client uses an HTTP method not in the set, `CgiError` is raised immediately. This is how you enforce that your login form can only be submitted via POST, for example.

```nim
# Accept only POST; reject GET and direct invocations
for key, val in decodeData({methodPost}):
  echo key, " => ", val
```

```nim
# The default — accept everything (useful during development)
for key, val in decodeData():
  echo key, " => ", val
```

---

### `readData` (from environment)

```nim
proc readData*(allowedMethods: set[RequestMethod] =
               {methodNone, methodPost, methodGet}): StringTableRef
```

Like `decodeData` (environment version), but collects **all** key-value pairs into a `StringTableRef` (a case-insensitive hash table from `std/strtabs`) and returns it. This is usually more convenient than the iterator when you need to access individual fields by name afterwards.

If the same key appears more than once in the query string, only the **last** value is kept (the table key is overwritten).

```nim
let data = readData()
echo data["username"]
echo data["email"]
```

```nim
# Restrict to POST only
let data = readData({methodPost})
echo "Password: ", data["password"]
```

---

### `readData` (from string)

```nim
proc readData*(data: string): StringTableRef
```

Parses a raw URL-encoded string into a `StringTableRef`. Useful when you have already read the raw data yourself, or when writing unit tests without a web server.

```nim
let table = readData("lang=nim&version=2")
echo table["lang"]     # nim
echo table["version"]  # 2
```

---

### `validateData`

```nim
proc validateData*(data: StringTableRef, validKeys: varargs[string])
```

Checks that every key in `data` is listed in `validKeys`. If any unexpected key is found, `CgiError` is raised with a message identifying the unknown variable name.

This is a simple but effective defence against form tampering: if a user manually crafts a request with field names your script does not expect, validation catches it immediately.

```nim
let data = readData()
# Raises CgiError if data contains any key other than "name" or "email"
validateData(data, "name", "email")

echo data["name"]
echo data["email"]
```

---

## Output — Sending a Response

CGI responses must begin with HTTP headers followed by a blank line, then the body. Forgetting the headers causes a "500 Internal Server Error" in the browser.

### `writeContentType`

```nim
proc writeContentType*()
```

Writes `Content-type: text/html\n\n` to `stdout`. This is the minimal required HTTP header for an HTML response. Call this **before** writing any HTML content. Without it, the web server will not know how to interpret your output.

```nim
writeContentType()
write(stdout, "<html><body><h1>Hello from Nim CGI!</h1></body></html>")
```

---

### `writeErrorMessage`

```nim
proc writeErrorMessage*(data: string)
```

Attempts to "reset" the browser's HTML rendering state by closing any open tags, then writes `data` inside a `<plaintext>` element. This makes raw text (including stack traces) readable in the browser without HTML escaping. Used internally by `setStackTraceStdout`, but can also be called directly to display error output.

```nim
try:
  discard  # ... your logic ...
except CgiError as e:
  writeErrorMessage("CGI Error: " & e.msg)
```

---

### `setStackTraceStdout`

```nim
proc setStackTraceStdout*()
```

Redirects Nim's internal error/stacktrace output from the server's error log to `stdout` (the browser). This makes debugging much easier during development: instead of hunting through server logs, you see the Nim stack trace directly in the browser window.

**Call this as early as possible** — ideally as the very first line of your CGI script — so even errors that occur before `writeContentType` are captured and displayed.

```nim
setStackTraceStdout()
writeContentType()
# ... rest of the script ...
```

---

### `xmlEncode`

```nim
proc xmlEncode*(s: string): string
```

Escapes a string so it is safe to embed inside HTML or XML. The following substitutions are made:

| Character | Encoded as |
|---|---|
| `&` | `&amp;` |
| `<` | `&lt;` |
| `>` | `&gt;` |
| `"` | `&quot;` |

Every other character is passed through unchanged.

**Always use `xmlEncode` when inserting user-supplied data into HTML output.** Failing to do so allows cross-site scripting (XSS) attacks — a user could submit `<script>alert(1)</script>` as their "name" and have it execute in other users' browsers.

```nim
let userInput = "<script>alert('XSS')</script>"
write(stdout, "<p>" & xmlEncode(userInput) & "</p>")
# Outputs: <p>&lt;script&gt;alert('XSS')&lt;/script&gt;</p>
# The browser renders it as harmless text, not code.
```

---

## Cookies

Cookies are set by sending an HTTP header before the body, and read from the `HTTP_COOKIE` environment variable that the web server provides.

### `setCookie`

```nim
proc setCookie*(name, value: string)
```

Sends a `Set-Cookie` HTTP header to the browser, instructing it to store the cookie. The header is written to `stdout` and **must be sent before `writeContentType`** (i.e., before any blank line that terminates the headers).

```nim
setCookie("sessionid", "abc123xyz")
setCookie("lang", "en")
writeContentType()
# ... HTML body ...
```

> **Note:** This is a bare-bones cookie setter — it sets only `name=value` with no `Path`, `Expires`, `HttpOnly`, or `Secure` attributes. For production use, set those attributes manually or use `std/cookies`.

---

### `getCookie`

```nim
proc getCookie*(name: string): string
```

Returns the value of the cookie named `name` from the current request. If no such cookie exists, returns an empty string `""`. The cookie data is parsed lazily from `HTTP_COOKIE` on the first call and cached in a thread-local table for subsequent calls.

```nim
let session = getCookie("sessionid")
if session == "":
  echo "No session — please log in."
else:
  echo "Welcome back! Session: ", session
```

---

### `existsCookie`

```nim
proc existsCookie*(name: string): bool
```

Returns `true` if a cookie with the given `name` is present in the current request, `false` otherwise. Use this when you need to distinguish between "cookie not set" and "cookie set to an empty string" — `getCookie` cannot make that distinction.

```nim
if existsCookie("preferences"):
  let prefs = getCookie("preferences")
  applyPreferences(prefs)
else:
  applyDefaults()
```

---

## CGI Environment Variables

The web server populates a standard set of environment variables before launching a CGI script. This module provides a dedicated getter function for each one. All functions return an empty string `""` when the corresponding variable is not set.

| Procedure | Environment Variable | Typical content |
|---|---|---|
| `getContentLength()` | `CONTENT_LENGTH` | Size of POST body in bytes, e.g. `"1234"` |
| `getContentType()` | `CONTENT_TYPE` | MIME type of POST body, e.g. `"application/x-www-form-urlencoded"` |
| `getDocumentRoot()` | `DOCUMENT_ROOT` | Absolute path to the server's document root |
| `getGatewayInterface()` | `GATEWAY_INTERFACE` | CGI version, e.g. `"CGI/1.1"` |
| `getHttpAccept()` | `HTTP_ACCEPT` | MIME types the browser will accept |
| `getHttpAcceptCharset()` | `HTTP_ACCEPT_CHARSET` | Character sets the browser prefers |
| `getHttpAcceptEncoding()` | `HTTP_ACCEPT_ENCODING` | Compression the browser supports, e.g. `"gzip, deflate"` |
| `getHttpAcceptLanguage()` | `HTTP_ACCEPT_LANGUAGE` | Browser's preferred languages, e.g. `"en-US,en;q=0.9"` |
| `getHttpConnection()` | `HTTP_CONNECTION` | Connection management, e.g. `"keep-alive"` |
| `getHttpCookie()` | `HTTP_COOKIE` | All cookies for this domain as a raw string |
| `getHttpHost()` | `HTTP_HOST` | The hostname from the request, e.g. `"example.com"` |
| `getHttpReferer()` | `HTTP_REFERER` | URL of the page the user came from |
| `getHttpUserAgent()` | `HTTP_USER_AGENT` | Browser identification string |
| `getPathInfo()` | `PATH_INFO` | Extra path component after the script name |
| `getPathTranslated()` | `PATH_TRANSLATED` | `PATH_INFO` translated to a filesystem path |
| `getQueryString()` | `QUERY_STRING` | Everything after `?` in the URL |
| `getRemoteAddr()` | `REMOTE_ADDR` | Client's IP address |
| `getRemoteHost()` | `REMOTE_HOST` | Client's hostname (if reverse DNS is enabled) |
| `getRemoteIdent()` | `REMOTE_IDENT` | Remote user identity (RFC 931, rarely used) |
| `getRemotePort()` | `REMOTE_PORT` | Client's TCP port number |
| `getRemoteUser()` | `REMOTE_USER` | Authenticated username (if HTTP auth is configured) |
| `getRequestMethod()` | `REQUEST_METHOD` | `"GET"`, `"POST"`, etc. |
| `getRequestURI()` | `REQUEST_URI` | Full request URI including query string |
| `getScriptFilename()` | `SCRIPT_FILENAME` | Absolute path to the CGI script on disk |
| `getScriptName()` | `SCRIPT_NAME` | URL path to the script, e.g. `"/cgi-bin/app"` |
| `getServerAddr()` | `SERVER_ADDR` | Server's IP address |
| `getServerAdmin()` | `SERVER_ADMIN` | Server administrator email |
| `getServerName()` | `SERVER_NAME` | Server hostname or IP as seen by the client |
| `getServerPort()` | `SERVER_PORT` | Port the server is listening on, e.g. `"80"` |
| `getServerProtocol()` | `SERVER_PROTOCOL` | HTTP version, e.g. `"HTTP/1.1"` |
| `getServerSignature()` | `SERVER_SIGNATURE` | Server version banner (if enabled) |
| `getServerSoftware()` | `SERVER_SOFTWARE` | Server software name and version |

```nim
# Log some diagnostic information
echo "Method: ", getRequestMethod()
echo "Client IP: ", getRemoteAddr()
echo "Host: ", getHttpHost()
echo "User-Agent: ", getHttpUserAgent()
```

---

## Error Handling

### `cgiError`

```nim
proc cgiError*(msg: string) {.noreturn.}
```

Raises a `CgiError` exception with the given message. Marked `{.noreturn.}` — it never returns normally. Use this to signal protocol-level errors in your own CGI validation code.

```nim
let method = getRequestMethod()
if method != "POST":
  cgiError("This endpoint only accepts POST requests, got: " & method)
```

---

## Testing and Debugging

### `setTestData`

```nim
proc setTestData*(keysvalues: varargs[string])
```

Simulates a GET request by populating the `REQUEST_METHOD` and `QUERY_STRING` environment variables with the given key-value pairs. Pairs are provided as alternating strings: `key1, value1, key2, value2, ...`.

This lets you test your CGI logic directly from the command line or from unit tests, without a web server. Only GET requests can be simulated this way.

```nim
when defined(debug):
  setTestData("username", "alice", "lang", "nim", "version", "2")

let data = readData()
echo data["username"]  # alice
echo data["lang"]      # nim
```

After this call, `readData()` and `decodeData()` behave exactly as if the browser had sent `?username=alice&lang=nim&version=2`.

---

## Re-exported from std/uri

The module re-exports two functions from `std/uri` so you can use them without an additional import:

**`encodeUrl(s: string): string`** — percent-encodes a string for use in a URL. Spaces become `%20`, special characters like `&`, `=`, `#` are escaped.

```nim
let safe = encodeUrl("hello world & more")
# hello%20world%20%26%20more
```

**`decodeUrl(s: string): string`** — reverses `encodeUrl`. Converts `%XX` sequences back to their original characters.

```nim
let original = decodeUrl("hello%20world%20%26%20more")
# hello world & more
```

---

## Complete Example

The following is a minimal but complete CGI script that accepts a name from a form, validates it, and responds with a personalised greeting. It demonstrates the full typical workflow.

```nim
import std/[strtabs, cgi]

# 1. Redirect stacktraces to browser during development
setStackTraceStdout()

# 2. Simulate test data when compiled with -d:debug
when defined(debug):
  setTestData("name", "Alice", "lang", "en")

# 3. Read submitted form data (POST or GET)
let data = readData()

# 4. Validate that only the expected fields are present
validateData(data, "name", "lang")

# 5. Send the Content-Type header
writeContentType()

# 6. Generate HTML — always xmlEncode user-supplied values
let name = xmlEncode(data["name"])
let lang = xmlEncode(data["lang"])

write(stdout, "<!DOCTYPE html>\n")
write(stdout, "<html><head><title>Greeting</title></head><body>\n")
write(stdout, "<h1>Hello, " & name & "!</h1>\n")
write(stdout, "<p>Your preferred language: " & lang & "</p>\n")
write(stdout, "<p>Your IP address: " & getRemoteAddr() & "</p>\n")
write(stdout, "</body></html>\n")
```

---

## Quick Reference

| Symbol | Description |
|---|---|
| `CgiError` | Exception type for CGI protocol errors |
| `RequestMethod` | Enum: `methodNone`, `methodPost`, `methodGet` |
| `decodeData(string)` | Iterator: decode URL-encoded string → (key, value) pairs |
| `decodeData(set)` | Iterator: read from environment → (key, value) pairs |
| `readData(set)` | Read all form data from environment → `StringTableRef` |
| `readData(string)` | Parse raw URL-encoded string → `StringTableRef` |
| `validateData(table, keys...)` | Raise `CgiError` if unexpected keys are present |
| `writeContentType()` | Write `Content-type: text/html` header to stdout |
| `writeErrorMessage(string)` | Write error text safely to browser |
| `setStackTraceStdout()` | Redirect Nim stacktraces to browser |
| `xmlEncode(string)` | Escape `& < > "` for safe HTML embedding |
| `setCookie(name, value)` | Send `Set-Cookie` header |
| `getCookie(name)` | Get cookie value by name (`""` if absent) |
| `existsCookie(name)` | Check if a cookie exists → `bool` |
| `cgiError(msg)` | Raise `CgiError` with message |
| `setTestData(key, val, ...)` | Simulate a GET request for testing |
| `encodeUrl(string)` | Percent-encode a URL component (re-exported) |
| `decodeUrl(string)` | Decode a percent-encoded string (re-exported) |
| `getContentLength()` … `getServerSoftware()` | Read CGI environment variables |
