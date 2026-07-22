# system/atomics — справочник модуля

> **Импорт:** `import system/atomics`
> **Область применения:** низкоуровневые атомарные операции над памятью для многопоточного кода — атомарные чтение/запись, обмен, сравнение-с-обменом (CAS), инкремент/декремент, битовые операции и барьеры памяти.

Модуль — это компилятороспецифичная обвязка над атомарными интринсиками GCC/Clang (`__atomic_*`) и MSVC (`_Interlocked*`), плюс несколько платформонезависимых процедур поверх них (`atomicInc`, `atomicDec`, `cas`, `fence`, `cpuRelax`). Общая конвенция модуля: почти каждая процедура принимает параметр `mem: AtomMemModel`, задающий модель памяти (барьер), с которой выполняется операция — от `ATOMIC_RELAXED` (без гарантий упорядочивания) до `ATOMIC_SEQ_CST` (полный барьер, самый безопасный и самый медленный вариант). Часть процедур существует в двух формах: "N"-форма (`atomicLoadN`, `atomicStoreN`, ...) работает со значением напрямую, обычная форма (`atomicLoad`, `atomicStore`, ...) — через указатель на второй операнд/результат (обобщённый, "generic", вариант интринсика). Модуль считается низкоуровневым: в прикладном коде почти всегда предпочтительнее модуль `std/atomics`, а этот файл — фундамент, на котором `std/atomics` построен.

---

## Оглавление

I. [Типы и константы](#i-типы-и-константы)
&nbsp;&nbsp;&nbsp;1. [`AtomType`](#1-atomtype)
&nbsp;&nbsp;&nbsp;2. [`AtomMemModel` и константы `ATOMIC_*`](#2-atommemmodel-и-константы-atomic)

II. [Атомарная загрузка и запись](#ii-атомарная-загрузка-и-запись)
&nbsp;&nbsp;&nbsp;1. [`atomicLoadN`](#1-atomicloadn)
&nbsp;&nbsp;&nbsp;2. [`atomicLoad`](#2-atomicload)
&nbsp;&nbsp;&nbsp;3. [`atomicStoreN`](#3-atomicstoren)
&nbsp;&nbsp;&nbsp;4. [`atomicStore`](#4-atomicstore)

III. [Атомарный обмен и сравнение-с-обменом](#iii-атомарный-обмен-и-сравнение-с-обменом)
&nbsp;&nbsp;&nbsp;1. [`atomicExchangeN`](#1-atomicexchangen)
&nbsp;&nbsp;&nbsp;2. [`atomicExchange`](#2-atomicexchange)
&nbsp;&nbsp;&nbsp;3. [`atomicCompareExchangeN`](#3-atomiccompareexchangen)
&nbsp;&nbsp;&nbsp;4. [`atomicCompareExchange`](#4-atomiccompareexchange)
&nbsp;&nbsp;&nbsp;5. [`cas`](#5-cas)

IV. [Арифметические и битовые атомарные операции](#iv-арифметические-и-битовые-атомарные-операции)
&nbsp;&nbsp;&nbsp;1. [`atomic*Fetch` (вернуть новое значение)](#1-atomicfetch-вернуть-новое-значение)
&nbsp;&nbsp;&nbsp;2. [`atomicFetch*` (вернуть старое значение)](#2-atomicfetch-вернуть-старое-значение)
&nbsp;&nbsp;&nbsp;3. [`atomicInc`, `atomicDec`](#3-atomicinc-atomicdec)
&nbsp;&nbsp;&nbsp;4. [`addAndFetch`](#4-addandfetch)

V. [Флаги, барьеры и служебные процедуры](#v-флаги-барьеры-и-служебные-процедуры)
&nbsp;&nbsp;&nbsp;1. [`atomicTestAndSet`, `atomicClear`](#1-atomictestandset-atomicclear)
&nbsp;&nbsp;&nbsp;2. [`atomicThreadFence`, `atomicSignalFence`, `fence`](#2-atomicthreadfence-atomicsignalfence-fence)
&nbsp;&nbsp;&nbsp;3. [`cpuRelax`](#3-cpurelax)
&nbsp;&nbsp;&nbsp;4. [`atomicAlwaysLockFree`, `atomicIsLockFree`](#4-atomicalwayslockfree-atomicislockfree)

VI. [Практические рецепты](#vi-практические-рецепты)
&nbsp;&nbsp;&nbsp;1. [Атомарный счётчик ссылок](#1-атомарный-счётчик-ссылок)
&nbsp;&nbsp;&nbsp;2. [Однократная ленивая инициализация (double-checked locking)](#2-однократная-ленивая-инициализация-double-checked-locking)
&nbsp;&nbsp;&nbsp;3. [Спин-блокировка на `atomicTestAndSet`](#3-спин-блокировка-на-atomictestandset)
&nbsp;&nbsp;&nbsp;4. [Безблокировочный стек (Treiber stack) на `cas`](#4-безблокировочный-стек-treiber-stack-на-cas)

VII. [Краткая таблица](#vii-краткая-таблица)

VIII. [Сводка: какую процедуру выбрать](#viii-сводка-какую-процедуру-выбрать)

---

## I. Типы и константы

### 1. `AtomType`

```nim
type
  AtomType* = SomeNumber|pointer|ptr|char|bool
```

**Что делает.** Это не тип в обычном смысле, а type class (класс типов) — ограничение, которое компилятор проверяет при инстанцировании generic-процедур модуля. Любая процедура вида `proc atomicXxx*[T: AtomType](...)` соглашается работать только с числами (`SomeNumber` — все целые и вещественные типы), нетипизированными указателями (`pointer`), типизированными указателями (`ptr`), символами (`char`) и булевыми значениями (`bool`). Смысл ограничения — атомарные интринсики процессора умеют работать только со значениями фиксированного маленького размера (1/2/4/8/16 байт), которые можно скопировать простым побитовым копированием; на структуры произвольного размера или seq/string атомарные операции напрямую не распространяются.

- **Параметры:** отсутствуют — это ограничение типа, а не процедура.

**Пример:**

```nim
# ЗАМЕЧАНИЕ: AtomType используется неявно, как ограничение generic-параметра
var counter: int = 0
discard atomicAddFetch(addr(counter), 1, ATOMIC_SEQ_CST) # int входит в SomeNumber -> AtomType
```

---

### 2. `AtomMemModel` и константы `ATOMIC_*`

```nim
type AtomMemModel* = distinct cint

var ATOMIC_RELAXED* {.importc: "__ATOMIC_RELAXED", nodecl.}: AtomMemModel
var ATOMIC_CONSUME* {.importc: "__ATOMIC_CONSUME", nodecl.}: AtomMemModel
var ATOMIC_ACQUIRE* {.importc: "__ATOMIC_ACQUIRE", nodecl.}: AtomMemModel
var ATOMIC_RELEASE* {.importc: "__ATOMIC_RELEASE", nodecl.}: AtomMemModel
var ATOMIC_ACQ_REL* {.importc: "__ATOMIC_ACQ_REL", nodecl.}: AtomMemModel
var ATOMIC_SEQ_CST* {.importc: "__ATOMIC_SEQ_CST", nodecl.}: AtomMemModel
```

**Что делает.** `AtomMemModel` — обёртка над целочисленным кодом модели памяти, который компилятор C передаёт своим `__atomic_*` интринсикам. Шесть констант — это шесть уровней "строгости" барьера:

- `ATOMIC_RELAXED` — никаких гарантий упорядочивания относительно других операций, гарантируется только атомарность самой операции;
- `ATOMIC_CONSUME` — упорядочивание только по цепочке зависимостей данных (на практике многие компиляторы трактуют его как `ATOMIC_ACQUIRE`);
- `ATOMIC_ACQUIRE` — барьер "не поднимать код выше" этой точки, используется для чтения;
- `ATOMIC_RELEASE` — барьер "не опускать код ниже" этой точки, используется для записи;
- `ATOMIC_ACQ_REL` — оба барьера сразу, для операций "прочитать-изменить-записать";
- `ATOMIC_SEQ_CST` — полный барьер и единый глобальный порядок для всех потоков; самый безопасный выбор по умолчанию, если нет причин выбирать более слабую модель ради производительности.

Аналогия: если барьер `ATOMIC_ACQUIRE`/`ATOMIC_RELEASE` — это шлагбаум, пропускающий движение только в одну сторону (запрещающий "заглядывать" в будущее или "опаздывать" из прошлого), то `ATOMIC_SEQ_CST` — это полностью перекрытый перекрёсток с единым светофором на все потоки одновременно.

- **Параметры:** константы, не принимают аргументов.

**Пример:**

```nim
var flag: bool = false
atomicStoreN(addr(flag), true, ATOMIC_RELEASE)      # публикуем результат работы
let v = atomicLoadN(addr(flag), ATOMIC_ACQUIRE)     # видим её не раньше, чем была опубликована
echo v # выводит true
```

---

## II. Атомарная загрузка и запись

### 1. `atomicLoadN`

```nim
proc atomicLoadN*[T: AtomType](p: ptr T, mem: AtomMemModel): T
```

**Что делает.** Атомарно читает значение по адресу `p` и возвращает его. В отличие от обычного разыменования `p[]`, гарантирует, что чтение не будет разбито компилятором или процессором на части и что оно упорядочено относительно других потоков согласно `mem`. Допустимые модели: `ATOMIC_RELAXED`, `ATOMIC_SEQ_CST`, `ATOMIC_ACQUIRE`, `ATOMIC_CONSUME` — модели, подразумевающие запись (`ATOMIC_RELEASE`, `ATOMIC_ACQ_REL`), для чтения не имеют смысла.

- `p: ptr T` — указатель на читаемую ячейку, не изменяется.
- `mem: AtomMemModel` — модель памяти для операции чтения.

**Пример:**

```nim
var x: int = 42
let y = atomicLoadN(addr(x), ATOMIC_SEQ_CST)
echo y # выводит 42
```

---

### 2. `atomicLoad`

```nim
proc atomicLoad*[T: AtomType](p, ret: ptr T, mem: AtomMemModel)
```

**Что делает.** Обобщённый (не "N") вариант загрузки: вместо возврата значения через `result`, пишет прочитанное значение по адресу `ret`. Используется компилятором в случаях, когда тип `T` слишком велик, чтобы эффективно возвращать его по значению, либо когда генерируется код без выделенного регистра возврата.

- `p: ptr T` — адрес, из которого читаем, не изменяется.
- `ret: ptr T` — адрес, куда атомарно прочитанное значение будет записано (изменяется).
- `mem: AtomMemModel` — модель памяти чтения.

**Пример:**

```nim
var source: int = 7
var dest: int
atomicLoad(addr(source), addr(dest), ATOMIC_ACQUIRE)
echo dest # выводит 7
```

---

### 3. `atomicStoreN`

```nim
proc atomicStoreN*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel)
```

**Что делает.** Атомарно записывает `val` по адресу `p`. Допустимые модели — `ATOMIC_RELAXED`, `ATOMIC_SEQ_CST`, `ATOMIC_RELEASE` (модели, подразумевающие чтение, для записи не подходят).

- `p: ptr T` — адрес назначения (изменяется).
- `val: T` — записываемое значение, не изменяется.
- `mem: AtomMemModel` — модель памяти записи.

**Пример:**

```nim
var ready: bool = false
atomicStoreN(addr(ready), true, ATOMIC_RELEASE)
echo ready # выводит true
```

---

### 4. `atomicStore`

```nim
proc atomicStore*[T: AtomType](p, val: ptr T, mem: AtomMemModel)
```

**Что делает.** Обобщённый вариант записи: значение передаётся не по значению, а через указатель `val`. Семантически эквивалентен `atomicStoreN(p, val[], mem)`, но соответствует сигнатуре интринсика `__atomic_store` (в отличие от `__atomic_store_n`).

- `p: ptr T` — адрес назначения (изменяется).
- `val: ptr T` — адрес источника значения, не изменяется самим вызовом.
- `mem: AtomMemModel` — модель памяти записи.

**Пример:**

```nim
var dest: int
let src = 99
atomicStore(addr(dest), unsafeAddr(src), ATOMIC_SEQ_CST)
echo dest # выводит 99
```

---

## III. Атомарный обмен и сравнение-с-обменом

### 1. `atomicExchangeN`

```nim
proc atomicExchangeN*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**Что делает.** Атомарно записывает `val` по адресу `p` и возвращает то значение, которое там было *до* записи — операция "обменяться" выполняется как единое неделимое действие, без окна, в котором другой поток мог бы вклиниться между чтением старого и записью нового значения.

- **Разбор реализации.** Ключевое отличие от последовательности "`atomicLoadN` + `atomicStoreN`" — атомарность всей пары операций целиком: конкурирующий поток либо видит состояние "до", либо "после", но не может вклиниться между ними. Допустимы все шесть моделей памяти.

- `p: ptr T` — адрес ячейки (изменяется).
- `val: T` — новое значение, не изменяется.
- `mem: AtomMemModel` — модель памяти операции.

**Пример:**

```nim
var lockFlag: int = 0
let old = atomicExchangeN(addr(lockFlag), 1, ATOMIC_ACQUIRE)
echo old # выводит 0 — до захвата блокировка была свободна
```

---

### 2. `atomicExchange`

```nim
proc atomicExchange*[T: AtomType](p, val, ret: ptr T, mem: AtomMemModel)
```

**Что делает.** Обобщённая версия обмена: новое значение читается из `val`, а старое значение по адресу `p` сохраняется в `ret`, вместо возврата через `result`.

- `p: ptr T` — адрес ячейки (изменяется).
- `val: ptr T` — адрес нового значения, не изменяется.
- `ret: ptr T` — адрес, куда сохраняется прежнее значение (изменяется).
- `mem: AtomMemModel` — модель памяти операции.

**Пример:**

```nim
var cell: int = 5
let newVal = 10
var oldVal: int
atomicExchange(addr(cell), unsafeAddr(newVal), addr(oldVal), ATOMIC_SEQ_CST)
echo oldVal, " ", cell # выводит 5 10
```

---

### 3. `atomicCompareExchangeN`

```nim
proc atomicCompareExchangeN*[T: AtomType](p, expected: ptr T, desired: T,
  weak: bool, success_memmodel: AtomMemModel, failure_memmodel: AtomMemModel): bool
```

**Что делает.** Это фундаментальная операция CAS (compare-and-swap): сравнивает содержимое по адресу `p` с содержимым `expected[]`; если они равны — атомарно записывает `desired` по адресу `p` и возвращает `true`; если не равны — записывает актуальное (изменившееся) значение из `p` обратно в `expected[]` и возвращает `false`. Всё сравнение и запись происходят как одна неделимая операция.

- **Разбор реализации.** Параметр `weak` разрешает "ложный провал" (spurious failure) — на некоторых архитектурах (например, ARM с LL/SC-инструкциями) слабый CAS может вернуть `false`, даже если значения фактически совпадали, потому что реализация не гарантирует атомарность в редких случаях прерывания цепочки load-linked/store-conditional. Слабый вариант дешевле и обычно используется в циклах повторных попыток (retry loop), где случайный "провал" просто приводит к ещё одной итерации. Сильный вариант (`weak = false`) никогда не проваливается ложно, но может быть медленнее — используйте его, если CAS выполняется один раз, не в цикле. `failure_memmodel` не может быть строже, чем `success_memmodel`, и не может быть `ATOMIC_RELEASE`/`ATOMIC_ACQ_REL` (нет смысла в барьере "публикации" на пути неудачи, где ничего не записывается).

- `p: ptr T` — сравниваемая и потенциально изменяемая ячейка (изменяется при успехе).
- `expected: ptr T` — ожидаемое значение на входе; при неудаче перезаписывается фактическим значением из `p` (изменяется).
- `desired: T` — значение, которое будет записано при совпадении, не изменяется.
- `weak: bool` — `true` разрешает ложный провал (для циклов повторов), `false` — строгая семантика.
- `success_memmodel: AtomMemModel` — модель памяти при успешном обмене.
- `failure_memmodel: AtomMemModel` — модель памяти при неудаче.

**Пример:**

```nim
var value: int = 10
var expected: int = 10
let ok = atomicCompareExchangeN(addr(value), addr(expected), 20,
  false, ATOMIC_SEQ_CST, ATOMIC_SEQ_CST)
echo ok, " ", value # выводит true 20 — значение совпало и было заменено

var expectedFail: int = 999   # заведомо неверное ожидание
let failed = atomicCompareExchangeN(addr(value), addr(expectedFail), 30,
  false, ATOMIC_SEQ_CST, ATOMIC_SEQ_CST)
echo failed, " ", expectedFail # выводит false 20 — expectedFail получил актуальное значение
```

---

### 4. `atomicCompareExchange`

```nim
proc atomicCompareExchange*[T: AtomType](p, expected, desired: ptr T,
  weak: bool, success_memmodel: AtomMemModel, failure_memmodel: AtomMemModel): bool
```

**Что делает.** Полностью повторяет семантику `atomicCompareExchangeN`, но желаемое значение `desired` тоже передаётся по адресу, а не по значению — это соответствует интринсику `__atomic_compare_exchange` (в отличие от `__atomic_compare_exchange_n`) и применяется компилятором для крупных типов, которые дешевле передавать по указателю.

- `p: ptr T` — сравниваемая и потенциально изменяемая ячейка (изменяется при успехе).
- `expected: ptr T` — адрес ожидаемого значения; при неудаче перезаписывается (изменяется).
- `desired: ptr T` — адрес желаемого значения, не изменяется вызовом.
- `weak: bool` — см. `atomicCompareExchangeN`.
- `success_memmodel: AtomMemModel` — модель памяти при успехе.
- `failure_memmodel: AtomMemModel` — модель памяти при неудаче.

**Пример:**

```nim
var value: int = 1
var expected: int = 1
let desired = 2
let ok = atomicCompareExchange(addr(value), addr(expected), unsafeAddr(desired),
  false, ATOMIC_SEQ_CST, ATOMIC_SEQ_CST)
echo ok, " ", value # выводит true 2
```

---

### 5. `cas`

```nim
proc cas*[T: bool|int|ptr](p: ptr T; oldValue, newValue: T): bool
```

**Что делает.** Платформонезависимая обёртка над сравнением-с-обменом: если значение по адресу `p` равно `oldValue`, записывает туда `newValue` и возвращает `true`; иначе не изменяет память и возвращает `false`. В отличие от `atomicCompareExchangeN`, здесь нет параметра модели памяти (всегда используется наиболее строгая, `ATOMIC_SEQ_CST`, либо соответствующий ей `_Interlocked*`/`__sync_bool_compare_and_swap` интринсик) и работает только с `bool`, `int`, `ptr` — минимальный набор, достаточный для большинства безблокировочных структур данных.

- **Разбор реализации.** У процедуры есть несколько взаимоисключающих реализаций, выбираемых на этапе компиляции директивой `when`: на MSVC — через `_InterlockedCompareExchange{8,32,64}` в зависимости от `sizeof(T)`; на tcc — через встроенную ассемблерную вставку с инструкцией `cmpxchg`/`lock`; при наличии `atomicCompareExchangeN` — делегирует в неё с `weak = false`; иначе — прямой `importc` на `__sync_bool_compare_and_swap` (более старый, но широко переносимый GCC-интринсик). Итог для пользователя один и тот же контракт, независимо от платформы.

- `p: ptr T` — сравниваемая и потенциально изменяемая ячейка (изменяется при успехе).
- `oldValue: T` — ожидаемое текущее значение, не изменяется.
- `newValue: T` — значение для записи при совпадении, не изменяется.

**Пример:**

```nim
var counter: int = 0
if cas(addr(counter), 0, 1):
  echo "захватили счётчик, теперь: ", counter # выводит: захватили счётчик, теперь: 1
if not cas(addr(counter), 0, 2):
  echo "повторный захват не удался, значение уже: ", counter
  # выводит: повторный захват не удался, значение уже: 1
```

---

## IV. Арифметические и битовые атомарные операции

### 1. `atomic*Fetch` (вернуть новое значение)

```nim
proc atomicAddFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicSubFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicOrFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicAndFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicXorFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicNandFetch*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**Что делает.** Семейство из шести процедур "прочитать-изменить-записать" (read-modify-write): атомарно применяет операцию (`+`, `-`, `or`, `and`, `xor`, `nand` — побитовое И с последующим отрицанием) к значению по адресу `p` и параметру `val`, сохраняет результат по `p` и **возвращает именно новое** (уже изменённое) значение. Все шесть моделей памяти допустимы для всех шести операций.

- **Разбор реализации.** Каждая процедура — тонкая обвязка над одноимённым интринсиком GCC (`__atomic_add_fetch` и т.д.), который на большинстве архитектур компилируется в одну атомарную RMW-инструкцию процессора (например, `lock xadd` на x86 для сложения) без явного цикла CAS — то есть без риска "проиграть гонку" и повторять попытку.

- `p: ptr T` — изменяемая ячейка (изменяется).
- `val: T` — второй операнд операции, не изменяется.
- `mem: AtomMemModel` — модель памяти операции.

**Пример:**

```nim
var flags: int = 0b0110
let newFlags = atomicOrFetch(addr(flags), 0b1000, ATOMIC_SEQ_CST)
echo newFlags # выводит 14 (0b1110) — новое значение после OR
```

---

### 2. `atomicFetch*` (вернуть старое значение)

```nim
proc atomicFetchAdd*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicFetchSub*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicFetchOr*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicFetchAnd*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicFetchXor*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
proc atomicFetchNand*[T: AtomType](p: ptr T, val: T, mem: AtomMemModel): T
```

**Что делает.** Зеркальное семейство к предыдущему: те же шесть операций (`+`, `-`, `or`, `and`, `xor`, `nand`), но возвращается значение, которое было по адресу `p` **до** применения операции. Сама память при этом изменяется точно так же, как и в `atomic*Fetch`-семействе — разница исключительно в том, что именно возвращается.

- **Разбор реализации.** Это различие важно на практике: например, атомарный счётчик задач, где нужно узнать, "был ли я последним" (значение до декремента было 1), требует именно `atomicFetchSub`, тогда как получить итоговое количество оставшихся задач после декремента — задача для `atomicSubFetch`.

- `p: ptr T` — изменяемая ячейка (изменяется).
- `val: T` — второй операнд операции, не изменяется.
- `mem: AtomMemModel` — модель памяти операции.

**Пример:**

```nim
var counter: int = 5
let oldValue = atomicFetchAdd(addr(counter), 3, ATOMIC_SEQ_CST)
echo oldValue, " ", counter # выводит 5 8 — старое значение и новое состояние ячейки
```

---

### 3. `atomicInc`, `atomicDec`

```nim
proc atomicInc*(memLoc: var int, x: int = 1): int {.discardable.}
proc atomicDec*(memLoc: var int, x: int = 1): int {.discardable.}
```

**Что делает.** Платформонезависимые атомарные инкремент/декремент целого числа на величину `x` (по умолчанию 1), возвращающие новое значение после изменения. В однопоточной сборке (`hasThreadSupport = false`) вырождаются в обычный неатомарный `inc`/`dec` — атомарность не нужна там, где потоков нет.

- **Разбор реализации.** Выбор реализации происходит на этапе компиляции через `when`: на GCC/Clang с поддержкой потоков используется `atomicAddFetch`/`atomicSubFetch` с `ATOMIC_SEQ_CST` (для декремента, если `atomicSubFetch` недоступен на данной платформе, эмулируется через `atomicAddFetch` с отрицательным `x`); на MSVC — через `addAndFetch` (обёртку над `_InterlockedExchangeAdd`) с ручной поправкой результата, поскольку `_InterlockedExchangeAdd` в оригинале возвращает *старое* значение, а не новое; иначе (нет поддержки потоков) — простой `inc`/`dec`. Помечены `{.discardable.}`, так что возвращаемое значение необязательно использовать, если важен лишь факт изменения.

- `memLoc: var int` — изменяемая переменная (изменяется).
- `x: int` — величина изменения, по умолчанию `1`, не изменяется.

**Пример:**

```nim
var refCount: int = 1
let afterInc = atomicInc(refCount)
echo afterInc # выводит 2

let afterDec = atomicDec(refCount)
echo afterDec # выводит 1
```

---

### 4. `addAndFetch`

```nim
proc addAndFetch*(p: ptr int, val: int): int
```

**Что делает.** Внутренняя платформенная процедура (не имеет отношения к типовому классу `AtomType` — работает конкретно с `int`), используемая как строительный блок для `atomicInc`/`atomicDec` на компиляторах MSVC и в однопоточной сборке без поддержки атомарных интринсиков. Атомарно прибавляет `val` к значению по адресу `p` и возвращает новое значение.

- **Разбор реализации.** На MSVC реализована через `_InterlockedExchangeAdd`/`_InterlockedExchangeAdd64` (которые в оригинале C-API возвращают *старое* значение — отсюда ручная коррекция `inc(result, x)`/`dec(result, x)` в вызывающем коде `atomicInc`/`atomicDec`). В ветке "нет ни GCC, ни VCC" (например, сборка без поддержки потоков вовсе) — тривиальный неатомарный `inc(p[], val); result = p[]`, поскольку в таком окружении конкуренции потоков не бывает.

- `p: ptr int` — изменяемая ячейка (изменяется).
- `val: int` — величина прибавления, не изменяется.

**Пример:**

```nim
var total: int = 100
let newTotal = addAndFetch(addr(total), 50)
echo newTotal # выводит 150
```

---

## V. Флаги, барьеры и служебные процедуры

### 1. `atomicTestAndSet`, `atomicClear`

```nim
proc atomicTestAndSet*(p: pointer, mem: AtomMemModel): bool
proc atomicClear*(p: pointer, mem: AtomMemModel)
```

**Что делает.** Работают с однобайтовым "атомарным флагом" по адресу `p`: `atomicTestAndSet` атомарно устанавливает байт в некоторое реализационно-зависимое ненулевое "установленное" значение и возвращает `true`, если байт уже был установлен *до* вызова (то есть флаг был занят); `atomicClear` атомарно сбрасывает байт в `0`. Пара используется как минимальный примитив блокировки — "занято/свободно" — без явного типа `AtomType`, поскольку работает на уровне сырого байта (`pointer`), а не типизированного значения.

- **Разбор реализации.** Это прямая обвязка над `__atomic_test_and_set`/`__atomic_clear` — теми же интринсиками, на которых в C++ построен `std::atomic_flag`. Гарантируется атомарность только самого байта-флага, а не произвольной структуры.

- `p: pointer` — адрес байта-флага (изменяется).
- `mem: AtomMemModel` — модель памяти операции (`atomicClear` допускает только `ATOMIC_RELAXED`, `ATOMIC_SEQ_CST`, `ATOMIC_RELEASE`).

**Пример:**

```nim
var flagByte: byte = 0
let wasSet = atomicTestAndSet(addr(flagByte), ATOMIC_ACQUIRE)
echo wasSet # выводит false — флаг был свободен, теперь установлен
atomicClear(addr(flagByte), ATOMIC_RELEASE)
```

---

### 2. `atomicThreadFence`, `atomicSignalFence`, `fence`

```nim
proc atomicThreadFence*(mem: AtomMemModel)
proc atomicSignalFence*(mem: AtomMemModel)
template fence*()
```

**Что делает.** Все три — барьеры памяти без привязки к конкретной ячейке. `atomicThreadFence` синхронизирует видимость памяти между потоками согласно `mem`, не выполняя при этом никакой атомарной операции чтения/записи сама по себе — это "чистый" барьер. `atomicSignalFence` — более узкий барьер: синхронизация не с другими потоками, а с обработчиками сигналов в том же потоке (важно только в контексте обработки сигналов ОС). `fence` — template-сокращение для наиболее строгого варианта: `atomicThreadFence(ATOMIC_SEQ_CST)`.

- **Разбор реализации.** Барьер без адреса — это способ сказать компилятору и процессору "не переупорядочивай инструкции вокруг этой точки", даже если ни одна конкретная переменная явно не читается и не пишется. Полезно, когда несколько атомарных операций уже выполнены с более слабой моделью (`ATOMIC_RELAXED`) ради скорости, а барьер нужен один раз в конце — как единая "точка синхронизации" вместо усиления модели у каждой операции по отдельности.

- `mem: AtomMemModel` — требуемая модель памяти барьера; все шесть моделей допустимы.

**Пример:**

```nim
var payload: int = 0
var published: bool = false

payload = 42                                    # готовим данные (обычная запись)
atomicThreadFence(ATOMIC_RELEASE)               # барьер: всё выше не "утечёт" ниже
atomicStoreN(addr(published), true, ATOMIC_RELAXED)
echo payload, " ", published # выводит 42 true
```

---

### 3. `cpuRelax`

```nim
proc cpuRelax*()
```

**Что делает.** Подсказка процессору внутри цикла активного ожидания (busy-wait / спин-цикла): "здесь можно ненадолго уступить конвейер", что снижает энергопотребление и позволяет соседнему логическому ядру (SMT/Hyper-Threading) работать эффективнее, не влияя на корректность программы.

- **Разбор реализации.** На x86/amd64 компилируется в инструкцию `pause` через ассемблерную вставку (либо в интринсик MSVC `YieldProcessor`) — сама по себе она не освобождает ядро и не переключает поток, а лишь снижает потребление энергии на время короткой паузы и уменьшает штраф при выходе из спекулятивного цикла. На платформах без специальной инструкции сводится к барьеру компилятора (`asm volatile ("" ::: "memory")`), не позволяющему компилятору "вырезать" пустой цикл ожидания при оптимизации.

- **Параметры:** нет.

**Пример:**

```nim
var lockFlag: int = 1 # представим, что блокировка уже кем-то захвачена
var attempts = 0
while lockFlag != 0 and attempts < 3:
  cpuRelax()   # уступаем процессор на время ожидания
  inc(attempts)
echo attempts # выводит 3 — цикл ожидания отработал заданное число раз
```

---

### 4. `atomicAlwaysLockFree`, `atomicIsLockFree`

```nim
proc atomicAlwaysLockFree*(size: int, p: pointer): bool
proc atomicIsLockFree*(size: int, p: pointer): bool
```

**Что делает.** Диагностические процедуры, отвечающие на вопрос "будут ли атомарные операции над объектом размером `size` байт реализованы без использования блокировок (мьютекса) на целевой архитектуре". `atomicAlwaysLockFree` разрешается **на этапе компиляции** (и `size` должно быть compile-time константой) — используется, чтобы статически выбрать более быстрый путь кода. `atomicIsLockFree` может при необходимости выполнить рантайм-проверку (вызвать `__atomic_is_lock_free`), если ответ не известен статически — например, когда лок-фридность зависит от фактического выравнивания указателя `p`, известного только в момент выполнения.

- **Разбор реализации.** Большинство архитектур гарантируют лок-фри операции только для размеров, не превышающих машинное слово (обычно 8 байт на 64-битных системах), и то при условии правильного выравнивания. Параметр `p` необязателен по смыслу (может быть `nil`) и служит подсказкой компилятору о выравнивании — если оно неизвестно, предполагается наихудший случай.

- `size: int` — размер объекта в байтах, должен быть известен на этапе компиляции для `atomicAlwaysLockFree`.
- `p: pointer` — необязательный указатель на объект (для оценки выравнивания), может быть `nil`.

**Пример:**

```nim
when atomicAlwaysLockFree(sizeof(int), nil):
  echo "атомарные операции над int лок-фри на этой платформе"
else:
  echo "атомарные операции над int могут использовать блокировку"
```

---

## VI. Практические рецепты

### 1. Атомарный счётчик ссылок

```nim
type RefCounted = object
  count: int

proc retain(r: var RefCounted) =
  discard atomicInc(r.count)

proc release(r: var RefCounted): bool =
  # ЗАМЕЧАНИЕ: atomicFetchSub возвращает значение ДО декремента —
  # если оно было равно 1, значит именно этот вызов обнулил счётчик
  result = atomicFetchSub(addr(r.count), 1, ATOMIC_ACQ_REL) == 1

var obj = RefCounted(count: 1)
retain(obj)                 # count: 2
echo release(obj)           # выводит false — ссылок ещё осталось
echo release(obj)           # выводит true  — последняя ссылка снята, можно освобождать ресурс
```

---

### 2. Однократная ленивая инициализация (double-checked locking)

```nim
var
  initialized: bool = false
  sharedValue: int = 0

proc ensureInitialized() =
  if atomicLoadN(addr(initialized), ATOMIC_ACQUIRE):
    return # уже инициализировано — быстрый путь без блокировки
  # медленный путь: только один поток реально выполнит инициализацию,
  # т.к. cas атомарно "застолбит" переход false -> true
  if cas(addr(initialized), false, true):
    sharedValue = 12345                     # тяжёлая инициализация
    atomicThreadFence(ATOMIC_RELEASE)       # публикуем sharedValue для других потоков

ensureInitialized()
echo sharedValue # выводит 12345
```

---

### 3. Спин-блокировка на `atomicTestAndSet`

```nim
type SpinLock = object
  flagByte: byte

proc acquire(lock: var SpinLock) =
  while atomicTestAndSet(addr(lock.flagByte), ATOMIC_ACQUIRE):
    cpuRelax() # ждём, не нагружая шину памяти лишними попытками записи

proc release(lock: var SpinLock) =
  atomicClear(addr(lock.flagByte), ATOMIC_RELEASE)

var lock: SpinLock
acquire(lock)
echo "критическая секция"
release(lock)
```

---

### 4. Безблокировочный стек (Treiber stack) на `cas`

```nim
type
  Node = ref object
    value: int
    next: Node
  LockFreeStack = object
    head: Node

proc push(s: var LockFreeStack, value: int) =
  var newNode = Node(value: value)
  var oldHead = s.head
  newNode.next = oldHead
  # повторяем, пока голова стека не совпадёт с ожидаемой на момент попытки:
  # если между чтением oldHead и cas другой поток успел изменить голову,
  # cas провалится и newNode.next будет обновлён на актуальную голову
  while not cas(addr(s.head), oldHead, newNode):
    oldHead = s.head
    newNode.next = oldHead

var stack: LockFreeStack
push(stack, 1)
push(stack, 2)
echo stack.head.value # выводит 2 — последний протолкнутый элемент наверху
```

---

## VII. Краткая таблица

| Задача | Процедура | Возвращает |
|---|---|---|
| Атомарно прочитать значение | `atomicLoadN` / `atomicLoad` | прочитанное значение |
| Атомарно записать значение | `atomicStoreN` / `atomicStore` | ничего |
| Записать и узнать, что было раньше | `atomicExchangeN` / `atomicExchange` | старое значение |
| Сравнить с ожидаемым и заменить (может провалиться) | `atomicCompareExchangeN` / `atomicCompareExchange` | `bool` успеха |
| Простой кросс-платформенный CAS для `bool`/`int`/`ptr` | `cas` | `bool` успеха |
| Арифметика/битовая операция, нужен результат ПОСЛЕ | `atomicAddFetch`, `atomicSubFetch`, `atomicOrFetch`, `atomicAndFetch`, `atomicXorFetch`, `atomicNandFetch` | новое значение |
| Арифметика/битовая операция, нужен результат ДО | `atomicFetchAdd`, `atomicFetchSub`, `atomicFetchOr`, `atomicFetchAnd`, `atomicFetchXor`, `atomicFetchNand` | старое значение |
| Атомарно ±1 (или на `x`) для `int`, кросс-платформенно | `atomicInc` / `atomicDec` | новое значение |
| Однобайтовый флаг занятости (спин-блок) | `atomicTestAndSet` / `atomicClear` | `bool` (был ли занят) / ничего |
| Барьер памяти без конкретной ячейки | `atomicThreadFence` / `fence` | ничего |
| Барьер только относительно обработчиков сигналов | `atomicSignalFence` | ничего |
| Уступить процессор в цикле активного ожидания | `cpuRelax` | ничего |
| Узнать, будут ли операции лок-фри | `atomicAlwaysLockFree` / `atomicIsLockFree` | `bool` |

---

## VIII. Сводка: какую процедуру выбрать

- Нужно просто безопасно прочитать/записать переменную из нескольких потоков → `atomicLoadN` / `atomicStoreN`.
- Нужно узнать предыдущее значение при перезаписи (например, реализовать флаг "первый вызов") → `atomicExchangeN`.
- Нужна условная замена "если там всё ещё то, что я ожидаю" → `cas` (для `bool`/`int`/`ptr`, без явной модели памяти) или `atomicCompareExchangeN` (если нужен контроль модели памяти и `weak`).
- Нужен CAS внутри цикла повторных попыток → `atomicCompareExchangeN` с `weak = true`; вне цикла, разово → `weak = false`.
- Нужно атомарно прибавить/вычесть число и сразу узнать новый итог → `atomicAddFetch`/`atomicSubFetch` (или платформонезависимые `atomicInc`/`atomicDec` для `int`).
- Нужно узнать значение, которое было до изменения (например, "был ли я последним, кто уменьшил счётчик до нуля") → `atomicFetchSub`.
- Нужна простая блокировка/флаг занятости без полноценного мьютекса → `atomicTestAndSet` + `atomicClear`, в паре с `cpuRelax` в цикле ожидания.
- Нужно расставить "точки синхронизации" без привязки к конкретной переменной → `atomicThreadFence` или сокращённый `fence()`.
- Нужно оптимизировать спин-цикл ожидания, не меняя его логику → добавить `cpuRelax()` внутрь цикла.
- Нужно на этапе компиляции выбрать более быстрый путь кода в зависимости от лок-фридности платформы → `atomicAlwaysLockFree`.
