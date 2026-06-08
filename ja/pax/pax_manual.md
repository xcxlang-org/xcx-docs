# PAX パッケージマネージャーマニュアル

PAX は XCX の公式パッケージマネージャーで、`xcx` バイナリに直接統合されています。依存関係の管理、プロジェクトのスキャフォールディング、レジストリへの公開、ビルドの自動化を担います。

---

## プロジェクト設定：`project.pax`

すべての PAX プロジェクトは、ルートディレクトリに `project.pax` ファイルを持つ必要があります。カスタムの宣言形式を使用します。

```pax
---
PAX Project Configuration
*---
/
    name        :: "my_project",
    version     :: "1.0.0",
    author      :: "yourname",
    description :: "A short description of the project",
    main        :: "src/main.xcx",
    tags  :: ["tag1", "tag2"],
    files :: ["src/main.xcx", "src/lib.xcx"],
    deps  :: [
        "somelib@1.0.0",
        "author/repo@latest",
        "https://example.com/lib.xcx"
    ]
/
```

### フィールド一覧

| フィールド    | 必須 | 説明                                                                         |
|---------------|------|------------------------------------------------------------------------------|
| `name`        | はい | プロジェクト/パッケージの論理名。                                             |
| `version`     | はい | セマンティックバージョン文字列（例：`"1.0.0"`）。                             |
| `author`      | いいえ | 作者名。レジストリに表示されます。                                           |
| `description` | いいえ | プロジェクトの短い説明。                                                     |
| `main`        | いいえ | カスタムエントリーポイント。省略時は `src/main.xcx` がデフォルト。           |
| `tags`        | いいえ | レジストリでの検索性を高めるタグのリスト。                                   |
| `files`       | いいえ | 公開時に含めるファイルの明示的なリスト。省略時は PAX が非コードファイルを自動除外。 |
| `deps`        | いいえ | 依存関係のリスト。バージョン指定、`author/repo` 短縮形、直接 URL をサポート。 |

### 依存関係の形式

```pax
deps :: [
    "mylib@1.0.0",           --- バージョン指定のレジストリパッケージ
    "mylib@latest",          --- 最新バージョンのレジストリパッケージ
    "author/mylib@2.1.0",    --- 作者プレフィックス付き
    "https://example.com/lib.xcx"  --- 直接 URL
]
```

---

## ロックファイル：`pax.lock`

PAX はインストールのたびに `pax.lock` ファイルを自動管理します。インストールされた各依存関係の正確な解決済みバージョンとローカルパスを記録します。

```json
{
  "mylib": { "version": "mylib@1.0.0", "file": "lib/mylib/" },
  "otherlib": { "version": "otherlib@latest", "file": "lib/otherlib/" }
}
```

- `pax.lock` を**手動で編集しないでください**。
- 再現可能なビルドのためにバージョン管理にコミットしてください。

---

## ディレクトリ構成

標準的な PAX プロジェクトのレイアウト：

```
my_project/
├── project.pax       # プロジェクト設定
├── pax.lock          # 自動生成されるロックファイル
├── src/
│   └── main.xcx      # メインエントリーポイント
├── lib/              # インストール済み依存関係（PAX が管理）
└── tests/            # プロジェクト固有のテスト
```

---

## コマンドリファレンス

PAX は `xcx pax <command>` で呼び出します。

| コマンド                     | 説明                                           |
|------------------------------|------------------------------------------------|
| `xcx pax new <name>`         | 新しいプロジェクト構造を作成する。             |
| `xcx pax install`            | すべての依存関係を `lib/` にダウンロードする。 |
| `xcx pax add <dep>`          | 依存関係を追加して即座にインストールする。     |
| `xcx pax remove <name>`      | `project.pax` から依存関係を削除する。         |
| `xcx pax clone <package>`    | 公開済みパッケージをローカルプロジェクトとしてダウンロードする。 |
| `xcx pax run [path]`         | プロジェクトのエントリーポイントを実行する。   |
| `xcx pax search <query>`     | レジストリで利用可能なパッケージを検索する。   |
| `xcx pax login <token>`      | レジストリ認証トークンを保存する。             |
| `xcx pax logout`             | 保存された認証トークンを削除する。             |
| `xcx pax whoami`             | 現在のレジストリアカウントを確認する。         |
| `xcx pax publish`            | プロジェクトを PAX レジストリに公開する。      |

---

## コマンド詳細

### `xcx pax new <name>`

標準ディレクトリレイアウトで新しいプロジェクトを作成します。

```sh
xcx pax new my_project
```

作成されるファイル：
```
my_project/
├── project.pax
├── src/
│   └── main.xcx
└── README.md
```

---

### `xcx pax install`

`project.pax` を読み込み、すべての `deps` を解決して `lib/` にダウンロードします。すでに存在するパッケージ（名前で判断）はスキップします。

```sh
xcx pax install
```

- 依存関係自身の `files` フィールドで宣言されたファイルのみインストールします。
- 依存関係に `files` フィールドがない場合、PAX は `README.md`、`LICENSE`、`tests/`、`docs/`、`.git*`、`project.pax` を自動除外します。
- 取得に成功するたびに `pax.lock` を更新します。

---

### `xcx pax add <dep>`

`project.pax` に依存関係を追加して即座にインストールを実行します。

```sh
xcx pax add mylib@1.0.0
xcx pax add author/mylib@latest
```

---

### `xcx pax remove <name>`

`project.pax` から依存関係エントリーを削除します。

```sh
xcx pax remove mylib
```

> **注意：** `lib/` フォルダと `pax.lock` エントリーは自動的にクリーンアップされません。`xcx pax install` を再実行するか、手動で削除してください。

---

### `xcx pax clone <package>`

PAX レジストリから公開済みパッケージを完全なローカルプロジェクトとしてダウンロードします。

```sh
xcx pax clone snake_game
xcx pax clone author/snake_game
```

- 現在の作業ディレクトリにパッケージ名のディレクトリを作成します。
- `project.pax`、`src/`、宣言されたすべてのファイルを含む完全なプロジェクト構造をコピーします。
- 依存関係は**自動的にインストールされません**。

**典型的なワークフロー：**
```sh
xcx pax clone snake_game
cd snake_game
xcx pax install
xcx pax run
```

---

### `xcx pax run [path]`

プロジェクトを実行します。エントリーポイントの解決順序：

1. 引数として指定されたパス（例：`xcx pax run other/main.xcx`）
2. `project.pax` の `main` フィールド
3. デフォルト：`src/main.xcx`

```sh
xcx pax run
xcx pax run src/alt.xcx
```

---

### `xcx pax search <query>`

PAX レジストリでクエリに一致するパッケージを検索します。

```sh
xcx pax search json
```

出力形式：
```
- json_utils (v1.2.0) by alice
- fast_json (v0.9.1) by bob
```

---

### `xcx pax login <token>`

レジストリ認証トークンを `.pax_token` にローカル保存します。

```sh
xcx pax login your_token_here
```

---

### `xcx pax logout`

保存された認証トークンを削除します。

```sh
xcx pax logout
```

---

### `xcx pax whoami`

現在のレジストリセッションを確認し、アカウント情報を表示します。

```sh
xcx pax whoami
# Account: alice [developer]
```

---

### `xcx pax publish`

現在のプロジェクトを PAX レジストリに公開します。ログインが必要です。

```sh
xcx pax publish
```

- `project.pax` からメタデータ（`name`、`version`、`description`）を読み込みます。
- すべてのプロジェクトファイルをスキャンしてアップロードします（`.zip`、`.pax_token`、`.git` を除く）。
- 圧縮アーカイブとファイルマニフェストをレジストリに送信します。

> **注意：** 公開前に `project.pax` の `version` を更新してください。レジストリは重複バージョンを拒否する場合があります。

---

## レジストリ設定

PAX はデフォルトで `pax.xcxlang.com` に接続します。プロジェクトルートの `.pax_config` ファイルで上書きできます：

```json
{
  "registry": "https://my-custom-registry.example.com"
}
```

- `localhost` と `127.0.0.1` には自動的に HTTP が使用されます。
- その他すべてのホストには HTTPS が使用されます。
