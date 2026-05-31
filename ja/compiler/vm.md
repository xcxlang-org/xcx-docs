# XCX 仮想マシン（VM）— v3.1

XCX VM は、Cranelift 上に構築されたトレーシング JIT コンパイラによって拡張された、XCX バイトコードを実行するためのカスタム**レジスタベース**ランタイムです。

## アーキテクチャ

- **ファイル**: `src/backend/vm.rs`
- **実行モデル**: フラットなレジスタファイル上での Fetch-Decode-Execute ループ（`execute_bytecode`）
- **レジスタファイル**: フレームごとに所有される `Vec<Value>`、`u8` スロット番号でインデックス
- **グローバル**: すべてのワーカースレッド間で共有される `Arc<RwLock<Vec<Value>>>` による単一のフラット `Vec<Value>`
- **JIT**: `src/backend/jit.rs` — ホットトレース用の Cranelift ベースネイティブコードコンパイラ

### VM 状態

```rust
pub struct VM {
    pub globals:     Arc<RwLock<Vec<Value>>>,
    pub error_count: AtomicUsize,
    pub traces:      Arc<RwLock<HashMap<usize, Arc<Trace>>>>,
    pub jit:         Mutex<JIT>,
}
```

`VM` は `Arc<VM>` でラップされ、HTTP ワーカースレッド間で共有されます。各ワーカーはプライベートなローカルを持つ独自の `Executor` を作成します。

### Executor 状態

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

`SharedContext` は安価にクローンできます（2 つの `Arc` ポインタのインクリメント）し、各ワーカースレッドに独立して渡されます。ディープコピーは発生しません。

---

## 値表現：NaN-boxing

すべての値は単一の `Value(u64)` — 64 ビットワードです。XCX は **NaN-boxing** を使用します：IEEE 754 quiet NaN のビットパターンを型タグプレフィックスとして再利用します。

```
Bit layout: [63..52: exponent/QNAN] [51..48: type tag] [47..0: payload]

Float : stored directly as f64 bits — does NOT have QNAN_BASE prefix set
Int   : QNAN_BASE | TAG_INT  | (i48 value & 0x0000_FFFF_FFFF_FFFF)
Bool  : QNAN_BASE | TAG_BOOL | (0 or 1)
Date  : QNAN_BASE | TAG_DATE | (i48 timestamp ms)
Ptr   : QNAN_BASE | TAG_XXX  | (pointer & 0x0000_FFFF_FFFF_FFFF)
```

### タグ定数

| 定数 | 値 | 型 |
|---|---|---|
| `QNAN_BASE` | `0x7FF0_0000_0000_0000` | base NaN marker |
| `TAG_INT`   | `0x0001_0000_0000_0000` | 48-bit signed integer |
| `TAG_BOOL`  | `0x0002_0000_0000_0000` | boolean (payload 0/1) |
| `TAG_DATE`  | `0x0003_0000_0000_0000` | 48-bit timestamp (ms) |
| `TAG_STR`   | `0x0004_0000_0000_0000` | `Arc<String>` pointer |
| `TAG_ARR`   | `0x0005_0000_0000_0000` | `Arc<RwLock<Vec<Value>>>` pointer |
| `TAG_SET`   | `0x0006_0000_0000_0000` | `Arc<RwLock<SetData>>` pointer |
| `TAG_MAP`   | `0x0007_0000_0000_0000` | `Arc<RwLock<Vec<(Value,Value)>>>` pointer |
| `TAG_TBL`   | `0x0008_0000_0000_0000` | `Arc<RwLock<TableData>>` pointer |
| `TAG_FUNC`  | `0x0009_0000_0000_0000` | function index (u32) |
| `TAG_ROW`   | `0x000A_0000_0000_0000` | `Arc<RowRef>` pointer |
| `TAG_JSON`  | `0x000B_0000_0000_0000` | `Arc<RwLock<serde_json::Value>>` pointer |
| `TAG_FIB`   | `0x000C_0000_0000_0000` | `Arc<RwLock<FiberState>>` pointer |

ポインタペイロードは下位 48 ビットのみを使用します — ユーザースペースポインタが 48 ビットに収まる x86-64 および AArch64 プラットフォームすべてで有効です。

### ポインタ値の参照カウント

ポインタタグ付き値は `Arc` 参照カウントを保持します。VM は代入、return、コレクション変更のたびに `inc_ref()` / `dec_ref()` で手動管理し、ガベージコレクタなしでヒープ割り当てオブジェクト（文字列、配列、JSON、fiber など）が参照されなくなったときに解放されるようにします。

---

## 命令セット（OpCodes）

すべての opcode はレジスタベースです：オペランドスタックではなく、名前付き `u8` レジスタスロットを参照します。

### レジスタ／変数の移動

| OpCode | 説明 |
|---|---|
| `LoadConst { dst, idx }` | `constants[idx]` をレジスタ `dst` にロード |
| `Move { dst, src }` | レジスタ `src` を `dst` にコピー |
| `GetVar { dst, idx }` | `globals[idx]` を `dst` にロード（globals を read-lock） |
| `SetVar { idx, src }` | `src` を `globals[idx]` に書き込み（globals を write-lock） |

### 算術

すべての算術演算は 3 レジスタ：`dst = src1 OP src2`。ランタイム型ディスパッチにより、整数、浮動小数点、文字列連結、日付算術、セット演算の各パスが選択されます。

`Add`, `Sub`, `Mul`, `Div`, `Mod`, `Pow`, `IntConcat` (`++`)

### 比較（結果は Bool）

`Equal`, `NotEqual`, `Greater`, `Less`, `GreaterEqual`, `LessEqual`

### 論理

`And { dst, src1, src2 }`, `Or { dst, src1, src2 }`, `Not { dst, src }`, `Has { dst, src1, src2 }`

### 制御フロー

| OpCode | 説明 |
|---|---|
| `Jump { target }` | 無条件ジャンプ；後方ジャンプ時に `hot_counts[target]` をインクリメント |
| `JumpIfFalse { src, target }` | `src` が `Bool(false)` のときジャンプ |
| `JumpIfTrue { src, target }` | `src` が `Bool(true)` のときジャンプ |
| `Call { dst, func_idx, base, arg_count }` | 関数呼び出し；引数は `locals[base..base+arg_count]`；結果 → `dst` |
| `Return { src }` | 現在のフレームから `src` の値を return |
| `ReturnVoid` | 値なしで return |
| `Halt` | 実行停止（明示的、または回復不能エラー時） |

### コレクション

| OpCode | 説明 |
|---|---|
| `ArrayInit { dst, base, count }` | `base` から `count` 個のレジスタを収集 → `dst` に新しい Array |
| `SetInit { dst, base, count }` | `count` 個のレジスタを収集 → `dst` に新しい Set |
| `SetRange { dst, start, end, step, has_step }` | レジスタ値から範囲 Set を構築 |
| `MapInit { dst, base, count }` | `count` 個のキー値ペア（`base` から交互のペアのレジスタ）を収集 → 新しい Map |
| `TableInit { dst, skeleton_idx, base, row_count }` | 列スキーマ定数 + 行値から Table を構築 |

### セット演算

`SetUnion`, `SetIntersection`, `SetDifference`, `SetSymDifference` — すべて 3 レジスタ、両オペランドは `TAG_SET` である必要があります。

### メソッドディスパッチ

| OpCode | 説明 |
|---|---|
| `MethodCall { dst, kind, base, arg_count }` | `MethodKind` enum による組み込みメソッドのディスパッチ — ランタイムでの文字列ルックアップなし |
| `MethodCallCustom { dst, method_name_idx, base, arg_count }` | 定数テーブルの文字列による動的メソッド（JSON フィールド、エイリアス）のディスパッチ |

`base` はレシーバレジスタを指します；引数は `locals[base+1..base+1+arg_count]` です。`MethodKind` は約 50 の組み込みメソッド（`Push`, `Pop`, `Get`, `Insert`, `Update`, `Delete`, `Where`, `Join`, `Sort`, `Format`, `Next`, `IsDone`, `Close` など）をカバーする `#[derive(Copy)]` enum です。コンパイラは `map_method_kind()` 経由でコンパイル時にメソッド名を `MethodKind` バリアントに解決します。

### Fiber 操作

| OpCode | 説明 |
|---|---|
| `FiberCreate { dst, func_idx, base, arg_count }` | `FiberState` を割り当て、引数から locals を事前設定、`dst` に `Fiber` を格納 |
| `Yield { src }` | fiber を一時停止、`src` の値を呼び出し元に return |
| `YieldVoid` | void fiber を一時停止 |

### I/O とシステム

| OpCode | 説明 |
|---|---|
| `Print { src }` | `locals[src]` を stdout に出力 |
| `Input { dst }` | stdin から 1 行読み取り → `dst` |
| `Wait { src }` | `src` ミリ秒スリープ |
| `HaltAlert { src }` | アラートメッセージを出力、実行を継続 |
| `HaltError { src }` | エラー + span 情報を出力、フレームを停止、error count をインクリメント |
| `HaltFatal { src }` | fatal + span 情報を出力、フレームを停止、error count をインクリメント |
| `TerminalExit` | `std::process::exit(0)` |
| `TerminalClear` | ANSI エスケープまたは OS コマンドでターミナルをクリア |
| `TerminalRun { dst, cmd_src }` | 外部コマンドを実行、結果 → `dst` |
| `EnvGet { dst, src }` | `src` で名付けられた環境変数を読み取り → `dst` |
| `EnvArgs { dst }` | CLI 引数の `Array<String>` を `dst` に push |

### HTTP

| OpCode | 説明 |
|---|---|
| `HttpCall { dst, method_idx, url_src, body_src }` | `ureq` によるシンプルな HTTP 呼び出し（GET/POST など）、結果 JSON → `dst` |
| `HttpRequest { dst, arg_src }` | 設定 map からの完全な HTTP 呼び出し（method、url、headers、body、timeout）→ `dst` |
| `HttpRespond { status_src, body_src, headers_src }` | ハンドラ fiber 内から HTTP レスポンスを送信；制御を返すために `Yield` をトリガー |
| `HttpServe { func_idx, port_src, host_src, workers_src, routes_src }` | `tiny_http` サーバーを起動、ワーカースレッドを spawn、`SHUTDOWN` までメインスレッドをブロック |

### ストレージ

`StoreWrite { base }`, `StoreRead { dst, base }`, `StoreAppend { base }`, `StoreExists { dst, base }`, `StoreDelete { base }`

### JSON

`JsonParse { dst, src }`, `JsonBind { idx, json_src, path_src }`, `JsonBindLocal { dst, json_src, path_src }`, `JsonInject { table_idx, json_src, mapping_src }`, `JsonInjectLocal { table_reg, json_src, mapping_src }`

### 型キャスト

`CastInt { dst, src }`, `CastFloat { dst, src }`, `CastString { dst, src }`, `CastBool { dst, src }`

### 暗号と日付

`CryptoHash { dst, pass_src, alg_src }`, `CryptoVerify { dst, pass_src, hash_src, alg_src }`, `CryptoToken { dst, len_src }`, `DateNow { dst }`

### ループ最適化

これらの opcode はコンパイラによって発行され、一般的なループカウンタパターンを単一命令に融合し、ディスパッチオーバーヘッドを削減して JIT のトレーサビリティを向上させます。

| OpCode | 説明 |
|---|---|
| `IncLocal { reg }` | レジスタ `reg` の整数を 1 インクリメント |
| `IncVar { idx }` | `idx` のグローバルを 1 インクリメント |
| `LoopNext { reg, limit_reg, target }` | `reg` をインクリメント、`reg <= limit_reg` なら `target` へジャンプ、さもなければ fall through |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target }` | `inc_reg`（別カウンタ、例：配列インデックス）をインクリメント、`reg`（ループ変数）をインクリメント、条件付きジャンプ |
| `IncVarLoopNext { g_idx, reg, limit_reg, target }` | `IncLocalLoopNext` と同様だが `g_idx` はグローバルカウンタ |

---

## トレーシング JIT コンパイラ

### 概要

XCX 2.2 には、[Cranelift](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift) コード生成フレームワークを使用してホットループをネイティブマシンコードに自動コンパイルする**トレーシング JIT** が含まれています。

JIT はプログラマにとって完全に透過的です — 自動的に有効化され、型ガードまたはサポートされていない操作でインタプリタにフォールバックします。

### トレース検出

`target < current_ip`（後方、すなわちループバックエッジ）である各 `Jump { target }` で：

1. `hot_counts[target]` がインクリメントされます。
2. `hot_counts[target] >= 50` になると、`start_ip = target` で新しい `Trace` が開始されます。
3. トレース記録はゲートされます：`trace_cache[target].is_none()`（コンパイル済みトレースがまだ存在しない）かつ `is_recording` が false の場合にのみ開始します。

### トレース記録

`is_recording` が true の間、インタプリタは各 opcode を通常どおり実行し、**かつ**現在のランタイム型に特化した `TraceOp` を記録します。例：

- 2 つの整数レジスタに対する `Add` は、汎用 `Add` ではなく `GuardInt { reg: src1 }`、`GuardInt { reg: src2 }`、`AddInt { dst, src1, src2 }` を記録します。
- **取られない** `JumpIfFalse` は `GuardTrue { reg: src, fail_ip: target }` を記録します — 分岐が常に false であることを主張します。
- **取られる** `JumpIfFalse` は `GuardFalse { reg: src, fail_ip: next_ip }` を記録します。

opcode がトレースできない場合（複雑なメソッド呼び出し、文字列操作など）、記録は中止され `is_recording` は false に戻されます。

### トレースコンパイル

トレースされたループが `start_ip` に戻ると、完全な `Trace` が Cranelift の `JIT::compile()` 関数（`src/backend/jit.rs`）に渡されます。Cranelift は `TraceOp` シーケンスを次のシグネチャのネイティブ関数にコンパイルします：

```rust
unsafe extern "C" fn(
    locals_ptr: *mut Value,
    globals_ptr: *mut Value,
    consts_ptr:  *const Value,
) -> i32
```

戻り値は再開する次の IP です（0 = 通常継続、正 = サイドエグジット IP、負 = halt）。

コンパイルされた関数ポインタは `Trace::native_ptr`（`AtomicPtr<u8>`）に格納され、`Arc<Trace>` は `vm.traces`（グローバル共有）と `trace_cache`（Executor ごとの高速パス）の両方に挿入されます。

### トレース実行

ディスパッチループの各反復で、次の opcode をフェッチする前に：

```
if trace_cache[current_ip].is_some() {
    execute_trace(trace, ip, locals, &mut glbs)
    continue
}
```

コンパイル済みネイティブ関数が利用可能な場合（`native_ptr != null`）、`transmute` 経由で直接呼び出されます — ループ本体全体についてインタプリタを完全にバイパスします。JIT がまだコンパイルしていない場合、中間ステップとして解釈実行の `TraceOp` パスが使用されます。

### TraceOp バリアント

| バリアント | 説明 |
|---|---|
| `LoadConst`, `Move` | 定数値によるレジスタ移動 |
| `AddInt/SubInt/MulInt/DivInt/ModInt` | div/mod ゼロ用 fail-IP 付き整数算術 |
| `AddFloat/SubFloat/MulFloat/DivFloat/ModFloat` | 浮動小数点算術 |
| `CmpInt / CmpFloat` | `cc: u8` 条件コードによる比較 |
| `GuardInt / GuardFloat` | 型ガード — レジスタの型が誤っている場合トレースを終了 |
| `GuardTrue / GuardFalse` | 分岐ガード — 予期しない分岐方向でトレースを終了 |
| `CastIntToFloat` | int レジスタを float に拡張 |
| `IncLocal / IncVar` | 単一レジスタ／グローバルのインクリメント |
| `LoopNextInt` | 範囲ループ用の融合インクリメント + 条件付きジャンプ |
| `IncVarLoopNext / IncLocalLoopNext` | 配列および for-range ループ用の融合バリアント |
| `GetVar / SetVar` | グローバル変数アクセス |
| `And / Or / Not` | ブール論理 |
| `Jump` | 無条件ジャンプ（トレース内でループバック検出をトリガー） |

---

## 実行フロー

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

関数は `run_frame(func_id, params)` 経由で呼び出され、`chunk.max_locals` に合わせて事前サイズ設定された新しい `locals` ベクタを作成します。`current_spans` は呼び出された関数の span テーブルにスワップされ、return 時に復元されます。

---

## Fiber 実行モデル

Fiber は OS スレッドではなく、**協調的コルーチン**です。

```rust
pub struct FiberState {
    pub func_id:       usize,
    pub ip:            usize,
    pub locals:        Vec<Value>,    // moved out during resume, moved back after
    pub is_done:       bool,
    pub yielded_value: Option<Value>, // cached value for IsDone + Next pattern
}
```

### 再開シーケンス（`resume_fiber`）

1. `func_id`、`ip` を読み取り、`std::mem::take` 経由で `FiberState` から `locals` を**ムーブ** — クローンなし。
2. ムーブした locals で `fiber.ip` から `execute_bytecode` を実行。
3. `Yield` 時：`fiber_yielded = true` を設定。locals を `FiberState` に戻す。`fiber.ip` を更新。yield された値を return。
4. `Return` / バイトコード終了時：`fiber.is_done = true` を設定。最終値を return。

再開／一時停止では初期 `Vec` 作成を除きヒープ割り当ては発生しません — ムーブのみです。

### IsDone / Next パターン

`IsDone` は `FiberState::is_done` を確認し、`yielded_value` がキャッシュされているかを考慮します。`Next` はキャッシュされた `yielded_value` が存在すればそれを取得し、さもなければ `resume_fiber` を呼び出します。これにより `for x in fiber` ループが fiber を二重に進めることがないことを保証します。

### Fiber に対する For ループ（`ForIterType::Fiber`）

コンパイラは以下を発行します：
1. `MethodCall(IsDone)` → 終了へ `JumpIfTrue`
2. `MethodCall(Next)` → ループ変数に代入
3. ループ本体
4. ステップ 1 へ `Jump` で戻る
5. `break` 時：`MethodCall(Close, base=fiber_reg)` がジャンプ前に fiber を done としてマーク

---

## HTTP サーバー（`HttpServe`）

`HttpServe` は `tiny_http::Server` を起動し、N 個の OS スレッドを spawn します：

```rust
for _ in 0..workers {
    let server  = server.clone();    // Arc<tiny_http::Server>
    let vm      = vm_arc.clone();    // Arc<VM>
    let ctx     = self.ctx.clone();  // SharedContext (two Arc clones)
    let routes  = routes.clone();    // Arc<Vec<(String, usize)>>
    std::thread::spawn(move || { /* recv → match route → run handler fiber */ });
}
```

各ワーカーは独自の locals を持つ独自の `Executor` を実行します。グローバルは `Arc<RwLock<Vec<Value>>>` 経由で共有されます。

### リクエスト処理

各着信リクエストについて、ワーカーは：
1. `"METHOD /path"` キーをルートテーブルと照合（大文字小文字を区別しない）。
2. `{ method, url, body, ip, headers }` の JSON オブジェクトを `Value::Json` として構築。
3. `tiny_http::Request` を `Arc<Mutex<Option<tiny_http::Request>>>` に格納し、`http_req` 経由で新しい `Executor` に渡す。
4. そのワーカーの `Executor` 内でマッチしたハンドラ fiber を同期的に実行。
5. ハンドラが `net.respond(...)` を呼び出すと、VM は `HttpRespond` を実行しレスポンスを送信、ハンドラを終了するために `OpResult::Yield` を return。
6. ハンドラが `net.respond` を呼び出さずに終了した場合、ワーカーは `500` フォールバックレスポンスを送信。

### グレースフルシャットダウン

`SHUTDOWN` は `vm.rs` 内の `pub static AtomicBool` です。`main.rs` の Ctrl+C ハンドラが `true` に設定します。ワーカーは `recv_timeout(100ms)` サイクルごとに確認します。メインスレッドは `sleep(500ms)` ループでブロックし、`SHUTDOWN` もポーリングします。設定されるとすべてのループが終了し、プロセスはクリーンに終了します。

---

## コンパイラ（`src/backend/mod.rs`）

### レジスタ割り当て

`FunctionCompiler` は `next_local: usize` で次に利用可能なレジスタを追跡します：

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

ローカル（名前付き変数）は `define_local(id, slot)` 経由でスロットに割り当てられ、`scopes: Vec<HashMap<StringId, usize>>` に格納されます。一時変数は `push_reg()`/`pop_reg()` を使用 — 式の結果が消費されると再利用されます。

`max_locals_used` は記録され、関数呼び出し時に `FunctionChunk::max_locals` が正確なサイズの locals ベクタを事前割り当てできるようにします。

### 定数の重複排除

`CompileContext::add_constant` は `string_constants: HashMap<String, usize>` 経由で文字列定数を重複排除します。重複する文字列定数（例：メソッド名引数として多数出現する `"insert"`）は同じ定数テーブルスロットを再利用します。

### 2 パスコンパイル

**パス 1** — `register_globals_recursive`：
- すべてのグローバル変数と fiber-decl インスタンスにスロットインデックスを割り当て
- すべての関数／fiber に関数インデックスを割り当て
- `functions: Vec<FunctionChunk>` に空の `FunctionChunk` スロットを事前割り当て

**パス 2** — `compile_stmt` / `compile_expr`：
- `emit(op, span)` 経由で span とペアになったバイトコードを発行
- `main` のトップレベル文は `GetVar`/`SetVar`（グローバル）を使用；入れ子文はレジスタを使用

### `FunctionChunk`

```rust
pub struct FunctionChunk {
    pub bytecode:   Arc<Vec<OpCode>>,
    pub spans:      Arc<Vec<Span>>,    // spans[i] corresponds to bytecode[i]
    pub is_fiber:   bool,
    pub max_locals: usize,
}
```

バイトコードと spans は `Arc` でラップされ、HTTP ワーカースレッド間でコピーなしに共有できます。

---

## ランタイムエラー報告

VM 内のすべてのランタイムエラーは `self.current_span_info(ip)` を付加し、`current_spans[ip - 1]` を参照して `" [line: X, col: Y]"` を返します。例：

```
ERROR: R303: Array index out of bounds: 5 [line: 14, col: 7]
```

`current_spans` は各 `run_frame` 呼び出しで正しい `Arc<Vec<Span>>` にスワップされ、終了時に復元されます。

---

## メモリモデル

- **ガベージコレクタなし**。`Arc` による参照カウント（NaN-boxed ポインタ値用の手動 `inc_ref`/`dec_ref` 付き）。
- **スカラー値**（Int、Float、Bool、Date、Function index）：`u64` 内に完全に格納 — ヒープ割り当てゼロ。
- **コレクション値**：`Arc<RwLock<T>>` が共有所有権を提供。コレクション `Value` のクローンは `Arc` カウンタのみをインクリメント。
- **変更**：`.insert()`、`.update()`、`.delete()` は write lock を取得。同一コレクションへのすべてのハンドルが変更を見ます。
- **読み取り専用メソッド**：`.size()`、`.get()`、`.contains()` は read lock を取得 — 複数の同時読み取りが許可されます。

---

## セキュリティ制御

### ネットワーク SSRF 保護（`is_safe_url`）

ブロック対象：
- `file://` URL
- `169.254.x.x`（リンクローカル / AWS メタデータエンドポイント）
- プライベート範囲：`10.x`、`192.168.x`、`172.16–31.x`（localhost でない場合）

`HttpCall` と `HttpRequest` の両方に適用されます。

### HTTP ボディ制限

`HttpServe` では、`into_string()` 後にレスポンスボディをチェックします。**10 MB** を超える場合、実際のボディの代わりに `413` エラー JSON を返します。

### CORS ヘッダー

すべての `HttpServe` レスポンスには自動的に以下が含まれます：
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

`OPTIONS` プリフライトリクエストはハンドラを呼び出さずに `204` レスポンスを受け取ります。
