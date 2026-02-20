# Справочник модуля `std/mimetypes` (Nim)

> Модуль реализует базу данных MIME-типов: преобразование расширений файлов в MIME-типы и обратно, а также регистрацию пользовательских типов.

---

## Содержание

1. [Типы данных](#типы-данных)
2. [Константа `mimes`](#константа-mimes)
3. [Функции](#функции)
   - [newMimetypes](#newmimetypes)
   - [getMimetype](#getmimetype)
   - [getExt](#getext)
   - [register](#register)
4. [Поведение и тонкости](#поведение-и-тонкости)
5. [Категории встроенных типов](#категории-встроенных-типов)
6. [Практические примеры](#практические-примеры)

---

## Типы данных

### `MimeDB`

Объект, хранящий таблицу соответствий «расширение → MIME-тип».

```nim
type
  MimeDB* = object
    mimes: OrderedTable[string, string]
```

Поле `mimes` является приватным — работа с базой ведётся исключительно через экспортированные функции. Порядок записей соответствует порядку их добавления (используется `OrderedTable`), что важно для функции `getExt` — она возвращает **первое** найденное расширение для данного MIME-типа.

---

## Константа `mimes`

```nim
const mimes* = { "ez": "application/andrew-inset", ... }
```

Встроенная константа-таблица, содержащая более **700 пар** «расширение → MIME-тип». Экспортируется публично и может использоваться напрямую, если нужен доступ к исходным данным без создания `MimeDB`.

Охватываемые категории: `application/`, `audio/`, `chemical/`, `font/`, `image/`, `message/`, `model/`, `text/`, `video/`, `x-conference/`.

Среди включённых типов — поддержка Nim-файлов:
```nim
"nim"   → "text/nim"
"nimble" → "text/nimble"
"nimf"  → "text/nim"
"nims"  → "text/nim"
```

```nim
import std/mimetypes

# Использование константы напрямую
for ext, mime in mimes:
  if mime == "image/png":
    echo ext   # png
```

---

## Функции

### `newMimetypes`

```nim
func newMimetypes*(): MimeDB
```

Создаёт новую базу данных MIME-типов, заранее заполненную всеми встроенными записями из константы `mimes`.

**Возвращает:** инициализированный объект `MimeDB`.

Функция является `func` (без побочных эффектов в смысле Nim), хотя внутренне использует `{.cast(noSideEffect).}` для преобразования таблицы при инициализации.

```nim
import std/mimetypes

var db = newMimetypes()

# База сразу готова к работе
echo db.getMimetype("html")  # text/html
echo db.getMimetype("mp3")   # audio/mpeg
echo db.getMimetype("png")   # image/png
```

---

### `getMimetype`

```nim
func getMimetype*(mimedb: MimeDB, ext: string, default = "text/plain"): string
```

Возвращает MIME-тип по расширению файла.

**Параметры:**
- `mimedb` — база данных MIME-типов
- `ext` — расширение файла (с ведущей точкой или без неё)
- `default` — значение, возвращаемое при отсутствии расширения в базе (по умолчанию `"text/plain"`)

**Поведение:**
- Расширение приводится к **нижнему регистру** перед поиском (регистронезависимо).
- Ведущая точка в расширении **игнорируется автоматически** (`".html"` и `"html"` дают одинаковый результат).
- При пустой строке или неизвестном расширении возвращается `default`.

```nim
import std/mimetypes

var db = newMimetypes()

# Базовые примеры
echo db.getMimetype("html")   # text/html
echo db.getMimetype("mp4")    # video/mp4
echo db.getMimetype("png")    # image/png
echo db.getMimetype("json")   # application/json
echo db.getMimetype("pdf")    # application/pdf

# Регистр не важен
echo db.getMimetype("MP4")    # video/mp4
echo db.getMimetype("HTML")   # text/html
echo db.getMimetype("PNG")    # image/png

# Ведущая точка игнорируется
echo db.getMimetype(".html")  # text/html
echo db.getMimetype(".MP4")   # video/mp4

# Неизвестное расширение → default
echo db.getMimetype("xyz123")        # text/plain  (значение по умолчанию)
echo db.getMimetype("INVALID")       # text/plain
echo db.getMimetype("")              # text/plain

# Кастомное значение по умолчанию
echo db.getMimetype("xyz", "application/octet-stream")
# application/octet-stream

# Nim-файлы
echo db.getMimetype("nim")    # text/nim
echo db.getMimetype("nimble") # text/nimble

# Извлечение из полного пути с помощью splitFile
import std/os
let (_, _, ext) = splitFile("/data/image.JPEG")
echo db.getMimetype(ext)      # image/jpeg
```

---

### `getExt`

```nim
func getExt*(mimedb: MimeDB, mimetype: string, default = "txt"): string
```

Возвращает расширение файла по MIME-типу.

**Параметры:**
- `mimedb` — база данных MIME-типов
- `mimetype` — MIME-тип для поиска
- `default` — значение, возвращаемое при отсутствии типа в базе (по умолчанию `"txt"`)

**Поведение:**
- MIME-тип приводится к **нижнему регистру** перед поиском.
- Возвращает расширение **без ведущей точки**.
- Поскольку один MIME-тип может соответствовать нескольким расширениям, возвращается **первое** найденное в порядке вставки в `OrderedTable`.
- При пустой строке или неизвестном типе возвращается `default`.

```nim
import std/mimetypes

var db = newMimetypes()

# Базовые примеры
echo db.getExt("text/html")          # html
echo db.getExt("video/mp4")          # mp4
echo db.getExt("image/png")          # png
echo db.getExt("application/json")   # json
echo db.getExt("audio/mpeg")         # mpga  (первое в базе для этого типа)

# Регистр не важен
echo db.getExt("TEXT/HTML")          # html
echo db.getExt("VIDEO/MP4")          # mp4

# Неизвестный тип → default
echo db.getExt("INVALID/NONEXISTENT")  # txt
echo db.getExt("")                     # txt

# Кастомное значение по умолчанию
echo db.getExt("application/x-unknown", "bin")
# bin

# Важно: один тип — несколько расширений,
# возвращается первое зарегистрированное
echo db.getExt("image/jpeg")  # jpeg (а не jpg — jpeg идёт первым в базе)
echo db.getExt("audio/ogg")   # oga  (а не ogg или spx)
```

---

### `register`

```nim
func register*(mimedb: var MimeDB, ext: string, mimetype: string)
```

Регистрирует новое соответствие «расширение → MIME-тип» в базе данных. Позволяет добавлять собственные или переопределять существующие записи.

**Параметры:**
- `mimedb` — база данных (изменяемая, `var`)
- `ext` — расширение файла (без точки)
- `mimetype` — MIME-тип

**Поведение:**
- Оба аргумента приводятся к **нижнему регистру** перед сохранением.
- Если расширение уже существует — его значение **перезаписывается**.
- Пустая строка (после `strip`) в `ext` или `mimetype` вызывает `AssertionDefect`.
- Не принимает ведущую точку в `ext` — она будет частью ключа и поиск по такому расширению перестанет работать корректно.

```nim
import std/mimetypes

var db = newMimetypes()

# Добавление нового типа
db.register(ext = "fakext", mimetype = "text/fakelang")
echo db.getMimetype("fakext")   # text/fakelang
echo db.getMimetype("FAKEXT")   # text/fakelang  (регистронезависимо)
echo db.getExt("text/fakelang") # fakext

# Переопределение существующего типа
db.register(ext = "json", mimetype = "application/json5")
echo db.getMimetype("json")     # application/json5

# Регистр входных данных не важен — всё сохраняется в нижнем регистре
db.register(ext = "MYEXT", mimetype = "APPLICATION/MY-TYPE")
echo db.getMimetype("myext")    # application/my-type
echo db.getMimetype("MYEXT")    # application/my-type

# Ошибка при пустом аргументе (AssertionDefect)
# db.register(ext = "", mimetype = "text/plain")       # ошибка!
# db.register(ext = "txt", mimetype = "")              # ошибка!
# db.register(ext = "  ", mimetype = "text/plain")     # ошибка! (пробелы)

# Добавление типов для проекта
db.register("toml",  "application/toml")
db.register("graphql", "application/graphql")
db.register("wasm",  "application/wasm")
db.register("avif",  "image/avif")
db.register("heic",  "image/heic")
```

---

## Поведение и тонкости

### Регистронезависимость

Все функции приводят расширения и MIME-типы к нижнему регистру перед поиском или сохранением. Это делает API полностью регистронезависимым.

```nim
var db = newMimetypes()
# Все следующие вызовы возвращают одно и то же:
doAssert db.getMimetype("mp4")  == "video/mp4"
doAssert db.getMimetype("MP4")  == "video/mp4"
doAssert db.getMimetype("Mp4")  == "video/mp4"
doAssert db.getMimetype(".MP4") == "video/mp4"
```

### Ведущая точка

`getMimetype` обрабатывает расширения как с точкой, так и без неё. `register` принимает расширение **без точки** — точка станет частью ключа и нарушит поиск.

```nim
var db = newMimetypes()

# Правильно:
echo db.getMimetype(".html")  # text/html  ← точка обрезается автоматически
db.register("newext", "text/new")  # без точки

# Неправильно (точка сохранится как часть ключа):
# db.register(".newext", "text/new")
# db.getMimetype("newext")  →  "text/plain"  (не найдено!)
```

### Несколько расширений для одного MIME-типа

База содержит множество MIME-типов, которым соответствует несколько расширений. `getExt` возвращает только первое найденное.

```nim
var db = newMimetypes()

# image/jpeg соответствует: jpeg, jpg, jpe
echo db.getExt("image/jpeg")  # jpeg

# audio/mpeg соответствует: mpga, mp2, mp2a, mp3, m2a, m3a
echo db.getExt("audio/mpeg")  # mpga

# text/plain соответствует: txt, text, conf, def, list, log, in
echo db.getExt("text/plain")  # txt
```

### `getExt` как линейный поиск

Функция `getExt` выполняет **линейный обход** всей таблицы, так как поиск идёт по значению, а не по ключу. При базе в 700+ записей это практически не заметно, но для горячих путей в высоконагруженном коде рекомендуется кешировать результаты.

---

## Категории встроенных типов

Модуль содержит расширения для следующих основных категорий:

| Категория       | Примеры расширений                                      |
|-----------------|---------------------------------------------------------|
| `application/`  | `pdf`, `zip`, `json`, `xml`, `doc`, `xls`, `ppt`, `jar`, `apk`, `epub` |
| `audio/`        | `mp3`, `ogg`, `wav`, `flac`, `aac`, `m4a`, `opus`, `weba` |
| `chemical/`     | `cif`, `cml`, `xyz`                                     |
| `font/`         | `ttf`, `otf`, `woff`, `woff2`, `ttc`                   |
| `image/`        | `png`, `jpeg`, `gif`, `webp`, `svg`, `bmp`, `ico`, `tiff`, `psd` |
| `message/`      | `eml`, `mime`                                           |
| `model/`        | `dae`, `wrl`, `x3d`, `obj`, `stl`                      |
| `text/`         | `html`, `css`, `js`, `csv`, `txt`, `xml`, `c`, `java`, `nim` |
| `video/`        | `mp4`, `avi`, `mkv`, `webm`, `mov`, `ogv`, `flv`, `wmv` |
| `x-conference/` | `ice`                                                   |

---

## Практические примеры

### HTTP-сервер: установка Content-Type по расширению файла

```nim
import std/[mimetypes, os, net]

var db = newMimetypes()

proc getContentType(filename: string): string =
  let (_, _, ext) = splitFile(filename)
  # ext уже содержит точку, getMimetype её обрежет
  db.getMimetype(ext, default = "application/octet-stream")

echo getContentType("index.html")    # text/html
echo getContentType("style.CSS")     # text/css
echo getContentType("app.bundle.js") # text/javascript
echo getContentType("data.UNKNOWN")  # application/octet-stream
echo getContentType("archive.tar.gz")# application/octet-stream (gz→ не найдено)
```

### Валидация загружаемых файлов по MIME-типу

```nim
import std/mimetypes

var db = newMimetypes()

const allowedImages = ["image/jpeg", "image/png", "image/gif", "image/webp"]

proc isAllowedImage(ext: string): bool =
  db.getMimetype(ext) in allowedImages

echo isAllowedImage("jpg")  # true
echo isAllowedImage("png")  # true
echo isAllowedImage("exe")  # false
echo isAllowedImage("SVG")  # false (svg = image/svg+xml, не в списке)
```

### Инициализация с дополнительными типами для современных форматов

```nim
import std/mimetypes

proc newAppMimetypes(): MimeDB =
  result = newMimetypes()
  # Форматы, которых нет в стандартной базе:
  result.register("avif",    "image/avif")
  result.register("heic",    "image/heic")
  result.register("heif",    "image/heif")
  result.register("jxl",     "image/jxl")
  result.register("wasm",    "application/wasm")
  result.register("toml",    "application/toml")
  result.register("graphql", "application/graphql+json")
  result.register("ndjson",  "application/x-ndjson")
  result.register("jsonld",  "application/ld+json")

var db = newAppMimetypes()
echo db.getMimetype("avif")    # image/avif
echo db.getMimetype("wasm")    # application/wasm
echo db.getMimetype("graphql") # application/graphql+json
```

### Определение типа медиафайла

```nim
import std/mimetypes

var db = newMimetypes()

proc mediaKind(filename: string): string =
  let mime = db.getMimetype(filename[filename.rfind('.')+1..^1])
  if mime.startsWith("image/"): "image"
  elif mime.startsWith("video/"): "video"
  elif mime.startsWith("audio/"): "audio"
  elif mime.startsWith("text/"): "text"
  else: "binary"

echo mediaKind("photo.jpg")    # image
echo mediaKind("clip.mp4")     # video
echo mediaKind("song.flac")    # audio
echo mediaKind("readme.txt")   # text
echo mediaKind("archive.zip")  # binary
```

### Использование константы `mimes` напрямую

```nim
import std/mimetypes

# Подсчёт всех расширений по категориям
var counts: array[0..9, int]
let cats = ["application", "audio", "chemical", "font",
            "image", "message", "model", "text", "video", "x-conference"]

for ext, mime in mimes:
  for i, cat in cats:
    if mime.startsWith(cat & "/"):
      inc counts[i]
      break

for i, cat in cats:
  echo cat, ": ", counts[i]
```

### Маппинг через `getExt` для генерации ссылок

```nim
import std/mimetypes

var db = newMimetypes()

# Получить расширение файла из заголовка Content-Type ответа
proc extFromContentType(contentType: string): string =
  # Убрать параметры вроде "; charset=utf-8"
  var mime = contentType
  let semi = mime.find(';')
  if semi >= 0:
    mime = mime[0..<semi].strip()
  db.getExt(mime, "bin")

echo extFromContentType("image/png")                # png
echo extFromContentType("text/html; charset=utf-8") # html
echo extFromContentType("application/json")         # json
echo extFromContentType("application/unknown")      # bin
```
