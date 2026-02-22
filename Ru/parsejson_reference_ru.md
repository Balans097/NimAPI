# Справочник по модулю `parsejson`

> **Модуль:** `parsejson`  
> **Назначение:** Низкоуровневый потоковый событийный JSON-парсер. Используется внутри модуля `json`, но пригоден и самостоятельно — для экономной по памяти обработки больших JSON-документов без построения дерева разбора.

---

## Обзор

`parsejson` реализует **pull-парсер**: вы многократно вызываете `next`, продвигая парсер на одно событие за раз, и проверяете результат через `kind`, `str`, `getInt` и `getFloat`. Это та же модель, что SAX в XML или `encoding/json`'s `Decoder.Token` в Go — вы потребляете ровно столько документа, сколько нужно, и никогда не платите за построение полного дерева в памяти.

Парсер основан на `lexbase.BaseLexer`, обеспечивающем буферизованный построчный ввод, корректную обработку CR/LF/CRLF и отслеживание позиции. Дополнительно парсер допускает `//` и `/* */` комментарии во входных данных — технически нестандартный JSON, но полезный при разборе конфигурационных файлов.

### Типичный сценарий использования

```
open → цикл { next → проверить kind → прочитать значение } → close
```

---

## Типы

### `JsonEventKind`

```nim
type JsonEventKind* = enum
  jsonError,        ## произошла ошибка разбора
  jsonEof,          ## ввод исчерпан
  jsonString,       ## разобрано строковое значение
  jsonInt,          ## разобран целочисленный литерал
  jsonFloat,        ## разобран литерал с плавающей точкой
  jsonTrue,         ## литерал `true`
  jsonFalse,        ## литерал `false`
  jsonNull,         ## литерал `null`
  jsonObjectStart,  ## токен `{`
  jsonObjectEnd,    ## токен `}`
  jsonArrayStart,   ## токен `[`
  jsonArrayEnd      ## токен `]`
```

Каждый вызов `next` переводит парсер в одно из этих состояний. Чтение `kind()` сообщает текущее состояние. События напрямую отражают структурные токены JSON: `jsonObjectStart` приходит при встрече `{`, затем идёт последовательность событий ключей и значений, и наконец `jsonObjectEnd` при встрече `}`.

---

### `TokKind`

```nim
type TokKind* = enum
  tkError, tkEof, tkString, tkInt, tkFloat,
  tkTrue, tkFalse, tkNull,
  tkCurlyLe, tkCurlyRi,     ## { }
  tkBracketLe, tkBracketRi, ## [ ]
  tkColon, tkComma
```

Сырой лексический токен, возвращаемый `getTok`. `JsonEventKind` — это высокоуровневое семантическое событие, получаемое из токена после обработки конечным автоматом в `next`. Большинству пользователей нужен только `JsonEventKind`; `TokKind` актуален при прямых вызовах `getTok` или `eat`.

---

### `JsonError`

```nim
type JsonError* = enum
  errNone,              ## нет ошибки
  errInvalidToken,      ## неверный символ или последовательность
  errStringExpected,    ## ожидалась строка
  errColonExpected,     ## ожидался символ ':'
  errCommaExpected,     ## ожидался символ ','
  errBracketRiExpected, ## ожидался символ ']'
  errCurlyRiExpected,   ## ожидался символ '}'
  errQuoteExpected,     ## ожидался '"' или '\'' (незакрытая строка)
  errEOC_Expected,      ## ожидался '*/' (незакрытый блочный комментарий)
  errEofExpected,       ## ожидался конец файла, но найдены ещё токены
  errExprExpected       ## ожидалось выражение
```

Когда `kind` равен `jsonError`, внутреннее поле `err` хранит одно из этих значений. Вызовите `errorMsg`, чтобы получить полное человекочитаемое описание с именем файла, строкой и колонкой.

---

### `JsonParser`

```nim
type JsonParser* = object of BaseLexer
  a*: string          ## буфер накопления: хранит текст текущего токена
  tok*: TokKind       ## последний сырой лексический токен
  kind: JsonEventKind ## текущее семантическое событие (читается через kind())
  err: JsonError      ## текущая ошибка (читается через errorMsg())
  state: seq[ParserState]
  filename: string
  rawStringLiterals: bool
```

Объект парсера. **Должен объявляться как `var`** — все изменяющие процедуры принимают его по `var`. Поля `a` и `tok` публичны: `a` содержит сырой текст текущего строкового, целого или вещественного токена; `tok` — последний вид сырого токена.

Наследование от `BaseLexer` даёт парсеру автоматический буферизованный ввод-вывод, корректную нормализацию переносов строк и механизм `lineNumber`/`bufpos`, используемый `getLine` и `getColumn`.

---

### `JsonKindError`

```nim
type JsonKindError* = object of ValueError
```

Вызывается макросом `to` из модуля `json`, когда вид JSON не совпадает с ожидаемым типом Nim. Сам `parsejson` его не вызывает — он объявлен здесь как часть общего публичного API.

---

### `JsonParsingError`

```nim
type JsonParsingError* = object of ValueError
```

Вызывается `raiseParseErr`. Несёт форматированное сообщение об ошибке, созданное `errorMsgExpected`, с именем файла, строкой и колонкой.

---

### `errorMessages`

```nim
const errorMessages*: array[JsonError, string]
```

Массив времени компиляции, отображающий каждый вариант `JsonError` на его описание на английском. Индексируется значением `JsonError`. Полезен, если нужно напечатать ошибку без вызова `errorMsg` — например, когда значение `JsonError` уже есть, но объекта парсера нет.

```nim
echo errorMessages[errColonExpected]   # → "':' expected"
```

---

## Жизненный цикл

### `open`

```nim
proc open*(my: var JsonParser, input: Stream, filename: string;
           rawStringLiterals = false)
```

Инициализирует парсер переданным `Stream` и строкой имени файла. Имя файла используется только в сообщениях об ошибках — это может быть любой описательный ярлык, необязательно реальный путь.

**`rawStringLiterals`** управляет тем, как строковые токены сохраняются в `my.a`:
- `false` (по умолчанию): управляющие последовательности декодируются в реальные символы, обрамляющие кавычки убираются. `"hello\nworld"` → `hello` + перенос строки + `world`.
- `true`: строка хранится дословно, включая окружающие `"` и все `\`-последовательности без изменений. `"hello\nworld"` → `"hello\nworld"`. Полезно для обработки JSON без повторного кодирования строк.

```nim
import std/streams

var p: JsonParser
let s = newStringStream("""{"name": "Алиса", "age": 30}""")
p.open(s, "inline")
defer: p.close()
```

---

### `close`

```nim
proc close*(my: var JsonParser) {.inline.}
```

Закрывает парсер и освобождает ресурсы `BaseLexer` (вместе с ними и связанный поток). Всегда вызывайте `close` по завершении — или используйте `defer`.

```nim
p.open(s, "data.json")
defer: p.close()
```

---

## Управление парсером

### `next`

```nim
proc next*(my: var JsonParser)
```

Продвигает парсер к **следующему семантическому событию** и обновляет `my.kind`. Это центральная управляющая процедура — парсер ничего не делает до вызова `next`. После каждого вызова читайте `kind()`, чтобы узнать, что было найдено, и вызывайте `str`, `getInt` или `getFloat` по необходимости.

`next` реализован как конечный автомат со стеком состояний `ParserState`. Каждый структурный токен (`{`, `}`, `[`, `]`) добавляет в стек или извлекает из него состояние. Стек позволяет корректно обрабатывать произвольно вложенный JSON.

```nim
p.next()
while p.kind != jsonEof:
  case p.kind
  of jsonObjectStart: echo "{"
  of jsonObjectEnd:   echo "}"
  of jsonArrayStart:  echo "["
  of jsonArrayEnd:    echo "]"
  of jsonString:      echo "строка: ", p.str
  of jsonInt:         echo "целое: ",  p.getInt
  of jsonFloat:       echo "float: ",  p.getFloat
  of jsonTrue:        echo "true"
  of jsonFalse:       echo "false"
  of jsonNull:        echo "null"
  of jsonError:
    echo p.errorMsg
    break
  of jsonEof: break
  p.next()
```

---

### `getTok`

```nim
proc getTok*(my: var JsonParser): TokKind
```

Лексирует **следующий сырой токен** из ввода, предварительно пропуская пробелы и комментарии, и возвращает его `TokKind`. Обновляет `my.tok` и `my.a`. В отличие от `next`, **не** обновляет `my.kind` и не запускает конечный автомат — работает на чисто лексическом уровне, ниже `next`.

Используйте `getTok`, когда реализуете логику разбора на уровне токенов — например, в `eat`, или при интеграции `parsejson` с другим парсером, требующим прямого доступа к токенам.

```nim
let tok = p.getTok()
if tok == tkString:
  echo "сырой строковый токен: ", p.a
```

---

## Чтение значений

### `kind`

```nim
proc kind*(my: JsonParser): JsonEventKind {.inline.}
```

Возвращает текущий вид события, установленный последним вызовом `next`. Всегда сначала вызывайте `next`; чтение `kind` до любого вызова `next` возвращает начальное значение `jsonError`.

```nim
p.next()
if p.kind == jsonString:
  echo p.str
```

---

### `str`

```nim
proc str*(my: JsonParser): string {.inline.}
```

Возвращает текстовое содержимое текущего токена. Допустимо только когда `kind` равен `jsonString`, `jsonInt` или `jsonFloat`. В режиме отладки проверяет это утверждением.

- Для `jsonString`: декодированное строковое значение (или сырая строка в кавычках, если `rawStringLiterals = true`).
- Для `jsonInt` / `jsonFloat`: число так, как оно записано в источнике, например `"42"` или `"3.14e-2"`.

```nim
p.next()
assert p.kind == jsonString
echo p.str   # фактическое строковое значение без кавычек
```

---

### `getInt`

```nim
proc getInt*(my: JsonParser): BiggestInt {.inline.}
```

Разбирает текст текущего токена и возвращает его как `BiggestInt`. Допустимо только при `kind == jsonInt`. Внутренне вызывает `parseBiggestInt(my.a)`.

`BiggestInt` — это `int64` на 64-битных платформах. Для значений, помещающихся в обычный `int`, достаточно привести: `int(p.getInt())`.

```nim
p.next()
assert p.kind == jsonInt
let n = p.getInt()   # например, 42
```

---

### `getFloat`

```nim
proc getFloat*(my: JsonParser): float {.inline.}
```

Разбирает текст текущего токена и возвращает его как `float`. Допустимо только при `kind == jsonFloat`. Внутренне вызывает `parseFloat(my.a)`.

```nim
p.next()
assert p.kind == jsonFloat
let f = p.getFloat()   # например, 3.14
```

---

## Информация о позиции

### `getColumn`

```nim
proc getColumn*(my: JsonParser): int {.inline.}
```

Возвращает **1-индексированный номер столбца** текущей позиции парсера. Полезен для сообщений об ошибках и диагностики.

```nim
if p.kind == jsonError:
  echo "в столбце ", p.getColumn
```

---

### `getLine`

```nim
proc getLine*(my: JsonParser): int {.inline.}
```

Возвращает **1-индексированный номер строки** текущей позиции парсера. Все варианты переносов строк (CR, LF, CRLF) корректно отслеживаются базовым `BaseLexer`.

```nim
echo "строка: ", p.getLine, ", столбец: ", p.getColumn
```

---

### `getFilename`

```nim
proc getFilename*(my: JsonParser): string {.inline.}
```

Возвращает строку имени файла, переданную в `open`. Исключительно информационная — это любой ярлык, выбранный вами.

```nim
echo p.getFilename   # то, что вы передали как filename в open()
```

---

## Обработка ошибок

### `errorMsg`

```nim
proc errorMsg*(my: JsonParser): string
```

Возвращает форматированную строку ошибки для текущего события `jsonError`. Формат: `"имяФайла(строка, столбец) Error: сообщение"`. Допустимо только при `kind == jsonError`; иначе срабатывает утверждение.

```nim
p.next()
if p.kind == jsonError:
  echo p.errorMsg
  # например → "data.json(3, 12) Error: ':' expected"
```

---

### `errorMsgExpected`

```nim
proc errorMsgExpected*(my: JsonParser, e: string): string
```

Формирует сообщение об ошибке в том же формате `"имяФайла(строка, столбец) Error: <e> expected"`, но текст `e` задаёте вы сами, а не берёте из внутреннего поля ошибки парсера. Используется в `raiseParseErr` для форматирования пользовательских сообщений об ожидаемых элементах.

```nim
if p.kind != jsonString:
  echo p.errorMsgExpected("строка-ключ")
  # → "data.json(1, 2) Error: строка-ключ expected"
```

---

### `raiseParseErr`

```nim
proc raiseParseErr*(p: JsonParser, msg: string) {.noinline, noreturn.}
```

Выбрасывает `JsonParsingError` с сообщением, построенным `errorMsgExpected(p, msg)`. Помечен `noreturn`, поэтому компилятор знает, что выполнение после этого вызова не продолжается. Используйте для сигнализации структурных нарушений в пользовательской логике разбора.

```nim
if p.kind != jsonObjectStart:
  p.raiseParseErr("объект")
  # выбрасывает JsonParsingError: "data.json(1, 1) Error: объект expected"
```

---

### `eat`

```nim
proc eat*(p: var JsonParser, tok: TokKind)
```

Проверяет, что **текущий сырой токен** (`p.tok`) совпадает с `tok`, затем продвигает лексер вызовом `getTok`. Если токен не совпадает, вызывает `raiseParseErr` с именем ожидаемого токена.

Это стандартный вспомогательный метод «проверить и продвинуться» для построения рекурсивно-нисходящих парсеров поверх `parsejson`. Работает на уровне `TokKind`, а не `JsonEventKind`.

```nim
# Поглотить открывающую скобку JSON-объекта:
discard p.getTok()   # слексировать первый токен
p.eat(tkCurlyLe)     # убедиться, что это '{', и продвинуться
```

---

### `parseEscapedUTF16`

```nim
proc parseEscapedUTF16*(buf: cstring, pos: var int): int
```

Разбирает ровно 4 шестнадцатеричных символа из `buf`, начиная с позиции `pos`, декодируя их как кодовую единицу UTF-16. Возвращает декодированное целое значение или `-1` при ошибке. При успехе продвигает `pos` за 4 hex-цифры.

Это публичная утилита, используемая внутри парсера строк для обработки управляющих последовательностей `\uXXXX`, включая декодирование суррогатных пар. Экспортируется, чтобы код поверх `parsejson` мог повторно использовать ту же логику разбора escape-последовательностей.

```nim
var pos = 0
let codeUnit = parseEscapedUTF16(cstring("1F600"), pos)
# pos теперь 4, codeUnit равен 0x1F60 (первые 4 hex-цифры)
```

Примечание: суррогатные пары (`\uD800\uDC00` и подобные) собираются в правильную кодовую точку Unicode внутри приватной процедуры `parseString`; `parseEscapedUTF16` сама читает только одну группу из 4 hex-цифр.

---

## Полный пример использования

### Потоковая обработка большого JSON-массива

```nim
import std/[parsejson, streams]

proc countInts(json: string): int =
  var p: JsonParser
  p.open(newStringStream(json), "input")
  defer: p.close()

  p.next()
  if p.kind != jsonArrayStart:
    p.raiseParseErr("массив")

  p.next()
  while p.kind != jsonArrayEnd:
    if p.kind == jsonError:
      echo p.errorMsg
      break
    if p.kind == jsonInt:
      inc result
    p.next()

echo countInts("[1, 2, \"пропустить\", 3, null, 4]")   # → 4
```

### Извлечение одного поля без полного дерева разбора

```nim
proc findName(json: string): string =
  var p: JsonParser
  p.open(newStringStream(json), "input")
  defer: p.close()

  var depth = 0
  var expectValue = false
  p.next()
  while p.kind notin {jsonEof, jsonError}:
    case p.kind
    of jsonObjectStart: inc depth
    of jsonObjectEnd:   dec depth
    of jsonString:
      if expectValue and depth == 1:
        return p.str
      expectValue = (depth == 1 and p.str == "name")
    else:
      expectValue = false
    p.next()

let data = """{"id": 1, "name": "Алиса", "role": "admin"}"""
echo findName(data)   # → Алиса
```

### Использование `rawStringLiterals` для дословного сохранения строк

```nim
var p: JsonParser
p.open(newStringStream("""{"key": "hello\nworld"}"""), "rt", rawStringLiterals = true)
defer: p.close()

p.next()   # jsonObjectStart
p.next()   # jsonString → имя ключа
p.next()   # jsonString → значение, управляющие последовательности сохранены
echo p.str   # → "hello\nworld"  (с кавычками и обратным слешем)
```

---

## Справочник последовательностей событий

Что `next` производит для типичных JSON-структур:

**Скалярное значение** `42`:
```
jsonInt   → getInt() == 42
jsonEof
```

**Объект** `{"a": 1}`:
```
jsonObjectStart
jsonString  → str() == "a"      (ключ)
jsonInt     → getInt() == 1     (значение)
jsonObjectEnd
jsonEof
```

**Массив** `[true, null]`:
```
jsonArrayStart
jsonTrue
jsonNull
jsonArrayEnd
jsonEof
```

**Вложенная структура** `{"x": [1, 2]}`:
```
jsonObjectStart
jsonString   → "x"
jsonArrayStart
jsonInt      → 1
jsonInt      → 2
jsonArrayEnd
jsonObjectEnd
jsonEof
```

---

## Таблица ошибок

| Константа ошибки | Смысл | Типичная причина |
|-----------------|-------|-----------------|
| `errNone` | Нет ошибки | — |
| `errInvalidToken` | Неверный символ или escape | Плохой `\uXXXX`, неизвестный токен |
| `errStringExpected` | Требовалась строка | Ключ объекта не является строкой в кавычках |
| `errColonExpected` | Отсутствует `:` | `{"a" 1}` вместо `{"a": 1}` |
| `errCommaExpected` | Отсутствует `,` | `[1 2]` вместо `[1, 2]` |
| `errBracketRiExpected` | Отсутствует `]` | Массив не закрыт |
| `errCurlyRiExpected` | Отсутствует `}` | Объект не закрыт |
| `errQuoteExpected` | Незакрытая строка | Строковый литерал достиг EOF |
| `errEOC_Expected` | Незакрытый комментарий | `/* ...` без `*/` |
| `errEofExpected` | Лишние токены после корневого значения | `42 43` (два корневых значения) |
| `errExprExpected` | Ожидалось выражение | Отсутствует значение в объекте |

| Тип исключения | Вызывается в | Наследует от |
|---------------|-------------|-------------|
| `JsonParsingError` | `raiseParseErr` | `ValueError` |
| `JsonKindError` | макрос `json.to` | `ValueError` |
