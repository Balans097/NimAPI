# Nim BigInt & BigInt Random — Module Reference

> **Modules:** `bigints` · `bigints/random`  
> **Install:** `nimble install bigints`

This reference covers every exported symbol from both modules. `bigints` implements arbitrary-precision integers; `bigints/random` adds random sampling over the `BigInt` type using Nim's standard `std/random` machinery.

Sections are ordered by theme: construction, comparisons, arithmetic, bit manipulation, iteration, number-theoretic functions, conversion, and finally random generation.

---

## Table of Contents

1. [Type: BigInt](#type-bigint)
2. [Construction — initBigInt](#construction--initbigint)
3. [Comparison operators](#comparison-operators)
4. [Arithmetic operators](#arithmetic-operators)
5. [Increment and decrement](#increment-and-decrement)
6. [Successor and predecessor](#successor-and-predecessor)
7. [Exponentiation and shifts](#exponentiation-and-shifts)
8. [Bitwise operations](#bitwise-operations)
9. [Division and modulo](#division-and-modulo)
10. [Iterators](#iterators)
11. [Number theory](#number-theory)
12. [Conversion](#conversion)
13. [String I/O](#string-io)
14. [Random generation — bigints/random](#random-generation--bigintsrandom)

---

## Type: BigInt

```nim
type BigInt* = object
```

An arbitrary-precision signed integer. Internally it stores its magnitude as a sequence of 32-bit "limbs" (digits in base 2³²) and a separate sign flag. There is no practical upper bound on the value it can hold — it grows automatically.

**Bitwise operations** on `BigInt` treat negative values as if they were stored in two's complement with an infinite sign-extension bit. This matches the behaviour that Python programmers expect and makes operations like `not`, `and`, `or`, and `xor` consistent with the mathematical two's-complement model.

---

## Construction — `initBigInt`

`initBigInt` is the single family of constructors for `BigInt`. It is heavily overloaded to accept every primitive numeric type as well as strings.

---

### `initBigInt` from integer scalars

```nim
func initBigInt*[T: int8|int16|int32](val: T): BigInt
func initBigInt*[T: uint8|uint16|uint32](val: T): BigInt
func initBigInt*(val: int64): BigInt
func initBigInt*(val: uint64): BigInt
func initBigInt*(val: int): BigInt    # template, platform-width int
func initBigInt*(val: uint): BigInt   # template, platform-width uint
func initBigInt*(val: BigInt): BigInt # copy / identity
```

Converts any standard Nim integer to a `BigInt`. The sign is preserved correctly for signed types. Passing another `BigInt` produces a copy.

```nim
let a = 42.initBigInt           # from int literal
let b = (-999_999_999).initBigInt
let c = 18446744073709551615'u64.initBigInt  # uint64 max
let d = initBigInt(a)           # explicit copy
```

---

### `initBigInt` from a raw limb sequence

```nim
func initBigInt*(vals: sink seq[uint32], isNegative = false): BigInt
```

Constructs a `BigInt` from a sequence of 32-bit limbs in **little-endian** order (index 0 is the least significant limb). The `isNegative` flag sets the sign. This is a low-level constructor mostly useful when interfacing with cryptographic or binary protocols that already work in terms of 32-bit words.

```nim
# 10 + 2 * 2^32  — same as the integer 10 + 2 shl 32
let a = @[10'u32, 2'u32].initBigInt
```

---

### `initBigInt` from a string

```nim
func initBigInt*(str: string, base: range[2..36] = 10): BigInt
```

Parses `str` as a `BigInt` in the given `base` (2 through 36). The default base is 10. Raises `ValueError` for malformed or empty input.

Accepted formats:
- An optional leading `+` or `-` sign.
- Digits valid for the chosen base (`0–9`, `a–z` / `A–Z`).
- Underscore `_` as a digit-group separator anywhere **except** at the start or end of the number.

```nim
let a = initBigInt("1234")            # decimal: 1234
let b = initBigInt("1234", base = 8) # octal:   668
let c = initBigInt("ff", base = 16)  # hex:     255
let d = initBigInt("-42")            # negative: -42
let e = initBigInt("1_000_000")      # underscores allowed: 1000000
```

---

## Comparison operators

All comparison operators return `bool` and work for any combination of signs and magnitudes. They are defined as `func` (pure functions with no side effects).

### `==`

```nim
func `==`*(a, b: BigInt): bool
```

Returns `true` if `a` and `b` represent the same integer.

```nim
assert 5.initBigInt == (3.initBigInt + 2.initBigInt)
assert 0.initBigInt == initBigInt("0")
```

---

### `<`

```nim
func `<`*(a, b: BigInt): bool
```

Returns `true` if `a` is strictly less than `b`.

```nim
assert 3.initBigInt < 5.initBigInt
assert (-10).initBigInt < 0.initBigInt
```

---

### `<=`

```nim
func `<=`*(a, b: BigInt): bool
```

Returns `true` if `a` is less than or equal to `b`.

```nim
assert 5.initBigInt <= (3.initBigInt + 2.initBigInt)  # equal
assert 3.initBigInt <= 5.initBigInt                    # strictly less
```

> **Note:** `!=`, `>`, and `>=` are derived automatically by Nim from the above three.

---

## Arithmetic operators

All arithmetic operators return a new `BigInt` — they do not modify their operands.

### Unary minus `-`

```nim
func `-`*(a: BigInt): BigInt
```

Negates `a`. Returns the additive inverse.

```nim
let a = 5.initBigInt
assert (-a) == (-5).initBigInt
assert (-(-a)) == a
```

---

### `abs`

```nim
func abs*(a: BigInt): BigInt
```

Returns the absolute value of `a`. Positive numbers and zero are returned unchanged; negative numbers have their sign flipped.

```nim
assert abs((-42).initBigInt) == 42.initBigInt
assert abs(42.initBigInt)    == 42.initBigInt
```

---

### `+` and `+=`

```nim
func `+`*(a, b: BigInt): BigInt
template `+=`*(a: var BigInt, b: BigInt)
```

Adds two `BigInt` values. `+=` is an in-place update. Sign handling is fully automatic.

```nim
let sum = 5.initBigInt + 10.initBigInt    # 15
var x = 100.initBigInt
x += 23.initBigInt                         # x is now 123
```

---

### `-` and `-=`

```nim
func `-`*(a, b: BigInt): BigInt
template `-=`*(a: var BigInt, b: BigInt)
```

Subtracts `b` from `a`. Handles all sign combinations correctly, including results that change sign.

```nim
assert 15.initBigInt - 10.initBigInt  == 5.initBigInt
assert (-15).initBigInt - 10.initBigInt == (-25).initBigInt
assert 15.initBigInt - (-10).initBigInt == 25.initBigInt
```

---

### `*` and `*=`

```nim
func `*`*(a, b: BigInt): BigInt
template `*=`*(a: var BigInt, b: BigInt)
```

Multiplies two `BigInt` values. Uses schoolbook (long) multiplication across 32-bit limbs. The sign of the result follows the standard sign-product rule.

```nim
assert 421.initBigInt * 200.initBigInt == 84200.initBigInt
assert (-3).initBigInt * 4.initBigInt  == (-12).initBigInt

# Compute a large factorial:
var result = 1.initBigInt
for i in 1..20:
  result *= i.initBigInt
# result == 2432902008176640000
```

---

## Increment and decrement

### `inc`

```nim
func inc*(a: var BigInt, b: int = 1)
```

Increases `a` in place by `b` (default 1). Equivalent to `a += b.initBigInt` but more efficient for small `int` values because it avoids allocating a temporary `BigInt`.

```nim
var a = 15.initBigInt
inc a          # a == 16
inc(a, 7)      # a == 23
inc(a, -5)     # a == 18 (negative increment = decrement)
```

---

### `dec`

```nim
func dec*(a: var BigInt, b: int = 1)
```

Decreases `a` in place by `b` (default 1).

```nim
var a = 15.initBigInt
dec a          # a == 14
dec(a, 5)      # a == 9
```

---

## Successor and predecessor

### `succ`

```nim
func succ*(a: BigInt, b: int = 1): BigInt
```

Returns the `b`-th successor of `a` — i.e., `a + b` — without modifying `a`. `b` can be negative, in which case it behaves like `pred`.

```nim
assert succ(10.initBigInt)    == 11.initBigInt
assert succ(10.initBigInt, 5) == 15.initBigInt
```

---

### `pred`

```nim
func pred*(a: BigInt, b: int = 1): BigInt
```

Returns the `b`-th predecessor of `a` — i.e., `a - b`.

```nim
assert pred(10.initBigInt)    == 9.initBigInt
assert pred(10.initBigInt, 4) == 6.initBigInt
```

---

## Exponentiation and shifts

### `pow`

```nim
func pow*(x: BigInt, y: Natural): BigInt
```

Raises `x` to the power `y`. Uses binary exponentiation (repeated squaring), which is O(log y) multiplications. The exponent `y` is a plain `Natural` (a non-negative `int`), not a `BigInt`.

```nim
assert pow(2.initBigInt, 10) == 1024.initBigInt
assert pow(3.initBigInt, 0)  == 1.initBigInt    # anything^0 = 1
echo pow(2.initBigInt, 100)  # 2^100, a 31-digit number
```

> For modular exponentiation, see `powmod`.

---

### `shl` — left shift

```nim
func `shl`*(x: BigInt, y: Natural): BigInt
```

Shifts `x` to the left by `y` bits, which is equivalent to multiplying by 2^y. The sign of `x` is preserved. Shifting zero always returns zero.

```nim
assert 24.initBigInt shl 1 == 48.initBigInt   # × 2
assert 24.initBigInt shl 2 == 96.initBigInt   # × 4
assert 1.initBigInt shl 64 == initBigInt("18446744073709551616")
```

---

### `shr` — right shift

```nim
func `shr`*(x: BigInt, y: Natural): BigInt
```

Shifts `x` to the right by `y` bits, **arithmetically**: for negative values the shift fills from the left with 1-bits (floor division by 2^y), which is the two's-complement convention.

```nim
assert 24.initBigInt shr 1 == 12.initBigInt
assert 24.initBigInt shr 2 == 6.initBigInt
assert (-5).initBigInt shr 1 == (-3).initBigInt  # floor(-5/2) = -3
```

---

## Bitwise operations

All bitwise operators treat negative `BigInt` values as if they were in two's complement representation with an infinite number of sign-extension bits. This matches Python's integer semantics and allows the identities `not x == -(x + 1)`, `x and -1 == x`, etc., to hold.

### `not`

```nim
func `not`*(a: BigInt): BigInt
```

Bitwise complement. Follows the two's-complement identity: `not x = -(x + 1)`.

```nim
assert not 0.initBigInt == (-1).initBigInt
assert not 5.initBigInt == (-6).initBigInt
assert not (-1).initBigInt == 0.initBigInt
```

---

### `and`

```nim
func `and`*(a, b: BigInt): BigInt
```

Bitwise AND. Works correctly for all sign combinations by internally converting to two's-complement form for the operation.

```nim
assert 0b1100.initBigInt and 0b1010.initBigInt == 0b1000.initBigInt
assert (-1).initBigInt and 42.initBigInt == 42.initBigInt  # -1 is all-ones
```

---

### `or`

```nim
func `or`*(a, b: BigInt): BigInt
```

Bitwise OR.

```nim
assert 0b1100.initBigInt or 0b1010.initBigInt == 0b1110.initBigInt
```

---

### `xor`

```nim
func `xor`*(a, b: BigInt): BigInt
```

Bitwise XOR.

```nim
assert 0b1100.initBigInt xor 0b1010.initBigInt == 0b0110.initBigInt
assert x xor x == 0.initBigInt  # any number XOR itself is zero
```

---

## Division and modulo

The division model used by `div` and `mod` is **floor division** (also called Python-style division). The quotient is rounded toward negative infinity, and the remainder always has the same sign as the divisor. This is different from Nim's built-in `div`/`mod` for integers, which truncates toward zero.

The following identities hold for all nonzero `b`:
- `a == (a div b) * b + (a mod b)`
- `sign(a mod b) == sign(b)`

### `div`

```nim
func `div`*(a, b: BigInt): BigInt
```

Floor-division of `a` by `b`. Raises `DivByZeroDefect` if `b` is zero.

```nim
let a = 17.initBigInt
let b = 5.initBigInt
assert a div b        == 3.initBigInt    #  17/5  → floor(3.4)  = 3
assert (-a) div b     == (-4).initBigInt # -17/5  → floor(-3.4) = -4
assert a div (-b)     == (-4).initBigInt #  17/-5 → floor(-3.4) = -4
assert (-a) div (-b)  == 3.initBigInt    # -17/-5 → floor(3.4)  = 3
```

---

### `mod`

```nim
func `mod`*(a, b: BigInt): BigInt
```

The remainder after floor-division. The sign of the result always matches the sign of `b`. Raises `DivByZeroDefect` if `b` is zero.

```nim
let a = 17.initBigInt
let b = 5.initBigInt
assert a mod b        == 2.initBigInt     # positive mod positive → positive
assert (-a) mod b     == 3.initBigInt     # negative mod positive → positive
assert a mod (-b)     == (-3).initBigInt  # positive mod negative → negative
assert (-a) mod (-b)  == (-2).initBigInt  # negative mod negative → negative
```

---

### `divmod`

```nim
func divmod*(a, b: BigInt): tuple[q, r: BigInt]
```

Computes both the floor-quotient and the remainder in a single pass — more efficient than calling `div` and `mod` separately when you need both. Raises `DivByZeroDefect` if `b` is zero.

```nim
let (q, r) = divmod(17.initBigInt, 5.initBigInt)
assert q == 3.initBigInt
assert r == 2.initBigInt
```

---

## Iterators

Iterators over `BigInt` ranges follow the same naming convention as Nim's built-in numeric iterators.

### `countup`

```nim
iterator countup*(a, b: BigInt, step: int32 = 1): BigInt
```

Yields each `BigInt` from `a` up to `b` (inclusive), stepping by `step`. The `step` is a plain `int32`.

```nim
for x in countup(1.initBigInt, 5.initBigInt):
  echo x   # 1, 2, 3, 4, 5

for x in countup(0.initBigInt, 10.initBigInt, 3):
  echo x   # 0, 3, 6, 9
```

---

### `countdown`

```nim
iterator countdown*(a, b: BigInt, step: int32 = 1): BigInt
```

Yields each `BigInt` from `a` down to `b` (inclusive), stepping down by `step`.

```nim
for x in countdown(5.initBigInt, 1.initBigInt):
  echo x   # 5, 4, 3, 2, 1
```

---

### `..` (inclusive range)

```nim
iterator `..`*(a, b: BigInt): BigInt
```

Counts from `a` to `b` inclusive with step 1. Enables natural range syntax.

```nim
for x in 1.initBigInt .. 5.initBigInt:
  echo x   # 1, 2, 3, 4, 5
```

---

### `..<` (exclusive range)

```nim
iterator `..<`*(a, b: BigInt): BigInt
```

Counts from `a` up to — but not including — `b`.

```nim
for x in 1.initBigInt ..< 5.initBigInt:
  echo x   # 1, 2, 3, 4  (5 is excluded)
```

---

## Number theory

### `fastLog2`

```nim
func fastLog2*(a: BigInt): int
```

Returns ⌊log₂|a|⌋ — the position of the most significant set bit in the absolute value of `a`. Two special cases:

- Returns `-1` if `a` is zero (undefined logarithm).
- Works on the magnitude only: `fastLog2(-8.initBigInt)` returns 3, the same as `fastLog2(8.initBigInt)`.

This is very fast — it only looks at the highest limb and uses a single CPU instruction. It is particularly useful to quickly estimate the bit-length of a `BigInt` without a full conversion.

```nim
assert fastLog2(0.initBigInt)    == -1
assert fastLog2(1.initBigInt)    == 0
assert fastLog2(8.initBigInt)    == 3   # 2^3 = 8
assert fastLog2(255.initBigInt)  == 7   # highest set bit is bit 7
assert fastLog2(256.initBigInt)  == 8   # 2^8 = 256
```

---

### `gcd`

```nim
func gcd*(a, b: BigInt): BigInt
```

Returns the greatest common divisor of `a` and `b`. The result is always non-negative, regardless of the signs of the inputs. Uses the binary GCD algorithm (Stein's algorithm), which avoids division and is efficient for multi-precision integers.

```nim
assert gcd(54.initBigInt, 24.initBigInt) == 6.initBigInt
assert gcd(0.initBigInt, 7.initBigInt)   == 7.initBigInt  # gcd(0, n) = n
assert gcd((-6).initBigInt, 9.initBigInt) == 3.initBigInt # sign ignored
```

---

### `invmod`

```nim
func invmod*(a, modulus: BigInt): BigInt
```

Computes the **modular inverse** of `a` modulo `modulus` — that is, the unique integer `x` in `[1, modulus−1]` such that `(a * x) mod modulus == 1`. Uses the extended Euclidean algorithm.

Raises:
- `DivByZeroDefect` if `modulus` is zero or `a` is zero.
- `ValueError` if `modulus` is negative or if no inverse exists (i.e., `gcd(a, modulus) != 1`).

The return value is always in the range `[1, modulus − 1]`.

```nim
# 3 * 5 ≡ 15 ≡ 1 (mod 7)
assert invmod(3.initBigInt, 7.initBigInt) == 5.initBigInt

# Modular inverse is used in RSA key generation and modular arithmetic
```

---

### `powmod`

```nim
func powmod*(base, exponent, modulus: BigInt): BigInt
```

Computes `(base ^ exponent) mod modulus` efficiently. The result is always in `[0, modulus − 1]`.

Uses fast binary exponentiation while keeping intermediate values reduced modulo `modulus`, so the computation never has to multiply together numbers larger than `modulus²`. This makes it practical for moduli of thousands of bits (RSA, Diffie-Hellman, etc.).

Special handling:
- If `exponent` is negative, `base` is first replaced with its modular inverse.
- If `modulus == 1`, the result is always 0.

Raises:
- `DivByZeroDefect` if `modulus` is zero.
- `ValueError` if `modulus` is negative.

```nim
# 2^3 mod 7 = 8 mod 7 = 1
assert powmod(2.initBigInt, 3.initBigInt, 7.initBigInt) == 1.initBigInt

# Fermat's little theorem: a^(p-1) ≡ 1 (mod p) for prime p
let p = 17.initBigInt
assert powmod(3.initBigInt, pred(p), p) == 1.initBigInt
```

---

## Conversion

### `toInt`

```nim
func toInt*[T: SomeInteger](x: BigInt): Option[T]
```

Attempts to convert `x` to the requested integer type `T`. Because a `BigInt` may hold values far outside the range of any fixed-width type, the result is wrapped in an `Option`:

- Returns `some(value)` if `x` fits in `T`.
- Returns `none(T)` if `x` is out of range or (for unsigned types) negative.

This is the safe alternative to a direct cast. It forces you to explicitly handle the out-of-range case rather than silently truncating.

```nim
import std/options

assert toInt[int8](44.initBigInt)  == some(44'i8)
assert toInt[int8](200.initBigInt) == none(int8)   # 200 > 127
assert toInt[uint8](200.initBigInt) == some(200'u8)
assert toInt[uint8]((-1).initBigInt) == none(uint8) # unsigned can't be negative
assert toInt[int](1_000_000.initBigInt) == some(1_000_000)
```

---

## String I/O

### `toString`

```nim
func toString*(a: BigInt, base: range[2..36] = 10): string
```

Converts `a` to a string in the given `base` (2 through 36). The default is base 10. Negative numbers are prefixed with `-`. No base prefix (`0x`, `0b`, etc.) is added — the caller is responsible for that if needed. Letters for digits above 9 are lowercase.

```nim
let a = 55.initBigInt
assert toString(a)       == "55"
assert toString(a, 2)    == "110111"    # binary
assert toString(a, 8)    == "67"        # octal
assert toString(a, 16)   == "37"        # hexadecimal
assert toString(a, 36)   == "1j"        # base 36

echo "0x" & toString(255.initBigInt, 16)  # "0xff" — add prefix manually
```

---

### `$`

```nim
func `$`*(a: BigInt): string
```

The default string conversion. Equivalent to `toString(a, 10)`. This is what `echo` and string interpolation call automatically.

```nim
let big = pow(2.initBigInt, 100)
echo big  # prints the decimal value of 2^100
echo $big  # same
```

---

## Random generation — `bigints/random`

> **Module:** `bigints/random`  
> **Import:** `import bigints/random`  
> **Depends on:** `std/random` (Nim's built-in Xoshiro-based PRNG)

This module extends Nim's `Rand` type with overloads of `rand` that produce uniformly distributed random `BigInt` values within a specified range.

**Uniformity:** The implementation generates random limbs bottom-up and uses a careful approach for the most-significant bits to ensure the result is uniformly distributed across the requested range, with a retry loop to eliminate bias.

---

### `rand` with a `Rand` state and a `Slice[BigInt]`

```nim
func rand*(r: var Rand, x: Slice[BigInt]): BigInt
```

Returns a uniformly distributed random `BigInt` in the closed interval `[x.a, x.b]`. Uses and updates the `Rand` state `r`. This is the stateful, reproducible form — by seeding `r` with a fixed value you get deterministic sequences.

Precondition: `x.a <= x.b`. Violating this raises an assertion error.

```nim
import std/random
import bigints/random

var rng = initRand(12345)                # seeded RNG
let lo = 1.initBigInt
let hi = initBigInt("1000000000000000000000")  # 10^21

let r = rng.rand(lo..hi)
assert r >= lo and r <= hi
```

---

### `rand` with a `Rand` state and a `max: BigInt`

```nim
func rand*(r: var Rand, max: BigInt): BigInt
```

Returns a uniformly distributed random `BigInt` in `[0, max]`. A convenience shorthand for `rand(r, 0.initBigInt..max)`.

```nim
var rng = initRand(42)
let modulus = initBigInt("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFFFFFFFFFFFF", base = 16)
let k = rng.rand(modulus)  # random scalar for elliptic-curve operations
```

---

### `rand` with a `Slice[BigInt]` (global state)

```nim
proc rand*(x: Slice[BigInt]): BigInt
```

The stateless convenience variant that uses a module-level `Rand` object (seeded with 777 at program start). Useful for quick scripts and non-reproducible sampling. For repeatable results, prefer the `func` overloads that take an explicit `Rand` argument.

```nim
import bigints/random

# Quick random BigInt — not reproducible across runs unless seeded elsewhere
let r = rand(0.initBigInt .. 1_000_000.initBigInt)
echo r
```

---

### `rand` with a `max: BigInt` (global state)

```nim
proc rand*(max: BigInt): BigInt
```

Returns a random `BigInt` in `[0, max]` using the global PRNG state. Equivalent to `rand(0.initBigInt..max)`.

```nim
import bigints/random

let lucky = rand(100.initBigInt)  # 0..100 inclusive
```

---

## Quick-reference table

| Task | Function / Operator |
|---|---|
| Create from integer | `42.initBigInt` |
| Create from string (decimal) | `initBigInt("123456789")` |
| Create from string (other base) | `initBigInt("ff", base=16)` |
| Create from raw limbs | `@[lo, hi].initBigInt` |
| Arithmetic | `+`, `-`, `*`, `div`, `mod`, `-` (unary) |
| Combined div+mod | `divmod(a, b)` |
| Compound assignment | `+=`, `-=`, `*=` |
| In-place increment/decrement | `inc(a)`, `dec(a)` |
| Immutable next/prev | `succ(a)`, `pred(a)` |
| Power | `pow(x, y)` (y: Natural) |
| Bit shifts | `shl`, `shr` |
| Bitwise | `not`, `and`, `or`, `xor` |
| Comparisons | `==`, `<`, `<=` (and derived `!=`, `>`, `>=`) |
| Absolute value | `abs(a)` |
| Bit length | `fastLog2(a)` |
| GCD | `gcd(a, b)` |
| Modular inverse | `invmod(a, modulus)` |
| Modular exponentiation | `powmod(base, exp, modulus)` |
| To fixed-width int (safe) | `toInt[T](x)` → `Option[T]` |
| To string | `$a`, `toString(a, base)` |
| Random in range | `rand(rng, lo..hi)` |
| Random 0..max | `rand(rng, max)` |
| Iterate inclusive | `for x in a..b` |
| Iterate exclusive | `for x in a..<b` |
| Iterate with step | `countup(a, b, step)`, `countdown(a, b, step)` |
