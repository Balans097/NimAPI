# Справочник модуля `std/uri` для Nim

> Модуль реализует разбор URI в соответствии со стандартом **RFC 3986**.
> URI (Uniform Resource Identifier) — унифицированный идентификатор ресурса, однозначно указывающий на объект в интернете.
> URL (Uniform Resource Locator) — подмножество URI, которое дополнительно указывает, *как* получить доступ к ресурсу.
>
> ⚠️ **Важно о безопасности:** Парсеры этого модуля **не выполняют** проверку безопасности. В критичных сценариях необходимо самостоятельно валидировать и санировать входные URI.

---

## Содержание

1. [Типы данных](#типы-данных)
2. [uriParseError](#uriparseerror)
3. [initUri](#inituri)
4. [parseUri (запись в переменную)](#parseuri-запись-в-переменную)
5. [parseUri (возврат значения)](#parseuri-возврат-значения)
6. [encodeUrl](#encodeurl)
7. [decodeUrl](#decodeurl)
8. [encodeQuery](#encodequery)
9. [decodeQuery](#decodequery)
10. [isAbsolute](#isabsolute)
11. [combine (два URI)](#combine-два-uri)
12. [combine (несколько URI)](#combine-несколько-uri)
13. [Оператор /](#оператор-)
14. [Оператор ?](#оператор--1)
15. [Оператор $](#оператор--2)
16. [getDataUri](#getdatauri)

---

## Типы данных

### `Uri`

Центральный тип модуля. Представляет разобранный URI, разбитый на отдельные компоненты согласно структуре, описанной в RFC 3986:

```
scheme://username:password@hostname:port/path?query#anchor
```

```nim
type Uri* = object
  scheme*:   string   # протокол: "https", "ftp", "mailto" и т.д.
  username*: string   # имя пользователя перед символом '@'
  password*: string   # пароль после ':' в части пользователя
  hostname*: string   # доменное имя или IP-адрес
  port*:     string   # номер порта как строка (например, "8080")
  path*:     string   # путь к ресурсу (например, "/docs/index.html")
  query*:    string   # всё после '?' (например, "q=nim&lang=ru")
  anchor*:   string   # фрагмент после '#' (например, "section-3")
  opaque*:   bool     # true для непрозрачных URI типа "mailto:user@example.com"
  isIpv6*:   bool     # true, когда hostname — это IPv6-адрес
```

**Пример — как строка распределяется по полям:**
```nim
import std/uri

let u = parseUri("https://alice:secret@example.com:443/path?q=1#top")
assert u.scheme   == "https"
assert u.username == "alice"
assert u.password == "secret"
assert u.hostname == "example.com"
assert u.port     == "443"
assert u.path     == "/path"
assert u.query    == "q=1"
assert u.anchor   == "top"
```

---

### `Url`

Обёртка над `string` с пометкой `distinct` — отдельный тип для URL-строк. Благодаря этому компилятор не позволяет случайно перепутать обычную строку с URL.

```nim
type Url* = distinct string
```

---

### `UriParseError`

Тип исключения (наследник `ValueError`), возбуждаемого, когда парсер встречает некорректный ввод.

```nim
type UriParseError* = object of ValueError
```

---

## `uriParseError`

```nim
proc uriParseError*(msg: string) {.noreturn.}
```

**Что делает:** Возбуждает исключение `UriParseError` с заданным сообщением. Это вспомогательная процедура, используемая внутри модуля, но доступная и для пользовательского кода. Аннотация `{.noreturn.}` сообщает компилятору, что управление никогда не возвращается из этого вызова — это помогает анализу потока управления.

**Когда использовать:** Когда нужно сигнализировать об ошибке при своей собственной валидации URI, используя тот же тип исключения, что и модуль.

```nim
import std/uri

proc validateScheme(uri: Uri) =
  if uri.scheme notin ["http", "https", "ftp"]:
    uriParseError("Неподдерживаемая схема: " & uri.scheme)

try:
  validateScheme(parseUri("smtp://mail.example.com"))
except UriParseError as e:
  echo e.msg  # "Неподдерживаемая схема: smtp"
```

---

## `initUri`

```nim
func initUri*(isIpv6 = false): Uri
```

**Что делает:** Создаёт пустой объект `Uri`, в котором все строковые поля установлены в `""`, а булевы — в `false`. Параметр `isIpv6` заранее устанавливает флаг IPv6: если он равен `true`, то при преобразовании URI обратно в строку с помощью оператора `$` хост будет автоматически обёрнут в квадратные скобки `[...]`, как того требует RFC 3986.

**Когда использовать:** Когда URI нужно сконструировать программно, задавая поля вручную, а не разбирать из готовой строки.

```nim
import std/uri

# Собираем IPv6-URI вручную
var u = initUri(isIpv6 = true)
u.scheme   = "tcp"
u.hostname = "2001:db8::1"
u.port     = "9000"

echo $u  # tcp://[2001:db8::1]:9000
```

---

## `parseUri` (запись в переменную)

```nim
func parseUri*(uri: string, result: var Uri)
```

**Что делает:** Разбирает строку `uri` и записывает результат в **уже существующую** переменную `Uri`, переданную по ссылке. Перед разбором переменная полностью очищается, поэтому любые ранее сохранённые данные отбрасываются. Эта перегрузка позволяет многократно использовать одну переменную без выделения памяти на каждом вызове — что важно при обработке большого количества URI в цикле.

**Когда использовать:** В узких местах производительности, где нужно минимизировать выделения памяти.

```nim
import std/uri

var res = initUri()

parseUri("https://nim-lang.org/docs/manual.html", res)
assert res.scheme   == "https"
assert res.hostname == "nim-lang.org"
assert res.path     == "/docs/manual.html"

# Повторно используем ту же переменную — она автоматически очищается
parseUri("ftp://files.example.com", res)
assert res.scheme   == "ftp"
assert res.hostname == "files.example.com"
assert res.path     == ""
```

---

## `parseUri` (возврат значения)

```nim
func parseUri*(uri: string): Uri
```

**Что делает:** Разбирает URI-строку `uri` и **возвращает** новый объект `Uri`. Это наиболее удобная форма: она вызывает `initUri()`, а затем — вариант с записью в переменную.

**Когда использовать:** В большинстве повседневных сценариев, когда производительность не является основным приоритетом.

```nim
import std/uri

let u = parseUri("ftp://Alice:s3cr3t@files.example.com/data")
assert u.scheme   == "ftp"
assert u.username == "Alice"
assert u.password == "s3cr3t"
assert u.hostname == "files.example.com"
assert u.path     == "/data"
```

**Относительные URI** (без схемы) тоже обрабатываются корректно:

```nim
let rel = parseUri("/docs/index.html")
assert rel.scheme   == ""
assert rel.hostname == ""
assert rel.path     == "/docs/index.html"
```

---

## `encodeUrl`

```nim
func encodeUrl*(s: string, usePlus = true): string
```

**Что делает:** Кодирует строку для безопасного использования в URL согласно RFC 3986. Символы из «незарезервированного» набора (`a-z`, `A-Z`, `0-9`, `-`, `.`, `_`, `~`) остаются без изменений. Все остальные символы заменяются последовательностью `%XX`, где `XX` — двузначный шестнадцатеричный код символа. Пробелы обрабатываются особо: при `usePlus = true` (по умолчанию) они превращаются в `+`, а при `usePlus = false` — в `%20`.

**Когда использовать:** Перед тем как поместить произвольный пользовательский ввод в строку запроса URL, сегмент пути или любой другой компонент, где специальные символы нарушат структуру URL.

```nim
import std/uri

# Двоеточие и слеши кодируются, чтобы не восприниматься как разделители URL
echo encodeUrl("https://nim-lang.org")
# → "https%3A%2F%2Fnim-lang.org"

# Пробелы → '+' по умолчанию (стиль кодирования форм)
echo encodeUrl("привет мир")
# → "%D0%BF%D1%80%D0%B8%D0%B2%D0%B5%D1%82+%D0%BC%D0%B8%D1%80"

# Пробелы → '%20' при usePlus = false (строгий RFC 3986)
echo encodeUrl("hello world", usePlus = false)
# → "hello%20world"
```

---

## `decodeUrl`

```nim
func decodeUrl*(s: string, decodePlus = true): string
```

**Что делает:** Обратная операция к `encodeUrl`. Заменяет каждую последовательность `%XX` соответствующим символом. При `decodePlus = true` (по умолчанию) символы `+` также преобразуются в пробел. Если `%XX` содержит не шестнадцатеричные символы, последовательность остаётся нетронутой — исключение не возбуждается.

**Когда использовать:** При разборе значений параметров, полученных из URL — например, в обработчике HTTP-запроса.

```nim
import std/uri

echo decodeUrl("hello+world")           # → "hello world"
echo decodeUrl("hello+world", false)    # → "hello+world"  ('+' не трогается)
echo decodeUrl("caf%C3%A9")             # → "café"
echo decodeUrl("abc%xyz")               # → "abc%xyz"  (некорректная последовательность сохраняется)
```

---

## `encodeQuery`

```nim
func encodeQuery*(query: openArray[(string, string)],
                  usePlus = true,
                  omitEq  = true,
                  sep     = '&'): string
```

**Что делает:** Принимает набор пар `(ключ, значение)` и собирает их в строку запроса URL. Каждый ключ и значение URL-кодируются через `encodeUrl`. Пары соединяются символом-разделителем (по умолчанию `&`). Если значение — пустая строка, то при `omitEq = true` (по умолчанию) знак `=` пропускается, и в строке остаётся только ключ.

**Когда использовать:** Когда нужно превратить структурированные данные в строку запроса — например, при формировании поискового запроса или URL для отправки формы.

```nim
import std/uri

echo encodeQuery({"q": "nim программирование", "lang": "ru"})
# → "q=nim+%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5&lang=ru"

# Пустое значение: '=' опускается по умолчанию
echo encodeQuery({"debug": ""})
# → "debug"

# Сохранить '=' даже для пустых значений
echo encodeQuery({"debug": ""}, omitEq = false)
# → "debug="

# Использовать ';' как разделитель (альтернативный стиль)
echo encodeQuery({"a": "1", "b": "2"}, sep = ';')
# → "a=1;b=2"
```

---

## `decodeQuery`

```nim
iterator decodeQuery*(data: string, sep = '&'): tuple[key, value: string]
```

**Что делает:** Итератор, разбирающий строку запроса URL и поочерёдно выдающий пары `(ключ, значение)`. Ключи и значения декодируются из URL-формата на лету. Символ-разделитель (по умолчанию `&`) определяет границы между парами. Корректно обрабатывает крайние случаи: пустые ключи, пустые значения, `=` внутри значений.

**Когда использовать:** При получении строки запроса из URL (например, в обработчике веб-сервера) и необходимости извлечь значения отдельных параметров.

```nim
import std/uri

let qs = "name=Alice&city=New+York&verified"

for key, value in decodeQuery(qs):
  echo key, " → ", value
# name     → Alice
# city     → New York
# verified → ""

# Собрать в таблицу
import std/tables
var params = newTable[string, string]()
for k, v in decodeQuery(qs):
  params[k] = v
```

---

## `isAbsolute`

```nim
func isAbsolute*(uri: Uri): bool
```

**Что делает:** Возвращает `true`, если URI является абсолютным — то есть имеет непустое поле `scheme` **и** непустой `hostname` или `path`. Относительный URI (без схемы, например `/docs/index.html` или `../img/logo.png`) возвращает `false`.

**Когда использовать:** Чтобы проверить, является ли URI самодостаточным или его нужно разрешить относительно базового URI.

```nim
import std/uri

assert parseUri("https://nim-lang.org").isAbsolute       # true
assert parseUri("ftp://files.example.com/x").isAbsolute  # true
assert not parseUri("/relative/path").isAbsolute          # false — нет схемы
assert not parseUri("just-a-name").isAbsolute             # false
```

---

## `combine` (два URI)

```nim
func combine*(base: Uri, reference: Uri): Uri
```

**Что делает:** Объединяет `reference` URI с базовым `base` URI по алгоритму разрешения из [RFC 3986 §5.2.2](https://tools.ietf.org/html/rfc3986#section-5.2.2). Логика в кратком изложении:

- Если у `reference` есть своя схема — он полностью заменяет `base`.
- Если у `reference` есть authority (hostname) — он заменяет authority и path базового URI.
- Если `reference` имеет абсолютный путь (начинается с `/`) — путь `base` заменяется.
- Если `reference` имеет относительный путь — он объединяется с путём `base`, заменяя сегменты после последнего `/`.

Точечные сегменты (`.` и `..`) в результирующем пути удаляются.

**Когда использовать:** При разрешении гиперссылок относительно URL текущего документа — именно так поступает браузер.

```nim
import std/uri

let base = parseUri("https://nim-lang.org/foo/bar")

# Абсолютный путь ссылки заменяет всё после хоста
let a = combine(base, parseUri("/baz"))
assert a.path == "/baz"

# Относительный путь объединяется с базовым (сегмент "bar" заменяется)
let b = combine(base, parseUri("baz"))
assert b.path == "/foo/baz"

# Завершающий слеш в базовом означает «прикрепить к директории»
let c = combine(parseUri("https://nim-lang.org/foo/bar/"), parseUri("baz"))
assert c.path == "/foo/bar/baz"
```

---

## `combine` (несколько URI)

```nim
func combine*(uris: varargs[Uri]): Uri
```

**Что делает:** Удобная перегрузка, применяющая `combine` последовательно к списку URI слева направо. Эквивалентно вложенным вызовам `combine(combine(a, b), c)` и т.д.

**Когда использовать:** При построении URI из нескольких последовательных частей.

```nim
import std/uri

let result = combine(
  parseUri("https://nim-lang.org/"),
  parseUri("docs/"),
  parseUri("manual.html")
)
assert result.hostname == "nim-lang.org"
assert result.path     == "/docs/manual.html"
```

---

## Оператор `/`

```nim
func `/`*(x: Uri, path: string): Uri
```

**Что делает:** Добавляет сегмент пути к пути URI `x`. Оператор намеренно снисходителен к слешам в начале и конце — он нормализует их так, чтобы не возникало двойных слешей (`//`) и не пропускался разделитель между сегментами. В отличие от `combine`, справа передаётся обычная строка, а не `Uri`.

**Когда использовать:** В большинстве случаев построения URL путём добавления сегментов. Предпочтительнее `combine`, когда нужно просто расширить путь.

```nim
import std/uri

let base = parseUri("https://nim-lang.org")

echo $(base / "docs" / "manual.html")
# → "https://nim-lang.org/docs/manual.html"

# Слеши обрабатываются автоматически — двойных не возникнет
echo $(parseUri("https://example.com/a/b") / "c")
# path = "/a/b/c"

echo $(parseUri("https://example.com/a/b/") / "c")
# path = "/a/b/c"
```

---

## Оператор `?`

```nim
func `?`*(u: Uri, query: openArray[(string, string)]): Uri
```

**Что делает:** Прикрепляет строку запроса к URI. Пары `(ключ, значение)` кодируются через `encodeQuery` и записываются в поле `query` возвращаемого URI. Оператор спроектирован для естественного связывания с оператором `/`.

**Когда использовать:** На финальном шаге построения URL запроса, когда нужно добавить параметры.

```nim
import std/uri

let url = parseUri("https://api.example.com") / "search" ? {"q": "Nim", "page": "2"}
echo $url
# → "https://api.example.com/search?q=Nim&page=2"
```

---

## Оператор `$`

```nim
func `$`*(u: Uri): string
```

**Что делает:** Преобразует объект `Uri` обратно в каноническую строку. Все поля собираются в правильном порядке с необходимой пунктуацией (`://`, `@`, `:`, `/`, `?`, `#`). Пустые поля просто пропускаются. IPv6-адреса автоматически оборачиваются в `[...]`.

**Когда использовать:** Всякий раз, когда нужно превратить `Uri` обратно в строку — для вывода, отправки по сети или сохранения.

```nim
import std/uri

let u = parseUri("https://alice@example.com:8080/path?q=1#sec")
assert $u == "https://alice@example.com:8080/path?q=1#sec"

# Работает и для URI, собранных вручную
var v = initUri(isIpv6 = true)
v.scheme   = "https"
v.hostname = "::1"
v.port     = "443"
echo $v   # → "https://[::1]:443"
```

---

## `getDataUri`

```nim
proc getDataUri*(data, mime: string, encoding = "utf-8"): string
```

**Что делает:** Кодирует произвольные бинарные или текстовые данные как [Data URI](https://en.wikipedia.org/wiki/Data_URI_scheme) согласно RFC 2397. Результат — самодостаточный URI вида:

```
data:<mime>;charset=<encoding>;base64,<base64-данные>
```

Данные кодируются в обычный Base64 (не URL-safe вариант — так требует RFC 2397). Data URI часто используются для встраивания изображений, шрифтов и других ресурсов напрямую в HTML или CSS без отдельного HTTP-запроса.

**Когда использовать:** Чтобы встроить небольшие ресурсы (иконки, SVG, шрифты) прямо в HTML или JSON без отдельных файлов.

```nim
import std/uri

# Встраиваем текстовые данные
echo getDataUri("Привет, мир!", "text/plain")
# → "data:text/plain;charset=utf-8;base64,0J/RgNC40LLQtdGCLCDQvNC40YAh"

# SVG-изображение для использования прямо в атрибуте src тега <img>
let svg = "<svg>...</svg>"
let dataUri = getDataUri(svg, "image/svg+xml")
# → "data:image/svg+xml;charset=utf-8;base64,PHN2Zz4uLi48L3N2Zz4="

# Пример из документации модуля
assert getDataUri("Nim", "text/plain") == "data:text/plain;charset=utf-8;base64,Tmlt"
```

---

## Быстрая шпаргалка

| Задача | Функция / оператор |
|---|---|
| Разобрать URI-строку | `parseUri("https://...")` |
| Создать пустой URI | `initUri()` + задать поля |
| Добавить сегмент пути | `uri / "segment"` |
| Добавить параметры запроса | `uri ? {"key": "value"}` |
| Разрешить относительный URI | `combine(base, reference)` |
| URI → строка | `$uri` |
| Закодировать для URL | `encodeUrl(s)` |
| Декодировать из URL | `decodeUrl(s)` |
| Собрать строку запроса | `encodeQuery(pairs)` |
| Разобрать строку запроса | `for k, v in decodeQuery(qs)` |
| Проверить абсолютность | `uri.isAbsolute` |
| Data URI (Base64) | `getDataUri(data, mime)` |
