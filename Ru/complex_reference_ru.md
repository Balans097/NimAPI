# Справочник по модулю `complex`

> **Модуль:** `complex`  
> **Источник:** Стандартная библиотека Nim    
> **Назначение:** Тип комплексного числа, арифметические операторы, трансцендентные функции, конвертация между полярными и прямоугольными координатами, форматирование строк.

---

## Обзор

**Комплексное число** — это пара `(re, im)`, где `re` — вещественная часть, `im` — мнимая. Вместе они описывают точку (или вектор) на двумерной комплексной плоскости. Этот модуль даёт Nim полноценную поддержку этого математического объекта: обобщённый тип `Complex[T]`, работающий с `float32` и `float64`, полный набор арифметических операторов, свободно смешивающихся с обычными вещественными числами, стандартные трансцендентные функции, расширенные на комплексную область (тригонометрические, гиперболические, логарифмические, показательные, степени), и преобразования между прямоугольными `(re, im)` и полярными `(r, φ)` координатами.

Реализация полностью в виде `func` (без побочных эффектов). Внутренние проверки отключены для производительности.

---

## Типы

### `Complex[T]`

```nim
type Complex*[T: SomeFloat] = object
  re*, im*: T
```

Основной тип. `T` должен быть `float32` или `float64` (ограничен `SomeFloat`). Оба поля публичны — можно читать и писать `z.re` и `z.im` напрямую.

### `Complex64`

```nim
type Complex64* = Complex[float64]
```

Псевдоним для наиболее частого случая — 64-битной точности.

### `Complex32`

```nim
type Complex32* = Complex[float32]
```

Псевдоним для 32-битной точности — удобен в графике или обработке сигналов, где важен размер памяти.

---

## Экспортируемые символы

| Символ | Вид | Краткое описание |
|--------|-----|-----------------|
| `complex` | func | Обобщённый конструктор `(re, im)` |
| `complex32` | func | Конструктор для `Complex32` |
| `complex64` | func | Конструктор для `Complex64` |
| `im` | шаблон ×4 | Мнимая единица или мнимое число из вещественного |
| `==` | func | Точное равенство |
| `+`, `-`, `*`, `/` | func ×3 каждый | Арифметика: complex/complex, real/complex, complex/real |
| `+=`, `-=`, `*=`, `/=` | func | Арифметика на месте |
| `abs` | func | Модуль (расстояние от начала координат) |
| `abs2` | func | Квадрат модуля (быстрее, чем `abs^2`) |
| `sgn` | func | Единичный вектор направления (или 0) |
| `conjugate` | func | Комплексно-сопряжённое |
| `inv` | func | Мультипликативная обратная (1/z) |
| `sqrt` | func | Главный квадратный корень |
| `exp` | func | Показательная функция `e^z` |
| `ln` | func | Натуральный логарифм (главное значение) |
| `log10` | func | Логарифм по основанию 10 |
| `log2` | func | Логарифм по основанию 2 |
| `pow` | func ×2 | Степень: complex^complex, complex^real |
| `sin`, `cos`, `tan` | func | Тригонометрические функции |
| `arcsin`, `arccos`, `arctan` | func | Обратные тригонометрические |
| `cot`, `sec`, `csc` | func | Котангенс, секанс, косеканс |
| `arccot`, `arcsec`, `arccsc` | func | Обратные кот/сек/косек |
| `sinh`, `cosh`, `tanh` | func | Гиперболические функции |
| `arcsinh`, `arccosh`, `arctanh` | func | Обратные гиперболические |
| `coth`, `sech`, `csch` | func | Гиперб. кот/сек/косек |
| `arccoth`, `arcsech`, `arccsch` | func | Обратные гиперб. кот/сек/косек |
| `phase` | func | Угол в полярной форме (arctan2) |
| `polar` | func | Полярные координаты `(r, phi)` |
| `rect` | func | Комплексное из полярных `(r, phi)` |
| `almostEqual` | func | Приближённое равенство с допуском ULP |
| `$` | func | Строка `"(re, im)"` |
| `formatValue` | proc | Интеграция с `strformat` (поддержка макроса `&`) |

---

## Конструкторы

### `complex`

```nim
func complex*[T: SomeFloat](re: T; im: T = 0.0): Complex[T]
```

Обобщённый конструктор. `T` выводится из типа `re`. Мнимая часть по умолчанию `0.0`, поэтому `complex(3.0)` создаёт чисто вещественное число `3 + 0i`.

```nim
let z1 = complex(1.0, 2.0)          # 1 + 2i  (float64)
let z2 = complex(3.0)               # 3 + 0i
let z3 = complex(0.5'f32, 1.0'f32)  # float32 версия
```

---

### `complex32` и `complex64`

```nim
func complex32*(re: float32; im: float32 = 0.0): Complex32
func complex64*(re: float64; im: float64 = 0.0): Complex64
```

Типизированные версии для случаев, когда точность должна быть явной — особенно полезны, когда обобщённый `T` не выводится автоматически.

```nim
let a = complex32(1.5, -0.5)   # явный float32
let b = complex64(1.5, -0.5)   # явный float64
```

---

### `im` — Мнимая единица и мнимые числа

```nim
template im*(arg: typedesc[float32]): Complex32   # мнимая единица i (32-бит)
template im*(arg: typedesc[float64]): Complex64   # мнимая единица i (64-бит)
template im*(arg: float32): Complex32             # arg * i  (32-бит)
template im*(arg: float64): Complex64             # arg * i  (64-бит)
```

Четыре перегрузки позволяют писать литералы комплексных чисел естественно. Передача **типа** даёт чистую мнимую единицу `0 + 1i`. Передача **значения** даёт `0 + arg*i`.

```nim
let i64 = im(float64)      # 0 + 1i  (Complex64)
let z   = 3.0 + 4.0.im    # 3 + 4i  — почти математическая запись
let z2  = complex(2.0) + im(float64)   # 2 + 1i
```

Перегрузки, принимающие значения, позволяют писать `a + b.im`, что читается почти как математическая запись `a + bi`.

---

## Арифметические операторы

Все операторы обобщены по `[T: SomeFloat]`. Каждый из `+`, `-`, `*`, `/` имеет три перегрузки: `complex op complex`, `real op complex` и `complex op real`, поэтому никогда не нужно оборачивать вещественное число перед смешиванием с комплексным.

### `==`

```nim
func `==`*[T](x, y: Complex[T]): bool
```

Точное побитовое равенство обоих компонент. Для результатов вычислений с плавающей точкой предпочтительнее использовать `almostEqual`.

```nim
assert complex(1.0, 2.0) == complex(1.0, 2.0)
assert not (complex(1.0, 2.0) == complex(1.0, 2.000001))
```

---

### `+`  — Сложение

```nim
func `+`*[T](x: T; y: Complex[T]): Complex[T]      # вещественное + комплексное
func `+`*[T](x: Complex[T]; y: T): Complex[T]      # комплексное + вещественное
func `+`*[T](x, y: Complex[T]): Complex[T]         # комплексное + комплексное
```

Поканальное: `(a+bi) + (c+di) = (a+c) + (b+d)i`. Прибавление вещественного числа сдвигает только вещественную часть.

```nim
let z1 = complex(1.0, 2.0)
let z2 = complex(3.0, -4.0)
assert z1 + z2 == complex(4.0, -2.0)
assert z1 + 10.0 == complex(11.0, 2.0)
assert 10.0 + z1 == complex(11.0, 2.0)
```

---

### `-`  — Вычитание и унарный минус

```nim
func `-`*[T](z: Complex[T]): Complex[T]            # унарный: -(a+bi) = -a-bi
func `-`*[T](x: T; y: Complex[T]): Complex[T]
func `-`*[T](x: Complex[T]; y: T): Complex[T]
func `-`*[T](x, y: Complex[T]): Complex[T]
```

```nim
let z = complex(3.0, -4.0)
assert -z == complex(-3.0, 4.0)                     # унарный минус
assert complex(1.0, 2.0) - z == complex(-2.0, 6.0)
assert 10.0 - z == complex(7.0, 4.0)               # re=10-3, im=0-(-4)
```

---

### `*`  — Умножение

```nim
func `*`*[T](x: T; y: Complex[T]): Complex[T]
func `*`*[T](x: Complex[T]; y: T): Complex[T]
func `*`*[T](x, y: Complex[T]): Complex[T]
```

Комплексное умножение по формуле `(ac−bd) + (bc+ad)i`:

```nim
let z1 = complex(1.0, 2.0)
let z2 = complex(3.0, -4.0)
# (1·3 − 2·(-4)) + (2·3 + 1·(-4))i = 11 + 2i
assert z1 * z2 == complex(11.0, 2.0)
assert 2.0 * z1 == complex(2.0, 4.0)   # масштабирует оба компонента
```

---

### `/`  — Деление

```nim
func `/`*[T](x: Complex[T]; y: T): Complex[T]
func `/`*[T](x: T; y: Complex[T]): Complex[T]
func `/`*[T](x, y: Complex[T]): Complex[T]
```

Реализовано через `x * conjugate(y) / abs2(y)` — численно устойчивый алгоритм.

```nim
from std/math import almostEqual
let z1 = complex(1.0, 2.0)
let z2 = complex(3.0, -4.0)
assert almostEqual(z1 / z2, complex(-0.2, 0.4))
assert z1 / 2.0 == complex(0.5, 1.0)
```

---

### Операторы на месте: `+=`, `-=`, `*=`, `/=`

```nim
func `+=`*[T](x: var Complex[T]; y: Complex[T])
func `-=`*[T](x: var Complex[T]; y: Complex[T])
func `*=`*[T](x: var Complex[T]; y: Complex[T])
func `/=`*[T](x: var Complex[T]; y: Complex[T])
```

`*=` использует временную переменную, чтобы не затереть вещественную часть до вычисления мнимой.

```nim
var z = complex(1.0, 2.0)
z += complex(0.0, 1.0)   # z = 1 + 3i
z *= complex(2.0, 0.0)   # z = 2 + 6i
z -= complex(1.0, 1.0)   # z = 1 + 5i
z /= complex(0.0, 1.0)   # z = 5 - 1i
```

---

## Модуль и направление

### `abs`

```nim
func abs*[T](z: Complex[T]): T
```

Возвращает **модуль** числа `z` — евклидово расстояние от начала координат `√(re² + im²)`. Вычисляется через `hypot` — безопасно численно.

```nim
from std/math import almostEqual
let z = complex(3.0, 4.0)
assert almostEqual(abs(z), 5.0)         # египетский треугольник 3-4-5
assert abs(im(float64)) == 1.0          # мнимая единица лежит на единичной окружности
```

---

### `abs2`

```nim
func abs2*[T](z: Complex[T]): T
```

Возвращает **квадрат модуля** `re² + im²` без извлечения корня. Эффективнее, чем `abs(z) * abs(z)`, когда нужно только сравнить или использовать квадрат.

```nim
let z = complex(3.0, 4.0)
assert abs2(z) == 25.0   # 9 + 16, корень не нужен

# Быстрое сравнение расстояний без двух вызовов sqrt:
if abs2(z) > 100.0 * 100.0:
  echo "за пределами диска радиуса 100"
```

---

### `sgn`

```nim
func sgn*[T](z: Complex[T]): Complex[T]
```

Возвращает **единичный комплексный вектор** в направлении `z` — то есть `z / |z|`. Если `z = 0`, безопасно возвращает ноль.

Геометрически `sgn(z)` — это проекция `z` на единичную окружность: сохраняется только направление.

```nim
from std/math import almostEqual
let z = complex(3.0, 4.0)
let u = sgn(z)
assert almostEqual(abs(u), 1.0)
assert almostEqual(u, complex(0.6, 0.8))   # (3/5, 4/5)

assert sgn(complex(0.0, 0.0)) == complex(0.0, 0.0)
```

---

### `conjugate`

```nim
func conjugate*[T](z: Complex[T]): Complex[T]
```

Меняет знак мнимой части: `conjugate(a+bi) = a-bi`. Геометрически — отражение точки относительно вещественной оси.

Ключевое свойство: `z * conjugate(z) = abs2(z)` — всегда неотрицательное вещественное число. На этом основано деление комплексных чисел.

```nim
let z = complex(1.0, 2.0)
assert conjugate(z) == complex(1.0, -2.0)

let prod = z * conjugate(z)
assert prod.im == 0.0         # нет мнимой части
assert prod.re == abs2(z)     # = 1 + 4 = 5
```

---

### `inv`

```nim
func inv*[T](z: Complex[T]): Complex[T]
```

Возвращает **мультипликативно обратное** `1/z = conjugate(z) / abs2(z)`.

```nim
from std/math import almostEqual
let z = complex(1.0, 2.0)
assert almostEqual(inv(z) * z, complex(1.0, 0.0))   # z*(1/z) = 1

let i = im(float64)
assert inv(i) == complex(0.0, -1.0)   # 1/i = -i
```

---

## Трансцендентные функции

### `sqrt`

```nim
func sqrt*[T](z: Complex[T]): Complex[T]
```

**Главный квадратный корень**: результат имеет неотрицательную вещественную часть (при нулевой вещественной — неотрицательную мнимую). Алгоритм численно устойчив.

```nim
from std/math import almostEqual
let i = im(float64)

assert almostEqual(sqrt(complex(-1.0, 0.0)), i)              # √(-1) = i
assert almostEqual(sqrt(complex(0.0, 2.0)), complex(1.0, 1.0)) # √(2i) = 1+i

let z = complex(3.0, 4.0)
assert almostEqual(sqrt(z) * sqrt(z), z)   # кругооборот
```

---

### `exp`

```nim
func exp*[T](z: Complex[T]): Complex[T]
```

`e^z` по формуле Эйлера: `e^(a+bi) = e^a · (cos b + i·sin b)`. Вещественная часть управляет модулем, мнимая — углом.

```nim
from std/math import almostEqual, E, PI

# Тождество Эйлера: e^(iπ) = -1
let euler = exp(complex(0.0, PI))
assert almostEqual(euler.re, -1.0)
assert almostEqual(euler.im,  0.0)

# Чисто вещественный показатель — то же самое, что обычный exp:
assert almostEqual(exp(complex(1.0, 0.0)).re, E)
```

---

### `ln`

```nim
func ln*[T](z: Complex[T]): Complex[T]
```

**Главное значение** натурального логарифма: `ln(z) = ln(|z|) + i·arctan2(im, re)`. Мнимая часть (фаза) лежит в `(-π, π]`.

```nim
from std/math import almostEqual, PI

assert almostEqual(ln(exp(complex(1.0, 0.0))), complex(1.0, 0.0))

# ln(-1) = iπ  (главное значение)
let lnNeg1 = ln(complex(-1.0, 0.0))
assert almostEqual(lnNeg1.re, 0.0)
assert almostEqual(lnNeg1.im, PI)
```

---

### `log10` и `log2`

```nim
func log10*[T](z: Complex[T]): Complex[T]
func log2*[T](z: Complex[T]): Complex[T]
```

Логарифмы по основаниям 10 и 2, вычисляемые как `ln(z) / ln(основание)`. Ветвь разреза та же, что у `ln`.

```nim
from std/math import almostEqual
assert almostEqual(log10(complex(100.0, 0.0)), complex(2.0, 0.0))
assert almostEqual(log2(complex(8.0, 0.0)),    complex(3.0, 0.0))
```

---

### `pow`

```nim
func pow*[T](x, y: Complex[T]): Complex[T]    # x^y, оба комплексные
func pow*[T](x: Complex[T]; y: T): Complex[T] # x^y, y — вещественное
```

Общее комплексное возведение в степень с несколькими быстрыми ветвями: `0^0=1`, `x^1=x`, `x^(-1)=inv(x)`, `x^2=x*x`, `x^0.5=sqrt(x)`, вещественное основание с вещественным показателем сводится к скалярному `pow`, `e^y` использует `exp(y)`.

```nim
from std/math import almostEqual
assert almostEqual(pow(complex(2.0, 0.0), complex(3.0, 0.0)), complex(8.0, 0.0))

# i^i = e^(-π/2) ≈ 0.20788 — знаменито тем, что является вещественным числом:
assert almostEqual(pow(im(float64), im(float64)), complex(0.20788, 0.0), 1000)

assert almostEqual(pow(complex(4.0, 0.0), 0.5), complex(2.0, 0.0))  # перегрузка real
```

---

## Тригонометрические функции

Все круговые функции расширены на комплексную плоскость через аналитическое продолжение. При `z.im == 0` совпадают со своими вещественными аналогами.

### `sin` и `arcsin`

```nim
func sin*[T](z: Complex[T]): Complex[T]
func arcsin*[T](z: Complex[T]): Complex[T]
```

`sin(a+bi) = sin(a)·cosh(b) + i·cos(a)·sinh(b)`

```nim
from std/math import almostEqual, PI
assert almostEqual(sin(complex(PI/2, 0.0)), complex(1.0, 0.0))   # sin(π/2) = 1

let z = complex(1.0, 1.0)
assert almostEqual(arcsin(sin(z)), z)   # кругооборот
```

---

### `cos` и `arccos`

```nim
func cos*[T](z: Complex[T]): Complex[T]
func arccos*[T](z: Complex[T]): Complex[T]
```

`cos(a+bi) = cos(a)·cosh(b) − i·sin(a)·sinh(b)`

```nim
from std/math import almostEqual
assert almostEqual(cos(complex(0.0, 0.0)), complex(1.0, 0.0))
let z = complex(1.0, 1.0)
assert almostEqual(arccos(cos(z)), z)
```

---

### `tan`, `arctan`, `cot`, `arccot`, `sec`, `arcsec`, `csc`, `arccsc`

```nim
func tan*[T](z: Complex[T]): Complex[T]      # sin(z)/cos(z)
func arctan*[T](z: Complex[T]): Complex[T]
func cot*[T](z: Complex[T]): Complex[T]      # cos(z)/sin(z)
func arccot*[T](z: Complex[T]): Complex[T]
func sec*[T](z: Complex[T]): Complex[T]      # 1/cos(z)
func arcsec*[T](z: Complex[T]): Complex[T]
func csc*[T](z: Complex[T]): Complex[T]      # 1/sin(z)
func arccsc*[T](z: Complex[T]): Complex[T]
```

Полный набор круговых функций и их обратных. Все выводятся из `sin`/`cos` через стандартные тождества. Обратные функции реализованы через комплексный логарифм.

```nim
from std/math import almostEqual
let z = complex(0.5, 0.5)
assert almostEqual(arctan(tan(z)), z)
assert almostEqual(arccot(cot(z)), z)
assert almostEqual(arcsec(sec(z)), z)
assert almostEqual(arccsc(csc(z)), z)
```

---

## Гиперболические функции

### `sinh`, `cosh`, `tanh`, `coth`, `sech`, `csch`

```nim
func sinh*[T](z: Complex[T]): Complex[T]    # (exp(z) - exp(-z)) / 2
func cosh*[T](z: Complex[T]): Complex[T]    # (exp(z) + exp(-z)) / 2
func tanh*[T](z: Complex[T]): Complex[T]    # sinh(z)/cosh(z)
func coth*[T](z: Complex[T]): Complex[T]    # cosh(z)/sinh(z)
func sech*[T](z: Complex[T]): Complex[T]    # 2/(exp(z)+exp(-z))
func csch*[T](z: Complex[T]): Complex[T]    # 2/(exp(z)-exp(-z))
```

Связаны с круговыми через `sinh(iz) = i·sin(z)` и `cosh(iz) = cos(z)`.

```nim
from std/math import almostEqual
let z = complex(1.0, 1.0)
# Пифагорейское тождество: cosh² - sinh² = 1
assert almostEqual(cosh(z)*cosh(z) - sinh(z)*sinh(z), complex(1.0, 0.0))
```

---

### `arcsinh`, `arccosh`, `arctanh`, `arccoth`, `arcsech`, `arccsch`

```nim
func arcsinh*[T](z: Complex[T]): Complex[T]
func arccosh*[T](z: Complex[T]): Complex[T]
func arctanh*[T](z: Complex[T]): Complex[T]
func arccoth*[T](z: Complex[T]): Complex[T]
func arcsech*[T](z: Complex[T]): Complex[T]
func arccsch*[T](z: Complex[T]): Complex[T]
```

Все реализованы через комплексный логарифм, расширяя вещественные обратные гиперболические функции на всю комплексную плоскость.

```nim
from std/math import almostEqual
let z = complex(0.5, 0.5)
assert almostEqual(arcsinh(sinh(z)), z)
assert almostEqual(arctanh(tanh(z)), z)
```

---

## Полярные координаты

Комплексные числа имеют два эквивалентных представления. **Прямоугольное** `(re, im)` — точка на плоскости. **Полярное** `(r, φ)` — расстояние от начала координат и угол от положительной вещественной оси.

### `phase`

```nim
func phase*[T](z: Complex[T]): T
```

Возвращает **аргумент** (угол) `z` в радианах через `arctan2(z.im, z.re)`. Результат в диапазоне `(-π, π]`.

```nim
from std/math import almostEqual, PI
assert phase(complex(1.0, 0.0))  == 0.0           # положительная вещественная ось
assert almostEqual(phase(complex(0.0, 1.0)), PI/2) # 90°
assert almostEqual(phase(complex(-1.0, 0.0)), PI)  # 180°
assert almostEqual(phase(complex(1.0, 1.0)), PI/4) # 45°
```

---

### `polar`

```nim
func polar*[T](z: Complex[T]): tuple[r, phi: T]
```

Конвертирует в полярную форму, возвращая `(r: abs(z), phi: phase(z))`. Обратная операция — `rect`.

```nim
from std/math import almostEqual, sqrt, PI
let z = complex(1.0, 1.0)
let (r, phi) = z.polar
assert almostEqual(r, sqrt(2.0))
assert almostEqual(phi, PI/4)
```

---

### `rect`

```nim
func rect*[T](r, phi: T): Complex[T]
```

Создаёт комплексное число из полярных координат: `re = r·cos(φ)`, `im = r·sin(φ)`. Обратная операция к `polar`.

```nim
from std/math import almostEqual, PI

# Кругооборот:
let z = complex(3.0, 4.0)
let (r, phi) = z.polar
assert almostEqual(rect(r, phi), z)

# Единичный вектор при 90°:
assert almostEqual(rect(1.0, PI/2), complex(0.0, 1.0))
```

---

## Сравнение

### `almostEqual`

```nim
func almostEqual*[T: SomeFloat](x, y: Complex[T]; unitsInLastPlace: Natural = 4): bool
```

Проверяет **приближённое равенство** двух комплексных чисел, сравнивая вещественную и мнимую части по отдельности с допуском ULP (Units in the Last Place — единицы в последнем разряде). По умолчанию `unitsInLastPlace = 4` — подходит для большинства вычислений. Используйте вместо `==` везде, где есть цепочки арифметики с плавающей точкой.

`unitsInLastPlace = 0` — строгое равенство; большее значение допускает большее накопление ошибки.

```nim
from std/math import almostEqual

let z1 = complex(1.0, 2.0)
let z2 = complex(3.0, -4.0)

assert almostEqual(z1 + z2, complex(4.0, -2.0))
assert almostEqual(z1 * z2, complex(11.0, 2.0))
assert almostEqual(z1 / z2, complex(-0.2, 0.4))

# Строгое совпадение:
assert almostEqual(z1, z1, unitsInLastPlace = 0)
```

---

## Строковое представление

### `$`

```nim
func `$`*(z: Complex): string
```

Возвращает строку `"(re, im)"` с форматированием по умолчанию.

```nim
echo complex(1.0, 2.0)    # → (1.0, 2.0)
echo complex(0.0, -1.0)   # → (0.0, -1.0)
```

---

### `formatValue`

```nim
proc formatValue*(result: var string; value: Complex; specifier: string)
```

Интегрирует `Complex` с макросом `&` из `strformat`. Два режима:

- **Без `j` в спецификаторе:** форматирует как кортеж `"(re, im)"`, применяя числовой формат к каждому компоненту.
- **С `j` в спецификаторе:** форматирует как `"(A+Bj)"` (математическая/инженерная запись). Символ `j` заменяется внутренне на `g` (или ваш числовой формат) перед передачей в форматтер float.

```nim
import std/strformat

let z = complex(1.5, -2.75)

echo &"{z}"        # → (1.5, -2.75)
echo &"{z:.2f}"    # → (1.50, -2.75)
echo &"{z:j}"      # → (1.5-2.75j)
echo &"{z:.3fj}"   # → (1.500-2.750j)
echo &"{z:+.2fj}"  # → (+1.50-2.75j)
```

Суффикс `j` стандартен в Python и инженерной электрике для обозначения мнимой единицы (где `i` зарезервирован для тока).

---

## Типичные паттерны

### Поворот точки в 2D

Умножение на единичное комплексное число при угле `φ` поворачивает любое комплексное число на `φ`:

```nim
from std/math import PI, almostEqual
let point    = complex(1.0, 0.0)     # на вещественной оси
let rotation = rect(1.0, PI/2)       # поворот на 90°
let rotated  = point * rotation
assert almostEqual(rotated, complex(0.0, 1.0))
```

### Проверка i² = −1

```nim
from std/math import almostEqual
let i = im(float64)
assert almostEqual(i * i, complex(-1.0, 0.0))
```

### Поворотный множитель ДПФ

```nim
from std/math import PI
proc twiddleFactor(k, n: int): Complex64 =
  rect(1.0, -2.0 * PI * k.float / n.float)
```

### Накопление суммы на месте

```nim
var sum = complex(0.0, 0.0)
for z in myComplexSeq:
  sum += z
```

---

## Таблица ошибок

| Ситуация | Поведение | Примечание |
|----------|-----------|-----------|
| Деление на ноль (`z / complex(0,0)`) | `(Inf, NaN)` или аналог | Семантика IEEE 754, исключений нет |
| `sqrt(complex(0,0))` | Возвращает `complex(0,0)` | Явная проверка нуля в реализации |
| `sgn(complex(0,0))` | Возвращает `complex(0,0)` | Безопасно: нулевой вход → нулевой выход |
| `T` вне `SomeFloat` | Ошибка компиляции | Принимается только `float32`/`float64` |
| `almostEqual` с `unitsInLastPlace=0` | Строгое равенство | Эквивалентно `==` |
