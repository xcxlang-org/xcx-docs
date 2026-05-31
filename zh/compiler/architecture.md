# XCX 编译器架构 — v3.1

XCX 编译器使用 Rust 实现，采用多阶段流水线架构。

## 编译流水线

```
Source Code
    │
    ▼
1. Lexer (Scanner)        — src/lexer/scanner.rs
    │  Produces: Token stream
    ▼
2. Parser (Pratt)         — src/parser/pratt.rs
    │  Produces: Raw AST (Program)
    ▼
3. Expander               — src/parser/expander.rs
    │  Produces: Expanded AST (include directives resolved, aliases prefixed)
    ▼
4. Type Checker (Sema)    — src/sema/checker.rs
    │  Produces: Validated, annotated AST
    ▼
5. Compiler (Backend)     — src/backend/mod.rs
    │  Produces: FunctionChunk (main) + Arc<Vec<Value>> constants + Arc<Vec<FunctionChunk>> functions
    ▼
6. VM                     — src/backend/vm.rs
    │  Executes register-based bytecode
    │  Hot loops detected → Trace recording begins
    ▼
7. JIT (Cranelift)        — src/backend/jit.rs
       Compiles recorded traces to native machine code
```

> **说明**：Expander 属于 `src/parser/` 模块，但作为独立的解析后阶段运行，位于语义分析之前。JIT 是可选的加速层，对热循环自动激活——若 trace 尚未编译完成，字节码执行会不间断继续。

## 项目结构

```
src/
├── lexer/
│   ├── scanner.rs      # Byte-level scanner (&[u8])
│   └── token.rs        # TokenKind and Span definitions
├── parser/
│   ├── pratt.rs        # Pratt parser (token stream → AST)
│   ├── expander.rs     # Include resolution and alias prefixing
│   └── ast.rs          # AST node definitions (Expr, Stmt, Type, Program)
├── sema/
│   ├── checker.rs      # Type checker and variable resolver
│   ├── symbol_table.rs # Hierarchical scope/symbol table with parent-pointer chain
│   └── interner.rs     # String interner (str → StringId)
├── backend/
│   ├── mod.rs          # Bytecode compiler (AST → OpCode) with register allocator
│   ├── vm.rs           # Register-based VM with NaN-boxed Values + tracing JIT hooks
│   ├── jit.rs          # Cranelift-based trace compiler
│   └── repl.rs         # Interactive REPL
└── diagnostic/
    └── report.rs       # Error reporter with source highlighting
```

## 诊断系统

编译器使用 `Reporter` 结构体生成带上下文的错误消息。每条错误包含：
- **级别**：ERROR 或 HALT 变体
- **位置**：行号与列号
- **可视化高亮**：相关源代码行及 `~~~` 下划线

语义错误（`TypeError`）在检查阶段收集到 `Vec` 中，并在字节码生成开始前一次性报告。若存在任何错误，编译在此阶段停止。

VM 产生的运行时错误包含源位置信息（行号与列号），这些信息来自与每个 `FunctionChunk` 字节码一同存储的 `spans` 表。

## 关键设计决策

### NaN-Boxed Values
所有值均通过 NaN-boxing 表示为单个 `u64`。IEEE 754 quiet NaN 的高位用作类型标签，低 48 位承载 payload（整数、布尔值或指针）。浮点数原样存储。这意味着每个 `Value` 恰好 8 字节——标量无需堆分配，无 enum 标签开销，基本类型无指针间接访问。完整标签布局见 `src/backend/vm.rs`。

### Register-Based VM
VM 采用**基于寄存器**的设计，而非基于栈。每个函数帧拥有按槽号索引的扁平 `Vec<Value>`。OpCode 直接引用源寄存器与目标寄存器（`dst`、`src1`、`src2` 字段）。编译器的 `FunctionCompiler` 为临时寄存器维护简单的 bump 分配器（`push_reg()` / `pop_reg()`）。局部变量与临时值位于同一扁平数组中——没有独立的操作数栈。

### Tracing JIT (Cranelift)
热回跳通过每帧的 `hot_counts: Vec<usize>` 计数器检测。当循环回边达到阈值（50 次迭代）时，开始 trace 录制阶段。`Executor` 录制 `TraceOp` 变体——解释器操作的有类型、特化版本——直到 trace 闭合（循环回到起始 IP）。完成的 `Trace` 通过 Cranelift 编译为原生代码并缓存在 `trace_cache` 中。该循环的后续迭代完全绕过解释器。trace 中的 Guard（`GuardInt`、`GuardFloat`、`GuardTrue`、`GuardFalse`）处理类型特化；guard 失败会导致 side-exit，在正确的 IP 处回到解释器。

### String Interning
所有标识符与字符串字面量通过 `Interner` 内联为 `StringId`（u32）。这避免了整个流水线中重复的字符串分配与堆比较。

### Constant Deduplication
编译器维护 `string_constants: HashMap<String, usize>`，确保每个唯一字符串值在常量表中只存储一次。这对编译期间频繁发出的内置方法名尤其有效。

### Method Dispatch via Enum
内置方法调用编译为 `OpCode::MethodCall { kind: MethodKind, base, arg_count }`，其中 `MethodKind` 是涵盖约 50 个内置方法的 `Copy` enum。未知或动态方法名（例如 JSON 字段访问）使用独立的 `OpCode::MethodCallCustom { method_name_idx, base, arg_count }` 路径。这消除了 VM 分派循环中的字符串比较。

### Two-Pass Compilation
后端在发出任何字节码之前，先执行第一遍（`register_globals_recursive`）为所有全局变量、函数与 fiber 分配索引。

### Span-Annotated Bytecode
每个发出的 OpCode 都与源 AST 中的 `Span` 配对。`FunctionChunk` 在 `bytecode: Arc<Vec<OpCode>>` 旁存储 `spans: Arc<Vec<Span>>`。VM 利用此信息生成精确到行的运行时错误消息。

### Fiber-as-Coroutine
Fiber 不是 OS 线程。每个 `FiberState` 存储自己的 `ip` 与 `locals`，由 VM 协作式恢复。yield 时 locals 移回 `FiberState`；resume 时再移出。除初始 `Vec` 创建外，无额外分配。

### Thread-Safe Value Representation
所有共享可变集合使用 `Arc<RwLock<T>>`（通过 `parking_lot`）。`FunctionChunk` 字节码与 spans 包装在 `Arc<Vec<...>>` 中，以便在 HTTP 工作线程间共享而无需复制。`Value` 为 `Copy` + `Send` + `Sync`。

### Graceful HTTP Shutdown
全局 `AtomicBool`（`SHUTDOWN`）由 Ctrl+C 信号处理程序（通过 `ctrlc` crate）设置。所有 HTTP 工作线程与主分派循环轮询此标志，在进程终止前干净退出。

### Scope Chain Symbol Table
`SymbolTable` 使用父指针链接链而非深拷贝。进入函数作用域时创建引用父表的新 `SymbolTable`——查找沿链向上遍历。函数作用域进入为 O(1) 而非 O(n)。

### Byte-Level Scanner
词法分析器在 `&[u8]`（原始源字节的引用）上操作，不分配 `Vec<char>`。Unicode 处理仅在必要时进行。注释检测使用 `slice.starts_with(b"---")`。
