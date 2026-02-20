# Справочник модуля `atomics` (Nim)

> Модуль `atomics` предоставляет **атомарные типы и операции** для реализации алгоритмов без блокировок (lock-free).  
> По умолчанию использует примитивы C11; при компиляции с `-d:nimUseCppAtomics` — `std::atomic` из C++11.

> ⚠️ **API нестабилен** и может измениться в будущих версиях Nim.

---

## Содержание

- [Типы данных](#типы-данных)
- [Порядок памяти (MemoryOrder)](#порядок-памяти-memoryorder)
- [Atomic — операции доступа](#atomic--операции-доступа)
- [Atomic — операции сравнения и обмена](#atomic--операции-сравнения-и-обмена)
- [Atomic — числовые операции](#atomic--числовые-операции)
- [Atomic — операторы и удобные обёртки](#atomic--операторы-и-удобные-обёртки)
- [AtomicFlag — операции](#atomicflag--операции)
- [Барьеры памяти](#барьеры-памяти)
- [Руководство по выбору порядка памяти](#руководство-по-выбору-порядка-памяти)
- [Краткая шпаргалка](#краткая-шпаргалка)

---

## Типы данных

### `Atomic[T]`

Атомарный объект с базовым типом `T`. Гарантирует, что операции чтения и записи не будут разорваны другими потоками.

Для тривиальных типов (`SomeNumber`, `bool`, `enum`, `ptr`, `pointer`) размером до 8 байт используются аппаратные атомарные инструкции. Для остальных типов применяется спин-блокировка через `AtomicFlag`.

```nim
var counter: Atomic[int]
var flag: Atomic[bool]
var p: Atomic[ptr MyObj]
```

---

### `AtomicFlag`

Атомарный булев флаг. Единственный тип, гарантированно реализованный без блокировок на всех платформах согласно стандарту C11. Изначально находится в сброшенном состоянии (`false`).

```nim
var flag: AtomicFlag
```

---

## Порядок памяти (MemoryOrder)

`MemoryOrder` задаёт **ограничения на переупорядочивание** операций над памятью вокруг атомарной операции. Чем строже порядок, тем больше гарантий синхронизации и тем выше потенциальные накладные расходы.

По умолчанию во всех операциях используется `moSequentiallyConsistent` — самый строгий и безопасный вариант.

### `moRelaxed`

Никаких ограничений на переупорядочивание. Гарантируется только атомарность самой операции и упорядочивание относительно других атомарных операций на **той же переменной** в **том же потоке**.

Применяется: счётчики статистики, где точный порядок несущественен.

```nim
var c: Atomic[int]
c.store(1, moRelaxed)
let v = c.load(moRelaxed)
```

---

### `moConsume`

Ослабленный вариант `moAcquire` с зависимостью по данным. **Не рекомендуется** к использованию — семантика пересматривается в стандарте. Предпочитайте `moAcquire`.

---

### `moAcquire`

Применяется к операциям **загрузки**. Ни одно чтение или запись в текущем потоке не может быть переставлено **перед** этой операцией. Образует «acquire-барьер».

Типичная пара: `load(moAcquire)` в потребителе + `store(moRelease)` в производителе.

```nim
# Потребитель:
let ready = flag.load(moAcquire)
if ready:
  let data = sharedData.load(moRelaxed)  # видно после acquire
```

---

### `moRelease`

Применяется к операциям **записи**. Ни одно чтение или запись в текущем потоке не может быть переставлено **после** этой операции. Образует «release-барьер».

```nim
# Производитель:
sharedData.store(42, moRelaxed)
flag.store(true, moRelease)  # гарантирует видимость sharedData
```

---

### `moAcquireRelease`

Для операций **чтение-модификация-запись** (RMW): одновременно действует как `moAcquire` и `moRelease`. Используется в `exchange`, `compareExchange`, `fetchAdd` и т.п.

```nim
let old = counter.exchange(0, moAcquireRelease)
```

---

### `moSequentiallyConsistent`

Самый строгий порядок. Ведёт себя как `moAcquire` для загрузок, как `moRelease` для записей и как `moAcquireRelease` для RMW-операций. Дополнительно гарантирует, что **все потоки** наблюдают одинаковый общий порядок всех `moSequentiallyConsistent`-операций.

**Значение по умолчанию** для всех операций в модуле.

```nim
var loc: Atomic[int]
loc.store(42)            # moSequentiallyConsistent
let v = loc.load()       # moSequentiallyConsistent
```

---

## Atomic — операции доступа

### `load[T](location: var Atomic[T]; order: MemoryOrder = moSequentiallyConsistent): T`

Атомарно считывает и возвращает текущее значение атомарного объекта.

```nim
var loc: Atomic[int]
loc.store(42)

let v1 = loc.load()                  # moSequentiallyConsistent
let v2 = loc.load(moRelaxed)         # без барьера
let v3 = loc.load(moAcquire)         # с acquire-барьером
assert v1 == 42
```

---

### `store[T](location: var Atomic[T]; desired: T; order: MemoryOrder = moSequentiallyConsistent)`

Атомарно записывает значение `desired` в атомарный объект.

> Допустимые порядки: `moRelaxed`, `moRelease`, `moSequentiallyConsistent`.

```nim
var loc: Atomic[int]
loc.store(10)
loc.store(20, moRelaxed)
loc.store(30, moRelease)
assert loc.load() == 30
```

---

### `exchange[T](location: var Atomic[T]; desired: T; order: MemoryOrder = moSequentiallyConsistent): T`

Атомарно заменяет значение на `desired` и **возвращает старое** значение. Операция «прочитать-и-записать» (RMW).

```nim
var loc: Atomic[int]
loc.store(7)

let old = loc.exchange(99)
assert old == 7
assert loc.load() == 99
```

---

## Atomic — операции сравнения и обмена

### `compareExchange[T](location, expected, desired, order): bool`  
### `compareExchange[T](location, expected, desired, success, failure): bool`

**CAS (Compare-And-Swap)** — атомарно сравнивает значение атомарного объекта с `expected`:
- Если **равны**: записывает `desired` и возвращает `true`. `expected` остаётся равным переданному значению.
- Если **не равны**: загружает текущее значение в `expected` и возвращает `false`. Значение объекта не меняется.

Двухпараметрическая версия позволяет задать разные порядки памяти для случаев успеха (`success`) и неудачи (`failure`).

```nim
var loc: Atomic[int]
loc.store(7)

var expected = 7
let ok = loc.compareExchange(expected, 100, moRelaxed, moRelaxed)
assert ok == true
assert loc.load() == 100
assert expected == 7     # expected не изменился при успехе

# Неудача:
var expected2 = 0        # неверное ожидаемое значение
let ok2 = loc.compareExchange(expected2, 200, moRelaxed, moRelaxed)
assert ok2 == false
assert expected2 == 100  # expected2 обновлён реальным значением
assert loc.load() == 100 # объект не изменился

# Паттерн lock-free обновления:
var counter: Atomic[int]
counter.store(0)
var current = counter.load(moRelaxed)
while not counter.compareExchange(current, current + 1, moRelaxed, moRelaxed):
  discard  # current автоматически обновлён, повторяем
```

---

### `compareExchangeWeak[T](location, expected, desired, order): bool`  
### `compareExchangeWeak[T](location, expected, desired, success, failure): bool`

Слабая версия CAS. Идентична `compareExchange`, но допускает **ложные неудачи** (spurious failure): может вернуть `false` даже если значения равны. На некоторых архитектурах (ARM) компилируется в более эффективный код.

Используйте в циклах повторных попыток, где ложная неудача только приведёт к лишней итерации:

```nim
var loc: Atomic[int]
loc.store(5)
var expected = loc.load(moRelaxed)
# Слабая версия в цикле:
while not loc.compareExchangeWeak(expected, expected * 2, moRelaxed, moRelaxed):
  discard
assert loc.load() == 10
```

> **`compareExchange` vs `compareExchangeWeak`**: используйте `compareExchange` если цикл повторной попытки не нужен (однократная проверка). Используйте `compareExchangeWeak` внутри циклов — это может быть быстрее на архитектурах с LL/SC (ARM, RISC-V).

---

## Atomic — числовые операции

Все численные операции доступны только для типов `SomeInteger`. Каждая атомарно выполняет действие и возвращает **значение до изменения**.

### `fetchAdd[T: SomeInteger](location, value, order): T`

Атомарно прибавляет `value` и возвращает **старое** значение.

```nim
var c: Atomic[int]
c.store(5)
assert c.fetchAdd(3) == 5   # было 5, стало 8
assert c.load() == 8
assert c.fetchAdd(2) == 8   # было 8, стало 10
```

---

### `fetchSub[T: SomeInteger](location, value, order): T`

Атомарно вычитает `value` и возвращает **старое** значение.

```nim
var c: Atomic[int]
c.store(10)
assert c.fetchSub(3) == 10  # было 10, стало 7
assert c.load() == 7
```

---

### `fetchAnd[T: SomeInteger](location, value, order): T`

Атомарно применяет побитовое AND с `value` и возвращает **старое** значение.

```nim
var flags: Atomic[int]
flags.store(0b1111)
let old = flags.fetchAnd(0b1010)
assert old == 0b1111        # старое значение
assert flags.load() == 0b1010
```

---

### `fetchOr[T: SomeInteger](location, value, order): T`

Атомарно применяет побитовое OR с `value` и возвращает **старое** значение.

```nim
var flags: Atomic[int]
flags.store(0b0101)
let old = flags.fetchOr(0b1010)
assert old == 0b0101
assert flags.load() == 0b1111
```

---

### `fetchXor[T: SomeInteger](location, value, order): T`

Атомарно применяет побитовое XOR с `value` и возвращает **старое** значение.

```nim
var flags: Atomic[int]
flags.store(0b1100)
let old = flags.fetchXor(0b1010)
assert old == 0b1100
assert flags.load() == 0b0110
```

---

## Atomic — операторы и удобные обёртки

### `atomicInc[T: SomeInteger](location: var Atomic[T]; value: T = 1)`

Атомарно увеличивает значение на `value` (по умолчанию на 1). Обёртка над `fetchAdd`, результат отбрасывается.

```nim
var c: Atomic[int]
c.store(0)
c.atomicInc()      # +1
c.atomicInc(5)     # +5
assert c.load() == 6
```

---

### `atomicDec[T: SomeInteger](location: var Atomic[T]; value: T = 1)`

Атомарно уменьшает значение на `value` (по умолчанию на 1). Обёртка над `fetchSub`.

```nim
var c: Atomic[int]
c.store(10)
c.atomicDec()      # -1 → 9
c.atomicDec(4)     # -4 → 5
assert c.load() == 5
```

---

### `+=[T: SomeInteger](location: var Atomic[T]; value: T)`

Оператор атомарного сложения с присваиванием. Эквивалент `atomicInc`.

```nim
var c: Atomic[int]
c.store(0)
c += 3
c += 7
assert c.load() == 10
```

---

### `-=[T: SomeInteger](location: var Atomic[T]; value: T)`

Оператор атомарного вычитания с присваиванием. Эквивалент `atomicDec`.

```nim
var c: Atomic[int]
c.store(10)
c -= 3
assert c.load() == 7
```

---

## AtomicFlag — операции

### `testAndSet(location: var AtomicFlag; order: MemoryOrder = moSequentiallyConsistent): bool`

Атомарно устанавливает флаг в `true` и возвращает **предыдущее** значение.

- Возвращает `false` — флаг **не был** установлен, теперь установлен.
- Возвращает `true` — флаг **уже был** установлен.

Используется для реализации спин-блокировок:

```nim
var flag: AtomicFlag

# Первый вызов: флаг не установлен → устанавливаем, возвращаем false
assert not flag.testAndSet()

# Второй вызов: флаг уже установлен → возвращаем true
assert flag.testAndSet()

# Спин-блокировка:
var lock: AtomicFlag
while lock.testAndSet(moAcquire): discard   # ждём захвата
# --- критическая секция ---
lock.clear(moRelease)                        # освобождаем
```

---

### `clear(location: var AtomicFlag; order: MemoryOrder = moSequentiallyConsistent)`

Атомарно сбрасывает флаг в `false`.

> Допустимые порядки: `moRelaxed`, `moRelease`, `moSequentiallyConsistent`.

```nim
var flag: AtomicFlag
discard flag.testAndSet()   # установить
assert flag.testAndSet()    # проверить: установлен
flag.clear(moRelaxed)       # сбросить
assert not flag.testAndSet() # снова не установлен
```

---

## Барьеры памяти

### `fence(order: MemoryOrder)`

Устанавливает **барьер памяти для потоков** (thread fence) без выполнения атомарной операции. Обеспечивает нужный порядок видимости операций над обычными (неатомарными) переменными.

```nim
# Производитель:
sharedData = 42              # обычная запись
fence(moRelease)             # барьер: всё выше видно до барьера

# Потребитель:
fence(moAcquire)             # барьер: всё ниже идёт после барьера
let v = sharedData           # читаем 42
```

---

### `signalFence(order: MemoryOrder)`

Барьер памяти **только для компилятора** (signal fence). Запрещает компилятору переставлять операции, но **не вставляет** инструкции CPU для упорядочивания памяти. Используется для предотвращения оптимизаций компилятора в обработчиках сигналов.

```nim
signalFence(moAcquireRelease)  # только барьер компилятора
```

---

## Руководство по выбору порядка памяти

| Сценарий | load | store | RMW |
|---|---|---|---|
| Максимальная безопасность (по умолчанию) | `moSequentiallyConsistent` | `moSequentiallyConsistent` | `moSequentiallyConsistent` |
| Производитель-потребитель | `moAcquire` | `moRelease` | — |
| Счётчики без синхронизации | `moRelaxed` | `moRelaxed` | `moRelaxed` |
| Мьютекс: захват | — | — | `moAcquire` |
| Мьютекс: освобождение | — | `moRelease` | — |

> **Правило безопасности**: начинайте с `moSequentiallyConsistent` (значение по умолчанию). Ослабляйте порядок только при наличии чёткого понимания модели памяти и подтверждённой необходимости оптимизации.

---

## Краткая шпаргалка

### Atomic[T]

| Операция | Функция | Возвращает |
|---|---|---|
| Прочитать | `load(order)` | `T` — текущее значение |
| Записать | `store(desired, order)` | — |
| Обменять | `exchange(desired, order)` | `T` — старое значение |
| CAS сильный | `compareExchange(expected, desired, ...)` | `bool` — успех |
| CAS слабый | `compareExchangeWeak(expected, desired, ...)` | `bool` — успех |
| Прибавить | `fetchAdd(value, order)` | `T` — старое значение |
| Вычесть | `fetchSub(value, order)` | `T` — старое значение |
| Побитовое AND | `fetchAnd(value, order)` | `T` — старое значение |
| Побитовое OR | `fetchOr(value, order)` | `T` — старое значение |
| Побитовое XOR | `fetchXor(value, order)` | `T` — старое значение |
| Инкремент | `atomicInc(value)` / `+= value` | — |
| Декремент | `atomicDec(value)` / `-= value` | — |

### AtomicFlag

| Операция | Функция | Возвращает |
|---|---|---|
| Установить и прочитать | `testAndSet(order)` | `bool` — значение **до** установки |
| Сбросить | `clear(order)` | — |

### Порядок MemoryOrder (от слабого к строгому)

```
moRelaxed  <  moConsume  <  moAcquire / moRelease  <  moAcquireRelease  <  moSequentiallyConsistent
```

### Примечания

- API помечен как **нестабильный**: сигнатуры могут измениться.
- По умолчанию используются примитивы C11; для C++ `std::atomic` — `-d:nimUseCppAtomics`.
- `AtomicFlag` — единственный тип, **гарантированно** реализованный без блокировок на всех платформах.
- `Atomic[T]` для нетривиальных типов (не `SomeNumber`/`bool`/`ptr`) реализован через спин-блокировку на `AtomicFlag`.
- Все `fetch*`-функции возвращают значение **до** изменения.
