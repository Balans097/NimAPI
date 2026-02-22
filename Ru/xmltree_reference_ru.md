# xmltree — справочник по модулю

> Модуль стандартной библиотеки Nim для построения и обработки XML-деревьев в памяти.
> Позволяет **создавать**, **просматривать**, **обходить**, **изменять** и **сериализовать** XML-документы без обращения к файлам или парсерам — всё хранится в виде дерева объектов `XmlNode`.

---

## Содержание

1. [Основные типы](#основные-типы)
2. [Константы](#константы)
3. [Конструкторы узлов](#конструкторы-узлов)
4. [Доступ к тексту и тегу](#доступ-к-тексту-и-тегу)
5. [Внутренний текст](#внутренний-текст)
6. [Управление деревом](#управление-деревом)
7. [Индексация и итерация](#индексация-и-итерация)
8. [Атрибуты](#атрибуты)
9. [Поиск](#поиск)
10. [Сериализация](#сериализация)
11. [Экранирование](#экранирование)
12. [Низкоуровневые оптимизации](#низкоуровневые-оптимизации)
13. [Клиентские данные](#клиентские-данные)
14. [Макрос `<>`](#макрос-)

---

## Основные типы

### `XmlNode`

```nim
type XmlNode* = ref XmlNodeObj
```

Фундаментальный строительный блок модуля. Каждый элемент XML-документа — тег, текстовый фрагмент, комментарий, CDATA-секция или ссылка на сущность — представлен как `XmlNode`. Поскольку это ссылочный тип (`ref`), узлы передаются по ссылке: две переменные могут указывать на одно и то же поддерево.

---

### `XmlNodeKind`

```nim
type XmlNodeKind* = enum
  xnText,         # обычный текст
  xnVerbatimText, # текст без экранирования
  xnElement,      # тег с атрибутами и дочерними узлами
  xnCData,        # секция <![CDATA[…]]>
  xnEntity,       # ссылка &name;
  xnComment       # комментарий <!-- … -->
```

Каждый `XmlNode` обладает полем вида (`kind`), соответствующим одному из значений этого перечисления. Знание вида — обязательное условие корректного использования большинства процедур: например, `tag` работает только с `xnElement`, а попытка вызвать её на узле другого вида вызовет ошибку утверждения в рантайме.

---

### `XmlAttributes`

```nim
type XmlAttributes* = StringTableRef
```

Регистрочувствительная таблица «строка → строка», хранящая атрибуты элементного узла. Таблица инициализируется **лениво**: новые элементы создаются с `nil`-атрибутами, чтобы не тратить память зря. Значение отличное от `nil` появляется только в момент реального присвоения атрибутов.

---

## Константы

### `xmlHeader`

```nim
const xmlHeader* = "<?xml version=\"1.0\" encoding=\"UTF-8\" ?>\n"
```

Стандартная декларация XML-документа. Если нужно сформировать полноценный, самостоятельный XML-файл (а не фрагмент), эту строку следует добавить в начало сериализованного дерева.

```nim
let doc = newElement("root")
doc.add newText("привет")
echo xmlHeader & $doc
# <?xml version="1.0" encoding="UTF-8" ?>
# <root>привет</root>
```

---

## Конструкторы узлов

Эти процедуры создают узлы конкретного вида.

---

### `newElement`

```nim
proc newElement*(tag: sink string): XmlNode
```

Создаёт **элементный узел** — основную рабочую единицу XML. Элемент имеет имя тега, необязательные атрибуты и список дочерних узлов. После создания список детей пустой, атрибуты равны `nil`.

```nim
let p = newElement("p")
p.add newText("Привет, мир!")
echo $p
# <p>Привет, мир!</p>
```

---

### `newText`

```nim
proc newText*(text: sink string): XmlNode
```

Создаёт **текстовый узел**. При сериализации спецсимволы (`<`, `>`, `&`, `"`, `'`) автоматически экранируются, гарантируя корректность выходного XML.

```nim
let t = newText("Цена < 10 & скидка > 0")
echo $t
# Цена &lt; 10 &amp; скидка &gt; 0
```

---

### `newVerbatimText`

```nim
proc newVerbatimText*(text: sink string): XmlNode
```

Аналог `newText`, но содержимое вставляется **без какого-либо экранирования**. Применяется, когда строка уже содержит корректный XML/HTML или специальные последовательности, которые не нужно трогать. Если в тексте есть неэкранированные `<` или `&`, результирующий документ будет некорректным — используйте осознанно.

```nim
let raw = newVerbatimText("&уже;&#x2013;экранировано")
echo $raw
# &уже;&#x2013;экранировано   (двойного экранирования не происходит)
```

---

### `newComment`

```nim
proc newComment*(comment: sink string): XmlNode
```

Создаёт **узел-комментарий**. В сериализованном виде принимает форму `<!-- текст -->`. Комментарии видны в исходном XML, но игнорируются XML-процессорами.

```nim
let c = newComment("TODO: убрать этот блок")
echo $c
# <!-- TODO: убрать этот блок -->
```

---

### `newCData`

```nim
proc newCData*(cdata: sink string): XmlNode
```

Создаёт **CDATA-секцию**. Содержимое оборачивается в `<![CDATA[` … `]]>` и не экранируется сериализатором. Традиционно используется для встраивания блоков текста с большим количеством спецсимволов — например, JavaScript или CSS внутри XML-документа.

```nim
let js = newCData("if (a < b && c > 0) { return true; }")
echo $js
# <![CDATA[if (a < b && c > 0) { return true; }]]>
```

---

### `newEntity`

```nim
proc newEntity*(entity: string): XmlNode
```

Создаёт **узел-ссылку на сущность**. Имя заключается в `&` … `;`. Встроенные сущности (`&amp;`, `&lt;` и т. д.) автоматически обрабатываются в `newText`; `newEntity` предназначен для именованных сущностей, определённых в DTD, или кастомных ссылок.

```nim
let e = newEntity("copy")
echo $e
# &copy;
```

---

### `newXmlTree`

```nim
proc newXmlTree*(tag: sink string,
                 children: openArray[XmlNode],
                 attributes: XmlAttributes = nil): XmlNode
```

Удобный конструктор, создающий элементный узел и сразу заполняющий его детьми и атрибутами за один вызов. Эквивалентен последовательному вызову `newElement`, нескольких `add` и присвоения `attrs`. Идеален для компактного построения деревьев в одном выражении.

```nim
let attrs = {"lang": "ru", "dir": "ltr"}.toXmlAttributes
let html  = newXmlTree("html", [
  newXmlTree("head", [newElement("title")]),
  newElement("body")
], attrs)
echo $html
# <html dir="ltr" lang="ru">
#   <head>
#     <title />
#   </head>
#   <body />
# </html>
```

> **Замечание:** Порядок атрибутов в выводе не гарантирован, поскольку `XmlAttributes` — это хеш-таблица.

---

## Доступ к тексту и тегу

### `text` (геттер)

```nim
proc text*(n: XmlNode): lent string
```

Возвращает текстовое содержимое узла. Работает для `xnText`, `xnVerbatimText`, `xnComment`, `xnCData` и `xnEntity`. Вызов на `xnElement` вызовет ошибку утверждения.

```nim
let c = newComment("черновик")
echo c.text   # черновик
```

---

### `text=` (сеттер)

```nim
proc `text=`*(n: XmlNode, text: sink string)
```

Заменяет текстовое содержимое листового узла. Допустим для тех же видов, что и геттер.

```nim
var e = newEntity("nbsp")
e.text = "copy"
echo $e   # &copy;
```

---

### `tag` (геттер)

```nim
proc tag*(n: XmlNode): lent string
```

Возвращает имя тега элементного узла. Узел обязан быть `xnElement`.

```nim
let el = newElement("section")
echo el.tag   # section
```

---

### `tag=` (сеттер)

```nim
proc `tag=`*(n: XmlNode, tag: sink string)
```

Изменяет имя тега существующего элементного узла, не затрагивая детей и атрибуты.

```nim
var el = newElement("div")
el.tag = "section"
echo el.tag   # section
```

---

### `rawText`

```nim
proc rawText*(n: XmlNode): string
```

Возвращает внутреннюю строку текста путём перемещения или поверхностного копирования (в зависимости от режима GC). Это **оптимизация для узкоспециализированных сериализаторов**; при использовании деструкторов вызов опустошает поле узла. Для обычной работы используйте `text`.

---

### `rawTag`

```nim
proc rawTag*(n: XmlNode): string
```

То же, что `rawText`, но для поля тега. Аналогичные предостережения.

---

## Внутренний текст

### `innerText`

```nim
proc innerText*(n: XmlNode): string
```

Рекурсивно обходит поддерево `n` и конкатенирует содержимое всех текстовых узлов (`xnText`) и ссылок на сущности (`xnEntity`), игнорируя теги элементов, комментарии и CDATA. Именно это видел бы пользователь в браузере как «видимый текст» HTML-элемента.

```nim
var article = newElement("article")
article.add newElement("h1")        # текста нет, пропускается
article.add newText("Привет ")
article.add newEntity("mdash")      # имя сущности включается в результат
article.add newText(" мир")

echo innerText(article)   # Привет mdash мир
```

---

## Управление деревом

### `add` (один ребёнок)

```nim
proc add*(father, son: XmlNode)
```

Добавляет один дочерний узел в конец списка детей `father`. `father` должен быть `xnElement`.

```nim
let ul = newElement("ul")
ul.add newXmlTree("li", [newText("Пункт 1")])
ul.add newXmlTree("li", [newText("Пункт 2")])
echo $ul
# <ul>
#   <li>Пункт 1</li>
#   <li>Пункт 2</li>
# </ul>
```

---

### `add` (несколько детей)

```nim
proc add*(father: XmlNode, sons: openArray[XmlNode])
```

Добавляет массив или последовательность дочерних узлов за один раз. Функционально равнозначен циклическому вызову однодетного `add`.

```nim
let row = newElement("tr")
row.add @[newXmlTree("td", [newText("A")]),
          newXmlTree("td", [newText("B")])]
```

---

### `insert` (один ребёнок)

```nim
proc insert*(father, son: XmlNode, index: int)
```

Вставляет одного ребёнка на указанную позицию. Все существующие дети, начиная с `index`, сдвигаются вправо. Если `index` выходит за пределы текущей длины, узел добавляется в конец.

```nim
var list = newElement("ol")
list.add newXmlTree("li", [newText("второй")])
list.insert(newXmlTree("li", [newText("первый")]), 0)
echo $list
# <ol>
#   <li>первый</li>
#   <li>второй</li>
# </ol>
```

---

### `insert` (несколько детей)

```nim
proc insert*(father: XmlNode, sons: openArray[XmlNode], index: int)
```

Вставляет массив дочерних узлов в указанную позицию, сохраняя их взаимный порядок.

---

### `delete` (по индексу)

```nim
proc delete*(n: XmlNode, i: Natural)
```

Удаляет ребёнка на позиции `i`. Последующие дети смещаются влево. При выходе `i` за пределы вызывается ошибка.

```nim
var nav = newElement("nav")
nav.add newXmlTree("a", [newText("Главная")])
nav.add newXmlTree("a", [newText("О нас")])
nav.delete(0)   # удаляем «Главная»
echo $nav
# <nav>
#   <a>О нас</a>
# </nav>
```

---

### `delete` (по срезу)

```nim
proc delete*(n: XmlNode, slice: Slice[int])
```

Удаляет непрерывный диапазон дочерних узлов. Например, `n.delete(1..3)` удаляет детей с индексами 1, 2 и 3.

```nim
var row = newElement("tr")
for i in 1..5:
  row.add newXmlTree("td", [newText($i)])
row.delete(1..3)   # оставляем только td[0] и td[4]
```

---

### `replace` (по индексу)

```nim
proc replace*(n: XmlNode, i: Natural, replacement: openArray[XmlNode])
```

Заменяет ребёнка на позиции `i` на набор узлов из `replacement`. Количество заменяющих узлов может не совпадать с единицей: один узел можно развернуть в несколько.

```nim
var root = newElement("root")
root.add newElement("placeholder")
root.replace(0, @[newElement("real"), newElement("content")])
echo $root
# <root>
#   <real />
#   <content />
# </root>
```

---

### `replace` (по срезу)

```nim
proc replace*(n: XmlNode, slice: Slice[int], replacement: openArray[XmlNode])
```

Заменяет диапазон дочерних узлов на новый набор. Размеры диапазона и набора замены не обязаны совпадать.

---

### `clear`

```nim
proc clear*(n: var XmlNode)
```

Рекурсивно удаляет **всех** детей `n` и каждого из потомков-элементов. Сам узел и его атрибуты сохраняются — стирается только содержимое. Полезно для повторного использования скелета дерева с новыми данными.

```nim
var form = newXmlTree("form",
  [newElement("input"), newElement("button")],
  {"action": "/submit"}.toXmlAttributes)
echo form.len   # 2
clear(form)
echo form.len   # 0
echo $form      # <form action="/submit" />
```

---

## Индексация и итерация

### `len`

```nim
proc len*(n: XmlNode): int
```

Возвращает количество прямых детей. Для нелементных узлов всегда равно 0.

```nim
let parent = newXmlTree("ul", [newElement("li"), newElement("li")])
echo parent.len   # 2
```

---

### `kind`

```nim
proc kind*(n: XmlNode): XmlNodeKind
```

Возвращает вид узла. Используйте для ветвления перед вызовом видоспецифичных процедур.

```nim
proc describe(n: XmlNode) =
  case n.kind
  of xnElement: echo "элемент: ", n.tag
  of xnText:    echo "текст: ",   n.text
  of xnComment: echo "коммент: ", n.text
  else:         echo "другое"
```

---

### `[]` (чтение)

```nim
proc `[]`*(n: XmlNode, i: int): XmlNode
```

Возвращает `i`-й дочерний узел (нумерация с нуля). Вызов на нелементном узле вызывает ошибку утверждения.

```nim
let parent = newXmlTree("ul", [newElement("li"), newElement("li")])
echo parent[0].tag   # li
```

---

### `[]` (запись)

```nim
proc `[]`*(n: var XmlNode, i: int): var XmlNode
```

Возвращает изменяемую ссылку на `i`-й дочерний узел — для модификации на месте.

---

### `items` (итератор)

```nim
iterator items*(n: XmlNode): XmlNode
```

Перебирает прямых детей элементного узла по порядку. Именно этот итератор задействуется при `for child in parent:`.

```nim
let menu = newXmlTree("menu",
  [newXmlTree("item", [newText("Файл")]),
   newXmlTree("item", [newText("Правка")])])

for item in menu:
  echo item.innerText
# Файл
# Правка
```

---

### `mitems` (изменяемый итератор)

```nim
iterator mitems*(n: var XmlNode): var XmlNode
```

Как `items`, но возвращает изменяемые ссылки, позволяя модифицировать каждого ребёнка без явной индексации.

```nim
var list = newXmlTree("ul",
  [newXmlTree("li", [newText("старый")])])

for child in mitems(list):
  if child.kind == xnElement:
    child[0].text = "новый"

echo $list   # <ul><li>новый</li></ul>
```

---

## Атрибуты

### `toXmlAttributes`

```nim
proc toXmlAttributes*(keyValuePairs: varargs[tuple[key, val: string]]): XmlAttributes
```

Преобразует набор пар `(ключ, значение)` в таблицу `XmlAttributes`. Идиоматический способ вызова — через синтаксис литерала `{}`.

```nim
let attrs = {"class": "hero", "id": "banner"}.toXmlAttributes
let div = newElement("div")
div.attrs = attrs
echo $div
# <div class="hero" id="banner" />
```

---

### `attrs` (геттер)

```nim
proc attrs*(n: XmlNode): XmlAttributes
```

Возвращает таблицу атрибутов элементного узла, или `nil`, если атрибуты ещё не задавались. Перед итерацией или обращением по ключу всегда проверяйте на `nil`.

```nim
let el = newElement("span")
echo el.attrs == nil   # true — атрибутов нет
```

---

### `attrs=` (сеттер)

```nim
proc `attrs=`*(n: XmlNode, attr: XmlAttributes)
```

Заменяет всю таблицу атрибутов элемента. Передача `nil` очищает атрибуты.

```nim
var el = newElement("input")
el.attrs = {"type": "text", "name": "q"}.toXmlAttributes
echo $el
# <input name="q" type="text" />
```

---

### `attrsLen`

```nim
proc attrsLen*(n: XmlNode): int
```

Возвращает количество атрибутов. Возвращает 0 как в случае `nil`-атрибутов, так и при наличии пустой таблицы.

```nim
let el = newElement("br")
echo el.attrsLen   # 0
el.attrs = {"class": "wide"}.toXmlAttributes
echo el.attrsLen   # 1
```

---

### `attr`

```nim
proc attr*(n: XmlNode, name: string): string
```

Ищет значение одного атрибута по имени. Возвращает пустую строку, если атрибут отсутствует или `attrs` равен `nil`. Безопасно вызывать без предварительной проверки на `nil`.

```nim
let img = newElement("img")
img.attrs = {"src": "фото.png", "alt": "Фотография"}.toXmlAttributes
echo img.attr("src")       # фото.png
echo img.attr("missing")   # (пустая строка)
```

---

## Поиск

### `child`

```nim
proc child*(n: XmlNode, name: string): XmlNode
```

Ищет первого **прямого** ребёнка `n`, чей тег совпадает с `name`. Возвращает `nil`, если такого нет. Поиск регистрозависимый и не рекурсивный — смотрит только один уровень вглубь.

```nim
let root = newXmlTree("root", [
  newXmlTree("meta", []),
  newXmlTree("body", [newText("содержимое")])
])
let body = root.child("body")
echo innerText(body)   # содержимое
```

---

### `findAll` (с накоплением)

```nim
proc findAll*(n: XmlNode, tag: string,
              result: var seq[XmlNode],
              caseInsensitive = false)
```

Рекурсивно обходит всё поддерево `n` и **дописывает** в существующую последовательность `result` все элементные узлы с именем тега `tag`. Так как результаты дописываются, а не перезаписываются, можно последовательно вызывать процедуру для нескольких поддеревьев, аккумулируя результаты в одной переменной.

```nim
var matches: seq[XmlNode]
root.findAll("td", matches)
echo matches.len   # все <td> в любой точке дерева
```

---

### `findAll` (с возвратом seq)

```nim
proc findAll*(n: XmlNode, tag: string,
              caseInsensitive = false): seq[XmlNode]
```

Сокращённая форма, создающая и возвращающая новую последовательность. Удобна, когда не нужно переиспользовать накопитель.

```nim
let allLinks = page.findAll("a")
for link in allLinks:
  echo link.attr("href")
```

Флаг `caseInsensitive` делает сравнение тегов нечувствительным к регистру — полезно при работе с HTML, где регистр тегов может различаться.

```nim
let allDivs = doc.findAll("div", caseInsensitive = true)
# находит <div>, <Div>, <DIV> и т. д.
```

---

## Сериализация

### `add` (строковый перегруз)

```nim
proc add*(result: var string, n: XmlNode,
          indent = 0, indWidth = 2, addNewLines = true)
```

Дописывает сериализованное XML-представление `n` в строку `result`. Это низкоуровневый блок, на котором строится `$`. Параметры форматирования: `indent` — начальный отступ (в пробелах), `indWidth` — дополнительные пробелы на каждый уровень вложенности, `addNewLines` — переносить ли элементы на отдельные строки.

```nim
var output = "<!-- сгенерировано -->\n"
output.add(myTree, indent = 0, indWidth = 4)
```

---

### `$` (преобразование в строку)

```nim
proc `$`*(n: XmlNode): string
```

Преобразует узел (или всё дерево) в строковое XML-представление с форматированием по умолчанию (отступ 2 пробела, переносы строк включены). Декларация `<?xml … ?>` не добавляется — используйте `xmlHeader` отдельно, если нужен полноценный документ.

```nim
let tree = newXmlTree("note",
  [newXmlTree("to",   [newText("Алиса")]),
   newXmlTree("from", [newText("Боб")]),
   newXmlTree("body", [newText("До встречи!")])])
echo $tree
# <note>
#   <to>Алиса</to>
#   <from>Боб</from>
#   <body>До встречи!</body>
# </note>
```

---

## Экранирование

### `escape`

```nim
proc escape*(s: string): string
```

Возвращает новую строку, в которой XML-спецсимволы заменены соответствующими сущностями:

| Символ | Заменяется на |
|--------|---------------|
| `<`    | `&lt;`        |
| `>`    | `&gt;`        |
| `&`    | `&amp;`       |
| `"`    | `&quot;`      |
| `'`    | `&apos;`      |

Используйте, когда нужно вставить произвольную строку в атрибут или в текст вне `newText`-узла.

```nim
let safe = escape("<script>alert('xss')</script>")
echo safe
# &lt;script&gt;alert(&apos;xss&apos;)&lt;/script&gt;
```

---

### `addEscaped`

```nim
proc addEscaped*(result: var string, s: string)
```

In-place, экономичная по памяти версия `escape`. Вместо создания новой строки дописывает экранированное содержимое в существующую переменную `result`. Предпочтительна в горячих циклах или при формировании больших документов.

```nim
var buf = "<description>"
buf.addEscaped("Цена < 100 & скидка > 0")
buf.add("</description>")
echo buf
# <description>Цена &lt; 100 &amp; скидка &gt; 0</description>
```

---

## Низкоуровневые оптимизации

### `rawText` и `rawTag`

Эти процедуры предназначены исключительно для высокопроизводительных сериализаторов и генераторов. Они возвращают внутреннее строковое поле путём перемещения (`move`) или поверхностного копирования (`shallowCopy`) без выделения памяти. При использовании GC с деструкторами вызов `rawText` или `rawTag` **опустошает** соответствующее поле узла, фактически разрушая его. Не используйте в прикладном коде.

---

## Клиентские данные

### `clientData` (геттер)

```nim
proc clientData*(n: XmlNode): int
```

Возвращает целочисленное поле, полностью предоставленное разработчику для собственных нужд. Внутри стандартной библиотеки это поле использует HTML-парсер и генератор для хранения состояния. Вы можете задействовать его в собственных инструментах поверх `xmltree`.

---

### `clientData=` (сеттер)

```nim
proc `clientData=`*(n: XmlNode, data: int)
```

Устанавливает значение клиентского поля.

```nim
let node = newElement("section")
node.clientData = 42
echo node.clientData   # 42
```

---

## Макрос `<>`

```nim
macro `<>`*(x: untyped): untyped
```

Компилируемый макрос, предоставляющий лаконичный, HTML-подобный синтаксис для построения XML-деревьев. Атрибуты записываются как выражения `ключ = значение`, дочерние узлы передаются позиционно. Макрос разворачивается на этапе компиляции в эквивалентный вызов `newXmlTree` / `newStringTable` — накладных расходов в рантайме нет.

**Синтаксис:**

```nim
<>имяТега(атрибут1 = "значение1", атрибут2 = "значение2", узел1, узел2)
```

**Примеры:**

```nim
# Простой элемент с атрибутом
let link = <>a(href = "https://nim-lang.org", newText("Nim"))
echo $link
# <a href="https://nim-lang.org">Nim</a>

# Вложенная структура
let card = <>div(class = "card",
  <>h2(newText("Заголовок")),
  <>p(newText("Текст.")))
echo $card
# <div class="card">
#   <h2>Заголовок</h2>
#   <p>Текст.</p>
# </div>

# Самозакрывающийся элемент (без детей)
let br = <>br()
echo $br
# <br />
```

> **Совет:** Имена атрибутов с дефисом (например, `data-lang`) поддерживаются — макрос убирает пробелы, которые иначе появились бы в результате токенизации Nim.

---

## Сводная таблица

| Процедура / Макрос | Что делает |
|---|---|
| `newElement(tag)` | Создать элементный узел |
| `newText(s)` | Создать экранированный текстовый узел |
| `newVerbatimText(s)` | Создать неэкранированный текстовый узел |
| `newComment(s)` | Создать узел-комментарий |
| `newCData(s)` | Создать CDATA-секцию |
| `newEntity(s)` | Создать ссылку на сущность |
| `newXmlTree(tag, дети, атрибуты)` | Создать элемент с детьми и атрибутами |
| `n.text` / `n.text=` | Получить/установить текст листового узла |
| `n.tag` / `n.tag=` | Получить/установить имя тега |
| `innerText(n)` | Собрать весь текст рекурсивно |
| `n.add(сын)` | Добавить одного ребёнка в конец |
| `n.add(сыновья)` | Добавить нескольких детей в конец |
| `n.insert(сын, i)` | Вставить одного ребёнка на позицию |
| `n.insert(сыновья, i)` | Вставить нескольких детей на позицию |
| `n.delete(i)` | Удалить ребёнка по индексу |
| `n.delete(срез)` | Удалить диапазон детей |
| `n.replace(i, узлы)` | Заменить ребёнка по индексу |
| `n.replace(срез, узлы)` | Заменить диапазон детей |
| `clear(n)` | Рекурсивно удалить всех детей |
| `n.len` | Количество прямых детей |
| `n.kind` | Вид узла (`XmlNodeKind`) |
| `n[i]` | Получить i-го ребёнка |
| `for c in n` | Итерация по детям |
| `for c in mitems(n)` | Изменяемая итерация |
| `toXmlAttributes({…})` | Создать таблицу атрибутов |
| `n.attrs` / `n.attrs=` | Получить/установить таблицу атрибутов |
| `n.attrsLen` | Количество атрибутов |
| `n.attr(name)` | Получить значение одного атрибута |
| `n.child(name)` | Найти первого прямого ребёнка по тегу |
| `n.findAll(tag)` | Найти всех потомков по тегу |
| `$n` | Сериализовать в строку |
| `s.add(n)` | Дописать сериализованный узел в строку |
| `escape(s)` | Экранировать XML-спецсимволы |
| `addEscaped(result, s)` | Эффективное экранирование in-place |
| `xmlHeader` | Константа: стандартная декларация XML |
| `<>тег(атрибуты, дети)` | Макрос: лаконичное построение дерева |
