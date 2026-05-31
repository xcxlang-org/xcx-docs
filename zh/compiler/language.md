# XCX 语言参考 — v3.1

> XCX 语言语法与语义的完整参考。

---

## 目录

1. [数据类型](#数据类型)
2. [变量与常量](#变量与常量)
3. [运算符](#运算符)
4. [控制流](#控制流)
5. [函数](#函数)
6. [纤程（协程）](#纤程协程)
7. [集合](#集合)
8. [JSON](#json)
9. [表](#表)
10. [数据库（SQLite）](#数据库sqlite)
11. [网络（HTTP）](#网络http)
12. [存储（文件）](#存储文件)
13. [终端](#终端)
14. [日期与时间](#日期与时间)
15. [随机数](#随机数)
16. [加密](#加密)
17. [诊断与 Halt](#诊断与-halt)
18. [Include](#include)

---

## 数据类型

| XCX 类型 | 说明 | 示例 |
|---|---|---|
| `i` | 48 位整数 | `i: age = 25;` |
| `f` | 64 位浮点数 | `f: pi = 3.14;` |
| `s` | UTF-8 字符串 | `s: name = "Alice";` |
| `b` | 布尔值 | `b: flag = true;` |
| `date` | 日期/时间（毫秒） | `date: d = date("2024-01-01");` |
| `json` | 任意 JSON | `json: data = <<<{}>>>;` |
| `array:T` | 类型化数组 | `array:i nums = [1, 2, 3];` |
| `set:N/Q/Z/S/C/B` | 集合 | `set:N mySet = set:N { 1,,100 };` |
| `map:K<->V` | 映射 | `map:s<->i scores = ["Alice" :: 100];` |
| `table` | 关系表 | `table: ...` |
| `fiber:T` | 协程 | `fiber:i f = myFiber(10);` |
| `database` | SQLite 连接 | `database: db = database { ... };` |

### 类型转换

```xcx
i(x)   --- to integer
f(x)   --- to float
s(x)   --- to string
b(x)   --- to boolean
```

---

## 变量与常量

```xcx
--- Declaration with explicit type
i: count = 0;
s: greeting = "Hello";

--- Constant (cannot be reassigned)
const f: PI = 3.14159;

--- Type Inference
var result = 42;        --- type inferred as int
var name = "Bob";       --- type inferred as string
```

### 赋值

```xcx
count = count + 1;
greeting = "World";
```

---

## 运算符

### 算术

| 运算符 | 说明 | 示例 |
|---|---|---|
| `+` | 加法 / 字符串拼接 | `a + b` |
| `-` | 减法 | `a - b` |
| `*` | 乘法 | `a * b` |
| `/` | 除法 | `a / b` |
| `%` | 取模 | `a % b` |
| `^` | 幂运算 | `a ^ b` |
| `++` | 数字拼接 | `12 ++ 34` = `1234` |

### 比较

`==`、`!=`、`>`、`<`、`>=`、`<=`

### 逻辑

| 运算符 | 替代写法 | 说明 |
|---|---|---|
| `AND` | `&&` | 逻辑与 |
| `OR` | `||` | 逻辑或 |
| `NOT` | `!!` | 逻辑非 |
| `HAS` | — | 包含关系（`"abc" HAS "b"`） |

### 集合运算符

| 运算符 | 符号 | Unicode |
|---|---|---|
| `UNION` | | `∪` |
| `INTERSECTION` | | `∩` |
| `DIFFERENCE` | `\` | |
| `SYMMETRIC_DIFFERENCE` | | `⊕` |

### 日期运算符

```xcx
date("2024-01-15") + 7    --- date + days
date("2024-01-15") - 7    --- date - days
date("2024-01-15") - date("2024-01-01")  --- difference in ms
```

---

## 控制流

### If / ElseIf / Else

```xcx
if (condition) then;
    ...
elseif (other_condition) then;
    ...
else;
    ...
end;
```

### While

```xcx
while (condition) do;
    ...
end;
```

### For（数值范围）

```xcx
for i in 1 to 100 do;
    >! i;
end;

--- With step
for i in 0 to 100 @step 5 do;
    >! i;
end;
```

### For（数组 / 集合 / 纤程）

```xcx
for item in myArray do;
    >! item;
end;

for elem in mySet do;
    >! elem;
end;

for val in myFiber do;
    >! val;
end;
```

### Break 与 Continue

```xcx
while (true) do;
    if (condition) then;
        break;
    end;
    if (skip) then;
        continue;
    end;
end;
```

---

## 函数

### 花括号风格（类 C）

```xcx
func add(i: a, i: b -> i) {
    return a + b;
}
```

### XCX 风格（关键字块）

```xcx
func:i: add(i: a, i: b) do;
    return a + b;
end;
```

### Lambda

```xcx
var double = x -> x * 2;
var sum = (x, y) -> x + y;
```

### 调用

```xcx
i: result = add(3, 4);
```

---

## 纤程（协程）

纤程是协作式协程 — 不是 OS 线程。

### 纤程定义

```xcx
fiber counter(i: start) {
    i: n = start;
    while (true) do;
        yield n;
        n = n + 1;
    end;
}
```

### 纤程实例化

```xcx
fiber:i: c = counter(1);
```

### 遍历纤程

```xcx
for val in c do;
    >! val;
    if (val >= 10) then;
        break;
    end;
end;
```

### 手动恢复

```xcx
i: val = c.next();
b: done = c.isDone();
c.close();
```

### Yield 委托

```xcx
fiber pipeline(array:i data) {
    yield from processData(data);
}
```

### 无返回值纤程

```xcx
fiber doWork() {
    --- do work
    yield;  --- suspension without value
}

fiber:void: w = doWork();
w.run();
```

---

## 集合

### 数组

```xcx
array:i nums = [1, 2, 3, 4, 5];
nums.push(6);
i: first = nums.get(0);
nums.set(0, 99);
nums.delete(0);
i: size = nums.size();
b: has = nums.contains(3);
nums.sort();
nums.reverse();
s: joined = nums.join(", ");
```

### 集合

```xcx
--- Natural set with range
set:N primes = set:N { 2, 3, 5, 7, 11 };

--- Set with range
set:Z mySet = set:Z { -10,,10 };

--- Set with step
set:N evens = set:N { 2,,100 @step 2 };

--- Operations
primes.add(13);
primes.remove(2);
b: hasFive = primes.contains(5);
i: count = primes.size();

--- Set operations
set:N union = a UNION b;
set:N inter = a INTERSECTION b;
set:N diff  = a DIFFERENCE b;
set:N sym   = a SYMMETRIC_DIFFERENCE b;
```

### 映射

```xcx
map:s<->i scores = ["Alice" :: 100, "Bob" :: 85];
scores.set("Charlie", 90);
i: aliceScore = scores.get("Alice");
scores.remove("Bob");
array:s keys = scores.keys();
```

---

## JSON

```xcx
--- JSON Parsing
json: data = json.parse(jsonString);

--- Raw block (inline JSON)
json: config = <<<{ "host": "localhost", "port": 8080 }>>>;

--- Field Access
json: host = data.host;
json: nested = data.get("/server/host");

--- Modification
data.set("/port", 9090);
data.push("/items", "newItem");

--- Binding to variables
s: hostname;
data.bind("/host", hostname);

--- Injecting into a table
data.inject(mapping, myTable);

--- Checking
b: exists = data.exists("/optional");
i: count = data.size();
```

---

## 表

```xcx
--- Table Definition
table: users = table {
    columns = [
        id   :: i @auto,
        name :: s,
        age  :: i,
        email :: s
    ]
    rows = EMPTY
};

--- Inserting
users.insert("Alice", 30, "alice@example.com");
users.insert(name = "Bob", age = 25, email = "bob@example.com");

--- Querying
table: adults = users.where(row -> row.age >= 18);
table: result = users.join(orders, "id", "userId");

--- Displaying
users.show();

--- Rows
i: count = users.count();
json: row = users.get(0);

--- Updating
users.update(0, ["Alice Updated", 31, "alice@new.com"]);

--- Deleting
users.delete(0);

--- Conversion to JSON
json: usersJson = users.toJson();

--- Saving (@pk required for save())
table: products = table {
    columns = [id :: i @pk, name :: s, price :: f]
    rows = EMPTY
};
products.save("Laptop", 999.99);  --- insert or update
```

---

## 数据库（SQLite）

```xcx
--- Connection
database: db = database {
    engine = "sqlite",
    path = "myapp.db",
    users = users,
    products = products
};

--- Schema is automatically synchronized

--- Inserting
db.insert(users, "Alice", 30, "alice@example.com");

--- Querying
table: result = db.fetch(users);

--- Filtered Query
table: adults = db.fetch(users.where(row -> row.age >= 18));

--- Raw SQL
json: rows = db.queryRaw("SELECT * FROM users WHERE age > 25");

--- Parameterized SQL
json: res = db.exec("INSERT INTO logs (msg) VALUES (?)", ["event"]);

--- Transactions
db.begin();
db.insert(users, "Charlie", 20, "charlie@example.com");
db.commit();

--- Deleting (requires .where())
db.remove(users).where(row -> row.age < 18);

--- Schema
db.sync(users);
db.drop(users);
db.truncate(users);
```

---

## 网络（HTTP）

### 简单 HTTP 请求

```xcx
--- GET
json: resp = net.get("https://api.example.com/data");

--- POST
json: resp = net.post("https://api.example.com/users", payload);

--- PUT, DELETE, PATCH
json: resp = net.put("https://api.example.com/users/1", data);
json: resp = net.delete("https://api.example.com/users/1");
```

### 完整请求

```xcx
net.request {
    method = "POST",
    url = "https://api.example.com/users",
    headers = ["Authorization" :: "Bearer token123"],
    body = payload,
    timeout = 5000
} as resp;

--- Checking the response
b: ok = resp.ok;
i: status = resp.status;
json: body = resp.body;
```

### HTTP 服务器

```xcx
fiber getHandler(json: req) {
    s: path = req.url;
    json: response = <<<{ "message": "Hello World" }>>>;
    net.respond(200, response);
}

serve: myServer {
    port = 8080,
    host = "0.0.0.0",
    workers = 4,
    routes = [
        ["GET /"    :: getHandler],
        ["POST /api" :: postHandler]
    ]
};
```

---

## 存储（文件）

```xcx
--- Read / Write
s: content = store.read("data.txt");
store.write("output.txt", "Hello World");
store.append("log.txt", "new line\n");

--- Checking
b: exists = store.exists("file.txt");
i: size = store.size("file.txt");
b: isDir = store.isDir("path/");

--- Directory Operations
store.mkdir("newdir");
array:s files = store.list("mydir");
array:s matches = store.glob("*.xcx");

--- Deleting
store.delete("temp.txt");

--- Archiving
b: zipped = store.zip("folder", "archive.zip");
b: ok = store.unzip("archive.zip", "output/");
```

---

## 终端

```xcx
--- Print without newline
.terminal!write "Hello ";
.terminal!write "World\n";

--- Clear screen
.terminal!clear;

--- Exit program
.terminal!exit;

--- Run system command
b: ok = .terminal!run "ls -la";

--- Raw mode (for interactive apps)
.terminal!raw;

--- Read key (non-blocking)
s: key = input.key();

--- Read key (blocking)
s: key = input.key() @wait;

--- Check input availability
b: ready = input.ready();

--- Cursor
.terminal!cursor on;
.terminal!cursor off;
.terminal!move 10 5;    --- column 10, line 5

--- Return to normal mode
.terminal!normal;
```

---

## 日期与时间

```xcx
--- Current date
date: now = date.now();

--- Date Literal
date: d = date("2024-01-15");
date: d2 = date("15/01/2024", "DD/MM/YYYY");

--- Components
i: year = d.year();
i: month = d.month();
i: day = d.day();
i: hour = d.hour();
i: minute = d.minute();
i: second = d.second();

--- Formatting
s: formatted = d.format("YYYY-MM-DD");
s: withTime = d.format("YYYY-MM-DD HH:mm:ss");

--- Arithmetic
date: tomorrow = d + 1;
date: yesterday = d - 1;
i: diff = date("2024-12-31") - date("2024-01-01");
```

---

## 随机数

```xcx
--- Random integer (range)
i: n = random.int(1, 100);

--- Random integer with step
i: even = random.int(2, 100 @step 2);

--- Random float
f: x = random.float(0.0, 1.0);

--- Random float with step
f: y = random.float(0.0, 10.0 @step 0.5);

--- Random element from collection
array:s colors = ["red", "green", "blue"];
s: picked = random.choice from colors;

set:N nums = set:N { 1,,10 };
i: val = random.choice from nums;
```

---

## 加密

```xcx
--- Hashing
s: hash = crypto.hash(password, "bcrypt");
s: hash2 = crypto.hash(password, "argon2");

--- Verification
b: ok = crypto.verify(password, hash, "bcrypt");
b: ok2 = crypto.verify(password, hash2, "argon2");

--- Random token (hex)
s: token = crypto.token(32);   --- 32 hex characters

--- Base64
s: encoded = crypto.hash(data, "base64_encode");
s: decoded = crypto.hash(data, "base64_decode");
```

---

## 诊断与 Halt

```xcx
--- Print (>!)
>! "Hello World";
>! age;
>! "Value: " + s(count);

--- Input (>?)
>? name;       --- reads into an existing variable
>? age;

--- Halt levels
halt.alert >! "Warning";          --- continues execution
halt.error >! "Logic Error";      --- stops frame
halt.fatal >! "Critical Error";   --- stops frame

--- Wait
@wait(1000);     --- wait 1 second (ms)
@wait 500;       --- alternative syntax
```

---

## Include

```xcx
--- Simple include (deduplicated — once)
include "utils.xcx";

--- Include with alias (all names get a prefix)
include "math.xcx" as math;

--- Usage after aliased include
f: result = math.sin(3.14);
f: cosVal = math.cos(0.0);
```

### 搜索路径

1. 相对于当前文件所在目录
2. 在 `lib/` 目录中（CWD 和 XCX 库路径）

---

## 环境变量与 CLI

```xcx
--- Environment Variable
s: apiKey = env.get("API_KEY");

--- Command Line Arguments
array:s args = env.args();
s: firstArg = args.get(0);
```

---

## 注释

```xcx
--- Single-line comment

---
Multi-line
comment
*---
```
