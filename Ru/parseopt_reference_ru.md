# Справочник модуля `std/parseopt` — Nim Standard Library

> Стандартный парсер аргументов командной строки для Nim.

---

## Содержание

1. [Обзор модуля](#1-обзор-модуля)
2. [Поддерживаемый синтаксис](#2-поддерживаемый-синтаксис)
3. [Типы и перечисления](#3-типы-и-перечисления)
4. [Инициализация: `initOptParser`](#4-инициализация-initoptparser)
5. [Итератор `getopt`](#5-итератор-getopt)
6. [Ручной обход: `next`](#6-ручной-обход-next)
7. [Параметры `shortNoVal` и `longNoVal`](#7-параметры-shortnoval-и-longnoval)
8. [Режимы парсера (`CliMode`)](#8-режимы-парсера-climode)
9. [Получение остатка: `cmdLineRest` и `remainingArgs`](#9-получение-остатка-cmdlinerest-и-remainingargs)
10. [Устаревший API](#10-устаревший-api)
11. [Практические примеры](#11-практические-примеры)
12. [Таблица быстрого доступа](#12-таблица-быстрого-доступа)

---

## 1. Обзор модуля

Модуль `std/parseopt` реализует полнофункциональный парсер аргументов командной строки. Он поддерживает короткие и длинные опции, аргументы, группировку (bundling) коротких флагов и несколько режимов совместимости.

### Подключение

```nim
import std/parseopt
```

### Связанные модули

| Модуль | Назначение |
|--------|------------|
| `std/os` | Низкоуровневый доступ к аргументам (`paramCount`, `paramStr`) |
| `std/parseutils` | Утилиты для разбора токенов, чисел, идентификаторов |
| `std/strutils` | Операции со строками (полезны при обработке значений опций) |
| `std/parsecfg` | Парсер конфигурационных файлов |

---

## 2. Поддерживаемый синтаксис

### Базовые форматы

| Формат | Тип | Описание |
|--------|-----|----------|
| `-a` | Короткая опция без значения | Флаг |
| `-a:5` | Короткая опция со значением (через `:`) | Опция с явным разделителем |
| `-a=5` | Короткая опция со значением (через `=`) | Опция с явным разделителем |
| `-cde` | Группировка коротких опций | Три флага `c`, `d`, `e` в одном аргументе |
| `-fgh=5` | Группировка с значением у последней | Флаги `f`, `g`, затем `h=5` |
| `--foo` | Длинная опция без значения | Флаг |
| `--foo:bar` | Длинная опция со значением | Через разделитель `:` |
| `--foo=bar` | Длинная опция со значением | Через разделитель `=` |
| `file.txt` | Аргумент | Всё, что не начинается с `-` |

### Особые случаи

**Значения, начинающиеся с разделителя** — корректны:

```
--foo::     → опция foo, значение ":"
--foo=:     → опция foo, значение ":"
--foo:=     → опция foo, значение "="
--foo==     → опция foo, значение "="
```

**Разделитель `--`** — специальная длинная опция с пустым именем (`key == ""`). Традиционно означает «все следующие токены — это аргументы». Парсер не обрабатывает `--` автоматически — программист сам ловит этот случай и вызывает `remainingArgs`.

**Пробелы вокруг разделителя** (в `NimMode` и `LaxMode`):

```
--foo: bar   → опция foo, значение "bar"  (пробел после : разрешён)
--foo =bar   → опция foo, значение "bar"  (пробел перед = разрешён)
```

---

## 3. Типы и перечисления

### `CmdLineKind` — вид распознанного токена

```nim
type
  CmdLineKind* = enum
    cmdEnd,          ## Конец потока аргументов
    cmdArgument,     ## Аргумент (не опция), например имя файла
    cmdLongOption,   ## Длинная опция: --foo
    cmdShortOption   ## Короткая опция: -f
```

Это перечисление лежит в поле `kind` объекта `OptParser` и возвращается итератором `getopt`.

| Значение | Когда встречается | Что содержит `key` | Что содержит `val` |
|----------|-------------------|--------------------|--------------------|
| `cmdEnd` | Токены закончились | `""` | `""` |
| `cmdArgument` | Токен без `-` | Сам аргумент | `""` |
| `cmdLongOption` | Токен `--foo` | `"foo"` | Значение или `""` |
| `cmdShortOption` | Токен `-f` | `"f"` | Значение или `""` |

> Специальный случай `--`: `kind = cmdLongOption`, `key = ""`, `val = ""`.

---

### `CliMode` — режим поведения парсера

```nim
type
  CliMode* = enum
    LaxMode,  ## Наиболее гибкий режим (POSIX-подобный)
    NimMode,  ## Стандартный режим Nim (по умолчанию)
    GnuMode   ## GNU-совместимый режим
```

Подробное описание режимов — в [разделе 8](#8-режимы-парсера-climode).

---

### `OptParser` — объект парсера

```nim
type
  OptParser* = object of RootObj
    kind*: CmdLineKind  ## Тип последнего распознанного токена
    key*:  string       ## Имя опции или текст аргумента
    val*:  string       ## Значение опции (пусто, если не задано)
    # ... внутренние поля (pos, idx, cmds, rules и др.)
```

Поля `kind`, `key` и `val` — единственные публичные поля объекта. Остальные — детали реализации и недоступны напрямую.

После каждого вызова `next` или каждой итерации `getopt` эти поля обновляются.

---

## 4. Инициализация: `initOptParser`

Существует две публичные перегрузки: принимающая строку и принимающая `seq[string]`.

### Из строки

```nim
proc initOptParser*(cmdline = "";
                    shortNoVal: set[char] = {};
                    longNoVal: seq[string] = @[];
                    mode: CliMode = NimMode): OptParser
```

Строка `cmdline` разбивается на токены по правилам shell-цитирования (с учётом кавычек). Если `cmdline` пуст, читаются реальные аргументы процесса через `os.paramStr`.

```nim
import std/parseopt

# Из строки (удобно для тестов)
var p1 = initOptParser("--left --debug:3 -l -r:2")

# Из строки с указанием флагов без значений
var p2 = initOptParser("--left -lr",
                        shortNoVal = {'l', 'r'},
                        longNoVal = @["left"])

# Из реальной командной строки
var p3 = initOptParser()

# С режимом GNU
var p4 = initOptParser("--foo=bar -c val", mode = GnuMode)
```

---

### Из `seq[string]`

```nim
proc initOptParser*(cmdline: seq[string];
                    shortNoVal: set[char] = {};
                    longNoVal: seq[string] = @[];
                    mode: CliMode = NimMode): OptParser
```

Принимает уже разбитый список токенов — это точнее, чем строка, потому что не требует повторного разбора shell-кавычек. Оптимально для передачи `commandLineParams()`.

```nim
import std/parseopt, std/os

# Из реальных аргументов — самый правильный способ
var p = initOptParser(commandLineParams())

# Из явного списка (например, в тестах)
var p2 = initOptParser(@["--output", "file.txt", "-v", "input.nim"])

# С опциями без значений
var p3 = initOptParser(
  @["--verbose", "-n", "10"],
  shortNoVal = {'v'},
  longNoVal = @["verbose", "help"]
)
```

---

### Параметры `initOptParser` — сводка

| Параметр | Тип | По умолчанию | Описание |
|----------|-----|--------------|----------|
| `cmdline` | `string` или `seq[string]` | `""` / `@[]` | Аргументы для разбора. Пусто → реальные аргументы процесса |
| `shortNoVal` | `set[char]` | `{}` | Короткие опции, **не принимающие** значения |
| `longNoVal` | `seq[string]` | `@[]` | Длинные опции, **не принимающие** значения |
| `mode` | `CliMode` | `NimMode` | Режим поведения парсера |

---

## 5. Итератор `getopt`

Итератор `getopt` — основной и наиболее удобный способ обойти все токены. Он существует в двух вариантах.

### Из существующего `OptParser`

```nim
iterator getopt*(p: var OptParser): tuple[kind: CmdLineKind, key, val: string]
```

Итерирует по токенам, хранящимся в уже созданном `OptParser`. **Автоматически останавливается** на `cmdEnd` — проверять его внутри цикла `for` не нужно. При использовании `case` с `cmdEnd` следует ставить `assert(false)` или `discard`.

```nim
import std/parseopt

var p = initOptParser("--left --debug:3 -l -r:2")
for kind, key, val in p.getopt():
  case kind
  of cmdArgument:
    echo "Аргумент: ", key
  of cmdLongOption, cmdShortOption:
    if val == "":
      echo "Флаг: ", key
    else:
      echo "Опция: ", key, " = ", val
  of cmdEnd:
    assert false  # никогда не достигается
```

---

### Напрямую из аргументов

```nim
iterator getopt*(cmdline: seq[string] = @[];
                 shortNoVal: set[char] = {};
                 longNoVal: seq[string] = @[];
                 mode: CliMode = NimMode):
    tuple[kind: CmdLineKind, key, val: string]
```

Создаёт `OptParser` внутри и итерирует сразу. Параметры те же, что у `initOptParser`. Если `cmdline` пуст, используются реальные аргументы процесса.

```nim
import std/parseopt

# Самый короткий способ обработать реальные аргументы
for kind, key, val in getopt():
  case kind
  of cmdShortOption, cmdLongOption:
    echo key, " = ", val
  of cmdArgument:
    echo "файл: ", key
  of cmdEnd: discard

# С явным списком и настройками
for kind, key, val in getopt(
    @["--output=file.txt", "-v"],
    shortNoVal = {'v'},
    longNoVal = @["verbose"]):
  discard
```

---

### Полный пример с обработкой всех типов токенов

```nim
import std/parseopt

let cmds = "-ab -e:5 --foo --bar=20 file.txt".parseCmdLine()
var output: seq[string] = @[]

for kind, key, val in getopt(cmds):
  case kind
  of cmdEnd: break
  of cmdShortOption, cmdLongOption:
    if val == "":
      output.add("Option: " & key)
    else:
      output.add("Option and value: " & key & ", " & val)
  of cmdArgument:
    output.add("Argument: " & key)

doAssert output == @[
  "Option: a",
  "Option: b",
  "Option and value: e, 5",
  "Option: foo",
  "Option and value: bar, 20",
  "Argument: file.txt"
]
```

---

## 6. Ручной обход: `next`

```nim
proc next*(p: var OptParser)
```

Продвигает парсер на один токен вперёд. После вызова поля `p.kind`, `p.key` и `p.val` содержат данные о текущем токене. Когда аргументы закончились, `p.kind` становится `cmdEnd`.

Используется, когда нужен более тонкий контроль над процессом разбора: например, прерваться на середине, пропустить токен, или обработать `--` с последующим получением остатка.

```nim
import std/parseopt

var p = initOptParser("--left -r:2 file.txt")

p.next()
doAssert p.kind == cmdLongOption and p.key == "left"

p.next()
doAssert p.kind == cmdShortOption and p.key == "r" and p.val == "2"

p.next()
doAssert p.kind == cmdArgument and p.key == "file.txt"

p.next()
doAssert p.kind == cmdEnd   # конец потока
```

### Цикл с `next` вручную

```nim
import std/parseopt

var p = initOptParser("--verbose -o output.txt data.csv")
while true:
  p.next()
  case p.kind
  of cmdEnd: break
  of cmdLongOption:
    echo "длинная: --", p.key, if p.val != "": "=" & p.val else: ""
  of cmdShortOption:
    echo "короткая: -", p.key, if p.val != "": "=" & p.val else: ""
  of cmdArgument:
    echo "аргумент: ", p.key
```

---

## 7. Параметры `shortNoVal` и `longNoVal`

Параметры `shortNoVal` и `longNoVal` — ключевой механизм для управления тем, **может ли опция принимать значение без явного разделителя**.

### Поведение по умолчанию (пустые `shortNoVal`/`longNoVal`)

Когда оба параметра пусты (по умолчанию):

- Короткая опция принимает значение **только** через явный разделитель (`-k:val`, `-k=val`) или прямое примыкание (`-kval` — значение `val` у опции `k`).
- `-j4` без разделителя разбирается как **два** отдельных флага: `j` и `4`.
- `--foo bar` — `--foo` — флаг без значения, `bar` — отдельный аргумент.

```nim
import std/parseopt

var p = initOptParser("-j4 --first bar")
# shortNoVal пуст → -j и 4 это два разных флага
for kind, key, val in p.getopt():
  echo kind, ": ", key, " = ", val
# Вывод:
# cmdShortOption: j =
# cmdShortOption: 4 =
# cmdLongOption:  first =
# cmdArgument:    bar =
```

---

### С заданными `shortNoVal`/`longNoVal`

Когда параметры заданы, парсер знает, какие опции **не ожидают значения**. Тогда:

- `-j4` при `shortNoVal = {'j'}` **не содержит** `j`: `-j4` → опция `j`, значение `4`
- `--foo bar` при `longNoVal = @["bar"]` — `--foo` — опция **с** значением `bar` (следующий токен)
- Если же `j` есть в `shortNoVal`, то `-j4` → флаг `j`, затем флаг `4` (как раньше)

```nim
import std/parseopt

var p = initOptParser("-j4 --first bar",
                      shortNoVal = {'c'},      # 'j' не в списке → может принимать значение
                      longNoVal = @["second"]) # "first" не в списке → может принимать значение
for kind, key, val in p.getopt():
  echo kind, ": ", key, " = ", val
# Вывод:
# cmdShortOption: j = 4
# cmdLongOption:  first = bar
```

---

### Активация next-argument value-taking

> ⚠️ **Важно:** Next-argument value-taking (значение из следующего токена) включается **только** тогда, когда `shortNoVal` или `longNoVal` **не пусты**. Если все ваши опции принимают значения, передайте фиктивный элемент:

```nim
# Все опции принимают значения, но хотим включить --foo bar
var p = initOptParser(cmdline,
  shortNoVal = {'\0'},   # фиктивный элемент
  longNoVal = @[""])     # фиктивный элемент
```

---

### Парсер не запрещает явные значения для noVal-опций

Если пользователь передаёт значение явно (`--foo:bar`) для опции, помеченной как `longNoVal`, парсер всё равно распознаёт это значение — он не выдаёт ошибку. Это позволяет обнаружить ошибочное использование в коде приложения:

```nim
import std/[sequtils, os, parseopt]

let cmds = "-n:9 --foo:bar".parseCmdLine()
let parsed = toSeq(cmds.getopt(shortNoVal = {'n'}, longNoVal = @["foo"]))

for (kind, key, val) in parsed:
  case kind
  of cmdShortOption, cmdLongOption:
    if key in ["n", "foo"] and val != "":
      echo "Ошибка: опция ", key, " не принимает значений, но получила: ", val
  else: discard

# parsed == @[(cmdShortOption, "n", "9"), (cmdLongOption, "foo", "bar")]
```

---

### Сводная таблица поведения

| Синтаксис | `shortNoVal = {}` | `shortNoVal = {'j'}` | `shortNoVal` не содержит `j` |
|-----------|-------------------|--------------------|-------------------------------|
| `-j4` | флаги `j`, `4` | флаги `j`, `4` | опция `j = "4"` |
| `-j:4` | опция `j = "4"` | опция `j = "4"` | опция `j = "4"` |
| `-j 4` (LaxMode) | флаг `j`, аргумент `4` | флаг `j`, аргумент `4` | опция `j = "4"` |

---

## 8. Режимы парсера (`CliMode`)

> ⚠️ Режимы `LaxMode` и `GnuMode` являются **экспериментальными** и могут изменяться в будущих версиях Nim.

Режим парсера задаётся параметром `mode` в `initOptParser` или `getopt`. По умолчанию используется `NimMode`.

---

### `NimMode` (по умолчанию)

Стандартный режим Nim. Баланс между удобством и предсказуемостью.

**Характеристики:**

- Оба разделителя `:` и `=` допустимы
- Пробелы **до и после** разделителя допустимы: `--foo: bar`, `--foo =bar`
- Группировка коротких флагов: `-abc` → флаги `a`, `b`, `c`
- Смежные значения у коротких опций: `-kval` → опция `k = "val"` (только если `k` не в `shortNoVal`)
- Next-argument (`-k val`) **не поддерживается** по умолчанию (только при непустом `shortNoVal`)
- Значения, начинающиеся с `-`, трактуются как новые опции

```nim
import std/parseopt

# NimMode (по умолчанию)
for kind, key, val in getopt(@["--foo:bar", "--baz =qux", "-k5"]):
  echo key, " = ", val
# foo = bar
# baz = qux
# k = 5  (смежное значение)
```

---

### `LaxMode`

Наиболее гибкий режим. Сочетает `NimMode` с POSIX-подобной обработкой коротких опций.

**Дополнительно к `NimMode`:**

- `-c val` поддерживается: следующий токен становится значением `-c`
- `-abc val`: флаги `a`, `b`, затем опция `c = "val"`
- Значения, начинающиеся с `-`, можно передавать как аргументы опций: `-n -10`

```nim
import std/parseopt

# LaxMode: -c val работает
for kind, key, val in getopt(
    @["-c", "hello", "--level", "5"],
    shortNoVal = {'\0'},  # активируем next-arg
    longNoVal = @[""],
    mode = LaxMode):
  echo key, " = ", val
# c = hello
# level = 5
```

---

### `GnuMode`

Следует соглашениям GNU getopt. Более строгий в части разделителей.

**Характеристики:**

- **Только `=`** является разделителем (`:` не является разделителем)
- Пробелы вокруг `=` **не допускаются**: `--foo =bar` → опция `foo` без значения, аргумент `=bar`
- `-c val` поддерживается (как LaxMode)
- Значения, начинающиеся с `-`, допустимы как аргументы опций
- Двоеточие `:` не имеет специального значения

```nim
import std/parseopt

# GnuMode: только = как разделитель
for kind, key, val in getopt(
    @["--foo=bar", "--baz:qux"],
    mode = GnuMode):
  echo key, " = ", val
# foo = bar
# baz:qux =    ← `:` не разделитель → вся строка — имя опции
```

---

### Сравнительная таблица режимов

| Поведение | `NimMode` | `LaxMode` | `GnuMode` |
|-----------|-----------|-----------|-----------|
| Разделитель `:` | ✅ | ✅ | ❌ |
| Разделитель `=` | ✅ | ✅ | ✅ |
| Пробел до разделителя | ✅ | ✅ | ❌ |
| Пробел после разделителя | ✅ | ✅ | ❌ |
| `-kval` (смежное значение) | ✅ | ✅ | ✅ |
| `-k val` (следующий токен) | ❌\* | ✅\* | ✅\* |
| Значения, начинающиеся с `-` | ❌ | ✅ | ✅ |
| Группировка `-abc` | ✅ | ✅ | ✅ |

\* Требует непустых `shortNoVal`/`longNoVal`

---

### Разбор одинаковых строк в разных режимах

```nim
import std/parseopt

let cmdline = @["--foo:bar", "--baz", "=qux", "-c", "-10"]

for mode in [NimMode, LaxMode, GnuMode]:
  echo "=== ", mode, " ==="
  for kind, key, val in getopt(cmdline,
      shortNoVal = {'\0'}, longNoVal = @["baz"], mode = mode):
    echo "  ", kind, ": '", key, "' = '", val, "'"
```

---

## 9. Получение остатка: `cmdLineRest` и `remainingArgs`

Обе процедуры предназначены для получения токенов, которые ещё не были обработаны парсером. Типичный сценарий — обработка разделителя `--`.

---

### `remainingArgs`

```nim
proc remainingArgs*(p: OptParser): seq[string]
```

Возвращает `seq[string]` из ещё не обработанных токенов. **Предпочтительный** способ — возвращает токены в том виде, в каком они были переданы, без изменений.

```nim
import std/parseopt

var p = initOptParser("--left -r:2 -- foo.txt bar.txt")
while true:
  p.next()
  if p.kind == cmdLongOption and p.key == "":  # поймали "--"
    break

let rest = p.remainingArgs
doAssert rest == @["foo.txt", "bar.txt"]

# Теперь можно использовать rest как список файлов
for f in rest:
  echo "Обрабатываем файл: ", f
```

---

### `cmdLineRest`

```nim
proc cmdLineRest*(p: OptParser): string
```

Возвращает остаток командной строки в виде одной строки, с восстановленным экранированием (через `quoteShellCommand`). Доступна только на платформах, где определена `quoteShellCommand` (POSIX / Windows).

```nim
import std/parseopt

var p = initOptParser("--left -r:2 -- foo.txt bar.txt")
while true:
  p.next()
  if p.kind == cmdLongOption and p.key == "":
    break

echo p.cmdLineRest   # => "foo.txt bar.txt"
```

> **Когда что использовать:**  
> - `remainingArgs` — для программной обработки: возвращает структурированный `seq[string]`  
> - `cmdLineRest` — для передачи остатка другой программе или shell-команде целиком

---

## 10. Устаревший API

### `allowWhitespaceAfterColon` (deprecated)

```nim
proc initOptParser*(cmdline = ""; shortNoVal: set[char] = {};
                    longNoVal: seq[string] = @[];
                    allowWhitespaceAfterColon: bool): OptParser
  {.deprecated: "`allowWhitespaceAfterColon` is deprecated, use parser modes instead".}
```

Старая версия `initOptParser`, контролировавшая пробелы вокруг разделителей. Заменена параметром `mode`:

| Старый вызов | Новый эквивалент |
|---|---|
| `initOptParser(cmd, allowWhitespaceAfterColon = true)` | `initOptParser(cmd, mode = NimMode)` ← по умолчанию |
| `initOptParser(cmd, allowWhitespaceAfterColon = false)` | `initOptParser(cmd, mode = GnuMode)` |

---

## 11. Практические примеры

### Минимальный CLI с двумя опциями и одним аргументом

```nim
import std/parseopt

proc writeHelp() =
  echo """
Использование: mytool [опции] <файл>

Опции:
  -v, --verbose     Подробный вывод
  -o, --output=FILE Файл для записи результата (по умолчанию: stdout)
  -h, --help        Показать эту справку
"""

proc writeVersion() =
  echo "mytool v1.0.0"

var
  verbose  = false
  output   = ""
  filename = ""

for kind, key, val in getopt(shortNoVal = {'v', 'h'},
                              longNoVal = @["verbose", "help"]):
  case kind
  of cmdEnd: break
  of cmdArgument:
    filename = key
  of cmdShortOption, cmdLongOption:
    case key
    of "h", "help":    writeHelp(); quit(0)
    of "v", "verbose": verbose = true
    of "o", "output":  output = val
    else:
      echo "Неизвестная опция: ", key
      quit(1)

if filename == "":
  writeHelp()
  quit(1)

echo "Файл: ", filename
echo "Verbose: ", verbose
echo "Output: ", if output == "": "(stdout)" else: output
```

---

### Значения по умолчанию и валидация

```nim
import std/parseopt, std/strutils

# Значения по умолчанию
var
  host    = "localhost"
  port    = 8080
  debug   = false
  workers = 4

for kind, key, val in getopt(shortNoVal = {'d'},
                              longNoVal = @["debug"]):
  case kind
  of cmdEnd: break
  of cmdArgument:
    echo "Лишний аргумент: ", key
  of cmdShortOption, cmdLongOption:
    case key
    of "h", "host":
      host = val
    of "p", "port":
      try:
        port = parseInt(val)
        if port notin 1..65535:
          raise newException(ValueError, "порт вне диапазона")
      except ValueError as e:
        echo "Ошибка: неверный порт: ", e.msg
        quit(1)
    of "d", "debug":
      debug = true
    of "w", "workers":
      workers = parseInt(val)
    else:
      echo "Неизвестная опция: ", key
      quit(1)

echo "Сервер: ", host, ":", port
echo "Отладка: ", debug
echo "Воркеры: ", workers
```

---

### Обработка `--` и positional-аргументов после него

```nim
import std/parseopt

var
  options: seq[(string, string)] = @[]
  files:   seq[string] = @[]

var p = initOptParser()
while true:
  p.next()
  case p.kind
  of cmdEnd: break
  of cmdArgument:
    files.add p.key
  of cmdLongOption:
    if p.key == "":   # встретили "--"
      files.add p.remainingArgs()   # всё после "--" — файлы
      break
    options.add((p.key, p.val))
  of cmdShortOption:
    options.add((p.key, p.val))

echo "Опции: ", options
echo "Файлы: ", files
```

---

### Парсер с sub-командами (git-стиль)

```nim
import std/parseopt

type SubCommand = enum
  scNone, scBuild, scTest, scRun

var
  subcmd  = scNone
  verbose = false
  output  = ""
  args:   seq[string] = @[]

var p = initOptParser()
p.next()

# Первый токен — субкоманда
if p.kind == cmdArgument:
  case p.key
  of "build": subcmd = scBuild
  of "test":  subcmd = scTest
  of "run":   subcmd = scRun
  else:
    echo "Неизвестная команда: ", p.key
    quit(1)
else:
  echo "Ожидается субкоманда (build, test, run)"
  quit(1)

# Остальные токены — опции субкоманды
for kind, key, val in p.getopt():
  case kind
  of cmdEnd: break
  of cmdArgument: args.add key
  of cmdShortOption, cmdLongOption:
    case key
    of "v", "verbose": verbose = true
    of "o", "output":  output = val
    else:
      echo "Неизвестная опция: ", key

echo "Команда: ", subcmd
echo "Verbose: ", verbose
echo "Output:  ", output
echo "Аргументы: ", args
```

---

### Использование в тестах (без реальных аргументов)

```nim
import std/parseopt

proc parseMyArgs(cmdline: seq[string]): tuple[verbose: bool, files: seq[string]] =
  result = (verbose: false, files: @[])
  for kind, key, val in getopt(cmdline, shortNoVal = {'v'},
                                         longNoVal = @["verbose"]):
    case kind
    of cmdArgument:
      result.files.add key
    of cmdShortOption, cmdLongOption:
      if key in ["v", "verbose"]:
        result.verbose = true
    of cmdEnd: break

# Юнит-тесты:
let r1 = parseMyArgs(@["-v", "file.txt"])
doAssert r1.verbose == true
doAssert r1.files == @["file.txt"]

let r2 = parseMyArgs(@["--verbose", "a.nim", "b.nim"])
doAssert r2.verbose == true
doAssert r2.files == @["a.nim", "b.nim"]

let r3 = parseMyArgs(@["file.txt"])
doAssert r3.verbose == false
```

---

### Режим GNU — совместимость с `getopt_long`

```nim
import std/parseopt

# GNU-стиль: только =, нет пробелов вокруг разделителя, значения могут начинаться с -
for kind, key, val in getopt(
    @["--output=result.txt", "--count=-5", "-v"],
    shortNoVal = {'v'},
    longNoVal = @["help"],
    mode = GnuMode):
  case kind
  of cmdShortOption, cmdLongOption:
    echo key, " => ", val
  of cmdArgument:
    echo "arg: ", key
  of cmdEnd: break
# output => result.txt
# count  => -5          ← значение начинается с -, в GnuMode это допустимо
# v      =>
```

---

## 12. Таблица быстрого доступа

### Типы и перечисления

| Имя | Вид | Описание |
|-----|-----|----------|
| `CmdLineKind` | `enum` | Тип распознанного токена |
| `cmdEnd` | значение | Конец аргументов |
| `cmdArgument` | значение | Позиционный аргумент |
| `cmdLongOption` | значение | Длинная опция `--foo` |
| `cmdShortOption` | значение | Короткая опция `-f` |
| `CliMode` | `enum` | Режим поведения парсера |
| `NimMode` | значение | Стандартный режим (по умолчанию) |
| `LaxMode` | значение | Гибкий POSIX-подобный режим |
| `GnuMode` | значение | GNU-совместимый режим |
| `OptParser` | `object` | Объект парсера |
| `OptParser.kind` | поле `CmdLineKind` | Тип последнего токена |
| `OptParser.key` | поле `string` | Имя опции / текст аргумента |
| `OptParser.val` | поле `string` | Значение опции (или `""`) |

### Процедуры и итераторы

| Имя | Сигнатура | Описание |
|-----|-----------|----------|
| `initOptParser` | `(string, ...) → OptParser` | Инициализация из строки |
| `initOptParser` | `(seq[string], ...) → OptParser` | Инициализация из списка |
| `next` | `(var OptParser)` | Разобрать следующий токен |
| `getopt` | `(var OptParser) → iter` | Итерировать существующий парсер |
| `getopt` | `(seq[string], ...) → iter` | Создать парсер и итерировать |
| `remainingArgs` | `(OptParser) → seq[string]` | Необработанные токены (список) |
| `cmdLineRest` | `(OptParser) → string` | Необработанные токены (строка) |

### Поддерживаемые синтаксисы по режимам

| Синтаксис | `NimMode` | `LaxMode` | `GnuMode` |
|-----------|:---------:|:---------:|:---------:|
| `--foo:bar` | ✅ | ✅ | ❌ |
| `--foo=bar` | ✅ | ✅ | ✅ |
| `--foo: bar` (пробел) | ✅ | ✅ | ❌ |
| `--foo =bar` (пробел) | ✅ | ✅ | ❌ |
| `--foo bar` (next-arg) | ✅\* | ✅\* | ✅\* |
| `-kval` (смежное) | ✅ | ✅ | ✅ |
| `-k val` (next-arg) | ❌ | ✅\* | ✅\* |
| `-k -10` (значение с `-`) | ❌ | ✅\* | ✅\* |
| `-abc` (группировка) | ✅ | ✅ | ✅ |

\* Требует непустых `shortNoVal`/`longNoVal`

---

*Документ составлен по исходному коду `std/parseopt` из стандартной библиотеки Nim. Совместим с Nim 2.x.*
