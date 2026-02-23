# `endians` Module Reference

> **Nim Standard Library — Byte-Order Conversion Helpers**
> API status: **Unstable**.

---

## Overview

Every multi-byte integer value stored in memory has a **byte order** — the sequence in which its constituent bytes are laid out. Two conventions exist:

- **Big-endian (BE):** the most significant byte comes first (lowest address). Used by most network protocols, SPARC, PowerPC (historically).
- **Little-endian (LE):** the least significant byte comes first. Used by x86/x86-64, ARM in default mode, RISC-V.

When you read binary data from a file, receive a network packet, or communicate with hardware, you frequently need to convert between these orders. The `endians` module provides nine low-level procedures for doing exactly that, covering 16-, 32-, and 64-bit values in three families:

| Family | What it does |
|--------|-------------|
| `swapEndianN` | Unconditionally reverses all N bytes. |
| `littleEndianN` | Ensures the output is in little-endian order (no-op on LE machines). |
| `bigEndianN` | Ensures the output is in big-endian order (no-op on BE machines). |

All procedures work on **raw memory** via `pointer` arguments — they are deliberately type-agnostic so they work equally well on integers, floats, or any plain-old-data type of the matching size.

### Compile-time optimisation

On GCC, Clang, ICC, and MSVC the swap operations are implemented using compiler intrinsics (`__builtin_bswap16/32/64`, `_bswap`, `_byteswap_*`), which typically compile down to a **single CPU instruction** (e.g. `BSWAP` on x86). On other compilers a portable byte-by-byte fallback is used automatically.

### How to use

```nim
import std/endians
```

---

## Conceptual background: what does "swap" mean?

Given the 32-bit hexadecimal value `0x01020304`, its bytes in memory are:

```
Address:  0x00  0x01  0x02  0x03
Big-end:  0x01  0x02  0x03  0x04   ← most significant byte first
Lil-end:  0x04  0x03  0x02  0x01   ← least significant byte first
```

Swapping the endianness means reversing the byte sequence in-place, so big-endian becomes little-endian and vice versa.

---

## Procedure Reference

All procedures share the same signature pattern:

```nim
proc nameN*(outp, inp: pointer)
```

- `inp` — pointer to the **source** buffer (read from).
- `outp` — pointer to the **destination** buffer (written to).
- Both pointers must reference valid memory of **at least N/8 bytes** (N = 16, 32 or 64).
- `inp` and `outp` **may alias** (point to the same memory) — the implementation uses an intermediate copy when needed.
- None of the procedures allocate heap memory.

---

### `swapEndian16`

```nim
proc swapEndian16*(outp, inp: pointer)
```

#### What it does

Reads 2 bytes from `inp` and writes them to `outp` in **reversed order** — byte 0 of the input becomes byte 1 of the output, and vice versa. This is an unconditional operation: it always swaps, regardless of the current CPU's byte order.

Use this when you *know* you need to reverse bytes — for example, when you have already determined that the source and destination endianness differ.

#### Memory layout

```
inp:  [A] [B]
outp: [B] [A]
```

#### Example

```nim
import std/endians

var src = [0x01'u8, 0x02]   # raw bytes: 0x01, 0x02
var dst: array[2, uint8]

swapEndian16(addr dst, addr src)
assert dst == [0x02'u8, 0x01]  # bytes are reversed

# Practical case: reading a big-endian int16 on a little-endian machine
var beBytes = [0x00'u8, 0xFF]  # big-endian representation of 255
var value: int16
swapEndian16(addr value, addr beBytes[0])
assert value == 255
```

---

### `swapEndian32`

```nim
proc swapEndian32*(outp, inp: pointer)
```

#### What it does

Reads 4 bytes from `inp` and writes them to `outp` with all four bytes in **reversed order**. Like `swapEndian16`, this is unconditional — it always reverses regardless of machine endianness.

#### Memory layout

```
inp:  [A] [B] [C] [D]
outp: [D] [C] [B] [A]
```

#### Example

```nim
import std/endians

var src = [0x01'u8, 0x02, 0x03, 0x04]
var dst: array[4, uint8]

swapEndian32(addr dst, addr src)
assert dst == [0x04'u8, 0x03, 0x02, 0x01]

# Practical case: convert a float32 received over a big-endian network
var netFloat: array[4, uint8] = [0x42'u8, 0xF6, 0xE9, 0x79]  # 123.456 in BE
var hostFloat: float32
swapEndian32(addr hostFloat, addr netFloat[0])
# hostFloat now holds 123.456 (on a little-endian host)
```

---

### `swapEndian64`

```nim
proc swapEndian64*(outp, inp: pointer)
```

#### What it does

Reads 8 bytes from `inp` and writes them to `outp` with all eight bytes in **reversed order**. The largest of the three swap operations — covers `int64`, `uint64`, `float64`, and any other 8-byte type.

#### Memory layout

```
inp:  [A] [B] [C] [D] [E] [F] [G] [H]
outp: [H] [G] [F] [E] [D] [C] [B] [A]
```

#### Example

```nim
import std/endians

var a = [1'u8, 2, 3, 4, 5, 6, 7, 8]
var b: array[8, uint8]

swapEndian64(addr b, addr a)
assert b == [8'u8, 7, 6, 5, 4, 3, 2, 1]

# In-place swap (outp == inp is safe):
swapEndian64(addr a, addr a)
assert a == [8'u8, 7, 6, 5, 4, 3, 2, 1]
```

---

### `littleEndian16`

```nim
proc littleEndian16*(outp, inp: pointer)
```

#### What it does

Copies 2 bytes from `inp` to `outp` and **ensures the result is in little-endian order**.

- On a **little-endian** machine (x86, ARM default): this is a plain `copyMem` — zero cost.
- On a **big-endian** machine: this calls `swapEndian16` to reverse the bytes.

This is the idiomatic procedure to use when you want to write a value to a format that mandates little-endian encoding (e.g. most Windows binary formats, WAV/BMP headers, USB descriptors).

#### Example

```nim
import std/endians

var value: uint16 = 0x0102   # native integer
var buf: array[2, uint8]

littleEndian16(addr buf, addr value)
# On little-endian host: buf == [0x02, 0x01]  (no swap needed, direct copy)
# On big-endian host:    buf == [0x02, 0x01]  (swap performed)
# Either way: buf always holds the LE representation.
```

---

### `littleEndian32`

```nim
proc littleEndian32*(outp, inp: pointer)
```

#### What it does

Copies 4 bytes from `inp` to `outp` ensuring the result is in **little-endian order** — swapping only when necessary (on big-endian hosts).

Typical use cases: writing 32-bit integers to file formats with specified LE encoding (ZIP, ELF, most image formats).

#### Example

```nim
import std/endians

var value: uint32 = 0xDEADBEEF
var buf: array[4, uint8]

littleEndian32(addr buf, addr value)
# buf now holds 0xDEADBEEF encoded as little-endian bytes:
# [0xEF, 0xBE, 0xAD, 0xDE]  — on any host
```

---

### `littleEndian64`

```nim
proc littleEndian64*(outp, inp: pointer)
```

#### What it does

Copies 8 bytes from `inp` to `outp` ensuring the result is in **little-endian order**. Covers 64-bit integers and `float64` values.

#### Example

```nim
import std/endians

var ts: int64 = 1_700_000_000_000'i64   # Unix timestamp in milliseconds
var buf: array[8, uint8]

littleEndian64(addr buf, addr ts)
# buf holds ts in LE byte order, ready to be written to a LE binary file
```

---

### `bigEndian16`

```nim
proc bigEndian16*(outp, inp: pointer)
```

#### What it does

Copies 2 bytes from `inp` to `outp` ensuring the result is in **big-endian order** — the byte order used by TCP/IP (network byte order), most audio formats (AIFF, MIDI), and processor architectures like SPARC and older PowerPC.

- On a **big-endian** machine: plain `copyMem`, no swap.
- On a **little-endian** machine: calls `swapEndian16`.

#### Example

```nim
import std/endians

# Preparing a 16-bit port number for a raw network packet header
var port: uint16 = 8080
var netBuf: array[2, uint8]

bigEndian16(addr netBuf, addr port)
# netBuf == [0x1F, 0x90]  (8080 = 0x1F90, big-endian)
```

---

### `bigEndian32`

```nim
proc bigEndian32*(outp, inp: pointer)
```

#### What it does

Copies 4 bytes from `inp` to `outp` in **big-endian order**. Essential for constructing or parsing network packets, serialising data for MIDI, writing PNG chunk lengths, and any other format that mandates network/big-endian byte order.

#### Example

```nim
import std/endians

# PNG stores chunk lengths as 32-bit big-endian unsigned integers
var length: uint32 = 1024
var pngBuf: array[4, uint8]

bigEndian32(addr pngBuf, addr length)
# pngBuf == [0x00, 0x00, 0x04, 0x00]  — correct PNG encoding of 1024
```

---

### `bigEndian64`

```nim
proc bigEndian64*(outp, inp: pointer)
```

#### What it does

Copies 8 bytes from `inp` to `outp` in **big-endian order**. The 64-bit counterpart of `bigEndian32` — useful for 64-bit timestamps, file offsets in formats like EXIF or MKV, and 64-bit network protocol fields.

#### Example

```nim
import std/endians

var fileSize: uint64 = 9_000_000_000'u64
var buf: array[8, uint8]

bigEndian64(addr buf, addr fileSize)
# buf holds fileSize in BE order — ready for a protocol header field
```

---

## Comparison table

| Procedure | Bytes | Behaviour on LE host | Behaviour on BE host |
|-----------|-------|----------------------|----------------------|
| `swapEndian16` | 2 | always swaps | always swaps |
| `swapEndian32` | 4 | always swaps | always swaps |
| `swapEndian64` | 8 | always swaps | always swaps |
| `littleEndian16` | 2 | `copyMem` (no-op) | swaps |
| `littleEndian32` | 4 | `copyMem` (no-op) | swaps |
| `littleEndian64` | 8 | `copyMem` (no-op) | swaps |
| `bigEndian16` | 2 | swaps | `copyMem` (no-op) |
| `bigEndian32` | 4 | swaps | `copyMem` (no-op) |
| `bigEndian64` | 8 | swaps | `copyMem` (no-op) |

---

## Common patterns and pitfalls

### Reading a struct from a binary file

```nim
import std/endians

type Header = object
  magic: uint32
  length: uint32

proc readHeader(data: openArray[byte]): Header =
  var raw: Header
  copyMem(addr raw, unsafeAddr data[0], sizeof(Header))
  # File is big-endian (e.g. network protocol):
  bigEndian32(addr result.magic, addr raw.magic)
  bigEndian32(addr result.length, addr raw.length)
```

### Pitfall: buffer too small

Every procedure assumes the buffers hold **at least** as many bytes as the suffix number implies (2, 4, or 8). Passing a shorter buffer causes **undefined behaviour** — there is no bounds checking.

```nim
var tiny: array[2, uint8]
# WRONG — buffer is only 2 bytes, but swapEndian32 needs 4:
# swapEndian32(addr result, addr tiny[0])  # UB!
```

### Pitfall: using `swapEndian` when you should use `bigEndian`/`littleEndian`

`swapEndianN` is correct *only* when you are sure the source and destination endianness differ. If you write code that uses `swapEndianN` unconditionally and then run it on a big-endian machine, the conversion will be wrong. Prefer `bigEndianN`/`littleEndianN` — they are portable by design.

---

## Quick reference

```
Need to...                                 Use...
─────────────────────────────────────────────────────────────
Reverse bytes, always                      swapEndianN
Write value for LE format (WAV, ELF...)    littleEndianN
Write value for BE format (PNG, TCP/IP...) bigEndianN
Read value from LE format                  littleEndianN  (src → native)
Read value from BE format                  bigEndianN     (src → native)
```
