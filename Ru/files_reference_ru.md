# files — справочник по модулю

## Обзор

Модуль `std/files` предоставляет **типобезопасные файловые операции**, построенные поверх типа `Path` из `std/paths`. Это рекомендуемый способ работы с файлами в современном Nim-коде: использование `Path` вместо сырых строк предотвращает случайное смешение строк-путей с обычными строками на уровне компиляции.

Каждая процедура в модуле — тонкая инлайновая обёртка вокруг соответствующей процедуры в `std/private/osfiles`. Реальная логика находится там; `std/files` просто переэкспортирует её с параметрами типа `Path`.

**Два типа реэкспортируются из нижнего слоя**, чтобы вызывающему коду не нужно было ничего импортировать дополнительно:

- `FilePermission` — перечисление, описывающее отдельные биты прав доступа в стиле POSIX.
- `CopyFlag` — перечисление, управляющее поведением при работе с символическими ссылками во время копирования.

### Значения `FilePermission`

| Значение | Смысл |
|---|---|
| `fpUserRead` | Владелец может читать |
| `fpUserWrite` | Владелец может писать |
| `fpUserExec` | Владелец может исполнять |
| `fpGroupRead` | Группа может читать |
| `fpGroupWrite` | Группа может писать |
| `fpGroupExec` | Группа может исполнять |
| `fpOthersRead` | Остальные могут читать |
| `fpOthersWrite` | Остальные могут писать |
| `fpOthersExec` | Остальные могут исполнять |

На Windows существует только понятие «только для чтения». Из всех `FilePermission` практический эффект имеет лишь `fpUserWrite`: его наличие означает, что файл доступен на запись; его отсутствие — что файл защищён от записи. Все остальные биты принимаются без ошибки, но никакого практического эффекта не оказывают.

### Значения `CopyFlag`

| Значение | Смысл |
|---|---|
| `cfSymlinkFollow` | Если источник — символическая ссылка, копировать файл, на который она указывает (по умолчанию) |
| `cfSymlinkAsIs` | Копировать саму символическую ссылку, не следуя за ней |

`CopyFlag` игнорируется на Windows — там символические ссылки всегда пропускаются.

---

## `fileExists`

```nim
proc fileExists*(filename: Path): bool
  {.inline, tags: [ReadDirEffect], sideEffect.}
```

### Описание

Возвращает `true`, если `filename` существует на файловой системе **и** является обычным файлом или символической ссылкой, указывающей на него. Для всего остального возвращает `false`: каталоги, файлы устройств, именованные каналы, сокеты и несуществующие пути.

Прагма `sideEffect` сигнализирует, что процедура выполняет реальный запрос к файловой системе при каждом вызове — она не является чистой, и её результат может меняться между вызовами даже с одним и тем же аргументом.

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `filename` | `Path` | Путь к проверяемому файлу |

### Возвращаемое значение

`true`, если путь указывает на существующий обычный файл или символическую ссылку на него; `false` в остальных случаях.

### Примеры

**Базовая проверка перед чтением:**

```nim
import std/files, std/paths

let cfg = Path("config.toml")
if fileExists(cfg):
  echo "Конфигурация найдена, загружаем…"
else:
  echo "Файл конфигурации отсутствует — используем значения по умолчанию."
```

**Различие между файлом и каталогом:**

```nim
import std/files, std/paths

let p = Path("/etc")
echo fileExists(p)   # false — /etc является каталогом, а не файлом
```

**Проверка перед перезаписью (предупреждение пользователя):**

```nim
import std/files, std/paths

let output = Path("report.pdf")
if fileExists(output):
  echo "Внимание: ", output, " будет перезаписан."
```

---

## `tryRemoveFile`

```nim
proc tryRemoveFile*(file: Path): bool {.inline, tags: [WriteDirEffect].}
```

### Описание

Пытается удалить `file` с файловой системы. Возвращает `true` при успехе или `false`, если удаление не удалось по любой причине — включая отсутствие прав, блокировку файла или указание на каталог. Принципиально важно: процедура также возвращает `true`, если файл изначально не существовал — отсутствующий файл не считается ошибкой.

На Windows атрибут «только для чтения» игнорируется, поэтому файл, защищённый от записи, можно удалить этой процедурой.

Используйте `tryRemoveFile`, когда удаление — это необязательный шаг очистки, а сбой допустим. Используйте `removeFile`, когда сбой должен трактоваться как ошибка.

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `file` | `Path` | Путь к удаляемому файлу |

### Возвращаемое значение

`true`, если файл удалён или не существовал; `false`, если удаление не удалось.

### Примеры

**Тихая очистка — игнорировать сбой:**

```nim
import std/files, std/paths

discard tryRemoveFile(Path("session.lock"))
# Если файл блокировки существовал — он удалён. Если нет — не страшно.
```

**Предупреждение при неудачной очистке:**

```nim
import std/files, std/paths

let tmp = Path("/tmp/myapp_scratch.bin")
if not tryRemoveFile(tmp):
  echo "Предупреждение: не удалось удалить временный файл ", tmp
```

**Идемпотентная подготовка в тестах:**

```nim
import std/files, std/paths

proc setupTest() =
  discard tryRemoveFile(Path("test_output.txt"))  # начинаем чисто, отсутствие не страшно
  # … создаём свежие тестовые данные …
```

---

## `removeFile`

```nim
proc removeFile*(file: Path) {.inline, tags: [WriteDirEffect].}
```

### Описание

Удаляет `file` с файловой системы. Если удаление не удаётся, возбуждается `OSError` с системным сообщением об ошибке. Как и `tryRemoveFile`, отсутствие файла **не** является ошибкой — в этом случае процедура завершается нормально.

На Windows атрибут «только для чтения» игнорируется.

Это строгий аналог `tryRemoveFile`. Предпочитайте его, когда сбой при удалении означает реальную проблему, которую вызывающий код не может или не должен молча игнорировать.

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `file` | `Path` | Путь к удаляемому файлу |

### Возбуждает

`OSError`, если файл существует, но не может быть удалён.

### Примеры

**Обязательная очистка после обработки:**

```nim
import std/files, std/paths

let tmpPath = Path("intermediate.bin")
processData(tmpPath)
removeFile(tmpPath)   # должно успеть; сбой означает серьёзную проблему
```

**Удаление файла как часть атомарной замены:**

```nim
import std/files, std/paths

# Сначала пишем во временный файл, затем атомарно заменяем цель
let target = Path("data.db")
let tmp    = Path("data.db.tmp")
writeDatabase(tmp)
removeFile(target)   # удаляем старый; возбуждаем ошибку если не можем
moveFile(tmp, target)
```

**Явная обработка `OSError`:**

```nim
import std/files, std/paths

try:
  removeFile(Path("locked_file.txt"))
except OSError as e:
  echo "Не удалось удалить файл: ", e.msg
```

---

## `moveFile`

```nim
proc moveFile*(source, dest: Path)
  {.inline, tags: [ReadDirEffect, ReadIOEffect, WriteIOEffect].}
```

### Описание

Перемещает (или переименовывает) файл с пути `source` по пути `dest`. Если `dest` уже существует, он **перезаписывается**. При сбое возбуждается `OSError`.

Символические ссылки обрабатываются на уровне самой ссылки — если `source` является символической ссылкой, перемещается именно она, а не файл, на который она указывает. Это делает `moveFile` безопасным для перемещения символических ссылок без случайного следования за ними.

В пределах одной файловой системы операция, как правило, реализуется как атомарное переименование, что делает её эффективным и надёжным способом замены файла обновлённой версией. При переходе между файловыми системами реализация откатывается к последовательности «скопировать — удалить».

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `source` | `Path` | Существующий файл или символическая ссылка для перемещения |
| `dest` | `Path` | Путь назначения (родительский каталог должен существовать) |

### Возбуждает

`OSError` при сбое операции.

### Примеры

**Переименование файла:**

```nim
import std/files, std/paths

moveFile(Path("draft.md"), Path("final.md"))
```

**Атомарная замена — пишем во временный, затем перемещаем на место:**

```nim
import std/files, std/paths

let live = Path("config.json")
let tmp  = Path("config.json.new")
writeConfig(tmp)
moveFile(tmp, live)   # атомарно перезаписывает live
```

**Перемещение символической ссылки без следования за ней:**

```nim
import std/files, std/paths

# Если "link" — символическая ссылка на "target.txt", после этого вызова
# "newlink" будет указывать на "target.txt", а "link" исчезнет.
moveFile(Path("link"), Path("newlink"))
```

---

## `copyFile`

```nim
proc copyFile*(source, dest: Path;
               options = cfSymlinkFollow;
               bufferSize = 16_384)
  {.inline, tags: [ReadDirEffect, ReadIOEffect, WriteIOEffect].}
```

### Описание

Копирует содержимое `source` в `dest`. Родительский каталог `dest` должен уже существовать; промежуточные каталоги процедура не создаёт.

**Обработка прав доступа зависит от платформы:**

- **Windows:** атрибуты файла копируются автоматически вместе с содержимым. Никаких дополнительных шагов не требуется.
- **Не-Windows:** копируется только содержимое файла. Файл назначения наследует права по умолчанию для вновь создаваемых файлов (как правило, определяются `umask` процесса). Для сохранения прав используйте `copyFileWithPermissions` или вызывайте `getFilePermissions` / `setFilePermissions` вручную.

Если `dest` уже существует, его атрибуты (владелец, права) сохраняются, а содержимое перезаписывается.

Параметр `options` управляет обработкой символических ссылок на не-Windows системах. По умолчанию `cfSymlinkFollow` означает: если `source` — символическая ссылка, копируется файл, на который она указывает. `cfSymlinkAsIs` копирует саму ссылку.

`bufferSize` (по умолчанию 16 КиБ) задаёт размер буфера чтения/записи при копировании. Большие значения могут улучшить пропускную способность на быстрых накопителях ценой большего потребления памяти; меньшие снижают потребление памяти ценой большего числа системных вызовов.

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `source` | `Path` | Копируемый файл |
| `dest` | `Path` | Путь назначения (родительский каталог должен существовать) |
| `options` | `CopyFlag` | Поведение при работе со ссылками; по умолчанию `cfSymlinkFollow` |
| `bufferSize` | `int` | Размер буфера ввода-вывода в байтах; по умолчанию `16_384` |

### Возбуждает

`OSError` при сбое копирования.

### Примеры

**Простое копирование файла:**

```nim
import std/files, std/paths

copyFile(Path("template.html"), Path("output/index.html"))
```

**Копирование с увеличенным буфером для больших файлов:**

```nim
import std/files, std/paths

copyFile(Path("video.mp4"), Path("backup/video.mp4"),
         bufferSize = 1_048_576)   # буфер 1 МиБ
```

**Копирование символической ссылки как ссылки (без следования за ней):**

```nim
import std/files, std/paths

copyFile(Path("link_to_data"), Path("backup/link_to_data"),
         options = cfSymlinkAsIs)
```

**Сохранение прав доступа на не-Windows (ручной подход):**

```nim
import std/files, std/paths

let src  = Path("script.sh")
let dest = Path("bin/script.sh")
copyFile(src, dest)
setFilePermissions(dest, getFilePermissions(src))
```

---

## `copyFileWithPermissions`

```nim
proc copyFileWithPermissions*(source, dest: Path;
                              ignorePermissionErrors = true;
                              options = cfSymlinkFollow) {.inline.}
```

### Описание

Копирует `source` в `dest`, а затем также копирует **биты прав доступа** файла, объединяя `copyFile`, `getFilePermissions` и `setFilePermissions` в один удобный вызов.

На Windows это эквивалентно `copyFile`, поскольку там атрибуты копируются автоматически. На не-Windows системах копирование прав происходит после копирования содержимого и **не является атомарным**: существует короткий промежуток между двумя шагами, в течение которого файл назначения существует с неправильными правами. Обычно это допустимо, но может иметь значение в контекстах, критичных к безопасности.

Если чтение или установка прав доступа завершается неудачей, поведение определяется `ignorePermissionErrors`:

- `true` (по умолчанию) — ошибка молча игнорируется; содержимое файла всё равно копируется.
- `false` — возбуждается `OSError`.

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `source` | `Path` | Копируемый файл |
| `dest` | `Path` | Путь назначения (родительский каталог должен существовать) |
| `ignorePermissionErrors` | `bool` | Если `true`, молча игнорировать ошибки копирования прав; по умолчанию `true` |
| `options` | `CopyFlag` | Поведение при работе со ссылками; по умолчанию `cfSymlinkFollow` |

### Возбуждает

`OSError` при сбое копирования файла, или если `ignorePermissionErrors` равен `false` и операция с правами завершается неудачей.

### Примеры

**Копирование с сохранением прав (с допуском ошибок прав доступа):**

```nim
import std/files, std/paths

copyFileWithPermissions(Path("setup.sh"), Path("dist/setup.sh"))
# dest будет иметь те же биты прав, что и source
```

**Строгий режим — завершать с ошибкой, если права не могут быть скопированы:**

```nim
import std/files, std/paths

copyFileWithPermissions(Path("private_key.pem"),
                        Path("backup/private_key.pem"),
                        ignorePermissionErrors = false)
# Возбуждает OSError при сбое chmod — важно для файлов, критичных к безопасности
```

---

## `copyFileToDir`

```nim
proc copyFileToDir*(source, dir: Path;
                    options = cfSymlinkFollow;
                    bufferSize = 16_384) {.inline.}
```

### Описание

Копирует файл по пути `source` в каталог `dir`, сохраняя исходное имя файла. Каталог назначения должен уже существовать. Это удобная обёртка вокруг `copyFile`, которая самостоятельно вычисляет `dest = dir / source.lastPathPart`, избавляя от шаблонного кода по извлечению имени файла и построению пути назначения.

Параметры `options` и `bufferSize` имеют тот же смысл, что и в `copyFile`.

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `source` | `Path` | Копируемый файл |
| `dir` | `Path` | Каталог назначения (должен существовать) |
| `options` | `CopyFlag` | Поведение при работе со ссылками; по умолчанию `cfSymlinkFollow` |
| `bufferSize` | `int` | Размер буфера ввода-вывода в байтах; по умолчанию `16_384` |

### Возбуждает

`OSError` при сбое копирования или если `dir` не существует.

### Примеры

**Копирование одного файла в каталог:**

```nim
import std/files, std/paths

copyFileToDir(Path("README.md"), Path("dist/"))
# Результат: dist/README.md
```

**Пакетное копирование — несколько файлов в один каталог:**

```nim
import std/files, std/paths

let assets = [Path("icon.png"), Path("style.css"), Path("app.js")]
for f in assets:
  copyFileToDir(f, Path("public/static/"))
```

---

## `getFilePermissions`

```nim
proc getFilePermissions*(filename: Path): set[FilePermission]
  {.inline, tags: [ReadDirEffect].}
```

### Описание

Возвращает биты прав доступа файла `filename` в виде Nim-множества значений `FilePermission`. Каждый элемент в множестве означает, что соответствующее право предоставлено.

На Windows значимо только состояние «только для чтения». Если файл доступен на запись, в результат включается `fpUserWrite`; если защищён от записи — `fpUserWrite` отсутствует. Все остальные значения прав на Windows всегда присутствуют в результате (поскольку соответствующая концепция там не существует).

Возбуждает `OSError` при сбое (например, файл не существует, недостаточно привилегий для запроса).

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `filename` | `Path` | Файл, права которого запрашиваются |

### Возвращаемое значение

`set[FilePermission]` — множество предоставленных прав доступа.

### Примеры

**Чтение и вывод прав доступа:**

```nim
import std/files, std/paths

let perms = getFilePermissions(Path("data.csv"))
if fpUserWrite in perms:
  echo "Файл доступен на запись владельцу."
if fpOthersRead in perms:
  echo "Файл доступен на чтение всем."
```

**Сохранение прав перед изменением:**

```nim
import std/files, std/paths

let f = Path("script.sh")
let original = getFilePermissions(f)

# Временно убираем права на исполнение
setFilePermissions(f, original - {fpUserExec, fpGroupExec, fpOthersExec})
doWork(f)
setFilePermissions(f, original)   # восстанавливаем
```

---

## `setFilePermissions`

```nim
proc setFilePermissions*(filename: Path;
                         permissions: set[FilePermission];
                         followSymlinks = true)
  {.inline, tags: [ReadDirEffect, WriteDirEffect].}
```

### Описание

Применяет набор прав `permissions` к файлу `filename`, полностью заменяя существующие права.

Когда `followSymlinks` равен `true` (по умолчанию) и `filename` является символической ссылкой, права применяются к **файлу-цели**, а не к самой ссылке. При значении `false` права применяются к самой ссылке — но это ничего не даёт на Windows и на большинстве POSIX-систем (включая Linux), где `lchmod` недоступен или всегда возвращает ошибку, поскольку права символических ссылок там не соблюдаются.

На Windows значение имеет только `fpUserWrite`: его отсутствие делает файл защищённым от записи; его наличие — доступным на запись.

Возбуждает `OSError` при сбое.

### Параметры

| Имя | Тип | Описание |
|---|---|---|
| `filename` | `Path` | Файл, права которого изменяются |
| `permissions` | `set[FilePermission]` | Полный набор применяемых прав |
| `followSymlinks` | `bool` | Следовать символическим ссылкам (по умолчанию `true`); `false` — no-op на большинстве POSIX-систем |

### Возбуждает

`OSError` при сбое операции.

### Примеры

**Сделать скрипт исполняемым:**

```nim
import std/files, std/paths

setFilePermissions(Path("deploy.sh"),
  {fpUserRead, fpUserWrite, fpUserExec,
   fpGroupRead, fpGroupExec,
   fpOthersRead, fpOthersExec})
# Эквивалент chmod 755
```

**Сделать файл доступным только для чтения всем:**

```nim
import std/files, std/paths

setFilePermissions(Path("archive.tar.gz"),
  {fpUserRead, fpGroupRead, fpOthersRead})
# Эквивалент chmod 444
```

**Ограничить приватный ключ правом чтения только для владельца:**

```nim
import std/files, std/paths

setFilePermissions(Path("id_rsa"), {fpUserRead})
# Эквивалент chmod 400 — стандартное требование для приватных ключей SSH
```

---

## Сводная таблица

| Процедура | Теги эффектов | Возбуждает | Описание |
|---|---|---|---|
| `fileExists(filename)` | `ReadDirEffect` | — | Проверить, является ли путь существующим файлом |
| `tryRemoveFile(file)` | `WriteDirEffect` | — | Удалить файл; вернуть `false` при сбое |
| `removeFile(file)` | `WriteDirEffect` | `OSError` | Удалить файл; возбудить ошибку при сбое |
| `moveFile(source, dest)` | `ReadDirEffect`, `ReadIOEffect`, `WriteIOEffect` | `OSError` | Переместить или переименовать файл |
| `copyFile(source, dest, …)` | `ReadDirEffect`, `ReadIOEffect`, `WriteIOEffect` | `OSError` | Скопировать содержимое файла |
| `copyFileWithPermissions(source, dest, …)` | как у `copyFile` | `OSError` | Скопировать содержимое и биты прав доступа |
| `copyFileToDir(source, dir, …)` | как у `copyFile` | `OSError` | Скопировать файл в каталог, сохранив имя |
| `getFilePermissions(filename)` | `ReadDirEffect` | `OSError` | Прочитать биты прав доступа файла |
| `setFilePermissions(filename, perms, …)` | `ReadDirEffect`, `WriteDirEffect` | `OSError` | Установить биты прав доступа файла |
