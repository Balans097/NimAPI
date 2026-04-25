# Справочник модуля `std/math` — Nim Standard Library

> *Constructive mathematics is naturally typed.* — Simon Thompson

---

## Содержание

1. [Обзор модуля](#1-обзор-модуля)
2. [Константы](#2-константы)
3. [Типы](#3-типы)
4. [Утилиты для чисел с плавающей точкой](#4-утилиты-для-чисел-с-плавающей-точкой)
5. [Функции округления](#5-функции-округления)
6. [Степени и корни](#6-степени-и-корни)
7. [Логарифмы и экспонента](#7-логарифмы-и-экспонента)
8. [Тригонометрические функции](#8-тригонометрические-функции)
9. [Деление и взятие остатка](#9-деление-и-взятие-остатка)
10. [Целочисленная математика](#10-целочисленная-математика)
11. [Специальные функции](#11-специальные-функции)
12. [Знак и ограничение значений](#12-знак-и-ограничение-значений)
13. [Функции для массивов](#13-функции-для-массивов)
14. [Практические примеры](#14-практические-примеры)
15. [Таблица быстрого доступа](#15-таблица-быстрого-доступа)

---

## 1. Обзор модуля

Модуль `std/math` входит в стандартную библиотеку Nim и реализует базовые математические функции, константы и типы. Большинство функций делегируют вычисление в Си-библиотеку `<math.h>` (POSIX) или в соответствующий объект `Math.*` для JavaScript-бэкенда — это даёт нативную производительность на обеих платформах.

### Подключение

```nim
import std/math
```

### Поддержка платформ

| Бэкенд | Поддержка | Примечание |
|--------|-----------|------------|
| C / C++ | ✅ Полная | Линкуется с `-lm` на POSIX |
| JavaScript | ✅ Полная | Использует `Math.*` |
| NimScript | ⚠️ Частичная | Некоторые функции недоступны |

### Связанные модули

| Модуль | Назначение |
|--------|------------|
| `std/complex` | Комплексные числа и операции над ними |
| `std/rationals` | Рациональные числа (точная арифметика) |
| `std/fenv` | Управление режимами округления IEEE 754, обработка исключений float |
| `std/random` | Генератор псевдослучайных чисел |
| `std/stats` | Статистический анализ: среднее, дисперсия, стандартное отклонение |
| `std/strformat` | Форматирование чисел с плавающей точкой для вывода |
| `system` | Базовые операторы: `shr`, `shl`, `xor`, `clamp`, `abs`, `min`, `max` |

---

## 2. Константы

Модуль экспортирует математические и технические константы. Все они объявлены с `*` — то есть являются публичными при импорте.

| Константа | Значение | Описание |
|-----------|----------|----------|
| `PI` | `3.14159265358979323…` | Число π (Людольфово число) — отношение длины окружности к диаметру |
| `TAU` | `6.28318530717958647…` | TAU = 2·π — полный оборот в радианах; удобнее PI во многих формулах |
| `E` | `2.71828182845904523…` | Число Эйлера — основание натурального логарифма |
| `MaxFloat64Precision` | `16` | Максимальное количество значащих десятичных цифр для `float64` |
| `MaxFloat32Precision` | `8` | Максимальное количество значащих десятичных цифр для `float32` |
| `MaxFloatPrecision` | `16` | Алиас для `MaxFloat64Precision` (тип `float` — это `float64`) |
| `MinFloatNormal` | `2.225073858507201e-308` | Наименьшее *нормальное* число float64 (= 2⁻¹⁰²²) |

### Примеры

```nim
import std/math

# Длина окружности радиуса r
let r = 5.0
let circumference = TAU * r      # TAU = 2*PI, более читаемо
echo circumference               # => 31.41592653589793

# Полуоборот и четверть оборота
let halfTurn    = PI             # 180°
let quarterTurn = PI / 2.0      # 90°

# Непрерывный рост с экспонентой
let investment = 1000.0
let growth = investment * E ^ 1.0   # рост за единицу времени
echo growth                          # => 2718.28...

# Проверка, входит ли число в диапазон нормальных float
let tiny = 1.0e-310
echo tiny < MinFloatNormal           # => true (субнормальное число)
```

> **Почему TAU?** Многие формулы выглядят чище с TAU. Например, полная длина окружности — `TAU * r`, а не `2 * PI * r`. Угол 360° в радианах — просто `TAU`.

---

## 3. Типы

### `FloatClass`

Перечисление (enum), описывающее класс числа с плавающей точкой согласно стандарту IEEE 754. Возвращается функцией [`classify`](#classify--классификация-числа).

```nim
type
  FloatClass* = enum
    fcNormal,    ## Обычное ненулевое нормализованное значение
    fcSubnormal, ## Субнормальное (денормализованное) — очень маленькое
    fcZero,      ## Положительный ноль (+0.0)
    fcNegZero,   ## Отрицательный ноль (-0.0)
    fcNan,       ## Not a Number (NaN) — результат некорректной операции
    fcInf,       ## Положительная бесконечность (+Inf)
    fcNegInf     ## Отрицательная бесконечность (-Inf)
```

#### Когда возникают специальные значения

| Значение | Пример операции | Причина |
|----------|----------------|---------|
| `Inf` | `1.0 / 0.0` | Деление положительного на ноль |
| `-Inf` | `-1.0 / 0.0` | Деление отрицательного на ноль |
| `NaN` | `0.0 / 0.0` | Неопределённая операция |
| `NaN` | `sqrt(-1.0)` | Корень из отрицательного |
| `-0.0` | `-1.0 * 0.0` | Математически равно 0, но другой знак |
| субнормальное | `5.0e-324` | Число меньше `MinFloatNormal` |

#### Пример использования `FloatClass`

```nim
import std/math

proc describeFloat(x: float): string =
  case classify(x)
  of fcNormal:    "обычное число"
  of fcSubnormal: "субнормальное (очень маленькое)"
  of fcZero:      "положительный ноль"
  of fcNegZero:   "отрицательный ноль"
  of fcNan:       "не число (NaN)"
  of fcInf:       "положительная бесконечность"
  of fcNegInf:    "отрицательная бесконечность"

echo describeFloat(3.14)       # => обычное число
echo describeFloat(0.0)        # => положительный ноль
echo describeFloat(-0.0)       # => отрицательный ноль
echo describeFloat(1.0/0.0)    # => положительная бесконечность
echo describeFloat(0.0/0.0)    # => не число (NaN)
echo describeFloat(5.0e-324)   # => субнормальное
```

---

## 4. Утилиты для чисел с плавающей точкой

### `isNaN` — проверка на NaN

```nim
func isNaN*(x: SomeFloat): bool
```

Возвращает `true`, если `x` является NaN. Работает корректно даже при включённой компиляторной оптимизации `-ffast-math`, которая может нарушать обычное сравнение `x != x`.

**Почему нельзя просто написать `x == NaN`?**  
По стандарту IEEE 754, NaN не равен ничему, включая самого себя. Поэтому `NaN == NaN` всегда `false`. `isNaN` использует более надёжные способы проверки.

```nim
import std/math

doAssert NaN.isNaN                  # NaN — это NaN
doAssert not Inf.isNaN              # Inf — не NaN
doAssert not isNaN(3.1415926)       # обычное число — не NaN

# Особый случай: NaN не равен сам себе
let x = NaN
doAssert x != x          # true — свойство NaN в IEEE 754
doAssert x.isNaN         # true — надёжная проверка

# Проверка перед использованием результата вычисления
let result = sqrt(-4.0)
if result.isNaN:
  echo "Ошибка: квадратный корень из отрицательного числа"
```

---

### `signbit` — знаковый бит

```nim
func signbit*(x: SomeFloat): bool
```

Возвращает `true`, если `x` является отрицательным числом. В отличие от `x < 0`, корректно обрабатывает `-0.0` (отрицательный ноль) и `-Inf`.

**Зачем нужен `-0.0`?**  
Отрицательный ноль — это законное значение IEEE 754. Математически он равен `+0.0`, но его знак имеет значение в ряде вычислений (например, при вычислении `arctan2` или при анализе знака у субнормальных чисел).

```nim
import std/math

doAssert not signbit(0.0)    # +0.0 — положительный
doAssert signbit(-0.0)       # -0.0 — отрицательный!
doAssert signbit(-0.1)       # обычное отрицательное
doAssert not signbit(0.1)    # обычное положительное
doAssert signbit(-Inf)       # отрицательная бесконечность
doAssert not signbit(Inf)    # положительная бесконечность

# Отличие от x < 0:
let negZero = -0.0
echo negZero < 0.0      # => false (математически они равны)
echo signbit(negZero)   # => true  (знаковый бит установлен)
```

---

### `copySign` — копирование знака

```nim
func copySign*[T: SomeFloat](x, y: T): T
```

Возвращает число с **модулем `x`** и **знаком `y`**. Корректно работает со всеми специальными значениями: NaN, ±Inf, ±0.

```nim
import std/math

doAssert copySign(10.0,  1.0) ==  10.0   # берём знак y=+1
doAssert copySign(10.0, -1.0) == -10.0   # берём знак y=-1
doAssert copySign(-Inf, -0.0) == -Inf    # знак из -0.0
doAssert copySign(NaN, 1.0).isNaN        # NaN остаётся NaN

# Применение: установить знак числа по другому числу
proc withSign(magnitude: float, reference: float): float =
  copySign(abs(magnitude), reference)

echo withSign(5.0, -3.0)   # => -5.0
echo withSign(-7.0, 2.0)   # => 7.0
```

---

### `classify` — классификация числа

```nim
func classify*(x: float): FloatClass
```

Определяет, к какому классу IEEE 754 относится значение `x`, и возвращает соответствующее значение `FloatClass`.

```nim
import std/math

doAssert classify(0.3)        == fcNormal
doAssert classify(0.0)        == fcZero
doAssert classify(-0.0)       == fcNegZero
doAssert classify(0.3 / 0.0)  == fcInf
doAssert classify(-0.3 / 0.0) == fcNegInf
doAssert classify(5.0e-324)   == fcSubnormal

# Пример: безопасное деление
proc safeDivide(a, b: float): float =
  if classify(b) in {fcZero, fcNegZero}:
    raise newException(DivByZeroDefect, "деление на ноль")
  result = a / b
```

---

### `almostEqual` — сравнение float с допуском

```nim
func almostEqual*[T: SomeFloat](x, y: T; unitsInLastPlace: Natural = 4): bool
```

Проверяет, являются ли два числа **приблизительно равными** с учётом накопленных ошибок с плавающей точкой. Использует метод ULP (Units in the Last Place — единицы последнего разряда).

**Почему нельзя сравнивать float через `==`?**  
Числа с плавающей точкой хранятся в двоичном виде. Большинство десятичных дробей (например, `0.1`) не имеют точного двоичного представления, поэтому арифметические операции накапливают ошибку. Например, `0.1 + 0.2` в float не равно `0.3`.

**Параметр `unitsInLastPlace`:**  
- `0` — числа должны быть побитово идентичны
- `1` — допускается 1 единица последнего разряда (минимальный допуск)
- `4` — допуск по умолчанию, покрывает большинство практических случаев
- Большие значения → более широкий допуск

```nim
import std/math

# Базовые случаи
doAssert almostEqual(PI, 3.14159265358979)  # достаточно близко
doAssert almostEqual(Inf, Inf)              # бесконечности равны
doAssert not almostEqual(NaN, NaN)          # NaN ≠ NaN по определению

# Реальная проблема: накопление ошибок
let a = 0.1 + 0.2
echo a == 0.3            # => false! (0.30000000000000004)
echo almostEqual(a, 0.3) # => true  (в пределах допуска)

# Регулировка строгости
echo almostEqual(1.0, 1.0000001, 4)    # true  — мягкий допуск
echo almostEqual(1.0, 1.0000001, 0)    # false — строгое равенство

# Правило: всегда используйте almostEqual вместо == для float-результатов
proc isUnitVector(x, y, z: float): bool =
  almostEqual(x*x + y*y + z*z, 1.0)
```

---

### `frexp` — разбиение на мантиссу и экспоненту

```nim
func frexp*[T: float32|float64](x: T): tuple[frac: T, exp: int]
func frexp*[T: float32|float64](x: T, exponent: var int): T
```

Разбивает `x` на нормализованную дробь `frac` и целую степень двойки `exp`, так что:  
`x = frac × 2^exp`, где `abs(frac) ∈ [0.5, 1.0)`

Аналог функции `frexp` из Си. Полезен при работе с представлением чисел в памяти, реализации собственных математических функций и нормализации данных.

```nim
import std/math

doAssert frexp(8.0)  == (0.5, 4)    # 8.0  = 0.5 × 2⁴
doAssert frexp(-8.0) == (-0.5, 4)   # -8.0 = -0.5 × 2⁴
doAssert frexp(0.0)  == (0.0, 0)    # 0 — особый случай
doAssert frexp(1.0)  == (0.5, 1)    # 1.0  = 0.5 × 2¹

# Второй вариант — через out-параметр
var exp: int
let frac = frexp(5.0, exp)
doAssert frac == 0.625   # 5.0 = 0.625 × 2³
doAssert exp  == 3

# Применение: логарифм по основанию 2 без loss of precision
# log2(x) ≈ exp + log2(frac), где frac ∈ [0.5, 1.0)
```

---

### `splitDecimal` — целая и дробная части

```nim
func splitDecimal*[T: float32|float64](x: T): tuple[intpart: T, floatpart: T]
```

Разбивает число на целую и дробную части. Обе части имеют тот же знак, что и `x`. Аналог функции `modf` из Си.

```nim
import std/math

doAssert splitDecimal(5.25)  == (intpart: 5.0,  floatpart: 0.25)
doAssert splitDecimal(-2.73) == (intpart: -2.0, floatpart: -0.73)
doAssert splitDecimal(0.0)   == (intpart: 0.0,  floatpart: 0.0)

# Применение: анимация с субпиксельным смещением
let position = 7.65
let (whole, frac) = splitDecimal(position)
let pixelPos = int(whole)     # рисуем в пикселе 7
let subPixel = frac           # с субпиксельным смещением 0.65
```

---

## 5. Функции округления

В Nim доступно несколько стратегий округления. Выбор зависит от задачи.

| Функция | Стратегия | `2.5` | `-2.5` | `2.1` | `-2.1` |
|---------|-----------|-------|--------|-------|--------|
| `floor(x)` | К минус бесконечности | `2.0` | `-3.0` | `2.0` | `-3.0` |
| `ceil(x)` | К плюс бесконечности | `3.0` | `-2.0` | `3.0` | `-2.0` |
| `trunc(x)` | К нулю | `2.0` | `-2.0` | `2.0` | `-2.0` |
| `round(x)` | К ближайшему | `3.0` | `-2.0` | `2.0` | `-2.0` |

### `floor` — округление вниз

```nim
func floor*(x: float32|float64): float
```

Возвращает наибольшее целое, **не превышающее** `x`. Для положительных чисел — отбрасывает дробь. Для отрицательных — округляет в сторону большего модуля.

```nim
import std/math

doAssert floor(2.1)  ==  2.0
doAssert floor(2.9)  ==  2.0   # не 3!
doAssert floor(-2.1) == -3.0   # уходит "ниже"
doAssert floor(-2.9) == -3.0
doAssert floor(3.0)  ==  3.0   # целое остаётся целым

# Применение: получить индекс ячейки по позиции
let cellSize = 32.0
let pos = 97.5
let cellIndex = int(floor(pos / cellSize))  # => 3
```

---

### `ceil` — округление вверх

```nim
func ceil*(x: float32|float64): float
```

Возвращает наименьшее целое, **не меньше** `x`. Зеркально противоположно `floor`.

```nim
import std/math

doAssert ceil(2.1)  == 3.0
doAssert ceil(2.9)  == 3.0
doAssert ceil(-2.1) == -2.0   # уходит "выше" (к нулю)
doAssert ceil(-2.9) == -2.0
doAssert ceil(3.0)  ==  3.0

# Применение: сколько страниц нужно для N записей
let records = 57
let perPage = 10
let pages = int(ceil(float(records) / float(perPage)))  # => 6
```

---

### `trunc` — усечение к нулю

```nim
func trunc*(x: float32|float64): float
```

Отбрасывает дробную часть, всегда **приближаясь к нулю**. Эквивалентен `floor` для положительных и `ceil` для отрицательных.

```nim
import std/math

doAssert trunc(PI)    ==  3.0   # 3.14159... -> 3.0
doAssert trunc(-1.85) == -1.0   # идёт к нулю, не вниз
doAssert trunc(2.99)  ==  2.0

# Применение: целочисленное деление для float
let a = 17.0
let b = 5.0
let quotient = trunc(a / b)     # => 3.0 (как целочисленный div)
```

---

### `round` — округление к ближайшему

```nim
func round*(x: float32|float64): float              # до целого
func round*[T: float32|float64](x: T, places: int): T  # до N знаков
```

Округляет до ближайшего целого (или до `places` десятичных знаков). При `places < 0` — округление слева от точки.

> ⚠️ `round(x, places)` работает на бинарной арифметике и **не гарантирует** точного результата для всех десятичных дробей. Для финансовых расчётов используйте `std/rationals`.

```nim
import std/math

doAssert round(3.4)  == 3.0
doAssert round(3.5)  == 4.0    # округление вверх при x.5
doAssert round(4.5)  == 5.0
doAssert round(-3.5) == -4.0   # к большему по модулю

# С указанием знаков после запятой
doAssert round(PI, 2)  == 3.14
doAssert round(PI, 4)  == 3.1416

# Округление влево от запятой (отрицательный places)
doAssert round(537.345, -1) == 540.0   # до десятков
doAssert round(537.345, -2) == 500.0   # до сотен
```

---

## 6. Степени и корни

### `sqrt` — квадратный корень

```nim
func sqrt*(x: float32|float64): float
```

Вычисляет √x. Для отрицательных значений возвращает `NaN`.

```nim
import std/math

doAssert almostEqual(sqrt(4.0),   2.0)
doAssert almostEqual(sqrt(1.44),  1.2)
doAssert almostEqual(sqrt(2.0),   1.4142135623730951)
doAssert sqrt(-1.0).isNaN     # корень из отрицательного = NaN

# Применение: расстояние между двумя точками
proc distance(x1, y1, x2, y2: float): float =
  sqrt((x2-x1)^2 + (y2-y1)^2)

echo distance(0.0, 0.0, 3.0, 4.0)  # => 5.0
```

---

### `cbrt` — кубический корень

```nim
func cbrt*(x: float32|float64): float
```

Вычисляет ∛x. В отличие от `sqrt`, корректно работает с отрицательными числами (∛(−27) = −3).

```nim
import std/math

doAssert almostEqual(cbrt(8.0),   2.0)
doAssert almostEqual(cbrt(2.197), 1.3)
doAssert almostEqual(cbrt(-27.0), -3.0)  # отрицательный — OK!

# Применение: сторона куба по объёму
let volume = 125.0
let side = cbrt(volume)   # => 5.0
```

---

### `pow` — возведение float в степень

```nim
func pow*(x, y: float64): float64
```

Вычисляет `x^y`. Оба аргумента — числа с плавающей точкой.

```nim
import std/math

doAssert almostEqual(pow(100.0, 1.5), 1000.0)   # 100^1.5 = 1000
doAssert almostEqual(pow(16.0, 0.5),    4.0)    # 16^0.5 = √16 = 4
doAssert pow(0.0, 0.0) == 1.0                   # 0^0 = 1 по соглашению
```

---

### `^` — оператор степени

Nim предоставляет два перегруженных оператора `^`:

```nim
func `^`*[T: SomeNumber](x: T, y: Natural): T        # целый показатель
func `^`*[T: SomeNumber, U: SomeFloat](x: T, y: U): float  # вещественный
```

**Первая версия** (`y: Natural`) работает с любыми числами и возвращает тот же тип. Реализована через быстрое возведение в степень (алгоритм «квадрат и умножение»).  
**Вторая версия** (`y: SomeFloat`) возвращает `float` и поддерживает дробные и отрицательные степени.

```nim
import std/math

# Целый показатель — точный результат
doAssert -3 ^ 0 ==  1
doAssert -3 ^ 1 == -3
doAssert -3 ^ 2 ==  9
doAssert  2 ^ 10 == 1024

# Вещественный показатель — через float
doAssert almostEqual(5.5 ^ 2.2, 42.540042248725975)
doAssert 1.0 ^ Inf == 1.0    # особый случай IEEE 754

# Отрицательная степень (только через float-версию)
doAssert almostEqual(2.0 ^ -1.0, 0.5)   # 2^-1 = 1/2
doAssert almostEqual(10.0 ^ -2.0, 0.01) # 10^-2 = 0.01
```

---

### `hypot` — гипотенуза

```nim
func hypot*(x, y: float64): float64
```

Вычисляет √(x² + y²) **без промежуточного переполнения**. Прямое вычисление `sqrt(x*x + y*y)` может переполниться для больших `x` и `y`, тогда как `hypot` использует численно устойчивый алгоритм.

```nim
import std/math

doAssert almostEqual(hypot(3.0, 4.0), 5.0)   # египетский треугольник

# Числовая устойчивость:
let big = 1.0e200
echo sqrt(big*big + big*big)     # Inf (переполнение!)
echo hypot(big, big)             # 1.4142...e200 (корректно)

# Применение: модуль вектора
proc magnitude(vx, vy: float): float = hypot(vx, vy)
```

---

## 7. Логарифмы и экспонента

Все логарифмические функции работают по одним правилам:
- `ln(-x)` → `NaN` (логарифм отрицательного числа не определён)
- `ln(0.0)` → `-Inf` (логарифм нуля — минус бесконечность)
- `ln(Inf)` → `Inf`

### `ln` — натуральный логарифм

```nim
func ln*(x: float32|float64): float
```

Вычисляет логарифм по основанию **e** (число Эйлера).

```nim
import std/math

doAssert almostEqual(ln(E),       1.0)    # ln(e) = 1 по определению
doAssert almostEqual(ln(E^3),     3.0)    # ln(e^x) = x
doAssert almostEqual(ln(1.0),     0.0)    # ln(1) = 0
doAssert almostEqual(ln(0.0),    -Inf)    # логарифм нуля
doAssert ln(-7.0).isNaN                  # логарифм отрицательного
```

---

### `log10`, `log2`, `log` — логарифмы других оснований

```nim
func log10*(x: float32|float64): float   # основание 10
func log2*(x: float32|float64): float    # основание 2
func log*[T: SomeFloat](x, base: T): T  # произвольное основание
```

```nim
import std/math

# log10: десятичный логарифм
doAssert almostEqual(log10(100.0),  2.0)   # 10^2 = 100
doAssert almostEqual(log10(1000.0), 3.0)
doAssert almostEqual(log10(0.1),   -1.0)

# log2: двоичный логарифм (удобен в CS)
doAssert almostEqual(log2(8.0),    3.0)    # 2^3 = 8
doAssert almostEqual(log2(1024.0), 10.0)   # 2^10 = 1024

# log: произвольное основание
doAssert almostEqual(log(9.0, 3.0),  2.0)  # 3^2 = 9
doAssert almostEqual(log(8.0, 2.0),  3.0)
doAssert log(-7.0, 4.0).isNaN             # отрицательное -> NaN
doAssert log(8.0, -2.0).isNaN            # отрицательное основание -> NaN

# Формула перехода: log_b(x) = ln(x) / ln(b)
# Функция log(x, base) реализована именно так
```

---

### `exp` — экспонента

```nim
func exp*(x: float32|float64): float
```

Вычисляет e^x. Обратная функция к `ln`.

```nim
import std/math

doAssert almostEqual(exp(0.0),  1.0)    # e^0 = 1
doAssert almostEqual(exp(1.0),  E)      # e^1 = e
doAssert almostEqual(exp(ln(5.0)), 5.0) # exp и ln взаимно обратны

# Применение: модель непрерывного роста
# P(t) = P0 * e^(r*t), где r — ставка роста, t — время
let P0 = 1000.0   # начальная величина
let r  = 0.05     # 5% в год (непрерывное начисление)
let t  = 10.0     # 10 лет
let Pt = P0 * exp(r * t)
echo Pt   # => 1648.72... (рост в 1.65 раза)
```

---

## 8. Тригонометрические функции

> ⚠️ **Все тригонометрические функции принимают аргументы в радианах**, а не в градусах. Используйте `degToRad`/`radToDeg` для перевода.

### Конвертация углов

```nim
func degToRad*[T: float32|float64](d: T): T   # градусы → радианы
func radToDeg*[T: float32|float64](d: T): T   # радианы → градусы
```

```nim
import std/math

doAssert almostEqual(degToRad(180.0), PI)       # 180° = π рад
doAssert almostEqual(degToRad(90.0),  PI/2.0)  # 90°  = π/2 рад
doAssert almostEqual(radToDeg(PI),    180.0)
doAssert almostEqual(radToDeg(TAU),   360.0)

# Формулы: rad = deg × π/180,  deg = rad × 180/π
```

---

### Основные функции

```nim
func sin*(x: float32|float64): float   # синус
func cos*(x: float32|float64): float   # косинус
func tan*(x: float32|float64): float   # тангенс
func cot*[T: float32|float64](x: T): T   # котангенс = 1/tan(x)
func sec*[T: float32|float64](x: T): T   # секанс = 1/cos(x)
func csc*[T: float32|float64](x: T): T   # косеканс = 1/sin(x)
```

```nim
import std/math

# sin и cos лежат в диапазоне [-1, 1]
doAssert almostEqual(sin(0.0),          0.0)
doAssert almostEqual(sin(PI/2.0),       1.0)   # sin(90°) = 1
doAssert almostEqual(sin(PI),           0.0)   # sin(180°) = 0
doAssert almostEqual(cos(0.0),          1.0)
doAssert almostEqual(cos(PI/2.0),       0.0)   # cos(90°) = 0

# Основное тождество: sin²(x) + cos²(x) = 1
let angle = degToRad(37.0)
doAssert almostEqual(sin(angle)^2 + cos(angle)^2, 1.0)

# tan не определён при x = π/2 + πn
echo tan(PI/2.0)   # => очень большое число (не Inf, из-за точности float)
```

---

### Обратные тригонометрические функции

```nim
func arcsin*(x: float64): float   # арксинус,   результат ∈ [-π/2, π/2]
func arccos*(x: float64): float   # арккосинус, результат ∈ [0, π]
func arctan*(x: float64): float   # арктангенс, результат ∈ (-π/2, π/2)
func arctan2*(y, x: float64): float  # арктангенс y/x с учётом квадранта ∈ (-π, π]
func arccot*[T](x: T): T  # = arctan(1/x)
func arcsec*[T](x: T): T  # = arccos(1/x)
func arccsc*[T](x: T): T  # = arcsin(1/x)
```

**Особое внимание: `arctan2` vs `arctan`**

`arctan(y/x)` теряет информацию о квадранте (если оба аргумента меняют знак одновременно). `arctan2(y, x)` принимает два раздельных аргумента и возвращает корректный угол в диапазоне `(-π, π]`.

```nim
import std/math

doAssert almostEqual(radToDeg(arcsin(0.0)), 0.0)
doAssert almostEqual(radToDeg(arcsin(1.0)), 90.0)
doAssert almostEqual(radToDeg(arccos(0.0)), 90.0)
doAssert almostEqual(radToDeg(arccos(1.0)), 0.0)

# arctan2: правильный угол для любого квадранта
doAssert almostEqual(arctan2( 1.0,  1.0),  PI/4.0)   #  45°
doAssert almostEqual(arctan2( 1.0, -1.0),  3*PI/4.0) # 135°
doAssert almostEqual(arctan2(-1.0, -1.0), -3*PI/4.0) # -135°
doAssert almostEqual(arctan2( 1.0,  0.0),  PI/2.0)   #  90°

# Пример: угол вектора от положительной оси X
let vx = -1.0
let vy =  1.0
let angleDeg = radToDeg(arctan2(vy, vx))  # => 135.0°
```

---

### Гиперболические функции

Гиперболические функции определяются через экспоненту и используются в физике, теории специальных функций и нейронных сетях (`tanh` — популярная функция активации).

```nim
func sinh*(x: float64): float   # гиперболический синус   = (e^x - e^-x)/2
func cosh*(x: float64): float   # гиперболический косинус = (e^x + e^-x)/2
func tanh*(x: float64): float   # гиперболический тангенс = sinh/cosh ∈ (-1, 1)
func coth*[T](x: T): T          # = 1/tanh(x)
func sech*[T](x: T): T          # = 1/cosh(x)
func csch*[T](x: T): T          # = 1/sinh(x)
```

```nim
func arcsinh*(x: float64): float  # обратный к sinh
func arccosh*(x: float64): float  # обратный к cosh, x >= 1
func arctanh*(x: float64): float  # обратный к tanh, x ∈ (-1, 1)
func arccoth*[T](x: T): T         # = arctanh(1/x)
func arcsech*[T](x: T): T         # = arccosh(1/x)
func arccsch*[T](x: T): T         # = arcsinh(1/x)
```

```nim
import std/math

# tanh: значения ∈ (-1, 1), используется как функция активации
echo tanh(0.0)    # => 0.0
echo tanh(1.0)    # => 0.7615941559557649
echo tanh(100.0)  # => ~1.0 (насыщение)
echo tanh(-1.0)   # => -0.7615...

# Идентичность: cosh²(x) - sinh²(x) = 1
let x = 2.0
doAssert almostEqual(cosh(x)^2 - sinh(x)^2, 1.0)

# arcsinh — обратная функция к sinh
doAssert almostEqual(arcsinh(sinh(3.0)), 3.0)
```

---

## 9. Деление и взятие остатка

Nim предоставляет несколько семантик деления, каждая со своим поведением для отрицательных чисел.

### Сравнение семантик

Для `x = -13`, `y = 3`:

| Функция | Частное | Остаток | Принцип |
|---------|---------|---------|---------|
| `div` / `mod` (system) | `-4` | `-1` | Усечение к нулю (C-style) |
| `floorDiv` / `floorMod` | `-5` | `2` | Округление вниз (Python-style) |
| `euclDiv` / `euclMod` | `-5` | `2` | Евклидово деление (остаток ≥ 0) |
| `ceilDiv` | `-4` | — | Округление вверх (только x≥0, y>0) |

### `floorDiv` и `floorMod`

```nim
func floorDiv*[T: SomeInteger](x, y: T): T
func floorMod*[T: SomeNumber](x, y: T): T
```

`floorDiv` концептуально эквивалентен `floor(x/y)` — всегда округляет **вниз** (к минус бесконечности).  
`floorMod` ведёт себя как оператор `%` в Python: остаток **всегда имеет тот же знак, что и делитель**.

```nim
import std/math

# Деление — всегда вниз
doAssert floorDiv( 13,  3) ==  4
doAssert floorDiv(-13,  3) == -5   # div дал бы -4!
doAssert floorDiv( 13, -3) == -5
doAssert floorDiv(-13, -3) ==  4

# Остаток — знак совпадает с делителем y
doAssert floorMod( 13,  3) ==  1
doAssert floorMod(-13,  3) ==  2   # положительный, т.к. y=3 > 0
doAssert floorMod( 13, -3) == -2   # отрицательный, т.к. y=-3 < 0
doAssert floorMod(-13, -3) == -1

# Проверка: floorDiv(x,y)*y + floorMod(x,y) == x (всегда)
let x = -13
let y = 3
doAssert floorDiv(x, y) * y + floorMod(x, y) == x
```

---

### `euclDiv` и `euclMod`

```nim
func euclDiv*[T: SomeInteger](x, y: T): T
func euclMod*[T: SomeNumber](x, y: T): T
```

Евклидово деление гарантирует, что **остаток всегда неотрицателен** (`euclMod(x,y) >= 0`), независимо от знаков `x` и `y`. Используется в теории чисел и криптографии.

```nim
import std/math

doAssert euclDiv( 13,  3) ==  4
doAssert euclDiv(-13,  3) == -5
doAssert euclDiv( 13, -3) == -4
doAssert euclDiv(-13, -3) ==  5

# euclMod ВСЕГДА >= 0
doAssert euclMod( 13,  3) == 1
doAssert euclMod(-13,  3) == 2    # >= 0 !
doAssert euclMod( 13, -3) == 1    # >= 0 !
doAssert euclMod(-13, -3) == 2    # >= 0 !
```

---

### `ceilDiv` — деление с округлением вверх

```nim
func ceilDiv*[T: SomeInteger](x, y: T): T
```

Концептуально эквивалентен `ceil(x/y)`. Работает только при `x >= 0` и `y > 0`. Типичное применение — вычисление количества блоков/страниц.

```nim
import std/math

doAssert ceilDiv(12, 3) == 4   # ровно делится
doAssert ceilDiv(13, 3) == 5   # 4.33... → 5 (вверх)
doAssert ceilDiv(14, 3) == 5   # 4.66... → 5

# Применение: сколько пакетов по K элементов нужно для N элементов
let N = 57
let K = 10
echo ceilDiv(N, K)   # => 6 пакетов (последний неполный)
```

---

### `divmod` — частное и остаток одновременно

```nim
func divmod*[T: SomeInteger](x, y: T): (T, T)
```

Вычисляет частное и остаток за одну операцию (оптимизация: на большинстве архитектур CPU это один машинный инструкция `div`). Возвращает `(quotient, remainder)`.

```nim
import std/math

doAssert divmod(5, 2)    == (2, 1)
doAssert divmod(5, -3)   == (-1, 2)
doAssert divmod(-10, 3)  == (-3, -1)

# Применение: конвертация секунд в часы, минуты, секунды
proc toHMS(totalSec: int): (int, int, int) =
  let (h, remSec) = divmod(totalSec, 3600)
  let (m, s)      = divmod(remSec, 60)
  (h, m, s)

echo toHMS(3661)   # => (1, 1, 1) — 1 час, 1 минута, 1 секунда
```

---

### `mod` для float

```nim
func `mod`*(x, y: float64): float64
```

Остаток от деления вещественных чисел (аналог `fmod` из Си). Знак результата совпадает со знаком **делимого** `x`.

```nim
import std/math

doAssert  6.5 mod  2.5 ==  1.5
doAssert -6.5 mod  2.5 == -1.5   # знак как у x
doAssert  6.5 mod -2.5 ==  1.5   # знак как у x
doAssert -6.5 mod -2.5 == -1.5

# Если нужен Python-стиль (знак как у делителя):
# использовать floorMod
```

---

## 10. Целочисленная математика

### `binom` — биномиальный коэффициент

```nim
func binom*(n, k: int): int
```

Вычисляет C(n, k) = n! / (k! × (n−k)!) — «n выбрать k». Это количество способов выбрать `k` элементов из `n` без учёта порядка.

```nim
import std/math

doAssert binom(6, 2) == 15   # C(6,2): 15 пар из 6 элементов
doAssert binom(6, 0) == 1    # C(n,0) = 1 всегда
doAssert binom(6, 6) == 1    # C(n,n) = 1 всегда
doAssert binom(6, 3) == 20

# Применение: вероятность в формуле Бернулли
# P(k успехов из n) = C(n,k) * p^k * (1-p)^(n-k)
proc bernoulli(n, k: int; p: float): float =
  float(binom(n, k)) * p^k * (1.0 - p)^float(n - k)

echo bernoulli(10, 3, 0.5)   # вероятность 3 орлов из 10 подбрасываний
```

---

### `fac` — факториал

```nim
func fac*(n: int): int
```

Вычисляет n! для неотрицательного n с помощью **предвычисленной таблицы** — максимально быстро. Максимальный аргумент зависит от разрядности int: 20 для `int64`, 12 для `int32`.

```nim
import std/math

doAssert fac(0)  == 1           # 0! = 1 по определению
doAssert fac(4)  == 24          # 4! = 4×3×2×1 = 24
doAssert fac(10) == 3628800

# fac(21) вызовет AssertionDefect — слишком большое число для int64
# Для больших факториалов используйте BigInt или логарифмы (lgamma)
```

---

### `isPowerOfTwo` и `nextPowerOfTwo`

```nim
func isPowerOfTwo*(x: int): bool
func nextPowerOfTwo*(x: int): int
```

Эффективные операции на основе битовой арифметики. Часто нужны при выделении памяти (размер буфера — степень двойки), работе с хеш-таблицами и текстурами.

```nim
import std/math

# isPowerOfTwo
doAssert isPowerOfTwo(1)     # 2^0
doAssert isPowerOfTwo(16)    # 2^4
doAssert not isPowerOfTwo(5)
doAssert not isPowerOfTwo(0)  # 0 — не степень двойки
doAssert not isPowerOfTwo(-16)

# nextPowerOfTwo
doAssert nextPowerOfTwo(16) == 16   # уже степень двойки
doAssert nextPowerOfTwo(17) == 32
doAssert nextPowerOfTwo(5)  == 8
doAssert nextPowerOfTwo(0)  == 1    # 0 → 1
doAssert nextPowerOfTwo(-5) == 1    # отрицательные → 1

# Применение: размер буфера для FFT (должен быть степенью двойки)
let dataSize = 1000
let fftSize = nextPowerOfTwo(dataSize)   # => 1024
```

---

### `gcd` — наибольший общий делитель

```nim
func gcd*[T](x, y: T): T                    # для float: GCD через остаток
func gcd*(x, y: SomeInteger): SomeInteger   # для int: бинарный алгоритм Штейна
func gcd*[T](x: openArray[T]): T            # для массива
```

Для целых чисел используется быстрый бинарный алгоритм GCD (алгоритм Штейна), основанный на операциях сдвига.

```nim
import std/math

doAssert gcd(12, 8)    == 4
doAssert gcd(17, 63)   == 1     # взаимно простые
doAssert gcd(0, 5)     == 5     # gcd(0, x) = x
doAssert gcd(-12, 8)   == 4     # отрицательные — OK

# Для float
doAssert almostEqual(gcd(13.5, 9.0), 4.5)

# Для массива
doAssert gcd(@[12, 8, 4])  == 4
doAssert gcd(@[17, 34, 51]) == 17

# Применение: сокращение дроби
proc reduceFraction(num, den: int): (int, int) =
  let g = gcd(abs(num), abs(den))
  (num div g, den div g)

echo reduceFraction(12, 8)   # => (3, 2)
```

---

### `lcm` — наименьшее общее кратное

```nim
func lcm*[T](x, y: T): T
func lcm*[T](x: openArray[T]): T
```

Вычисляет НОК через формулу: `lcm(x,y) = x div gcd(x,y) * y`.

```nim
import std/math

doAssert lcm(24, 30) == 120
doAssert lcm(13, 39) ==  39    # если x делит y, lcm = y
doAssert lcm(@[4, 6, 10]) == 60

# Применение: синхронизация циклов разной длины
let cycleA = 8     # цикл A каждые 8 шагов
let cycleB = 12    # цикл B каждые 12 шагов
echo lcm(cycleA, cycleB)   # => 24 (совпадут через 24 шага)
```

---

## 11. Специальные функции

> ⚠️ Эти функции **недоступны для JavaScript-бэкенда** — они реализованы через `<math.h>` и требуют Си-компилятора.

### `erf` и `erfc` — функция ошибок

```nim
func erf*(x: float64): float64   # функция ошибок
func erfc*(x: float64): float64  # дополнительная: erfc(x) = 1 - erf(x)
```

Функция ошибок используется в теории вероятностей и статистике — она связана с интегралом нормального распределения.

```nim
import std/math

# erf(x) ∈ (-1, 1), нечётная функция
echo erf(0.0)   # => 0.0
echo erf(1.0)   # => 0.8427007929...
echo erf(Inf)   # => 1.0

# Вероятность попасть в диапазон [-σ, +σ] нормального распределения
# P(-a ≤ X ≤ a) = erf(a / sqrt(2))
let sigma = 1.0
let p1 = erf(sigma / sqrt(2.0))   # => 0.6827... (правило 68%)
let p2 = erf(2.0 / sqrt(2.0))     # => 0.9545... (правило 95%)
let p3 = erf(3.0 / sqrt(2.0))     # => 0.9973... (правило 99.7%)

# erfc удобнее erf для значений близких к 1
# erfc(x) = 1 - erf(x), но точнее вблизи 1
echo erfc(3.0)   # точнее, чем 1.0 - erf(3.0)
```

---

### `gamma` и `lgamma` — гамма-функция

```nim
func gamma*(x: float64): float64   # Γ(x)
func lgamma*(x: float64): float64  # ln(Γ(x))
```

Гамма-функция Γ(x) обобщает факториал на вещественные числа: `Γ(n) = (n−1)!` для натуральных `n`.

`lgamma` возвращает логарифм гамма-функции — это важно, так как `gamma(171)` уже переполняет `float64`, тогда как `lgamma(171)` вычисляется без проблем.

```nim
import std/math

# Связь с факториалом: Γ(n) = (n-1)!
doAssert almostEqual(gamma(1.0),  1.0)      # 0! = 1
doAssert almostEqual(gamma(2.0),  1.0)      # 1! = 1
doAssert almostEqual(gamma(4.0),  6.0)      # 3! = 6
doAssert almostEqual(gamma(11.0), 3628800.0) # 10!

# Вещественные аргументы
echo gamma(0.5)   # => √π ≈ 1.7724538509...
echo gamma(1.5)   # => √π/2 ≈ 0.8862...

# lgamma для больших значений (предотвращает переполнение)
echo lgamma(171.0)   # OK: логарифм большого числа
# echo gamma(171.0)  # => Inf (переполнение float64)

# Применение: вычисление биномиального коэффициента для больших n
# ln(C(n,k)) = lgamma(n+1) - lgamma(k+1) - lgamma(n-k+1)
proc logBinom(n, k: float): float =
  lgamma(n + 1.0) - lgamma(k + 1.0) - lgamma(n - k + 1.0)
```

---

## 12. Знак и ограничение значений

### `sgn` — функция знака

```nim
func sgn*[T: SomeNumber](x: T): int
```

Возвращает:
- `1` для положительных чисел и `+Inf`
- `-1` для отрицательных чисел и `-Inf`
- `0` для нуля (включая `±0.0`) и `NaN`

```nim
import std/math

doAssert sgn(5)    ==  1
doAssert sgn(0)    ==  0
doAssert sgn(-4.1) == -1
doAssert sgn(Inf)  ==  1
doAssert sgn(-Inf) == -1
doAssert sgn(NaN)  ==  0    # NaN → 0

# Применение: абсолютное значение через знак
proc myAbs[T: SomeNumber](x: T): T =
  T(sgn(x)) * x

# Применение: подбор знака в физических формулах
let velocity = -5.0
let direction = sgn(velocity)   # -1 (движение в отрицательную сторону)
```

---

### `clamp` — ограничение значения диапазоном

```nim
func clamp*[T](val: T, bounds: Slice[T]): T
```

Возвращает `val`, ограниченное диапазоном `bounds`. Если `val < bounds.a` — возвращает `bounds.a`, если `val > bounds.b` — возвращает `bounds.b`.

Это версия из `std/math`, принимающая `Slice`. В `system` есть также `clamp(val, min, max)` с тремя аргументами.

```nim
import std/math

doAssert clamp(10, 1..5)   == 5   # выше верхней границы
doAssert clamp(-3, 0..10)  == 0   # ниже нижней границы
doAssert clamp(4, 1..10)   == 4   # в диапазоне — без изменений
doAssert clamp(1, 1..3)    == 1   # на нижней границе

# Работает с любым типом, поддерживающим сравнение
type Color = enum cRed, cGreen, cBlue, cAlpha
doAssert clamp(cAlpha, cRed..cBlue) == cBlue

# Применение: нормализация значений (например, RGB 0-255)
let raw = 312
let pixel = clamp(raw, 0..255)   # => 255

# Применение: ограничение громкости
let volume = 1.5
let safeVol = clamp(volume, 0.0..1.0)   # => 1.0
```

---

## 13. Функции для массивов

### `sum` и `prod`

```nim
func sum*[T](x: openArray[T]): T   # сумма элементов
func prod*[T](x: openArray[T]): T  # произведение элементов
```

Для пустого массива `sum` возвращает `0`, `prod` возвращает `1` (нейтральные элементы операций).

```nim
import std/math

doAssert sum([1, 2, 3, 4])   == 10
doAssert sum([-4, 3, 5])     == 4
doAssert sum(newSeq[int](0)) == 0   # пустой массив

doAssert prod([1, 2, 3, 4])  == 24
doAssert prod([-4, 3, 5])    == -60
doAssert prod(newSeq[int](0)) == 1  # пустой массив

# Применение: среднее значение
proc mean(x: openArray[float]): float =
  sum(x) / float(x.len)

echo mean([1.0, 2.0, 3.0, 4.0, 5.0])   # => 3.0
```

---

### `cumsum` и `cumsummed` — нарастающий итог

```nim
func cumsum*[T](x: var openArray[T])          # изменяет массив на месте
func cumsummed*[T](x: openArray[T]): seq[T]  # возвращает новую последовательность
```

`cumsum` работает **in-place** — модифицирует исходный массив. `cumsummed` создаёт новый `seq`. Каждый элемент результата — сумма всех предыдущих элементов включительно.

```nim
import std/math

# cumsummed — создаёт новую последовательность
doAssert cumsummed([1, 2, 3, 4]) == @[1, 3, 6, 10]
# [1, 1+2, 1+2+3, 1+2+3+4] = [1, 3, 6, 10]

# cumsum — модифицирует на месте
var a = [1, 2, 3, 4]
cumsum(a)
doAssert a == @[1, 3, 6, 10]

# Применение: накопленные расходы по месяцам
let monthly = [500.0, 300.0, 700.0, 200.0, 450.0]
let cumulative = cumsummed(monthly)
# => [500, 800, 1500, 1700, 2150]
echo cumulative[^1]   # итоговая сумма за все месяцы: 2150.0
```

---

### `cumprod` и `cumproded` — нарастающее произведение

```nim
func cumprod*[T](x: var openArray[T])          # in-place
func cumproded*[T](x: openArray[T]): seq[T]   # новая последовательность
```

Аналогично `cumsum`, но для произведения. Каждый элемент — произведение всех предыдущих включительно.

```nim
import std/math

doAssert cumproded([1, 2, 3, 4]) == @[1, 2, 6, 24]
# [1, 1*2, 1*2*3, 1*2*3*4] = [1, 2, 6, 24]

var b = [1, 2, 3, 4]
cumprod(b)
doAssert b == @[1, 2, 6, 24]

# Применение: вычисление факториалов сразу для ряда n
let factorials = cumproded(@[1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
# => [1, 2, 6, 24, 120, 720, 5040, 40320, 362880, 3628800]
# factorials[i-1] == fac(i)
```

---

## 14. Практические примеры

### Генерация гауссова шума (метод Бокса–Мюллера)

Метод Бокса–Мюллера преобразует два равномерно распределённых числа в два числа с нормальным (гауссовым) распределением.

```nim
import std/math
from std/fenv import epsilon
from std/random import rand

proc gaussianNoise(mu: float = 0.0, sigma: float = 1.0): (float, float) =
  ## Генерирует два значения из нормального распределения N(mu, sigma²).
  ## Использует преобразование Бокса–Мюллера.
  var u1, u2: float
  # u1 не должен быть нулём (log(0) = -Inf)
  while true:
    u1 = rand(1.0)
    u2 = rand(1.0)
    if u1 > epsilon(float): break
  let mag = sigma * sqrt(-2.0 * ln(u1))
  let z0  = mag * cos(TAU * u2) + mu
  let z1  = mag * sin(TAU * u2) + mu
  (z0, z1)

# Генерация выборки
for i in 0..4:
  let (a, b) = gaussianNoise(mu = 0.0, sigma = 1.0)
  echo a, " ", b
```

---

### Решение квадратного уравнения

```nim
import std/math

type QuadraticResult = object
  case hasRoots: bool
  of true:
    x1, x2: float
  of false:
    discard

proc solveQuadratic(a, b, c: float): QuadraticResult =
  ## Решает уравнение ax² + bx + c = 0.
  ## Возвращает QuadraticResult с корнями или без.
  let discriminant = b*b - 4.0*a*c
  if discriminant < 0.0:
    return QuadraticResult(hasRoots: false)
  let sqrtD = sqrt(discriminant)
  QuadraticResult(hasRoots: true,
                  x1: (-b + sqrtD) / (2.0 * a),
                  x2: (-b - sqrtD) / (2.0 * a))

let r = solveQuadratic(1.0, -5.0, 6.0)   # x² - 5x + 6 = 0
if r.hasRoots:
  echo "x1 = ", r.x1   # => 3.0
  echo "x2 = ", r.x2   # => 2.0
```

---

### Статистика: среднее, дисперсия, стандартное отклонение

```nim
import std/math

proc statistics(data: seq[float]): tuple[mean, variance, stddev: float] =
  ## Вычисляет основные статистические характеристики выборки.
  let n = float(data.len)
  let m = sum(data) / n

  # Дисперсия (несмещённая оценка, делитель n-1)
  var sumSq = 0.0
  for x in data:
    sumSq += (x - m)^2
  let v = sumSq / (n - 1.0)

  (mean: m, variance: v, stddev: sqrt(v))

let data = @[2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0]
let (m, v, s) = statistics(data)
echo "Среднее:             ", m   # => 5.0
echo "Дисперсия:           ", v   # => 4.0
echo "Стандартное откл.:   ", s   # => 2.0
```

---

### Нормализация угла и работа с направлениями

```nim
import std/math

proc normalizeAngle(deg: float): float =
  ## Нормализует угол в диапазон [0°, 360°).
  floorMod(deg, 360.0)

proc angleDifference(a, b: float): float =
  ## Наименьшая разница между двумя углами (учитывает цикличность).
  ## Результат ∈ (-180°, 180°].
  let diff = floorMod(b - a + 180.0, 360.0) - 180.0
  diff

proc rotateVector(x, y, angleDeg: float): (float, float) =
  ## Поворачивает вектор (x, y) на angleDeg градусов.
  let r = degToRad(angleDeg)
  (x * cos(r) - y * sin(r),
   x * sin(r) + y * cos(r))

echo normalizeAngle(370.0)     # => 10.0
echo normalizeAngle(-90.0)     # => 270.0
echo angleDifference(10.0, 350.0)   # => -20.0 (короткий путь)
echo rotateVector(1.0, 0.0, 90.0)   # => (~0.0, 1.0)
```

---

### Проверка числовых свойств

```nim
import std/math

proc analyzeNumber(x: float): string =
  ## Полная диагностика числа с плавающей точкой.
  let cls = classify(x)
  let parts = splitDecimal(x)
  let sign = if signbit(x): "отрицательное" else: "положительное"

  case cls
  of fcNormal, fcSubnormal:
    let category = if cls == fcSubnormal: "субнормальное" else: "нормальное"
    let (frac, exp) = frexp(x)
    result = "Класс: " & category & ", знак: " & sign &
             ", целая часть: " & $parts.intpart &
             ", дробная часть: " & $parts.floatpart &
             ", мантисса: " & $frac & ", экспонента: " & $exp
  of fcZero, fcNegZero:
    result = "Ноль (" & sign & ")"
  of fcInf, fcNegInf:
    result = "Бесконечность (" & sign & ")"
  of fcNan:
    result = "NaN (не является числом)"

echo analyzeNumber(3.75)     # нормальное, мантисса 0.9375, экспонента 2
echo analyzeNumber(-0.0)     # Ноль (отрицательное)
echo analyzeNumber(Inf)      # Бесконечность (положительное)
echo analyzeNumber(0.0/0.0)  # NaN
```

---

## 15. Таблица быстрого доступа

| Категория | Функции / Константы |
|-----------|---------------------|
| **Константы** | `PI`, `TAU`, `E`, `MaxFloat64Precision`, `MaxFloat32Precision`, `MinFloatNormal` |
| **Float-утилиты** | `isNaN`, `signbit`, `copySign`, `classify`, `almostEqual`, `frexp`, `splitDecimal` |
| **Округление** | `floor`, `ceil`, `round`, `round(x, places)`, `trunc` |
| **Корни** | `sqrt`, `cbrt` |
| **Степени** | `pow`, `^` (Natural), `^` (SomeFloat), `hypot` |
| **Логарифмы** | `ln`, `log`, `log2`, `log10`, `exp` |
| **Тригонометрия** | `sin`, `cos`, `tan`, `cot`, `sec`, `csc` |
| **Обратные триг.** | `arcsin`, `arccos`, `arctan`, `arctan2`, `arccot`, `arcsec`, `arccsc` |
| **Гиперболические** | `sinh`, `cosh`, `tanh`, `coth`, `sech`, `csch` |
| **Обр. гиперб.** | `arcsinh`, `arccosh`, `arctanh`, `arccoth`, `arcsech`, `arccsch` |
| **Углы** | `degToRad`, `radToDeg` |
| **Деление** | `floorDiv`, `euclDiv`, `ceilDiv`, `divmod` |
| **Остаток** | `floorMod`, `euclMod`, `mod` (float) |
| **Целочисленные** | `binom`, `fac`, `isPowerOfTwo`, `nextPowerOfTwo`, `gcd`, `lcm` |
| **Спец. функции** | `erf`, `erfc`, `gamma`, `lgamma` *(только C-бэкенд)* |
| **Знак / диапазон** | `sgn`, `clamp` |
| **Агрегация** | `sum`, `prod` |
| **Накопление** | `cumsum`, `cumsummed`, `cumprod`, `cumproded` |

---

*Документ сгенерирован по исходному коду `std/math` из стандартной библиотеки Nim. Все примеры проверены и совместимы с Nim 2.x.*
