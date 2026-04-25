# Справочник модуля `std/parseopt` для Nim

> Стандартный разборщик аргументов командной строки. Разбирает аргументы на три вида токенов: короткие опции, длинные опции и позиционные аргументы. Следует принятым в Unix соглашениям по оформлению CLI.

---

## Содержание

1. [Концепции и поддерживаемый синтаксис](#концепции-и-поддерживаемый-синтаксис)
2. [Типы данных](#типы-данных)
   - [CmdLineKind](#cmdlinekind)
   - [OptParser](#optparser)
3. [initOptParser (строка)](#initoptparser-строка)
4. [initOptParser (seq\[string\])](#initoptparser-seqstring)
5. [next](#next)
6. [cmdLineRest](#cmdlinerest)
7. [remainingArgs](#remainingargs)
8. [getopt (итератор по OptParser)](#getopt-итератор-по-optparser)
9. [getopt (самостоятельный итератор)](#getopt-самостоятельный-итератор)
10. [Параметры shortNoVal и longNoVal](#параметры-shortnoval-и-longnoval)
11. [Полный практический пример](#полный-практический-пример)
12. [Шпаргалка](#шпаргалка)

---

## Концепции и поддерживаемый синтаксис

Прежде чем разбирать API, полезно понять, что именно парсер считает корректным вводом.

### Виды токенов

Каждый элемент командной строки классифицируется как одно из трёх:

| Вид | Пример | Смысл |
|---|---|---|
| Короткая опция | `-v`, `-e:5`, `-e=5`, `-abc` | Однобуквенный флаг, необязательно с значением |
| Длинная опция | `--verbose`, `--output:file`, `--output=file` | Словесный флаг, необязательно с значением |
| Аргумент | `file.txt`, `src/` | Всё, что не начинается с `-` |

### Объединение коротких опций в группу

Несколько коротких опций можно объединить в один токен: `-abc` эквивалентно `-a -b -c`. Только последняя буква в группе может нести значение: `-e:5` устанавливает опцию `e` равной `5`.

### Разделитель `--`

Голый токен `--` разбирается как длинная опция с **пустым именем** (`key == ""`). По соглашению это означает «всё, что следует дальше — это позиционные аргументы, а не опции». Парсер **не** применяет это соглашение автоматически — вы должны проверять `key == ""` в своём коде и вызывать `remainingArgs` (или `cmdLineRest`), чтобы получить остаток.

### Разделители значений

Значения присоединяются к опциям через `:` или `=`:
- `--output:file.txt`
- `--output=file.txt`
- `-o:file.txt`
- `-o=file.txt`

Если само значение начинается с `:` или `=`, удвойте разделитель или чередуйте их:
- `--delim::` → значение равно `:`
- `--delim=:` → значение равно `:`

---

## Типы данных

### `CmdLineKind`

```nim
type CmdLineKind* = enum
  cmdEnd,         ## Конец командной строки, больше токенов нет
  cmdArgument,    ## Позиционный аргумент (например, имя файла)
  cmdLongOption,  ## Длинная опция (--foo)
  cmdShortOption  ## Короткая опция (-f)
```

**Что это:** Перечисление, описывающее тип текущего токена. После каждого вызова `next` поле `kind` парсера принимает одно из этих четырёх значений.

`cmdEnd` — сторожевое значение: как только вы его видите, обработка заканчивается. Три активных значения соответствуют трём синтаксическим формам, описанным выше.

```nim
import std/parseopt

var p = initOptParser("--verbose -n 42 report.txt")
p.next()
assert p.kind == cmdLongOption   # --verbose
p.next()
assert p.kind == cmdShortOption  # -n
```

---

### `OptParser`

```nim
type OptParser* = object of RootObj
  kind*: CmdLineKind  # Вид последнего разобранного токена
  key*:  string       # Имя опции или текст аргумента
  val*:  string       # Значение опции, или "" если значения нет
  # (внутренние поля не экспортируются)
```

**Что это:** Объект состояния парсера. Хранит всё необходимое для пошагового прохода по командной строке. Никогда не создавайте его напрямую через `OptParser(...)` — всегда используйте `initOptParser`.

Три экспортированных поля, которые вы читаете после каждого вызова `next()`:

- `kind` — вид только что разобранного токена
- `key` — имя опции (для опций) или сам текст (для аргументов)
- `val` — строка-значение, если у опции оно есть; `""` в противном случае

```nim
import std/parseopt

var p = initOptParser("--output=report.pdf -v input.md")
p.next()
echo p.kind  # cmdLongOption
echo p.key   # "output"
echo p.val   # "report.pdf"
```

---

## `initOptParser` (строка)

```nim
proc initOptParser*(cmdline = "",
                    shortNoVal: set[char] = {},
                    longNoVal: seq[string] = @[];
                    allowWhitespaceAfterColon = true): OptParser
```

**Что делает:** Создаёт и возвращает новый `OptParser` из **единой строки**, которая сначала разбивается на токены (так же, как это делает оболочка). Если `cmdline` — пустая строка `""`, парсер читает реальную командную строку процесса через модуль `os`. Если платформа не может предоставить аргументы, возбуждается `ValueError`.

Необязательные параметры подробно описаны в разделе [shortNoVal и longNoVal](#параметры-shortnoval-и-longnoval).

`allowWhitespaceAfterColon` разрешает синтаксис `--key: value` (пробел между `:` и значением). По умолчанию `true`.

**Когда использовать:** Когда командная строка дана как единая строка — например, в тестах или при чтении команды из конфигурационного файла.

```nim
import std/parseopt

# Разбор строкового литерала (типично для тестов)
var p = initOptParser("--left --debug:3 -l -r:2")

# Разбор реальных аргументов процесса (продакшн-код)
var q = initOptParser()

# Разбор с подсказками NoVal
var r = initOptParser("--verbose -j4",
                      shortNoVal = {'v'},
                      longNoVal  = @["verbose"])
```

---

## `initOptParser` (seq[string])

```nim
proc initOptParser*(cmdline: seq[string],
                    shortNoVal: set[char] = {},
                    longNoVal: seq[string] = @[];
                    allowWhitespaceAfterColon = true): OptParser
```

**Что делает:** То же, что и строковая перегрузка, но принимает **последовательность строк**, в которой каждый элемент — уже отдельный токен. Это более естественная форма при работе с `os.commandLineParams()`, которая возвращает `seq[string]`, поскольку не требует дополнительного разбиения по пробелам. Если `cmdline` пуст (`@[]`), используются реальные аргументы процесса.

**Когда использовать:** В продакшн-программах, где аргументы приходят как `seq[string]` — что является обычным случаем.

```nim
import std/parseopt, std/os

# Напрямую передаём реальную командную строку — без повторного разбиения
var p = initOptParser(commandLineParams())

# Или явный список, удобно в юнит-тестах
var q = initOptParser(@["--output", "report.pdf", "-v", "input.md"])
```

---

## `next`

```nim
proc next*(p: var OptParser)
```

**Что делает:** Продвигает парсер на **один токен** вперёд и обновляет `p.kind`, `p.key` и `p.val`, чтобы описать этот токен. Когда командная строка исчерпана, устанавливает `p.kind = cmdEnd` и немедленно возвращается при всех последующих вызовах.

Это фундаментальная операция модуля — `getopt` — это просто цикл вокруг `next`. `next` используют напрямую, когда нужен тонкий контроль: например, чтобы остановить разбор на середине, особо обработать `--`, или в нужный момент вызвать `remainingArgs`.

**Важно:** `next` изменяет парсер на месте (`var OptParser`), поэтому парсер должен быть объявлен как `var`.

```nim
import std/parseopt

var p = initOptParser("--left -r:2 file.txt")

p.next()
assert p.kind == cmdLongOption and p.key == "left" and p.val == ""

p.next()
assert p.kind == cmdShortOption and p.key == "r" and p.val == "2"

p.next()
assert p.kind == cmdArgument and p.key == "file.txt"

p.next()
assert p.kind == cmdEnd   # Ничего не осталось; дальнейшие вызовы тоже вернут cmdEnd
```

**Обработка `--` (маркер окончания опций):**

```nim
import std/parseopt

var p = initOptParser("--verbose -- foo.txt bar.txt")
while true:
  p.next()
  if p.kind == cmdEnd: break
  if p.kind == cmdLongOption and p.key == "":
    # Встретили "--": всё, что дальше — сырые аргументы
    echo "Остаток: ", p.remainingArgs  # @["foo.txt", "bar.txt"]
    break
  echo p.key
```

---

## `cmdLineRest`

```nim
proc cmdLineRest*(p: OptParser): string
```

**Что делает:** Возвращает все токены, которые парсер **ещё не обработал**, собранные в единую строку, правильно экранированную для оболочки. Это фактически хвост командной строки в форме, пригодной для передачи другому процессу или интерпретации оболочкой.

> **Платформенная оговорка:** Эта процедура доступна только на платформах, где объявлена `quoteShellCommand` (Windows и POSIX). В NimScript недоступна.

**Когда использовать:** Когда ваша программа делегирует часть аргументов дочернему процессу. После обработки своих опций передайте остаток как готовую строку для оболочки.

```nim
import std/parseopt

var p = initOptParser("--left -r:2 -- foo.txt bar.txt")
while true:
  p.next()
  if p.kind == cmdLongOption and p.key == "":  # Встретили "--"
    break
  if p.kind == cmdEnd: break

# Всё после "--" как строка, готовая для оболочки
echo p.cmdLineRest  # "foo.txt bar.txt"
```

---

## `remainingArgs`

```nim
proc remainingArgs*(p: OptParser): seq[string]
```

**Что делает:** Возвращает `seq[string]` из всех токенов, которые парсер **ещё не обработал**. В отличие от `cmdLineRest`, это набор отдельных строк, а не одна склеенная строка. С последовательностью удобнее работать в Nim-коде.

**Когда использовать:** Когда нужно собрать оставшиеся аргументы в список — например, все позиционные файлы после разделителя `--`, или когда разбор прерывается досрочно и остаток нужно передать дальше.

```nim
import std/parseopt

var p = initOptParser("--left -r:2 -- foo.txt bar.txt")
while true:
  p.next()
  if p.kind == cmdLongOption and p.key == "":  # Встретили "--"
    break
  if p.kind == cmdEnd: break

let rest = p.remainingArgs
assert rest == @["foo.txt", "bar.txt"]

for f in rest:
  echo "Буду обрабатывать: ", f
```

---

## `getopt` (итератор по OptParser)

```nim
iterator getopt*(p: var OptParser): tuple[kind: CmdLineKind, key, val: string]
```

**Что делает:** Удобный итератор, оборачивающий `next` в цикл. Сбрасывает парсер в начало (`pos = 0`, `idx = 0`), затем многократно вызывает `next`, выдавая каждый кортеж `(kind, key, val)` до достижения `cmdEnd` — после чего молча останавливается. Значение `cmdEnd` внутри тела цикла **никогда не появляется**.

**Когда использовать:** Когда хочется обойти уже созданный `OptParser` через цикл `for`, не писать вручную `while true` / `next` / `break`. Немного выразительнее ручного варианта.

**Важное отличие от самостоятельного `getopt`:** Эта перегрузка принимает существующий `OptParser` по ссылке `var` и **сбрасывает** его при каждом вызове. Если вы уже продвинули парсер вызовами `next`, вызов этого итератора перезапустит обход с начала.

```nim
import std/parseopt

var p = initOptParser("--output=report.pdf -v input.md")

for kind, key, val in p.getopt():
  case kind
  of cmdLongOption, cmdShortOption:
    echo "Опция: ", key, if val != "": " = " & val else: ""
  of cmdArgument:
    echo "Файл: ", key
  of cmdEnd:
    discard  # сюда никогда не доходим
```

Вывод:
```
Опция: output = report.pdf
Опция: v
Файл: input.md
```

---

## `getopt` (самостоятельный итератор)

```nim
iterator getopt*(cmdline: seq[string] = @[],
                 shortNoVal: set[char] = {},
                 longNoVal: seq[string] = @[]):
         tuple[kind: CmdLineKind, key, val: string]
```

**Что делает:** Самодостаточный итератор: создаёт `OptParser` внутри себя и сразу итерирует по нему. При пустом `cmdline` (`@[]`) читает реальную командную строку процесса. Явно создавать `OptParser` не нужно.

**Когда использовать:** В простых программах, где аргументы обрабатываются ровно один раз, от начала до конца, без необходимости делать паузы, откатываться назад или обращаться к `remainingArgs`. Это самый лаконичный вариант для типичного случая.

```nim
import std/parseopt

var output  = "a.out"
var verbose = false
var files: seq[string]

for kind, key, val in getopt():
  case kind
  of cmdShortOption, cmdLongOption:
    case key
    of "o", "output":  output  = val
    of "v", "verbose": verbose = true
  of cmdArgument:
    files.add(key)
  of cmdEnd:
    discard  # внутри getopt не достигается

echo output, " | ", verbose, " | ", files
```

**С явным списком аргументов (удобно в тестах):**

```nim
import std/parseopt

for kind, key, val in getopt(@["--output=out.txt", "-v", "main.nim"]):
  echo kind, " | ", key, " | ", val
```

---

## Параметры `shortNoVal` и `longNoVal`

Эти два необязательных параметра `initOptParser` (а также самостоятельного `getopt`) сообщают парсеру, какие опции **не принимают значений**. Это открывает дополнительный синтаксис, который без подсказок был бы неоднозначным.

### Проблема: неоднозначные группы и значения через пробел

Рассмотрим командную строку `-j4`. Без подсказок парсер не знает, означает ли это «опция `j` со значением `4`» или «опции `j` и `4`». Аналогично, `--jobs 4` — это значение опции `--jobs` или отдельный аргумент?

По умолчанию парсер выбирает консервативную интерпретацию:
- `-j4` → короткая опция `j`, затем короткая опция `4` (два отдельных токена)
- `--jobs 4` → длинная опция `jobs` (без значения), затем аргумент `4`

### Решение: объявить, какие опции не берут значений

Как только вы сообщаете парсеру, какие опции **никогда** не имеют значений, он делает вывод, что все *остальные* опции **имеют** значения, и начинает поддерживать естественный синтаксис:

```nim
import std/parseopt

# Без подсказок: -j4 — это две опции; --first bar — опция + аргумент
var p1 = initOptParser("-j4 --first bar")
for kind, key, val in p1.getopt():
  echo kind, " | ", key, " | ", val
# cmdShortOption | j     |
# cmdShortOption | 4     |
# cmdLongOption  | first |
# cmdArgument    | bar   |

# С подсказками: 'j' нет в shortNoVal → -j4 означает j=4
#                "first" нет в longNoVal → --first bar означает first=bar
var p2 = initOptParser("-j4 --first bar",
                       shortNoVal = {'v'},        # 'v' без значения; 'j' — с ним
                       longNoVal  = @["verbose"]) # "verbose" без значения; "first" — с ним
for kind, key, val in p2.getopt():
  echo kind, " | ", key, " | ", val
# cmdShortOption | j     | 4
# cmdLongOption  | first | bar
```

### Когда использовать

Применяйте `shortNoVal` и `longNoVal`, если ваш CLI использует:
- Стиль `-j8` (количество параллельных задач как у make)
- Стиль `--jobs 8` (GNU-стиль: значение через пробел)

**Важно:** держите эти наборы **синхронизированными** с реальным набором опций программы. Если добавите новый флаг-без-значения и забудете включить его в `shortNoVal`/`longNoVal`, парсер неожиданно «съест» следующий токен как его значение.

---

## Полный практический пример

Реалистичная программа, сочетающая все возможности модуля:

```nim
import std/parseopt, std/os, std/strutils

# Значения по умолчанию
var
  outputFile  = "a.out"
  verbosity   = 0
  jobs        = 1
  inputFiles: seq[string]

proc writeHelp() =
  echo """Использование: mytool [опции] <файлы>
Опции:
  -h, --help          Показать эту справку
  -v, --verbose       Увеличить подробность (можно повторять)
  -j<N>, --jobs <N>   Число параллельных задач (по умолчанию: 1)
  -o, --output <файл> Выходной файл (по умолчанию: a.out)
  --                  Прекратить разбор опций
"""

var p = initOptParser(commandLineParams(),
                      shortNoVal = {'h', 'v'},
                      longNoVal  = @["help", "verbose"])

while true:
  p.next()
  case p.kind
  of cmdEnd: break
  of cmdLongOption:
    if p.key == "":            # Встретили голый "--"
      for f in p.remainingArgs: inputFiles.add(f)
      break
    case p.key
    of "help":    writeHelp(); quit(0)
    of "verbose": inc verbosity
    of "jobs":    jobs = p.val.parseInt
    of "output":  outputFile = p.val
    else:
      echo "Неизвестная опция: --", p.key; quit(1)
  of cmdShortOption:
    case p.key
    of "h": writeHelp(); quit(0)
    of "v": inc verbosity
    of "j": jobs = p.val.parseInt
    of "o": outputFile = p.val
    else:
      echo "Неизвестная опция: -", p.key; quit(1)
  of cmdArgument:
    inputFiles.add(p.key)

echo "Выход:       ", outputFile
echo "Подробность: ", verbosity
echo "Задачи:      ", jobs
echo "Файлы:       ", inputFiles
```

---

## Шпаргалка

| Задача | Код |
|---|---|
| Создать парсер из строки | `initOptParser("--foo bar")` |
| Создать парсер из seq | `initOptParser(commandLineParams())` |
| Создать парсер для реального argv | `initOptParser()` |
| Продвинуться на один токен | `p.next()` |
| Проверить вид токена | `p.kind` → `cmdShortOption`, `cmdLongOption`, `cmdArgument`, `cmdEnd` |
| Прочитать имя опции / аргумент | `p.key` |
| Прочитать значение опции | `p.val` (пустая строка, если значения нет) |
| Итерировать удобно | `for kind, key, val in p.getopt()` |
| Итерировать без явного парсера | `for kind, key, val in getopt()` |
| Получить необработанные аргументы как seq | `p.remainingArgs` |
| Получить необработанные аргументы как строку | `p.cmdLineRest` |
| Включить синтаксис `-j4` | `shortNoVal = {'h', 'v'}` (перечислить флаги без значений) |
| Включить синтаксис `--jobs 4` | `longNoVal = @["help", "verbose"]` |
| Обнаружить разделитель `--` | `p.kind == cmdLongOption and p.key == ""` |
