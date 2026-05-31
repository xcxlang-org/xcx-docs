# XCX Parser — v3.1

XCX Parser はトークンストリームを高レベルの抽象構文木 (AST) へ変換する。

## アーキテクチャ: Pratt Parsing

XCX は **Pratt Parser** (Top-Down Operator Precedence) を使用する。

- **File**: `src/parser/pratt.rs`
- **Lookahead**: 1 トークン (`current` + `peek`)、`advance()` で手動進行。
- **Error Recovery**: 構文エラー時、`synchronize()` が次のセミコロンまたは既知の文開始キーワード (`func`, `fiber`, `if`, `for`, `const`, `return`, `>!` など) までトークンをスキップする。

`Parser` 構造体はライフタイム `'a` でソース文字列を借用し、`Scanner<'a>` も同じライフタイムでパラメータ化され、バイトスライスベースのスキャナーを反映する。

### Precedence Levels (lowest → highest)

| Level | Operators |
|---|---|
| `Lowest` | — |
| `Lambda` | `->` |
| `Assignment` | `=` |
| `LogicalOr` | `OR`, `\|\|` |
| `LogicalAnd` | `AND`, `&&` |
| `Equals` | `==`, `!=` |
| `LessGreater` | `>`, `<`, `>=`, `<=`, `HAS` |
| `Sum` | `+`, `-`, `++` |
| `SetOp` | `UNION`, `INTERSECTION`, `DIFFERENCE`, `SYMMETRIC_DIFFERENCE` |
| `Product` | `*`, `/`, `%` |
| `Power` | `^` |
| `Prefix` | `-x` |
| `Call` | `.`, `[` |

## 文ディスパッチ

`parse_statement_internal()` は現在のトークンに基づいてディスパッチする:

- **Type keywords** (`i`, `f`, `s`, `b`, `array`, `set`, `map`, `date`, `table`, `json`) → `parse_var_decl()`、または `=` が続く場合は `parse_assignment()`
- **`const`** → `is_const = true` で `parse_var_decl()`
- **`var`** (identifier) → 型推論変数宣言
- **`>!`** → `parse_print_stmt()`
- **`>?`** → `parse_input_stmt()`
- **`halt`** → `parse_halt_stmt()`
- **`if`** → `parse_if_statement()`
- **`while`** → `parse_while_statement()`
- **`for`** → `parse_for_statement()`
- **`break`** / **`continue`** → `parse_break_statement()` / `parse_continue_statement()`
- **`func`** → `parse_func_def()`
- **`fiber`** → `parse_fiber_statement()` (peek に基づき def または decl へディスパッチ)
- **`return`** → `parse_return_stmt()`
- **`yield`** → `parse_yield_stmt()` (`yield expr`、`yield from expr`、`yield;` を処理)
- **`@wait`** → `parse_wait_stmt()`
- **`serve`** → `parse_serve_stmt()`
- **`net`** → `parse_net_stmt()`
- **`include`** → `parse_include_stmt()`
- **Identifier + `=`** → `parse_assignment()`
- **Identifier + `(`** → `parse_func_call_stmt()`
- **Anything else** → `parse_expr_stmt()`

## 関数定義スタイル

XCX は関数定義に 2 つの構文的に異なるスタイルをサポートする:

**Brace style** (C-like):
```xcx
func name(i: x, s: y -> i) {
    return x + 1;
}
```

**XCX style** (keyword block):
```xcx
func:i: name(i: x, s: y) do;
    return x + 1;
end;
```

両方とも同一の `StmtKind::FunctionDef` AST ノードを生成する。brace style の戻り値型はパラメータリスト内または `)` の後に `-> type` で宣言される。

## Fiber 文

`parse_fiber_statement()` は `peek` を見て判定する:
- `peek == Colon` → `parse_fiber_decl()` (インスタンス化: `fiber:T: varname = fiberDef(args);`)
- それ以外 → `parse_fiber_def()` (定義: `fiber name(params) { body }`)

`parse_fiber_decl()` は、型と名前をパースした後に現在のトークンが `(` の場合も処理 — その場合 `finish_fiber_def()` (先頭に `fiber:` 型注釈付きの定義) へ切り替える。

## パース対象の主要構文

- **Variable declarations**: `i: name = expr;`, `const s: NAME = expr;`, `var name = expr;`
- **Control flow**: `if (cond) then; ... elseif (cond) then; ... else; ... end;`
- **While loop**: `while (cond) do; ... end;`
- **For loop**: `for x in expr do; ... end;` および `for x in start to end @step n do; ... end;`
- **Functions**: `func` (上記 2 スタイル)
- **Fibers**: `fiber name(params) { body }` および `fiber:T: varname = fiberName(args);`
- **Yield**: `yield expr;`, `yield from expr;`, `yield;`
- **HTTP**: `serve: name { port=..., routes=... };`, `net.get(url)`, `net.request { ... } as resp;`, `net.respond(status, body);`
- **Collections**: Array `[a, b, c]`, Set `set:N { 1,,10 }`, Map `[k :: v, ...]`, Table `table { columns=[...] rows=[...] }`
- **Raw blocks**: インライン JSON/文字列用 `<<<...>>>`
- **Include**: `include "path";` または `include "path" as alias;`
- **I/O**: `>! expr;` (print), `>? varname;` (input)
- **Halt**: `halt.alert >! msg;`, `halt.error >! msg;`, `halt.fatal >! msg;`
- **Wait**: `@wait(ms);` または `@wait ms;`
- **Date literals**: `date("2024-01-01")` または `date("01/01/2024", "DD/MM/YYYY")`

## 式のパース

`parse_expression(precedence)` は左辺に `parse_prefix()` を呼び出し、peek トークンの優先度が現在の最小値を超える間 `parse_infix(left)` をループで呼び出す。

主要な prefix パーサー:
- **Identifiers**: `(` が続く場合は `FunctionCall`、それ以外は `Identifier` としてパース。
- **Literals**: `IntLiteral`, `FloatLiteral`, `StringLiteral`, `True`, `False`
- **Unary minus**: `Binary { left: IntLiteral(0), op: Minus, right }` としてパース (独立した `Unary::Neg` はない)
- **`not` / `!`**: `Unary { op: Not/Bang, right }`
- **`(`...`)` groups**: 単一式 → アンラップ; カンマ区切り複数 → `Tuple`
- **`[`...`]`**: 最初の要素の後に `::` が続く場合は `MapLiteral`、それ以外は `ArrayLiteral`
- **`{`...`}`**: `ArrayOrSetLiteral` としてパース (型は意味解析またはコンパイル時に解決)
- **`set:N { }` など**: 既知の `SetType` 付き明示的 `SetLiteral`
- **`map { schema=[...] data=[...] }`**: 明示的 `MapLiteral`
- **`table { columns=[...] rows=[...] }`**: `TableLiteral`
- **`random.choice from expr`**: `RandomChoice`
- **`date(...)`**: `DateLiteral`
- **`net.get/post/put/delete/patch(...)` など**: `NetCall` または `NetRespond`
- **`<<<...>>>`**: `RawBlock`
- **`.terminal!cmd`**: `TerminalCommand`

主要な infix パーサー:
- **`.`**: `parse_dot_infix` — `(` が続く場合は `MethodCall`、それ以外は `MemberAccess`; `.[key]` インデックスアクセスも処理
- **`[`**: `parse_index_infix` → `Index`
- **`->`**: `parse_lambda_infix` → `Lambda`
- **All binary operators**: `Binary { left, op, right }`

## `parse_expr_stmt()` Post-Processing

完全な式文をパースした後、`parse_expr_stmt()` は結果が `MethodCall` かどうかを確認する:
- メソッド名 `bind` で引数 2 つ、第 2 引数が `Identifier` → `StmtKind::JsonBind` として書き換え
- メソッド名 `inject` で引数 2 つ → `StmtKind::JsonInject` として書き換え

これにより文レベルで `json.bind("path", target);` と `json.inject(mapping, table);` の糖衣構文が可能になる。

## Expander (`src/parser/expander.rs`)

Expander はパース**後**、意味解析**前**に実行される。独立したツリー書き換えパスである。

### Responsibilities

**Include resolution**: `include "file.xcx";` はそのファイルのインライン AST に置き換えられる。循環依存は `visiting_files: HashSet<PathBuf>` で検出される。ファイルは `included_files: HashSet<PathBuf>` で重複排除される (エイリアス付きでない限り各ファイルは一度だけ)。

**Alias prefixing**: `include "math.xcx" as math;` により、そのファイルのすべてのトップレベル名が `math.name` へリネームされる。呼び出しサイト (`math.sin(x)`) は `expand_expr_inplace` により `MethodCall` から `FunctionCall { name: "math.sin" }` へ書き換えられる。`prefix_program()` / `prefix_stmt_impl()` / `prefix_expr_impl()` 関数がサブ AST 全体を走査し、トップレベルシンボルへのすべての参照をリネームする。

**Fiber name prefixing**: `FiberDecl::fiber_name` 参照もプレフィックスされ、リネーム後の fiber インスタンス化が正しく解決される。

**YieldFrom prefixing**: `StmtKind::YieldFrom` 式を走査し、`yield from` 内の fiber コンストラクタ呼び出しもリネームされる。

**Protected names** (プレフィックスされない): `json`, `date`, `store`, `halt`, `terminal`, `net`, `env`, `crypto`, `EMPTY`, `math`, `random`, `i`, `f`, `s`, `b`, `from`, `main`.

**Include path search order**:
1. 現在のファイルのディレクトリからの相対パス
2. `lib/` ディレクトリ (CWD からの相対、次に実行ファイルのパスから上方向にたどる)

## AST 定義 (`src/parser/ast.rs`)

### `Expr` — Expression nodes

| Variant | Description |
|---|---|
| `IntLiteral(i64)` | 整数定数 |
| `FloatLiteral(f64)` | 浮動小数点定数 |
| `StringLiteral(StringId)` | インターン済み文字列 |
| `BoolLiteral(bool)` | `true` / `false` |
| `Identifier(StringId)` | 変数または関数名 |
| `Binary { left, op, right }` | 二項演算 |
| `Unary { op, right }` | 単項演算 (`not`, `!`) |
| `FunctionCall { name, args }` | インターン名による関数呼び出し |
| `MethodCall { receiver, method, args }` | 値に対するドット呼び出し |
| `MemberAccess { receiver, member }` | 呼び出しなしのドットアクセス |
| `Index { receiver, index }` | 角括弧インデックス `a[i]` |
| `Lambda { params, return_type, body }` | 矢印ラムダ `x -> expr` |
| `ArrayLiteral { elements }` | 明示的 `[a, b, c]` |
| `ArrayOrSetLiteral { elements }` | 曖昧な `{a, b, c}` — 後で解決 |
| `SetLiteral { set_type, elements, range }` | オプション範囲付き型付き集合 |
| `MapLiteral { key_type, value_type, elements }` | Map リテラル |
| `TableLiteral { columns, rows }` | Table リテラル |
| `DateLiteral { date_string, format }` | `date("2024-01-01")` |
| `Tuple(Vec<Expr>)` | 括弧付きカンマ区切りリスト |
| `NetCall { method, url, body }` | HTTP 呼び出し式 |
| `NetRespond { status, body, headers }` | HTTP 応答式 |
| `RawBlock(StringId)` | `<<<...>>>` 生コンテンツ |
| `TerminalCommand(cmd, arg)` | `.terminal !cmd` |
| `RandomChoice { set }` | `random.choice from set` |

### `Stmt` — Statement nodes

主要バリアント: `VarDecl`, `Assign`, `Print`, `Input`, `If`, `While`, `For`, `Break`, `Continue`, `FunctionDef`, `FiberDef`, `FiberDecl`, `Return`, `Yield`, `YieldFrom`, `YieldVoid`, `Include`, `Serve`, `NetRequestStmt`, `JsonBind`, `JsonInject`, `Halt`, `Wait`, `ExprStmt`, `FunctionCallStmt`.

### `Type` — Type system

`Int`, `Float`, `String`, `Bool`, `Date`, `Json`, `Array(Box<Type>)`, `Set(SetType)`, `Map(Box<Type>, Box<Type>)`, `Table(Vec<ColumnDef>)`, `Fiber(Option<Box<Type>>)`, `Builtin(StringId)`, `Unknown`.

`SetType` バリアント: `N` (Natural), `Z` (Integer), `Q` (Rational/Float), `S` (String), `C` (Char/String), `B` (Boolean).

### `ForIterType`

`Range` (数値 `start to end`)、`Array`、`Set`、`Fiber` — 型チェッカーが設定し、コンパイラが正しいループパターンを出力するために使用する。

### `ColumnDef`

```rust
pub struct ColumnDef {
    pub name:    StringId,
    pub ty:      Type,
    pub is_auto: bool,    // @auto columns are auto-incremented on insert
}
```

## 文字列 Interner

すべての文字列値 (識別子、文字列リテラル、メソッド名) は `Interner` 経由で `StringId (u32)` へインターンされる。interner はパーサーで作成され、以降のすべてのフェーズに渡される。つまり checker、compiler、VM はすべて名前比較に `String` 比較の代わりに数値 ID を使用する。
