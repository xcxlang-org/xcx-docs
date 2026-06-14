# PAX パッケージマネージャ マニュアル

PAX は XCX 公式のパッケージマネージャで、`xcx` バイナリに直接統合されている。依存関係、プロジェクトのスキャフォールディング、ビルド自動化を管理する。

## プロジェクト設定: `project.pax`

すべての PAX プロジェクトはルートディレクトリに `project.pax` ファイルを持つ必要がある。カスタムの宣言型フォーマットを使用する。

```pax
---
PAX Project Configuration
*---
/
    name :: "my_project",
    deps :: [
        "user/repo",
        "https://example.com/lib.xcx"
    ]
/
```

- **name**: プロジェクトの論理名。
- **deps**: 依存関係のリスト。GitHub ショートカット (`user/repo`) と直接 URL をサポートする。

## コマンドリファレンス

PAX は `xcx pax <command>` で呼び出す。

| コマンド                  | 説明                                          |
|--------------------------|------------------------------------------------------|
| `xcx pax new <name>`     | 新しいプロジェクト構成を生成する。                   |
| `xcx pax clone <package>`| 公開パッケージをローカルプロジェクトとしてダウンロードする。    |
| `xcx pax install`        | 依存関係を `lib/` ディレクトリに取得する。      |
| `xcx pax add <dep>`      | 依存関係を追加し、即座にインストールする。       |
| `xcx pax remove <name>`  | `project.pax` から依存関係を削除する。             |
| `xcx pax search <query>` | レジストリで利用可能なパッケージを検索する。        |
| `xcx pax run [path]`     | プロジェクトを実行する (エントリ: `src/main.xcx`)。        |

### `xcx pax clone <package>`

PAX レジストリから公開パッケージを新しいローカルディレクトリにダウンロードし、実行または変更の準備を整える。

```sh
xcx pax clone snake_game
xcx pax clone beta/snake_game
```

- 引数はレジストリに記載されているパッケージ名 (`author/` プレフィックスは任意)。
- 現在の作業ディレクトリにパッケージ名のディレクトリを作成する。
- 完全なプロジェクト構成をコピーする: `project.pax`、`src/`、および `files` で宣言されたその他のファイル。
- 依存関係は**自動的には**インストールしない — クローン後にクローン先ディレクトリ内で `xcx pax install` を実行する。

```sh
xcx pax clone snake_game
cd snake_game
xcx pax install
xcx pax run
```

## ディレクトリ構成

標準的な PAX プロジェクトは以下のレイアウトに従う:
- `project.pax`: 設定。
- `src/`: ソースコード (メインエントリ: `main.xcx`)。
- `lib/`: ダウンロードされた依存関係 (PAX が管理)。
- `tests/`: プロジェクト固有のテスト。
