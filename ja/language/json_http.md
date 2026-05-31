# XCX 3.1 JSON と HTTP

## JSON

XCX 3.1 の JSON オブジェクトは **可変（mutable）** です。

### 作成

```xcx
json: config <<< {"port": 8080, "debug": false} >>>;
json: user   <<< {"name": "", "age": 0} >>>;
```

リテラル内の値はプレースホルダ（`""`、`0`、`false`）にでき、後から `.set()` で埋められます。

### コレクションからのシリアライズ

`.toJson()` メソッドを使うと、XCX のコレクションから直接 JSON オブジェクトや配列を作成できます。次で利用できます。

- **Map**: JSON オブジェクトを返します。
- **Table**: オブジェクトの JSON 配列を返します。

マッピングと挙動の詳細は [コレクションのドキュメント](collections.md) を参照してください。

### パース

```xcx
json: parsed = json.parse(raw_string);
```

> [!CAUTION]
> **無効な JSON でのパニック（R305）**: パースに失敗すると、VM は直ちに終了します。パース前に文字列の内容を確認してください。

### 可変性のパターン

ゼロ値でスキーマを宣言し、その後値を設定します。

```xcx
json: resp <<< {"token": "", "role": "", "uid": 0, "ok": false} >>>;
resp.set("token", crypto.token(32));
resp.set("role",  "admin");
resp.set("uid",   42);
resp.set("ok",    true);
yield net.respond(200, resp);
```

### JSON メソッド

| メソッド                    | シグネチャ             | 戻り値 | 説明                                                    |
|---------------------------|-----------------------|---------|----------------------------------------------------------------|
| `.exists(path)`           | `(s) → b`             | `b`     | パスが存在し null でないか確認                          |
| `.get(path/idx)`          | `(s/i) → json`        | `json`  | パスまたはインデックスの要素を取得                                  |
| `.bind(path, var)`        | `(s, ref) → b`        | `b`     | 値を **事前に宣言した** XCX 変数に取り出す            |
| `.set(path, val)`         | `(s, T) → b`          | `b`     | パスの値を設定。キーがなければ作成                     |
| `.push(val)`              | `(json) → b`          | `b`     | JSON 配列ノードに要素を追加                           |
| `.size()` / `.count()`    | `() → i`              | `i`     | キー数（オブジェクト）または要素数（配列）                    |
| `.toStr()`                | `() → s`              | `s`     | JSON 文字列にシリアライズ                                      |
| `.inject(path, map, tbl)` | `(s, map, table) → b` | `b`     | JSON 配列を XCX テーブルに一括インポート                       |
| `.first()`                | `() → json`           | `json`  | JSON 配列の先頭要素を返す。空の場合は halt.error |

> [!NOTE]
> **JSON 配列での `.push()`**: `.push()` は配列（`[]`）の JSON ノードにのみ使えます。JSON オブジェクト（`{}`）に `.push()` を呼ぶと `halt.error` になります。
>
> ```xcx
> json: data <<< {"items": []} >>>;
> json: obj <<< {"id": 1} >>>;
> data.get("items").push(obj);   --- OK: "items" is an array
> data.push(obj);                --- halt.error: data is an object, not an array
> ```

> [!IMPORTANT]
> **`.bind()` の構文**: 第 2 引数は **事前に宣言した変数** でなければなりません。型をインラインで宣言することはできません。
>
> ```xcx
> --- Wrong
> req.bind("ip", s: ip);
>
> --- Correct
> s: ip;
> req.bind("ip", ip);
> ```

### パス表記

ネストしたアクセスにはドット表記とブラケット表記の両方が使えます。

```xcx
--- Nested field
json: cfg <<< {"server": {"host": "localhost"}} >>>;
s: host;
cfg.bind("server.host", host);

--- Array index
json: resp <<< {"items": []} >>>;
resp.set("items[0]", first_item);
resp.set("items[1]", second_item);
```

### .inject() — 一括インポート

JSON 配列を XCX テーブルに直接インポートします。

```xcx
json: data <<< {"users":[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]} >>>;
table: imported { columns=[uid::i, uname::s] rows=[EMPTY] };
map: mapping { schema=[s<->s] data=["uid"::"id", "uname"::"name"] };
data.inject("users", mapping, imported);
imported.show();
```

### HTTP リクエスト内の JSON

HTTP ハンドラ内では、`json: req` は次の構造を持ちます。

```json
{
    "method":  "POST",
    "path":    "/api/login",
    "query":   { "page": "1" },
    "headers": { "authorization": "Bearer ..." },
    "body":    { ... },
    "ip":      "1.2.3.4"
}
```

```xcx
s: ip;
req.bind("ip", ip);

json: body;
req.bind("body", body);

json: headers;
req.bind("headers", headers);

s: auth;
headers.bind("authorization", auth);
```

---

## HTTP クライアント

### 高レベル API

| メソッド                 | シグネチャ          | 戻り値 |
|------------------------|--------------------|---------|
| `net.get(url)`         | `(s) → json`       | `json`  |
| `net.post(url, body)`  | `(s, json) → json` | `json`  |
| `net.put(url, body)`   | `(s, json) → json` | `json`  |
| `net.delete(url)`      | `(s) → json`       | `json`  |

```xcx
json: resp = net.get("https://api.example.com/users");

json: body <<< {"name": "Alice"} >>>;
json: resp = net.post("https://api.example.com/users", body);
```

### レスポンスオブジェクト

```json
{
    "status":  200,
    "ok":      true,
    "body":    { ... },
    "headers": { "content-type": "application/json" },
    "text":    "...",
    "error":   "..."
}
```

| フィールド     | 型   | 説明                                                       |
|-----------|--------|-------------------------------------------------------------------|
| `status`  | `i`    | HTTP ステータスコード                                                  |
| `ok`      | `b`    | ステータスが 200 以上 300 未満のとき `true`                               |
| `body`    | `json` | パース済みレスポンス本体（JSON の場合）。それ以外は `null`                  |
| `headers` | `json` | レスポンスヘッダ                                                  |
| `text`    | `s`    | 生の文字列としてのレスポンス本体（常に利用可能）                    |
| `error`   | `s`    | リクエスト失敗時のエラーメッセージ。成功時は空文字列               |

`ok` はステータスが 200 以上 300 未満のとき `true` です。`body` にアクセスする前に必ず `ok` を確認してください。

```xcx
json: resp = net.get("https://api.example.com/data");
if (resp.ok) then;
    --- working with resp.body
else;
    s: err;
    if (resp.exists("error")) then;
        resp.bind("error", err);
    end;
    >! "Error: " + s(resp.get("status")) + " | " + err;
end;
```

### 低レベルビルダー（`net.request`）

```xcx
net.request {
    method  = "POST",
    url     = "https://api.example.com/data",
    headers = ["Authorization" :: "Bearer xyz", "X-App" :: "XCX"],
    body    = my_json,
    timeout = 5000
} as resp;
```

| フィールド     | 型        | 必須 | デフォルト  | 説明                           |
|-----------|-------------|----------|----------|---------------------------------------|
| `method`  | `s`         | はい      | —        | `"GET"`、`"POST"`、`"PUT"` など      |
| `url`     | `s`         | はい      | —        | スキーム付きの完全な URL                  |
| `headers` | `map:s<->s` | いいえ       | `{}`     | 追加のリクエストヘッダ            |
| `body`    | `json`      | いいえ       | `null`   | GET と DELETE では無視            |
| `timeout` | `i`         | いいえ       | `10000`  | ミリ秒                          |

---

## HTTP サーバー

### serve ディレクティブ

```xcx
serve: app {
    port    = 8080,
    host    = "0.0.0.0",
    workers = 4,
    routes  = [
        "POST   /api/login"     :: handle_login,
        "GET    /api/user"      :: handle_user,
        "DELETE /api/users"     :: handle_delete,
        "OPTIONS *"             :: handle_options,
        "*"                     :: handle_404
    ]
};
```

| フィールド     | 型         | 必須 | デフォルト       | 説明                               |
|-----------|--------------|----------|---------------|-------------------------------------------|
| `port`    | `i`          | はい      | —             | ポート番号（1–65535）                     |
| `host`    | `s`          | いいえ       | `"127.0.0.1"` | `"0.0.0.0"` = すべてのインターフェース              |
| `workers` | `i`          | いいえ       | `1`           | リクエストあたりのファイバー                        |
| `routes`  | route list   | はい      | —             | 上から順に照合。最初に一致したルートが採用   |

> [!NOTE]
> `serve:` は **終端ステートメント** です。このディレクティブの後のコードは実行されません。
> ワイルドカード `*` は任意のメソッド・パスに一致します — 常に最後に置いてください。

### ハンドラファイバー

各ハンドラは `fiber name(json: req -> json)` のシグネチャを持つファイバーでなければなりません。

```xcx
fiber handle_health(json: req -> json) {
    yield net.respond(200, <<< {"status": "ok"} >>>);
};
```

`yield net.respond(...)` を呼ばないハンドラは、自動的に `500 Internal Server Error` になります。

### net.respond()

```xcx
yield net.respond(200, my_json);
yield net.respond(201, my_json, ["Location" :: "/users/42"]);
yield net.respond(204, <<< {} >>>);
yield net.respond(404, <<< {"error": "not found"} >>>);
```

| パラメータ | 型        | 必須 | 説明                          |
|-----------|-------------|----------|--------------------------------------|
| status    | `i`         | はい      | HTTP ステータスコード                     |
| body      | `json \| s` | はい      | JSON オブジェクトまたは生文字列。空 JSON は `<<< {} >>>` |
| headers   | `map:s<->s` | いいえ       | 追加のレスポンスヘッダ          |

### CORS とプリフライト

既定では、XCX エンジンはレスポンスに次のヘッダが無い場合、自動的に CORS サポートを付与します。

- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

> [!TIP]
> **推奨プラクティス**: エンジンがデフォルトを提供しますが、これらのヘッダをコードで **明示的に宣言することを強く推奨** します。環境が変わってもアプリケーションの挙動が予測可能になり、セキュリティポリシーが明確に文書化されます。

プリフライト（`OPTIONS`）には、許可するメソッドとヘッダを明示する専用ハンドラを使います。

```xcx
fiber handle_options(json: req -> json) {
    yield net.respond(204, <<< {} >>>, [
        {
            "Access-Control-Allow-Methods": "GET, POST, DELETE, PATCH, OPTIONS",
            "Access-Control-Allow-Headers": "Content-Type, Authorization, X-CSRF-TOKEN"
        }
    ]);
};
```

### 同時リクエスト

ファイバーにより I/O を重ねられます。

```xcx
fiber fetch(s: url -> json) {
    yield net.get(url);
};

fiber:json: f1 = fetch("https://api.example.com/users");
fiber:json: f2 = fetch("https://api.example.com/posts");
json: r1 = f1.next();
json: r2 = f2.next();
```

### セキュリティ制約

| 制約                      | 挙動                                           |
|---------------------------------|----------------------------------------------------|
| `localhost` / `127.0.0.1`       | 既定で許可                                 |
| `169.254.x.x`（リンクローカル）      | `halt.fatal` — SSRF 保護                     |
| `10.x`、`172.16.x`、`192.168.x` | 本番モードではブロック                         |
| `file://` URL                  | `halt.fatal`                                       |
| レスポンス本体の最大サイズ          | 10 MB                                              |
| 受信リクエスト本体の最大サイズ       | 10 MB — ハンドラを呼ばず 413 を返す   |
