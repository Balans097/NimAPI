# Nim `winlean.nim` — полная обёртка над Windows API

> **О модуле.** `winlean.nim` — это тонкая, самодостаточная обёртка над подмножеством Windows API, необходимым runtime-библиотеке и стандартной библиотеке Nim. Название происходит от флага препроцессора `WIN32_LEAN_AND_MEAN`, который он передаёт компилятору C — этот флаг отключает редко используемые части `<windows.h>`. Вместо огромного модуля `windows` runtime Nim импортирует только этот файл. Модуль полупубличный: он является частью пространства имён `std` и доступен для импорта пользовательским кодом на Windows, однако намеренно предоставляет лишь тщательно отобранное, стабильное подмножество Win32.

---

## Настройка модуля

```nim
import std/dynlib
{.passc: "-DWIN32_LEAN_AND_MEAN".}
```

Флаг `WIN32_LEAN_AND_MEAN` передаётся компилятору C для каждого файла, включающего этот модуль. Он сообщает `<windows.h>` пропустить Winsock 1.x, мультимедийные API и другие крупные опциональные подсистемы, существенно сокращая время компиляции.

При определении `nimPreviewSlimSystem` импортируется `FileHandle` из `std/syncio` и `widestrs` для поддержки `WideCString`.

---

## Псевдонимы примитивных типов

Эти псевдонимы отображают имена типов Windows SDK на типы Nim. Они существуют для того, чтобы сигнатуры Win32 API можно было объявлять в точности так, как они задокументированы в Windows SDK.

| Имя в Nim | Базовый тип | Эквивалент Windows SDK | Примечания |
|---|---|---|---|
| `WinChar` | `Utf16Char` | `WCHAR` | Кодовая единица UTF-16; используется во всех вызовах с суффиксом `W` |
| `Handle` | `int` | `HANDLE` | Непрозрачный дескриптор ОС; `INVALID_HANDLE_VALUE = Handle(-1)` |
| `LONG` | `int32` | `LONG` | 32-битное знаковое целое |
| `ULONG` | `int32` | `ULONG` | 32-битное беззнаковое целое (в Nim отображено на знаковый `int32`) |
| `PULONG` | `ptr int` | `PULONG` | Указатель на `ULONG` |
| `WINBOOL` | `int32` | `BOOL` | Win32-булево: **ненулевое = успех**, ноль = ошибка — противоположно POSIX-соглашению |
| `PBOOL` | `ptr WINBOOL` | `PBOOL` | Указатель на `WINBOOL` |
| `DWORD` | `int32` | `DWORD` | 32-битное беззнаковое значение (отображено на знаковый `int32`; побитовые операции применяйте осторожно) |
| `PDWORD` | `ptr DWORD` | `PDWORD` | Указатель на `DWORD` |
| `LPINT` | `ptr int32` | `LPINT` | Указатель на `int32` |
| `ULONG_PTR` | `uint` | `ULONG_PTR` | Беззнаковое целое размером с указатель; используется для ключей завершения и аналогичного |
| `PULONG_PTR` | `ptr uint` | `PULONG_PTR` | Указатель на `ULONG_PTR` |
| `HDC` | `Handle` | `HDC` | Дескриптор контекста устройства |
| `HGLRC` | `Handle` | `HGLRC` | Дескриптор контекста рендеринга OpenGL |
| `BYTE` | `uint8` | `BYTE` | Беззнаковое 8-битное значение |
| `SockLen` | `cuint` | `socklen_t` | Длина адреса сокета |
| `SocketHandle` | `distinct int` | `SOCKET` | Отдельный тип, исключающий случайное смешение с `Handle` |
| `WinSizeT` | `uint32` или `uint64` | `SIZE_T` | Беззнаковое целое размером с указатель; ширина выбирается по флагу `cpu32` |

> **Соглашение `WINBOOL`.** Windows `BOOL` ненулевое при успехе — это противоположно POSIX. Используйте `isSuccess(result)` вместо `result != 0` для читаемости кода.

---

## Структуры

### `SECURITY_ATTRIBUTES`

```nim
SECURITY_ATTRIBUTES* = object
  nLength*: int32
  lpSecurityDescriptor*: pointer
  bInheritHandle*: WINBOOL
```

Управляет тем, могут ли дочерние процессы наследовать вновь создаваемый объект ядра (канал, файл, процесс, поток). Установите `bInheritHandle = 1` для разрешения наследования. `lpSecurityDescriptor = nil` применяет дескриптор безопасности вызывающего по умолчанию. `nLength` всегда должно равняться `sizeof(SECURITY_ATTRIBUTES)`.

---

### `STARTUPINFO`

```nim
STARTUPINFO* = object
  cb*: int32              # Должно быть sizeof(STARTUPINFO)
  lpReserved*: cstring    # Должно быть nil
  lpDesktop*: cstring     # Рабочий стол/оконная станция, или nil
  lpTitle*: cstring       # Заголовок консольного окна, или nil
  dwX*, dwY*: int32       # Положение окна
  dwXSize*, dwYSize*: int32
  dwXCountChars*, dwYCountChars*: int32
  dwFillAttribute*: int32
  dwFlags*: int32         # Комбинация флагов STARTF_*
  wShowWindow*: int16
  cbReserved2*: int16
  lpReserved2*: pointer
  hStdInput*, hStdOutput*, hStdError*: Handle
```

Передаётся в `createProcessW`. Ключевые поля: установите `dwFlags = STARTF_USESTDHANDLES` и заполните `hStdInput`/`hStdOutput`/`hStdError` для перенаправления стандартного ввода-вывода. Установите `dwFlags = STARTF_USESHOWWINDOW` и `wShowWindow = SW_SHOWNORMAL` (или `0` для скрытия) для управления состоянием окна.

---

### `PROCESS_INFORMATION`

```nim
PROCESS_INFORMATION* = object
  hProcess*: Handle
  hThread*: Handle
  dwProcessId*: int32
  dwThreadId*: int32
```

Заполняется `createProcessW` при успехе. `hProcess` и `hThread` — открытые дескрипторы, которые **необходимо закрыть** через `closeHandle`, когда они больше не нужны. `dwProcessId`/`dwThreadId` — числовые идентификаторы, остающиеся действительными даже после закрытия дескрипторов.

---

### `FILETIME`

```nim
FILETIME* = object  # НЕЛЬЗЯ ИСПОЛЬЗОВАТЬ int64 ИЗ-ЗА ВЫРАВНИВАНИЯ
  dwLowDateTime*: DWORD
  dwHighDateTime*: DWORD
```

64-битная метка времени, представляющая 100-наносекундные интервалы с 1 января 1601 года (UTC). Хранится в двух полях `DWORD`, поскольку Windows гарантирует только 4-байтное выравнивание этой структуры — размещение в `int64` нарушило бы разметку на некоторых архитектурах. Используйте `rdFileTime` для преобразования в `int64` и `toFILETIME` — в обратную сторону.

---

### `BY_HANDLE_FILE_INFORMATION`

```nim
BY_HANDLE_FILE_INFORMATION* = object
  dwFileAttributes*: DWORD
  ftCreationTime*, ftLastAccessTime*, ftLastWriteTime*: FILETIME
  dwVolumeSerialNumber*: DWORD
  nFileSizeHigh*, nFileSizeLow*: DWORD
  nNumberOfLinks*: DWORD
  nFileIndexHigh*, nFileIndexLow*: DWORD
```

Возвращается `getFileInformationByHandle`. Индекс файла (`nFileIndexHigh`/`nFileIndexLow`) вместе с `dwVolumeSerialNumber` однозначно идентифицирует открытый файл — можно проверить, ссылаются ли два дескриптора на один и тот же файл (аналог сравнения inode в POSIX).

---

### `OSVERSIONINFO`

```nim
OSVERSIONINFO* = object
  dwOSVersionInfoSize*: DWORD
  dwMajorVersion*, dwMinorVersion*, dwBuildNumber*, dwPlatformId*: DWORD
  szCSDVersion*: array[0..127, WinChar]
```

Используется с `getVersionExW`/`getVersionExA`. Перед вызовом всегда устанавливайте `dwOSVersionInfoSize = sizeof(OSVERSIONINFO)`. На Windows 8.1+ номера версий зависят от манифеста приложения, поэтому определение по номеру сборки надёжнее.

---

### `WIN32_FIND_DATA`

```nim
WIN32_FIND_DATA* {.pure.} = object
  dwFileAttributes*: int32
  ftCreationTime*, ftLastAccessTime*, ftLastWriteTime*: FILETIME
  nFileSizeHigh*, nFileSizeLow*: int32
  dwReserved0, dwReserved1: int32     # приватные
  cFileName*: array[0..MAX_PATH - 1, WinChar]
  cAlternateFileName*: array[0..13, WinChar]
```

Заполняется `findFirstFileW` и `findNextFileW`. `cFileName` — длинное имя файла (до `MAX_PATH` широких символов). `cAlternateFileName` — короткое имя в формате 8.3 (может быть пустым, если генерация коротких имён отключена на томе). Используйте `rdFileSize` для объединения `nFileSizeLow`/`nFileSizeHigh` в `int64`.

---

### `OVERLAPPED`

```nim
OVERLAPPED* {.pure, inheritable.} = object
  internal*: PULONG
  internalHigh*: PULONG
  offset*: DWORD
  offsetHigh*: DWORD
  hEvent*: Handle
```

Ключевая структура для асинхронного (overlapped) ввода-вывода. Передавайте указатель на `OVERLAPPED` в `readFile`, `writeFile`, `WSARecv`, `WSASend` и т.д. для асинхронной работы. После отправки используйте `getQueuedCompletionStatus` на порте IOCP или опрашивайте `hasOverlappedIoCompleted` для обнаружения завершения. `offset`/`offsetHigh` задают позицию в файле при файловом вводе-выводе; для сокетов игнорируются. `hEvent` может опционально содержать событие с ручным сбросом, которое сигнализируется при завершении.

`internal` и `internalHigh` используются ОС и не должны записываться приложением — они хранят код состояния и число переданных байт, пока операция выполняется.

---

### `GUID`

```nim
GUID* = object
  D1*: int32
  D2*: int16
  D3*: int16
  D4*: array[0..7, int8]
```

Стандартный глобальный уникальный идентификатор. Используется с `WSAIoctl` для запроса указателей на функции-расширения (`WSAID_CONNECTEX`, `WSAID_ACCEPTEX`, `WSAID_GETACCEPTEXSOCKADDRS`).

---

### Типы обратных вызовов для OVERLAPPED

```nim
POVERLAPPED* = ptr OVERLAPPED
POVERLAPPED_COMPLETION_ROUTINE* = proc(para1: DWORD, para2: DWORD,
                                       para3: POVERLAPPED) {.stdcall.}
```

`POVERLAPPED_COMPLETION_ROUTINE` — тип, передаваемый расширенным функциям отправки/получения. `para1` — код ошибки, `para2` — количество переданных байт, `para3` — структура overlapped.

---

### Структуры адресов Winsock

| Тип | Соответствует | Назначение |
|---|---|---|
| `SockAddr` | `SOCKADDR` | Универсальный адрес сокета (семейство + непрозрачные данные) |
| `Sockaddr_in` | `SOCKADDR_IN` | IPv4-адрес + порт |
| `Sockaddr_in6` | `SOCKADDR_IN6` | IPv6-адрес + порт + flow info + scope |
| `Sockaddr_storage` | `SOCKADDR_STORAGE` | Достаточно большая структура для любого адреса; безопасна для `cast` к конкретному типу |
| `InAddr` | `IN_ADDR` | Сырой IPv4-адрес как `uint32` |
| `In6_addr` | `IN6_ADDR` | Сырой IPv6-адрес как массив из 16 байт |
| `AddrInfo` | `ADDRINFOA` | Узел связного списка из `getaddrinfo`; хранит разрешённый адрес + метаданные |
| `TFdSet` | custom | Набор сокетов для `select`; максимум `FD_SETSIZE` (64) сокетов |
| `Timeval` | `struct timeval` | Таймаут для `select`: `tv_sec` секунд + `tv_usec` микросекунд |
| `WSAData` | `WSADATA` | Информация о версии Winsock, заполняемая `wsaStartup` |
| `TWSABuf` | `WSABUF` | Дескриптор буфера scatter/gather: `len` + указатель `buf` |
| `Servent` | `servent` | Запись службы (имя, порт, протокол) |
| `Hostent` | `hostent` | Устаревшая запись хоста (имя, список адресов) |

---

### Типы указателей на WSA-функции-расширения

Получаются в runtime через `WSAIoctl` + `SIO_GET_EXTENSION_FUNCTION_POINTER`:

| Тип | Расширение | Назначение |
|---|---|---|
| `WSAPROC_ACCEPTEX` | `AcceptEx` | Принять соединение и получить первые данные за один вызов |
| `WSAPROC_CONNECTEX` | `ConnectEx` | Подключиться и отправить первые данные за один вызов (требует привязанного сокета) |
| `WSAPROC_GETACCEPTEXSOCKADDRS` | `GetAcceptExSockaddrs` | Разобрать локальный/удалённый адрес из буфера `AcceptEx` |

---

### Типы безопасности

| Тип | Назначение |
|---|---|
| `SID_IDENTIFIER_AUTHORITY` | 6-байтный компонент полномочий идентификатора безопасности |
| `SID` | Объект идентификатора безопасности (переменная длина; на практике используйте `PSID`) |
| `PSID` | `ptr SID` |

---

### Типы волокон и обратных вызовов

```nim
LPFIBER_START_ROUTINE* = proc(param: pointer) {.stdcall.}
WAITORTIMERCALLBACK* = proc(para1: pointer, para2: int32) {.stdcall.}
```

`LPFIBER_START_ROUTINE` — точка входа для волокна, создаваемого через `CreateFiber`. `WAITORTIMERCALLBACK` — обратный вызов, регистрируемый через `registerWaitForSingleObject`.

---

## Константы

### Управление процессами и потоками

| Константа | Значение | Смысл |
|---|---|---|
| `STARTF_USESHOWWINDOW` | 1 | Поле `wShowWindow` в `STARTUPINFO` действительно |
| `STARTF_USESTDHANDLES` | 256 | Поля `hStdInput`/`hStdOutput`/`hStdError` в `STARTUPINFO` действительны |
| `HIGH_PRIORITY_CLASS` | 128 | Высокий приоритет планирования для `createProcessW` |
| `IDLE_PRIORITY_CLASS` | 64 | Приоритет фонового/простаивающего процесса |
| `NORMAL_PRIORITY_CLASS` | 32 | Приоритет по умолчанию |
| `REALTIME_PRIORITY_CLASS` | 256 | Реального времени (использовать с осторожностью) |
| `DETACHED_PROCESS` | 8 | Новый процесс без консоли |
| `CREATE_NO_WINDOW` | 0x08000000 | Не создавать видимое консольное окно |
| `CREATE_UNICODE_ENVIRONMENT` | 1024 | Блок окружения в формате Unicode |
| `SW_SHOWNORMAL` | 1 | Показать и активировать окно в нормальном режиме |

### Возвращаемые значения ожидания

| Константа | Значение | Смысл |
|---|---|---|
| `WAIT_OBJECT_0` | 0 | Объект был сигнализирован |
| `WAIT_TIMEOUT` | 0x102 | Истёк таймаут |
| `WAIT_FAILED` | 0xFFFFFFFF | Ошибка — вызвать `getLastError()` |
| `INFINITE` | -1 | Ждать бесконечно |
| `STILL_ACTIVE` | 0x103 | Процесс ещё работает (из `getExitCodeProcess`) |
| `MAXIMUM_WAIT_OBJECTS` | 64 | Максимум дескрипторов для `waitForMultipleObjects` |

### Константы стандартных дескрипторов

| Константа | Значение | Смысл |
|---|---|---|
| `STD_INPUT_HANDLE` | -10 | Стандартный ввод консоли |
| `STD_OUTPUT_HANDLE` | -11 | Стандартный вывод консоли |
| `STD_ERROR_HANDLE` | -12 | Стандартный поток ошибок консоли |
| `INVALID_HANDLE_VALUE` | `Handle(-1)` | Сигнальное значение неудачных операций с дескриптором |

### Константы каналов

| Константа | Смысл |
|---|---|
| `PIPE_ACCESS_DUPLEX` | Именованный канал двунаправленный |
| `PIPE_ACCESS_INBOUND` | Данные текут только от клиента к серверу |
| `PIPE_ACCESS_OUTBOUND` | Данные текут только от сервера к клиенту |
| `PIPE_NOWAIT` | Неблокирующий режим (устарело; предпочтительно overlapped I/O) |
| `SYNCHRONIZE` | Право доступа, необходимое для ожидания объекта |
| `HANDLE_FLAG_INHERIT` | Дескриптор наследуется дочерними процессами |

### Атрибуты файлов (`FILE_ATTRIBUTE_*`)

| Константа | Смысл |
|---|---|
| `FILE_ATTRIBUTE_READONLY` | Только для чтения |
| `FILE_ATTRIBUTE_HIDDEN` | Скрытый в Проводнике |
| `FILE_ATTRIBUTE_SYSTEM` | Системный файл ОС |
| `FILE_ATTRIBUTE_DIRECTORY` | Элемент является каталогом |
| `FILE_ATTRIBUTE_ARCHIVE` | Отмечен для резервного копирования |
| `FILE_ATTRIBUTE_NORMAL` | Никаких других атрибутов |
| `FILE_ATTRIBUTE_TEMPORARY` | Временный файл; ОС может держать в кэше |
| `FILE_ATTRIBUTE_REPARSE_POINT` | Имеет точку повторного разбора (символическая ссылка, точка монтирования) |
| `MAX_PATH` | 260 — максимальная длина пути в символах |

### Флаги файлов (`FILE_FLAG_*`)

| Константа | Смысл |
|---|---|
| `FILE_FLAG_OVERLAPPED` | Включить асинхронный (overlapped) ввод-вывод — обязателен для IOCP |
| `FILE_FLAG_WRITE_THROUGH` | Операции записи идут на диск, минуя кэш |
| `FILE_FLAG_NO_BUFFERING` | Обойти кэш ОС; требует буферов, выровненных по сектору |
| `FILE_FLAG_RANDOM_ACCESS` | Подсказка: случайный доступ (отключает упреждающее чтение) |
| `FILE_FLAG_SEQUENTIAL_SCAN` | Подсказка: последовательный доступ (включает упреждающее чтение) |
| `FILE_FLAG_DELETE_ON_CLOSE` | Файл удаляется, когда закрыты все дескрипторы |
| `FILE_FLAG_BACKUP_SEMANTICS` | Необходим для открытия каталогов или файлов с правами резервного копирования |
| `FILE_FLAG_OPEN_REPARSE_POINT` | Открыть саму точку повторного разбора, а не её цель |

### Доступ к файлу и совместное использование

| Константа | Смысл |
|---|---|
| `GENERIC_READ` | Доступ на чтение |
| `GENERIC_WRITE` | Доступ на запись |
| `GENERIC_ALL` | Полный доступ |
| `FILE_SHARE_READ` | Разрешить одновременных читателей |
| `FILE_SHARE_WRITE` | Разрешить одновременных писателей |
| `FILE_SHARE_DELETE` | Разрешить переименование/удаление при открытом файле |

### Режимы создания файла

| Константа | Смысл |
|---|---|
| `CREATE_NEW` | Ошибка, если файл уже существует |
| `CREATE_ALWAYS` | Всегда создавать; усекает существующий файл |
| `OPEN_EXISTING` | Ошибка, если файл не существует |
| `OPEN_ALWAYS` | Открыть существующий или создать новый |

### Проецирование файлов в память

| Константа | Смысл |
|---|---|
| `PAGE_READONLY` | Проецируемый регион только для чтения |
| `PAGE_READWRITE` | Проецируемый регион для чтения и записи |
| `PAGE_EXECUTE_READ` | Исполняемый + читаемый |
| `PAGE_EXECUTE_READWRITE` | Исполняемый + для чтения и записи |
| `FILE_MAP_READ` | Проецировать с правами чтения |
| `FILE_MAP_WRITE` | Проецировать с правами записи |

### Флаги `MoveFileEx`

| Константа | Смысл |
|---|---|
| `MOVEFILE_REPLACE_EXISTING` | Заменить назначение, если оно существует |
| `MOVEFILE_COPY_ALLOWED` | Разрешить перемещение между томами (копирование + удаление) |
| `MOVEFILE_DELAY_UNTIL_REBOOT` | Запланировать переименование при следующей перезагрузке |
| `MOVEFILE_WRITE_THROUGH` | Дождаться сброса на диск перед возвратом |

### Права доступа к процессу

| Константа | Смысл |
|---|---|
| `PROCESS_TERMINATE` | Право завершить процесс |
| `PROCESS_CREATE_THREAD` | Право создавать потоки |
| `PROCESS_VM_READ` / `PROCESS_VM_WRITE` | Право читать/записывать память процесса |
| `PROCESS_QUERY_INFORMATION` | Право запрашивать информацию о процессе |
| `PROCESS_SUSPEND_RESUME` | Право приостанавливать/возобновлять |

### Флаги пула потоков (`WT_*`)

| Константа | Смысл |
|---|---|
| `WT_EXECUTEDEFAULT` | Использовать поток пула по умолчанию |
| `WT_EXECUTEONLYONCE` | Обратный вызов срабатывает один раз, затем авторегистрация отменяется |
| `WT_EXECUTELONGFUNCTION` | Подсказка: обратный вызов выполняется долго |
| `WT_EXECUTEINPERSISTENTTHREAD` | Использовать выделенный постоянный поток |

### Флаги сетевых событий сокета (`FD_*`)

Биты для `wsaEventSelect`. Комбинируйте через `or`:

| Константа | Смысл |
|---|---|
| `FD_READ` | Данные доступны для чтения |
| `FD_WRITE` | В буфере отправки есть место |
| `FD_ACCEPT` | Входящее соединение готово |
| `FD_CONNECT` | Соединение установлено |
| `FD_CLOSE` | Уведомление о корректном закрытии |
| `FD_OOB` | Внеполосные данные |
| `FD_ALL_EVENTS` | Все вышеперечисленные |

### Коды ошибок Winsock

| Константа | Значение | Смысл |
|---|---|---|
| `WSAEWOULDBLOCK` | 10035 | Неблокирующая операция заблокировалась бы |
| `WSAEINPROGRESS` | 10036 | Блокирующая операция выполняется |
| `WSAEINTR` | 10004 | Блокирующий вызов прерван |
| `WSAECONNRESET` | 10054 | Соединение сброшено удалённой стороной |
| `WSAECONNABORTED` | 10053 | Соединение прервано |
| `WSAEADDRINUSE` | 10048 | Адрес уже используется |
| `WSAETIMEDOUT` | 10060 | Соединение по таймауту |
| `WSAESHUTDOWN` | 10058 | Отправка невозможна после shutdown |
| `WSAEDISCON` | 10101 | Корректное завершение в процессе |
| `WSAENETRESET` | 10052 | Сеть сброшена |
| `WSANOTINITIALISED` | 10093 | `wsaStartup` не был вызван |
| `WSAENOTSOCK` | 10038 | Не является сокетом |
| `ERROR_IO_PENDING` | 997 | Операция overlapped ввода-вывода ожидает завершения (также `WSA_IO_PENDING`) |
| `STATUS_PENDING` | 0x103 | Операция ввода-вывода ещё не завершена |

### Коды ошибок Win32

| Константа | Значение | Смысл |
|---|---|---|
| `NO_ERROR` | 0 | Успех |
| `ERROR_FILE_NOT_FOUND` | 2 | Файл не найден |
| `ERROR_PATH_NOT_FOUND` | 3 | Путь не найден |
| `ERROR_ACCESS_DENIED` | 5 | Доступ запрещён |
| `ERROR_NO_MORE_FILES` | 18 | Конец перечисления каталога |
| `ERROR_LOCK_VIOLATION` | 33 | Область файла заблокирована |
| `ERROR_HANDLE_EOF` | 38 | Чтение за концом файла |
| `ERROR_FILE_EXISTS` | 80 | Файл уже существует |
| `ERROR_BAD_ARGUMENTS` | 165 | Неверный аргумент |
| `ERROR_NETNAME_DELETED` | 64 | Сетевое имя больше недоступно |
| `INVALID_FILE_SIZE` | -1 | Возвращается `getFileSize` при ошибке |

### IOC-коды Winsock

| Константа | Смысл |
|---|---|
| `IOC_OUT` | Выходной параметр заполняется вызовом |
| `IOC_IN` | Входной параметр потребляется вызовом |
| `IOC_INOUT` | Оба |
| `IOC_WS2` | Специфичный для Winsock2 ioctl |
| `SIO_GET_EXTENSION_FUNCTION_POINTER` | Запрос для `AcceptEx`, `ConnectEx` и др. |
| `SO_UPDATE_ACCEPT_CONTEXT` | Требуется после `AcceptEx` для завершения наследования сокета |

### Семейства адресов сокетов

| Константа | Значение | Смысл |
|---|---|---|
| `AF_UNSPEC` | 0 | Не задано |
| `AF_INET` | 2 | IPv4 |
| `AF_INET6` | 23 | IPv6 |

### Прочие константы IOCP / сокетов

| Константа | Смысл |
|---|---|
| `AI_V4MAPPED` | Возвращать IPv4-mapped IPv6-адреса, если IPv6 не найден |
| `INADDR_ANY` | Привязка ко всем локальным IPv4-интерфейсам |
| `INADDR_LOOPBACK` | IPv4-петля (127.0.0.1) |
| `INADDR_BROADCAST` | Ограниченная широковещательная рассылка |
| `FD_SETSIZE` | Максимум сокетов в `TFdSet` (64 на Windows) |
| `MSG_PEEK` | Просмотреть входящие данные без их потребления |

### Константы безопасности

| Константа | Смысл |
|---|---|
| `SECURITY_NT_AUTHORITY` | Идентификатор полномочий для стандартных Windows SID |
| `SECURITY_BUILTIN_DOMAIN_RID` | 32 — встроенный домен |
| `DOMAIN_ALIAS_RID_ADMINS` | 544 — псевдоним Администраторов |

### Константы волокон

| Константа | Значение | Смысл |
|---|---|---|
| `FIBER_FLAG_FLOAT_SWITCH` | 0x01 | Сохранять/восстанавливать состояние операций с плавающей точкой при переключении волокон |

---

## Вспомогательные функции Nim

Эти чистые Nim-функции определены в `winlean.nim`, они не являются импортами из Win32.

### `isSuccess`

```nim
proc isSuccess*(a: WINBOOL): bool {.inline.}
```

Возвращает `a != 0`. Существует для документирования намерения: Windows `BOOL` ненулевое при успехе, что противоположно POSIX. Используйте `isSuccess(result)` вместо `result != 0` при проверке результатов Win32-вызовов.

```nim
let ok = closeHandle(h)
if not ok.isSuccess:
  raise newException(OSError, "CloseHandle failed: " & $getLastError())
```

---

### `rdFileTime`

```nim
proc rdFileTime*(f: FILETIME): int64
```

Объединяет два `DWORD`-поля `FILETIME` в единый `int64`, представляющий 100-наносекундные интервалы с 1 января 1601 года. Объединение использует беззнаковые 32-битные преобразования во избежание знакового расширения младшего слова.

```nim
var info: BY_HANDLE_FILE_INFORMATION
discard getFileInformationByHandle(h, addr info)
let writeTime = rdFileTime(info.ftLastWriteTime)
```

---

### `rdFileSize`

```nim
proc rdFileSize*(f: WIN32_FIND_DATA): int64
```

Объединяет `nFileSizeLow` и `nFileSizeHigh` из `WIN32_FIND_DATA` в единый `int64` — размер файла. Та же техника беззнаковых преобразований, что и в `rdFileTime`.

---

### `toFILETIME`

```nim
proc toFILETIME*(t: int64): FILETIME
```

Преобразует 64-битную метку времени Windows (100-нс интервалы с 1601 года) обратно в структуру `FILETIME`. Полезно при вызове `setFileTime`.

```nim
let ft = toFILETIME(someTimestamp)
discard setFileTime(h, nil, nil, ft.addr)
```

---

### `FD_ISSET` / `FD_SET` / `FD_ZERO`

```nim
proc FD_ISSET*(socket: SocketHandle, set: var TFdSet): cint
proc FD_SET*(socket: SocketHandle, s: var TFdSet)
proc FD_ZERO*(s: var TFdSet)
```

Стандартные макросы манипуляции `fd_set` из BSD-сокетов, реализованные как Nim-процедуры. Windows не экспортирует их из заголовка способом, доступным для Nim, поэтому они реализованы здесь заново.

- `FD_ZERO` очищает набор (устанавливает `fd_count` в 0).
- `FD_SET` добавляет сокет в набор (до `FD_SETSIZE` сокетов).
- `FD_ISSET` проверяет, находится ли сокет в наборе (оборачивает `__WSAFDIsSet`).

---

### `hasOverlappedIoCompleted`

```nim
template hasOverlappedIoCompleted*(lpOverlapped): bool =
  (cast[uint](lpOverlapped.internal) != STATUS_PENDING)
```

Проверяет завершение операции overlapped ввода-вывода, инспектируя поле `internal`. Аналог C-макроса `HasOverlappedIoCompleted`. Возвращает `true`, когда операция завершилась (ОС записывает финальный статус в `internal`).

---

### `WSAIORW`

```nim
template WSAIORW*(x, y): untyped = (IOC_INOUT or x or y)
```

Вычисляет двунаправленный Winsock-код ioctl. Используется для построения `SIO_GET_EXTENSION_FUNCTION_POINTER`.

---

### `inet_ntop` (обёртка совместимости)

```nim
proc inet_ntop*(family: cint, paddr: pointer, pStringBuffer: cstring,
                stringBufSize: int32): cstring {.stdcall.}
```

Преобразует двоичный сетевой адрес в текстовое представление (например, `"192.168.1.1"`). На Windows Vista и новее делегирует нативному `inet_ntop` из `ws2_32.dll`. На Windows XP использует `inet_ntop_emulated`, основанный на `WSAAddressToStringA`. Указатель на настоящую функцию разрешается при загрузке модуля через `dynlib.loadLib`.

Параметры соответствуют POSIX: `family` — `AF_INET` или `AF_INET6`, `paddr` указывает на `InAddr` или `In6_addr`, `pStringBuffer` принимает результат, `stringBufSize` — ёмкость буфера. При ошибке возвращает `nil`.

---

## Функции Windows API

### Обработка ошибок

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `getLastError` | `(): int32` | kernel32 | Получить последний код ошибки Win32 для вызывающего потока |
| `setLastError` | `(error: int32)` | kernel32 | Установить последний код ошибки (полезно в эмуляционном коде) |
| `formatMessageW` | `(dwFlags, lpSource, dwMessageId, dwLanguageId: int32, lpBuffer: pointer, nSize: int32, arguments: pointer): int32` | kernel32 | Форматировать сообщение об ошибке по коду; передавайте `dwFlags = 0x1300` для типичного использования |
| `localFree` | `(p: pointer)` | kernel32 | Освободить буфер, выделенный `formatMessageW` при использовании `FORMAT_MESSAGE_ALLOCATE_BUFFER` |
| `wsaGetLastError` | `(): cint` | ws2_32 | Последняя ошибка Winsock (эквивалент `getLastError` для сокетных операций) |

---

### Управление дескрипторами

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `closeHandle` | `(hObject: Handle): WINBOOL` | kernel32 | Закрыть любой дескриптор объекта ядра; должен вызываться для каждого дескриптора, возвращённого kernel32 |
| `duplicateHandle` | `(hSourceProcessHandle, hSourceHandle, hTargetProcessHandle: Handle, lpTargetHandle: ptr Handle, dwDesiredAccess: DWORD, bInheritHandle: WINBOOL, dwOptions: DWORD): WINBOOL` | kernel32 | Дублировать дескриптор, возможно в другой процесс; используйте `DUPLICATE_SAME_ACCESS` для копирования прав |
| `getHandleInformation` | `(hObject: Handle, lpdwFlags: ptr DWORD): WINBOOL` | kernel32 | Запросить флаги дескриптора (например, `HANDLE_FLAG_INHERIT`) |
| `setHandleInformation` | `(hObject: Handle, dwMask: DWORD, dwFlags: DWORD): WINBOOL` | kernel32 | Установить флаги дескриптора |
| `getCurrentProcess` | `(): Handle` | kernel32 | Вернуть псевдодескриптор текущего процесса; не требует закрытия |

---

### Управление процессами

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `createProcessW` | `(lpApplicationName, lpCommandLine: WideCString, ..., lpStartupInfo: var STARTUPINFO, lpProcessInformation: var PROCESS_INFORMATION): WINBOOL` | kernel32 | Запустить новый процесс; при успехе заполняет `PROCESS_INFORMATION` |
| `terminateProcess` | `(hProcess: Handle, uExitCode: int): WINBOOL` | kernel32 | Принудительно завершить процесс с указанным кодом выхода |
| `getExitCodeProcess` | `(hProcess: Handle, lpExitCode: var int32): WINBOOL` | kernel32 | Получить код выхода; возвращает `STILL_ACTIVE`, если процесс ещё работает |
| `openProcess` | `(dwDesiredAccess: DWORD, bInheritHandle: WINBOOL, dwProcessId: DWORD): Handle` | kernel32 | Открыть дескриптор существующего процесса по PID |
| `suspendThread` | `(hThread: Handle): int32` | kernel32 | Приостановить поток; возвращает предыдущее число приостановок |
| `resumeThread` | `(hThread: Handle): int32` | kernel32 | Возобновить поток; возвращает предыдущее число приостановок |
| `getSystemTimes` | `(lpIdleTime, lpKernelTime, lpUserTime: var FILETIME): WINBOOL` | kernel32 | Получить системное распределение процессорного времени |
| `getProcessTimes` | `(hProcess: Handle; lpCreationTime, lpExitTime, lpKernelTime, lpUserTime: var FILETIME): WINBOOL` | kernel32 | Получить процессорное время конкретного процесса |

---

### Ожидание и синхронизация

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `waitForSingleObject` | `(hHandle: Handle, dwMilliseconds: int32): int32` | kernel32 | Блокировать до сигнала дескриптора или истечения таймаута; возвращает `WAIT_OBJECT_0`, `WAIT_TIMEOUT` или `WAIT_FAILED` |
| `waitForMultipleObjects` | `(nCount: DWORD, lpHandles: PWOHandleArray, bWaitAll: WINBOOL, dwMilliseconds: DWORD): DWORD` | kernel32 | Ждать до 64 дескрипторов одновременно |
| `createEvent` | `(lpEventAttributes: ptr SECURITY_ATTRIBUTES, bManualReset: DWORD, bInitialState: DWORD, lpName: ptr Utf16Char): Handle` | kernel32 | Создать событие с ручным или автоматическим сбросом |
| `setEvent` | `(hEvent: Handle): cint` | kernel32 | Сигнализировать событие |
| `registerWaitForSingleObject` | `(phNewWaitObject: ptr Handle, hObject: Handle, Callback: WAITORTIMERCALLBACK, Context: pointer, dwMilliseconds: ULONG, dwFlags: ULONG): bool` | kernel32 | Зарегистрировать обратный вызов, срабатывающий при сигнале дескриптора (пул потоков) |
| `unregisterWait` | `(WaitHandle: Handle): DWORD` | kernel32 | Отменить регистрацию из `registerWaitForSingleObject` |
| `postQueuedCompletionStatus` | `(CompletionPort: Handle, dwNumberOfBytesTransferred: DWORD, dwCompletionKey: ULONG_PTR, lpOverlapped: pointer): bool` | kernel32 | Поместить синтетический пакет завершения в порт IOCP |
| `sleep` | `(dwMilliseconds: int32)` | kernel32 | Приостановить вызывающий поток |

---

### Порты завершения ввода-вывода (IOCP)

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `createIoCompletionPort` | `(FileHandle: Handle, ExistingCompletionPort: Handle, CompletionKey: ULONG_PTR, NumberOfConcurrentThreads: DWORD): Handle` | kernel32 | Создать или ассоциировать дескриптор с портом IOCP; вызывайте с `FileHandle = INVALID_HANDLE_VALUE` и `ExistingCompletionPort = 0` для создания нового порта |
| `getQueuedCompletionStatus` | `(CompletionPort: Handle, lpNumberOfBytesTransferred: PDWORD, lpCompletionKey: PULONG_PTR, lpOverlapped: ptr POVERLAPPED, dwMilliseconds: DWORD): WINBOOL` | kernel32 | Извлечь уведомление о завершённом вводе-выводе; `lpOverlapped` равно `nil` при таймауте |
| `getOverlappedResult` | `(hFile: Handle, lpOverlapped: POVERLAPPED, lpNumberOfBytesTransferred: var DWORD, bWait: WINBOOL): WINBOOL` | kernel32 | Получить результат завершённой операции overlapped ввода-вывода |

---

### Файловый ввод-вывод

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `createFileW` | `(lpFileName: WideCString, dwDesiredAccess, dwShareMode: DWORD, lpSecurityAttributes: pointer, dwCreationDisposition, dwFlagsAndAttributes: DWORD, hTemplateFile: Handle): Handle` | kernel32 | Открыть или создать файл, каталог, канал или устройство |
| `createFileA` | аналогично, `lpFileName: cstring` | kernel32 | ANSI-версия `createFileW` |
| `deleteFileW` / `deleteFileA` | `(pathName: WideCString/cstring): int32` | kernel32 | Удалить файл |
| `readFile` | `(hFile: Handle, buffer: pointer, nNumberOfBytesToRead: int32, lpNumberOfBytesRead: ptr int32, lpOverlapped: pointer): WINBOOL` | kernel32 | Читать из файла или канала; передавайте `nil` для `lpOverlapped` при синхронном вводе-выводе |
| `writeFile` | `(hFile: Handle, buffer: pointer, nNumberOfBytesToWrite: int32, lpNumberOfBytesWritten: ptr int32, lpOverlapped: pointer): WINBOOL` | kernel32 | Писать в файл или канал |
| `flushFileBuffers` | `(hFile: Handle): WINBOOL` | kernel32 | Сбросить кэш ОС на диск |
| `setEndOfFile` | `(hFile: Handle): WINBOOL` | kernel32 | Усечь или расширить файл до текущей позиции указателя |
| `setFilePointer` | `(hFile: Handle, lDistanceToMove: LONG, lpDistanceToMoveHigh: ptr LONG, dwMoveMethod: DWORD): DWORD` | kernel32 | Переместить файловый указатель; `dwMoveMethod` — `FILE_BEGIN`, `FILE_CURRENT` или `FILE_END` |
| `getFileSize` | `(hFile: Handle, lpFileSizeHigh: ptr DWORD): DWORD` | kernel32 | Получить размер файла; объедините возвращённый `DWORD` с `*lpFileSizeHigh` для больших файлов |
| `getFileInformationByHandle` | `(hFile: Handle, lpFileInformation: ptr BY_HANDLE_FILE_INFORMATION): WINBOOL` | kernel32 | Получить подробные метаданные, включая время создания/изменения и число жёстких ссылок |
| `setFileTime` | `(hFile: Handle, lpCreationTime, lpLastAccessTime, lpLastWriteTime: LPFILETIME): WINBOOL` | kernel32 | Установить временные метки файла; передавайте `nil` для неизменяемых |
| `setFileAttributesW` | `(lpFileName: WideCString, dwFileAttributes: int32): WINBOOL` | kernel32 | Изменить атрибуты файла |
| `getFileAttributesW` | `(lpFileName: WideCString): int32` | kernel32 | Запросить атрибуты файла; возвращает `-1` при ошибке |

---

### Операции с каталогами и путями

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `createDirectoryW` | `(pathName: WideCString, security: pointer = nil): int32` | kernel32 | Создать один каталог (нерекурсивно) |
| `removeDirectoryW` | `(lpPathName: WideCString): int32` | kernel32 | Удалить пустой каталог |
| `getCurrentDirectoryW` | `(nBufferLength: int32, lpBuffer: WideCString): int32` | kernel32 | Получить текущий рабочий каталог |
| `setCurrentDirectoryW` | `(lpPathName: WideCString): int32` | kernel32 | Установить текущий рабочий каталог |
| `getFullPathNameW` | `(lpFileName: WideCString, nBufferLength: int32, lpBuffer: WideCString, lpFilePart: var WideCString): int32` | kernel32 | Разрешить путь до абсолютной формы |
| `findFirstFileW` | `(lpFileName: WideCString, lpFindFileData: var WIN32_FIND_DATA): Handle` | kernel32 | Начать перечисление каталога; возвращает дескриптор поиска |
| `findNextFileW` | `(hFindFile: Handle, lpFindFileData: var WIN32_FIND_DATA): int32` | kernel32 | Перейти к следующему элементу перечисления |
| `findClose` | `(hFindFile: Handle)` | kernel32 | Закрыть дескриптор поиска из `findFirstFileW` |
| `createSymbolicLinkW` | `(lpSymlinkFileName, lpTargetFileName: WideCString, flags: DWORD): int32` | kernel32 | Создать символическую ссылку (требует `SeCreateSymbolicLinkPrivilege` или режима разработчика) |
| `createHardLinkW` | `(lpFileName, lpExistingFileName: WideCString, security: pointer = nil): int32` | kernel32 | Создать жёсткую ссылку |
| `moveFileW` | `(lpExistingFileName, lpNewFileName: WideCString): WINBOOL` | kernel32 | Переименовать файл |
| `moveFileExW` | `(lpExistingFileName, lpNewFileName: WideCString, flags: DWORD): WINBOOL` | kernel32 | Переименовать с флагами (например, `MOVEFILE_REPLACE_EXISTING`) |
| `copyFileW` | `(lpExistingFileName, lpNewFileName: WideCString, bFailIfExists: WINBOOL): WINBOOL` | kernel32 | Скопировать файл |

---

### Проецирование файлов в память

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `createFileMappingW` | `(hFile: Handle, lpFileMappingAttributes: pointer, flProtect, dwMaximumSizeHigh, dwMaximumSizeLow: DWORD, lpName: pointer): Handle` | kernel32 | Создать объект отображения файла; передайте `hFile = INVALID_HANDLE_VALUE` для анонимного (ОЗУ-backed) отображения |
| `mapViewOfFileEx` | `(hFileMappingObject: Handle, dwDesiredAccess: DWORD, dwFileOffsetHigh, dwFileOffsetLow: DWORD, dwNumberOfBytesToMap: WinSizeT, lpBaseAddress: pointer): pointer` | kernel32 | Спроецировать представление файла в адресное пространство процесса |
| `unmapViewOfFile` | `(lpBaseAddress: pointer): WINBOOL` | kernel32 | Снять ранее спроецированное представление |
| `flushViewOfFile` | `(lpBaseAddress: pointer, dwNumberOfBytesToFlush: DWORD): WINBOOL` | kernel32 | Сбросить «грязные» страницы спроецированного представления на диск |

---

### Каналы

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `createPipe` | `(hReadPipe, hWritePipe: var Handle, lpPipeAttributes: var SECURITY_ATTRIBUTES, nSize: int32): WINBOOL` | kernel32 | Создать анонимный канал; использовать с `SECURITY_ATTRIBUTES.bInheritHandle` для перенаправления вввода-вывода дочернего процесса |
| `createNamedPipe` | `(lpName: WideCString, dwOpenMode, dwPipeMode, nMaxInstances, ...: int32, lpSecurityAttributes: ptr SECURITY_ATTRIBUTES): Handle` | kernel32 | Создать экземпляр сервера именованного канала |
| `peekNamedPipe` | `(hNamedPipe: Handle, ...): bool` | kernel32 | Запросить доступные байты без их потребления |

---

### Окружение и модуль

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `getEnvironmentStringsW` | `(): WideCString` | kernel32 | Получить блок окружения в виде строки с двойным нулём на конце |
| `freeEnvironmentStringsW` | `(para1: WideCString): int32` | kernel32 | Освободить блок, возвращённый `getEnvironmentStringsW` |
| `setEnvironmentVariableW` | `(lpName, lpValue: WideCString): int32` | kernel32 | Установить или удалить переменную окружения (`lpValue = nil` удаляет) |
| `getCommandLineW` | `(): WideCString` | kernel32 | Получить полную командную строку текущего процесса |
| `getModuleFileNameW` | `(handle: Handle, buf: WideCString, size: int32): int32` | kernel32 | Получить путь текущего исполняемого файла (передайте `0` для `handle`) |

---

### Стандартные дескрипторы ввода-вывода

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `getStdHandle` | `(nStdHandle: int32): Handle` | kernel32 | Получить дескриптор stdin, stdout или stderr через константы `STD_*_HANDLE` |
| `setStdHandle` | `(nStdHandle: int32, hHandle: Handle): WINBOOL` | kernel32 | Перенаправить стандартный дескриптор |
| `get_osfhandle` | `(fd: cint): Handle` | `<io.h>` | Преобразовать файловый дескриптор C-runtime в Win32 `Handle` |

---

### Время

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `getSystemTimeAsFileTime` | `(lpSystemTimeAsFileTime: var FILETIME)` | kernel32 | Получить текущее UTC-время как `FILETIME` (точность миллисекунды) |
| `getSystemTimePreciseAsFileTime` | `(lpSystemTimeAsFileTime: var FILETIME)` | kernel32 | Высокоточная версия (точность 100 нс); требует Windows 8 / Server 2012 |

---

### Версия ОС

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `getVersionExW` | `(lpVersionInfo: ptr OSVERSIONINFO): WINBOOL` | kernel32 | Получить версию Windows (устарело на Windows 10; требует манифеста) |
| `getVersionExA` | аналогично, ANSI | kernel32 | ANSI-версия |
| `getVersion` | `(): DWORD` | kernel32 | Устаревший упакованный формат версии; младший байт — мажор, старший байт младшего слова — минор |

---

### Оболочка

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `shellExecuteW` | `(hwnd: Handle, lpOperation, lpFile, lpParameters, lpDirectory: WideCString, nShowCmd: int32): Handle` | shell32 | Открыть или запустить файл через оболочку (аналог двойного щелчка); `lpOperation = "runas"` для запроса повышения привилегий |

---

### Волокна

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `CreateFiber` | `(stackSize: int, fn: LPFIBER_START_ROUTINE, param: pointer): pointer` | kernel32 | Создать волокно с заданным размером стека и точкой входа |
| `CreateFiberEx` | `(stkCommit, stkReserve: int, flags: int32, fn: LPFIBER_START_ROUTINE, param: pointer): pointer` | kernel32 | Создать волокно с раздельными commit/reserve; передайте `FIBER_FLAG_FLOAT_SWITCH` для сохранения состояния FP |
| `ConvertThreadToFiber` | `(param: pointer): pointer` | kernel32 | Преобразовать вызывающий поток в волокно |
| `ConvertThreadToFiberEx` | `(param: pointer, flags: int32): pointer` | kernel32 | Преобразовать поток в волокно с флагами |
| `DeleteFiber` | `(fiber: pointer)` | kernel32 | Удалить волокно и освободить его стек |
| `SwitchToFiber` | `(fiber: pointer)` | kernel32 | Переключить выполнение на другое волокно |
| `GetCurrentFiber` | `(): pointer` | `windows.h` (встроенная) | Вернуть указатель текущего волокна |

---

### Консоль

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `readConsoleInput` | `(hConsoleInput: Handle, lpBuffer: pointer, nLength: cint, lpNumberOfEventsRead: ptr cint): cint` | kernel32 | Читать сырые события ввода (клавиши, мышь, изменение размера окна) из консоли |

---

### Инициализация Winsock

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `wsaStartup` | `(wVersionRequired: int16, WSData: ptr WSAData): cint` | ws2_32 | Инициализировать Winsock; должен вызываться перед любой сокетной операцией; запрашивайте версию `0x0202` для Winsock 2.2 |

---

### Разрешение имён

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `getaddrinfo` | `(nodename, servname: cstring, hints: ptr AddrInfo, res: var ptr AddrInfo): cint` | ws2_32 | Разрешить хост/сервис в связный список `AddrInfo`; современная замена `gethostbyname` |
| `freeAddrInfo` | `(ai: ptr AddrInfo)` | ws2_32 | Освободить список, возвращённый `getaddrinfo` |
| `getnameinfo` | `(a1: ptr SockAddr, a2: SockLen, ..., a7: cint): cint` | ws2_32 | Обратное разрешение адреса сокета в строку хоста/сервиса |
| `gethostbyname` | `(name: cstring): ptr Hostent` | ws2_32 | Устаревший поиск хоста (только IPv4; используйте `getaddrinfo`) |
| `gethostbyaddr` | `(ip: ptr InAddr, len: cuint, theType: cint): ptr Hostent` | ws2_32 | Устаревший обратный поиск |
| `gethostname` | `(hostname: cstring, len: cint): cint` | ws2_32 | Получить имя локальной машины |
| `getservbyname` | `(name, proto: cstring): ptr Servent` | ws2_32 | Поиск службы по имени |
| `getservbyport` | `(port: cint, proto: cstring): ptr Servent` | ws2_32 | Поиск службы по номеру порта |
| `getprotobyname` | `(name: cstring): ptr Protoent` | ws2_32 | Поиск протокола по имени (например, `"tcp"`) |
| `getprotobynumber` | `(proto: cint): ptr Protoent` | ws2_32 | Поиск протокола по номеру |

---

### Основные операции с сокетами

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `socket` | `(af, typ, protocol: cint): SocketHandle` | ws2_32 | Создать сокет |
| `closesocket` | `(s: SocketHandle): cint` | ws2_32 | Закрыть сокет (не `closeHandle`) |
| `bindSocket` | `(s: SocketHandle, name: ptr SockAddr, namelen: SockLen): cint` | ws2_32 | Привязать сокет к локальному адресу; в Nim переименован из `bind` во избежание конфликта с ключевым словом |
| `listen` | `(s: SocketHandle, backlog: cint): cint` | ws2_32 | Перевести сокет в пассивный режим (серверный) |
| `accept` | `(s: SocketHandle, a: ptr SockAddr, addrlen: ptr SockLen): SocketHandle` | ws2_32 | Принять входящее соединение |
| `connect` | `(s: SocketHandle, name: ptr SockAddr, namelen: SockLen): cint` | ws2_32 | Инициировать соединение |
| `shutdown` | `(s: SocketHandle, how: cint): cint` | ws2_32 | Отключить отправку и/или получение на сокете |

---

### Ввод-вывод сокетов

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `send` | `(s: SocketHandle, buf: pointer, len, flags: cint): cint` | ws2_32 | Отправить данные; возвращает число отправленных байт или `-1` |
| `sendto` | `(s: SocketHandle, buf: pointer, len, flags: cint, to: ptr SockAddr, tolen: SockLen): cint` | ws2_32 | Отправить датаграмму на указанный адрес |
| `recv` | `(s: SocketHandle, buf: pointer, len, flags: cint): cint` | ws2_32 | Получить данные |
| `recvfrom` | `(s: SocketHandle, buf: cstring, len, flags: cint, fromm: ptr SockAddr, fromlen: ptr SockLen): cint` | ws2_32 | Получить датаграмму и записать адрес отправителя |
| `select` | `(nfds: cint, readfds, writefds, exceptfds: ptr TFdSet, timeout: ptr Timeval): cint` | ws2_32 | Мультиплексная проверка готовности к вводу-выводу; `nfds` игнорируется на Windows, но должен быть указан для POSIX-совместимости |

---

### Параметры сокетов

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `getsockname` | `(s: SocketHandle, name: ptr SockAddr, namelen: ptr SockLen): cint` | ws2_32 | Получить локальный адрес привязанного/подключённого сокета |
| `getpeername` | `(s: SocketHandle, name: ptr SockAddr, namelen: ptr SockLen): cint` | ws2_32 | Получить удалённый адрес подключённого сокета |
| `getsockopt` | `(s: SocketHandle, level, optname: cint, optval: pointer, optlen: ptr SockLen): cint` | ws2_32 | Прочитать параметр сокета |
| `setsockopt` | `(s: SocketHandle, level, optname: cint, optval: pointer, optlen: SockLen): cint` | ws2_32 | Записать параметр сокета; используйте `SOL_SOCKET` для уровня сокета, `IPPROTO_TCP` для TCP |

---

### Асинхронный ввод-вывод сокетов Winsock2

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `WSARecv` | `(s: SocketHandle, buf: ptr TWSABuf, bufCount: DWORD, bytesReceived, flags: PDWORD, lpOverlapped: POVERLAPPED, completionProc: POVERLAPPED_COMPLETION_ROUTINE): cint` | ws2_32 | Асинхронное получение с разброс-буферами |
| `WSARecvFrom` | аналогично + `name`, `namelen` | ws2_32 | Асинхронное получение датаграммы |
| `WSASend` | `(s: SocketHandle, buf: ptr TWSABuf, bufCount: DWORD, bytesSent: PDWORD, flags: DWORD, lpOverlapped: POVERLAPPED, completionProc: POVERLAPPED_COMPLETION_ROUTINE): cint` | ws2_32 | Асинхронная отправка со сбор-буферами |
| `WSASendTo` | аналогично + `name`, `namelen` | ws2_32 | Асинхронная отправка датаграммы |
| `WSAIoctl` | `(s: SocketHandle, dwIoControlCode: DWORD, ..., lpcbBytesReturned: PDWORD, lpOverlapped: POVERLAPPED, lpCompletionRoutine: POVERLAPPED_COMPLETION_ROUTINE): cint` | ws2_32 | Операция управления ввода-вывода сокета; используйте с `SIO_GET_EXTENSION_FUNCTION_POINTER` для загрузки `AcceptEx`/`ConnectEx` |

---

### WSA-события сети

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `wsaCreateEvent` | `(): Handle` | ws2_32 | Создать объект события Winsock (с ручным сбросом) |
| `wsaCloseEvent` | `(hEvent: Handle): bool` | ws2_32 | Закрыть событие Winsock |
| `wsaResetEvent` | `(hEvent: Handle): bool` | ws2_32 | Сбросить (снять сигнал) события Winsock |
| `wsaEventSelect` | `(s: SocketHandle, hEventObject: Handle, lNetworkEvents: clong): cint` | ws2_32 | Ассоциировать WSA-событие с сетевыми событиями сокета; событие сигнализируется при наступлении любого из условий `FD_*` |

---

### Устаревшее преобразование адресов

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `inet_addr` | `(cp: cstring): uint32` | ws2_32 | Разобрать строку IPv4 в десятичной нотации с точками в `uint32`; при ошибке возвращает `INADDR_NONE` |
| `inet_ntoa` | `(i: InAddr): cstring` | ws2_32 | Преобразовать IPv4-адрес в строку с точками; результат находится в статическом буфере — копируйте немедленно |

---

### Безопасность

| Функция | Сигнатура | DLL | Назначение |
|---|---|---|---|
| `allocateAndInitializeSid` | `(pIdentifierAuthority: ptr SID_IDENTIFIER_AUTHORITY, nSubAuthorityCount: BYTE, nSubAuthority0..7: DWORD, pSid: ptr PSID): WINBOOL` | Advapi32 | Выделить и инициализировать идентификатор безопасности; поддерживает до 8 суб-полномочий |
| `checkTokenMembership` | `(tokenHandle: Handle, sidToCheck: PSID, isMember: PBOOL): WINBOOL` | Advapi32 | Проверить, является ли токен членом указанного SID (например, группы Администраторов) |
| `freeSid` | `(pSid: PSID): PSID` | Advapi32 | Освободить SID, выделенный `allocateAndInitializeSid`; возвращаемое значение должно быть `nil` |

---

## Примеры использования

### Проверка прав администратора

```nim
import winlean

proc isAdmin(): bool =
  var authority = SID_IDENTIFIER_AUTHORITY(
    value: SECURITY_NT_AUTHORITY
  )
  var adminSid: PSID
  if allocateAndInitializeSid(addr authority, 2,
      SECURITY_BUILTIN_DOMAIN_RID.DWORD,
      DOMAIN_ALIAS_RID_ADMINS.DWORD,
      0, 0, 0, 0, 0, 0,
      addr adminSid) == 0:
    return false
  var isMember: WINBOOL
  discard checkTokenMembership(0.Handle, adminSid, addr isMember)
  discard freeSid(adminSid)
  return isMember.isSuccess
```

### Запуск дочернего процесса с перенаправлением stdout

```nim
import winlean

var sa = SECURITY_ATTRIBUTES(
  nLength: sizeof(SECURITY_ATTRIBUTES).int32,
  bInheritHandle: 1
)
var readPipe, writePipe: Handle
discard createPipe(readPipe, writePipe, sa, 0)
# Сделать конец чтения ненаследуемым
discard setHandleInformation(readPipe, HANDLE_FLAG_INHERIT, 0)

var si = STARTUPINFO(cb: sizeof(STARTUPINFO).int32,
  dwFlags: STARTF_USESTDHANDLES,
  hStdOutput: writePipe, hStdError: writePipe,
  hStdInput: getStdHandle(STD_INPUT_HANDLE))
var pi: PROCESS_INFORMATION

if createProcessW(nil, newWideCString("cmd.exe /c dir"),
                  nil, nil, 1, 0, nil, nil, si, pi).isSuccess:
  discard closeHandle(writePipe)  # Закрыть нашу копию для передачи EOF
  # Читать из readPipe ...
  discard closeHandle(pi.hProcess)
  discard closeHandle(pi.hThread)
discard closeHandle(readPipe)
```

### Проецирование файла в память

```nim
import winlean

let h = createFileW(newWideCString("data.bin"),
  GENERIC_READ.DWORD, FILE_SHARE_READ.DWORD, nil,
  OPEN_EXISTING.DWORD, FILE_ATTRIBUTE_NORMAL.DWORD, 0.Handle)
let mapping = createFileMappingW(h, nil, PAGE_READONLY.DWORD, 0, 0, nil)
let view = mapViewOfFileEx(mapping, FILE_MAP_READ.DWORD, 0, 0, 0, nil)
# использовать view как указатель на содержимое файла ...
discard unmapViewOfFile(view)
discard closeHandle(mapping)
discard closeHandle(h)
```

### Разрешение имени хоста через `getaddrinfo`

```nim
import winlean

var hints = AddrInfo(ai_family: AF_UNSPEC, ai_socktype: SOCK_STREAM.cint)
var res: ptr AddrInfo
if getaddrinfo("example.com", "443", addr hints, res) == 0:
  var it = res
  while it != nil:
    # использовать it.ai_addr, it.ai_addrlen ...
    it = it.ai_next
  freeAddrInfo(res)
```
