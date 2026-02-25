# Справочник модуля `htmlparser` (Nim)

> Модуль стандартной библиотеки Nim для разбора «живого» HTML.  
> Парсит неидеальный HTML-код и возвращает дерево `XmlNode`.
>
> ⚠️ **Устаревший модуль.** С текущей версии помечен как `deprecated`.  
> Рекомендуется установить: `nimble install htmlparser` и импортировать `pkg/htmlparser`.

---

## Содержание

1. [Обзор и принцип работы](#обзор-и-принцип-работы)
2. [Типы и константы](#типы-и-константы)
3. [Парсинг HTML](#парсинг-html)
   - [parseHtml (Stream + ошибки)](#parsehtmls-stream-filename-string-errors-var-seqstring-xmlnode)
   - [parseHtml (Stream)](#parsehtmls-stream-xmlnode)
   - [parseHtml (строка)](#parsehtmlhtml-string-xmlnode)
   - [loadHtml (файл + ошибки)](#loadhtmlpath-string-errors-var-seqstring-xmlnode)
   - [loadHtml (файл)](#loadhtmlpath-string-xmlnode)
4. [Работа с тегами](#работа-с-тегами)
   - [htmlTag (XmlNode)](#htmltagnxmlnode-htmltag)
   - [htmlTag (строка)](#htmltags-string-htmltag)
5. [Работа с HTML-сущностями](#работа-с-html-сущностями)
   - [runeToEntity](#runetoentityrune-rune-string)
   - [entityToRune](#entitytoruneentity-string-rune)
   - [entityToUtf8](#entitytoutf8entity-string-string)
6. [Константа `tagToStr`](#константа-tagtostr)
7. [Перечисление `HtmlTag`](#перечисление-htmltag)
8. [Практические примеры](#практические-примеры)
9. [Быстрая шпаргалка](#быстрая-шпаргалка)

---

## Обзор и принцип работы

Модуль `htmlparser` разбирает HTML-документы с любым количеством ошибок и возвращает дерево `XmlNode` из модуля `std/xmltree`. Все теги в результирующем дереве приводятся к **нижнему регистру**.

Результирующий `XmlNode` использует поле `clientData` внутри модуля — клиентский код не должен его модифицировать.

```nim
import std/htmlparser
import std/xmltree

let html = parseHtml("<html><body><p>Привет!</p></body></html>")
echo html  # выведет нормализованный XML
```

---

## Типы и константы

### `HtmlTag`

```nim
type HtmlTag* = enum
  tagUnknown, tagA, tagAbbr, tagAcronym, tagAddress, ...
```

Перечисление всех поддерживаемых HTML-тегов в алфавитном порядке. Включает как актуальные, так и устаревшие теги (помечены в документации как `deprecated`).

Специальные значения:
- `tagUnknown` — неизвестный или нераспознанный тег

### `tagToStr`

```nim
const tagToStr* = ["a", "abbr", "acronym", ...]
```

Массив строк, сопоставленный с `HtmlTag` (начиная с `tagA`). Позволяет получить строковое имя тега по его значению перечисления.

```nim
echo tagToStr[tagDiv.ord - 1]  # "div"
```

> Индексация начинается с `tagA` (индекс 0 соответствует `tagA`, а не `tagUnknown`).

---

## Парсинг HTML

### `parseHtml(s: Stream, filename: string, errors: var seq[string]): XmlNode`

```nim
proc parseHtml*(s: Stream, filename: string,
                errors: var seq[string]): XmlNode
```

Парсит HTML из потока `s`. Имя файла `filename` используется в сообщениях об ошибках. Все ошибки парсинга добавляются в список `errors`.

**Параметры:**
- `s` — входной поток (`Stream`) с HTML-содержимым
- `filename` — имя файла (для диагностики)
- `errors` — изменяемый список строк, куда записываются ошибки

**Возвращает:** корневой `XmlNode` документа.

```nim
import std/[htmlparser, xmltree, streams]

var errors: seq[string] = @[]
let stream = newStringStream("<html><body><p>Текст</p></body></html>")
let doc = parseHtml(stream, "test.html", errors)

if errors.len > 0:
  for e in errors:
    echo "Ошибка: ", e
else:
  echo "Парсинг успешен"

echo doc
```

---

### `parseHtml(s: Stream): XmlNode`

```nim
proc parseHtml*(s: Stream): XmlNode
```

Упрощённая версия: парсит HTML из потока, игнорируя все ошибки. Внутри использует имя `"unknown_html_doc"`.

```nim
import std/[htmlparser, streams]

let s = newStringStream("<div class='box'><p>Hello</p></div>")
let doc = parseHtml(s)
echo doc
```

---

### `parseHtml(html: string): XmlNode`

```nim
proc parseHtml*(html: string): XmlNode
```

Самый удобный вариант: принимает HTML прямо в виде строки, ошибки игнорируются.

```nim
import std/htmlparser

let doc = parseHtml("""
  <html>
    <head><title>Тест</title></head>
    <body>
      <h1>Заголовок</h1>
      <p>Параграф с <b>жирным</b> текстом.</p>
    </body>
  </html>
""")

echo doc
```

---

### `loadHtml(path: string, errors: var seq[string]): XmlNode`

```nim
proc loadHtml*(path: string, errors: var seq[string]): XmlNode
```

Загружает HTML из файла по пути `path` и парсит его. Ошибки парсинга добавляются в `errors`. Если файл не найден — выбрасывает `IOError`.

**Параметры:**
- `path` — путь к HTML-файлу
- `errors` — список для сбора ошибок парсинга

```nim
import std/htmlparser

var errors: seq[string] = @[]
let doc = loadHtml("index.html", errors)

for e in errors:
  echo "Предупреждение: ", e

echo "Документ загружен"
```

---

### `loadHtml(path: string): XmlNode`

```nim
proc loadHtml*(path: string): XmlNode
```

Упрощённый вариант: загружает HTML из файла, игнорируя все ошибки.

```nim
import std/htmlparser

let doc = loadHtml("page.html")
echo doc
```

---

## Работа с тегами

### `htmlTag(n: XmlNode): HtmlTag`

```nim
proc htmlTag*(n: XmlNode): HtmlTag
```

Возвращает тег узла `n` как значение перечисления `HtmlTag`. Результат кешируется в `clientData` узла для ускорения повторных вызовов.

```nim
import std/[htmlparser, xmltree]

let doc = parseHtml("<div><p>Текст</p></div>")
for node in doc:
  if node.kind == xnElement:
    case node.htmlTag
    of tagDiv: echo "Нашли div"
    of tagP:   echo "Нашли параграф"
    else: discard
```

---

### `htmlTag(s: string): HtmlTag`

```nim
proc htmlTag*(s: string): HtmlTag
```

Конвертирует строку `s` в значение `HtmlTag`. Нечувствителен к регистру. Если тег не распознан — возвращает `tagUnknown`.

```nim
import std/htmlparser

echo htmlTag("div")     # tagDiv
echo htmlTag("DIV")     # tagDiv (регистронезависимо)
echo htmlTag("p")       # tagP
echo htmlTag("custom")  # tagUnknown
```

---

## Работа с HTML-сущностями

### `runeToEntity(rune: Rune): string`

```nim
proc runeToEntity*(rune: Rune): string
```

Конвертирует символ Unicode (`Rune`) в числовую HTML-сущность вида `#N`.  
Возвращает пустую строку, если `rune.ord <= 0`.

```nim
import std/[htmlparser, unicode]

echo runeToEntity(Rune(0))         # ""
echo runeToEntity(Rune(-1))        # ""
echo runeToEntity("Ü".runeAt(0))   # "#220"
echo runeToEntity("∈".runeAt(0))   # "#8712"
echo runeToEntity("©".runeAt(0))   # "#169"
```

**Применение:** экранирование символов перед вставкой в HTML-документ.

```nim
import std/[htmlparser, unicode]

proc escapeChar(c: char): string =
  let r = Rune(c.ord)
  let entity = runeToEntity(r)
  if entity.len > 0: "&" & entity & ";"
  else: $c
```

---

### `entityToRune(entity: string): Rune`

```nim
proc entityToRune*(entity: string): Rune
```

Конвертирует имя HTML-сущности или числовой код в `Rune` (кодовую точку Unicode).  
Возвращает `Rune(0)` если сущность неизвестна.

**Поддерживаемые форматы входной строки** (без окружающих `&` и `;`):
- Именованная: `"Uuml"`, `"gt"`, `"amp"`, `"quest"` и т.д.
- Десятичная: `"#220"`, `"#63"`
- Шестнадцатеричная: `"#x000DC"`, `"#X3a3"` (нечувствительно к регистру `x`)

```nim
import std/[htmlparser, unicode]

echo entityToRune("")        # Rune(0) — слишком коротко
echo entityToRune("a")       # Rune(0) — неизвестно
echo entityToRune("gt")      # '>'.runeAt(0)
echo entityToRune("Uuml")    # 'Ü'.runeAt(0)
echo entityToRune("quest")   # '?'.runeAt(0)
echo entityToRune("#63")     # '?'.runeAt(0)
echo entityToRune("#x003F")  # '?'.runeAt(0)
```

---

### `entityToUtf8(entity: string): string`

```nim
proc entityToUtf8*(entity: string): string
```

Конвертирует HTML-сущность в строку UTF-8. Это надстройка над `entityToRune` — удобна там, где нужна строка, а не кодовая точка.  
Возвращает `""` если сущность неизвестна.

> **Примечание:** HTML-парсер уже автоматически преобразует сущности в UTF-8 при парсинге. Эта функция нужна для ручного декодирования.

```nim
import std/htmlparser

echo entityToUtf8("")       # ""
echo entityToUtf8("a")      # ""
echo entityToUtf8("gt")     # ">"
echo entityToUtf8("Uuml")   # "Ü"
echo entityToUtf8("quest")  # "?"
echo entityToUtf8("#63")    # "?"
echo entityToUtf8("Sigma")  # "Σ"
echo entityToUtf8("#931")   # "Σ"
echo entityToUtf8("#x3A3")  # "Σ"
echo entityToUtf8("#X3a3")  # "Σ"
```

---

## Перечисление `HtmlTag`

Полный список значений `HtmlTag` (алфавитный порядок):

| Значение | HTML-тег | Примечание |
|----------|----------|------------|
| `tagUnknown` | — | Неизвестный тег |
| `tagA` | `<a>` | |
| `tagAbbr` | `<abbr>` | Устарел |
| `tagAcronym` | `<acronym>` | |
| `tagAddress` | `<address>` | |
| `tagApplet` | `<applet>` | Устарел |
| `tagArea` | `<area>` | |
| `tagArticle` | `<article>` | |
| `tagAside` | `<aside>` | |
| `tagAudio` | `<audio>` | |
| `tagB` | `<b>` | |
| `tagBase` | `<base>` | |
| `tagBdi` | `<bdi>` | |
| `tagBdo` | `<bdo>` | Устарел |
| `tagBasefont` | `<basefont>` | Устарел |
| `tagBig` | `<big>` | |
| `tagBlockquote` | `<blockquote>` | |
| `tagBody` | `<body>` | |
| `tagBr` | `<br>` | |
| `tagButton` | `<button>` | |
| `tagCanvas` | `<canvas>` | |
| `tagCaption` | `<caption>` | |
| `tagCenter` | `<center>` | Устарел |
| `tagCite` | `<cite>` | |
| `tagCode` | `<code>` | |
| `tagCol` | `<col>` | |
| `tagColgroup` | `<colgroup>` | |
| `tagCommand` | `<command>` | |
| `tagDatalist` | `<datalist>` | |
| `tagDd` | `<dd>` | |
| `tagDel` | `<del>` | |
| `tagDetails` | `<details>` | |
| `tagDfn` | `<dfn>` | |
| `tagDialog` | `<dialog>` | |
| `tagDiv` | `<div>` | |
| `tagDir` | `<dir>` | Устарел |
| `tagDl` | `<dl>` | |
| `tagDt` | `<dt>` | |
| `tagEm` | `<em>` | |
| `tagEmbed` | `<embed>` | |
| `tagFieldset` | `<fieldset>` | |
| `tagFigcaption` | `<figcaption>` | |
| `tagFigure` | `<figure>` | |
| `tagFont` | `<font>` | Устарел |
| `tagFooter` | `<footer>` | |
| `tagForm` | `<form>` | |
| `tagFrame` | `<frame>` | |
| `tagFrameset` | `<frameset>` | Устарел |
| `tagH1`–`tagH6` | `<h1>`–`<h6>` | |
| `tagHead` | `<head>` | |
| `tagHeader` | `<header>` | |
| `tagHgroup` | `<hgroup>` | |
| `tagHtml` | `<html>` | |
| `tagHr` | `<hr>` | |
| `tagI` | `<i>` | |
| `tagIframe` | `<iframe>` | Устарел |
| `tagImg` | `<img>` | |
| `tagInput` | `<input>` | |
| `tagIns` | `<ins>` | |
| `tagIsindex` | `<isindex>` | Устарел |
| `tagKbd` | `<kbd>` | |
| `tagKeygen` | `<keygen>` | |
| `tagLabel` | `<label>` | |
| `tagLegend` | `<legend>` | |
| `tagLi` | `<li>` | |
| `tagLink` | `<link>` | |
| `tagMap` | `<map>` | |
| `tagMark` | `<mark>` | |
| `tagMenu` | `<menu>` | Устарел |
| `tagMeta` | `<meta>` | |
| `tagMeter` | `<meter>` | |
| `tagNav` | `<nav>` | |
| `tagNobr` | `<nobr>` | Устарел |
| `tagNoframes` | `<noframes>` | Устарел |
| `tagNoscript` | `<noscript>` | |
| `tagObject` | `<object>` | |
| `tagOl` | `<ol>` | |
| `tagOptgroup` | `<optgroup>` | |
| `tagOption` | `<option>` | |
| `tagOutput` | `<output>` | |
| `tagP` | `<p>` | |
| `tagParam` | `<param>` | |
| `tagPre` | `<pre>` | |
| `tagProgress` | `<progress>` | |
| `tagQ` | `<q>` | |
| `tagRp` | `<rp>` | |
| `tagRt` | `<rt>` | |
| `tagRuby` | `<ruby>` | |
| `tagS` | `<s>` | Устарел |
| `tagSamp` | `<samp>` | |
| `tagScript` | `<script>` | |
| `tagSection` | `<section>` | |
| `tagSelect` | `<select>` | |
| `tagSmall` | `<small>` | |
| `tagSource` | `<source>` | |
| `tagSpan` | `<span>` | |
| `tagStrike` | `<strike>` | Устарел |
| `tagStrong` | `<strong>` | |
| `tagStyle` | `<style>` | |
| `tagSub` | `<sub>` | |
| `tagSummary` | `<summary>` | |
| `tagSup` | `<sup>` | |
| `tagTable` | `<table>` | |
| `tagTbody` | `<tbody>` | |
| `tagTd` | `<td>` | |
| `tagTextarea` | `<textarea>` | |
| `tagTfoot` | `<tfoot>` | |
| `tagTh` | `<th>` | |
| `tagThead` | `<thead>` | |
| `tagTime` | `<time>` | |
| `tagTitle` | `<title>` | |
| `tagTr` | `<tr>` | |
| `tagTrack` | `<track>` | |
| `tagTt` | `<tt>` | |
| `tagU` | `<u>` | Устарел |
| `tagUl` | `<ul>` | |
| `tagVar` | `<var>` | |
| `tagVideo` | `<video>` | |
| `tagWbr` | `<wbr>` | |

---

## Практические примеры

### Пример 1: Извлечение всех ссылок

```nim
import std/[htmlparser, xmltree, strtabs]

let html = loadHtml("page.html")

for a in html.findAll("a"):
  if a.attrs != nil and a.attrs.hasKey("href"):
    echo a.attrs["href"]
```

---

### Пример 2: Замена расширений в ссылках (из документации)

```nim
import std/[htmlparser, xmltree, strtabs, os, strutils]

proc transformHyperlinks() =
  let html = loadHtml("input.html")

  for a in html.findAll("a"):
    if a.attrs != nil and a.attrs.hasKey("href"):
      let (dir, filename, ext) = splitFile(a.attrs["href"])
      if cmpIgnoreCase(ext, ".rst") == 0:
        a.attrs["href"] = dir / filename & ".html"

  writeFile("output.html", $html)

transformHyperlinks()
```

---

### Пример 3: Фильтрация по типу тега с `HtmlTag`

```nim
import std/[htmlparser, xmltree]

let doc = parseHtml("""
  <html><body>
    <h1>Заголовок</h1>
    <p>Параграф 1</p>
    <p>Параграф 2</p>
    <ul>
      <li>Пункт 1</li>
      <li>Пункт 2</li>
    </ul>
  </body></html>
""")

proc collectByTag(node: XmlNode, tag: HtmlTag, result: var seq[XmlNode]) =
  if node.kind == xnElement and node.htmlTag == tag:
    result.add(node)
  for child in node:
    collectByTag(child, tag, result)

var paragraphs: seq[XmlNode] = @[]
collectByTag(doc, tagP, paragraphs)

for p in paragraphs:
  echo p.innerText
```

---

### Пример 4: Ручное декодирование сущностей

```nim
import std/htmlparser

# Сущности в разных форматах
let entities = ["gt", "lt", "amp", "Uuml", "Sigma", "#169", "#x1F600"]
for e in entities:
  let decoded = entityToUtf8(e)
  if decoded.len > 0:
    echo "&", e, "; → ", decoded
  else:
    echo "&", e, "; → неизвестно"
```

---

### Пример 5: Парсинг с обработкой ошибок

```nim
import std/[htmlparser, streams]

let brokenHtml = """
  <html>
    <body>
      <p>Незакрытый параграф
      <div>Вложен неправильно</p></div>
    </body>
  </html>
"""

var errors: seq[string] = @[]
let doc = parseHtml(newStringStream(brokenHtml), "broken.html", errors)

echo "Ошибок парсинга: ", errors.len
for e in errors:
  echo "  - ", e
echo "Документ всё равно разобран:"
echo doc
```

---

### Пример 6: Проверка тега по строке

```nim
import std/htmlparser

proc isFormElement(tagName: string): bool =
  htmlTag(tagName) in {tagInput, tagSelect, tagTextarea, tagButton}

echo isFormElement("input")    # true
echo isFormElement("div")      # false
echo isFormElement("TEXTAREA") # true (регистронезависимо)
```

---

## Быстрая шпаргалка

| Задача | Функция |
|--------|---------|
| Разобрать HTML-строку | `parseHtml(htmlString)` |
| Разобрать HTML из потока | `parseHtml(stream)` |
| Разобрать с ловлей ошибок | `parseHtml(stream, filename, errors)` |
| Загрузить HTML из файла | `loadHtml("path.html")` |
| Загрузить с ловлей ошибок | `loadHtml("path.html", errors)` |
| Тег узла как enum | `node.htmlTag` |
| Строка → enum тега | `htmlTag("div")` → `tagDiv` |
| Rune → HTML-сущность | `runeToEntity(rune)` → `"#220"` |
| Сущность → Rune | `entityToRune("Uuml")` |
| Сущность → UTF-8 строка | `entityToUtf8("Uuml")` → `"Ü"` |
| Enum тега → строка | `tagToStr[tagDiv.ord - 1]` |

### Форматы HTML-сущностей для `entityToRune`/`entityToUtf8`

| Формат | Пример | Описание |
|--------|--------|----------|
| Именованная | `"Uuml"` | Без `&` и `;` |
| Десятичная | `"#220"` | С префиксом `#` |
| Шестнадцатеричная | `"#xDC"` или `"#XDC"` | С префиксом `#x` или `#X` |
