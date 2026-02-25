# `varints` Module Reference

> Variable-length integer encoding inspired by SQLite.  
> **API status:** Unstable.

---

## What are varints?

A *varint* (variable-length integer) is a compact binary encoding for integers where small values occupy fewer bytes than large ones. Instead of always using 8 bytes for a `uint64`, a value like `42` is stored in just **1 byte**, while the full 64-bit range still fits in at most **9 bytes**.

This is especially useful in file formats, network protocols, and databases where most numbers are small and storage efficiency matters.

The encoding used here mirrors the one in **SQLite 4** and works as follows:

| Value range        | Bytes used |
|--------------------|-----------|
| 0 – 240            | 1         |
| 241 – 2 287        | 2         |
| 2 288 – 67 823     | 3         |
| 67 824 – 16 777 215| 4         |
| …up to 2⁶⁴−1      | up to 9   |

---

## Exported symbols

### `maxVarIntLen` — constant

```nim
const maxVarIntLen* = 9
```

**What it is:** The maximum number of bytes any single varint can occupy.

Use this constant to allocate buffers safely — no varint will ever exceed 9 bytes, regardless of the value being encoded.

**Example:**

```nim
var buf: array[maxVarIntLen, byte]   # always large enough
let written = writeVu64(buf, someValue)
```

---

### `writeVu64` — encode an unsigned 64-bit integer

```nim
proc writeVu64*(z: var openArray[byte], x: uint64): int
```

**What it does:** Encodes the unsigned integer `x` into the byte buffer `z` using the varint format. Returns the number of bytes actually written (1–9).

**Parameters:**

- `z` — a mutable byte buffer. Must be at least `maxVarIntLen` (9) bytes long to be safe for any input.
- `x` — the value to encode.

**Return value:** The count of bytes written. Pass this back to `readVu64` so the decoder knows where the next varint starts.

**Key insight:** Small numbers are cheap. The value `0` costs 1 byte; the value `2^64 − 1` costs 9 bytes. Everything in between scales smoothly.

**Example:**

```nim
import varints

var buf: array[maxVarIntLen, byte]

# Encode a small number — uses only 1 byte
var n = writeVu64(buf, 100u64)
echo n          # → 1
echo buf[0]     # → 100

# Encode a large number — uses more bytes
n = writeVu64(buf, 300_000u64)
echo n          # → 4
```

---

### `readVu64` — decode an unsigned 64-bit integer

```nim
proc readVu64*(z: openArray[byte]; pResult: var uint64): int
```

**What it does:** Reads a varint from the byte buffer `z`, stores the decoded value in `pResult`, and returns the number of bytes consumed. Returns `0` if the buffer is too short to hold a complete varint (truncated data).

**Parameters:**

- `z` — a byte buffer containing encoded data (read-only).
- `pResult` — output variable that receives the decoded `uint64` value.

**Return value:** The number of bytes consumed, matching what `writeVu64` returned when encoding. A return value of `0` signals that the buffer does not contain a complete varint and more data is needed.

**Key insight:** The first byte acts as a *discriminator* — its value alone tells the decoder exactly how many additional bytes to read. This makes decoding a single-pass, branch-only operation with no loops.

**Example:**

```nim
import varints

var buf: array[maxVarIntLen, byte]
var value: uint64

# Round-trip: encode then decode
let written = writeVu64(buf, 12345u64)
let read    = readVu64(buf, value)

assert written == read    # same number of bytes
assert value == 12345u64
echo value   # → 12345

# Detecting a truncated buffer
var shortBuf: array[1, byte]
shortBuf[0] = 252u8           # discriminator that requires 6 bytes total
let result = readVu64(shortBuf, value)
echo result  # → 0  (buffer too short, value unchanged)
```

---

### `encodeZigzag` — map signed integers to unsigned

```nim
proc encodeZigzag*(x: int64): uint64 {.inline.}
```

**What it does:** Converts a *signed* `int64` into an *unsigned* `uint64` using the zigzag encoding scheme, making it suitable for compression with `writeVu64`.

**Why this is needed:** Varint encoding is efficient only for *small* unsigned values. A negative signed integer like `-1` has the bit pattern `0xFFFFFFFFFFFFFFFF`, which would require the full 9 bytes. Zigzag encoding solves this by interleaving positive and negative numbers:

```
0  → 0
-1 → 1
1  → 2
-2 → 3
2  → 4
...
```

This way, numbers with small *absolute values* (both positive and negative) produce small unsigned results that compress well.

**Formula:** `(x << 1) XOR (x >> 63)`

**Example:**

```nim
import varints

echo encodeZigzag(0'i64)    # → 0
echo encodeZigzag(-1'i64)   # → 1
echo encodeZigzag(1'i64)    # → 2
echo encodeZigzag(-2'i64)   # → 3
echo encodeZigzag(100'i64)  # → 200
echo encodeZigzag(-100'i64) # → 199
```

**Typical usage pattern — encoding a signed integer as a compact varint:**

```nim
var buf: array[maxVarIntLen, byte]
let signed: int64 = -42
let n = writeVu64(buf, encodeZigzag(signed))
echo n  # → 1  (only 1 byte for a small magnitude value!)
```

---

### `decodeZigzag` — recover a signed integer from zigzag form

```nim
proc decodeZigzag*(x: uint64): int64 {.inline.}
```

**What it does:** The exact inverse of `encodeZigzag`. Takes a zigzag-encoded `uint64` and returns the original `int64`.

**Formula:** `(x >> 1) XOR -(x AND 1)` (implemented as a bit cast in the source).

**Example:**

```nim
import varints

echo decodeZigzag(0u64)   # → 0
echo decodeZigzag(1u64)   # → -1
echo decodeZigzag(2u64)   # → 1
echo decodeZigzag(3u64)   # → -2
echo decodeZigzag(199u64) # → -100
echo decodeZigzag(200u64) # → 100
```

**Full round-trip for signed integers:**

```nim
import varints

var buf: array[maxVarIntLen, byte]
var raw: uint64

let original: int64 = -9999

# Encode
let written = writeVu64(buf, encodeZigzag(original))

# Decode
let consumed = readVu64(buf, raw)
let recovered = decodeZigzag(raw)

assert recovered == original
echo recovered  # → -9999
echo written    # → 2  (small magnitude = small varint)
```

---

## Combining everything — a streaming example

```nim
import varints

# Suppose we want to write a stream of signed deltas between sensor readings.
# Deltas are often small, so varint + zigzag gives great compression.

let deltas: seq[int64] = @[0, 5, -3, 120, -1, 0, 1000, -800]

var stream: seq[byte] = @[]
var tmp: array[maxVarIntLen, byte]

for d in deltas:
  let n = writeVu64(tmp, encodeZigzag(d))
  stream.add(tmp[0 ..< n])

echo "Encoded size: ", stream.len, " bytes"  # Much smaller than 8×8 = 64 bytes

# --- Decoding ---
var pos = 0
var raw: uint64
while pos < stream.len:
  let n = readVu64(toOpenArray(stream, pos, stream.high), raw)
  if n == 0: break   # truncated
  echo decodeZigzag(raw)
  pos += n
```

---

## Quick cheat-sheet

| Symbol            | Kind      | Purpose                                      |
|-------------------|-----------|----------------------------------------------|
| `maxVarIntLen`    | constant  | Maximum bytes a varint can occupy (= 9)      |
| `writeVu64`       | proc      | Encode `uint64` → varint bytes               |
| `readVu64`        | proc      | Decode varint bytes → `uint64`               |
| `encodeZigzag`    | proc      | Map `int64` → `uint64` for signed compression|
| `decodeZigzag`    | proc      | Map `uint64` → `int64` (inverse of above)    |
