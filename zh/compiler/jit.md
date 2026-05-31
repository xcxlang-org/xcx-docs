# XCX 追踪 JIT（Cranelift）— 文档

> **文件：** `src/backend/jit.rs`  
> XCX 3.1 包含**双模式 JIT**，使用 [Cranelift](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift) 框架自动将热点循环和频繁调用的函数编译为本机机器码。

---

## 目录

1. [概述](#概述)
2. [两种 JIT 模式](#两种-jit-模式)
3. [追踪 JIT — 热点循环检测](#追踪-jit--热点循环检测)
4. [追踪录制](#追踪录制)
5. [追踪编译](#追踪编译)
6. [追踪执行](#追踪执行)
7. [方法 JIT — 函数编译](#方法-jit--函数编译)
8. [TraceOp 变体](#traceop-变体)
9. [导出的 C 函数](#导出的-c-函数)
10. [JIT 配置](#jit-配置)

---

## 概述

JIT 对开发者**完全透明** — 它自动激活，并在遇到类型守卫或不支持的操作时回退到解释器。

存在两条独立的编译路径：

```
[Tracing JIT] — for hot loops
  Interpreter executes loop
      ↓ (50 iterations)
  Trace recording starts
      ↓ (loop returns to start IP)
  Trace compiled by Cranelift → native C function (3-arg signature)
      ↓ (next iteration)
  Native code executed directly (interpreter bypass)
      ↓ (failed guard)
  Fallback to interpreter at the correct IP

[Method JIT] — for frequently-called functions without loops
  Function is called
      ↓ (10 calls)
  Entire function bytecode compiled by Cranelift → native C function (5-arg signature)
      ↓ (next call)
  Native code called directly (bypasses interpreter + tracing machinery)
```

---

## 两种 JIT 模式

### JITFunction（追踪）

```rust
pub type JITFunction = unsafe extern "C" fn(
    *mut VMValue,   // locals_ptr
    *mut VMValue,   // globals_ptr
    *const VMValue, // consts_ptr
) -> i32            // next_ip (0 = continue, >0 = side-exit IP, <0 = halt)
```

用于热点循环追踪。返回值为下一条指令指针。

### MethodJitFunction（方法）

```rust
pub type MethodJitFunction = unsafe extern "C" fn(
    *mut VMValue,   // locals_ptr
    *mut VMValue,   // globals_ptr
    *const VMValue, // consts_ptr
    *mut VM,        // vm_ptr
    *mut Executor,  // executor_ptr
) -> u64            // return value as raw Value bits
```

用于编译完整函数。直接以 NaN-boxed 位形式返回函数返回值。需要访问 VM 和 Executor 以进行递归调用和方法分派。

---

## 追踪 JIT — 热点循环检测

在每次 `Jump { target }` 且 `target < current_ip`（向后跳转，即循环回边）时：

1. `hot_counts[target]` 递增
2. 当 `hot_counts[target] >= 50` 时，以 `start_ip = target` 启动新的 `Trace`
3. 追踪录制受保护：仅当 `trace_cache[target].is_none()`（无已编译追踪）且 `is_recording` 为 false 时才开始

```rust
fn check_start_recording(&mut self, target_ip: usize, threshold: usize) {
    if target_ip < self.hot_counts.len() {
        let hc = unsafe { self.hot_counts.get_unchecked_mut(target_ip) };
        *hc += 1;
        if *hc >= threshold && !self.is_recording && self.trace_cache[target_ip].is_none() {
            self.recording_trace = Some(Trace { ops: vec![], start_ip: target_ip, ... });
            self.is_recording = true;
        }
    }
}
```

主 `Jump` 操作码的阈值为 **50**，融合循环操作码（`LoopNext`、`IncVarLoopNext`、`IncLocalLoopNext`）的阈值为 **1000**。

---

## 追踪录制

当 `is_recording` 为 true 时，解释器正常执行每条操作码，**并**为当前运行时类型录制专门的 `TraceOp`。例如：

- 两个整数寄存器上的 `Add` 录制 `GuardInt { reg: src1 }`、`GuardInt { reg: src2 }`、`AddInt { dst, src1, src2 }` — 而非通用 `Add`
- **未**执行的 `JumpIfFalse` 录制 `GuardTrue { reg: src, fail_ip: target }` — 断言分支始终为 false
- **已**执行的 `JumpIfFalse` 录制 `GuardFalse { reg: src, fail_ip: next_ip }`

若某操作码无法追踪（复杂方法调用、字符串操作等），则中止录制并将 `is_recording` 设回 false。

### 可追踪的内容

| 可追踪 | 不可追踪 |
|---|---|
| Int/Float 算术 | 对象方法（除 ArraySize、ArrayGet、ArrayPush、ArrayUpdate、SetSize、SetContains） |
| Int/Float 比较 | 字符串操作 |
| 布尔逻辑 | 函数调用 |
| 全局访问/修改 | I/O、HTTP |
| 计数器递增 | 纤程操作 |
| 数组操作：size、get、push、update | |
| 集合操作：size、contains | |
| RandomInt、RandomFloat、RandomChoice | |
| Pow、IntConcat、Has | |
| CastIntToFloat | |

---

## 追踪编译

当追踪返回到 `start_ip` 时，完整的 `Trace` 被传递给 Cranelift 中的 `JIT::compile()`。Cranelift 将 `TraceOp` 序列编译为具有 `JITFunction` 签名（3 个参数，`i32` 返回）的本机函数。

编译后的函数指针存储在 `Trace::native_ptr`（`AtomicPtr<u8>`）中，`Arc<Trace>` 被插入 `vm.traces`（全局共享）和 `trace_cache`（每个 Executor 的快速路径）。

### Cranelift 编译器设置

```rust
flag_builder.set("opt_level", "speed").unwrap();
flag_builder.set("use_colocated_libcalls", "false").unwrap();
flag_builder.set("is_pic", "false").unwrap();
flag_builder.set("regalloc_checker", "false").unwrap();
```

---

## 追踪执行

在每次循环分派迭代中，获取下一条操作码之前：

```rust
if !self.is_recording && current_ip < self.trace_cache.len() {
    if let Some(trace) = &self.trace_cache[current_ip] {
        let jit_res = self.execute_trace(trace, ip, locals, glbs.as_mut().unwrap());
        if let Some(res) = jit_res { return res; }
        continue;
    }
}
```

若有可用的已编译本机函数（`native_ptr != null`），则通过 `transmute` 直接调用 — 完全绕过解释器执行整个循环体。若 JIT 尚未编译，则使用解释型 `TraceOp` 路径作为中间步骤。

---

## 方法 JIT — 函数编译

**无循环**（`has_loops = false`）且字节码指令少于 500 条的函数符合方法 JIT 编译条件。

### 触发条件

同一函数被调用 **10 次**后，异步触发 JIT 编译：

```rust
let count = chunk.call_count.fetch_add(1, Ordering::Relaxed);
if count == 10 {
    let mut jit = vm_copy.jit.lock();
    match jit.compile_method(func_id_copy, &chunk_copy, &self.ctx.constants) {
        Ok(ptr) => {
            chunk_copy.jit_ptr.store(ptr as *mut u8, Ordering::Release);
        }
        Err(_) => {}
    }
}
```

编译后的指针存储在 `FunctionChunk::jit_ptr`（`Arc<AtomicPtr<u8>>`）中，在所有线程间共享。

### 快速路径执行

在每次 `run_frame_with_guard` 调用时，首先检查 JIT 指针：

```rust
let jit_ptr = chunk.jit_ptr.load(Ordering::Relaxed);
if !jit_ptr.is_null() && !self.is_recording {
    let jit_fn: MethodJitFunction = unsafe { std::mem::transmute(jit_ptr) };
    // Prepare locals from params, call native function directly
    let res_bits = unsafe { jit_fn(locals.as_mut_ptr(), glbs_ptr, consts.as_ptr(), vm_ptr, executor_ptr) };
    return Some(Value(res_bits));
}
```

这会绕过整个解释器循环、追踪机制，以及已编译函数的 hot_counts/trace_cache 分配。

### `compile_method` — 完整字节码编译

`JIT::compile_method` 将完整的 `FunctionChunk` 编译为本机代码。它处理的操作码子集比追踪编译更大：

- 所有算术、比较和逻辑操作码（仅整数）
- `LoadConst`、`Move`、`GetVar`、`SetVar`，含完整引用计数
- 控制流：`Jump`、`JumpIfFalse`、`JumpIfTrue`、`Return`、`ReturnVoid`
- 循环操作码：`LoopNext`、`IncLocalLoopNext`、`IncVarLoopNext`、`ArrayLoopNext`
- `MethodCall` — 通过 `xcx_jit_method_dispatch` extern C 函数分派
- `Call` — 通过 `xcx_jit_call_recursive` 递归调用（仅支持自递归调用内联；其他函数调用回退到解释器路径）

不支持的操作码会导致提前 `return` 零值（false），从而优雅回退。

### 方法 JIT 中的引用计数

与追踪 JIT（为速度跳过引用计数）不同，方法 JIT 通过 `xcx_jit_inc_ref` 和 `xcx_jit_dec_ref` extern C 函数包含完整的 `inc_ref`/`dec_ref` 调用。这是必需的，因为编译后的函数可能在生命周期内持有并释放堆分配的值。

指针检查通过检查标签位内联完成，再调用引用计数辅助函数，避免对标量值的不必要函数调用开销。

---

## 快速路径池化

Executor 维护三个池，以避免在深度递归或频繁调用的函数中进行 O(N) 分配：

```rust
hot_counts_pool:   Vec<Vec<usize>>,
trace_cache_pool:  Vec<Vec<Option<Arc<Trace>>>>,
locals_pool:       Vec<Vec<Value>>,
```

进入 `run_frame_with_guard` 时，从池中获取向量（或新分配）。返回时，将向量归还池中。这消除了热点代码路径中大多数每次调用的堆分配。

`locals_pool` 也被 `xcx_jit_call_recursive` 用于自递归编译函数的快速路径。

---

## TraceOp 变体

### 数据移动

| 变体 | 说明 |
|---|-----
| `LoadConst { dst, val }` | 将常量加载到寄存器 |
| `Move { dst, src }` | 复制寄存器 |
| `GetVar { dst, idx }` | 加载全局变量 |
| `SetVar { idx, src }` | 存储全局变量 |

### 整数算术

| 变体 | 说明 |
|---|---|
| `AddInt / SubInt / MulInt` | 基本运算（环绕） |
| `DivInt / ModInt` | 含 `fail_ip`，用于除零/溢出 |
| `PowInt` | 通过 extern C 函数求幂 |
| `IntConcat` | 数字拼接（123 ++ 456 = 123456） |

### 浮点算术

| 变体 | 说明 |
|---|---|
| `AddFloat / SubFloat / MulFloat` | 基本运算 |
| `DivFloat / ModFloat` | 含 `fail_ip`，用于除零 |
| `PowFloat` | 通过 extern C 函数求幂 |
| `CastIntToFloat` | int → float 转换 |

### 类型守卫

| 变体 | 说明 |
|---|---|
| `GuardInt { reg, ip }` | 若寄存器不是 Int，则侧向退出到 `ip` |
| `GuardFloat { reg, ip }` | 若寄存器不是 Float，则侧向退出到 `ip` |
| `GuardTrue { reg, fail_ip }` | 若布尔值为 false，则侧向退出 |
| `GuardFalse { reg, fail_ip }` | 若布尔值为 true，则侧向退出 |

### 比较

| 变体 | 说明 |
|---|---|
| `CmpInt { dst, src1, src2, cc }` | 带条件码的整数比较 |
| `CmpFloat { dst, src1, src2, cc }` | 带条件码的浮点比较 |

**条件码（cc）：**

| cc | IntCC | FloatCC |
|---|---|---|
| 0 | Equal | Equal |
| 1 | NotEqual | NotEqual |
| 2 | SignedGreaterThan | GreaterThan |
| 3 | SignedLessThan | LessThan |
| 4 | SignedGreaterThanOrEqual | GreaterThanOrEqual |
| 5 | SignedLessThanOrEqual | LessThanOrEqual |

### 循环控制

| 变体 | 说明 |
|---|---|
| `LoopNextInt { reg, limit_reg, target, exit_ip }` | 范围循环的递增 + 条件跳转 |
| `IncVarLoopNext { g_idx, reg, limit_reg, target, exit_ip }` | 全局计数器 + 范围循环 |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target, exit_ip }` | 局部计数器 + 范围循环 |
| `IncLocal { reg }` | 简单局部变量递增 |
| `IncVar { g_idx }` | 简单全局变量递增 |
| `Jump { target_ip }` | 无条件跳转（触发循环返回检测） |

### 集合

| 变体 | 说明 |
|---|---|
| `ArraySize { dst, src }` | 获取数组大小 |
| `ArrayGet { dst, arr_reg, idx_reg, fail_ip }` | 获取元素（含边界检查） |
| `ArrayPush { arr_reg, val_reg }` | 向数组添加元素 |
| `ArrayUpdate { arr_reg, idx_reg, val_reg, fail_ip }` | 更新索引处元素（含边界检查） |
| `SetSize { dst, src }` | 获取集合大小 |
| `SetContains { dst, set_reg, val_reg }` | 检查集合成员关系 |

### 随机数

| 变体 | 说明 |
|---|---|
| `RandomInt { dst, min, max, step, has_step }` | 带范围/步长的随机整数 |
| `RandomFloat { dst, min, max, step, has_step, step_is_float }` | 带范围/步长的随机浮点数 |
| `RandomChoice { dst, src }` | 从集合中随机选取元素 |

### 逻辑

`And / Or / Not` — 对布尔位进行运算

---

## 导出的 C 函数

JIT 通过 C ABI 调用外部 Rust 函数，以处理无法简单内联的操作。所有函数在 JIT 初始化期间注册到 `JITBuilder`。

### 算术与数学

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_random_int(min: i64, max: i64, step: i64, has_step: bool) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_random_float(min: f64, max: f64, step: f64, has_step: bool) -> f64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_pow_int(a: i64, b: i64) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_pow_float(a: f64, b: f64) -> f64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_int_concat(a: i64, b: i64) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_has(container: Value, item: Value) -> bool

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_random_choice(col: Value) -> Value
```

### 集合操作

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_size(arr: Value) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_get(arr: Value, idx: i64) -> Value

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_push(arr: Value, val: Value)

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_array_update(arr: Value, idx: i64, val: Value) -> i32
// Returns 1 on success, 0 on out-of-bounds (triggers JIT side-exit)

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_set_size(set: Value) -> i64

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_set_contains(set: Value, val: Value) -> bool
```

### 引用计数

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_inc_ref(v: Value)
// Increments Arc ref count if value is a pointer type

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_dec_ref(v: Value)
// Decrements Arc ref count if value is a pointer type; may free memory
```

专供方法 JIT 管理堆分配值的生命周期。追踪 JIT 为获得最大循环性能而跳过引用计数。

### 方法分派与递归调用

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_method_dispatch(
    dst: u8,
    kind: u8,           // MethodKind cast to u8
    receiver: Value,
    args_ptr: *const Value,
    arg_count: u8,
    locals_ptr: *mut Value,
    executor_ptr: *mut Executor,
)
// Dispatches a built-in method call from within a compiled function.
// Delegates to Executor::handle_method_call or handle_database_method.

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_call_recursive(
    func_id_idx: usize,
    params_ptr: *const Value,
    params_count: u8,
    vm_ptr: *const VM,
    executor_ptr: *mut Executor,
    globals_ptr: *mut Value,
) -> u64
// Calls a function from within a compiled function.
// Fast path: if the target function is also JIT-compiled, calls it directly
//            using locals from the executor's pool (avoids locking).
// Slow path: falls back to run_frame_with_guard for uncompiled functions.
```

---

## JIT 配置

```rust
pub struct JIT {
    builder_context: FunctionBuilderContext,
    pub ctx:         codegen::Context,
    module:          JITModule,
}
```

初始化：
1. 使用本机 ISA 创建 `JITBuilder`（通过 `cranelift_native` 自动检测主机）
2. 注册所有外部 C 符号（算术、集合、引用计数、分派）
3. 设置 `opt_level: "speed"` 以获得最大性能
4. 创建管理可执行内存的 `JITModule`

### 追踪编译（`compile`）

1. 清除上下文（`module.clear_context`）
2. 以 `JITFunction` 签名声明函数：`(locals, globals, consts) -> i32`
3. 检测循环存在（`has_loop`）— 若存在 `LoopNextInt`、`IncVarLoopNext` 等，将 ops 包装在 Cranelift 循环块中
4. 为每个 `TraceOp` 构建 Cranelift IR
5. 编译并 finalize 定义
6. 返回代码指针（`get_finalized_function`）

### 方法编译（`compile_method`）

1. 清除上下文（`module.clear_context`）
2. 以 `MethodJitFunction` 签名声明函数：`(locals, globals, consts, vm, executor) -> u64`
3. 预扫描字节码以获取所有跳转目标，创建正确数量的 Cranelift 块
4. 为每个支持的 `OpCode` 构建 Cranelift IR
5. seal 所有块并 finalize
6. 返回代码指针；存储在 `FunctionChunk::jit_ptr`

---

## JIT 中的 NaN-boxing 宏

JIT 使用宏在 Cranelift IR 中处理 NaN-boxed 值：

```rust
// Unpack integer (sign-extend from 48 bits)
macro_rules! unpack_int {
    ($val:expr) => {{
        let shl = b.ins().ishl_imm($val, 16);
        b.ins().sshr_imm(shl, 16)
    }};
}

// Pack integer
macro_rules! pack_int {
    ($raw:expr) => {{
        let lo = b.ins().band($raw, mask_48);
        b.ins().bor(qnan_tag_int, lo)
    }};
}

// Optimization: For NaN-boxed Int, increment is a simple iadd_imm(val, 1)
// on the entire 64-bit word — tag bits remain unaffected!
let lnxt_bits = b.ins().iadd_imm(lv, 1);
b.ins().store(trusted(), lnxt_bits, la, 0);
```

此优化有效是因为 NaN 标签位于高位，整数值占据低 48 位 — 对整个字加 1 仅改变值部分（只要 48 位域内无溢出）。所有融合循环操作码中对局部和全局变量均使用相同的快速路径递增。
