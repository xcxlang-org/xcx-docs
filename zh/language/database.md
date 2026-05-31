# XCX 3.1 数据库

XCX 3.1 通过 `database:` 类型和一组内置方法提供原生关系型数据库支持。3.1 版本目前仅支持 **SQLite**。

---

## 目录

1. [连接声明（`database:`）](#1-connection-declaration-database)
2. [表列属性](#2-table-column-attributes)
3. [命名参数](#3-named-arguments)
4. [结果捕获（`as`）](#4-result-capture-as)
5. [纤程作用域限制（Windows）](#5-fiber-scoping-limitations-windows)
6. [API 参考](#6-api-reference)
   - 5.1 [DDL](#51-ddl)
   - 5.2 [写入操作](#52-write-operations)
   - 5.3 [读取操作](#53-read-operations)
   - 5.4 [删除操作](#54-delete-operations)
   - 5.5 [事务](#55-transactions)
   - 5.6 [其他](#56-other)
6. [类型映射（XCX ↔ SQL）](#6-type-mapping-xcx--sql)
7. [安全约束](#7-security-constraints)
8. [完整示例](#8-full-example)

---

## 1. 连接声明（`database:`）

```xcx
database: app {
    engine  = "sqlite",
    path    = "app.db"
};
```

### 字段

| 字段       | 类型 | 必填 | 默认值  | 说明                                      |
|------------|------|------|---------|-------------------------------------------|
| `engine`   | `s`  | 是   | —       | 数据库引擎。目前仅 `"sqlite"`。           |
| `path`     | `s`  | 是   | —       | `.db` 文件路径（相对于项目根目录）。      |
| `timeout`  | `i`  | 否   | `5000`  | 操作超时（毫秒）。                        |
| `readonly` | `b`  | 否   | `false` | 只读模式。                                |

### 行为

- 连接是**延迟**的——首次使用时才打开。
- 连接失败（如权限不足）时会自动触发 `halt.error`。
- SQLite 在文件不存在时会创建数据库文件（除非 `readonly = true`）。
- 允许多个并发连接——每个连接有独立名称。

```xcx
database: main { engine = "sqlite", path = "main.db" };
database: logs { engine = "sqlite", path = "logs.db" };

yield main.sync(users);
yield logs.sync(events);
```

### I/O 模型

所有涉及磁盘或驱动的数据库操作都是 I/O 操作：

| 上下文       | 行为           |
|--------------|----------------|
| 纤程内部     | 需要 `yield`   |
| 纤程外部     | 同步阻塞       |

适用于**读取**（`fetch`、`query`、`queryRaw`）、**写入**（`push`、`save`、`insert`、`exec`）、**删除**（`remove`、`exec`）和 **DDL**（`sync`、`drop`）操作。

**例外——无需 `yield` 的方法。** 仅操作事务状态或元数据的方法在任何上下文中都不需要 `yield`：

- `db.has()`、`db.close()`、`db.isOpen()`
- `db.begin()`、`db.commit()`、`db.rollback()`

> `begin()`、`commit()` 和 `rollback()` 不触及数据——它们仅管理驱动中的事务状态。

---

## 2. 表列属性

这些属性扩展 `table:` 块声明，供数据库模块使用。

### 属性

| 属性            | 说明                                                         | SQL 等价            |
|-----------------|--------------------------------------------------------------|---------------------|
| `@pk`           | 主键。可与 `@auto` 组合。                                    | `PRIMARY KEY`       |
| `@unique`       | 列内值必须唯一。                                             | `UNIQUE`            |
| `@optional`     | SQL 中该列可为 `NULL`。XCX 读取时返回该类型的默认值。        | `NULL`              |
| `@default(v)`   | SQL 中的默认列值。                                           | `DEFAULT v`         |
| `@fk(t.col)`    | 指向表 `t` 的列 `col` 的外键。                               | `REFERENCES t(col)` |

### 示例

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

### `@optional` 行为

XCX 没有独立的 `null` 类型。SQL 中 `@optional` 列可存储 `NULL`，但读入 XCX 时会自动替换为该类型的默认值：

| XCX 类型 | SQL 为 NULL 时的值 |
|----------|--------------------|
| `i`      | `0`                |
| `f`      | `0.0`              |
| `s`      | `""`               |
| `b`      | `false`            |

> **注意：** `@optional` 仅表示 SQL 侧接受 `NULL`。XCX 不保留 `NULL` 与默认值之间的区别——该信息在读取时丢失。若应用逻辑需要区分这两种情况，请在单独列中存储标志。

### `add()` 和 `insert()` 的列省略规则

| 列属性                  | 可省略？     | 写入 SQL 的内容        |
|-------------------------|--------------|------------------------|
| `@auto`                 | 是（始终）   | 自动生成值             |
| `@default(v)`           | 是           | `DEFAULT` 中的值 `v`   |
| `@optional`             | 是           | `NULL`                 |
| `@optional @default(v)` | 是           | `DEFAULT` 中的值 `v`   |
| 无属性                  | **否**       | 编译错误               |

> `@auto` 列永远不能显式提供——无论位置参数还是命名参数。尝试这样做是**编译错误**。

---

## 3. 命名参数

命名参数允许向 `table.add()`、`table.insert()` 和 `db.insert()` **按列名**而非按位置传值。这是可选的附加功能——所有现有位置调用仍然完全有效。

命名参数**仅**适用于表插入操作。不适用于用户定义的函数、纤程或其他内置方法。列名由编译器从 `table:` 模式静态获知。

### 语法

```xcx
--- Positional (backward compatible)
users.add("Alice", 25, "", "user");

--- Named
users.add(name = "Alice", age = 25, phone = "", role = "user");

--- With db.insert()
yield app.insert(users, name = "Alice", age = 25) as saved;
```

**命名空间分离。** `=` 左侧始终是列名。右侧是局部作用域中的表达式。两者是独立的命名空间——不会冲突：

```xcx
s: name = "Alice";
users.add(name = name, age = 25);
--- left "name"  = column users.name
--- right "name" = local variable
```

**求值顺序。** 右侧表达式按书写顺序从左到右求值：

```xcx
users.add(name = get_name(), age = get_age());
--- get_name() is called before get_age()
```

### 混合位置与命名参数

位置参数和命名参数可以混合。**位置参数必须在命名参数之前。**

```xcx
--- OK — positional before named
users.add("Alice", age = 25, role = "admin");

--- COMPILE ERROR — named before positional
users.add(name = "Alice", 25, "admin");
```

**赋值规则。** 位置参数按声明顺序从左到右分配给列，**跳过 `@auto`**，直到用完。命名参数按名称填充剩余列。

```xcx
--- "Alice" → name, rest named
users.add("Alice", age = 25, role = "admin");
--- phone omitted → NULL (@optional)

--- "Alice", 25 → name, age; role named, phone omitted
users.add("Alice", 25, role = "mod");
```

位置参数**连续**分配，不跳过中间列。跳过中间列只能通过命名参数，或完全省略（若该列可选）：

```xcx
--- COMPILE ERROR — compiler assigns: name="Alice", age="" — type mismatch
--- there is no way to "skip" age positionally
users.add("Alice", "", "user");
```

同一列既按位置又按名称提供是**编译错误**：

```xcx
--- COMPILE ERROR — name provided twice
users.add("Alice", name = "Bob", age = 25);
```

### 编译时规则

| 规则                     | 行为       |
|--------------------------|------------|
| 提供了 `@auto` 列        | 编译错误   |
| 未知列名                 | 编译错误   |
| 重复列名                 | 编译错误   |
| 省略必填列               | 编译错误   |
| 命名参数在位置参数之前   | 编译错误   |

**完整性表：**

| 列属性                  | 可省略？     | 省略时的值                           |
|-------------------------|--------------|--------------------------------------|
| `@auto`                 | 是（始终）   | 自动生成值                           |
| `@default(v)`           | 是           | `DEFAULT` 中的值 `v`                 |
| `@optional`             | 是           | `NULL`（读取时 → XCX 默认值）        |
| `@optional @default(v)` | 是           | `DEFAULT` 中的值 `v`                 |
| 无属性                  | **否**       | 编译错误                             |

```xcx
--- OK — role has @default("user"), phone is @optional
users.add(name = "Alice", age = 25);

--- COMPILE ERROR — age is required
users.add(name = "Alice");
```

---

## 4. 结果捕获（`as`）

`as` 是 XCX 中捕获块操作结果的通用机制：

```xcx
--- HTTP request
net.request { method = "GET", url = "..." } as resp;

--- DB operations
yield app.insert(users, name = "Alice", age = 25) as saved;
yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
```

### 写入/删除结果对象

`db.insert()`、`db.save()`、`db.push()` 和 `db.exec()` 返回包含两个字段的对象：

| 字段       | 类型 | 说明                                              |
|------------|------|---------------------------------------------------|
| `affected` | `i`  | 操作更改/插入的行数                               |
| `insertId` | `i`  | 最后插入记录的 ID（`@auto @pk`）；无插入时为 `0`  |

```xcx
yield app.insert(users, name = "Alice", age = 25) as saved;
>! "New user ID: " + s(saved.insertId);

yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
>! "Deleted: " + s(res.affected);
```

使用 `as` 是可选的——若不需要结果可省略：

```xcx
yield app.save(users);
```

### `affected` 和 `insertId` 语义

| 操作                       | `affected`              | `insertId`                |
|----------------------------|-------------------------|---------------------------|
| `insert()` — 单行          | `1`                     | 插入行的 ID               |
| `push()` — 多行            | 插入行数                | 最后插入行的 ID           |
| `save()` — upsert（插入）  | `1`                     | 插入行的 ID               |
| `save()` — upsert（更新）  | `1`                     | `0`                       |
| `exec(DELETE ...)`         | 删除行数                | `0`                       |
| `exec(UPDATE ...)`         | 更新行数                | `0`                       |

> 操作失败（约束违反、无连接、`readonly`）时会触发 `halt.error`，执行不会到达使用 `as` 结果的代码。若代码能看到结果对象，则它始终有效。

---

## 5. 纤程作用域限制（Windows）

在 XCX 3.1 中，特别是在 Windows 环境下，用数据库结果直接初始化纤程局部变量时，后续行偶尔会出现 `Undefined variable` [S101] 错误。

### 推荐变通方案

为确保变量在纤程内条件块之间绝对可见，建议**先声明变量类型**，再在单独语句中赋值数据库结果。

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

此模式确保变量在被分析器访问前已正确注册到符号表。该限制计划在 XCX 4.0 中通过原生架构解决。

---

## 6. API 参考

### 5.1 DDL

```xcx
--- Creates the SQL table if it does not exist (based on XCX table schema)
yield app.sync(users);

--- Drops the SQL table
yield app.drop(users);

--- Checks if table exists in the database (no data touched — no yield)
b: exists = app.has(users);
```

### 5.2 写入操作

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

出错时（违反 `@unique`、`@fk`、无连接）会触发 `halt.error`。

### 5.3 读取操作

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

#### `queryRaw` 结果上的 `.first()`

`.first()` 将 JSON 数组的第一个元素作为单个 `json` 对象返回。查询无行时触发 `halt.error`。典型用于聚合查询（`COUNT`、`SUM`、`MAX`）和最多返回一行的查找。

```xcx
json: row = yield app.queryRaw("SELECT COUNT(*) as n FROM users").first();
i: total; row.bind("n", total);
>! "Users: " + s(total);
```

若不确定结果非空，使用 `.first()` 前先检查大小：

```xcx
json: results = yield app.queryRaw("SELECT * FROM users WHERE age > ?", [100]);
if (results.size() > 0) then;
    json: row = results.get(0);
end;
```

#### `.where()` 中支持的操作符

`.where()` 过滤器会编译为 SQL `WHERE` 子句。仅支持 XCX 表达式的有限子集：

| 类别       | 支持内容                         |
|------------|----------------------------------|
| 比较       | `==`、`!=`、`<`、`<=`、`>`、`>=` |
| 逻辑       | `AND`、`OR`、`NOT`               |
| 值         | 字面量、局部变量                 |
| 左操作数   | 仅列名                           |

函数调用、字符串方法和嵌套表达式在 `db.fetch()` 的 `.where()` 中不支持。使用不支持的表达式是**编译错误**。

```xcx
--- OK
table: r = yield app.fetch(users).where(age > 18 AND role == "admin");

--- COMPILE ERROR — string method not allowed in where() with fetch
table: r = yield app.fetch(users).where(name.lower() == "alice");
```

链式 `.where()` 用 `AND` 连接条件：

```xcx
table: r = yield app.fetch(users)
    .where(age > 18)
    .where(role == "admin");
--- SQL equivalent: WHERE age > 18 AND role = 'admin'
```

#### `db.fetch()` 与本地 `rows`

`db.fetch()` 仅读取 SQL 数据库中的数据——`table:` 块中声明的本地 `rows` 会被忽略。本地表定义仅作为模式提示。

```xcx
table: users {
    columns = [ id :: i @auto @pk, name :: s ]
    rows = [ ("Alice") ]   --- these rows do NOT go into fetch
};

yield app.sync(users);

--- Returns only rows stored in app.db
table: all = yield app.fetch(users);
```

要将本地 `rows` 插入数据库，在 `db.fetch()` 之前使用 `db.push(users)`。

### 5.4 删除操作

```xcx
--- DELETE with filter — .where() is required (D401)
yield app.remove(users).where(age < 18);

--- Delete all rows
yield app.truncate(users);

--- Raw DELETE with parameters
yield app.exec("DELETE FROM users WHERE id = ?", [5]) as res;
```

> **不带 `.where()` 的 `db.remove()` 是编译错误（D401）。** 要删除所有行，请显式使用 `db.truncate(users)`。

### 5.5 事务

```xcx
app.begin();
yield app.save(users);
yield app.save(posts);
app.commit();
```

> 若事务内任何操作触发 `halt.error`，会在帧中止前**自动回滚**事务。连接恢复到 `begin()` 之前的状态。

`rollback()` 用于在错误发生前，在条件逻辑中显式取消事务：

```xcx
app.begin();
b: valid = validate_users(users);
if (NOT valid) then;
    app.rollback();
end;
yield app.save(users);
app.commit();
```

### 5.6 其他

```xcx
app.close();
b: alive = app.isOpen();
```

---

## 6. 类型映射（XCX ↔ SQL）

| XCX 类型 | SQL 类型（SQLite）             |
|----------|--------------------------------|
| `i`      | `INTEGER`                      |
| `f`      | `REAL`                         |
| `s`      | `TEXT`                         |
| `b`      | `INTEGER`（0 / 1）             |
| `date`   | `TEXT`（`YYYY-MM-DD HH:mm:ss`）|

---

## 7. 安全约束

| 约束                              | 行为                                           |
|-----------------------------------|------------------------------------------------|
| 路径包含 `..`                     | `halt.fatal` — 路径遍历                        |
| 绝对路径                          | `halt.fatal`                                   |
| 含未验证输入的原始 SQL            | 开发者负责清理——使用 `?` 参数                  |
| `readonly = true` + DML 操作      | `halt.error`                                   |
| 无 `@pk` 的表上调用 `save()`      | 编译错误                                       |
| 不带 `.where()` 的 `remove()`     | 编译错误（D401）                               |

始终使用带 `?` 的预处理语句：

```xcx
--- WRONG
app.exec("DELETE FROM users WHERE name = '" + name + "'");

--- CORRECT
app.exec("DELETE FROM users WHERE name = ?", [name]);
```

---

## 8. 完整示例

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
