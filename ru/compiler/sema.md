# Семантический анализ XCX (Sema) — документация

> **Файлы:** `src/sema/checker.rs`, `src/sema/symbol_table.rs`, `src/sema/interner.rs`

---

## Содержание

1. [Обзор](#обзор)
2. [Интернирование строк](#интернирование-строк)
3. [Таблица символов](#таблица-символов)
4. [Проверка типов](#проверка-типов)
5. [Правила совместимости типов](#правила-совместимости-типов)
6. [Коды ошибок](#коды-ошибок)
7. [Отчёт об ошибках](#отчёт-об-ошибках)

---

## Обзор

Фаза Sema проверяет AST на логическую корректность и типы до генерации байткода. Три компонента:

```
Interner → StringId (u32)
     ↓
SymbolTable → иерархические области типов
     ↓
Checker → накопление TypeErrors
```

Компиляция только если `Vec<TypeError>` пуст.

---

## Интернирование строк

**Файл:** `src/sema/interner.rs`

`Interner` сопоставляет `&str → StringId(u32)`. Единый источник идентичности строк. Создаётся при лексировании/разборе, передаётся в checker и compiler.

```rust
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

**Реализация:**

```rust
pub struct Interner {
    map:     HashMap<String, StringId>,
    strings: Vec<String>,
}
```

Каждая уникальная строка хранится один раз в `strings`. Далее — числовые ID без сравнений строк в куче.

---

## Таблица символов

**Файл:** `src/sema/symbol_table.rs`

`SymbolTable` управляет привязками во вложенных областях через **цепочку родительских указателей**.

### Структура

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // ref to surrounding scope
    scopes: Vec<HashMap<String, Type>>,   // stack of scope frames (local)
    consts: Vec<HashSet<String>>,         // which names are const, per frame
}
```

### Дочерняя область

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

O(1) — один пустой `HashMap`. Поиск поднимается по `parent`.

### Жизненный цикл области

| Метод | Описание |
|-------|----------|
| `enter_scope()` | Новый кадр — для `if`, `while`, `for` |
| `exit_scope()` | Выход из области |
| `new_with_parent(parent)` | Дочерняя таблица — тела функций/файберов |
| `define(name, ty, is_const)` | Запись во **внутреннюю** область |
| `lookup(name)` | От внутренней к внешней, затем `parent` |
| `has_in_current_scope(name)` | Только внутренний кадр — переопределение |
| `is_const(name)` | Проверка `const` в кадре объявления |

### Затенение запрещено

Повторное объявление в **текущей** области — `RedefinedVariable`. Имена родителя видны, но не переобъявляются в дочерней области.

---

## Проверка типов

**Файл:** `src/sema/checker.rs`

`Checker` обходит AST и накапливает `TypeError`.

### Состояние Checker

```rust
pub struct Checker<'a> {
    interner:          &'a Interner,
    loop_depth:        usize,
    functions:         HashMap<String, FunctionSignature>,
    fiber_context:     Option<Option<Type>>,  // None=outside fiber, Some(None)=void, Some(Some(T))=typed
    is_table_lambda:   bool,
    fiber_has_yield:   bool,
    in_yield_expr:     bool,
    last_expr_was_db_io: bool,
}
```

### Флаги контекста

| Поле | Назначение |
|------|------------|
| `loop_depth` | Вложенность циклов; 0 → ошибка для `break`/`continue` |
| `fiber_context` | Контекст файбера (вне / void / typed) |
| `fiber_has_yield` | Был ли yield в теле |
| `is_table_lambda` | Предикат `.where()`, столбцы через `__row_tmp` |
| `in_yield_expr` | Внутри выражения yield |
| `last_expr_was_db_io` | Флаг операций I/O БД |

### Предварительное сканирование

Сканирование `FunctionDef` и `FiberDef` в текущем списке инструкций — вызов до определения, взаимная рекурсия. Регистрация сигнатур и встроенных приведений `i`, `f`, `s`, `b`.

### Правила вывода типов

- Снизу вверх от литералов
- `Unknown` — подстановка
- `Json` совместим с любым типом
- `Int op Float → Float`
- Пустой `[]` наследует тип из контекста
- `Table([])` совместим с любым `Table(cols)`

---

## Правила совместимости типов

`is_compatible(expected, actual)`:

| Правило | Описание |
|---------|----------|
| Любой `Unknown` | Совместимо |
| Любой `Json` | Совместимо |
| `Int` ↔ `Float` | Числовое повышение |
| `Int` ↔ `Date` | Совместимо |
| `Set` ↔ `Array` | При совпадении элемента |
| `Set(N)` ↔ `Set(Z)` | Целочисленные множества |
| `Set(S)` ↔ `Set(C)` | Строковые |
| `Table([])` ↔ `Table(cols)` | Пустой список столбцов |
| `Table(a)` ↔ `Table(b)` | Попарно по столбцам |
| `Map` | Рекурсивно |
| `Fiber` | По правилам void/typed |

---

## Коды ошибок

| Код | Условие |
|-----|---------|
| `[S101] UndefinedVariable(name)` | Использование до объявления |
| `[S102] RedefinedVariable(name)` | Двойное объявление |
| `[S103] TypeMismatch` | Несовпадение типов |
| `[S104] InvalidBinaryOp` | Несовместимые типы для оператора |
| `[S105] ConstReassignment(name)` | Присваивание константе |
| `[S106] BreakOutsideLoop` | `break` вне цикла |
| `[S107] ContinueOutsideLoop` | `continue` вне цикла |
| `[S108] IndexAccessNotSupported` | Индексация неподдерживаемого типа |
| `[S109] PropertyNotFound` | Свойство не найдено |
| `[S110] MethodNotFound` | Метод не найден |
| `[S111] InvalidArgumentCount` | Неверное число аргументов |
| `[S208] YieldOutsideFiber` | `yield` вне файбера |
| `[S209] FiberTypeMismatch` | `yield expr` в пустом файбере |
| `[S210] ReturnTypeMismatchInFiber` | `return;` без значения в typed-файбере |
| `[S211] CannotIterateOverVoidFiber` | Итерация по void-файберу |
| `[S212] CannotRunTypedFiber` | `.run()` на typed-файбере |
| `[S301] WherePredicateNameCollision` | Конфликт имён в `.where()` |
| `[S302] TableRowCountMismatch` | Число значений строки ≠ схеме |
| `[D401] Rule violation` | `remove()` без `.where()` |
| `Other(msg)` | Прочие ошибки |

---

## Отчёт об ошибках

`TypeError` с `Span`. Пример вывода:

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

При любой ошибке байткод не генерируется.

---

## Проверка вызовов функций

1. `self.functions`
2. Таблица символов
3. Файбер → `Type::Fiber(Some(ret))`
4. Лишние аргументы допускаются
5. Иначе `UndefinedVariable`

---

## Проверка `.where()` у таблиц

1. `__row_tmp: Table(cols)`
2. `is_table_lambda = true`
3. Сбор идентификаторов предиката
4. Конфликт с столбцом → S301
5. Тип предиката — `Bool`
6. Сброс флага и выход из области
