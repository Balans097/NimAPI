# Справочник модуля `std/math` (Nim)

> Базовые математические функции для Nim.  
> Тригонометрические функции работают в **радианах**. Используйте `degToRad` / `radToDeg` для перевода.  
> Модуль доступен для **JavaScript-бэкенда**.

---

## Содержание

1. [Константы](#константы)
2. [Типы](#типы)
3. [Классификация и свойства чисел с плавающей точкой](#классификация-и-свойства-чисел-с-плавающей-точкой)
4. [Корни и степени](#корни-и-степени)
5. [Логарифмы и экспоненты](#логарифмы-и-экспоненты)
6. [Тригонометрия](#тригонометрия)
7. [Обратные тригонометрические функции](#обратные-тригонометрические-функции)
8. [Гиперболические функции](#гиперболические-функции)
9. [Обратные гиперболические функции](#обратные-гиперболические-функции)
10. [Дополнительные тригонометрические функции](#дополнительные-тригонометрические-функции)
11. [Округление и деление](#округление-и-деление)
12. [Специальные функции (только C)](#специальные-функции-только-c)
13. [Преобразование углов](#преобразование-углов)
14. [Целочисленные операции](#целочисленные-операции)
15. [Агрегатные функции для массивов](#агрегатные-функции-для-массивов)

---

## Константы

| Константа | Значение | Описание |
|---|---|---|
| `PI` | `3.14159265358979…` | Число π (Людольфово число) |
| `TAU` | `6.28318530717958…` | TAU = 2 × π |
| `E` | `2.71828182845904…` | Число Эйлера |
| `MaxFloat64Precision` | `16` | Максимальное количество значимых цифр после запятой для `float64` |
| `MaxFloat32Precision` | `8` | Максимальное количество значимых цифр после запятой для `float32` |
| `MaxFloatPrecision` | `16` | Синоним `MaxFloat64Precision` |
| `MinFloatNormal` | `2.225e-308` | Наименьшее нормальное число `float` (= 2⁻¹⁰²²) |

```nim
import std/math

echo PI          # 3.141592653589793
echo TAU         # 6.283185307179586
echo E           # 2.718281828459045
echo MinFloatNormal  # 2.2250738585072014e-308
```

---

## Типы

### `FloatClass`

Перечисление, описывающее класс числа с плавающей точкой. Возвращается функцией `classify`.

| Значение | Описание |
|---|---|
| `fcNormal` | Обычное ненулевое число |
| `fcSubnormal` | Субнормальное (очень маленькое) число |
| `fcZero` | Положительный ноль |
| `fcNegZero` | Отрицательный ноль (`-0.0`) |
| `fcNan` | Не-число (NaN) |
| `fcInf` | Положительная бесконечность |
| `fcNegInf` | Отрицательная бесконечность |

---

## Классификация и свойства чисел с плавающей точкой

### `classify(x: float): FloatClass`

Определяет класс числа с плавающей точкой.

```nim
import std/math

echo classify(0.3)         # fcNormal
echo classify(0.0)         # fcZero
echo classify(-0.0)        # fcNegZero
echo classify(0.3 / 0.0)   # fcInf
echo classify(-0.3 / 0.0)  # fcNegInf
echo classify(5.0e-324)    # fcSubnormal
echo classify(NaN)         # fcNan
```

---

### `isNaN(x: SomeFloat): bool`

Проверяет, является ли значение NaN. Эффективнее, чем `classify(x) == fcNan`. Работает даже с флагом `-ffast-math`.

```nim
import std/math

echo NaN.isNaN        # true
echo Inf.isNaN        # false
echo (3.14).isNaN     # false
echo (0.0 / 0.0).isNaN # true
```

---

### `signbit(x: SomeFloat): bool`

Возвращает `true`, если `x` имеет отрицательный знаковый бит. Отличает `-0.0` от `+0.0`.

```nim
import std/math

echo signbit(0.0)   # false
echo signbit(-0.0)  # true
echo signbit(-5.0)  # true
echo signbit(5.0)   # false
```

---

### `copySign[T: SomeFloat](x, y: T): T`

Возвращает значение с **величиной** `x` и **знаком** `y`. Работает с NaN, бесконечностью и нулём.

```nim
import std/math

echo copySign(10.0, 1.0)    # 10.0
echo copySign(10.0, -1.0)   # -10.0
echo copySign(-Inf, -0.0)   # -Inf
echo copySign(NaN, 1.0).isNaN  # true
```

---

### `almostEqual[T: SomeFloat](x, y: T; unitsInLastPlace: Natural = 4): bool`

Проверяет приблизительное равенство двух чисел с плавающей точкой с помощью [машинного эпсилона](https://ru.wikipedia.org/wiki/Машинный_эпсилон).

`unitsInLastPlace` задаёт допустимое число единиц последнего разряда (ULP). Значение `0` требует точного совпадения.

```nim
import std/math

echo almostEqual(PI, 3.14159265358979)  # true
echo almostEqual(Inf, Inf)              # true
echo almostEqual(NaN, NaN)             # false (NaN != NaN)
echo almostEqual(1.0, 1.0 + 1e-15)    # true  (разница меньше эпсилона)
echo almostEqual(1.0, 1.1)             # false
```

---

### `sgn[T: SomeNumber](x: T): int`

Функция знака.

- Возвращает `1` для положительных чисел и `+Inf`.
- Возвращает `-1` для отрицательных чисел и `-Inf`.
- Возвращает `0` для `+0.0`, `-0.0` и `NaN`.

```nim
import std/math

echo sgn(5)     # 1
echo sgn(0)     # 0
echo sgn(-4.1)  # -1
echo sgn(Inf)   # 1
echo sgn(NaN)   # 0
```

---

## Корни и степени

### `sqrt(x: float32|float64): float32|float64`

Вычисляет квадратный корень из `x`.

```nim
import std/math

echo sqrt(4.0)    # 2.0
echo sqrt(1.44)   # 1.2
echo sqrt(0.0)    # 0.0
echo sqrt(-1.0)   # NaN
```

---

### `cbrt(x: float32|float64): float32|float64`

Вычисляет кубический корень из `x`. В отличие от `sqrt`, работает с отрицательными числами.

```nim
import std/math

echo cbrt(8.0)    # 2.0
echo cbrt(2.197)  # 1.3 (приближённо)
echo cbrt(-27.0)  # -3.0
```

---

### `pow(x, y: float32|float64): float32|float64`

Вычисляет `x` в степени `y`. Для целых показателей предпочтительнее использовать оператор `^`.

```nim
import std/math

echo pow(100.0, 1.5)  # 1000.0
echo pow(16.0, 0.5)   # 4.0
echo pow(2.0, 10.0)   # 1024.0
```

---

### `^[T: SomeNumber](x: T, y: Natural): T`

Оператор возведения в целочисленную (неотрицательную) степень. Работает с любым числовым типом.

```nim
import std/math

echo 2 ^ 10    # 1024
echo -3 ^ 2    # 9
echo 2.0 ^ 8   # 256.0
echo -3 ^ 0    # 1
```

---

### `^[T: SomeNumber, U: SomeFloat](x: T, y: U): float`

Оператор возведения в степень с дробным или отрицательным показателем. Обработка ошибок следует спецификации C++.

```nim
import std/math

echo 5.5 ^ 2.2     # ~42.54
echo 1.0 ^ Inf     # 1.0
echo 4.0 ^ (-0.5)  # 0.5
```

---

### `hypot(x, y: float32|float64): float32|float64`

Вычисляет длину гипотенузы прямоугольного треугольника: `sqrt(x² + y²)`. Устойчив к переполнению и потере точности.

```nim
import std/math

echo hypot(3.0, 4.0)   # 5.0
echo hypot(5.0, 12.0)  # 13.0
```

---

### `isPowerOfTwo(x: int): bool`

Возвращает `true`, если `x` является степенью двойки. Ноль и отрицательные числа возвращают `false`.

```nim
import std/math

echo isPowerOfTwo(16)   # true
echo isPowerOfTwo(5)    # false
echo isPowerOfTwo(0)    # false
echo isPowerOfTwo(-16)  # false
echo isPowerOfTwo(1)    # true
```

---

### `nextPowerOfTwo(x: int): int`

Возвращает `x`, округлённое **вверх** до ближайшей степени двойки. Ноль и отрицательные числа дают `1`.

```nim
import std/math

echo nextPowerOfTwo(16)   # 16
echo nextPowerOfTwo(5)    # 8
echo nextPowerOfTwo(0)    # 1
echo nextPowerOfTwo(-16)  # 1
echo nextPowerOfTwo(17)   # 32
```

---

## Логарифмы и экспоненты

### `ln(x: float32|float64): float32|float64`

Вычисляет натуральный логарифм `x` (основание e).

```nim
import std/math

echo ln(exp(4.0))  # 4.0
echo ln(1.0)       # 0.0
echo ln(0.0)       # -Inf
echo ln(-7.0)      # NaN (x < 0 не определено)
```

---

### `log[T: SomeFloat](x, base: T): T`

Вычисляет логарифм `x` по произвольному основанию `base`.

```nim
import std/math

echo log(9.0, 3.0)    # 2.0
echo log(0.0, 2.0)    # -Inf
echo log(-7.0, 4.0)   # NaN
echo log(8.0, -2.0)   # NaN
```

---

### `log10(x: float32|float64): float32|float64`

Вычисляет десятичный логарифм (основание 10).

```nim
import std/math

echo log10(100.0)    # 2.0
echo log10(0.0)      # -Inf
echo log10(-100.0)   # NaN
echo log10(1000.0)   # 3.0
```

---

### `log2(x: float32|float64): float32|float64`

Вычисляет двоичный логарифм (основание 2).

```nim
import std/math

echo log2(8.0)    # 3.0
echo log2(1.0)    # 0.0
echo log2(0.0)    # -Inf
echo log2(-2.0)   # NaN
```

---

### `exp(x: float32|float64): float32|float64`

Вычисляет экспоненту: `e^x`.

```nim
import std/math

echo exp(1.0)   # 2.718281828459045 (== E)
echo exp(0.0)   # 1.0
echo exp(-1.0)  # ~0.3679
```

---

### `frexp[T: float32|float64](x: T): tuple[frac: T, exp: int]`

Разбивает `x` на нормализованную дробь `frac` и показатель степени `exp` такие, что `abs(frac) ∈ [0.5, 1.0)` и `x == frac × 2^exp`.

```nim
import std/math

echo frexp(8.0)    # (frac: 0.5, exp: 4)
echo frexp(-8.0)   # (frac: -0.5, exp: 4)
echo frexp(0.0)    # (frac: 0.0, exp: 0)
```

---

### `frexp[T: float32|float64](x: T, exponent: var int): T`

Перегрузка `frexp`: сохраняет показатель в переменную и возвращает дробную часть.

```nim
import std/math

var exp: int
echo frexp(5.0, exp)  # 0.625
echo exp              # 3
```

---

### `splitDecimal[T: float32|float64](x: T): tuple[intpart: T, floatpart: T]`

Разбивает `x` на целую и дробную части (обе имеют тот же знак, что `x`). Аналог `modf` из C.

```nim
import std/math

echo splitDecimal(5.25)   # (intpart: 5.0, floatpart: 0.25)
echo splitDecimal(-2.73)  # (intpart: -2.0, floatpart: -0.73)
echo splitDecimal(0.0)    # (intpart: 0.0, floatpart: 0.0)
```

---

## Тригонометрия

> Все функции принимают аргумент в **радианах**. Используйте `degToRad` для перевода из градусов.

### `sin(x: float32|float64): float32|float64`

Вычисляет синус угла `x` в радианах.

```nim
import std/math

echo sin(PI / 6)            # 0.5
echo sin(degToRad(90.0))    # 1.0
echo sin(0.0)               # 0.0
```

---

### `cos(x: float32|float64): float32|float64`

Вычисляет косинус угла `x`.

```nim
import std/math

echo cos(2 * PI)            # 1.0
echo cos(degToRad(60.0))    # 0.5
echo cos(PI)                # -1.0
```

---

### `tan(x: float32|float64): float32|float64`

Вычисляет тангенс угла `x`.

```nim
import std/math

echo tan(degToRad(45.0))  # 1.0
echo tan(PI / 4)          # 1.0
echo tan(0.0)             # 0.0
```

---

### `cot[T: float32|float64](x: T): T`

Котангенс: `1 / tan(x)`.

```nim
import std/math

echo cot(PI / 4)  # 1.0
```

---

### `sec[T: float32|float64](x: T): T`

Секанс: `1 / cos(x)`.

```nim
import std/math

echo sec(0.0)  # 1.0
```

---

### `csc[T: float32|float64](x: T): T`

Косеканс: `1 / sin(x)`.

```nim
import std/math

echo csc(PI / 2)  # 1.0
```

---

## Обратные тригонометрические функции

### `arcsin(x: float32|float64): float32|float64`

Арксинус `x`, результат в диапазоне `[-π/2, π/2]`.

```nim
import std/math

echo radToDeg(arcsin(0.0))  # 0.0
echo radToDeg(arcsin(1.0))  # 90.0
echo arcsin(-1.0)           # -1.5707... (-π/2)
```

---

### `arccos(x: float32|float64): float32|float64`

Арккосинус `x`, результат в диапазоне `[0, π]`.

```nim
import std/math

echo radToDeg(arccos(0.0))  # 90.0
echo radToDeg(arccos(1.0))  # 0.0
```

---

### `arctan(x: float32|float64): float32|float64`

Арктангенс `x`, результат в диапазоне `(-π/2, π/2)`.

```nim
import std/math

echo arctan(1.0)            # ~0.7854 (π/4)
echo radToDeg(arctan(1.0))  # 45.0
```

---

### `arctan2(y, x: float32|float64): float32|float64`

Арктангенс `y/x` с учётом знаков обоих аргументов (полный 4-квадрантный арктангенс). Диапазон результата `(-π, π]`. Корректно работает при `x ≈ 0`.

```nim
import std/math

echo radToDeg(arctan2(1.0, 0.0))   # 90.0
echo radToDeg(arctan2(0.0, -1.0))  # 180.0
echo radToDeg(arctan2(-1.0, 0.0))  # -90.0
```

---

### `arccot[T: float32|float64](x: T): T`

Обратный котангенс: `arctan(1/x)`.

---

### `arcsec[T: float32|float64](x: T): T`

Обратный секанс: `arccos(1/x)`.

---

### `arccsc[T: float32|float64](x: T): T`

Обратный косеканс: `arcsin(1/x)`.

---

## Гиперболические функции

### `sinh(x: float32|float64): float32|float64`

Гиперболический синус.

```nim
import std/math

echo sinh(0.0)  # 0.0
echo sinh(1.0)  # ~1.1752
```

---

### `cosh(x: float32|float64): float32|float64`

Гиперболический косинус.

```nim
import std/math

echo cosh(0.0)  # 1.0
echo cosh(1.0)  # ~1.5431
```

---

### `tanh(x: float32|float64): float32|float64`

Гиперболический тангенс (диапазон `(-1, 1)`).

```nim
import std/math

echo tanh(0.0)  # 0.0
echo tanh(1.0)  # ~0.7616
```

---

### `coth[T: float32|float64](x: T): T`

Гиперболический котангенс: `1 / tanh(x)`.

---

### `sech[T: float32|float64](x: T): T`

Гиперболический секанс: `1 / cosh(x)`.

---

### `csch[T: float32|float64](x: T): T`

Гиперболический косеканс: `1 / sinh(x)`.

---

## Обратные гиперболические функции

### `arcsinh(x: float32|float64): float32|float64`

Обратный гиперболический синус.

### `arccosh(x: float32|float64): float32|float64`

Обратный гиперболический косинус (определён для `x ≥ 1`).

### `arctanh(x: float32|float64): float32|float64`

Обратный гиперболический тангенс (определён для `|x| < 1`).

### `arccoth[T: float32|float64](x: T): T`

Обратный гиперболический котангенс: `arctanh(1/x)`.

### `arcsech[T: float32|float64](x: T): T`

Обратный гиперболический секанс: `arccosh(1/x)`.

### `arccsch[T: float32|float64](x: T): T`

Обратный гиперболический косеканс: `arcsinh(1/x)`.

```nim
import std/math

echo arcsinh(0.0)   # 0.0
echo arccosh(1.0)   # 0.0
echo arctanh(0.5)   # ~0.5493
```

---

## Округление и деление

### `floor(x: float32|float64): float32|float64`

Возвращает наибольшее целое, не превышающее `x` (округление вниз).

```nim
import std/math

echo floor(2.1)   # 2.0
echo floor(2.9)   # 2.0
echo floor(-3.5)  # -4.0
echo floor(-2.0)  # -2.0
```

---

### `ceil(x: float32|float64): float32|float64`

Возвращает наименьшее целое, не меньшее `x` (округление вверх).

```nim
import std/math

echo ceil(2.1)   # 3.0
echo ceil(2.9)   # 3.0
echo ceil(-2.1)  # -2.0
```

---

### `trunc(x: float32|float64): float32|float64`

Усекает `x` до целой части (округление к нулю).

```nim
import std/math

echo trunc(PI)     # 3.0
echo trunc(-1.85)  # -1.0
echo trunc(2.9)    # 2.0
```

---

### `round(x: float32|float64): float32|float64`

Округляет до ближайшего целого по математическим правилам (от нуля для .5).

```nim
import std/math

echo round(3.4)  # 3.0
echo round(3.5)  # 4.0
echo round(4.5)  # 5.0
```

---

### `round[T: float32|float64](x: T, places: int): T`

Округляет `x` до `places` знаков после запятой.

- `places > 0` — знаки после запятой.
- `places = 0` — до целого.
- `places < 0` — до степеней десяти слева от запятой.

> ⚠️ Функция **ненадёжна** из-за ограничений двоичного представления float.

```nim
import std/math

echo round(PI, 2)    # 3.14
echo round(PI, 4)    # 3.1416
echo round(537.345, -1)  # 540.0
```

---

### `` `mod`(x, y: float32|float64): float32|float64 ``

Остаток от деления для чисел с плавающей точкой. Знак результата совпадает со знаком **делимого** `x`.

```nim
import std/math

echo  6.5 mod  2.5  #  1.5
echo -6.5 mod  2.5  # -1.5
echo  6.5 mod -2.5  #  1.5
echo -6.5 mod -2.5  # -1.5
```

---

### `floorDiv[T: SomeInteger](x, y: T): T`

Целочисленное деление с округлением **вниз** (`floor(x/y)`). Отличается от `div`, который округляет к нулю.

```nim
import std/math

echo floorDiv( 13,  3)  #  4
echo floorDiv(-13,  3)  # -5  (div дал бы -4)
echo floorDiv( 13, -3)  # -5
echo floorDiv(-13, -3)  #  4
```

---

### `floorMod[T: SomeNumber](x, y: T): T`

Остаток от деления, согласованный с `floorDiv`. Ведёт себя как оператор `%` в Python — результат всегда имеет знак **делителя** `y`.

```nim
import std/math

echo floorMod( 13,  3)  #  1
echo floorMod(-13,  3)  #  2  (а не -1!)
echo floorMod( 13, -3)  # -2
echo floorMod(-13, -3)  # -1
```

---

### `euclDiv[T: SomeInteger](x, y: T): T`

Евклидово деление. Остаток всегда неотрицателен.

```nim
import std/math

echo euclDiv(13, 3)    #  4
echo euclDiv(-13, 3)   # -5
echo euclDiv(13, -3)   # -4
echo euclDiv(-13, -3)  #  5
```

---

### `euclMod[T: SomeNumber](x, y: T): T`

Евклидов остаток. **Всегда неотрицателен**, независимо от знаков `x` и `y`.

```nim
import std/math

echo euclMod( 13,  3)  # 1
echo euclMod(-13,  3)  # 2
echo euclMod( 13, -3)  # 1
echo euclMod(-13, -3)  # 2
```

---

### `ceilDiv[T: SomeInteger](x, y: T): T`

Целочисленное деление с округлением **вверх** (`ceil(x/y)`). Требует `x ≥ 0` и `y > 0`.

```nim
import std/math

echo ceilDiv(12, 3)  # 4
echo ceilDiv(13, 3)  # 5
echo ceilDiv(1, 3)   # 1
```

---

### `divmod[T: SomeInteger](x, y: T): (T, T)`

Вычисляет частное и остаток за одну операцию. Возвращает `(quotient, remainder)`.

```nim
import std/math

echo divmod(5, 2)    # (2, 1)
echo divmod(5, -3)   # (-1, 2)
echo divmod(-10, 3)  # (-3, -1)
```

---

### `clamp[T](val: T, bounds: Slice[T]): T`

Ограничивает значение `val` диапазоном `bounds`. Удобная форма записи через срез.

```nim
import std/math

echo clamp(10, 1..5)  # 5
echo clamp(3, 1..5)   # 3
echo clamp(0, 1..5)   # 1
```

---

## Специальные функции (только C)

> ⚠️ Следующие функции **недоступны для JavaScript-бэкенда**.

### `erf(x: float32|float64): float32|float64`

Функция ошибок (error function).

```nim
import std/math

echo erf(0.0)   # 0.0
echo erf(1.0)   # ~0.8427
echo erf(Inf)   # 1.0
```

---

### `erfc(x: float32|float64): float32|float64`

Дополнительная функция ошибок: `erfc(x) = 1 - erf(x)`.

```nim
import std/math

echo erfc(0.0)  # 1.0
echo erfc(1.0)  # ~0.1573
```

---

### `gamma(x: float32|float64): float32|float64`

Гамма-функция. Обобщение факториала: `gamma(n) == fac(n-1)` для целых `n > 0`.

```nim
import std/math

echo gamma(1.0)   # 1.0
echo gamma(4.0)   # 6.0   (= 3!)
echo gamma(11.0)  # 3628800.0 (= 10!)
echo gamma(0.5)   # ~1.7725 (= sqrt(PI))
```

---

### `lgamma(x: float32|float64): float32|float64`

Натуральный логарифм гамма-функции `ln(gamma(x))`. Полезен для предотвращения переполнения.

```nim
import std/math

echo lgamma(1.0)   # 0.0
echo lgamma(11.0)  # ~15.104  (ln(3628800))
```

---

## Преобразование углов

### `degToRad[T: float32|float64](d: T): T`

Переводит градусы в радианы.

```nim
import std/math

echo degToRad(180.0)  # ~3.14159 (== PI)
echo degToRad(90.0)   # ~1.5708  (== PI/2)
echo degToRad(360.0)  # ~6.2832  (== TAU)
```

---

### `radToDeg[T: float32|float64](d: T): T`

Переводит радианы в градусы.

```nim
import std/math

echo radToDeg(PI)      # 180.0
echo radToDeg(2 * PI)  # 360.0
echo radToDeg(PI / 4)  # 45.0
```

---

## Целочисленные операции

### `binom(n, k: int): int`

Вычисляет [биномиальный коэффициент](https://ru.wikipedia.org/wiki/Биномиальный_коэффициент) C(n, k).

```nim
import std/math

echo binom(6, 2)   # 15
echo binom(6, 0)   # 1
echo binom(-6, 2)  # 1
echo binom(10, 3)  # 120
```

---

### `fac(n: int): int`

Вычисляет [факториал](https://ru.wikipedia.org/wiki/Факториал) `n!`. Допустимый диапазон ограничен размером `int` (до 20 на 64-битных системах).

```nim
import std/math

echo fac(0)   # 1
echo fac(4)   # 24
echo fac(10)  # 3628800
echo fac(20)  # 2432902008176640000
```

---

### `gcd[T](x, y: T): T`

Вычисляет наибольший общий делитель (НОД) двух значений. Для целых использует быстрый бинарный алгоритм (алгоритм Штайна). Для `float` результат не всегда точен.

```nim
import std/math

echo gcd(12, 8)       # 4
echo gcd(17, 63)      # 1
echo gcd(13.5, 9.0)   # 4.5
```

---

### `gcd[T](x: openArray[T]): T`

Вычисляет НОД всех элементов массива.

```nim
import std/math

echo gcd(@[12, 8, 4])      # 4
echo gcd(@[13.5, 9.0])     # 4.5
```

---

### `lcm[T](x, y: T): T`

Вычисляет наименьшее общее кратное (НОК) двух значений.

```nim
import std/math

echo lcm(24, 30)  # 120
echo lcm(13, 39)  # 39
echo lcm(4, 6)    # 12
```

---

### `lcm[T](x: openArray[T]): T`

Вычисляет НОК всех элементов массива.

```nim
import std/math

echo lcm(@[24, 30])    # 120
echo lcm(@[4, 6, 10])  # 60
```

---

## Агрегатные функции для массивов

### `sum[T](x: openArray[T]): T`

Вычисляет сумму элементов. Для пустого массива возвращает `0`.

```nim
import std/math

echo sum([1, 2, 3, 4])   # 10
echo sum([-4, 3, 5])     # 4
echo sum([0.5, 1.5])     # 2.0
echo sum(newSeq[int]())  # 0
```

---

### `prod[T](x: openArray[T]): T`

Вычисляет произведение элементов. Для пустого массива возвращает `1`.

```nim
import std/math

echo prod([1, 2, 3, 4])  # 24
echo prod([-4, 3, 5])    # -60
echo prod([2, 2, 2])     # 8
```

---

### `cumsummed[T](x: openArray[T]): seq[T]`

Возвращает новую последовательность с накопленными (префиксными) суммами. Для пустого массива — `@[]`.

```nim
import std/math

echo cumsummed([1, 2, 3, 4])  # @[1, 3, 6, 10]
echo cumsummed([5, -1, 3])    # @[5, 4, 7]
```

---

### `cumsum[T](x: var openArray[T])`

Преобразует массив **на месте** в накопленные суммы.

```nim
import std/math

var a = [1, 2, 3, 4]
cumsum(a)
echo a  # @[1, 3, 6, 10]
```

---

### `cumproded[T](x: openArray[T]): seq[T]`

Возвращает новую последовательность с накопленными (префиксными) произведениями.

```nim
import std/math

echo cumproded([1, 2, 3, 4])  # @[1, 2, 6, 24]
```

---

### `cumprod[T](x: var openArray[T])`

Преобразует массив **на месте** в накопленные произведения.

```nim
import std/math

var a = [1, 2, 3, 4]
cumprod(a)
echo a  # @[1, 2, 6, 24]
```

---

## Полные примеры

### Генерация нормального распределения (Box–Muller)

```nim
import std/math
import std/[random, fenv]

proc gaussianNoise(mu = 0.0, sigma = 1.0): (float, float) =
  var u1, u2: float
  while true:
    u1 = rand(1.0)
    u2 = rand(1.0)
    if u1 > epsilon(float): break
  let mag = sigma * sqrt(-2.0 * ln(u1))
  let z0 = mag * cos(2 * PI * u2) + mu
  let z1 = mag * sin(2 * PI * u2) + mu
  (z0, z1)

randomize()
echo gaussianNoise()
```

---

### Решение прямоугольного треугольника

```nim
import std/math

let a = 3.0
let b = 4.0
let c = hypot(a, b)
echo "Гипотенуза: ", c          # 5.0

let angle_A = radToDeg(arctan2(a, b))
echo "Угол A: ", angle_A, "°"   # ~36.87°
```

---

### Сравнение операций деления

```nim
import std/math

let x = -13
let y = 3

echo "div (к нулю):       ", x div y         # -4
echo "floorDiv (вниз):    ", floorDiv(x, y)  # -5
echo "euclDiv (евклидово):", euclDiv(x, y)   # -5
echo "ceilDiv (вверх):    ", ceilDiv(13, y)  # 5

echo "mod (к нулю):       ", x mod y         # -1
echo "floorMod (вниз):    ", floorMod(x, y)  #  2
echo "euclMod (≥0):       ", euclMod(x, y)   #  2
```

---

### Накопленные суммы для скользящих данных

```nim
import std/math

let data = [10, 5, 8, 3, 7]
let prefix = cumsummed(data)
echo prefix  # @[10, 15, 23, 26, 33]

# Сумма подотрезка [1..3] через prefix sums:
let rangeSum = prefix[3] - prefix[0]
echo rangeSum  # 16 (= 5 + 8 + 3)
```
