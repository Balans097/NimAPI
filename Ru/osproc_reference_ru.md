# Справочник модуля `osproc` (Nim)

> Модуль стандартной библиотеки Nim для запуска и управления процессами ОС.  
> Предоставляет средства для создания дочерних процессов, управления их потоками ввода-вывода, ожидания завершения и параллельного выполнения команд.

---

## Содержание

1. [Обзор](#обзор)
2. [Типы](#типы)
   - [ProcessOption](#processoption)
   - [Process](#process)
3. [Запуск процессов](#запуск-процессов)
   - [startProcess](#startprocess)
   - [execProcess](#execprocess)
   - [execCmd](#execcmd)
   - [execCmdEx](#execcmdex)
   - [execProcesses](#execprocesses)
4. [Управление процессом](#управление-процессом)
   - [close](#close)
   - [suspend](#suspend)
   - [resume](#resume)
   - [terminate](#terminate)
   - [kill](#kill)
5. [Состояние процесса](#состояние-процесса)
   - [running](#running)
   - [processID](#processid)
   - [waitForExit](#waitforexit)
   - [peekExitCode](#peekexitcode)
   - [hasData](#hasdata)
6. [Потоки ввода-вывода](#потоки-ввода-вывода)
   - [inputStream](#inputstream)
   - [outputStream](#outputstream)
   - [errorStream](#errorstream)
   - [peekableOutputStream](#peekableoutputstream)
   - [peekableErrorStream](#peekableerrorstream)
   - [inputHandle](#inputhandle)
   - [outputHandle](#outputhandle)
   - [errorHandle](#errorhandle)
7. [Чтение вывода](#чтение-вывода)
   - [lines (итератор)](#lines-итератор)
   - [readLines](#readlines)
8. [Системная информация](#системная-информация)
   - [countProcessors](#countprocessors)
9. [Экспортируемые функции оболочки](#экспортируемые-функции-оболочки)
10. [Практические примеры](#практические-примеры)
11. [Быстрая шпаргалка](#быстрая-шпаргалка)

---

## Обзор

Модуль `osproc` реализует расширенные средства запуска и взаимодействия с дочерними процессами. Он позволяет:
- Запускать исполняемые файлы с аргументами и пользовательской средой окружения
- Читать и писать в потоки `stdin`/`stdout`/`stderr` дочернего процесса
- Приостанавливать, возобновлять, завершать процессы
- Запускать множество процессов параллельно
- Получать код возврата

**Связанные модули:** `os`, `streams`, `memfiles`

---

## Типы

### `ProcessOption`

```nim
type ProcessOption* = enum
  poEchoCmd,          ## Вывести команду перед выполнением
  poUsePath,          ## Искать исполняемый файл через PATH
  poEvalCommand,      ## Передать команду напрямую в оболочку (shell)
  poStdErrToStdOut,   ## Объединить stderr и stdout
  poParentStreams,     ## Использовать потоки родительского процесса
  poInteractive,      ## Оптимизировать буферизацию для интерактивных UI
  poDaemon            ## Windows: без окна; Unix: запуск как демон (WIP)
```

Флаги передаются в `startProcess` для управления поведением запускаемого процесса.

> ⚠️ `poEvalCommand` — передаёт команду напрямую в `sh`/`cmd.exe`. Используйте только с доверенным источником!  
> ⚠️ На Windows `poUsePath` включён по умолчанию.

---

### `Process`

```nim
type Process* = ref ProcessObj
```

Объект, представляющий запущенный процесс ОС. Возвращается функцией `startProcess`.  
После завершения работы с процессом необходимо вызвать `close`.

---

## Запуск процессов

### `startProcess`

```nim
proc startProcess*(command: string, workingDir: string = "",
    args: openArray[string] = [], env: StringTableRef = nil,
    options: set[ProcessOption] = {poStdErrToStdOut}): owned(Process)
```

Запускает процесс и возвращает объект `Process`. Никогда не возвращает `nil` — при ошибке выбрасывает `OSError`.

**Параметры:**
- `command` — путь к исполняемому файлу
- `workingDir` — рабочий каталог процесса (`""` = текущий)
- `args` — аргументы командной строки (без имени программы!)
- `env` — переменные среды окружения (`nil` = унаследовать от родителя)
- `options` — набор флагов `ProcessOption`

> ⚠️ При использовании `poEvalCommand` нельзя передавать `args`.  
> ⚠️ После работы необходимо вызвать `close(p)`.

```nim
import std/[osproc, os]

# Простой запуск
var p = startProcess("ls", args = ["-la"], options = {poUsePath})
echo p.outputStream.readAll()
p.close()

# С пользовательской средой
import std/strtabs
var env = newStringTable({"MY_VAR": "hello"})
var p2 = startProcess("printenv", env = env, options = {poUsePath})
echo p2.outputStream.readAll()
p2.close()

# С указанием рабочего каталога
var p3 = startProcess("pwd", workingDir = "/tmp", options = {poUsePath})
echo p3.outputStream.readAll()
p3.close()
```

---

### `execProcess`

```nim
proc execProcess*(command: string, workingDir: string = "",
    args: openArray[string] = [], env: StringTableRef = nil,
    options: set[ProcessOption] = {poStdErrToStdOut, poUsePath, poEvalCommand}): string
```

Удобная обёртка: запускает команду и возвращает весь её вывод как строку. Блокирует до завершения.

> ⚠️ По умолчанию включён `poEvalCommand` — для совместимости. Всегда явно передавайте `options`!

```nim
import std/osproc

# Выполнение через оболочку (poEvalCommand включён по умолчанию)
let out1 = execProcess("ls -la /tmp")
echo out1

# Предпочтительный способ — без оболочки:
let out2 = execProcess("ls", args = ["-la", "/tmp"], options = {poUsePath})
echo out2

# Компиляция Nim-файла:
let result = execProcess("nim", args = ["c", "-r", "hello.nim"],
                          options = {poUsePath})
echo result
```

---

### `execCmd`

```nim
proc execCmd*(command: string): int
```

Выполняет команду через оболочку системы и возвращает код возврата. Потоки stdin/stdout/stderr наследуются от родителя (вывод идёт прямо в терминал).

> Аналог функции `system()` в Си.

```nim
import std/osproc

let code = execCmd("echo Hello, World!")
echo "Код возврата: ", code   # 0

let errCode = execCmd("ls /nonexistent")
echo "Код ошибки: ", errCode  # != 0
```

---

### `execCmdEx`

```nim
proc execCmdEx*(command: string,
                options: set[ProcessOption] = {poStdErrToStdOut, poUsePath},
                env: StringTableRef = nil,
                workingDir = "",
                input = ""): tuple[output: string, exitCode: int]
```

Удобная обёртка, возвращающая и вывод, и код возврата в одном кортеже. Если задан `input`, он передаётся в stdin дочернего процесса.

> ⚠️ Может заблокироваться, если `input.len` превышает размер буфера канала ОС.

```nim
import std/osproc

# Простое использование
let (output, code) = execCmdEx("echo hello")
echo output    # "hello\n"
echo code      # 0

# Проверка ошибок
let (_, errCode) = execCmdEx("ls /nonexistent")
echo errCode  # не 0

# С пользовательской средой (только Posix)
when defined(posix):
  import std/strtabs
  let (out2, _) = execCmdEx("echo $FOO",
                              env = newStringTable({"FOO": "bar"}))
  echo out2  # "bar\n"

# С передачей данных на stdin
var res = execCmdEx("nim r --hints:off -", options = {}, input = "echo 3*4")
echo res.output   # "12\n"
echo res.exitCode # 0

# С указанием рабочего каталога
when defined(posix):
  let (pwd, _) = execCmdEx("echo $PWD", workingDir = "/")
  echo pwd  # "/\n"
```

---

### `execProcesses`

```nim
proc execProcesses*(cmds: openArray[string],
    options = {poStdErrToStdOut, poParentStreams},
    n = countProcessors(),
    beforeRunEvent: proc(idx: int) = nil,
    afterRunEvent: proc(idx: int, p: Process) = nil): int
```

Выполняет список команд `cmds` **параллельно**, создавая до `n` процессов одновременно. Возвращает максимальный абсолютный код возврата среди всех процессов.

**Параметры:**
- `cmds` — список команд для выполнения (каждая передаётся через `poEvalCommand`)
- `options` — флаги для каждого процесса
- `n` — максимальное число одновременных процессов (по умолчанию = числу процессоров)
- `beforeRunEvent` — колбэк, вызываемый перед запуском `idx`-й команды
- `afterRunEvent` — колбэк, вызываемый после завершения `idx`-й команды

```nim
import std/osproc

let commands = ["echo one", "echo two", "echo three", "echo four"]

let maxCode = execProcesses(commands, n = 2,
  beforeRunEvent = proc(i: int) =
    echo "Запускаем: ", commands[i],
  afterRunEvent = proc(i: int, p: Process) =
    echo "Завершено: ", commands[i]
)
echo "Максимальный код возврата: ", maxCode
```

---

## Управление процессом

### `close`

```nim
proc close*(p: Process)
```

Освобождает ресурсы процесса после его завершения. **Всегда вызывайте после работы с процессом.**

> ⚠️ Если процесс ещё выполняется — он будет принудительно завершён. Это может привести к зомби-процессам.

```nim
var p = startProcess("sleep", args = ["5"], options = {poUsePath})
# ... работа ...
p.close()
```

---

### `suspend`

```nim
proc suspend*(p: Process)
```

Приостанавливает выполнение процесса `p`.  
На Unix — отправляет `SIGSTOP`, на Windows — вызывает `SuspendThread`.

```nim
var p = startProcess("some_long_task", options = {poUsePath})
p.suspend()
# ... сделать что-то ...
p.resume()
p.close()
```

---

### `resume`

```nim
proc resume*(p: Process)
```

Возобновляет выполнение приостановленного процесса `p`.  
На Unix — отправляет `SIGCONT`, на Windows — вызывает `ResumeThread`.

```nim
p.suspend()
# ... пауза ...
p.resume()
```

---

### `terminate`

```nim
proc terminate*(p: Process)
```

Мягко останавливает процесс. На Unix отправляет `SIGTERM` (процесс может обработать сигнал и завершиться корректно), на Windows вызывает `TerminateProcess()`.

```nim
var p = startProcess("my_server", options = {poUsePath})
# ...
p.terminate()
discard p.waitForExit()
p.close()
```

---

### `kill`

```nim
proc kill*(p: Process)
```

Принудительно уничтожает процесс. На Unix отправляет `SIGKILL` (процесс не может перехватить этот сигнал), на Windows является алиасом для `terminate`.

```nim
var p = startProcess("stubborn_process", options = {poUsePath})
if p.running:
  p.kill()
p.close()
```

---

## Состояние процесса

### `running`

```nim
proc running*(p: Process): bool
```

Возвращает `true`, если процесс `p` ещё выполняется. Не блокирует вызывающий поток.

```nim
var p = startProcess("sleep", args = ["2"], options = {poUsePath})
echo p.running  # true
discard p.waitForExit()
echo p.running  # false
p.close()
```

---

### `processID`

```nim
proc processID*(p: Process): int
```

Возвращает PID (идентификатор процесса) запущенного процесса `p`.

```nim
var p = startProcess("sleep", args = ["10"], options = {poUsePath})
echo "PID: ", p.processID
p.terminate()
p.close()
```

---

### `waitForExit`

```nim
proc waitForExit*(p: Process, timeout: int = -1): int
```

Блокирует текущий поток и ожидает завершения процесса `p`. Возвращает код возврата.

**Параметры:**
- `timeout` — тайм-аут в **миллисекундах** (`-1` = ждать бесконечно)

На Unix, если процесс завершился из-за сигнала, возвращает `128 + номер_сигнала`.

> ⚠️ Не используйте с процессами без `poParentStreams` — может произойти взаимная блокировка при переполнении буфера вывода.

```nim
var p = startProcess("ls", args = ["-la"], options = {poUsePath, poStdErrToStdOut})
echo p.outputStream.readAll()
let code = p.waitForExit()
echo "Код: ", code
p.close()

# С тайм-аутом
var p2 = startProcess("sleep", args = ["10"], options = {poUsePath})
let code2 = p2.waitForExit(timeout = 1000)  # ждём 1 секунду
echo code2
p2.close()
```

---

### `peekExitCode`

```nim
proc peekExitCode*(p: Process): int
```

Не блокирует. Возвращает `-1` если процесс ещё выполняется, иначе — код возврата.  
На Unix, если завершился из-за сигнала, возвращает `128 + номер_сигнала`.

```nim
var p = startProcess("sleep", args = ["5"], options = {poUsePath})
while p.peekExitCode() == -1:
  echo "Ещё работает..."
  # ... можно делать другую работу ...
echo "Завершился с кодом: ", p.peekExitCode()
p.close()
```

---

### `hasData`

```nim
proc hasData*(p: Process): bool
```

Возвращает `true`, если в выходном потоке процесса есть данные для чтения (неблокирующая проверка). Полезно для неблокирующего опроса вывода.

```nim
var p = startProcess("ping", args = ["localhost", "-c", "3"], options = {poUsePath})
while p.running or p.hasData:
  if p.hasData:
    echo p.outputStream.readLine()
p.close()
```

---

## Потоки ввода-вывода

> ⚠️ **Все возвращаемые потоки и дескрипторы НЕ нужно закрывать вручную** — они закрываются автоматически при вызове `close(p)`.

### `inputStream`

```nim
proc inputStream*(p: Process): Stream
```

Возвращает поток для **записи** в stdin дочернего процесса.

```nim
import std/[osproc, streams]

var p = startProcess("cat", options = {poUsePath})
p.inputStream.write("Привет от родителя\n")
p.inputStream.flush()
# закрыть stdin, чтобы cat получил EOF:
p.inputStream.close()
echo p.outputStream.readAll()
p.close()
```

---

### `outputStream`

```nim
proc outputStream*(p: Process): Stream
```

Возвращает поток для **чтения** stdout дочернего процесса. Не поддерживает peek/write/setOption — для этого используйте `peekableOutputStream`.

```nim
import std/[osproc, streams]

var p = startProcess("echo", args = ["hello world"], options = {poUsePath})
let out = p.outputStream.readAll()
echo out  # "hello world\n"
p.close()
```

---

### `errorStream`

```nim
proc errorStream*(p: Process): Stream
```

Возвращает поток для **чтения** stderr дочернего процесса. Не поддерживает peek/write/setOption — для этого используйте `peekableErrorStream`.

```nim
import std/[osproc, streams]

var p = startProcess("ls", args = ["/nonexistent"], options = {poUsePath})
discard p.waitForExit()
let err = p.errorStream.readAll()
echo "Ошибка: ", err
p.close()
```

---

### `peekableOutputStream`

```nim
proc peekableOutputStream*(p: Process): Stream
```

Как `outputStream`, но возвращаемый поток поддерживает операцию peek (просмотр без потребления данных). Доступно с Nim 1.3.

```nim
import std/[osproc, streams]

var p = startProcess("echo", args = ["test"], options = {poUsePath})
let s = p.peekableOutputStream
# можно выполнить peek перед чтением
discard p.waitForExit()
p.close()
```

---

### `peekableErrorStream`

```nim
proc peekableErrorStream*(p: Process): Stream
```

Как `errorStream`, но поддерживает peek. Доступно с Nim 1.3.

---

### `inputHandle`

```nim
proc inputHandle*(p: Process): FileHandle
```

Возвращает низкоуровневый `FileHandle` для stdin процесса. Полезно при прямой работе с файловыми дескрипторами ОС.

---

### `outputHandle`

```nim
proc outputHandle*(p: Process): FileHandle
```

Возвращает `FileHandle` для stdout процесса.

---

### `errorHandle`

```nim
proc errorHandle*(p: Process): FileHandle
```

Возвращает `FileHandle` для stderr процесса.

```nim
import std/osproc

var p = startProcess("echo", args = ["hello"], options = {poUsePath})
echo "stdin fd:  ", p.inputHandle
echo "stdout fd: ", p.outputHandle
echo "stderr fd: ", p.errorHandle
p.close()
```

---

## Чтение вывода

### `lines` (итератор)

```nim
iterator lines*(p: Process, keepNewLines = false): string
```

Удобный итератор для построчного чтения stdout дочернего процесса. По завершении автоматически вызывает `waitForExit`. Доступно с Nim 1.3.

**Параметры:**
- `keepNewLines` — если `true`, символы переноса строки сохраняются в каждой строке

```nim
import std/osproc

# Запуск двух процессов параллельно с чтением вывода
const opts = {poUsePath, poDaemon, poStdErrToStdOut}
var ps: seq[Process]
for prog in ["echo one", "echo two"]:
  ps.add startProcess("sh", args = ["-c", prog], options = {poUsePath})

for p in ps:
  var i = 0
  for line in p.lines:
    echo line
    i.inc
    if i > 100: break
  p.close()
```

---

### `readLines`

```nim
proc readLines*(p: Process): (seq[string], int)
```

Читает весь stdout процесса и возвращает кортеж `(строки, код_возврата)`. Ждёт завершения процесса. Доступно с Nim 1.3.

```nim
import std/osproc

const opts = {poUsePath, poDaemon, poStdErrToStdOut}
var ps: seq[Process]
for prog in ["ls /tmp", "ls /etc"]:
  ps.add startProcess("sh", args = ["-c", prog], options = {poUsePath})

for p in ps:
  let (lines, exitCode) = p.readLines
  if exitCode != 0:
    echo "Ошибка:"
    for line in lines: echo line
  else:
    echo "Успех: ", lines.len, " строк"
  p.close()
```

---

## Системная информация

### `countProcessors`

```nim
proc countProcessors*(): int
```

Возвращает число процессоров/ядер системы. Возвращает `0`, если определить не удалось. Использует `cpuinfo.countProcessors` внутри.

```nim
import std/osproc

echo "Число ядер: ", countProcessors()

# Запустить по одному процессу на каждое ядро:
let n = max(1, countProcessors())
echo "Параллельных процессов: ", n
```

---

## Экспортируемые функции оболочки

Модуль `osproc` реэкспортирует следующие функции из модуля `os`:

| Функция | Описание |
|---------|----------|
| `quoteShell(s)` | Экранирует строку для безопасной передачи в оболочку (платформозависимо) |
| `quoteShellWindows(s)` | Экранирует строку для `cmd.exe` |
| `quoteShellPosix(s)` | Экранирует строку для Unix-оболочки |

```nim
import std/osproc

let arg = "hello world"
echo quoteShell(arg)         # 'hello world' (Unix) или "hello world" (Windows)
echo quoteShellPosix(arg)    # 'hello world'
echo quoteShellWindows(arg)  # "hello world"

# Безопасное построение команды:
let cmd = "myapp " & quoteShell("/path/with spaces/file.txt")
let code = execCmd(cmd)
```

---

## Практические примеры

### Пример 1: Захват stdout и stderr раздельно

```nim
import std/[osproc, streams]

var p = startProcess("ls", args = ["/nonexistent", "/tmp"],
                     options = {poUsePath})
let stdout_out = p.outputStream.readAll()
let stderr_out = p.errorStream.readAll()
let code = p.waitForExit()

echo "stdout:\n", stdout_out
echo "stderr:\n", stderr_out
echo "Код: ", code
p.close()
```

---

### Пример 2: Двусторонняя коммуникация (stdin → stdout)

```nim
import std/[osproc, streams]

# Запускаем Python для вычислений
var p = startProcess("python3", options = {poUsePath})
p.inputStream.write("print(2 ** 32)\n")
p.inputStream.flush()
p.inputStream.close()  # сигнал EOF

let result = p.outputStream.readAll()
echo result.strip()  # "4294967296"
discard p.waitForExit()
p.close()
```

---

### Пример 3: Неблокирующий мониторинг процесса

```nim
import std/[osproc, os]

var p = startProcess("ping", args = ["-c", "5", "localhost"],
                     options = {poUsePath, poStdErrToStdOut})

while p.running:
  if p.hasData:
    let line = p.outputStream.readLine()
    echo ">> ", line
  else:
    sleep(50)

let code = p.waitForExit()
echo "Завершился: ", code
p.close()
```

---

### Пример 4: Параллельный сбор данных

```nim
import std/[osproc, strutils]

# Одновременный запрос к нескольким хостам
let hosts = ["google.com", "github.com", "nim-lang.org"]
var procs = newSeq[Process](hosts.len)

for i, host in hosts:
  procs[i] = startProcess("ping",
    args = ["-c", "1", host],
    options = {poUsePath, poStdErrToStdOut})

for i, p in procs:
  let code = p.waitForExit()
  if code == 0:
    echo hosts[i], " — доступен"
  else:
    echo hosts[i], " — недоступен"
  p.close()
```

---

### Пример 5: Использование `execProcesses` с колбэками

```nim
import std/osproc

let tasks = [
  "echo задача_1",
  "sleep 1 && echo задача_2",
  "echo задача_3"
]

var results: seq[string]

let maxCode = execProcesses(tasks, n = 2,
  beforeRunEvent = proc(i: int) =
    echo "[СТАРТ] ", tasks[i],
  afterRunEvent = proc(i: int, p: Process) =
    echo "[КОНЕЦ] ", tasks[i]
)
echo "Наибольший код возврата: ", maxCode
```

---

### Пример 6: Безопасный запуск с `poUsePath` вместо `poEvalCommand`

```nim
import std/osproc

# ПЛОХО: уязвимо к инъекции
let bad = execProcess("ls " & userInput)

# ХОРОШО: аргументы передаются отдельно
let good = execProcess("ls", args = [userInput], options = {poUsePath})
echo good
```

---

## Быстрая шпаргалка

### Выбор функции запуска

| Функция | Блокирует? | Возвращает | Когда использовать |
|---------|-----------|------------|-------------------|
| `execCmd` | Да | Код возврата | Нужен только код, вывод в терминал |
| `execProcess` | Да | Строка вывода | Нужен весь вывод как строка |
| `execCmdEx` | Да | `(вывод, код)` | Нужны и вывод, и код возврата |
| `startProcess` | Нет | `Process` | Полный контроль, фоновые задачи |
| `execProcesses` | Да | Макс. код | Параллельное выполнение списка |

### Управление процессом

| Задача | Код |
|--------|-----|
| Запустить | `var p = startProcess("cmd", args=["a"], options={poUsePath})` |
| Ждать завершения | `discard p.waitForExit()` |
| Ждать с тайм-аутом | `discard p.waitForExit(timeout=5000)` |
| Проверить статус | `p.running` |
| Заглянуть в код | `p.peekExitCode()` |
| Приостановить | `p.suspend()` |
| Возобновить | `p.resume()` |
| Мягко завершить | `p.terminate()` |
| Принудительно убить | `p.kill()` |
| Очистить ресурсы | `p.close()` |
| Получить PID | `p.processID` |

### Потоки

| Поток | Направление | Функция |
|-------|-------------|---------|
| stdin | → процесс | `p.inputStream` |
| stdout | ← процесс | `p.outputStream` |
| stderr | ← процесс | `p.errorStream` |
| stdout (peek) | ← процесс | `p.peekableOutputStream` |
| stderr (peek) | ← процесс | `p.peekableErrorStream` |
| stdin fd | → процесс | `p.inputHandle` |
| stdout fd | ← процесс | `p.outputHandle` |
| stderr fd | ← процесс | `p.errorHandle` |

### Часто используемые наборы флагов

```nim
# Захватить вывод, найти в PATH
{poUsePath, poStdErrToStdOut}

# Интерактивный/фоновый процесс
{poUsePath, poDaemon, poStdErrToStdOut}

# Параллельное выполнение
{poStdErrToStdOut, poParentStreams}

# Команда через оболочку (только для доверенного ввода!)
{poUsePath, poEvalCommand}
```
