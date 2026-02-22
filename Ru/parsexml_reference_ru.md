# Справочник модуля `parsexml`

> **Стандартная библиотека Nim — `std/parsexml`**
> Высокопроизводительный потоковый парсер XML и HTML с коррекцией ошибок.
> Поддерживается только кодировка UTF-8.

---

## Содержание

1. [Обзор и принцип работы](#обзор-и-принцип-работы)
2. [Типы](#типы)
   - [XmlEventKind](#xmleventkind)
   - [XmlErrorKind](#xmlerrorkind)
   - [XmlParseOption](#xmlparseoption)
   - [XmlParser](#xmlparser)
3. [Жизненный цикл парсера](#жизненный-цикл-парсера)
   - [open](#open)
   - [close](#close)
4. [Управление парсером](#управление-парсером)
   - [next](#next)
   - [kind](#kind)
5. [Чтение данных событий](#чтение-данных-событий)
   - [charData](#chardata)
   - [elementName](#elementname)
   - [entityName](#entityname)
   - [attrKey](#attrkey)
   - [attrValue](#attrvalue)
   - [piName](#piname)
   - [piRest](#pirest)
6. [Позиция и диагностика](#позиция-и-диагностика)
   - [getLine](#getline)
   - [getColumn](#getcolumn)
   - [getFilename](#getfilename)
   - [errorMsg (событие xmlError)](#errormsg-событие-xmlerror)
   - [errorMsg (произвольный текст)](#errormsg-произвольный-текст)
   - [errorMsgExpected](#errormsgexpected)
7. [Низкоуровневые / оптимизационные процедуры](#низкоуровневые--оптимизационные-процедуры)
   - [rawData](#rawdata)
   - [rawData2](#rawdata2)
8. [Полные рабочие примеры](#полные-рабочие-примеры)

---

## Обзор и принцип работы

`parsexml` — это **pull-парсер** (он же *потоковый* или *событийный* парсер). В отличие от DOM-парсера, он **не строит дерево** документа в памяти. Вместо этого вы вызываете `next()` в цикле, а парсер переходит к следующему *событию*. Затем вы проверяете `kind`, чтобы узнать, что было обнаружено, и вызываете нужный аксессор для получения данных.

```
┌─────────────────────────────────────────────────────────┐
│  Ваш код управляет парсером через next()                 │
│                                                          │
│  loop:                                                   │
│    next(x)           ← перейти к следующему токену       │
│    case x.kind       ← что нашли?                        │
│    of xmlCharData:   ← текст между тегами                │
│      использовать x.charData                             │
│    of xmlElementStart:                                   │
│      использовать x.elementName                          │
│    of xmlEof: break  ← конец файла                       │
└─────────────────────────────────────────────────────────┘
```

Ключевые особенности, которые важно знать:

- **Парсер не проверяет соответствие тегов.** Он не сообщит вам, что `<div>` не закрыт соответствующим `</div>`. Проверку соответствия открывающих и закрывающих тегов вы должны реализовать сами.
- **Устойчивость к ошибкам.** Парсер рассчитан на обработку «дикого HTML» — некорректно сформированных документов, которые широко распространены в сети. При ошибках он пытается восстановиться, а не прерваться с исключением.
- **Тег с атрибутами разбивается на несколько событий.** Например, `<a href="x">` порождает три последовательных события: `xmlElementOpen` → `xmlAttribute` → `xmlElementClose`. Тег без атрибутов порождает единственное событие `xmlElementStart`.

---

## Типы

### `XmlEventKind`

Перечисление, описывающее все возможные события парсера. После каждого вызова `next()` значение `kind` будет одним из следующих:

| Значение | Когда возникает | Пример в XML/HTML |
|---|---|---|
| `xmlError` | Произошла ошибка разбора | некорректный ввод |
| `xmlEof` | Достигнут конец входного потока | — |
| `xmlCharData` | Обычный текст между тегами | `Привет, мир!` |
| `xmlWhitespace` | Пробельные символы (только с опцией `reportWhitespace`) | `   ` |
| `xmlComment` | Комментарий (только с опцией `reportComments`) | `<!-- примечание -->` |
| `xmlPI` | Инструкция обработки | `<?xml version="1.0"?>` |
| `xmlElementStart` | Открывающий тег **без** атрибутов | `<br>`, `<p>` |
| `xmlElementEnd` | Закрывающий тег | `</p>` |
| `xmlElementOpen` | Начало тега **с** атрибутами (только имя тега, `>` ещё не встречен) | `<a ` |
| `xmlAttribute` | Пара ключ=значение | `href="https://..."` |
| `xmlElementClose` | Закрывающий `>` элемента с атрибутами | `>` |
| `xmlCData` | Содержимое секции CDATA | `<![CDATA[ raw ]]>` |
| `xmlEntity` | Нераспознанная именованная сущность | `&nbsp;` |
| `xmlSpecial` | Любая конструкция `<!...>`, кроме CDATA и комментариев | `<!DOCTYPE html>` |

> **Важно:** Различие между `xmlElementStart` и тройкой `xmlElementOpen` + `xmlAttribute` + `xmlElementClose` принципиально. Если вы ищете только `xmlElementStart`, вы пропустите все элементы, у которых есть атрибуты.

---

### `XmlErrorKind`

Перечисление конкретных ошибок разбора. Как правило, вам не нужно обращаться к нему напрямую — используйте `errorMsg()` для получения читаемого сообщения об ошибке.

| Значение | Описание |
|---|---|
| `errNone` | Ошибок нет |
| `errEndOfCDataExpected` | Отсутствует `]]>` в конце секции CDATA |
| `errNameExpected` | Ожидалось имя тега или атрибута |
| `errSemicolonExpected` | Отсутствует `;` после ссылки на сущность |
| `errQmGtExpected` | Отсутствует `?>` в конце инструкции обработки |
| `errGtExpected` | Отсутствует `>` в конце тега |
| `errEqExpected` | Отсутствует `=` между именем атрибута и значением |
| `errQuoteExpected` | Отсутствуют кавычки (`"` или `'`) вокруг значения атрибута |
| `errEndOfCommentExpected` | Отсутствует `-->` в конце комментария |
| `errAttributeValueExpected` | Значение атрибута пустое или отсутствует |

---

### `XmlParseOption`

Набор флагов, передаваемых в `open()` для управления поведением парсера.

| Флаг | Эффект |
|---|---|
| `reportWhitespace` | Генерировать события `xmlWhitespace` для пробельных символов. По умолчанию пробелы между тегами молча пропускаются. |
| `reportComments` | Генерировать события `xmlComment` и заполнять `charData` текстом комментария. По умолчанию комментарии молча игнорируются. |
| `allowUnquotedAttribs` | Принимать значения атрибутов без кавычек, как это часто встречается в старом HTML (`<td width=100>`). |
| `allowEmptyAttribs` | Принимать атрибуты без значения (`<input disabled>`). |

---

### `XmlParser`

Объект парсера. Объявите его как `var`, инициализируйте через `open()`, управляйте через `next()`, освобождайте ресурсы через `close()`.

```nim
var x: XmlParser
open(x, stream, "myfile.xml")
# ... работа с x ...
close(x)
```

Рассматривайте `XmlParser` как непрозрачный тип и обращайтесь к его состоянию только через документированные процедуры и шаблоны.

---

## Жизненный цикл парсера

### `open`

```nim
proc open*(my: var XmlParser; input: Stream; filename: string;
           options: set[XmlParseOption] = {})
```

**Что делает:** Инициализирует парсер и привязывает его к входному потоку `Stream`. Должна вызываться перед любой другой процедурой. Аргумент `filename` используется только для форматирования сообщений об ошибках — это не обязательно должен быть реальный путь к файлу.

**Параметры:**
- `input` — любой объект `Stream` (файловый поток, строковый поток и т.д.).
- `filename` — метка для сообщений об ошибках (например, имя файла или `"<stdin>"`).
- `options` — набор флагов `XmlParseOption` (по умолчанию: пустой набор).

```nim
import std/[streams, parsexml]

# Разбор из файла
var s = newFileStream("data.xml", fmRead)
var x: XmlParser
open(x, s, "data.xml")

# Разбор строки в памяти
var s2 = newStringStream("<root><item>hello</item></root>")
var x2: XmlParser
open(x2, s2, "<string>")

# HTML-режим с расширенными опциями
var s3 = newFileStream("page.html", fmRead)
var x3: XmlParser
open(x3, s3, "page.html", {allowUnquotedAttribs, allowEmptyAttribs})
```

---

### `close`

```nim
proc close*(my: var XmlParser)
```

**Что делает:** Закрывает парсер и освобождает ресурсы, связанные с потоком. Всегда вызывайте `close()` после завершения работы — даже если вы остановили разбор досрочно (например, сразу после нахождения нужных данных).

```nim
x.close()
```

---

## Управление парсером

### `next`

```nim
proc next*(my: var XmlParser)
```

**Что делает:** Переводит парсер к следующему событию. Это двигатель pull-парсера. Каждый вызов продвигает парсер ровно на один токен. После вызова проверьте `my.kind`, чтобы узнать, что было разобрано, и вызовите нужный аксессор для получения данных.

`next()` молча отбрасывает комментарии и пробельные символы, если соответствующие опции (`reportComments`, `reportWhitespace`) не были переданы в `open()`. Первая инструкция обработки `<?xml ... ?>` также молча пропускается.

```nim
var s = newStringStream("<msg lang=\"ru\">Привет</msg>")
var x: XmlParser
open(x, s, "")

x.next()  # → xmlElementOpen  (у тега есть атрибут)
echo x.elementName  # "msg"

x.next()  # → xmlAttribute
echo x.attrKey    # "lang"
echo x.attrValue  # "ru"

x.next()  # → xmlElementClose  (символ '>' закрывает <msg lang="ru">)

x.next()  # → xmlCharData
echo x.charData  # "Привет"

x.next()  # → xmlElementEnd
echo x.elementName  # "msg"

x.next()  # → xmlEof

x.close()
```

---

### `kind`

```nim
proc kind*(my: XmlParser): XmlEventKind
```

**Что делает:** Возвращает тип последнего разобранного события. Это центральный дискриминатор — почти любое использование парсера начинается с проверки `kind`.

```nim
x.next()
case x.kind
of xmlElementStart: echo "открывающий тег: ", x.elementName
of xmlElementEnd:   echo "закрывающий тег: ", x.elementName
of xmlCharData:     echo "текст: ", x.charData
of xmlEof:          echo "конец файла"
else:               discard
```

---

## Чтение данных событий

Эти шаблоны и процедуры позволяют читать данные, связанные с текущим событием. **Каждый аксессор допустимо вызывать только тогда, когда `kind` соответствует задокументированным событиям.** В режиме отладки вызов аксессора с несоответствующим `kind` вызывает ошибку утверждения (`assert`). В режиме release поведение не определено.

---

### `charData`

```nim
template charData*(my: XmlParser): string
```

**Допустимо для:** `xmlCharData`, `xmlWhitespace`, `xmlComment`, `xmlCData`, `xmlSpecial`

**Что делает:** Возвращает текстовое содержимое текущего события. Для `xmlCharData` — это обычный текст между тегами. Для `xmlComment` — текст комментария (без разделителей `<!--` и `-->`). Для `xmlCData` — необработанные данные внутри `<![CDATA[...]]>`. Для `xmlSpecial` — необработанное содержимое конструкций `<!...>`, например деклараций DOCTYPE.

```nim
# Ввод: <p>Привет <b>мир</b></p>
# После next(), если kind == xmlCharData:
echo x.charData  # "Привет "

# Ввод: <!-- это комментарий -->
# (парсер открыт с опцией reportComments)
# После next(), если kind == xmlComment:
echo x.charData  # " это комментарий "

# Ввод: <![CDATA[<raw & текст>]]>
# После next(), если kind == xmlCData:
echo x.charData  # "<raw & текст>"
```

---

### `elementName`

```nim
template elementName*(my: XmlParser): string
```

**Допустимо для:** `xmlElementStart`, `xmlElementEnd`, `xmlElementOpen`

**Что делает:** Возвращает имя элемента — имя тега без угловых скобок. Для `xmlElementStart` и `xmlElementEnd` — полное имя тега. Для `xmlElementOpen` — имя тега, атрибуты которого ещё не были полностью разобраны.

```nim
# Ввод: <title>Nim</title>
x.next()  # xmlElementStart
echo x.elementName  # "title"

x.next()  # xmlCharData → "Nim"

x.next()  # xmlElementEnd
echo x.elementName  # "title"
```

```nim
# Ввод: <a href="https://nim-lang.org">
x.next()  # xmlElementOpen (у тега есть атрибуты)
echo x.elementName  # "a"
# не вызывайте charData здесь — неверный тип события!
```

---

### `entityName`

```nim
template entityName*(my: XmlParser): string
```

**Допустимо для:** `xmlEntity`

**Что делает:** Возвращает имя нераспознанной именованной сущности — текст между `&` и `;`. Стандартные XML-сущности (`&lt;`, `&gt;`, `&amp;`, `&apos;`, `&quot;`) и числовые ссылки на символы (`&#65;`, `&#x41;`) декодируются прозрачно и появляются как `xmlCharData`, а не как `xmlEntity`. Событие `xmlEntity` возникает только для действительно неизвестных сущностей, таких как `&nbsp;` или `&copy;`.

```nim
# Ввод: <p>Copyright &copy; 2024</p>
# После приземления на xmlEntity:
echo x.entityName  # "copy"
# Теперь можно вывести "&copy;" или найти символ в таблице HTML-сущностей.
```

---

### `attrKey`

```nim
template attrKey*(my: XmlParser): string
```

**Допустимо для:** `xmlAttribute`

**Что делает:** Возвращает имя атрибута (левая часть пары `ключ="значение"`).

```nim
# Ввод: <img src="photo.png" alt="Фото">
x.next()  # xmlElementOpen → "img"
x.next()  # xmlAttribute
echo x.attrKey    # "src"
echo x.attrValue  # "photo.png"
x.next()  # xmlAttribute
echo x.attrKey    # "alt"
echo x.attrValue  # "Фото"
x.next()  # xmlElementClose
```

---

### `attrValue`

```nim
template attrValue*(my: XmlParser): string
```

**Допустимо для:** `xmlAttribute`

**Что делает:** Возвращает значение атрибута (правая часть пары `ключ="значение"`) — без кавычек и с декодированными стандартными сущностями. Всегда читайте `attrKey` и `attrValue` вместе в рамках одного события `xmlAttribute`.

*(Пример — см. `attrKey` выше.)*

---

### `piName`

```nim
template piName*(my: XmlParser): string
```

**Допустимо для:** `xmlPI`

**Что делает:** Возвращает имя инструкции обработки — идентификатор, стоящий сразу после `<?`. Стандартная декларация `<?xml ... ?>` в начале документа молча поглощается процедурой `next()` и никогда не появляется как событие.

```nim
# Ввод: <?xml-stylesheet type="text/css" href="style.css"?>
# После next(), если kind == xmlPI:
echo x.piName  # "xml-stylesheet"
echo x.piRest  # " type=\"text/css\" href=\"style.css\""
```

---

### `piRest`

```nim
template piRest*(my: XmlParser): string
```

**Допустимо для:** `xmlPI`

**Что делает:** Возвращает всё, что стоит после имени инструкции обработки, вплоть до закрывающего `?>`. Это необработанный остаток инструкции — если вам нужны структурированные данные из неё, разбирайте строку самостоятельно.

*(Пример — см. `piName` выше.)*

---

## Позиция и диагностика

### `getLine`

```nim
proc getLine*(my: XmlParser): int
```

**Что делает:** Возвращает номер строки (начиная с 1) во входном потоке, на которой сейчас находится парсер. Полезно для формирования сообщений об ошибках.

---

### `getColumn`

```nim
proc getColumn*(my: XmlParser): int
```

**Что делает:** Возвращает номер столбца (начиная с 0) во входном потоке, на котором сейчас находится парсер.

---

### `getFilename`

```nim
proc getFilename*(my: XmlParser): string
```

**Что делает:** Возвращает строку с именем файла, переданную в `open()`. Используется внутри `errorMsg()` для формирования корректно отформатированных сообщений об ошибках.

---

### `errorMsg` (событие xmlError)

```nim
proc errorMsg*(my: XmlParser): string
```

**Допустимо для:** событий `xmlError`.

**Что делает:** Возвращает человекочитаемое сообщение об ошибке разбора, описывающее причину возникновения события `xmlError`. Сообщение форматируется следующим образом:

```
имя_файла(строка, столбец) Error: описание
```

Это основной способ отчитываться об ошибках парсера. Всегда вызывайте эту процедуру при обнаружении `xmlError` — не формируйте сообщения об ошибках вручную.

```nim
while true:
  x.next()
  case x.kind
  of xmlError:
    echo x.errorMsg()   # например: "data.xml(12, 5) Error: '>' expected"
    x.next()            # попытка восстановления и продолжения
  of xmlEof: break
  else: discard
```

---

### `errorMsg` (произвольный текст)

```nim
proc errorMsg*(my: XmlParser; msg: string): string
```

**Что делает:** Форматирует произвольную строку `msg` в тот же формат `имя_файла(строка, столбец) Error: msg`, что используется самим парсером. Применяйте это, когда вы обнаруживаете семантическую ошибку в клиентском коде (например, недопустимое вложение тегов) и хотите, чтобы сообщение выглядело единообразно с ошибками парсера.

```nim
if x.elementName != "root":
  echo x.errorMsg("ожидался элемент <root> в качестве корня документа")
  # → "schema.xml(1, 1) Error: ожидался элемент <root> в качестве корня документа"
```

---

### `errorMsgExpected`

```nim
proc errorMsgExpected*(my: XmlParser; tag: string): string
```

**Что делает:** Удобная обёртка вокруг `errorMsg`. Форматирует сообщение вида `<tag> expected` в стандартном формате ошибок парсера. Предназначена специально для ситуации, когда вы ожидали конкретный закрывающий тег, но получили что-то другое — проверку, которую парсер не может выполнить за вас.

```nim
x.next()  # ожидаем </title>
if x.kind != xmlElementEnd or x.elementName != "title":
  echo x.errorMsgExpected("/title")
  # → "index.html(8, 3) Error: </title> expected"
```

---

## Низкоуровневые / оптимизационные процедуры

Эти процедуры открывают прямой доступ к внутренним строковым буферам парсера по ссылке. Они существуют исключительно для производительно-критичного кода, которому нужно избежать лишних копирований строк. В обычных ситуациях предпочитайте именованные шаблоны-аксессоры.

### `rawData`

```nim
proc rawData*(my: var XmlParser): lent string
```

**Что делает:** Возвращает ссылку на основной внутренний строковый буфер парсера (`a`). В зависимости от контекста содержит то же значение, что `charData`, `elementName`, `entityName`, `attrKey` или `piName`. Так как это ссылка типа `lent` (заимствованная), вы не должны хранить её дольше, чем до следующего вызова `next()`.

---

### `rawData2`

```nim
proc rawData2*(my: var XmlParser): lent string
```

**Что делает:** Возвращает ссылку на вторичный внутренний строковый буфер парсера (`b`). В зависимости от контекста содержит то же значение, что `attrValue` или `piRest`. Те же ограничения по времени жизни, что у `rawData`.

---

## Полные рабочие примеры

### Пример 1: Извлечь весь текст из XML-документа

```nim
import std/[streams, parsexml]

var s = newStringStream("""
  <doc>
    <section>Текст первого раздела.</section>
    <!-- проигнорированный комментарий -->
    <section>Текст второго раздела.</section>
  </doc>
""")

var x: XmlParser
open(x, s, "<string>")

while true:
  x.next()
  case x.kind
  of xmlCharData:
    let text = x.charData.strip()
    if text.len > 0:
      echo text
  of xmlEof: break
  of xmlError: echo x.errorMsg()
  else: discard

x.close()
# Вывод:
# Текст первого раздела.
# Текст второго раздела.
```

---

### Пример 2: Собрать все атрибуты конкретного элемента

В этом примере мы собираем все атрибуты каждого тега `<meta>` в HTML-документе — типичная задача при парсинге HTML.

```nim
import std/[streams, parsexml, strutils, tables]

proc collectMetaTags(html: string): seq[Table[string, string]] =
  var s = newStringStream(html)
  var x: XmlParser
  open(x, s, "<html>", {allowUnquotedAttribs, allowEmptyAttribs})

  var inMeta = false
  var current: Table[string, string]

  while true:
    x.next()
    case x.kind
    of xmlElementOpen:
      if cmpIgnoreCase(x.elementName, "meta") == 0:
        inMeta = true
        current = initTable[string, string]()
    of xmlAttribute:
      if inMeta:
        current[x.attrKey] = x.attrValue
    of xmlElementClose:
      if inMeta:
        result.add(current)
        inMeta = false
    of xmlEof: break
    else: discard

  x.close()

let html = """
  <html><head>
    <meta name="description" content="Тестовая страница">
    <meta charset="UTF-8">
  </head></html>
"""

for tag in collectMetaTags(html):
  echo tag
```

---

### Пример 3: Устойчивый к ошибкам цикл разбора (рекомендуемый шаблон)

```nim
import std/[streams, parsexml]

var s = newFileStream("data.xml", fmRead)
var x: XmlParser
open(x, s, "data.xml")

while true:
  x.next()
  case x.kind
  of xmlEof:
    break                              # нормальное завершение
  of xmlError:
    stderr.writeLine x.errorMsg()     # выводим ошибку и пробуем продолжить
  of xmlElementStart:
    echo "ОТКРЫТ: ", x.elementName
  of xmlElementEnd:
    echo "ЗАКРЫТ: ", x.elementName
  of xmlElementOpen:
    echo "ТЕГ С АТРИБУТАМИ: ", x.elementName
  of xmlAttribute:
    echo "  АТРИБУТ: ", x.attrKey, " = ", x.attrValue
  of xmlElementClose:
    echo "  >"
  of xmlCharData:
    let t = x.charData.strip()
    if t.len > 0: echo "ТЕКСТ:  ", t
  of xmlCData:
    echo "CDATA: ", x.charData
  of xmlComment:
    discard  # игнорируется, если не задана опция reportComments
  of xmlEntity:
    echo "СУЩНОСТЬ: &", x.entityName, ";"
  of xmlPI:
    echo "ИНСТРУКЦИЯ: ", x.piName, " | ", x.piRest
  of xmlSpecial:
    echo "СПЕЦИАЛЬНЫЙ: ", x.charData
  of xmlWhitespace:
    discard  # игнорируется, если не задана опция reportWhitespace

x.close()
```

---

### Пример 4: Проверка закрывающего тега (клиентская проверка парности тегов)

Поскольку парсер не проверяет соответствие тегов, вот простой шаблон для самостоятельной реализации этой проверки:

```nim
import std/[streams, parsexml]

proc parseTitle(x: var XmlParser): string =
  # Вызывается сразу после того, как мы обработали xmlElementStart для "title"
  x.next()
  while x.kind == xmlCharData:
    result.add(x.charData)
    x.next()
  if x.kind == xmlElementEnd and x.elementName == "title":
    discard  # всё хорошо
  else:
    echo x.errorMsgExpected("/title")

var s = newStringStream("<html><head><title>Привет, мир</title></head></html>")
var x: XmlParser
open(x, s, "")

while true:
  x.next()
  case x.kind
  of xmlElementStart:
    if x.elementName == "title":
      echo "Заголовок: ", parseTitle(x)
  of xmlEof: break
  else: discard

x.close()
# Вывод:
# Заголовок: Привет, мир
```
