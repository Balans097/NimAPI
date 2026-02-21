# smtp.nim — Справочник модуля

> **Установка:** `nimble install smtp`  
> **Стандарты:** RFC 5321 (протокол SMTP), RFC 2822 (формат сообщений)  
> **Поддержка SSL:** компилировать с флагом `-d:ssl`

Модуль решает две самостоятельные, но связанные задачи:

1. **Составление письма** — построение корректного RFC 2822-объекта с заголовками, телом и списками адресатов.
2. **Транспортировка по SMTP** — подключение к почтовому серверу, аутентификация и передача письма.

Для обеих задач есть синхронный (`Smtp`) и асинхронный (`AsyncSmtp`) варианты; их API идентичны по форме.

---

## Типы

### `Email`
Представляет один почтовый адрес с необязательным отображаемым именем.

```nim
# Внутренние поля (доступ через процедуры):
#   address: string   — адрес почтового ящика, напр. "alice@example.com"
#   name:    string   — отображаемое имя,     напр. "Алиса"
```

### `Message`
Полное RFC 2822-сообщение — заголовки плюс тело. Передаётся серверу в виде строки через `sendMail`.

### `Smtp` / `AsyncSmtp`
Дескриптор открытого подключения к SMTP-серверу. `Smtp` — для блокирующего кода, `AsyncSmtp` — внутри `async`-процедур.

### `ReplyError`
Исключение, возникающее при неожиданном коде ответа от сервера. Наследует `IOError`.

### `Port` *(реэкспортируется из `net`)*
Типизированный целочисленный тип для номеров TCP-портов, напр. `Port 465`.

---

## Составление письма

### `createEmail`

```nim
proc createEmail*(address: string, name: string = ""): Email
```

**Что делает.** Создаёт значение типа `Email` из строки адреса и необязательного отображаемого имени. Имя — это то, что почтовый клиент показывает рядом с адресом как «человекочитаемый» ярлык.

**Важно:** ни `address`, ни `name` не должны содержать символов возврата каретки (`\r`) или перевода строки (`\n`). Такие символы открывают путь к атаке «внедрение заголовка» (header injection). При их наличии поднимается `AssertionDefect`.

```nim
let alice = createEmail("alice@example.com", "Алиса Смит")
let bot   = createEmail("no-reply@example.com")   # имя не обязательно

echo alice  # → "Алиса Смит" <alice@example.com>
echo bot    # → no-reply@example.com
```

---

### `createMessage`

```nim
proc createMessage*[T: Email | string](
    mSubject, mBody: string,
    sender: T,
    mTo:      seq[T] = @[],
    mCc:      seq[T] = @[],
    mBcc:     seq[T] = @[],
    mReplyTo: seq[T] = @[],
    otherHeaders: openArray[tuple[name, value: string]] = @[],
): Message
```

**Что делает.** Создаёт структурированный объект `Message`. По аналогии с формой «Написать письмо» вы заполняете тему, тело, адрес отправителя и списки получателей. Результат — не строка; вызовите `$msg`, чтобы получить готовый RFC 2822-текст.

**Параметр типа `T`.** Принимает как обычные строки-адреса, так и объекты `Email` — удобно при наличии или отсутствии отображаемых имён.

**Защита от инъекций.** Тема (`mSubject`) не должна содержать символов новой строки — иначе `AssertionDefect`.

```nim
# Простой вариант — только строки
let msg = createMessage(
    mSubject = "Ежемесячный отчёт",
    mBody    = "Отчёт во вложении.",
    sender   = "reports@example.com",
    mTo      = @["manager@example.com"],
    mCc      = @["archive@example.com"],
)

# Расширенный — с объектами Email и дополнительными заголовками
let sender = createEmail("alerts@example.com", "Система мониторинга")
let cto    = createEmail("cto@example.com", "CTO")

let msg2 = createMessage(
    mSubject     = "⚠ Критическое заполнение диска",
    mBody        = "Раздел /dev/sda1 заполнен на 95%.",
    sender       = sender,
    mTo          = @[cto],
    otherHeaders = @[("X-Priority", "1"), ("X-Mailer", "MyApp/1.0")],
)
```

> **Устаревшие перегрузки** — `createMessage(mSubject, mBody, mTo, mCc)` и `createMessage(mSubject, mBody, mTo, mCc, otherHeaders)` сохранены для обратной совместимости, но не принимают отправителя. Рекомендуется использовать перегрузку выше.

---

### `sender`

```nim
proc sender*(msg: Message): Email
```

**Что делает.** Возвращает объект `Email`, записанный как отправитель при создании сообщения. Удобно, когда нужно программно проверить или залогировать адрес отправителя, не разбирая исходную строку письма.

```nim
let msg = createMessage("Привет", "Тело", "alice@example.com", mTo = @["bob@example.com"])
echo msg.sender().address   # → alice@example.com
```

---

### `recipients`

```nim
proc recipients*(msg: Message): seq[Email]
```

**Что делает.** Возвращает объединённый список **всех** получателей: To + Cc + Bcc. Именно эти адреса нужно передать в SMTP-команды `RCPT TO`. Обратите внимание: адреса Bcc включены здесь, хотя в заголовках письма они скрыты от видимых получателей.

```nim
let msg = createMessage(
    "Привет", "Тело", "me@example.com",
    mTo  = @["a@example.com"],
    mCc  = @["b@example.com"],
    mBcc = @["c@example.com"],
)

for r in msg.recipients():
    echo r.address   # a@example.com, b@example.com, c@example.com
```

---

### `$` (Email → строка)

```nim
proc `$`*(email: Email): string
```

**Что делает.** Преобразует `Email` в строку по стандарту RFC 2822. Если задано отображаемое имя — результат `"Имя" <адрес>`, иначе просто `адрес`.

```nim
echo createEmail("alice@example.com", "Алиса")   # "Алиса" <alice@example.com>
echo createEmail("alice@example.com")             # alice@example.com
```

---

### `$` (Message → строка)

```nim
proc `$`*(msg: Message): string
```

**Что делает.** Сериализует `Message` в полный RFC 2822-текст: все заголовки, пустая строка-разделитель, затем тело. Именно эту строку нужно передать в `sendMail`.

```nim
let msg = createMessage("Тест", "Привет!", "me@example.com", mTo = @["you@example.com"])
echo $msg
# From: me@example.com
# To: you@example.com
# Subject: Тест
#
# Привет!
```

---

## Управление подключением

### `newSmtp`

```nim
proc newSmtp*(useSsl = false, debug = false, sslContext: SslContext = nil): Smtp
```

**Что делает.** Создаёт и возвращает новый синхронный SMTP-клиент. Объект хранит сокет и настройки, но **не устанавливает** соединение — для этого вызовите `connect` отдельно.

| Параметр | Назначение |
|----------|-----------|
| `useSsl` | Обернуть сокет SSL/TLS сразу при создании (подходит для порта 465 с неявным TLS). Требует `-d:ssl`. |
| `debug` | Выводить в stdout каждую отправленную (`C:`) и полученную (`S:`) строку. Незаменимо при отладке. |
| `sslContext` | Собственный `SslContext` для тонкой настройки сертификатов; `nil` — использовать встроенный разрешающий контекст. |

```nim
# Обычное подключение (порт 25 / 587, затем STARTTLS)
let smtp = newSmtp(debug = true)

# Неявный SSL (порт 465)
let smtps = newSmtp(useSsl = true)
```

---

### `newAsyncSmtp`

```nim
proc newAsyncSmtp*(useSsl = false, debug = false, sslContext: SslContext = nil): AsyncSmtp
```

**Что делает.** То же самое, что `newSmtp`, но возвращает `AsyncSmtp` для использования внутри `async`-процедур. Все последующие вызовы (`connect`, `auth`, `sendMail`, …) должны предваряться `await`.

```nim
import asyncdispatch

proc main() {.async.} =
    let smtp = newAsyncSmtp(useSsl = true, debug = true)
    await smtp.connect("smtp.gmail.com", Port 465)
    # ...

waitFor main()
```

---

### `connect`

```nim
proc connect*(smtp: Smtp | AsyncSmtp, address: string, port: Port) {.multisync.}
```

**Что делает.** Открывает TCP-подключение к почтовому серверу, ожидает приветственный баннер (`220`) и согласовывает возможности через `EHLO` (с откатом на `HELO` для старых серверов). Сетевое взаимодействие начинается именно здесь.

Поднимает `ReplyError`, если приветствие не `220`, или ошибку сокета, если хост недоступен.

```nim
smtp.connect("smtp.example.com", Port 587)   # порт STARTTLS
smtp.connect("smtp.gmail.com",   Port 465)   # порт неявного SSL
```

---

### `startTls`

```nim
proc startTls*(smtp: Smtp | AsyncSmtp, sslContext: SslContext = nil) {.multisync.}
```

**Что делает.** Переводит уже открытое TCP-соединение в режим TLS: отправляет команду `STARTTLS`, выполняет TLS-рукопожатие и заново согласовывает возможности по зашифрованному каналу. Вызывать **после** `connect` и **до** `auth`, чтобы учётные данные не ушли по открытому каналу.

Требует `-d:ssl`. Поднимает `ReplyError`, если сервер не поддерживает `STARTTLS`.

```nim
smtp.connect("smtp.mailtrap.io", Port 2525)
smtp.startTls()          # поднимаем TLS
smtp.auth("user", "pw")  # логин уже по зашифрованному каналу
```

---

### `auth`

```nim
proc auth*(smtp: Smtp | AsyncSmtp, username, password: string) {.multisync.}
```

**Что делает.** Аутентифицирует пользователя на SMTP-сервере через механизм `AUTH LOGIN`. Имя пользователя и пароль кодируются Base64 (как требует протокол) и передаются в режиме «вызов—ответ». При успехе сервер отвечает кодом `235`; любой другой код поднимает `ReplyError`.

> **Замечание по безопасности:** Base64 — это кодирование, а не шифрование. Всегда вызывайте `startTls` (или используйте `useSsl = true`) до `auth`, чтобы учётные данные передавались по зашифрованному каналу.

```nim
smtp.auth("myuser@gmail.com", "пароль-приложения")
```

---

### `close`

```nim
proc close*(smtp: Smtp | AsyncSmtp) {.multisync.}
```

**Что делает.** Отправляет серверу команду `QUIT` для корректного завершения сеанса SMTP и закрывает сокет. Всегда вызывайте `close` после окончания работы, чтобы сервер мог аккуратно освободить ресурсы сессии.

```nim
smtp.sendMail(msg)
smtp.close()
```

---

## Отправка писем

### `sendMail` (низкоуровневый вариант)

```nim
proc sendMail*(smtp: Smtp | AsyncSmtp, fromAddr: string, toAddrs: seq[string], msg: string) {.multisync.}
```

**Что делает.** Выполняет полную последовательность SMTP-команд для передачи письма: `MAIL FROM`, `RCPT TO` для каждого получателя, `DATA`, тело письма, завершающая точка (`.`). Строка `msg` должна быть полным RFC 2822-текстом (результат `$myMessage`).

Это низкоуровневый вариант: вы сами управляете адресом отправителя конверта и списком получателей. Полезно, когда адрес конверта (`MAIL FROM`) должен отличаться от заголовка `From:`.

**Защита от инъекций:** `fromAddr` и каждый элемент `toAddrs` не должны содержать символов новой строки — иначе `AssertionDefect`.

```nim
let body = $msg   # сериализуем Message в строку RFC 2822
smtp.sendMail("sender@example.com", @["recipient@example.com"], body)
```

---

### `sendMail` (удобный вариант)

```nim
proc sendMail*(smtp: Smtp | AsyncSmtp, msg: Message) {.multisync.}
```

**Что делает.** Обёртка над низкоуровневым вариантом. Автоматически извлекает адрес отправителя из `msg.sender()` и полный список получателей из `msg.recipients()` (To + Cc + Bcc), сериализует письмо и вызывает `sendMail`. Для большинства приложений этого достаточно.

```nim
let msg = createMessage("Тема", "Тело", "me@example.com", mTo = @["you@example.com"])
smtp.sendMail(msg)   # отправитель и получатели определяются автоматически
```

---

## Низкоуровневый / расширительный API

Эти процедуры входят в публичный API, но предназначены для продвинутых сценариев — например, реализации нестандартных SMTP-расширений (`AUTH PLAIN`, `CHUNKING` и т. д.). В обычной работе они не нужны.

### `debugSend`

```nim
proc debugSend*(smtp: Smtp | AsyncSmtp, cmd: string) {.multisync.}
```

**Что делает.** Отправляет произвольную строку через SMTP-сокет. Если `debug = true`, предварительно выводит `C:<cmd>` в stdout. Используйте, когда нужно отправить команду, которую модуль не предоставляет напрямую.

```nim
smtp.debugSend("NOOP\c\L")   # ping-команда, чтобы не разорвалось соединение
```

---

### `debugRecv`

```nim
proc debugRecv*(smtp: Smtp | AsyncSmtp): Future[string] {.multisync.}
```

**Что делает.** Считывает одну строку от SMTP-сервера. Если `debug = true`, выводит `S:<строка>` в stdout. Возвращает сырую строку ответа.

```nim
let banner = smtp.debugRecv()   # напр. "220 smtp.example.com ESMTP"
```

---

### `checkReply`

```nim
proc checkReply*(smtp: Smtp | AsyncSmtp, reply: string) {.multisync.}
```

**Что делает.** Читает следующий ответ сервера через `debugRecv` и проверяет, начинается ли он с ожидаемого префикса `reply` (обычно трёхзначный код состояния). Если нет — отправляет `QUIT` и поднимает `ReplyError`. Используется в паре с `debugSend` при построении собственных последовательностей SMTP-команд.

```nim
smtp.debugSend("NOOP\c\L")
smtp.checkReply("250")   # поднимет ReplyError, если ответ другой
```

---

## Полные примеры

### Gmail (неявный SSL, порт 465)

```nim
import smtp

let msg = createMessage(
    mSubject = "Привет из Nim",
    mBody    = "Это письмо отправлено программой на Nim.",
    sender   = "me@gmail.com",
    mTo      = @["friend@example.com"],
)

let smtp = newSmtp(useSsl = true, debug = true)
smtp.connect("smtp.gmail.com", Port 465)
smtp.auth("me@gmail.com", "пароль-приложения")
smtp.sendMail(msg)
smtp.close()
```

### STARTTLS (порт 587 / 2525)

```nim
import smtp

let msg = createMessage(
    mSubject = "Тест STARTTLS",
    mBody    = "Зашифровано через STARTTLS.",
    sender   = "me@example.com",
    mTo      = @["you@example.com"],
)

let smtp = newSmtp(debug = true)
smtp.connect("smtp.mailtrap.io", Port 2525)
smtp.startTls()
smtp.auth("username", "password")
smtp.sendMail(msg)
smtp.close()
```

### Асинхронная отправка

```nim
import smtp, asyncdispatch

proc sendAsync() {.async.} =
    let msg = createMessage(
        "Асинхронное письмо", "Отправлено асинхронно!",
        sender = "bot@example.com",
        mTo = @["admin@example.com"],
    )
    let smtp = newAsyncSmtp(useSsl = true)
    await smtp.connect("smtp.example.com", Port 465)
    await smtp.auth("bot@example.com", "секрет")
    await smtp.sendMail(msg)
    await smtp.close()

waitFor sendAsync()
```
