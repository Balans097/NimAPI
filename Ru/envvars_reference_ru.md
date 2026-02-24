# envvars — справочник по модулю

## Обзор

Модуль `std/envvars` предоставляет полный кроссплатформенный интерфейс для чтения, записи и перебора **переменных окружения** — пар ключ/значение, которые каждый процесс операционной системы наследует от родительского процесса и передаёт дочерним.

Модуль работает единообразно во всех бэкендах и на всех целевых платформах Nim:

| Целевая платформа | Реализация |
|---|---|
| Native (Linux, macOS, Windows и др.) | Стандартная библиотека C (`getenv`, `setenv`, `unsetenv`, `environ`) |
| Windows (специфика) | API с широкими строками (`_wgetenv`, `SetEnvironmentVariableW`) для корректной работы с Unicode |
| Node.js | JavaScript-объект `process.env` |
| Nim VM (`nimvm`) | Внутренние VM-операции компилятора |

**Интеграция с системой эффектов.** Экспортируются два типа эффектов, чтобы система эффектов Nim могла отслеживать доступ к окружению в сигнатурах процедур:

- `ReadEnvEffect` — применяется к процедурам, **читающим** переменные окружения. Наследуется от `ReadIOEffect`.
- `WriteEnvEffect` — применяется к процедурам, **записывающим или удаляющим** переменные окружения. Наследуется от `WriteIOEffect`.

Вызывающий код, аннотированный ограниченными наборами эффектов, получит ошибку компиляции при попытке вызвать процедуру, помеченную эффектом, которого он не допускает. Это стандартный механизм Nim для контроля побочных эффектов ввода-вывода.

---

## `getEnv`

```nim
proc getEnv*(key: string, default = ""): string {.tags: [ReadEnvEffect].}
```

### Описание

Читает значение переменной окружения с именем `key` и возвращает его в виде строки. Если переменная с таким именем отсутствует в окружении текущего процесса, возвращается значение `default` (по умолчанию `""`).

Важная тонкость: переменная окружения может существовать и при этом законно содержать **пустую строку** в качестве значения. В этом случае `getEnv` возвращает `""` — неотличимо от ситуации, когда переменной нет вовсе. Когда это различие важно — например, при проверке обязательной конфигурации — используйте сначала `existsEnv` для подтверждения наличия, а затем вызывайте `getEnv`.

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `key` | `string` | Имя читаемой переменной окружения |
| `default` | `string` | Значение, возвращаемое при отсутствии переменной. По умолчанию `""`. |

### Возвращаемое значение

Строковое значение переменной или `default`, если она не существует.

### Примеры

**Чтение известной переменной:**

```nim
import std/envvars

let home = getEnv("HOME")
echo "Домашний каталог: ", home
# На Linux/macOS: /home/username
# На Windows: "" если HOME не задана; вместо неё используйте USERPROFILE
```

**Запасное значение для необязательной конфигурации:**

```nim
import std/envvars

let host = getEnv("DB_HOST", "localhost")
let port = getEnv("DB_PORT", "5432")
echo "Подключаемся к ", host, ":", port
# Если DB_HOST не задана → "localhost"
# Если DB_HOST=myserver → "myserver"
```

**Неоднозначность пустой строки и как с ней работать:**

```nim
import std/envvars

# Допустим, APP_TOKEN не задана вовсе.
# Оба вызова вернут "":
echo getEnv("APP_TOKEN")           # ""
echo getEnv("APP_TOKEN", "")       # ""

# Чтобы выяснить, существует ли переменная:
if existsEnv("APP_TOKEN"):
  let token = getEnv("APP_TOKEN")
  # token всё равно может быть "", если кто-то сделал: export APP_TOKEN=
else:
  echo "APP_TOKEN не задана"
```

**Бэкенд Node.js — тот же API, реализованный через `process.env`:**

```nim
# Компиляция: nim js -d:nodejs myapp.nim
import std/envvars
echo getEnv("PATH")   # читает process.env["PATH"]
```

---

## `existsEnv`

```nim
proc existsEnv*(key: string): bool {.tags: [ReadEnvEffect].}
```

### Описание

Проверяет, присутствует ли переменная окружения с именем `key` в окружении текущего процесса. Возвращает `true`, если переменная существует (вне зависимости от её значения, в том числе пустого), и `false`, если она отсутствует вовсе.

Это правильный способ проверить *наличие* переменной, когда её значение может законно быть пустой строкой. Сравнение `getEnv(key) != ""` ошибочно уравняло бы отсутствующую переменную и присутствующую, но пустую.

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `key` | `string` | Имя проверяемой переменной окружения |

### Возвращаемое значение

`true`, если переменная существует; `false` в противном случае.

### Примеры

**Простая проверка наличия:**

```nim
import std/envvars

if existsEnv("CI"):
  echo "Работаем в среде CI"
else:
  echo "Локальный запуск"
```

**Защита обязательной конфигурации:**

```nim
import std/envvars

const required = ["DATABASE_URL", "SECRET_KEY", "REDIS_URL"]

for varName in required:
  if not existsEnv(varName):
    raise newException(ValueError, "Обязательная переменная не задана: " & varName)

echo "Все переменные на месте, запускаемся…"
```

**Различие между отсутствующей и пустой переменной:**

```nim
import std/envvars

putEnv("EMPTY_VAR", "")    # переменная существует, но значение — ""

echo existsEnv("EMPTY_VAR")          # true  — переменная ЕСТЬ
echo getEnv("EMPTY_VAR") == ""       # true  — но значение пустое
echo existsEnv("NONEXISTENT_VAR")    # false — этой переменной нет вовсе
```

---

## `putEnv`

```nim
proc putEnv*(key, val: string) {.tags: [WriteEnvEffect].}
```

### Описание

Устанавливает переменную окружения с именем `key` в значение `val`, создавая её, если она не существует, или перезаписывая, если существует. Изменение немедленно становится видимым для всех последующих вызовов `getEnv` и `existsEnv` в том же процессе. Дочерние процессы, запущенные после вызова, унаследуют новое значение.

Если операция завершается неудачей по любой причине, возбуждается `OSError` с системным сообщением об ошибке. На Windows выполняется дополнительная проверка: ключ должен быть непустым и не содержать символ `=` (оба условия недопустимы в именах переменных окружения Windows).

**Потокобезопасность.** Переменные окружения — глобальный ресурс на уровне процесса. Вызов `putEnv` из нескольких потоков одновременно, или вызов во время итерации другим потоком через `envPairs`, является неопределённым поведением на большинстве платформ. При необходимости синхронизируйте доступ извне.

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `key` | `string` | Имя устанавливаемой переменной окружения |
| `val` | `string` | Присваиваемое значение |

### Возбуждает

`OSError` при сбое системного вызова.

### Примеры

**Создание новой переменной:**

```nim
import std/envvars

putEnv("MY_APP_MODE", "production")
assert getEnv("MY_APP_MODE") == "production"
```

**Перезапись существующей переменной:**

```nim
import std/envvars

putEnv("LOG_LEVEL", "info")
echo getEnv("LOG_LEVEL")   # info

putEnv("LOG_LEVEL", "debug")
echo getEnv("LOG_LEVEL")   # debug
```

**Установка переменной для дочернего процесса:**

```nim
import std/envvars, std/osproc

putEnv("WORKER_THREADS", "8")
# Любой процесс, запущенный после этого, унаследует WORKER_THREADS=8
let p = startProcess("myworker")
```

**Обработка ошибок на Windows — недопустимый ключ:**

```nim
import std/envvars

try:
  putEnv("", "value")        # пустой ключ → OSError на Windows
except OSError as e:
  echo "Не удалось задать переменную: ", e.msg

try:
  putEnv("KEY=NAME", "v")    # '=' в ключе → OSError на Windows
except OSError as e:
  echo "Не удалось задать переменную: ", e.msg
```

---

## `delEnv`

```nim
proc delEnv*(key: string) {.tags: [WriteEnvEffect].}
```

### Описание

Удаляет переменную окружения с именем `key` из окружения текущего процесса. После этого вызова `existsEnv(key)` вернёт `false`, а `getEnv(key)` — `""` (или любое другое значение по умолчанию, указанное вызывающей стороной). Дочерние процессы, запущенные после вызова, не унаследуют удалённую переменную.

При неудаче возбуждается `OSError`. На Windows ключ должен быть непустым и не содержать `=`; нарушение любого из этих условий немедленно вызывает `OSError`, до какого-либо системного вызова.

Удаление несуществующей переменной не является ошибкой на POSIX-системах (базовый вызов `unsetenv` просто ничего не делает). На Windows, однако, попытка удалить несуществующую переменную через `_putenv` может вернуть ошибку в зависимости от версии рантайма — для максимальной переносимости оборачивайте вызов проверкой `existsEnv`.

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `key` | `string` | Имя удаляемой переменной окружения |

### Возбуждает

`OSError` при сбое системного вызова.

### Примеры

**Удаление переменной:**

```nim
import std/envvars

putEnv("TEMP_FLAG", "1")
assert existsEnv("TEMP_FLAG")

delEnv("TEMP_FLAG")
assert not existsEnv("TEMP_FLAG")
assert getEnv("TEMP_FLAG") == ""
```

**Безопасное удаление — проверка перед удалением для максимальной переносимости:**

```nim
import std/envvars

if existsEnv("SESSION_TOKEN"):
  delEnv("SESSION_TOKEN")
```

**Восстановление окружения в тестовых фикстурах:**

```nim
import std/envvars

proc withEnv(key, val: string, body: proc()) =
  let hadBefore = existsEnv(key)
  let oldVal = getEnv(key)
  putEnv(key, val)
  try:
    body()
  finally:
    if hadBefore:
      putEnv(key, oldVal)   # восстанавливаем исходное значение
    else:
      delEnv(key)           # убираем то, что добавили

withEnv("FEATURE_FLAG", "1"):
  doTest()
```

---

## `envPairs` (итератор)

```nim
iterator envPairs*(): tuple[key, value: string] {.tags: [ReadEnvEffect].}
```

### Описание

Перебирает **все переменные окружения** текущего процесса, выдавая каждую в виде кортежа `(key, value)` из строк. Порядок итерации не гарантирован и зависит от операционной системы и порядка установки переменных. Каждая переменная, присутствующая в момент вызова итератора, посещается ровно один раз.

`envPairs` работает на всех поддерживаемых целевых платформах: нативные бинарники, Node.js (`process.env`) и Nim VM. На нативных платформах реализация читает непосредственно из блока окружения ОС — на POSIX-системах это глобальный массив `environ`; на Windows используется широкостроковый API `GetEnvironmentStringsW`.

**Семантика снимка.** Изменение окружения (через `putEnv` или `delEnv`) во время итерации является неопределённым поведением. Итератор отражает состояние окружения на момент начала работы; если нужно изменять окружение в процессе — сначала сделайте снимок в `seq`.

### Что выдаёт

`tuple[key, value: string]` — имя и значение каждой переменной окружения.

### Примеры

**Вывод всех переменных окружения:**

```nim
import std/envvars

for key, value in envPairs():
  echo key, "=", value
# PATH=/usr/local/bin:/usr/bin:/bin
# HOME=/home/alice
# TERM=xterm-256color
# … (все переменные окружения)
```

**Поиск переменных с заданным префиксом:**

```nim
import std/envvars

for key, value in envPairs():
  if key.startsWith("APP_"):
    echo key, " → ", value
# APP_MODE → production
# APP_PORT → 8080
```

**Сбор в таблицу для произвольного доступа:**

```nim
import std/envvars, std/tables

var env = initTable[string, string]()
for key, value in envPairs():
  env[key] = value

echo env.getOrDefault("HOME", "(не задано)")
```

**Безопасный снимок перед изменением окружения:**

```nim
import std/envvars, std/sequtils

# Сначала собираем, потом изменяем — избегаем неопределённого поведения
let snapshot = toSeq(envPairs())

for (key, _) in snapshot:
  if key.startsWith("LEGACY_"):
    delEnv(key)
```

**Подсчёт общего числа переменных и суммарного объёма данных:**

```nim
import std/envvars

var count = 0
var totalBytes = 0
for key, value in envPairs():
  inc count
  inc totalBytes, key.len + value.len + 1  # +1 за разделитель '='

echo "Всего переменных: ", count
echo "Всего байт данных: ", totalBytes
```

---

## Типы эффектов

### `ReadEnvEffect`

```nim
type ReadEnvEffect* = object of ReadIOEffect
```

Маркер эффекта времени компиляции. Автоматически применяется (через `{.tags: [ReadEnvEffect].}`) к `getEnv`, `existsEnv` и `envPairs`. Любая процедура, вызывающая одну из них, также приобретёт `ReadEnvEffect` через распространение эффектов, если только оно явно не подавлено.

Используйте этот тип в аннотациях собственных процедур, когда хотите сообщить, что процедура читает из окружения, или когда хотите ограничить вызывающих через `.forbids`:

```nim
import std/envvars

proc configValue(key: string): string {.tags: [ReadEnvEffect].} =
  getEnv(key, "default")
```

### `WriteEnvEffect`

```nim
type WriteEnvEffect* = object of WriteIOEffect
```

Маркер эффекта времени компиляции, применяемый к `putEnv` и `delEnv`. Распространяется на любую процедуру, вызывающую любую из них.

```nim
import std/envvars

proc applyConfig() {.tags: [WriteEnvEffect].} =
  putEnv("LOG_LEVEL", "warn")
  putEnv("MAX_CONN", "100")
```

---

## Сводная таблица

| Символ | Вид | Тег эффекта | Назначение |
|---|---|---|---|
| `getEnv(key, default)` | proc | `ReadEnvEffect` | Прочитать переменную; вернуть default при отсутствии |
| `existsEnv(key)` | proc | `ReadEnvEffect` | Проверить наличие переменной |
| `putEnv(key, val)` | proc | `WriteEnvEffect` | Задать (создать или перезаписать) переменную |
| `delEnv(key)` | proc | `WriteEnvEffect` | Удалить переменную |
| `envPairs()` | iterator | `ReadEnvEffect` | Перебрать все переменные как кортежи `(key, value)` |
| `ReadEnvEffect` | type | — | Тег эффекта для чтения из окружения |
| `WriteEnvEffect` | type | — | Тег эффекта для записи в окружение |
