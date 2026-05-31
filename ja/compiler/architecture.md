# XCX コンパイラアーキテクチャ — v3.1

XCX コンパイラは Rust で実装され、多段パイプラインアーキテクチャに従う。

## コンパイルパイプライン

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

> **Note**: Expander は `src/parser/` モジュールの一部だが、パース後の独立したフェーズとして、意味解析の前に実行される。JIT はホットループ向けの任意の高速化レイヤーで、自動的に有効化される — トレースがまだコンパイルされていない場合でもバイトコード実行は中断されない。

## プロジェクト構成

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

## 診断システム

コンパイラは `Reporter` 構造体を用いて文脈付きエラーメッセージを生成する。各エラーには以下が含まれる:
- **Level**: ERROR または HALT バリアント
- **Location**: 行番号と列番号
- **Visual highlight**: `~~~` 下線付きの該当ソース行

意味解析エラー (`TypeError`) は検査フェーズ中に `Vec` へ収集され、バイトコード生成開始前に一括報告される。エラーが存在する場合、コンパイルはその時点で停止する。

VM によって生成されるランタイムエラーには、各 `FunctionChunk` のバイトコードとともに保存される `spans` テーブルから導出されたソース位置情報 (行と列) が含まれる。

## 主要な設計判断

### NaN-Boxed Values
すべての値は NaN-boxing により単一の `u64` として表現される。IEEE 754 quiet NaN の上位ビットを型タグとして使用し、下位 48 ビットにペイロード (整数、ブール、ポインタ) を格納する。浮動小数点数はそのまま格納される。つまり各 `Value` は正確に 8 バイト — スカラーにヒープ割り当てなし、enum タグのオーバーヘッドなし、プリミティブにポインタ間接参照なし。完全なタグレイアウトは `src/backend/vm.rs` を参照。

### Register-Based VM
VM は**レジスタベース**であり、スタックベースではない。各関数フレームはスロット番号でインデックスされるフラットな `Vec<Value>` を所有する。OpCode はソース/宛先レジスタを直接参照する (`dst`, `src1`, `src2` フィールド)。コンパイラの `FunctionCompiler` は一時レジスタ用に単純なバンプアロケータ (`push_reg()` / `pop_reg()`) を維持する。ローカルと一時変数は同じフラット配列に存在 — 別のオペランドスタックはない。

### Tracing JIT (Cranelift)
ホットな後方ジャンプは、フレームごとの `hot_counts: Vec<usize>` カウンタで検出される。ループのバックエッジが閾値 (50 回) に達すると、トレース記録フェーズが開始される。`Executor` は `TraceOp` バリアント — インタプリタ op の型付き・特殊化版 — を記録し、トレースが閉じる (開始 IP へのループバック) まで続ける。完成した `Trace` は Cranelift 経由でネイティブコードへコンパイルされ、`trace_cache` にキャッシュされる。そのループの以降の反復はインタプリタを完全にバイパスする。トレース内のガード (`GuardInt`, `GuardFloat`, `GuardTrue`, `GuardFalse`) が型特殊化を処理し、ガード失敗時は正しい IP へインタプリタへのサイドエグジットが発生する。

### String Interning
すべての識別子と文字列リテラルは `Interner` 経由で `StringId` (u32) へインターンされる。パイプライン全体での繰り返しの文字列割り当てとヒープ比較を回避する。

### Constant Deduplication
コンパイラは `string_constants: HashMap<String, usize>` を維持し、各一意の文字列値が定数テーブルに一度だけ格納されることを保証する。コンパイル中に頻繁に出力される組み込みメソッド名に特に効果的。

### Method Dispatch via Enum
組み込みメソッド呼び出しは `OpCode::MethodCall { kind: MethodKind, base, arg_count }` としてコンパイルされ、`MethodKind` は約 50 の組み込みメソッドをカバーする `Copy` enum である。未知または動的なメソッド名 (例: JSON フィールドアクセス) は別の `OpCode::MethodCallCustom { method_name_idx, base, arg_count }` パスを使用する。VM ディスパッチループでの文字列比較を排除する。

### Two-Pass Compilation
バックエンドは最初のパス (`register_globals_recursive`) で、バイトコードを出力する前にすべてのグローバル、関数、fiber にインデックスを割り当てる。

### Span-Annotated Bytecode
出力される各 opcode はソース AST の `Span` とペアになる。`FunctionChunk` は `bytecode: Arc<Vec<OpCode>>` と並んで `spans: Arc<Vec<Span>>` を保存する。VM はこれを用いて行精度のランタイムエラーメッセージを生成する。

### Fiber-as-Coroutine
Fiber は OS スレッドではない。各 `FiberState` は独自の `ip` と `locals` を保持し、VM によって協調的に再開される。yield 時に locals は `FiberState` へ移動し、resume 時に再び移動される。初期 `Vec` 作成を超える割り当ては発生しない。

### Thread-Safe Value Representation
すべての共有可変コレクションは `Arc<RwLock<T>>` (`parking_lot` 経由) を使用する。`FunctionChunk` のバイトコードと spans は `Arc<Vec<...>>` でラップされ、コピーなしで HTTP ワーカースレッド間で共有可能。`Value` は `Copy` + `Send` + `Sync`。

### Graceful HTTP Shutdown
グローバル `AtomicBool` (`SHUTDOWN`) は Ctrl+C シグナルハンドラ (`ctrlc` crate 経由) によって設定される。すべての HTTP ワーカースレッドとメインディスパッチループはこのフラグをポーリングし、プロセス終了前にクリーンに終了する。

### Scope Chain Symbol Table
`SymbolTable` は深いクローンの代わりに親ポインタ連結チェーンを使用する。関数スコープに入ると、親への参照を持つ新しい `SymbolTable` が作成される — ルックアップはチェーンを上方向にたどる。関数スコープのエントリは O(n) ではなく O(1)。

### Byte-Level Scanner
Lexer は `&[u8]` (元のソースバイトへの参照) 上で動作し、`Vec<char>` を割り当てない。Unicode 処理は必要な箇所でのみ行う。コメント検出は `slice.starts_with(b"---")` を使用する。
