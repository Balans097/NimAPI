# smtp.nim — Module Reference

> **Installation:** `nimble install smtp`  
> **Standards:** RFC 5321 (SMTP protocol), RFC 2822 (message format)  
> **SSL support:** compile with `-d:ssl`

This module provides everything you need to compose and send email from a Nim program. It covers two orthogonal concerns that are easy to conflate:

1. **Message composition** — building a well-formed RFC 2822 email object (headers, body, recipients).
2. **SMTP transport** — connecting to a mail server, authenticating, and handing the message off for delivery.

Both synchronous (`Smtp`) and asynchronous (`AsyncSmtp`) clients are available; their APIs are identical in shape.

---

## Types

### `Email`
Represents a single email address, optionally paired with a human-readable display name.

```nim
# Internal fields (accessed via procs):
#   address: string   — the actual mailbox, e.g. "alice@example.com"
#   name:    string   — optional display name, e.g. "Alice"
```

### `Message`
A complete RFC 2822 message — headers and body — ready to be serialised and sent.

### `Smtp` / `AsyncSmtp`
A handle to an open (or pending) connection to an SMTP server. Use `Smtp` for blocking code and `AsyncSmtp` inside `async` procedures.

### `ReplyError`
Raised whenever the SMTP server returns an unexpected response code. Inherits from `IOError`.

### `Port` *(re-exported from `net`)*
A distinct integer type for TCP port numbers, e.g. `Port 465`.

---

## Message Construction

### `createEmail`

```nim
proc createEmail*(address: string, name: string = ""): Email
```

**What it does.** Constructs an `Email` value from a raw mailbox address and an optional display name. The display name appears in email clients as the human-friendly label shown next to (or instead of) the bare address.

**Important:** Neither `address` nor `name` may contain carriage-return (`\r`) or newline (`\n`) characters — such characters are the building blocks of header-injection attacks. Passing them raises `AssertionDefect`.

```nim
let alice = createEmail("alice@example.com", "Alice Smith")
let bot   = createEmail("no-reply@example.com")   # name is optional

echo alice  # → "Alice Smith" <alice@example.com>
echo bot    # → no-reply@example.com
```

---

### `createMessage`

```nim
proc createMessage*[T: Email | string](
    mSubject, mBody: string,
    sender: T,
    mTo:      seq[T] = @[],
    mCc:      seq[T] = @[],
    mBcc:     seq[T] = @[],
    mReplyTo: seq[T] = @[],
    otherHeaders: openArray[tuple[name, value: string]] = @[],
): Message
```

**What it does.** Creates a fully-structured `Message` object. Think of this as filling in the "compose" form of an email client: you supply a subject, a body, the sender's address, and the various recipient lists. The result is a structured object — it is not yet a string of RFC 2822 text; call `$msg` when you need the wire format.

**Type parameter `T`.** Accepts either plain `string` addresses or `Email` objects, making it easy to use with or without display names.

**Header injection protection.** The subject must not contain newlines (`AssertionDefect` otherwise).

```nim
# Simple usage with plain strings
let msg = createMessage(
    mSubject = "Monthly report",
    mBody    = "Please find the report attached.",
    sender   = "reports@example.com",
    mTo      = @["manager@example.com"],
    mCc      = @["archive@example.com"],
)

# Rich usage with Email objects and extra headers
let sender = createEmail("alerts@example.com", "Alert System")
let boss   = createEmail("cto@example.com", "CTO")

let msg2 = createMessage(
    mSubject     = "⚠ Disk usage critical",
    mBody        = "Server /dev/sda1 is at 95% capacity.",
    sender       = sender,
    mTo          = @[boss],
    otherHeaders = @[("X-Priority", "1"), ("X-Mailer", "MyApp/1.0")],
)
```

> **Deprecated overloads** — `createMessage(mSubject, mBody, mTo, mCc)` and `createMessage(mSubject, mBody, mTo, mCc, otherHeaders)` exist for backwards compatibility but do not accept a `sender`. Prefer the overload shown above.

---

### `sender`

```nim
proc sender*(msg: Message): Email
```

**What it does.** Returns the `Email` value that was recorded as the message sender when `createMessage` was called. Useful when you want to inspect or log the sender address programmatically without parsing the raw message string.

```nim
let msg = createMessage("Hi", "Body", "alice@example.com", mTo = @["bob@example.com"])
echo msg.sender().address   # → alice@example.com
```

---

### `recipients`

```nim
proc recipients*(msg: Message): seq[Email]
```

**What it does.** Returns the combined list of **all** recipient `Email` values: To + Cc + Bcc. This is exactly the set of addresses that must appear in `RCPT TO` SMTP commands — it includes Bcc addresses even though those are hidden from the message headers as seen by the recipient.

```nim
let msg = createMessage(
    "Hello", "Body", "me@example.com",
    mTo  = @["a@example.com"],
    mCc  = @["b@example.com"],
    mBcc = @["c@example.com"],
)

for r in msg.recipients():
    echo r.address   # a@example.com, b@example.com, c@example.com
```

---

### `$` (Email → string)

```nim
proc `$`*(email: Email): string
```

**What it does.** Converts an `Email` to its RFC 2822 string form. When a display name is present the output is `"Display Name" <address>`; without a name it is simply `address`.

```nim
echo createEmail("alice@example.com", "Alice")   # "Alice" <alice@example.com>
echo createEmail("alice@example.com")             # alice@example.com
```

---

### `$` (Message → string)

```nim
proc `$`*(msg: Message): string
```

**What it does.** Serialises the `Message` into a complete RFC 2822-formatted string: all headers followed by a blank line followed by the body. This is the text you pass to `sendMail` (and what mail servers actually transmit).

```nim
let msg = createMessage("Test", "Hello!", "me@example.com", mTo = @["you@example.com"])
echo $msg
# From: me@example.com
# To: you@example.com
# Subject: Test
#
# Hello!
```

---

## Connection Management

### `newSmtp`

```nim
proc newSmtp*(useSsl = false, debug = false, sslContext: SslContext = nil): Smtp
```

**What it does.** Allocates and returns a new synchronous SMTP client object. The object holds the underlying TCP (or SSL-wrapped) socket and configuration, but does **not** connect yet — call `connect` afterwards.

| Parameter | Meaning |
|-----------|---------|
| `useSsl` | Wrap the socket with SSL/TLS immediately (suitable for implicit-TLS ports like 465). Requires `-d:ssl`. |
| `debug` | Echo every sent (`C:`) and received (`S:`) line to stdout. Invaluable during development. |
| `sslContext` | Provide your own `SslContext` for custom certificate settings; `nil` uses a built-in permissive context. |

```nim
# Plain connection (port 25 / 587 with STARTTLS later)
let smtp = newSmtp(debug = true)

# Implicit SSL (port 465)
let smtps = newSmtp(useSsl = true)
```

---

### `newAsyncSmtp`

```nim
proc newAsyncSmtp*(useSsl = false, debug = false, sslContext: SslContext = nil): AsyncSmtp
```

**What it does.** Same as `newSmtp` but returns an `AsyncSmtp` suitable for use inside `async` procedures. All subsequent calls (`connect`, `auth`, `sendMail`, …) must be `await`-ed.

```nim
import asyncdispatch

proc main() {.async.} =
    let smtp = newAsyncSmtp(useSsl = true, debug = true)
    await smtp.connect("smtp.gmail.com", Port 465)
    # ...

waitFor main()
```

---

### `connect`

```nim
proc connect*(smtp: Smtp | AsyncSmtp, address: string, port: Port) {.multisync.}
```

**What it does.** Opens the TCP connection to the mail server, waits for the greeting banner (`220`), and then negotiates capabilities via `EHLO` (falling back to `HELO` for old servers). This is the moment the network round-trip actually happens.

Raises `ReplyError` if the server greeting is not `220`, or a socket error if the host is unreachable.

```nim
smtp.connect("smtp.example.com", Port 587)   # STARTTLS port
smtp.connect("smtp.gmail.com",   Port 465)   # implicit SSL port
```

---

### `startTls`

```nim
proc startTls*(smtp: Smtp | AsyncSmtp, sslContext: SslContext = nil) {.multisync.}
```

**What it does.** Upgrades a plain TCP connection to TLS by sending the `STARTTLS` command and performing the TLS handshake. Use this after `connect` on a non-SSL port (typically 587) but **before** `auth`. After the handshake the module re-sends `EHLO` to re-negotiate capabilities over the encrypted channel.

Requires `-d:ssl`. Raises `ReplyError` if the server does not accept `STARTTLS`.

```nim
smtp.connect("smtp.mailtrap.io", Port 2525)
smtp.startTls()          # upgrade to TLS
smtp.auth("user", "pw")  # credentials sent only over the encrypted link
```

---

### `auth`

```nim
proc auth*(smtp: Smtp | AsyncSmtp, username, password: string) {.multisync.}
```

**What it does.** Authenticates to the SMTP server using the `AUTH LOGIN` mechanism. Credentials are Base64-encoded (as required by the protocol) and exchanged in a challenge-response sequence. On success the server replies `235`; any other code raises `ReplyError`.

> **Security note:** Base64 is *not* encryption. Always call `startTls` (or use `useSsl = true`) before `auth` so the credentials travel inside an encrypted channel.

```nim
smtp.auth("myuser@gmail.com", "app-specific-password")
```

---

### `close`

```nim
proc close*(smtp: Smtp | AsyncSmtp) {.multisync.}
```

**What it does.** Sends the `QUIT` command to gracefully terminate the SMTP session and then closes the underlying socket. Always call this when you are done sending mail to let the server clean up the session properly.

```nim
smtp.sendMail(msg)
smtp.close()
```

---

## Sending Mail

### `sendMail` (raw strings)

```nim
proc sendMail*(smtp: Smtp | AsyncSmtp, fromAddr: string, toAddrs: seq[string], msg: string) {.multisync.}
```

**What it does.** Performs the full SMTP message submission sequence: `MAIL FROM`, one `RCPT TO` per recipient, `DATA`, message body, and terminating `.`. The `msg` string should be a complete RFC 2822-formatted message (i.e. the output of `$myMessage`).

This is the lower-level overload. You manage the from-address and recipient list yourself, which is useful when you need fine-grained control (e.g. envelope sender differs from the `From:` header).

**Header injection protection.** Both `fromAddr` and every element of `toAddrs` must not contain newlines — `AssertionDefect` is raised otherwise.

```nim
let body = $msg   # serialise Message to RFC 2822 string
smtp.sendMail("sender@example.com", @["recipient@example.com"], body)
```

---

### `sendMail` (Message convenience overload)

```nim
proc sendMail*(smtp: Smtp | AsyncSmtp, msg: Message) {.multisync.}
```

**What it does.** A convenience wrapper around the raw-strings overload. It extracts the sender address from `msg.sender()` and the full recipient list from `msg.recipients()` (To + Cc + Bcc), serialises the message, and calls through to `sendMail`. For most applications this is all you need.

```nim
let msg = createMessage("Subject", "Body", "me@example.com", mTo = @["you@example.com"])
smtp.sendMail(msg)   # sender and recipients derived automatically
```

---

## Low-Level / Extension API

These procedures are part of the public API but are intended for advanced use cases — such as implementing custom SMTP extensions (e.g. `AUTH PLAIN`, `CHUNKING`) — rather than day-to-day mail sending.

### `debugSend`

```nim
proc debugSend*(smtp: Smtp | AsyncSmtp, cmd: string) {.multisync.}
```

**What it does.** Sends a raw string over the SMTP socket. If `debug = true` it first prints `C:<cmd>` to stdout. Use this when you need to issue SMTP commands that the module does not expose as first-class procs.

```nim
smtp.debugSend("NOOP\c\L")   # keep-alive ping
```

---

### `debugRecv`

```nim
proc debugRecv*(smtp: Smtp | AsyncSmtp): Future[string] {.multisync.}
```

**What it does.** Reads one line from the SMTP server. If `debug = true` it prints `S:<line>` to stdout. Returns the raw response line as a string.

```nim
let banner = smtp.debugRecv()   # e.g. "220 smtp.example.com ESMTP"
```

---

### `checkReply`

```nim
proc checkReply*(smtp: Smtp | AsyncSmtp, reply: string) {.multisync.}
```

**What it does.** Reads the next server response with `debugRecv` and verifies that it starts with the expected `reply` prefix (typically a 3-digit status code). If the response does not match, a `QUIT` is sent and `ReplyError` is raised. Combine with `debugSend` when building custom SMTP command sequences.

```nim
smtp.debugSend("NOOP\c\L")
smtp.checkReply("250")   # raises ReplyError if server replied something else
```

---

## Complete Examples

### Gmail (implicit SSL, port 465)

```nim
import smtp

let msg = createMessage(
    mSubject = "Hello from Nim",
    mBody    = "This was sent by a Nim program.",
    sender   = "me@gmail.com",
    mTo      = @["friend@example.com"],
)

let smtp = newSmtp(useSsl = true, debug = true)
smtp.connect("smtp.gmail.com", Port 465)
smtp.auth("me@gmail.com", "app-password")
smtp.sendMail(msg)
smtp.close()
```

### STARTTLS (port 587 / 2525)

```nim
import smtp

let msg = createMessage(
    mSubject = "STARTTLS test",
    mBody    = "Encrypted with STARTTLS.",
    sender   = "me@example.com",
    mTo      = @["you@example.com"],
)

let smtp = newSmtp(debug = true)
smtp.connect("smtp.mailtrap.io", Port 2525)
smtp.startTls()
smtp.auth("username", "password")
smtp.sendMail(msg)
smtp.close()
```

### Async send

```nim
import smtp, asyncdispatch

proc sendAsync() {.async.} =
    let msg = createMessage(
        "Async mail", "Sent asynchronously!",
        sender = "bot@example.com",
        mTo = @["admin@example.com"],
    )
    let smtp = newAsyncSmtp(useSsl = true)
    await smtp.connect("smtp.example.com", Port 465)
    await smtp.auth("bot@example.com", "secret")
    await smtp.sendMail(msg)
    await smtp.close()

waitFor sendAsync()
```
