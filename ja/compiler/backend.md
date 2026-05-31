# XCX バックエンド — コンパイラと VM

> **ファイル：** `src/backend/mod.rs`、`src/backend/vm.rs`

---

## 目次

1. [バイトコードコンパイラ](#バイトコードコンパイラ)
2. [値表現 — NaN-boxing](#値表現--nan-boxing)
3. [命令セット（OpCodes）](#命令セットopcodes)
4. [VM アーキテクチャ](#vm-アーキテクチャ)
5. [Fiber 実行モデル](#fiber-実行モデル)
6. [HTTP サーバー](#http-サーバー)
7. [メモリ管理](#メモリ管理)
8. [ループ最適化](#ループ最適化)

---

## バイトコードコンパイラ

**ファイル：** `src/backend/mod.rs`

### レジスタ割り当て

`FunctionCompiler` は `next_local: usize` 経由で次に利用可能なレジスタを追跡します：

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

ローカル変数（名前付き）は `define_local(id, slot)` 経由でスロットに割り当てられ、`scopes: Vec<HashMap<StringId, usize>>` に格納されます。一時変数は `push_reg()`/`pop_reg()` を使用 — 式の結果が消費されると再利用されます。

`max_locals_used` は記録され、関数呼び出し時に `FunctionChunk::max_locals` が正確なサイズの locals ベクタを事前割り当てできるようにします。

### 定数の重複排除

`CompileContext::add_constant` は `string_constants: HashMap<String, usize>` を使用して文字列定数を重複排除します。重複する文字列定数（例：メソッド名引数として複数回出現する `"insert"`）は同じ定数テーブルスロットを再利用します。

### 2 パスコンパイル

**パス 1** — `register_globals_recursive`：
- すべてのグローバル変数と fiber-decl インスタンスにスロットインデックスを割り当て
- すべての関数／fiber に関数インデックスを割り当て
- `functions: Vec<FunctionChunk>` に空の `FunctionChunk` スロットを事前割り当て

**パス 2** — `compile_stmt` / `compile_expr`：
- `emit(op, span)` 経由で span とペアになったバイトコードを発行
- `main` のトップレベル命令は `GetVar`/`SetVar`（グローバル）を使用；入れ子命令はレジスタを使用

### FunctionChunk

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

## 値表現 — NaN-boxing

各値は単一の `Value(u64)` — 64 ビットワードです。XCX は **NaN-boxing** を使用します：IEEE 754 quiet NaN のビットパターンを型タグプレフィックスとして再利用します。

```
Bit layout: [63..52: exponent/QNAN] [51..48: type tag] [47..0: payload]

Float : stored directly as f64 bits — DOES NOT have the QNAN_BASE prefix set
Int   : QNAN_BASE | TAG_INT  | (i48 value & 0x0000_FFFF_FFFF_FFFF)
Bool  : QNAN_BASE | TAG_BOOL | (0 or 1)
Date  : QNAN_BASE | TAG_DATE | (i48 ms timestamp)
Ptr   : QNAN_BASE | TAG_XXX  | (pointer & 0x0000_FFFF_FFFF_FFFF)
```

### タグ定数

| 定数 | 値 | 型 |
|---|---|---|
| `QNAN_BASE` | `0x7FF0_0000_0000_0000` | base NaN marker |
| `TAG_INT` | `0x0001_0000_0000_0000` | 48-bit signed integer |
| `TAG_BOOL` | `0x0002_0000_0000_0000` | boolean (payload 0/1) |
| `TAG_DATE` | `0x0003_0000_0000_0000` | 48-bit timestamp (ms) |
| `TAG_STR` | `0x0004_0000_0000_0000` | `Arc<Vec<u8>>` pointer |
| `TAG_ARR` | `0x0005_0000_0000_0000` | `Arc<RwLock<Vec<Value>>>` pointer |
| `TAG_SET` | `0x0006_0000_0000_0000` | `Arc<RwLock<SetData>>` pointer |
| `TAG_MAP` | `0x0007_0000_0000_0000` | `Arc<RwLock<Vec<(Value,Value)>>>` pointer |
| `TAG_TBL` | `0x0008_0000_0000_0000` | `Arc<RwLock<TableData>>` pointer |
| `TAG_FUNC` | `0x0009_0000_0000_0000` | function index (u32) |
| `TAG_ROW` | `0x000A_0000_0000_0000` | `Arc<RowRef>` pointer |
| `TAG_JSON` | `0x000B_0000_0000_0000` | `Arc<RwLock<serde_json::Value>>` pointer |
| `TAG_FIB` | `0x000C_0000_0000_0000` | `Arc<RwLock<FiberState>>` pointer |
| `TAG_DB` | `0x000D_0000_0000_0000` | `Arc<DatabaseData>` pointer |

ポインタペイロードは下位 48 ビットのみを使用します — ユーザースペースポインタが 48 ビットに収まる x86-64 および AArch64 プラットフォームすべてで有効です。

### 参照カウント

ポインタタグ付き値は `Arc` 参照カウントを保持します。VM は代入、return、コレクション変更のたびに `inc_ref()` / `dec_ref()` で手動管理し、ガベージコレクタなしでヒープ割り当てオブジェクトが参照されなくなったときに解放されるようにします。

---

## 命令セット（OpCodes）

すべての opcode はレジスタベースです：オペランドスタックではなく、名前付き `u8` レジスタスロットを参照します。

### レジスタ／変数の移動

| OpCode | 説明 |
|---|---|
| `LoadConst { dst, idx }` | `constants[idx]` をレジスタ `dst` にロード |
| `Move { dst, src }` | レジスタ `src` を `dst` にコピー |
| `GetVar { dst, idx }` | `globals[idx]` を `dst` にロード（globals を read-lock） |
| `SetVar { idx, src }` | `src` を `globals[idx]` に格納（globals を write-lock） |

### 算術

すべての算術演算は 3 レジスタ：`dst = src1 OP src2`。ランタイム型ディスパッチにより、整数、浮動小数点、文字列連結、日付算術、セット演算の各パスが選択されます。

`Add`, `Sub`, `Mul`, `Div`, `Mod`, `Pow`, `IntConcat` (`++`)

### 比較（Bool 結果）

`Equal`, `NotEqual`, `Greater`, `Less`, `GreaterEqual`, `LessEqual`

### 論理

`And { dst, src1, src2 }`, `Or { dst, src1, src2 }`, `Not { dst, src }`, `Has { dst, src1, src2 }`

### 制御フロー

| OpCode | 説明 |
|---|---|
| `Jump { target }` | 無条件ジャンプ；後方ジャンプ時に `hot_counts[target]` をインクリメント |
| `JumpIfFalse { src, target }` | `src` が `Bool(false)` のときジャンプ |
| `JumpIfTrue { src, target }` | `src` が `Bool(true)` のときジャンプ |
| `Call { dst, func_idx, base, arg_count }` | 関数呼び出し；結果 → `dst` |
| `Return { src }` | 現在のフレームから `src` の値を return |
| `ReturnVoid` | 値なしで return |
| `Halt` | 実行停止 |

### コレクション

| OpCode | 説明 |
|---|---|
| `ArrayInit { dst, base, count }` | `base` から `count` 個のレジスタを収集 → `dst` に新しい配列 |
| `SetInit { dst, base, count }` | `count` 個のレジスタを収集 → `dst` に新しい set |
| `SetRange { dst, start, end, step, has_step }` | レジスタ値から範囲 set を構築 |
| `MapInit { dst, base, count }` | `count` 個のキー値ペアを収集 → `dst` に新しい map |
| `TableInit { dst, skeleton_idx, base, row_count }` | 列スキーマ定数 + 行値から table を構築 |

### セット演算

`SetUnion`, `SetIntersection`, `SetDifference`, `SetSymDifference` — すべて 3 レジスタ、両オペランドは `TAG_SET` である必要があります。

### メソッドディスパッチ

| OpCode | 説明 |
|---|---|
| `MethodCall { dst, kind, base, arg_count }` | `MethodKind` enum による組み込みメソッドのディスパッチ — ランタイムでの文字列ルックアップなし |
| `MethodCallCustom { dst, method_name_idx, base, arg_count }` | 定数テーブルの文字列による動的メソッド（JSON フィールド、エイリアス）のディスパッチ |
| `MethodCallNamed { dst, kind, base, arg_count, names_idx }` | 名前付き引数付きメソッド呼び出し |

`base` はレシーバレジスタを指します；引数は `locals[base+1..base+1+arg_count]` にあります。`MethodKind` は約 50 の組み込みメソッド（`Push`, `Pop`, `Get`, `Insert`, `Update`, `Delete`, `Where`, `Join`, `Sort`, `Format`, `Next`, `IsDone`, `Close` など）をカバーする `#[derive(Copy)]` enum です。

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
| `Input { dst, ty }` | stdin から 1 行読み取り、型キャストして → `dst` |
| `Wait { src }` | `src` ミリ秒スリープ |
| `HaltAlert { src }` | 警告メッセージを出力、実行を継続 |
| `HaltError { src }` | エラー + span 情報を出力、フレームを停止、error_count をインクリメント |
| `HaltFatal { src }` | fatal + span 情報を出力、フレームを停止、error_count をインクリメント |
| `TerminalExit` | `std::process::exit(0)` |
| `TerminalClear` | ANSI エスケープでターミナルをクリア |
| `TerminalRaw / TerminalNormal` | raw ターミナルモードの有効化／無効化 |
| `TerminalCursor { on }` | カーソルの表示／非表示 |
| `TerminalMove { x_src, y_src }` | カーソルを移動 |
| `TerminalWrite { src }` | 改行なしで書き込み |
| `InputKey { dst }` | キーを読み取り（非ブロッキング） |
| `InputKeyWait { dst }` | キーを読み取り（ブロッキング） |
| `InputReady { dst }` | 入力が利用可能か確認 |
| `EnvGet { dst, src }` | 環境変数を読み取り |
| `EnvArgs { dst }` | CLI 引数を取得 |

### HTTP

| OpCode | 説明 |
|---|---|
| `HttpCall { dst, method_idx, url_src, body_src }` | `ureq` によるシンプルな HTTP 呼び出し、JSON 結果 → `dst` |
| `HttpRequest { dst, arg_src }` | 設定 map からの完全な HTTP 呼び出し（method、url、headers、body、timeout）→ `dst` |
| `HttpRespond { status_src, body_src, headers_src }` | fiber ハンドラ内から HTTP レスポンスを送信 |
| `HttpServe { func_idx, port_src, host_src, workers_src, routes_src }` | `tiny_http` サーバーを起動、ワーカースレッドを spawn |

### ストレージ

`StoreWrite`, `StoreRead`, `StoreAppend`, `StoreExists`, `StoreDelete`, `StoreList`, `StoreIsDir`, `StoreSize`, `StoreMkdir`, `StoreGlob`, `StoreZip`, `StoreUnzip`

### JSON

`JsonParse { dst, src }`, `JsonBind { idx, json_src, path_src }`, `JsonBindLocal { dst, json_src, path_src }`, `JsonInject { table_idx, json_src, mapping_src }`, `JsonInjectLocal { table_reg, json_src, mapping_src }`

### 型キャスト

`CastInt { dst, src }`, `CastFloat { dst, src }`, `CastString { dst, src }`, `CastBool { dst, src }`

### 暗号と日付

`CryptoHash`, `CryptoVerify`, `CryptoToken`, `DateNow`

### データベース

`DatabaseInit { dst, engine_src, path_src, tables_base_reg, table_count }`

---

## VM アーキテクチャ

**ファイル：** `src/backend/vm.rs`

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

`SharedContext` は安価にクローンできます（2 つの `Arc` カウンタのインクリメント）し、各ワーカースレッドに独立して渡されます。ディープコピーは発生しません。

### 実行フロー

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

## Fiber 実行モデル

Fiber は OS スレッドではなく、**協調的コルーチン**です。

```rust
pub struct FiberState {
    pub func_id:       usize,
    pub ip:            usize,
    pub locals:        Vec<Value>,    // moved during resume, returned after
    pub is_done:       bool,
    pub yielded_value: Option<Value>, // cache for IsDone + Next pattern
}
```

### 再開シーケンス（`resume_fiber`）

1. `func_id`、`ip` を読み取り、`std::mem::take` 経由で `FiberState` から `locals` を**ムーブ** — クローンなし
2. ムーブした locals で `fiber.ip` から `execute_bytecode` を開始
3. `Yield` 時：`fiber_yielded = true` を設定。locals を `FiberState` に戻す。`fiber.ip` を更新。yield された値を return
4. `Return` / バイトコード終了時：`fiber.is_done = true` を設定。最終値を return

再開／一時停止では初期 `Vec` 作成を除きヒープ割り当ては発生しません — ムーブのみです。

### IsDone / Next パターン

`IsDone` は `FiberState::is_done` を確認し、`yielded_value` がキャッシュされているかを考慮します。`Next` はキャッシュされた `yielded_value` が存在すればそれを取得し、さもなければ `resume_fiber` を呼び出します。これにより `for x in fiber` ループが fiber を二重に進めることがないことを保証します。

### Fiber に対する For ループ（`ForIterType::Fiber`）

コンパイラは以下を発行します：
1. `MethodCall(IsDone)` → 終了へ `JumpIfTrue`
2. `MethodCall(Next)` → ループ変数に代入
3. ループ本体
4. ステップ 1 へ `Jump` で戻る
5. `break` 時：`MethodCall(Close, base = fiber_reg)` がジャンプ前に fiber を done としてマーク

---

## HTTP サーバー

`HttpServe` は `tiny_http` サーバーを起動し、N 個の OS スレッドを spawn します：

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
1. `"METHOD /path"` キーをルートテーブルと照合（大文字小文字を区別しない）
2. `{ method, url, body, ip, headers }` の JSON オブジェクトを `Value::Json` として構築
3. `tiny_http::Request` を `Arc<Mutex<Option<tiny_http::Request>>>` に格納し、新しい `Executor` に渡す
4. そのワーカーの `Executor` 内でマッチしたハンドラ fiber を同期的に実行
5. ハンドラが `net.respond(...)` を呼び出すと、VM は `HttpRespond` を実行しレスポンスを送信
6. ハンドラが `net.respond` を呼び出さずに終了した場合、ワーカーは `500` レスポンスを送信

### グレースフルシャットダウン

`SHUTDOWN` は `vm.rs` 内の `pub static AtomicBool` です。`main.rs` の Ctrl+C ハンドラが `true` に設定します。ワーカースレッドは `recv_timeout(100ms)` ごとに確認します。メインスレッドは `sleep(500ms)` ループでブロックし、`SHUTDOWN` もポーリングします。設定されるとすべてのループが終了し、プロセスはクリーンに終了します。

---

## メモリ管理

- **ガベージコレクタなし**。`Arc` による参照カウント（NaN-boxed ポインタ値用の手動 `inc_ref`/`dec_ref` 付き）。
- **スカラー値**（Int、Float、Bool、Date、Function index）：`u64` 内に完全に格納 — ヒープ割り当てゼロ。
- **コレクション値**：`Arc<RwLock<T>>` が共有所有権を保証。コレクション `Value` のクローンは `Arc` カウンタのみをインクリメント。
- **変更**：`.insert()`、`.update()`、`.delete()` は write lock を取得。同一コレクションへのすべてのハンドルが変更を見ます。
- **読み取り専用メソッド**：`.size()`、`.get()`、`.contains()` は read lock を取得 — 複数の同時読み取りが許可されます。

---

## ループ最適化

これらの opcode はコンパイラによって発行され、一般的なループカウンタパターンを単一命令に融合し、ディスパッチオーバーヘッドを削減して JIT トレーシングを改善します：

| OpCode | 説明 |
|---|---|
| `IncLocal { reg }` | レジスタ `reg` の整数を 1 インクリメント |
| `IncVar { idx }` | `idx` のグローバルを 1 インクリメント |
| `LoopNext { reg, limit_reg, target }` | `reg` をインクリメント、`reg <= limit_reg` なら `target` へジャンプ |
| `IncLocalLoopNext { inc_reg, reg, limit_reg, target }` | `inc_reg`（別カウンタ、例：配列インデックス）をインクリメント、`reg`（ループ変数）をインクリメント、条件付きジャンプ |
| `IncVarLoopNext { g_idx, reg, limit_reg, target }` | `IncLocalLoopNext` と同様だが `g_idx` はグローバルカウンタ |
| `ArrayLoopNext { idx_reg, size_reg, target }` | 融合配列インデックス反復 |

### 最適化変換

コンパイラはループステップ終了前に最後に発行された命令を確認します。`IncVar` または `IncLocal` の場合、それぞれ `IncVarLoopNext` または `IncLocalLoopNext` に置き換え、インクリメントを条件付きループテストと融合します：

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
