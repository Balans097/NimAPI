# strutils.nim — справочник модуля

> **Импорт:** `import std/strutils`
> **Область применения:** дополнительные процедуры, итераторы и шаблоны для работы со строками (`string`) — классификация символов, преобразование регистра, разбиение и объединение строк, поиск, замена, обрезка/выравнивание, преобразования "число ↔ строка", экранирование, а также форматирование чисел с плавающей точкой и размеров.

Базовые операции со строками (`$`, `&`, `add`, операторы `in`/`contains` и `notin`) уже определены в модуле `system`. Модуль `strutils` строится на их основе и предоставляет всё остальное, что обычно требуется при работе с текстом в Nim.

Большинство процедур поддерживают **синтаксис вызова методов** (`method call syntax`), поэтому запись вида `s.split(',').join("-")` читается как цепочка преобразований.

---

## Оглавление

1. [Обзор всех функций](#обзор-всех-функций)
2. [Константы, типы и вспомогательные средства](#константы-типы-и-вспомогательные-средства)
3. [Основная часть с описанием функций](#основная-часть-с-описанием-функций)
   - [Классификация символов](#классификация-символов)
   - [Преобразование регистра](#преобразование-регистра)
   - [Нормализация и сравнение](#нормализация-и-сравнение)
   - [Разбиение строк](#разбиение-строк)
   - [Объединение строк](#объединение-строк)
   - [Преобразования "число ↔ строка"](#преобразования-число--строка)
   - [Разбор (парсинг)](#разбор-парсинг)
   - [Повторение, заполнение и выравнивание](#повторение-заполнение-и-выравнивание)
   - [Отступы (indentation)](#отступы-indentation)
   - [Удаление и обрезка](#удаление-и-обрезка)
   - [Операции с префиксами/суффиксами](#операции-с-префиксамисуффиксами)
   - [Поиск](#поиск)
   - [Подсчёт и проверка вхождения](#подсчёт-и-проверка-вхождения)
   - [Замена](#замена)
   - [Экранирование](#экранирование)
   - [Проверка идентификаторов](#проверка-идентификаторов)
   - [Форматирование чисел с плавающей точкой](#форматирование-чисел-с-плавающей-точкой)
   - [Форматирование размеров и инженерная запись](#форматирование-размеров-и-инженерная-запись)
   - [Интерполяция строк (`%` и `format`)](#интерполяция-строк--и-format)
   - [Токенизация](#токенизация)
4. [Рекомендации: какую функцию когда использовать](#рекомендации-какую-функцию-когда-использовать)

---

## Обзор всех функций

| Имя | Тип | Сигнатура (кратко) | Назначение |
|---|---|---|---|
| `isAlphaAscii` | func | `(c: char): bool` | Является ли `c` ASCII-буквой? |
| `isAlphaNumeric` | func | `(c: char): bool` | Буква или цифра? |
| `isDigit` | func | `(c: char): bool` | Является ли `c` цифрой `0-9`? |
| `isSpaceAscii` | func | `(c: char): bool` | Является ли `c` пробельным символом? |
| `isLowerAscii` | func | `(c: char): bool` | Буква нижнего регистра ASCII? |
| `isUpperAscii` | func | `(c: char): bool` | Буква верхнего регистра ASCII? |
| `toLowerAscii` | func | `(c: char\|s: string)` | Преобразовать в нижний регистр (только ASCII) |
| `toUpperAscii` | func | `(c: char\|s: string)` | Преобразовать в верхний регистр (только ASCII) |
| `capitalizeAscii` | func | `(s: string): string` | Сделать первую букву заглавной |
| `nimIdentNormalize` | func | `(s: string): string` | Нормализация как идентификатор Nim |
| `normalize` | func | `(s: string): string` | Нижний регистр + удаление `_` |
| `cmpIgnoreCase` | func | `(a, b: string): int` | Сравнение без учёта регистра |
| `cmpIgnoreStyle` | func | `(a, b: string): int` | Сравнение без учёта "стиля" |
| `split` (итератор/func) | iterator/func | `(s, sep\|seps\|sepStr, maxsplit)` | Разбиение на подстроки |
| `rsplit` (итератор/func) | iterator/func | `(s, sep\|seps\|sepStr, maxsplit)` | Разбиение справа |
| `splitLines` (итератор/func) | iterator/func | `(s, keepEol)` | Разбиение на строки (lines) |
| `splitWhitespace` (итератор/func) | iterator/func | `(s, maxsplit)` | Разбиение по пробельным символам с обрезкой краёв |
| `toBin` | func | `(x: BiggestInt, len)` | Целое → двоичная строка |
| `toOct` | func | `(x: BiggestInt, len)` | Целое → восьмеричная строка |
| `toHex` | func | `(x: SomeInteger \| string)` | Целое/байты → шестнадцатеричная строка |
| `toOctal` | func | `(c: char): string` | Символ → 3-значное восьмеричное число |
| `fromBin` / `fromOct` / `fromHex` | func | `[T](s: string): T` | Разбор двоичного/восьмеричного/hex числа |
| `intToStr` | func | `(x: int, minchars)` | Целое → десятичная строка с нулями слева |
| `parseInt` / `parseBiggestInt` / `parseUInt` / `parseBiggestUInt` | func | `(s: string)` | Разбор десятичных целых |
| `parseFloat` | func | `(s: string): float` | Разбор числа с плавающей точкой |
| `parseBinInt` / `parseOctInt` / `parseHexInt` | func | `(s: string): int` | Разбор bin/oct/hex с префиксами |
| `parseHexStr` | func | `(s: string): string` | Hex-текст → байтовая строка |
| `parseBool` | func | `(s: string): bool` | Разбор `"yes"/"true"/"1"` и т.п. |
| `parseEnum` | func | `[T](s: string[, default: T]): T` | Строка → значение enum |
| `repeat` | func | `(c: char\|s: string, n)` | Повторить символ/строку `n` раз |
| `spaces` | func | `(n: Natural): string` | `n` пробелов |
| `align` | func | `(s, count, padding)` | Выравнивание по правому краю (заполнение слева) |
| `alignLeft` | func | `(s, count, padding)` | Выравнивание по левому краю (заполнение справа) |
| `center` | func | `(s, width, fillChar)` | Центрирование строки |
| `indent` | func | `(s, count, padding)` | Добавить отступ каждой строке |
| `unindent` | func | `(s, count, padding)` | Удалить отступ |
| `indentation` | func | `(s: string): Natural` | Общий отступ всех строк |
| `dedent` | func | `(s, count)` | Удалить общий отступ (только пробелы) |
| `delete` | func | `(s: var string, slice\|first,last)` | Удалить срез символов |
| `startsWith` / `endsWith` | func | `(s, prefix\|suffix: char\|string)` | Проверка префикса/суффикса |
| `continuesWith` | func | `(s, substr, start)` | Содержит ли `s` `substr` начиная с `start`? |
| `removePrefix` / `removeSuffix` | func | `(s: var string, chars\|c\|str)` | Удалить префикс/суффикс на месте |
| `addSep` | func | `(dest: var string, sep, startLen)` | Условно добавить разделитель |
| `allCharsInSet` | func | `(s, theSet): bool` | Все символы входят в множество? |
| `abbrev` | func | `(s, possibilities): int` | Однозначное сокращение |
| `join` | func | `(a: openArray, sep)` | Объединить элементы в строку |
| `initSkipTable` / `find` (с таблицей) | func | `(sub)` / `(a, s, sub, start, last)` | Поиск Boyer–Moore–Horspool |
| `find` | func | `(s, sub: char\|set[char]\|string, start, last)` | Поиск подстроки/символа/множества |
| `rfind` | func | `(s, sub, start, last)` | Поиск с конца |
| `count` | func | `(s, sub: char\|set[char]\|string)` | Подсчёт вхождений |
| `countLines` | func | `(s): int` | Подсчёт строк (lines) |
| `contains` | func | `(s, sub\|chars): bool` | Проверка вхождения |
| `replace` | func | `(s, sub, by)` | Заменить все вхождения |
| `replaceWord` | func | `(s, sub, by)` | Заменить вхождения целых слов |
| `multiReplace` | func | `(s, replacements)` | Несколько замен за один проход |
| `insertSep` | func | `(s, sep, digits)` | Вставить разделители разрядов (тысяч) |
| `escape` / `unescape` | func | `(s, prefix, suffix)` | Экранирование/деэкранирование |
| `validIdentifier` | func | `(s): bool` | Корректный ли идентификатор? |
| `formatBiggestFloat` / `formatFloat` | func | `(f, format, precision, decimalSep)` | Форматирование float |
| `trimZeros` | func | `(x: var string, decimalSep)` | Удаление нулей в конце |
| `formatSize` | func | `(bytes, decimalSep, prefix, includeSpace)` | Человекочитаемые размеры в байтах |
| `formatEng` | func | `(f, precision, trim, siPrefix, unit, ...)` | Инженерная запись |
| `addf` | func | `(s: var string, formatstr, a)` | Эффективный `add(s, formatstr % a)` |
| `%` | func | `(formatstr, a)` | Оператор интерполяции строк |
| `format` | func | `(formatstr, a)` | Аналог `%` с автоматическим преобразованием в строку |
| `strip` | func | `(s, leading, trailing, chars)` | Обрезка ведущих/конечных символов |
| `stripLineEnd` | func | `(s: var string)` | Удаление одного перевода строки в конце |
| `tokenize` | iterator | `(s, seps)` | Возвращает пары `(token, isSep)` |
| `isEmptyOrWhitespace` | func | `(s): bool` | Пустая строка или только пробелы? |

---

## Константы, типы и вспомогательные средства

### `Whitespace*: set[char]`
Все символы, считающиеся пробельными: пробел, табуляция, вертикальная табуляция, возврат каретки, перевод строки, перевод страницы — т.е. `{' ', '\t', '\v', '\r', '\l', '\f'}`.

### `Letters*: set[char]`
Множество ASCII-букв: `{'A'..'Z', 'a'..'z'}`.

### `UppercaseLetters*: set[char]`
`{'A'..'Z'}` — буквы верхнего регистра ASCII.

### `LowercaseLetters*: set[char]`
`{'a'..'z'}` — буквы нижнего регистра ASCII.

### `PunctuationChars*: set[char]`
Все символы пунктуации ASCII: `{'!'..'/', ':'..'@', '['..'`', '{'..'~'}`.

### `Digits*: set[char]`
`{'0'..'9'}`.

### `HexDigits*: set[char]`
`{'0'..'9', 'A'..'F', 'a'..'f'}`.

### `IdentChars*: set[char]`
Символы, из которых может состоять идентификатор Nim: `{'a'..'z', 'A'..'Z', '0'..'9', '_'}`.

### `IdentStartChars*: set[char]`
Символы, с которых может *начинаться* идентификатор: `{'a'..'z', 'A'..'Z', '_'}`.

### `Newlines*: set[char]`
`{'\13', '\10'}` — возврат каретки и перевод строки, символы, которыми может начинаться последовательность перевода строки.

### `PrintableChars*: set[char]`
Объединение `Letters + Digits + PunctuationChars + Whitespace`.

### `AllChars*: set[char]`
Все 256 возможных значений байта, `{'\x00'..'\xFF'}`. В основном полезно для создания *инвертированных* множеств, например `AllChars - Digits`, чтобы с помощью `find` найти первый символ, не являющийся цифрой.

### `SkipTable* = array[char, int]`
Предвычисленная таблица сдвигов, используемая алгоритмом поиска подстроки Boyer–Moore–Horspool (см. `initSkipTable` и вариант `find` с таблицей). Создайте её один раз с помощью `initSkipTable(sub)` и повторно используйте для поиска одной и той же подстроки `sub` в разных строках — это избавляет от повторного вычисления таблицы при каждом вызове.

### `FloatFormatMode* = enum`
Управляет тем, как `formatFloat`/`formatBiggestFloat` отображают числа:
- `ffDefault` — наиболее короткое представление, допускающее обратное преобразование (аналог `%g` в C).
- `ffDecimal` — представление с фиксированной запятой, `precision` знаков после точки.
- `ffScientific` — научная запись с экспонентой `e`, `precision` значащих цифр.

### `BinaryPrefixMode* = enum`
Управляет тем, какие префиксы единиц использует `formatSize`:
- `bpIEC` — двоичные приставки IEC/ISO (`Ki`, `Mi`, `Gi`, … с базой 1024), например `"4KiB"`.
- `bpColloquial` — повседневные SI-подобные названия, применяемые к величинам с базой 1024 (`k`, `M`, `G`, …), например `"4kB"`.

### Реэкспорт из `std/unicode`
Из модуля `std/unicode` для удобства реэкспортированы `toLower` и `toUpper` (преобразование регистра с учётом Unicode), дополняющие ASCII-функции `toLowerAscii`/`toUpperAscii`, определённые здесь.

---

## Основная часть с описанием функций

### Классификация символов

#### `isAlphaAscii(c: char): bool`
Возвращает `true`, если `c` — ASCII-буква (`a-z` или `A-Z`). Проверяется только ASCII; для полноценной поддержки Unicode используйте модуль `unicode`.
```nim
doAssert isAlphaAscii('e') == true
doAssert isAlphaAscii('E') == true
doAssert isAlphaAscii('8') == false
```

#### `isAlphaNumeric(c: char): bool`
Возвращает `true`, если `c` — ASCII-буква или цифра (`a-z`, `A-Z`, `0-9`).
```nim
doAssert isAlphaNumeric('n') == true
doAssert isAlphaNumeric('8') == true
doAssert isAlphaNumeric(' ') == false
```

#### `isDigit(c: char): bool`
Возвращает `true`, если `c` — одна из цифр `0-9`.
```nim
doAssert isDigit('n') == false
doAssert isDigit('8') == true
```

#### `isSpaceAscii(c: char): bool`
Возвращает `true`, если `c` входит в множество `Whitespace`.
```nim
doAssert isSpaceAscii('n') == false
doAssert isSpaceAscii(' ') == true
doAssert isSpaceAscii('\t') == true
```

#### `isLowerAscii(c: char): bool`
Возвращает `true`, если `c` — буква нижнего регистра ASCII. Проверяются только ASCII-символы — для полной поддержки Unicode используйте модуль `unicode`.
```nim
doAssert isLowerAscii('e') == true
doAssert isLowerAscii('E') == false
doAssert isLowerAscii('7') == false
```

#### `isUpperAscii(c: char): bool`
Возвращает `true`, если `c` — буква верхнего регистра ASCII. Проверяются только ASCII-символы.
```nim
doAssert isUpperAscii('e') == false
doAssert isUpperAscii('E') == true
doAssert isUpperAscii('7') == false
```

---

### Преобразование регистра

#### `toLowerAscii(c: char): char`
Возвращает символ `c` в нижнем регистре. Работает только для `A-Z`; остальные символы возвращаются без изменений. Для символов Unicode используйте `unicode.toLower`.
```nim
doAssert toLowerAscii('A') == 'a'
doAssert toLowerAscii('e') == 'e'
```
Замечание о реализации: для буквы верхнего регистра преобразование выполняется инвертированием 5-го бита (`xor 0b0010_0000`) — это стандартный приём ASCII, поскольку буквы верхнего и нижнего регистра отличаются ровно этим битом.

#### `toLowerAscii(s: string): string`
Преобразует каждый символ строки `s` в нижний регистр (только `A-Z` ASCII); остальные символы остаются без изменений.
```nim
doAssert toLowerAscii("FooBar!") == "foobar!"
```
См. также: `normalize` — дополнительно удаляет символы подчёркивания.

#### `toUpperAscii(c: char): char`
Возвращает символ `c` в верхнем регистре. Работает только для `a-z`. Для символов Unicode используйте `unicode.toUpper`.
```nim
doAssert toUpperAscii('a') == 'A'
doAssert toUpperAscii('E') == 'E'
```

#### `toUpperAscii(s: string): string`
Преобразует каждый символ строки `s` в верхний регистр (только `a-z` ASCII).
```nim
doAssert toUpperAscii("FooBar!") == "FOOBAR!"
```

#### `capitalizeAscii(s: string): string`
Делает заглавной только **первую** букву строки `s` (только ASCII `a-z`). Для пустой строки возвращается пустая строка. Если первый символ не буква, он остаётся без изменений.
```nim
doAssert capitalizeAscii("foo") == "Foo"
doAssert capitalizeAscii("-bar") == "-bar"
```

---

### Нормализация и сравнение

#### `nimIdentNormalize(s: string): string`
Нормализует `s` так, как компилятор Nim нормализует идентификаторы для сравнения: переводит в нижний регистр и удаляет все символы подчёркивания, **кроме** первого символа строки. Первый символ сохраняется как есть (только приводится регистр).

> ⚠️ Обратные кавычки (`` ` ``) **не** обрабатываются особым образом — они остаются как есть, пробелы также сохраняются. Альтернативу, учитывающую обратные кавычки, см. в `nimIdentBackticksNormalize` (модуль `dochelpers`).

```nim
doAssert nimIdentNormalize("Foo_bar") == "Foobar"
```

#### `normalize(s: string): string`
Переводит `s` в нижний регистр и удаляет **все** символы подчёркивания (включая первый символ строки, в отличие от `nimIdentNormalize`). Эту функцию **нельзя** использовать для нормализации имён идентификаторов Nim — для этого предназначена `nimIdentNormalize`.
```nim
doAssert normalize("Foo_bar") == "foobar"
doAssert normalize("Foo Bar") == "foo bar"
```

#### `cmpIgnoreCase(a, b: string): int`
Сравнение строк без учёта регистра. Возвращает `0`, если строки равны, отрицательное число, если `a < b`, и положительное, если `a > b` (та же конвенция, что и у `system.cmp`).
```nim
doAssert cmpIgnoreCase("FooBar", "foobar") == 0
doAssert cmpIgnoreCase("bar", "Foo") < 0
doAssert cmpIgnoreCase("Foo5", "foo4") > 0
```

#### `cmpIgnoreStyle(a, b: string): int`
По смыслу эквивалентно `cmp(normalize(a), normalize(b))`, но реализовано без выделения временных строк — то есть сравнение "без учёта стиля": регистр и символы подчёркивания игнорируются. Эту функцию **нельзя** использовать для сравнения имён идентификаторов Nim (для них действуют правила `nimIdentNormalize`, где подчёркивание в первом символе имеет значение); для идентификаторов используйте `macros.eqIdent`.
```nim
doAssert cmpIgnoreStyle("foo_bar", "FooBar") == 0
doAssert cmpIgnoreStyle("foo_bar_5", "FooBar4") > 0
```

> Обе функции сравнения компилируются с отключёнными проверками границ/переполнения (`{.push checks: off.}`), поскольку являются "горячими точками", используемыми самим компилятором Nim.

---

### Разбиение строк

Для каждой операции разбиения в Nim предусмотрены как варианты **итератора**, так и **func** (возвращающие последовательность). Итераторы более экономны по памяти при использовании в цикле `for`, в то время как варианты-функции возвращают `seq[string]` и удобны, когда нужен весь результат сразу (например, для индексирования или `len`).

#### `split(s: string, sep: char, maxsplit: int = -1): string` *(итератор)*
Разбивает `s` по каждому вхождению символа `sep`. Пустые поля между соседними разделителями (или в начале/конце строки) сохраняются как пустые строки.
```nim
for word in split(";;this;is;an;;example;;;", ';'):
  writeLine(stdout, word)
# ""
# ""
# "this"
# "is"
# "an"
# ""
# "example"
# ""
# ""
# ""
```
`maxsplit` ограничивает количество выполняемых разбиений (по умолчанию `-1` — без ограничения); по достижении лимита остаток строки возвращается как последний элемент целиком.

См. также: `rsplit` (разбиение справа), `splitLines`, `splitWhitespace`, а также вариант-функцию `split`, возвращающую `seq`.

#### `split(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): string` *(итератор)*
Разбивает `s` в местах, где встречается максимальная подряд идущая последовательность символов из `seps` — то есть *группа* следующих друг за другом символов-разделителей считается одним разделителем.
```nim
for word in split("this\lis an\texample"):
  writeLine(stdout, word)
# "this"
# "is"
# "an"
# "example"

for word in split("this:is;an$example", {';', ':', '$'}):
  writeLine(stdout, word)
# тот же результат, что и выше

let date = "2012-11-20T22:08:08.398990"
let separators = {' ', '-', ':', 'T'}
for number in split(date, separators):
  writeLine(stdout, number)
# "2012"
# "11"
# "20"
# "22"
# "08"
# "08.398990"
```
> **Примечание:** пустое множество разделителей возвращает исходную строку без изменений (интерпретируется как "разбиение по отсутствующему элементу").

#### `split(s: string, sep: string, maxsplit: int = -1): string` *(итератор)*
Разбивает `s` по каждому вхождению подстроки `sep` (вся строка `sep` действует как один разделитель).
```nim
for word in split("thisDATAisDATAcorrupted", "DATA"):
  writeLine(stdout, word)
# "this"
# "is"
# "corrupted"
```
> **Примечание:** пустая строка-разделитель `""` возвращает исходную строку без изменений.

#### `rsplit(s: string, sep: char | set[char] | string, maxsplit: int = -1[, keepSeparators: bool]): string` *(итератор)*
Работает так же, как соответствующий итератор `split`, но обрабатывает строку **с конца** (справа) к началу. Это имеет значение только при заданном `maxsplit`: в случае `rsplit` сначала выполняются разбиения, ближайшие к *концу* строки, поэтому "неразбитая" часть оказывается в **начале**.
```nim
for piece in "foo:bar".rsplit(':'):
  echo piece
# "bar"
# "foo"

for piece in "foo bar".rsplit(Whitespace):
  echo piece
# "bar"
# "foo"

for piece in "foothebar".rsplit("the"):
  echo piece
# "bar"
# "foo"
```
Перегрузка со строковым разделителем также принимает `keepSeparators: bool` (по умолчанию `false`).
> **Примечание:** пустой разделитель (пустое множество или пустая строка) возвращает исходную строку без изменений.

#### `splitLines(s: string, keepEol = false): string` *(итератор)*
Разбивает `s` на строки (lines). Распознаются все три варианта перевода строки: `LF` (`\n`), `CR` (`\r`) и `CRLF` (`\r\n`) — каждый считается **одним** разделителем строк. По умолчанию символы перевода строки убираются из каждой возвращаемой строки; передайте `keepEol = true`, чтобы оставить их.
```nim
for line in splitLines("\nthis\nis\nan\n\nexample\n"):
  writeLine(stdout, line)
# ""
# "this"
# "is"
# "an"
# ""
# "example"
# ""
```
См. также: `splitWhitespace`, `countLines` (эффективнее, если нужно только количество строк).

#### `splitWhitespace(s: string, maxsplit: int = -1): string` *(итератор)*
Разбивает `s` по последовательностям пробельных символов, **также удаляя пробелы в начале и конце**. Если `maxsplit` положительно, выполняется не более указанного числа разбиений, а оставшийся текст (включая окружающие его пробелы) возвращается как последний элемент в неизменном виде.
```nim
let s = "  foo \t bar  baz  "
for ms in [-1, 1, 2, 3]:
  echo "------ maxsplit = ", ms, ":"
  for item in s.splitWhitespace(maxsplit=ms):
    echo '"', item, '"'
# ------ maxsplit = -1:
# "foo"
# "bar"
# "baz"
# ------ maxsplit = 1:
# "foo"
# "bar  baz  "
# ------ maxsplit = 2:
# "foo"
# "bar"
# "baz  "
# ------ maxsplit = 3:
# "foo"
# "bar"
# "baz"
```

#### `split(s: string, sep: char, maxsplit: int = -1): seq[string]`
Вариант-функция итератора `split` по одному символу, возвращающая `seq`.
```nim
doAssert "a,b,c".split(',') == @["a", "b", "c"]
doAssert "".split(' ') == @[""]
```

#### `split(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): seq[string]`
Вариант-функция итератора `split` по множеству символов, возвращающая `seq`.
```nim
doAssert "a,b;c".split({',', ';'}) == @["a", "b", "c"]
doAssert "".split({' '}) == @[""]
doAssert "empty seps return unsplit s".split({}) == @["empty seps return unsplit s"]
```

#### `split(s: string, sep: string, maxsplit: int = -1): seq[string]`
Вариант-функция итератора `split` по строковому разделителю, возвращающая `seq`. Это самая часто используемая перегрузка для разбора данных в стиле CSV.
```nim
doAssert "a,b,c".split(",") == @["a", "b", "c"]
doAssert "a man a plan a canal panama".split("a ") == @["", "man ", "plan ", "canal panama"]
doAssert "".split("Elon Musk") == @[""]
doAssert "a  largely    spaced sentence".split(" ") == @["a", "", "largely", "", "", "", "spaced", "sentence"]
doAssert "a  largely    spaced sentence".split(" ", maxsplit = 1) == @["a", " largely    spaced sentence"]
doAssert "empty sep returns unsplit s".split("") == @["empty sep returns unsplit s"]
```
Обратите внимание, что при разбиении по литералу `" "` пустые строки для последовательных пробелов сохраняются — если требуется игнорировать повторяющиеся пробелы, используйте `splitWhitespace`.

#### `rsplit(s: string, sep: char, maxsplit: int = -1): seq[string]`
Вариант-функция `rsplit` по одному символу, возвращающая результат в **исходном порядке слева-направо** (внутренне элементы собираются в обратном порядке, а затем разворачиваются).

Типичный пример использования — обработка путей, где требуется отделить последние компоненты:
```nim
var tailSplit = rsplit("Root#Object#Method#Index", '#', maxsplit=1)
# tailSplit == @["Root#Object#Method", "Index"]
```

#### `rsplit(s: string, seps: set[char] = Whitespace, maxsplit: int = -1): seq[string]`
Вариант-функция `rsplit` по множеству символов, в исходном порядке.
```nim
var tailSplit = rsplit("Root#Object#Method#Index", {'#'}, maxsplit=1)
# tailSplit == @["Root#Object#Method", "Index"]
```
> **Примечание:** пустое множество разделителей возвращает исходную строку без изменений.

#### `rsplit(s: string, sep: string, maxsplit: int = -1): seq[string]`
Вариант-функция `rsplit` по строковому разделителю, в исходном порядке.
```nim
doAssert "a  largely    spaced sentence".rsplit(" ", maxsplit = 1) == @["a  largely    spaced", "sentence"]
doAssert "a,b,c".rsplit(",") == @["a", "b", "c"]
doAssert "a man a plan a canal panama".rsplit("a ") == @["", "man ", "plan ", "canal panama"]
doAssert "".rsplit("Elon Musk") == @[""]
doAssert "a  largely    spaced sentence".rsplit(" ") == @["a", "", "largely", "", "", "", "spaced", "sentence"]
doAssert "empty sep returns unsplit s".rsplit("") == @["empty sep returns unsplit s"]
```
> **Примечание:** пустая строка-разделитель возвращает исходную строку без изменений.

#### `splitLines(s: string, keepEol = false): seq[string]`
Вариант-функция итератора `splitLines`, возвращающая `seq`.

#### `splitWhitespace(s: string, maxsplit: int = -1): seq[string]`
Вариант-функция итератора `splitWhitespace`, возвращающая `seq`.

---

### Токенизация

#### `tokenize(s: string, seps: set[char] = Whitespace): tuple[token: string, isSep: bool]` *(итератор)*
Разбивает `s` на чередующиеся подстроки — последовательности символов-разделителей и последовательности обычных символов, возвращая пары `(token, isSep)`. В отличие от `split`, возвращаются **обе** части — и разделители, и содержимое — ничего не отбрасывается.
```nim
for word in tokenize("  this is an  example  "):
  writeLine(stdout, word)
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
Полезно, когда нужно **восстановить** исходную строку точно, сохраняя исходное расположение пробелов/разделителей, например для форматтера или генератора текста.

---

### Объединение строк

#### `join(a: openArray[string], sep: string = ""): string`
Объединяет все строки из `a`, вставляя `sep` между соседними элементами (но не перед первым и не после последнего).
```nim
doAssert join(["A", "B", "Conclusion"], " -> ") == "A -> B -> Conclusion"
```
Реализация заранее вычисляет необходимый объём памяти и выделяет результирующую строку за один раз, что эффективно даже для больших массивов.

#### `join[T: not string](a: openArray[T], sep: string = ""): string`
Универсальная перегрузка для массивов любого нестрокового типа `T`: каждый элемент преобразуется в строку через `$` и объединяется с разделителем `sep`.
```nim
doAssert join([1, 2, 3], " -> ") == "1 -> 2 -> 3"
```

---

### Преобразования "число ↔ строка"

#### `toBin(x: BiggestInt, len: Positive): string`
Преобразует `x` в двоичное представление длиной **ровно** `len` символов (дополняется нулями слева либо обрезается до младших `len` бит, если `x` не помещается). Префикс `0b` не добавляется.
```nim
let
  a = 29
  b = 257
doAssert a.toBin(8) == "00011101"
doAssert b.toBin(8) == "00000001"   # обрезано — 257 не помещается в 8 бит
doAssert b.toBin(9) == "100000001"
```

#### `toOct(x: BiggestInt, len: Positive): string`
Преобразует `x` в восьмеричное представление длиной ровно `len` символов, без префикса `0o`. Не путать с `toOctal(c: char)`, которая преобразует *порядковое значение одного символа*.
```nim
let
  a = 62
  b = 513
doAssert a.toOct(3) == "076"
doAssert b.toOct(3) == "001"   # обрезано до 3 восьмеричных цифр
doAssert b.toOct(5) == "01001"
```

#### `toHex[T: SomeInteger](x: T, len: Positive): string`
Преобразует `x` в шестнадцатеричное представление длиной ровно `len` символов. `x` рассматривается как **беззнаковое** значение (поэтому отрицательное число отображается через своё представление в дополнительном коде). Префикс `0x` не добавляется.
```nim
let
  a = 62'u64
  b = 4097'u64
doAssert a.toHex(3) == "03E"
doAssert b.toHex(3) == "001"     # обрезано до 3 hex-цифр
doAssert b.toHex(4) == "1001"
doAssert toHex(62, 3) == "03E"
doAssert toHex(-8, 6) == "FFFFF8" # представление в дополнительном коде
```

#### `toHex[T: SomeInteger](x: T): string`
Сокращение для `toHex(x, T.sizeof * 2)` — то есть результат имеет длину, равную полной разрядности `T` в hex-цифрах (2 hex-цифры на байт).
```nim
doAssert toHex(1984'i64) == "00000000000007C0"
doAssert toHex(1984'i16) == "07C0"
```

#### `toHex(s: string): string`
Преобразует строку байтов в её шестнадцатеричное представление. Длина результата ровно в два раза больше длины входной строки. Префикс `0x` не добавляется.
```nim
let
  a = "1"
  b = "A"
  c = "\0\255"
doAssert a.toHex() == "31"
doAssert b.toHex() == "41"
doAssert c.toHex() == "00FF"
```
См. также: `parseHexStr` для обратной операции (hex-текст → байты).

#### `toOctal(c: char): string`
Преобразует порядковое значение символа `c` в восьмеричное представление, всегда ровно 3 символа в длину (возможно, с ведущими нулями, без префикса `0o`). Не путать с `toOct(x: BiggestInt, len)`.
```nim
doAssert toOctal('1') == "061"
doAssert toOctal('A') == "101"
doAssert toOctal('a') == "141"
doAssert toOctal('!') == "041"
```

#### `fromBin[T: SomeInteger](s: string): T`
Разбирает двоичный целочисленный литерал из `s` в тип `T`. Допускается необязательный префикс `0b`/`0B`; символы подчёркивания в `s` игнорируются (поэтому группировка цифр вида `0b_0100_1000` разрешена). Если `s` не является корректным двоичным литералом, выбрасывается `ValueError`.
**Переполнение не проверяется** — если значение слишком велико для `T`, сохраняются только младшие биты, которые помещаются (тихое усечение/перенос).
```nim
let s = "0b_0100_1000_1000_1000_1110_1110_1001_1001"
doAssert fromBin[int](s) == 1216933529
doAssert fromBin[int8](s) == 0b1001_1001'i8
doAssert fromBin[int8](s) == -103'i8        # перенос как знаковое 8-битное
doAssert fromBin[uint8](s) == 153
doAssert s.fromBin[:int16] == 0b1110_1110_1001_1001'i16
doAssert s.fromBin[:uint64] == 1216933529'u64
```

#### `fromOct[T: SomeInteger](s: string): T`
Разбирает восьмеричный целочисленный литерал из `s` в тип `T`. Допускается необязательный префикс `0o`/`0O`; символы подчёркивания игнорируются. При некорректном литерале выбрасывается `ValueError`. **Проверка переполнения отсутствует** — усекается до младших цифр, помещающихся в `T`.
```nim
let s = "0o_123_456_777"
doAssert fromOct[int](s) == 21913087
doAssert fromOct[int8](s) == 0o377'i8
doAssert fromOct[int8](s) == -1'i8
doAssert fromOct[uint8](s) == 255'u8
doAssert s.fromOct[:int16] == 24063'i16
doAssert s.fromOct[:uint64] == 21913087'u64
```

#### `fromHex[T: SomeInteger](s: string): T`
Разбирает шестнадцатеричный целочисленный литерал из `s` в тип `T`. Допускается необязательный префикс `0x`, `0X` или `#`; символы подчёркивания игнорируются. При некорректном литерале выбрасывается `ValueError`. **Проверка переполнения отсутствует** — усекается до младших цифр, помещающихся в `T`.
```nim
let s = "0x_1235_8df6"
doAssert fromHex[int](s) == 305499638
doAssert fromHex[int8](s) == 0xf6'i8
doAssert fromHex[int8](s) == -10'i8
doAssert fromHex[uint8](s) == 246'u8
doAssert s.fromHex[:int16] == -29194'i16
doAssert s.fromHex[:uint64] == 305499638'u64
```

#### `intToStr(x: int, minchars: Positive = 1): string`
Преобразует `x` в его десятичное строковое представление, дополняя слева символами `'0'` до тех пор, пока результат не достигнет длины не менее `minchars`. Знак минус (для отрицательных чисел) добавляется *после* дополнения нулями абсолютного значения.
```nim
doAssert intToStr(1984) == "1984"
doAssert intToStr(1984, 6) == "001984"
```

#### `insertSep(s: string, sep = '_', digits = 3): string`
Вставляет символ-разделитель `sep` через каждые `digits` символов, считая от **правого** конца `s` (то есть аналогично группировке цифр числа по тысячам). Хотя функция работает с любой строкой, она предназначена для строк, представляющих числа — необязательный нечисловой префикс (например, знак `-`) сохраняется и не учитывается при группировке.
```nim
doAssert insertSep("1000000") == "1_000_000"
```

---

### Разбор (парсинг)

#### `parseInt(s: string): int`
Разбирает десятичное целое число из `s`. **Вся** строка должна быть корректным целым числом (включая необязательный знак), иначе выбрасывается `ValueError`.
```nim
doAssert parseInt("-0042") == -42
```

#### `parseBiggestInt(s: string): BiggestInt`
То же, что `parseInt`, но возвращает `BiggestInt` (наибольший знаковый целочисленный тип) — для значений, которые могут не поместиться в обычный `int`.

#### `parseUInt(s: string): uint`
Разбирает десятичное **беззнаковое** целое число из `s`. Вся строка должна быть корректной, иначе выбрасывается `ValueError`.

#### `parseBiggestUInt(s: string): BiggestUInt`
То же, что `parseUInt`, но возвращает `BiggestUInt`.

#### `parseFloat(s: string): float`
Разбирает десятичное число с плавающей точкой из `s`. Вся строка должна быть корректным числом, иначе выбрасывается `ValueError`. Специальные значения `nan`, `inf`, `-inf` распознаются без учёта регистра.
```nim
doAssert parseFloat("3.14") == 3.14
doAssert parseFloat("inf") == 1.0/0
```

#### `parseBinInt(s: string): int`
Разбирает двоичное целое число из `s`. Допустим необязательный префикс `0b`/`0B`; символы подчёркивания игнорируются. При некорректной или пустой строке выбрасывается `ValueError`.
```nim
let
  a = "0b11_0101"
  b = "111"
doAssert a.parseBinInt() == 53
doAssert b.parseBinInt() == 7
```

#### `parseOctInt(s: string): int`
Разбирает восьмеричное целое число из `s`. Допустим необязательный префикс `0o`/`0O`; символы подчёркивания игнорируются. При некорректной или пустой строке выбрасывается `ValueError`.

#### `parseHexInt(s: string): int`
Разбирает шестнадцатеричное целое число из `s`. Допустим необязательный префикс `0x`, `0X` или `#`; символы подчёркивания игнорируются. При некорректной или пустой строке выбрасывается `ValueError`.

#### `parseHexStr(s: string): string`
Преобразует строку в hex-кодировке (каждый байт представлен ровно двумя hex-цифрами, регистронезависимо) в соответствующую байтовую строку. Выбрасывает `ValueError`, если `s` имеет нечётную длину или содержит некорректную hex-цифру.
```nim
let
  a = "41"
  b = "3161"
  c = "00ff"
doAssert parseHexStr(a) == "A"
doAssert parseHexStr(b) == "1a"
doAssert parseHexStr(c) == "\0\255"
```
См. также: `toHex(s: string)` для обратной операции.

#### `parseBool(s: string): bool`
Разбирает текстовое значение "истина/ложь" (без учёта регистра и символов подчёркивания, через `normalize`):
- `true` для: `y`, `yes`, `true`, `1`, `on`.
- `false` для: `n`, `no`, `false`, `0`, `off`.
- Любое другое значение вызывает исключение `ValueError`.
```nim
let a = "n"
doAssert parseBool(a) == false
```

#### `parseEnum[T: enum](s: string): T`
Разбирает `s` в значение перечисления `T`, сравнивая имена в "стиль-независимой" манере (без учёта регистра и подчёркиваний, кроме первой буквы, которая остаётся регистрозависимой — те же правила, что у `cmpIgnoreStyle`/`nimIdentNormalize`). Выбрасывает `ValueError`, если `s` не соответствует ни одному полю. Если у `T` несколько полей нормализуются к одинаковой строке — это ошибка времени компиляции.
```nim
type
  MyEnum = enum
    first = "1st",
    second,
    third = "3rd"

doAssert parseEnum[MyEnum]("1_st") == first
doAssert parseEnum[MyEnum]("second") == second
doAssertRaises(ValueError):
  echo parseEnum[MyEnum]("third")
```

#### `parseEnum[T: enum](s: string, default: T): T`
То же, что и выше, но возвращает `default`, если `s` не соответствует ни одному полю, вместо выброса `ValueError`.
```nim
type
  MyEnum = enum
    first = "1st",
    second,
    third = "3rd"

doAssert parseEnum[MyEnum]("1_st") == first
doAssert parseEnum[MyEnum]("second") == second
doAssert parseEnum[MyEnum]("last", third) == third
```

---

### Повторение, заполнение и выравнивание

#### `repeat(c: char, count: Natural): string`
Возвращает строку из `count` копий символа `c`.
```nim
let a = 'z'
doAssert a.repeat(5) == "zzzzz"
```

#### `repeat(s: string, n: Natural): string`
Возвращает строку `s`, повторённую подряд `n` раз (т.е. `n` копий `s` без разделителя).
```nim
doAssert "+ foo +".repeat(3) == "+ foo ++ foo ++ foo +"
```

#### `spaces(n: Natural): string`
Возвращает строку из `n` пробельных символов. Реализована как `repeat(' ', n)`. Удобна для ручного выравнивания текста по левому краю.
```nim
let
  width = 15
  text1 = "Hello user!"
  text2 = "This is a very long string"
doAssert text1 & spaces(max(0, width - text1.len)) & "|" == "Hello user!    |"
doAssert text2 & spaces(max(0, width - text2.len)) & "|" == "This is a very long string|"
```
См. также: `align`, `alignLeft`, `indent`, `center`.

#### `align(s: string, count: Natural, padding = ' '): string`
**Выравнивает по правому краю**: вставляет символы `padding` *перед* `s` до тех пор, пока общая длина не станет равной `count`. Если `s.len >= count`, `s` возвращается без изменений (никогда не обрезается). Для выравнивания по левому краю используйте `alignLeft`.
```nim
assert align("abc", 4) == " abc"
assert align("a", 0) == "a"
assert align("1232", 6) == "  1232"
assert align("1232", 6, '#') == "##1232"
```

#### `alignLeft(s: string, count: Natural, padding = ' '): string`
**Выравнивает по левому краю**: добавляет символы `padding` *после* `s` до тех пор, пока общая длина не станет равной `count`. Если `s.len >= count`, `s` возвращается без изменений. Для выравнивания по правому краю используйте `align`.
```nim
assert alignLeft("abc", 4) == "abc "
assert alignLeft("a", 0) == "a"
assert alignLeft("1232", 6) == "1232  "
assert alignLeft("1232", 6, '#') == "1232##"
```

#### `center(s: string, width: int, fillChar: char = ' '): string`
Центрирует `s` в строке длиной `width`, используя `fillChar` для заполнения с обеих сторон. Если требуемое суммарное заполнение нечётное, **дополнительный** символ заполнения добавляется справа (т.е. заполнение слева равно `(width - s.len) div 2`). Если `width <= s.len`, `s` возвращается без изменений.
```nim
let a = "foo"
doAssert a.center(2) == "foo"
doAssert a.center(5) == " foo "
doAssert a.center(6) == " foo  "
```

---

### Отступы (indentation)

#### `indent(s: string, count: Natural, padding: string = " "): string`
Добавляет `count` копий `padding` в начало **каждой строки** `s` (строки получаются через `splitLines`, затем объединяются через `\n`).
> ⚠️ Эта функция **не** сохраняет исходные символы перевода строки `s` — все переводы строк в результате становятся `\n`, независимо от того, использовался ли во входной строке `\r\n`, `\r` или `\n`.
```nim
doAssert indent("First line\c\l and second line.", 2) == "  First line\l   and second line."
```

#### `unindent(s: string, count: Natural = int.high, padding: string = " "): string`
Удаляет до `count` вхождений `padding` из начала **каждой строки** `s`. Если у строки меньше `count` ведущих копий `padding`, удаляются только реально присутствующие. Значение по умолчанию `count = int.high` означает "удалить весь ведущий `padding` у каждой строки".
> ⚠️ Как и `indent`, эта функция не сохраняет исходные символы перевода строки — результат объединяется через `\n`.
```nim
let x = """
  Hello
    There
""".unindent()

doAssert x == "Hello\nThere\n"
```
См. также: `dedent` для удаления только того отступа, который является *общим* для всех строк.

#### `indentation(s: string): Natural`
Возвращает количество ведущих пробелов, общее для **всех** непустых (не состоящих только из пробелов) строк `s` (строки, состоящие только из пробельных символов, при вычислении минимума игнорируются). Возвращает `0`, если `s` пуста или не содержит непустых строк. *(Доступно начиная с Nim 1.3.)*

#### `dedent(s: string, count: Natural = indentation(s)): string`
Аналогично `unindent`, но значение `count` по умолчанию равно результату `indentation(s)` — то есть по умолчанию удаляется только тот отступ, который **является общим для всех строк**, при этом сохраняется *относительная* дополнительная отступка. Поддерживает только пробел (`" "`) как символ заполнения.
> ⚠️ Не сохраняет исходные символы перевода строки (результат использует `\n`). *(Доступно начиная с Nim 1.3.)*
```nim
let x = """
  Hello
    There
""".dedent()

doAssert x == "Hello\n  There\n"
```
Сравните с примером для `unindent` выше: `dedent` сохраняет дополнительный двухпробельный отступ строки `"There"` относительно `"Hello"`, в то время как `unindent()` (с поведением по умолчанию — удалить *все* ведущие пробелы) полностью его убирает.

---

### Удаление и обрезка

#### `delete(s: var string, slice: Slice[int])`
Удаляет символы по индексам `s[slice]` **на месте**, сдвигая последующие символы влево. Если срез содержит индексы вне диапазона (при включённых проверках границ), выбрасывается `IndexDefect`. Это строковый аналог `sequtils.delete` для последовательностей.
```nim
var a = "abcde"
doAssertRaises(IndexDefect): a.delete(4..5)
assert a == "abcde"
a.delete(4..4)
assert a == "abcd"
a.delete(1..2)
assert a == "ad"
a.delete(1..<1)  # пустой срез — ничего не делает
assert a == "ad"
```

#### `delete(s: var string, first, last: int)` — **устарела**
Старая форма `delete` с двумя целочисленными аргументами. Удаляет символы на позициях `first..last` (включительно). **Устарела** — используйте `delete(s, first..last)` (перегрузку с `Slice[int]`). В отличие от варианта со срезом, эта форма усекает `last` до `s.len`, а не выбрасывает исключение при выходе за границы.
```nim
var a = "abracadabra"
a.delete(4, 5)
doAssert a == "abradabra"
a.delete(1, 6)
doAssert a == "ara"
a.delete(2, 999)
doAssert a == "ar"
```

#### `strip(s: string, leading = true, trailing = true, chars: set[char] = Whitespace): string`
Возвращает копию `s` без ведущих и/или конечных символов из `chars` (по умолчанию `chars` — это `Whitespace`). Параметрами `leading`/`trailing` управляется, какие концы обрезаются; если оба равны `false`, `s` возвращается без изменений.
```nim
let a = "  vhellov   "
let b = strip(a)
doAssert b == "vhellov"

doAssert a.strip(leading = false) == "  vhellov"
doAssert a.strip(trailing = false) == "vhellov   "

doAssert b.strip(chars = {'v'}) == "hello"
doAssert b.strip(leading = false, chars = {'v'}) == "vhello"

let c = "blaXbla"
doAssert c.strip(chars = {'b', 'a'}) == "laXbl"
doAssert c.strip(chars = {'b', 'a', 'l'}) == "X"
```
См. также: `strbasics.strip` для версии "на месте"; `stripLineEnd` для удаления только перевода строки в конце.

#### `stripLineEnd(s: var string)`
Удаляет **одну** последовательность перевода строки в конце `s` на месте: `\r\n`, `\n`, `\r`, `\v` или `\f` (не более одного такого экземпляра, даже если `s` заканчивается несколькими переводами строк). Также известна как "chomp". Полезна после чтения строки через `osproc.execCmdEx` или похожие API, которые могут оставлять перевод строки в конце.
```nim
var s = "foo\n\n"
s.stripLineEnd
doAssert s == "foo\n"   # удалён только ОДИН завершающий \n
s = "foo\r\n"
s.stripLineEnd
doAssert s == "foo"     # \r\n считается одной последовательностью
```

---

### Операции с префиксами/суффиксами

#### `startsWith(s: string, prefix: char): bool`
Возвращает `true`, если `s` непуста и её первый символ равен `prefix`.
```nim
let a = "abracadabra"
doAssert a.startsWith('a') == true
doAssert a.startsWith('b') == false
```

#### `startsWith(s, prefix: string): bool`
Возвращает `true`, если `s` начинается со строки `prefix`. Если `prefix == ""`, всегда возвращается `true`.
```nim
let a = "abracadabra"
doAssert a.startsWith("abra") == true
doAssert a.startsWith("bra") == false
```

#### `endsWith(s: string, suffix: char): bool`
Возвращает `true`, если `s` непуста и её последний символ равен `suffix`.
```nim
let a = "abracadabra"
doAssert a.endsWith('a') == true
doAssert a.endsWith('b') == false
```

#### `endsWith(s, suffix: string): bool`
Возвращает `true`, если `s` заканчивается строкой `suffix`. Если `suffix == ""`, всегда возвращается `true`.
```nim
let a = "abracadabra"
doAssert a.endsWith("abra") == true
doAssert a.endsWith("dab") == false
```

#### `continuesWith(s, substr: string, start: Natural): bool`
Возвращает `true`, если начиная с индекса `start` в `s` следующие символы точно совпадают с `substr` (т.е. `s[start ..< start+substr.len] == substr`). Пустая строка `substr` всегда даёт `true`.
```nim
let a = "abracadabra"
doAssert a.continuesWith("ca", 4) == true
doAssert a.continuesWith("ca", 5) == false
doAssert a.continuesWith("dab", 6) == true
```

#### `removePrefix(s: var string, chars: set[char] = Newlines)`
Удаляет **на месте** все ведущие символы `s`, принадлежащие `chars` (по умолчанию `Newlines`). Повторяющиеся/разные символы из множества удаляются до тех пор, пока не встретится символ, не входящий в множество.
```nim
var userInput = "\r\n*~Hello World!"
userInput.removePrefix
doAssert userInput == "*~Hello World!"
userInput.removePrefix({'~', '*'})
doAssert userInput == "Hello World!"

var otherInput = "?!?Hello!?!"
otherInput.removePrefix({'!', '?'})
doAssert otherInput == "Hello!?!"
```

#### `removePrefix(s: var string, c: char)`
Удаляет на месте все ведущие вхождения одного символа `c`. Эквивалентно `removePrefix(s, chars = {c})`.
```nim
var ident = "pControl"
ident.removePrefix('p')
doAssert ident == "Control"
```

#### `removePrefix(s: var string, prefix: string)`
Удаляет на месте **одно** вхождение `prefix` из начала `s`, но только если `s` действительно начинается с `prefix` (и `prefix` непуст). В отличие от перегрузок с `chars`/`char`, повторное применение не происходит — удаляется только первое совпадение.
```nim
var answers = "yesyes"
answers.removePrefix("yes")
doAssert answers == "yes"
```

#### `removeSuffix(s: var string, chars: set[char] = Newlines)`
Удаляет на месте все конечные символы `s`, принадлежащие `chars` (по умолчанию `Newlines`).
```nim
var userInput = "Hello World!*~\r\n"
userInput.removeSuffix
doAssert userInput == "Hello World!*~"
userInput.removeSuffix({'~', '*'})
doAssert userInput == "Hello World!"

var otherInput = "Hello!?!"
otherInput.removeSuffix({'!', '?'})
doAssert otherInput == "Hello"
```

#### `removeSuffix(s: var string, c: char)`
Удаляет на месте все конечные вхождения одного символа `c`. Эквивалентно `removeSuffix(s, chars = {c})`.
```nim
var table = "users"
table.removeSuffix('s')
doAssert table == "user"

var dots = "Trailing dots......."
dots.removeSuffix('.')
doAssert dots == "Trailing dots"
```

#### `removeSuffix(s: var string, suffix: string)`
Удаляет на месте **одно** вхождение `suffix` из конца `s`, но только если `s` действительно заканчивается на `suffix`. Повторное применение не выполняется.
```nim
var answers = "yeses"
answers.removeSuffix("es")
doAssert answers == "yes"
```

---

### Поиск

#### `initSkipTable(a: var SkipTable, sub: string)`
Инициализирует существующую таблицу `SkipTable` `a` для поиска подстроки `sub` с помощью предобработки алгоритма Boyer–Moore–Horspool. Используйте это (или `initSkipTable(sub)`) один раз, а затем повторно используйте таблицу для нескольких вызовов `find(a, s, sub, ...)` с той же `sub` для повышения производительности.

#### `initSkipTable(sub: string): SkipTable`
Возвращает новую, инициализированную с нуля таблицу `SkipTable` для `sub`. Удобная обёртка над перегрузкой с параметром `var`.

#### `find(a: SkipTable, s, sub: string, start: Natural = 0, last = -1): int`
Ищет `sub` внутри `s[start..last]`, используя предвычисленную таблицу `SkipTable` (алгоритм Boyer–Moore–Horspool). Если `last < 0`, по умолчанию принимается `s.high`. Возвращает индекс первого совпадения (относительно `s[0]`), либо `-1`, если `sub` не найдена. Пустая `sub` соответствует совпадению немедленно в позиции `start`. Поиск регистрозависимый.

#### `find(s: string, sub: char, start: Natural = 0, last = -1): int`
Ищет символ `sub` в `s[start..last]`. Если `last` не указано или отрицательно, по умолчанию принимается `s.high`. Возвращает индекс первого совпадения относительно `s[0]` (вычтите `start` самостоятельно, если нужен индекс относительно `start`), либо `-1`, если не найдено. Поиск регистрозависимый. На "родных" целевых платформах для скорости используется функция `memchr` из библиотеки C, если это возможно.
См. также: `rfind`, `replace(s, sub: char, by: char)`.

#### `find(s: string, chars: set[char], start: Natural = 0, last = -1): int`
Ищет первый символ в `s[start..last]`, принадлежащий множеству `chars`. Возвращает его индекс (относительно `s[0]`), либо `-1`, если ни один символ `s[start..last]` не входит в `chars`.
См. также: `rfind`, `multiReplace`. Распространённый приём — `find(s, AllChars - Digits)` для поиска первого символа, *не являющегося* цифрой.

#### `find(s, sub: string, start: Natural = 0, last = -1): int`
Ищет подстроку `sub` в `s[start..last]`. Если `last` не указано или отрицательно, по умолчанию принимается `s.high`. Возвращает индекс первого совпадения относительно `s[0]`, либо `-1`, если не найдено. Поиск регистрозависимый. Внутренне для подстрок длины 1 используется поиск по одному символу, для платформ Linux/BSD/macOS (и только когда `last < 0`) — функция `memmem`, в остальных случаях — поиск по таблице `SkipTable` (Boyer–Moore–Horspool).
См. также: `rfind`, `replace(s, sub: string, by: string)`.

#### `rfind(s: string, sub: char, start: Natural = 0, last = -1): int`
Ищет `sub` в `s[start..last]`, но просматривает строку с **конца** к началу (`start`), то есть находит *последнее* вхождение. Если `last` не указано, по умолчанию принимается `s.high`. Возвращает индекс относительно `s[0]`, либо `-1`, если не найдено.

#### `rfind(s: string, chars: set[char], start: Natural = 0, last = -1): int`
Аналогично `find(s, chars, ...)`, но просматривает строку с конца к началу (`start`), возвращая индекс *последнего* совпавшего символа, либо `-1`, если не найдено.

#### `rfind(s, sub: string, start: Natural = 0, last = -1): int`
Аналогично `find(s, sub, ...)`, но просматривает строку с конца к началу (`start`), возвращая индекс *последнего* вхождения `sub`, либо `-1`, если не найдено. Для пустой `sub` возвращает `max(start, last)` (или `max(start, s.len)`, если `last < 0`) — то есть самую правую допустимую позицию.

---

### Подсчёт и проверка вхождения

#### `count(s: string, sub: char): int`
Подсчитывает, сколько раз символ `sub` встречается в `s`.

#### `count(s: string, subs: set[char]): int`
Подсчитывает, сколько символов `s` принадлежит множеству `subs`. Множество `subs` должно быть непустым (проверяется через `doAssert`).

#### `count(s: string, sub: string, overlapping: bool = false): int`
По умолчанию подсчитывает неперекрывающиеся вхождения подстроки `sub` в `s`. Если `overlapping = true`, вхождения могут перекрываться (поиск продвигается на 1 символ после каждого совпадения вместо `sub.len`). `sub` должна быть непустой (проверяется через `doAssert`).

#### `countLines(s: string): int`
Возвращает количество строк (lines) в `s`. Эквивалентно `len(splitLines(s))`, но значительно эффективнее, так как не строит промежуточных строк/последовательностей — функция просто сканирует `s` один раз, считая последовательности перевода строки (`\r`, `\n`, `\r\n`, каждая считается один раз) и добавляет 1. Строка может быть пустой.
```nim
doAssert countLines("First line\l and second line.") == 2
```

#### `contains(s, sub: string): bool`
Возвращает `true`, если `sub` встречается где-либо в `s`. Эквивалентно `find(s, sub) >= 0`. Эта процедура стоит за операторами `in`/`notin` Nim для строк (например, `"sub" in s`).

#### `contains(s: string, chars: set[char]): bool`
Возвращает `true`, если хотя бы один символ `s` принадлежит `chars`. Эквивалентно `find(s, chars) >= 0`.

#### `allCharsInSet(s: string, theSet: set[char]): bool`
Возвращает `true`, если **каждый** символ `s` принадлежит `theSet`. Пустая строка возвращает `true` (тривиально истинно).
```nim
doAssert allCharsInSet("aeea", {'a', 'e'}) == true
doAssert allCharsInSet("", {'a', 'e'}) == true
```

#### `isEmptyOrWhitespace(s: string): bool`
Возвращает `true`, если `s` пуста или состоит исключительно из символов из `Whitespace`. Реализована через `allCharsInSet(s, Whitespace)`.

---

### Замена

#### `replace(s, sub: string, by = ""): string`
Возвращает копию `s`, в которой **все** вхождения `sub` заменены на `by`. Если `sub` пуста, `s` возвращается без изменений. Для подстроки `sub` длины 1 используется быстрый посимвольный поиск; для более длинных — таблица `SkipTable` (Boyer–Moore–Horspool).
См. также: `find`, `replace(s, sub, by: char)` для одиночных символов, `replaceWord`, `multiReplace` для нескольких подстрок/символов за один раз.

#### `replace(s: string, sub, by: char): string`
Возвращает копию `s`, в которой каждое вхождение символа `sub` заменено на символ `by`. Оптимизированный частный случай строковой `replace` для одиночных символов (результат имеет ту же длину, что и `s`).

#### `replaceWord(s, sub: string, by = ""): string`
Аналогично `replace`, но заменяет только те вхождения `sub`, которые окружены **границами слова** — то есть символ непосредственно до и непосредственно после совпадения (если он есть) *не* должен быть "символом слова" (`'a'..'z'`, `'A'..'Z'`, `'0'..'9'`, `'_'` или любой байт `'\128'..'\255'`). Это сопоставимо с `\b` в регулярных выражениях. Если `sub` пуста, `s` возвращается без изменений.

#### `multiReplace(s: string, replacements: varargs[(string, string)]): string`
Выполняет несколько замен подстрок за **один проход слева направо** по `s`, что эффективнее, чем последовательные вызовы `replace`. Правила:
- В каждой позиции, если может совпасть несколько правил, применяется **первое подходящее** (в порядке аргументов).
- После применения замены сканирование продолжается **после** совпавшего (исходного) текста — перекрытия с *текстом замены* не учитываются, и подстроки, пересекающие границу замены, не сопоставляются.
- Если результат не длиннее входной строки, требуется только одно выделение памяти.
```nim
# Замена местами вхождений 'a' и 'b':
doAssert multireplace("abba", [("a", "b"), ("b", "a")]) == "baab"

# Вторая замена ("ab") совпадает и выполняется первой, после чего сканирование
# продолжается с 'c', поэтому замена "bc" так и не совпадает и пропускается.
doAssert multireplace("abc", [("bc", "x"), ("ab", "_b")]) == "_bc"
```

#### `multiReplace(s: openArray[char], replacements: varargs[(set[char], char)]): string`
Однопроходная замена на уровне **символов**: для каждого символа `s` определяющим становится первое правило `(set[char], char)`, множество которого содержит этот символ; если ни одно правило не подходит, символ остаётся без изменений. Полезно для очистки строк (например, удаления символов, недопустимых в именах файлов).
```nim
const WinSanitationRules = [
  ({'\0'..'\31'}, ' '),
  ({'"'}, '\''),
  ({'/', '\\', ':', '|'}, '-'),
  ({'*', '?', '<', '>'}, '_'),
]
const file = "a/file:with?invalid*chars.txt"
doAssert file.multiReplace(WinSanitationRules) == "a-file-with_invalid_chars.txt"
```

---

### Экранирование

#### `escape(s: string, prefix = "\"", suffix = "\""): string`
Возвращает экранированное представление `s`, обёрнутое в `prefix`/`suffix` (оба по умолчанию равны `"`, так что результат выглядит как строковый литерал Nim). Правила экранирования:
- Байты `'\0'..'\31'` и `'\127'..'\255'` становятся `\xHH` (двузначное шестнадцатеричное число в верхнем регистре).
- `\` становится `\\`.
- `'` становится `\'`.
- `"` становится `\"`.
- Все остальные символы копируются без изменений.

> Примечание: эта схема экранирования **отличается** от `system.addEscapedChar` (которая создаёт совместимые с исходным кодом Nim экранирующие последовательности, например `\n`, `\t` и т.п.) — `escape` всегда использует `\xHH` для непечатаемых символов/байтов со значением выше 127.

См. также: `unescape` для обратной операции.

#### `unescape(s: string, prefix = "\"", suffix = "\""): string`
Выполняет обратную операцию по отношению к `escape`: удаляет `prefix` из начала и `suffix` из конца `s` (выбрасывая `ValueError`, если они отсутствуют) и преобразует последовательности `\xHH`, `\\`, `\'`, `\"` обратно в исходные байты. Любая другая последовательность `\X` передаётся в результат буквально как `\X`.

---

### Проверка идентификаторов

#### `validIdentifier(s: string): bool`
Возвращает `true`, если `s` является синтаксически корректным идентификатором: первый символ должен принадлежать `IdentStartChars` (буква или `_`), а каждый последующий символ — `IdentChars` (буквы, цифры или `_`). Пустая строка не является корректным идентификатором.
```nim
doAssert "abc_def08".validIdentifier
```

---

### Сопоставление сокращений

#### `abbrev(s: string, possibilities: openArray[string]): int`
Находит индекс элемента в `possibilities`, для которого `s` является однозначным префиксом:
- Возвращает индекс совпавшего элемента, если ровно один элемент начинается с `s`.
- Возвращает `-1`, если **ни один** элемент не начинается с `s`.
- Возвращает `-2`, если **несколько** элементов начинаются с `s` (неоднозначность) — *кроме случая*, когда `s` сама является точным совпадением одного из них: тогда это точное совпадение побеждает, несмотря на другие совпадения по префиксу.
```nim
doAssert abbrev("fac", ["college", "faculty", "industry"]) == 1
doAssert abbrev("foo", ["college", "faculty", "industry"]) == -1 # не найдено
doAssert abbrev("fac", ["college", "faculty", "faculties"]) == -2 # неоднозначно
doAssert abbrev("college", ["college", "colleges", "industry"]) == 0
```

---

### Прочие вспомогательные средства

#### `addSep(dest: var string, sep = ", ", startLen: Natural = 0)`
Добавляет `sep` к `dest` **только если** `dest.len > startLen`. Сокращённая запись для:
```nim
if dest.len > startLen: add(dest, sep)
```
Предназначена для инкрементального построения вывода с разделителями, когда разделитель должен быть пропущен перед самым первым элементом.
```nim
var arr = "["
for x in items([2, 3, 5, 7, 11]):
  addSep(arr, startLen = len("["))
  add(arr, $x)
add(arr, "]")
doAssert arr == "[2, 3, 5, 7, 11]"
```

---

### Форматирование чисел с плавающей точкой

#### `formatBiggestFloat(f: BiggestFloat, format: FloatFormatMode = ffDefault, precision: range[-1..32] = 16, decimalSep = '.'): string`
Преобразует `f` в строку согласно `format`:
- `ffDecimal` — `precision` означает количество цифр **после десятичной точки**.
- `ffScientific` — `precision` означает максимальное количество **значащих цифр**.
- `ffDefault` — используется наиболее короткое представление, допускающее обратное преобразование.

Значение по умолчанию `precision = 16` — это максимальное количество значащих цифр после точки для `BiggestFloat`. Передайте `precision = -1`, чтобы реализация выбрала "красивое" форматирование сама. `decimalSep` определяет, какой символ используется как десятичный разделитель (полезно для локалей, где используется `,`).
```nim
let x = 123.456
doAssert x.formatBiggestFloat() == "123.4560000000000"
doAssert x.formatBiggestFloat(ffDecimal, 4) == "123.4560"
doAssert x.formatBiggestFloat(ffScientific, 2) == "1.23e+02"
```

#### `formatFloat(f: float, format: FloatFormatMode = ffDefault, precision: range[-1..32] = 16, decimalSep = '.'): string`
То же, что `formatBiggestFloat`, но для обычного типа `float` (и значение точности по умолчанию относится к значащим цифрам `float`). На практике просто делегирует вызов `formatBiggestFloat`.
```nim
let x = 123.456
doAssert x.formatFloat() == "123.4560000000000"
doAssert x.formatFloat(ffDecimal, 4) == "123.4560"
doAssert x.formatFloat(ffScientific, 2) == "1.23e+02"
```

#### `trimZeros(x: var string, decimalSep = '.')`
Удаляет конечные нули из отформатированной строки числа с плавающей точкой `x` **на месте**. Действует только на дробную часть (после `decimalSep`); если присутствует экспонента (`e`), нули обрезаются только до маркера экспоненты. Если после удаления нулей после `decimalSep` ничего не останется, сам разделитель также удаляется.
```nim
var x = "123.456000000"
x.trimZeros()
doAssert x == "123.456"
```

---

### Форматирование размеров и инженерная запись

#### `formatSize(bytes: int64, decimalSep = '.', prefix = bpIEC, includeSpace = false): string`
Форматирует `bytes` как человекочитаемый размер с суффиксом двоичной единицы измерения, округляя до 3 значащих десятичных цифр (с удалением конечных нулей). По умолчанию используются двоичные приставки IEC/ISO (`Ki`, `Mi`, `Gi`, … — степени 1024 с суффиксом "i", указывающим на двоичность, а не десятичность). Передайте `prefix = bpColloquial`, чтобы вместо этого использовать привычные названия (`k`, `M`, `G`, …) для тех же величин с базой 1024. Установите `includeSpace = true`, чтобы вставить пробел между числом и единицей измерения, как рекомендует стандарт SI.
```nim
doAssert formatSize((1'i64 shl 31) + (300'i64 shl 20)) == "2.293GiB"
doAssert formatSize((2.234*1024*1024).int) == "2.233MiB"
doAssert formatSize(4096, includeSpace = true) == "4 KiB"
doAssert formatSize(4096, prefix = bpColloquial, includeSpace = true) == "4 kB"
doAssert formatSize(4096) == "4KiB"
doAssert formatSize(5_378_934, prefix = bpColloquial, decimalSep = ',') == "5,129MB"
```

#### `formatEng(f: BiggestFloat, precision: range[0..32] = 10, trim: bool = true, siPrefix: bool = false, unit: string = "", decimalSep = '.', useUnitSpace = false): string`
Форматирует `f`, используя **инженерную запись**: экспонента (если есть) всегда кратна 3, а мантисса остаётся в диапазоне `-1000.0 < мантисса < 1000.0`. Значения с `-1000.0 < f < 1000.0` отображаются вообще без экспоненты.

- `precision` — количество цифр после десятичной точки (или, если `trim = true`, *максимальное* количество отображаемых цифр).
- `trim` (по умолчанию `true`) — удаляет конечные нули дробной части, давая наиболее короткое точное представление (до `precision` цифр). Если `false`, всегда отображается ровно `precision` цифр.
- `siPrefix` (по умолчанию `false`) — если `true`, заменяет экспоненту `eN` соответствующей буквой SI-приставки (`k`, `M`, `G`, `T`, `P`, `E` для положительных экспонент; `m`, `u`, `n`, `p`, `f`, `a` для отрицательных — обратите внимание, вместо `μ` используется `u`, согласно ISO 2955). Числа, экспонента которых выходит за диапазон, покрываемый этими приставками (примерно от `1e-18` до `1e18`), всё равно используют `eN`, даже если `siPrefix` истинно.
- `unit` — строка единицы измерения, добавляемая к результату.
- `useUnitSpace` — если `true`, перед единицей измерения вставляется пробел даже при отсутствии SI-приставки (точное расположение пробела зависит от наличия экспоненты/приставки).
- `decimalSep` — символ, используемый как десятичный разделитель.

```nim
formatEng(0, 2, trim=false) == "0.00"
formatEng(0, 2) == "0"
formatEng(0.053, 0) == "53e-3"
formatEng(52731234, 2) == "52.73e6"
formatEng(-52731234, 2) == "-52.73e6"

formatEng(4100, siPrefix=true, unit="V") == "4.1 kV"
formatEng(4.1, siPrefix=true, unit="V") == "4.1 V"
formatEng(4.1, siPrefix=true) == "4.1"          # без пробела, если нет единицы измерения
formatEng(4100, siPrefix=true) == "4.1 k"
formatEng(4.1, siPrefix=true, unit="") == "4.1 "      # пробел при unit=""
formatEng(4100, siPrefix=true, unit="") == "4.1 k"
formatEng(4100) == "4.1e3"
formatEng(4100, unit="V") == "4.1e3 V"
formatEng(4100, unit="", useUnitSpace=true) == "4.1e3 " # пробел при useUnitSpace=true
```

---

### Интерполяция строк (`%` и `format`)

#### `addf(s: var string, formatstr: string, a: varargs[string, `$`])`
Эффективный, выполняемый на месте эквивалент `add(s, formatstr % a)` — добавляет результат интерполяции `formatstr` непосредственно к `s` без промежуточного выделения памяти под результат `%`. Эта процедура реализует логику подстановки, описанную ниже; `%` и `format` являются тонкими обёртками над ней.

#### `% (formatstr: string, a: openArray[string]): string`
**Оператор подстановки**. Сканирует `formatstr` на наличие плейсхолдеров и заменяет их значениями из `a`. Поддерживаемые формы плейсхолдеров:

- **`$N`** (и `$-N`) — позиционная ссылка. `$1` соответствует `a[0]`, `$2` — `a[1]` и т.д. (индексация с 1). `$-1` соответствует `a[a.high]` (считая с конца).
- **`$#`** — обозначает "следующий" позиционный аргумент, при каждом использовании увеличивая внутренний счётчик. Позволяет не нумеровать вручную последовательные плейсхолдеры.
- **`$$`** — создаёт буквальный символ `$` в выводе.
- **`${...}`** — форма с фигурными скобками. Если содержимое между скобками чисто числовое, оно ведёт себя как `$N`/`$-N` (позиционная ссылка). В противном случае содержимое рассматривается как *именованный* плейсхолдер (см. ниже).
- **`$identifier`** (последовательность символов `[A-Za-z0-9_\128-\255]` после `$`) — именованный плейсхолдер. В этом случае `a` интерпретируется как плоский список пар **ключ, значение**: элементы с чётными индексами (0, 2, 4, …) — ключи, с нечётными (1, 3, 5, …) — соответствующие значения. Имя плейсхолдера сравнивается с ключами с помощью `cmpIgnoreStyle` (без учёта регистра и подчёркиваний).

Если `formatstr` некорректна (например, ссылается на индекс вне диапазона или на несуществующий именованный плейсхолдер), выбрасывается `ValueError`.

```nim
"$1 eats $2." % ["The cat", "fish"]
# "The cat eats fish."

"$# eats $#." % ["The cat", "fish"]
# "The cat eats fish."

"$animal eats $food." % ["animal", "The cat", "food", "fish"]
# "The cat eats fish."
```

#### `% (formatstr, a: string): string`
Сокращение для `formatstr % [a]` — интерполирует *одно* строковое значение как `$1`/`$#`.

#### `format(formatstr: string, a: varargs[string, `$`]): string`
То же, что `formatstr % a`, но дополнительно поддерживает **автоматическое преобразование в строку**: аргументы, которые ещё не являются строками, преобразуются через `$` перед подстановкой. Используйте эту функцию при передаче нестроковых значений (чисел, объектов с процедурой `$` и т.п.) напрямую.

---

## Рекомендации: какую функцию когда использовать

- **Разбиение входного текста:**
  - Используйте `splitWhitespace`, когда нужно токенизировать произвольный текст и не важны последовательности из нескольких пробелов/табуляций — ведущие и завершающие пробелы также отбрасываются.
  - Используйте `split(s, ",")` (или другой фиксированный разделитель — строку/символ) для структурированных данных, например в формате CSV, где важно сохранять пустые поля между соседними разделителями.
  - Используйте `split(s, seps: set[char])`, когда несколько различных *односимвольных* разделителей должны считаться равнозначными (например, `{',', ';'}`).
  - Используйте `rsplit` с `maxsplit = 1`, когда нужно отделить только **последний** компонент (например, расширение файла, последний сегмент пути), сохранив остальную часть строки целой.
  - Используйте `splitLines` для построчной обработки; если нужно только количество строк, предпочтите `countLines` (она быстрее и не требует выделения памяти).
  - Используйте `tokenize`, когда нужно **точно восстановить** исходную строку (включая все последовательности разделителей), например для форматтера или генератора текста.

- **Объединение строк обратно:**
  - Используйте `join` как для `seq[string]` (прямая перегрузка), так и для последовательностей других типов (универсальная `join[T: not string]`, преобразующая значения через `$`).

- **Заполнение / выравнивание текста для отображения:**
  - Используйте `align` для выравнивания по правому краю (например, числовые столбцы), `alignLeft` — по левому краю (например, текстовые столбцы), `center` — для центрирования заголовков/названий.
  - Используйте `spaces`, когда требуется "сырое" заполнение пробелами без вычисления целевой ширины столбца, как это делают `align`/`alignLeft`.
  - Используйте `indent`/`unindent`/`dedent` для изменения отступов многострочного текста на уровне блока; предпочитайте `dedent` перед `unindent`, когда нужно сохранить *относительные* отступы между строками (например, вложенные блоки кода), а `unindent` — когда нужно убрать фиксированное количество/все ведущие пробелы у каждой строки.

- **Поиск и замена:**
  - Используйте `find`/`rfind`, чтобы найти одно вхождение (первое или последнее) и получить его индекс.
  - Используйте `contains` (или операторы `in`/`notin`), когда нужна только проверка "да/нет" на вхождение, а не позиция.
  - Используйте `count`, когда нужно количество вхождений, а не их позиции.
  - Используйте `replace` для простых операций "заменить все"; используйте `replaceWord`, если совпадения должны быть целыми словами; используйте `multiReplace` (строковый или посимвольный вариант), когда нужно применить **несколько** замен за один эффективный проход — особенно для очистки входных данных от нескольких запрещённых подстрок/символов сразу.
  - Если предстоит многократно искать **одну и ту же** подстроку в разных строках, заранее вычислите `SkipTable` через `initSkipTable` и повторно используйте её через `find(table, s, sub, ...)` для повышения производительности.

- **Обрезка и очистка:**
  - Используйте `strip` для общего удаления ведущих/конечных символов (по умолчанию — пробельных).
  - Используйте `stripLineEnd` специально для удаления одного завершающего перевода строки (например, после чтения строки из процесса или файла).
  - Используйте `removePrefix`/`removeSuffix` для целевого удаления на месте известного префикса/суффикса (строки, одного символа или множества символов).

- **Работа с регистром и идентификаторами:**
  - Используйте `toLowerAscii`/`toUpperAscii`/`capitalizeAscii` для изменения регистра только в ASCII; для полной поддержки Unicode используйте реэкспортированные `unicode.toLower`/`unicode.toUpper`.
  - Используйте `cmpIgnoreCase` для простого сравнения без учёта регистра.
  - Используйте `cmpIgnoreStyle`/`normalize` для "стиль-независимого" сравнения строк, ориентированных на человека (конфигурационных и т.п.) — без учёта регистра и подчёркиваний — но **никогда** для сравнения идентификаторов Nim; для этого используйте `nimIdentNormalize` (нормализация в стиле компилятора) или `macros.eqIdent`.
  - Используйте `validIdentifier`, чтобы проверить, можно ли использовать строку как идентификатор синтаксически, например перед генерацией кода.

- **Числа и строки:**
  - Используйте `parseInt`/`parseFloat`/`parseBool`/`parseEnum` и аналогичные функции для разбора текста, введённого пользователем или взятого из конфигурации, в типизированные значения — при некорректном вводе они выбрасывают `ValueError`, поэтому оборачивайте вызовы в `try/except` (или предварительно проверяйте данные), если ввод может быть недостоверным.
  - Используйте `parseBinInt`/`parseOctInt`/`parseHexInt` (возвращающие `int`) для простых случаев, либо `fromBin`/`fromOct`/`fromHex` (универсальные по целевому целочисленному типу), когда нужна конкретная разрядность (например, `uint8`, `int16`) — учтите, что они **не** проверяют переполнение.
  - Используйте `toBin`/`toOct`/`toHex`/`toOctal`/`intToStr` для представления чисел в текстовом виде фиксированной ширины (например, для hex-дампов, визуализации в двоичном виде, идентификаторов с дополнением нулями).
  - Используйте `insertSep` для добавления разделителей разрядов (тысяч) при отображении больших чисел в удобочитаемом виде.
  - Используйте `parseHexStr`/`toHex(s: string)` для преобразования между необработанными байтами и их hex-текстовым представлением (например, для хешей, отладки бинарных данных).

- **Числа с плавающей точкой / размеры:**
  - Используйте `formatFloat`/`formatBiggestFloat` для общего форматирования чисел с явным контролем точности/нотации; сочетайте с `trimZeros`, если нужно убрать конечные нули из результата с фиксированной точностью.
  - Используйте `formatSize` для отображения размеров файлов, объёма используемой памяти, скоростей передачи и т.п. в удобочитаемых двоичных единицах.
  - Используйте `formatEng` в научных/инженерных контекстах, где ожидаются SI-приставки (k, M, G, m, µ, n, …) или экспоненты, кратные трём (например, электроника, измерительные данные).
  - Для более сложного или локализованного форматирования также рассмотрите модуль `strformat` (интерполяция строк со спецификаторами формата).

- **Построение строк с подстановкой:**
  - Используйте `%`/`format` для простого шаблонного текста, где плейсхолдеры (`$1`, `$#`, `$name`) заполняются из списка строк/значений; используйте `addf`, если результат добавляется непосредственно в существующий строковый буфер, чтобы избежать дополнительных выделений памяти.
  - Для более сложной или типобезопасной интерполяции предпочтительнее синтаксис `&"..."` модуля `strformat`.
