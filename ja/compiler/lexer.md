# XCX Lexer (Scanner) — v3.1

Lexer は生のソースバイトストリームを離散的なトークンのストリームへ変換する責務を持つ。

## 実装詳細

- **File**: `src/lexer/scanner.rs`
- **Technique**: `&[u8]` 上での手動・即時・バイト単位スキャン。
- **API**: 単一メソッド `next_token(&mut self, interner: &mut Interner) -> Token`。パーサーからオンデマンドで呼び出される (イテレータではない)。
- **Lookahead**: `peek()` による 1 バイト先読み、`peek_next()` / `peek_at(offset)` による 2 バイト先読み。

## 内部構造

```rust
pub struct Scanner<'a> {
    source:   &'a [u8],   // borrowed reference to original source — no allocation
    pos:      usize,      // byte position
    char_pos: usize,      // Unicode character position (for Span.col)
    line:     usize,
    col:      usize,
}
```

`Scanner` はソース文字列をバイトスライスとして借用する。事前の `Vec<char>` 変換は行わない — すべてのスキャンはバイトレベル。`char_pos` は UTF-8 継続バイト (`10xxxxxx`) 以外のバイトに対してのみインクリメントされ、すべての文字をデコードせずに Span 報告用の Unicode 文字数が正確に保たれる。

## トークン型

トークンは `src/lexer/token.rs` で `TokenKind` enum として定義される。各 `Token` はエラー報告用の `Span { line, col, len }` を持つ。`len` はバイトではなく Unicode 文字数 (`char_pos` の差分) で計測される。

主要カテゴリ:

| Category | Examples |
|---|---|
| Literals | `IntLiteral(i64)`, `FloatLiteral(f64)`, `StringLiteral(StringId)`, `True`, `False` |
| Type keywords | `TypeI`, `TypeF`, `TypeS`, `TypeB`, `Array`, `Set`, `Map`, `Table`, `Json`, `Date`, `Fiber` |
| Set type keywords | `TypeSetN`, `TypeSetQ`, `TypeSetZ`, `TypeSetS`, `TypeSetB`, `TypeSetC` |
| Control flow | `If`, `Then`, `ElseIf`, `Else`, `End`, `While`, `Do`, `For`, `In`, `To`, `Break`, `Continue` |
| Functions/Fibers | `Func`, `Return`, `Fiber`, `Yield` |
| Operators | `Plus`, `Minus`, `Star`, `Slash`, `Caret`, `PlusPlus`, `Has`, `And`, `Or`, `Not` |
| Set operators | `Union`, `Intersection`, `Difference`, `SymDifference` |
| Special punctuation | `GreaterBang` (`>!`), `GreaterQuestion` (`>?`), `DoubleColon` (`::`), `DoubleComma` (`,,`), `Bridge` (`<->`) |
| Builtins | `Net`, `Serve`, `Store`, `Halt`, `Terminal`, `Json`, `Date` |
| Special | `RawBlock(StringId)`, `AtStep`, `AtAuto`, `AtWait` |

## 特殊スキャン機能

### Raw Blocks
`<<<` と `>>>` で区切られる。間のすべてが単一の `RawBlock(StringId)` トークンとして取得され、インライン JSON や複数行文字列データに使用される。

```
<<<
  { "key": "value" }
>>>
```

検出は `self.source[self.pos..].starts_with(b"<<<")` を使用 — メイン match arm の `<` 曖昧性解消の一部として開く。生コンテンツは `>>>` が見つかるまでバイト単位で取得される。

### Comments
XCX は `---` をコメント区切りとして使用する。検出は `self.source[self.pos..].starts_with(b"---")`:
- **Single-line**: `--- this is a comment` (`---` 以降の同一行に非空白コンテンツ)
- **Multi-line**: 行末まで空白のみが続く `---` がブロックを開き、`*---` で閉じる

複数行クローズは `self.source[self.pos..].starts_with(b"*---")` で検出される。スキャナーは `---` 消費後に先読みし、どちらのモードが適用されるかを判定する。

### Unicode Set Operators
スキャナーは UTF-8 バイト列に対する `starts_with` で Unicode 記号を認識する:
- `∪` → `TokenKind::Union`
- `∩` → `TokenKind::Intersection`
- `\` (ASCII backslash) → `TokenKind::Difference`
- `⊕` → `TokenKind::SymDifference`

マルチバイト Unicode 演算子では、先頭バイト (`c >= 128`) 後の残りの継続バイトを消費するため、適切な回数だけ `advance()` が追加で呼ばれる。

### Else/ElseIf Disambiguation
スキャナーは `else` / `els` 認識後に先読みし、次の単語が `if` かどうかを確認 — そうであれば 2 語を単一の `ElseIf` トークンにまとめる。保存位置 (`after_ws_pos`, `after_ws_char_pos`, `after_ws_line`, `after_ws_col`) により、次の単語が `if` でない場合にバックトラックできる。

### `@` Directives
`@` で始まるトークンは ASCII 英字バイトを消費し、結果をマッチングする:
- `@step` → `AtStep`
- `@auto` → `AtAuto`
- `@wait` → `AtWait`

未知の `@` シーケンスは `TokenKind::Unknown('@')` を生成する。

### Dot-Dot (`..`) → `To`
連続する 2 つのドット `..` は `To` トークン (範囲式で使用) としてスキャンされ、単一の `.` (`Dot`) とは区別される。

### Double-Comma (`,,`) → `DoubleComma`
連続する 2 つのカンマ `,,` は `DoubleComma` (集合範囲リテラルで使用: `set:N { 1,,10 }`) としてスキャンされる。

### Identifier Scanning
`identifier()` は ASCII 英数字バイト、アンダースコア、`>= 128` のバイト (UTF-8 識別子を含む) の連続区間を取得する。取得したバイト範囲は `std::str::from_utf8` で変換され、キーワードマッチングのために小文字化される。大文字小文字を区別するマッチを先に確認 (例: 集合型の `"N"`, `"Q"`, `"Z"`、大文字キーワードバリアントの `"UNION"`, `"HAS"`, `"AND"`)。非キーワード識別子は `Interner::intern()` に渡される。

### Number Scanning
`number()` は ASCII 数字バイトを蓄積する。`.` の後に数字が続く場合、トークンは `FloatLiteral` になる。バイト範囲は `str::from_utf8` で変換され `.parse()` でパースされる。

### String Scanning
`string()` はエスケープシーケンス (`\n`, `\t`, `\r`, `\"`, `\\`, 8 進 `\NNN`, 16 進 `\xHH`) をバイト単位で処理し、`Vec<u8>` を構築してから `String::from_utf8` で変換する。結果はインターンされる。

## 文字列割り当て

すべての識別子と文字列リテラルは `Interner::intern()` を通過し、`StringId (u32)` を返す。生の `String` は interner の内部 `Vec<String>` に一度だけ格納され、パイプラインの残りは数値 ID で動作し、型検査とコンパイル中のヒープ比較を排除する。
