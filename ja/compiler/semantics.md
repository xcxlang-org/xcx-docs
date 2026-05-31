# XCX 意味解析（Sema）— v3.1

Sema フェーズは、バイトコード生成の前に AST の論理的正しさと型の一貫性を検証します。2 つのコンポーネントで構成されます：**シンボルテーブル**と**型チェッカー**です。

## 文字列インターナー（`src/sema/interner.rs`）

`Interner` は `&str → StringId(u32)` をマッピングします。コンパイラ内のすべての文字列 ID の唯一の情報源です。字句解析／構文解析時に作成され、参照渡しでチェッカーとコンパイラに渡されます。

```
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

## シンボルテーブル（`src/sema/symbol_table.rs`）

`SymbolTable` は、**親ポインタで連結されたチェーン**を用いて、入れ子スコープ間の変数バインディングを管理します。

### 構造

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // reference to enclosing scope
    scopes: Vec<HashMap<String, Type>>,   // stack of scope frames (local to this table)
    consts: Vec<HashSet<String>>,         // which names are const, per scope frame
}
```

### 子スコープの作成

関数または fiber 本体に入る際、囲むテーブルをクローンするのではなく、その参照を持つ新しい `SymbolTable` が作成されます：

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

これは O(1) です — 新しいフレーム用に空の `HashMap` が 1 つだけ割り当てられます。ルックアップは必要に応じて親チェーンを辿ります。

### スコープのライフサイクル

- `enter_scope()` / `exit_scope()` — テーブル内のローカルフレームを push/pop します。`if`、`while`、`for` 本体で使用されます。
- `new_with_parent(parent)` — 親にリンクされた子テーブルを作成します。関数および fiber 本体で使用されます。
- `define(name, ty, is_const)` — 常に現在のテーブルの**最も内側**（現在）スコープに書き込みます。
- `lookup(name)` — ローカルスコープを最内側から最外側へ走査し、その後 `parent` ポインタチェーンを辿ります。
- `has_in_current_scope(name)` — 現在のテーブルの最内側フレームのみを確認します — 同一ブロック内の再定義検出に使用されます。
- `is_const(name)` — `name` を所有するスコープを見つけ、`consts[scope_index]` を確認します。

### 重要：変数のシャドウイングなし

XCX は**変数のシャドウイングをサポートしません**。**現在のスコープ**にすでに存在する変数を定義すると `RedefinedVariable` が発生します。親スコープの変数にはアクセスできますが、同じ名前で子スコープに再宣言することはできません。

## 型チェッカー（`src/sema/checker.rs`）

`Checker` 構造体は AST を走査し、`TypeError` 値を蓄積します。結果の `Vec<TypeError>` が空の場合にのみプログラムがコンパイルされます。

### チェッカーの状態

```rust
pub struct Checker<'a> {
    interner:       &'a Interner,
    loop_depth:     usize,
    functions:      HashMap<String, FunctionSignature>,
    fiber_context:  Option<Option<Type>>,
    is_table_lambda: bool,
    fiber_has_yield: bool,
}
```

### プリスキャンパス（`pre_scan_stmts`）

いずれかの文本体をチェックする前に、チェッカーは**現在の文リスト**内のすべての `FunctionDef` および `FiberDef` ノードに対して**前方宣言スキャン**（再帰的ではない）を実行します。これにより、ソースファイル内で定義より前に関数や fiber を呼び出すことができます（相互再帰、宣言前呼び出し）。

見つかった各関数／fiber について、チェッカーは以下を登録します：
- `self.functions` に `FunctionSignature { params: Vec<Type>, return_type: Option<Type>, is_fiber: bool }`
- `SymbolTable` に `Type::Unknown`（関数の場合）または `Type::Fiber(...)`（fiber の場合）のエントリ

組み込みキャスト関数 `i`、`f`、`s`、`b` は、`Type::Unknown` パラメータと対応する戻り値型で事前登録されます。

### コンテキストフラグ

| フィールド | 目的 |
|---|---|
| `loop_depth: usize` | `while`/`for` のネスト深度を追跡します。0 のとき `break`/`continue` はエラーです。fiber 本体に入ると 0 にリセットされます。 |
| `fiber_context: Option<Option<Type>>` | `None` = fiber 外；`Some(None)` = void fiber；`Some(Some(T))` = `T` を yield する型付き fiber。 |
| `fiber_has_yield: bool` | `yield` に遭遇したときに設定されます。入れ子 fiber 定義をまたいで保存／復元されます。 |
| `is_table_lambda: bool` | `.where()` 述語内で設定されます；`__row_tmp` でルックアップすることで、裸の列名を識別子として許可します。 |

### 型推論ルール

- 式の型はリテラルからボトムアップで推論され、演算子を通じて伝播します。
- `Type::Unknown` はワイルドカードとして機能します — `Unknown` を含む任意の演算はエラーなく通過します。
- `Type::Json` は代入および比較において任意の型と互換です。
- 数値の昇格：`Int op Float → Float`。
- 空の配列リテラル `[]` は、利用可能であれば代入コンテキストから型を継承します。
- `Type::Table([])`（空の列リスト）は任意の `Table(cols)` と互換です — 推論後に列情報が変数の宣言型へ伝播されます。

### `is_compatible(expected, actual) -> bool`

主要な互換性ルール：

| ルール | 説明 |
|---|---|
| いずれかが `Unknown` | 常に互換 |
| いずれかが `Json` | 常に互換 |
| `Int` ↔ `Float` | 相互互換（数値昇格） |
| `Int` ↔ `Date` | 互換（タイムスタンプは整数） |
| `Set(X)` ↔ `Array(inner)` | 内部要素型が一致するとき互換 |
| `Set(N)` ↔ `Set(Z)` | 互換（いずれも整数型セット） |
| `Set(S)` ↔ `Set(C)` | 互換（いずれも文字列型セット） |
| `Table([])` ↔ `Table(cols)` | いずれかの列リストが空のとき互換 |
| `Table(a)` ↔ `Table(b)` | 長さが一致し、すべての列型がペアごとに互換のとき互換 |
| `Map(k1,v1)` ↔ `Map(k2,v2)` | 再帰的にチェック |
| `Fiber(None)` ↔ `Fiber(None)` | 両方 void fiber |
| `Fiber(Some(T))` ↔ `Fiber(Some(U))` | T と U が互換のときのみ互換 |

### 関数呼び出しのチェック

名前付き関数呼び出し（`ExprKind::FunctionCall`）の場合：
1. まず `self.functions` でルックアップ（宣言済み関数／fiber 用）。
2. 見つからなければシンボルテーブルでルックアップ（第一級関数値、動的呼び出し用）。
3. 解決されたシグネチャが fiber の場合、呼び出しは `Type::Fiber(Some(ret))` を返します（fiber のインスタンス化であり、直接呼び出しではありません）。
4. 宣言されたパラメータ数を超える追加引数はチェックされますが拒否されません（可変長許容）。
5. 解決できない呼び出しは `UndefinedVariable` を追加します。

文レベルの関数呼び出し（`StmtKind::FunctionCallStmt`）の場合：
- 引数個数の不一致 → `Other("Function expects N arguments, got M")`。
- シンボルテーブルに存在しない未知の名前 → `UndefinedVariable`。

### Fiber 宣言のチェック（`FiberDecl`）

1. `self.functions`（優先）またはシンボルテーブルで `fiber_name` をルックアップ。
2. 解決されたシグネチャが `is_fiber: true` であることを検証。
3. 引数型をパラメータと照合。
4. 新しい変数を `Type::Fiber(inner_type)` で定義。

### テーブルに対する `.where()` のチェック

`table.where(predicate)` をチェックする際：
1. 一時スコープを開き、`__row_tmp: Table(cols)` を定義。
2. `is_table_lambda` を `true` に設定。
3. `collect_pred_idents()` が述語内で使用されるすべての `Identifier` 名を収集。
4. 識別子が外側スコープに存在し、**かつ**列名と一致する場合 → `S301 WherePredicateNameCollision`。
5. 述語を型チェック；`Bool` を返す必要があります。
6. スコープを終了し、`is_table_lambda` を復元。

### `Table.join()` のチェック

チェッカーは列定義をマージします：左テーブルの列に加え、（`StringId` の等価性により）まだ存在しない右テーブルの列を追加します。列名が競合する場合、右テーブルの列が保持されます。結果型は `Type::Table(combined_cols)` です。

### `for` ループのチェック

`ForIterType` の `iter_type` フィールドは、`start` の推論型に基づいてチェック中に変更されます：
- `Type::Array(_)` → `ForIterType::Array` を設定、ループ変数は内部型を取得
- `Type::Set(st)` → `ForIterType::Set` を設定、ループ変数はセットの要素型を取得
- `Type::Table(cols)` → `ForIterType::Array` として扱われ、ループ変数は `Type::Table(cols)` を取得
- `Type::Fiber(inner)` → `ForIterType::Fiber` を設定、ループ変数は内部 yield 型を取得
- `Type::Int`（`to` キーワードあり）→ `ForIterType::Range` のまま、ループ変数は `Int`

---

## 検証済みエラーコード

| コード | 条件 |
|---|---|
| `UndefinedVariable(name)` | 宣言前に名前が使用された |
| `RedefinedVariable(name)` | 同一スコープで名前が 2 回宣言された |
| `ConstReassignment(name)` | `const` 変数への代入 |
| `TypeMismatch { expected, actual }` | 式の型が期待型と一致しない |
| `InvalidBinaryOp { op, left, right }` | 互換性のない型で演算子が使用された |
| `BreakOutsideLoop` | `while`/`for` 外の `break` |
| `ContinueOutsideLoop` | `while`/`for` 外の `continue` |
| `[S208] YieldOutsideFiber` | fiber 本体外で `yield` が使用された |
| `[S209] FiberTypeMismatch` | void fiber 内の `yield expr;`（`yield;` であるべき） |
| `[S210] ReturnTypeMismatchInFiber` | 型付き fiber 内の裸の `return;`（戻り値が欠落） |
| `[S301] WherePredicateNameCollision` | `.where()` 内でローカル変数名がテーブル列名と競合 |
| `Other(msg)` | 各種コンテキストエラー（引数個数、未知のメソッド、void fiber の反復など） |

---

## エラー報告

`TypeError` 値は `Span { line, col, len }` を保持します。チェック後、`main.rs` は各エラーを `Reporter::error()` に渡し、以下を出力します：
1. `ERROR: <message>`
2. 行番号付きの該当ソース行
3. `col` から長さ `len` の `~~~` 下線

エラー報告の直後にコンパイルは停止します — いずれかの `TypeError` が存在する場合、バイトコードは生成されません。
