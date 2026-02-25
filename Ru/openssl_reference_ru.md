# openssl.nim — полный справочник функций

> **Модуль**: `std/openssl` (стандартная библиотека Nim)  
> **Назначение**: Тонкая Nim-обёртка над C-библиотекой OpenSSL (`libssl` + `libcrypto`). Охватывает управление TLS-контекстами, работу с сертификатами, криптографические хэши, HMAC, RSA-операции, MD5, BIO-ввод/вывод, а также расширения ALPN и SNI.  
> **Минимальная поддерживаемая версия**: OpenSSL ≥ 1.1.0 (динамическая компоновка по умолчанию).  
> **Флаг компиляции**: при использовании модуля компилятору Nim **обязательно** передавать `-d:ssl`.

---

## Содержание

1. [Ключевые типы](#ключевые-типы)
2. [Важные константы](#важные-константы)
3. [Инициализация библиотеки](#инициализация-библиотеки)
4. [Выбор SSL/TLS-метода](#выбор-ssltls-метода)
5. [Управление SSL-контекстом (SslCtx)](#управление-ssl-контекстом-sslctx)
6. [Управление SSL-соединением (SslPtr)](#управление-ssl-соединением-sslptr)
7. [Загрузка сертификатов и ключей](#загрузка-сертификатов-и-ключей)
8. [BIO — абстракция ввода/вывода](#bio--абстракция-вводавывода)
9. [Обработка ошибок](#обработка-ошибок)
10. [SNI — указание имени сервера](#sni--указание-имени-сервера)
11. [ECDH и настройка шифров](#ecdh-и-настройка-шифров)
12. [PSK — обратные вызовы с предварительно разделённым ключом](#psk--обратные-вызовы-с-предварительно-разделённым-ключом)
13. [ALPN — согласование протокола прикладного уровня](#alpn--согласование-протокола-прикладного-уровня)
14. [Просмотр X.509-сертификатов](#просмотр-x509-сертификатов)
15. [Хранилище X.509-сертификатов](#хранилище-x509-сертификатов)
16. [EVP-хэш-функции](#evp-хэш-функции)
17. [HMAC](#hmac)
18. [RSA-операции](#rsa-операции)
19. [Чтение ключей из PEM](#чтение-ключей-из-pem)
20. [EVP-подписание (DigestSign)](#evp-подписание-digestsign)
21. [Функции MD5](#функции-md5)
22. [Версия и управление памятью](#версия-и-управление-памятью)

---

## Ключевые типы

| Тип | Описание |
|-----|----------|
| `SslPtr` | Непрозрачный указатель, соответствующий `SSL*` в C API. Представляет одно активное TLS-соединение. |
| `SslCtx` | Псевдоним `SslPtr`, используемый как **контекст** (`SSL_CTX*`). Хранит общие настройки для множества соединений. |
| `PSSL_METHOD` | Указатель на дескриптор метода SSL/TLS (например, TLS 1.2, TLS 1.3). |
| `BIO` | Универсальный объект ввода/вывода OpenSSL. Может оборачивать сокет, буфер в памяти, SSL-поток и т. д. |
| `PX509` | Указатель на структуру X.509-сертификата. |
| `PX509_NAME` | Указатель на отличительное имя X.509 (subject или issuer). |
| `EVP_MD` | Указатель на дескриптор алгоритма хэширования (SHA-256 и т. д.). |
| `EVP_MD_CTX` | Контекстный объект, хранящий состояние текущего вычисления хэша. |
| `EVP_PKEY` | Указатель на универсальный открытый/закрытый ключ. |
| `EVP_PKEY_CTX` | Контекст для операций с `EVP_PKEY`-ключами (подпись, шифрование). |
| `PRSA` | Указатель на структуру RSA-ключа. |
| `PaddingType` | Перечисление схем дополнения для RSA: `RSA_PKCS1_PADDING`, `RSA_PKCS1_OAEP_PADDING`, `RSA_NO_PADDING` и т. д. |
| `MD5_CTX` | Структура, хранящая промежуточное состояние инкрементального вычисления MD5. |
| `pem_password_cb` | Тип обратного вызова для чтения зашифрованных PEM-ключей. |
| `PskClientCallback` | Тип обратного вызова для предоставления PSK-идентификатора и ключа на стороне клиента. |
| `PskServerCallback` | Тип обратного вызова для проверки PSK на стороне сервера. |

---

## Важные константы

### Коды ошибок SSL (возвращаются `SSL_get_error`)

| Константа | Значение | Смысл |
|-----------|----------|-------|
| `SSL_ERROR_NONE` | 0 | Ошибок нет. |
| `SSL_ERROR_SSL` | 1 | Фатальная ошибка протокола SSL/TLS. Проверьте очередь ошибок. |
| `SSL_ERROR_WANT_READ` | 2 | Неблокирующий I/O: нужно подождать, пока данные станут доступны для чтения. |
| `SSL_ERROR_WANT_WRITE` | 3 | Неблокирующий I/O: нужно подождать, пока сокет станет готов к записи. |
| `SSL_ERROR_SYSCALL` | 5 | Ошибка на уровне ОС; проверьте `errno`. |
| `SSL_ERROR_ZERO_RETURN` | 6 | Соединение чисто закрыто удалённой стороной. |

### Флаги режима SSL (используются с `SSLCTXSetMode`)

| Константа | Смысл |
|-----------|-------|
| `SSL_MODE_ENABLE_PARTIAL_WRITE` | Разрешить `SSL_write` возвращать управление при частичной записи. |
| `SSL_MODE_ACCEPT_MOVING_WRITE_BUFFER` | Разрешить повторную отправку с другим указателем буфера (содержимое должно быть тем же). |
| `SSL_MODE_AUTO_RETRY` | В блокирующем режиме автоматически повторять операции при `WANT_READ`/`WANT_WRITE`. |

### Флаги параметров SSL (используются с `SSL_CTX_ctrl`)

| Константа | Смысл |
|-----------|-------|
| `SSL_OP_NO_SSLv2` | Отключить SSLv2. |
| `SSL_OP_NO_SSLv3` | Отключить SSLv3. |
| `SSL_OP_NO_TLSv1` | Отключить TLSv1.0. |
| `SSL_OP_NO_TLSv1_1` | Отключить TLSv1.1. |

### Результаты проверки сертификатов

`X509_V_OK` (0) означает успех. Ненулевое значение — ошибка, например:

| Константа | Смысл |
|-----------|-------|
| `X509_V_ERR_CERT_HAS_EXPIRED` | Срок действия сертификата истёк. |
| `X509_V_ERR_DEPTH_ZERO_SELF_SIGNED_CERT` | Листовой сертификат самоподписан. |
| `X509_V_ERR_CERT_REVOKED` | Сертификат отозван. |

---

## Инициализация библиотеки

### `SSL_library_init(): cint`

Инициализирует библиотеку OpenSSL. Должна вызываться **один раз** до использования любых других SSL-функций. На OpenSSL ≥ 1.1.0 делегирует вызов `OPENSSL_init_ssl`; на старых версиях вызывает оригинальный `SSL_library_init`. Возвращаемое значение можно игнорировать.

```nim
discard SSL_library_init()
```

### `SSL_load_error_strings()`

Регистрирует человекочитаемые строки ошибок, чтобы `ERR_error_string` возвращала осмысленные сообщения. Удалена в OpenSSL 1.1.0, поэтому обёртка превращается в пустую операцию на новых версиях.

```nim
SSL_load_error_strings()  # безопасно на любой версии
```

### `ERR_load_BIO_strings()`

Загружает строки ошибок для BIO-операций. Аналогично `SSL_load_error_strings`, является no-op на OpenSSL ≥ 1.1.0.

### `OpenSSL_add_all_algorithms()`

Регистрирует все доступные алгоритмы хэширования и шифрования во внутренней таблице поиска. Необходима перед поиском алгоритмов по имени. No-op на OpenSSL ≥ 1.1.0, но безвредна для вызова.

```nim
SSL_library_init()
SSL_load_error_strings()
OpenSSL_add_all_algorithms()
```

### `getOpenSSLVersion(): culong`

Возвращает версию библиотеки OpenSSL в виде упакованного беззнакового целого числа в формате `0xMNN00PP0L`. Возвращает 0, если библиотека недоступна.

```nim
let v = getOpenSSLVersion()
echo v.toHex  # например "10101000" для OpenSSL 1.1.1
```

### `CRYPTO_malloc_init()`

Заменяет внутренний аллокатор OpenSSL на аллокатор Nim (`allocShared`/`deallocShared`). Вызывайте в самом начале при совместном использовании Nim GC и OpenSSL, чтобы избежать освобождения памяти через чужой аллокатор. No-op на платформах, где это неприменимо.

---

## Выбор SSL/TLS-метода

Эти функции возвращают `PSSL_METHOD`, который передаётся в `SSL_CTX_new`. Они определяют, какие версии протокола будут согласовываться.

### `TLS_method(): PSSL_METHOD`

Современный, рекомендуемый метод. Поддерживает SSLv3, TLSv1.0, TLSv1.1, TLSv1.2 и TLSv1.3 (в зависимости от версии OpenSSL). Согласовывается наивысшая взаимно поддерживаемая версия.

```nim
let ctx = SSL_CTX_new(TLS_method())
```

### `TLS_client_method(): PSSL_METHOD`

То же, что `TLS_method()`, но явно для клиентских соединений.

### `TLS_server_method(): PSSL_METHOD`

То же, что `TLS_method()`, но явно для серверных соединений.

### `SSLv23_method(): PSSL_METHOD`

Псевдоним совместимости для `TLS_method()`. В старом OpenSSL название «SSLv23» означало «договориться о наилучшей доступной версии».

### `SSLv23_client_method(): PSSL_METHOD`

Псевдоним совместимости для `TLS_client_method()`.

### `TLSv1_method(): PSSL_METHOD`

Принудительно использует только TLSv1.0. Избегайте, если только не нужна совместимость с устаревшим оборудованием; TLSv1.0 считается устаревшим.

### `SSLv2_method() / SSLv3_method(): PSSL_METHOD`

Принудительно использует SSLv2 или SSLv3. Оба протокола **криптографически скомпрометированы**. Предоставляются исключительно для исторической совместимости.

---

## Управление SSL-контекстом (SslCtx)

`SslCtx` — это фабрика и объект конфигурации. Вы создаёте один контекст, настраиваете на нём сертификаты, ключи, параметры проверки и список шифров, а затем порождаете из него отдельные объекты `SslPtr`-соединений.

### `SSL_CTX_new(meth: PSSL_METHOD): SslCtx`

Создаёт новый SSL-контекст с заданным методом. При ошибке возвращает `nil`.

```nim
let ctx = SSL_CTX_new(TLS_client_method())
if ctx.isNil:
  raise newException(Exception, "Не удалось создать SSL-контекст")
```

### `SSL_CTX_free(ctx: SslCtx)`

Уничтожает контекст и освобождает всю связанную память. Вызывайте по завершении работы с контекстом.

```nim
SSL_CTX_free(ctx)
```

### `SSL_CTX_set_verify(s: SslCtx, mode: int, cb: proc)`

Управляет проверкой сертификата удалённой стороны.  
- `mode = SSL_VERIFY_NONE` — не проверять (опасно для клиентов!).  
- `mode = SSL_VERIFY_PEER` — запросить и проверить сертификат удалённой стороны.  
- `cb` — необязательный пользовательский обратный вызов проверки; передайте `nil` для использования поведения по умолчанию.

```nim
# Требовать проверку сертификата, коллбэк по умолчанию
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, nil)
```

### `SSL_CTX_load_verify_locations(ctx, CAfile, CApath): cint`

Указывает контексту, где искать доверенные CA-сертификаты. `CAfile` — PEM-файл с одним или несколькими сертификатами; `CApath` — директория с сертификатами, именованными по хэшу. Передайте `nil` для любого из параметров, чтобы его игнорировать. Возвращает 1 в случае успеха.

```nim
let ok = SSL_CTX_load_verify_locations(ctx, "/etc/ssl/certs/ca-certificates.crt", nil)
```

### `SSL_CTX_set_cipher_list(s: SslCtx, ciphers: cstring): cint`

Задаёт список допустимых наборов шифров TLS 1.0–1.2 в нотации OpenSSL через двоеточие. Возвращает 1, если хотя бы один шифр валиден.

```nim
discard SSL_CTX_set_cipher_list(ctx, "HIGH:!aNULL:!MD5")
```

### `SSL_CTX_set_ciphersuites(ctx: SslCtx, str: cstring): cint`

Задаёт список допустимых наборов шифров **TLS 1.3**. Синтаксис отличается от `set_cipher_list` — используйте имена вида `TLS_AES_256_GCM_SHA384`.

```nim
discard SSL_CTX_set_ciphersuites(ctx, "TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256")
```

### `SSL_CTX_ctrl(ctx, cmd, larg, parg): clong`

Низкоуровневая управляющая функция, используемая внутри для многих настроек SSL-контекста. Предпочитайте вспомогательные обёртки (`SSLCTXSetMode`, `SSL_CTX_set_ecdh_auto` и т. д.) прямому вызову.

### `SSLCTXSetMode(ctx: SslCtx, mode: int): int`

Удобная обёртка над `SSL_CTX_ctrl`, устанавливающая один или несколько битов режима. Возвращает новое суммарное значение флагов.

```nim
discard SSLCTXSetMode(ctx, SSL_MODE_AUTO_RETRY)
```

### `SSL_CTX_set_session_id_context(context, sid_ctx, sid_ctx_len)`

Задаёт строку идентификатора сессии, которая используется для идентификации сессий, созданных этим контекстом. Важно для возобновления сессий на стороне сервера при использовании нескольких контекстов.

### `SSL_CTX_get_ex_new_index / SSL_CTX_set_ex_data / SSL_CTX_get_ex_data`

Три функции для хранения произвольных пользовательских данных в `SslCtx` по индексу:

1. `SSL_CTX_get_ex_new_index` — зарегистрировать новый слот.
2. `SSL_CTX_set_ex_data` — сохранить `pointer` в этом слоте.
3. `SSL_CTX_get_ex_data` — извлечь сохранённый указатель.

```nim
let idx = SSL_CTX_get_ex_new_index(0, nil, nil, nil, nil)
var myData = "привет"
discard SSL_CTX_set_ex_data(ctx, idx, addr myData)
let p = SSL_CTX_get_ex_data(ctx, idx)
```

---

## Управление SSL-соединением (SslPtr)

### `SSL_new(context: SslCtx): SslPtr`

Создаёт новый объект SSL-соединения из контекста. Соединение ещё не привязано к сокету.

```nim
let ssl = SSL_new(ctx)
```

### `SSL_free(ssl: SslPtr)`

Уничтожает объект соединения и освобождает его память.

### `SSL_set_fd(ssl: SslPtr, fd: SocketHandle): cint`

Связывает SSL-объект с файловым дескриптором ОС-сокета. После этого вызова можно использовать `SSL_connect` или `SSL_accept` для выполнения TLS-рукопожатия.

```nim
discard SSL_set_fd(ssl, socket.fd)
```

### `SSL_connect(ssl: SslPtr): cint`

Выполняет TLS-рукопожатие на **клиентской** стороне. Возвращает 1 при успехе, ≤ 0 при ошибке (используйте `SSL_get_error` для расшифровки).

```nim
let ret = SSL_connect(ssl)
if ret != 1:
  let err = SSL_get_error(ssl, ret.cint)
  echo "Рукопожатие не удалось: ", err
```

### `SSL_accept(ssl: SslPtr): cint`

Выполняет TLS-рукопожатие на **серверной** стороне. Возвращает 1 при успехе.

### `SSL_do_handshake(ssl: SslPtr): cint` *(через `sslDoHandshake`)*

Универсальное рукопожатие, работающее как для клиента, так и для сервера в зависимости от состояния соединения.

### `SSL_read(ssl: SslPtr, buf: pointer, num: int): cint`

Читает расшифрованные прикладные данные из соединения в `buf`. Возвращает количество прочитанных байт, 0 при чистом закрытии или отрицательное значение при ошибке.

```nim
var buf = newString(4096)
let n = SSL_read(ssl, buf.cstring, buf.len.cint)
if n > 0:
  echo buf[0..<n]
```

### `SSL_write(ssl: SslPtr, buf: cstring, num: int): cint`

Шифрует и отправляет прикладные данные. Возвращает количество записанных байт или ≤ 0 при ошибке.

```nim
let msg = "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n"
discard SSL_write(ssl, msg, msg.len)
```

### `SSL_pending(ssl: SslPtr): cint`

Возвращает количество байт, уже расшифрованных и буферизованных внутри OpenSSL, готовых к чтению без дополнительного обращения к сети.

### `SSL_shutdown(ssl: SslPtr): cint`

Отправляет TLS-оповещение `close_notify` для начала чистого завершения сессии. Может потребоваться два вызова (первый отправляет, второй ждёт ответа от удалённой стороны). Возвращает 0, когда первый `close_notify` отправлен; 1, когда обе стороны завершили соединение; <0 при ошибке.

### `SSL_set_shutdown / SSL_get_shutdown`

Вручную установить или запросить флаги состояния завершения (`SSL_SENT_SHUTDOWN`, `SSL_RECEIVED_SHUTDOWN`).

### `SSL_get_error(s: SslPtr, ret_code: cint): cint`

Переводит «сырое» возвращаемое значение `SSL_connect`, `SSL_read`, `SSL_write` и т. д. в осмысленный код ошибки. Всегда вызывайте сразу после неудачной SSL-операции.

```nim
let err = SSL_get_error(ssl, ret.cint)
case err
of SSL_ERROR_WANT_READ:   echo "нужно больше данных"
of SSL_ERROR_ZERO_RETURN: echo "соединение закрыто"
else: echo "ошибка ", err
```

### `SSL_get_verify_result(ssl: SslPtr): int`

Возвращает результат проверки сертификата удалённой стороны, выполненной во время рукопожатия. Сравнивайте с `X509_V_OK` (0).

```nim
if SSL_get_verify_result(ssl) != X509_V_OK:
  raise newException(Exception, "Проверка сертификата не пройдена")
```

### `SSL_get_SSL_CTX(ssl: SslPtr): SslCtx`

Возвращает контекст, связанный с данным соединением.

### `SSL_set_SSL_CTX(ssl: SslPtr, ctx: SslCtx): SslCtx`

Заменяет контекст существующего SSL-соединения. Полезно при виртуальном хостинге (в SNI-обратных вызовах).

### `SSL_get0_verified_chain(ssl: SslPtr): PSTACK`

Возвращает проверенную цепочку сертификатов после успешного рукопожатия в виде `PSTACK` (стек OpenSSL). Удобно для просмотра промежуточных сертификатов.

### `SSL_in_init(ssl: SslPtr): cint`

Возвращает ненулевое значение, если TLS-рукопожатие ещё не завершено.

### `sslSetConnectState(s: SslPtr)`

Переводит SSL-объект в режим **клиента** перед вызовом `sslDoHandshake`. Используется при неблокирующем I/O через BIO.

### `sslSetAcceptState(s: SslPtr)`

Переводит SSL-объект в режим **сервера** перед вызовом `sslDoHandshake`.

### `sslRead / sslPeek / sslWrite`

Низкоуровневые псевдонимы для `SSL_read` / неразрушающего чтения / `SSL_write`, используемые внутри сетевого уровня Nim. Принимают буфер типа `cstring` вместо сырого указателя.

### `sslSetBio(ssl: SslPtr, rbio, wbio: BIO)`

Подключает два BIO-объекта к SSL-дескриптору: `rbio` для чтения, `wbio` для записи. Полезно при неблокирующем I/O через in-memory BIO без прямого использования сокетов.

```nim
let rbio = bioNew(bioSMem())
let wbio = bioNew(bioSMem())
sslSetBio(ssl, rbio, wbio)
```

---

## Загрузка сертификатов и ключей

### `SSL_CTX_use_certificate_file(ctx, filename, typ): cint`

Загружает один сертификат из файла `filename`. `typ` — `SSL_FILETYPE_PEM` (1) или `SSL_FILETYPE_ASN1` (2).

```nim
discard SSL_CTX_use_certificate_file(ctx, "server.crt", SSL_FILETYPE_PEM)
```

### `SSL_CTX_use_certificate_chain_file(ctx, filename): cint`

Загружает сертификат *и* дополнительные промежуточные сертификаты из PEM-файла. Предпочтительнее `use_certificate_file` для production-серверов.

```nim
discard SSL_CTX_use_certificate_chain_file(ctx, "fullchain.pem")
```

### `SSL_CTX_use_PrivateKey_file(ctx, filename, typ): cint`

Загружает закрытый ключ из файла `filename` (формат PEM или DER).

```nim
discard SSL_CTX_use_PrivateKey_file(ctx, "server.key", SSL_FILETYPE_PEM)
```

### `SSL_CTX_check_private_key(ctx: SslCtx): cint`

Проверяет, что загруженный в `ctx` закрытый ключ соответствует сертификату. Возвращает 1 при совпадении. Всегда вызывайте после загрузки обоих.

```nim
if SSL_CTX_check_private_key(ctx) != 1:
  raise newException(Exception, "Несоответствие ключа и сертификата")
```

---

## BIO — абстракция ввода/вывода

BIO (Basic I/O) — уровень абстракции ввода/вывода OpenSSL. Может представлять сокет, SSL-поток или буфер в памяти.

### `bioNew(b: PBIO_METHOD): BIO`

Выделяет новый BIO заданного типа.

### `bioFreeAll(b: BIO)`

Освобождает BIO и все BIO, связанные с ним в цепочке.

### `bioSMem(): PBIO_METHOD`

Возвращает метод BIO для in-memory буфера. Используйте с `bioNew` для создания буферного BIO.

```nim
let memBio = bioNew(bioSMem())  # BIO в виде буфера в памяти
```

### `bioCtrlPending(b: BIO): cint`

Возвращает количество байт, ожидающих чтения из BIO.

### `bioRead(b: BIO, Buf: cstring, length: cint): cint`

Читает `length` байт из BIO в `Buf`. Возвращает число прочитанных байт.

### `bioWrite(b: BIO, Buf: cstring, length: cint): cint`

Записывает `length` байт из `Buf` в BIO.

### `BIO_new_mem_buf(data: pointer, len: cint): BIO`

Создаёт BIO в режиме чтения поверх существующего буфера в памяти. Полезно для передачи PEM-данных из строки вместо файла.

```nim
let pemData = "-----BEGIN CERTIFICATE-----\n..."
let bio = BIO_new_mem_buf(pemData.cstring, pemData.len.cint)
```

### `BIO_new_ssl_connect(ctx: SslCtx): BIO`

Создаёт цепочку BIO, содержащую SSL BIO и connect BIO. Результат — высокоуровневый объект, позволяющий открыть SSL-соединение без работы с сырыми сокетами.

```nim
let bio = BIO_new_ssl_connect(ctx)
```

### `BIO_ctrl(bio, cmd, larg, arg): int`

Низкоуровневая управляющая функция. Предпочитайте обёртки ниже.

### `BIO_get_ssl(bio, ssl: ptr SslPtr): int`

Извлекает `SslPtr` из цепочки BIO (например, созданной с помощью `BIO_new_ssl_connect`).

### `BIO_set_conn_hostname(bio, name): int`

Задаёт имя хоста и порт для connect-BIO в формате `"hostname:port"`.

```nim
discard BIO_set_conn_hostname(bio, "example.com:443")
```

### `BIO_do_connect(bio) / BIO_do_handshake(bio): int`

Инициирует TCP-соединение и TLS-рукопожатие в цепочке connect-BIO.

```nim
if BIO_do_connect(bio) <= 0:
  ERR_print_errors_fp(stderr)
```

### `BIO_read(b, data, length): cint`

Стандартное BIO-чтение (прямой импорт из C, в отличие от camelCase `bioRead`).

### `BIO_write(b, data, length): cint`

Стандартная BIO-запись.

### `BIO_free(b: BIO): cint`

Освобождает один BIO-объект.

---

## Обработка ошибок

### `ERR_get_error(): culong`

Извлекает и возвращает самый старый код ошибки из очереди ошибок OpenSSL. Возвращает 0, если очередь пуста. Вызывайте в цикле до получения 0, чтобы опустошить все ошибки.

```nim
var code = ERR_get_error()
while code != 0:
  echo ERR_error_string(code, nil)
  code = ERR_get_error()
```

### `ERR_peek_last_error(): culong`

Возвращает самый последний код ошибки **без** удаления его из очереди.

### `ERR_error_string(e: culong, buf: cstring): cstring`

Преобразует код ошибки в читаемую строку. Передайте `nil` для `buf`, чтобы использовать внутренний статический буфер.

```nim
echo ERR_error_string(ERR_get_error(), nil)
```

### `ERR_print_errors_fp(fp: File)`

Выводит всю очередь ошибок в файл (обычно `stderr`). Удобно при отладке.

```nim
ERR_print_errors_fp(stderr)
```

### `ErrClearError()`

Очищает все ошибки из очереди. Вызывайте перед операцией, если хотите отслеживать только *новые* ошибки.

### `ErrFreeStrings()`

Освобождает память, занятую загруженными строками ошибок. Вызывайте при завершении программы.

### `ErrRemoveState(pid: cint)`

Удаляет состояние ошибки для потока, идентифицированного `pid`. Используется при очистке ресурсов потока.

---

## SNI — указание имени сервера

SNI позволяет TLS-клиенту объявить в ClientHello имя хоста, к которому он подключается, чтобы сервер мог выбрать правильный сертификат до завершения рукопожатия.

### `SSL_set_tlsext_host_name(ssl: SslPtr, name: cstring): int`

Задаёт SNI-имя хоста на **клиентской** стороне. Вызывайте до `SSL_connect`. Возвращает 1 при успехе.

```nim
discard SSL_set_tlsext_host_name(ssl, "example.com")
```

### `SSL_get_servername(ssl: SslPtr, typ: cint): cstring`

На **серверной** стороне возвращает имя хоста, переданное клиентом в ClientHello. Обычно вызывается внутри SNI-обратного вызова для реализации виртуального хостинга. Может вернуть `nil`, если SNI не использовался.

```nim
let host = SSL_get_servername(ssl, TLSEXT_NAMETYPE_host_name)
```

### `SSL_CTX_set_tlsext_servername_callback(ctx, cb): int`

Регистрирует обратный вызов, который вызывается при получении ClientHello с SNI-именем хоста. Внутри коллбэка можно вызвать `SSL_set_SSL_CTX` для переключения на другой контекст с другим сертификатом.

```nim
proc sniCb(ssl: SslPtr, cb_id: int, arg: pointer): int {.cdecl.} =
  let host = SSL_get_servername(ssl, TLSEXT_NAMETYPE_host_name)
  if host == "other.example.com":
    discard SSL_set_SSL_CTX(ssl, otherCtx)
  return SSL_TLSEXT_ERR_OK

discard SSL_CTX_set_tlsext_servername_callback(ctx, sniCb)
```

### `SSL_CTX_set_tlsext_servername_arg(ctx, arg): int`

Задаёт пользовательский указатель данных, который будет передан в SNI-обратный вызов, зарегистрированный выше.

---

## ECDH и настройка шифров

### `SSL_CTX_set_ecdh_auto(ctx, onoff: cint): cint`

Включает или отключает автоматический выбор кривой ECDH. На OpenSSL ≥ 1.1.0 функция всегда включена и функция возвращает 1 без каких-либо действий.

```nim
discard SSL_CTX_set_ecdh_auto(ctx, 1)
```

### `OPENSSL_sk_num(stack: PSTACK): int`

Возвращает количество элементов в универсальном стеке OpenSSL (например, цепочке сертификатов).

### `OPENSSL_sk_value(stack: PSTACK, index: int): pointer`

Возвращает элемент по индексу из стека OpenSSL в виде сырого указателя. Приведите к нужному типу.

### `OPENSSL_config(configName: cstring)`

Загружает файл конфигурации OpenSSL. Передайте `nil` для использования файла по умолчанию (`openssl.cnf`).

---

## PSK — обратные вызовы с предварительно разделённым ключом

TLS с PSK (Pre-Shared Key) позволяет аутентификацию без сертификатов с использованием общего секрета, согласованного за пределами протокола.

### `SSL_CTX_set_psk_client_callback(ctx, callback)`

Регистрирует обратный вызов на **клиентской** стороне, который вызывается при согласовании PSK-шифронабора. Коллбэк должен заполнить буферы `identity` и `psk` и вернуть длину PSK.

```nim
proc myPskClient(ssl: SslPtr; hint: cstring; identity: cstring;
                 maxIdentLen: cuint; psk: ptr uint8;
                 maxPskLen: cuint): cuint {.cdecl.} =
  copyMem(identity, "client1", 8)
  let secret = "mysecret"
  copyMem(psk, secret.cstring, secret.len)
  return secret.len.cuint

SSL_CTX_set_psk_client_callback(ctx, myPskClient)
```

### `SSL_CTX_set_psk_server_callback(ctx, callback)`

Аналогично, но для **серверной** стороны. Коллбэк получает строку идентификатора и должен заполнить PSK-буфер.

### `SSL_CTX_use_psk_identity_hint(ctx, hint: cstring): cint`

Задаёт подсказку, которую сервер отправляет клиенту для выбора правильного PSK-идентификатора. Возвращает 1 при успехе.

### `SSL_get_psk_identity(ssl: SslPtr): cstring`

Возвращает PSK-идентификатор, выбранный клиентом после PSK-рукопожатия.

---

## ALPN — согласование протокола прикладного уровня

ALPN позволяет клиенту и серверу согласовать прикладной протокол (например, `"h2"` для HTTP/2, `"http/1.1"`) в процессе TLS-рукопожатия.

### `SSL_CTX_set_alpn_protos(ctx, protos, protos_len): cint`

Задаёт список протоколов, которые готов использовать **клиент**. `protos` — байтовая строка в проводном формате, где перед каждым протоколом стоит байт длины. Возвращает 0 при успехе (обратите внимание: инвертировано относительно большинства функций OpenSSL!).

```nim
# "h2" в проводном формате: \x02h2
let protos = "\x02h2\x08http/1.1"
discard SSL_CTX_set_alpn_protos(ctx, protos, protos.len.cuint)
```

### `SSL_set_alpn_protos(ssl, protos, protos_len): cint`

Версия для одного соединения; переопределяет настройку контекста для конкретного SSL-объекта.

### `SSL_CTX_set_alpn_select_cb(ctx, cb, arg)`

Серверный обратный вызов для ALPN. OpenSSL вызывает его со списком протоколов клиента, и коллбэк выбирает один. Обычно используется вместе с `SSL_select_next_proto`.

### `SSL_get0_alpn_selected(ssl, data, len)`

После рукопожатия возвращает согласованный ALPN-протокол. `data` указывает во внутренний буфер OpenSSL (не освобождайте).

```nim
var proto: cstring
var protoLen: cuint
SSL_get0_alpn_selected(ssl, addr proto, addr protoLen)
echo "Согласован: ", proto[0..<protoLen.int]
```

### `SSL_select_next_proto(out_proto, outlen, server, server_len, client, client_len): cint`

Вспомогательная функция, выполняющая пересечение списков протоколов сервера и клиента по RFC 7301. Возвращает `OPENSSL_NPN_NEGOTIATED` (1) при нахождении совпадения.

### `SSL_CTX_set_next_protos_advertised_cb / SSL_CTX_set_next_proto_select_cb / SSL_get0_next_proto_negotiated`

Более старые варианты NPN (Next Protocol Negotiation), вытесненные ALPN, но присутствующие для совместимости со старыми клиентами.

---

## Просмотр X.509-сертификатов

### `SSL_get_peer_certificate(ssl: SslCtx): PX509`

Возвращает сертификат удалённой стороны (предъявленный во время рукопожатия). Возвращает `nil`, если сертификат не был предоставлен. На OpenSSL 3 делегирует вызов `SSL_get1_peer_certificate`.

```nim
let cert = SSL_get_peer_certificate(ssl)
if cert.isNil:
  echo "Сертификат отсутствует"
```

### `X509_get_subject_name(a: PX509): PX509_NAME`

Возвращает `PX509_NAME` субъекта сертификата (идентификатор владельца).

### `X509_get_issuer_name(a: PX509): PX509_NAME`

Возвращает `PX509_NAME` издателя (идентификатор CA, подписавшего сертификат).

### `X509_NAME_oneline(a: PX509_NAME, buf: cstring, size: cint): cstring`

Преобразует `PX509_NAME` в читаемую однострочную строку вида `/C=US/O=Example/CN=example.com`.

```nim
let subject = X509_NAME_oneline(X509_get_subject_name(cert), nil, 0)
echo subject
```

### `X509_NAME_get_text_by_NID(subject, NID, buf, size): cint`

Извлекает текст конкретного поля из имени по его NID (числовому идентификатору). Например, NID 13 — это `CN` (общее имя).

### `X509_check_host(cert, name, namelen, flags, peername): cint`

Проверяет, действителен ли сертификат для заданного имени хоста. Возвращает 1 при совпадении, 0 при несовпадении, -1 при ошибке. Это корректная, RFC-совместимая функция проверки имени хоста.

```nim
if X509_check_host(cert, "example.com", 11, 0, nil) != 1:
  raise newException(Exception, "Несовпадение имени хоста")
```

### `X509_free(cert: PX509)`

Освобождает объект сертификата. Вызывайте после использования сертификата, возвращённого `SSL_get_peer_certificate`.

### `d2i_X509(b: string): PX509`

Декодирует DER/BER-байтовую строку в структуру X.509-сертификата в памяти.

```nim
let certBytes = readFile("cert.der")
let cert = d2i_X509(certBytes)
```

### `i2d_X509(cert: PX509): string`

Кодирует X.509-сертификат из памяти в DER-байтовую строку.

```nim
let der = i2d_X509(cert)
writeFile("cert.der", der)
```

---

## Хранилище X.509-сертификатов

### `X509_STORE_new(): PX509_STORE`

Создаёт новое пустое хранилище сертификатов.

### `X509_STORE_free(v: PX509_STORE)`

Освобождает хранилище сертификатов и уменьшает счётчики ссылок на все хранящиеся сертификаты.

### `X509_STORE_add_cert(ctx, x: PX509): cint`

Добавляет сертификат в хранилище. Возвращает 1 при успехе.

```nim
let store = X509_STORE_new()
discard X509_STORE_add_cert(store, cert)
```

### `X509_STORE_lock / X509_STORE_unlock`

Потокобезопасная блокировка/разблокировка хранилища.

### `X509_STORE_up_ref(v: PX509_STORE): cint`

Увеличивает счётчик ссылок хранилища. Требуется при совместном использовании хранилища в нескольких контекстах.

### `X509_STORE_set_flags / set_purpose / set_trust`

Тонкая настройка поведения хранилища: расширенные флаги валидации, ограничение назначения сертификатов, модель доверия.

### `X509_OBJECT_new() / X509_OBJECT_free`

Выделяет и освобождает универсальный объект хранилища X.509 (обёртка над сертификатами, CRL и т. д.).

---

## EVP-хэш-функции

EVP (Envelope) — высокоуровневая абстракция OpenSSL для криптографических операций. Алгоритмы хэширования представлены значениями `EVP_MD`.

### Дескрипторы алгоритмов

| Функция | Алгоритм |
|---------|----------|
| `EVP_md_null()` | Нет операции (пустой хэш) |
| `EVP_md5()` | MD5 (128 бит, **не использовать для безопасности**) |
| `EVP_sha1()` | SHA-1 (160 бит, устарел для безопасности) |
| `EVP_sha224()` | SHA-224 |
| `EVP_sha256()` | SHA-256 ✓ рекомендуется |
| `EVP_sha384()` | SHA-384 |
| `EVP_sha512()` | SHA-512 |
| `EVP_ripemd160()` | RIPEMD-160 |
| `EVP_whirlpool()` | Whirlpool |

### `EVP_MD_size(md: EVP_MD): cint`

Возвращает размер вывода алгоритма хэширования в байтах.

```nim
echo EVP_MD_size(EVP_sha256())  # → 32
```

### `EVP_MD_CTX_create(): EVP_MD_CTX`

Выделяет новый контекст хэширования. На Linux отображается на `EVP_MD_CTX_new`; на macOS/Windows — на `EVP_MD_CTX_create`.

### `EVP_MD_CTX_destroy(ctx: EVP_MD_CTX)`

Освобождает контекст хэширования. На Linux отображается на `EVP_MD_CTX_free`.

### `EVP_DigestInit_ex(ctx, typ, engine): cint`

Инициализирует контекст хэширования для заданного алгоритма. Передайте `nil` для `engine`, чтобы использовать движок по умолчанию.

```nim
let ctx = EVP_MD_CTX_create()
discard EVP_DigestInit_ex(ctx, EVP_sha256(), nil)
```

### `EVP_DigestUpdate(ctx, data: pointer, len: cuint): cint`

Подаёт порцию данных в текущее вычисление хэша. Может вызываться многократно.

```nim
let msg = "hello world"
discard EVP_DigestUpdate(ctx, msg.cstring, msg.len.cuint)
```

### `EVP_DigestFinal_ex(ctx, buffer: pointer, size: ptr cuint): cint`

Завершает вычисление хэша и записывает результат в `buffer`. Устанавливает `size` в количество записанных байт.

```nim
var hash: array[32, uint8]
var hashLen: cuint
discard EVP_DigestFinal_ex(ctx, addr hash[0], addr hashLen)
```

---

## HMAC

HMAC (Hash-based Message Authentication Code, код аутентификации сообщений на основе хэша) сочетает секретный ключ с хэш-функцией для аутентификации данных.

### `HMAC(evp_md, key, key_len, d, n, md, md_len): cstring`

Одноразовое вычисление HMAC.  
- `evp_md` — алгоритм хэширования (например, `EVP_sha256()`).  
- `key` / `key_len` — секретный ключ.  
- `d` / `n` — данные для аутентификации.  
- `md` — выходной буфер (не менее `EVP_MAX_MD_SIZE` байт); передайте `nil` для внутреннего статического буфера.  
- `md_len` — устанавливается в количество записанных байт.

```nim
var outLen: cuint
let secret = "mysecret"
let data = "message"
let result = HMAC(EVP_sha256(),
                  secret.cstring, secret.len.cint,
                  data, data.len.csize_t,
                  nil, addr outLen)
# result указывает на 32-байтный HMAC-SHA256
```

---

## RSA-операции

### `RSA_size(rsa: PRSA): cint`

Возвращает размер RSA-модуля в байтах (размер ключа / 8). Выходной буфер для шифрования должен быть не менее этого размера.

### `RSA_public_encrypt(flen, fr, to, rsa, padding): cint`

Шифрует `flen` байт из `fr` с помощью RSA открытого ключа, сохраняя результат в `to`. Возвращает длину шифротекста или −1 при ошибке.

```nim
var cipher = newString(RSA_size(rsa))
let n = RSA_public_encrypt(msg.len.cint, cast[ptr uint8](msg.cstring),
                            cast[ptr uint8](cipher.cstring), rsa, RSA_PKCS1_OAEP_PADDING)
```

### `RSA_private_decrypt(flen, fr, to, rsa, padding): cint`

Расшифровывает шифротекст `fr` с помощью RSA закрытого ключа.

### `RSA_private_encrypt(flen, fr, to, rsa, padding): cint`

Подписывает (шифрует закрытым ключом) сообщение. Используется в «сырой» подписи RSA PKCS#1 v1.5 (для нового кода предпочтительнее EVP-подписание).

### `RSA_public_decrypt(flen, fr, to, rsa, padding): cint`

Верифицирует (расшифровывает открытым ключом) подпись.

### `RSA_verify(kind, origMsg, origMsgLen, signature, signatureLen, rsa): cint`

Проверяет RSA-подпись по исходному сообщению. `kind` — NID использованного алгоритма хэширования. Возвращает 1 при валидной подписи, 0 — при невалидной.

### `RSA_free(rsa: PRSA)`

Освобождает структуру RSA-ключа.

---

## Чтение ключей из PEM

### `PEM_read_bio_PrivateKey(bp, x, cb, u): EVP_PKEY`

Читает PEM-кодированный закрытый ключ из BIO. Поддерживает RSA, EC и другие типы ключей. `cb` — коллбэк для ввода пароля при зашифрованных ключах; передайте `nil` для незашифрованных.

```nim
let bio = BIO_new_mem_buf(pemKey.cstring, pemKey.len.cint)
let pkey = PEM_read_bio_PrivateKey(bio, nil, nil, nil)
```

### `PEM_read_bio_RSA_PUBKEY / PEM_read_RSA_PUBKEY`

Читает RSA открытый ключ из BIO или FILE-указателя в формате `PUBLIC KEY` (SubjectPublicKeyInfo).

### `PEM_read_bio_RSAPublicKey / PEM_read_RSAPublicKey`

Читает RSA открытый ключ в более старом формате `RSA PUBLIC KEY` (PKCS#1).

### `PEM_read_bio_RSAPrivateKey / PEM_read_RSAPrivateKey`

Читает RSA закрытый ключ в PEM-формате из BIO или FILE-указателя.

### `EVP_PKEY_free(p: EVP_PKEY)`

Освобождает объект `EVP_PKEY` и связанный ключевой материал.

---

## EVP-подписание (DigestSign)

Эти функции выполняют совмещённую операцию «хэшировать + подписать» с использованием `EVP_MD_CTX`.

### `EVP_DigestSignInit(ctx, pctx, typ, e, pkey): cint`

Инициализирует контекст для подписания заданным закрытым ключом и алгоритмом хэширования. `pctx` (необязательно) получает указатель на контекст ключа для установки дополнения и т. д.

```nim
let ctx = EVP_MD_CTX_create()
discard EVP_DigestSignInit(ctx, nil, EVP_sha256(), nil, pkey)
```

### `EVP_DigestUpdate(ctx, data, len)` *(общий с хэшированием)*

Подаёт данные в конвейер подписи.

### `EVP_DigestSignFinal(ctx, data: pointer, len: ptr csize_t): cint`

Завершает формирование подписи. Сначала вызовите с `data = nil`, чтобы определить требуемый размер буфера, затем — с выделенным буфером.

```nim
var sigLen: csize_t
discard EVP_DigestSignFinal(ctx, nil, addr sigLen)
var sig = newString(sigLen.int)
discard EVP_DigestSignFinal(ctx, sig.cstring, addr sigLen)
```

### `EVP_PKEY_CTX_new(pkey, e): EVP_PKEY_CTX`

Создаёт контекст ключа для заданного `EVP_PKEY`. Используется для установки режима дополнения RSA.

### `EVP_PKEY_CTX_free(pkeyCtx: EVP_PKEY_CTX)`

Освобождает контекст ключа.

### `EVP_PKEY_sign_init(c: EVP_PKEY_CTX): cint`

Инициализирует контекст ключа для операции подписания.

### `EVP_MD_CTX_cleanup(ctx: EVP_MD_CTX): cint`

Очищает состояние MD-контекста без его освобождения, чтобы его можно было повторно использовать.

---

## Функции MD5

> **Предупреждение**: MD5 криптографически скомпрометирован. Используйте только в некриптографических целях — например, для контрольных сумм или ключей кэша.

### `md5_File(file: string): string`

Высокоуровневая удобная функция. Читает файл по пути `file` и возвращает его MD5-дайджест в виде 32-символьной строки в нижнем регистре (эквивалент вывода `md5sum`).

```nim
echo md5_File("photo.jpg")  # например "d8e8fca2dc0f896fd7cb4cb0031ba249"
```

### `md5_Str(str: string): string`

Возвращает MD5-дайджест строки в виде 32-символьной строки в нижнем регистре.

```nim
echo md5_Str("hello")  # "5d41402abc4b2a76b9719d911017c592"
```

### `md5_Init(c: var MD5_CTX): cint`

Инициализирует `MD5_CTX` для нового инкрементального вычисления хэша.

### `md5_Update(c: var MD5_CTX, data: pointer, len: csize_t): cint`

Подаёт порцию данных в текущее вычисление MD5.

### `md5_Final(md: cstring, c: var MD5_CTX): cint`

Завершает вычисление и записывает 16-байтный бинарный дайджест в `md`.

### `md5(d: ptr uint8, n: csize_t, md: ptr uint8): ptr uint8`

Одноразовый MD5: хэширует `n` байт по адресу `d` и записывает результат в `md`. Возвращает `md`.

### `md5_Transform(c: var MD5_CTX, b: ptr uint8)`

Обрабатывает один 512-битный блок напрямую. Низкоуровневая функция; редко нужна вне самой библиотеки.

---

## Версия и управление памятью

### `getOpenSSLVersion(): culong`

Возвращает версию OpenSSL в упакованном виде. См. [Инициализация библиотеки](#инициализация-библиотеки).

### `CRYPTO_malloc_init()`

Заменяет аллокатор OpenSSL на GC-совместимый аллокатор Nim. См. [Инициализация библиотеки](#инициализация-библиотеки).

### `useOpenssl3*: bool`

Константа времени компиляции, равная `true` при сборке против OpenSSL 3.x (через `-d:useOpenssl3` или когда строка версии начинается с `'3'`).

### `DLLSSLName* / DLLUtilName*`

Константы времени компиляции с именами библиотек для конкретной платформы (например, `"libssl.so.3"`, `"libcrypto-1_1-x64.dll"`). Полезны при загрузке дополнительных символов через `dynlib`.

---

## Быстрый старт: пример TLS-клиента

```nim
import openssl

# 1. Инициализация
discard SSL_library_init()
SSL_load_error_strings()
OpenSSL_add_all_algorithms()

# 2. Создание контекста
let ctx = SSL_CTX_new(TLS_client_method())
discard SSL_CTX_load_verify_locations(ctx, "/etc/ssl/certs/ca-certificates.crt", nil)
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, nil)

# 3. Создание SSL-объекта и привязка к файловому дескриптору сокета
let ssl = SSL_new(ctx)
discard SSL_set_fd(ssl, mySocket.fd)
discard SSL_set_tlsext_host_name(ssl, "example.com")

# 4. Рукопожатие
if SSL_connect(ssl) != 1:
  ERR_print_errors_fp(stderr)
  quit(1)

# 5. Проверка сертификата
if SSL_get_verify_result(ssl) != X509_V_OK:
  echo "Проверка сертификата не пройдена"
  quit(1)

# 6. Отправка/получение данных
discard SSL_write(ssl, "GET / HTTP/1.0\r\n\r\n", 18)
var buf = newString(4096)
let n = SSL_read(ssl, buf.cstring, buf.len.cint)
echo buf[0..<n]

# 7. Завершение соединения
discard SSL_shutdown(ssl)
SSL_free(ssl)
SSL_CTX_free(ctx)
```
