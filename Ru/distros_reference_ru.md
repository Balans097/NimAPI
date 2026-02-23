# Справочник модуля `distros`

**Модуль:** `distros` (часть стандартной библиотеки Nim)   
**Назначение:** Определение текущего дистрибутива Linux (или другой ОС) во время выполнения и генерация правильной команды нативного менеджера пакетов для установки внешней зависимости.

---

## Обзор

Когда пакет Nim зависит от C-библиотеки или системного инструмента, он не может установить эту зависимость сам — ему нужно попросить пользователя запустить менеджер пакетов своей ОС. Трудность в том, что каждый дистрибутив Linux использует свой менеджер пакетов (`apt-get`, `pacman`, `yum`, `emerge`, …), и каждый дистрибутив может давать одной и той же библиотеке разные имена.

`distros` решает обе части задачи:

1. **Определение** — `detectOs` говорит, на какой ОС или дистрибутиве запущена программа.
2. **Генерация команды** — `foreignDep` / `foreignDepInstallCmd` формируют точную команду `apt-get install …`, `pacman -S …` или её аналог для текущей системы.

Модуль в основном используется из `config.nims` / скриптов сборки Nimble, но работает в любой программе Nim.

### Стратегия определения

Для общих семейств ОС (Windows, Linux, macOS, BSD, POSIX) определение выполняется через проверки `defined()` во время компиляции — быстро и без накладных расходов.

Для конкретных дистрибутивов Linux модуль запрашивает до четырёх команд оболочки, кешируя каждый результат после первого вызова:

| Команда | Что раскрывает |
|---|---|
| `cat /etc/os-release \| grep ^ID=` | Официальный ID дистрибутива (`ubuntu`, `arch`, `fedora`, …) |
| `lsb_release -d` | Читаемая строка-описание |
| `uname -a` | Ядро + архитектура + имя хоста |
| `hostnamectl` | Имя ОС/дистрибутива на основе systemd |

NixOS и среды сборки Nix определяются через проверку переменных окружения `NIX_BUILD_TOP` и `__NIXOS_SET_ENVIRONMENT_DONE`, без запуска команд оболочки.

---

## Экспортируемый API

| Символ | Вид | Описание |
|---|---|---|
| `Distribution` | enum | Все известные ОС и дистрибутивы Linux |
| `LacksDevPackages` | const set | Дистрибутивы без разбиения на `-dev`-пакеты |
| `foreignDeps` | var (seq) | Накопленный список команд внешних зависимостей |
| `detectOs(d)` | template | Определить, совпадает ли текущая ОС с `d` |
| `foreignDepInstallCmd(pkg)` | proc | Вернуть команду установки `pkg` для текущей ОС |
| `foreignCmd(cmd, requiresSudo)` | proc | Зарегистрировать произвольную команду оболочки как зависимость |
| `foreignDep(pkg)` | proc | Зарегистрировать имя пакета как внешнюю зависимость |
| `echoForeignDeps()` | proc | Вывести все зарегистрированные зависимости в stdout |

---

## `Distribution`

```nim
type Distribution* {.pure.} = enum
  Windows, Posix, MacOSX, Linux,
  Ubuntu, Debian, Gentoo, Fedora, RedHat,
  OpenSUSE, Manjaro, Alpine, ArchLinux, NixOS,
  BSD, FreeBSD, NetBSD, OpenBSD, DragonFlyBSD,
  Haiku,
  ... # ещё ~80 значений
```

### Что это такое

`{.pure.}` перечисление, содержащее все семейства ОС и дистрибутивы Linux, о которых знает модуль. Поскольку перечисление `{.pure.}`, каждое значение при прямом использовании в коде нужно квалифицировать: `Distribution.Ubuntu`. Однако шаблон `detectOs` добавляет этот квалификатор автоматически, поэтому на практике вы его никогда не вводите.

Перечисление охватывает:

- **Общие семейства:** `Windows`, `Posix`, `MacOSX`, `Linux`, `BSD` — определяются во время компиляции через `defined()`.
- **Варианты BSD:** `FreeBSD`, `NetBSD`, `OpenBSD`, `DragonFlyBSD` — через `uname`.
- **Основные дистрибутивы Linux:** `Ubuntu`, `Debian`, `Fedora`, `RedHat`, `ArchLinux`, `Gentoo`, `Alpine`, `NixOS` и многие другие — через `/etc/os-release`, `lsb_release`, `uname` или `hostnamectl`.
- **Haiku** — через `defined(haiku)`.

### Пример

```nim
import distros

# Перечислить все значения
for d in Distribution:
  echo d
```

---

## `LacksDevPackages`

```nim
const LacksDevPackages* = {
  Distribution.Gentoo,
  Distribution.Slackware,
  Distribution.ArchLinux,
  Distribution.Artix,
  Distribution.Antergos,
  Distribution.BlackArch,
  Distribution.ArchBang
}
```

### Что это такое

Константное множество значений `Distribution`, идентифицирующее дистрибутивы, где пакеты с заголовочными файлами для разработки **не отделены** от пакетов времени выполнения. В Debian/Ubuntu-подобных системах библиотека поставляется двумя пакетами: `libfoo` (runtime) и `libfoo-dev` (заголовки + статическая библиотека). В Arch Linux, Gentoo и Slackware есть только один пакет, содержащий и то, и другое.

### Когда использовать

Обращайтесь к `LacksDevPackages`, когда нужно решить, следует ли запрашивать суффикс `-dev`:

```nim
import distros

proc installLibfoo() =
  if detectOs(Ubuntu) or detectOs(Debian):
    foreignDep "libfoo-dev"   # нужен -dev вариант
  elif detectOs(ArchLinux):
    foreignDep "libfoo"       # Arch поставляет заголовки в том же пакете

# Или обобщённо:
proc devPkg(baseName: string): string =
  for d in LacksDevPackages:
    if detectOsImpl(d): return baseName
  return baseName & "-dev"
```

---

## `foreignDeps`

```nim
var foreignDeps*: seq[string] = @[]
```

### Что это такое

Модульная последовательность, накапливающая каждую команду оболочки, зарегистрированную через `foreignCmd` или `foreignDep`. Каждый элемент — готовая к выполнению строка, например `"sudo apt-get install libssl-dev"`.

Эта переменная определяется только тогда, когда модуль **не** скомпилирован в Nimble (`not defined(nimble)`). Внутри скрипта Nimble переменную `foreignDeps` предоставляет сам Nimble через `nimscriptapi`.

Вы можете итерироваться по `foreignDeps` самостоятельно или вызвать `echoForeignDeps()` для вывода всех команд.

### Пример

```nim
import distros

foreignDep "zlib"
foreignDep "openssl"

echo "Запустите следующие команды для установки зависимостей:"
for cmd in foreignDeps:
  echo "  ", cmd
```

---

## `detectOs`

```nim
template detectOs*(d: untyped): bool
```

### Что делает

Определяет, работает ли программа в данный момент на ОС или дистрибутиве, названном `d`. Возвращает `true`, если текущее окружение совпадает, `false` — в противном случае.

`detectOs` — **шаблон**, а не процедура, что даёт два практических преимущества:

- **Не нужен префикс `Distribution.`** Пишете `detectOs(Ubuntu)` вместо `detectOs(Distribution.Ubuntu)`.
- **Короткое замыкание во время компиляции для семейств ОС.** Для `Windows`, `Linux`, `MacOSX`, `Posix` и `BSD` проверка сводится к вызову `defined()` и вычисляется во время компиляции без накладных расходов во время выполнения.

Для специфических дистрибутивов Linux шаблон делегирует в `detectOsImpl`, которая запускает соответствующие команды оболочки (кешируя их после первого вызова).

### Детали определения по дистрибутивам

| Дистрибутив | Метод определения |
|---|---|
| `Windows`, `Linux`, `MacOSX`, `Posix`, `BSD` | `defined(...)` во время компиляции |
| `FreeBSD`, `NetBSD`, `OpenBSD` | `$d in uname()` |
| `Gentoo` | `"-Gentoo "` в `uname()` |
| `Ubuntu`, `Debian`, `Fedora`, `Alpine`, `Void`, `CentOS`, `Mageia`, `Zorin`, `Elementary`, `OpenMandriva` | имя дистрибутива в поле ID файла `/etc/os-release` |
| `RedHat` | `"rhel"` в поле ID файла `/etc/os-release` |
| `ArchLinux` | `"arch"` в поле ID файла `/etc/os-release` |
| `Artix` | `"artix"` в поле ID файла `/etc/os-release` |
| `NixOS` | наличие переменной окружения `NIX_BUILD_TOP` или `__NIXOS_SET_ENVIRONMENT_DONE` |
| `OpenSUSE` | `"suse"` в `uname()` или `lsb_release` |
| `GoboLinux` | `"-Gobo "` в `uname()` |
| `Solaris` | `"sun"` или `"solaris"` в `uname()` |
| `Haiku` | `defined(haiku)` |
| Все остальные дистрибутивы | имя проверяется во всех четырёх источниках (`os-release`, `lsb_release`, `uname`, `hostnamectl`) |

### Когда использовать

Вызывайте `detectOs` в начале любой платформозависимой ветки в скрипте Nimble или `config.nims`, чтобы выбрать правильные имена пакетов и команды установки для текущей системы.

### Примеры

```nim
import distros

# Пример 1 — простой двухветвевой выбор
if detectOs(Ubuntu) or detectOs(Debian):
  foreignDep "libssl-dev"
elif detectOs(ArchLinux) or detectOs(Manjaro):
  foreignDep "openssl"
elif detectOs(Fedora):
  foreignDep "openssl-devel"

# Пример 2 — вложенные проверки для разных имён пакетов
if detectOs(Linux):
  if detectOs(Alpine):
    foreignDep "sqlite-dev"
  elif detectOs(Fedora) or detectOs(RedHat) or detectOs(CentOS):
    foreignDep "sqlite-devel"
  else:
    foreignDep "libsqlite3-dev"
elif detectOs(MacOSX):
  foreignDep "sqlite3"
elif detectOs(Windows):
  foreignDep "sqlite"  # через Chocolatey

# Пример 3 — только семейство ОС
if detectOs(Windows):
  echo "Работает на Windows"
elif detectOs(MacOSX):
  echo "Работает на macOS"
elif detectOs(Linux):
  echo "Работает на Linux"

# Пример 4 — NixOS / среда Nix
if detectOs(NixOS):
  foreignDep "openssl"       # nix-env -i, sudo не нужен
else:
  foreignDep "libssl-dev"
```

---

## `foreignDepInstallCmd`

```nim
proc foreignDepInstallCmd*(foreignPackageName: string): (string, bool)
```

### Что делает

Принимает имя пакета и возвращает кортеж из:
- Полной команды оболочки для установки этого пакета в текущей ОС (например, `"apt-get install libz-dev"`).
- `bool`, указывающего, требует ли команда прав суперпользователя/администратора (`true` = нужен sudo/root, `false` = можно запустить как обычный пользователь).

Команда выбирается на основе определённой ОС/дистрибутива:

| ОС / Дистрибутив | Команда | Нужен root |
|---|---|---|
| Windows | `choco install <pkg>` | `false` |
| BSD (общий) | `ports install <pkg>` | `true` |
| Ubuntu, Debian, Elementary, KNOPPIX, SteamOS | `apt-get install <pkg>` | `true` |
| Gentoo | `emerge install <pkg>` | `true` |
| Fedora | `yum install <pkg>` | `true` |
| RedHat | `rpm install <pkg>` | `true` |
| OpenSUSE | `yast -i <pkg>` | `true` |
| Slackware | `installpkg <pkg>` | `true` |
| OpenMandriva | `urpmi <pkg>` | `true` |
| ZenWalk | `netpkg install <pkg>` | `true` |
| NixOS | `nix-env -i <pkg>` | `false` |
| Solaris, FreeBSD | `pkg install <pkg>` | `true` |
| NetBSD, OpenBSD | `pkg_add <pkg>` | `true` |
| PCLinuxOS | `rpm -ivh <pkg>` | `true` |
| ArchLinux, Manjaro, Artix | `pacman -S <pkg>` | `true` |
| Void | `xbps-install <pkg>` | `true` |
| Haiku | `pkgman install <pkg>` | `true` |
| macOS / другие | `brew install <pkg>` | `false` |
| Неизвестный Linux | `<your package manager here> install <pkg>` | `true` |

### Когда использовать

Вызывайте `foreignDepInstallCmd`, когда вам нужна строка команды для отображения, логирования или построения более сложного скрипта установки. Если вы просто хотите зарегистрировать зависимость стандартным образом — вызывайте `foreignDep`, который обращается к `foreignDepInstallCmd` внутри.

### Примеры

```nim
import distros

# Посмотреть, какая команда была бы использована, не регистрируя её
let (cmd, needsSudo) = foreignDepInstallCmd("libpng-dev")
echo "Команда установки: ", cmd
echo "Нужен root: ", needsSudo

# Изменить команду перед регистрацией
let (baseCmd, sudo) = foreignDepInstallCmd("ffmpeg")
if needsSudo:
  foreignCmd("sudo " & baseCmd)
else:
  foreignCmd(baseCmd)
```

---

## `foreignCmd`

```nim
proc foreignCmd*(cmd: string; requiresSudo = false)
```

### Что делает

Добавляет полностью сформированную команду оболочки в список `foreignDeps`. Если `requiresSudo` равно `true`, перед сохранением к `cmd` добавляется префикс `"sudo "`.

Это низкоуровневая процедура регистрации. Все высокоуровневые процедуры (`foreignDep` и косвенно `foreignDepInstallCmd`) в конечном счёте вызывают `foreignCmd`.

Внутри скрипта Nimble команда добавляется в переменную Nimble `nimscriptapi.foreignDeps`, а не в модульную переменную `foreignDeps`.

### Когда использовать

Используйте `foreignCmd`, когда:
- Нужно зарегистрировать команду, которая не является простой установкой пакета (например, `pip install`, `gem install` или многошаговый скрипт сборки).
- Строка команды уже построена вами и нужно просто её записать.
- Требуется полный контроль над точной строкой в списке зависимостей.

### Примеры

```nim
import distros

# Пример 1 — зарегистрировать зависимость Python
foreignCmd("pip install numpy", requiresSudo = false)

# Пример 2 — зарегистрировать шаг ручной сборки (нужен root)
foreignCmd("make install", requiresSudo = true)

# Пример 3 — платформозависимая пользовательская команда
if detectOs(Ubuntu):
  foreignCmd("add-apt-repository ppa:example/ppa", requiresSudo = true)
  foreignCmd("apt-get update", requiresSudo = true)
  foreignCmd("apt-get install example-lib-dev", requiresSudo = true)
elif detectOs(MacOSX):
  foreignCmd("brew tap example/tap")
  foreignCmd("brew install example-lib")
```

---

## `foreignDep`

```nim
proc foreignDep*(foreignPackageName: string)
```

### Что делает

Основной высокоуровневый API для регистрации внешней (не-Nim) зависимости от пакета. Получив только имя пакета, она:

1. Вызывает `foreignDepInstallCmd(foreignPackageName)` для определения правильной команды установки и необходимости sudo.
2. Передаёт оба значения в `foreignCmd`, который добавляет `"sudo "` при необходимости и помещает результат в `foreignDeps`.

При вызове `foreignDep` не нужно знать текущий дистрибутив — определение происходит автоматически внутри `foreignDepInstallCmd`.

### Когда использовать

`foreignDep` — первый выбор в большинстве случаев: вы знаете имя пакета для каждого дистрибутива, вызываете `foreignDep` внутри подходящей ветки `detectOs`, а модуль делает всё остальное.

Единственное, чего `foreignDep` не может сделать за вас — это обработать дистрибутивы, где одна и та же библиотека имеет разные имена пакетов. Это нужно делать самостоятельно, вызывая `foreignDep` с правильным именем внутри корректной ветки `detectOs`.

### Примеры

```nim
import distros

# Пример 1 — базовое использование в скрипте Nimble
if detectOs(Ubuntu) or detectOs(Debian):
  foreignDep "libblas-dev"
  foreignDep "liblapack-dev"
elif detectOs(Fedora) or detectOs(RedHat):
  foreignDep "blas-devel"
  foreignDep "lapack-devel"
elif detectOs(ArchLinux):
  foreignDep "blas"
  foreignDep "lapack"

# Пример 2 — NixOS не требует sudo, но имя отличается
if detectOs(NixOS):
  foreignDep "openblas"
else:
  foreignDep "libopenblas-dev"

# Пример 3 — после регистрации смотрим, что собралось
foreignDep "zlib"
for cmd in foreignDeps:
  echo cmd
# На Ubuntu выводит: sudo apt-get install zlib
```

---

## `echoForeignDeps`

```nim
proc echoForeignDeps*()
```

### Что делает

Перебирает все команды, хранящиеся в `foreignDeps`, и выводит каждую в стандартный вывод через `echo`. Ничего больше.

Это стандартный способ показать накопленный список зависимостей в конце задачи установки Nimble, производя вывод для пользователя вида:

```
sudo apt-get install libblas-dev
sudo apt-get install libvoodoo
```

### Когда использовать

Вызывайте `echoForeignDeps` в конце хука `before install` или `after install` Nimble, или в конце задачи `config.nims`, после того как все вызовы `foreignDep` / `foreignCmd` сделаны. Нет смысла вызывать её раньше, чем зарегистрированы все зависимости.

### Пример

```nim
import distros

# Регистрируем зависимости в зависимости от текущей ОС
if detectOs(Ubuntu) or detectOs(Debian):
  foreignDep "libssl-dev"
  foreignDep "libffi-dev"
elif detectOs(Fedora):
  foreignDep "openssl-devel"
  foreignDep "libffi-devel"
elif detectOs(ArchLinux):
  foreignDep "openssl"
  foreignDep "libffi"
elif detectOs(MacOSX):
  foreignDep "openssl"
  foreignDep "libffi"
elif detectOs(Windows):
  foreignDep "openssl"
  foreignDep "libffi"

echo "Для завершения установки выполните следующие команды:"
echoForeignDeps()
# Пример вывода на Ubuntu:
# Для завершения установки выполните следующие команды:
# sudo apt-get install libssl-dev
# sudo apt-get install libffi-dev
```

---

## Полный рабочий пример — скрипт сборки Nimble

```nim
# В файле config.nims или .nimble вашего пакета:

import distros

task checkDeps, "Проверить и показать необходимые системные зависимости":

  # ── OpenSSL ────────────────────────────────────────────────────────────────
  if detectOs(Ubuntu) or detectOs(Debian) or detectOs(Elementary):
    foreignDep "libssl-dev"
  elif detectOs(Fedora) or detectOs(RedHat) or detectOs(CentOS):
    foreignDep "openssl-devel"
  elif detectOs(ArchLinux) or detectOs(Manjaro) or detectOs(Artix):
    foreignDep "openssl"
  elif detectOs(Alpine):
    foreignDep "openssl-dev"
  elif detectOs(NixOS):
    foreignDep "openssl"          # nix-env -i, sudo не нужен
  elif detectOs(MacOSX):
    foreignDep "openssl"          # brew, sudo не нужен
  elif detectOs(Windows):
    foreignDep "openssl"          # choco, sudo не нужен

  # ── SQLite ─────────────────────────────────────────────────────────────────
  if detectOs(Ubuntu) or detectOs(Debian):
    foreignDep "libsqlite3-dev"
  elif detectOs(Fedora):
    foreignDep "sqlite-devel"
  elif detectOs(ArchLinux):
    foreignDep "sqlite"
  elif detectOs(Alpine):
    foreignDep "sqlite-dev"
  elif not detectOs(Windows):     # Windows поставляет SQLite в комплекте; остальные — через brew/pkg
    foreignDep "sqlite"

  # ── Пользовательский шаг сборки ────────────────────────────────────────────
  if detectOs(Linux) and not (detectOs(NixOS)):
    foreignCmd "ldconfig", requiresSudo = true

  # ── Вывод ──────────────────────────────────────────────────────────────────
  if foreignDeps.len > 0:
    echo "\nДля завершения установки выполните следующие команды:"
    echoForeignDeps()
  else:
    echo "Дополнительные системные зависимости не требуются."
```

---

## Замечания о дизайне и оговорки

**Команды оболочки запускаются лениво и кешируются.** Четыре команды определения (`uname`, `cat /etc/os-release`, `lsb_release`, `hostnamectl`) выполняются только при первой необходимости, а их вывод хранится в модульных строковых переменных. Последующие вызовы `detectOs` для любого дистрибутива из того же источника переиспользуют кешированную строку. Это делает многократные вызовы `detectOs` дешёвыми.

**Совместимость с NimScript.** Модуль работает как в скомпилированных программах Nim, так и в NimScript (`config.nims`). В NimScript вместо `execProcess` для запуска команд оболочки используется `gorge`.

**Имена пакетов — ваша ответственность.** `foreignDep` принимает любую переданную строку. Если вы передадите неправильное имя пакета для дистрибутива, сгенерированная команда будет синтаксически правильной, но семантически неверной. Всегда проверяйте на реальном целевом дистрибутиве.

**`foreignDeps` не определена под Nimble.** Когда модуль используется из скрипта Nimble (`defined(nimble)` истинно), `foreignDeps` — это переменная самого Nimble из `nimscriptapi`. Прямое обращение к модульной `foreignDeps` из скрипта Nimble может вести себя не так, как ожидается.

**Неизвестные дистрибутивы попадают в заглушку.** Если ни один из известных дистрибутивов Linux не совпал в `foreignDepInstallCmd`, возвращаемая команда буквально содержит `"<your package manager here> install <pkg>"`. Это наглядное напоминание пользователю, что нужную команду придётся найти самостоятельно.
