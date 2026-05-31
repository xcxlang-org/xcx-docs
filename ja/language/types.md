# XCX 3.1 データ型

## 単純型

| 記号          | 型           | デフォルト     | 例                         |
|---------------|-------------|--------------|----------------------------|
| `i` / `int`   | 整数        | `0`          | `42`, `-7`, `0`            |
| `f` / `float` | 浮動小数点  | `0.0`        | `3.14`, `-0.5`, `2.0`      |
| `s` / `str`   | 文字列      | `""`         | `"hello"`, `""`            |
| `b` / `bool`  | 真偽値      | `false`      | `true`, `false`            |
| `date`        | 日付        | `1970-01-01` | `date("2024-12-25")`       |
| `json`        | JSON オブジェクト | `null` | `<<< {"key": "value"} >>>` |

## 複合型

| 記号         | 型                       | 宣言の例                                     |
|--------------|-------------------------|----------------------------------------------|
| `array:i`    | 整数の配列              | `array:i: nums {1, 2, 3}`                    |
| `array:f`    | 浮動小数点の配列        | `array:f: vals {1.0, 2.5}`                   |
| `array:s`    | 文字列の配列            | `array:s: words {"a", "b"}`                  |
| `array:b`    | 真偽値の配列            | `array:b: flags {true, false}`               |
| `array:json` | JSON オブジェクトの配列 | `array:json: items`                          |
| `set:N`      | 自然数の集合            | `set:N: s {1,,10}`                           |
| `set:Z`      | 整数の集合              | `set:Z: s {-2, 0, 2}`                        |
| `set:Q`      | 有理数（浮動小数点）の集合 | `set:Q: s {0.5, 1.0, 1.5}`                |
| `set:S`      | 文字列の集合            | `set:S: s {"a", "b"}`                        |
| `set:B`      | 真偽値の集合            | `set:B: s {true, false}`                     |
| `set:C`      | 文字の集合              | `set:C: s {"A",,"Z"}`                        |
| `map`        | キー・値マップ          | `map: m { schema = [s <-> i] data = [...] }` |
| `table`      | リレーショナルテーブル  | `table: t { columns = [...] rows = [...] }`  |
| `database:`  | データベース接続        | `database: db { engine = "sqlite", path = "app.db" }` |
| `fiber:T`    | 型付きファイバ          | `fiber:b: f = my_fiber(arg)`                 |
| `fiber:`     | void ファイバ           | `fiber: f = my_void_fiber(arg)`              |

### array:json

`array:json` は `json` 型の要素を格納する配列です。主に、JSON オブジェクトの配列を返す HTTP レスポンスを扱う際に使用します。

```xcx
--- Declare an empty JSON array
array:json: items;

--- Iterate over a JSON array (e.g. from a network response)
i: size = resp.body.size();
i: idx = 0;
while (idx < size) do;
    json: item = resp.body.get(idx);
    s: name; item.bind("name", name);
    >! name;
    idx = idx + 1;
end;

--- Add an element
json: obj <<< {"key": "value"} >>>;
items.push(obj);
```

`array:json` のメソッドは、他の配列型と同一です（`.push()`、`.pop()`、`.get(i)`、`.size()` など）。

### database:

`database:` はリレーショナルデータベースへの接続を表します。設定ブロック付きで宣言し、組み込みメソッド経由で使用します。API の詳細は [データベースドキュメント](database.md) を参照してください。

```xcx
database: app {
    engine = "sqlite",
    path   = "app.db"
};

yield app.sync(users);
table: all = yield app.fetch(users);
```

## デフォルト値

```xcx
int:   def_int;     --- 0 (alias for i)
float: def_float;   --- 0.0 (alias for f)
str:   def_str;     --- "" (alias for s)
bool:  def_bool;    --- false (alias for b)
```

## 型キャスト

```xcx
f: x = 3.7;
i: n = i(x);       --- 3 (truncate toward zero)

i: m = 42;
f: y = f(m);       --- 42.0

i: num = 99;
s: str = s(num);   --- "99"
```

> [!NOTE]
> `b → i` 変換は**意図的に禁止**されています。`true + 1` は型エラーとなり、C 系言語でよくある論理バグを防ぎます。
