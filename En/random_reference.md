# std/random — Module Reference

> Nim's standard pseudo-random number generator (PRNG), based on the **xoroshiro128+** algorithm.
>
> ⚠️ **This module must NOT be used for cryptographic purposes.** For cryptographically secure randomness, use [`std/sysrand`](https://nim-lang.org/docs/sysrand.html).

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Types](#types)
3. [Initialization & Seeding](#initialization--seeding)
4. [The `rand` Family](#the-rand-family)
5. [Sampling](#sampling)
6. [Shuffling](#shuffling)
7. [Gaussian Distribution](#gaussian-distribution)
8. [Low-Level & Advanced](#low-level--advanced)
9. [Thread Safety](#thread-safety)
10. [Quick-Reference Summary](#quick-reference-summary)

---

## Core Concepts

### Default RNG vs. explicit `Rand` state

Every proc in this module comes in two flavors:

- **Default RNG flavor** — no `Rand` parameter. Uses a single module-level global state. Convenient for simple programs, but **not thread-safe** and always starts from the same seed unless `randomize()` is called.
- **Explicit state flavor** — takes a `var Rand` as the first parameter. Fully self-contained; multiple independent generators can coexist, each with their own state. Required for correct multi-threaded use.

### Reproducibility

If you call `randomize()` before using any RNG proc, the generator is seeded from the system's random source and will produce different results each run. If you skip `randomize()`, the generator always starts at the same fixed seed and produces the same sequence every time — useful for reproducible tests.

---

## Types

### `Rand`

```nim
type Rand* = object
  a0, a1: Ui  # internal state fields (private)
```

The full state of a random number generator instance. It is a plain value type (`object`, not `ref`), so it can be copied, stored in arrays, and passed between threads freely — but every thread that uses it must own its **own copy**.

Creating a `Rand`:

```nim
var r = initRand(42)        # from a fixed seed
var r = initRand()          # from system random source (time-seeded on JS)
```

---

### `randState` (template)

```nim
template randState*(): untyped
```

Exposes the internal module-level default `Rand` state so that other library modules can read or modify it. Useful for authors of modules that build on top of `std/random`. Ordinary application code should not need this.

---

## Initialization & Seeding

### `initRand(seed)` — seeded initialization

```nim
proc initRand*(seed: int64): Rand
```

Creates a brand-new `Rand` state seeded with a specific integer. The same seed always produces the same sequence of random numbers, which makes this ideal for reproducible simulations, tests, and procedural generation.

When `seed == 0`, an internal non-zero fallback is used automatically (because zero is a degenerate fixed point for the underlying algorithm).

```nim
var r = initRand(42)
echo r.rand(100)   # always the same value for seed 42
```

To get a different sequence each run, seed from the current time:

```nim
import std/times
let now = getTime()
var r = initRand(now.toUnix * 1_000_000_000 + now.nanosecond)
```

---

### `initRand()` — automatic initialization

```nim
proc initRand(): Rand
```

Creates a `Rand` state seeded automatically from the operating system's random source (`sysrand.urandom` on native platforms, `epochTime` on JS). The resulting state is independent of the default RNG. This is the preferred way to initialize a per-thread or per-object generator when you do not care about reproducibility.

> **Note:** Does not work in the compile-time VM (NimScript at compile time).

```nim
var r = initRand()   # unique each time the program runs
```

---

### `randomize(seed)` — seed the default RNG

```nim
proc randomize*(seed: int64)
```

Reinitializes the **default** (global) RNG with a specific integer seed. After this call, all default-flavor procs (`rand(100)`, `sample(arr)`, etc.) will produce a deterministic sequence corresponding to that seed.

```nim
randomize(123)
echo rand(100)   # same value every run with seed 123
```

---

### `randomize()` — seed default RNG from system

```nim
proc randomize()
```

Reinitializes the default RNG using the system's random source. Call this **once**, before any other use of default-flavor procs, to ensure different results each run. This is the single most important call in any program that needs non-deterministic randomness.

```nim
randomize()      # call once at program start
echo rand(100)   # different every run
```

> **Note:** Does not work in the compile-time VM.

---

## The `rand` Family

`rand` is heavily overloaded. The correct overload is selected automatically based on the argument type.

---

### `rand(max: int)` / `r.rand(max: Natural)` — random integer in `0..max`

```nim
proc rand*(max: int): int
proc rand*(r: var Rand; max: Natural): int
```

Returns a uniformly distributed integer in the **closed** range `0..max` (both endpoints inclusive). The upper bound is part of the output range.

```nim
let n = rand(9)        # one of 0, 1, 2, … 9
var r = initRand(1)
let m = r.rand(9)      # same range, explicit state
```

A common mistake is treating `rand(n)` as if it returns `0..<n`. It does not — it includes `n`. To pick an index into an array of length `n`, use `rand(n - 1)` or the slice form `rand(0..<n)`.

---

### `rand(max: float)` / `r.rand(max: range[0.0..high(float)])` — random float in `0.0..max`

```nim
proc rand*(max: float): float
proc rand*(r: var Rand; max: range[0.0..high(float)]): float
```

Returns a uniformly distributed floating-point number in `[0.0, max]`. The implementation uses bit manipulation of the raw `uint64` output of the underlying generator for a uniform distribution over the representable floats in that range, avoiding bias.

```nim
let f = rand(1.0)     # in [0.0, 1.0]
let p = rand(100.0)   # a random percentage
```

---

### `rand(x: HSlice)` / `r.rand(x: HSlice)` — random value in a slice

```nim
proc rand*[T: Ordinal or SomeFloat](x: HSlice[T, T]): T
proc rand*[T: Ordinal or SomeFloat](r: var Rand; x: HSlice[T, T]): T
```

The most flexible form of `rand`. Given a slice `a..b`, returns a value uniformly distributed in `[a, b]`. Works for integers, floats, and **enums without holes**.

```nim
let die    = rand(1..6)           # classic six-sided die
let temp   = rand(-20.0 .. 40.0)  # a float in a range
let chance = rand(0.0 .. 1.0)     # probability
```

For enum types, all variants in the range are equally probable:

```nim
type Direction = enum North, East, South, West
let dir = rand(North..West)
```

---

### `rand(t: typedesc[T])` / `r.rand(t: typedesc[T])` — full-range ordinal

```nim
proc rand*[T: Ordinal](t: typedesc[T]): T
proc rand*[T: Ordinal](r: var Rand; t: typedesc[T]): T
```

Returns a random value spanning the **entire** range of type `T` — from `low(T)` to `high(T)`. Works for any ordinal type: integers, booleans, enums, and range types.

```nim
let b  = rand(bool)           # true or false, 50/50
let c  = rand(char)           # any character in 0..255
let i8 = rand(int8)           # any int8 (-128..127)
let u  = rand(uint32)         # any uint32

type Color = enum Red, Green, Blue
let col = rand(Color)         # any Color value
```

This is the cleanest way to generate a uniformly random value of a type when you want every possible value equally likely.

---

## Sampling

### `sample(a: openArray[T])` / `r.sample(a)` — pick from a sequence

```nim
proc sample*[T](a: openArray[T]): lent T
proc sample*[T](r: var Rand; a: openArray[T]): T
```

Returns a uniformly random element from the array or sequence `a`. All elements have equal probability. The array must be non-empty (otherwise an index out-of-bounds error occurs at runtime).

```nim
let colors = ["red", "green", "blue"]
let pick = sample(colors)   # one of the three colors, equally likely

var r = initRand(1)
let pick2 = r.sample(colors)
```

---

### `sample(s: set[T])` / `r.sample(s)` — pick from a set

```nim
proc sample*[T](s: set[T]): T
proc sample*[T](r: var Rand; s: set[T]): T
```

Returns a uniformly random element from the Nim `set`. The set must be non-empty. Because sets have no guaranteed ordering, the element is selected by counting: a random index into the set's cardinality is computed, and the set is iterated to find the corresponding element.

```nim
let odd = {1, 3, 5, 7, 9}
echo sample(odd)   # one of the five odd numbers
```

---

### `sample(a, cdf)` / `r.sample(a, cdf)` — weighted sampling via CDF

```nim
proc sample*[T, U](a: openArray[T]; cdf: openArray[U]): T
proc sample*[T, U](r: var Rand; a: openArray[T]; cdf: openArray[U]): T
```

Returns a random element from `a` according to a **cumulative distribution function** (CDF) stored in `cdf`. This enables weighted random selection: elements with higher weights are more likely to be chosen.

`cdf` must have the same length as `a`, its values must be non-decreasing, and the last value must be positive. It does not need to be normalized to any particular maximum. The `math.cumsummed` proc is the natural way to build a CDF from an array of counts or weights.

```nim
import std/math

let prizes   = ["jackpot", "car", "bike", "pen", "nothing"]
let weights  = [1,         5,    20,     100,   500      ]
let cdf      = weights.cumsummed
# cdf == [1, 6, 26, 126, 626]
# "nothing" has 500/626 ≈ 80% probability

randomize()
echo sample(prizes, cdf)   # most likely "nothing"
```

The CDF approach is much more efficient for large arrays than repeated calls to `sample` with rejection, since each call is O(log n) via binary search.

---

## Shuffling

### `shuffle(x)` / `r.shuffle(x)` — in-place Fisher-Yates shuffle

```nim
proc shuffle*[T](x: var openArray[T])
proc shuffle*[T](r: var Rand; x: var openArray[T])
```

Rearranges the elements of `x` in a uniformly random permutation, in place. Uses the Fisher-Yates algorithm, which guarantees every permutation is equally probable.

```nim
var deck = ["A", "2", "3", "4", "5", "6", "7", "8", "9", "10", "J", "Q", "K"]
randomize()
shuffle(deck)   # deck is now in a random order

var r = initRand(42)
r.shuffle(deck)   # reproducible shuffle with explicit state
```

`shuffle` modifies the sequence in place and returns nothing. The original order is lost after the call.

---

## Gaussian Distribution

### `gauss(mu, sigma)` / `r.gauss(mu, sigma)` — normally distributed float

```nim
proc gauss*(mu = 0.0, sigma = 1.0): float
proc gauss*(r: var Rand; mu = 0.0; sigma = 1.0): float
```

Returns a floating-point random number drawn from a **Gaussian (normal) distribution** with mean `mu` and standard deviation `sigma`. The default parameters (`mu = 0.0`, `sigma = 1.0`) produce the standard normal distribution.

The implementation uses the ratio-of-uniforms method, which is both efficient and numerically well-behaved.

```nim
# Heights of adults: mean 170 cm, std dev 10 cm
randomize()
let height = gauss(mu = 170.0, sigma = 10.0)

# Noise in a simulation
var r = initRand(7)
let noise = r.gauss(0.0, 0.01)   # small Gaussian perturbation
```

Unlike `rand`, which produces a uniform distribution, `gauss` produces a bell-curve distribution. About 68% of values fall within one `sigma` of `mu`, and about 95% within two `sigma`.

---

## Low-Level & Advanced

### `next(r)` — raw 64-bit output

```nim
proc next*(r: var Rand): uint64
```

Advances the RNG state by one step and returns the raw, unprocessed `uint64` output of the xoroshiro128+ algorithm. This is the single primitive from which all higher-level procs are built.

You would use `next` directly only when building a custom distribution or when you need the maximum throughput from the generator without any scaling or transformation overhead.

```nim
var r = initRand(2019)
let raw = r.next()   # a raw 64-bit pseudo-random value
```

Note that consecutive calls to `next` produce values that are strongly uniform across all 64 bits, but individual bits or small subsets may have weak correlations typical of xoroshiro-family generators.

---

### `skipRandomNumbers(s)` — jump ahead 2⁶⁴ steps

```nim
proc skipRandomNumbers*(s: var Rand)
```

Advances the `Rand` state by **2⁶⁴** steps in one O(1) operation, as if `next` had been called that many times. This makes it possible to partition the generator's output into non-overlapping subsequences for use in parallel computations.

The canonical multi-threaded pattern: create one `Rand`, assign it to thread 0, call `skipRandomNumbers`, assign to thread 1, call `skipRandomNumbers`, assign to thread 2, and so on. Each thread then has a distinct subsequence of 2⁶⁴ values that will never overlap with any other thread's subsequence — as long as no single thread generates more than 2⁶⁴ numbers.

```nim
var r = initRand(2024)

var r0 = r              # thread 0 gets this state
r.skipRandomNumbers()
var r1 = r              # thread 1 gets this state
r.skipRandomNumbers()
var r2 = r              # thread 2 gets this state
# r0, r1, r2 will never produce overlapping sequences
```

---

## Thread Safety

The default-flavor procs (those without a `Rand` parameter) are **not thread-safe** because they all read and write the same global `state` variable without any synchronization.

To generate random numbers correctly in a multi-threaded program:

1. Create a dedicated `Rand` state for each thread using `initRand()` or `initRand(seed)`.
2. Use the explicit-state overloads (`r.rand(...)`, `r.sample(...)`, etc.) exclusively.
3. If the sequences must be guaranteed non-overlapping, use `skipRandomNumbers` to distribute the state space across threads.

```nim
# Correct multi-threaded pattern
proc workerThread(seed: int64) {.thread.} =
  var r = initRand(seed)          # each thread owns its state
  for _ in 1..1000:
    echo r.rand(0..100)           # safe — no shared state

# Or use skipRandomNumbers for non-overlapping subsequences:
var r = initRand()
for i in 0..<numThreads:
  let threadState = r             # copy current state to thread i
  r.skipRandomNumbers()           # advance global state for next thread
  createThread(threads[i], worker, threadState)
```

---

## Quick-Reference Summary

| Proc / Template | What it does |
|---|---|
| `initRand(seed)` | Create a new `Rand` from a fixed seed |
| `initRand()` | Create a new `Rand` from system random source |
| `randomize(seed)` | Seed the default RNG with a fixed value |
| `randomize()` | Seed the default RNG from system random source |
| `rand(max: int)` | Random int in `0..max` (default RNG) |
| `r.rand(max: Natural)` | Random int in `0..max` (explicit state) |
| `rand(max: float)` | Random float in `0.0..max` (default RNG) |
| `r.rand(max: float)` | Random float in `0.0..max` (explicit state) |
| `rand(a..b)` | Random value in slice `a..b` (default RNG) |
| `r.rand(a..b)` | Random value in slice `a..b` (explicit state) |
| `rand(T)` | Random value over full range of type `T` (default RNG) |
| `r.rand(T)` | Random value over full range of type `T` (explicit state) |
| `sample(arr)` | Random element from array/seq (default RNG) |
| `r.sample(arr)` | Random element from array/seq (explicit state) |
| `sample(s: set)` | Random element from set (default RNG) |
| `r.sample(s: set)` | Random element from set (explicit state) |
| `sample(arr, cdf)` | Weighted random element via CDF (default RNG) |
| `r.sample(arr, cdf)` | Weighted random element via CDF (explicit state) |
| `shuffle(arr)` | Fisher-Yates shuffle in place (default RNG) |
| `r.shuffle(arr)` | Fisher-Yates shuffle in place (explicit state) |
| `gauss(mu, sigma)` | Gaussian-distributed float (default RNG) |
| `r.gauss(mu, sigma)` | Gaussian-distributed float (explicit state) |
| `r.next()` | Raw `uint64` output from the generator |
| `r.skipRandomNumbers()` | Jump ahead 2⁶⁴ steps (for parallel use) |
| `randState()` | Access the module-level default `Rand` state |
