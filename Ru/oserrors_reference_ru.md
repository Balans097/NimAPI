# std/oserrors — полный справочник модуля

> **Входит в стандартную библиотеку Nim** (`std/oserrors`).

## Обзор

Модуль `oserrors` — это тонкий, но принципиально важный мост между механизмом сообщения об ошибках операционной системы и системой исключений Nim. Когда вы вызываете функции уровня ОС — открываете файлы, запускаете процессы, привязываете сокеты — система фиксирует причину любого сбоя в виде числового кода ошибки. Данный модуль предоставляет три вещи:

1. Типизированную обёртку над этим числовым кодом (`OSErrorCode`), которая не позволяет компилятору случайно смешать его с обычными целыми числами.
2. Способ прочитать последний зафиксированный код ошибки ОС (`osLastError`).
3. Инструменты для преобразования кода в читаемое сообщение (`osErrorMsg`) или немедленного возбуждения исключения Nim (`newOSError`, `raiseOSError`).

На **POSIX**-системах (Linux, macOS, BSD) коды поступают из глобальной переменной C `errno`. На **Windows** — из `GetLastError()`. Модуль прозрачно абстрагирует обе платформы.

---

## Тип

### `OSErrorCode`

```nim
type OSErrorCode* = distinct int32
```

**Отличный** (`distinct`) тип-обёртка над 32-битным знаковым целым, представляющий числовой код ошибки ОС. Поскольку тип объявлен как `distinct`, система типов Nim считает его полностью отдельным от `int32` — нельзя случайно передать «голое» целое там, где ожидается `OSErrorCode`, или сравнить его с обычным числом без явного приведения.

Типичный жизненный цикл `OSErrorCode`:

```
Вызов ОС завершился ошибкой
            │
            ▼
    osLastError()           ← немедленно захватить код
            │
            ├─► osErrorMsg()          ← преобразовать в читаемую строку
            │
            └─► raiseOSError() / newOSError()  ← превратить в исключение
```

---

## Операторы для `OSErrorCode`

### `==`

```nim
proc `==`*(err1, err2: OSErrorCode): bool {.borrow.}
```

**Сравнивает два значения `OSErrorCode` на равенство.**

Реализован через `borrow` — делегирует напрямую равенству `int32`. Используйте это, чтобы проверить, соответствует ли захваченный код известной константе.

**Пример:**

```nim
import std/oserrors

let err = osLastError()
if err == OSErrorCode(0):
  echo "Ошибок не зафиксировано."
```

---

### `$`

```nim
proc `$`*(err: OSErrorCode): string {.borrow.}
```

**Преобразует `OSErrorCode` в его десятичное строковое представление.**

Возвращает сам числовой код в виде строки — полезно для низкоуровневого логирования. Это **не** читаемое описание ошибки; для этого используйте `osErrorMsg`.

**Пример:**

```nim
import std/oserrors

let err = osLastError()
echo "Код ошибки: ", $err              # например "2"
echo "Описание: ", osErrorMsg(err)     # например "No such file or directory"
```

---

## Процедуры

---

### `osLastError`

```nim
proc osLastError*(): OSErrorCode {.sideEffect.}
```

**Возвращает код ошибки, установленный последним неудачным вызовом ОС в текущем потоке.**

Это всегда первое, что нужно вызвать после обнаружения сбоя операции ОС. В момент вызова код фиксируется и оборачивается в `OSErrorCode`, который можно проверить или передать дальше. Если откладывать этот вызов — даже на одну-две несвязанные функции — на Windows есть риск потерять оригинальный код ошибки.

**Поведение по платформам:**

| Платформа | Источник кода |
|---|---|
| POSIX (Linux, macOS, …) | Глобальная переменная C `errno` |
| Windows | `GetLastError()` |
| NimScript | всегда возвращает `0` (нет доступа к ОС) |

> **Предупреждение для Windows:** некоторые вызовы Windows API внутри сбрасывают `GetLastError()` в `0` как побочный эффект, даже когда они завершаются успешно. Вызывайте `osLastError()` *немедленно* после неудачного вызова, до любого другого взаимодействия с ОС. На POSIX это не проблема, так как `errno` записывается только при сбое.

**Пример:**

```nim
import std/oserrors

let code = osLastError()
if code != OSErrorCode(0):
  echo "Ошибка ОС: ", osErrorMsg(code)
```

**Пример — правильный шаблон использования (сначала захватить, потом проверить):**

```nim
import std/oserrors

let failed = someOsCall() < 0    # обнаружить сбой
let code = osLastError()         # захватить сразу, пока errno/GetLastError свеж
if failed:
  raiseOSError(code)
```

---

### `osErrorMsg`

```nim
proc osErrorMsg*(errorCode: OSErrorCode): string
```

**Преобразует числовой код ошибки ОС в человекочитаемую строку-описание на английском языке.**

Внутренне вызывает `strerror()` на POSIX и `FormatMessageW()` на Windows — то есть текст сообщения приходит от самой ОС и отражает формулировки конкретной платформы. Если код равен `0` или ОС не может предоставить сообщение для заданного кода, возвращается пустая строка `""` (не исключение).

**Пример:**

```nim
import std/oserrors

echo osErrorMsg(OSErrorCode(0))   # ""
echo osErrorMsg(OSErrorCode(1))   # "Operation not permitted"  (POSIX)
echo osErrorMsg(OSErrorCode(2))   # "No such file or directory"  (POSIX)
```

**Пример — построение собственного сообщения об ошибке:**

```nim
import std/oserrors

proc checkResult(rc: int) =
  if rc < 0:
    let code = osLastError()
    let msg = osErrorMsg(code)
    echo "Сбой (", $code, "): ", msg

checkResult(-1)
```

---

### `newOSError`

```nim
proc newOSError*(
  errorCode: OSErrorCode,
  additionalInfo = ""
): owned(ref OSError) {.noinline.}
```

**Создаёт и возвращает новый объект исключения `OSError`, не возбуждая его.**

Это невозбуждающий аналог `raiseOSError`. Используйте его, когда хотите сконструировать исключение, возможно проверить или дополнить его, и только потом возбудить — или вообще передать куда-то ещё.

Поле `msg` исключения заполняется результатом `osErrorMsg(errorCode)`. Если полученное сообщение пусто (код `0` или неизвестен), используется текст-заглушка `"unknown OS error"`. Сам `errorCode` также сохраняется в объекте исключения, чтобы вызывающий код мог проверить его программно.

Если `additionalInfo` не пуст, он добавляется к сообщению с новой строки с префиксом `"Additional info: "`. Намеренно без завершающей пунктуации — чтобы не мешать функции «перейти к файлу» в IDE, если в качестве дополнительной информации передаётся путь к файлу.

**Пример — создание без немедленного возбуждения:**

```nim
import std/oserrors

proc tryOpen(path: string): ref OSError =
  # ... попытка операции ...
  let code = osLastError()
  if code != OSErrorCode(0):
    return newOSError(code, "при открытии: " & path)
  return nil

let err = tryOpen("/несуществующий/путь")
if err != nil:
  echo err.msg
```

**Пример — добавление дополнительного контекста:**

```nim
import std/oserrors

let code = osLastError()
let err = newOSError(code, "path: /etc/shadow")
# err.msg может выглядеть так:
# "Permission denied\nAdditional info: path: /etc/shadow"
```

---

### `raiseOSError`

```nim
proc raiseOSError*(errorCode: OSErrorCode, additionalInfo = "") {.noinline.}
```

**Создаёт исключение `OSError` из заданного кода ошибки и немедленно его возбуждает.**

Это наиболее распространённый способ превратить сбой ОС в исключение Nim. Процедура — однострочный эквивалент `raise newOSError(errorCode, additionalInfo)` и покрывает подавляющее большинство ситуаций, когда нужно обнаружить сбой и немедленно передать его вверх по стеку вызовов.

Параметр `additionalInfo` позволяет прикрепить контекстную информацию — например, путь к файлу или имя операции — которая появится в сообщении исключения. Это значительно облегчает диагностику по стеку вызовов или записи в логе.

**Пример — простейшее использование:**

```nim
import std/oserrors

proc mustSucceed(rc: cint) =
  if rc < 0:
    raiseOSError(osLastError())
```

**Пример — с дополнительным контекстом:**

```nim
import std/oserrors

proc openFile(path: string): FileHandle =
  let handle = rawOpen(path)   # гипотетический «сырой» вызов
  if handle < 0:
    raiseOSError(osLastError(), path)
  return handle
```

Итоговое сообщение исключения может выглядеть так:

```
No such file or directory
Additional info: /home/user/missing.txt
```

**Пример — перехват и анализ возбуждённой ошибки:**

```nim
import std/oserrors

try:
  raiseOSError(OSErrorCode(13), "чтение конфига")
except OSError as e:
  echo "Поймано OSError: ", e.msg
  echo "Код ошибки:      ", e.errorCode
```

---

## Типичный рабочий процесс

Четыре экспортируемых символа спроектированы для совместного использования по предсказуемому шаблону:

```
1. Сделать вызов ОС.
2. Если он завершился неудачно — немедленно вызвать osLastError(), чтобы захватить код.
3а. Если нужен только текст:          osErrorMsg(code)
3б. Если нужно возбудить исключение:   raiseOSError(code, extraInfo)
3в. Если нужен объект исключения:      newOSError(code, extraInfo)
```

```nim
import std/oserrors, std/posix

proc readLink(path: string): string =
  var buf = newString(4096)
  let n = readlink(path.cstring, buf.cstring, buf.len.csize_t)
  if n < 0:
    raiseOSError(osLastError(), path)   # шаги 2 + 3б совместно
  buf.setLen(n)
  buf
```

---

## Сводная таблица

| Символ | Вид | Назначение |
|--------|-----|------------|
| `OSErrorCode` | тип | Типизированная обёртка над числом ошибки ОС |
| `==` | оператор | Сравнить два кода ошибки |
| `$` | оператор | Представить код как десятичную строку |
| `osLastError()` | процедура | Захватить последний код ошибки ОС |
| `osErrorMsg(code)` | процедура | Перевести код в читаемое описание |
| `newOSError(code, info)` | процедура | Создать объект исключения `OSError` |
| `raiseOSError(code, info)` | процедура | Создать и немедленно возбудить `OSError` |

---

> **Смотри также:** [`system.OSError`](https://nim-lang.org/docs/system.html#OSError) — тип исключения, который возбуждает этот модуль; [`std/os`](https://nim-lang.org/docs/os.html) — высокоуровневые операции ОС, использующие `oserrors` внутри.
