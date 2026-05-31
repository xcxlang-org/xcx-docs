# XCX 语义分析（Sema）— v3.1

Sema 阶段在字节码生成前验证 AST 的逻辑正确性与类型一致性。它由两个组件组成：**符号表**与**类型检查器**。

## String Interner（`src/sema/interner.rs`）

`Interner` 映射 `&str → StringId(u32)`。它是编译器中所有字符串标识的唯一来源。在词法/解析阶段创建，通过引用传递给检查器与编译器。

```
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

## 符号表（`src/sema/symbol_table.rs`）

`SymbolTable` 使用**父指针链接链**管理嵌套作用域中的变量绑定。

### 结构

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // reference to enclosing scope
    scopes: Vec<HashMap<String, Type>>,   // stack of scope frames (local to this table)
    consts: Vec<HashSet<String>>,         // which names are const, per scope frame
}
```

### 创建子作用域

进入函数或 fiber 体时，创建引用外围表的新 `SymbolTable`，而非克隆：

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

这是 O(1)——仅为新帧分配单个空 `HashMap`。查找按需沿父链向上遍历。

### 作用域生命周期

- `enter_scope()` / `exit_scope()` 在表内 push/pop 局部帧——用于 `if`、`while`、`for` 体。
- `new_with_parent(parent)` 创建链接到父表的子表——用于函数与 fiber 体。
- `define(name, ty, is_const)` 始终写入当前表的**最内层**（当前）作用域。
- `lookup(name)` 从最内层到最外层遍历局部作用域，然后沿 `parent` 指针链继续。
- `has_in_current_scope(name)` 仅检查当前表的最内层帧——用于检测同一块内的重定义。
- `is_const(name)` 找到拥有 `name` 的作用域，然后检查 `consts[scope_index]`。

### 重要：不支持变量遮蔽

XCX **不支持**变量遮蔽。在**当前作用域**中定义已存在的变量会触发 `RedefinedVariable`。父作用域中的变量可访问，但不能在子作用域中用相同名称重新声明。

## 类型检查器（`src/sema/checker.rs`）

`Checker` 结构体遍历 AST 并累积 `TypeError` 值。仅当结果 `Vec<TypeError>` 为空时才编译程序。

### Checker 状态

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

### 预扫描遍（`pre_scan_stmts`）

在检查任何语句体之前，检查器对**当前语句列表**（非递归）中所有 `FunctionDef` 与 `FiberDef` 节点执行**前向声明扫描**。这允许函数与 fiber 在源文件中定义之前被调用（相互递归、先调用后声明）。

对每个找到的函数/fiber，检查器注册：
- `FunctionSignature { params: Vec<Type>, return_type: Option<Type>, is_fiber: bool }` 到 `self.functions`
- 符号表中的条目，类型为 `Type::Unknown`（函数）或 `Type::Fiber(...)`（fiber）

内置转换函数 `i`、`f`、`s`、`b` 预注册，`Type::Unknown` 参数及对应返回类型。

### 上下文标志

| 字段 | 用途 |
|---|---|
| `loop_depth: usize` | 跟踪 `while`/`for` 嵌套深度。为零时 `break`/`continue` 为错误。进入 fiber 体时重置为 0。 |
| `fiber_context: Option<Option<Type>>` | `None` = 不在 fiber 中；`Some(None)` = void fiber；`Some(Some(T))` = 产出 `T` 的类型化 fiber。 |
| `fiber_has_yield: bool` | 遇到 `yield` 时设置。嵌套 fiber 定义间保存/恢复。 |
| `is_table_lambda: bool` | 在 `.where()` 谓词内设置；允许裸列名作为标识符，通过在 `__row_tmp` 中查找。 |

### 类型推断规则

- 表达式类型从字面量自底向上推断，并通过运算符传播。
- `Type::Unknown` 作为通配符——涉及 `Unknown` 的任何操作均不报错。
- `Type::Json` 在赋值与比较中与任何类型兼容。
- 数值提升：`Int op Float → Float`。
- 空数组字面量 `[]` 在可用时从赋值上下文继承类型。
- `Type::Table([])`（空列列表）与任何 `Table(cols)` 兼容——列信息在推断后回写到变量声明类型。

### `is_compatible(expected, actual) -> bool`

主要兼容规则：

| 规则 | 说明 |
|---|---|
| 任一为 `Unknown` | 始终兼容 |
| 任一为 `Json` | 始终兼容 |
| `Int` ↔ `Float` | 相互兼容（数值提升） |
| `Int` ↔ `Date` | 兼容（时间戳为整数） |
| `Set(X)` ↔ `Array(inner)` | 元素类型匹配时兼容 |
| `Set(N)` ↔ `Set(Z)` | 兼容（均为整数类型集合） |
| `Set(S)` ↔ `Set(C)` | 兼容（均为字符串类型集合） |
| `Table([])` ↔ `Table(cols)` | 任一列列表为空时兼容 |
| `Table(a)` ↔ `Table(b)` | 长度匹配且所有列类型两两兼容时兼容 |
| `Map(k1,v1)` ↔ `Map(k2,v2)` | 递归检查 |
| `Fiber(None)` ↔ `Fiber(None)` | 均为 void fiber |
| `Fiber(Some(T))` ↔ `Fiber(Some(U))` | 当且仅当 T 与 U 兼容时兼容 |

### 函数调用检查

对于命名函数调用（`ExprKind::FunctionCall`）：
1. 先在 `self.functions` 中查找（已声明的函数/fiber）。
2. 若未找到，在符号表中查找（一等函数值、动态调用）。
3. 若解析的签名为 fiber，调用返回 `Type::Fiber(Some(ret))`（fiber 实例化，非直接调用）。
4. 超出声明参数数量的额外参数会检查但不拒绝（可变参数容忍）。
5. 未解析的调用推送 `UndefinedVariable`。

对于语句级函数调用（`StmtKind::FunctionCallStmt`）：
- 参数数量不匹配 → `Other("Function expects N arguments, got M")`。
- 符号表中未知名称 → `UndefinedVariable`。

### Fiber 声明检查（`FiberDecl`）

1. 在 `self.functions`（优先）或符号表中查找 `fiber_name`。
2. 验证解析的签名具有 `is_fiber: true`。
3. 对照参数检查实参类型。
4. 用 `Type::Fiber(inner_type)` 定义新变量。

### Table 上 `.where()` 的检查

检查 `table.where(predicate)` 时：
1. 打开临时作用域并定义 `__row_tmp: Table(cols)`。
2. 设置 `is_table_lambda = true`。
3. `collect_pred_idents()` 收集谓词中使用的所有 `Identifier` 名称。
4. 若任何标识符同时存在于外围作用域**且**匹配列名 → `S301 WherePredicateNameCollision`。
5. 对谓词进行类型检查；必须返回 `Bool`。
6. 退出作用域并恢复 `is_table_lambda`。

### `Table.join()` 检查

检查器合并列定义：左表列加上右表中尚未存在的列（按 `StringId` 相等）。若列名冲突，保留右表的列。结果类型为 `Type::Table(combined_cols)`。

### `for` 循环检查

`ForIterType` 的 `iter_type` 字段在检查期间根据 `start` 的推断类型变更：
- `Type::Array(_)` → 设置 `ForIterType::Array`，循环变量获得内部类型
- `Type::Set(st)` → 设置 `ForIterType::Set`，循环变量获得集合元素类型
- `Type::Table(cols)` → 视为 `ForIterType::Array`，循环变量获得 `Type::Table(cols)`
- `Type::Fiber(inner)` → 设置 `ForIterType::Fiber`，循环变量获得内部 yield 类型
- `Type::Int`（带 `to` 关键字）→ 保持 `ForIterType::Range`，循环变量为 `Int`

---

## 已验证错误码

| 代码 | 条件 |
|---|---|
| `UndefinedVariable(name)` | 声明前使用名称 |
| `RedefinedVariable(name)` | 同一作用域内重复声明 |
| `ConstReassignment(name)` | 对 `const` 变量赋值 |
| `TypeMismatch { expected, actual }` | 表达式类型与期望不符 |
| `InvalidBinaryOp { op, left, right }` | 运算符与不兼容类型一起使用 |
| `BreakOutsideLoop` | `break` 在 `while`/`for` 外 |
| `ContinueOutsideLoop` | `continue` 在 `while`/`for` 外 |
| `[S208] YieldOutsideFiber` | `yield` 在任何 fiber 体外使用 |
| `[S209] FiberTypeMismatch` | void fiber 内 `yield expr;`（应为 `yield;`） |
| `[S210] ReturnTypeMismatchInFiber` | 类型化 fiber 中裸 `return;`（缺少返回值） |
| `[S301] WherePredicateNameCollision` | 局部变量名与 `.where()` 中表列名冲突 |
| `Other(msg)` | 各种上下文错误（参数数量、未知方法、void fiber 迭代等） |

---

## 错误报告

`TypeError` 值携带 `Span { line, col, len }`。检查后，`main.rs` 将每条错误传递给 `Reporter::error()`，输出：
1. `ERROR: <message>`
2. 带行号的相关源代码行
3. 从 `col` 开始、长度为 `len` 的 `~~~` 下划线

若存在任何 `TypeError`，错误报告后立即停止编译——不生成字节码。
