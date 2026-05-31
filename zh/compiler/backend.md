# XCX 后端 — 编译器与 VM

> **文件：** `src/backend/mod.rs`、`src/backend/vm.rs`

---

## 目录

1. [字节码编译器](#字节码编译器)
2. [值表示 — NaN-Boxing](#值表示--nan-boxing)
3. [指令集（OpCodes）](#指令集opcodes)
4. [VM 架构](#vm-架构)
5. [纤程执行模型](#纤程执行模型)
6. [HTTP 服务器](#http-服务器)
7. [内存管理](#内存管理)
8. [循环优化](#循环优化)

---

## 字节码编译器

**文件：** `src/backend/mod.rs`

### 寄存器分配

`FunctionCompiler` 通过 `next_local: usize` 跟踪下一个可用寄存器：

```rust
pub fn push_reg(&mut self) -> u8 {
    let r = self.next_local as u8;
    self.next_local += 1;
    if self.next_local > self.max_locals_used {
        self.max_locals_used = self.next_local;
    }
    r
}

pub fn pop_reg(&mut self) {
    self.next_local -= 1;
}
```

局部变量（具名）通过 `define_local(id, slot)` 分配到槽位，并存储在 `scopes: Vec<HashMap<StringId, usize>>` 中。临时变量使用 `push_reg()`/`pop_reg()` — 当表达式结果被消费后会复用。

`max_locals_used` 会被记录，以便 `FunctionChunk::max_locals` 在函数调用时能预分配大小恰好合适的 locals 向量。

### 常量去重

`CompileContext::add_constant` 使用 `string_constants: HashMap<String, usize>` 对字符串常量去重。重复的字符串常量（例如作为方法名参数多次出现的 `"insert"`）会复用同一常量表槽位。

### 两遍编译

**第一遍** — `register_globals_recursive`：
- 为每个全局变量和 fiber-decl 实例分配槽位索引
- 为每个函数/纤程分配函数索引
- 在 `functions: Vec<FunctionChunk>` 中预分配空的 `FunctionChunk` 槽位

**第二遍** — `compile_stmt` / `compile_expr`：
- 通过 `emit(op, span)` 发射与 span 配对的字节码
- `main` 中的顶层指令使用 `GetVar`/`SetVar`（全局变量）；嵌套指令使用寄存器

### FunctionChunk

```rust
pub struct FunctionChunk {
    pub bytecode:   Arc<Vec<OpCode>>,
    pub spans:      Arc<Vec<Span>>,    // spans[i] corresponds to bytecode[i]
    pub is_fiber:   bool,
    pub max_locals: usize,
}
```

字节码和 spans 被包装在 `Arc` 中，以便在 HTTP 工作线程之间共享而无需复制。

---

## 值表示 — NaN-Boxing

每个值都是单个 `Value(u64)` — 一个 64 位字。XCX 使用 **NaN-boxing**：将 IEEE 754 quiet NaN 的位模式重新用作类型标签前缀。

```
Bit layout: [63..52: exponent/QNAN] [51..48: type tag] [47..0: payload]

Float : stored directly as f64 bits — DOES NOT have the QNAN_BASE prefix set
Int   : QNAN_BASE | TAG_INT  | (i48 value & 0x0000_FFFF_FFFF_FFFF)
Bool  : QNAN_BASE | TAG_BOOL | (0 or 1)
Date  : QNAN_BASE | TAG_DATE | (i48 ms timestamp)
Ptr   : QNAN_BASE | TAG_XXX  | (pointer & 0x0000_FFFF_FFFF_FFFF)
```

### 标签常量

| 常量 | 值 | 类型 |
|---|---|---|
| `QNAN_BASE` | `0x7FF0_0000_0000_0000` | 基础 NaN 标记 |
| `TAG_INT` | `0x0001_0000_0000_0000` | 48 位有符号整数 |
| `TAG_BOOL` | `0x0002_0000_0000_0000` | 布尔值（载荷 0/1） |
| `TAG_DATE` | `0x0003_0000_0000_0000` | 48 位时间戳（毫秒） |
| `TAG_STR` | `0x0004_0000_0000_0000` | `Arc<Vec<u8>>` 指针 |
| `TAG_ARR` | `0x0005_0000_0000_0000` | `Arc<RwLock<Vec<Value>>>` 指针 |
| `TAG_SET` | `0x0006_0000_0000_0000` | `Arc<RwLock<SetData>>` 指针 |
| `TAG_MAP` | `0x0007_0000_0000_0000` | `Arc<RwLock<Vec<(Value,Value)>>>` 指针 |
| `TAG_TBL` | `0x0008_0000_0000_0000` | `Arc<RwLock<TableData>>` 指针 |
| `TAG_FUNC` | `0x0009_0000_0000_0000` | 函数索引（u32） |
| `TAG_ROW` | `0x000A_0000_0000_0000` | `Arc<RowRef>` 指针 |
| `TAG_JSON` | `0x000B_0000_0000_0000` | `Arc<RwLock<serde_json::Value>>` 指针 |
| `TAG_FIB` | `0x000C_0000_0000_0000` | `Arc<RwLock<FiberState>>` 指针 |
| `TAG_DB` | `0x000D_0000_0000_0000` | `Arc<DatabaseData>` 指针 |

指针载荷仅使用低 48 位 — 在所有用户空间指针可容纳于 48 位的 x86-64 和 AArch64 平台上均有效。

### 引用计数

带指针标签的值携带 `Arc` 引用计数。VM 在每次赋值、返回和集合修改时通过 `inc_ref()` / `dec_ref()` 手动管理它们 — 确保堆分配对象在不再被引用时释放，而无需垃圾收集器。

---

## 指令集（OpCodes）

所有操作码均为基于寄存器的：它们引用具名的 `u8` 寄存器槽位，而非操作数栈。

### 寄存器 / 变量移动

| OpCode | 说明 |
|---|---|
| `LoadConst { dst, idx }` | 将 `constants[idx]` 加载到寄存器 `dst` |
| `Move { dst, src }` | 将寄存器 `src` 复制到 `dst` |
| `GetVar { dst, idx }` | 将 `globals[idx]` 加载到 `dst`（读锁 globals） |
| `SetVar { idx, src }` | 将 `src` 存储到 `globals[idx]`（写锁 globals） |

### 算术运算

所有算术运算均为三寄存器：`dst = src1 OP src2`。运行时类型分派选择整数、浮点、字符串拼接、日期算术或集合运算路径。

`Add`、`Sub`、`Mul`、`Div`、`Mod`、`Pow`、`IntConcat`（`++`）

### 比较（结果为 Bool）

`Equal`、`NotEqual`、`Greater`、`Less`、`GreaterEqual`、`LessEqual`

### 逻辑运算

`And { dst, src1, src2 }`、`Or { dst, src1, src2 }`、`Not { dst, src }`、`Has { dst, src1, src2 }`

### 控制流

| OpCode | 说明 |
|---|---|
| `Jump { target }` | 无条件跳转；向后跳转时递增 `hot_counts[target]` |
| `JumpIfFalse { src, target }` | 若 `src` 为 `Bool(false)` 则跳转 |
| `JumpIfTrue { src, target }` | 若 `src` 为 `Bool(true)` 则跳转 |
| `Call { dst, func_idx, base, arg_count }` | 调用函数；结果 → `dst` |
| `Return { src }` | 从当前帧返回 `src` 中的值 |
| `ReturnVoid` | 无返回值返回 |
| `Halt` | 停止执行 |

### 集合

| OpCode | 说明 |
|---|---|
| `ArrayInit { dst, base, count }` | 从 `base` 起收集 `count` 个寄存器 → 在 `dst` 中创建新数组 |
| `SetInit { dst, base, count }` | 收集 `count` 个寄存器 → 在 `dst` 中创建新集合 |
| `SetRange { dst, start, end, step, has_step }` | 从寄存器值构建范围集合 |
| `MapInit { dst, base, count }` | 收集 `count` 个键值对 → 在 `dst` 中创建新映射 |
| `TableInit { dst, skeleton_idx, base, row_count }` | 从列模式常量 + 行值构建表 |

### 集合运算

`SetUnion`、`SetIntersection`、`SetDifference`、`SetSymDifference` — 均为三寄存器，两个操作数必须为 `TAG_SET`。

### 方法分派

| OpCode | 说明 |
|---|---|
| `MethodCall { dst, kind, base, arg_count }` | 通过 `MethodKind` 枚举分派内置方法 — 无运行时字符串查找 |
| `MethodCallCustom { dst, method_name_idx, base, arg_count }` | 通过常量表中的字符串分派动态方法（JSON 字段、别名） |
| `MethodCallNamed { dst, kind, base, arg_count, names_idx }` | 带命名参数的方法调用 |

`base` 指向接收者寄存器；参数位于 `locals[base+1..base+1+arg_count]`。`MethodKind` 是一个 `#[derive(Copy)]` 枚举，涵盖约 50 个内置方法（`Push`、`Pop`、`Get`、`Insert`、`Update`、`Delete`、`Where`、`Join`、`Sort`、`Format`、`Next`、`IsDone`、`Close` 等）。

### 纤程操作

| OpCode | 说明 |
|---|---|
| `FiberCreate { dst, func_idx, base, arg_count }` | 分配 `FiberState`，从参数预填充 locals，将 `Fiber` 存储到 `dst` |
| `Yield { src }` | 挂起纤程，将 `src` 中的值返回给调用者 |
| `YieldVoid` | 挂起无返回值纤程 |

### I/O 与系统

| OpCode | 说明 |
|---|---|
| `Print { src }` | 将 `locals[src]` 打印到 stdout |
| `Input { dst, ty }` | 从 stdin 读取一行 → 经类型转换后存入 `dst` |
| `Wait { src }` | 休眠 `src` 毫秒 |
| `HaltAlert { src }` | 打印警告消息，继续执行 |
| `HaltError { src }` | 打印错误 + span 信息，停止帧，递增 error_count |
| `HaltFatal { src }` | 打印致命错误 + span 信息，停止帧，递增 error_count |
| `TerminalExit` | `std::process::exit(0)` |
| `TerminalClear` | 通过 ANSI 转义序列清屏 |
| `TerminalRaw / TerminalNormal` | 启用/禁用原始终端模式 |
| `TerminalCursor { on }` | 显示/隐藏光标 |
| `TerminalMove { x_src, y_src }` | 移动光标 |
| `TerminalWrite { src }` | 写入（无换行） |
| `InputKey { dst }` | 读取按键（非阻塞） |
| `InputKeyWait { dst }` | 读取按键（阻塞） |
| `InputReady { dst }` | 检查是否有可用输入 |
| `EnvGet { dst, src }` | 读取环境变量 |
| `EnvArgs { dst }` | 获取 CLI 参数 |

### HTTP

| OpCode | 说明 |
|---|---|
| `HttpCall { dst, method_idx, url_src, body_src }` | 通过 `ureq` 进行简单 HTTP 调用，JSON 结果 → `dst` |
| `HttpRequest { dst, arg_src }` | 从配置映射进行完整 HTTP 调用（method、url、headers、body、timeout）→ `dst` |
| `HttpRespond { status_src, body_src, headers_src }` | 在纤程处理程序内发送 HTTP 响应 |
| `HttpServe { func_idx, port_src, host_src, workers_src, routes_src }` | 启动 `tiny_http` 服务器，生成工作线程 |

### 存储

`StoreWrite`、`StoreRead`、`StoreAppend`、`StoreExists`、`StoreDelete`、`StoreList`、`StoreIsDir`、`StoreSize`、`StoreMkdir`、`StoreGlob`、`StoreZip`、`StoreUnzip`

### JSON

`JsonParse { dst, src }`、`JsonBind { idx, json_src, path_src }`、`JsonBindLocal { dst, json_src, path_src }`、`JsonInject { table_idx, json_src, mapping_src }`、`JsonInjectLocal { table_reg, json_src, mapping_src }`

### 类型转换

`CastInt { dst, src }`、`CastFloat { dst, src }`、`CastString { dst, src }`、`CastBool { dst, src }`

### 加密与日期

`CryptoHash`、`CryptoVerify`、`CryptoToken`、`DateNow`

### 数据库

`DatabaseInit { dst, engine_src, path_src, tables_base_reg, table_count }`

---

## VM 架构

**文件：** `src/backend/vm.rs`

### VM 状态

```rust
pub struct VM {
    pub globals:     Arc<RwLock<Vec<Value>>>,
    pub error_count: AtomicUsize,
    pub traces:      Arc<RwLock<HashMap<usize, Arc<Trace>>>>,
    pub jit:         Mutex<JIT>,
}
```

`VM` 被包装在 `Arc<VM>` 中，并在 HTTP 工作线程之间共享。每个工作线程创建自己的 `Executor`，拥有私有的 locals。

### Executor 状态

```rust
struct Executor {
    vm:               Arc<VM>,
    ctx:              SharedContext,
    current_spans:    Option<Arc<Vec<Span>>>,
    fiber_yielded:    bool,
    hot_counts:       Vec<usize>,         // IP backward jump counter
    recording_trace:  Option<Trace>,      // trace being recorded
    is_recording:     bool,
    trace_cache:      Vec<Option<Arc<Trace>>>, // compiled traces indexed by start IP
    http_req:         Option<Arc<Mutex<Option<tiny_http::Request>>>>,
    http_req_val:     Option<Value>,
    terminal_raw_enabled: bool,
}
```

### SharedContext

```rust
pub struct SharedContext {
    pub constants: Arc<Vec<Value>>,
    pub functions: Arc<Vec<FunctionChunk>>,
}
```

`SharedContext` 可廉价克隆（两次 `Arc` 计数递增），并独立传递给每个工作线程。无需深拷贝。

### 执行流程

```
VM::run(main_chunk, ctx)                        [Arc<VM>]
  └─ Executor::run_frame_owned(main_chunk)
       └─ execute_bytecode_inner(bytecode, &mut ip, &mut locals, ...)
            │
            ├─ [JIT Fast Path] if trace_cache[ip].is_some():
            │     execute_trace(trace, ip, locals, globals)
            │     → returns next IP or None
            │
            └─ [Interpreter Path] fetch opcode, execute
                 ├─ Continue      → proceed normally
                 ├─ Jump(t)       → ip = t; increment hot_counts if backward
                 ├─ Return(val)   → exit frame, return val
                 ├─ Yield(val)    → suspend (fiber), return val to caller
                 └─ Halt          → stop, increment error_count
```

---

## 纤程执行模型

纤程是**协作式协程**，而非 OS 线程。

```rust
pub struct FiberState {
    pub func_id:       usize,
    pub ip:            usize,
    pub locals:        Vec<Value>,    // moved during resume, returned after
    pub is_done:       bool,
    pub yielded_value: Option<Value>, // cache for IsDone + Next pattern
}
```

### 恢复序列（`resume_fiber`）

1. 通过 `std::mem::take` 从 `FiberState` **移动**读取 `func_id`、`ip` 和 `locals` — 无克隆
2. 使用移动后的 locals 从 `fiber.ip` 开始 `execute_bytecode`
3. 遇到 `Yield`：设置 `fiber_yielded = true`。将 locals 移回 `FiberState`。更新 `fiber.ip`。返回 yield 的值。
4. 遇到 `Return` / 字节码结束：设置 `fiber.is_done = true`。返回最终值。

恢复/挂起不会触发除初始 `Vec` 创建之外的任何堆分配 — 仅涉及移动。

### IsDone / Next 模式

`IsDone` 检查 `FiberState::is_done`，同时考虑 `yielded_value` 是否已缓存。`Next` 若存在则取缓存的 `yielded_value`，否则调用 `resume_fiber`。这确保 `for x in fiber` 循环永远不会双重推进纤程。

### 遍历纤程的 For 循环（`ForIterType::Fiber`）

编译器发射：
1. `MethodCall(IsDone)` → `JumpIfTrue` 退出
2. `MethodCall(Next)` → 赋值给循环变量
3. 循环体
4. `Jump` 回到步骤 1
5. 遇到 `break`：`MethodCall(Close, base = fiber_reg)` 在跳转前将纤程标记为完成

---

## HTTP 服务器

`HttpServe` 启动 `tiny_http` 服务器并生成 N 个 OS 线程：

```rust
for _ in 0..workers {
    let server  = server.clone();    // Arc<tiny_http::Server>
    let vm      = vm_arc.clone();    // Arc<VM>
    let ctx     = self.ctx.clone();  // SharedContext (two Arc clones)
    let routes  = routes.clone();    // Arc<Vec<(String, usize)>>
    std::thread::spawn(move || { /* recv → match route → run handler fiber */ });
}
```

每个工作线程运行自己的 `Executor`，拥有独立的 locals。全局变量通过 `Arc<RwLock<Vec<Value>>>` 共享。

### 请求处理

对于每个传入请求，工作线程：
1. 将 `"METHOD /path"` 键与路由表匹配（不区分大小写）
2. 构建 JSON 对象 `{ method, url, body, ip, headers }` 作为 `Value::Json`
3. 将 `tiny_http::Request` 存储在 `Arc<Mutex<Option<tiny_http::Request>>>` 中，并传递给新的 `Executor`
4. 在该工作线程的 `Executor` 中同步运行匹配的处理纤程
5. 当处理程序调用 `net.respond(...)` 时，VM 执行 `HttpRespond` 发送响应
6. 若处理程序退出时未调用 `net.respond`，工作线程发送 `500` 响应

### 优雅关闭

`SHUTDOWN` 是 `vm.rs` 中的 `pub static AtomicBool`。`main.rs` 中的 Ctrl+C 处理程序将其设为 `true`。工作线程在每次 `recv_timeout(100ms)` 时检查它。主线程也在 `sleep(500ms)` 循环中阻塞并轮询 `SHUTDOWN`。一旦设置，所有循环退出，进程干净终止。

---

## 内存管理

- **无垃圾收集器**。通过 `Arc` 进行引用计数（对 NaN-boxed 指针值手动 `inc_ref`/`dec_ref`）。
- **标量值**（Int、Float、Bool、Date、函数索引）：完全存储在 `u64` 内 — 零堆分配。
- **集合值**：`Arc<RwLock<T>>` 确保共享所有权。克隆集合 `Value` 仅递增 `Arc` 计数器。
- **变更**：`.insert()`、`.update()`、`.delete()` 获取写锁。同一集合的所有句柄都能看到变更。
- **只读方法**：`.size()`、`.get()`、`.contains()` 获取读锁 — 允许多个并发读者。

---

## 循环优化

这些操作码由编译器发射，将常见循环计数器模式合并为单条指令，减少分派开销并改善 JIT 追踪：

| OpCode | 说明 |
|---|---|
| `IncLocal { reg }` | 将寄存器 `reg` 中的整数加 1 |
| `IncVar { idx }` | 将 `idx` 处的全局变量加 1 |
| `LoopNext { reg, limit_reg, target }` | 递增 `reg`，若 `reg <= limit_reg` 则跳转到 `target` |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target }` | 递增 `inc_reg`（独立计数器，如数组索引），递增 `reg`（循环变量），条件跳转 |
| `IncVarLoopNext { g_idx, reg, limit_reg, target }` | 类似 `IncLocalLoopNext`，但 `g_idx` 为全局计数器 |
| `ArrayLoopNext { idx_reg, size_reg, target }` | 合并的数组索引迭代 |

### 优化变换

编译器在循环步骤结束前检查最后发射的指令。若为 `IncVar` 或 `IncLocal`，则分别替换为 `IncVarLoopNext` 或 `IncLocalLoopNext`，将递增与条件循环测试合并：

```rust
match self.bytecode[len - 1] {
    OpCode::IncVar { idx } => {
        self.bytecode.pop();
        self.emit(OpCode::IncVarLoopNext { g_idx: idx, reg: loop_var_reg, ... });
    }
    OpCode::IncLocal { reg } => {
        self.bytecode.pop();
        self.emit(OpCode::IncLocalLoopNext { inc_reg: reg, ... });
    }
}
```
