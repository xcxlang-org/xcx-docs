//==compiler/parser.md==\\
# XCX 语法分析器 — v3.0

XCX 语法分析器将 token 流转换为高层抽象语法树（AST）。

## 架构：Pratt Parsing

XCX 使用 **Pratt Parser**（自顶向下运算符优先级解析）。

- **文件**：`src/parser/pratt.rs`
- **前瞻**：一个 token（`current` + `peek`），通过 `advance()` 手动推进。
- **错误恢复**：语法错误时，`synchronize()` 跳过 token 直到下一个分号或已知语句起始关键字（`func`、`fiber`、`if`、`for`、`const`、`return`、`>!` 等）。

`Parser` 结构体在生命周期 `'a` 内借用源字符串，`Scanner<'a>` 使用相同生命周期参数，反映基于字节切片的扫描器设计。

### 优先级级别（从低到高）

| 级别 | 运算符 |
|---|---|
| `Lowest` | — |
| `Lambda` | `->` |
| `Assignment` | `=` |
| `LogicalOr` | `OR`, `\|\|` |
| `LogicalAnd` | `AND`, `&&` |
| `Equals` | `==`, `!=` |
| `LessGreater` | `>`, `<`, `>=`, `<=`, `HAS` |
| `Sum` | `+`, `-`, `++` |
| `SetOp` | `UNION`, `INTERSECTION`, `DIFFERENCE`, `SYMMETRIC_DIFFERENCE` |
| `Product` | `*`, `/`, `%` |
| `Power` | `^` |
| `Prefix` | `-x` |
| `Call` | `.`, `[` |

## 语句分派

`parse_statement_internal()` 根据当前 token 分派：

- **类型关键字**（`i`、`f`、`s`、`b`、`array`、`set`、`map`、`date`、`table`、`json`）→ `parse_var_decl()`，若后跟 `=` 则 → `parse_assignment()`
- **`const`** → `parse_var_decl()`，`is_const = true`
- **`var`**（标识符）→ 类型推断变量声明
- **`>!`** → `parse_print_stmt()`
- **`>?`** → `parse_input_stmt()`
- **`halt`** → `parse_halt_stmt()`
- **`if`** → `parse_if_statement()`
- **`while`** → `parse_while_statement()`
- **`for`** → `parse_for_statement()`
- **`break`** / **`continue`** → `parse_break_statement()` / `parse_continue_statement()`
- **`func`** → `parse_func_def()`
- **`fiber`** → `parse_fiber_statement()`（根据 peek 分派到 def 或 decl）
- **`return`** → `parse_return_stmt()`
- **`yield`** → `parse_yield_stmt()`（处理 `yield expr`、`yield from expr` 与 `yield;`）
- **`@wait`** → `parse_wait_stmt()`
- **`serve`** → `parse_serve_stmt()`
- **`net`** → `parse_net_stmt()`
- **`include`** → `parse_include_stmt()`
- **Identifier + `=`** → `parse_assignment()`
- **Identifier + `(`** → `parse_func_call_stmt()`
- **其他** → `parse_expr_stmt()`

## 函数定义风格

XCX 支持两种语法不同的函数定义风格：

**大括号风格**（类 C）：
```xcx
func name(i: x, s: y -> i) {
    return x + 1;
}
```

**XCX 风格**（关键字块）：
```xcx
func:i: name(i: x, s: y) do;
    return x + 1;
end;
```

两者产生相同的 `StmtKind::FunctionDef` AST 节点。大括号风格的返回类型在参数列表内或 `)` 之后用 `-> type` 声明。

## Fiber 语句

`parse_fiber_statement()` 查看 `peek` 以决定：
- `peek == Colon` → `parse_fiber_decl()`（实例化：`fiber:T: varname = fiberDef(args);`）
- 否则 → `parse_fiber_def()`（定义：`fiber name(params) { body }`）

`parse_fiber_decl()` 还处理解析类型与名称后当前 token 为 `(` 的情况——此时转向 `finish_fiber_def()`（带 leading `fiber:` 类型注解的定义）。

## 主要解析结构

- **变量声明**：`i: name = expr;`、`const s: NAME = expr;`、`var name = expr;`
- **控制流**：`if (cond) then; ... elseif (cond) then; ... else; ... end;`
- **While 循环**：`while (cond) do; ... end;`
- **For 循环**：`for x in expr do; ... end;` 与 `for x in start to end @step n do; ... end;`
- **函数**：`func`（两种风格，见上文）
- **Fiber**：`fiber name(params) { body }` 与 `fiber:T: varname = fiberName(args);`
- **Yield**：`yield expr;`、`yield from expr;`、`yield;`
- **HTTP**：`serve: name { port=..., routes=... };`、`net.get(url)`、`net.request { ... } as resp;`、`net.respond(status, body);`
- **集合**：Array `[a, b, c]`、Set `set:N { 1,,10 }`、Map `[k :: v, ...]`、Table `table { columns=[...] rows=[...] }`
- **Raw blocks**：`<<<...>>>` 用于内联 JSON/字符串
- **Include**：`include "path";` 或 `include "path" as alias;`
- **I/O**：`>! expr;`（print）、`>? varname;`（input）
- **Halt**：`halt.alert >! msg;`、`halt.error >! msg;`、`halt.fatal >! msg;`
- **Wait**：`@wait(ms);` 或 `@wait ms;`
- **Date 字面量**：`date("2024-01-01")` 或 `date("01/01/2024", "DD/MM/YYYY")`

## 表达式解析

`parse_expression(precedence)` 对左侧调用 `parse_prefix()`，然后在 peek token 优先级超过当前最小值时循环调用 `parse_infix(left)`。

主要前缀解析器：
- **Identifiers**：若后跟 `(`，解析为 `FunctionCall`；否则为 `Identifier`。
- **Literals**：`IntLiteral`、`FloatLiteral`、`StringLiteral`、`True`、`False`
- **一元减号**：解析为 `Binary { left: IntLiteral(0), op: Minus, right }`（无独立 `Unary::Neg`）
- **`not` / `!`**：`Unary { op: Not/Bang, right }`
- **`(`...`)` 分组**：单个表达式 → 解包；多个逗号分隔 → `Tuple`
- **`[`...`]`**：若首元素后跟 `::`，解析为 `MapLiteral`；否则 `ArrayLiteral`
- **`{`...`}`**：解析为 `ArrayOrSetLiteral`（类型在语义或编译时解析）
- **`set:N { }` 等**：显式 `SetLiteral`，已知 `SetType`
- **`map { schema=[...] data=[...] }`**：显式 `MapLiteral`
- **`table { columns=[...] rows=[...] }`**：`TableLiteral`
- **`random.choice from expr`**：`RandomChoice`
- **`date(...)`**：`DateLiteral`
- **`net.get/post/put/delete/patch(...)` 等**：`NetCall` 或 `NetRespond`
- **`<<<...>>>`**：`RawBlock`
- **`.terminal!cmd`**：`TerminalCommand`

主要中缀解析器：
- **`.`**：`parse_dot_infix`——若后跟 `(` 产生 `MethodCall`，否则 `MemberAccess`；也处理 `.[key]` 索引访问
- **`[`**：`parse_index_infix` → `Index`
- **`->`**：`parse_lambda_infix` → `Lambda`
- **所有二元运算符**：`Binary { left, op, right }`

## `parse_expr_stmt()` 后处理

解析完整表达式语句后，`parse_expr_stmt()` 检查结果是否为 `MethodCall`：
- 方法名 `bind` 且 2 个参数、第二参数为 `Identifier` → 重写为 `StmtKind::JsonBind`
- 方法名 `inject` 且 2 个参数 → 重写为 `StmtKind::JsonInject`

这允许在语句级别使用语法糖 `json.bind("path", target);` 与 `json.inject(mapping, table);`。

## Expander（`src/parser/expander.rs`）

Expander 在解析**之后**、语义分析**之前**运行。它是独立的树重写遍。

### 职责

**Include 解析**：`include "file.xcx";` 被替换为该文件的内联 AST。循环依赖通过 `visiting_files: HashSet<PathBuf>` 检测。文件通过 `included_files: HashSet<PathBuf>` 去重（每个文件只 include 一次，除非有别名）。

**别名前缀**：`include "math.xcx" as math;` 使该文件所有顶层名称重命名为 `math.name`。调用点（`math.sin(x)`）由 `expand_expr_inplace` 从 `MethodCall` 重写为 `FunctionCall { name: "math.sin" }`。`prefix_program()` / `prefix_stmt_impl()` / `prefix_expr_impl()` 函数遍历整个子 AST，重命名所有顶层符号引用。

**Fiber 名称前缀**：`FiberDecl::fiber_name` 引用也会加前缀，以便重命名后 fiber 实例化正确解析。

**YieldFrom 前缀**：遍历 `StmtKind::YieldFrom` 表达式，使 `yield from` 内的 fiber 构造调用也被重命名。

**受保护名称**（永不加前缀）：`json`、`date`、`store`、`halt`、`terminal`、`net`、`env`、`crypto`、`EMPTY`、`math`、`random`、`i`、`f`、`s`、`b`、`from`、`main`。

**Include 路径搜索顺序**：
1. 相对于当前文件目录
2. 在 `lib/` 目录中（相对于 CWD，然后从可执行文件路径向上遍历）

## AST 定义（`src/parser/ast.rs`）

### `Expr` — 表达式节点

| 变体 | 说明 |
|---|---|
| `IntLiteral(i64)` | 整数常量 |
| `FloatLiteral(f64)` | 浮点常量 |
| `StringLiteral(StringId)` | 内联字符串 |
| `BoolLiteral(bool)` | `true` / `false` |
| `Identifier(StringId)` | 变量或函数名 |
| `Binary { left, op, right }` | 二元运算 |
| `Unary { op, right }` | 一元运算（`not`、`!`） |
| `FunctionCall { name, args }` | 按内联名称的函数调用 |
| `MethodCall { receiver, method, args }` | 值上的点调用 |
| `MemberAccess { receiver, member }` | 无调用的点访问 |
| `Index { receiver, index }` | 方括号索引 `a[i]` |
| `Lambda { params, return_type, body }` | 箭头 lambda `x -> expr` |
| `ArrayLiteral { elements }` | 显式 `[a, b, c]` |
| `ArrayOrSetLiteral { elements }` | 歧义 `{a, b, c}` — 稍后解析 |
| `SetLiteral { set_type, elements, range }` | 带可选范围的类型化集合 |
| `MapLiteral { key_type, value_type, elements }` | Map 字面量 |
| `TableLiteral { columns, rows }` | Table 字面量 |
| `DateLiteral { date_string, format }` | `date("2024-01-01")` |
| `Tuple(Vec<Expr>)` | 括号逗号分隔列表 |
| `NetCall { method, url, body }` | HTTP 调用表达式 |
| `NetRespond { status, body, headers }` | HTTP respond 表达式 |
| `RawBlock(StringId)` | `<<<...>>>` 原始内容 |
| `TerminalCommand(cmd, arg)` | `.terminal !cmd` |
| `RandomChoice { set }` | `random.choice from set` |

### `Stmt` — 语句节点

主要变体：`VarDecl`、`Assign`、`Print`、`Input`、`If`、`While`、`For`、`Break`、`Continue`、`FunctionDef`、`FiberDef`、`FiberDecl`、`Return`、`Yield`、`YieldFrom`、`YieldVoid`、`Include`、`Serve`、`NetRequestStmt`、`JsonBind`、`JsonInject`、`Halt`、`Wait`、`ExprStmt`、`FunctionCallStmt`。

### `Type` — 类型系统

`Int`、`Float`、`String`、`Bool`、`Date`、`Json`、`Array(Box<Type>)`、`Set(SetType)`、`Map(Box<Type>, Box<Type>)`、`Table(Vec<ColumnDef>)`、`Fiber(Option<Box<Type>>)`、`Builtin(StringId)`、`Unknown`。

`SetType` 变体：`N`（Natural）、`Z`（Integer）、`Q`（Rational/Float）、`S`（String）、`C`（Char/String）、`B`（Boolean）。

### `ForIterType`

`Range`（数值 `start to end`）、`Array`、`Set`、`Fiber`——由类型检查器设置，编译器用于发出正确的循环模式。

### `ColumnDef`

```rust
pub struct ColumnDef {
    pub name:    StringId,
    pub ty:      Type,
    pub is_auto: bool,    // @auto columns are auto-incremented on insert
}
```

## String Interner

所有字符串值（标识符、字符串字面量、方法名）通过 `Interner` 内联为 `StringId (u32)`。内联器在解析器中创建并传递给后续所有阶段。这意味着检查器、编译器与 VM 均使用数字 ID 进行名称比较，而非 `String` 比较。
