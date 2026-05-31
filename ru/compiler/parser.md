# Парсер XCX — v3.1

Парсер XCX преобразует поток токенов в дерево абстрактного синтаксиса (AST) высокого уровня.

## Архитектура: разбор Pratt

XCX использует **парсер Pratt** (Top-Down Operator Precedence).

- **Файл**: `src/parser/pratt.rs`
- **Заглядывание**: один токен (`current` + `peek`), продвижение через `advance()`.
- **Восстановление после ошибок**: при синтаксической ошибке `synchronize()` пропускает токены до `;` или известного начала инструкции (`func`, `fiber`, `if`, `for`, `const`, `return`, `>!` и т.д.).

Структура `Parser` заимствует исходную строку на время жизни `'a`; `Scanner<'a>` использует то же время жизни — отражение байтового сканера.

### Уровни приоритета (от низшего к высшему)

| Уровень        | Операторы |
|----------------|----------|
| `Lowest`       | —        |
| `Lambda`       | `->`     |
| `Assignment`   | `=`      |
| `LogicalOr`    | `OR`, `\|\|` |
| `LogicalAnd`   | `AND`, `&&` |
| `Equals`       | `==`, `!=` |
| `LessGreater`  | `>`, `<`, `>=`, `<=`, `HAS` |
| `Sum`          | `+`, `-`, `++` |
| `SetOp`        | `UNION`, `INTERSECTION`, `DIFFERENCE`, `SYMMETRIC_DIFFERENCE` |
| `Product`      | `*`, `/`, `%` |
| `Power`        | `^`      |
| `Prefix`       | `-x`     |
| `Call`         | `.`, `[` |

## Диспетчеризация инструкций

`parse_statement_internal()` выбирает разбор по текущему токену:

- **Ключевые слова типов** (`i`, `f`, `s`, `b`, `array`, `set`, `map`, `date`, `table`, `json`) → `parse_var_decl()` или `parse_assignment()` при `=`
- **`const`** → `parse_var_decl()` с `is_const = true`
- **`var`** (идентификатор) → объявление с выводом типа
- **`>!`** → `parse_print_stmt()`
- **`>?`** → `parse_input_stmt()`
- **`halt`** → `parse_halt_stmt()`
- **`if`** → `parse_if_statement()`
- **`while`** → `parse_while_statement()`
- **`for`** → `parse_for_statement()`
- **`break`** / **`continue`** → соответствующие парсеры
- **`func`** → `parse_func_def()`
- **`fiber`** → `parse_fiber_statement()` (определение или объявление по peek)
- **`return`** → `parse_return_stmt()`
- **`yield`** → `parse_yield_stmt()` (`yield expr`, `yield from expr`, `yield;`)
- **`@wait`** → `parse_wait_stmt()`
- **`serve`** → `parse_serve_stmt()`
- **`net`** → `parse_net_stmt()`
- **`include`** → `parse_include_stmt()`
- **Идентификатор + `=`** → `parse_assignment()`
- **Идентификатор + `(`** → `parse_func_call_stmt()`
- **Иное** → `parse_expr_stmt()`

## Стили определения функций

**Стиль со скобками** (как в C):
```xcx
func name(i: x, s: y -> i) {
    return x + 1;
}
```

**Стиль XCX** (блок по ключевым словам):
```xcx
func:i: name(i: x, s: y) do;
    return x + 1;
end;
```

Оба дают одинаковый узел `StmtKind::FunctionDef`. В стиле со скобками тип возврата — `-> type` в списке параметров или после `)`.

## Инструкции с файберами

`parse_fiber_statement()` смотрит на `peek`:
- `peek == Colon` → `parse_fiber_decl()` (экземпляр: `fiber:T: varname = fiberDef(args);`)
- иначе → `parse_fiber_def()` (определение: `fiber name(params) { body }`)

`parse_fiber_decl()` также обрабатывает случай, когда после типа и имени идёт `(` — тогда вызывается `finish_fiber_def()` (определение с префиксом `fiber:`).

## Основные разбираемые конструкции

- **Объявления переменных**: `i: name = expr;`, `const s: NAME = expr;`, `var name = expr;`
- **Управление потоком**: `if (cond) then; ... elseif ... else; ... end;`
- **Цикл while**: `while (cond) do; ... end;`
- **Цикл for**: `for x in expr do; ... end;` и `for x in start to end @step n do; ... end;`
- **Функции и файберы** (два стиля)
- **Yield**: `yield expr;`, `yield from expr;`, `yield;`
- **HTTP**: `serve:`, `net.get`, `net.request { ... } as resp;`, `net.respond`
- **Коллекции**: массив, множество, карта, таблица
- **Сырые блоки**: `<<<...>>>`
- **Include**: `include "path";` или `include "path" as alias;`
- **Ввод-вывод**: `>!`, `>?`
- **Halt**: `halt.alert`, `halt.error`, `halt.fatal`
- **Wait**: `@wait(ms);`
- **Даты**: `date("2024-01-01")` или с форматом

## Разбор выражений

`parse_expression(precedence)` вызывает `parse_prefix()`, затем в цикле `parse_infix(left)`, пока приоритет peek выше текущего минимума.

Префиксные разборы:
- **Идентификаторы**: при `(` — `FunctionCall`, иначе `Identifier`
- **Литералы**, унарный минус как `Binary { left: 0, op: Minus, right }`
- **`not` / `!`**: `Unary`
- **`(`...`)`**: одно выражение или `Tuple`
- **`[`...`]`**: при `::` — `MapLiteral`, иначе `ArrayLiteral`
- **`{`...`}`**: `ArrayOrSetLiteral`
- **`set:N { }`**: `SetLiteral`
- **`map { }`**, **`table { }`**, **`random.choice from`**, **`date(...)`**, **`net.*`**, **`<<<>>>`**, **`.terminal!cmd`**

Инфиксные: **`.`** → `MethodCall` или `MemberAccess`; **`[`** → `Index`; **`->`** → `Lambda`; остальные — `Binary`.

## Постобработка `parse_expr_stmt()`

После разбора выражения-инструкции:
- `bind` с 2 аргументами, второй — `Identifier` → `StmtKind::JsonBind`
- `inject` с 2 аргументами → `StmtKind::JsonInject`

Это даёт синтаксис `json.bind("path", target);` и `json.inject(mapping, table);` на уровне инструкций.

## Expander (`src/parser/expander.rs`)

Выполняется **после** разбора, **до** семантики — отдельный проход переписывания дерева.

### Обязанности

**Разрешение include**: `include "file.xcx";` заменяется встроенным AST файла. Циклы — `visiting_files`. Дедупликация — `included_files`.

**Префиксы псевдонимов**: `include "math.xcx" as math;` переименовывает символы в `math.name`. `math.sin(x)` → `FunctionCall { name: "math.sin" }`.

**Префиксы имён файберов** и **YieldFrom** — аналогично для `FiberDecl` и `yield from`.

**Защищённые имена** (без префикса): `json`, `date`, `store`, `halt`, `terminal`, `net`, `env`, `crypto`, `EMPTY`, `math`, `random`, `i`, `f`, `s`, `b`, `from`, `main`.

**Порядок поиска include**:
1. Относительно каталога текущего файла
2. Каталог `lib/` (от CWD, затем вверх от пути к исполняемому файлу)

## Определения AST (`src/parser/ast.rs`)

### `Expr`

| Вариант | Описание |
|---------|----------|
| `IntLiteral`, `FloatLiteral`, `StringLiteral`, `BoolLiteral` | Литералы |
| `Identifier` | Имя переменной или функции |
| `Binary`, `Unary` | Операции |
| `FunctionCall`, `MethodCall`, `MemberAccess`, `Index` | Вызовы и доступ |
| `Lambda` | `x -> expr` |
| `ArrayLiteral`, `ArrayOrSetLiteral`, `SetLiteral`, `MapLiteral`, `TableLiteral` | Коллекции |
| `DateLiteral`, `Tuple`, `NetCall`, `NetRespond`, `RawBlock`, `TerminalCommand`, `RandomChoice` | Прочее |

### `Stmt`

Основные варианты: `VarDecl`, `Assign`, `Print`, `Input`, `If`, `While`, `For`, `Break`, `Continue`, `FunctionDef`, `FiberDef`, `FiberDecl`, `Return`, `Yield`, `YieldFrom`, `YieldVoid`, `Include`, `Serve`, `NetRequestStmt`, `JsonBind`, `JsonInject`, `Halt`, `Wait`, `ExprStmt`, `FunctionCallStmt`.

### `Type`

`Int`, `Float`, `String`, `Bool`, `Date`, `Json`, `Array`, `Set`, `Map`, `Table`, `Fiber`, `Builtin`, `Unknown`.

`SetType`: `N`, `Z`, `Q`, `S`, `C`, `B`.

### `ForIterType`

`Range`, `Array`, `Set`, `Fiber` — задаётся checker и используется компилятором для эмиссии цикла.

### `ColumnDef`

```rust
pub struct ColumnDef {
    pub name:    StringId,
    pub ty:      Type,
    pub is_auto: bool,    // @auto columns are auto-incremented on insert
}
```

## Интернирование строк

Все строки интернируются в `StringId (u32)` через `Interner` — checker, compiler и VM сравнивают числовые ID, а не `String`.
