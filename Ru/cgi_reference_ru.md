# cgi — Справочник по модулю

> **Стандартная библиотека Nim** | `import std/cgi`

## Что такое CGI?

**CGI (Common Gateway Interface)** — это оригинальный протокол, позволяющий веб-серверу запускать программы и возвращать динамический контент. Когда браузер отправляет форму или переходит по URL, веб-сервер (Apache, Nginx и др.) запускает ваш исполняемый файл и общается с ним через **переменные окружения** и **стандартные потоки ввода-вывода**:

- Веб-сервер устанавливает переменные окружения вроде `REQUEST_METHOD`, `QUERY_STRING`, `HTTP_HOST` и другие перед запуском вашей программы.
- Для POST-запросов тело формы передаётся в `stdin` вашей программы.
- Ваша программа пишет HTTP-ответ (начиная с заголовков) в `stdout`.

Этот модуль предоставляет всё необходимое для написания CGI-скриптов на Nim: чтение и декодирование данных форм, доступ ко всем стандартным переменным CGI-окружения, работа с cookie, безопасное кодирование вывода и отладка.

Модуль также реэкспортирует `encodeUrl` и `decodeUrl` из `std/uri`, поэтому URL-кодирование доступно без дополнительного импорта.

---

## Содержание

1. [Типы](#типы)
   - [CgiError](#cgierror)
   - [RequestMethod](#requestmethod)
2. [Ввод — чтение данных формы](#ввод--чтение-данных-формы)
   - [decodeData (из строки)](#decodedata-из-строки)
   - [decodeData (из окружения)](#decodedata-из-окружения)
   - [readData (из окружения)](#readdata-из-окружения)
   - [readData (из строки)](#readdata-из-строки)
   - [validateData](#validatedata)
3. [Вывод — отправка ответа](#вывод--отправка-ответа)
   - [writeContentType](#writecontenttype)
   - [writeErrorMessage](#writeerrormessage)
   - [setStackTraceStdout](#setstacktracestdout)
   - [xmlEncode](#xmlencode)
4. [Cookie](#cookie)
   - [setCookie](#setcookie)
   - [getCookie](#getcookie)
   - [existsCookie](#existscookie)
5. [Переменные CGI-окружения](#переменные-cgi-окружения)
6. [Обработка ошибок](#обработка-ошибок)
   - [cgiError](#cgierror-1)
7. [Тестирование и отладка](#тестирование-и-отладка)
   - [setTestData](#settestdata)
8. [Реэкспорт из std/uri](#реэкспорт-из-stduri)
9. [Полный пример](#полный-пример)
10. [Краткая сводка](#краткая-сводка)

---

## Типы

### `CgiError`

```nim
type CgiError* = object of IOError
```

Тип исключения, выбрасываемого при ошибках на уровне протокола CGI — например, клиент использует HTTP-метод, который ваш скрипт не разрешает, или невозможно прочитать `stdin`. Наследует `IOError`, поэтому при необходимости его можно перехватить через `except IOError`.

---

### `RequestMethod`

```nim
type RequestMethod* = enum
  methodNone,   ## переменная REQUEST_METHOD не установлена
  methodPost,   ## клиент использовал POST
  methodGet     ## клиент использовал GET
```

Перечисление трёх возможных состояний HTTP-метода запроса с точки зрения CGI-окружения. Используется как фильтр в `decodeData` и `readData`, чтобы ограничить HTTP-методы, которые ваш скрипт принимает.

- `methodNone` — скрипт запущен вне веб-сервера (например, из командной строки). Удобно для тестирования.
- `methodPost` — браузер отправил форму с `method="POST"`.
- `methodGet` — браузер отправил форму с `method="GET"` или перешёл по URL со строкой запроса.

---

## Ввод — чтение данных формы

Это самая важная часть любого CGI-скрипта: получение данных, отправленных пользователем.

### `decodeData` (из строки)

```nim
iterator decodeData*(data: string): tuple[key, value: string]
```

Декодирует URL-кодированную строку запроса (например, `name=Alice&age=30`) и возвращает пары `(ключ, значение)` по одной. Строка — это сырые закодированные данные: как правило, содержимое `QUERY_STRING` или тело POST-запроса, прочитанное вручную.

URL-кодирование обрабатывается автоматически: `%20` становится пробелом, `+` — пробелом, `%2F` — `/` и т.д.

```nim
let qs = "city=New+York&country=US&pop=8336817"
for key, val in decodeData(qs):
  echo key, " = ", val
# city = New York
# country = US
# pop = 8336817
```

---

### `decodeData` (из окружения)

```nim
iterator decodeData*(allowedMethods: set[RequestMethod] =
    {methodNone, methodPost, methodGet}): tuple[key, value: string]
```

Наиболее часто используемый итератор ввода. Автоматически читает из правильного места — `stdin` для POST-запросов, `QUERY_STRING` для GET-запросов — декодирует данные и возвращает пары `(ключ, значение)`.

Параметр `allowedMethods` работает как шлюз безопасности: если клиент использует HTTP-метод, не входящий в множество, немедленно выбрасывается `CgiError`. Именно так можно потребовать, чтобы форма входа принималась только через POST.

```nim
# Принимать только POST; отклонять GET и прямые вызовы
for key, val in decodeData({methodPost}):
  echo key, " => ", val
```

```nim
# По умолчанию — принимать всё (удобно при разработке)
for key, val in decodeData():
  echo key, " => ", val
```

---

### `readData` (из окружения)

```nim
proc readData*(allowedMethods: set[RequestMethod] =
               {methodNone, methodPost, methodGet}): StringTableRef
```

Аналог `decodeData` (версия из окружения), но собирает **все** пары ключ-значение в `StringTableRef` (регистронезависимую хеш-таблицу из `std/strtabs`) и возвращает её. Обычно это удобнее итератора, когда впоследствии нужно обращаться к отдельным полям по имени.

Если один и тот же ключ встречается несколько раз в строке запроса, сохраняется только **последнее** значение (запись в таблице перезаписывается).

```nim
let data = readData()
echo data["username"]
echo data["email"]
```

```nim
# Ограничить только POST
let data = readData({methodPost})
echo "Пароль: ", data["password"]
```

---

### `readData` (из строки)

```nim
proc readData*(data: string): StringTableRef
```

Разбирает сырую URL-кодированную строку в `StringTableRef`. Удобно, когда сырые данные уже прочитаны самостоятельно, или при написании юнит-тестов без веб-сервера.

```nim
let table = readData("lang=nim&version=2")
echo table["lang"]     # nim
echo table["version"]  # 2
```

---

### `validateData`

```nim
proc validateData*(data: StringTableRef, validKeys: varargs[string])
```

Проверяет, что каждый ключ в `data` входит в список `validKeys`. Если обнаружен неожиданный ключ, выбрасывается `CgiError` с сообщением, указывающим имя неизвестной переменной.

Это простая, но эффективная защита от подделки форм: если пользователь вручную составил запрос с именами полей, которые скрипт не ожидает, валидация сразу это поймает.

```nim
let data = readData()
# Выбросит CgiError, если data содержит ключи, отличные от "name" или "email"
validateData(data, "name", "email")

echo data["name"]
echo data["email"]
```

---

## Вывод — отправка ответа

CGI-ответ должен начинаться с HTTP-заголовков, за которыми следует пустая строка, а затем тело. Если забыть заголовки, браузер получит ошибку «500 Internal Server Error».

### `writeContentType`

```nim
proc writeContentType*()
```

Записывает `Content-type: text/html\n\n` в `stdout`. Это минимально необходимый HTTP-заголовок для HTML-ответа. Вызывайте его **до** того, как начнёте отправлять HTML-контент. Без него веб-сервер не будет знать, как интерпретировать ваш вывод.

```nim
writeContentType()
write(stdout, "<html><body><h1>Привет из Nim CGI!</h1></body></html>")
```

---

### `writeErrorMessage`

```nim
proc writeErrorMessage*(data: string)
```

Пытается «сбросить» состояние HTML-рендеринга браузера, закрывая незакрытые теги, а затем записывает `data` внутри элемента `<plaintext>`. Это делает сырой текст (в том числе стек-трейсы) читаемым в браузере без HTML-экранирования. Используется внутри `setStackTraceStdout`, но может вызываться и напрямую для отображения ошибок.

```nim
try:
  discard  # ... ваша логика ...
except CgiError as e:
  writeErrorMessage("Ошибка CGI: " & e.msg)
```

---

### `setStackTraceStdout`

```nim
proc setStackTraceStdout*()
```

Перенаправляет внутренний вывод ошибок и стек-трейсов Nim из журнала сервера в `stdout` (браузер). Это значительно упрощает отладку: вместо того чтобы искать записи в серверных логах, вы видите стек-трейс Nim прямо в окне браузера.

**Вызывайте как можно раньше** — в идеале самой первой строкой CGI-скрипта — чтобы даже ошибки, возникающие до вызова `writeContentType`, были перехвачены и отображены.

```nim
setStackTraceStdout()
writeContentType()
# ... остальная часть скрипта ...
```

---

### `xmlEncode`

```nim
proc xmlEncode*(s: string): string
```

Экранирует строку так, чтобы её было безопасно вставлять внутрь HTML или XML. Выполняются следующие замены:

| Символ | Кодируется как |
|---|---|
| `&` | `&amp;` |
| `<` | `&lt;` |
| `>` | `&gt;` |
| `"` | `&quot;` |

Все остальные символы передаются без изменений.

**Всегда используйте `xmlEncode` при вставке пользовательских данных в HTML-вывод.** Пренебрежение этим открывает уязвимость XSS (межсайтовый скриптинг): пользователь может отправить `<script>alert(1)</script>` в качестве своего «имени», и этот код выполнится в браузерах других пользователей.

```nim
let userInput = "<script>alert('XSS')</script>"
write(stdout, "<p>" & xmlEncode(userInput) & "</p>")
# Выводит: <p>&lt;script&gt;alert('XSS')&lt;/script&gt;</p>
# Браузер отображает это как безобидный текст, а не код.
```

---

## Cookie

Cookie устанавливаются отправкой HTTP-заголовка до тела ответа и читаются из переменной окружения `HTTP_COOKIE`, которую предоставляет веб-сервер.

### `setCookie`

```nim
proc setCookie*(name, value: string)
```

Отправляет браузеру HTTP-заголовок `Set-Cookie`, предписывая ему сохранить cookie. Заголовок записывается в `stdout` и **должен быть отправлен до `writeContentType`** — то есть до пустой строки, завершающей блок заголовков.

```nim
setCookie("sessionid", "abc123xyz")
setCookie("lang", "ru")
writeContentType()
# ... HTML тело ...
```

> **Примечание:** Это базовая установка cookie — задаётся только `имя=значение` без атрибутов `Path`, `Expires`, `HttpOnly` или `Secure`. Для продакшн-использования задайте эти атрибуты вручную или используйте `std/cookies`.

---

### `getCookie`

```nim
proc getCookie*(name: string): string
```

Возвращает значение cookie с именем `name` из текущего запроса. Если такого cookie нет, возвращает пустую строку `""`. Данные cookie лениво разбираются из `HTTP_COOKIE` при первом вызове и кэшируются в локальной для потока таблице для последующих вызовов.

```nim
let session = getCookie("sessionid")
if session == "":
  echo "Сессия отсутствует — пожалуйста, войдите."
else:
  echo "С возвращением! Сессия: ", session
```

---

### `existsCookie`

```nim
proc existsCookie*(name: string): bool
```

Возвращает `true`, если cookie с данным именем присутствует в текущем запросе, иначе `false`. Используйте это, когда нужно различать «cookie не установлен» и «cookie установлен в пустую строку» — `getCookie` такого различия не делает.

```nim
if existsCookie("preferences"):
  let prefs = getCookie("preferences")
  applyPreferences(prefs)
else:
  applyDefaults()
```

---

## Переменные CGI-окружения

Веб-сервер заполняет стандартный набор переменных окружения перед запуском CGI-скрипта. Модуль предоставляет отдельную функцию-геттер для каждой из них. Все функции возвращают пустую строку `""`, если соответствующая переменная не установлена.

| Процедура | Переменная окружения | Типичное содержимое |
|---|---|---|
| `getContentLength()` | `CONTENT_LENGTH` | Размер тела POST в байтах, например `"1234"` |
| `getContentType()` | `CONTENT_TYPE` | MIME-тип тела POST, например `"application/x-www-form-urlencoded"` |
| `getDocumentRoot()` | `DOCUMENT_ROOT` | Абсолютный путь к корню документов сервера |
| `getGatewayInterface()` | `GATEWAY_INTERFACE` | Версия CGI, например `"CGI/1.1"` |
| `getHttpAccept()` | `HTTP_ACCEPT` | MIME-типы, которые браузер принимает |
| `getHttpAcceptCharset()` | `HTTP_ACCEPT_CHARSET` | Предпочитаемые браузером кодировки |
| `getHttpAcceptEncoding()` | `HTTP_ACCEPT_ENCODING` | Поддерживаемые браузером виды сжатия, например `"gzip, deflate"` |
| `getHttpAcceptLanguage()` | `HTTP_ACCEPT_LANGUAGE` | Предпочитаемые языки браузера, например `"ru-RU,ru;q=0.9"` |
| `getHttpConnection()` | `HTTP_CONNECTION` | Управление соединением, например `"keep-alive"` |
| `getHttpCookie()` | `HTTP_COOKIE` | Все cookie для домена в виде сырой строки |
| `getHttpHost()` | `HTTP_HOST` | Имя хоста из запроса, например `"example.com"` |
| `getHttpReferer()` | `HTTP_REFERER` | URL страницы, с которой пришёл пользователь |
| `getHttpUserAgent()` | `HTTP_USER_AGENT` | Строка идентификации браузера |
| `getPathInfo()` | `PATH_INFO` | Дополнительный путь после имени скрипта |
| `getPathTranslated()` | `PATH_TRANSLATED` | `PATH_INFO`, переведённый в путь файловой системы |
| `getQueryString()` | `QUERY_STRING` | Всё после `?` в URL |
| `getRemoteAddr()` | `REMOTE_ADDR` | IP-адрес клиента |
| `getRemoteHost()` | `REMOTE_HOST` | Имя хоста клиента (если включён обратный DNS) |
| `getRemoteIdent()` | `REMOTE_IDENT` | Идентификация удалённого пользователя (RFC 931, редко используется) |
| `getRemotePort()` | `REMOTE_PORT` | Номер TCP-порта клиента |
| `getRemoteUser()` | `REMOTE_USER` | Имя аутентифицированного пользователя (при настроенной HTTP-аутентификации) |
| `getRequestMethod()` | `REQUEST_METHOD` | `"GET"`, `"POST"` и т.д. |
| `getRequestURI()` | `REQUEST_URI` | Полный URI запроса, включая строку запроса |
| `getScriptFilename()` | `SCRIPT_FILENAME` | Абсолютный путь к CGI-скрипту на диске |
| `getScriptName()` | `SCRIPT_NAME` | URL-путь к скрипту, например `"/cgi-bin/app"` |
| `getServerAddr()` | `SERVER_ADDR` | IP-адрес сервера |
| `getServerAdmin()` | `SERVER_ADMIN` | Email администратора сервера |
| `getServerName()` | `SERVER_NAME` | Имя хоста или IP сервера, видимое клиенту |
| `getServerPort()` | `SERVER_PORT` | Порт, который слушает сервер, например `"80"` |
| `getServerProtocol()` | `SERVER_PROTOCOL` | Версия HTTP, например `"HTTP/1.1"` |
| `getServerSignature()` | `SERVER_SIGNATURE` | Баннер версии сервера (если включён) |
| `getServerSoftware()` | `SERVER_SOFTWARE` | Название и версия программного обеспечения сервера |

```nim
# Вывод диагностической информации
echo "Метод: ", getRequestMethod()
echo "IP клиента: ", getRemoteAddr()
echo "Хост: ", getHttpHost()
echo "User-Agent: ", getHttpUserAgent()
```

---

## Обработка ошибок

### `cgiError`

```nim
proc cgiError*(msg: string) {.noreturn.}
```

Выбрасывает исключение `CgiError` с данным сообщением. Помечен `{.noreturn.}` — никогда не возвращается в нормальном режиме. Используйте для сигнализирования об ошибках уровня протокола в собственном CGI-коде валидации.

```nim
let method = getRequestMethod()
if method != "POST":
  cgiError("Этот эндпоинт принимает только POST-запросы, получено: " & method)
```

---

## Тестирование и отладка

### `setTestData`

```nim
proc setTestData*(keysvalues: varargs[string])
```

Имитирует GET-запрос, заполняя переменные окружения `REQUEST_METHOD` и `QUERY_STRING` переданными парами ключ-значение. Пары передаются как чередующиеся строки: `ключ1, значение1, ключ2, значение2, ...`.

Это позволяет тестировать CGI-логику прямо из командной строки или юнит-тестов, без веб-сервера. Таким способом можно симулировать только GET-запросы.

```nim
when defined(debug):
  setTestData("username", "alice", "lang", "nim", "version", "2")

let data = readData()
echo data["username"]  # alice
echo data["lang"]      # nim
```

После этого вызова `readData()` и `decodeData()` ведут себя в точности так, как если бы браузер отправил `?username=alice&lang=nim&version=2`.

---

## Реэкспорт из std/uri

Модуль реэкспортирует две функции из `std/uri`, чтобы их можно было использовать без дополнительного импорта:

**`encodeUrl(s: string): string`** — процентно-кодирует строку для использования в URL. Пробелы становятся `%20`, специальные символы вроде `&`, `=`, `#` экранируются.

```nim
let safe = encodeUrl("привет мир & ещё")
# %D0%BF%D1%80%D0%B8%D0%B2%D0%B5%D1%82%20%D0%BC%D0%B8%D1%80%20%26%20%D0%B5%D1%89%D1%91
```

**`decodeUrl(s: string): string`** — обратная операция к `encodeUrl`. Преобразует `%XX`-последовательности обратно в оригинальные символы.

```nim
let original = decodeUrl("hello%20world%20%26%20more")
# hello world & more
```

---

## Полный пример

Ниже — минимальный, но полный CGI-скрипт, принимающий имя из формы, валидирующий его и отвечающий персонализированным приветствием. Демонстрирует типичный рабочий процесс целиком.

```nim
import std/[strtabs, cgi]

# 1. Перенаправить стек-трейсы в браузер при разработке
setStackTraceStdout()

# 2. Симулировать тестовые данные при компиляции с -d:debug
when defined(debug):
  setTestData("name", "Алиса", "lang", "ru")

# 3. Прочитать отправленные данные формы (POST или GET)
let data = readData()

# 4. Проверить, что присутствуют только ожидаемые поля
validateData(data, "name", "lang")

# 5. Отправить заголовок Content-Type
writeContentType()

# 6. Сгенерировать HTML — всегда применяйте xmlEncode к пользовательским данным
let name = xmlEncode(data["name"])
let lang = xmlEncode(data["lang"])

write(stdout, "<!DOCTYPE html>\n")
write(stdout, "<html><head><title>Приветствие</title></head><body>\n")
write(stdout, "<h1>Привет, " & name & "!</h1>\n")
write(stdout, "<p>Ваш язык: " & lang & "</p>\n")
write(stdout, "<p>Ваш IP-адрес: " & getRemoteAddr() & "</p>\n")
write(stdout, "</body></html>\n")
```

---

## Краткая сводка

| Символ | Описание |
|---|---|
| `CgiError` | Тип исключения для ошибок протокола CGI |
| `RequestMethod` | Перечисление: `methodNone`, `methodPost`, `methodGet` |
| `decodeData(string)` | Итератор: декодировать URL-строку → пары (ключ, значение) |
| `decodeData(set)` | Итератор: читать из окружения → пары (ключ, значение) |
| `readData(set)` | Прочитать все данные формы из окружения → `StringTableRef` |
| `readData(string)` | Разобрать сырую URL-строку → `StringTableRef` |
| `validateData(table, keys...)` | Выбросить `CgiError`, если обнаружены неожиданные ключи |
| `writeContentType()` | Записать заголовок `Content-type: text/html` в stdout |
| `writeErrorMessage(string)` | Безопасно вывести сообщение об ошибке в браузер |
| `setStackTraceStdout()` | Перенаправить стек-трейсы Nim в браузер |
| `xmlEncode(string)` | Экранировать `& < > "` для безопасной вставки в HTML |
| `setCookie(name, value)` | Отправить заголовок `Set-Cookie` |
| `getCookie(name)` | Получить значение cookie по имени (`""` если отсутствует) |
| `existsCookie(name)` | Проверить наличие cookie → `bool` |
| `cgiError(msg)` | Выбросить `CgiError` с сообщением |
| `setTestData(key, val, ...)` | Симулировать GET-запрос для тестирования |
| `encodeUrl(string)` | Процентно-кодировать компонент URL (реэкспорт) |
| `decodeUrl(string)` | Декодировать процентно-кодированную строку (реэкспорт) |
| `getContentLength()` … `getServerSoftware()` | Читать переменные CGI-окружения |
