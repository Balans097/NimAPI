
# 📘 Документация пакета `db_postgres`

## 🔍 Введение
`db_postgres` — высокоуровневая обёртка над нативной библиотекой `libpq` для СУБД PostgreSQL в языке Nim. Модуль предоставляет унифицированный API для работы с реляционными данными, поддерживает параметризованные запросы (`?` для обычных и `$1, $2...` для подготовленных), безопасные и быстрые итераторы, а также автоматическое приведение типов к строковому представлению.

> **Примечание:** Модуль не реализует ORM-механизмы. Для сохранения максимальной близости к API `libpq` и упрощения сериализации, все строки возвращаются как `seq[string]`, а значения `NULL` преобразуются в пустые строки `""`.

## 📦 Установка
Для использования модуля необходимо установить пакет `db_connector`:
```bash
nimble install db_connector
```

## 🧱 Основные типы данных

| Тип | Описание | Назначение |
|-----|----------|------------|
| `DbConn` | `PPGconn` | Дескриптор активного соединения с сервером PostgreSQL. |
| `Row` | `seq[string]` | Представляет одну строку результата запроса. Все значения приводятся к строковому типу. |
| `InstantRow` | `object { res: PPGresult }` | Лёгкий дескриптор, содержащий указатель на внутренний буфер результата. Позволяет извлекать столбцы «по требованию» без аллокаций. |
| `SqlPrepared` | `distinct string` | Строковый идентификатор подготовленного на сервере SQL-запроса. |
| `DbType` | Структура | Содержит метаданные типа столбца: `kind` (категория типа), `size`, `name` (имя типа в БД), `tableName` и др. |
| `DbColumns` | `seq[DbType]` | Коллекция метаданных всех столбцов, возвращаемых запросом. |

## ⚙️ Базовый API

### 🔌 Управление соединением
- `open(connection, user, password, database): DbConn` — открывает соединение. Поддерживает стандартные строки подключения (ключ-значение) в параметре `database`.
- `close(db: DbConn)` — безопасно закрывает соединение и освобождает ресурсы `libpq`.
- `setEncoding(db: DbConn, encoding: string): bool` — устанавливает кодировку клиентского соединения.

### 📝 Выполнение запросов
- `exec(db, query, args)` — выполняет запрос. При ошибке выбрасывает `DbError`.
- `tryExec(db, query, args): bool` — выполняет запрос, возвращает `true` при успехе.
- `execAffectedRows(db, query, args): int64` — выполняет запрос и возвращает количество изменённых строк (для `UPDATE`/`DELETE`/`INSERT ... RETURNING`).

### 🔍 Выборка данных
- `fastRows(db, query, args): iterator Row` — быстрый итератор. **В отличие от SQLite, для PostgreSQL он безопасен** и не блокирует соединение при прерывании.
- `rows(db, query, args): iterator Row` — безопасный итератор (аллоцирует все строки в память перед итерацией).
- `instantRows(db, query, args): iterator InstantRow` — возвращает дескриптор строки для ленивого доступа.
- `instantRows(db, columns, query, args): iterator InstantRow` — дополнительно заполняет `columns` метаданными о типах столбцов.
- `getRow(db, query, args): Row` — возвращает первую строку результата.
- `getAllRows(db, query, args): seq[Row]` — возвращает все строки в виде последовательности.
- `getValue(db, query, args): string` — возвращает значение первого столбца первой строки.

### 🛠 Подготовленные выражения
- `prepare(db, stmtName, query, nParams): SqlPrepared` — компилирует запрос на сервере. Параметры указываются как `$1, $2...`.
- `exec(db, stmtName, args)` / `tryExec(db, stmtName, args): bool` — выполнение подготовленного запроса.

### 📊 Вставка и возврат идентификаторов
- `tryInsertID(db, query, args): int64` — выполняет `INSERT` и возвращает `rowid`. Автоматически добавляет `RETURNING id` к запросу. Работает только если первичный ключ называется `id`.
- `insertID(db, query, args): int64` — аналогично, но выбрасывает `DbError` при ошибке.
- `tryInsert(db, query, pkName, args): int64` — позволяет указать произвольное имя первичного ключа (`pkName`).
- `insert(db, query, pkName, args): int64` — аналогично с выбросом исключения.

### 🛡 Утилиты
- `dbQuote(s: string): string` — экранирует одинарные кавычки (`'` → `''`) и нуль-байты (`\0` → `\\0`).
- `dbError(db: DbConn)` — явно вызывает исключение с текстом ошибки от `pqErrorMessage`.

---

## 💻 Примеры использования

### 1. Подключение, создание таблицы и вставка данных
```nim
import
  db_connector/db_postgres
  std/math
  std/assertions

let
  # Открываем соединение к локальному серверу
  db = open("localhost", "postgres", "secret", "company_db")

# Удаляем таблицу, если она существует
exec(db, sql"DROP TABLE IF EXISTS employees")

# Создаём таблицу с типичной структурой
exec(db, sql"""
  CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    department VARCHAR(50),
    salary NUMERIC(10, 2)
  )
""")

# Вставка одной записи с автоподстановкой параметров через ?
exec(db, sql"INSERT INTO employees (full_name, department, salary) VALUES (?, ?, ?)",
     "Alice Smith", "Engineering", 95000.00)

# Пакетная вставка внутри транзакции
exec(db, sql"BEGIN")
for i in 1 .. 5:
  let
    empName = "Employee_" & $i
    dept = "Support"
    pay = 50000.0 + (float(i) * 2000.0)
  exec(db, sql"INSERT INTO employees (full_name, department, salary) VALUES (?, ?, ?)",
       empName, dept, pay)
exec(db, sql"COMMIT")

# Получаем ID последней вставленной строки (модуль сам добавит RETURNING id)
let
  lastId = tryInsertID(db, sql"INSERT INTO employees (full_name, department, salary) VALUES (?, ?, ?)",
                       "Bob Jones", "HR", 65000.00)

# Проверяем результат
assert(lastId > 0, "Ошибка вставки: возвращён отрицательный или нулевой ID")

# Закрываем соединение
close(db)
```

### 2. Выборка данных, `InstantRow` и метаданные столбцов
```nim
import
  db_connector/db_postgres
  std/strutils
  std/assertions

let
  db = open("localhost", "postgres", "secret", "company_db")

echo("--- Стандартный итератор (rows) ---")
for row in rows(db, sql"SELECT id, full_name, salary FROM employees WHERE salary > ?", 70000.00):
  let
    uid = parseInt(row[0])
    name = row[1]
    pay = parseFloat(row[2])
  echo("ID: " & $uid & " | Сотрудник: " & name & " | Зарплата: " & $pay)

echo("\n--- Ленивый итератор (instantRows) ---")
for row in instantRows(db, sql"SELECT full_name, department FROM employees"):
  # Доступ к столбцам по индексу без копирования данных в память Nim
  let
    name = row[0]
    dept = row[1]
  if len(dept) > 0:
    echo(name & " работает в отделе: " & dept)

echo("\n--- Получение метаданных столбцов ---")
var
  cols: DbColumns
# Итератор инициализирует cols при первом вызове
for _ in instantRows(db, cols, sql"SELECT * FROM employees LIMIT 0"):
  discard

for col in cols:
  echo("Столбец: " & col.name & " | Тип PostgreSQL: " & col.typ.name &
       " | Таблица: " & col.tableName)

echo("\n--- Получение одиночного значения ---")
let
  firstEmp = getValue(db, sql"SELECT full_name FROM employees ORDER BY id LIMIT 1")
echo("Первый сотрудник: " & firstEmp)

close(db)
```

### 3. Подготовленные выражения и работа с произвольным первичным ключом
```nim
import
  db_connector/db_postgres
  std/assertions

let
  # Подключение через Unix-сокет (значительно повышает производительность на Linux)
  db = open("/var/run/postgresql", "postgres", "secret", "company_db")

# Подготовка именованного запроса.
# Параметры указываются как $1, $2, $3 (стандарт PostgreSQL)
let
  updateStmt = prepare(db, "update_salary_by_id", sql"""
    UPDATE employees SET salary = $1 WHERE id = $2
  """, 2)

# Выполнение подготовленного запроса с подстановкой аргументов
exec(db, updateStmt, "115000.00", "1")

# Проверка обновления
let
  newSalary = getValue(db, sql"SELECT salary FROM employees WHERE id = 1")
assert(newSalary == "115000.00", "Зарплата не была обновлена корректно")

# Пример с произвольным именем первичного ключа (не 'id')
let
  customPkId = tryInsert(db, sql"INSERT INTO employees (full_name, department, salary) VALUES (?, ?, ?)",
                         "id", "Charlie Brown", "Finance", 72000.00)
if customPkId < 0:
  echo("Вставка не удалась или таблица не имеет PK 'id'")

# Подсчёт строк, затронутых последним запросом
let
  affectedCount = execAffectedRows(db, sql"UPDATE employees SET department = 'Executive' WHERE salary > ?", 100000.00)
echo("Обновлено записей: " & $affectedCount)

# Явное освобождение подготовленного запроса на стороне сервера
exec(db, sql"DEALLOCATE update_salary_by_id")

close(db)
```

---

## 📖 Подробные пояснения к архитектуре

### Почему `seq[string]` для строк?
В исходном коде явно указано, что `Row` реализован как `seq[string]`. Это сделано по трём причинам:
1. **Нативность:** Ближе всего к базовому API `libpq` (`PQgetvalue` возвращает `char*`), минимизируя накладные расходы на преобразование типов на стороне клиента.
2. **Абстракция:** PostgreSQL поддерживает десятки специфичных типов (`JSONB`, `UUID`, `INET`, `RANGE`, `NUMERIC` с произвольной точностью). Приведение всего к строке скрывает эту сложность от пользователя.
3. **Удобство проброса:** Если данные нужно просто логировать, передать в другой запрос или сериализовать в JSON, строковое представление является универсальным форматом.

### Как работает `InstantRow`?
`InstantRow` не копирует данные в память Nim при каждой итерации. Вместо этого он хранит указатель на внутренний буфер `PGresult`. Операция `row[col]` вызывает `PQgetvalue(res, 0, col)` напрямую. Это экономит память и ускоряет чтение огромных выборок, но требует, чтобы доступ к данным происходил **только внутри тела цикла**. После завершения итерации или очистки `PGresult` буфер освобождается, и попытка обратиться к `row` приведёт к неопределённому поведению.

### Подготовленные выражения (`SqlPrepared`)
В PostgreSQL подготовленные выражения хранятся на стороне сервера и идентифицируются строковым именем (`distinct string`). При вызове `prepare(db, "stmt_name", sql"$1, $2", 2)` сервер кэширует план выполнения.
Каждый вызов `exec(db, stmtName, args)` автоматически:
1. Сбрасывает состояние предыдущего выполнения.
2. Подставляет аргументы в бинарном или текстовом формате.
3. Выполняет запрос через `PQexecPrepared`.
Для явного удаления запроса с сервера используется стандартный SQL: `exec(db, sql"DEALLOCATE stmt_name")`.

### Безопасность и экранирование
Модуль поддерживает два механизма параметризации:
- **`SqlQuery` (`?`)**: Автоматически преобразуется в безопасный формат через `dbFormat` и `dbQuote`.
- **`SqlPrepared` (`$1, $2...`)**: Использует нативный протокол PostgreSQL, полностью исключая риск инъекций и снижая нагрузку на парсер БД.

Если вам всё же необходимо вручную вставить строку в динамический запрос, используйте `dbQuote(str)`:
```nim
let
  safeName = dbQuote(userInput)
exec(db, sql"SELECT * FROM employees WHERE full_name = " & safeName)
```
Функция `dbQuote` корректно обрабатывает одинарные кавычки (`'` → `''`) и нуль-терминаторы (`\0` → `\\0`), что важно для бинарных данных в текстовых полях.

---

## ⚠️ Важные замечания

1. **Безопасность `fastRows` для PostgreSQL:** В отличие от SQLite, итератор `fastRows` для PostgreSQL **безопасен**. Движок `libpq` использует режим `PGRES_SINGLE_TUPLE`, который корректно обрабатывает прерывание цикла (`break`) без блокировки соединения.
2. **Кодировка:** `setEncoding` вызывает `PQsetClientEncoding`. Изменение применяется только к текущему соединению и не влияет на существующие данные в таблицах.
3. **Возврат ID (`tryInsertID` / `insertID`):** Модуль автоматически добавляет `RETURNING id` к запросу. Это работает **только** если столбец первичного ключа называется `id`. Для произвольных имён используйте `tryInsert(db, query, "my_pk_name", args)`.
4. **Типы параметров в `SqlPrepared`:** При подготовке запроса сервер ожидает строгого соответствия типов. Все аргументы передаются как строки и автоматически преобразуются сервером PostgreSQL согласно целевому типу столбца. Для явного приведения используйте SQL-касты (`$1::int`).
5. **Ошибки:** Все процедуры с приставкой `try` возвращают статус успеха/ошибки или `-1` без выброса исключений. Остальные процедуры при ошибке вызывают `dbError(db)`, который выбрасывает `ref DbError` с сообщением от `PQerrorMessage`.
6. **Unix-сокеты:** Для максимальной производительности на Linux рекомендуется передавать путь к сокету в параметр `connection` вместо `localhost`. Это позволяет обойти сетевой стек TCP и ускорить операции на 30%–175%.

## 🏁 Заключение
Пакет `db_postgres` предоставляет сбалансированный, высокопроизводительный интерфейс к PostgreSQL. Строгое разделение на быстрые/безопасные итераторы, поддержка серверных подготовленных выражений, автоматическая работа с метаданными столбцов и встроенная оптимизация под Unix-сокеты делают его пригодным как для скриптов администрирования, так и для высоконагруженных веб-приложений на Nim.



