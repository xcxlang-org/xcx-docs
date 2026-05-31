# XCX 3.1 データベース

XCX 3.1 は `database:` 型と組み込みメソッド群により、ネイティブなリレーショナルデータベースをサポートします。バージョン 3.1 では **SQLite のみ** をサポートします。

---

## 目次

1. [接続の宣言（`database:`）](#1-connection-declaration-database)
2. [テーブル列属性](#2-table-column-attributes)
3. [名前付き引数](#3-named-arguments)
4. [結果の取得（`as`）](#4-result-capture-as)
5. [ファイバースコープの制限（Windows）](#5-fiber-scoping-limitations-windows)
6. [API リファレンス](#6-api-reference)
   - 5.1 [DDL](#51-ddl)
   - 5.2 [書き込み操作](#52-write-operations)
   - 5.3 [読み取り操作](#53-read-operations)
   - 5.4 [削除操作](#54-delete-operations)
   - 5.5 [トランザクション](#55-transactions)
   - 5.6 [その他](#56-other)
6. [型マッピング（XCX ↔ SQL）](#6-type-mapping-xcx--sql)
7. [セキュリティ制約](#7-security-constraints)
8. [完全な例](#8-full-example)

---

<a id="1-connection-declaration-database"></a>

## 1. 接続の宣言（`database:`）

```xcx
database: app {
    engine  = "sqlite",
    path    = "app.db"
};
```

### フィールド

| フィールド      | 型 | 必須 | デフォルト | 説明                                      |
|------------|------|----------|---------|--------------------------------------------------|
| `engine`   | `s`  | はい      | —       | データベースエンジン。現在は `"sqlite"` のみ。      |
| `path`     | `s`  | はい      | —       | `.db` ファイルへのパス（プロジェクトルートからの相対）。|
| `timeout`  | `i`  | いいえ       | `5000`  | 操作タイムアウト（ミリ秒）。                         |
| `readonly` | `b`  | いいえ       | `false` | 読み取り専用モード。                                  |

### 挙動

- 接続は **遅延** — 初回利用時にオープンされます。
- 接続失敗時（権限不足など）は、自動的に `halt.error` が発生します。
- SQLite は、ファイルが存在しなければ作成します（`readonly = true` の場合を除く）。
- 複数の同時接続が可能 — それぞれ独自の名前を持ちます。

```xcx
database: main { engine = "sqlite", path = "main.db" };
database: logs { engine = "sqlite", path = "logs.db" };

yield main.sync(users);
yield logs.sync(events);
```

### I/O モデル

ディスクまたはドライバに触れる DB 操作はすべて I/O 操作です。

| コンテキスト        | 挙動              |
|----------------|-----------------------|
| ファイバー内 | `yield` が必要       |
| ファイバー外| 同期的にブロック   |

これは **読み取り**（`fetch`、`query`、`queryRaw`）、**書き込み**（`push`、`save`、`insert`、`exec`）、**削除**（`remove`、`exec`）、**DDL**（`sync`、`drop`）に適用されます。

**例外 — `yield` 不要のメソッド。** トランザクション状態またはメタデータのみを扱うメソッドは、どのコンテキストでも `yield` は不要です。

- `db.has()`、`db.close()`、`db.isOpen()`
- `db.begin()`、`db.commit()`、`db.rollback()`

> `begin()`、`commit()`、`rollback()` はデータに触れません — ドライバ内のトランザクション状態のみを管理します。

---

<a id="2-table-column-attributes"></a>

## 2. テーブル列属性

これらの属性は `table:` ブロック宣言を拡張し、データベースモジュールで使われます。

### 属性

| 属性       | 説明                                                        | SQL 相当      |
|-----------------|--------------------------------------------------------------------|---------------------|
| `@pk`           | 主キー。`@auto` と組み合わせ可能。                        | `PRIMARY KEY`       |
| `@unique`       | 列内で値が一意でなければならない。                            | `UNIQUE`            |
| `@optional`     | SQL で `NULL` 可。読み取り時 XCX は型のデフォルト値を返す。 | `NULL`        |
| `@default(v)`   | SQL の列デフォルト値。                                       | `DEFAULT v`         |
| `@fk(t.col)`    | テーブル `t` の列 `col` への外部キー。                | `REFERENCES t(col)` |

### 例

```xcx
table: users {
    columns = [
        id      :: i @auto @pk,
        name    :: s @unique,
        age     :: i,
        phone   :: s @optional,
        role    :: s @default("user")
    ]
    rows = [EMPTY]
};

table: posts {
    columns = [
        id      :: i @auto @pk,
        user_id :: i @fk(users.id),
        title   :: s,
        body    :: s @optional
    ]
    rows = [EMPTY]
};
```

### `@optional` の挙動

XCX には独立した型としての `null` はありません。SQL で `NULL` を格納できる `@optional` 列も、XCX に読み込むと型のデフォルトに自動置換されます。

| XCX 型 | SQL が NULL のときの値 |
|----------|---------------------|
| `i`      | `0`                 |
| `f`      | `0.0`               |
| `s`      | `""`                |
| `b`      | `false`             |

> **注:** `@optional` は SQL 側で `NULL` を許容することを示すだけです。XCX は `NULL` とデフォルト値の区別を保持しません — 読み取り時にその情報は失われます。アプリケーションロジックで区別が必要なら、別列にフラグを保存してください。

### `add()` と `insert()` の列省略ルール

| 列属性        | 省略可能？ | SQL に入る値             |
|-------------------------|-----------------|--------------------------------|
| `@auto`                 | はい（常に）    | 自動生成値           |
| `@default(v)`           | はい             | `DEFAULT` の値 `v`       |
| `@optional`             | はい             | `NULL`                         |
| `@optional @default(v)` | はい             | `DEFAULT` の値 `v`       |
| 属性なし           | **いいえ**          | コンパイルエラー                  |

> `@auto` 列は、位置引数でも名前付き引数でも明示的に渡せません。そうしようとすると **コンパイルエラー** です。

---

<a id="3-named-arguments"></a>

## 3. 名前付き引数

名前付き引数では、`table.add()`、`table.insert()`、`db.insert()` に値を **列名で** 渡せます（位置ではなく）。任意の追加機能で、既存の位置引数呼び出しはそのまま有効です。

名前付き引数は **テーブルへの insert 操作のみ** に使えます。ユーザー定義関数、ファイバー、その他の組み込みメソッドには適用されません。列名は `table:` スキーマからコンパイラが静的に把握します。

### 構文

```xcx
--- Positional (backward compatible)
users.add("Alice", 25, "", "user");

--- Named
users.add(name = "Alice", age = 25, phone = "", role = "user");

--- With db.insert()
yield app.insert(users, name = "Alice", age = 25) as saved;
```

**名前空間の分離。** `=` の左辺は常に列名。右辺はローカルスコープの式。これらは独立した名前空間 — 衝突しません。

```xcx
s: name = "Alice";
users.add(name = name, age = 25);
--- left "name"  = column users.name
--- right "name" = local variable
```

**評価順序。** 右辺の式は、記述順に左から右へ評価されます。

```xcx
users.add(name = get_name(), age = get_age());
--- get_name() is called before get_age()
```

### 位置引数と名前付き引数の混在

位置引数と名前付き引数を混在できます。**位置引数は名前付き引数より前に置く必要があります。**

```xcx
--- OK — positional before named
users.add("Alice", age = 25, role = "admin");

--- COMPILE ERROR — named before positional
users.add(name = "Alice", 25, "admin");
```

**割り当てルール。** 位置引数は宣言順に左から右へ列に割り当てられ、**`@auto` はスキップ** され、尽きるまで続きます。名前付き引数は名前で残りの列を埋めます。

```xcx
--- "Alice" → name, rest named
users.add("Alice", age = 25, role = "admin");
--- phone omitted → NULL (@optional)

--- "Alice", 25 → name, age; role named, phone omitted
users.add("Alice", 25, role = "mod");
```

位置引数は **スキップせず** 順に割り当てられます。中間列を飛ばすには、名前付き引数を使うか、省略可能な列として省略するしかありません。

```xcx
--- COMPILE ERROR — compiler assigns: name="Alice", age="" — type mismatch
--- there is no way to "skip" age positionally
users.add("Alice", "", "user");
```

同じ列を位置と名前の両方で渡すと **コンパイルエラー** です。

```xcx
--- COMPILE ERROR — name provided twice
users.add("Alice", name = "Bob", age = 25);
```

### コンパイル時ルール

| ルール | 挙動 |
|------|----------|
| `@auto` 列を指定 | コンパイルエラー |
| 未知の列名 | コンパイルエラー |
| 列名の重複 | コンパイルエラー |
| 必須列の省略 | コンパイルエラー |
| 位置引数より前の名前付き引数 | コンパイルエラー |

**完全性テーブル:**

| 列属性        | 省略可能？ | 省略時の値                   |
|-------------------------|-----------------|--------------------------------------|
| `@auto`                 | はい（常に）    | 自動生成値                 |
| `@default(v)`           | はい             | `DEFAULT` の値 `v`             |
| `@optional`             | はい             | `NULL`（読み取り時は XCX デフォルト）       |
| `@optional @default(v)` | はい             | `DEFAULT` の値 `v`             |
| 属性なし           | **いいえ**          | コンパイルエラー                        |

```xcx
--- OK — role has @default("user"), phone is @optional
users.add(name = "Alice", age = 25);

--- COMPILE ERROR — age is required
users.add(name = "Alice");
```

---

<a id="4-result-capture-as"></a>

## 4. 結果の取得（`as`）

`as` はブロック操作の結果を取得する一般的な XCX の仕組みです。

```xcx
--- HTTP request
net.request { method = "GET", url = "..." } as resp;

--- DB operations
yield app.insert(users, name = "Alice", age = 25) as saved;
yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
```

### 書き込み／削除の結果オブジェクト

`db.insert()`、`db.save()`、`db.push()`、`db.exec()` は 2 フィールドを持つオブジェクトを返します。

| フィールド      | 型 | 説明                                                         |
|------------|------|---------------------------------------------------------------------|
| `affected` | `i`  | 操作で変更／挿入された行数                  |
| `insertId` | `i`  | 最後に挿入されたレコードの ID（`@auto @pk`）。挿入がなければ `0`      |

```xcx
yield app.insert(users, name = "Alice", age = 25) as saved;
>! "New user ID: " + s(saved.insertId);

yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
>! "Deleted: " + s(res.affected);
```

`as` は任意です — 結果が不要なら省略できます。

```xcx
yield app.save(users);
```

### `affected` と `insertId` の意味

| 操作                  | `affected`                      | `insertId`                        |
|----------------------------|---------------------------------|-----------------------------------|
| `insert()` — 1 行    | `1`                             | 挿入行の ID                |
| `push()` — 複数行   | 挿入行数         | 最後に挿入された行の ID           |
| `save()` — upsert（insert） | `1`                             | 挿入行の ID                |
| `save()` — upsert（update） | `1`                             | `0`                               |
| `exec(DELETE ...)`         | 削除行数          | `0`                               |
| `exec(UPDATE ...)`         | 更新行数          | `0`                               |

> 操作失敗時（制約違反、接続なし、`readonly`）は `halt.error` が発生し、`as` 結果を使うコードには到達しません。コードが結果オブジェクトを見られる場合、それは常に有効です。

---

<a id="5-fiber-scoping-limitations-windows"></a>

## 5. ファイバースコープの制限（Windows）

XCX 3.1 では、特に Windows 環境で、データベース結果でファイバーローカル変数を直接初期化すると、後続行で `Undefined variable` [S101] が発生することがあります。

### 推奨ワークアラウンド

ファイバー内の条件ブロック間で確実に可視にするには、**先に変数の型を宣言** し、別ステートメントで DB 結果を代入することを推奨します。

```xcx
fiber run(-> b) {
    --- Recommended pattern
    b: exists_before;
    exists_before = db.has(logs);
    
    if (exists_before) then;
        >! "Table exists";
    end;
}
```

このパターンでは、解析器がアクセスする前に変数がシンボルテーブルに正しく登録されます。この制限は XCX 4.0 でアーキテクチャ上ネイティブに解消する予定です。

---

<a id="6-api-reference"></a>

## 6. API リファレンス

<a id="51-ddl"></a>

### 5.1 DDL

```xcx
--- Creates the SQL table if it does not exist (based on XCX table schema)
yield app.sync(users);

--- Drops the SQL table
yield app.drop(users);

--- Checks if table exists in the database (no data touched — no yield)
b: exists = app.has(users);
```

<a id="52-write-operations"></a>

### 5.2 書き込み操作

```xcx
--- INSERT all rows from a local XCX table into SQL
yield app.push(users);

--- INSERT OR UPDATE (upsert) — requires @pk; no @pk = compile error
--- Implementation: INSERT INTO ... ON CONFLICT(pk) DO UPDATE SET ...
yield app.save(users);

--- INSERT a single row — positional or named
yield app.insert(users, "Alice", 25) as saved;
yield app.insert(users, name = "Alice", age = 25) as saved;
```

エラー時（`@unique`、`@fk` 違反、接続なし）は `halt.error` が発生します。

<a id="53-read-operations"></a>

### 5.3 読み取り操作

```xcx
--- SELECT * — returns a native XCX table
table: all_users = yield app.fetch(users);

--- SELECT with XCX-style filter (compiles to SQL WHERE)
table: adults = yield app.fetch(users).where(age > 18);

--- SELECT with raw SQL — requires schema hint as first argument
table: result = yield app.query(users, "SELECT * FROM users WHERE age > ?", [18]);

--- Raw SQL without schema — always returns a JSON array
json: raw = yield app.queryRaw("SELECT COUNT(*) as n FROM users");
--- raw = [{"n": 42}]

--- Get the first row
json: row = yield app.queryRaw("SELECT COUNT(*) as n FROM users").first();
--- row = {"n": 42}
--- halt.error if result is empty
```

#### `queryRaw` 結果での `.first()`

`.first()` は JSON 配列の先頭要素を単一の `json` オブジェクトとして返します。行が 0 件なら `halt.error` です。集約クエリ（`COUNT`、`SUM`、`MAX`）や最大 1 行のルックアップでよく使います。

```xcx
json: row = yield app.queryRaw("SELECT COUNT(*) as n FROM users").first();
i: total; row.bind("n", total);
>! "Users: " + s(total);
```

結果が空でないと確信できない場合は、`.first()` の前にサイズを確認してください。

```xcx
json: results = yield app.queryRaw("SELECT * FROM users WHERE age > ?", [100]);
if (results.size() > 0) then;
    json: row = results.get(0);
end;
```

#### `.where()` でサポートされる演算子

`.where()` フィルタは SQL の `WHERE` 句にコンパイルされます。XCX 式の限られた部分集合のみサポートされます。

| カテゴリ      | サポート                        |
|---------------|----------------------------------|
| 比較   | `==`, `!=`, `<`, `<=`, `>`, `>=` |
| 論理       | `AND`, `OR`, `NOT`               |
| 値        | リテラル、ローカル変数        |
| 左オペランド | 列名のみ                |

関数呼び出し、文字列メソッド、ネストした式は `db.fetch()` の `.where()` ではサポートされません。未サポートの式は **コンパイルエラー** です。

```xcx
--- OK
table: r = yield app.fetch(users).where(age > 18 AND role == "admin");

--- COMPILE ERROR — string method not allowed in where() with fetch
table: r = yield app.fetch(users).where(name.lower() == "alice");
```

`.where()` の連鎖は条件を `AND` で結合します。

```xcx
table: r = yield app.fetch(users)
    .where(age > 18)
    .where(role == "admin");
--- SQL equivalent: WHERE age > 18 AND role = 'admin'
```

#### `db.fetch()` とローカル `rows`

`db.fetch()` は SQL データベースのデータのみを読みます — `table:` ブロックで宣言したローカル `rows` は無視されます。ローカルテーブル定義はスキーマヒントとしてのみ機能します。

```xcx
table: users {
    columns = [ id :: i @auto @pk, name :: s ]
    rows = [ ("Alice") ]   --- these rows do NOT go into fetch
};

yield app.sync(users);

--- Returns only rows stored in app.db
table: all = yield app.fetch(users);
```

ローカル `rows` を DB に入れるには、`db.fetch()` の前に `db.push(users)` を使います。

<a id="54-delete-operations"></a>

### 5.4 削除操作

```xcx
--- DELETE with filter — .where() is required (D401)
yield app.remove(users).where(age < 18);

--- Delete all rows
yield app.truncate(users);

--- Raw DELETE with parameters
yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
```

> **`db.remove()` に `.where()` がないとコンパイルエラー（D401）。** 全行削除には明示的に `db.truncate(users)` を使います。

<a id="55-transactions"></a>

### 5.5 トランザクション

```xcx
app.begin();
yield app.save(users);
yield app.save(posts);
app.commit();
```

> トランザクション内のいずれかの操作が `halt.error` を起こすと、フレームが中止される前にトランザクションは **自動的にロールバック** されます。接続は `begin()` 前の状態に戻ります。

`rollback()` は、エラーが起きる前に条件分岐で明示的にトランザクションを取り消すためのものです。

```xcx
app.begin();
b: valid = validate_users(users);
if (NOT valid) then;
    app.rollback();
end;
yield app.save(users);
app.commit();
```

<a id="56-other"></a>

### 5.6 その他

```xcx
app.close();
b: alive = app.isOpen();
```

---

<a id="6-type-mapping-xcx--sql"></a>

## 6. 型マッピング（XCX ↔ SQL）

| XCX 型 | SQL 型（SQLite）              |
|----------|--------------------------------|
| `i`      | `INTEGER`                      |
| `f`      | `REAL`                         |
| `s`      | `TEXT`                         |
| `b`      | `INTEGER` (0 / 1)              |
| `date`   | `TEXT` (`YYYY-MM-DD HH:mm:ss`) |

---

<a id="7-security-constraints"></a>

## 7. セキュリティ制約

| 制約                          | 挙動                                                   |
|-------------------------------------|------------------------------------------------------------|
| パスに `..` を含む                | `halt.fatal` — パストラバーサル                              |
| 絶対パス                       | `halt.fatal`                                               |
| 未検証入力を含む生 SQL      | サニタイズは開発者の責任 — `?` パラメータを使う |
| `readonly = true` + DML 操作   | `halt.error`                                               |
| `@pk` のないテーブルでの `save()`     | コンパイルエラー                                              |
| `.where()` のない `remove()`       | コンパイルエラー（D401）                                       |

常に `?` 付きのプリペアドステートメントを使ってください。

```xcx
--- WRONG
app.exec("DELETE FROM users WHERE name = '" + name + "'");

--- CORRECT
app.exec("DELETE FROM users WHERE name = ?", [name]);
```

---

<a id="8-full-example"></a>

## 8. 完全な例

```xcx
database: app {
    engine = "sqlite",
    path   = "data.db"
};

table: users {
    columns = [
        id    :: i @auto @pk,
        name  :: s @unique,
        email :: s @unique,
        age   :: i,
        phone :: s @optional
    ]
    rows = [EMPTY]
};

yield app.sync(users);

fiber handle_get_users(json: req -> json) {
    table: all = yield app.fetch(users);
    yield net.respond(200, all.toJson());
};

fiber handle_create_user(json: req -> json) {
    json: body;
    req.bind("body", body);

    s: name;  body.bind("name", name);
    s: email; body.bind("email", email);
    i: age;   body.bind("age", age);

    --- Named arguments — readable and order-independent
    yield app.insert(users, name = name, email = email, age = age) as saved;

    json: resp <<< {"ok": true, "id": 0} >>>;
    resp.set("id", saved.insertId);
    yield net.respond(201, resp);
};

serve: api {
    port   = 8080,
    routes = [
        "GET  /users" :: handle_get_users,
        "POST /users" :: handle_create_user
    ]
};
```
