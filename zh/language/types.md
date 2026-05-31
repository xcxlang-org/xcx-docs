# XCX 3.1 数据类型

## 简单类型

| 符号          | 类型       | 默认值       | 示例                       |
|---------------|-----------|--------------|----------------------------|
| `i` / `int`   | 整数       | `0`          | `42`, `-7`, `0`            |
| `f` / `float` | 浮点数     | `0.0`        | `3.14`, `-0.5`, `2.0`      |
| `s` / `str`   | 字符串     | `""`         | `"hello"`, `""`            |
| `b` / `bool`  | 布尔值     | `false`      | `true`, `false`            |
| `date`        | 日期       | `1970-01-01` | `date("2024-12-25")`       |
| `json`        | JSON 对象  | `null`       | `<<< {"key": "value"} >>>` |

## 复合类型

| 符号         | 类型                     | 声明示例                                     |
|--------------|-------------------------|----------------------------------------------|
| `array:i`    | 整数数组                 | `array:i: nums {1, 2, 3}`                    |
| `array:f`    | 浮点数数组               | `array:f: vals {1.0, 2.5}`                   |
| `array:s`    | 字符串数组               | `array:s: words {"a", "b"}`                  |
| `array:b`    | 布尔值数组               | `array:b: flags {true, false}`               |
| `array:json` | JSON 对象数组            | `array:json: items`                          |
| `set:N`      | 自然数集合               | `set:N: s {1,,10}`                           |
| `set:Z`      | 整数集合                 | `set:Z: s {-2, 0, 2}`                        |
| `set:Q`      | 有理数（浮点）集合       | `set:Q: s {0.5, 1.0, 1.5}`                   |
| `set:S`      | 字符串集合               | `set:S: s {"a", "b"}`                        |
| `set:B`      | 布尔值集合               | `set:B: s {true, false}`                     |
| `set:C`      | 字符集合                 | `set:C: s {"A",,"Z"}`                        |
| `map`        | 键值映射                 | `map: m { schema = [s <-> i] data = [...] }` |
| `table`      | 关系表                   | `table: t { columns = [...] rows = [...] }`  |
| `database:`  | 数据库连接               | `database: db { engine = "sqlite", path = "app.db" }` |
| `fiber:T`    | 带类型纤程               | `fiber:b: f = my_fiber(arg)`                 |
| `fiber:`     | 无返回值纤程             | `fiber: f = my_void_fiber(arg)`              |

### array:json

`array:json` 是存储 `json` 类型元素的数组。主要用于处理返回 JSON 对象数组的 HTTP 响应。

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

`array:json` 的方法与其他数组类型相同（`.push()`、`.pop()`、`.get(i)`、`.size()` 等）。

### database:

`database:` 表示与关系型数据库的连接。通过配置块声明，并使用其内置方法进行操作。完整 API 请参阅[数据库文档](database.md)。

```xcx
database: app {
    engine = "sqlite",
    path   = "app.db"
};

yield app.sync(users);
table: all = yield app.fetch(users);
```

## 默认值

```xcx
int:   def_int;     --- 0 (alias for i)
float: def_float;   --- 0.0 (alias for f)
str:   def_str;     --- "" (alias for s)
bool:  def_bool;    --- false (alias for b)
```

## 类型转换

```xcx
f: x = 3.7;
i: n = i(x);       --- 3 (truncate toward zero)

i: m = 42;
f: y = f(m);       --- 42.0

i: num = 99;
s: str = s(num);   --- "99"
```

> [!NOTE]
> `b → i` 转换被**有意禁止**。`true + 1` 属于类型错误，用于避免 C 系语言中常见的逻辑缺陷。
