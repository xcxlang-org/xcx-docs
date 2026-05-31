# XCX 虚拟机（VM）— v3.1

XCX VM 是一个自定义的**基于寄存器**的运行时，用于执行 XCX 字节码，并由基于 Cranelift 的追踪 JIT 编译器增强。

## 架构

- **文件**：`src/backend/vm.rs`
- **执行模型**：在扁平寄存器文件上运行的取指-译码-执行循环（`execute_bytecode`）
- **寄存器文件**：每个帧拥有的 `Vec<Value>`，由 `u8` 槽位号索引
- **全局变量**：单个扁平 `Vec<Value>`，位于 `Arc<RwLock<Vec<Value>>>` 之后，在所有工作线程间共享
- **JIT**：`src/backend/jit.rs` — 基于 Cranelift 的热点追踪本机代码编译器

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
    hot_counts:       Vec<usize>,         // per-IP backward-jump counter
    recording_trace:  Option<Trace>,      // trace being recorded
    is_recording:     bool,
    trace_cache:      Vec<Option<Arc<Trace>>>, // compiled traces indexed by start IP
    http_req:         Option<Arc<Mutex<Option<tiny_http::Request>>>>,
    http_req_val:     Option<Value>,
}
```

### SharedContext

```rust
pub struct SharedContext {
    pub constants: Arc<Vec<Value>>,
    pub functions: Arc<Vec<FunctionChunk>>,
}
```

`SharedContext` 可廉价克隆（两次 `Arc` 指针递增），并独立传递给每个工作线程。不发生深拷贝。

---

## 值表示：NaN-Boxing

每个值都是单个 `Value(u64)` — 一个 64 位字。XCX 使用 **NaN-boxing**：将 IEEE 754 quiet NaN 的位模式重新用作类型标签前缀。

```
Bit layout: [63..52: exponent/QNAN] [51..48: type tag] [47..0: payload]

Float : stored directly as f64 bits — does NOT have QNAN_BASE prefix set
Int   : QNAN_BASE | TAG_INT  | (i48 value & 0x0000_FFFF_FFFF_FFFF)
Bool  : QNAN_BASE | TAG_BOOL | (0 or 1)
Date  : QNAN_BASE | TAG_DATE | (i48 timestamp ms)
Ptr   : QNAN_BASE | TAG_XXX  | (pointer & 0x0000_FFFF_FFFF_FFFF)
```

### 标签常量

| 常量 | 值 | 类型 |
|---|---|---|
| `QNAN_BASE` | `0x7FF0_0000_0000_0000` | 基础 NaN 标记 |
| `TAG_INT`   | `0x0001_0000_0000_0000` | 48 位有符号整数 |
| `TAG_BOOL`  | `0x0002_0000_0000_0000` | 布尔值（载荷 0/1） |
| `TAG_DATE`  | `0x0003_0000_0000_0000` | 48 位时间戳（毫秒） |
| `TAG_STR`   | `0x0004_0000_0000_0000` | `Arc<String>` 指针 |
| `TAG_ARR`   | `0x0005_0000_0000_0000` | `Arc<RwLock<Vec<Value>>>` 指针 |
| `TAG_SET`   | `0x0006_0000_0000_0000` | `Arc<RwLock<SetData>>` 指针 |
| `TAG_MAP`   | `0x0007_0000_0000_0000` | `Arc<RwLock<Vec<(Value,Value)>>>` 指针 |
| `TAG_TBL`   | `0x0008_0000_0000_0000` | `Arc<RwLock<TableData>>` 指针 |
| `TAG_FUNC`  | `0x0009_0000_0000_0000` | 函数索引（u32） |
| `TAG_ROW`   | `0x000A_0000_0000_0000` | `Arc<RowRef>` 指针 |
| `TAG_JSON`  | `0x000B_0000_0000_0000` | `Arc<RwLock<serde_json::Value>>` 指针 |
| `TAG_FIB`   | `0x000C_0000_0000_0000` | `Arc<RwLock<FiberState>>` 指针 |

指针载荷仅使用低 48 位 — 在所有用户空间指针可容纳于 48 位的 x86-64 和 AArch64 平台上均有效。

### 指针值的引用计数

带指针标签的值携带 `Arc` 引用计数。VM 在每次赋值、返回和集合修改时通过 `inc_ref()` / `dec_ref()` 手动管理它们 — 确保堆分配对象（字符串、数组、JSON、纤程等）在不再被引用时释放，而无需垃圾收集器。

---

## 指令集（OpCodes）

所有操作码均为基于寄存器的：它们引用具名的 `u8` 寄存器槽位，而非操作数栈。

### 寄存器 / 变量移动

| OpCode | 说明 |
|---|---|
| `LoadConst { dst, idx }` | 将 `constants[idx]` 加载到寄存器 `dst` |
| `Move { dst, src }` | 将寄存器 `src` 复制到 `dst` |
| `GetVar { dst, idx }` | 将 `globals[idx]` 加载到 `dst`（读锁 globals） |
| `SetVar { idx, src }` | 将 `src` 写入 `globals[idx]`（写锁 globals） |

### 算术运算

所有算术操作均为三寄存器：`dst = src1 OP src2`。运行时类型分派选择整数、浮点、字符串拼接、日期算术或集合运算路径。

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
| `Call { dst, func_idx, base, arg_count }` | 调用函数；参数为 `locals[base..base+arg_count]`；结果 → `dst` |
| `Return { src }` | 从当前帧返回 `src` 中的值 |
| `ReturnVoid` | 无返回值返回 |
| `Halt` | 停止执行（显式或不可恢复错误） |

### 集合

| OpCode | 说明 |
|---|---|
| `ArrayInit { dst, base, count }` | 从 `base` 起收集 `count` 个寄存器 → 在 `dst` 中创建新 Array |
| `SetInit { dst, base, count }` | 收集 `count` 个寄存器 → 在 `dst` 中创建新 Set |
| `SetRange { dst, start, end, step, has_step }` | 从寄存器值构建范围 Set |
| `MapInit { dst, base, count }` | 收集 `count` 个键值对（从 `base` 起交替排列的寄存器）→ 新 Map |
| `TableInit { dst, skeleton_idx, base, row_count }` | 从列模式常量 + 行值构建 Table |

### 集合运算

`SetUnion`、`SetIntersection`、`SetDifference`、`SetSymDifference` — 均为三寄存器，两个操作数必须为 `TAG_SET`。

### 方法分派

| OpCode | 说明 |
|---|---|
| `MethodCall { dst, kind, base, arg_count }` | 通过 `MethodKind` 枚举分派内置方法 — 运行时无字符串查找 |
| `MethodCallCustom { dst, method_name_idx, base, arg_count }` | 通过常量表中的字符串分派动态方法（JSON 字段、别名） |

`base` 指向接收者寄存器；参数为 `locals[base+1..base+1+arg_count]`。`MethodKind` 是一个 `#[derive(Copy)]` 枚举，涵盖约 50 个内置方法（`Push`、`Pop`、`Get`、`Insert`、`Update`、`Delete`、`Where`、`Join`、`Sort`、`Format`、`Next`、`IsDone`、`Close` 等）。编译器在编译时通过 `map_method_kind()` 将方法名解析为 `MethodKind` 变体。

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
| `Input { dst }` | 从 stdin 读取一行 → `dst` |
| `Wait { src }` | 休眠 `src` 毫秒 |
| `HaltAlert { src }` | 打印 alert 消息，继续执行 |
| `HaltError { src }` | 打印错误 + span 信息，停止帧，递增错误计数 |
| `HaltFatal { src }` | 打印 fatal 消息 + span 信息，停止帧，递增错误计数 |
| `TerminalExit` | `std::process::exit(0)` |
| `TerminalClear` | 通过 ANSI 转义或 OS 命令清屏 |
| `TerminalRun { dst, cmd_src }` | 执行外部命令，结果 → `dst` |
| `EnvGet { dst, src }` | 读取 `src` 命名的环境变量 → `dst` |
| `EnvArgs { dst }` | 将 CLI 参数的 `Array<String>` 推入 → `dst` |

### HTTP

| OpCode | 说明 |
|---|---|
| `HttpCall { dst, method_idx, url_src, body_src }` | 通过 `ureq` 进行简单 HTTP 调用（GET/POST 等），结果 JSON → `dst` |
| `HttpRequest { dst, arg_src }` | 从配置映射进行完整 HTTP 调用（method、url、headers、body、timeout）→ `dst` |
| `HttpRespond { status_src, body_src, headers_src }` | 在处理纤程内发送 HTTP 响应；触发 `Yield` 将控制权交还 |
| `HttpServe { func_idx, port_src, host_src, workers_src, routes_src }` | 启动 `tiny_http` 服务器，生成工作线程，阻塞主线程直到 `SHUTDOWN` |

### 存储

`StoreWrite { base }`、`StoreRead { dst, base }`、`StoreAppend { base }`、`StoreExists { dst, base }`、`StoreDelete { base }`

### JSON

`JsonParse { dst, src }`、`JsonBind { idx, json_src, path_src }`、`JsonBindLocal { dst, json_src, path_src }`、`JsonInject { table_idx, json_src, mapping_src }`、`JsonInjectLocal { table_reg, json_src, mapping_src }`

### 类型转换

`CastInt { dst, src }`、`CastFloat { dst, src }`、`CastString { dst, src }`、`CastBool { dst, src }`

### 加密与日期

`CryptoHash { dst, pass_src, alg_src }`、`CryptoVerify { dst, pass_src, hash_src, alg_src }`、`CryptoToken { dst, len_src }`、`DateNow { dst }`

### 循环优化

这些操作码由编译器发射，将常见循环计数器模式融合为单条指令，减少分派开销并改善 JIT 可追踪性。

| OpCode | 说明 |
|---|---|
| `IncLocal { reg }` | 将寄存器 `reg` 中的整数加 1 |
| `IncVar { idx }` | 将 `idx` 处的全局变量加 1 |
| `LoopNext { reg, limit_reg, target }` | 递增 `reg`，若 `reg <= limit_reg` 则跳转到 `target`，否则继续 |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target }` | 递增 `inc_reg`（独立计数器，如数组索引），递增 `reg`（循环变量），条件跳转 |
| `IncVarLoopNext { g_idx, reg, limit_reg, target }` | 类似 `IncLocalLoopNext`，但 `g_idx` 为全局计数器 |

---

## 追踪 JIT 编译器

### 概述

XCX 2.2 包含**追踪 JIT**，使用 [Cranelift](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift) 代码生成框架自动将热点循环编译为本机机器码。

JIT 对程序员完全透明 — 它自动激活，并在类型守卫或不支持的操作时回退到解释器。

### 追踪检测

在每次 `Jump { target }` 且 `target < current_ip`（向后跳转，即循环回边）时：

1. `hot_counts[target]` 递增。
2. 当 `hot_counts[target] >= 50` 时，以 `start_ip = target` 启动新的 `Trace`。
3. 追踪录制受门控：仅当 `trace_cache[target].is_none()`（尚无已编译追踪）且 `is_recording` 为 false 时才开始。

### 追踪录制

当 `is_recording` 为 true 时，解释器正常执行每条操作码，**并**为当前运行时类型录制专门的 `TraceOp`。例如：

- 两个整数寄存器上的 `Add` 录制 `GuardInt { reg: src1 }`、`GuardInt { reg: src2 }`、`AddInt { dst, src1, src2 }` — 而非通用 `Add`。
- **未**执行的 `JumpIfFalse` 录制 `GuardTrue { reg: src, fail_ip: target }` — 断言分支始终为 false。
- **已**执行的 `JumpIfFalse` 录制 `GuardFalse { reg: src, fail_ip: next_ip }`。

若某操作码无法追踪（复杂方法调用、字符串操作等），则中止录制并将 `is_recording` 设回 false。

### 追踪编译

当追踪循环执行回到 `start_ip` 时，完整的 `Trace` 被交给 Cranelift `JIT::compile()` 函数（`src/backend/jit.rs`）。Cranelift 将 `TraceOp` 序列编译为具有以下签名的本机函数：

```rust
unsafe extern "C" fn(
    locals_ptr: *mut Value,
    globals_ptr: *mut Value,
    consts_ptr:  *const Value,
) -> i32
```

返回值为恢复执行的下一条 IP（0 = 正常继续，正数 = 侧向退出 IP，负数 = halt）。

编译后的函数指针存储在 `Trace::native_ptr`（`AtomicPtr<u8>`）中，`Arc<Trace>` 被插入 `vm.traces`（全局共享）和 `trace_cache`（每个 Executor 的快速路径）。

### 追踪执行

在分派循环的每次迭代中，获取下一条操作码之前：

```
if trace_cache[current_ip].is_some() {
    execute_trace(trace, ip, locals, &mut glbs)
    continue
}
```

若有可用的已编译本机函数（`native_ptr != null`），则通过 `transmute` 直接调用 — 完全绕过解释器执行整个循环体。若 JIT 尚未编译，则使用解释型 `TraceOp` 路径作为中间步骤。

### TraceOp 变体

| 变体 | 说明 |
|---|---|
| `LoadConst`, `Move` | 带常量值的寄存器移动 |
| `AddInt/SubInt/MulInt/DivInt/ModInt` | 整数算术，含除/模零的 fail-IP |
| `AddFloat/SubFloat/MulFloat/DivFloat/ModFloat` | 浮点算术 |
| `CmpInt / CmpFloat` | 使用 `cc: u8` 条件码的比较 |
| `GuardInt / GuardFloat` | 类型守卫 — 寄存器类型错误时退出追踪 |
| `GuardTrue / GuardFalse` | 分支守卫 — 分支方向意外时退出追踪 |
| `CastIntToFloat` | 将 int 寄存器扩展为 float |
| `IncLocal / IncVar` | 单寄存器/全局递增 |
| `LoopNextInt` | 范围循环的合并递增 + 条件跳转 |
| `IncVarLoopNext / IncLocalLoopNext` | 数组和 for-range 循环的融合变体 |
| `GetVar / SetVar` | 全局变量访问 |
| `And / Or / Not` | 布尔逻辑 |
| `Jump` | 无条件跳转（在追踪中触发循环回边检测） |

---

## 执行流程

```
VM::run(main_chunk, ctx)                        [Arc<VM>]
  └─ Executor::run_frame_owned(main_chunk)
       └─ execute_bytecode(bytecode, &mut ip, &mut locals)
            │
            ├─ [JIT fast path] if trace_cache[ip].is_some():
            │     execute_trace(trace, ip, locals, globals)
            │     → returns next IP or None
            │
            └─ [Interpreter path] fetch opcode, execute
                 ├─ Continue      → advance ip normally
                 ├─ Jump(t)       → ip = t; increment hot_counts if backward
                 ├─ Return(val)   → exit frame, return val
                 ├─ Yield(val)    → suspend (fiber), return val to caller
                 └─ Halt          → stop, increment error_count
```

函数通过 `run_frame(func_id, params)` 调用，创建预分配为 `chunk.max_locals` 大小的新 `locals` 向量。`current_spans` 在每次 `run_frame` 调用时切换到被调用函数的 span 表，返回时恢复。

---

## 纤程执行模型

纤程是**协作式协程**，不是线程。

```rust
pub struct FiberState {
    pub func_id:       usize,
    pub ip:            usize,
    pub locals:        Vec<Value>,    // moved out during resume, moved back after
    pub is_done:       bool,
    pub yielded_value: Option<Value>, // cached value for IsDone + Next pattern
}
```

### 恢复序列（`resume_fiber`）

1. 通过 `std::mem::take` 从 `FiberState` **移动**读取 `func_id`、`ip` 和 `locals` — 无克隆。
2. 使用移动后的 locals 从 `fiber.ip` 运行 `execute_bytecode`。
3. 遇到 `Yield`：设置 `fiber_yielded = true`。将 locals 移回 `FiberState`。更新 `fiber.ip`。返回 yield 的值。
4. 遇到 `Return` / 字节码结束：设置 `fiber.is_done = true`。返回最终值。

恢复/挂起除初始 `Vec` 创建外不涉及堆分配 — 仅移动。

### IsDone / Next 模式

`IsDone` 检查 `FiberState::is_done`，同时考虑 `yielded_value` 是否已缓存。`Next` 若存在则取缓存的 `yielded_value`（来自先前已运行的 resume），否则调用 `resume_fiber`。这确保 `for x in fiber` 循环永远不会双重推进纤程。

### 遍历纤程的 For 循环（`ForIterType::Fiber`）

编译器发射：
1. `MethodCall(IsDone)` → `JumpIfTrue` 退出
2. `MethodCall(Next)` → 赋值给循环变量
3. 循环体
4. `Jump` 回到步骤 1
5. 遇到 `break`：`MethodCall(Close, base=fiber_reg)` 在跳出前将纤程标记为完成

---

## HTTP 服务器（`HttpServe`）

`HttpServe` 启动 `tiny_http::Server` 并生成 N 个 OS 线程：

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
1. 将 `"METHOD /path"` 键与路由表匹配（不区分大小写）。
2. 构建 JSON 对象 `{ method, url, body, ip, headers }` 作为 `Value::Json`。
3. 将 `tiny_http::Request` 存储在 `Arc<Mutex<Option<tiny_http::Request>>>` 中，通过 `http_req` 传递给新的 `Executor`。
4. 在该工作线程的 `Executor` 中同步运行匹配的处理纤程。
5. 当处理程序调用 `net.respond(...)` 时，VM 执行 `HttpRespond` 发送响应并返回 `OpResult::Yield` 以结束处理程序。
6. 若处理程序退出时未调用 `net.respond`，工作线程发送 `500` 回退响应。

### 优雅关闭

`SHUTDOWN` 是 `vm.rs` 中的 `pub static AtomicBool`。`main.rs` 中的 Ctrl+C 处理程序将其设为 `true`。工作线程在每次 `recv_timeout(100ms)` 周期检查它。主线程也在 `sleep(500ms)` 循环中阻塞并轮询 `SHUTDOWN`。一旦设置，所有循环退出，进程干净终止。

---

## 编译器（`src/backend/mod.rs`）

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

局部变量（具名）通过 `define_local(id, slot)` 分配到槽位，并存储在 `scopes: Vec<HashMap<StringId, usize>>` 中。临时变量使用 `push_reg()`/`pop_reg()` — 表达式结果被消费后复用。

`max_locals_used` 会被记录，以便 `FunctionChunk::max_locals` 在函数调用时能预分配大小恰好合适的 locals 向量。

### 常量去重

`CompileContext::add_constant` 通过 `string_constants: HashMap<String, usize>` 对字符串常量去重。重复的字符串常量（例如作为方法名参数多次出现的 `"insert"`）复用同一常量表槽位。

### 两遍编译

**第一遍** — `register_globals_recursive`：
- 为每个全局变量和 fiber-decl 实例分配槽位索引
- 为每个函数/纤程分配函数索引
- 在 `functions: Vec<FunctionChunk>` 中预分配空的 `FunctionChunk` 槽位

**第二遍** — `compile_stmt` / `compile_expr`：
- 通过 `emit(op, span)` 发射与 span 配对的字节码
- `main` 中的顶层语句使用 `GetVar`/`SetVar`（全局变量）；嵌套语句使用寄存器

### `FunctionChunk`

```rust
pub struct FunctionChunk {
    pub bytecode:   Arc<Vec<OpCode>>,
    pub spans:      Arc<Vec<Span>>,    // spans[i] corresponds to bytecode[i]
    pub is_fiber:   bool,
    pub max_locals: usize,
}
```

字节码和 spans 被 `Arc` 包装，以便在 HTTP 工作线程之间共享而无需复制。

---

## 运行时错误报告

VM 中的每个运行时错误都会追加 `self.current_span_info(ip)`，它通过查找 `current_spans[ip - 1]` 返回 `" [line: X, col: Y]"`。示例：

```
ERROR: R303: Array index out of bounds: 5 [line: 14, col: 7]
```

`current_spans` 在每次 `run_frame` 调用时切换到正确的 `Arc<Vec<Span>>`，退出时恢复。

---

## 内存模型

- **无垃圾收集器**。通过 `Arc` 进行引用计数（对 NaN-boxed 指针值手动 `inc_ref`/`dec_ref`）。
- **标量值**（Int、Float、Bool、Date、函数索引）：完全存储在 `u64` 内 — 零堆分配。
- **集合值**：`Arc<RwLock<T>>` 提供共享所有权。克隆集合 `Value` 仅递增 `Arc` 计数器。
- **变更**：`.insert()`、`.update()`、`.delete()` 获取写锁。同一集合的所有句柄都能看到变更。
- **只读方法**：`.size()`、`.get()`、`.contains()` 获取读锁 — 允许多个并发读者。

---

## 安全控制

### 网络 SSRF 防护（`is_safe_url`）

被阻止的目标：
- `file://` URL
- `169.254.x.x`（链路本地 / AWS 元数据端点）
- 私有范围：`10.x`、`192.168.x`、`172.16–31.x`（非 localhost 时）

应用于 `HttpCall` 和 `HttpRequest`。

### HTTP 响应体限制

在 `HttpServe` 中，`into_string()` 后检查响应体。若超过 **10 MB**，则返回 `413` 错误 JSON，而非实际响应体。

### CORS 头

所有 `HttpServe` 响应自动包含：
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

`OPTIONS` 预检请求收到 `204` 响应，不调用任何处理程序。
