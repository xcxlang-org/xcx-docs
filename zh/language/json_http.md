# XCX 3.1 JSON 与 HTTP

## JSON

XCX 3.1 中的 JSON 对象是**可变的**。

### 创建

```xcx
json: config <<< {"port": 8080, "debug": false} >>>;
json: user   <<< {"name": "", "age": 0} >>>;
```

字面量中的值可以是占位符（`""`、`0`、`false`），之后通过 `.set()` 填充。

### 从集合序列化
也可使用 `.toJson()` 方法直接从 XCX 集合创建 JSON 对象和数组。适用于：
- **映射**：返回 JSON 对象。
- **表**：返回 JSON 对象数组。

映射与行为的更多细节见[集合文档](collections.md)。

### 解析

```xcx
json: parsed = json.parse(raw_string);
```

> [!CAUTION]
> **无效 JSON 时崩溃（R305）**：解析失败时，VM 会立即终止。解析前请验证字符串内容。

### 可变模式

用零值声明模式，再填充：

```xcx
json: resp <<< {"token": "", "role": "", "uid": 0, "ok": false} >>>;
resp.set("token", crypto.token(32));
resp.set("role",  "admin");
resp.set("uid",   42);
resp.set("ok",    true);
yield net.respond(200, resp);
```

### JSON 方法

| 方法                      | 签名                  | 返回值 | 说明                                              |
|---------------------------|-----------------------|--------|---------------------------------------------------|
| `.exists(path)`           | `(s) → b`             | `b`    | 检查路径是否存在且非 null                         |
| `.get(path/idx)`          | `(s/i) → json`        | `json` | 获取路径或索引处的元素                            |
| `.bind(path, var)`         | `(s, ref) → b`        | `b`    | 将值提取到**已声明**的 XCX 变量中                 |
| `.set(path, val)`         | `(s, T) → b`          | `b`    | 设置路径处的值；键不存在则创建                    |
| `.push(val)`              | `(json) → b`          | `b`    | 向 JSON 数组节点追加元素                          |
| `.size()` / `.count()`    | `() → i`              | `i`    | 键数量（对象）或元素数量（数组）                  |
| `.toStr()`                | `() → s`              | `s`    | 序列化为 JSON 字符串                              |
| `.inject(path, map, tbl)` | `(s, map, table) → b` | `b`    | 将 JSON 数组批量导入 XCX 表                       |
| `.first()`                | `() → json`           | `json` | 返回 JSON 数组的第一个元素；为空时 `halt.error`   |

> [!NOTE]
> **JSON 数组上的 `.push()`**：`.push()` 仅适用于数组（`[]`）类型的 JSON 节点。对 JSON 对象（`{}`）调用 `.push()` 会导致 `halt.error`。
>
> ```xcx
> json: data <<< {"items": []} >>>;
> json: obj <<< {"id": 1} >>>;
> data.get("items").push(obj);   --- OK: "items" is an array
> data.push(obj);                --- halt.error: data is an object, not an array
> ```

> [!IMPORTANT]
> **`.bind()` 语法**：第二个参数必须是**先前已声明的变量**。不能内联声明类型。
>
> ```xcx
> --- Wrong
> req.bind("ip", s: ip);
>
> --- Correct
> s: ip;
> req.bind("ip", ip);
> ```

### 路径表示法

嵌套访问同时支持点表示法和方括号表示法：

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

### .inject() — 批量导入

将 JSON 数组直接导入 XCX 表：

```xcx
json: data <<< {"users":[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]} >>>;
table: imported { columns=[uid::i, uname::s] rows=[EMPTY] };
map: mapping { schema=[s<->s] data=["uid"::"id", "uname"::"name"] };
data.inject("users", mapping, imported);
imported.show();
```

### HTTP 请求中的 JSON

在 HTTP 处理器内，`json: req` 的结构如下：

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

## HTTP 客户端

### 高级 API

| 方法                   | 签名               | 返回值 |
|------------------------|--------------------|--------|
| `net.get(url)`         | `(s) → json`       | `json` |
| `net.post(url, body)`  | `(s, json) → json` | `json` |
| `net.put(url, body)`   | `(s, json) → json` | `json` |
| `net.delete(url)`      | `(s) → json`       | `json` |

```xcx
json: resp = net.get("https://api.example.com/users");

json: body <<< {"name": "Alice"} >>>;
json: resp = net.post("https://api.example.com/users", body);
```

### 响应对象

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

| 字段      | 类型   | 说明                                              |
|-----------|--------|---------------------------------------------------|
| `status`  | `i`    | HTTP 状态码                                       |
| `ok`      | `b`    | 状态码 >= 200 且 < 300 时为 `true`                |
| `body`    | `json` | 解析后的响应体（若为 JSON）；否则为 `null`        |
| `headers` | `json` | 响应头                                            |
| `text`    | `s`    | 响应体原始字符串（始终可用）                      |
| `error`   | `s`    | 请求失败时的错误信息；成功时为空字符串            |

`ok` 在状态码 >= 200 且 < 300 时为 `true`。访问 `body` 前应先检查 `ok`。

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

### 低级构建器（`net.request`）

```xcx
net.request {
    method  = "POST",
    url     = "https://api.example.com/data",
    headers = ["Authorization" :: "Bearer xyz", "X-App" :: "XCX"],
    body    = my_json,
    timeout = 5000
} as resp;
```

| 字段      | 类型        | 必填 | 默认值   | 说明                          |
|-----------|-------------|------|----------|-------------------------------|
| `method`  | `s`         | 是   | —        | `"GET"`、`"POST"`、`"PUT"` 等 |
| `url`     | `s`         | 是   | —        | 带 scheme 的完整 URL          |
| `headers` | `map:s<->s` | 否   | `{}`     | 附加请求头                    |
| `body`    | `json`      | 否   | `null`   | GET 和 DELETE 时忽略          |
| `timeout` | `i`         | 否   | `10000`  | 毫秒                          |

---

## HTTP 服务器

### serve 指令

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

| 字段      | 类型         | 必填 | 默认值        | 说明                              |
|-----------|--------------|------|---------------|-----------------------------------|
| `port`    | `i`          | 是   | —             | 端口号（1–65535）                 |
| `host`    | `s`          | 否   | `"127.0.0.1"` | `"0.0.0.0"` = 所有接口            |
| `workers` | `i`          | 否   | `1`           | 每个请求的纤程数                  |
| `routes`  | route list   | 是   | —             | 自上而下匹配，先匹配者优先        |

> [!NOTE]
> `serve:` 是**终止语句**。该指令之后的代码不会执行。
> 通配符 `*` 匹配任意方法或路径——应放在最后。

### 处理器纤程

每个处理器必须是签名为 `fiber name(json: req -> json)` 的纤程：

```xcx
fiber handle_health(json: req -> json) {
    yield net.respond(200, <<< {"status": "ok"} >>>);
};
```

未调用 `yield net.respond(...)` 的处理器会自动返回 `500 Internal Server Error`。

### net.respond()

```xcx
yield net.respond(200, my_json);
yield net.respond(201, my_json, ["Location" :: "/users/42"]);
yield net.respond(204, <<< {} >>>);
yield net.respond(404, <<< {"error": "not found"} >>>);
```

| 参数    | 类型        | 必填 | 说明                                              |
|---------|-------------|------|---------------------------------------------------|
| status  | `i`         | 是   | HTTP 状态码                                       |
| body    | `json \| s` | 是   | JSON 对象或原始字符串；空 JSON 使用 `<<< {} >>>`  |
| headers | `map:s<->s` | 否   | 附加响应头                                        |

### CORS 与预检

默认情况下，XCX 引擎会在响应中缺少以下头时自动添加 CORS 支持：
- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS, DELETE, PATCH`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-TOKEN`

> [!TIP]
> **推荐做法**：虽然引擎提供默认值，但**强烈建议**在代码中显式声明这些头。这能确保应用在不同环境中行为可预测，并清晰记录其安全策略。

对于预检（`OPTIONS`）支持，使用专用处理器显式设置允许的方法与头：

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

### 并发请求

纤程允许 I/O 重叠：

```xcx
fiber fetch(s: url -> json) {
    yield net.get(url);
};

fiber:json: f1 = fetch("https://api.example.com/users");
fiber:json: f2 = fetch("https://api.example.com/posts");
json: r1 = f1.next();
json: r2 = f2.next();
```

### 安全约束

| 约束                              | 行为                                           |
|-----------------------------------|------------------------------------------------|
| `localhost` / `127.0.0.1`         | 默认允许                                       |
| `169.254.x.x`（链路本地）         | `halt.fatal` — SSRF 防护                       |
| `10.x`、`172.16.x`、`192.168.x`   | 生产模式下阻止                                 |
| `file://` URL                     | `halt.fatal`                                   |
| 最大响应体大小                    | 10 MB                                          |
| 最大入站请求体                    | 10 MB — 返回 413，不调用处理器                 |
