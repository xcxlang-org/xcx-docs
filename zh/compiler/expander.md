# XCX Expander — 文档

> **文件：** `src/parser/expander.rs`  
> 在解析**之后**、语义分析**之前**运行。

---

## 目录

1. [概述](#概述)
2. [职责](#职责)
3. [解析 include](#解析-include)
4. [别名前缀](#别名前缀)
5. [受保护名称](#受保护名称)
6. [Include 路径搜索顺序](#include-路径搜索顺序)

---

## 概述

Expander 是在解析阶段之后、语义分析之前运行的独立 AST 重写遍。它处理 `include` 与 `include ... as alias` 指令。

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

## 职责

### 1. 解析 include
`include "file.xcx";` 被替换为该文件的内联 AST。

- 循环依赖通过 `visiting_files: HashSet<PathBuf>` 检测
- 文件通过 `included_files: HashSet<PathBuf>` 去重（每个文件只 include 一次，除非有别名）

### 2. 别名前缀
`include "math.xcx" as math;` 使该文件所有顶层名称获得前缀 `math.name`。调用点（`math.sin(x)`）通过 `expand_expr_inplace` 从 `MethodCall` 重写为 `FunctionCall { name: "math.sin" }`。

函数 `prefix_program()` / `prefix_stmt_impl()` / `prefix_expr_impl()` 遍历整个子 AST，重命名所有顶层符号引用。

### 3. Fiber 名称前缀
`FiberDecl::fiber_name` 引用也会加前缀，以便重命名后 fiber 实例化正确解析。

### 4. YieldFrom 前缀
遍历 `StmtKind::YieldFrom` 表达式，使 `yield from` 内的 fiber 构造调用也被重命名。

---

## 解析 include

```xcx
include "utils.xcx";           --- Simple include (deduplicated)
include "math.xcx" as math;    --- Aliased include
```

别名 include 之后：
- `math.xcx` 的所有顶层符号获得 `math.` 前缀
- 对 `math.sin(x)` 的调用从 `MethodCall` 重写为 `FunctionCall { name: "math.sin" }`

示例：

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

## 别名前缀

`prefix_stmt_impl` / `prefix_expr_impl` 算法：

1. 从程序收集所有顶层名称（`top_level_names: HashSet<StringId>`）
2. 对每条语句/表达式：若标识符属于 `top_level_names`，替换为 `prefix.name`
3. 特别处理：
   - `FiberDecl::fiber_name` — 以便重命名后实例化正常工作
   - `StmtKind::YieldFrom` — `yield from` 中的 fiber 表达式
   - 别名对象上的 `MethodCall` → 重写为 `FunctionCall`

---

## 受保护名称

以下名称**永不**加前缀（受保护的内置项）：

```
json    date    store   halt    terminal
net     env     crypto  EMPTY   math
random  i       f       s       b
from    main
```

---

## Include 路径搜索顺序

1. 相对于当前文件目录
2. 在 `lib/` 目录中（相对于 CWD，然后从可执行文件路径向上遍历）

可通过以下方式添加额外路径：

```rust
expander.add_include_path(path);
```

在 `main.rs` 中，相对于 CWD 的 `lib/` 目录会自动添加：

```rust
if let Ok(cwd) = std::env::current_dir() {
    let lib_path = cwd.join("lib");
    if lib_path.exists() {
        expander.add_include_path(lib_path);
    }
}
```

---

## 错误检测

| 错误 | 说明 |
|---|---|
| `Circular dependency` | 检测到 include 循环（`visiting_files`） |
| `File not found` | `include` 文件在任何搜索路径中均不存在 |
| `Could not read file` | 读取文件时的 I/O 错误 |
