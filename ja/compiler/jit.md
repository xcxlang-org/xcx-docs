# XCX トレーシング JIT（Cranelift）— ドキュメント

> **ファイル：** `src/backend/jit.rs`  
> XCX 3.1 には、[Cranelift](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift) フレームワークを使用してホットループと頻繁に呼び出される関数の両方をネイティブマシンコードに自動コンパイルする**デュアルモード JIT** が含まれています。

---

## 目次

1. [概要](#概要)
2. [2 つの JIT モード](#2-つの-jit-モード)
3. [トレーシング JIT — ホットループ検出](#トレーシング-jit--ホットループ検出)
4. [トレース記録](#トレース記録)
5. [トレースコンパイル](#トレースコンパイル)
6. [トレース実行](#トレース実行)
7. [メソッド JIT — 関数コンパイル](#メソッド-jit--関数コンパイル)
8. [TraceOp バリアント](#traceop-バリアント)
9. [エクスポート C 関数](#エクスポート-c-関数)
10. [JIT 設定](#jit-設定)

---

## 概要

JIT は開発者にとって**完全に透過的**です — 自動的に有効化され、型ガードまたはサポートされていない操作に遭遇するとインタプリタにフォールバックします。

2 つの独立したコンパイルパスがあります：

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

## 2 つの JIT モード

### JITFunction（トレーシング）

```rust
pub type JITFunction = unsafe extern "C" fn(
    *mut VMValue,   // locals_ptr
    *mut VMValue,   // globals_ptr
    *const VMValue, // consts_ptr
) -> i32            // next_ip (0 = continue, >0 = side-exit IP, <0 = halt)
```

ホットループトレース用。戻り値は次の命令ポインタです。

### MethodJitFunction（メソッド）

```rust
pub type MethodJitFunction = unsafe extern "C" fn(
    *mut VMValue,   // locals_ptr
    *mut VMValue,   // globals_ptr
    *const VMValue, // consts_ptr
    *mut VM,        // vm_ptr
    *mut Executor,  // executor_ptr
) -> u64            // return value as raw Value bits
```

コンパイル済み完全関数用。NaN-boxed ビットとして関数の戻り値を直接返します。再帰呼び出しとメソッドディスパッチのために VM と Executor へのアクセスが必要です。

---

## トレーシング JIT — ホットループ検出

`target < current_ip`（後方ジャンプ、すなわちループ戻りエッジ）である各 `Jump { target }` で：

1. `hot_counts[target]` がインクリメントされます
2. `hot_counts[target] >= 50` になると、`start_ip = target` で新しい `Trace` が開始されます
3. トレース記録は保護されます：`trace_cache[target].is_none()`（コンパイル済みトレースなし）かつ `is_recording` が false の場合にのみ開始

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

しきい値はメイン `Jump` opcode では **50**、融合ループ opcode（`LoopNext`、`IncVarLoopNext`、`IncLocalLoopNext`）では **1000** です。

---

## トレース記録

`is_recording` が true のとき、インタプリタは各 opcode を通常どおり実行し、**かつ**現在のランタイム型に特化した `TraceOp` を記録します。例：

- 2 つの整数レジスタに対する `Add` は、汎用 `Add` ではなく `GuardInt { reg: src1 }`、`GuardInt { reg: src2 }`、`AddInt { dst, src1, src2 }` を記録します
- **取られない** `JumpIfFalse` は `GuardTrue { reg: src, fail_ip: target }` を記録します — 分岐が常に false であることを主張
- **取られる** `JumpIfFalse` は `GuardFalse { reg: src, fail_ip: next_ip }` を記録します

opcode がトレースできない場合（複雑なメソッド呼び出し、文字列操作など）、記録は中止され `is_recording` は false に戻されます。

### トレース可能なもの

| トレース可能 | トレース不可 |
|---|---|
| Int/Float 算術 | オブジェクトメソッド（ArraySize、ArrayGet、ArrayPush、ArrayUpdate、SetSize、SetContains を除く） |
| Int/Float 比較 | 文字列操作 |
| ブール論理 | 関数呼び出し |
| グローバルアクセス／変更 | I/O、HTTP |
| カウンタインクリメント | Fiber 操作 |
| 配列操作：size、get、push、update | |
| セット操作：size、contains | |
| RandomInt、RandomFloat、RandomChoice | |
| Pow、IntConcat、Has | |
| CastIntToFloat | |

---

## トレースコンパイル

トレースが `start_ip` に戻ると、完全な `Trace` が Cranelift の `JIT::compile()` に渡されます。Cranelift は `TraceOp` シーケンスを `JITFunction` シグネチャ（3 パラメータ、`i32` 戻り値）のネイティブ関数にコンパイルします。

コンパイルされた関数ポインタは `Trace::native_ptr`（`AtomicPtr<u8>`）に格納され、`Arc<Trace>` は `vm.traces`（グローバル共有）と `trace_cache`（Executor ごとの高速パス）の両方に挿入されます。

### Cranelift コンパイラ設定

```rust
flag_builder.set("opt_level", "speed").unwrap();
flag_builder.set("use_colocated_libcalls", "false").unwrap();
flag_builder.set("is_pic", "false").unwrap();
flag_builder.set("regalloc_checker", "false").unwrap();
```

---

## トレース実行

各ループディスパッチ反復で、次の opcode をフェッチする前に：

```rust
if !self.is_recording && current_ip < self.trace_cache.len() {
    if let Some(trace) = &self.trace_cache[current_ip] {
        let jit_res = self.execute_trace(trace, ip, locals, glbs.as_mut().unwrap());
        if let Some(res) = jit_res { return res; }
        continue;
    }
}
```

コンパイル済みネイティブ関数が利用可能な場合（`native_ptr != null`）、`transmute` 経由で直接呼び出されます — ループ本体全体についてインタプリタを完全にバイパスします。JIT がまだコンパイルしていない場合、中間ステップとして解釈実行の `TraceOp` パスが使用されます。

---

## メソッド JIT — 関数コンパイル

ループを**含まない**関数（`has_loops = false`）で、バイトコード命令が 500 未満のものが Method JIT コンパイルの対象です。

### トリガー

同一関数への **10 回目**の呼び出し後、JIT コンパイルが非同期でトリガーされます：

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

コンパイルされたポインタは `FunctionChunk::jit_ptr`（`Arc<AtomicPtr<u8>>`）に格納され、すべてのスレッド間で共有されます。

### 高速パス実行

各 `run_frame_with_guard` 呼び出しで、まず JIT ポインタが確認されます：

```rust
let jit_ptr = chunk.jit_ptr.load(Ordering::Relaxed);
if !jit_ptr.is_null() && !self.is_recording {
    let jit_fn: MethodJitFunction = unsafe { std::mem::transmute(jit_ptr) };
    // Prepare locals from params, call native function directly
    let res_bits = unsafe { jit_fn(locals.as_mut_ptr(), glbs_ptr, consts.as_ptr(), vm_ptr, executor_ptr) };
    return Some(Value(res_bits));
}
```

これにより、コンパイル済み関数についてインタプリタループ、トレーシング機構、`hot_counts`/`trace_cache` の割り当てをすべてバイパスします。

### `compile_method` — 完全バイトコードコンパイル

`JIT::compile_method` は完全な `FunctionChunk` をネイティブコードにコンパイルします。トレースコンパイルより多くの opcode サブセットを処理します：

- すべての算術、比較、論理 opcode（整数のみ）
- 完全な参照カウント付き `LoadConst`、`Move`、`GetVar`、`SetVar`
- 制御フロー：`Jump`、`JumpIfFalse`、`JumpIfTrue`、`Return`、`ReturnVoid`
- ループ opcode：`LoopNext`、`IncLocalLoopNext`、`IncVarLoopNext`、`ArrayLoopNext`
- `MethodCall` — `xcx_jit_method_dispatch` extern C 関数経由でディスパッチ
- `Call` — `xcx_jit_call_recursive` 経由の再帰呼び出し（インラインでサポートされるのは自己再帰呼び出しのみ；その他の関数呼び出しはインタプリタパスにフォールスルー）

サポートされていない opcode は早期 `return` でゼロ値（false）を返し、グレースフルにフォールバックします。

### Method JIT における参照カウント

トレーシング JIT（速度のため ref-counting をスキップ）とは異なり、Method JIT は `xcx_jit_inc_ref` および `xcx_jit_dec_ref` extern C 関数経由で完全な `inc_ref`/`dec_ref` 呼び出しを含みます。コンパイル済み関数がライフタイムを通じてヒープ割り当て値を保持・解放する可能性があるため必要です。

ポインタチェックは ref-count ヘルパーを呼び出す前にタグビットをインラインで検査し、スカラー値に対する不要な関数呼び出しオーバーヘッドを回避します。

---

## 高速パスプーリング

Executor は、深く再帰的または頻繁に呼び出される関数での O(N) 割り当てを避けるため、3 つのプールを維持します：

```rust
hot_counts_pool:   Vec<Vec<usize>>,
trace_cache_pool:  Vec<Vec<Option<Arc<Trace>>>>,
locals_pool:       Vec<Vec<Value>>,
```

`run_frame_with_guard` に入る際、ベクタはプールから取得（または新規割り当て）されます。return 時にプールへ返却されます。これによりホットコードパスでの呼び出しごとのヒープ割り当ての大部分が排除されます。

`locals_pool` は自己再帰コンパイル関数の高速パス用に `xcx_jit_call_recursive` でも使用されます。

---

## TraceOp バリアント

### データ移動

| バリアント | 説明 |
|---|-----
| `LoadConst { dst, val }` | 定数をレジスタにロード |
| `Move { dst, src }` | レジスタをコピー |
| `GetVar { dst, idx }` | グローバルをロード |
| `SetVar { idx, src }` | グローバルを格納 |

### 整数算術

| バリアント | 説明 |
|---|---|
| `AddInt / SubInt / MulInt` | 基本演算（ラッピング） |
| `DivInt / ModInt` | ゼロ除算／オーバーフロー用 `fail_ip` 付き |
| `PowInt` | extern C 関数による累乗 |
| `IntConcat` | 桁連結（123 ++ 456 = 123456） |

### 浮動小数点算術

| バリアント | 説明 |
|---|---|
| `AddFloat / SubFloat / MulFloat` | 基本演算 |
| `DivFloat / ModFloat` | ゼロ除算用 `fail_ip` 付き |
| `PowFloat` | extern C 関数による累乗 |
| `CastIntToFloat` | int → float 変換 |

### 型ガード

| バリアント | 説明 |
|---|---|
| `GuardInt { reg, ip }` | レジスタが Int でない場合 `ip` へサイドエグジット |
| `GuardFloat { reg, ip }` | レジスタが Float でない場合 `ip` へサイドエグジット |
| `GuardTrue { reg, fail_ip }` | ブール値が false の場合サイドエグジット |
| `GuardFalse { reg, fail_ip }` | ブール値が true の場合サイドエグジット |

### 比較

| バリアント | 説明 |
|---|---|
| `CmpInt { dst, src1, src2, cc }` | 条件コード付き整数比較 |
| `CmpFloat { dst, src1, src2, cc }` | 条件コード付き浮動小数点比較 |

**条件コード（cc）：**

| cc | IntCC | FloatCC |
|---|---|---|
| 0 | Equal | Equal |
| 1 | NotEqual | NotEqual |
| 2 | SignedGreaterThan | GreaterThan |
| 3 | SignedLessThan | LessThan |
| 4 | SignedGreaterThanOrEqual | GreaterThanOrEqual |
| 5 | SignedLessThanOrEqual | LessThanOrEqual |

### ループ制御

| バリアント | 説明 |
|---|---|
| `LoopNextInt { reg, limit_reg, target, exit_ip }` | 範囲ループ用インクリメント + 条件付きジャンプ |
| `IncVarLoopNext { g_idx, reg, limit_reg, target, exit_ip }` | グローバルカウンタ + 範囲ループ |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target, exit_ip }` | ローカルカウンタ + 範囲ループ |
| `IncLocal { reg }` | 単純なローカル変数インクリメント |
| `IncVar { g_idx }` | 単純なグローバル変数インクリメント |
| `Jump { target_ip }` | 無条件ジャンプ（ループ戻り検出をトリガー） |

### コレクション

| バリアント | 説明 |
|---|---|
| `ArraySize { dst, src }` | 配列サイズを取得 |
| `ArrayGet { dst, arr_reg, idx_reg, fail_ip }` | 要素を取得（境界チェック付き） |
| `ArrayPush { arr_reg, val_reg }` | 配列に要素を追加 |
| `ArrayUpdate { arr_reg, idx_reg, val_reg, fail_ip }` | インデックスの要素を更新（境界チェック付き） |
| `SetSize { dst, src }` | セットサイズを取得 |
| `SetContains { dst, set_reg, val_reg }` | セットメンバーシップを確認 |

### ランダム

| バリアント | 説明 |
|---|---|
| `RandomInt { dst, min, max, step, has_step }` | 範囲／ステップ付きランダム整数 |
| `RandomFloat { dst, min, max, step, has_step, step_is_float }` | 範囲／ステップ付きランダム浮動小数点 |
| `RandomChoice { dst, src }` | コレクションからランダム要素 |

### 論理

`And / Or / Not` — ブールビットに対する演算

---

## エクスポート C 関数

JIT は自明にインライン化できない操作のために C ABI 経由で外部 Rust 関数を呼び出します。すべて JIT 初期化時に `JITBuilder` に登録されます。

### 算術と数学

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

### コレクション操作

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

### 参照カウント

```rust
#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_inc_ref(v: Value)
// Increments Arc ref count if value is a pointer type

#[unsafe(no_mangle)]
pub extern "C" fn xcx_jit_dec_ref(v: Value)
// Decrements Arc ref count if value is a pointer type; may free memory
```

Method JIT 専用でヒープ割り当て値のライフタイムを管理します。トレーシング JIT は最大ループ性能のため ref-counting をスキップします。

### メソッドディスパッチと再帰呼び出し

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

## JIT 設定

```rust
pub struct JIT {
    builder_context: FunctionBuilderContext,
    pub ctx:         codegen::Context,
    module:          JITModule,
}
```

初期化：
1. ネイティブ ISA で `JITBuilder` を作成（`cranelift_native` によるホスト自動検出）
2. すべての外部 C シンボルを登録（算術、コレクション、ref-counting、ディスパッチ）
3. 最大性能のため `opt_level: "speed"` を設定
4. 実行可能メモリを管理する `JITModule` を作成

### トレースコンパイル（`compile`）

1. コンテキストをクリア（`module.clear_context`）
2. `JITFunction` シグネチャで関数を宣言：`(locals, globals, consts) -> i32`
3. ループの存在を検出（`has_loop`）— `LoopNextInt`、`IncVarLoopNext` などが存在する場合、ops を Cranelift ループブロックでラップ
4. 各 `TraceOp` の Cranelift IR を構築
5. コンパイルして定義を確定
6. コードポインタを返す（`get_finalized_function`）

### メソッドコンパイル（`compile_method`）

1. コンテキストをクリア（`module.clear_context`）
2. `MethodJitFunction` シグネチャで関数を宣言：`(locals, globals, consts, vm, executor) -> u64`
3. 正しい数の Cranelift ブロックを作成するため、すべてのジャンプターゲットについてバイトコードを事前スキャン
4. サポートされる各 `OpCode` の Cranelift IR を構築
5. すべてのブロックを seal して確定
6. コードポインタを返す；`FunctionChunk::jit_ptr` に格納

---

## JIT における NaN-boxing マクロ

JIT は Cranelift IR で NaN-boxed 値を扱うためにマクロを使用します：

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

この最適化は、NaN タグが上位ビットにあり、整数値が下位 48 ビットを占めるため機能します — ワード全体に 1 を加算しても値部分のみが変更されます（48 ビットドメインでオーバーフローがない限り）。同じ高速パスインクリメントは、融合ループ opcode すべてでローカル変数とグローバル変数の両方に使用されます。
