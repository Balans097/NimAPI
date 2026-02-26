# `pcre.nim` — справочник модуля

> **Модуль:** `pcre` — привязка Nim к C-библиотеке PCRE (Perl Compatible Regular Expressions)  
> **Режим привязки:** Динамическая библиотека по умолчанию (`libpcre.so.3/.1`, `libpcre.dylib`, `pcre64.dll`/`pcre32.dll`). Используйте `-d:usePcreHeader`, чтобы вместо этого привязываться к `<pcre.h>` на этапе компиляции.  
> **Аудитория:** Это **низкоуровневая прямая C-API привязка**. Для обычной работы с регулярными выражениями в Nim лучше подходит высокоуровневый модуль `std/re` (который использует эту привязку внутри). Используйте `pcre.nim` напрямую, когда нужен JIT-компилятор, DFA-матчинг, коллбэки, пользовательские таблицы символов или ограничения ресурсов на уровне отдельного сопоставления.

---

## Архитектура модуля

```
Константы версии        PCRE_MAJOR, PCRE_MINOR, PCRE_DATE
Флаги опций             CASELESS, MULTILINE, UTF8, PARTIAL_SOFT, …
Коды ошибок             ERROR_NOMATCH, ERROR_BADUTF8, …
INFO / CONFIG селекторы INFO_CAPTURECOUNT, CONFIG_JIT, …
STUDY / EXTRA флаги     STUDY_JIT_COMPILE, EXTRA_MATCH_LIMIT, …
Непрозрачные типы       Pcre, JitStack
Структуры данных        ExtraData, CalloutBlock
Тип коллбэка            JitCallback
Привязки C-функций      compile, exec, study, jit_exec, fullinfo, …
```

Типичный поток работы:

```
compile() ──► [опц.] study() ──► exec() / jit_exec() / dfa_exec()
                                         │
                            fullinfo() ◄─┤  (запрос метаданных шаблона)
              get_substring() / get_named_substring() ◄─┘  (извлечение захватов)
```

---

## Константы версии

```nim
PCRE_MAJOR*     = 8
PCRE_MINOR*     = 36
PCRE_PRERELEASE* = true
PCRE_DATE*      = "2014-09-26"
```

Определяют версию заголовочного файла PCRE, с которым была написана привязка. Это константы времени компиляции — они **не** меняются в соответствии с версией PCRE-библиотеки, установленной на целевой машине. Для получения версии рантаймовой библиотеки вызовите `version()`.

```nim
echo PCRE_MAJOR, ".", PCRE_MINOR  # "8.36" — версия заголовка, зашитая при компиляции
echo version()                     # например, "8.45 2021-06-15" — реально установленная
```

---

## Константы флагов опций

Все флаги — это `int`-маски, комбинируемые через `or`. PCRE помечает каждый флаг тегами контекстов, где он допустим:

| Тег | Применяется в |
|---|---|
| `C1` | Только `compile()` — сохраняется в скомпилированном шаблоне |
| `C2` | Только поведение `compile()` — не сохраняется |
| `C3` | `compile()`, `exec()`, `dfa_exec()` |
| `C4` | `compile()`, `exec()`, `dfa_exec()`, `study()` |
| `C5` | `compile()`, `exec()`, `study()` |
| `E` | `exec()` во время сопоставления |
| `D` | `dfa_exec()` во время сопоставления |
| `J` | Совместим с JIT-исполнением |

---

### Флаги поведения шаблона (время компиляции)

---

#### `CASELESS` — `0x00000001` · C1

Делает весь шаблон регистронезависимым. ASCII-буквы совпадают в любом регистре. В сочетании с `UTF8` и `UCP` применяются правила Unicode-свёртки регистра.

```nim
# Совпадает с "Hello", "HELLO", "hElLo", …
let re = compile("hello", CASELESS, addr err, addr errOff, nil)
```

---

#### `MULTILINE` — `0x00000002` · C1

Заставляет `^` и `$` совпадать с началом и концом **каждой строки** (разделённых `\n`), а не только с границами всей строки-субъекта. Аналог `re.MULTILINE` в Python.

```nim
# Совпадает с "foo" в начале любой строки, а не только в самом начале
let re = compile("^foo", MULTILINE, addr err, addr errOff, nil)
```

---

#### `DOTALL` — `0x00000004` · C1

Делает `.` совпадающим с **любым** символом, включая `\n`. По умолчанию `.` не переходит границы строк. В других диалектах регулярных выражений этот режим часто называют «однострочным» (single-line mode).

```nim
# Совпадает с "a\nb" — точка пересекает перенос строки
let re = compile("a.b", DOTALL, addr err, addr errOff, nil)
```

---

#### `EXTENDED` — `0x00000008` · C1

Незаключённые в кавычки пробелы и комментарии `# …` в тексте шаблона игнорируются. Позволяет писать читаемые, самодокументирующиеся шаблоны.

```nim
let pat = r"""
  \d{4}   # год
  -
  \d{2}   # месяц
  -
  \d{2}   # день
"""
let re = compile(pat, EXTENDED, addr err, addr errOff, nil)
```

---

#### `ANCHORED` — `0x00000010` · C4, E, D

Принудительно привязывает сопоставление к `startoffset` (или к самому началу субъекта, если `startoffset` равен 0). Эквивалентно добавлению `\A` в начало шаблона.

---

#### `DOLLAR_ENDONLY` — `0x00000020` · C2

Заставляет `$` совпадать только в абсолютном конце строки-субъекта, игнорируя завершающий `\n`. Без этого флага `$` также совпадает непосредственно перед финальным переводом строки.

---

#### `EXTRA` — `0x00000040` · C1

Включает более строгую проверку синтаксиса шаблона: нераспознанные управляющие последовательности типа `\q` становятся ошибками компиляции, а не проходят «насквозь». Специфичный для PCRE режим строгости сверх совместимости с Perl.

---

#### `UNGREEDY` — `0x00000200` · C1

Глобально инвертирует жадность кванторов: `*`, `+`, `?` по умолчанию становятся нежадными (ленивыми); `*?`, `+?`, `??` — жадными. Экономит необходимость добавлять `?` к каждому квантору в шаблонах, где нежадность является нормой.

---

#### `NO_AUTO_CAPTURE` — `0x00001000` · C1

Отключает автоматическую нумерацию простых групп `(…)` — они становятся незахватывающими. Именованные группы `(?P<n>…)` / `(?<n>…)` по-прежнему работают. Уменьшает требуемый размер вектора смещений.

---

#### `AUTO_CALLOUT` — `0x00004000` · C1

Вставляет автоматические точки коллбэка (номер 255) перед каждым элементом шаблона. Используется для пошаговой отладки выполнения шаблона через коллбэк-функции.

---

#### `FIRSTLINE` — `0x00040000` · C3

Требует, чтобы сопоставление начиналось в первой строке субъекта. Движок не будет пробовать сопоставления, начинающиеся после первого символа перевода строки.

---

#### `DUPNAMES` — `0x00080000` · C1

Разрешает нескольким захватывающим группам иметь одинаковое имя. Без этого флага дублирующиеся имена — ошибка компиляции. Полезно в чередующихся шаблонах, где логически эквивалентные группы появляются в разных ветках.

```nim
# Обе ветки называют свою группу "word"
let re = compile(r"(?P<word>\w+)|(?P<word>\d+)", DUPNAMES, addr err, addr errOff, nil)
```

---

#### `JAVASCRIPT_COMPAT` — `0x02000000` · C5

Подстраивает ряд поведений под семантику регулярных выражений JavaScript — например, синтаксис `\u{HHHH}` и немного другие правила для пустых совпадений.

---

### Флаги режима UTF

```nim
UTF8*  = 0x00000800  # C4 — субъект в UTF-8
UTF16* = 0x00000800  # C4 — синоним для Pcre16
UTF32* = 0x00000800  # C4 — синоним для Pcre32
```

Все три константы имеют одинаковое значение бита. Активная кодировка определяется используемым непрозрачным типом (`Pcre` для UTF-8, `Pcre16`, `Pcre32`). При активации символьные классы, `.`, `\w` и т.д. работают над кодовыми точками Unicode, а не над сырыми байтами.

```nim
NO_UTF8_CHECK*  = 0x00002000  # C1 E D J
NO_UTF16_CHECK* = 0x00002000  # синоним
NO_UTF32_CHECK* = 0x00002000  # синоним
```

Пропускает проверку корректности UTF в строке-субъекте. Используйте только когда гарантированно знаете, что входные данные валидны — некорректный ввод с этим флагом может вызвать неопределённое поведение.

---

#### `UCP` — `0x20000000` · C3

Заставляет `\w`, `\d`, `\s`, `\b` и POSIX-классы (`[:alpha:]` и др.) использовать данные Unicode-свойств вместо ASCII-таблиц. Требует активного UTF-режима. Без `UCP` `\w` совпадает только с `[a-zA-Z0-9_]` даже в UTF-8 режиме.

---

### Флаги времени сопоставления (E / D / J)

Передаются в `exec()`, `dfa_exec()` или `jit_exec()` для настройки поведения конкретного вызова.

---

#### `NOTBOL` — `0x00000080` · E D J

Сообщает движку, что начало `subject` **не** является началом строки. `^` не будет совпадать с позицией 0. Необходим для корректного сопоставления при поиске внутри подстроки более длинного текста.

---

#### `NOTEOL` — `0x00000100` · E D J

Сообщает движку, что конец `subject` **не** является концом строки. `$` не будет совпадать с финальной позицией. Зеркало `NOTBOL` для якоря конца строки.

---

#### `NOTEMPTY` — `0x00000400` · E D J

Отклоняет совпадения нулевой длины. Предотвращает бесконечные циклы при итеративном поиске всех совпадений шаблонами, способными совпадать с пустой строкой (например, `a*`).

---

#### `NOTEMPTY_ATSTART` — `0x10000000` · E D J

Как `NOTEMPTY`, но отклоняет пустое совпадение только в позиции `startoffset`, а не везде. Пустые совпадения в других позициях по-прежнему допустимы. Полезен в итеративных циклах сопоставления.

---

#### `PARTIAL_SOFT` / `PARTIAL` — `0x00008000` · E D J

Включает **мягкое частичное сопоставление**. Если шаблон поглощает весь субъект без полного успеха, но мог бы успешно совпасть с большим вводом — движок возвращает `ERROR_PARTIAL` вместо `ERROR_NOMATCH`. Полное совпадение всегда имеет приоритет над частичным.

```nim
# "20" — начало потенциального совпадения с r"\d{4}-\d{2}"?
let rc = exec(re, nil, "20", 2, 0, PARTIAL_SOFT, addr ov[0], 30)
if rc == ERROR_PARTIAL:
  echo "Нужно больше данных для решения"
```

---

#### `PARTIAL_HARD` — `0x08000000` · E D J

Как `PARTIAL_SOFT`, но частичное совпадение имеет приоритет над полным. Используйте, когда хотите обнаружить, может ли шаблон *продолжиться* за пределами конца субъекта — полезно для потоковых парсеров.

---

#### `NO_START_OPTIMIZE` / `NO_START_OPTIMISE` — `0x04000000` · C2 E D

Отключает оптимизации стартовой позиции в PCRE (первосимвольная битовая карта, определение привязки). Обычно никогда не нужно, но гарантирует срабатывание коллбэков даже в позициях, которые движок иначе бы пропустил.

---

### Флаги переноса строки — C3 E D

Определяют, какая последовательность байт считается переводом строки. Влияет на `^`, `$`, `.` и `\R`.

| Флаг | Последовательность перевода строки |
|---|---|
| `NEWLINE_CR` | Только `\r` |
| `NEWLINE_LF` | Только `\n` (умолчание библиотеки) |
| `NEWLINE_CRLF` | Только последовательность `\r\n` |
| `NEWLINE_ANY` | Любой Unicode-перевод строки |
| `NEWLINE_ANYCRLF` | `\r`, `\n` или `\r\n` |

---

### Флаги последовательности `\R` — C3 E D

| Флаг | Что совпадает с `\R` |
|---|---|
| `BSR_ANYCRLF` | `\r`, `\n` или `\r\n` |
| `BSR_UNICODE` | Все Unicode-переводы строки |

---

### Пары с совпадающими битами

Некоторые пары флагов делят один бит. PCRE различает их по контексту (компиляция vs. выполнение):

```nim
NEVER_UTF*       = 0x00010000  # C1: запретить UTF-режим в compile()
DFA_SHORTEST*    = 0x00010000  # D:  DFA возвращает только кратчайшее совпадение

NO_AUTO_POSSESS* = 0x00020000  # C1: отключить авто-присвоение
DFA_RESTART*     = 0x00020000  # D:  возобновить DFA после частичного совпадения
```

---

## Коды ошибок

Отрицательные возвращаемые значения `exec()`, `dfa_exec()`, `jit_exec()`, `compile()` и информационных функций. Неотрицательные значения `exec()` обозначают успех (количество найденных захватывающих подстрок).

| Константа | Значение | Когда возникает |
|---|---|---|
| `ERROR_NOMATCH` | -1 | Шаблон не совпал |
| `ERROR_NULL` | -2 | Передан `nil`-указатель там, где он недопустим |
| `ERROR_BADOPTION` | -3 | Установлен нераспознанный бит опции |
| `ERROR_BADMAGIC` | -4 | Скомпилированный шаблон имеет неверное магическое число — несоответствие версий? |
| `ERROR_UNKNOWN_OPCODE` | -5 | Неизвестный опкод в скомпилированном шаблоне (псевдоним `ERROR_UNKNOWN_NODE`) |
| `ERROR_NOMEMORY` | -6 | Выделение памяти внутри PCRE не удалось |
| `ERROR_NOSUBSTRING` | -7 | Запрошенная номерная группа захвата не существует |
| `ERROR_MATCHLIMIT` | -8 | Превышен лимит `ExtraData.match_limit` |
| `ERROR_CALLOUT` | -9 | Коллбэк-функция вернула ненулевое значение (прерывание) |
| `ERROR_BADUTF8` | -10 | Некорректный UTF-8/16/32 в субъекте (псевдонимы `ERROR_BADUTF16`, `ERROR_BADUTF32`) |
| `ERROR_BADUTF8_OFFSET` | -11 | `startoffset` попадает внутрь многобайтового символа |
| `ERROR_PARTIAL` | -12 | Найдено частичное совпадение (при `PARTIAL_SOFT`/`PARTIAL_HARD`) |
| `ERROR_BADPARTIAL` | -13 | Шаблон не подходит для частичного сопоставления |
| `ERROR_INTERNAL` | -14 | Внутренняя ошибка PCRE — никогда не должна возникать |
| `ERROR_BADCOUNT` | -15 | `ovecsize` не кратно 3 |
| `ERROR_RECURSIONLIMIT` | -21 | Превышен лимит `ExtraData.match_limit_recursion` |
| `ERROR_BADNEWLINE` | -23 | Конфликтующие опции перевода строки |
| `ERROR_BADOFFSET` | -24 | `startoffset` отрицательный или выходит за конец субъекта |
| `ERROR_SHORTUTF8` | -25 | `startoffset` попадает внутрь незавершённого UTF-символа |
| `ERROR_RECURSELOOP` | -26 | Рекурсивный вызов шаблона обнаружил бесконечный цикл |
| `ERROR_JIT_STACKLIMIT` | -27 | Переполнение JIT-стека |
| `ERROR_BADMODE` | -28 | Несоответствие режима 8/16/32-бит между шаблоном и вызовом exec |
| `ERROR_BADLENGTH` | -32 | Передана отрицательная `length` в `exec()` |
| `ERROR_UNSET` | -33 | Запрошенная именованная/номерная подстрока не была задана в этом совпадении |

```nim
let rc = exec(re, nil, subject, subject.len.cint, 0, 0, addr ov[0], 30)
case rc
of ERROR_NOMATCH:    echo "нет совпадения"
of ERROR_PARTIAL:    echo "частичное совпадение — нужно больше данных"
of ERROR_MATCHLIMIT: echo "шаблон слишком сложен для этого ввода"
else:
  if rc < 0: echo "ошибка PCRE: ", rc
  else:      echo "найдено ", rc, " захватывающих подстрок"
```

---

## Коды ошибок валидности UTF

`UTF8_ERR0..UTF8_ERR22`, `UTF16_ERR0..UTF16_ERR4` и `UTF32_ERR0..UTF32_ERR3` — вторичные коды ошибок, заполняемые при сбое проверки UTF-валидности. `ERR0` всегда означает «нет ошибки». Более высокие номера идентифицируют конкретный вид некорректной байтовой последовательности. Точное значение каждого кода описано в документации PCRE.

---

## Константы `INFO_*` — селекторы `fullinfo()`

Передаются в `fullinfo()` как аргумент `what` для запроса конкретного свойства скомпилированного шаблона.

| Константа | Значение | Тип `where` | Что возвращает |
|---|---|---|---|
| `INFO_OPTIONS` | 0 | `ptr cint` | Битовая маска опций времени компиляции |
| `INFO_SIZE` | 1 | `ptr csize_t` | Размер скомпилированного шаблона в байтах |
| `INFO_CAPTURECOUNT` | 2 | `ptr cint` | Количество групп захвата |
| `INFO_BACKREFMAX` | 3 | `ptr cint` | Наибольший номер обратной ссылки |
| `INFO_FIRSTCHAR` / `INFO_FIRSTBYTE` | 4 | `ptr cint` | Значение первого символа, если совпадение привязано к литералу |
| `INFO_FIRSTTABLE` | 5 | `ptr pointer` | Указатель на 256-битную карту первых символов |
| `INFO_LASTLITERAL` | 6 | `ptr cint` | Литерал, обязательно присутствующий в любом совпадении |
| `INFO_NAMEENTRYSIZE` | 7 | `ptr cint` | Размер каждой записи в таблице имён |
| `INFO_NAMECOUNT` | 8 | `ptr cint` | Количество именованных групп захвата |
| `INFO_NAMETABLE` | 9 | `ptr pointer` | Указатель на таблицу именованных групп |
| `INFO_STUDYSIZE` | 10 | `ptr csize_t` | Размер данных изучения |
| `INFO_MINLENGTH` | 15 | `ptr cint` | Минимальная длина совпадения |
| `INFO_JIT` | 16 | `ptr cint` | 1 — JIT-скомпилирован, 0 — нет |
| `INFO_JITSIZE` | 17 | `ptr csize_t` | Размер JIT-скомпилированного кода |
| `INFO_MAXLOOKBEHIND` | 18 | `ptr cint` | Максимальная длина ретроспективной проверки |
| `INFO_MATCHLIMIT` | 23 | `ptr clong` | Лимит совпадений, встроенный через `(*LIMIT_MATCH=N)` |
| `INFO_RECURSIONLIMIT` | 24 | `ptr clong` | Лимит рекурсии, встроенный через `(*LIMIT_RECURSION=N)` |
| `INFO_MATCH_EMPTY` | 25 | `ptr cint` | 1 — шаблон может совпасть с пустой строкой |

```nim
var nCap: cint
discard fullinfo(re, nil, INFO_CAPTURECOUNT, addr nCap)
echo "Групп захвата: ", nCap

var canEmpty: cint
discard fullinfo(re, nil, INFO_MATCH_EMPTY, addr canEmpty)
if canEmpty == 1:
  echo "Внимание: шаблон может совпасть с пустой строкой"
```

---

## Константы `CONFIG_*` — селекторы `config()`

Передаются в `config()` как `what` для запроса конфигурации **библиотеки** — не конкретного шаблона.

| Константа | Значение | Что возвращает |
|---|---|---|
| `CONFIG_UTF8` | 0 | 1 — UTF-8 поддержка скомпилирована |
| `CONFIG_NEWLINE` | 1 | Код символа перевода строки по умолчанию |
| `CONFIG_LINK_SIZE` | 2 | Внутренний размер ссылки (2, 3 или 4) |
| `CONFIG_POSIX_MALLOC_THRESHOLD` | 3 | Порог POSIX malloc |
| `CONFIG_MATCH_LIMIT` | 4 | Лимит совпадений по умолчанию |
| `CONFIG_STACKRECURSE` | 5 | 1 — движок рекурсирует на C-стеке |
| `CONFIG_UNICODE_PROPERTIES` | 6 | 1 — поддержка свойств Unicode (`\p{}`) скомпилирована |
| `CONFIG_MATCH_LIMIT_RECURSION` | 7 | Лимит глубины рекурсии по умолчанию |
| `CONFIG_BSR` | 8 | Поведение `\R` по умолчанию (0 = Unicode, 1 = CR/LF) |
| `CONFIG_JIT` | 9 | 1 — JIT-поддержка скомпилирована |
| `CONFIG_UTF16` | 10 | 1 — UTF-16 поддержка скомпилирована |
| `CONFIG_JITTARGET` | 11 | Строка целевой архитектуры JIT (указатель на `cstring`) |
| `CONFIG_UTF32` | 12 | 1 — UTF-32 поддержка скомпилирована |
| `CONFIG_PARENS_LIMIT` | 13 | Максимальная глубина вложенности скобок |

```nim
var jitAvail: cint
discard config(CONFIG_JIT, addr jitAvail)
if jitAvail == 0:
  echo "JIT в этой сборке PCRE недоступен — study() с STUDY_JIT_COMPILE ничего не даст"
```

---

## Константы `STUDY_*` — флаги опций `study()`

Передаются в `study()` как аргумент `options`.

| Константа | Значение | Эффект |
|---|---|---|
| `STUDY_JIT_COMPILE` | 0x0001 | JIT-компиляция шаблона для полного сопоставления |
| `STUDY_JIT_PARTIAL_SOFT_COMPILE` | 0x0002 | JIT-компиляция для `PARTIAL_SOFT`-сопоставления |
| `STUDY_JIT_PARTIAL_HARD_COMPILE` | 0x0004 | JIT-компиляция для `PARTIAL_HARD`-сопоставления |
| `STUDY_EXTRA_NEEDED` | 0x0008 | Всегда выделять блок `ExtraData`, даже если изучение не дало данных |

Комбинируйте через `or` для получения нескольких JIT-вариантов за один вызов:

```nim
let extra = study(re,
  STUDY_JIT_COMPILE or STUDY_JIT_PARTIAL_SOFT_COMPILE,
  addr studyErr)
```

---

## Константы `EXTRA_*` — битовая маска поля `ExtraData.flags`

Установите соответствующий бит в `ExtraData.flags` для каждого поля, которое вы заполняете, перед передачей `ExtraData` в `exec()`.

| Константа | Значение | Соответствующее поле |
|---|---|---|
| `EXTRA_STUDY_DATA` | 0x0001 | `study_data` — устанавливается автоматически через `study()` |
| `EXTRA_MATCH_LIMIT` | 0x0002 | `match_limit` |
| `EXTRA_CALLOUT_DATA` | 0x0004 | `callout_data` |
| `EXTRA_TABLES` | 0x0008 | `tables` |
| `EXTRA_MATCH_LIMIT_RECURSION` | 0x0010 | `match_limit_recursion` |
| `EXTRA_MARK` | 0x0020 | `mark` (выходное поле) |
| `EXTRA_EXECUTABLE_JIT` | 0x0040 | `executable_jit` — устанавливается автоматически через `study()` с JIT |

```nim
# Защита от катастрофического возврата на ненадёжном вводе:
var extra = ExtraData()
extra.flags = EXTRA_MATCH_LIMIT or EXTRA_MATCH_LIMIT_RECURSION
extra.match_limit            = 100_000
extra.match_limit_recursion  =   5_000
discard exec(re, addr extra, subject, subject.len.cint, 0, 0, addr ov[0], 30)
```

---

## Типы

---

### `Pcre`, `Pcre16`, `Pcre32`

```nim
type
  Pcre*   = object
  Pcre16* = object
  Pcre32* = object
```

Непрозрачные типы нулевого размера, используемые **только как цели указателей** (`ptr Pcre`). Они представляют скомпилированный шаблон регулярного выражения. Вы никогда не инспектируете и не создаёте их напрямую — `compile()` возвращает `ptr Pcre`, все остальные функции его потребляют. `Pcre` — для 8-битных (байтовых или UTF-8) шаблонов; `Pcre16`/`Pcre32` — для 16- и 32-битных вариантов PCRE (их функции в этот файл привязки не включены).

---

### `JitStack`, `JitStack16`, `JitStack32`

```nim
type
  JitStack*   = object
  JitStack16* = object
  JitStack32* = object
```

Непрозрачный дескриптор **JIT-стека выполнения**. JIT-движку нужен отдельный стек вызовов, отличный от системного C-стека. Создаётся через `jit_stack_alloc()`, привязывается к шаблону через `assign_jit_stack()`, освобождается через `jit_stack_free()`.

---

### `ExtraData`

```nim
type
  ExtraData* = object
    flags*:                  clong
    study_data*:             pointer
    match_limit*:            clong
    callout_data*:           pointer
    tables*:                 pointer
    match_limit_recursion*:  clong
    mark*:                   pointer
    executable_jit*:         pointer
```

**Что это:**  
Расширяемый блок параметров, передаваемый функциям сопоставления для дополнительной настройки и получения дополнительного вывода. Каждое поле — опциональное: вы должны установить соответствующий бит `EXTRA_*` в `flags` для каждого используемого поля. Незафлаженные поля игнорируются PCRE.

**Описание полей:**

- **`flags`** — Битовая маска из констант `EXTRA_*`, объявляющая активные поля. Всегда заполняйте перед передачей структуры.
- **`study_data`** — Непрозрачный указатель, заполняемый `study()`. Не изменяйте вручную.
- **`match_limit`** — Максимальное число рекурсивных шагов возврата. Предотвращает катастрофический возврат на патологических входных данных. Флаг: `EXTRA_MATCH_LIMIT`.
- **`callout_data`** — Произвольный пользовательский указатель, передаваемый дословно в `CalloutBlock.callout_data` при вызовах коллбэка. Флаг: `EXTRA_CALLOUT_DATA`.
- **`tables`** — Указатель на локализованные таблицы символов от `maketables()`. Флаг: `EXTRA_TABLES`.
- **`match_limit_recursion`** — Отдельный лимит глубины рекурсивных вызовов (отличается от `match_limit`). Флаг: `EXTRA_MATCH_LIMIT_RECURSION`.
- **`mark`** — *Выходное поле.* После успешного совпадения PCRE записывает сюда указатель на последнее встреченное `(*MARK:name)`. Флаг: `EXTRA_MARK`.
- **`executable_jit`** — Указатель на JIT-скомпилированный код, устанавливается `study()`. Не трогайте.

```nim
# Ограничить рискованный шаблон 50 000 попытками сопоставления:
var extra = ExtraData()
extra.flags = EXTRA_MATCH_LIMIT
extra.match_limit = 50_000
let rc = exec(re, addr extra, subject, subject.len.cint, 0, 0, addr ov[0], 30)
if rc == ERROR_MATCHLIMIT:
  echo "Ввод отклонён: шаблон слишком сложен"
```

---

### `CalloutBlock`

```nim
type
  CalloutBlock* = object
    version*         : cint
    callout_number*  : cint
    offset_vector*   : ptr cint
    subject*         : cstring
    subject_length*  : cint
    start_match*     : cint
    current_position*: cint
    capture_top*     : cint
    capture_last*    : cint
    callout_data*    : pointer
    pattern_position*: cint
    next_item_length*: cint
    mark*            : pointer
```

**Что это:**  
Живой снимок внутреннего состояния движка, доставляемый вашей коллбэк-функции каждый раз, когда движок достигает точки `(?C)` в шаблоне (или перед каждым элементом при `AUTO_CALLOUT`). Коллбэки позволяют наблюдать, логировать или прерывать сопоставление по ходу его выполнения.

**Ключевые поля:**

- **`version`** — Версия структуры (0, 1 или 2). Более высокие версии добавляют поля в конце.
- **`callout_number`** — Число из `(?Cn)`, или 255 для автоматических коллбэков.
- **`subject`** / **`subject_length`** — Полная строка-субъект, в которой идёт сопоставление.
- **`start_match`** — Смещение в байтах начала текущей попытки сопоставления.
- **`current_position`** — Текущее смещение движка внутри субъекта.
- **`capture_top`** — На единицу больше наибольшего номера установленной группы захвата.
- **`capture_last`** — Номер последней закрытой группы захвата.
- **`callout_data`** — Указатель из `ExtraData.callout_data`, передаётся без изменений.
- **`pattern_position`** / **`next_item_length`** — Позиция в шаблоне и длина следующего его элемента — полезно для трассировки.
- **`mark`** — Текущее значение `(*MARK:name)` (присутствует с версии 2).

Коллбэк-функция возвращает `0` для продолжения или ненулевое значение для прерывания с `ERROR_CALLOUT`. В C вы устанавливаете `pcre_callout` глобально; из Nim — через `{.emit:…}`.

---

### `JitCallback`

```nim
type JitCallback* = proc (a: pointer): ptr JitStack {.cdecl.}
```

**Что это:**  
C-совместимый (`cdecl`) коллбэк, вызываемый JIT-движком непосредственно перед каждой попыткой сопоставления для получения `JitStack`. Параметр `a` — это то, что вы передали как `data` в `assign_jit_stack()`.

Основной сценарий использования — **потокозависимые JIT-стеки**: храните `ptr JitStack` в потокозависимой переменной и возвращайте его из коллбэка. Это исключает конкуренцию и делает JIT-сопоставление потокобезопасным.

```nim
var myThreadStack {.threadvar.}: ptr JitStack

proc getStack(data: pointer): ptr JitStack {.cdecl.} =
  if myThreadStack == nil:
    myThreadStack = jit_stack_alloc(32768, 1 shl 20)
  return myThreadStack

assign_jit_stack(extra, getStack, nil)
```

---

### `PPcre`, `PJitStack` *(устаревшие)*

```nim
type
  PPcre*     {.deprecated.} = ptr Pcre
  PJitStack* {.deprecated.} = ptr JitStack
```

Устаревшие псевдонимы из более ранней версии привязки. Используйте `ptr Pcre` и `ptr JitStack` напрямую.

---

## Функции

Все функции импортированы из разделяемой библиотеки PCRE через C-вызывательное соглашение. Имена в Nim соответствуют C-именам по правилу `pcre_<nimName>` (например, Nim-функция `compile` → C-функция `pcre_compile`).

---

### `compile`

```nim
proc compile*(pattern: cstring,
              options: cint,
              errptr: ptr cstring,
              erroffset: ptr cint,
              tableptr: pointer): ptr Pcre
```

**Что делает:**  
Компилирует строку шаблона регулярного выражения во внутреннее представление, пригодное для многократного использования при сопоставлении. Это главная точка входа в PCRE — компилируйте один раз, сопоставляйте много раз.

**Параметры:**
- `pattern` — Строка регулярного выражения с нулевым терминатором.
- `options` — Побитовое OR компиляционных флагов (`CASELESS`, `MULTILINE`, `UTF8` и т.д.).
- `errptr` — При сбое: заполняется статическим читаемым сообщением об ошибке.
- `erroffset` — При сбое: заполняется байтовым смещением в `pattern`, где обнаружена ошибка.
- `tableptr` — Пользовательские таблицы символов от `maketables()`, или `nil` для встроенных ASCII-таблиц.

**Возвращает:** Ненулевой `ptr Pcre` при успехе. `nil` при сбое — изучите `errptr` и `erroffset`.

**Владение:** Скомпилированный шаблон не освобождается автоматически. `std/re` управляет временем жизни через GC Nim. При прямом использовании — декрементируйте счётчик ссылок через `refcount(re, -1)` или вызывайте C-функцию `pcre_free()` напрямую.

```nim
var err: cstring
var errOff: cint
let re = compile(r"(\d{4})-(\d{2})-(\d{2})", 0, addr err, addr errOff, nil)
if re == nil:
  quit("Ошибка шаблона в позиции " & $errOff & ": " & $err)
```

---

### `compile2`

```nim
proc compile2*(pattern: cstring,
               options: cint,
               errorcodeptr: ptr cint,
               errptr: ptr cstring,
               erroffset: ptr cint,
               tableptr: pointer): ptr Pcre
```

**Что делает:**  
Идентична `compile()`, но дополнительно заполняет `errorcodeptr` числовым кодом ошибки при сбое. Полезна, когда нужно программно различать типы ошибок (например, сбой валидности UTF против синтаксической ошибки) без разбора строки.

```nim
var errCode: cint
var err: cstring
var errOff: cint
let re = compile2(r"[\x{FFFFFF}]", UTF8,
                  addr errCode, addr err, addr errOff, nil)
if re == nil:
  echo "Код ошибки ", errCode, ": ", err
```

---

### `exec`

```nim
proc exec*(code: ptr Pcre,
           extra: ptr ExtraData,
           subject: cstring,
           length: cint,
           startoffset: cint,
           options: cint,
           ovector: ptr cint,
           ovecsize: cint): cint
```

**Что делает:**  
Основная функция сопоставления. Пытается сопоставить скомпилированный шаблон с `subject`, начиная с `startoffset`. Использует NFA-движок с возвратом.

**Параметры:**
- `code` — Скомпилированный шаблон от `compile()`.
- `extra` — Необязательные `ExtraData` для лимитов, коллбэков, таблиц или `nil`.
- `subject` — Строка для поиска.
- `length` — Длина в байтах (не кодовых точках для UTF-8). Передайте `-1`, чтобы PCRE вызвал `strlen`.
- `startoffset` — Байтовое смещение для начала сопоставления. Используйте `0` для старта.
- `options` — Флаги вызова: `NOTBOL`, `NOTEOL`, `NOTEMPTY`, `PARTIAL_SOFT` и т.д.
- `ovector` — Выходной массив пар `cint`: `[start0, end0, start1, end1, …]`. Пара 0 = всё совпадение; пара 1 = первая группа захвата и т.д. Размер должен быть кратен 3.
- `ovecsize` — Количество элементов в `ovector`. Минимальное безопасное значение: `(количествоГрупп + 1) * 3`.

**Возвращает:**
- `> 0` — Совпадение найдено. Значение = число заполненных пар захватов. `0` означает, что вектор был слишком мал.
- `ERROR_NOMATCH (-1)` — Нет совпадения.
- Другое отрицательное — Код ошибки.

```nim
var ov: array[30, cint]   # место для 9 групп + всё совпадение
let subj = "Дата: 2024-03-15"
let rc = exec(re, nil, subj, subj.len.cint, 0, 0, addr ov[0], 30)
if rc > 0:
  echo "всё совпадение: ", subj[ov[0]..<ov[1]]
  echo "год:   ", subj[ov[2]..<ov[3]]
  echo "месяц: ", subj[ov[4]..<ov[5]]
  echo "день:  ", subj[ov[6]..<ov[7]]
```

**Итерация по всем совпадениям:**

```nim
var start = 0
while start <= subj.len:
  let rc = exec(re, nil, subj, subj.len.cint, start.cint, 0, addr ov[0], 30)
  if rc < 0: break
  echo subj[ov[0]..<ov[1]]
  # Продвигаемся за совпадение; при пустом — на 1 вперёд во избежание бесконечного цикла
  start = if ov[1] > ov[0]: ov[1] else: ov[1] + 1
```

---

### `jit_exec`

```nim
proc jit_exec*(code: ptr Pcre,
               extra: ptr ExtraData,
               subject: cstring,
               length: cint,
               startoffset: cint,
               options: cint,
               ovector: ptr cint,
               ovecsize: cint,
               jstack: ptr JitStack): cint
```

**Что делает:**  
Семантически идентична `exec()`, но для сопоставления использует JIT-скомпилированный машинный код — значительно быстрее (обычно в 5–20 раз) для шаблонов, изученных с `STUDY_JIT_COMPILE`. При отсутствии JIT или неудаче компиляции молча переключается на интерпретируемое сопоставление.

Дополнительный параметр `jstack` задаёт JIT-стек непосредственно для этого вызова. Передайте `nil`, чтобы использовать стек, зарегистрированный через `assign_jit_stack()`.

```nim
# Полный JIT-конвейер:
var studyErr: cstring
let extra = study(re, STUDY_JIT_COMPILE, addr studyErr)
let jstack = jit_stack_alloc(32768, 1 shl 20)
assign_jit_stack(extra, nil, cast[pointer](jstack))

let rc = jit_exec(re, extra, subj, subj.len.cint,
                  0, 0, addr ov[0], 30, jstack)

# Очистка:
jit_stack_free(jstack)
free_study(extra)
```

---

### `dfa_exec`

```nim
proc dfa_exec*(code: ptr Pcre,
               extra: ptr ExtraData,
               subject: cstring,
               length: cint,
               startoffset: cint,
               options: cint,
               ovector: ptr cint,
               ovecsize: cint,
               workspace: ptr cint,
               wscount: cint): cint
```

**Что делает:**  
Альтернативный движок сопоставления на базе **DFA (детерминированного конечного автомата)**. Ключевые отличия от `exec()`:

- **Без возврата.** Устойчив к катастрофическому возврату ("ReDoS"). Время работы всегда O(n) от длины субъекта.
- **Находит все совпадения** в текущей позиции одновременно (вектор содержит все, начиная с наидлиннейшего), а не только первое.
- **Без групп захвата.** Только `ovector[0..1]` (граница всего совпадения) значима; все остальные пары не используются.
- Как правило, медленнее JIT-ускоренного `exec()` для типичных шаблонов.

**Дополнительные параметры:**
- `workspace` — Рабочий массив `cint` для внутреннего состояния DFA. Минимум 20 элементов; сложные шаблоны требуют больше.
- `wscount` — Длина `workspace`.

```nim
var ov: array[30, cint]
var ws: array[200, cint]
let rc = dfa_exec(re, nil, subj, subj.len.cint,
                  0, 0, addr ov[0], 30, addr ws[0], 200)
if rc > 0:
  # ov[0..1] = наидлиннейшее, ov[2..3] = второе по длине, …
  for i in 0 ..< rc:
    echo "совпадение ", i, ": ", subj[ov[i*2]..<ov[i*2+1]]
```

---

### `study`

```nim
proc study*(code: ptr Pcre,
            options: cint,
            errptr: ptr cstring): ptr ExtraData
```

**Что делает:**  
Анализирует скомпилированный шаблон и формирует блок `ExtraData` с оптимизационными данными. Главное применение — запрос **JIT-компиляции**, которая транслирует шаблон в нативный машинный код и может ускорить сопоставление в 5–100 раз.

**Параметры:**
- `code` — Скомпилированный шаблон от `compile()`.
- `options` — Флаги `STUDY_*`. Используйте `STUDY_JIT_COMPILE` для JIT.
- `errptr` — Заполняется сообщением об ошибке, если само изучение завершилось неудачно.

**Возвращает:** `ptr ExtraData` при успехе. Возвращает `nil` без ошибки, если изучение не дало полезных данных и `STUDY_EXTRA_NEEDED` не установлен — это нормально для простых шаблонов. Возвращает `nil` с установленным `errptr` при настоящем сбое.

**Жизненный цикл:** Необходимо освободить через `free_study()` по завершении работы.

```nim
var studyErr: cstring
let extra = study(re, STUDY_JIT_COMPILE, addr studyErr)
if studyErr != nil:
  echo "Ошибка изучения: ", studyErr
elif extra != nil:
  var jitDone: cint
  discard fullinfo(re, extra, INFO_JIT, addr jitDone)
  echo if jitDone == 1: "JIT активен" else: "JIT недоступен на этой платформе"
```

---

### `free_study`

```nim
proc free_study*(extra: ptr ExtraData)
```

**Что делает:**  
Освобождает блок `ExtraData`, возвращённый `study()`, включая любой встроенный JIT-скомпилированный код. Всегда используйте в паре с `study()`.

```nim
free_study(extra)
```

---

### `fullinfo`

```nim
proc fullinfo*(code: ptr Pcre,
               extra: ptr ExtraData,
               what: cint,
               where: pointer): cint
```

**Что делает:**  
Запрашивает метаданные скомпилированного шаблона. Передайте константу `INFO_*` как `what` и указатель на переменную подходящего типа как `where`. Возвращает `0` при успехе или отрицательный код ошибки.

```nim
# Сколько групп захвата в шаблоне?
var nCap: cint
discard fullinfo(re, nil, INFO_CAPTURECOUNT, addr nCap)
var ov = newSeq[cint]((nCap + 1) * 3)

# Может ли шаблон совпасть с пустой строкой? (важно для безопасности цикла)
var canEmpty: cint
discard fullinfo(re, nil, INFO_MATCH_EMPTY, addr canEmpty)
if canEmpty == 1:
  echo "Внимание: шаблон может совпасть с пустыми строками"
```

---

### `config`

```nim
proc config*(what: cint, where: pointer): cint
```

**Что делает:**  
Запрашивает конфигурацию **библиотеки** PCRE (не конкретного шаблона). Используйте константы `CONFIG_*` как `what`. Полезно для обнаружения возможностей перед попыткой операций, зависящих от конкретных параметров сборки PCRE.

```nim
var hasUcp: cint
discard config(CONFIG_UNICODE_PROPERTIES, addr hasUcp)
if hasUcp == 0:
  echo "Свойства Unicode \\p{} недоступны — пересоберите PCRE с --enable-unicode-properties"
```

---

### `get_substring`

```nim
proc get_substring*(subject: cstring,
                    ovector: ptr cint,
                    stringcount: cint,
                    stringnumber: cint,
                    stringptr: cstringArray): cint
```

**Что делает:**  
Выделяет новую C-строку с текстом группы захвата номер `stringnumber` (0 = всё совпадение) и записывает её адрес в `stringptr`. `stringcount` — значение, возвращённое `exec()`. Возвращает длину строки или отрицательный код ошибки.

Необходимо освободить через `free_substring()`.

```nim
var sub: cstringArray
let len = get_substring(subj, addr ov[0], rc, 1, addr sub)
if len >= 0:
  echo "группа 1: ", sub
  free_substring(sub)
```

> На практике в Nim обычно чище извлекать подстроки напрямую из `subject` по индексам вектора: `subject[ov[2]..<ov[3]]`.

---

### `get_named_substring`

```nim
proc get_named_substring*(code: ptr Pcre,
                          subject: cstring,
                          ovector: ptr cint,
                          stringcount: cint,
                          stringname: cstring,
                          stringptr: cstringArray): cint
```

**Что делает:**  
Как `get_substring()`, но извлекает группу захвата по **имени** (из синтаксиса `(?P<n>…)` или `(?<n>…)`). Возвращает длину подстроки или отрицательный код ошибки.

```nim
var yearStr: cstringArray
discard get_named_substring(re, subj, addr ov[0], rc, "year", addr yearStr)
echo "год: ", yearStr
free_substring(yearStr)
```

---

### `copy_substring`

```nim
proc copy_substring*(subject: cstring,
                     ovector: ptr cint,
                     stringcount: cint,
                     stringnumber: cint,
                     buffer: cstring,
                     buffersize: cint): cint
```

**Что делает:**  
Как `get_substring()`, но копирует в **буфер, предоставленный вызывающим кодом**, без выделения памяти. `free_substring()` не нужен. Возвращает `ERROR_NOMEMORY`, если буфер слишком мал.

```nim
var buf: array[256, char]
let rc2 = copy_substring(subj, addr ov[0], rc, 0,
                         cast[cstring](addr buf[0]), 256)
if rc2 >= 0:
  echo "совпадение: ", cast[cstring](addr buf[0])
```

---

### `copy_named_substring`

```nim
proc copy_named_substring*(code: ptr Pcre,
                           subject: cstring,
                           ovector: ptr cint,
                           stringcount: cint,
                           stringname: cstring,
                           buffer: cstring,
                           buffersize: cint): cint
```

**Что делает:**  
Объединяет `copy_substring` и `get_named_substring`: извлекает именованную группу захвата в буфер вызывающего кода без выделения памяти.

---

### `get_substring_list`

```nim
proc get_substring_list*(subject: cstring,
                         ovector: ptr cint,
                         stringcount: cint,
                         listptr: ptr cstringArray): cint
```

**Что делает:**  
Выделяет нулетерминированный массив C-строк, содержащий **все** захваченные подстроки (индекс 0 = всё совпадение). Удобен для вывода всех захватов сразу. Освободить через `free_substring_list()`.

```nim
var list: cstringArray
discard get_substring_list(subj, addr ov[0], rc, addr list)
var i = 0
while list[i] != nil:
  echo "группа ", i, ": ", list[i]
  inc i
free_substring_list(list)
```

---

### `free_substring`

```nim
proc free_substring*(stringptr: cstring)
```

Освобождает строку, выделенную `get_substring()` или `get_named_substring()`.

---

### `free_substring_list`

```nim
proc free_substring_list*(stringptr: cstringArray)
```

Освобождает список, выделенный `get_substring_list()`.

---

### `get_stringnumber`

```nim
proc get_stringnumber*(code: ptr Pcre, name: cstring): cint
```

**Что делает:**  
Преобразует **имя** группы захвата в её **номер**. Возвращает номер (≥ 1) при успехе или отрицательный код ошибки, если имя не найдено в шаблоне.

```nim
let yearIdx = get_stringnumber(re, "year")
if yearIdx > 0:
  echo "год: ", subj[ov[yearIdx*2]..<ov[yearIdx*2+1]]
```

---

### `get_stringtable_entries`

```nim
proc get_stringtable_entries*(code: ptr Pcre,
                              name: cstring,
                              first: cstringArray,
                              last: cstringArray): cint
```

**Что делает:**  
При активном `DUPNAMES`, когда несколько групп разделяют одно имя, возвращает указатели на первую и последнюю записи во внутренней таблице имён для данного имени, позволяя итерировать по всем группам с этим именем. Возвращает размер записи при успехе.

---

### `maketables`

```nim
proc maketables*(): pointer
```

**Что делает:**  
Генерирует полный набор таблиц классификации символов, используя текущую локаль C-библиотеки (устанавливается через `setlocale`). Возвращаемый указатель можно передать в `compile()` как `tableptr`, чтобы сделать свёртку регистра и символьные классы локалезависимыми. Возвращает `nil` при ошибке выделения памяти.

Владелец — вызывающий код; освобождается через C-функцию `free()`.

```nim
let tables = maketables()
let re = compile(r"\w+", CASELESS, addr err, addr errOff, tables)
```

---

### `refcount`

```nim
proc refcount*(code: ptr Pcre, adjust: cint): cint
```

**Что делает:**  
Изменяет счётчик ссылок скомпилированного шаблона на `adjust` и возвращает новое значение. Передайте `+1` для инкремента, `-1` для декремента (освобождение при достижении 0), `0` для запроса без изменения. Используется обёртками высокого уровня, реализующими совместное владение скомпилированными шаблонами.

---

### `version`

```nim
proc version*(): cstring
```

**Что делает:**  
Возвращает статическую строку от PCRE-библиотеки, описывающую её рантаймовую версию и дату, например `"8.45 2021-06-15"`. Используйте для проверки соответствия установленной разделяемой библиотеки тому, что ожидает ваш код.

```nim
echo "Привязка к заголовку: ", PCRE_MAJOR, ".", PCRE_MINOR
echo "Рантаймовая библиотека: ", version()
```

---

### `pattern_to_host_byte_order`

```nim
proc pattern_to_host_byte_order*(code: ptr Pcre,
                                 extra: ptr ExtraData,
                                 tables: pointer): cint
```

**Что делает:**  
Выполняет переставку байт скомпилированного шаблона, сериализованного на машине с другим порядком байт. Возвращает `0`, если перестановка была нужна и выполнена; `ERROR_BADENDIANNESS` — если шаблон уже в нужном порядке байт. Актуально только в сценариях кросс-платформенной сериализации/кеширования.

---

### `jit_stack_alloc`

```nim
proc jit_stack_alloc*(startsize: cint, maxsize: cint): ptr JitStack
```

**Что делает:**  
Выделяет JIT-стек выполнения, начинающийся с `startsize` байт и способный вырасти до `maxsize` байт по требованию. Возвращает `nil` при ошибке выделения.

**Рекомендации по размеру:** `32 * 1024` для старта / `1 shl 20` (1 МБ) максимум — хороший выбор по умолчанию. Шаблоны с глубокой рекурсией или длинным вводом могут требовать больше.

```nim
let stack = jit_stack_alloc(32 * 1024, 1 shl 20)
if stack == nil: quit("Не удалось выделить JIT-стек")
```

---

### `jit_stack_free`

```nim
proc jit_stack_free*(stack: ptr JitStack)
```

Освобождает JIT-стек, выделенный `jit_stack_alloc()`. Всегда используйте в паре.

---

### `assign_jit_stack`

```nim
proc assign_jit_stack*(extra: ptr ExtraData,
                       callback: JitCallback,
                       data: pointer)
```

**Что делает:**  
Привязывает JIT-стек (или коллбэк, его предоставляющий) к `ExtraData` JIT-скомпилированного шаблона. Два режима:

- **Статический стек:** передайте `nil` как `callback` и `ptr JitStack` как `data`. PCRE всегда использует этот стек. Прост, но небезопасен для многопоточного использования при разделении одного `extra` между потоками.
- **Динамический коллбэк:** передайте `JitCallback` как `callback` и произвольный контекстный указатель как `data`. PCRE вызывает коллбэк перед каждым сопоставлением для получения стека. Идеален для потокозависимых стеков.

```nim
# Статический — подходит для однопоточного использования:
assign_jit_stack(extra, nil, cast[pointer](stack))

# Динамический — правильно для многопоточного использования:
assign_jit_stack(extra, getThreadStack, nil)
```

---

### `jit_free_unused_memory`

```nim
proc jit_free_unused_memory*()
```

**Что делает:**  
Освобождает память в пуле JIT-аллокатора исполняемого кода, которая больше не используется ни одним скомпилированным шаблоном. Полезно в долгоживущих программах, компилирующих множество шаблонов с течением времени, для предотвращения бесконтрольного роста JIT-памяти.

---

## Полные примеры использования

### Минимальное жизнеспособное сопоставление

```nim
import pcre

var err: cstring
var errOff: cint
let re = compile(r"(\w+)\s+(\w+)", 0, addr err, addr errOff, nil)
if re == nil:
  quit("Ошибка компиляции в позиции " & $errOff & ": " & $err)

var ov: array[9, cint]  # (2 группы + всё совпадение) * 3
let subj = "hello world"
let rc = exec(re, nil, subj, subj.len.cint, 0, 0, addr ov[0], 9)
if rc > 0:
  echo "совпадение:  ", subj[ov[0]..<ov[1]]
  echo "слово 1:     ", subj[ov[2]..<ov[3]]
  echo "слово 2:     ", subj[ov[4]..<ov[5]]
```

### JIT-ускоренное сопоставление в цикле

```nim
var studyErr: cstring
let extra = study(re, STUDY_JIT_COMPILE, addr studyErr)
let jstack = jit_stack_alloc(32768, 1 shl 20)
assign_jit_stack(extra, nil, cast[pointer](jstack))

var start = 0
while start <= subj.len:
  let rc = jit_exec(re, extra, subj, subj.len.cint,
                    start.cint, 0, addr ov[0], 9, jstack)
  if rc < 0: break
  echo subj[ov[0]..<ov[1]]
  start = if ov[1] > ov[0]: ov[1] else: ov[1] + 1

jit_stack_free(jstack)
free_study(extra)
```

### Защита от ReDoS через лимиты сопоставления

```nim
var extra = ExtraData()
extra.flags = EXTRA_MATCH_LIMIT or EXTRA_MATCH_LIMIT_RECURSION
extra.match_limit            = 100_000
extra.match_limit_recursion  =   5_000

let rc = exec(re, addr extra, untrustedInput, untrustedInput.len.cint,
              0, 0, addr ov[0], 9)
case rc
of ERROR_MATCHLIMIT: echo "Ввод отклонён: достигнут лимит сопоставления"
of ERROR_NOMATCH:    echo "Нет совпадения"
else: discard
```

### Именованные группы захвата

```nim
let datePat = compile(r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})",
                      0, addr err, addr errOff, nil)
var ov: array[12, cint]
let rc = exec(datePat, nil, "2024-03-15", 10, 0, 0, addr ov[0], 12)
if rc > 0:
  let yearN = get_stringnumber(datePat, "year")
  echo "год: ", "2024-03-15"[ov[yearN*2]..<ov[yearN*2+1]]
```
