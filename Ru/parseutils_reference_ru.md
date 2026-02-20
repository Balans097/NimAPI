# Справочник модуля `std/parseutils` (Nim)

> Модуль содержит вспомогательные процедуры для лексического разбора строк: токены, числа, идентификаторы, пропуск символов, интерполяция и разбор «человекочитаемых» размеров.

**Соглашение о возвращаемом значении:** большинство процедур возвращают **количество обработанных символов** или `0` в случае ошибки разбора. Это позволяет легко строить цепочечные парсеры вручную, сдвигая позицию на полученное значение.

---

## Содержание

1. [Типы данных](#типы-данных)
2. [Разбор чисел в разных системах счисления](#разбор-чисел-в-разных-системах-счисления)
3. [Разбор целых чисел](#разбор-целых-чисел)
4. [Разбор вещественных чисел](#разбор-вещественных-чисел)
5. [Разбор размеров с единицами](#разбор-размеров-с-единицами)
6. [Разбор идентификаторов и символов](#разбор-идентификаторов-и-символов)
7. [Пропуск символов (skip)](#пропуск-символов-skip)
8. [Захват токенов (parse)](#захват-токенов-parse)
9. [Интерполяция строк](#интерполяция-строк)
10. [Практические рецепты](#практические-рецепты)

---

## Типы данных

### `InterpolatedKind`

Перечисление для итератора `interpolatedFragments`. Описывает тип фрагмента интерполированной строки:

| Значение    | Описание                                         |
|-------------|--------------------------------------------------|
| `ikStr`     | Обычная строковая часть                          |
| `ikDollar`  | Экранированный знак `$$` → `$`                   |
| `ikVar`     | Переменная `$name`                               |
| `ikExpr`    | Выражение `${expr}`                              |

---

## Разбор чисел в разных системах счисления

Все три процедуры (`parseBin`, `parseOct`, `parseHex`) имеют единую сигнатуру и поведение:

- Принимают `openArray[char]` или `string` + `start`.
- Пишут результат в переменную `number: var T`, где `T: SomeInteger`.
- Возвращают количество разобранных символов или `0` при ошибке.
- **Не проверяют переполнение** — при переполнении хранятся последние подходящие биты.
- Поддерживают символ-разделитель `_` (игнорируется).
- Автоматически распознают префиксы: `0b`/`0B` (бин.), `0o`/`0O` (окт.), `0x`/`0X` или `#` (шест.).

---

### `parseBin[T](s, number, maxLen=0)` / `parseBin[T](s, number, start=0, maxLen=0)`

Разбирает двоичное число.

**Параметры:**
- `s` — входная строка или `openArray[char]`
- `number` — переменная для записи результата
- `start` — начальная позиция (только для строковой версии)
- `maxLen` — максимальное количество символов для разбора (`0` = до конца)

```nim
import std/parseutils

var num: int
doAssert parseBin("0100_1110", num) == 9
doAssert num == 0b0100_1110   # 78

# С префиксом 0b
doAssert parseBin("0b1010", num) == 6
doAssert num == 10

# Ошибка: не двоичная цифра
doAssert parseBin("2101", num) == 0

# Тип int8 — хранятся последние 8 бит (переполнение без ошибки)
var b8: int8
doAssert parseBin("0b_0100_1110_0110_1001_1110_1101", b8) == 32
doAssert b8 == 0b1110_1101'i8   # -19

# maxLen: разобрать не более 9 символов начиная с позиции 3
doAssert parseBin("0b_0100_1110_0110_1001_1110_1101", b8, 3, 9) == 9
doAssert b8 == 0b0100_1110'i8
```

---

### `parseOct[T](s, number, maxLen=0)` / `parseOct[T](s, number, start=0, maxLen=0)`

Разбирает восьмеричное число. Распознаёт префикс `0o`/`0O`.

```nim
import std/parseutils

var num: int
doAssert parseOct("0o23464755", num) == 10
doAssert num == 5138925

# Ошибка: 8 не является восьмеричной цифрой
doAssert parseOct("8", num) == 0

# Переполнение int8 — без ошибки
var n8: int8
doAssert parseOct("0o_1464_755", n8) == 11
doAssert n8 == -19

# Разбор с произвольной позиции
doAssert parseOct("0o_1464_755", n8, 3, 3) == 3
doAssert n8 == 102
```

---

### `parseHex[T](s, number, maxLen=0)` / `parseHex[T](s, number, start=0, maxLen=0)`

Разбирает шестнадцатеричное число. Распознаёт `0x`/`0X` и `#` как префиксы. Нечувствителен к регистру (`a`–`f` / `A`–`F`).

```nim
import std/parseutils

var num: int
doAssert parseHex("4E_69_ED", num) == 8
doAssert num == 5138925

# Префикс #
doAssert parseHex("#ABC", num) == 4
doAssert num == 0xABC

# Ошибка
doAssert parseHex("X", num) == 0

# uint8
var u8: uint8
doAssert parseHex("0xFF", u8) == 4
doAssert u8 == 255

# Ограничение длины: 2 hex-цифры начиная с позиции 3
var n8: int8
doAssert parseHex("0x_4E_69_ED", n8, 3, 2) == 2
doAssert n8 == 0x4E'i8
```

---

## Разбор целых чисел

### `parseBiggestInt(s, number): int`

**Сигнатуры:**
```nim
proc parseBiggestInt(s: openArray[char], number: var BiggestInt): int
proc parseBiggestInt(s: string, number: var BiggestInt, start = 0): int
```

Разбирает знаковое целое максимального размера (`BiggestInt`). Поддерживает знак `+`/`-` и разделитель `_`. Бросает `ValueError` при выходе за диапазон.

```nim
import std/parseutils

var res: BiggestInt
doAssert parseBiggestInt("9223372036854775807", res) == 19
doAssert res == 9223372036854775807   # int64.high

doAssert parseBiggestInt("-2024_05_09", res) == 11
doAssert res == -20240509

# С позиции 7: разбирается "02" → 2 символа
doAssert parseBiggestInt("-2024_05_02", res, 7) == 4
doAssert res == 502

# Нет числа — возвращает 0, number не изменяется
doAssert parseBiggestInt("abc", res) == 0
```

---

### `parseInt(s, number): int`

**Сигнатуры:**
```nim
proc parseInt(s: openArray[char], number: var int): int
proc parseInt(s: string, number: var int, start = 0): int
```

Как `parseBiggestInt`, но тип результата `int`. Бросает `ValueError` при выходе за диапазон платформенного `int`.

```nim
import std/parseutils

var res: int
doAssert parseInt("-2024_05_02", res) == 11
doAssert res == -20240502

doAssert parseInt("  42", res) == 0   # пробел не является допустимым началом
doAssert parseInt("42  ", res) == 2   # пробел останавливает разбор
doAssert res == 42

# С позиции:
doAssert parseInt("-100abc", res, 1) == 3  # "100"
doAssert res == 100
```

---

### `parseSaturatedNatural(s, b): int`

**Сигнатуры:**
```nim
proc parseSaturatedNatural(s: openArray[char], b: var int): int
proc parseSaturatedNatural(s: string, b: var int, start = 0): int
```

Разбирает натуральное (неотрицательное) целое без риска исключения переполнения. При переполнении `b` устанавливается в `high(int)`. Рекомендуется вместо `parseInt` там, где важна устойчивость к переполнению.

```nim
import std/parseutils

var res = 0
discard parseSaturatedNatural("848", res)
doAssert res == 848

# Переполнение → high(int)
discard parseSaturatedNatural("999999999999999999999999", res)
doAssert res == high(int)

# Отрицательный знак не принимается
doAssert parseSaturatedNatural("-5", res) == 0
```

---

### `parseBiggestUInt(s, number): int`

**Сигнатуры:**
```nim
proc parseBiggestUInt(s: openArray[char], number: var BiggestUInt): int
proc parseBiggestUInt(s: string, number: var BiggestUInt, start = 0): int
```

Разбирает беззнаковое целое максимального размера. Отрицательные числа (строки, начинающиеся с `-`) вызывают `ValueError`.

```nim
import std/parseutils

var res: BiggestUInt
doAssert parseBiggestUInt("1111111111111111111", res) == 19
doAssert res == 1111111111111111111'u64

doAssert parseBiggestUInt("12", res) == 2
doAssert res == 12
```

---

### `parseUInt(s, number): int`

**Сигнатуры:**
```nim
proc parseUInt(s: openArray[char], number: var uint): int
proc parseUInt(s: string, number: var uint, start = 0): int
```

Как `parseBiggestUInt`, но тип результата `uint`.

```nim
import std/parseutils

var res: uint
doAssert parseUInt("3450", res) == 4
doAssert res == 3450

# Начиная с позиции 2: "50"
doAssert parseUInt("3450", res, 2) == 2
doAssert res == 50
```

---

## Разбор вещественных чисел

### `parseBiggestFloat(s, number): int`

**Сигнатуры:**
```nim
proc parseBiggestFloat(s: openArray[char], number: var BiggestFloat): int
proc parseBiggestFloat(s: string, number: var BiggestFloat, start = 0): int
```

Разбирает вещественное число в тип `BiggestFloat` (`float64`). Возвращает `0` при ошибке.

```nim
import std/parseutils

var res: float
discard parseBiggestFloat("3.14159", res)
doAssert res == 3.14159
```

---

### `parseFloat(s, number): int`

**Сигнатуры:**
```nim
proc parseFloat(s: openArray[char], number: var float): int
proc parseFloat(s: string, number: var float, start = 0): int
```

Разбирает вещественное число в тип `float`. Поддерживает экспоненциальную запись (`1e3`, `2.5E-4`).

```nim
import std/parseutils

var res: float
doAssert parseFloat("32", res) == 2
doAssert res == 32.0

doAssert parseFloat("32.57", res) == 5
doAssert res == 32.57

# С позиции 3: "57" → 57.0
doAssert parseFloat("32.57", res, 3) == 2
doAssert res == 57.0

doAssert parseFloat("1.5e3", res) == 5
doAssert res == 1500.0

# Ошибка
doAssert parseFloat("abc", res) == 0
```

---

## Разбор размеров с единицами

### `parseSize(s, size, alwaysBin=false): int`

**Сигнатура:**
```nim
func parseSize(s: openArray[char], size: var int64, alwaysBin = false): int
```

Разбирает «человекочитаемые» размеры данных с метрическими или двоичными префиксами. Возвращает количество разобранных символов или `0` при ошибке. Значение округляется до ближайшего целого. При переполнении `int64` результат насыщается до `int64.high`.

**Поддерживаемые префиксы** (регистронезависимо, кроме `i`):

| Префикс       | Метрический        | Двоичный (`...i`)   |
|---------------|--------------------|---------------------|
| `k` / `K`     | 10³ = 1 000        | 2¹⁰ = 1 024         |
| `m` / `M`     | 10⁶ = 1 000 000    | 2²⁰ = 1 048 576     |
| `g` / `G`     | 10⁹                | 2³⁰                 |
| `t` / `T`     | 10¹²               | 2⁴⁰                 |
| `p` / `P`     | 10¹⁵               | 2⁵⁰                 |
| `e` / `E`     | 10¹⁸               | 2⁶⁰                 |

Необязательный завершающий `B`/`b` игнорируется.  
`alwaysBin = true` — все префиксы трактуются как двоичные.

```nim
import std/parseutils

var res: int64

# Метрические единицы
doAssert parseSize("10.5 MB", res) == 7
doAssert res == 10_500_000

# Двоичные (суффикс 'i')
doAssert parseSize("64 mib", res) == 6
doAssert res == 67108864   # 64 * 2^20

# Форсированный двоичный режим
doAssert parseSize("1G/h", res, alwaysBin = true) == 2  # '/' останавливает разбор
doAssert res == 1073741824   # 2^30

# Просто число без единиц
doAssert parseSize("1024", res) == 4
doAssert res == 1024

# Ошибка (отрицательный)
doAssert parseSize("-5 MB", res) == 0
```

---

## Разбор идентификаторов и символов

### `parseIdent(s, ident): int` / `parseIdent(s): string`

**Сигнатуры:**
```nim
proc parseIdent(s: openArray[char], ident: var string): int
proc parseIdent(s: openArray[char]): string
proc parseIdent(s: string, ident: var string, start = 0): int
proc parseIdent(s: string, start = 0): string
```

Разбирает идентификатор Nim (первый символ из `{'a'..'z','A'..'Z','_'}`, далее `{'a'..'z','A'..'Z','0'..'9','_'}`).

- Версия с `var ident` — записывает результат в переменную, возвращает длину; при ошибке `ident` **не изменяется**.
- Версия без `var` — возвращает строку или `""` при ошибке.

```nim
import std/parseutils

var res: string

doAssert parseIdent("Hello World", res) == 5
doAssert res == "Hello"

# С позиции 6
doAssert parseIdent("Hello World", res, 6) == 5
doAssert res == "World"

# Пробел не является допустимым началом
doAssert parseIdent("Hello World", res, 5) == 0
# res остаётся "World" — не изменился при ошибке

# Версия, возвращающая строку
doAssert parseIdent("_myVar123 rest") == "_myVar123"
doAssert parseIdent("123abc") == ""   # не идентификатор (начинается с цифры)
```

---

### `parseChar(s, c): int`

**Сигнатуры:**
```nim
proc parseChar(s: openArray[char], c: var char): int
proc parseChar(s: string, c: var char, start = 0): int
```

Читает один символ. Возвращает `1` при успехе, `0` если позиция за концом строки.

```nim
import std/parseutils

var c: char

doAssert "nim".parseChar(c, 0) == 1
doAssert c == 'n'

doAssert "nim".parseChar(c, 2) == 1
doAssert c == 'm'

# За концом строки
doAssert "nim".parseChar(c, 3) == 0
# c не изменяется
```

---

## Пропуск символов (skip)

### `skipWhitespace(s): int`

**Сигнатуры:**
```nim
proc skipWhitespace(s: openArray[char]): int
proc skipWhitespace(s: string, start = 0): int
```

Пропускает пробельные символы (`' '`, `'\t'`, `'\v'`, `'\r'`, `'\n'`, `'\f'`). Возвращает количество пропущенных символов.

```nim
import std/parseutils

doAssert skipWhitespace("Hello World") == 0
doAssert skipWhitespace(" Hello World") == 1
doAssert skipWhitespace("Hello World", 5) == 1   # пробел между словами
doAssert skipWhitespace("Hello  World", 5) == 2  # два пробела
doAssert skipWhitespace("\t\n  data") == 4
```

---

### `skip(s, token): int`

**Сигнатуры:**
```nim
proc skip(s, token: openArray[char]): int
proc skip(s, token: string, start = 0): int
```

Проверяет, совпадает ли строка `token` с содержимым `s` начиная с текущей позиции. Возвращает длину `token` при совпадении или `0` при несовпадении. Чувствителен к регистру.

```nim
import std/parseutils

doAssert skip("2019-01-22", "2019") == 4
doAssert skip("2019-01-22", "19") == 0      # не совпадает с позиции 0
doAssert skip("2019-01-22", "19", 2) == 2   # совпадает с позиции 2
doAssert skip("CAPlow", "CAP") == 3
doAssert skip("CAPlow", "cap") == 0         # регистр важен
```

---

### `skipIgnoreCase(s, token): int`

**Сигнатуры:**
```nim
proc skipIgnoreCase(s, token: openArray[char]): int
proc skipIgnoreCase(s, token: string, start = 0): int
```

Как `skip`, но сравнение без учёта регистра (только ASCII).

```nim
import std/parseutils

doAssert skipIgnoreCase("CAPlow", "cap") == 3
doAssert skipIgnoreCase("CAPlow", "CAP") == 3
doAssert skipIgnoreCase("Hello", "HELL") == 4
doAssert skipIgnoreCase("Hello", "world") == 0
```

---

### `skipUntil(s, until: set[char]): int`

**Сигнатуры:**
```nim
proc skipUntil(s: openArray[char], until: set[char]): int
proc skipUntil(s: string, until: set[char], start = 0): int
```

Пропускает символы, пока не встретится любой символ из набора `until` или конец строки.

```nim
import std/parseutils

doAssert skipUntil("Hello World", {'W', 'e'}) == 1   # 'e' на позиции 1
doAssert skipUntil("Hello World", {'W'}) == 6         # 'W' на позиции 6
doAssert skipUntil("Hello World", {'z'}) == 11        # не найдено — до конца
```

---

### `skipUntil(s, until: char): int`

**Сигнатуры:**
```nim
proc skipUntil(s: openArray[char], until: char): int
proc skipUntil(s: string, until: char, start = 0): int
```

Как предыдущий вариант, но ищет один конкретный символ.

```nim
import std/parseutils

doAssert skipUntil("Hello World", 'o') == 4    # первая 'o'
doAssert skipUntil("Hello World", 'o', 4) == 0 # уже на 'o'
doAssert skipUntil("Hello World", 'W') == 6
doAssert skipUntil("Hello World", 'z') == 11   # не найдено
```

---

### `skipWhile(s, toSkip): int`

**Сигнатуры:**
```nim
proc skipWhile(s: openArray[char], toSkip: set[char]): int
proc skipWhile(s: string, toSkip: set[char], start = 0): int
```

Пропускает символы, пока они входят в набор `toSkip`.

```nim
import std/parseutils
from std/strutils import Digits, Letters

doAssert skipWhile("Hello World", {'H', 'e'}) == 2
doAssert skipWhile("Hello World", {'e'}) == 0  # 'H' не входит

# Практический пример: пропустить цифры
let s = "2019 school start"
let yearEnd = skipWhile(s, Digits)   # == 4
echo s[0 ..< yearEnd]   # "2019"

# С позиции
doAssert skipWhile("Hello World", {'W', 'o', 'r'}, 6) == 3  # "Wor"
```

---

## Захват токенов (parse)

### `parseUntil(s, token, until: set[char]): int`

**Сигнатуры:**
```nim
proc parseUntil(s: openArray[char], token: var string, until: set[char]): int
proc parseUntil(s: string, token: var string, until: set[char], start = 0): int
```

Читает символы в `token` до тех пор, пока не встретится символ из набора `until` или конец строки.

```nim
import std/parseutils

var tok: string

doAssert parseUntil("Hello World", tok, {'W', 'o', 'r'}) == 4
doAssert tok == "Hell"

doAssert parseUntil("Hello World", tok, {'W', 'r'}) == 6
doAssert tok == "Hello "

# С позиции 3
doAssert parseUntil("Hello World", tok, {'W', 'r'}, 3) == 3
doAssert tok == "lo "
```

---

### `parseUntil(s, token, until: char): int`

**Сигнатуры:**
```nim
proc parseUntil(s: openArray[char], token: var string, until: char): int
proc parseUntil(s: string, token: var string, until: char, start = 0): int
```

Читает символы в `token` до конкретного символа-ограничителя.

```nim
import std/parseutils

var tok: string

doAssert parseUntil("Hello World", tok, 'W') == 6
doAssert tok == "Hello "

doAssert parseUntil("Hello World", tok, 'o') == 4
doAssert tok == "Hell"

# С позиции 2
doAssert parseUntil("Hello World", tok, 'o', 2) == 2
doAssert tok == "ll"

# Практика: разобрать поля CSV вручную
let csv = "alice,30,engineer"
var field: string
var pos = 0
pos += parseUntil(csv, field, ',', pos); echo field; inc pos  # alice
pos += parseUntil(csv, field, ',', pos); echo field; inc pos  # 30
pos += parseUntil(csv, field, ',', pos); echo field           # engineer
```

---

### `parseUntil(s, token, until: string): int`

**Сигнатуры:**
```nim
proc parseUntil(s: openArray[char], token: var string, until: string): int
proc parseUntil(s: string, token: var string, until: string, start = 0): int
```

Читает символы в `token` до появления подстроки-ограничителя.

```nim
import std/parseutils

var tok: string

doAssert parseUntil("Hello World", tok, "Wor") == 6
doAssert tok == "Hello "

doAssert parseUntil("Hello World", tok, "Wor", 2) == 4
doAssert tok == "llo "

# Практика: вырезать содержимое HTML-тега
let html = "<b>Bold text</b>"
var content: string
var p = skip(html, "<b>")              # пропустить открывающий тег
discard parseUntil(html, content, "</b>", p)
echo content   # Bold text
```

---

### `parseWhile(s, token, validChars): int`

**Сигнатуры:**
```nim
proc parseWhile(s: openArray[char], token: var string, validChars: set[char]): int
proc parseWhile(s: string, token: var string, validChars: set[char], start = 0): int
```

Читает символы в `token`, пока они входят в набор `validChars`.

```nim
import std/parseutils
from std/strutils import Digits, Letters

var tok: string

doAssert parseWhile("Hello World", tok, {'W', 'o', 'r'}) == 0  # 'H' не входит
doAssert parseWhile("Hello World", tok, {'W', 'o', 'r'}, 6) == 3
doAssert tok == "Wor"

# Выделить слово из строки
doAssert parseWhile("hello123 rest", tok, Letters + Digits) == 8
doAssert tok == "hello123"

# Разбор числа
doAssert parseWhile("  42px", tok, Digits, 2) == 2
doAssert tok == "42"
```

---

### `captureBetween(s, first, second='\0'): string`

**Сигнатуры:**
```nim
proc captureBetween(s: openArray[char], first: char, second = '\0'): string
proc captureBetween(s: string, first: char, second = '\0', start = 0): string
```

Находит первое вхождение `first`, затем возвращает всё до `second`. Если `second == '\0'`, ищет второе вхождение `first`.

```nim
import std/parseutils

# До второго 'e'
doAssert captureBetween("Hello World", 'e') == "llo World"

# Между 'e' и 'r'
doAssert captureBetween("Hello World", 'e', 'r') == "llo Wo"

# С позиции 6: между 'l' и следующим 'l'
doAssert captureBetween("Hello World", 'l', start = 6) == "d"

# Практика: содержимое скобок
doAssert captureBetween("func(arg1, arg2)", '(', ')') == "arg1, arg2"

# Содержимое кавычек
doAssert captureBetween("""say "hello" now""", '"') == "hello"
```

---

## Интерполяция строк

### `interpolatedFragments(s): (InterpolatedKind, string)`

**Сигнатуры:**
```nim
iterator interpolatedFragments(s: openArray[char]): tuple[kind: InterpolatedKind, value: string]
iterator interpolatedFragments(s: string): tuple[kind: InterpolatedKind, value: string]
```

Разбивает строку на фрагменты для интерполяции в стиле `$var` / `${expr}` / `$$`. Бросает `ValueError` при незакрытой скобке `${` или недопустимом использовании `$`.

| Паттерн      | Тип       | Значение в `value`   |
|--------------|-----------|----------------------|
| `$name`      | `ikVar`   | `name`               |
| `${expr}`    | `ikExpr`  | `expr` (без `{}`)    |
| `$$`         | `ikDollar`| `$`                  |
| прочее       | `ikStr`   | сам фрагмент         |

```nim
import std/parseutils

var parts: seq[tuple[kind: InterpolatedKind, value: string]]
for k, v in interpolatedFragments("  $this is ${an  example}  $$"):
  parts.add (k, v)

doAssert parts == @[
  (ikStr,    "  "),
  (ikVar,    "this"),
  (ikStr,    " is "),
  (ikExpr,   "an  example"),
  (ikStr,    "  "),
  (ikDollar, "$")
]

# Реализация простого шаблонизатора
import std/tables
let tpl = "Hello, $name! You have $count messages."
let vars = {"name": "Alice", "count": "5"}.toTable
var result = ""
for kind, val in interpolatedFragments(tpl):
  case kind
  of ikStr:    result.add val
  of ikVar:    result.add vars.getOrDefault(val, "???")
  of ikExpr:   result.add vars.getOrDefault(val, "???")
  of ikDollar: result.add "$"
echo result   # Hello, Alice! You have 5 messages.
```

---

## Практические рецепты

### Ручной парсер с цепочечными вызовами

```nim
import std/parseutils

proc parseDate(s: string): tuple[y, m, d: int] =
  var pos = 0
  var num: int
  pos += parseInt(s, num, pos); result.y = num
  pos += skip(s, "-", pos)
  pos += parseInt(s, num, pos); result.m = num
  pos += skip(s, "-", pos)
  pos += parseInt(s, num, pos); result.d = num

let date = parseDate("2024-03-15")
echo date   # (y: 2024, m: 3, d: 15)
```

### Фильтрация строк по шаблону дат

```nim
import std/parseutils

let logs = @["2019-01-10: OK_", "2019-01-11: FAIL_", "2019-01: aaaa"]
var outp: seq[string]

for log in logs:
  var res: string
  if parseUntil(log, res, ':') == 10:   # YYYY-MM-DD == 10 символов
    outp.add(res & " - " & captureBetween(log, ' ', '_'))

doAssert outp == @["2019-01-10 - OK", "2019-01-11 - FAIL"]
```

### Выделение числа из строки

```nim
import std/parseutils
from std/strutils import Digits, parseInt

let input1 = "2019 school start"
let input2 = "3 years back"
let startYear = input1[0 ..< skipWhile(input1, Digits)]  # "2019"
let yearsBack = input2[0 ..< skipWhile(input2, Digits)]  # "3"
let examYear = parseInt(startYear) + parseInt(yearsBack)
doAssert "Examination is in " & $examYear == "Examination is in 2022"
```

### Разбор конфигурационной строки `key=value`

```nim
import std/parseutils

proc parseKV(s: string): tuple[key, value: string] =
  var pos = 0
  pos += skipWhitespace(s, pos)
  pos += parseIdent(s, result.key, pos)
  pos += skipWhitespace(s, pos)
  pos += skip(s, "=", pos)
  pos += skipWhitespace(s, pos)
  discard parseUntil(s, result.value, {'\n', '#'}, pos)

let kv = parseKV("  timeout = 30  # seconds")
echo kv.key    # timeout
echo kv.value  # 30  
```
