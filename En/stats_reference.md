# stats — Module Reference

> **Nim standard library** | `import std/stats`

## What Does This Module Do?

This module provides **single-pass online statistical analysis**. "Single-pass" means every value is processed exactly once as it arrives, and all statistics are updated incrementally. There is no need to store the entire dataset in memory — you can push millions of values one by one and query any statistic at any moment without re-scanning the data.

The module offers two accumulator types:

- **`RunningStat`** — collects statistics for a single numeric series: count, min, max, sum, mean, variance, standard deviation, skewness, and kurtosis.
- **`RunningRegress`** — collects two paired series simultaneously and computes linear regression statistics: slope, intercept, and Pearson correlation.

Both types implement the numerically stable Knuth/Welford one-pass algorithm, which avoids the catastrophic cancellation that naive implementations suffer from with large or nearly-equal values.

For one-off calculations on an existing array or sequence, the module also provides standalone free procedures that accept `openArray[T]` directly.

---

## When to Use What

| Situation | Recommended approach |
|---|---|
| Need only one statistic from an array | Standalone proc: `mean(data)`, `variance(data)`, etc. |
| Need multiple statistics from the same data | Push once to `RunningStat`, call multiple procs |
| Data arrives as a stream (file, network, sensor) | `RunningStat` with repeated `push` calls |
| Two paired series, regression needed | `RunningRegress` |
| Parallel computation with later merge | Compute separate accumulators, combine with `+` |

---

## Table of Contents

1. [Types](#types)
   - [RunningStat](#runningstat)
   - [RunningRegress](#runningregress)
2. [RunningStat — Setup](#runningstat--setup)
   - [clear](#clear-runningstat)
   - [push (single float)](#push-single-float)
   - [push (single int)](#push-single-int)
   - [push (openArray)](#push-openarray)
3. [RunningStat — Statistics](#runningstat--statistics)
   - [mean](#mean-runningstat)
   - [variance and varianceS](#variance-and-variances)
   - [standardDeviation and standardDeviationS](#standarddeviation-and-standarddeviations)
   - [skewness and skewnessS](#skewness-and-skewnesss)
   - [kurtosis and kurtosisS](#kurtosis-and-kurtosiss)
4. [RunningStat — Combining and Display](#runningstat--combining-and-display)
   - [+ operator](#-operator-runningstat)
   - [+= operator](#-operator-runningstat-1)
   - [$ operator](#-operator-runningstat-2)
5. [RunningRegress — Setup](#runningregress--setup)
   - [clear](#clear-runningregress)
   - [push (single pair)](#push-single-pair)
   - [push (array pair)](#push-array-pair)
6. [RunningRegress — Statistics](#runningregress--statistics)
   - [slope](#slope)
   - [intercept](#intercept)
   - [correlation](#correlation)
7. [RunningRegress — Combining](#runningregress--combining)
   - [+ operator](#-operator-runningregress)
   - [+= operator](#-operator-runningregress-1)
8. [Standalone Array Procedures](#standalone-array-procedures)
9. [Population vs Sample Statistics](#population-vs-sample-statistics)
10. [Complete Example](#complete-example)
11. [Quick Reference](#quick-reference)

---

## Types

### `RunningStat`

```nim
type RunningStat* = object
  n*:          int    ## number of values pushed so far
  min*, max*:  float  ## smallest and largest value seen
  sum*:        float  ## sum of all values pushed
```

An accumulator that maintains enough internal state to compute all supported statistics in O(1) after each `push`. The public fields `n`, `min`, `max`, and `sum` can be read directly at any time. All other statistics are computed on demand by calling the corresponding procedure.

**Important:** The object must be declared as `var` because every `push` modifies it. A zero-initialised `RunningStat` (the default) is immediately ready to use — you do not need to call any constructor.

```nim
var s: RunningStat      # zero-initialised, ready to use
s.push(3.14)
echo s.n    # 1
echo s.min  # 3.14
```

---

### `RunningRegress`

```nim
type RunningRegress* = object
  n*:       int          ## number of (x, y) pairs pushed
  x_stats*: RunningStat  ## full statistics for the x series
  y_stats*: RunningStat  ## full statistics for the y series
```

An accumulator for bivariate data. Each `push` takes a matched `(x, y)` pair. In addition to exposing the full `RunningStat` for each series independently through `x_stats` and `y_stats`, it computes the joint statistics needed for linear regression and correlation.

```nim
var r: RunningRegress
r.push(1.0, 2.0)
r.push(2.0, 4.0)
r.push(3.0, 6.0)
echo r.x_stats.mean()   # 2.0
echo r.y_stats.mean()   # 4.0
echo r.slope()          # 2.0  (perfect linear relationship)
```

---

## RunningStat — Setup

### `clear` (RunningStat)

```nim
proc clear*(s: var RunningStat)
```

Resets the accumulator to its initial zero state, discarding all previously pushed values. Equivalent to assigning a fresh `RunningStat` object, but more explicit about intent.

```nim
var s: RunningStat
s.push(@[1.0, 2.0, 3.0])
echo s.n     # 3

s.clear()
echo s.n     # 0 — all data forgotten

s.push(99.0)
echo s.mean  # 99.0 — fresh start
```

---

### `push` (single float)

```nim
proc push*(s: var RunningStat, x: float)
```

The fundamental data-ingestion procedure. Updates all four statistical moments, `min`, `max`, and `sum` in a single O(1) pass using the numerically stable algorithm from Knuth's *The Art of Computer Programming*, Vol. 2, §4.2.2.

```nim
var s: RunningStat
s.push(1.0)
s.push(5.0)
s.push(3.0)
echo s.mean()  # 3.0
echo s.min     # 1.0
echo s.max     # 5.0
```

---

### `push` (single int)

```nim
proc push*(s: var RunningStat, x: int)
```

Convenience overload that converts `x` to `float` and delegates to the float version. Allows pushing integer data without manual conversion.

```nim
var s: RunningStat
s.push(10)
s.push(20)
s.push(30)
echo s.mean()  # 20.0
```

---

### `push` (openArray)

```nim
proc push*(s: var RunningStat, x: openArray[float|int])
```

Pushes all elements of `x` sequentially. Accepts arrays, sequences, or any other type that satisfies the `openArray` constraint. Both `float` and `int` elements are accepted in the same call.

```nim
var s: RunningStat
s.push(@[1.0, 2.0, 3.0, 4.0, 5.0])
echo s.n      # 5
echo s.mean() # 3.0

# Can continue pushing more data after the array
s.push(6.0)
echo s.n      # 6
```

---

## RunningStat — Statistics

All statistics are computed on demand from the accumulated internal moments. None of them modify the accumulator — they accept `s: RunningStat` (not `var`), so you can call them freely without worrying about side effects.

### `mean` (RunningStat)

```nim
proc mean*(s: RunningStat): float
```

Returns the arithmetic mean (average) of all pushed values: the sum of all values divided by their count. This is the first statistical moment.

```nim
var s: RunningStat
s.push(@[2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0])
echo s.mean()  # 5.0
```

---

### `variance` and `varianceS`

```nim
proc variance*(s: RunningStat): float   # population variance
proc varianceS*(s: RunningStat): float  # sample variance
```

**Population variance** (`variance`) treats the data as the complete population. It divides the sum of squared deviations by `n`. Use this when your data *is* the entire population you care about.

**Sample variance** (`varianceS`) treats the data as a sample drawn from a larger population. It divides by `n - 1` (Bessel's correction) to produce an unbiased estimate of the true population variance. Use this when your data is a sample and you want to generalise to the population.

`varianceS` returns `0.0` when `n <= 1` (cannot estimate variance from a single point).

```nim
var s: RunningStat
s.push(@[2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0])
echo s.variance()   # 4.0   — population: divide by n=8
echo s.varianceS()  # 4.571  — sample: divide by n-1=7
```

---

### `standardDeviation` and `standardDeviationS`

```nim
proc standardDeviation*(s: RunningStat): float   # population std dev
proc standardDeviationS*(s: RunningStat): float  # sample std dev
```

The square root of the corresponding variance. Standard deviation is expressed in the same units as the original data, making it more interpretable than variance.

- `standardDeviation` = `sqrt(variance(s))`
- `standardDeviationS` = `sqrt(varianceS(s))`

The same population-vs-sample distinction applies as for variance.

```nim
var s: RunningStat
s.push(@[2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0])
echo s.standardDeviation()   # 2.0   — population
echo s.standardDeviationS()  # 2.138 — sample (unbiased estimate)
```

---

### `skewness` and `skewnessS`

```nim
proc skewness*(s: RunningStat): float   # population skewness
proc skewnessS*(s: RunningStat): float  # sample (adjusted) skewness
```

Skewness is the **third standardised moment**. It measures the asymmetry of the distribution around the mean.

- **Zero skewness** — the distribution is symmetric (e.g. a normal distribution).
- **Positive skewness** — the right tail is longer; most values cluster on the left, with a few outliers far to the right. Example: income distributions.
- **Negative skewness** — the left tail is longer; most values cluster on the right with a few extreme low values. Example: age at retirement.

`skewnessS` applies an additional correction factor `sqrt(n*(n-1)) / (n-2)` for small samples, analogous to Bessel's correction for variance. Requires at least 3 data points.

```nim
var s: RunningStat
# Symmetric data — near-zero skewness
s.push(@[1.0, 2.0, 3.0, 4.0, 5.0])
echo s.skewness()   # ~0.0

# Right-skewed data
s.clear()
s.push(@[1.0, 1.0, 1.0, 1.0, 10.0])
echo s.skewness()   # positive — the 10.0 pulls the right tail out
```

---

### `kurtosis` and `kurtosisS`

```nim
proc kurtosis*(s: RunningStat): float   # population excess kurtosis
proc kurtosisS*(s: RunningStat): float  # sample (adjusted) excess kurtosis
```

Kurtosis is the **fourth standardised moment**. Both procedures return *excess kurtosis* — kurtosis minus 3 — so that a normal distribution has excess kurtosis of 0.

- **Excess kurtosis = 0** — normal (Gaussian) distribution, "mesokurtic".
- **Excess kurtosis > 0** — "leptokurtic": heavier tails and a sharper central peak than normal. Outliers are more common than expected. Example: financial returns.
- **Excess kurtosis < 0** — "platykurtic": lighter tails and a flatter peak than normal. Outliers are rarer than expected. Example: a uniform distribution (excess kurtosis ≈ −1.2).

`kurtosisS` applies the standard small-sample correction and requires at least 4 data points.

```nim
var s: RunningStat
s.push(@[1.0, 2.0, 1.0, 4.0, 1.0, 4.0, 1.0, 2.0])
echo s.kurtosis()   # -1.0  (platykurtic — flatter than normal)
echo s.kurtosisS()  # -0.7  (sample-corrected)
```

---

## RunningStat — Combining and Display

### `+` operator (RunningStat)

```nim
proc `+`*(a, b: RunningStat): RunningStat
```

Merges two accumulators into a single combined `RunningStat` that represents the statistics of both datasets concatenated. The merge is mathematically exact — the result is identical to having pushed all values into a single accumulator from the start.

This is the key operation for **parallel computation**: split a large dataset, process each part independently (possibly in separate threads), then merge the results with `+`.

```nim
var part1, part2: RunningStat
part1.push(@[1.0, 2.0, 3.0])
part2.push(@[4.0, 5.0, 6.0])

let combined = part1 + part2
echo combined.n      # 6
echo combined.mean() # 3.5  — same as pushing all 6 values at once
echo combined.min    # 1.0
echo combined.max    # 6.0
```

---

### `+=` operator (RunningStat)

```nim
proc `+=`*(a: var RunningStat, b: RunningStat)
```

In-place version of `+`. Merges `b` into `a`, updating `a` in place. Equivalent to `a = a + b`.

```nim
var total: RunningStat
total.push(@[1.0, 2.0, 3.0])

var extra: RunningStat
extra.push(@[4.0, 5.0])

total += extra
echo total.n      # 5
echo total.mean() # 3.0
```

---

### `$` operator (RunningStat)

```nim
proc `$`*(a: RunningStat): string
```

Returns a multi-line human-readable summary of the accumulator's current state. Useful for quick inspection during debugging. The exact format may change between Nim versions, so do not parse the output programmatically.

Currently includes: number of probes, min, max, sum, mean, and standard deviation.

```nim
var s: RunningStat
s.push(@[1.0, 2.0, 3.0, 4.0, 5.0])
echo s
# RunningStat(
#   number of probes: 5
#   max: 5.0
#   min: 1.0
#   sum: 15.0
#   mean: 3.0
#   std deviation: 1.4142135623730951
# )
```

---

## RunningRegress — Setup

### `clear` (RunningRegress)

```nim
proc clear*(r: var RunningRegress)
```

Resets the accumulator to its initial zero state, clearing both `x_stats`, `y_stats`, and the internal joint accumulator `s_xy`.

```nim
var r: RunningRegress
r.push(1.0, 2.0)
r.clear()
echo r.n  # 0
```

---

### `push` (single pair)

```nim
proc push*(r: var RunningRegress, x, y: float)
proc push*(r: var RunningRegress, x, y: int)
```

Pushes a single matched `(x, y)` data point. The two overloads accept `float` or `int` values. Int values are automatically converted to `float`.

Each call updates the internal statistics for both `x` and `y` independently (via their `RunningStat` objects) and also updates the joint accumulator used for regression and correlation.

```nim
var r: RunningRegress
r.push(0.0, 1.0)   # (x=0, y=1)
r.push(1.0, 3.0)   # (x=1, y=3)
r.push(2.0, 5.0)   # (x=2, y=5)
# Perfect linear relationship: y = 2x + 1
echo r.slope()      # 2.0
echo r.intercept()  # 1.0
```

---

### `push` (array pair)

```nim
proc push*(r: var RunningRegress, x, y: openArray[float|int])
```

Pushes two equal-length arrays of `x` and `y` values as matched pairs. The arrays must be the same length — an assertion fires if they are not. Convenient when you already have your x and y series as separate arrays.

```nim
var r: RunningRegress
let xs = @[1.0, 2.0, 3.0, 4.0, 5.0]
let ys = @[2.0, 4.0, 5.0, 4.0, 5.0]
r.push(xs, ys)
echo r.n            # 5
echo r.correlation() # Pearson r
```

---

## RunningRegress — Statistics

### `slope`

```nim
proc slope*(r: RunningRegress): float
```

Returns the slope (β₁) of the ordinary least-squares regression line `y = slope * x + intercept`. The slope represents the expected change in `y` for a one-unit increase in `x`.

```nim
var r: RunningRegress
r.push(@[1.0, 2.0, 3.0], @[2.0, 4.0, 6.0])
echo r.slope()  # 2.0 — y increases by 2 for every 1-unit increase in x
```

---

### `intercept`

```nim
proc intercept*(r: RunningRegress): float
```

Returns the y-intercept (β₀) of the regression line — the predicted value of `y` when `x` equals zero. Together with `slope`, it fully defines the least-squares regression line.

```nim
var r: RunningRegress
r.push(@[1.0, 2.0, 3.0], @[3.0, 5.0, 7.0])
# y = 2x + 1
echo r.slope()      # 2.0
echo r.intercept()  # 1.0

# Use the line to predict:
let x_new = 10.0
let y_predicted = r.slope() * x_new + r.intercept()
echo y_predicted  # 21.0
```

---

### `correlation`

```nim
proc correlation*(r: RunningRegress): float
```

Returns the Pearson product-moment correlation coefficient (r) between the `x` and `y` series. The value always lies in the range **[−1.0, 1.0]**:

- **+1.0** — perfect positive linear relationship (as x increases, y increases proportionally).
- **0.0** — no linear relationship (x and y are linearly uncorrelated).
- **−1.0** — perfect negative linear relationship (as x increases, y decreases proportionally).

Note that correlation measures only *linear* association. Two variables can be strongly related in a non-linear way and still have a correlation near zero.

```nim
var r: RunningRegress

# Perfect positive correlation
r.push(@[1.0, 2.0, 3.0], @[2.0, 4.0, 6.0])
echo r.correlation()  # 1.0

r.clear()

# No linear correlation
r.push(@[1.0, 2.0, 3.0, 4.0], @[4.0, 1.0, 4.0, 1.0])
echo r.correlation()  # ~0.0
```

---

## RunningRegress — Combining

### `+` operator (RunningRegress)

```nim
proc `+`*(a, b: RunningRegress): RunningRegress
```

Merges two regression accumulators. The result represents the combined regression as if all pairs had been pushed into a single accumulator. As with `RunningStat`, this enables parallel computation of regression statistics.

```nim
var r1, r2: RunningRegress
r1.push(@[1.0, 2.0], @[2.0, 4.0])
r2.push(@[3.0, 4.0], @[6.0, 8.0])

let combined = r1 + r2
echo combined.slope()  # 2.0 — same as pushing all 4 pairs at once
```

---

### `+=` operator (RunningRegress)

```nim
proc `+=`*(a: var RunningRegress, b: RunningRegress)
```

In-place merge. Equivalent to `a = a + b`.

---

## Standalone Array Procedures

For convenience, all statistics available through `RunningStat` are also available as free procedures that accept an `openArray[T]` directly. These are best when you need a **single statistic** from an existing sequence.

If you need **multiple statistics** from the same data, create a `RunningStat`, push once, and call the methods — this avoids re-scanning the data for each statistic.

| Procedure | Description |
|---|---|
| `mean[T](x: openArray[T]): float` | Arithmetic mean |
| `variance[T](x: openArray[T]): float` | Population variance |
| `varianceS[T](x: openArray[T]): float` | Sample variance |
| `standardDeviation[T](x: openArray[T]): float` | Population standard deviation |
| `standardDeviationS[T](x: openArray[T]): float` | Sample standard deviation |
| `skewness[T](x: openArray[T]): float` | Population skewness |
| `skewnessS[T](x: openArray[T]): float` | Sample skewness |
| `kurtosis[T](x: openArray[T]): float` | Population excess kurtosis |
| `kurtosisS[T](x: openArray[T]): float` | Sample excess kurtosis |

```nim
let data = @[2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0]

echo mean(data)               # 5.0
echo standardDeviation(data)  # 2.0

# Equivalent, but only one pass for both:
var s: RunningStat
s.push(data)
echo s.mean()               # 5.0
echo s.standardDeviation()  # 2.0
```

---

## Population vs Sample Statistics

Many statistics in this module come in two flavours. Choosing the right one matters:

**Population statistics** (`variance`, `standardDeviation`, `skewness`, `kurtosis`) treat the data you have as the **complete population**. They divide by `n`. Use these when your dataset is exhaustive — e.g., the heights of every student in one specific class.

**Sample statistics** (`varianceS`, `standardDeviationS`, `skewnessS`, `kurtosisS`) treat the data as a **random sample** from a larger, unobserved population. They apply correction factors (Bessel's correction for variance, analogous adjustments for higher moments) to produce unbiased estimates. Use these when you want to generalise — e.g., estimating the height distribution of all students everywhere based on a sample from one class.

As a rule of thumb: if in doubt, use the sample variants (`S` suffix). They are the conservative choice that most statistical textbooks default to.

---

## Complete Example

The following example demonstrates both `RunningStat` and `RunningRegress` in a realistic scenario: analysing a stream of temperature sensor readings and finding the linear trend over time.

```nim
import std/[stats, math]

# Simulated temperature readings (°C) over 10 time steps
let times  = @[0.0, 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0]
let temps  = @[20.1, 20.5, 20.8, 21.2, 21.4, 21.9, 22.1, 22.5, 22.8, 23.2]

# --- Single-series statistics ---
var s: RunningStat
s.push(temps)

echo "Count:      ", s.n
echo "Min:        ", s.min
echo "Max:        ", s.max
echo "Mean:       ", s.mean()
echo "Std dev:    ", s.standardDeviationS().round(3)
echo "Skewness:   ", s.skewnessS().round(3)
echo "Kurtosis:   ", s.kurtosisS().round(3)
echo s

# --- Linear regression: temperature vs time ---
var r: RunningRegress
r.push(times, temps)

echo "Slope:       ", r.slope().round(4),     " °C per step"
echo "Intercept:   ", r.intercept().round(4), " °C at t=0"
echo "Correlation: ", r.correlation().round(4)

# Predict temperature at step 15
let t_future = 15.0
let predicted = r.slope() * t_future + r.intercept()
echo "Predicted at t=15: ", predicted.round(2), " °C"

# --- Parallel merge ---
var part_a, part_b: RunningStat
part_a.push(temps[0 ..< 5])   # first half
part_b.push(temps[5 ..< 10])  # second half

let merged = part_a + part_b
echo "Merged mean: ", merged.mean()  # same as s.mean()
assert almostEqual(merged.mean(), s.mean())
```

---

## Quick Reference

**`RunningStat` fields (public):**

| Field | Type | Description |
|---|---|---|
| `n` | `int` | Number of values pushed |
| `min` | `float` | Smallest value seen |
| `max` | `float` | Largest value seen |
| `sum` | `float` | Sum of all values |

**`RunningStat` procedures:**

| Procedure | Description |
|---|---|
| `clear(s)` | Reset accumulator |
| `push(s, x: float/int)` | Add a single value |
| `push(s, x: openArray)` | Add all values from array |
| `mean(s)` | Arithmetic mean |
| `variance(s)` | Population variance (÷ n) |
| `varianceS(s)` | Sample variance (÷ n−1) |
| `standardDeviation(s)` | Population std dev |
| `standardDeviationS(s)` | Sample std dev |
| `skewness(s)` | Population skewness (3rd moment) |
| `skewnessS(s)` | Sample skewness |
| `kurtosis(s)` | Population excess kurtosis (4th moment − 3) |
| `kurtosisS(s)` | Sample excess kurtosis |
| `a + b` | Merge two accumulators |
| `a += b` | In-place merge |
| `$s` | Human-readable summary string |

**`RunningRegress` fields (public):**

| Field | Type | Description |
|---|---|---|
| `n` | `int` | Number of (x, y) pairs pushed |
| `x_stats` | `RunningStat` | Full stats for the x series |
| `y_stats` | `RunningStat` | Full stats for the y series |

**`RunningRegress` procedures:**

| Procedure | Description |
|---|---|
| `clear(r)` | Reset accumulator |
| `push(r, x, y: float/int)` | Add a single (x, y) pair |
| `push(r, x, y: openArray)` | Add matched array pairs |
| `slope(r)` | OLS regression slope (β₁) |
| `intercept(r)` | OLS regression intercept (β₀) |
| `correlation(r)` | Pearson correlation coefficient (−1 to +1) |
| `a + b` | Merge two accumulators |
| `a += b` | In-place merge |
