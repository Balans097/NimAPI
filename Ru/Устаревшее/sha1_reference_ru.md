# std/sha1 — справочник модуля

> **Входит в стандартную библиотеку Nim** (`std/sha1`).

---

> ⚠️ **УСТАРЕЛ (DEPRECATED)**
>
> Этот модуль объявлен устаревшим. Рекомендуемая замена — пакет `checksums` из Nimble:
>
> ```
> nimble install checksums
> ```
>
> Затем импортируйте:
>
> ```nim
> import checksums/sha1
> ```
>
> API в `checksums/sha1` полностью идентичен этому модулю. Весь код, написанный под `std/sha1`, переносится заменой одной строки импорта.

---

## Обзор

`std/sha1` реализует [SHA-1 (Secure Hash Algorithm 1)](https://ru.wikipedia.org/wiki/SHA-1) — криптографическую хеш-функцию, принимающую на вход данные произвольной длины и производящую фиксированный 160-битный (20-байтовый) дайджест, который обычно представляется как строка из 40 символов в верхнем регистре шестнадцатеричной нотации.

### Важное замечание по безопасности

SHA-1 **криптографически взломан** и не должен применяться в целях, критичных для безопасности: цифровые подписи, проверка сертификатов, хеширование паролей. Практические атаки на коллизии SHA-1 были продемонстрированы. Для задач, требующих криптографической стойкости, используйте SHA-256 или SHA-3.

SHA-1 остаётся применимым в некриптографических задачах: контрольные суммы файлов для обнаружения случайных ошибок, сравнение идентичности данных в системах сборки или кешах, генерация детерминированных идентификаторов там, где устойчивость к намеренно подобранным коллизиям не требуется.

### Два стиля использования

Модуль предлагает два уровня API:

**Высокоуровневый (однократный вызов):** `secureHash` и `secureHashFile` делают всё за один вызов. Используйте их, когда всё входное значение доступно целиком.

**Низкоуровневый (потоковый):** `newSha1State` → `update` → `finalize`. Используйте, когда данные поступают порциями — сетевые потоки, большие файлы, читаемые по частям, инкрементальное построение входных данных.

---

## Типы

### `Sha1Digest`

```nim
type Sha1Digest* = array[0 .. 19, uint8]
```

«Сырой» 20-байтовый (160-битный) массив, хранящий бинарный SHA-1 дайджест. Это непосредственный результат хеш-вычисления. Вы редко будете работать с этим типом напрямую — `SecureHash` оборачивает его и предоставляет более удобный интерфейс.

---

### `SecureHash`

```nim
type SecureHash* = distinct Sha1Digest
```

`Distinct`-обёртка над `Sha1Digest`. Благодаря `distinct` компилятор не позволит случайно перепутать `SecureHash` с обычным байтовым массивом — они сравниваются только через предоставленный оператор `==` и преобразуются в строку и обратно через `$` и `parseSecureHash`. Это тип, возвращаемый `secureHash`, `secureHashFile` и `parseSecureHash`, и именно с ним вы будете работать в большинстве кода.

---

### `Sha1State`

```nim
type Sha1State* = object
  count: int
  state: array[5, uint32]
  buf:   array[64, byte]
```

Изменяемый объект состояния для инкрементального (потокового) хеширования. Хранит текущее состояние вычисления SHA-1: счётчик байтов, пять 32-битных рабочих переменных и 64-байтовый буфер блока. Создаётся через `newSha1State`, наполняется данными через `update`, финальный дайджест извлекается через `finalize`. После вызова `finalize` состояние нельзя переиспользовать — для следующего хеша создайте новое.

---

## Низкоуровневый потоковый API

Эти три процедуры образуют конвейер. Используйте их, когда данные поступают инкрементально.

---

### `newSha1State`

```nim
proc newSha1State*(): Sha1State
```

**Создаёт и возвращает свежеинициализированный `Sha1State`.**

Состояние устанавливается в пять начальных значений SHA-1, предписанных стандартом (`0x67452301`, `0xEFCDAB89`, `0x98BADCFE`, `0x10325476`, `0xC3D2E1F0`). Счётчик байтов сбрасывается в ноль. Это отправная точка каждого потокового хеш-вычисления.

Если вы используете высокоуровневые `secureHash` или `secureHashFile`, вам никогда не нужно вызывать это напрямую — они делают это внутри себя.

**Пример:**

```nim
import std/sha1

var state = newSha1State()
state.update("первая порция")
state.update(" вторая порция")
let digest: Sha1Digest = state.finalize()
```

---

### `update`

```nim
proc update*(ctx: var Sha1State, data: openArray[char])
```

**Подаёт порцию данных в текущее вычисление SHA-1.**

Можно вызывать `update` любое количество раз с порциями любого размера. Алгоритм буферизует данные внутри в 64-байтовых блоках; как только накапливается полный блок, он немедленно обрабатывается и буфер очищается. Последний неполный блок удерживается до вызова `finalize`.

Хеш строки, поданной целиком за раз, идентичен хешу той же строки, поданной множеством маленьких частей — разбивка на порции незаметна для результата.

После того как `finalize` был вызван, повторный вызов `update` даёт неопределённое поведение. Создайте новый `Sha1State`, если нужно хешировать что-то ещё.

**Пример — хеширование большого файла порциями:**

```nim
import std/sha1

proc hashStream(f: File): SecureHash =
  var state = newSha1State()
  var buf = newString(8192)
  while true:
    let n = readChars(f, buf)
    if n == 0: break
    buf.setLen(n)
    state.update(buf)
    if n < 8192: break
  SecureHash(state.finalize())
```

**Пример — подача данных несколькими вызовами:**

```nim
import std/sha1

var state = newSha1State()
state.update("Hello")
state.update(", ")
state.update("World")
let h = SecureHash(state.finalize())
assert h == parseSecureHash("0A4D55A8D778E5022FAB701977C5D840BBC486D0")
# Тот же результат, что и secureHash("Hello, World")
```

---

### `finalize`

```nim
proc finalize*(ctx: var Sha1State): Sha1Digest
```

**Завершает вычисление SHA-1 и возвращает 20-байтовый дайджест.**

`finalize` выполняет обязательное выравнивание по стандарту SHA-1: добавляет бит `1`, затем достаточное количество нулевых байтов, чтобы общая длина составила 56 байт по модулю 64, и наконец кодирует исходную длину сообщения как 64-битное целое в формате big-endian для заполнения последнего блока. Это выравнивание требуется спецификацией SHA-1 и гарантирует, что дайджест корректно отражает полное входное значение независимо от его длины.

Возвращаемое значение — `Sha1Digest` (сырой `array[20, uint8]`). Оберните его в `SecureHash(...)`, чтобы получить типизированный хеш, который можно сравнивать и преобразовывать в строку.

**Пример:**

```nim
import std/sha1

var state = newSha1State()
state.update("Hello World")
let raw: Sha1Digest = state.finalize()
let hash = SecureHash(raw)
echo $hash   # "0A4D55A8D778E5022FAB701977C5D840BBC486D0"
```

---

## Высокоуровневый однократный API

---

### `secureHash`

```nim
proc secureHash*(str: openArray[char]): SecureHash
```

**Хеширует строку (или любой `openArray[char]`) за один вызов и возвращает её `SecureHash`.**

Это наиболее удобная точка входа для хеширования данных, которые уже целиком находятся в памяти. Внутри создаётся `Sha1State`, вызывается `update` один раз со всем входом, и возвращается `SecureHash(state.finalize())`. Нет никакого смысла использовать потоковый API, если ваши данные уже представлены строкой или массивом.

**Пример:**

```nim
import std/sha1

let hash = secureHash("Hello World")
assert $hash == "0A4D55A8D778E5022FAB701977C5D840BBC486D0"

let nameHash = secureHash("John Doe")
assert $nameHash == "AE6E4D1209F17B460503904FAD297B31E9CF6362"
```

**Пример — использование как ключ кеша:**

```nim
import std/sha1, std/tables

var cache: Table[string, string]

proc cachedProcess(input: string): string =
  let key = $secureHash(input)
  if key in cache:
    return cache[key]
  result = expensiveOperation(input)
  cache[key] = result
```

---

### `secureHashFile`

```nim
proc secureHashFile*(filename: string): SecureHash
```

**Хеширует всё содержимое файла и возвращает его `SecureHash`.**

Файл читается внутренне порциями по 8192 байта через потоковый API, поэтому файлы произвольного размера обрабатываются без полной загрузки в память. Файл открывается и закрывается автоматически. Если файл не удаётся открыть, возбуждается стандартное исключение ввода-вывода Nim.

**Пример — базовая контрольная сумма файла:**

```nim
import std/sha1

let hash = secureHashFile("myproject.nim")
echo $hash
```

**Пример — проверка файла по известному хешу:**

```nim
import std/sha1

let
  computed = secureHashFile("downloaded.zip")
  expected = parseSecureHash("10DFAEBF6BFDBC7939957068E2EFACEC4972933C")

if computed == expected:
  echo "Целостность файла подтверждена."
else:
  echo "НЕСОВПАДЕНИЕ — файл может быть повреждён или подменён."
```

---

## Преобразование и сравнение

---

### `$`

```nim
proc `$`*(self: SecureHash): string
```

**Преобразует `SecureHash` в канонический 40-символьный шестнадцатеричный дамп в верхнем регистре.**

Каждый из 20 байтов отображается как два шестнадцатеричных символа, всегда в верхнем регистре, всегда с ведущим нулём если значение байта меньше 16. Результат всегда ровно 40 символов в длину.

**Пример:**

```nim
import std/sha1

let hash = secureHash("Hello World")
let s = $hash
assert s == "0A4D55A8D778E5022FAB701977C5D840BBC486D0"
assert s.len == 40
```

---

### `parseSecureHash`

```nim
proc parseSecureHash*(hash: string): SecureHash
```

**Разбирает 40-символьную шестнадцатеричную строку и возвращает `SecureHash`.**

Это обратная операция к `$`. Используйте её для восстановления `SecureHash` из хеш-строки, ранее сохранённой в файле, базе данных, конфигурации или полученной по сети. Внутренний `parseHexInt` принимает как заглавные, так и строчные шестнадцатеричные цифры.

Входная строка должна быть ровно 40 символов и содержать только допустимые шестнадцатеричные цифры. Валидация не выполняется — передача некорректной строки породит мусорный `SecureHash` или ошибку времени выполнения. Используйте `isValidSha1Hash` перед вызовом, чтобы защититься от плохих входных данных.

**Пример:**

```nim
import std/sha1

let stored = "0A4D55A8D778E5022FAB701977C5D840BBC486D0"
let hash = parseSecureHash(stored)
assert hash == secureHash("Hello World")
```

---

### `==`

```nim
proc `==`*(a, b: SecureHash): bool
```

**Возвращает `true`, если два значения `SecureHash` идентичны — то есть представляют одинаковый дайджест.**

Два хеша равны тогда и только тогда, когда все 20 их байтов совпадают. На практике это означает, что хешировавшиеся входные данные дали одинаковый 160-битный результат.

> **Примечание:** это сравнение **не является** константным по времени. Его не безопасно использовать для верификации секретов (например, сравнения хеша пароля с сохранённым значением в потоке аутентификации). Для этих целей применяйте специализированную функцию сравнения с константным временем.

**Пример:**

```nim
import std/sha1

let a = secureHash("Hello World")
let b = secureHash("Goodbye World")
let c = parseSecureHash("0A4D55A8D778E5022FAB701977C5D840BBC486D0")

assert a != b   # разные входные данные → разные хеши
assert a == c   # одинаковый дайджест из двух разных источников
```

---

### `isValidSha1Hash`

```nim
proc isValidSha1Hash*(s: string): bool
```

**Возвращает `true`, если `s` выглядит как корректная строка SHA-1 дайджеста в шестнадцатеричном формате.**

Проверяет два свойства: строка должна быть ровно 40 символов, и каждый символ должен быть шестнадцатеричной цифрой (`0`–`9`, `a`–`f`, `A`–`F`). Это только проверка формата — не верификация того, что строка соответствует какому-либо конкретному входному значению.

Используйте это перед вызовом `parseSecureHash`, чтобы защититься от некорректного ввода и избежать ошибок времени выполнения.

**Пример:**

```nim
import std/sha1

assert isValidSha1Hash("0A4D55A8D778E5022FAB701977C5D840BBC486D0") == true
assert isValidSha1Hash("0a4d55a8d778e5022fab701977c5d840bbc486d0") == true  # строчные — OK
assert isValidSha1Hash("ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ") == false  # не шестнадцатеричные
assert isValidSha1Hash("0A4D55")  == false  # слишком коротко
assert isValidSha1Hash("")        == false  # пустая строка
```

**Пример — безопасный разбор с валидацией:**

```nim
import std/sha1

proc tryParseHash(s: string): SecureHash =
  if not isValidSha1Hash(s):
    raise newException(ValueError, "Некорректный SHA-1 хеш: " & s)
  parseSecureHash(s)
```

---

## Типичные сценарии использования

### Сценарий 1 — Хешировать строку и сравнить с известным значением

```nim
import std/sha1

let hash = secureHash("my data")
echo $hash
```

### Сценарий 2 — Проверить скачанный файл

```nim
import std/sha1

let expected = parseSecureHash("DEADBEEFDEADBEEFDEADBEEFDEADBEEFDEADBEEF")
let actual   = secureHashFile("file.bin")

assert actual == expected, "Проверка целостности не прошла!"
```

### Сценарий 3 — Инкрементальное хеширование потока

```nim
import std/sha1

var state = newSha1State()
state.update(headerBytes)
state.update(bodyBytes)
state.update(trailerBytes)
let hash = SecureHash(state.finalize())
echo $hash
```

### Сценарий 4 — Безопасный круговой маршрут через строковое представление

```nim
import std/sha1

let original = secureHash("some input")
let asString = $original                   # сериализация
let restored = parseSecureHash(asString)   # десериализация
assert original == restored
```

---

## Сводная таблица

| Символ | Сигнатура | Назначение |
|--------|-----------|------------|
| `Sha1Digest` | `array[20, uint8]` | Сырой бинарный 160-битный дайджест |
| `SecureHash` | `distinct Sha1Digest` | Типизированное хеш-значение для сравнения и вывода |
| `Sha1State` | `object` | Изменяемое состояние для инкрементального хеширования |
| `newSha1State()` | `() → Sha1State` | Инициализировать новое потоковое хеш-вычисление |
| `update(ctx, data)` | `(var Sha1State, openArray[char]) → void` | Подать порцию данных в вычисление |
| `finalize(ctx)` | `(var Sha1State) → Sha1Digest` | Завершить вычисление и извлечь дайджест |
| `secureHash(str)` | `openArray[char] → SecureHash` | Однократно: хешировать строку |
| `secureHashFile(name)` | `string → SecureHash` | Однократно: хешировать файл по пути |
| `$hash` | `SecureHash → string` | Форматировать как 40-символьный hex в верхнем регистре |
| `parseSecureHash(s)` | `string → SecureHash` | Разобрать 40-символьную hex-строку |
| `a == b` | `(SecureHash, SecureHash) → bool` | Сравнить два дайджеста |
| `isValidSha1Hash(s)` | `string → bool` | Проверить формат перед разбором |

---

> **Смотри также:** [`checksums/sha1`](https://github.com/nim-lang/checksums) — рекомендуемая замена этого устаревшего модуля; [`std/md5`](https://nim-lang.org/docs/md5.html) — алгоритм контрольной суммы MD5; [`std/hashes`](https://nim-lang.org/docs/hashes.html) — быстрое некриптографическое хеширование типов Nim; [`std/base64`](https://nim-lang.org/docs/base64.html) — кодирование и декодирование Base64.
