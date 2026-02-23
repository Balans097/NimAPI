# Справочник модуля `unicode`

> **Стандартная библиотека Nim — поддержка UTF-8 / Unicode**
> Совместимо с **Unicode v12.0.0**.

---

## Общее описание

Модуль `unicode` — это стандартный способ работы с **юникодным текстом** в Nim. Он оперирует строками в кодировке UTF-8 — тем же типом `string`, который используется повсюду в Nim — но рассматривает их как последовательности **Rune** (кодовых точек Unicode), а не как сырые байты.

Главная вещь, которую нужно понять перед использованием модуля, — это разница между тремя видами «длины» строки:

```
s = "añyóng"

s.len      == 8   # сырые байты (a=1, ñ=2, y=1, ó=2, n=1, g=1)
s.runeLen  == 6   # кодовые точки Unicode (символы в человеческом понимании)
s.graphemeLen(1) == 2  # байт, занятых графемой на байтовом индексе 1 (ñ)
```

Поскольку UTF-8 — кодировка переменной ширины, наивное байтовое индексирование в юникодной строке для символов вне ASCII даёт некорректный результат. Этот модуль предоставляет все необходимые инструменты для корректной работы с текстом на любом языке.

```nim
import std/unicode
```

---

## Тип `Rune`

```nim
type Rune* = distinct int32
```

`Rune` хранит одну **кодовую точку** Unicode — целое число в диапазоне U+0000 … U+10FFFF. Это отдельный тип, отличный от `int32`, поэтому нельзя случайно смешать его с обычными числами.

Один видимый на экране символ может состоять из одного *базового* Rune и нуля или более *комбинирующих* Rune (диакритических знаков, модификаторов). Например, `ñ` в форме NFD — это базовый rune `n` (U+006E) плюс комбинирующая тильда (U+0303).

**Создание Rune:**

```nim
let r1 = "ñ".runeAt(0)     # из строки по байтовому индексу 0
let r2 = Rune(0x00F1)      # из значения кодовой точки
let r3 = 'n'.Rune          # из char (только ASCII)
```

---

## Измерение и навигация

### `runeLen`

```nim
proc runeLen*(s: openArray[char]): int
proc runeLen*(s: string): int
```

Возвращает количество **кодовых точек Unicode** (Rune) в строке `s`. Это «человеческое» число символов — не длина в байтах.

```nim
let s = "añyóng"
assert s.len     == 8  # байт
assert s.runeLen == 6  # rune
assert "€".runeLen == 1  # один rune, но 3 байта
```

---

### `runeLenAt`

```nim
proc runeLenAt*(s: openArray[char], i: Natural): int
proc runeLenAt*(s: string, i: Natural): int
```

Возвращает количество **байт**, которое занимает Rune, начинающийся на **байтовом индексе** `i` (1–6). Это ключевой инструмент для продвижения байтового указателя ровно на одну кодовую точку.

```nim
let s = "añyóng"
assert s.runeLenAt(0) == 1  # 'a'  — ASCII → 1 байт
assert s.runeLenAt(1) == 2  # 'ñ'  — U+00F1 → 2 байта
assert s.runeLenAt(3) == 1  # 'y'  — ASCII → 1 байт
assert s.runeLenAt(4) == 2  # 'ó'  — U+00F3 → 2 байта
```

---

### `size`

```nim
proc size*(r: Rune): int
```

Возвращает количество байт, которое Rune `r` занимает в кодировке UTF-8 (1–6).

```nim
let runes = toRunes("aá€𐍈")
assert runes[0].size == 1  # ASCII
assert runes[1].size == 2  # расширенная латиница
assert runes[2].size == 3  # знак евро U+20AC
assert runes[3].size == 4  # готская буква U+10348
```

---

### `graphemeLen`

```nim
proc graphemeLen*(s: openArray[char]; i: Natural): Natural
proc graphemeLen*(s: string; i: Natural): Natural
```

Возвращает количество **байт**, принадлежащих кластеру графем, начинающемуся на байтовом индексе `i`. Кластер графем — это то, что человек воспринимает как один «символ»: базовый Rune плюс все следующие за ним комбинирующие Rune.

```nim
# "añyóng" — ñ и ó здесь хранятся как прекомпозированные формы (NFC):
let s = "añyóng"
assert s.graphemeLen(0) == 1  # 'a'
assert s.graphemeLen(1) == 2  # 'ñ' (U+00F1, прекомпозированный)
assert s.graphemeLen(3) == 1  # 'y'

# С комбинирующим символом:
let t = "e\u0301"  # 'e' + комбинирующий знак ударения
assert t.graphemeLen(0) == 3  # 'e' + модификатор = 3 байта
```

---

### `runeOffset`

```nim
proc runeOffset*(s: openArray[char], pos: Natural, start: Natural = 0): int
proc runeOffset*(s: string, pos: Natural, start: Natural = 0): int
```

Преобразует **индекс Rune** `pos` (считая от начала или от байтового смещения `start`) в **байтовый индекс** в строке `s`. Возвращает `-1`, если `pos` превышает число Rune.

> ⚠️ Сложность O(n) — нужно просканировать строку с начала. При многократном доступе лучше заранее преобразовать в `seq[Rune]`.

```nim
let s = "añyóng"
# Rune 0='a'(байт 0), 1='ñ'(байт 1), 2='y'(байт 3), 3='ó'(байт 4), ...
assert s.runeOffset(0) == 0
assert s.runeOffset(1) == 1
assert s.runeOffset(2) == 3
assert s.runeOffset(3) == 4
assert s.runeOffset(6) == -1  # за пределами строки
```

---

### `runeReverseOffset`

```nim
proc runeReverseOffset*(s: openArray[char], rev: Positive): (int, int)
proc runeReverseOffset*(s: string, rev: Positive): (int, int)
```

Отсчитывает `rev` кодовых точек **с конца** строки `s` (нумерация с 1: `rev=1` — последний Rune). Возвращает кортеж `(байтовоеСмещение, всегоRune)`. Если `rev` превышает число Rune, `байтовоеСмещение` будет отрицательным.

> ⚠️ O(n) — то же предупреждение, что и для `runeOffset`.

```nim
let s = "añyóng"  # 6 rune
let (offset, total) = s.runeReverseOffset(1)  # последний rune 'g'
assert total == 6
# offset указывает на байт, с которого начинается 'g'
```

---

### `lastRune`

```nim
proc lastRune*(s: openArray[char]; last: int): (Rune, int)
proc lastRune*(s: string; last: int): (Rune, int)
```

Возвращает последний Rune в `s[0..last]` и его длину в байтах. Удобно для сканирования строки в обратном направлении без полного преобразования.

```nim
let s = "añ"
let (rune, byteLen) = s.lastRune(s.high)
assert $rune == "ñ"
assert byteLen == 2
```

---

## Чтение Rune

### `runeAt`

```nim
proc runeAt*(s: openArray[char], i: Natural): Rune
proc runeAt*(s: string, i: Natural): Rune
```

Возвращает Rune, начинающийся на **байтовом индексе** `i`. Это низкоуровневый быстрый доступ — `i` является *байтовой* позицией, а не позицией Rune.

```nim
let s = "añyóng"
assert s.runeAt(0) == "a".runeAt(0)  # байт 0 → 'a'
assert s.runeAt(1) == "ñ".runeAt(0)  # байт 1 → 'ñ' (2 байта)
assert s.runeAt(3) == "y".runeAt(0)  # байт 3 → 'y'
# runeAt(2) вернёт второй байт 'ñ' — continuation byte.
# Используйте runeLenAt для корректного продвижения.
```

---

### `runeAtPos`

```nim
proc runeAtPos*(s: openArray[char], pos: int): Rune
proc runeAtPos*(s: string, pos: int): Rune
```

Возвращает Rune на **позиции Rune** `pos` (нумерация с 0, в кодовых точках). Удобнее `runeAt`, но работает за O(n).

> ⚠️ Медленно при повторном доступе — лучше итерировать через `runes` или преобразовать в `seq[Rune]`.

```nim
let s = "añyóng"
assert s.runeAtPos(0) == "a".runeAt(0)
assert s.runeAtPos(1) == "ñ".runeAt(0)
assert s.runeAtPos(2) == "y".runeAt(0)
```

---

### `runeStrAtPos`

```nim
proc runeStrAtPos*(s: openArray[char], pos: Natural): string
proc runeStrAtPos*(s: string, pos: Natural): string
```

Возвращает Rune на позиции `pos` в виде **строки UTF-8** (а не значения `Rune`). Удобно, когда нужна строковая форма напрямую.

> ⚠️ O(n) — то же предупреждение, что и у `runeAtPos`.

```nim
let s = "añyóng"
assert s.runeStrAtPos(1) == "ñ"
assert s.runeStrAtPos(3) == "ó"
```

---

### `fastRuneAt` (шаблон)

```nim
template fastRuneAt*(s: openArray[char] or string, i: int, result: untyped, doInc = true)
```

**Самый быстрый** способ прочитать один Rune с позиции `i`. Декодирует Rune в `result` и, если `doInc = true`, продвигает `i` на число обработанных байт. Некорректные байтовые последовательности дают символ замены U+FFFD.

Шаблон инлайнится — никакого overhead вызова функции. Используется внутри всех прочих процедур чтения Rune. Применяйте его в критических по производительности циклах сканирования.

```nim
var s = "añy"
var i = 0
var r: Rune

fastRuneAt(s, i, r)          # r = 'a', i = 1
fastRuneAt(s, i, r)          # r = 'ñ', i = 3
fastRuneAt(s, i, r)          # r = 'y', i = 4

# Чтение без продвижения:
i = 1
fastRuneAt(s, i, r, doInc = false)  # r = 'ñ', i по-прежнему = 1
```

---

## Извлечение подстроки

### `runeSubStr`

```nim
proc runeSubStr*(s: openArray[char], pos: int, len: int = int.high): string
proc runeSubStr*(s: string, pos: int, len: int = int.high): string
```

Возвращает подстроку, начинающуюся с **позиции Rune** `pos` и охватывающую `len` Rune. Оба параметра могут быть **отрицательными** — тогда они отсчитываются от конца строки. Если `len` не задан — означает «до конца строки».

```nim
let s = "Hänsel  ««: 10,00€"

assert s.runeSubStr(0, 2)   == "Hä"      # rune 0..1
assert s.runeSubStr(10, 1)  == ":"       # rune 10
assert s.runeSubStr(-6)     == "10,00€"  # последние 6 rune до конца
assert s.runeSubStr(10)     == ": 10,00€" # от rune 10 до конца
assert s.runeSubStr(12, 5)  == "10,00"   # 5 rune начиная с rune 12
assert s.runeSubStr(-6, 3)  == "10,"     # 3 rune начиная за 6 от конца
```

---

## Преобразование между Rune и строкой

### `toUTF8`

```nim
proc toUTF8*(c: Rune): string
```

Преобразует один Rune в его строковое UTF-8 представление.

```nim
let r = "ñ".runeAt(0)
assert r.toUTF8 == "ñ"
assert Rune(0x20AC).toUTF8 == "€"
```

---

### `$` (Rune → строка)

```nim
proc `$`*(rune: Rune): string
```

Псевдоним для `toUTF8`. Позволяет использовать `echo rune` и интерполяцию строк в привычном виде.

```nim
let r = Rune(0x03B1)  # α
echo $r              # "α"
echo "Буква: " & $r  # "Буква: α"
```

---

### `$` (seq[Rune] → строка)

```nim
proc `$`*(runes: seq[Rune]): string
```

Преобразует последовательность Rune обратно в строку UTF-8. Обратная операция по отношению к `toRunes`.

```nim
let runes = @[Rune('H'), "ä".runeAt(0), Rune('i')]
assert $runes == "Häi"
```

---

### `add`

```nim
proc add*(s: var string; c: Rune)
```

Добавляет один Rune `c` к строке `s`. Эффективнее, чем `s &= $c`, — не создаёт промежуточных аллокаций.

```nim
var s = "abc"
s.add("ä".runeAt(0))
s.add(Rune(0x20AC))  # знак евро
assert s == "abcä€"
```

---

### `toRunes`

```nim
proc toRunes*(s: openArray[char]): seq[Rune]
proc toRunes*(s: string): seq[Rune]
```

Декодирует всю строку в `seq[Rune]`. После преобразования индексирование по Rune работает за O(1) — полезно, когда нужно многократно обращаться к позициям или изменять отдельные Rune.

```nim
let runes = toRunes("aáä")
assert runes[0] == "a".runeAt(0)
assert runes[1] == "á".runeAt(0)
assert runes[2] == "ä".runeAt(0)
assert runes.len == 3

# Туда и обратно:
assert $runes == "aáä"
```

---

### `fastToUTF8Copy` (шаблон)

```nim
template fastToUTF8Copy*(c: Rune, s: var string, pos: int, doInc = true)
```

Копирует UTF-8 байты Rune `c` в **заранее выделенную** строку `s` начиная с байтовой позиции `pos`. Если `doInc = true`, `pos` продвигается на число записанных байт. Это высокопроизводительный строительный блок для формирования строк — используется внутри `toUTF8`, `add` и конвертирующих процедур.

```nim
var buf = newString(3)  # заранее выделяем под один 3-байтовый rune (€)
var pos = 0
fastToUTF8Copy(Rune(0x20AC), buf, pos)
assert buf == "€"
assert pos == 3
```

---

## Валидация

### `validateUtf8`

```nim
proc validateUtf8*(s: openArray[char]): int
proc validateUtf8*(s: string): int
```

Сканирует строку `s` на наличие некорректных UTF-8 последовательностей. Возвращает **байтовый индекс** первого некорректного байта или `-1`, если строка валидна. Также обнаруживает overlong-кодирования (угроза безопасности в старых парсерах).

```nim
assert validateUtf8("hello")    == -1         # валидный ASCII
assert validateUtf8("añyóng")   == -1         # валидный UTF-8
assert validateUtf8("hello\xFF") == 5         # 0xFF на байте 5 некорректен
```

---

## Итераторы

### `runes`

```nim
iterator runes*(s: openArray[char]): Rune
iterator runes*(s: string): Rune
```

Итерирует по всем Rune строки `s`. Это **идиоматичный и эффективный** способ обхода всех кодовых точек — предпочтительнее индексного доступа.

```nim
let s = "añy"
for r in s.runes:
  echo $r   # выводит: a  ñ  y

# Сбор в последовательность:
import std/sequtils
let runes = toSeq(s.runes)
```

---

### `utf8`

```nim
iterator utf8*(s: openArray[char]): string
iterator utf8*(s: string): string
```

Аналогично `runes`, но каждая кодовая точка возвращается в виде **строки** (её байтов UTF-8), а не значения `Rune`.

```nim
for ch in "añy".utf8:
  echo ch, " (", ch.len, " байт)"
# a (1 байт)
# ñ (2 байта)
# y (1 байт)
```

---

### `split` (итератор, несколько разделителей)

```nim
iterator split*(s: openArray[char], seps: openArray[Rune] = unicodeSpaces,
                maxsplit: int = -1): string
iterator split*(s: string, seps: openArray[Rune] = unicodeSpaces,
                maxsplit: int = -1): string
```

Разбивает строку `s` по любому из Rune в `seps`, возвращая подстроки. По умолчанию разделитель — `unicodeSpaces`, то есть все пробельные символы Unicode, а не только ASCII-пробел. Параметр `maxsplit` ограничивает число разбиений (-1 — без ограничений).

```nim
import std/sequtils

# По умолчанию: разбивка по пробельным символам Unicode
assert toSeq("hello world\t是".split) == @["hello", "world", "是"]

# Пользовательские разделители
assert toSeq(split("añyóng:hÃllo;是", ";:".toRunes)) ==
  @["añyóng", "hÃllo", "是"]
```

---

### `split` (итератор, один разделитель)

```nim
iterator split*(s: openArray[char], sep: Rune, maxsplit: int = -1): string
iterator split*(s: string, sep: Rune, maxsplit: int = -1): string
```

Аналогично многосимвольному варианту, но разбивает по единственному Rune `sep`. Последовательные разделители порождают пустые строки между ними.

```nim
import std/sequtils

let parts = toSeq(split("a;;b;c", ";".runeAt(0)))
assert parts == @["a", "", "b", "c"]
```

---

### `splitWhitespace` (итератор)

```nim
iterator splitWhitespace*(s: openArray[char]): string
iterator splitWhitespace*(s: string): string
```

Разбивает строку `s` по пробельным Unicode-символам — эквивалент `split(s, unicodeSpaces)`. Несколько идущих подряд пробельных символов трактуются как один разделитель.

```nim
for word in "  hello\t世界  ".splitWhitespace:
  echo word   # "hello", "世界"
```

---

## Процедуры, возвращающие последовательности

### `split` (процедура, несколько разделителей)

```nim
proc split*(s: openArray[char], seps: openArray[Rune] = unicodeSpaces,
            maxsplit: int = -1): seq[string]
proc split*(s: string, seps: openArray[Rune] = unicodeSpaces,
            maxsplit: int = -1): seq[string]
```

То же, что итератор `split`, но возвращает `seq[string]`.

```nim
let parts = "a,b,c".split(",".toRunes)
assert parts == @["a", "b", "c"]
```

---

### `split` (процедура, один разделитель)

```nim
proc split*(s: openArray[char], sep: Rune, maxsplit: int = -1): seq[string]
proc split*(s: string, sep: Rune, maxsplit: int = -1): seq[string]
```

```nim
let parts = "a::b:c".split(":".runeAt(0))
assert parts == @["a", "", "b", "c"]
```

---

### `splitWhitespace` (процедура)

```nim
proc splitWhitespace*(s: openArray[char]): seq[string]
proc splitWhitespace*(s: string): seq[string]
```

Возвращает `seq[string]` слов, разбитых по пробельным символам Unicode.

```nim
assert "  hello\t世界  ".splitWhitespace == @["hello", "世界"]
```

---

## Смена регистра (один Rune)

### `toLower` (Rune)

```nim
proc toLower*(c: Rune): Rune
```

Возвращает строчный вариант Rune `c` согласно таблицам свёртки регистра Unicode. Если у `c` нет строчной формы (цифры, знаки препинания), возвращается `c` без изменений.

> Предпочитайте `toLower` вместо `toUpper` для сравнения без учёта регистра — Unicode определяет более надёжные таблицы для строчных букв.

```nim
assert toLower("A".runeAt(0)) == "a".runeAt(0)
assert toLower("Γ".runeAt(0)) == "γ".runeAt(0)
assert toLower("1".runeAt(0)) == "1".runeAt(0)  # без изменений
```

---

### `toUpper` (Rune)

```nim
proc toUpper*(c: Rune): Rune
```

Возвращает прописной вариант Rune `c`.

```nim
assert toUpper("a".runeAt(0)) == "A".runeAt(0)
assert toUpper("γ".runeAt(0)) == "Γ".runeAt(0)
```

---

### `toTitle` (Rune)

```nim
proc toTitle*(c: Rune): Rune
```

Преобразует `c` в **титульный** регистр — отдельную категорию Unicode, используемую для некоторых диграфов (например, `dz` → `Dz`). Для большинства Rune результат совпадает с `toUpper`.

```nim
assert toTitle("a".runeAt(0)) == "A".runeAt(0)
```

---

## Смена регистра (целая строка)

### `toUpper` (строка)

```nim
proc toUpper*(s: openArray[char]): string
proc toUpper*(s: string): string
```

Возвращает новую строку, в которой каждый Rune преобразован в верхний регистр.

```nim
assert toUpper("abγ") == "ABΓ"
assert toUpper("héllo") == "HÉLLO"
```

---

### `toLower` (строка)

```nim
proc toLower*(s: openArray[char]): string
proc toLower*(s: string): string
```

Возвращает новую строку, в которой каждый Rune преобразован в нижний регистр.

```nim
assert toLower("ABΓ") == "abγ"
assert toLower("HÉLLO") == "héllo"
```

---

### `swapCase`

```nim
proc swapCase*(s: openArray[char]): string
proc swapCase*(s: string): string
```

Возвращает новую строку, в которой прописные Rune становятся строчными, а строчные — прописными. Rune без регистра (цифры, знаки препинания, CJK) остаются без изменений.

```nim
assert swapCase("Αlpha Βeta Γamma") == "αLPHA βETA γAMMA"
assert swapCase("Hello, World!") == "hELLO, wORLD!"
```

---

### `capitalize`

```nim
proc capitalize*(s: openArray[char]): string
proc capitalize*(s: string): string
```

Преобразует **первый Rune** строки `s` в верхний регистр, оставляя остальное без изменений.

```nim
assert capitalize("βeta") == "Βeta"
assert capitalize("hello world") == "Hello world"  # только первый rune
```

---

### `title`

```nim
proc title*(s: openArray[char]): string
proc title*(s: string): string
```

Возвращает новую строку, в которой **первый Rune каждого слова** (отделённого пробелами) преобразован в верхний регистр. Остальные символы слова остаются без изменений.

```nim
assert title("αlpha βeta γamma") == "Αlpha Βeta Γamma"
assert title("the quick brown fox") == "The Quick Brown Fox"
```

---

## Классификация (один Rune)

### `isLower`

```nim
proc isLower*(c: Rune): bool
```

Возвращает `true`, если `c` — строчная буква Unicode.

> Предпочтительнее `isLower` вместо `isUpper` по соображениям производительности (та же внутренняя причина, что и у `toLower`).

```nim
assert isLower("a".runeAt(0))
assert isLower("γ".runeAt(0))
assert not isLower("A".runeAt(0))
assert not isLower("1".runeAt(0))
```

---

### `isUpper`

```nim
proc isUpper*(c: Rune): bool
```

Возвращает `true`, если `c` — прописная буква Unicode.

```nim
assert isUpper("A".runeAt(0))
assert isUpper("Γ".runeAt(0))
assert not isUpper("a".runeAt(0))
```

---

### `isAlpha` (Rune)

```nim
proc isAlpha*(c: Rune): bool
```

Возвращает `true`, если `c` — **алфавитный** символ Unicode — любая буква в любом скрипте, не только латиница.

```nim
assert isAlpha("a".runeAt(0))   # латиница
assert isAlpha("α".runeAt(0))   # греческий
assert isAlpha("中".runeAt(0))  # CJK
assert not isAlpha("1".runeAt(0))
assert not isAlpha(" ".runeAt(0))
```

---

### `isTitle`

```nim
proc isTitle*(c: Rune): bool
```

Возвращает `true`, если `c` — кодовая точка Unicode с титульным регистром (узкая категория для диграфов, например `ǅ`).

---

### `isWhiteSpace`

```nim
proc isWhiteSpace*(c: Rune): bool
```

Возвращает `true`, если `c` — пробельный символ Unicode. Охватывает все Unicode-пробелы: не только ASCII-пробел, табуляцию и переводы строк, но и неразрывный пробел, идеографический пробел и т.д.

```nim
assert isWhiteSpace(" ".runeAt(0))
assert isWhiteSpace("\t".runeAt(0))
assert isWhiteSpace("\u00A0".runeAt(0))   # неразрывный пробел
assert not isWhiteSpace("a".runeAt(0))
```

---

### `isCombining`

```nim
proc isCombining*(c: Rune): bool
```

Возвращает `true`, если `c` — **комбинирующий символ** Unicode (диакритический знак или модификатор, прикрепляющийся к предшествующему базовому символу). Охватываемые диапазоны: U+0300–U+036F, U+1AB0–U+1AFF, U+1DC0–U+1DFF, U+20D0–U+20FF, U+FE20–U+FE2F. Для ASCII оптимизировано и возвращает `false` немедленно.

```nim
assert isCombining(Rune(0x0301))  # комбинирующее острое ударение (◌́)
assert not isCombining("a".runeAt(0))
```

---

## Классификация (целая строка)

### `isAlpha` (строка)

```nim
proc isAlpha*(s: openArray[char]): bool
proc isAlpha*(s: string): bool
```

Возвращает `true`, если **каждый** Rune в `s` является алфавитным, и строка не пуста.

```nim
assert isAlpha("añyóng")
assert isAlpha("αβγ")
assert not isAlpha("añyóng123")
assert not isAlpha("")
```

---

### `isSpace` (строка)

```nim
proc isSpace*(s: openArray[char]): bool
proc isSpace*(s: string): bool
```

Возвращает `true`, если **каждый** Rune в `s` является пробельным символом Unicode, и строка не пуста.

```nim
assert isSpace("\t\n \r\f\v")
assert not isSpace("  a  ")
assert not isSpace("")
```

---

## Сравнение

### `cmpRunesIgnoreCase`

```nim
proc cmpRunesIgnoreCase*(a, b: openArray[char]): int
proc cmpRunesIgnoreCase*(a, b: string): int
```

Сравнивает две строки UTF-8 **без учёта регистра** с использованием таблиц свёртки регистра Unicode. Возвращает `0` при равенстве, отрицательное значение если `a < b`, и положительное если `a > b`.

```nim
assert cmpRunesIgnoreCase("abc", "ABC") == 0
assert cmpRunesIgnoreCase("αβγ", "ΑΒΓ") == 0
assert cmpRunesIgnoreCase("a", "b") < 0
```

---

### Операторы сравнения Rune

```nim
proc `==`*(a, b: Rune): bool
proc `<%`*(a, b: Rune): bool   # меньше (беззнаковое сравнение кодовых точек)
proc `<=%`*(a, b: Rune): bool  # меньше или равно (беззнаковое)
```

Сравнивают два Rune по значению их кодовой точки Unicode. Операторы `<%` и `<=%` используют **беззнаковое** сравнение, что соответствует порядку Unicode для допустимых кодовых точек.

```nim
let a = "ú".runeAt(0)  # U+00FA
let b = "ü".runeAt(0)  # U+00FC
assert a <% b
assert a <=% b
assert a == "ú".runeAt(0)
```

---

## Манипуляция строками

### `reversed`

```nim
proc reversed*(s: openArray[char]): string
proc reversed*(s: string): string
```

Возвращает строку `s` с Rune в обратном порядке. **Комбинирующие символы обрабатываются корректно** — они остаются привязанными к своему базовому символу и перемещаются вместе с ним, так что кластеры графем не разрушаются.

```nim
assert reversed("Hello") == "olleH"
assert reversed("先秦兩漢") == "漢兩秦先"
# Комбинирующие символы остаются с базовыми:
assert reversed("as⃝df̅") == "f̅ds⃝a"
assert reversed("a⃞b⃞c⃞") == "c⃞b⃞a⃞"
```

---

### `strip`

```nim
proc strip*(s: openArray[char], leading = true, trailing = true,
            runes: openArray[Rune] = unicodeSpaces): string
proc strip*(s: string, leading = true, trailing = true,
            runes: openArray[Rune] = unicodeSpaces): string
```

Удаляет ведущие и/или замыкающие Rune из строки `s`. По умолчанию убирает все пробельные символы Unicode с обоих концов. Передайте собственный массив `runes`, чтобы удалять конкретные символы. Установите `leading = false` или `trailing = false`, чтобы обрезать только один конец.

```nim
assert strip("\t áñyóng   ") == "áñyóng"
assert strip("\t áñyóng   ", leading = false) == "\t áñyóng"
assert strip("\t áñyóng   ", trailing = false) == "áñyóng   "

# Обрезка пользовательских символов
let cutRunes = "*".toRunes
assert strip("**hello**", runes = cutRunes) == "hello"
```

---

### `repeat`

```nim
proc repeat*(c: Rune, count: Natural): string
```

Возвращает строку, состоящую из `count` копий Rune `c`.

```nim
assert repeat("ñ".runeAt(0), 5) == "ñññññ"
assert repeat(Rune(0x2665), 3) == "♥♥♥"
```

---

### `align`

```nim
proc align*(s: openArray[char], count: Natural, padding = ' '.Rune): string
proc align*(s: string, count: Natural, padding = ' '.Rune): string
```

**Выравнивает по правому краю**: добавляет символы `padding` перед строкой `s`, чтобы она имела длину `count` Rune. Если `s.runeLen >= count`, строка возвращается без изменений. Заполнение измеряется в **Rune**, а не в байтах — многобайтовые символы заполнения работают корректно.

```nim
assert align("abc", 6) == "   abc"
assert align("Åge", 5) == "  Åge"
assert align("×", 4, '_'.Rune) == "___×"
assert align("1232", 6, '#'.Rune) == "##1232"
```

---

### `alignLeft`

```nim
proc alignLeft*(s: openArray[char], count: Natural, padding = ' '.Rune): string
proc alignLeft*(s: string, count: Natural, padding = ' '.Rune): string
```

**Выравнивает по левому краю**: добавляет символы `padding` после строки `s`, чтобы она имела длину `count` Rune.

```nim
assert alignLeft("abc", 6) == "abc   "
assert alignLeft("Åge", 5) == "Åge  "
assert alignLeft("×", 4, '_'.Rune) == "×___"
```

---

### `translate`

```nim
proc translate*(s: openArray[char], replacements: proc(key: string): string): string
proc translate*(s: string, replacements: proc(key: string): string): string
```

Разбивает строку `s` на слова (разделённые пробелами), вызывает `replacements(слово)` для каждого и собирает результат обратно, сохраняя исходные пробелы. Колбек получает слово как строку UTF-8 и должен вернуть замену.

```nim
proc wordToNumber(s: string): string =
  case s
  of "one": "1"
  of "two": "2"
  else: s

assert translate("one two three", wordToNumber) == "1 2 three"

# Практический пример: перевод слов
proc en2fr(w: string): string =
  case w
  of "hello": "bonjour"
  of "world": "monde"
  else: w

assert translate("hello world", en2fr) == "bonjour monde"
```

---

## Рекомендации по производительности

| Задача | Лучший подход |
|--------|--------------|
| Подсчитать символы в строке | `s.runeLen` |
| Пройти по всем символам | `for r in s.runes` |
| Случайный доступ по индексу символа | `toRunes(s)` и затем `[]` |
| Собрать строку из Rune | `var s = ""; s.add(rune)` в цикле |
| Продвинуться на один Rune в цикле сканирования | `fastRuneAt(s, i, r)` |
| Вырезать подстроку по позициям символов | `s.runeSubStr(pos, len)` |
| Проверить, что входные данные — валидный UTF-8 | `s.validateUtf8 == -1` |
| Сравнение без учёта регистра | `cmpRunesIgnoreCase(a, b)` |
| Развернуть строку с сохранением графем | `s.reversed` |

---

## Краткая шпаргалка

```
Задача                                       Процедура / Итератор
──────────────────────────────────────────────────────────────────────────
Число Rune в строке                          runeLen(s)
Байтовая длина Rune на байтовом индексе i    runeLenAt(s, i)
Байтовая длина значения Rune                 size(r)
Байт графемы на байтовом индексе i           graphemeLen(s, i)
Rune по байтовому индексу i                  runeAt(s, i)
Rune по индексу Rune pos (медленно)          runeAtPos(s, pos)
Rune как строка по индексу Rune pos          runeStrAtPos(s, pos)
Индекс Rune → байтовое смещение              runeOffset(s, pos)
Rune с конца → байтовое смещение + счётчик   runeReverseOffset(s, rev)
Последний Rune и его байтовая длина          lastRune(s, last)
Быстрое декодирование Rune + продвижение     fastRuneAt(s, i, r)
Rune → строка UTF-8                          toUTF8(r) / $r
Добавить Rune к строке                       s.add(r)
Строка → последовательность Rune             toRunes(s)
Последовательность Rune → строка             $runes
Проверить валидность UTF-8                   validateUtf8(s)
Итерировать по Rune                          for r in s.runes
Итерировать по Rune как строкам              for ch in s.utf8
Подстрока по позициям Rune                   runeSubStr(s, pos, len)
Строчный Rune / строка                       toLower(c) / toLower(s)
Прописной Rune / строка                      toUpper(c) / toUpper(s)
Титульный регистр Rune                       toTitle(c)
Инвертировать регистр                        swapCase(s)
Сделать прописной первый Rune                capitalize(s)
Первый Rune каждого слова — прописной        title(s)
Строчная буква?                              isLower(c)
Прописная буква?                             isUpper(c)
Алфавитный символ?                           isAlpha(c) / isAlpha(s)
Пробельный символ?                           isWhiteSpace(c) / isSpace(s)
Комбинирующий символ?                        isCombining(c)
Сравнение без учёта регистра                 cmpRunesIgnoreCase(a, b)
Развернуть строку (по Rune)                  reversed(s)
Обрезать пробелы / символы                  strip(s)
Повторить Rune                               repeat(c, n)
Выровнять по правому краю (в Rune)           align(s, count)
Выровнять по левому краю (в Rune)            alignLeft(s, count)
Разбить по пробелам                          splitWhitespace(s)
Разбить по разделителям Rune                 split(s, sep) / split(s, seps)
Заменить слова через колбек                  translate(s, proc)
```
