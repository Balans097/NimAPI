# Модуль `terminal` Nim — Полный справочник

> **Модуль:** `std/terminal`  
> **Назначение:** Управление терминалом/консолью — цвета, стили, движение курсора, очистка экрана, размер терминала, ввод паролей и многое другое.  
> **Реализация:** ANSI escape-последовательности на UNIX/macOS; Windows Console API на Windows.

---

## ⚠️ Важные примечания

- **Изменения стиля постоянны** — они сохраняются после завершения программы. Используйте `exitprocs.addExitProc(resetAttributes)` для восстановления настроек по умолчанию.
- **Скрытый курсор** — если вы вызвали `hideCursor`, обязательно восстановите курсор через `showCursor` перед выходом.
- **Два вида функций стилизации:**
  - `styledWriteLine`, `styledEcho`, `styledWrite` — **временный** эффект, автосброс после вызова.
  - `setForegroundColor`, `setBackgroundColor`, `setStyle` — **постоянный** эффект, нужно вручную вызывать `resetAttributes`.

---

## Содержание

1. [Константы](#константы)
2. [Типы](#типы)
3. [Размер терминала](#размер-терминала)
4. [Позиция курсора](#позиция-курсора)
5. [Видимость курсора](#видимость-курсора)
6. [Движение курсора](#движение-курсора)
7. [Очистка экрана и строки](#очистка-экрана-и-строки)
8. [Стили текста](#стили-текста)
9. [Цвета](#цвета)
10. [True Color (24-bit)](#true-color-24-bit)
11. [Вспомогательные ANSI-функции](#вспомогательные-ansi-функции)
12. [Стилизованный вывод](#стилизованный-вывод)
13. [Ввод](#ввод)
14. [Утилиты](#утилиты)
15. [Шаблоны-сокращения для stdout](#шаблоны-сокращения-для-stdout)
16. [Сводная таблица доступности по платформам](#сводная-таблица-доступности-по-платформам)
17. [Полные примеры](#полные-примеры)

---

## Константы

### `ansiResetCode`

```nim
const ansiResetCode* = "\e[0m"
```

ANSI escape-последовательность для сброса всех атрибутов терминала (цвет, яркость, стиль) к значениям по умолчанию. Можно записывать напрямую в файл или встраивать в строки.

**Пример:**
```nim
import std/terminal
echo "цветной текст" & ansiResetCode & "обычный текст"
```

---

## Типы

### `Style`

```nim
type Style* = enum
  styleBright = 1,        ## яркий/жирный текст
  styleDim,               ## тусклый текст
  styleItalic,            ## курсив (на некоторых терминалах — инверсия)
  styleUnderscore,        ## подчёркнутый текст
  styleBlink,             ## мигающий текст
  styleBlinkRapid,        ## быстрое мигание (поддерживается не везде)
  styleReverse,           ## инверсия цветов переднего и заднего плана
  styleHidden,            ## скрытый/невидимый текст
  styleStrikethrough      ## зачёркнутый текст
```

Перечисление модификаторов стиля текста. Используется с `setStyle`, `styledWrite`, `styledWriteLine` и `styledEcho`.

---

### `ForegroundColor`

```nim
type ForegroundColor* = enum
  fgBlack = 30,   ## чёрный
  fgRed,          ## красный
  fgGreen,        ## зелёный
  fgYellow,       ## жёлтый
  fgBlue,         ## синий
  fgMagenta,      ## пурпурный
  fgCyan,         ## циановый
  fgWhite,        ## белый
  fg8Bit,         ## 256-цветный (не поддерживается — используйте `enableTrueColors`)
  fgDefault       ## цвет переднего плана по умолчанию
```

Стандартное 8-цветное перечисление цветов переднего плана. Значения — ANSI-коды цветов (30–39).

---

### `BackgroundColor`

```nim
type BackgroundColor* = enum
  bgBlack = 40,   ## чёрный
  bgRed,          ## красный
  bgGreen,        ## зелёный
  bgYellow,       ## жёлтый
  bgBlue,         ## синий
  bgMagenta,      ## пурпурный
  bgCyan,         ## циановый
  bgWhite,        ## белый
  bg8Bit,         ## 256-цветный (не поддерживается — используйте `enableTrueColors`)
  bgDefault       ## цвет фона по умолчанию
```

Стандартное 8-цветное перечисление цветов фона. Значения — ANSI-коды (40–49).

---

### `TerminalCmd`

```nim
type TerminalCmd* = enum
  resetStyle,   ## сбросить все атрибуты текста
  fgColor,      ## следующий аргумент Color будет цветом переднего плана
  bgColor       ## следующий аргумент Color будет цветом фона
```

Специальные токены команд для использования в аргументах `styledWrite`, `styledWriteLine` и `styledEcho`. `fgColor` и `bgColor` служат «переключателями режима» перед значением типа `Color`.

**Пример:**
```nim
import std/terminal
styledEcho styleBright, fgGreen, "[OK]", resetStyle, " Готово"
```

---

## Размер терминала

### `terminalWidth`

```nim
proc terminalWidth*(): int
```

Возвращает ширину терминала в символьных столбцах. Последовательно пробует несколько источников:

- **POSIX:** переменная `$COLUMNS` → `ioctl` на stdin/stdout/stderr → управляющий терминал → по умолчанию `80`
- **Windows:** `GetConsoleScreenBufferInfo` на stdin/stdout/stderr → по умолчанию `80`

**Возвращает:** Ширину в столбцах. Никогда не возвращает 0 (минимум 80).

**Пример:**
```nim
import std/terminal
echo "Ширина терминала: ", terminalWidth(), " столбцов"
```

---

### `terminalHeight`

```nim
proc terminalHeight*(): int
```

Возвращает высоту терминала в строках. Последовательно пробует несколько источников:

- **POSIX:** переменная `$LINES` → `ioctl` на stdin/stdout/stderr → управляющий терминал → по умолчанию `0`
- **Windows:** `GetConsoleScreenBufferInfo` → по умолчанию `0`

**Возвращает:** Высоту в строках. Возвращает `0`, если определить не удалось.

**Пример:**
```nim
import std/terminal
let h = terminalHeight()
if h > 0:
  echo "Высота терминала: ", h, " строк"
else:
  echo "Не удалось определить высоту"
```

---

### `terminalSize`

```nim
proc terminalSize*(): tuple[w, h: int]
```

Возвращает ширину и высоту терминала в виде именованного кортежа. Удобная обёртка над `terminalWidth()` и `terminalHeight()`.

**Возвращает:** `(w: int, h: int)` — ширина и высота.

**Пример:**
```nim
import std/terminal
let (w, h) = terminalSize()
echo "Размер: ", w, "×", h

# Адаптивный расчёт:
let halfWidth = terminalWidth() div 2
```

---

### `terminalWidthIoctl`

```nim
# POSIX:
proc terminalWidthIoctl*(fds: openArray[int]): int
# Windows:
proc terminalWidthIoctl*(handles: openArray[Handle]): int
```

Низкоуровневый вспомогательный вызов: запрашивает ширину терминала через `ioctl` (POSIX) или `GetConsoleScreenBufferInfo` (Windows) для указанных дескрипторов/дескрипторов файлов. Возвращает `0`, если ни один не ответил.

**Пример:**
```nim
import std/terminal
# Попытка для конкретных дескрипторов:
let w = terminalWidthIoctl([1, 2])  # stdout и stderr
echo "Ширина: ", w
```

---

### `terminalHeightIoctl`

```nim
# POSIX:
proc terminalHeightIoctl*(fds: openArray[int]): int
# Windows:
proc terminalHeightIoctl*(handles: openArray[Handle]): int
```

Низкоуровневый запрос высоты терминала через указанные дескрипторы. Возвращает `0`, если ни один не ответил.

**Пример:**
```nim
import std/terminal
let h = terminalHeightIoctl([0, 1, 2])
echo "Высота: ", h
```

---

## Позиция курсора

### `getCursorPos`

```nim
proc getCursorPos*(): tuple[x, y: int]
  {.raises: [ValueError, IOError, OSError].}
```

Возвращает текущую позицию курсора в виде кортежа `(x, y)` (отсчёт от нуля).

- **POSIX:** записывает ANSI escape-код `\e[6n` в stdout и читает ответ из stdin в режиме raw. Требует настоящего терминала (не перенаправления).
- **Windows:** вызывает `GetConsoleScreenBufferInfo`.

**Возвращает:** `(x, y)` — столбец и строка курсора (0-based).

**Исключения:**
- `ValueError` — если ответ терминала некорректен или неполон
- `IOError` — при ошибке ввода-вывода
- `OSError` — при ошибке Windows API

**Пример:**
```nim
import std/terminal
let (x, y) = getCursorPos()
echo "Курсор: столбец ", x, ", строка ", y
```

---

### `setCursorPos`

```nim
proc setCursorPos*(f: File, x, y: int)
template setCursorPos*(x, y: int)    # сокращение для stdout
```

Перемещает курсор в абсолютную позицию `(x, y)`. Начало координат `(0, 0)` — верхний левый угол терминала.

**Параметры:**
- `f` — файл вывода (например, `stdout`, `stderr`)
- `x` — столбец (0-based)
- `y` — строка (0-based)

**Пример:**
```nim
import std/terminal
setCursorPos(0, 0)          # перейти в левый верхний угол (stdout)
stdout.setCursorPos(10, 5)  # столбец 10, строка 5
```

---

### `setCursorXPos`

```nim
proc setCursorXPos*(f: File, x: int)
template setCursorXPos*(x: int)    # сокращение для stdout
```

Перемещает курсор в столбец `x` текущей строки. Позиция по вертикали (Y) не изменяется.

**Параметры:**
- `f` — файл вывода
- `x` — столбец (0-based)

**Пример:**
```nim
import std/terminal
stdout.setCursorXPos(0)   # перейти в начало текущей строки
setCursorXPos(20)         # перейти в столбец 20 в stdout
```

---

### `setCursorYPos` *(только Windows)*

```nim
proc setCursorYPos*(f: File, y: int)         # только Windows
template setCursorYPos*(x: int)              # сокращение для stdout, только Windows
```

Перемещает курсор в строку `y`, сохраняя столбец. **Только Windows** — не поддерживается на UNIX.

**Пример:**
```nim
import std/terminal
when defined(windows):
  stdout.setCursorYPos(10)
```

---

## Видимость курсора

### `hideCursor`

```nim
proc hideCursor*(f: File)
template hideCursor*()    # сокращение для stdout
```

Делает курсор терминала невидимым. Курсор продолжает перемещаться, но не отображается.

> **Важно:** Всегда восстанавливайте видимость курсора перед выходом. Используйте `showCursor` или зарегистрируйте его через `exitprocs.addExitProc`.

**Пример:**
```nim
import std/[terminal, exitprocs]
hideCursor()
addExitProc(proc() = showCursor())   # восстановить при выходе

# ... анимация ...

showCursor()
```

---

### `showCursor`

```nim
proc showCursor*(f: File)
template showCursor*()    # сокращение для stdout
```

Делает курсор снова видимым после `hideCursor`.

**Пример:**
```nim
import std/terminal
hideCursor()
# ... работа ...
showCursor()
```

---

## Движение курсора

Все процедуры движения существуют в двух формах: принимающей явный `File` и шаблоне-сокращении для `stdout`.

### `cursorUp`

```nim
proc cursorUp*(f: File, count = 1)
template cursorUp*(count = 1)    # сокращение для stdout
```

Перемещает курсор вверх на `count` строк. Столбец не изменяется.

**Параметры:**
- `count` — количество строк для движения вверх (по умолчанию: 1)

**Пример:**
```nim
import std/terminal
stdout.cursorUp(3)         # переместить на 3 строки вверх
cursorUp()                 # переместить на 1 строку вверх в stdout
```

---

### `cursorDown`

```nim
proc cursorDown*(f: File, count = 1)
template cursorDown*(count = 1)    # сокращение для stdout
```

Перемещает курсор вниз на `count` строк.

**Пример:**
```nim
import std/terminal
cursorDown(2)
write(stdout, "На две строки ниже исходной позиции")
```

---

### `cursorForward`

```nim
proc cursorForward*(f: File, count = 1)
template cursorForward*(count = 1)    # сокращение для stdout
```

Перемещает курсор вправо на `count` столбцов.

**Параметры:**
- `count` — количество столбцов для движения вправо (по умолчанию: 1)

**Пример:**
```nim
import std/terminal
write(stdout, ">>")
cursorForward(5)
write(stdout, "<<")    # печатается на 5 позиций правее
```

---

### `cursorBackward`

```nim
proc cursorBackward*(f: File, count = 1)
template cursorBackward*(count = 1)    # сокращение для stdout
```

Перемещает курсор влево на `count` столбцов.

**Параметры:**
- `count` — количество столбцов для движения влево (по умолчанию: 1)

**Пример:**
```nim
import std/terminal
write(stdout, "Привет!")
cursorBackward(7)         # вернуться на 7 позиций назад
write(stdout, "Мир!   ") # перезаписать "Привет!"
```

---

## Очистка экрана и строки

### `eraseLine`

```nim
proc eraseLine*(f: File)
template eraseLine*()    # сокращение для stdout
```

Стирает всю текущую строку (от столбца 0 до конца) и перемещает курсор в начало строки (`x = 0`). Незаменимо для прогресс-баров и спиннеров.

**Пример:**
```nim
import std/[terminal, os]
for i in 1..10:
  write(stdout, "Прогресс: " & $i & "/10")
  stdout.flushFile()
  sleep(200)
  eraseLine()
write(stdout, "Готово!\n")
```

---

### `eraseScreen`

```nim
proc eraseScreen*(f: File)
template eraseScreen*()    # сокращение для stdout
```

Очищает весь экран, используя текущий цвет фона, и перемещает курсор в начальную позицию (верхний левый угол).

**Пример:**
```nim
import std/terminal
eraseScreen()
setCursorPos(0, 0)
write(stdout, "Чистый экран!")
```

---

## Стили текста

### `setStyle`

```nim
proc setStyle*(f: File, style: set[Style])
template setStyle*(style: set[Style])    # сокращение для stdout
```

Применяет один или несколько стилей текста к `f`. Эффект сохраняется до явного вызова `resetAttributes`.

**Параметры:**
- `style` — `set[Style]` (можно комбинировать несколько стилей)

> **Примечание:** На Windows поддерживаются только `styleBright`, `styleBlink`, `styleReverse` и `styleUnderscore`; остальные игнорируются.

**Пример:**
```nim
import std/terminal
stdout.setStyle({styleBright, styleUnderscore})
write(stdout, "Жирный и подчёркнутый")
resetAttributes()

setStyle({styleItalic, styleBlink})
write(stdout, "Курсив с миганием")
resetAttributes()
```

---

### `writeStyled`

```nim
proc writeStyled*(txt: string, style: set[Style] = {styleBright})
```

Записывает `txt` в stdout с заданным `style`, а затем **автоматически восстанавливает** предыдущий стиль. В отличие от `setStyle`, безопасен без последующего `resetAttributes`.

**Параметры:**
- `txt` — строка для вывода
- `style` — применяемые стили (по умолчанию: `{styleBright}`)

**Пример:**
```nim
import std/terminal
write(stdout, "Обычный → ")
writeStyled("Жирный текст", {styleBright})
write(stdout, " → снова обычный\n")

writeStyled("[ОШИБКА]", {styleBright, styleReverse})
```

---

### `resetAttributes` (для File)

```nim
proc resetAttributes*(f: File)
```

Сбрасывает все атрибуты текста (цвет, яркость, стиль) файла `f` к стандартным. На POSIX записывает `ansiResetCode`. На Windows восстанавливает исходное слово атрибутов консоли, запомненное при запуске.

**Пример:**
```nim
import std/terminal
stdout.setForegroundColor(fgRed)
write(stdout, "Красный текст")
stdout.resetAttributes()
write(stdout, "Обычный текст\n")
```

---

### `resetAttributes` (для stdout)

```nim
proc resetAttributes*() {.noconv.}
```

Сбрасывает все атрибуты на `stdout`. Рекомендуется регистрировать как процедуру выхода:

```nim
import std/[terminal, exitprocs]
addExitProc(resetAttributes)
```

**Пример:**
```nim
import std/terminal
setForegroundColor(fgBlue)
write(stdout, "Синий текст")
resetAttributes()
```

---

### `ansiStyleCode`

```nim
proc ansiStyleCode*(style: int): string
template ansiStyleCode*(style: Style): string
template ansiStyleCode*(style: static[Style]): string   # оптимизировано на этапе компиляции
```

Возвращает «сырую» ANSI escape-строку для заданного кода стиля или значения перечисления `Style`. Полезно для встраивания кодов стиля в строки.

**Пример:**
```nim
import std/terminal
echo ansiStyleCode(styleBright)       # "\e[1m"
echo ansiStyleCode(styleUnderscore)   # "\e[4m"
echo ansiStyleCode(1)                 # "\e[1m"
```

---

## Цвета

### `setForegroundColor` (именованный цвет)

```nim
proc setForegroundColor*(f: File, fg: ForegroundColor, bright = false)
template setForegroundColor*(fg: ForegroundColor, bright = false)  # stdout
```

Устанавливает цвет переднего плана (текста) терминала. Эффект сохраняется до вызова `resetAttributes`.

**Параметры:**
- `f` — файл вывода
- `fg` — значение перечисления `ForegroundColor`
- `bright` — если `true`, использует яркий/высокоинтенсивный вариант (добавляет 60 к ANSI-коду)

**Пример:**
```nim
import std/terminal
stdout.setForegroundColor(fgRed)
write(stdout, "Красный текст\n")

stdout.setForegroundColor(fgGreen, bright = true)
write(stdout, "Ярко-зелёный текст\n")

stdout.setForegroundColor(fgDefault)    # восстановить по умолчанию
```

---

### `setBackgroundColor` (именованный цвет)

```nim
proc setBackgroundColor*(f: File, bg: BackgroundColor, bright = false)
template setBackgroundColor*(bg: BackgroundColor, bright = false)  # stdout
```

Устанавливает цвет фона терминала. Эффект сохраняется до вызова `resetAttributes`.

**Пример:**
```nim
import std/terminal
stdout.setBackgroundColor(bgBlue)
stdout.setForegroundColor(fgWhite)
write(stdout, "Белый на синем\n")
stdout.resetAttributes()

setBackgroundColor(bgYellow, bright = true)
write(stdout, "Текст на ярко-жёлтом фоне\n")
resetAttributes()
```

---

### `setForegroundColor` (true color)

```nim
proc setForegroundColor*(f: File, color: Color)
template setForegroundColor*(color: Color)    # сокращение для stdout
```

Устанавливает цвет переднего плана как RGB true color (`Color` из `std/colors`). Работает только после вызова `enableTrueColors()`.

**Параметры:**
- `color` — значение `Color` из `std/colors` (напр. `colRed`, `rgb(255, 128, 0)`)

**Пример:**
```nim
import std/[terminal, colors]
enableTrueColors()
stdout.setForegroundColor(colDodgerBlue)
write(stdout, "Синий Dodger\n")
stdout.resetAttributes()
disableTrueColors()
```

---

### `setBackgroundColor` (true color)

```nim
proc setBackgroundColor*(f: File, color: Color)
template setBackgroundColor*(color: Color)    # сокращение для stdout
```

Устанавливает цвет фона как RGB true color. Работает только после `enableTrueColors()`.

**Пример:**
```nim
import std/[terminal, colors]
enableTrueColors()
stdout.setBackgroundColor(colLimeGreen)
write(stdout, "Текст на лаймово-зелёном фоне\n")
stdout.resetAttributes()
```

---

## True Color (24-bit)

### `enableTrueColors`

```nim
proc enableTrueColors*()
```

Включает поддержку 24-битного true color (RGB) для терминала. Должна быть вызвана до использования true color вариантов `setForegroundColor` и `setBackgroundColor`.

- **POSIX:** проверяет переменную окружения `$COLORTERM` на значения `"truecolor"` или `"24bit"`.
- **Windows:** проверяет версию ОС (≥ Windows 10 build 10586) или `$ANSICON_DEF`, затем включает `ENABLE_VIRTUAL_TERMINAL_PROCESSING`.

**Пример:**
```nim
import std/[terminal, colors]
enableTrueColors()
if isTrueColorSupported():
  stdout.setForegroundColor(rgb(255, 128, 0))
  write(stdout, "Оранжевый текст через true color!\n")
  stdout.resetAttributes()
disableTrueColors()
```

---

### `disableTrueColors`

```nim
proc disableTrueColors*()
```

Отключает поддержку 24-битного true color. На Windows убирает флаг `ENABLE_VIRTUAL_TERMINAL_PROCESSING` из режима консоли.

**Пример:**
```nim
import std/terminal
enableTrueColors()
# ... цветной вывод ...
disableTrueColors()
```

---

### `isTrueColorSupported`

```nim
proc isTrueColorSupported*(): bool
```

Возвращает `true`, если терминал поддерживает true color **и** был вызван `enableTrueColors()`. Возвращает `false` до вызова `enableTrueColors()`, даже если терминал его поддерживает.

**Пример:**
```nim
import std/[terminal, colors]
enableTrueColors()
if isTrueColorSupported():
  echo "True color доступен!"
  stdout.setForegroundColor(colDeepPink)
  write(stdout, "Ярко-розовый!\n")
  stdout.resetAttributes()
else:
  echo "Переключение в режим 8 цветов"
  stdout.setForegroundColor(fgRed)
  write(stdout, "Красный\n")
  stdout.resetAttributes()
```

---

## Вспомогательные ANSI-функции

Эти функции возвращают «сырые» ANSI escape-строки, ничего не записывая в файл. Полезны для построения стилизованных строк, логирования или вывода не в терминал.

### `ansiForegroundColorCode` (именованный цвет)

```nim
proc ansiForegroundColorCode*(fg: ForegroundColor, bright = false): string
template ansiForegroundColorCode*(fg: static[ForegroundColor],
                                  bright: static[bool] = false): string
```

Возвращает ANSI escape-строку для указанного цвета переднего плана.

**Пример:**
```nim
import std/terminal
echo ansiForegroundColorCode(fgRed)                  # "\e[31m"
echo ansiForegroundColorCode(fgGreen, bright=true)   # "\e[92m"

# Построение цветной строки:
let s = ansiForegroundColorCode(fgBlue) & "Синий текст" & ansiResetCode
echo s
```

---

### `ansiForegroundColorCode` (true color)

```nim
proc ansiForegroundColorCode*(color: Color): string
template ansiForegroundColorCode*(color: static[Color]): string
```

Возвращает ANSI true-color (24-bit) escape-строку для `color` как код цвета переднего плана. Формат: `\e[38;2;R;G;Bm`.

**Пример:**
```nim
import std/[terminal, colors]
let code = ansiForegroundColorCode(colCornflowerBlue)
echo code & "Васильковый" & ansiResetCode
```

---

### `ansiBackgroundColorCode` (true color)

```nim
proc ansiBackgroundColorCode*(color: Color): string
template ansiBackgroundColorCode*(color: static[Color]): string
```

Возвращает ANSI true-color escape-строку для `color` как код цвета фона. Формат: `\e[48;2;R;G;Bm`.

**Пример:**
```nim
import std/[terminal, colors]
let bgCode = ansiBackgroundColorCode(colGold)
let fgCode = ansiForegroundColorCode(fgBlack)
echo bgCode & fgCode & "Чёрный на золотом" & ansiResetCode
```

---

## Стилизованный вывод

### `styledWrite`

```nim
macro styledWrite*(f: File, m: varargs[typed]): untyped
```

Макрос, аналогичный `write`, но понимающий аргументы стиля терминала. Аргументы типов `Style`, `set[Style]`, `ForegroundColor`, `BackgroundColor`, `Color` или `TerminalCmd` не записываются в файл напрямую — вместо этого вызываются соответствующие терминальные процедуры.

- Аргументы стиля **перед** строкой влияют только на эту строку.
- После вызова атрибуты **автоматически сбрасываются**.

**Типы принимаемых аргументов:**

| Тип | Действие |
|-----|----------|
| `string` | Записывается в `f` напрямую |
| `Style` | Вызывает `setStyle(f, {style})` |
| `set[Style]` | Вызывает `setStyle(f, style)` |
| `ForegroundColor` | Вызывает `setForegroundColor(f, color)` |
| `BackgroundColor` | Вызывает `setBackgroundColor(f, color)` |
| `Color` | Устанавливает true-цвет переднего/заднего плана (в зависимости от предшествующего `fgColor`/`bgColor`) |
| `TerminalCmd` | `resetStyle` сбрасывает атрибуты; `fgColor`/`bgColor` задают режим для следующего `Color` |

**Пример:**
```nim
import std/terminal
stdout.styledWrite(fgRed, "Ошибка: ", resetStyle, "что-то пошло не так")
stdout.styledWrite({styleBright, styleUnderscore}, "Важно!")
stdout.styledWrite(fgGreen, "зелёный ", fgBlue, "синий ", "без цвета")
```

---

### `styledWriteLine`

```nim
template styledWriteLine*(f: File, args: varargs[untyped])
```

Вызывает `styledWrite(f, args)` и добавляет перевод строки `"\n"` в конце. Наиболее часто используемая процедура стилизованного вывода.

**Пример:**
```nim
import std/terminal

proc error(msg: string) =
  styledWriteLine(stderr, fgRed, "Ошибка: ", resetStyle, msg)

proc success(msg: string) =
  styledWriteLine(stdout, fgGreen, styleBright, "✓ ", resetStyle, msg)

error("Файл не найден")
success("Сборка завершена")

stdout.styledWriteLine(bgCyan, fgBlack, "  ИНФО  ", resetStyle, " Приложение запущено")
```

---

### `styledEcho`

```nim
template styledEcho*(args: varargs[untyped])
```

Удобное сокращение для `stdout.styledWriteLine(args)`. Выводит стилизованный текст в stdout с автоматическим переводом строки.

**Пример:**
```nim
import std/terminal

# Несколько стилей в одном вызове:
styledEcho styleBright, fgGreen, "[ПРОЙДЕНО]", resetStyle, fgGreen, " Тест прошёл!"
styledEcho fgRed, "[ОШИБКА]", resetStyle, " Тест не прошёл"
styledEcho {styleBright, styleUnderscore}, "Заголовок раздела"

# Прогресс-бар:
styledEcho fgYellow, "Загрузка: ", fgGreen, "████████░░", resetStyle, " 80%"
```

---

## Ввод

### `getch`

```nim
proc getch*(): char
```

Читает один символ из терминала без его отображения на экране. Блокируется до нажатия клавиши. На POSIX временно переключает терминал в режим raw.

**Возвращает:** Символ нажатой клавиши.

**Поведение по платформам:**
- **POSIX:** сохраняет режим терминала, переключает в raw, читает один байт, восстанавливает режим.
- **Windows:** использует `ReadConsoleInput` для ожидания события нажатия клавиши.

> **Примечание:** Специальные клавиши (стрелки, F-клавиши) генерируют несколько байт на POSIX (escape-последовательности). `getch` вернёт только первый байт.

**Пример:**
```nim
import std/terminal
write(stdout, "Нажмите любую клавишу для продолжения...")
let ch = getch()
echo "\nВы нажали: ", ch.repr

# Навигация по меню:
write(stdout, "Нажмите 'q' для выхода, 'h' для справки: ")
let key = getch()
case key
of 'q': echo "\nВыход..."
of 'h': echo "\nСправка: ..."
else: echo "\nНеизвестная клавиша"
```

---

### `readPasswordFromStdin` (с var string)

```nim
proc readPasswordFromStdin*(prompt: string, password: var string): bool
  {.tags: [ReadIOEffect, WriteIOEffect].}
```

Читает пароль из stdin без его отображения. Показывает `prompt`, читает ввод, затем восстанавливает режим отображения символов.

**Параметры:**
- `prompt` — строка приглашения для пользователя
- `password` — переменная вывода для записи пароля (не должна быть nil)

**Возвращает:** `true` при успехе; `false` при достижении конца файла.

**Поведение по платформам:**
- **Windows:** открывает `CONIN$` с отключённым флагом `ENABLE_ECHO_INPUT`.
- **POSIX:** очищает флаг `ECHO` в настройках termios.

**Пример:**
```nim
import std/terminal
var pass: string
let ok = readPasswordFromStdin("Введите пароль: ", pass)
if ok:
  echo "Пароль длиной: ", pass.len, " символов"
else:
  echo "Достигнут конец файла"
```

---

### `readPasswordFromStdin` (возвращает строку)

```nim
proc readPasswordFromStdin*(prompt = "password: "): string
```

Удобная перегрузка, напрямую возвращающая пароль строкой.

**Параметры:**
- `prompt` — строка приглашения (по умолчанию: `"password: "`)

**Возвращает:** Введённый пароль в виде строки.

**Пример:**
```nim
import std/terminal
let password = readPasswordFromStdin()
let adminPass = readPasswordFromStdin("Пароль администратора: ")

if password == "секрет":
  echo "Доступ разрешён"
```

---

## Утилиты

### `isatty`

```nim
proc isatty*(f: File): bool
```

Возвращает `true`, если `f` подключён к реальному терминальному устройству (TTY), и `false`, если это канал, перенаправление файла или другой нетерминальный поток.

**Платформа:** Использует POSIX `isatty()` или Windows `_isatty()`.

**Пример:**
```nim
import std/terminal

if isatty(stdout):
  echo "Работаем в терминале — цвета включены"
  setForegroundColor(fgGreen)
  write(stdout, "Цветной вывод\n")
  resetAttributes()
else:
  echo "Вывод перенаправлен — цвета отключены"
```

---

## Шаблоны-сокращения для stdout

Модуль предоставляет удобные шаблоны-обёртки для всех основных процедур с `stdout` по умолчанию:

| Сокращение | Эквивалент |
|-----------|------------|
| `hideCursor()` | `hideCursor(stdout)` |
| `showCursor()` | `showCursor(stdout)` |
| `setCursorPos(x, y)` | `setCursorPos(stdout, x, y)` |
| `setCursorXPos(x)` | `setCursorXPos(stdout, x)` |
| `setCursorYPos(y)` *(Windows)* | `setCursorYPos(stdout, y)` |
| `cursorUp(n)` | `cursorUp(stdout, n)` |
| `cursorDown(n)` | `cursorDown(stdout, n)` |
| `cursorForward(n)` | `cursorForward(stdout, n)` |
| `cursorBackward(n)` | `cursorBackward(stdout, n)` |
| `eraseLine()` | `eraseLine(stdout)` |
| `eraseScreen()` | `eraseScreen(stdout)` |
| `setStyle(s)` | `setStyle(stdout, s)` |
| `setForegroundColor(fg)` | `setForegroundColor(stdout, fg)` |
| `setBackgroundColor(bg)` | `setBackgroundColor(stdout, bg)` |
| `setForegroundColor(c)` | `setForegroundColor(stdout, c)` |
| `setBackgroundColor(c)` | `setBackgroundColor(stdout, c)` |
| `resetAttributes()` | `resetAttributes(stdout)` |
| `styledEcho(...)` | `stdout.styledWriteLine(...)` |

---

## Сводная таблица доступности по платформам

| Функция | Linux | macOS | Windows | Примечания |
|---------|:-----:|:-----:|:-------:|-----------|
| `terminalWidth` | ✓ | ✓ | ✓ | |
| `terminalHeight` | ✓ | ✓ | ✓ | |
| `terminalSize` | ✓ | ✓ | ✓ | |
| `terminalWidthIoctl` | ✓ | ✓ | ✓ | Разная сигнатура |
| `terminalHeightIoctl` | ✓ | ✓ | ✓ | Разная сигнатура |
| `getCursorPos` | ✓ | ✓ | ✓ | POSIX требует реального TTY |
| `setCursorPos` | ✓ | ✓ | ✓ | |
| `setCursorXPos` | ✓ | ✓ | ✓ | |
| `setCursorYPos` | ✗ | ✗ | ✓ | Только Windows |
| `hideCursor` | ✓ | ✓ | ✓ | |
| `showCursor` | ✓ | ✓ | ✓ | |
| `cursorUp` | ✓ | ✓ | ✓ | |
| `cursorDown` | ✓ | ✓ | ✓ | |
| `cursorForward` | ✓ | ✓ | ✓ | |
| `cursorBackward` | ✓ | ✓ | ✓ | |
| `eraseLine` | ✓ | ✓ | ✓ | |
| `eraseScreen` | ✓ | ✓ | ✓ | |
| `setStyle` | ✓ | ✓ | ✓ | Ограниченные стили на Windows |
| `writeStyled` | ✓ | ✓ | ✓ | |
| `resetAttributes` | ✓ | ✓ | ✓ | |
| `ansiStyleCode` | ✓ | ✓ | ✓ | |
| `setForegroundColor` | ✓ | ✓ | ✓ | |
| `setBackgroundColor` | ✓ | ✓ | ✓ | |
| `setForegroundColor(Color)` | ✓ | ✓ | ✓ | Требует `enableTrueColors` |
| `setBackgroundColor(Color)` | ✓ | ✓ | ✓ | Требует `enableTrueColors` |
| `ansiForegroundColorCode` | ✓ | ✓ | ✓ | |
| `ansiBackgroundColorCode` | ✓ | ✓ | ✓ | |
| `enableTrueColors` | ✓ | ✓ | ✓ | Win10+ build 10586 |
| `disableTrueColors` | ✓ | ✓ | ✓ | |
| `isTrueColorSupported` | ✓ | ✓ | ✓ | |
| `styledWrite` | ✓ | ✓ | ✓ | |
| `styledWriteLine` | ✓ | ✓ | ✓ | |
| `styledEcho` | ✓ | ✓ | ✓ | |
| `getch` | ✓ | ✓ | ✓ | |
| `readPasswordFromStdin` | ✓ | ✓ | ✓ | |
| `isatty` | ✓ | ✓ | ✓ | |
| `ansiResetCode` | ✓ | ✓ | ✓ | константа |

---

## Полные примеры

### Прогресс-бар

```nim
import std/[terminal, os, strutils]

proc progressBar(value, total: int) =
  let pct = value * 100 div total
  let filled = value * 40 div total
  let bar = "#".repeat(filled) & "-".repeat(40 - filled)

  eraseLine()
  stdout.styledWrite(fgCyan, "[", fgGreen, bar, fgCyan, "] ",
                     fgYellow, $pct & "%", resetStyle)
  stdout.flushFile()

hideCursor()
addExitProc(proc() = showCursor(); resetAttributes())

for i in 1..100:
  progressBar(i, 100)
  sleep(30)
  if i < 100: cursorUp()

echo ""
styledEcho fgGreen, styleBright, "Завершено!"
```

---

### Цветной логгер

```nim
import std/terminal

type LogLevel = enum DEBUG, INFO, WARN, ERROR

proc log(level: LogLevel, msg: string) =
  let (label, color) = case level
    of DEBUG: ("[DEBUG]", fgCyan)
    of INFO:  ("[INFO] ", fgGreen)
    of WARN:  ("[WARN] ", fgYellow)
    of ERROR: ("[ERROR]", fgRed)

  if isatty(stderr):
    stderr.styledWriteLine(color, styleBright, label, resetStyle, " ", msg)
  else:
    stderr.writeLine(label & " " & msg)

log(INFO, "Сервер запущен на порту 8080")
log(WARN, "Использование памяти выше 80%")
log(ERROR, "Соединение отклонено")
```

---

### Строка состояния редактора

```nim
import std/terminal

proc drawStatusBar(filename: string, line, col, total: int) =
  let w = terminalWidth()
  let h = terminalHeight()

  # Переместить в последнюю строку:
  setCursorPos(0, h - 1)
  stdout.setBackgroundColor(bgWhite)
  stdout.setForegroundColor(fgBlack)

  let info = " " & filename & "  Стр " & $line & ", Стлб " & $col &
             "  " & $total & " строк "
  let padding = " ".repeat(max(0, w - info.len))
  write(stdout, info & padding)
  resetAttributes()

drawStatusBar("main.nim", 42, 15, 300)
```

---

### Интерактивное меню

```nim
import std/terminal

proc menu(items: seq[string]): int =
  hideCursor()
  var selected = 0
  while true:
    for i, item in items:
      eraseLine()
      if i == selected:
        stdout.styledWriteLine(fgBlack, bgWhite, " > " & item & " ", resetStyle)
      else:
        stdout.writeLine("   " & item)
    cursorUp(items.len)

    let key = getch()
    case key
    of 'k', 'A': selected = max(0, selected - 1)          # вверх
    of 'j', 'B': selected = min(items.high, selected + 1)  # вниз
    of '\r', '\n':
      cursorDown(items.len)
      showCursor()
      return selected
    else: discard

let choice = menu(@["Новый файл", "Открыть файл", "Настройки", "Выход"])
echo "Вы выбрали пункт: ", choice
```

---

### Градиент true color

```nim
import std/[terminal, colors]

enableTrueColors()
if isTrueColorSupported():
  let w = terminalWidth()
  for x in 0..<w:
    let r = x * 255 div w
    let b = 255 - r
    stdout.setBackgroundColor(rgb(r.uint8, 0, b.uint8))
    write(stdout, " ")
  resetAttributes()
  echo ""
else:
  echo "True color не поддерживается"
disableTrueColors()
```

---

*Сгенерировано на основе модуля `std/terminal` стандартной библиотеки Nim.*
