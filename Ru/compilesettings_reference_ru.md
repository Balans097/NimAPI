# Справочник модуля `compilesettings`

> **Назначение модуля:** `compilesettings` позволяет опрашивать компилятор Nim о его собственной конфигурации **во время компиляции**. Пока программа компилируется, можно спросить «куда пишется выходной файл?», «какой бэкенд используется?», «какие пути поиска активны?» — и вшить ответы прямо в константы, статические утверждения или логику условной компиляции. Все запросы происходят исключительно во время компиляции; никаких накладных расходов во время выполнения нет вообще.

---

## Содержание

1. [Тип: SingleValueSetting](#тип-singlevaluesetting)
2. [Тип: MultipleValueSetting](#тип-multiplevaluesetting)
3. [querySetting](#querysetting)
4. [querySettingSeq](#querysettingseq)
5. [Справочник полей SingleValueSetting](#справочник-полей-singlevaluesetting)
6. [Справочник полей MultipleValueSetting](#справочник-полей-multiplevaluesetting)
7. [Практические паттерны](#практические-паттерны)

---

## Тип: `SingleValueSetting`

```nim
type SingleValueSetting* {.pure.} = enum
  arguments, outFile, outDir, nimcacheDir, projectName, projectPath,
  projectFull, command, commandLine, linkOptions, compileOptions,
  ccompilerPath, backend, libPath, gc, mm
```

### Что это такое

Перечисление, называющее каждую настройку компилятора, которая разрешается в **одну строку**. Каждый член соответствует одному фрагменту информации, которую компилятор хранит внутри — пути к директории, списку флагов, имени бэкенда и т.д. Вы передаёте один из этих членов в `querySetting`, чтобы получить соответствующее значение.

Перечисление помечено `{.pure.}`, поэтому члены должны быть квалифицированы: `SingleValueSetting.outDir`, а не просто `outDir`.

> **Примечание о совместимости:** Новые значения добавляются только в конец этого перечисления. Это сохраняет бинарную совместимость между версиями компилятора Nim — бинарный файл, скомпилированный со старым Nim, по-прежнему корректно читает перечисление без неправильной интерпретации существующих членов.

---

## Тип: `MultipleValueSetting`

```nim
type MultipleValueSetting* {.pure.} = enum
  nimblePaths, searchPaths, lazyPaths, commandArgs, cincludes, clibs
```

### Что это такое

Перечисление, называющее каждую настройку компилятора, которая разрешается в **последовательность строк** — обычно потому, что базовая настройка естественным образом содержит несколько значений (несколько путей поиска, несколько подключаемых библиотек и т.д.). Вы передаёте один из этих членов в `querySettingSeq`, чтобы получить соответствующий `seq[string]`.

Как и `SingleValueSetting`, это перечисление `{.pure.}` и растёт только в конце.

---

## `querySetting`

```nim
proc querySetting*(setting: SingleValueSetting): string {.compileTime, noSideEffect.}
```

### Что делает

Возвращает текущее значение однозначной настройки компилятора в виде `string`. Вызов полностью перехватывается виртуальной машиной (VM) компилятора во время компиляции — документация Nim описывает это как «реализовано в vmops», то есть компилятор перехватывает вызов и возвращает ответ напрямую, не исполняя никакого кода Nim.

Поскольку процедура является `compileTime`, её нельзя вызвать во время выполнения. Она имеет смысл только внутри объявлений `const`, блоков `static`, веток `when`, макросов и других контекстов времени компиляции.

Прагма `noSideEffect` подтверждает, что запрос настройки не оказывает наблюдаемого эффекта на состояние программы — это чистое чтение.

### Ограничение `compileTime` на практике

Если вы попытаетесь вызвать `querySetting` из обычной процедуры времени выполнения, компилятор отклонит это. Значение всегда нужно захватывать в `const`:

```nim
import std/compilesettings

# ✅ Верно — вычисляется во время компиляции
const outputDir = querySetting(SingleValueSetting.outDir)
const backend   = querySetting(SingleValueSetting.backend)

# ❌ Неверно — нельзя вызывать compileTime-процедуру во время выполнения
proc show() =
  echo querySetting(SingleValueSetting.outDir)  # ошибка компиляции
```

### Примеры

```nim
import std/compilesettings

# 1. Вшить путь nimcache в бинарник для диагностики
const nimcache = querySetting(SingleValueSetting.nimcacheDir)
echo "Кэш компиляции находился в: ", nimcache

# 2. Убедиться, что используется конкретный бэкенд
const be = querySetting(SingleValueSetting.backend)
static:
  assert be == "c", "Эта библиотека поддерживает только C-бэкенд, получено: " & be

# 3. Вшить полный путь к проекту — полезно в генерируемом коде или сообщениях об ошибках
const projectFull = querySetting(SingleValueSetting.projectFull)
echo "Скомпилировано из: ", projectFull

# 4. Показать путь к C-компилятору для отладки сборки
const cc = querySetting(SingleValueSetting.ccompilerPath)
echo "Использованный C-компилятор: ", cc

# 5. Условная компиляция на основе менеджера памяти
const memMgr = querySetting(SingleValueSetting.mm)
when memMgr == "orc":
  echo "Работаем с управлением памятью ORC"
else:
  echo "Работаем с: ", memMgr
```

---

## `querySettingSeq`

```nim
proc querySettingSeq*(setting: MultipleValueSetting): seq[string] {.compileTime, noSideEffect.}
```

### Что делает

Возвращает текущее значение многозначной настройки компилятора в виде `seq[string]`. Как и `querySetting`, вызов перехватывается VM компилятора и не имеет стоимости во время выполнения. Все те же ограничения применимы: только `compileTime`, необходимо использовать в `const`, `static`, макросе или аналогичном контексте времени компиляции.

Результат — это обычный `seq[string]` Nim во время компиляции, поэтому по нему можно итерироваться, проверять его длину, индексироваться в него или передавать другим процедурам времени компиляции.

### Примеры

```nim
import std/compilesettings

# 1. Захватить и вшить активные пути поиска
const paths = querySettingSeq(MultipleValueSetting.searchPaths)
static:
  echo "Пути поиска во время компиляции:"
  for p in paths:
    echo "  ", p

# 2. Проверить, что необходимый путь Nimble присутствует
const nimblePaths = querySettingSeq(MultipleValueSetting.nimblePaths)
static:
  var found = false
  for p in nimblePaths:
    if "mypackage" in p:
      found = true
  assert found, "Путь Nimble для mypackage не найден — установлен ли пакет?"

# 3. Просмотреть пути включения C для диагностики сборки
const cincludes = querySettingSeq(MultipleValueSetting.cincludes)
const clibs     = querySettingSeq(MultipleValueSetting.clibs)
static:
  echo "C-включения (", cincludes.len, "):"
  for p in cincludes: echo "  -I", p
  echo "C-библиотеки (", clibs.len, "):"
  for l in clibs: echo "  -l", l

# 4. Вшить количество путей Nimble как константу времени выполнения
const nimblePathCount = querySettingSeq(MultipleValueSetting.nimblePaths).len
echo "Путей Nimble активных при компиляции: ", nimblePathCount
```

---

## Справочник полей `SingleValueSetting`

Подробное описание каждого члена перечисления.

---

### `arguments`
⚠️ *Экспериментальный*

Аргументы, переданные программе после флага `-r` при вызове компилятора как `nim r myfile.nim arg1 arg2`. Редко нужен в коде библиотек; полезен в инструментах метасборки.

---

### `outFile`
⚠️ *Экспериментальный*

Имя создаваемого выходного файла (например, `myapp` или `myapp.exe`). Не включает директорию — объедините с `outDir` для полного пути.

---

### `outDir`

Директория, в которую будет записан скомпилированный выходной файл. Управляется ключом компилятора `--outDir`. Полезен, когда ваша сборка генерирует дополнительные файлы (ресурсы, манифесты, конфигурацию), которые должны лежать рядом с бинарником.

```nim
const outputDir = querySetting(SingleValueSetting.outDir)
# Используйте outputDir, чтобы во время сборки создать сопутствующий файл рядом с бинарником
```

---

### `nimcacheDir`

Путь к директории `nimcache` — папке, где Nim хранит промежуточные C/C++ исходные файлы, объектные файлы и другие артефакты процесса компиляции. Полезен для диагностики и для пользовательских шагов сборки, которым нужно проверять или очищать сгенерированный C-код.

```nim
const nimcache = querySetting(SingleValueSetting.nimcacheDir)
static: echo "Промежуточные C-файлы находятся в: ", nimcache
```

---

### `projectName`

Имя компилируемого проекта (исходного файла) без расширения и директории. Для файла `src/myapp.nim` это будет `"myapp"`. Удобно для встраивания строк версий, имён модулей или генерации имён файлов.

```nim
const name = querySetting(SingleValueSetting.projectName)
const versionString = name & " v1.0"
```

---

### `projectPath`
⚠️ *Экспериментальный*

Некоторый путь, связанный с компилируемым проектом. Точное значение считается экспериментальным и может отличаться от `projectFull`. Для стабильного поведения предпочитайте `projectFull`.

---

### `projectFull`

**Полный абсолютный путь** к основному компилируемому исходному файлу. Это наиболее надёжный способ узнать, где находится источник; полезен для встраивания ссылок на исходник в генерируемый код или сообщения об ошибках.

```nim
const src = querySetting(SingleValueSetting.projectFull)
static: echo "Собирается: ", src
```

---

### `command`
⚠️ *Экспериментальный*

Подкоманда компилятора Nim, которая выполняется: `"c"`, `"cpp"`, `"doc"`, `"js"` и т.д. Позволяет ветвить компиляцию в зависимости от того, что было запрошено у компилятора.

```nim
const cmd = querySetting(SingleValueSetting.command)
when cmd == "doc":
  # включить объявления только для документации
  discard
```

---

### `commandLine`
⚠️ *Экспериментальный*

Полная строка командной строки, переданная компилятору Nim. Редко нужна в прикладном коде; наиболее полезна в продвинутом метапрограммировании и интеграции с системами сборки.

---

### `linkOptions`

Дополнительные параметры, которые будут переданы компоновщику. Они поступают из флагов `--passL` или прагм `.passl`. Полезны для проверки того, что ожидаемые флаги компоновщика активны.

```nim
const lo = querySetting(SingleValueSetting.linkOptions)
static: echo "Флаги компоновщика: ", lo
```

---

### `compileOptions`

Дополнительные параметры, которые будут переданы C/C++ компилятору. Они поступают из флагов `--passC` или прагм `.passc`. Полезны для диагностики неожиданных флагов компилятора.

```nim
const co = querySetting(SingleValueSetting.compileOptions)
static: echo "Флаги C-компилятора: ", co
```

---

### `ccompilerPath`

Путь в файловой системе к исполняемому файлу C или C++ компилятора, который будет вызывать Nim. Полезен, когда вашей сборке нужно точно знать, какая цепочка инструментов активна (например, чтобы вызвать её напрямую для пользовательского шага).

```nim
const cc = querySetting(SingleValueSetting.ccompilerPath)
static: echo "Цепочка инструментов: ", cc
```

---

### `backend`

Используемый бэкенд компиляции. Распространённые значения: `"c"`, `"cpp"`, `"objc"`, `"js"`. И `nim doc --backend:js`, и `nim js` дают `backend = "js"`. Это канонический способ ветвиться во время компиляции в зависимости от целевой среды.

```nim
const be = querySetting(SingleValueSetting.backend)
when be == "js":
  # JavaScript-специфичные импорты и логика
  import std/jsffi
```

---

### `libPath`

Абсолютный путь к стандартной библиотеке Nim (`--lib`). Доступен начиная с Nim 1.5.1. Полезен, когда инструментарий сборки должен напрямую находить исходные файлы стандартной библиотеки.

```nim
const stdlib = querySetting(SingleValueSetting.libPath)
static: echo "Стандартная библиотека находится в: ", stdlib
```

---

### `gc`
⛔ *Устаревший* — используйте `mm` вместо него.

---

### `mm`

Стратегия управления памятью, выбранная для этой сборки. Распространённые значения: `"orc"`, `"arc"`, `"refc"`, `"boehm"`, `"none"`. Используйте это, чтобы писать библиотеки, адаптирующие своё поведение, или выдающие понятную ошибку во время компиляции, если им требуется конкретный менеджер памяти.

```nim
const memMgr = querySetting(SingleValueSetting.mm)
static:
  assert memMgr in ["orc", "arc"],
    "Эта библиотека требует ARC или ORC, но получено: " & memMgr
```

---

## Справочник полей `MultipleValueSetting`

---

### `nimblePaths`

Список директорий, зарегистрированных Nimble в качестве корней пакетов. Каждая запись — это путь, под которым установлены пакеты Nimble. Полезен для проверки того, что ожидаемые зависимости доступны.

---

### `searchPaths`

Полный список директорий, в которых компилятор будет искать импортируемые модули (флаги `--path` / `-p` и значения по умолчанию). Проверка этого во время компиляции позволяет верифицировать среду разрешения модулей.

---

### `lazyPaths`
⚠️ *Экспериментальный*

Дополнительные пути, по которым поиск ведётся лениво (по требованию). Семантика считается экспериментальной.

---

### `commandArgs`

Список аргументов, переданных самому вызову компилятора Nim (не программе через `-r`). Полезен в макросах, которым нужно реагировать на конкретные флаги компилятора.

---

### `cincludes`

Список директорий поиска `#include`, передаваемых C/C++ компилятору (`--cincludes` / `-p` для C). Каждая запись соответствует флагу `-I<путь>`. Полезен, когда ваш Nim-код оборачивает C-библиотеки и вам нужно убедиться, что нужные заголовки находятся в пути включения.

---

### `clibs`

Список библиотек, которые будут скомпонованы в конечный бинарный файл через C-компоновщик. Каждая запись обычно соответствует флагу `-l<имя>`. Полезен для диагностики отсутствующих или неожиданных зависимостей времени компоновки.

---

## Практические паттерны

### Паттерн 1 — Бэкенд-специфичные импорты модулей

```nim
import std/compilesettings

const be = querySetting(SingleValueSetting.backend)

when be == "js":
  import std/jsffi
  proc alert(msg: cstring) {.importjs: "alert(#)".}
else:
  proc alert(msg: string) = echo msg

alert("Привет из бэкенда " & be & "!")
```

### Паттерн 2 — Принудительная проверка требований сборки во время компиляции

```nim
import std/compilesettings

# Принудительный менеджер памяти
const mm = querySetting(SingleValueSetting.mm)
static:
  assert mm == "orc",
    "MyLib требует --mm:orc. Активный: " & mm

# Проверить, что необходимый путь Nimble присутствует
const np = querySettingSeq(MultipleValueSetting.nimblePaths)
static:
  assert np.len > 0, "Пути Nimble не найдены — установлены ли пакеты?"
```

### Паттерн 3 — Встраивание метаданных сборки в бинарник

```nim
import std/compilesettings

const BuildInfo = (
  project:  querySetting(SingleValueSetting.projectName),
  backend:  querySetting(SingleValueSetting.backend),
  mm:       querySetting(SingleValueSetting.mm),
  outDir:   querySetting(SingleValueSetting.outDir),
)

proc printBuildInfo*() =
  echo "Проект  : ", BuildInfo.project
  echo "Бэкенд  : ", BuildInfo.backend
  echo "Память  : ", BuildInfo.mm
  echo "Выход   : ", BuildInfo.outDir
```

### Паттерн 4 — Диагностический дамп во время компиляции

```nim
import std/compilesettings

static:
  echo "=== Диагностика времени компиляции ==="
  echo "Проект   : ", querySetting(SingleValueSetting.projectFull)
  echo "Бэкенд   : ", querySetting(SingleValueSetting.backend)
  echo "Память   : ", querySetting(SingleValueSetting.mm)
  echo "Nimcache : ", querySetting(SingleValueSetting.nimcacheDir)
  let sp = querySettingSeq(MultipleValueSetting.searchPaths)
  echo "Пути поиска (", sp.len, "):"
  for p in sp: echo "  ", p
```
