# Expander XCX — документация

> **Файл:** `src/parser/expander.rs`  
> Выполняется **после** разбора, **до** семантического анализа.

---

## Содержание

1. [Обзор](#обзор)
2. [Обязанности](#обязанности)
3. [Разрешение include](#разрешение-include)
4. [Префиксы псевдонимов](#префиксы-псевдонимов)
5. [Защищённые имена](#защищённые-имена)
6. [Порядок поиска include](#порядок-поиска-include)

---

## Обзор

Expander — отдельный проход переписывания AST после парсера и до семантики. Обрабатывает `include` и `include ... as alias`.

```rust
pub struct Expander<'a> {
    interner:       &'a mut Interner,
    visiting_files: HashSet<PathBuf>,   // for circular dependency detection
    included_files: HashSet<PathBuf>,   // deduplication (each file once)
    aliases:        HashMap<StringId, String>,
    include_paths:  Vec<PathBuf>,       // additional search paths
}
```

---

## Обязанности

### 1. Разрешение include
`include "file.xcx";` заменяется встроенным AST этого файла.

- Циклические зависимости — `visiting_files: HashSet<PathBuf>`
- Дедупликация — `included_files` (каждый файл один раз, кроме случаев с псевдонимом)

### 2. Префиксы псевдонимов
`include "math.xcx" as math;` добавляет префикс `math.` ко всем символам верхнего уровня. `math.sin(x)` переписывается из `MethodCall` в `FunctionCall { name: "math.sin" }` через `expand_expr_inplace`.

Функции `prefix_program()` / `prefix_stmt_impl()` / `prefix_expr_impl()` обходят под-AST и переименовывают ссылки на символы верхнего уровня.

### 3. Префиксы имён файберов
Ссылки `FiberDecl::fiber_name` тоже получают префикс для корректного создания экземпляров после переименования.

### 4. Префиксы YieldFrom
В `StmtKind::YieldFrom` вызовы конструкторов файберов внутри `yield from` также переименовываются.

---

## Разрешение include

```xcx
include "utils.xcx";           --- Simple include (deduplicated)
include "math.xcx" as math;    --- Aliased include
```

После include с псевдонимом:
- все символы из `math.xcx` получают префикс `math.`
- вызов `math.sin(x)` → `FunctionCall("math.sin", [3.14])`

Пример:

```xcx
--- math.xcx defines:
func sin(f: x -> f) { ... }
func cos(f: x -> f) { ... }

--- After include "math.xcx" as math:
--- sin → math.sin
--- cos → math.cos
--- Call math.sin(3.14) → FunctionCall("math.sin", [3.14])
```

---

## Префиксы псевдонимов

Алгоритм `prefix_stmt_impl` / `prefix_expr_impl`:

1. Собирает имена верхнего уровня (`top_level_names`)
2. Если идентификатор в этом наборе — замена на `prefix.name`
3. Особые случаи: `FiberDecl::fiber_name`, `YieldFrom`, `MethodCall` на объекте с псевдонимом → `FunctionCall`

---

## Защищённые имена

Следующие имена **никогда** не получают префикс:

```
json    date    store   halt    terminal
net     env     crypto  EMPTY   math
random  i       f       s       b
from    main
```

---

## Порядок поиска include

1. Относительно каталога текущего файла
2. Каталог `lib/` (от CWD, затем вверх от исполняемого файла)

Дополнительные пути:

```rust
expander.add_include_path(path);
```

В `main.rs` автоматически добавляется `lib/` относительно CWD:

```rust
if let Ok(cwd) = std::env::current_dir() {
    let lib_path = cwd.join("lib");
    if lib_path.exists() {
        expander.add_include_path(lib_path);
    }
}
```

---

## Обнаружение ошибок

| Ошибка | Описание |
|--------|----------|
| `Circular dependency` | Цикл include (`visiting_files`) |
| `File not found` | Файл не найден ни в одном пути поиска |
| `Could not read file` | Ошибка I/O при чтении |
