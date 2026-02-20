# Модуль Nim `std/htmlparser` — Полный справочник

> **Установка:** `nimble install htmlparser`
>
> Парсер HTML, устойчивый к «дикому» реальному HTML. Строит дерево `XmlNode`, совместимое с `std/xmltree`.  
> Парсер ориентирован на браузерное восстановление после ошибок, а не на строгое соответствие стандарту. Поведение может измениться в будущих версиях по мере уточнения понятия «дикий HTML».

---

## Содержание

1. [Обзор](#обзор)
2. [Типы](#типы)
3. [Константы](#константы)
4. [Идентификация тегов](#идентификация-тегов)
5. [Преобразование HTML-сущностей](#преобразование-html-сущностей)
6. [Разбор HTML](#разбор-html)
7. [Загрузка из файла](#загрузка-из-файла)
8. [Работа с результирующим деревом](#работа-с-результирующим-деревом)
9. [Перечисление HtmlTag — полный список](#перечисление-htmltag--полный-список)
10. [Таблица быстрого доступа](#таблица-быстрого-доступа)

---

## Обзор

`std/htmlparser` читает HTML из строки, потока или файла и возвращает дерево `XmlNode`. Поскольку парсер намеренно снисходителен, он справляется с HTML, который отклонили бы строгие XML-парсеры: незакрытые теги, атрибуты без кавычек, пустые атрибуты и прочее.

```nim
import std/htmlparser
import std/xmltree

# Разбор строки HTML
let tree = parseHtml("<h1>Привет <b>Мир</b></h1>")
echo tree   # → <h1>Привет <b>Мир</b></h1>

# Загрузка файла и вывод как XHTML
echo loadHtml("page.html")
```

> **Замечание:** Все имена тегов в результирующем дереве — **строчными буквами**.  
> **Замечание:** Поле `clientData` каждого `XmlNode` используется парсером для кэширования значений `HtmlTag`. Не используйте `clientData` самостоятельно.

---

## Типы

### `HtmlTag`

```nim
type HtmlTag* = enum
  tagUnknown,     ## Любой тег, не входящий в список известных
  tagA,           ## <a>
  tagAbbr,        ## <abbr>
  tagAcronym,     ## <acronym>
  tagAddress,     ## <address>
  # … (полный список — в конце документа)
  tagWbr          ## <wbr>
```

Перечисление, охватывающее все известные парсеру HTML-теги в алфавитном порядке. Используется для идентификации тегов в разобранном дереве без хрупких строковых сравнений.

```nim
import std/htmlparser, std/xmltree

let tree = parseHtml("<p>Привет</p>")
for node in tree:
  if node.kind == xnElement and node.htmlTag == tagP:
    echo "Найден абзац: ", node
```

---

## Константы

### `tagToStr`

```nim
const tagToStr*: array[…, string]
```

Отображает каждое значение перечисления `HtmlTag` (начиная с `tagA`) на строчное имя HTML-тега. Используется для обратного преобразования значений перечисления в строки.

```nim
echo tagToStr[tagA.ord - 1]    # → "a"
echo tagToStr[tagDiv.ord - 1]  # → "div"
```

> **Замечание:** Массив начинается с `tagA` (индекс 0). Для `tagUnknown` записи нет.

---

### `InlineTags`

```nim
const InlineTags*: set[HtmlTag]
```

Множество всех HTML-тегов, считающихся **строчными** элементами (размещаются в потоке текста). Включает: `a`, `abbr`, `acronym`, `applet`, `b`, `basefont`, `bdo`, `big`, `br`, `button`, `cite`, `code`, `del`, `dfn`, `em`, `font`, `i`, `img`, `ins`, `input`, `iframe`, `kbd`, `label`, `map`, `object`, `q`, `s`, `samp`, `script`, `select`, `small`, `span`, `strike`, `strong`, `sub`, `sup`, `textarea`, `tt`, `u`, `var`, `wbr`.

```nim
if node.htmlTag in InlineTags:
  echo node.tag, " — строчный элемент"
```

---

### `BlockTags`

```nim
const BlockTags*: set[HtmlTag]
```

Множество всех HTML-тегов, считающихся **блочными** элементами. Включает: `address`, `blockquote`, `center`, `del`, `dir`, `div`, `dl`, `fieldset`, `form`, `h1`–`h6`, `hr`, `ins`, `isindex`, `menu`, `noframes`, `noscript`, `ol`, `p`, `pre`, `table`, `ul`.

```nim
if node.htmlTag in BlockTags:
  echo node.tag, " — блочный элемент"
```

---

### `SingleTags`

```nim
const SingleTags*: set[HtmlTag]
```

Множество **пустых (void) / самозакрывающихся** тегов, которые никогда не имеют дочерних элементов. Парсер не ожидает закрывающего тега для них. Включает: `area`, `base`, `basefont`, `br`, `col`, `frame`, `hr`, `img`, `isindex`, `link`, `meta`, `param`, `source`, `wbr`.

```nim
if node.htmlTag in SingleTags:
  echo node.tag, " — void-элемент (закрывающий тег не нужен)"
```

---

## Идентификация тегов

### `htmlTag(n: XmlNode)`

```nim
proc htmlTag*(n: XmlNode): HtmlTag
```

Возвращает значение перечисления `HtmlTag` для разобранного `XmlNode`. Результат кэшируется в `n.clientData` после первого вызова, поэтому повторные вызовы не требуют накладных расходов.

```nim
import std/htmlparser, std/xmltree

let tree = parseHtml("<div><p>текст</p></div>")
for node in tree:
  case node.htmlTag
  of tagDiv: echo "найден div"
  of tagP:   echo "найден абзац"
  else: discard
```

---

### `htmlTag(s: string)`

```nim
proc htmlTag*(s: string): HtmlTag
```

Преобразует произвольную строку в соответствующее значение `HtmlTag`. Сравнение **нечувствительно к регистру**. Возвращает `tagUnknown`, если строка не является известным HTML-тегом.

```nim
echo htmlTag("DIV")    # → tagDiv
echo htmlTag("span")   # → tagSpan
echo htmlTag("foobar") # → tagUnknown
```

---

## Преобразование HTML-сущностей

### `runeToEntity`

```nim
proc runeToEntity*(rune: Rune): string
```

Преобразует Unicode `Rune` в строку с **числовой HTML-сущностью** (формат `#NNN`). Возвращает `""` для `Rune(0)` и любого отрицательного руна.

```nim
import std/unicode, std/htmlparser

echo runeToEntity(Rune(0))           # → ""
echo runeToEntity(Rune(-1))          # → ""
echo runeToEntity("Ü".runeAt(0))    # → "#220"
echo runeToEntity("∈".runeAt(0))    # → "#8712"
```

---

### `entityToRune`

```nim
proc entityToRune*(entity: string): Rune
```

Преобразует имя HTML-сущности или числовую ссылку в `Rune`. Возвращает `Rune(0)` для неизвестных или некорректных сущностей.

Поддерживаемые форматы:
- Именованные сущности: `"gt"`, `"Uuml"`, `"amp"`, `"Sigma"` и т. д.
- Десятичные числовые: `"#63"`, `"#931"`, `"#0931"`
- Шестнадцатеричные числовые: `"#x3A3"`, `"#x03A3"`, `"#X3a3"` (`x` нечувствительно к регистру)

Разделители `&` и `;` **не включаются** — передавайте только содержимое между ними.

```nim
import std/unicode, std/htmlparser

echo entityToRune("")        # → Rune(0)
echo entityToRune("gt")      # → Rune('>')
echo entityToRune("Uuml")    # → Rune('Ü')
echo entityToRune("#63")     # → Rune('?')
echo entityToRune("#x3A3")   # → Rune('Σ')
```

---

### `entityToUtf8`

```nim
proc entityToUtf8*(entity: string): string
```

Преобразует имя HTML-сущности или числовую ссылку в **UTF-8 строку**. Возвращает `""` для неизвестных сущностей. Это удобная обёртка над `entityToRune` + `toUTF8`.

Парсер HTML автоматически применяет это преобразование во время разбора; используйте эту функцию, когда нужно декодировать сущности вручную.

```nim
echo entityToUtf8("")        # → ""
echo entityToUtf8("gt")      # → ">"
echo entityToUtf8("Uuml")    # → "Ü"
echo entityToUtf8("quest")   # → "?"
echo entityToUtf8("#63")     # → "?"
echo entityToUtf8("#931")    # → "Σ"
echo entityToUtf8("#x3A3")   # → "Σ"
echo entityToUtf8("#X3a3")   # → "Σ"
```

---

## Разбор HTML

### `parseHtml(s, filename, errors)`

```nim
proc parseHtml*(s: Stream, filename: string,
                errors: var seq[string]): XmlNode
```

Разбирает HTML из открытого потока `s`. Параметр `filename` используется только в сообщениях об ошибках. Каждая восстановимая ошибка разбора добавляется строкой в `errors`.

Возвращает корневой `XmlNode` разобранного документа. Если документ содержит единственный корневой элемент — он возвращается напрямую; в противном случае возвращается синтетический элемент-обёртка `"document"`.

```nim
import std/streams, std/htmlparser

var errors: seq[string]
let s = newStringStream("<p>Привет <b>Мир")
let tree = parseHtml(s, "my_doc.html", errors)
for e in errors: echo "Ошибка разбора: ", e
echo tree
```

---

### `parseHtml(s: Stream)`

```nim
proc parseHtml*(s: Stream): XmlNode
```

Разбирает HTML из потока, молча отбрасывая все ошибки разбора.

```nim
import std/streams, std/htmlparser

let tree = parseHtml(newStringStream("<ul><li>A<li>B<li>C"))
echo tree
```

---

### `parseHtml(html: string)`

```nim
proc parseHtml*(html: string): XmlNode
```

Разбирает HTML непосредственно из строки. Все ошибки разбора игнорируются. Это наиболее удобная перегрузка для быстрого использования.

```nim
import std/htmlparser, std/xmltree

let tree = parseHtml("""
  <html>
    <body>
      <h1>Заголовок</h1>
      <p>Абзац со <a href="https://nim-lang.org">ссылкой</a>.</p>
    </body>
  </html>
""")

for a in tree.findAll("a"):
  echo a.attrs["href"]
# → https://nim-lang.org
```

---

## Загрузка из файла

### `loadHtml(path, errors)`

```nim
proc loadHtml*(path: string, errors: var seq[string]): XmlNode
```

Открывает файл по пути `path`, разбирает его HTML-содержимое и возвращает дерево `XmlNode`. Каждая ошибка разбора добавляется в `errors`. Бросает `IOError`, если файл не удаётся открыть.

```nim
import std/htmlparser

var errors: seq[string]
let tree = loadHtml("page.html", errors)
if errors.len > 0:
  echo "Предупреждения при разборе:"
  for e in errors: echo "  ", e
echo tree
```

---

### `loadHtml(path: string)`

```nim
proc loadHtml*(path: string): XmlNode
```

Открывает и разбирает HTML-файл, игнорируя все ошибки разбора. Бросает `IOError`, если файл не удаётся прочитать.

```nim
import std/htmlparser
echo loadHtml("mydirty.html")
```

---

## Работа с результирующим деревом

Возвращаемый `XmlNode` — стандартный узел `std/xmltree`. Все процедуры `xmltree` работают с ним в обычном режиме.

```nim
import std/htmlparser, std/xmltree, std/strtabs

let tree = parseHtml("""
  <html><body>
    <a href="page.rst">ReST страница</a>
    <a href="other.html">HTML страница</a>
  </body></html>
""")

# Обход всех тегов <a>
for a in tree.findAll("a"):
  echo a.attrs.getOrDefault("href", "(нет href)")

# Проверка через HtmlTag
for node in tree:
  if node.kind == xnElement:
    echo node.tag, " → ", node.htmlTag

# Модификация атрибутов
for a in tree.findAll("a"):
  if a.attrs.hasKey("href") and a.attrs["href"].endsWith(".rst"):
    a.attrs["href"] = a.attrs["href"][0..^5] & ".html"

# Сериализация обратно в строку HTML
echo $tree
```

### Практический пример — преобразование гиперссылок

```nim
import std/htmlparser, std/xmltree, std/strtabs, std/os, std/strutils

proc transformHyperlinks() =
  let html = loadHtml("input.html")

  for a in html.findAll("a"):
    if a.attrs.hasKey("href"):
      let (dir, filename, ext) = splitFile(a.attrs["href"])
      if cmpIgnoreCase(ext, ".rst") == 0:
        a.attrs["href"] = dir / filename & ".html"

  writeFile("output.html", $html)
```

---

## Перечисление HtmlTag — полный список

| Значение | HTML-тег | Примечание |
|---|---|---|
| `tagUnknown` | *(любой неизвестный тег)* | |
| `tagA` | `<a>` | |
| `tagAbbr` | `<abbr>` | устаревший по комментарию в модуле |
| `tagAcronym` | `<acronym>` | |
| `tagAddress` | `<address>` | |
| `tagApplet` | `<applet>` | устаревший |
| `tagArea` | `<area>` | void |
| `tagArticle` | `<article>` | |
| `tagAside` | `<aside>` | |
| `tagAudio` | `<audio>` | |
| `tagB` | `<b>` | |
| `tagBase` | `<base>` | void |
| `tagBdi` | `<bdi>` | |
| `tagBdo` | `<bdo>` | устаревший по комментарию в модуле |
| `tagBasefont` | `<basefont>` | устаревший, void |
| `tagBig` | `<big>` | |
| `tagBlockquote` | `<blockquote>` | |
| `tagBody` | `<body>` | |
| `tagBr` | `<br>` | void |
| `tagButton` | `<button>` | |
| `tagCanvas` | `<canvas>` | |
| `tagCaption` | `<caption>` | |
| `tagCenter` | `<center>` | устаревший |
| `tagCite` | `<cite>` | |
| `tagCode` | `<code>` | |
| `tagCol` | `<col>` | void |
| `tagColgroup` | `<colgroup>` | |
| `tagCommand` | `<command>` | |
| `tagDatalist` | `<datalist>` | |
| `tagDd` | `<dd>` | |
| `tagDel` | `<del>` | |
| `tagDetails` | `<details>` | |
| `tagDfn` | `<dfn>` | |
| `tagDialog` | `<dialog>` | |
| `tagDiv` | `<div>` | |
| `tagDir` | `<dir>` | устаревший |
| `tagDl` | `<dl>` | |
| `tagDt` | `<dt>` | |
| `tagEm` | `<em>` | |
| `tagEmbed` | `<embed>` | |
| `tagFieldset` | `<fieldset>` | |
| `tagFigcaption` | `<figcaption>` | |
| `tagFigure` | `<figure>` | |
| `tagFont` | `<font>` | устаревший |
| `tagFooter` | `<footer>` | |
| `tagForm` | `<form>` | |
| `tagFrame` | `<frame>` | void |
| `tagFrameset` | `<frameset>` | устаревший |
| `tagH1`–`tagH6` | `<h1>`–`<h6>` | |
| `tagHead` | `<head>` | |
| `tagHeader` | `<header>` | |
| `tagHgroup` | `<hgroup>` | |
| `tagHtml` | `<html>` | |
| `tagHr` | `<hr>` | void |
| `tagI` | `<i>` | |
| `tagIframe` | `<iframe>` | устаревший по комментарию в модуле |
| `tagImg` | `<img>` | void |
| `tagInput` | `<input>` | void |
| `tagIns` | `<ins>` | |
| `tagIsindex` | `<isindex>` | устаревший, void |
| `tagKbd` | `<kbd>` | |
| `tagKeygen` | `<keygen>` | |
| `tagLabel` | `<label>` | |
| `tagLegend` | `<legend>` | |
| `tagLi` | `<li>` | |
| `tagLink` | `<link>` | void |
| `tagMap` | `<map>` | |
| `tagMark` | `<mark>` | |
| `tagMenu` | `<menu>` | устаревший |
| `tagMeta` | `<meta>` | void |
| `tagMeter` | `<meter>` | |
| `tagNav` | `<nav>` | |
| `tagNobr` | `<nobr>` | устаревший |
| `tagNoframes` | `<noframes>` | устаревший |
| `tagNoscript` | `<noscript>` | |
| `tagObject` | `<object>` | |
| `tagOl` | `<ol>` | |
| `tagOptgroup` | `<optgroup>` | |
| `tagOption` | `<option>` | |
| `tagOutput` | `<o>` | |
| `tagP` | `<p>` | |
| `tagParam` | `<param>` | void |
| `tagPre` | `<pre>` | |
| `tagProgress` | `<progress>` | |
| `tagQ` | `<q>` | |
| `tagRp` | `<rp>` | |
| `tagRt` | `<rt>` | |
| `tagRuby` | `<ruby>` | |
| `tagS` | `<s>` | устаревший |
| `tagSamp` | `<samp>` | |
| `tagScript` | `<script>` | |
| `tagSection` | `<section>` | |
| `tagSelect` | `<select>` | |
| `tagSmall` | `<small>` | |
| `tagSource` | `<source>` | void |
| `tagSpan` | `<span>` | |
| `tagStrike` | `<strike>` | устаревший |
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
| `tagU` | `<u>` | устаревший |
| `tagUl` | `<ul>` | |
| `tagVar` | `<var>` | |
| `tagVideo` | `<video>` | |
| `tagWbr` | `<wbr>` | void |

---

## Таблица быстрого доступа

| Символ | Вид | Описание |
|---|---|---|
| `HtmlTag` | перечисление | Идентифицирует каждый известный HTML-тег; `tagUnknown` — для нераспознанных |
| `tagToStr` | константный массив | Отображает значения `HtmlTag` на строчные имена тегов |
| `InlineTags` | константное множество | Множество строчных HTML-тегов |
| `BlockTags` | константное множество | Множество блочных HTML-тегов |
| `SingleTags` | константное множество | Множество void/самозакрывающихся тегов (без детей и закрывающего тега) |
| `htmlTag(XmlNode)` | proc | Получить `HtmlTag` разобранного узла (кэшируется в `clientData`) |
| `htmlTag(string)` | proc | Преобразовать строку имени тега в `HtmlTag` (без учёта регистра) |
| `runeToEntity` | proc | Преобразовать `Rune` → числовую HTML-сущность (`#NNN`) |
| `entityToRune` | proc | Преобразовать имя или ссылку HTML-сущности → `Rune` |
| `entityToUtf8` | proc | Преобразовать имя или ссылку HTML-сущности → UTF-8 строку |
| `parseHtml(Stream, string, var seq)` | proc | Разобрать HTML из потока; собирать ошибки в seq |
| `parseHtml(Stream)` | proc | Разобрать HTML из потока; игнорировать ошибки |
| `parseHtml(string)` | proc | Разобрать HTML из строки; игнорировать ошибки |
| `loadHtml(string, var seq)` | proc | Загрузить и разобрать HTML-файл; собирать ошибки |
| `loadHtml(string)` | proc | Загрузить и разобрать HTML-файл; игнорировать ошибки |
