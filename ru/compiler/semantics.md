# Семантический анализ XCX (Sema) — v3.1

Фаза Sema проверяет AST на логическую корректность и согласованность типов до генерации байткода. Состоит из **таблицы символов** и **проверки типов**.

## Интернирование строк (`src/sema/interner.rs`)

`Interner` сопоставляет `&str → StringId(u32)`. Единый источник идентичности строк в компиляторе. Создаётся при лексировании/разборе, передаётся в checker и compiler.

```
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

## Таблица символов (`src/sema/symbol_table.rs`)

`SymbolTable` управляет привязками переменных во вложенных областях через **цепочку родительских указателей**.

### Структура

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // reference to enclosing scope
    scopes: Vec<HashMap<String, Type>>,   // stack of scope frames (local to this table)
    consts: Vec<HashSet<String>>,         // which names are const, per scope frame
}
```

### Дочерняя область

При входе в тело функции или файбера создаётся новая `SymbolTable` со ссылкой на внешнюю:

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

Это O(1) — выделяется один пустой `HashMap`. Поиск поднимается по цепочке `parent`.

### Жизненный цикл области

- `enter_scope()` / `exit_scope()` — для тел `if`, `while`, `for`.
- `new_with_parent(parent)` — для тел функций и файберов.
- `define(name, ty, is_const)` — только во **внутренней** области текущей таблицы.
- `lookup(name)` — от внутренней к внешней, затем по `parent`.
- `has_in_current_scope(name)` — только внутренний кадр — для `RedefinedVariable`.
- `is_const(name)` — проверка флага `const` в кадре, где объявлено имя.

### Важно: затенение запрещено

XCX **не** поддерживает затенение. Повторное объявление в **текущей** области — `RedefinedVariable`. Переменные родителя видны, но переобъявить то же имя в дочерней области нельзя.

## Проверка типов (`src/sema/checker.rs`)

`Checker` обходит AST и накапливает `TypeError`. Компиляция только если `Vec<TypeError>` пуст.

### Состояние Checker

```rust
pub struct Checker<'a> {
    interner:       &'a Interner,
    loop_depth:     usize,
    functions:      HashMap<String, FunctionSignature>,
    fiber_context:  Option<Option<Type>>,
    is_table_lambda: bool,
    fiber_has_yield: bool,
}
```

### Предварительное сканирование (`pre_scan_stmts`)

Перед проверкой тел выполняется **сканирование** всех `FunctionDef` и `FiberDef` в **текущем** списке инструкций (не рекурсивно). Это позволяет вызывать функции до их определения в файле.

Для каждой функции/файбера регистрируется:
- `FunctionSignature { params, return_type, is_fiber }` в `self.functions`
- запись в `SymbolTable` с `Type::Unknown` или `Type::Fiber(...)`

Встроенные приведения `i`, `f`, `s`, `b` предрегистрируются с соответствующими типами возврата.

### Флаги контекста

| Поле | Назначение |
|------|------------|
| `loop_depth` | Вложенность `while`/`for`. Ноль → ошибка для `break`/`continue`. Сбрасывается в теле файбера. |
| `fiber_context` | `None` — не в файбере; `Some(None)` — пустой файбер; `Some(Some(T))` — типизированный с yield `T`. |
| `fiber_has_yield` | Был ли `yield` в теле файбера. |
| `is_table_lambda` | Внутри предиката `.where()`; голые имена столбцов через `__row_tmp`. |

### Правила вывода типов

- Типы выражений снизу вверх от литералов.
- `Type::Unknown` — подстановочный тип.
- `Type::Json` совместим с любым типом в присваиваниях и сравнениях.
- Числовое повышение: `Int op Float → Float`.
- Пустой массив `[]` наследует тип из контекста присваивания.
- `Type::Table([])` совместим с любым `Table(cols)` — столбцы дополняются после вывода.

### `is_compatible(expected, actual)`

| Правило | Описание |
|---------|----------|
| Любой `Unknown` | Всегда совместимо |
| Любой `Json` | Всегда совместимо |
| `Int` ↔ `Float` | Взаимно (числовое повышение) |
| `Int` ↔ `Date` | Совместимо |
| `Set(X)` ↔ `Array(inner)` | При совпадении элемента |
| `Set(N)` ↔ `Set(Z)` | Оба целочисленные |
| `Set(S)` ↔ `Set(C)` | Оба строковые |
| `Table([])` ↔ `Table(cols)` | Если один список столбцов пуст |
| `Table(a)` ↔ `Table(b)` | Попарная совместимость столбцов |
| `Map` | Рекурсивно |
| `Fiber` | По правилам void/typed и совместимости внутреннего типа |

### Проверка вызовов функций

Для `ExprKind::FunctionCall`:
1. Поиск в `self.functions`.
2. Иначе в таблице символов (значения функций первого класса).
3. Если сигнатура — файбер, вызов даёт `Type::Fiber(Some(ret))` (создание экземпляра).
4. Лишние аргументы допускаются (вариадичная толерантность).
5. Не найдено → `UndefinedVariable`.

Для `StmtKind::FunctionCallStmt`:
- Несовпадение числа аргументов → `Other(...)`.
- Неизвестное имя → `UndefinedVariable`.

### Проверка объявлений файберов (`FiberDecl`)

1. Поиск `fiber_name` в `functions` или символах.
2. `is_fiber: true`.
3. Сопоставление типов аргументов.
4. Новая переменная с `Type::Fiber(inner_type)`.

### Проверка `.where()` у таблиц

1. Временная область, `__row_tmp: Table(cols)`.
2. `is_table_lambda = true`.
3. `collect_pred_idents()` — идентификаторы в предикате.
4. Имя и в внешней области, и среди столбцов → `S301 WherePredicateNameCollision`.
5. Предикат должен иметь тип `Bool`.
6. Сброс флага и выход из области.

### Проверка `Table.join()`

Объединение столбцов: левые + правые, отсутствующие в левой таблице. При конфликте имён сохраняется правый столбец. Результат: `Type::Table(combined_cols)`.

### Проверка циклов `for`

Поле `iter_type` в `ForIterType` уточняется по типу `start`:
- `Array` → переменная цикла — внутренний тип массива
- `Set` → элемент множества
- `Table` → как массив, переменная — `Table(cols)`
- `Fiber` → тип yield
- `Int` с `to` → `Range`, переменная — `Int`

---

## Коды ошибок

| Код | Условие |
|-----|---------|
| `UndefinedVariable(name)` | Использование до объявления |
| `RedefinedVariable(name)` | Двойное объявление в одной области |
| `ConstReassignment(name)` | Присваивание константе |
| `TypeMismatch { expected, actual }` | Несовпадение типов |
| `InvalidBinaryOp { op, left, right }` | Несовместимые типы для оператора |
| `BreakOutsideLoop` | `break` вне цикла |
| `ContinueOutsideLoop` | `continue` вне цикла |
| `[S208] YieldOutsideFiber` | `yield` вне файбера |
| `[S209] FiberTypeMismatch` | `yield expr` в пустом файбере |
| `[S210] ReturnTypeMismatchInFiber` | `return;` без значения в типизированном файбере |
| `[S301] WherePredicateNameCollision` | Конфликт имени столбца и локальной переменной в `.where()` |
| `Other(msg)` | Прочие контекстные ошибки |

---

## Отчёт об ошибках

`TypeError` несёт `Span`. `Reporter::error()` выводит сообщение, строку и `~~~`. При любой ошибке байткод не генерируется.
