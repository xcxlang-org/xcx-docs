# XCX 3.1 集合

## 数组

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

### 数组方法

| 方法              | 签名         | 返回值 | 说明                                                         |
|-------------------|--------------|--------|--------------------------------------------------------------|
| `.size()`         | `() → i`     | `i`    | 元素数量                                                     |
| `.get(i)`         | `(i) → T`    | `T`    | 位置 `i` 处的元素（从 0 开始）；越界时 `halt.error`          |
| `.push(val)`      | `(T) → b`    | `b`    | 在末尾追加元素                                               |
| `.pop()`          | `() → T`     | `T`    | 移除并返回最后一个元素                                       |
| `.insert(i, val)` | `(i, T) → b` | `b`    | 在位置 `i` 插入，后续元素后移；越界时 `halt.error`           |
| `.update(i, val)` | `(i, T) → b` | `b`    | 覆盖位置 `i` 处的元素；越界时 `halt.error`                   |
| `.delete(i)`      | `(i) → b`    | `b`    | 删除位置 `i` 处的元素；越界时 `halt.error`                   |
| `.find(val)`      | `(T) → i`    | `i`    | 首次出现的索引，未找到则为 `-1`                              |
| `.contains(val)`  | `(T) → b`    | `b`    | 检查值是否存在                                               |
| `.isEmpty()`      | `() → b`     | `b`    | 为空时返回 `true`                                            |
| `.clear()`        | `() → b`     | `b`    | 移除所有元素                                                 |
| `.sort()`         | `() → b`     | `b`    | 升序排序（原地）                                             |
| `.reverse()`      | `() → b`     | `b`    | 反转顺序（原地）                                             |
| `.toStr()`        | `() → s`     | `s`    | 将数组序列化为 JSON 格式字符串                               |
| `.toJson()`       | `() → json`  | `json` | 将数组转换为原生 JSON 结构                                   |
| `.show()`         | `() → b`     | `b`    | 将内容打印到终端                                             |

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

### 域

| 符号 | 类型             | 示例                          |
|------|------------------|-------------------------------|
| `N`  | 自然数（≥ 0）    | `set:N: s {0, 1, 2}`          |
| `Z`  | 整数             | `set:Z: s {-3, 0, 3}`         |
| `Q`  | 有理数（浮点）   | `set:Q: s {0.5, 1.0}`         |
| `S`  | 字符串           | `set:S: s {"a", "b"}`         |
| `B`  | 布尔值           | `set:B: s {true, false}`      |
| `C`  | 字符             | `set:C: s {"A",,"Z"}`         |

### 初始化

集合可用显式值或范围初始化。范围**两端均包含**。

```xcx
set:N: small  {1,,5};                  --- {1, 2, 3, 4, 5}
set:N: evens  {0,,100 @step 2};        --- {0, 2, 4, ...}
set:Q: thirds {0.0,,1.0 @step 0.33};
set:C: letters {"A",,"Z"};            --- all uppercase letters
```

集合会**自动去重**元素。

### 集合运算

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

### 集合方法

| 方法           | 签名      | 返回值 | 说明                                |
|----------------|-----------|--------|-------------------------------------|
| `.size()`      | `() → i`  | `i`    | 元素数量                            |
| `.isEmpty()`   | `() → b`  | `b`    | 为空时返回 `true`                   |
| `.contains(v)` | `(T) → b` | `b`    | 检查成员关系                        |
| `.add(v)`      | `(T) → b` | `b`    | 添加元素（忽略重复）                |
| `.remove(v)`   | `(T) → b` | `b`    | 移除元素（不存在时无操作）          |
| `.clear()`     | `() → b`  | `b`    | 移除所有元素                        |
| `.show()`      | `() → b`  | `b`    | 将 `{elem, elem, ...}` 打印到终端   |

### 随机选取与迭代

```xcx
--- Random selection from a set:
i: picked_set = random.choice from small;

--- Random selection from an array:
array:i: nums {1, 2, 3, 4, 5};
i: picked_arr = random.choice from nums;
```

从空集合或空数组中选取将返回 `false`。

### 迭代

```xcx
for p in small do;
    >! p;
end;
```

---

## 映射

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

### 映射方法

| 方法             | 签名            | 返回值    | 说明                                   |
|------------------|-----------------|-----------|----------------------------------------|
| `.size()`        | `() → i`        | `i`       | 键值对数量                             |
| `.get(key)`      | `(K) → V`       | `V`       | 返回值；键不存在时 `halt.error`        |
| `.contains(key)` | `(K) → b`       | `b`       | 检查键是否存在                         |
| `.insert(k, v)`  | `(K, V) → b`    | `b`       | 插入或覆盖                             |
| `.remove(key)`   | `(K) → b`       | `b`       | 删除键值对；键不存在时返回 `false`     |
| `.keys()`        | `() → array:K`  | `array:K` | 返回键数组                             |
| `.values()`      | `() → array:V`  | `array:V` | 返回值数组                             |
| `.clear()`       | `() → b`        | `b`       | 移除所有键值对                         |
| `.toStr()`       | `() → s`        | `s`       | 将映射序列化为 JSON 格式字符串         |
| `.show()`        | `() → b`        | `b`       | 将映射内容打印到终端                   |
| `.toJson()`      | `() → json`     | `json`    | 将映射序列化为 JSON 对象               |

映射键在生成的 JSON 对象中会转换为字符串。

### 映射序列化（toJson）

#### 签名
`.toJson() → json`

#### 说明
将映射序列化为 JSON 对象。所有键会通过 `.toString()` 表示转换为字符串，以满足 JSON 对象键的要求。

```xcx
map: scores {
    schema = [s <-> i]
    data = [ "alice" :: 100, "bob" :: 85 ]
};
json: j = scores.toJson();
```

**JSON 输出：**
```json
{
    "alice": 100,
    "bob": 85
}
```

#### 特殊情况行为
- **空映射**：返回空对象 `{}`。
- **嵌套结构**：若映射包含数组、其他映射或表，会递归序列化为其 JSON 等价物。
- **键转换**：所有映射键在生成的 JSON 对象中通过标准 `.toString()` 表示转换为字符串。

#### 类型映射
| XCX 类型 | JSON 类型 |
|----------|-----------|
| `i`      | `number`  |
| `f`      | `number`  |
| `s`      | `string`  |
| `b`      | `boolean` |
| `date`   | `string`（格式 `"YYYY-MM-DD HH:mm:ss"`） |
| `json`   | （不变）  |

使用 `.get()` 前应先调用 `.contains()`：

```xcx
if (ages.contains("alice")) then;
    >! ages.get("alice");
end;
```

---

## 表

带可选自增列的关系型数据结构。

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

`i` 列上的 `@auto` 修饰符会创建自增 ID——在 `.insert()` 和 `.add()` 中会跳过该列。

> [!NOTE]
> 其他列属性（`@pk`、`@unique`、`@optional`、`@default(v)`、`@fk(t.col)`）在将表连接到数据库时使用。详见[数据库文档](database.md)。

### 行访问

```xcx
products[0].name    --- "Laptop" (sugar for .get(0))
products[1].price   --- 1499.50
```

### 表方法

| 方法                 | 签名                    | 返回值 | 说明                                      |
|----------------------|-------------------------|--------|-------------------------------------------|
| `.count()`           | `() → i`                | `i`    | 行数                                      |
| `.get(i)`            | `(i) → row`             | `row`  | 索引 `i` 处的行                           |
| `.insert(vals...)`   | `(T...) → b`            | `b`    | 添加行（跳过 `@auto` 列）                 |
| `.add(vals...)`      | `(T...) → b`            | `b`    | `.insert()` 的别名——行为相同              |
| `.update(i, vals)`   | `(i, [T...]) → b`       | `b`    | 替换行值；`@auto` 列保留                  |
| `.delete(i)`         | `(i) → b`               | `b`    | 删除索引 `i` 处的行                       |
| `.where(pred)`       | `(expr) → table`        | `table`| 过滤——返回新表                            |
| `.join(t, pred)`     | `(table, pred) → table` | `table`| 与另一表内连接                            |
| `.toJson()`          | `() → json`             | `json` | 将所有行序列化为 JSON 对象数组            |
| `.show()`            | `() → b`                | `b`    | 以 ASCII 格式打印表                       |

### `.add()` 和 `.insert()` 的命名参数

当表具有数据库列属性时，可按列名而非位置传值。命名参数是**可选的**——位置调用仍然完全有效。

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

--- Positional (backward compatible)
users.add("Alice", 25, "", "user");

--- Named
users.add(name = "Alice", age = 25, phone = "", role = "user");

--- Mixed — positional args must come first
users.add("Alice", age = 25, role = "admin");
```

**命名空间分离。** `=` 左侧始终是列名。右侧是局部作用域中的表达式。两者是独立的命名空间——不会冲突：

```xcx
s: name = "Alice";
users.add(name = name, age = 25);
--- left "name"  = column users.name
--- right "name" = local variable
```

规则：位置参数必须在命名参数之前；`@auto` 列永远不能传入；省略必填列（非 `@optional`、非 `@default`）是编译错误；同一调用中重复的列名是编译错误。完整规范见[数据库文档](database.md)。

### 过滤（where）

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
> **`.where()` 中的名称冲突（S301）**：在谓词内部，列名优先于局部变量。若局部变量与列同名，请重命名变量以避免编译错误。
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

### 连接

```xcx
--- Key-based join
table: report = users.join(orders, "id", "user_id");

--- Lambda join
table: custom = tableA.join(tableB, (a, b) -> a.id == b.ref_id);
```

当连接后的表共享列名（连接键除外）时，结果列会加上 `{table_name}_` 前缀。

### 序列化（toJson）

#### 签名
`.toJson() → json`

#### 说明
将所有行序列化为 JSON 数组，每行成为以列名为键的对象。`@auto` 列会包含在结果中。

#### 格式
始终返回 JSON 数组（`[...]`）。空表返回 `[]`。

```xcx
table: products {
    columns = [ id :: i @auto, name :: s, price :: f ]
    rows = [ ("Laptop", 2999.99), ("Phone", 1499.50) ]
};

json: result = products.toJson();
```

**JSON 输出：**
```json
[
    {"id": 1, "name": "Laptop", "price": 2999.99},
    {"id": 2, "name": "Phone",  "price": 1499.50}
]
```

#### 类型映射
| XCX 类型        | JSON 类型                                 |
|-----------------|-------------------------------------------|
| `i` / `int`     | `number`                                  |
| `f` / `float`   | `number`                                  |
| `s` / `str`     | `string`                                  |
| `b` / `bool`    | `boolean`                                 |
| `date`          | `string`（格式 `"YYYY-MM-DD HH:mm:ss"`）  |

#### 特殊情况行为
- **空表**：返回空数组 `[]`。
- **过滤后的表**：仅序列化当前表中的行（`.where()` 之后）。
- **连接后的表**：包含连接结果的所有列。
