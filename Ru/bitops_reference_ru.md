# Модуль Nim `std/bitops` — полный справочник

> Низкоуровневые утилиты для побитовых операций над целыми числами в Nim.  
> Там, где это возможно, автоматически используются компиляторные интринсики (GCC, LLVM/Clang, VCC, ICC) для максимальной производительности.  
> Модуль совместим с JavaScript, NimScript и виртуальной машиной времени компиляции.

---

## Содержание

1. [Типы](#типы)
2. [Побитовые логические операторы](#побитовые-логические-операторы)
3. [Извлечение битового среза](#извлечение-битового-среза)
4. [Создание масок](#создание-масок)
5. [Маскирование — немутирующее (функциональный стиль)](#маскирование--немутирующее-функциональный-стиль)
6. [Маскирование — мутирующее (in-place)](#маскирование--мутирующее-in-place)
7. [Операции над одним битом](#операции-над-одним-битом)
8. [Операции над несколькими битами](#операции-над-несколькими-битами)
9. [Проверка бита](#проверка-бита)
10. [Подсчёт и анализ](#подсчёт-и-анализ)
11. [Циклический сдвиг (ротация)](#циклический-сдвиг-ротация)
12. [Разворот битов](#разворот-битов)
13. [Флаги компилятора](#флаги-компилятора)

---

## Типы

### `BitsRange[T]`

```nim
type BitsRange*[T] = range[0..sizeof(T)*8-1]
```

Тип-диапазон, описывающий все допустимые позиции битов для целочисленного типа `T`.  
Для `uint8` — это `0..7`, для `uint32` — `0..31`.

---

## Побитовые логические операторы

### `bitnot`

```nim
func bitnot*[T: SomeInteger](x: T): T
```

Возвращает **побитовое дополнение** (NOT) числа `x`. Каждый 0 становится 1 и наоборот.

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

Вычисляет **побитовое И** всех аргументов. Принимает два и более целых числа одного типа.  
Результирующий бит равен 1 только тогда, когда все соответствующие исходные биты равны 1.

```nim
doAssert bitand(0b1100'u8, 0b1010'u8) == 0b1000'u8
doAssert bitand(0b1111'u8, 0b1010'u8, 0b1001'u8) == 0b1000'u8
```

---

### `bitor`

```nim
macro bitor*[T: SomeInteger](x, y: T; z: varargs[T]): T
```

Вычисляет **побитовое ИЛИ** всех аргументов. Результирующий бит равен 1, если **хотя бы один** исходный бит равен 1.

```nim
doAssert bitor(0b1100'u8, 0b0011'u8) == 0b1111'u8
doAssert bitor(0b0001'u8, 0b0010'u8, 0b0100'u8) == 0b0111'u8
```

---

### `bitxor`

```nim
macro bitxor*[T: SomeInteger](x, y: T; z: varargs[T]): T
```

Вычисляет **побитовое исключающее ИЛИ** (XOR) всех аргументов. Результирующий бит равен 1, когда **количество единичных битов среди источников нечётное**.

```nim
doAssert bitxor(0b1100'u8, 0b1010'u8) == 0b0110'u8
doAssert bitxor(0b0001'u8, 0b0011'u8, 0b0111'u8) == 0b0101'u8
```

---

## Извлечение битового среза

### `bitsliced`

```nim
func bitsliced*[T: SomeInteger](v: T; slice: Slice[int]): T
```

Возвращает **извлечённый и сдвинутый вправо** диапазон битов из `v`. Границы диапазона включены с обеих сторон; допустимы оба варианта `..` и `..<`.

```nim
doAssert 0b10111.bitsliced(2 .. 4) == 0b101   # биты 2,3,4 → сдвиг вправо на 2
doAssert 0b11100.bitsliced(0 .. 2) == 0b100
doAssert 0b11100.bitsliced(0 ..< 3) == 0b100  # то же, что 0..2
```

> **Подсказка:** В результате `slice.a` становится битом 0 выходного значения.

---

### `bitslice`

```nim
proc bitslice*[T: SomeInteger](v: var T; slice: Slice[int])
```

**Мутирующая** версия `bitsliced`. Изменяет `v` на месте.

```nim
var x = 0b101110
x.bitslice(2 .. 4)
doAssert x == 0b011
```

---

## Создание масок

### `toMask`

```nim
func toMask*[T: SomeInteger](slice: Slice[int]): T
```

Создаёт битовую маску с единичными битами именно в позициях, описанных `slice`.

```nim
doAssert toMask[int32](1 .. 3) == 0b1110'i32   # биты 1, 2, 3 установлены
doAssert toMask[int32](0 .. 3) == 0b1111'i32   # биты 0, 1, 2, 3 установлены
```

---

## Маскирование — немутирующее (функциональный стиль)

Эти функции **возвращают новое значение**, не изменяя оригинал.

---

### `masked(v, mask)`

```nim
proc masked*[T: SomeInteger](v, mask: T): T
```

Возвращает `v AND mask` — сохраняет только те биты `v`, которые также равны 1 в `mask`.

```nim
let v = 0b0000_0011'u8
doAssert v.masked(0b0000_1010'u8) == 0b0000_0010'u8
```

---

### `masked(v, slice)`

```nim
func masked*[T: SomeInteger](v: T; slice: Slice[int]): T
```

Возвращает `v AND toMask[T](slice)` — сохраняет только биты в заданном диапазоне.

```nim
let v = 0b0000_1011'u8
doAssert v.masked(1 .. 3) == 0b0000_1010'u8
```

---

### `setMasked(v, mask)`

```nim
func setMasked*[T: SomeInteger](v, mask: T): T
```

Возвращает `v OR mask` — устанавливает все биты, равные 1 в `mask`.

```nim
let v = 0b0000_0011'u8
doAssert v.setMasked(0b0000_1010'u8) == 0b0000_1011'u8
```

---

### `setMasked(v, slice)`

```nim
func setMasked*[T: SomeInteger](v: T; slice: Slice[int]): T
```

Возвращает `v` со всеми битами в заданном диапазоне, установленными в 1.

```nim
let v = 0b0000_0011'u8
doAssert v.setMasked(2 .. 3) == 0b0000_1111'u8
```

---

### `clearMasked(v, mask)`

```nim
func clearMasked*[T: SomeInteger](v, mask: T): T
```

Возвращает `v` со всеми битами, равными 1 в `mask`, **сброшенными в 0**.  
Эквивалентно `v AND NOT mask`.

```nim
let v = 0b0000_0011'u8
doAssert v.clearMasked(0b0000_1010'u8) == 0b0000_0001'u8
```

---

### `clearMasked(v, slice)`

```nim
func clearMasked*[T: SomeInteger](v: T; slice: Slice[int]): T
```

Возвращает `v` со всеми битами в заданном диапазоне, **сброшенными в 0**.

```nim
let v = 0b0000_0011'u8
doAssert v.clearMasked(1 .. 3) == 0b0000_0001'u8
```

---

### `flipMasked(v, mask)`

```nim
func flipMasked*[T: SomeInteger](v, mask: T): T
```

Возвращает `v XOR mask` — **инвертирует** все биты, равные 1 в `mask`.

```nim
let v = 0b0000_0011'u8
doAssert v.flipMasked(0b0000_1010'u8) == 0b0000_1001'u8
```

---

### `flipMasked(v, slice)`

```nim
func flipMasked*[T: SomeInteger](v: T; slice: Slice[int]): T
```

Возвращает `v` со всеми битами в заданном диапазоне, **инвертированными**.

```nim
let v = 0b0000_0011'u8
doAssert v.flipMasked(1 .. 3) == 0b0000_1101'u8
```

---

## Маскирование — мутирующее (in-place)

Эти процедуры изменяют первый аргумент непосредственно (`var T`).

---

### `mask(v, mask)`

```nim
proc mask*[T: SomeInteger](v: var T; mask: T)
```

Изменяет `v` на `v AND mask`.

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

Изменяет `v`, сохраняя только биты в заданном диапазоне.

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

Изменяет `v` на `v OR mask` — устанавливает все биты, равные 1 в `mask`.

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

Изменяет `v`, устанавливая все биты в заданном диапазоне в 1.

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

Изменяет `v` на `v AND NOT mask` — сбрасывает в 0 все биты, равные 1 в `mask`.

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

Изменяет `v`, сбрасывая в 0 все биты в заданном диапазоне.

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

Изменяет `v` на `v XOR mask` — инвертирует все биты, равные 1 в `mask`.

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

Изменяет `v`, инвертируя все биты в заданном диапазоне.

```nim
var v = 0b0000_0011'u8
v.flipMask(1 .. 3)
doAssert v == 0b0000_1101'u8
```

---

## Операции над одним битом

### `setBit`

```nim
proc setBit*[T: SomeInteger](v: var T; bit: BitsRange[T])
```

Устанавливает бит на позиции `bit` в **1**. Бит 0 — наименее значимый бит (LSB).

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

Устанавливает бит на позиции `bit` в **0**.

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

**Инвертирует** бит на позиции `bit`.

```nim
var v = 0b0000_0011'u8
v.flipBit(2'u8)
doAssert v == 0b0000_0111'u8
```

---

## Операции над несколькими битами

### `setBits`

```nim
macro setBits*(v: typed; bits: varargs[typed]): untyped
```

Устанавливает **несколько битов** в 1 за один вызов.

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

Сбрасывает **несколько битов** в 0 за один вызов.

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

**Инвертирует несколько битов** за один вызов.

```nim
var v = 0b0000_1111'u8
v.flipBits(1, 3, 5, 7)
doAssert v == 0b1010_0101'u8
```

---

## Проверка бита

### `testBit`

```nim
proc testBit*[T: SomeInteger](v: T; bit: BitsRange[T]): bool
```

Возвращает `true`, если бит на позиции `bit` равен **1**.

```nim
let v = 0b0000_1111'u8
doAssert v.testBit(0)         # бит 0 установлен
doAssert not v.testBit(7)     # бит 7 сброшен
```

---

## Подсчёт и анализ

### `countSetBits`

```nim
func countSetBits*(x: SomeInteger): int
```

Подсчитывает количество **единичных битов** в `x` (вес Хэмминга / popcount).  
Использует компиляторные интринсики (`__builtin_popcount`) там, где они доступны.

```nim
doAssert countSetBits(0b0000_0011'u8) == 2
doAssert countSetBits(0b1010_1010'u8) == 4
```

---

### `popcount`

```nim
func popcount*(x: SomeInteger): int
```

Псевдоним для `countSetBits`. Предоставляется для совместимости с C-именованием.

```nim
doAssert popcount(0b1111_0000'u8) == 4
```

---

### `parityBits`

```nim
func parityBits*(x: SomeInteger): int
```

Возвращает **1**, если количество единичных битов **нечётно**, **0** — если чётно.  
Применяется в схемах обнаружения ошибок.

```nim
doAssert parityBits(0b0000_0000'u8) == 0  # 0 единиц → чётно
doAssert parityBits(0b0101_0001'u8) == 1  # 3 единицы → нечётно
doAssert parityBits(0b0110_1001'u8) == 0  # 4 единицы → чётно
doAssert parityBits(0b0111_1111'u8) == 1  # 7 единиц → нечётно
```

---

### `firstSetBit`

```nim
func firstSetBit*(x: SomeInteger): int
```

Возвращает **1-базированный** индекс **наименее значимого** установленного бита.  
Возвращает 0 при `x = 0`, если активен флаг `noUndefinedBitOpts`; иначе результат не определён.

```nim
doAssert firstSetBit(0b0000_0001'u8) == 1
doAssert firstSetBit(0b0000_0010'u8) == 2
doAssert firstSetBit(0b0000_1111'u8) == 1  # LSB установлен
doAssert firstSetBit(0b0000_1000'u8) == 4
```

---

### `fastLog2`

```nim
func fastLog2*(x: SomeInteger): int
```

Быстрое целочисленное **floor(log₂(x))**. Эквивалентно позиции старшего установленного бита (0-базированная).  
Возвращает -1 при `x = 0`, если активен флаг `noUndefinedBitOpts`; иначе не определён.

```nim
doAssert fastLog2(0b0000_0001'u8) == 0  # 2^0 = 1
doAssert fastLog2(0b0000_0010'u8) == 1  # 2^1 = 2
doAssert fastLog2(0b0000_1111'u8) == 3  # старший бит на позиции 3
```

> **Применение:** `fastLog2(x) + 1` — количество битов, необходимых для представления `x`.

---

### `countLeadingZeroBits`

```nim
func countLeadingZeroBits*(x: SomeInteger): int
```

Возвращает количество **ведущих нулевых битов** (от старшего бита вниз).  
При `x = 0` возвращает 0, если активен `noUndefinedBitOpts`; иначе не определён.

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

Возвращает количество **завершающих нулевых битов** (от младшего бита вверх).  
При `x = 0` возвращает 0, если активен `noUndefinedBitOpts`; иначе не определён.

```nim
doAssert countTrailingZeroBits(0b0000_0001'u8) == 0
doAssert countTrailingZeroBits(0b0000_0010'u8) == 1
doAssert countTrailingZeroBits(0b0000_1000'u8) == 3
doAssert countTrailingZeroBits(0b0000_1111'u8) == 0
```

> **Связь:** `countTrailingZeroBits(x) == firstSetBit(x) - 1` для ненулевого `x`.

---

## Циклический сдвиг (ротация)

В отличие от обычного сдвига, при ротации биты, «выпавшие» с одного конца, переносятся на другой.

### `rotateLeftBits`

```nim
func rotateLeftBits*[T: SomeUnsignedInt](value: T, shift: range[0..(sizeof(T)*8)]): T
```

Выполняет **циклический сдвиг влево** на `shift` позиций.

```nim
doAssert rotateLeftBits(0b0110_1001'u8, 4) == 0b1001_0110'u8
doAssert rotateLeftBits(0b00111100_11000011'u16, 8) == 0b11000011_00111100'u16
```

---

### `rotateRightBits`

```nim
func rotateRightBits*[T: SomeUnsignedInt](value: T, shift: range[0..(sizeof(T)*8)]): T
```

Выполняет **циклический сдвиг вправо** на `shift` позиций.

```nim
doAssert rotateRightBits(0b0110_1001'u8, 4) == 0b1001_0110'u8
doAssert rotateRightBits(0b00111100_11000011'u16, 8) == 0b11000011_00111100'u16
```

> **Примечание:** Обе функции используют аппаратные интринсики (инструкции `ROL`/`ROR` на x86) при компиляции через GCC, Clang, VCC или ICC.

---

## Разворот битов

### `reverseBits`

```nim
func reverseBits*[T: SomeUnsignedInt](x: T): T
```

Возвращает `x` с битами в **обратном порядке** — MSB становится LSB и наоборот.

```nim
doAssert reverseBits(0b10100100'u8) == 0b00100101'u8
doAssert reverseBits(0xdd'u8)       == 0xbb'u8
doAssert reverseBits(0xddbb'u16)    == 0xddbb'u16   # палиндром в битах
doAssert reverseBits(0xdeadbeef'u32) == 0xf77db57b'u32
```

---

## Флаги компилятора

| Флаг | Действие |
|---|---|
| `-d:noIntrinsicsBitOpts` | Отключает компиляторные интринсики; используются чистые реализации на Nim. |
| `-d:noUndefinedBitOpts` | Принудительно возвращает определённый результат (обычно 0 или -1) для граничных случаев (например, `x = 0`) в `firstSetBit`, `fastLog2`, `countLeadingZeroBits`, `countTrailingZeroBits`. Небольшое снижение производительности. |

---

## Таблица быстрого доступа

| Функция | Категория | Возвращает новое? | Мутирует? |
|---|---|:---:|:---:|
| `bitnot` | Логика | ✓ | — |
| `bitand` | Логика | ✓ | — |
| `bitor` | Логика | ✓ | — |
| `bitxor` | Логика | ✓ | — |
| `bitsliced` | Срез битов | ✓ | — |
| `bitslice` | Срез битов | — | ✓ |
| `toMask` | Создание маски | ✓ | — |
| `masked` | Маскирование | ✓ | — |
| `mask` | Маскирование | — | ✓ |
| `setMasked` | Маскирование | ✓ | — |
| `setMask` | Маскирование | — | ✓ |
| `clearMasked` | Маскирование | ✓ | — |
| `clearMask` | Маскирование | — | ✓ |
| `flipMasked` | Маскирование | ✓ | — |
| `flipMask` | Маскирование | — | ✓ |
| `setBit` | Один бит | — | ✓ |
| `clearBit` | Один бит | — | ✓ |
| `flipBit` | Один бит | — | ✓ |
| `setBits` | Несколько битов | — | ✓ |
| `clearBits` | Несколько битов | — | ✓ |
| `flipBits` | Несколько битов | — | ✓ |
| `testBit` | Проверка | ✓ (`bool`) | — |
| `countSetBits` | Анализ | ✓ | — |
| `popcount` | Анализ | ✓ | — |
| `parityBits` | Анализ | ✓ | — |
| `firstSetBit` | Анализ | ✓ | — |
| `fastLog2` | Анализ | ✓ | — |
| `countLeadingZeroBits` | Анализ | ✓ | — |
| `countTrailingZeroBits` | Анализ | ✓ | — |
| `rotateLeftBits` | Ротация | ✓ | — |
| `rotateRightBits` | Ротация | ✓ | — |
| `reverseBits` | Разворот | ✓ | — |
