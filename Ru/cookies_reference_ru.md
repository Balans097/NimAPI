# Справочник модуля `cookies`

> **Модуль:** `std/cookies`  
> **Назначение:** Разбор заголовка `Cookie` в запросах от клиентов и построение заголовка `Set-Cookie` в ответах от серверов.

---

## Общее описание

HTTP-куки — небольшие пары «ключ=значение», которые сервер предписывает браузеру сохранить и отправлять обратно при последующих запросах. Модуль `cookies` покрывает оба направления этого обмена:

- **Разбор** — чтение заголовка `Cookie: a=1; b=2`, который **клиент** (браузер) присылает серверу (`parseCookies`).
- **Построение** — формирование заголовка `Set-Cookie: key=value; Domain=…; Secure; …`, который **сервер** отправляет клиенту (`setCookie`).

Модуль намеренно минималистичен. URL-кодирование и декодирование значений куки он не выполняет — это обязанность вызывающего кода.

---

## Экспортируемые символы

---

### `SameSite`

```nim
type SameSite* {.pure.} = enum
  Default, None, Lax, Strict
```

#### Что это такое

Перечисление, представляющее значение атрибута `SameSite` куки. Этот атрибут управляет тем, при каких межсайтовых запросах браузер вправе отправлять куку. Он является основной защитой от атак типа CSRF (Cross-Site Request Forgery — подделка межсайтовых запросов).

#### Значения и их смысл

| Значение | Действие |
|---|---|
| `Default` | `setCookie` **не добавляет** атрибут `SameSite` вовсе. Браузер применяет собственное умолчание (в современных браузерах обычно `Lax`). |
| `None` | Кука отправляется при любых запросах, включая кросс-сайтовые. **Требует `secure = true`** — иначе `setCookie` выбросит ошибку `doAssert` в рантайме. |
| `Lax` | Кука отправляется при запросах в рамках того же сайта и при переходах верхнего уровня (например, по ссылке), но не при кросс-сайтовых подзапросах (изображения, iframe). Разумный компромисс. |
| `Strict` | Кука отправляется **только** при запросах в рамках того же сайта. Максимальная изоляция; пользователь не будет аутентифицирован при переходе с внешнего сайта. |

#### Примеры

```nim
import std/cookies

# SameSite.Strict: максимальная защита, кука никогда не отправляется кросс-сайтово
echo setCookie("session", "abc123", secure = true,
               httpOnly = true, sameSite = SameSite.Strict)
# → Set-Cookie: session=abc123; Secure; HttpOnly; SameSite=Strict

# SameSite.Lax: хорошее умолчание для большинства веб-приложений
echo setCookie("prefs", "dark-mode", sameSite = SameSite.Lax)
# → Set-Cookie: prefs=dark-mode; SameSite=Lax

# SameSite.None без Secure — программа упадёт с doAssert:
# echo setCookie("track", "x", sameSite = SameSite.None)
```

---

### `parseCookies`

```nim
proc parseCookies*(s: string): StringTableRef
```

#### Что делает

Разбирает значение HTTP-заголовка `Cookie` — компактный формат с разделителем-точкой с запятой, который **клиенты** (браузеры) отправляют серверам — и возвращает нечувствительную к регистру таблицу `StringTable`, отображающую имена куки в их значения.

#### Важное отличие

Процедура предназначена для заголовка **`Cookie`** (клиент → сервер), который выглядит так:

```
Cookie: name1=value1; name2=value2
```

Она **не предназначена** для разбора заголовка `Set-Cookie` (сервер → клиент), имеющего принципиально иной формат с атрибутами `Domain`, `Path`, `Expires` и т. д.

#### Как работает внутри

Парсер обходит строку посимвольно:

1. Пропускает начальные пробелы и символы табуляции перед именем куки.
2. Читает символы до `=` — это ключ.
3. Читает символы до `;` (или конца строки) — это значение.
4. Сохраняет пару в таблице и переходит за разделитель `;`.

URL-декодирования не выполняется. Значения сохраняются ровно так, как они присутствуют в заголовке.

#### Результат нечувствителен к регистру

Возвращаемая `StringTableRef` создаётся с параметром `modeCaseInsensitive`, то есть `cookieJar["Session"]`, `cookieJar["session"]` и `cookieJar["SESSION"]` обращаются к одной и той же записи.

#### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `s` | `string` | Сырое значение заголовка `Cookie` (без префикса `Cookie:` и пробела после него). |

#### Возвращаемое значение

`StringTableRef` (из `std/strtabs`), где ключи — имена куки, значения — содержимое куки.

#### Примеры

```nim
import std/cookies, std/strtabs

# Базовый разбор
let jar = parseCookies("a=1; foo=bar")
assert jar["a"] == "1"
assert jar["foo"] == "bar"
```

```nim
import std/cookies, std/strtabs

# Поиск по ключу нечувствителен к регистру
let jar = parseCookies("Session=xyz; Theme=dark")
assert jar["session"] == "xyz"   # нижний регистр работает
assert jar["THEME"] == "dark"    # верхний регистр — тоже
```

```nim
import std/cookies, std/strtabs

# Типичное серверное использование: извлечение Cookie из HTTP-запроса
# (псевдокод; реальное получение заголовка зависит от HTTP-фреймворка)
let rawCookieHeader = "user_id=42; csrf_token=abc; lang=en"
let cookies = parseCookies(rawCookieHeader)

if "user_id" in cookies:
  echo "Аутентифицированный пользователь: ", cookies["user_id"]
else:
  echo "Куки user_id нет — гостевой доступ"
```

```nim
import std/cookies, std/strtabs

# Корректная обработка пустого заголовка Cookie
let cookies = parseCookies("")
echo cookies.len  # → 0, никакого краша
```

---

### `setCookie` (expires как строка)

```nim
proc setCookie*(key, value: string, domain = "", path = "",
                expires = "", noName = false,
                secure = false, httpOnly = false,
                maxAge = none(int),
                sameSite = SameSite.Default): string
```

#### Что делает

Формирует строку HTTP-заголовка `Set-Cookie`. Сервер включает эту строку в HTTP-ответ дословно, давая браузеру команду сохранить куку с указанными атрибутами.

#### Параметры подробно

| Параметр | Тип | Умолч. | Смысл |
|---|---|---|---|
| `key` | `string` | *(обязателен)* | Имя куки. |
| `value` | `string` | *(обязателен)* | Значение куки. |
| `domain` | `string` | `""` | Ограничивает куку указанным доменом и его поддоменами. Если пропустить, кука действует только для точного хоста, который её выставил. |
| `path` | `string` | `""` | Ограничивает куку URL-адресами, путь которых начинается с этого значения. `"/"` означает весь сайт. |
| `expires` | `string` | `""` | Дата истечения в виде уже отформатированной строки (RFC 1123, например `"Mon, 01 Jan 2030 00:00:00 GMT"`). Если пропустить, кука сессионная и удалится при закрытии браузера. |
| `noName` | `bool` | `false` | При `true` пропускает префикс `Set-Cookie:`. Удобно, если заголовки собираются вручную и нужна только строка атрибутов. |
| `secure` | `bool` | `false` | При `true` браузер отправляет куку только по HTTPS. Настоятельно рекомендуется для любых чувствительных данных. |
| `httpOnly` | `bool` | `false` | При `true` JavaScript на странице не может получить куку через `document.cookie`. Это основная защита от кражи куки через XSS. |
| `maxAge` | `Option[int]` | `none(int)` | Время жизни куки в **секундах** с момента получения. В современных браузерах имеет приоритет над `Expires`. Передайте `some(0)`, чтобы немедленно удалить куку. |
| `sameSite` | `SameSite` | `SameSite.Default` | Управляет отправкой куки при кросс-сайтовых запросах. Подробнее — в разделе `SameSite`. |

#### Замечание о безопасности из исходника модуля

> *«Куки могут быть уязвимы. Рассмотрите использование `secure=true`, `httpOnly=true` и `sameSite=Strict`.»*

Для сессионных куки и токенов аутентификации эта комбинация почти всегда является правильным выбором.

#### Примеры

```nim
import std/cookies

# Минимальная кука — только имя и значение
echo setCookie("lang", "ru")
# → Set-Cookie: lang=ru
```

```nim
import std/cookies

# Хорошо защищённая сессионная кука
echo setCookie("session_id", "s3cr3t",
               path = "/",
               secure = true,
               httpOnly = true,
               sameSite = SameSite.Strict)
# → Set-Cookie: session_id=s3cr3t; Path=/; Secure; HttpOnly; SameSite=Strict
```

```nim
import std/cookies, std/options

# Кука, истекающая через 1 час (3600 секунд)
echo setCookie("token", "abc", maxAge = some(3600))
# → Set-Cookie: token=abc; Max-Age=3600
```

```nim
import std/cookies, std/options

# Удаление куки: Max-Age=0
echo setCookie("session_id", "", maxAge = some(0))
# → Set-Cookie: session_id=; Max-Age=0
```

```nim
import std/cookies

# noName=true: только строка атрибутов без "Set-Cookie: "
let attrs = setCookie("id", "99", path = "/api", noName = true)
echo attrs
# → id=99; Path=/api
```

```nim
import std/cookies

# SameSite=None требует Secure=true — иначе doAssert упадёт в рантайме
# Это упадёт:
# echo setCookie("x", "y", sameSite = SameSite.None)

# Правильно:
echo setCookie("x", "y", secure = true, sameSite = SameSite.None)
# → Set-Cookie: x=y; Secure; SameSite=None
```

---

### `setCookie` (expires как DateTime/Time)

```nim
proc setCookie*(key, value: string, expires: DateTime | Time,
                domain = "", path = "", noName = false,
                secure = false, httpOnly = false,
                maxAge = none(int),
                sameSite = SameSite.Default): string
```

#### Что делает

Перегрузка `setCookie`, которая принимает значение типа `DateTime` или `Time` (из `std/times`) вместо строки с датой, сформатированной вручную. Дата автоматически приводится в формат RFC 1123, требуемый стандартом HTTP (`ddd, dd MMM yyyy HH:mm:ss GMT`), после чего вызывается строковая версия `setCookie`.

Это предпочтительный способ задать дату истечения, поскольку он полностью исключает ошибки форматирования строки.

#### Как работает преобразование

Аргумент `expires` сначала приводится к UTC методом `.utc`, затем форматируется как:

```
Mon, 01 Jan 2030 00:00:00 GMT
```

Получившаяся строка передаётся в первую перегрузку `setCookie` в параметр `expires`.

#### Параметры

Все параметры идентичны строковой версии, кроме `expires`:

| Параметр | Тип | Описание |
|---|---|---|
| `expires` | `DateTime` или `Time` | Момент истечения куки. Автоматически приводится к UTC. |

#### Примеры

```nim
import std/cookies, std/times

# Кука, истекающая в конкретную дату
let expiry = dateTime(2030, mJan, 1, 0, 0, 0, zone = utc())
echo setCookie("promo", "SALE2030", expires = expiry)
# → Set-Cookie: promo=SALE2030; Expires=Tue, 01 Jan 2030 00:00:00 GMT
```

```nim
import std/cookies, std/times

# Кука «запомни меня» на 30 дней
let in30days = now().utc + 30.days
echo setCookie("remember_me", "token_xyz",
               expires = in30days,
               path = "/",
               secure = true,
               httpOnly = true,
               sameSite = SameSite.Strict)
# → Set-Cookie: remember_me=token_xyz; Expires=...; Path=/; Secure; HttpOnly; SameSite=Strict
```

```nim
import std/cookies, std/times

# Использование Time вместо DateTime
let t: Time = getTime() + initDuration(hours = 1)
echo setCookie("flash", "welcome", expires = t)
# → Set-Cookie: flash=welcome; Expires=<через 1 час в GMT>
```

---

## Быстрый справочник: комбинации атрибутов

| Задача | Рекомендуемая комбинация |
|---|---|
| Сессионный токен аутентификации | `secure=true`, `httpOnly=true`, `sameSite=Strict`, `path="/"` |
| Долгосрочный вход («запомни меня») | То же + `expires` или `maxAge` с будущей датой |
| CSRF-токен, читаемый JavaScript | `sameSite=Strict` (без `httpOnly`) |
| Сторонняя / кросс-сайтовая кука | `secure=true`, `sameSite=None` |
| Удалить куку | `maxAge=some(0)` |
| Получить только строку атрибутов | `noName=true` |

---

## Типичные ловушки

**1. `parseCookies` против разбора `Set-Cookie`**  
`parseCookies` понимает только компактный формат `Cookie:`. Не передавайте ей заголовок `Set-Cookie:` — атрибуты (`Domain`, `Path` и т. д.) будут восприняты как значения куки.

**2. `SameSite=None` без `Secure`**  
Передача `sameSite = SameSite.None` без `secure = true` вызывает `doAssert` и завершает программу. Это соответствует требованию браузеров: куки с `SameSite=None` должны передаваться только по HTTPS.

**3. Нет автоматического URL-кодирования**  
Ни `parseCookies`, ни `setCookie` не кодируют и не декодируют значения куки. Если значения могут содержать `=`, `;` или не-ASCII-символы, закодируйте их (например, с помощью `std/uri.encodeUrl`) перед передачей в `setCookie` и декодируйте после разбора.

**4. `Expires` против `Max-Age`**  
`Max-Age` (в секундах, относительное значение) имеет приоритет над `Expires` (абсолютная дата) во всех современных браузерах. Предпочитайте `maxAge` для надёжности; используйте `expires`-строку только для совместимости с очень старыми клиентами.
