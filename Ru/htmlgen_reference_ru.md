# Модуль Nim `std/htmlgen` — полный справочник

> Генератор HTML и XML кода на этапе компиляции.  
> Каждый HTML-тег представлен макросом, который производит строку (`string`) во время компиляции — без накладных расходов в рантайме для самой структуры тегов.

**Рекомендуемый стиль импорта:**

```nim
from std/htmlgen import nil
# затем используйте полные имена:
let s = htmlgen.p("Привет, ", htmlgen.b("мир"))
```

Это позволяет избежать конфликтов имён с распространёнными идентификаторами Nim: `div`, `object`, `var`, `template`, `s`, `i` и др.

---

## Содержание

1. [Принцип работы](#принцип-работы)
2. [Экспортируемые константы](#экспортируемые-константы)
3. [Базовая процедура](#базовая-процедура)
4. [Макросы HTML-тегов](#макросы-html-тегов)
   - [Структура документа](#структура-документа)
   - [Метаданные и head](#метаданные-и-head)
   - [Секционирование](#секционирование)
   - [Текстовое содержимое](#текстовое-содержимое)
   - [Встроенная семантика текста](#встроенная-семантика-текста)
   - [Формы и элементы ввода](#формы-и-элементы-ввода)
   - [Таблицы](#таблицы)
   - [Встроенный контент и медиа](#встроенный-контент-и-медиа)
   - [Скриптинг и шаблоны](#скриптинг-и-шаблоны)
   - [Интерактивные элементы](#интерактивные-элементы)
   - [Ruby-аннотации](#ruby-аннотации)
   - [Устаревшие элементы](#устаревшие-элементы)
5. [Макросы MathML](#макросы-mathml)
6. [Правила атрибутов](#правила-атрибутов)
7. [Полная таблица тегов](#полная-таблица-тегов)

---

## Принцип работы

Каждый макрос принимает:
- **Именованные аргументы** (`attr = "value"`) → становятся HTML-атрибутами
- **Позиционные аргументы** (строки, вызовы других макросов) → становятся дочерним содержимым

```nim
# Синтаксис: имяТега(attr1="val", attr2="val", дочерний1, дочерний2, ...)
echo a(href="https://nim-lang.org", "Nim")
# → <a href="https://nim-lang.org">Nim</a>

echo p(class="intro", "Привет ", b("мир"))
# → <p class="intro">Привет <b>мир</b></p>
```

**Листовые элементы** (void-элементы: `<br>`, `<img>`, `<input>` и др.) не могут содержать дочерний контент — только атрибуты — и генерируют самозакрывающийся тег:

```nim
echo br()
# → <br />

echo img(src="photo.jpg", alt="Фото")
# → <img src="photo.jpg" alt="Фото" />
```

Модуль выполняет **проверку атрибутов на этапе компиляции**: передача неизвестного атрибута для тега вызывает ошибку компиляции.  
Атрибуты `data-*` всегда разрешены для любого элемента.

```nim
echo p(`data-value`="42", "текст")
# → <p data-value="42">текст</p>
```

---

## Экспортируемые константы

### `coreAttr`

```nim
const coreAttr*: string
```

Строка с пробельным разделением, содержащая глобальные HTML-атрибуты:  
`accesskey`, `class`, `contenteditable`, `dir`, `hidden`, `id`, `lang`, `spellcheck`, `style`, `tabindex`, `title`, `translate`.

---

### `eventAttr`

```nim
const eventAttr*: string
```

Строка со всеми стандартными атрибутами-обработчиками событий HTML:  
`onabort`, `onblur`, `oncancel`, `oncanplay`, `oncanplaythrough`, `onchange`, `onclick`, `oncuechange`, `ondblclick`, `ondurationchange`, `onemptied`, `onended`, `onerror`, `onfocus`, `oninput`, `oninvalid`, `onkeydown`, `onkeypress`, `onkeyup`, `onload`, `onloadeddata`, `onloadedmetadata`, `onloadstart`, `onmousedown`, `onmouseenter`, `onmouseleave`, `onmousemove`, `onmouseout`, `onmouseover`, `onmouseup`, `onmousewheel`, `onpause`, `onplay`, `onplaying`, `onprogress`, `onratechange`, `onreset`, `onresize`, `onscroll`, `onseeked`, `onseeking`, `onselect`, `onshow`, `onstalled`, `onsubmit`, `onsuspend`, `ontimeupdate`, `ontoggle`, `onvolumechange`, `onwaiting`.

---

### `ariaAttr`

```nim
const ariaAttr*: string
```

Содержит `role` — ARIA-атрибут роли HTML-элемента.

---

### `commonAttr`

```nim
const commonAttr*: string = coreAttr & eventAttr & ariaAttr
```

Объединение всех трёх констант выше. Используется внутри модуля как набор атрибутов по умолчанию для большинства тегов.

---

## Базовая процедура

### `xmlCheckedTag`

```nim
proc xmlCheckedTag*(argsList: NimNode, tag: string,
    optAttr = "", reqAttr = "", isLeaf = false): NimNode
```

Строительный блок, на котором основаны все макросы тегов. Используйте его для определения **собственных XML/HTML-тегов**, которых нет в модуле.

Параметры:
- `argsList` — список аргументов макроса (`NimNode`)
- `tag` — строка с именем тега (например, `"section"`)
- `optAttr` — строка с опциональными атрибутами через пробел
- `reqAttr` — строка с обязательными атрибутами (их отсутствие вызывает ошибку компиляции)
- `isLeaf` — `true` генерирует самозакрывающийся тег (`<tag />`), `false` — `<tag>...</tag>`

```nim
import std/[htmlgen, macros]

macro myCustomTag*(e: varargs[untyped]): untyped =
  result = xmlCheckedTag(e, "x-custom", "color size" & commonAttr)

echo myCustomTag(color="red", "Привет")
# → <x-custom color="red">Привет</x-custom>
```

---

## Макросы HTML-тегов

Все макросы имеют одинаковую сигнатуру:

```nim
macro имяТега*(e: varargs[untyped]): untyped
```

Возвращают `string`.

---

### Структура документа

#### `html`
Генерирует `<html>`. Опциональный атрибут: `xmlns`.
```nim
echo html(xmlns="http://www.w3.org/1999/xhtml", head(""), body("содержимое"))
```

#### `head`
Генерирует `<head>`. Содержит элементы метаданных.
```nim
echo head(title("Моя страница"), meta(charset="utf-8"))
```

#### `body`
Генерирует `<body>`. Дополнительные атрибуты: `onafterprint`, `onbeforeprint`, `onbeforeunload`, `onhashchange`, `onmessage`, `onoffline`, `ononline`, `onpagehide`, `onpageshow`, `onpopstate`, `onstorage`, `onunload`.
```nim
echo body(onload="init()", p("Привет"))
```

---

### Метаданные и head

#### `title`
Генерирует `<title>`.
```nim
echo title("Моя страница")  # → <title>Моя страница</title>
```

#### `base`
Листовой элемент. Опциональные атрибуты: `href`, `target`.
```nim
echo base(href="https://example.com/", target="_blank")
# → <base href="https://example.com/" target="_blank" />
```

#### `link`
Листовой элемент. Опциональные атрибуты: `href`, `crossorigin`, `rel`, `media`, `hreflang`, `type`, `sizes`.
```nim
echo link(rel="stylesheet", href="style.css")
# → <link rel="stylesheet" href="style.css" />
```

#### `meta`
Листовой элемент. Опциональные атрибуты: `name`, `http-equiv`, `content`, `charset`.
```nim
echo meta(charset="utf-8")
# → <meta charset="utf-8" />
echo meta(name="description", content="Мой сайт")
# → <meta name="description" content="Мой сайт" />
```

#### `style`
Генерирует `<style>`. Опциональные атрибуты: `media`, `type`.
```nim
echo style(type="text/css", "body { margin: 0 }")
```

---

### Секционирование

#### `article`
Генерирует `<article>`.

#### `aside`
Генерирует `<aside>`.

#### `footer`
Генерирует `<footer>`.

#### `header`
Генерирует `<header>`.

#### `h1` .. `h6`
Заголовки уровней 1–6.
```nim
echo h1("Главный заголовок")    # → <h1>Главный заголовок</h1>
echo h2("Подзаголовок")         # → <h2>Подзаголовок</h2>
```

#### `main`
Генерирует `<main>`.

#### `nav`
Генерирует `<nav>`.

#### `section`
Генерирует `<section>`.

---

### Текстовое содержимое

#### `blockquote`
Генерирует `<blockquote>`. Опциональный атрибут: `cite`.
```nim
echo blockquote(cite="https://example.com", "Мудрая цитата.")
```

#### `dd`
Генерирует `<dd>` (определение в списке описаний).

#### `div`
Генерирует `<div>`. Обратите внимание: в Nim `div` — ключевое слово деления, поэтому необходимо заключать в обратные кавычки или использовать квалифицированное имя.
```nim
echo `div`(class="container", p("текст"))
```

#### `dl`
Генерирует `<dl>` (список описаний).

#### `dt`
Генерирует `<dt>` (термин в списке описаний).

#### `figcaption`
Генерирует `<figcaption>`.

#### `figure`
Генерирует `<figure>`.
```nim
echo figure(img(src="cat.jpg", alt="Кот"), figcaption("Кот на солнце"))
```

#### `hr`
Листовой элемент, аргументов нет.
```nim
echo hr()  # → <hr />
```

#### `li`
Генерирует `<li>`. Опциональный атрибут: `value`.

#### `ol`
Генерирует `<ol>`. Опциональные атрибуты: `reversed`, `start`, `type`.
```nim
echo ol(start="3", li("Третий"), li("Четвёртый"))
```

#### `p`
Генерирует `<p>`.
```nim
echo p(class="note", "Это абзац.")
```

#### `pre`
Генерирует `<pre>`.

#### `ul`
Генерирует `<ul>`.
```nim
echo ul(li("Яблоко"), li("Банан"), li("Вишня"))
# → <ul><li>Яблоко</li><li>Банан</li><li>Вишня</li></ul>
```

---

### Встроенная семантика текста

#### `a`
Генерирует `<a>`. Опциональные атрибуты: `href`, `target`, `download`, `rel`, `hreflang`, `type`.
```nim
echo a(href="https://nim-lang.org", target="_blank", "Язык Nim")
# → <a href="https://nim-lang.org" target="_blank">Язык Nim</a>
```

#### `abbr`
Генерирует `<abbr>` (аббревиатура).
```nim
echo abbr(title="HyperText Markup Language", "HTML")
```

#### `address`
Генерирует `<address>`.

#### `b`
Генерирует `<b>` (выделение полужирным).

#### `bdi`
Генерирует `<bdi>` (изоляция двунаправленного текста).

#### `bdo`
Генерирует `<bdo>` (переопределение направления текста).

#### `br`
Листовой элемент, содержимого нет.
```nim
echo br()  # → <br />
```

#### `cite`
Генерирует `<cite>`.

#### `code`
Генерирует `<code>`.
```nim
echo code("""echo "Hello"""")
```

#### `data`
Генерирует `<data>`. Опциональный атрибут: `value`.

#### `dfn`
Генерирует `<dfn>` (определяемый термин).

#### `em`
Генерирует `<em>` (выделение с акцентом).

#### `i`
Генерирует `<i>`. При необходимости используйте `` `i` ``.

#### `kbd`
Генерирует `<kbd>` (ввод с клавиатуры).
```nim
echo kbd("Ctrl+C")
```

#### `mark`
Генерирует `<mark>` (выделение маркером).

#### `q`
Генерирует `<q>` (встроенная цитата). Опциональный атрибут: `cite`.

#### `s`
Генерирует `<s>` (зачёркнутый текст). Используйте `` `s` ``.

#### `samp`
Генерирует `<samp>` (пример вывода программы).

#### `small`
Генерирует `<small>`.

#### `span`
Генерирует `<span>`.
```nim
echo span(class="highlight", "важный текст")
```

#### `strong`
Генерирует `<strong>`.

#### `sub`
Генерирует `<sub>` (подстрочный индекс).

#### `sup`
Генерирует `<sup>` (надстрочный индекс).
```nim
echo p("H", sub("2"), "O и x", sup("2"))
```

#### `time`
Генерирует `<time>`. Опциональный атрибут: `datetime`.
```nim
echo time(datetime="2024-01-01", "Новый год")
```

#### `u`
Генерирует `<u>` (подчёркнутый текст).

#### `var`
Генерирует `<var>`. Используйте `` `var` ``.

#### `wbr`
Листовой элемент. Возможность переноса слова.
```nim
echo wbr()  # → <wbr />
```

---

### Формы и элементы ввода

#### `form`
Генерирует `<form>`. Опциональные атрибуты: `accept-charset`, `action`, `autocomplete`, `enctype`, `method`, `name`, `novalidate`, `target`.
```nim
echo form(action="/submit", `method`="post",
  input(`type`="text", name="username"),
  button(`type`="submit", "Отправить")
)
```

#### `button`
Генерирует `<button>`. Опциональные атрибуты: `autofocus`, `disabled`, `form`, `formaction`, `formenctype`, `formmethod`, `formnovalidate`, `formtarget`, `menu`, `name`, `type`, `value`.

#### `datalist`
Генерирует `<datalist>`.

#### `fieldset`
Генерирует `<fieldset>`. Опциональные атрибуты: `disabled`, `form`, `name`.

#### `input`
Листовой элемент. Опциональные атрибуты: `accept`, `alt`, `autocomplete`, `autofocus`, `checked`, `dirname`, `disabled`, `form`, `formaction`, `formenctype`, `formmethod`, `formnovalidate`, `formtarget`, `height`, `inputmode`, `list`, `max`, `maxlength`, `min`, `minlength`, `multiple`, `name`, `pattern`, `placeholder`, `readonly`, `required`, `size`, `src`, `step`, `type`, `value`, `width`.
```nim
echo input(`type`="email", name="email", placeholder="вы@example.com", required="required")
# → <input type="email" name="email" placeholder="вы@example.com" required="required" />
```

#### `keygen`
Генерирует `<keygen>`. Опциональные атрибуты: `autofocus`, `challenge`, `disabled`, `form`, `keytype`, `name`.

#### `label`
Генерирует `<label>`. Опциональные атрибуты: `form`, `for`.
```nim
echo label(`for`="email", "Адрес email:")
```

#### `legend`
Генерирует `<legend>`.

#### `meter`
Генерирует `<meter>`. Опциональные атрибуты: `value`, `min`, `max`, `low`, `high`, `optimum`.
```nim
echo meter(value="70", min="0", max="100", "70%")
```

#### `optgroup`
Генерирует `<optgroup>`. Обязательный атрибут: `label`. Опциональный: `disabled`.

#### `option`
Генерирует `<option>`. Опциональные атрибуты: `disabled`, `label`, `selected`, `value`.

#### `output`
Генерирует `<output>`. Опциональные атрибуты: `for`, `form`, `name`.

#### `progress`
Генерирует `<progress>`. Опциональные атрибуты: `value`, `max`.
```nim
echo progress(value="40", max="100")
```

#### `select`
Генерирует `<select>`. Опциональные атрибуты: `autofocus`, `disabled`, `form`, `multiple`, `name`, `required`, `size`.
```nim
echo select(name="color",
  option(value="red", "Красный"),
  option(value="blue", "Синий")
)
```

#### `textarea`
Генерирует `<textarea>`. Опциональные атрибуты: `autocomplete`, `autofocus`, `cols`, `dirname`, `disabled`, `form`, `inputmode`, `maxlength`, `minlength`, `name`, `placeholder`, `readonly`, `required`, `rows`, `wrap`.

---

### Таблицы

#### `caption`
Генерирует `<caption>`.

#### `col`
Листовой элемент. Опциональный атрибут: `span`.

#### `colgroup`
Генерирует `<colgroup>`. Опциональный атрибут: `span`.

#### `table`
Генерирует `<table>`. Опциональные атрибуты: `border`, `sortable`.
```nim
echo table(
  thead(tr(th("Имя"), th("Возраст"))),
  tbody(
    tr(td("Алиса"), td("30")),
    tr(td("Боб"),   td("25"))
  )
)
```

#### `tbody`
Генерирует `<tbody>`.

#### `td`
Генерирует `<td>`. Опциональные атрибуты: `colspan`, `rowspan`, `headers`.

#### `tfoot`
Генерирует `<tfoot>`.

#### `th`
Генерирует `<th>`. Опциональные атрибуты: `colspan`, `rowspan`, `headers`, `abbr`, `scope`, `axis`, `sorted`.

#### `thead`
Генерирует `<thead>`.

#### `tr`
Генерирует `<tr>`.

---

### Встроенный контент и медиа

#### `area`
Листовой элемент. Обязательный атрибут: `alt`. Опциональные: `coords`, `download`, `href`, `hreflang`, `rel`, `shape`, `target`, `type`.

#### `audio`
Генерирует `<audio>`. Опциональные атрибуты: `src`, `crossorigin`, `preload`, `autoplay`, `mediagroup`, `loop`, `muted`, `controls`.
```nim
echo audio(src="sound.mp3", controls="controls")
```

#### `canvas`
Генерирует `<canvas>`. Опциональные атрибуты: `width`, `height`.

#### `embed`
Листовой элемент. Опциональные атрибуты: `src`, `type`, `height`, `width`.

#### `iframe`
Генерирует `<iframe>`. Опциональные атрибуты: `src`, `srcdoc`, `name`, `sandbox`, `width`, `height`, `loading`.

#### `img`
Листовой элемент. Обязательные атрибуты: `src`, `alt`. Опциональные: `crossorigin`, `usemap`, `ismap`, `height`, `width`, `loading`.
```nim
echo img(src="logo.png", alt="Логотип", width="200", height="80")
# → <img src="logo.png" alt="Логотип" width="200" height="80" />
```

#### `map`
Генерирует `<map>`. Опциональный атрибут: `name`.

#### `object`
Генерирует `<object>`. Используйте `` `object` ``. Опциональные атрибуты: `data`, `type`, `typemustmatch`, `name`, `usemap`, `form`, `width`, `height`.

#### `param`
Листовой элемент. Обязательные атрибуты: `name`, `value`.

#### `picture`
Генерирует `<picture>`.

#### `portal`
Генерирует `<portal>`. Опциональные атрибуты: `width`, `height`, `type`, `src`, `disabled`.

#### `source`
Листовой элемент. Обязательный атрибут: `src`. Опциональный: `type`.
```nim
echo video(
  source(src="video.webm", `type`="video/webm"),
  source(src="video.mp4",  `type`="video/mp4"),
  "Ваш браузер не поддерживает видео."
)
```

#### `track`
Листовой элемент. Обязательный атрибут: `src`. Опциональные: `kind`, `srclang`, `label`, `default`.

#### `video`
Генерирует `<video>`. Опциональные атрибуты: `src`, `crossorigin`, `poster`, `preload`, `autoplay`, `mediagroup`, `loop`, `muted`, `controls`, `width`, `height`.

---

### Скриптинг и шаблоны

#### `noscript`
Генерирует `<noscript>`.

#### `script`
Генерирует `<script>`. Опциональные атрибуты: `src`, `type`, `charset`, `async`, `defer`, `crossorigin`.
```nim
echo script(src="app.js", defer="defer")
```

#### `slot`
Генерирует `<slot>`.

#### `template`
Генерирует `<template>`. Используйте `` `template` ``.

---

### Интерактивные элементы

#### `details`
Генерирует `<details>`. Опциональный атрибут: `open`.
```nim
echo details(open="open",
  summary("Нажмите, чтобы раскрыть"),
  p("Скрытое содержимое.")
)
```

#### `dialog`
Генерирует `<dialog>`. Опциональный атрибут: `open`.

#### `menu`
Генерирует `<menu>`.

#### `summary`
Генерирует `<summary>`.

---

### Ruby-аннотации

#### `rb`
Генерирует `<rb>` (база ruby).

#### `rp`
Генерирует `<rp>` (запасные скобки ruby).

#### `rt`
Генерирует `<rt>` (текст ruby).

#### `rtc`
Генерирует `<rtc>` (контейнер текста ruby).

#### `ruby`
Генерирует `<ruby>`.
```nim
echo ruby("漢", rp("("), rt("Kan"), rp(")"))
```

---

### Устаревшие элементы

Эти теги присутствуют в модуле для полноты, однако в HTML5 они считаются устаревшими. Не используйте их в новых проектах.

#### `big`
Генерирует `<big>`.

#### `center`
Генерирует `<center>`.

#### `marquee`
Генерирует `<marquee>`. Атрибуты: `behavior`, `bgcolor`, `direction`, `height`, `hspace`, `loop`, `scrollamount`, `scrolldelay`, `truespeed`, `vspace`, `width`, `onbounce`, `onfinish`, `onstart`.

#### `tt`
Генерирует `<tt>` (телетайпный текст).

---

## Макросы MathML

MathML является частью HTML5 (ISO/IEC 40314:2015). Все макросы MathML принимают `mathbackground`, `mathcolor`, `href` и `commonAttr` в качестве общих атрибутов.

```nim
echo math(
  semantics(
    mrow(
      msup(mi("x"), mn("42"))
    )
  )
)
# → <math><semantics><mrow><msup><mi>x</mi><mn>42</mn></msup></mrow></semantics></math>
```

| Макрос | HTML-элемент | Дополнительные атрибуты |
|---|---|---|
| `math` | `<math>` | `overflow` |
| `maction` | `<maction>` | `actiontype selection` |
| `menclose` | `<menclose>` | `notation` |
| `merror` | `<merror>` | — |
| `mfenced` | `<mfenced>` | `open close separators` |
| `mfrac` | `<mfrac>` | `linethickness numalign` |
| `mglyph` | `<mglyph>` | `src valign` |
| `mi` | `<mi>` | `mathsize mathvariant` |
| `mlabeledtr` | `<mlabeledtr>` | `columnalign groupalign rowalign` |
| `mmultiscripts` | `<mmultiscripts>` | `subscriptshift superscriptshift` |
| `mn` | `<mn>` | `mathsize mathvariant` |
| `mo` | `<mo>` | `fence form largeop lspace mathsize mathvariant movablelimits rspace separator stretchy symmetric` |
| `mover` | `<mover>` | `accent` |
| `mpadded` | `<mpadded>` | `depth lspace voffset` |
| `mphantom` | `<mphantom>` | — |
| `mroot` | `<mroot>` | — |
| `mrow` | `<mrow>` | — |
| `ms` | `<ms>` | `lquote mathsize mathvariant rquote` |
| `mspace` | `<mspace>` | `linebreak` |
| `msqrt` | `<msqrt>` | — |
| `mstyle` | `<mstyle>` | `decimalpoint displaystyle infixlinebreakstyle scriptlevel scriptminsize scriptsizemultiplier` |
| `msub` | `<msub>` | `subscriptshift` |
| `msubsup` | `<msubsup>` | `subscriptshift superscriptshift` |
| `msup` | `<msup>` | `superscriptshift` |
| `mtable` | `<mtable>` | `align alignmentscope columnalign columnlines columnspacing columnwidth displaystyle equalcolumns equalrows frame framespacing groupalign rowalign rowlines rowspacing side width` |
| `mtd` | `<mtd>` | `columnalign columnspan groupalign rowalign rowspan` |
| `mtext` | `<mtext>` | `mathsize mathvariant` |
| `munder` | `<munder>` | `accentunder align` |
| `munderover` | `<munderover>` | `accentunder accent align` |
| `semantics` | `<semantics>` | `definitionURL encoding cd src` |
| `annotation` | `<annotation>` | `definitionURL encoding cd src` |
| `` `annotation-xml` `` | `<annotation-xml>` | `definitionURL encoding cd src` |

---

## Правила атрибутов

| Правило | Подробности |
|---|---|
| Неизвестный атрибут | Ошибка компиляции |
| Атрибуты `data-*` | Разрешены на любом элементе |
| Обязательные атрибуты | Отсутствие → ошибка компиляции |
| Теги-ключевые слова Nim | Заключайте в обратные кавычки: `` `div` ``, `` `var` ``, `` `object` ``, `` `template` `` |
| Атрибуты с дефисом | Заключайте в обратные кавычки: `` `accept-charset` ``, `` `http-equiv` ``, `` `annotation-xml` `` |

---

## Полная таблица тегов

| Макрос | Тег | Листовой? | Обязательные атрибуты | Ключевые опциональные атрибуты |
|---|---|:---:|---|---|
| `a` | `<a>` | | | `href target download rel hreflang type` |
| `abbr` | `<abbr>` | | | |
| `address` | `<address>` | | | |
| `area` | `<area>` | ✓ | `alt` | `coords href shape target` |
| `article` | `<article>` | | | |
| `aside` | `<aside>` | | | |
| `audio` | `<audio>` | | | `src controls autoplay loop muted` |
| `b` | `<b>` | | | |
| `base` | `<base>` | ✓ | | `href target` |
| `bdi` | `<bdi>` | | | |
| `bdo` | `<bdo>` | | | |
| `big` | `<big>` | | | *(устаревший)* |
| `blockquote` | `<blockquote>` | | | `cite` |
| `body` | `<body>` | | | события жизненного цикла страницы |
| `br` | `<br>` | ✓ | | |
| `button` | `<button>` | | | `type name value disabled` |
| `canvas` | `<canvas>` | | | `width height` |
| `caption` | `<caption>` | | | |
| `center` | `<center>` | | | *(устаревший)* |
| `cite` | `<cite>` | | | |
| `code` | `<code>` | | | |
| `col` | `<col>` | ✓ | | `span` |
| `colgroup` | `<colgroup>` | | | `span` |
| `data` | `<data>` | | | `value` |
| `datalist` | `<datalist>` | | | |
| `dd` | `<dd>` | | | |
| `del` | `<del>` | | | `cite datetime` |
| `details` | `<details>` | | | `open` |
| `dfn` | `<dfn>` | | | |
| `dialog` | `<dialog>` | | | `open` |
| `` `div` `` | `<div>` | | | |
| `dl` | `<dl>` | | | |
| `dt` | `<dt>` | | | |
| `em` | `<em>` | | | |
| `embed` | `<embed>` | ✓ | | `src type width height` |
| `fieldset` | `<fieldset>` | | | `disabled form name` |
| `figcaption` | `<figcaption>` | | | |
| `figure` | `<figure>` | | | |
| `footer` | `<footer>` | | | |
| `form` | `<form>` | | | `action method enctype name` |
| `h1`–`h6` | `<h1>`–`<h6>` | | | |
| `head` | `<head>` | | | |
| `header` | `<header>` | | | |
| `hr` | `<hr>` | ✓ | | |
| `html` | `<html>` | | | `xmlns` |
| `i` | `<i>` | | | |
| `iframe` | `<iframe>` | | | `src srcdoc name sandbox width height loading` |
| `img` | `<img>` | ✓ | `src alt` | `width height loading crossorigin` |
| `input` | `<input>` | ✓ | | `type name value placeholder required` |
| `ins` | `<ins>` | | | `cite datetime` |
| `kbd` | `<kbd>` | | | |
| `keygen` | `<keygen>` | | | `name keytype disabled` |
| `label` | `<label>` | | | `for form` |
| `legend` | `<legend>` | | | |
| `li` | `<li>` | | | `value` |
| `link` | `<link>` | ✓ | | `rel href type media` |
| `main` | `<main>` | | | |
| `map` | `<map>` | | | `name` |
| `mark` | `<mark>` | | | |
| `marquee` | `<marquee>` | | | *(устаревший)* |
| `menu` | `<menu>` | | | |
| `meta` | `<meta>` | ✓ | | `charset name content http-equiv` |
| `meter` | `<meter>` | | | `value min max low high optimum` |
| `nav` | `<nav>` | | | |
| `noscript` | `<noscript>` | | | |
| `` `object` `` | `<object>` | | | `data type name width height` |
| `ol` | `<ol>` | | | `reversed start type` |
| `optgroup` | `<optgroup>` | | `label` | `disabled` |
| `option` | `<option>` | | | `value selected disabled label` |
| `output` | `<output>` | | | `for form name` |
| `p` | `<p>` | | | |
| `param` | `<param>` | ✓ | `name value` | |
| `picture` | `<picture>` | | | |
| `portal` | `<portal>` | | | `src type width height disabled` |
| `pre` | `<pre>` | | | |
| `progress` | `<progress>` | | | `value max` |
| `q` | `<q>` | | | `cite` |
| `rb` | `<rb>` | | | |
| `rp` | `<rp>` | | | |
| `rt` | `<rt>` | | | |
| `rtc` | `<rtc>` | | | |
| `ruby` | `<ruby>` | | | |
| `s` | `<s>` | | | |
| `samp` | `<samp>` | | | |
| `script` | `<script>` | | | `src type async defer crossorigin` |
| `section` | `<section>` | | | |
| `select` | `<select>` | | | `name multiple size disabled` |
| `slot` | `<slot>` | | | |
| `small` | `<small>` | | | |
| `source` | `<source>` | ✓ | `src` | `type` |
| `span` | `<span>` | | | |
| `strong` | `<strong>` | | | |
| `style` | `<style>` | | | `media type` |
| `sub` | `<sub>` | | | |
| `summary` | `<summary>` | | | |
| `sup` | `<sup>` | | | |
| `table` | `<table>` | | | `border sortable` |
| `tbody` | `<tbody>` | | | |
| `td` | `<td>` | | | `colspan rowspan headers` |
| `` `template` `` | `<template>` | | | |
| `textarea` | `<textarea>` | | | `name rows cols placeholder required` |
| `tfoot` | `<tfoot>` | | | |
| `th` | `<th>` | | | `colspan rowspan scope abbr sorted` |
| `thead` | `<thead>` | | | |
| `time` | `<time>` | | | `datetime` |
| `title` | `<title>` | | | |
| `tr` | `<tr>` | | | |
| `track` | `<track>` | ✓ | `src` | `kind srclang label default` |
| `tt` | `<tt>` | | | *(устаревший)* |
| `u` | `<u>` | | | |
| `ul` | `<ul>` | | | |
| `` `var` `` | `<var>` | | | |
| `video` | `<video>` | | | `src controls width height poster` |
| `wbr` | `<wbr>` | ✓ | | |
