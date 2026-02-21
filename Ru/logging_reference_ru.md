# logging.nim — Справочник модуля

> **Импорт:** `import std/logging`  
> **JavaScript-бекенд:** доступен только `ConsoleLogger`; файловые логгеры и `defaultFilename` недоступны.

Модуль предоставляет лёгкую и расширяемую систему логирования. В её основе лежат три взаимосвязанных концепции:

1. **Уровни (Level)** — шкала важности, определяющая, какие сообщения имеют значение.
2. **Логгеры** — объекты, которые знают *куда* писать (консоль, файл, ротируемый файл) и *с какого порога* они реагируют.
3. **Обработчики (handlers)** — поточно-локальный реестр, позволяющий одним вызовом `log` отправить сообщение сразу всем логгерам.

---

## Архитектура

```
Ваш код
   │
   ├── logger.log(lvlInfo, "msg")   ← прямой вызов конкретного логгера
   │
   └── log(lvlInfo, "msg")          ← шаблон: рассылает ВСЕМ зарегистрированным
           │
           ├── ConsoleLogger         (stdout / stderr)
           ├── FileLogger            (обычный файл)
           └── RollingFileLogger     (файлы с ротацией)
```

Сообщение доходит до логгера только если проходит **два независимых фильтра**:
- **Глобальный фильтр** (`setLogFilter`) — поточно-локальный нижний порог, действующий на все логгеры сразу.
- **Собственный `levelThreshold` логгера** — задаётся при создании конкретного логгера.

---

## Типы

### `Level`

```nim
type Level* = enum
  lvlAll, lvlDebug, lvlInfo, lvlNotice, lvlWarn, lvlError, lvlFatal, lvlNone
```

Лестница важности, от наименьшего (`lvlAll`) к наибольшему (`lvlNone`). Каждое значение несёт устоявшийся смысл:

| Значение | Смысл |
|---|---|
| `lvlAll` | Специальное: разрешает все сообщения. Используйте как `levelThreshold`, чтобы принимать всё. |
| `lvlDebug` | Подробности для разработчика; в продакшене обычно отключено. |
| `lvlInfo` | Штатные события — нормальные, незначительные. |
| `lvlNotice` | Нормальные, но заметные события, о которых оператор должен знать. |
| `lvlWarn` | Что-то неожиданное случилось; приложение работает, но требует внимания. |
| `lvlError` | Восстановимый сбой — приложение продолжает работу в ухудшенном режиме. |
| `lvlFatal` | Невосстановимый сбой. Приложение должно завершиться. Вызов `fatal` сам **не завершает** программу — это ответственность разработчика. |
| `lvlNone` | Специальное: подавляет всё. Используйте как `levelThreshold`, чтобы полностью заглушить логгер. |

---

### `Logger`

```nim
type Logger* = ref object of RootObj
  levelThreshold*: Level
  fmtStr*: string
```

Абстрактный базовый класс всех логгеров. Для создания собственного логгера нужно унаследовать этот тип и переопределить метод `log`. Два поля присутствуют в каждом логгере и доступны для записи:

- **`levelThreshold`** — сообщения ниже этого уровня молча отбрасываются данным логгером.
- **`fmtStr`** — строка формата, которая добавляется перед каждым сообщением (см. [Строки формата](#строки-формата)).

---

### `ConsoleLogger`

```nim
type ConsoleLogger* = ref object of Logger
  useStderr*: bool
  flushThreshold*: Level
```

Выводит сообщения в терминал. На JS-бекенде использует `console.*` браузера. Два дополнительных поля:

- **`useStderr`** — при `true` пишет в `stderr` вместо `stdout`.
- **`flushThreshold`** — сообщения на этом уровне и выше немедленно сбрасывают буфер. По умолчанию сброс происходит только при `lvlError` и выше.

---

### `FileLogger` *(недоступен на JS)*

```nim
type FileLogger* = ref object of Logger
  file*: File
  flushThreshold*: Level
```

Пишет сообщения в файл. Поле `file` — это дескриптор открытого файла; при необходимости его можно читать или заменять напрямую.

---

### `RollingFileLogger` *(недоступен на JS)*

```nim
type RollingFileLogger* = ref object of FileLogger
  # (поля ротации — внутренние)
```

Расширяет `FileLogger` автоматической ротацией: когда текущий файл набирает `maxLines` строк, он переименовывается в `<имя>.1` (бывший `.1` становится `.2` и т. д.), а вместо него открывается новый пустой файл. Это предотвращает неограниченный рост лог-файлов.

---

## Константы

### `LevelNames`

```nim
const LevelNames*: array[Level, string] =
  ["DEBUG", "DEBUG", "INFO", "NOTICE", "WARN", "ERROR", "FATAL", "NONE"]
```

Таблица соответствия `Level` → строковое имя. `lvlAll` отображается как `"DEBUG"` — если все уровни активны, текущее отображаемое имя равно `DEBUG`. Используется внутри подстановки строк формата и доступна для пользовательских логгеров.

---

### `defaultFmtStr`

```nim
const defaultFmtStr* = "$levelname "
```

Строка формата, используемая по умолчанию. Даёт вывод вида `INFO некоторое сообщение`. Пробел после `$levelname` отделяет префикс от тела сообщения.

---

### `verboseFmtStr`

```nim
const verboseFmtStr* = "$levelid, [$datetime] -- $appname: "
```

Расширенный пресет: однобуквенный идентификатор уровня, полная метка времени и имя приложения. Пример вывода: `I, [2024-01-15T10:32:04] -- myapp: сообщение`. Передаётся как аргумент `fmtStr` при создании любого логгера.

---

## Строки формата

Строки формата — это шаблоны, в которых плейсхолдеры вида `$переменная` заменяются актуальными значениями перед добавлением пользовательского сообщения. Все символы вне плейсхолдеров копируются без изменений.

| Плейсхолдер | Заменяется на |
|---|---|
| `$date` | Текущая дата (`YYYY-MM-DD`) |
| `$time` | Текущее время (`HH:MM:SS`) |
| `$datetime` | `$date` + `T` + `$time` |
| `$app` | Полный путь к исполняемому файлу (`os.getAppFilename()`) |
| `$appname` | Имя исполняемого файла без расширения и директории |
| `$appdir` | Директория, содержащая исполняемый файл |
| `$levelid` | Первый символ имени уровня (`D`, `I`, `N`, `W`, `E`, `F`) |
| `$levelname` | Полное имя уровня (`DEBUG`, `INFO`, `NOTICE`, `WARN`, `ERROR`, `FATAL`) |

`$app`, `$appname` и `$appdir` недоступны на JS-бекенде.

```nim
# Пользовательский префикс: [10:32:04] - INFO:
var logger = newConsoleLogger(fmtStr = "[$time] - $levelname: ")
logger.log(lvlInfo, "сервер запущен на порту 8080")
# Вывод: [10:32:04] - INFO: сервер запущен на порту 8080
```

---

## Создание логгеров

### `newConsoleLogger`

```nim
proc newConsoleLogger*(
    levelThreshold = lvlAll,
    fmtStr = defaultFmtStr,
    useStderr = false,
    flushThreshold = defaultFlushThreshold
): ConsoleLogger
```

**Что делает.** Создаёт `ConsoleLogger`. По умолчанию принимает все сообщения, использует стандартную строку формата, пишет в `stdout` и сбрасывает буфер только при `lvlError` и выше.

Это наиболее распространённая отправная точка для любой программы на Nim, которой нужно логирование. В продакшене `levelThreshold` обычно поднимают до `lvlInfo` или `lvlWarn`.

```nim
# Минимальный — всё в stdout
var log = newConsoleLogger()

# Только предупреждения и выше — в stderr, всегда сбрасывать буфер
var errLog = newConsoleLogger(
    levelThreshold = lvlWarn,
    useStderr = true,
    flushThreshold = lvlAll
)

# С подробными метками времени
var verboseLog = newConsoleLogger(fmtStr = verboseFmtStr)
```

---

### `newFileLogger` (из дескриптора файла) *(не JS)*

```nim
proc newFileLogger*(
    file: File,
    levelThreshold = lvlAll,
    fmtStr = defaultFmtStr,
    flushThreshold = defaultFlushThreshold
): FileLogger
```

**Что делает.** Создаёт `FileLogger`, оборачивающий уже открытый дескриптор `File`. Используйте этот вариант, когда нужен полный контроль над тем, как открыт файл — например, чтобы разделить дескриптор с другими частями программы или использовать нестандартный режим.

```nim
let f = open("app.log", fmWrite)
var logger = newFileLogger(f, levelThreshold = lvlInfo)
```

---

### `newFileLogger` (по имени файла) *(не JS)*

```nim
proc newFileLogger*(
    filename = defaultFilename(),
    mode: FileMode = fmAppend,
    levelThreshold = lvlAll,
    fmtStr = defaultFmtStr,
    bufSize: int = -1,
    flushThreshold = defaultFlushThreshold
): FileLogger
```

**Что делает.** Более удобный вариант — файл открывается внутри процедуры. По умолчанию файл открывается для дозаписи (`fmAppend`), поэтому перезапуск приложения не затирает предыдущие записи.

`bufSize` управляет буфером ввода-вывода:
- `-1` — использовать умолчание ОС/рантайма (обычно 4–8 КБ).
- `0` — без буферизации: каждая запись сразу уходит в ОС. Надёжно, но медленно.
- `> 0` — фиксированный размер буфера в байтах.

```nim
# Дозапись в "app.log", только ошибки
var errLog = newFileLogger("app.log", levelThreshold = lvlError)

# Перезапись при каждом запуске, подробный формат, без буферизации
var devLog = newFileLogger("dev.log", mode = fmWrite,
                           fmtStr = verboseFmtStr, bufSize = 0)
```

---

### `newRollingFileLogger` *(не JS)*

```nim
proc newRollingFileLogger*(
    filename = defaultFilename(),
    mode: FileMode = fmReadWrite,
    levelThreshold = lvlAll,
    fmtStr = defaultFmtStr,
    maxLines: Positive = 1000,
    bufSize: int = -1,
    flushThreshold = defaultFlushThreshold
): RollingFileLogger
```

**Что делает.** Создаёт `RollingFileLogger`. Когда текущий файл набирает `maxLines` строк, выполняется ротация: `app.log` → `app.log.1`, прежний `app.log.1` → `app.log.2` и т. д. Затем открывается новый пустой `app.log`. Старые ротированные файлы никогда не удаляются автоматически.

Правильный выбор для долгоживущих сервисов, где важно ограничить занимаемое место на диске без ручного управления логами.

```nim
# По умолчанию: ротация каждые 1000 строк
var logger = newRollingFileLogger("service.log")

# Ротация каждые 200 строк, только ошибки
var errorLog = newRollingFileLogger(
    "errors.log",
    levelThreshold = lvlError,
    maxLines = 200
)
```

---

### `defaultFilename` *(не JS)*

```nim
proc defaultFilename*(): string
```

**Что делает.** Возвращает путь к лог-файлу по умолчанию, используемый `newFileLogger` и `newRollingFileLogger`, когда аргумент `filename` не задан. Формируется как путь к запущенному исполняемому файлу с заменой расширения на `.log`. Например, если исполняемый файл — `/usr/bin/myapp`, функция вернёт `/usr/bin/myapp.log`.

```nim
echo defaultFilename()   # напр. /home/user/project/myapp.log
```

---

## Методы логирования

У каждого типа логгера есть метод `log`. Вызов этого метода напрямую адресует **только данный конкретный логгер** — список зарегистрированных обработчиков при этом не задействуется. Это важно, когда нужно направить конкретное сообщение в конкретное место без широковещательной рассылки.

### `log` (ConsoleLogger)

```nim
method log*(logger: ConsoleLogger, level: Level, args: varargs[string, `$`])
```

**Что делает.** Форматирует и выводит сообщение в консоль, если `level` проходит оба фильтра — глобальный и `levelThreshold` данного логгера. Несколько `args` конкатенируются. Можно передавать любые значения, для которых определён `$` — преобразование выполняется автоматически.

На JS-бекенде для каждого уровня выбирается соответствующий метод браузера (`console.debug`, `console.warn` и т. д.).

```nim
var log = newConsoleLogger()
log.log(lvlInfo,  "Сервер запущен на порту ", 8080)
log.log(lvlError, "Не удалось открыть файл: ", filename)
```

---

### `log` (FileLogger) *(не JS)*

```nim
method log*(logger: FileLogger, level: Level, args: varargs[string, `$`])
```

**Что делает.** Та же семантика, что у `ConsoleLogger`, но запись идёт в обёрнутый файл. Немедленно сбрасывает буфер, если `level >= logger.flushThreshold`.

```nim
var log = newFileLogger("app.log")
log.log(lvlWarn, "Диск заполнен более чем на 80%")
```

---

### `log` (RollingFileLogger) *(не JS)*

```nim
method log*(logger: RollingFileLogger, level: Level, args: varargs[string, `$`])
```

**Что делает.** То же самое, что `FileLogger.log`, но перед записью проверяет, не нужна ли ротация. Если текущий файл достиг `maxLines`, ротация выполняется прозрачно, после чего сообщение пишется уже в новый файл.

```nim
var log = newRollingFileLogger("rolling.log", maxLines = 500)
log.log(lvlInfo, "тик ", tickCount)   # ротируется незаметно по мере надобности
```

---

### `log` (базовый Logger)

```nim
method log*(logger: Logger, level: Level, args: varargs[string, `$`])
```

**Что делает.** Реализация базового класса — не делает ничего. Существует для того, чтобы пользовательские логгеры могли наследовать `Logger` и переопределять только этот метод. Напрямую вызывать его на экземпляре простого `Logger` не имеет смысла.

```nim
type SyslogLogger = ref object of Logger

method log*(logger: SyslogLogger, level: Level, args: varargs[string, `$`]) =
  if level >= logger.levelThreshold:
    sendToSyslog(substituteLog(logger.fmtStr, level, args))
```

---

## Реестр обработчиков

Система обработчиков — это механизм широковещательного логирования. Регистрация логгера через `addHandler` означает, что каждый последующий вызов шаблона `log` (и специализированных сокращений) в данном потоке будет автоматически диспетчеризован в этот логгер.

> **Многопоточность:** и список обработчиков, и глобальный фильтр являются **поточно-локальными переменными**. Если логирование нужно в нескольких потоках, вызывайте `addHandler` и `setLogFilter` отдельно в каждом потоке.

### `addHandler`

```nim
proc addHandler*(handler: Logger)
```

**Что делает.** Добавляет `handler` в список зарегистрированных логгеров текущего потока. После этого каждый вызов шаблона `log(...)` в данном потоке будет диспетчеризован в `handler` (с учётом фильтрации по уровню).

Если один и тот же логгер добавлен несколько раз, он будет получать каждое сообщение несколько раз — дедупликация не выполняется.

```nim
var consoleLog = newConsoleLogger()
var fileLog    = newFileLogger("app.log", levelThreshold = lvlWarn)

addHandler(consoleLog)   # всё видно в терминале
addHandler(fileLog)      # предупреждения и выше — на диск
```

---

### `removeHandler`

```nim
proc removeHandler*(handler: Logger)
```

**Что делает.** Удаляет первое вхождение `handler` из списка зарегистрированных обработчиков. Если логгер был добавлен несколько раз, вызывайте эту процедуру по одному разу на каждую регистрацию.

```nim
addHandler(debugLog)
# ... выполняем отладочную работу ...
removeHandler(debugLog)   # отключаем debug-логгер на оставшееся время
```

---

### `getHandlers`

```nim
proc getHandlers*(): seq[Logger]
```

**Что делает.** Возвращает снимок текущего списка зарегистрированных обработчиков для вызывающего потока. Полезен для инспекции, тестирования или условной регистрации.

```nim
if debugLog notin getHandlers():
    addHandler(debugLog)
```

---

## Глобальный фильтр

### `setLogFilter`

```nim
proc setLogFilter*(lvl: Level)
```

**Что делает.** Устанавливает глобальный минимальный уровень для текущего потока. Любое сообщение ниже `lvl` молча отбрасывается ещё до того, как дойдёт до проверки `levelThreshold` конкретного логгера. По умолчанию установлен `lvlAll` — глобальная фильтрация отсутствует.

Это самый быстрый способ в рантайме заглушить целые категории вывода, не трогая отдельные логгеры — например, переключиться с подробного отладочного вывода на предупреждения перед входом в критически важный по производительности участок.

```nim
setLogFilter(lvlWarn)   # отныне debug и info подавляются глобально
warn("это появится")
info("а это — нет")
```

---

### `getLogFilter`

```nim
proc getLogFilter*(): Level
```

**Что делает.** Возвращает текущий уровень глобального фильтра для вызывающего потока.

```nim
echo "Текущий фильтр: ", getLogFilter()   # напр. WARN
```

---

## Шаблоны логирования

Эти шаблоны рассылают сообщения **всем зарегистрированным обработчикам** одновременно (в отличие от вызова `log` у конкретного логгера). Сначала проверяется глобальный фильтр, затем каждый обработчик применяет свой `levelThreshold`.

### `log` (шаблон)

```nim
template log*(level: Level, args: varargs[string, `$`])
```

**Что делает.** Главный широковещательный шаблон. Рассылает сообщение каждому зарегистрированному обработчику. Все уровневые сокращения ниже делегируют именно ему.

```nim
log(lvlInfo, "Воркер запущен, id потока: ", threadId)
```

---

### `debug`, `info`, `notice`, `warn`, `error`, `fatal`

```nim
template debug* (args: varargs[string, `$`])
template info*  (args: varargs[string, `$`])
template notice*(args: varargs[string, `$`])
template warn*  (args: varargs[string, `$`])
template error* (args: varargs[string, `$`])
template fatal* (args: varargs[string, `$`])
```

**Что делают.** Удобные обёртки вокруг `log` с фиксированным уровнем. Вызов `error("msg")` полностью эквивалентен `log(lvlError, "msg")`. Их использование вместо общего `log` делает код в местах вызовов более читаемым и позволяет легко найти все вызовы определённого уровня через поиск по тексту.

`fatal` заслуживает особого внимания: он только логирует сообщение, но **не завершает** программу. После вызова `fatal` необходимо добавить `quit` или `raise` самостоятельно.

```nim
debug("промах кэша для ключа: ", key)
info("пользователь ", userId, " вошёл в систему")
notice("конфигурационный файл перезагружен")
warn("время ответа превысило порог: ", elapsed, "мс")
error("запрос к базе данных завершился ошибкой: ", errMsg)
fatal("нехватка памяти — продолжение невозможно")
quit(1)
```

---

## Низкоуровневая утилита

### `substituteLog`

```nim
proc substituteLog*(frmt: string, level: Level,
                    args: varargs[string, `$`]): string
```

**Что делает.** Движок форматирования, используемый внутри всех логгеров. Принимает строку формата, разворачивает все плейсхолдеры `$переменная` в актуальные значения для заданного `level`, затем добавляет конкатенированные `args` и возвращает готовую строку.

Напрямую вызывать эту функцию придётся только при создании пользовательского логгера. Понимание её работы объясняет, как устроены строки формата и сборка сообщений.

```nim
echo substituteLog("[$levelname] ", lvlWarn, "диск заполнен")
# Вывод: [WARN] диск заполнен

echo substituteLog("$levelid|", lvlError, "код=", 500)
# Вывод: E|код=500
```

---

## Полные примеры

### Минимальное логирование в консоль

```nim
import std/logging

var log = newConsoleLogger(levelThreshold = lvlInfo)
addHandler(log)

info("Приложение запущено")
warn("Конфигурационный файл не найден, используются умолчания")
error("Не удалось подключиться к базе данных")
```

### Несколько обработчиков с разными порогами

```nim
import std/logging

# Всё видно в терминале в процессе разработки
addHandler(newConsoleLogger())

# Только предупреждения и выше — в постоянный файл
addHandler(newFileLogger("app.log", levelThreshold = lvlWarn))

# Ротируемый файл для всех сообщений, циклы по 5000 строк
addHandler(newRollingFileLogger("trace.log", maxLines = 5000))

info("Сервер готов")        # → консоль + ротируемый файл
warn("Высокое потребление памяти") # → все три
error("Таймаут запроса")    # → все три
```

### Пользовательский логгер

```nim
import std/logging

type NetworkLogger = ref object of Logger
  endpoint: string

method log*(logger: NetworkLogger, level: Level,
            args: varargs[string, `$`]) =
  if level >= logger.levelThreshold:
    let msg = substituteLog(logger.fmtStr, level, args)
    sendHttpPost(logger.endpoint, msg)   # гипотетическая функция

var netLog = NetworkLogger(
    endpoint: "https://logs.example.com/ingest",
    levelThreshold: lvlError,
    fmtStr: defaultFmtStr
)
addHandler(netLog)

error("Платёжный шлюз недоступен")   # отправлено по HTTP
```

### Изменение уровня детализации в рантайме

```nim
import std/logging

addHandler(newConsoleLogger())

setLogFilter(lvlDebug)
debug("подробная диагностика при запуске...")

setLogFilter(lvlWarn)
debug("это сообщение будет молча отброшено")
warn("а это — появится")
```

### Логирование в нескольких потоках

```nim
import std/logging, std/threadpool

proc workerThread() =
    # В каждом потоке обработчики нужно регистрировать заново
    addHandler(newConsoleLogger(levelThreshold = lvlInfo))
    setLogFilter(lvlInfo)
    info("Рабочий поток запущен")

spawn workerThread()
sync()
```
