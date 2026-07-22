# parsecfg — справочник модуля

> **Импорт:** `import std/parsecfg`
> **Область применения:** разбор и запись конфигурационных файлов формата, близкого к Windows `.ini`, но поддерживающего строковые литералы Nim (обычные, raw- и тройные строки).

Модуль решает две связанные, но разные задачи. Во-первых, это низкоуровневый
потоковый (событийный) парсер: `CfgParser` читает входной поток токен за
токеном и через `next` отдаёт по одному событию (`CfgEvent`) — начало секции,
пара ключ-значение, опция вида `--ключ:значение` или ошибка. Этот слой не
хранит ничего в памяти целиком и подходит для больших файлов или для
собственной логики обработки.

Во-вторых, поверх событийного парсера построен табличный API: тип `Config`
(таблица таблиц) и процедуры `loadConfig`/`writeConfig`/`getSectionValue`/
`setSectionKey`, которые загружают весь файл в память и дают удобный доступ
"секция → ключ → значение". Именно этот слой стоит использовать, если не
нужен полный контроль над процессом разбора.

Общая конвенция модуля: у ключей вида `--ключ:значение` (записываются в
файле через двойное тире) в табличном API имя ключа хранится с префиксом
`"--"`, чтобы отличать их от обычных пар `ключ=значение`. Пустая строка `""`
как имя секции обозначает "общую" секцию — то, что записано в файле до
первого заголовка `[секция]`.

---

## Оглавление

I. [Типы и вспомогательные средства](#типы-и-вспомогательные-средства)
   1. [`CfgEventKind`](#cfgeventkind)
   2. [`CfgEvent`](#cfgevent)
   3. [`CfgParser`](#cfgparser)
   4. [`Config`](#config)
II. [Потоковый парсер — низкоуровневый событийный API](#потоковый-парсер--низкоуровневый-событийный-api)
   1. [`open`](#open)
   2. [`next`](#next)
   3. [`close`](#close)
   4. [`getColumn`, `getLine`, `getFilename`](#getcolumn-getline-getfilename)
   5. [`errorStr`, `warningStr`, `ignoreMsg`](#errorstr-warningstr-ignoremsg)
III. [Табличный API — загрузка и создание конфигурации](#табличный-api--загрузка-и-создание-конфигурации)
   1. [`newConfig`](#newconfig)
   2. [`loadConfig` (из потока)](#loadconfig-из-потока)
   3. [`loadConfig` (из файла)](#loadconfig-из-файла)
   4. [`getSectionValue`](#getsectionvalue)
   5. [`setSectionKey`](#setsectionkey)
   6. [`delSection`](#delsection)
   7. [`delSectionKey`](#delsectionkey)
   8. [`sections` (итератор)](#sections-итератор)
IV. [Запись и сериализация конфигурации](#запись-и-сериализация-конфигурации)
   1. [`writeConfig` (в поток)](#writeconfig-в-поток)
   2. [`writeConfig` (в файл)](#writeconfig-в-файл)
   3. [`` `$` ``](#-)
V. [Практические рецепты](#практические-рецепты)
   1. [Создание конфигурации с нуля и сохранение в файл](#создание-конфигурации-с-нуля-и-сохранение-в-файл)
   2. [Чтение с значениями по умолчанию](#чтение-с-значениями-по-умолчанию)
   3. [Обновление существующего файла (round-trip)](#обновление-существующего-файла-round-trip)
   4. [Перебор всех секций и ключей](#перебор-всех-секций-и-ключей)
   5. [Собственная обработка событий с игнорированием неизвестного](#собственная-обработка-событий-с-игнорированием-неизвестного)
VI. [Краткая таблица](#краткая-таблица)
VII. [Сводка: какую процедуру выбрать](#сводка-какую-процедуру-выбрать)

---

## Типы и вспомогательные средства

### `CfgEventKind`

```nim
type
  CfgEventKind* = enum
    cfgEof,          ## конец файла достигнут
    cfgSectionStart, ## распознан заголовок `[section]`
    cfgKeyValuePair, ## распознана пара `key=value`
    cfgOption,       ## распознана опция командной строки `--key=value`
    cfgError         ## во время разбора произошла ошибка
```

**Что делает.** Перечисление видов событий, которые может вернуть `next`.
Это "тег" вариантного объекта `CfgEvent` — по значению `kind` определяется,
какие поля события заполнены.

- **Список значений:**
  - `cfgEof` — поток исчерпан, разбор окончен;
  - `cfgSectionStart` — встречен заголовок секции `[имя]`;
  - `cfgKeyValuePair` — встречена обычная пара `ключ=значение` или `ключ:значение`;
  - `cfgOption` — встречена запись вида `--ключ:значение` (без секции она относится к "общей" части файла);
  - `cfgError` — синтаксическая ошибка; разбор не останавливается исключением, событие просто сообщает об этом.

---

### `CfgEvent`

```nim
type
  CfgEvent* = object of RootObj
    case kind*: CfgEventKind
    of cfgEof: nil
    of cfgSectionStart:
      section*: string
    of cfgKeyValuePair, cfgOption:
      key*, value*: string
    of cfgError:
      msg*: string
```

**Что делает.** Вариантный объект-событие, возвращаемый `next`. Набор
доступных полей зависит от `kind`: у `cfgSectionStart` есть только `section`,
у `cfgKeyValuePair`/`cfgOption` — пара `key`/`value`, у `cfgError` — `msg`.
Попытка обратиться к полю, не относящемуся к текущему `kind` (например, к
`section` у события с `kind == cfgError`), — ошибка на этапе выполнения,
как у любого вариантного объекта Nim.

- **Список параметров/полей:**
  - `kind: CfgEventKind` — определяет активный вариант;
  - `section: string` — имя секции (только для `cfgSectionStart`);
  - `key, value: string` — ключ и значение (для `cfgKeyValuePair`/`cfgOption`); `value == ""`, если значение не было указано в файле;
  - `msg: string` — текст ошибки, уже отформатированный с именем файла, строкой и столбцом (только для `cfgError`).

**Пример:**

```nim
import std/streams

var p: CfgParser
open(p, newStringStream("[Package]\nname=hello\n"), "example.ini")
while true:
  var e = next(p)
  case e.kind
  of cfgEof: break
  of cfgSectionStart:
    echo "секция: " & e.section  # выводит: секция: Package
  of cfgKeyValuePair:
    echo e.key & " = " & e.value  # выводит: name = hello
  of cfgOption:
    echo "опция: " & e.key
  of cfgError:
    echo e.msg
close(p)
```

---

### `CfgParser`

```nim
type
  CfgParser* = object of BaseLexer
```

**Что делает.** Непрозрачный объект-парсер: наследуется от `BaseLexer`
(`std/lexbase`) и хранит текущий буфер ввода, позицию чтения, номер строки
и предпрочитанный (lookahead) токен. Наружу поля не выставлены — доступ к
состоянию идёт только через процедуры `getLine`, `getColumn`, `getFilename`.

**Разбор реализации.** Парсер устроен как разборщик с одним токеном
предпросмотра (lookahead 1): после `open` и после каждого `next` в
парсере уже "лежит" следующий токен, готовый к анализу. Это упрощает
`next` — не нужно откатываться назад при определении, что за конструкция
началась (пара ключ-значение или заголовок секции), достаточно посмотреть
на один токен вперёд.

- **Список параметров:** нет публичных полей; объект объявляется как `var`-переменная и передаётся по ссылке во все процедуры модуля (`open`, `next`, `close` и т.д.).

---

### `Config`

```nim
type
  Config* = OrderedTableRef[string, OrderedTableRef[string, string]]
```

**Что делает.** Табличное представление всего конфигурационного файла в
памяти: внешняя упорядоченная таблица отображает имя секции на внутреннюю
упорядоченную таблицу "ключ → значение". Порядок вставки секций и ключей
сохраняется — это важно при последующей записи через `writeConfig`, чтобы
файл не "перемешивался" при каждом сохранении.

- **Список параметров:**
  - внешний ключ — имя секции (`""` — общая секция, то, что до первого `[...]`);
  - внутренний ключ — имя параметра внутри секции (для `--опций` хранится вместе с префиксом `"--"`);
  - значение — строка (модуль не выполняет преобразование типов — все значения хранятся как текст).

---

## Потоковый парсер — низкоуровневый событийный API

### `open`

```nim
proc open*(c: var CfgParser, input: Stream, filename: string, lineOffset = 0)
```

**Что делает.** Инициализирует парсер `c` заданным входным потоком.
`filename` используется только в текстах сообщений об ошибках/предупреждениях,
на сам разбор не влияет. `lineOffset` сдвигает нумерацию строк в этих же
сообщениях — полезно, если разбираемый текст — это вставка внутри другого
файла и нужно, чтобы номера строк соответствовали исходному документу.

**Разбор реализации.** Вызывает `lexbase.open` для настройки буферизованного
чтения потока, сбрасывает предыдущее состояние токена (`tkInvalid`, пустой
литерал), прибавляет `lineOffset` к счётчику строк и сразу считывает первый
токен вызовом внутреннего `rawGetTok`. Именно поэтому сразу после `open`
парсер готов к первому вызову `next` — токен предпросмотра уже на месте.

- **Список параметров:**
  - `c: var CfgParser` — инициализируемый парсер;
  - `input: Stream` — уже открытый входной поток;
  - `filename: string` — имя файла для сообщений об ошибках;
  - `lineOffset: int` — сдвиг нумерации строк, по умолчанию `0`.

**Пример:**

```nim
import std/streams

var f = newFileStream("example.ini", fmRead)
doAssert f != nil, "не удалось открыть файл"
var p: CfgParser
open(p, f, "example.ini")
close(p)
```

---

### `next`

```nim
proc next*(c: var CfgParser): CfgEvent
```

**Что делает.** Возвращает очередное событие разбора и продвигает парсер
вперёд. Это единственная процедура, которая управляет ходом разбора —
её вызывают в цикле до получения `cfgEof`. При встрече токена `[` парсер
ожидает символ и закрывающую `]`; если что-то из этого не совпало —
возвращается `cfgError`, но разбор не прерывается исключением и можно
продолжать звать `next` дальше.

**Разбор реализации.** Решение о виде события принимается по текущему
токену предпросмотра `c.tok.kind`:

- `tkEof` → `cfgEof`;
- `tkDashDash` (`--`) → читается следующий токен и результат строится через
  вспомогательную `getKeyValPair` с `kind = cfgOption`;
- `tkSymbol` → тоже `getKeyValPair`, но с `kind = cfgKeyValuePair`;
- `tkBracketLe` (`[`) → ожидается `tkSymbol`, затем `tkBracketRi` (`]`);
  несовпадение на любом из этих шагов даёт `cfgError` с указанием, что
  ожидалось;
- любой другой токен (`tkInvalid`, `tkEquals`, `tkColon`, `tkBracketRi` вне
  контекста) → `cfgError` "invalid token".

Общий приём — читать пару `ключ` + необязательный разделитель (`=` или `:`)
+ `значение` в `getKeyValPair`: если после ключа не следует `=`/`:`, значение
остаётся пустой строкой (`value == ""`), что и описано в документации поля
`CfgEvent.value`.

- **Список параметров:** `c: var CfgParser` — парсер, из которого извлекается событие.

**Пример (граничный случай — ключ без значения):**

```nim
import std/streams

var p: CfgParser
open(p, newStringStream("key_without_value\n"), "[stream]")
let e = next(p)
doAssert e.kind == cfgKeyValuePair
doAssert e.key == "key_without_value"
doAssert e.value == ""  # значение не указано — остаётся пустым
close(p)
```

**Пример (ошибочный случай):**

```nim
import std/streams

var p: CfgParser
open(p, newStringStream("[section\n"), "[stream]")  # нет закрывающей ]
let e = next(p)
doAssert e.kind == cfgError
echo e.msg  # выводит отформатированное сообщение с именем файла, строкой и столбцом
close(p)
```

---

### `close`

```nim
proc close*(c: var CfgParser)
```

**Что делает.** Закрывает парсер и связанный с ним входной поток. Вызывается
один раз после того, как `next` вернул `cfgEof` (или разбор прерван
досрочно).

- **Список параметров:** `c: var CfgParser`.

---

### `getColumn`, `getLine`, `getFilename`

```nim
proc getColumn*(c: CfgParser): int
proc getLine*(c: CfgParser): int
proc getFilename*(c: CfgParser): string
```

**Что делает.** Три простых геттера текущего положения парсера: столбец,
строка и имя файла, переданное в `open`. Используются в основном для
собственных сообщений об ошибках — стандартные `errorStr`/`warningStr` уже
используют их внутри себя.

- **Список параметров:** `c: CfgParser` (без `var` — только чтение состояния) во всех трёх случаях.

**Пример:**

```nim
import std/streams

var p: CfgParser
open(p, newStringStream("a=b\n"), "cfg.ini")
discard next(p)
echo getFilename(p) & ":" & $getLine(p) & ":" & $getColumn(p)
close(p)
```

---

### `errorStr`, `warningStr`, `ignoreMsg`

```nim
proc errorStr*(c: CfgParser, msg: string): string
proc warningStr*(c: CfgParser, msg: string): string
proc ignoreMsg*(c: CfgParser, e: CfgEvent): string
```

**Что делает.** Форматируют текстовое сообщение вида
`имя_файла(строка, столбец) Error: текст` (или `Warning:` соответственно),
используя текущую позицию парсера. `ignoreMsg` — специализированный
вариант для случая, когда вызывающий код решил проигнорировать событие
(например, неизвестную секцию или неподдерживаемую опцию): формулировка
сообщения зависит от `e.kind` — "section ignored", "key ignored", "command
ignored"; для `cfgError` возвращается сам текст ошибки, для `cfgEof` —
пустая строка.

- **Список параметров:**
  - `errorStr`/`warningStr`: `c: CfgParser`, `msg: string` — произвольный текст сообщения;
  - `ignoreMsg`: `c: CfgParser`, `e: CfgEvent` — событие, которое игнорируется.

**Пример:** см. рецепт [«Собственная обработка событий с игнорированием неизвестного»](#собственная-обработка-событий-с-игнорированием-неизвестного) ниже — это основной сценарий использования `ignoreMsg`.

---

## Табличный API — загрузка и создание конфигурации

### `newConfig`

```nim
proc newConfig*(): Config
```

**Что делает.** Создаёт пустую таблицу конфигурации — точку отсчёта для
программного построения файла настроек (в противоположность его чтению
с диска).

- **Список параметров:** нет.

**Пример:**

```nim
var dict = newConfig()
doAssert len(dict) == 0
```

---

### `loadConfig` (из потока)

```nim
proc loadConfig*(stream: Stream, filename: string = "[stream]"): Config
```

**Что делает.** Читает весь поток через внутренний `CfgParser` и строит
`Config` целиком в памяти, группируя пары ключ-значение по последней
встреченной секции.

**Разбор реализации.** Внутри — тот же цикл `while true: next(p)`, что и в
примерах для низкоуровневого API, только результат не печатается, а
накапливается: переменная `curSection` хранит имя последней секции
(изначально `""` — общая секция); при `cfgKeyValuePair` пара кладётся во
внутреннюю таблицу текущей секции; при `cfgOption` — туда же, но ключ
получает префикс `"--"`, чтобы не путаться с обычными ключами при
последующей записи. Важная особенность: при `cfgError` цикл просто
прерывается (`break`) — исключение не поднимается, и до места ошибки файл
будет загружен частично, без какого-либо предупреждения об этом.

- **Список параметров:**
  - `stream: Stream` — открытый поток с содержимым конфигурации;
  - `filename: string` — имя для сообщений об ошибках, по умолчанию `"[stream]"`.

**Пример:**

```nim
import std/streams

let dict = loadConfig(newStringStream("[Package]\nname=hello\n"))
doAssert getSectionValue(dict, "Package", "name") == "hello"
```

---

### `loadConfig` (из файла)

```nim
proc loadConfig*(filename: string): Config
```

**Что делает.** То же самое, но принимает имя файла на диске, а не готовый
поток.

**Разбор реализации.** При обычном исполнении открывает файл (`open`,
`fmRead`), оборачивает его в `FileStream` и делегирует в вариант с потоком,
закрывая поток по завершении (`defer`). При компиляции для `nimvm`
(например, в NimScript) используется обходной путь: файл целиком читается
через `readFile` в строку и оборачивается в `StringStream`, поскольку
низкоуровневый `open` потока через `{.importc.}` в NimScript недоступен.

- **Список параметров:** `filename: string` — путь к файлу конфигурации.

**Пример:**

```nim
let dict = loadConfig("config.ini")
echo getSectionValue(dict, "Package", "name")
```

---

### `getSectionValue`

```nim
proc getSectionValue*(dict: Config, section, key: string, defaultVal = ""): string
```

**Что делает.** Возвращает значение ключа `key` в секции `section`; если
секции или ключа нет — возвращает `defaultVal` вместо ошибки или
исключения. Это основной способ безопасного чтения значений из уже
загруженной конфигурации.

- **Список параметров:**
  - `dict: Config` — таблица конфигурации;
  - `section: string` — имя секции (`""` — общая секция);
  - `key: string` — имя ключа;
  - `defaultVal: string` — значение по умолчанию, если ключ не найден; по умолчанию `""`.

**Пример:**

```nim
let dict = loadConfig(newStringStream("[Package]\nname=hello\n"))
doAssert getSectionValue(dict, "Package", "name") == "hello"
doAssert getSectionValue(dict, "Package", "version", "0.1.0") == "0.1.0"  # ключа нет — вернулось значение по умолчанию
```

---

### `setSectionKey`

```nim
proc setSectionKey*(dict: var Config, section, key, value: string)
```

**Что делает.** Устанавливает значение ключа в указанной секции. Если
секции ещё не существует, она создаётся автоматически — вызывающему коду
не нужно заранее проверять её наличие.

- **Список параметров:**
  - `dict: var Config` — изменяемая таблица конфигурации;
  - `section, key, value: string` — секция, ключ и новое значение.

**Пример:**

```nim
var dict = newConfig()
setSectionKey(dict, "Package", "name", "hello")
setSectionKey(dict, "Package", "--threads", "on")  # ключ-опция командной строки
doAssert getSectionValue(dict, "Package", "name") == "hello"
```

---

### `delSection`

```nim
proc delSection*(dict: var Config, section: string)
```

**Что делает.** Удаляет секцию целиком вместе со всеми её ключами. Если
секции с таким именем нет — ничего не происходит (тихий no-op, как у
`del` обычных таблиц Nim).

- **Список параметров:** `dict: var Config`, `section: string`.

**Пример:**

```nim
var dict = loadConfig(newStringStream("[Author]\nname=nim-lang\n"))
delSection(dict, "Author")
doAssert getSectionValue(dict, "Author", "name", "нет автора") == "нет автора"
```

---

### `delSectionKey`

```nim
proc delSectionKey*(dict: var Config, section, key: string)
```

**Что делает.** Удаляет один ключ внутри секции. Если после удаления в
секции не осталось ни одного ключа, удаляется и сама секция — так таблица
не засоряется пустыми записями.

**Разбор реализации.** Перед удалением проверяется `dict[section].len == 1`:
если удаляемый ключ был единственным, вызывается `dict.del(section)`
(удаление всей секции), иначе — точечное удаление ключа из внутренней
таблицы. Если секции или ключа не существует — вызов тихо ничего не
делает.

- **Список параметров:** `dict: var Config`, `section: string`, `key: string`.

**Пример:**

```nim
var dict = loadConfig(newStringStream("[Author]\nname=nim-lang\nwebsite=nim-lang.org\n"))
delSectionKey(dict, "Author", "website")
doAssert getSectionValue(dict, "Author", "website", "нет сайта") == "нет сайта"
doAssert getSectionValue(dict, "Author", "name") == "nim-lang"  # секция осталась, т.к. есть ещё ключ
```

---

### `sections` (итератор)

```nim
iterator sections*(dict: Config): lent string
```

**Что делает.** Перебирает имена всех секций таблицы, включая пустую строку
`""` (общая секция), если в ней есть хоть один ключ. Порядок обхода —
порядок вставки, поскольку `Config` — упорядоченная таблица.

- **Список параметров:** `dict: Config` — перебираемая таблица.

**Пример:**

```nim
let dict = loadConfig(newStringStream("charset=utf-8\n[Package]\nname=hello\n"))
for section in sections(dict):
  echo "секция: \"" & section & "\""
# выводит:
# секция: ""
# секция: "Package"
```

---

## Запись и сериализация конфигурации

### `writeConfig` (в поток)

```nim
proc writeConfig*(dict: Config, stream: Stream)
```

**Что делает.** Записывает содержимое таблицы в поток в формате `.ini`,
секция за секцией, в порядке вставки. Пустая секция `""` записывается без
заголовка `[...]`. Комментарии не поддерживаются — при чтении файла они
отбрасываются, и, соответственно, при записи взяться им неоткуда (эта
особенность явно отмечена в исходной документации процедуры).

**Разбор реализации.** Запись — это, по сути, обращение процесса разбора.
Модуль решает для каждого имени секции и каждого значения, нужно ли его
заключать в кавычки, по одному критерию: содержит ли строка только
"безопасные" символы (`SymChars` — буквы, цифры, `_`, пробел, `./\-` и
байты `\x80..\xFF`). Если да — запись идёт как есть; если нет — строка
оборачивается в обычные кавычки `"..."`, а если внутри уже встречается `"`
— в тройные `"""..."""`, чтобы не столкнуться с завершающей кавычкой.
Ключи с префиксом `"--"` распознаются по первым двум символам и пишутся
через двоеточие (`--ключ:значение`) вместо `=`, как того требует синтаксис
опций командной строки. Экранирование обратного слэша и переносов строк
внутри значения выполняет внутренняя `replace`.

- **Список параметров:**
  - `dict: Config` — записываемая таблица;
  - `stream: Stream` — открытый на запись поток.

**Пример:**

```nim
import std/streams

var dict = newConfig()
setSectionKey(dict, "", "charset", "utf-8")
setSectionKey(dict, "Package", "name", "hello")
setSectionKey(dict, "Package", "--threads", "on")
let stream = newStringStream()
writeConfig(dict, stream)
echo stream.data
# выводит:
# charset=utf-8
# [Package]
# name=hello
# --threads:on
```

---

### `writeConfig` (в файл)

```nim
proc writeConfig*(dict: Config, filename: string)
```

**Что делает.** То же самое, но открывает файл на запись самостоятельно
(режим `fmWrite`, то есть предыдущее содержимое файла полностью
перезаписывается) и закрывает его по завершении.

- **Список параметров:** `dict: Config`, `filename: string` — путь к файлу назначения.

**Пример:**

```nim
var dict = loadConfig("config.ini")
setSectionKey(dict, "Author", "name", "nim-lang")
writeConfig(dict, "config.ini")
```

---

### `` `$` ``

```nim
proc `$`*(dict: Config): string
```

**Что делает.** Возвращает содержимое таблицы в виде строки в том же
формате, что и `writeConfig` — удобно для отладочной печати или сравнения
в тестах без записи на диск.

**Разбор реализации.** Оборачивает `writeConfig` вокруг `StringStream`,
после чего забирает накопленные данные через `stream.data`.

- **Список параметров:** `dict: Config`.

**Пример:**

```nim
var dict = newConfig()
setSectionKey(dict, "Package", "name", "hello")
doAssert $dict == "[Package]\nname=hello\n"
```

---

## Практические рецепты

### Создание конфигурации с нуля и сохранение в файл

```nim
var dict = newConfig()
setSectionKey(dict, "", "charset", "utf-8")           # общая секция
setSectionKey(dict, "Package", "name", "hello")
setSectionKey(dict, "Package", "--threads", "on")      # опция командной строки
setSectionKey(dict, "Author", "name", "nim-lang")
setSectionKey(dict, "Author", "website", "nim-lang.org")
writeConfig(dict, "generated.ini")
```

### Чтение с значениями по умолчанию

```nim
let dict = loadConfig("config.ini")
let
  charset = getSectionValue(dict, "", "charset", "utf-8")
  threads = getSectionValue(dict, "Package", "--threads", "off")
  version = getSectionValue(dict, "Package", "version", "0.0.0")
echo charset & " / " & threads & " / " & version
```

### Обновление существующего файла (round-trip)

```nim
var dict = loadConfig("config.ini")
setSectionKey(dict, "Author", "name", "nim-lang")
delSectionKey(dict, "Author", "website")   # убрали устаревший ключ
writeConfig(dict, "config.ini")            # перезаписали файл на месте
```

### Перебор всех секций и ключей

```nim
let dict = loadConfig("config.ini")
for section in sections(dict):
  echo "[" & section & "]"
  for key, value in pairs(dict[section]):
    echo "  " & key & " = " & value
```

### Собственная обработка событий с игнорированием неизвестного

```nim
import std/streams

const knownSections = ["Package", "Author"]

var p: CfgParser
open(p, newFileStream("config.ini"), "config.ini")
while true:
  var e = next(p)
  case e.kind
  of cfgEof: break
  of cfgSectionStart:
    if not contains(knownSections, e.section):
      echo ignoreMsg(p, e)  # предупреждение с указанием файла/строки/столбца
  of cfgKeyValuePair:
    echo e.key & " = " & e.value
  of cfgOption:
    echo ignoreMsg(p, e)   # опции в этом сценарии не поддерживаются
  of cfgError:
    echo errorStr(p, e.msg)
close(p)
```

---

## Краткая таблица

| Задача | Изменяет `dict` | Возвращает |
|---|---|---|
| Разобрать файл событие за событием | нет (только `CfgParser`) | `CfgEvent` через `next` |
| Загрузить весь файл в таблицу | нет | новый `Config` (`loadConfig`) |
| Создать пустую конфигурацию | нет | новый `Config` (`newConfig`) |
| Прочитать значение с запасным вариантом | нет | `string` (`getSectionValue`) |
| Установить/создать ключ | да | — (`setSectionKey`) |
| Удалить секцию целиком | да | — (`delSection`) |
| Удалить один ключ (и секцию, если пуста) | да | — (`delSectionKey`) |
| Перебрать имена секций по порядку | нет | `string` за итерацию (`sections`) |
| Сохранить в поток/файл | нет | — (`writeConfig`) |
| Получить содержимое как строку | нет | `string` (`` `$` ``) |

---

## Сводка: какую процедуру выбрать

- Нужно прочитать файл целиком и работать с ним как со словарём → `loadConfig` + `getSectionValue`.
- Нужно собрать конфигурацию программно и записать на диск → `newConfig` + `setSectionKey` + `writeConfig`.
- Нужно читать значения, которых может не быть, без лишних проверок → `getSectionValue` с параметром `defaultVal`.
- Нужно удалить один параметр, не заботясь об опустевшей секции → `delSectionKey`.
- Нужно удалить секцию целиком → `delSection`.
- Нужно перебрать все секции в порядке файла → итератор `sections`.
- Нужен полный контроль над разбором (свой формат ошибок, частичная загрузка, потоковая обработка без буферизации всего файла) → низкоуровневый `CfgParser` с `open`/`next`/`close`.
- Нужно сообщить пользователю, что нераспознанная директива проигнорирована → `ignoreMsg`.
- Нужно получить текст ошибки/предупреждения с указанием строки и столбца вручную → `errorStr`/`warningStr`.
