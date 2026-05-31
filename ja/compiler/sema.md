# XCX 意味解析（Sema）— ドキュメント

> **ファイル：** `src/sema/checker.rs`、`src/sema/symbol_table.rs`、`src/sema/interner.rs`

---

## 目次

1. [概要](#概要)
2. [文字列インターナー](#文字列インターナー)
3. [シンボルテーブル](#シンボルテーブル)
4. [型チェッカー](#型チェッカー)
5. [型互換性ルール](#型互換性ルール)
6. [エラーコード](#エラーコード)
7. [エラー報告](#エラー報告)

---

## 概要

Sema フェーズは、バイトコード生成の前に AST の論理的正しさと型の一貫性を検証します。3 つのコンポーネントで構成されます：

```
Interner → StringId (u32)
     ↓
SymbolTable → hierarchical type scopes
     ↓
Checker → collection of TypeErrors
```

結果の `Vec<TypeError>` が空の場合にのみプログラムがコンパイルされます。

---

## 文字列インターナー

**ファイル：** `src/sema/interner.rs`

`Interner` は `&str → StringId(u32)` をマッピングします。コンパイラ内のすべての文字列 ID の唯一の情報源です。字句解析／構文解析時に作成され、参照渡しでチェッカーとコンパイラに渡されます。

```rust
Interner::intern("foo") → StringId(42)
Interner::lookup(StringId(42)) → "foo"
```

**実装：**

```rust
pub struct Interner {
    map:     HashMap<String, StringId>,
    strings: Vec<String>,
}
```

各一意の文字列は `strings: Vec<String>` に 1 回だけ格納されます。パイプラインの残りは数値 ID で動作し、型チェックとコンパイル中のヒープ比較を排除します。

---

## シンボルテーブル

**ファイル：** `src/sema/symbol_table.rs`

`SymbolTable` は、**親ポインタのチェーン**を用いて入れ子スコープ内の変数バインディングを管理します。

### 構造

```rust
pub struct SymbolTable<'a> {
    parent: Option<&'a SymbolTable<'a>>,  // ref to surrounding scope
    scopes: Vec<HashMap<String, Type>>,   // stack of scope frames (local)
    consts: Vec<HashSet<String>>,         // which names are const, per frame
}
```

### 子スコープの作成

関数または fiber 本体に入る際、囲むテーブルをクローンするのではなく、その参照を持つ新しい `SymbolTable` が作成されます：

```rust
let mut func_symbols = SymbolTable::new_with_parent(symbols);
```

これは O(1) です — 新しいフレーム用に空の `HashMap` が 1 つだけ割り当てられます。ルックアップは必要に応じて親チェーンを辿ります。

### スコープのライフサイクル

| メソッド | 説明 |
|---|---|
| `enter_scope()` | 新しいフレームを push — `if`、`while`、`for` 本体用 |
| `exit_scope()` | 現在のスコープを終了 |
| `new_with_parent(parent)` | 子テーブルを作成 — 関数／fiber 本体用 |
| `define(name, ty, is_const)` | 常に**最も内側**のスコープに書き込み |
| `lookup(name)` | 内側から外側へ走査し、その後親へ |
| `has_in_current_scope(name)` | 最内側フレームのみを確認 — 再定義検出用 |
| `is_const(name)` | 所有スコープが名前を const としてマークしたか確認 |

### 変数のシャドウイングなし

XCX は**変数のシャドウイングをサポートしません**。**現在のスコープ**にすでに存在する変数を定義すると `RedefinedVariable` が返されます。親スコープの変数にはアクセスできますが、同じ名前で子スコープに再宣言することはできません。

---

## 型チェッカー

**ファイル：** `src/sema/checker.rs`

`Checker` 構造体は AST を走査し、`TypeError` 値を蓄積します。

### チェッカーの状態

```rust
pub struct Checker<'a> {
    interner:          &'a Interner,
    loop_depth:        usize,
    functions:         HashMap<String, FunctionSignature>,
    fiber_context:     Option<Option<Type>>,  // None=outside fiber, Some(None)=void, Some(Some(T))=typed
    is_table_lambda:   bool,
    fiber_has_yield:   bool,
    in_yield_expr:     bool,
    last_expr_was_db_io: bool,
}
```

### コンテキストフラグ

| フィールド | 目的 |
|---|---|
| `loop_depth: usize` | `while`/`for` のネスト深度を追跡します。0 のとき `break`/`continue` はエラーです。fiber 本体に入ると 0 にリセットされます。 |
| `fiber_context: Option<Option<Type>>` | `None` = fiber 外；`Some(None)` = void fiber；`Some(Some(T))` = `T` を yield する型付き fiber |
| `fiber_has_yield: bool` | `yield` に遭遇したときに設定されます。入れ子 fiber 定義により保存／復元されます。 |
| `is_table_lambda: bool` | `.where()` 述語内で設定されます；`__row_tmp` 経由で裸の列名を識別子として許可 |
| `in_yield_expr: bool` | yield 式内にいるかどうかを追跡 |
| `last_expr_was_db_io: bool` | データベース I/O 操作用フラグ |

### プリスキャンパス

いずれかの文本体をチェックする前に、チェッカーは**現在の文リスト**内のすべての `FunctionDef` および `FiberDef` ノードに対して**前方宣言スキャン**（非再帰的）を実行します。これにより、ソースファイル内で定義より前に関数や fiber を呼び出すことができます（相互再帰、宣言前呼び出し）。

見つかった各関数／fiber について、チェッカーは以下を登録します：
- `self.functions` に `FunctionSignature { params: Vec<Type>, return_type: Option<Type>, is_fiber: bool }`
- `SymbolTable` に `Type::Unknown`（関数の場合）または `Type::Fiber(...)`（fiber の場合）のエントリ

組み込みキャスト関数 `i`、`f`、`s`、`b` は事前登録されます。

### 型推論ルール

- 式の型はリテラルからボトムアップで推論され、演算子を通じて伝播します。
- `Type::Unknown` はワイルドカードとして機能します — `Unknown` を含む任意の演算はエラーなく通過します。
- `Type::Json` は代入および比較において任意の型と互換です。
- 数値の昇格：`Int op Float → Float`。
- 空の配列リテラル `[]` は、利用可能であれば代入コンテキストから型を継承します。
- `Type::Table([])`（空の列リスト）は任意の `Table(cols)` と互換です。

---

## 型互換性ルール

`is_compatible(expected, actual) -> bool` 関数：

| ルール | 説明 |
|---|---|
| いずれかが `Unknown` | 常に互換 |
| いずれかが `Json` | 常に互換 |
| `Int` ↔ `Float` | 相互互換（数値昇格） |
| `Int` ↔ `Date` | 互換（タイムスタンプは整数） |
| `Set(X)` ↔ `Array(inner)` | 要素型が一致するとき互換 |
| `Set(N)` ↔ `Set(Z)` | 互換（いずれも整数型セット） |
| `Set(S)` ↔ `Set(C)` | 互換（いずれも文字列型セット） |
| `Table([])` ↔ `Table(cols)` | いずれかの列リストが空のとき互換 |
| `Table(a)` ↔ `Table(b)` | 長さが一致し、列型がペアごとに一致するとき互換 |
| `Map(k1,v1)` ↔ `Map(k2,v2)` | 再帰的にチェック |
| `Fiber(None)` ↔ `Fiber(None)` | 両方 void fiber |
| `Fiber(Some(T))` ↔ `Fiber(Some(U))` | T と U が互換のとき互換 |

---

## エラーコード

| コード | 条件 |
|---|---|
| `[S101] UndefinedVariable(name)` | 宣言前に名前が使用された |
| `[S102] RedefinedVariable(name)` | 同一スコープで名前が 2 回宣言された |
| `[S103] TypeMismatch { expected, actual }` | 式の型が期待型と一致しない |
| `[S104] InvalidBinaryOp { op, left, right }` | 互換性のない型で演算子が使用された |
| `[S105] ConstReassignment(name)` | `const` 変数への代入 |
| `[S106] BreakOutsideLoop` | `while`/`for` 外の `break` |
| `[S107] ContinueOutsideLoop` | `while`/`for` 外の `continue` |
| `[S108] IndexAccessNotSupported(type)` | サポートされていない型へのインデックスアクセス |
| `[S109] PropertyNotFound` | 型にプロパティが存在しない |
| `[S110] MethodNotFound` | 型にメソッドが存在しない |
| `[S111] InvalidArgumentCount` | 引数の数が不正 |
| `[S208] YieldOutsideFiber` | fiber 本体外で `yield` が使用された |
| `[S209] FiberTypeMismatch` | void fiber 内の `yield expr;`（`yield;` であるべき） |
| `[S210] ReturnTypeMismatchInFiber` | 型付き fiber 内の裸の `return;`（戻り値が欠落） |
| `[S211] CannotIterateOverVoidFiber` | void fiber の反復 |
| `[S212] CannotRunTypedFiber` | 型付き fiber に対する `.run()` の呼び出し |
| `[S301] WherePredicateNameCollision` | `.where()` 内でローカル変数名が列名と競合 |
| `[S302] TableRowCountMismatch` | テーブル行の列数がスキーマと異なる |
| `[D401] Rule violation` | `.where()` なしの `remove()` |
| `Other(msg)` | その他のコンテキストエラー（引数個数、未知のメソッドなど） |

---

## エラー報告

`TypeError` 値は `Span { line, col, len }` を保持します。チェック後、`main.rs` は各エラーを `Reporter::error()` に渡し、以下を出力します：

```
ERROR: [S101] Undefined variable: foo
   12 | i: x = foo + 1;
              ~~~
```

エラー報告の直後にコンパイルは停止します — いずれかの `TypeError` が存在する場合、バイトコードは生成されません。

---

## 関数呼び出しのチェック

名前付き関数呼び出し（`ExprKind::FunctionCall`）の場合：
1. まず `self.functions` でルックアップ（宣言済み関数／fiber 用）
2. 見つからなければシンボルテーブルでルックアップ（第一級関数値、動的呼び出し用）
3. 解決されたシグネチャが fiber の場合、呼び出しは `Type::Fiber(Some(ret))` を返します（fiber のインスタンス化であり、直接呼び出しではない）
4. 宣言されたパラメータ数を超える追加引数はチェックされますが拒否されません（可変長許容）
5. 解決できない呼び出しは `UndefinedVariable` エラーを追加

---

## テーブルに対する `.where()` のチェック

`table.where(predicate)` をチェックする際：
1. 一時スコープを開き、`__row_tmp: Table(cols)` を定義
2. `is_table_lambda` を `true` に設定
3. `collect_pred_idents()` が述語内で使用されるすべての `Identifier` 名を収集
4. 識別子が外側スコープと列名の**両方**に存在する場合 → `S301 WherePredicateNameCollision`
5. 述語をチェック；`Bool` を返す必要がある
6. スコープを閉じ、`is_table_lambda` を復元
