# XCX 3.1 標準ライブラリとモジュール

## 組み込みモジュール

### crypto

暗号ユーティリティ。

| メソッド                              | 戻り値 | 説明                                    |
|-------------------------------------|---------|------------------------------------------------|
| `crypto.hash(data, "bcrypt")`       | `s`     | bcrypt でパスワードをハッシュ                   |
| `crypto.hash(data, "argon2")`       | `s`     | argon2 でパスワードをハッシュ（推奨）     |
| `crypto.hash(data, "base64_encode")`| `s`     | バイナリデータ／文字列を Base64 文字列にエンコード    |
| `crypto.hash(data, "base64_decode")`| `s`     | Base64 文字列をバイナリ／文字列にデコード    |
| `crypto.verify(password, hash, algo)` | `b`   | パスワードがハッシュと一致すれば `true`        |
| `crypto.token(length)`              | `s`     | 指定長のランダム 16 進トークンを生成     |

```xcx
s: hash  = crypto.hash(password, "bcrypt");
s: hash2 = crypto.hash(password, "argon2");
b: valid = crypto.verify(password, hash2, "argon2");
s: token = crypto.token(32);
```

### store（ファイル I/O）

すべてのパスはプロジェクトルートからの **相対パス** でなければなりません。絶対パスやパストラバーサル（`..`）は `halt.fatal` を発生させます。

| メソッド              | シグネチャ                 | 戻り値 | 説明                                  |
|---------------------|---------------------------|---------|----------------------------------------------|
| `store.write(p, c)` | `(s, s) → b`              | `b`     | ファイルを上書き。必要ならディレクトリを作成。 |
| `store.read(p)`     | `(s) → s`                 | `s`     | ファイル内容を返す。無い場合は `halt.fatal`。 |
| `store.append(p, c)`| `(s, s) → b`              | `b`     | ファイルに追記。無ければ作成。         |
| `store.exists(p)`   | `(s) → b`                 | `b`     | 存在確認。副作用なし。           |
| `store.delete(p)`   | `(s) → b`                 | `b`     | ファイルまたはディレクトリを削除（再帰）。       |
| `store.list(p)`     | `(s) → array:s`           | `array:s`| ファイルとフォルダの一覧を返す。          |
| `store.isDir(p)`    | `(s) → b`                 | `b`     | パスがディレクトリなら `true`。               |
| `store.size(p)`     | `(s) → i`                 | `i`     | ファイルサイズ（バイト）。                          |
| `store.mkdir(p)`    | `(s) → b`                 | `b`     | ディレクトリを作成（再帰）。               |
| `store.glob(pat)`   | `(s) → array:s`           | `array:s`| グロブパターンに一致するファイルを返す。         |
| `store.zip(s, t)`   | `(s, s) → b`              | `b`     | ソースをターゲット zip にアーカイブ。               |
| `store.unzip(z, d)` | `(s, s) → b`              | `b`     | zip を展開先に解凍。                 |

```xcx
store.write("log.txt", "First line");
store.append("log.txt", "\nSecond line");
s: content = store.read("log.txt");
if (store.exists("lock.pid")) then;
    >! "Already running";
end;
```

### env

| メソッド          | シグネチャ      | 戻り値    | 説明                                               |
|-----------------|----------------|------------|-----------------------------------------------------------|
| `env.get(name)` | `(s) → s`      | `s`        | 環境変数の値を返す。未設定なら `halt.error`       |
| `env.args()`    | `() → array:s` | `array:s`  | プログラムに渡された CLI 引数を配列で返す   |

```xcx
s: db_url = env.get("DATABASE_URL");

array:s: args = env.args();
for arg in args do;
    >! arg;
end;
```

### random

| メソッド                            | シグネチャ           | 戻り値 | 説明                                                                 |
|-----------------------------------|---------------------|---------|-----------------------------------------------------------------------------|
| `random.choice from col`          | `(set/array) → T`   | `T`     | 指定した set または array からランダムに 1 要素を選ぶ。                      |
| `random.int(min, max @step num)`  | `(i, i, @i) → i`    | `i`     | 範囲 `[min, max]` のランダム整数。既定の step: `1`。            |
| `random.float(min, max @step num)`| `(f, f, @f) → f`    | `f`     | 範囲 `[min, max]` のランダム浮動小数。既定の step: `0.5`。            |


```xcx
set:N: pool {1,,10};
i: picked = random.choice from pool;

--- Works on arrays too
array:s: words {"hello", "world"};
s: w = random.choice from words;

--- Picks 1, 3, 5, 7, or 9
i: odd = random.int(1, 10 @step 2);

--- Picks 0.0, 0.5, 1.0, 1.5, or 2.0
f: weight = random.float(0.0, 2.0);

--- Picks 0.0, 0.25, 0.5, 0.75, 1.0
f: precision = random.float(0.0, 1.0 @step 0.25);
```

### date（モジュール）

```xcx
date: now = date.now();
```

---

## モジュールシステム

### include

別ファイルのコードを現在の名前空間にマージします。

```xcx
include "utils.xcx";
include "math.xcx" as m;

m.PI;
m.sqrt(16.0);
```

エイリアスなし — すべてのシンボルが現在の名前空間で直接使えます。エイリアスあり — トップレベルのシンボルはすべて `alias.symbol` 形式になります。

循環依存はコンパイル時に検出され、拒否されます。
