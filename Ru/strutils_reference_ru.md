# Справочник модуля `std/strutils` (Nim)

> Модуль `strutils` предоставляет процедуры, итераторы и шаблоны для работы со строками ASCII.
> Доступен для JavaScript-бэкенда. Для поддержки Unicode используйте модуль `std/unicode`.

---

## Оглавление

1. [Константы — наборы символов](#константы--наборы-символов)
2. [Проверка символов](#проверка-символов)
3. [Изменение регистра](#изменение-регистра)
4. [Нормализация и сравнение](#нормализация-и-сравнение)
5. [Разбиение строк (split)](#разбиение-строк-split)
6. [Разбиение с конца (rsplit)](#разбиение-с-конца-rsplit)
7. [Разбиение по строкам и пробелам](#разбиение-по-строкам-и-пробелам)
8. [Конвертация чисел в строки](#конвертация-чисел-в-строки)
9. [Парсинг строк в числа и булевы значения](#парсинг-строк-в-числа-и-булевы-значения)
10. [Повторение и выравнивание](#повторение-и-выравнивание)
11. [Отступы и форматирование блоков](#отступы-и-форматирование-блоков)
12. [Обрезка строк (strip)](#обрезка-строк-strip)
13. [Проверки начала и конца строки](#проверки-начала-и-конца-строки)
14. [Удаление префиксов и суффиксов](#удаление-префиксов-и-суффиксов)
15. [Поиск в строках](#поиск-в-строках)
16. [Подсчёт вхождений](#подсчёт-вхождений)
17. [Замена подстрок](#замена-подстрок)
18. [Объединение строк (join)](#объединение-строк-join)
19. [Форматирование чисел с плавающей точкой](#форматирование-чисел-с-плавающей-точкой)
20. [Форматирование строк (`%` и `format`)](#форматирование-строк--и-format)
21. [Прочие утилиты](#прочие-утилиты)
22. [SkipTable — оптимизированный поиск](#skiptable--оптимизированный-поиск)
23. [Итератор tokenize](#итератор-tokenize)

---

## Константы — наборы символов

```nim
Whitespace*      = {' ', '\t', '\v', '\r', '\l', '\f'}
Letters*         = {'A'..'Z', 'a'..'z'}
UppercaseLetters* = {'A'..'Z'}
LowercaseLetters* = {'a'..'z'}
Digits*          = {'0'..'9'}
HexDigits*       = {'0'..'9', 'A'..'F', 'a'..'f'}
PunctuationChars* = {'!'..'/', ':'..'@', '['..'`', '{'..'~'}
IdentChars*      = {'a'..'z', 'A'..'Z', '0'..'9', '_'}
IdentStartChars* = {'a'..'z', 'A'..'Z', '_'}
Newlines*        = {'\13', '\10'}
PrintableChars*  = Letters + Digits + PunctuationChars + Whitespace
AllChars*        = {'\x00'..'\xFF'}
```

`AllChars` удобен для создания инвертированных наборов при поиске некорректных символов:

```nim
let invalid = AllChars - Digits
doAssert "01234".find(invalid) == -1
doAssert "01A34".find(invalid) == 2
```

---

## Проверка символов

### `isAlphaAscii`
```nim
func isAlphaAscii(c: char): bool
```
Возвращает `true`, если символ является буквой A–Z или a–z.

```nim
doAssert isAlphaAscii('e') == true
doAssert isAlphaAscii('E') == true
doAssert isAlphaAscii('8') == false
```

---

### `isAlphaNumeric`
```nim
func isAlphaNumeric(c: char): bool
```
Возвращает `true`, если символ является буквой или цифрой (A–Z, a–z, 0–9).

```nim
doAssert isAlphaNumeric('n') == true
doAssert isAlphaNumeric('8') == true
doAssert isAlphaNumeric(' ') == false
```

---

### `isDigit`
```nim
func isDigit(c: char): bool
```
Возвращает `true`, если символ является цифрой 0–9.

```nim
doAssert isDigit('n') == false
doAssert isDigit('8') == true
```

---

### `isSpaceAscii`
```nim
func isSpaceAscii(c: char): bool
```
Возвращает `true`, если символ является пробельным (пробел, таб, и т.д.).

```nim
doAssert isSpaceAscii('n') == false
doAssert isSpaceAscii(' ') == true
doAssert isSpaceAscii('\t') == true
```

---

### `isLowerAscii`
```nim
func isLowerAscii(c: char): bool
```
Возвращает `true`, если символ является строчной буквой ASCII.

```nim
doAssert isLowerAscii('e') == true
doAssert isLowerAscii('E') == false
doAssert isLowerAscii('7') == false
```

---

### `isUpperAscii`
```nim
func isUpperAscii(c: char): bool
```
Возвращает `true`, если символ является заглавной буквой ASCII.

```nim
doAssert isUpperAscii('e') == false
doAssert isUpperAscii('E') == true
doAssert isUpperAscii('7') == false
```

---

### `isEmptyOrWhitespace`
```nim
func isEmptyOrWhitespace(s: string): bool
```
Возвращает `true`, если строка пустая или состоит только из пробельных символов.

```nim
doAssert isEmptyOrWhitespace("") == true
doAssert isEmptyOrWhitespace("   ") == true
doAssert isEmptyOrWhitespace("  a ") == false
```

---

### `allCharsInSet`
```nim
func allCharsInSet(s: string, theSet: set[char]): bool
```
Возвращает `true`, если все символы строки входят в набор `theSet`. Пустая строка возвращает `true`.

```nim
doAssert allCharsInSet("aeea", {'a', 'e'}) == true
doAssert allCharsInSet("", {'a', 'e'}) == true
doAssert allCharsInSet("aeba", {'a', 'e'}) == false
```

---

### `validIdentifier`
```nim
func validIdentifier(s: string): bool
```
Возвращает `true`, если строка является валидным Nim-идентификатором (начинается с `IdentStartChars`, продолжается `IdentChars`).

```nim
doAssert "abc_def08".validIdentifier == true
doAssert "0abc".validIdentifier == false
doAssert "".validIdentifier == false
```

---

## Изменение регистра

### `toLowerAscii`
```nim
func toLowerAscii(c: char): char
func toLowerAscii(s: string): string
```
Переводит символ или строку в нижний регистр (только A–Z → a–z).

```nim
doAssert toLowerAscii('A') == 'a'
doAssert toLowerAscii('e') == 'e'
doAssert toLowerAscii("FooBar!") == "foobar!"
```

---

### `toUpperAscii`
```nim
func toUpperAscii(c: char): char
func toUpperAscii(s: string): string
```
Переводит символ или строку в верхний регистр (только a–z → A–Z).

```nim
doAssert toUpperAscii('a') == 'A'
doAssert toUpperAscii('E') == 'E'
doAssert toUpperAscii("FooBar!") == "FOOBAR!"
```

---

### `capitalizeAscii`
```nim
func capitalizeAscii(s: string): string
```
Переводит первый символ строки в верхний регистр. Работает только для A–Z.

```nim
doAssert capitalizeAscii("foo") == "Foo"
doAssert capitalizeAscii("-bar") == "-bar"
```

---

## Нормализация и сравнение

### `normalize`
```nim
func normalize(s: string): string
```
Переводит строку в нижний регистр и удаляет все символы `_`. Не предназначен для нормализации Nim-идентификаторов.

```nim
doAssert normalize("Foo_bar") == "foobar"
doAssert normalize("Foo Bar") == "foo bar"
```

---

### `nimIdentNormalize`
```nim
func nimIdentNormalize(s: string): string
```
Нормализует строку как Nim-идентификатор: переводит в нижний регистр и убирает `_` (кроме первого символа).

```nim
doAssert nimIdentNormalize("Foo_bar") == "Foobar"
```

---

### `cmpIgnoreCase`
```nim
func cmpIgnoreCase(a, b: string): int
```
Сравнивает две строки без учёта регистра. Возвращает `0` если равны, `< 0` если `a < b`, `> 0` если `a > b`.

```nim
doAssert cmpIgnoreCase("FooBar", "foobar") == 0
doAssert cmpIgnoreCase("bar", "Foo") < 0
doAssert cmpIgnoreCase("Foo5", "foo4") > 0
```

---

### `cmpIgnoreStyle`
```nim
func cmpIgnoreStyle(a, b: string): int
```
Сравнивает строки без учёта стиля (регистр + символы `_`). Эквивалентно `cmp(normalize(a), normalize(b))`, но без выделения памяти. Не использовать для Nim-идентификаторов.

```nim
doAssert cmpIgnoreStyle("foo_bar", "FooBar") == 0
doAssert cmpIgnoreStyle("foo_bar_5", "FooBar4") > 0
```

---

## Разбиение строк (split)

Все функции `split` имеют три перегрузки: по символу, набору символов или строке. Существуют как итераторы, так и функции-аналоги возвращающие `seq[string]`.

### `split` (итератор)
```nim
iterator split(s: string, sep: char, maxsplit: int = -1): string
iterator split(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): string
iterator split(s: string, sep: string, maxsplit: int = -1): string
```
Разбивает строку `s` на подстроки. `maxsplit` ограничивает количество разбиений (-1 = без ограничений). Пустой разделитель возвращает исходную строку целиком.

```nim
for word in split("this is an example"):
  echo word  # "this", "is", "an", "example"

for word in split("a,b,,c", ','):
  echo word  # "a", "b", "", "c"

for word in split("aXbYc", {'X', 'Y'}):
  echo word  # "a", "b", "c"
```

---

### `split` (func)
```nim
func split(s: string, sep: char, maxsplit: int = -1): seq[string]
func split(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): seq[string]
func split(s: string, sep: string, maxsplit: int = -1): seq[string]
```
Аналог итератора, но возвращает `seq[string]`.

```nim
doAssert "a,b,c".split(',') == @["a", "b", "c"]
doAssert "".split(' ') == @[""]
doAssert "a,b;c".split({',', ';'}) == @["a", "b", "c"]
doAssert "a,b,c".split(",") == @["a", "b", "c"]
doAssert "a  spaced sentence".split(" ", maxsplit = 1) ==
  @["a", " spaced sentence"]
doAssert "empty seps return unsplit s".split({}) == @["empty seps return unsplit s"]
```

---

## Разбиение с конца (rsplit)

### `rsplit` (итератор)
```nim
iterator rsplit(s: string, sep: char, maxsplit: int = -1): string
iterator rsplit(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): string
iterator rsplit(s: string, sep: string, maxsplit: int = -1, keepSeparators: bool = false): string
```
Аналог `split`, но разбивает строку **с конца**. Подстроки выдаются в обратном порядке.

```nim
for piece in "foo:bar".rsplit(':'):
  echo piece  # "bar", затем "foo"
```

---

### `rsplit` (func)
```nim
func rsplit(s: string, sep: char, maxsplit: int = -1): seq[string]
func rsplit(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): seq[string]
func rsplit(s: string, sep: string, maxsplit: int = -1): seq[string]
```
Аналог итератора, возвращает результат в **исходном** порядке (не обратном).

```nim
doAssert "a  largely    spaced sentence".rsplit(" ", maxsplit = 1) ==
  @["a  largely    spaced", "sentence"]
doAssert "a,b,c".rsplit(",") == @["a", "b", "c"]
# Типичный use case: разбор пути
var tailSplit = rsplit("Root#Object#Method#Index", '#', maxsplit=1)
# @["Root#Object#Method", "Index"]
```

---

## Разбиение по строкам и пробелам

### `splitLines` (итератор и func)
```nim
iterator splitLines(s: string, keepEol = false): string
func splitLines(s: string, keepEol = false): seq[string]
```
Разбивает строку на строки. Поддерживает CR, LF, CRLF. Если `keepEol = true`, символы конца строки сохраняются.

```nim
assert splitLines("first line\nsecond line\nthird line") ==
  @["first line", "second line", "third line"]

for line in splitLines("\nfoo\nbar\n"):
  echo line  # "", "foo", "bar", ""
```

---

### `splitWhitespace` (итератор и func)
```nim
iterator splitWhitespace(s: string, maxsplit: int = -1): string
func splitWhitespace(s: string, maxsplit: int = -1): seq[string]
```
Разбивает по пробельным символам, отбрасывая ведущие и хвостовые пробелы.

```nim
let s = "  foo \t bar  baz  "
doAssert s.splitWhitespace() == @["foo", "bar", "baz"]
doAssert s.splitWhitespace(maxsplit = 1) == @["foo", "bar  baz  "]
```

---

### `countLines`
```nim
func countLines(s: string): int
```
Возвращает количество строк (эффективнее чем `len(splitLines(s))`).

```nim
doAssert countLines("First line\l and second line.") == 2
doAssert countLines("a\nb\nc") == 3
```

---

## Конвертация чисел в строки

### `toBin`
```nim
func toBin(x: BiggestInt, len: Positive): string
```
Преобразует число в двоичное представление (строка длиной `len`, без префикса `0b`).

```nim
doAssert 29.toBin(8) == "00011101"
doAssert 257.toBin(9) == "100000001"
```

---

### `toOct`
```nim
func toOct(x: BiggestInt, len: Positive): string
```
Преобразует число в восьмеричное представление (строка длиной `len`, без префикса `0o`).

```nim
doAssert 62.toOct(3) == "076"
doAssert 513.toOct(5) == "01001"
```

---

### `toOctal`
```nim
func toOctal(c: char): string
```
Преобразует символ в восьмеричное представление. Всегда возвращает строку длиной 3.

```nim
doAssert toOctal('1') == "061"
doAssert toOctal('A') == "101"
doAssert toOctal('a') == "141"
```

---

### `toHex` (числа)
```nim
func toHex[T: SomeInteger](x: T, len: Positive): string
func toHex[T: SomeInteger](x: T): string  # = toHex(x, T.sizeof * 2)
```
Преобразует число в шестнадцатеричное представление (без префикса `0x`). Обрабатывает как беззнаковое.

```nim
doAssert 62'u64.toHex(3) == "03E"
doAssert toHex(-8, 6) == "FFFFF8"
doAssert toHex(1984'i64) == "00000000000007C0"
doAssert toHex(1984'i16) == "07C0"
```

---

### `toHex` (строка)
```nim
func toHex(s: string): string
```
Преобразует байтовую строку в шестнадцатеричное представление. Результат в два раза длиннее входной строки.

```nim
doAssert "1".toHex() == "31"
doAssert "A".toHex() == "41"
doAssert "\0\255".toHex() == "00FF"
```

---

### `intToStr`
```nim
func intToStr(x: int, minchars: Positive = 1): string
```
Преобразует целое число в строку. Добавляет ведущие нули до минимальной длины `minchars`.

```nim
doAssert intToStr(1984) == "1984"
doAssert intToStr(1984, 6) == "001984"
doAssert intToStr(-42, 5) == "-0042"
```

---

### `insertSep`
```nim
func insertSep(s: string, sep = '_', digits = 3): string
```
Вставляет разделитель `sep` каждые `digits` символов справа (полезно для форматирования чисел).

```nim
doAssert insertSep("1000000") == "1_000_000"
doAssert insertSep("1000000", ',') == "1,000,000"
```

---

## Парсинг строк в числа и булевы значения

Все функции парсинга выбрасывают `ValueError` при некорректном вводе.

### `parseInt`
```nim
func parseInt(s: string): int
```
```nim
doAssert parseInt("-0042") == -42
```

### `parseBiggestInt`
```nim
func parseBiggestInt(s: string): BiggestInt
```

### `parseUInt`
```nim
func parseUInt(s: string): uint
```

### `parseBiggestUInt`
```nim
func parseBiggestUInt(s: string): BiggestUInt
```

### `parseFloat`
```nim
func parseFloat(s: string): float
```
Поддерживает `NAN`, `INF`, `-INF` (регистронезависимо).

```nim
doAssert parseFloat("3.14") == 3.14
doAssert parseFloat("inf") == 1.0/0
```

---

### `parseBinInt`
```nim
func parseBinInt(s: string): int
```
Парсит двоичную строку. Поддерживает префиксы `0b`, `0B` и символ `_`.

```nim
doAssert "0b11_0101".parseBinInt() == 53
doAssert "111".parseBinInt() == 7
```

---

### `parseOctInt`
```nim
func parseOctInt(s: string): int
```
Парсит восьмеричную строку. Поддерживает префиксы `0o`, `0O` и символ `_`.

---

### `parseHexInt`
```nim
func parseHexInt(s: string): int
```
Парсит шестнадцатеричную строку. Поддерживает префиксы `0x`, `0X`, `#` и символ `_`.

---

### `fromBin` / `fromOct` / `fromHex`
```nim
func fromBin[T: SomeInteger](s: string): T
func fromOct[T: SomeInteger](s: string): T
func fromHex[T: SomeInteger](s: string): T
```
Обобщённые версии парсеров с явным указанием типа результата. Не проверяют переполнение.

```nim
doAssert fromBin[int]("0b_0100_1000") == 72
doAssert fromOct[int]("0o777") == 511
doAssert fromHex[int]("0xFF") == 255
doAssert fromHex[int8]("0xFF") == -1'i8   # переполнение без ошибки
```

---

### `parseHexStr`
```nim
func parseHexStr(s: string): string
```
Преобразует hex-строку обратно в байтовую строку. Регистронезависимо.

```nim
doAssert parseHexStr("41") == "A"
doAssert parseHexStr("00ff") == "\0\255"
```

---

### `parseBool`
```nim
func parseBool(s: string): bool
```
Парсит булево значение. Принимает: `y, yes, true, 1, on` → `true`; `n, no, false, 0, off` → `false`. Регистронезависимо (нормализует через `normalize`).

```nim
doAssert parseBool("yes") == true
doAssert parseBool("False") == false
doAssert parseBool("on") == true
```

---

### `parseEnum`
```nim
func parseEnum[T: enum](s: string): T
func parseEnum[T: enum](s: string, default: T): T
```
Парсит значение перечисления. Сравнение выполняется в стиле Nim (первый символ с учётом регистра, остальное — без + игнорирует `_`). При ошибке: первая перегрузка выбрасывает `ValueError`, вторая возвращает `default`.

```nim
type Color = enum red, green, blue

doAssert parseEnum[Color]("red") == red
doAssert parseEnum[Color]("GREEN") == green
doAssert parseEnum[Color]("unknown", blue) == blue
```

---

## Повторение и выравнивание

### `repeat`
```nim
func repeat(c: char, count: Natural): string
func repeat(s: string, n: Natural): string
```
Повторяет символ `count` раз или строку `n` раз.

```nim
doAssert 'z'.repeat(5) == "zzzzz"
doAssert "+ foo +".repeat(3) == "+ foo ++ foo ++ foo +"
```

---

### `spaces`
```nim
func spaces(n: Natural): string
```
Возвращает строку из `n` пробелов. Удобно для выравнивания.

```nim
let width = 15
let text = "Hello user!"
doAssert text & spaces(max(0, width - text.len)) & "|" == "Hello user!    |"
```

---

### `align`
```nim
func align(s: string, count: Natural, padding = ' '): string
```
Выравнивает строку по правому краю до длины `count`, добавляя символ `padding` слева.

```nim
assert align("abc", 4) == " abc"
assert align("1232", 6) == "  1232"
assert align("1232", 6, '#') == "##1232"
assert align("toolong", 3) == "toolong"  # без изменений
```

---

### `alignLeft`
```nim
func alignLeft(s: string, count: Natural, padding = ' '): string
```
Выравнивает строку по левому краю до длины `count`, добавляя `padding` справа.

```nim
assert alignLeft("abc", 4) == "abc "
assert alignLeft("1232", 6) == "1232  "
assert alignLeft("1232", 6, '#') == "1232##"
```

---

### `center`
```nim
func center(s: string, width: int, fillChar: char = ' '): string
```
Центрирует строку в поле шириной `width`, заполняя символом `fillChar`. При нечётном количестве пустых символов дополнительный попадает справа.

```nim
doAssert "foo".center(5) == " foo "
doAssert "foo".center(6) == " foo  "
doAssert "foo".center(2) == "foo"   # без изменений
```

---

## Отступы и форматирование блоков

### `indent`
```nim
func indent(s: string, count: Natural, padding: string = " "): string
```
Добавляет `count * padding` в начало каждой строки в `s`.

```nim
doAssert indent("First line\nsecond line.", 2) == "  First line\n  second line."
doAssert indent("A\nB", 1, ">>") == ">>A\n>>B"
```

---

### `unindent`
```nim
func unindent(s: string, count: Natural = int.high, padding: string = " "): string
```
Убирает не более `count` экземпляров `padding` в начале каждой строки.

```nim
let x = """
  Hello
    There
""".unindent()
doAssert x == "Hello\nThere\n"
```

---

### `indentation`
```nim
func indentation(s: string): Natural
```
Возвращает количество пробелов, общих для всех непустых строк в `s`.

```nim
doAssert indentation("  foo\n  bar") == 2
doAssert indentation("  foo\n    bar") == 2
```

---

### `dedent`
```nim
func dedent(s: string, count: Natural = indentation(s)): string
```
Убирает общий отступ всех строк в `s` (только пробелы). По умолчанию убирает ровно столько, сколько общего отступа у всех строк.

```nim
let x = """
  Hello
    There
""".dedent()
doAssert x == "Hello\n  There\n"
```

---

### `delete`
```nim
func delete(s: var string, slice: Slice[int])
```
Удаляет символы по срезу `slice` из строки `s` (на месте). Выбрасывает `IndexDefect` при выходе за границы.

```nim
var a = "abcde"
a.delete(1..2)
assert a == "ade"

a.delete(1..<1)  # пустой срез — без изменений
assert a == "ade"
```

---

## Обрезка строк (strip)

### `strip`
```nim
func strip(s: string, leading = true, trailing = true,
           chars: set[char] = Whitespace): string
```
Удаляет символы из набора `chars` с начала (`leading`) и/или конца (`trailing`) строки. По умолчанию — пробельные символы с обоих концов.

```nim
doAssert strip("  vhellov   ") == "vhellov"
doAssert "  vhellov   ".strip(leading = false) == "  vhellov"
doAssert "  vhellov   ".strip(trailing = false) == "vhellov   "
doAssert "vhellov".strip(chars = {'v'}) == "hello"
```

---

### `stripLineEnd`
```nim
func stripLineEnd(s: var string)
```
Удаляет **один** символ конца строки (`\r`, `\n`, `\r\n`, `\f`, `\v`) с конца строки (на месте). Аналог `chomp` в других языках.

```nim
var s = "foo\n\n"
s.stripLineEnd
doAssert s == "foo\n"

s = "foo\r\n"
s.stripLineEnd
doAssert s == "foo"
```

---

## Проверки начала и конца строки

### `startsWith`
```nim
func startsWith(s: string, prefix: char): bool
func startsWith(s, prefix: string): bool
```
Возвращает `true`, если строка начинается с заданного символа или строки. Пустой `prefix` всегда возвращает `true`.

```nim
let a = "abracadabra"
doAssert a.startsWith('a') == true
doAssert a.startsWith("abra") == true
doAssert a.startsWith("bra") == false
```

---

### `endsWith`
```nim
func endsWith(s: string, suffix: char): bool
func endsWith(s, suffix: string): bool
```
Возвращает `true`, если строка заканчивается заданным символом или строкой.

```nim
let a = "abracadabra"
doAssert a.endsWith('a') == true
doAssert a.endsWith("abra") == true
doAssert a.endsWith("dab") == false
```

---

### `continuesWith`
```nim
func continuesWith(s, substr: string, start: Natural): bool
```
Возвращает `true`, если строка `s` содержит `substr`, начиная с позиции `start`.

```nim
let a = "abracadabra"
doAssert a.continuesWith("ca", 4) == true
doAssert a.continuesWith("ca", 5) == false
doAssert a.continuesWith("dab", 6) == true
```

---

## Удаление префиксов и суффиксов

### `removePrefix`
```nim
func removePrefix(s: var string, chars: set[char] = Newlines)
func removePrefix(s: var string, c: char)
func removePrefix(s: var string, prefix: string)
```
Удаляет **все** символы из набора / один символ / первый вхождение строки-префикса с начала строки (на месте).

```nim
var s = "\r\n*~Hello World!"
s.removePrefix           # удаляет символы из Newlines
doAssert s == "*~Hello World!"

s.removePrefix({'~', '*'})
doAssert s == "Hello World!"

var answers = "yesyes"
answers.removePrefix("yes")
doAssert answers == "yes"
```

---

### `removeSuffix`
```nim
func removeSuffix(s: var string, chars: set[char] = Newlines)
func removeSuffix(s: var string, c: char)
func removeSuffix(s: var string, suffix: string)
```
Аналог `removePrefix`, но работает с конца строки.

```nim
var s = "Hello World!*~\r\n"
s.removeSuffix
doAssert s == "Hello World!*~"

var table = "users"
table.removeSuffix('s')
doAssert table == "user"

var answers = "yeses"
answers.removeSuffix("es")
doAssert answers == "yes"
```

---

## Поиск в строках

### `find`
```nim
func find(s: string, sub: char, start: Natural = 0, last = -1): int
func find(s: string, chars: set[char], start: Natural = 0, last = -1): int
func find(s, sub: string, start: Natural = 0, last = -1): int
func find(a: SkipTable, s, sub: string, start: Natural = 0, last = -1): int
```
Ищет первое вхождение символа, набора символов или подстроки в диапазоне `start..last`. Возвращает индекс относительно `s[0]` или -1 если не найдено. Чувствителен к регистру.

```nim
doAssert "abcdef".find('c') == 2
doAssert "abcdef".find({'c', 'e'}) == 2
doAssert "abcdef".find("cd") == 2
doAssert "abcdef".find('z') == -1
doAssert "abcabc".find('c', start = 3) == 5  # поиск с позиции 3
```

---

### `rfind`
```nim
func rfind(s: string, sub: char, start: Natural = 0, last = -1): int
func rfind(s: string, chars: set[char], start: Natural = 0, last = -1): int
func rfind(s, sub: string, start: Natural = 0, last = -1): int
```
Ищет **последнее** вхождение, сканируя строку с конца (от `last` к `start`).

```nim
doAssert "abcabc".rfind('c') == 5
doAssert "abcabc".rfind("bc") == 4
```

---

### `contains`
```nim
func contains(s, sub: string): bool
func contains(s: string, chars: set[char]): bool
```
Возвращает `true` если подстрока или символ из набора присутствует в строке. Аналог `find(s, sub) >= 0`.

```nim
doAssert "hello world".contains("world") == true
doAssert "hello".contains({'x', 'y'}) == false
```

---

## Подсчёт вхождений

### `count`
```nim
func count(s: string, sub: char): int
func count(s: string, subs: set[char]): int
func count(s: string, sub: string, overlapping: bool = false): int
```
Подсчитывает вхождения символа, набора символов или подстроки. При `overlapping = true` считаются перекрывающиеся вхождения.

```nim
doAssert "aababc".count('a') == 3
doAssert "aababc".count({'a', 'b'}) == 5
doAssert "aababc".count("ab") == 2
doAssert "aaa".count("aa") == 1           # без перекрытий
doAssert "aaa".count("aa", overlapping = true) == 2
```

---

## Замена подстрок

### `replace`
```nim
func replace(s, sub: string, by = ""): string
func replace(s: string, sub, by: char): string
```
Заменяет **все** вхождения `sub` на `by`. Версия для символов оптимизирована.

```nim
doAssert "Hello World".replace("World", "Nim") == "Hello Nim"
doAssert "aabbcc".replace('b', 'x') == "aaxxcc"
doAssert "a,b,c".replace(",") == "abc"  # удаление
```

---

### `replaceWord`
```nim
func replaceWord(s, sub: string, by = ""): string
```
Заменяет `sub` только если оно окружено границами слова (аналог `\b` в regexp).

```nim
# "bar" не заменяется внутри "foobar"
doAssert "foo bar foobar".replaceWord("bar", "baz") == "foo baz foobar"
```

---

### `multiReplace` (строки)
```nim
func multiReplace(s: string, replacements: varargs[(string, string)]): string
```
Выполняет несколько замен за один проход. При нескольких совпадениях применяется первое по порядку.

```nim
doAssert "abba".multiReplace([("a", "b"), ("b", "a")]) == "baab"
doAssert "abc".multiReplace([("bc", "x"), ("ab", "_b")]) == "_bc"
```

---

### `multiReplace` (наборы символов)
```nim
func multiReplace(s: openArray[char]; replacements: varargs[(set[char], char)]): string
```
Выполняет замену символов по наборам за один проход. Первое подходящее правило применяется.

```nim
const WinRules = [
  ({'\0'..'\31'}, ' '),
  ({'"'}, '\''),
  ({'/', '\\', ':', '|'}, '-'),
  ({'*', '?', '<', '>'}, '_'),
]
doAssert "a/file:with?invalid*chars.txt".multiReplace(WinRules) ==
  "a-file-with_invalid_chars.txt"
```

---

## Объединение строк (join)

### `join`
```nim
func join(a: openArray[string], sep: string = ""): string
proc join[T: not string](a: openArray[T], sep: string = ""): string
```
Объединяет элементы массива в строку, разделяя их `sep`. Вторая перегрузка конвертирует элементы через `$`.

```nim
doAssert join(["A", "B", "C"], " -> ") == "A -> B -> C"
doAssert join([1, 2, 3], ", ") == "1, 2, 3"
doAssert join(["a", "b", "c"]) == "abc"
```

---

## Форматирование чисел с плавающей точкой

### `FloatFormatMode`
```nim
type FloatFormatMode* = enum
  ffDefault    # краткая форма
  ffDecimal    # десятичная нотация
  ffScientific # научная нотация (e)
```

### `formatFloat`
```nim
func formatFloat(f: float, format: FloatFormatMode = ffDefault,
                 precision: range[-1..32] = 16; decimalSep = '.'): string
```
Преобразует `float` в строку. При `precision == -1` выбирается наиболее компактная форма.

```nim
doAssert 123.456.formatFloat() == "123.4560000000000"
doAssert 123.456.formatFloat(ffDecimal, 4) == "123.4560"
doAssert 123.456.formatFloat(ffScientific, 2) == "1.23e+02"
```

---

### `formatBiggestFloat`
```nim
func formatBiggestFloat(f: BiggestFloat, format: FloatFormatMode = ffDefault,
                        precision: range[-1..32] = 16; decimalSep = '.'): string
```
Аналог `formatFloat` для типа `BiggestFloat`.

---

### `trimZeros`
```nim
func trimZeros(x: var string; decimalSep = '.')
```
Удаляет хвостовые нули у строкового представления числа с плавающей точкой (на месте).

```nim
var x = "123.456000000"
x.trimZeros()
doAssert x == "123.456"

var y = "1.20e+03"
y.trimZeros()
doAssert y == "1.2e+03"
```

---

### `formatSize`
```nim
func formatSize(bytes: int64, decimalSep = '.', prefix = bpIEC,
                includeSpace = false): string
```
Форматирует размер в байтах в человекочитаемую форму. `bpIEC` использует стандарт IEC (KiB, MiB...), `bpColloquial` — разговорные единицы (kB, MB...).

```nim
doAssert formatSize(4096) == "4KiB"
doAssert formatSize(4096, includeSpace = true) == "4 KiB"
doAssert formatSize(4096, prefix = bpColloquial, includeSpace = true) == "4 kB"
doAssert formatSize((1'i64 shl 31) + (300'i64 shl 20)) == "2.293GiB"
```

---

### `formatEng`
```nim
func formatEng(f: BiggestFloat, precision: range[0..32] = 10,
               trim: bool = true, siPrefix: bool = false,
               unit: string = "", decimalSep = '.', useUnitSpace = false): string
```
Форматирует число в инженерной нотации (экспонента кратна 3). Поддерживает SI-префиксы.

```nim
doAssert formatEng(4100) == "4.1e3"
doAssert formatEng(4100, siPrefix = true, unit = "V") == "4.1 kV"
doAssert formatEng(0.053, 0) == "53e-3"
doAssert formatEng(52731234, 2) == "52.73e6"
```

---

## Форматирование строк (`%` и `format`)

### `%` (оператор форматирования)
```nim
func `%`(formatstr: string, a: openArray[string]): string
func `%`(formatstr, a: string): string
```
Выполняет подстановку в строке формата. Переменные обозначаются:
- `$1`, `$2`, ... — по позиции (с 1)
- `$#` — следующий аргумент
- `$$` — буквальный `$`
- `$name` / `${name}` — именованная переменная (чётные индексы = ключи, нечётные = значения)

```nim
doAssert "$1 eats $2." % ["The cat", "fish"] == "The cat eats fish."
doAssert "$# eats $#." % ["The cat", "fish"] == "The cat eats fish."
doAssert "$animal eats $food." % ["animal", "The cat", "food", "fish"] ==
  "The cat eats fish."
doAssert "value: $$42" % [] == "value: $42"
```

---

### `addf`
```nim
func addf(s: var string, formatstr: string, a: varargs[string, `$`])
```
Аналог `add(s, formatstr % a)`, но более эффективный (не создаёт промежуточную строку).

```nim
var result = "Numbers: "
result.addf("$1 and $2", "42", "7")
doAssert result == "Numbers: 42 and 7"
```

---

### `format` (форматирование)
```nim
func format(formatstr: string, a: varargs[string, `$`]): string
```
Аналог `formatstr % a` с поддержкой автоматической конвертации через `$`.

```nim
doAssert format("$1 + $2 = $3", 1, 2, 3) == "1 + 2 = 3"
```

---

## Прочие утилиты

### `addSep`
```nim
func addSep(dest: var string, sep = ", ", startLen: Natural = 0)
```
Добавляет разделитель `sep` к `dest`, если `dest.len > startLen`. Удобно для построения списков.

```nim
var arr = "["
for x in [2, 3, 5, 7, 11]:
  addSep(arr, startLen = len("["))
  arr.add($x)
arr.add("]")
doAssert arr == "[2, 3, 5, 7, 11]"
```

---

### `abbrev`
```nim
func abbrev(s: string, possibilities: openArray[string]): int
```
Возвращает индекс первого элемента из `possibilities`, который начинается с `s`. Возвращает `-1` если не найдено, `-2` если неоднозначно.

```nim
doAssert abbrev("fac", ["college", "faculty", "industry"]) == 1
doAssert abbrev("foo", ["college", "faculty", "industry"]) == -1
doAssert abbrev("fac", ["college", "faculty", "faculties"]) == -2
doAssert abbrev("college", ["college", "colleges"]) == 0  # точное совпадение
```

---

### `escape`
```nim
func escape(s: string, prefix = "\"", suffix = "\""): string
```
Экранирует строку: контрольные символы → `\xHH`, `\` → `\\`, `'` → `\'`, `"` → `\"`. Оборачивает в `prefix`/`suffix`.

```nim
doAssert escape("Hello\nWorld") == "\"Hello\\x0AWorld\""
doAssert escape("test", prefix = "", suffix = "") == "test"
```

---

### `unescape`
```nim
func unescape(s: string, prefix = "\"", suffix = "\""): string
```
Обратная операция к `escape`. Выбрасывает `ValueError`, если строка не начинается с `prefix` или не заканчивается `suffix`.

```nim
doAssert unescape("\"Hello\\x0AWorld\"") == "Hello\nWorld"
```

---

## SkipTable — оптимизированный поиск

`SkipTable` — предвычисленная таблица для алгоритма Бойера–Мура–Хорспула. Используйте при многократном поиске одной подстроки.

### `initSkipTable`
```nim
func initSkipTable(a: var SkipTable, sub: string)
func initSkipTable(sub: string): SkipTable
```
Инициализирует таблицу для подстроки `sub`.

### `find` (SkipTable)
```nim
func find(a: SkipTable, s, sub: string, start: Natural = 0, last = -1): int
```
Ищет `sub` в `s` используя предвычисленную таблицу.

```nim
let needle = "example"
let table = initSkipTable(needle)
let haystack = "This is an example string"

doAssert table.find(haystack, needle) == 11
doAssert table.find(haystack, needle, start = 12) == -1
```

---

## Итератор tokenize

### `tokenize`
```nim
iterator tokenize(s: string, seps: set[char] = Whitespace): tuple[token: string, isSep: bool]
```
Разбивает строку на токены и разделители, возвращая кортежи `(token, isSep)`. Разделители не отбрасываются — они тоже возвращаются как токены с `isSep = true`.

```nim
for word in tokenize("  this is an  example  "):
  echo word
# ("  ", true)
# ("this", false)
# (" ", true)
# ("is", false)
# (" ", true)
# ("an", false)
# ("  ", true)
# ("example", false)
# ("  ", true)
```
