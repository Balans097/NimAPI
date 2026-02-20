# Nim `std/parsesql` — Справочник по модулю

> **Модуль:** `std/parsesql`  
> **Статус:** Нестабильный API — интерфейс может меняться между версиями Nim.  
> **Область применения:** Разбор синтаксиса PostgreSQL и стандарта SQL ANSI.

`parsesql` реализует полный SQL-конвейер из трёх этапов: **лексический анализ** (токенизация исходного текста), **синтаксический анализ** (построение AST из токенов) и **рендеринг** (преобразование AST обратно в SQL-строку). Публичная поверхность намеренно невелика — основная сложность скрыта внутри лексера и парсера — но понимание типов узлов AST необходимо для работы с результатом.

---

## Содержание

1. [Типы](#типы)
   - [SqlLexer](#sqllexer)
   - [SqlParseError](#sqlparseerror)
   - [SqlNodeKind](#sqlnodekind)
   - [SqlNode и SqlNodeObj](#sqlnode-и-sqlnodeobj)
   - [SqlParser](#sqlparser)
2. [Конструирование узлов AST](#конструирование-узлов-ast)
   - [newNode](#newnode)
   - [add](#add)
   - [len](#len)
   - [[] — оператор индексации](#---оператор-индексации)
3. [Разбор SQL](#разбор-sql)
   - [parseSql из строки](#parsesql-из-строки)
   - [parseSql из Stream](#parsesql-из-stream)
4. [Рендеринг](#рендеринг)
   - [renderSql](#rendersql)
   - [$](#-оператор-долларa)
   - [treeRepr](#treerepr)
5. [Справочник по SqlNodeKind](#справочник-по-sqlnodekind)
6. [Полный пример использования](#полный-пример-использования)

---

## Типы

### `SqlLexer`

```nim
type SqlLexer* = object of BaseLexer
```

Токенизатор. Расширяет `BaseLexer` (из `std/lexbase`) специфичной для SQL логикой распознавания всех видов токенов: идентификаторов, экранированных идентификаторов, строковых констант (обычных, escape-строк, dollar-quoted, битовых и шестнадцатеричных), целых и дробных чисел, операторов и знаков препинания.

`SqlLexer` обычно не используется напрямую. Он встроен внутрь `SqlParser`, который, в свою очередь, автоматически создаётся и управляется функцией `parseSql`. Тип публичен прежде всего для того, чтобы пользователи могли расширять или подклассировать парсер.

---

### `SqlParseError`

```nim
type SqlParseError* = object of ValueError
```

Исключение, которое бросается при обнаружении синтаксической ошибки в `parseSql`. Сообщение об ошибке включает имя файла (если оно было передано), номер строки и номер столбца в формате:

```
имяфайла(строка, столбец) Error: описание
```

При разборе строки без имени файла часть с именем файла остаётся пустой. Всегда оборачивайте вызовы `parseSql` в `try/except SqlParseError`, если нужно обрабатывать некорректный ввод.

```nim
try:
  let ast = parseSql("SELECT * FORM users")  # "FORM" — не ключевое слово
except SqlParseError as e:
  echo "Синтаксическая ошибка: ", e.msg
```

---

### `SqlNodeKind`

```nim
type SqlNodeKind* = enum
  nkNone, nkIdent, nkQuotedIdent,
  nkStringLit, nkBitStringLit, nkHexStringLit,
  nkIntegerLit, nkNumericLit,
  nkPrimaryKey, nkForeignKey, nkNotNull, nkNull,
  nkStmtList, nkDot, nkDotDot, nkPrefix, nkInfix,
  nkCall, nkPrGroup, nkColumnReference,
  nkReferences, nkDefault, nkCheck, nkConstraint,
  nkUnique, nkIdentity, nkColumnDef,
  nkInsert, nkUpdate, nkDelete,
  nkSelect, nkSelectDistinct, nkSelectColumns, nkSelectPair,
  nkAsgn, nkFrom, nkFromItemPair,
  nkJoin, nkNaturalJoin, nkUsing,
  nkGroup, nkLimit, nkOffset, nkHaving, nkOrder, nkDesc,
  nkUnion, nkIntersect, nkExcept,
  nkColumnList, nkValueList, nkWhere,
  nkCreateTable, nkCreateTableIfNotExists,
  nkCreateType, nkCreateTypeIfNotExists,
  nkCreateIndex, nkCreateIndexIfNotExists,
  nkEnumDef
```

Полное перечисление всех видов узлов SQL AST. Поле `kind` каждого `SqlNode` имеет этот тип. Существуют два широких семейства:

**Листовые узлы** (объединяются под именем `LiteralNodes`): `nkIdent`, `nkQuotedIdent`, `nkStringLit`, `nkBitStringLit`, `nkHexStringLit`, `nkIntegerLit`, `nkNumericLit`. Хранят текстовое значение в поле `strVal` и не имеют дочерних узлов (`len` всегда равен 0).

**Внутренние узлы**: всё остальное. Хранят логическую подструктуру в поле `sons: seq[SqlNode]`.

Описание каждого значения и его дочерних узлов — в разделе [Справочник по SqlNodeKind](#справочник-по-sqlnodekind).

---

### `SqlNode` и `SqlNodeObj`

```nim
type
  SqlNode* = ref SqlNodeObj
  SqlNodeObj* = object
    case kind*: SqlNodeKind
    of LiteralNodes:
      strVal*: string
    else:
      sons*: seq[SqlNode]
```

Узел SQL-дерева разбора. `SqlNode` — ссылочный тип: копирование переменной `SqlNode` копирует ссылку, а не сам узел.

**Доступ к данным:**
- Для листовых узлов читайте `node.strVal`, чтобы получить текст токена.
- Для внутренних узлов перебирайте `node.sons` или используйте `node[i]` для доступа к дочерним элементам.
- Всегда проверяйте `node.kind` перед обращением к `strVal` или `sons`: вариантный объект делает только одно поле допустимым в каждый момент.

```nim
let ast = parseSql("SELECT id FROM users")
# ast.kind == nkStmtList
# ast[0].kind == nkSelect
# ast[0][0].kind == nkSelectColumns
# ast[0][0][0].kind == nkSelectPair
# ast[0][0][0][0].kind == nkIdent
# ast[0][0][0][0].strVal == "id"
```

---

### `SqlParser`

```nim
type SqlParser* = object of SqlLexer
  tok: Token
  considerTypeParams: bool
```

Объект парсера, встраивающий лексер. Обычно создаётся и уничтожается автоматически внутри `parseSql`. Флаг `considerTypeParams` (доступный через одноимённый параметр `parseSql`) управляет тем, сохраняются ли параметры типов столбцов — например, `255` в `VARCHAR(255)` — в AST или молча отбрасываются.

Когда `considerTypeParams = false` (по умолчанию), `VARCHAR(255)` порождает простой узел `nkIdent` со `strVal = "varchar"`. Когда `true` — узел `nkCall`, первый дочерний элемент которого — имя типа, остальные — аргументы размера.

---

## Конструирование узлов AST

Эти функции позволяют строить или изменять AST вручную — для генерации кода, трансформаций AST или программного построения SQL-запросов.

### `newNode`

```nim
proc newNode*(k: SqlNodeKind): SqlNode
proc newNode*(k: SqlNodeKind, s: string): SqlNode
proc newNode*(k: SqlNodeKind, sons: seq[SqlNode]): SqlNode
```

Создаёт новый `SqlNode` вида `k`. Три перегрузки:

- **Без дополнительных аргументов:** создаёт пустой листовой или внутренний узел без дочерних элементов. Для литеральных видов устанавливает `strVal = ""`; для остальных `sons = @[]`.
- **С `s: string`:** создаёт листовой узел со `strVal = s`. Имеет смысл только для `LiteralNodes`.
- **С `sons: seq[SqlNode]`:** создаёт внутренний узел с уже прикреплёнными дочерними элементами.

```nim
let ident  = newNode(nkIdent, "users")           # nkIdent, strVal = "users"
let select = newNode(nkSelect)                    # nkSelect, дочерних нет
let infix  = newNode(nkInfix, @[
  newNode(nkIdent, "="),
  newNode(nkIdent, "id"),
  newNode(nkIntegerLit, "42")
])
```

---

### `add`

```nim
proc add*(father, n: SqlNode)
```

Добавляет `n` как последний дочерний элемент `father`. `father` должен быть внутренним узлом (не `LiteralNode`) — добавление дочерних элементов к листовому узлу обратилось бы к полю `sons` несуществующего варианта.

```nim
let selectCols = newNode(nkSelectColumns)
selectCols.add(newNode(nkSelectPair, @[newNode(nkIdent, "*")]))
```

---

### `len`

```nim
proc len*(n: SqlNode): int
```

Возвращает количество дочерних элементов `n`. Для листовых (`LiteralNodes`) узлов всегда возвращает 0 вне зависимости от того, было бы поле `sons` доступно. Для внутренних — возвращает `n.sons.len`.

```nim
let ast = parseSql("SELECT a, b FROM t")
echo ast.len           # 1 (один оператор)
echo ast[0].len        # несколько дочерних: SelectColumns, From, ...
echo ast[0][0][0].len  # 0, если это листовой nkIdent
```

---

### `[]` — оператор индексации

```nim
proc `[]`*(n: SqlNode; i: int): SqlNode
proc `[]`*(n: SqlNode; i: BackwardsIndex): SqlNode
```

Обращается к `i`-му дочернему элементу `n` по индексу (отсчёт с нуля). Перегрузка с `BackwardsIndex` поддерживает синтаксис `n[^1]` (последний дочерний элемент). Это простые обёртки над `n.sons[i]` и `n.sons[n.len - int(i)]`; при выходе за границы индекса они вызывают панику.

```nim
let ast = parseSql("SELECT x FROM t")
let firstStmt = ast[0]    # узел nkSelect
let lastChild = ast[^1]   # то же, если оператор один
```

---

## Разбор SQL

### `parseSql` из строки

```nim
proc parseSql*(input: string, filename = "", considerTypeParams = false): SqlNode
```

Основная точка входа. Разбирает SQL-текст из `input` и возвращает корень AST — всегда узел `nkStmtList`, даже для одного оператора.

**Параметры:**
- `input` — SQL-текст для разбора.
- `filename` — используется только в сообщениях об ошибках; передайте путь к файлу, если SQL читается с диска, или оставьте пустым.
- `considerTypeParams` — если `true`, параметры размера типов столбцов вроде `VARCHAR(255)` представляются в AST узлами `nkCall`. При `false` (по умолчанию) размер разбирается, но отбрасывается.

**Возвращает:** `SqlNode` с `kind == nkStmtList`, чьи дочерние элементы — отдельные разобранные операторы.

**Бросает:** `SqlParseError` при любой синтаксической ошибке.

```nim
import std/parsesql

let ast = parseSql("SELECT id, name FROM users WHERE active = 1")
echo ast.kind    # nkStmtList
echo ast.len     # 1
echo ast[0].kind # nkSelect

# Несколько операторов, разделённых точками с запятой:
let multi = parseSql("INSERT INTO t VALUES (1); SELECT * FROM t;")
echo multi.len   # 2
```

---

### `parseSql` из Stream

```nim
proc parseSql*(input: Stream, filename: string, considerTypeParams = false): SqlNode
```

То же, что строковая перегрузка, но читает из `std/streams.Stream`. Полезно при разборе больших SQL-файлов без загрузки всего файла в память.

```nim
import std/[parsesql, streams]

let f = newFileStream("schema.sql", fmRead)
defer: f.close()
let ast = parseSql(f, "schema.sql")
```

---

## Рендеринг

### `renderSql`

```nim
proc renderSql*(n: SqlNode, upperCase = false): string
```

Преобразует `SqlNode` (обычно корневой `nkStmtList`, возвращённый `parseSql`, но подойдёт любой под-узел) обратно в SQL-строку. Вывод компактный — без лишних отступов и переносов строк — но синтаксически корректный и допускающий обратный разбор.

**`upperCase`:** при `true` все ключевые слова SQL (`SELECT`, `FROM`, `WHERE` и т. д.) выводятся в верхнем регистре. При `false` (по умолчанию) — в нижнем.

Рендерер обрабатывает каждый `SqlNodeKind` грамматики и применяет следующие соглашения:
- Обычные идентификаторы выводятся как есть, если содержат только печатаемые символы; иначе заключаются в двойные кавычки.
- Экранированные идентификаторы всегда выводятся в двойных кавычках, с удвоением внутренних `"`.
- Строковые литералы — в одинарных кавычках, управляющие символы экранируются как `\xNN`.
- Битовые строки выводятся с префиксом `b'...'`, шестнадцатеричные — `x'...'`.
- Идентификаторы, совпадающие с зарезервированными ключевыми словами (в настоящее время: `select`, `from`, `where`, `group`, `limit`, `offset`, `having`, `count`), автоматически заключаются в двойные кавычки во избежание неоднозначности.

```nim
let ast = parseSql("select id , name from users where id=42")
echo renderSql(ast)
# → "select id,name from users where id = 42"

echo renderSql(ast, upperCase = true)
# → "SELECT id,name FROM users WHERE id = 42"
```

Точность round-trip: разбор с последующим рендерингом даёт функционально эквивалентный SQL, но пробельные символы и регистр литералов нормализуются. Комментарии отбрасываются (лексер их игнорирует).

---

### `$` (оператор доллара)

```nim
proc `$`*(n: SqlNode): string
```

Псевдоним для `renderSql(n)` с `upperCase = false`. Делает `SqlNode` совместимым с `echo`, строковой интерполяцией и любой функцией, ожидающей тип с оператором `$`.

```nim
let ast = parseSql("DELETE FROM logs WHERE age > 30")
echo ast        # использует $, то же что renderSql(ast)
echo $ast[0]    # рендеринг только узла DELETE
```

---

### `treeRepr`

```nim
proc treeRepr*(s: SqlNode): string
```

Возвращает читаемое, структурированное по отступам текстовое представление AST — по одному узлу на строку, с отступом в два пробела на каждый уровень вложенности. Для листовых узлов `strVal` выводится на той же строке после имени вида. Незаменимо при отладке: позволяет точно понять, какую структуру `parseSql` строит для конкретного запроса.

```nim
let ast = parseSql("SELECT a + 1 FROM t")
echo treeRepr(ast)
```

Пример вывода:
```
nkStmtList
  nkSelect
    nkSelectColumns
      nkSelectPair
        nkInfix
          nkIdent +
          nkIdent a
          nkIntegerLit 1
    nkFrom
      nkFromItemPair
        nkIdent t
```

---

## Справочник по SqlNodeKind

В этом разделе описано каждое значение `SqlNodeKind`: соответствующая SQL-конструкция и расположение дочерних элементов.

### Листовые узлы (LiteralNodes)

Все имеют поле `strVal` и не имеют дочерних элементов.

| Вид | Что представляет | Содержимое `strVal` |
|---|---|---|
| `nkNone` | Заглушка / отсутствующее необязательное предложение | `""` |
| `nkIdent` | Неэкранированный идентификатор или ключевое слово как идентификатор | Текст идентификатора, приведённый к нижнему регистру |
| `nkQuotedIdent` | Идентификатор в двойных кавычках `"name"` | Текст идентификатора без кавычек |
| `nkStringLit` | Строка в одинарных кавычках `'abc'`, escape-строка `e'abc'` или dollar-quoted `$$abc$$` | Декодированное содержимое строки |
| `nkBitStringLit` | Битовая строка `B'0101'` | Битовые цифры без префикса и кавычек |
| `nkHexStringLit` | Шестнадцатеричная строка `X'1F'` | Hex-цифры без префикса и кавычек |
| `nkIntegerLit` | Целочисленный литерал `42` | Текст литерала |
| `nkNumericLit` | Дробный или научный литерал `3.14`, `1e5` | Текст литерала |

---

### Узлы ограничений и столбцов

| Вид | Дочерние элементы | Что представляет |
|---|---|---|
| `nkPrimaryKey` | Идентификаторы (на уровне таблицы) или ничего (на уровне столбца) | `PRIMARY KEY` |
| `nkForeignKey` | Список идентификаторов + дочерний `nkReferences` | `FOREIGN KEY ... REFERENCES ...` |
| `nkNotNull` | нет | `NOT NULL` |
| `nkNull` | нет | `NULL` |
| `nkUnique` | нет (уровень столбца) или список идентификаторов | `UNIQUE` |
| `nkIdentity` | нет | `IDENTITY` |
| `nkDefault` | дочернее выражение | `DEFAULT <выражение>` |
| `nkCheck` | дочернее выражение | `CHECK (<выражение>)` |
| `nkConstraint` | имя `nkIdent` + выражение | `CONSTRAINT имя CHECK <выражение>` |
| `nkReferences` | дочерний `nkColumnReference` | `REFERENCES таблица(столбец)` |
| `nkColumnReference` | имя таблицы + одно или более имён столбцов | Ссылка на столбец с необязательным скобочным списком |
| `nkColumnDef` | имя + тип + ноль или более узлов ограничений | Определение одного столбца в `CREATE TABLE` |

---

### Узлы выражений

| Вид | Дочерние элементы | Что представляет |
|---|---|---|
| `nkPrefix` | оператор `nkIdent` + операнд | Унарное префиксное выражение: `-x`, `NOT x` |
| `nkInfix` | оператор `nkIdent` + левый + правый | Бинарное инфиксное выражение: `a + b`, `x = 42` |
| `nkCall` | имя функции + ноль или более аргументов | Вызов функции: `count(*)`, `substr(s, 1, 3)` |
| `nkPrGroup` | одно или более выражений | Скобочная группа выражений `(a, b)` |
| `nkDot` | левый + правый | Составное имя: `схема.таблица` или `таблица.столбец` |
| `nkDotDot` | левый + правый | Обозначение dotdot (редкое расширение PostgreSQL) |
| `nkAsgn` | левый + правый | Присваивание `столбец = выражение` (используется в `UPDATE SET`) |

**Приоритет операторов** (от наименьшего к наибольшему при разборе):
`OR` → `AND` → `BETWEEN` → пользовательские операторы → сравнения/`IS`/`LIKE`/`IN` → `+`/`-` → `*`/`/`/`%`

---

### Узлы операторов

#### SELECT

| Вид | Дочерние элементы | Что представляет |
|---|---|---|
| `nkSelect` | `nkSelectColumns` + необязательные предложения ниже | Оператор `SELECT` |
| `nkSelectDistinct` | то же, что `nkSelect` | Оператор `SELECT DISTINCT` |
| `nkSelectColumns` | одно или более `nkSelectPair` или `nkIdent` (`*`) | Список столбцов |
| `nkSelectPair` | выражение + необязательный псевдоним | `выражение [AS псевдоним]` в списке столбцов |
| `nkFrom` | одно или более `nkFromItemPair` или узлов JOIN | Предложение `FROM` |
| `nkFromItemPair` | таблица/подзапрос + необязательный псевдоним | Один элемент в `FROM`, например `t AS x` |
| `nkJoin` | тип JOIN (`nkIdent`) + левый + правый + ON/USING | Явное объединение: `LEFT JOIN t ON ...` |
| `nkNaturalJoin` | тип JOIN (`nkIdent`) + левый + правый | `NATURAL [тип] JOIN` |
| `nkUsing` | одно или более имён столбцов `nkIdent` | `USING (col1, col2)` |
| `nkWhere` | единственное выражение | `WHERE <выражение>` |
| `nkGroup` | одно или более выражений | `GROUP BY выражение, ...` |
| `nkOrder` | одно или более выражений или `nkDesc` | `ORDER BY выражение [DESC], ...` |
| `nkDesc` | единственный дочерний элемент — выражение | Обёртка над выражением `ORDER BY ... DESC` |
| `nkHaving` | одно или более выражений | `HAVING выражение, ...` |
| `nkLimit` | единственное выражение | `LIMIT выражение` |
| `nkOffset` | единственное выражение | `OFFSET выражение` |
| `nkUnion` | нет | Маркер ключевого слова `UNION` |
| `nkIntersect` | нет | Маркер ключевого слова `INTERSECT` |
| `nkExcept` | нет | Маркер ключевого слова `EXCEPT` |

#### INSERT

| Вид | Дочерние элементы | Что представляет |
|---|---|---|
| `nkInsert` | имя таблицы + `nkColumnList`/`nkNone` + `nkValueList`/`nkDefault` | `INSERT INTO таблица [(столбцы)] VALUES (...)` или `INSERT INTO таблица DEFAULT VALUES` |
| `nkColumnList` | одно или более имён столбцов `nkIdent` | Список `(col1, col2)` после имени таблицы |
| `nkValueList` | одно или более выражений | Список `VALUES (выражение, ...)` |

#### UPDATE

| Вид | Дочерние элементы | Что представляет |
|---|---|---|
| `nkUpdate` | имя таблицы + одно или более `nkAsgn` + `nkWhere`/`nkNone` | `UPDATE таблица SET столбец=выражение [WHERE ...]` |

#### DELETE

| Вид | Дочерние элементы | Что представляет |
|---|---|---|
| `nkDelete` | имя таблицы + `nkWhere`/`nkNone` | `DELETE FROM таблица [WHERE ...]` |

#### DDL (определение данных)

| Вид | Дочерние элементы | Что представляет |
|---|---|---|
| `nkCreateTable` | имя таблицы + узлы столбцов/ограничений | `CREATE TABLE имя (...)` |
| `nkCreateTableIfNotExists` | то же | `CREATE TABLE IF NOT EXISTS имя (...)` |
| `nkCreateType` | имя типа + `nkEnumDef` или тип данных | `CREATE TYPE имя AS ...` |
| `nkCreateTypeIfNotExists` | то же | `CREATE TYPE IF NOT EXISTS имя AS ...` |
| `nkCreateIndex` | имя индекса + имя таблицы + имена столбцов | `CREATE INDEX имя ON таблица (столбцы)` |
| `nkCreateIndexIfNotExists` | то же | `CREATE INDEX IF NOT EXISTS имя ON ...` |
| `nkEnumDef` | одно или более `nkStringLit` | `ENUM ('значение1', 'значение2', ...)` |

#### Верхний уровень

| Вид | Дочерние элементы | Что представляет |
|---|---|---|
| `nkStmtList` | одно или более узлов операторов | Корневой узел; содержит все разобранные операторы |

---

## Полный пример использования

```nim
import std/parsesql

# 1. Разбор
let sql = """
  CREATE TABLE users (
    id   INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    role VARCHAR(20) DEFAULT 'viewer'
  );
  SELECT u.id, u.name
  FROM   users AS u
  WHERE  u.role = 'admin'
  ORDER  BY u.name DESC
  LIMIT  10;
"""

let ast = parseSql(sql, filename = "example.sql", considerTypeParams = false)

# 2. Просмотр дерева
echo treeRepr(ast)

# 3. Обход: подсчёт узлов SELECT
proc countSelects(n: SqlNode): int =
  if n == nil: return 0
  if n.kind == nkSelect or n.kind == nkSelectDistinct:
    result = 1
  if n.kind notin {nkIdent, nkQuotedIdent, nkStringLit,
                   nkBitStringLit, nkHexStringLit,
                   nkIntegerLit, nkNumericLit}:
    for child in n.sons:
      result += countSelects(child)

echo "Операторов SELECT: ", countSelects(ast)  # 1

# 4. Модификация: изменить значение LIMIT
let selectNode = ast[1]  # второй оператор
for child in selectNode.sons:
  if child.kind == nkLimit:
    child.sons[0] = newNode(nkIntegerLit, "25")  # LIMIT 10 → LIMIT 25

# 5. Рендеринг
echo renderSql(ast, upperCase = true)

# 6. Обработка синтаксических ошибок
try:
  discard parseSql("SELECT FORM users")
except SqlParseError as e:
  echo e.msg  # example.sql(1, 8) Error: ...
```

---

## Таблица быстрого доступа

| Задача | API |
|---|---|
| Разобрать SQL-строку в AST | `parseSql(sql)` или `parseSql(sql, filename)` |
| Разобрать SQL из потока в AST | `parseSql(stream, filename)` |
| Преобразовать AST в SQL-строку | `renderSql(node)` или `$node` |
| Рендеринг с ключевыми словами в верхнем регистре | `renderSql(node, upperCase = true)` |
| Отладочная печать структуры AST | `treeRepr(node)` |
| Создать новый узел | `newNode(kind)`, `newNode(kind, strVal)`, `newNode(kind, sons)` |
| Добавить дочерний узел | `father.add(child)` |
| Количество дочерних элементов | `node.len` |
| Доступ к i-му дочернему элементу | `node[i]`, `node[^1]` |
| Читать значение листового узла | `node.strVal` (только для `LiteralNodes`) |
| Проверить тип узла | `node.kind == nkSelect` и т. д. |
| Перехватить синтаксическую ошибку | `try: parseSql(...) except SqlParseError: ...` |
| Сохранить параметры типов (например VARCHAR(255)) | `parseSql(sql, considerTypeParams = true)` |
