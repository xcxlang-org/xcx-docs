# XCX 语义分析（Sema）— 文档

> **文件：** `src/sema/checker.rs`、`src/sema/symbol_table.rs`、`src/sema/interner.rs`

---

## 目录

1. [概述](#概述)
2. [String Interner](#string-interner)
3. [符号表](#符号表)
4. [类型检查器](#类型检查器)
5. [类型兼容规则](#类型兼容规则)
6. [错误码](#错误码)
7. [错误报告](#错误报告)

---

## 概述

Sema 阶段在字节码生成前验证 AST 的逻辑正确性与类型一致性。它由三个组件组成：

```
Interner → StringId (u32)
     ↓
SymbolTable → hierarchical type scopes
     ↓
Checker → collection of TypeErrors
```

仅当结果 `Vec<TypeError>` 为空时才编译程序。

---

## String Interner

**文件：** `src/sema/interner.rs`

`Interner` 映射 `&str → StringId(u32)`。它是编译器中所有字符串标识的唯一来源。在词法/解析阶段创建，通过引用传递给检查器与编译器。

```rust
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

**实现：**

```rust
pub struct Interner {
    map:     HashMap<String, StringId>,
    strings: Vec<String>,
}
```

每个唯一字符串在 `strings: Vec<String>` 中只存储一次。流水线其余部分使用数字 ID，消除类型检查与编译期间的堆比较。

---

## 符号表

**文件：** `src/sema/symbol_table.rs`

`SymbolTable` 使用**父指针链**管理嵌套作用域中的变量绑定。

### 结构

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // ref to surrounding scope
    scopes: Vec<HashMap<String, Type>>,   // stack of scope frames (local)
    consts: Vec<HashSet<String>>,         // which names are const, per frame
}
```

### 创建子作用域

进入函数或 fiber 体时，创建引用外围表的新 `SymbolTable`，而非克隆：

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

这是 O(1)——仅为新帧分配一个空 `HashMap`。查找按需遍历父链。

### 作用域生命周期

| 方法 | 说明 |
|---|---|
| `enter_scope()` | 推入新帧 — 用于 `if`、`while`、`for` 体 |
| `exit_scope()` | 退出当前作用域 |
| `new_with_parent(parent)` | 创建子表 — 用于函数/fiber 体 |
| `define(name, ty, is_const)` | 始终写入**最内层**作用域 |
| `lookup(name)` | 从内向外遍历，然后到父表 |
| `has_in_current_scope(name)` | 仅检查最内层帧 — 用于重定义检测 |
| `is_const(name)` | 检查拥有作用域是否将名称标记为 const |

### 不支持变量遮蔽

XCX **不支持**变量遮蔽。在**当前作用域**中定义已存在的变量返回 `RedefinedVariable`。父作用域中的变量可访问，但不能在子作用域中用相同名称重新声明。

---

## 类型检查器

**文件：** `src/sema/checker.rs`

`Checker` 结构体遍历 AST 并累积 `TypeError` 值。

### Checker 状态

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

### 上下文标志

| 字段 | 用途 |
|---|---|
| `loop_depth: usize` | 跟踪 `while`/`for` 嵌套深度。为零时 `break`/`continue` 为错误。进入 fiber 体时重置为 0。 |
| `fiber_context: Option<Option<Type>>` | `None` = 不在 fiber 中；`Some(None)` = void fiber；`Some(Some(T))` = 产出 `T` 的类型化 fiber |
| `fiber_has_yield: bool` | 遇到 `yield` 时设置。嵌套 fiber 定义间保存/恢复。 |
| `is_table_lambda: bool` | 在 `.where()` 谓词内设置；通过 `__row_tmp` 允许裸列名作为标识符 |
| `in_yield_expr: bool` | 跟踪是否在 yield 表达式内 |
| `last_expr_was_db_io: bool` | 数据库 I/O 操作标志 |

### 预扫描遍

在检查任何语句体之前，检查器对**当前语句列表**（非递归）中所有 `FunctionDef` 与 `FiberDef` 节点执行**前向声明扫描**。这允许函数与 fiber 在源文件中定义之前被调用（相互递归、先调用后声明）。

对每个找到的函数/fiber，检查器注册：
- `FunctionSignature { params: Vec<Type>, return_type: Option<Type>, is_fiber: bool }` 到 `self.functions`
- 符号表中的条目，类型为 `Type::Unknown`（函数）或 `Type::Fiber(...)`（fiber）

内置转换函数 `i`、`f`、`s`、`b` 预注册。

### 类型推断规则

- 表达式类型从字面量自底向上推断，并通过运算符传播。
- `Type::Unknown` 作为通配符——涉及 `Unknown` 的任何操作均不报错。
- `Type::Json` 在赋值与比较中与任何类型兼容。
- 数值提升：`Int op Float → Float`。
- 空数组字面量 `[]` 在可用时从赋值上下文继承类型。
- `Type::Table([])`（空列列表）与任何 `Table(cols)` 兼容。

---

## 类型兼容规则

`is_compatible(expected, actual) -> bool` 函数：

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
| `Table(a)` ↔ `Table(b)` | 长度匹配且列类型两两兼容时兼容 |
| `Map(k1,v1)` ↔ `Map(k2,v2)` | 递归检查 |
| `Fiber(None)` ↔ `Fiber(None)` | 均为 void fiber |
| `Fiber(Some(T))` ↔ `Fiber(Some(U))` | 当 T 与 U 兼容时兼容 |

---

## 错误码

| 代码 | 条件 |
|---|---|
| `[S101] UndefinedVariable(name)` | 声明前使用名称 |
| `[S102] RedefinedVariable(name)` | 同一作用域内重复声明 |
| `[S103] TypeMismatch { expected, actual }` | 表达式类型与期望类型不符 |
| `[S104] InvalidBinaryOp { op, left, right }` | 运算符与不兼容类型一起使用 |
| `[S105] ConstReassignment(name)` | 对 `const` 变量赋值 |
| `[S106] BreakOutsideLoop` | `break` 在 `while`/`for` 外 |
| `[S107] ContinueOutsideLoop` | `continue` 在 `while`/`for` 外 |
| `[S108] IndexAccessNotSupported(type)` | 对不支持的类型进行索引 |
| `[S109] PropertyNotFound` | 类型上不存在该属性 |
| `[S110] MethodNotFound` | 类型上不存在该方法 |
| `[S111] InvalidArgumentCount` | 参数数量不正确 |
| `[S208] YieldOutsideFiber` | `yield` 在 fiber 体外使用 |
| `[S209] FiberTypeMismatch` | void fiber 内 `yield expr;`（应为 `yield;`） |
| `[S210] ReturnTypeMismatchInFiber` | 类型化 fiber 中裸 `return;`（缺少返回值） |
| `[S211] CannotIterateOverVoidFiber` | 迭代 void fiber |
| `[S212] CannotRunTypedFiber` | 对类型化 fiber 调用 `.run()` |
| `[S301] WherePredicateNameCollision` | 局部变量名与 `.where()` 中列名冲突 |
| `[S302] TableRowCountMismatch` | 表行列数与 schema 不符 |
| `[D401] Rule violation` | 无 `.where()` 的 `remove()` |
| `Other(msg)` | 其他上下文错误（参数数量、未知方法等） |

---

## 错误报告

`TypeError` 值携带 `Span { line, col, len }`。检查后，`main.rs` 将每条错误传递给 `Reporter::error()`，输出：

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

若存在任何 `TypeError`，报告错误后立即停止编译——不生成字节码。

---

## 函数调用检查

对于命名函数调用（`ExprKind::FunctionCall`）：
1. 先在 `self.functions` 中查找（已声明的函数/fiber）
2. 若未找到，在符号表中查找（一等函数值、动态调用）
3. 若解析的签名为 fiber，调用返回 `Type::Fiber(Some(ret))`（fiber 实例化，非直接调用）
4. 超出声明参数数量的额外参数会检查但不拒绝（可变参数容忍）
5. 未解析的调用添加 `UndefinedVariable` 错误

---

## Table 上 `.where()` 的检查

检查 `table.where(predicate)` 时：
1. 打开临时作用域并定义 `__row_tmp: Table(cols)`
2. 设置 `is_table_lambda = true`
3. `collect_pred_idents()` 收集谓词中使用的所有 `Identifier` 名称
4. 若任何标识符同时存在于外围作用域**且**匹配列名 → `S301 WherePredicateNameCollision`
5. 检查谓词；必须返回 `Bool`
6. 关闭作用域并恢复 `is_table_lambda`
