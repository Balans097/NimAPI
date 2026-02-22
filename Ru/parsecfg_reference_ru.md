# Справочник модуля `parsecfg`

> **Стандартная библиотека Nim — `std/parsecfg`**
> Высокопроизводительный парсер конфигурационных файлов в стиле INI и полный
> API для чтения и записи структурированных конфигурационных данных.
> Поддерживает как потоковый pull-парсер, так и высокоуровневый табличный интерфейс.

---

## Содержание

1. [Обзор и принцип работы](#обзор-и-принцип-работы)
2. [Синтаксис конфигурационных файлов](#синтаксис-конфигурационных-файлов)
3. [Типы](#типы)
   - [CfgEventKind](#cfgeventkind)
   - [CfgEvent](#cfgevent)
   - [CfgParser](#cfgparser)
   - [Config](#config)
4. [Низкоуровневый pull-парсер](#низкоуровневый-pull-парсер)
   - [open](#open)
   - [close](#close)
   - [next](#next)
   - [getLine](#getline)
   - [getColumn](#getcolumn)
   - [getFilename](#getfilename)
   - [errorStr](#errorstr)
   - [warningStr](#warningstr)
   - [ignoreMsg](#ignoremsg)
5. [Высокоуровневый табличный API](#высокоуровневый-табличный-api)
   - [newConfig](#newconfig)
   - [loadConfig (из потока)](#loadconfig-из-потока)
   - [loadConfig (из файла)](#loadconfig-из-файла)
   - [getSectionValue](#getsectionvalue)
   - [setSectionKey](#setsectionkey)
   - [delSectionKey](#deletesectionkey)
   - [delSection](#deletesection)
   - [sections](#sections)
   - [writeConfig (в поток)](#writeconfig-в-поток)
   - [writeConfig (в файл)](#writeconfig-в-файл)
   - [$ (в строку)](#-в-строку)
6. [Какой API выбрать?](#какой-api-выбрать)
7. [Полные рабочие примеры](#полные-рабочие-примеры)

---

## Обзор и принцип работы

Модуль `parsecfg` предоставляет два принципиально разных способа работы с конфигурационными файлами:

**1. Pull-парсер (низкоуровневый)** — потоковая обработка событие за событием. Вы вызываете `next()` в цикле и получаете по одному `CfgEvent` за раз. Экономичен по памяти для больших файлов; позволяет реагировать на каждый токен по мере его разбора.

**2. Config-таблица (высокоуровневый)** — весь файл загружается в `OrderedTableRef`, сопоставляющий секции парам «ключ-значение». Прост, привычен и достаточен для подавляющего большинства реальных приложений.

```
Конфиг-файл на диске
        │
        │  loadConfig(filename)        ← высокоуровневый: одна команда всё загружает
        ▼
  Config (таблица)
  ├─ getSectionValue(section, key)
  ├─ setSectionKey(section, key, val)
  ├─ delSectionKey / delSection
  ├─ writeConfig(filename)
  └─ $dict  →  string

        │
        │  open(parser, stream, ...)   ← низкоуровневый: ручной цикл событий
        ▼
  CfgParser
  └─ цикл: next(parser) → CfgEvent
       ├─ cfgSectionStart  → e.section
       ├─ cfgKeyValuePair  → e.key, e.value
       ├─ cfgOption        → e.key, e.value
       ├─ cfgError         → e.msg
       └─ cfgEof           → break
```

Тип `Config` — псевдоним `OrderedTableRef[string, OrderedTableRef[string, string]]`: таблица секций, каждая из которых содержит упорядоченную таблицу пар «ключ-значение». Порядок вставки секций и ключей сохраняется.

---

## Синтаксис конфигурационных файлов

Парсер понимает расширенный вариант формата Windows `.ini`. Ключевые моменты:

**Секции** объявляются строкой `[ИмяСекции]`. Пробелы вокруг имени секции обрезаются. Пустая секция `""` — это неявная глобальная секция для ключей, написанных до первого заголовка `[...]`.

**Пары ключ-значение** используют `=` или `:` как разделитель:
```ini
name = hello
port: 8080
```

**Параметры командной строки** начинаются с `--` и используют `:` как разделитель. Они приходят как события `cfgOption` (и хранятся в `Config`-таблице с префиксом `--`):
```ini
--threads:on
--opt:speed
```

**Строковые литералы** следуют соглашениям Nim:
- Простые символы: `value`, `nim-lang.org`, `some/path`
- Строки в кавычках: `"hello world"` — поддерживают escape-последовательности (`\n`, `\t`, `\x41` и т.д.)
- Сырые строки: `r"C:\Users\nim"` — обратные слеши буквальны, без обработки escape
- Тройные кавычки: `"""многострочное значение"""` — могут занимать несколько строк

**Комментарии** начинаются с `#` или `;` и продолжаются до конца строки:
```ini
# это комментарий
; это тоже
key = value  # встроенные комментарии НЕ поддерживаются — # входит в значение
```

**Встроенные комментарии не поддерживаются.** Всё после `=` или `:` до конца токена — это значение, включая символы `#`.

**Пробелы** вокруг ключей, значений и имён секций обрезаются.

**Ключи без значений** допустимы — `value` устанавливается в `""`.

Полный пример:
```ini
charset = "utf-8"

[Package]
name = "hello"
--threads:on

[Author]
name = "nim-lang"
website = "nim-lang.org"
```

---

## Типы

### `CfgEventKind`

```nim
type CfgEventKind* = enum
  cfgEof,          ## достигнут конец файла
  cfgSectionStart, ## разобран заголовок [section]
  cfgKeyValuePair, ## разобрана пара key=value или key:value
  cfgOption,       ## разобран параметр --key:value
  cfgError         ## возникла ошибка разбора (исключения не выбрасываются)
```

Дискриминатор для `CfgEvent`. Каждый вызов `next()` возвращает событие одного из этих пяти видов. Ошибки не выбрасывают исключений — они приходят как события `cfgError`, чтобы вызывающий код сам решал, как с ними поступить.

---

### `CfgEvent`

```nim
type CfgEvent* = object of RootObj
  case kind*: CfgEventKind
  of cfgEof:         (ничего)
  of cfgSectionStart:
    section*: string
  of cfgKeyValuePair, cfgOption:
    key*:   string
    value*: string   ## "" если значение не указано
  of cfgError:
    msg*: string
```

Вариантный объект, описывающий один разобранный токен. Доступные поля зависят от `kind`:

| `kind` | Доступные поля | Что содержат |
|---|---|---|
| `cfgEof` | — | ничего; сигнализирует конец ввода |
| `cfgSectionStart` | `section` | имя секции (без `[` и `]`) |
| `cfgKeyValuePair` | `key`, `value` | ключ и значение; `value = ""` если не указано |
| `cfgOption` | `key`, `value` | имя параметра без `--`; `value` может быть `""` |
| `cfgError` | `msg` | форматированная строка ошибки с файлом, строкой, столбцом |

---

### `CfgParser`

```nim
type CfgParser* = object of BaseLexer
```

Объект парсера для низкоуровневого pull-парсер API. Объявите как `var`, инициализируйте через `open()`, управляйте через `next()`, освобождайте через `close()`. Внутреннее состояние управляется модулем — не обращайтесь к его полям напрямую.

---

### `Config`

```nim
type Config* = OrderedTableRef[string, OrderedTableRef[string, string]]
```

Структура данных высокоуровневого API. Это ссылочный тип (выделяется на куче), поэтому передавать его можно без копирования. Порядок вставки секций и ключей сохраняется.

- **Внешняя таблица** сопоставляет имена секций (`string`) с внутренними таблицами.
- **Внутренняя таблица** сопоставляет имена ключей с их строковыми значениями.
- **Глобальная секция** (ключи до первого заголовка `[...]`) хранится под пустым строковым ключом `""`.
- Записи `--option` хранятся с префиксом `--`: `dict["Section"]["--threads"]`.

---

## Низкоуровневый pull-парсер

Используйте этот API, когда нужно обработать конфиг-файл как поток, реагировать на отдельные события или когда файл слишком велик для полной загрузки в память.

### `open`

```nim
proc open*(c: var CfgParser, input: Stream, filename: string, lineOffset = 0)
```

**Что делает:** Инициализирует парсер и привязывает его к входному потоку `Stream`. После `open` парсер стоит перед первым токеном; вызов `next()` читает первое событие.

**Параметры:**
- `input` — любой объект `Stream`: `newFileStream`, `newStringStream` и т.д.
- `filename` — используется только для форматирования сообщений об ошибках; реальным путём может и не быть.
- `lineOffset` — смещает отображаемый номер строки на указанное количество. Полезно, когда содержимое конфига встроено в больший файл и нужно, чтобы ошибки указывали абсолютный номер строки.

```nim
import std/[streams, parsecfg]

var s = newFileStream("app.cfg", fmRead)
var p: CfgParser
open(p, s, "app.cfg")
# p готов; вызывайте next(p) для чтения событий
```

```nim
# Разбор конфиг-строки, встроенной в код
var s = newStringStream("[db]\nhost=localhost\nport=5432\n")
var p: CfgParser
open(p, s, "<embedded>")
```

---

### `close`

```nim
proc close*(c: var CfgParser)
```

**Что делает:** Закрывает парсер и связанный поток. Всегда вызывайте `close` по завершении — идиоматичный паттерн: `defer: p.close()` сразу после `open`.

```nim
open(p, stream, "config.ini")
defer: p.close()
```

---

### `next`

```nim
proc next*(c: var CfgParser): CfgEvent
```

**Что делает:** Переводит парсер к следующему событию и возвращает его. Это главный двигатель цикла pull-парсера. Каждый вызов потребляет ровно один логический токен из входного потока.

Шаблон цикла всегда одинаков: вызываем `next`, разветвляемся по `e.kind`, выходим при `cfgEof`.

```nim
while true:
  var e = next(p)
  case e.kind
  of cfgEof:          break
  of cfgSectionStart: echo "секция: ", e.section
  of cfgKeyValuePair: echo e.key, " = ", e.value
  of cfgOption:       echo "--", e.key, ": ", e.value
  of cfgError:        echo "ошибка: ", e.msg
```

**Важно:** Парсер **не выбрасывает исключений** при синтаксических ошибках. Синтаксические ошибки приходят как события `cfgError`. Получив `cfgError`, вы можете продолжить вызывать `next()` для попытки восстановления — или немедленно прервать цикл.

---

### `getLine`

```nim
proc getLine*(c: CfgParser): int
```

**Что делает:** Возвращает текущий номер строки (начиная с 1) в позиции парсера. Полезно при формировании собственных сообщений об ошибках или предупреждениях с информацией о позиции.

---

### `getColumn`

```nim
proc getColumn*(c: CfgParser): int
```

**Что делает:** Возвращает текущий номер столбца (начиная с 0) в позиции парсера.

---

### `getFilename`

```nim
proc getFilename*(c: CfgParser): string
```

**Что делает:** Возвращает строку `filename`, переданную в `open`. Используется внутри `errorStr` и `warningStr` для формирования последовательно отформатированных сообщений.

---

### `errorStr`

```nim
proc errorStr*(c: CfgParser, msg: string): string
```

**Что делает:** Форматирует `msg` в стандартный формат сообщения об ошибке:

```
имя_файла(строка, столбец) Error: msg
```

Используйте это, когда вы обнаруживаете *семантическую* ошибку в своём коде — например, отсутствует обязательный ключ — и хотите, чтобы сообщение выглядело единообразно с ошибками парсера.

```nim
if not dict.hasKey("db"):
  echo p.errorStr("обязательная секция [db] отсутствует")
  # → "config.ini(42, 1) Error: обязательная секция [db] отсутствует"
```

---

### `warningStr`

```nim
proc warningStr*(c: CfgParser, msg: string): string
```

**Что делает:** Аналогично `errorStr`, но форматирует сообщение с префиксом `Warning:` вместо `Error:`:

```
имя_файла(строка, столбец) Warning: msg
```

Используйте для нефатальных проблем — неизвестных ключей, которые игнорируются, устаревших настроек и т.д.

```nim
echo p.warningStr("неизвестный ключ '" & key & "' будет проигнорирован")
# → "config.ini(15, 1) Warning: неизвестный ключ 'legacy_mode' будет проигнорирован"
```

---

### `ignoreMsg`

```nim
proc ignoreMsg*(c: CfgParser, e: CfgEvent): string
```

**Что делает:** Возвращает стандартное предупреждение «эта запись игнорируется» для любого `CfgEvent`. Внутри делегирует `warningStr`, поэтому сообщение включает файл, строку и столбец. Для событий `cfgError` возвращает `e.msg` без изменений. Для `cfgEof` возвращает `""`.

Это удобное сокращение для типичного паттерна: вы хотите молча пропустить неизвестную секцию или ключ, но всё же залогировать предупреждение.

```nim
while true:
  var e = next(p)
  case e.kind
  of cfgEof: break
  of cfgSectionStart:
    if e.section notin ["Package", "Author"]:
      echo p.ignoreMsg(e)  # "config.ini(8, 1) Warning: section ignored: Unknown"
  of cfgKeyValuePair:
    if e.key == "legacy":
      echo p.ignoreMsg(e)  # "config.ini(9, 1) Warning: key ignored: legacy"
  else: discard
```

---

## Высокоуровневый табличный API

Этот API загружает весь конфиг в память в виде вложенной упорядоченной таблицы. Для большинства приложений только он и нужен.

### `newConfig`

```nim
proc newConfig*(): Config
```

**Что делает:** Создаёт и возвращает пустую `Config`-таблицу. Используйте как отправную точку, когда хотите построить конфиг из кода программно, а затем записать его на диск.

```nim
import std/parsecfg

var cfg = newConfig()
cfg.setSectionKey("", "charset", "utf-8")
cfg.setSectionKey("Server", "host", "localhost")
cfg.setSectionKey("Server", "port", "8080")
```

---

### `loadConfig` (из потока)

```nim
proc loadConfig*(stream: Stream, filename: string = "[stream]"): Config
```

**Что делает:** Разбирает конфиг из открытого `Stream` и возвращает результат как `Config`-таблицу. `filename` необязателен и используется только для сообщений об ошибках. Разбор останавливается на первом событии `cfgError`.

Используйте, когда у вас уже есть `Stream` — например, строковый поток из конфига, встроенного в код, сетевое соединение или распакованный буфер в памяти.

```nim
import std/[streams, parsecfg]

let cfgText = """
[server]
host = localhost
port = 8080
"""
let cfg = loadConfig(newStringStream(cfgText))
echo cfg.getSectionValue("server", "host")  # "localhost"
```

---

### `loadConfig` (из файла)

```nim
proc loadConfig*(filename: string): Config
```

**Что делает:** Открывает файл по пути `filename`, разбирает его содержимое и возвращает результат как `Config`. Это наиболее типичная точка входа для чтения реального конфиг-файла. Работает как в скомпилированном коде, так и в NimScript (путь NimScript использует `readFile` внутри).

```nim
import std/parsecfg

let cfg = loadConfig("settings.ini")
echo cfg.getSectionValue("Database", "host")
echo cfg.getSectionValue("Database", "port", "5432")  # значение по умолчанию
```

---

### `getSectionValue`

```nim
proc getSectionValue*(dict: Config, section, key: string, defaultVal = ""): string
```

**Что делает:** Возвращает значение, связанное с `key` в заданной `section`. Если секция не существует, или ключ в ней отсутствует — возвращает `defaultVal` (по умолчанию пустая строка). Никогда не выбрасывает исключений.

- Для доступа к **глобальной секции** (ключи до первого `[...]`) передайте `section = ""`.
- Для доступа к записям `--option` передайте полное имя с префиксом: `key = "--threads"`.

```nim
let cfg = loadConfig("app.ini")

# Чтение без значения по умолчанию (возвращает "" при отсутствии)
let host = cfg.getSectionValue("Database", "host")

# Чтение с явным значением по умолчанию
let port = cfg.getSectionValue("Database", "port", "5432")

# Чтение из глобальной секции
let charset = cfg.getSectionValue("", "charset")

# Чтение параметра командной строки из конфига
let threads = cfg.getSectionValue("Build", "--threads")
```

---

### `setSectionKey`

```nim
proc setSectionKey*(dict: var Config, section, key, value: string)
```

**Что делает:** Устанавливает значение `key` равным `value` в заданной секции `section`. Если секция ещё не существует — создаёт её. Если ключ уже существует — перезаписывает его. Порядок вставки секций и ключей сохраняется.

- Используйте `section = ""` для глобальной секции.
- Ключ, начинающийся с `--`, сохраняет параметр командной строки.

```nim
var cfg = loadConfig("app.ini")
cfg.setSectionKey("Database", "host", "db.example.com")
cfg.setSectionKey("Database", "port", "5432")
cfg.setSectionKey("", "version", "2")           # глобальный ключ
cfg.setSectionKey("Build", "--opt", "speed")    # --параметр
cfg.writeConfig("app.ini")
```

---

### `delSectionKey`

```nim
proc delSectionKey*(dict: var Config, section, key: string)
```

**Что делает:** Удаляет отдельный ключ из секции. Если после удаления ключа секция становится пустой — она тоже удаляется из таблицы. Ничего не делает, если секция или ключ не существуют.

```nim
var cfg = loadConfig("app.ini")
cfg.delSectionKey("Author", "website")   # удалить один ключ
cfg.delSectionKey("Legacy", "old_key")   # безопасно, даже если секции нет
cfg.writeConfig("app.ini")
```

---

### `delSection`

```nim
proc delSection*(dict: var Config, section: string)
```

**Что делает:** Удаляет целую секцию со всеми её ключами из конфиг-таблицы. Ничего не делает, если секция не существует.

```nim
var cfg = loadConfig("app.ini")
cfg.delSection("DeprecatedSettings")  # удалить всю секцию
cfg.writeConfig("app.ini")
```

---

### `sections`

```nim
iterator sections*(dict: Config): lent string
```

**Что делает:** Перебирает все имена секций в `Config` в порядке их вставки. Возвращает каждое имя секции как `lent string` (заимствованная ссылка — не сохраняйте её за пределами тела цикла без копирования).

Глобальная секция `""` включается в перечисление, если она существует в таблице.

```nim
let cfg = loadConfig("app.ini")

for section in cfg.sections:
  echo "Найдена секция: ", section

# Если нужно сохранить секции:
var sectionNames: seq[string]
for s in cfg.sections:
  sectionNames.add(s)  # копирование через add безопасно
```

---

### `writeConfig` (в поток)

```nim
proc writeConfig*(dict: Config, stream: Stream)
```

**Что делает:** Сериализует всю `Config`-таблицу в открытый `Stream` в формате INI. Вывод подходит для обратного чтения через `loadConfig`. Секции записываются в порядке вставки; ключи внутри каждой секции — тоже.

> **Примечание:** Комментарии из исходного файла **не сохраняются**. Поскольку тип `Config` хранит только пары «ключ-значение», все `#` и `;` комментарии из исходного файла теряются при загрузке и повторной записи конфига.

Строковые значения, содержащие специальные символы или пробелы, автоматически заключаются в кавычки или тройные кавычки. Ключи `--option` записываются с префиксом `--` и разделителем `:`.

```nim
import std/[streams, parsecfg]

var cfg = newConfig()
cfg.setSectionKey("Server", "host", "localhost")
cfg.setSectionKey("Server", "port", "8080")

var s = newStringStream()
cfg.writeConfig(s)
echo s.data
# [Server]
# host=localhost
# port=8080
```

---

### `writeConfig` (в файл)

```nim
proc writeConfig*(dict: Config, filename: string)
```

**Что делает:** Записывает `Config` в файл, создавая его или перезаписывая. Это естественная пара к `loadConfig(filename)` и основной способ сохранить изменения.

```nim
var cfg = loadConfig("settings.ini")
cfg.setSectionKey("App", "version", "2.0")
cfg.writeConfig("settings.ini")  # перезапись на месте
```

---

### `$` (в строку)

```nim
proc `$`*(dict: Config): string
```

**Что делает:** Сериализует `Config` в Nim-строку в формате INI. Эквивалентно вызову `writeConfig` на `newStringStream()` с последующим чтением результата. Полезно для отладки, логирования или сравнения состояния конфига в тестах.

```nim
var cfg = newConfig()
cfg.setSectionKey("", "charset", "utf-8")
cfg.setSectionKey("Package", "name", "myapp")
echo $cfg
# charset=utf-8
# [Package]
# name=myapp
```

---

## Какой API выбрать?

| Сценарий | Рекомендуемый API |
|---|---|
| Прочитать конфиг-файл и обратиться к нескольким значениям | `loadConfig(filename)` + `getSectionValue` |
| Создать новый конфиг в коде и записать на диск | `newConfig()` + `setSectionKey` + `writeConfig` |
| Изменить существующий конфиг-файл | `loadConfig` + `setSectionKey`/`delSectionKey` + `writeConfig` |
| Обработать очень большой конфиг без полной загрузки | pull-парсер: `open` + цикл с `next` + `close` |
| Проверить структуру конфига или обнаружить неизвестные ключи | pull-парсер: `open` + цикл с `next`, используйте `ignoreMsg`/`warningStr` |
| Встроить конфиг-строку прямо в код | `loadConfig(newStringStream(...))` |
| Перебрать все секции конфига | итератор `sections` |

---

## Полные рабочие примеры

### Пример 1: Полный цикл pull-парсера со всеми типами событий

```nim
import std/[streams, parsecfg]

let cfgText = """
charset = "utf-8"
[Package]
name = hello
--threads:on
[Author]
name = nim-lang
website = nim-lang.org
"""

var s = newStringStream(cfgText)
var p: CfgParser
open(p, s, "<string>")
defer: p.close()

while true:
  let e = next(p)
  case e.kind
  of cfgEof:
    echo "--- конец конфига ---"
    break
  of cfgSectionStart:
    echo "[", e.section, "]"
  of cfgKeyValuePair:
    echo "  ", e.key, " = ", e.value
  of cfgOption:
    echo "  --", e.key, ": ", e.value
  of cfgError:
    echo "ОШИБКА: ", e.msg
    break  # прервать при первой ошибке; можно продолжить для попытки восстановления
```

Вывод:
```
  charset = utf-8
[Package]
  name = hello
  --threads: on
[Author]
  name = nim-lang
  website = nim-lang.org
--- конец конфига ---
```

---

### Пример 2: Программное создание конфига и запись на диск

```nim
import std/parsecfg

var cfg = newConfig()

# Глобальные ключи (без секции)
cfg.setSectionKey("", "charset", "utf-8")

# Секция [Package]
cfg.setSectionKey("Package", "name", "myapp")
cfg.setSectionKey("Package", "version", "1.0.0")
cfg.setSectionKey("Package", "--threads", "on")

# Секция [Database]
cfg.setSectionKey("Database", "host", "localhost")
cfg.setSectionKey("Database", "port", "5432")
cfg.setSectionKey("Database", "password", r"s3cr3t\pass")  # содержит обратный слеш

cfg.writeConfig("generated.ini")
echo $cfg
```

---

### Пример 3: Загрузка, изменение и сохранение существующего конфига

```nim
import std/parsecfg

# Загрузка
var cfg = loadConfig("app.ini")

# Чтение текущих значений
let oldHost = cfg.getSectionValue("Database", "host", "localhost")
echo "Текущий хост: ", oldHost

# Изменение
cfg.setSectionKey("Database", "host", "db.production.example.com")
cfg.setSectionKey("Database", "port", "5432")

# Удаление устаревшего ключа
cfg.delSectionKey("Legacy", "oldSetting")

# Удаление ненужной секции
cfg.delSection("Debug")

# Сохранение
cfg.writeConfig("app.ini")
```

---

### Пример 4: Валидация конфига по обязательной схеме

Использование pull-парсера для проверки наличия обязательных секций и ключей:

```nim
import std/[streams, parsecfg, sets]

proc validateConfig(path: string): bool =
  let required = toHashSet(["host", "port", "password"])
  var found: HashSet[string]
  var inDB = false

  var s = newFileStream(path, fmRead)
  var p: CfgParser
  open(p, s, path)
  defer: p.close()

  while true:
    let e = next(p)
    case e.kind
    of cfgEof: break
    of cfgError:
      echo p.errorStr(e.msg)
      return false
    of cfgSectionStart:
      inDB = (e.section == "Database")
    of cfgKeyValuePair:
      if inDB: found.incl(e.key)
    else: discard

  let missing = required - found
  if missing.len > 0:
    for key in missing:
      echo p.warningStr("обязательный ключ '" & key & "' отсутствует в [Database]")
    return false
  return true

if validateConfig("app.ini"):
  echo "Конфиг корректен"
else:
  echo "Валидация конфига не прошла"
```

---

### Пример 5: Вывод всех секций и количества ключей в каждой

```nim
import std/parsecfg

let cfg = loadConfig("app.ini")

for section in cfg.sections:
  let label = if section == "": "<глобальная>" else: "[" & section & "]"
  var count = 0
  if cfg.hasKey(section):
    for _ in cfg[section].pairs:
      inc count
  echo label, ": ", count, " ключ(ей)"
```

---

### Пример 6: Тест «туда и обратно» — записать, загрузить, проверить

```nim
import std/parsecfg

var original = newConfig()
original.setSectionKey("", "encoding", "utf-8")
original.setSectionKey("Net", "host", "example.com")
original.setSectionKey("Net", "port", "443")
original.setSectionKey("Net", "--tls", "on")

# Запись во временный файл
original.writeConfig("/tmp/roundtrip.ini")

# Загрузка обратно
let loaded = loadConfig("/tmp/roundtrip.ini")

# Проверка
assert loaded.getSectionValue("", "encoding") == "utf-8"
assert loaded.getSectionValue("Net", "host") == "example.com"
assert loaded.getSectionValue("Net", "port") == "443"
assert loaded.getSectionValue("Net", "--tls") == "on"

echo "Тест «туда и обратно» прошёл успешно"
```
