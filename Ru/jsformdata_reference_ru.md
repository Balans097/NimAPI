# Справочник модуля `jsformdata`

> **Модуль:** `std/jsformdata` (стандартная библиотека Nim)  
> **Цель компиляции:** Только JavaScript — использование с нативным бэкендом является фатальной ошибкой компиляции.  
> **Назначение:** Nim-привязки к браузерному [`FormData`](https://developer.mozilla.org/en-US/docs/Web/API/FormData) API для построения и управления данными в форматке перед отправкой через `fetch()` или `XMLHttpRequest`.

---

## Контекст: что такое FormData?

`FormData` — нативная браузерная структура данных, представляющая набор пар ключ-значение, точно так же, как данные отправляются пользователем через HTML-элемент `<form>` с `enctype="multipart/form-data"`. Стандартный способ для:

- Загрузки файлов вместе с другими полями на сервер.
- Отправки структурированных данных формы через `fetch()` без ручной сериализации в JSON или URL-кодированные строки.
- Программного заполнения или модификации данных формы перед отправкой.

**Принципиальное поведенческое отличие от обычного словаря:** `FormData` — это **мультимап** — он намеренно допускает несколько значений под одним ключом. Это соответствует тому, как работают HTML-чекбоксы и поля множественного выбора. Из-за этого операции `add` и `[]=` имеют разную семантику:

- `add` (→ `FormData.append`) **добавляет** новую запись, даже если ключ уже существует. Все записи с этим ключом накапливаются.
- `[]=` (→ `FormData.set`) **заменяет** все существующие записи для ключа одной новой.

Понимание этого различия — самое важное, что нужно знать перед использованием модуля.

---

## Тип `FormData`

```nim
type FormData* = ref object of JsRoot
```

Ref-объект, оборачивающий JavaScript-объект `FormData`. Как ref-тип, передаётся по ссылке и собирается сборщиком мусора. Все мутации выполняются на месте над базовым JavaScript-объектом.

---

## Экспортируемые функции

---

### `newFormData`

```nim
func newFormData*(): FormData
```

**Что делает**

Создаёт новый пустой объект `FormData`. Соответствует `new FormData()` в JavaScript. Результирующий объект имеет ноль записей и готов принимать данные через `add`, `[]=` или `put`.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
assert data.len == 0
```

---

### `add` (без имени файла)

```nim
func add*(self: FormData; name: cstring; value: SomeNumber | bool | cstring | Blob)
```

**Что делает**

Добавляет новую пару ключ-значение в `FormData`, вызывая JavaScript `FormData.append(name, value)`. Если запись с `name` уже существует, новая запись добавляется **рядом с ней** — существующая не заменяется. Обе записи сосуществуют в порядке добавления.

Это ключевое поведенческое отличие от `[]=`: `add` накапливает, `[]=` заменяет.

Допустимые типы значений: любое Nim-число (`int`, `float` и т. д. через `SomeNumber`), `bool`, `cstring` или DOM-объект `Blob`.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data.add("color", "red".cstring)
data.add("color", "blue".cstring)   # НЕ заменяет "red"

let colors = data.getAll("color")
assert colors == @["red".cstring, "blue".cstring]  # обе присутствуют
```

---

### `add` (с именем файла)

```nim
func add*(self: FormData; name: cstring; value: SomeNumber | bool | cstring | Blob; filename: cstring)
```

**Что делает**

Трёхаргументный вариант `add`, прикрепляющий подсказку `filename` к записи. Когда `value` является `Blob` или `File`, это имя файла отправляется серверу как предлагаемое имя в заголовке `Content-Disposition` этой multipart-части. Для не-Blob значений аргумент `filename` технически принимается, но большинство серверов его игнорирует.

Как и двухаргументный вариант, эта функция **добавляет** запись, не заменяя существующие для того же ключа.

**Пример**

```nim
import std/[jsformdata, dom]

let data = newFormData()
let blob: Blob = ...   # получен из file input или создан программно
data.add("upload", blob, "report.pdf".cstring)
```

---

### `delete`

```nim
func delete*(self: FormData; name: cstring)
```

**Что делает**

Удаляет **все** записи, ключ которых совпадает с `name`, вызывая JavaScript `FormData.delete(name)`. Это операция «всё или ничего»: если под `"color"` было добавлено три записи, все три удаляются. Удалить только одну запись из группы дубликатов с помощью этой функции нельзя — для этого нужно вызвать `getAll`, отфильтровать нужное, удалить все через `delete` и заново добавить через `add`.

Если записи с `name` нет, вызов не вызывает ошибок — это no-op.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data.add("tag", "alpha".cstring)
data.add("tag", "beta".cstring)
data.add("tag", "gamma".cstring)

data.delete("tag")       # удаляет ВСЕ три записи "tag"
assert not data.hasKey("tag")
```

---

### `getAll`

```nim
func getAll*(self: FormData; name: cstring): seq[cstring]
```

**Что делает**

Возвращает `seq[cstring]` со **всеми значениями** для заданного ключа в порядке добавления. Это правильный способ читать данные из ключа, у которого может быть несколько значений (добавленных через `add`). Если ключ не существует, возвращается пустой seq.

В отличие от `[]`, который возвращает только **первое** значение для ключа.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data.add("fruit", "apple".cstring)
data.add("fruit", "banana".cstring)
data.add("fruit", "cherry".cstring)

let fruits = data.getAll("fruit")
assert fruits.len == 3
assert fruits[0] == "apple".cstring
assert fruits[2] == "cherry".cstring

# Несуществующий ключ:
assert data.getAll("vegetable") == @[]
```

---

### `hasKey`

```nim
func hasKey*(self: FormData; name: cstring): bool
```

**Что делает**

Возвращает `true`, если в `FormData` существует хотя бы одна запись с заданным ключом, вызывая JavaScript `FormData.has(name)`. Иначе возвращает `false`. Это безопасная проверка существования — исключений не вызывает.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data["username"] = "alice".cstring

assert data.hasKey("username") == true
assert data.hasKey("password") == false
```

---

### `keys`

```nim
func keys*(self: FormData): seq[cstring]
```

**Что делает**

Возвращает `seq[cstring]` всех ключей `FormData` в порядке добавления. Если один ключ встречается несколько раз (через `add`), он появляется в возвращаемой последовательности несколько раз. Оборачивает `Array.from(formData.keys())`.

Это поведение отличается от того, чего можно ожидать при трактовке `FormData` как словаря — перебор ключей даёт полную картину с дубликатами, а не дедуплицированное множество.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data.add("a", "1".cstring)
data.add("b", "2".cstring)
data.add("a", "3".cstring)   # дублирующийся ключ

let ks = data.keys()
assert ks == @["a".cstring, "b".cstring, "a".cstring]
# "a" встречается дважды, потому что есть две записи с этим ключом
```

---

### `values`

```nim
func values*(self: FormData): seq[cstring]
```

**Что делает**

Возвращает `seq[cstring]` всех значений `FormData` в порядке добавления — одно значение на запись. Оборачивает `Array.from(formData.values())`. Значения возвращаются в той же позиционной последовательности, что и соответствующие записи в `keys()` и `pairs()`.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data.add("x", "10".cstring)
data.add("y", "20".cstring)

let vs = data.values()
assert vs == @["10".cstring, "20".cstring]
```

---

### `pairs`

```nim
func pairs*(self: FormData): seq[tuple[key, val: cstring]]
```

**Что делает**

Возвращает `seq` кортежей `(key, val)` для каждой записи `FormData` в порядке добавления. Оборачивает `Array.from(formData.entries())`. Это наиболее полное представление данных: в отличие от отдельных `keys()` или `values()`, каждый кортеж держит ключ и значение вместе, и все дубликаты видны.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data.add("name", "Алиса".cstring)
data.add("role", "admin".cstring)
data.add("role", "editor".cstring)

for (k, v) in data.pairs():
  echo $k, " = ", $v
# name = Алиса
# role = admin
# role = editor
```

---

### `put` (с именем файла)

```nim
func put*(self: FormData; name: cstring; value: SomeNumber | bool | cstring | Blob; filename: cstring)
```

**Что делает**

Устанавливает (заменяет) запись для `name` значением `value` с прикреплённым `filename`, вызывая JavaScript `FormData.set(name, value, filename)`. Все существующие записи для `name` удаляются и заменяются этой одной новой записью. `filename` прикрепляется как подсказка content-disposition — полезно при загрузке файлов-Blob.

Это трёхаргументный аналог `[]=` для случаев с файловой подсказкой.

**Пример**

```nim
import std/[jsformdata, dom]

let data = newFormData()
let blob: Blob = ...
data.put("attachment", blob, "final_report.pdf".cstring)
# Любые предыдущие записи "attachment" удалены; осталась только эта
```

---

### `[]=` — установить (заменить)

```nim
func `[]=`*(self: FormData; name: cstring; value: SomeNumber | bool | cstring | Blob)
```

**Что делает**

Устанавливает запись для `name` в `value`, вызывая JavaScript `FormData.set(name, value)`. Если записи для `name` уже существуют, они **все удаляются** и заменяются этой одной новой записью. Это операция **замены** — в противоположность `add`, которая является операцией **добавления**.

Используйте `[]=`, когда у ключа должно быть ровно одно значение. Используйте `add`, когда нужно накапливать несколько значений.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data.add("status", "draft".cstring)
data.add("status", "pending".cstring)   # теперь 2 записи "status"

data["status"] = "published".cstring    # заменяет ОБЕ одной записью

assert data.getAll("status") == @["published".cstring]
```

---

### `[]` — получить (первое значение)

```nim
func `[]`*(self: FormData; name: cstring): cstring
```

**Что делает**

Возвращает **первое** значение, связанное с `name`, вызывая JavaScript `FormData.get(name)`. Если записи с `name` нет — возвращает `nil` (JavaScript `null`). Если под одним ключом несколько записей — возвращается только первая; используйте `getAll` для получения всех.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data.add("lang", "en".cstring)
data.add("lang", "fr".cstring)

assert data["lang"] == "en".cstring    # только первое значение
assert data.getAll("lang").len == 2    # оба доступны через getAll

# Несуществующий ключ возвращает nil:
assert data["missing"] == nil
```

---

### `clear`

```nim
func clear*(self: FormData)
```

**Что делает**

Удаляет все записи из `FormData`, оставляя его пустым. Это вспомогательная функция, реализованная в Nim (не прямое отображение нативного JavaScript-метода, поскольку в веб-спецификации у `FormData` нет `clear()`). Внутри она получает все текущие ключи и вызывает `delete` для каждого из них, в результате чего объект становится полностью пустым.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data["a"] = "1".cstring
data["b"] = "2".cstring
data.add("c", "3".cstring)

assert data.len == 3
data.clear()
assert data.len == 0
assert not data.hasKey("a")
```

---

### `toCstring`

```nim
func toCstring*(self: FormData): cstring
```

**Что делает**

Сериализует объект `FormData` в JSON-`cstring`, вызывая JavaScript `JSON.stringify()`. Полезно для отладки и логирования — позволяет быстро просмотреть содержимое без перебора через `pairs()`.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data["user"] = "bob".cstring
echo $data.toCstring()
```

---

### `$`

```nim
func `$`*(self: FormData): string
```

**Что делает**

Оператор `$` (stringify) Nim для `FormData`. Возвращает Nim-строку с JSON-представлением объекта. Тонкая обёртка над `toCstring`.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
data["x"] = "42".cstring
echo $data
```

---

### `len`

```nim
func len*(self: FormData): int
```

**Что делает**

Возвращает общее количество **записей** в `FormData`. Поскольку дублирующиеся ключи допустимы, `len` считает отдельные записи, а не уникальные ключи. `FormData`, в котором ключ `"color"` был добавлен три раза, имеет `len == 3`, а не `len == 1`.

Реализовано через перечисление всех записей: `Array.from(formData.entries()).length`.

**Пример**

```nim
import std/jsformdata

let data = newFormData()
assert data.len == 0

data.add("tag", "nim".cstring)
data.add("tag", "js".cstring)
data.add("tag", "web".cstring)
assert data.len == 3   # 3 записи, все под ключом "tag"

data.delete("tag")
assert data.len == 0
```

---

## `add` против `[]=`: центральное различие

Это наиболее частый источник путаницы при работе с `FormData`. Сравнение бок о бок:

```nim
import std/jsformdata

let data = newFormData()

# --- add (добавить) ---
data.add("color", "red".cstring)
data.add("color", "blue".cstring)
assert data.getAll("color") == @["red".cstring, "blue".cstring]  # обе присутствуют
assert data.len == 2

# --- []= (установить/заменить) ---
data["color"] = "green".cstring    # удаляет "red" И "blue", добавляет "green"
assert data.getAll("color") == @["green".cstring]                # только одна
assert data.len == 1
```

**Практическое правило:**
- Используйте `[]=`, когда у ключа должно быть **ровно одно значение** (как у текстового поля формы).
- Используйте `add`, когда у ключа может быть **несколько значений** (как у множественного выбора или группы чекбоксов).

---

## Практические примеры

### Формирование данных для входа в систему

```nim
import std/[jsformdata, jsfetch, asyncjs]
from std/httpcore import HttpPost

proc login(username, password: cstring) {.async.} =
  let data = newFormData()
  data["username"] = username
  data["password"] = password

  let opts = newFetchOptions(metod = HttpPost, body = nil)
  # FormData передаётся напрямую как тело запроса:
  let resp = await fetch("https://api.example.com/login".cstring, opts)
  echo $resp.status
```

### Загрузка файла с метаданными

```nim
import std/[jsformdata, dom]

let data = newFormData()
data["description"] = "Ежемесячный отчёт".cstring
data["category"] = "finance".cstring
# Добавляем blob с подсказкой имени файла:
let file: Blob = ...  # например, из элемента file input
data.add("file", file, "report_march.pdf".cstring)
```

### Просмотр всех записей

```nim
import std/jsformdata

let data = newFormData()
data.add("sizes", "S".cstring)
data.add("sizes", "M".cstring)
data.add("sizes", "L".cstring)
data["color"] = "blue".cstring

echo "Ключи: ", $data.keys()    # @["sizes", "sizes", "sizes", "color"]
echo "Длина: ", data.len         # 4

for (k, v) in data.pairs():
  echo $k, " → ", $v
# sizes → S
# sizes → M
# sizes → L
# color → blue
```

---

## Сводная таблица

| Символ | Вид | JS-эквивалент | Назначение |
|--------|-----|--------------|------------|
| `FormData` | тип | `FormData` | Мультимап записей ключ-значение формы |
| `newFormData()` | func | `new FormData()` | Создать пустой `FormData` |
| `add(name, value)` | func | `.append(name, value)` | **Добавить** — добавляет рядом с существующими |
| `add(name, value, filename)` | func | `.append(name, value, filename)` | Добавить с подсказкой имени файла (для Blob) |
| `delete(name)` | func | `.delete(name)` | Удалить **все** записи для ключа |
| `getAll(name)` | func | `.getAll(name)` | Получить все значения для ключа как `seq` |
| `hasKey(name)` | func | `.has(name)` | Проверить наличие хотя бы одной записи с ключом |
| `keys()` | func | `Array.from(.keys())` | Все ключи по порядку (с дубликатами) |
| `values()` | func | `Array.from(.values())` | Все значения по порядку |
| `pairs()` | func | `Array.from(.entries())` | Все кортежи (ключ, значение) по порядку |
| `put(name, value, filename)` | func | `.set(name, value, filename)` | **Заменить** все записи ключа, с именем файла |
| `data[name] = value` | func | `.set(name, value)` | **Заменить** все записи ключа |
| `data[name]` | func | `.get(name)` | Получить **первое** значение (`nil` при отсутствии) |
| `clear()` | func | *(удобство Nim)* | Удалить все записи |
| `toCstring()` | func | `JSON.stringify()` | JSON-сериализация в `cstring` |
| `$` | func | *(Nim)* | JSON-сериализация в `string` |
| `len` | func | `Array.from(.entries()).length` | Общее число записей (с дубликатами) |
