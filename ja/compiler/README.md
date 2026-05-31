# XCX コンパイラ — ドキュメント v3.1

> Rust で実装された XCX 言語コンパイラ。多段パイプライン: lexer → parser → semantic analysis → bytecode compiler → virtual machine → JIT (Cranelift)。

---

## 目次

1. [アーキテクチャ概要](#アーキテクチャ概要)
2. [クイックスタート](#クイックスタート)
3. [プロジェクト構成](#プロジェクト構成)
4. [コンパイルパイプライン](#コンパイルパイプライン)
5. [モジュール](#モジュール)
   - [Lexer (Scanner)](#lexer-scanner)
   - [Parser (Pratt)](#parser-pratt)
   - [Expander](#expander)
   - [Semantic Analysis (Sema)](#semantic-analysis-sema)
   - [Compiler (Backend)](#compiler-backend)
   - [Virtual Machine (VM)](#virtual-machine-vm)
   - [JIT (Cranelift)](#jit-cranelift)
6. [主要な設計判断](#主要な設計判断)
7. [診断システム](#診断システム)
8. [セキュリティ](#セキュリティ)

---

## アーキテクチャ概要

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

## クイックスタート

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

**REPL 内:**

```
xcx> !help     # display help
xcx> !clear    # clear screen
xcx> !exit     # exit
```

---

## プロジェクト構成

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

## コンパイルパイプライン

### 1. Lexer
ソースバイト列をトークンに変換する。`&[u8]` 上で動作 — `Vec<char>` の割り当ては行わない。

### 2. Parser
Pratt Parser (Top-Down Operator Precedence) が AST を構築する。1 トークン先読み。`synchronize()` によるエラー回復。

### 3. Expander
パース**後**、意味解析**前**に実行される。`include` ディレクティブを解決し、エイリアス名にプレフィックスを付ける。

### 4. Semantic Analysis
型を検査し、未定義変数を検出し、fiber/loop コンテキストを検証する。バイトコード生成前にすべてのエラーを収集する。

### 5. Compiler
2 パス。Pass 1: グローバル/関数を登録。Pass 2: Span アノテーション付きでバイトコードを出力。

### 6. VM
レジスタベースの仮想マシン。各フレームは `u8` スロット番号でインデックスされるフラットな `Vec<Value>` を持つ。8 バイト値 (NaN-boxing)。

### 7. JIT
Cranelift によりホットループをネイティブコードへ自動コンパイル。開発者からは透過的。

---

## モジュール

詳細は個別ファイルを参照:

- [`docs/lexer.md`](docs/lexer.md) — Lexer / Scanner
- [`docs/parser.md`](docs/parser.md) — Pratt Parser and AST
- [`docs/expander.md`](docs/expander.md) — Expander
- [`docs/sema.md`](docs/sema.md) — Semantic Analysis
- [`docs/backend.md`](docs/backend.md) — Compiler and VM
- [`docs/jit.md`](docs/jit.md) — Cranelift JIT
- [`docs/language.md`](docs/language.md) — XCX Language Reference

---

## 主要な設計判断

### NaN-Boxing による値表現
各値は単一の `u64` (8 バイト)。IEEE 754 quiet NaN の上位ビットを型タグとして使用し、下位 48 ビットにペイロードを格納する。スカラーにヒープ割り当てなし、enum のタグオーバーヘッドなし、プリミティブにポインタ間接参照なし。

### レジスタベース VM
オペランドスタックの代わりに、フレームごとのフラットな `Vec<Value>` 配列。OpCode はソース/宛先レジスタを直接参照する。コンパイラでは単純なバンプポインタによるレジスタ割り当て。

### トレーシング JIT (Cranelift)
後方ジャンプ (ループエッジ) は IP ごとにカウントされる。閾値 (50 回) に達するとトレース記録が開始される。完成したトレースは Cranelift によりネイティブコードへコンパイルされる。型ガード (`GuardInt`, `GuardFloat`) が型特殊化を処理し、ガード失敗時はインタプリタへフォールバックする。

### 文字列インターン
すべての識別子と文字列リテラルは `Interner` により `StringId (u32)` へインターンされる。パイプライン全体での繰り返しの文字列割り当てとヒープ比較を排除する。

### 2 パスコンパイル
Pass 1 (`register_globals_recursive`) は、バイトコードを出力する前にすべてのグローバルと関数にインデックスを割り当てる — 相互再帰と宣言前呼び出しを可能にする。

---

## 診断システム

`src/diagnostic/report.rs` の `Reporter` が文脈付きエラーメッセージを生成する:

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

各エラーには以下が含まれる:
- **Level**: ERROR または HALT
- **Location**: 行番号と列番号
- **Visual highlighting**: `~~~` 下線付きの該当ソース行

意味解析エラー (`TypeError`) は検査フェーズ中に `Vec` へ収集され、バイトコード生成前に一括報告される。

---

## セキュリティ

### ネットワークにおける SSRF 保護
ブロック対象:
- `file://` URL
- `169.254.x.x` (link-local / AWS metadata endpoint)
- プライベート範囲: `10.x`, `192.168.x`, `172.16–31.x` (localhost を除く)

`HttpCall` と `HttpRequest` の両方に適用される。

### HTTP コンテンツサイズ制限
`HttpServe` では、`into_string()` 後にレスポンス内容を検査する。**10 MB** を超える場合、実際の内容の代わりに 413 JSON エラーを返す。

### CORS ヘッダー
すべての `HttpServe` レスポンスに自動的に以下が含まれる:
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

プリフライト `OPTIONS` リクエストはハンドラを呼ばず `204` レスポンスを返す。

### 安全なファイルパス
`store.*` 操作はパスを検証 — `..`、絶対パス、Windows ドライブ文字をブロックする。
