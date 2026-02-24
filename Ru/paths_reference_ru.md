# Справочник по модулю `paths`

> **Модуль:** `std/paths`  
> **Язык:** Nim  
> **Назначение:** Типобезопасная, кросс-платформенная работа с путями файловой системы — соединение, разбиение, нормализация и анализ путей без каких-либо операций ввода/вывода.

---

## Ключевая идея: зачем нужен тип `Path`?

В большинстве языков пути файловой системы — обычные строки. Это порождает целый класс незаметных ошибок: можно случайно передать URL туда, где ожидается путь, соединить пути через `&` вместо правильного разделителя, или сравнить через `==` два пути, ссылающихся на один файл, но записанных по-разному. Ни одна из этих ошибок не обнаруживается на этапе компиляции.

`std/paths` решает эту проблему, делая `Path` **отдельным типом** — внутри он основан на `string`, но компилятор обращается с ним как с совершенно другим типом. Вы не можете случайно перепутать `Path` со `string` или с другими distinct-строками. Все операции с путями в этом модуле работают исключительно со значениями `Path`, и система типов обеспечивает корректность на этапе компиляции.

Модуль **исключительно вычислительный**: он оперирует строками путей. Никаких операций ввода/вывода, системных вызовов и проверок того, существуют ли на самом деле файл или директория. Для операций, затрагивающих файловую систему, смотрите `std/files` и `std/dirs`.

---

## Экспортируемые платформенные константы (из `osseps`)

Эти константы описывают соглашения о разделителях текущей операционной системы. Они реэкспортируются, чтобы можно было писать кросс-платформенный код без явного импорта `osseps`.

| Константа | Unix | Windows | Описание |
|---|---|---|---|
| `DirSep` | `'/'` | `'\\'` | Основной разделитель директорий |
| `AltSep` | `'/'` | `'/'` | Альтернативный разделитель (на Windows `/` тоже принимается) |
| `PathSep` | `':'` | `';'` | Разделитель между путями в переменных типа `PATH` |
| `FileSystemCaseSensitive` | `true` | `false` | Различает ли ОС `Foo` от `foo` |
| `ExeExt` | `""` | `"exe"` | Расширение исполняемых файлов (без ведущей точки) |
| `ScriptExt` | `""` | `"bat"` | Расширение файлов скриптов |
| `DynlibFormat` | `"lib$1.so"` | `"$1.dll"` | Формат имён файлов разделяемых библиотек |

---

## Тип `Path`

```nim
type Path* = distinct string
```

`Path` — это `string` «под капотом», но компилятор воспринимает его как совершенно отдельный тип. Преобразование между ними выполняется явно:

```nim
let p = Path("/usr/local/bin")   # оборачиваем строковый литерал в Path
let s = string(p)                 # или: p.string — разворачиваем обратно в string
echo $p                           # $ преобразует Path в string для вывода
```

---

## Операторы

### `/` — Соединение путей

```nim
func `/`*(head, tail: Path): Path
```

**Что делает:** Соединяет два компонента пути в один, помещая между ними платформенно-правильный разделитель. Это безопасный, идиоматичный способ строить пути в Nim — никогда не используйте конкатенацию строк `&` для этой цели.

Оператор нормализует результат: лишние разделители сжимаются, компоненты `.` (текущая директория) упрощаются, а завершающий разделитель сохраняется тогда и только тогда, когда `tail` оканчивается на него (или если `tail` пустой и `head` оканчивается на него). `..` и символические ссылки **не** разрешаются — для этого используйте `normalizePath` или `absolutePath`.

```nim
import std/paths

let base = Path("/home/user")
let config = Path(".config/myapp")

let full = base / config
assert $full == "/home/user/.config/myapp"

# Корректно работает независимо от завершающих слешей:
assert $(Path("/usr/") / Path("bin")) == "/usr/bin"
assert $(Path("/usr")  / Path("bin")) == "/usr/bin"

# Построение пути по частям:
let log = Path("/var") / Path("log") / Path("myapp") / Path("error.log")
assert $log == "/var/log/myapp/error.log"
```

---

### `/../` — Подняться вверх, затем спуститься

```nim
func `/../`*(head, tail: Path): Path
```

**Что делает:** Эквивалентен `parentDir(head) / tail`, но безопасен, когда `head` не имеет родителя. Если `head` уже является корневой директорией (у неё нет родителя), оператор возвращается к `head / tail` вместо того, чтобы подниматься выше корня.

Думайте об этом как «подняться на один уровень от `head`, затем спуститься в `tail`».

```nim
import std/paths

# Обычный случай: поднимаемся от "project/src", спускаемся в "docs":
let p = Path("project/src") /.. Path("docs")
# parentDir("project/src") == "project", затем / "docs"
assert $p == "project/docs"

# Граничный случай: head является корнем — остаёмся в корне:
let q = Path("/") /.. Path("etc")
assert $q == "/etc"
```

---

### `==` — Сравнение путей

```nim
func `==`*(x, y: Path): bool
```

**Что делает:** Сравнивает два пути на равенство с учётом платформы. На файловых системах с учётом регистра (Linux, большинство Unix) сравнение регистрозависимо. На файловых системах без учёта регистра (Windows, macOS по умолчанию) сравнение регистронезависимо.

Это означает, что `Path("Foo/Bar") == Path("foo/bar")` даёт `true` на Windows, но `false` на Linux — в точности соответствуя реальному поведению файловой системы.

```nim
import std/paths

when defined(windows):
  assert Path("C:/Users/Alice") == Path("c:/users/alice")
else:
  assert Path("/home/alice") == Path("/home/alice")
  assert Path("/home/Alice") != Path("/home/alice")
```

---

### `$` — Преобразование в строку

```nim
template `$`*(x: Path): string
```

**Что делает:** Преобразует `Path` в обычную `string`, пригодную для передачи в `echo`, строковое форматирование или любой API, не принимающий `Path` напрямую.

```nim
import std/paths

let p = Path("/etc/hosts")
echo $p             # /etc/hosts
echo "Файл: " & $p  # Файл: /etc/hosts
```

---

### `add` — Добавление на месте

```nim
func add*(x: var Path, y: Path)
```

**Что делает:** Добавляет `y` к `x` на месте (in-place) с той же семантикой нормализации, что и `/`. Полезно при пошаговом построении пути внутри цикла без создания промежуточных `Path`-значений.

```nim
import std/paths

var p = Path("/var/log")
p.add Path("myapp")
p.add Path("access.log")
assert $p == "/var/log/myapp/access.log"
```

---

## Разбиение и декомпозиция

### `splitPath`

```nim
func splitPath*(path: Path): tuple[head, tail: Path]
```

**Что делает:** Разбивает `path` на пару `(head, tail)`, где `head` — всё до последнего разделителя, а `tail` — финальный компонент. Инвариант `head / tail == path` выполняется для большинства путей (с граничными случаями вроде `"/"`, у которого `tail` пуст).

Это низкоуровневое разбиение: оно всегда делит по последнему разделителю вне зависимости от того, является ли последний компонент файлом или директорией.

```nim
import std/paths

let (head, tail) = splitPath(Path("/home/user/documents/report.txt"))
assert $head == "/home/user/documents"
assert $tail == "report.txt"

let (h2, t2) = splitPath(Path("/usr"))
assert $h2 == "/"
assert $t2 == "usr"

# Круговой обход:
assert head / tail == Path("/home/user/documents/report.txt")
```

---

### `splitFile`

```nim
func splitFile*(path: Path): tuple[dir, name: Path, ext: string]
```

**Что делает:** Разбивает `path` на три части: компонент директории, базовое имя файла (без расширения) и расширение (включая ведущую точку). Это самая полная декомпозиция — полезна, когда нужно реконструировать путь с другим именем или расширением.

Правила:
- `dir` не оканчивается на разделитель (если это не корень `/`).
- `ext` включает ведущую `.` (например, `".txt"`), или `""` если расширения нет.
- Если путь не содержит компонента директории, `dir` — пустой `Path`.

```nim
import std/paths

let (dir, name, ext) = splitFile(Path("/home/user/report.tar.gz"))
assert $dir  == "/home/user"
assert $name == "report.tar"
assert ext   == ".gz"

# Реконструкция с другим расширением:
let newPath = dir / Path($name & ".bz2")
assert $newPath == "/home/user/report.tar.bz2"

# Без расширения:
let (d2, n2, e2) = splitFile(Path("/usr/bin/ls"))
assert $n2 == "ls"
assert e2  == ""

# Без директории:
let (d3, n3, e3) = splitFile(Path("readme.md"))
assert $d3 == ""
assert $n3 == "readme"
assert e3  == ".md"
```

---

## Извлечение имени файла

### `extractFilename`

```nim
func extractFilename*(path: Path): Path
```

**Что делает:** Возвращает только часть `path`, соответствующую имени файла — последний компонент вместе с расширением. Эквивалентно `splitFile(path).name & splitFile(path).ext` или `splitPath(path).tail`.

Завершающий разделитель означает, что путь ссылается на директорию: в этом случае `extractFilename` возвращает пустой `Path`.

```nim
import std/paths

assert $extractFilename(Path("/home/user/photo.jpg")) == "photo.jpg"
assert $extractFilename(Path("/usr/bin/nim"))         == "nim"
assert $extractFilename(Path("/var/log/"))            == ""
```

---

### `lastPathPart`

```nim
func lastPathPart*(path: Path): Path
```

**Что делает:** Как `extractFilename`, но **игнорирует завершающие разделители**. Если `path` оканчивается на `/`, сначала его убирает, а затем возвращает последний компонент. Соответствует `basename` в shell-скриптах и большинстве других языков.

Используйте `lastPathPart`, когда хотите получить содержательное последнее имя пути — будь то файл или директория.

```nim
import std/paths

assert $lastPathPart(Path("/home/user/photo.jpg")) == "photo.jpg"
assert $lastPathPart(Path("/var/log/"))             == "log"   # / в конце игнорируется
assert $lastPathPart(Path("/usr/bin"))              == "bin"
```

---

## Работа с расширениями

### `changeFileExt`

```nim
func changeFileExt*(filename: Path, ext: string): Path
```

**Что делает:** Возвращает новый путь, идентичный `filename`, но с расширением файла, заменённым на `ext`. Если у `filename` нет расширения, `ext` добавляется. Если `ext` равен `""`, любое имеющееся расширение удаляется.

Передавайте `ext` **без** ведущей точки — `"txt"`, а не `".txt"`. Функция добавляет точку сама.

```nim
import std/paths

assert $changeFileExt(Path("report.docx"), "pdf")  == "report.pdf"
assert $changeFileExt(Path("/src/main.nim"), "c")  == "/src/main.c"
assert $changeFileExt(Path("archive.tar.gz"), "")  == "archive.tar"
assert $changeFileExt(Path("Makefile"), "bak")     == "Makefile.bak"
```

---

### `addFileExt`

```nim
func addFileExt*(filename: Path, ext: string): Path
```

**Что делает:** Добавляет расширение `ext` к `filename`, но **только если** у файла ещё нет расширения. Если расширение уже есть — возвращает `filename` без изменений. Как и `changeFileExt`, передавайте `ext` без ведущей точки.

Это вариант «убедиться, что расширение есть» — двойное добавление невозможно.

```nim
import std/paths

assert $addFileExt(Path("report"), "pdf")       == "report.pdf"
assert $addFileExt(Path("report.pdf"), "pdf")   == "report.pdf"  # без изменений
assert $addFileExt(Path("archive.tar"), "gz")   == "archive.tar" # расширение уже есть
```

---

## Навигация по директориям

### `parentDir`

```nim
func parentDir*(path: Path): Path
```

**Что делает:** Возвращает родительскую директорию `path`. Обрабатывает нормализацию: перед вычислением родителя завершающий разделитель убирается, поэтому `parentDir(Path("/foo/bar/"))` корректно возвращает `/foo`, а не `/foo/bar`.

Если `path` уже является корневой директорией, `parentDir` возвращает сам корень.

```nim
import std/paths

assert $parentDir(Path("/home/user/file.txt"))  == "/home/user"
assert $parentDir(Path("/home/user/"))          == "/home"
assert $parentDir(Path("/"))                    == "/"

# Работает и с относительными путями:
assert $parentDir(Path("project/src/main.nim")) == "project/src"
```

---

### `tailDir`

```nim
func tailDir*(path: Path): Path
```

**Что делает:** Возвращает всё в `path` после первого компонента директории — противоположность `parentDir`. Если у `path` только один компонент (или это корень), возвращает пустой `Path`.

Полезно для удаления известного префикса из пути.

```nim
import std/paths

assert $tailDir(Path("/home/user/file.txt"))  == "user/file.txt"
assert $tailDir(Path("/usr"))                 == ""
assert $tailDir(Path("a/b/c"))               == "b/c"
```

---

### `parentDirs` *(итератор)*

```nim
iterator parentDirs*(path: Path, fromRoot=false, inclusive=true): Path
```

**Что делает:** Обходит все родительские директории `path`, возвращая каждую как `Path`.

- **`inclusive = true`** (по умолчанию): в обход включается сам `path` как первое (или последнее) значение.
- **`inclusive = false`**: возвращаются только строгие предки; сам `path` исключается.
- **`fromRoot = false`** (по умолчанию): обход идёт от `path` вверх к корню. Первое возвращаемое значение — `path` (или его родитель при `inclusive = false`), каждое последующее — на уровень ближе к корню.
- **`fromRoot = true`**: обход идёт от корня вниз к `path`. Первое значение — корень.

Относительные пути обходятся как есть, без преобразования в абсолютные.

```nim
import std/paths

# От листа к корню (по умолчанию):
for p in parentDirs(Path("/a/b/c/file.txt")):
  echo $p
# /a/b/c/file.txt
# /a/b/c
# /a/b
# /a
# /

# От корня к листу, исключая сам путь:
for p in parentDirs(Path("/a/b/c"), fromRoot=true, inclusive=false):
  echo $p
# /
# /a
# /a/b
```

---

## Абсолютные и относительные пути

### `isAbsolute`

```nim
func isAbsolute*(path: Path): bool
```

**Что делает:** Возвращает `true`, если `path` является абсолютным путём. На Unix это означает, что он начинается с `/`. На Windows это включает и пути с буквой диска (`C:\...`), и UNC-пути по сети (`\\server\share`).

```nim
import std/paths

assert isAbsolute(Path("/usr/bin"))        == true
assert isAbsolute(Path("relative/path"))   == false
assert isAbsolute(Path("./local"))         == false
```

---

### `isRootDir`

```nim
func isRootDir*(path: Path): bool
```

**Что делает:** Возвращает `true`, если `path` является корнем файловой системы — `/` на Unix, корень диска типа `C:\` на Windows.

```nim
import std/paths

assert isRootDir(Path("/"))    == true
assert isRootDir(Path("/usr")) == false
```

---

### `isRelativeTo`

```nim
proc isRelativeTo*(path: Path, base: Path): bool
```

**Что делает:** Возвращает `true`, если `path` является потомком `base` — то есть `base` является префиксом `path` с точки зрения компонентов пути (не просто строковым префиксом). Для получения осмысленного результата оба пути желательно передавать абсолютными.

```nim
import std/paths

assert isRelativeTo(Path("/home/user/docs"), Path("/home/user"))  == true
assert isRelativeTo(Path("/home/user"),      Path("/home/user"))  == true
assert isRelativeTo(Path("/home/bob"),       Path("/home/user"))  == false
```

---

### `relativePath`

```nim
proc relativePath*(path, base: Path, sep = DirSep): Path
```

**Что делает:** Вычисляет относительный путь от `base` до `path` — путь, который нужно написать, если текущая директория — это `base`, а нужно достичь `path`.

Необязательный параметр `sep` управляет разделителем в результирующей строке. Установка `sep = '/'` гарантирует использование только прямых слешей — полезно для построения URL или кросс-платформенного вывода относительных путей.

На Windows, если `path` и `base` находятся на разных дисках, образовать относительный путь невозможно — функция возвращает `path` без изменений (он будет абсолютным).

```nim
import std/paths

assert $relativePath(Path("/home/user/docs"), Path("/home/user")) == "docs"
assert $relativePath(Path("/a/b/c"), Path("/a/x/y"))             == "../../b/c"
assert $relativePath(Path("/a/b"),   Path("/a/b"))               == "."

# Принудительно использовать прямые слеши:
assert $relativePath(Path("/a/b/c"), Path("/a"), '/') == "b/c"
```

---

## Нормализация

### `normalizePath`

```nim
proc normalizePath*(path: var Path)
```

**Что делает:** Нормализует `path` на месте: разрешает компоненты `.` (текущая директория) и `..` (родительская директория), сжимает лишние разделители, приводит разделители к платформенному виду. **Не** делает путь абсолютным и **не** разрешает символические ссылки.

```nim
import std/paths

var p = Path("/home/user/./docs/../photos")
normalizePath(p)
assert $p == "/home/user/photos"

var q = Path("a//b///c")
normalizePath(q)
assert $q == "a/b/c"
```

---

### `normalizePathEnd`

```nim
proc normalizePathEnd*(path: var Path, trailingSep = false)
```

**Что делает:** Корректирует только завершающий разделитель `path`. При `trailingSep = true` гарантирует, что путь оканчивается на разделитель. При `trailingSep = false` (по умолчанию) — гарантирует, что не оканчивается. Остальная часть пути не изменяется.

Полезно, когда нужно обеспечить согласованность завершающего разделителя перед сравнением или соединением путей.

```nim
import std/paths

var p = Path("/home/user/")
normalizePathEnd(p, trailingSep = false)
assert $p == "/home/user"

var q = Path("/home/user")
normalizePathEnd(q, trailingSep = true)
assert $q == "/home/user/"
```

---

### `normalizeExe`

```nim
proc normalizeExe*(file: var Path)
```

**Что делает:** Нормализует путь к исполняемому файлу. На Unix: если `file` не содержит разделителя (то есть это просто имя вроде `"python"`), оставляет без изменений — будет найден через `PATH`. Если разделитель есть, добавляет `"./"` в начало при его отсутствии, чтобы путь трактовался как явный, а не как поиск по `PATH`. На Windows обеспечивает наличие расширения `.exe`.

```nim
import std/paths

var p = Path("myprogram")
normalizeExe(p)
# На Unix: p == "myprogram" (поиск через PATH)

var q = Path("./myprogram")
normalizeExe(q)
# На Unix: q == "./myprogram" (явный относительный путь)
```

---

### `absolutePath`

```nim
proc absolutePath*(path: Path, root = getCurrentDir()): Path
```

**Что делает:** Если `path` уже абсолютный, возвращает его без изменений. Если `path` относительный, разрешает его относительно `root` (по умолчанию — текущая рабочая директория) и возвращает результирующий абсолютный путь с нормализацией.

Важно: это **лексическая** операция — `realpath` не вызывается, символические ссылки не разрешаются. Используйте для получения строки абсолютного пути; не полагайтесь на то, что путь ссылается на существующий файл.

```nim
import std/paths

# Абсолютные пути возвращаются без изменений:
assert $absolutePath(Path("/etc/hosts")) == "/etc/hosts"

# Относительные пути разрешаются относительно CWD по умолчанию:
let abs = absolutePath(Path("src/main.nim"))
assert isAbsolute(abs)

# Или относительно явного корня:
let abs2 = absolutePath(Path("config.toml"), root = Path("/opt/myapp"))
assert $abs2 == "/opt/myapp/config.toml"
```

---

## Рабочая директория и пути пользователя

### `getCurrentDir`

```nim
proc getCurrentDir*(): Path
```

**Что делает:** Возвращает текущую рабочую директорию выполняющегося процесса как абсолютный `Path`. Определяется в момент выполнения, а не компиляции — отражает место, откуда был запущен процесс, или место, куда он последний раз был перемещён через `setCurrentDir` (из `std/dirs`).

```nim
import std/paths

let cwd = getCurrentDir()
echo "Запущено из: ", $cwd

# Полезно как основа для разрешения относительных путей:
let configFile = cwd / Path("config.toml")
```

---

### `expandTilde`

```nim
proc expandTilde*(path: Path): Path
```

**Что делает:** Разворачивает ведущий `~` в `path` до домашней директории пользователя (возвращаемой `getHomeDir()` из `std/appdirs`). Правила разворачивания:

- `Path("~")` один — полный путь к домашней директории.
- `Path("~/foo")` — домашняя директория, соединённая с `foo`.
- `Path("~\foo")` — то же на Windows (оба разделителя обрабатываются).
- Любой путь, не начинающийся с `~`, — возвращается без изменений.

Форма `~имя_пользователя` (домашняя директория конкретного пользователя) **не** поддерживается и возвращается как есть.

```nim
import std/paths, std/appdirs

let home = getHomeDir()

assert expandTilde(Path("~"))            == home
assert expandTilde(Path("~/Documents"))  == home / Path("Documents")
assert expandTilde(Path("/etc/hosts"))   == Path("/etc/hosts")  # без изменений
```

---

### `unixToNativePath`

```nim
func unixToNativePath*(path: Path, drive = Path("")): Path
```

**Что делает:** Преобразует путь в Unix-стиле (использующий `/`, `.`, `..`) в нативный формат текущей операционной системы. На Unix — не меняет ничего. На Windows — конвертирует `/` в `\`, соответствующим образом обрабатывает `.` и `..`.

Параметр `drive` используется на системах с понятием букв дисков. Если `path` абсолютный (начинается с `/`) и `drive` указан, к нему добавляется префикс буквы диска. Если `drive` пустой (по умолчанию), используется диск текущей рабочей директории.

```nim
import std/paths

when defined(windows):
  assert $unixToNativePath(Path("/home/user")) == "\\home\\user"
else:
  assert $unixToNativePath(Path("/home/user")) == "/home/user"
```

---

## Хеширование

### `hash`

```nim
func hash*(x: Path): Hash
```

**Что делает:** Вычисляет хеш для `Path`, пригодный для использования в `Table`, `HashSet` и других хеш-контейнерах. Хеш вычисляется после нормализации пути (поэтому `"/foo/./bar"` и `"/foo/bar"` дают одинаковый хеш) и регистронезависим на файловых системах без учёта регистра.

Эта функция обычно не вызывается напрямую — она срабатывает автоматически, когда `Path` используется как ключ хеш-таблицы.

```nim
import std/[paths, tables]

var registry: Table[Path, string]
registry[Path("/etc/hosts")] = "системный файл hosts"
registry[Path("/etc/./hosts")] = "перезапись"  # тот же ключ после нормализации

assert registry.len == 1
```

---

## Краткий справочник: функции декомпозиции

| Функция | Вход | Возвращает |
|---|---|---|
| `splitPath` | `"/a/b/c.txt"` | `head="/a/b"`, `tail="c.txt"` |
| `splitFile` | `"/a/b/c.tar.gz"` | `dir="/a/b"`, `name="c.tar"`, `ext=".gz"` |
| `extractFilename` | `"/a/b/c.txt"` | `"c.txt"` |
| `lastPathPart` | `"/a/b/c/"` | `"c"` (завершающий `/` игнорируется) |
| `parentDir` | `"/a/b/c.txt"` | `"/a/b"` |
| `tailDir` | `"/a/b/c"` | `"b/c"` |

## Краткий справочник: функции работы с расширениями

| Функция | Расширение уже есть? | Поведение |
|---|---|---|
| `changeFileExt(p, "pdf")` | Да | Заменяет имеющееся расширение |
| `changeFileExt(p, "pdf")` | Нет | Добавляет расширение |
| `changeFileExt(p, "")` | Да | Удаляет расширение |
| `addFileExt(p, "pdf")` | Да | **Возвращает без изменений** |
| `addFileExt(p, "pdf")` | Нет | Добавляет расширение |
