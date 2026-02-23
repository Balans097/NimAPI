# Модуль `reservedmem` для Nim — полный справочник

## Обзор

Модуль `reservedmem` предоставляет две структуры данных для управления **резервированием виртуального адресного пространства**: `ReservedMem` (сырой байтовый буфер) и `ReservedMemSeq[T]` (типизированная последовательность поверх него). Обе разделяют одну фундаментальную гарантию:

> Зарезервируй большую область виртуального адресного пространства заранее, а затем выделяй физическую память (коммит) только для страниц, которые реально используются — и базовый адрес буфера **никогда не меняется**.

В этом принципиальное отличие от обычного `seq[T]` Nim. `seq` растёт, выделяя новый, больший блок и копируя туда всё содержимое — старый указатель становится недействительным. `ReservedMemSeq` никогда не копирует данные, потому что уже занял максимальный адресный диапазон у ОС. По мере необходимости страницы выделяются на месте; когда страницы перестают использоваться, их можно вернуть ОС (декоммит), при этом виртуальный адресный диапазон остаётся стабильным.

### Модель виртуальной памяти в двух словах

Современные ОС предоставляют каждому процессу большое виртуальное адресное пространство (обычно 128 ТиБ на 64-битном Linux или 8 ТиБ на 64-битном Windows). Отображение (маппинг) области этого пространства практически ничего не стоит — оно не потребляет RAM или своп. Физическая память расходуется только при **коммите** (или `mprotect` на POSIX).

`reservedmem` использует это следующим образом:
1. **Резервирует** максимальный размер одним системным вызовом при инициализации (дёшево).
2. **Коммитит** страницы постепенно по мере того, как `setLen` / `add` запрашивают больше пространства.
3. **Декоммитит** страницы при значительном уменьшении используемой области через `setLen`.

### Когда использовать этот модуль

- Нужен буфер или массив, который **не должен перемещаться** (например, вы раздаёте сырые указатели или срезы другому коду).
- Известна верхняя граница размера, но точный размер заранее неизвестен.
- Нужно избежать затрат на копирование при росте (полезно для больших буферов или горячих путей исполнения).
- Вы строите пользовательские аллокаторы, кольцевые буферы или структуры данных, похожие на memory-mapped файлы.

### Стабильность API

Помечен как **Unstable API** — сигнатуры или поведение могут измениться без предупреждения об устаревании.

---

## Экспортируемый API — краткий обзор

### Утилиты для указателей

| Символ | Вид | Описание |
|--------|-----|---------|
| `distance` | шаблон | Знаковое байтовое расстояние между двумя указателями |
| `shift` | шаблон | Сдвинуть указатель на заданное число байт |

### Флаги доступа к памяти

| Константа | Windows | POSIX | Значение |
|-----------|---------|-------|---------|
| `memExec` | `PAGE_EXECUTE` | `PROT_EXEC` | Только выполнение |
| `memExecRead` | `PAGE_EXECUTE_READ` | `PROT_EXEC\|PROT_READ` | Выполнение + чтение |
| `memExecReadWrite` | `PAGE_EXECUTE_READWRITE` | `PROT_EXEC\|PROT_READ\|PROT_WRITE` | Выполнение + чтение + запись |
| `memRead` | `PAGE_READONLY` | `PROT_READ` | Только чтение |
| `memReadWrite` | `PAGE_READWRITE` | `PROT_READ\|PROT_WRITE` | Чтение + запись (по умолчанию) |

### Типы

| Тип | Описание |
|-----|---------|
| `MemAccessFlags` | Целочисленный псевдоним флагов защиты страниц ОС |
| `ReservedMem` | Сырой байтовый буфер над зарезервированным адресным диапазоном |
| `ReservedMemSeq[T]` | Типизированная последовательность над зарезервированным диапазоном |

### API `ReservedMem`

| Символ | Вид | Описание |
|--------|-----|---------|
| `ReservedMem.init` | процедура | Зарезервировать и опционально закоммитить начальную область |
| `len` | функция | Количество байт в используемой области |
| `commitedLen` | функция | Количество байт, закоммиченных (обеспеченных RAM) |
| `maxLen` | функция | Общее количество байт в зарезервированном диапазоне |
| `setLen` | процедура | Изменить размер используемой области, коммитя или декоммитя страницы |

### API `ReservedMemSeq[T]`

| Символ | Вид | Описание |
|--------|-----|---------|
| `ReservedMemSeq.init` | процедура | Зарезервировать и опционально закоммитить хранилище типизированных элементов |
| `len` | функция | Количество элементов в последовательности |
| `commitedLen` | функция | Количество элементов, для которых память закоммичена |
| `maxLen` | функция | Максимальное количество элементов, которое может хранить резервация |
| `setLen` | процедура | Изменить размер последовательности |
| `add` | процедура | Добавить один элемент в конец |
| `pop` | процедура | Удалить и вернуть последний элемент |
| `[]` | функция | Индексный доступ (по позиции или `BackwardsIndex`) |

---

## Шаблоны для работы с указателями

### `distance`

```nim
template distance*(lhs, rhs: pointer): int
```

Возвращает **знаковое байтовое расстояние** от указателя `lhs` до указателя `rhs`, вычисленное как `cast[int](rhs) - cast[int](lhs)`. Положительный результат означает, что `rhs` находится по более высокому адресу; отрицательный — по более низкому.

Шаблон существует потому, что Nim не допускает арифметику непосредственно над `pointer` — сначала нужно привести к `int`. `distance` скрывает это приведение типов.

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 4096, initLen = 100)
# Расстояние от начала до конца используемой области:
echo distance(m.mem.memStart, m.mem.usedMemEnd)   # 100
```

---

### `shift`

```nim
template shift*(p: pointer, distance: int): pointer
```

Возвращает новый указатель, который находится на `distance` байт впереди `p` в памяти. Отрицательные значения сдвигают назад. Эквивалент C-выражения `(char*)p + distance`.

Как и `distance`, существует, чтобы избежать повторяющегося `cast[pointer](cast[int](p) + d)`.

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 1024, initLen = 64)
let endPtr = shift(m.mem.memStart, m.len)  # указатель на конец используемой области
```

---

## Константы флагов доступа к памяти

Эти константы — Nim-эквиваленты флагов защиты страниц ОС, передаваемых в `VirtualAlloc` (Windows) или `mmap`/`mprotect` (POSIX). Они определяют, какие операции допустимы над закоммиченными страницами.

| Константа | Чтение | Запись | Выполнение | Типичное применение |
|-----------|--------|--------|-----------|---------------------|
| `memRead` | ✓ | — | — | Данные только для чтения |
| `memReadWrite` | ✓ | ✓ | — | Обычные данные (**по умолчанию**) |
| `memExec` | — | — | ✓ | Редко используется отдельно |
| `memExecRead` | ✓ | — | ✓ | JIT: чтение + выполнение, без записи |
| `memExecReadWrite` | ✓ | ✓ | ✓ | JIT: запись кода, затем выполнение |

Константа `memNoAccess` (недоступные страницы) определена внутри, но не экспортируется — она используется для начального резервирования на Windows (`PAGE_NOACCESS`) и как POSIX-эквивалент `PROT_NONE` для `mmap`.

```nim
import std/reservedmem

# Буфер только для чтения:
var roMem = ReservedMem.init(maxLen = 65536, accessFlags = memRead)

# Буфер для JIT-кода:
var codeBuf = ReservedMem.init(maxLen = 1024 * 1024,
                               accessFlags = memExecReadWrite)
```

---

## `ReservedMem`

```nim
type ReservedMem* = object
```

Объект, управляющий зарезервированной областью виртуального адресного пространства. Внутри отслеживает четыре указателя в эту область:

| Внутреннее поле | Значение |
|----------------|---------|
| `memStart` | Базовый адрес зарезервированной области (константен после init) |
| `usedMemEnd` | Один байт за последним байтом «используемой» части |
| `committedMemEnd` | Один байт за последним байтом, обеспеченным физической памятью |
| `memEnd` | Один байт за всей зарезервированной областью |

Инвариант всегда соблюдается: `memStart ≤ usedMemEnd ≤ committedMemEnd ≤ memEnd`.

Физическая память коммитится **выровненными по страницам блоками** (гранулярность выделения ОС — обычно 4 КиБ на POSIX и 64 КиБ на Windows). Поэтому закоммиченная область может быть больше используемой.

---

### `ReservedMem.init`

```nim
proc init*(T: type ReservedMem,
           maxLen: Natural,
           initLen: Natural = 0,
           initCommitLen = initLen,
           memStart = pointer(nil),
           accessFlags = memReadWrite,
           maxCommittedAndUnusedPages = 3): ReservedMem
```

Создаёт и возвращает новый `ReservedMem`. Это единственный конструктор — аналога `new` нет.

#### Параметры

| Параметр | По умолчанию | Описание |
|----------|-------------|---------|
| `maxLen` | *(обязательный)* | Общее количество байт для резервирования в виртуальном адресном пространстве. Это жёсткий верхний предел — буфер никогда не сможет вырасти за его пределы. **Не** потребляет RAM. |
| `initLen` | `0` | Начальный «используемый» размер в байтах. Байты в `[0, initLen)` считаются валидными данными сразу после инициализации. |
| `initCommitLen` | `initLen` | Сколько байт сразу обеспечить физической памятью. Должно быть ≥ `initLen`. Внутри округляется вверх до размера страницы ОС. |
| `memStart` | `nil` | Подсказка ОС о желаемом расположении резервации. `nil` — ОС выбирает сама (настоятельно рекомендуется). |
| `accessFlags` | `memReadWrite` | Флаги защиты страниц для всех закоммиченных страниц. |
| `maxCommittedAndUnusedPages` | `3` | Сколько лишних закоммиченных, но неиспользуемых страниц допустимо до декоммита при уменьшении через `setLen`. Большее значение снижает частоту декоммит/рекоммит циклов ценой бо́льшего расхода RAM. |

`initLen ≤ initCommitLen` проверяется через `assert`; нарушение приводит к краше в режиме отладки.

#### Что происходит на уровне ОС

**Windows:** `VirtualAlloc(MEM_RESERVE)` резервирует адресный диапазон, затем `VirtualAlloc(MEM_COMMIT)` обеспечивает начальные страницы памятью.

**POSIX:** `mmap(PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS)` резервирует без выделения памяти; `mprotect(accessFlags)` обеспечивает начальные страницы.

#### Примеры

```nim
import std/reservedmem

# Зарезервировать 1 МиБ, пока ничего не коммитить:
var m1 = ReservedMem.init(maxLen = 1024 * 1024)
assert m1.len == 0
assert m1.commitedLen == 0
assert m1.maxLen == 1024 * 1024

# Зарезервировать 1 МиБ, сразу закоммитить и пометить 64 КиБ как используемые:
var m2 = ReservedMem.init(maxLen = 1024 * 1024,
                          initLen = 65536,
                          initCommitLen = 65536)
assert m2.len == 65536
assert m2.maxLen == 1024 * 1024
```

---

### `len` (ReservedMem)

```nim
func len*(m: ReservedMem): int
```

Возвращает количество байт в **используемой** части буфера, то есть `distance(memStart, usedMemEnd)`. Это логический размер, с которым работает ваш код; закоммиченные, но неиспользуемые байты за этой границей невидимы.

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 4096, initLen = 100)
assert m.len == 100
```

---

### `commitedLen` (ReservedMem)

```nim
func commitedLen*(m: ReservedMem): int
```

Возвращает количество байт, в данный момент **обеспеченных физической памятью** — `distance(memStart, committedMemEnd)`. Это значение всегда ≥ `len` и всегда кратно размеру страницы ОС. Полезно для диагностики и для определения того, вызовет ли следующий `setLen` коммит страниц.

> **Примечание:** Имя `commitedLen` (с одним `t`) соответствует написанию в исходном коде — это не опечатка в справочнике.

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 1024 * 1024, initLen = 1)
# commitedLen округляется до размера страницы (например, 4096 на Linux):
assert m.commitedLen >= 1
assert m.commitedLen mod 4096 == 0   # выровнено по границе страницы
assert m.commitedLen >= m.len
```

---

### `maxLen` (ReservedMem)

```nim
func maxLen*(m: ReservedMem): int
```

Возвращает общее количество байт в зарезервированном адресном диапазоне — `distance(memStart, memEnd)`. Это предел, переданный в `init` как `maxLen`. `setLen` выдаст ошибку, если попытаться расти за этот предел.

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 1024 * 1024)
assert m.maxLen == 1024 * 1024
```

---

### `setLen` (ReservedMem)

```nim
proc setLen*(m: var ReservedMem, newLen: int)
```

Изменяет используемый размер буфера до `newLen` байт.

**Увеличение** (`newLen > len`): если новая граница выходит за пределы текущей закоммиченной области, коммитятся дополнительные страницы (Windows: `VirtualAlloc(MEM_COMMIT)`; POSIX: `mprotect` с сохранёнными `accessFlags`). Расширение коммита всегда округляется вверх до гранулярности страниц ОС.

**Уменьшение** (`newLen < len`): указатель используемой области перемещается назад. Если зазор между `usedMemEnd` и `committedMemEnd` теперь превышает `maxCommittedAndUnusedPages * pageSize`, лишние страницы декоммитятся (Windows: `VirtualFree(MEM_DECOMMIT)`; POSIX: `posix_madvise(POSIX_MADV_DONTNEED)`), освобождая физическую RAM при сохранении виртуального адресного диапазона.

**Гарантия стабильности:** базовый адрес `memStart` никогда не меняется, сколько бы циклов роста и уменьшения ни произошло.

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 4096 * 100)

m.setLen(4096)        # коммитим первую страницу
assert m.len == 4096

m.setLen(4096 * 10)   # коммитим ещё страницы
assert m.len == 4096 * 10

m.setLen(0)           # уменьшаем; лишние страницы могут быть декоммичены
assert m.len == 0
```

#### Запись данных после setLen

`setLen` не инициализирует вновь закоммиченную память — это ваша ответственность:

```nim
import std/reservedmem

var m = ReservedMem.init(maxLen = 1024)
m.setLen(64)

let buf = cast[ptr array[64, byte]](m.mem.memStart)
for i in 0 ..< 64:
  buf[i] = byte(i)
```

---

## `ReservedMemSeq[T]`

```nim
type ReservedMemSeq*[T] = object
  mem: ReservedMem
```

Типизированная последовательность поверх `ReservedMem`. Внутри это не что иное, как `ReservedMem`, где `sizeof(T)` используется как коэффициент пересчёта между количеством элементов и байтовыми смещениями. Предоставляет высокоуровневый, `seq`-подобный API (`[]`, `add`, `pop`, `setLen`) с гарантией стабильного адреса.

**Важные ограничения по сравнению с `seq[T]`:**

- Нет автоматического управления памятью — нужно явно вызвать `setLen(0)` или дать объекту выйти из области видимости (но деструктора нет, поэтому память ОС освобождается только при завершении процесса, или если вызвать OS-API вручную). Деструкторы помечены как TODO в исходном коде.
- Нет копирования при росте.
- Проверка границ через `rangeCheck` (отключается с `-d:danger`).
- `pop` **не** запускает деструктор удалённого элемента (TODO в исходном коде).

---

### `ReservedMemSeq.init`

```nim
proc init*(SeqType: type ReservedMemSeq,
           maxLen: Natural,
           initLen: Natural = 0,
           initCommitLen: Natural = 0,
           memStart = pointer(nil),
           accessFlags = memReadWrite,
           maxCommittedAndUnusedPages = 3): SeqType
```

Создаёт `ReservedMemSeq[T]`. Все размерные параметры указываются в **количестве элементов**, а не байтах — процедура внутри умножает на `sizeof(T)` перед вызовом `ReservedMem.init`.

#### Параметры

| Параметр | По умолчанию | Описание |
|----------|-------------|---------|
| `maxLen` | *(обязательный)* | Максимальное количество элементов, которое когда-либо сможет содержать последовательность |
| `initLen` | `0` | Количество элементов, считающихся используемыми сразу после создания |
| `initCommitLen` | `0` | Количество элементов, хранилище для которых физически закоммичено при инициализации |
| `memStart` | `nil` | Подсказка ОС о базовом адресе (оставьте `nil`) |
| `accessFlags` | `memReadWrite` | Флаги защиты страниц |
| `maxCommittedAndUnusedPages` | `3` | Порог декоммита (в страницах ОС, не в элементах) |

```nim
import std/reservedmem

# Стабильная-адресная последовательность до 1 миллиона int, пока ничего не коммичено:
var s = ReservedMemSeq[int].init(maxLen = 1_000_000)
assert s.len == 0
assert s.maxLen == 1_000_000

# Предварительно закоммитить хранилище для 1000 элементов:
var t = ReservedMemSeq[float64].init(maxLen = 100_000, initCommitLen = 1000)
assert t.commitedLen >= 1000
```

---

### `len` (ReservedMemSeq)

```nim
func len*[T](s: ReservedMemSeq[T]): int
```

Возвращает количество элементов в последовательности (`mem.len div sizeof(T)`).

```nim
import std/reservedmem

var s = ReservedMemSeq[int32].init(maxLen = 1000, initLen = 5)
assert s.len == 5
```

---

### `commitedLen` (ReservedMemSeq)

```nim
func commitedLen*[T](s: ReservedMemSeq[T]): int
```

Возвращает количество элементов, для которых физическая память закоммичена (`mem.commitedLen div sizeof(T)`). Может быть больше `len` из-за округления по гранулярности страниц.

---

### `maxLen` (ReservedMemSeq)

```nim
func maxLen*[T](s: ReservedMemSeq[T]): int
```

Возвращает максимальное количество элементов, которое последовательность может когда-либо содержать без ошибки (`mem.maxLen div sizeof(T)`).

---

### `setLen` (ReservedMemSeq)

```nim
proc setLen*[T](s: var ReservedMemSeq[T], newLen: int)
```

Изменяет размер последовательности до `newLen` элементов, коммитя или декоммитя страницы ОС по мере необходимости. Поведение зеркалит `ReservedMem.setLen`, масштабированное на `sizeof(T)`.

> **Примечание:** Деструкторы удалённых элементов **не вызываются** (помечено как TODO в исходном коде). Для типов с нетривиальной очисткой вызывайте деструкторы вручную перед уменьшением размера.

```nim
import std/reservedmem

var s = ReservedMemSeq[int].init(maxLen = 10_000)
s.setLen(500)
assert s.len == 500
s.setLen(100)
assert s.len == 100
```

---

### `add`

```nim
proc add*[T](s: var ReservedMemSeq[T], val: T)
```

Добавляет один элемент в конец последовательности. Внутри вызывает `setLen(len + 1)`, затем присваивает `val` новому последнему слоту. Может вызвать коммит страницы, если новый элемент выходит за пределы закоммиченной области.

```nim
import std/reservedmem

var s = ReservedMemSeq[int].init(maxLen = 10_000)
s.add(42)
s.add(99)
assert s.len == 2
assert s[0] == 42
assert s[1] == 99
```

---

### `pop`

```nim
proc pop*[T](s: var ReservedMemSeq[T]): T
```

Удаляет и возвращает последний элемент последовательности. После чтения значения вызывает `setLen(len - 1)`, что может вызвать декоммит страниц.

Проверяет, что последовательность не пуста (`usedMemEnd != memStart`); вызов `pop` на пустой последовательности приводит к крашу в режиме отладки.

> **Примечание:** Деструктор удалённого элемента **не вызывается**.

```nim
import std/reservedmem

var s = ReservedMemSeq[int].init(maxLen = 100)
s.add(10)
s.add(20)
s.add(30)

assert s.pop() == 30
assert s.pop() == 20
assert s.len == 1
```

---

### `[]` — доступ к элементам

```nim
func `[]`*[T](s: ReservedMemSeq[T], pos: Natural): lent T
func `[]`*[T](s: var ReservedMemSeq[T], pos: Natural): var T
func `[]`*[T](s: ReservedMemSeq[T], rpos: BackwardsIndex): lent T
func `[]`*[T](s: var ReservedMemSeq[T], rpos: BackwardsIndex): var T
```

Предоставляет индексный доступ к элементам. Поддерживается как прямая индексация (`Natural`), так и обратная (`BackwardsIndex`, записывается как `s[^1]`). Изменяемый (`var`) доступ доступен, когда `s` — переменная `var`.

Границы проверяются через `rangeCheck` — выход за пределы вызывает исключение в обычных сборках и является неопределённым поведением с `-d:danger`.

```nim
import std/reservedmem

var s = ReservedMemSeq[float64].init(maxLen = 1000)
s.add(1.5)
s.add(2.5)
s.add(3.5)

assert s[0] == 1.5
assert s[2] == 3.5
assert s[^1] == 3.5   # BackwardsIndex
assert s[^2] == 2.5

s[0] = 99.9           # изменяемый доступ
assert s[0] == 99.9
```

---

## Полные практические примеры

### Паттерн 1 — Стабильный указатель на растущий буфер журнала

```nim
import std/reservedmem

# Зарезервировать место для 1 миллиона записей — базовый адрес никогда не меняется
var log = ReservedMemSeq[uint64].init(maxLen = 1_000_000)

proc recordEvent(id: uint64) =
  log.add(id)

recordEvent(1)
recordEvent(2)
recordEvent(3)

# Сырой указатель, взятый до вызовов add, по-прежнему действителен:
let firstEntry = cast[ptr uint64](log.mem.memStart)
assert firstEntry[] == 1   # действительно — базовый адрес не изменился
```

---

### Паттерн 2 — Использование как стека

```nim
import std/reservedmem

var stack = ReservedMemSeq[int].init(maxLen = 10_000)

# Push
for i in 1..5:
  stack.add(i)

# Pop
while stack.len > 0:
  echo stack.pop()   # выводит 5 4 3 2 1
```

---

### Паттерн 3 — Предварительный коммит для кода с жёсткими требованиями к задержке

В некоторых сценариях (аудио реального времени, игровые циклы) не нужно, чтобы первые несколько вызовов `add` вызывали page fault ОС. Предварительно закоммитьте хранилище, которое точно понадобится:

```nim
import std/reservedmem

const MaxFrameObjects = 8192

var objects = ReservedMemSeq[int64].init(
  maxLen = 1_000_000,
  initCommitLen = MaxFrameObjects   # предварительно закоммитить первые 8192 слота
)

# Первые 8192 вызовов add не вызывают page fault:
for i in 0 ..< MaxFrameObjects:
  objects.add(int64(i))
```

---

### Паттерн 4 — Сырой байтовый буфер для пользовательского сериализатора

```nim
import std/reservedmem

var buf = ReservedMem.init(maxLen = 16 * 1024 * 1024)   # 16 МиБ максимум

proc writeInt32(m: var ReservedMem, value: int32) =
  let offset = m.len
  m.setLen(offset + 4)
  cast[ptr int32](m.mem.memStart.shift(offset))[] = value

proc writeBytes(m: var ReservedMem, data: openArray[byte]) =
  let offset = m.len
  m.setLen(offset + data.len)
  let dst = m.mem.memStart.shift(offset)
  copyMem(dst, unsafeAddr data[0], data.len)

buf.writeInt32(42)
buf.writeInt32(-1)
echo buf.len   # 8
```

---

## Условия возникновения ошибок

Все OS-операции проверяются. При сбое вызывается `raiseOSError(osLastError())`, которая генерирует `OSError` с сообщением об ошибке платформы.

| Ситуация | Поведение |
|----------|----------|
| `init` — ОС отказывает в резервировании | Выбрасывается `OSError` |
| `setLen` — коммит не удался | Выбрасывается `OSError` |
| `setLen` — декоммит/madvise не удался | Выбрасывается `OSError` |
| `initLen > initCommitLen` | `AssertionDefect` (в режиме отладки) |
| `[]` — выход за пределы | `RangeDefect` (если не `-d:danger`) |
| `pop` на пустой последовательности | `AssertionDefect` (в режиме отладки) |

---

## Смотрите также

- [`std/oserrors`](https://nim-lang.org/docs/oserrors.html) — `raiseOSError`, `osLastError`
- [`std/posix`](https://nim-lang.org/docs/posix.html) — `mmap`, `mprotect`, `posix_madvise`
- [Документация Windows VirtualAlloc](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualalloc)
- [man-страница mmap(2) Linux](https://man7.org/linux/man-pages/man2/mmap.2.html)
