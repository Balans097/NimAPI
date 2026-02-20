# Справочник модуля `std/unicode` (Nim)

> Модуль обеспечивает поддержку кодировки **UTF-8** и работу с Unicode v12.0.0.  
> Совместим с: `strutils`, `unidecode`, `encodings`.

---

## Содержание

1. [Типы](#типы)
2. [Кодирование и декодирование](#кодирование-и-декодирование)
3. [Длина и позиционирование](#длина-и-позиционирование)
4. [Доступ к рунам](#доступ-к-рунам)
5. [Валидация](#валидация)
6. [Классификация рун](#классификация-рун)
7. [Преобразование регистра](#преобразование-регистра)
8. [Строковые операции](#строковые-операции)
9. [Итераторы](#итераторы)
10. [Операторы сравнения](#операторы-сравнения)

---

## Типы

### `Rune`

```nim
type Rune* = distinct int32
```

Тип для хранения одного кодовой точки Unicode. Базовый тип — `int32`.

Одна `Rune` может комбинироваться с другими для формирования видимого символа (графемы).

```nim
import std/unicode

let r = "ñ".runeAt(0)
echo r           # ñ
echo r.ord       # 241
echo r.toUTF8    # "ñ"
```

---

## Кодирование и декодирование

### `toUTF8(c: Rune): string`

Преобразует руну в её UTF-8 представление в виде строки.

```nim
import std/unicode

let r = "ó".runeAt(0)
echo r.toUTF8       # "ó"
echo r.toUTF8.len   # 2 (байта)
```

---

### `$(rune: Rune): string`

Псевдоним для `toUTF8`. Позволяет использовать руну напрямую со строковыми операциями.

```nim
import std/unicode

let r = "ñ".runeAt(0)
echo $r          # "ñ"
echo "Hello, " & $r  # "Hello, ñ"
```

---

### `$(runes: seq[Rune]): string`

Преобразует последовательность рун обратно в строку. Обратная операция к `toRunes`.

```nim
import std/unicode

let someString = "öÑ"
let runes = toRunes(someString)
echo $runes      # "öÑ"
assert $runes == someString
```

---

### `add(s: var string; c: Rune)`

Добавляет руну в конец строки.

```nim
import std/unicode

var s = "abc"
let c = "ä".runeAt(0)
s.add(c)
echo s   # "abcä"
```

---

### `fastToUTF8Copy(c: Rune, s: var string, pos: int, doInc = true)` *(template)*

Низкоуровневый шаблон: копирует UTF-8 байты руны `c` в заранее выделенную строку `s` начиная с позиции `pos`.

- Если `doInc = true` (по умолчанию), `pos` увеличивается на количество записанных байтов.
- Для максимальной производительности строка `s` должна быть заранее выделена с достаточным запасом.

```nim
import std/unicode

let r = "ñ".runeAt(0)
var s = newString(0)
var pos = 0
fastToUTF8Copy(r, s, pos)
echo s   # "ñ"
```

---

### `fastRuneAt(s, i, result, doInc = true)` *(template)*

Низкоуровневый шаблон: читает руну из строки `s` на байтовой позиции `i` и помещает её в `result`.

- Если `doInc = true`, `i` увеличивается на количество байтов, занятых руной.
- При неверной UTF-8 последовательности возвращает руну-замену `U+FFFD`.

```nim
import std/unicode

let s = "añyóng"
var i = 1  # позиция 'ñ'
var r: Rune
fastRuneAt(s, i, r)
echo r   # ñ
echo i   # 3 (сдвинулся на 2 байта)
```

---

## Длина и позиционирование

### `runeLen(s): int`

Возвращает количество рун в строке (не байтов).

```nim
import std/unicode

let a = "añyóng"
echo a.len      # 8 (байт)
echo a.runeLen  # 6 (рун)
```

---

### `runeLenAt(s, i: Natural): int`

Возвращает число байтов, занимаемых руной, начинающейся на байтовой позиции `i`.

```nim
import std/unicode

let a = "añyóng"
echo a.runeLenAt(0)  # 1 (ASCII 'a')
echo a.runeLenAt(1)  # 2 ('ñ' — двухбайтовый символ)
echo a.runeLenAt(3)  # 1 ('y')
```

---

### `size(r: Rune): int`

Возвращает количество байтов, которое занимает данная руна в UTF-8.

```nim
import std/unicode

let runes = toRunes("aá")
echo size(runes[0])  # 1 ('a' — ASCII)
echo size(runes[1])  # 2 ('á')
```

---

### `runeOffset(s, pos: Natural, start: Natural = 0): int`

Возвращает байтовую позицию руны с индексом `pos` (считая руны). Необязательный параметр `start` задаёт начальную байтовую позицию для поиска. Возвращает `-1`, если строка закончилась раньше.

> ⚠️ **Медленная операция.** По возможности используйте итератор или `seq[Rune]`.

```nim
import std/unicode

let a = "añyóng"
echo a.runeOffset(0)  # 0
echo a.runeOffset(1)  # 1
echo a.runeOffset(3)  # 4
echo a.runeOffset(4)  # 6
```

---

### `runeReverseOffset(s, rev: Positive): (int, int)`

Возвращает кортеж `(byteOffset, totalRunes)`, где `byteOffset` — байтовое смещение руны с конца строки (нумерация с 1). Если рун недостаточно, `byteOffset` будет отрицательным.

> ⚠️ **Медленная операция.**

```nim
import std/unicode

let a = "añyóng"
let (offset, total) = a.runeReverseOffset(1)
echo offset  # байтовая позиция последней руны
echo total   # всего рун в строке: 6
```

---

### `graphemeLen(s, i: Natural): Natural`

Возвращает число байтов, принадлежащих графеме начиная с байтового индекса `i`, включая следующие за ней объединяющие символы.

```nim
import std/unicode

let a = "añyóng"
echo a.graphemeLen(1)  # 2 (ñ = 'n' + диакритика)
echo a.graphemeLen(3)  # 1 (y — обычный ASCII)
echo a.graphemeLen(4)  # 2 (ó)
```

---

### `lastRune(s, last: int): (Rune, int)`

Возвращает кортеж `(rune, byteLen)` для последней руны в подстроке `s[0..last]`.

```nim
import std/unicode

let a = "añyóng"
let (r, size) = a.lastRune(a.high)
echo r     # g
echo size  # 1
```

---

## Доступ к рунам

### `runeAt(s, i: Natural): Rune`

Возвращает руну на **байтовой** позиции `i`. Не выполняет проверку корректности байта — он может указывать внутрь многобайтового символа.

```nim
import std/unicode

let a = "añyóng"
echo a.runeAt(0)  # a
echo a.runeAt(1)  # ñ  (начало двухбайтового символа)
echo a.runeAt(2)  # ñ  (второй байт того же символа)
echo a.runeAt(3)  # y
```

---

### `runeAtPos(s, pos: int): Rune`

Возвращает руну по **индексу руны** (не байта) `pos`.

> ⚠️ **Медленная операция** — для каждого вызова проходит по строке с начала.

```nim
import std/unicode

let a = "añyóng"
echo a.runeAtPos(0)  # a
echo a.runeAtPos(1)  # ñ
echo a.runeAtPos(3)  # ó
```

---

### `runeStrAtPos(s, pos: Natural): string`

Возвращает руну на позиции `pos` (индекс руны) как UTF-8 строку.

> ⚠️ **Медленная операция.**

```nim
import std/unicode

let a = "añyóng"
echo a.runeStrAtPos(1)  # "ñ"
echo a.runeStrAtPos(3)  # "ó"
```

---

### `runeSubStr(s, pos: int, len: int = int.high): string`

Возвращает подстроку, начиная с руны номер `pos`, длиной `len` рун.

- Отрицательные `pos` и `len` отсчитываются от конца строки.
- Если `len` не указан — берётся до конца строки.

```nim
import std/unicode

let s = "Hänsel  ««: 10,00€"
echo s.runeSubStr(0, 2)   # "Hä"
echo s.runeSubStr(10, 1)  # ":"
echo s.runeSubStr(-6)     # "10,00€"
echo s.runeSubStr(10)     # ": 10,00€"
echo s.runeSubStr(12, 5)  # "10,00"
echo s.runeSubStr(-6, 3)  # "10,"
```

---

### `toRunes(s): seq[Rune]`

Преобразует строку в последовательность рун. Позволяет использовать индексацию по рунам с O(1).

```nim
import std/unicode

let a = toRunes("aáä")
echo a[0]  # a
echo a[1]  # á
echo a[2]  # ä
echo a.len # 3
```

---

## Валидация

### `validateUtf8(s): int`

Проверяет, является ли строка корректным UTF-8.

- Возвращает `-1`, если строка корректна.
- Возвращает байтовый индекс первого некорректного байта, если нет.

```nim
import std/unicode

echo validateUtf8("añyóng")      # -1 (всё ок)
echo validateUtf8("abc\xff")     # 3  (позиция плохого байта)
echo validateUtf8("")            # -1
```

---

## Классификация рун

### `isLower(c: Rune): bool`

Возвращает `true`, если руна является строчной буквой.

> Предпочтительнее использовать `isLower` вместо `isUpper` — это несколько эффективнее.

```nim
import std/unicode

echo "a".runeAt(0).isLower  # true
echo "A".runeAt(0).isLower  # false
echo "α".runeAt(0).isLower  # true (греческая строчная)
```

---

### `isUpper(c: Rune): bool`

Возвращает `true`, если руна является заглавной буквой.

```nim
import std/unicode

echo "A".runeAt(0).isUpper  # true
echo "Γ".runeAt(0).isUpper  # true (греческая заглавная)
echo "a".runeAt(0).isUpper  # false
```

---

### `isAlpha(c: Rune): bool`

Возвращает `true`, если руна является буквой (любого алфавита).

```nim
import std/unicode

echo "a".runeAt(0).isAlpha  # true
echo "ñ".runeAt(0).isAlpha  # true
echo "1".runeAt(0).isAlpha  # false
echo " ".runeAt(0).isAlpha  # false
```

---

### `isAlpha(s): bool`

Возвращает `true`, если **все** символы строки являются буквами. Пустая строка даёт `false`.

```nim
import std/unicode

echo "añyóng".isAlpha  # true
echo "abc123".isAlpha  # false
echo "".isAlpha        # false
```

---

### `isTitle(c: Rune): bool`

Возвращает `true`, если руна является titlecase-символом (особый класс Unicode, одновременно и upper и lower — например, `ǅ`).

```nim
import std/unicode

echo "A".runeAt(0).isTitle  # false
```

---

### `isWhiteSpace(c: Rune): bool`

Возвращает `true`, если руна является пробельным символом Unicode (пробел, табуляция, перевод строки и т.д.).

```nim
import std/unicode

echo " ".runeAt(0).isWhiteSpace   # true
echo "\t".runeAt(0).isWhiteSpace  # true
echo "a".runeAt(0).isWhiteSpace   # false
```

---

### `isSpace(s): bool`

Возвращает `true`, если **все** символы строки являются пробельными. Пустая строка даёт `false`.

```nim
import std/unicode

echo "\t\n \r".isSpace  # true
echo "  a  ".isSpace    # false
echo "".isSpace         # false
```

---

### `isCombining(c: Rune): bool`

Возвращает `true`, если руна является объединяющим (combining) символом Unicode (диакритика, надстрочные знаки и пр.).

```nim
import std/unicode

# U+0301 — объединяющий акут (combining acute accent)
let combining = Rune(0x0301)
echo combining.isCombining  # true
echo "a".runeAt(0).isCombining  # false
```

---

## Преобразование регистра

### `toLower(c: Rune): Rune`

Преобразует руну в строчную.

```nim
import std/unicode

echo "A".runeAt(0).toLower  # a
echo "Γ".runeAt(0).toLower  # γ
echo "a".runeAt(0).toLower  # a (уже строчная)
```

---

### `toUpper(c: Rune): Rune`

Преобразует руну в заглавную.

> По возможности предпочитайте `toLower` — он немного эффективнее.

```nim
import std/unicode

echo "a".runeAt(0).toUpper  # A
echo "γ".runeAt(0).toUpper  # Γ
```

---

### `toTitle(c: Rune): Rune`

Преобразует руну в titlecase.

```nim
import std/unicode

let r = "a".runeAt(0).toTitle
echo r  # A (для большинства символов совпадает с toUpper)
```

---

### `toLower(s): string`

Возвращает строку, в которой все руны преобразованы в строчные.

```nim
import std/unicode

echo toLower("ABΓ")       # "abγ"
echo "ÄÖÜ".toLower        # "äöü"
```

---

### `toUpper(s): string`

Возвращает строку, в которой все руны преобразованы в заглавные.

```nim
import std/unicode

echo toUpper("abγ")       # "ABΓ"
echo "hello, мир".toUpper # "HELLO, МИР"
```

---

### `swapCase(s): string`

Меняет регистр всех рун на противоположный.

```nim
import std/unicode

echo swapCase("Αlpha Βeta Γamma")  # "αLPHA βETA γAMMA"
echo swapCase("Hello")             # "hELLO"
```

---

### `capitalize(s): string`

Переводит первую руну строки в верхний регистр, остальные оставляет без изменений.

```nim
import std/unicode

echo capitalize("βeta")   # "Βeta"
echo capitalize("hello")  # "Hello"
echo capitalize("")        # ""
```

---

### `title(s): string`

Возвращает строку в стиле «заголовка»: первая буква каждого слова — заглавная.

```nim
import std/unicode

echo title("αlpha βeta γamma")  # "Αlpha Βeta Γamma"
echo title("hello world")       # "Hello World"
```

---

## Строковые операции

### `repeat(c: Rune, count: Natural): string`

Возвращает строку из `count` повторений руны `c`.

```nim
import std/unicode

let n = "ñ".runeAt(0)
echo n.repeat(5)  # "ñññññ"
echo n.repeat(0)  # ""
```

---

### `reversed(s): string`

Возвращает строку в обратном порядке рун. Корректно обрабатывает объединяющие символы (combining characters) — они остаются привязаны к своим базовым символам.

```nim
import std/unicode

echo reversed("Reverse this!")  # "!siht esreveR"
echo reversed("先秦兩漢")        # "漢兩秦先"
echo reversed("as⃝df̅")         # "f̅ds⃝a"  (combining корректно)
```

---

### `translate(s, replacements: proc(key: string): string): string`

Заменяет слова в строке `s` с помощью функции `replacements`. Слова разделяются пробельными символами Unicode.

```nim
import std/unicode

proc wordToNumber(s: string): string =
  case s
  of "one": "1"
  of "two": "2"
  else: s

let a = "one two three four"
echo a.translate(wordToNumber)  # "1 2 three four"
```

---

### `strip(s, leading = true, trailing = true, runes = unicodeSpaces): string`

Удаляет указанные руны с начала и/или конца строки.

- `leading = true` — удалять с начала.
- `trailing = true` — удалять с конца.
- `runes` — множество удаляемых рун; по умолчанию все Unicode-пробелы.

```nim
import std/unicode

let a = "\táñyóng   "
echo a.strip                    # "áñyóng"
echo a.strip(leading = false)   # "\táñyóng"
echo a.strip(trailing = false)  # "áñyóng   "

# Удаление кастомных символов
let b = "***hello***"
echo b.strip(runes = @["*".runeAt(0)])  # "hello"
```

---

### `align(s, count: Natural, padding = ' '.Rune): string`

Выравнивает строку по правому краю до длины `count` рун, добавляя `padding` слева. Если длина строки уже >= `count`, строка возвращается без изменений.

```nim
import std/unicode

echo align("abc", 4)               # " abc"
echo align("1232", 6)              # "  1232"
echo align("1232", 6, '#'.Rune)   # "##1232"
echo align("Åge", 5)              # "  Åge"
echo align("×", 4, '_'.Rune)     # "___×"
```

---

### `alignLeft(s, count: Natural, padding = ' '.Rune): string`

Выравнивает строку по левому краю до длины `count` рун, добавляя `padding` справа.

```nim
import std/unicode

echo alignLeft("abc", 4)              # "abc "
echo alignLeft("1232", 6)            # "1232  "
echo alignLeft("1232", 6, '#'.Rune)  # "1232##"
echo alignLeft("Åge", 5)             # "Åge  "
echo alignLeft("×", 4, '_'.Rune)    # "×___"
```

---

### `cmpRunesIgnoreCase(a, b): int`

Сравнивает две UTF-8 строки без учёта регистра.

- `0` — строки равны.
- `< 0` — `a` лексикографически меньше `b`.
- `> 0` — `a` лексикографически больше `b`.

```nim
import std/unicode

echo cmpRunesIgnoreCase("Hello", "hello")  # 0
echo cmpRunesIgnoreCase("abc", "abd")      # < 0
echo cmpRunesIgnoreCase("Ä", "ä")         # 0
```

---

### `split(s, seps: openArray[Rune] = unicodeSpaces, maxsplit = -1): seq[string]`

Разбивает строку по набору рун-разделителей. Возвращает последовательность подстрок.

- `seps` — список рун-разделителей; по умолчанию все Unicode-пробелы.
- `maxsplit` — максимальное число разбиений (`-1` — без ограничений).

```nim
import std/unicode

# По пробелам
echo "hello world  nim".split()
# @["hello", "world", "nim"]

# По кастомным разделителям
echo split("añyóng:hÃllo;是", ";:".toRunes)
# @["añyóng", "hÃllo", "是"]
```

---

### `split(s, sep: Rune, maxsplit = -1): seq[string]`

Разбивает строку по одной руне-разделителю.

```nim
import std/unicode

echo split("a;b;;c", ";".runeAt(0))
# @["a", "b", "", "c"]
```

---

### `splitWhitespace(s): seq[string]`

Разбивает строку по всем пробельным рунам Unicode. Аналог `split()` без аргументов, но возвращает `seq[string]`.

```nim
import std/unicode

echo splitWhitespace("hello  world\tnim")
# @["hello", "world", "nim"]
```

---

## Итераторы

### `iterator runes(s): Rune`

Перебирает все руны строки по одной.

```nim
import std/unicode

for r in runes("añyóng"):
  echo r
# a
# ñ
# y
# ó
# n
# g
```

---

### `iterator utf8(s): string`

Перебирает все руны строки, возвращая каждую как UTF-8 строку.

```nim
import std/unicode

for c in utf8("añyóng"):
  echo c, " (", c.len, " байт)"
# a (1 байт)
# ñ (2 байта)
# y (1 байт)
# ó (2 байта)
# n (1 байт)
# g (1 байт)
```

---

### `iterator split(s, seps, maxsplit)` / `iterator split(s, sep, maxsplit)`

Итераторные версии `split`. Не создают промежуточную последовательность — эффективны для обработки больших строк.

```nim
import std/unicode

for part in split("a:b:c", ":".runeAt(0)):
  echo part
# a
# b
# c
```

---

### `iterator splitWhitespace(s)`

Итераторная версия `splitWhitespace`.

```nim
import std/unicode

for word in splitWhitespace("hello   world"):
  echo word
# hello
# world
```

---

## Операторы сравнения

### `==(a, b: Rune): bool`

Проверяет равенство двух рун (по кодовой точке).

```nim
import std/unicode

echo "a".runeAt(0) == "a".runeAt(0)  # true
echo "a".runeAt(0) == "b".runeAt(0)  # false
```

---

### `<%(a, b: Rune): bool`

Проверяет, что кодовая точка `a` строго меньше кодовой точки `b`. Беззнаковое сравнение.

```nim
import std/unicode

let a = "ú".runeAt(0)
let b = "ü".runeAt(0)
echo a <% b  # true
```

---

### `<=%(a, b: Rune): bool`

Проверяет, что кодовая точка `a` меньше или равна кодовой точке `b`. Беззнаковое сравнение.

```nim
import std/unicode

let a = "ú".runeAt(0)
let b = "ü".runeAt(0)
echo a <=% b  # true
echo b <=% b  # true
```

---

## Полные примеры

### Подсчёт символов в многоязычной строке

```nim
import std/unicode

let text = "Hello, мир! 你好"
echo "Байт: ", text.len       # 20+
echo "Рун:  ", text.runeLen   # 13 (символов)
```

---

### Итерация и фильтрация по типу символа

```nim
import std/unicode

let s = "Привет 123! Мир."
var letters, digits, other: int

for r in runes(s):
  if r.isAlpha: inc letters
  elif r.isWhiteSpace: discard
  else: inc other

echo "Букв: ", letters
echo "Прочих: ", other
```

---

### Разворот строки с диакритикой

```nim
import std/unicode

# Объединяющие символы корректно привязаны к своим базовым символам
let s = "as⃝df̅"
echo reversed(s)  # "f̅ds⃝a"
```

---

### Построение таблицы с выравниванием по колонкам

```nim
import std/unicode

let words = ["Åge", "先秦", "a", "Unicode"]
for w in words:
  echo alignLeft(w, 12) & "|" & align($w.runeLen, 5)
```

---

### Нечувствительный к регистру поиск

```nim
import std/unicode

proc containsIgnoreCase(text, pattern: string): bool =
  # Простая проверка через toLower
  toLower(text).find(toLower(pattern)) >= 0

echo containsIgnoreCase("Привет МИР", "мир")  # true
```

---

### Безопасная работа с позициями в строке

```nim
import std/unicode

let s = "Hänsel"
# ПРАВИЛЬНО — работаем через seq[Rune] для индексации
let runes = toRunes(s)
echo runes[1]        # ä
echo runes.len       # 6

# МЕДЛЕННО, но работает для разовых обращений
echo s.runeAtPos(1)  # ä
echo s.runeSubStr(1, 3)  # "äns"
```
