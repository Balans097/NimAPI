# Nim BigInt и BigInt Random — Справочник по модулям

> **Модули:** `bigints` · `bigints/random`  
> **Установка:** `nimble install bigints`

Этот справочник охватывает каждый экспортируемый символ обоих модулей. `bigints` реализует целые числа произвольной точности; `bigints/random` добавляет генерацию случайных `BigInt` через стандартный `std/random`.

Разделы упорядочены тематически: создание, сравнения, арифметика, битовые операции, итераторы, теоретико-числовые функции, преобразование типов и, наконец, генерация случайных чисел.

---

## Содержание

1. [Тип: BigInt](#тип-bigint)
2. [Создание — initBigInt](#создание--initbigint)
3. [Операторы сравнения](#операторы-сравнения)
4. [Арифметические операторы](#арифметические-операторы)
5. [Инкремент и декремент](#инкремент-и-декремент)
6. [Преемник и предшественник](#преемник-и-предшественник)
7. [Возведение в степень и сдвиги](#возведение-в-степень-и-сдвиги)
8. [Побитовые операции](#побитовые-операции)
9. [Деление и остаток](#деление-и-остаток)
10. [Итераторы](#итераторы)
11. [Теория чисел](#теория-чисел)
12. [Преобразование типов](#преобразование-типов)
13. [Строковый ввод-вывод](#строковый-ввод-вывод)
14. [Генерация случайных чисел — bigints/random](#генерация-случайных-чисел--bigintsrandom)

---

## Тип: BigInt

```nim
type BigInt* = object
```

Знаковое целое число произвольной точности. Внутри хранит величину как последовательность 32-битных «лимбов» (цифр в системе счисления с основанием 2³²) и отдельный флаг знака. Практического верхнего предела для хранимого значения нет — объект растёт по мере необходимости.

**Побитовые операции** над `BigInt` трактуют отрицательные значения так, как будто они хранятся в дополнительном коде с бесконечным знаковым расширением. Это совпадает с поведением, ожидаемым программистами на Python, и делает операции `not`, `and`, `or` и `xor` согласованными с математической двоичной моделью.

---

## Создание — `initBigInt`

`initBigInt` — единственное семейство конструкторов для `BigInt`. Оно сильно перегружено: принимает все примитивные числовые типы Nim и строки.

---

### `initBigInt` из целочисленных скаляров

```nim
func initBigInt*[T: int8|int16|int32](val: T): BigInt
func initBigInt*[T: uint8|uint16|uint32](val: T): BigInt
func initBigInt*(val: int64): BigInt
func initBigInt*(val: uint64): BigInt
func initBigInt*(val: int): BigInt    # шаблон, ширина зависит от платформы
func initBigInt*(val: uint): BigInt   # шаблон, ширина зависит от платформы
func initBigInt*(val: BigInt): BigInt # копирование / идентичность
```

Конвертирует любой стандартный целочисленный тип Nim в `BigInt`. Знак для знаковых типов сохраняется корректно. Передача другого `BigInt` создаёт его копию.

```nim
let a = 42.initBigInt                          # из литерала int
let b = (-999_999_999).initBigInt
let c = 18446744073709551615'u64.initBigInt    # максимум uint64
let d = initBigInt(a)                          # явное копирование
```

---

### `initBigInt` из последовательности лимбов

```nim
func initBigInt*(vals: sink seq[uint32], isNegative = false): BigInt
```

Конструирует `BigInt` из последовательности 32-битных лимбов в **порядке от младшего к старшему** (индекс 0 — наименее значимый лимб). Флаг `isNegative` задаёт знак. Это низкоуровневый конструктор, полезный при взаимодействии с криптографическими или бинарными протоколами, уже работающими с 32-битными словами.

```nim
# 10 + 2 * 2^32 — то же самое, что целое число 10 + 2 shl 32
let a = @[10'u32, 2'u32].initBigInt
```

---

### `initBigInt` из строки

```nim
func initBigInt*(str: string, base: range[2..36] = 10): BigInt
```

Разбирает `str` как `BigInt` в заданном `base` (от 2 до 36). По умолчанию основание 10. Бросает `ValueError` при неверном или пустом вводе.

Допустимые форматы:
- Необязательный ведущий знак `+` или `-`.
- Цифры, допустимые для выбранного основания (`0–9`, `a–z` / `A–Z`).
- Подчёркивание `_` как разделитель групп цифр в любом месте, **кроме** начала и конца числа.

```nim
let a = initBigInt("1234")             # десятичное: 1234
let b = initBigInt("1234", base = 8)  # восьмеричное: 668
let c = initBigInt("ff", base = 16)   # шестнадцатеричное: 255
let d = initBigInt("-42")             # отрицательное: -42
let e = initBigInt("1_000_000")       # подчёркивания допустимы: 1000000
```

---

## Операторы сравнения

Все операторы сравнения возвращают `bool` и работают для любых комбинаций знаков и величин. Объявлены как `func` (чистые функции без побочных эффектов).

### `==`

```nim
func `==`*(a, b: BigInt): bool
```

Возвращает `true`, если `a` и `b` представляют одно и то же целое число.

```nim
assert 5.initBigInt == (3.initBigInt + 2.initBigInt)
assert 0.initBigInt == initBigInt("0")
```

---

### `<`

```nim
func `<`*(a, b: BigInt): bool
```

Возвращает `true`, если `a` строго меньше `b`.

```nim
assert 3.initBigInt < 5.initBigInt
assert (-10).initBigInt < 0.initBigInt
```

---

### `<=`

```nim
func `<=`*(a, b: BigInt): bool
```

Возвращает `true`, если `a` меньше или равно `b`.

```nim
assert 5.initBigInt <= (3.initBigInt + 2.initBigInt)  # равно
assert 3.initBigInt <= 5.initBigInt                    # строго меньше
```

> **Примечание:** `!=`, `>` и `>=` автоматически выводятся Nim из трёх вышеперечисленных.

---

## Арифметические операторы

Все арифметические операторы возвращают новый `BigInt` — операнды не изменяются.

### Унарный минус `-`

```nim
func `-`*(a: BigInt): BigInt
```

Меняет знак `a` на противоположный. Возвращает аддитивно обратное число.

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

Возвращает абсолютное значение `a`. Положительные числа и ноль возвращаются без изменений; у отрицательных переворачивается знак.

```nim
assert abs((-42).initBigInt) == 42.initBigInt
assert abs(42.initBigInt)    == 42.initBigInt
```

---

### `+` и `+=`

```nim
func `+`*(a, b: BigInt): BigInt
template `+=`*(a: var BigInt, b: BigInt)
```

Складывает два значения `BigInt`. `+=` — обновление на месте. Знаки обрабатываются автоматически.

```nim
let sum = 5.initBigInt + 10.initBigInt    # 15
var x = 100.initBigInt
x += 23.initBigInt                         # x теперь 123
```

---

### `-` и `-=`

```nim
func `-`*(a, b: BigInt): BigInt
template `-=`*(a: var BigInt, b: BigInt)
```

Вычитает `b` из `a`. Все комбинации знаков, включая смену знака результата, обрабатываются корректно.

```nim
assert 15.initBigInt - 10.initBigInt     == 5.initBigInt
assert (-15).initBigInt - 10.initBigInt  == (-25).initBigInt
assert 15.initBigInt - (-10).initBigInt  == 25.initBigInt
```

---

### `*` и `*=`

```nim
func `*`*(a, b: BigInt): BigInt
template `*=`*(a: var BigInt, b: BigInt)
```

Перемножает два значения `BigInt`. Использует классическое (длинное) умножение по 32-битным лимбам. Знак результата определяется по стандартному правилу произведения знаков.

```nim
assert 421.initBigInt * 200.initBigInt == 84200.initBigInt
assert (-3).initBigInt * 4.initBigInt  == (-12).initBigInt

# Вычислить факториал большого числа:
var result = 1.initBigInt
for i in 1..20:
  result *= i.initBigInt
# result == 2432902008176640000
```

---

## Инкремент и декремент

### `inc`

```nim
func inc*(a: var BigInt, b: int = 1)
```

Увеличивает `a` на месте на `b` (по умолчанию 1). Эквивалентно `a += b.initBigInt`, но эффективнее для малых значений `int`, так как не создаёт временный `BigInt`.

```nim
var a = 15.initBigInt
inc a           # a == 16
inc(a, 7)       # a == 23
inc(a, -5)      # a == 18 (отрицательный шаг = декремент)
```

---

### `dec`

```nim
func dec*(a: var BigInt, b: int = 1)
```

Уменьшает `a` на месте на `b` (по умолчанию 1).

```nim
var a = 15.initBigInt
dec a           # a == 14
dec(a, 5)       # a == 9
```

---

## Преемник и предшественник

### `succ`

```nim
func succ*(a: BigInt, b: int = 1): BigInt
```

Возвращает `b`-й преемник `a` — то есть `a + b` — не изменяя `a`. При отрицательном `b` ведёт себя как `pred`.

```nim
assert succ(10.initBigInt)    == 11.initBigInt
assert succ(10.initBigInt, 5) == 15.initBigInt
```

---

### `pred`

```nim
func pred*(a: BigInt, b: int = 1): BigInt
```

Возвращает `b`-й предшественник `a` — то есть `a - b`.

```nim
assert pred(10.initBigInt)    == 9.initBigInt
assert pred(10.initBigInt, 4) == 6.initBigInt
```

---

## Возведение в степень и сдвиги

### `pow`

```nim
func pow*(x: BigInt, y: Natural): BigInt
```

Возводит `x` в степень `y`. Использует бинарное возведение в степень (быстрое возведение через последовательное возведение в квадрат) — O(log y) умножений. Показатель `y` — обычный `Natural` (неотрицательный `int`), а не `BigInt`.

```nim
assert pow(2.initBigInt, 10) == 1024.initBigInt
assert pow(3.initBigInt, 0)  == 1.initBigInt    # всё^0 = 1
echo pow(2.initBigInt, 100)  # 2^100, 31-значное число
```

> Для модульного возведения в степень см. `powmod`.

---

### `shl` — сдвиг влево

```nim
func `shl`*(x: BigInt, y: Natural): BigInt
```

Сдвигает `x` влево на `y` бит, что эквивалентно умножению на 2^y. Знак `x` сохраняется. Сдвиг нуля всегда возвращает ноль.

```nim
assert 24.initBigInt shl 1 == 48.initBigInt   # × 2
assert 24.initBigInt shl 2 == 96.initBigInt   # × 4
assert 1.initBigInt shl 64 == initBigInt("18446744073709551616")
```

---

### `shr` — сдвиг вправо

```nim
func `shr`*(x: BigInt, y: Natural): BigInt
```

Сдвигает `x` вправо на `y` бит **арифметически**: для отрицательных значений слева заполняется единицами (деление нацело на 2^y), что соответствует соглашению дополнительного кода.

```nim
assert 24.initBigInt shr 1   == 12.initBigInt
assert 24.initBigInt shr 2   == 6.initBigInt
assert (-5).initBigInt shr 1 == (-3).initBigInt  # floor(-5/2) = -3
```

---

## Побитовые операции

Все побитовые операторы трактуют отрицательные `BigInt` так, как если бы они хранились в дополнительном коде с бесконечным количеством знаковых битов. Это совпадает с семантикой целых чисел Python и позволяет выполняться тождествам `not x == -(x + 1)`, `x and -1 == x` и т. д.

### `not`

```nim
func `not`*(a: BigInt): BigInt
```

Побитовое дополнение. Следует тождеству дополнительного кода: `not x = -(x + 1)`.

```nim
assert not 0.initBigInt  == (-1).initBigInt
assert not 5.initBigInt  == (-6).initBigInt
assert not (-1).initBigInt == 0.initBigInt
```

---

### `and`

```nim
func `and`*(a, b: BigInt): BigInt
```

Побитовое И. Корректно работает для всех комбинаций знаков, внутренне приводя к форме дополнительного кода.

```nim
assert 0b1100.initBigInt and 0b1010.initBigInt == 0b1000.initBigInt
assert (-1).initBigInt and 42.initBigInt == 42.initBigInt  # -1 — все единицы
```

---

### `or`

```nim
func `or`*(a, b: BigInt): BigInt
```

Побитовое ИЛИ.

```nim
assert 0b1100.initBigInt or 0b1010.initBigInt == 0b1110.initBigInt
```

---

### `xor`

```nim
func `xor`*(a, b: BigInt): BigInt
```

Побитовое исключающее ИЛИ.

```nim
assert 0b1100.initBigInt xor 0b1010.initBigInt == 0b0110.initBigInt
assert x xor x == 0.initBigInt  # любое число XOR себя — ноль
```

---

## Деление и остаток

Модель деления, используемая `div` и `mod`, — **деление с округлением к минус бесконечности** (также называемое делением в стиле Python). Частное округляется к отрицательной бесконечности, а остаток всегда имеет тот же знак, что и делитель. Это отличается от встроенного `div`/`mod` Nim для целых, который усекает к нулю.

Для любого ненулевого `b` выполняются тождества:
- `a == (a div b) * b + (a mod b)`
- `sign(a mod b) == sign(b)`

### `div`

```nim
func `div`*(a, b: BigInt): BigInt
```

Деление нацело с округлением к минус бесконечности. Бросает `DivByZeroDefect` при `b == 0`.

```nim
let a = 17.initBigInt
let b = 5.initBigInt
assert a div b        == 3.initBigInt     #  17/5  → floor(3.4)  = 3
assert (-a) div b     == (-4).initBigInt  # -17/5  → floor(-3.4) = -4
assert a div (-b)     == (-4).initBigInt  #  17/-5 → floor(-3.4) = -4
assert (-a) div (-b)  == 3.initBigInt     # -17/-5 → floor(3.4)  = 3
```

---

### `mod`

```nim
func `mod`*(a, b: BigInt): BigInt
```

Остаток после деления с округлением к минус бесконечности. Знак результата всегда совпадает со знаком `b`. Бросает `DivByZeroDefect` при `b == 0`.

```nim
let a = 17.initBigInt
let b = 5.initBigInt
assert a mod b        == 2.initBigInt     # положит. mod положит. → положит.
assert (-a) mod b     == 3.initBigInt     # отрицат. mod положит. → положит.
assert a mod (-b)     == (-3).initBigInt  # положит. mod отрицат. → отрицат.
assert (-a) mod (-b)  == (-2).initBigInt  # отрицат. mod отрицат. → отрицат.
```

---

### `divmod`

```nim
func divmod*(a, b: BigInt): tuple[q, r: BigInt]
```

Вычисляет частное и остаток за один проход — эффективнее, чем вызывать `div` и `mod` по отдельности, когда нужны оба результата. Бросает `DivByZeroDefect` при `b == 0`.

```nim
let (q, r) = divmod(17.initBigInt, 5.initBigInt)
assert q == 3.initBigInt
assert r == 2.initBigInt
```

---

## Итераторы

Итераторы по диапазонам `BigInt` следуют тем же соглашениям об именовании, что и встроенные числовые итераторы Nim.

### `countup`

```nim
iterator countup*(a, b: BigInt, step: int32 = 1): BigInt
```

Выдаёт каждый `BigInt` от `a` до `b` включительно, с шагом `step`. Шаг — обычный `int32`.

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

Выдаёт каждый `BigInt` от `a` вниз до `b` включительно, убывая на `step`.

```nim
for x in countdown(5.initBigInt, 1.initBigInt):
  echo x   # 5, 4, 3, 2, 1
```

---

### `..` (включительный диапазон)

```nim
iterator `..`*(a, b: BigInt): BigInt
```

Считает от `a` до `b` включительно с шагом 1. Обеспечивает естественный синтаксис диапазонов.

```nim
for x in 1.initBigInt .. 5.initBigInt:
  echo x   # 1, 2, 3, 4, 5
```

---

### `..<` (исключительный диапазон)

```nim
iterator `..<`*(a, b: BigInt): BigInt
```

Считает от `a` до — но не включая — `b`.

```nim
for x in 1.initBigInt ..< 5.initBigInt:
  echo x   # 1, 2, 3, 4  (5 исключено)
```

---

## Теория чисел

### `fastLog2`

```nim
func fastLog2*(a: BigInt): int
```

Возвращает ⌊log₂|a|⌋ — позицию старшего установленного бита в абсолютном значении `a`. Два особых случая:

- Возвращает `-1`, если `a` равно нулю (логарифм не определён).
- Работает только с величиной: `fastLog2(-8.initBigInt)` возвращает 3, как и `fastLog2(8.initBigInt)`.

Работает очень быстро — смотрит только на старший лимб и использует одну CPU-инструкцию. Особенно полезен для быстрой оценки битовой длины `BigInt` без полного преобразования.

```nim
assert fastLog2(0.initBigInt)    == -1
assert fastLog2(1.initBigInt)    == 0
assert fastLog2(8.initBigInt)    == 3   # 2^3 = 8
assert fastLog2(255.initBigInt)  == 7   # старший бит — бит 7
assert fastLog2(256.initBigInt)  == 8   # 2^8 = 256
```

---

### `gcd`

```nim
func gcd*(a, b: BigInt): BigInt
```

Возвращает наибольший общий делитель (НОД) `a` и `b`. Результат всегда неотрицателен вне зависимости от знаков аргументов. Использует бинарный алгоритм НОД (алгоритм Штейна), который избегает деления и эффективен для многоточных целых.

```nim
assert gcd(54.initBigInt, 24.initBigInt)  == 6.initBigInt
assert gcd(0.initBigInt, 7.initBigInt)    == 7.initBigInt  # НОД(0, n) = n
assert gcd((-6).initBigInt, 9.initBigInt) == 3.initBigInt  # знак игнорируется
```

---

### `invmod`

```nim
func invmod*(a, modulus: BigInt): BigInt
```

Вычисляет **обратный элемент** `a` по модулю `modulus` — то есть единственное целое `x` из `[1, modulus−1]` такое, что `(a * x) mod modulus == 1`. Использует расширенный алгоритм Евклида.

Бросает:
- `DivByZeroDefect`, если `modulus` равен нулю или `a` равен нулю.
- `ValueError`, если `modulus` отрицателен или обратного элемента не существует (то есть `gcd(a, modulus) != 1`).

Результат всегда находится в диапазоне `[1, modulus − 1]`.

```nim
# 3 * 5 ≡ 15 ≡ 1 (mod 7)
assert invmod(3.initBigInt, 7.initBigInt) == 5.initBigInt

# Обратный элемент используется при генерации ключей RSA и в модульной арифметике
```

---

### `powmod`

```nim
func powmod*(base, exponent, modulus: BigInt): BigInt
```

Эффективно вычисляет `(base ^ exponent) mod modulus`. Результат всегда находится в `[0, modulus − 1]`.

Использует быстрое бинарное возведение в степень, сохраняя промежуточные значения взятыми по модулю, так что никогда не приходится перемножать числа больше чем `modulus²`. Это делает функцию практически применимой для модулей размером в тысячи бит (RSA, Диффи-Хеллман и т. д.).

Специальная обработка:
- Если `exponent` отрицателен, `base` сначала заменяется его обратным элементом.
- Если `modulus == 1`, результат всегда 0.

Бросает:
- `DivByZeroDefect`, если `modulus` равен нулю.
- `ValueError`, если `modulus` отрицателен.

```nim
# 2^3 mod 7 = 8 mod 7 = 1
assert powmod(2.initBigInt, 3.initBigInt, 7.initBigInt) == 1.initBigInt

# Малая теорема Ферма: a^(p-1) ≡ 1 (mod p) для простого p
let p = 17.initBigInt
assert powmod(3.initBigInt, pred(p), p) == 1.initBigInt
```

---

## Преобразование типов

### `toInt`

```nim
func toInt*[T: SomeInteger](x: BigInt): Option[T]
```

Пытается конвертировать `x` в запрошенный целочисленный тип `T`. Поскольку `BigInt` может хранить значения далеко за пределами диапазона любого фиксированного типа, результат обёрнут в `Option`:

- Возвращает `some(value)`, если `x` помещается в `T`.
- Возвращает `none(T)`, если `x` выходит за диапазон или (для беззнаковых типов) отрицателен.

Это безопасная альтернатива прямому приведению типов. Она заставляет явно обработать случай выхода за диапазон вместо молчаливого усечения.

```nim
import std/options

assert toInt[int8](44.initBigInt)    == some(44'i8)
assert toInt[int8](200.initBigInt)   == none(int8)    # 200 > 127
assert toInt[uint8](200.initBigInt)  == some(200'u8)
assert toInt[uint8]((-1).initBigInt) == none(uint8)   # беззнаковый не бывает отрицательным
assert toInt[int](1_000_000.initBigInt) == some(1_000_000)
```

---

## Строковый ввод-вывод

### `toString`

```nim
func toString*(a: BigInt, base: range[2..36] = 10): string
```

Преобразует `a` в строку в заданном основании `base` (от 2 до 36). По умолчанию — основание 10. Отрицательные числа получают префикс `-`. Никаких префиксов основания (`0x`, `0b` и т. д.) не добавляется — при необходимости вызывающий добавляет их сам. Буквы для цифр выше 9 — строчные.

```nim
let a = 55.initBigInt
assert toString(a)       == "55"
assert toString(a, 2)    == "110111"    # двоичное
assert toString(a, 8)    == "67"        # восьмеричное
assert toString(a, 16)   == "37"        # шестнадцатеричное
assert toString(a, 36)   == "1j"        # основание 36

echo "0x" & toString(255.initBigInt, 16)  # "0xff" — префикс добавляется вручную
```

---

### `$`

```nim
func `$`*(a: BigInt): string
```

Стандартное строковое преобразование. Эквивалентно `toString(a, 10)`. Именно это вызывают `echo` и строковая интерполяция.

```nim
let big = pow(2.initBigInt, 100)
echo big   # выводит десятичное значение 2^100
echo $big  # то же самое
```

---

## Генерация случайных чисел — `bigints/random`

> **Модуль:** `bigints/random`  
> **Импорт:** `import bigints/random`  
> **Зависит от:** `std/random` (встроенный ГПСЧ Nim на основе Xoshiro)

Этот модуль расширяет тип `Rand` Nim перегрузками `rand`, производящими равномерно распределённые случайные значения `BigInt` в заданном диапазоне.

**Равномерность:** Реализация генерирует случайные лимбы снизу вверх и использует аккуратный подход для старших битов, гарантируя равномерное распределение в запрошенном диапазоне с помощью цикла повторных попыток для устранения смещения.

---

### `rand` с состоянием `Rand` и `Slice[BigInt]`

```nim
func rand*(r: var Rand, x: Slice[BigInt]): BigInt
```

Возвращает равномерно распределённый случайный `BigInt` в замкнутом интервале `[x.a, x.b]`. Использует и обновляет состояние `Rand r`. Это состоятельная, воспроизводимая форма — засеивая `r` фиксированным значением, вы получаете детерминированные последовательности.

Предусловие: `x.a <= x.b`. Нарушение вызывает ошибку утверждения.

```nim
import std/random
import bigints/random

var rng = initRand(12345)                          # ГСЧ с сидом
let lo = 1.initBigInt
let hi = initBigInt("1000000000000000000000")      # 10^21

let r = rng.rand(lo..hi)
assert r >= lo and r <= hi
```

---

### `rand` с состоянием `Rand` и `max: BigInt`

```nim
func rand*(r: var Rand, max: BigInt): BigInt
```

Возвращает равномерно распределённый случайный `BigInt` в `[0, max]`. Удобное сокращение для `rand(r, 0.initBigInt..max)`.

```nim
var rng = initRand(42)
let modulus = initBigInt("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFFFFFFFFFFFF", base = 16)
let k = rng.rand(modulus)  # случайный скаляр для операций на эллиптической кривой
```

---

### `rand` с `Slice[BigInt]` (глобальное состояние)

```nim
proc rand*(x: Slice[BigInt]): BigInt
```

Удобный вариант без явного состояния, использующий модульный объект `Rand` (инициализированный значением 777 при запуске программы). Полезен для быстрых скриптов и невоспроизводимой генерации. Для воспроизводимых результатов предпочтительнее перегрузки `func`, принимающие явный аргумент `Rand`.

```nim
import bigints/random

# Быстрый случайный BigInt — не воспроизводится между запусками без явного сида
let r = rand(0.initBigInt .. 1_000_000.initBigInt)
echo r
```

---

### `rand` с `max: BigInt` (глобальное состояние)

```nim
proc rand*(max: BigInt): BigInt
```

Возвращает случайный `BigInt` в `[0, max]`, используя глобальное состояние ГПСЧ. Эквивалентно `rand(0.initBigInt..max)`.

```nim
import bigints/random

let lucky = rand(100.initBigInt)  # от 0 до 100 включительно
```

---

## Таблица быстрого доступа

| Задача | Функция / Оператор |
|---|---|
| Создать из целого числа | `42.initBigInt` |
| Создать из строки (десятичное) | `initBigInt("123456789")` |
| Создать из строки (другое основание) | `initBigInt("ff", base=16)` |
| Создать из сырых лимбов | `@[lo, hi].initBigInt` |
| Арифметика | `+`, `-`, `*`, `div`, `mod`, `-` (унарный) |
| Деление и остаток одновременно | `divmod(a, b)` |
| Присваивание с операцией | `+=`, `-=`, `*=` |
| Инкремент/декремент на месте | `inc(a)`, `dec(a)` |
| Неизменяемый следующий/предыдущий | `succ(a)`, `pred(a)` |
| Возведение в степень | `pow(x, y)` (y: Natural) |
| Побитовые сдвиги | `shl`, `shr` |
| Побитовые операции | `not`, `and`, `or`, `xor` |
| Сравнения | `==`, `<`, `<=` (и производные `!=`, `>`, `>=`) |
| Абсолютное значение | `abs(a)` |
| Битовая длина | `fastLog2(a)` |
| НОД | `gcd(a, b)` |
| Обратный по модулю | `invmod(a, modulus)` |
| Возведение в степень по модулю | `powmod(base, exp, modulus)` |
| В фиксированный тип (безопасно) | `toInt[T](x)` → `Option[T]` |
| В строку | `$a`, `toString(a, base)` |
| Случайный в диапазоне | `rand(rng, lo..hi)` |
| Случайный от 0 до max | `rand(rng, max)` |
| Итерация включительно | `for x in a..b` |
| Итерация исключительно | `for x in a..<b` |
| Итерация с шагом | `countup(a, b, step)`, `countdown(a, b, step)` |
