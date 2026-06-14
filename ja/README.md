# XCX 技術ドキュメント スイート

> 📌 **ご注意**: この日本語ドキュメントは AI によって翻訳されているため、不正確さや誤訳がある可能性があります。申し訳ございません。より正確な情報については、[英語版ドキュメント](../en/README.md)をご参照ください。

XCX 3.1 と XCX コンパイラの公式技術ドキュメントへようこそ。

## 📖 言語リファレンス
XCX言語の構文と機能に関する包括的なガイド。

- [構文の基本](language/syntax.md): コメント、識別子、ブロック
- [変数と定数](language/variables.md): 宣言、再割り当て、シャドウイング
- [データ型](language/types.md): 単純型、複合型、型キャスト
- [演算子](language/operators.md): 算術、文字列、論理、比較、セット
- [制御フロー](language/control_flow.md): 条件分岐とループ
- [関数とファイバー](language/functions_fibers.md): サブルーチン、コルーチン、委譲
- [コレクション](language/collections.md): 配列、セット、マップ、テーブル（リレーショナル）
- [JSON と HTTP](language/json_http.md): データ交換、ネットワーク処理、安全性
- [日付と時刻](language/dates.md): 作成、フィールド、フォーマット
- [入出力とターミナル](language/io_terminal.md): 入力、出力、遅延、システムコマンド
- [文字列メソッド](language/string_methods.md): 文字列ユーティリティの包括的なリスト
- [エラーハンドリング](language/errors_halt.md): ハルトレベル（alert、error、fatal）
- [標準ライブラリ](language/library_modules.md): 組み込みモジュール（`crypto`、`store`、`env`）

## ⚙️ コンパイラ内部
XCX コンパイラと VM がどのように動作するかについての技術的な詳細な解説。

- [アーキテクチャの概要](compiler/architecture.md): コンパイルパイプライン
- [レキサー](compiler/lexer.md): トークン化と再帰的スキャン
- [パーサー](compiler/parser.md): Pratt パースと AST 生成
- [セマンティック分析](compiler/semantics.md): 型チェックとシンボル解決
- [仮想マシン](compiler/vm.md): スタックマシンアーキテクチャと OpCodes

## 📦 ツーリング
XCX エコシステムツールのガイド。

- [PAX マニュアル](pax/pax_manual.md): パッケージ管理とプロジェクト構造

---
*ドキュメント バージョン: 3.0.0 *
