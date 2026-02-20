# Nim `std/bitops` Module — Complete Reference

> Low-level bit manipulation utilities for Nim integers.  
> Compiler intrinsics are used automatically on GCC, LLVM/Clang, VCC, and ICC for maximum performance.  
> The module is also compatible with JavaScript, NimScript, and the compile-time VM.

---

## Table of Contents

1. [Types](#types)
2. [Bitwise Logic Operators](#bitwise-logic-operators)
3. [Bit Slicing](#bit-slicing)
4. [Mask Creation](#mask-creation)
5. [Masking — Read-Only (functional)](#masking--read-only-functional)
6. [Masking — In-Place (mutating)](#masking--in-place-mutating)
7. [Single-Bit Operations](#single-bit-operations)
8. [Multi-Bit Operations](#multi-bit-operations)
9. [Bit Testing](#bit-testing)
10. [Counting & Analysis](#counting--analysis)
11. [Bit Rotation](#bit-rotation)
12. [Bit Reversal](#bit-reversal)
13. [Compiler Flags](#compiler-flags)

---

## Types

### `BitsRange[T]`

```nim
type BitsRange*[T] = range[0..sizeof(T)*8-1]
```

A range type representing all valid bit positions for integer type `T`.  
For `uint8` this is `0..7`; for `uint32` it is `0..31`.

---

## Bitwise Logic Operators

### `bitnot`

```nim
func bitnot*[T: SomeInteger](x: T): T
```

Returns the **bitwise complement** (NOT) of `x`. Every 0 becomes 1 and vice-versa.

```nim
import std/bitops

doAssert bitnot(0b0000_1111'u8) == 0b1111_0000'u8
doAssert bitnot(0'u8) == 255'u8
```

---

### `bitand`

```nim
macro bitand*[T: SomeInteger](x, y: T; z: varargs[T]): T
```

Computes the **bitwise AND** of all arguments. Accepts two or more integers of the same type.  
A result bit is 1 only when **all** corresponding source bits are 1.

```nim
doAssert bitand(0b1100'u8, 0b1010'u8) == 0b1000'u8
doAssert bitand(0b1111'u8, 0b1010'u8, 0b1001'u8) == 0b1000'u8
```

---

### `bitor`

```nim
macro bitor*[T: SomeInteger](x, y: T; z: varargs[T]): T
```

Computes the **bitwise OR** of all arguments. A result bit is 1 when **at least one** source bit is 1.

```nim
doAssert bitor(0b1100'u8, 0b0011'u8) == 0b1111'u8
doAssert bitor(0b0001'u8, 0b0010'u8, 0b0100'u8) == 0b0111'u8
```

---

### `bitxor`

```nim
macro bitxor*[T: SomeInteger](x, y: T; z: varargs[T]): T
```

Computes the **bitwise XOR** of all arguments. A result bit is 1 when the **number of 1-bits across sources is odd**.

```nim
doAssert bitxor(0b1100'u8, 0b1010'u8) == 0b0110'u8
doAssert bitxor(0b0001'u8, 0b0011'u8, 0b0111'u8) == 0b0101'u8
```

---

## Bit Slicing

### `bitsliced`

```nim
func bitsliced*[T: SomeInteger](v: T; slice: Slice[int]): T
```

Returns an **extracted and right-shifted** slice of bits from `v`. The bit range is inclusive on both ends and may use `..` or `..<`.

```nim
doAssert 0b10111.bitsliced(2 .. 4) == 0b101   # bits 2,3,4 → shift right by 2
doAssert 0b11100.bitsliced(0 .. 2) == 0b100
doAssert 0b11100.bitsliced(0 ..< 3) == 0b100  # same as 0..2
```

> **Tip:** The result is shifted so that `slice.a` becomes bit 0 of the output.

---

### `bitslice`

```nim
proc bitslice*[T: SomeInteger](v: var T; slice: Slice[int])
```

**Mutating** version of `bitsliced`. Modifies `v` in place.

```nim
var x = 0b101110
x.bitslice(2 .. 4)
doAssert x == 0b011
```

---

## Mask Creation

### `toMask`

```nim
func toMask*[T: SomeInteger](slice: Slice[int]): T
```

Creates a bitmask with 1-bits exactly in the positions described by `slice`.

```nim
doAssert toMask[int32](1 .. 3) == 0b1110'i32   # bits 1, 2, 3 set
doAssert toMask[int32](0 .. 3) == 0b1111'i32   # bits 0, 1, 2, 3 set
```

---

## Masking — Read-Only (functional)

These functions **return a new value** without modifying the original.

---

### `masked(v, mask)`

```nim
proc masked*[T: SomeInteger](v, mask: T): T
```

Returns `v` AND `mask` — keeps only the bits in `v` that are also 1 in `mask`.

```nim
let v = 0b0000_0011'u8
doAssert v.masked(0b0000_1010'u8) == 0b0000_0010'u8
```

---

### `masked(v, slice)`

```nim
func masked*[T: SomeInteger](v: T; slice: Slice[int]): T
```

Returns `v` ANDed with `toMask[T](slice)` — keeps only the bits within the given range.

```nim
let v = 0b0000_1011'u8
doAssert v.masked(1 .. 3) == 0b0000_1010'u8
```

---

### `setMasked(v, mask)`

```nim
func setMasked*[T: SomeInteger](v, mask: T): T
```

Returns `v` OR `mask` — sets all bits that are 1 in `mask`.

```nim
let v = 0b0000_0011'u8
doAssert v.setMasked(0b0000_1010'u8) == 0b0000_1011'u8
```

---

### `setMasked(v, slice)`

```nim
func setMasked*[T: SomeInteger](v: T; slice: Slice[int]): T
```

Returns `v` with all bits in the range set to 1.

```nim
let v = 0b0000_0011'u8
doAssert v.setMasked(2 .. 3) == 0b0000_1111'u8
```

---

### `clearMasked(v, mask)`

```nim
func clearMasked*[T: SomeInteger](v, mask: T): T
```

Returns `v` with all bits that are 1 in `mask` **cleared to 0**.  
Equivalent to `v AND NOT mask`.

```nim
let v = 0b0000_0011'u8
doAssert v.clearMasked(0b0000_1010'u8) == 0b0000_0001'u8
```

---

### `clearMasked(v, slice)`

```nim
func clearMasked*[T: SomeInteger](v: T; slice: Slice[int]): T
```

Returns `v` with all bits in the given range **cleared to 0**.

```nim
let v = 0b0000_0011'u8
doAssert v.clearMasked(1 .. 3) == 0b0000_0001'u8
```

---

### `flipMasked(v, mask)`

```nim
func flipMasked*[T: SomeInteger](v, mask: T): T
```

Returns `v` XOR `mask` — **toggles** all bits that are 1 in `mask`.

```nim
let v = 0b0000_0011'u8
doAssert v.flipMasked(0b0000_1010'u8) == 0b0000_1001'u8
```

---

### `flipMasked(v, slice)`

```nim
func flipMasked*[T: SomeInteger](v: T; slice: Slice[int]): T
```

Returns `v` with all bits in the given range **toggled**.

```nim
let v = 0b0000_0011'u8
doAssert v.flipMasked(1 .. 3) == 0b0000_1101'u8
```

---

## Masking — In-Place (mutating)

These procedures modify their first argument directly (`var T`).

---

### `mask(v, mask)`

```nim
proc mask*[T: SomeInteger](v: var T; mask: T)
```

Mutates `v` to `v AND mask`.

```nim
var v = 0b0000_0011'u8
v.mask(0b0000_1010'u8)
doAssert v == 0b0000_0010'u8
```

---

### `mask(v, slice)`

```nim
proc mask*[T: SomeInteger](v: var T; slice: Slice[int])
```

Mutates `v`, keeping only bits within the slice range.

```nim
var v = 0b0000_1011'u8
v.mask(1 .. 3)
doAssert v == 0b0000_1010'u8
```

---

### `setMask(v, mask)`

```nim
proc setMask*[T: SomeInteger](v: var T; mask: T)
```

Mutates `v` to `v OR mask` — sets all bits that are 1 in `mask`.

```nim
var v = 0b0000_0011'u8
v.setMask(0b0000_1010'u8)
doAssert v == 0b0000_1011'u8
```

---

### `setMask(v, slice)`

```nim
proc setMask*[T: SomeInteger](v: var T; slice: Slice[int])
```

Mutates `v`, setting all bits in the given range to 1.

```nim
var v = 0b0000_0011'u8
v.setMask(2 .. 3)
doAssert v == 0b0000_1111'u8
```

---

### `clearMask(v, mask)`

```nim
proc clearMask*[T: SomeInteger](v: var T; mask: T)
```

Mutates `v` to `v AND NOT mask` — clears all bits that are 1 in `mask`.

```nim
var v = 0b0000_0011'u8
v.clearMask(0b0000_1010'u8)
doAssert v == 0b0000_0001'u8
```

---

### `clearMask(v, slice)`

```nim
proc clearMask*[T: SomeInteger](v: var T; slice: Slice[int])
```

Mutates `v`, clearing all bits in the given range to 0.

```nim
var v = 0b0000_1011'u8
v.clearMask(1 .. 3)
doAssert v == 0b0000_0001'u8
```

---

### `flipMask(v, mask)`

```nim
proc flipMask*[T: SomeInteger](v: var T; mask: T)
```

Mutates `v` to `v XOR mask` — toggles all bits that are 1 in `mask`.

```nim
var v = 0b0000_0011'u8
v.flipMask(0b0000_1010'u8)
doAssert v == 0b0000_1001'u8
```

---

### `flipMask(v, slice)`

```nim
proc flipMask*[T: SomeInteger](v: var T; slice: Slice[int])
```

Mutates `v`, toggling all bits in the given range.

```nim
var v = 0b0000_0011'u8
v.flipMask(1 .. 3)
doAssert v == 0b0000_1101'u8
```

---

## Single-Bit Operations

### `setBit`

```nim
proc setBit*[T: SomeInteger](v: var T; bit: BitsRange[T])
```

Sets the bit at position `bit` to **1**. Bit 0 is the least significant bit.

```nim
var v = 0b0000_0011'u8
v.setBit(5'u8)
doAssert v == 0b0010_0011'u8
```

---

### `clearBit`

```nim
proc clearBit*[T: SomeInteger](v: var T; bit: BitsRange[T])
```

Sets the bit at position `bit` to **0**.

```nim
var v = 0b0000_0011'u8
v.clearBit(1'u8)
doAssert v == 0b0000_0001'u8
```

---

### `flipBit`

```nim
proc flipBit*[T: SomeInteger](v: var T; bit: BitsRange[T])
```

**Toggles** the bit at position `bit`.

```nim
var v = 0b0000_0011'u8
v.flipBit(2'u8)
doAssert v == 0b0000_0111'u8
```

---

## Multi-Bit Operations

### `setBits`

```nim
macro setBits*(v: typed; bits: varargs[typed]): untyped
```

Sets **multiple bits** to 1 in a single call.

```nim
var v = 0b0000_0011'u8
v.setBits(3, 5, 7)
doAssert v == 0b1010_1011'u8
```

---

### `clearBits`

```nim
macro clearBits*(v: typed; bits: varargs[typed]): untyped
```

Clears **multiple bits** to 0 in a single call.

```nim
var v = 0b1111_1111'u8
v.clearBits(1, 3, 5, 7)
doAssert v == 0b0101_0101'u8
```

---

### `flipBits`

```nim
macro flipBits*(v: typed; bits: varargs[typed]): untyped
```

**Toggles multiple bits** in a single call.

```nim
var v = 0b0000_1111'u8
v.flipBits(1, 3, 5, 7)
doAssert v == 0b1010_0101'u8
```

---

## Bit Testing

### `testBit`

```nim
proc testBit*[T: SomeInteger](v: T; bit: BitsRange[T]): bool
```

Returns `true` if the bit at position `bit` is **1**.

```nim
let v = 0b0000_1111'u8
doAssert v.testBit(0)         # bit 0 is set
doAssert not v.testBit(7)     # bit 7 is clear
```

---

## Counting & Analysis

### `countSetBits`

```nim
func countSetBits*(x: SomeInteger): int
```

Counts the number of **1-bits** in `x` (Hamming weight / popcount).  
Uses compiler intrinsics (`__builtin_popcount`) when available.

```nim
doAssert countSetBits(0b0000_0011'u8) == 2
doAssert countSetBits(0b1010_1010'u8) == 4
```

---

### `popcount`

```nim
func popcount*(x: SomeInteger): int
```

Alias for `countSetBits`. Provided for familiarity with C-style naming.

```nim
doAssert popcount(0b1111_0000'u8) == 4
```

---

### `parityBits`

```nim
func parityBits*(x: SomeInteger): int
```

Returns **1** if the number of set bits is **odd**, **0** if even.  
Useful for error-detection schemes.

```nim
doAssert parityBits(0b0000_0000'u8) == 0  # 0 ones  → even
doAssert parityBits(0b0101_0001'u8) == 1  # 3 ones  → odd
doAssert parityBits(0b0110_1001'u8) == 0  # 4 ones  → even
doAssert parityBits(0b0111_1111'u8) == 1  # 7 ones  → odd
```

---

### `firstSetBit`

```nim
func firstSetBit*(x: SomeInteger): int
```

Returns the **1-based** index of the **least significant** set bit.  
Returns 0 for `x = 0` when `noUndefinedBitOpts` is active; otherwise the result is undefined for zero.

```nim
doAssert firstSetBit(0b0000_0001'u8) == 1
doAssert firstSetBit(0b0000_0010'u8) == 2
doAssert firstSetBit(0b0000_1111'u8) == 1  # LSB is set
doAssert firstSetBit(0b0000_1000'u8) == 4
```

---

### `fastLog2`

```nim
func fastLog2*(x: SomeInteger): int
```

Fast integer **floor(log₂(x))**. Equivalent to the position of the most significant set bit (0-based).  
Returns -1 for `x = 0` when `noUndefinedBitOpts` is set; otherwise undefined.

```nim
doAssert fastLog2(0b0000_0001'u8) == 0  # 2^0 = 1
doAssert fastLog2(0b0000_0010'u8) == 1  # 2^1 = 2
doAssert fastLog2(0b0000_1111'u8) == 3  # highest bit at position 3
```

> **Common use:** `fastLog2(x) + 1` gives the number of bits required to represent `x`.

---

### `countLeadingZeroBits`

```nim
func countLeadingZeroBits*(x: SomeInteger): int
```

Returns the number of **leading zero bits** (from the most significant bit downwards).  
For `x = 0` the result is 0 when `noUndefinedBitOpts` is set; otherwise undefined.

```nim
doAssert countLeadingZeroBits(0b0000_0001'u8) == 7
doAssert countLeadingZeroBits(0b0000_1111'u8) == 4
doAssert countLeadingZeroBits(0b1000_0000'u8) == 0
```

---

### `countTrailingZeroBits`

```nim
func countTrailingZeroBits*(x: SomeInteger): int
```

Returns the number of **trailing zero bits** (from the least significant bit upwards).  
For `x = 0` the result is 0 when `noUndefinedBitOpts` is set; otherwise undefined.

```nim
doAssert countTrailingZeroBits(0b0000_0001'u8) == 0
doAssert countTrailingZeroBits(0b0000_0010'u8) == 1
doAssert countTrailingZeroBits(0b0000_1000'u8) == 3
doAssert countTrailingZeroBits(0b0000_1111'u8) == 0
```

> **Relation:** `countTrailingZeroBits(x) == firstSetBit(x) - 1` for non-zero `x`.

---

## Bit Rotation

Unlike shifts, rotation wraps bits that fall off one end back to the other end.

### `rotateLeftBits`

```nim
func rotateLeftBits*[T: SomeUnsignedInt](value: T, shift: range[0..(sizeof(T)*8)]): T
```

Rotates bits of `value` **left** by `shift` positions.

```nim
doAssert rotateLeftBits(0b0110_1001'u8, 4) == 0b1001_0110'u8
doAssert rotateLeftBits(0b00111100_11000011'u16, 8) == 0b11000011_00111100'u16
```

---

### `rotateRightBits`

```nim
func rotateRightBits*[T: SomeUnsignedInt](value: T, shift: range[0..(sizeof(T)*8)]): T
```

Rotates bits of `value` **right** by `shift` positions.

```nim
doAssert rotateRightBits(0b0110_1001'u8, 4) == 0b1001_0110'u8
doAssert rotateRightBits(0b00111100_11000011'u16, 8) == 0b11000011_00111100'u16
```

> **Note:** Both `rotateLeftBits` and `rotateRightBits` use hardware intrinsics (e.g., `ROL`/`ROR` instructions on x86) when compiled with GCC, Clang, VCC, or ICC.

---

## Bit Reversal

### `reverseBits`

```nim
func reverseBits*[T: SomeUnsignedInt](x: T): T
```

Returns `x` with all bits in **reversed order** — the MSB becomes the LSB and vice-versa.

```nim
doAssert reverseBits(0b10100100'u8) == 0b00100101'u8
doAssert reverseBits(0xdd'u8)       == 0xbb'u8
doAssert reverseBits(0xddbb'u16)    == 0xddbb'u16   # palindrome in bits
doAssert reverseBits(0xdeadbeef'u32) == 0xf77db57b'u32
```

---

## Compiler Flags

| Flag | Effect |
|---|---|
| `-d:noIntrinsicsBitOpts` | Disables compiler intrinsics; uses pure Nim fallback implementations. |
| `-d:noUndefinedBitOpts` | Forces defined output (usually 0 or -1) for edge cases like `x = 0` in `firstSetBit`, `fastLog2`, `countLeadingZeroBits`, `countTrailingZeroBits`. Small performance cost. |

---

## Quick Reference Table

| Function | Category | Returns new? | Mutates? |
|---|---|:---:|:---:|
| `bitnot` | Logic | ✓ | — |
| `bitand` | Logic | ✓ | — |
| `bitor` | Logic | ✓ | — |
| `bitxor` | Logic | ✓ | — |
| `bitsliced` | Slicing | ✓ | — |
| `bitslice` | Slicing | — | ✓ |
| `toMask` | Mask creation | ✓ | — |
| `masked` | Masking | ✓ | — |
| `mask` | Masking | — | ✓ |
| `setMasked` | Masking | ✓ | — |
| `setMask` | Masking | — | ✓ |
| `clearMasked` | Masking | ✓ | — |
| `clearMask` | Masking | — | ✓ |
| `flipMasked` | Masking | ✓ | — |
| `flipMask` | Masking | — | ✓ |
| `setBit` | Single bit | — | ✓ |
| `clearBit` | Single bit | — | ✓ |
| `flipBit` | Single bit | — | ✓ |
| `setBits` | Multi-bit | — | ✓ |
| `clearBits` | Multi-bit | — | ✓ |
| `flipBits` | Multi-bit | — | ✓ |
| `testBit` | Testing | ✓ (`bool`) | — |
| `countSetBits` | Analysis | ✓ | — |
| `popcount` | Analysis | ✓ | — |
| `parityBits` | Analysis | ✓ | — |
| `firstSetBit` | Analysis | ✓ | — |
| `fastLog2` | Analysis | ✓ | — |
| `countLeadingZeroBits` | Analysis | ✓ | — |
| `countTrailingZeroBits` | Analysis | ✓ | — |
| `rotateLeftBits` | Rotation | ✓ | — |
| `rotateRightBits` | Rotation | ✓ | — |
| `reverseBits` | Reversal | ✓ | — |
