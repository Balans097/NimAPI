
# 📘 Документация пакета `db_sqlite`

## 🔍 Введение
`db_sqlite` — высокоуровневая обёртка над библиотекой SQLite для языка программирования Nim. Модуль предоставляет унифицированный API для работы с реляционными данными, поддерживает подстановку параметров, подготовленные выражения, работу с бинарными данными (`BLOB`) и эффективные итераторы для выборки строк.

> **Примечание:** Данный модуль не реализует ORM-механизмы. Для сохранения простоты и максимальной близости к нативному API SQLite, все строки возвращаются как `seq[string]`, а значения `NULL` преобразуются в пустые строки `""`.

## 📦 Установка
Для использования модуля необходимо установить пакет `db_connector`:
```bash
nimble install db_connector
```

## 🧱 Основные типы данных

| Тип | Описание | Назначение |
|-----|----------|------------|
| `DbConn` | `PSqlite3` | Дескриптор активного соединения с базой данных. |
| `Row` | `seq[string]` | Представляет одну строку результата запроса. Все значения приводятся к строковому типу. |
| `InstantRow` | `PStmt` | Лёгкий дескриптор, позволяющий извлекать значения столбцов «по требованию» внутри итератора. |
| `SqlPrepared` | `distinct PStmt` | Идентификатор предварительно скомпилированного SQL-запроса. |
| `DbType` | Структура | Содержит метаданные типа столбца: `kind` (тип), `size`, `name`, `primaryKey` и др. |
| `DbColumns` | `seq[DbType]` | Коллекция метаданных всех столбцов, возвращаемых запросом. |

## ⚙️ Базовый API

### 🔌 Управление соединением
- `open(connection, user, password, database): DbConn` — открывает соединение. Для SQLite используются только параметры `connection` (путь к файлу) и `database` (обычно пустой).
- `close(db: DbConn)` — безопасно закрывает соединение и освобождает ресурсы.

### 📝 Выполнение запросов
- `exec(db, query, args)` — выполняет запрос. При ошибке выбрасывает исключение `DbError`.
- `tryExec(db, query, args): bool` — выполняет запрос, возвращает `true` при успехе, `false` при ошибке.
- `execAffectedRows(db, query, args): int64` — выполняет запрос и возвращает количество изменённых строк (для `UPDATE`/`DELETE`).

### 🔍 Выборка данных
- `fastRows(db, query, args): iterator Row` — быстрый итератор. **Внимание:** прерывание цикла (`break`) может заблокировать следующее выполнение запросов до закрытия соединения.
- `rows(db, query, args): iterator Row` — безопасный итератор (медленнее, но позволяет использовать `break`).
- `instantRows(db, query, args): iterator InstantRow` — возвращает дескриптор строки для ленивого доступа к столбцам.
- `getRow(db, query, args): Row` — возвращает первую строку результата. Если строк нет, возвращает `Row` с пустыми строками.
- `getAllRows(db, query, args): seq[Row]` — возвращает все строки результата в виде последовательности.
- `getValue(db, query, args): string` — возвращает значение первого столбца первой строки.

### 🛠 Подготовленные выражения
- `prepare(db, query: string): SqlPrepared` — компилирует SQL-запрос для многократного использования.
- `finalize(stmt: SqlPrepared)` — освобождает ресурсы подготовленного запроса.
- `bindParam(ps, paramIdx, val)` — привязывает значение к конкретному индексу параметра (`?`).
- `bindNull(ps, paramIdx)` — привязывает `NULL`.
- `bindParams(ps, args...)` — макрос для массовой привязки аргументов. Автоматически определяет типы.

### 📊 Вставка и возврат идентификаторов
- `tryInsertID(db, query, args): int64` — выполняет `INSERT` и возвращает `rowid`. При ошибке возвращает `-1`.
- `insertID(db, query, args): int64` — аналогично, но выбрасывает `DbError` при ошибке.

### 🛡 Утилиты
- `dbQuote(s: string): string` — экранирует одинарные кавычки для безопасной вставки в `VARCHAR`.
- `dbError(db: DbConn)` — явно вызывает исключение с текстом ошибки SQLite.
- `setEncoding(db, encoding: string): bool` — устанавливает кодировку базы (только до создания первой таблицы).

---

## 💻 Примеры использования

### 1. Подключение, создание таблицы и вставка данных
```nim
import
  db_connector/db_sqlite
  std/math
  std/assertions

let
  # Открываем соединение. Для SQLite достаточно имени файла.
  db = open("app_data.db", "", "", "")

# Удаляем таблицу, если она существует
exec(db, sql"DROP TABLE IF EXISTS users")

# Создаём таблицу
exec(db, sql"""
  CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    age INT,
    balance DECIMAL(10, 2)
  )
""")

# Вставка одной записи с автоподстановкой параметров
exec(db, sql"INSERT INTO users (username, age, balance) VALUES (?, ?, ?)",
     "alice", 30, 150.50)

# Пакетная вставка в транзакции
exec(db, sql"BEGIN")
for i in 1 .. 10:
  let
    name = "user_" & $i
    age = i * 5
    bal = sqrt(float(i))
  exec(db, sql"INSERT INTO users (username, age, balance) VALUES (?, ?, ?)",
       name, age, bal)
exec(db, sql"COMMIT")

# Получаем ID последней вставленной строки
let
  lastId = tryInsertID(db, sql"INSERT INTO users (username, age, balance) VALUES (?, ?, ?)",
                       "bob", 25, 0.0)

# Проверяем результат
assert(lastId > 0, "Ошибка вставки: вернулся отрицательный ID")

# Закрываем соединение
close(db)
```

### 2. Выборка данных и работа с `InstantRow`
```nim
import
  db_connector/db_sqlite
  std/strutils
  std/assertions

let
  db = open("app_data.db", "", "", "")

echo("--- Обычный итератор (rows) ---")
for row in rows(db, sql"SELECT id, username, age FROM users WHERE age > ?" > 10):
  # Доступ по индексу, преобразование типов через стандартные функции
  let
    uid = parseInt(row[0])
    uname = row[1]
    uage = parseInt(row[2])
  echo("ID: " & $uid & " | Имя: " & uname & " | Возраст: " & $uage)

echo("\n--- Ленивый итератор (instantRows) ---")
for row in instantRows(db, sql"SELECT username, balance FROM users"):
  # row[col] возвращает строку напрямую из памяти SQLite
  let
    name = row[0]
    money = row[1]
  # Пропускаем строки с нулевым балансом
  if parseFloat(money) > 0.0:
    echo(name & " имеет баланс: " & money)

echo("\n--- Получение одного значения ---")
let
  firstUser = getValue(db, sql"SELECT username FROM users ORDER BY id LIMIT 1")
echo("Первый пользователь: " & firstUser)

echo("\n--- Получение метаданных столбцов ---")
var
  cols: DbColumns
for _ in instantRows(db, cols, sql"SELECT * FROM users LIMIT 0"):
  discard # Итератор не требуется, нам нужна только инициализация cols

for col in cols:
  echo("Столбец: " & col.name & " | Тип: " & col.typ.name &
       " | Первичный ключ: " & $col.primaryKey)

close(db)
```

### 3. Работа с бинарными данными (`BLOB`) и подготовленные выражения
```nim
import
  db_connector/db_sqlite
  std/random
  std/assertions

let
  db = open("blob_test.db", "", "", "")

# Создаём таблицу с полем BLOB
exec(db, sql"DROP TABLE IF EXISTS media")
exec(db, sql"CREATE TABLE media (id INTEGER PRIMARY KEY, data BLOB)")

# Генерируем тестовые бинарные данные (массив байтов)
var
  originalData = newSeq[byte](128)
randomize()
for i in 0 .. len(originalData) - 1:
  originalData[i] = byte(rand(255))

# Подготавливаем запрос для вставки
let
  insertStmt = prepare(db, "INSERT INTO media (id, data) VALUES (?, ?)")

# Привязываем параметры вручную
bindParam(insertStmt, 1, 1'i32)
bindParam(insertStmt, 2, originalData, true) # true = SQLite скопирует данные

# Выполняем
exec(db, insertStmt)
finalize(insertStmt)

# Подготавливаем запрос для чтения
let
  selectStmt = prepare(db, "SELECT data FROM media WHERE id = ?")
bindParam(selectStmt, 1, 1'i32)

# Извлекаем BLOB
var
  retrievedData = getValue(db, selectStmt)

# Проверяем идентичность данных
assert(len(retrievedData) == len(originalData), "Размеры данных не совпадают")
for i in 0 .. len(originalData) - 1:
  assert(retrievedData[i] == originalData[i], "Данные повреждены на индексе " & $i)

echo("Бинарные данные успешно сохранены и прочитаны.")

finalize(selectStmt)
close(db)
```

---

## 📖 Подробные пояснения к архитектуре

### Почему `seq[string]` для строк?
В исходном коде явно указано, что `Row` реализован как `seq[string]`. Это сделано по трём причинам:
1. **Нативность:** Ближе всего к базовому API SQLite (`char**`), минимизируя накладные расходы на преобразование типов.
2. **Абстракция:** База данных может возвращать десятки типов (`INTEGER`, `REAL`, `BLOB`, `TEXT`, специфичные расширения). Приведение всего к строке скрывает эту сложность от пользователя.
3. **Удобство проброса:** Если данные нужно просто логировать, передать в другой запрос или сериализовать в JSON, строковое представление является универсальным форматом.

### Как работает `InstantRow`?
`InstantRow` не копирует данные в память Nim при каждой итерации. Вместо этого он хранит указатель на внутренний буфер `sqlite3_stmt`. Метод `row[col]` вызывает `column_text(stmt, col)` напрямую. Это экономит память и ускоряет чтение огромных выборок, но требует, чтобы доступ к данным происходил **только внутри тела цикла**. После завершения итерации буфер освобождается, и попытка обратиться к `row` приведёт к неопределённому поведению.

### Подготовленные выражения (`SqlPrepared`)
Используются для многократного выполнения одинаковых запросов с разными параметрами.
```nim
let
  stmt = prepare(db, "UPDATE items SET price = ? WHERE id = ?")
```
При каждом вызове `exec(db, stmt, ...)` или `bindParam(stmt, idx, val)` происходит сброс состояния (`reset`) и очистка привязок (`clear_bindings`), что гарантирует изоляцию операций. После завершения работы **обязательно** вызывайте `finalize(stmt)`, иначе произойдёт утечка памяти.

### Безопасность и экранирование
Модуль поддерживает безопасную подстановку параметров через `?`:
```nim
# ✅ Безопасно
exec(db, sql"SELECT * FROM users WHERE name = ?", "O'Connor")

# ❌ Опасно (риск SQL-инъекций)
exec(db, sql"SELECT * FROM users WHERE name = '" & userInput & "'")
```
Если вам всё же необходимо вручную вставить строку в запрос, используйте `dbQuote(str)`:
```nim
let
  safeName = dbQuote(userInput)
exec(db, sql"SELECT * FROM users WHERE name = " & safeName)
```

---

## ⚠️ Важные замечания

1. **Прерывание `fastRows`:** Никогда не используйте `break` внутри `fastRows`. SQLite оставляет транзакцию чтения открытой, и следующий запрос завершится ошибкой `unable to close due to ...`. Используйте `rows` для безопасного прерывания.
2. **Кодировка:** `setEncoding` должен вызываться **до** создания первой таблицы. После инициализации базы данных изменение кодировки игнорируется движком SQLite.
3. **Типы параметров:** При использовании `bindParam` или `bindParams` макрос автоматически выбирает перегруженную функцию (`int32`, `int64`, `float64`, `string`, `openArray[byte]`). Для `NULL` используйте макрос `bindParams(stmt, nil)` или явный `bindNull(stmt, idx)`.
4. **Ошибки:** Все процедуры с приставкой `try` возвращают статус успеха/ошибки без выброса исключений. Остальные процедуры при ошибке вызывают `dbError(db)`, который выбрасывает `ref DbError` с сообщением от `sqlite3.errmsg(db)`.

## 🏁 Заключение
Пакет `db_sqlite` предоставляет баланс между производительностью и простотой использования. Строгое разделение на быстрые/безопасные итераторы, поддержка бинарных данных и подготовленных выражений делает его пригодным как для разработки сценариев утилит, так и для высоконагруженных серверных приложений на Nim.



