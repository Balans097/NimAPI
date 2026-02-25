# std/socketstreams — справочник модуля

> **Входит в стандартную библиотеку Nim** (`std/socketstreams`).

## Обзор

`std/socketstreams` соединяет два самостоятельно полезных абстракции: **сокеты** (из `std/net`) и **потоки** (из `std/streams`). Поток в Nim — это единый интерфейс для последовательного чтения и записи: один и тот же код, читающий из файлового потока, работает с любой другой реализацией потока. Этот модуль предоставляет объекты потоков, опирающихся на сетевой сокет — и все помощники уровня потока (`readLine`, `readStr`, `peekStr`, `write`, `setPosition`, `getPosition`, `flush`, `close` и т.д.) работают прозрачно поверх сокетного соединения.

Модуль предоставляет **два различных типа потоков** с разными контрактами:

| Тип | Запись? | Чтение? | Откуда поступают данные при чтении |
|---|---|---|---|
| `ReadSocketStream` | Нет | Да | Живой сокет (`recv`) + внутренний буфер |
| `WriteSocketStream` | Да | Да | Только внутренний буфер (никогда из сокета) |

Понимание **внутреннего буфера** — ключ к правильному использованию обоих типов:

- В `ReadSocketStream` входящие байты принимаются из сокета *однократно* и сохраняются в буфере. После этого перемотка назад перечитывает из буфера — сокет больше не вызывается. Буфер растёт по мере поступления новых данных.
- В `WriteSocketStream` записи уходят в буфер, чтения возвращают из него же, а `flush` — явное действие, отправляющее буферизованные байты через сокет. Сам сокет никогда не читается.

---

## Типы

### `ReadSocketStreamObj` / `ReadSocketStream`

```nim
type
  ReadSocketStream* = ref ReadSocketStreamObj
  ReadSocketStreamObj* = object of StreamObj
    data: Socket     # базовый сокет
    pos:  int        # текущая позиция чтения/записи в buf
    buf:  seq[byte]  # внутренний буфер накопления
```

Ссылочный объект потока, оборачивающий `Socket` для **чтения**. Все байты, принятые из сокета, накапливаются в `buf`, индексируемом через `pos`. Как только данные попали в буфер — они остаются там до вызова `resetStream`. Это означает, что `setPosition(0)` с последующим `readLine` вернёт данные из буфера без каких-либо обращений к сокету.

`ReadSocketStream` **не поддерживает** `write` и `flush` — попытки их вызова приведут к ошибке, поскольку `writeDataImpl` не установлен.

---

### `WriteSocketStreamObj` / `WriteSocketStream`

```nim
type
  WriteSocketStream* = ref WriteSocketStreamObj
  WriteSocketStreamObj* = object of ReadSocketStreamObj
    lastFlush: int   # индекс первого байта, ещё не отправленного
```

Наследует `ReadSocketStreamObj` и добавляет возможность записи. Дополнительное поле `lastFlush` — отметка максимума: байты с `0` до `lastFlush - 1` уже были отправлены через сокет и неизменяемы. Попытка их перезаписи вызывает `IOError`. Байты начиная с `lastFlush` всё ещё можно записывать.

В отличие от `ReadSocketStream`, чтение из `WriteSocketStream` **полностью локально** — оно читает из `buf`, а не из сокета.

`atEnd` у `WriteSocketStream` возвращает `true`, когда `pos == buf.len`, то есть позиция достигла конца записанного.

---

## Конструирующие процедуры

---

### `newReadSocketStream`

```nim
proc newReadSocketStream*(s: Socket): owned ReadSocketStream
```

**Создаёт новый `ReadSocketStream`, оборачивающий заданный сокет.**

Возвращаемый поток владеет своим объектом потока (но не сокетом — вы сохраняете владение `s` и несёте ответственность за то, чтобы не закрывать его отдельно, пока поток используется). Внутренний буфер изначально пуст, `pos` равен `0`.

Подключает следующие операции потока: `close`, `atEnd` (всегда `false`), `setPosition`, `getPosition`, `readData`, `peekData`, `readDataStr`. Операции записи **не подключены**.

**Пример — чтение ответа по UDP:**

```nim
import std/[net, socketstreams]

var socket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
socket.sendTo("127.0.0.1", Port(12345), "PING")

var stream = newReadSocketStream(socket)
echo stream.readLine()     # получает из сокета, сохраняет в буфер
stream.setPosition(0)
echo stream.readLine()     # читает из буфера — сокет не вызывается
stream.close()
```

---

### `newWriteSocketStream`

```nim
proc newWriteSocketStream*(s: Socket): owned WriteSocketStream
```

**Создаёт новый `WriteSocketStream`, оборачивающий заданный сокет.**

Буфер изначально пуст, `pos` равен `0`, `lastFlush` равен `0` (ничего ещё не отправлено). Подключает: `close`, `atEnd`, `setPosition`, `getPosition`, `writeData`, `readData`, `peekData`, `flush`. Операция `readDataStr` **не подключена**.

**Пример — построение и отправка пакета:**

```nim
import std/[net, socketstreams]

var socket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
socket.connect("127.0.0.1", Port(12345))

var stream = newWriteSocketStream(socket)
stream.write("NOM")
stream.setPosition(1)
echo stream.peekStr(2)    # "OM"  — читает из буфера
stream.write("I")         # перезаписывает позицию 1 символом 'I'
stream.setPosition(0)
echo stream.readStr(3)    # "NIM" — читает из буфера
stream.flush()            # отправляет "NIM" через сокет
```

---

## `resetStream`

```nim
proc resetStream*(s: ReadSocketStream)
proc resetStream*(s: WriteSocketStream)
```

**Очищает внутренний буфер и сбрасывает позицию потока в `0`.**

Два перегруженных варианта — по одному для каждого типа потока. Для `WriteSocketStream` `resetStream` также сбрасывает `lastFlush` в `0`.

Вызов `resetStream` **никак не влияет** на базовый сокет — никакие данные не отправляются, соединение не закрывается, сокет остаётся открытым и рабочим. Это исключительно операция управления буфером.

После `resetStream`:
- Следующее чтение из `ReadSocketStream` получит свежие данные из сокета (буфер пуст, обслуживать из кеша нечего).
- Следующая запись в `WriteSocketStream` начнёт заполнять буфер с начала, а следующий `flush` отправит начиная с позиции `0`.

**Пример — повторное чтение из сокета после очистки буфера:**

```nim
import std/[net, socketstreams]

var socket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
var stream = newReadSocketStream(socket)

socket.sendTo("127.0.0.1", Port(12345), "ЗАПРОС 1")
echo stream.readLine()    # читает из сокета в буфер

stream.resetStream()      # буфер очищен; pos = 0

socket.sendTo("127.0.0.1", Port(12345), "ЗАПРОС 2")
echo stream.readLine()    # читает из сокета снова
```

---

## Поведенческие детали

### ReadSocketStream — семантика буфера

Буфер растёт лениво по мере поступления данных. Когда вы вызываете операцию чтения или подглядывания, и запрошенный диапазон выходит за границы текущего буфера, поток вызывает `recv` на базовом сокете, чтобы заполнить недостающие байты. Если сокет возвращает меньше байтов, чем запрошено (например, короткая UDP-датаграмма), возвращаемый счётчик отражает фактически доступное количество.

Перемотка назад (`setPosition` на позицию, уже находящуюся в буфере) никогда не обращается к сокету — данные обслуживаются целиком из буфера. Это позволяет безопасно реализовывать парсеры с необходимостью отката: вы платите за обращение к сокету лишь один раз за каждый байт.

`atEnd` у `ReadSocketStream` всегда возвращает `false`. Поскольку TCP-поток или UDP-сокет всегда могут принять ещё данные, надёжно определить, когда наступил «конец», без прикладного фреймирования невозможно.

### WriteSocketStream — неизменяемость после flush

`flush` помечает текущую длину буфера как новый `lastFlush`. Всё, что находится перед этой точкой, **неизменяемо**: если вы переместите позицию назад в уже отправленный регион и попытаетесь записать, немедленно вызывается `IOError` с сообщением `"Unable to write into buffer that has already been sent"`. Вы всё ещё можете *читать* (подглядывать) из отправленного региона — он остаётся в буфере — но изменить его нельзя.

Это предотвращает труднообнаруживаемые ошибки, когда данные в полёте молча перезаписываются. Если нужен полностью чистый буфер — вызовите `resetStream`.

### WriteSocketStream — чтение за конец буфера

В `WriteSocketStream` попытка прочитать или подглядеть за текущим концом буфера вызывает `IOError`: `"Unable to read past end of write buffer"`. Резервного обращения к сокету нет — чтение приходит только из того, что было явно записано в буфер.

---

## Унаследованные операции потока

`ReadSocketStream` и `WriteSocketStream` оба наследуют полный интерфейс `std/streams`. Следующие операции (предоставляемые `std/streams`) работают прозрачно на обоих типах с учётом описанных выше ограничений:

| Операция | ReadSocketStream | WriteSocketStream |
|---|---|---|
| `readLine()` | ✔ читает из сокета/буфера | ✘ (нет чтения из сокета) |
| `readStr(n)` | ✔ | ✔ читает из буфера |
| `peekStr(n)` | ✔ | ✔ подглядывает в буфер |
| `write(x)` | ✘ не поддерживается | ✔ пишет в буфер |
| `flush()` | ✘ нет операции | ✔ отправляет буфер через сокет |
| `setPosition(n)` | ✔ | ✔ |
| `getPosition()` | ✔ | ✔ |
| `atEnd()` | всегда `false` | `true` когда pos == buf.len |
| `close()` | ✔ закрывает сокет | ✔ закрывает сокет |
| `resetStream(s)` | ✔ очищает буфер, pos=0 | ✔ очищает буфер, pos=0, lastFlush=0 |

---

## Полные примеры использования

### ReadSocketStream — шаблон «перемотать и перечитать»

```nim
import std/[net, socketstreams]

var socket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
var stream = newReadSocketStream(socket)

socket.sendTo("127.0.0.1", Port(12345), "SOME REQUEST")

let line = stream.readLine()   # получает из сокета
echo line

stream.setPosition(0)
echo stream.readLine()         # обслуживается из буфера; сокет не вызывается

stream.resetStream()           # буфер очищен
echo stream.readLine()         # получает из сокета снова

stream.close()                 # закрывает базовый сокет
```

### WriteSocketStream — построить, проверить, отправить

```nim
import std/[net, socketstreams]

var socket = newSocket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
socket.connect("127.0.0.1", Port(12345))

var sendStream = newWriteSocketStream(socket)

sendStream.write("NOM")         # буфер: [N, O, M], pos = 3
sendStream.setPosition(1)
echo sendStream.peekStr(2)      # "OM" — подглядывает позиции 1-2

sendStream.write("I")           # буфер: [N, I, M], pos = 2
sendStream.setPosition(0)
echo sendStream.readStr(3)      # "NIM", pos = 3

echo sendStream.getPosition()   # 3

sendStream.flush()              # отправляет "NIM" через сокет; lastFlush = 3

sendStream.setPosition(1)
sendStream.write("I")           # ОШИБКА: позиция 1 находится до lastFlush (3)
```

---

## Сводная таблица

| Символ | Вид | Назначение |
|--------|-----|------------|
| `ReadSocketStream` | ref object | Поток только для чтения, опирающийся на сокет |
| `WriteSocketStream` | ref object | Поток чтения/записи с семантикой «буфер → flush» |
| `newReadSocketStream(s)` | процедура | Обернуть сокет в ReadSocketStream |
| `newWriteSocketStream(s)` | процедура | Обернуть сокет в WriteSocketStream |
| `resetStream(s: ReadSocketStream)` | процедура | Очистить буфер, сбросить pos в 0 |
| `resetStream(s: WriteSocketStream)` | процедура | Очистить буфер, сбросить pos и lastFlush в 0 |

---

> **Смотри также:** [`std/streams`](https://nim-lang.org/docs/streams.html) — полный интерфейс потоков (все операции, наследуемые сокетными потоками); [`std/net`](https://nim-lang.org/docs/net.html) — тип `Socket` и создание сокетов.
