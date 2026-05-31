# XCX Expander — ドキュメント

> **File:** `src/parser/expander.rs`  
> パース**後**、意味解析**前**に実行される。

---

## 目次

1. [概要](#概要)
2. [責務](#責務)
3. [include の解決](#include-の解決)
4. [エイリアスプレフィックス](#エイリアスプレフィックス)
5. [保護名](#保護名)
6. [include パス検索順序](#include-パス検索順序)

---

## 概要

Expander はパースフェーズの後、意味解析の前に実行される独立した AST 書き換えパスである。`include` および `include ... as alias` ディレクティブを処理する。

```rust
pub struct Expander<'a> {
    interner:       &'a mut Interner,
    visiting_files: HashSet<PathBuf>,   // for circular dependency detection
    included_files: HashSet<PathBuf>,   // deduplication (each file once)
    aliases:        HashMap<StringId, String>,
    include_paths:  Vec<PathBuf>,       // additional search paths
}
```

---

## 責務

### 1. include の解決
`include "file.xcx";` はそのファイルのインライン AST に置き換えられる。

- 循環依存は `visiting_files: HashSet<PathBuf>` で検出される
- ファイルは `included_files: HashSet<PathBuf>` で重複排除される (エイリアス付きでない限り各ファイルは一度だけ)

### 2. エイリアスプレフィックス
`include "math.xcx" as math;` により、そのファイルのすべてのトップレベル名に `math.name` プレフィックスが付与される。呼び出しサイト (`math.sin(x)`) は `expand_expr_inplace` により `MethodCall` から `FunctionCall { name: "math.sin" }` へ書き換えられる。

関数 `prefix_program()` / `prefix_stmt_impl()` / `prefix_expr_impl()` がサブ AST 全体を走査し、すべてのトップレベルシンボル参照をリネームする。

### 3. Fiber 名プレフィックス
`FiberDecl::fiber_name` 参照もプレフィックスされ、リネーム後の fiber インスタンス化が正しく解決される。

### 4. YieldFrom プレフィックス
`StmtKind::YieldFrom` 式を走査し、`yield from` 内の fiber コンストラクタ呼び出しもリネームされる。

---

## include の解決

```xcx
include "utils.xcx";           --- Simple include (deduplicated)
include "math.xcx" as math;    --- Aliased include
```

エイリアス付き include の後:
- `math.xcx` のすべてのトップレベルシンボルに `math.` プレフィックスが付く
- `math.sin(x)` への呼び出しは `MethodCall` から `FunctionCall { name: "math.sin" }` へ書き換えられる

例:

```xcx
--- math.xcx defines:
func sin(f: x -> f) { ... }
func cos(f: x -> f) { ... }

--- After include "math.xcx" as math:
--- sin → math.sin
--- cos → math.cos
--- Call math.sin(3.14) → FunctionCall("math.sin", [3.14])
```

---

## エイリアスプレフィックス

`prefix_stmt_impl` / `prefix_expr_impl` アルゴリズム:

1. プログラムからすべてのトップレベル名を収集 (`top_level_names: HashSet<StringId>`)
2. 各文/式について: 識別子が `top_level_names` に属する場合、`prefix.name` に置き換える
3. 特に以下を処理:
   - `FiberDecl::fiber_name` — リネーム後のインスタンス化を可能にする
   - `StmtKind::YieldFrom` — `yield from` 内の fiber 式
   - エイリアス付きオブジェクトへの `MethodCall` → `FunctionCall` へ書き換え

---

## 保護名

以下の名前は**決して**プレフィックスされない (保護された組み込み):

```
json    date    store   halt    terminal
net     env     crypto  EMPTY   math
random  i       f       s       b
from    main
```

---

## include パス検索順序

1. 現在のファイルのディレクトリからの相対パス
2. `lib/` ディレクトリ (CWD からの相対、次に実行ファイルのパスから上方向にたどる)

追加パスは以下で追加可能:

```rust
expander.add_include_path(path);
```

`main.rs` では、CWD からの `lib/` ディレクトリが自動的に追加される:

```rust
if let Ok(cwd) = std::env::current_dir() {
    let lib_path = cwd.join("lib");
    if lib_path.exists() {
        expander.add_include_path(lib_path);
    }
}
```

---

## エラー検出

| エラー | 説明 |
|---|---|
| `Circular dependency` | include ループを検出 (`visiting_files`) |
| `File not found` | `include` ファイルがいずれの検索パスにも存在しない |
| `Could not read file` | ファイル読み取り中の I/O エラー |
