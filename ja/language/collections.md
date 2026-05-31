# XCX 3.1 コレクション

## 配列

```xcx
array:i: nums {10, 20, 30};
nums.size();           --- 3
nums.get(0);           --- 10
nums.push(40);         --- adds 40 to the end
i: last = nums.pop();  --- removes and returns last element
nums.sort();           --- sorts in-place
nums.reverse();        --- reverses in-place
nums.show();           --- prints contents to terminal
```

### 配列メソッド

| メソッド          | シグネチャ   | 戻り値 | 説明                                                                  |
|-------------------|--------------|--------|-----------------------------------------------------------------------|
| `.size()`         | `() → i`     | `i`    | 要素数                                                                |
| `.get(i)`         | `(i) → T`    | `T`    | 位置 `i` の要素（0 始まり）；範囲外なら `halt.error`                  |
| `.push(val)`      | `(T) → b`    | `b`    | 末尾に要素を追加                                                      |
| `.pop()`          | `() → T`     | `T`    | 末尾の要素を削除して返す                                              |
| `.insert(i, val)` | `(i, T) → b` | `b`    | 位置 `i` に挿入し、以降をシフト；範囲外なら `halt.error`              |
| `.update(i, val)` | `(i, T) → b` | `b`    | 位置 `i` の要素を上書き；範囲外なら `halt.error`                      |
| `.delete(i)`      | `(i) → b`    | `b`    | 位置 `i` の要素を削除；範囲外なら `halt.error`                        |
| `.find(val)`      | `(T) → i`    | `i`    | 最初に出現するインデックス、見つからなければ `-1`                     |
| `.contains(val)`  | `(T) → b`    | `b`    | 値が存在するか確認                                                    |
| `.isEmpty()`      | `() → b`     | `b`    | 空なら `true`                                                         |
| `.clear()`        | `() → b`     | `b`    | すべての要素を削除                                                    |
| `.sort()`         | `() → b`     | `b`    | 昇順にソート（インプレース）                                          |
| `.reverse()`      | `() → b`     | `b`    | 順序を反転（インプレース）                                            |
| `.toStr()`        | `() → s`     | `s`    | 配列を JSON 形式の文字列にシリアライズ                                |
| `.toJson()`       | `() → json`  | `json` | 配列をネイティブ JSON 構造に変換                                      |
| `.show()`         | `() → b`     | `b`    | 内容をターミナルに出力                                                |

```xcx
array:i: nums {5, 2, 8, 1};
nums.sort();            --- {1, 2, 5, 8}
nums.reverse();         --- {8, 5, 2, 1}
nums.push(99);          --- {8, 5, 2, 1, 99}
i: last = nums.pop();   --- last = 99, nums = {8, 5, 2, 1}
nums.insert(1, 15);     --- inserts 15 at position 1
nums.update(0, 5);      --- sets element 0 to 5
nums.delete(3);         --- removes element at position 3
b: found = nums.contains(5);
i: idx   = nums.find(5);
b: empty = nums.isEmpty();
```

---

## 集合

### ドメイン

| 記号 | 型                 | 例                            |
|------|--------------------|-------------------------------|
| `N`  | 自然数（≥ 0）      | `set:N: s {0, 1, 2}`          |
| `Z`  | 整数               | `set:Z: s {-3, 0, 3}`         |
| `Q`  | 有理数（浮動小数点） | `set:Q: s {0.5, 1.0}`       |
| `S`  | 文字列             | `set:S: s {"a", "b"}`         |
| `B`  | 真偽値             | `set:B: s {true, false}`      |
| `C`  | 文字               | `set:C: s {"A",,"Z"}`         |

### 初期化

集合は明示的な値または範囲で初期化できます。範囲は両端とも**包含的**です。

```xcx
set:N: small  {1,,5};                  --- {1, 2, 3, 4, 5}
set:N: evens  {0,,100 @step 2};        --- {0, 2, 4, ...}
set:Q: thirds {0.0,,1.0 @step 0.33};
set:C: letters {"A",,"Z"};            --- all uppercase letters
```

集合は要素を**自動的に重複排除**します。

### 集合演算

```xcx
set:N: setA {1,,5};
set:N: setB {3,,7};

set:N: u  = setA UNION setB;
set:N: i  = setA INTERSECTION setB;
set:N: d  = setA DIFFERENCE setB;
set:N: sd = setA SYMMETRIC_DIFFERENCE setB;

--- Unicode symbols are equivalent
setA ∪ setB
setA ∩ setB
setA \ setB
setA ⊕ setB
```

### 集合メソッド

| メソッド       | シグネチャ | 戻り値 | 説明                                       |
|----------------|------------|--------|--------------------------------------------|
| `.size()`      | `() → i`   | `i`    | 要素数                                     |
| `.isEmpty()`   | `() → b`   | `b`    | 空なら `true`                              |
| `.contains(v)` | `(T) → b`  | `b`    | 所属を確認                                 |
| `.add(v)`      | `(T) → b`  | `b`    | 要素を追加（重複は無視）                   |
| `.remove(v)`   | `(T) → b`  | `b`    | 要素を削除（存在しなければ何もしない）     |
| `.clear()`     | `() → b`   | `b`    | すべての要素を削除                         |
| `.show()`      | `() → b`   | `b`    | `{elem, elem, ...}` 形式でターミナルに出力 |

### ランダム選択と反復

```xcx
--- Random selection from a set:
i: picked_set = random.choice from small;

--- Random selection from an array:
array:i: nums {1, 2, 3, 4, 5};
i: picked_arr = random.choice from nums;
```

空の集合または配列から選択すると `false` が返されます。

### 反復

```xcx
for p in small do;
    >! p;
end;
```

---

## マップ

```xcx
map: ages {
    schema = [s <-> i]
    data = [ "alice" :: 30, "bob" :: 25 ]
};

--- Empty Map
map: scores {
    schema = [s <-> i]   --- both separators are equivalent (<-> and <=>)
    data = [EMPTY]
};
```

### マップメソッド

| メソッド         | シグネチャ      | 戻り値    | 説明                                       |
|------------------|-----------------|-----------|--------------------------------------------|
| `.size()`        | `() → i`        | `i`       | キー・値ペアの数                           |
| `.get(key)`      | `(K) → V`       | `V`       | 値を返す；キーがなければ `halt.error`      |
| `.contains(key)` | `(K) → b`       | `b`       | キーが存在するか確認                       |
| `.insert(k, v)`  | `(K, V) → b`    | `b`       | 挿入または上書き                           |
| `.remove(key)`   | `(K) → b`       | `b`       | ペアを削除；キーがなければ `false`         |
| `.keys()`        | `() → array:K`  | `array:K` | キーの配列を返す                           |
| `.values()`      | `() → array:V`  | `array:V` | 値の配列を返す                             |
| `.clear()`       | `() → b`        | `b`       | すべてのペアを削除                         |
| `.toStr()`       | `() → s`        | `s`       | マップを JSON 形式の文字列にシリアライズ   |
| `.show()`        | `() → b`        | `b`       | マップの内容をターミナルに出力             |
| `.toJson()`      | `() → json`     | `json`    | マップを JSON オブジェクトにシリアライズ   |

結果の JSON オブジェクトでは、マップのキーは文字列に変換されます。

### マップのシリアライズ（toJson）

#### シグネチャ
`.toJson() → json`

#### 説明
マップを JSON オブジェクトにシリアライズします。JSON オブジェクトのキー要件を満たすため、すべてのキーは `.toString()` 表現を用いて文字列に変換されます。

```xcx
map: scores {
    schema = [s <-> i]
    data = [ "alice" :: 100, "bob" :: 85 ]
};
json: j = scores.toJson();
```

**JSON 出力：**
```json
{
    "alice": 100,
    "bob": 85
}
```

#### 特殊ケースの挙動
- **空のマップ**：空オブジェクト `{}` を返します。
- **ネスト構造**：マップに配列、他のマップ、テーブルが含まれる場合、それらは再帰的に JSON 相当物にシリアライズされます。
- **キー変換**：すべてのマップキーは、標準の `.toString()` 表現を用いて結果の JSON オブジェクト内で文字列に変換されます。

#### 型マッピング
| XCX 型   | JSON 型   |
|----------|-----------|
| `i`      | `number`  |
| `f`      | `number`  |
| `s`      | `string`  |
| `b`      | `boolean` |
| `date`   | `string`（形式 `"YYYY-MM-DD HH:mm:ss"`） |
| `json`   | （変更なし） |

`.get()` の前に必ず `.contains()` を使用してください：

```xcx
if (ages.contains("alice")) then;
    >! ages.get("alice");
end;
```

---

## テーブル

オートインクリメント列を持てるリレーショナルデータ構造です。

```xcx
table: products {
    columns = [ id :: i @auto, name :: s, price :: f ]
    rows = [ ("Laptop", 2999.99), ("Phone", 1499.50) ]
};

--- Empty Table
table: logs {
    columns = [ id :: i @auto, msg :: s ]
    rows = [EMPTY]
};
```

`i` 列の `@auto` 修飾子はオートインクリメント ID を作成します — `.insert()` と `.add()` ではスキップされます。

> [!NOTE]
> 追加の列属性（`@pk`、`@unique`、`@optional`、`@default(v)`、`@fk(t.col)`）は、テーブルをデータベースに接続する際に使用します。詳細は [データベースドキュメント](database.md) を参照してください。

### 行アクセス

```xcx
products[0].name    --- "Laptop" (sugar for .get(0))
products[1].price   --- 1499.50
```

### テーブルメソッド

| メソッド             | シグネチャ              | 戻り値 | 説明                                             |
|----------------------|-------------------------|--------|--------------------------------------------------|
| `.count()`           | `() → i`                | `i`    | 行数                                             |
| `.get(i)`            | `(i) → row`             | `row`  | インデックス `i` の行                            |
| `.insert(vals...)`   | `(T...) → b`            | `b`    | 行を追加（`@auto` 列はスキップ）                 |
| `.add(vals...)`      | `(T...) → b`            | `b`    | `.insert()` の別名 — 同一の挙動                  |
| `.update(i, vals)`   | `(i, [T...]) → b`       | `b`    | 行の値を置換；`@auto` 列は保持                   |
| `.delete(i)`         | `(i) → b`               | `b`    | インデックス `i` の行を削除                      |
| `.where(pred)`       | `(expr) → table`        | `table`| フィルタ — 新しいテーブルを返す                  |
| `.join(t, pred)`     | `(table, pred) → table` | `table`| 別テーブルとの内部結合                           |
| `.toJson()`          | `() → json`             | `json` | すべての行を JSON オブジェクトの配列にシリアライズ |
| `.show()`            | `() → b`                | `b`    | テーブルを ASCII 形式で出力                      |

### `.add()` と `.insert()` の名前付き引数

テーブルにデータベース列属性がある場合、位置ではなく列名で値を渡せます。名前付き引数は**任意**です — 位置引数による呼び出しも引き続き有効です。

```xcx
table: users {
    columns = [
        id    :: i @auto @pk,
        name  :: s @unique,
        age   :: i,
        phone :: s @optional,
        role  :: s @default("user")
    ]
    rows = [EMPTY]
};

--- Positional
users.add("Alice", 25, "", "user");

--- Named
users.add(name = "Alice", age = 25, phone = "", role = "user");

--- Mixed — positional args must come first
users.add("Alice", age = 25, role = "admin");
```

**名前空間の分離。** `=` の左辺は常に列名です。右辺はローカルスコープの式です。これらは独立した 2 つの名前空間であり、競合しません：

```xcx
s: name = "Alice";
users.add(name = name, age = 25);
--- left "name"  = column users.name
--- right "name" = local variable
```

ルール：位置引数は名前付き引数より前に置く；`@auto` 列は渡せない；必須列（`@optional` でも `@default` でもない）を省略するとコンパイルエラー；同一呼び出し内で列名が重複するとコンパイルエラー。完全な仕様は [データベースドキュメント](database.md) を参照してください。

### フィルタ（where）

```xcx
--- Shorthand syntax (column names usable directly)
table: expensive = products.where(price > 1000.0);
table: named     = products.where(name HAS "Pro");

--- Lambda
table: r = products.where(row -> row.price > 1000.0);

--- Chaining
table: result = products
    .where(price > 1000.0)
    .where(name HAS "Pro");
```

> [!IMPORTANT]
> **`.where()` における名前の競合（S301）**：述語内では列名がローカル変数より優先されます。ローカル変数が列名と同じ名前の場合、コンパイルエラーを避けるために変数名を変更してください。
>
> ```xcx
> --- Wrong (conflict: 'token' exists both as column and parameter)
> fiber verify(s: token) {
>     table: sess = db.sessions.where(token == token);
> };
>
> --- Correct
> fiber verify(s: t) {
>     table: sess = db.sessions.where(token == t);
> };
> ```

### 結合

```xcx
--- Key-based join
table: report = users.join(orders, "id", "user_id");

--- Lambda join
table: custom = tableA.join(tableB, (a, b) -> a.id == b.ref_id);
```

結合したテーブルが同じ列名（結合キー以外）を共有する場合、結果の列には `{table_name}_` プレフィックスが付きます。

### シリアライズ（toJson）

#### シグネチャ
`.toJson() → json`

#### 説明
テーブルのすべての行を JSON 配列にシリアライズします。各行は列名をキーとするオブジェクトになります。`@auto` 列も結果に含まれます。

#### 形式
常に JSON 配列（`[...]`）を返します。空のテーブルは `[]` を返します。

```xcx
table: products {
    columns = [ id :: i @auto, name :: s, price :: f ]
    rows = [ ("Laptop", 2999.99), ("Phone", 1499.50) ]
};

json: result = products.toJson();
```

**JSON 出力：**
```json
[
    {"id": 1, "name": "Laptop", "price": 2999.99},
    {"id": 2, "name": "Phone",  "price": 1499.50}
]
```

#### 型マッピング
| XCX 型          | JSON 型                                   |
|-----------------|-------------------------------------------|
| `i` / `int`     | `number`                                  |
| `f` / `float`   | `number`                                  |
| `s` / `str`     | `string`                                  |
| `b` / `bool`    | `boolean`                                 |
| `date`          | `string`（形式 `"YYYY-MM-DD HH:mm:ss"`）  |

#### 特殊ケースの挙動
- **空のテーブル**：空配列 `[]` を返します。
- **フィルタ済みテーブル**：現在テーブル内にある行（`.where()` 適用後）のみがシリアライズされます。
- **結合済みテーブル**：結合結果のすべての列が含まれます。
