# Справочник модуля `pegs` (Nim)

> Модуль стандартной библиотеки Nim для работы с **PEG** (Parsing Expression Grammars).  
> Поддерживает Unicode, супероператоры для ускорения, встроенный DSL и событийный парсинг.

---

## Содержание

1. [Обзор и синтаксис PEG](#обзор-и-синтаксис-peg)
2. [Типы](#типы)
3. [Создание PEG-объектов из строки](#создание-peg-объектов-из-строки)
4. [Конструкторы PEG (DSL)](#конструкторы-peg-dsl)
   - [Терминалы](#терминалы)
   - [Операторы (комбинаторы)](#операторы-комбинаторы)
   - [Предикаты](#предикаты)
   - [Захват (capture) и обратные ссылки](#захват-capture-и-обратные-ссылки)
   - [Якоря и специальные символы](#якоря-и-специальные-символы)
   - [Шаблоны-сокращения](#шаблоны-сокращения)
   - [Нетерминалы](#нетерминалы)
5. [Аксессоры `Peg`](#аксессоры-peg)
6. [Аксессоры `NonTerminal`](#аксессоры-nonterminal)
7. [Тип `Captures`](#тип-captures)
8. [Поиск и сопоставление](#поиск-и-сопоставление)
9. [Замена](#замена)
10. [Разбивка строки](#разбивка-строки)
11. [Низкоуровневое сопоставление](#низкоуровневое-сопоставление)
12. [Событийный парсер](#событийный-парсер)
13. [Утилиты](#утилиты)
14. [Быстрая шпаргалка](#быстрая-шпаргалка)

---

## Обзор и синтаксис PEG

PEG (Parsing Expression Grammar) — формальная система описания языков, альтернатива регулярным выражениям и контекстно-свободным грамматикам. В отличие от regexp, PEG-сопоставление детерминировано (нет неоднозначности) и не использует запоминание (мемоизацию) — вместо этого применяются супероператоры для ускорения.

### Синтаксис PEG-строк (для `peg` / `parsePeg`)

| Элемент | Синтаксис | Описание |
|---------|-----------|----------|
| Литерал | `'hello'` | Точное совпадение |
| Без регистра | `i'hello'` | Без учёта регистра |
| Без стиля | `y'hello'` | Без учёта стиля (`_`, `-`) |
| Любой символ | `.` | Любой байт |
| Любой руна | `_` | Любой Unicode-символ |
| Набор символов | `[a-zA-Z]` | Диапазоны/символы |
| Отрицание набора | `[^0-9]` | Исключение символов |
| Группа | `(a b c)` | Последовательность |
| Выбор | `a / b` | Упорядоченный выбор |
| Опционально | `a?` | 0 или 1 раз |
| Жадное повтор. | `a*` | 0 и более раз |
| Обязат. повтор. | `a+` | 1 и более раз |
| И-предикат | `&a` | Проверка без потребления |
| НЕ-предикат | `!a` | Отрицание без потребления |
| Поиск | `@a` | Пропустить до первого совпадения |
| Захват | `{a}` | Сохранить совпавшую подстроку |
| Обратная ссылка | `$1` | Ссылка на 1-й захват |
| Нетерминал | `Name` | Именованное правило |
| Правило | `Name <- a b c` | Определение нетерминала |
| Начало строки | `^` | Якорь начала |
| Конец строки | `$` | Якорь конца (= `!.`) |
| Перенос строки | `\n` | CR, LF или CR+LF |
| Unicode буква | `\letter` | Любая буква Unicode |
| Unicode строч. | `\lower` | Строчная буква Unicode |
| Unicode заглав. | `\upper` | Заглавная буква Unicode |
| Unicode title | `\title` | Title-символ Unicode |
| Unicode пробел | `\white` | Пробельный символ Unicode |

**Экранирование:**

| Последовательность | Значение |
|--------------------|----------|
| `\t` | Табуляция |
| `\n` | Новая строка |
| `\r` / `\c` | Возврат каретки |
| `\\` | Обратный слеш |
| `\xHH` | Символ с кодом HH (hex) |

**Константа:** `MaxSubpatterns* = 20` — максимальное число захватываемых подшаблонов.

---

## Типы

```nim
type
  PegKind* = enum
    pkEmpty, pkAny, pkAnyRune, pkNewLine,
    pkLetter, pkLower, pkUpper, pkTitle, pkWhitespace,
    pkTerminal, pkTerminalIgnoreCase, pkTerminalIgnoreStyle,
    pkChar, pkCharChoice, pkNonTerminal,
    pkSequence, pkOrderedChoice,
    pkGreedyRep, pkGreedyRepChar, pkGreedyRepSet, pkGreedyAny,
    pkOption, pkAndPredicate, pkNotPredicate,
    pkCapture, pkBackRef, pkBackRefIgnoreCase, pkBackRefIgnoreStyle,
    pkSearch, pkCapturedSearch,
    pkRule, pkList, pkStartAnchor

  NonTerminalFlag* = enum
    ntDeclared, ntUsed

  Peg* = object          ## Узел PEG-дерева
  NonTerminal* = ref object  ## Именованный нетерминальный символ

  Captures* = object     ## Результат захватов (captures)
```

---

## Создание PEG-объектов из строки

### `peg`

```nim
func peg*(pattern: string): Peg
```

Создаёт `Peg` из строки-шаблона. Задуман для использования в качестве **строкового модификатора** (raw string modifier):

```nim
import std/pegs

let p = peg"{\ident} \s* '=' \s* {.*}"
if "name = Alice" =~ p:
  echo matches[0]  # "name"
  echo matches[1]  # "Alice"
```

---

### `parsePeg`

```nim
func parsePeg*(pattern: string, filename = "pattern",
               line = 1, col = 0): Peg
```

Расширенный вариант `peg`: принимает имя файла и позицию строки/столбца для точных сообщений об ошибках. Отслеживает строки и столбцы внутри `pattern`.

```nim
import std/pegs

let p = parsePeg("{\\ident}", "config.peg", line = 10, col = 3)
```

---

### `escapePeg`

```nim
func escapePeg*(s: string): string
```

Экранирует строку `s` так, чтобы она воспринималась как **буквальный** PEG-шаблон (все специальные символы экранируются).

```nim
import std/pegs

let raw = "hello (world)"
let escaped = escapePeg(raw)  # -> "'hello (world)'"
let p = peg(escaped)
echo "hello (world)" =~ p     # true
```

---

### `$` (конвертация в строку)

```nim
func `$`*(r: Peg): string
```

Конвертирует `Peg` обратно в его строковое представление. Полезно для отладки.

```nim
import std/pegs

let p = sequence(term("ab"), *charSet({'0'..'9'}))
echo $p  # выведет строковое PEG-представление
```

---

## Конструкторы PEG (DSL)

### Терминалы

#### `term(t: string): Peg`

```nim
func term*(t: string): Peg
```

Создаёт PEG для точного совпадения со строкой `t`. Если строка состоит из одного символа — создаёт более эффективный `pkChar`.

```nim
let p = term("hello")
echo "hello world".find(p)  # 0
```

---

#### `term(t: char): Peg`

```nim
func term*(t: char): Peg
```

Создаёт PEG для точного совпадения с символом `t`. Символ `'\0'` не допускается.

```nim
let p = term('!')
```

---

#### `termIgnoreCase(t: string): Peg`

```nim
func termIgnoreCase*(t: string): Peg
```

Создаёт PEG для совпадения со строкой `t` без учёта регистра.

```nim
let p = termIgnoreCase("hello")
echo "HELLO".match(p)  # true
echo "Hello".match(p)  # true
```

---

#### `termIgnoreStyle(t: string): Peg`

```nim
func termIgnoreStyle*(t: string): Peg
```

Создаёт PEG для совпадения без учёта стиля: символы `_` и `-` игнорируются, а регистр не учитывается.

```nim
let p = termIgnoreStyle("helloWorld")
echo "hello_world".match(p)  # true
echo "Hello-World".match(p)  # true
```

---

#### `charSet(s: set[char]): Peg`

```nim
func charSet*(s: set[char]): Peg
```

Создаёт PEG, совпадающий с одним символом из набора `s`. Символ `'\0'` не допускается.

```nim
let p = charSet({'a'..'z', 'A'..'Z'})
echo "hello".match(p)    # true (совпадает первый символ)
```

---

### Операторы (комбинаторы)

#### `/` — упорядоченный выбор

```nim
func `/`*(a: varargs[Peg]): Peg
```

Создаёт **упорядоченный выбор**: пробует альтернативы слева направо, возвращает первое совпадение.

```nim
let p = term("cat") / term("dog") / term("bird")
echo "dog is here".find(p)  # 0, совпало "dog"
```

---

#### `sequence` — последовательность

```nim
func sequence*(a: varargs[Peg]): Peg
```

Создаёт **последовательность**: все PEG должны совпасть подряд.

```nim
let p = sequence(term("hello"), term(" "), term("world"))
echo "hello world".match(p)  # true
```

---

#### `?` — опциональный

```nim
func `?`*(a: Peg): Peg
```

Совпадает с `a` **0 или 1 раз**. Если `a` уже жадный (`*`, `?`) — возвращает `a` без изменений.

```nim
let p = sequence(term("color"), ?term("u"), term("r"))
echo "color".match(p)   # false (нет 'r')
echo "colour".match(p)  # true
echo "color".find(peg"'color' 'u'? 'r'")  # 0
```

---

#### `*` — жадное повторение (0 и более)

```nim
func `*`*(a: Peg): Peg
```

Совпадает с `a` **0 и более раз** (жадно).

```nim
let p = *charSet({'0'..'9'})
echo "12345abc".matchLen(p)  # 5
```

---

#### `+` — жадное положительное повторение (1 и более)

```nim
func `+`*(a: Peg): Peg
```

Совпадает с `a` **1 и более раз**. Эквивалентно `sequence(a, *a)`.

```nim
let digits = +charSet({'0'..'9'})
echo "42 rest".matchLen(digits)  # 2
```

---

#### `!*` — поиск (search)

```nim
func `!*`*(a: Peg): Peg
```

**Поиск**: пропускает символы до первого совпадения с `a`. Соответствует оператору `@a` в строковом синтаксисе.

```nim
let p = !*term("end")
echo "some text end".matchLen(p)  # длина всего до "end"
```

---

#### `!*\` — поиск с захватом

```nim
func `!*\`*(a: Peg): Peg
```

**Поиск с захватом**: как `!*`, но захватывает пропущенный текст. Соответствует `{@}a` в строковом синтаксисе.

```nim
# Захватывает всё до "end"
let p = !*\term("end")
```

---

### Предикаты

#### `&` — И-предикат (lookahead)

```nim
func `&`*(a: Peg): Peg
```

Проверяет, что следующий фрагмент совпадает с `a`, **не потребляя** входные символы.

```nim
# Совпасть с "a", только если за ним идёт "b"
let p = sequence(&term("ab"), term("a"))
echo "ab".match(p)  # true, потреблено только "a"
echo "ac".match(p)  # false
```

---

#### `!` — НЕ-предикат (negative lookahead)

```nim
func `!`*(a: Peg): Peg
```

Проверяет, что следующий фрагмент **не** совпадает с `a`, не потребляя символов.

```nim
# Совпасть с любым символом, кроме цифры
let p = sequence(!charSet({'0'..'9'}), any())
echo "a5".matchLen(p)  # 1 (потреблено только 'a')
```

---

### Захват (capture) и обратные ссылки

#### `capture`

```nim
func capture*(a: Peg = Peg(kind: pkEmpty)): Peg
```

Создаёт **захватывающую группу**: сохраняет совпавший фрагмент в массив captures. Без аргумента создаёт пустой захват.

```nim
import std/pegs

var caps: array[0..19, string]
let p = sequence(capture(+charSet({'a'..'z'})), term("="),
                 capture(+charSet({'a'..'z'})))
discard "key=val".match(p, caps)
echo caps[0]  # "key"
echo caps[1]  # "val"
```

---

#### `backref`

```nim
func backref*(index: range[1..MaxSubpatterns],
              reverse: bool = false): Peg
```

Создаёт **обратную ссылку** на `index`-й захват (нумерация с 1). `reverse = true` — нумерация с конца.

```nim
# Найти удвоенное слово
let p = sequence(capture(+charSet({'a'..'z'})), term(" "), backref(1))
echo "hello hello".match(p)  # true
echo "hello world".match(p)  # false
```

---

#### `backrefIgnoreCase`

```nim
func backrefIgnoreCase*(index: range[1..MaxSubpatterns],
                        reverse: bool = false): Peg
```

Обратная ссылка без учёта регистра.

```nim
let p = sequence(capture(+charSet({'a'..'z'})), term(" "), backrefIgnoreCase(1))
echo "hello HELLO".match(p)  # true
```

---

#### `backrefIgnoreStyle`

```nim
func backrefIgnoreStyle*(index: range[1..MaxSubpatterns],
                         reverse: bool = false): Peg
```

Обратная ссылка без учёта стиля.

---

### Якоря и специальные символы

#### `any` — любой символ

```nim
func any*: Peg
```

Совпадает с **любым одним байтом** (аналог `.` в regexp).

```nim
let p = *any()   # поглощает всю строку
```

---

#### `anyRune` — любой Unicode-символ

```nim
func anyRune*: Peg
```

Совпадает с **любым Unicode-символом** (рунаом), независимо от числа байт в UTF-8.

```nim
let p = *anyRune()
```

---

#### `newLine` — перенос строки

```nim
func newLine*: Peg
```

Совпадает с `\r\n`, `\n` или `\r`.

```nim
let lines = split("a\nb\r\nc", newLine())
```

---

#### `startAnchor` — начало строки

```nim
func startAnchor*: Peg
```

Совпадает только в самом начале входной строки (аналог `^`).

```nim
let p = sequence(startAnchor(), term("start"))
echo "start here".match(p)   # true
echo "not start".match(p)    # false
```

---

#### `endAnchor` — конец строки

```nim
func endAnchor*: Peg
```

Совпадает только в конце строки (эквивалентно `!any()`).

```nim
let p = sequence(*any(), endAnchor())
echo "anything".match(p)  # true
```

---

#### `unicodeLetter`, `unicodeLower`, `unicodeUpper`, `unicodeTitle`, `unicodeWhitespace`

```nim
func unicodeLetter*: Peg    # \letter — любая буква Unicode
func unicodeLower*: Peg     # \lower  — строчная буква Unicode
func unicodeUpper*: Peg     # \upper  — заглавная буква Unicode
func unicodeTitle*: Peg     # \title  — Title-символ Unicode
func unicodeWhitespace*: Peg # \white — пробельный символ Unicode
```

```nim
let p = +unicodeLetter()
echo "Привет123".matchLen(p)  # длина "Привет" в байтах
```

---

### Шаблоны-сокращения

Эти шаблоны раскрываются в вызовы `charSet`:

| Шаблон | Эквивалент | Описание |
|--------|------------|----------|
| `letters` | `charSet({'A'..'Z', 'a'..'z'})` | Латинские буквы |
| `digits` | `charSet({'0'..'9'})` | Цифры |
| `whitespace` | `charSet({' ', '\9'..'\13'})` | Пробельные символы ASCII |
| `identChars` | `charSet({'a'..'z','A'..'Z','0'..'9','_'})` | Символы идентификатора |
| `identStartChars` | `charSet({'a'..'z','A'..'Z','_'})` | Начало идентификатора |
| `ident` | `sequence(identStartChars, *identChars)` | Стандартный идентификатор |
| `natural` | `+digits` | Натуральное число (`\d+`) |

```nim
import std/pegs

let p = sequence(ident, *sequence(whitespace, ident))
echo "foo bar baz".match(p)  # true
```

---

### Нетерминалы

#### `newNonTerminal`

```nim
func newNonTerminal*(name: string, line, column: int): NonTerminal
```

Создаёт нетерминальный символ для использования в грамматиках с именованными правилами.

---

#### `nonterminal`

```nim
func nonterminal*(n: NonTerminal): Peg
```

Создаёт `Peg`-узел, ссылающийся на нетерминал `n`. Если символ объявлен и достаточно мал — правило инлайнируется для оптимизации.

```nim
import std/pegs

# Грамматика лучше строится через parsePeg/peg:
let grammar = peg"""
  Expr    <- Term ('+' Term)*
  Term    <- Factor ('*' Factor)*
  Factor  <- [0-9]+ / '(' Expr ')'
"""
```

---

## Аксессоры `Peg`

Функции для получения внутренних данных `Peg`-узла. Применимы только к соответствующим вариантам (`kind`).

| Функция | Возвращает | Применимо к |
|---------|------------|-------------|
| `kind(p)` | `PegKind` | Любой `Peg` |
| `term(p)` | `string` | `pkTerminal`, `pkTerminalIgnoreCase`, `pkTerminalIgnoreStyle` |
| `ch(p)` | `char` | `pkChar`, `pkGreedyRepChar` |
| `charChoice(p)` | `ref set[char]` | `pkCharChoice`, `pkGreedyRepSet` |
| `nt(p)` | `NonTerminal` | `pkNonTerminal` |
| `index(p)` | `int` | `pkBackRef`..`pkBackRefIgnoreStyle` |

```nim
let p = term("hello")
echo p.kind   # pkTerminal
echo p.term   # "hello"
```

---

#### `items(p: Peg): Peg` (итератор)

```nim
iterator items*(p: Peg): Peg
```

Перебирает дочерние узлы `Peg`-дерева.

```nim
let p = sequence(term("a"), term("b"), term("c"))
for child in p:
  echo $child
```

---

#### `pairs(p: Peg): (int, Peg)` (итератор)

```nim
iterator pairs*(p: Peg): (int, Peg)
```

Перебирает дочерние узлы с их индексами.

```nim
for i, child in peg"('a' / 'b' / 'c')":
  echo i, ": ", $child
```

---

## Аксессоры `NonTerminal`

| Функция | Возвращает | Описание |
|---------|------------|----------|
| `name(nt)` | `string` | Имя нетерминала |
| `line(nt)` | `int` | Строка объявления |
| `col(nt)` | `int` | Столбец объявления |
| `flags(nt)` | `set[NonTerminalFlag]` | Флаги (`ntDeclared`, `ntUsed`) |
| `rule(nt)` | `Peg` | PEG-правило символа |

---

## Тип `Captures`

```nim
type Captures* = object
```

Хранит результаты захватов после `rawMatch`. Содержит до `MaxSubpatterns` (20) позиций.

#### `bounds(c: Captures, i: range[0..MaxSubpatterns-1]): tuple[first, last: int]`

```nim
func bounds*(c: Captures, i: range[0..MaxSubpatterns-1]): tuple[first, last: int]
```

Возвращает позиции `[first..last]` `i`-го захвата (0-индексированный).

```nim
import std/pegs

var c: Captures
let s = "hello world"
let p = peg"{\\ident} \\s+ {\\ident}"
discard rawMatch(s, p, 0, c)
echo c.bounds(0)  # (0, 4)  — "hello"
echo c.bounds(1)  # (6, 10) — "world"
```

---

## Поиск и сопоставление

### `=~` — оператор сопоставления

```nim
template `=~`*(s: string, pattern: Peg): bool
```

Удобный оператор: вызывает `match` с **неявно объявленным** массивом `matches`, доступным в блоке `if`. Каждая ветка `elif` получает свой независимый `matches`.

```nim
import std/pegs

let line = "key = value"
if line =~ peg"{\ident} \s* '=' \s* {.*}":
  echo "Ключ: ", matches[0]    # "key"
  echo "Значение: ", matches[1] # "value"
elif line =~ peg"{'#'.*}":
  echo "Комментарий: ", matches[0]
else:
  echo "Синтаксическая ошибка"
```

---

### `match` — совпадение от начала

```nim
func match*(s: string, pattern: Peg,
            matches: var openArray[string], start = 0): bool
func match*(s: string, pattern: Peg, start = 0): bool
```

Возвращает `true`, если `s[start..]` совпадает с `pattern`. В варианте с `matches` — записывает захваченные подстроки.

```nim
import std/pegs

var caps = newSeq[string](20)
if "2024-01-15".match(peg"{\d+} '-' {\d+} '-' {\d+}", caps):
  echo caps[0]  # "2024"
  echo caps[1]  # "01"
  echo caps[2]  # "15"
```

---

### `matchLen` — длина совпадения

```nim
func matchLen*(s: string, pattern: Peg,
               matches: var openArray[string], start = 0): int
func matchLen*(s: string, pattern: Peg, start = 0): int
```

Как `match`, но возвращает **длину совпадения** (или -1). Длина 0 возможна. После совпадения суффикс строки может остаться непроверенным.

```nim
import std/pegs

let n = "123abc".matchLen(peg"\d+")
echo n  # 3
```

---

### `find` — поиск в строке

```nim
func find*(s: string, pattern: Peg,
           matches: var openArray[string], start = 0): int
func find*(s: string, pattern: Peg, start = 0): int
```

Возвращает **начальную позицию** первого совпадения в `s` (начиная с `start`), или -1. В варианте с `matches` — заполняет захваты.

```nim
import std/pegs

echo "foo 42 bar".find(peg"\d+")  # 4
```

---

### `findBounds` — границы совпадения

```nim
func findBounds*(s: string, pattern: Peg,
                 matches: var openArray[string],
                 start = 0): tuple[first, last: int]
```

Возвращает кортеж `(first, last)` первого совпадения с `pattern`, или `(-1, 0)`. Заполняет `matches`.

```nim
import std/pegs

let (a, b) = "text: hello world".findBounds(peg"{\ident+}", @["", ""])
# нет нужды в @[], передаём var openArray
var caps = newSeq[string](20)
let bounds = "text: hello world".findBounds(peg"{\w+}", caps)
echo bounds  # (first: 6, last: 10) — "hello"
```

---

### `findAll` — все совпадения

```nim
iterator findAll*(s: string, pattern: Peg, start = 0): string
func findAll*(s: string, pattern: Peg, start = 0): seq[string]
```

Перебирает / возвращает все совпадающие **подстроки**. Если совпадений нет — пустой seq.

```nim
import std/pegs

for word in "one two three".findAll(peg"\ident"):
  echo word  # "one", "two", "three"

let nums = "a1 b22 c333".findAll(peg"\d+")
echo nums  # @["1", "22", "333"]
```

---

### `contains` — проверка наличия

```nim
func contains*(s: string, pattern: Peg, start = 0): bool
func contains*(s: string, pattern: Peg,
               matches: var openArray[string], start = 0): bool
```

Эквивалентно `find(s, pattern, start) >= 0`. Возвращает `true`, если шаблон найден.

```nim
import std/pegs

echo "hello123".contains(peg"\d+")  # true
echo "abcdef".contains(peg"\d+")    # false
```

---

### `startsWith` — совпадение в начале

```nim
func startsWith*(s: string, prefix: Peg, start = 0): bool
```

Возвращает `true`, если `s` начинается с шаблона `prefix` (начиная с позиции `start`).

```nim
import std/pegs

echo "hello world".startsWith(peg"\ident")  # true
echo "123 world".startsWith(peg"\ident")    # false
```

---

### `endsWith` — совпадение в конце

```nim
func endsWith*(s: string, suffix: Peg, start = 0): bool
```

Возвращает `true`, если `s` заканчивается шаблоном `suffix`.

```nim
import std/pegs

echo "file.nim".endsWith(peg"'.' \ident")  # true
echo "file.txt".endsWith(peg"'.nim'")      # false
```

---

## Замена

### `replace` — замена без захватов

```nim
func replace*(s: string, sub: Peg, by = ""): string
```

Заменяет все вхождения `sub` в `s` строкой `by`. Захваты в `by` недоступны.

```nim
import std/pegs

echo "hello   world".replace(peg"\s+", " ")   # "hello world"
echo "aaa123bbb".replace(peg"\d+", "NUM")      # "aaaNU​Mbbb"
```

---

### `replacef` — замена с форматированием захватов

```nim
func replacef*(s: string, sub: Peg, by: string): string
```

Заменяет вхождения `sub`, используя `by` как форматную строку. Захваты доступны через `$1`, `$2`, ... и `$#` (аналог `strutils.%`).

```nim
import std/pegs

let result = "var1=key; var2=key2".replacef(
  peg"{\ident} '=' {\ident}",
  "$1<-$2$2"
)
echo result  # "var1<-keykey; var2<-key2key2"
```

---

### `replace` с колбэком

```nim
func replace*(s: string, sub: Peg,
              cb: proc(match, cnt: int,
                       caps: openArray[string]): string): string
```

Заменяет вхождения, вызывая функцию `cb` для каждого совпадения:
- `match` — индекс совпадения (с 0)
- `cnt` — число захватов
- `caps` — массив захваченных подстрок

```nim
import std/pegs

func handler(m, n: int, c: openArray[string]): string =
  if m > 0: result.add(", ")
  result.add(case n:
    of 2: c[0].toLower & ": '" & c[1] & "'"
    of 1: c[0].toLower & ": ''"
    else: "")

let s = "Var1=key1;var2=Key2;   VAR3"
echo s.replace(peg"{\ident}('='{\ident})* ';'* \s*", handler)
# var1: 'key1', var2: 'Key2', var3: ''
```

---

### `parallelReplace` — параллельная замена

```nim
func parallelReplace*(s: string,
                      subs: varargs[tuple[pattern: Peg, repl: string]]): string
```

Применяет несколько замен **параллельно** (одновременно, не последовательно). Первое совпадающее правило применяется, остальные не проверяются для той же позиции.

```nim
import std/pegs

let result = "hello world 123".parallelReplace(
  (peg"\ident", "WORD"),
  (peg"\d+",    "NUM")
)
echo result  # "WORD WORD NUM"
```

---

### `transformFile` (не JS)

```nim
proc transformFile*(infile, outfile: string,
                    subs: varargs[tuple[pattern: Peg, repl: string]])
```

Читает файл `infile`, применяет `parallelReplace` и записывает результат в `outfile`. Вызывает `IOError` при ошибке. **Недоступно в JS-бэкенде.**

```nim
import std/pegs

transformFile("input.txt", "output.txt",
  (peg"'TODO'", "DONE"),
  (peg"\d{4}", "YEAR")
)
```

---

## Разбивка строки

### `split` — разбивка по PEG-разделителю

```nim
iterator split*(s: string, sep: Peg): string
func split*(s: string, sep: Peg): seq[string]
```

Разбивает строку `s` по разделителю `sep` (PEG-шаблон). Версия-функция возвращает `seq[string]`.

```nim
import std/pegs

for word in split("one,,two,,,three", peg",+"):
  echo word  # "one", "two", "three"

let parts = split("00232this02939is39an22example111", peg"\d+")
echo parts  # @["this", "is", "an", "example"]
```

---

## Низкоуровневое сопоставление

### `rawMatch`

```nim
func rawMatch*(s: string, p: Peg, start: int,
               c: var Captures): int
```

**Низкоуровневая** функция сопоставления — движок PEG. Все остальные операции в итоге вызывают её. Возвращает длину совпадения или -1.

- `s` — строка для анализа
- `p` — PEG-шаблон
- `start` — позиция начала сопоставления
- `c` — объект для записи захватов

```nim
import std/pegs

var c: Captures
let s = "hello 42"
let p = peg"\ident \s+ \d+"
let len = rawMatch(s, p, 0, c)
echo len  # 8 (вся строка)

# Доступ к захватам через bounds:
let p2 = peg"{\ident} \s+ {\d+}"
var c2: Captures
discard rawMatch(s, p2, 0, c2)
echo s[c2.bounds(0).first .. c2.bounds(0).last]  # "hello"
echo s[c2.bounds(1).first .. c2.bounds(1).last]  # "42"
```

---

## Событийный парсер

### `eventParser`

```nim
template eventParser*(pegAst, handlers: untyped): (proc(s: string): int)
```

Генерирует **событийный процедурный парсер** на основе PEG-дерева и блоков обработчиков. Возвращает `proc(s: string): int` — процедуру, вызываемую для разбора строки. Возвращает -1 при неудаче или длину совпадения.

Блоки обработчиков для каждого `PegKind`:
- `enter:` — вызывается при входе в элемент (доступны `s`, `p`, `start`)
- `leave:` — вызывается при выходе (доступны `s`, `p`, `start`, `length`; при неудаче `length = -1`)

```nim
import std/[strutils, pegs]

let pegAst = """
  Expr    <- Sum
  Sum     <- Product (('+' / '-') Product)*
  Product <- Value (('*' / '/') Value)*
  Value   <- [0-9]+ / '(' Expr ')'
""".peg

var valStack: seq[float] = @[]
var opStack = ""

let parseExpr = pegAst.eventParser:
  pkNonTerminal:
    leave:
      if length > 0:
        let matchStr = s.substr(start, start + length - 1)
        case p.nt.name
        of "Value":
          try: valStack.add matchStr.parseFloat
          except ValueError: discard
        of "Sum", "Product":
          try: discard matchStr.parseFloat
          except ValueError:
            if valStack.len > 1 and opStack.len > 0:
              let b = valStack.pop
              let a = valStack.pop
              valStack.add case opStack[^1]:
                of '+': a + b
                of '-': a - b
                of '*': a * b
                else: a / b
              opStack.setLen opStack.high
  pkChar:
    leave:
      if length == 1 and valStack.len > 0:
        let c = s[start]
        if c in {'+', '-', '*', '/'}:
          opStack.add c

let len = parseExpr("(5+3)/2-7*22")
echo valStack[^1]  # результат вычисления
```

---

## Утилиты

### `$` — строковое представление PEG

```nim
func `$`*(r: Peg): string
```

Конвертирует `Peg` в читаемую строку PEG-синтаксиса. Удобно для отладки.

```nim
import std/pegs

let p = sequence(+letters, *sequence(term(" "), +letters))
echo $p  # что-то вроде "([A-Za-z]+ (' ' [A-Za-z]+)*)"
```

---

## Быстрая шпаргалка

### Создание шаблона

| Задача | Код |
|--------|-----|
| Из строки (удобно) | `peg"\\ident '=' {.*}"` |
| Из строки с позицией | `parsePeg(s, "file.peg", 1, 0)` |
| Экранировать строку | `escapePeg("a+b")` |
| Любой символ | `any()` |
| Любой Unicode | `anyRune()` |
| Набор символов | `charSet({'a'..'z'})` |
| Буквальная строка | `term("hello")` |
| Без регистра | `termIgnoreCase("hello")` |
| Последовательность | `sequence(a, b, c)` |
| Выбор | `a / b / c` |
| 0 или 1 | `?a` |
| 0 и более | `*a` |
| 1 и более | `+a` |
| Не-предикат | `!a` |
| И-предикат | `&a` |
| Захват | `capture(a)` |
| Обратная ссылка | `backref(1)` |
| Поиск | `!*a` |
| Начало строки | `startAnchor()` |
| Конец строки | `endAnchor()` |

### Сопоставление

| Задача | Код |
|--------|-----|
| Быстрая проверка | `s =~ peg"..."` |
| Совпадение bool | `s.match(p)` |
| С захватами | `s.match(p, caps)` |
| Длина совпадения | `s.matchLen(p)` |
| Поиск позиции | `s.find(p)` |
| Границы | `s.findBounds(p, caps)` |
| Все совпадения | `s.findAll(p)` |
| Содержит? | `s.contains(p)` |
| Начинается с? | `s.startsWith(p)` |
| Заканчивается на? | `s.endsWith(p)` |

### Трансформация

| Задача | Код |
|--------|-----|
| Замена | `s.replace(p, "new")` |
| Замена с захватами | `s.replacef(p, "$1-$2")` |
| Замена с колбэком | `s.replace(p, myProc)` |
| Параллельная замена | `s.parallelReplace([(p1,"a"),(p2,"b")])` |
| Трансформация файла | `transformFile("in","out", subs)` |
| Разбивка | `s.split(p)` |
| Низкий уровень | `rawMatch(s, p, 0, captures)` |
