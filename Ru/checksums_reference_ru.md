# Nim — контрольные суммы и хеширование паролей: справочник по модулям

> **Пакет:** `checksums` (установка: `nimble install checksums`)  
> **Зонтичный модуль:** `docutils` — одним импортом подключает `md5`, `sha1`, `sha2`, `sha3` и `bcrypt`.

Этот справочник охватывает каждый экспортируемый символ всех пяти модулей. Разделы упорядочены от самого простого (MD5) к самому защищённому (bcrypt) — так удобнее читать последовательно. Внутри каждого раздела функции расположены в порядке их типичного использования.

---

## Содержание

1. [MD5](#md5)
2. [SHA-1](#sha-1)
3. [SHA-2](#sha-2)
4. [SHA-3 и SHAKE](#sha-3-и-shake)
5. [bcrypt](#bcrypt)
6. [Как выбрать алгоритм](#как-выбрать-алгоритм)

---

## MD5

> **Модуль:** `md5`

MD5 производит 128-битный (16-байтовый) дайджест. Он быстр и широко применяется в **незащищённых** сценариях: контрольные суммы файлов для обнаружения случайного повреждения, ключи кешей, совместимость с устаревшими системами. **Не используйте MD5 там, где важна безопасность** — коллизии генерируются за секунды даже на потребительском железе.

Модуль работает во время компиляции и в JavaScript-целях, что нетипично и иногда полезно для метапрограммирования.

---

### Тип: `MD5Digest`

```nim
type MD5Digest* = array[0..15, uint8]
```

16-элементный массив байт, содержащий «сырой» результат MD5. Преобразуется в строку hex-символов оператором `$`.

---

### Тип: `MD5Context`

```nim
type MD5Context* {.final.} = object
```

Хранит промежуточное состояние при потоковом хешировании. Создаёте контекст, кормите данными частями, финализируете. Процедуры `toMD5` и `getMD5` управляют этим жизненным циклом за вас автоматически.

---

### `md5Init`

```nim
proc md5Init*(c: var MD5Context)
```

Сбрасывает (или инициализирует) `MD5Context` в начальное состояние. Вызывайте перед подачей данных при использовании низкоуровневого потокового API.

```nim
var ctx: MD5Context
md5Init(ctx)
```

> **Примечание:** Если используете `toMD5` или `getMD5`, вызывать `md5Init` не нужно.

---

### `md5Update`

```nim
proc md5Update*(c: var MD5Context, input: openArray[uint8])
proc md5Update*(c: var MD5Context, input: cstring, len: int)
```

Добавляет очередной блок данных в уже инициализированный `MD5Context`. Вызывается любое количество раз — каждый вызов продолжает вычисление с того места, где остановился предыдущий. Это ключ к хешированию больших входных данных без загрузки всего содержимого в память.

```nim
var ctx: MD5Context
md5Init(ctx)
md5Update(ctx, cast[openArray[uint8]]("Привет, "))
md5Update(ctx, cast[openArray[uint8]]("Мир!"))
# Контекст теперь отражает «Привет, Мир!» как единое целое
```

---

### `md5Final`

```nim
proc md5Final*(c: var MD5Context, digest: var MD5Digest)
```

Добавляет дополнение MD5, завершает вычисление и записывает 16-байтовый результат в `digest`. После вызова контекст находится в неопределённом состоянии и не должен использоваться без повторной инициализации.

```nim
var ctx: MD5Context
var d: MD5Digest
md5Init(ctx)
md5Update(ctx, cast[openArray[uint8]]("abc"))
md5Final(ctx, d)
echo $d  # "900150983cd24fb0d6963f7d28e17f72"
```

---

### `toMD5`

```nim
proc toMD5*(s: string): MD5Digest
```

Удобная обёртка в один вызов: инициализирует контекст, подаёт в него `s`, финализирует и возвращает «сырой» `MD5Digest`. Используйте, когда вся строка доступна целиком.

```nim
let digest = toMD5("abc")
assert $digest == "900150983cd24fb0d6963f7d28e17f72"
```

---

### `getMD5`

```nim
proc getMD5*(s: string): string
```

То же самое, что `toMD5`, но сразу конвертирует результат в строку из строчных hex-символов. Наиболее эргономичный вариант для типичного сценария «дай мне hex-контрольную сумму».

```nim
assert getMD5("abc") == "900150983cd24fb0d6963f7d28e17f72"
assert getMD5("") == "d41d8cd98f00b204e9800998ecf8427e"  # хеш пустой строки хорошо известен
```

---

### `$` (MD5Digest → string)

```nim
proc `$`*(d: MD5Digest): string
```

Преобразует «сырой» массив `MD5Digest` в строковое hex-представление в нижнем регистре (32 символа).

```nim
let d = toMD5("nim")
echo $d  # строчный hex
```

---

### `==` (MD5Digest)

```nim
proc `==`*(D1, D2: MD5Digest): bool
```

Побайтово сравнивает два значения `MD5Digest`. Возвращает `true`, если все байты совпадают. Обратите внимание: это **не** сравнение за постоянное время. Не используйте для сравнения секретов с пользовательским вводом в коде, где важна безопасность.

```nim
assert toMD5("abc") == toMD5("abc")
assert toMD5("abc") != toMD5("xyz")
```

---

## SHA-1

> **Модуль:** `sha1`

SHA-1 производит 160-битный (20-байтовый) дайджест. Он **официально устарел с 2011 года**, так как существуют практические атаки на коллизии. Тем не менее SHA-1 широко применяется для идентификаторов коммитов git, устаревших контрольных сумм и протокольной совместимости. Сам модуль не устарел — он останется доступным.

---

### Тип: `Sha1Digest`

```nim
type Sha1Digest* = array[0 .. 19, uint8]
```

«Сырые» 20 байт SHA-1.

---

### Тип: `SecureHash`

```nim
type SecureHash* = distinct Sha1Digest
```

Псевдоним над `Sha1Digest`, дающий ему отдельную идентичность в системе типов. Операторы `$` и `==` определены именно для `SecureHash`.

---

### Тип: `Sha1State`

```nim
type Sha1State* = object
```

Изменяемый объект, хранящий состояние потокового вычисления SHA-1. Нужен для низкоуровневого API с `update`/`finalize`.

---

### `newSha1State`

```nim
proc newSha1State*(): Sha1State
```

Создаёт и возвращает свежеинициализированный `Sha1State`. Точка входа для инкрементального (потокового) хеширования.

```nim
var state = newSha1State()
```

> `secureHash` вызывает это внутри — вам нужно только при потоковом использовании.

---

### `update` (SHA-1)

```nim
proc update*(ctx: var Sha1State, data: openArray[char])
```

Добавляет очередной блок символов в текущее вычисление SHA-1. Вызывается многократно для потоковой обработки. Порядок вызовов имеет значение — каждый вызов дописывает к накапливаемому сообщению.

```nim
var state = newSha1State()
state.update("Привет, ")
state.update("Мир!")
# Эквивалентно хешированию "Привет, Мир!" за один раз
```

---

### `finalize` (SHA-1)

```nim
proc finalize*(ctx: var Sha1State): Sha1Digest
```

Завершает дополнение и сжатие SHA-1, возвращает «сырой» 20-байтовый `Sha1Digest`. После финализации состояние следует выбросить.

```nim
var state = newSha1State()
state.update("Hello World")
let raw: Sha1Digest = state.finalize()
```

---

### `secureHash`

```nim
proc secureHash*(str: openArray[char]): SecureHash
```

Однострочник: хеширует `str` за один вызов, возвращает `SecureHash`. Наиболее распространённая точка входа.

```nim
let h = secureHash("Hello World")
assert $h == "0A4D55A8D778E5022FAB701977C5D840BBC486D0"
```

---

### `secureHashFile`

```nim
proc secureHashFile*(filename: string): SecureHash
```

Открывает файл, читает его блоками по 8 КБ (потоково) и возвращает SHA-1 хеш файла. Экономит память при работе с большими файлами.

```nim
let fileHash = secureHashFile("data.bin")
```

---

### `parseSecureHash`

```nim
proc parseSecureHash*(hash: string): SecureHash
```

Разбирает 40-символьную строку hex в верхнем регистре обратно в значение `SecureHash`. Полезно, когда хеш хранится в базе данных или конфиг-файле и нужно сравнить его со свежевычисленным.

```nim
let stored = parseSecureHash("0A4D55A8D778E5022FAB701977C5D840BBC486D0")
assert secureHash("Hello World") == stored
```

---

### `$` (SecureHash → string)

```nim
proc `$`*(self: SecureHash): string
```

Возвращает 40-символьную hex-строку в верхнем регистре.

```nim
echo $secureHash("John Doe")  # "AE6E4D1209F17B460503904FAD297B31E9CF6362"
```

---

### `==` (SecureHash)

```nim
proc `==`*(a, b: SecureHash): bool
```

Сравнивает два значения `SecureHash`. Как и аналог для MD5, это **не** сравнение за постоянное время.

---

### `isValidSha1Hash`

```nim
proc isValidSha1Hash*(s: string): bool
```

Возвращает `true`, если `s` ровно 40 символов в длину и состоит исключительно из hex-цифр (`0–9`, `a–f`, `A–F`). Быстрая проверка формата — она **не** верифицирует, что строка соответствует каким-либо реальным данным.

```nim
assert isValidSha1Hash("0A4D55A8D778E5022FAB701977C5D840BBC486D0") == true
assert isValidSha1Hash("не-хеш") == false
assert isValidSha1Hash("0A4D") == false  # слишком коротко
```

---

## SHA-2

> **Модуль:** `sha2`

SHA-2 — действующий стандартный семейство алгоритмов для общего криптографического хеширования. Модуль реализует четыре варианта:

| Вариант | Вывод | Внутреннее слово | Типичное применение |
|---------|-------|------------------|---------------------|
| SHA-224 | 28 байт | 32-бит | Когда нужна безопасность SHA-256, но вывод должен быть компактнее |
| SHA-256 | 32 байт | 32-бит | Общего назначения; наиболее распространён |
| SHA-384 | 48 байт | 64-бит | Усечённый SHA-512 для повышенной безопасности без полного 512-битного вывода |
| SHA-512 | 64 байт | 64-бит | Максимальная безопасность; медленнее на 32-битных платформах |

Модуль предлагает два уровня API. **Статический** уровень (`ShaStateStatic`) предпочтителен — компилятор гарантирует правильный размер буфера и возвращает типизированный массив нужного размера. **Нетипизированный** уровень (`ShaState`) жертвует безопасностью в обмен на гибкость (выбор алгоритма в рантайме, внешний буфер).

---

### Тип: `ShaInstance`

```nim
type ShaInstance* = enum
  Sha_224, Sha_256, Sha_384, Sha_512
```

Перечисление для выбора варианта SHA-2. Передаётся в `initSha` или `secureHash`, когда алгоритм определяется динамически.

---

### Типы: `ShaDigest_224/256/384/512`

```nim
type
  ShaDigest_224* = array[28, char]
  ShaDigest_256* = array[32, char]
  ShaDigest_384* = array[48, char]
  ShaDigest_512* = array[64, char]
```

Массивы фиксированного размера для «сырого» результата каждого варианта. Поскольку размер закодирован в типе, компилятор ловит несоответствия на этапе компиляции.

---

### Тип: `ShaStateStatic[instance]`

```nim
type ShaStateStatic*[instance: static ShaInstance] = distinct ShaState
```

Статически типизированное состояние вычисления SHA-2. Параметр типа `instance` встроен в сам тип, поэтому `digest()` возвращает точный массив `ShaDigest_NNN` без рантаймовой диспетчеризации. **Предпочитайте этот тип `ShaState` везде, где возможно.**

---

### Тип: `ShaState`

```nim
type ShaState* = distinct ShaContext
```

Нетипизированный вариант. Алгоритм выбирается в рантайме; пользователь сам отвечает за передачу буфера достаточного размера в `digest`.

---

### `digestLength` (SHA-2)

```nim
func digestLength*(instance: ShaInstance): int
```

Возвращает количество байт вывода для данного `ShaInstance` в рантайме. Полезно при выделении буфера перед вызовом нетипизированного `digest`.

```nim
assert digestLength(Sha_256) == 32
assert digestLength(Sha_512) == 64
```

---

### `initSha_224` / `initSha_256` / `initSha_384` / `initSha_512`

```nim
func initSha_224*(): ShaStateStatic[Sha_224]
func initSha_256*(): ShaStateStatic[Sha_256]
func initSha_384*(): ShaStateStatic[Sha_384]
func initSha_512*(): ShaStateStatic[Sha_512]
```

Создают свежее, статически типизированное состояние SHA-2, готовое принимать данные. Это рекомендованные точки входа для большинства задач. Каждая функция возвращает состояние, привязанное к конкретному размеру дайджеста — компилятор проверит все последующие операции.

```nim
var hasher = initSha_256()
```

---

### `initSha`

```nim
func initSha*(instance: ShaInstance): ShaState
```

Создаёт нетипизированное состояние SHA, выбранное по рантаймовому значению. Используйте, когда алгоритм неизвестен на этапе компиляции (например, конфигурация пользователя, согласование протокола).

```nim
let algo = Sha_512  # может прийти из конфига
var hasher = initSha(algo)
```

---

### `initShaStateStatic`

```nim
func initShaStateStatic*(instance: static ShaInstance): ShaStateStatic[instance]
```

Обобщённая форма четырёх хелперов `initSha_NNN`. Полезна при написании обобщённого кода, параметризованного по `ShaInstance`.

---

### `update` (SHA-2)

```nim
proc update*[instance: static ShaInstance](state: var ShaStateStatic[instance]; data: openArray[char])
proc update*(state: var ShaState; data: openArray[char])
```

Добавляет блок данных в хешер. Вызывается любое количество раз с фрагментами любого размера — результат идентичен конкатенации всех фрагментов и хешированию за один раз. Именно этот метод вы вызываете в потоковом цикле.

```nim
var hasher = initSha_256()
hasher.update("The quick brown fox ")
hasher.update("jumps over the lazy dog")
```

---

### `digest` (SHA-2, статический)

```nim
proc digest*(state: var ShaStateStatic[Sha_224]): ShaDigest_224
proc digest*(state: var ShaStateStatic[Sha_256]): ShaDigest_256
proc digest*(state: var ShaStateStatic[Sha_384]): ShaDigest_384
proc digest*(state: var ShaStateStatic[Sha_512]): ShaDigest_512
```

Финализирует хеш и возвращает массив дайджеста подходящего размера. Возвращаемый тип определяется тем, какой `ShaStateStatic` у вас есть. После вызова `digest` состояние исчерпано — нельзя снова вызывать `update`.

```nim
var hasher = initSha_256()
hasher.update("The quick brown fox jumps over the lazy dog")
let d = hasher.digest()
assert $d == "d7a8fbb307d7809469ca9abcb0082e4f8d5651e46d3cdb762d02d0bf37c9e592"
```

---

### `digest` (SHA-2, нетипизированный)

```nim
proc digest*(state: var ShaState; dest: var openArray[char]): int
```

Финализирует хеш и записывает результат в предоставленный вызывающим буфер `dest`. Возвращает количество фактически записанных байт. Если `dest` меньше дайджеста, вывод молча усекается — убедитесь, что выделяете достаточно места (используйте `digestLength` для определения нужного размера).

```nim
var hasher = initSha(Sha_512)
hasher.update("hello")
var buf = newString(64)
let written = hasher.digest(buf)
assert written == 64
```

---

### `secureHash` (SHA-2, статический)

```nim
proc secureHash*(instance: static ShaInstance; data: openArray[char]): auto
```

Однострочник со статической типизацией: инициализировать, обновить, вернуть дайджест. Возвращаемый тип автоматически является нужным `ShaDigest_NNN` для выбранного экземпляра. Используйте, когда данные доступны целиком и потоковость не нужна.

```nim
let d = secureHash(Sha_256, "Hello World")
echo $d
```

---

### `secureHash` (SHA-2, динамический)

```nim
proc secureHash*(instance: ShaInstance; data: openArray[char]): seq[char]
```

То же самое, но с выбором экземпляра в рантайме. Возвращает `seq[char]` соответствующей длины.

```nim
let algo = Sha_384
let d = secureHash(algo, "Hello World")
assert d.len == 48
```

---

### `$` (дайджест SHA-2)

Оператор `$` для массивов дайджестов SHA-2 реэкспортируется из `sha_utils`. Преобразует любой `ShaDigest_NNN` в строку строчных hex-символов.

```nim
let d = secureHash(Sha_256, "abc")
echo $d  # "ba7816bf8f01cfea414140de5dae2ec73b00361bbef0469348423f656b8f3d3"
```

---

## SHA-3 и SHAKE

> **Модуль:** `sha3`

SHA-3 основан на конструкции **Keccak-sponge** («губка»), что делает его структурно независимым от MD5/SHA-1/SHA-2 вопреки общему названию «SHA». Эта независимость — значительное преимущество: слабости SHA-2 не переносятся на SHA-3.

Модуль реализует два семейства API:

- **SHA-3** — дайджесты фиксированного размера (224 / 256 / 384 / 512 бит), интерфейс аналогичен SHA-2.
- **SHAKE** — функции с расширяемым выводом (XOF, extendable-output function). Вы «выжимаете» столько байт, сколько нужно, за столько чтений, сколько нужно. Три уровня безопасности: 128, 256 и 512 бит.

---

### Тип: `Sha3Instance`

```nim
type Sha3Instance* = enum
  Sha3_224, Sha3_256, Sha3_384, Sha3_512
```

Выбирает вариант SHA-3.

---

### Типы: `Sha3Digest_224/256/384/512`

Массивы фиксированного размера, аналогичные типам дайджестов SHA-2.

---

### Тип: `Sha3StateStatic[instance]`

Статически типизированное состояние вычисления SHA-3. Возвращается хелперами `initSha3_NNN`. Рекомендуется для большинства задач.

---

### Тип: `Sha3State`

Нетипизированное состояние SHA-3 с выбором экземпляра в рантайме.

---

### `digestLength` (SHA-3)

```nim
func digestLength*(instance: Sha3Instance): int
```

Возвращает количество выходных байт для данного экземпляра SHA-3: 28, 32, 48 или 64.

---

### `initSha3_224` / `initSha3_256` / `initSha3_384` / `initSha3_512`

```nim
func initSha3_224*(): Sha3StateStatic[Sha3_224]
func initSha3_256*(): Sha3StateStatic[Sha3_256]
func initSha3_384*(): Sha3StateStatic[Sha3_384]
func initSha3_512*(): Sha3StateStatic[Sha3_512]
```

Создают свежий статически типизированный хешер SHA-3. Рекомендованные точки входа.

```nim
var hasher = initSha3_256()
```

---

### `initSha3`

```nim
func initSha3*(instance: Sha3Instance): Sha3State
```

Создаёт нетипизированное состояние SHA-3 в рантайме. Используйте, когда вариант — не компилируемая константа.

---

### `initSha3StateStatic`

```nim
func initSha3StateStatic*(instance: static Sha3Instance): Sha3StateStatic[instance]
```

Обобщённая форма хелперов `initSha3_NNN`.

---

### `update` (SHA-3)

```nim
proc update*[instance: static Sha3Instance](state: var Sha3StateStatic[instance]; data: openArray[char])
proc update*(state: var Sha3State; data: openArray[char])
```

Добавляет данные в хешер инкрементально. Семантика идентична `update` SHA-2.

```nim
var hasher = initSha3_256()
hasher.update("The quick brown fox ")
hasher.update("jumps over the lazy dog")
```

---

### `digest` (SHA-3, статический)

```nim
proc digest*(state: var Sha3StateStatic[Sha3_224]): Sha3Digest_224
proc digest*(state: var Sha3StateStatic[Sha3_256]): Sha3Digest_256
proc digest*(state: var Sha3StateStatic[Sha3_384]): Sha3Digest_384
proc digest*(state: var Sha3StateStatic[Sha3_512]): Sha3Digest_512
```

Финализирует и возвращает типизированный массив дайджеста.

```nim
var hasher = initSha3_256()
hasher.update("The quick brown fox jumps over the lazy dog")
let d = hasher.digest()
assert $d == "69070dda01975c8c120c3aada1b282394e7f032fa9cf32f4cb2259a0897dfc04"
```

---

### `digest` (SHA-3, нетипизированный)

```nim
proc digest*(state: var Sha3State; dest: var openArray[char]): int
```

То же, что нетипизированный `digest` SHA-2: записывает в предоставленный буфер, возвращает количество байт, усекает при переполнении.

---

### `secureHash` (SHA-3, статический)

```nim
proc secureHash*(instance: static Sha3Instance; data: openArray[char]): auto
```

Однострочное хеширование с выбором SHA-3 экземпляра на этапе компиляции.

```nim
let d = secureHash(Sha3_512, "Hello World")
echo $d
```

---

### `secureHash` (SHA-3, динамический)

```nim
proc secureHash*(instance: Sha3Instance; data: openArray[char]): seq[char]
```

Однострочное хеширование с выбором экземпляра в рантайме. Возвращает `seq[char]`.

---

### Тип: `ShakeInstance`

```nim
type ShakeInstance* = enum
  Shake128   # 128-битная стойкость
  Shake256   # 256-битная стойкость
  Shake512   # 512-битная (предложение Keccak, не входит в стандарт SHA-3)
```

Выбирает уровень безопасности SHAKE.

---

### Тип: `ShakeState`

```nim
type ShakeState* = distinct KeccakState
```

Хранит изменяемое состояние вычисления SHAKE. В отличие от SHA-3, у SHAKE нет статического варианта — длина вывода по определению переменна.

---

### `digestLength` (SHAKE)

```nim
func digestLength*(instance: ShakeInstance): int
```

Возвращает **размер блока по умолчанию** для экземпляра SHAKE: 16, 32 или 64 байта. Это информационная функция — `shakeOut` может записывать любое количество байт независимо от неё.

---

### `initShake`

```nim
func initShake*(instance: ShakeInstance): ShakeState
```

Создаёт свежее состояние SHAKE для данного экземпляра. Единственная точка входа для всех операций SHAKE.

```nim
var xof = initShake(Shake128)
```

---

### `update` (SHAKE)

```nim
proc update*(state: var ShakeState; data: openArray[char])
```

Добавляет данные в фазу поглощения SHAKE. Вызывается любое количество раз перед `finalize`.

```nim
xof.update("The quick brown fox ")
xof.update("jumps over the lazy dog")
```

---

### `finalize` (SHAKE)

```nim
proc finalize*(state: var ShakeState)
```

Сигнализирует об окончании ввода. **Должен быть вызван перед любым вызовом `shakeOut`.** После вызова больше нельзя добавлять данные через `update`.

```nim
xof.finalize()
```

---

### `shakeOut`

```nim
proc shakeOut*(state: var ShakeState; dest: var openArray[char])
```

«Выжимает» байты из XOF в буфер `dest`. Вызывается многократно после `finalize` — каждый вызов продолжает с того места, где остановился предыдущий, производя непрерывный поток. Суммарный вывод одинаков вне зависимости от того, как вы его разбиваете на вызовы.

Это ключевая особенность XOF: вы запрашиваете ровно столько байт, сколько требует ваш протокол, и поток детерминирован.

```nim
var xof = initShake(Shake128)
xof.update("secret")
xof.finalize()

var key: array[32, char]
var iv:  array[16, char]

xof.shakeOut(key)  # первые 32 байта
xof.shakeOut(iv)   # следующие 16 байт
# key и iv выведены из одного исходного материала
```

---

## bcrypt

> **Модуль:** `bcrypt`

bcrypt — алгоритм **хеширования паролей**, а не хеш общего назначения. Он намеренно медленный благодаря настраиваемому коэффициенту стоимости, что делает атаки перебором дорогостоящими даже на специализированном железе. Также включает соль на каждый пароль для защиты от предвычисленных таблиц (rainbow tables).

Модуль реализует подверсию `2b` (текущий стандарт OpenBSD) и умеет проверять хеши `2a` и `2y` (PHP) для совместимости.

**bcrypt — правильный выбор для хранения паролей пользователей.** Не используйте MD5, SHA-1 или даже SHA-256 для паролей — они быстрые, а это именно то, чего вы не хотите при хранении паролей.

---

### Тип: `CostFactor`

```nim
type CostFactor* = range[4..31]
```

Целое число в диапазоне `[4, 31]`, представляющее log₂ числа итераций bcrypt. Стоимость `n` означает 2ⁿ раундов расширения ключа. Каждая единица увеличения примерно удваивает время вычисления.

- **4** — минимум; практически нет задержки (только для тестирования).
- **10–12** — распространённые продакшн-значения на железе 2020-х годов. Стремитесь к ~100–300 мс на хеш на вашем сервере.
- **14+** — для сценариев высокой безопасности, где бюджет задержки позволяет.

---

### Тип: `Salt`

```nim
type Salt = object  # непрозрачный
```

Непрозрачный объект, хранящий 16 случайных байт соли, коэффициент стоимости и символ подверсии bcrypt (`'a'`, `'b'` или `'y'`). Никогда не конструируйте `Salt` напрямую — используйте `generateSalt` или `parseSalt`. Оператор `$` конвертирует `Salt` в строку вида `$2b$NN$...`.

---

### Тип: `SaltedHash`

```nim
type SaltedHash = object  # непрозрачный
```

Непрозрачный объект, хранящий завершённый хеш bcrypt вместе с солью, коэффициентом стоимости и 24 хешированными байтами. Оператор `$` конвертирует его в каноническую строку `$2b$NN$...`, которую вы сохраняете в базе данных.

---

### `generateSalt`

```nim
proc generateSalt*(cost: CostFactor): Salt {.raises: ResourceExhaustedError.}
```

Генерирует новую случайную 16-байтовую соль с использованием ГПСЧ операционной системы (через `std/sysrand`). Всегда использует подверсию `2b`. Параметр `cost` управляет стоимостью последующего хеширования.

Вызывайте один раз для каждого пароля при регистрации. **Никогда не переиспользуйте соль для разных паролей.**

```nim
let salt = generateSalt(12)  # ~300 мс на современном железе — настройте под свой сервер
```

> **Ошибка:** Бросает `ResourceExhaustedError`, если ОС не может выдать достаточно энтропии (крайне редко).

---

### `parseSalt`

```nim
proc parseSalt*(salt: string): Salt {.raises: ValueError.}
```

Разбирает `Salt` из преамбулы соли bcrypt (`$2b$NN$<22символа>`) или из полной строки хеша bcrypt. Это мост между строковым миром (хранение в базе данных) и объектом `Salt`.

```nim
# Из одной преамбулы
let s1 = parseSalt("$2b$12$LzUyyYdKBoEy9V4NTvxDH.")

# Из полного сохранённого хеша — полезно для извлечения соли обратно
let s2 = parseSalt("$2b$12$LzUyyYdKBoEy9V4NTvxDH.O11KQP30/Zyp5pQAQ.0Cy89WnkD5Jjy")

assert $s1 == $s2  # одна и та же соль в обоих случаях
```

> **Ошибка:** Бросает `ValueError` при неподдерживаемых подверсиях, выходящем за диапазон коэффициенте стоимости или искажённых данных соли.

---

### `bcrypt`

```nim
proc bcrypt*(password: openArray[char]; salt: Salt): SaltedHash
```

Основная функция хеширования. Принимает пароль и соль, выполняет 2^cost раундов расширения ключа на основе Blowfish и возвращает `SaltedHash`. Преобразование результата через `$` даёт каноническую строку для хранения.

Ключевые детали реализации:
- Пароли длиннее **72 байт** молча усекаются (поведение `2b`/`2y`).
- Подверсия `2a` воспроизводит старый баг переполнения OpenBSD для обратной совместимости.

```nim
let salt   = generateSalt(10)
let hashed = bcrypt("правильная лошадь батарея скобка", salt)
let stored = $hashed  # сохраните это в базе данных
echo stored  # "$2b$10$..."
```

> **Предупреждение безопасности:** Если соль поступает из ненадёжного источника, злоумышленник может передать соль с коэффициентом стоимости 31, вызвав отказ в обслуживании — ваш сервер будет вынужден вычислять астрономически дорогой хеш. Всегда генерируйте соли самостоятельно.

---

### `verify`

```nim
proc verify*(password: openArray[char]; knownGood: string): bool
```

Стандартная проверка при входе. Повторно хеширует `password` с солью, встроенной в `knownGood`, и сравнивает результат. Возвращает `true`, если пароль совпадает.

Это **правильный** способ аутентификации пароля — не пытайтесь вручную извлекать и сравнивать байты хеша.

```nim
let stored = "$2b$10$LzUyyYdKBoEy9V4NTvxDH.O11KQP30/Zyp5pQAQ.0Cy89WnkD5Jjy"

if verify(userInputPassword, stored):
  echo "Доступ разрешён"
else:
  echo "Доступ запрещён"
```

> **Предупреждение безопасности:** То же предостережение относительно ненадёжных строк `knownGood` применимо здесь так же, как и к самому `bcrypt` — управляемый злоумышленником коэффициент стоимости в хеш-строке может вызвать DoS.

---

## Как выбрать алгоритм

| Сценарий использования | Рекомендация |
|---|---|
| Хранение паролей | **bcrypt** (cost ≥ 10) |
| Криптографическая целостность общего назначения | **SHA-256** или **SHA3-256** |
| Высокая безопасность / защита на будущее | **SHA-512** или **SHA3-512** |
| Ключевыведение переменной длины | **SHAKE256** |
| Совместимость с устаревшими протоколами | SHA-1 (только чтение; не создавайте новых применений) |
| Контрольные суммы файлов, незащищённые задачи | MD5 или SHA-1 |

Никогда не используйте MD5 или SHA-1 в новом коде, где важна безопасность. Для хранения паролей также стоит рассмотреть Argon2 (не входит в этот пакет) для защиты от GPU-атак.
