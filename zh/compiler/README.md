# XCX 编译器 — 文档 v3.1

> 使用 Rust 实现的 XCX 语言编译器。多阶段流水线：词法分析器 → 语法分析器 → 语义分析 → 字节码编译器 → 虚拟机 → JIT（Cranelift）。

---

## 目录

1. [架构概览](#架构概览)
2. [快速开始](#快速开始)
3. [项目结构](#项目结构)
4. [编译流水线](#编译流水线)
5. [模块](#模块)
   - [词法分析器（Scanner）](#词法分析器scanner)
   - [语法分析器（Pratt）](#语法分析器pratt)
   - [Expander](#expander)
   - [语义分析（Sema）](#语义分析sema)
   - [编译器（Backend）](#编译器backend)
   - [虚拟机（VM）](#虚拟机vm)
   - [JIT（Cranelift）](#jitcranelift)
6. [关键设计决策](#关键设计决策)
7. [诊断系统](#诊断系统)
8. [安全](#安全)

---

## 架构概览

```
Source Code (.xcx)
        │
        ▼
  1. Lexer         src/lexer/scanner.rs       → token stream
        │
        ▼
  2. Parser        src/parser/pratt.rs        → raw AST (Program)
        │
        ▼
  3. Expander      src/parser/expander.rs     → expanded AST (include, aliases)
        │
        ▼
  4. Sema          src/sema/checker.rs        → validated, annotated AST
        │
        ▼
  5. Compiler    src/backend/mod.rs         → FunctionChunk + constants + functions
        │
        ▼
  6. VM            src/backend/vm.rs          → bytecode execution (register-based)
        │  hot loops → trace recording
        ▼
  7. JIT           src/backend/jit.rs         → native machine code (Cranelift)
```

---

## 快速开始

```bash
# Start REPL
xcx

# Run file
xcx program.xcx

# Version
xcx --version

# Help
xcx --help
```

**REPL 内：**

```
xcx> !help     # display help
xcx> !clear    # clear screen
xcx> !exit     # exit
```

---

## 项目结构

```
src/
├── lexer/
│   ├── scanner.rs        # Byte scanner (&[u8])
│   └── token.rs          # TokenKind and Span
├── parser/
│   ├── pratt.rs          # Pratt Parser (tokens → AST)
│   ├── expander.rs       # Resolving include and prefixing aliases
│   └── ast.rs            # AST node definitions (Expr, Stmt, Type, Program)
├── sema/
│   ├── checker.rs        # Type checking and variable resolution
│   ├── symbol_table.rs   # Hierarchical symbol table
│   └── interner.rs       # String interner (str → StringId)
├── backend/
│   ├── mod.rs            # Bytecode compiler (AST → OpCode)
│   ├── vm.rs             # Register-based VM with NaN-boxing + JIT hooks
│   ├── jit.rs            # Cranelift trace compiler
│   └── repl.rs           # Interactive REPL
└── diagnostic/
    └── report.rs         # Error reporter with code highlighting
```

---

## 编译流水线

### 1. Lexer
将源字节转换为 token。在 `&[u8]` 上工作——无 `Vec<char>` 分配。

### 2. Parser
Pratt Parser（自顶向下运算符优先级）构建 AST。单 token 前瞻。通过 `synchronize()` 进行错误恢复。

### 3. Expander
在解析**之后**、语义分析**之前**运行。解析 `include` 指令并为别名名称加前缀。

### 4. Semantic Analysis
检查类型、检测未定义变量、验证 fiber/循环上下文。在生成字节码前收集所有错误。

### 5. Compiler
两遍编译。第一遍：注册全局变量/函数。第二遍：发出带 Span 注解的字节码。

### 6. VM
基于寄存器的虚拟机。每帧有按 `u8` 槽号索引的扁平 `Vec<Value>`。8 字节值（NaN-boxing）。

### 7. JIT
通过 Cranelift 自动将热循环编译为原生代码。对开发者透明。

---

## 模块

详细说明见独立文件：

- [`docs/lexer.md`](docs/lexer.md) — 词法分析器 / Scanner
- [`docs/parser.md`](docs/parser.md) — Pratt Parser 与 AST
- [`docs/expander.md`](docs/expander.md) — Expander
- [`docs/sema.md`](docs/sema.md) — 语义分析
- [`docs/backend.md`](docs/backend.md) — 编译器与 VM
- [`docs/jit.md`](docs/jit.md) — Cranelift JIT
- [`docs/language.md`](docs/language.md) — XCX 语言参考

---

## 关键设计决策

### NaN-Boxing of values
每个值为单个 `u64`（8 字节）。IEEE 754 quiet NaN 的高位用作类型标签。低 48 位承载 payload。标量无堆分配，enum 无标签开销，基本类型无指针间接访问。

### Register-based VM
不使用操作数栈——每帧使用扁平 `Vec<Value>` 数组。OpCode 直接引用源/目标寄存器。编译器中简单的 bump 指针寄存器分配。

### Tracing JIT (Cranelift)
回跳（循环边）按 IP 计数。达到阈值（50 次迭代）后开始 trace 录制。完成的 trace 由 Cranelift 编译为原生代码。类型 Guard（`GuardInt`、`GuardFloat`）处理类型特化；guard 失败触发回退到解释器。

### String Interning
所有标识符与字符串字面量由 `Interner` 内联为 `StringId (u32)`。消除整个流水线中重复的字符串分配与堆比较。

### Two-pass Compilation
第一遍（`register_globals_recursive`）在发出任何字节码之前为所有全局变量与函数分配索引——支持相互递归与先调用后声明。

---

## 诊断系统

`src/diagnostic/report.rs` 中的 `Reporter` 生成带上下文的错误消息：

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

每条错误包含：
- **级别**：ERROR 或 HALT
- **位置**：行号与列号
- **可视化高亮**：相关源代码行及 `~~~` 下划线

语义错误（`TypeError`）在检查阶段收集到 `Vec` 中，在字节码生成前一次性报告。

---

## 安全

### 网络中的 SSRF 防护
被阻止的目标：
- `file://` URL
- `169.254.x.x`（链路本地 / AWS 元数据端点）
- 私有范围：`10.x`、`192.168.x`、`172.16–31.x`（不含 localhost）

同时应用于 `HttpCall` 与 `HttpRequest`。

### HTTP 内容大小限制
在 `HttpServe` 中，`into_string()` 后检查响应内容。若超过 **10 MB**，返回 413 JSON 错误而非实际内容。

### CORS 头
所有 `HttpServe` 响应自动包含：
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

预检 `OPTIONS` 请求返回 `204` 响应，不调用处理程序。

### 安全文件路径
`store.*` 操作验证路径——阻止 `..`、绝对路径与 Windows 驱动器字母。
