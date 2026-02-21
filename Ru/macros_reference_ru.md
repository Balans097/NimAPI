# macros.nim — Справочник модуля

> **Импорт:** `import std/macros`  
> **Назначение:** Доступ к абстрактному синтаксическому дереву (AST) Nim и его преобразование во время компиляции. Всё в этом модуле выполняется *при компиляции*, внутри определений `macro`.

---

## Что такое макрос в Nim?

Макрос — это процедура, которая выполняется **во время компиляции**, получает фрагменты исходного кода программы в виде дерева объектов `NimNode` и возвращает новое дерево `NimNode`, заменяющее исходный код. Именно так Nim реализует DSL с нулевыми накладными расходами, кодогенерацию и статическую рефлексию без каких-либо затрат во время исполнения.

Центральный тип — `NimNode`: каждый фрагмент синтаксиса Nim — литерал, имя переменной, выражение вызова, целое определение `proc` — представляется как `NimNode` с конкретным `NimNodeKind`.

---

## Основные типы

### `NimNodeKind`

Перечисление примерно с 150 значениями, классифицирующее каждую синтаксическую конструкцию Nim. Наиболее часто встречающиеся:

| Вид | Что представляет |
|---|---|
| `nnkIdent` | Идентификатор, напр. `foo` |
| `nnkSym` | Разрешённый символ (после семантического анализа) |
| `nnkIntLit` | Целочисленный литерал |
| `nnkStrLit` | Строковый литерал |
| `nnkCall` | Вызов процедуры `f(a, b)` |
| `nnkInfix` | Инфиксный вызов `a + b` |
| `nnkStmtList` | Последовательность операторов |
| `nnkProcDef` | Определение `proc` |
| `nnkEmpty` | Отсутствующий/необязательный узел-заглушка |

**Наборы узлов** (полезные константы):

| Константа | Содержимое |
|---|---|
| `nnkLiterals` | Все литеральные виды (`nnkCharLit..nnkNilLit`) |
| `nnkCallKinds` | Все вызовоподобные узлы |
| `RoutineNodes` | Все виды определений подпрограмм (proc, func, method, …) |
| `AtomicNodes` | Листовые узлы без дочерних |
| `CallNodes` | То же, что `nnkCallKinds` |

---

### `NimTypeKind`

Классифицирует типы в представлении компилятора (напр. `ntyInt`, `ntyObject`, `ntySeq`). Получается через `typeKind`.

---

### `NimSymKind`

Классифицирует, на что ссылается разрешённый символ (напр. `nskProc`, `nskVar`, `nskType`, `nskField`). Получается через `symKind`.

---

### `LineInfo`

```nim
type LineInfo* = object
  filename*: string
  line*, column*: int
```

Позиция в исходном коде. Возвращается `lineInfoObj`. Может быть преобразована в строку `"файл(строка, столбец)"` через `$`.

---

### `BindSymRule`

Управляет тем, как `bindSym` разрешает перегруженные символы:

| Значение | Поведение |
|---|---|
| `brClosed` | Только символы текущей области видимости; по умолчанию |
| `brOpen` | Открыт для перегрузок по правилам обобщённого связывания |
| `brForceOpen` | Всегда возвращает узел открытого выбора |

---

## Инспекция узлов

### `len`

```nim
proc len*(n: NimNode): int
```

Возвращает количество прямых дочерних узлов `n`. Листовые узлы (литералы, идентификаторы) имеют `len == 0`.

```nim
# nnkCall для "echo(a, b)" имеет 3 дочерних: echo, a, b
assert callNode.len == 3
```

---

### `[]` / `[]=` — доступ к дочерним узлам

```nim
proc `[]`*(n: NimNode, i: int): NimNode
proc `[]`*(n: NimNode, i: BackwardsIndex): NimNode   # n[^1]
proc `[]`*[T,U](n: NimNode, x: HSlice[T,U]): seq[NimNode]
proc `[]=`*(n: NimNode, i: int, child: NimNode)
proc `[]=`*(n: NimNode, i: BackwardsIndex, child: NimNode)
```

Читает или заменяет дочерний узел по индексу. Отрицательная индексация (`n[^1]`) обращается с конца. Срезовая индексация возвращает `seq[NimNode]`.

```nim
let call = quote do: foo(1, 2)
assert $call[0] == "foo"   # имя процедуры
assert call[1].intVal == 1
call[2] = newLit(99)       # заменяем второй аргумент
```

---

### `kind`

```nim
proc kind*(n: NimNode): NimNodeKind
```

Возвращает вид узла `n`. Самый фундаментальный дискриминант — всегда проверяйте `kind` перед чтением `intVal`, `strVal` или `floatVal`.

```nim
if n.kind == nnkStrLit:
  echo "строка: ", n.strVal
```

---

### `intVal`, `floatVal`, `strVal`

```nim
proc intVal*(n: NimNode): BiggestInt
proc floatVal*(n: NimNode): BiggestFloat
proc strVal*(n: NimNode): string
```

Читают полезную нагрузку литерала или идентификатора. Каждая действительна только для соответствующего `kind`:

- `intVal` — целочисленные и символьные литералы, символы полей перечисления
- `floatVal` — вещественные литералы
- `strVal` — строковые литералы, идентификаторы (`nnkIdent`), символы (`nnkSym`), комментарии

```nim
let n = newLit(42)
assert n.intVal == 42

let id = ident("myProc")
assert id.strVal == "myProc"
```

---

### `strVal=`

```nim
proc `strVal=`*(n: NimNode, val: string)
```

Устанавливает строковое содержимое строкового литерала или комментария. Недопустимо для `nnkIdent` и `nnkSym` — создайте новый узел через `ident(string)`.

---

### `intVal=` / `floatVal=`

```nim
proc `intVal=`*(n: NimNode, val: BiggestInt)
proc `floatVal=`*(n: NimNode, val: BiggestFloat)
```

Мутируют числовую нагрузку литерального узла на месте.

---

### `symKind`

```nim
proc symKind*(symbol: NimNode): NimSymKind
```

Для узла вида `nnkSym` возвращает, на что он ссылается. Полезно для различения символа `nskProc` от `nskType` или `nskVar`.

```nim
if sym.symKind == nskProc:
  echo "Это символ процедуры"
```

---

### `boolVal`

```nim
proc boolVal*(n: NimNode): bool
```

Извлекает булевое значение из узла `nnkIntLit` или связанного символа `true`/`false`. Избавляет от написания `n.intVal != 0` напрямую.

---

### `last`

```nim
proc last*(node: NimNode): NimNode
```

Возвращает последний дочерний узел. Эквивалентно `node[^1]`. Удобно при обходе списков операторов.

---

### `==` (структурное равенство)

```nim
proc `==`*(a, b: NimNode): bool
```

Два узла равны, если они **структурно идентичны** — одинаковый вид, одинаковая нагрузка, одинаковые дочерние. Два независимо построенных узла с одинаковым содержимым будут равны.

---

### `sameType`

```nim
proc sameType*(a, b: NimNode): bool
```

Возвращает `true`, если `a` и `b` представляют один и тот же тип, в том числе через псевдонимы (`type MyInt = int` — тот же тип, что и `int`).

---

### `eqIdent`

```nim
proc eqIdent*(a: string; b: string): bool
proc eqIdent*(a: NimNode; b: string): bool
proc eqIdent*(a: string; b: NimNode): bool
proc eqIdent*(a: NimNode; b: NimNode): bool
```

**Нечувствительное к стилю** сравнение идентификаторов: `eqIdent("myProc", "myproc")` — `true`. Автоматически разворачивает `nnkPostfix` (маркер экспорта `*`) и имена в обратных кавычках. Всегда используйте `eqIdent` вместо `$node == name` при сравнении идентификаторов.

```nim
if n.eqIdent("result"):
  echo "Найдена переменная result"
```

---

### `or` (запасной вариант для пустого узла)

```nim
template `or`*(x, y: NimNode): NimNode
```

Вычисляет `x`; если он `nil` или `nnkEmpty` — возвращает `y`. Позволяет элегантно выстраивать цепочки «попробуй это или то».

```nim
let body = node.body or newStmtList()
```

---

## Инспекция типов

### `getType`

```nim
proc getType*(n: NimNode): NimNode
proc getType*(n: typedesc): NimNode
```

Возвращает представление типа `n` в виде `NimNode`. Результат — само дерево узлов, которое можно обходить. Рекурсивные типы предварительно выравниваются во избежание бесконечной рекурсии; вызовите `getType` снова на результирующем узле, чтобы пройти по псевдонимам.

```nim
macro showType(x: typed): untyped =
  echo x.getType.repr
showType(3.14)  # выводит: float64
```

---

### `typeKind`

```nim
proc typeKind*(n: NimNode): NimTypeKind
```

Для узла, полученного через `getType`, возвращает категорию типа (`ntyInt`, `ntyObject`, `ntySeq` и т. д.).

---

### `getTypeInst`

```nim
proc getTypeInst*(n: NimNode): NimNode
proc getTypeInst*(n: typedesc): NimNode
```

Возвращает тип **в том виде, в каком он записан в месте объявления**, сохраняя псевдонимы. Полезно, когда нужно знать, что переменная была объявлена как `Vec4f`, а не раскрытый `Vec[4, float32]`.

---

### `getTypeImpl`

```nim
proc getTypeImpl*(n: NimNode): NimNode
proc getTypeImpl*(n: typedesc): NimNode
```

Возвращает тип **после раскрытия всех псевдонимов** — конкретную реализацию. В отличие от `getTypeInst`, все промежуточные псевдонимы отброшены: видно финальное определение `object`, `tuple` и т. д.

---

### `getImpl`

```nim
proc getImpl*(symbol: NimNode): NimNode
```

Возвращает полный AST объявления символа — полное тело `proc`, полное определение `type` и т. д. Возвращает `nil` для символов без доступного исходного кода (напр. встроенных magic-символов).

```nim
macro showImpl(x: typed): untyped =
  echo x.getImpl.treeRepr
```

---

### `getImplTransformed`

```nim
proc getImplTransformed*(symbol: NimNode): NimNode
```

Для типизированного `proc` возвращает AST **после прохода трансформаций компилятора** — после переписывания `defer`, трансформации итераторов и т. д. Полезно для отладки внутренних преобразований компилятора. (Доступно с Nim 1.3.5.)

---

### `isInstantiationOf`

```nim
proc isInstantiationOf*(instanceProcSym, genProcSym: NimNode): bool
```

Проверяет, является ли `instanceProcSym` конкретной инстанциацией обобщённой процедуры `genProcSym`. Полезно для верификации символов proc против обобщённых шаблонов, возвращаемых `bindSym`.

---

### `signatureHash`

```nim
proc signatureHash*(n: NimNode): string
```

Возвращает стабильную хеш-строку, производную от сигнатуры символа: тип, владеющий модуль и другие факторы. Тот же идентификатор, который бекенд использует для искажения имён символов (name mangling).

---

### `symBodyHash`

```nim
proc symBodyHash*(s: NimNode): string
```

Более глубокий хеш, чем `signatureHash`: включает тело реализации и рекурсивно хеширует все используемые внутри proc и переменные. Полезен для инвалидации кешей в инструментах кодогенерации.

---

### `isExported`

```nim
proc isExported*(n: NimNode): bool
```

Возвращает `true`, если символ, представленный `n`, экспортирован (помечен `*`).

---

### `getSize`, `getAlign`, `getOffset`

```nim
proc getSize*(arg: NimNode): int
proc getAlign*(arg: NimNode): int
proc getOffset*(arg: NimNode): int
```

Аналоги `sizeof`, `alignof` и `offsetof` во время компиляции. Возвращают отрицательное значение, если компилятор не может определить результат. `getOffset` принимает единственный символ-узел поля, а не пару тип+поле.

```nim
macro checkSize(T: typedesc): untyped =
  let sz = getType(T).getSize
  echo "Размер типа: ", sz
```

---

## Построение узлов

### `newNimNode`

```nim
proc newNimNode*(kind: NimNodeKind, lineInfoFrom: NimNode = nil): NimNode
```

Фундаментальный конструктор узлов. Создаёт новый узел заданного вида без дочерних. `lineInfoFrom` — любой существующий узел, из которого копируется позиция в исходном коде, чтобы ошибки компилятора указывали на осмысленное место.

```nim
let call = newNimNode(nnkCall)
call.add(ident("echo"), newLit("hello"))
```

---

### `newTree`

```nim
proc newTree*(kind: NimNodeKind, children: varargs[NimNode]): NimNode
```

Более удобный `newNimNode`, который немедленно добавляет дочерние узлы. Идиоматический способ строить составные узлы AST.

```nim
# Эквивалентно выражению `a + b` в коде
let sum = newTree(nnkInfix, ident("+"), a, b)

# Можно также использовать вид как конструктор:
let sum2 = nnkInfix.newTree(ident("+"), a, b)
```

---

### `newLit`

```nim
proc newLit*(c: char): NimNode
proc newLit*(i: int | int8 | int16 | int32 | int64): NimNode
proc newLit*(i: uint | uint8 | uint16 | uint32 | uint64): NimNode
proc newLit*(b: bool): NimNode
proc newLit*(s: string): NimNode
proc newLit*(f: float32 | float64): NimNode
proc newLit*(arg: enum): NimNode
proc newLit*(arg: object): NimNode
proc newLit*(arg: ref object): NimNode
proc newLit*[N,T](arg: array[N,T]): NimNode
proc newLit*[T](arg: seq[T]): NimNode
proc newLit*[T](s: set[T]): NimNode
proc newLit*[T: tuple](arg: T): NimNode
```

Преобразует значение Nim, известное во время компиляции, в его AST-представление в виде литерала. Для составных типов (объекты, массивы, последовательности, кортежи) работает рекурсивно. Полученный узел, вставленный в генерируемый код, воспроизведёт исходное значение.

```nim
let n = newLit(42)        # nnkIntLit с intVal = 42
let s = newLit("hello")   # nnkStrLit
let a = newLit([1, 2, 3]) # nnkBracket с тремя nnkIntLit-дочерними
```

---

### `ident`

```nim
proc ident*(name: string): NimNode
```

Создаёт узел `nnkIdent` из строки. Правильный способ ввести новое имя в генерируемый код. Не связывается ни с каким символом — компилятор разрешает его в месте использования сгенерированного кода.

```nim
let varName = ident("myCounter")
let stmt = newVarStmt(varName, newLit(0))
# Генерирует: var myCounter = 0
```

---

### `newIdentNode`

```nim
proc newIdentNode*(i: string): NimNode
```

Псевдоним для `ident(string)`. Предпочтительнее использовать `ident`.

---

### `bindSym`

```nim
proc bindSym*(ident: string | NimNode,
              rule: BindSymRule = brClosed): NimNode
```

Создаёт узел, **немедленно связанный** с конкретным символом в текущей области видимости, а не просто именем. В отличие от `ident`, результирующий узел несёт разрешённую идентичность символа, поэтому перегрузка и затенение имён в месте вызова не влияют на то, какой символ используется. Необходимо, когда нужно гарантировать, что генерируемый код вызовет определённую процедуру вне зависимости от локальных перегрузок.

```nim
# Генерирует код, вызывающий именно echo из system, а не возможную перегрузку
let echoSym = bindSym("echo")
result = newCall(echoSym, newLit("безопасно"))
```

---

### `genSym`

```nim
proc genSym*(kind: NimSymKind = nskLet; ident = ""): NimNode
```

Генерирует **гигиенически уникальный** символ, гарантированно не совпадающий ни с каким именем в окружающем коде. Используйте для временных переменных внутри макросов во избежание захвата переменных. `ident` — подсказка для отладчика, на уникальность не влияет.

```nim
let tmp = genSym(nskVar, "temp")
result = newVarStmt(tmp, expr)
result.add newAssignment(output, tmp)
```

---

### `newCall`

```nim
proc newCall*(theProc: NimNode | string, args: varargs[NimNode]): NimNode
```

Сокращение для построения узла `nnkCall`: первый аргумент — вызываемый (узел или строка-имя), затем узлы аргументов.

```nim
# Генерирует: echo("hello", x)
let call = newCall("echo", newLit("hello"), ident("x"))
```

---

### `newStrLitNode`

```nim
proc newStrLitNode*(s: string): NimNode
```

Создаёт узел `nnkStrLit`. Эквивалентно `newLit(s)`.

---

### `newIntLitNode`

```nim
proc newIntLitNode*(i: BiggestInt): NimNode
```

Создаёт узел `nnkIntLit`. Эквивалентно `newLit(i)`.

---

### `newFloatLitNode`

```nim
proc newFloatLitNode*(f: BiggestFloat): NimNode
```

Создаёт узел `nnkFloatLit`.

---

### `newCommentStmtNode`

```nim
proc newCommentStmtNode*(s: string): NimNode
```

Создаёт узел `nnkCommentStmt` с заданной строкой. Полезно для встраивания комментариев-документации в генерируемый код.

---

### `newNilLit`

```nim
proc newNilLit*(): NimNode
```

Создаёт литерал `nil` (`nnkNilLit`).

---

### `newEmptyNode`

```nim
proc newEmptyNode*(): NimNode
```

Создаёт узел `nnkEmpty` — заглушку для отсутствующих необязательных полей AST (напр. отсутствующий тип возврата, отсутствующее значение по умолчанию). Многие вспомогательные `newXxx` используют `newEmptyNode()` внутри.

---

### `newStmtList`

```nim
proc newStmtList*(stmts: varargs[NimNode]): NimNode
```

Создаёт узел `nnkStmtList` с заданными операторами. Основной строительный блок возвращаемых значений макросов.

```nim
result = newStmtList(
  newVarStmt(ident("x"), newLit(1)),
  newCall("echo", ident("x"))
)
# Генерирует:
#   var x = 1
#   echo x
```

---

### `newProc`

```nim
proc newProc*(name = newEmptyNode();
              params: openArray[NimNode] = [newEmptyNode()];
              body: NimNode = newStmtList();
              procType = nnkProcDef;
              pragmas: NimNode = newEmptyNode()): NimNode
```

Высокоуровневый конструктор определений процедур. `params[0]` — тип возврата; последующие элементы — узлы `nnkIdentDefs` параметров. `procType` выбирает вид создаваемой подпрограммы (proc, func, method, iterator и т. д.).

```nim
# Генерирует: proc double(x: int): int = x * 2
let p = newProc(
  name   = ident("double"),
  params = [ident("int"),
            newIdentDefs(ident("x"), ident("int"))],
  body   = newTree(nnkInfix, ident("*"), ident("x"), newLit(2))
)
```

---

### `newIfStmt`

```nim
proc newIfStmt*(branches: varargs[tuple[cond, body: NimNode]]): NimNode
```

Создаёт узел `nnkIfStmt` из пар условие-тело. Каждый кортеж становится одной ветвью `elif`.

```nim
let check = newIfStmt(
  (cond: ident("flag"), body: newCall("echo", newLit("on")))
)
# Генерирует: if flag: echo "on"
```

---

### `newEnum`

```nim
proc newEnum*(name: NimNode, fields: openArray[NimNode],
              public, pure: bool): NimNode
```

Создаёт полную секцию `type … = enum`. `name` должен быть `nnkIdent`. Поля могут быть простыми `nnkIdent` или `nnkEnumFieldDef` (для явных значений).

```nim
let e = newEnum(ident("Color"), [ident("Red"), ident("Green"), ident("Blue")],
                public = true, pure = false)
# Генерирует: type Color* = enum Red, Green, Blue
```

---

### `newVarStmt`, `newLetStmt`, `newConstStmt`

```nim
proc newVarStmt*(name, value: NimNode): NimNode
proc newLetStmt*(name, value: NimNode): NimNode
proc newConstStmt*(name, value: NimNode): NimNode
```

Создают объявления переменной, let или const с одним именем. Тип выводится автоматически; для явного типа используйте `newNimNode(nnkVarSection)` вручную.

```nim
newVarStmt(ident("count"), newLit(0))  # var count = 0
newLetStmt(ident("pi"), newLit(3.14))  # let pi = 3.14
```

---

### `newAssignment`

```nim
proc newAssignment*(lhs, rhs: NimNode): NimNode
```

Создаёт узел `nnkAsgn`: `lhs = rhs`.

---

### `newDotExpr`

```nim
proc newDotExpr*(a, b: NimNode): NimNode
```

Создаёт `a.b` — узел `nnkDotExpr` для доступа к полю или метода.

```nim
newDotExpr(ident("obj"), ident("field"))  # obj.field
```

---

### `newColonExpr`

```nim
proc newColonExpr*(a, b: NimNode): NimNode
```

Создаёт `a: b` — `nnkExprColonExpr`, используемый в именованных аргументах и конструкторах объектов.

---

### `newIdentDefs`

```nim
proc newIdentDefs*(name, kind: NimNode;
                   default = newEmptyNode()): NimNode
```

Создаёт трёхузловой `nnkIdentDefs`: `name: kind = default`. Используется для параметров и переменных внутри `nnkFormalParams` и `nnkVarSection`. `kind` или `default` могут быть `newEmptyNode()`.

```nim
newIdentDefs(ident("x"), ident("int"))              # x: int
newIdentDefs(ident("y"), newEmptyNode(), newLit(0)) # y = 0
```

---

### `newBlockStmt`

```nim
proc newBlockStmt*(label, body: NimNode): NimNode
proc newBlockStmt*(body: NimNode): NimNode
```

Создаёт оператор `block: …` с необязательной меткой.

---

### `newPar`

```nim
proc newPar*(exprs: NimNode): NimNode
```

Создаёт узел в скобках. Не используйте `nnkPar` для конструирования кортежей — используйте `nnkTupleConstr`.

---

### `nestList`

```nim
proc nestList*(op: NimNode; pack: NimNode): NimNode
proc nestList*(op: NimNode; pack: NimNode; init: NimNode): NimNode
```

Сворачивает список узлов в право-ассоциативное дерево бинарных вызовов: `[a, b, c]` → `op(a, op(b, c))`. С `init` даёт `op(a, op(b, op(c, init)))`. AST-эквивалент операции fold/reduce.

```nim
# Генерирует: a and b and c
let combined = nestList(ident("and"), conditions)
```

---

### `prefix`, `postfix`, `infix`

```nim
proc prefix*(node: NimNode; op: string): NimNode   # op node
proc postfix*(node: NimNode; op: string): NimNode  # node op
proc infix*(a: NimNode; op: string; b: NimNode): NimNode  # a op b
```

Удобные конструкторы операторных выражений.

```nim
let negated  = prefix(ident("x"), "not")          # not x
let exported = postfix(ident("foo"), "*")          # foo* (маркер экспорта)
let added    = infix(ident("a"), "+", ident("b")) # a + b
```

---

### `unpackPrefix`, `unpackPostfix`, `unpackInfix`

```nim
proc unpackPrefix*(node: NimNode): tuple[node: NimNode; op: string]
proc unpackPostfix*(node: NimNode): tuple[node: NimNode; op: string]
proc unpackInfix*(node: NimNode): tuple[left: NimNode; op: string; right: NimNode]
```

Обратная операция к конструкторам — разбирает операторный узел обратно на составляющие.

```nim
let (left, op, right) = infix(a, "+", b).unpackInfix
assert op == "+"
```

---

### `add`

```nim
proc add*(father, child: NimNode): NimNode {.discardable.}
proc add*(father: NimNode, children: varargs[NimNode]): NimNode {.discardable.}
```

Добавляет один или несколько дочерних узлов к `father`. Возвращает `father` для цепочечных вызовов.

```nim
let list = newStmtList()
list.add(newCall("echo", newLit("a")))
list.add(newCall("echo", newLit("b")))
```

---

### `del`

```nim
proc del*(father: NimNode, idx = 0, n = 1)
```

Удаляет `n` последовательных дочерних узлов начиная с индекса `idx`.

---

### `insert`

```nim
proc insert*(a: NimNode; pos: int; b: NimNode)
```

Вставляет узел `b` в `a` на позицию `pos`, сдвигая последующие дочерние вправо. Если `pos` выходит за конец, пустые узлы добавляются для заполнения пробела.

---

### `copyNimNode`

```nim
proc copyNimNode*(n: NimNode): NimNode
```

Поверхностная копия: копирует сам `n`, но **не** его дочерние. Новый узел имеет тот же вид и нагрузку, но ноль дочерних.

---

### `copyNimTree`

```nim
proc copyNimTree*(n: NimNode): NimNode
```

Глубокая копия: рекурсивно копирует `n` и всех его дочерних. Используйте, когда нужно изменить поддерево, не затрагивая оригинал.

---

### `copy`

```nim
proc copy*(node: NimNode): NimNode
```

Псевдоним для `copyNimTree`.

---

### `copyChildrenTo`

```nim
proc copyChildrenTo*(src, dest: NimNode)
```

Глубоко копирует всех дочерних `src` и дописывает их в `dest`. Полезно для слияния поддеревьев.

---

## Аксессоры подпрограмм

Эти процедуры читают и записывают именованные подузлы определения подпрограммы. Работают с любым узлом из `RoutineNodes`.

### `name` / `name=`

```nim
proc name*(someProc: NimNode): NimNode
proc `name=`*(someProc: NimNode; val: NimNode)
```

Получает или устанавливает узел имени подпрограммы. Автоматически разворачивает `nnkPostfix` (маркер экспорта) и `nnkAccQuoted` (имена в обратных кавычках), возвращая «голый» идентификатор.

---

### `params` / `params=`

```nim
proc params*(someProc: NimNode): NimNode
proc `params=`*(someProc: NimNode; params: NimNode)
```

Получает или заменяет узел `nnkFormalParams` подпрограммы (или типа proc). `params[0]` — тип возврата; последующие дочерние — узлы `nnkIdentDefs`.

---

### `pragma` / `pragma=`

```nim
proc pragma*(someProc: NimNode): NimNode
proc `pragma=`*(someProc: NimNode; val: NimNode)
```

Получает или заменяет узел `nnkPragma` подпрограммы.

---

### `addPragma`

```nim
proc addPragma*(someProc, pragma: NimNode)
```

Добавляет один прагма в список прагм подпрограммы, создавая узел `nnkPragma`, если его ещё нет.

```nim
myProc.addPragma(ident("noSideEffect"))
```

---

### `body` / `body=`

```nim
proc body*(someProc: NimNode): NimNode
proc `body=`*(someProc: NimNode, val: NimNode)
```

Получает или заменяет тело подпрограммы, `block`, `while` или `for` узла. Для подпрограмм — список операторов с индексом 6 в AST.

---

### `basename` / `basename=`

```nim
proc basename*(a: NimNode): NimNode
proc `basename=`*(a: NimNode; val: string)
```

Извлекает или устанавливает идентификатор из узла, который может быть обёрнут в `nnkPrefix`, `nnkPostfix` или `nnkPragmaExpr`. Например, из `foo*` (публичный экспорт) возвращает идентификатор `foo`.

---

### `hasArgOfName`

```nim
proc hasArgOfName*(params: NimNode; name: string): bool
```

Проверяет, есть ли в `nnkFormalParams` параметр с заданным именем.

---

### `addIdentIfAbsent`

```nim
proc addIdentIfAbsent*(dest: NimNode, ident: string)
```

Добавляет `ident` в `dest` (прагму или аналогичный список) только если его там нет. Предотвращает дубликаты при условном добавлении прагм.

---

## Итерация

### `items`

```nim
iterator items*(n: NimNode): NimNode
```

Перебирает прямых дочерних `n`.

```nim
for child in callNode:
  echo child.kind
```

---

### `pairs`

```nim
iterator pairs*(n: NimNode): (int, NimNode)
```

Перебирает прямых дочерних с индексом.

---

### `children`

```nim
iterator children*(n: NimNode): NimNode
```

Эквивалентен `items`. Более явный псевдоним для читаемости.

---

### `findChild`

```nim
template findChild*(n: NimNode; cond: untyped): NimNode
```

Возвращает первого дочернего `n`, удовлетворяющего `cond`, или `nil`. Шаблон вводит переменную `it` — текущий дочерний.

```nim
# Найти первый экспортируемый параметр
let exported = findChild(params, it.kind == nnkPostfix)
```

---

## Квазицитирование и кодогенерация

### `quote`

```nim
proc quote*(bl: typed, op = "``"): NimNode
```

**Квазицитирование**: пишете Nim-код как есть и получаете его AST. Выражения в обратных кавычках внутри блока интерполируют значения `NimNode` из окружающей области видимости.

Это самый читаемый способ получить сложный AST: записываете, как должен выглядеть генерируемый код, затем вставляете переменные части.

```nim
macro assert2(cond: untyped): untyped =
  let msg = newLit("Assertion failed: " & cond.repr)
  result = quote do:
    if not `cond`:
      raise newException(AssertionError, `msg`)
```

Можно задать пользовательский оператор интерполяции, чтобы избежать конфликтов с операторами Nim в цитируемом блоке:

```nim
result = quote("@") do:
  let value = @expr + 1   # @ — оператор интерполяции здесь
```

---

### `getAst`

```nim
proc getAst*(macroOrTemplate: untyped): NimNode
```

Разворачивает вызов макроса или шаблона и возвращает результирующий узел AST, не вставляя его в окружающий код. Полезно для инспекции или дальнейшей трансформации вывода другого макроса.

---

## Валидация

### `expectKind`

```nim
proc expectKind*(n: NimNode, k: NimNodeKind)
proc expectKind*(n: NimNode; k: set[NimNodeKind])
```

Утверждает, что `n` имеет ожидаемый вид (или входит в набор видов). Если нет — компиляция прерывается с описательным сообщением об ошибке, указывающим на позицию `n` в исходном коде. Фундаментальная защита в любом макросе, валидирующем входные данные.

```nim
proc myMacroHelper(n: NimNode) =
  n.expectKind(nnkProcDef)
  # теперь безопасно читать n[6] (тело)
```

---

### `expectLen`

```nim
proc expectLen*(n: NimNode, len: int)
proc expectLen*(n: NimNode, min, max: int)
```

Утверждает, что `n` имеет ровно `len` дочерних, или от `min` до `max`. Иначе прерывает компиляцию с ошибкой.

---

### `expectMinLen`

```nim
proc expectMinLen*(n: NimNode, min: int)
```

Утверждает, что `n` имеет не менее `min` дочерних.

---

### `expectIdent`

```nim
proc expectIdent*(n: NimNode, name: string)
```

Утверждает, что `eqIdent(n, name)` истинно. Для проверки наличия конкретного ключевого слова или имени в нужном месте входного AST.

---

## Диагностика во время компиляции

### `error`

```nim
proc error*(msg: string, n: NimNode = nil)
```

Прерывает компиляцию с сообщением об ошибке. Необязательный `n` задаёт позицию в исходном коде для стрелки в выводе компилятора. Именно так макросы сообщают об ошибках использования.

```nim
if n.kind != nnkProcDef:
  error("Ожидалось определение proc", n)
```

---

### `warning`

```nim
proc warning*(msg: string, n: NimNode = nil)
```

Выводит предупреждение компилятора без прерывания.

---

### `hint`

```nim
proc hint*(msg: string, n: NimNode = nil)
```

Выводит подсказку компилятора (информационное сообщение).

---

## Визуализация и отладка AST

Эти инструменты незаменимы при написании макросов: они показывают точную структуру AST, которую порождает фрагмент кода, — именно её должен воспроизводить или обрабатывать ваш макрос.

### `treeRepr`

```nim
proc treeRepr*(n: NimNode): string
```

Преобразует дерево узлов в удобочитаемый текст с отступами. Показывает виды и значения.

```
StmtList
  VarSection
    IdentDefs
      Ident "x"
      Empty
      IntLit 42
```

---

### `lispRepr`

```nim
proc lispRepr*(n: NimNode; indented = false): string
```

S-выражение AST. Компактнее `treeRepr`; вложенность обозначается скобками.

---

### `astGenRepr`

```nim
proc astGenRepr*(n: NimNode): string
```

Преобразует AST в **код Nim, который строит его** с помощью `newTree`, `newLit` и т. д. Скопируйте вывод как отправную точку для макроса.

```nim
nnkVarSection.newTree(
  nnkIdentDefs.newTree(
    newIdentNode("x"),
    newEmptyNode(),
    newLit(42)
  )
)
```

---

### `dumpTree`

```nim
macro dumpTree*(s: untyped): untyped
```

Выводит `treeRepr` своего аргумента **во время компиляции**. Первый и важнейший шаг при написании нового макроса: оберните код, который вы хотите обрабатывать, в `dumpTree`, чтобы увидеть, какой AST он порождает.

```nim
dumpTree:
  for i in 0 ..< 10:
    echo i
```

---

### `dumpLisp`

```nim
macro dumpLisp*(s: untyped): untyped
```

Выводит `lispRepr` во время компиляции.

---

### `dumpAstGen`

```nim
macro dumpAstGen*(s: untyped): untyped
```

Выводит `astGenRepr` во время компиляции. Используйте, чтобы получить готовую отправную точку для ручного построения того же AST.

---

### `expandMacros`

```nim
macro expandMacros*(body: typed): untyped
```

Выводит развёрнутый (пост-макросный) код `body` во время компиляции, не изменяя то, что компилируется. Полезно для проверки вывода другого макроса.

```nim
expandMacros:
  dump(x + y)   # выводит развёртывание `dump` во время компиляции
```

---

## Позиция в исходном коде

### `lineInfo`

```nim
proc lineInfo*(arg: NimNode): string
```

Возвращает `"файл(строка, столбец)"` для позиции данного узла в исходном коде. Используйте в сообщениях об ошибках, чтобы указать пользователю на нужную строку.

---

### `lineInfoObj`

```nim
proc lineInfoObj*(n: NimNode): LineInfo
```

Возвращает структурированный объект `LineInfo` (файл, строка, столбец) для `n`.

---

### `copyLineInfo`

```nim
proc copyLineInfo*(arg: NimNode, info: NimNode)
```

Копирует позицию из `info` в `arg`. Используйте, чтобы генерируемые узлы указывали на правильное место в пользовательском исходном коде.

---

### `setLineInfo`

```nim
proc setLineInfo*(arg: NimNode, file: string, line: int, column: int)
proc setLineInfo*(arg: NimNode, lineInfo: LineInfo)
```

Явно устанавливает позицию в исходном коде для узла. При использовании `quote` добавляйте информацию о позиции **после** блока quote, так как `quote` её перезаписывает.

---

## Разбор строк в AST

### `parseExpr`

```nim
proc parseExpr*(s: string; filename: string = ""): NimNode
```

Разбирает строку с Nim-выражением в узел AST. При ошибках парсинга поднимает `ValueError`. Полезно для динамической кодогенерации из строковых шаблонов, хотя `quote` обычно удобнее.

---

### `parseStmt`

```nim
proc parseStmt*(s: string; filename: string = ""): NimNode
```

Как `parseExpr`, но принимает один или несколько операторов.

---

### `toStrLit`

```nim
proc toStrLit*(n: NimNode): NimNode
```

Преобразует `NimNode` в строку его исходного кода через `repr` и оборачивает в узел строкового литерала. Полезно для встраивания строкового представления узла в генерируемый код.

---

## `$` — узел в строку

```nim
proc `$`*(node: NimNode): string
```

Извлекает строковое имя из узлов идентификатора, символа, строкового литерала, quoted и postfix. Для постфикс-маркера экспорта (`foo*`) добавляет `"*"`. Для неподходящих видов поднимает ошибку. Используйте `strVal` для литеральных узлов; `$` — для идентификаторов и символов.

---

## Интроспекция прагм

### `hasCustomPragma`

```nim
macro hasCustomPragma*(n: typed, cp: typed{nkSym}): untyped
```

Во время компиляции возвращает `true`, если `n` (поле, proc или тип) несёт пользовательскую прагму `cp`. Проверка работает даже если прагма имеет аргументы.

```nim
template myTag() {.pragma.}
type Foo = object
  tagged {.myTag.}: int

var o: Foo
assert o.tagged.hasCustomPragma(myTag)
```

---

### `getCustomPragmaVal`

```nim
macro getCustomPragmaVal*(n: typed, cp: typed{nkSym}): untyped
```

Извлекает **значение** пользовательской прагмы с аргументами. Один аргумент — возвращает его напрямую. Несколько аргументов — возвращает именованный кортеж. Поднимает ошибку компиляции, если прагма не найдена.

```nim
template key(s: string) {.pragma.}
type Foo = object
  field {.key: "f".}: int

var o: Foo
assert o.field.getCustomPragmaVal(key) == "f"
```

---

## Прочее

### `unpackVarargs`

```nim
macro unpackVarargs*(callee: untyped; args: varargs[untyped]): untyped
```

Вызывает `callee`, передавая каждый элемент `args` как отдельный аргумент. Решает проблему проброса `varargs` через промежуточный шаблон или макрос, включая граничный случай пустого `varargs`.

```nim
template forward(fn: typed; args: varargs[untyped]): untyped =
  unpackVarargs(fn, args)
```

---

### `getProjectPath`

```nim
proc getProjectPath*(): string
```

Возвращает директорию **корневого файла** компиляции (файла, переданного команде `nim`). В отличие от `currentSourcePath`, который возвращает файл с местом вызова. Полезно для поиска соседних ресурсных файлов в проекте.

---

### `extractDocCommentsAndRunnables`

```nim
proc extractDocCommentsAndRunnables*(n: NimNode): NimNode
```

Для узла тела подпрограммы возвращает `nnkStmtList` только с ведущими doc-комментариями (`## …`) и `runnableExamples`, останавливаясь при первом операторе, который не является ни тем, ни другим. Используется трансформерами кода на основе макросов для сохранения документации.

---

### `nodeID`

```nim
proc nodeID*(n: NimNode): int
```

Возвращает внутренний ID компилятора для `n`, если компилятор собран с `-d:useNodeids`. Иначе возвращает `-1`. Предназначен исключительно для отладки компилятора.

---

## Рабочий процесс написания макроса

```
1. Оберните тестовый ввод в dumpTree — посмотрите, какой AST получит ваш макрос.
2. В начале макроса напишите валидацию (expectKind, expectLen).
3. Стройте вывод через quote do: … для читаемости,
   или через newTree / newCall / newLit для точного контроля.
4. Используйте genSym для любых временных переменных во избежание проблем гигиены.
5. Используйте bindSym вместо ident там, где символ должен ссылаться на конкретное
   определение вне зависимости от перегрузок в месте вызова.
6. Проверяйте вывод через expandMacros или echo result.treeRepr.
```

### Минимальный рабочий пример

```nim
import std/macros

macro repeat*(n: static int, body: untyped): untyped =
  ## Повторяет `body` ровно `n` раз во время компиляции.
  result = newStmtList()
  for _ in 0 ..< n:
    result.add(body.copyNimTree)  # глубокая копия — каждая итерация независима

repeat(3):
  echo "hello"
# Разворачивается в:
#   echo "hello"
#   echo "hello"
#   echo "hello"
```

### Инспекция генерируемого кода

```nim
import std/macros

dumpTree:
  if x > 0: echo "positive"

# Выводит во время компиляции:
# IfStmt
#   ElifBranch
#     Infix
#       Ident ">"
#       Ident "x"
#       IntLit 0
#     StmtList
#       Command
#         Ident "echo"
#         StrLit "positive"
```
