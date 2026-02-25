# Справочник модуля `jsfetch`

> **Модуль:** `std/jsfetch` (стандартная библиотека Nim)  
> **Цель компиляции:** Только JavaScript — использование с нативным бэкендом является фатальной ошибкой компиляции.  
> **Назначение:** Идиоматичные Nim-привязки к [Fetch API](https://developer.mozilla.org/docs/Web/API/Fetch_API) браузера для выполнения HTTP-запросов из Nim-кода, компилируемого в JavaScript.  
> **Реэкспортирует:** `std/jsformdata`, `std/jsheaders`

---

## Контекст: Fetch API в Nim

Fetch API браузера — современный стандарт для HTTP-запросов из JavaScript. Он основан на Promises (асинхронный), пришёл на смену устаревшему `XMLHttpRequest` и поддерживается во всех современных браузерах и Node.js.

Этот модуль отображает Fetch API на Nim-идиомы: JavaScript Promises становятся значениями `Future[T]` из `std/asyncjs`, строковые опции заменяются типизированными перечислениями, а конфигурация запроса выражается через структурированный объект `FetchOptions`. В итоге код сетевых запросов выглядит естественно на Nim и при этом компилируется в стандартные вызовы JavaScript `fetch()`.

**Ключевые понятия перед использованием модуля:**

- Все сетевые вызовы **асинхронны** — они возвращают `Future[Response]` и должны быть `await`-нуты внутри процедуры с `{.async.}`.
- Модуль работает в браузерах и в Node.js (с полифилом `node-fetch` или Node.js 18+).
- Запросы `GET` и `HEAD` молча игнорируют переданное `body` — это соответствует спецификации HTTP и принудительно применяется в `newFetchOptions`.

---

## Экспортируемые типы

---

### `FetchOptions`

```nim
type FetchOptions* = ref object of JsRoot
  keepalive*: bool
  metod* {.importjs: "method".}: cstring
  body*, integrity*, referrer*, mode*, credentials*, cache*, redirect*, referrerPolicy*: cstring
  headers*: Headers
```

**Что это такое**

Ref-объект, содержащий все настраиваемые параметры вызова `fetch()` — непосредственное отражение JavaScript-словаря [`RequestInit`](https://developer.mozilla.org/en-US/docs/Web/API/fetch#options). Как правило, не создаётся вручную — используйте `newFetchOptions` или `unsafeNewFetchOptions`.

**Поля**

| Поле | Тип | Описание |
|------|-----|----------|
| `metod` | `cstring` | HTTP-метод (`"GET"`, `"POST"` и т. д.). Названо `metod` (не `method`), потому что `method` — ключевое слово Nim. Отображается в JavaScript `method`. |
| `body` | `cstring` | Тело запроса. Автоматически становится `nil` для `GET` и `HEAD` при использовании `newFetchOptions`. |
| `mode` | `cstring` | CORS-режим — одно из значений перечисления `FetchModes`. |
| `credentials` | `cstring` | Управление отправкой cookies и заголовков аутентификации — одно из `FetchCredentials`. |
| `cache` | `cstring` | Поведение кэша — одно из `FetchCaches`. |
| `redirect` | `cstring` | Обработка перенаправлений — одно из `FetchRedirects`. |
| `referrer` | `cstring` | URL реферера или `"client"` для использования значения по умолчанию. |
| `referrerPolicy` | `cstring` | Политика реферера — одно из `FetchReferrerPolicies`. |
| `integrity` | `cstring` | Хеш целостности подресурса (SRI), или `""` для отключения проверки. |
| `keepalive` | `bool` | Если `true`, запрос продолжается даже после закрытия страницы (полезно для аналитических маяков). |
| `headers` | `Headers` | Заголовки запроса из `std/jsheaders`. |

---

### `Response`

```nim
type Response* = ref object of JsRoot
  bodyUsed*, ok*, redirected*: bool
  typ* {.importjs: "type".}: cstring
  url*, statusText*: cstring
  status*: cint
  headers*: Headers
  body*: cstring
```

**Что это такое**

Представляет ответ на вызов `fetch()`, оборачивая браузерный объект [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response). Поля на практике только для чтения — их изменение не влияет на базовый JavaScript-объект.

**Поля**

| Поле | Тип | Описание |
|------|-----|----------|
| `ok` | `bool` | `true`, если `status` находится в диапазоне 200–299. Первое поле, которое нужно проверить. |
| `status` | `cint` | HTTP-код статуса (например, `200`, `404`, `500`). |
| `statusText` | `cstring` | Текстовое описание статуса (например, `"OK"`, `"Not Found"`). |
| `url` | `cstring` | Конечный URL после всех перенаправлений. |
| `redirected` | `bool` | `true`, если ответ является результатом перенаправления. |
| `headers` | `Headers` | Заголовки ответа. |
| `body` | `cstring` | Необработанное тело как `cstring`. Для структурированного доступа используйте `.text()`, `.json()` или `.formData()`. |
| `bodyUsed` | `bool` | `true`, если тело уже было прочитано (потоки тела можно потребить только один раз). |
| `typ` | `cstring` | Тип ответа: `"basic"`, `"cors"`, `"opaque"` и т. д. Названо `typ`, потому что `type` — ключевое слово Nim. |

---

### `Request`

```nim
type Request* = ref object of JsRoot
  bodyUsed*, ok*, redirected*: bool
  typ* {.importjs: "type".}: cstring
  url*, statusText*: cstring
  status*: cint
  headers*: Headers
  body*: cstring
```

**Что это такое**

Представляет объект HTTP-запроса, оборачивая браузерный [`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request). Имеет такой же набор полей, что и `Response`. Используется, когда нужно построить и проверить запрос независимо от его отправки — например, чтобы клонировать или проверить заголовки перед диспетчеризацией.

---

### `FetchModes`

```nim
type FetchModes* = enum
  fmCors = "cors"
  fmNoCors = "no-cors"
  fmSameOrigin = "same-origin"
```

Управляет обработкой браузером межсайтовых запросов:

| Значение | Смысл |
|----------|-------|
| `fmCors` | Стандартный CORS — сервер должен включать заголовки `Access-Control-Allow-*`. По умолчанию для большинства запросов. |
| `fmNoCors` | Ограничивает запрос безопасным подмножеством методов и заголовков. Ответ будет «непрозрачным» — нельзя прочитать его тело или статус. Полезно для запросов к сторонним серверам в стиле «отправил и забыл». |
| `fmSameOrigin` | Завершается ошибкой, если запрос пересекает границу источника. Используйте, когда намеренно хотите гарантировать же-источниковость. |

---

### `FetchCredentials`

```nim
type FetchCredentials* = enum
  fcInclude = "include"
  fcSameOrigin = "same-origin"
  fcOmit = "omit"
```

Управляет отправкой cookies, HTTP-аутентификации и TLS-клиентских сертификатов:

| Значение | Смысл |
|----------|-------|
| `fcInclude` | Всегда отправлять credentials, включая межсайтовые запросы. |
| `fcSameOrigin` | Отправлять только для же-источниковых запросов. По умолчанию. |
| `fcOmit` | Никогда не отправлять credentials. |

---

### `FetchCaches`

```nim
type FetchCaches* = enum
  fchDefault = "default"
  fchNoStore = "no-store"
  fchReload = "reload"
  fchNoCache = "no-cache"
  fchForceCache = "force-cache"
```

Управляет взаимодействием с HTTP-кэшем браузера:

| Значение | Смысл |
|----------|-------|
| `fchDefault` | Стандартное поведение кэша: использовать свежий кэшированный ответ, ревалидировать устаревший. |
| `fchNoStore` | Никогда не читать из кэша и не писать в него. Всегда идёт в сеть. |
| `fchReload` | Всегда идёт в сеть, но сохраняет результат в кэш. |
| `fchNoCache` | Всегда ревалидирует на сервере, даже если кэшированный ответ свежий. |
| `fchForceCache` | Использовать кэшированный ответ, даже устаревший; идёт в сеть только при отсутствии записи в кэше. |

---

### `FetchRedirects`

```nim
type FetchRedirects* = enum
  frFollow = "follow"
  frError = "error"
  frManual = "manual"
```

Управляет поведением при получении от сервера ответа-перенаправления:

| Значение | Смысл |
|----------|-------|
| `frFollow` | Автоматически следовать перенаправлениям (до лимита браузера). По умолчанию. |
| `frError` | Отклонить `Future` с ошибкой при обнаружении перенаправления. |
| `frManual` | Не следовать перенаправлению; вернуть «непрозрачный» ответ для ручной обработки. |

---

### `FetchReferrerPolicies`

```nim
type FetchReferrerPolicies* = enum
  frpNoReferrer = "no-referrer"
  frpNoReferrerWhenDowngrade = "no-referrer-when-downgrade"
  frpOrigin = "origin"
  frpOriginWhenCrossOrigin = "origin-when-cross-origin"
  frpUnsafeUrl = "unsafe-url"
```

Управляет информацией, отправляемой в HTTP-заголовке `Referer`:

| Значение | Смысл |
|----------|-------|
| `frpNoReferrer` | Заголовок `Referer` не отправляется совсем. |
| `frpNoReferrerWhenDowngrade` | Отправлять полный URL для запросов без понижения протокола; не отправлять при HTTPS→HTTP. По умолчанию. |
| `frpOrigin` | Отправлять только источник (схема + хост + порт), без пути. |
| `frpOriginWhenCrossOrigin` | Полный URL для же-источниковых; только источник для межсайтовых. |
| `frpUnsafeUrl` | Всегда отправлять полный URL. Может раскрыть приватные пути сторонним сайтам — использовать осторожно. |

---

## Экспортируемые функции и процедуры

---

### `newFetchOptions`

```nim
func newFetchOptions*(
  metod = HttpGet;
  body: cstring = nil;
  mode = fmCors;
  credentials = fcSameOrigin;
  cache = fchDefault;
  referrerPolicy = frpNoReferrerWhenDowngrade;
  keepalive = false;
  redirect = frFollow;
  referrer = "client".cstring;
  integrity = "".cstring;
  headers: Headers = newHeaders()
): FetchOptions
```

**Что делает**

**Безопасный и рекомендуемый** конструктор `FetchOptions`. Все параметры имеют разумные значения по умолчанию, соответствующие спецификации Fetch API, — нужно указать только то, что отличается от дефолтов.

Ключевое безопасное поведение: если `metod` — `HttpGet` или `HttpHead`, поле `body` автоматически устанавливается в `nil` вне зависимости от переданного значения. Это соответствует спецификации HTTP (запросы GET и HEAD не должны иметь тела) и предотвращает целый класс незаметных ошибок.

Все перечислимые параметры (`mode`, `credentials`, `cache`, `referrerPolicy`, `redirect`) автоматически конвертируются в строковые значения — вы работаете с типобезопасными Nim-перечислениями и никогда не пишете строки вручную.

**Параметры**

| Параметр | По умолчанию | Описание |
|----------|-------------|----------|
| `metod` | `HttpGet` | HTTP-метод из `std/httpcore`. |
| `body` | `nil` | Тело запроса. Игнорируется для GET/HEAD. |
| `mode` | `fmCors` | CORS-режим. |
| `credentials` | `fcSameOrigin` | Политика credentials. |
| `cache` | `fchDefault` | Политика кэша. |
| `referrerPolicy` | `frpNoReferrerWhenDowngrade` | Политика реферера. |
| `keepalive` | `false` | Продолжать запрос после закрытия страницы. |
| `redirect` | `frFollow` | Политика перенаправлений. |
| `referrer` | `"client"` | Значение реферера. |
| `integrity` | `""` | Хеш целостности подресурса. |
| `headers` | `newHeaders()` | Заголовки запроса. |

**Пример: простой GET (все значения по умолчанию)**

```nim
import std/jsfetch

let opts = newFetchOptions()   # GET, cors, same-origin credentials, default cache
```

**Пример: POST с JSON-телом**

```nim
import std/[jsfetch, jsheaders]
from std/httpcore import HttpPost

var h = newHeaders()
h["Content-Type"] = "application/json"

let opts = newFetchOptions(
  metod = HttpPost,
  body = """{"username": "alice", "password": "secret"}""".cstring,
  headers = h
)
```

**Пример: body автоматически игнорируется для GET**

```nim
from std/httpcore import HttpGet

let opts = newFetchOptions(
  metod = HttpGet,
  body = "это будет nil".cstring   # молча устанавливается в nil
)
assert opts.body == nil
```

---

### `unsafeNewFetchOptions`

```nim
proc unsafeNewFetchOptions*(
  metod, body, mode, credentials, cache, referrerPolicy: cstring;
  keepalive: bool;
  redirect = "follow".cstring;
  referrer = "client".cstring;
  integrity = "".cstring;
  headers: Headers = newHeaders()
): FetchOptions
```

**Что делает**

Альтернативный конструктор `FetchOptions`, принимающий сырые значения `cstring` для всех полей вместо типизированных перечислений. **Валидации нет** — переданные строки попадают напрямую в результирующий объект. Передача неверной строки (например, `"PSOT"` вместо `"POST"`) приведёт к ошибке в рантайме браузера, а не ошибке компиляции Nim.

Существует как аварийный выход для случаев, когда типизированный конструктор слишком ограничителен — например, если Fetch API получил новые значения опций, ещё не отражённые в Nim-перечислениях.

**Когда предпочесть `newFetchOptions`:** почти всегда. Используйте `unsafeNewFetchOptions` только когда нужно передать сырую строку, не охваченную существующими перечислениями, или при интеграции с кодом, который уже работает со строками.

**Пример**

```nim
import std/jsfetch

let opts = unsafeNewFetchOptions(
  metod = "POST".cstring,
  body = """{"key": "value"}""".cstring,
  mode = "no-cors".cstring,
  credentials = "omit".cstring,
  cache = "no-cache".cstring,
  referrerPolicy = "no-referrer".cstring,
  keepalive = false
)
assert opts.metod == "POST".cstring
assert opts.mode == "no-cors".cstring
```

---

### `newResponse`

```nim
func newResponse*(body: cstring | FormData): Response
```

**Что делает**

Создаёт новый объект `Response` с заданным телом, непосредственно вызывая JavaScript `new Response(body)`. **Сетевых запросов не выполняет** — создаёт синтетический ответ в памяти, идентично тому, как работает `new Response(...)` в JavaScript.

Основной сценарий — сервис-воркеры и тестирование: когда нужно сформировать ответ из кэша или обработчика перехвата без обращения к сети.

**Параметры**

| Параметр | Тип | Описание |
|----------|-----|----------|
| `body` | `cstring` или `FormData` | Содержимое тела синтетического ответа. |

**Пример**

```nim
import std/jsfetch

# Синтетический текстовый ответ:
let r = newResponse("-. .. --".cstring)

# Синтетический ответ из FormData:
import std/jsformdata
var fd = newFormData()
fd.append("field", "value")
let r2 = newResponse(fd)
```

---

### `newRequest` (только URL)

```nim
func newRequest*(url: cstring): Request
```

**Что делает**

Создаёт объект `Request` для заданного URL без дополнительных опций, вызывая `new Request(url)` в JavaScript. Ничего не отправляет в сеть.

Полезно, когда нужно передать объект `Request` непосредственно в `fetch()` или проверить/клонировать его перед диспетчеризацией.

**Пример**

```nim
import std/jsfetch

let req = newRequest("https://api.example.com/data".cstring)
```

---

### `newRequest` (URL + опции)

```nim
func newRequest*(url: cstring; fetchOptions: FetchOptions): Request
```

**Что делает**

Создаёт объект `Request`, объединяющий URL и конфигурацию `FetchOptions`, вызывая `new Request(url, fetchOptions)`. По эффекту эквивалентно `fetch(url, fetchOptions)` — используйте этот вариант, когда нужен переиспользуемый самодостаточный объект запроса (например, для клонирования и отправки несколько раз).

**Пример**

```nim
import std/jsfetch
from std/httpcore import HttpPost

let opts = newFetchOptions(metod = HttpPost, body = "данные".cstring)
let req = newRequest("https://api.example.com/submit".cstring, opts)
```

---

### `clone`

```nim
func clone*(self: Response | Request): Response
```

**Что делает**

Создаёт идентичную копию объекта `Response` или `Request`. Это необходимо, потому что потоки тела можно прочитать **только один раз** — после вызова `.text()`, `.json()` или `.formData()` на `Response` его тело потребляется и не может быть прочитано снова. Клонирование до чтения позволяет сохранить нетронутую копию.

Обратите внимание: тип возврата всегда `Response` вне зависимости от входного типа — так работает базовый JavaScript API.

**Пример**

```nim
import std/[jsfetch, asyncjs]

proc handleResponse(resp: Response) {.async.} =
  let copy = resp.clone()           # сохраняем оригинал
  let text = await resp.text()      # потребляет resp.body
  # resp.bodyUsed теперь true — но copy всё ещё нетронута
  let text2 = await copy.text()     # читаем из клона
```

---

### `text`

```nim
proc text*(self: Response): Future[cstring]
```

**Что делает**

Читает тело ответа как обычный текст в виде `cstring`. Это асинхронная операция — тело потоком приходит из сети, поэтому чтение возвращает `Future`. Оборачивает `Response.prototype.text()`.

**Важно:** тело можно прочитать только один раз. После `await resp.text()` поле `resp.bodyUsed` становится `true`, а дальнейшие попытки чтения завершатся ошибкой. Используйте `clone()`, если нужно прочитать тело несколько раз.

**Пример**

```nim
import std/[jsfetch, asyncjs]

proc fetchText() {.async.} =
  let resp = await fetch("https://example.com/data.txt".cstring)
  if resp.ok:
    let content = await resp.text()
    echo $content
```

---

### `json`

```nim
proc json*(self: Response): Future[JsObject]
```

**Что делает**

Читает тело ответа и разбирает его как JSON, возвращая результат в виде `JsObject`. Оборачивает `Response.prototype.json()`. Разбор выполняет движок JavaScript — если тело не является валидным JSON, возвращаемый `Future` будет отклонён с `SyntaxError`.

`JsObject` из `std/jsffi` — слабо типизированный JavaScript-объект. Его свойства доступны через индексирование `[]` или можно привести к типизированному Nim-объекту через `to()`.

**Пример**

```nim
import std/[jsfetch, asyncjs, jsffi]

proc fetchJson() {.async.} =
  let resp = await fetch("https://api.github.com/users/nim-lang".cstring)
  if resp.ok:
    let data = await resp.json()
    echo $data["login"]   # обращение к свойству JsObject
```

---

### `formData`

```nim
proc formData*(self: Response): Future[FormData]
```

**Что делает**

Читает тело ответа и разбирает его как `multipart/form-data` или `application/x-www-form-urlencoded`, возвращая объект `FormData`. Оборачивает `Response.prototype.formData()`. Полезно, когда серверный эндпойнт возвращает данные в форматке вместо JSON.

**Пример**

```nim
import std/[jsfetch, asyncjs, jsformdata]

proc fetchForm() {.async.} =
  let resp = await fetch("https://example.com/form-endpoint".cstring)
  if resp.ok:
    let fd = await resp.formData()
    echo $fd.get("field-name")
```

---

### `fetch` (простой GET)

```nim
proc fetch*(url: cstring | Request): Future[Response]
```

**Что делает**

Отправляет HTTP `GET` запрос на `url` со всеми опциями по умолчанию и возвращает `Future[Response]`. Самый простой способ выполнить сетевой запрос. Оборачивает JavaScript `fetch(url)`.

`Future` разрешается в `Response`, когда приходят заголовки ответа (не после загрузки всего тела). `Future` отклоняется только при сетевых ошибках (нет соединения, DNS-ошибка) — HTTP-коды ошибок вроде 404 и 500 всё равно успешно разрешаются; проверяйте `response.ok` или `response.status`.

**Пример**

```nim
import std/[jsfetch, asyncjs]

proc loadData() {.async.} =
  let resp = await fetch("https://httpbin.org/get".cstring)
  assert resp.ok
  assert resp.status == 200.cint
  let body = await resp.text()
  echo $body
```

---

### `fetch` (с опциями)

```nim
proc fetch*(url: cstring | Request; options: FetchOptions): Future[Response]
```

**Что делает**

Отправляет HTTP-запрос на `url` с конфигурацией из `options`. Это полнофункциональный вариант — используйте его для всего, кроме простого GET. Оборачивает JavaScript `fetch(url, options)`.

**Пример: POST с JSON**

```nim
import std/[jsfetch, jsheaders, asyncjs, jsffi]
from std/httpcore import HttpPost

proc postJson() {.async.} =
  var h = newHeaders()
  h["Content-Type"] = "application/json"

  let opts = newFetchOptions(
    metod = HttpPost,
    body = """{"name": "nim"}""".cstring,
    headers = h
  )
  let resp = await fetch("https://httpbin.org/post".cstring, opts)
  assert resp.ok
  let data = await resp.json()
  echo $data
```

**Пример: цепочка через `.then()` (Promise-стиль)**

```nim
import std/[jsfetch, asyncjs, jsconsole, jsffi]
from std/sugar import `=>`

proc example() {.async.} =
  await fetch("https://api.github.com/users/torvalds".cstring)
    .then((response: Response) => response.json())
    .then((json: JsObject) => console.log(json))
    .catch((err: Error) => console.log("Запрос не удался", err))

discard example()
```

---

### `toCstring`

```nim
func toCstring*(self: Request | Response | FetchOptions): cstring
```

**Что делает**

Сериализует любой из трёх основных типов в JSON-`cstring`, вызывая JavaScript `JSON.stringify()` на базовом объекте. Полезно для отладки, логирования или передачи конфигурации в виде строки другому JavaScript interop-коду.

**Пример**

```nim
import std/jsfetch

let opts = newFetchOptions()
echo $opts.toCstring()
# {"method":"GET","mode":"cors","credentials":"same-origin",...}
```

---

### `$`

```nim
func `$`*(self: Request | Response | FetchOptions): string
```

**Что делает**

Оператор `$` (stringify) Nim для типов `Request`, `Response` и `FetchOptions`. Возвращает Nim-строку с JSON-представлением объекта. Реализован как `$toCstring(self)` — тонкая обёртка над `toCstring`.

**Пример**

```nim
import std/jsfetch

let opts = newFetchOptions()
echo $opts    # выводит JSON-представление
```

---

## Полные примеры использования

### Минимальный GET-запрос

```nim
import std/[jsfetch, asyncjs]

proc getData() {.async.} =
  let resp = await fetch("https://httpbin.org/get".cstring)
  if resp.ok:
    let text = await resp.text()
    echo $text
  else:
    echo "Ошибка: ", $resp.status

discard getData()
```

### POST с JSON-телом и заголовками

```nim
import std/[jsfetch, jsheaders, asyncjs, jsffi]
from std/httpcore import HttpPost

proc createUser() {.async.} =
  var headers = newHeaders()
  headers["Content-Type"] = "application/json"
  headers["Authorization"] = "Bearer my-token"

  let opts = newFetchOptions(
    metod = HttpPost,
    body = """{"name": "Алиса", "email": "alice@example.com"}""".cstring,
    headers = headers
  )

  let resp = await fetch("https://api.example.com/users".cstring, opts)
  case resp.status
  of 201.cint:
    let user = await resp.json()
    echo "Создан: ", $user["id"]
  of 400.cint:
    echo "Неверный запрос: ", $resp.statusText
  else:
    echo "Неожиданный статус: ", $resp.status

discard createUser()
```

### Двойное чтение тела через `clone`

```nim
import std/[jsfetch, asyncjs]

proc inspectResponse() {.async.} =
  let resp = await fetch("https://example.com/data".cstring)
  let copy = resp.clone()

  let raw = await resp.text()
  echo "Длина тела: ", raw.len

  # resp уже потреблён, но copy всё ещё пригоден:
  let again = await copy.text()
  echo "Клон: ", $again

discard inspectResponse()
```

---

## Сводная таблица

| Символ | Вид | Назначение |
|--------|-----|------------|
| `FetchOptions` | тип | Конфигурация вызова `fetch()` |
| `Response` | тип | Результат вызова `fetch()` |
| `Request` | тип | Самодостаточный объект HTTP-запроса |
| `FetchModes` | перечисление | Опции CORS-режима (`cors`, `no-cors`, `same-origin`) |
| `FetchCredentials` | перечисление | Политика credentials (`include`, `same-origin`, `omit`) |
| `FetchCaches` | перечисление | Политика кэша (5 вариантов) |
| `FetchRedirects` | перечисление | Политика перенаправлений (`follow`, `error`, `manual`) |
| `FetchReferrerPolicies` | перечисление | Политика реферера (5 вариантов) |
| `newFetchOptions` | func | **Безопасный** конструктор опций с типизированными перечислениями |
| `unsafeNewFetchOptions` | proc | **Небезопасный** конструктор опций со строками |
| `newResponse` | func | Создать синтетический `Response` (без сетевого запроса) |
| `newRequest` (×2) | func | Создать объект `Request`, опционально с опциями |
| `clone` | func | Скопировать `Response`/`Request` перед потреблением тела |
| `text` | proc | Прочитать тело как `Future[cstring]` |
| `json` | proc | Прочитать тело как `Future[JsObject]` |
| `formData` | proc | Прочитать тело как `Future[FormData]` |
| `fetch` (×2) | proc | Отправить HTTP-запрос; возвращает `Future[Response]` |
| `toCstring` | func | JSON-сериализация в `cstring` |
| `$` | func | JSON-сериализация в `string` |
