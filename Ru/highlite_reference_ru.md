# highlite — справочник модуля

> **Импорт:** `import std/highlite`
> **Область применения:** пословный (токенный) разбор исходного кода на нескольких
> языках программирования — Nim, C, C++, C#, Java, Python, а также YAML,
> Cmd/Console-сессий — для подсветки синтаксиса.

Модуль не строит дерево разбора и не проверяет код на корректность — он лишь
последовательно "нарезает" входную строку на токены (ключевое слово,
идентификатор, число, строковый литерал, комментарий, оператор и т.д.),
возвращая для каждого токена его класс `TokenClass` и границы в исходном
буфере. Общая конвенция модуля: один и тот же объект `GeneralTokenizer`
инициализируется один раз процедурой `initGeneralTokenizer`, а затем в цикле
вызывается `getNextToken`, которая на каждом шаге продвигает внутреннюю
позицию и заполняет поля `kind`, `start`, `length`. Конкретный язык при этом
передаётся отдельным параметром `lang: SourceLanguage` — это позволяет
переиспользовать один и тот же токенизатор для нескольких вложенных языков
(например, для `Cmd`-команды внутри `Console`-сессии).

---

## Оглавление

I. [Типы и константы](#типы-и-константы)
&nbsp;&nbsp;&nbsp;1. [`SourceLanguage`](#sourcelanguage)
&nbsp;&nbsp;&nbsp;2. [`TokenClass`](#tokenclass)
&nbsp;&nbsp;&nbsp;3. [`GeneralTokenizer`](#generaltokenizer)
&nbsp;&nbsp;&nbsp;4. [Таблицы соответствий](#таблицы-соответствий)

II. [Определение языка по строке](#определение-языка-по-строке)
&nbsp;&nbsp;&nbsp;1. [`getSourceLanguage`](#getsourcelanguage)

III. [Инициализация токенизатора](#инициализация-токенизатора)
&nbsp;&nbsp;&nbsp;1. [`initGeneralTokenizer` (из `cstring`)](#initgeneraltokenizer-из-cstring)
&nbsp;&nbsp;&nbsp;2. [`initGeneralTokenizer` (из `string`)](#initgeneraltokenizer-из-string)
&nbsp;&nbsp;&nbsp;3. [`deinitGeneralTokenizer`](#deinitgeneraltokenizer)

IV. [Получение токенов](#получение-токенов)
&nbsp;&nbsp;&nbsp;1. [`getNextToken`](#getnexttoken)
&nbsp;&nbsp;&nbsp;2. [`tokenize`](#tokenize)

V. [Практические рецепты](#практические-рецепты)
&nbsp;&nbsp;&nbsp;1. [Разложение строки кода на цветные фрагменты](#разложение-строки-кода-на-цветные-фрагменты)
&nbsp;&nbsp;&nbsp;2. [Подсчёт строк комментариев в файле](#подсчёт-строк-комментариев-в-файле)
&nbsp;&nbsp;&nbsp;3. [Извлечение всех идентификаторов и ключевых слов](#извлечение-всех-идентификаторов-и-ключевых-слов)
&nbsp;&nbsp;&nbsp;4. [Определение языка по расширению и подсветка "на лету"](#определение-языка-по-расширению-и-подсветка-на-лету)
&nbsp;&nbsp;&nbsp;5. [Подсветка команды с выводом (Console)](#подсветка-команды-с-выводом-console)

VI. [Краткая таблица](#краткая-таблица)

VII. [Сводка: какую процедуру выбрать](#сводка-какую-процедуру-выбрать)

---

## Типы и константы

### `SourceLanguage`

```nim
type
  SourceLanguage* = enum
    langNone, langNim, langCpp, langCsharp, langC, langJava,
    langYaml, langPython, langCmd, langConsole
```

**Что делает.** Перечисление поддерживаемых языков разбора. `langNone`
означает "язык не определён" и используется как значение по умолчанию для
непроинициализированного токенизатора; передавать его напрямую в
`getNextToken` нельзя (см. раздел о `getNextToken`).

**Разбор реализации.** Порядок значений в перечислении не случаен — он должен
совпадать поэлементно с порядком строк в константах `sourceLanguageToStr` и
`sourceLanguageToAlpha` (раздел "Таблицы соответствий"): значение enum
используется как индекс массива, поэтому любое рассогласование порядка молча
сломает сопоставление имени языка с его enum-значением.

- **Параметры:** тип не принимает параметров, это набор именованных констант.

```nim
var lang: SourceLanguage
lang = langPython
echo lang        # выводит langPython
echo ord(lang)    # выводит 7 — порядковый номер в перечислении
```

---

### `TokenClass`

```nim
type
  TokenClass* = enum
    gtEof, gtNone, gtWhitespace, gtDecNumber, gtBinNumber, gtHexNumber,
    gtOctNumber, gtFloatNumber, gtIdentifier, gtKeyword, gtStringLit,
    gtLongStringLit, gtCharLit, gtEscapeSequence,
    gtOperator, gtPunctuation, gtComment, gtLongComment, gtRegularExpression,
    gtTagStart, gtTagEnd, gtKey, gtValue, gtRawData, gtAssembler,
    gtPreprocessor, gtDirective, gtCommand, gtRule, gtHyperlink, gtLabel,
    gtReference, gtPrompt, gtProgramOutput, gtProgram, gtOption, gtOther
```

**Что делает.** Перечисляет все возможные классы токена, которые может
вернуть `getNextToken`. Не каждый язык использует все значения: например,
`gtTagStart`/`gtLabel`/`gtReference` встречаются только при разборе YAML, а
`gtProgram`/`gtOption`/`gtPrompt` — только при разборе `Cmd`/`Console`.
`gtEof` — признак конца буфера, им завершается любой цикл разбора; `gtNone` —
"токен неопределённого класса" (например, одиночный непонятный символ).

**Разбор реализации.** Общий для всех языков разбора набор классов позволяет
писать один и тот же код подсветки, не разветвляя его по языку: вызывающая
сторона реагирует на `TokenClass`, а не на то, какой именно `langNextToken`
его выдал.

- **Параметры:** тип не принимает параметров.

```nim
let классToken = gtKeyword
case классToken
of gtKeyword:
  echo "Это ключевое слово языка"
of gtComment, gtLongComment:
  echo "Это комментарий"
else:
  echo "Другой класс токена"
```

---

### `GeneralTokenizer`

```nim
type
  GeneralTokenizer* = object of RootObj
    kind*: TokenClass
    start*, length*: int
    buf: cstring
    pos: int
    state: TokenClass
    lang: SourceLanguage
```

**Что делает.** Хранит всё состояние процесса разбора одного буфера: указатель
на сам буфер (`buf`), текущую позицию чтения (`pos`), класс и границы
последнего найденного токена (`kind`, `start`, `length`), а также `state` —
внутреннее "продолженное" состояние разбора между вызовами (например,
"мы внутри многострочного комментария" или "внутри строкового литерала,
экранированный символ ещё не закончен").

**Разбор реализации.** Наружу (через `*`) открыты только три поля: `kind`,
`start`, `length` — это единственное, что нужно потребителю модуля после
каждого вызова `getNextToken`. Поля `buf`, `pos`, `state`, `lang` оставлены
приватными: они образуют внутренний конечный автомат построчного разбора
(state machine), и его переходы (например, "войти в строку — выйти по
экранированию — вернуться в строку") не предназначены для прямого чтения или
записи снаружи модуля.

- **Поля:**
  - `kind*: TokenClass` — класс токена, найденного последним вызовом `getNextToken`;
  - `start*, length*: int` — начало и длина найденного токена внутри исходного буфера (не строки-копии!);
  - `buf: cstring` (приватное) — сам разбираемый буфер;
  - `pos: int` (приватное) — текущая позиция чтения;
  - `state: TokenClass` (приватное) — "подвешенное" состояние между токенами;
  - `lang: SourceLanguage` (приватное) — язык, с которым был вызван последний `getNextToken`.

```nim
var g: GeneralTokenizer
initGeneralTokenizer(g, "let x = 1")
getNextToken(g, langNim)
echo g.kind    # выводит gtKeyword ("let" — ключевое слово)
echo g.start   # выводит 0
echo g.length  # выводит 3
```

---

### Таблицы соответствий

```nim
const
  sourceLanguageToStr*: array[SourceLanguage, string] = ["none",
    "Nim", "C++", "C#", "C", "Java", "Yaml", "Python", "Cmd", "Console"]
  sourceLanguageToAlpha*: array[SourceLanguage, string] = ["none",
    "Nim", "cpp", "csharp", "C", "Java", "Yaml", "Python", "Cmd", "Console"]
  tokenClassToStr*: array[TokenClass, string] = ["Eof", "None", "Whitespace",
    "DecNumber", "BinNumber", "HexNumber", "OctNumber", "FloatNumber",
    "Identifier", "Keyword", "StringLit", "LongStringLit", "CharLit",
    "EscapeSequence", "Operator", "Punctuation", "Comment", "LongComment",
    "RegularExpression", "TagStart", "TagEnd", "Key", "Value", "RawData",
    "Assembler", "Preprocessor", "Directive", "Command", "Rule", "Hyperlink",
    "Label", "Reference", "Prompt", "ProgramOutput",
    "program", "option", "Other"]
```

**Что делает.** Три массива, индексируемые самими перечислениями (а не
целыми числами), дающие человекочитаемое строковое имя для каждого значения
`SourceLanguage` или `TokenClass`. `sourceLanguageToStr` даёт "красивое"
отображаемое имя ("C++", "C#"), а `sourceLanguageToAlpha` — вариант,
состоящий только из букв и пригодный, например, как имя CSS-класса или
как часть RST/HTML-роли ("cpp", "csharp").

**Разбор реализации.** Поскольку массивы объявлены как `array[SourceLanguage, string]`
и `array[TokenClass, string]`, компилятор Nim сам проверяет на этапе
компиляции, что число строк в инициализаторе в точности совпадает с числом
значений перечисления — рассинхронизация (забытая или лишняя строка) не
скомпилируется, а не проявится как баг в рантайме.

- **Параметры:** константы, индексируются значением соответствующего перечисления.

```nim
echo sourceLanguageToStr[langCsharp]    # выводит "C#"
echo sourceLanguageToAlpha[langCsharp]  # выводит "csharp"
echo tokenClassToStr[gtKeyword]         # выводит "Keyword"
```

---

## Определение языка по строке

### `getSourceLanguage`

```nim
proc getSourceLanguage*(name: string): SourceLanguage
```

**Что делает.** По произвольно набранному имени языка (не чувствительно ни к
регистру, ни к дефисам/подчёркиваниям — сравнение идёт через `cmpIgnoreStyle`)
возвращает соответствующее значение `SourceLanguage`. Если имя не найдено ни
среди "красивых" (`sourceLanguageToStr`), ни среди "буквенных"
(`sourceLanguageToAlpha`) вариантов, возвращается `langNone`.

**Разбор реализации.** Поиск линейный (O(n) по числу языков, а их всего
десяток, так что это несущественно) и последовательно проверяет оба массива
соответствий для каждого языка — сначала "красивое" имя, затем "буквенное".
Перебор начинается не с `langNone` (`succ(low(SourceLanguage))`), поскольку
`langNone` — это как раз результат "ничего не найдено", а не то, что можно
ввести как имя.

- **Параметры:**
  - `name: string` — имя языка в произвольном регистре и написании ("C++", "c++", "CPP", "cpp" — все распознаются).

```nim
echo getSourceLanguage("Nim")     # выводит langNim
echo getSourceLanguage("c++")     # выводит langCpp — распознано через sourceLanguageToAlpha
echo getSourceLanguage("C#")      # выводит langCsharp
echo getSourceLanguage("паскаль") # выводит langNone — язык не поддерживается
```

Практический сценарий — выбор языка подсветки по расширению файла показан в
разделе "Практические рецепты" (рецепт 4).

---

## Инициализация токенизатора

### `initGeneralTokenizer` (из `cstring`)

```nim
proc initGeneralTokenizer*(g: var GeneralTokenizer, buf: cstring)
```

**Что делает.** Обнуляет всё состояние токенизатора `g` и связывает его с
буфером `buf`. После вызова токенизатор готов к первому вызову
`getNextToken`. Буфер не копируется — `g` лишь хранит ссылку на переданный
`cstring`, поэтому он должен оставаться живым (не быть освобождён или
переиспользован) на всё время работы с токенизатором.

**Разбор реализации.** `kind` и `state` устанавливаются в `low(TokenClass)`
(то есть `gtEof`), `lang` — в `low(SourceLanguage)` (то есть `langNone`), а
`start`/`length`/`pos` обнуляются. Работа именно с `cstring`, а не со `string`,
важна для внутреннего разбора: процедуры вида `nimNextToken` читают буфер
посимвольно по индексу вплоть до нулевого байта-терминатора, что было бы
небезопасно для среза Nim-строки без гарантированного завершающего `\0`.

- **Параметры:**
  - `g: var GeneralTokenizer` — токенизатор, который будет инициализирован (изменяемый);
  - `buf: cstring` — разбираемый буфер; должен жить не меньше, чем сам `g`.

```nim
var g: GeneralTokenizer
let буфер: cstring = "proc main() = discard"
initGeneralTokenizer(g, буфер)
getNextToken(g, langNim)
echo g.kind  # выводит gtKeyword ("proc")
```

---

### `initGeneralTokenizer` (из `string`)

```nim
proc initGeneralTokenizer*(g: var GeneralTokenizer, buf: string)
```

**Что делает.** Удобная обёртка над предыдущей перегрузкой: принимает
обычную Nim-строку и сама приводит её к `cstring` перед вызовом.
Для подавляющего большинства случаев использования модуля это
и есть тот вариант, который нужен вызывающей стороне.

**Разбор реализации.** Реализация — это буквально один делегирующий вызов:
`initGeneralTokenizer(g, cstring(buf))`. Здесь важно понимать неявный
контракт Nim: преобразование `string` в `cstring` не выделяет новую память —
оно действительно только пока жива исходная строка `buf`; если строка выйдет
из области видимости раньше токенизатора, дальнейшие вызовы `getNextToken`
станут обращаться к освобождённой памяти.

- **Параметры:**
  - `g: var GeneralTokenizer` — токенизатор, который будет инициализирован (изменяемый);
  - `buf: string` — разбираемый исходный код; должна пережить весь цикл разбора.

```nim
var g: GeneralTokenizer
var исходник = "for i in 0..10: echo i"
initGeneralTokenizer(g, исходник)
getNextToken(g, langNim)
echo g.kind  # выводит gtKeyword ("for")

# граничный случай — пустая строка
var пустой: GeneralTokenizer
initGeneralTokenizer(пустой, "")
getNextToken(пустой, langNim)
echo пустой.kind  # выводит gtEof — токенов нет, сразу конец буфера
```

---

### `deinitGeneralTokenizer`

```nim
proc deinitGeneralTokenizer*(g: var GeneralTokenizer)
```

**Что делает.** Формально освобождает ресурсы, связанные с токенизатором.
На практике текущая реализация ничего не делает (`discard`) — процедура
существует ради симметрии с `initGeneralTokenizer` и как точка расширения на
будущее (если у типа появятся ресурсы, требующие явного освобождения, —
например, управляемая память вне GC).

**Разбор реализации.** Наличие "пустого" деинициализатора — обычная практика
Nim-библиотек: вызывающий код, который вызывает `deinitGeneralTokenizer`
после использования, продолжит работать без изменений даже если в будущей
версии модуля процедура перестанет быть пустой.

- **Параметры:**
  - `g: var GeneralTokenizer` — деинициализируемый токенизатор.

```nim
var g: GeneralTokenizer
initGeneralTokenizer(g, "x")
getNextToken(g, langNim)
deinitGeneralTokenizer(g)  # на сегодня — не более чем формальность
```

---

## Получение токенов

### `getNextToken`

```nim
proc getNextToken*(g: var GeneralTokenizer, lang: SourceLanguage)
```

**Что делает.** Главная рабочая процедура модуля: разбирает ровно один
следующий токен, начиная с текущей позиции `g.pos`, и записывает результат в
`g.kind`, `g.start`, `g.length`, одновременно продвигая `g.pos` за пределы
найденного токена. Вызывать её нужно в цикле до тех пор, пока `g.kind` не
станет равным `gtEof`. Передача `lang = langNone` — ошибка использования: она
вызывает `assert false` (аварийное завершение при включённых assert'ах).

**Разбор реализации.** Процедура — это диспетчер (`case lang of ...`),
который делегирует фактический разбор одной из внутренних,
непубличных (без `*`) процедур конкретного языка:
`nimNextToken`, `clikeNextToken` (используется для C/C++/C#/Java через
тонкие обёртки `cNextToken`/`cppNextToken`/`csharpNextToken`/`javaNextToken`,
которые лишь передают разный список ключевых слов и флаги вроде
"есть ли препроцессор" или "поддерживаются ли вложенные комментарии"),
`yamlNextToken`, `pythonNextToken` (тонкая обёртка над `nimNextToken` с
Python-овским списком ключевых слов) и `cmdNextToken` (используется и для
`Cmd`, и — с флагом приглашения `$` — для `Console`). Общая идея всех этих
внутренних процедур одинакова и напоминает работу светофора с "памятью":
если предыдущий вызов оставил незакрытым многострочный литерал или
комментарий, это записано в `g.state`, и следующий вызов первым делом
проверяет именно `state`, а не читает символ с нуля — так реализуется разбор
конструкций, которые могут занимать несколько вызовов подряд (например,
экранированная последовательность внутри строки как отдельный токен
`gtEscapeSequence`).

- **Параметры:**
  - `g: var GeneralTokenizer` — токенизатор с уже инициализированным буфером (изменяемый);
  - `lang: SourceLanguage` — язык, по правилам которого разбирается следующий токен; не должен быть `langNone`.

```nim
var g: GeneralTokenizer
initGeneralTokenizer(g, "let x = 42 # ответ")
while true:
  getNextToken(g, langNim)
  if g.kind == gtEof:
    break
  echo tokenClassToStr[g.kind]
# выводит по очереди:
# Keyword, Whitespace, Identifier, Whitespace, Operator, Whitespace,
# DecNumber, Whitespace, Comment

# ошибочный случай — язык не задан
var g2: GeneralTokenizer
initGeneralTokenizer(g2, "x")
doAssertRaises(AssertionDefect):
  getNextToken(g2, langNone)
```

---

### `tokenize`

```nim
proc tokenize*(text: string, lang: SourceLanguage): seq[(string, TokenClass)]
```

**Что делает.** Высокоуровневая обёртка над связкой
"`initGeneralTokenizer` + цикл `getNextToken`": сразу разбирает всю строку
`text` целиком и возвращает готовую последовательность пар
"(подстрока токена, класс токена)". Удобна, когда не нужен пошаговый
контроль над разбором (как в примере выше), а нужен просто список токенов
целиком.

**Разбор реализации.** Внутри процедура хранит только `prevPos` — позицию
конца предыдущего токена — и на каждой итерации вырезает срез
`text[prevPos ..< g.pos]`, то есть именно ту подстроку исходного текста,
которая соответствует токену, найденному последним `getNextToken`. Заметьте:
здесь, в отличие от прямой работы с `GeneralTokenizer`, вызывающему коду не
нужно самому вычислять срез по `g.start`/`g.length` — это уже сделано.
Сложность — O(n) по длине текста, поскольку каждый символ входного буфера
посещается ровно в одном токене.

- **Параметры:**
  - `text: string` — разбираемый исходный код целиком;
  - `lang: SourceLanguage` — язык разбора; не должен быть `langNone`.

```nim
let токены = tokenize("var y = 1", langNim)
for (текстТокена, классТокена) in токены:
  echo "'", текстТокена, "' -> ", tokenClassToStr[классТокена]
# выводит:
# 'var' -> Keyword
# ' ' -> Whitespace
# 'y' -> Identifier
# ' ' -> Whitespace
# '=' -> Operator
# ' ' -> Whitespace
# '1' -> DecNumber

# граничный случай — пустая строка
echo len(tokenize("", langNim))  # выводит 0 — токенов нет вовсе
```

---

## Практические рецепты

### Разложение строки кода на цветные фрагменты

Типичная задача подсветки: сопоставить каждому токену CSS-класс и собрать
готовый HTML.

```nim
proc подсветитьВHtml(код: string, lang: SourceLanguage): string =
  var g: GeneralTokenizer
  initGeneralTokenizer(g, код)
  result = ""
  while true:
    getNextToken(g, lang)
    if g.kind == gtEof:
      break
    let фрагмент = substr(код, g.start, g.start + g.length - 1)
    add(result, "<span class=\"tok-")
    add(result, tokenClassToStr[g.kind])
    add(result, "\">")
    add(result, фрагмент)
    add(result, "</span>")

echo подсветитьВHtml("if x: echo x", langNim)
```

---

### Подсчёт строк комментариев в файле

Комбинация `tokenize` и подсчёта символов перевода строки внутри токенов
класса `gtComment`/`gtLongComment`.

```nim
proc подсчитатьСтрокиКомментариев(текст: string, lang: SourceLanguage): int =
  let токены = tokenize(текст, lang)
  result = 0
  for (фрагмент, класс) in токены:
    if класс in {gtComment, gtLongComment}:
      inc(result, 1 + count(фрагмент, '\n'))

echo подсчитатьСтрокиКомментариев("# однострочный\n# ещё один\nx = 1", langPython)
```

---

### Извлечение всех идентификаторов и ключевых слов

Полезно для построения индекса имён (например, для "перехода к
определению" в редакторе).

```nim
proc собратьИмена(текст: string, lang: SourceLanguage): tuple[идентификаторы, ключевые: seq[string]] =
  let токены = tokenize(текст, lang)
  for (фрагмент, класс) in токены:
    case класс
    of gtIdentifier: add(result.идентификаторы, фрагмент)
    of gtKeyword: add(result.ключевые, фрагмент)
    else: discard

let (идент, ключ) = собратьИмена("proc foo(bar: int) = discard", langNim)
echo идент  # выводит @["foo", "bar", "int"]
echo ключ   # выводит @["proc", "discard"]
```

---

### Определение языка по расширению и подсветка "на лету"

Комбинация `getSourceLanguage` с обычной строковой логикой определения
расширения файла.

```nim
proc языкПоИмениФайла(путь: string): SourceLanguage =
  let части = split(путь, '.')
  if len(части) < 2:
    return langNone
  case части[^1]
  of "nim": getSourceLanguage("Nim")
  of "py": getSourceLanguage("Python")
  of "cpp", "cc", "cxx": getSourceLanguage("C++")
  of "cs": getSourceLanguage("C#")
  of "java": getSourceLanguage("Java")
  of "yaml", "yml": getSourceLanguage("Yaml")
  else: langNone

echo языкПоИмениФайла("main.cpp")     # выводит langCpp
echo языкПоИмениФайла("script.rb")    # выводит langNone — язык не поддерживается
```

---

### Подсветка команды с выводом (Console)

`Console` — язык для интерактивных сессий: строки с `$`-приглашением
разбираются как команда (`Cmd`), остальные — как вывод программы.

```nim
let сессия = """$ echo "привет"
привет"""
let токены = tokenize(сессия, langConsole)
for (фрагмент, класс) in токены:
  if класс == gtProgramOutput:
    echo "вывод программы: ", фрагмент
  elif класс == gtPrompt:
    echo "приглашение: ", фрагмент
```

---

## Краткая таблица

| Задача                                                          | Изменяет `g` | Возвращает                          |
|------------------------------------------------------------------|:---:|--------------------------------------|
| Определить `SourceLanguage` по произвольному имени               | —   | `SourceLanguage`                     |
| Начать разбор буфера (`cstring`)                                  | да  | (только инициализирует `g`)          |
| Начать разбор буфера (`string`)                                   | да  | (только инициализирует `g`)          |
| Формально завершить работу с токенизатором                       | да  | (не имеет эффекта в текущей версии)  |
| Разобрать один следующий токен, зная язык                        | да  | (результат — в полях `g.kind/start/length`) |
| Разобрать всю строку целиком и получить список токенов сразу      | нет (свой внутренний `g`) | `seq[(string, TokenClass)]` |
| Получить строковое имя языка или класса токена для вывода         | —   | `string` (через `sourceLanguageToStr`/`sourceLanguageToAlpha`/`tokenClassToStr`) |

---

## Сводка: какую процедуру выбрать

- Нужно узнать `SourceLanguage` по строке, введённой пользователем или взятой
  из конфигурации → `getSourceLanguage`.
- Нужно разобрать код целиком и сразу получить готовый список токенов,
  не заботясь о цикле и срезах → `tokenize`.
- Нужен пошаговый разбор с полным контролем (например, чтобы прервать
  разбор раньше конца или обрабатывать токены по одному без накопления
  всего списка в памяти) → `initGeneralTokenizer` + цикл `getNextToken`
  до `gtEof`.
- Буфер уже существует как `cstring` (например, получен из C-интеропа) →
  `initGeneralTokenizer(g, buf: cstring)`; в остальных случаях (обычная
  Nim-строка) — перегрузка с `string`.
- Нужно вывести имя языка или класса токена человеку (лог, отладка, CSS-класс)
  → `sourceLanguageToStr`/`sourceLanguageToAlpha` или `tokenClassToStr`,
  а не `$lang`/`$kind` (последнее даст программное имя enum-значения вроде
  `langCpp`, а не "C++").
- Нужно быстро проверить/показать поддерживаемые языки → `SourceLanguage`
  и `sourceLanguageToStr` вместе как справочная таблица (раздел "Таблицы
  соответствий").
