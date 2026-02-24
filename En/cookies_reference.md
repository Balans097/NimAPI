# `cookies` Module Reference

> **Module:** `std/cookies`  
> **Purpose:** Parsing the `Cookie` request header sent by clients, and building `Set-Cookie` response headers sent by servers.

---

## Overview

HTTP cookies are small key-value pairs that a server instructs the browser to store and send back on subsequent requests. The `cookies` module covers both directions of this flow:

- **Parsing** — reading the `Cookie: a=1; b=2` header that a client sends to the server (`parseCookies`).
- **Building** — generating the `Set-Cookie: key=value; Domain=…; Secure; …` header that a server sends to the client (`setCookie`).

The module is deliberately minimal. It does not perform URL-encoding/decoding of cookie values — that responsibility falls on the caller.

---

## Exported Symbols

---

### `SameSite`

```nim
type SameSite* {.pure.} = enum
  Default, None, Lax, Strict
```

#### What it is

An enum that represents the value of the `SameSite` cookie attribute, which controls when the browser is allowed to send a cookie along with cross-site requests. This attribute is the primary defence against Cross-Site Request Forgery (CSRF) attacks.

#### Values explained

| Value | Effect |
|---|---|
| `Default` | `setCookie` will **not** emit a `SameSite` attribute at all. The browser applies its own default (usually `Lax` in modern browsers). |
| `None` | The cookie is sent with all requests, including cross-site ones. **Requires `secure = true`**, otherwise `setCookie` will raise a `doAssert` error at runtime. |
| `Lax` | The cookie is sent with same-site requests and top-level navigations (e.g., clicking a link), but not with cross-site subrequests (images, iframes). A reasonable middle ground. |
| `Strict` | The cookie is sent **only** with same-site requests. Maximum isolation; the user will not be authenticated when arriving from an external link. |

#### Example

```nim
import std/cookies

# SameSite.Strict: maximum protection, the cookie is never sent cross-site
echo setCookie("session", "abc123", secure = true,
               httpOnly = true, sameSite = SameSite.Strict)
# → Set-Cookie: session=abc123; Secure; HttpOnly; SameSite=Strict

# SameSite.Lax: good default for most web apps
echo setCookie("prefs", "dark-mode", sameSite = SameSite.Lax)
# → Set-Cookie: prefs=dark-mode; SameSite=Lax

# SameSite.None without Secure — this will crash with a doAssert:
# echo setCookie("track", "x", sameSite = SameSite.None)
```

---

### `parseCookies`

```nim
proc parseCookies*(s: string): StringTableRef
```

#### What it does

Parses the value of a `Cookie` HTTP request header — the compact, semicolon-separated format that **clients** (browsers) send to servers — and returns a case-insensitive `StringTable` mapping cookie names to their values.

#### Important distinction

This procedure is designed for the **`Cookie`** header (client → server), which looks like:

```
Cookie: name1=value1; name2=value2
```

It is **not** intended for parsing `Set-Cookie` headers (server → client), which have a completely different format with attributes like `Domain`, `Path`, `Expires`, etc.

#### How it works internally

The parser walks through the string character by character:

1. Skips leading spaces and tabs before each cookie name.
2. Reads characters up to `=` to get the key.
3. Reads characters up to `;` (or end of string) to get the value.
4. Stores the pair in the table and moves past the `;` separator.

There is no URL-decoding. Values are stored verbatim as they appear in the header.

#### The result is case-insensitive

The returned `StringTableRef` uses `modeCaseInsensitive`, meaning `cookieJar["Session"]`, `cookieJar["session"]`, and `cookieJar["SESSION"]` all refer to the same entry.

#### Parameters

| Name | Type | Description |
|---|---|---|
| `s` | `string` | The raw value of the `Cookie` header (without the `Cookie:` prefix itself). |

#### Return value

A `StringTableRef` (from `std/strtabs`) where keys are cookie names and values are cookie values.

#### Examples

```nim
import std/cookies, std/strtabs

# Basic parsing
let jar = parseCookies("a=1; foo=bar")
assert jar["a"] == "1"
assert jar["foo"] == "bar"
```

```nim
import std/cookies, std/strtabs

# Case-insensitive key lookup
let jar = parseCookies("Session=xyz; Theme=dark")
assert jar["session"] == "xyz"   # lowercase works
assert jar["THEME"] == "dark"    # uppercase works too
```

```nim
import std/cookies, std/strtabs

# Typical server-side usage: extracting the Cookie header from an HTTP request
# (pseudo-code; actual header retrieval depends on your HTTP framework)
let rawCookieHeader = "user_id=42; csrf_token=abc; lang=en"
let cookies = parseCookies(rawCookieHeader)

if "user_id" in cookies:
  echo "Authenticated user: ", cookies["user_id"]
else:
  echo "No user cookie found — guest access"
```

```nim
import std/cookies, std/strtabs

# Graceful handling of an empty Cookie header
let cookies = parseCookies("")
echo cookies.len  # → 0, no crash
```

---

### `setCookie` (string expires)

```nim
proc setCookie*(key, value: string, domain = "", path = "",
                expires = "", noName = false,
                secure = false, httpOnly = false,
                maxAge = none(int),
                sameSite = SameSite.Default): string
```

#### What it does

Builds a `Set-Cookie` HTTP response header string. The server includes this string verbatim in the HTTP response, instructing the browser to store the cookie with the specified attributes.

#### Parameters in depth

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `key` | `string` | *(required)* | Cookie name. |
| `value` | `string` | *(required)* | Cookie value. |
| `domain` | `string` | `""` | Restricts the cookie to the specified domain and its subdomains. If omitted, the cookie applies only to the exact host that set it. |
| `path` | `string` | `""` | Restricts the cookie to URLs whose path begins with this value. `"/"` means the whole site. |
| `expires` | `string` | `""` | Expiry date as a pre-formatted string (RFC 1123, e.g. `"Mon, 01 Jan 2030 00:00:00 GMT"`). If omitted, the cookie is a session cookie and is deleted when the browser closes. |
| `noName` | `bool` | `false` | When `true`, omits the `Set-Cookie:` prefix. Useful when you are composing headers yourself and need only the attribute string. |
| `secure` | `bool` | `false` | When `true`, the browser will only send this cookie over HTTPS connections. Strongly recommended for sensitive cookies. |
| `httpOnly` | `bool` | `false` | When `true`, JavaScript running in the page cannot access this cookie via `document.cookie`. This is the primary defence against XSS-based cookie theft. |
| `maxAge` | `Option[int]` | `none(int)` | Cookie lifetime in **seconds** from the moment it is received. Takes precedence over `Expires` in modern browsers. Pass `some(0)` to delete a cookie immediately. |
| `sameSite` | `SameSite` | `SameSite.Default` | Controls cross-site sending behaviour. See the `SameSite` section above. |

#### Security note from the module itself

> *"Cookies can be vulnerable. Consider setting `secure=true`, `httpOnly=true` and `sameSite=Strict`."*

For session cookies and authentication tokens this combination is almost always the right choice.

#### Examples

```nim
import std/cookies

# Minimal cookie — just a name and value
echo setCookie("lang", "en")
# → Set-Cookie: lang=en
```

```nim
import std/cookies

# A well-secured session cookie
echo setCookie("session_id", "s3cr3t",
               path = "/",
               secure = true,
               httpOnly = true,
               sameSite = SameSite.Strict)
# → Set-Cookie: session_id=s3cr3t; Path=/; Secure; HttpOnly; SameSite=Strict
```

```nim
import std/cookies, std/options

# Cookie that expires after 1 hour (3600 seconds)
echo setCookie("token", "abc", maxAge = some(3600))
# → Set-Cookie: token=abc; Max-Age=3600
```

```nim
import std/cookies, std/options

# Deleting a cookie by setting Max-Age to 0
echo setCookie("session_id", "", maxAge = some(0))
# → Set-Cookie: session_id=; Max-Age=0
```

```nim
import std/cookies

# noName=true: get only the cookie attributes string without "Set-Cookie: " prefix
let attrs = setCookie("id", "99", path = "/api", noName = true)
echo attrs
# → id=99; Path=/api
```

```nim
import std/cookies

# SameSite=None requires Secure=true — otherwise doAssert fires
# This would crash at runtime:
# echo setCookie("x", "y", sameSite = SameSite.None)

# Correct usage:
echo setCookie("x", "y", secure = true, sameSite = SameSite.None)
# → Set-Cookie: x=y; Secure; SameSite=None
```

---

### `setCookie` (DateTime/Time expires)

```nim
proc setCookie*(key, value: string, expires: DateTime | Time,
                domain = "", path = "", noName = false,
                secure = false, httpOnly = false,
                maxAge = none(int),
                sameSite = SameSite.Default): string
```

#### What it does

An overload of `setCookie` that accepts a `DateTime` or `Time` value (from `std/times`) instead of a hand-crafted expiry string. It automatically formats the date into the RFC 1123 format required by the HTTP standard (`ddd, dd MMM yyyy HH:mm:ss GMT`) and then delegates to the string-based overload.

This is the preferred way to set an expiry date, because it eliminates the risk of formatting the date string incorrectly.

#### How the conversion works

The `expires` argument is first converted to UTC with `.utc`, then formatted as:

```
Mon, 01 Jan 2030 00:00:00 GMT
```

The resulting string is passed to the first `setCookie` overload as its `expires` parameter.

#### Parameters

All parameters are identical to the string-based overload, except `expires`:

| Parameter | Type | Description |
|---|---|---|
| `expires` | `DateTime` or `Time` | The expiry moment. Gets converted to UTC automatically. |

#### Examples

```nim
import std/cookies, std/times

# Cookie that expires at a specific calendar date
let expiry = dateTime(2030, mJan, 1, 0, 0, 0, zone = utc())
echo setCookie("promo", "SALE2030", expires = expiry)
# → Set-Cookie: promo=SALE2030; Expires=Tue, 01 Jan 2030 00:00:00 GMT
```

```nim
import std/cookies, std/times

# Cookie that expires 30 days from now
let in30days = now().utc + 30.days
echo setCookie("remember_me", "token_xyz",
               expires = in30days,
               path = "/",
               secure = true,
               httpOnly = true,
               sameSite = SameSite.Strict)
# → Set-Cookie: remember_me=token_xyz; Expires=...; Path=/; Secure; HttpOnly; SameSite=Strict
```

```nim
import std/cookies, std/times

# Using Time instead of DateTime
let t: Time = getTime() + initDuration(hours = 1)
echo setCookie("flash", "welcome", expires = t)
# → Set-Cookie: flash=welcome; Expires=<1 hour from now in GMT>
```

---

## Quick Reference: Attribute Combinations

| Goal | Recommended combination |
|---|---|
| Session authentication token | `secure=true`, `httpOnly=true`, `sameSite=Strict`, `path="/"` |
| Remember-me / persistent login | same as above + `expires` or `maxAge` set to future |
| CSRF token readable by JS | `sameSite=Strict` (no `httpOnly`) |
| Third-party / cross-site embed | `secure=true`, `sameSite=None` |
| Delete (expire) a cookie | `maxAge=some(0)` |
| Inspect the raw attribute string | `noName=true` |

---

## Common Pitfalls

**1. `parseCookies` vs `Set-Cookie` parsing**  
`parseCookies` only understands the compact `Cookie:` format. Do not pass it a `Set-Cookie:` header — it will misparse the attributes (`Domain`, `Path`, etc.) as cookie values.

**2. `SameSite=None` without `Secure`**  
Passing `sameSite = SameSite.None` without `secure = true` triggers a `doAssert` and crashes the program. This matches the browser requirement: `SameSite=None` cookies must be sent over HTTPS.

**3. No automatic URL-encoding**  
Neither `parseCookies` nor `setCookie` encodes or decodes cookie values. If your values may contain `=`, `;`, or non-ASCII characters, encode them (e.g. with `std/uri.encodeUrl`) before passing them in, and decode them after parsing.

**4. `Expires` vs `Max-Age`**  
`Max-Age` (in seconds, relative) takes precedence over `Expires` (absolute date) in all modern browsers. Prefer `maxAge` for reliability; use `expires` only for backwards compatibility with very old clients.
