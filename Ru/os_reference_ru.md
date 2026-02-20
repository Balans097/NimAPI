# Модуль `os` Nim — Полный справочник

> **Модуль:** `std/os`  
> **Назначение:** Базовые средства операционной системы — переменные окружения, директории, пути, информация о файлах, запуск команд и многое другое.

---

## Содержание

1. [Константы](#константы)
2. [Типы](#типы)
3. [Работа с путями](#работа-с-путями)
4. [Экранирование для оболочки](#экранирование-для-оболочки)
5. [Поиск исполняемых файлов](#поиск-исполняемых-файлов)
6. [Временные метки файлов](#временные-метки-файлов)
7. [Размер и информация о файле](#размер-и-информация-о-файле)
8. [Права доступа к файлам](#права-доступа-к-файлам)
9. [Жёсткие ссылки](#жёсткие-ссылки)
10. [Процессы и система](#процессы-и-система)
11. [Информация о приложении](#информация-о-приложении)
12. [Скрытые файлы и валидация имён](#скрытые-файлы-и-валидация-имён)
13. [Устаревшие псевдонимы](#устаревшие-псевдонимы)

---

## Константы

### `invalidFilenameChars`

```nim
const invalidFilenameChars*: set[char]
```

Набор символов, которые могут делать имя файла недопустимым на Linux, Windows или macOS.

| Платформа | Запрещённые символы |
|-----------|---------------------|
| Linux     | `/` |
| macOS     | `:` |
| Windows   | Все остальные из набора |

**Значение:** `{'/', '\\', ':', '*', '?', '"', '<', '>', '|', '^', '\0'}`

**Пример:**
```nim
import std/os

let name = "мой:файл*имя"
for ch in name:
  if ch in invalidFilenameChars:
    echo "Недопустимый символ: ", ch
# Вывод:
# Недопустимый символ: :
# Недопустимый символ: *
```

---

### `invalidFilenames`

```nim
const invalidFilenames*: array[...]
```

Список зарезервированных имён файлов, недопустимых на Windows (и, возможно, других платформах). Включает `CON`, `PRN`, `AUX`, `NUL`, `COM0`–`COM9`, `LPT0`–`LPT9`.

**Пример:**
```nim
import std/os, std/strutils

let name = "CON"
for invalid in invalidFilenames:
  if cmpIgnoreCase(name, invalid) == 0:
    echo name, " — зарезервированное имя файла!"
# Вывод: CON — зарезервированное имя файла!
```

---

### `ExeExts`

```nim
const ExeExts*: array[...]
```

Расширения исполняемых файлов, специфичные для платформы.

- **Windows:** `["exe", "cmd", "bat"]`
- **POSIX:** `[""]`

**Пример:**
```nim
import std/os
echo ExeExts  # на Linux: [""]
              # на Windows: ["exe", "cmd", "bat"]
```

---

## Типы

### `FileInfo`

```nim
type FileInfo* = object
  id*: tuple[device: DeviceId, file: FileId]
  kind*: PathComponent
  size*: BiggestInt
  permissions*: set[FilePermission]
  linkCount*: BiggestInt
  lastAccessTime*: times.Time
  lastWriteTime*: times.Time
  creationTime*: times.Time
  blockSize*: int
  isSpecial*: bool
```

Содержит информацию об объекте файловой системы. Возвращается функцией `getFileInfo`.

| Поле | Описание |
|------|----------|
| `id` | Кортеж: идентификатор устройства и файла |
| `kind` | `pcFile`, `pcDir`, `pcLinkToFile`, `pcLinkToDir` |
| `size` | Размер файла в байтах |
| `permissions` | Набор значений `FilePermission` |
| `linkCount` | Количество жёстких ссылок |
| `lastAccessTime` | Время последнего доступа |
| `lastWriteTime` | Время последней модификации |
| `creationTime` | Время создания (ненадёжно на некоторых ОС) |
| `blockSize` | Предпочтительный размер блока ввода-вывода |
| `isSpecial` | `true` для нестандартных файлов (FIFO, устройства) на Unix |

---

### `DeviceId` / `FileId`

```nim
when defined(windows):
  type DeviceId* = int32
  type FileId* = int64
elif defined(posix):
  type DeviceId* = Dev
  type FileId* = Ino
```

Платформозависимые типы идентификаторов устройства и иноды.

---

## Работа с путями

### `expandTilde`

```nim
proc expandTilde*(path: string): string
```

Разворачивает `~` или `~/...` в полный путь до домашней директории. Если строка не начинается с `~`, возвращается без изменений. На Windows поддерживаются оба варианта: `~/` и `~\`.

**Параметры:**
- `path` — строка пути, возможно начинающаяся с `~`

**Возвращает:** Полный абсолютный путь.

**Пример:**
```nim
import std/os

echo expandTilde("~/documents")       # напр. /home/user/documents
echo expandTilde("~")                 # напр. /home/user/
echo expandTilde("/абсолютный/путь")  # /абсолютный/путь (без изменений)
echo expandTilde("относительный")     # относительный (без изменений)
```

---

## Экранирование для оболочки

### `quoteShellWindows`

```nim
proc quoteShellWindows*(s: string): string
```

Экранирует строку для безопасной передачи в Windows API / `cmd.exe`. Основан на Python `subprocess.list2cmdline`. Корректно обрабатывает пробелы, табуляции, обратные слэши и кавычки.

**Параметры:**
- `s` — строка-аргумент для экранирования

**Возвращает:** Безопасно экранированную строку для Windows.

**Пример:**
```nim
import std/os

echo quoteShellWindows("hello world")   # "hello world"
echo quoteShellWindows("без-пробелов")  # без-пробелов
echo quoteShellWindows(`с"кавычкой`)    # "с\"кавычкой"
echo quoteShellWindows("")              # ""
```

---

### `quoteShellPosix`

```nim
proc quoteShellPosix*(s: string): string
```

Экранирует строку для безопасной передачи в POSIX-оболочку (bash, sh и т.д.). Безопасные символы (буквенно-цифровые, `%+-./_ :=@`) оставляются без изменений; остальные заключаются в одинарные кавычки.

**Параметры:**
- `s` — строка-аргумент для экранирования

**Возвращает:** Безопасно экранированную строку для POSIX-оболочки.

**Пример:**
```nim
import std/os

echo quoteShellPosix("hello world")    # 'hello world'
echo quoteShellPosix("safe-string")    # safe-string
echo quoteShellPosix("")               # ''
echo quoteShellPosix("it's here")      # 'it'"'"'s here'
```

---

### `quoteShell`

```nim
proc quoteShell*(s: string): string
```

Экранирует строку под текущую платформу. Вызывает `quoteShellWindows` на Windows и `quoteShellPosix` на POSIX. Доступен на Windows, POSIX и Nintendo Switch.

**Параметры:**
- `s` — строка-аргумент для экранирования

**Возвращает:** Экранированную строку для текущей платформы.

**Пример:**
```nim
import std/os

let arg = "hello world"
echo quoteShell(arg)
# Linux/macOS: 'hello world'
# Windows:     "hello world"
```

---

### `quoteShellCommand`

```nim
proc quoteShellCommand*(args: openArray[string]): string
```

Объединяет и экранирует несколько аргументов в одну строку команды, готовую к передаче оболочке.

**Параметры:**
- `args` — массив строк-аргументов

**Возвращает:** Строку аргументов через пробел, каждый из которых экранирован.

**Пример:**
```nim
import std/os

# На POSIX:
echo quoteShellCommand(["ls", "-la", "моя папка"])
# ls -la 'моя папка'

# На Windows:
echo quoteShellCommand(["dir", "", "C:\\Program Files"])
# dir "" "C:\Program Files"
```

---

## Поиск исполняемых файлов

### `findExe`

```nim
proc findExe*(exe: string, followSymlinks: bool = true;
              extensions: openArray[string] = ExeExts): string
```

Ищет исполняемый файл с именем `exe` сначала в текущей директории, затем в директориях из переменной окружения `PATH`. При необходимости автоматически добавляет платформозависимые расширения.

**Параметры:**
- `exe` — имя исполняемого файла (с расширением или без)
- `followSymlinks` — если `true` (по умолчанию), символические ссылки разрешаются до реального файла
- `extensions` — список расширений для перебора; по умолчанию `ExeExts`

**Возвращает:** Полный путь к исполняемому файлу или `""`, если не найден.

**Недоступен на:** NimScript / JS.

**Пример:**
```nim
import std/os

let pythonPath = findExe("python3")
if pythonPath.len > 0:
  echo "Python найден: ", pythonPath
else:
  echo "Python не найден в PATH"

# Поиск без разрешения симлинков:
let path = findExe("gcc", followSymlinks = false)
echo path
```

---

## Временные метки файлов

### `getLastModificationTime`

```nim
proc getLastModificationTime*(file: string): times.Time
```

Возвращает время последнего изменения файла.

**Параметры:**
- `file` — путь к файлу

**Возвращает:** Значение `times.Time`.

**Исключения:** `OSError` при ошибке доступа.

**Пример:**
```nim
import std/os, std/times

let t = getLastModificationTime("myfile.txt")
echo t.format("yyyy-MM-dd HH:mm:ss")
```

---

### `getLastAccessTime`

```nim
proc getLastAccessTime*(file: string): times.Time
```

Возвращает время последнего чтения или записи файла.

**Параметры:**
- `file` — путь к файлу

**Возвращает:** Значение `times.Time`.

**Исключения:** `OSError` при ошибке доступа.

**Пример:**
```nim
import std/os, std/times

let t = getLastAccessTime("myfile.txt")
echo "Последний доступ: ", t.format("yyyy-MM-dd HH:mm:ss")
```

---

### `getCreationTime`

```nim
proc getCreationTime*(file: string): times.Time
```

Возвращает время создания файла.

> **Примечание (POSIX):** На POSIX-системах может возвращать время последнего изменения атрибутов файла (`ctime`), а не реальное время создания.

**Параметры:**
- `file` — путь к файлу

**Возвращает:** Значение `times.Time`.

**Исключения:** `OSError` при ошибке доступа.

**Пример:**
```nim
import std/os, std/times

let t = getCreationTime("myfile.txt")
echo "Создан: ", t.format("yyyy-MM-dd")
```

---

### `fileNewer`

```nim
proc fileNewer*(a, b: string): bool
```

Возвращает `true`, если файл `a` новее файла `b` (т.е. время его последнего изменения позже).

**Параметры:**
- `a` — путь к первому файлу
- `b` — путь к второму файлу

**Возвращает:** `true`, если `a` новее `b`.

**Пример:**
```nim
import std/os

if fileNewer("output.bin", "source.nim"):
  echo "Результат актуален"
else:
  echo "Необходима перекомпиляция"
```

---

### `setLastModificationTime`

```nim
proc setLastModificationTime*(file: string, t: times.Time)
```

Устанавливает время последней модификации файла.

**Параметры:**
- `file` — путь к файлу
- `t` — новое время модификации

**Исключения:** `OSError` при ошибке.

**Пример:**
```nim
import std/os, std/times

let newTime = now().toTime()
setLastModificationTime("myfile.txt", newTime)
echo "Временная метка обновлена"
```

---

## Размер и информация о файле

### `getFileSize`

```nim
proc getFileSize*(file: string): BiggestInt
```

Возвращает размер файла в байтах.

**Параметры:**
- `file` — путь к файлу

**Возвращает:** Размер в байтах как `BiggestInt`.

**Исключения:** `OSError` при ошибке.

**Пример:**
```nim
import std/os

let size = getFileSize("myfile.txt")
echo "Размер файла: ", size, " байт"

# Человекочитаемый вид:
if size < 1024:
  echo size, " Б"
elif size < 1024 * 1024:
  echo size div 1024, " КБ"
else:
  echo size div (1024*1024), " МБ"
```

---

### `getFileInfo` (по дескриптору)

```nim
proc getFileInfo*(handle: FileHandle): FileInfo
```

Возвращает `FileInfo` для файла, представленного системным дескриптором.

**Исключения:** `OSError` при недопустимом дескрипторе.

**Пример:**
```nim
import std/os

var f: File
if open(f, "myfile.txt"):
  let info = getFileInfo(f.getFileHandle())
  echo "Размер: ", info.size
  close(f)
```

---

### `getFileInfo` (по объекту File)

```nim
proc getFileInfo*(file: File): FileInfo
```

Возвращает `FileInfo` для открытого объекта `File`.

**Исключения:** `IOError` если `file` равен nil; `OSError` при системной ошибке.

**Пример:**
```nim
import std/os

var f: File
if open(f, "myfile.txt"):
  let info = getFileInfo(f)
  echo "Размер блока: ", info.blockSize
  echo "Это директория: ", info.kind == pcDir
  close(f)
```

---

### `getFileInfo` (по пути)

```nim
proc getFileInfo*(path: string, followSymlink = true): FileInfo
```

Возвращает `FileInfo` для файла по указанному пути. При `followSymlink = true` (по умолчанию) символические ссылки разрешаются. При `false` — информация о самой ссылке.

**Параметры:**
- `path` — путь в файловой системе
- `followSymlink` — следовать ли по символическим ссылкам

**Исключения:** `OSError` если путь не существует или доступ запрещён.

**Пример:**
```nim
import std/os

let info = getFileInfo("/etc/hosts")
echo "Тип: ", info.kind         # pcFile
echo "Размер: ", info.size
echo "Права: ", info.permissions

# Информация о самой символической ссылке:
let symlinkInfo = getFileInfo("/etc/localtime", followSymlink = false)
echo "Это симлинк: ", symlinkInfo.kind in {pcLinkToFile, pcLinkToDir}
```

---

### `sameFileContent`

```nim
proc sameFileContent*(path1, path2: string): bool
```

Возвращает `true`, если содержимое обоих файлов побайтово идентично. Использует буферизованное чтение для эффективности.

**Параметры:**
- `path1` — путь к первому файлу
- `path2` — путь ко второму файлу

**Возвращает:** `true`, если содержимое одинаково.

**Пример:**
```nim
import std/os

if sameFileContent("оригинал.txt", "резервная_копия.txt"):
  echo "Файлы идентичны"
else:
  echo "Файлы отличаются"
```

---

## Права доступа к файлам

### `inclFilePermissions`

```nim
proc inclFilePermissions*(filename: string, permissions: set[FilePermission])
```

Добавляет указанные права к существующим правам файла. Эквивалентно:
```nim
setFilePermissions(filename, getFilePermissions(filename) + permissions)
```

**Параметры:**
- `filename` — путь к файлу
- `permissions` — набор значений `FilePermission` для добавления

**Пример:**
```nim
import std/os

# Сделать скрипт исполняемым для владельца:
inclFilePermissions("myscript.sh", {fpUserExec})

# Добавить право чтения для группы:
inclFilePermissions("shared.cfg", {fpGroupRead})
```

---

### `exclFilePermissions`

```nim
proc exclFilePermissions*(filename: string, permissions: set[FilePermission])
```

Удаляет указанные права из существующих прав файла. Эквивалентно:
```nim
setFilePermissions(filename, getFilePermissions(filename) - permissions)
```

**Параметры:**
- `filename` — путь к файлу
- `permissions` — набор значений `FilePermission` для удаления

**Пример:**
```nim
import std/os

# Запретить запись для остальных пользователей:
exclFilePermissions("public.html", {fpOthersWrite})

# Сделать файл доступным только для чтения:
exclFilePermissions("readonly.cfg", {fpUserWrite, fpGroupWrite, fpOthersWrite})
```

---

## Жёсткие ссылки

### `createHardlink`

```nim
proc createHardlink*(src, dest: string)
```

Создаёт жёсткую ссылку `dest`, указывающую на файл `src`. После этого оба пути ссылаются на одни и те же данные.

> **Предупреждение:** На некоторых ОС создание жёстких ссылок разрешено только суперпользователю/администратору.

**Параметры:**
- `src` — путь к существующему файлу
- `dest` — путь, по которому будет создана ссылка

**Исключения:** `OSError` при ошибке.

**Пример:**
```nim
import std/os

createHardlink("original.dat", "hardlink.dat")

# Оба пути указывают на одни данные:
echo sameFileContent("original.dat", "hardlink.dat")  # true

# Количество ссылок увеличилось:
let info = getFileInfo("original.dat")
echo "Количество ссылок: ", info.linkCount  # 2
```

---

## Процессы и система

### `sleep`

```nim
proc sleep*(milsecs: int)
```

Приостанавливает выполнение на `milsecs` миллисекунд. Отрицательное значение немедленно возвращает управление.

**Параметры:**
- `milsecs` — время в миллисекундах

**Пример:**
```nim
import std/os

echo "Начало..."
sleep(2000)    # пауза на 2 секунды
echo "Готово!"

# Отрицательное значение безопасно и ничего не делает:
sleep(-100)
```

---

### `execShellCmd`

```nim
proc execShellCmd*(command: string): int
```

Выполняет команду оболочки и ожидает её завершения. Возвращает код завершения (0 при успехе).

> **Примечание:** Для расширенного управления вводом-выводом процесса используйте `osproc.execProcess`.

**Параметры:**
- `command` — строка команды (напр. `"ls -la"`)

**Возвращает:** Код завершения процесса оболочки.

**Пример:**
```nim
import std/os

let code = execShellCmd("echo Привет из оболочки")
echo "Код завершения: ", code   # 0

# Проверка наличия команды:
if execShellCmd("which python3") == 0:
  echo "python3 доступен"

# Запуск команды с ошибкой:
let err = execShellCmd("false")
echo "Код ошибки: ", err   # ненулевой
```

---

### `exitStatusLikeShell`

```nim
proc exitStatusLikeShell*(status: cint): cint
```

Преобразует «сырой» код завершения из C-функции `system()` в код завершения в стиле оболочки. На POSIX: если процесс был завершён сигналом, возвращает `128 + номер_сигнала`.

**Параметры:**
- `status` — «сырой» статус от C `system()`

**Возвращает:** Код завершения в стиле оболочки.

**Пример:**
```nim
import std/os

# Обычно используется внутренне, но может быть полезен при обёртке C-вызовов:
let rawStatus: cint = 0
echo exitStatusLikeShell(rawStatus)  # 0
```

---

### `isAdmin`

```nim
proc isAdmin*: bool
```

Возвращает `true`, если текущий процесс запущен с привилегиями администратора/суперпользователя.

- **Windows:** проверяет членство в группе Administrators
- **POSIX:** проверяет, равен ли эффективный UID (`euid`) нулю

**Пример:**
```nim
import std/os

if isAdmin():
  echo "Запущено от имени администратора/root"
else:
  echo "Запущено от имени обычного пользователя"
  echo "Некоторые операции могут потребовать повышенных прав"
```

---

### `getCurrentProcessId`

```nim
proc getCurrentProcessId*(): int
```

Возвращает идентификатор текущего процесса (PID).

**Возвращает:** Целочисленный PID текущего процесса.

**Пример:**
```nim
import std/os

echo "PID текущего процесса: ", getCurrentProcessId()
# напр. PID текущего процесса: 12345
```

---

## Информация о приложении

### `expandFilename`

```nim
proc expandFilename*(filename: string): string
```

Возвращает полный абсолютный путь к существующему файлу. Разрешает символические ссылки до реального пути.

**Параметры:**
- `filename` — относительный или абсолютный путь к существующему файлу

**Возвращает:** Абсолютный разрешённый путь.

**Исключения:** `OSError` если файл не существует или доступ запрещён.

**Пример:**
```nim
import std/os

# Из скрипта в /home/user/projects/:
echo expandFilename("../data/file.txt")
# напр. /home/user/data/file.txt

echo expandFilename("./main.nim")
# напр. /home/user/projects/main.nim
```

---

### `getAppFilename`

```nim
proc getAppFilename*(): string
```

Возвращает абсолютный путь к исполняемому файлу текущего приложения. Разрешает символические ссылки. Возвращает пустую строку, если путь определить не удалось.

**Возвращает:** Абсолютный путь или `""` при ошибке.

**Пример:**
```nim
import std/os

let exe = getAppFilename()
echo "Это приложение: ", exe
# напр. /usr/local/bin/myapp
```

---

### `getAppDir`

```nim
proc getAppDir*(): string
```

Возвращает директорию, содержащую исполняемый файл текущего приложения. Эквивалентно `splitFile(getAppFilename()).dir`.

**Возвращает:** Строку с путём к директории.

**Пример:**
```nim
import std/os

let dir = getAppDir()
echo "Директория приложения: ", dir

# Загрузка конфигурации относительно исполняемого файла:
let config = dir / "config.ini"
if fileExists(config):
  echo "Конфиг найден: ", config
```

---

### `getCurrentCompilerExe`

```nim
proc getCurrentCompilerExe*(): string {.compileTime.}
```

Возвращает путь к текущему запущенному компилятору Nim (или nimble) **во время компиляции**. Полезно в макросах или nimscript для программного вызова компилятора.

**Возвращает:** Путь к исполняемому файлу компилятора Nim (вычисляется при компиляции).

**Пример:**
```nim
import std/os

const nimPath = getCurrentCompilerExe()
echo "Компилятор: ", nimPath  # вычисляется при компиляции
```

---

## Скрытые файлы и валидация имён

### `isHidden`

```nim
proc isHidden*(path: string): bool
```

Определяет, является ли файл или директория по указанному пути скрытой.

- **POSIX:** `true`, если последний компонент пути начинается с `.` и это не `.` и не `..`
- **Windows:** `true`, если файл существует и имеет атрибут `FILE_ATTRIBUTE_HIDDEN`

> **Примечание:** Пути не нормализуются перед проверкой.

**Параметры:**
- `path` — путь к файлу или директории

**Возвращает:** `true`, если объект скрытый.

**Пример:**
```nim
import std/os

# Примеры для POSIX:
echo isHidden(".bashrc")           # true
echo isHidden("/home/user/.ssh")   # true (последний компонент .ssh)
echo isHidden(".foo/bar")          # false (bar не является скрытым)
echo isHidden(".")                 # false
echo isHidden("..")                # false
echo isHidden(".hidden/")          # true (завершающий слэш допустим)
echo isHidden("visible.txt")       # false
```

---

### `isValidFilename`

```nim
func isValidFilename*(filename: string, maxLen = 259.Positive): bool
```

Возвращает `true`, если `filename` является допустимым именем файла для кросс-платформенного использования (Windows, Linux, macOS). Проверяет по `invalidFilenameChars`, `invalidFilenames` и максимальной длине.

> **Предупреждение:** Проверяет только имя файла, а не полный путь.

**Параметры:**
- `filename` — строка имени файла для проверки (только имя, без пути)
- `maxLen` — максимально допустимая длина (по умолчанию 259 — ограничение Windows `MAX_PATH` минус один)

**Возвращает:** `true`, если имя допустимо на всех платформах.

**Условия недопустимости:**
- Пробел в начале или конце
- Заканчивается на точку `.`
- Содержит символы из `invalidFilenameChars` (`/ \ : * ? " < > | ^ \0`)
- Совпадает с зарезервированным именем Windows без учёта регистра (`CON`, `AUX`, `NUL`, `COM1`–`COM9`, `LPT1`–`LPT9` и др.)
- Пустая строка
- Длина превышает `maxLen`

**Пример:**
```nim
import std/os

echo isValidFilename("myFile.txt")    # true
echo isValidFilename(" myFile.txt")   # false (пробел в начале)
echo isValidFilename("myFile.txt ")   # false (пробел в конце)
echo isValidFilename("myFile.")       # false (заканчивается точкой)
echo isValidFilename("con.txt")       # false (зарезервированное имя)
echo isValidFilename("my:file")       # false (двоеточие недопустимо)
echo isValidFilename("")              # false (пустая строка)
echo isValidFilename("foo/bar")       # false (содержит слэш)

# Своя максимальная длина:
echo isValidFilename("a".repeat(260), maxLen = 100)  # false
```

---

## Устаревшие псевдонимы

### `existsFile` *(устарело)*

```nim
template existsFile*(args: varargs[untyped]): untyped {.deprecated: "use fileExists".}
```

Используйте вместо него `fileExists`.

---

### `existsDir` *(устарело)*

```nim
template existsDir*(args: varargs[untyped]): untyped {.deprecated: "use dirExists".}
```

Используйте вместо него `dirExists`.

---

## Реэкспортируемые модули

Модуль `os` реэкспортирует следующие подмодули, делая все их символы доступными напрямую:

| Модуль | Содержимое |
|--------|-----------|
| `std/private/ospaths2` | Работа с путями: `joinPath`, `splitPath`, `splitFile`, `changeFileExt`, `addFileExt`, `parentDir`, `isAbsolute` и др. |
| `std/private/osfiles` | Операции с файлами: `copyFile`, `moveFile`, `removeFile`, `fileExists`, `getFilePermissions`, `setFilePermissions` |
| `std/private/osdirs` | Операции с директориями: `createDir`, `removeDir`, `copyDir`, `dirExists`, `walkDir`, `walkDirRec`, `getCurrentDir`, `setCurrentDir` |
| `std/private/ossymlinks` | Символические ссылки: `createSymlink`, `expandSymlink`, `symlinkExists` |
| `std/private/osappdirs` | Директории приложения: `getHomeDir`, `getConfigDir`, `getTempDir`, `getAppDir` |
| `std/cmdline` | Аргументы командной строки: `paramCount`, `paramStr`, `commandLineParams` |
| `std/oserrors` | Типы ошибок ОС: `OSError`, `osLastError`, `raiseOSError` |
| `std/envvars` | Переменные окружения: `getEnv`, `putEnv`, `delEnv`, `existsEnv`, `envPairs` |
| `std/private/osseps` | Разделители путей: `DirSep`, `AltSep`, `PathSep`, `ExeExt` |

---

## Сводная таблица доступности по платформам

| Функция | Linux | macOS | Windows | NimScript | JS |
|---------|:-----:|:-----:|:-------:|:---------:|:--:|
| `expandTilde` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `quoteShellWindows` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `quoteShellPosix` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `quoteShell` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `quoteShellCommand` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `findExe` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getLastModificationTime` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getLastAccessTime` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getCreationTime` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `fileNewer` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `setLastModificationTime` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getFileSize` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getFileInfo` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `sameFileContent` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `inclFilePermissions` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `exclFilePermissions` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `createHardlink` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `sleep` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `execShellCmd` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `isAdmin` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getCurrentProcessId` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `expandFilename` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getAppFilename` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getAppDir` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `getCurrentCompilerExe` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `isHidden` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `isValidFilename` | ✓ | ✓ | ✓ | ✓ | ✓ |

---

*Сгенерировано на основе модуля `std/os` стандартной библиотеки Nim.*
